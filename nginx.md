以下の手順で **(1) nginx コンテナを立てる → (2) GitLab から自動デプロイする CI/CD** を仕上げます。
_GitLab（:8181/8443）・GitLab Runner は既に Docker 上で稼働している前提です。_

---

## 0. ざっくり全体像

```text
EC2
├─ GitLab            :8181 / 8443   （既に起動済み）
├─ GitLab Runner     docker executor（既に登録済み）
└─ nginx-app         :80            ← これを追加
        ▲
        └─ /srv/site  … HTML を置くだけ
              ▲
              └─ GitLab repo の site/ を CI が rsync
```

---

## 1. ホスト側ディレクトリ & テストページ

```bash
sudo mkdir -p /srv/site
sudo chown $(whoami):$(whoami) /srv/site

echo "Hello GitLab CI/CD! – $(date)" > /srv/site/index.html
```

---

## 2. nginx-app サービスを docker-compose に追記

### 2-1 既存 `docker-compose.yml` へ追加（例）

```yaml
services:
  gitlab: # すでにある GitLab 定義 …
  gitlab-runner: # すでにある Runner 定義 …

  nginx-app: # ★ 追加
    image: nginx:alpine
    container_name: nginx-app
    volumes:
      - /srv/site:/usr/share/nginx/html:ro
    ports:
      - "80:80" # 80 番を公開
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost || exit 1"]
      interval: 30s
      timeout: 3s
      retries: 3
```

```bash
docker compose up -d nginx-app
curl http://localhost          #=> “Hello GitLab CI/CD!”
```

---

## 3. GitLab Runner からホストの `/srv/site` と Docker に触れるようにする

`/srv/gitlab-runner/config/config.toml` を編集して **volumes に `/srv/site:/srv/site`** を追加。
（`/var/run/docker.sock` は登録時に付けていれば OK）

```toml
[[runners]]
  executor = "docker"
  [runners.docker]
    privileged = true
    volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/srv/site:/srv/site"]
    # 他の設定はそのまま
```

> 設定変更後：`docker restart gitlab-runner`

---

## 4. GitLab プロジェクトを作成してコードを push

```bash
git clone <your_repo>.git
cd your_repo
mkdir site
echo "Hello GitLab CI/CD from Git!" > site/index.html
git add site/index.html
git commit -m "init: add static site"
git push
```

---

## 5. `.gitlab-ci.yml` を置く

プロジェクト直下に以下を追加して commit → push。
**タグ `docker` は runner 登録時に設定したものと合わせてください。**

```yaml
stages: [test, deploy]

.default_rules: &default_rules
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: always
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"'
      when: always
    - when: never

test:
  stage: test
  image: alpine
  <<: *default_rules
  script:
    - test -f site/index.html

deploy:
  stage: deploy
  tags: [docker]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"'
      when: always
  script:
    - rsync -av --delete site/ /srv/site/
    - docker compose exec nginx-app nginx -s reload
  environment:
    name: production
    url: http://$CI_SERVER_HOST
```

---

## 6. 動作確認フロー

| 手順                                         | 期待動作                                      |
| -------------------------------------------- | --------------------------------------------- |
| ① MR を作成                                  | `test` ジョブが走り、index.html 有無チェック  |
| ② main にマージ / 直接 push                  | `deploy` ジョブが実行 → `/srv/site/` へ rsync |
| ③ ジョブ完了後ブラウザを更新                 | 変更が反映されていれば成功                    |
| ④ `docker logs nginx-app` でアクセスログ確認 | デプロイ後アクセスが記録されていることを確認  |

---

## 7. HTTPS を付けたい場合（任意）

1. `nginxproxy/nginx-proxy` ＋ `acme-companion` を compose に追加。
2. `nginx-app` には `VIRTUAL_HOST=example.com` 環境変数を付与。
3. GitLab は引き続き 8181/8443、公開 nginx は 80/443 で分離。

---

## 8. よくあるハマりポイント

| 症状 / エラー                        | 原因                                | 解決策                                                    |
| ------------------------------------ | ----------------------------------- | --------------------------------------------------------- |
| `deploy` ジョブで `/srv/site` が無い | Runner コンテナに volume 未マウント | `config.toml` の `volumes` に `/srv/site:/srv/site` 追加  |
| `docker compose exec` が失敗         | Runner → docker.sock が無権限       | `privileged = true` & `/var/run/docker.sock` をマウント   |
| ブラウザが古い内容を表示             | キャッシュ                          | `Ctrl+F5` or nginx で `add_header Cache-Control no-store` |
| 443 で GitLab/公開 nginx が衝突      | 両方が同ポート使用                  | GitLab は 8443、公開 nginx は 443 でポート分離            |

---

### これで完成 🎉

- **EC2 上の 3 コンテナ（GitLab / Runner / nginx-app）だけ** で静的サイトの CI/CD が回ります。
- SPA や API サーバーに差し替える場合は

  1. `build` ステップで成果物を `site/` へコピーし、
  2. `deploy` で同じ rsync → reload を行えば OK。

まずは `index.html` をちょっと書き換えて push → 自動反映を確認してみてください！

**以下では **シェルで打つコマンド** と **YAML の各行\*_ を 1 行ずつ取り上げ、「何が起こるか」「なぜ必要か」を丁寧に解説します。
_――見ながら手を動かせば、仕組みがスッと頭に入るはずです。\*

---

## 1. ホスト側ディレクトリ & テストページ

| コマンド                                                      | 何をする？                                                                                                | なぜ必要？                                                                                          |
| ------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `sudo mkdir -p /srv/site`                                     | `/srv/site` というディレクトリを **root 権限**で作成。`-p` で途中の親ディレクトリが無くてもまとめて作る。 | nginx コンテナが “ここをドキュメントルートにする” ための **実体** を用意。                          |
| `sudo chown $(whoami):$(whoami) /srv/site`                    | できた `/srv/site` を「今ログインしているユーザ (whoami)」に所有権変更。                                  | `rsync` で書き込みを行うのは Runner → ホストの `/srv/site`。root のままだと書き込み権限でコケがち。 |
| `echo "Hello GitLab CI/CD! – $(date)" > /srv/site/index.html` | テスト用の `index.html` を一発生成。`$(date)` で現在時刻も埋め込み。                                      | nginx が正しく立ち上がったか `curl` で確認する足がかり。                                            |

---

## 2. docker-compose.yml の追加行を分解

```yaml
nginx-app:
  image: nginx:alpine # ①
  container_name: nginx-app # ②
  volumes: # ③
    - /srv/site:/usr/share/nginx/html:ro
  ports:
    - "80:80" # ④
  restart: always # ⑤
  healthcheck: # ⑥
    test: ["CMD-SHELL", "wget -qO- http://localhost || exit 1"]
    interval: 30s
    timeout: 3s
    retries: 3
```

| 番号 | 行                                              | 意味                                                                                                     |
| ---- | ----------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| ①    | `image: nginx:alpine`                           | 公式 nginx の **最小イメージ（Alpine Linux ベース）** を使う。サイズ ≈5 MB と激軽。                      |
| ②    | `container_name: nginx-app`                     | コンテナに固定名を付ける。`docker compose exec nginx-app` で参照しやすくするため。                       |
| ③    | `volumes:` `/srv/site:/usr/share/nginx/html:ro` | ホスト `/srv/site` を **コンテナのドキュメントルート**にバインド。`:ro` で読み取り専用＝誤上書きを防止。 |
| ④    | `ports: "80:80"`                                | ホストの **80 番ポート** をコンテナの 80 番に公開。ブラウザが `http://<EC2_IP>/` で見られる。            |
| ⑤    | `restart: always`                               | EC2 再起動や nginx がクラッシュしても **自動再起動**。運用でありがちな「なぜか落ちてた」を回避。         |
| ⑥    | `healthcheck:`                                  | 30 秒ごとに `wget` で自己チェック。応答が無いと **Docker が状態を unhealthy に** → 監視で検知しやすい。  |

### 起動コマンド

| コマンド                         | 何をする？                                                                             |
| -------------------------------- | -------------------------------------------------------------------------------------- |
| `docker compose up -d nginx-app` | `-d` でバックグラウンド起動。`docker ps` で `Up` 状態確認可。                          |
| `curl http://localhost`          | **EC2 内から** HTTP リクエスト。`Hello GitLab CI/CD!` が返れば nginx + volume が正常。 |

---

## 3. Runner の `config.toml` 追記

```toml
volumes = [
 "/var/run/docker.sock:/var/run/docker.sock",
 "/srv/site:/srv/site"
]
```

| 行                                          | 意味                                                                                                                                      |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `/var/run/docker.sock:/var/run/docker.sock` | Runner コンテナ内から **ホストの Docker デーモン** を叩けるようにする。「コンテナ in コンテナ」の定番設定。`privileged = true` も忘れず。 |
| `/srv/site:/srv/site`                       | Runner が **ビルド成果物を直接ホストへ rsync** できるようにする。パスを同じにすると混乱がない。                                           |

```bash
docker restart gitlab-runner
```

> `config.toml` は読み込み時にしか反映されないので **必ず再起動**。

---

## 4. Git 操作

| コマンド                             | 意味                                                        |
| ------------------------------------ | ----------------------------------------------------------- |
| `git clone <your_repo>.git`          | GitLab に作った空リポジトリをローカルへ複製。               |
| `mkdir site`                         | CI が拾う “静的サイト置き場”。                              |
| `echo "Hello ..." > site/index.html` | サンプルページをコミット対象に。                            |
| `git add`, `commit`, `push`          | GitLab にコードを反映 ⇒ Push 時点で CI が走る仕掛けになる。 |

---

## 5. `.gitlab-ci.yml` の各ブロック解説

### ヘッダ

```yaml
stages: [test, deploy]
```

- **パイプラインは 2 段階**：`test` → `deploy`。上流が落ちれば下流は動かない。

### ルール共通部分

```yaml
.default_rules: &default_rules
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"' # (1)
      when: always
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"' # (2)
      when: always
    - when: never # どちらでもなければ実行しない
```

| ラベル | 意味                                                   |
| ------ | ------------------------------------------------------ |
| (1)    | **MR パイプライン**（レビュー時）でも動かす。          |
| (2)    | **main 直 push**（マージ後 or 直コミット）でも動かす。 |

### test ジョブ

```yaml
test:
  stage: test
  image: alpine                # 3 MB の汎用 Linux
  <<: *default_rules
  script:
    - test -f site/index.html  # (A)
```

| 行  | 意味                                                                               |
| --- | ---------------------------------------------------------------------------------- |
| (A) | `site/index.html` が無ければ `exit 1` → パイプライン失敗。**最低限の品質ゲート**。 |

### deploy ジョブ

```yaml
deploy:
  stage: deploy
  tags: [docker] # Runner を選択
  rules: # main に push した時だけ
    - if: '$CI_COMMIT_BRANCH=="main" && $CI_PIPELINE_SOURCE=="push"'
      when: always
  script:
    - rsync -av --delete site/ /srv/site/ # (B)
    - docker compose exec nginx-app nginx -s reload # (C)
  environment:
    name: production
    url: http://$CI_SERVER_HOST
```

| ラベル | コマンド                                        | 作用                                                                               |
| ------ | ----------------------------------------------- | ---------------------------------------------------------------------------------- |
| (B)    | `rsync -av --delete site/ /srv/site/`           | **差分同期 + 古いファイル削除**。`a` はパーミッション保持、`v` は verbose。        |
| (C)    | `docker compose exec nginx-app nginx -s reload` | nginx ワーカープロセスへ **再読込シグナル**。設定変更なし & ダウンタイムほぼゼロ。 |

---

## 6. 動作確認フロー

| 手順                      | 何が起こる？                                                           |
| ------------------------- | ---------------------------------------------------------------------- |
| ① MR 作成                 | Push トリガで **`test` ジョブ**のみ実行 → OK/NG がレビュー画面に表示。 |
| ② main へマージ           | マージコミットが push され **`deploy` ジョブ** も実行。                |
| ③ ブラウザ確認            | `/srv/site` が更新 → nginx が reload → 新しいページが返る。            |
| ④ `docker logs nginx-app` | アクセスログが時刻付きで出るので「本当に見えてるか」検証に便利。       |

---

## 7. HTTPS 化オプション一言

- `nginxproxy/nginx-proxy` が **自動で vhost を生成**。
- `acme-companion` が **Let’s Encrypt 証明書を取得・更新**。
- コンテナ環境変数 `VIRTUAL_HOST, LETSENCRYPT_HOST` だけで完結するので、GitLab 本体を 8443、公開 nginx を 443 にしてポートを分けるのが最もラク。

---

## 8. ハマりポイント再掲（原因 → 対処の因果が肝）

| 症状                             | 直接原因                       | 根本対策                                            |
| -------------------------------- | ------------------------------ | --------------------------------------------------- |
| `/srv/site` が無い               | Runner から見えない            | Runner の **volumes 追記** & 再起動                 |
| `docker compose exec` 権限エラー | docker.sock が read-only       | `privileged=true` + `/var/run/docker.sock` マウント |
| 反映されない                     | ブラウザキャッシュ             | `Ctrl+F5` or `Cache-Control: no-store`              |
| 443 ポート衝突                   | GitLab & 公開 nginx 両方が 443 | **GitLab → 8443、公開 nginx → 443** に整理          |

---

### ✅ まとめ

1. **/srv/site** を “共有フォルダ” に見立てて nginx と Runner で共有。
2. **rsync → nginx reload** の 2 行だけでデプロイ完了。
3. 何か動かなくなったら **volumes / 権限 / ポート衝突** を疑うと大抵当たります。

コマンドの意味まで腹落ちしたら、あとは `site/` を **React ビルド成果物** に差し替えるなど自由自在です。まずは `index.html` の文言を変えて push ➡️ 自動反映を体験してみてください 🚀
\*\*

## これからやること ― ざっくり 3 行まとめ

1. **“Web ページの置き場”** を `/srv/site` に用意し、nginx コンテナで公開する。
2. GitLab プロジェクトに **`site/` フォルダ**を置き、GitLab Runner が `/srv/site` へコピーする仕組み（CI/CD）を書く。
3. `main` ブランチに push すると **自動で反映**されることを確認する。

以下、専門用語をかみ砕きながら **完全初心者モード** で進めます 🔰

---

## 0. ゴールと現在地の確認

| 項目             | 今どこ？                                    | ゴール                       |
| ---------------- | ------------------------------------------- | ---------------------------- |
| GitLab 本体      | Docker で起動済み（`http://<EC2_IP>:8181`） | そのまま使う                 |
| GitLab Runner    | Docker で起動 & 登録済み                    | nginx 用のジョブを実行       |
| nginx            | まだ無い                                    | Docker コンテナで公開        |
| `.gitlab-ci.yml` | まだ無い                                    | **1 ファイル**で自動デプロイ |

---

## 1. 用語を 3 つだけ覚える

| 用語                        | ざっくり説明                                                                      |
| --------------------------- | --------------------------------------------------------------------------------- |
| **ジョブ (job)**            | “これやっといて” という **1 つの作業単位**（テストを走らせる、ビルドする…）       |
| **ステージ (stage)**        | ジョブをグループにしたもの。上流が成功すると下流が動く。<br>例：`test` → `deploy` |
| **パイプライン (pipeline)** | ジョブとステージを **全部まとめた流れ**。push したら自動で始まる。                |

---

## 2. ホストに公開フォルダを作る

```bash
# 1) 公開ディレクトリを作る
sudo mkdir -p /srv/site

# 2) 自分（ec2-user など）に書き込み権を渡す
sudo chown $(whoami):$(whoami) /srv/site

# 3) とりあえずテストページ
echo "初めての GitLab CI/CD! $(date)" > /srv/site/index.html
```

> **ここがポイント**
>
> - /srv/site が **実物の Web ルート** になる。
> - Runner からも nginx からも見えるように “共通のフォルダ” にしておく。

---

## 3. nginx コンテナを 1 分で立てる

1. **docker-compose.yml**（新規か既存に追加）

   ```yaml
   services:
     nginx-app: # 名前は自由
       image: nginx:alpine
       container_name: nginx-app
       volumes:
         - /srv/site:/usr/share/nginx/html:ro
       ports:
         - "80:80"
       restart: always
   ```

2. 起動

   ```bash
   docker compose up -d nginx-app
   curl http://localhost        # さっきのテキストが返れば OK
   ```

---

## 4. Runner から /srv/site を触れるようにする

`/srv/gitlab-runner/config/config.toml` を開き、**volumes** に追記：

```toml
volumes = [
  "/var/run/docker.sock:/var/run/docker.sock",  # docker 制御用
  "/srv/site:/srv/site"                         # 公開フォルダ
]
```

```bash
docker restart gitlab-runner    # 変更を反映
```

---

## 5. GitLab プロジェクトを用意してコードを push

```bash
git clone http://<EC2_IP>:8181/<namespace>/<project>.git
cd <project>
mkdir site
echo "Git から来たよ！" > site/index.html
git add site/index.html
git commit -m "feat: first page"
git push -u origin main
```

---

## 6. `.gitlab-ci.yml` を 18 行で書く

> プロジェクトの **ルート** に配置 → commit → push

```yaml
stages: [test, deploy] # 1) まず段階を並べる

unit-test: # ------------ test ジョブ ------------
  stage: test
  image: alpine
  script:
    - test -f site/index.html # 2) ファイルが無ければエラーで終了

deploy-to-nginx: # ------------ deploy ジョブ ------------
  stage: deploy
  tags: [docker] # Runner 登録時の tag と一致させる
  rules: # main ブランチに push した時だけ
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE=="push"'
      when: always
  script:
    - rsync -av --delete site/ /srv/site/ # 3) コピー
    - docker compose exec nginx-app nginx -s reload # 4) nginx を再読込
  environment:
    name: production
    url: http://$CI_SERVER_HOST
```

### 行ごとの意味

| 行                       | 役割                                                                        |
| ------------------------ | --------------------------------------------------------------------------- |
| `stages`                 | パイプラインが **test → deploy** の順に流れる宣言                           |
| `unit-test` ジョブ       | 最低限 “index.html があるか” だけ確認                                       |
| `deploy-to-nginx` ジョブ | - `rsync` で /srv/site を丸ごと更新<br>- `nginx -s reload` で数秒で切り替え |
| `rules`                  | main への push でしか deploy しない安全弁                                   |
| `tags`                   | どの Runner が動くか指定（docker executor の Runner）                       |

---

## 7. 動かしてみる

1. `index.html` の中身を編集して commit / push
2. GitLab の **CI/CD › Pipelines** を開く
3. `unit-test` → `deploy-to-nginx` がどちらも ✅ になったら成功
4. ブラウザで `http://<EC2_IP>/` をリロード → 新しい文言が出れば OK

---

## 8. もう少し先へ（キャッシュ・パッケージの“さわり”だけ）

| 機能                   | 1 行まとめ                                                   | 例を書くと…                                                     |
| ---------------------- | ------------------------------------------------------------ | --------------------------------------------------------------- |
| **cache:**             | “次のジョブ／次のパイプラインでも再利用したいファイル置き場” | `cache: {paths: [node_modules/]}` で npm install を高速化       |
| **artifacts:**         | “あとでダウンロードしたい or 次のジョブに渡したい成果物”     | `artifacts: {paths: [build.zip], expire_in: 1 week}`            |
| **Container Registry** | GitLab 付属の **Docker Hub もどき**                          | `docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA`                 |
| **Generic Packages**   | ZIP やバイナリをバージョン付きで保存                         | API で `/packages/generic/<name>/<ver>/file.zip` へアップロード |

> まずは **動くパイプラインを体験** → 慣れたら `cache` でビルド高速化、`Registry` でイメージ管理にステップアップするのがおすすめです。

---

### 📌 まとめ

- **今やったこと**：`site/` → `/srv/site/` → `nginx reload` を GitLab に肩代わりさせた。
- **覚えるキモ**：

  1. `script:` は “実際に EC2 で叩くシェルコマンド” がそのまま入る。
  2. Runner が見えるフォルダに **volume マウント** しておくとコピーが楽。
  3. トラブル時は **volumes / 権限 / ポート競合** をまず疑う。

これで **“push → 自動公開”** の基本が完成です。
もし行き詰まったら _どのコマンドがエラーを出しているか_ をログで確認 → 対象のコンテナに `docker exec -it <name> sh` で入って手作業すると原因が見つかりやすいですよ。

### どうまとめる？──「具体すぎず ✕ 抽象すぎず」のバランス案

> **ゴール**：
> 第 4 章を読んだ読者が
>
> 1. **自社プロジェクトに合わせて `.gitlab-ci.yml` を書ける**
> 2. ビルド時間短縮や成果物管理まで踏み込める
>    ― ここまで到達できれば十分です。
>    そのために **“汎用テンプレート＋ミニ具体例”** の二段構えが最も伝わりやすくなります。

---

## 推奨構成（章タイトルと見出し例）

| 節   | 見出し                                    | ねらい                                                                            | 書き方のヒント                                                       |
| ---- | ----------------------------------------- | --------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| 4-1  | `.gitlab-ci.yml` の全体構成               | **最小 4 要素**<br>`image / variables / stages / jobs` を図解                     | 4 行サンプル → 箱図で「上から下へ流れる」を視覚化                    |
| 4-2  | よく使うキーワード & テンプレート集       | **include / extends / rules / parallel** など “覚えると世界が変わる” 単語を一覧化 | 1 行サンプル＋用途メモだけに抑え、詳細は公式 URL へリンク            |
| 4-3  | キャッシュとアーティファクト              | **① 差分ビルド高速化 ② 成果物受け渡し** を 1 ページで比較                         | npm キャッシュ＋レポート共有の _汎用_ 例を載せる                     |
| 4-4  | GitLab Packages で成果物管理              | **Container Registry / Generic Package / npm Registry** を表に整理                | **図：開発 → CI → Registry → 本番** の流れで “なぜ必要か” を先に示す |
| 付録 | ミニ具体例：静的サイトを nginx でデプロイ | _最小動作_ を 30 行で提示（nginx 例）。本文リンクだけ。                           | 付録なので「動かしたい人はここをコピペ」で済む                       |

---

### 4-1 `.gitlab-ci.yml` の骨格（5 行だけの超ミニ例）

```yaml
image: alpine # 1. 実行環境
stages: [test, deploy] # 2. パイプライン段階

unit-test: # 3. ジョブ①
  stage: test
  script: echo OK

deploy-prod: # 4. ジョブ②
  stage: deploy
  script: echo deploy
```

> _ここで伝えること_
>
> - “上から順” に流れるイメージ
> - **ジョブは「キ-バリ (key-value)」の塊** ― 覚えるタグは `stage / script / rules` の 3 つから始める

---

### 4-2 テンプレート集（抜粋）

| キーワード   | 効果                     | 1 行サンプル                                                     |
| ------------ | ------------------------ | ---------------------------------------------------------------- |
| **include**  | 公式や共通 CI を取り込む | `include: {template: 'Go.gitlab-ci.yml'}`                        |
| **extends**  | ジョブの継承             | `extends: .docker-build`                                         |
| **rules**    | 発火条件                 | <small>`rules: {if: '$CI_COMMIT_TAG', when: on_success}`</small> |
| **parallel** | 行列ビルド               | <small>`parallel: {matrix: [{PY: 3.9}, {PY: 3.12}]}`</small>     |

> 🙋‍♂️ **ポイント**：「どう使う？」より「何が出来る？」を先に並べると初心者が迷わない。

---

### 4-3 キャッシュ vs アーティファクト対比表

| 比較項目           | **cache** (高速化)                          | **artifacts** (成果物)     |
| ------------------ | ------------------------------------------- | -------------------------- |
| 保存タイミング     | ジョブ _開始時_ に復元<br>終了時に保存      | ジョブ終了後にアップロード |
| 共有範囲           | パイプラインを超えて再利用可                | 後続ジョブ／ダウンロード用 |
| 代表用途           | `node_modules/`, `~/.m2`                    | HTML レポート, JAR, ZIP    |
| キー指定           | `key:` 必須 → 衝突防止                      | 不要（常に一意の ID）      |
| ベストプラクティス | キーに `$CI_COMMIT_REF_SLUG` でブランチ分離 | `expire_in` で自動削除     |

---

### 4-4 GitLab Packages の使い分け早見

| 成果物タイプ        | 保存先             | push コマンド例                                                 | pull コマンド例    |
| ------------------- | ------------------ | --------------------------------------------------------------- | ------------------ |
| **Docker イメージ** | Container Registry | `docker push $CI_REGISTRY_IMAGE:tag`                            | `docker pull ...`  |
| **ZIP/Binary**      | Generic Packages   | `curl --upload-file .../file.zip $API_URL/packages/generic/...` | 同 URL を wget     |
| **npm ライブラリ**  | npm Registry       | `npm publish --registry $GL_NPM_REGISTRY`                       | `npm i @scope/pkg` |

> 🔑 **コツ**：先に “運用課題” を書く → Packages が解決策になる構成にすると納得感が高い。

---

## 付録リンク例（nginx デプロイ）

> _「まずは動かしたい」読者向けにワンクリックで済む実用サンプル_ > [https://gitlab.example.com/snippets/1234](https://gitlab.example.com/snippets/1234)

---

### まとめ – なぜこの構成？

- **メイン本文**：抽象度をそろえ _“型”_ を学ばせる。
- **付録**：コピーだけで動く具体例を置き “体験” を保証。
- こうすると **初心者** も **経験者** も読み飛ばしやすく、運用ガイドとして過不足がありません。

> **結論**：「nginx の丸ごと手順」は付録へ分離し、本文は今の“汎用テンプレ＋対比表”スタイルで一般化するのが最適です。
> 以下を **まるごと ZIP で渡すか、GitLab “Import repository” で取り込む** だけで
> 静的サイトの CI→CD がすぐに体験できる最小サンプルです。
> （Runner は docker-executor、タグ `docker` が付いている前提）

---

## 1️⃣ リポジトリ構成

```
quick-start-nginx/
├── README.md
├── .gitlab-ci.yml          ← パイプライン定義（18 行）
├── site/                   ← そのまま Web に出るフォルダ
│   └── index.html
└── docker-compose.yml      ← EC2 で動かす nginx コンテナ
```

### 主要ファイル

**site/index.html**

```html
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="utf-8" />
    <title>Hello GitLab CI/CD</title>
  </head>
  <body>
    <h1>Hello GitLab CI/CD!</h1>
    <p>更新日時: <span id="ts"></span></p>
    <script>
      document.getElementById("ts").textContent = new Date();
    </script>
  </body>
</html>
```

**docker-compose.yml**

```yaml
version: "3.9"
services:
  nginx-app:
    image: nginx:alpine
    container_name: nginx-app
    volumes:
      - /srv/site:/usr/share/nginx/html:ro
    ports:
      - "80:80"
    restart: always
```

**.gitlab-ci.yml**

```yaml
stages: [test, deploy]

unit-test: # 1. HTML があるかだけ確認
  stage: test
  image: alpine
  script: test -f site/index.html

deploy-to-nginx: # 2. main へ push したら公開
  stage: deploy
  tags: [docker] # ★ Runner 登録時に付けた tag
  rules:
    - if: '$CI_COMMIT_BRANCH=="main" && $CI_PIPELINE_SOURCE=="push"'
      when: always
  script:
    - rsync -av --delete site/ /srv/site/
    - docker compose exec nginx-app nginx -s reload
  environment:
    name: production
    url: http://$CI_SERVER_HOST
```

---

## 2️⃣ 使い方（所要 3 分）

### A. EC2 側セットアップ（初回だけ）

```bash
## 1) ドキュメントルート & テストページ
sudo mkdir -p /srv/site && sudo chown $(whoami) /srv/site
echo "First deploy @ $(date)" > /srv/site/index.html

## 2) nginx コンテナ起動
docker compose -f docker-compose.yml up -d nginx-app
```

> ブラウザで `http://<EC2_IP>/` が見えたら OK。

### B. サンプルを GitLab に取り込む

1. GitLab › New Project › **“Import project” → “Upload .zip”**
   ZIP はそのまま quick-start-nginx ディレクトリを固めるだけ。
   （または `git clone` → origin 変更で push でも可）
2. **Settings › CI/CD › Runners** で
   docker-runner に **Tag = `docker`** が付いているか確認。
3. リポジトリの **`site/index.html` を 1 行編集** → commit → push

### C. 確認

| 見る場所                    | 期待結果                              |
| --------------------------- | ------------------------------------- |
| **CI/CD › Pipelines**       | `unit-test`→`deploy-to-nginx` が緑 ✅ |
| ブラウザ `http://<EC2_IP>/` | 変更後の文言が即反映                  |
| `docker logs nginx-app`     | 最新アクセスが記録されている          |

---

## 3️⃣ カスタマイズの道しるべ

| やりたいこと     | 触る場所                               | 変更例                                                         |
| ---------------- | -------------------------------------- | -------------------------------------------------------------- |
| 依存インストール | `.gitlab-ci.yml` の `unit-test` ジョブ | `image: node:20-alpine` → `npm ci && npm test`                 |
| SPA ビルド       | 同上                                   | `npm run build` → `rsync build/ /srv/site/`                    |
| HTTPS            | `docker-compose.yml`                   | `nginxproxy/nginx-proxy` + `acme-companion` を追加             |
| 複数環境         | `deploy-to-nginx` ジョブ               | `environment: name: review/$CI_COMMIT_REF_SLUG` で Review Apps |

---

## 4️⃣ “ワンクリック” インポート URL 例（自社 GitLab 用）

```
https://gitlab.example.com/projects/new#import_project
```

- 画面に ZIP をドラッグ & ドロップするだけ
- あるいは **テンプレートグループ**を用意し
  `include: {project: devops/ci-templates, file: /static-nginx.yml}`
  と書けばどのプロジェクトでも 1 行で流用できます。

---

> **このサンプル自体を運用ガイドの付録として置く**
>
> - 本文：汎用的な説明で “型” を解説
> - 付録：ZIP or Snippet で **即動くコード**
>   という構成にすれば、初心者はコピー、経験者は改造と、両方フォローできます。

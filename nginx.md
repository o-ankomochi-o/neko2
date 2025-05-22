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

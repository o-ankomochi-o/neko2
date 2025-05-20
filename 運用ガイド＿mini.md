## GitLab CI/CD 開始・運用ガイド（ミニマム版）

> **対象読者**
>
> - 既に Azure DevOps で CI/CD を運用しており、同等機能を GitLab（セルフホスト）へ最短で移行したいエンジニア
> - AWS 上に Docker で GitLab 本体と GitLab Runner を構築する予定の方

---

### 1. はじめに

- **ゴール**
  1 日で “ビルド → テスト → AWS へデプロイ” が通るパイプラインを GitLab 上で動かす。
- **進め方**

  1. EC2 & Docker Compose で GitLab 本体を立てる
  2. 同じホストに Docker executor の GitLab Runner を登録
  3. `.gitlab-ci.yml` をコミットしてパイプライン実行

---

### 2. 前提条件と準備

| 要素             | 推奨値                          | 備考                                                      |
| ---------------- | ------------------------------- | --------------------------------------------------------- |
| AWS インスタンス | `t3.medium` (2 vCPU / 4 GB)     | 小規模 POC 向け。利用ユーザが増える場合は `t3.large` 以上 |
| OS               | Ubuntu 22.04 LTS                | Amazon Linux でも可                                       |
| ドメイン         | 任意                            | IP アドレスでの運用も可能                                 |
| ポート           | 80 / 443 / 22                   | GitLab Web (HTTP/HTTPS), SSH                              |
| 永続ボリューム   | `/srv/gitlab` に 50 GB 以上     | Git リポジトリ & DB 用                                    |
| 事前インストール | Docker 25.x / Docker Compose v2 | `sudo apt install docker.io docker-compose-plugin`        |

> **セキュリティグループ**
>
> - インバウンド: 80, 443, 22（自社 IP 制限推奨）
> - アウトバウンド: すべて許可（必要に応じて制限）

---

### 3. GitLab コンテナのセットアップ

1. **ディレクトリ準備**

   ```bash
   sudo mkdir -p /srv/gitlab/{config,logs,data}
   sudo chown -R 1000:1000 /srv/gitlab   # GitLab UID/GID=1000
   ```

2. **`docker-compose.yml` (最小)**

   ```yaml
   version: "3.9"
   services:
     gitlab:
       image: gitlab/gitlab-ee:17.0.1-ee.0 # CE 版なら gitlab-ce
       container_name: gitlab
       hostname: gitlab.example.com # IP の場合は EC2 パブリック IP
       restart: always
       shm_size: 256m
       ports:
         - "80:80"
         - "443:443"
         - "22:22"
       volumes:
         - /srv/gitlab/config:/etc/gitlab
         - /srv/gitlab/logs:/var/log/gitlab
         - /srv/gitlab/data:/var/opt/gitlab
       environment:
         GITLAB_ROOT_PASSWORD: "ChangeMeStrong!" # 初期 root PW
   ```

3. **起動**

   ```bash
   sudo docker compose up -d
   ```

4. **初期確認**

   - ブラウザで `http://<EC2パブリックIP>` → root / `ChangeMeStrong!` でログイン
   - SSH も確認: `ssh git@<EC2 IP>`（fingerprint メッセージが出れば OK）

---

### 4. GitLab Runner のセットアップ（同ホスト）

1. **Runner 用ディレクトリ**

   ```bash
   sudo mkdir -p /srv/gitlab-runner
   ```

2. **Runner コンテナ起動**

   ```bash
   sudo docker run -d --name gitlab-runner --restart always \
     -v /srv/gitlab-runner/config:/etc/gitlab-runner \
     -v /var/run/docker.sock:/var/run/docker.sock \
     gitlab/gitlab-runner:alpine
   ```

3. **Runner 登録**

   ```bash
   sudo docker exec -it gitlab-runner gitlab-runner register
   ```

   - **GitLab URL**: `http://gitlab`（同ホスト内）または `http://<EC2 IP>`
   - **Registration Token**: GitLab Web → **\[Admin] > Runners > Registration token**
   - **Executor**: `docker`
   - **Default image**: `docker:25.0.3-dind` など（パイプライン内で DinD を使う場合）
   - **Tags**: `docker` など任意

> _Docker executor + DinD_ でコンテナをビルド →Push したい場合は、ジョブ内で `services: [docker:dind]` を利用。

---

### 5. 最小 `.gitlab-ci.yml` サンプル

```yaml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

build:
  stage: build
  tags: [docker]
  services:
    - docker:dind
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
    - docker build -t "$IMAGE_TAG" .
    - docker push "$IMAGE_TAG"
  artifacts:
    paths: [docker-image.tar] # 任意。必要に応じて省略

test:
  stage: test
  tags: [docker]
  script:
    - docker run --rm "$IMAGE_TAG" pytest # 例：pytest 実行

deploy:
  stage: deploy
  tags: [docker]
  script:
    - echo "Deploying to EC2..."
    - ssh ec2-user@$TARGET_HOST "docker pull $IMAGE_TAG && docker compose up -d"
  environment:
    name: production
    url: http://$TARGET_HOST
  when: manual # 手動トリガにしたい場合
```

- **Protected variables**

  - `TARGET_HOST`, `CI_REGISTRY_USER`, `CI_REGISTRY_PASSWORD` を GitLab → **Settings > CI/CD > Variables** に追加（保護・マスク推奨）。

---

### 6. AWS へのデプロイ例（ECS Fargate）

1. **事前準備**

   - ECR リポジトリ作成
   - タスク定義／サービス作成（最初は手動で OK）

2. **デプロイ用スクリプト断片**

   ```yaml
   deploy:
     stage: deploy
     image: amazon/aws-cli:2
     script:
       - aws ecr get-login-password --region ap-northeast-1 \
         | docker login --username AWS --password-stdin $ECR_REGISTRY
       - docker pull "$IMAGE_TAG"
       - docker tag "$IMAGE_TAG" "$ECR_IMAGE"
       - docker push "$ECR_IMAGE"
       - aws ecs update-service \
         --cluster MyCluster \
         --service MyService \
         --force-new-deployment
   ```

---

### 7. 運用の基本

| 項目                  | コマンド・手順                                                            | 推奨頻度         |
| --------------------- | ------------------------------------------------------------------------- | ---------------- |
| GitLab バックアップ   | `sudo gitlab-backup create`                                               | 1 日 1 回 (cron) |
| Runner バージョン UP  | `docker pull gitlab/gitlab-runner:alpine && docker restart gitlab-runner` | 月次             |
| GitLab アップグレード | `docker pull gitlab/gitlab-ee:<新Ver>` → `docker compose up -d`           | 四半期           |
| ログ確認              | `docker logs gitlab [-f]` / `gitlab-runner`                               | 随時             |
| ディスク使用率        | `df -h /srv/gitlab`                                                       | 週次             |

---

### 8. よくあるトラブルと対処

| 症状                      | 原因候補                         | 対処                                                           |
| ------------------------- | -------------------------------- | -------------------------------------------------------------- |
| **Runner が `unhealthy`** | DinD のキャッシュ肥大化          | `docker system prune -af` を Runner 内で実行                   |
| **パイプライン timeout**  | コンテナイメージが巨大           | マルチステージビルド & キャッシュ利用                          |
| **ディスク不足**          | リポジトリ増加・バックアップ保持 | EBS サイズを拡張 (`growpart` → `resize2fs`)                    |
| **証明書エラー**          | 自己署名 TLS                     | Let’s Encrypt + `gitlab.rb` で `letsencrypt['enabled'] = true` |

---

## おわりに

このガイドは **“まず動かす”** ことにフォーカスしています。
運用を続ける中で、以下の追加強化を検討してください。

1. **監視**: Prometheus/Grafana（GitLab に同梱）でメトリクス可視化
2. **SAST / DAST**: GitLab 付属のセキュリティジョブを有効化
3. **オートスケール Runner**: スポット EC2 または Fargate Spot

最小構成が安定したら段階的にスケールアップし、Azure DevOps からの完全移行を目指しましょう。

## GitLab CI/CD 開始・運用ガイド（ミニマム版・外部リポジトリ連携対応）

> **ゴール**
> 外部 Git リポジトリ（GitHub／Azure DevOps など）の
>
> - **プッシュ** ⇒ 5 分以内に GitLab へ Pull Mirroring
> - **プルリクエスト** ⇒ GitLab 側で自動生成された _Merge Request_ をトリガに **ビルド & テスト**
> - **main ブランチへの Push** のみ **デプロイ**
>
> までを 1 日で動かす。

---

### 1. はじめに

- 読者：Azure DevOps から GitLab（セルフホスト）へ最短移行したい人
- 手順：

  1. EC2 + Docker Compose で GitLab 本体を構築
  2. 同ホストに Docker-executor の GitLab Runner を登録
  3. **外部リポジトリを Pull Mirroring** → `.gitlab-ci.yml` をコミット
  4. Merge Request と main push で動く CI/CD を確認

---

### 2. 前提条件と準備

| 要素           | 推奨値                          |
| -------------- | ------------------------------- |
| EC2            | `t3.medium` / Ubuntu 22.04      |
| ポート         | 80 / 443 / 22                   |
| 永続ストレージ | `/srv/gitlab` 50 GB+            |
| ソフトウェア   | Docker 25.x / Docker Compose v2 |

セキュリティグループは 80/443/22 を自社 IP に制限することを推奨。

---

### 3. GitLab コンテナを立てる

（※手順は以前の `docker-compose.yml` と同一。省略可）

---

### 4. GitLab Runner を立てる

同ホストで Docker executor を登録（前回と同手順）。Runner Tag に `docker` を付与。

---

### 5. **外部リポジトリ Pull Mirroring 設定**

1. GitLab に空プロジェクトを作成
2. **Settings › Repository › Mirroring repositories**

   - **Repository URL**：外部リポジトリ HTTPS URL
   - **Mirror direction**：**Pull**
   - 認証：PAT（GitHub）/ Personal Access Token（Azure DevOps）
   - 「Mirror only protected branches」**OFF**

3. 保存 → 5 分おきに自動同期（セルフホストは cron 調整可）
4. **General › Merge Request** で

   - _Pipelines for external pull requests_ **ON**
   - _Pull-based mirror merges_ **ON**
     これで外部 PR ⇒ GitLab 側 Merge Request が生成される。

---

### 6. `.gitlab-ci.yml` ― PR でテスト、main でデプロイ

```yaml
stages:
  - build
  - test
  - deploy

variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

#========================================
# ジョブ共通ルール
#========================================
.default_rules: &rules
  rules:
    # Merge Request (外部 PR 含む) は常にビルド＆テスト
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: always

    # main ブランチへの push はビルド＆テスト＆デプロイ
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"'
      when: always

    # それ以外はスキップ
    - when: never

#----------------------------------------
build:
  stage: build
  image: docker:25
  services: [docker:25-dind]
  tags: [docker]
  <<: *rules
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
    - docker build -t "$IMAGE_TAG" .
    - docker push "$IMAGE_TAG"

#----------------------------------------
test:
  stage: test
  image: docker:25
  services: [docker:25-dind]
  tags: [docker]
  <<: *rules
  script:
    - docker run --rm "$IMAGE_TAG" pytest

#----------------------------------------
deploy:
  stage: deploy
  image: amazon/aws-cli:2
  tags: [docker]
  rules:
    # main ブランチの push のときだけデプロイ
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"'
      when: always
    - when: never
  script: |
    aws ecr get-login-password --region ap-northeast-1 \
      | docker login --username AWS --password-stdin $ECR_REGISTRY

    docker pull "$IMAGE_TAG"
    docker tag "$IMAGE_TAG" "$ECR_IMAGE"
    docker push "$ECR_IMAGE"

    aws ecs update-service \
      --cluster $ECS_CLUSTER \
      --service $ECS_SERVICE \
      --force-new-deployment
  environment:
    name: production
    url: http://$TARGET_HOST
```

**変数は GitLab › Settings › CI/CD › Variables** で保護・マスク付きで登録。

- `CI_REGISTRY_USER`, `CI_REGISTRY_PASSWORD`
- `ECR_REGISTRY`, `ECR_IMAGE`, `ECS_CLUSTER`, `ECS_SERVICE`, `TARGET_HOST`

---

### 7. 動作確認フロー

1. **main ブランチ**に push

   - GitLab が 5 分以内に同期 → Build → Test → **Deploy 実行**

2. **外部 PR** を作成

   - GitLab に MR が自動生成 → Build → Test のみ実行（Deploy なし）

3. PR マージ → main に push → デプロイ

---

### 8. 運用のポイント（抜粋）

| 項目                            | 推奨頻度  | 補足                                                                  |
| ------------------------------- | --------- | --------------------------------------------------------------------- |
| GitLab & Runner アップグレード  | 月次      | `docker pull` → `docker compose up -d`                                |
| Backup (`gitlab-backup create`) | 1 日 1 回 | S3 など外部に退避                                                     |
| ミラー遅延監視                  | 随時      | Pull Mirror が失敗すると `repository_mirror_lapse` メトリクスが上がる |
| Runner ディスク掃除             | 週次      | `docker system prune -af`                                             |

---

## まとめ

- **Pull Mirroring** で外部リポジトリを GitLab に自動取り込み
- **`rules:`** で

  - `merge_request_event` → Build & Test
  - `main` の push → Build & Test & Deploy

- デプロイ例は ECR + ECS Fargate に push／強制再デプロイ

この最小構成で、**PR 単位の品質ゲート**と **main ブランチ自動本番反映**を両立できます。

### インスタンス増やしたくない PoC 用ミニマム構成案

| 目的                         | どうやるか                                                                            | 必要インスタンス数 |
| ---------------------------- | ------------------------------------------------------------------------------------- | ------------------ |
| **① Git リポジトリ & CI/CD** | GitLab + GitLab Runner を同じ EC2 上の Docker コンテナで動かす                        | **1 台**           |
| **② デプロイ先 Web アプリ**  | 静的サイト (Nginx コンテナ) を **同じ EC2** に配置し、パイプラインで再ビルド & 再起動 | 同上               |
| **③ 外部アクセス**           | GitLab はポート **8181/8443**、公開サイトはポート **80** に割り当て                   | –                  |

> **まとめ：EC2 ×1（t3.medium 以上）で完結**
> GitLab 用と公開サイト用でコンテナを分けるだけ。ランナーも GitLab コンテナと同居させ、Docker executor で自分自身を操作します。

---

## 1. アーキテクチャ図（ざっくり）

```
┌────────────── EC2 (t3.medium) ──────────────┐
│                                              │
│  ┌─────────────────────────────┐             │
│  │ gitlab (ports 8181,8443,22) │             │
│  └─────────────────────────────┘             │
│  ┌──────────────────────────────┐            │
│  │ gitlab-runner (Docker exec)  │  ⇽──┐ CI   │
│  └──────────────────────────────┘      │     │
│  ┌──────────────────────────────┐      │     │
│  │ nginx-app (port 80)          │  ◀───┘ CD  │
│  └──────────────────────────────┘            │
│                                              │
└──────────────────────────────────────────────┘
```

- **GitLab**：WebUI は `http://<IP>:8181`
- **公開サイト**：ブラウザでは普通に `http://<IP>/`
- **Runner**：Docker executor（DinD 不要。ホスト Docker をそのまま利用）

---

## 2. Docker Compose（例）

```yaml
version: "3.9"
services:
  gitlab:
    image: gitlab/gitlab-ee:17.0.1-ee.0 # CE でも可
    hostname: gitlab.local
    restart: always
    ports:
      - "8181:80" # GitLab HTTP
      - "8443:443" # GitLab HTTPS
      - "2222:22" # GitLab SSH
    volumes:
      - ./gitlab/config:/etc/gitlab
      - ./gitlab/logs:/var/log/gitlab
      - ./gitlab/data:/var/opt/gitlab
    environment:
      GITLAB_ROOT_PASSWORD: "ChangeMeStrong!"

  gitlab-runner:
    image: gitlab/gitlab-runner:alpine
    depends_on: [gitlab]
    volumes:
      - ./runner/config:/etc/gitlab-runner
      - /var/run/docker.sock:/var/run/docker.sock
    restart: always

  nginx-app: # 公開用の超シンプル静的サイト
    build: ./site # site/Dockerfile で `FROM nginx:alpine` など
    image: my-nginx-site # CI でこのタグを上書き
    ports:
      - "80:80"
    restart: always
```

> **注** : まずは手動起動 (`docker compose up -d`) でページが見える状態を作っておく。

---

## 3. `.gitlab-ci.yml` — PR はビルド/テスト、main でデプロイ

```yaml
stages: [build, test, deploy]

variables:
  LOCAL_IMAGE: my-nginx-site:$CI_COMMIT_SHORT_SHA

.default_rules: &rules
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: always
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"'
      when: always
    - when: never

build:
  stage: build
  tags: [docker]
  image: docker:25
  services: [docker:25-dind]
  <<: *rules
  script:
    - docker build -t "$LOCAL_IMAGE" site
    - docker save "$LOCAL_IMAGE" > image.tar
  artifacts:
    paths: [image.tar]

test:
  stage: test
  tags: [docker]
  image: docker:25
  services: [docker:25-dind]
  <<: *rules
  script:
    - docker load -i image.tar
    - docker run --rm "$LOCAL_IMAGE" /bin/sh -c 'test -f /usr/share/nginx/html/index.html'

deploy:
  stage: deploy
  tags: [docker]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"'
      when: always
    - when: never
  script: |
    echo "Loading image to host Docker..."
    docker load -i image.tar
    echo "Stopping old container & starting new one"
    docker compose stop nginx-app
    docker compose rm -f nginx-app
    docker compose up -d nginx-app
  environment:
    name: production
    url: http://$CI_SERVER_HOST
```

**ポイント**

1. **DinD** でイメージをビルド → `image.tar` として持ち回し
2. `deploy` ジョブは **同一ホスト**の Docker にロードし、`docker compose up -d` で nginx-app を置き換え
3. Runner がホスト Docker を共有しているので **追加インスタンス不要**

---

## 4. セットアップ手順まとめ

1. **EC2 起動**（t3.medium、ポート 80/8181/8443/2222/22 を開ける）
2. Docker & Docker Compose インストール
3. 上記 `docker-compose.yml` を置いて `docker compose up -d`
4. Runner を `docker exec -it gitlab-runner gitlab-runner register` で登録

   - GitLab URL = `http://localhost:8181`
   - Executor = `docker`
   - Tag = `docker`

5. 外部リポジトリを **Pull Mirroring** 設定
6. サンプル `site/` に `index.html` を置き、`.gitlab-ci.yml` を main に push

---

## 5. 見え方（ユーザ視点）

- **`http://<EC2_IP>/`**

  - 最初は “It works!” などの静的ページ
  - PR を出すとパイプラインが Build/Test だけ走る
  - main へマージすると 1 ～ 2 分後にページが自動更新（デプロイ完了）

- **`http://<EC2_IP>:8181/`**

  - GitLab WebUI。PR 状況やパイプラインログ確認可能

---

## 6. 省リソースのコツ

| テクニック                                                               | 効果                                |
| ------------------------------------------------------------------------ | ----------------------------------- |
| GitLab の `gitlab.rb` で `puma['worker_processes'] = 2` などリソース削減 | メモリ圧縮                          |
| Runner は **single-build concurrency**（デフォルト 1 並列）              | CPU 逼迫を防ぐ                      |
| nginx-app は静的ファイルのみ                                             | イメージサイズ ≒ 20 MB でビルド高速 |

---

### これで **EC2 ×1 台** で「PR → Build/Test、main → Deploy」まで一通り体験できます。

運用がこなれてきたら、

- GitLab を別インスタンスに分離、
- ECR/ECS へ移行、
- HTTPS 化（Let’s Encrypt）
  等を段階的に追加すれば OK です。

### 「nginx-app って事前準備が要るの？」に対するシンプル化プラン

> **結論：** > *Dockerfile ビルドすら要らない* ─  公式 **nginx\:alpine** をそのまま使い、
> **ホスト側ディレクトリをマウント**して静的ファイルを配信する構成に変えれば、準備は
> `index.html` を Git リポジトリに置くだけで OK です。

---

## 1. もっと簡単な `docker-compose.yml`

```yaml
version: "3.9"
services:
  gitlab: # ここは前回と同じ（省略）

  gitlab-runner: # ここも同じ（省略）

  nginx-app:
    image: nginx:alpine # 公式イメージをそのまま
    volumes:
      # ↓ リポジトリ内の site/ をホスト経由で公開
      - ./site:/usr/share/nginx/html:ro
    ports:
      - "80:80"
    restart: always
```

- **ポイント**

  - `build:` を削除し **Dockerfile いらず**。
  - `volumes:` でリポジトリ直下の `site/` フォルダを **読み取り専用 (ro)** でマウント。
  - `site/index.html` さえコミットすればページが表示される。

---

## 2. 事前にやること — たった 2 ステップ

1. **リポジトリにフォルダを作る**

   ```
   mkdir site
   echo '<h1>Hello GitLab CI/CD!</h1>' > site/index.html
   git add site/index.html
   git commit -m "add placeholder page"
   ```

2. EC2 で `docker compose up -d` を再実行（サービス再作成）。

これでブラウザから `http://<EC2 IP>/` にアクセスすると、`index.html` がそのまま表示されます。

---

## 3. `.gitlab-ci.yml` も超ライト版へ

静的ファイルだけなら「ビルド」も「イメージ push」も不要。
**アーティファクトをホストにコピー**して Nginx を再起動するだけで完結します。

```yaml
stages: [test, deploy]

.default_rules: &rules
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: always
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"'
      when: always
    - when: never

#--------------------------
test:
  stage: test
  <<: *rules
  image: alpine
  script:
    - echo "🧪 静的ファイル構文チェック"
    - test -f site/index.html # とりあえず存在確認だけ

#--------------------------
deploy:
  stage: deploy
  rules:
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"'
      when: always
    - when: never
  tags: [docker]
  script: |
    echo "🚚 サイトをホストへ同期"
    rsync -av --delete site/ /srv/gitlab-poc/site/

    echo "🔄 nginx をリロード"
    docker compose exec nginx-app nginx -s reload
  environment:
    name: production
    url: http://$CI_SERVER_HOST
```

### 変更点

| 従来                                                        | 簡易版            |
| ----------------------------------------------------------- | ----------------- |
| Docker in Docker でイメージ build / push                    | **不要**          |
| Runner → ECR → ECS 更新                                     | **不要**          |
| deploy ジョブはホスト上の **`rsync` + `nginx reload`** だけ | シェル 2 行で済む |

> **ホスト側公開ディレクトリ**
>
> - 上の例では `/srv/gitlab-poc/site/` にしています。<br> > `docker-compose.yml` の `volumes:` を
>   `- /srv/gitlab-poc/site:/usr/share/nginx/html:ro`
>   に揃えておけば、`rsync` 先と 1:1 対応します。

---

## 4. インスタンス数 & リソースまとめ

| 構成要素      | 動かし方                    | CPU/Mem 負荷    | 追加インスタンス |
| ------------- | --------------------------- | --------------- | ---------------- |
| GitLab 本体   | `gitlab` コンテナ           | 中程度 (1–2 GB) | なし             |
| GitLab Runner | 同ホスト、`docker` executor | 軽い            | なし             |
| 公開サイト    | `nginx` コンテナ            | 非常に軽い      | なし             |

> **合計 EC2 台数：1 台**（t3.medium で十分）
> **料金感:** 東京リージョン on-demand で \~0.05 USD/時 ≒ 4 USD/日。
> PoC のあいだは _7 日で < 30 USD_ に収まる規模です。

---

### これで「

- PR ＝ Build/Test（静的ファイル Lint 程度）
- main Push ＝ rsync → nginx reload で即反映
  」を **EC2 ×1 台だけ** で確認できます。

将来コンテナアプリやマイクロサービスを載せ替えたくなったら、

- `nginx` を ECS / EKS へ移行
- `rsync` をイメージ Push + `ecs update-service` に変更
  とステップアップすれば OK です。

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

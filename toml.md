### “`config.toml` が存在するのに空”――原因と対処 3 ステップ

（**結論 ▶︎ Runner がまだ “登録” されていないだけ** です）

---

## 1 ️⃣ まず確認：Runner は本当に GitLab と紐づいている？

```bash
docker exec -it gitlab-runner gitlab-runner verify
```

- **「Verifying runner… is valid」** と出れば登録済み
  → ⬇︎ へ進んで中身だけ書き足す
- **「ERROR: Runner not found」** と出たら未登録
  → ❷ で新規登録すれば `config.toml` が自動生成されます

---

## 2 ️⃣ 未登録なら `gitlab-runner register` を実行

```bash
docker exec -it gitlab-runner gitlab-runner register
# GitLab URL →  http://<EC2_IP>:8181
# Token      →  GitLab 画面 > Settings > CI/CD > Runners の Registration token
# Description → 任意（例: ec2-docker-runner）
# Tags       → docker
# Executor   → docker
```

> **登録が完了すると** `/etc/gitlab-runner/config.toml` が生成され、
> バインドマウントしているので **ホストの `/srv/gitlab-runner/config.toml`** にも即反映されます。
> 中を開くと、自動で `[runners]` ブロックが書き込まれているはずです。

---

## 3 ️⃣ 目的の `volumes` 行だけ追記して保存

```toml
[[runners]]
  executor = "docker"
  [runners.docker]
    privileged = true
    volumes = [
      "/var/run/docker.sock:/var/run/docker.sock",
      "/srv/site:/srv/site"      # ← ★ これを追加
    ]
```

```bash
sudo nano /srv/gitlab-runner/config.toml   # 追記後 Ctrl+O → Enter → Ctrl+X
docker restart gitlab-runner               # 設定を読み込みなおす
```

---

### もし **verify では OK なのにファイルだけ空** なら…

1. **違うファイルを見ている**

   - `docker exec gitlab-runner ls -l /etc/gitlab-runner`
     真の `config.toml` がサブディレクトリ (`/etc/gitlab-runner/config/config.toml`) にある場合があります。
     → そのパスをホストへバインドし直す

     ```bash
     docker rm -f gitlab-runner
     docker run -d --name gitlab-runner --restart always \
       -v /var/run/docker.sock:/var/run/docker.sock \
       -v /srv/gitlab-runner:/etc/gitlab-runner/config \
       gitlab/gitlab-runner:alpine
     ```

2. **ファイルが zero byte で壊れている**

   - いったん削除して Runner を再登録すると再生成されます。

     ```bash
     sudo rm /srv/gitlab-runner/config.toml
     docker restart gitlab-runner
     # → verify でエラーになるはず → register し直し
     ```

---

## ✅ これで `config.toml` に内容が入り、追記も完了

- **`cat /srv/gitlab-runner/config.toml`** で追記行が確認できる
- GitLab でパイプラインを走らせ、`deploy` ジョブが `/srv/site` にアクセスできれば成功です。

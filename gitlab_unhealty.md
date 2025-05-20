`STATUS` 列に `Up 5 minutes (unhealthy)` と出るのは、**コンテナ自体は起動しているものの、Docker が実行している “ヘルスチェック” が連続で失敗している**ことを示します。
(スペルミスで `unhealty` と見えることもありますが、意味は同じです)

---

## しくみ ─ HEALTHCHECK とは？

1. **Dockerfile／docker-compose.yml に `HEALTHCHECK` 命令**が書かれている

   - 例）`HEALTHCHECK CMD curl -f http://localhost:80 || exit 1`

2. コンテナ起動後、指定間隔で上記コマンドを実行
3. 既定で **3 回連続で失敗すると `unhealthy` に遷移**

   - 成功すれば `healthy`、失敗が続けば `unhealthy`

---

## よくある原因

| 主な原因                          | 具体例                                                     | 典型的な GitLab コンテナの場合        |
| --------------------------------- | ---------------------------------------------------------- | ------------------------------------- |
| アプリ起動が遅い / 初期設定未完了 | DB マイグレーション中、初期シード中                        | `gitlab-rails ...` が長時間走っている |
| リソース不足                      | メモリ 4 GB 未満、CPU 過負荷                               | `Cannot allocate memory`              |
| ポート・URL 不一致                | `HEALTHCHECK` が叩くポートを閉じた / `external_url` が違う | 443 を設定したのに 80 をチェック      |
| 証明書 or 認証失敗                | HTTPS の自己署名証明書で `curl -f` が失敗                  | `SSL certificate problem`             |
| 設定ミス・権限エラー              | マウント先の権限不足、設定ファイル不足                     | `/var/opt/gitlab` の権限 700 など     |

---

## 調べ方 (手早く状況を掴む)

```bash
# ヘルスチェックの詳細ログ
docker inspect --format '{{json .State.Health}}' <コンテナ名> | jq

# 直近の 100 行のログ
docker logs --tail 100 <コンテナ名>
```

特に `Log` に出る `ExitCode`, `Output`, `Error` を確認すると、
失敗しているコマンドとエラー内容が丸わかりです。

---

## 直すヒント

1. **まずはログ確認** — 何が失敗しているか特定
2. **リソースを増やす／待つ**

   - GitLab など重量級は初回起動に 5–10 分以上かかることも

3. **`docker exec` で中に入り、手動でコマンド実行**

   ```bash
   docker exec -it <コンテナ名> /bin/bash
   # 例: GitLab
   gitlab-ctl status
   gitlab-ctl tail
   ```

4. **設定を見直す**

   - `docker-compose.yml` の環境変数 (`GITLAB_OMNIBUS_CONFIG` 等)
   - ボリュームのパーミッション (`chown -R 1000:1000`)

5. **ヘルスチェックを一時的に無効化**することも可能

   - 検証中だけ `HEALTHCHECK NONE` を指定して原因切り分け

---

### もし GitLab でずっと `unhealthy` のままなら…

- **メモリ不足 (要 4 GB 以上)**
- **外部 URL (`external_url`) と実際のポートが不一致**
- **初期 `gitlab-ctl reconfigure` が失敗している**

これらが鉄板ポイントです。ログで該当メッセージがないか探してみてください。

---

💡 **次の一手**
`docker inspect ...Health` の出力か `docker logs` のエラー行を共有してもらえれば、よりピンポイントで原因を絞り込めます！

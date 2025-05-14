**`status: restarting` のままになっているということは、GitLab コンテナがクラッシュして自動再起動を繰り返している状態**です。

---

## 🛠 次にやるべきこと

### ✅ 1. ログを確認する（最重要）

```bash
docker logs gitlab
```

このログに、**再起動の原因となっているエラーメッセージ**が必ず出ています。

> 🔍 特に注目するのは、下のようなキーワード：
>
> - `Error`
> - `Permission denied`
> - `Address already in use`
> - `Cannot bind`
> - `nginx: [emerg]`

---

## 🎯 よくある原因と対処法

| 原因例                                | ログ例                        | 解決策                                                |
| ------------------------------------- | ----------------------------- | ----------------------------------------------------- |
| ポート競合（例：`80:80`が他で使用中） | `bind() to 0.0.0.0:80 failed` | 別のプロセスが使っていないか確認： `sudo lsof -i :80` |
| ディレクトリの権限ミス                | `Permission denied`           | `sudo chown -R 1000:1000 /srv/gitlab` で修正          |
| external_url 設定ミス                 | `nginx: invalid host`         | URL が `http://your-ip` になっているか確認            |
| 古い設定が壊れている                  | `reconfigure failed`          | `docker volume rm` でまっさらにする or 設定見直し     |

---

## ✅ 2. まずは権限を再設定してみましょう

以下のコマンドで、GitLab コンテナが使うフォルダの権限を修正します：

```bash
sudo chown -R 1000:1000 /srv/gitlab
```

これは、GitLab の内部ユーザー（UID 1000）が正しく読み書きできるようにするためです。

---

## ✅ 3. external_url の再確認

docker-compose.yml の該当部分：

```yaml
external_url 'http://<あなたのEC2のIPアドレス>'
```

⚠️ ここに `https://` と書いていた場合、証明書の準備がないと失敗します。**必ず `http://` にしてください。**

---

## ✅ 4. 再構成を試す

設定が変わった場合は、再構成を試みます：

```bash
docker exec -it gitlab gitlab-ctl reconfigure
```

これでうまくいけば、GitLab が初期化されて `status` が `run:` に変わるはずです。

---

## ✅ 5. どうしても起動しない場合の最終手段

```bash
docker-compose down -v
```

これでコンテナとボリューム（保存データ）を削除し、まっさらな状態に戻します（初期化されますのでご注意ください）。

---

## 🔁 ログ確認して一緒に原因調査します！

もしよろしければ、以下のコマンドで出てくるログの**最後の 20〜30 行くらい**を貼ってください：

```bash
docker logs gitlab --tail 30
```

それを見れば、ピンポイントで「なぜ GitLab が再起動を繰り返しているのか」が特定できます！

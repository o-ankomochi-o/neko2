現在 GitLab CE を Docker で動かしていて、EE（Enterprise Edition）へ切り替えたい場合、**ボリュームはそのままに、イメージだけを `gitlab/gitlab-ee` に変更**するだけで OK です。データや設定は引き継がれます。

---

## ✅ EE 版への切り替え手順（Docker 編）

以下の手順で `gitlab-ce` → `gitlab-ee` に安全に移行できます。

---

### 🔧 手順

1. **現在の GitLab を停止**

```bash
docker stop gitlab
docker rm gitlab
```

※ `docker-compose down` でも可（compose を使ってる場合）

---

2. **docker-compose.yml の `image` を以下のように修正**

```yaml
image: gitlab/gitlab-ee:latest
```

---

3. **EE イメージを取得（pull）**

```bash
docker pull gitlab/gitlab-ee:latest
```

---

4. **再起動（compose or docker run）**

```bash
docker-compose up -d
```

または（compose を使わない場合）：

```bash
docker run --detach \
  --hostname gitlab.example.com \
  --publish 443:443 --publish 80:80 --publish 22:22 \
  --name gitlab \
  --restart always \
  --shm-size 256m \
  --env GITLAB_ROOT_PASSWORD=ChangeMeStrong! \
  --volume /srv/gitlab/config:/etc/gitlab \
  --volume /srv/gitlab/logs:/var/log/gitlab \
  --volume /srv/gitlab/data:/var/opt/gitlab \
  gitlab/gitlab-ee:latest
```

---

## ✅ 注意点

| 項目                                       | 内容                                                                                                                             |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| **データ保持**                             | `/srv/gitlab/` 以下のボリュームを再利用する限り、**ユーザー・リポジトリ・設定はそのまま保持**されます。                          |
| **ライセンス未登録の EE は Free 機能のみ** | 見た目は変わりません。ライセンス登録をしていない限り、**機能は CE と同等**です。                                                 |
| **ライセンス登録したい場合**               | EE に切り替えたあと、GitLab の管理画面 → **Admin Area → License** からライセンスキーを登録できます（購入または無料トライアル）。 |

---

## ✅ 反映確認

ログイン後に「Admin Area（管理者エリア）」へ行き、下記で確認可能：

- **GitLab Enterprise Edition** と表示されるか？
- 「ライセンスを有効化」ボタンがあるか？

---

## 🚀 まとめ

```diff
- image: gitlab/gitlab-ce:latest
+ image: gitlab/gitlab-ee:latest
```

↑ この 1 行変更＋再起動で、CE → EE（Free）の切り替えは完了です。

---

ご希望であれば、`ライセンスの登録画面までの画面遷移` や `動作確認項目チェックリスト` もご案内可能です！

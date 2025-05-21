以下では、\*\*「なぜそのコマンドを打つのか」「その項目は何を意味するのか」\*\*を、Docker や CI/CD が初めての方にもわかるように順を追って説明します。

---

## 0. 全体像をざっくりつかむ

| 役者              | 役割                                                                           | 例えるなら…    |
| ----------------- | ------------------------------------------------------------------------------ | -------------- |
| **GitLab 本体**   | リポジトリや Issue、パイプライン設定（`.gitlab-ci.yml`）を保管する「司令塔」   | “校舎”         |
| **GitLab Runner** | 司令塔からの指示を受け取り、テストやデプロイのジョブを実際に実行する「働き手」 | “部活動の部員” |
| **Docker**        | Runner を気軽に作ったり壊したりできる「コンテナの箱」                          | “部室”         |

> **ポイント**:
> _GitLab_ ＝ 管理画面／設定を置くところ、
> _Runner_ ＝ 実際に CI/CD 処理を回すエンジン。
> どちらも EC2 上で Docker コンテナとして動かせるので、物理サーバーを汚さず管理できます。

---

## 1. GitLab Runner を Docker で起動するコマンド

```bash
docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```

| パラメータ                                        | ざっくり説明                                                                                                                           |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `docker run -d`                                   | バックグラウンド（デタッチ）でコンテナを起動                                                                                           |
| `--name gitlab-runner`                            | コンテナにつけるわかりやすい名前                                                                                                       |
| `--restart always`                                | EC2 が再起動しても Runner が自動起動する保険                                                                                           |
| `-v /srv/gitlab-runner/config:/etc/gitlab-runner` | **設定ファイルをホスト側に保存**しておくための共有フォルダ（コンテナを作り直しても設定が消えない）                                     |
| `-v /var/run/docker.sock:/var/run/docker.sock`    | **Docker in Docker** を実現する鍵。Runner が「ホストの Docker エンジン」を使って新しいコンテナを起動できるようにするパイプ（ソケット） |
| `gitlab/gitlab-runner:latest`                     | 公式イメージの最新タグ                                                                                                                 |

> **なぜ Docker がおすすめ？**
>
> - 失敗してもコンテナを消してやり直すだけ。
> - バージョンアップが `docker pull` だけで終わる。
> - 複数 Runner が欲しくなったらコピペで増設可能。

---

## 2. Runner を GitLab に登録する

```bash
docker exec -it gitlab-runner gitlab-runner register
```

- `docker exec -it` は **既に動いているコンテナの中に入って**コマンドを実行する魔法のドア。
- `gitlab-runner register` は Runner を GitLab 本体にひも付ける初期手続きです。

### 対話プロンプトの意味

| 質問                     | 詳細                                                                                                                                       |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **GitLab instance URL**  | あなたの GitLab サーバーの URL。`http://<EC2のIP>` や `https://gitlab.example.com` など                                                    |
| **registration token**   | GitLab の **プロジェクト > Settings > CI/CD > Runners** にある文字列。これを渡すと「この Runner はこのプロジェクト専用ですよ」と認識される |
| **description**          | Web 画面に表示される Runner のニックネーム。複数台あるとき識別に便利（例: `docker-runner-tokyo`）                                          |
| **tags**                 | `.gitlab-ci.yml` でジョブを渡すときの「宛先ラベル」。ジョブ側で `tags: ["docker"]` と書くと、この Runner だけが実行候補になる              |
| **executor**             | Runner がジョブをどう実行するかの方式。**`docker`** を選ぶと、ジョブごとに新しいコンテナを作って汚れを残さない仕組み                       |
| **default Docker image** | `executor` が `docker` の場合、「ジョブで何も指定されていないとき使うベース OS」。Python が欲しければ `python:3.10` など                   |

> **ワンポイント**:
> チームで複数言語を使うなら、ジョブ側で `image:` を個別指定すれば OK。ここは「とりあえず動くもの」を入れておくだけで十分です。

---

## 3. 登録できたかを Web UI で確認

1. GitLab で対象プロジェクトを開く
2. `Settings` → `CI/CD` → **Runners** セクションへ
3. **"Active Runners"** に先ほどの description（例: `docker-runner`）が表示 → **✔️ 完了！**

もし表示されない場合は…

| 症状                    | チェックポイント                                             |
| ----------------------- | ------------------------------------------------------------ |
| Runner が Offline 表示  | コンテナが落ちていないか `docker ps` で確認                  |
| Not registered          | **トークン** を間違えていないか再登録                        |
| ジョブが Pending のまま | `.gitlab-ci.yml` の `tags:` が Runner の tags と合っているか |

---

## 4. “ポート 80 / 443 が開いている” とは？

- **ポート 80 (HTTP)** と **443 (HTTPS)** は「Web サイトの玄関口」。
- EC2 の **Security Group** でこのポートが **0.0.0.0/0** へ開放されていないと、外から GitLab UI にアクセスできず Runner も登録できません。
- HTTPS 化（443）すると **TLS 証明書** が必要。Let’s Encrypt + Nginx Proxy などは後で拡張可。

---

## 5. もっと学びたい方へ “次の一手”

| 次のステップ                          | ざっくりメリット                                           |
| ------------------------------------- | ---------------------------------------------------------- |
| **Volume を暗号化 / IAM 連携**        | 秘密情報の漏えいリスク低減                                 |
| **Cache や artifacts 保存先を S3 に** | ビルド成果物をクラウドに保管し、コンテナを軽量化           |
| **Auto-Scaling Runner**               | ジョブ数に応じて Runner コンテナを自動増減し、コスト最適化 |

---

### まとめ

- **GitLab Runner は「CI/CD の実行者」**。Docker 化で管理がラク。
- **register コマンド**で GitLab 本体と“名刺交換”させる。
- Web UI に Runner が表示＝準備完了。あとは `.gitlab-ci.yml` を書けば自動テスト／デプロイが走り出します！

これで、「なぜこの操作をするの？」という疑問がクリアになれば幸いです。🤖💨

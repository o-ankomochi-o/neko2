承知しました！初心者向けに、**各コマンドの意味や背景も丁寧に解説**しながら、もう一度ステップバイステップで説明します。Windows を使っている場合でも、EC2 に接続して Linux 上で操作する形になります。

---

## 🧭 全体の流れ

1. EC2 インスタンスを立てる（AWS マネジメントコンソール）
2. EC2 に SSH 接続する（Windows なら TeraTerm や PowerShell を使う）
3. GitLab を動かすために Docker 環境を作る
4. GitLab 用の設定ファイルを作成する
5. GitLab を起動してアクセス確認

---

## 🔐 ステップ 1: EC2 に SSH 接続（Windows から）

### 🔸 使うもの

- `.pem`ファイル（EC2 作成時にダウンロードした秘密鍵）
- PowerShell もしくは TeraTerm（SSH ソフト）

### 🔸 例（PowerShell）での接続

```bash
ssh -i C:\path\to\your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

- `-i` は秘密鍵を指定するオプション
- `ubuntu@...` はユーザー名 + IP アドレスです（Amazon Linux の場合は `ec2-user`）

---

## 🐳 ステップ 2: Docker と Docker Compose のインストール

```bash
sudo apt update
```

👉 パッケージの最新情報を取得（パッケージ管理の仕組みを最新化）

```bash
sudo apt install -y docker.io docker-compose
```

👉 Docker 本体と、複数のコンテナをまとめて管理する`docker-compose`をインストール
※ `-y` は「yes（全部インストール OK）」を自動で答えるオプション

```bash
sudo usermod -aG docker $USER
```

👉 `docker`コマンドを sudo なしで使えるように、ユーザーを docker グループに追加

```bash
newgrp docker
```

👉 上記の設定をすぐ反映するためのコマンド（ログアウトしなくても OK にする）

---

## 📁 ステップ 3: GitLab 用フォルダの作成

```bash
sudo mkdir -p /srv/gitlab/config /srv/gitlab/logs /srv/gitlab/data
```

👉 `-p` は必要なディレクトリを一気に作成。GitLab が設定・ログ・データを保存する場所を用意しています。

```bash
sudo chown -R 1000:1000 /srv/gitlab
```

👉 GitLab のコンテナが内部で使うユーザー（UID:1000）にアクセス権限を与えています。

---

## 📦 ステップ 4: docker-compose.yml の作成

```bash
nano docker-compose.yml
```

👉 `nano` はテキストエディタです（ここで yml ファイルを作って内容を記述します）

記述する内容は以下のようにシンプル：

```yaml
version: "3"
services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    restart: always
    hostname: your-ec2-public-ip
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://your-ec2-public-ip'
    ports:
      - "80:80"
      - "443:443"
      - "22:22"
    volumes:
      - /srv/gitlab/config:/etc/gitlab
      - /srv/gitlab/logs:/var/log/gitlab
      - /srv/gitlab/data:/var/opt/gitlab
```

- `image`：GitLab の Docker イメージ名（`ce`は Community Edition）
- `ports`：外部と内部のポートをつなぐ（HTTP/HTTPS/SSH）
- `volumes`：データの保存先をコンテナの外にして永続化

💡 入力後は `Ctrl + O` → `Enter` → `Ctrl + X` で保存＆終了

---

## 🚀 ステップ 5: GitLab を起動する

```bash
docker-compose up -d
```

- `up`：Docker コンテナを起動
- `-d`：バックグラウンド（detached）で実行
  👉 実行後、GitLab のセットアップに数分かかります

---

## 🔑 ステップ 6: 初期パスワードの確認

```bash
docker exec -it $(docker ps -q --filter name=gitlab) grep 'Password:' /etc/gitlab/initial_root_password
```

👉 GitLab の管理者（root）アカウント用の初期パスワードを取得できます

---

## 🌐 ステップ 7: Web ブラウザでアクセス

ブラウザで以下にアクセス：

```
http://<EC2のパブリックIP>
```

初回ログイン：

- ユーザー名：`root`
- パスワード：さっき確認したもの

---

## 🔁 次に知っておくと良いこと

- GitLab のアップデート：`docker pull`して `docker-compose up -d` で更新
- バックアップ＆リストアの方法
- 独自ドメイン + HTTPS（Let’s Encrypt）も設定可能

---

他にも「GitLab Runner を EC2 に登録したい」「オンプレミスに移行したい」などあれば気軽に聞いてください！

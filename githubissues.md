## 付録 — GitLab Issues でカンバンボードをつくる

> **目的**：タスクの “見える化” と担当状況の把握を一元化し、Slack でのメンション漏れを防ぐ。

1. **ボードを作成**

   1. プロジェクト画面左メニュー **「Plan › Boards」** を開く。
   2. 右上 **「Create board」** → 任意のボード名（例：`開発タスク`）を入力し **「Create board」**。

2. **カラム（リスト）を追加**

   - 既定の **「Open」** は残し、右端の **「Add list」** で以下を追加するとシンプルなカンバンになる。

     | 推奨ラベル | 用途               |
     | ---------- | ------------------ |
     | `Doing`    | 着手中             |
     | `Review`   | MR レビュー待ち    |
     | `Done`     | 完了（マージ済み） |

   - それぞれ **「Label」** を選択して作成。ラベルが無ければその場で新規作成できる。

3. **Issue をボードに載せる**

   - **「＋」** ボタン → Issue タイトル入力で追加。
   - 既存 Issue はドラッグ＆ドロップでカラム間を移動。状態が変わるたびに `Open` → `Doing` → … の順でスライドさせる。

4. **担当者と期日を設定**

   - Issue 詳細右側 **「Assignee」** に担当開発者を追加。
   - **「Due date」** を入れておくとボード上で期限超過が赤表示され、見落とし防止に便利。

5. **Slack 連携（任意）**

   1. **「Settings › Integrations › Slack notifications」** で Webhook URL を登録。
   2. `Issue events` と `Merge request events` にチェック → 進捗変更を自動ポスト。

> **運用 Tips**
>
> - **WIP 制限**：`Doing` カラムに枚数制限（例：3）を設けて “やりかけタスク” の溜め込みを防ぐ。
> - **フィルタ**：ボード右上の `Filter` で `assignee:=me` を選ぶと自分の担当だけに絞れる。
> - **スプリント管理**：ラベルに `Sprint-2025W22` のような“時期タグ”を付け、ボードをラベルで切り替えるとバーンダウンの代用になる。

この 5 ステップだけで **“Slack 通知付きカンバン”** が完成し、タスクの受け渡し・進捗共有を GitLab 内で完結できます。
GitLab の Issue ボードで「New list」を追加する際に、ラベルが選択できない場合、以下の手順でラベルを作成し、ボードにリストを追加することができます。

---

## 🛠️ ラベルの作成手順

1. **プロジェクトのラベル管理画面にアクセス**:

   - 左サイドバーで「**Manage › Labels**」を選択します。([apps.risksciences.ucla.edu][1])

2. **新しいラベルを作成**:

   - 「**New label**」ボタンをクリックします。
   - 以下の情報を入力します：

     - **Title**: 例）`Doing`
     - **Description**: （任意）ラベルの説明
     - **Color**: （任意）ラベルの色

   - 「**Create label**」をクリックしてラベルを作成します。

   同様の手順で、`Review`、`Done`などのラベルも作成できます。

---

## 🧱 ボードにリストを追加する手順

1. **Issue ボードにアクセス**:

   - 左サイドバーで「**Plan › Issue boards**」を選択します。

2. **新しいリストを追加**:

   - ボード上部の「**New list**」ボタンをクリックします。
   - 「**Label**」を選択し、先ほど作成したラベル（例：`Doing`）を選びます。
   - 「**Add to board**」をクリックしてリストを追加します。([docs.gitlab.com][2])

   同様の手順で、`Review`、`Done`などのリストも追加できます。

---

## 🔍 注意点

- ラベルが存在しない場合、リストに追加できません。
- ラベルは**プロジェクトレベル**で作成する必要があります。
- ラベルの作成には、プロジェクトでの**Reporter**以上の権限が必要です。

---

これらの手順に従うことで、`Open`と`Closed`以外のリストを Issue ボードに追加し、カンバン方式でのタスク管理が可能になります。

何か不明な点があれば、お気軽にお尋ねください。

[1]: https://apps.risksciences.ucla.edu/gitlab/help/user/project/labels.md?utm_source=chatgpt.com "Labels · Project · User · Help · GitLab"
[2]: https://docs.gitlab.com/user/project/issue_board/?utm_source=chatgpt.com "Issue boards | GitLab Docs"

GitLab の最新バージョンでは、既存の「Issue boards」を利用してカンバンボードを簡単に構築できます。以下に、既存ボードを活用したカンバン運用の手順をまとめました。

---

## ✅ 既存の Issue Board を活用したカンバンボードの構築手順

### 1. **「Plan › Issue boards」へアクセス**

- 左サイドバーの「Plan」から「Issue boards」を選択します。
- 既にボードが存在する場合、そのボードをそのまま活用できます。

### 2. **カラム（リスト）の追加**

- デフォルトで「Open」と「Closed」のリストが表示されています。
- 新しいリストを追加するには、ボード上部の「New list」ボタンをクリックします。
- 表示されるパネルで、以下のようなラベルを選択または新規作成して追加します：

  - `Doing`：作業中
  - `Review`：レビュー待ち
  - `Done`：完了

これらのリストは、対応するラベルが付与された Issue を自動的に表示します。

### 3. **Issue の作成と移動**

- 各リストの上部にある「Create new issue」ボタンをクリックして、新しい Issue を作成できます。
- 既存の Issue をドラッグ＆ドロップでリスト間を移動させることで、ラベルが自動的に更新され、Issue のステータスを変更できます。

### 4. **Issue の詳細設定**

- 各 Issue カードをクリックすると、右側に詳細パネルが表示されます。
- ここで以下の情報を設定できます：

  - **Assignee**：担当者の割り当て
  - **Due date**：期限の設定
  - **Labels**：ラベルの追加・変更
  - **Milestone**：マイルストーンの設定([GitLab ドキュメント][1])

### 5. **Slack との連携（任意）**

- プロジェクトの「Settings › Integrations › Slack notifications」から、Slack の Webhook URL を登録します。
- `Issue events`や`Merge request events`にチェックを入れることで、Issue やマージリクエストの更新が Slack に通知されます。

---

## 💡 運用のヒント

- **WIP 制限**：各リストに作業中の Issue 数の上限を設定することで、作業の集中と効率化を図れます。
- **フィルター機能**：ボード上部のフィルターを活用して、特定の担当者やラベル、マイルストーンなどで Issue を絞り込めます。
- **スプリント管理**：ラベルに`Sprint-2025W22`のような形式を使用し、スプリントごとに Issue を管理できます。

---

このように、既存の Issue Board を活用することで、GitLab 内で効率的なカンバンボードを構築し、タスクの可視化と進捗管理を一元化できます。

[1]: https://docs.gitlab.com/user/project/issue_board/?utm_source=chatgpt.com "Issue boards | GitLab Docs"

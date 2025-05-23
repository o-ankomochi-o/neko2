### GitLab CI/CD YAML vs. Azure Pipelines YAML ― “対応表＋書き換えガイド”

| 観点                 | **Azure DevOps (`azure-pipelines.yml`)**                  | **GitLab CI (`.gitlab-ci.yml`)**                                         | 置き換えのコツ                                                      |
| -------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------ | ------------------------------------------------------------------- |
| **トップ階層キー**   | `trigger`, `variables`, `stages`, `jobs`, `steps`, `pool` | `stages`, `variables`, `workflow`, _各 job_                              | **「jobs 階層が無い」**<br>⇒ GitLab は _job 名: …_ が最上位ブロック |
| **トリガ**           | `yaml<br>trigger:<br>  branches: [ main ]`                | `yaml<br>workflow:<br>  rules:<br>    - if: '$CI_COMMIT_BRANCH=="main"'` | 単純なブランチトリガなら `workflow:rules` に 1 行                   |
| **Stage 定義**       | `yaml<br>stages:<br>- Build<br>- Deploy`                  | `yaml<br>stages: [build, deploy]`                                        | 名前は自由。GitLab は配列で宣言                                     |
| **エージェント指定** | `yaml<br>pool: vmImage: ubuntu-latest`                    | `yaml<br>tags: [docker]` (job 内)                                        | Azure の“プール”≒ GitLab の “Runner タグ”                           |
| **変数**             | `yaml<br>variables:<br>  NODE_ENV: production`            | 同形でそのまま                                                           | 機密は `masked` / `protected` で GUI 追加                           |
| **ジョブ (job)**     | `yaml<br>- job: Build<br>  steps: …`                      | `yaml<br>build-job:<br>  stage: build<br>  script: …`                    | `job:` → “任意の job 名:” に改名                                    |
| **ステップ (step)**  | `script:`, `bash:`, `task:` 等                            | `script:` に Bash 列挙                                                   | 公式 Task はシェルスクリプトへ置き換え（例下）                      |
| **キャッシュ**       | `task: CacheBeta@0`                                       | `yaml<br>cache: {paths: [node_modules/]}`                                | キーは `$CI_COMMIT_REF_SLUG` などで分離                             |
| **アーティファクト** | `publish: $(Build.ArtifactStagingDirectory)`              | `yaml<br>artifacts: {paths: [build/]} `                                  | `expire_in:` を忘れず                                               |
| **テンプレート読込** | `template:` or `extends:`                                 | `include:` or `extends:`                                                 | パス指定でほぼ同じ概念                                              |
| **マトリクス**       | `strategy: matrix:`                                       | `parallel: matrix:`                                                      | キー名以外は似ている                                                |
| **環境 / デプロイ**  | `environment: 'prod'` + `deployment` ジョブ               | `environment: {name: production}` + 任意スクリプト                       | Manual/auto gate は `when: manual` 等で再現                         |

---

## 1. “Hello-World” を写経して比較

### Azure Pipelines

```yaml
trigger: [ main ]

stages:
- stage: Build
  jobs:
  - job: Build
    pool: vmImage: ubuntu-latest
    steps:
      - script: echo "build"
        displayName: Build step

- stage: Deploy
  dependsOn: Build
  jobs:
  - job: Deploy
    pool: vmImage: ubuntu-latest
    steps:
      - script: echo "deploy"
```

### GitLab CI に書き直す

```yaml
workflow:
  rules:
    - if: '$CI_COMMIT_BRANCH=="main"'

stages: [build, deploy]

build-job:
  stage: build
  tags: [docker] # ← Runner タグ
  script:
    - echo "build"

deploy-job:
  stage: deploy
  needs: [build-job] # Azure の dependsOn
  tags: [docker]
  script:
    - echo "deploy"
```

---

## 2. Azure “task:” を GitLab に移植するパターン集

| Azure Task                             | GitLab 置き換え例                                                                    |
| -------------------------------------- | ------------------------------------------------------------------------------------ |
| `task: NodeTool@0 inputs: version: 20` | `image: node:20-alpine` もしくは `n` でインストール                                  |
| `task: PublishBuildArtifacts@1`        | `artifacts: {paths: [build/], expire_in: 1 week}`                                    |
| `task: Docker@2` (build/push)          | `script: docker build … && docker push …` <br>認証は `$CI_REGISTRY`, `$CI_JOB_TOKEN` |

> **Tip**: Azure の Marketplace タスクはただのラッパー。実体は Bash/Powershell コマンドなので、その中身を `script:` に書けば動く。

---

## 3. 移行手順チェックリスト

1. **Runner タグ設計**

   - MS-hosted → Self-hosted Runner
   - Windows ビルドは別 Runner が必要

2. **シークレット移行**

   - Azure Library/KeyVault → GitLab CI/CD “Variables”
   - `protected` & `masked` を設定

3. **アーティファクト/イメージ保管**

   - Azure Artifacts → GitLab Packages / Container Registry

4. **パイプライン YAML 変換**

   - 段階的に：Stage → job → step を順に潰す
   - 変換表を参照しつつ `include:` テンプレで共通化

5. **環境・承認フロー**

   - Azure “Environments” の手動承認 → GitLab `when: manual` + 保護ブランチ

---

## 4. 便利なスクリプト変換ツールは？

現状 _公式コンバータ_ は無し。
ただし **90% は機械的置換** で済むので、下記ワンライナーが意外と役立ちます。

```bash
# 例: Azure の `script:` 行を GitLab の list に変換
sed -E 's/^ +- script: (.*)$/    - \1/' azure-pipelines.yml > .gitlab-ci.yml
```

> 複雑なテンプレート (extends, parameters) は **手動で読んで意図を掴む** 方が早いことが多い。

---

## まとめ

- **概念はほぼ 1:1**：Stage → stage, Job → job, Step → script 行。
- 違いは “YAML 階層” と “Runner/Agent 指定方法”。
- 移行は **シンプルなジョブから段階的に書き換え** → 変数・パッケージ周りを調整するとスムーズです。

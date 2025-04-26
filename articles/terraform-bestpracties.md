---
title: "Google Cloudデータ基盤におけるTerraformのベストプラクティス"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["googlecloud","terraform"]
published: true
---

# 想定読者

- Google Cloud でデータ基盤（BigQuery、Cloud Storage、Cloud Composer、IAM、VPC、Cloud Run など）を Terraform で管理するベストプラクティスを知りたい

- Terraform を活用することで可搬性・セキュリティ・運用効率の向上させたい。

## Terraformのコード構造とコードスタイルの統一

### フォルダ構成

Terraform のプロジェクトは、以下のように**リソースごとに分割**すると管理しやすい。

```
terraform/
├── modules/              # 再利用可能なモジュール
│   ├── vpc/
│   ├── bigquery/
│   ├── storage/
├── environments/
│   ├── prod/             # 本番環境のTerraformコード
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── terraform.tfvars
```

### **各ファイルの役割**
| ファイル | 役割 |
|---------|------|
| `main.tf` | リソース定義（例：BigQueryのデータセット、VPC） |
| `variables.tf` | 変数の定義 |
| `terraform.tfvars` | 変数の値（環境ごとに設定） |
| `outputs.tf` | Terraformの実行結果（例：BigQueryのデータセットID） |

---

## モジュールの実装例
```tf:modules/bigquery/main.tf
resource "google_bigquery_dataset" "this" {
  dataset_id = var.dataset_id
  project    = var.project_id
  location   = var.location
}
```

## モジュールの利用例
```tf:environments/prod/main.tf
module "bigquery" {
  source     = "../../modules/bigquery"
  dataset_id = "analytics"
  project_id = "my-gcp-project"
  location   = "US"
}
```

# Terraformの状態管理

Terraform のステートファイルは、Google Cloud Storage（GCS）に保存することで、下記メリットを享受できる。

- チームでの Terraform 管理が容易（ローカルの変更が他のメンバーと競合しない）
- バックアップ可能（GCS のバージョニング機能で過去の状態を復元できる）

## ステートファイルの GCS バックエンド実装

```tf
terraform {
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "prod/state"
  }
}
```

# IAMの最小権限の原則

Terraform で IAM ポリシーを管理する際は、必要最小限の権限を付与することで、セキュリティリスクを減らす。
またパスワードや API キーなどの機密情報は、環境変数や管理ツールを使用して安全に取り扱う。

## IAM ポリシーの設定例
```tf
resource "google_project_iam_member" "bq_reader" {
  project = var.project_id
  role    = "roles/bigquery.dataViewer"
  member  = "user:analyst@example.com"
}
```

## 推奨するIAM管理のポイント
- サービスアカウントごとに権限を細かく分ける
- roles/editor や roles/owner を乱用しない

# CI/CDパイプラインの活用

Terraform の変更を手作業で適用しない。
GitHubActions などで、CI/CD 自動化することで、ミスを防ぎ、安全性を向上できる。

## CI/CD実装例
```yml
name: Terraform CI/CD

on:
  pull_request:
    branches:
      - main

defaults:
  run:
    working-directory: ${{ env.tf_actions_working_dir }}

permissions:
  pull-requests: write

jobs:
  terraform:
    runs-on: ubuntu-latest
    env:
      tf_actions_working_dir: ./terraform  # 必要に応じて作業ディレクトリを指定

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.11.1  # 必要に応じてTerraformのバージョンを指定

      - name: Terraform fmt
        id: fmt
        run: terraform fmt -check
        continue-on-error: true

      - name: Terraform Init
        id: init
        run: terraform init -input=false

      - name: Terraform Validate
        id: validate
        run: terraform validate -no-color

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -input=false
        continue-on-error: true

      - name: Post Plan Comment
        uses: actions/github-script@v7
        if: github.event_name == 'pull_request'
        env:
          PLAN: ${{ steps.plan.outputs.stdout }}
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const { data: comments } = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
            });
            const botComment = comments.find(comment => {
              return comment.user.type === 'Bot' && comment.body.includes('Terraform Format and Style');
            });

            const output = `#### Terraform Format and Style 🖌\`${{ steps.fmt.outcome }}\`
            #### Terraform Initialization ⚙️\`${{ steps.init.outcome }}\`
            #### Terraform Validation 🤖\`${{ steps.validate.outcome }}\`
            <details><summary>Validation Output</summary>

            \`\`\`
            ${{ steps.validate.outputs.stdout }}
            \`\`\`

            </details>

            #### Terraform Plan 📖\`${{ steps.plan.outcome }}\`

            <details><summary>Show Plan</summary>

            \`\`\`
            ${process.env.PLAN}
            \`\`\`

            </details>

            *Pusher: @${{ github.actor }}, Action: \`${{ github.event_name }}\`, Working Directory: \`${{ env.tf_actions_working_dir }}\`, Workflow: \`${{ github.workflow }}\`*`;

            if (botComment) {
              await github.rest.issues.updateComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: botComment.id,
                body: output,
              });
            } else {
              await github.rest.issues.createComment({
                issue_number: context.issue.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body: output,
              });
            }
```

## GitHubActions解説

| **設定項目** | **説明** |
|--------------|----------------------------------|
| `terraform_version` | 必要に応じて、特定の Terraform バージョンを指定 |
| `working-directory` | `defaults` セクションで作業ディレクトリを指定し、各ステップでのディレクトリ指定を省略 |
| `continue-on-error: true` | `terraform fmt` と `terraform plan` ステップでエラーが発生しても、ワークフロー全体を停止せずに続行 |
| `Post Plan Comment` | `actions/github-script@v7` を使用して、Pull Request に terraform planの結果をコメント投稿 |

# BigQueryのコスト最適化

## パーティション分割し、無駄なデータスキャンを防ぐ
```tf
resource "google_bigquery_table" "events" {
  dataset_id = "analytics"
  table_id   = "events"

  time_partitioning {
    type = "DAY"
  }
}
```

## 不要なデータを自動削除する
```tf
resource "google_bigquery_table" "logs" {
  dataset_id = "system_logs"
  table_id   = "audit_logs"

  time_partitioning {
    type           = "DAY"
    expiration_ms  = 7776000000 # 90日後に削除
  }
}
```

# Cloud Storageのコスト削減
## Coldline/Nearline へ自動移行

```tf
resource "google_storage_bucket_lifecycle_rule" "archive" {
  condition {
    age = 90
  }
  action {
    type = "SetStorageClass"
    storage_class = "COLDLINE"
  }
}
```

# まとめ
| **項目** | **ベストプラクティス** | **詳細説明** |
|----------|------------------|------------------|
| **コード構造** | リソースごとに明確に分割 | Terraform のコードを `modules/` と `environments/` に分け、メンテナンスしやすい構成にする |
| **モジュール活用** | BigQuery、IAM、VPC などをモジュール化 | 再利用可能なモジュールを作成し、環境ごとの差異を最小限に抑える |
| **ステート管理** | **GCS** で保存 | Terraform の状態管理を GCS に集約し、チームで整合性を担保した開発・運用を可能にする |
| **IAM管理** | **最小権限の原則** を適用 | IAM のロールを細かく分け、`roles/editor` などの過剰な権限付与を避ける |
| **CI/CD** | **Pull Request経由でTerraform適用** | GitHub Actions を利用し、Terraform の `plan` 結果を PR に自動コメントしてレビューしやすくする |
| **コスト最適化** | **BigQueryのパーティション管理、Cloud Storageのライフサイクルルール** | BigQuery のパーティションを利用し、不要なデータスキャンを防ぐ。Cloud Storage のライフサイクルルールを設定し、不要データを Coldline/Nearline に移行する |


# 参考文献

- https://www.terraform-best-practices.com/
- https://developer.hashicorp.com/terraform/cloud-docs/recommended-practices/
- https://cloud.google.com/docs/terraform/best-practices/general-style-structure/
- https://github.com/hashicorp/setup-terraform
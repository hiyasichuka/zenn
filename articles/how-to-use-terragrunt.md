---
title: "TerragruntでBigQueryリソースを効率的に管理したい"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gcp","bigquery","terragrunt"]
published: true
---

# はじめに

Terraformを快適に利用するためのラッパーツールのTerragruntを使って、BigQueryのデータセットとテーブルを構築する方法を解説します。
Terraformのみだと、環境（dev/prodなど）ごとにコードを複製したり、stateファイルの管理が煩雑になる問題に直面することがあります。
本記事は、それらの問題をTerragruntでどのように解決するか体感するためのハンズオン記事です。

---

## 目標

* GCS（Google Cloud Storage）上でTerraformのstateを管理する
* dev環境とprod環境で異なるBigQueryデータセット・テーブルを作成する
* Terragruntを用いてTerraform構成を共通化・再利用可能にする

## 前提条件（準備）

以下ツールが利用できることを確認してください。

```sh
terraform -v
# Terraform v1.11.4
# on linux_amd64

terragrunt -v
# terragrunt version 0.78.0

gcloud -v
# Google Cloud SDK 457.0.0
# bq 2.0.100
# bundled-python3-unix 3.11.6
# core 2023.12.08
# gcloud-crc32c 1.0.0
# gsutil 5.27
```

---

### gcloud CLIの初期化と認証

```bash
gcloud init
gcloud auth application-default login
```

認証キー（サービスアカウント）を使う場合は、環境変数に設定してください：

```bash
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/your-key.json"
```

---

## ディレクトリ構成

```bash
infra/
├── modules/                         # Terraformの再利用可能なモジュール
│   └── bigquery/                    # BigQueryのモジュール群（dataset, table）
│       ├── dataset.tf
│       ├── table.tf
│       └── variables.tf
└── environments/                    # Terragruntで管理する各環境の構成
    ├── terragrunt.hcl               # 全環境共通の設定（remote_stateなど）
    └── dev/
    │   └── terragrunt.hcl           # dev環境の個別設定
    └── prod/
        └── terragrunt.hcl           # prod環境の個別設定
```

## TerraformとTerragruntの構成を分ける理由

Terraform(.tf)とTerragrunt(terragrunt.hcl)は、役割が異なるツールです。

| ツール        | 役割                                        |
| ---------- | ----------------------------------------- |
| Terraform  | BigQueryデータセット・テーブルなど「何を作るか（リソース定義）」を記述する |
| Terragrunt | dev/prodなど「どこでどう使うか（環境やステートの管理）」を制御する     |

このように目的が異なるため、**再利用性の高いTerraformコード（モジュール）は `modules/` に集約し、環境ごとの適用構成や変数定義は `environments/` に配置する**という構成を採用しています。これにより、次のような利点があります。

* Terraformモジュールの再利用がしやすくなる（別プロジェクトでも流用可能）
* dev/prod環境でのstateファイルの混乱を防げる
* 共通処理（state管理やproject IDなど）を1箇所にまとめられる
* `terragrunt run-all` で複数環境を一括操作可能
* 環境固有の一時的なリソースも `environments/` 側で柔軟に管理可能

## モジュール定義（modules/bigquery）

### dataset.tf

```hcl
resource "google_bigquery_dataset" "default" {
  dataset_id                  = var.dataset_id
  location                    = "US"
  delete_contents_on_destroy = true
}
```

### table.tf

```hcl
resource "google_bigquery_table" "default" {
  dataset_id = google_bigquery_dataset.default.dataset_id
  table_id   = var.table_id

  schema = jsonencode([
    {
      name = "id"
      type = "STRING"
      mode = "REQUIRED"
    },
    {
      name = "created_at"
      type = "TIMESTAMP"
      mode = "NULLABLE"
    }
  ])
}
```

### variables.tf

```hcl
variable "dataset_id" {
  description = "The ID of the BigQuery dataset"
  type        = string
}

variable "table_id" {
  description = "The ID of the BigQuery table"
  type        = string
}
```

## environments構成

### environments/terragrunt.hcl（共通設定）

```hcl
locals {
  project_id = "my-terragrunt-demo"
  region     = "us-central1"
}

remote_state {
  backend = "gcs"
  config = {
    bucket = "my-terragrunt-state-bucket"
    prefix = "${path_relative_to_include()}/terraform.tfstate"
  }
}
```

この設定により、Terraformのstateファイルは各環境（dev/prod）ごとにディレクトリ構造に応じてGCSに分離して保存されます。

### environments/dev/terragrunt.hcl

開発環境用のデータセットとテーブルのリソースを定義します。

```hcl
include {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../modules/bigquery"
}

inputs = {
  dataset_id = "dev_dataset"
  table_id   = "user_table"
}
```

### environments/prod/terragrunt.hcl

本番環境用のデータセットとテーブルのリソースを定義します。

```hcl
include {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../modules/bigquery"
}

inputs = {
  dataset_id = "prod_dataset"
  table_id   = "user_table"
}
```

## find\_in\_parent\_folders() の意味と役割

Terragruntでは、各環境（dev, prodなど）の `terragrunt.hcl` に以下のような記述があります。

```hcl
include {
  path = find_in_parent_folders()
}
```

この `find_in_parent_folders()` は、「親ディレクトリから共通設定の `terragrunt.hcl` を探して取り込む」関数です。
`environments/dev/terragrunt.hcl` などから、親フォルダを上にたどって `environments/terragrunt.hcl` を見つけ、そこに定義された `locals` や `remote_state` を **自動的に継承**してくれます。したがって、各環境ごとの差分（`inputs`など）だけに着目すればよくなります。

## GCSバケットの作成（state保存用）

```bash
gsutil mb -p my-terragrunt-demo -l us-central1 gs://my-terragrunt-state-bucket
```

バケットのstateファイルは下記コマンドで確認できます。

```bash
gsutil ls -r gs://my-terragrunt-state-bucket
```

## 適用手順

### 1. dev環境の初期化と適用

```bash
cd environments/dev
terragrunt init
terragrunt apply
```

### 2. prod環境の適用

```bash
cd ../prod
terragrunt apply
```

### 3. 複数環境を一括適用（任意）

```bash
cd ..
terragrunt run-all apply
```

この `run-all` コマンドは、複数のTerragrunt構成を**自動で依存順に**適用する便利な機能です。

## Terragruntを使うメリット（特にBigQueryで）

* 環境ごとに重複していたコードを `inputs` で簡潔に切り替えられる
* GCSへのstate保存を `remote_state` に一元定義できる
* 複数ディレクトリの管理が `run-all` によって簡素化
* ディレクトリ構成がそのままstate管理に活かされ、可視性が高い
* 将来的なデータセット・テーブルの追加にも強い構成

## クリーンアップ

リソースを削除する場合は以下を実行します。

```bash
cd environments
terragrunt run-all destroy
```

## まとめ

Terraform単体で運用する際の「環境ごとの記述の重複」や「手動管理の煩雑さ」といった問題に対して、Terragruntがどのように解決するかを体感してみました。
インフラ環境・構成の切り替えが多いケースでは、Terragruntは非常に強力なツールです。再利用性と保守性の高い構成を実現するために、今後のプロジェクトでも積極的に活用していきたいですね。


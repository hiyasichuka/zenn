---
title: 'Terraform で Vertex AI 接続を作成し、BigQuery ML から呼び出せるようにする'
emoji: '🧠'
type: 'tech'
topics: ['terraform', 'bigquery', 'vertexai', 'gcp']
published: true
---

## 背景

BigQuery ML は、`REMOTE MODEL` により Vertex AI のモデルを呼び出して予測処理が可能です。ただし、これを実現するには **BigQuery ↔ Vertex AI 間のコネクション（接続）設定** と **IAM 権限の適切な付与** が必要です。

この記事では、Terraform を使って以下のような構成を行う方法と、その実装上の工夫ポイントを紹介します。

## ゴール

- Terraform で Vertex AI 用の BigQuery 接続を作成
- 自動作成されるサービスアカウントに適切な IAM ロールを付与
- BigQuery ML から Vertex AI モデルを呼び出して推論可能にする

## やること

1. Vertex AI 接続リソースの Terraform 実装
2. 自動生成される SA への IAM ロール付与
3. モデル作成・呼び出しのテスト（BigQuery ML）

## Terraform 構成と実装

### ファイル構成の方針

一般的なプロジェクトでは、 BigQuery とのコネクションリソースを `bigquery-connection.tf` といった形で管理していると思われます。ただ、Vertex AI 接続は動的 SA へのロール付与という**特殊な挙動**があるため、以下のように **別ファイルで分離管理** する方針としました。

```
terraform/google/bigquery/
├── bigquery-connection.tf     # 一般的な接続定義
├── vertex-ai-connection.tf    # Vertex AI接続専用（今回追加するやつ）
```

このようにすることで、特殊な処理を明確に分離し、メンテナンス性・見通しの良さを担保しています。

### Terraform 実装例

```hcl
# Vertex AI 接続リソースの作成
resource "google_bigquery_connection" "vertex_ai" {
  connection_id = "vertex_ai"         # 接続名（任意）
  project       = var.project_id      # プロジェクトID（変数で管理）
  location      = "US"                # リージョン（例：US、asia-northeast1）

  cloud_resource {}                   # GCP内部サービス（Vertex AIなど）との接続を確立するためのブロック。必須。
}

# 自動作成される SA に Vertex AI モデル呼び出し権限を付与
resource "google_project_iam_member" "vertex_ai_sa_aiplatform_user" {
  project = var.project_id
  role    = "roles/aiplatform.user"
  member  = "serviceAccount:${google_bigquery_connection.vertex_ai.cloud_resource[0].service_account_id}" # 接続作成時に自動生成されるSAを動的に参照し、IAMロールを安全に割り当てる

  depends_on = [google_bigquery_connection.vertex_ai]
}
```

### 順序制御について

- Vertex AI の接続を作成すると、接続用の**サービスアカウントが動的に作成される**
- その SA に対して `roles/aiplatform.user` を **後から付与** する必要がある
- `depends_on` により、SA の生成が完了してから IAM ロール付与が走るよう **明示的に制御**

### 動的に生成される SA を参照

Vertex AI 用の接続を作成すると、BigQuery 側でサービスアカウント（SA）が自動的に生成されます。この SA に対してロールを付与する必要がありますが、Terraform では次のように書くことで、静的な文字列ではなく動的に生成された SA を参照できます。

```hcl
member  = "serviceAccount:${google_bigquery_connection.vertex_ai.cloud_resource[0].service_account_id}"
```

`cloud_resource[0].service_account_id` というようにインデックス指定が必要になりますが、これは cloud_resource がリスト構造で返るためです。

## 動作確認クエリ

適宜 `project_id` や `データセットID` を置き換えて実行してください。

```sql
CREATE OR REPLACE MODEL `test_dataset.vertex_model`
REMOTE WITH CONNECTION `project_id.region.vertex_ai`
OPTIONS (
  endpoint = 'gemini-2.0-flash-001',
);
```

## 命名スタイルの補足

Terraform 公式のスタイルガイドでは、`resource_name` は**スネークケースが推奨**されています。

https://developer.hashicorp.com/terraform/language/style#resource-naming

## おわりに

画面からの設定ではなく、Terraform でのコード管理により、再現性のある環境構築が可能になります。特に Vertex AI のような動的なリソースを扱う場合、IAM 権限の付与や接続設定をコードで明示化することで、運用の透明性と安全性が向上します。

## 参考

https://cloud.google.com/bigquery/docs/iterate-generate-text-calls?hl=ja#create_a_connection

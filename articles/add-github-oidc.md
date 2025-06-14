---
title: "GitHub ActionsからGCPを操作するためのOIDC認証をTerraformで構築する"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gcp","oidc","actions","terraform"]
published: true
---


# はじめに

GitHub ActionsからGoogle Cloudリソースを安全に操作するためには、**OIDC（OpenID Connect）によるWorkload Identity Federation**の構成が推奨されています。

本記事では、TerraformでOIDC関連リソースを構築し、GitHub Actionsから安全にGCPへアクセスする方法を紹介します。

## なぜOIDC認証？

サービスアカウントキー（JSON）をGitHub Secretsに保存して、それをActionsで使うのが手っ取り早いですが、以下のような問題があります。

* 秘密鍵の漏洩リスク
* キーのローテーション管理が面倒
* IAMベストプラクティスに反する

代わりに、OIDCを使えばキーレスで安全に認証できます。


## Workload Identity Federationとは？

GCPにおけるWorkload Identity Federationは、外部サービス（GitHub、Azureなど）からの認証情報を使用して、Google Cloudのサービスアカウントにアクセス権を委任できる仕組みです。

この仕組みの中核となるのが、以下の2つのリソースです。

**Workload Identity Pool**

- 外部IDを受け入れるための論理的な「プール（空間）」
- 複数のプロバイダーを束ねることができる

**Workload Identity Pool Provider**

- 実際にどのIDプロバイダー（今回はGitHub）からの認証を許可するかを定義するエンティティ
- issuer_uri（認証元）や属性マッピング（assertionのどの項目を使うか）などを設定

これらを組み合わせることで、「GitHub ActionsのOIDCトークンなら、GCP上のこのサービスアカウントになりすませる」という設定が実現できます。

## Terraform構成の全体像

以下の3つのリソースをTerraformで作成します

1. **Workload Identity Pool**
2. **OIDC Provider（GitHub Actions用）**
3. **サービスアカウント & ポリシーバインディング**


## Terraformコードの実装例

### 1. Workload Identity Pool

```hcl
resource "google_iam_workload_identity_pool" "github" {
  workload_identity_pool_id = "github-actions"
  display_name              = "GitHub Actions Pool"
  description               = "OIDC Pool for GitHub Actions"
}
```

### 2. OIDC Providerの作成

```hcl
resource "google_iam_workload_identity_pool_provider" "github_provider" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.github.workload_identity_pool_id
  workload_identity_pool_provider_id = "github-provider"
  display_name                       = "GitHub Actions Provider"

  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }

  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.repository" = "assertion.repository"
    "attribute.actor"      = "assertion.actor"
    "attribute.aud"        = "assertion.aud"
  }

  attribute_condition = "attribute.repository == \"your-org/your-repo\""
}
```

`attribute_condition` を使うことで、対象の組織やリポジトリに制限できます。



### 3. サービスアカウントとIAMバインディング

```hcl
resource "google_service_account" "github_actions" {
  account_id   = "terraform-apply"
  display_name = "GitHub Actions Terraform Apply SA"
}

resource "google_service_account_iam_member" "github_bind" {
  service_account_id = google_service_account.github_actions.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.repository/your-org/your-repo"
}
```

上記により、GitHub Actionsからこのサービスアカウントになりすまし（impersonate）できるようになります。



## GitHub Actionsの設定

以下のように `google-github-actions/auth` を使ってOIDC認証します

```yaml
- name: Authenticate to Google Cloud via OIDC
  uses: google-github-actions/auth@v2
  with:
    project_id: 'your-gcp-project-id'
    workload_identity_provider: 'projects/xxxx/locations/global/workloadIdentityPools/github-actions/providers/github-provider'
    service_account: 'terraform-apply@your-gcp-project-id.iam.gserviceaccount.com'
```



## ポイントまとめ

| 項目                 | 内容                                           |
|----------------------|------------------------------------------------|
| 鍵レス認証           | サービスアカウントキー不要で安全             |
| Terraform管理        | インフラコードとして再利用＆一貫性           |
| リポジトリ制限       | `attribute_condition` によるアクセス制御     |
| GitHub Actions連携   | `google-github-actions/auth` による認証設定  |


# 参考ドキュメント

https://cloud.google.com/iam/docs/workload-identity-federation?hl=ja

https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/iam_workload_identity_pool_provider#example-usageiam-workload-identity-pool-provider-github-actions

https://github.com/google-github-actions/auth/tree/main#preferred-direct-workload-identity-federation
---
title: "GitHubでClaude Codeを呼び出す"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["claude","gcp"]
published: true
---

## 概要

Claude CodeをGitHub Actions経由で利用し、Pull Requestに対する自動レビューを実現しました。
Vertex AIでClaude Sonnet 4をデプロイし、GitHub Actionsと連携する方法をまとめました。

## 1. Claude Codeとは

Claude Codeは、Anthropic社が提供するコーディング支援エージェントです。
GitHubのPull Requestに @claude とコメントをすると、Claudeがコードの改善点や設計観点を提案してくれる便利なツールです。
例えば、関数の命名センスやtypo、セキュリティの考慮不足など、レビュー担当者が見落としがちな視点を自動で指摘してくれます。

詳しい情報は以下をご参照ください。
[https://docs.anthropic.com/ja/docs/claude-code/overview](https://docs.anthropic.com/ja/docs/claude-code/overview)


## 2. セットアップ手順

### 2.1 Claudeのモデルを利用できるようにする

Claude Codeを使うには、LLM（Sonnetなど）をデプロイしておく必要があります。
代表的な選択肢は以下のとおりです。

* AnthropicのAPI（APIキー利用）
* AWS Bedrock（IAM経由）
* Google Cloud Vertex AI（Workload Identity経由）

今回は Vertex AI を使って Claude Sonnet 4 を利用しました。

1. Google Cloud Consoleから Vertex AI → Model Garden へ
2. Claude Sonnet モデルを選択し、有効にする
3. 求められる入力項目を埋めて、利用申請（数分で完了）

![img](/images/enable-vertex-ai-claude.png)

> 💡Claude Sonnet 4 米国のサポートリージョンが、us-east5 のみ
> https://cloud.google.com/vertex-ai/generative-ai/docs/partner-models/claude/sonnet-4?hl=ja


### 2.2 認証情報の準備

GitHub ActionsからGCPリソース（Vertex AI）へアクセスするには、OIDC経由でService Accountを用いた認証が必要です。（AWSの場合はAssumeRoleの設定が必要）
今回は、サービスアカウントの作成、権限付与およびアイデンティティプールといったリソース作成は割愛します。詳細な手順については、[こちらの記事](https://zenn.dev/takaha4k/articles/add-github-oidc)をご参照ください。


認証用Composite Action(Workflowから呼び出せる部品化したやつ)の例

`.github/actions/auth-gcp/action.yml`

```yaml
name: Authenticate to Google Cloud (OIDC)
description: Authenticates to GCP via OIDC
runs:
  using: 'composite'
  steps:
    - uses: google-github-actions/auth@v2.1.10
      with:
        project_id: '****'
        workload_identity_provider: 'projects/****/locations/global/workloadIdentityPools/github-actions/providers/***'
        service_account: '****.iam.gserviceaccount.com'
```

> Service Accountを使用する構成では、APIキー（ANTHROPIC_API_KEY）は不要です。

### 2.3 Workflowの作成 (`.github/workflows/claude.yml`)

`@claude` をトリガーとして起動させるGitHub Actionsの定義例

```yaml
name: Claude PR Action

permissions:
  contents: write
  pull-requests: write
  issues: write
  id-token: write

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]

jobs:
  claude-pr:
    if: |
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'issues' && contains(github.event.issue.body, '@claude'))
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4.2.2

      - uses: ./.github/actions/auth-gcp # 先述したcomposite(部品)を使う

      - uses: anthropics/claude-code-action@v0.0.44
        with:
          timeout_minutes: 30
          github_token: ${{ secrets.GITHUB_TOKEN }}
          use_vertex: "true" # ←Google CloudのClaude利用する場合は設定必須
          model: "claude-sonnet-4@20250514"
        env: # ↓Google CloudのClaude利用する場合は設定必須
          ANTHROPIC_VERTEX_PROJECT_ID: "***"
          CLOUD_ML_REGION: "us-east5"
```

## 3. 運用上のポイント

* イシューまたはPRコメントで `@claude` を含むコメントで自動実行
* 新たにGitHub Appを作成せず、GitHub Actions Botで運用
* タイムアウトはデフォルト値よりも低めにしておく

## 4. まとめ

* GitHub Actionsワークフローにより、Claude Codeを導入
* Vertex AIとOIDCを組み合わせることで、セキュアにデプロイ・運用可能
* コストやトリガー頻度には注意し、最小構成から始めるべし

# 参考

https://github.com/anthropics/claude-code

https://docs.anthropic.com/ja/docs/claude-code/google-vertex-ai
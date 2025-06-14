---
title: "GitHub ActionsからCloud Run Functionsを安全に呼び出すまでの道のり（OIDC & IDトークン）"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["actions","gcp","functions"]
published: true
---

# はじめに

GitHub ActionsからCloud Run Functionsを呼び出す処理を構築する中でつまずいたポイントと、その対応方法をまとめます。

## つまずきポイント1：IDトークンが取得できない

GitHub ActionsではOIDCでの認証が済んでいるため、以下のようにIDトークンを取得してCloud Functionsにリクエストを投げようとしました。

`gcloud auth print-identity-token`

しかし、以下のようなエラーが発生して失敗します。

`ERROR: (gcloud.auth.print-identity-token) No identity token can be obtained from the current credentials.`


OIDC認証では `gcloud auth print-identity-token` が機能しないという落とし穴に気づくのに時間がかかりました。

## つまずきポイント2：Cloud FunctionsごとにAudience指定が必要

調査の結果、`google-github-actions/auth` アクションで IDトークンを取得するには、以下のように `token_format: 'id_token'` と `id_token_audience` を明示的に指定する必要があることが分かりました。

```yaml
- name: Authenticate to Google Cloud via OIDC
  uses: google-github-actions/auth@v2
  with:
    project_id: '****'
    workload_identity_provider: 'projects/xxx/locations/global/workloadIdentityPools/xxx/providers/xxx'
    service_account: '****'
    token_format: 'id_token'  # ← これが必要
    id_token_audience: 'https://us-central1-xxx.cloudfunctions.net/xxxx'  # ← これも必要

```

Cloud Functionsが複数存在する場合、それぞれの関数URLがaudienceになるため、認証ステップを関数ごとに分ける必要が出てきます。

# 解決策：関数ごとにトークンを取得し、HTTPリクエストで呼び出す

gcloud functions call や gcloud run services invoke を使った呼び出しは難航したため、最終的には curl によるHTTPリクエストでの実行に切り替えました。

認証ステップで取得した IDトークン を使って、Authorization: Bearer ヘッダーを付けてリクエストを送る方式です。

```bash
http_status=$(curl -s -o /dev/null -w '%{http_code}' -X POST "$function_url" \
  -H "Authorization: Bearer $id_token" \
  -H "Content-Type: application/json" \
  -d "$payload")
```

# 最終的な構成（workflowの一部抜粋）

```yml
env:
  xxx_URL: https://us-central1-xxx.cloudfunctions.net/xxx
  yyy_URL: https://us-central1-xxx.cloudfunctions.net/yyy

steps:
  - id: auth-xxx
    uses: google-github-actions/auth@v2
    with:
      project_id: '****'
      workload_identity_provider: 'projects/xxx/locations/global/workloadIdentityPools/xxx/providers/xxx'
      service_account: '****'
      token_format: 'id_token'
      id_token_audience: ${{ env.xxx_URL }}

  - id: auth-yyy
    uses: google-github-actions/auth@v2
    with:
      project_id: '****'
      workload_identity_provider: 'projects/xxx/locations/global/workloadIdentityPools/xxx/providers/xxx'
      service_account: '****'
      token_format: 'id_token'
      id_token_audience: ${{ env.yyy_URL }}

  - name: Call Cloud Function
    run: |
      http_status=$(curl -s -o /dev/null -w '%{http_code}' -X POST "${{ env.xxx_URL }}" \
        -H "Authorization: Bearer ${{ steps.auth-xxx.outputs.id_token }}" \
        -H "Content-Type: application/json" \
        -d "$payload")

      echo "Cloud Function responded with status: $http_status"
```


# ポイントまとめ

| 課題                           | 原因                 | 解決策                                 |
| ---------------------------- | ------------------ | ----------------------------------- |
| `print-identity-token` が使えない | OIDC認証ではサポートされていない | `google-github-actions/auth` の出力を使う |
| 関数ごとにaudienceが異なる            | ワイルドカードや複数指定は不可    | 認証ステップを関数ごとに定義して切り替える               |

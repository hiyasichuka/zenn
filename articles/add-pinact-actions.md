---
title: "pinact-actionを使ってバージョンをハッシュで固定化"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["actions","pinact"]
published: true
---

# 想定読者

CIのセキュリティ強化のため、GitHub Actionsで利用する各ライブラリのバージョンを、**自動でSHAに置き換えてcommitまで自動化**する仕組みを導入したい。


## なぜ固定（SHA指定）するのか

Actionsでこういう記述があります。

```yaml
uses: actions/checkout@v4
```

これは @v4 というタグ指定でしかなく、中身が書き換わる可能性があります。仮に悪意ある変更が含まれた場合、セキュリティ事故に繋がります。
そこで、以下のように SHAでバージョンを固定するのが望ましいです。

```yaml
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

これにより、どのコミットが使われるかが明確になり、安全性が担保されます。

## `pinact-action` を使って自動で固定化

[`suzuki-shunsuke/pinact-action`](https://github.com/suzuki-shunsuke/pinact-action) というGitHub Actionを使うと、
`.github/workflows/*.yml` や `action.yaml` の `uses:` を自動でSHA固定に書き換えてくれます。

自動的にチェックし、必要に応じてcommitまでやってくれる優れものです。


## 誰がcommitするか問題

`pinact-actions` は修正後のファイルを `commit` しますが、誰の権限で `push` するかの方法によって問題があります。

| 方法                         | 課題                                        |
| -------------------------- | ----------------------------------------- |
| `GITHUB_TOKEN`             | 権限足りない。              |
| PAT（Personal Access Token） | 署名なし。漏洩リスクもある                             |
| ✅ GitHub App（今回採用）         | ちょっと手間がかかる |


## Verified Commit とは？

GitHub で commit メッセージの横に "✅ Verified" と出るやつです。
GitHub App を使うと、この Verified Commit が自動で実現できます。

## GitHub App を使う準備

### 手順

1. GitHub App を作成する。
2. `Contents: write`, `Workflows: write` のパーミッションを設定
3. インストール対象を "Only select repositories" にし、対象のリポジトリを選択
4. App ID / 秘密鍵 (PEM) を取得
5. `APP_ID`, `APP_PRIVATE_KEY` を **対象のリポジトリの Secrets に登録**



## 実際のワークフロー例

```yaml
name: Pinact
on:
  pull_request: {}

permissions:
  contents: write
  pull-requests: write

jobs:
  pinact:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          persist-credentials: false
      
      - uses: suzuki-shunsuke/pinact-action@d735505f3decf76fca3fdbb4c952e5b3eba0ffdd # v0.1.2
        with:
            app_id: ${{secrets.APP_ID}}
            app_private_key: ${{secrets.APP_PRIVATE_KEY}}
```

## ハマりどころ

* GitHub App を "作っただけ" ではダメ。**リポジトリへのインストール**を忘れると `404` になる
* `APP_PRIVATE_KEY` のSecrets登録は、PEMの中身をそのまま貼ればOK（改行そのままでOK）


## 参考資料

* [pinact-action - GitHub](https://github.com/suzuki-shunsuke/pinact-action)
* [GitHub App 作成・インストール手順](https://docs.github.com/en/apps/creating-github-apps)
* [GitHub Actions のセキュリティベストプラクティス](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)

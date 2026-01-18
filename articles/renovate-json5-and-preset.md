---
title: "renovate.jsonをJSON5形式にしてコメントを書けるようにする"
emoji: "🔧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["renovate", "json5"]
published: true
---

# はじめに

依存関係の自動更新ツールとして広く使われている **Renovate**。
設定ファイル `renovate.json` を管理する中で、「なぜこの設定を入れたのか」コメントを残したいと思ったことはありませんか？

Renovate は **JSON5 形式**に対応しており、コメントが書けるようになっています。この記事では、JSON5 形式への移行方法と、便利な config-preset について紹介します。

# 想定読者

- Renovate を使っている方
- 設定ファイルにコメントを残したい方
- Renovate の設定をより効率的に管理したい方

# JSON5 とは

JSON5 は JSON の拡張仕様で、以下のような特徴があります。

- **コメントが書ける**（`//` や `/* */`）
- 末尾のカンマが許容される
- キー名をクォートなしで書ける
- 複数行の文字列が書ける

```json5
{
  // これはコメントです
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended",
  ], // 末尾カンマOK
}
```

# Renovate で JSON5 を使う方法

Renovate は以下のファイル名を自動的に認識します。

| ファイル名 | 形式 |
|-----------|------|
| `renovate.json` | JSON |
| `renovate.json5` | JSON5 |
| `.renovaterc` | JSON |
| `.renovaterc.json` | JSON5 |


```json5
// renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended"
  ],
  // patch バージョンは自動マージ
  "packageRules": [
    {
      "matchUpdateTypes": ["patch"],
      "automerge": true
    }
  ]
}
```

:::message
公式ドキュメント: [Configuration Options - Renovate Docs](https://docs.renovatebot.com/configuration-options/)
:::

# config-preset を活用する

Renovate には **config-preset** という仕組みがあり、よく使う設定をパッケージとして再利用できます。

## 代表的な preset

### 1. Renovate 公式 preset

```json5
{
  "extends": [
    "config:recommended",    // 推奨設定
    "schedule:weekly",       // 週次実行
    ":automergeMinor"        // マイナーバージョンは自動マージ
  ]
}
```

### 2. サードパーティ製の preset（aqua-renovate-config の例）

公式以外にも、様々なツールが Renovate 用の preset を公開しています。

例えば **[aqua](https://aquaproj.github.io/)** は、CLI ツールのバージョン管理ツールです。`aqua.yaml` に定義したツール（terraform、kubectl など）を宣言的に管理できます。

aqua を使っているプロジェクトでは、専用の preset を使うことで `aqua.yaml` 内のバージョンも Renovate が自動更新してくれます。

```json5
{
  "extends": [
    "config:recommended",       // Renovate の推奨設定を使用
    ":label(renovate)",         // 作成される PR に「renovate」ラベルを自動で付ける
    "github>aquaproj/aqua-renovate-config#2.9.0",  // aqua.yaml のバージョン更新を検知するための設定
    "github>aquaproj/aqua-renovate-config:aqua-renovate-config#2.9.0(renovate\\.json5)"  // ↑のプリセット自体のバージョン更新 PR を作成。renovate.json5 ファイル名にだけマッチさせるため、ドットをエスケープしている
  ]
}
```

このように、自分が使っているツールに preset があるか探してみると、設定が楽になることがあります。

参考: [aqua-renovate-config | aqua](https://aquaproj.github.io/docs/products/aqua-renovate-config/#aqua-renovate-config-preset)

## カスタム preset の作り方

チーム共通の設定を GitHub リポジトリに置いて、preset として参照することもできます。

```json5
// 参照する側
{
  "extends": [
    "github>your-org/renovate-config"
  ]
}
```

# 設定変更後の再実行

Renovate の設定を変更した後、すぐに反映させたい場合は **Dependency Dashboard** から手動でトリガーできます。

1. リポジトリの Issues から「Dependency Dashboard」を開く
2. 「Check this box to trigger a request for Renovate to run again on this repository」にチェックを入れる

これで設定変更した内容でPRが自動作成されます。

# 実践的な設定例

以下は、コメント付きで管理しやすくした設定例です。

```json5
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended"
  ],

  // PR のラベル設定
  "labels": ["dependencies"],

  // 自動マージ設定
  "packageRules": [
    {
      // patch バージョンは自動マージ
      "matchUpdateTypes": ["patch"],
      "automerge": true,
      "automergeType": "pr"
    },
    {
      // dev dependencies は自動マージ
      "matchDepTypes": ["devDependencies"],
      "automerge": true
    }
  ],

  // スケジュール設定（日本時間の月曜朝）
  "schedule": ["before 9am on Monday"],
  "timezone": "Asia/Tokyo"
}
```

# まとめ

- Renovate は **JSON5 形式**に対応しており、コメントが書ける
- **config-preset** を使うと設定を再利用できる
- サードパーティ製の preset（aqua-renovate-config など）も探してみよう
- 設定変更後は **Dependency Dashboard** から再実行できる

設定ファイルにコメントを残すことで、「なぜこの設定にしたのか」を後から確認でき、チームでのメンテナンスがしやすくなります。ぜひ JSON5 形式を活用してみてください。

# 参考リンク

- [Configuration Options - Renovate Docs](https://docs.renovatebot.com/configuration-options/)
- [aqua-renovate-config | aqua](https://aquaproj.github.io/docs/products/aqua-renovate-config/)
- [JSON5 仕様](https://json5.org/)

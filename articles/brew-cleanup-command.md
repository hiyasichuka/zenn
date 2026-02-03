---
title: "Homebrewの更新・掃除を1コマンドで終わらせる自作関数を作った"
emoji: "🧹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["homebrew", "zsh", "shell", "mac", "効率化"]
published: true
---

## はじめに

Mac ユーザーにおなじみのパッケージ管理ツール **Homebrew**。
非常に便利ですが、日々のメンテナンスとして各種コマンドを毎回手打ちするのは面倒に感じませんか。

```bash
brew update
brew upgrade
brew upgrade --cask
brew cleanup
...
```

そこで、これらをまとめて実行して環境を整理する自作関数 `brewup` を作成しました。
実行は、たったこれだけです。

```bash
brewup
```

`.zshrc` に定義して利用しているので紹介します。

## 自作関数 `brewup`

以下のコードを `.zshrc`（Bash の場合は `.bashrc`）に追記するだけです。

```bash
# Homebrew一括アップデート関数
brewup() {
  set -e  # エラーが発生したらそこで中断する
  echo "== brew update =="
  brew update
  echo ""
  
  echo "== brew upgrade (formula) =="
  brew upgrade
  echo ""
  
  echo "== brew upgrade (cask, greedy) =="
  brew upgrade --cask --greedy
  echo ""
  
  echo "== brew cleanup =="
  brew cleanup
  echo ""
  
  echo "== brew autoremove =="
  brew autoremove
  echo ""
  
  echo "✅ brew update all done"
}
```

設定を反映させるために、シェルを再起動するか `source ~/.zshrc` を実行してください。
あとはターミナルで `brewup` と打つだけで、全自動でメンテナンスが走ります。

## こだわりポイントと解説

ただコマンドを並べただけに見えますが、いくつかポイントがあります。

### 1. `set -e` で安全性を確保

冒頭の `set -e` は、**「途中でエラーが発生したら直ちに処理を中断する」** という設定です。
例えば `brew update` でネットワークエラーが起きたのに、構わず `upgrade` を走らせて変な状態になるのを防げます。

### 2. `--greedy` オプションで Cask も更新対象に含める

```bash
brew upgrade --cask --greedy
```

通常、Google Chrome や Slack のようにアプリ自身が自動更新機能を持つものは、Homebrew 側での更新対象から除外されます。
`--greedy` オプションを付与することで、それらも含めて Homebrew 経由で強制的に最新版へアップグレードを試みます。これにより、アプリごとの更新状況に依存せず、常に最新の状態を保てます。

### 3. `autoremove` でゴミを残さない

```bash
brew autoremove
```

パッケージの削除や更新を繰り返すと、ツールの依存関係として導入された「現在は不要なもの」が残る原因になります。
`brew autoremove` を実行すれば、これらを自動削除してディスク容量の節約につながります。

## おわりに

この `brewup` コマンドにより、朝イチや休憩前に実行するだけで Mac の開発環境を最新・クリーンに保てます。

Homebrew のメンテナンスをサボりがちな方は、ぜひ `.zshrc` へペタッと貼って使ってみてください。

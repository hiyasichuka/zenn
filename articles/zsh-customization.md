---
title: "zshをカスタマイズして快適なターミナル環境を構築する"
emoji: "🐚"
type: "tech"
topics: ["zsh", "terminal", "starship"]
published: false
---

# はじめに

macOS標準のzshを、最小限の拡張でカスタマイズします。
導入するのは **antidote（プラグインマネージャ）** と **Starship（プロンプト）** の2つだけです。


## 前提条件

- macOS
- Homebrew がインストール済み

---

## antidote のインストール

[antidote](https://github.com/mattmc3/antidote) でzshプラグインを管理します。

### 1. Homebrewでインストール

```bash
brew install antidote
```

### 2. .zshrc に設定を追加

`~/.zshrc` に以下を追加します。

```zsh
# antidote
source $(brew --prefix antidote)/share/antidote/antidote.zsh

antidote bundle <<EOF
zsh-users/zsh-autosuggestions
zsh-users/zsh-syntax-highlighting
EOF
```

`zsh-autosuggestions` は過去のコマンド履歴から入力候補を提案し、`zsh-syntax-highlighting` はコマンドの構文を色分けします。

---

## Starship のインストール

[Starship](https://starship.rs/) でプロンプトをカスタマイズします。

### 1. Homebrewでインストール

```bash
brew install starship
```

### 2. .zshrc に設定を追加

`~/.zshrc` に以下を追加します。

```zsh
# Starship
eval "$(starship init zsh)"
```

### 3. その他の便利な設定

`~/.zshrc` に以下も追加します。

```zsh
# 補完機能の有効化
autoload -Uz compinit && compinit

# 履歴設定
HISTFILE=~/.zsh_history
HISTSIZE=10000
SAVEHIST=10000
setopt share_history          # 複数のzshセッション間で履歴を共有
setopt hist_ignore_dups       # 重複するコマンドを履歴に保存しない
setopt hist_ignore_space      # スペースで始まるコマンドを履歴に保存しない

# ディレクトリ移動
setopt auto_cd                # ディレクトリ名だけでcd
setopt auto_pushd             # cd時に自動でpushd
setopt pushd_ignore_dups      # 重複するディレクトリをスタックに追加しない

# エイリアス
alias ll='ls -lah'
alias la='ls -A'
alias ..='cd ..'
alias ...='cd ../..'
```

## 設定を反映

ターミナルを再起動します。

```bash
exec zsh
```

これでセットアップ完了です。

## 動作確認

セットアップ後、以下の機能が利用できます。

- コマンド入力時に過去の履歴からグレーで候補が表示される
- 有効なコマンドは緑、無効なコマンドは赤で色分けされる
- Starshipによってプロンプトがカスタマイズされる

## 補足

- zshの補完機能は、標準機構と各CLIツールの定義が自動的に利用されます
- Starshipの詳細なカスタマイズは[公式ドキュメント](https://starship.rs/config/)を参照してください

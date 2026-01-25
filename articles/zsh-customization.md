---
title: "zshをカスタマイズして快適なターミナル環境を構築する"
emoji: "🐚"
type: "tech"
topics: ["zsh", "terminal", "starship"]
published: true
---

# はじめに

macOS標準のzshを、最小限の拡張でカスタマイズします。
導入するのは **antidote（プラグインマネージャ）** と **Starship（プロンプト）** の2つだけです。


## 前提条件

- macOS
- Homebrew がインストール済み

## antidote のインストール

[antidote](https://github.com/mattmc3/antidote) でzshプラグインを管理します。

### 1. Homebrewでインストール

```bash
brew install antidote
```

### 2. プラグイン設定ファイルを作成

以下のコマンドで `~/.zsh_plugins.txt` を作成します。

```bash
cat <<EOF > ~/.zsh_plugins.txt
zsh-users/zsh-autosuggestions
zsh-users/zsh-syntax-highlighting
EOF
```

`zsh-autosuggestions` は過去のコマンド履歴から入力候補を提案し、`zsh-syntax-highlighting` はコマンドの構文を色分けします。

### 3. .zshrc に設定を追加

`~/.zshrc` に以下を追加します。

```zsh
# antidote
source $(brew --prefix)/opt/antidote/share/antidote/antidote.zsh
antidote load
```


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

これだけで、Starshipがデフォルト設定で動作します。

### 3. 設定ファイルでカスタマイズ（オプション）

デフォルトのまま使っても十分便利ですが、さらにカスタマイズしたい場合は設定ファイルを作成します。

```bash
mkdir -p ~/.config
touch ~/.config/starship.toml
```

以下は設定例です（`~/.config/starship.toml`）：

```toml
# ~/.config/starship.toml

# エディタ補完用のスキーマ設定
"$schema" = 'https://starship.rs/config-schema.json'

# プロンプトの前に空行を挿入
add_newline = true

# プロンプトのシンボルを変更
[character]
success_symbol = '[➜](bold green) '
error_symbol = '[✗](bold red) '

# パッケージバージョン表示を無効化（プロンプトをシンプルに保つため）
[package]
disabled = true

# Git ブランチの設定
[git_branch]
symbol = '🌱 '
truncation_length = 4
truncation_symbol = ''
ignore_branches = ['master', 'main']

# Git コミットハッシュの設定
[git_commit]
commit_hash_length = 4
tag_symbol = '🔖 '

# Git の統計情報を表示
[git_metrics]
added_style = 'bold blue'
format = '[+$added]($added_style)/[-$deleted]($deleted_style) '

# Git Status に絵文字を使う
[git_status]
conflicted = '🏳'
ahead = '🏎💨'
behind = '😰'
diverged = '😵'
up_to_date = '✓'
untracked = '🤷'
stashed = '📦'
modified = '📝'
staged = '[++\($count\)](green)'
renamed = '👅'
deleted = '🗑'
```

### 4. その他の便利な設定

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
- **Starshipは設定ファイルなしでも十分便利に使えます**。デフォルト設定で満足できる場合は `starship.toml` を作る必要はありません
- カスタマイズしたい場合は[公式ドキュメント](https://starship.rs/config/)を参照してください
- よく使われる設定例は[Presets](https://starship.rs/presets/)で確認できます
- 設定を変更した場合は `exec zsh` で再読み込みできます

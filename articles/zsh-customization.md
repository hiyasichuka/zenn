---
title: "zshをカスタマイズして快適なターミナル環境を構築する"
emoji: "🐚"
type: "tech"
topics: ["zsh", "terminal", "starship", "antidote"]
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

```bash:ターミナル
cat <<EOF > ~/.zsh_plugins.txt
# ディレクトリ履歴をスマートに管理
rupa/z

# 補完機能の初期化と強化
mattmc3/ez-compinit
zsh-users/zsh-completions kind:fpath path:src

# コマンド履歴から入力候補を提案
zsh-users/zsh-autosuggestions

# コマンドの構文を色分け（高速版）
zdharma-continuum/fast-syntax-highlighting kind:defer

# コマンド履歴の部分一致検索
zsh-users/zsh-history-substring-search
EOF
```

#### プラグインの説明

- **rupa/z**: よく使うディレクトリに素早く移動できます（`z ディレクトリ名` で移動）
- **mattmc3/ez-compinit**: 補完機能の初期化を簡略化します
- **zsh-users/zsh-completions**: 多くのコマンドの補完定義を追加します
- **zsh-users/zsh-autosuggestions**: 過去のコマンド履歴から入力候補をグレーで提案します（右矢印キーまたは`Ctrl+E`で確定）
- **zdharma-continuum/fast-syntax-highlighting**: 有効なコマンドは緑、無効なコマンドは赤で色分けします（`kind:defer`で起動を高速化）
- **zsh-users/zsh-history-substring-search**: ↑↓キーでコマンド履歴を部分一致検索できます

### 3. .zshrc に設定を追加

`~/.zshrc` に以下を追加します。

```zsh:~/.zshrc
# antidote
source $(brew --prefix)/opt/antidote/share/antidote/antidote.zsh
antidote load

# 履歴検索のキーバインド設定（↑↓キーで履歴を部分一致検索）
bindkey '^[[A' history-substring-search-up
bindkey '^[[B' history-substring-search-down
```


## Starship のインストール

[Starship](https://starship.rs/) でプロンプトをカスタマイズします。

### 1. Homebrewでインストール

```bash
brew install starship
```

### 2. .zshrc に設定を追加

`~/.zshrc` に以下を追加します。

```zsh:~/.zshrc
# Starship
eval "$(starship init zsh)"
```

これだけで、Starshipがデフォルト設定で動作します。

### 3. 設定ファイルでカスタマイズ（オプション）

デフォルトのまま使っても十分便利ですが、さらにカスタマイズしたい場合は設定ファイルを作成します。

```bash:ターミナル
mkdir -p ~/.config
touch ~/.config/starship.toml
```

以下は設定例です（`~/.config/starship.toml`）。

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

```zsh:~/.zshrc
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

### 5. 最終的な .zshrc の全体像

参考までに、最終的な `~/.zshrc` の全体構成は以下のようになります。

```zsh:~/.zshrc
# 履歴設定
HISTFILE=~/.zsh_history
HISTSIZE=10000
SAVEHIST=10000
setopt share_history
setopt hist_ignore_dups
setopt hist_ignore_space

# ディレクトリ移動
setopt auto_cd
setopt auto_pushd
setopt pushd_ignore_dups

# antidote
source $(brew --prefix)/opt/antidote/share/antidote/antidote.zsh
antidote load

# 履歴検索のキーバインド設定
bindkey '^[[A' history-substring-search-up
bindkey '^[[B' history-substring-search-down

# Starship
eval "$(starship init zsh)"

# エイリアス
alias ll='ls -lah'
alias la='ls -A'
alias ..='cd ..'
alias ...='cd ../..'
```

## 設定を反映

ターミナルを再起動します。

```bash:ターミナル
exec zsh
```

これでセットアップ完了です。

## 動作確認

セットアップ後、以下の機能が利用できます。

### プラグインの動作確認

- **コマンド入力補完**: コマンドを入力すると、過去の履歴からグレーで候補が表示されます
- **構文ハイライト**: 有効なコマンドは緑、無効なコマンドは赤で色分けされます
- **履歴検索**: コマンドの一部を入力して↑↓キーを押すと、部分一致で履歴を検索できます
- **ディレクトリジャンプ**: `z プロジェクト名` でよく使うディレクトリに移動できます（数回訪れたディレクトリを記憶します）

### Starshipのカスタマイズ確認

- プロンプトがカスタマイズされ、Gitリポジトリではブランチ名や変更状態が表示されます
- カレントディレクトリがわかりやすく表示されます

## 補足

### プラグインについて

- **`kind:defer`**: プラグインの読み込みを遅延させることで、zshの起動を高速化します
- **`kind:fpath`**: 補完定義などを関数パス（fpath）に追加します
- プラグインを追加・削除した場合は `exec zsh` で再読み込みしてください

### zコマンドの使い方

`z` コマンドは、一度訪れたディレクトリを記憶し、ディレクトリ名の一部で素早く移動できます。

```bash:ターミナル
# 最初は通常通りcdで移動（zが学習します）
cd ~/Projects/my-project

# 次回からはzで移動可能
z my-project   # ~/Projects/my-projectに移動
z proj         # 部分一致でも移動可能
```

### Starshipのカスタマイズ

- zshの補完機能は、標準機構と各CLIツールの定義が自動的に利用されます
- **Starshipは設定ファイルなしでも十分便利に使えます**。デフォルト設定で満足できる場合は `starship.toml` を作る必要はありません
- カスタマイズしたい場合は[公式ドキュメント](https://starship.rs/config/)を参照してください
- 設定を変更した場合は `exec zsh` で再読み込みできます

### 参考リンク

- [antidote公式ドキュメント](https://antidote.sh/)
- [Starship公式ドキュメント](https://starship.rs/)

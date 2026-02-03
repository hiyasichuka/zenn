---
title: "anyenvからmiseに乗り換えて開発環境を現代化する"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mise", "anyenv", "rust", "nodejs", "python"]
published: true
---

## はじめに

長らく開発ツールのバージョン管理に **anyenv** を利用してきましたが、最近では更新が止まっており、新しいツールへの移行を決断しました。

そこで、Rust 製で高速に動作し、**複数言語のバージョン管理・環境変数・タスク実行を一元管理できる**次世代のツールマネージャーである **mise** に乗り換えたので、その導入手順と魅力を紹介します。

## miseの導入方法

### 1. インストール

macOS であれば Homebrew で簡単にインストールできます。

```bash
brew install mise
```

### 2. シェルの設定

`.zshrc` や `.bashrc` に以下の設定を追加して、mise を有効化します。

```bash
# zshの場合
echo 'eval "$(mise activate zsh)"' >> ~/.zshrc

# bashの場合
echo 'eval "$(mise activate bash)"' >> ~/.bashrc
```

設定後はシェルを再起動するか、以下を実行して反映させます。

```bash
# zshの場合
source ~/.zshrc

# bashの場合
source ~/.bashrc
```


## 基本的な使い方

### デフォルトバージョンを設定する

anyenv（nodenv/pyenv）と同様に、システム全体で使用するデフォルトのバージョンを設定しておくと便利です。これを設定しないと、プロジェクト固有の設定がないディレクトリでコマンドが見つからない状態になります。

```bash
# グローバルで常に使用するバージョンを指定
# @の後ろには具体的なバージョン番号を指定（例: node@22 は Node.js 22.x系の最新版）
mise use --global node@22
mise use --global python@3.12
```

このコマンドを実行すると `~/.config/mise/config.toml` に設定が書き込まれます。

### プロジェクトごとにバージョンを指定する

プロジェクト固有のバージョンを使いたい場合は、プロジェクトのルートディレクトリで以下を実行します。

```bash
# カレントディレクトリに .mise.toml が作成される
mise use node@20
mise use python@3.11
```

これで、そのディレクトリ配下に入った時だけ指定したバージョンが有効になります。

### ツールのインストール

バージョンを指定しただけでは、実際のツールはまだインストールされていません。以下のコマンドで必要なツールをインストールします。

```bash
# .mise.toml に記載されているツールを一括インストール
mise install

# 特定のツールのみインストール
mise install node@20
```

### .mise.toml による管理

`mise use` を実行すると、以下のような `.mise.toml` が作成されます。

```toml
[tools]
node = "20"
python = "3.11"
```

このファイルをリポジトリにコミットしておけば、他のメンバーは以下の手順で同じ環境を構築できます。

1. リポジトリをクローン
2. プロジェクトのルートディレクトリに移動
3. `mise install` を実行

これだけで `.mise.toml` に記載されているすべてのツールが自動的にインストールされます。

:::message
**mise use と mise install の使い分け**

似たような挙動をしますが、役割が異なります。

- **`mise use`**: 使用するバージョンを固定設定するコマンドです。`.mise.toml` を書き換えます。指定したバージョンが未インストールの場合はインストールも行います。
- **`mise install`**: `.mise.toml` に書かれたツールをインストールするコマンドです。設定ファイルは変更しません。

**使い分けの例**:
- 自分でバージョンを決めるとき → `mise use`
- リポジトリを clone してきた直後など、既存の設定に合わせて環境を作るとき → `mise install`
:::

## anyenvからの移行手順

anyenv から mise へ移行手順を下記に整理します。

### 1. 既存のバージョン確認

まず、現在使用しているツールのバージョンを確認しておきます。

```bash
# 例: Node.jsのバージョン確認
node -v
# 例: Pythonのバージョン確認
python --version
```

### 2. mise のセットアップ

mise をインストールし、同じバージョンをグローバルまたはプロジェクト単位で設定します。

```bash
# mise のインストール
brew install mise

# シェル設定を追加
echo 'eval "$(mise activate zsh)"' >> ~/.zshrc
source ~/.zshrc

# 既存と同じバージョンを設定
mise use --global node@18  # 既存のバージョンに合わせる
mise use --global python@3.11
```

### 3. 動作確認

mise が正しく動作することを確認してから、anyenv をアンインストールします。

```bash
# バージョンが正しく表示されることを確認
node -v
python --version
```

### 4. anyenv のアンインストール（任意）

mise が問題なく動作することを確認できたら、anyenv を削除します。

```bash
# .zshrc から anyenv の設定を削除（手動編集）
# 以下の行を削除: export PATH="$HOME/.anyenv/bin:$PATH"
# 以下の行を削除: eval "$(anyenv init -)"

# anyenv ディレクトリを削除
rm -rf ~/.anyenv

# シェルを再起動
exec $SHELL
```

## anyenvからの移行で感じたメリット

1. **圧倒的なスピード**: Rust 製ということもあり、シェルの起動やコマンドのレスポンスが非常に高速である。
2. **設定の一元管理**: `.mise.toml` 1 つでツール、環境変数、タスクが管理できるため、プロジェクトのリポジトリに含めることでチーム開発もスムーズになる。
3. **asdf プラグインとの互換性**: asdf の膨大なプラグイン資産をそのまま利用できるため、マイナーなツールにも幅広く対応している。

## トラブルシューティング

### コマンドが見つからない

`mise: command not found` と表示される場合は、シェルの設定が正しく読み込まれていない可能性があります。

```bash
# 設定を再読み込み
source ~/.zshrc

# または、シェルを再起動
exec $SHELL
```

### ツールのバージョンが切り替わらない

ディレクトリに `.mise.toml` があるのにバージョンが切り替わらない場合は、mise が有効化されているか確認します。

```bash
# mise の状態を確認
mise doctor

# 現在有効なツールを確認
mise current
```

### 古いバージョンを削除したい

使用していないバージョンを削除して、ディスクスペースを節約できます。

```bash
# インストール済みのバージョン一覧
mise list

# 特定のバージョンをアンインストール
mise uninstall node@18

# 現在使用していないバージョンをすべて削除
mise prune
```

## おわりに

mise に移行することで開発体験は確実に向上しました。
もし「最近開発環境の動作が重いな」と感じている方がいれば、ぜひ試してみてください。

## 参考文献

- [mise-en-place | mise-en-place](https://mise.jdx.dev/)
- [jdx/mise: dev tools, env vars, task runner](https://github.com/jdx/mise)
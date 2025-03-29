---
title: "Git Bash に Zsh を導入する方法"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["windows"]
published: true
---

# はじめに

Windows を利用していると macOS で利用しているカスタマイズ済みのzshシェルが恋しくなります。
WSL（Windows Subsystem for Linux）でセットアップは可能ですが、WSL は Windows のツールとの相性が悪いことがあります。
そこで、Git Bash の Bash の代わりに Zsh を動かす方法をこの記事で紹介します。

---

# 事前準備

Zshのバイナリを `C:\Program Files\Git\usr\` にコピーするためには、**管理者権限が必要**です。

1. Windows スタートメニューで「Git Bash」を検索  
2. **右クリック → 「管理者として実行」** を選択して起動

その後、以下の手順を Git Bash 上で実行してください。

# 1. Zsh のパッケージをダウンロード

MSYS2 のパッケージリポジトリから最新版の Zsh をダウンロードします。
2025/3/29時点の最新は、zsh-5.9-3-x86_64.pkg.tar.zstです。（バージョンは随時更新されます）

```bash
cd "/c/Program Files/Git"
# Zsh のパッケージをダウンロード（バージョン: 5.9-3）
curl -LO https://mirror.msys2.org/msys/x86_64/zsh-5.9-3-x86_64.pkg.tar.zst

# 解凍
tar -xvf zsh-5.9-3-x86_64.pkg.tar.zst

# 圧縮ファイルを削除
rm -f zsh-5.9-3-x86_64.pkg.tar.zst

```

# 2. Zsh を起動して初期設定

Git Bash を開いて以下のコマンドを実行します

```bash
zsh
```

初回起動時には、**Zsh の初期設定ウィザード（zsh-newuser-install）**が表示されます。


# 3. Git Bash 起動時に Zsh を自動起動させる

`nano ~/.zshrc` でエディタ起動して、に下記を追記します。

```bash
if [ -t 1 ]; then
  exec zsh
fi
```

## 4. 確認

下記コマンドを叩いて、 zsh と表示されれば完了です。

```bash
echo $0
```
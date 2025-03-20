---
title: "Chocolateyのインストール手順"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["windows"]
published: true
---

# Chocolateyとはなにか

Windowsのパッケージ管理ツール「Chocolatey」を[公式](https://chocolatey.org/install)に準拠した手順でセットアップします。
Chocolateyは、MacのHomebrewと同様に、コマンドラインからツールを簡単にインストール・管理できる仕組みです。

## インストール手順

### 1. PowerShellを管理者として起動

スタートメニューで「PowerShell」を検索し、右クリックして「管理者として実行」を選択します。


### 2. インストールコマンドを実行

以下のコマンドをコピー＆ペーストして、Chocolateyをインストールします。

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```



### 3. インストール完了の確認

以下のコマンドでChocolateyのバージョンを確認し、正しくインストールされたかを確認します。

```powershell
choco -v
```

バージョン番号が表示されれば、正常にインストール完了です。

## Chocolateyの基本的な使い方

### パッケージのインストール例

例としてGoogle Chromeをインストールする場合は以下のようになります。

```powershell
choco install googlechrome
```

### Chocolatey本体のアップグレード

Chocolatey自体をアップグレードするには以下を実行します。

```powershell
choco upgrade chocolatey
```

### インストール済みパッケージのアップグレード

全てのインストール済みパッケージを最新の状態にするには以下を実行します。

```powershell
choco upgrade all
```

### インストール済みパッケージのアンインストール

不要になったパッケージを削除するには以下を実行します。

```powershell
choco uninstall パッケージ名
```

例えばGoogle Chromeを削除するには以下のようになります。

```powershell
choco uninstall googlechrome
```
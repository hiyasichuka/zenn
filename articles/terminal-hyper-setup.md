---
title: "ターミナルのデザインをポケモンにする（Windows編）"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["design"]
published: true
---


# ゴール

ターミナルをこんな感じのデザインにします。

![alt text](/images/pokemon.png)


# やること

WindowsのマシンにHyperとhyper-pokemonを入れてセットアップする

## 1. Hyperとは？
[Hyper](https://hyper.is/) は、Electronで開発されたカスタマイズ可能なターミナルです。
JavaScript/HTML/CSSを使って拡張やカスタマイズが容易にできるみたいです。

## 2. Hyperのインストール

### 2.1 Hyperのダウンロード
以下の公式サイトから最新のHyperをダウンロードします。

https://hyper.is/

### 2.2 Hyperのインストール


1. ダウンロードした `.exe` または `.app` ファイルを実行します。
2. インストールウィザードに従ってインストールを完了させます。
3. インストールが完了したら、Hyperを起動して正しく動作するか確認します。

**ちなみにMacでHomebrewを利用している場合は、` brew install --cask hyper` でインストールできます**

## 3. hyper-pokemonのインストール

Hyperを起動後、下記のコマンドを実行します。

```sh
hyper i hyper-pokemon
```

`Ctrl + ,` のショートカットキーを叩くことで、Hyperの設定ファイル `~/.hyper.js` を開きます。


つづいて、`plugins` の配列に `"hyper-pokemon"` を追加します。

```js
module.exports = {
  config: {
    // 他の設定...
  },
  plugins: ["hyper-pokemon"],
};
```


`config` に以下を追加して、ポケモンテーマを変更できます。

```js
module.exports = {
  config: {
    pokemon: 'random', // 他のポケモンに変更可能
    unibody: 'true', // ヘッダーと本体でデザインが区別できるようになります
    poketab: 'true', // タブに動くポケモンのアイコンが出現します。
    // 他の設定...
  },
  plugins: ["hyper-pokemon"],
};
```

適用できるポケモンのリストは[GitHubリポジトリ](https://github.com/klaudiosinani/hyper-pokemon)を参照してください。



## 5. 動作確認
Hyperを開き、テーマが適用されていることを確認してください。
テーマが正しく適用されていれば、セットアップ完了です！🎉

![alt text](/images/pokemon2.png)

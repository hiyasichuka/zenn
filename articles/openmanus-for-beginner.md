---
title: "OpenManusとは？AIエージェントの未来を変えるオープンソースプロジェクト"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["OpenManus","AI"]
published: true
---


## OpenManusとは

最近注目されている**Manus** という AI エージェントがありましたが、これは招待コードがないとアクセスできませんでした。
一方で、OpenManus は**その制約を取り払い、オープンな環境で誰でもAIエージェントを作成・改良できるようにした**点が特徴です。
OpenManus は、開発者たちが自由にフィードバックを出し合い、 プルリクエストを通じて改善を重ねながら進化していくプロジェクトとなっています。
逆にいうとコミュニティの参加者が増えないと廃れる可能性があります。

## OpenManusの利用手順

### Python環境をセットアップ

まず、Python 環境を作成します。

`conda` の場合は下記です。
```bash
conda create -n open_manus python=3.12
conda activate open_manus
```

`venv` の場合は下記です。
```bash
python -m venv open_manus
source open_manus/bin/activate
```

### OpenManusをダウンロード

```bash
git clone https://github.com/mannaandpoem/OpenManus.git
cd OpenManus
```

### 必要なライブラリをインストール

```bash
pip install -r requirements.txt
```

### OpenManusの設定ファイルの編集

config.py ファイルを開き、API キーなどの必要な設定を入力します。

### OpenManusを起動

```bash
python run_flow.py
```

### プロンプト入力

※　API キーを正しく設定できていないため、エラーしています。

![alt text](/images/image.png)


## まとめ

OpenManus は、誰でも自由に AI エージェントを開発・カスタマイズできるオープンソースプロジェクトです。
開発者自身が AI エージェントを構築し、柔軟に拡張できるのが大きな魅力です。

「AI エージェントを作りたい」「AI 活用をもっと身近にしたい」と考えている人にとって、 **OpenManusは要チェック** ですね。

# 参考資料

https://github.com/mannaandpoem/OpenManus
https://openmanus.org/

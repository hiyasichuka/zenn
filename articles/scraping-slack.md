---
title: "Slack APIとPythonでスレッド込みメッセージ収集を完全自動化する"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python","slack"]
published: true
---

## はじめに

Slackを業務ツールとして使用している会社員の皆様、いかがお過ごしでしょうか。

期末の評価面談やキャリアの棚卸しをするとき、**自分がどのような発信をしてきたのか（Slackの履歴）を手作業でさかのぼるのは膨大な時間**がかかりますよね。
これを解消するために、自分がSlackで発信したメッセージを一括で収集し、テキストデータとして活用できるローカルPCから実行するPythonプログラム（Notebook）を作りました。

収集したデータは、GeminiなどのLLMに渡して要約・分類・インサイト抽出などを自動化できるので、非常に効率的に自分の行動やマインドのふりかえりが可能となります。

この記事では、自分が投稿したメッセージを期間指定で取得し、スレッドも含めてCSVとして出力する方法を解説します。また、レートリミットの回避や接続瞬断によるエラーにも対応した、実運用を意識した実装例も合わせてご紹介します。

## 手順

### 1. 前提条件

- Python が使えること（`python --version` が通る）
- VS Code を想定（Python / Jupyter 拡張をインストール）
- [Slack API: Your Apps](https://api.slack.com/apps/) で App を作成し、**User Token Scopes** を設定して **User OAuth Token**（`xoxp-...`）を取得  



### 2. 作成物

プログラムのソースコードは下記となります。
適宜 `git clone` してご利用ください。

https://github.com/hiyasichuka/slack-msg-collector

### 3. アプリの作成とトークンの発行について

事前にSlack Appの作成の上、対象ワークスペースへインストールする必要があります。

#### アプリを作成する様子

![アプリ作成](/images/slack-msg-collector/create-new-app.png)

また、User OAuth Tokenには適切に権限を付与してください

#### 適切な権限を与えている様子

![権限](/images/slack-msg-collector/user-token-scope.png)

機微情報を含む `.env` はリポジトリで管理していないため、cpコマンドなどでファイルを新規作成してください。

Slackの **OAuth & Permissions → User OAuth Token** で発行されたトークンを `.env` に設定します。

```sh:.env
SLACK_TOKEN=xoxp-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 4. 環境構築

```sh
# Python
python --version
#Python 3.13.7

# 環境変数のセットアップ
cp .env.sample .env
# .envに発行したUser OAuth Token を貼り付けてください

# 実行環境のセットアップ
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 5. Notebookで実行

初回ノートブック実行時は、Python実行環境を選択する必要があります。下記画像の通り、先ほど用意した`.venv`を選択しましょう。

![実行環境選択](/images/slack-msg-collector/select-venv.png)

1. 設定セルで取得するチャンネルIDと取得期間（JSTで時刻まで可）を指定
1. notebook.ipynb を上から順に実行
1. 取得件数が表示され、my_slack_logs.csv が生成されます

#### テスト用にあるチャンネルに投稿したSlack

![Slack画面](/images/slack-msg-collector/skack-syoseki.png)


#### プログラム実行後にCSVとして出力されている
![実行結果](/images/slack-msg-collector/exec-syoseki.png)

### 出力CSVフォーマット

* `channel`: チャンネルID
* `text`: 投稿本文
* `timestamp`: `YYYY-MM-DD HH:MM:SS JST`
* `thread_ts`: 親スレッドのタイムスタンプ
* `is_thread_parent`: 投稿が親スレッドかどうか（true/false）


## 工夫ポイント

### 1. レート制限回避とリトライ対応

Slack APIは短時間の連続呼び出しでRateLimit（429）が発生します。
そこで、SDKの自動リトライと自前の保険を併用しています。


#### SDKの自動リトライ
```python
        retry_handlers = [
            RateLimitErrorRetryHandler(max_retry_count=3),
            ConnectionErrorRetryHandler(max_retry_count=3),
        ]
        self.client = WebClient(token=token, retry_handlers=retry_handlers, timeout=30)
```

### 自前のリトライ（指数バックオフ）

```python
def backoff_sleep(base=1.5, factor=1.8, attempt=0, max_sleep=30):
    sleep = min(max_sleep, base * (factor ** attempt)) * (0.5 + random())
    time.sleep(sleep)
```

```python
try:
    response = self.client.conversations_history(...)
except (IncompleteRead, httpx.ReadTimeout, httpx.RemoteProtocolError, urllib3.exceptions.ProtocolError) as e:
                        if attempt >= 3:
                            print(f"[{channel}] 再試行上限。historyスキップ: {e}")
                            return messages
                        attempt += 1    # attempt を増やしつつ backoff_sleep(...) して再試行
                        print(f"[{channel}] 断片読込/接続エラー。{attempt}回目のリトライ…")
                        backoff_sleep(attempt=attempt)
```


### 2. 自分のメッセージだけを抽出

Slackのレスポンスには全ユーザーのメッセージが含まれます。
`auth_test()` で自分の `user_id= を取得し、自分の投稿のみにフィルタしています。
メッセージ量が多い場合、処理に時間がかかる点はご了承ください。

### 3. スレッドの取得

親メッセージとスレッド（返信）の取得するAPIが異なります。
そのため、 `conversations_history` で親を確認後、`conversations_replies` でスレッド内を再取得しています。


## まとめ

本記事では、Slack APIを使って自分の投稿メッセージを期間指定で収集し、CSVとして保存する方法を紹介しました。
また下記の工夫点についても軽く触れました。

* User Tokenを使い、スレッド込みで自分の投稿を取得
* リミットレート制限/接続エラーに耐えるリトライ処理
* CSV形式によりBIやLLMへの投入が容易

この仕組みを発展させることで、任意ユーザーのメッセージ取得にも対応可能です。
Slackのログ活用を進めたい方は、ぜひこのサンプルをベースに試してみてください。

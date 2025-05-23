---
title: "API設計におけるHTTPステータスコード早見表"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["api","http"]
published: true
---

## 1. 成功レスポンス (2xx系)

| シーン | ステータス | 説明 |
|--------|------------|------|
| データ取得 (GET) の成功 | **200 OK** | リクエストが正常に処理され、期待するデータが返された。 |
| 新規リソース作成 (POST) の成功 | **201 Created** | リソースの作成が完了し、LocationヘッダーでリソースのURLを返す場合もある。 |
| 非同期処理の受理 (キュー投入など) | **202 Accepted** | リクエストは正常に受理されたが、処理はまだ完了していない。 |
| 更新・削除 (PUT/DELETE) の成功（レスポンスボディ無し） | **204 No Content** | 処理は正常に完了したが、返却するデータは無い。 |

## 2. クライアントエラー (4xx系)

| シーン | ステータス | 説明 |
|--------|------------|------|
| パラメータ不足・不正値・バリデーションエラー | **400 Bad Request** | リクエストに不正な値や形式エラーが含まれている。 |
| 認証失敗（未認証状態） | **401 Unauthorized** | 認証情報が不足または無効である。 |
| 認証済みだが権限不足 | **403 Forbidden** | 認証は成功しているが、当該リソースへのアクセス権が無い。 |
| 指定リソースが存在しない | **404 Not Found** | リクエストされたリソースが存在しない。 |
| クライアントが受け入れ可能なレスポンス形式が無い | **406 Not Acceptable** | Acceptヘッダーに基づくレスポンスが用意できない。 |
| リソースの状態が競合している（重複登録・バージョン不整合等） | **409 Conflict** | リクエストが現在のリソース状態と矛盾している。 |

## 3. サーバーエラー (5xx系)

| シーン | ステータス | 説明 |
|--------|------------|------|
| サーバー内部エラー（予期しない例外等） | **500 Internal Server Error** | アプリケーションサーバー内部の要因によるエラー。 |
| 外部APIやバックエンドサービスとの通信失敗 | **502 Bad Gateway** | 接続先のバックエンドサービスからエラー応答や無応答を受けた場合。 |
| サービス過負荷・メンテナンス等で一時的に利用不可 | **503 Service Unavailable** | サービスが一時的に利用不可（レート制限含む）。 |
| バックエンドの応答遅延によるタイムアウト | **504 Gateway Timeout** | API Gateway 等が指定時間内にバックエンドから応答を受け取れなかった。 |

## 4. リトライ制御・バックオフ設計時の目安

| シーン | ステータス | 説明 |
|--------|------------|------|
| 一時的な障害でリトライ推奨 | **503 Service Unavailable** | 一時的な障害や過負荷の可能性があるため、指数バックオフを用いたリトライ設計が推奨される（サーバーエラーとしても使用されるが、リトライ可能な状況を示す場合もある）。 |
| パラメータやリクエスト内容の修正が必要 | **400 Bad Request** | リクエストに不正な値や形式エラーが含まれており、ユーザーが修正する必要がある。 |
| 権限不足の修正が必要 | **403 Forbidden** | 認証は成功しているが、アクセス権が無いため、ユーザーが権限を修正する必要がある。 |

## 実践Tips

- アプリ内部のエラーは **500 Internal Server Error** を返す。
- POSTでリソース作成時は **201 Created** を返すのが正確。（200でごまかさない）。
- バリデーションエラー時は **400 Bad Request** を返し、詳細はJSONで明示する（例: `{"error": "name is required"}`）。
- レートリミット時は **429 Too Many Requests** を使う（Retry-Afterヘッダーを活用することで、いつ再試行してほしいか伝える）。

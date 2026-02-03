---
title: "Google Calendar APIのデータをCloud Composer経由でBigQueryにロードする"
emoji: "📅"
type: "tech"
topics: ["GoogleCloud", "BigQuery", "Composer", "Airflow", "Python"]
published: true
---

## はじめに

Google Calendar のイベントデータを BigQuery に取り込んでおくと、色々な分析に使えます。

- 祝日・休日データと売上を JOIN して「祝日は売上が伸びるか」を検証
- 社内カレンダーのイベント数と稼働時間の相関を可視化
- 会議の多い日と生産性指標の関係を分析

Google Calendar API を使えばデータ取得自体は簡単なんですが、Cloud Composer（Airflow）で実行しようとするとつまずきがありました。

この記事では、実際にハマったポイントと解決策を中心に、実装方法を整理します。

## 前提

- Cloud Composer 2.x（Airflow 2.x）
- BigQuery
- Secret Manager
- Python 3.11

## 公開カレンダーと組織内カレンダーの違い

当然ですが、会社ドメインに閉じたカレンダーは外部からアクセスできません。公開カレンダーと組織内カレンダーで実装方法が変わるので、先に整理しておきます。

### 公開カレンダー → 特別な認証は不要

日本の祝日カレンダー（`ja.japanese#holiday@group.v.calendar.google.com`）のような公開カレンダーは、誰でもアクセスできます。


### 組織内カレンダー → ドメインユーザーの認証が必要

Google Workspace の組織内カレンダーはドメイン制限がかかっているので、そのドメインのユーザーとして認証する必要があります。サービスアカウントでアクセスしようとすると `404 Not Found` が返ってきます。

```
# ドメイン外からアクセスした場合のエラー
googleapiclient.errors.HttpError: <HttpError 404 ... "Not Found">
```

組織内ユーザーの OAuth2 クレデンシャルを使う必要があります。

## 実装例

### 公開カレンダー（日本の祝日）を取得する DAG

Cloud Composer 環境では `google.auth.default()` で環境のサービスアカウントを使えます。

:::message
Google Calendar API の詳細は[公式リファレンス](https://developers.google.com/calendar/api/v3/reference)を参照してください。
:::

```python
"""Google Calendar API から日本の祝日を取得し、BigQuery にロードする DAG."""

from __future__ import annotations

import datetime
from typing import Any

import google.auth
from airflow.decorators import dag, task
from airflow.providers.google.cloud.hooks.bigquery import BigQueryHook
from google.cloud import bigquery
from googleapiclient.discovery import build

# =============================================================================
# 定数
# =============================================================================
GCP_CONN_ID: str = "google_cloud_default"
PROJECT_ID: str = "your-project-id"
DATASET_NAME: str = "your-dataset"
TABLE_NAME: str = "your-table"
TABLE_REF: str = f"{PROJECT_ID}.{DATASET_NAME}.{TABLE_NAME}"
CALENDAR_ID: str = "ja.japanese#holiday@group.v.calendar.google.com"

TIME_MIN_YEARS: int = 1  # 取得開始年（現在から何年前か）
TIME_MAX_YEARS: int = 1  # 取得終了年（現在から何年後か）

DEFAULT_ARGS: dict[str, Any] = {
    "owner": "your-team",
    "start_date": datetime.datetime(2024, 1, 1),
    "retries": 3,
    "retry_delay": datetime.timedelta(minutes=10),
}


def get_time_range() -> tuple[str, str]:
    """データ取得期間を計算する（JST基準）."""
    tz_jst = datetime.timezone(datetime.timedelta(hours=9))
    today = datetime.datetime.now(tz_jst).date()
    time_min = datetime.datetime(today.year - TIME_MIN_YEARS, 1, 1, 0, 0, 0, tzinfo=tz_jst)
    time_max = datetime.datetime(today.year + TIME_MAX_YEARS, 12, 31, 23, 59, 59, tzinfo=tz_jst)
    return time_min.isoformat(), time_max.isoformat()


def fetch_holidays() -> list[dict[str, str]]:
    """Google Calendar API から祝日を取得する."""
    credentials, _ = google.auth.default(
        scopes=["https://www.googleapis.com/auth/calendar.readonly"]
    )
    service = build("calendar", "v3", credentials=credentials)
    time_min, time_max = get_time_range()

    response = (
        service.events()
        .list(
            calendarId=CALENDAR_ID,
            timeMin=time_min,
            timeMax=time_max,
            singleEvents=True,
            orderBy="startTime",
        )
        .execute()
    )

    return [
        {
            "date": event["start"]["date"],
            "name": event["summary"],
        }
        for event in response.get("items", [])
        if event.get("summary") and "date" in event.get("start", {})
    ]


def load_to_bigquery(client: bigquery.Client, rows: list[dict[str, str]]) -> None:
    """BigQuery にデータをロードする（WRITE_TRUNCATE）."""
    job_config = bigquery.LoadJobConfig(
        write_disposition=bigquery.WriteDisposition.WRITE_TRUNCATE,
    )
    job = client.load_table_from_json(rows, TABLE_REF, job_config=job_config)
    job.result()


@dag(
    dag_id=f"{DATASET_NAME}.{TABLE_NAME}",
    default_args=DEFAULT_ARGS,
    description="日本の祝日を Google Calendar API から取得して BigQuery にロード",
    schedule="0 9 1 * *",  # 毎月1日 09:00 JST
    max_active_runs=1,
    catchup=False,
    tags=["calendar", DATASET_NAME],
)
def import_holidays_dag() -> None:
    @task
    def import_holidays_task() -> None:
        hook = BigQueryHook(gcp_conn_id=GCP_CONN_ID)
        client = hook.get_client(project_id=PROJECT_ID)

        holidays = fetch_holidays()

        if not holidays:
            raise ValueError("No holidays found")

        load_to_bigquery(client, holidays)

    import_holidays_task()


import_holidays_dag()
```

### 組織内カレンダーを取得する DAG

OAuth2 クレデンシャルを使う場合は、Airflow Connection 経由で認証情報を渡します。

#### Secret Manager に Connection を登録

Secret Manager に OAuth2 クレデンシャルを含む Connection を登録します。
Terraform で書くとこんな感じです。

:::message
Airflow Connection の URI 形式については [公式ドキュメント](https://airflow.apache.org/docs/apache-airflow-providers-google/stable/connections/gcp.html) を参照してください。
:::

```terraform
resource "google_secret_manager_secret_version" "google_cloud_calendar" {
  secret = google_secret_manager_secret.airflow_connections["google_cloud_calendar"].id
  secret_data = join(
    "&",
    [
      "google-cloud-platform://?extra__google_cloud_platform__credential_config_file=${urlencode(var.calendar_credentials)}",
      "extra__google_cloud_platform__scope=${urlencode("https://www.googleapis.com/auth/calendar.readonly")}",
      "extra__google_cloud_platform__project=${var.project_id}",
    ]
  )
}
```

#### DAG 側の実装

```python
"""組織内カレンダーを取得する DAG."""

from __future__ import annotations

import datetime
from typing import TYPE_CHECKING, Any

from airflow.decorators import dag, task
from airflow.providers.google.cloud.hooks.bigquery import BigQueryHook
from google.cloud import bigquery
from googleapiclient.discovery import build

if TYPE_CHECKING:
    from google.oauth2.credentials import Credentials

# OAuth2 クレデンシャルを持つ Connection を指定
GCP_CONN_ID: str = "google_cloud_calendar"
PROJECT_ID: str = "your-project-id"
DATASET_NAME: str = "your-dataset"
TABLE_NAME: str = "your-table"
TABLE_REF: str = f"{PROJECT_ID}.{DATASET_NAME}.{TABLE_NAME}"
CALENDAR_ID: str = "your-organization-calendar@group.calendar.google.com"


def fetch_calendar_events(credentials: Credentials) -> list[dict[str, Any]]:
    """Google Calendar API からイベントを取得する."""
    service = build("calendar", "v3", credentials=credentials)
    # ... 以下同様
```

`BigQueryHook(gcp_conn_id=GCP_CONN_ID).get_credentials()` で OAuth2 クレデンシャルを取得できます。

## ハマったポイントと対処法

実装中にいくつかエラーに遭遇したので、原因と対処法をまとめておきます。

### API のレスポンス件数制限

Google Calendar API の `events.list` はデフォルトで最大 250 件、`maxResults` パラメータを指定しても最大 2500 件までしか返しません。
それ以上のイベントがある場合は `nextPageToken` を使ってページネーションする必要があります。

```python
def fetch_calendar_events(credentials: Credentials) -> list[dict[str, Any]]:
    """Google Calendar API からイベントを取得する（ページネーション対応）."""
    service = build("calendar", "v3", credentials=credentials)
    time_min, time_max = get_time_range()

    all_events: list[dict[str, Any]] = []
    page_token: str | None = None

    while True:
        response = (
            service.events()
            .list(
                calendarId=CALENDAR_ID,
                timeMin=time_min,
                timeMax=time_max,
                singleEvents=True,
                orderBy="startTime",
                maxResults=2500,  # 1リクエストの最大取得件数
                pageToken=page_token,
            )
            .execute()
        )

        events = [
            {
                "event_name": event["summary"],
                "description": event.get("description", ""),
                "html_link": event.get("htmlLink", ""),
                "creator_email": event.get("creator", {}).get("email", ""),
                "start_datetime": _extract_datetime(event["start"]),
                "end_datetime": _extract_datetime(event["end"], is_end=True),
            }
            for event in response.get("items", [])
            if event.get("summary")
        ]
        all_events.extend(events)

        page_token = response.get("nextPageToken")
        if not page_token:
            break

    return all_events
```

:::message
詳細は [Events: list のドキュメント](https://developers.google.com/calendar/api/v3/reference/events/list) を参照してください。
:::

### `invalid_scope: Bad Request`

OAuth2 リフレッシュトークンを発行したときに `calendar.readonly` スコープを含めていなかったのが原因でした。

後からスコープを追加しても効かないので、必要なスコープを含めて再認証するか、カレンダー専用の Connection を新規作成する必要があります。

```python
# OAuth2 認証時に必要なスコープ
SCOPES = [
    "https://www.googleapis.com/auth/calendar.readonly"
]
```

:::message
OAuth2 のスコープについては [Google の OAuth 2.0 スコープ一覧](https://developers.google.com/identity/protocols/oauth2/scopes#calendar) を参照してください。
:::

### `404 Not Found`

カレンダーへのアクセス権限がない場合に発生します。

### `AttributeError: 'Credentials' object has no attribute 'with_scopes'`

`BigQueryHook.get_credentials()` で取得した Credentials オブジェクトには `with_scopes()` メソッドがありません。

- サービスアカウントの場合: `google.auth.default(scopes=[...])` でスコープ付きで取得
- OAuth2 の場合: Connection 設定時にスコープを指定しておく

## イベントの日時処理

Google Calendar API のイベントは2種類の日時形式を返してきます。

- **時刻指定イベント**: `dateTime` キーで `"2025-01-29T15:00:00+09:00"` のような ISO 8601 形式
- **終日イベント**: `date` キーで `"2025-01-29"` のような日付のみ

BigQuery にロードする際は DATETIME 型に統一したいので、終日イベントは開始を `00:00:00`、終了を `23:59:59` として変換しています。

:::message
Events リソースの詳細は [Events: list のレスポンス仕様](https://developers.google.com/calendar/api/v3/reference/events/list) を参照してください。
:::

```python
def _extract_datetime(event_time: dict[str, str], is_end: bool = False) -> str:
    """イベントの日時を DATETIME 形式で抽出する."""
    if "dateTime" in event_time:
        # 時刻指定イベント: "2025-01-29T15:00:00+09:00"
        dt = datetime.datetime.fromisoformat(event_time["dateTime"])
        return dt.strftime("%Y-%m-%dT%H:%M:%S")
    
    # 終日イベント: "date" キーのみ（例: "2025-01-29"）
    date = datetime.datetime.strptime(event_time["date"], "%Y-%m-%d")
    if is_end:
        # 終了日は 23:59:59 に設定
        return date.replace(hour=23, minute=59, second=59).strftime("%Y-%m-%dT%H:%M:%S")
    return date.strftime("%Y-%m-%dT%H:%M:%S")
```

## BigQuery テーブル設計例

### 祝日テーブル

```sql
CREATE TABLE IF NOT EXISTS `project.your_dataset.holidays` (
  date DATE NOT NULL,
  name STRING NOT NULL
)
OPTIONS (
  description = '日本の祝日マスタ'
);
```

### カレンダーイベントテーブル

```sql
CREATE TABLE IF NOT EXISTS `project.your_dataset.company_calendar` (
  event_name STRING NOT NULL,
  description STRING,
  html_link STRING,
  creator_email STRING,
  start_datetime DATETIME NOT NULL,
  end_datetime DATETIME NOT NULL
)
OPTIONS (
  description = '社内カレンダーイベント'
);
```

## まとめ

| カレンダー種別 | 認証方式 | 実装方法 |
|--------------|---------|---------|
| 公開カレンダー | サービスアカウント | `google.auth.default(scopes=[...])` |
| 組織内カレンダー | ドメインユーザーの OAuth2 | Airflow Connection + `BigQueryHook.get_credentials()` |


## 参考

- [Google Calendar API リファレンス](https://developers.google.com/calendar/api/v3/reference)
- [Events: list](https://developers.google.com/calendar/api/v3/reference/events/list) - イベント取得 API の詳細
- [OAuth 2.0 Scopes for Google APIs](https://developers.google.com/identity/protocols/oauth2/scopes#calendar) - カレンダー関連のスコープ一覧
- [Airflow Google Cloud Connection](https://airflow.apache.org/docs/apache-airflow-providers-google/stable/connections/gcp.html) - Connection URI の設定方法
- [Cloud Composer ドキュメント](https://cloud.google.com/composer/docs)

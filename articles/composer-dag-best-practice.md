---
title: "Cloud ComposerのDAGを最適化するTips"
emoji: "✨"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["ariflow","googlecloud","composer"]
published: true
---

[Cloud Composer 最適化ガイド（公式）](https://cloud.google.com/blog/ja/products/data-analytics/optimize-cloud-composer-via-better-airflow-dags/) を一読したんで、下記に整理します。

## はじめに（基礎知識）

AirflowやComposer、DAGについて知らない人向けに、簡単に説明しておきます。

- **Apache Airflowとは？**
  - ワークフロー（複数のタスクの流れ）をプログラムで管理・自動化するOSS。

- **Cloud Composerとは？**
  - Google Cloud上でApache Airflowを簡単に使えるように提供されているフルマネージドサービス。インフラ管理に煩わされず、ワークフローの作成や実行に集中できます。

- **DAGとは？**
  - Directed Acyclic Graphの略で、Airflowで処理を実行する際のタスクの流れを定義したもの。

---

## Tips20選

### 1. わかりやすいファイル名にする
- DAGの目的、頻度、担当チームなどがわかるようにする。
```bash
daily_sales_report_marketing.py
```

### 2. 冪等性（再実行可能性）を意識する
- 何回実行しても同じ結果が得られるようタスクをつくる。
```sql
MERGE INTO table USING source ON table.id = source.id
WHEN MATCHED THEN UPDATE SET...
WHEN NOT MATCHED THEN INSERT...
```

### 3. 複雑な処理はDAGの外で定義する
- DAG定義内に複雑な処理を書かない。


### 4. Jinjaテンプレート利用する
- 日付や変数を動的に設定可能
```python
query = "SELECT * FROM table WHERE date = '{{ ds }}'"
```

### 5. カスタムOperatorを活用する
- よく使う処理を再利用可能にする。
```python
class CustomOperator(BaseOperator):
    def execute(self, context):
        # カスタム処理
        pass
```

### 6. Pythonパッケージを使って整理する
- 再利用可能なコードは別パッケージ化する。

```
dags/
├── common_package/  # 再利用可能な処理を入れるディレクトリ
│   ├── __init__.py
│   ├── data_utils.py
│   └── data_queries.py
└── daily_sales_report.py # common_packageの処理を呼び出す
``` 

### 7. Airflow Variablesで環境変数を管理する
```python
from airflow.models import Variable
db_url = Variable.get("DATABASE_URL")
```

### 8. Sensorの間隔調整する
- ポーリング頻度を調整して無駄な負荷を減らす。

:::message
 Airflowでは、DAGの中で「他のDAGの処理が完了するのを待つ」タスクとして Sensor（センサー） があります。このSensorが、指定した間隔で外部の状態をチェックするための頻度を「ポーリング頻度」といいます。
:::


```python
sensor = ExternalTaskSensor(
    poke_interval=300
)
```


### 9. DAGのタスクを並列実行可能にする
- DAGの依存関係を整理して並列化。
```python
task_1 >> [task_2, task_3] >> task_4
```

### 10. DAGのキャッチアップを無効化
- 過去のタスクの自動再実行を防ぐ。
```python
dag = DAG(catchup=False)
```

### 11. スケジュールを適切に設定する
- 必要最低限の頻度にする。
- https://crontab.guru/ を利用して、人でもわかるようコメントに頻度を明記しておく

```python
schedule_interval="0 8 * * *" # At 08:00
```

### 12. 再試行（リトライ）の設定しておく
- 失敗した時の動きを明確にしておく。
```python
retries=3,
retry_delay=timedelta(minutes=5)
```

### 13. DAGを一時停止状態で作成
```python
dag = DAG(is_paused_upon_creation=True)
```

### 14. 明確なドキュメントを追加
- DAGの目的や処理内容をdocstringで記述する。


```python:サンプルDAG
from airflow import DAG
from airflow.operators.python_operator import PythonOperator
from datetime import datetime

"""
DAG名: daily_sales_report
目的: 毎日の売上レポートを作成し、Google Driveにアップロードする。
処理内容:
  1. BigQueryから昨日の売上データを抽出
  2. 売上データをCSVに変換
  3. CSVファイルをGoogle Driveへアップロード
"""

default_args = {
    'owner': 'sales_analytics_team',
    'start_date': datetime(2024, 1, 1)
}

dag = DAG(
    dag_id='daily_sales_report',
    schedule_interval='0 8 * * *',  # 毎日午前8時
    default_args=default_args,
    catchup=False,
)

def extract_sales_data(**kwargs):
    """BigQueryから売上データを抽出する処理"""
    pass

def upload_report_to_drive(**kwargs):
    """生成したレポートをGoogle Driveにアップロードする処理"""
    pass

extract_task = PythonOperator(
    task_id='extract_sales_data',
    python_callable=extract_sales_data,
    dag=dag,
)

upload_task = PythonOperator(
    task_id='upload_report',
    python_callable=upload_report_to_drive
)

extract_sales_data >> upload_report_to_drive

```

### 15. タスクのオーナーを明記する

```python
default_args = {"owner": "data_team"}
```

### 16. トリガールールを活用する

```python
trigger_rule='all_done'
```

### 17. 静的な開始日付を設定
- datetime.now()など動的日付を避けることで予期しない実行を防ぐ。

### 18. タイムアウトを設定
- DAGが長時間動き続けるのを防ぐ。
```python
dagrun_timeout=timedelta(hours=2)
```

### 19. ログの標準化
- トラブルシューティングを楽にするため、ログフォーマット統一する。
```python
import logging
logging.info("Start processing: %s", execution_date)
```

### 20. DAGのタグ付け
- UI上でフィルターや分類を容易にする。
```python
dag = DAG(tags=['sales', 'daily'])
```

# おわりに

Cloud Composerを活用することで、データ処理の自動化を実現できます。
しかし、適切な設定を行わないと、パフォーマンス低下や運用負担の増加につながります。
本記事で紹介したTipsを取り入れることで、パフォーマンス向上や運用負担の軽減していきましょう。
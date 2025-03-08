---
title: "Cloud ComposerのDAGを最適化するTips"
emoji: "✨"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["ariflow"]
published: false
---

[Cloud Composer 最適化ガイド（公式）](https://cloud.google.com/blog/ja/products/data-analytics/optimize-cloud-composer-via-better-airflow-dags/) を一読したんで、下記に整理します。

## はじめに（基礎知識）

AirflowやComposer、DAGについて知らない人向けに、簡単に補足しておきます。

- **Apache Airflowとは？**
  - ワークフロー（複数のタスクの流れ）をプログラムで管理・自動化するOSS。

- **Cloud Composerとは？**
  - Google Cloud上でApache Airflowを簡単に使えるように提供されているフルマネージドサービス。インフラ管理に煩わされず、ワークフローの作成や実行に集中できます。

- **DAGとは？**
  - Directed Acyclic Graphの略で、Airflowで処理を実行する際のタスクの流れを定義したもの。

---

## Tips20選

### 1. わかりやすいファイル名にする
- DAGの目的、頻度などがわかるようにする。
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

### 3. 複雑な処理はDAGの外で
- DAG定義内に複雑な処理を書かない。


### 4. Jinjaテンプレート利用する
- 動的な日付や変数を簡単に扱える。
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
- 共通のコードは別パッケージ化する。

### 7. Airflow Variablesで環境変数を管理する
```python
from airflow.models import Variable
db_url = Variable.get("DATABASE_URL")
```

### 8. Sensorの間隔調整する
- ポーリング頻度を調整して無駄な負荷を減らす。
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
- https://crontab.guru/ を利用して、人間でもわかるように実行タイミングをコメントに明記しておく

```python
schedule_interval="0 8 * * *" # At 08:00
```

### 12. 再試行（リトライ）の設定しておく
- 失敗した時の動きを明確に。
```python
retries=3,
retry_delay=timedelta(minutes=5)
```

### 13. DAGを一時停止状態で作成
```python
dag = DAG(is_paused_upon_creation=True)
```

### 14. 明確なドキュメントを追加
- DAGの目的や処理内容をdocstringで記述。

### 15. タスクのオーナーを明記する

```python
default_args = {"owner": "data_team"}
```

### 16. トリガールールを活用する
- タスクの実行条件を細かく制御。
```python
trigger_rule='all_done'
```

### 17. 静的な開始日付を設定
- current_date()など動的日付を避けることで予期しない実行を防ぐ。

### 18. タイムアウトを設定
- DAGが長時間動き続けるのを防ぐ。
```python
dagrun_timeout=timedelta(hours=2)
```

### 19. ログの標準化
- 一貫性のあるログでトラブルシューティングを楽に。
```python
import logging
logging.info("Start processing: %s", execution_date)
```

### 20. DAGのタグ付け
- UI上でフィルターや分類を容易にする。
```python
dag = DAG(tags=['sales', 'daily'])
```


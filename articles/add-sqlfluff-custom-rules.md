---
title: 'SQLFluffカスタムルールの作成とCI統合方法'
emoji: '📘'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## はじめに

SQL の品質管理において、組織固有のルールを適用したいケースは少なくありません。本記事では、SQLFluff のカスタムルールプラグインを作成し、GitHub Actions CI に統合する方法を解説します。

## 背景

SQLFluff は BigQuery をはじめとする様々な SQL クエリに対応したリンター・フォーマッターです。
標準ルールは充実していますが、独自の SQL ルールを強制したい場合、カスタムルールが必要になります。

### 実装したカスタムルール

今回は例として以下のルールを実装します

**CUSTOM_L001**: `CROSS JOIN` の使用禁止

- 理由: 意図しない全件結合によるパフォーマンス低下を防ぐため
- 対策: 明示的な JOIN 条件の使用を促す

## プロジェクト構造

```
sqlfluff-plugins/
├── pyproject.toml                    # パッケージ設定
├── MANIFEST.in                       # 設定ファイルの配布指定
├── README.md                         # プラグイン説明
├── src/
│   └── custom_rules/
│       ├── __init__.py               # プラグイン登録
│       ├── rules.py                  # ルール実装
│       └── plugin_default_config.cfg # デフォルト設定
└── test/
    └── rules/
        ├── rule_test_cases_test.py   # テスト実行
        └── test_cases/               # YAMLテストケース
            └── custom_l001.yml
```

### 見慣れないファイルの説明

**MANIFEST.in (ファイルの配布設定)**

Python パッケージをビルドする際、デフォルトでは `.py` ファイルのみが含まれます。`.cfg` や `.yml` などの設定ファイルを配布パッケージに含めるには、`MANIFEST.in` で明示的に指定する必要があります。

```
# MANIFEST.in の例
include src/custom_rules/plugin_default_config.cfg
```

これにより、`pip install` 時に設定ファイルもインストールされます。

**hookimpl (プラグイン登録の仕組み)**

SQLFluff は [pluggy](https://pluggy.readthedocs.io/) というプラグインシステムを使用しています。`@hookimpl` デコレータを使うことで、「このメソッドは SQLFluff のプラグイン用の実装です」と宣言できます。

主な hook メソッド

- `get_rules()`: カスタムルールのクラスリストを返す
- `load_default_config()`: プラグインのデフォルト設定を読み込む

SQLFluff が起動時にこれらのメソッドを自動的に検出・実行し、カスタムルールが利用可能になります。

## 実装手順

### 1. プラグインパッケージの設定

`pyproject.toml` でプラグインのエントリーポイントを定義します:

```toml
[project]
name = "sqlfluff-plugin-custom-rules"
version = "0.1.0"
dependencies = ["sqlfluff>=3.3.0"]

[project.entry-points.sqlfluff]
custom_rules = "custom_rules"

[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"
```

`MANIFEST.in` で設定ファイルを配布対象に含めます:

```
include src/custom_rules/plugin_default_config.cfg
```

### 2. プラグインの hook 実装

`src/custom_rules/__init__.py` で SQLFluff のプラグインシステムに接続します:

```python
"""Custom SQLFluff rules plugin."""

from typing import Any

from sqlfluff.core.config import load_config_resource
from sqlfluff.core.plugin import hookimpl
from sqlfluff.core.rules import BaseRule


@hookimpl
def get_rules() -> list[type[BaseRule]]:
    """Get plugin rules.

    NOTE: Rules are imported only on fetch to manage import times
    when rules aren't used.
    """
    from custom_rules.rules import Rule_CUSTOM_L001

    return [Rule_CUSTOM_L001]


@hookimpl
def load_default_config() -> dict[str, Any]:
    """Loads the default configuration for the plugin."""
    return load_config_resource(
        package="custom_rules",
        file_name="plugin_default_config.cfg",
    )
```

### 3. カスタムルールの実装

`src/custom_rules/rules.py` にルールロジックを実装します:

```python
"""Custom SQLFluff rules."""

from typing import Optional

from sqlfluff.core.rules import BaseRule, LintResult, RuleContext
from sqlfluff.core.rules.crawlers import SegmentSeekerCrawler


class Rule_CUSTOM_L001(BaseRule):
    """CROSS JOIN の使用を禁止.

    **アンチパターン**

    .. code-block:: sql

        SELECT * FROM table1
        CROSS JOIN table2

    **ベストプラクティス**

    .. code-block:: sql

        SELECT * FROM table1
        INNER JOIN table2 ON table1.id = table2.id
    """

    name = "custom.no_cross_join"
    groups = ("all",)
    crawl_behaviour = SegmentSeekerCrawler({"from_clause"})

    def _eval(self, context: RuleContext) -> Optional[LintResult]:
        """Evaluate."""
        assert context.segment.is_type("from_clause")

        raw_upper = context.segment.raw.upper()
        if "CROSS JOIN" in raw_upper:
            return LintResult(
                anchor=context.segment,
                description="CROSS JOIN は推奨されません。明示的なJOIN条件を使用してください。",
            )

        return None
```

### 4. YAML ベースのテストケース

`test/rules/test_cases/custom_l001.yml`:

```yaml
rule: CUSTOM_L001

test_cross_join_fail:
  fail_str: |
    SELECT *
    FROM table1
    CROSS JOIN table2

test_inner_join_pass:
  pass_str: |
    SELECT *
    FROM table1
    INNER JOIN table2 ON table1.id = table2.id
```

`test/rules/rule_test_cases_test.py`:

```python
"""Test cases for custom rules."""

import pytest
from sqlfluff.utils.testing.rules import load_test_cases

@pytest.mark.parametrize(
    "test_case",
    load_test_cases(
        test_cases_path="test_cases/custom_l001.yml",
    ),
    ids=lambda case: case.name,
)
def test_custom_l001(test_case):
    """Test CUSTOM_L001."""
    test_case.assert_rule_pass_in_sql()
```

### 5. GitHub Actions CI 統合

プラグインのテスト用ワークフロー (`.github/workflows/test-sqlfluff-plugins.yml`):

```yaml
name: Test SQLFluff Plugins

on:
  pull_request:
    paths:
      - 'sqlfluff-plugins/**/*.py'
      - 'sqlfluff-plugins/**/*.yml'
      - 'sqlfluff-plugins/**/pyproject.toml'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.13'
      - name: Install dependencies
        run: |
          pip install pytest
          pip install ./sqlfluff-plugins
      - name: Run tests
        working-directory: sqlfluff-plugins
        run: pytest test/ -v
```

SQL リント用ワークフロー (既存の `infra-precheck.yml` に統合):

```yaml
lint_sql:
  needs: changes
  if: ${{ needs.changes.outputs.sql == 'true' }}
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v5
      with:
        fetch-depth: 0
    - run: |
        pip install sqlfluff==${{ env.SQLFLUFF_VERSION }}
        pip install ./sqlfluff-plugins
    - name: SQL Linting with sqlfluff
      shell: bash -euo pipefail {0}
      run: |
        merge_base=$(git merge-base origin/main HEAD)
        files=$(git diff "$merge_base" --name-only --diff-filter=d -- '*.sql')
        for file in $files; do
          echo "File name: $file"
          sqlfluff lint --ignore parsing "$file"
        done
```

## 運用のポイント

### ルールの命名規則

SQLFluff のルール命名規則に従う必要があります:

- クラス名: `Rule_<PREFIX>_<CODE>` (例: `Rule_CUSTOM_L001`)
- ルール識別子: `<prefix>.<name>` (例: `custom.no_cross_join`)
- コード: 正規表現 `[A-Z0-9]{4}` にマッチする 4 文字 (例: `L001`)

### テストの重要性

YAML ベースのテストケースにより、以下を検証できます:

- `fail_str`: ルール違反として検出されるべき SQL
- `pass_str`: ルールに準拠した SQL

最低限以下のケースをカバーすることを推奨します:

1. 基本的な違反ケース
2. 適切に対処された合格ケース
3. ルールの対象外となるケース

## ハマりポイントと解決策

### 1. ConfigInfo インポートエラー

初期実装で `ConfigInfo` を import していましたが、SQLFluff 3.3.0 では不要でした:

```python
# ❌ 不要なインポート
from sqlfluff.core.rules import BaseRule, ConfigInfo

# ✅ 必要最小限
from sqlfluff.core.rules import BaseRule
```

`get_configs_info()` hook はカスタム設定が必要な場合のみ実装すればよく、今回は不要でした。

### 2. ルール命名の失敗

当初 `Rule_CUSTOM_CROSSJOIN` のような命名を試みましたが、正規表現 `Rule_?([A-Z]{1}[a-zA-Z]+)?_([A-Z0-9]{4})` にマッチせずエラーになりました。4 文字コードが必須です。

### 3. ワークフローの分離

プラグインのユニットテスト (開発用) と、実際の SQL リント (品質チェック) は目的が異なるため、別ワークフローに分離しました:

- `test-sqlfluff-plugins.yml`: プラグイン自体のテスト
- `infra-precheck.yml`: SQL ファイルに対するリント

## まとめ

SQLFluff のプラグインシステムを活用することで、組織固有のルールを型通りに実装・運用できます。重要なポイントは:

1. **公式のプラグイン例を参考にする**: SQLFluff リポジトリの `plugins/sqlfluff-plugin-example` は最良の教材
2. **適切なテストカバレッジ**: YAML ベースのテストで様々なケースを検証
3. **CI 統合**: プラグインテストと実運用リントを分離
4. **段階的な導入**: まずは重要度の高いルールから実装

今後、ルールを追加する場合も `rules.py` に新しいクラスを追加し、`__init__.py` の `get_rules()` に登録するだけです。

## 参考資料

- [SQLFluff 公式ドキュメント](https://docs.sqlfluff.com/)
- [SQLFluff Plugin Guide](https://docs.sqlfluff.com/en/stable/perma/plugin_guide.html)
- [SQLFluff Plugin Development](https://docs.sqlfluff.com/en/stable/perma/plugin_dev.html)
- [sqlfluff-plugin-example](https://github.com/sqlfluff/sqlfluff/tree/main/plugins/sqlfluff-plugin-example)

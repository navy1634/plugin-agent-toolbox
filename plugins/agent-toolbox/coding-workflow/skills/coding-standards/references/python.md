---
name: coding-standards
description: Python 3.12+ 開発向けのコーディング標準、ベストプラクティス、およびパターン。PEP 8、ruff、mypy 準拠。
---

# Python Coding Standards

Python 3.12+ に固有のコーディング標準です。共通の構造、formatter、コメント、変更範囲の契約は親 skill に従い、振る舞い変更のテスト設計・mock・coverage は [TDD workflow](../../tdd-workflow/SKILL.md) に従います。特定技術に依存するアプリケーション設計は [backend-patterns](../../backend-patterns/SKILL.md) を正本とします。

## Python language conventions

Python での名前、型注釈、import、Enum、可変値の記法と具体例を扱います。共通の命名・型・可変性の原則は親 skill の [Naming](../SKILL.md#naming)、[Types and Public Contracts](../SKILL.md#types-and-public-contracts)、[State and Mutability](../SKILL.md#state-and-mutability) を正本とします。

### Naming examples

```python
# ✅ 良い例: 意図が明確な名前（snake_case）
market_search_query = 'election'
is_user_authenticated = True
total_revenue = 1000
user_data = fetch_user_data()

# ❌ 悪い例: 意図が不明な名前
q = 'election'
flag = True
x = 1000
d = get_data()
```

### Function naming examples

```python
# ✅ 良い例: 動詞と目的語を含む名前（snake_case）
async def fetch_market_data(market_id: str) -> Market:
    pass

def calculate_similarity(vector_a: list[float], vector_b: list[float]) -> float:
    pass

def is_valid_email(email: str) -> bool:
    pass

# ❌ 悪い例: 意図が不明、または名詞だけの名前
async def market(market_id: str) -> Market:
    pass

def similarity(vector_a: list[float], vector_b: list[float]) -> float:
    pass

def email(address: str) -> bool:
    pass
```

### Constant naming examples

```python
# ✅ 良い例: 定数は UPPER_CASE
MAX_RETRIES = 3
API_TIMEOUT_SECONDS = 30
DEFAULT_PAGE_SIZE = 20

# ❌ 悪い例: 定数として読めない名前
max_retries = 3
api_timeout = 30
```

### Mutable value examples

Python の dict/list を変更しない例と、所有権・公開契約が明確な場合の可変操作の構文例を示します。

```python
# ✅ 良い例: 既存の値を変更せず、新しい値を作成する
user_updated = {**user, 'name': 'New Name'}
items_updated = [*items, new_item]

# 複数の更新も辞書展開で新しい値を作成する
config_updated = {**config, 'debug': True, 'timeout': 60}

# ❌ 悪い例: 呼び出し元が所有する値を直接変更する
user['name'] = 'New Name'
items.append(new_item)
```

次のような変更は例外として認められます。ただし、変更される値の所有者と、変更しない代替手段を検討した理由が仕様またはコメントから分かる場合に限ります。

```python
# ✅ 例外: この関数内で作成した値を、追加のコピーなしに組み立てる
work_items: list[Item] = []
work_items.extend(new_items)

def extend_items(items: list[Item], new_items: list[Item]) -> None:
    """仕様に従って呼び出し元のリストを更新する"""
    items.extend(new_items)
```

### Type syntax and annotation examples

Python 3.12+ の built-in generic、union syntax、`Any` を使った型注釈の具体例を示します。公開境界の共通契約は親 skill の [Types and Public Contracts](../SKILL.md#types-and-public-contracts) に従います。

```python
# ✅ 良い例: 引数と返り値を完全に注釈する
def get_market(market_id: str) -> Market | None:
    """市場を識別子で取得する"""
    pass

async def create_user(email: str, name: str) -> User:
    """ユーザーを作成する"""
    pass

def filter_markets(markets: list[Market], status: str) -> list[Market]:
    """状態で市場を絞り込む"""
    pass

# ❌ 悪い例: 型注釈がなく、mypy で検査できない
def get_market(id):
    pass

def create_user(email, name):
    pass
```

### Python syntax and import constraints

- Python 3.12+ では built-in generic（`list[str]` など）と union syntax（`str | None` など）を使う
- `typing` module の import は禁止し、`from typing import Any` だけを既定の例外とする。第三者 API の互換性などで別の構文が不可避な場合は、明示的なユーザー承認を得て理由を記録する
- 通常の production code で関数内に関数を定義しない。callback、decorator、closure など、外部 API または既存の設計が lexical scope の保持を要求する場合だけ例外とし、抽出できない理由を記録する
- import はファイル先頭に置く。optional dependency の遅延ロード、解消できない import cycle、起動時に読み込めない重い依存など、関数内 import が不可避な場合だけ例外とし、対象、理由、失敗時の扱いを記録する
- 全角の括弧や記号をコード・コメントへ入れない（RUF003 compliance）

### Enum syntax for grouped values

Python 3.11+ では `Enum` と `StrEnum` を使い、関数の引数や返り値にも Enum 型を注釈します。グループ化した値に Enum 相当の型を使う共通原則は、親 skill の [Grouped Values and Observability](../SKILL.md#grouped-values-and-observability) に従います。

```python
from enum import StrEnum


class MarketStatus(StrEnum):
    """市場の状態"""

    ACTIVE = 'active'
    RESOLVED = 'resolved'
    CLOSED = 'closed'


def is_open_market(status: MarketStatus) -> bool:
    """市場が受付中か確認する"""
    return status is MarketStatus.ACTIVE
```

## Python code patterns

Python 標準の例外、logging、asyncio、module/package の配置と命名の具体例を扱います。

### Exceptions and logging

Python の例外型、`raise ... from`、logging の具体例を示します。共通の失敗処理は親 skill の [Errors and Failure Handling](../SKILL.md#errors-and-failure-handling) に従います。

```python
# ✅ 良い例: 境界ごとに例外を処理し、原因を保持して利用者向け例外へ変換する
import json
import logging

def parse_payload(payload: str) -> dict[str, object]:
    """外部から受け取った payload を辞書へ変換する"""
    try:
        parsed = json.loads(payload)
        if not isinstance(parsed, dict):
            raise TypeError('payload は object である必要があります')
        return parsed

    except (json.JSONDecodeError, TypeError) as error:
        logging.error('payload の解析に失敗しました: %s', error)
        raise ValueError('payload の形式が不正です') from error

    except Exception:
        logging.exception('予期しないエラーが発生しました')
        raise

# ❌ 悪い例: 外部境界の失敗を処理しない
def parse_payload(payload: str) -> dict[str, object]:
    return json.loads(payload)
```

### Asyncio examples

Python の `asyncio.gather` を使った並行処理の具体例を示します。共通の依存順序と失敗時の扱いは親 skill の [Concurrency and Ordering](../SKILL.md#concurrency-and-ordering) に従います。

```python
# ✅ 良い例: 依存関係がない処理は並列に実行する
import asyncio

async def fetch_all_data() -> tuple[list[User], list[Market], list[Stat]]:
    """複数のデータを並列に取得する"""
    users, markets, stats = await asyncio.gather(fetch_users(), fetch_markets(), fetch_stats())
    return users, markets, stats

# ❌ 悪い例: 依存関係がない処理を順番に待つ
async def fetch_all_data() -> tuple[list[User], list[Market], list[Stat]]:
    users = await fetch_users()
    markets = await fetch_markets()
    stats = await fetch_stats()
    return users, markets, stats
```

### Python module and package layout examples

Python の module、package、test の配置とファイル命名の具体例を示します。実際のレイアウトは既存プロジェクトの規約に従います。

#### Project structure

```
src/
├── __init__.py
├── main.py                      # entry point
└── market.py                    # Python module
tests/
├── __init__.py
├── unit/
├── integration/
└── e2e/
```

#### File naming

```
src/market.py                     # module は snake_case
tests/unit/test_market_service.py # test_*.py または *_test.py
```

## Python documentation

Python のコメントと Google Style／PEP 257 docstring の具体例、module docstring の扱いを示します。文書の共通契約は親 skill の [Documentation](../SKILL.md#documentation) に従います。

### Comment examples

```python
# ✅ 良い例: 何をするかではなく、なぜそうするかを説明する
# 障害中に API へ負荷を集中させないため指数バックオフを使う
delay = min(1000 * (2 ** retry_count), 30000)

# 大きなリストでの性能を優先し、あえてミュータブルな操作にしている
items.extend(new_items)

# ❌ 悪い例: コードをそのまま言い換える
# カウンタを1増やす
counter += 1

# name にユーザー名を代入する
name = user.name
```

### Google Style and PEP 257 docstring examples

```python
# ✅ 良い例: すべての公開関数に docstring を付ける
def calculate_similarity(vector_a: list[float], vector_b: list[float]) -> float:
    """2つのベクトルのコサイン類似度を計算する

    Args:
        vector_a: 1つ目の浮動小数点ベクトル
        vector_b: 2つ目の浮動小数点ベクトル

    Returns:
        0から1までの類似度

    Raises:
        ValueError: ベクトルの長さが異なるか、空の場合

    Example:
        >>> similarity = calculate_similarity([1, 0, 0], [0, 1, 0])
        >>> similarity
        0.0
    """
    pass

# ✅ 良い例: class にも docstring を付ける
class Market:
    """予測市場を表す

    Attributes:
        id: 市場を一意に識別する値
        name: 市場名
        status: 現在の市場状態（active、resolved、closed）
    """

    id: str
    name: str
    status: str
```

### Python documentation conventions

- docstring は Google Style で記述し、Python の一般的な規約として PEP 257 に従う
- モジュールレベルの docstring は記載しない（ファイル冒頭に `"""..."""` を置かない）
- docstring、コメント、ドキュメントは日本語で記述し、日本語文末に `。` を付けない
- Google Style の `Args:`、`Returns:`、`Raises:`、`Yields:` などの見出しを使い、型は Python の型注釈へ記載する
- docstring の改行、インデント、空白、行幅は formatter の出力に従い、エージェントの判断で固定しない

```python
def create_user(name: str, age: int) -> dict[str, str | int]:
    """ユーザーを作成する

    Args:
        name: ユーザー名
        age: ユーザーの年齢

    Returns:
        作成されたユーザー情報を含む辞書

    Raises:
        ValueError: age が 0 未満の場合
    """
```

## Python testing examples

pytest、fixture、mock、Testcontainers、coverage、RED-GREEN-REFACTOR、DoD は [TDD workflow](../../tdd-workflow/SKILL.md) とその reference を正本とします。ここでは Python の AAA と振る舞いベースのテスト名の具体例だけを示します。

### AAA examples

```python
def test_calculate_similarity_identical_vectors() -> None:
    """同一のベクトルの類似度を検証する"""
    # Arrange: 入力を準備する
    vector_a = [1, 0, 0]
    vector_b = [1, 0, 0]

    # Act: 公開関数を実行する
    similarity = calculate_similarity(vector_a, vector_b)

    # Assert: 仕様上の結果を検証する
    assert similarity == 1.0

def test_calculate_similarity_orthogonal_vectors() -> None:
    """直交するベクトルの類似度を検証する"""
    # Arrange: 入力を準備する
    vector_a = [1, 0, 0]
    vector_b = [0, 1, 0]

    # Act: 公開関数を実行する
    similarity = calculate_similarity(vector_a, vector_b)

    # Assert: 仕様上の結果を検証する
    assert similarity == 0.0

def test_calculate_similarity_invalid_vectors() -> None:
    """長さが異なる入力が拒否されることを検証する"""
    # Act: 仕様上エラーになる入力で公開関数を実行する
    try:
        calculate_similarity([1, 0], [1, 0, 0])
    except ValueError:
        pass
    else:
        raise AssertionError('ValueError が送出されませんでした')
```

### Behavior-based test naming examples

```python
# ✅ 良い例: 検証する振る舞いが分かるテスト名
def test_returns_empty_list_when_no_markets_match_query() -> None:
    pass

def test_raises_error_when_openai_api_key_missing() -> None:
    pass

def test_falls_back_to_substring_search_when_redis_unavailable() -> None:
    pass

# ❌ 悪い例: 何を検証するか分からないテスト名
def test_works() -> None:
    pass

def test_search() -> None:
    pass
```

## Python complexity and performance examples

Python での長大な関数、深いネスト、magic number の具体例を示します。共通の複雑度と保守性の原則、基本閾値は親 skill の [Complexity and Maintainability](../SKILL.md#complexity-and-maintainability) に従います。

### Long function example

```python
# ❌ 悪い例: 50行を超える関数
def process_market_data(data: dict) -> Market:
    # 100行の処理
    pass

# ✅ 良い例: 小さな関数へ分割する
def process_market_data(raw_data: dict) -> Market:
    """未加工の市場データを処理する"""
    validated = validate_market_data(raw_data)
    transformed = transform_market_data(validated)
    return save_market(transformed)

def validate_market_data(data: dict) -> dict:
    """市場データを検証する"""
    pass

def transform_market_data(data: dict) -> Market:
    """市場 entity へ変換する"""
    pass
```

### Deep nesting example

```python
# ❌ 悪い例: 5段階以上の深いネスト
def check_permission(user: User | None, market: Market | None) -> bool:
    if user:
        if user.is_admin:
            if market:
                if market.is_active:
                    if has_permission(user, market):
                        # 処理を実行する
                        pass

# ✅ 良い例: early return でネストを浅くする
def check_permission(user: User, market: Market) -> bool:
    """ユーザーが権限を持つか確認する"""
    if not user:
        return False
    if not user.is_admin:
        return False
    if not market:
        return False
    if not market.is_active:
        return False

    return has_permission(user, market)
```

### Magic number example

```python
# ❌ 悪い例: 意味が説明されていない数値
if retry_count > 3:
    raise Exception('Max retries exceeded')

time.sleep(0.5)

# ✅ 良い例: 名前付き定数を使う
MAX_RETRIES = 3
DEBOUNCE_DELAY_MS = 500

if retry_count > MAX_RETRIES:
    raise MaxRetriesExceeded()

time.sleep(DEBOUNCE_DELAY_MS / 1000)
```

### Python performance examples

Python の import、list comprehension、generator による性能上の具体例を示します。性能に関する共通原則は親 skill の [Performance](../SKILL.md#performance) に従います。

#### Import placement for heavy modules

import はファイル先頭に置くことを原則とし、関数内 import を通常の遅延最適化として使いません。optional dependency の遅延ロード、解消できない import cycle、起動時に読み込めない重い依存など、関数内で読み込む必要が明確な場合だけ例外とします。例外を選ぶときは、理由、対象 dependency、失敗時の扱いを日本語コメントまたはドキュメントに残します。

```python
# ✅ 例外: この処理を使わない起動経路で重い optional dependency を読み込まない
def analyze_with_ml(data: list[float]) -> float:
    """機械学習モデルでデータを分析する"""
    import numpy as np

    return float(np.mean(data))
```

```python
# ✅ 良い例: 頻繁に使う dependency はファイル先頭で import する
import numpy as np

def process_arrays(arrays: list[np.ndarray]) -> np.ndarray:
    """複数の配列を処理する"""
    return np.concatenate(arrays)
```

#### List comprehensions over loops

```python
# ✅ 良い例: 単純な変換は list comprehension を使う
markets_active = [m for m in markets if m.status == 'active']
market_names = [m.name for m in markets]
tuples = [(m.id, m.name) for m in markets]

# ❌ 悪い例: 単純な変換に手動 loop を使う
markets_active = []
for m in markets:
    if m.status == 'active':
        markets_active.append(m)
```

#### Generator for large datasets

```python
# ✅ 良い例: 大きなデータセットでは generator でメモリ使用量を抑える
from collections.abc import Iterator

def read_large_file(filepath: str) -> Iterator[str]:
    """大きなファイルを一行ずつ読み取る"""
    with open(filepath, encoding='utf-8') as f:
        for line in f:
            yield line.strip()

# 利用例
for line in read_large_file('huge_file.txt'):
    process(line)  # 一行ずつ処理する

# ❌ 悪い例: ファイル全体を読み込む
with open('huge_file.txt', encoding='utf-8') as f:
    lines = f.readlines()  # すべての行をメモリへ読み込む
    for line in lines:
        process(line)
```

## Python-specific checklist

共通の完了条件は親 skill の [Checklist](../SKILL.md#checklist) とプロジェクトの DoD に具体的に定義されています。ここでは Python 固有の確認項目だけを示します。

- [ ] プロジェクトで定義された mypy の型検査 task が成功している
- [ ] 公開関数・クラスの Python docstring が Google Style と PEP 257 に従い、呼び出し側に必要な情報を含んでいる
- [ ] 本番コードに `print()` を残さず、logging または observability を使っている

## Python task-runner examples

Python の format、lint、型検査、test は、プロジェクトで定義された task runner の task を実行します。`black`、`ruff`、`mypy`、`pytest` などの underlying tool を直接呼ばず、親 skill の formatter・task runner 方針に従います。たとえば uv と taskipy が定義されているプロジェクトでは、次の task を実行します。

```bash
uv run task format
uv run task lint
uv run task type-check
uv run task test
```

task runner がない Python プロジェクトでは、作業指示または既存 CI に定義された検査を使います。存在しない task 名、設定ファイル、formatter を推測して追加しません。開発中の単一テスト実行はプロジェクトの規約が許す場合だけ行い、完了判定ではプロジェクト全体の task を実行します。

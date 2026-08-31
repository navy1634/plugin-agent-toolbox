---
name: coding-standards
description: Python 3.12+ 開発向けのコーディング標準、ベストプラクティス、およびパターン。PEP 8、ruff、mypy 準拠。
---

# Python Coding Standards & Best Practices

Python 3.12+ でのコーディング標準。

## Code Quality Principles

### 1. Readability First（可読性優先）

- Code is read more than written
- Clear variable and function names
- Self-documenting code preferred over comments
- Consistent formatting (black, ruff)

### 2. KISS (Keep It Simple, Stupid)

- Simplest solution that works
- Avoid over-engineering
- No premature optimization
- Easy to understand > clever code

### 3. DRY (Don't Repeat Yourself)

- Extract common logic into functions
- Create reusable classes/utilities
- Share utilities across modules
- Avoid copy-paste programming

### 4. YAGNI (You Aren't Gonna Need It)

- Don't build features before they're needed
- Avoid speculative generality
- Add complexity only when required
- Start simple, refactor when needed

## Python Naming Standards

### Variable Naming

```python
# ✅ GOOD: Descriptive names (snake_case)
market_search_query = 'election'
is_user_authenticated = True
total_revenue = 1000
user_data = fetch_user_data()

# ❌ BAD: Unclear names
q = 'election'
flag = True
x = 1000
d = get_data()
```

### Function Naming

```python
# ✅ GOOD: Verb-noun pattern, snake_case
async def fetch_market_data(market_id: str) -> Market:
    pass

def calculate_similarity(vector_a: list[float], vector_b: list[float]) -> float:
    pass

def is_valid_email(email: str) -> bool:
    pass

# ❌ BAD: Unclear or noun-only
async def market(id):
    pass

def similarity(a, b):
    pass

def email(e):
    pass
```

### Constants

```python
# ✅ GOOD: UPPER_CASE for constants
MAX_RETRIES = 3
API_TIMEOUT_SECONDS = 30
DEFAULT_PAGE_SIZE = 20

# ❌ BAD: Not constant-like
max_retries = 3
api_timeout = 30
```

## Immutability Pattern (CRITICAL)

```python
# ✅ ALWAYS create new objects, NEVER mutate
user_updated = {
    **user,
    'name': 'New Name'
}

items_updated = [*items, new_item]

# Dict unpacking for multiple updates
config_updated = {
    **config,
    'debug': True,
    'timeout': 60
}

# ❌ NEVER mutate directly
user['name'] = 'New Name'  # BAD - mutation!
items.append(new_item)     # BAD - mutation!
```

## Type Hints (REQUIRED)

```python
# ✅ GOOD: Complete type hints
def get_market(market_id: str) -> Market | None:
    """Fetch market by ID."""
    pass

async def create_user(email: str, name: str) -> User:
    """Create new user."""
    pass

def filter_markets(markets: list[Market], status: str) -> list[Market]:
    """Filter markets by status."""
    pass

# With pydantic
from pydantic import BaseModel, EmailStr

class UserCreate(BaseModel):
    email: EmailStr
    name: str
    age: int

# ❌ BAD: No type hints (mypy will fail)
def get_market(id):
    pass

def create_user(email, name):
    pass
```

## Error Handling

```python
# ✅ GOOD: Comprehensive error handling
import logging

def fetch_market_data(market_id: str) -> Market:
    """Fetch market data from API."""
    try:
        response = httpx.get(f'/api/markets/{market_id}')
        response.raise_for_status()
        return Market(**response.json())

    except httpx.HTTPStatusError as e:
        logging.error(f'API error: {e.response.status_code}')
        raise ValueError(f'Failed to fetch market {market_id}') from e

    except Exception as e:
        logging.error(f'Unexpected error: {e}')
        raise

# ❌ BAD: No error handling
def fetch_market_data(market_id):
    response = httpx.get(f'/api/markets/{market_id}')
    return Market(**response.json())
```

## Async/Await Best Practices

```python
# ✅ GOOD: Parallel execution when possible
import asyncio

async def fetch_all_data() -> tuple[list[User], list[Market], list[Stat]]:
    """Fetch data in parallel."""
    users, markets, stats = await asyncio.gather(
        fetch_users(),
        fetch_markets(),
        fetch_stats()
    )
    return users, markets, stats

# ❌ BAD: Sequential when unnecessary
async def fetch_all_data():
    users = await fetch_users()      # Wait for this
    markets = await fetch_markets()  # Then this
    stats = await fetch_stats()      # Then this
    return users, markets, stats
```

## Validation (Pydantic)

```python
# ✅ GOOD: Schema validation with pydantic
from pydantic import BaseModel, EmailStr, field_validator

class CreateMarketRequest(BaseModel):
    name: str
    description: str
    end_date: str
    tags: list[str]

    @field_validator('name')
    @classmethod
    def name_not_empty(cls, v: str) -> str:
        if not v or not v.strip():
            raise ValueError('Name cannot be empty')
        return v.strip()

    @field_validator('description')
    @classmethod
    def description_length(cls, v: str) -> str:
        if len(v) < 10:
            raise ValueError('Description must be at least 10 chars')
        return v

# Usage in FastAPI
@app.post('/markets')
async def create_market(request: CreateMarketRequest):
    """Create market with validated input."""
    # request is already validated
    pass

# ❌ BAD: Manual validation
def create_market(data: dict):
    if not data.get('name'):
        raise ValueError('name required')
    if len(data.get('description', '')) < 10:
        raise ValueError('description too short')
    # ...
```

## File Organization

### Project Structure

```
src/
├── __init__.py
├── main.py                      # FastAPI app entry point
├── app/
│   ├── __init__.py
│   ├── api/                     # API routes
│   │   ├── __init__.py
│   │   ├── market_router.py     # Market endpoints
│   │   └── serializer/          # Request/response models
│   │       ├── __init__.py
│   │       └── market_serializer.py
│   ├── usecase/                 # Business logic
│   │   ├── __init__.py
│   │   └── market_use_case.py
│   ├── domain/                  # Domain logic
│   │   ├── __init__.py
│   │   ├── entity/              # Entity definitions
│   │   │   ├── __init__.py
│   │   │   └── market.py
│   │   ├── service/             # Domain services
│   │   │   ├── __init__.py
│   │   │   └── market_service.py
│   │   └── interface/           # Interfaces/ABCs
│   │       ├── __init__.py
│   │       ├── repository.py
│   │       └── provider.py
│   └── infrastructure/          # DB, external APIs
│       ├── __init__.py
│       ├── models/              # SQLAlchemy models
│       │   ├── __init__.py
│       │   └── market.py
│       ├── repository/          # DB operations
│       │   ├── __init__.py
│       │   └── market_repository.py
│       └── provider/            # External APIs
│           ├── __init__.py
│           └── openai_provider.py
└── tests/
    ├── __init__.py
    ├── unit/
    ├── integration/
    └── e2e/
```

### File Naming

```
app/api/market_router.py          # snake_case for modules
app/domain/entity/market.py       # Entity definition
app/domain/service/market_service.py
app/infrastructure/repository/market_repository.py
tests/unit/test_market_service.py  # test_*.py or *_test.py
```

## Comments & Documentation

### When to Comment

```python
# ✅ GOOD: Explain WHY, not WHAT
# 障害中に API へ負荷を集中させないため指数バックオフを使う
delay = min(1000 * (2 ** retry_count), 30000)

# 大きなリストでの性能を優先し、あえてミュータブルな操作にしている
items.extend(new_items)

# ❌ BAD: Stating the obvious
# カウンタを1増やす
counter += 1

# name にユーザー名を代入する
name = user.name
```

### Docstrings (Google Style)

```python
# ✅ GOOD: Docstring for all public functions
def calculate_similarity(
    vector_a: list[float],
    vector_b: list[float]
) -> float:
    """Calculate cosine similarity between two vectors.

    Args:
        vector_a: First vector of floats.
        vector_b: Second vector of floats.

    Returns:
        Similarity score between 0 and 1.

    Raises:
        ValueError: If vectors have different lengths or are empty.

    Example:
        >>> similarity = calculate_similarity([1, 0, 0], [0, 1, 0])
        >>> similarity
        0.0
    """
    pass

# ✅ GOOD: Class docstring
class Market:
    """Represents a prediction market.

    Attributes:
        id: Unique market identifier.
        name: Market name.
        status: Current market status (active, resolved, closed).
    """

    id: str
    name: str
    status: str
```

## Database Queries

```python
# ✅ GOOD: Select only needed columns (ORM)
from sqlalchemy import select

statement = (
    select(Market.id, Market.name, Market.status)
    .where(Market.status == 'active')
    .limit(10)
)
result = session.execute(statement)

# ✅ GOOD: Use parameterized queries (no injection risk)
statement = select(Market).where(Market.id == market_id)
market = session.execute(statement).scalar_one_or_none()

# ❌ BAD: Select all columns
statement = select(Market)

# ❌ BAD: String concatenation (SQL injection!)
query = f"SELECT * FROM markets WHERE id = '{market_id}'"
```

## Testing Standards

### Test Structure (AAA Pattern)

```python
import pytest

def test_calculate_similarity_identical_vectors():
    """Test similarity of identical vectors."""
    # Arrange
    vector_a = [1, 0, 0]
    vector_b = [1, 0, 0]

    # Act
    similarity = calculate_similarity(vector_a, vector_b)

    # Assert
    assert similarity == 1.0

def test_calculate_similarity_orthogonal_vectors():
    """Test similarity of orthogonal vectors."""
    # Arrange
    vector_a = [1, 0, 0]
    vector_b = [0, 1, 0]

    # Act
    similarity = calculate_similarity(vector_a, vector_b)

    # Assert
    assert similarity == 0.0

def test_calculate_similarity_invalid_vectors():
    """Test similarity with invalid input."""
    # Act & Assert
    with pytest.raises(ValueError):
        calculate_similarity([1, 0], [1, 0, 0])
```

### Test Naming

```python
# ✅ GOOD: Descriptive test names
def test_returns_empty_list_when_no_markets_match_query():
    pass

def test_raises_error_when_openai_api_key_missing():
    pass

def test_falls_back_to_substring_search_when_redis_unavailable():
    pass

# ❌ BAD: Vague test names
def test_works():
    pass

def test_search():
    pass
```

## Code Smell Detection

### 1. Long Functions

```python
# ❌ BAD: Function > 50 lines
def process_market_data(data):
    # 100 lines of code
    pass

# ✅ GOOD: Split into smaller functions
def process_market_data(raw_data: dict) -> Market:
    """Process raw market data."""
    validated = validate_market_data(raw_data)
    transformed = transform_market_data(validated)
    return save_market(transformed)

def validate_market_data(data: dict) -> dict:
    """Validate market data."""
    pass

def transform_market_data(data: dict) -> Market:
    """Transform to Market entity."""
    pass
```

### 2. Deep Nesting

```python
# ❌ BAD: 5+ levels of nesting
def check_permission(user, market):
    if user:
        if user.is_admin:
            if market:
                if market.is_active:
                    if has_permission(user, market):
                        # Do something
                        pass

# ✅ GOOD: Early returns
def check_permission(user: User, market: Market) -> bool:
    """Check if user has permission."""
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

### 3. Magic Numbers

```python
# ❌ BAD: Unexplained numbers
if retry_count > 3:
    raise Exception('Max retries exceeded')

time.sleep(0.5)

# ✅ GOOD: Named constants
MAX_RETRIES = 3
DEBOUNCE_DELAY_MS = 500

if retry_count > MAX_RETRIES:
    raise MaxRetriesExceeded()

time.sleep(DEBOUNCE_DELAY_MS / 1000)
```

## Performance Best Practices

### Lazy Import for Heavy Modules

```python
# ✅ GOOD: Lazy import in function if used rarely
def analyze_with_ml(data: list) -> dict:
    """Analyze data with ML model (heavy)."""
    import numpy as np  # Only import when needed
    return np.mean(data)

# ✅ GOOD: Import at top if used frequently
import numpy as np  # Frequent use

def process_arrays(arrays: list[np.ndarray]) -> np.ndarray:
    """Process numpy arrays."""
    return np.concatenate(arrays)
```

### List Comprehensions Over Loops

```python
# ✅ GOOD: List comprehension
markets_active = [m for m in markets if m.status == 'active']
market_names = [m.name for m in markets]
tuples = [(m.id, m.name) for m in markets]

# ❌ BAD: Manual loop
markets_active = []
for m in markets:
    if m.status == 'active':
        markets_active.append(m)
```

### Generator for Large Datasets

```python
# ✅ GOOD: Generator for memory efficiency
def read_large_file(filepath: str):
    """Yield lines from large file."""
    with open(filepath) as f:
        for line in f:
            yield line.strip()

# Usage
for line in read_large_file('huge_file.txt'):
    process(line)  # Processes one line at a time

# ❌ BAD: Load entire file
with open('huge_file.txt') as f:
    lines = f.readlines()  # Loads all into memory
    for line in lines:
        process(line)
```

## Checklist Before Marking Code Complete

- [ ] Code is readable and well-named
- [ ] Functions are small (< 50 lines)
- [ ] Files are focused (< 400 lines typical)
- [ ] No deep nesting (> 4 levels)
- [ ] Proper error handling with try/except
- [ ] No hardcoded values (use constants)
- [ ] No mutation (immutable patterns used)
- [ ] Type hints on all functions
- [ ] Pass mypy type checking
- [ ] Docstrings for public functions
- [ ] No `print()` (use logging)
- [ ] No commented-out code

## Python Syntax Constraints

- NO `typing` module（明示的なユーザー承認がある場合のみ例外）
- NO functions defined inside functions
- NO imports inside functions
- ALL imports must be at file top
- NO full-width brackets or symbols (RUF003 compliance)

## Documentation

- Docstring は Google Style 必須（Python の場合は PEP 257 準拠）
- モジュールレベルの docstring は記載しない（ファイル冒頭に `"""..."""` を置かない）
- docstring、コメント、ドキュメントはすべて日本語で記述する
- 英語で書くのは識別子などコード本体だけで、docstring とコメントはその対象に含めない

### Docstring に書くこと

呼び出し側がその処理を使うために必要な情報だけを書く。概要と要約であり、スコープ内のロジックの説明書ではない。

- 引数や返り値の型・意味・使い方は `Args:` / `Returns:` / `Raises:` の責務。何が入ってくるかを概要にも重ねて書くと冗長になるだけ
- 「〜としているのは〜のため」のようなロジックの理由は書かない。コメントに書く
- 文の途中で不用意に改行しない。一行を長くしすぎない。どちらも読みにくくなるため例外は認めない

### コメントに書くこと

その処理を理解する補助として、処理が何なのかと、非自明なロジックをなぜそうしたのかを簡潔に書く。

- 名前・型・直下のコードを見れば分かることは書かない。なぜそうしたのかも、コードから読み取れない場合のみ書く
- 以前どうだったか、却下した代替案など、今どうなっているか以外の情報はいらない
- 呼んでいる関数の先のロジックは書かない。その関数を見れば分かる

```python
def create_user(name: str, age: int) -> dict[str, str | int]:
    """ユーザーを作成する。

    Args:
        name: ユーザー名。
        age: ユーザーの年齢。

    Returns:
        作成されたユーザー情報を含む辞書。

    Raises:
        ValueError: age が 0 未満の場合。
    """
```

## Design Patterns

- Always follow existing implementation patterns
- Organize by feature/domain, not by type (layer)
- One concern per file
- High cohesion, low coupling
- Related code should be colocated

## Tools & Commands

```bash
# Format code
black src/

# Lint and fix
ruff check --fix src/

# Type checking
mypy src/

# Run tests
pytest tests/ -v

# Coverage
pytest tests/ --cov=src/

# All checks
black src/ && ruff check --fix src/ && mypy src/ && pytest tests/ --cov=src/
```

---
name: tdd-workflow
description: Python 向けテスト駆動開発。新機能、バグ修正、リファクタリング時に使用。pytest ユニット/統合/E2E テストで 80%+ カバレッジを確保。
---

# Test-Driven Development Workflow

This skill ensures all code development follows TDD principles with comprehensive test coverage.

## When to Activate

- Writing new features or functionality
- Fixing bugs or issues
- Refactoring existing code
- Adding API endpoints
- Creating new components

## Core Principles

### 1. Tests BEFORE Code

ALWAYS write tests first, then implement code to make tests pass.

### 2. Coverage Requirements

- Minimum 80% coverage (unit + integration + E2E)
- All edge cases covered
- Error scenarios tested
- Boundary conditions verified

### 3. Test Types

#### Unit Tests

- Individual functions and utilities
- Service logic
- Pure functions
- Helpers and utilities

#### Integration Tests

- API endpoints
- Database operations
- Service interactions
- External API calls

#### E2E Tests (Playwright)

- Critical user flows
- Complete workflows
- Browser automation
- UI interactions

## TDD Workflow Steps

### Step 1: Write User Journeys

```
As a [role], I want to [action], so that [benefit]

Example:
As a user, I want to search for markets semantically,
so that I can find relevant markets even without exact keywords.
```

### Step 2: Generate Test Cases

For each user journey, create comprehensive test cases:

```python
import pytest

class TestSemanticSearch:
    def test_returns_relevant_markets_for_query(self):
        # Test implementation
        pass

    def test_handles_empty_query_gracefully(self):
        # Test edge case
        pass

    def test_falls_back_to_substring_search_when_redis_unavailable(self):
        # Test fallback behavior
        pass

    def test_sorts_results_by_similarity_score(self):
        # Test sorting logic
        pass
```

### Step 3: Run Tests (They Should Fail)

```bash
uv run pytest
# Tests should fail - we haven't implemented yet
```

### Step 4: Implement Code

Write minimal code to make tests pass:

```python
# Implementation guided by tests
async def search_markets(query: str) -> list[dict]:
    # Implementation here
    pass
```

### Step 5: Run Tests Again

```bash
uv run pytest
# Tests should now pass
```

### Step 6: Refactor

Improve code quality while keeping tests green:

- Remove duplication
- Improve naming
- Optimize performance
- Enhance readability

### Step 7: Verify Coverage

```bash
uv run pytest --cov
# Verify 80%+ coverage achieved
```

## Testing Patterns

### Unit Test Pattern (pytest)

```python
import pytest
from app.domain.entities.button import Button

class TestButtonComponent:
    def test_renders_with_correct_text(self):
        button = Button(text="Click me")
        assert button.text == "Click me"

    def test_calls_on_click_when_clicked(self, mocker):
        handle_click = mocker.Mock()
        button = Button(text="Click", on_click=handle_click)
        button.click()
        assert handle_click.call_count == 1

    def test_is_disabled_when_disabled_true(self):
        button = Button(text="Click", disabled=True)
        assert button.is_disabled is True

    def test_enabled_by_default(self):
        button = Button(text="Click")
        assert button.is_disabled is False
```

### API Integration Test Pattern (FastAPI + pytest)

```python
import pytest
from httpx import AsyncClient
from app.main import app

@pytest.mark.asyncio
class TestMarketsAPI:
    async def test_returns_markets_successfully(self):
        async with AsyncClient(app=app, base_url="http://test") as client:
            response = await client.get("/api/markets")
            
            assert response.status_code == 200
            data = response.json()
            assert data["success"] is True
            assert isinstance(data["data"], list)

    async def test_validates_query_parameters(self):
        async with AsyncClient(app=app, base_url="http://test") as client:
            response = await client.get("/api/markets?limit=invalid")
            assert response.status_code == 400

    async def test_handles_database_errors_gracefully(self, mocker):
        # Mock database failure
        mocker.patch(
            "app.infrastructure.repositories.market_repository.find_all",
            side_effect=Exception("Database connection failed")
        )
        
        async with AsyncClient(app=app, base_url="http://test") as client:
            response = await client.get("/api/markets")
            assert response.status_code == 500
            data = response.json()
            assert data["success"] is False
```

### E2E Test Pattern (Playwright)

```python
import pytest
from playwright.async_api import async_playwright, Page

@pytest.mark.asyncio
class TestMarketSearchE2E:
    async def test_user_can_search_and_filter_markets(self):
        async with async_playwright() as p:
            browser = await p.chromium.launch()
            page = await browser.new_page()
            
            # Navigate to markets page
            await page.goto("http://localhost:3000/")
            await page.click("a[href='/markets']")
            
            # Verify page loaded
            h1 = page.locator("h1")
            await h1.wait_for()
            assert await h1.text_content() == "Markets"
            
            # Search for markets
            await page.fill("input[placeholder='Search markets']", "election")
            
            # Wait for debounce
            await page.wait_for_timeout(600)
            
            # Verify search results displayed
            results = page.locator("[data-testid='market-card']")
            count = await results.count()
            assert count == 5
            
            # Verify results contain search term
            first_result = results.first
            text = await first_result.text_content()
            assert "election" in text.lower()
            
            # Filter by status
            await page.click("button:has-text('Active')")
            
            # Verify filtered results
            count = await results.count()
            assert count == 3
            
            await browser.close()

    async def test_user_can_create_new_market(self):
        async with async_playwright() as p:
            browser = await p.chromium.launch()
            page = await browser.new_page()
            
            # Navigate to creator dashboard
            await page.goto("http://localhost:3000/creator-dashboard")
            
            # Fill market creation form
            await page.fill("input[name='name']", "Test Market")
            await page.fill("textarea[name='description']", "Test description")
            await page.fill("input[name='endDate']", "2025-12-31")
            
            # Submit form
            await page.click("button[type='submit']")
            
            # Verify success message
            success_msg = page.locator("text=Market created successfully")
            await success_msg.wait_for()
            assert await success_msg.is_visible()
            
            # Verify redirect to market page
            await page.wait_for_url("/markets/test-market*")
            
            await browser.close()
```

## Test File Organization

```
src/
├── app/
│   ├── domain/
│   │   ├── entities/
│   │   │   ├── button.py
│   │   │   └── market.py
│   │   ├── services/
│   │   │   └── search_service.py
│   │   └── interfaces/
│   │       └── repository.py
│   ├── usecase/
│   │   └── market_use_case.py
│   ├── api/
│   │   └── markets_router.py
│   └── infrastructure/
│       ├── repositories/
│       │   └── market_repository.py
│       └── providers/
│           └── external_provider.py
└── tests/
    ├── unit/
    │   ├── test_button.py              # Unit tests
    │   ├── test_search_service.py
    │   └── test_market_entity.py
    ├── integration/
    │   ├── test_markets_api.py         # Integration tests
    │   └── test_market_repository.py
    └── e2e/
        ├── test_markets_search.py       # E2E tests
        ├── test_market_creation.py
        └── test_auth.py
```

## Mocking External Services

### Database Mock (SQLAlchemy)

```python
import pytest
from unittest.mock import AsyncMock
from app.infrastructure.repositories.market_repository import MarketRepository

@pytest.fixture
def mock_market_repo(mocker):
    repo = AsyncMock(spec=MarketRepository)
    repo.find_by_id = AsyncMock(return_value={
        "id": 1,
        "name": "Test Market",
        "description": "Test description"
    })
    repo.find_all = AsyncMock(return_value=[
        {"id": 1, "name": "Test Market 1"},
        {"id": 2, "name": "Test Market 2"}
    ])
    return repo

def test_with_mocked_repository(mock_market_repo):
    # Use mocked repository in tests
    result = mock_market_repo.find_by_id(1)
    assert result["name"] == "Test Market"
```

### Redis Mock

```python
import pytest
from unittest.mock import AsyncMock

@pytest.fixture
def mock_redis(mocker):
    redis = AsyncMock()
    redis.search_markets_by_vector = AsyncMock(return_value=[
        {"slug": "test-market", "similarity_score": 0.95},
        {"slug": "another-market", "similarity_score": 0.87}
    ])
    redis.check_health = AsyncMock(return_value={"connected": True})
    return redis

@pytest.mark.asyncio
async def test_with_redis_mock(mock_redis):
    results = await mock_redis.search_markets_by_vector("query")
    assert len(results) == 2
    assert results[0]["similarity_score"] == 0.95
```

### External API Mock (OpenAI)

```python
import pytest
from unittest.mock import AsyncMock

@pytest.fixture
def mock_openai(mocker):
    client = AsyncMock()
    # Mock 1536-dim embedding
    embedding = [0.1] * 1536
    client.embeddings.create = AsyncMock(return_value={
        "data": [{"embedding": embedding}]
    })
    return client

@pytest.mark.asyncio
async def test_with_openai_mock(mock_openai):
    result = await mock_openai.embeddings.create("test query")
    embedding = result["data"][0]["embedding"]
    assert len(embedding) == 1536
    assert embedding[0] == 0.1
```

## Test Coverage Verification

### Run Coverage Report

```bash
uv run pytest --cov
```

### Generate HTML Coverage Report

```bash
uv run pytest --cov --cov-report=html
# Open htmlcov/index.html in browser
```

### Coverage Thresholds (pytest.ini or pyproject.toml)

```ini
[tool:pytest]
addopts = --cov=app --cov-report=term-missing --cov-fail-under=80
```

Or in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
addopts = "--cov=app --cov-report=term-missing --cov-fail-under=80"
```

## Common Testing Mistakes to Avoid

### ❌ WRONG: Testing Implementation Details

```python
# Don't test internal state
def test_counter_state():
    counter = Counter()
    counter.increment()
    assert counter._count == 1  # Implementation detail
```

### ✅ CORRECT: Test User-Visible Behavior

```python
# Test what users see
def test_counter_increments():
    counter = Counter()
    result = counter.increment()
    assert result == 1
```

### ❌ WRONG: Brittle Selectors

```python
# Breaks easily
await page.click(".css-class-xyz")
```

### ✅ CORRECT: Semantic Selectors

```python
# Resilient to changes
await page.click("button:has-text('Submit')")
await page.click("[data-testid='submit-button']")
```

### ❌ WRONG: No Test Isolation

```python
# Tests depend on each other
@pytest.fixture(scope="module")
def shared_user():
    return create_user("test@example.com")

def test_creates_user(shared_user):
    # Uses shared fixture
    pass

def test_updates_same_user(shared_user):
    # Depends on previous test
    pass
```

### ✅ CORRECT: Independent Tests

```python
# Each test creates its own data
def test_creates_user():
    user = create_test_user("user1@example.com")
    assert user.email == "user1@example.com"

def test_updates_user():
    user = create_test_user("user2@example.com")
    user.update_email("newemail@example.com")
    assert user.email == "newemail@example.com"
```

## Continuous Testing

### Watch Mode During Development

```bash
uv run pytest --watch
# Tests run automatically on file changes
```

Or using pytest-watch:

```bash
pip install pytest-watch
ptw
```

### Pre-Commit Hook

```bash
uv run pytest && uv run lint
```

### CI/CD Integration

```yaml
# GitHub Actions
- name: Run Tests
  run: uv run pytest --cov
- name: Upload Coverage
  uses: codecov/codecov-action@v3
```

## Best Practices

1. **Write Tests First** - Always TDD
2. **One Assert Per Test** - Focus on single behavior
3. **Descriptive Test Names** - Explain what's tested
4. **Arrange-Act-Assert** - Clear test structure
5. **Mock External Dependencies** - Isolate unit tests
6. **Test Edge Cases** - None, empty, boundary, large values
7. **Test Error Paths** - Not just happy paths
8. **Keep Tests Fast** - Unit tests < 50ms each
9. **Clean Up After Tests** - Use fixtures for automatic cleanup
10. **Review Coverage Reports** - Identify gaps

## Prohibited

- unittest
- unittest.mock
- pytest.skip（明示的なユーザー承認と理由記載がある場合のみ例外）

## Execution Rules

DoD 検証では必ずタスクランナー経由で実行。

```bash
# 正しい: タスクランナー経由
uv run task test
uv run task lint
uv run task format
uv run task type-check

# 開発中の個別テスト: 許可
uv run pytest tests/test_user.py::test_create -v

# 禁止: ツールの直接実行
uv run ruff check .      # NG
uv run mypy .            # NG
```

## Coverage Requirements

- Minimum 80% coverage required
- High-risk code requires 100% coverage:
  - Authentication/Authorization
  - Financial operations
  - Data validation
  - Security-critical paths

## Success Metrics

- 80%+ code coverage achieved
- All tests passing (green)
- No skipped or disabled tests
- Fast test execution (< 30s for unit tests)
- E2E tests cover critical user flows
- Tests catch bugs before production

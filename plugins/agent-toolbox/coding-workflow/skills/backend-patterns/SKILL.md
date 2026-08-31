---
name: backend-patterns
description: FastAPI + Python 向けのバックエンド アーキテクチャパターン、API 設計、データベース最適化、およびサーバーサイド ベストプラクティス。クリーンアーキテクチャ 4層設計。
---

# Backend Development Patterns

Backend architecture patterns and best practices for scalable server-side applications.

## API Design Patterns

### RESTful API Structure (FastAPI)

```python
# ✅ Resource-based URLs
GET    /api/markets                 # List resources
GET    /api/markets/{id}            # Get single resource
POST   /api/markets                 # Create resource
PUT    /api/markets/{id}            # Replace resource
PATCH  /api/markets/{id}            # Update resource
DELETE /api/markets/{id}            # Delete resource

# ✅ Query parameters for filtering, sorting, pagination
GET /api/markets?status=active&sort=-volume&limit=20&skip=0
```

### Repository Pattern (SQLAlchemy)

```python
# ✅ GOOD: Abstract data access logic
from abc import ABC, abstractmethod
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

class MarketRepository(ABC):
    """Market repository interface."""

    @abstractmethod
    async def find_all(self, status: str | None = None, limit: int = 10) -> list['Market']:
        pass

    @abstractmethod
    async def find_by_id(self, market_id: str) -> 'Market' | None:
        pass

    @abstractmethod
    async def create(self, data: 'CreateMarketRequest') -> 'Market':
        pass

    @abstractmethod
    async def update(self, market_id: str, data: dict) -> 'Market':
        pass

    @abstractmethod
    async def delete(self, market_id: str) -> None:
        pass


# SQLAlchemy implementation
class MarketRepositoryImpl(MarketRepository):
    """SQLAlchemy-based market repository."""

    def __init__(self, session: AsyncSession):
        self.session = session

    async def find_all(self, status: str | None = None, limit: int = 10) -> list[Market]:
        """Fetch all markets with optional filtering."""
        stmt = select(Market).limit(limit)

        if status:
            stmt = stmt.where(Market.status == status)

        result = await self.session.execute(stmt)
        return result.scalars().all()

    async def find_by_id(self, market_id: str) -> Market | None:
        """Fetch market by ID."""
        stmt = select(Market).where(Market.id == market_id)
        result = await self.session.execute(stmt)
        return result.scalars().first()
```

### Service Layer Pattern

```python
# ✅ GOOD: Business logic separated from data access
import logging

class MarketService:
    """Market business logic service."""

    def __init__(self, repo: MarketRepository):
        self.repo = repo
        self.logger = logging.getLogger(__name__)

    async def search_markets(self, query: str, limit: int = 10) -> list[Market]:
        """Search markets using semantic similarity."""
        try:
            # Business logic: generate embedding
            embedding = await self._generate_embedding(query)

            # Vector search
            results = await self._vector_search(embedding, limit)

            # Fetch full data
            market_ids = [r.id for r in results]
            markets = await self.repo.find_by_ids(market_ids)

            # Sort by similarity
            score_map = {r.id: r.score for r in results}
            return sorted(
                markets,
                key=lambda m: score_map.get(m.id, 0),
                reverse=True
            )
        except Exception as e:
            self.logger.error(f'Search failed: {e}')
            raise

    async def _generate_embedding(self, query: str) -> list[float]:
        """Generate text embedding using Claude API."""
        # Implementation
        pass

    async def _vector_search(self, embedding: list[float], limit: int) -> list:
        """Vector search implementation."""
        # Implementation
        pass
```

### Dependency Injection (FastAPI)

```python
# ✅ GOOD: FastAPI dependency injection
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

router = APIRouter(prefix="/markets", tags=["markets"])

async def get_session() -> AsyncSession:
    """Provide database session."""
    async with async_session_maker() as session:
        yield session

async def get_market_repo(session: AsyncSession = Depends(get_session)) -> MarketRepository:
    """Provide market repository."""
    return MarketRepositoryImpl(session)

async def get_market_service(repo: MarketRepository = Depends(get_market_repo)) -> MarketService:
    """Provide market service."""
    return MarketService(repo)

@router.get("/")
async def list_markets(
    service: MarketService = Depends(get_market_service),
    skip: int = 0,
    limit: int = 20
):
    """List all markets."""
    markets = await service.repo.find_all(limit=limit)
    return {"success": True, "data": markets}
```

## Database Patterns

### Query Optimization

```python
# ✅ GOOD: Select only needed columns
from sqlalchemy import select

stmt = (
    select(Market.id, Market.name, Market.status, Market.volume)
    .where(Market.status == 'active')
    .order_by(Market.volume.desc())
    .limit(10)
)
markets = await session.execute(stmt)

# ❌ BAD: Select everything
stmt = select(Market)  # Selects all columns
```

### N+1 Query Prevention

```python
# ❌ BAD: N+1 query problem (N database hits)
async def get_markets_with_creators_bad(session: AsyncSession):
    """Inefficient: N+1 queries."""
    stmt = select(Market)
    result = await session.execute(stmt)
    markets = result.scalars().all()

    # N additional queries!
    for market in markets:
        creator = await session.get(User, market.creator_id)
        market.creator = creator

    return markets

# ✅ GOOD: Batch fetch with single query
async def get_markets_with_creators_good(session: AsyncSession):
    """Efficient: 1-2 queries."""
    # Get all markets
    stmt = select(Market)
    result = await session.execute(stmt)
    markets = result.scalars().all()

    # Get all creators in single query
    creator_ids = [m.creator_id for m in markets]
    stmt = select(User).where(User.id.in_(creator_ids))
    result = await session.execute(stmt)
    creators = result.scalars().all()

    # Map creators to markets
    creator_map = {c.id: c for c in creators}
    for market in markets:
        market.creator = creator_map.get(market.creator_id)

    return markets
```

### Transaction Pattern

```python
# ✅ GOOD: Explicit transaction management
from sqlalchemy.ext.asyncio import AsyncSession

async def create_market_with_position(
    session: AsyncSession,
    market_data: dict,
    position_data: dict
):
    """Create market and position atomically."""
    try:
        # Begin transaction
        async with session.begin_nested():
            # Create market
            market = Market(**market_data)
            session.add(market)
            await session.flush()

            # Create position
            position_data['market_id'] = market.id
            position = Position(**position_data)
            session.add(position)
            await session.flush()

        # Commit transaction
        await session.commit()
        return market, position

    except Exception as e:
        await session.rollback()
        logger.error(f'Transaction failed: {e}')
        raise
```

## Caching Strategies

### Redis Caching Layer

```python
# ✅ GOOD: Cache-aside pattern with Redis
import json
from redis.asyncio import Redis

class CachedMarketRepository:
    """Repository with Redis caching."""

    def __init__(self, base_repo: MarketRepository, redis: Redis):
        self.base_repo = base_repo
        self.redis = redis
        self.cache_ttl = 300  # 5 minutes

    async def find_by_id(self, market_id: str) -> Market | None:
        """Find market with caching."""
        cache_key = f"market:{market_id}"

        # Try cache first
        cached = await self.redis.get(cache_key)
        if cached:
            return Market(**json.loads(cached))

        # Cache miss - fetch from DB
        market = await self.base_repo.find_by_id(market_id)

        if market:
            # Cache for TTL
            await self.redis.setex(
                cache_key,
                self.cache_ttl,
                json.dumps(market.dict())
            )

        return market

    async def invalidate_cache(self, market_id: str) -> None:
        """Invalidate cache for market."""
        await self.redis.delete(f"market:{market_id}")
```

## Error Handling Patterns

### Centralized Error Handler

```python
# ✅ GOOD: Custom exception hierarchy
import logging
from fastapi import FastAPI, Request, status
from fastapi.responses import JSONResponse

class ApiError(Exception):
    """Base API error."""

    def __init__(self, status_code: int, message: str, details: dict | None = None):
        self.status_code = status_code
        self.message = message
        self.details = details or {}

class ValidationError(ApiError):
    """Validation error (400)."""
    def __init__(self, message: str, details: dict = None):
        super().__init__(400, message, details)

class NotFoundError(ApiError):
    """Resource not found error (404)."""
    def __init__(self, message: str):
        super().__init__(404, message)

class UnauthorizedError(ApiError):
    """Unauthorized error (401)."""
    def __init__(self, message: str = "Unauthorized"):
        super().__init__(401, message)

app = FastAPI()

@app.exception_handler(ApiError)
async def api_error_handler(request: Request, exc: ApiError):
    """Handle API errors."""
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "success": False,
            "error": exc.message,
            "details": exc.details
        }
    )

@app.exception_handler(Exception)
async def generic_error_handler(request: Request, exc: Exception):
    """Handle unexpected errors."""
    logging.error(f"Unexpected error: {exc}")
    return JSONResponse(
        status_code=500,
        content={
            "success": False,
            "error": "Internal server error"
        }
    )

# Usage in endpoint
@app.get("/markets/{market_id}")
async def get_market(market_id: str):
    market = await db.find_market(market_id)
    if not market:
        raise NotFoundError(f"Market {market_id} not found")
    return {"success": True, "data": market}
```

### Retry with Exponential Backoff

```python
# ✅ GOOD: Retry logic with backoff
import asyncio
from collections.abc import Callable

async def retry_with_backoff(
    func: Callable[..., None],
    max_retries: int = 3,
    base_delay: float = 1.0
) -> None:
    """Retry function with exponential backoff."""
    last_error = None

    for attempt in range(max_retries):
        try:
            return await func()
        except Exception as e:
            last_error = e

            if attempt < max_retries - 1:
                # Exponential backoff: 1s, 2s, 4s
                delay = base_delay * (2 ** attempt)
                await asyncio.sleep(delay)

    raise last_error

# Usage
async def fetch_from_api():
    """Fetch with retries."""
    return await retry_with_backoff(
        lambda: httpx.get("https://api.example.com/data"),
        max_retries=3
    )
```

## Authentication & Authorization

### JWT Token Validation

```python
# ✅ GOOD: JWT token handling
import jwt
from datetime import datetime, timedelta
from fastapi import HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthCredentials

security = HTTPBearer()
SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"

def create_token(user_id: str, expires_delta: timedelta | None = None) -> str:
    """Create JWT token."""
    if expires_delta is None:
        expires_delta = timedelta(hours=24)

    expire = datetime.utcnow() + expires_delta
    to_encode = {"user_id": user_id, "exp": expire}

    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def verify_token(credentials: HTTPAuthCredentials = Security(security)) -> dict[str, str]:
    """Verify JWT token from request."""
    try:
        payload = jwt.decode(
            credentials.credentials,
            SECRET_KEY,
            algorithms=[ALGORITHM]
        )
        user_id = payload.get("user_id")
        if user_id is None:
            raise UnauthorizedError("Invalid token")
        return {"user_id": user_id}
    except jwt.ExpiredSignatureError:
        raise UnauthorizedError("Token expired")
    except jwt.InvalidTokenError:
        raise UnauthorizedError("Invalid token")

# Usage in protected endpoint
@app.get("/protected")
async def protected_route(user: dict = Depends(verify_token)):
    """Protected endpoint requiring authentication."""
    return {"success": True, "user_id": user["user_id"]}
```

### Role-Based Access Control

```python
# ✅ GOOD: RBAC implementation
from enum import Enum

class Role(str, Enum):
    ADMIN = "admin"
    MODERATOR = "moderator"
    USER = "user"

class Permission(str, Enum):
    READ = "read"
    WRITE = "write"
    DELETE = "delete"
    ADMIN = "admin"

# Role to permissions mapping
ROLE_PERMISSIONS: dict[Role, set[Permission]] = {
    Role.ADMIN: {Permission.READ, Permission.WRITE, Permission.DELETE, Permission.ADMIN},
    Role.MODERATOR: {Permission.READ, Permission.WRITE, Permission.DELETE},
    Role.USER: {Permission.READ, Permission.WRITE},
}

def has_permission(user_role: Role, permission: Permission) -> bool:
    """Check if user has permission."""
    return permission in ROLE_PERMISSIONS[user_role]

async def require_permission(required_permission: Permission):
    """Dependency for checking permissions."""
    async def check_permission(user: dict = Depends(verify_token)):
        user_role = Role(user.get("role", Role.USER))
        if not has_permission(user_role, required_permission):
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return user

    return check_permission

# Usage
@app.delete("/markets/{market_id}")
async def delete_market(
    market_id: str,
    user: dict = Depends(require_permission(Permission.DELETE))
):
    """Delete market (requires DELETE permission)."""
    await db.delete_market(market_id)
    return {"success": True}
```

## Logging & Monitoring

### Structured Logging

```python
# ✅ GOOD: Structured logging with JSON
import logging
import json
from datetime import datetime

class JsonFormatter(logging.Formatter):
    """Format logs as JSON."""

    def format(self, record: logging.LogRecord) -> str:
        log_data = {
            "timestamp": datetime.utcnow().isoformat(),
            "level": record.levelname,
            "message": record.getMessage(),
            "logger": record.name,
        }

        # Add extra fields if present
        if hasattr(record, "user_id"):
            log_data["user_id"] = record.user_id
        if hasattr(record, "request_id"):
            log_data["request_id"] = record.request_id

        if record.exc_info:
            log_data["exception"] = self.formatException(record.exc_info)

        return json.dumps(log_data)

# Setup logger
logger = logging.getLogger(__name__)
handler = logging.StreamHandler()
handler.setFormatter(JsonFormatter())
logger.addHandler(handler)

# Usage
logger.info("Market created", extra={"user_id": "123", "market_id": "456"})
logger.error("Failed to process order", exc_info=True, extra={"order_id": "789"})
```

## Clean Architecture (4層)

API層 → Usecase層 → Domain層 ← Infrastructure層

| 層 | 責務 | 主要コンポーネント |
|----|------|-------------------|
| API | HTTP処理、リクエスト/レスポンス変換 | Router, Serializer |
| Usecase | ビジネスロジック調整、フロー管理 | UseCase, Injector |
| Domain | ビジネスロジック、契約定義 | Entity, Interface, Service, Serializer |
| Infrastructure | 技術実装（外部API、DB） | Provider, Repository |

### 依存関係

- **許可:** API→Usecase, Usecase→Domain+Infrastructure, Infrastructure→Domain Interface
- **禁止:** Domain→API, Domain→Infrastructure（直接）, Infrastructure→Usecase

### Provider vs Repository

| 項目 | Provider | Repository |
|------|----------|------------|
| 対象 | 外部 API | データベース |
| セッション | 不要 | 必要（injector から受領） |
| Injector切替 | `ENVIRONMENT` + モックフラグ | `ENVIRONMENT` + `session is None` |

### Injector パターン

```python
def product_repository_injector(session: AsyncSession | None) -> ProductRepositoryInterface:
    if ENVIRONMENT == "test" or session is None:
        return ProductRepositoryMock()
    return ProductRepositoryImpl(session)
```

### エラーハンドリング（層別）

| 層 | エラークラス | 例 |
|----|------------|-----|
| Domain | `DomainError` | `DomainError(code="INVALID_STATE", message="...")` |
| Usecase | `ApplicationError` | `ApplicationError(kind=ResourceKind.X, message="...")` |
| API | `HTTPException` | `HTTPException(status_code=400, detail="...")` |

### 新機能実装順序

1. **Domain層** → Entity, Interface, Business Logic
2. **Usecase層** → ハンドラー, Service連携
3. **Infrastructure層** → Provider/Repository実装
4. **API層** → Router, Serializer

### テスト戦略（層別）

| 層 | テスト種別 | モック対象 |
|----|----------|-----------|
| Domain | ユニットテスト | Repository/Provider |
| Usecase | 統合テスト | Repository/Provider |
| Infrastructure | 統合テスト | なし（実DB/API） |
| API | E2Eテスト | なし |

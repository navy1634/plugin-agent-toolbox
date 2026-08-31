---
name: project-guidelines-example
description: プロジェクト固有スキルの例。FastAPI + Python バックエンド向けテンプレート。独自プロジェクト用にカスタマイズしてください。
---

# Project Guidelines Skill (Example)

This is an example of a project-specific skill for Python + FastAPI backend. Use this as a template for your own projects.

Based on a real production application architecture.

---

## When to Use

Reference this skill when working on the specific project it's designed for. Project skills contain:

- Architecture overview
- File structure
- Code patterns
- Testing requirements
- Deployment workflow

---

## Architecture Overview

**Tech Stack:**

- **Backend**: FastAPI (Python 3.12+), Pydantic models, SQLAlchemy
- **Database**: PostgreSQL via AWS RDS
- **AI**: Claude API with tool calling and structured output
- **Cache**: Redis via AWS ElastiCache
- **Async**: asyncio + httpx
- **Testing**: pytest + pytest-asyncio
- **Deployment**: AWS ECS on Fargate
- **Container Registry**: AWS ECR
- **Package Manager**: Poetry

**Services Architecture:**

```
┌────────────────────────────────────────────────────┐
│              FastAPI Application                   │
│  Python 3.12+ + Pydantic + SQLAlchemy              │
│  Deployed: AWS ECS Fargate                         │
└────────────────────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
    ┌─────────┐  ┌─────────┐  ┌─────────┐
    │ RDS     │  │ Claude  │  │ElastiCache
    │PostgreSQL  API     │  Redis   │
    └─────────┘  └─────────┘  └─────────┘
```

---

## File Structure

```
project/
├── pyproject.toml                  # Poetry config
├── poetry.lock                     # Locked dependencies
├── .env.example                    # Environment template
├── .env.local                      # Local development (gitignored)
├── src/
│   ├── __init__.py
│   ├── main.py                     # FastAPI app entry
│   ├── config.py                   # Settings/configuration
│   ├── app/
│   │   ├── __init__.py
│   │   ├── api/                    # API routes
│   │   │   ├── __init__.py
│   │   │   ├── market_router.py    # Market endpoints
│   │   │   └── serializer/         # Request/response models
│   │   │       ├── __init__.py
│   │   │       └── market_serializer.py
│   │   ├── usecase/                # Business logic
│   │   │   ├── __init__.py
│   │   │   └── market_use_case.py
│   │   ├── domain/                 # Domain logic
│   │   │   ├── __init__.py
│   │   │   ├── entity/             # Entity definitions
│   │   │   ├── service/            # Domain services
│   │   │   └── interface/          # ABC/protocols
│   │   └── infrastructure/         # DB, external APIs
│   │       ├── __init__.py
│   │       ├── models/             # SQLAlchemy models
│   │       ├── repository/         # DB operations
│   │       └── provider/           # External API clients
│   └── common/
│       ├── __init__.py
│       ├── exception.py            # Custom exceptions
│       └── logger.py               # Logging setup
├── tests/
│   ├── __init__.py
│   ├── conftest.py                 # pytest fixtures
│   ├── unit/                       # Unit tests
│   ├── integration/                # Integration tests
│   └── e2e/                        # End-to-end tests
└── docs/
    ├── README.md
    └── API.md
```

---

## Code Patterns

### API Response Format (FastAPI + Pydantic)

```python
from pydantic import BaseModel

class ApiResponse[T](BaseModel):
    """Standard API response wrapper."""
    success: bool
    data: T | None = None
    error: str | None = None
    meta: dict | None = None

    @classmethod
    def ok(cls, data: T, meta: dict | None = None) -> "ApiResponse[T]":
        """Create success response."""
        return cls(success=True, data=data, meta=meta)

    @classmethod
    def fail(cls, error: str) -> "ApiResponse[T]":
        """Create error response."""
        return cls(success=False, error=error)
```

### FastAPI Endpoint Pattern

```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession
from app.usecase.market_use_case import MarketUseCase
from app.api.serializer.market_serializer import (
    CreateMarketRequest,
    MarketResponse
)

router = APIRouter(prefix="/markets", tags=["markets"])

@router.post("/", response_model=MarketResponse, status_code=status.HTTP_201_CREATED)
async def create_market(
    request: CreateMarketRequest,
    session: AsyncSession = Depends(get_session)
) -> MarketResponse:
    """Create a new market."""
    use_case = MarketUseCase(session)
    market = await use_case.create_market(request)
    return MarketResponse.from_entity(market)

@router.get("/{market_id}", response_model=MarketResponse)
async def get_market(
    market_id: str,
    session: AsyncSession = Depends(get_session)
) -> MarketResponse:
    """Get market details."""
    use_case = MarketUseCase(session)
    market = await use_case.get_market(market_id)

    if not market:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Market {market_id} not found"
        )

    return MarketResponse.from_entity(market)
```

### Claude AI Integration (Structured Output)

```python
from anthropic import Anthropic
from pydantic import BaseModel

class AnalysisResult(BaseModel):
    """Structured analysis output."""
    summary: str
    key_points: list[str]
    confidence: float

async def analyze_with_claude(content: str) -> AnalysisResult:
    """Analyze content using Claude with structured output."""
    client = Anthropic()

    response = client.messages.create(
        model="claude-opus-4-5-20250514",
        max_tokens=1024,
        messages=[{"role": "user", "content": content}],
        tools=[{
            "name": "provide_analysis",
            "description": "Provide structured analysis",
            "input_schema": AnalysisResult.model_json_schema()
        }],
        tool_choice={"type": "tool", "name": "provide_analysis"}
    )

    # Extract tool use result
    tool_use = next(
        block for block in response.content
        if block.type == "tool_use"
    )

    return AnalysisResult(**tool_use.input)
```

### Repository Pattern (SQLAlchemy)

```python
from abc import ABC, abstractmethod
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

class MarketRepository(ABC):
    """Market data access interface."""

    @abstractmethod
    async def find_by_id(self, market_id: str) -> 'Market' | None:
        pass

    @abstractmethod
    async def find_all(self, limit: int = 10) -> list['Market']:
        pass

    @abstractmethod
    async def create(self, data: CreateMarketRequest) -> 'Market':
        pass

class MarketRepositoryImpl(MarketRepository):
    """SQLAlchemy implementation."""

    def __init__(self, session: AsyncSession):
        self.session = session

    async def find_by_id(self, market_id: str) -> Market | None:
        """Fetch market by ID."""
        stmt = select(Market).where(Market.id == market_id)
        result = await self.session.execute(stmt)
        return result.scalars().first()

    async def find_all(self, limit: int = 10) -> list[Market]:
        """Fetch all markets."""
        stmt = select(Market).limit(limit)
        result = await self.session.execute(stmt)
        return result.scalars().all()

    async def create(self, data: CreateMarketRequest) -> Market:
        """Create new market."""
        market = Market(
            name=data.name,
            description=data.description,
            end_date=data.end_date
        )
        self.session.add(market)
        await self.session.flush()
        return market
```

---

## Testing Requirements

### Backend (pytest)

```bash
# Run all tests
poetry run pytest tests/

# Run with coverage
poetry run pytest tests/ --cov=src --cov-report=html

# Run specific test file
poetry run pytest tests/unit/test_market_service.py -v

# Watch mode
poetry run pytest tests/ --watch
```

**Test structure (pytest + asyncio):**

```python
import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from src.main import app

@pytest.fixture
async def db_session():
    """Provide async database session for tests."""
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    async with AsyncSession(engine) as session:
        yield session

@pytest.fixture
async def client():
    """Provide async test client."""
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac

@pytest.mark.asyncio
async def test_create_market(client: AsyncClient):
    """Test market creation."""
    response = await client.post(
        "/markets/",
        json={
            "name": "Test Market",
            "description": "A test market",
            "end_date": "2025-12-31"
        }
    )

    assert response.status_code == 201
    data = response.json()
    assert data["success"] is True
    assert data["data"]["name"] == "Test Market"

@pytest.mark.asyncio
async def test_get_market_not_found(client: AsyncClient):
    """Test getting non-existent market."""
    response = await client.get("/markets/nonexistent")
    assert response.status_code == 404
```

---

## Deployment Workflow

### Pre-Deployment Checklist

- [ ] All tests passing: `poetry run pytest tests/ --cov=src`
- [ ] Type checking passing: `poetry run mypy src/`
- [ ] Linting passing: `poetry run ruff check src/`
- [ ] Code formatting: `poetry run black src/`
- [ ] No hardcoded secrets
- [ ] Environment variables documented in `.env.example`
- [ ] Database migrations up to date
- [ ] Dependencies updated: `poetry update`

### Deployment Commands

```bash
# Set AWS variables
export AWS_ACCOUNT_ID=123456789012
export AWS_REGION=us-east-1
export ECR_REPOSITORY=project-backend
export SERVICE_NAME=project-backend
export CLUSTER_NAME=project-cluster

# Build container image
docker build -t project-backend:latest .

# Tag image for ECR
docker tag project-backend:latest \
  $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY:latest

# Login to ECR
aws ecr get-login-password --region $AWS_REGION | \
  docker login --username AWS --password-stdin \
  $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

# Push to ECR
docker push \
  $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY:latest

# Deploy to ECS Fargate
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service $SERVICE_NAME \
  --region $AWS_REGION \
  --force-new-deployment
```

### Dockerfile (Fargate Optimized)

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install dependencies
RUN pip install --no-cache-dir poetry

# Copy poetry files
COPY pyproject.toml poetry.lock ./

# Install dependencies
RUN poetry install --no-dev --no-interaction --no-ansi

# Copy source code
COPY src/ src/

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

# Run FastAPI with Uvicorn
CMD ["poetry", "run", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### ECS Task Definition (JSON)

```json
{
  "family": "project-backend",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "project-backend",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/project-backend:latest",
      "portMappings": [
        {
          "containerPort": 8000,
          "hostPort": 8000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "ENVIRONMENT",
          "value": "production"
        },
        {
          "name": "LOG_LEVEL",
          "value": "INFO"
        }
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:project/db-url"
        },
        {
          "name": "ANTHROPIC_API_KEY",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:project/anthropic-key"
        },
        {
          "name": "REDIS_URL",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:project/redis-url"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/project-backend",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 10
      }
    }
  ],
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskRole"
}
```

### Environment Variables

```bash
# .env for local development
DATABASE_URL=postgresql://user:password@localhost/db_name
ANTHROPIC_API_KEY=sk-ant-...
REDIS_URL=redis://localhost:6379
LOG_LEVEL=INFO
ENVIRONMENT=development

# Production (stored in AWS Secrets Manager)
# Retrieve with:
# aws secretsmanager get-secret-value --secret-id project/db-url --region us-east-1
# aws secretsmanager get-secret-value --secret-id project/anthropic-key --region us-east-1
# aws secretsmanager get-secret-value --secret-id project/redis-url --region us-east-1

DATABASE_URL=postgresql://prod-user:***@project-db.c9akciq32.us-east-1.rds.amazonaws.com:5432/production
ANTHROPIC_API_KEY=***
REDIS_URL=redis://project-redis.abc123.ng.0001.use1.cache.amazonaws.com:6379
ENVIRONMENT=production
LOG_LEVEL=INFO
```

### AWS Infrastructure Setup

```bash
# Create ECR repository
aws ecr create-repository \
  --repository-name project-backend \
  --region us-east-1

# Create ECS cluster
aws ecs create-cluster \
  --cluster-name project-cluster \
  --region us-east-1

# Create CloudWatch log group
aws logs create-log-group \
  --log-group-name /ecs/project-backend \
  --region us-east-1

# Register task definition
aws ecs register-task-definition \
  --cli-input-json file://task-definition.json \
  --region us-east-1

# Create ECS service
aws ecs create-service \
  --cluster project-cluster \
  --service-name project-backend \
  --task-definition project-backend:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx,subnet-yyy],securityGroups=[sg-xxx],assignPublicIp=ENABLED}" \
  --load-balancers targetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/project-backend/xxx,containerName=project-backend,containerPort=8000 \
  --region us-east-1
```

---

## Critical Rules

1. **No emojis** in code, comments, or documentation
2. **Immutability** - never mutate objects or collections
3. **Type hints** - all functions require type hints
4. **TDD** - write tests before implementation
5. **80% coverage** minimum for all code
6. **Small files** - 200-400 lines typical, 800 max
7. **No logging statements** - use proper logging module
8. **Input validation** - use Pydantic for all inputs
9. **Error handling** - comprehensive try/catch with logging
10. **No hardcoded values** - use config/environment

---

## Quality Gates

```bash
# Run all checks before commit
poetry run black src/ tests/
poetry run ruff check --fix src/ tests/
poetry run mypy src/
poetry run pytest tests/ --cov=src --cov-report=term-missing

# All must pass!
```

---

## Related Skills

- `coding-standards.md` - Python 3.12+ coding standards
- `backend-patterns.md` - FastAPI + clean architecture patterns
- `tdd-workflow.md` - pytest test-driven development
- `security-review.md` - Security checklist for APIs

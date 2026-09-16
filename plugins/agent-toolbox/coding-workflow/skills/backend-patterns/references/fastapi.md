# FastAPI と Pydantic

この reference は、FastAPI または Pydantic を使って presentation 層の API adapter を実装・変更するときに読みます。FastAPI と Pydantic 固有の型、依存性注入、exception handler は presentation 層に閉じ込め、domain と application はそれらを import しません。REST、request/response、protocol-neutral な境界契約は [api-frameworks](api-frameworks.md) を正とします。

## Pydantic による境界検証

FastAPI で Pydantic を使う場合は、request body を `BaseModel` に束縛し、field validator で API 境界の形式と入力制約を検証します。検証済みの request を use case へ渡し、response も専用の schema へ明示的に変換します。domain の不変条件や権限判定を Pydantic の validator だけで済ませません。

```python
from fastapi import FastAPI
from pydantic import BaseModel, field_validator

app = FastAPI()


class CreateMarketRequest(BaseModel):
    name: str
    description: str
    end_date: str
    tags: list[str]

    @field_validator("name")
    @classmethod
    def name_not_empty(cls: type[BaseModel], value: str) -> str:
        if not value or not value.strip():
            raise ValueError("名前は空にできません")
        return value.strip()

    @field_validator("description")
    @classmethod
    def description_length(cls: type[BaseModel], value: str) -> str:
        if len(value) < 10:
            raise ValueError("説明は10文字以上で指定してください")
        return value


class MarketResponse(BaseModel):
    id: str
    name: str
    status: str


@app.post("/markets", response_model=MarketResponse)
async def create_market(request: CreateMarketRequest) -> MarketResponse:
    market = await market_use_case.execute(request)
    return MarketResponse(id=market.id, name=market.name, status=market.status)
```

この例では、空白だけの `name` を拒否して前後の空白を除去し、`description` の最小長を API 境界で検証します。response は `MarketResponse` を経由するため、domain entity や ORM model のフィールドをそのまま公開しません。

## FastAPI の依存性注入と構成例

次は FastAPI と SQLAlchemy を使う場合の構成例です。domain と application は FastAPI の `Depends` や SQLAlchemy の session を import しません。

```python
from collections.abc import AsyncIterator

from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

router = APIRouter(prefix="/markets", tags=["markets"])


async def get_session() -> AsyncIterator[AsyncSession]:
    async with async_session_maker() as session:
        yield session


def get_market_repository(session: AsyncSession = Depends(get_session)) -> MarketRepository:
    return SqlAlchemyMarketRepository(session)


def get_market_use_case(repository: MarketRepository = Depends(get_market_repository)) -> ListMarkets:
    return ListMarkets(repository)


@router.get("/")
async def list_markets(use_case: ListMarkets = Depends(get_market_use_case), skip: int = 0, limit: int = 20) -> MarketListResponse:
    markets = await use_case.execute(skip=skip, limit=limit)
    return MarketListResponse.from_domain(markets)
```

この例では、DI 関数が具体 adapter を composition root として組み立て、handler は公開 request と response の変換だけを担当します。test では `get_market_use_case` を同じ use case contract を満たす依存へ差し替え、API の契約を検証します。

非同期 endpoint であっても、呼び出し先が同期処理なら無理に async 化しません。blocking I/O を event loop 上で直接実行しないこと、client と session の lifecycle を request または application の境界に合わせることを確認します。

## FastAPI の error handler

FastAPI では、層の error を exception handler で API response へ集中変換します。domain と application は `HTTPException` や response class を import しません。

```python
import logging

from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

logger = logging.getLogger(__name__)
app = FastAPI()


class ApiError(Exception):
    def __init__(self, status_code: int, code: str, message: str, details: list[dict[str, object]] | None = None) -> None:
        self.status_code = status_code
        self.code = code
        self.message = message
        self.details = details or []


class ValidationError(ApiError):
    def __init__(self, message: str, details: list[dict[str, object]] | None = None) -> None:
        super().__init__(400, "INVALID_REQUEST", message, details)


class NotFoundError(ApiError):
    def __init__(self, resource: str) -> None:
        super().__init__(404, "RESOURCE_NOT_FOUND", f"{resource} が見つかりません。")


class UnauthorizedError(ApiError):
    def __init__(self) -> None:
        super().__init__(401, "UNAUTHENTICATED", "認証が必要です。")


@app.exception_handler(ApiError)
async def handle_api_error(request: Request, exc: ApiError) -> JSONResponse:
    request_id = request.headers.get("x-request-id")
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": {
                "code": exc.code,
                "message": exc.message,
                "details": exc.details,
                "request_id": request_id,
            }
        },
    )


@app.exception_handler(Exception)
async def handle_unexpected_error(request: Request, exc: Exception) -> JSONResponse:
    logger.exception("unexpected backend error")
    return JSONResponse(
        status_code=500,
        content={"error": {"code": "INTERNAL_ERROR", "message": "内部エラーが発生しました。"}},
    )
```

実際の FastAPI application では、request ID の生成・検証、exception handler の登録、validation error の統合、response の schema を既存 middleware と合わせます。未認証と権限不足、resource 不在とアクセス拒否などの区別は、公開契約と security-review の要件に従います。

## FastAPI 固有の確認項目

- request body が `BaseModel` へ束縛され、必須・任意 field、型、形式、サイズ、validator の拒否条件が公開契約と一致しています。
- Pydantic の validator に domain の不変条件や認可判断を隠しておらず、検証済み request を use case へ渡しています。
- response_model または専用 response schema へ明示的に変換し、domain entity や ORM model の内部 field を直接公開していません。
- `Depends` による session、repository、use case の組み立てが composition root に集約され、handler が SQL、ORM query、外部 SDK、内部 repository を直接呼び出していません。
- test では依存性を use case contract を満たす依存へ差し替え、公開 API の schema、status、error を検証しています。
- `ApiError` と予期しない exception の handler が登録され、status、machine-readable code、details、request ID が公開契約へ対応しています。
- FastAPI の validation error、serialization、未知 field、default status の挙動を確認し、独自の error response と矛盾させていません。
- 非同期 endpoint 上で blocking I/O を実行せず、client と session の lifecycle、timeout、切断時の扱いを確認しています。

API 全体の正常系、空結果、pagination、認証・認可、再送・重複の検証は [api-frameworks](api-frameworks.md) の検証観点と併せて確認します。

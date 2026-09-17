# 永続化、ORM、cache、性能

この reference は database、ORM、cache、transaction、query、性能を変更するときに読みます。具体的な製品や library の選択は既存の project 方針に合わせ、repository/provider の port と domain model の境界は [architecture](architecture.md) の契約を守ります。

## 永続化の基本契約

- repository の port は domain または application の内側に置き、データストアを表す ORM model、session、query builder、driver の型を内側へ漏らしません。
- adapter は永続化 model と domain object を明示的に変換します。database の null、日時、decimal、enum、削除状態などの意味を domain 側の契約へ写像します。
- query は必要な列と件数に絞り、暗黙の全件取得、N+1、不要な join、未検証の sort/filter、offset の過度な深掘りを避けます。
- index は実際の検索条件、sort、cardinality、更新負荷、保持期間を確認して追加します。index を増やす場合は読み取り改善と書き込み・容量コストを記録します。
- 複数の書き込みが一つの業務操作として原子的である必要がある場合、transaction 境界を use case 単位で明示します。commit、rollback、isolation、lock、競合時の応答を定義します。
- retry や再送で同じ書き込みが複数回実行されても、重複や部分適用が起きない idempotency を設計します。unique constraint、idempotency key、upsert などの採用理由を残します。

## Repository pattern の具体例

次の例は SQLAlchemy を使う場合の形です。interface は application 側に置き、実装だけが SQLAlchemy と session を知ります。

```python
from abc import ABC, abstractmethod

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession


class MarketRepository(ABC):
    @abstractmethod
    async def find_all(self, status: str | None = None, limit: int = 10) -> list[Market]:
        raise NotImplementedError

    @abstractmethod
    async def find_by_id(self, market_id: str) -> Market | None:
        raise NotImplementedError

    @abstractmethod
    async def create(self, data: CreateMarketRequest) -> Market:
        raise NotImplementedError

    @abstractmethod
    async def update(self, market_id: str, data: dict[str, object]) -> Market:
        raise NotImplementedError

    @abstractmethod
    async def delete(self, market_id: str) -> None:
        raise NotImplementedError


class SqlAlchemyMarketRepository(MarketRepository):
    def __init__(self, session: AsyncSession) -> None:
        self.session = session

    async def find_all(self, status: str | None = None, limit: int = 10) -> list[Market]:
        statement = select(MarketModel).limit(limit)
        if status is not None:
            statement = statement.where(MarketModel.status == status)

        result = await self.session.execute(statement)
        return [MarketMapper.to_domain(model) for model in result.scalars().all()]

    async def find_by_id(self, market_id: str) -> Market | None:
        statement = select(MarketModel).where(MarketModel.id == market_id)
        result = await self.session.execute(statement)
        model = result.scalars().first()
        return None if model is None else MarketMapper.to_domain(model)
```

実際の project の型、同期・非同期方式、mapper の位置は既存規約に合わせます。repository の method は use case が必要とする能力だけを表し、汎用 query builder をそのまま公開しません。

## Query optimization

一覧や検索では、response に必要な列だけを選択します。

```python
statement = (
    select(
        MarketModel.id,
        MarketModel.name,
        MarketModel.status,
        MarketModel.volume,
    )
    .where(MarketModel.status == "active")
    .order_by(MarketModel.volume.desc(), MarketModel.id)
    .limit(10)
)
markets = await session.execute(statement)
```

`select(MarketModel)` のように全列を読む実装は、列数、payload、join、object materialization のコストを確認せずに採用しません。取得したデータをさらに API 用 DTO へ変換し、永続化 model を返さないでください。

### Parameterized query

ORM の expression または bind parameter を使い、入力値を SQL の構造から分離します。検索条件を文字列として連結せず、入力値の型と許可値を repository の契約に従って検証します。

```python
from sqlalchemy import select

statement = select(MarketModel).where(MarketModel.id == market_id)
result = await session.execute(statement)
```

raw SQL が必要な場合も、値の位置に bind parameter を使います。

```python
from sqlalchemy import text

statement = text("SELECT id, name FROM markets WHERE id = :market_id")
result = await session.execute(statement, {"market_id": market_id})
```

入力値を SQL 文字列へ埋め込む実装は採用しません。

```python
query = f"SELECT * FROM markets WHERE id = '{market_id}'"
```

### N+1 の回避

N+1 は一覧を取得した後、各行の関連を一件ずつ読む形で発生します。次の bad 例のようにループ内で `session.get` を実行せず、join、eager load、または ID の一括取得でクエリ数を固定します。

```python
# 避ける例
markets = (await session.execute(select(MarketModel))).scalars().all()
for market in markets:
    market.creator = await session.get(UserModel, market.creator_id)
```

```python
# ID の一括取得で関連をまとめて読む例
markets = (await session.execute(select(MarketModel))).scalars().all()
creator_ids = {market.creator_id for market in markets}

creators = []
if creator_ids:
    creators = (
        await session.execute(
            select(UserModel).where(UserModel.id.in_(creator_ids))
        )
    ).scalars().all()

creator_map = {creator.id: creator for creator in creators}
for market in markets:
    market.creator = creator_map.get(market.creator_id)
```

空の ID 集合を query に渡した場合の library の挙動も確認します。関連が多い場合は、取得件数、join による重複、memory 使用量、ページングの一貫性を測定したうえで eager load と一括取得を選びます。

## Transaction pattern

複数の書き込みが一つの業務操作に属するなら、use case が一つの transaction 境界として扱います。session の開始・commit・rollback を adapter の暗黙動作にせず、project の unit-of-work 方針に合わせて一箇所で管理します。

```python
from sqlalchemy.ext.asyncio import AsyncSession


async def create_market_with_position(session: AsyncSession, market_data: dict[str, object], position_data: dict[str, object]) -> tuple[MarketModel, PositionModel]:
    async with session.begin():
        market = MarketModel(**market_data)
        session.add(market)
        await session.flush()

        position = PositionModel(**position_data, market_id=market.id)
        session.add(position)
        await session.flush()

    return market, position
```

`session.begin()` の context manager が commit と rollback を担う方式です。手動管理を選ぶ場合も、例外時の rollback、commit 後の response、部分失敗、再実行時の重複を明示します。nested transaction や savepoint は、外側の transaction と異なる意味になるため、必要性と isolation を記録せずに使いません。

次の条件を変更に応じて定義します。

- transaction に含める読み書き、commit 前後に実行してよい外部呼び出し。
- isolation level、optimistic/pessimistic lock、version、競合時に返す error。
- timeout、deadlock、serialization failure の retry 可否と上限。
- 外部 API、queue、cache invalidation を commit とどう整合させるか。
- worker の再送や client の retry で同じ command が実行されたときの idempotency。

## Cache-aside と Redis の例

cache は source of truth ではありません。採用する場合は key、schema、TTL、stale data の許容、更新・削除時の invalidation、cache 障害時の fallback、stampede 対策を定義します。

```python
import json

from redis.asyncio import Redis


class CachedMarketRepository:
    def __init__(self, base_repository: MarketRepository, redis: Redis) -> None:
        self.base_repository = base_repository
        self.redis = redis
        self.cache_ttl_seconds = 300

    async def find_by_id(self, market_id: str) -> Market | None:
        cache_key = f"market:{market_id}"
        cached = await self.redis.get(cache_key)
        if cached is not None:
            return Market.from_dict(json.loads(cached))

        market = await self.base_repository.find_by_id(market_id)
        if market is not None:
            await self.redis.setex(
                cache_key,
                self.cache_ttl_seconds,
                json.dumps(market.to_dict()),
            )
        return market

    async def invalidate(self, market_id: str) -> None:
        await self.redis.delete(f"market:{market_id}")
```

cache hit の値を信頼できる期間、serialization version、negative cache の扱い、更新処理と invalidation の順序を決めます。cache の読み取り失敗を database の障害と同じ成功応答へ隠さず、可用性と一貫性の優先順位を記録します。

## 性能の受入条件

性能変更では、変更前後で同じ条件を比較できるように次を残します。

- データ量、分布、同時実行数、cache warm/cold、ネットワーク、hardware の前提。
- query 数、実行時間、rows scanned、response size、memory、lock 待ち、error rate の測定方法。
- index、pagination、cache、batch、connection pool、serialization の変更内容と副作用。
- p50 だけでなく、要件に応じた p95/p99、timeout、負荷時の degradation。
- 性能改善が一貫性、freshness、transaction、運用コストをどう変えるか。

クエリ計画や実測なしに「高速化した」と判定しません。repository の統合テストでは、必要に応じて実際の database または test container を使い、mock だけで query planner、transaction、constraint、lock の挙動を保証しないでください。

## 永続化の検証観点

- repository/provider の port と adapter の境界を確認し、ORM、driver、cache client の型が domain/application に漏れていないことを確認します。
- 必要列・件数、N+1、空集合、index、pagination、transaction、rollback、競合、retry、idempotency の各判断の根拠を確認します。
- cache を使う場合は TTL、key、invalidation、stale data、障害時の挙動を受入条件へ対応づけます。
- 実測または対象技術に即した統合確認の方法を確認し、性能と一貫性の未確認範囲を記録します。

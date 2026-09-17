# レイヤード／クリーンアーキテクチャ

この reference は、特定の言語やフレームワークではなく、層の責務と依存方向を設計するときに読みます。既存プロジェクトが別の命名を採用している場合も、名前ではなく依存の向きと責務で対応づけます。

## 基本形

基本の依存方向は `presentation → application → domain ← infrastructure` です。infrastructure は domain が定義した port を実装し、composition root が concrete adapter を application へ注入します。

```text
┌──────────────────────┐
│ presentation / API   │  protocol、入力、認証情報、response
└──────────┬───────────┘
           │ 呼び出す
┌──────────▼───────────┐
│ application / usecase│  業務フロー、認可、transaction
└──────────┬───────────┘
           │ port を利用
┌──────────▼───────────┐
│ domain               │  業務規則、entity、value object、契約
└──────────▲───────────┘
           │ port を実装
┌──────────┴───────────┐
│ infrastructure       │  DB、cache、外部 API、queue、filesystem
└──────────────────────┘
```

| 層 | 責務 | 代表的な構成要素 | 依存してよい対象 |
| --- | --- | --- | --- |
| presentation / API | HTTP、RPC、message などの protocol 処理、入力の形式検証、主体の確定、response 変換 | router、handler、request/response schema、protocol adapter | application の公開 interface、protocol library |
| application / use case | 業務フローの調整、port の協調、認可の適用、transaction 境界、結果の組み立て | use case、application service、command/query、input/output DTO | domain の型と port、時計や ID 生成などの抽象 |
| domain | 業務規則、不変条件、状態遷移、値の意味、内側の契約 | entity、value object、domain service、repository/provider interface、domain error | 標準ライブラリと domain 自身の型 |
| infrastructure / adapter | 外部技術との接続、永続化・復元、外部形式との変換、client lifecycle | repository/provider 実装、ORM model、SDK client、mapper、configuration adapter | domain の port、対応する技術 library |

presentation は application を呼び出すだけにし、application は具体的な ORM、SDK、HTTP client、database session を参照しません。domain は presentation、application、infrastructure の具体実装を知りません。infrastructure から application を呼び戻す依存も作りません。

composition root、bootstrap、依存性注入設定など、具体 adapter を選択して配線する場所だけは全層へ依存できます。ただし、配線の責務を domain や各 use case へ分散させません。

## repository と provider

repository と provider はどちらも port と adapter の境界で扱いますが、対象と失敗特性が異なります。

| 項目 | repository | provider |
| --- | --- | --- |
| 対象 | database、object store などのデータストア | 外部 API、検索・推論・決済などのサービス、計算資源、queue |
| 内側の契約 | entity や query 結果、保存・更新の意味 | 外部サービスの能力、request/result、外部エラーの意味 |
| adapter の責務 | query、ORM model、transaction、データ mapper | SDK、認証情報、timeout、retry、rate limit、response mapper |
| use case から見えるもの | 技術名ではなく必要な読み書きの port | 技術名ではなく必要な外部能力の port |

repository や provider の interface は内側に置き、実装は infrastructure に置きます。adapter の戻り値として ORM model、SDK の response、transport の例外を内側へ渡しません。session や client の lifecycle が必要な場合は、composition root または application の transaction policy で明示的に管理します。

## module 構成例

既存の規約に合わせることを優先します。次は `src/app/api/usecase/domain/infrastructure` の責務名を使った具体的な構成例です。`api` は presentation、`usecase` は application に対応し、依存方向と責務の説明は言語や framework に依存しません。

```text
src/
├── app/
│   ├── api/
│   │   └── serializer/
│   ├── usecase/
│   ├── domain/
│   │   ├── entity/
│   │   ├── service/
│   │   └── interface/
│   └── infrastructure/
│       ├── models/
│       ├── repository/
│       └── provider/
└── tests/
    ├── unit/
    ├── integration/
    └── e2e/
```

`api/serializer` は request と response の公開形式を変換し、`usecase` は業務フローを調整します。`domain/entity` と `domain/service` は業務規則を持ち、`domain/interface` は repository/provider などの内側の契約を定義します。`infrastructure/models`、`repository`、`provider` は具体的な DB・外部 API・その他の技術 adapter を実装します。entrypoint や言語固有の module 初期化ファイルは、採用言語の規約に従って `src` 側へ配置します。

この構成でも、`api` が `infrastructure` を直接 import したり、異なる機能の private module を横断して呼び出したりしないことが重要です。共有するのは安定した domain または application の公開契約に限定します。

## port と依存性注入の例

use case は、必要な能力だけを port として受け取ります。次の例では、DB 製品や外部検索製品は use case のコードに現れません。

```text
interface MarketRepository
    find_by_id(id) -> Market or None
    find_by_ids(ids) -> list[Market]

interface EmbeddingProvider
    create(text) -> Embedding

interface VectorSearchProvider
    search(embedding, limit) -> list[SearchHit]

class SearchMarkets
    constructor(repository, embedding_provider, vector_search_provider)

    execute(query, limit)
        embedding = embedding_provider.create(query)
        hits = vector_search_provider.search(embedding, limit)
        markets = repository.find_by_ids(ids_of(hits))
        return sort_by_similarity(markets, hits)
```

composition root では、実行環境の値を use case 内で判定して mock に切り替えるのではなく、実行対象に合わせた adapter を明示的に組み立てます。

```text
function build_application(configuration)
    repository = SqlMarketRepository(configuration.database)
    embedding_provider = ExternalEmbeddingProvider(configuration.embedding)
    vector_search_provider = VectorSearchAdapter(configuration.vector_store)
    return SearchMarkets(repository, embedding_provider, vector_search_provider)

function build_test_application(test_dependencies)
    return SearchMarkets(
        test_dependencies.repository,
        test_dependencies.embedding_provider,
        test_dependencies.vector_search_provider,
    )
```

この形にすると、`ENVIRONMENT == "test"` や `session is None` のような暗黙の条件を本番の業務ロジックへ持ち込まず、test double の選択と本番 adapter の選択を同じ port 契約で検証できます。

## 新機能の設計順序

次の順序を基本とします。既存の公開契約や移行制約がある場合は、その理由を記録します。

1. 公開 API または message の正常系、入力、出力、エラー、認証境界、互換性を定義します。
2. domain の entity、value object、不変条件、状態遷移、domain error を定義します。
3. domain または application に必要な repository/provider の port を定義します。
4. application の use case、認可、transaction、retry、冪等性を定義します。
5. infrastructure の repository/provider、mapper、client lifecycle を実装します。
6. presentation の handler、schema、protocol mapping と composition root の配線を実装します。
7. 各層の受入条件をテストし、依存方向と公開契約に対する証拠を確認します。

既存の境界をまたぐ変更では、実装を始める前に公開契約、エラー変換、データ移行、失敗時に残る状態を記録します。

## 層別テスト

テストは実装の内部呼び出しを再現するためではなく、各層の契約と仕様を確認するために分けます。

| 層 | 主な確認対象 | 依存先の扱い |
| --- | --- | --- |
| domain | entity、value object、業務規則、不変条件、状態遷移 | 外部 I/O を持たせず、必要な port は契約で扱う |
| application | use case の正常系、業務上の拒否、port の協調、認可、transaction の結果 | port contract に沿った double を注入する |
| infrastructure | query、mapper、transaction、cache、外部 API の実際の挙動 | 実 DB、test container、emulator、recorded response など対象技術に沿った方法を使う |
| presentation | request/response schema、status、error、認証境界、pagination、再送 | protocol client による contract または E2E test を使う |

domain の unit test に実際の DB や HTTP client を持ち込まず、infrastructure の test で用意した fake の振る舞いだけを信頼して実装しないでください。テストの共通契約は [tdd-workflow](../../tdd-workflow/SKILL.md) を参照します。

## 設計レビューの観点

- 依存グラフをたどり、domain が外側の技術へ依存していないことを確認します。
- presentation に SQL、ORM query、外部 SDK、業務上の分岐がなく、application が公開 protocol の型を返していないことを確認します。
- port が use case に必要な能力だけを表し、adapter の型・例外・設定が内側へ漏れていないことを確認します。
- concrete adapter の選択が composition root に集約され、test 用差し替えが暗黙の環境分岐になっていないことを確認します。
- 新しい公開契約、失敗時の状態、transaction、認可、外部接続、テスト境界を変更内容へ対応づけます。

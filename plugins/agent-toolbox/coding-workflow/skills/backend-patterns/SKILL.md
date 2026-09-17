---
name: backend-patterns
description: 言語やフレームワークを問わず、レイヤード／クリーンアーキテクチャを基に API とバックエンドの境界、依存方向、公開契約を設計・変更するときに使う。
---

# Backend Patterns

この skill は、バックエンドの機能をレイヤードアーキテクチャまたはクリーンアーキテクチャの考え方で分割し、公開 API と内部の業務処理、外部技術との境界を設計・レビューするために使います。言語、フレームワーク、データベース、クラウド、メッセージング製品に依存しない判断をこのファイルに置き、技術固有の実装方法や実例は関係する reference に置きます。

## 適用条件

次のいずれかに該当する変更で使用します。

- 公開 API、Webhook、RPC、メッセージ consumer などの入出力境界を追加・変更する場合。
- 既存の domain、use case、repository、provider、adapter の責務や依存方向を変更する場合。
- データベース、cache、外部 API、queue などの永続化・外部接続を導入・変更する場合。
- 認証済み主体、権限、tenant、所有権を業務処理へ渡す場合。
- エラー契約、retry、transaction、logging、metrics、tracing などのバックエンド横断設計を変更する場合。

単純な言語構文や formatter、linter、型検査の規約は [coding-standards](../coding-standards/SKILL.md) を正とし、テストの進め方と project の DoD は [tdd-workflow](../tdd-workflow/SKILL.md) を併用します。コード例を実装へ移すときは既存 project の formatter を実行し、関数 signature、parameter list、その他の whitespace の改行位置を、docs の作者が横幅に基づいて手動で決めません。認証・認可、外部入力、secret、機密データ、公開面の脅威が変わる場合は [security-review](../../../planning-review-docs/skills/security-review/SKILL.md) を必ず併用します。

## 設計契約

### 境界と依存方向

基本の依存方向は次のとおりです。

```text
presentation/API → application/use case → domain ← infrastructure/adapter
                                      ↑
                           port の実装を注入
```

| 層 | 担当する責務 | 置いてよいもの |
| --- | --- | --- |
| presentation / API | プロトコル処理、入力の形式検証、認証情報からの主体の確定、response と status への変換 | router、handler、request/response schema、protocol adapter |
| application / use case | 業務フローの調整、transaction 境界、認可の適用、port の協調、結果の組み立て | use case、application service、command/query、input/output DTO |
| domain | 業務規則、状態遷移、不変条件、値オブジェクト、内側の port 契約 | entity、value object、domain service、repository/provider interface、domain error |
| infrastructure / adapter | DB、cache、外部 API、queue、filesystem など技術との接続と変換 | repository/provider 実装、ORM model、client、serializer、configuration adapter |

presentation は application に依存し、application は domain の型と port に依存します。infrastructure は domain の port を実装します。domain は外側の層や技術型を import せず、application は concrete な ORM・SDK・transport に依存しません。composition root は concrete adapter を選択して port へ注入する例外的な場所です。

次の依存は許可しません。

- domain から presentation、infrastructure、ORM、HTTP、SDK への依存。
- application から concrete な repository、provider、database session、transport への依存。
- presentation から SQL、ORM query、外部 SDK を直接呼び出すこと。
- infrastructure から application の use case を呼び戻すこと。
- 層をまたぐためだけの循環依存や、依存方向を隠す global singleton。

### 責務の分離

- API adapter は入力を内部型へ変換し、application の結果を公開契約へ変換します。業務判断や永続化 query を持たせません。
- use case は業務上の手順を表し、repository/provider の port だけを呼び出します。transport の status や ORM model を返しません。
- domain は外部 I/O を行わず、業務規則を純粋に評価できる構造にします。外部と協調する必要がある場合は port を定義します。
- repository はデータストアの読み書き、provider は外部 API・計算資源・メッセージングなどの呼び出しを抽象化します。どちらも技術型を内側へ漏らしません。
- mapper は API DTO、domain object、永続化 model の変換を担当し、暗黙の変換や内部 model の直接公開を避けます。
- composition root では constructor injection などの明示的な依存性注入を使い、test 用 adapter の選択を domain の条件分岐へ埋め込みません。

### 公開 API 契約

公開する protocol に応じて形式は変わりますが、少なくとも次を契約として決めます。

- resource と操作の表現、path、method または message type、versioning、互換性を明記します。REST では URL を名詞で表し、action 名の乱立を避けます。
- request の必須・任意、型、形式、サイズ、範囲、許可値、default、null、unknown field の扱いを定義します。入力検証は境界と domain invariant の二段階で行います。
- response の schema、status または message ack、空結果、pagination、sort、filter、timestamp、identifier、cache の意味を定義します。内部 entity や ORM model をそのまま公開しません。
- domain error、入力エラー、認証・認可エラー、競合、依存先障害、予期しないエラーの外部表現と、retry 可否を分けて定義します。内部 stack trace、SQL、secret、個人情報は返しません。
- timeout、rate limit、payload size、pagination 上限、idempotency、重複配信、順序、再送の扱いを state-changing な契約へ含めます。
- 後方互換性を壊す変更では versioning、移行期間、deprecation、既存 consumer への影響を明示します。

### 入出力とエラー境界

- presentation は transport 固有のエラーへ変換し、domain と application のエラーを安定した machine-readable code へ写像します。
- domain error は不変条件や業務上の拒否、application error は use case の前提や競合、infrastructure error は外部依存の失敗として分類します。分類は project の既存契約に合わせます。
- unexpected error は内部へ記録し、利用者には一般化した応答だけを返します。例外を握りつぶしたり、すべてを成功応答に変換したりしません。
- retry は一時的な失敗だけを対象にし、最大回数、timeout、backoff、jitter、idempotency、dead letter や fallback を明示します。
- log、metric、trace は request または operation を追跡できる相関情報を持たせ、secret、token、個人情報、生の入力、SQL を不用意に出力しません。

### 認証と認可の境界

- request adapter で credential を検証し、認証済み主体を application へ明示的に渡します。署名、期限、issuer、audience、失効、鍵更新などの方式は採用する認証基盤に合わせます。
- application または policy 層で、主体・資源・操作の組み合わせを server side で認可します。UI や URL を隠すことだけで認可を済ませません。
- tenant、所有者、resource scope、管理者権限などの条件を対象 resource に近い場所で検証し、権限のない主体の存在や内部情報を不要に漏らしません。
- 未認証と認証済みだが権限がない状態は、既存の公開契約に従って区別します。詳細な脅威分析と安全要件は security-review に委譲します。

### 永続化と外部接続

- repository と provider の port は domain または application の内側に定義し、具体 adapter と model を外側に閉じ込めます。
- 複数の書き込みが一つの業務操作として原子的である必要がある場合、transaction 境界を use case として明示します。commit、rollback、isolation、競合、再実行時の idempotency を定義します。
- query は必要な列と件数に絞り、N+1、暗黙の全件取得、不要な join、未検証の sort/filter を避けます。性能変更には測定方法とデータ量の前提を残します。
- cache は source of truth ではなく、TTL、key、stale data、invalidation、stampede、障害時の fallback を説明できる場合だけ導入します。
- 外部接続は timeout、retry、rate limit、response 検証、失敗時の状態、外部 client の lifecycle を契約に含めます。技術固有の実装は [persistence](references/persistence.md) と [error-and-observability](references/error-and-observability.md) を読みます。

### テスト可能性

- domain の規則は外部 I/O なしで unit test できるようにします。
- application は port の contract double を注入して、正常系、業務上の拒否、依存先エラー、transaction 境界、認可を検証します。
- infrastructure は実際の database、cache、外部 API emulator または test container など、対象技術の挙動を確認できる統合テストで検証します。
- presentation は公開 API の schema、status、error、認証境界、pagination を contract または E2E test で検証します。
- テストは実装の内部呼び出し順や private method ではなく、仕様、公開契約、受入条件に依存させます。詳細なテストの進め方は tdd-workflow に従います。

## 作業の進め方

1. 既存の公開契約、依存方向、データフロー、エラー契約、認証境界、repository/provider を確認し、変更前の境界を記録します。
2. 受入条件から resource、command、query、domain invariant、外部 I/O、失敗時の状態を定義します。
3. API または message の公開契約と adapter の変換を定め、domain の entity、value object、error、port を設計します。
4. application の use case と transaction・認可の適用位置を定め、port を通じて必要な adapter を協調させます。
5. infrastructure adapter、mapper、composition root を実装し、外部技術の型が内側へ漏れないことを確認します。
6. 層ごとのテストと既存 project の lint、型検査、security check を実行し、実行した確認と未確認の範囲を分けて報告します。

設計上の選択肢がある場合は、既存の契約、依存方向、変更範囲、運用性、テスト可能性、性能、可逆性を同じ基準で比較し、採用理由と不採用理由を記録します。結論に合わせて後から理由を作ってはいけません。

## 完了条件

- 変更後の層、依存方向、port、concrete adapter、composition root の責務が説明でき、禁止した依存や循環依存がありません。
- 公開 API または message の正常系、入力エラー、認証・認可エラー、domain error、依存先障害、空結果、pagination、再送・重複の契約が定義されています。
- API DTO、domain object、永続化 model が適切に分離され、内部型、SQL、SDK、secret が公開応答へ漏れていません。
- transaction、cache、retry、timeout、競合、冪等性、失敗時に残る状態が変更の要件に応じて説明できます。
- log、metric、trace による観測方法と機密値のマスキングが定義され、運用上の失敗を追跡できます。
- domain、application、infrastructure、presentation の各境界に対するテストまたは既存チェックが受入条件を満たし、テストが実装詳細に依存していません。
- 変更した契約、未解決事項、残余リスク、未確認の前提が記録されています。

## References

- [Architecture](references/architecture.md) — 層、依存方向、module 構成、provider／repository、依存性注入、層別テストを設計・レビューするときに読む。
- [API and REST contracts](references/api-frameworks.md) — REST の resource、request/response、protocol-neutral な adapter と DI の契約を変更するときに読む。
- [FastAPI](references/fastapi.md) — FastAPI、Pydantic、FastAPI の DI、exception handler、JSON response を実装・変更するときに読む。
- [Persistence](references/persistence.md) — database、ORM、query、transaction、cache、性能を変更するときに読む。
- [Error and observability](references/error-and-observability.md) — framework-neutral な error hierarchy、status mapping、retry、logging、metrics、tracing、monitoring を変更するときに読む。
- [Auth boundaries](references/auth-boundaries.md) — credential、token、session、role、permission、tenant、resource ownership を変更するときに読む。脅威分析は security-review と併用する。

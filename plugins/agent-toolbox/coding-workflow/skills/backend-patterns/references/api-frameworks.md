# API と REST の契約

この reference は、API の公開契約や protocol adapter を追加・変更するときに読みます。REST 以外の RPC、GraphQL、message consumer でも、同じ境界契約を protocol に合わせて適用します。

## REST の resource 契約

resource 指向の API では URL を名詞で表し、操作は HTTP method で表します。

```text
GET    /api/markets                  一覧を取得する
GET    /api/markets/{id}             1 件を取得する
POST   /api/markets                  resource を作成する
PUT    /api/markets/{id}             resource 全体を置き換える
PATCH  /api/markets/{id}             resource の一部を更新する
DELETE /api/markets/{id}             resource を削除する

GET /api/markets?status=active&sort=-volume&limit=20&skip=0
```

action 名の endpoint を増やす前に、既存 resource の状態遷移として表せないかを検討します。action が独立した command として必要な場合は、既存 API の命名規約と idempotency を確認してから採用します。

一覧 API では、絞り込み、並び替え、pagination を query parameter に分け、次を契約へ含めます。

- `limit` の default と最大値、`skip` または cursor の形式と失効条件。
- sort key、昇順・降順、同値時の安定した tie-breaker、許可しない sort key の扱い。
- status filter などの許可値、複数条件の組み合わせ、未知の parameter の扱い。
- 0 件の response schema、total count の有無、次ページの有無、削除済み resource の扱い。
- 一覧取得中の更新や削除がある場合の一貫性、cursor の再利用、retry と重複の扱い。

## 入出力 schema と status

request/response schema は API 境界に定義し、domain entity、ORM model、SDK response を直接公開しません。入力を domain object へ変換した後に業務規則を検証し、domain の結果を response DTO へ明示的に変換します。request payload と response payload は別々の契約として示し、同じ JSON object に混在させません。

request payload の構成例です。

```json
{
  "name": "example",
  "status": "active"
}
```

response payload の構成例です。

```json
{
  "data": {
    "id": "market-123",
    "name": "example",
    "status": "active"
  }
}
```

上記は構成例であり、`data` の wrapper、field 名、identifier、timestamp の形式は既存の公開契約に合わせます。少なくとも次を対象 endpoint ごとに定義します。

- 成功時の status、response schema、nullable field、空結果の形式。
- 入力形式エラー、認証・認可エラー、resource 不在、競合、依存先障害、予期しないエラーの status と machine-readable code。
- response に含めない内部情報、エラー詳細の公開範囲、request identifier の扱い。
- timeout、payload size、rate limit、idempotency key、再送、cache header または message ack の意味。

API adapter は入力の型、形式、サイズ、範囲、許可値を検証します。domain の不変条件は domain 側で再度検証し、API の検証を迂回する別の入口があっても契約が壊れないようにします。

## 依存性注入と adapter

request adapter が application service または use case を生成し、use case が repository/provider の interface だけを受け取る構成にします。endpoint から SQL、ORM query、外部 SDK を直接呼び出さないでください。

```text
request
  ↓ 形式検証と認証情報の変換
API handler
  ↓ input DTO
application use case
  ↓ domain port
repository/provider adapter
  ↓ domain result
response mapper
  ↓ 公開 schema
response
```

非同期 API であっても、呼び出し先が同期処理なら無理に async 化しません。blocking I/O を event loop 上で直接実行しないこと、client と session の lifecycle を request または application の境界に合わせることを確認します。

## API adapter の確認項目

- routing、schema、middleware、dependency injection、exception handler が presentation 層の外へ漏れていません。
- API 実装の default status、validation error、serialization、未知 field の挙動が公開契約に合っています。
- request ごとの timeout、body size、rate limit、CORS、content type、compression の設定根拠が説明できます。
- handler が use case を一度だけ呼び、結果を明示的に公開 schema へ変換しています。use case の内部 repository を handler が直接操作していません。
- protocol の再送、重複、順序、切断、deadline、ack がある場合、application の冪等性と失敗時の状態に対応しています。

## endpoint の検証観点

対象 endpoint または consumer について、正常系、必須・任意入力、型・形式・サイズ境界、空結果、pagination 境界、認証・認可エラー、resource 不在、domain error、依存先エラー、timeout、再送・重複を検証します。既存の API contract test、integration test、E2E test、lint、型検査がある場合は、プロジェクトの手順に従って結果と未確認範囲を記録します。

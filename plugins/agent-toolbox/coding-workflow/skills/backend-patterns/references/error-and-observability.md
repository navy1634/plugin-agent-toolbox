# エラー処理と観測性

この reference は、層をまたぐ error mapping、外部依存の retry、logging、metrics、tracing、monitoring を変更するときに読みます。安全性に関わる secret、個人情報、token、入力値の扱いは [security-review](../../../../planning-review-docs/skills/security-review/SKILL.md) と併用します。

## エラーの分類と境界

domain、application、infrastructure、presentation のエラーを分類し、内部の例外と利用者へ返す status・message を分けます。project に既存の error hierarchy がある場合は、それを壊さずに対応づけます。

| 層 | 失敗の意味 | 外部へ返すときの判断 |
| --- | --- | --- |
| domain | 不変条件違反、状態遷移の拒否、業務上存在しない操作 | machine-readable な業務 error code へ変換する |
| application | use case の前提不足、resource 不在、競合、冪等性違反 | 公開契約の status と error code へ変換する |
| infrastructure | database、cache、外部 API、queue、filesystem の失敗 | retry 可否を分類し、内部詳細を隠した依存先 error へ変換する |
| presentation | request の形式不正、protocol の timeout、認証情報の不備 | protocol 固有の validation、認証、認可 error へ変換する |

利用者向け response には内部 stack trace、SQL、table 名、SDK response、credential、token、個人情報、内部 URL を含めません。unexpected error は内部へ exception と相関 ID を記録し、外部には一般化した error code だけを返します。

## Error response の契約

API では、既存の response 形式に合わせて、少なくとも次を定義します。

```json
{
  "error": {
    "code": "MARKET_NOT_FOUND",
    "message": "指定された resource は見つかりません。",
    "details": [],
    "request_id": "request-123"
  }
}
```

この JSON は構成例であり、field 名や `details` の公開範囲は project の契約を正とします。`message` は利用者向けの安定した説明、`code` は client が分岐できる値とし、stack trace や例外文字列をそのまま使用しません。


## Metrics、tracing、monitoring

operation ごとに、少なくとも次の観測可能性を検討します。

- request count、成功・失敗 count、error code 別の割合、retry 回数、timeout、queue backlog。
- latency の p50、p95、p99、database query time、外部 dependency の latency。
- resource 使用量、connection pool、cache hit/miss、transaction rollback、deadlock。
- request ID と trace ID の伝播、外部 call の span、失敗した層と adapter の特定。
- alert の閾値、window、severity、runbook、通知先、誤検知時の調整方法。

metric label や log field に高 cardinality の生入力、token、secret、個人情報を使いません。監視が発火したときに、利用者影響、失敗箇所、再試行の有無、復旧操作を追跡できる構造にします。

## エラーと観測性の検証観点

- 層ごとの error が公開 status・code へ一貫して変換され、unexpected error の内部詳細が response に漏れないことを確認します。
- retry 対象、最大回数、timeout、backoff、jitter、idempotency、retry exhaustion の挙動を確認します。
- error の再現テストまたは既存の contract/integration test で、想定外エラーを成功応答に変換していないことを確認します。
- log、metric、trace に request または operation を追跡する相関情報があり、secret、token、個人情報、生入力を記録しないことを確認します。
- 監視条件、alert の閾値、未確認の外部依存範囲、残余リスクを記録します。

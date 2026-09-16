---
name: security-review
description: 認証・認可、外部入力、機密情報、決済、第三者連携、または公開境界の脅威面が変わる設計・実装・レビューで使う。
---

# Security Review

この skill は、変更によって増減する脅威面と信頼境界を明らかにし、必要な安全要件を設計・実装・レビューへ反映するために使います。コードだけでなく、設定、IaC、CI、データフロー、外部サービス連携も対象にします。

## 適用条件

次のいずれかに該当する変更では、この skill を使います。明示的なセキュリティレビュー、脅威分析、インシデント対応を求められた場合も対象です。

- 認証、認可、セッション、token、role、tenant、権限を追加または変更する場合。
- API、Web UI、Webhook、ファイル upload、CLI、queue などの外部入力や公開境界を追加または変更する場合。
- secret、credential、秘密鍵、個人情報、決済情報、その他の機密データを扱う場合。
- 決済、残高、所有権、削除、権限変更など、損失や権限逸脱につながる操作を追加または変更する場合。
- 依存関係、第三者 API、外部 network、redirect、SSRF の可能性、データ保存先、暗号化、backup、監査を追加または変更する場合。
- Terraform、AWS、GitHub Actions などで公開範囲、IAM、credential、実行権限、secret、state、artifact の扱いを変更する場合。provider や製品固有の設定は対応する skill も読みます。

## この skill が定めること

### 脅威面と信頼境界

変更前に、保護対象の資産、利用者・管理者・サービスなどの主体、外部入力、外部出力、保存先、外部サービス、権限境界を洗い出します。信頼できない値を client、URL、UI、network 内部、既存 database などの見かけだけで信頼せず、境界ごとに検証と認可を置きます。

データフローでは、どの主体がどの資産に対してどの操作を行うか、どこで権限が変わるか、失敗時に何が残るかを追跡します。対象外にした境界は、対象外とした理由を記録します。

### 横断的な安全契約

- 外部から来る入力は不正値を前提にし、境界で型、形式、サイズ、範囲、許可値を検証してから query、command、HTML、path、外部 API へ渡します。出力先の文脈に応じた encoding または sanitization も行います。
- 保護された操作は server side で主体、資源、操作の組み合わせを認可し、最小権限と default deny を基本にします。UI や URL を隠すだけで認可を済ませません。
- secret、credential、秘密鍵、token、機密データを source、設定、fixture、画像、ログ、error、response、artifact、履歴へ転載しません。保存する機密データは、保存先、保持期間、暗号化、アクセス主体、削除、backup、rotation を決めます。
- 外部依存と外部通信は、許可する source、endpoint、権限、TLS、timeout、retry、response の検証を決め、再現可能な依存解決と監査可能な変更履歴を保ちます。
- state-changing な処理では、再送、replay、二重実行、競合、部分失敗を考慮し、必要に応じて rate limit、timeout、idempotency、監査イベント、復旧手順を設計します。
- 利用者向けの response は内部構造、stack trace、SQL、secret、内部 URL を漏らさず、server log と監査記録も機密値をマスキングします。

## 作業手順

1. 要件、受入条件、既存の実装と設定を読み、資産、主体、trust boundary、data flow、変更による threat surface を記録します。
2. 認証認可、入力と Web、secret と data、依存と response、Solana のうち、変更に関係する reference だけを読みます。該当しない領域も、対象外とした理由を計画またはレビュー記録へ残します。
3. 脅威または misuse case ごとに、影響、発生条件、緩和策、緩和策の対象ファイル・component、失敗時の挙動、検証方法を定義します。既存の要件や repository 方針で決まらない分岐を推測しません。
4. 正常系だけでなく、境界値、不正入力、権限不足、期限切れ、再送、依存障害、秘密情報の誤出力など、観測可能な安全性のテストまたは静的確認を受入条件へ対応づけます。検証スクリプトを新設して標準 tooling の代替にしてはいけません。
5. 実装・レビュー後に、実行した確認と未確認の範囲を分けて記録し、残余リスク、例外承認、risk owner、期限または見直し条件を更新します。

## 計画・受入契約

セキュリティに関係する計画または work order には、少なくとも次の要素を含めます。planner を経由しない作業では、同じ情報を作業指示やレビュー記録に置きます。

- 保護対象の資産と機密度、関係する主体、trust boundary、data flow、および変更後に増える公開面。
- 認証、認可、入力、secret、機密データ、出力、依存、外部通信、監査、monitoring、復旧のうち、該当する安全要件と対象範囲。
- 各安全要件を満たす component、設定、公開契約、失敗時の挙動、担当する書き込み範囲。
- 脅威・misuse case、影響、緩和策、テストまたは静的確認、確認できない前提。
- 未解決のリスク、採用した例外、risk owner、対応期限または再評価条件。

受入条件は「安全そうである」といった印象ではなく、対象となる control とその証拠で判定できる形にします。重大な未解決リスク、未承認の例外、根拠のない安全宣言が残る場合は完了扱いにしません。低減できない残余リスクを受け入れる場合も、既存の承認方針に従い、受入者と期限を明記します。

## 完了条件

- 変更に関係する threat surface と trust boundary を記録し、各対象領域の reference または対象外の理由を確認できます。
- 定義した安全要件、認証認可、入力検証、secret／data lifecycle、外部依存、response、監査などの control に対して、実装と検証の証拠があります。
- source、設定、テスト、ログ、response、artifact、履歴に secret または不要な機密情報を露出していません。
- 重大な未解決リスクや未承認の例外がなく、残余リスクと未確認範囲が記録されています。
- プロジェクトで定義された既存の test、lint、security check、plan、review の結果を、対象全体に対する証拠として報告しています。独自の検証スクリプトを追加して完了条件を満たしたことにしてはいけません。

実際の secret 漏えい、認証回避、重大な権限逸脱を発見した場合は、その経路の継続を止め、client または既存の security incident 手順へ直ちに返します。credential の revoke／rotation、影響範囲の封じ込め、証拠保全、利用者への連絡などの外部操作は、承認された運用手順に従って実施します。

## Reference の選択

| 変更の境界 | 読む reference |
| --- | --- |
| secret、credential、個人情報、決済情報、保存・ログ・backup | [secrets-and-data.md](references/secrets-and-data.md) |
| 外部入力、API、Web、upload、SQL、HTML、CSRF、rate limit、SSRF | [input-and-web.md](references/input-and-web.md) |
| 認証、session、token、role、permission、tenant、RLS | [authentication-and-authorization.md](references/authentication-and-authorization.md) |
| 依存、第三者 API、response、error、監査、monitoring、incident | [dependencies-and-response.md](references/dependencies-and-response.md) |

複数の境界にまたがる場合は、該当する reference をすべて読みます。reference は具体的な脅威、例、検査項目を定めるものであり、横断的な資産・境界・証拠・残余リスクの判断を省略する理由にはなりません。

## 他 skill との分担

Terraform、AWS、GitHub Actions などの provider・製品・tool 固有の権限、設定、state、workflow、実行コマンドは、それぞれの skill を正とします。security-review はそれらの詳細を複製せず、脅威面、trust boundary、最小権限、secret／data lifecycle、検証証拠、残余リスクが計画と受入条件に含まれているかを確認します。テストの構成や project の DoD は、該当する test／coding skill と repository の task runner に従います。

# 依存関係と対応

第三者 package、SDK、base image、外部 API、WebHook、外部 network、response、ログ、監査、monitoring、production deployment を変更する場合に読みます。具体的な package manager、cloud provider、CI の操作は対応する ecosystem skill と repository の task runner を正とし、ここでは依存と運用に共通する安全要件を確認します。

## 依存関係と第三者連携

依存ごとに source、version、lock、integrity、権限、transitive dependency、脆弱性 advisory、更新方針、risk owner を確認します。lock file または同等の固定情報を commit し、CI と production で再現可能な解決方法を使います。自動更新を使う場合も、差分、権限、脆弱性、互換性を review できる状態にします。

未修正の脆弱性は、影響を受ける機能と version、到達可能性、緩和策、期限、risk owner、受入または再評価条件を記録します。脆弱性がないという宣言だけで完了にせず、実行した advisory scan と対象範囲を証拠として残します。

第三者 API や WebHook では、endpoint、TLS、認証情報の scope、送信データ、受信データ、署名検証、timeout、retry、rate limit、redirect、失敗時の副作用を定義します。外部から返る data、status、header、redirect を無条件に信頼せず、schema と上限を検証します。不要な outbound network、権限、個人情報の送信を許可しません。

決済や権限変更などの外部 callback は、署名、timestamp、nonce、idempotency、再送、順序逆転、部分成功を確認し、client の成功画面だけを支払い・状態変更の根拠にしません。

## Response とエラー

利用者向け response は generic なエラー形式にし、exception message、stack trace、SQL、内部 URL、ファイル path、依存サービスの応答、secret を返しません。HTTP status、error code、request ID、retry 可否を既存 API 契約に合わせます。詳細な診断は server log に限定し、秘密情報と個人情報をマスキングします。

production の Web 境界では、HTTPS の強制、CSP、X-Frame-Options または同等の frame 制御、適切な CORS、content-type、cookie、cache、security header を確認します。具体的な header 値は Web framework と repository 方針に合わせ、未使用の `unsafe-eval`、`unsafe-inline`、wildcard origin を無条件に許可しません。

## 監査と monitoring

認証、認可、権限変更、secret の利用、機密 data の read／write／delete、外部 callback、rate limit 超過、security check の失敗を、actor、resource、operation、結果、request ID、時刻とともに監査します。token、password、秘密鍵、カード情報を監査イベントに含めません。

監査ログは改ざん、削除、過剰閲覧を防ぎ、保持期間、アクセス主体、alert、backup、復旧を既存の運用方針へ結び付けます。monitoring が検出すべき異常、通知先、severity、誤検知時の扱いを計画します。

## 自動 security test と公開前確認

自動 security test では、変更に該当する次の境界を対象にします。テストの構成と実行方法は既存の test skill／task runner に従い、独自の検証スクリプトを追加して標準 tooling の代替にしません。

- secret scan、設定・artifact・履歴への secret 混入。
- 入力 schema、upload、path、SQL／command injection、XSS、CSRF、SSRF、CORS、redirect、rate limit。
- 認証、token の期限・失効、認可、tenant／owner、RLS、権限不足時の副作用。
- timeout、retry、外部 API／WebHook の署名、response schema、再送、idempotency、部分失敗。
- error response、ログの masking、監査 event、monitoring、backup と復旧。

security test の期待結果は、既存 API 契約に合わせて具体的な status、response、監査、副作用で確認します。

| シナリオ | 観測する受入結果の例 |
| --- | --- |
| 認証が必要な endpoint を未認証で呼ぶ場合 | 401 を返し、保護された data や内部詳細を返しません |
| 管理者権限が必要な操作を一般 user が呼ぶ場合 | 403 を返し、状態を変更せず、必要な監査 event を記録します |
| schema に反する入力を送る場合 | 400 など既存契約の client error を返し、query や外部副作用を起こしません |
| rate limit を超える場合 | 429 など既存契約の status と retry 情報を返し、制限 event を記録します |

production 公開前は、該当する項目を確認します。

- [ ] secret が source、設定、fixture、ログ、response、artifact、履歴にありません。
- [ ] 外部入力、file upload、SQL／command、XSS、CSRF、SSRF、CORS、redirect が確認されています。
- [ ] 認証、token、認可、role、tenant／owner、RLS の境界が確認されています。
- [ ] rate limit、timeout、retry、idempotency、部分失敗の扱いが確認されています。
- [ ] HTTPS、CSP、frame 制御、security header、cookie、cache、content-type が確認されています。
- [ ] dependency advisory、lock、source、権限、更新方針、未修正リスクが確認されています。
- [ ] 保存・送信 data の暗号化、アクセス制御、保持、削除、backup が確認されています。
- [ ] error response と log／audit の masking、monitoring、alert が確認されています。
- [ ] blockchain を扱う場合は、wallet signature、transaction、cluster、account constraint を確認しています。

## Incident response

重大な脆弱性、secret 漏えい、認証回避、権限逸脱を発見した場合は、影響する経路の継続を止め、既存の security incident 手順と client／evaluator へ直ちに返します。承認された手順に従い、次の順序で対応します。

1. 検知内容、時刻、対象、影響を記録します。
2. 関係者へ連絡し、不要な拡散を防ぎます。
3. 漏えいした credential を revoke／rotation します。
4. 影響範囲を封じ込め、攻撃経路と権限を遮断します。
5. log、artifact、snapshot などの証拠を保全します。
6. 安全性を確認して復旧し、再発防止策を実装します。
7. 利用者向け応答には内部詳細、secret、stack trace を出しません。

調査中の残余リスク、未解決事項、期限、risk owner を完了扱いにせず、既存の承認方針に従って再評価します。

## 追加資料

一般的な脅威分類や Web の攻撃手法を調べるときは、次の資料を必要な範囲で参照します。Next.js や Supabase の資料は、その技術を実際に使う場合だけ読みます。

- [OWASP Top 10](https://owasp.org/www-project-top-ten/) は、Web アプリケーションの代表的な脅威分類を確認するときに読みます。
- [Next.js Security](https://nextjs.org/docs/security) は、Next.js 固有の security header や rendering 境界を確認するときに読みます。
- [Supabase Security](https://supabase.com/docs/guides/auth) は、Supabase の auth、RLS、data access を確認するときに読みます。
- [Web Security Academy](https://portswigger.net/web-security) は、Web の脆弱性シナリオと検証観点を具体化するときに読みます。

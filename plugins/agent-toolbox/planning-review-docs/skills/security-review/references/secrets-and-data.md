# 秘密情報と機密データ

secret、credential、秘密鍵、token、個人情報、決済情報、またはそれらを含む可能性のある設定・ログ・artifact を扱う場合に読みます。保存や送信が本当に必要かを先に判断し、必要な場合だけライフサイクルを設計します。

## 秘密情報の取り扱い

API key、password、token、private key、seed phrase を source、設定ファイル、テスト fixture、画像、ログ、error、response、artifact に hard-code または転載しません。値そのものをダミーに置き換えても、実値に似た値を出力する設計や、secret を含む fixture を commit する設計は避けます。

危険な例は、source や設定へ secret を直接書くことです。

```ts
const apiKey = 'sk-proj-xxxxx'
const dbPassword = 'password123'
```

概念例では、secret manager または環境変数から読み、未設定なら利用開始前に明示的なエラーにします。

```ts
const apiKey = process.env.API_KEY
if (!apiKey) throw new Error('API_KEY is not configured')
```

実際の secret の値、token の一部、秘密鍵、seed phrase をレビュー記録やテスト出力へ貼り付けません。機密値が漏えいした場合は、承認された運用手順に従って revoke／rotation し、履歴、CI log、cache、backup、artifact への残存と影響範囲を調べます。

環境変数を使う場合でも、次を確認します。

- `.env*`、ローカル設定、生成済み設定、debug dump が repository と artifact の ignore 対象になっています。
- Git の現在の差分だけでなく、履歴、tag、branch、CI log、issue、PR、画像に secret が残っていません。
- 本番の secret は、対象環境が提供する secret manager または同等の保護された保管先で管理し、通常の設定値と分離されています。
- secret の参照主体、権限、scope、期限、rotation、revoke、監査を定め、不要な component へ渡しません。
- 未設定、期限切れ、権限不足、rotation 中の失敗が安全に扱われ、secret を error に含めません。

## 機密データのライフサイクル

個人情報、決済情報、認証情報、wallet 情報などは、項目ごとに次を計画します。

- 収集・保存・送信する目的と、保存先およびアクセス主体を決めます。
- 保持期間と削除方法を決め、backup、replica、cache、検索 index に残る期間も確認します。
- 保存時と送信時の暗号化、鍵の管理、rotation、復旧時の保護を決めます。
- tenant、owner、role などの条件をデータの read／write へ適用し、過剰な service role や管理用 bypass を作りません。
- 削除、訂正、エクスポート、アクセス監査などの利用者要求と、失敗時に残るデータを扱います。

決済情報は、決済事業者の token や参照 ID を優先し、カード番号、CVV、認証情報を自前で保存・ログ出力しません。必要な場合は、保存理由、範囲、保持期間、アクセス制御、削除方法を明記します。

## ログとエラー

password、token、secret、カード番号、CVV、秘密鍵、個人情報をログへ出しません。業務上必要な識別には、user ID、request ID、または許可された範囲の下 4 桁など、復元できない値だけを使います。

利用者向けの error は generic な内容にし、stack trace、SQL、内部 URL、ファイル path、依存サービスの応答、credential を返しません。詳細な診断は server log に限定し、そこでも機密値をマスキングし、アクセス制御、保持期間、監査を適用します。

テスト、スクリーンショット、開発用 dashboard、CI artifact に本番値や本番値に似すぎた値を使いません。fixture の値から本番の個人や credential を推測できないことを確認します。

## 確認項目

- [ ] source、設定、fixture、画像、ログ、error、response、artifact、履歴に hard-code された secret がありません。
- [ ] `.env*` と生成物が ignore され、Git 履歴と CI log も確認されています。
- [ ] secret manager または環境変数の参照、未設定時の失敗、scope、rotation、revoke が定義されています。
- [ ] 機密データの保存先、保持期間、暗号化、アクセス主体、削除、backup、replica、cache が定義されています。
- [ ] ログと利用者向け error が password、token、カード情報、stack trace、SQL、内部 URL を漏らしません。
- [ ] テスト値、画像、CI artifact が本番の secret や個人情報を含みません。
- [ ] 漏えい時の停止、影響調査、credential rotation、証拠保全、連絡先が既存の incident 手順に結び付いています。

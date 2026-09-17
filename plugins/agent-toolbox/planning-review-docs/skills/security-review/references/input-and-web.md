# 入力と Web 境界

API、Web UI、Webhook、CLI、queue、file upload など、信頼できない入力を受け取る場合に読みます。入力を処理する component と、query、command、HTML、path、外部 API などの出力先を対応づけ、境界を越えるたびに安全性を確認します。

## Schema と入力検証

外部入力は `unknown` として扱い、schema で型、必須・nullable、長さ、範囲、形式、列挙値、許可する未知フィールドを明示的に検証します。allowlist を優先し、検証前の値を domain 処理、query、ログ、外部 API へ渡しません。検証エラーは利用者向けの安全な形式へ変換し、内部の stack trace や schema の実装詳細を返しません。

概念例では、`email`、`name`、`age` の型と範囲を schema に記載してから処理します。実装では、プロジェクトが採用している schema validator または同等機能を使います。

```ts
const CreateUserSchema = schema.object({
  email: schema.string().email(),
  name: schema.string().min(1).max(100),
  age: schema.number().integer().min(0).max(150),
})

const validated = CreateUserSchema.parse(untrustedInput)
```

空値、巨大値、不正型、未知フィールド、重複値、Unicode の正規化、content-type の不一致を、endpoint の契約に応じて確認します。

## File upload

upload はサイズ、MIME、拡張子、実体の file signature、圧縮展開後のサイズ、保存先、アクセス権限、保持期間を制限します。client が申告した MIME、拡張子、元のファイル名だけを信頼しません。保存時は生成した名前と許可した storage root を使い、path traversal、symbolic link、実行可能形式、公開 URL の推測を防ぎます。

概念例では、上限と許可形式を要件で固定し、検証を upload の保存前に行います。5 MB は旧例の上限であり、全ての project に強制する値ではありません。

```ts
function validateFileUpload(file: File) {
  const maxSize = 5 * 1024 * 1024
  const allowedTypes = ['image/jpeg', 'image/png', 'image/gif']
  const allowedExtensions = ['.jpg', '.jpeg', '.png', '.gif']

  if (file.size > maxSize || !allowedTypes.includes(file.type)) {
    throw new Error('Invalid file')
  }
  const extension = getExtension(file.name)
  if (!extension || !allowedExtensions.includes(extension)) {
    throw new Error('Invalid file')
  }
  verifyFileSignature(file, allowedTypes)
}
```

## Injection と出力コンテキスト

SQL は文字列連結せず、parameterized query、ORM、query builder の変数束縛を使います。query の識別子や sort key も allowlist で選び、値だけを parameter に渡します。shell、template、LDAP、NoSQL、OS command なども、入力を文字列として連結しない API を選びます。command 実行が必要な場合は、許可する command と引数を固定し、shell 解釈、redirect、環境変数の注入を避けます。

文字列連結は injection を許します。

```ts
const query = `SELECT * FROM users WHERE email = '${email}'`
await db.query(query)
```

値は parameterized query、ORM、query builder の変数束縛へ渡します。

```ts
await db.query('SELECT * FROM users WHERE email = $1', [email])
```

HTML は user content を context に応じて escape し、HTML を許可する必要がある場合だけ sanitizer の allowlist を使います。dynamic script、event handler、危険な URL scheme を許可しません。CSP は `default-src`、script、style、image、font、connect の許可先を最小化し、`unsafe-eval` や `unsafe-inline` は必要性と代替策を記録した場合だけ検討します。

```ts
const clean = sanitizer.sanitize(userHtml, {
  allowedTags: ['b', 'i', 'em', 'strong', 'p'],
  allowedAttributes: [],
})
return renderAsHtml(clean)
```

CSP の概念例では、許可先を明示し、不要な script 実行源を許可しません。

```text
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  connect-src 'self' https://api.example.com
```

path を受け取る処理は canonicalize 後に許可 root 内か確認し、`..`、絶対 path、symbolic link、archive 展開による traversal を拒否します。redirect 先、template、header、content-type も入力値をそのまま埋め込みません。

## CSRF と Cookie

cookie による session を使う state-changing request には、CSRF token または同等の origin／request 検証を設けます。cookie は少なくとも `HttpOnly; Secure; SameSite=Strict` を基準にし、cross-site が要件に必要な場合は緩和理由、代替防御、対象 endpoint を記録します。token を URL、ログ、error、画面へ出しません。

```text
if not verify_csrf_token(request.header("X-CSRF-Token")):
    return a generic 403 response
```

## API の濫用とネットワーク境界

API には IP または user 単位の rate limit を設け、認証、password reset、検索、upload、export など高コスト・高価値の処理は短い window と低い上限を検討します。limit 超過時の status、Retry-After、監査、burst、proxy 越しの client 識別を定義します。

通常 endpoint と高コスト endpoint で制限を分けます。たとえば 15 分あたり 100 回と 1 分あたり 10 回は構成例であり、実際の値は認証強度、処理コスト、利用形態から決めます。limit 超過時は 429 など既存契約の status と retry 情報を返し、認証情報や内部状態を漏らしません。

CORS は wildcard を避け、許可する origin、method、header、credential の組み合わせを明示します。server が URL を取得する場合は SSRF を前提に、scheme、host、port、redirect、DNS 解決後の private／link-local／metadata address、response size、timeout を制限します。外部 redirect は許可先を固定し、open redirect を作りません。

request と response の content-type、サイズ、timeout、compression、pagination、streaming の上限を endpoint ごとに決めます。大きな入力や遅い依存で worker、memory、connection pool が枯渇しないことを確認します。

## 検証項目

- [ ] 空値、巨大値、不正型、未知フィールド、境界値、Unicode、content-type 不一致を拒否または契約どおりに扱います。
- [ ] SQL、command、template、LDAP、NoSQL、path、redirect への入力が連結されず、allowlist、parameter、canonicalization が適用されています。
- [ ] upload のサイズ、MIME、拡張子、実体、保存先、path traversal、圧縮爆弾、実行権限を確認しています。
- [ ] user content の escape／sanitization、危険な URL、CSP、dynamic script の扱いを確認しています。
- [ ] state-changing request の CSRF、cookie 属性、origin 検証を確認しています。
- [ ] rate limit、CORS、SSRF、redirect、request／response size、content-type、timeout を endpoint ごとに確認しています。
- [ ] 不正入力、SQL injection payload、command injection、XSS payload、CSRF 不備、rate limit 超過、危険な upload、SSRF 先をテストまたは静的確認しています。

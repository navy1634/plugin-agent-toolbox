# 認証と認可

login、session、token、role、permission、tenant、管理者操作、所有権確認など、主体を識別または操作を許可する場合に読みます。採用する認証方式を先に明記し、方式に依存する安全な保存、検証、失効、監査を設計します。

## 認証と Token

認証方式、session／token の期限、失効、refresh、rotation、replay 防止、issuer、audience、署名アルゴリズム、署名鍵の保管・rotation・失効を確認します。JWT などの token は署名、`exp`、`nbf`、issuer、audience、必要な nonce を server side で検証し、client が申告した user、role、tenant、account を identity の根拠にしません。アルゴリズムを token の header から無条件に選びません。

browser で使う token は XSS に弱い `localStorage` へ置かず、用途に応じて `HttpOnly; Secure; SameSite=Strict` cookie などを選びます。cross-site が要件にある場合は、CSRF 防御と緩和理由を記録します。session fixation、logout 後の再利用、期限切れ、refresh token の再利用、複数 device の revoke を確認します。

概念例では、秘密値を URL や local storage に置かず、必要な属性を cookie に付けます。

```ts
res.setHeader(
  'Set-Cookie',
  `token=${token}; HttpOnly; Secure; SameSite=Strict; Max-Age=3600`,
)
```

password は平文や可逆暗号で保存せず、project が承認した password hashing と cost policy を使います。MFA、password reset、account recovery、login failure、credential rotation を追加する場合は、それぞれの abuse case と rate limit を計画します。

## 認可の決定

認可は server side で、主体、資源、操作、tenant／owner、状態の組み合わせを毎回検証します。UI、route の非表示、推測困難な ID、client が送る role のみで保護しません。default deny、最小権限、fail closed を基本にし、管理者や service role の bypass は対象操作、呼び出し元、監査を限定します。

たとえば削除処理では、対象資源の所有者または requester の role／permission を先に確認し、権限がなければ処理を実行せず 403 と監査イベントを返します。

```ts
const requester = await db.users.findUnique({ where: { id: requesterId } })
if (!requester || requester.role !== 'admin') {
  return response.json({ error: 'Forbidden' }, { status: 403 })
}
await db.users.delete({ where: { id: targetUserId } })
```

権限表には、主体、resource、operation、許可条件、拒否条件、状態遷移、監査対象を記載します。未認証（401）と認証済みだが許可されない場合（403）の外部契約は、既存 API と揃え、不要な user／resource の存在推測を許しません。

## Tenant と Row-Level Security

multi-tenant や DB の row-level security を使う場合は、tenant／owner 条件を全ての read／write、検索、集計、export、削除へ適用します。policy の default deny、migration 前後の状態、admin／service role の迂回経路、background job の tenant context を確認します。

概念例では、RLS を有効にし、requester の identity と row の owner を server 側で比較します。

```sql
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users view own data"
  ON users FOR SELECT
  USING (auth.uid() = id);

CREATE POLICY "Users update own data"
  ON users FOR UPDATE
  USING (auth.uid() = id);
```

policy の test は、同一 tenant、別 tenant、未認証、管理者、service role、存在しない resource、並行した role 変更を分けて確認します。

## 監査と確認項目

権限変更、login、logout、失敗、token revoke、機密 resource の read／write／delete を、actor、resource、operation、結果、request ID、時刻とともに監査します。token、password、秘密鍵を監査イベントへ出しません。監査ログの改ざん防止、保持期間、アクセス権限、alert を既存の運用方針へ結び付けます。

- [ ] 認証方式、session／token の期限、失効、refresh、rotation、replay 防止、issuer、audience、署名鍵が定義されています。
- [ ] browser token が安全に保存され、cookie 属性、CSRF、session fixation、logout、refresh token 再利用を確認しています。
- [ ] password、MFA、reset、recovery、login failure の安全な処理と rate limit を確認しています。
- [ ] 全ての保護操作で主体・資源・操作・tenant／owner を server side で検証し、default deny と最小権限が適用されています。
- [ ] UI や URL の秘匿だけに依存せず、client が申告した user、role、tenant、account を信頼していません。
- [ ] 401 と 403 の契約、存在推測への対策、権限拒否時の副作用と監査を確認しています。
- [ ] RLS または同等の tenant／owner 制御が全 read／write に適用され、service role の bypass が限定されています。
- [ ] 同一 tenant、別 tenant、未認証、権限不足、期限切れ、revoke 済み、再利用、role 変更、並行操作をテストまたは静的確認しています。

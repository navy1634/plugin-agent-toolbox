# 認証・認可の境界

この reference は、credential、token、session、role、permission、tenant、resource ownership の設計や API への組み込みを変更するときに読みます。脅威分析、secret の保管、credential の rotation、入力攻撃、監査要件は [security-review](../../../../planning-review-docs/skills/security-review/SKILL.md) と併用し、この reference ではバックエンドの層境界と業務契約を定めます。

## 認証と認可の責務

認証は「誰であるか」を確定し、認可は「その主体が対象 resource に対して操作できるか」を判定します。二つを同じ boolean や UI の表示制御へまとめません。

```text
credential / token
        ↓
presentation adapter で形式・署名・期限を検証
        ↓
authenticated principal を application へ渡す
        ↓
application / policy で principal・resource・operation を認可
        ↓
use case が repository/provider を通じて処理
```

- request adapter で credential を検証し、認証済み主体、tenant、scope、token の version など、application が必要とする最小の principal に変換します。
- application または policy 層で、主体・対象 resource・操作の組み合わせを server side で判定します。UI、URL の秘匿、client から送られた role だけに依存しません。
- resource の owner、tenant、組織、project、scope などの条件は、対象 resource に近い use case または domain policy で確認します。
- domain は JWT library、HTTP header、session cookie などの transport を直接知りません。必要な主体や権限を値と port で受け取ります。
- 未認証と認証済みだが権限がない状態は、既存の公開契約に従って 401 と 403 などへ区別します。resource の存在や他 tenant の情報を不要に明らかにしない設計も契約へ含めます。

## Token、session、principal の契約

採用する認証方式に応じて、次を明示します。

| 項目 | 決めること |
| --- | --- |
| credential | token、session、API key、mTLS などの形式、送信場所、複数 credential の優先順位 |
| token validation | 署名アルゴリズムの allowlist、鍵取得、期限、issuer、audience、not-before、token version、失効 |
| principal | subject、tenant、role、scope、認証強度など、use case へ渡す値と欠落時の扱い |
| lifecycle | expiry、refresh、logout、revoke、鍵更新、session rotation、clock skew |
| error contract | credential 不備、期限切れ、scope 不足、resource 不在、権限不足の status・code・公開範囲 |
| propagation | downstream API、queue、job、audit event へ渡す主体情報と、渡してはいけない credential |

token を検証しただけで resource authorization が完了したとみなさないでください。署名済み token の claim も外部入力として扱い、必要な tenant、role、scope、subject の形式と組み合わせを確認します。

## JWT 検証の Python 例

次は認証 adapter で JWT を検証し、application が扱う principal へ変換する構成例です。鍵や設定を source code に埋め込まず、issuer の公開鍵取得と rotation は採用する認証基盤の方式へ合わせます。

```python
from dataclasses import dataclass
from datetime import datetime

import jwt


@dataclass(frozen=True)
class Principal:
    subject: str
    tenant_id: str | None
    roles: frozenset[str]
    scopes: frozenset[str]


class InvalidCredential(Exception):
    pass


def verify_access_token(token: str, verification_key: str, *, issuer: str, audience: str) -> Principal:
    try:
        claims = jwt.decode(
            token,
            verification_key,
            algorithms=["RS256"],
            issuer=issuer,
            audience=audience,
            options={"require": ["sub", "exp", "iss", "aud"]},
        )
    except jwt.PyJWTError as error:
        raise InvalidCredential from error

    subject = claims.get("sub")
    if not isinstance(subject, str) or not subject:
        raise InvalidCredential

    tenant_id = claims.get("tenant_id")
    roles = claims.get("roles", [])
    scope_claim = claims.get("scope", "")
    if tenant_id is not None and not isinstance(tenant_id, str):
        raise InvalidCredential
    if not isinstance(roles, list) or not all(isinstance(role, str) for role in roles):
        raise InvalidCredential
    if not isinstance(scope_claim, str):
        raise InvalidCredential

    return Principal(
        subject=subject,
        tenant_id=tenant_id,
        roles=frozenset(roles),
        scopes=frozenset(scope_claim.split()),
    )


def issue_access_token(subject: str, signing_key: str, *, issuer: str, audience: str, expires_at: datetime, tenant_id: str | None = None) -> str:
    claims = {
        "sub": subject,
        "iss": issuer,
        "aud": audience,
        "exp": expires_at,
        **({"tenant_id": tenant_id} if tenant_id is not None else {}),
    }
    return jwt.encode(claims, signing_key, algorithm="RS256")
```

この例の validation key、issuer、audience、algorithm は構成から注入します。`datetime.now(timezone.utc)` などの時計の扱い、clock skew、JWK cache、鍵 rotation、失効確認、refresh token の保護は認証基盤の仕様を確認します。token の発行が必要な場合も、署名鍵を application の通常設定や test fixture へ書き込まず、発行者の責務と lifecycle を分けます。

## Role-Based Access Control の例

role と permission の対応は domain または policy 層に置き、endpoint の文字列比較へ散在させません。default deny を基本に、操作単位の permission と resource 条件を組み合わせます。

```python
from enum import StrEnum


class Role(StrEnum):
    ADMIN = "admin"
    MODERATOR = "moderator"
    USER = "user"


class Permission(StrEnum):
    READ = "read"
    WRITE = "write"
    DELETE = "delete"
    ADMIN = "admin"


ROLE_PERMISSIONS: dict[Role, frozenset[Permission]] = {
    Role.ADMIN: frozenset(
        {Permission.READ, Permission.WRITE, Permission.DELETE, Permission.ADMIN}
    ),
    Role.MODERATOR: frozenset(
        {Permission.READ, Permission.WRITE, Permission.DELETE}
    ),
    Role.USER: frozenset({Permission.READ, Permission.WRITE}),
}


def has_permission(role: Role, permission: Permission) -> bool:
    return permission in ROLE_PERMISSIONS.get(role, frozenset())


def can_delete_market(principal: Principal, market: Market) -> bool:
    if principal.tenant_id != market.tenant_id:
        return False
    if Role.ADMIN.value in principal.roles and has_permission(
        Role.ADMIN, Permission.DELETE
    ):
        return True
    return (
        Role.MODERATOR.value in principal.roles
        and has_permission(Role.MODERATOR, Permission.DELETE)
        and principal.subject == market.owner_id
    )
```

上記の `can_delete_market` は構成例であり、管理者が tenant を越えられるか、moderator が他人の resource を削除できるかは project の要件で決めます。role の default、未知の role、複数 role、scope と role の優先順位、owner check、tenant check、resource の soft delete を曖昧にしません。

### API adapter への適用

framework の dependency injection や middleware で主体を確定しても、use case へ権限判定を委譲します。

```text
DELETE /markets/{market_id}
        ↓ credential を検証
principal = authenticate(request)
        ↓ resource を読み、tenant・owner・permission を確認
DeleteMarket.execute(principal, market_id)
        ↓ 許可された場合だけ state を変更
response mapper が成功または公開 error へ変換
```

handler が token の claim にある `role` だけを見て削除したり、repository の内部 method を直接呼び出したりしてはいけません。認可判断に必要な resource の取得と、transaction 内での再確認が必要な場合は use case の責務として定義します。

## 監査と公開情報

認証・認可の結果を監査する場合は、主体、tenant、操作、対象 resource、結果、request/trace identifier、時刻を記録します。token、secret、不要な個人情報、生の credential は記録しません。拒否理由を利用者へ返す詳細と監査 log の詳細を分けます。

security-review と併用するときは、認証 bypass、権限昇格、tenant 越境、replay、session fixation、credential 漏えい、誤った error disclosure を threat または misuse case として受入条件へ対応づけます。

## 認証認可の検証観点

- credential の形式、validation、principal の schema、expiry、refresh/revoke、鍵更新、失効、clock skew の方式を確認します。
- request adapter が認証を担い、application または policy が主体・resource・operation・tenant・owner の認可を server side で判定することを確認します。
- 未認証、認証済みだが権限不足、resource 不在、tenant 越境の公開 status・code・情報量が契約に合うことを確認します。
- role、permission、scope、owner、tenant の default deny と未知値の扱いをテストまたは受入条件で確認します。
- token、secret、不要な個人情報が response、log、fixture、artifact、履歴へ漏れていないことを確認し、security-review の安全要件と残余リスクを記録します。

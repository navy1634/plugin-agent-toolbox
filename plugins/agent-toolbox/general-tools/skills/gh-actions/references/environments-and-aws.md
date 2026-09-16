# Environment と AWS

複数環境を扱う deploy workflow では、GitHub Environment、実行 ref、承認者、AWS account／region、OIDC role を一つの trust boundary として設計します。workflow 側は実行時の値の受け渡しと gate を定義し、IAM Role／Provider、network、resource の設計は `terraform` skill と repository が定める管理場所に委譲します。

## GitHub Environment

- `dev`、`stg`、`prd` などの Environment ごとに secrets／vars を分離し、deploy job の `environment:` で対象を明示します。
- 本番 Environment には required reviewer と branch protection を設定し、workflow の YAML や手動入力だけで approval gate を迂回できないようにします。
- Environment 固有の `AWS_DEPLOY_ROLE_ARN`、`API_ENDPOINT` などを Repository 直下の `PRD_ROLE_ARN` のような値へ平置きしません。
- `environment:` の値、job 名、PR check 名へ環境名を露出させ、誤った環境への plan／apply を UI で発見できるようにします。
- Environment の secret は、その Environment を宣言した job に承認後だけ公開される前提で、untrusted な PR job から参照できる構成を作りません。

## trigger から環境を決める

同じ deploy workflow で複数環境を扱う場合は、通常は trigger の branch／tag を許可した Environment へ対応付けます。`workflow_dispatch` は原則使用せず、実行確認または push／schedule などでは成立しない要件がある場合だけ、choice input と明示した ref を使って許可します。任意の環境名を文字列で受け取ったり、未知の ref を安全そうな環境へ暗黙に fallback したりしません。

| trigger | Environment | 条件 |
| --- | --- | --- |
| `push` to `main` | `prd` | protected branch への merge 済み commit だけを対象にする |
| `push` to `develop` | `dev` | develop branch の保護と実行権限を確認する |
| `workflow_dispatch`（例外） | `inputs.environment` | 原則使用せず、理由、実行 ref、choice input、Environment の allowlist、trust boundary を記録する |

```yaml
name: deploy

on:
  pull_request:
    types:
      - closed
    branches:
      - main
      - develop
    paths:
      - "**"
      - ".github/workflows/terraform-apply.yml"

permissions:
  contents: read

jobs:
  deploy:
    name: deploy (${{ github.ref == 'refs/heads/main' && 'prd' ||  'dev' }})
    environment: ${{ github.ref == 'refs/heads/master' && 'production' || 'development' }}
    permissions:
      contents: read
      id-token: write
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - uses: ./.github/actions/aws-login
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
      - name: deploy を実行
        run: uv run task deploy
```

この標準例では trigger が `main` と `develop` に限定されているため、Environment name の式に default branch への暗黙 fallback を置いていません。別の branch／tag を追加する場合は、表、trigger、Environment、権限、trust policy を同時に更新します。

## `workflow_dispatch` を例外で許可する場合

push や schedule では成立しない実行確認など、手動起動に固有の要件がある場合だけ `workflow_dispatch` を許可します。許可理由を PR または Issue と run input に記録し、`gh workflow run` では protected ref を `--ref` で明示します。input の型、必須性、選択肢、Environment、実行 ref の対応を allowlist にし、untrusted な PR head、任意の shell、任意の AWS account を選べるようにしません。

```yaml
on:
  workflow_dispatch:
    inputs:
      ref:
        type: choice
        description: 実行する protected ref
        options: [main, develop]
        required: true
      environment:
        type: choice
        description: deploy 先の Environment
        options: [dev, prd]
        required: true
      reason:
        type: string
        description: push／schedule では成立しない要件
        required: true

permissions:
  contents: read

jobs:
  deploy:
    name: deploy (${{ inputs.environment }})
    if: >-
      (inputs.ref == 'main' && github.ref == 'refs/heads/main' && inputs.environment == 'prd')
      || (inputs.ref == 'develop' &&  inputs.environment == 'dev')
    environment: ${{ inputs.environment }}
    permissions:
      contents: read
      id-token: write
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - uses: ./.github/actions/aws-login
        with:
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
      - name: deploy を実行
        run: uv run task deploy
```

この例の `ref` input と実際の `github.ref` の一致、Environment の choice、required reviewer、OIDC role の trust policy を確認できない場合は、手動起動を追加しません。実行確認だけが目的なら、まず通常の push／PR の結果確認で代替できないか検討します。

## AWS OIDC

long-lived な AWS access key を GitHub Secrets に保存せず、GitHub の OIDC token から短時間の credentials を発行します。認証 job にだけ `id-token: write` を与え、role ARN は対象 Environment の vars から渡します。

```yaml
permissions:
  contents: read

jobs:
  deploy:
    environment: prd
    permissions:
      contents: read
      id-token: write
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
        with:
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
```

OIDC を追加・変更・レビューするときは、少なくとも次を確認します。

- IAM role の trust policy が対象 repository、Environment または protected ref に限定されている
- `sub`、audience（通常 `sts.amazonaws.com`）、repository owner／name、ref の条件が意図した workflow と一致している
- 対象 account、region、role session name、session policy が deploy の責務に対して最小である
- `id-token: write` と AWS credentials が untrusted な PR、fork、検査 job、不要な step へ伝播していない
- AWS account／region の違いが Environment vars と job 名に反映され、誤った account への deploy を発見できる
- Role／Provider の作成、更新、削除の管理場所と変更手順が `terraform` skill および repository の source of truth と一致している

workflow 内へ access key、secret access key、session token をハードコードせず、AWS の resource／IAM の詳細をこの reference に重複定義しません。

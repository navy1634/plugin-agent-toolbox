# 再利用可能ワークフローと composite action

検証対象や目的が独立している CI は、別 workflow にします。
CI と CD／deploy も、別 workflow にします。
共通設定は再利用しても、workflow の責務は混ぜません。
job は段階、step は操作、action は再利用する関連 step 群を表します。

再利用は、権限・入力・失敗時の契約を一箇所で保つ必要がある場合に行います。
job 内で繰り返す関連 step は、composite action を既定の共有先にします。
reusable workflow は既定にしません。
同じ責務の job graph、Environment、approval、permissions を共有する場合に限ります。
composite action で表せない場合だけ採用します。
caller の secret、Environment、permissions、runner に暗黙に依存させません。

## 切り出し先の選択

| 対象 | composite action（既定） | reusable workflow（例外） |
| --- | --- | --- |
| 責務 | 一つの job 内で繰り返す、一つのまとまった操作の step 群 | 同じ責務を担う複数 job の graph、承認、Environment、job 単位の permissions |
| 定義場所 | `.github/actions/<name>/action.yml` | `.github/workflows/reusable-*.yml` |
| 呼び出し方 | step の `uses: ./.github/actions/<name>` | job の `uses: ./.github/workflows/<file>.yml` |
| 契約 | `inputs`、outputs、`runs.using`、各 step の失敗条件 | `on.workflow_call.inputs`、`secrets`、outputs、permissions |
| 適用判断 | 同じ責務の job／step 群が2箇所以上に現れる | 同じ責務の job graph が複数 workflow にまたがって必要で、composite action で表現できない |

## job、step、action の境界

- job は、一つの workflow の責務に属する段階を表します
- 異なる権限、runner、Environment、依存関係を持つ段階は、別 job にします
- step は一つのまとまった操作を担当します。名前と失敗条件を明示します
- lint、test、build など、個別結果を確認する検査は別 step にします
- 一つの task runner command が一つの検査を表す場合、その command は一つの step で実行できます
- composite action は、複数 workflow／job で再利用する関連 step をまとめます
- 行数削減だけを目的に、別の操作や責務を一つの action にまとめません
- reusable workflow は、同じ責務の job graph と権限・承認境界を共有する場合に使います
- CI の完了後に CD を行うだけの理由で、両方を同じ reusable workflow にしません

次のような処理は複数 workflow／job で繰り返す場合に composite action へ集約します。各 workflow に同じ設定を並べると、SHA、option、権限、region の変更漏れによる drift が起きます。

| 処理 | 切り出し先の例 |
| --- | --- |
| AWS OIDC 認証 | `.github/actions/aws-login` |
| runtime と task runner の setup（uv、pnpm、mise 等） | `.github/actions/setup-<lang>` |
| cache 復元と依存インストール | `.github/actions/install-deps` |
| ECR login と image push | `.github/actions/ecr-push` |

ただし、入力、権限、Environment、失敗時の扱いが異なる処理を無理に共通化しません。共通化によって caller ごとの trust boundary が見えなくなる場合は、別 component または別 workflow に分けます。

## reusable workflow の契約

reusable workflow は、job graph、Environment、approval、job 単位の permissions などを caller 間で共有する必要があり、composite action では表現できない場合だけ採用します。`workflow_call` の input は `type`、`required`、必要なら default と description を明示します。secret は名前と required を列挙し、`secrets: inherit` による暗黙の全量継承を避けます。callee 側の permissions、Environment、outputs、失敗条件を caller が確認できるようにします。

```yaml
# .github/workflows/reusable-terraform-plan.yml
name: reusable Terraform plan

on:
  workflow_call:
    inputs:
      working-directory:
        type: string
        required: true
        description: Terraform root の相対パス
    secrets:
      plan-comment-token:
        required: false

permissions:
  contents: read

jobs:
  plan:
    name: Terraform plan (${{ inputs.working-directory }})
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4.2.2
      - name: plan を実行
        working-directory: ${{ inputs.working-directory }}
        run: terraform plan -input=false
```

caller は必要な input と secret だけを渡します。`with` の値、secret、permissions、Environment は callee の契約と一致させ、caller の untrusted input をそのまま privileged な reusable workflow へ渡しません。

```yaml
jobs:
  terraform_plan:
    uses: ./.github/workflows/reusable-terraform-plan.yml
    with:
      working-directory: infrastructure/app
    secrets:
      plan-comment-token: ${{ secrets.PLAN_COMMENT_TOKEN }}
```

reusable workflow が Environment を使う場合は、callee 側で対象を固定するか、許可した choice と対応表を通してのみ選択させます。caller の `secrets: inherit`、暗黙の `GITHUB_TOKEN` write 権限、未検証の path／command を契約にしません。

## composite action の契約

composite action は `.github/actions/<name>/action.yml` に置き、必須 input、default、description、output、runtime、失敗時の挙動を明示します。
外部 action を内部で呼ぶ場合は release tag を指定し、`gh-actions-lock` の lockfile に含めます。

```yaml
# .github/actions/aws-login/action.yml
name: aws-login
description: OIDC で AWS にログインする
inputs:
  role-to-assume:
    required: true
    description: AssumeRole 対象の IAM Role ARN
  aws-region:
    required: false
    default: ap-northeast-1
    description: AWS リージョン
runs:
  using: composite
  steps:
    - uses: aws-actions/configure-aws-credentials@v4.0.2
      with:
        role-to-assume: ${{ inputs.role-to-assume }}
        aws-region: ${{ inputs.aws-region }}
```

呼び出し側は、role ARN や region を Environment vars などの source of truth から渡します。

```yaml
- uses: ./.github/actions/aws-login
  with:
    role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
    aws-region: ${{ vars.AWS_REGION }}
```

AWS OIDC の role trust policy、account、resource、Terraform の module／state は [Environments and AWS](environments-and-aws.md) と `terraform` skill の責務に従います。composite action は IAM resource の定義場所を隠すために使いません。

## 再利用時の確認

- caller と component の入力、出力、secret、permissions、Environment、runner、timeout が明示されている
- component 内の外部 action が `actions.lock` に記録され、`gh actions-lock --verify` で検証できる
- untrusted input が component 内で shell、path、command、Environment 名に直接展開されない
- action の失敗が caller で成功扱いにならず、`continue-on-error` の意味と最終判定が一致する
- cache、artifact、ログが caller 間で秘密値や untrusted data を混ぜない
- reusable workflow は job graph を、composite action は job 内の step を担当し、境界を越えて責務を重複させていない

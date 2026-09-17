# Terraform CI

Terraform は専用の task runner がない repository もあるため、GitHub Actions から Terraform の gate を直接実行する例外を認めます。対象 root、provider、account、region、backend、state、module の設計と安全境界は `terraform` skill に従い、この reference では workflow 上の実行順序、権限、Environment、結果の受け渡しだけを扱います。

## gate の分離

| gate | 実行条件 | 受入条件 |
| --- | --- | --- |
| check（format／validate／lint） | 変更された Terraform root | 単一の `check` job／check step で `terraform fmt -recursive -check`、`terraform validate`、`tflint --config .tflint.hcl --recursive` がすべて成功する |
| plan | check 成功後、PR ごと、対象 root ごと | `terraform plan` の差分を確認し、PR comment には `tfcmt plan --patch -- terraform plan ...` を使う |
| apply | main への merge 後（`workflow_dispatch` は例外条件を満たす場合だけ） | 対象 Environment の approval、trusted ref、対象 account／region、plan との対応を確認する |

format、validate、lint は checkout を含む単一の `check` job／check step にまとめ、plan と apply は別 job にします。check が失敗した場合に plan、comment、apply などの後続処理が privileged な操作を開始しない job graph にします。PR の plan comment に write 権限が必要な場合は、untrusted な plan 実行 job と trusted な comment 更新 job の permissions を分け、write 権限を untrusted code へ広げません。

## workflow の構成例

```yaml
name: Terraform CI

on:
  pull_request:
    paths:
      - '**/*.tf'
      - '**/*.tfvars'
      - '.tflint.hcl'

permissions:
  contents: read

concurrency:
  group: terraform-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  check:
    name: Terraform check (fmt / validate / lint)
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - name: Terraform fmt / validate / lint
        run: |
          terraform fmt -recursive -check
          terraform validate
          tflint --config .tflint.hcl --recursive

  plan:
    name: Terraform plan
    needs: check
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - run: terraform plan -input=false
```

この例は配置と依存関係を示す構成例です。実際の repository で必要な `terraform init`、対象 root、backend、provider credentials、plan file、PR comment の方法は既存の Terraform skill、task runner、CI の source of truth に合わせます。

## plan と apply

- PR の plan は変更の影響をレビューできる形で保存し、resource の追加、変更、削除、置換、権限、network、暗号化、コスト、state address を確認します。
- apply は PR の untrusted code や任意 branch から直接起動せず、main merge 後に対象 Environment approval を経て実行します。manual trigger が必要な場合は [Workflow の構成](workflow-structure.md) の `workflow_dispatch` 例外条件を満たすことを確認します。
- apply の job には `environment:`、`contents: read`、必要な場合だけ `id-token: write` を付与し、Environment vars の role ARN／region を使います。
- plan と apply が異なる commit、root、account、region、workspace、Environment に対して実行されないよう、plan の識別子と対象を記録します。
- `terraform apply -auto-approve` を main 以外で実行せず、production の apply、destroy、state 操作、権限変更は承認済み CI/CD 経路に限定します。
- apply 中の cancel、再実行、concurrency を安全に扱えるよう、同一 Environment の同時 deploy を一つに制限し、外部操作の idempotence と停止条件を確認します。

### PR comment と tfcmt

Terraform plan の結果を PR comment に投稿する場合は `tfcmt` を使い、Terraform command を `tfcmt` の後ろへ渡します。公式の呼び出し形は次のとおりです。

```yaml
jobs:
  plan_comment:
    name: Terraform plan comment
    needs: check
    permissions:
      contents: read
      pull-requests: write
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - name: Terraform plan を PR comment に投稿
        env:
          GITHUB_TOKEN: ${{ github.token }}
        run: tfcmt plan --patch -- terraform plan -input=false
```

この例の `plan_comment` job は、checkout する ref と実行 code が trusted であることを確認した経路でだけ使います。untrusted な `pull_request` の PR head を checkout して実行する job に `pull-requests: write` を与えません。untrusted な PR の plan が必要な場合は、read-only の plan job と、検証済みの plan 結果だけを扱う trusted な comment job を分離し、comment job で PR head の code、script、action を実行しない構成にします。分離を安全に実現できない場合は、write token を広げるために `pull_request_target` を使わず、自動 comment を追加しません。

tfcmt の公式 Getting Started は、Terraform、tfcmt、GitHub Access Token を要件とし、PR comment に `Pull Requests: Write`、PR label に `Issues: Write` を要求しています。comment だけなら job の `pull-requests: write` に限定し、label を使う場合だけ `issues: write` を追加します。`GITHUB_TOKEN` は `${{ github.token }}` から渡せますが、選択した tfcmt version が受け付ける環境変数を公式仕様で確認します。公式の環境変数仕様では `GITHUB_TOKEN` に加え、tfcmt 4.8.0 以降の `TFCMT_GITHUB_TOKEN` が案内されています。

binary の導入は repository の既存 task runner、toolchain、dependency 管理を source of truth とし、workflow に未検証の download URL、version、installer、checksum を推測で追加しません。既存の管理方法がない場合は、[tfcmt の公式 install 手順](https://suzuki-shunsuke.github.io/tfcmt/install/)で選択した version の binary、checksum／attestation、PATH 設定を確認してから追加します。GitHub Actions の built-in context から owner、repository、PR、SHA を取得できない event で使う場合は、[公式の環境変数仕様](https://suzuki-shunsuke.github.io/tfcmt/environment-variable/)と選択 version に従い、必要な値だけを明示します。

## AWS OIDC の受け渡し

AWS の credentials は long-lived access key ではなく `aws-actions/configure-aws-credentials` の OIDC を使います。role ARN は Repository／Environment vars で管理し、認証 job だけに `id-token: write` を付与します。

```yaml
jobs:
  apply:
    name: Terraform apply (prd)
    if: github.ref == 'refs/heads/main'
    environment: prd
    permissions:
      contents: read
      id-token: write
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: ./.github/actions/aws-login
        with:
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
      - run: terraform apply -input=false plan.tfplan
```

OIDC の trust policy、IAM Role／Provider、account、region、Terraform module／state の管理場所は `terraform` skill と repository の既存方針に合わせます。OIDC IAM Provider／Role は Terraform code として管理せず、外部の管理元を記録して Terraform で重複作成・変更しません。gh-actions の変更では、workflow が適切な Environment と role ARN を受け取り、untrusted な job に credentials が伝播しないことを確認します。

## 禁止パターン

| 禁止 | 代替 |
| --- | --- |
| PR の untrusted job から `terraform apply` | trusted な main merge または承認済み Environment の apply job |
| format／validate／lint の失敗を `continue-on-error` で通す | gate を失敗させ、原因を修正してから plan へ進む |
| workflow に環境別 role ARN や access key を直書きする | Environment vars と OIDC |
| plan と apply の対象 root／commit／account を暗黙に変える | 対象を input、Environment、plan metadata で固定して確認する |
| Terraform に task runner があるのに workflow だけ別コマンドを定義する | repository の task runner を source of truth にする |

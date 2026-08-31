{{- /* chezmoi:template:left-delimiter=[[ right-delimiter=]] */ -}}
---
name: gh-actions
description: GitHub Actions ワークフロー開発規約。権限最小化、action の SHA pin、OIDC、タスクランナー経由の実行、reusable workflow、Terraform CI、gh CLI 運用、DoD を含む。
---

# GitHub Actions 開発規約

## 基本方針

- `.github/workflows/*.yml` は「実行仕様」であり「実行手順の再定義」ではない。ローカルと同じタスクランナーを経由する
- `permissions:` は必ず明示。default は使わない
- 全 `uses:` は commit SHA で pin。バージョンはコメント併記
- secrets は GitHub Secrets / OIDC 経由でのみ渡す。ハードコード禁止
- AWS 認証は OIDC 必須（IAM Role/Provider は AWS Console 管理、terraform skill 参照）
- `actionlint` で構文検証してからコミット

## ディレクトリ構成

```
.github/
├── workflows/
│   ├── ci.yml              # PR で走る lint / test
│   ├── deploy-{env}.yml    # 環境別 deploy
│   ├── release.yml         # release 用
│   └── reusable-*.yml      # workflow_call の再利用 workflow
└── actions/
    └── <name>/action.yml   # composite action（プロジェクト内で再利用）
```

## 命名規則

| 対象 | 規則 | 例 |
|------|------|----|
| workflow ファイル | kebab-case、目的が一目で分かる名前 | `ci.yml`, `deploy-prd.yml` |
| workflow の `name:` | 日本語または英語で人間可読 | `CI (lint / test)` |
| job id | snake_case、短く | `lint`, `unit_test`, `build_image` |
| step の `name:` | 日本語で目的を書く | `依存関係をインストール` |
| reusable workflow | `reusable-*.yml` を前置 | `reusable-terraform-plan.yml` |

## permissions（最小権限）

- ワークフローの先頭で `permissions:` を明示し、default に依存しない
- 基本は `contents: read` のみ。書き込みが必要なジョブでスコープを広げる
- `id-token: write` は OIDC 認証を行うジョブでのみ付与
- 複数ジョブがある場合、job 単位で `permissions:` を上書きするのが望ましい

```yaml
permissions:
  contents: read

jobs:
  deploy:
    permissions:
      contents: read
      id-token: write   # OIDC 用
```

## secrets / vars

- 機密値は `secrets`、非機密の環境依存値は `vars` を使う
- 環境で値が変わるものは Repository 直下ではなく Environment スコープに置く（後述「環境切り分け」参照）
- long-lived な AWS access key を secrets に置くのは禁止。OIDC で置き換える
- workflow 内で `echo $SECRET` や `set -x` は禁止。マスクが必要なら `::add-mask::`

## 環境切り分け（GitHub Environments）（CRITICAL）

複数環境（dev / stg / prd 等）を扱うワークフローは、GitHub Environments を使って secrets / vars を環境ごとに分離する。Repository 直下の secrets / vars に環境別の値を並べる運用は禁止。

- 各環境ごとに Environment を作成し、環境固有の値（`AWS_DEPLOY_ROLE_ARN`、`API_ENDPOINT` 等）は Environment vars / secrets に置く
- prd Environment には **required reviewer** と **branch protection** を設定して approval ゲートを設ける
- job に `environment:` を宣言すると、その環境の secrets / vars のみ露出する。他環境の値は参照できない
- どの環境で動くかは job の `environment:` で明示し、暗黙のフォールバックに頼らない

### base branch から動的に環境を切り替える

同一の deploy workflow で複数環境を扱う場合、実行トリガーの ref から環境名を決定する。人手で入力させず、ブランチと環境の対応を workflow に固定する。

| トリガー | 環境 |
|---------|------|
| `push` to `main` | `prd` |
| `push` to `develop` | `dev` |
| `workflow_dispatch`（手動） | `inputs.environment` |

```yaml
on:
  push:
    branches: [main, develop]
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [dev, prd]
        required: true

jobs:
  deploy:
    environment: >-
      ${{
        github.event_name == 'workflow_dispatch' && inputs.environment
        || github.ref == 'refs/heads/main' && 'prd'
        || 'dev'
      }}
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - uses: ./.github/actions/aws-login
        with:
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
      - run: uv run task deploy
```

- `role-to-assume` の ARN は Environment vars に置くことで、`vars.AWS_DEPLOY_ROLE_ARN` が自動で環境に応じた値になる
- Environment 名を job のログ・PR チェック名にも露出させる（誤った環境で apply されたことを目視で気付けるように）

## action の pin（CRITICAL）

- 全 `uses:` は commit SHA で pin する。`@v4` などのタグ参照は禁止
- バージョンはコメントで併記し、Dependabot / Renovate で SHA 更新を追跡する
- 公式 action (`actions/*`) も例外なく pin

```yaml
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
- uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
```

## 実行制御

- `concurrency:` で同ブランチ・同 PR の並走を止める。in-progress を cancel するのが基本
- `timeout-minutes:` を job に必ず明示（default の 360 分は長すぎる）
- `if:` は明示。skip 条件を暗黙にしない

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    timeout-minutes: 15
```

## matrix

- `fail-fast: false` は失敗を最後まで観測したいときのみ設定（default は fail-fast: true）
- 例外行は `include` / `exclude` で明示
- matrix 変数は job の `name:` に埋め込み、UI 上で識別できるようにする

## cache

- `actions/cache` の key はロックファイルのハッシュを含める（再現性）
- restore-keys は prefix 一致で fallback を効かせる
- 巨大な cache は per-branch でなく main の cache を restore する

## タスクランナー経由の実行（CRITICAL）

CI で lint / format / test / build を実行するときも、ローカルと同じタスクランナーを経由する。個別のツール（ruff、mypy、pytest、eslint、prettier、tsc など）を workflow から直接呼ばない。

| 言語 / ツール | 呼び出し方 |
|--------------|----------|
| Python (uv + taskipy) | `uv run task lint` / `uv run task test` |
| Node.js (pnpm / npm) | `pnpm lint` / `pnpm test`（`npm run lint`） |
| mise | `mise run lint` / `mise run test` |
| Makefile | `make lint` / `make test` |

**理由**: ローカルと CI で lint/test 設定が乖離すると、CI 通過とローカル通過が一致しなくなる。プロジェクトのタスクランナーが正しい実行方法の source of truth。workflow 側にツールのオプションや対象パスを書き込むのは、そのソースを分裂させる行為で禁止。

例外は Terraform（`terraform fmt` などタスクランナーを持たないことが多い）。この場合も、コマンド定義はワークフロー内に閉じ込め、ローカル DoD と同じコマンドを使う。

## Reusable Workflow / Composite Action

- 同じジョブが 2 箇所以上に現れたら reusable workflow か composite action に切り出す
- reusable workflow: `on: workflow_call:` の input / secret を明示、`type:` と `required:` を書く
- composite action: `.github/actions/<name>/action.yml` に置く
- どちらも独立してテスト可能にする（呼び出し元に依存しない）

### 複数回使う前提の処理は必ず切り出す（CRITICAL）

AWS 認証、依存インストール、キャッシュ復元など、複数の workflow / job で繰り返す処理は必ず composite action として `.github/actions/<name>/` に切り出す。各 workflow に同じ `uses:` を並べる運用は禁止。理由: SHA 更新・オプション変更のたびに複数ファイルを触ることになり、drift の温床になる。

代表例: AWS OIDC 認証 → `.github/actions/aws-login/action.yml`

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
    - uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
      with:
        role-to-assume: ${{ inputs.role-to-assume }}
        aws-region: ${{ inputs.aws-region }}
```

呼び出し側は `uses: ./.github/actions/aws-login` の 1 行に集約される。SHA 更新やリージョン変更は composite action の 1 箇所を直せば全 workflow に反映される。

切り出し対象の目安:

| 処理 | 切り出し先 |
|------|----------|
| AWS OIDC 認証 | `.github/actions/aws-login` |
| 言語ランタイム + タスクランナーのセットアップ（uv / pnpm / mise 等） | `.github/actions/setup-<lang>` |
| キャッシュ復元 + 依存インストール | `.github/actions/install-deps` |
| ECR ログイン + イメージ push | `.github/actions/ecr-push` |

## Terraform ワークフロー

- fmt gate: `terraform fmt -recursive -check`
- lint gate: `terraform validate` + `tflint --config .tflint.hcl --recursive`
- plan: PR で毎回実行し、差分を PR にコメント
- apply: main マージ後または `workflow_dispatch` + Environment approval
- AWS 認証は `aws-actions/configure-aws-credentials` で OIDC。`role-to-assume` の ARN は Repository/Environment vars で管理
- OIDC の IAM Role / Provider は Terraform で管理しない（terraform skill の OIDC IAM Resources 節参照）

## 禁止パターン

| 禁止 | 代替 |
|------|------|
| `uses: <name>@vN`（タグ / ブランチ参照） | `uses: <name>@<sha> # vX.Y.Z` |
| workflow から個別ツール直接呼び出し（`ruff`, `pytest`, `eslint` 等） | タスクランナー経由（`uv run task lint` 等） |
| workflow 内で `echo $SECRET` / `set -x` | secret を print しない、必要なら `::add-mask::` |
| `pull_request_target` で PR head を checkout | 使う場合は base の SHA を明示、permissions を厳格化 |
| long-lived AWS access key を secrets に保存 | OIDC + `aws-actions/configure-aws-credentials` |
| `permissions:` を書かない（default 依存） | 明示的に最小権限を宣言 |
| `timeout-minutes:` 未設定 | job ごとに明示 |
| `terraform apply -auto-approve` を main 以外で | Environment approval 必須 |
| self-hosted runner を isolation なしで使う | ephemeral runner / container jobs |
| workflow ファイルを本番運用中に無停止で書き換える | 一時的な `if:` で無効化してからマージ |
| 環境別の値を Repository secrets / vars に平置き（`PRD_ROLE_ARN` 等） | Environment vars / secrets で環境スコープに分離 |
| `if: github.ref == 'refs/heads/main'` を各 step に散らして環境分岐 | job の `environment:` で一括切替 |
| 各 workflow に `aws-actions/configure-aws-credentials` を直接並べる | `.github/actions/aws-login` に集約 |

## gh CLI での確認・運用（補助）

ワークフロー編集後の動作確認と、失敗調査に使う。

| コマンド | 用途 |
|---------|------|
| `gh workflow list` | workflow 一覧を確認 |
| `gh workflow run <file> -f key=value` | `workflow_dispatch` を手動トリガー |
| `gh run list --workflow <file> --limit 10` | 直近の実行を確認 |
| `gh run watch <run-id>` | 実行中の run を追跡 |
| `gh run view <run-id> --log-failed` | 失敗ジョブのログのみ表示 |
| `gh run view <run-id> --log` | 全ログを表示 |
| `gh run rerun <run-id> --failed` | 失敗ジョブのみ再実行 |
| `gh run cancel <run-id>` | 実行中のジョブをキャンセル |

- `gh` 実行時に破壊的操作（`gh run cancel`、`gh workflow disable` 等）はユーザー確認を取る
- workflow の変更を CI で確認せずマージするのは禁止。PR で必ず緑を確認する

## Definition of Done

workflow 変更時のチェックリスト（全 PASS 必須）。

- [ ] `permissions:` を明示し最小化した
- [ ] 全 `uses:` を commit SHA で pin し、バージョンをコメント併記した
- [ ] `concurrency:` と `timeout-minutes:` を設定した
- [ ] secrets を print / echo していない
- [ ] AWS 認証は OIDC（long-lived key なし）
- [ ] 複数環境がある場合は Environment で分離し、環境依存の値は Environment vars / secrets に置いた
- [ ] 環境判定は base branch or `workflow_dispatch` 入力から動的に行い、workflow に固定した（人手で毎回選ばせない）
- [ ] AWS 認証など複数回登場する処理は `.github/actions/<name>/` に切り出し、workflow は 1 行呼び出しに集約した
- [ ] lint / test はタスクランナー経由で呼んでいる（個別ツール直呼び出しなし）
- [ ] `actionlint` で構文検証済み
- [ ] PR で workflow が実際に走り、緑になったことを確認してからマージ

### 検証コマンド例

```bash
# ローカルで actionlint
actionlint .github/workflows/*.yml

# PR 作成後、CI が回ったら
gh run list --limit 5
gh run view <run-id> --log-failed   # 失敗した場合
```

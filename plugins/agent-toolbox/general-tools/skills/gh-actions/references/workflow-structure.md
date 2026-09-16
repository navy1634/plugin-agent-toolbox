# Workflow の構成

workflow の責務、実行対象、job 間の依存、実行制御を一つの構成として確認します。既存 repository の構成と命名がある場合は、それを優先します。

## ディレクトリ構成

標準的な配置は次のとおりです。workflow は job の組み合わせ、composite action は job 内で繰り返す step 群として責務を分けます。`.github/workflows` に置く reusable workflow は通常の workflow と同じ場所で実行定義を構成するため、composite action で表現できない job 単位の共有が必要な場合だけ使います。

```text
.github/
├── workflows/
│   ├── ci.yml              # PR で lint / test を実行する
│   ├── deploy-{env}.yml    # 環境別に deploy する
│   ├── release.yml         # release を作成・公開する
│   └── reusable-*.yml      # workflow_call（composite で表現できない job 単位の共有）
└── actions/
    └── <name>/action.yml   # repository 内で再利用する composite action
```

workflow のファイル名は kebab-case とし、目的が一目で分かる名前を付けます。`name:`、job id、step の `name:` も実行結果を読む人が目的と対象を識別できる値にします。

YAML の改行、インデント、整形は repository の formatter、lint 設定、既存 workflow に合わせます。この reference は手動の改行規則を新たに定めず、formatter がある場合はその出力を採用します。

| 対象 | 規則 | 例 |
| --- | --- | --- |
| workflow ファイル | kebab-case、目的を表す | `ci.yml`、`deploy-prd.yml` |
| workflow の `name:` | 日本語または英語で人間可読 | `CI (lint / test)` |
| job id | snake_case、短くする | `lint`、`unit_test`、`build_image` |
| step の `name:` | 実行目的と対象を書く | `依存関係をインストール` |
| reusable workflow | `reusable-` を前置し、例外用途であることを確認する | `reusable-terraform-plan.yml` |

## trigger と実行対象

event、ref、checkout 対象、実行 actor、利用できる token／secret を一緒に確認します。event の値や PR 由来の code は untrusted input になり得るため、workflow を起動できることだけで trusted とみなしません。

| trigger | 実行対象の決め方 | 設計時の確認 |
| --- | --- | --- |
| `pull_request` | PR の merge ref または明示した commit | fork を含む untrusted code に secret、OIDC、write 権限を渡さない |
| `push` | 許可した branch／tag の pushed commit | branch protection と push 権限を確認し、deploy 対象を固定する |
| `workflow_dispatch`（例外） | 手動実行時の明示した ref と choice input | 原則使用せず、理由、ref、input、Environment の allowlist、trust boundary を記録する |
| `schedule` | 定義した cron の default branch | 時刻ずれ、重複実行、secret と外部操作の影響を確認する |

CI、plan、deploy の trigger を同じにせず、untrusted な検査と privileged な操作を job または workflow で分けます。`workflow_dispatch` は原則使用せず、実行確認または push／schedule などでは成立しない要件がある場合だけ許可します。許可時は、使用理由、実行 ref、input、Environment の allowlist、trust boundary、承認条件を workflow と運用記録に残します。手動入力で任意の shell や任意の Environment を指定できる構成は採用しません。

## job graph と step

job 間の依存は `needs`、job の結果は outputs、step 間のデータは明示的な input／output で表します。job が並列でよい場合も、独立していることが読み取れる構成にします。

```yaml
name: CI (lint / test)

on:
  pull_request:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    name: lint
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - name: lint を実行
        run: uv run task lint

  unit_test:
    name: unit test
    needs: lint
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - name: test を実行
        run: uv run task test
```

- `needs` を省略して実行順を暗黙にせず、後続 job が前段の失敗を無視する `if: always()` を必要な cleanup 以外で使いません。
- `if:` は skip、cleanup、承認後処理などの意図を表す条件に限定します。branch や Environment の判定を各 step に散らさず、job の `environment:` や job-level condition でまとめます。
- `continue-on-error` は許容する失敗、後続処理、最終的な成功判定を定義した場合だけ使います。検査失敗を成功に見せる目的では使いません。
- job の `timeout-minutes` は必ず設定し、step の command が無期限に待機しないようにします。
- output、artifact、cache、ログは後続処理に必要な最小限とし、secret、credential、個人情報を含めません。

## concurrency と再実行

同じ branch または PR の古い CI を残す必要がなければ、workflow と ref または PR を含む group にまとめて `cancel-in-progress: true` を設定します。deploy、migration、外部 API への非可逆操作は途中キャンセルが安全とは限らないため、同時実行を一つに制限するか、cancel しない構成と安全な停止条件を選びます。

再実行では、対象 ref、permissions、Environment、承認境界が初回より広がらないことを確認します。外部操作は同じ run を再実行しても二重作成・二重公開が起きないよう idempotent にし、idempotence を保証できない場合は run 単位の状態確認と停止条件を用意します。

## matrix

- 複数 runtime、OS、provider version などを試す場合は matrix の値を明示し、job の `name:` に埋め込んで結果を識別できるようにします。
- `fail-fast: false` は、ある組み合わせの失敗後も全組み合わせの結果を観測する必要がある場合だけ設定します。不要な場合は既定の fail-fast を使います。
- 例外行は `include`、除外行は `exclude` で理由が分かる形にします。matrix 値から secret 名や任意の command を組み立てません。

```yaml
strategy:
  fail-fast: false
  matrix:
    python-version: ['3.12', '3.13']

name: test (Python ${{ matrix.python-version }})
```

## cache

cache は再現性と機密性を優先して設計します。

- cache key に OS、runtime、依存関係、lock file の hash を含めます。例として Python なら `hashFiles('**/uv.lock')`、Node.js なら `hashFiles('**/pnpm-lock.yaml')` を使います。
- `restore-keys` は prefix 一致の fallback として使いますが、異なる依存関係を完全一致と誤認しないよう、復元後に依存関係を検査します。
- 巨大な cache を branch ごとに複製せず、必要なら main の cache を restore してから対象 ref の key を優先します。
- secret、credential、生成された個人情報を cache に保存しません。cache の書き込みが untrusted な PR と trusted branch の間で混ざらないよう key と権限を確認します。

```yaml
- name: 依存関係の cache を復元
  uses: actions/cache@<検証済みコミットSHA> # v4.x
  with:
    path: ~/.cache/uv
    key: ${{ runner.os }}-uv-${{ hashFiles('**/uv.lock') }}
    restore-keys: |
      ${{ runner.os }}-uv-
```

実際の workflow では `<検証済みコミットSHA>` を残さず、action の commit SHA と version comment を確定させます。

# 検証と Definition of Done

workflow の変更は、ローカルの構文・静的確認、GitHub 上の CI 実行、deploy／外部環境の結果を混同せずに確認します。actionlint が成功しても、event、権限、secret、Environment、実際の job graph が意図どおりに動いた証拠にはならないため、実行範囲ごとに結果を報告します。

## 変更前後の確認

対象 workflow と composite action を確認してから、次を実施します。

1. `.github/workflows/*.yml` と `.yaml`、`.github/actions/**/action.yml` の変更範囲を特定する。
2. `actionlint` と YAML 構文、全外部 `uses:` の commit SHA pin、version comment を確認する。
3. `permissions:`、job の `needs`、`if:`、`timeout-minutes`、`concurrency`、matrix、cache、artifact の境界を確認する。
4. trigger ごとの ref、actor、checkout code、untrusted input、secret／OIDC、Environment、write token の境界を確認する。
5. `workflow_dispatch` を原則使用していないことを確認する。使用している場合は、push／schedule では成立しない要件または実行確認という理由、`--ref` と実際の ref、input の型・allowlist、Environment、approval、trust boundary の記録を確認する。
6. CI の lint／format／test／build が repository の task runner と同じコマンド・設定で実行されることを確認する。
7. deploy が対象 Environment、trusted ref、approval、account／region、plan と apply の対応を満たすことを確認する（該当する場合）。

## ローカル検証

```bash
# workflow の actionlint
actionlint .github/workflows/*.yml

# YAML 拡張子が混在する場合は対象を追加して確認する
actionlint .github/workflows/*.yaml
```

CI の lint／format／test／build は、個別ツールを workflow から直呼び出しせず、repository が定める task runner を source of truth とします。

| 言語／仕組み | 呼び出し例 |
| --- | --- |
| Python（uv + taskipy） | `uv run task lint`、`uv run task test` |
| Node.js（pnpm／npm） | `pnpm lint`、`pnpm test`、`npm run lint` |
| mise | `mise run lint`、`mise run test` |
| Makefile | `make lint`、`make test` |

workflow 側へ ruff、mypy、pytest、eslint、prettier、tsc などの option や対象 path を複製しません。Terraform に専用 task runner がない場合だけ、[Terraform CI](terraform-ci.md) の `terraform fmt`、`terraform validate`、TFLint、plan gate を workflow 内で実行します。

## GitHub 側の確認

workflow 編集後の実行確認と失敗調査には `gh` CLI を使います。状態を変えるコマンドは、依頼された対象と理由を確認してから実行します。

| コマンド | 用途 |
| --- | --- |
| `gh workflow list` | workflow 一覧と有効状態を確認する |
| `gh workflow run <file> -f key=value` | `workflow_dispatch` の例外理由、ref、allowlist、trust boundary を確認済みの場合だけ input 付きで手動起動する |
| `gh workflow run <file> --ref <trusted-ref> -f key=value` | 許可した protected ref を明示して例外の workflow_dispatch を起動する |
| `gh run list --limit 5` | repository 全体の直近 run を確認する |
| `gh run list --workflow <file> --limit 10` | 直近の run と branch／commit を確認する |
| `gh run watch <run-id>` | 実行中の run を追跡する |
| `gh run view <run-id> --log-failed` | 失敗 job のログだけを確認する |
| `gh run view <run-id> --log` | run 全体のログを確認する |
| `gh run rerun <run-id> --failed` | 失敗 job だけ再実行する |
| `gh run cancel <run-id>` | 実行中の run をキャンセルする |
| `gh workflow disable <file>` | workflow を無効化する（依頼された場合だけ） |

`gh run cancel`、`gh workflow disable`、rerun などの状態変更を、単なる確認のために実行しません。workflow の変更を CI で確認せずに merge せず、deploy の成功は GitHub run の成功だけで判断せず、対象環境側の証拠も確認します。

## Definition of Done

workflow 変更時は、該当しない項目を理由付きで明示し、確認できた項目を全て満たします。

- [ ] `permissions:` を明示し、job ごとに最小化した
- [ ] 全外部 `uses:` を commit SHA で pin し、version をコメント併記した
- [ ] `concurrency:` と `timeout-minutes:` を設定し、cancel／再実行の安全性を確認した
- [ ] secrets、`GITHUB_TOKEN`、OIDC credentials を print／echo、artifact、cache、output に出していない
- [ ] AWS 認証は OIDC で、long-lived key を保存していない（AWS を使う場合）
- [ ] 複数環境を Environment で分離し、環境依存の値を Environment vars／secrets に置いた（該当する場合）
- [ ] 原則 push／PR／schedule で実行し、`workflow_dispatch` を使う場合は理由、実行 ref、input、Environment の allowlist、trust boundary、承認条件を記録した
- [ ] AWS 認証など複数回登場する処理を `.github/actions/<name>/` に集約した（該当する場合）
- [ ] lint／format／test／build を task runner 経由で呼び出し、個別 tool の直呼び出しを追加していない
- [ ] Terraform の fmt、validate、TFLint、PR plan、承認済み apply の gate を確認した（Terraform の場合）
- [ ] `actionlint` と必要な YAML／静的検証が成功した
- [ ] PR で workflow が実際に走り、関連 check が緑になったことを確認してから merge する
- [ ] ローカル、GitHub CI、deploy／外部環境の証拠と未確認事項を分けて報告した

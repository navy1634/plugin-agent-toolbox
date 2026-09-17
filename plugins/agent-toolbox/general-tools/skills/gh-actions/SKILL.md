---
name: gh-actions
description: GitHub Actions の workflow、reusable workflow、composite action を追加・変更・レビューし、CI/CD の実行境界と結果を確認するときに使う。
---

# GitHub Actions

GitHub Actions の workflow は、イベントを受けて検査、build、plan、deploy などの job を実行する仕様です。workflow に実装手順を重複させず、repository が定める task runner と source of truth を使い、ローカルと CI の結果が同じ契約になるようにします。既存の運用方針がある場合は、それを確認してから変更します。

## 適用条件

- `.github/workflows/*.yml` または `.yaml` の新規作成、変更、レビューを行うときに使います。
- `.github/actions/<name>/action.yml` の composite action、`workflow_call` の reusable workflow、CI、release、plan、deploy の設計を扱うときに使います。
- Git の branch、commit、Issue、Pull Request 本文などの汎用操作は `git-workflow` の規約に従い、この skill で重複定義しません。
- Terraform の HCL、module、state、resource、IAM resource の設計は `terraform` の規約に従い、この skill では workflow から Terraform を安全に呼び出す境界だけを定義します。

## 常時適用する契約

### workflow の責務

- workflow は、trigger、信頼境界、job graph、外部サービスへの受け渡し、結果の可視化を定義します。アプリケーションの実装手順や lint／test のオプションを workflow に複製しません。
- CI、plan、deploy、release の責務を分け、権限、secret、environment、失敗時の影響が異なる処理を一つの無制限な job にまとめません。
- workflow、job、step の入力、出力、成果物、失敗条件を追跡できるようにします。暗黙の環境変数、暗黙の job 順序、暗黙の権限継承に依存しません。
- YAML の改行、インデント、整形規則をこの skill で新たに固定せず、repository の formatter、lint 設定、既存 workflow の構成を source of truth とします。手動で見た目だけを整える変更を追加しません。

### 変更前に確定する情報

既存の workflow、composite action、task runner、runtime、branch／tag、Environment、Repository／Environment secrets と vars、branch protection、外部 action の version と SHA、直近の CI 結果を確認します。併せて、次の対応を変更前に決めます。

- どの event、ref、actor が実行を開始するか
- event の値や checkout する commit が信頼済みか
- どの job が untrusted code を扱い、どの job が secret、OIDC、write 権限を必要とするか
- job 間の依存、並行実行、cancel、再実行、timeout、失敗時の停止または後続処理
- CI の成功、plan の承認、deploy の実行、release の公開を何で受入とするか

既存の convention と task runner が判明しないまま、workflow に新しい実行方式や個別ツールの直接呼び出しを追加しません。

### trigger と信頼境界

- event、実行 ref、actor、checkout 対象、token／secret の可用性を一組の trust boundary として設計します。`pull_request`、fork からの PR、`workflow_dispatch` の input、issue／PR の title・body・comment、branch 名、commit message などは untrusted input として扱います。
- untrusted code を checkout して実行する job には secret、OIDC、書き込み権限を渡しません。検査 job と privileged な plan／deploy job を分離し、trusted な base branch、tag、Environment approval などで昇格条件を明示します。
- `pull_request_target` で PR head の code を checkout または実行しません。採用する場合も base 側の固定 SHA、入力の allowlist、最小権限、secret の非露出を確認します。
- `workflow_dispatch` は原則使用せず、実行確認または push／schedule などでは成立しない要件がある場合だけ許可します。許可する場合は、使用理由、実行 ref、input、Environment の allowlist、trust boundary、承認条件を明示し、任意の環境名や shell 断片をそのまま実行しません。

### job と step の依存関係

- job 間の順序とデータ受け渡しは `needs` と明示した outputs で表現します。step の順序、入力、出力、失敗時の動作も暗黙にしません。
- `if:` は skip、cleanup、承認後の処理など意図した条件にだけ使い、条件分岐を各 step に散らしません。`continue-on-error` や失敗を無視する処理は、許容する失敗と後続への影響を記録した場合だけ使います。
- job ごとに runner、`timeout-minutes`、permissions、必要なら `environment` を明示します。成果物、cache、ログには secret、credential、個人情報を含めません。
- 失敗は原則として fail closed とし、検査の失敗を成功に見せません。再実行で同じ ref、権限、Environment 境界が維持され、外部操作が重複しても安全になるよう idempotence と rollback／停止条件を確認します。

### 権限、secret、第三者 action

- workflow の `permissions:` を明示し、基本は `contents: read` とします。書き込みが必要な job だけ job 単位で scope を広げ、`id-token: write` は OIDC 認証 job に限定します。
- secret は GitHub Secrets または OIDC から必要な job／Environment へだけ渡し、非機密の環境依存値は `vars` に置きます。secret を shell、ログ、artifact、cache、job output に出力せず、long-lived credential を新たに保存しません。
- 外部 `uses:` は commit SHA で pin し、コメントで元の version を併記します。公式 action も例外にせず、更新時には取得元、release、依存、必要権限、実行コードを確認します。
- untrusted input を shell や action input に渡す場合は allowlist、環境変数への分離、適切な quoting を行い、文字列連結による script injection を許しません。

### 再利用と責務の境界

- 同じ job／step 群が複数箇所に現れる場合は、まず `.github/actions/<name>/action.yml` の composite action で job 内の手順を共有します。`.github/workflows` に置かれる reusable workflow は通常の workflow と実行定義が混ざるため既定にせず、job graph、Environment、approval、job 単位の permissions などを共有し、composite action で表現できない場合だけ採用します。
- reusable workflow は `workflow_call` の input、secret、type、required、outputs、permissions を呼び出し側との契約として明示します。composite action は input、output、失敗時の挙動を明示し、caller の暗黙の secret や環境に依存させません。
- AWS 認証、runtime／task runner の setup、cache と依存インストール、ECR login と image push など、複数 workflow／job で繰り返す処理は一箇所へ集約し、設定の drift を防ぎます。

### CI、CD、deploy の受入条件

- lint、format、test、build は repository の task runner を経由し、CI だけが異なるコマンドやオプションを持たないようにします。Terraform に専用 task runner がない場合の `fmt`、`validate`、TFLint、plan は [Terraform CI](references/terraform-ci.md) の境界に従います。
- Terraform plan の結果を PR comment に投稿する場合は `tfcmt plan --patch -- terraform plan ...` を使い、必要な write 権限を trusted な comment 経路だけへ付与します。untrusted な PR code に comment 用 token を渡しません。
- deploy は CI と分離し、trusted な ref、対象 Environment、必要な reviewer、対象 account／region、権限、承認、plan と apply の対応を確認します。本番相当の変更を PR や untrusted な ref の権限だけで apply しません。
- concurrency は workload の性質に合わせて設定します。CI や同一 PR の古い実行は通常 cancel しますが、deploy や中断できない外部操作は同時実行を一つにするか、安全な停止条件を定めます。
- workflow の変更は、構文検証だけで完了扱いにせず、対象 event、branch／tag、job graph、権限、secret 非露出、CI の実行結果、deploy を変更した場合の実環境側の結果を分けて確認します。

## 常時禁止するパターン

- `uses: <name>@vN` のような tag／branch 参照、明示しない `permissions:`、未設定の job timeout、secret の `echo`／`set -x` を残しません。
- long-lived AWS access key を secrets に保存せず、AWS 認証は OIDC とします。OIDC の IAM Role／Provider は Terraform code で管理せず、管理元の記録と変更手順は `terraform` skill と repository の既存方針に従います。
- task runner があるのに `ruff`、`pytest`、`eslint`、`prettier`、`tsc` などを workflow から直接呼び出しません。Terraform の専用 gate は例外として [Terraform CI](references/terraform-ci.md) に従います。
- `pull_request_target` で untrusted な PR head を実行せず、Environment の境界を無視した環境別 secret／vars の平置き、権限の広い shared job、失敗を隠す `continue-on-error` を採用しません。
- 本番運用中の workflow を無停止で書き換えず、一時的な `if:` で無効化してから変更を反映します。
- 環境判定を各 step の `if: github.ref == 'refs/heads/main'` に散らさず、job の `environment:` と明示した対応表で一括して扱います。
- AWS 認証など複数 workflow／job で使う処理を各 workflow に直接並べず、`.github/actions/<name>/` へ集約します。

## 完了条件

- [ ] 対象 event、ref、actor、checkout 対象、信頼境界、secret／OIDC の可用範囲を確認した
- [ ] `permissions:` を workflow と必要な job に明示し、最小権限にした
- [ ] 外部 `uses:` をすべて commit SHA で pin し、version コメントを併記した
- [ ] job の `needs`、step の順序、outputs、`if:`、失敗時の扱い、`timeout-minutes`、`concurrency` を意図どおりに定義した
- [ ] secret、credential、個人情報がログ、artifact、cache、output に露出しないことを確認した
- [ ] untrusted code と privileged な job を分離し、shell injection と `pull_request_target` の危険な checkout を排除した
- [ ] `workflow_dispatch` を原則使用せず、使用した場合は理由、実行 ref、input、Environment の allowlist、trust boundary、承認条件を確認した
- [ ] Environment、branch／tag、OIDC、deploy の承認と対象を明示した（該当する場合）
- [ ] lint／format／test／build は task runner 経由で実行し、Terraform の場合は専用 gate を確認した
- [ ] Terraform plan の PR comment に tfcmt を使い、binary／token／permissions を source of truth に合わせ、write 権限を untrusted code に広げていない（該当する場合）
- [ ] reusable workflow／composite action の input、secret、output、permissions、失敗時契約を確認した（該当する場合）
- [ ] actionlint、関連するローカル検証、GitHub CI、deploy の証拠を、実施範囲と未確認範囲を分けて報告した

## 詳細資料

- [Workflow structure](references/workflow-structure.md) — workflow の配置、命名、trigger、job／step graph、実行制御、matrix、cache を設計・変更するときに読む。
- [Permissions and secrets](references/permissions-and-secrets.md) — `permissions`、secret、vars、Environment、fork／PR からの secret 境界を設計・レビューするときに読む。
- [Actions and supply chain](references/actions-and-supply-chain.md) — action の SHA pin、第三者 action、untrusted input、script injection、`pull_request_target` をレビューするときに読む。
- [Environments and AWS](references/environments-and-aws.md) — GitHub Environment、複数環境の切替、AWS OIDC の受け渡しを設計・レビューするときに読む。
- [Reuse](references/reuse.md) — reusable workflow と composite action の切り出し、caller／callee 契約、共通処理の集約を行うときに読む。
- [Terraform CI](references/terraform-ci.md) — Terraform の fmt、validate、TFLint、plan、apply を GitHub Actions に組み込むときに読む。
- [Verification](references/verification.md) — actionlint、task runner、GitHub 側の run 確認、DoD、証拠の報告を行うときに読む。

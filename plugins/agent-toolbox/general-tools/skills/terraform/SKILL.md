---
name: terraform
description: AWS Terraform の設定を追加・変更・レビューし、対象環境の plan と安全性を確認するときに使う。
---

# Terraform

Terraform で AWS インフラを扱うときは、対象 module、state、provider、account、region、既存の構成・命名・セキュリティ方針を確認してから変更します。Terraform のコード、plan、ログへ credential や secret を書き込みません。

## 適用条件

- AWS Terraform の設定を新規作成、変更、削除、移行、レビューするときに使います。
- 対象 root と環境が明確でない場合は、repository の構成と依頼範囲から対象を特定してから操作します。
- Terraform 以外の AWS CLI 操作だけを行う場合は、この skill の Terraform 固有手順を適用しません。

## 基本方針

- Terraform 1.12 以上と AWS Provider の最新安定版を、対象 repository が定める方法で使用します。
- 対象 root で `terraform fmt`、`terraform validate`、TFLint を実行し、結果を報告します。
- Terraform の設定変更では `terraform test` と `.tftest.hcl`／`.tftest.json` を追加・実行せず、format、validate、lint、plan で検証します。
- `terraform-docs` による module README の生成は CI に委ねます。ユーザーから明示的な実行依頼がない限りローカル実行せず、生成領域の手作業による作成・補完・再現・更新を禁止します。実行依頼があっても生成ツールを使えない場合は手書きで代替せず、README を変更しないまま未実行として報告します。
- provider、account、region、profile、backend、workspace の境界を暗黙にせず、変更前に確認します。
- 既存の task runner、CI、module 構成、命名、セキュリティ方針がある場合は、それらを優先します。

## 作業の進め方

1. 対象 root、環境、module、backend、state、AWS account、region、profile を確認します。
2. 対象ディレクトリへ移動してから Terraform コマンドを実行し、`terraform plan` を変更の根拠にします。
3. 変更を最小の責務単位で実装し、resource 差分、state address、権限、ネットワーク、暗号化への影響を確認します。
4. 必要な format、validate、lint、plan を対象範囲で実行し、実行した範囲と未実行の範囲を分けて記録します。
5. `apply` または `destroy` が依頼に含まれる場合だけ、対象環境と承認済みの実行経路を確認してから実行します。

## 常時適用する安全境界

- `terraform -chdir=<dir> ...` は使わず、対象ディレクトリへ移動して実行します。
- 本番の `apply`、`destroy`、state 操作、権限変更を手動で行わず、承認済みの CI/CD 経路に限定します。
- `terraform import`、`terraform state mv`、`terraform state rm` は手動で実行せず、`import`、`moved`、`removed` block を PR に含めて plan で確認します。
- state を変更する `import`、`moved`、`removed` block は resource 定義へ混在させず、root module ごとに `imports.tf`、`moved.tf`、`removed.tf` という block 種類別の専用ファイルへ記述します。repository 固有の既存命名がある場合はそれを優先します。
- `import`、`moved`、`removed` block は環境 root module にのみ記述し、再利用 child module 内では禁止します。移行対象が child module 内の resource でも、block は環境 root module の専用ファイルに記述します。
- `for_each` は、同じ構成と責務の resource または module を map の key／set の member ごとに1つ作成する場合だけ許可し、入力要素数と作成 instance 数を一致させます。
- `depends_on` は原則禁止とし、resource の属性参照と module の input／output で依存関係を表現します。参照では表せない依存がある場合だけ、[構成とコーディング規約](references/conventions.md#依存関係)の例外条件に従います。
- `terraform taint` は使わず、置換が必要な場合は `terraform apply -replace=<address>` の影響を確認します。
- `-target` は障害復旧などの限定用途にとどめ、理由と対象を記録します。
- local state、認証情報、secret、個人情報を repository、plan、ログ、生成物へ残しません。
- 破壊、置換、権限変更、コスト増加がある場合は、対象、影響、復旧手段を plan と PR に記載します。

## 完了条件

- 対象 root の変更が意図した resource 差分と state 影響に収まっている。
- 対象 repository が定める format、validate、lint、plan がエラーなしで完了している。
- `terraform test` と `.tftest.hcl`／`.tftest.json` を Terraform 専用の完了条件に含めず、新規作成・実行していない。
- AWS の認証、IAM、ネットワーク、暗号化、秘密値に関する確認結果を報告している。
- 実行できなかった検証や外部環境の未確認事項を、成功扱いにせず理由付きで報告している。
- `apply` または `destroy` を行った場合は、対象環境と承認済みの経路を記録している。

## 詳細資料

- [構成とコーディング規約](references/conventions.md) — Terraform の構成、命名、HCL の記法、ファイル分割を扱うときに読む。
- [module と state](references/modules-and-state.md) — module の入出力、backend、state、resource address の移行を扱うときに読む。
- [AWS セキュリティ](references/aws-security.md) — AWS の IAM、ネットワーク、暗号化、秘密値、OIDC 境界を扱うときに読む。
- [検証](references/verification.md) — format、validate、lint、plan、DoD、結果報告を扱うときに読む。

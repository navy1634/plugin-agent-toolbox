# 機能 module と state

## module の境界

- 1 module は、利用者に提供する一つの機能または機能群を単位とし、その機能を実現する resource をまとめます。AWS サービスや resource 種別だけを単位に module を作りません。
- 機能 module は、常に一緒に管理する resource と、明確な ownership・lifecycle・権限境界をまとめます。
- 環境 root は機能 module を組み合わせて環境固有値と依存関係を管理し、resource の実装詳細を抱え込みません。
- child module が submodule を呼ぶ構成も選択できますが、入力・出力・依存関係を文書化し、原則として浅い階層を保ちます。深い nesting や循環的な責務分割は、再利用性や可読性への具体的な利点がある場合だけ採用します。
- output は利用側が必要とする値だけを公開し、未参照 output、デバッグ用 output、オブジェクト全体の output は作りません。

## variable と output の契約

- すべての variable に日本語の `description` と `type` を付け、入力制約を `validation` に記述します。
- `default` は環境に依存しない値だけに設定し、環境差分を default で隠しません。
- output の `description` には参照元または利用目的を記載し、機密値は `sensitive = true` にします。

```hcl
variable "vpc_cidr" {
  description = "VPC に割り当てる CIDR"
  type        = string

  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "vpc_cidr は有効な CIDR で指定してください"
  }
}

output "vpc_id" {
  description = "環境 root が参照する VPC ID"
  value       = aws_vpc.this.id
}
```

## state と backend

- state は S3 の remote backend を使い、`{component}/terraform.tfstate` の粒度で環境と component を分離します。
- backend は暗号化（`encrypt = true`）し、state の同時実行防止が必要な場合は repository が採用する方式を明示します。
- `terraform_remote_state` は使用禁止とし、別 state の resource ID を HCL に hard-code しません。
- 同じ root module の機能間は module output を直接渡し、別 root module との依存は明示的な input variable または AWS provider の data source で解決します。値の所有者、更新手順、受け渡し方法を記録します。

```hcl
variable "vpc_id" {
  description = "user-registration 機能が接続する VPC ID"
  type        = string
}

module "user_registration" {
  source = "../../modules/user-registration"
  vpc_id = var.vpc_id
}
```

## resource address の移行

resource address を変更する場合は、手動の state 操作ではなく `imports.tf` と `moved.tf` に HCL block を記述し、PR で plan の差分を確認します。

```hcl
import {
  to = aws_s3_bucket.logs
  id = "example-logs"
}

moved {
  from = aws_s3_bucket.old_logs
  to   = aws_s3_bucket.logs
}
```

移行後に不要となった block は apply の結果を確認してから削除し、削除理由と state address の変化をコミットおよび PR に残します。

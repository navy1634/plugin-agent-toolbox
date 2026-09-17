# 構成とコーディング規約

## バージョンとツール設定

- Terraform 1.12 以上、AWS Provider は最新安定版を使用します。
- Terraform のプロジェクトルートに `.tflint.hcl` と `.terraform-docs.yml` を置きます。
- TFLint の `terraform_required_version`、`terraform_required_providers`、`terraform_standard_module_structure` は無効化しません。
- module の `README.md` は `terraform-docs` で生成し、手書きの内容を混在させません。

## ディレクトリとファイル

次は、環境ごとに root module と state を分け、同じ repository 内に再利用する child module を置く場合の構成例です。唯一の標準として固定せず、既存 repository の ownership、lifecycle、state 境界、配布方法を基準に選び、構成を変更する場合は移行理由を記録します。

```
terraform/
├── .tflint.hcl
├── .terraform-docs.yml
├── env/
│   └── {dev,stg,prd}/
│       ├── backend.tf
│       ├── providers.tf
│       ├── versions.tf
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── modules/
    └── {user-registration,scheduled-export}/
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        └── README.md
```

再利用 module を別 repository または module registry で配布する場合は、環境 root から version を固定した module source を参照し、この repository に `modules/` を持たせません。

module のディレクトリ名は提供する機能を表し、`vpc`、`s3`、`iam` などの AWS サービス名や resource 種別を主たる module の単位にしません。

Terraform は同じ module のトップレベルにあるすべての `.tf` ファイルを評価するため、ファイル名は読み手のための整理です。`main.tf`、`variables.tf`、`outputs.tf`、`locals.tf`、`data.tf`、`providers.tf`、`backend.tf`、`versions.tf` などを既存 repository の責務に合わせて使い、行数だけを理由に分割せず、責務・変更頻度・所有単位が分かれるときに分割します。

## 命名とタグ

| 環境ディレクトリ | 短縮名 | `var.environment` |
| --- | --- | --- |
| `env/dev/` | `dev` | `development` |
| `env/stg/` | `stg` | `staging` |
| `env/prd/` | `prod` | `production` |

- resource、variable、local、output の識別子は snake_case にします。
- AWS の Name tag は `{project}-{purpose}-{env}`、output 名は `{resource_type}_{purpose}_{attribute}` の形式にします。
- 共通タグは provider の `default_tags` で付与し、resource 側には Name tag だけを指定します。
- 共通タグを `merge()` で resource ごとに再構成しません。

## HCL の記述順序

- `terraform fmt -recursive` を適用し、等号の位置は formatter に任せます。
- resource の属性は、必須引数、任意引数、ネストした block、meta-argument、`tags` の順に置きます。
- `for_each` と `count` は原則として使いません。`for_each` は subnet のように、同一の resource 構成と責務を複数の入力要素へ繰り返し適用する場合だけ許可します。
- `count` は使わず、環境ごとの resource の有無や個数を `count` や条件式で切り替えません。要件上どうしても環境差が必要な場合は、機能 module の構成または明示的な設定として表現し、理由を記録します。
- `format("%s-…")` のような遅延的な文字列組み立ては使わず、構造が読み手に明示される文字列補間を使います。
- security group の ingress と egress は `aws_security_group_rule` で個別に定義し、resource 内の inline rule を使いません。

次のように、各 subnet で CIDR や Availability Zone が異なっていても、resource の構成と責務を揃えて生成対象のキーを明示します。

```hcl
resource "aws_subnet" "private" {
  for_each          = var.private_subnets
  vpc_id            = var.vpc_id
  cidr_block        = each.value.cidr_block
  availability_zone = each.value.availability_zone

  tags = {
    Name = "${var.project}-private-${each.key}-${var.environment}"
  }
}
```

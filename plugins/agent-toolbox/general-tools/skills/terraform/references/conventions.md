# 構成とコーディング規約

## バージョンとツール設定

- Terraform 1.12 以上、AWS Provider は最新安定版を使用します。
- Terraform のプロジェクトルートに `.tflint.hcl` と `.terraform-docs.yml` を置きます。
- TFLint の `terraform_required_version`、`terraform_required_providers`、`terraform_standard_module_structure` は無効化しません。
- module の `README.md` は `terraform-docs` で生成し、手書きの内容を混在させません。
- `terraform-docs` の出力更新は CI に委ねます。ユーザーから明示的な実行依頼がない限りローカル実行せず、設定や既存 README から出力を推測して表を手書きしたり、生成領域を補完・再現したりしてはいけません。実行依頼があっても生成ツールを使えない場合は手書きで代替せず、README を変更しないまま未実行の理由を報告します。

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

## 環境ごとの差分

- 環境 root、state、provider、account の分離は実行境界であり、それだけでは機能の環境差分を設ける理由になりません。
- 共有 child module は全 call site で同じ実装を使います。module が prd からも呼ばれていることは影響範囲の確認事項であり、変更を dev 限定にする理由ではありません。
- 依頼または承認済み設計で環境限定の振る舞いが明示されている場合だけ、その差分を表現します。現在の root、profile、実行ロール、過去の別依頼から対象環境を推測しません。
- 環境名による条件式、特定環境だけを有効化する flag、同じ機能を持つ module の複製を、環境差分を作る目的で追加しません。環境ごとの値が必要な場合も、root の構成または既存の module 契約で表現します。
- 共有 module の全 call site、現行の ADR・契約、対象 root の plan を確認し、変更が及ぶ環境を明らかにします。明示された環境限定要件を既存契約で満たせない場合は、独自の分岐を作らず、影響と必要な設計判断を示します。

## 依存関係

`depends_on` は原則禁止とします。resource の属性を参照し、module 間では必要な output を input に直接渡して、Terraform が依存関係を推論できる構成にします。参照で表現済みの依存を重ねて指定したり、module の責務分割や循環依存の問題を実行順序の指定で回避したりしてはいけません。

例外は、他の resource の処理完了に依存するものの、その属性値を必要とせず、参照では依存関係を表せない場合に限ります。使用箇所の日本語コメントに、必要な依存関係と参照では解決できない理由を記載し、対象を必要最小限に絞ります。module 全体への依存は影響範囲が広いため避け、例外として使う場合も `terraform plan` で不要な変更・置換や未確定値の増加がないか確認します。

## HCL の記述順序

- `terraform fmt -recursive` を適用し、等号の位置は formatter に任せます。
- resource の属性は、必須引数、任意引数、ネストした block、meta-argument、`tags` の順に置きます。
- `for_each` と `count` は原則として使いません。`for_each` は subnet のように、同一の resource または module の構成と責務を map の各 key または set の各 member へ1対1で適用し、その要素数と作成 instance 数を一致させる場合だけ許可します。resource の有無の切り替え、異なる resource の代替、入力の一部だけを作成する条件分岐には使いません。
- `count` は使わず、環境ごとの resource の有無や個数を `count` や条件式で切り替えません。環境差分を追加する条件と表現方法は[環境ごとの差分](#環境ごとの差分)に従います。
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

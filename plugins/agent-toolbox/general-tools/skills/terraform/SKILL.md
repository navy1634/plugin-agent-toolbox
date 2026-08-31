---
name: terraform
description: Terraform (AWS) 開発規約。ディレクトリ構成、命名規則、モジュール設計、セキュリティチェック、禁止パターン、DoD を含む包括的な IaC 開発ガイドライン。
---

# Terraform 開発規約 (AWS)

## 基本方針

- Terraform 1.12+、AWS Provider 最新安定版
- `terraform fmt` + `terraform validate` + tflint + trivy 必須
- AWS プロファイル: `--profile` 必須、`providers.tf` で明示

## ツール設定ファイル

プロジェクトルート（`terraform/`）に `.tflint.hcl` と `.terraform-docs.yml` を必ず配置。無効化ルール: `terraform_required_version`, `terraform_required_providers`, `terraform_standard_module_structure`

## ディレクトリ構成

```
terraform/
├── .tflint.hcl / .terraform-docs.yml
├── env/{dev,stg,prd}/           # main.tf, variables.tf, outputs.tf, backend.tf, providers.tf
└── modules/{networking,compute,database,iam,route53,monitoring}/
    ├── main.tf, variables.tf, outputs.tf, README.md(自動生成)
```

ファイル分割: `main.tf`, `variables.tf`, `outputs.tf`, `locals.tf`, `data.tf`, `providers.tf`, `backend.tf`, `versions.tf`。300行目安で分割。

## 命名規則

**env / environment の対応:**

| フォルダ | `env`(短縮) | `var.environment`(正式) |
|---------|------------|----------------------|
| `env/dev/` | `dev` | `development` |
| `env/stg/` | `stg` | `staging` |
| `env/prd/` | `prod` | `production` |

- リソース名・変数名: **スネークケース**（NG: キャメルケース、曖昧な名前）
- Name タグ形式: `{project}-{purpose}-{env}`
- 出力名形式: `{リソース種別}_{用途}_{属性}`
- 共通タグ: provider `default_tags` で付与（`merge()` 禁止）
- リソース側は Name タグのみ指定

## モジュール設計

- 1モジュール = 1論理インフラ単位。module in module 禁止
- 全変数に `description`(日本語) + `type` 必須。`validation` で入力制約
- `default` は環境非依存の場合のみ
- README.md は `terraform-docs` 自動生成（手書き禁止）
- output: 必要最小限のみ。`description` に参照元を明記。機密情報は `sensitive = true`
- デバッグ output / オブジェクト全体の output / 未参照 output は禁止

## 状態管理

- S3 バックエンド必須: `{component}/terraform.tfstate`
- 暗号化必須（`encrypt = true`）、DynamoDB ロック必須
- ローカル state 禁止。環境 x コンポーネントで分離
- `terraform_remote_state` で他 state 参照（ID ハードコード禁止）

## OIDC IAM Resources

- IAM resources for GitHub Actions OIDC integration (IAM OIDC Provider, IAM Role, Trust Policy, etc.) **do not exist as Terraform code locally or in any GitHub repository**
- These are assumed to be created and managed via the AWS Console or other external means
- Do not create or modify OIDC IAM resources via Terraform

## セキュリティチェック

- [ ] S3 パブリックアクセスブロック有効
- [ ] SG `0.0.0.0/0` ingress は ALB/NLB のみ
- [ ] RDS/ElastiCache はプライベートサブネット
- [ ] 暗号化有効（S3, RDS, EBS, EFS）
- [ ] IAM 最小権限（ワイルドカード `*` 禁止）
- [ ] シークレットは SSM Parameter / Secrets Manager
- [ ] VPC フローログ・CloudTrail 有効
- [ ] 認証情報の HCL ハードコード絶対禁止

## 変数値管理

- `terraform.tfvars` / `*.auto.tfvars` 使用禁止 → Secrets Manager で管理
- パス形式: `/{project}/{name}-{env}`、JSON 形式で格納
- `data "aws_secretsmanager_secret_version"` + `jsondecode` で取得
- `-var` による変数指定も基本不使用

## コーディングスタイル

- `terraform fmt -recursive` 常時適用。`=` アライメント揃える
- リソース定義順序: 必須引数 → オプション → ネストブロック → メタ引数 → tags
- `for_each` 優先（`count` は boolean 0/1 切替のみ）
- SG ルールは `aws_security_group_rule` で個別定義（インライン禁止）

## コマンド実行

`terraform -chdir=<dir> <command>` および `terraform -chdir <dir> <command>` は禁止する。対象ディレクトリへ移動してから terraform を実行する。

`-chdir` はサブコマンドより前に置くグローバルオプションなので、コマンド文字列が `terraform -chdir=...` で始まり、`terraform apply` や `terraform destroy` を対象にした権限拒否設定をすり抜ける。どのディレクトリに対して実行したのかもコマンドを読むだけでは追いにくく、環境を取り違えたまま apply が通ってしまう。危険な操作を止める仕組みが働かなくなるため、記法にかかわらず使わない。

## import / moved

- `terraform import` / `terraform state mv` コマンド禁止 → HCL ブロック使用
- `imports.tf` / `moved.tf` にまとめて記述。apply 後に削除してコミット
- 必ず PR レビュー経由。plan で差分確認必須

## 禁止パターン

| パターン | 代替 |
|---------|------|
| `terraform apply` 手動実行(prd) | CI/CD 経由 |
| `terraform state` 手動操作 | ユーザー承認必須 |
| `terraform taint` | `terraform apply -replace` |
| `terraform import` コマンド | `import` ブロック |
| `terraform state mv` コマンド | `moved` ブロック |
| `terraform -chdir=<dir>` | 対象ディレクトリへ移動してから実行 |
| ローカル state | S3 必須 |
| ID ハードコード | data source / remote state |
| `merge()` タグ結合 | `default_tags` |
| `terraform.tfvars` | Secrets Manager |
| インライン ingress/egress | `aws_security_group_rule` |

## Definition of Done

### コード品質ゲート（全 PASS 必須）

```bash
cd terraform/ && terraform fmt -recursive && tflint --config .tflint.hcl --recursive
cd env/{dev,stg,prd}/ && terraform validate && terraform plan
```

### チェックリスト

- [ ] 上記コマンド全てエラーゼロ
- [ ] Plan: 意図しない削除・再作成なし、SG/IAM 変更目視確認、コスト確認
- [ ] 命名: スネークケース、description(日本語)+type 全定義、Name タグ形式準拠
- [ ] セキュリティ: 上記セキュリティチェック全項目
- [ ] State: S3 バックエンド設定済、適切な粒度で分離
- [ ] モジュール: validation 設定、後方互換性維持

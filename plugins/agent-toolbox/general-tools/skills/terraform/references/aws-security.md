# AWS セキュリティ

## provider とアカウント境界

- provider の account、region、profile を暗黙にせず、`providers.tf` で対象を明示します。
- AWS CLI を実行するときは対象環境の `--profile` を明示し、実行前に account ID と region を確認します。
- 本番 account への `apply`、`destroy`、権限変更は、対象環境の承認済み CI/CD 経路に限定します。

## ネットワーク、暗号化、ログ

- S3 bucket は public access block を有効にし、S3、RDS、EBS、EFS の保存データを暗号化します。
- RDS と ElastiCache は private subnet に配置し、security group の `0.0.0.0/0` ingress は公開ロードバランサーなど必要な入口だけに限定します。
- IAM policy は最小権限とし、不要な wildcard `*` を許可しません。
- VPC flow logs と CloudTrail を有効にし、監査対象の account と region を取り違えないようにします。

## OIDC の管理境界

GitHub Actions の OIDC 連携に使う IAM OIDC Provider、IAM Role、trust policy などは、ローカルまたは GitHub repository の Terraform code として管理しません。AWS Console などの外部手段で作成・管理されている場合は、その管理元を記録し、Terraform で重複作成または変更しません。

## 秘密値

- 認証情報や秘密値を HCL、`terraform.tfvars`、`*.auto.tfvars`、plan の共有先、ログへ書き込みません。
- Secrets Manager または SSM Parameter を data source から参照し、state に保存される値の機密性も確認します。
- パラメータの標準パスは `/{project}/{name}-{env}` とし、複数値は JSON で格納します。

```hcl
data "aws_secretsmanager_secret_version" "database" {
  secret_id = "/example/database-prod"
}

locals {
  database_secret = jsondecode(
    data.aws_secretsmanager_secret_version.database.secret_string
  )
}
```

`-var` に秘密値を渡す運用は基本的に採用せず、既存 repository が例外を定めている場合だけ、その理由と露出範囲を確認します。

## 確認項目

- [ ] S3 public access block が有効である
- [ ] 公開 ingress が必要な入口に限定されている
- [ ] RDS と ElastiCache が private subnet にある
- [ ] S3、RDS、EBS、EFS の暗号化が有効である
- [ ] IAM が最小権限で wildcard を避けている
- [ ] VPC flow logs と CloudTrail が有効である
- [ ] 認証情報が HCL、tfvars、ログに露出していない

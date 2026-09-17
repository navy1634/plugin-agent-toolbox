# 検証

## 実行順序

変更対象の Terraform root へ移動して、次の検証を実行します。`terraform -chdir` は使わず、CI とローカルで同じ task runner または同じコマンドを使います。

```bash
cd terraform/
terraform fmt -recursive -check
tflint --config .tflint.hcl --recursive

cd env/dev/
terraform validate
terraform plan
```

複数環境を変更する場合は、対象環境ごとに `terraform validate` と `terraform plan` を実行します。backend や provider の初期化が必要な repository では、既存の初期化手順を先に実行し、認証情報や state を出力へ漏らしません。

## plan の確認

plan の結果について、次の項目を確認します。

- 意図しない create、update、destroy、replace がない
- IAM と security group の変更が要件どおりである
- 暗号化、public access、network boundary が維持されている
- state address の変更と resource 移行が想定どおりである
- module の input validation と後方互換性が保たれている
- backend の対象、暗号化、state path が正しい
- secret、credential、個人情報が plan、ログ、成果物へ露出していない
- 変更によるコストと復旧手段を把握している

## DoD と報告

- [ ] 対象 root の format、validate、lint がエラーなしで完了している
- [ ] 対象環境の plan を確認し、意図しない差分がない
- [ ] AWS セキュリティ確認項目をすべて確認している
- [ ] 実行した root、未実行の root、実行結果を分けて報告している
- [ ] apply または destroy を行った場合は、対象環境と承認済み経路を明記している

検証できない環境や外部 CI の結果は成功とみなさず、未確認の理由と必要な次の確認を報告します。

---
name: aws-cli
description: AWS CLI 実行ルール。プロファイル管理（<profile>-dev/<profile>-prod）、安全なコマンド実行、禁止操作の定義。
---

# AWS CLI Rules

## AWS Profiles

| Environment | Profile | Purpose |
|-------------|---------|---------|
| Development | `<profile>-dev` | AWS resource operations in dev |
| Production | `<profile>-prod` | AWS resource operations in prod |

## Command Execution Rules

All AWS CLI commands must explicitly specify `--profile`.

```bash
# Development
aws s3 ls --profile <profile>-dev
aws ecs list-services --cluster my-cluster --profile <profile>-dev

# Production
aws s3 ls --profile <profile>-prod
aws rds describe-db-instances --profile <profile>-prod
```

- Executing without `--profile` is prohibited (prevents fallback to default profile)
- Setting `AWS_PROFILE` environment variable per session is also acceptable

```bash
export AWS_PROFILE=<profile>-dev
```

## Prohibited Actions

- AWS CLI execution without `--profile` (prevents unintended environment targeting)
- Destructive operations (delete, update) with production profile (`<profile>-prod`) require user confirmation

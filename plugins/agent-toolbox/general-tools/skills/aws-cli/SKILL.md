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

## Expired Credentials

When credentials expire during a requested verification:

1. Identify the credential provider used by the AWS CLI invocation.
2. Refresh its credentials yourself.
3. Retry the original verification.

For an expired IAM Identity Center session, run:

```bash
aws sso login --sso-session <session-name>
```

Do not hand reauthentication to the user.
If the identity provider requires account-owner action, explain the exact action.
Resume the login process after that action.

Distinguish authentication failure from IAM permission failure.
If authentication cannot be completed, report live permissions as unconfirmed.

## Prohibited Actions

- AWS CLI execution without `--profile` (prevents unintended environment targeting)
- Destructive operations (delete, update) with production profile (`<profile>-prod`) require user confirmation

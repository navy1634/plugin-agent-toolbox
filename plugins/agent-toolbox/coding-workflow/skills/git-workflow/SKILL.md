---
name: git-workflow
description: ブランチ、コミット、Pull Request、Issue、または差分を変える Git 操作を行うときに使う。
---

# Git Workflow

Git で変更を加える前に、対象 repository の branch、差分、履歴、remote 状態を確認し、既存の未コミット作業を保護する。変更の目的と対象を一つに絞り、無関係な整形や生成物を混ぜない。

## 適用条件

branch の作成・切替、差分の確認、staging、commit、Issue、Pull Request、remote の確認、履歴や作業ツリーに影響する操作を行うときに使う。Git の状態を読むだけの調査でも、既存差分を誤って扱わないために安全境界を適用する。

## 基本フロー

1. git status --porcelain=v2、staged／unstaged の diff、直近の log、現在 branch、必要な remote 情報を確認する
2. ユーザーや他担当者の差分、対象範囲、変更してよいファイル、完了条件を特定する
3. 変更を目的ごとに実施し、既存の命名・履歴・repository の template に合わせる
4. diff、status、対象テストや lint、commit／Issue／PR の内容を読み、意図しない変更がないことを確認する
5. ローカル、remote、CI、deploy の証拠を分けて報告する

git add、git commit、git push はユーザーの責任であり、現在の依頼に操作と対象が明記された場合だけ実行する。禁止コマンドは、後述の一覧を毎回適用し、reference を追加で読まなければならない状態にしない。

## 常時適用する禁止事項

force push（--force、--force-with-lease）、reset、checkout、restore、clean、stash、rebase、cherry-pick、revert、branch -D、am、apply、commit --amend、tag 操作は、履歴や作業ツリーを変更するため無条件に禁止する。merge は、現在の依頼に対象が明記され、影響と復旧方法を確認できる場合だけ扱う。ブランチ削除と push は、現在の依頼に操作と対象が明記されている場合だけ行う。

git add -A や git add . は使わず、対象ファイルを明示する。.env、credential、秘密情報、バイナリを stage しない。--no-verify や --no-gpg-sign で hook や署名を無効化しない。コマンドは対象ディレクトリへ移動してから一つずつ実行し、完了報告ではローカル結果と外部状態を分ける。

## References

- [Issue](references/issue.md) — Issue の作成、本文、template、検証を扱うときに読む。
- [Pull Requests](references/pull-requests.md) — PR の作成、本文、template、検証を扱うときに読む。
- [Development](references/development.md) — branch、add、commit、push、レビュー前後の開発フローを扱うときに読む。

## 完了条件

意図した差分、branch、履歴、remote 状態を再確認し、不要な変更、秘密情報、生成物を残さない。必要な検証が完了し、未確認の CI／remote／deploy 状態を明示し、ユーザーが次に行う操作を分けて報告する。

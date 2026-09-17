# 計画ファイルの形式

## 保存先

共有 rules が標準の計画保存先を定めている環境では、次の場所を使います。

```text
~/.agents/plan/<repository-slug>/<task-slug>/plan.md
```

`<repository-slug>` は `git rev-parse --path-format=absolute --git-common-dir` が返す絶対パスの末尾名から `.git` を除いて作ります。worktree のディレクトリ名は repository の識別子に使いません。
`<task-slug>` は作業開始日を先頭にした `YYYYMMDD-...` 形式にします。

## 標準構成

対象に該当しない節は理由なく空欄にせず、不要である根拠を本文へ記載するか、節自体を省きます。

```markdown
# Plan: [Title]

## Approval
- [ ] Reviewed and approved by user

## Overview
[目的、対象範囲、期待する結果]

## Requirements and Acceptance Criteria
[確定済みの要件、制約、受入条件]

## Current State and Evidence
[確認したファイル、設定、テスト、既存パターン、未コミット差分]

## Design Decisions
- [Decision title](./ADR-001-decision-title.md)

## Shared Contract
[公開名、path、型、入出力、error、外部境界、書き込み範囲]

## Dependencies and Execution Order
[手順や担当境界の依存関係、順序、並行実行条件]

## Steps
1. [Step title]
   - Target: [対象ファイルまたは構成要素]
   - Change: [具体的な変更]
   - Reason: [必要な理由]
   - Depends on: [前提となる手順]
   - Verification: [確認方法]
   - Reversibility: [戻し方または不要な理由]

## Test and Static Verification Strategy
[受入条件と unit、integration、E2E、静的確認の対応]

## Risks and Reversibility
[発生条件、影響、緩和策、検知方法、rollback]

## Unresolved Items or Review Conditions
[担当、必要な証拠、期限、実装停止条件]

## Success Criteria
- [ ] [観測可能で判定可能な条件]
```

ADR を作成した判断は、plan の `Design Decisions` に判断ごとの ADR へのリンクだけを記載し、選択肢、不採用理由、決定、根拠、影響を重複して再掲しません。

振る舞いを変更しない計画では `Shared Contract` のうち不要な項目を省けます。複数の担当境界がない場合でも、誰がどの範囲を変更するかは `Steps` から判定できるようにします。

## Approval

新規計画では `Approval` を `[ ]` のまま作成します。計画を作成または更新する側が `[x]` に変更したり、依頼者の明示的な操作を自己申告で代替したりしません。「実装して」「再開して」などの実装指示は、計画承認を意味しません。

## 既存計画の更新

更新前に `plan.md` の現状と差分を読み、依頼者の編集と `Approval` を含む未変更箇所を保持します。修正対象だけを patch し、増分更新のためにファイル全体を再生成したり、一時ファイルから上書きしたりしません。計画の全面書き直しは、依頼者が明示した場合に限ります。

設計変更を反映した場合は、影響する共有契約、依存順、手順、検証戦略、リスク、成功条件を同じ更新で整合させます。

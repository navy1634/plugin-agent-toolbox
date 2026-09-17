# Issue 作成

## template の選択と定義

既存の .github/ISSUE_TEMPLATE/ に bug、feature、question などの template があれば、事象に合うものを選び、欄の順序と必須条件をそのまま使う。Issue form は Markdown ではなく YAML の input、textarea、dropdown、checkboxes で必須条件を型として表す。

Issue template は .github/ISSUE_TEMPLATE/*.yml に置く。空 Issue を許可しない場合は config.yml の blank_issues_enabled: false と contact_links を使う。タイトルへ詳細や複数の変更を詰め込まず、各欄へ分ける。

```yaml
name: バグ報告
description: 想定挙動と実際の挙動が異なる
labels: [bug]
body:
  - type: textarea
    id: what-happened
    attributes:
      label: 何が起きたか
      description: 想定挙動と実際の挙動を書く
    validations:
      required: true
  - type: textarea
    id: reproduction
    attributes:
      label: 再現手順
      description: 上から順に実行できる形で書く
    validations:
      required: true
  - type: input
    id: version
    attributes:
      label: バージョン
    validations:
      required: true
```

template を新設・変更した場合は YAML 構文、利用可能な field type、重複しない id、required field、config.yml の設定を確認する。既存 template の欄と必須条件を変更する場合は、その理由と受入条件を Issue または PR に記載する。

## タイトルと本文

タイトルは一行で単一の事象または提案を表す。ファイル名、原因の推測、複数の変更、長いログをタイトルへ詰め込まない。

バグ Issue には少なくとも次を記載する。

- 想定していた挙動と実際に起きた挙動
- 上から順に実行できる最小の再現手順
- version、OS、runtime、関連する依存や設定
- 期待結果、実際の結果、発生頻度、影響範囲
- 必要なログ、stack trace、screenshot（secret、token、個人情報を除く）

機能提案には、解決したい課題、対象利用者、受入条件、代替案、互換性・運用への影響を記載する。質問や調査 Issue では、確認した範囲、未確認事項、次に必要な情報を明記する。

## 作成前後の確認

同じ事象の既存 Issue、関連 PR、version を検索し、重複作成を避ける。作成後は Issue URL、labels、assignee、milestone、関連 PR を確認し、外部 tracker への同期や通知が完了したと推測しない。Issue の作成・更新を外部へ行う操作は、現在の依頼に対象が明記されている場合だけ実行する。

# 開発フロー

## branch 戦略

branch 名は <issue-number>-<short-description> または <prefix>-<short-description> とし、小文字・ハイフン区切り・英語で目的を表す。例は 123-user-auth、feature-user-auth、fix-login-timeout です。main／master へ直接 commit せず、短命な feature branch を使う。

通常の流れは main ← feature branch とし、統合前に base branch の最新状態と差分を確認する。競合解消で merge が必要な場合は、対象と影響を確認してから行い、履歴を書き換える rebase は使わない。

## 変更から commit まで

1. root の preflight に従って status、branch、差分、履歴を確認する
2. 目的に対応する branch を用意し、対象ファイルだけを変更する
3. formatter、lint、test、type check など project の検証を実行する
4. git diff と git diff --stat を読み、意図した変更だけであることを確認する
5. git add <file> のように対象を明示して staging し、git diff --cached を確認する
6. commit が依頼対象なら、hook を有効にしたまま一行の commit message で作成する

git add -A や git add . は使わない。未追跡の .env、credential、秘密情報、バイナリ、生成物が混入していないことを確認する。

## commit message

commit message は日本語一行で、変更の目的だけを書く。prefix、scope、ファイル名、理由、検証結果、変更項目の列挙を含めない。例は OAuth2によるログイン機能を追加、レート制限のカウント二重加算を修正 です。一行で表せない場合は変更を分割する。空 commit は作成しない。

pre-commit hook が失敗した場合は原因を修正して新しい commit として再試行し、--no-verify や amend で隠さない。

## push とレビュー

push は現在の依頼に操作と対象が明記されている場合だけ行う。push 前に local branch、remote、upstream、commit、秘密情報、PR 用の差分を確認する。push 後は PR template に従って PR を作成し、CI が完了してから review を依頼する。

レビュー指摘で変更した場合は、差分と対象検証を再確認してから追加 commit を作成する。ローカル、remote、CI、deploy の証拠を混同せず、未確認の状態を明記する。

`main` / `master` などのデフォルトブランチに対して直接pushを行わない。

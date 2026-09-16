# Actions とサプライチェーン

外部 action は workflow の権限で code を実行する依存関係です。取得元、実行される commit、必要な権限、transitive dependency、runner に渡す credential を確認してから採用・更新します。

## commit SHA で pin する

すべての外部 `uses:` は tag や branch ではなく commit SHA で pin し、コメントで元の version を併記します。公式 action（`actions/*`）も例外にしません。`@v4`、`@main`、短縮 SHA、ユーザー入力から組み立てた ref は採用しません。

```yaml
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
- uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
```

repository 内の composite action を `uses: ./.github/actions/<name>` で参照する場合は local path のため外部 SHA はありません。ただし、その action の中で呼び出す外部 action はすべて SHA pin し、caller から渡す input、secret、権限を明示します。

version comment は人間が SHA の対応 version を追跡するために残します。Dependabot または Renovate を使う場合は、更新 PR で SHA、version、release、差分、必要権限の変更を確認してから取り込みます。自動更新を有効にしても、権限や実行 code のレビューを省略しません。

## 第三者 action のレビュー

新しい action または SHA の更新時は、少なくとも次を確認します。

- owner、repository、release page、commit が意図した取得元と version に一致する
- action の `action.yml`、実行 script、runtime 依存、nested action、container image が確認可能である
- 必要な `GITHUB_TOKEN` scope、secret、Environment 値が最小で、不要な credential を runner に渡さない
- PR、fork、手動入力、schedule など各 trigger で action が扱う code と入力が安全である
- self-hosted runner を使う場合、untrusted code と trusted job を共有せず、ephemeral runner または container isolation を用意する
- action の失敗、timeout、再実行、artifact／cache への書き込みが許容範囲に収まる

source と commit を検証できない action、repository の token を過剰に送信する action、script や image が意図せず mutable tag に依存する action は採用しません。

## untrusted input と script injection

workflow input、PR title／body、branch 名、commit message、issue comment、変更ファイル名などは untrusted input です。これらを `run:` の script 本文、shell command、path、action の `uses:`、権限や Environment の決定に直接埋め込みません。

```yaml
# 禁止
- run: echo "${{ github.event.pull_request.title }}"

# 環境変数へ分離し、shell で quote する
- name: PR title を表示
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: |
    printf '%s\n' "$PR_TITLE"
```

環境変数へ分離しても、`eval`、未引用の展開、command substitution、path traversal、`git` の option injection を許してはいけません。Environment、branch、action input の切替は choice、固定値、明示した対応表で allowlist します。

## `pull_request_target` の境界

`pull_request_target` は base branch の workflow と権限で動くため、PR head を checkout して script、build、test、composite action、third-party action を実行する構成は禁止です。次のような組み合わせを作りません。

```yaml
# 禁止。privileged な pull_request_target で PR head を実行する
on: pull_request_target
steps:
  - uses: actions/checkout@<PR head の ref>
  - run: ./scripts/from-the-pull-request.sh
```

PR に反応して comment や label だけを更新する必要がある場合も、base 側の固定 code、入力の allowlist、最小 permissions、secret 非露出を確認します。検査は `pull_request` の read-only job、privileged な更新は trusted な workflow に分ける方を優先します。

## 禁止パターンと代替

| 禁止 | 代替 |
| --- | --- |
| `uses: <name>@vN`（tag／branch 参照） | `uses: <name>@<commit-sha> # vX.Y.Z` |
| untrusted input を `run:` 本文へ直接展開 | `env` へ渡して quote し、allowlist と validation を行う |
| `pull_request_target` で PR head を checkout／実行 | `pull_request` の read-only 検査と trusted な更新 job を分離する |
| third-party action に不要な secret／write token を渡す | action の契約に必要な値だけを job／step に渡す |
| isolation なしの self-hosted runner で untrusted code を実行 | ephemeral runner または container isolation を使う |

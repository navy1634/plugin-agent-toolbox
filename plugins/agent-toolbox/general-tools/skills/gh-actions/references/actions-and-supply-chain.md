# Actions とサプライチェーン

外部 action は workflow の権限で code を実行する依存関係です。取得元、実行される commit、必要な権限、transitive dependency、runner に渡す credential を確認してから採用・更新します。

## gh-actions-lock による依存固定

外部 `uses:` には読みやすい release tag を記述し、検証済み commit は `.github/workflows/actions.lock` で管理します。
SHA を `uses:` に直書きせず、lockfile は手編集しません。
公式 action（`actions/*`）も同じ方法で管理します。

```yaml
- uses: actions/checkout@v4.2.2
- uses: aws-actions/configure-aws-credentials@v4.0.2
```

依存を追加・変更したら repository の root で次を実行し、workflow と lockfile を同じ変更に含めます。

```bash
gh extension install github/gh-actions-lock
gh actions-lock
gh actions-lock --verify
```

extension は通常実行で lockfile にない ref を解決します。
既存依存を意図的に更新する場合は `gh actions-lock --relock` を使い、差分と upstream release を確認します。
CI では `gh actions-lock --verify` を実行します。
公式の [verify-lockfile.yml](https://github.com/github/gh-actions-lock/blob/main/.github/workflows/verify-lockfile.yml) を導入した場合は、その workflow も lockfile に含めるため、追加後に `gh actions-lock` を再実行します。
`--verify-local` は lockfile の coverage だけを調べるため、CI の検証には使いません。

CLI の flag は変更される可能性があるため、迷った場合は [gh-actions-lock README](https://github.com/github/gh-actions-lock#readme) と `gh actions-lock --help` を確認します。

同一 repository 内の composite action を `uses: ./.github/actions/<name>` で参照するだけなら、その path 自体に lock entry はありません。
local composite action が外部 action を呼ぶ場合、extension は workspace path を追跡しないため `local-action` の blocking finding になります。
`gh actions-lock --migrate-local-actions` は同一 repository の path を `$/...` に変換して依存を追跡できるようにします。
`--ignore-local-actions` は warning に下げるだけで依存を lock しません。
未記録の外部依存があれば、実行時に失敗する可能性があります。
外部 reusable workflow を `jobs.<id>.uses` から呼ぶ依存は、呼び出し元の lock 対象外です。
呼び出し先 repository の lockfile と適用範囲を別に確認します。

release tag と lockfile の対応が意図した取得元・version・commit であることを確認します。
Dependabot または Renovate を使う場合も、更新 PR で release、lockfile の変更、差分、必要権限の変更を確認してから取り込みます。
自動更新を有効にしても、権限や実行 code のレビューを省略しません。

## 第三者 action のレビュー

新しい action または SHA の更新時は、少なくとも次を確認します。

- workflow の目的に対して action の機能が過不足なく合い、独自実装より保守負担を減らす
- owner、repository、release page、commit が意図した publisher と version に一致する
- license、release／commit の履歴、issue 対応、security advisory への対応方針を確認できる
- stars、forks、Marketplace の利用指標、公開された利用例など複数の導入実績を確認する
- popularity は利用実績の参考にとどめ、安全性の根拠にはしない
- `action.yml`、実行される script／container、source、release artifact の生成過程を確認できる
- runtime 依存、nested action、container image の依存元を確認できる
- 必要な `GITHUB_TOKEN` scope、secret、Environment 値が最小で、不要な credential を runner に渡さない
- PR、fork、手動入力、schedule など各 trigger で action が扱う code と入力が安全である
- self-hosted runner を使う場合、untrusted code と trusted job を共有せず、ephemeral runner または container isolation を用意する
- action の失敗、timeout、再実行、artifact／cache への書き込みが許容範囲に収まる

lockfile は実行する commit の固定に使いますが、その action が安全である証明にはなりません。
source と commit を検証できない action、保守状況が不明な action、repository の token を過剰に送信する action、script や image が意図せず mutable tag に依存する action は採用しません。

### 独自実装の前に既存 Action を評価する

PR 作成・更新など汎用的な GitHub 操作では、独自の Git 操作や API 呼び出しを書く前に既存 Action を調べます。
たとえば [`peter-evans/create-pull-request`](https://github.com/peter-evans/create-pull-request) は、workflow workspace の変更を commit し、Pull Request を作成・更新する候補です。
これは候補例であり、無条件に承認した一覧ではありません。

候補 Action が要件を満たし、上記の保守・供給網・権限レビューを通る場合は、それを優先します。
要件に合う action がない、または安全性を確認できない場合は、満たせない要件を記録して独自実装を最小限にします。
単純な一操作を置き換えるために不要な Action 依存を追加せず、action の導入コストと自作ロジックの保守コストを比較します。

`peter-evans/create-pull-request` を採用する場合も、公式 README の token、permissions、repository 設定、後続 workflow の trigger に関する条件をその時点で確認します。
現在の [公式 README](https://github.com/peter-evans/create-pull-request#workflow-permissions) は、Actions に Pull Request 作成を許可する Repository 設定を求めています。
`GITHUB_TOKEN` を使う場合は `contents: write` と `pull-requests: write` が必要で、その token で作成した PR は `push` や `pull_request` workflow を起動しません。
この制約が後続 CI の要件や trust boundary と合うことを確かめ、採用時点の README でも再確認します。

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
| `uses:` に commit SHA を直書きする | release tag を `uses:` に記述し、`gh-actions-lock` で lockfile に記録する |
| lock entry がない外部 action | `gh actions-lock` を実行し、`gh actions-lock --verify` を通す |
| untrusted input を `run:` 本文へ直接展開 | `env` へ渡して quote し、allowlist と validation を行う |
| `pull_request_target` で PR head を checkout／実行 | `pull_request` の read-only 検査と trusted な更新 job を分離する |
| third-party action に不要な secret／write token を渡す | action の契約に必要な値だけを job／step に渡す |
| isolation なしの self-hosted runner で untrusted code を実行 | ephemeral runner または container isolation を使う |

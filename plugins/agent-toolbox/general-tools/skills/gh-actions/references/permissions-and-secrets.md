# 権限とシークレット

workflow の token、secret、vars、Environment を、実行 event と job の信頼境界に合わせて設定します。権限を広げることで処理を通すのではなく、必要な API と job を先に特定して最小権限にします。

## `permissions`

workflow の先頭で `permissions` を明示し、GitHub の default 権限に依存しません。最初は `contents: read` とし、書き込みが必要な job だけ job 単位で scope を広げます。

```yaml
permissions:
  contents: read

jobs:
  test:
    permissions:
      contents: read
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - run: uv run task test

  deploy:
    permissions:
      contents: read
      id-token: write
    environment: prd
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
      - uses: ./.github/actions/aws-login
        with:
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
```

よく使う scope は次のように目的と対応付けます。実際に不要な scope は宣言せず、job の用途を変えたときは権限も再評価します。

| scope | 使用条件 | 注意点 |
| --- | --- | --- |
| `contents: read` | checkout、repository の read | CI の基本値にする |
| `contents: write` | release、tag、repository ファイルの更新 | trusted ref と job に限定する |
| `pull-requests: write` | PR comment、label、review の更新 | untrusted な PR の code 実行 job に付与しない |
| `checks: write`／`statuses: write` | status や check の更新 | 書き込み先と token の用途を限定する |
| `packages: read`／`write` | package の取得・公開 | package と repository の権限を分ける |
| `id-token: write` | OIDC token の発行 | OIDC 認証を行う job だけに付与する |
| `security-events: write` | CodeQL 等の結果を upload | 解析 job のみに付与する |

指定した scope が action の内部で必要かを action の仕様と実行ログで確認します。`write-all`、広い `permissions` の一括指定、workflow 全体への `id-token: write` は採用しません。

## `secrets`、`vars`、Environment

機密値は `secrets`、機密でない設定値は `vars` に置きます。環境で値が変わるものは Repository 直下へ `PRD_ROLE_ARN` のように並べず、GitHub Environment の vars／secrets に分離します。

| 値 | 置き場所 | job での参照 |
| --- | --- | --- |
| API token、private key、password | Repository または Environment `secrets` | `${{ secrets.API_TOKEN }}` |
| account、region、role ARN、endpoint など非機密の環境値 | Environment `vars` | `${{ vars.AWS_DEPLOY_ROLE_ARN }}` |
| 環境に依存しない非機密設定 | Repository `vars` または repository の設定 | `${{ vars.SOME_SETTING }}` |

deploy job には `environment:` を宣言し、その Environment の値だけを露出させます。本番 Environment には required reviewer と branch protection を設定し、workflow の YAML だけで承認を迂回できないようにします。Environment の詳細な branch／ref 対応は [Environments and AWS](environments-and-aws.md) を参照します。

## fork と pull request

fork からの `pull_request` は untrusted code として扱います。secret、OIDC token、write 権限を使う必要がある処理を同じ job に置かず、次のように検査と privileged な処理を分離します。

1. `pull_request` で checkout、lint、test だけを実行し、read 権限と必要最小限の cache だけを使う。
2. trusted な base branch の merge 後、または明示した reviewer／Environment approval 後に plan、deploy、comment 更新を実行する。
3. privileged な workflow が PR head の code、script、action を実行しないことを確認する。

`pull_request_target` は base branch の workflow と権限で実行されるため、PR head を checkout して実行する構成は禁止します。採用する特別な理由がある場合でも、base 側の固定 SHA だけを読む構成、入力の allowlist、最小権限、secret 非露出を個別に確認します。

## secret の受け渡しと出力

- 必要な job／step にだけ secret を渡し、workflow、reusable workflow、composite action の境界を越えて暗黙に継承させません。
- `secrets` 全体の JSON 化、`echo $SECRET`、secret を `printf` する処理、`set -x`、debug log、artifact、cache、job output への出力は禁止します。secret を command argument、URL、エラーメッセージへ含めないようにします。
- 外部値をログへ表示する必要がある場合は secret と連結せず、値そのものが機密でないことを確認します。マスクが必要な値は `::add-mask::` を使いますが、マスクだけで不適切な受け渡しを正当化しません。
- `GITHUB_TOKEN` も権限を持つ credential です。必要な job に明示した permissions と同じ境界で扱い、untrusted input の command にそのまま渡しません。

## shell へ渡す入力

PR title、body、branch 名、commit message、issue comment、workflow input などを shell script に埋め込まず、環境変数へ分離して quote します。

```yaml
- name: PR title を表示
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: |
    printf '%s\n' "$PR_TITLE"
```

次のように expression を shell の script 本文へ直接展開する構成は、入力に quote や command substitution が含まれた場合の script injection を許すため禁止します。

```yaml
# 禁止
- run: echo "${{ github.event.pull_request.title }}"
```

allowlist が必要な Environment、branch、action input は、choice、固定値、明示した対応表で検証します。入力を検証できないまま secret や write 権限のある job に渡しません。

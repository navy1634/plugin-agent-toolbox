---
name: write-zenn-qiita-articles
description: 会話内容や作業ログを根拠に、Zenn と Qiita の記事を各サービスの形式で作成・更新し、GitHub 連携による公開準備まで行う。Zenn／Qiita の記事化、技術ブログ化、既存記事の編集、公開前レビューを依頼されたときに使う。markdownlint と textlint の両方を必ず実行し、両方が成功するまで記事作成を完了扱いにしない。
---

# Zenn／Qiita の記事を作成する

会話から確認できる事実、決定事項、手順、検証結果だけを抽出し、Zenn と Qiita の記事へ変換する。記事は、Zenn または Qiita と連携した投稿先リポジトリを特定できた場合だけ、そのリポジトリへ作成する。両方を指定された場合は、共通の事実整理を使いながら各サービスの front matter、配置先、読者に合わせて別々に仕上げる。

## 開始時に確認する

1. 会話で投稿先リポジトリが明示されていれば、そのパスを使う。明示されていなければ `ZENN_QIITA_REPOSITORY` 環境変数を確認する。
2. 上記の指定がない場合は、カレントの Git ルートを候補にできるか確認する。Zenn は `mise.toml` または `package.json` に Zenn CLI の宣言があり、`articles/` または `books/` があることを確認する。Qiita は `qiita.config.json`、`package.json` の Qiita CLI 宣言、または Qiita 公式 Action のいずれかがあることを確認する。
3. 投稿先を特定できない場合は、リポジトリを検索したり新しいディレクトリを作ったりせず、記事作成を中止して必要な投稿先の指定を報告する。サービス側の GitHub 連携先はローカルファイルだけでは断定できないため、候補リポジトリの remote とサービス側設定の確認が必要であることも報告する。
4. 投稿先の Git ルートへ移動してから `git status --short --branch` と `rg --files --hidden -g '!.git'` で現状を確認し、既存記事や未コミット変更を保持する。
5. `mise.toml`、`mise.lock`、`package.json`、`Makefile`、既存の lint 設定を読み、CLI と検査コマンドのプロジェクト規約を特定する。
6. 対象サービスを会話から判定する。Zenn、Qiita、両方の指定がなければ、会話または対象リポジトリの文脈から推定し、成果物が変わるのに判定できない場合だけ確認する。
7. 公開を明示されていない限り、記事は下書きとして作成し、外部サービスへの投稿や `git push` は実行しない。

## 会話を記事の設計へ変換する

次の順で記事の骨子を作る。

1. 読者、記事の目的、前提、問題、実施内容、結果、注意点を会話から抽出する。
2. ユーザーが提示した URL、コマンド出力、ファイル内容、検証結果を根拠として扱い、会話にない数値、効果、仕様、経験談を補わない。
3. 事実、推測、提案を本文上で区別する。外部情報を補う場合は公式ドキュメントを優先し、参照元をリンクする。
4. タイトル、slug、概要、見出し、コード例、公開状態を決める。slug は指定がなければ内容を表す小文字の英数字とハイフンで作り、既存ファイルと衝突させない。
5. 会話に秘密情報、アクセストークン、個人情報、内部 URL が含まれていても記事へ転載しない。実値が必要な箇所は安全なプレースホルダーへ置き換える。

## CLI で記事の雛形を作る

対象リポジトリを特定できるまで、記事ディレクトリや雛形を作成しない。特定後は、記事ディレクトリや雛形を手作業で作らず、対象リポジトリで有効な CLI を `mise` 経由で使う。

1. Zenn CLI が使えない場合は `mise install` を実行する。必要なら `mise exec -- zenn init` で Zenn の初期構成を生成する。
2. Qiita CLI が使えない場合は `mise install` を実行する。必要なら `mise exec -- qiita init` で Qiita の設定と GitHub Actions を生成する。
3. Zenn の記事は `mise exec -- zenn new:article --slug <slug>` で作成し、`articles/<slug>.md` を編集する。
4. Qiita の記事は `mise exec -- qiita new <basename>` で作成し、CLI が生成した `public/<basename>.md` を編集する。
5. 生成先が既に存在する場合は内容を読み、既存記事を上書きしない。更新依頼でない限り新しい slug または basename を選ぶ。

## Zenn の記事形式

Zenn の記事は `articles/` に置く。生成された front matter を保ち、会話の内容に合わせて値だけを更新する。

```yaml
---
title: "記事タイトル"
emoji: "📝"
type: "tech"
topics:
  - "topic"
published: false
---
```

- 技術的な内容は `type: "tech"`、考察やアイデアは `type: "idea"` とする。
- topic は記事に直接関係するものだけを選び、サービス名や一般的すぎる語を過剰に追加しない。
- 公開を明示された場合だけ `published: true` とする。公開予約などの日時は依頼がある場合だけ設定する。
- 画像は実在するファイルだけを参照する。Zenn の GitHub 連携で管理する画像はリポジトリの `images/` に置き、記事から `/images/<path>` で参照する。

## Qiita の記事形式

Qiita の記事は `public/` に置く。`qiita new` が生成した front matter を基準にし、Zenn の `topics` を Qiita の `tags` へ機械的にコピーしない。

```yaml
---
title: 記事タイトル
tags:
  - tag
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: true
---
```

- 公開前の下書きは `ignorePublish: true` とし、明示的に公開を依頼された場合だけ `false` にする。
- 限定共有を依頼された場合だけ `private: true` とする。公開記事では `private: false` を使う。
- 既存記事の `id` と `updated_at` は消さない。Qiita CLI と GitHub Actions が投稿・更新時に管理する。
- `organization_url_name` と `slide` は依頼がない限り生成値を維持する。

## 本文を仕上げる

1. タイトルと front matter の内容が本文と一致することを確認する。
2. 冒頭で背景と結論を示し、その後に手順、判断理由、検証結果、制約、次の行動を並べる。
3. コマンドは実際に会話やリポジトリで確認できるものだけを載せ、実行結果を捏造しない。
4. Zenn と Qiita のどちらにも投稿する場合、本文をそのまま複製せず、見出し、リンク、タグ、説明の粒度を各読者向けに調整する。ただし、事実と検証結果は一致させる。
5. 下書き状態では、読者に見せる本文へエージェントの内部推論や作業者向けメモを残さない。

## 公開前の必須検査

作成または変更した Markdown ファイルを明示的に対象にし、次の検査を必ず両方実行する。片方が失敗しても、もう片方を省略せず、全結果を確認してから修正する。

### markdownlint

1. 既存の `package.json` scripts、`mise.toml` tasks、CI 定義を確認し、プロジェクトで定めた `markdownlint` または `markdownlint-cli2` のコマンドを優先する。
2. 専用コマンドがない場合は、設定ファイル（`.markdownlint*`）を確認し、`markdownlint-cli2 "<対象ファイル>"` を実行する。CLI が未導入なら、リポジトリのパッケージ管理方針に従って導入し、導入できない場合は未完了として報告する。
3. 対象ファイルを修正したら、同じ対象に再実行する。警告を許容する設定でも、エラーが残る状態を合格としない。

### textlint

1. 既存の `package.json` scripts、`mise.toml` tasks、`.textlintrc*`、`textlint` 設定を確認し、プロジェクトで定めたコマンドを優先する。
2. 設定がある場合は、その設定とルールで `textlint <対象ファイル>` を実行する。
3. 設定もルールもない場合は、Node.js のパッケージ管理方針に従って `textlint` と `textlint-rule-preset-ja-technical-writing` を導入し、`textlint --preset ja-technical-writing <対象ファイル>` を実行する。textlint はルールを内蔵しないため、ルールなしの実行を検査成功とみなさない。
4. 誤検知を例外扱いにする場合は、対象語と理由を設定へ明示する。本文を無理由に書き換えて検査を隠すことは禁止する。
5. 日本語本文を修正したら、同じ対象に再実行する。textlint を実行できない場合は完了と報告せず、未検査として残す。

### 追加の受け入れ確認

- `git diff --check` を実行し、空白エラーがないことを確認する。
- Zenn は `articles/`、Qiita は `public/` にあることを確認する。
- front matter の必須キー、公開状態、slug、タグまたは topic を確認する。
- Qiita の公開設定を使う場合は `.github/workflows/publish.yml`、`QIITA_TOKEN`、`permissions: contents: write` の存在を確認する。
- Zenn の公開設定を使う場合は、同期先ブランチを Zenn のダッシュボードで設定する必要があることを報告する。GitHub 連携の画面操作や `git push` は明示的な依頼なしに代行しない。

## 完了報告

次の内容を日本語で簡潔に報告する。

- 作成または更新したファイルと対象サービス。
- 会話から採用した要点と、判断した公開状態。
- 実行した `markdownlint` と `textlint` の実コマンドおよび結果。
- 未検査、未設定、GitHub Secret、Zenn のアカウント連携など、ユーザーの操作が必要な残作業。

lint のどちらか一方でも未実行、失敗、ルールなしの場合は「完了」と書かず、未完了の理由を示す。

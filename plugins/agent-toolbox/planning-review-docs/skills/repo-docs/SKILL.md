---
name: repo-docs
description: GitHubリポジトリで整備するガイドライン系ドキュメント（README.md、CONTRIBUTING.md、Issue/PR テンプレート、CODEOWNERS 等）の規約。ファイルの要否判断、配置、各ファイルの必須セクション、禁止パターン、DoD を含む。
---

# GitHub リポジトリのガイドライン整備規約

## 基本方針

- 対象は GitHub 配下の非公開リポジトリ。OSS 公開向けの CODE_OF_CONDUCT、FUNDING、LICENSE、SECURITY の外部開示は対象外
- リポジトリを開いた人が「これは何か」「どう動かすか」「どう貢献するか」「困ったらどこを見るか」の 4 つに、README とリンクだけで辿り着ける状態にする
- ドキュメントはコードの source of truth に従属させる。バージョン番号、環境変数、依存パッケージなどはコード側で管理し、ドキュメントからは参照だけにする
- 「あるべきだが空」より「まだ書いていない」の方が誠実。中身のない `CONTRIBUTING.md` を置くくらいなら削除する
- ファイル名と配置は GitHub が認識する規約（`.github/` 配下、`PULL_REQUEST_TEMPLATE.md` 等の命名）に従う

## ファイル一覧と要否判断

| ファイル | 判断基準 | 配置 |
|---------|---------|------|
| `README.md` | 必須（例外なし） | リポジトリ直下 |
| `.github/PULL_REQUEST_TEMPLATE.md` | 複数人が PR を作る全てのリポジトリ | `.github/` |
| `.github/ISSUE_TEMPLATE/` | Issue を受け付けるリポジトリ | `.github/ISSUE_TEMPLATE/` |
| `CODEOWNERS` | レビュアーの自動割当が必要 | `.github/CODEOWNERS` |
| `CONTRIBUTING.md` | 外部 or 別チームから貢献を受ける | 直下または `.github/` |
| `CHANGELOG.md` | バージョンを切ってリリースする | 直下 |
| `SUPPORT.md` | 問い合わせ導線を Issue と分けたい | 直下または `.github/` |

配置ルールの補足として、直下と `.github/` の両方で認識されるファイルは、直下にコードやスクリプトが多いなら `.github/` に寄せる。GitHub の UI（Insights → Community Standards）で認識されているかを最後に確認する。

## 各ファイルの構成

### README.md

README は配置階層ごとに役割が違う。ルート／アプリ／ライブラリ／docs の 4 種別を混同すると、ルートに詳細な API が並んだり、アプリ配下に全体像が書かれたりして読者が迷う。

| 種別 | 配置 | 主読者 | 主目的 |
|-----|-----|-------|-------|
| ルート README | `/README.md` | 初めてリポジトリを開く人、新規メンバー | 全体像と入口。「これは何か」「どこに何があるか」を示す |
| アプリ／サービス README | `/apps/<name>/README.md`, `/services/<name>/README.md` | そのアプリを触る開発者・オンコール | 単体で動かす手順と、システムでの役割 |
| パッケージ README | `/packages/<name>/README.md`（配布物では npm/PyPI にそのまま出る） | ライブラリ利用者 | インストール、API、使用例 |
| docs 配下の README | `/docs/README.md` | ドキュメント検索者 | 目次。個別ドキュメントへの導線 |

種別を判別する目安として、そのディレクトリが「単体でビルド／実行できる成果物」なら**アプリ／パッケージ README**、そうでなく複数の成果物や設定をまとめる場所ならその親側の README がカバーする。

#### ルート README（`/README.md`）の必須セクション

1. プロジェクト名と 1 行説明（何をするか、誰のためか）
2. 概要（3〜5 行。動機と現状のスコープ）
3. リポジトリ構成（monorepo なら各サブディレクトリの役割と、各 README への相対リンク）
4. 動作環境（対応 OS、必要な言語ランタイム、依存する外部サービス）
5. 全体セットアップ（clone から `mise install` / `pnpm install` 等、ワークスペース全体の準備まで）
6. よく使うコマンド（ワークスペース横断のタスク。個別アプリ固有のコマンドはサブ README に委ねる）
7. トラブルシューティング（全体セットアップで起きがちな失敗）
8. 外部リンク（CONTRIBUTING、上位ドキュメント、Notion / Confluence 等の関連ページ）

**書かない**もの。

- 個別アプリのエンドポイント一覧、内部 API 仕様（→ アプリ README）
- 個別ライブラリの使い方（→ パッケージ README）
- ソースを見れば分かる関数一覧、クラス図

#### アプリ／サービス README（`/apps/<name>/README.md` 等）の必須セクション

1. アプリ名と 1 行説明（このアプリがシステム全体で何を担うか）
2. アーキテクチャ位置づけ（依存する他アプリ／サービス、呼び出される側／呼び出す側の関係）
3. 動作環境（このアプリ固有のランタイム、外部依存、必要なローカルサービス）
4. セットアップ（このアプリだけを触りたい人向け。ルートのセットアップ済みを前提にしてよい）
5. 起動方法（開発モード、本番相当モード、コンテナ起動など）
6. 使い方の入口（Web なら URL とルート一覧の見出し、CLI ならエントリコマンド、API なら OpenAPI/エンドポイント一覧のリンク）
7. ディレクトリ構成（このアプリ内の主要ディレクトリの役割）
8. コマンド一覧（このアプリの `package.json` / `Taskfile` 等で定義されているタスクの目的）
9. 環境変数（このアプリが読む変数のみ。値ではなく、変数名・必須／任意・用途・既定値）
10. テスト実行方法（単体・統合・E2E の分類とコマンド）
11. トラブルシューティング（このアプリで起きがちな失敗）
12. デプロイ／運用メモ（本番の動線、監視ダッシュボード、oncall ランブックへのリンク）

#### パッケージ README（`/packages/<name>/README.md`、社内ライブラリ）の必須セクション

社内レジストリ（GitHub Packages、CodeArtifact、社内 PyPI 等）に配布したときの説明ページとしてそのまま表示されることを意識する。

1. パッケージ名と 1 行説明
2. インストール（`npm i @org/<name>` / `uv add <name>` 等。社内レジストリの認証設定が必要ならそのリンク）
3. 最小の使用例（コピペで動くコードブロック）
4. 公開 API 一覧（型シグネチャと 1 行説明、詳細は生成ドキュメントへリンク）
5. 対応バージョン（Node.js / Python 等のランタイム下限）
6. 変更履歴へのリンク（CHANGELOG または Releases）
7. 貢献方法（リポジトリの CONTRIBUTING への GitHub URL リンク。レジストリ表示時に切れないよう相対リンクではなくフルパスで書く）

#### docs 配下の README（`/docs/README.md`）

目次に徹する。個別ドキュメントの中身は書かない。

1. このディレクトリで扱うドキュメントの範囲
2. 目次（各ドキュメントへの相対リンクと 1 行説明）
3. 追加時のルール（命名規則、置き場所、レビュー導線）

#### 全 README 共通のルール

- 環境変数は `.env.example` を各アプリ／パッケージに置き、README では値の一覧を書かず存在を案内するだけにする（README とコードの drift を防ぐ）
- コマンド一覧はタスクランナー定義（`Taskfile`、`package.json`、`mise.toml` 等）を source of truth とし、README からは「詳細は `<定義ファイル>` を参照」に留める
- 相対リンクはリポジトリ内配置に耐えるように書く（配布物では GitHub URL に書き換えるビルドがない限り相対で構わない）
- 「昔はこうだった」を残さない。削除された機能・非推奨化された手順は即座に消す

### CONTRIBUTING.md

「これから初めて PR を出す人」向けに、開発から PR マージまでの道のりを一直線に書く。

必須セクション。

1. 開発環境の構築（README のセットアップと重複するなら README を参照する）
2. ブランチ運用（命名規則、main への直接 push 禁止など）
3. コミット規約（Conventional Commits を使うか、日本語一行か等）
4. テスト実行方法（タスクランナー経由で）
5. lint / format 実行方法（同上）
6. PR 作成の手順（テンプレートに沿う、CI が緑になってから review 依頼、等）
7. レビュー観点（何を見るか、CODEOWNERS のガイド）
8. Issue の切り方（テンプレートを使う、再現手順を書く）

コード品質基準（テストカバレッジ、型注釈、命名規則など）は個別 skill / 設定ファイルに委ね、CONTRIBUTING からは参照する。ここに転記すると skill との drift が生まれる。

### .github/PULL_REQUEST_TEMPLATE.md

セクション構造は 3 つで足りる。過剰に増やすと空欄が並ぶだけの PR が量産される。

```markdown
## 概要

<!-- 何をなぜ変えたかを 2〜3 行。具体的な変更内容は「変更点」に書く -->

## 変更点

<!-- 箇条書きで具体的な変更内容 -->

## 動作確認

- [ ] ローカルで lint / test が通ることを確認
- [ ] 関連する Issue: #

## その他

<!-- レビュー時に注目してほしい点、後回しにした事項、スクリーンショット等 -->
```

タイトルと「概要」は見出しであって詳細の置き場ではない。タイトルはその PR が扱う 1 事項を 1 行で示し、ファイル名や変更理由、「〜し、〜も」で繋いだ二つ目の変更は入れない。概要は何をなぜ変えたかに留め、変更内容の列挙は「変更点」、検証結果は「動作確認」、レビューへの注記は「その他」がそれぞれ受け持つ。セクションが用意されている事柄を概要に重ねて書くと、読み手は同じ話を二度読むことになる。

複数種類の PR（feature / bugfix / release）を扱うなら、`.github/PULL_REQUEST_TEMPLATE/` ディレクトリ配下に複数ファイルを置き、`?template=xxx.md` クエリで切り替える。

### .github/ISSUE_TEMPLATE/

YAML 形式（`.yml`）を使う。Markdown 形式（`.md`）は選択肢の型検証ができない。

配置例。

```
.github/
└── ISSUE_TEMPLATE/
    ├── bug_report.yml      # バグ報告
    ├── feature_request.yml # 機能提案
    └── config.yml          # Issue 作成画面の設定
```

`config.yml` では `blank_issues_enabled: false` を設定し、テンプレート以外の Issue を作らせない運用にできる。外部リンク（Discord、Slack、Discussions など）を `contact_links` で誘導する。

Issue のタイトルも PR と同じで、起きている事象を 1 行で示すに留める。再現手順、環境、ログはそれぞれ専用の欄が受け持つので、タイトルに詰め込まない。`title` に接頭辞を持たせたいときは、テンプレート側の `title:` に `[バグ] ` のような雛形を置いて入力者に書かせない。

`bug_report.yml` の最低構成。

```yaml
name: バグ報告
description: 想定と違う挙動を報告する
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
      placeholder: v1.2.3
    validations:
      required: true
  - type: textarea
    id: environment
    attributes:
      label: 環境
      description: OS、ランタイム、関連する依存
```

### CODEOWNERS

`.github/CODEOWNERS` に置く（直下でも認識されるが `.github/` が慣例）。

- パスは globs、上から順に評価され、**最後にマッチした行が有効**
- チーム指定は `@org/team-name`、個人指定は `@username`
- 対象ファイルの PR で自動的にレビュアー割当される（Branch protection の required reviewers と組み合わせて機能する）

```
# デフォルトのオーナー
*           @kunikeni

# インフラは infra チーム
/terraform/ @org/infra
/.github/   @org/infra

# ドキュメントは docs チーム
*.md        @org/docs
```


### CHANGELOG.md

バージョンタグを切る運用でのみ用意する。Web サービスの継続デプロイ運用では GitHub Releases + PR タイトルで十分。

- [Keep a Changelog](https://keepachangelog.com/) 形式を推奨
- 手書きで維持するなら PR マージ時に必ず追記する運用に合意する
- 自動生成なら `release-drafter` などの action で PR ラベルから組み立てる

## リポジトリ種別ごとの推奨セット

Organization 内で扱う 3 種別を対象とする。

| 種別 | 最低限 | 推奨追加 |
|-----|-------|---------|
| 個人・実験用（Organization 配下の検証リポジトリ） | README | — |
| 単一チームで所有 | README, PR テンプレート, CODEOWNERS | CONTRIBUTING（オンボーディング用途）、ISSUE テンプレート |
| 複数チーム利用（社内ライブラリ、共通基盤等） | README, PR/Issue テンプレート, CODEOWNERS, CONTRIBUTING | CHANGELOG（バージョン管理する場合）、docs 配下の目次 README |

## 更新の考え方

- README のセットアップ手順は「新しく入った人が上から実行して動く」ことを定期的に検証する。動かなくなったら即修正する
- 環境変数の追加は `.env.example` と同時に行う。片方だけの変更は禁止
- 依存ツールのバージョンを README に書くなら、`mise.toml` / `.tool-versions` などの実定義との整合を必ず取る（もしくは README には書かず「詳細は `mise.toml` を参照」とする）
- 削除された機能・非推奨化された手順は README から即座に削除する。「昔はこうだった」を残さない

## 禁止パターン

| 禁止 | 代替 |
|------|------|
| README に環境変数の値そのものを書く | `.env.example` を置き、README からは案内だけ |
| README にコマンド一覧をベタ書きし、タスクランナー定義と重複させる | タスクランナー定義を source of truth にし、README からは「詳細は `Taskfile` / `package.json` 参照」に |
| ルート README に個別アプリの API 仕様やエンドポイント一覧を書く | アプリ／サービス README に置き、ルートからはリンクだけ |
| Issue / PR テンプレートを `.md` で書き、選択肢を自然文で並べる | `.yml` で `type: dropdown` / `checkboxes` を使う |
| PR / Issue のタイトルに複数の変更やファイル名、理由を詰め込む | 扱う 1 事項だけを 1 行で書き、残りは本文の該当セクションへ |
| 「概要」に変更内容の列挙や検証結果を書く | 概要は 2〜3 行に留め、「変更点」「動作確認」に振り分ける |
| CONTRIBUTING に「テストを書きましょう」等の抽象論を並べる | 具体的なコマンドと最低カバレッジ・除外対象を書く |
| CHANGELOG に「小さい変更」「バグ修正」など内容のない項目 | 変更内容と影響を 1 行で明記、内部リファクタは載せない |
| README 冒頭に長い目次を置く（GitHub は自動生成する） | 目次を書かず GitHub の自動目次に任せる |

## Definition of Done

新規リポジトリ立ち上げ、または既存リポジトリのドキュメント整備 PR の完了条件。

- [ ] リポジトリ種別に応じた最低限のファイルが揃っている（上の表を参照）
- [ ] ルート／アプリ／パッケージ／docs の各 README が、階層に応じた責務で書かれている（ルートに個別 API が混ざっていない、アプリ README に全体像が漏れていない等）
- [ ] README のセットアップ手順を上から実行して、実際に起動できることを確認した
- [ ] 環境変数は `.env.example` に列挙され、README からはそこへ案内している
- [ ] コマンド一覧はタスクランナー定義と乖離していない
- [ ] Issue / PR テンプレートは `.yml` 形式で、必須項目に `required: true` が付いている
- [ ] CODEOWNERS のパスが実在し、指定されたユーザー／チームが Organization に存在する
- [ ] Branch protection の required reviewers と CODEOWNERS が整合している
- [ ] 各ドキュメント内の相互リンクが切れていない（`markdown-link-check` 等で確認）

### 検証コマンド例

```bash
# リンク切れ検査
npx markdown-link-check README.md CONTRIBUTING.md

# Issue テンプレートの構文確認
gh api repos/<org>/<repo>/contents/.github/ISSUE_TEMPLATE

# CODEOWNERS の解釈確認（GitHub UI）
open https://github.com/<org>/<repo>/settings/access
```

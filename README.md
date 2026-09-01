# AI Skills Plugin

Codex と Claude Code で共通利用できる、開発支援 skill の marketplace 。

## 概要

設計、実装、テスト、レビュー、Git、GitHub Actions、AWS、Terraform、技術記事作成などの作業規約を、用途別のプラグインとして配布します。

各 skill は `plugins/agent-toolbox/<plugin-name>/skills/` にあります。
Codex 用と Claude Code 用のマニフェストを同じプラグインに収録しています。

## 収録プラグイン

### `coding-workflow`

コーディングと開発ワークフローを扱います。
`backend-patterns`、`coding-rules`、`coding-standards`、
`git-workflow`、`tdd-workflow` を収録しています。

### `general-tools`

AWS、GitHub Actions、Terraform の運用規約を扱います。
`aws-cli`、`gh-actions`、`terraform` を収録しています。

### `planning-review-docs`

計画、レビュー、リポジトリ文書、セキュリティを扱います。
`planner`、`project-guidelines-example`、`repo-docs`、
`security-review` を収録しています。

### `specialized`

事後検証と技術記事作成を扱います。
`postmortem`、`write-zenn-qiita-articles` を収録しています。

## 動作環境

Git と、プラグイン機能に対応した Codex CLI または Claude Code が必要です。利用する CLI だけを導入してください。

## Codex への導入

marketplace を登録します。

```bash
codex plugin marketplace add navy1634/ai-skills-plugin
```

必要なプラグインを追加します。すべて利用する場合は、次の4件を実行します。

```bash
codex plugin add coding-workflow@agent-toolbox
codex plugin add general-tools@agent-toolbox
codex plugin add planning-review-docs@agent-toolbox
codex plugin add specialized@agent-toolbox
```

## Claude Code への導入

marketplace を登録します。

```bash
claude plugin marketplace add navy1634/ai-skills-plugin
```

必要なプラグインを追加します。すべて利用する場合は、次の4件を実行します。

```bash
claude plugin install coding-workflow@agent-toolbox
claude plugin install general-tools@agent-toolbox
claude plugin install planning-review-docs@agent-toolbox
claude plugin install specialized@agent-toolbox
```

## 更新

Codex では marketplace を更新した後、更新するプラグインを追加し直します。

```bash
codex plugin marketplace upgrade agent-toolbox
codex plugin add coding-workflow@agent-toolbox
```

Claude Code では marketplace と対象プラグインを順に更新します。

```bash
claude plugin marketplace update agent-toolbox
claude plugin update coding-workflow@agent-toolbox
```

更新した skill を確実に読み込むため、更新後は新しいセッションを開始してください。

## ローカル開発

リポジトリを取得し、ルートで marketplace 定義を検証します。

```bash
git clone https://github.com/navy1634/ai-skills-plugin.git
cd ai-skills-plugin
claude plugin validate .
```

各 `SKILL.md` は、ファイル先頭に `name` と `description` を含む
YAML フロントマターが必要です。`Skipped loading` と表示された場合は、
対象ファイルの先頭が `---` で始まっていることと、フロントマターより前に
テンプレート指示や本文がないことを確認してください。

marketplace の構成は
[`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json)、
各プラグインの定義は
`plugins/agent-toolbox/<plugin-name>/.codex-plugin/plugin.json` と
`plugins/agent-toolbox/<plugin-name>/.claude-plugin/plugin.json` を参照してください。

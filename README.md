# AI Skills Plugin

Codex と Claude Code で共通利用できる、開発支援 skill の marketplace。

## 概要

設計、実装、テスト、レビュー、Git、GitHub Actions、AWS、Terraform、技術記事作成などの作業 skill を、用途別のプラグインとして配布する。agent と常時適用する rules は marketplace に複製せず、chezmoi の共通原本で管理する。

各 skill は `plugins/agent-toolbox/<plugin-name>/skills/` にある。
Codex 用と Claude Code 用のマニフェストを同じプラグインに収録している。

## リポジトリ構成

```text
.
├── .claude-plugin/
│   └── marketplace.json
├── plugins/
│   ├── README.md
│   └── agent-toolbox/
│       └── <plugin-name>/
│           ├── .claude-plugin/plugin.json
│           ├── .codex-plugin/plugin.json
│           └── skills/
│               └── <skill-name>/
│                   ├── SKILL.md
│                   └── references/
└── README.md
```

`.claude-plugin/marketplace.json` は marketplace と plugin の一覧を定義し、各 plugin の `plugin.json` は製品ごとの plugin metadata を定義する。収録 plugin と skill の詳細は [plugins/README.md](plugins/README.md) を参照する。

## 動作環境

Git と、プラグイン機能に対応した Codex CLI または Claude Code が必要。

## Codex への導入

marketplace を登録する。

```bash
codex plugin marketplace add navy1634/plugin-agent-toolbox
```

必要なプラグインを追加する。すべて利用する場合は、次の4件を実行する。

```bash
codex plugin add coding-workflow@agent-toolbox
codex plugin add general-tools@agent-toolbox
codex plugin add planning-review-docs@agent-toolbox
codex plugin add specialized@agent-toolbox
```

## Claude Code への導入

marketplace を登録する。

```bash
claude plugin marketplace add navy1634/plugin-agent-toolbox
```

必要なプラグインを追加する。すべて利用する場合は、次の4件を実行する。

```bash
claude plugin install coding-workflow@agent-toolbox
claude plugin install general-tools@agent-toolbox
claude plugin install planning-review-docs@agent-toolbox
claude plugin install specialized@agent-toolbox
```

## 更新

Codex では marketplace を更新した後、更新するプラグインを追加し直す。

```bash
codex plugin marketplace upgrade agent-toolbox
codex plugin add coding-workflow@agent-toolbox
```

Claude Code では marketplace と対象プラグインを順に更新する。

```bash
claude plugin marketplace update agent-toolbox
claude plugin update coding-workflow@agent-toolbox
```

更新した skill を確実に読み込むため、更新後は新しいセッションを開始する。

## ローカルでの検証

リポジトリを取得し、ルートで marketplace 定義を検証する。

```bash
git clone https://github.com/navy1634/plugin-agent-toolbox.git
cd plugin-agent-toolbox
claude plugin validate .
```

各 `SKILL.md` は、ファイル先頭に `name` と `description` を含む
YAML フロントマターが必要。`Skipped loading` と表示された場合は、
対象ファイルの先頭が `---` で始まっていることと、フロントマターより前に
テンプレート指示や本文がないことを確認する。

marketplace の構成は
[`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json)、
各プラグインの定義は
`plugins/agent-toolbox/<plugin-name>/.codex-plugin/plugin.json` と
`plugins/agent-toolbox/<plugin-name>/.claude-plugin/plugin.json` を参照する。

## スキルと共通設定の境界

この marketplace が配布するのは、特定作業で必要な skill です。`SKILL.md` には、その skill を使う時点で必要な適用条件、汎用的な判断、完了条件を記載し、`references/` には言語、媒体、provider、ツールなどに固有の手順や具体例を記載します。利用時は `SKILL.md` の汎用契約を読み、対象に関係する reference を追加で読みます。

planner skill は planner agent 経由の作業でも、agent を経由しない計画作成でも利用します。agent の責務、Approval、常時適用する `AGENTS.md` と rules は marketplace に複製せず、Claude Code と Codex で共通利用する chezmoi の原本を正本として管理します。

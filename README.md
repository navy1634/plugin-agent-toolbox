# AI Skills Plugin

Codex と Claude Code で共通利用できる、開発支援 skill の marketplace 。

## 概要

設計、実装、テスト、レビュー、Git、GitHub Actions、AWS、Terraform、技術記事作成などの作業規約を、用途別のプラグインとして配布する。

各 skill は `plugins/agent-toolbox/<plugin-name>/skills/` にある。
Codex 用と Claude Code 用のマニフェストを同じプラグインに収録している。

## 動作環境

Git と、プラグイン機能に対応した Codex CLI または Claude Code が必要。

## Codex への導入

marketplace を登録する。

```bash
codex plugin marketplace add navy1634/ai-skills-plugin
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
claude plugin marketplace add navy1634/ai-skills-plugin
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

## ローカル開発

リポジトリを取得し、ルートで marketplace 定義を検証する。

```bash
git clone https://github.com/navy1634/ai-skills-plugin.git
cd ai-skills-plugin
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

# Qiita

## 投稿先を決める

次の順で Qiita と連携する repository を特定します。

1. 会話で投稿先 path が指定されていれば、その Git root を使う
2. 指定がなければ `ZENN_QIITA_REPOSITORY` を確認する
3. それもなければ current Git root を候補にし、`qiita.config.json`、`package.json` の Qiita CLI 宣言、または Qiita 公式 Action のいずれかと `public/` を確認する

候補が見つからなければ、記事 directory や別 repository を新しく作らず、投稿先の指定が必要だと報告します。投稿先の Git root へ移動して `git status --short --branch` と `rg --files --hidden -g '!.git'` を実行し、既存記事と未コミット変更を保持します。`mise.toml`、`mise.lock`、`package.json`、`Makefile`、lint 設定も読みます。

## 雛形と配置

記事は `public/<basename>.md` に置きます。手作業で front matter や空ファイルを作らず、対象 repository で有効な task または CLI を使います。

```bash
qiita new <basename>
```

CLI が未導入なら、対象 repository の規約に従って `mise install` を実行し、shell の PATH または task runner から CLI を呼び出します。環境を一回の command だけで明示的に用意する規約の場合に限り `mise exec -- qiita ...` を使います。必要な場合だけ `qiita init` で設定と GitHub Actions を生成します。既存 basename がある場合は内容を読み、更新依頼がない限り上書きせず、新しい basename を選びます。

## front matter と公開状態

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

公開前は `ignorePublish: true`、限定共有の依頼だけ `private: true` とします。Qiita の Markdown 記法、見出し、コードブロックに合わせ、front matter の値と本文のタイトルを一致させます。

`id` と `updated_at` は既存記事で削除せず、Qiita CLI と GitHub Actions の管理に任せます。`organization_url_name` と `slide` は依頼がない限り生成値を維持します。Zenn の topics を Qiita の tags へ機械的にコピーせず、Qiita の読者に直接関係する tag だけを選びます。

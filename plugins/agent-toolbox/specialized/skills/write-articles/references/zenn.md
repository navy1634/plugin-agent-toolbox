# Zenn

## 投稿先を決める

次の順で Zenn と連携する repository を特定します。

1. 会話で投稿先 path が指定されていれば、その Git root を使う
2. 指定がなければ `ZENN_QIITA_REPOSITORY` を確認する
3. それもなければ current Git root を候補にし、`mise.toml` または `package.json` に Zenn CLI の宣言と `articles/` または `books/` があることを確認する

候補が見つからなければ、記事 directory や別 repository を新しく作らず、投稿先の指定が必要だと報告します。投稿先の Git root へ移動して `git status --short --branch` と `rg --files --hidden -g '!.git'` を実行し、既存記事と未コミット変更を保持します。`mise.toml`、`mise.lock`、`package.json`、`Makefile`、lint 設定も読みます。

## 雛形と配置

記事は `articles/<slug>.md`、本は `books/` 配下に置きます。手作業で空ファイルを作らず、対象 repository で有効な task または CLI を使います。

```bash
zenn new:article --slug <slug>
```

CLI が未導入なら、対象 repository の規約に従って `mise install` を実行し、shell の PATH または task runner から CLI を呼び出します。環境を一回の command だけで明示的に用意する規約の場合に限り `mise exec -- zenn ...` を使います。必要な場合だけ `zenn init` で初期構成を作ります。既存 slug がある場合は内容を読み、更新依頼がない限り上書きせず、新しい slug を選びます。

## front matter と公開状態

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

技術記事は `type: tech`、考察は `type: idea` とし、topic は直接関係するものだけにします。公開依頼がない限り `published: false` を維持します。画像は実在する `images/` のファイルだけを参照し、記事からは `/images/<path>` で参照します。

タイトル、emoji、type、topics、published は本文と一致させます。公開予約や日時は依頼がある場合だけ設定し、GitHub 連携の同期先 branch はダッシュボード側の設定であることを完了報告へ残します。

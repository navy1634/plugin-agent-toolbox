# 記事検証

作成または変更した Markdown を明示的に列挙し、markdownlint と textlint の両方を同じ対象へ実行します。片方が失敗してももう片方を省略せず、両方の結果を確認してから修正します。

## 構成の受入検査

lint の前に、読者と目的が一文で定まり、本文が次の順序で追えることを確認します。内容に不要な章を省く場合は、記事の目的に照らした理由を記録します。

- タイトルと概要が一致し、読者が得る価値を示している
- 背景と問題が確認済みの事実で説明されている
- version、環境、依存、前提知識が記載されている
- 結論または採用方針が手順より前に示されている
- 手順、設定、コード例が上から再現でき、前提 command が明示されている
- 判断理由、選択肢、不採用理由、トレードオフが事実と区別されている
- 検証 command、対象、結果、証拠の範囲が記載されている
- 制約、未検証、環境差、残るリスク、rollback が隠されていない
- 次の行動または運用上の引き継ぎ先が示されている

本文中の数値、効果、仕様、経験談、command、URL は根拠へ辿れます。コード例はコピーして実行できる最小構成にし、placeholder のままでは実値が必要な箇所を説明します。複数媒体の記事は、事実・引用元・検証結果が一致していることを突き合わせます。

## markdownlint

1. `package.json` scripts、`mise.toml` tasks、CI 定義に `markdownlint` または `markdownlint-cli2` があれば、その command と設定を優先する
2. 専用 command がなければ `.markdownlint*` を確認し、`markdownlint-cli2 "<対象ファイル>"` を実行する
3. CLI が未導入なら、repository の package manager 方針に従って導入を試み、導入できなければ未完了として報告する
4. 対象を修正したら同じ対象へ再実行し、警告を許容する設定でも error が残る状態を合格にしない

## textlint

1. `package.json` scripts、`mise.toml` tasks、`.textlintrc*`、textlint 設定があれば、その rule と command を優先する
2. 設定がなければ `textlint --preset ja-technical-writing <対象ファイル>` を使い、rule なしの実行を検査成功とみなさない
3. rule が未導入なら `textlint` と `textlint-rule-preset-ja-technical-writing` を package manager 方針に従って導入し、導入できなければ未完了として報告する
4. 誤検知を例外扱いにする場合は対象語と理由を設定へ明示し、本文を無理由に書き換えて検査を隠さない
5. 日本語本文を修正したら同じ対象へ再実行し、実行できない場合は完了と報告しない

## 追加確認

`git diff --check`、front matter の必須 key、slug／basename、Zenn の `articles/`、Qiita の `public/`、公開状態、相互リンク、コード例を確認します。Zenn と Qiita の本文を両方作った場合は、事実と検証結果が一致することも確認します。

Qiita の公開設定では `.github/workflows/publish.yml`、`QIITA_TOKEN`、`permissions: contents: write` の存在を確認します。Zenn の公開設定では同期先 branch を Zenn の dashboard で設定する必要があることを報告します。GitHub 連携の画面操作、外部投稿、`git push` は明示的な依頼なしに行いません。

完了報告には作成・更新ファイル、対象サービス、採用した事実、公開状態、実行した実コマンドと結果、未設定の secret／連携を記載します。未検査や rule 未導入なら完了扱いにしません。

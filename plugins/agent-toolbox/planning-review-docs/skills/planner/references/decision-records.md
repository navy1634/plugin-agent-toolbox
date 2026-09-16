# 設計判断と decision record

## 適用条件

この reference は、共有 rules または repository 規約が、plan 内の設計判断に加えて ADR などの decision record を要求するときに使います。decision record の作成者、更新者、保存先、命名は実行環境の共有 rules を正とし、この skill で役割分担を上書きしません。

別記録を要求されていない場合でも、非自明な判断の比較と結論は plan の `Design Decisions` から省きません。一方で、既存パターンに従うだけの機械的な変更や、局所的な実装詳細まで decision record にしません。

## 記録する判断

次のいずれかを変更または確定する判断は、別記録の候補です。

- architecture、責務境界、公開 contract、依存方向
- data model、migration、state、互換性
- authentication、authorization、secret、network などの security 境界
- 外部 service、provider、protocol、運用経路
- 複数の実装境界の ownership、依存順、段階的な移行
- 将来の変更コストまたは復旧方法へ継続的に影響する選択

## 必須内容

先に結論を選び、その結論を支持する候補や根拠だけを集めてはいけません。決定前に問題と制約を明確にし、要件と受入条件から評価基準を定め、複数の実行可能な候補を調査して同じ基準で比較します。

各記録には、repository の言語に合わせた同等の見出しを使えますが、次の内容をこの順序で含めます。

```markdown
# ADR: [Decision title]

## Background
[背景、解決する問題、制約]

## Evaluation Criteria
[要件、受入条件、リスクから導いた評価基準と優先順位]

## Options Considered
[調査した複数の現実的な選択肢と、同じ評価基準による比較]

## Rejected Because
[不採用の選択肢と理由]

## Decision
[採用した決定]

## Rationale
[採用理由と根拠]

## Impact
[実装、運用、互換性、security、cost への影響]

## Unresolved Items or Review Conditions
[未解決事項、見直す条件、必要な証拠]
```

問題、評価基準、複数の選択肢、不採用理由を、採用した決定より先に記録します。plan と decision record で結論、共有契約、影響、見直し条件が食い違わないようにします。

## フィードバック時の扱い

フィードバックが設計上の問題を示す場合は、plan の比較と決定を再評価し、共有 rules が指定する所有者へ decision record の更新を返します。実装上の不足だけであれば decision record を変更せず、plan の該当手順と契約に沿った修正へ限定します。

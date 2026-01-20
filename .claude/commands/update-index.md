TripTrip戦略計画プロジェクトの文書インデックスと行数カウンターを更新してください。

## タスク

### 1. 文書の検出と確認
以下のディレクトリから新規作成または更新されたMarkdownファイルを検出：
- `business-strategy/` - 全サブフォルダ（追加セクション11-17含む）
- `it-strategy/` - 全サブフォルダ（追加セクション11-13含む）

**追加セクションの配置先**:
- Business Strategy: 11-appendix, 13-case-studies, 14-esg-strategy, 16-global-expansion, 17-organization-hr
- IT Strategy: 11-ai-ml-strategy, 12-specifications, 13-global-expansion

各ファイルについて：
- ファイル名から文書IDを特定（例: Doc-MA-001.md）
- ファイルの行数をカウント
- ファイルの最終更新日時を取得

### 2. document-index.mdの更新
`progress-tracking/document-index.md`を更新：
- 該当文書のステータスを「Pending」から「Complete」に変更
- 行数を記録
- 最終更新日を記録
- 担当エージェントを確認

### 3. line-counter.mdの更新
`progress-tracking/line-counter.md`を更新：
- 総行数を再計算
- 進捗率を更新
- セクション別の完了状況を更新
- エージェント別の生産性を更新

### 4. 更新サマリーの表示
更新結果を以下の形式で表示：

```
━━━━━━━━━━━━━━━━━━━━━━━
📝 文書インデックス更新完了
━━━━━━━━━━━━━━━━━━━━━━━

【新規/更新文書】
✅ Doc-XX-XXX: [文書名] - X,XXX行
✅ Doc-XX-XXX: [文書名] - X,XXX行

【進捗状況】
- 完了文書: X→Y文書（+Z）
- 総行数: X,XXX→Y,YYY行（+Z,ZZZ）
- 進捗率: X.X%→Y.Y%（+Z.Z%）

【現在のフェーズ】
フェーズX: [フェーズ名]（XX%完了）

【次の推奨タスク】
- Doc-XX-XXX: [文書名]
- Doc-XX-XXX: [文書名]
```

### 注意事項
- 既存のデータを破壊しないよう、慎重に更新
- 行数は実際のファイルから正確にカウント
- 文書IDと実際のファイル名の整合性を確認
- 更新前にバックアップを考慮

正確で効率的な更新処理を実行してください。
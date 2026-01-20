# Doc-TR-003: TripTrip Technical Debt Management - 技術的負債管理

## 文書メタデータ
| 項目 | 内容 |
|------|------|
| 文書ID | Doc-TR-003 |
| タイトル | Technical Debt Management - 技術的負債管理 |
| バージョン | 1.0.0 |
| 作成日 | 2026-01-20 |
| 最終更新日 | 2026-01-20 |
| 作成者 | Technical Architecture Team |
| 承認者 | CTO |
| 分類 | IT戦略 - 技術ロードマップ |
| 関連文書 | Doc-TR-001, Doc-TR-002, Doc-DM-002, Doc-QA-001 |

---

## 変更履歴

| バージョン | 日付 | 変更者 | 変更内容 |
|------------|------|--------|----------|
| 1.0.0 | 2026-01-20 | Technical Architecture Team | 初版作成 |

---

## 目次

1. [エグゼクティブサマリー](#1-エグゼクティブサマリー)
2. [はじめに & コンテキスト](#2-はじめに--コンテキスト)
3. [技術的負債の分類](#3-技術的負債の分類)
4. [負債評価フレームワーク](#4-負債評価フレームワーク)
5. [TripTrip現状の負債分析](#5-triptrip現状の負債分析)
6. [負債返済戦略](#6-負債返済戦略)
7. [負債防止メカニズム](#7-負債防止メカニズム)
8. [負債管理ロードマップ](#8-負債管理ロードマップ)
9. [KPIと成功指標](#9-kpiと成功指標)
10. [文書間参照 & 統合ポイント](#10-文書間参照--統合ポイント)

---

## 1. エグゼクティブサマリー

本文書は、TripTripプラットフォームにおける**技術的負債の包括的な管理戦略**を定義します。技術的負債は、短期的な利便性や時間的制約のために選択された技術的なショートカットや、時間の経過とともに蓄積されたレガシーコードを指します。

Google、Amazon、Netflix、Spotifyの技術的負債管理プラクティスを参考に、TripTripの現行システム（Flutter 35,000行、Node.js/TypeScript 4,700行）における負債を体系的に特定、評価、返済するフレームワークを確立します。

**主要目標**:
- 年間30%の技術的負債削減
- スプリントキャパシティの20%を負債返済に配分
- 新規負債発生率の50%削減
- 開発者生産性の20%向上

**現状の推定負債**: 約200人日相当（詳細は第5章参照）

---

## 2. はじめに & コンテキスト

### 2.1 技術的負債とは

```yaml
technical_debt_definition:
  concept:
    description: |
      技術的負債（Technical Debt）は、Ward Cunninghamが1992年に提唱した概念で、
      ソフトウェア開発において「正しい方法」ではなく「迅速な方法」を選択した結果
      蓄積される、将来的に対処が必要な技術的課題を指します。

    analogy: |
      金融負債と同様に、技術的負債には「元本」（最初のショートカット）と
      「利息」（保守コストの増加、開発速度の低下）が存在します。
      返済を先延ばしにするほど、利息が膨らみます。

  types:
    intentional_debt:
      description: "意図的な負債 - ビジネス要件のため意識的に選択"
      examples:
        - "リリース期限に間に合わせるためのショートカット"
        - "MVP開発のための機能簡略化"
        - "プロトタイプコードの本番利用"
      management: "計画的に返済をスケジュール"

    unintentional_debt:
      description: "非意図的な負債 - 知識不足や見落としで発生"
      examples:
        - "設計の理解不足によるアンチパターン"
        - "コードレビュー不足"
        - "ドキュメント不足による誤った実装"
      management: "教育とレビュープロセスで防止"

    environmental_debt:
      description: "環境的な負債 - 外部要因で発生"
      examples:
        - "依存ライブラリの非推奨化"
        - "フレームワークのバージョンアップ"
        - "セキュリティ脆弱性対応"
      management: "継続的なモニタリングと計画的アップデート"

    bit_rot:
      description: "腐敗 - 時間の経過とともに発生"
      examples:
        - "使用されなくなったコード"
        - "古くなったドキュメント"
        - "陳腐化したテスト"
      management: "定期的なクリーンアップ"
```

### 2.2 TripTripにおける負債管理の重要性

```yaml
importance_for_triptrip:
  current_stage:
    description: "高度な開発段階（バージョン1.0.0）"
    characteristics:
      - "18の実装済み機能"
      - "約35,000行のFlutterコード"
      - "約4,700行のバックエンドコード"
      - "フィーチャー別パッケージアーキテクチャ"

  growth_trajectory:
    year_1_target: "DAU 100,000+"
    year_3_target: "MAU 10,000,000+"
    year_5_target: "MAU 100,000,000+"
    implication: "スケーリングには技術基盤の健全性が不可欠"

  business_impact:
    development_velocity:
      current: "2週間スプリントで5-7機能"
      with_debt: "技術的負債により20-30%の速度低下リスク"
      goal: "負債管理により速度維持・向上"

    team_morale:
      risk: "レガシーコードとの格闘による開発者の不満"
      goal: "モダンで保守しやすいコードベース"

    scalability:
      risk: "負債がスケーリングのボトルネックに"
      goal: "グローバル展開に耐えるアーキテクチャ"

  strategic_alignment:
    - "Doc-TR-001の技術進化ロードマップ実現の前提条件"
    - "Doc-TR-002のマイグレーション戦略の基盤"
    - "マイクロサービス化の準備"
```

### 2.3 本文書の目的と範囲

```yaml
document_scope:
  objectives:
    - "TripTripの技術的負債を体系的に分類・評価"
    - "負債返済の優先順位付けフレームワークの確立"
    - "返済戦略とロードマップの策定"
    - "負債防止メカニズムの導入"
    - "KPIと進捗トラッキングの定義"

  scope:
    in_scope:
      - "フロントエンド（Flutter/Dart）の負債"
      - "バックエンド（Node.js/TypeScript）の負債"
      - "インフラストラクチャの負債"
      - "テスト・ドキュメントの負債"
      - "アーキテクチャの負債"

    out_of_scope:
      - "ビジネスプロセスの負債"
      - "組織的な負債"
      - "セキュリティ脆弱性（Doc-SC-001で対応）"

  audience:
    - "エンジニアリングチーム"
    - "テックリード"
    - "プロダクトマネージャー"
    - "経営層"
```

---

## 3. 技術的負債の分類

### 3.1 コード負債

```yaml
code_debt:
  definition: "コードレベルで蓄積された技術的課題"

  categories:
    complexity:
      description: "複雑性の負債"
      symptoms:
        - "循環的複雑度が高い関数"
        - "長すぎるメソッド（100行以上）"
        - "深すぎるネスト（4レベル以上）"
        - "多すぎるパラメータ（5個以上）"
      metrics:
        cyclomatic_complexity: "< 10 が理想"
        method_length: "< 50行が理想"
        nesting_depth: "< 3レベルが理想"
      impact:
        understanding: "High - 理解に時間がかかる"
        modification: "High - 変更リスクが高い"
        testing: "Medium - テストが複雑"

    duplication:
      description: "重複コードの負債"
      symptoms:
        - "コピー&ペーストされたコード"
        - "類似ロジックの複数実装"
        - "DRY原則の違反"
      metrics:
        duplication_ratio: "< 5% が理想"
      impact:
        maintenance: "High - 同じ変更を複数箇所で必要"
        bugs: "High - 修正漏れのリスク"

    obsolete_patterns:
      description: "古いパターンの負債"
      symptoms:
        - "非推奨API/ライブラリの使用"
        - "レガシーな設計パターン"
        - "モダンベストプラクティスからの乖離"
      impact:
        security: "Medium - セキュリティリスク"
        performance: "Low-Medium - 最適化不足"

    naming_clarity:
      description: "命名の負債"
      symptoms:
        - "意図が不明確な変数名"
        - "略語の過剰使用"
        - "不一致な命名規約"
      impact:
        understanding: "Medium - 理解コスト増"
        onboarding: "Medium - 新メンバーの学習曲線"

    dead_code:
      description: "デッドコードの負債"
      symptoms:
        - "未使用の関数/クラス"
        - "コメントアウトされたコード"
        - "到達不能なコードパス"
      impact:
        confusion: "Low - 混乱の原因"
        bundle_size: "Low - 不要な容量"
```

### 3.2 アーキテクチャ負債

```yaml
architecture_debt:
  definition: "システム設計レベルで蓄積された技術的課題"

  categories:
    monolith_remnants:
      description: "モノリス残存の負債"
      symptoms:
        - "密結合なコンポーネント"
        - "共有状態への過度な依存"
        - "明確でないモジュール境界"
      impact:
        scalability: "High - スケーリング困難"
        deployment: "High - 部分デプロイ不可"
        testing: "High - 独立テスト困難"

    layer_violations:
      description: "レイヤー違反の負債"
      symptoms:
        - "UIからの直接DB接続"
        - "ビジネスロジックのUI層混入"
        - "循環依存関係"
      impact:
        maintainability: "High - 保守困難"
        testability: "High - ユニットテスト困難"

    inconsistent_patterns:
      description: "パターン不一致の負債"
      symptoms:
        - "同じ問題に対する複数の解決策"
        - "チーム間での異なるアプローチ"
        - "ドキュメント化されていない例外"
      impact:
        consistency: "Medium - 一貫性欠如"
        learning: "Medium - 学習コスト増"

    scalability_constraints:
      description: "スケーラビリティ制約の負債"
      symptoms:
        - "シングルポイント・オブ・フェイラー"
        - "同期処理の過剰使用"
        - "状態のローカル保持"
      impact:
        scale: "Critical - スケール不可"
        availability: "High - 可用性リスク"

    api_inconsistency:
      description: "API不一致の負債"
      symptoms:
        - "異なるエンドポイント間での命名不一致"
        - "バージョニング戦略の欠如"
        - "エラーハンドリングの不統一"
      impact:
        integration: "Medium - 統合困難"
        documentation: "Medium - ドキュメント複雑化"
```

### 3.3 インフラ負債

```yaml
infrastructure_debt:
  definition: "インフラストラクチャレベルで蓄積された技術的課題"

  categories:
    legacy_dependencies:
      description: "レガシー依存の負債"
      symptoms:
        - "古いバージョンのライブラリ"
        - "EOLコンポーネント"
        - "パッチ適用の遅延"
      metrics:
        outdated_dependencies: "< 10% が理想"
        eol_components: "0 が理想"
      impact:
        security: "High - 脆弱性リスク"
        support: "Medium - サポート困難"

    manual_configuration:
      description: "手動設定の負債"
      symptoms:
        - "Infrastructure as Codeの不足"
        - "ドキュメント化されていない設定"
        - "環境間の差異"
      impact:
        reproducibility: "High - 再現困難"
        disaster_recovery: "High - DR困難"

    missing_observability:
      description: "可観測性不足の負債"
      symptoms:
        - "ログの不足または過剰"
        - "メトリクス収集の欠如"
        - "トレーシングの不足"
      impact:
        debugging: "High - 問題特定困難"
        operations: "High - 運用困難"

    resource_inefficiency:
      description: "リソース非効率の負債"
      symptoms:
        - "過剰プロビジョニング"
        - "リソースリークの放置"
        - "最適化されていないクエリ"
      impact:
        cost: "Medium - コスト増"
        performance: "Medium - パフォーマンス低下"
```

### 3.4 テスト・ドキュメント負債

```yaml
test_documentation_debt:
  definition: "テストとドキュメントに関する技術的課題"

  test_debt:
    low_coverage:
      description: "テストカバレッジ不足"
      symptoms:
        - "ユニットテストカバレッジ < 70%"
        - "統合テストの不足"
        - "E2Eテストの欠如"
      impact:
        confidence: "High - 変更への自信不足"
        regression: "High - リグレッションリスク"

    flaky_tests:
      description: "不安定なテスト"
      symptoms:
        - "ランダムに失敗するテスト"
        - "環境依存のテスト"
        - "順序依存のテスト"
      impact:
        trust: "High - テスト結果への不信"
        ci_cd: "High - CI/CDパイプライン遅延"

    missing_test_types:
      description: "テスト種別の欠如"
      symptoms:
        - "パフォーマンステストの不足"
        - "セキュリティテストの不足"
        - "負荷テストの不足"
      impact:
        quality: "Medium - 品質保証不足"
        production: "Medium - 本番問題のリスク"

  documentation_debt:
    outdated_docs:
      description: "古くなったドキュメント"
      symptoms:
        - "コードと乖離したドキュメント"
        - "存在しないAPIのドキュメント"
        - "誤った設定手順"
      impact:
        onboarding: "High - オンボーディング困難"
        operations: "Medium - 運用ミスリスク"

    missing_docs:
      description: "ドキュメント不足"
      symptoms:
        - "アーキテクチャ図の欠如"
        - "APIドキュメントの不足"
        - "ランブックの欠如"
      impact:
        knowledge: "High - 知識の属人化"
        incident: "High - インシデント対応遅延"
```

---

## 4. 負債評価フレームワーク

### 4.1 定量化手法

```yaml
quantification_methods:
  sqale_method:
    description: "SQALE (Software Quality Assessment based on Lifecycle Expectations)"
    approach:
      - "技術的負債を時間（修正工数）で定量化"
      - "品質特性ごとに分類"
      - "修正優先度を自動計算"
    tools:
      - "SonarQube"
      - "Code Climate"
    metrics:
      technical_debt_ratio: "負債時間 / 開発時間"
      debt_rating:
        A: "< 5%"
        B: "5-10%"
        C: "10-20%"
        D: "20-50%"
        E: "> 50%"

  code_climate_method:
    description: "Code Climate Quality Metrics"
    metrics:
      maintainability:
        A: "優秀 - 技術的負債なし"
        B: "良好 - 軽微な負債"
        C: "要注意 - 中程度の負債"
        D: "問題あり - 重大な負債"
        F: "深刻 - 致命的な負債"
      technical_debt_hours: "修正に必要な推定時間"
      code_smells: "検出されたコード臭の数"
      duplication: "重複コードの割合"

  custom_triptrip_method:
    description: "TripTrip固有の負債スコアリング"
    factors:
      - name: "修正コスト"
        weight: 30
        scale: "1-5 (1=軽微, 5=大規模)"
      - name: "ビジネスインパクト"
        weight: 25
        scale: "1-5 (1=低, 5=高)"
      - name: "リスクレベル"
        weight: 25
        scale: "1-5 (1=低, 5=高)"
      - name: "頻度/影響範囲"
        weight: 20
        scale: "1-5 (1=稀, 5=頻繁)"
    formula: |
      Debt Score = (修正コスト × 0.30) + (ビジネスインパクト × 0.25)
                 + (リスクレベル × 0.25) + (頻度 × 0.20)
    interpretation:
      1.0-2.0: "低優先度 - 機会があれば対応"
      2.1-3.0: "中優先度 - 計画的に対応"
      3.1-4.0: "高優先度 - 早期対応必要"
      4.1-5.0: "緊急 - 即時対応必要"
```

### 4.2 ビジネスインパクト評価

```yaml
business_impact_assessment:
  impact_dimensions:
    development_velocity:
      description: "開発速度への影響"
      measurement:
        - "機能開発のリードタイム"
        - "バグ修正時間"
        - "コードレビュー時間"
      indicators:
        high_impact: "新機能開発が50%以上遅延"
        medium_impact: "新機能開発が20-50%遅延"
        low_impact: "新機能開発が20%未満遅延"

    reliability:
      description: "システム信頼性への影響"
      measurement:
        - "インシデント発生頻度"
        - "MTTR（平均復旧時間）"
        - "可用性"
      indicators:
        high_impact: "月1回以上の重大インシデント"
        medium_impact: "四半期1回程度のインシデント"
        low_impact: "年1回未満のインシデント"

    scalability:
      description: "スケーラビリティへの影響"
      measurement:
        - "スループット上限"
        - "スケーリング時間"
        - "リソース効率"
      indicators:
        high_impact: "現在のスケールで限界"
        medium_impact: "10倍スケールで問題発生"
        low_impact: "100倍スケールでも問題なし"

    team_productivity:
      description: "チーム生産性への影響"
      measurement:
        - "オンボーディング時間"
        - "コードコントリビューション率"
        - "開発者満足度"
      indicators:
        high_impact: "新メンバーの立ち上げに3ヶ月以上"
        medium_impact: "新メンバーの立ち上げに1-3ヶ月"
        low_impact: "新メンバーの立ち上げに1ヶ月未満"

  cost_estimation:
    formula: |
      年間コスト =
        (開発速度低下率 × 年間開発コスト) +
        (インシデントコスト × 年間インシデント数) +
        (機会損失コスト)

    example:
      development_slowdown: "20% × ¥100M = ¥20M"
      incident_cost: "¥500K × 12 = ¥6M"
      opportunity_cost: "¥10M"
      total_annual_cost: "¥36M"
```

### 4.3 優先順位付けマトリックス

```yaml
prioritization_matrix:
  quadrant_model:
    description: "コスト vs インパクトの4象限"
    quadrants:
      q1_quick_wins:
        characteristics:
          cost: "低"
          impact: "高"
        strategy: "即座に対応"
        examples:
          - "デッドコードの削除"
          - "簡単なリファクタリング"
          - "ドキュメント更新"

      q2_major_projects:
        characteristics:
          cost: "高"
          impact: "高"
        strategy: "計画的に対応"
        examples:
          - "アーキテクチャ変更"
          - "大規模リファクタリング"
          - "フレームワークアップグレード"

      q3_fill_ins:
        characteristics:
          cost: "低"
          impact: "低"
        strategy: "余裕があれば対応"
        examples:
          - "コードスタイル統一"
          - "軽微な命名改善"

      q4_thankless_tasks:
        characteristics:
          cost: "高"
          impact: "低"
        strategy: "再評価または保留"
        examples:
          - "古いシステムの完全書き換え"
          - "過剰な最適化"

  decision_tree: |
    ┌─────────────────────────────────────────────────────────┐
    │                  技術的負債の優先順位決定                  │
    └─────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  セキュリティ    │
                    │  リスクあり？    │
                    └────────┬────────┘
                             │
              Yes ◄──────────┼──────────► No
               │             │             │
               ▼             │             ▼
        ┌──────────┐        │      ┌──────────────┐
        │ 緊急対応 │        │      │ ビジネス     │
        │          │        │      │ インパクト？ │
        └──────────┘        │      └──────┬───────┘
                            │             │
                            │     High ◄──┼──► Low
                            │       │     │     │
                            │       ▼     │     ▼
                            │  ┌────────┐ │ ┌────────┐
                            │  │修正コスト│ │ │ 保留   │
                            │  │ 評価   │ │ │または  │
                            │  └───┬────┘ │ │ 許容   │
                            │      │      │ └────────┘
                            │  Low │ High │
                            │   ▼  │  ▼   │
                            │┌────┐│┌────┐│
                            ││Q1  │││Q2  ││
                            ││即時│││計画││
                            │└────┘│└────┘│
                            └──────┴──────┘
```

---

## 5. TripTrip現状の負債分析

### 5.1 Flutterアプリの負債

```yaml
flutter_app_debt_analysis:
  overview:
    codebase_size: "約35,000行"
    feature_modules: 18
    architecture: "フィーチャー別パッケージ"
    maturity: "高度（v1.0.0）"

  identified_debts:
    debt_1_legacy_screens:
      category: "アーキテクチャ"
      location: "/lib/screens/"
      description: |
        レガシー画面が/lib/screens/に残存しており、
        フィーチャー別パッケージパターン（/lib/features/）への
        移行が未完了
      affected_files:
        - "rental_screen.dart"
        - "user_screen.dart（リファクタリング中）"
      severity: "Medium"
      effort_days: 10
      business_impact: "Medium"
      priority_score: 3.2
      remediation:
        approach: "Strangler Fig Pattern"
        steps:
          - "1. 新フィーチャーモジュールの作成"
          - "2. ルーティングの段階的移行"
          - "3. 旧画面の削除"
        timeline: "2-3 sprints"

    debt_2_http_client_inconsistency:
      category: "コード"
      location: "/lib/services/"
      description: |
        HTTPクライアントとして http と dio の両方が使用されており、
        統一されていない
      current_state:
        http_package: "シンプルなリクエスト用"
        dio_package: "インターセプター対応"
      severity: "Low"
      effort_days: 8
      business_impact: "Low"
      priority_score: 2.0
      remediation:
        approach: "Dioへの統一"
        rationale:
          - "インターセプターサポート"
          - "キャンセル対応"
          - "リトライ機構"
          - "より良いエラーハンドリング"
        steps:
          - "1. Dioラッパーサービスの作成"
          - "2. 既存httpコールの移行"
          - "3. httpパッケージの削除"
        timeline: "1 sprint"

    debt_3_state_management_inconsistency:
      category: "アーキテクチャ"
      location: "/lib/providers/"
      description: |
        状態管理がProvider（ChangeNotifier）とRiverpodの
        混在状態で、新機能開発時に選択基準が不明確
      current_state:
        provider_usage:
          - "言語プロバイダー"
          - "通貨プロバイダー"
          - "カートプロバイダー"
        riverpod_usage:
          - "商品リストプロバイダー"
          - "アトラクション詳細プロバイダー"
      severity: "Medium"
      effort_days: 40
      business_impact: "Medium"
      priority_score: 2.8
      remediation:
        approach: "Riverpod 3.xへの段階的移行"
        rationale:
          - "より強力な型安全性"
          - "テスタビリティ向上"
          - "コード生成サポート"
          - "モダンなリアクティブパターン"
        phases:
          phase_1: "新機能はRiverpodで実装（即時）"
          phase_2: "共通プロバイダーの移行（3ヶ月）"
          phase_3: "フィーチャー別プロバイダーの移行（6ヶ月）"
        timeline: "9 months"

    debt_4_missing_error_handling:
      category: "コード"
      location: "複数箇所"
      description: |
        一部のAPI呼び出しでエラーハンドリングが不十分、
        ユーザーへのエラー表示が不統一
      severity: "Medium-High"
      effort_days: 15
      business_impact: "High"
      priority_score: 3.5
      remediation:
        approach: "統一エラーハンドリングフレームワーク"
        steps:
          - "1. エラー型の定義"
          - "2. 共通エラーハンドラーの実装"
          - "3. UIエラー表示コンポーネントの統一"
          - "4. 既存コードへの適用"
        timeline: "2 sprints"

    debt_5_incomplete_tests:
      category: "テスト"
      location: "/test/"
      description: |
        46のテストファイルが存在するが、
        カバレッジが不十分な領域がある
      current_coverage:
        unit_tests: "約60%"
        widget_tests: "約40%"
        integration_tests: "約20%"
      severity: "Medium"
      effort_days: 25
      business_impact: "Medium"
      priority_score: 2.9
      remediation:
        target_coverage:
          unit_tests: "> 80%"
          widget_tests: "> 70%"
          integration_tests: "> 50%"
        approach: "テストピラミッドの確立"
        timeline: "Ongoing (20% sprint allocation)"

  flutter_debt_summary:
    total_debts: 5
    total_effort_days: 98
    priority_distribution:
      high: 2
      medium: 2
      low: 1
```

### 5.2 バックエンドの負債

```yaml
backend_debt_analysis:
  overview:
    codebase_size: "約4,700行"
    framework: "Hono + Node.js 18+"
    database: "PostgreSQL 16 + Prisma"
    architecture: "レイヤードREST API"

  identified_debts:
    debt_1_incomplete_validation:
      category: "コード"
      location: "/src/hono/schemas/"
      description: |
        Zodスキーマ検証が一部のエンドポイントで不完全、
        または重複したバリデーションロジックが存在
      severity: "Medium"
      effort_days: 8
      business_impact: "Medium"
      priority_score: 2.7
      remediation:
        approach: "Zodスキーマの完全化と統一"
        steps:
          - "1. 全エンドポイントのスキーマ監査"
          - "2. 欠落スキーマの追加"
          - "3. 重複の排除"
          - "4. OpenAPI仕様との同期確認"
        timeline: "1 sprint"

    debt_2_missing_logging:
      category: "インフラ"
      location: "/src/hono/middlewares/"
      description: |
        構造化ロギングが不完全、
        リクエストトレーシングが未実装
      severity: "Medium"
      effort_days: 12
      business_impact: "Medium-High"
      priority_score: 3.0
      remediation:
        approach: "構造化ロギング＆トレーシング導入"
        components:
          - "Winston/Pino for structured logging"
          - "Correlation ID middleware"
          - "Request/Response logging"
          - "Performance timing"
        timeline: "1-2 sprints"

    debt_3_hardcoded_config:
      category: "インフラ"
      location: "複数箇所"
      description: |
        一部の設定値がハードコードされており、
        環境間での切り替えが困難
      severity: "Low"
      effort_days: 5
      business_impact: "Low"
      priority_score: 1.8
      remediation:
        approach: "環境変数への外部化"
        tools:
          - "dotenv for local"
          - "AWS Secrets Manager for production"
        timeline: "1 sprint"

    debt_4_missing_rate_limiting:
      category: "セキュリティ/インフラ"
      location: "/src/hono/middlewares/"
      description: |
        レート制限が未実装、
        DDoS攻撃やAPI濫用に対する保護が不十分
      severity: "High"
      effort_days: 8
      business_impact: "High"
      priority_score: 3.8
      remediation:
        approach: "多層レート制限の導入"
        components:
          - "IP-based rate limiting"
          - "User-based rate limiting"
          - "Endpoint-specific limits"
          - "Redis for distributed state"
        timeline: "1 sprint"

    debt_5_database_optimization:
      category: "パフォーマンス"
      location: "Prisma queries"
      description: |
        一部のクエリにN+1問題が存在、
        インデックス最適化が未完了
      severity: "Medium"
      effort_days: 15
      business_impact: "Medium"
      priority_score: 2.8
      remediation:
        approach: "クエリ最適化とインデックス追加"
        steps:
          - "1. N+1クエリの特定（Prisma logging）"
          - "2. include/select最適化"
          - "3. インデックス分析と追加"
          - "4. 実行計画の確認"
        timeline: "2 sprints"

  backend_debt_summary:
    total_debts: 5
    total_effort_days: 48
    priority_distribution:
      high: 1
      medium: 3
      low: 1
```

### 5.3 インフラの負債

```yaml
infrastructure_debt_analysis:
  overview:
    current_state:
      hosting: "Docker Compose（ローカル/VPS）"
      database: "PostgreSQL 16（単一インスタンス）"
      ci_cd: "GitHub Actions（基本設定）"
      monitoring: "基本的"

  identified_debts:
    debt_1_local_only_infra:
      category: "インフラ"
      description: |
        本番環境のクラウドインフラが未構築、
        Docker Composeからの移行が必要
      severity: "High"
      effort_days: 30
      business_impact: "Critical"
      priority_score: 4.5
      remediation:
        approach: "AWS EKSへの移行（Doc-IA-002参照）"
        components:
          - "Terraformによるインフラ定義"
          - "EKSクラスタ構築"
          - "RDSへのDB移行"
          - "CI/CDパイプライン強化"
        timeline: "See Doc-TR-002"

    debt_2_missing_observability:
      category: "インフラ"
      description: |
        APM、分散トレーシング、
        アラートの体系的な仕組みが未構築
      severity: "High"
      effort_days: 20
      business_impact: "High"
      priority_score: 3.8
      remediation:
        approach: "フルスタック可観測性の導入"
        components:
          - "Datadog APM"
          - "Prometheus + Grafana"
          - "Jaeger for tracing"
          - "PagerDuty for alerts"
        timeline: "2-3 sprints"

    debt_3_manual_deployments:
      category: "インフラ"
      description: |
        デプロイプロセスの一部が手動、
        GitOps完全自動化が未達成
      severity: "Medium"
      effort_days: 15
      business_impact: "Medium"
      priority_score: 3.0
      remediation:
        approach: "GitOps完全自動化（Doc-DO-003参照）"
        components:
          - "ArgoCD導入"
          - "Argo Rollouts"
          - "自動ロールバック"
        timeline: "See Doc-DO-003"

    debt_4_no_disaster_recovery:
      category: "インフラ"
      description: |
        災害復旧計画が未策定、
        バックアップ/リストア手順が未整備
      severity: "High"
      effort_days: 20
      business_impact: "Critical"
      priority_score: 4.2
      remediation:
        approach: "DR計画の策定と実装"
        components:
          - "RPO/RTO定義"
          - "自動バックアップ設定"
          - "DR環境構築"
          - "定期DRテスト"
        timeline: "3 sprints"

  infrastructure_debt_summary:
    total_debts: 4
    total_effort_days: 85
    priority_distribution:
      critical: 2
      high: 1
      medium: 1
```

### 5.4 テストカバレッジのギャップ

```yaml
test_coverage_gap_analysis:
  current_state:
    test_files: 46
    flutter_tests:
      unit_tests: "約60%カバレッジ"
      widget_tests: "約40%カバレッジ"
      integration_tests: "約20%カバレッジ"
    backend_tests:
      unit_tests: "約50%カバレッジ"
      api_tests: "約30%カバレッジ"

  identified_gaps:
    gap_1_critical_path_coverage:
      area: "決済・予約フロー"
      current_coverage: "30%"
      target_coverage: "95%"
      risk: "High - 収益に直結"
      effort_days: 15

    gap_2_error_scenarios:
      area: "エラーハンドリング"
      current_coverage: "20%"
      target_coverage: "80%"
      risk: "Medium - ユーザー体験に影響"
      effort_days: 10

    gap_3_edge_cases:
      area: "境界値テスト"
      current_coverage: "25%"
      target_coverage: "70%"
      risk: "Medium - バグの温床"
      effort_days: 8

    gap_4_performance_tests:
      area: "パフォーマンステスト"
      current_coverage: "0%"
      target_coverage: "主要APIの100%"
      risk: "High - スケーリング時の問題"
      effort_days: 12

    gap_5_security_tests:
      area: "セキュリティテスト"
      current_coverage: "10%"
      target_coverage: "OWASP Top 10の100%"
      risk: "Critical - セキュリティリスク"
      effort_days: 15

  test_debt_summary:
    total_gaps: 5
    total_effort_days: 60
    priority_distribution:
      critical: 1
      high: 2
      medium: 2
```

### 5.5 負債サマリー

```yaml
total_debt_summary:
  by_category:
    flutter_app:
      total_effort_days: 98
      debts: 5
      highest_priority: "エラーハンドリング統一"

    backend:
      total_effort_days: 48
      debts: 5
      highest_priority: "レート制限導入"

    infrastructure:
      total_effort_days: 85
      debts: 4
      highest_priority: "本番インフラ構築"

    testing:
      total_effort_days: 60
      debts: 5
      highest_priority: "決済フローテスト"

  grand_total:
    total_effort_days: 291
    total_debt_items: 19

  priority_ranking:
    immediate:
      - "本番インフラ構築（85日）"
      - "レート制限導入（8日）"
      - "セキュリティテスト（15日）"

    short_term:
      - "エラーハンドリング統一（15日）"
      - "可観測性導入（20日）"
      - "決済フローテスト（15日）"
      - "DR計画策定（20日）"

    medium_term:
      - "状態管理統一（40日）"
      - "レガシー画面移行（10日）"
      - "テストカバレッジ向上（継続）"

    long_term:
      - "HTTPクライアント統一（8日）"
      - "設定外部化（5日）"
```

---

## 6. 負債返済戦略

### 6.1 20%ルールの実践

```yaml
twenty_percent_rule:
  principle:
    description: |
      スプリントキャパシティの20%を技術的負債の返済に
      恒常的に割り当てる

  implementation:
    sprint_allocation:
      total_capacity: "100%"
      feature_development: "60%"
      bug_fixes: "15%"
      technical_debt: "20%"
      buffer: "5%"

    calculation_example:
      team_size: "5 engineers"
      sprint_length: "2 weeks"
      working_days: "10 days × 5 = 50 person-days"
      debt_allocation: "50 × 0.20 = 10 person-days/sprint"
      annual_debt_capacity: "10 × 26 sprints = 260 person-days"

    debt_backlog_management:
      grooming:
        frequency: "月次"
        activities:
          - "新規負債アイテムの追加"
          - "優先順位の見直し"
          - "見積もりの更新"

      selection_criteria:
        - "優先度スコア"
        - "依存関係"
        - "チームスキル"
        - "現在のスプリント目標との整合性"

      tracking:
        tool: "Jira - Tech Debt エピック"
        labels: "tech-debt, priority-high/medium/low"
        metrics:
          - "スプリントごとの消化ポイント"
          - "残存負債量"
          - "新規負債発生量"

  guardrails:
    minimum_allocation: "15%（緊急時でも確保）"
    maximum_allocation: "30%（大規模返済フェーズ）"
    escalation: "連続3スプリントで達成できない場合、経営層へエスカレーション"
```

### 6.2 大規模リファクタリング計画

```yaml
major_refactoring_plan:
  approach: "計画的かつ段階的なリファクタリング"

  refactoring_projects:
    project_1_state_management:
      name: "状態管理のRiverpod統一"
      scope: "Flutter全体"
      total_effort: "40人日"
      duration: "9ヶ月"
      phases:
        phase_1:
          name: "新規機能はRiverpod"
          duration: "即時開始"
          effort: "追加コストなし"
          deliverables:
            - "Riverpodコーディングガイドライン"
            - "テンプレート/スニペット"

        phase_2:
          name: "共通プロバイダーの移行"
          duration: "3ヶ月"
          effort: "15人日"
          scope:
            - "言語プロバイダー"
            - "通貨プロバイダー"
            - "カートプロバイダー"
          approach: "1プロバイダーずつ移行"

        phase_3:
          name: "フィーチャー別プロバイダーの移行"
          duration: "6ヶ月"
          effort: "25人日"
          scope: "各フィーチャーモジュール内のProvider"
          approach: "フィーチャー単位で移行"

      success_criteria:
        - "Providerパッケージの依存削除"
        - "全テストがパス"
        - "パフォーマンス劣化なし"

    project_2_legacy_screen_migration:
      name: "レガシー画面の移行"
      scope: "/lib/screens/ → /lib/features/"
      total_effort: "10人日"
      duration: "2-3スプリント"
      approach: "Strangler Fig Pattern"
      steps:
        - step: 1
          action: "新フィーチャーモジュール作成"
          effort: "2日/画面"
        - step: 2
          action: "ルーティング更新"
          effort: "1日"
        - step: 3
          action: "旧画面削除"
          effort: "0.5日"
        - step: 4
          action: "テスト更新"
          effort: "1日"

    project_3_error_handling:
      name: "統一エラーハンドリングフレームワーク"
      scope: "フロントエンド＆バックエンド"
      total_effort: "15人日"
      duration: "2スプリント"
      deliverables:
        frontend:
          - "ErrorType enum定義"
          - "ErrorHandlerService"
          - "ErrorDisplayWidget"
          - "ErrorBoundary"
        backend:
          - "AppError クラス階層"
          - "ErrorMiddleware"
          - "統一エラーレスポンス形式"
```

### 6.3 Strangler Fig Patternでのマイグレーション

```yaml
strangler_fig_pattern:
  description: |
    レガシーシステムを段階的に新システムに置き換える手法。
    新機能は新システムで実装し、既存機能を徐々に移行する。

  application_in_triptrip:
    legacy_screens_migration:
      current_state:
        legacy_location: "/lib/screens/"
        target_location: "/lib/features/"
        affected_screens:
          - "rental_screen.dart"
          - "user_screen.dart"

      migration_approach:
        step_1_create_facade:
          description: "ルーターをファサードとして活用"
          implementation: |
            // router.dart
            GoRoute(
              path: '/rental',
              builder: (context, state) {
                // Feature flag or gradual migration
                if (FeatureFlags.useNewRentalScreen) {
                  return const RentalFeatureScreen();
                }
                return const LegacyRentalScreen();
              },
            ),

        step_2_create_new_module:
          description: "新フィーチャーモジュールの作成"
          structure: |
            features/rental/
            ├── rental_feature_screen.dart
            ├── widgets/
            │   ├── rental_item_card.dart
            │   └── rental_filter_widget.dart
            ├── providers/
            │   └── rental_provider.dart
            ├── models/
            │   └── rental_item.dart
            └── services/
                └── rental_service.dart

        step_3_parallel_testing:
          description: "並行テスト期間"
          duration: "1スプリント"
          activities:
            - "新旧画面の機能比較テスト"
            - "パフォーマンス比較"
            - "ユーザー受入テスト"

        step_4_traffic_shift:
          description: "トラフィックの段階的移行"
          stages:
            - "5%のユーザーを新画面へ（1週間）"
            - "25%のユーザーを新画面へ（1週間）"
            - "100%のユーザーを新画面へ"

        step_5_legacy_removal:
          description: "レガシーコードの削除"
          checklist:
            - "[ ] 全トラフィックが新画面を使用"
            - "[ ] エラー率が正常"
            - "[ ] レガシーコードの参照がゼロ"
            - "[ ] レガシーファイルの削除"
            - "[ ] テストの更新"

    http_client_migration:
      description: "http → dio への移行"
      approach: "Adapter Pattern + Strangler Fig"
      steps:
        - step: 1
          action: "DioAdapterの作成"
          code: |
            // http_client_adapter.dart
            abstract class HttpClientAdapter {
              Future<Response> get(String url);
              Future<Response> post(String url, {dynamic body});
              // ...
            }

            class DioAdapter implements HttpClientAdapter {
              final Dio _dio;
              // Implementation
            }

        - step: 2
          action: "既存サービスのAdapter使用への移行"
          approach: "1サービスずつ移行"

        - step: 3
          action: "httpパッケージの削除"
          timing: "全サービス移行完了後"
```

### 6.4 継続的リファクタリングの文化

```yaml
continuous_refactoring_culture:
  principles:
    boy_scout_rule:
      description: "コードを触ったら、必ず少しきれいにして離れる"
      implementation:
        - "PRでの軽微なリファクタリングを奨励"
        - "「リファクタリングコミット」の分離"
        - "コードオーナーによるレビュー"

    red_green_refactor:
      description: "TDDサイクルでのリファクタリング"
      cycle:
        - "Red: 失敗するテストを書く"
        - "Green: テストを通す最小限のコード"
        - "Refactor: コードを改善"

  practices:
    refactoring_fridays:
      description: "金曜日午後をリファクタリング時間に"
      frequency: "隔週"
      activities:
        - "技術的負債バックログの消化"
        - "コードレビューで見つけた改善"
        - "ペアプログラミングでのリファクタリング"

    tech_debt_budgeting:
      description: "PRごとの負債予算"
      rules:
        - "新規負債を追加する場合は、同等以上の負債を返済"
        - "負債追加には Tech Lead の承認が必要"
        - "負債追加理由のドキュメント化"

    knowledge_sharing:
      description: "リファクタリング知識の共有"
      activities:
        - "週次Tech Talk（15分）"
        - "リファクタリングパターンの共有"
        - "事例研究の発表"

  metrics_tracking:
    velocity_impact:
      measure: "負債返済前後のベロシティ比較"
      target: "負債返済による速度向上の可視化"

    developer_satisfaction:
      measure: "四半期アンケート"
      questions:
        - "コードベースの作業しやすさ"
        - "技術的負債の改善実感"
        - "リファクタリング時間の十分さ"
```

---

## 7. 負債防止メカニズム

### 7.1 ガバナンスとレビュー

```yaml
governance_and_review:
  architecture_review_board:
    description: "アーキテクチャ決定の審査機関"
    composition:
      - "CTO (Chair)"
      - "Tech Leads (各チーム)"
      - "Senior Engineers (ローテーション)"

    responsibilities:
      - "重要なアーキテクチャ決定のレビュー"
      - "技術標準の策定"
      - "負債蓄積の防止"

    triggers:
      - "新しいフレームワーク/ライブラリの導入"
      - "大規模なアーキテクチャ変更"
      - "技術標準からの逸脱"

    process:
      - "ADR（Architecture Decision Record）の提出"
      - "レビュー会議（週次）"
      - "承認/修正/却下の決定"

  code_review_standards:
    mandatory_checks:
      - "コーディング規約の遵守"
      - "テストカバレッジ（新規コードは80%以上）"
      - "セキュリティチェック"
      - "パフォーマンス考慮"

    debt_prevention_checks:
      - "既存パターンとの整合性"
      - "コード重複の回避"
      - "適切な抽象化レベル"
      - "ドキュメントの更新"

    reviewer_guidelines:
      - "「動く」だけでなく「保守可能」かを評価"
      - "負債追加の明示的な承認が必要"
      - "リファクタリング提案を歓迎"

  design_review:
    when:
      - "新機能の設計段階"
      - "大規模な変更の前"
      - "パフォーマンスクリティカルな実装"

    participants:
      - "担当エンジニア"
      - "Tech Lead"
      - "関連チームのエンジニア"

    output:
      - "設計ドキュメント"
      - "ADR（必要な場合）"
      - "負債リスクの評価"
```

### 7.2 自動化と可視化

```yaml
automation_and_visibility:
  static_analysis:
    tools:
      flutter:
        - tool: "dart analyze"
          purpose: "静的解析"
          ci_integration: "PR blocking"
        - tool: "flutter_lints"
          purpose: "Lintルール"
          config: "analysis_options.yaml"
        - tool: "DCM (Dart Code Metrics)"
          purpose: "コードメトリクス"
          metrics:
            - "Cyclomatic Complexity"
            - "Lines of Code"
            - "Technical Debt"

      backend:
        - tool: "ESLint"
          purpose: "Linting"
          ci_integration: "PR blocking"
        - tool: "SonarQube"
          purpose: "品質ゲート"
          metrics:
            - "Code Smells"
            - "Technical Debt Ratio"
            - "Duplications"

    ci_integration:
      quality_gates:
        - "新規コードのカバレッジ > 80%"
        - "Code Smells 増加なし"
        - "セキュリティ脆弱性なし"
        - "重複コード増加なし"

  debt_dashboard:
    description: "技術的負債の可視化ダッシュボード"
    components:
      overall_health:
        - "総負債量（人日）"
        - "負債トレンド（増減）"
        - "返済進捗"

      category_breakdown:
        - "コード負債"
        - "アーキテクチャ負債"
        - "インフラ負債"
        - "テスト負債"

      priority_view:
        - "緊急対応必要"
        - "高優先度"
        - "中優先度"
        - "低優先度"

      team_view:
        - "チーム別の負債所有"
        - "チーム別の返済進捗"

    tools:
      - "Grafana dashboard（SonarQube連携）"
      - "Jira dashboard（負債バックログ）"
      - "Custom metrics collector"

    visibility:
      - "エンジニアリング会議で週次レビュー"
      - "経営層への月次レポート"
      - "オフィスモニターでの常時表示"

  dependency_monitoring:
    tools:
      - tool: "Dependabot"
        purpose: "依存関係の自動更新PR"
      - tool: "Snyk"
        purpose: "脆弱性スキャン"
      - tool: "pub outdated / npm outdated"
        purpose: "古い依存関係の検出"

    process:
      - "週次の依存関係レビュー"
      - "セキュリティアップデートは即時対応"
      - "メジャーアップデートは計画的に"
```

### 7.3 Definition of Doneへの負債チェック

```yaml
definition_of_done:
  standard_dod:
    code_quality:
      - "[ ] コーディング規約に準拠"
      - "[ ] 静的解析エラーなし"
      - "[ ] セキュリティスキャンパス"

    testing:
      - "[ ] ユニットテスト追加（カバレッジ80%以上）"
      - "[ ] 統合テスト追加（該当する場合）"
      - "[ ] 手動テスト完了"

    documentation:
      - "[ ] コードコメント（複雑なロジック）"
      - "[ ] API ドキュメント更新"
      - "[ ] README 更新（該当する場合）"

    review:
      - "[ ] コードレビュー完了"
      - "[ ] Tech Lead 承認（該当する場合）"

  debt_specific_dod:
    debt_assessment:
      - "[ ] 新規負債の有無を評価"
      - "[ ] 負債追加の場合、理由を記載"
      - "[ ] 負債追加の場合、返済計画を作成"

    debt_prevention:
      - "[ ] 既存パターンとの整合性確認"
      - "[ ] 重複コードの回避確認"
      - "[ ] Boy Scout Rule 適用（触った箇所の改善）"

    debt_payback:
      - "[ ] 関連する負債の返済を検討"
      - "[ ] 負債バックログの更新"

  pr_template:
    template: |
      ## Summary
      [変更内容の説明]

      ## Type of Change
      - [ ] New feature
      - [ ] Bug fix
      - [ ] Refactoring
      - [ ] Technical debt payoff

      ## Technical Debt Assessment
      - [ ] No new technical debt introduced
      - [ ] Technical debt added (explain below)
      - [ ] Technical debt reduced

      ### If debt added:
      - Reason:
      - Estimated effort to fix:
      - Planned fix date:

      ## Testing
      - [ ] Unit tests added/updated
      - [ ] Integration tests added/updated
      - [ ] Manual testing completed

      ## Checklist
      - [ ] Code follows style guidelines
      - [ ] Documentation updated
      - [ ] No security vulnerabilities introduced
```

---

## 8. 負債管理ロードマップ

### 8.1 Phase 1: 緊急対応（0-6ヶ月）

```yaml
phase_1_emergency:
  timeline: "Month 1-6"
  theme: "重大負債の特定と緊急対応"

  objectives:
    - "クリティカルな負債の即時対応"
    - "負債可視化の仕組み構築"
    - "20%ルールの確立"

  key_initiatives:
    initiative_1_critical_debt:
      name: "クリティカル負債の対応"
      items:
        - task: "レート制限導入"
          effort: "8日"
          sprint: "1"
        - task: "セキュリティテスト追加"
          effort: "15日"
          sprint: "1-2"
        - task: "決済フローテスト強化"
          effort: "15日"
          sprint: "2-3"

    initiative_2_visibility:
      name: "負債可視化システム構築"
      items:
        - task: "SonarQube導入"
          effort: "5日"
          sprint: "1"
        - task: "負債ダッシュボード作成"
          effort: "5日"
          sprint: "2"
        - task: "CI/CD品質ゲート設定"
          effort: "3日"
          sprint: "2"

    initiative_3_process:
      name: "負債管理プロセス確立"
      items:
        - task: "20%ルール導入"
          effort: "プロセス変更"
          sprint: "1"
        - task: "DoD更新"
          effort: "1日"
          sprint: "1"
        - task: "PRテンプレート更新"
          effort: "1日"
          sprint: "1"

  success_criteria:
    - "クリティカル負債ゼロ"
    - "負債ダッシュボード稼働"
    - "全スプリントで20%負債対応を達成"

  deliverables:
    - "レート制限ミドルウェア"
    - "セキュリティテストスイート"
    - "SonarQubeレポート"
    - "負債可視化ダッシュボード"
    - "更新されたDoD/PRテンプレート"
```

### 8.2 Phase 2: 計画的返済（6-18ヶ月）

```yaml
phase_2_planned_payback:
  timeline: "Month 7-18"
  theme: "計画的な負債返済と予防強化"

  objectives:
    - "高優先度負債の計画的返済"
    - "大規模リファクタリングの実行"
    - "負債発生率の削減"

  key_initiatives:
    initiative_1_state_management:
      name: "状態管理統一（Riverpod）"
      total_effort: "40日"
      phases:
        - phase: "共通プロバイダー移行"
          sprint: "14-19"
          effort: "15日"
        - phase: "フィーチャー別移行"
          sprint: "20-32"
          effort: "25日"

    initiative_2_observability:
      name: "可観測性強化"
      total_effort: "32日"
      items:
        - task: "APM導入（Datadog）"
          sprint: "14-16"
          effort: "12日"
        - task: "分散トレーシング"
          sprint: "17-18"
          effort: "8日"
        - task: "構造化ロギング"
          sprint: "19-20"
          effort: "12日"

    initiative_3_testing:
      name: "テストカバレッジ向上"
      approach: "継続的（各スプリント20%配分）"
      targets:
        unit_tests: "60% → 80%"
        integration_tests: "30% → 60%"
        e2e_tests: "20% → 50%"

    initiative_4_legacy_migration:
      name: "レガシー画面移行"
      total_effort: "10日"
      sprint: "20-22"

    initiative_5_error_handling:
      name: "エラーハンドリング統一"
      total_effort: "15日"
      sprint: "15-17"

  success_criteria:
    - "高優先度負債50%削減"
    - "テストカバレッジ目標達成"
    - "新規負債発生率30%削減"
    - "開発者満足度向上"

  quarterly_milestones:
    q3_y1:
      - "状態管理移行Phase 2開始"
      - "APM稼働"
      - "エラーハンドリング統一完了"

    q4_y1:
      - "状態管理移行Phase 2完了"
      - "分散トレーシング稼働"
      - "レガシー画面移行完了"

    q1_y2:
      - "状態管理移行Phase 3進行中"
      - "テストカバレッジ目標70%達成"

    q2_y2:
      - "状態管理統一完了"
      - "テストカバレッジ目標達成"
      - "Phase 2完了"
```

### 8.3 Phase 3: 持続可能体制（18-36ヶ月）

```yaml
phase_3_sustainable:
  timeline: "Month 19-36"
  theme: "持続可能な負債管理体制の確立"

  objectives:
    - "負債発生を最小化する文化の定着"
    - "自動化による予防の強化"
    - "継続的な改善サイクルの確立"

  key_initiatives:
    initiative_1_automation:
      name: "負債検出・防止の自動化"
      items:
        - task: "AIコードレビュー導入"
          description: "GitHub Copilot / Amazon CodeGuru"
          effort: "10日"
        - task: "自動リファクタリング提案"
          description: "IDE統合ツール"
          effort: "5日"
        - task: "負債予測アラート"
          description: "ML駆動の負債予測"
          effort: "20日"

    initiative_2_culture:
      name: "継続的改善文化の定着"
      items:
        - task: "リファクタリング認定制度"
          description: "優秀なリファクタリング貢献者の表彰"
        - task: "技術的負債ゼロの日"
          description: "四半期に1日、全員で負債対応"
        - task: "内部勉強会の定例化"
          description: "月次のベストプラクティス共有"

    initiative_3_metrics:
      name: "メトリクス駆動の改善"
      items:
        - task: "負債KPIの経営指標化"
          description: "経営ダッシュボードへの組み込み"
        - task: "チーム別負債スコアカード"
          description: "四半期レビューの標準項目に"
        - task: "技術負債ROI分析"
          description: "負債返済の投資対効果測定"

    initiative_4_prevention:
      name: "先進的な予防策"
      items:
        - task: "アーキテクチャフィットネス関数"
          description: "自動アーキテクチャ検証"
          effort: "15日"
        - task: "技術レーダー"
          description: "技術選定の標準化"
        - task: "負債バジェット制度"
          description: "チーム別の負債許容量管理"

  success_criteria:
    - "技術的負債の年間増加率 < 5%"
    - "新規負債発生率50%削減（Phase 1比）"
    - "開発者満足度4.5/5.0以上"
    - "負債起因のインシデントゼロ"

  long_term_vision:
    description: |
      Phase 3完了時点で、TripTripは技術的負債を
      「問題」ではなく「管理可能な資産」として
      扱える成熟した組織になることを目指します。

    characteristics:
      - "負債は可視化され、定量的に管理されている"
      - "返済は計画的かつ継続的に行われている"
      - "新規負債は意図的かつ承認済みのみ"
      - "チーム文化として負債管理が定着"
      - "技術基盤は常に健全な状態を維持"
```

---

## 9. KPIと成功指標

### 9.1 主要KPI

```yaml
key_performance_indicators:
  debt_volume:
    name: "技術的負債量"
    unit: "人日"
    current: "291人日"
    targets:
      year_1: "200人日（-31%）"
      year_2: "140人日（-52%）"
      year_3: "100人日（-66%）"
    measurement: "月次SonarQubeスキャン + 手動評価"

  debt_ratio:
    name: "技術的負債比率"
    formula: "負債修正時間 / 総開発時間"
    current: "15%"
    targets:
      year_1: "< 12%"
      year_2: "< 8%"
      year_3: "< 5%"
    measurement: "SonarQube Technical Debt Ratio"

  new_debt_rate:
    name: "新規負債発生率"
    formula: "月間新規負債量 / 月間コード変更量"
    current: "測定開始"
    targets:
      year_1: "ベースライン確立"
      year_2: "ベースライン比-30%"
      year_3: "ベースライン比-50%"
    measurement: "PR分析 + SonarQube差分"

  payback_velocity:
    name: "負債返済速度"
    unit: "人日/スプリント"
    current: "0（未開始）"
    targets:
      year_1: "10人日/スプリント"
      year_2: "10人日/スプリント（維持）"
      year_3: "8人日/スプリント（削減）"
    measurement: "Jira Tech Debt エピックの完了ポイント"

  code_quality_score:
    name: "コード品質スコア"
    scale: "A-F"
    current: "C"
    targets:
      year_1: "B"
      year_2: "A"
      year_3: "A（維持）"
    measurement: "Code Climate / SonarQube"
```

### 9.2 運用メトリクス

```yaml
operational_metrics:
  development_velocity:
    name: "開発ベロシティ"
    measurement: "スプリントあたりのストーリーポイント"
    expected_impact: "負債返済により+15%向上"
    tracking: "Jira Velocity Chart"

  cycle_time:
    name: "サイクルタイム"
    measurement: "コミットから本番デプロイまでの時間"
    expected_impact: "負債返済により-20%短縮"
    tracking: "GitHub Insights"

  bug_rate:
    name: "バグ発生率"
    measurement: "新機能あたりのバグ数"
    expected_impact: "負債返済により-30%削減"
    tracking: "Jira Bug Tracking"

  test_coverage:
    name: "テストカバレッジ"
    targets:
      unit: "80%"
      integration: "60%"
      e2e: "50%"
    tracking: "CI/CD Coverage Reports"

  code_review_time:
    name: "コードレビュー時間"
    expected_impact: "コード品質向上により-25%短縮"
    tracking: "GitHub PR Metrics"
```

### 9.3 組織メトリクス

```yaml
organizational_metrics:
  developer_satisfaction:
    name: "開発者満足度"
    measurement: "四半期アンケート（1-5スケール）"
    questions:
      - "コードベースの作業しやすさ"
      - "技術的負債のストレス"
      - "リファクタリング時間の十分さ"
    target: "4.5/5.0"

  onboarding_time:
    name: "オンボーディング時間"
    measurement: "新メンバーの生産性到達時間"
    current: "3ヶ月"
    target: "6週間"

  knowledge_distribution:
    name: "知識分散度"
    measurement: "コードオーナーシップの分散"
    target: "各モジュールに3人以上のコントリビューター"

  debt_awareness:
    name: "負債認知度"
    measurement: "チームメンバーの負債理解度"
    target: "全員が負債バックログを理解・参照"
```

---

## 10. 文書間参照 & 統合ポイント

### 10.1 関連文書

```yaml
document_references:
  upstream_documents:
    - doc_id: "Doc-TR-001"
      title: "技術進化ロードマップ"
      relationship: "負債管理は技術進化の前提条件"
      key_dependencies:
        - "技術スタック進化計画"
        - "フレームワークアップグレード"

    - doc_id: "Doc-TR-002"
      title: "プラットフォームマイグレーション戦略"
      relationship: "マイグレーション時の負債対応"
      key_dependencies:
        - "移行前の負債解消"
        - "移行後の負債防止"

  downstream_documents:
    - doc_id: "Doc-DM-002"
      title: "DevOps＆継続的デリバリー"
      relationship: "CI/CDでの品質ゲート"
      provides:
        - "自動品質チェック"
        - "負債検出の自動化"

    - doc_id: "Doc-QA-001"
      title: "品質保証戦略"
      relationship: "テスト負債の管理"
      provides:
        - "テストカバレッジ目標"
        - "テスト戦略"

  cross_references:
    - doc_id: "Doc-DO-003"
      title: "Deployment Strategies"
      relationship: "デプロイメントにおける負債考慮"

    - doc_id: "Doc-IA-002"
      title: "Kubernetes & コンテナ戦略"
      relationship: "インフラ負債の管理"

    - doc_id: "Doc-SC-001"
      title: "セキュリティアーキテクチャ"
      relationship: "セキュリティ関連の負債"
```

### 10.2 統合ポイント

```yaml
integration_points:
  sprint_planning:
    integration: "スプリント計画での20%負債配分"
    responsible: "Scrum Master / Tech Lead"
    frequency: "各スプリント開始時"

  code_review:
    integration: "PRでの負債チェック"
    responsible: "全エンジニア"
    frequency: "各PR"

  architecture_review:
    integration: "設計レビューでの負債評価"
    responsible: "Architecture Review Board"
    frequency: "重要な設計変更時"

  ci_cd_pipeline:
    integration: "品質ゲートでの自動チェック"
    responsible: "DevOps Team"
    frequency: "各ビルド"

  monthly_review:
    integration: "月次負債レビュー会議"
    responsible: "Engineering Leadership"
    frequency: "月次"

  quarterly_planning:
    integration: "四半期の負債返済計画"
    responsible: "CTO / Tech Leads"
    frequency: "四半期"
```

---

## 付録

### A. 負債評価テンプレート

```yaml
debt_assessment_template:
  identification:
    id: "[DEBT-XXX]"
    title: "[負債の簡潔な説明]"
    category: "[コード/アーキテクチャ/インフラ/テスト]"
    location: "[ファイルパス/モジュール]"
    discovered_date: "[日付]"
    discovered_by: "[発見者]"

  description:
    summary: "[負債の詳細説明]"
    symptoms:
      - "[症状1]"
      - "[症状2]"
    root_cause: "[根本原因]"

  assessment:
    severity: "[Critical/High/Medium/Low]"
    effort_days: "[推定工数（人日）]"
    business_impact: "[ビジネスへの影響]"
    risk_level: "[リスクレベル]"
    frequency: "[発生頻度/影響範囲]"
    priority_score: "[1.0-5.0]"

  remediation:
    approach: "[修正アプローチ]"
    steps:
      - "[ステップ1]"
      - "[ステップ2]"
    timeline: "[予定期間]"
    dependencies: "[依存関係]"

  tracking:
    status: "[New/In Progress/Resolved/Deferred]"
    assigned_to: "[担当者]"
    sprint: "[対応スプリント]"
    resolution_date: "[解決日]"
    notes: "[備考]"
```

### B. 用語集

| 用語 | 定義 |
|------|------|
| Technical Debt | 技術的負債。短期的な利便性のために蓄積された技術的課題 |
| SQALE | Software Quality Assessment based on Lifecycle Expectations |
| Cyclomatic Complexity | 循環的複雑度。コードの複雑さの指標 |
| Code Smell | コードの臭い。問題の兆候を示すコードパターン |
| Strangler Fig Pattern | 段階的にレガシーを置き換える移行パターン |
| Boy Scout Rule | 「来たときよりも美しく」の原則 |
| ADR | Architecture Decision Record |

### C. ツールリスト

| ツール | 用途 | URL |
|--------|------|-----|
| SonarQube | コード品質分析 | sonarqube.org |
| Code Climate | 品質メトリクス | codeclimate.com |
| Dart Code Metrics | Flutter品質分析 | pub.dev/packages/dart_code_metrics |
| ESLint | JavaScript/TypeScript Linting | eslint.org |
| Dependabot | 依存関係管理 | github.com/dependabot |
| Snyk | セキュリティスキャン | snyk.io |

---

**文書情報**
- Document ID: Doc-TR-003
- Version: 1.0.0
- Last Updated: 2026-01-20
- Status: Active
- Total Lines: 1,500+
- Author: Technical Architecture Team

**関連文書**
- Doc-TR-001: 技術進化ロードマップ
- Doc-TR-002: プラットフォームマイグレーション戦略
- Doc-DM-002: DevOps＆継続的デリバリー
- Doc-QA-001: 品質保証戦略
- Doc-DO-003: Deployment Strategies

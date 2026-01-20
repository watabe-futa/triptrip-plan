# Doc-DM-001: TripTripアジャイルデリバリーフレームワーク

## エグゼクティブサマリー

本文書は、TripTripプラットフォーム開発のためのアジャイルデリバリーフレームワークを定義します。Spotify、Google、Amazonのアジャイル実践を統合し、2週間スプリントを基本とした開発プロセス、ユーザーストーリーの作成と見積もり方法論、Definition of Done（DoD）と受入基準、バックログ管理と優先順位付けフレームワーク、チームベロシティ追跡と予測手法を包括的に策定します。15名から150名へのチームスケーリングに対応し、Spotifyモデル（スクワッド、チャプター、トライブ、ギルド）の段階的適用を通じて、高い開発生産性と品質を維持しながら、顧客価値を継続的に届けることを目指します。

---

## 第1章 はじめに & コンテキスト

### 1.1 目的 & スコープ

#### 1.1.1 文書の目的

本アジャイルデリバリーフレームワーク文書は、TripTripの開発組織全体で一貫したアジャイルプラクティスを実現するための包括的なガイドラインを提供します。

**主要目的**:

1. **開発プロセスの標準化**: チーム間で一貫した開発プロセスを確立
2. **品質基準の明確化**: Definition of Doneと受入基準の統一
3. **生産性の可視化**: ベロシティ追跡と予測の仕組み構築
4. **継続的改善の促進**: レトロスペクティブを通じた組織学習
5. **スケーラビリティの確保**: チーム拡大に対応したフレームワーク

**スコープ定義**:

```
┌─────────────────────────────────────────────────────────────┐
│             アジャイルフレームワークスコープ                 │
├─────────────────────────────────────────────────────────────┤
│ インスコープ:                                                │
│  ├─ スクラムフレームワークの定義                            │
│  ├─ スプリント構造とセレモニー                              │
│  ├─ ユーザーストーリーと見積もり                            │
│  ├─ バックログ管理                                          │
│  ├─ Definition of Done                                      │
│  ├─ メトリクスと継続的改善                                  │
│  └─ スケーリングフレームワーク（Spotify Model）             │
├─────────────────────────────────────────────────────────────┤
│ アウトオブスコープ:                                          │
│  ├─ 詳細な技術プラクティス（→Doc-QA-001参照）              │
│  ├─ DevOps/CI/CDプロセス（→Doc-DO-001参照）                │
│  ├─ 人員計画詳細（→Doc-IR-002参照）                        │
│  └─ プロジェクト管理ツール設定詳細                          │
└─────────────────────────────────────────────────────────────┘
```

#### 1.1.2 対象読者

- エンジニアリングチーム全員
- プロダクトマネージャー
- スクラムマスター / アジャイルコーチ
- エンジニアリングマネージャー
- 経営層（CTO、VP Engineering）

### 1.2 アジャイル原則

#### 1.2.1 TripTripアジャイルマニフェスト

**コアバリュー**:

```
┌─────────────────────────────────────────────────────────────┐
│              TripTrip アジャイルマニフェスト                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  私たちは以下を重視します:                                  │
│                                                             │
│  1. プロセスやツールよりも「個人と対話」                   │
│     ├─ ペアプログラミング、モブプログラミング             │
│     ├─ 対面/同期コミュニケーション優先                    │
│     └─ 心理的安全性の確保                                 │
│                                                             │
│  2. 包括的なドキュメントよりも「動くソフトウェア」         │
│     ├─ 頻繁なデリバリー（日次デプロイ）                   │
│     ├─ 継続的インテグレーション                           │
│     └─ プロダクションでの検証                             │
│                                                             │
│  3. 契約交渉よりも「顧客との協調」                         │
│     ├─ ユーザーフィードバックの積極的収集                 │
│     ├─ ステークホルダーとの定期的対話                     │
│     └─ 顧客価値の最優先                                   │
│                                                             │
│  4. 計画に従うことよりも「変化への対応」                   │
│     ├─ 短いフィードバックループ                           │
│     ├─ 柔軟なスコープ調整                                 │
│     └─ 学習に基づく適応                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 1.2.2 フレームワーク選定理由

**スクラム + カンバン ハイブリッドアプローチ**:

```yaml
framework_selection:
  primary: Scrum
  rationale:
    - 明確なスプリントサイクルによる計画性
    - 定期的なセレモニーによるチームの同期
    - 役割の明確化による責任分担
    - 業界での広い採用実績

  secondary: Kanban
  rationale:
    - 運用チームの継続的フロー管理
    - ボトルネックの可視化
    - WIP制限による品質確保
    - 緊急対応の柔軟性

  hybrid_approach:
    product_development: Scrum（2週間スプリント）
    operations: Kanban（継続的フロー）
    platform: Scrumban（スプリント + WIP制限）

  scaling_framework: Spotify Model
  rationale:
    - 自律性と整合性のバランス
    - 技術的な横断連携（チャプター）
    - 文化的適合性（日本の組織文化）
```

### 1.3 フレームワーク選定理由

#### 1.3.1 Spotifyモデル採用の背景

```
┌─────────────────────────────────────────────────────────────┐
│                  Spotifyモデル概要                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                     ┌─────────────┐                         │
│                     │   Tribe     │                         │
│                     │ (トライブ)  │                         │
│                     └──────┬──────┘                         │
│                            │                                │
│       ┌────────────────────┼────────────────────┐          │
│       ▼                    ▼                    ▼          │
│  ┌─────────┐         ┌─────────┐         ┌─────────┐      │
│  │  Squad  │         │  Squad  │         │  Squad  │      │
│  │スクワッド│         │スクワッド│         │スクワッド│      │
│  └─────────┘         └─────────┘         └─────────┘      │
│       │                    │                    │          │
│       └────────────────────┼────────────────────┘          │
│                            │                                │
│                     ┌──────┴──────┐                         │
│                     │  Chapter   │                         │
│                     │ (チャプター)│                         │
│                     │ 技術横断   │                         │
│                     └─────────────┘                         │
│                                                             │
│                     ┌─────────────┐                         │
│                     │   Guild    │                         │
│                     │ (ギルド)   │                         │
│                     │ 興味横断   │                         │
│                     └─────────────┘                         │
│                                                             │
│  Squad: 自律的な開発チーム（6-10名）                        │
│  Chapter: 同じ専門性を持つメンバーの集まり                  │
│  Tribe: 関連するSquadの集合（40-150名）                     │
│  Guild: 共通の興味を持つメンバーのコミュニティ              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 第2章 スクラムフレームワーク

### 2.1 スプリント構造とサイクル

#### 2.1.1 スプリント概要

```yaml
sprint_structure:
  duration: 2週間（10営業日）
  start_day: 月曜日
  end_day: 金曜日（翌週）

  capacity_planning:
    working_days: 10日
    meeting_overhead: 1日相当
    buffer: 0.5日
    available_capacity: 8.5日/人/スプリント

  sprint_phases:
    sprint_planning: Day 1（月曜AM）
    daily_execution: Day 1-9
    sprint_review: Day 10（金曜PM）
    sprint_retrospective: Day 10（金曜PM、Review後）
    refinement: Day 5（水曜PM、中間）

  release_cadence:
    staging: Daily（自動）
    production: Weekly（木曜）
    hotfix: As needed（即時）
```

#### 2.1.2 スプリントカレンダー

```
┌─────────────────────────────────────────────────────────────┐
│                  2週間スプリントカレンダー                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Week 1                                                     │
│  ┌─────┬─────┬─────┬─────┬─────┐                          │
│  │ Mon │ Tue │ Wed │ Thu │ Fri │                          │
│  ├─────┼─────┼─────┼─────┼─────┤                          │
│  │Plan │Daily│Daily│Daily│Daily│                          │
│  │ning │     │Refine│    │     │                          │
│  │     │     │ment │     │     │                          │
│  └─────┴─────┴─────┴─────┴─────┘                          │
│                                                             │
│  Week 2                                                     │
│  ┌─────┬─────┬─────┬─────┬─────┐                          │
│  │ Mon │ Tue │ Wed │ Thu │ Fri │                          │
│  ├─────┼─────┼─────┼─────┼─────┤                          │
│  │Daily│Daily│Daily│Daily│Review│                         │
│  │     │     │     │Release│Retro│                         │
│  │     │     │     │      │     │                          │
│  └─────┴─────┴─────┴─────┴─────┘                          │
│                                                             │
│  Daily Standup: 10:00-10:15 (毎日)                         │
│  Sprint Planning: 10:00-12:00 (Day 1)                      │
│  Backlog Refinement: 14:00-15:30 (Day 5)                   │
│  Sprint Review: 14:00-15:00 (Day 10)                       │
│  Sprint Retrospective: 15:30-16:30 (Day 10)                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 スクラムロールと責任

#### 2.2.1 プロダクトオーナー（PO）

```yaml
product_owner:
  responsibilities:
    primary:
      - プロダクトビジョンの策定と共有
      - プロダクトバックログの管理
      - ユーザーストーリーの作成と優先順位付け
      - 受入基準の定義
      - ステークホルダーとの連携

    secondary:
      - スプリントレビューでのデモ調整
      - リリース計画への参加
      - 市場・競合分析

  authority:
    - バックログの優先順位決定権
    - ユーザーストーリーの受入/拒否
    - スコープ調整の最終決定

  time_allocation:
    backlog_management: 30%
    stakeholder_engagement: 25%
    sprint_ceremonies: 15%
    user_research: 15%
    strategy_planning: 15%

  per_squad: 1名（複数スクワッド兼任可）
```

#### 2.2.2 スクラムマスター（SM）

```yaml
scrum_master:
  responsibilities:
    servant_leadership:
      - チームの障害除去
      - プロセス改善のファシリテーション
      - チーム間の調整
      - 組織へのアジャイル浸透支援

    process_facilitation:
      - セレモニーのファシリテーション
      - タイムボックスの管理
      - アジャイルプラクティスのコーチング
      - メトリクスの収集と可視化

    team_support:
      - チームの自己組織化支援
      - コンフリクト解決の支援
      - 心理的安全性の確保
      - チームの成長支援

  authority:
    - プロセスに関する助言
    - 障害のエスカレーション
    - セレモニーの時間管理

  time_allocation:
    facilitation: 25%
    coaching: 25%
    impediment_removal: 20%
    process_improvement: 15%
    metrics_reporting: 15%

  per_squad: 1名（2スクワッド兼任可、成熟したチーム）
```

#### 2.2.3 開発チーム

```yaml
development_team:
  composition:
    ideal_size: 6-8名
    minimum: 4名
    maximum: 10名

  skills:
    required:
      - フロントエンド開発
      - バックエンド開発
      - テスト作成
    recommended:
      - UI/UXデザイン
      - DevOps
      - データ分析

  responsibilities:
    - スプリント目標へのコミットメント
    - ユーザーストーリーの実装
    - 品質の担保（テスト、コードレビュー）
    - 見積もりの提供
    - 継続的改善への参加

  autonomy:
    - 作業の自己組織化
    - 技術的意思決定（ガードレール内）
    - タスクの分解と割り当て
    - デイリーの運営

  cross_functionality:
    goal: チーム内で完結した開発能力
    t_shaped_skills: 1つの深い専門性 + 幅広い基礎
```

### 2.3 スクラムセレモニー詳細

#### 2.3.1 スプリントプランニング

```yaml
sprint_planning:
  duration: 2時間（2週間スプリント）
  participants: PO, SM, 開発チーム全員

  agenda:
    part_1_what:
      duration: 60分
      activities:
        - スプリント目標の設定
        - プロダクトバックログのレビュー
        - ユーザーストーリーの選択
        - 受入基準の確認

    part_2_how:
      duration: 60分
      activities:
        - タスクへの分解
        - 技術的アプローチの議論
        - 依存関係の特定
        - キャパシティの確認

  outputs:
    - スプリント目標
    - スプリントバックログ
    - タスクボード初期状態
    - コミットメント

  facilitation_tips:
    - POによる優先ストーリーの事前共有
    - リファインメント完了済みストーリーのみ対象
    - 70-80%のキャパシティでコミット
    - 不確実性の高いストーリーは早期着手
```

#### 2.3.2 デイリースクラム

```yaml
daily_scrum:
  duration: 15分（最大）
  time: 10:00-10:15
  participants: 開発チーム（PO, SMはオプション）
  format: スタンドアップ（オフライン）/ オンライン

  structure:
    traditional_questions:
      - 昨日やったこと
      - 今日やること
      - 障害・ブロッカー

    alternative_walk_the_board:
      - 右から左へボードをレビュー
      - ブロックされたアイテムにフォーカス
      - フローの改善点を特定

  ground_rules:
    - 時間厳守（開始・終了）
    - 問題解決は別途
    - 全員が発言
    - スプリント目標への焦点

  remote_considerations:
    - カメラオン推奨
    - 非同期更新も許容（タイムゾーン差）
    - Slack/Teams連携
```

#### 2.3.3 スプリントレビュー

```yaml
sprint_review:
  duration: 1時間
  participants:
    required: PO, SM, 開発チーム
    invited: ステークホルダー、他チーム

  agenda:
    - スプリント目標の振り返り: 5分
    - 完了したストーリーのデモ: 35分
    - ステークホルダーフィードバック: 15分
    - 次スプリントの優先事項確認: 5分

  demo_guidelines:
    - 本番環境または本番相当で実施
    - ユーザー視点でのシナリオ
    - 技術的詳細は最小限
    - 質疑応答の時間確保

  outputs:
    - インクリメントの承認/フィードバック
    - バックログへの追加アイテム
    - 優先順位の調整
    - リリース判断

  metrics_review:
    - スプリント目標達成度
    - 完了ストーリーポイント
    - バーンダウンチャート
    - 品質メトリクス
```

#### 2.3.4 スプリントレトロスペクティブ

```yaml
sprint_retrospective:
  duration: 1時間
  participants: PO, SM, 開発チーム
  facilitator: SM（ローテーション可）

  formats:
    standard_start_stop_continue:
      - Start: 新しく始めること
      - Stop: やめること
      - Continue: 続けること

    mad_sad_glad:
      - Mad: 怒り・不満
      - Sad: 悲しみ・残念
      - Glad: 喜び・感謝

    sailboat:
      - Wind: 推進力
      - Anchor: 足かせ
      - Rocks: リスク
      - Island: 目標

    4ls:
      - Liked: 良かったこと
      - Learned: 学んだこと
      - Lacked: 足りなかったこと
      - Longed for: 望むこと

  ground_rules:
    - Vegas Rule: ここで話したことはここに留める
    - 非難ではなく改善にフォーカス
    - 全員の声を聴く
    - 具体的なアクションにつなげる

  outputs:
    - 改善アクションアイテム（最大3つ）
    - オーナーと期限
    - 前回アクションの振り返り
```

---

## 第3章 バックログ管理

### 3.1 ユーザーストーリー作成

#### 3.1.1 ユーザーストーリーフォーマット

```yaml
user_story_template:
  format: |
    As a [ユーザータイプ],
    I want [機能/行動],
    so that [価値/目的].

  example: |
    As a 旅行者,
    I want ホテルを日付と場所で検索できる,
    so that 旅行計画に合った宿泊先を見つけられる.

  components:
    title:
      - 簡潔で理解しやすい
      - アクション動詞で開始
      - 例: "ホテル検索機能の実装"

    description:
      - ストーリーフォーマットで記述
      - コンテキストと価値を明確に
      - ユーザー視点で記述

    acceptance_criteria:
      format: Given-When-Then
      example: |
        Given ユーザーがホテル検索画面にいる
        When 日付と場所を入力して検索ボタンを押す
        Then 条件に合致するホテル一覧が表示される

    additional_fields:
      - story_points: フィボナッチ数列
      - priority: P0-P3
      - labels: feature, bug, tech-debt
      - epic: 親エピック
      - dependencies: 依存関係
```

#### 3.1.2 INVEST原則

```yaml
invest_criteria:
  independent:
    definition: 他のストーリーに依存せず単独で価値を提供
    check:
      - 単独でリリース可能か
      - 他ストーリーの完了を待つ必要があるか
    bad_example: "APIが完成したら画面を実装"
    good_example: "モックデータで画面を実装（API統合は別ストーリー）"

  negotiable:
    definition: 詳細は実装前に調整可能
    check:
      - 仕様が固定されすぎていないか
      - 実装方法に余地があるか
    bad_example: "このボタンは#FF0000の赤色で"
    good_example: "エラーを視覚的に目立たせる"

  valuable:
    definition: ユーザーまたはビジネスに価値を提供
    check:
      - エンドユーザーの価値は何か
      - ビジネスへのインパクトは
    bad_example: "データベースのテーブルを作成"
    good_example: "ユーザーが予約履歴を確認できる"

  estimable:
    definition: チームが見積もり可能
    check:
      - 技術的に理解できているか
      - 不確実性は許容範囲か
    bad_example: "パフォーマンスを改善"
    good_example: "ホテル検索のレスポンスを2秒以下に"

  small:
    definition: 1スプリント内で完了可能
    check:
      - 8ポイント以下か
      - 2-3日で完了可能か
    bad_example: "ユーザー管理システムの実装"
    good_example: "パスワードリセット機能の実装"

  testable:
    definition: 受入基準が検証可能
    check:
      - 完了の判断が明確か
      - テストケースが書けるか
    bad_example: "使いやすいUIにする"
    good_example: "3クリック以内で予約完了できる"
```

### 3.2 見積もり手法

#### 3.2.1 ストーリーポイント

```yaml
story_points:
  scale: Fibonacci (1, 2, 3, 5, 8, 13, 21)

  calibration:
    reference_stories:
      1_point:
        example: "ボタンのラベルテキスト変更"
        characteristics:
          - 変更箇所が1箇所
          - テストが単純
          - レビューが最小限

      3_points:
        example: "フォームにバリデーション追加"
        characteristics:
          - 複数ファイルの変更
          - ユニットテスト必要
          - 通常のレビュープロセス

      5_points:
        example: "新しいAPI エンドポイント追加"
        characteristics:
          - 設計が必要
          - 複数レイヤーにまたがる
          - 統合テスト必要

      8_points:
        example: "検索機能の新規実装"
        characteristics:
          - 複雑な実装
          - 複数のストーリーに分割検討
          - E2Eテスト必要

      13_points:
        example: "決済フローの実装"
        characteristics:
          - 高い複雑性
          - 外部連携あり
          - 分割を強く推奨

      21_points:
        description: "スプリント内完了不可、要分割"

  what_points_represent:
    includes:
      - 実装の複雑さ
      - 技術的不確実性
      - テストの工数
      - コードレビュー
    excludes:
      - 時間（日数・時間）
      - 担当者のスキル
      - 外部要因による待ち時間
```

#### 3.2.2 プランニングポーカー

```yaml
planning_poker:
  process:
    1_present:
      - POがストーリーを説明
      - 受入基準の確認
      - 質疑応答

    2_discuss:
      - 技術的アプローチの議論
      - 不明点の明確化
      - 依存関係の確認

    3_estimate:
      - 各メンバーがカードを選択
      - 同時に公開
      - 全員がコミット

    4_converge:
      - 最大・最小を選んだ人が理由を説明
      - 議論
      - 再見積もり（必要に応じて）
      - 合意形成

  tools:
    - Planning Poker Online
    - Jira内蔵機能
    - Miro/Figjam

  tips:
    - 2回で合意を目指す
    - 長引く場合はタイムボックス
    - 技術的スパイクを検討
```

### 3.3 優先順位付けフレームワーク

#### 3.3.1 優先順位付けマトリクス

```
┌─────────────────────────────────────────────────────────────┐
│                  優先順位付けマトリクス                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│           価値（ユーザー/ビジネス）                         │
│              高                  低                         │
│         ┌─────────────────┬─────────────────┐              │
│    低   │ P0: Do First   │ P2: Fill-in    │              │
│         │ ・高価値/低工数 │ ・低価値/低工数 │              │
│    工   │ ・クイックウィン│ ・空き時間で   │              │
│    数   │                 │                 │              │
│         ├─────────────────┼─────────────────┤              │
│    高   │ P1: Plan       │ P3: Maybe Later│              │
│         │ ・高価値/高工数 │ ・低価値/高工数 │              │
│         │ ・計画的に実施 │ ・再評価/却下  │              │
│         │                 │                 │              │
│         └─────────────────┴─────────────────┘              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 3.3.2 WSJF（Weighted Shortest Job First）

```yaml
wsjf_framework:
  formula: WSJF = Cost of Delay / Job Size

  cost_of_delay:
    components:
      user_business_value:
        scale: 1-10
        question: "これがないとユーザー/ビジネスにどれだけ影響があるか"

      time_criticality:
        scale: 1-10
        question: "遅れるとどれだけ価値が減少するか"

      risk_reduction_opportunity:
        scale: 1-10
        question: "リスク軽減や機会創出にどれだけ寄与するか"

    calculation: CoD = User Value + Time Criticality + Risk/Opportunity

  job_size:
    scale: 1-10
    basis: ストーリーポイントを相対的にマッピング

  prioritization:
    1. 全ストーリーのWSJFを計算
    2. 高いものから順に優先
    3. 依存関係を考慮して調整
    4. スプリントごとに再評価
```

---

## 第4章 品質とデリバリー

### 4.1 Definition of Done（DoD）

#### 4.1.1 ストーリーレベルのDoD

```yaml
definition_of_done:
  code_quality:
    - コードがコーディング標準に準拠
    - 静的解析エラー・警告がゼロ
    - コードレビュー完了（2名以上の承認）
    - セルフレビューチェックリスト完了

  testing:
    - ユニットテスト作成（カバレッジ80%以上）
    - 統合テスト作成（該当する場合）
    - 全テストがパス
    - 手動テスト完了（受入基準の確認）

  documentation:
    - APIドキュメント更新（該当する場合）
    - READMEの更新（該当する場合）
    - 技術的決定の記録（ADR）

  deployment:
    - ステージング環境にデプロイ済み
    - 環境変数・設定の確認
    - フィーチャーフラグの設定（該当する場合）

  acceptance:
    - 受入基準すべてを満たしている
    - POによるレビュー完了
    - パフォーマンス基準を満たしている

  security:
    - セキュリティスキャン通過
    - 機密情報のハードコードなし
    - 認証・認可の確認
```

#### 4.1.2 スプリントレベルのDoD

```yaml
sprint_done:
  all_stories:
    - 全コミットストーリーがDone
    - またはスコープ調整が合意済み

  release_readiness:
    - リリースノート作成
    - 本番環境への展開準備完了
    - ロールバック手順確認

  quality_gates:
    - E2Eテストスイート通過
    - パフォーマンステスト通過
    - セキュリティスキャン通過

  documentation:
    - スプリントサマリー記録
    - メトリクス更新
```

### 4.2 受入基準標準

#### 4.2.1 Given-When-Then形式

```yaml
acceptance_criteria_format:
  structure:
    given: 事前条件・コンテキスト
    when: アクション・トリガー
    then: 期待される結果

  example_1:
    story: "ホテル検索機能"
    criteria:
      - given: ユーザーがホテル検索画面を開いている
        when: 東京、2024/03/01-03/03と入力して検索する
        then: 東京のホテルが日付順にリストされる

      - given: ユーザーがホテル検索画面を開いている
        when: 該当するホテルがない条件で検索する
        then: "該当するホテルがありません"メッセージが表示される

      - given: ユーザーが検索結果を表示している
        when: 価格フィルターで10,000円以下を選択する
        then: 10,000円以下のホテルのみが表示される

  example_2:
    story: "ユーザー登録"
    criteria:
      - given: 未登録のユーザーが登録画面を開いている
        when: 有効なメールアドレスとパスワードを入力して登録する
        then: アカウントが作成され確認メールが送信される

      - given: 未登録のユーザーが登録画面を開いている
        when: 既に登録済みのメールアドレスで登録しようとする
        then: エラーメッセージ"このメールアドレスは登録済みです"が表示される

  best_practices:
    - 具体的で検証可能
    - ユーザー視点で記述
    - エッジケースも含める
    - 5-10個程度に収める
```

### 4.3 スプリントレビューとデモ

#### 4.3.1 デモガイドライン

```yaml
demo_guidelines:
  preparation:
    before_demo:
      - デモ環境の確認
      - テストデータの準備
      - スクリプトの作成
      - バックアップ計画

    demo_script:
      structure:
        - コンテキスト設定（30秒）
        - ユーザーストーリーの紹介（30秒）
        - デモ実行（3-5分）
        - 質疑応答（1-2分）

  execution:
    do:
      - ユーザー視点でシナリオを実行
      - 成功パスだけでなくエラー処理も示す
      - フィードバックを積極的に求める
      - 質問に対して正直に回答

    dont:
      - 技術的な詳細に深入り
      - 未完成の機能をデモ
      - 問題を隠す
      - 言い訳をする

  remote_considerations:
    - 画面共有の事前テスト
    - 音声品質の確認
    - 録画の許可確認
    - チャットでの質問対応
```

---

## 第5章 メトリクスと継続的改善

### 5.1 ベロシティとバーンダウン

#### 5.1.1 ベロシティ追跡

```yaml
velocity_tracking:
  definition: スプリント内で完了したストーリーポイントの合計

  calculation:
    completed_points: DoD を満たしたストーリーのポイント合計
    committed_points: スプリント開始時のコミットポイント
    completion_rate: completed / committed * 100

  tracking:
    tool: Jira / Linear
    frequency: スプリント終了時
    visualization: ベロシティチャート

  usage:
    capacity_planning:
      - 過去3-5スプリントの平均ベロシティを使用
      - 安定するまでは保守的に見積もり
      - 季節変動（休暇等）を考慮

    forecasting:
      - ベロシティ × 残りスプリント数 = 完了可能ポイント
      - 信頼区間を使用（±20%）
      - リリース計画に活用

  anti_patterns:
    - ベロシティの目標化
    - チーム間比較
    - ベロシティを上げるための品質低下
    - ポイントのインフレーション
```

#### 5.1.2 バーンダウンチャート

```
┌─────────────────────────────────────────────────────────────┐
│                  スプリントバーンダウン例                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Story Points                                               │
│  50 ┤●                                                      │
│     │ ●                                                     │
│  40 ┤  ●                                                    │
│     │   ●──理想線                                          │
│  30 ┤    ●                                                  │
│     │     ◆──実績線                                        │
│  20 ┤      ◆                                                │
│     │       ◆                                               │
│  10 ┤        ◆                                              │
│     │         ◆                                             │
│   0 ┼─────────●──────────────────────────────────────────   │
│     Day1  Day2  Day3  Day4  Day5  Day6  Day7  Day8  Day9   │
│                                                             │
│  理想: 線形に減少                                           │
│  実績: 実際の完了状況                                       │
│                                                             │
│  健全なパターン:                                            │
│  ・理想線と実績線が近い                                     │
│  ・後半に急降下しない                                       │
│  ・スプリント終了時にゼロ近く                               │
│                                                             │
│  問題パターン:                                              │
│  ・初日から遅れ（見積もり問題）                            │
│  ・後半に急降下（完了基準の妥協）                          │
│  ・平坦（ブロッカー発生）                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 レトロスペクティブ実践

#### 5.2.1 レトロスペクティブフォーマット

```yaml
retrospective_formats:
  start_stop_continue:
    description: シンプルで効果的な定番フォーマット
    when_to_use: 新チーム、定期的なレトロ
    process:
      1. 個人でポストイット作成（5分）
      2. 共有とグルーピング（10分）
      3. 議論（20分）
      4. アクション決定（15分）

  4ls:
    description: 多面的な振り返り
    categories:
      - Liked: 良かったこと
      - Learned: 学んだこと
      - Lacked: 足りなかったこと
      - Longed for: 望むこと
    when_to_use: 深い振り返りが必要な時

  sailboat:
    description: ビジュアルでエンゲージメント向上
    elements:
      wind: 推進力（チームを前進させるもの）
      anchor: 足かせ（チームを遅らせるもの）
      rocks: リスク（見えている危険）
      island: ゴール（目指すところ）
    when_to_use: チームの方向性を確認したい時

  timeline:
    description: スプリント全体を時系列で振り返り
    when_to_use:
      - 問題の多かったスプリント
      - 新しいプロセスを試した後

  futurospective:
    description: 将来視点で考える
    question: "最高のスプリントだったとしたら、何が起きていた？"
    when_to_use: マンネリ化防止、ビジョン形成
```

#### 5.2.2 改善アクションの追跡

```yaml
action_tracking:
  action_item_format:
    what: 何をするか（具体的に）
    who: 誰が担当か
    when: いつまでに完了か
    how_to_measure: 完了をどう判断するか

  tracking_process:
    1. レトロでアクション決定（最大3つ）
    2. バックログに追加（改善ストーリーとして）
    3. 次スプリントで着手
    4. 次回レトロで振り返り

  success_metrics:
    completion_rate: 80%以上
    impact_assessment: アクションの効果を評価
    recurring_issues: 同じ問題が再発しないか

  example_actions:
    process:
      - "コードレビューのチェックリストを作成する"
      - "スタンドアップの時間を10:00に変更して試す"

    technical:
      - "テストカバレッジレポートを毎日チェックする仕組みを作る"
      - "エラーログの可視化ダッシュボードを作成する"

    team:
      - "週1回のナレッジシェアリングセッションを始める"
      - "ペアプログラミングを週2回実施する"
```

### 5.3 改善アクションの追跡

#### 5.3.1 継続的改善サイクル

```
┌─────────────────────────────────────────────────────────────┐
│                  継続的改善サイクル                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│              ┌───────────────┐                              │
│              │    Plan       │                              │
│              │  改善計画策定  │                              │
│              └───────┬───────┘                              │
│                      │                                      │
│                      ▼                                      │
│  ┌───────────────┐       ┌───────────────┐                 │
│  │    Act        │◀─────│     Do        │                 │
│  │  標準化・展開 │       │   改善実施    │                 │
│  └───────┬───────┘       └───────┬───────┘                 │
│          │                       │                          │
│          │   ┌───────────────┐   │                          │
│          └──▶│    Check      │◀──┘                          │
│              │   効果測定    │                              │
│              └───────────────┘                              │
│                                                             │
│  頻度:                                                      │
│  ・スプリントレベル: 2週間サイクル                          │
│  ・チームレベル: 月次                                       │
│  ・組織レベル: 四半期                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 5.3.2 メトリクスダッシュボード

```yaml
metrics_dashboard:
  delivery_metrics:
    velocity:
      display: トレンドチャート
      target: 安定または緩やかな向上

    sprint_completion:
      calculation: 完了SP / コミットSP
      target: 80-100%

    cycle_time:
      calculation: 着手から完了までの時間
      target: ストーリーサイズに応じた基準

    lead_time:
      calculation: 起票から完了までの時間
      target: 継続的に短縮

  quality_metrics:
    defect_density:
      calculation: 本番バグ数 / 完了SP
      target: 0.5以下

    test_coverage:
      calculation: カバー行数 / 総行数
      target: 80%以上

    code_review_time:
      calculation: PR作成からマージまでの時間
      target: 24時間以内

  team_health_metrics:
    team_satisfaction:
      method: 定期サーベイ
      target: 4.0/5.0以上

    burnout_risk:
      indicators:
        - 残業時間
        - 休暇取得率
        - 離職予兆

  visualization:
    tool: Grafana / Looker / Notion
    update_frequency: リアルタイム / Daily
    access: 全チームメンバー
```

---

## 第6章 実装ロードマップ & 文書間参照

### 6.1 フレームワーク導入タイムライン

```
┌─────────────────────────────────────────────────────────────┐
│            アジャイルフレームワーク導入タイムライン          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Phase 1: Foundation (Month 1-3)                           │
│  ├─ 基本スクラムプロセスの導入                             │
│  ├─ ロールの定義と任命                                     │
│  ├─ セレモニーの確立                                       │
│  ├─ ツールセットアップ（Jira/Linear）                      │
│  └─ チームトレーニング                                     │
│                                                             │
│  Phase 2: Optimization (Month 4-6)                         │
│  ├─ DoDの継続的改善                                        │
│  ├─ 見積もり精度の向上                                     │
│  ├─ メトリクス収集と可視化                                 │
│  └─ レトロの定着                                           │
│                                                             │
│  Phase 3: Scaling (Month 7-12)                             │
│  ├─ 複数スクワッドへの展開                                 │
│  ├─ チャプターの設立                                       │
│  ├─ クロスチーム調整プロセス                               │
│  └─ スクワッド間依存関係管理                               │
│                                                             │
│  Phase 4: Maturity (Year 2+)                               │
│  ├─ トライブ構造への移行                                   │
│  ├─ ギルドの設立                                           │
│  ├─ 自己組織化の深化                                       │
│  └─ 継続的な進化                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 スケーリングマイルストーン

```yaml
scaling_milestones:
  m1_single_team:
    timing: Month 1-6
    team_size: 5-15名
    structure: 1 Scrum Team
    practices:
      - 基本スクラム
      - 2週間スプリント
      - 全セレモニー実施
    success_criteria:
      - 安定したベロシティ
      - 完了率 80%以上
      - チーム満足度 4.0以上

  m2_multiple_squads:
    timing: Month 7-12
    team_size: 15-30名
    structure: 2-3 Squads
    practices:
      - スクワッド単位のスクラム
      - Scrum of Scrums
      - チャプターミーティング
    success_criteria:
      - スクワッド間の整合性
      - 依存関係の可視化
      - 各スクワッドの自律性

  m3_tribe_structure:
    timing: Year 2
    team_size: 30-70名
    structure: 1-2 Tribes
    practices:
      - トライブレベルの計画
      - チャプターの成熟
      - ギルドの開始
    success_criteria:
      - トライブ全体のアライメント
      - 技術標準の統一
      - ナレッジ共有の活性化

  m4_scaled_organization:
    timing: Year 3+
    team_size: 70-150名
    structure: Multiple Tribes
    practices:
      - クロストライブ調整
      - SAFeエッセンス適用
      - 自律的な組織進化
    success_criteria:
      - グローバルな整合性
      - ローカルな俊敏性
      - 持続的な改善文化
```

### 6.3 関連文書参照

```yaml
document_references:
  implementation_documents:
    - Doc-IR-001: 開発ロードマップ Year 1-3
    - Doc-IR-002: リソース＆キャパシティプランニング
    - Doc-QA-001: 品質保証戦略
    - Doc-DO-001: DevOps戦略

  technical_documents:
    - Doc-TV-001: 技術ビジョン
    - Doc-SA-001: システムアーキテクチャ
    - Doc-SC-001: セキュリティアーキテクチャ

  operational_documents:
    - Doc-OS-001: オペレーションモデル
    - Doc-OS-002: サプライチェーン管理

  reference_frameworks:
    - Scrum Guide (2020)
    - Spotify Model
    - SAFe (参考)
    - LeSS (参考)
```

### 6.4 成功基準サマリー

```yaml
success_criteria:
  process_health:
    sprint_completion_rate:
      target: 80-100%
      measurement: 完了SP / コミットSP

    velocity_stability:
      target: ±15%の変動内
      measurement: 標準偏差

    ceremony_adherence:
      target: 95%以上
      measurement: セレモニー実施率

  delivery_quality:
    defect_escape_rate:
      target: 1%以下
      measurement: 本番バグ / リリース数

    lead_time:
      target: 14日以下（Epicレベル）
      measurement: アイデアから本番まで

    deployment_frequency:
      target: 日次
      measurement: 本番デプロイ回数

  team_effectiveness:
    team_satisfaction:
      target: 4.0/5.0以上
      measurement: 定期サーベイ

    retrospective_action_completion:
      target: 80%以上
      measurement: 完了アクション / 決定アクション

    skill_growth:
      target: 年間1レベル以上
      measurement: スキルマトリクス
```

---

## 付録

### 付録A: セレモニーチェックリスト

```yaml
ceremony_checklists:
  sprint_planning:
    before:
      - [ ] バックログがリファインメント済み
      - [ ] POがトップストーリーを準備
      - [ ] チームのキャパシティ確認
      - [ ] 前スプリントの振り返り完了

    during:
      - [ ] スプリント目標を設定
      - [ ] ストーリーの受入基準を確認
      - [ ] 見積もりを確認/更新
      - [ ] タスク分解を実施
      - [ ] コミットメントを確認

    after:
      - [ ] スプリントバックログを確定
      - [ ] ボードを更新
      - [ ] カレンダーにイベントを設定

  daily_scrum:
    before:
      - [ ] ボードを最新に更新
      - [ ] ブロッカーを事前に共有

    during:
      - [ ] 全員が参加
      - [ ] 15分以内に終了
      - [ ] ブロッカーを特定

    after:
      - [ ] ブロッカーの解決に着手
      - [ ] 必要に応じてフォローアップ会議

  sprint_review:
    before:
      - [ ] デモ環境の準備
      - [ ] ステークホルダーへの招待
      - [ ] デモスクリプトの作成

    during:
      - [ ] スプリント目標の振り返り
      - [ ] 完了ストーリーのデモ
      - [ ] フィードバック収集

    after:
      - [ ] フィードバックをバックログに反映
      - [ ] メトリクスの記録
      - [ ] リリース判断

  sprint_retrospective:
    before:
      - [ ] フォーマットの選択
      - [ ] ツールの準備
      - [ ] 前回アクションの確認

    during:
      - [ ] 全員の声を聴く
      - [ ] 具体的なアクションを決定
      - [ ] オーナーを決定

    after:
      - [ ] アクションをバックログに追加
      - [ ] 議事録の共有
```

### 付録B: ユーザーストーリーテンプレート

```markdown
## ストーリータイトル

### ストーリー
As a [ユーザータイプ],
I want [機能/行動],
so that [価値/目的].

### 受入基準
- [ ] **Given** [事前条件], **When** [アクション], **Then** [結果]
- [ ] **Given** [事前条件], **When** [アクション], **Then** [結果]
- [ ] **Given** [事前条件], **When** [アクション], **Then** [結果]

### 技術的考慮事項
- 依存関係:
- 技術的リスク:
- パフォーマンス要件:

### デザイン
- Figmaリンク:

### テスト計画
- ユニットテスト:
- 統合テスト:
- E2Eテスト:

### Story Points:
### Priority: P0/P1/P2/P3
### Labels:
### Epic:
```

### 付録C: Definition of Doneチェックリスト

```yaml
dod_checklist:
  code:
    - [ ] コーディング標準に準拠
    - [ ] 静的解析エラーなし
    - [ ] セルフレビュー完了
    - [ ] ピアレビュー完了（2名承認）

  testing:
    - [ ] ユニットテスト作成（カバレッジ80%+）
    - [ ] 統合テスト作成（該当する場合）
    - [ ] 全テストパス
    - [ ] 受入基準の手動確認

  documentation:
    - [ ] API仕様書更新（該当する場合）
    - [ ] README更新（該当する場合）
    - [ ] 技術的決定記録（重要な場合）

  deployment:
    - [ ] ステージングデプロイ成功
    - [ ] 環境変数確認
    - [ ] フィーチャーフラグ設定

  acceptance:
    - [ ] 受入基準すべて満たす
    - [ ] POレビュー完了
    - [ ] パフォーマンス基準クリア

  security:
    - [ ] セキュリティスキャンパス
    - [ ] 機密情報ハードコードなし
    - [ ] 認証・認可確認
```

---

**文書情報**

| 項目 | 内容 |
|------|------|
| Document ID | Doc-DM-001 |
| Version | 1.0.0 |
| Created | 2026-01-20 |
| Last Updated | 2026-01-20 |
| Status | Draft |
| Owner | Implementation Agent |
| Reviewers | CTO, Agile Coach, Engineering Managers |
| Total Lines | 1,500+ |

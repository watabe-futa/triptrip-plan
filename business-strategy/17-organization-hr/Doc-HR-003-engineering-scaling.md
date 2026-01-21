# Doc-HR-003: TripTripエンジニアリング組織スケーリング

## エグゼクティブサマリー

本文書は、TripTripエンジニアリング組織のスケーリング戦略を包括的に定義します。5人から100人以上への成長を見据え、組織スケーリングモデル、テックリード/エンジニアリングマネージャーのキャリアパス設計、オンボーディングプログラム、技術的卓越性の維持メカニズム、エンジニア生産性と満足度の測定・改善について詳細に規定します。Spotify、Google、Netflix、Metaのエンジニアリング組織スケーリングのベストプラクティスを参考に、急成長期における開発生産性の維持、技術的卓越性の確保、エンジニアリングカルチャーの醸成を実現します。

---

## 第1章 はじめに & コンテキスト

### 1.1 エンジニアリング組織のビジョン

#### 1.1.1 エンジニアリングビジョンステートメント

```
┌─────────────────────────────────────────────────────────────┐
│           TripTripエンジニアリングビジョン                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │     "技術で旅行体験を変革する、                      │   │
│  │      世界クラスのエンジニアリング組織を構築する"      │   │
│  │                                                     │   │
│  │     Building a World-Class Engineering Organization  │   │
│  │     that Transforms Travel Through Technology        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  【エンジニアリングの使命】                                  │
│  ├─ 顧客に愛されるプロダクトを迅速に届ける                 │
│  ├─ 技術的卓越性を追求し、業界をリードする                 │
│  ├─ エンジニアが成長し、活躍できる環境を創る               │
│  └─ 持続可能で信頼性の高いシステムを構築する               │
│                                                             │
│  【エンジニアリング原則】                                    │
│  1. Customer Obsession - 顧客価値を最優先                   │
│  2. Technical Excellence - 妥協なき技術品質                 │
│  3. Continuous Learning - 学び続ける姿勢                    │
│  4. Ownership - 自分のコードに責任を持つ                    │
│  5. Collaboration - 共に成長する                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 1.1.2 エンジニアリング組織の目標

```yaml
engineering_goals:
  year1:
    delivery:
      - "MVP機能の完成と国内ローンチ"
      - "決済・認証機能の本番化"
      - "パフォーマンス最適化"
    team:
      - "15-20名のコアチーム構築"
      - "DevOps/SRE基盤の確立"
      - "開発プロセスの標準化"
    culture:
      - "コードレビュー文化の定着"
      - "ドキュメンテーション習慣の確立"
      - "継続的学習の環境整備"

  year2:
    delivery:
      - "マイクロサービス移行開始"
      - "APAC展開対応"
      - "リアルタイム機能追加"
    team:
      - "50-65名へのスケーリング"
      - "スクワッドモデルの本格運用"
      - "テックリード層の育成"
    culture:
      - "自律的なチーム運営"
      - "技術共有の活性化"
      - "エンジニアブランディング確立"

  year3:
    delivery:
      - "グローバルプラットフォーム完成"
      - "AI/ML機能の本格導入"
      - "100万DAU対応のスケール"
    team:
      - "100名以上のグローバルチーム"
      - "トライブ構造の確立"
      - "海外拠点エンジニアリング"
    culture:
      - "イノベーション文化"
      - "オープンソース貢献"
      - "業界リーダーシップ"
```

### 1.2 スケーリングの課題

#### 1.2.1 成長段階別の技術的・組織的課題

```
┌─────────────────────────────────────────────────────────────┐
│            エンジニアリング組織スケーリングの課題             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  【5-15名: 創業期】                                          │
│  技術的課題:                                                │
│  ├─ アーキテクチャの属人化                                 │
│  ├─ テストカバレッジ不足                                   │
│  ├─ 技術的負債の蓄積                                       │
│  └─ ドキュメント不足                                       │
│                                                             │
│  組織的課題:                                                │
│  ├─ 役割の曖昧さ                                           │
│  ├─ オンボーディングの未整備                               │
│  ├─ ナレッジの暗黙知化                                     │
│  └─ バーンアウトリスク                                     │
│                                                             │
│  【15-50名: 成長期】                                         │
│  技術的課題:                                                │
│  ├─ コードベースの肥大化                                   │
│  ├─ 依存関係の複雑化                                       │
│  ├─ デプロイの遅延                                         │
│  └─ インシデント対応の属人化                               │
│                                                             │
│  組織的課題:                                                │
│  ├─ コミュニケーションオーバーヘッド                       │
│  ├─ 意思決定の遅延                                         │
│  ├─ 品質のばらつき                                         │
│  └─ 文化の希薄化                                           │
│                                                             │
│  【50-100名以上: 拡大期】                                    │
│  技術的課題:                                                │
│  ├─ マイクロサービス移行の複雑さ                           │
│  ├─ 分散システムの運用                                     │
│  ├─ セキュリティ・コンプライアンス                         │
│  └─ グローバル展開のインフラ                               │
│                                                             │
│  組織的課題:                                                │
│  ├─ サイロ化                                               │
│  ├─ 官僚化リスク                                           │
│  ├─ タイムゾーン調整                                       │
│  └─ 多文化マネジメント                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 1.2.2 スケーリングの失敗パターンと回避策

```yaml
scaling_anti_patterns:
  premature_optimization:
    description: "早すぎるマイクロサービス化"
    symptoms:
      - "チームサイズに対して複雑すぎるアーキテクチャ"
      - "運用負荷の急増"
      - "開発速度の低下"
    mitigation:
      - "チームサイズに合わせた段階的な分割"
      - "モジュラーモノリスからの移行"
      - "明確な分割基準の設定"

  hero_culture:
    description: "特定の個人への依存"
    symptoms:
      - "一人のエンジニアがボトルネック"
      - "バス係数1（Bus Factor = 1）"
      - "過労・バーンアウト"
    mitigation:
      - "ペアプログラミング・モブプログラミング"
      - "ドキュメンテーションの義務化"
      - "知識共有セッションの定期開催"

  process_bloat:
    description: "過度なプロセス導入"
    symptoms:
      - "承認待ちの長期化"
      - "会議過多"
      - "開発者の不満増加"
    mitigation:
      - "プロセスの定期的な見直し"
      - "チームへの権限委譲"
      - "「プロセス税」の可視化"

  culture_dilution:
    description: "急成長による文化の希薄化"
    symptoms:
      - "新メンバーの価値観不一致"
      - "「昔は良かった」症候群"
      - "エンゲージメント低下"
    mitigation:
      - "文化的オンボーディングの強化"
      - "バリューの行動への落とし込み"
      - "文化の体現者（カルチャーチャンピオン）"
```

### 1.3 業界ベストプラクティス（Spotify、Google、Netflix）

#### 1.3.1 各社のスケーリングアプローチ

```yaml
industry_best_practices:
  spotify:
    model: "スクワッド/トライブ/チャプター/ギルド"
    key_learnings:
      - "自律性と整合性のバランス"
      - "チャプターによる専門性の維持"
      - "ギルドによる知識共有"
    triptrip_adoption:
      - "スクワッドモデルをPhase 2から導入"
      - "チャプターで技術標準を維持"
      - "ギルドで横断的な知識共有"

  google:
    model: "大規模エンジニアリング組織運営"
    key_learnings:
      - "コードレビューの徹底（全コード）"
      - "モノレポ（単一リポジトリ）"
      - "強力な開発者ツール投資"
      - "20%ルール（イノベーション時間）"
    triptrip_adoption:
      - "コードレビュー必須化"
      - "開発者体験への投資"
      - "イノベーション時間の確保"

  netflix:
    model: "Freedom and Responsibility"
    key_learnings:
      - "高い自由度と高い期待"
      - "コンテキストを共有、コントロールは最小化"
      - "カオスエンジニアリング"
    triptrip_adoption:
      - "コンテキスト共有の徹底"
      - "意思決定の分散化"
      - "SREプラクティスの導入"

  meta:
    model: "Move Fast"
    key_learnings:
      - "高速なイテレーション"
      - "Bootcamp（集中オンボーディング）"
      - "インパクトベースの評価"
    triptrip_adoption:
      - "短いスプリントサイクル"
      - "構造化されたオンボーディング"
      - "成果ベースの評価"
```

---

## 第2章 組織スケーリングモデル

### 2.1 Phase 1: 創業期（5-15人）

#### 2.1.1 組織構造

```
┌─────────────────────────────────────────────────────────────┐
│              Phase 1: 創業期組織構造（5-15名）               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                        ┌───────────┐                        │
│                        │    CTO    │                        │
│                        └─────┬─────┘                        │
│                              │                              │
│              ┌───────────────┼───────────────┐             │
│              │               │               │             │
│              ▼               ▼               ▼             │
│        ┌───────────┐   ┌───────────┐   ┌───────────┐      │
│        │  Mobile   │   │  Backend  │   │ Platform  │      │
│        │   Team    │   │   Team    │   │   Team    │      │
│        │  (5名)    │   │  (4名)    │   │  (3名)    │      │
│        └───────────┘   └───────────┘   └───────────┘      │
│                                                             │
│  【チーム構成】                                              │
│  Mobile Team:                                               │
│  ├─ Tech Lead (1)                                          │
│  ├─ Senior Flutter Engineer (2)                            │
│  └─ Flutter Engineer (2)                                   │
│                                                             │
│  Backend Team:                                              │
│  ├─ Tech Lead (1)                                          │
│  ├─ Senior Backend Engineer (1)                            │
│  ├─ Backend Engineer (1)                                   │
│  └─ Data Engineer (1)                                      │
│                                                             │
│  Platform Team:                                             │
│  ├─ DevOps Engineer (1)                                    │
│  ├─ SRE Engineer (1)                                       │
│  └─ Security Engineer (1)                                  │
│                                                             │
│  横断: QA Engineer (2) ※各チームに分散                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 2.1.2 Phase 1の運営モデル

```yaml
phase1_operations:
  communication:
    daily_standup:
      format: "全員参加、15分"
      focus: "進捗、ブロッカー、今日のフォーカス"

    weekly_sync:
      format: "CTO主導、45分"
      agenda:
        - "週の振り返り"
        - "技術的課題の共有"
        - "次週の優先事項"

    async_default:
      primary_tool: "Slack"
      documentation: "Notion"

  decision_making:
    technical:
      daily: "個人/ペア → Tech Lead承認"
      weekly: "チーム → CTO承認"
      strategic: "CTO → 経営チーム"

  code_practices:
    review_policy: "全PR、1人以上のレビュー必須"
    merge_strategy: "Squash merge to main"
    ci_requirements:
      - "全テストパス"
      - "リンター通過"
      - "ビルド成功"

  delivery:
    sprint_length: "1週間"
    release_cadence: "週次（最低）"
    deployment: "GitHub Actions → 自動デプロイ"
```

#### 2.1.3 Phase 1の成功基準

```yaml
phase1_success_criteria:
  team_health:
    headcount: "12-15名"
    attrition: "<5% (1名未満)"
    engagement: ">4.0/5.0"

  delivery:
    velocity_stability: "±20%以内の変動"
    deployment_frequency: "日次可能"
    lead_time: "<3日"
    mttr: "<4時間"

  quality:
    test_coverage: ">60%"
    bug_escape_rate: "<5%"
    security_vulnerabilities: "Critical: 0"

  culture:
    documentation_coverage: ">70%"
    code_review_turnaround: "<24時間"
    knowledge_sharing: "月2回以上"
```

### 2.2 Phase 2: 成長期（15-50人）

#### 2.2.1 組織構造

```
┌─────────────────────────────────────────────────────────────┐
│              Phase 2: 成長期組織構造（15-50名）              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                        ┌───────────┐                        │
│                        │    CTO    │                        │
│                        └─────┬─────┘                        │
│                              │                              │
│        ┌─────────────────────┼─────────────────────┐       │
│        │                     │                     │       │
│        ▼                     ▼                     ▼       │
│  ┌───────────┐        ┌───────────┐        ┌───────────┐  │
│  │    EM     │        │    EM     │        │    EM     │  │
│  │ Product   │        │ Platform  │        │Experience │  │
│  └─────┬─────┘        └─────┬─────┘        └─────┬─────┘  │
│        │                     │                     │       │
│   ┌────┴────┐          ┌────┴────┐          ┌────┴────┐  │
│   │         │          │         │          │         │  │
│   ▼         ▼          ▼         ▼          ▼         ▼  │
│ ┌─────┐  ┌─────┐    ┌─────┐  ┌─────┐    ┌─────┐  ┌─────┐│
│ │Hotel│  │Comm.│    │Core │  │Data │    │Ticket│ │Search││
│ │Squad│  │Squad│    │Platf│  │Squad│    │Squad│  │Squad││
│ │(8名)│  │(8名)│    │(6名)│  │(4名)│    │(8名) │ │(6名) ││
│ └─────┘  └─────┘    └─────┘  └─────┘    └─────┘  └─────┘│
│                                                             │
│  【チャプター構成】                                          │
│  ├─ Frontend Chapter (Flutter)                             │
│  ├─ Backend Chapter (Node.js)                              │
│  ├─ Data Chapter                                           │
│  └─ QA Chapter                                             │
│                                                             │
│  合計: 40名                                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 2.2.2 スクワッド運営

```yaml
squad_operations:
  composition:
    typical_size: "6-10名"
    roles:
      product_manager: 1
      tech_lead: 1
      senior_engineer: 2-3
      engineer: 2-4
      qa_engineer: 0.5-1  # チャプター共有

  autonomy:
    owns:
      - "スプリント計画"
      - "技術的実装決定"
      - "運用（オンコール）"
    escalates:
      - "他スクワッドとの依存関係"
      - "アーキテクチャ変更"
      - "リソース要求"

  ceremonies:
    daily:
      standup: "15分、同期 or 非同期"
    weekly:
      refinement: "1時間、バックログ整理"
      demo: "30分、成果共有"
    bi_weekly:
      sprint_planning: "2時間"
      retrospective: "1時間"

  metrics:
    velocity: "ストーリーポイント/スプリント"
    cycle_time: "着手から完了までの日数"
    quality: "バグ発生率、テストカバレッジ"
```

#### 2.2.3 チャプター運営

```yaml
chapter_operations:
  frontend_chapter:
    lead: "Senior Flutter Engineer"
    scope:
      - "Flutter/Dartベストプラクティス"
      - "UIコンポーネント標準"
      - "パフォーマンス基準"
    activities:
      - "隔週チャプターミーティング（1時間）"
      - "コードレビュー基準の策定"
      - "技術ガイドラインの更新"

  backend_chapter:
    lead: "Senior Backend Engineer"
    scope:
      - "API設計ガイドライン"
      - "データベースパターン"
      - "サービス間通信"
    activities:
      - "隔週チャプターミーティング"
      - "技術勉強会（月次）"
      - "アーキテクチャレビュー参加"

  chapter_lead_responsibilities:
    - "専門領域の技術的方向性"
    - "メンバーの技術的成長支援"
    - "標準・ガイドラインの策定"
    - "採用への技術的インプット"
    # 注: 人事評価はEMが担当
```

### 2.3 Phase 3: 成熟期（50-100人以上）

#### 2.3.1 トライブ構造

```
┌─────────────────────────────────────────────────────────────┐
│             Phase 3: 成熟期組織構造（50-100名+）             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                        ┌───────────┐                        │
│                        │    CTO    │                        │
│                        └─────┬─────┘                        │
│                              │                              │
│     ┌────────────────────────┼────────────────────────┐    │
│     │                        │                        │    │
│     ▼                        ▼                        ▼    │
│ ┌─────────────┐       ┌─────────────┐       ┌─────────────┐│
│ │  Traveler   │       │   Supply    │       │  Platform   ││
│ │   Tribe     │       │   Tribe     │       │   Tribe     ││
│ │   (45名)    │       │   (35名)    │       │   (25名)    ││
│ │             │       │             │       │             ││
│ │  VP Eng     │       │  VP Eng     │       │  VP Eng     ││
│ │     │       │       │     │       │       │     │       ││
│ │  EM─┼─EM    │       │  EM─┼─EM    │       │  EM─┼─EM    ││
│ │     │       │       │     │       │       │     │       ││
│ │ ┌───┼───┐   │       │ ┌───┼───┐   │       │ ┌───┼───┐   ││
│ │ S1 S2 S3 S4 │       │ S1 S2 S3    │       │ S1 S2 S3    ││
│ └─────────────┘       └─────────────┘       └─────────────┘│
│                                                             │
│  S = Squad (8-10名)                                        │
│                                                             │
│  横断組織:                                                  │
│  ├─ Chapters: Frontend, Backend, Data, QA, Security        │
│  ├─ Guilds: ML, UX, Performance, Architecture              │
│  └─ Enabling: Developer Experience, Architecture Review    │
│                                                             │
│  合計: 105名                                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 2.3.2 トライブ運営

```yaml
tribe_operations:
  tribe_lead_responsibilities:
    strategic:
      - "トライブのロードマップ策定"
      - "リソース配分の決定"
      - "他トライブとの調整"
    tactical:
      - "スクワッド間の依存関係管理"
      - "トライブ内の技術的一貫性"
      - "採用計画の策定"
    people:
      - "EM/TLの育成"
      - "組織健康の監視"
      - "エスカレーション対応"

  tribe_ceremonies:
    weekly:
      tribe_sync: "30分、スクワッドリード間の同期"
    bi_weekly:
      tech_sync: "1時間、技術的課題の議論"
    monthly:
      tribe_demo: "1時間、トライブ全体での成果共有"
    quarterly:
      planning: "半日、次四半期の計画"
      retrospective: "2時間、振り返り"

  coordination_mechanisms:
    dependency_management:
      - "週次の依存関係レビュー"
      - "共有カレンダー"
      - "Slackチャンネル"
    technical_alignment:
      - "Architecture Decision Records (ADR)"
      - "RFCプロセス"
      - "チャプター経由の標準化"
```

### 2.4 チーム分割とスコープ定義

#### 2.4.1 チーム分割の原則

```yaml
team_split_principles:
  when_to_split:
    size_trigger: "チームが10名を超えたら分割検討"
    cognitive_load: "チームの責任範囲が広すぎる"
    delivery_bottleneck: "並行開発が困難"
    domain_boundary: "明確なドメイン境界が存在"

  how_to_split:
    domain_driven:
      approach: "ビジネスドメインに沿った分割"
      example: "Hotel Squad, Commerce Squad, Ticket Squad"
      benefits:
        - "エンドツーエンドの責任"
        - "ドメイン知識の深化"
      challenges:
        - "共通機能の重複リスク"

    capability_driven:
      approach: "技術的能力に沿った分割"
      example: "Frontend Team, Backend Team, Data Team"
      benefits:
        - "技術的専門性の深化"
        - "リソース効率"
      challenges:
        - "引き渡しのオーバーヘッド"

    triptrip_approach:
      phase1: "Capability-driven（技術別）"
      phase2_plus: "Domain-driven（ドメイン別）+ Platform"
      rationale:
        - "初期は技術的基盤構築が重要"
        - "成長後はドメイン自律性が重要"
```

#### 2.4.2 スコープ定義テンプレート

```yaml
squad_scope_template:
  squad_name: "[Squad Name]"

  mission: "[一文でのミッション]"

  domain_ownership:
    primary:
      - "[主要ドメイン1]"
      - "[主要ドメイン2]"
    secondary:
      - "[サポートドメイン]"

  service_ownership:
    services:
      - "[サービス名1]"
      - "[サービス名2]"
    apis:
      - "[API名1]"

  code_ownership:
    repositories:
      - "[リポジトリ1]"
    directories:
      - "[ディレクトリパス1]"

  kpis:
    business:
      - "[ビジネスKPI1]"
    technical:
      - "[技術KPI1]"

  dependencies:
    upstream:
      - "[依存先Squad1]"
    downstream:
      - "[依存元Squad1]"

  on_call:
    rotation: "[ローテーション方式]"
    escalation: "[エスカレーションパス]"
```

---

## 第3章 キャリアパス設計

### 3.1 ICトラック（Individual Contributor）

#### 3.1.1 エンジニアリングレベル定義

```
┌─────────────────────────────────────────────────────────────┐
│                エンジニアリングレベル定義                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  L1: Junior Engineer                                        │
│  ──────────────────                                         │
│  経験: 0-2年                                                │
│  スコープ: タスクレベル                                     │
│  期待:                                                      │
│  ├─ 明確な仕様に基づく実装                                 │
│  ├─ 基本的なテスト作成                                     │
│  ├─ コードレビューを受けて改善                             │
│  └─ ドキュメントの更新                                     │
│  影響範囲: 自分のタスク                                     │
│                                                             │
│  L2: Engineer                                               │
│  ──────────────────                                         │
│  経験: 2-4年                                                │
│  スコープ: 機能レベル                                       │
│  期待:                                                      │
│  ├─ 独立した機能開発                                       │
│  ├─ 設計の一部を担当                                       │
│  ├─ コードレビューに参加                                   │
│  └─ 技術的課題の解決                                       │
│  影響範囲: 担当機能、チーム                                 │
│                                                             │
│  L3: Senior Engineer                                        │
│  ──────────────────                                         │
│  経験: 4-7年                                                │
│  スコープ: プロジェクト/サービスレベル                      │
│  期待:                                                      │
│  ├─ 複雑な機能/サービスの設計・実装                        │
│  ├─ ジュニアのメンタリング                                 │
│  ├─ 技術的意思決定への貢献                                 │
│  └─ チーム横断での影響力                                   │
│  影響範囲: チーム、関連チーム                               │
│                                                             │
│  L4: Staff Engineer                                         │
│  ──────────────────                                         │
│  経験: 7-10年                                               │
│  スコープ: 複数チーム/ドメインレベル                        │
│  期待:                                                      │
│  ├─ アーキテクチャの設計・進化                             │
│  ├─ 技術戦略への貢献                                       │
│  ├─ 組織全体の技術的レベル向上                             │
│  └─ 技術的リーダーシップ                                   │
│  影響範囲: 複数チーム、組織                                 │
│                                                             │
│  L5: Principal Engineer                                     │
│  ──────────────────                                         │
│  経験: 10年以上                                             │
│  スコープ: 組織/業界レベル                                  │
│  期待:                                                      │
│  ├─ 全社的な技術方針の策定                                 │
│  ├─ 業界への影響（OSS、発信）                              │
│  ├─ 最も困難な技術課題の解決                               │
│  └─ CTOの技術的パートナー                                  │
│  影響範囲: 全社、業界                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 3.1.2 ICトラックのコンピテンシーマトリクス

```yaml
ic_competency_matrix:
  technical_skills:
    L1:
      coding: "基本的な実装、指導の下で作業"
      design: "小規模な実装の設計"
      testing: "ユニットテスト作成"
      debugging: "基本的なデバッグ"

    L2:
      coding: "複雑な実装、品質の高いコード"
      design: "機能レベルの設計"
      testing: "統合テスト、テスト戦略"
      debugging: "複雑な問題のトラブルシューティング"

    L3:
      coding: "最適化、パフォーマンス考慮"
      design: "サービス/コンポーネント設計"
      testing: "テスト戦略策定"
      debugging: "本番問題の迅速な解決"

    L4:
      coding: "ベストプラクティス確立"
      design: "システムアーキテクチャ設計"
      testing: "品質戦略策定"
      debugging: "組織的な問題解決能力向上"

    L5:
      coding: "業界標準の確立"
      design: "全社アーキテクチャ戦略"
      testing: "品質文化の構築"
      debugging: "未知の問題への対応"

  soft_skills:
    L1:
      communication: "明確な報告"
      collaboration: "チーム内での協力"
      mentoring: "なし"
      influence: "なし"

    L2:
      communication: "技術的な説明"
      collaboration: "チーム間の連携"
      mentoring: "ペアプログラミング"
      influence: "チーム内"

    L3:
      communication: "複雑な概念の説明"
      collaboration: "クロスチーム連携のリード"
      mentoring: "ジュニアのメンタリング"
      influence: "チーム、関連チーム"

    L4:
      communication: "技術戦略の発信"
      collaboration: "組織横断のイニシアチブ"
      mentoring: "シニアのメンタリング"
      influence: "組織全体"

    L5:
      communication: "外部への技術発信"
      collaboration: "業界連携"
      mentoring: "組織的なメンタリング体系"
      influence: "業界"
```

### 3.2 マネジメントトラック

#### 3.2.1 マネジメントレベル定義

```
┌─────────────────────────────────────────────────────────────┐
│                マネジメントレベル定義                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Tech Lead (TL)                                             │
│  ──────────────────                                         │
│  スコープ: チーム（技術面）                                  │
│  レポート: 0人（IC + リーダーシップ）                        │
│  責任:                                                      │
│  ├─ チームの技術的方向性                                   │
│  ├─ アーキテクチャ決定                                     │
│  ├─ コードレビュー基準                                     │
│  └─ 技術的メンタリング                                     │
│  Note: 人事評価は担当しない                                 │
│                                                             │
│  Engineering Manager (EM)                                   │
│  ──────────────────                                         │
│  スコープ: チーム/スクワッド（1つ）                          │
│  レポート: 5-10名                                           │
│  責任:                                                      │
│  ├─ チームのデリバリー                                     │
│  ├─ ピープルマネジメント                                   │
│  ├─ 採用・オンボーディング                                 │
│  └─ パフォーマンス管理                                     │
│                                                             │
│  Senior Engineering Manager                                 │
│  ──────────────────                                         │
│  スコープ: 複数チーム/スクワッド                             │
│  レポート: 2-3 EM + TL                                      │
│  責任:                                                      │
│  ├─ 複数チームの調整                                       │
│  ├─ EMの育成                                               │
│  ├─ 中期計画策定                                           │
│  └─ プロセス改善                                           │
│                                                             │
│  Director of Engineering                                    │
│  ──────────────────                                         │
│  スコープ: ドメイン/トライブ                                 │
│  レポート: 3-5 Sr.EM                                        │
│  責任:                                                      │
│  ├─ ドメインの技術戦略                                     │
│  ├─ 組織設計                                               │
│  ├─ 大規模プロジェクト管理                                 │
│  └─ ステークホルダー管理                                   │
│                                                             │
│  VP of Engineering                                          │
│  ──────────────────                                         │
│  スコープ: トライブ/エンジニアリング領域                     │
│  レポート: Directors                                        │
│  責任:                                                      │
│  ├─ エンジニアリング戦略                                   │
│  ├─ 予算・リソース管理                                     │
│  ├─ 組織横断イニシアチブ                                   │
│  └─ 経営チームとの連携                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 3.2.2 マネジメントコンピテンシー

```yaml
management_competencies:
  tech_lead:
    technical_leadership:
      - "チームの技術的方向性を決定"
      - "アーキテクチャ決定をリード"
      - "技術的負債の管理"
    mentoring:
      - "チームメンバーの技術的成長支援"
      - "コードレビューを通じた教育"
    collaboration:
      - "他チームとの技術的調整"
      - "ステークホルダーとの技術コミュニケーション"

  engineering_manager:
    people_management:
      - "1on1の実施（週次/隔週）"
      - "パフォーマンスレビュー"
      - "キャリア開発支援"
      - "採用面接・意思決定"
    delivery_management:
      - "スプリント計画・追跡"
      - "障害の除去"
      - "リスク管理"
    team_building:
      - "チーム文化の醸成"
      - "チームプロセスの改善"
      - "コンフリクト解決"

  director:
    strategic_leadership:
      - "中長期技術戦略"
      - "組織設計"
      - "予算計画"
    organizational_development:
      - "マネージャーの育成"
      - "組織健康の監視"
      - "変革管理"
    stakeholder_management:
      - "経営層への報告"
      - "他部門との連携"
      - "外部パートナーとの関係"
```

### 3.3 レベル定義と昇進基準

#### 3.3.1 昇進基準

```yaml
promotion_criteria:
  general_principles:
    - "現在のレベルで一貫して期待を超える"
    - "次のレベルの責任を既に担っている"
    - "ピアレビューで高評価"
    - "組織のニーズとの合致"

  ic_track:
    L1_to_L2:
      timeline: "1-2年"
      criteria:
        - "独立した機能開発ができる"
        - "コードレビューに建設的に参加"
        - "基本的な設計ができる"

    L2_to_L3:
      timeline: "2-3年"
      criteria:
        - "複雑な機能をリードして実装"
        - "ジュニアのサポート"
        - "技術的意思決定への貢献"
        - "チーム外への影響"

    L3_to_L4:
      timeline: "3-4年"
      criteria:
        - "複数チームにまたがる影響"
        - "アーキテクチャへの貢献"
        - "技術戦略への参画"
        - "組織的な技術リーダーシップ"

    L4_to_L5:
      timeline: "4年以上"
      criteria:
        - "全社的な技術影響"
        - "業界への貢献"
        - "最難関課題の解決"
        - "CTOレベルの技術パートナー"

  management_track:
    TL_to_EM:
      criteria:
        - "ピープルマネジメントへの志向"
        - "チームビルディング能力"
        - "コミュニケーション能力"
      note: "トラック変更は可逆的"

    EM_to_Sr_EM:
      timeline: "2-3年"
      criteria:
        - "複数チームの管理能力"
        - "マネージャー育成実績"
        - "組織改善の実績"

    Sr_EM_to_Director:
      timeline: "2-3年"
      criteria:
        - "大規模組織の管理"
        - "戦略的思考"
        - "経営視点"
```

### 3.4 評価・フィードバックプロセス

#### 3.4.1 パフォーマンスレビューサイクル

```yaml
performance_review_cycle:
  annual_cycle:
    q1: "年間目標設定"
    q2: "中間レビュー（軽量）"
    q3: "中間レビュー（軽量）"
    q4: "年次レビュー"

  annual_review:
    components:
      self_assessment:
        - "目標達成度"
        - "コンピテンシー自己評価"
        - "キャリア希望"

      peer_feedback:
        - "3-5名のピアから収集"
        - "強み、改善点"
        - "匿名オプションあり"

      manager_assessment:
        - "目標達成度評価"
        - "コンピテンシー評価"
        - "昇進推薦"

    output:
      - "パフォーマンス評価（5段階）"
      - "報酬調整推薦"
      - "昇進判断"
      - "開発計画"

  rating_scale:
    5_exceptional: "期待を大幅に超える"
    4_exceeds: "期待を超える"
    3_meets: "期待を満たす"
    2_developing: "一部期待を下回る"
    1_needs_improvement: "期待を大きく下回る"

  calibration:
    process:
      - "マネージャー間での評価調整"
      - "相対評価の確認"
      - "バイアスチェック"
    participants: "EM, Director, VP"
```

#### 3.4.2 継続的フィードバック

```yaml
continuous_feedback:
  one_on_ones:
    frequency: "週次 or 隔週"
    duration: "30分"
    agenda:
      - "最近の状況（仕事、個人）"
      - "障害・サポートニーズ"
      - "フィードバック（双方向）"
      - "キャリア・成長"
    ownership: "レポートが主導"

  real_time_feedback:
    mechanisms:
      - "Slackでの即時フィードバック"
      - "PRコメントでの具体的フィードバック"
      - "1on1での定期的なフィードバック"
    principles:
      - "SBI（Situation, Behavior, Impact）形式"
      - "具体的で行動可能"
      - "タイムリー（24-48時間以内）"

  360_feedback:
    frequency: "年次（年次レビュー時）"
    participants:
      - "直属マネージャー"
      - "ピア（3-5名）"
      - "直属レポート（該当する場合）"
      - "クロスファンクショナルパートナー"
```

---

## 第4章 オンボーディングプログラム

### 4.1 Week 1-4プログラム設計

#### 4.1.1 オンボーディングタイムライン

```
┌─────────────────────────────────────────────────────────────┐
│            エンジニアオンボーディング Week 1-4               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  【Week 1: 基盤構築】                                        │
│  Day 1:                                                     │
│  ├─ 09:00 Welcome & HR手続き                               │
│  ├─ 10:30 IT機器セットアップ                               │
│  ├─ 13:00 ランチ（チーム）                                 │
│  ├─ 14:00 アカウント設定（Slack, GitHub, etc.）            │
│  └─ 16:00 バディとの1on1                                   │
│                                                             │
│  Day 2-3:                                                   │
│  ├─ 開発環境セットアップ                                   │
│  ├─ アーキテクチャ概要セッション                           │
│  ├─ コードベースツアー                                     │
│  └─ 初めてのPR（小さなタスク）                             │
│                                                             │
│  Day 4-5:                                                   │
│  ├─ チーム紹介セッション                                   │
│  ├─ プロダクト概要セッション                               │
│  ├─ 開発フロー・ツール研修                                 │
│  └─ カルチャーオリエンテーション                           │
│                                                             │
│  【Week 2: 実践開始】                                        │
│  ├─ 最初の機能タスク（ペアプログラミング）                 │
│  ├─ コードレビュー参加開始                                 │
│  ├─ スタンドアップ参加                                     │
│  ├─ 1on1: マネージャー                                     │
│  └─ チームプロセス深掘り                                   │
│                                                             │
│  【Week 3: 自立支援】                                        │
│  ├─ 独立した小〜中規模タスク                               │
│  ├─ オンコール/運用トレーニング                            │
│  ├─ 専門領域Deep Dive（Flutter/Backend等）                 │
│  └─ クロスチーム紹介                                       │
│                                                             │
│  【Week 4: フィードバックと調整】                            │
│  ├─ 30日レビュー（マネージャー + バディ）                  │
│  ├─ オンボーディングフィードバック提出                     │
│  ├─ 中規模プロジェクトへのアサイン                         │
│  └─ 継続的学習計画の策定                                   │
│                                                             │
│  【成功基準】                                                │
│  ├─ Week 1: 環境構築完了、初PRマージ                       │
│  ├─ Week 2: 独立したタスク完了                             │
│  ├─ Week 4: 生産的なチームメンバーとして機能              │
│  └─ 90日: 完全に自立                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 4.1.2 オンボーディングチェックリスト

```yaml
onboarding_checklist:
  pre_start:
    hr:
      - "契約書類の準備"
      - "PCの発注"
      - "アカウントの事前作成"
    manager:
      - "バディのアサイン"
      - "最初のタスクの準備"
      - "チームへの通知"

  day_1:
    administrative:
      - "[ ] 出社/リモートログイン"
      - "[ ] HR手続き完了"
      - "[ ] 機器受取/セットアップ"
    technical:
      - "[ ] Slack参加"
      - "[ ] GitHub招待受諾"
      - "[ ] Notion/Linearアクセス"
    social:
      - "[ ] チームへの自己紹介"
      - "[ ] バディとの初回1on1"

  week_1:
    technical:
      - "[ ] 開発環境構築完了"
      - "[ ] アプリのローカル実行"
      - "[ ] 初PRの作成・マージ"
    learning:
      - "[ ] アーキテクチャセッション完了"
      - "[ ] プロダクトセッション完了"
      - "[ ] カルチャーオリエンテーション完了"
    social:
      - "[ ] チームメンバー全員と挨拶"
      - "[ ] マネージャーとの1on1"

  week_2_4:
    technical:
      - "[ ] 複数のPR完了"
      - "[ ] コードレビュー参加"
      - "[ ] オンコールトレーニング"
    learning:
      - "[ ] 専門領域Deep Dive"
      - "[ ] 関連チームとの接点"
    feedback:
      - "[ ] 30日レビュー完了"
      - "[ ] オンボーディングフィードバック提出"
```

### 4.2 バディ・メンタリング制度

#### 4.2.1 バディプログラム

```yaml
buddy_program:
  purpose:
    - "新入社員の早期適応支援"
    - "非公式な質問・相談窓口"
    - "社内ネットワーク構築の支援"
    - "文化的なガイダンス"

  buddy_selection:
    criteria:
      - "同じチームまたは近いチーム"
      - "1年以上の在籍"
      - "コミュニケーション能力"
      - "ボランティアベース"

  buddy_responsibilities:
    week_1:
      - "毎日のチェックイン（15分）"
      - "ランチ/コーヒーの同行"
      - "ツール・プロセスの案内"
    week_2_4:
      - "週2-3回のチェックイン"
      - "質問への回答"
      - "社内紹介"
    month_2_3:
      - "週1回のチェックイン"
      - "必要に応じたサポート"

  buddy_training:
    - "バディ役割の説明"
    - "よくある質問と回答"
    - "エスカレーションのタイミング"

  recognition:
    - "バディ貢献の評価への反映"
    - "バディ経験者の表彰"
```

#### 4.2.2 メンタリングプログラム

```yaml
mentoring_program:
  purpose:
    - "長期的なキャリア開発支援"
    - "技術的成長の加速"
    - "組織内のネットワーク構築"

  types:
    technical_mentoring:
      focus: "技術スキルの向上"
      duration: "6-12ヶ月"
      frequency: "月2回（30分）"
      matching: "技術領域に基づく"

    career_mentoring:
      focus: "キャリア開発"
      duration: "12ヶ月"
      frequency: "月1回（45分）"
      matching: "希望するキャリアパス"

    leadership_mentoring:
      focus: "リーダーシップ開発"
      target: "TL/EM候補"
      duration: "6ヶ月"
      frequency: "月2回（30分）"
      mentor: "Director以上"

  mentor_responsibilities:
    - "定期的なセッションの実施"
    - "目標設定の支援"
    - "フィードバックの提供"
    - "ネットワーク紹介"

  mentee_responsibilities:
    - "積極的な参加"
    - "目標の明確化"
    - "セッションの準備"
    - "学びの実践"
```

### 4.3 技術オンボーディング（コードベース、ツール）

#### 4.3.1 技術オンボーディングカリキュラム

```yaml
technical_onboarding:
  day_1_environment_setup:
    checklist:
      - "MacBook Proセットアップ"
      - "Homebrew, Git, Node.js, Flutter SDK"
      - "IDE（VSCode/Android Studio）"
      - "Docker Desktop"
    documentation: "setup-guide.md"
    support: "バディ + Slackチャンネル"

  day_2_3_codebase_tour:
    frontend:
      topics:
        - "プロジェクト構造（Feature-based Package）"
        - "状態管理（Provider + Riverpod）"
        - "UIコンポーネント"
        - "API通信層"
      activities:
        - "コードウォークスルー（TL）"
        - "サンプル機能の実装"

    backend:
      topics:
        - "Honoフレームワーク"
        - "Prisma ORM"
        - "API設計パターン"
        - "認証・認可"
      activities:
        - "APIエンドポイント追加"
        - "データベースマイグレーション"

  week_1_tools_training:
    development:
      - "Git/GitHub（ブランチ戦略、PR）"
      - "CI/CD（GitHub Actions）"
      - "テスト（ユニット、統合、E2E）"
    collaboration:
      - "Slack（チャンネル構成、マナー）"
      - "Notion（ドキュメント構成）"
      - "Linear（タスク管理）"
    operations:
      - "監視ダッシュボード"
      - "ログ検索"
      - "インシデント対応フロー"

  week_2_4_specialization:
    by_role:
      flutter_engineer:
        - "Flutterパフォーマンス最適化"
        - "プラットフォーム固有の実装"
        - "状態管理Deep Dive"

      backend_engineer:
        - "データベース設計"
        - "API設計原則"
        - "非同期処理"

      devops_engineer:
        - "インフラ構成"
        - "Kubernetes運用"
        - "セキュリティツール"
```

### 4.4 カルチャーオンボーディング

#### 4.4.1 カルチャーセッション

```yaml
culture_onboarding:
  day_1_welcome_session:
    presenter: "CEO or CTO"
    duration: "60分"
    content:
      - "TripTripのミッション・ビジョン"
      - "会社の歴史と現在地"
      - "コアバリュー（JOURNEY）"
      - "Q&A"

  week_1_culture_deep_dive:
    sessions:
      values_workshop:
        duration: "90分"
        format: "ワークショップ"
        content:
          - "各バリューの具体例"
          - "バリュー違反の例"
          - "自分の経験との紐付け"

      ways_of_working:
        duration: "60分"
        content:
          - "カルチャーコード"
          - "コミュニケーション規範"
          - "意思決定プロセス"

  ongoing_culture_immersion:
    week_2_4:
      - "各チームとのコーヒーチャット"
      - "Town Hall参加"
      - "Slackカルチャーチャンネル観察"

    month_1_3:
      - "カルチャーストーリー収集"
      - "バリュー実践レポート（30日、90日）"
      - "オンボーディングフィードバック"
```

---

## 第5章 技術的卓越性の維持

### 5.1 コードレビュー文化

#### 5.1.1 コードレビューガイドライン

```yaml
code_review_guidelines:
  principles:
    - "すべてのコードはレビューを経る"
    - "レビューは学習の機会"
    - "建設的かつ具体的なフィードバック"
    - "迅速なターンアラウンド（24時間以内）"

  reviewer_responsibilities:
    required:
      - "最低1名のApprovalが必要"
      - "変更の影響範囲を理解"
      - "具体的なフィードバック提供"
    optional:
      - "代替アプローチの提案"
      - "関連ドキュメントへのリンク"

  author_responsibilities:
    - "PRの目的を明確に記述"
    - "適切なサイズ（200-400行目安）"
    - "セルフレビューの実施"
    - "テストの追加"
    - "フィードバックへの対応"

  review_checklist:
    correctness:
      - "ロジックは正しいか"
      - "エッジケースは考慮されているか"
      - "バグの可能性はないか"
    design:
      - "設計は適切か"
      - "DRY、SOLID原則"
      - "既存パターンとの整合性"
    maintainability:
      - "コードは読みやすいか"
      - "命名は適切か"
      - "ドキュメントは十分か"
    security:
      - "セキュリティ上の問題はないか"
      - "入力のバリデーション"
      - "認証・認可の確認"
    testing:
      - "テストは十分か"
      - "テストは意味があるか"
      - "エッジケースのテスト"

  feedback_format:
    categories:
      must_change: "[Must] セキュリティ問題、バグ"
      should_change: "[Should] 設計改善、ベストプラクティス"
      nit: "[Nit] スタイル、好み"
      question: "[Q] 理解のための質問"
      praise: "[Praise] 良い点の称賛"
```

#### 5.1.2 コードレビューメトリクス

```yaml
code_review_metrics:
  turnaround_time:
    target: "<24時間（初回レビュー）"
    measurement: "PR作成〜初回レビューコメント"

  review_coverage:
    target: "100%（すべてのPR）"

  review_quality:
    indicators:
      - "フィードバックの具体性"
      - "レビュー後のバグ率"
      - "著者の満足度"

  reviewer_load:
    target: "1人あたり5PR/日以下"
    balancing: "自動アサイン/ローテーション"
```

### 5.2 技術共有（Tech Talk、読書会）

#### 5.2.1 技術共有プログラム

```
┌─────────────────────────────────────────────────────────────┐
│                  技術共有プログラム                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  【Tech Talk】                                               │
│  頻度: 隔週（金曜15:00-16:00）                              │
│  形式: 社内プレゼンテーション（30-45分 + Q&A）              │
│  トピック例:                                                │
│  ├─ 新技術の調査・検証結果                                 │
│  ├─ プロジェクトのアーキテクチャ決定                       │
│  ├─ 障害対応の振り返り                                     │
│  ├─ 外部カンファレンスの共有                               │
│  └─ 業界トレンド                                           │
│                                                             │
│  運営:                                                      │
│  ├─ 発表者: ボランティア + ローテーション                  │
│  ├─ 録画: 後で視聴可能                                     │
│  └─ Q&A: Slido等で非同期質問も可                           │
│                                                             │
│  【読書会】                                                  │
│  頻度: 週次（希望者）                                       │
│  形式: 輪読 + ディスカッション（30分）                      │
│  対象書籍例:                                                │
│  ├─ "Designing Data-Intensive Applications"                │
│  ├─ "The Staff Engineer's Path"                            │
│  ├─ "Team Topologies"                                      │
│  └─ Flutter/Dart関連書籍                                   │
│                                                             │
│  【ランチ&ラーン】                                           │
│  頻度: 月次                                                 │
│  形式: ランチ + 軽いトーク（外部ゲスト含む）                │
│  トピック: キャリア、技術トレンド、多様なテーマ            │
│                                                             │
│  【ライトニングトーク】                                      │
│  頻度: 月次（月末金曜）                                     │
│  形式: 5分×6人程度                                          │
│  トピック: 何でもOK（技術、趣味、TIL）                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 ハッカソン・イノベーションタイム

#### 5.3.1 ハッカソン

```yaml
hackathon:
  frequency: "四半期（年4回）"
  duration: "2日間"
  format:
    day_1:
      - "09:00 キックオフ、チーム編成"
      - "10:00 ハッキング開始"
      - "18:00 チェックイン"
    day_2:
      - "09:00 ハッキング継続"
      - "15:00 デモ準備"
      - "16:00 デモ（3分/チーム）"
      - "17:30 投票・表彰"

  themes:
    examples:
      - "ユーザー体験改善"
      - "開発者体験改善"
      - "新機能プロトタイプ"
      - "技術的負債解消"
      - "フリーテーマ"

  rules:
    team_size: "2-5名"
    cross_functional: "推奨"
    outcome: "デモ可能な成果物"

  awards:
    categories:
      - "Best Overall"
      - "Most Innovative"
      - "Best Technical"
      - "Most Likely to Ship"
    prizes:
      - "賞金/ギフトカード"
      - "全社表彰"
      - "実装への優先権"
```

#### 5.3.2 イノベーションタイム

```yaml
innovation_time:
  policy:
    allocation: "20%（週1日相当）"
    eligibility: "全エンジニア"
    approval: "マネージャー合意"

  guidelines:
    eligible_activities:
      - "新技術の調査・プロトタイピング"
      - "開発者ツールの改善"
      - "技術的負債の返済"
      - "OSSへの貢献"
      - "社内ツール開発"

    not_eligible:
      - "通常のプロダクト開発"
      - "副業・個人プロジェクト"

  tracking:
    - "週次レポート（任意）"
    - "四半期の振り返り共有"

  examples:
    past_innovations:
      - "CI/CDパイプラインの改善"
      - "社内CLIツール"
      - "テスト自動化フレームワーク"
      - "Flutterパフォーマンス分析ツール"
```

### 5.4 外部コミュニティ参加

#### 5.4.1 外部発信・参加支援

```yaml
external_engagement:
  conference_speaking:
    support:
      - "登壇準備時間の確保"
      - "スライドレビュー"
      - "参加費・旅費の補助"
    target: "月1回以上（チーム全体で）"
    target_conferences:
      - "iOSDC"
      - "DroidKaigi"
      - "ServerlessDays"
      - "DevOpsDays"
      - "国際Flutter/Dartカンファレンス"

  meetup_hosting:
    owned:
      - "TripTrip Tech Night（四半期）"
    co_hosted:
      - "Flutter Meetup Tokyo"
      - "TypeScript Meetup"

  oss_contribution:
    policy:
      - "業務関連OSSへの貢献推奨"
      - "貢献時間の業務認定"
      - "重要な貢献は評価に反映"
    target_projects:
      - "Flutter/Dart関連"
      - "Hono"
      - "Prisma"
      - "使用している主要ライブラリ"

  tech_blog:
    target: "月4本以上（チーム全体）"
    topics:
      - "技術Deep Dive"
      - "プロジェクト事例"
      - "障害対応振り返り"
      - "新技術検証"
    support:
      - "執筆時間の確保"
      - "レビュープロセス"
      - "公開後のプロモーション"
```

---

## 第6章 生産性 & 満足度

### 6.1 開発者生産性指標（DORA metrics）

#### 6.1.1 DORAメトリクス

```yaml
dora_metrics:
  deployment_frequency:
    definition: "本番へのデプロイ頻度"
    target:
      phase1: "週次以上"
      phase2: "日次以上"
      phase3: "オンデマンド"
    measurement: "GitHub Actions/デプロイログ"

  lead_time_for_changes:
    definition: "コミットから本番デプロイまでの時間"
    target:
      phase1: "<1週間"
      phase2: "<1日"
      phase3: "<1時間"
    measurement: "PR作成〜デプロイ完了"

  change_failure_rate:
    definition: "デプロイによる障害発生率"
    target: "<15%"
    measurement: "障害を引き起こしたデプロイ / 全デプロイ"

  time_to_restore:
    definition: "障害発生から復旧までの時間"
    target: "<1時間"
    measurement: "インシデント管理ツール"
```

#### 6.1.2 追加の生産性指標

```yaml
additional_productivity_metrics:
  code_metrics:
    pr_size:
      target: "200-400行（中央値）"
      rationale: "レビューしやすいサイズ"

    pr_cycle_time:
      definition: "PR作成〜マージ"
      target: "<24時間"

    code_review_turnaround:
      definition: "PR作成〜初回レビュー"
      target: "<4時間"

  delivery_metrics:
    sprint_completion:
      definition: "コミットしたストーリーの完了率"
      target: ">85%"

    bug_escape_rate:
      definition: "本番で発見されたバグの割合"
      target: "<5%"

  quality_metrics:
    test_coverage:
      target: ">80%"

    technical_debt_ratio:
      definition: "技術的負債対応に費やす時間の割合"
      target: "<20%"
```

### 6.2 エンゲージメント測定

#### 6.2.1 エンジニアエンゲージメント調査

```yaml
engineer_engagement_survey:
  frequency: "四半期"
  method: "匿名アンケート（Lattice/Culture Amp等）"

  dimensions:
    work_satisfaction:
      questions:
        - "日々の仕事にやりがいを感じるか"
        - "自分のスキルを活かせているか"
        - "適切な難易度の仕事が与えられているか"

    growth_opportunity:
      questions:
        - "成長機会が十分にあるか"
        - "新しいスキルを学べているか"
        - "キャリアパスが明確か"

    team_collaboration:
      questions:
        - "チームの協力体制は良好か"
        - "意見を言いやすい環境か"
        - "チームメンバーを信頼しているか"

    manager_relationship:
      questions:
        - "マネージャーからのサポートは十分か"
        - "フィードバックは役立っているか"
        - "1on1は有意義か"

    tools_and_process:
      questions:
        - "開発ツールは生産性を支えているか"
        - "プロセスは効率的か"
        - "技術的意思決定に参加できているか"

    work_life_balance:
      questions:
        - "ワークライフバランスは取れているか"
        - "働きすぎていると感じないか"
        - "休暇を取りやすいか"

  scoring:
    scale: "1-5（強く反対〜強く同意）"
    targets:
      overall: ">4.0"
      each_dimension: ">3.5"

  action_planning:
    - "結果の全体共有"
    - "チーム別の振り返り"
    - "改善アクションの策定"
    - "次回調査でのフォローアップ"
```

### 6.3 バーンアウト防止策

#### 6.3.1 バーンアウトの兆候と対策

```yaml
burnout_prevention:
  warning_signs:
    individual:
      - "長時間労働の継続"
      - "休暇の未取得"
      - "パフォーマンスの低下"
      - "コミュニケーションの減少"
      - "ネガティブな発言の増加"

    team:
      - "離職率の上昇"
      - "病欠の増加"
      - "スプリント完了率の低下"
      - "チーム雰囲気の悪化"

  preventive_measures:
    workload_management:
      - "スプリントキャパシティの適正化"
      - "オンコール負荷の分散"
      - "期限のリアリスティックな設定"

    rest_and_recovery:
      - "有給消化の推奨（最低60%）"
      - "連続勤務制限（10日以上禁止）"
      - "夜間・休日の連絡制限"

    support_systems:
      - "マネージャーの定期チェックイン"
      - "メンタルヘルスリソース"
      - "バディ/メンターによる支援"

  intervention:
    when_detected:
      - "1on1での直接対話"
      - "ワークロードの調整"
      - "必要に応じて休暇推奨"
      - "専門家への紹介（必要時）"
```

#### 6.3.2 持続可能なペース

```yaml
sustainable_pace:
  working_hours:
    target: "週40時間（標準）"
    overtime:
      - "繁忙期でも週50時間以内"
      - "2週間以上の残業継続禁止"
      - "残業の振替休暇推奨"

  on_call:
    rotation: "チーム内でローテーション"
    frequency: "1人あたり月1週以下"
    compensation: "オンコール手当 + 代休"
    escalation: "30分以内に応答できない場合"

  meetings:
    guidelines:
      - "No Meeting Wednesday（水曜午後）"
      - "25分/50分ミーティング"
      - "アジェンダなしの会議禁止"
    targets:
      - "個人: 1日3時間以下"
      - "IC: 週10時間以下"

  vacation:
    policy:
      - "年間最低10日以上の取得推奨"
      - "連続5日以上の休暇推奨"
      - "休暇中の連絡禁止"
    tracking: "マネージャーによる消化状況確認"
```

---

## 第7章 実装ロードマップ

### 7.1 Phase別実装計画

```yaml
engineering_scaling_roadmap:
  phase1_year1:
    q1_q2:
      organization:
        - "15名体制への採用"
        - "Mobile/Backend/Platformチーム形成"
        - "Tech Lead配置"
      process:
        - "コードレビュー必須化"
        - "CI/CD整備"
        - "オンボーディングプロセス確立"
      culture:
        - "Tech Talk開始（月次）"
        - "カルチャーコード策定"

    q3_q4:
      organization:
        - "20名体制達成"
        - "EM採用/内部昇進"
        - "QA機能強化"
      process:
        - "スプリント運営の安定化"
        - "オンコールローテーション"
      culture:
        - "ハッカソン初回開催"
        - "Tech Blog開始"

  phase2_year2:
    q1_q2:
      organization:
        - "40名体制、スクワッド形成"
        - "チャプター組織化"
      process:
        - "OKRフレームワーク本格化"
        - "DORAメトリクス計測開始"
      culture:
        - "ギルド設立"
        - "外部登壇増加"

    q3_q4:
      organization:
        - "60名体制"
        - "EM層の拡充"
      process:
        - "アーキテクチャレビュー制度化"
        - "RFCプロセス導入"
      culture:
        - "イノベーションタイム導入"

  phase3_year3:
    organization:
      - "100名+体制"
      - "トライブ構造確立"
      - "VP層の採用"
    process:
      - "グローバル開発プロセス"
      - "マルチタイムゾーン対応"
    culture:
      - "OSS貢献プログラム"
      - "国際カンファレンス主催"
```

---

## 第8章 文書間参照 & 統合ポイント

### 8.1 関連文書参照

```yaml
document_references:
  primary:
    - doc_id: "Doc-HR-001"
      title: "組織設計・組織文化戦略"
      relationship: "組織構造と文化の基盤"
      integration:
        - "組織ビジョン・バリューとの整合"
        - "カルチャーコードとの連携"

    - doc_id: "Doc-HR-002"
      title: "人材獲得・採用戦略"
      relationship: "採用とオンボーディングの連携"
      integration:
        - "採用基準との整合"
        - "オンボーディングへの接続"

    - doc_id: "Doc-DM-003"
      title: "チーム構造＆スケーリング"
      relationship: "IT戦略側のチーム設計"
      integration:
        - "技術的な組織構造"
        - "Spotifyモデルの詳細"

  secondary:
    - doc_id: "Doc-IR-001"
      title: "開発ロードマップ"
      relationship: "技術ロードマップとの連携"

    - doc_id: "Doc-IR-002"
      title: "リソース＆キャパシティプランニング"
      relationship: "人員計画の詳細"

    - doc_id: "EXISTING_APP_ANALYSIS.md"
      title: "既存アプリ分析"
      relationship: "現在の技術状況"
```

### 8.2 統合ポイント

```yaml
integration_summary:
  engineering_to_culture:
    principle: "エンジニアリング文化は全社文化の延長"
    alignment:
      - "技術的卓越性 = Excellence（コアバリュー）"
      - "知識共有 = Yes, And（コアバリュー）"
      - "継続的改善 = Nimble & Adaptive"

  process_to_delivery:
    principle: "プロセスはデリバリーを支援する"
    alignment:
      - "コードレビュー → 品質向上"
      - "CI/CD → デプロイ頻度向上"
      - "スプリント → 予測可能性向上"

  career_to_retention:
    principle: "キャリア成長がリテンションを支える"
    alignment:
      - "明確なレベル定義 → 目標設定"
      - "評価の透明性 → 公平感"
      - "成長機会 → エンゲージメント"
```

---

## 付録

### A. オンボーディングチェックリストテンプレート

```markdown
# エンジニアオンボーディングチェックリスト

## 新入社員情報
- 氏名:
- 入社日:
- チーム:
- マネージャー:
- バディ:

## Day 1
- [ ] 出社/ログイン
- [ ] HR手続き
- [ ] 機器受取
- [ ] Slackログイン
- [ ] GitHubアクセス
- [ ] バディ1on1

## Week 1
- [ ] 開発環境構築
- [ ] 初PR作成
- [ ] アーキテクチャセッション
- [ ] プロダクトセッション
- [ ] カルチャーオリエンテーション

## Week 2-4
- [ ] 独立したタスク完了
- [ ] コードレビュー参加
- [ ] オンコールトレーニング
- [ ] 30日レビュー

## 署名
- マネージャー:
- バディ:
- 新入社員:
```

### B. 1on1テンプレート

```markdown
# 1on1 ミーティングノート

日付:
参加者:

## チェックイン
- 最近の状況（仕事/個人）:
- エネルギーレベル（1-10）:

## トピック
1.
2.
3.

## フィードバック
- マネージャー→メンバー:
- メンバー→マネージャー:

## アクションアイテム
- [ ]
- [ ]

## 次回のフォーカス
-
```

### C. 用語集

| 用語 | 定義 |
|------|------|
| IC | Individual Contributor（個人貢献者）|
| TL | Tech Lead |
| EM | Engineering Manager |
| DORA | DevOps Research and Assessment |
| OKR | Objectives and Key Results |
| RFC | Request for Comments |
| ADR | Architecture Decision Record |
| PR | Pull Request |
| CI/CD | Continuous Integration / Continuous Deployment |
| SRE | Site Reliability Engineering |

---

**文書情報**
- 文書ID: Doc-HR-003
- バージョン: 1.0.0
- 作成日: 2026-01-21
- 最終更新: 2026-01-21
- ステータス: 完成
- 次回レビュー: 2026-04-21

---

*本文書は、TripTripエンジニアリング組織のスケーリング戦略の基盤文書として、すべての組織的・技術的意思決定の指針となります。*

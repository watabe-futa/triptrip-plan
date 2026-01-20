# Doc-DM-003: TripTripチーム構造＆スケーリング戦略

## エグゼクティブサマリー

本文書は、TripTripエンジニアリング組織のチーム構造設計とスケーリング戦略を包括的に定義します。Spotifyモデル（スクワッド、チャプター、トライブ、ギルド）を基盤とし、15名から150名への成長に対応する組織設計、クロスファンクショナルチーム構成、採用ロードマップ、オンボーディングプログラム、キャリアプログレッションフレームワークを策定します。Google、Meta、Amazonの組織スケーリングベストプラクティスを参考に、急成長期における開発生産性の維持、技術的卓越性の確保、エンジニアリングカルチャーの醸成を実現します。リモートワーク・ハイブリッドワークに対応したコラボレーションプロトコルも含め、世界クラスのエンジニアリング組織構築を目指します。

---

## 第1章 はじめに & コンテキスト

### 1.1 目的 & スコープ

#### 1.1.1 文書の目的

本チーム構造＆スケーリング戦略文書は、TripTripのエンジニアリング組織を持続的に成長させるための包括的なガイドラインを提供します。

**主要目的**:

1. **組織設計の明確化**: スケーリングに適した組織構造の定義
2. **チーム自律性の確保**: 自律的に価値を届けられるチーム単位の設計
3. **技術的整合性の維持**: チャプター/ギルドによる技術標準の統一
4. **人材育成の体系化**: キャリアパスとスキル開発の明確化
5. **カルチャーの維持**: 成長しながらエンジニアリング文化を保持

**スコープ定義**:

```
┌─────────────────────────────────────────────────────────────┐
│              チーム構造＆スケーリングスコープ                │
├─────────────────────────────────────────────────────────────┤
│ インスコープ:                                                │
│  ├─ Spotifyモデルの適用設計                                 │
│  ├─ スクワッド/チーム構成                                   │
│  ├─ チャプター＆ギルド運営                                  │
│  ├─ 採用ロードマップ                                        │
│  ├─ オンボーディングプログラム                              │
│  ├─ キャリアラダー＆コンピテンシー                          │
│  ├─ リモートワーク/コラボレーション                         │
│  └─ エンジニアリングカルチャー                              │
├─────────────────────────────────────────────────────────────┤
│ アウトオブスコープ:                                          │
│  ├─ 詳細な報酬設計（→Doc-IR-002参照）                      │
│  ├─ 採用プロセス詳細（→Doc-IR-002参照）                    │
│  ├─ アジャイルプロセス詳細（→Doc-DM-001参照）              │
│  └─ 非エンジニアリング組織設計                              │
└─────────────────────────────────────────────────────────────┘
```

#### 1.1.2 対象読者

- エンジニアリングリーダー（CTO、VP、Director）
- エンジニアリングマネージャー
- 人事部門（HR）
- 全エンジニア
- 経営層

### 1.2 組織成長目標

#### 1.2.1 フェーズ別チーム規模

```
┌─────────────────────────────────────────────────────────────┐
│              エンジニアリング組織成長計画                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  人数                                                       │
│  150 │                                    ████████████     │
│      │                               █████               │
│  100 │                          █████                    │
│      │                     █████                         │
│   60 │                █████                              │
│      │           █████                                   │
│   30 │      █████                                        │
│      │ █████                                             │
│    5 │█                                                  │
│      └───────────────────────────────────────────────────  │
│        Now   Q2Y1  Q4Y1  Q2Y2  Q4Y2  Q2Y3  Q4Y3          │
│                                                             │
│  成長フェーズ:                                              │
│  ├─ Phase 1 (5→15名): コアチーム形成                       │
│  ├─ Phase 2 (15→30名): マルチスクワッド                    │
│  ├─ Phase 3 (30→60名): トライブ構造導入                    │
│  └─ Phase 4 (60→150名): マルチトライブ                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 1.2.2 成長原則

```yaml
scaling_principles:
  conways_law:
    principle: "組織構造はシステムアーキテクチャを反映する"
    application:
      - チーム境界 = サービス境界
      - チームが所有するサービスを明確化
      - コミュニケーションパスの最小化

  dunbar_number:
    principle: "効果的な関係は150人まで"
    application:
      - トライブは最大150人
      - スクワッドは6-10人
      - 直接的なコラボレーション範囲を設計

  two_pizza_rule:
    principle: "2枚のピザで養えるチームサイズ"
    application:
      - スクワッドは6-10人
      - 意思決定の迅速化
      - オーナーシップの明確化

  inverse_conway:
    principle: "目標アーキテクチャに合わせて組織を設計"
    application:
      - マイクロサービス = 独立スクワッド
      - プラットフォーム = 専門チーム
      - ドメイン境界 = チーム境界
```

### 1.3 現在のチーム状況

#### 1.3.1 現状分析（EXISTING_APP_ANALYSIS.mdより）

```yaml
current_team_state:
  total_engineers: 5

  composition:
    tech_lead_architect: 1
    fullstack_engineers: 2
    mobile_engineer: 1
    junior_engineer: 1

  skill_coverage:
    flutter_dart: Strong (4/5 proficient)
    nodejs_typescript: Moderate (3/5 proficient)
    postgresql: Moderate (3/5 proficient)
    devops_cloud: Weak (1/5 partial)
    security: Gap (0/5)
    ml_ai: Gap (0/5)

  codebase_ownership:
    flutter_app: 35,000 lines
    backend: 4,700 lines
    documentation: 8,189 lines
    test_files: 46

  challenges:
    - 単一障害点（特定スキルの属人化）
    - DevOps/SREの不足
    - セキュリティ専門家の不在
    - チーム拡大時のナレッジ転送
```

---

## 第2章 組織設計

### 2.1 Spotifyモデル適用

#### 2.1.1 Spotifyモデル概要

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Spotifyモデル組織構造                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                           ┌───────────────────┐                         │
│                           │      Tribe        │                         │
│                           │    (トライブ)      │                         │
│                           │   40-150人        │                         │
│                           └─────────┬─────────┘                         │
│                                     │                                   │
│               ┌─────────────────────┼─────────────────────┐            │
│               ▼                     ▼                     ▼            │
│        ┌───────────┐         ┌───────────┐         ┌───────────┐      │
│        │  Squad A  │         │  Squad B  │         │  Squad C  │      │
│        │(スクワッド)│         │(スクワッド)│         │(スクワッド)│      │
│        │  6-10人   │         │  6-10人   │         │  6-10人   │      │
│        └─────┬─────┘         └─────┬─────┘         └─────┬─────┘      │
│              │                     │                     │            │
│              │         ┌───────────┴───────────┐         │            │
│              │         │       Chapter         │         │            │
│              └────────▶│     (チャプター)       │◀────────┘            │
│                        │   同じ専門性の集まり   │                        │
│                        │   (Backend, Frontend等)│                        │
│                        └───────────────────────┘                        │
│                                                                         │
│                        ┌───────────────────────┐                        │
│                        │        Guild          │                        │
│                        │      (ギルド)         │                        │
│                        │   興味ベースの集まり   │                        │
│                        │  (Security, MLなど)   │                        │
│                        └───────────────────────┘                        │
│                                                                         │
│  コンセプト:                                                            │
│  ├─ Squad: 自律的なミニスタートアップ                                   │
│  ├─ Chapter: 専門性による横断組織（上司関係）                           │
│  ├─ Tribe: 関連スクワッドの集合                                        │
│  └─ Guild: 興味・関心ベースのコミュニティ                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 2.1.2 TripTripへの適用方針

```yaml
spotify_model_adaptation:
  principles:
    autonomy_with_alignment:
      description: スクワッドは自律的だが全社目標と整合
      mechanisms:
        - OKRによる目標連携
        - アーキテクチャガイドライン
        - 共通のDefinition of Done

    mission_oriented_squads:
      description: 各スクワッドは明確なミッションを持つ
      examples:
        - Booking Squad: 予約体験の最大化
        - Growth Squad: ユーザー獲得・活性化
        - Platform Squad: 開発者生産性の向上

    cross_functional:
      description: スクワッド内で開発を完結
      composition:
        - Product Owner / Manager
        - Engineers (Backend + Frontend)
        - Designer
        - QA Engineer
      anti_pattern:
        - 他チームへの依存
        - ハンドオフの発生

  japanese_adaptation:
    # 日本の組織文化への適応
    consensus_building:
      description: 根回し文化との調和
      approach: Nemawashi meeting before decision

    seniority_respect:
      description: 年功序列への配慮
      approach: Chapter Leadはシニアが担当

    indirect_communication:
      description: 間接的なフィードバック文化
      approach: 1:1でのフィードバック重視
```

### 2.2 スクワッド構成

#### 2.2.1 スクワッド定義

```yaml
squad_definition:
  size:
    minimum: 4
    optimal: 6-8
    maximum: 10

  composition:
    required_roles:
      product_owner:
        count: 1
        responsibility: プロダクトビジョン、優先順位
        allocation: 100% or shared (max 2 squads)

      tech_lead:
        count: 1
        responsibility: 技術方針、コードレビュー
        allocation: 100%

      backend_engineer:
        count: 1-3
        responsibility: API、データベース、ビジネスロジック

      frontend_engineer:
        count: 1-3
        responsibility: Flutter UI、ユーザー体験

      qa_engineer:
        count: 0.5-1
        responsibility: テスト戦略、品質保証
        allocation: dedicated or shared

    optional_roles:
      designer:
        count: 0.5-1
        allocation: often shared across squads

      data_analyst:
        count: 0.5
        allocation: shared or embedded

  autonomy_scope:
    owns:
      - プロダクトバックログの優先順位（PO承認後）
      - 技術的実装方法
      - リリースタイミング
      - 内部プロセス改善

    shared:
      - アーキテクチャ決定（ADR経由）
      - 技術スタック選択
      - 品質基準

    escalates:
      - 大規模な技術変更
      - チーム間の依存関係解決
      - リソース追加要求
```

#### 2.2.2 Year 1-3 スクワッド計画

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    スクワッド構成計画 Year 1-3                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Year 1 Q1-Q2 (15-20名): 2 スクワッド                                   │
│  ═══════════════════════════════════════════════════════════════════    │
│                                                                         │
│  ┌───────────────────────────┐  ┌───────────────────────────┐         │
│  │    Product Squad (10)     │  │   Platform Squad (5)     │         │
│  │    ──────────────────     │  │   ────────────────       │         │
│  │    PO: 1                  │  │   Lead: 1                │         │
│  │    Tech Lead: 1           │  │   DevOps: 2              │         │
│  │    Backend: 3             │  │   SRE: 1                 │         │
│  │    Frontend: 3            │  │   Security: 1            │         │
│  │    QA: 1                  │  │                          │         │
│  │    Designer: 1            │  │   Mission:               │         │
│  │                           │  │   開発基盤・運用         │         │
│  │    Mission:               │  │                          │         │
│  │    コア機能開発           │  │                          │         │
│  └───────────────────────────┘  └───────────────────────────┘         │
│                                                                         │
│  Year 1 Q3-Q4 (25-30名): 4 スクワッド                                   │
│  ═══════════════════════════════════════════════════════════════════    │
│                                                                         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐        │
│  │  Booking (8)    │  │  Commerce (8)   │  │  Growth (6)     │        │
│  │  ────────────   │  │  ────────────   │  │  ────────────   │        │
│  │  ホテル・チケット│  │  商品・決済     │  │  獲得・活性化   │        │
│  │  予約体験       │  │  購入体験       │  │  リテンション   │        │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘        │
│                                                                         │
│  ┌─────────────────┐                                                   │
│  │  Platform (8)   │                                                   │
│  │  ────────────   │                                                   │
│  │  インフラ・SRE  │                                                   │
│  │  開発者体験     │                                                   │
│  └─────────────────┘                                                   │
│                                                                         │
│  Year 2 (60名): 7 スクワッド + 2 Platform                              │
│  ═══════════════════════════════════════════════════════════════════    │
│                                                                         │
│  Consumer Tribe:           Merchant Tribe:        Platform Tribe:      │
│  ├─ Booking Squad          ├─ Supplier Squad      ├─ Infrastructure   │
│  ├─ Commerce Squad         ├─ Inventory Squad     ├─ Data Platform    │
│  ├─ Growth Squad           └─ Analytics Squad     └─ Security         │
│  └─ Search Squad                                                       │
│                                                                         │
│  Year 3 (150名): 15+ スクワッド                                        │
│  ═══════════════════════════════════════════════════════════════════    │
│                                                                         │
│  Consumer Tribe (50):      Merchant Tribe (35):   Platform Tribe (45): │
│  ├─ Booking                ├─ Supplier Mgmt       ├─ Infrastructure   │
│  ├─ Commerce               ├─ Inventory           ├─ Data             │
│  ├─ Search & Discovery     ├─ Analytics           ├─ Security         │
│  ├─ User Experience        ├─ Operations          ├─ ML Platform      │
│  ├─ Growth                 └─ B2B Features        ├─ Developer Exp    │
│  └─ International                                 └─ QA Platform      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.3 チャプター & ギルド

#### 2.3.1 チャプター構成

```yaml
chapters:
  backend_chapter:
    purpose: バックエンドエンジニアの技術的整合性と成長
    lead: Staff Backend Engineer
    members: 全バックエンドエンジニア（スクワッド横断）

    responsibilities:
      technical_alignment:
        - API設計標準
        - データベースパターン
        - パフォーマンス基準
        - セキュリティプラクティス

      knowledge_sharing:
        - 技術ブログ執筆
        - Tech Talk発表
        - コードレビューガイドライン

      career_development:
        - スキル評価
        - メンタリングマッチング
        - 昇格推薦

    cadence:
      chapter_meeting: 隔週 (90分)
      tech_talk: 月1回
      skill_review: 四半期

  frontend_chapter:
    purpose: フロントエンド/モバイルエンジニアの技術的整合性
    lead: Staff Frontend Engineer
    members: 全Flutter/Webエンジニア

    responsibilities:
      technical_alignment:
        - UI/UXガイドライン
        - 状態管理パターン
        - パフォーマンス最適化
        - アクセシビリティ

      knowledge_sharing:
        - Flutterベストプラクティス
        - 新ウィジェット共有
        - デザインシステム進化

  platform_chapter:
    purpose: インフラ・SRE・DevOpsの技術的整合性
    lead: Staff SRE
    members: DevOps, SRE, セキュリティエンジニア

    responsibilities:
      technical_alignment:
        - インフラ標準
        - 監視・アラート基準
        - インシデント対応
        - コスト最適化

  data_chapter:
    purpose: データエンジニア・サイエンティストの整合性
    lead: Staff Data Engineer
    members: DE, DS, MLエンジニア
    timing: Year 2以降に設立
```

#### 2.3.2 ギルド構成

```yaml
guilds:
  security_guild:
    purpose: セキュリティ意識の組織横断的な向上
    coordinator: Security Champion (ローテーション)
    membership: 任意参加

    activities:
      - セキュリティレビュー
      - 脆弱性情報共有
      - セキュリティトレーニング
      - CTF（Capture The Flag）イベント

    cadence: 月1回 (60分)

  quality_guild:
    purpose: 品質文化の醸成
    coordinator: QA Lead
    membership: 任意参加

    activities:
      - テスト戦略共有
      - 自動化ベストプラクティス
      - バグバッシュイベント
      - 品質メトリクスレビュー

    cadence: 月1回 (60分)

  ml_ai_guild:
    purpose: ML/AI知識の共有と実験
    coordinator: ML Engineer (Year 2以降)
    membership: 任意参加

    activities:
      - ML論文読み会
      - プロトタイプ開発
      - ML Ops共有
      - ハッカソン

    cadence: 月2回 (60分)

  frontend_innovation_guild:
    purpose: UI/UX革新とトレンド探求
    coordinator: ローテーション
    membership: 任意参加

    activities:
      - 新技術調査
      - プロトタイピング
      - デザインシステム進化
      - ユーザビリティ研究

    cadence: 月1回 (60分)

  devex_guild:
    purpose: 開発者体験の継続的改善
    coordinator: Platform Engineer
    membership: 任意参加

    activities:
      - 開発環境改善
      - ツール評価
      - CI/CD最適化
      - ドキュメント改善

    cadence: 隔週 (60分)
```

---

## 第3章 チームスケーリング

### 3.1 フェーズ別チーム規模

#### 3.1.1 Phase 1: コアチーム (5→15名)

```yaml
phase_1_core_team:
  timeline: Year 1 Q1-Q2
  headcount:
    start: 5
    end: 15
    growth_rate: ~2名/月

  organizational_structure:
    type: Functional Teams
    teams:
      - Product Team (10)
      - Platform Team (5)

  leadership:
    cto: 1 (existing)
    tech_leads: 2 (to hire)
    em: 0 (CTO兼務)

  hiring_priorities:
    p0_critical:
      - Senior Backend Engineer (2)
      - DevOps Engineer (2)
      - Security Engineer (1)

    p1_high:
      - Senior Flutter Engineer (1)
      - QA Engineer (1)
      - Data Engineer (1)

  key_milestones:
    month_1:
      - DevOps Engineer 1名採用
      - CI/CD基盤強化開始
    month_2:
      - Senior Backend 1名採用
      - セキュリティ基盤整備
    month_3:
      - 残りの採用完了
      - スクワッド構造への移行準備

  challenges:
    - 急成長による文化希薄化
    - ナレッジ転送のボトルネック
    - 採用競争

  mitigations:
    - 強力なオンボーディングプログラム
    - ペアプログラミング必須化
    - 週次All Handsで文化浸透
```

#### 3.1.2 Phase 2: マルチスクワッド (15→30名)

```yaml
phase_2_multi_squad:
  timeline: Year 1 Q3-Q4
  headcount:
    start: 15
    end: 30
    growth_rate: ~3名/月

  organizational_structure:
    type: Squad-based
    squads:
      - Booking Squad (8)
      - Commerce Squad (8)
      - Growth Squad (6)
      - Platform Squad (8)

  leadership:
    cto: 1
    tech_leads: 4 (per squad)
    engineering_manager: 1 (to hire)

  hiring_priorities:
    p0_critical:
      - Engineering Manager (1)
      - Senior Engineers (per squad)

    p1_high:
      - QA Lead (1)
      - Data Scientist (1)
      - Additional Engineers

  coordination:
    scrum_of_scrums:
      frequency: 週2回
      participants: Tech Leads + EM
      purpose: 依存関係解決

    architecture_review:
      frequency: 隔週
      participants: Tech Leads + CTO
      purpose: 技術的整合性

  challenges:
    - スクワッド間の依存関係
    - コミュニケーションコスト増大
    - 標準化 vs 自律性のバランス

  mitigations:
    - 明確なAPI契約
    - 共通のDefinition of Done
    - チャプター活動の活性化
```

#### 3.1.3 Phase 3: トライブ構造 (30→60名)

```yaml
phase_3_tribe_structure:
  timeline: Year 2 H1
  headcount:
    start: 30
    end: 60
    growth_rate: ~5名/月

  organizational_structure:
    type: Tribe-based
    tribes:
      consumer_tribe:
        size: 25
        squads: [Booking, Commerce, Growth, Search]
        lead: Director of Engineering

      merchant_tribe:
        size: 15
        squads: [Supplier, Inventory, Analytics]
        lead: Senior EM

      platform_tribe:
        size: 20
        squads: [Infrastructure, Data, Security]
        lead: Director of Platform

  leadership:
    cto: 1
    directors: 2 (to hire)
    engineering_managers: 5
    tech_leads: 9
    staff_engineers: 2

  new_structures:
    chapters:
      - Backend Chapter (Lead: Staff BE)
      - Frontend Chapter (Lead: Staff FE)
      - Platform Chapter (Lead: Staff SRE)

    guilds:
      - Security Guild
      - Quality Guild
      - DevEx Guild

  coordination:
    tribe_sync:
      frequency: 週1回
      participants: Tribe Lead + Squad Leads
      purpose: トライブレベル調整

    cross_tribe_sync:
      frequency: 隔週
      participants: Tribe Leads + CTO
      purpose: 全社整合性

  challenges:
    - トライブ間のサイロ化
    - 中間管理職の急増
    - 意思決定の遅延リスク

  mitigations:
    - クロストライブプロジェクト
    - ギルドによる横断連携
    - 権限委譲の明確化
```

### 3.2 採用ロードマップ

#### 3.2.1 年間採用計画

```yaml
hiring_roadmap:
  year_1:
    total_hires: 21
    quarterly_breakdown:
      q1:
        hires: 6
        roles:
          - Senior Backend Engineer: 2
          - DevOps Engineer: 2
          - QA Engineer: 1
          - Security Engineer: 1
        priority: Critical infrastructure

      q2:
        hires: 6
        roles:
          - Senior Flutter Engineer: 1
          - Backend Engineer: 2
          - SRE: 1
          - Data Engineer: 1
          - QA Engineer: 1
        priority: Product velocity

      q3:
        hires: 5
        roles:
          - Engineering Manager: 1
          - Senior Backend: 1
          - Flutter Engineer: 2
          - Product Designer: 1
        priority: Team leadership

      q4:
        hires: 4
        roles:
          - Backend Engineer: 1
          - Frontend Engineer: 2
          - Data Scientist: 1
        priority: Growth capabilities

  year_2:
    total_hires: 37
    quarterly_breakdown:
      q1:
        hires: 10
        roles:
          - Director of Engineering: 1
          - Engineering Manager: 1
          - Staff Engineer: 1
          - Backend Engineer: 3
          - Frontend Engineer: 2
          - SRE: 1
          - ML Engineer: 1

      q2:
        hires: 10
        roles:
          - Engineering Manager: 1
          - Backend Engineer: 3
          - Frontend Engineer: 2
          - Data Engineer: 1
          - QA Engineer: 2
          - Security: 1

      q3:
        hires: 9
        roles:
          - Director of Platform: 1
          - Engineering Manager: 1
          - Backend Engineer: 2
          - Frontend Engineer: 2
          - ML Engineer: 1
          - QA Engineer: 2

      q4:
        hires: 8
        roles:
          - Engineering Manager: 1
          - Backend Engineer: 2
          - Frontend Engineer: 2
          - Data Engineer: 1
          - Security: 1
          - DevOps: 1

  year_3:
    total_hires: 70
    focus_areas:
      - Global expansion teams
      - ML/AI specialists
      - Enterprise features
      - Regional engineers
```

### 3.3 スキルマトリックス

#### 3.3.1 必要スキル定義

```yaml
skill_matrix:
  technical_skills:
    frontend:
      flutter:
        levels: [Basic, Intermediate, Advanced, Expert]
        required_distribution:
          expert: 20%
          advanced: 40%
          intermediate: 30%
          basic: 10%

      web_react:
        levels: [Basic, Intermediate, Advanced, Expert]
        timing: Year 2+ (Web expansion)

      state_management:
        technologies: [Provider, Riverpod, BLoC]
        mastery_expectation: all_engineers

    backend:
      typescript:
        levels: [Basic, Intermediate, Advanced, Expert]
        required_distribution:
          expert: 15%
          advanced: 45%
          intermediate: 30%
          basic: 10%

      nodejs:
        frameworks: [Hono, NestJS]
        mastery_expectation: backend_engineers

      go:
        timing: Year 2+ (microservices)
        adoption: new_services_only

      databases:
        primary: PostgreSQL
        additional: [Redis, Elasticsearch]
        mastery_expectation: backend_engineers

    infrastructure:
      kubernetes:
        levels: [Basic, Intermediate, Advanced, Expert]
        required_for: [DevOps, SRE, Platform]

      terraform:
        levels: [Basic, Intermediate, Advanced]
        required_for: [DevOps, Platform]

      gcp:
        services: [GKE, Cloud SQL, Cloud Run, Pub/Sub]
        certification_target: 50%

    data:
      sql:
        expectation: all_engineers
        advanced_for: [Data Engineers]

      analytics:
        tools: [BigQuery, Looker]
        required_for: [Data team, Growth squad]

      ml:
        timing: Year 2+
        required_for: [ML Engineers]

  soft_skills:
    communication:
      levels: [Developing, Proficient, Strong, Exemplary]
      expectation: Proficient minimum

    collaboration:
      levels: [Developing, Proficient, Strong, Exemplary]
      expectation: Proficient minimum

    leadership:
      levels: [Developing, Proficient, Strong, Exemplary]
      expectation_for_senior: Strong minimum

    problem_solving:
      levels: [Developing, Proficient, Strong, Exemplary]
      expectation: Strong minimum
```

---

## 第4章 オンボーディング & 育成

### 4.1 4週間オンボーディングプログラム

#### 4.1.1 オンボーディングジャーニー

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    4週間オンボーディングプログラム                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Week 1: Welcome & Foundation                                           │
│  ═══════════════════════════════════════════════════════════════════    │
│  Day 1: 入社式 & セットアップ                                           │
│  ├─ Welcome kit受け取り                                                │
│  ├─ IT機器セットアップ                                                 │
│  ├─ アカウント作成（Slack, GitHub, GCP等）                             │
│  ├─ セキュリティトレーニング                                           │
│  └─ バディ紹介                                                         │
│                                                                         │
│  Day 2-3: 会社 & プロダクト理解                                         │
│  ├─ 会社説明（ミッション、バリュー、歴史）                             │
│  ├─ プロダクトデモ（ユーザー視点）                                     │
│  ├─ ビジネスモデル説明                                                 │
│  ├─ 組織構造の理解                                                     │
│  └─ チームメンバー1:1開始                                              │
│                                                                         │
│  Day 4-5: 技術基盤理解                                                  │
│  ├─ アーキテクチャ概要                                                 │
│  ├─ 開発環境構築                                                       │
│  ├─ コードベースツアー                                                 │
│  ├─ CI/CDパイプライン理解                                              │
│  └─ 最初のPR作成（typo fix等）                                         │
│                                                                         │
│  Week 2: Deep Dive & First Contribution                                 │
│  ═══════════════════════════════════════════════════════════════════    │
│  Day 6-8: 担当領域の深掘り                                              │
│  ├─ 担当スクワッドの詳細説明                                           │
│  ├─ 関連サービスの理解                                                 │
│  ├─ データモデルの理解                                                 │
│  ├─ Good First Issue着手                                               │
│  └─ ペアプログラミング開始                                             │
│                                                                         │
│  Day 9-10: プロセス習得                                                 │
│  ├─ スクラムセレモニー参加                                             │
│  ├─ コードレビュープロセス                                             │
│  ├─ デプロイメント手順                                                 │
│  └─ 監視・アラート理解                                                 │
│                                                                         │
│  Week 3: Independent Work                                               │
│  ═══════════════════════════════════════════════════════════════════    │
│  ├─ 独立したストーリー担当                                             │
│  ├─ コードレビュー参加（レビュアーとして）                             │
│  ├─ ドキュメント貢献                                                   │
│  ├─ チャプターミーティング参加                                         │
│  └─ 2週目レビュー（マネージャー）                                      │
│                                                                         │
│  Week 4: Full Integration                                               │
│  ═══════════════════════════════════════════════════════════════════    │
│  ├─ フルスプリント参加                                                 │
│  ├─ オンコール見習い                                                   │
│  ├─ 改善提案                                                           │
│  ├─ 30日レビュー（目標設定）                                           │
│  └─ オンボーディング卒業                                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 4.1.2 オンボーディングチェックリスト

```yaml
onboarding_checklist:
  day_1:
    administrative:
      - [ ] 雇用契約書確認
      - [ ] 個人情報登録
      - [ ] 銀行口座登録
      - [ ] 緊急連絡先登録

    equipment:
      - [ ] ノートPC受け取り（MacBook Pro）
      - [ ] モニター（希望者）
      - [ ] キーボード・マウス
      - [ ] ヘッドセット

    access:
      - [ ] Google Workspace
      - [ ] Slack
      - [ ] GitHub
      - [ ] GCP Console
      - [ ] Jira / Linear
      - [ ] Notion
      - [ ] 1Password
      - [ ] VPN設定

  week_1:
    learning:
      - [ ] セキュリティ研修完了
      - [ ] コンプライアンス研修完了
      - [ ] プロダクト研修完了
      - [ ] アーキテクチャ研修完了

    development:
      - [ ] ローカル環境構築
      - [ ] テスト実行確認
      - [ ] 最初のPR作成・マージ

    social:
      - [ ] バディとの1:1 (3回以上)
      - [ ] チームメンバー全員と1:1
      - [ ] ランチ/コーヒー (3回以上)

  week_2:
    contribution:
      - [ ] Good First Issue完了
      - [ ] コードレビュー実施
      - [ ] スタンドアップ発言

    process:
      - [ ] スプリントプランニング参加
      - [ ] レトロスペクティブ参加

  week_3:
    independence:
      - [ ] 独立したストーリー完了
      - [ ] ドキュメント貢献
      - [ ] チャプターミーティング参加

  week_4:
    integration:
      - [ ] フルスプリント貢献
      - [ ] 改善提案1件以上
      - [ ] 30日レビュー完了
      - [ ] 60日目標設定完了
```

### 4.2 メンタリング制度

#### 4.2.1 バディシステム

```yaml
buddy_system:
  assignment:
    timing: 入社前に決定
    duration: 3ヶ月
    criteria:
      - 同じスクワッドまたは近い領域
      - 入社1年以上
      - メンタリング意欲

  responsibilities:
    buddy:
      - 毎日のチェックイン（Week 1-2）
      - 週次1:1（Week 3-12）
      - 質問への即座の対応
      - 社内ナビゲーション支援
      - 文化的なガイダンス

    new_hire:
      - 積極的な質問
      - フィードバックの提供
      - 改善提案

  cadence:
    week_1_2:
      daily_checkin: 15分
      topics:
        - 困っていること
        - 今日の目標
        - ブロッカー

    week_3_12:
      weekly_1on1: 30分
      topics:
        - 進捗確認
        - キャリア相談
        - フィードバック交換

  recognition:
    buddy_allowance: なし（期待される行動）
    excellent_buddy_award: 四半期表彰
```

#### 4.2.2 テクニカルメンタリング

```yaml
technical_mentoring:
  types:
    growth_mentoring:
      purpose: キャリア成長支援
      mentor: シニア以上
      mentee: ジュニア〜ミッド
      frequency: 隔週 (30-60分)
      duration: 6ヶ月〜1年

    skill_mentoring:
      purpose: 特定スキル習得
      mentor: スキル保有者
      mentee: 学習希望者
      frequency: 週1回 (60分)
      duration: 3ヶ月

    leadership_mentoring:
      purpose: リーダーシップ開発
      mentor: Director以上
      mentee: Senior / EM候補
      frequency: 月1回 (60分)
      duration: 6ヶ月〜1年

  matching_process:
    step_1: メンティがメンタリング希望を申請
    step_2: HR/EMがメンター候補をリストアップ
    step_3: 相性確認（カジュアル面談）
    step_4: マッチング決定
    step_5: 目標設定ミーティング

  success_criteria:
    - 目標達成度
    - メンティ満足度
    - スキル成長度
    - 継続希望
```

### 4.3 キャリアラダー

#### 4.3.1 エンジニアリングラダー

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    エンジニアリングキャリアラダー                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│        Individual Contributor              Management                   │
│              Track                           Track                      │
│                                                                         │
│              ┌───────┐                                                  │
│              │  L7   │ Distinguished Engineer                          │
│              │       │ (業界影響力)                                    │
│              └───┬───┘                                                  │
│                  │                                                      │
│              ┌───┴───┐                                                  │
│              │  L6   │ Principal Engineer                              │
│              │       │ (会社全体の技術戦略)                            │
│              └───┬───┘                                                  │
│                  │                         ┌───────┐                   │
│              ┌───┴───┐                     │  M4   │ VP of Engineering │
│              │  L5   │ Staff Engineer      │       │ (組織全体)        │
│              │       │ (組織横断影響)      └───┬───┘                   │
│              └───┬───┘                         │                        │
│                  │                         ┌───┴───┐                   │
│              ┌───┴───┐                     │  M3   │ Director          │
│              │  L4   │ Senior Engineer     │       │ (複数チーム)      │
│              │       │ (チーム影響)        └───┬───┘                   │
│              └───┬───┘                         │                        │
│                  │                         ┌───┴───┐                   │
│              ┌───┴───┐                     │  M2   │ Senior EM         │
│              │  L3   │ Engineer            │       │ (複数スクワッド)  │
│              │       │ (タスク自律)        └───┬───┘                   │
│              └───┬───┘                         │                        │
│                  │                         ┌───┴───┐                   │
│              ┌───┴───┐                     │  M1   │ Engineering Mgr   │
│              │  L2   │ Junior Engineer     │       │ (1スクワッド)     │
│              │       │ (指導下で作業)      └───────┘                   │
│              └───────┘                                                  │
│                                                                         │
│  転換ポイント:                                                          │
│  • L3→L4 昇格時にIC継続 or マネジメントを選択                         │
│  • M1→L4/L5 への転換も可能                                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 4.3.2 レベル別期待値

```yaml
level_expectations:
  l2_junior:
    experience: 0-2年
    scope: タスク
    autonomy: 指示のもと作業
    impact: 個人
    technical:
      - 基本的なコーディング
      - テスト作成
      - ドキュメント理解
    behavioral:
      - 積極的な質問
      - フィードバック受容
      - チーム規範の習得
    compensation: 400-600万円

  l3_engineer:
    experience: 2-4年
    scope: 機能
    autonomy: 方向性のもと自律
    impact: チーム
    technical:
      - 独立した機能開発
      - 設計参加
      - コードレビュー
    behavioral:
      - 問題解決
      - 協働
      - ジュニアサポート
    compensation: 600-900万円

  l4_senior:
    experience: 4-7年
    scope: プロジェクト
    autonomy: 目標設定後は自律
    impact: 複数チーム
    technical:
      - 技術的意思決定
      - 設計リード
      - 複雑な問題解決
    behavioral:
      - メンタリング
      - プロセス改善
      - ステークホルダー連携
    compensation: 900-1,300万円

  l5_staff:
    experience: 7-10年
    scope: 組織
    autonomy: 自ら目標設定
    impact: 組織全体
    technical:
      - アーキテクチャ決定
      - 技術戦略
      - 業界ベストプラクティス
    behavioral:
      - 組織横断リード
      - 外部発信
      - 文化形成
    compensation: 1,200-1,700万円

  l6_principal:
    experience: 10年以上
    scope: 会社/業界
    autonomy: 戦略立案
    impact: 会社・業界
    technical:
      - 技術ビジョン策定
      - 業界影響力
      - イノベーション
    behavioral:
      - エグゼクティブ連携
      - 組織設計
      - 業界リーダーシップ
    compensation: 1,700-2,500万円

  m1_em:
    experience: 5年以上（うちマネジメント1年以上）
    scope: 1スクワッド（6-10人）
    technical:
      - 技術的判断力
      - コードレビュー能力
    people:
      - 1:1実施
      - パフォーマンス管理
      - 採用参加
    delivery:
      - スプリント管理
      - ブロッカー除去
    compensation: 1,100-1,600万円
```

---

## 第5章 コラボレーション & カルチャー

### 5.1 リモートワークプロトコル

#### 5.1.1 ワークスタイルポリシー

```yaml
work_style_policy:
  model: Hybrid-First

  options:
    full_remote:
      eligibility: 全ポジション
      requirements:
        - 日本国内居住
        - 適切なインターネット環境
        - 集中できる作業環境
      office_requirement: 四半期1回のオフサイト参加

    hybrid:
      eligibility: 全ポジション
      default_schedule: 週2-3日オフィス
      flexibility: チームで調整

    full_office:
      eligibility: 希望者
      benefits: 固定デスク、設備充実

  core_hours:
    timezone: JST
    core_time: 11:00-16:00
    flexibility: コアタイム以外は自由
    exceptions:
      - 顧客対応チーム: シフト制
      - 国際チーム: オーバーラップ調整

  communication_expectations:
    response_time:
      slack:
        urgent: 30分以内
        normal: 4時間以内
        async: 24時間以内
      email:
        normal: 24時間以内

    availability:
      calendar_blocking: 必須
      status_update: Slack/カレンダー連携
      out_of_office: 事前通知必須

  equipment_support:
    provided:
      - ノートPC（MacBook Pro）
      - モニター（1-2台）
      - キーボード・マウス
      - ヘッドセット
      - Webカメラ

    allowance:
      remote_work: 3万円/月
      internet: 含む
      desk_chair: 初回5万円補助
```

#### 5.1.2 非同期コミュニケーション優先

```yaml
async_first_communication:
  principles:
    default_async:
      description: デフォルトは非同期、必要な場合のみ同期
      benefits:
        - 深い作業時間の確保
        - タイムゾーン対応
        - 記録の自動化

    documentation_culture:
      description: 決定事項は必ず文書化
      tools:
        - Notion: 長期ドキュメント
        - Slack: 日常コミュニケーション
        - ADR: アーキテクチャ決定

  channel_strategy:
    slack:
      synchronous_channels:
        - "#incidents" (緊急対応)
        - "#standup-{squad}" (デイリー)

      async_channels:
        - "#engineering" (全体共有)
        - "#squad-{name}" (スクワッド)
        - "#chapter-{name}" (チャプター)
        - "#random" (雑談)
        - "#til" (Today I Learned)

      thread_policy: 必ずスレッドで返信

    meetings:
      must_have_agenda: true
      default_duration: 25分 or 50分
      recording: 推奨（参加できない人向け）
      meeting_free_day: 水曜日

  documentation_requirements:
    meeting_outcomes:
      - 決定事項
      - アクションアイテム
      - 次のステップ

    project_updates:
      - 週次進捗（Notion）
      - ブロッカー共有
      - 成果物リンク
```

### 5.2 ツール選定

#### 5.2.1 コラボレーションツールスタック

```yaml
collaboration_tools:
  communication:
    primary: Slack Enterprise Grid
    features:
      - チャンネルベースコミュニケーション
      - Huddle（音声チャット）
      - ワークフロービルダー
    integrations:
      - GitHub
      - Jira
      - PagerDuty
      - Google Calendar

    secondary: Google Meet
    use_cases:
      - ビデオ会議
      - 画面共有
      - 録画

  documentation:
    primary: Notion
    structure:
      - Engineering Wiki
      - Squad Spaces
      - Project Documentation
      - Onboarding Hub
      - ADRs (Architecture Decision Records)

    code_documentation:
      - README.md (per repository)
      - API Documentation (OpenAPI)
      - Inline comments (必要時のみ)

  project_management:
    primary: Linear (または Jira)
    features:
      - スプリント管理
      - バックログ管理
      - ロードマップ
    integration:
      - GitHub (auto-link)
      - Slack (notifications)

  design:
    primary: Figma
    use_cases:
      - UI/UXデザイン
      - プロトタイピング
      - デザインシステム
      - ホワイトボード

  code:
    primary: GitHub Enterprise
    features:
      - ソースコード管理
      - Pull Request
      - Code Review
      - GitHub Actions
      - GitHub Copilot

  knowledge_sharing:
    internal:
      - Notion (documentation)
      - Slack #til (quick tips)
      - Tech Talks (recordings)

    external:
      - Zenn (tech blog)
      - Speaker Deck (presentations)
```

### 5.3 エンジニアリングカルチャー

#### 5.3.1 コアバリュー

```yaml
engineering_values:
  ownership:
    name: "You Build It, You Run It"
    description: 作ったものに責任を持つ
    behaviors:
      - 自分のコードの本番動作を監視
      - インシデント対応への参加
      - 継続的な改善提案
    anti_patterns:
      - "それは運用チームの仕事"
      - "デプロイしたら終わり"

  collaboration:
    name: "Better Together"
    description: 協力してより良いものを作る
    behaviors:
      - ペアプログラミング
      - 知識共有
      - 建設的なコードレビュー
    anti_patterns:
      - サイロ化
      - 情報の囲い込み

  learning:
    name: "Always Learning"
    description: 継続的に学び、成長する
    behaviors:
      - 新技術の探求
      - 失敗からの学習
      - フィードバックの受容
    anti_patterns:
      - "今のやり方で十分"
      - 失敗を隠す

  simplicity:
    name: "Keep It Simple"
    description: シンプルな解決策を選ぶ
    behaviors:
      - YAGNI原則
      - 過度な抽象化を避ける
      - 読みやすいコード
    anti_patterns:
      - 過剰エンジニアリング
      - 早すぎる最適化

  customer_focus:
    name: "Customer First"
    description: 顧客価値を最優先
    behaviors:
      - ユーザーフィードバックの重視
      - データ駆動型意思決定
      - 迅速な価値提供
    anti_patterns:
      - 技術的興味だけで判断
      - ユーザー不在の議論
```

#### 5.3.2 カルチャー醸成活動

```yaml
culture_activities:
  weekly:
    all_hands:
      frequency: 週1回
      duration: 30分
      content:
        - 会社アップデート
        - プロダクトデモ
        - 新メンバー紹介
        - 表彰

    tech_talk:
      frequency: 週1回
      duration: 45分
      content:
        - 技術的深掘り
        - 新技術紹介
        - ポストモーテム共有
      speakers: ローテーション

  monthly:
    demo_day:
      frequency: 月1回
      duration: 2時間
      content:
        - スクワッドデモ
        - ハックプロジェクト発表
        - 実験結果共有

    lunch_learn:
      frequency: 月2回
      duration: 1時間
      content:
        - 外部スピーカー
        - 書籍紹介
        - キャリア共有

  quarterly:
    hackathon:
      frequency: 四半期1回
      duration: 2日間
      rules:
        - チーム自由編成
        - 通常業務外のプロジェクト
        - 発表・投票・表彰

    offsite:
      frequency: 四半期1回
      duration: 1日
      content:
        - 戦略共有
        - チームビルディング
        - 振り返り

  recognition:
    peer_recognition:
      platform: Slack #kudos
      frequency: いつでも
      categories:
        - Great Collaboration
        - Technical Excellence
        - Customer Impact
        - Learning & Growth

    engineering_awards:
      frequency: 四半期
      categories:
        - Innovator of the Quarter
        - Quality Champion
        - Best Mentor
        - Culture Carrier
```

---

## 第6章 実装ロードマップ & 文書間参照

### 6.1 組織スケーリングタイムライン

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    組織スケーリングタイムライン                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Year 1                                                                 │
│  ═══════════════════════════════════════════════════════════════════    │
│  Q1: コアチーム強化                                                     │
│  ├─ DevOps/Security人材採用                                            │
│  ├─ オンボーディングプログラム確立                                     │
│  ├─ バディシステム開始                                                 │
│  └─ チーム文化の明文化                                                 │
│                                                                         │
│  Q2: スクワッド移行準備                                                 │
│  ├─ Tech Lead採用・育成                                                │
│  ├─ スクワッド定義                                                     │
│  ├─ チャプター構造設計                                                 │
│  └─ キャリアラダー公開                                                 │
│                                                                         │
│  Q3: マルチスクワッド運用                                               │
│  ├─ 4スクワッド体制開始                                                │
│  ├─ EM初採用                                                           │
│  ├─ チャプター活動開始                                                 │
│  └─ Scrum of Scrums導入                                                │
│                                                                         │
│  Q4: 最適化                                                             │
│  ├─ スクワッド間調整改善                                               │
│  ├─ ギルド活動開始                                                     │
│  ├─ 採用プロセス最適化                                                 │
│  └─ Year 2計画策定                                                     │
│                                                                         │
│  Year 2                                                                 │
│  ═══════════════════════════════════════════════════════════════════    │
│  H1: トライブ構造導入                                                   │
│  ├─ Director採用                                                       │
│  ├─ 3トライブ体制開始                                                  │
│  ├─ チャプターリード任命                                               │
│  └─ クロストライブ調整確立                                             │
│                                                                         │
│  H2: 国際展開対応                                                       │
│  ├─ リモートファースト強化                                             │
│  ├─ タイムゾーン対応                                                   │
│  ├─ 多言語チーム対応                                                   │
│  └─ グローバル採用開始                                                 │
│                                                                         │
│  Year 3                                                                 │
│  ═══════════════════════════════════════════════════════════════════    │
│  ├─ マルチトライブ運用成熟                                             │
│  ├─ リージョナルチーム設立                                             │
│  ├─ 組織の自己進化能力確立                                             │
│  └─ 150名規模での持続可能な成長                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 主要マイルストーン

```yaml
organization_milestones:
  m1_core_team_complete:
    date: Year 1 Q1 End
    criteria:
      - 15名体制達成
      - DevOps/Security人材確保
      - オンボーディングプログラム稼働
    success_metrics:
      - 採用目標達成率: 100%
      - オンボーディング満足度: 4.0+

  m2_squad_operational:
    date: Year 1 Q3
    criteria:
      - 4スクワッド体制
      - Tech Lead全スクワッドに配置
      - スクワッド自律的に運用
    success_metrics:
      - スクワッド間依存チケット: <10%
      - デリバリー速度維持

  m3_tribe_structure:
    date: Year 2 Q2
    criteria:
      - 3トライブ体制
      - Director 2名以上
      - チャプター活性化
    success_metrics:
      - 技術標準遵守率: 90%+
      - チャプター参加率: 80%+

  m4_scaled_organization:
    date: Year 2 Q4
    criteria:
      - 60名体制
      - リモートファースト確立
      - グローバル採用開始
    success_metrics:
      - リテンション率: 90%+
      - eNPS: 40+

  m5_global_ready:
    date: Year 3 Q4
    criteria:
      - 150名体制
      - 複数タイムゾーン運用
      - 組織の自己進化
    success_metrics:
      - 採用効率維持
      - 生産性維持
      - 文化維持
```

### 6.3 関連文書参照

```yaml
document_references:
  directly_related:
    - Doc-DM-001: アジャイルデリバリーフレームワーク
    - Doc-DM-002: DevOps＆継続的デリバリー
    - Doc-IR-002: リソース＆キャパシティプランニング

  implementation:
    - Doc-IR-001: 開発ロードマップ Year 1-3
    - Doc-QA-001: テスト戦略＆品質基準

  technical:
    - Doc-SA-001: システムアーキテクチャ
    - EXISTING_APP_ANALYSIS.md: 既存アプリ分析

  business:
    - Doc-OS-001: オペレーションモデル
    - Doc-FP-006: 人件費計画
```

### 6.4 成功基準サマリー

```yaml
success_criteria:
  organization_health:
    year_1:
      headcount: 26名
      voluntary_turnover: <15%
      enps: 30+
      diversity_goals: defined

    year_2:
      headcount: 63名
      voluntary_turnover: <12%
      enps: 40+
      diversity_goals: progress

    year_3:
      headcount: 133名
      voluntary_turnover: <10%
      enps: 50+
      diversity_goals: achieved

  team_effectiveness:
    squad_autonomy:
      metric: 外部依存チケット率
      target: <10%

    delivery_velocity:
      metric: ストーリーポイント/スプリント
      target: 安定または向上

    quality:
      metric: 本番バグ率
      target: <1%

  culture:
    knowledge_sharing:
      metric: Tech Talk参加率
      target: 70%+

    collaboration:
      metric: ペアプログラミング実施率
      target: 週1回以上

    learning:
      metric: 学習時間
      target: 週4時間以上
```

---

## 付録

### 付録A: スクワッドチャーターテンプレート

```yaml
squad_charter_template:
  squad_name: "[スクワッド名]"
  mission: "[一文でのミッション定義]"

  scope:
    owns:
      - "[所有するサービス/機能]"
    influences:
      - "[影響を与える領域]"
    out_of_scope:
      - "[対象外の領域]"

  members:
    product_owner: "[名前]"
    tech_lead: "[名前]"
    engineers:
      - "[名前] - [専門]"
    qa: "[名前]"
    designer: "[名前] (shared)"

  stakeholders:
    primary: "[主要ステークホルダー]"
    secondary: "[副次ステークホルダー]"

  success_metrics:
    - metric: "[KPI名]"
      target: "[目標値]"
      current: "[現在値]"

  communication:
    slack_channel: "#squad-[name]"
    meeting_cadence:
      standup: "毎日 10:00"
      planning: "隔週月曜 10:00"
      retro: "隔週金曜 15:00"

  dependencies:
    incoming:
      - "[依存元スクワッド]: [内容]"
    outgoing:
      - "[依存先スクワッド]: [内容]"

  working_agreements:
    - "[合意事項1]"
    - "[合意事項2]"
```

### 付録B: チャプターミーティングアジェンダ

```yaml
chapter_meeting_agenda:
  frequency: 隔週
  duration: 90分

  agenda:
    - item: "前回アクション確認"
      duration: 10分

    - item: "技術共有"
      duration: 30分
      format: ローテーション発表

    - item: "課題・ブロッカー共有"
      duration: 20分

    - item: "標準化議論"
      duration: 20分

    - item: "アクションアイテム決定"
      duration: 10分
```

### 付録C: 1:1ミーティングガイド

```yaml
one_on_one_guide:
  frequency: 週1回
  duration: 30分
  owner: レポート（部下）

  topics:
    regular:
      - 今週のハイライト/ローライト
      - ブロッカー・支援要請
      - フィードバック交換
      - キャリア・成長

    periodic:
      - 目標進捗確認（月次）
      - スキル評価（四半期）
      - キャリア計画（半期）

  manager_responsibilities:
    - アクティブリスニング
    - コーチング姿勢
    - フォローアップ
    - 記録保持

  anti_patterns:
    - ステータス報告会にする
    - 一方的に話す
    - キャンセルが多い
    - フォローアップしない
```

---

**文書情報**

| 項目 | 内容 |
|------|------|
| Document ID | Doc-DM-003 |
| Version | 1.0.0 |
| Created | 2026-01-20 |
| Last Updated | 2026-01-20 |
| Status | Draft |
| Owner | Implementation Agent |
| Reviewers | CTO, VP Engineering, HR |
| Total Lines | 1,900+ |

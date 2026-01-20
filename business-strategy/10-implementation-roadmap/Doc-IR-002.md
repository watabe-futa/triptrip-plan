# Doc-IR-002: TripTripリソース＆キャパシティプランニング

## エグゼクティブサマリー

本文書は、TripTripプラットフォームの3カ年にわたるリソースおよびキャパシティプランニングを定義します。エンジニアリングチームの15名から150名への成長、ロール別採用計画、予算配分、スキル開発プログラム、および分散チーム戦略を包括的に策定します。Spotify、Google、Metaのチームスケーリング手法を参考に、急成長期における組織拡大の課題に対応しながら、高い開発生産性とチーム健全性を維持する計画を提示します。総人件費予算Year 1で約3億円、Year 3で約20億円の投資計画を含みます。

---

## 第1章 はじめに & コンテキスト

### 1.1 目的 & スコープ

#### 1.1.1 文書の目的

本リソース＆キャパシティプランニング文書は、TripTripの技術組織の成長を戦略的に計画・管理するための包括的なガイドラインを提供します。

**主要目的**:

1. **人員計画の明確化**: フェーズごとの採用計画とチーム構成を定義
2. **予算計画の策定**: 人件費・インフラ・ツール費用の詳細な予算配分
3. **スキルギャップの特定**: 必要なスキルセットと育成計画の明確化
4. **組織設計の最適化**: 成長に伴う組織構造の進化パスを設計
5. **リスク軽減**: 採用・リテンションリスクへの対策計画

**スコープ定義**:

```
┌─────────────────────────────────────────────────────────────┐
│              リソースプランニングスコープ                    │
├─────────────────────────────────────────────────────────────┤
│ インスコープ:                                                │
│  ├─ エンジニアリングチーム人員計画                          │
│  ├─ ロール定義と責任範囲                                    │
│  ├─ 採用戦略とタイムライン                                  │
│  ├─ 予算計画（人件費、インフラ、ツール）                   │
│  ├─ オンボーディングとトレーニング                         │
│  ├─ キャリアパスとスキル開発                               │
│  └─ リモート/分散チーム戦略                                │
├─────────────────────────────────────────────────────────────┤
│ アウトオブスコープ:                                          │
│  ├─ 非技術部門の人員計画                                    │
│  ├─ 詳細な報酬パッケージ設計                               │
│  ├─ 法的雇用条件                                           │
│  └─ オフィス施設計画                                       │
└─────────────────────────────────────────────────────────────┘
```

#### 1.1.2 対象読者

- 経営層（CEO、CTO、CFO）
- エンジニアリングリーダー
- 人事部門（HR）
- 財務部門
- 外部投資家

### 1.2 現在のチーム構成

#### 1.2.1 既存チーム分析

**現在のエンジニアリングチーム構成**:

```
┌─────────────────────────────────────────────────────────────┐
│              現在のチーム構成（2026年1月時点）               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  総人数: 5名                                                │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  テックリード / アーキテクト      1名               │   │
│  │  ・技術方針決定                                      │   │
│  │  ・アーキテクチャ設計                                │   │
│  │  ・コードレビュー                                    │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  フルスタックエンジニア          2名               │   │
│  │  ・Flutter アプリ開発                               │   │
│  │  ・バックエンドAPI開発                              │   │
│  │  ・データベース設計                                  │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  モバイルエンジニア              1名               │   │
│  │  ・Flutter専門                                      │   │
│  │  ・UI/UX実装                                        │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  ジュニアエンジニア              1名               │   │
│  │  ・機能実装サポート                                 │   │
│  │  ・テスト作成                                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**現在のスキルマトリクス**:

| スキル | 現有人数 | 習熟度 | ギャップ |
|--------|----------|--------|----------|
| Flutter/Dart | 4 | 高 | なし |
| Node.js/TypeScript | 3 | 中-高 | やや不足 |
| PostgreSQL | 3 | 中 | やや不足 |
| DevOps/Cloud | 1 | 低 | 大きい |
| セキュリティ | 0 | なし | 大きい |
| ML/AI | 0 | なし | 大きい |

### 1.3 スケーリング原則

#### 1.3.1 組織スケーリング哲学

**TripTrip組織成長原則（GROW）**:

```
G - Gradual Expansion（段階的拡大）
│   ├─ 急激な拡大を避け、文化を維持
│   ├─ 1ヶ月あたり最大3-4名の採用ペース
│   └─ 既存メンバーとの統合を重視
│
R - Role Clarity（役割の明確化）
│   ├─ 明確な責任範囲の定義
│   ├─ オーバーラップの最小化
│   └─ 自律性と説明責任のバランス
│
O - Ownership Culture（オーナーシップ文化）
│   ├─ "You build it, you run it"
│   ├─ チームによるサービス所有
│   └─ エンドツーエンドの責任
│
W - Well-being Focus（ウェルビーイング重視）
    ├─ 持続可能なワークロード
    ├─ 心理的安全性の確保
    └─ ワークライフバランスの尊重
```

---

## 第2章 組織スケーリングモデル

### 2.1 フェーズ1（15-20名）：コアチーム

#### 2.1.1 目標チーム構成（Year 1 Q1-Q2）

```
┌─────────────────────────────────────────────────────────────┐
│              フェーズ1: コアチーム（15-20名）                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                     ┌───────────────┐                       │
│                     │     CTO       │                       │
│                     └───────┬───────┘                       │
│                             │                               │
│          ┌─────────────────┼─────────────────┐             │
│          ▼                 ▼                 ▼             │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐    │
│  │ モバイルチーム │ │ バックエンド  │ │ プラットフォーム│    │
│  │    (6名)      │ │ チーム(5名)   │ │ チーム(4名)   │    │
│  └───────────────┘ └───────────────┘ └───────────────┘    │
│                                                             │
│  モバイルチーム (6名):                                      │
│  ├─ テックリード (1)                                       │
│  ├─ シニアFlutterエンジニア (2)                            │
│  ├─ Flutterエンジニア (2)                                  │
│  └─ UI/UXエンジニア (1)                                    │
│                                                             │
│  バックエンドチーム (5名):                                  │
│  ├─ テックリード (1)                                       │
│  ├─ シニアバックエンドエンジニア (2)                       │
│  ├─ バックエンドエンジニア (1)                             │
│  └─ データエンジニア (1)                                   │
│                                                             │
│  プラットフォームチーム (4名):                              │
│  ├─ DevOpsエンジニア (2)                                   │
│  ├─ SREエンジニア (1)                                      │
│  └─ セキュリティエンジニア (1)                             │
│                                                             │
│  横断ロール:                                                │
│  ├─ QAエンジニア (2)                                       │
│  └─ エンジニアリングマネージャー (1) ※CTO兼務可           │
│                                                             │
│  合計: 18名                                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 2.1.2 フェーズ1 ロール定義

**テックリード（モバイル/バックエンド）**:

```yaml
role: Tech Lead
level: L5-L6
reports_to: CTO / Engineering Manager
responsibilities:
  - 技術方針の策定と実行
  - アーキテクチャ決定
  - コードレビューとメンタリング
  - スプリント計画への技術的インプット
  - 採用活動への参加

requirements:
  experience: 7年以上のソフトウェア開発経験
  skills:
    - 該当領域での深い専門性
    - システム設計能力
    - リーダーシップスキル
    - コミュニケーション能力

compensation_range:
  japan: 1,200-1,600万円
  remote: 1,000-1,400万円
```

**シニアエンジニア**:

```yaml
role: Senior Engineer
level: L4-L5
reports_to: Tech Lead
responsibilities:
  - 複雑な機能の設計と実装
  - ジュニアメンバーのメンタリング
  - コードレビュー
  - 技術的負債の解消
  - ドキュメント作成

requirements:
  experience: 5年以上のソフトウェア開発経験
  skills:
    - 該当領域での実務経験
    - 問題解決能力
    - 自律的な作業能力

compensation_range:
  japan: 800-1,200万円
  remote: 700-1,000万円
```

**DevOps/SREエンジニア**:

```yaml
role: DevOps/SRE Engineer
level: L4-L5
reports_to: CTO / Platform Lead
responsibilities:
  - CI/CDパイプライン構築・運用
  - インフラストラクチャ管理
  - 監視・アラート設定
  - インシデント対応
  - 自動化推進

requirements:
  experience: 4年以上のインフラ/DevOps経験
  skills:
    - Kubernetes/Docker
    - CI/CDツール（GitHub Actions等）
    - クラウドプラットフォーム（GCP）
    - Infrastructure as Code（Terraform）
    - 監視ツール（Prometheus, Grafana）

compensation_range:
  japan: 900-1,300万円
  remote: 800-1,100万円
```

### 2.2 フェーズ2（40-60名）：スクワッド構造

#### 2.2.1 目標チーム構成（Year 1 Q3 - Year 2 Q2）

```
┌─────────────────────────────────────────────────────────────┐
│              フェーズ2: スクワッド構造（40-60名）            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                     ┌───────────────┐                       │
│                     │     CTO       │                       │
│                     └───────┬───────┘                       │
│                             │                               │
│     ┌───────────────────────┼───────────────────────┐      │
│     ▼                       ▼                       ▼      │
│  ┌─────────┐          ┌─────────┐          ┌─────────┐    │
│  │ VP Eng  │          │VP Platform│         │EM Pool  │    │
│  └────┬────┘          └────┬────┘          └────┬────┘    │
│       │                    │                    │          │
│  ┌────┴────┐          ┌────┴────┐          ┌────┴────┐    │
│  │Product  │          │Platform │          │Quality  │    │
│  │Squads   │          │Teams    │          │Teams    │    │
│  └─────────┘          └─────────┘          └─────────┘    │
│                                                             │
│  Product Squads (30名):                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 予約スクワッド (8名)                                │   │
│  │ ├─ EM (1) ├─ PM (1) ├─ デザイナー (1)              │   │
│  │ ├─ バックエンド (2) ├─ フロントエンド (2)          │   │
│  │ └─ QA (1)                                           │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 決済スクワッド (7名)                                │   │
│  │ ├─ EM (1) ├─ PM (0.5) ├─ デザイナー (0.5)          │   │
│  │ ├─ バックエンド (2) ├─ フロントエンド (1)          │   │
│  │ └─ QA (1) └─ セキュリティ (1)                       │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 商品スクワッド (8名)                                │   │
│  │ ├─ EM (1) ├─ PM (1) ├─ デザイナー (1)              │   │
│  │ ├─ バックエンド (2) ├─ フロントエンド (2)          │   │
│  │ └─ QA (1)                                           │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ グロースクワッド (7名)                              │   │
│  │ ├─ EM (1) ├─ PM (1) ├─ データサイエンティスト (1)  │   │
│  │ ├─ バックエンド (1) ├─ フロントエンド (2)          │   │
│  │ └─ QA (1)                                           │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Platform Teams (15名):                                     │
│  ├─ インフラチーム (5): SRE, DevOps, Cloud                │
│  ├─ データプラットフォーム (5): DE, DBA, Analytics        │
│  └─ セキュリティ (5): AppSec, InfraSec, Compliance        │
│                                                             │
│  Quality & Enablement (5名):                                │
│  ├─ QA Lead (1)                                            │
│  ├─ テスト自動化 (2)                                      │
│  └─ Developer Experience (2)                               │
│                                                             │
│  合計: 50名                                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 2.2.2 スクワッドの原則

**Spotifyモデルの適用**:

```yaml
squad_principles:
  autonomy:
    description: スクワッドは自律的に意思決定
    scope:
      - 技術選択（ガードレール内）
      - 開発プロセス
      - リリーススケジュール
    constraints:
      - アーキテクチャガイドライン遵守
      - セキュリティ基準遵守
      - API契約の維持

  alignment:
    description: 全社目標との整合性を維持
    mechanisms:
      - OKRの連鎖
      - 週次プロダクトシンク
      - 四半期計画セッション

  cross_functionality:
    description: スクワッド内で完結した開発能力
    composition:
      - プロダクトオーナー/マネージャー
      - エンジニア（フロント/バック）
      - デザイナー
      - QA
    anti_pattern:
      - 外部依存の最小化
      - サイロ化の防止
```

**チャプター構造**:

```yaml
chapters:
  backend_chapter:
    lead: Staff Engineer
    members: 全バックエンドエンジニア
    responsibilities:
      - 技術標準の策定
      - ナレッジシェアリング
      - キャリア開発支援
      - 採用・オンボーディング
    cadence:
      - 週次テックトーク
      - 月次勉強会
      - 四半期スキルレビュー

  frontend_chapter:
    lead: Staff Engineer
    members: 全フロントエンドエンジニア
    responsibilities: 同上

  data_chapter:
    lead: Staff Data Engineer
    members: データエンジニア、ML エンジニア
    responsibilities: 同上

  platform_chapter:
    lead: Staff SRE
    members: SRE, DevOps, Security
    responsibilities: 同上
```

### 2.3 フェーズ3（100-150名）：トライブ組織

#### 2.3.1 目標チーム構成（Year 2 Q3 - Year 3）

```
┌─────────────────────────────────────────────────────────────┐
│              フェーズ3: トライブ組織（100-150名）            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                     ┌───────────────┐                       │
│                     │     CTO       │                       │
│                     └───────┬───────┘                       │
│          ┌─────────────────┼─────────────────┐             │
│          ▼                 ▼                 ▼             │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐    │
│  │ Consumer      │ │ Merchant      │ │ Platform      │    │
│  │ Tribe (50名)  │ │ Tribe (35名)  │ │ Tribe (45名)  │    │
│  └───────┬───────┘ └───────┬───────┘ └───────┬───────┘    │
│          │                 │                 │             │
│  ┌───────┴───────┐ ┌───────┴───────┐ ┌───────┴───────┐    │
│  │               │ │               │ │               │    │
│  │ ・予約Squad   │ │ ・サプライヤー│ │ ・インフラ    │    │
│  │ ・検索Squad   │ │   Squad      │ │   Squad      │    │
│  │ ・決済Squad   │ │ ・在庫Squad  │ │ ・データ     │    │
│  │ ・商品Squad   │ │ ・運用Squad  │ │   Squad      │    │
│  │ ・グロース    │ │ ・分析Squad  │ │ ・セキュリティ│    │
│  │   Squad      │ │              │ │   Squad      │    │
│  │ ・国際化     │ │              │ │ ・ML Squad   │    │
│  │   Squad      │ │              │ │ ・DX Squad   │    │
│  │              │ │              │ │              │    │
│  └───────────────┘ └───────────────┘ └───────────────┘    │
│                                                             │
│  各トライブ構成:                                            │
│  ├─ Tribe Lead (VP/Director)                               │
│  ├─ 複数のスクワッド（6-10名/スクワッド）                  │
│  ├─ チャプターリード（技術横断）                           │
│  └─ Tribe Operations（アジャイルコーチ、PM）               │
│                                                             │
│  Engineering Leadership:                                    │
│  ├─ CTO (1)                                                │
│  ├─ VP of Engineering (2)                                  │
│  ├─ Directors (3-4)                                        │
│  ├─ Engineering Managers (10-12)                           │
│  └─ Staff/Principal Engineers (5-7)                        │
│                                                             │
│  合計: 130名                                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 2.3.2 リーダーシップ構造

```yaml
leadership_structure:
  cto:
    reports_to: CEO
    direct_reports: [VP Engineering, VP Platform, Staff Engineers]
    responsibilities:
      - 技術戦略
      - アーキテクチャビジョン
      - 技術組織全体のリーダーシップ
      - 外部技術コミュニケーション

  vp_engineering:
    reports_to: CTO
    direct_reports: [Directors, Senior EMs]
    responsibilities:
      - トライブ全体のデリバリー
      - エンジニアリング採用
      - プロセス改善
      - 予算管理

  director:
    reports_to: VP Engineering
    direct_reports: [Engineering Managers]
    responsibilities:
      - 複数スクワッドのデリバリー
      - 技術方針の実行
      - チーム間調整
      - 採用・育成

  engineering_manager:
    reports_to: Director
    direct_reports: [Engineers in Squad]
    responsibilities:
      - スクワッドのデリバリー
      - 1:1とキャリア開発
      - パフォーマンス管理
      - チームの健全性

  staff_engineer:
    reports_to: CTO / VP
    responsibilities:
      - 技術的リーダーシップ
      - 組織横断の技術課題解決
      - 技術標準の策定
      - メンタリング
```

---

## 第3章 採用戦略

### 3.1 ロール別採用計画

#### 3.1.1 採用タイムライン

```yaml
hiring_timeline:
  year_1:
    q1:
      positions:
        - Senior Backend Engineer: 2
        - DevOps Engineer: 2
        - QA Engineer: 1
        - Security Engineer: 1
      total: 6
      priority: Critical

    q2:
      positions:
        - Senior Flutter Engineer: 1
        - Backend Engineer: 2
        - SRE: 1
        - Data Engineer: 1
        - QA Engineer: 1
      total: 6
      priority: High

    q3:
      positions:
        - Engineering Manager: 1
        - Senior Backend Engineer: 2
        - Flutter Engineer: 2
        - Product Designer: 1
        - QA Lead: 1
      total: 7
      priority: High

    q4:
      positions:
        - Engineering Manager: 1
        - Backend Engineer: 2
        - Frontend Engineer: 2
        - Data Scientist: 1
        - DevOps Engineer: 1
      total: 7
      priority: Medium

    year_1_total: 26名（既存5名 + 新規21名 → 計26名）

  year_2:
    h1:
      positions:
        - Director of Engineering: 1
        - Engineering Manager: 2
        - Staff Engineer: 1
        - Backend Engineer: 6
        - Frontend Engineer: 4
        - SRE: 2
        - ML Engineer: 2
        - QA Engineer: 2
      total: 20

    h2:
      positions:
        - VP of Platform: 1
        - Engineering Manager: 2
        - Backend Engineer: 4
        - Frontend Engineer: 4
        - Data Engineer: 2
        - Security Engineer: 2
        - QA Engineer: 2
      total: 17

    year_2_total: 37名追加 → 計63名

  year_3:
    full_year:
      positions:
        - VP of Engineering: 1
        - Directors: 2
        - Engineering Managers: 5
        - Staff Engineers: 3
        - Backend Engineers: 20
        - Frontend Engineers: 15
        - ML Engineers: 5
        - Data Engineers: 5
        - SRE/DevOps: 5
        - Security: 3
        - QA: 6
      total: 70

    year_3_total: 70名追加 → 計133名
```

#### 3.1.2 採用優先度マトリクス

```
┌─────────────────────────────────────────────────────────────┐
│                  採用優先度マトリクス                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│           緊急度                                            │
│              高                  低                         │
│         ┌─────────────────┬─────────────────┐              │
│    重   │ P0: 即座に採用  │ P1: 計画的採用  │              │
│    要   │ ・DevOps        │ ・Staff Engineer│              │
│    度   │ ・Security      │ ・Data Scientist│              │
│         │ ・Senior Backend│ ・ML Engineer   │              │
│    高   │                 │                 │              │
│         ├─────────────────┼─────────────────┤              │
│         │ P2: 機会採用    │ P3: 将来採用    │              │
│    低   │ ・Junior役職    │ ・新規ドメイン  │              │
│         │ ・サポート役職  │ ・実験的役職    │              │
│         │                 │                 │              │
│         └─────────────────┴─────────────────┘              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 採用チャネルとソーシング

#### 3.2.1 採用チャネル戦略

```yaml
recruiting_channels:
  primary_channels:
    direct_sourcing:
      platforms:
        - LinkedIn Recruiter
        - GitHub
        - Twitter/X
        - 技術ブログ
      target_ratio: 40%
      cost_per_hire: 50万円
      quality_score: High

    referral_program:
      incentive: 50-100万円（レベルによる）
      target_ratio: 30%
      cost_per_hire: 70万円
      quality_score: Highest

    job_boards:
      platforms:
        - Wantedly
        - Green
        - Findy
        - forkwell
        - 転職ドラフト
      target_ratio: 20%
      cost_per_hire: 80万円
      quality_score: Medium-High

  secondary_channels:
    agencies:
      usage: シニア・専門職のみ
      target_ratio: 10%
      cost_per_hire: 年収の30-35%
      quality_score: High

    tech_events:
      activities:
        - カンファレンス登壇
        - Meetup主催
        - ハッカソン開催
      purpose: Employer Branding
      cost: 年間500万円

    campus_recruiting:
      target_universities:
        - 東京大学
        - 京都大学
        - 東京工業大学
        - 早稲田大学
        - 慶應義塾大学
      internship_program: 3ヶ月有給インターン
      conversion_target: 50%
```

#### 3.2.2 Employer Branding戦略

```yaml
employer_branding:
  tech_blog:
    platform: Zenn / note / Medium
    frequency: 週1-2記事
    topics:
      - 技術的チャレンジ
      - アーキテクチャ決定
      - チーム文化
      - 開発プロセス
    kpi: 月間1万PV

  open_source:
    strategy:
      - 内部ツールのOSS化
      - 外部OSSへのコントリビューション
      - OSSスポンサーシップ
    budget: 年間300万円

  speaking:
    target_events:
      - iOSDC
      - DroidKaigi
      - Flutter Meetup
      - CloudNative Days
      - DevOpsDays
    frequency: 月1-2回

  social_media:
    platforms:
      - Twitter/X（技術発信）
      - LinkedIn（採用・企業情報）
      - YouTube（Tech Talk動画）
    content_calendar: 週3-5投稿
```

### 3.3 採用プロセスとインタビュー

#### 3.3.1 採用プロセスフロー

```
┌─────────────────────────────────────────────────────────────┐
│                    採用プロセスフロー                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 応募・ソーシング                                        │
│     │ (1-3日)                                               │
│     ▼                                                       │
│  2. 書類選考                                                │
│     │ スキルマッチ、経験確認                                │
│     │ (3-5日)                                               │
│     ▼                                                       │
│  3. カジュアル面談                                          │
│     │ 相互理解、文化フィット                                │
│     │ (30分, オンライン)                                    │
│     ▼                                                       │
│  4. 技術面接 Round 1                                        │
│     │ コーディング課題 or ライブコーディング                │
│     │ (60-90分)                                             │
│     ▼                                                       │
│  5. 技術面接 Round 2                                        │
│     │ システム設計、技術深堀り                              │
│     │ (60分)                                                │
│     ▼                                                       │
│  6. 行動面接                                                │
│     │ 行動特性、チームフィット                              │
│     │ (45分)                                                │
│     ▼                                                       │
│  7. 最終面接                                                │
│     │ CTO/VP面接、オファー準備                              │
│     │ (45分)                                                │
│     ▼                                                       │
│  8. オファー                                                │
│     │ 条件提示、交渉                                        │
│     │ (3-5日)                                               │
│     ▼                                                       │
│  9. 入社                                                    │
│                                                             │
│  目標リードタイム: 3-4週間                                  │
│  オファー承諾率目標: 80%                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 3.3.2 評価基準

```yaml
evaluation_criteria:
  technical_skills:
    weight: 40%
    assessment:
      - コーディング能力
      - システム設計能力
      - 技術的知識の深さ
      - 問題解決アプローチ
    scoring: 1-5 scale

  collaboration:
    weight: 25%
    assessment:
      - コミュニケーション
      - チームワーク
      - フィードバック受容性
      - 教え合う姿勢
    scoring: 1-5 scale

  ownership:
    weight: 20%
    assessment:
      - 自律性
      - 責任感
      - 主体性
      - 完遂力
    scoring: 1-5 scale

  growth_potential:
    weight: 15%
    assessment:
      - 学習意欲
      - 適応力
      - 好奇心
      - 自己改善
    scoring: 1-5 scale

  hiring_bar:
    minimum_score: 3.5 (各カテゴリ)
    strong_hire: 4.0+ (平均)
```

---

## 第4章 予算＆コスト計画

### 4.1 人件費モデル

#### 4.1.1 レベル別報酬体系

```yaml
compensation_bands:
  japan_market:
    ic_track:  # Individual Contributor
      l2_junior:
        base: 400-550万円
        equity: 0-50万円
        total: 400-600万円

      l3_mid:
        base: 550-750万円
        equity: 50-100万円
        total: 600-850万円

      l4_senior:
        base: 750-1000万円
        equity: 100-200万円
        total: 850-1200万円

      l5_staff:
        base: 1000-1300万円
        equity: 200-400万円
        total: 1200-1700万円

      l6_principal:
        base: 1300-1600万円
        equity: 400-700万円
        total: 1700-2300万円

    management_track:
      m1_em:
        base: 900-1200万円
        equity: 200-400万円
        total: 1100-1600万円

      m2_director:
        base: 1200-1600万円
        equity: 400-800万円
        total: 1600-2400万円

      m3_vp:
        base: 1600-2200万円
        equity: 800-1500万円
        total: 2400-3700万円

      m4_cto:
        base: 2000-2800万円
        equity: 1500-3000万円
        total: 3500-5800万円

  benefits:
    standard:
      - 健康保険・厚生年金
      - 通勤手当（上限5万円/月）
      - リモートワーク手当（3万円/月）
      - 書籍・学習補助（年間12万円）
      - カンファレンス参加補助
      - 健康診断・人間ドック

    equity:
      type: Stock Options / RSU
      vesting: 4年（1年クリフ、月次ベスティング）
```

#### 4.1.2 年間人件費予算

```yaml
headcount_budget:
  year_1:
    headcount:
      q1: 11
      q2: 17
      q3: 22
      q4: 26
      average: 19

    total_compensation:
      base_salary: 1.8億円
      equity_cost: 0.3億円
      benefits: 0.3億円
      hiring_cost: 0.3億円
      training: 0.1億円
      total: 2.8億円

  year_2:
    headcount:
      q1: 35
      q2: 45
      q3: 55
      q4: 63
      average: 50

    total_compensation:
      base_salary: 5.0億円
      equity_cost: 1.0億円
      benefits: 0.8億円
      hiring_cost: 0.7億円
      training: 0.3億円
      total: 7.8億円

  year_3:
    headcount:
      q1: 80
      q2: 100
      q3: 120
      q4: 133
      average: 108

    total_compensation:
      base_salary: 12.0億円
      equity_cost: 3.0億円
      benefits: 2.0億円
      hiring_cost: 1.5億円
      training: 0.8億円
      total: 19.3億円
```

### 4.2 インフラコスト

#### 4.2.1 クラウドインフラ予算

```yaml
infrastructure_budget:
  year_1:
    gcp:
      compute: 2,000万円
      database: 1,500万円
      networking: 500万円
      storage: 300万円
      other: 500万円
      subtotal: 4,800万円

    third_party:
      monitoring: 300万円  # Datadog
      cdn: 200万円  # CloudFlare
      security: 400万円  # Various
      subtotal: 900万円

    total: 5,700万円
    per_engineer: 22万円/月

  year_2:
    gcp:
      compute: 8,000万円
      database: 5,000万円
      networking: 2,000万円
      storage: 1,500万円
      other: 1,500万円
      subtotal: 1.8億円

    third_party:
      monitoring: 800万円
      cdn: 600万円
      security: 1,000万円
      subtotal: 2,400万円

    total: 2.04億円
    per_engineer: 34万円/月

  year_3:
    gcp:
      compute: 2.0億円
      database: 1.2億円
      networking: 0.5億円
      storage: 0.4億円
      other: 0.4億円
      subtotal: 4.5億円

    third_party:
      monitoring: 2,000万円
      cdn: 1,500万円
      security: 2,500万円
      subtotal: 6,000万円

    total: 5.1億円
    per_engineer: 40万円/月
```

### 4.3 ツール・サービスコスト

#### 4.3.1 開発ツール予算

```yaml
tooling_budget:
  development_tools:
    ide_licenses:
      jetbrains: 5万円/人/年
      other: 2万円/人/年

    source_control:
      github_enterprise: 21ドル/人/月
      annual_cost_estimate: 500万円 (year 2)

    ci_cd:
      github_actions: 従量課金
      estimated: 100万円/年

    code_quality:
      sonarqube: 30万円/年
      snyk: 50万円/年

  collaboration_tools:
    slack: 850円/人/月
    notion: 8ドル/人/月
    figma: 15ドル/人/月
    miro: 8ドル/人/月
    jira: 7.5ドル/人/月

  productivity:
    1password: 8ドル/人/月
    grammarly: 12ドル/人/月
    github_copilot: 19ドル/人/月

  annual_estimates:
    year_1: 1,500万円
    year_2: 4,000万円
    year_3: 9,000万円
```

#### 4.3.2 総予算サマリー

```
┌─────────────────────────────────────────────────────────────┐
│                    年間予算サマリー                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  カテゴリ          Year 1      Year 2      Year 3          │
│  ─────────────────────────────────────────────────         │
│  人件費            2.8億円     7.8億円     19.3億円        │
│  インフラ          0.6億円     2.0億円      5.1億円        │
│  ツール            0.2億円     0.4億円      0.9億円        │
│  その他            0.2億円     0.5億円      1.0億円        │
│  ─────────────────────────────────────────────────         │
│  合計              3.8億円    10.7億円     26.3億円        │
│                                                             │
│  一人当たり/月      167万円    178万円      203万円        │
│  一人当たり/年     2,000万円  2,140万円   2,440万円        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 第5章 チーム開発

### 5.1 オンボーディングプログラム

#### 5.1.1 オンボーディングフレームワーク

```
┌─────────────────────────────────────────────────────────────┐
│              30-60-90日オンボーディングプログラム            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Day 1-7: Setup & Orientation                               │
│  ├─ Day 1: Welcome & Admin                                 │
│  │   ・入社手続き、機器セットアップ                        │
│  │   ・Slack/Email/カレンダー設定                          │
│  │   ・セキュリティトレーニング                            │
│  │                                                         │
│  ├─ Day 2-3: Company & Product                             │
│  │   ・会社説明、ミッション・バリュー                      │
│  │   ・プロダクトデモ、ユーザージャーニー                  │
│  │   ・ビジネスモデル理解                                  │
│  │                                                         │
│  ├─ Day 4-5: Technical Setup                               │
│  │   ・開発環境構築                                        │
│  │   ・リポジトリアクセス、ブランチ戦略                    │
│  │   ・CI/CD、デプロイメントプロセス                       │
│  │                                                         │
│  └─ Day 6-7: Team Integration                              │
│      ・チーム紹介、1:1スケジュール                         │
│      ・スクワッドセレモニー参加                            │
│                                                             │
│  Week 2-4: Learning & First Contribution (Day 8-30)        │
│  ├─ コードベース理解                                       │
│  │   ・アーキテクチャウォークスルー                        │
│  │   ・主要モジュールの深掘り                              │
│  │                                                         │
│  ├─ First PR                                               │
│  │   ・Good First Issueへの取り組み                        │
│  │   ・コードレビュープロセス体験                          │
│  │                                                         │
│  └─ Buddy System                                           │
│      ・専属バディとの週次1:1                               │
│      ・質問・フィードバックの場                            │
│                                                             │
│  Day 31-60: Ownership & Independence                       │
│  ├─ 独立した機能開発                                       │
│  ├─ コードレビュー参加                                     │
│  ├─ オンコール見習い                                       │
│  └─ 30日レビュー（マネージャー）                           │
│                                                             │
│  Day 61-90: Full Productivity                              │
│  ├─ 完全な開発サイクル担当                                 │
│  ├─ チーム貢献（ドキュメント、改善提案）                   │
│  ├─ メンタリング開始（Junior向け）                         │
│  └─ 90日レビュー（目標設定）                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 5.1.2 オンボーディングチェックリスト

```yaml
onboarding_checklist:
  administrative:
    - 雇用契約書署名
    - 個人情報登録
    - 銀行口座登録
    - 社会保険手続き
    - 入館証発行

  equipment:
    - ノートPC支給（MacBook Pro）
    - モニター支給（希望者）
    - キーボード・マウス
    - ヘッドセット

  access_setup:
    - Google Workspace
    - Slack
    - GitHub
    - GCP Console
    - Jira / Linear
    - Notion
    - 1Password
    - VPN

  security:
    - セキュリティ研修完了
    - 2FA設定
    - パスワードポリシー確認
    - 機密保持誓約書

  development:
    - ローカル環境構築
    - テスト実行確認
    - PRテンプレート理解
    - デプロイメント手順確認

  cultural:
    - バリュー研修
    - 行動規範確認
    - ダイバーシティ研修
    - ハラスメント防止研修
```

### 5.2 トレーニング＆スキル開発

#### 5.2.1 学習開発プログラム

```yaml
learning_programs:
  technical_training:
    internal:
      tech_talks:
        frequency: 週1回
        duration: 30-60分
        topics: 技術深掘り、新技術紹介
        presenter: ローテーション

      study_groups:
        frequency: 隔週
        topics:
          - システム設計
          - アルゴリズム
          - 新技術探索

      code_reviews:
        purpose: 継続的学習
        guidelines: documented

    external:
      conferences:
        budget: 年間20万円/人
        target: 最低2回/年参加

      online_courses:
        platforms:
          - Udemy Business
          - Coursera
          - O'Reilly Learning
        budget: 年間12万円/人

      certifications:
        supported:
          - AWS/GCP認定
          - Kubernetes認定（CKA, CKAD）
          - セキュリティ認定
        budget: 全額会社負担

  leadership_development:
    new_manager_training:
      duration: 3ヶ月
      modules:
        - 1:1の効果的な実施
        - フィードバックの与え方
        - パフォーマンス管理
        - 採用面接スキル
        - コンフリクト解決

    senior_leadership:
      executive_coaching: VP以上
      external_programs: MBA短期プログラム等
```

#### 5.2.2 スキルマトリクス

```yaml
skill_matrix:
  technical_skills:
    frontend:
      flutter: [Basic, Intermediate, Advanced, Expert]
      react: [Basic, Intermediate, Advanced, Expert]
      typescript: [Basic, Intermediate, Advanced, Expert]

    backend:
      nodejs: [Basic, Intermediate, Advanced, Expert]
      go: [Basic, Intermediate, Advanced, Expert]
      python: [Basic, Intermediate, Advanced, Expert]

    infrastructure:
      kubernetes: [Basic, Intermediate, Advanced, Expert]
      terraform: [Basic, Intermediate, Advanced, Expert]
      gcp: [Basic, Intermediate, Advanced, Expert]

    data:
      sql: [Basic, Intermediate, Advanced, Expert]
      spark: [Basic, Intermediate, Advanced, Expert]
      ml: [Basic, Intermediate, Advanced, Expert]

  soft_skills:
    communication: [Developing, Proficient, Strong, Exemplary]
    leadership: [Developing, Proficient, Strong, Exemplary]
    mentoring: [Developing, Proficient, Strong, Exemplary]
    problem_solving: [Developing, Proficient, Strong, Exemplary]

  assessment_frequency: 半期
  development_plan: 個人別作成
```

### 5.3 キャリアパス設計

#### 5.3.1 デュアルトラックキャリア

```
┌─────────────────────────────────────────────────────────────┐
│              デュアルトラックキャリアパス                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│        Individual Contributor        Management             │
│              Track                     Track                │
│                                                             │
│              ┌───────┐                                      │
│              │  L6   │ Principal Engineer                   │
│              │       │                                      │
│              └───┬───┘                                      │
│                  │                                          │
│              ┌───┴───┐         ┌───────┐                   │
│              │  L5   │ Staff   │  M3   │ VP of Engineering │
│              │       │ Eng.    │       │                   │
│              └───┬───┘         └───┬───┘                   │
│                  │                 │                        │
│              ┌───┴───┐         ┌───┴───┐                   │
│              │  L4   │ Senior  │  M2   │ Director          │
│              │       │ Eng.    │       │                   │
│              └───┬───┘         └───┬───┘                   │
│                  │                 │                        │
│              ┌───┴───┐         ┌───┴───┐                   │
│              │  L3   │ Mid     │  M1   │ Eng. Manager      │
│              │       │ Eng.    │       │                   │
│              └───┬───┘         └───────┘                   │
│                  │                                          │
│              ┌───┴───┐                                      │
│              │  L2   │ Junior Engineer                     │
│              │       │                                      │
│              └───────┘                                      │
│                                                             │
│  移行可能:                                                  │
│  ・L4 Senior → M1 EM（マネジメント志向）                   │
│  ・M1 EM → L4/L5 Staff（IC志向に戻る）                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 5.3.2 レベル別期待値

```yaml
level_expectations:
  l2_junior:
    experience: 0-2年
    scope: タスク
    autonomy: 指示のもと作業
    impact: 個人
    skills:
      - 基本的なコーディング
      - コードレビュー受け入れ
      - チーム規範の習得

  l3_mid:
    experience: 2-4年
    scope: 機能
    autonomy: 方向性のもと自律
    impact: チーム
    skills:
      - 独立した機能開発
      - 設計参加
      - ジュニアサポート

  l4_senior:
    experience: 4-7年
    scope: プロジェクト
    autonomy: 目標設定後は自律
    impact: 複数チーム
    skills:
      - 技術的意思決定
      - 設計リード
      - メンタリング
      - プロセス改善

  l5_staff:
    experience: 7-10年
    scope: 組織
    autonomy: 自ら目標設定
    impact: 組織全体
    skills:
      - アーキテクチャ決定
      - 技術戦略
      - 組織横断リード
      - 外部発信

  l6_principal:
    experience: 10年以上
    scope: 会社/業界
    autonomy: 戦略立案
    impact: 会社・業界
    skills:
      - 技術ビジョン策定
      - 業界影響力
      - 組織設計
      - エグゼクティブ連携
```

---

## 第6章 実装ロードマップ & 文書間参照

### 6.1 リソース成長タイムライン

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    リソース成長タイムライン                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Headcount                                                              │
│  150 │                                            ████████████         │
│      │                                       █████                      │
│  100 │                                  █████                           │
│      │                             █████                                │
│   63 │                        █████                                     │
│      │                   █████                                          │
│   26 │              █████                                               │
│      │         █████                                                    │
│    5 │    █████                                                         │
│      └──────────────────────────────────────────────────────────────    │
│         Q1   Q2   Q3   Q4   Q1   Q2   Q3   Q4   Q1   Q2   Q3   Q4      │
│         ├──────Year 1──────┤├──────Year 2──────┤├──────Year 3──────┤    │
│                                                                         │
│  Organization Structure:                                                │
│  Year 1: Function-based Teams → Squads                                 │
│  Year 2: Squads → Multiple Squads with Chapters                        │
│  Year 3: Tribes with Squads, Chapters, Guilds                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 チーム構成マイルストーン

```yaml
team_milestones:
  m1_core_team:
    date: Year 1 Q2
    headcount: 15-20
    structure: Functional Teams
    leadership:
      - CTO
      - Tech Leads (2)
    capabilities:
      - Full stack development
      - Basic DevOps
      - QA foundation

  m2_squad_structure:
    date: Year 1 Q4
    headcount: 25-30
    structure: 3 Squads
    leadership:
      - CTO
      - Engineering Manager (1)
      - Tech Leads (3)
    capabilities:
      - Autonomous squads
      - 24/5 support coverage
      - Dedicated platform team

  m3_scaled_squads:
    date: Year 2 Q2
    headcount: 45-55
    structure: 5-6 Squads
    leadership:
      - CTO
      - Directors (2)
      - Engineering Managers (4)
    capabilities:
      - Multiple product streams
      - 24/7 support coverage
      - Dedicated security team

  m4_tribe_organization:
    date: Year 2 Q4
    headcount: 60-70
    structure: 2 Tribes
    leadership:
      - CTO
      - VPs (2)
      - Directors (3)
      - Engineering Managers (6)
    capabilities:
      - Multi-region operations
      - ML/AI capabilities
      - Enterprise support

  m5_global_organization:
    date: Year 3 Q4
    headcount: 130-150
    structure: 3 Tribes
    leadership:
      - CTO
      - VPs (3)
      - Directors (5)
      - Engineering Managers (12)
    capabilities:
      - Global 24/7 operations
      - Full ML platform
      - Multi-cloud
```

### 6.3 関連文書参照

```yaml
document_references:
  implementation_documents:
    - Doc-IR-001: 開発ロードマップ Year 1-3
    - Doc-DM-001: アジャイルデリバリーフレームワーク
    - Doc-QA-001: 品質保証戦略
    - Doc-DO-001: DevOps戦略

  technical_documents:
    - Doc-TV-001: 技術ビジョン
    - Doc-SA-001: システムアーキテクチャ
    - Doc-IA-001: インフラアーキテクチャ

  operational_documents:
    - Doc-OS-001: オペレーションモデル
    - Doc-OS-002: サプライチェーン管理
    - Doc-OS-003: カスタマーサービス戦略

  financial_documents:
    - Doc-FP-001: 財務計画
    - Doc-FP-002: 投資計画
    - Doc-FP-006: 人件費計画
    - Doc-FP-007: 設備投資計画
```

### 6.4 成功基準サマリー

```yaml
success_criteria:
  hiring:
    year_1:
      - 目標採用数達成: 21名
      - オファー承諾率: 75%以上
      - Time to Hire: 4週間以内
      - 採用品質スコア: 4.0以上

    year_2:
      - 目標採用数達成: 37名
      - オファー承諾率: 80%以上
      - リファラル率: 35%以上
      - 多様性目標達成

    year_3:
      - 目標採用数達成: 70名
      - グローバル採用確立
      - Employer Brandスコア向上

  retention:
    voluntary_turnover: 10%以下
    regretted_turnover: 5%以下
    engagement_score: 80以上
    enps: 40以上

  productivity:
    year_1:
      - 一人当たりアウトプット確立
      - オンボーディング完了率: 95%

    year_2:
      - 生産性維持（成長しながら）
      - チーム満足度: 80%以上

    year_3:
      - 業界トップレベルの生産性
      - イノベーション文化確立

  budget:
    actual_vs_plan: ±10%以内
    cost_per_hire: 業界平均以下
    roi_on_training: 測定・改善
```

---

## 付録

### 付録A: 役職別JD（Job Description）テンプレート

```yaml
jd_template:
  title: [役職名]
  level: [L2-L6 / M1-M4]
  location: Tokyo / Remote
  employment_type: Full-time

  about_company: |
    TripTripは、革新的な旅行体験を提供する
    モバイルアプリケーションサービスです...

  about_role: |
    [役割の概要と期待される貢献]

  responsibilities:
    - [責任1]
    - [責任2]
    - [責任3]

  requirements:
    must_have:
      - [必須要件1]
      - [必須要件2]
    nice_to_have:
      - [歓迎要件1]
      - [歓迎要件2]

  benefits:
    - 競争力のある報酬パッケージ
    - ストックオプション
    - フレキシブルワーク
    - 学習開発支援
    - 健康保険・福利厚生
```

### 付録B: 面接質問バンク

```yaml
interview_questions:
  technical:
    system_design:
      - "100万ユーザーをサポートする予約システムを設計してください"
      - "分散キャッシュシステムの設計と課題を説明してください"

    coding:
      - アルゴリズム問題（難易度：Medium-Hard）
      - システム実装問題（API設計、データモデリング）

  behavioral:
    ownership:
      - "困難な技術的決定を下した経験を教えてください"
      - "失敗から学んだ経験を教えてください"

    collaboration:
      - "チーム内の意見の相違をどう解決しましたか"
      - "他のチームとの協業で工夫した点は"

    growth:
      - "最近学んだ新しい技術について教えてください"
      - "キャリアの目標と、そこに向けた取り組みは"
```

### 付録C: リモートワークポリシー

```yaml
remote_work_policy:
  eligibility:
    - 全ポジション対象
    - 試用期間中は週2日出社推奨

  expectations:
    - コアタイム: 11:00-16:00 JST
    - カメラオン推奨（ミーティング時）
    - Slack応答: 業務時間内1時間以内

  equipment:
    provided:
      - ノートPC
      - モニター（申請制）
      - キーボード・マウス

    allowance:
      - リモートワーク手当: 3万円/月
      - インターネット補助: 含む

  communication:
    synchronous:
      - デイリースタンドアップ
      - 週次1:1
      - スプリントセレモニー

    asynchronous:
      - Slack/Notion優先
      - ドキュメント文化
```

---

**文書情報**

| 項目 | 内容 |
|------|------|
| Document ID | Doc-IR-002 |
| Version | 1.0.0 |
| Created | 2026-01-20 |
| Last Updated | 2026-01-20 |
| Status | Draft |
| Owner | Implementation Agent |
| Reviewers | CTO, VP Engineering, HR |
| Total Lines | 1,500+ |

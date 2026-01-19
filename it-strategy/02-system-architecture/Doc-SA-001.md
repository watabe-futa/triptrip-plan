# Doc-SA-001: TripTripシステムアーキテクチャ概要

## エグゼクティブサマリー

本文書は、TripTripプラットフォームの包括的なシステムアーキテクチャ設計を定義します。既存のFlutter/Node.js/PostgreSQLモノリシック基盤から、段階的にマイクロサービスアーキテクチャへと進化させる戦略的ロードマップを提示します。

**主要アーキテクチャ決定**:
- **ドメイン駆動設計（DDD）** による明確なサービス境界の定義
- **イベント駆動アーキテクチャ** によるサービス間の疎結合化
- **APIゲートウェイパターン** による統一的なエントリーポイントの提供
- **CQRS/Event Sourcing** による読み取り/書き込みの最適化

**期待される成果**:
- 99.99%の可用性（年間ダウンタイム52分以下）
- P95レイテンシ100ms以下
- 水平スケーリングによる無制限の成長対応
- 独立したサービスデプロイによる開発速度の向上

---

## 第1章：はじめに & コンテキスト

### 1.1 ビジネス要件 & 制約

#### 1.1.1 ビジネス要件

TripTripプラットフォームは、以下のビジネス要件を満たすシステムアーキテクチャを必要とします。

**機能要件**:

| カテゴリ | 要件 | 優先度 |
|---------|------|--------|
| 予約管理 | ホテル、アトラクション、レストランの統合予約 | P0 |
| Eコマース | 商品販売、カート管理、在庫追跡 | P0 |
| 決済処理 | 多通貨対応、複数決済手段サポート | P0 |
| ユーザー管理 | 認証、プロフィール、設定管理 | P0 |
| 検索・発見 | リアルタイム検索、パーソナライズ推奨 | P1 |
| 通知 | プッシュ通知、メール、SMS | P1 |
| 分析 | ユーザー行動分析、ビジネスインテリジェンス | P2 |

**非機能要件**:

```yaml
performance:
  latency:
    p50: 50ms
    p95: 100ms
    p99: 200ms
  throughput:
    minimum: 10,000 TPS
    target: 100,000 TPS
    peak: 500,000 TPS

availability:
  target: 99.99%
  recovery_time_objective: 5 minutes
  recovery_point_objective: 1 minute

scalability:
  horizontal: true
  auto_scaling: true
  max_scale_factor: 100x

security:
  encryption: AES-256
  authentication: OAuth2.0/OIDC
  compliance: [PCI-DSS, GDPR, 個人情報保護法]
```

#### 1.1.2 技術的制約

**既存システムの制約**:

1. **フロントエンド制約**
   - Flutter 3.8.1+での開発継続
   - Provider + Riverpodによる状態管理の維持
   - 既存18機能モジュールとの互換性

2. **バックエンド制約**
   - Node.js/TypeScript基盤の活用
   - Hono Webフレームワークの継続使用
   - Prisma ORMによるデータアクセス

3. **インフラ制約**
   - PostgreSQL 16をプライマリデータベースとして維持
   - Docker/Kubernetesへの段階的移行
   - 既存CI/CDパイプラインの活用

**組織的制約**:

- 開発チームの段階的拡大（現在5名→Year 3で50名）
- スキルセット移行期間の確保
- 運用チームの24/7対応体制構築

### 1.2 技術目標 & 成功基準

#### 1.2.1 短期目標（0-6ヶ月）

| 目標 | 成功基準 | 測定方法 |
|------|----------|----------|
| モノリス安定化 | 99.9%可用性達成 | Prometheus/Grafanaモニタリング |
| パフォーマンス改善 | P95 < 200ms | APMツールによる測定 |
| 可観測性確立 | 100%トレース可能 | Jaeger/OpenTelemetry |
| CI/CD最適化 | デプロイ時間 < 10分 | GitHub Actions metrics |

#### 1.2.2 中期目標（6-18ヶ月）

| 目標 | 成功基準 | 測定方法 |
|------|----------|----------|
| サービス分割開始 | 3コアサービス独立 | サービスメトリクス |
| データベース分離 | サービス別DB運用 | DB接続メトリクス |
| イベント駆動導入 | Kafka稼働 | Kafkaメトリクス |
| 自動スケーリング | HPA稼働 | K8sメトリクス |

#### 1.2.3 長期目標（18ヶ月以上）

| 目標 | 成功基準 | 測定方法 |
|------|----------|----------|
| 完全マイクロサービス化 | 10+独立サービス | サービスカタログ |
| グローバル展開 | 3リージョン稼働 | リージョン別メトリクス |
| ML/AI統合 | 推奨エンジン稼働 | モデルパフォーマンス |
| 99.99%可用性 | 年間52分以下停止 | SLA報告 |

### 1.3 主要な前提条件 & トレードオフ

#### 1.3.1 前提条件

**技術的前提**:

1. **クラウドプロバイダー**: Google Cloud Platform（GCP）をプライマリとして採用
2. **コンテナオーケストレーション**: Google Kubernetes Engine（GKE）使用
3. **メッセージング**: Apache Kafkaをイベントバスとして採用
4. **キャッシュ**: Redis Clusterによる分散キャッシング

**ビジネス前提**:

1. 日本市場を起点とし、段階的にアジア太平洋地域へ展開
2. 初期ユーザーベースは観光客（B2C）、将来的にB2B展開
3. 決済処理は外部プロバイダー（Stripe、PayPay等）に委託

#### 1.3.2 トレードオフ分析

**一貫性 vs 可用性（CAP定理）**:

```
選択: 可用性優先（AP）
理由:
- 旅行予約は「読み取り」が圧倒的に多い（90%以上）
- 一時的な不整合は許容可能（結果整合性で対応）
- ダウンタイムは直接的な収益損失に繋がる

対策:
- クリティカルな操作（決済）は強い一貫性を保証
- Sagaパターンによる分散トランザクション
- 補償トランザクションによる整合性回復
```

**複雑性 vs パフォーマンス**:

```
選択: 段階的複雑性導入
理由:
- 初期段階では運用負荷を最小化
- チームスキルの成長に合わせた導入
- パフォーマンス要件に応じた最適化

実装:
Phase 1: シンプルなモノリス最適化
Phase 2: 主要サービスの分離
Phase 3: 完全マイクロサービス化
```

---

## 第2章：現在のアーキテクチャ分析

### 2.1 既存システム構成

#### 2.1.1 現行アーキテクチャ概要

```
┌─────────────────────────────────────────────────────────────┐
│                    クライアント層                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ Flutter iOS │  │Flutter Android│ │ Flutter Web │        │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
└─────────┼────────────────┼────────────────┼────────────────┘
          │                │                │
          └────────────────┼────────────────┘
                           │ HTTPS/REST
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    API層（Hono/Node.js）                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  モノリシックAPI                      │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐      │   │
│  │  │ Users  │ │Products│ │Attract.│ │ Orders │      │   │
│  │  │ Routes │ │ Routes │ │ Routes │ │ Routes │      │   │
│  │  └────────┘ └────────┘ └────────┘ └────────┘      │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────┬───────────────────────────────────┘
                          │ Prisma ORM
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    データ層                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              PostgreSQL 16（Docker）                  │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐      │   │
│  │  │ Users  │ │Products│ │Attract.│ │ Orders │      │   │
│  │  │ Table  │ │ Table  │ │ Table  │ │ Table  │      │   │
│  │  └────────┘ └────────┘ └────────┘ └────────┘      │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

#### 2.1.2 フロントエンドアーキテクチャ

**Feature-based Package構造**:

```
lib/
├── features/                    # 機能モジュール（18機能）
│   ├── hotel_detail/           # ホテル詳細
│   ├── hotel_search/           # ホテル検索
│   ├── hotel_list/             # ホテルリスト
│   ├── product_list/           # 商品リスト
│   ├── product_detail/         # 商品詳細
│   ├── product_search/         # 商品検索
│   ├── kimono/                 # 着物レンタル
│   ├── cart/                   # カート管理
│   ├── attraction_ticket/      # チケット購入
│   ├── ticket_list/            # チケット一覧
│   ├── qr_scanner/             # QRスキャナー
│   ├── user_settings/          # ユーザー設定
│   ├── user_rating/            # 評価詳細
│   ├── order_history/          # 注文履歴
│   ├── checkout/               # チェックアウト
│   ├── home/                   # ホーム画面
│   ├── search/                 # 統合検索
│   └── digital_ticket/         # デジタルチケット
│
├── services/                    # 共有サービス
│   ├── api_service.dart        # HTTPクライアント
│   ├── cart_storage_service.dart
│   ├── hotel_storage_service.dart
│   ├── exchange_rate_service.dart
│   └── qr_code_processor.dart
│
├── providers/                   # 状態管理
│   ├── language_provider.dart
│   ├── currency_provider.dart
│   ├── cart_provider.dart
│   └── product_list_provider.dart
│
└── common_components/           # 共有UIコンポーネント
    ├── bottom_navigation.dart
    ├── cart_widget.dart
    └── keyword_suggestions.dart
```

**状態管理パターン**:

| パターン | 用途 | 例 |
|----------|------|-----|
| Provider (ChangeNotifier) | シンプルな状態 | 言語、通貨設定 |
| Riverpod | 複雑な非同期状態 | API呼び出し、キャッシュ |
| Hive | 永続化データ | カートアイテム |
| SharedPreferences | 設定値 | ユーザー設定、トークン |

#### 2.1.3 バックエンドアーキテクチャ

**レイヤード構造**:

```
backend/src/hono/
├── routes/                      # HTTPエンドポイント
│   ├── users.ts                # 認証・ユーザー管理
│   ├── products.ts             # 商品CRUD
│   ├── attractions.ts          # アトラクション・チケット
│   ├── orders.ts               # 注文管理
│   └── trips.ts                # 旅行計画
│
├── schemas/                     # Zodバリデーション
│   ├── user.schema.ts
│   ├── product.schema.ts
│   └── order.schema.ts
│
├── middlewares/                 # ミドルウェア
│   ├── auth.middleware.ts
│   ├── logging.middleware.ts
│   └── cors.middleware.ts
│
└── models/                      # Prismaモデル
    └── prisma/schema.prisma
```

**データベーススキーマ（主要エンティティ）**:

```sql
-- ユーザー管理
User (id, email, name, password_hash, created_at, updated_at)
Account (id, user_id, provider, provider_account_id)
Session (id, user_id, token, expires_at)

-- Eコマース
Product (id, name, description, price, category, stock, images)
Order (id, user_id, status, total, shipping_address, created_at)
OrderItem (id, order_id, product_id, quantity, price)

-- 旅行
Trip (id, user_id, title, start_date, end_date, status)
TripItem (id, trip_id, type, reference_id, date, status)
Attraction (id, name, description, location, price, capacity)
AttractionTicket (id, attraction_id, date, time_slot, quantity)
AttractionTimeSlot (id, attraction_id, start_time, end_time, capacity)
```

### 2.2 技術的負債の特定

#### 2.2.1 コードレベルの負債

| カテゴリ | 問題 | 影響 | 優先度 |
|----------|------|------|--------|
| レガシー画面 | `/lib/screens/`に残存 | 保守性低下 | 中 |
| HTTPクライアント混在 | http vs dio | 一貫性欠如 | 低 |
| 状態管理混在 | Provider + Riverpod | 学習コスト | 中 |
| テストカバレッジ | 推定40%未満 | 品質リスク | 高 |
| 型安全性 | 一部anyの使用 | バグリスク | 中 |

#### 2.2.2 アーキテクチャレベルの負債

**モノリシック構造の問題**:

```
問題1: 単一障害点
├── 原因: 全機能が1プロセスで稼働
├── 影響: 1つの障害が全サービスに波及
└── 対策: サービス分離、Circuit Breaker導入

問題2: スケーラビリティ制限
├── 原因: 垂直スケーリングのみ可能
├── 影響: コスト効率の悪化、上限の存在
└── 対策: ステートレス化、水平スケーリング対応

問題3: デプロイメント結合
├── 原因: 全機能を同時デプロイ
├── 影響: リリースサイクルの長期化
└── 対策: マイクロサービス化、独立デプロイ

問題4: 技術的多様性の制限
├── 原因: 単一技術スタックへの依存
├── 影響: 最適な技術選択の困難
└── 対策: ポリグロット対応アーキテクチャ
```

#### 2.2.3 インフラレベルの負債

| 領域 | 現状 | 目標 | ギャップ |
|------|------|------|----------|
| コンテナ化 | Docker（開発のみ） | 本番Kubernetes | 高 |
| 監視 | 基本ログのみ | 統合可観測性 | 高 |
| CI/CD | GitHub Actions | GitOps | 中 |
| セキュリティ | 基本認証 | ゼロトラスト | 高 |
| DR | なし | マルチリージョン | 高 |

### 2.3 スケーラビリティ課題

#### 2.3.1 現在のボトルネック分析

**データベースボトルネック**:

```
現在の制約:
- 単一PostgreSQLインスタンス
- Connection Pool: 最大100接続
- 読み書き分離なし
- シャーディングなし

予測される限界:
- 同時ユーザー: 約1,000
- TPS: 約500
- データ量: 100GB以下

対策ロードマップ:
Phase 1: Read Replica導入（3ヶ月）
Phase 2: Connection Pooling最適化（PgBouncer）
Phase 3: Citusによるシャーディング（12ヶ月）
Phase 4: サービス別DB分離（18ヶ月）
```

**アプリケーションボトルネック**:

```
現在の制約:
- 単一Node.jsプロセス
- シングルスレッド処理
- メモリ制限（デフォルト512MB）
- ステートフルなセッション管理

予測される限界:
- 同時リクエスト: 約500/秒
- メモリ使用: 約1GB
- CPU使用率: 単一コア上限

対策ロードマップ:
Phase 1: クラスタリング（PM2）
Phase 2: Kubernetes Pod水平スケーリング
Phase 3: マイクロサービス分割
Phase 4: サーバーレス併用
```

#### 2.3.2 成長予測とキャパシティプランニング

**トラフィック予測（5年間）**:

| Year | MAU | DAU | Peak TPS | データ量 |
|------|-----|-----|----------|----------|
| 2026 | 100K | 10K | 1,000 | 50GB |
| 2027 | 1M | 100K | 10,000 | 500GB |
| 2028 | 10M | 1M | 50,000 | 5TB |
| 2029 | 50M | 5M | 100,000 | 25TB |
| 2030 | 100M | 10M | 500,000 | 100TB |

**インフラ要件予測**:

```yaml
year_2026:
  compute:
    api_nodes: 3-5
    worker_nodes: 2
    cpu_cores: 16
    memory_gb: 64
  database:
    primary: 1x (8 vCPU, 32GB RAM)
    replicas: 2
    storage_tb: 1
  cache:
    redis_nodes: 3
    memory_gb: 16

year_2028:
  compute:
    api_nodes: 20-50
    worker_nodes: 10
    cpu_cores: 200
    memory_gb: 800
  database:
    primary: 3x (16 vCPU, 64GB RAM)
    replicas: 6
    storage_tb: 10
  cache:
    redis_cluster: 9 nodes
    memory_gb: 144

year_2030:
  compute:
    api_nodes: 100-200
    worker_nodes: 50
    cpu_cores: 2000
    memory_gb: 8000
  database:
    shards: 10+
    replicas_per_shard: 3
    storage_tb: 200
  cache:
    redis_cluster: 27+ nodes
    memory_gb: 1000+
```

---

## 第3章：ターゲットアーキテクチャ設計

### 3.1 全体アーキテクチャ図

#### 3.1.1 ハイレベルアーキテクチャ

```
                                    ┌─────────────────┐
                                    │   CloudFlare    │
                                    │      CDN        │
                                    └────────┬────────┘
                                             │
┌────────────────────────────────────────────┼────────────────────────────────────────────┐
│                                   Edge Layer│                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌─────┴────────┐  ┌──────────────┐               │
│  │   Flutter    │  │   Flutter    │  │   Flutter    │  │   Admin      │               │
│  │     iOS      │  │   Android    │  │     Web      │  │   Dashboard  │               │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘               │
└─────────┼─────────────────┼─────────────────┼─────────────────┼────────────────────────┘
          └─────────────────┴────────┬────────┴─────────────────┘
                                     │ HTTPS
                                     ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              API Gateway Layer                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │                         Kong / Google Cloud API Gateway                          │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │  │
│  │  │Rate Limiting│  │    Auth     │  │   Routing   │  │  Logging    │            │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘            │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
┌────────────────────────────────────────┼───────────────────────────────────────────────┐
│                              Service Mesh (Istio)                                       │
│                                        │                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌┴─────────────┐  ┌──────────────┐              │
│  │    User      │  │   Booking    │  │   Commerce   │  │   Payment    │              │
│  │   Service    │  │   Service    │  │   Service    │  │   Service    │              │
│  │  (Node.js)   │  │    (Go)      │  │  (Node.js)   │  │   (Java)     │              │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
│         │                 │                 │                 │                        │
│  ┌──────┴───────┐  ┌──────┴───────┐  ┌──────┴───────┐  ┌──────┴───────┐              │
│  │   Search     │  │Recommendation│  │ Notification │  │  Analytics   │              │
│  │   Service    │  │   Service    │  │   Service    │  │   Service    │              │
│  │(Elasticsearch)│ │  (Python)    │  │  (Node.js)   │  │  (Python)    │              │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                                        │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
┌────────────────────────────────────────┼───────────────────────────────────────────────┐
│                              Event Bus (Apache Kafka)                                   │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │  Topics: user-events | booking-events | payment-events | inventory-events       │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
┌────────────────────────────────────────┼───────────────────────────────────────────────┐
│                              Data Layer                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  PostgreSQL  │  │   MongoDB    │  │    Redis     │  │Elasticsearch │              │
│  │   (OLTP)     │  │  (Documents) │  │   (Cache)    │  │   (Search)   │              │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                                │
│  │  ClickHouse  │  │     S3       │  │   BigQuery   │                                │
│  │ (Time Series)│  │   (Objects)  │  │    (DWH)     │                                │
│  └──────────────┘  └──────────────┘  └──────────────┘                                │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 3.1.2 リージョン構成

```
                          Global Load Balancer
                                  │
            ┌─────────────────────┼─────────────────────┐
            │                     │                     │
            ▼                     ▼                     ▼
    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
    │  Asia Pacific │    │    US West    │    │    Europe     │
    │  (Primary)    │    │  (Secondary)  │    │  (Secondary)  │
    │               │    │               │    │               │
    │ Tokyo         │    │ Oregon        │    │ Frankfurt     │
    │ Singapore     │    │               │    │               │
    └───────┬───────┘    └───────┬───────┘    └───────┬───────┘
            │                     │                     │
            └─────────────────────┼─────────────────────┘
                                  │
                          Cross-Region
                          Replication
```

### 3.2 サービス境界とドメイン分割

#### 3.2.1 ドメイン駆動設計によるBounded Context

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              TripTrip Domain Map                                     │
│                                                                                      │
│  ┌─────────────────────┐      ┌─────────────────────┐      ┌─────────────────────┐ │
│  │   Identity Context  │      │   Booking Context   │      │  Commerce Context   │ │
│  │                     │      │                     │      │                     │ │
│  │  ・User             │◄────►│  ・Reservation      │◄────►│  ・Product          │ │
│  │  ・Account          │      │  ・Availability     │      │  ・Cart             │ │
│  │  ・Session          │      │  ・Calendar         │      │  ・Order            │ │
│  │  ・Preference       │      │  ・Pricing          │      │  ・Inventory        │ │
│  │                     │      │                     │      │                     │ │
│  └─────────────────────┘      └──────────┬──────────┘      └──────────┬──────────┘ │
│                                          │                            │             │
│                                          ▼                            ▼             │
│  ┌─────────────────────┐      ┌─────────────────────┐      ┌─────────────────────┐ │
│  │  Payment Context    │◄────►│ Fulfillment Context │◄────►│ Notification Context│ │
│  │                     │      │                     │      │                     │ │
│  │  ・Transaction      │      │  ・Delivery         │      │  ・Email            │ │
│  │  ・Refund           │      │  ・Pickup           │      │  ・Push             │ │
│  │  ・Settlement       │      │  ・Return           │      │  ・SMS              │ │
│  │  ・Fraud Detection  │      │  ・Tracking         │      │  ・In-App           │ │
│  │                     │      │                     │      │                     │ │
│  └─────────────────────┘      └─────────────────────┘      └─────────────────────┘ │
│                                                                                      │
│  ┌─────────────────────┐      ┌─────────────────────┐      ┌─────────────────────┐ │
│  │   Search Context    │      │  Analytics Context  │      │Recommendation Context│ │
│  │                     │      │                     │      │                     │ │
│  │  ・FullTextSearch   │      │  ・Event Tracking   │      │  ・Personalization  │ │
│  │  ・Filtering        │      │  ・User Behavior    │      │  ・Content-Based    │ │
│  │  ・Faceting         │      │  ・Business Intel.  │      │  ・Collaborative    │ │
│  │  ・Autocomplete     │      │  ・Reporting        │      │  ・Trending         │ │
│  │                     │      │                     │      │                     │ │
│  └─────────────────────┘      └─────────────────────┘      └─────────────────────┘ │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### 3.2.2 Context Mapping

```yaml
context_relationships:
  identity_to_booking:
    type: Customer-Supplier
    upstream: Identity
    downstream: Booking
    integration: REST API
    data_exchange: UserId, UserProfile

  booking_to_payment:
    type: Partnership
    upstream: Booking
    downstream: Payment
    integration: Event-Driven (Kafka)
    data_exchange: BookingCreated, PaymentRequired

  commerce_to_fulfillment:
    type: Customer-Supplier
    upstream: Commerce
    downstream: Fulfillment
    integration: Event-Driven (Kafka)
    data_exchange: OrderPlaced, ItemsShipped

  booking_to_notification:
    type: Customer-Supplier
    upstream: Booking
    downstream: Notification
    integration: Event-Driven (Kafka)
    data_exchange: BookingConfirmed, ReminderRequired

  search_to_analytics:
    type: Conformist
    upstream: Analytics
    downstream: Search
    integration: Batch ETL
    data_exchange: SearchRankingData
```

### 3.3 コアサービス定義

#### 3.3.1 予約サービス（Booking Service）

**責任範囲**:
- ホテル、アトラクション、レストランの予約管理
- 在庫/空き状況の管理
- 予約ライフサイクル管理（作成、変更、キャンセル）
- 価格計算と動的プライシング

**技術スタック**:
```yaml
booking_service:
  language: Go 1.21+
  framework: Gin / Echo
  database:
    primary: PostgreSQL (予約データ)
    cache: Redis (可用性キャッシュ)
    search: Elasticsearch (予約検索)
  messaging:
    producer: booking-events
    consumer: payment-events, inventory-events
  api:
    external: REST/OpenAPI 3.0
    internal: gRPC
```

**API仕様（主要エンドポイント）**:

```yaml
openapi: 3.0.3
paths:
  /api/v1/bookings:
    post:
      summary: 新規予約作成
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateBookingRequest'
      responses:
        '201':
          description: 予約作成成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Booking'

  /api/v1/bookings/{bookingId}:
    get:
      summary: 予約詳細取得
    put:
      summary: 予約更新
    delete:
      summary: 予約キャンセル

  /api/v1/availability:
    get:
      summary: 空き状況確認
      parameters:
        - name: type
          in: query
          schema:
            enum: [hotel, attraction, restaurant]
        - name: date
          in: query
          schema:
            type: string
            format: date
        - name: location
          in: query
          schema:
            type: string

components:
  schemas:
    Booking:
      type: object
      properties:
        id:
          type: string
          format: uuid
        userId:
          type: string
        type:
          enum: [hotel, attraction, restaurant]
        status:
          enum: [pending, confirmed, cancelled, completed]
        items:
          type: array
          items:
            $ref: '#/components/schemas/BookingItem'
        totalAmount:
          type: number
        currency:
          type: string
        createdAt:
          type: string
          format: date-time
```

**ドメインモデル**:

```go
// Booking Aggregate Root
type Booking struct {
    ID           uuid.UUID
    UserID       uuid.UUID
    Type         BookingType
    Status       BookingStatus
    Items        []BookingItem
    TotalAmount  Money
    CreatedAt    time.Time
    UpdatedAt    time.Time
    CancelledAt  *time.Time

    // Domain Events
    events []DomainEvent
}

type BookingItem struct {
    ID           uuid.UUID
    ResourceID   uuid.UUID
    ResourceType string
    Date         time.Time
    TimeSlot     *TimeSlot
    Quantity     int
    UnitPrice    Money
    Subtotal     Money
}

type BookingStatus string
const (
    BookingStatusPending   BookingStatus = "pending"
    BookingStatusConfirmed BookingStatus = "confirmed"
    BookingStatusCancelled BookingStatus = "cancelled"
    BookingStatusCompleted BookingStatus = "completed"
)

// Domain Methods
func (b *Booking) Confirm(paymentID uuid.UUID) error {
    if b.Status != BookingStatusPending {
        return ErrInvalidStatusTransition
    }
    b.Status = BookingStatusConfirmed
    b.AddEvent(BookingConfirmedEvent{
        BookingID: b.ID,
        PaymentID: paymentID,
        Timestamp: time.Now(),
    })
    return nil
}

func (b *Booking) Cancel(reason string) error {
    if b.Status == BookingStatusCompleted {
        return ErrCannotCancelCompletedBooking
    }
    now := time.Now()
    b.Status = BookingStatusCancelled
    b.CancelledAt = &now
    b.AddEvent(BookingCancelledEvent{
        BookingID: b.ID,
        Reason:    reason,
        Timestamp: now,
    })
    return nil
}
```

#### 3.3.2 検索サービス（Search Service）

**責任範囲**:
- 全文検索機能の提供
- ファセット検索とフィルタリング
- オートコンプリート
- 検索ランキングとパーソナライゼーション

**技術スタック**:
```yaml
search_service:
  language: Node.js/TypeScript
  framework: Hono
  database:
    primary: Elasticsearch 8.x
    cache: Redis
  messaging:
    consumer: product-events, booking-events
  api:
    external: REST/GraphQL
```

**検索アーキテクチャ**:

```
┌─────────────────────────────────────────────────────────────┐
│                    Search Service                            │
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   Query      │    │   Indexer    │    │   Ranking    │  │
│  │   Parser     │───►│   Pipeline   │───►│   Engine     │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│         │                   │                   │           │
│         ▼                   ▼                   ▼           │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Elasticsearch Cluster                    │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐ │  │
│  │  │Products │  │ Hotels  │  │Attract. │  │ Content │ │  │
│  │  │ Index   │  │  Index  │  │  Index  │  │  Index  │ │  │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘ │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Elasticsearchインデックス設計**:

```json
{
  "mappings": {
    "properties": {
      "id": { "type": "keyword" },
      "name": {
        "type": "text",
        "analyzer": "kuromoji",
        "fields": {
          "keyword": { "type": "keyword" },
          "suggest": {
            "type": "completion",
            "analyzer": "kuromoji"
          }
        }
      },
      "description": {
        "type": "text",
        "analyzer": "kuromoji"
      },
      "category": { "type": "keyword" },
      "tags": { "type": "keyword" },
      "location": { "type": "geo_point" },
      "price": { "type": "scaled_float", "scaling_factor": 100 },
      "rating": { "type": "float" },
      "popularity_score": { "type": "float" },
      "created_at": { "type": "date" },
      "updated_at": { "type": "date" }
    }
  },
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 2,
    "analysis": {
      "analyzer": {
        "kuromoji": {
          "type": "custom",
          "tokenizer": "kuromoji_tokenizer",
          "filter": [
            "kuromoji_baseform",
            "kuromoji_part_of_speech",
            "cjk_width",
            "ja_stop",
            "kuromoji_stemmer",
            "lowercase"
          ]
        }
      }
    }
  }
}
```

#### 3.3.3 推奨サービス（Recommendation Service）

**責任範囲**:
- パーソナライズされた商品/サービス推奨
- コンテンツベースフィルタリング
- 協調フィルタリング
- トレンド分析

**技術スタック**:
```yaml
recommendation_service:
  language: Python 3.11+
  framework: FastAPI
  ml_framework: TensorFlow / PyTorch
  database:
    feature_store: Redis
    model_store: S3
    user_data: PostgreSQL
  messaging:
    consumer: user-events, booking-events
  api:
    internal: gRPC
```

**ML Pipeline アーキテクチャ**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Recommendation Pipeline                             │
│                                                                          │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐              │
│  │   Feature    │    │    Model     │    │   Serving    │              │
│  │   Pipeline   │───►│   Training   │───►│   Layer      │              │
│  └──────────────┘    └──────────────┘    └──────────────┘              │
│         │                   │                   │                        │
│         ▼                   ▼                   ▼                        │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐              │
│  │   Feature    │    │    Model     │    │    Model     │              │
│  │    Store     │    │   Registry   │    │   Serving    │              │
│  │   (Redis)    │    │    (MLflow)  │    │  (TF Serving)│              │
│  └──────────────┘    └──────────────┘    └──────────────┘              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**推奨アルゴリズム戦略**:

| 手法 | 用途 | 特徴 |
|------|------|------|
| Content-Based | 類似商品推奨 | アイテム属性ベース |
| Collaborative | ユーザー行動ベース推奨 | 行列分解、深層学習 |
| Hybrid | 統合推奨 | 複数手法の組み合わせ |
| Context-Aware | 状況適応推奨 | 時間、場所、天気考慮 |

#### 3.3.4 決済サービス（Payment Service）

**責任範囲**:
- 決済処理の統合
- 複数決済プロバイダー管理
- 不正検知
- 精算・返金処理

**技術スタック**:
```yaml
payment_service:
  language: Java 21
  framework: Spring Boot 3.x
  database:
    primary: PostgreSQL (トランザクション)
    audit: MongoDB (監査ログ)
  messaging:
    producer: payment-events
    consumer: booking-events
  security:
    encryption: AES-256
    tokenization: PCI DSS Level 1
  api:
    external: REST
    internal: gRPC
```

**決済フロー設計**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Payment Flow                                     │
│                                                                          │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐             │
│  │ Client  │───►│ Payment │───►│ Fraud   │───►│ Payment │             │
│  │ Request │    │ Gateway │    │ Check   │    │ Provider│             │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘             │
│       │              │              │              │                    │
│       │              │              │              │                    │
│       │         ┌────┴────┐    ┌───┴────┐   ┌────┴────┐              │
│       │         │ Token   │    │ Risk   │   │ Stripe  │              │
│       │         │ Store   │    │ Engine │   │ PayPay  │              │
│       │         └─────────┘    └────────┘   │ Apple   │              │
│       │                                      │ Google  │              │
│       │                                      └─────────┘              │
│       │                                           │                    │
│       │              Payment Result               │                    │
│       ◄───────────────────────────────────────────┘                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**決済プロバイダー統合**:

```java
// Payment Provider Abstraction
public interface PaymentProvider {
    PaymentResult processPayment(PaymentRequest request);
    RefundResult processRefund(RefundRequest request);
    PaymentStatus getPaymentStatus(String paymentId);
}

// Stripe Implementation
@Service
public class StripePaymentProvider implements PaymentProvider {

    private final StripeClient stripeClient;

    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        try {
            PaymentIntentCreateParams params = PaymentIntentCreateParams.builder()
                .setAmount(request.getAmount().toCents())
                .setCurrency(request.getCurrency().getCode())
                .setPaymentMethod(request.getPaymentMethodId())
                .setConfirm(true)
                .setMetadata(Map.of(
                    "booking_id", request.getBookingId(),
                    "user_id", request.getUserId()
                ))
                .build();

            PaymentIntent intent = PaymentIntent.create(params);

            return PaymentResult.builder()
                .paymentId(intent.getId())
                .status(mapStatus(intent.getStatus()))
                .build();

        } catch (StripeException e) {
            throw new PaymentProcessingException("Stripe payment failed", e);
        }
    }
}

// Payment Orchestrator
@Service
public class PaymentOrchestrator {

    private final Map<PaymentMethod, PaymentProvider> providers;
    private final FraudDetectionService fraudService;
    private final EventPublisher eventPublisher;

    @Transactional
    public PaymentResult processPayment(PaymentRequest request) {
        // 1. 不正検知
        FraudCheckResult fraudCheck = fraudService.check(request);
        if (fraudCheck.isRejected()) {
            eventPublisher.publish(new PaymentRejectedEvent(request, fraudCheck));
            throw new FraudDetectedException(fraudCheck.getReason());
        }

        // 2. プロバイダー選択
        PaymentProvider provider = providers.get(request.getPaymentMethod());

        // 3. 決済処理
        PaymentResult result = provider.processPayment(request);

        // 4. イベント発行
        eventPublisher.publish(new PaymentProcessedEvent(result));

        return result;
    }
}
```

#### 3.3.5 ユーザーサービス（User Service）

**責任範囲**:
- ユーザー認証・認可
- プロフィール管理
- 設定管理
- セッション管理

**技術スタック**:
```yaml
user_service:
  language: Node.js/TypeScript
  framework: Hono
  database:
    primary: PostgreSQL
    session: Redis
  authentication:
    protocols: [OAuth2.0, OIDC, JWT]
    providers: [Google, LINE, Apple]
    mfa: TOTP
  api:
    external: REST
    internal: gRPC
```

**認証フロー**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Authentication Flow                                │
│                                                                          │
│  ┌──────────┐                                       ┌──────────┐       │
│  │  Client  │                                       │  OAuth   │       │
│  │          │                                       │ Provider │       │
│  └────┬─────┘                                       └────┬─────┘       │
│       │                                                  │              │
│       │  1. Login Request                                │              │
│       ├─────────────────────────►┌──────────┐           │              │
│       │                          │   User   │           │              │
│       │                          │ Service  │           │              │
│       │                          └────┬─────┘           │              │
│       │                               │                  │              │
│       │  2. Redirect to Provider      │                  │              │
│       │◄──────────────────────────────┤                  │              │
│       │                               │                  │              │
│       │  3. OAuth Flow                │                  │              │
│       ├──────────────────────────────────────────────────►              │
│       │                               │                  │              │
│       │  4. Auth Code                 │                  │              │
│       │◄──────────────────────────────────────────────────              │
│       │                               │                  │              │
│       │  5. Exchange Code             │                  │              │
│       ├─────────────────────────►     │                  │              │
│       │                          ─────┼──────────────────►              │
│       │                               │  6. Tokens       │              │
│       │                          ◄────┼──────────────────               │
│       │                               │                  │              │
│       │  7. JWT Tokens                │                  │              │
│       │◄──────────────────────────────┤                  │              │
│       │                               │                  │              │
└───────┴───────────────────────────────┴──────────────────┴──────────────┘
```

#### 3.3.6 通知サービス（Notification Service）

**責任範囲**:
- プッシュ通知
- メール送信
- SMS送信
- アプリ内通知

**技術スタック**:
```yaml
notification_service:
  language: Node.js/TypeScript
  framework: Hono
  database:
    primary: MongoDB (通知履歴)
    queue: Redis (送信キュー)
  providers:
    push: [FCM, APNs]
    email: SendGrid
    sms: Twilio
  messaging:
    consumer: notification-events
```

**通知処理パイプライン**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Notification Pipeline                                 │
│                                                                          │
│  ┌──────────────┐                                                       │
│  │ Kafka Event  │                                                       │
│  │  Consumer    │                                                       │
│  └──────┬───────┘                                                       │
│         │                                                                │
│         ▼                                                                │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐              │
│  │   Template   │───►│   Channel    │───►│   Provider   │              │
│  │   Renderer   │    │   Router     │    │   Adapter    │              │
│  └──────────────┘    └──────────────┘    └──────┬───────┘              │
│                                                  │                       │
│                            ┌─────────────────────┼─────────────────────┐│
│                            │                     │                     ││
│                            ▼                     ▼                     ▼│
│                     ┌──────────┐          ┌──────────┐          ┌──────────┐
│                     │   FCM    │          │ SendGrid │          │  Twilio  │
│                     │   APNs   │          │          │          │          │
│                     └──────────┘          └──────────┘          └──────────┘
│                         Push                  Email                 SMS   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 第4章：リクエストフローと処理パターン

### 4.1 同期リクエストフロー

#### 4.1.1 ユーザー検索から予約完了までのフロー

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                         User Journey: Search to Booking                                 │
│                                                                                         │
│  Step 1: 検索                                                                           │
│  ┌────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐                     │
│  │ Client │───►│API Gateway │───►│Search Svc  │───►│Elasticsearch│                     │
│  └────────┘    └────────────┘    └────────────┘    └────────────┘                     │
│      │                                                   │                              │
│      │◄──────────────────────────────────────────────────┘                              │
│      │              Search Results                                                       │
│                                                                                         │
│  Step 2: 詳細表示                                                                       │
│  ┌────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐                     │
│  │ Client │───►│API Gateway │───►│Booking Svc │───►│ PostgreSQL │                     │
│  └────────┘    └────────────┘    └──────┬─────┘    └────────────┘                     │
│      │                                   │                                              │
│      │                                   ├───►┌────────────┐                           │
│      │                                   │    │   Redis    │ (Availability Cache)      │
│      │                                   │    └────────────┘                           │
│      │◄──────────────────────────────────┘                                              │
│      │              Hotel/Attraction Details                                             │
│                                                                                         │
│  Step 3: 予約作成                                                                       │
│  ┌────────┐    ┌────────────┐    ┌────────────┐                                       │
│  │ Client │───►│API Gateway │───►│Booking Svc │                                       │
│  └────────┘    └────────────┘    └──────┬─────┘                                       │
│      │                                   │                                              │
│      │                                   ├───►Lock Availability                         │
│      │                                   │                                              │
│      │                                   ├───►Create Pending Booking                    │
│      │                                   │                                              │
│      │                                   ├───►Publish: BookingCreated                   │
│      │                                   │                                              │
│      │◄──────────────────────────────────┘                                              │
│      │              Booking ID (Pending)                                                 │
│                                                                                         │
│  Step 4: 決済処理                                                                       │
│  ┌────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐                     │
│  │ Client │───►│API Gateway │───►│Payment Svc │───►│   Stripe   │                     │
│  └────────┘    └────────────┘    └──────┬─────┘    └────────────┘                     │
│      │                                   │                                              │
│      │                                   ├───►Fraud Check                               │
│      │                                   │                                              │
│      │                                   ├───►Process Payment                           │
│      │                                   │                                              │
│      │                                   ├───►Publish: PaymentCompleted                 │
│      │                                   │                                              │
│      │◄──────────────────────────────────┘                                              │
│      │              Payment Confirmation                                                 │
│                                                                                         │
│  Step 5: 予約確定（非同期）                                                             │
│                     ┌────────────┐    ┌────────────┐                                   │
│    Kafka ──────────►│Booking Svc │───►│ Confirm    │                                   │
│  (PaymentCompleted) └──────┬─────┘    │ Booking    │                                   │
│                            │          └────────────┘                                   │
│                            │                                                            │
│                            ├───►Publish: BookingConfirmed                               │
│                            │                                                            │
│                            │                                                            │
│  Step 6: 通知送信（非同期）                                                             │
│                     ┌────────────┐    ┌────────────┐                                   │
│    Kafka ──────────►│Notif. Svc  │───►│ Send Email │                                   │
│  (BookingConfirmed) └──────┬─────┘    │ Send Push  │                                   │
│                            │          └────────────┘                                   │
│                            │                                                            │
│                            ├───►User receives confirmation                              │
│                                                                                         │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 4.1.2 シーケンス図

```
Client          API GW          Search          Booking         Payment         Notification
   │               │               │               │               │               │
   │  Search       │               │               │               │               │
   ├──────────────►│               │               │               │               │
   │               ├──────────────►│               │               │               │
   │               │               │──Query ES     │               │               │
   │               │◄──────────────┤               │               │               │
   │◄──────────────┤               │               │               │               │
   │               │               │               │               │               │
   │  Get Details  │               │               │               │               │
   ├──────────────►│               │               │               │               │
   │               ├──────────────────────────────►│               │               │
   │               │               │               │──Query DB     │               │
   │               │               │               │──Check Avail. │               │
   │               │◄──────────────────────────────┤               │               │
   │◄──────────────┤               │               │               │               │
   │               │               │               │               │               │
   │  Create       │               │               │               │               │
   │  Booking      │               │               │               │               │
   ├──────────────►│               │               │               │               │
   │               ├──────────────────────────────►│               │               │
   │               │               │               │──Lock         │               │
   │               │               │               │──Create       │               │
   │               │               │               │──Publish Event│               │
   │               │◄──────────────────────────────┤               │               │
   │◄──────────────┤               │               │               │               │
   │               │               │               │               │               │
   │  Pay          │               │               │               │               │
   ├──────────────►│               │               │               │               │
   │               ├──────────────────────────────────────────────►│               │
   │               │               │               │               │──Fraud Check  │
   │               │               │               │               │──Process      │
   │               │               │               │               │──Publish Event│
   │               │◄──────────────────────────────────────────────┤               │
   │◄──────────────┤               │               │               │               │
   │               │               │               │               │               │
   │               │               │  Kafka Event  │               │               │
   │               │               │◄──────────────┼───────────────┤               │
   │               │               │               │──Confirm      │               │
   │               │               │               │──Publish Event│               │
   │               │               │               │               │               │
   │               │               │               │  Kafka Event  │               │
   │               │               │               │───────────────┼──────────────►│
   │               │               │               │               │               │──Send
   │◄──────────────────────────────────────────────────────────────────────────────┤
   │               │               │               │               │   Notification│
```

### 4.2 非同期イベント処理

#### 4.2.1 イベント駆動アーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         Event-Driven Architecture                                    │
│                                                                                      │
│  Producers                    Event Bus                         Consumers            │
│  ──────────                   ─────────                         ─────────            │
│                                                                                      │
│  ┌──────────┐                ┌─────────────────────────────────────────────────┐    │
│  │ Booking  │───────────────►│                                                 │    │
│  │ Service  │ BookingCreated │           Apache Kafka                          │    │
│  └──────────┘                │                                                 │    │
│                              │  ┌─────────────────────────────────────────┐   │    │
│  ┌──────────┐                │  │ Topics:                                  │   │    │
│  │ Payment  │───────────────►│  │  • booking-events                       │   │    │
│  │ Service  │ PaymentComplete│  │  • payment-events                       │───┼───►│
│  └──────────┘                │  │  • user-events                          │   │    │
│                              │  │  • inventory-events                     │   │    │
│  ┌──────────┐                │  │  • notification-events                  │   │    │
│  │ Commerce │───────────────►│  └─────────────────────────────────────────┘   │    │
│  │ Service  │ OrderPlaced    │                                                 │    │
│  └──────────┘                └────────────────────────────────────┬────────────┘    │
│                                                                    │                 │
│                                                                    │                 │
│                                    ┌───────────────────────────────┼─────────────┐  │
│                                    │                               │             │  │
│                                    ▼                               ▼             ▼  │
│                             ┌──────────┐                    ┌──────────┐  ┌──────────┐
│                             │Analytics │                    │Notification│ │Inventory │
│                             │ Service  │                    │ Service   │ │ Service  │
│                             └──────────┘                    └──────────┘  └──────────┘
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### 4.2.2 イベントスキーマ定義

```typescript
// Event Base Schema
interface DomainEvent {
  eventId: string;          // UUID
  eventType: string;        // e.g., "BookingCreated"
  aggregateId: string;      // e.g., booking ID
  aggregateType: string;    // e.g., "Booking"
  timestamp: string;        // ISO 8601
  version: number;          // Schema version
  metadata: {
    correlationId: string;
    causationId: string;
    userId?: string;
    traceId: string;
  };
  payload: Record<string, unknown>;
}

// Booking Events
interface BookingCreatedEvent extends DomainEvent {
  eventType: "BookingCreated";
  payload: {
    bookingId: string;
    userId: string;
    type: "hotel" | "attraction" | "restaurant";
    items: Array<{
      resourceId: string;
      resourceType: string;
      date: string;
      quantity: number;
      unitPrice: number;
    }>;
    totalAmount: number;
    currency: string;
  };
}

interface BookingConfirmedEvent extends DomainEvent {
  eventType: "BookingConfirmed";
  payload: {
    bookingId: string;
    userId: string;
    paymentId: string;
    confirmedAt: string;
  };
}

// Payment Events
interface PaymentCompletedEvent extends DomainEvent {
  eventType: "PaymentCompleted";
  payload: {
    paymentId: string;
    bookingId: string;
    userId: string;
    amount: number;
    currency: string;
    provider: string;
    transactionId: string;
  };
}
```

#### 4.2.3 Saga パターンによる分散トランザクション

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    Booking Saga (Choreography Pattern)                               │
│                                                                                      │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐                   │
│  │ Booking  │────►│ Inventory│────►│ Payment  │────►│Notification│                  │
│  │ Service  │     │ Service  │     │ Service  │     │ Service   │                  │
│  └──────────┘     └──────────┘     └──────────┘     └──────────┘                   │
│       │                │                │                │                          │
│       │ BookingCreated │                │                │                          │
│       ├───────────────►│                │                │                          │
│       │                │ InventoryReserved               │                          │
│       │                ├───────────────►│                │                          │
│       │                │                │ PaymentCompleted                          │
│       │                │                ├───────────────►│                          │
│       │                │                │                │ NotificationSent         │
│       │                │                │                ├───────────────►          │
│       │                │                │                │                          │
│  Compensating Transactions (Rollback):                                              │
│       │                │                │                │                          │
│       │ PaymentFailed  │                │                │                          │
│       │◄───────────────┼────────────────┤                │                          │
│       │                │ InventoryReleased               │                          │
│       │◄───────────────┤                │                │                          │
│       │ BookingCancelled                │                │                          │
│       │                │                │                │                          │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

**Saga オーケストレーター実装**:

```typescript
// Saga Orchestrator
class BookingSagaOrchestrator {
  private readonly steps: SagaStep[] = [
    new CreateBookingStep(),
    new ReserveInventoryStep(),
    new ProcessPaymentStep(),
    new ConfirmBookingStep(),
    new SendNotificationStep(),
  ];

  async execute(command: CreateBookingCommand): Promise<SagaResult> {
    const context = new SagaContext(command);
    const executedSteps: SagaStep[] = [];

    try {
      for (const step of this.steps) {
        await step.execute(context);
        executedSteps.push(step);
      }
      return SagaResult.success(context.getBookingId());
    } catch (error) {
      // Compensate in reverse order
      for (const step of executedSteps.reverse()) {
        try {
          await step.compensate(context);
        } catch (compensationError) {
          // Log and continue with other compensations
          logger.error('Compensation failed', { step: step.name, error: compensationError });
        }
      }
      return SagaResult.failure(error);
    }
  }
}

// Saga Step Interface
interface SagaStep {
  name: string;
  execute(context: SagaContext): Promise<void>;
  compensate(context: SagaContext): Promise<void>;
}

// Example Step Implementation
class ReserveInventoryStep implements SagaStep {
  name = 'ReserveInventory';

  async execute(context: SagaContext): Promise<void> {
    const reservation = await this.inventoryService.reserve({
      items: context.getBookingItems(),
      expiresAt: context.getReservationExpiry(),
    });
    context.setInventoryReservationId(reservation.id);
  }

  async compensate(context: SagaContext): Promise<void> {
    const reservationId = context.getInventoryReservationId();
    if (reservationId) {
      await this.inventoryService.release(reservationId);
    }
  }
}
```

### 4.3 バッチ処理パイプライン

#### 4.3.1 バッチ処理アーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         Batch Processing Pipeline                                    │
│                                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │                         Cloud Scheduler (Cron)                                │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │  │
│  │  │ Daily 2:00  │  │ Hourly      │  │ Weekly Sun  │  │ Monthly 1st │         │  │
│  │  │ Analytics   │  │ Sync        │  │ Reports     │  │ Settlement  │         │  │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │  │
│  └─────────┼────────────────┼────────────────┼────────────────┼─────────────────┘  │
│            │                │                │                │                     │
│            ▼                ▼                ▼                ▼                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                           Cloud Pub/Sub                                      │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│            │                │                │                │                     │
│            ▼                ▼                ▼                ▼                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                      Cloud Dataflow / Apache Beam                            │   │
│  │                                                                              │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │   │
│  │  │ Analytics   │  │ Search      │  │ Report      │  │ Financial   │        │   │
│  │  │ Pipeline    │  │ Indexing    │  │ Generation  │  │ Settlement  │        │   │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │   │
│  └─────────┼────────────────┼────────────────┼────────────────┼─────────────────┘   │
│            │                │                │                │                     │
│            ▼                ▼                ▼                ▼                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │   BigQuery   │  │Elasticsearch │  │ Cloud Storage│  │  PostgreSQL  │           │
│  │              │  │              │  │    (PDF)     │  │              │           │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘           │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### 4.3.2 主要バッチジョブ

| ジョブ名 | スケジュール | 処理内容 | SLA |
|----------|--------------|----------|-----|
| Analytics ETL | 毎日 2:00 | ユーザー行動データをBigQueryに集約 | 4時間以内 |
| Search Reindex | 毎時 | 更新された商品/ホテルをES再インデックス | 30分以内 |
| Report Generation | 毎週日曜 | 週次ビジネスレポート生成 | 2時間以内 |
| Settlement | 毎月1日 | パートナーへの精算処理 | 24時間以内 |
| Data Cleanup | 毎日 3:00 | 期限切れデータの削除 | 2時間以内 |
| Backup | 毎日 4:00 | フルバックアップ | 4時間以内 |

---

## 第5章：サービス間通信

### 5.1 RESTful API設計原則

#### 5.1.1 API設計ガイドライン

**URL設計**:
```
# リソース指向
GET    /api/v1/bookings              # 一覧取得
POST   /api/v1/bookings              # 新規作成
GET    /api/v1/bookings/{id}         # 詳細取得
PUT    /api/v1/bookings/{id}         # 更新
DELETE /api/v1/bookings/{id}         # 削除

# サブリソース
GET    /api/v1/bookings/{id}/items   # 予約アイテム一覧
POST   /api/v1/bookings/{id}/cancel  # アクション

# クエリパラメータ
GET    /api/v1/bookings?status=confirmed&page=1&limit=20&sort=-createdAt
```

**レスポンス形式**:
```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "type": "hotel",
    "status": "confirmed",
    "items": [...],
    "totalAmount": 25000,
    "currency": "JPY",
    "createdAt": "2024-01-15T10:30:00Z"
  },
  "meta": {
    "requestId": "req-123456",
    "timestamp": "2024-01-15T10:30:01Z"
  }
}

// エラーレスポンス
{
  "success": false,
  "error": {
    "code": "BOOKING_NOT_FOUND",
    "message": "The requested booking does not exist",
    "details": {
      "bookingId": "550e8400-e29b-41d4-a716-446655440000"
    }
  },
  "meta": {
    "requestId": "req-123457",
    "timestamp": "2024-01-15T10:30:02Z"
  }
}
```

**ページネーション**:
```json
{
  "success": true,
  "data": [...],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8,
    "hasNext": true,
    "hasPrev": false
  },
  "links": {
    "self": "/api/v1/bookings?page=1&limit=20",
    "next": "/api/v1/bookings?page=2&limit=20",
    "last": "/api/v1/bookings?page=8&limit=20"
  }
}
```

### 5.2 gRPC採用領域

#### 5.2.1 gRPC使用ケース

| 通信パターン | プロトコル | 理由 |
|--------------|------------|------|
| 外部クライアント ↔ API GW | REST | 広い互換性、キャッシュ可能 |
| API GW ↔ サービス | gRPC | 低レイテンシ、型安全 |
| サービス ↔ サービス | gRPC | 高スループット、ストリーミング |
| 非同期通信 | Kafka | 疎結合、永続化 |

#### 5.2.2 Protocol Buffer定義

```protobuf
syntax = "proto3";

package triptrip.booking.v1;

import "google/protobuf/timestamp.proto";

// Booking Service
service BookingService {
  rpc CreateBooking(CreateBookingRequest) returns (Booking);
  rpc GetBooking(GetBookingRequest) returns (Booking);
  rpc UpdateBooking(UpdateBookingRequest) returns (Booking);
  rpc CancelBooking(CancelBookingRequest) returns (Booking);
  rpc ListBookings(ListBookingsRequest) returns (ListBookingsResponse);

  // Streaming
  rpc StreamBookingUpdates(StreamBookingUpdatesRequest)
    returns (stream BookingUpdate);
}

message Booking {
  string id = 1;
  string user_id = 2;
  BookingType type = 3;
  BookingStatus status = 4;
  repeated BookingItem items = 5;
  Money total_amount = 6;
  google.protobuf.Timestamp created_at = 7;
  google.protobuf.Timestamp updated_at = 8;
}

message BookingItem {
  string id = 1;
  string resource_id = 2;
  string resource_type = 3;
  google.protobuf.Timestamp date = 4;
  optional TimeSlot time_slot = 5;
  int32 quantity = 6;
  Money unit_price = 7;
  Money subtotal = 8;
}

message Money {
  int64 amount = 1;  // Minor units (e.g., cents)
  string currency = 2;
}

enum BookingType {
  BOOKING_TYPE_UNSPECIFIED = 0;
  BOOKING_TYPE_HOTEL = 1;
  BOOKING_TYPE_ATTRACTION = 2;
  BOOKING_TYPE_RESTAURANT = 3;
}

enum BookingStatus {
  BOOKING_STATUS_UNSPECIFIED = 0;
  BOOKING_STATUS_PENDING = 1;
  BOOKING_STATUS_CONFIRMED = 2;
  BOOKING_STATUS_CANCELLED = 3;
  BOOKING_STATUS_COMPLETED = 4;
}

message CreateBookingRequest {
  string user_id = 1;
  BookingType type = 2;
  repeated CreateBookingItem items = 3;
}

message CreateBookingItem {
  string resource_id = 1;
  google.protobuf.Timestamp date = 2;
  optional TimeSlot time_slot = 3;
  int32 quantity = 4;
}
```

### 5.3 イベント駆動メッセージング（Kafka）

#### 5.3.1 Kafka クラスター構成

```yaml
kafka_cluster:
  brokers: 3
  replication_factor: 3
  min_insync_replicas: 2

  topics:
    booking-events:
      partitions: 12
      retention_hours: 168  # 7 days
      cleanup_policy: delete

    payment-events:
      partitions: 6
      retention_hours: 720  # 30 days
      cleanup_policy: compact

    user-events:
      partitions: 12
      retention_hours: 168
      cleanup_policy: delete

    notification-events:
      partitions: 6
      retention_hours: 72  # 3 days
      cleanup_policy: delete

    analytics-events:
      partitions: 24
      retention_hours: 2160  # 90 days
      cleanup_policy: delete
```

#### 5.3.2 Producer/Consumer 設定

```typescript
// Kafka Producer Configuration
const producerConfig: ProducerConfig = {
  clientId: 'booking-service',
  brokers: ['kafka-1:9092', 'kafka-2:9092', 'kafka-3:9092'],

  // Reliability settings
  acks: -1,  // Wait for all replicas
  idempotent: true,
  maxInFlightRequests: 5,

  // Performance settings
  batchSize: 16384,
  linger: 10,  // ms
  compression: CompressionTypes.LZ4,

  // Retry settings
  retries: 3,
  retryBackoff: 100,
};

// Kafka Consumer Configuration
const consumerConfig: ConsumerConfig = {
  clientId: 'payment-service',
  groupId: 'payment-service-group',
  brokers: ['kafka-1:9092', 'kafka-2:9092', 'kafka-3:9092'],

  // Consumer settings
  sessionTimeout: 30000,
  heartbeatInterval: 3000,
  maxBytesPerPartition: 1048576,

  // Offset management
  autoCommit: false,
  fromBeginning: false,
};

// Event Publishing
class EventPublisher {
  private producer: Producer;

  async publish<T extends DomainEvent>(event: T): Promise<void> {
    const message = {
      key: event.aggregateId,
      value: JSON.stringify(event),
      headers: {
        'event-type': event.eventType,
        'correlation-id': event.metadata.correlationId,
        'trace-id': event.metadata.traceId,
      },
      timestamp: new Date(event.timestamp).getTime().toString(),
    };

    await this.producer.send({
      topic: this.getTopicForEvent(event),
      messages: [message],
    });
  }

  private getTopicForEvent(event: DomainEvent): string {
    const topicMap: Record<string, string> = {
      'BookingCreated': 'booking-events',
      'BookingConfirmed': 'booking-events',
      'PaymentCompleted': 'payment-events',
      'UserRegistered': 'user-events',
    };
    return topicMap[event.eventType] || 'default-events';
  }
}
```

### 5.4 APIゲートウェイパターン

#### 5.4.1 Kong Gateway 構成

```yaml
# Kong Configuration
_format_version: "3.0"

services:
  - name: booking-service
    url: http://booking-service:3000
    routes:
      - name: booking-routes
        paths:
          - /api/v1/bookings
        strip_path: false
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: redis
          redis_host: redis
      - name: jwt
        config:
          secret_is_base64: false

  - name: search-service
    url: http://search-service:3000
    routes:
      - name: search-routes
        paths:
          - /api/v1/search
    plugins:
      - name: rate-limiting
        config:
          minute: 1000
      - name: response-transformer
        config:
          add:
            headers:
              - "X-Cache-Status:HIT"

  - name: payment-service
    url: http://payment-service:3000
    routes:
      - name: payment-routes
        paths:
          - /api/v1/payments
    plugins:
      - name: rate-limiting
        config:
          minute: 50
      - name: request-termination
        config:
          status_code: 503
          message: "Payment service temporarily unavailable"
        enabled: false  # Circuit breaker (enable when needed)

plugins:
  - name: correlation-id
    config:
      header_name: X-Correlation-ID
      generator: uuid
      echo_downstream: true

  - name: prometheus
    config:
      per_consumer: true

  - name: request-transformer
    config:
      add:
        headers:
          - "X-Request-Start:$(date +%s%N)"
```

#### 5.4.2 リクエストルーティングとロードバランシング

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              API Gateway Architecture                                │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                              Kong Gateway                                       │ │
│  │                                                                                 │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │ │
│  │  │   Auth      │  │    Rate     │  │   Request   │  │  Response   │           │ │
│  │  │   Plugin    │─►│   Limiting  │─►│   Transform │─►│  Transform  │           │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘           │ │
│  │                                                                                 │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐  │ │
│  │  │                         Router                                          │  │ │
│  │  │                                                                         │  │ │
│  │  │    /api/v1/bookings/*  ───►  Booking Service (Upstream)                │  │ │
│  │  │    /api/v1/search/*    ───►  Search Service (Upstream)                 │  │ │
│  │  │    /api/v1/payments/*  ───►  Payment Service (Upstream)                │  │ │
│  │  │    /api/v1/users/*     ───►  User Service (Upstream)                   │  │ │
│  │  │                                                                         │  │ │
│  │  └─────────────────────────────────────────────────────────────────────────┘  │ │
│  │                                                                                 │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐  │ │
│  │  │                    Load Balancer (per Upstream)                         │  │ │
│  │  │                                                                         │  │ │
│  │  │    Algorithm: round-robin | least-connections | consistent-hashing     │  │ │
│  │  │    Health Checks: active + passive                                      │  │ │
│  │  │    Circuit Breaker: enabled                                             │  │ │
│  │  │                                                                         │  │ │
│  │  └─────────────────────────────────────────────────────────────────────────┘  │ │
│  │                                                                                 │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
│                          │                │                │                         │
│                          ▼                ▼                ▼                         │
│                   ┌──────────┐     ┌──────────┐     ┌──────────┐                   │
│                   │ Pod 1    │     │ Pod 2    │     │ Pod 3    │                   │
│                   │ Booking  │     │ Booking  │     │ Booking  │                   │
│                   │ Service  │     │ Service  │     │ Service  │                   │
│                   └──────────┘     └──────────┘     └──────────┘                   │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 第6章：データフローとインテグレーション

### 6.1 内部データフロー

#### 6.1.1 データフローアーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           Internal Data Flow                                         │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                              OLTP Layer                                         │ │
│  │                                                                                 │ │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐                 │ │
│  │  │  User    │    │ Booking  │    │ Commerce │    │ Payment  │                 │ │
│  │  │   DB     │    │   DB     │    │   DB     │    │   DB     │                 │ │
│  │  │(Postgres)│    │(Postgres)│    │(Postgres)│    │(Postgres)│                 │ │
│  │  └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘                 │ │
│  │       │               │               │               │                        │ │
│  └───────┼───────────────┼───────────────┼───────────────┼────────────────────────┘ │
│          │               │               │               │                          │
│          └───────────────┴───────────────┴───────────────┘                          │
│                                    │                                                 │
│                                    ▼                                                 │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                         CDC (Debezium)                                          │ │
│  │       Change Data Capture from all OLTP databases                               │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                                 │
│                                    ▼                                                 │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                         Apache Kafka                                            │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │ │
│  │  │ user-cdc    │  │ booking-cdc │  │commerce-cdc │  │ payment-cdc │           │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘           │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                                 │
│          ┌─────────────────────────┼─────────────────────────┐                      │
│          │                         │                         │                      │
│          ▼                         ▼                         ▼                      │
│  ┌──────────────┐          ┌──────────────┐          ┌──────────────┐              │
│  │ Stream       │          │ Batch ETL    │          │ Real-time    │              │
│  │ Processing   │          │ (Dataflow)   │          │ Analytics    │              │
│  │ (Flink)      │          │              │          │ (ClickHouse) │              │
│  └──────┬───────┘          └──────┬───────┘          └──────┬───────┘              │
│         │                         │                         │                       │
│         ▼                         ▼                         ▼                       │
│  ┌──────────────┐          ┌──────────────┐          ┌──────────────┐              │
│  │Elasticsearch │          │   BigQuery   │          │  Dashboards  │              │
│  │   (Search)   │          │    (DWH)     │          │   (Grafana)  │              │
│  └──────────────┘          └──────────────┘          └──────────────┘              │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 外部システム統合（OTA、決済、地図）

#### 6.2.1 外部統合アーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        External System Integration                                   │
│                                                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                       Integration Layer                                         │ │
│  │                                                                                 │ │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐             │ │
│  │  │   OTA Adapter    │  │  Payment Adapter │  │    Map Adapter   │             │ │
│  │  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘             │ │
│  │           │                     │                     │                        │ │
│  └───────────┼─────────────────────┼─────────────────────┼────────────────────────┘ │
│              │                     │                     │                          │
│              ▼                     ▼                     ▼                          │
│  ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐                 │
│  │   OTA Partners    │ │ Payment Providers │ │   Map Services    │                 │
│  │                   │ │                   │ │                   │                 │
│  │  ┌─────────────┐  │ │  ┌─────────────┐  │ │  ┌─────────────┐  │                 │
│  │  │   Booking   │  │ │  │   Stripe    │  │ │  │Google Maps  │  │                 │
│  │  │    .com     │  │ │  └─────────────┘  │ │  └─────────────┘  │                 │
│  │  └─────────────┘  │ │  ┌─────────────┐  │ │  ┌─────────────┐  │                 │
│  │  ┌─────────────┐  │ │  │   PayPay    │  │ │  │   Mapbox    │  │                 │
│  │  │   Expedia   │  │ │  └─────────────┘  │ │  └─────────────┘  │                 │
│  │  └─────────────┘  │ │  ┌─────────────┐  │ │                   │                 │
│  │  ┌─────────────┐  │ │  │ Apple Pay   │  │ │                   │                 │
│  │  │    Agoda    │  │ │  └─────────────┘  │ │                   │                 │
│  │  └─────────────┘  │ │  ┌─────────────┐  │ │                   │                 │
│  │                   │ │  │ Google Pay  │  │ │                   │                 │
│  │                   │ │  └─────────────┘  │ │                   │                 │
│  └───────────────────┘ └───────────────────┘ └───────────────────┘                 │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### 6.2.2 アダプターパターン実装

```typescript
// External Provider Interface
interface HotelProvider {
  searchHotels(criteria: SearchCriteria): Promise<HotelSearchResult>;
  getHotelDetails(hotelId: string): Promise<HotelDetails>;
  checkAvailability(request: AvailabilityRequest): Promise<Availability>;
  createBooking(request: BookingRequest): Promise<ExternalBooking>;
  cancelBooking(bookingId: string): Promise<CancellationResult>;
}

// Booking.com Adapter
class BookingComAdapter implements HotelProvider {
  private client: BookingComClient;
  private cache: CacheService;
  private circuitBreaker: CircuitBreaker;

  async searchHotels(criteria: SearchCriteria): Promise<HotelSearchResult> {
    return this.circuitBreaker.execute(async () => {
      const cacheKey = `booking_search:${JSON.stringify(criteria)}`;
      const cached = await this.cache.get(cacheKey);
      if (cached) return cached;

      const response = await this.client.post('/v2/hotels/search', {
        checkin: criteria.checkIn,
        checkout: criteria.checkOut,
        room_qty: criteria.rooms,
        guest_qty: criteria.guests,
        dest_id: criteria.destinationId,
        currency: criteria.currency,
      });

      const result = this.mapToInternalFormat(response.data);
      await this.cache.set(cacheKey, result, 300); // 5 min cache

      return result;
    });
  }

  private mapToInternalFormat(externalData: any): HotelSearchResult {
    return {
      hotels: externalData.result.map((h: any) => ({
        id: `booking_${h.hotel_id}`,
        externalId: h.hotel_id,
        provider: 'booking.com',
        name: h.hotel_name,
        rating: h.review_score / 2,
        price: {
          amount: h.min_total_price,
          currency: h.currency_code,
        },
        location: {
          latitude: h.latitude,
          longitude: h.longitude,
          address: h.address,
          city: h.city,
        },
        images: h.main_photo_url ? [h.main_photo_url] : [],
      })),
      totalCount: externalData.result.length,
    };
  }
}

// Aggregator Service
class HotelAggregatorService {
  private providers: Map<string, HotelProvider>;

  async searchAllProviders(
    criteria: SearchCriteria
  ): Promise<AggregatedSearchResult> {
    const results = await Promise.allSettled(
      Array.from(this.providers.entries()).map(async ([name, provider]) => {
        const result = await provider.searchHotels(criteria);
        return { provider: name, result };
      })
    );

    const successfulResults = results
      .filter((r): r is PromiseFulfilledResult<any> => r.status === 'fulfilled')
      .map((r) => r.value);

    // Merge and deduplicate results
    const hotels = this.mergeAndRank(successfulResults);

    return {
      hotels,
      providers: successfulResults.map((r) => r.provider),
      errors: results
        .filter((r): r is PromiseRejectedResult => r.status === 'rejected')
        .map((r) => r.reason),
    };
  }
}
```

### 6.3 サードパーティAPI管理

#### 6.3.1 API管理戦略

```yaml
api_management:
  rate_limiting:
    booking_com:
      requests_per_second: 50
      burst: 100
    expedia:
      requests_per_second: 30
      burst: 60
    stripe:
      requests_per_second: 100
      burst: 200

  circuit_breaker:
    failure_threshold: 5
    recovery_timeout: 30s
    half_open_requests: 3

  retry_policy:
    max_retries: 3
    initial_delay: 100ms
    max_delay: 5s
    exponential_backoff: true
    retryable_errors:
      - TIMEOUT
      - RATE_LIMITED
      - SERVER_ERROR

  caching:
    search_results:
      ttl: 300s
      strategy: cache_aside
    static_content:
      ttl: 86400s
      strategy: read_through
```

---

## 第7章：戦略的 & 技術的示唆

### 7.1 主要なアーキテクチャ洞察

#### 7.1.1 設計判断の根拠

| 決定事項 | 選択 | 根拠 |
|----------|------|------|
| サービス間通信 | gRPC（内部）+ REST（外部） | 内部は性能重視、外部は互換性重視 |
| イベントバス | Apache Kafka | 高スループット、永続性、エコシステム |
| データストア | ポリグロット永続化 | 用途に最適なストアを選択 |
| コンテナオーケストレーション | Kubernetes (GKE) | 業界標準、自動スケーリング |
| API Gateway | Kong | オープンソース、プラグインエコシステム |

#### 7.1.2 アーキテクチャ特性の優先順位

```
1. 可用性（Availability）     ████████████████████ 95%
2. スケーラビリティ            ██████████████████── 90%
3. パフォーマンス              █████████████████─── 85%
4. セキュリティ                █████████████████─── 85%
5. 保守性（Maintainability）  ███████████████───── 75%
6. 開発者体験                  ██████████████────── 70%
```

### 7.2 パフォーマンス & スケーラビリティ成果

#### 7.2.1 期待される改善

| メトリクス | 現状 | 目標（Phase 1） | 目標（Phase 3） |
|------------|------|-----------------|-----------------|
| P95 レイテンシ | 500ms | 200ms | 100ms |
| TPS | 500 | 5,000 | 100,000 |
| 可用性 | 99% | 99.9% | 99.99% |
| デプロイ頻度 | 週1回 | 日1回 | 日10回+ |
| MTTR | 4時間 | 30分 | 5分 |

### 7.3 リスク軽減戦略

#### 7.3.1 リスクマトリクス

| リスク | 影響度 | 発生確率 | 軽減策 |
|--------|--------|----------|--------|
| 分散システム障害 | 高 | 中 | Circuit Breaker、Saga補償 |
| データ不整合 | 高 | 中 | 結果整合性、CDC |
| サービス間依存 | 中 | 高 | 疎結合化、非同期通信 |
| 技術的複雑性 | 中 | 高 | 段階的移行、ドキュメント整備 |
| 運用負荷増大 | 中 | 高 | 自動化、可観測性強化 |

---

## 第8章：代替案 & トレードオフ分析

### 8.1 検討した選択肢と却下理由

#### 8.1.1 サービス通信プロトコル

| 選択肢 | 長所 | 短所 | 判定 |
|--------|------|------|------|
| REST only | シンプル、広い互換性 | 性能、型安全性 | 外部のみ採用 |
| gRPC only | 高性能、型安全 | 学習コスト、デバッグ困難 | 内部採用 |
| GraphQL | 柔軟なクエリ | 複雑性、キャッシュ困難 | BFFで検討 |

#### 8.1.2 メッセージング基盤

| 選択肢 | 長所 | 短所 | 判定 |
|--------|------|------|------|
| Apache Kafka | 高スループット、永続性 | 運用複雑性 | **採用** |
| RabbitMQ | シンプル、柔軟 | スループット限界 | 通知に部分採用 |
| Google Pub/Sub | マネージド | ベンダーロック | 将来検討 |

### 8.2 決定基準と正当化

#### 8.2.1 技術選定基準

```yaml
selection_criteria:
  performance:
    weight: 25%
    metrics: [latency, throughput, resource_efficiency]

  reliability:
    weight: 25%
    metrics: [availability, fault_tolerance, data_durability]

  maintainability:
    weight: 20%
    metrics: [code_quality, documentation, community_support]

  scalability:
    weight: 15%
    metrics: [horizontal_scaling, auto_scaling, cost_efficiency]

  security:
    weight: 15%
    metrics: [encryption, authentication, compliance]
```

### 8.3 将来の進化パスウェイ

#### 8.3.1 技術進化ロードマップ

```
2026 Q1-Q2: 基盤整備
├── モノリス安定化
├── 可観測性確立
├── CI/CD最適化
└── コンテナ化準備

2026 Q3-Q4: 初期分離
├── User Service分離
├── Search Service分離
├── Kafka導入
└── API Gateway導入

2027 H1: コア分離
├── Booking Service分離
├── Payment Service分離
├── Event Sourcingパイロット
└── gRPC移行開始

2027 H2: 拡張
├── Recommendation Service
├── Analytics Platform
├── ML Pipeline
└── マルチリージョン準備

2028+: 最適化
├── 完全マイクロサービス化
├── グローバル展開
├── エッジコンピューティング
└── AI/ML強化
```

---

## 第9章：実装 & マイグレーションロードマップ

### 9.1 フェーズ1: 基盤（0-6ヶ月）

#### 9.1.1 目標と成果物

```yaml
phase_1:
  objectives:
    - モノリシックアプリケーションの安定化
    - 可観測性プラットフォームの確立
    - CI/CDパイプラインの最適化
    - コンテナ化の準備

  deliverables:
    infrastructure:
      - Kubernetes クラスター構築（GKE）
      - Prometheus/Grafana 監視スタック
      - Jaeger 分散トレーシング
      - ELK ログ集約

    application:
      - Docker化（全サービス）
      - OpenTelemetry 統合
      - Health Check エンドポイント
      - Graceful Shutdown 実装

    process:
      - GitOps ワークフロー
      - 自動テスト強化
      - デプロイ自動化
      - インシデント対応手順

  milestones:
    - Month 1: 監視基盤構築完了
    - Month 2: コンテナ化完了
    - Month 3: K8s 開発環境稼働
    - Month 4: K8s ステージング稼働
    - Month 5: K8s 本番移行開始
    - Month 6: 本番完全移行
```

### 9.2 フェーズ2: スケール（6-18ヶ月）

#### 9.2.1 サービス分離計画

```yaml
phase_2:
  objectives:
    - 主要サービスの分離
    - イベント駆動アーキテクチャ導入
    - データベース分離
    - 自動スケーリング実現

  service_extraction_order:
    1_user_service:
      timeline: Month 7-9
      dependencies: none
      database: PostgreSQL (dedicated)
      risks: low

    2_search_service:
      timeline: Month 9-11
      dependencies: user_service
      database: Elasticsearch
      risks: medium

    3_notification_service:
      timeline: Month 10-12
      dependencies: user_service
      database: MongoDB
      risks: low

    4_booking_service:
      timeline: Month 12-15
      dependencies: [user_service, search_service]
      database: PostgreSQL (dedicated)
      risks: high

    5_payment_service:
      timeline: Month 15-18
      dependencies: [user_service, booking_service]
      database: PostgreSQL (dedicated)
      risks: critical

  infrastructure:
    - Kafka クラスター構築
    - Service Mesh (Istio) 導入
    - API Gateway (Kong) 導入
    - Redis Cluster 構築
```

### 9.3 フェーズ3: 最適化（18ヶ月以上）

#### 9.3.1 高度化計画

```yaml
phase_3:
  objectives:
    - 完全マイクロサービス化
    - グローバル展開
    - ML/AI統合
    - 継続的最適化

  deliverables:
    services:
      - Recommendation Service
      - Analytics Service
      - Content Management Service
      - Admin Service

    infrastructure:
      - マルチリージョン展開
      - エッジコンピューティング
      - ML Platform (Vertex AI)
      - Data Lake (BigQuery)

    optimization:
      - パフォーマンスチューニング
      - コスト最適化
      - セキュリティ強化
      - 運用自動化
```

---

## 第10章：文書間参照 & 依存関係

### 10.1 関連文書

| 文書ID | 文書名 | 関係 |
|--------|--------|------|
| Doc-TV-001 | 技術ビジョンとアーキテクチャ原則 | 親文書（原則定義） |
| Doc-SA-002 | マイクロサービス詳細設計 | 子文書（詳細設計） |
| Doc-DA-001 | データアーキテクチャ設計 | 関連文書（データ層） |
| Doc-SEC-001 | セキュリティアーキテクチャ | 関連文書（セキュリティ） |
| Doc-INF-001 | インフラストラクチャ設計 | 関連文書（インフラ） |

### 10.2 依存関係マトリクス

```
                    ┌───────┬───────┬───────┬───────┬───────┬───────┐
                    │ User  │Booking│Commerce│Payment│ Search│ Notif │
        ┌───────────┼───────┼───────┼───────┼───────┼───────┼───────┤
        │ User      │   -   │   ◄   │   ◄   │   ◄   │       │   ◄   │
        ├───────────┼───────┼───────┼───────┼───────┼───────┼───────┤
        │ Booking   │   ►   │   -   │   ◄►  │   ►   │   ◄   │   ►   │
        ├───────────┼───────┼───────┼───────┼───────┼───────┼───────┤
        │ Commerce  │   ►   │   ◄►  │   -   │   ►   │   ◄   │   ►   │
        ├───────────┼───────┼───────┼───────┼───────┼───────┼───────┤
        │ Payment   │   ►   │   ◄   │   ◄   │   -   │       │   ►   │
        ├───────────┼───────┼───────┼───────┼───────┼───────┼───────┤
        │ Search    │       │   ►   │   ►   │       │   -   │       │
        ├───────────┼───────┼───────┼───────┼───────┼───────┼───────┤
        │ Notif     │   ►   │   ◄   │   ◄   │   ◄   │       │   -   │
        └───────────┴───────┴───────┴───────┴───────┴───────┴───────┘

        ► = 依存する（呼び出す側）
        ◄ = 依存される（呼び出される側）
        ◄► = 双方向依存
```

---

## 付録

### A. 用語集

| 用語 | 定義 |
|------|------|
| API Gateway | 外部リクエストの単一エントリーポイント |
| Bounded Context | DDDにおけるドメインの境界 |
| CDC | Change Data Capture（変更データキャプチャ） |
| Circuit Breaker | 障害伝播を防ぐパターン |
| CQRS | Command Query Responsibility Segregation |
| DDD | Domain-Driven Design |
| Event Sourcing | イベントをソースとするデータ永続化 |
| gRPC | Google開発の高性能RPC |
| Saga | 分散トランザクションパターン |
| Service Mesh | サービス間通信を管理するインフラ層 |

### B. 参考文献

1. Martin Fowler - "Patterns of Enterprise Application Architecture"
2. Sam Newman - "Building Microservices"
3. Chris Richardson - "Microservices Patterns"
4. Eric Evans - "Domain-Driven Design"
5. Google SRE Book - "Site Reliability Engineering"

---

**文書情報**
- Document ID: Doc-SA-001
- Version: 1.0.0
- Last Updated: 2026-01-19
- Status: Draft
- Total Lines: 1,500+
- Author: Technical Architecture Agent
```

# Doc-SA-002: マイクロサービスアーキテクチャ・設計

## エグゼクティブサマリー

本文書は、TripTripプラットフォームのマイクロサービスアーキテクチャの詳細設計を定義します。ドメイン駆動設計（DDD）に基づくサービス境界の定義、サービス間通信パターン、レジリエンスパターン、デプロイメント戦略を包括的に設計します。既存のFlutter/Node.js/PostgreSQLモノリスから段階的にマイクロサービスへ移行するための具体的な設計指針を提供し、Netflix、Uber、Amazonレベルのスケーラビリティと運用効率を実現します。本設計は、10万MAUから1億MAUへの成長を支える基盤となり、開発チームの独立性と迅速なデリバリーを可能にします。

---

## 第1章：はじめに＆コンテキスト

### 1.1 マイクロサービス採用の背景

#### 1.1.1 現行モノリスの課題

TripTripの現行システムは、Node.js/Honoフレームワークによるモノリシックアーキテクチャで構築されています。現時点で約4,700行のバックエンドコードと約35,000行のFlutterフロントエンドコードが存在し、機能的には成熟していますが、スケーラビリティと開発効率の観点で以下の課題が顕在化しています。

```yaml
current_challenges:
  scalability:
    - description: 全機能が単一プロセスで稼働
      impact: 一部機能の負荷増加が全体に影響
      example: 検索機能の負荷がチェックアウト処理に影響

    - description: データベース接続の競合
      impact: 高負荷時のレスポンス劣化
      example: ホリデーシーズンの同時接続数増加

    - description: 単一のスケーリング単位
      impact: リソース効率の低下
      example: 検索のためにCPUを増やすと全機能にコスト発生

  development_velocity:
    - description: コードベースの肥大化
      impact: 変更の影響範囲が不明確
      example: 予約ロジック変更が商品表示に影響

    - description: デプロイの全体依存
      impact: リリースサイクルの長期化
      example: 小さな修正でも全体デプロイが必要

    - description: テストの複雑化
      impact: リグレッションリスクの増大
      example: 統合テストに長時間が必要

  reliability:
    - description: 単一障害点
      impact: 一つのバグが全体停止を招く
      example: メモリリークで全サービスがダウン

    - description: デプロイ時のダウンタイム
      impact: ユーザー体験の低下
      example: 深夜メンテナンスの必要性

  technology_constraints:
    - description: 技術スタックの固定化
      impact: 最適なツール選択の制限
      example: 検索にElasticsearch導入が困難
```

#### 1.1.2 マイクロサービス採用の目的

```yaml
adoption_objectives:
  business_objectives:
    - objective: 市場投入時間の短縮
      target: 新機能リリースサイクル 2週間 → 2日
      mechanism: 独立したサービスデプロイ

    - objective: システム可用性の向上
      target: 99.9% → 99.99%
      mechanism: 障害の局所化、独立したフェイルオーバー

    - objective: グローバル展開の基盤
      target: アジア太平洋 → 全世界
      mechanism: リージョン別サービス展開

  technical_objectives:
    - objective: 独立したスケーリング
      target: サービス別の最適リソース配分
      mechanism: Kubernetes HPA、サービス別メトリクス

    - objective: ポリグロット開発
      target: 機能に最適な言語・フレームワーク選択
      mechanism: Go（高性能）、Python（ML）、Node.js（I/O）

    - objective: 継続的デリバリー
      target: 1日複数回のデプロイ
      mechanism: 独立CI/CDパイプライン

  organizational_objectives:
    - objective: チームの自律性
      target: 機能チームが独立して開発・運用
      mechanism: サービスオーナーシップモデル

    - objective: 障害対応の迅速化
      target: MTTR（平均復旧時間）5分以下
      mechanism: サービス別モニタリング、自動復旧
```

### 1.2 ドメイン駆動設計アプローチ

#### 1.2.1 戦略的設計

```yaml
strategic_design:
  core_domain:
    name: Booking（予約）
    description: TripTripの競争優位性を生む中核領域
    characteristics:
      - 最も複雑なビジネスロジック
      - 高い投資優先度
      - 内製開発必須
    subdomains:
      - ホテル予約
      - アトラクションチケット
      - 在庫管理
      - 料金計算

  supporting_domains:
    - name: User Management（ユーザー管理）
      description: 認証・認可・プロファイル管理
      characteristics:
        - 汎用的な機能
        - 外部サービス活用可能（Auth0等）
      subdomains:
        - 認証・認可
        - プロファイル
        - 設定管理

    - name: Commerce（商取引）
      description: 商品販売・注文管理
      characteristics:
        - 標準的なEコマース機能
        - 成熟したパターンの適用
      subdomains:
        - 商品カタログ
        - カート管理
        - 注文処理

    - name: Payment（決済）
      description: 決済処理・精算
      characteristics:
        - 高いコンプライアンス要件
        - 外部ゲートウェイ連携
      subdomains:
        - 決済処理
        - 返金処理
        - 請求・精算

  generic_domains:
    - name: Notification（通知）
      description: マルチチャネル通知配信
      characteristics:
        - 標準的な機能
        - SaaSサービス活用可能

    - name: Search（検索）
      description: 全文検索・推薦
      characteristics:
        - 汎用検索エンジン活用
        - カスタマイズ層の追加

    - name: Analytics（分析）
      description: 行動分析・BI
      characteristics:
        - 標準的な分析基盤
        - データウェアハウス活用

    - name: Content（コンテンツ）
      description: 商品情報・メディア管理
      characteristics:
        - CMSパターンの適用
        - CDN連携
```

#### 1.2.2 Bounded Context（境界付けられたコンテキスト）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        BOUNDED CONTEXT MAP                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                      CORE DOMAIN                                     │    │
│  │                                                                      │    │
│  │  ┌───────────────────────────────────────────────────────────────┐  │    │
│  │  │                    BOOKING CONTEXT                             │  │    │
│  │  │                                                                │  │    │
│  │  │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │  │    │
│  │  │   │   Hotel     │  │ Attraction  │  │  Inventory  │          │  │    │
│  │  │   │  Booking    │  │   Ticket    │  │  Management │          │  │    │
│  │  │   │             │  │             │  │             │          │  │    │
│  │  │   │ ・予約作成  │  │ ・チケット  │  │ ・在庫照会  │          │  │    │
│  │  │   │ ・変更/取消 │  │   発券      │  │ ・確保/解放 │          │  │    │
│  │  │   │ ・確認処理  │  │ ・QRコード  │  │ ・同期処理  │          │  │    │
│  │  │   └─────────────┘  └─────────────┘  └─────────────┘          │  │    │
│  │  │                                                                │  │    │
│  │  │   Language: Go | Database: PostgreSQL | Event: Kafka           │  │    │
│  │  └───────────────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    SUPPORTING DOMAINS                                │    │
│  │                                                                      │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │    │
│  │  │  USER CONTEXT   │  │ COMMERCE CONTEXT│  │ PAYMENT CONTEXT │     │    │
│  │  │                 │  │                 │  │                 │     │    │
│  │  │ ・認証/認可     │  │ ・商品カタログ  │  │ ・決済処理      │     │    │
│  │  │ ・プロファイル  │  │ ・カート管理    │  │ ・返金処理      │     │    │
│  │  │ ・設定管理      │  │ ・注文処理      │  │ ・請求/精算     │     │    │
│  │  │                 │  │                 │  │                 │     │    │
│  │  │ Lang: Node.js   │  │ Lang: Node.js   │  │ Lang: Node.js   │     │    │
│  │  │ DB: PostgreSQL  │  │ DB: PostgreSQL  │  │ DB: PostgreSQL  │     │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    GENERIC DOMAINS                                   │    │
│  │                                                                      │    │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐       │    │
│  │  │  SEARCH    │ │NOTIFICATION│ │ ANALYTICS  │ │  CONTENT   │       │    │
│  │  │  CONTEXT   │ │  CONTEXT   │ │  CONTEXT   │ │  CONTEXT   │       │    │
│  │  │            │ │            │ │            │ │            │       │    │
│  │  │ Lang: Go   │ │ Lang: Node │ │ Lang: Python│ │ Lang: Node │       │    │
│  │  │ DB: ES     │ │ DB: MongoDB│ │ DB: BigQuery│ │ DB: MongoDB│       │    │
│  │  └────────────┘ └────────────┘ └────────────┘ └────────────┘       │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    CONTEXT RELATIONSHIPS                             │    │
│  │                                                                      │    │
│  │  User ──[Customer/Supplier]──► Booking                               │    │
│  │  Booking ──[Customer/Supplier]──► Payment                            │    │
│  │  Booking ──[Published Language]──► Search                            │    │
│  │  Booking ──[Conformist]──► Notification                              │    │
│  │  Commerce ──[Customer/Supplier]──► Payment                           │    │
│  │  Commerce ──[Shared Kernel]──► Content                               │    │
│  │  All ──[Published Language]──► Analytics                             │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 成功基準とKPI

```yaml
success_criteria:
  technical_kpis:
    deployment_frequency:
      current: 1回/週
      target_6m: 1回/日
      target_12m: 5回/日
      measurement: デプロイメント回数/期間

    lead_time:
      current: 2週間
      target_6m: 3日
      target_12m: 1日
      measurement: コミットからデプロイまでの時間

    mean_time_to_recovery:
      current: 30分
      target_6m: 10分
      target_12m: 5分
      measurement: 障害発生から復旧までの時間

    change_failure_rate:
      current: 15%
      target_6m: 10%
      target_12m: 5%
      measurement: 失敗したデプロイメント/全デプロイメント

  operational_kpis:
    service_availability:
      target: 99.99%
      measurement: (稼働時間/全時間) × 100

    latency_p99:
      target: 200ms
      measurement: 99パーセンタイルレスポンスタイム

    error_rate:
      target: 0.1%
      measurement: エラーリクエスト/全リクエスト

    throughput:
      target: 100,000 req/s
      measurement: 1秒あたりの処理リクエスト数

  business_kpis:
    feature_delivery_time:
      current: 4週間
      target: 1週間
      measurement: 要件定義からリリースまでの時間

    developer_productivity:
      target: 200% improvement
      measurement: 機能あたりの開発時間

    infrastructure_cost_efficiency:
      target: 30% reduction per transaction
      measurement: コスト/トランザクション
```

---

## 第2章：サービストポロジー設計

### 2.1 境界付けられたコンテキスト（Bounded Context）の定義

#### 2.1.1 コンテキスト境界の決定原則

```yaml
boundary_principles:
  domain_alignment:
    description: ビジネスドメインとの整合性
    criteria:
      - 単一のビジネス機能に対応
      - ビジネス用語の一貫性
      - ドメインエキスパートの知識範囲

  data_ownership:
    description: データ所有権の明確化
    criteria:
      - 各コンテキストが自身のデータを所有
      - 他コンテキストのデータは参照のみ
      - データ複製は最小限に

  team_alignment:
    description: チーム構造との整合性
    criteria:
      - 1チーム = 1-3サービス
      - チームの認知負荷を考慮
      - コミュニケーションパスの最適化

  change_frequency:
    description: 変更頻度の類似性
    criteria:
      - 同時に変更される機能をグループ化
      - 安定した機能と変化の激しい機能を分離
```

#### 2.1.2 サービス境界の定義

```yaml
service_boundaries:
  user_service:
    bounded_context: User Management
    responsibilities:
      - ユーザー認証（JWT発行/検証）
      - OAuth2.0/OIDC認証連携
      - ユーザープロファイル管理
      - 設定・嗜好管理
      - セッション管理
    data_ownership:
      - users
      - oauth_accounts
      - sessions
      - user_settings
    external_dependencies:
      - Auth0（Identity Provider）
      - Google/LINE/Apple（OAuth）
    team: Platform Team

  booking_service:
    bounded_context: Booking
    responsibilities:
      - ホテル客室予約
      - アトラクションチケット予約
      - 在庫管理（リアルタイム）
      - 予約ライフサイクル管理
      - キャンセル・変更処理
    data_ownership:
      - bookings
      - booking_items
      - inventory
      - inventory_holds
    external_dependencies:
      - ホテルAPIプロバイダー
      - Payment Service（内部）
    team: Booking Team

  payment_service:
    bounded_context: Payment
    responsibilities:
      - 決済処理（Stripe）
      - 返金処理
      - 請求書・領収書管理
      - 決済履歴
      - PCI-DSS準拠
    data_ownership:
      - payments
      - refunds
      - payment_events
      - invoices
    external_dependencies:
      - Stripe
      - PayPay
    team: Payment Team

  commerce_service:
    bounded_context: Commerce
    responsibilities:
      - 商品カタログ管理
      - カート管理
      - 注文処理
      - 在庫連携
    data_ownership:
      - products
      - categories
      - carts
      - cart_items
      - orders
      - order_items
    external_dependencies:
      - Content Service（商品情報）
      - Payment Service（決済）
    team: Commerce Team

  search_service:
    bounded_context: Search
    responsibilities:
      - 全文検索（ホテル、商品、アトラクション）
      - ファセット検索
      - オートコンプリート
      - 地理空間検索
      - 検索ランキング最適化
    data_ownership:
      - search_indices（Elasticsearch）
      - search_logs
      - popularity_scores
    external_dependencies:
      - Elasticsearch
      - Redis（キャッシュ）
    team: Discovery Team

  content_service:
    bounded_context: Content
    responsibilities:
      - 商品情報管理
      - ホテル情報管理
      - アトラクション情報管理
      - レビュー・評価管理
      - 画像・動画管理
    data_ownership:
      - hotel_contents
      - attraction_contents
      - reviews
      - media_assets
    external_dependencies:
      - Cloud Storage（メディア）
      - CDN（配信）
    team: Content Team

  notification_service:
    bounded_context: Notification
    responsibilities:
      - プッシュ通知
      - メール通知
      - SMS通知
      - アプリ内通知
      - 通知設定管理
    data_ownership:
      - notification_templates
      - notification_logs
      - device_tokens
    external_dependencies:
      - FCM/APNs
      - SendGrid
      - Twilio
    team: Engagement Team

  analytics_service:
    bounded_context: Analytics
    responsibilities:
      - ユーザー行動分析
      - ビジネスインテリジェンス
      - レポート生成
      - 推薦エンジン
    data_ownership:
      - events（BigQuery）
      - user_profiles（分析用）
      - recommendations
    external_dependencies:
      - BigQuery
      - Looker
      - TensorFlow
    team: Data Team
```

### 2.2 サービスマップとドメイン分割

#### 2.2.1 サービス依存関係マップ

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SERVICE DEPENDENCY MAP                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              ┌─────────────┐                                │
│                              │   Client    │                                │
│                              │  (Flutter)  │                                │
│                              └──────┬──────┘                                │
│                                     │                                        │
│                                     ▼                                        │
│                              ┌─────────────┐                                │
│                              │ API Gateway │                                │
│                              │   (Kong)    │                                │
│                              └──────┬──────┘                                │
│                                     │                                        │
│                                     ▼                                        │
│                              ┌─────────────┐                                │
│                              │     BFF     │                                │
│                              │  (GraphQL)  │                                │
│                              └──────┬──────┘                                │
│                                     │                                        │
│           ┌─────────────────────────┼─────────────────────────┐             │
│           │                         │                         │             │
│           ▼                         ▼                         ▼             │
│    ┌─────────────┐           ┌─────────────┐           ┌─────────────┐     │
│    │    User     │           │   Booking   │           │  Commerce   │     │
│    │   Service   │◄─────────►│   Service   │◄─────────►│   Service   │     │
│    └──────┬──────┘           └──────┬──────┘           └──────┬──────┘     │
│           │                         │                         │             │
│           │              ┌──────────┼──────────┐              │             │
│           │              │          │          │              │             │
│           │              ▼          ▼          ▼              │             │
│           │       ┌──────────┐┌──────────┐┌──────────┐       │             │
│           │       │ Payment  ││  Search  ││ Content  │       │             │
│           │       │ Service  ││ Service  ││ Service  │◄──────┘             │
│           │       └──────────┘└──────────┘└──────────┘                     │
│           │              │          │          │                            │
│           │              │          │          │                            │
│           └──────────────┼──────────┼──────────┼────────────────┐          │
│                          │          │          │                │          │
│                          ▼          ▼          ▼                ▼          │
│                   ┌─────────────────────────────────────────────────┐      │
│                   │              Notification Service               │      │
│                   └─────────────────────────────────────────────────┘      │
│                                         │                                   │
│                                         ▼                                   │
│                   ┌─────────────────────────────────────────────────┐      │
│                   │               Analytics Service                 │      │
│                   └─────────────────────────────────────────────────┘      │
│                                                                              │
│  ───────────────────────────────────────────────────────────────────────    │
│                          EVENT BUS (Apache Kafka)                           │
│  ───────────────────────────────────────────────────────────────────────    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 コアドメイン vs サポートドメイン

```yaml
domain_classification:
  core_domain:
    services:
      - booking_service
    characteristics:
      - 競争優位性の源泉
      - 最も複雑なビジネスロジック
      - 最高の技術投資
      - 内製開発必須
    investment_priority: Highest
    technology_choice:
      language: Go
      rationale: 高パフォーマンス、強型付け、並行処理

  supporting_domains:
    services:
      - user_service
      - commerce_service
      - payment_service
    characteristics:
      - ビジネスに必要だが差別化要因ではない
      - 成熟したパターンの適用
      - 一部外部サービス活用可能
    investment_priority: Medium
    technology_choice:
      language: TypeScript/Node.js
      rationale: 既存スキル活用、開発効率

  generic_domains:
    services:
      - search_service
      - content_service
      - notification_service
      - analytics_service
    characteristics:
      - 汎用的な機能
      - 外部サービス/SaaS活用可能
      - 標準的なソリューションで対応
    investment_priority: Low-Medium
    technology_choice:
      language: Mixed (Go, Node.js, Python)
      rationale: 機能に最適な選択
```

---

## 第3章：個別サービス詳細設計

### 3.1 ユーザードメインサービス

#### 3.1.1 認証サービス（JWT、OAuth2）

```yaml
authentication_service:
  service_name: User Service - Authentication Module
  description: JWT発行/検証、OAuth2連携を担当

  api_endpoints:
    # ユーザー登録
    - method: POST
      path: /v1/auth/register
      request_body:
        email: string (required)
        password: string (required, min 8 chars)
        name: string (required)
        language: string (default: ja)
        currency: string (default: JPY)
      response:
        201:
          user_id: string
          access_token: string
          refresh_token: string
          expires_in: number
        400: ValidationError
        409: EmailAlreadyExists

    # ログイン
    - method: POST
      path: /v1/auth/login
      request_body:
        email: string (required)
        password: string (required)
        mfa_code: string (optional)
      response:
        200:
          access_token: string
          refresh_token: string
          expires_in: number
          requires_mfa: boolean
        401: InvalidCredentials
        403: AccountLocked

    # トークンリフレッシュ
    - method: POST
      path: /v1/auth/refresh
      request_body:
        refresh_token: string (required)
      response:
        200:
          access_token: string
          refresh_token: string
          expires_in: number
        401: InvalidToken

    # OAuth認証開始
    - method: GET
      path: /v1/auth/oauth/{provider}
      path_params:
        provider: enum (google, line, apple)
      query_params:
        redirect_uri: string
      response:
        302: Redirect to provider

    # OAuthコールバック
    - method: POST
      path: /v1/auth/oauth/{provider}/callback
      request_body:
        code: string
        state: string
      response:
        200:
          access_token: string
          refresh_token: string
          is_new_user: boolean
        401: OAuthError

  jwt_configuration:
    algorithm: RS256
    access_token_ttl: 15 minutes
    refresh_token_ttl: 7 days
    issuer: triptrip.com
    audience: triptrip-api
    claims:
      - sub (user_id)
      - email
      - roles
      - permissions
      - iat
      - exp

  security_measures:
    password_policy:
      min_length: 8
      require_uppercase: true
      require_lowercase: true
      require_number: true
      require_special: false
      max_age_days: 90

    rate_limiting:
      login_attempts: 5 per 15 minutes
      registration: 3 per hour per IP
      password_reset: 3 per hour

    session_management:
      concurrent_sessions: 5
      session_timeout: 24 hours
      force_logout_on_password_change: true
```

#### 3.1.2 プロファイルサービス

```yaml
profile_service:
  service_name: User Service - Profile Module
  description: ユーザープロファイルの管理

  api_endpoints:
    # プロファイル取得
    - method: GET
      path: /v1/users/me
      headers:
        Authorization: Bearer {token}
      response:
        200:
          id: string
          email: string
          name: string
          avatar_url: string
          language: string
          currency: string
          email_verified: boolean
          mfa_enabled: boolean
          created_at: datetime

    # プロファイル更新
    - method: PATCH
      path: /v1/users/me
      headers:
        Authorization: Bearer {token}
      request_body:
        name: string (optional)
        avatar_url: string (optional)
        language: string (optional)
        currency: string (optional)
      response:
        200: Updated profile
        400: ValidationError

    # アバター画像アップロード
    - method: POST
      path: /v1/users/me/avatar
      content_type: multipart/form-data
      request_body:
        file: binary (max 5MB, jpg/png)
      response:
        200:
          avatar_url: string
        400: InvalidFileFormat
        413: FileTooLarge

  data_model:
    user_profile:
      fields:
        - id: UUID, PK
        - email: VARCHAR(255), UNIQUE, NOT NULL
        - name: VARCHAR(100), NOT NULL
        - avatar_url: VARCHAR(500)
        - language: VARCHAR(10), DEFAULT 'ja'
        - currency: VARCHAR(3), DEFAULT 'JPY'
        - email_verified: BOOLEAN, DEFAULT FALSE
        - created_at: TIMESTAMP
        - updated_at: TIMESTAMP
      indexes:
        - email (unique)
        - created_at
```

#### 3.1.3 嗜好・設定サービス

```yaml
settings_service:
  service_name: User Service - Settings Module
  description: ユーザー設定と嗜好の管理

  api_endpoints:
    # 設定取得
    - method: GET
      path: /v1/users/me/settings
      response:
        200:
          notification:
            email: boolean
            push: boolean
            sms: boolean
            marketing: boolean
          privacy:
            search_history: boolean
            personalization: boolean
            analytics: boolean
          display:
            theme: string
            compact_mode: boolean

    # 設定更新
    - method: PUT
      path: /v1/users/me/settings
      request_body:
        notification: object
        privacy: object
        display: object
      response:
        200: Updated settings

    # 通知設定更新
    - method: PATCH
      path: /v1/users/me/settings/notification
      request_body:
        email: boolean
        push: boolean
        sms: boolean
        marketing: boolean
      response:
        200: Updated notification settings
```

### 3.2 予約ドメインサービス

#### 3.2.1 検索・ディスカバリーサービス

```yaml
search_discovery_service:
  service_name: Search Service
  description: 高速な全文検索とディスカバリー機能

  technology:
    language: Go
    framework: Gin
    search_engine: Elasticsearch 8.x
    cache: Redis

  api_endpoints:
    # ホテル検索
    - method: GET
      path: /v1/search/hotels
      query_params:
        q: string (search query)
        location: string (city/area)
        lat: number (latitude)
        lng: number (longitude)
        radius: number (km)
        check_in: date
        check_out: date
        guests: number
        rooms: number
        min_price: number
        max_price: number
        amenities: string[] (comma separated)
        rating: number (min rating)
        sort: enum (relevance, price_asc, price_desc, rating, distance)
        page: number
        limit: number (default: 20, max: 100)
      response:
        200:
          total: number
          page: number
          limit: number
          results:
            - id: string
              name: string
              description: string
              location:
                lat: number
                lng: number
                address: string
              rating: number
              review_count: number
              price_range:
                min: number
                max: number
                currency: string
              images:
                - url: string
                  alt: string
              amenities: string[]
              availability: boolean
              distance_km: number
          facets:
            amenities:
              - name: string
                count: number
            price_ranges:
              - range: string
                count: number
            ratings:
              - value: number
                count: number

    # アトラクション検索
    - method: GET
      path: /v1/search/attractions
      query_params:
        q: string
        location: string
        category: string[]
        date: date
        min_price: number
        max_price: number
        sort: enum (relevance, price_asc, price_desc, rating, popularity)
        page: number
        limit: number
      response:
        200:
          total: number
          results: array
          facets: object

    # 商品検索
    - method: GET
      path: /v1/search/products
      query_params:
        q: string
        category: string[]
        min_price: number
        max_price: number
        sort: enum
        page: number
        limit: number
      response:
        200:
          total: number
          results: array
          facets: object

    # オートコンプリート
    - method: GET
      path: /v1/search/autocomplete
      query_params:
        q: string (min 2 chars)
        type: enum (hotel, attraction, product, all)
        limit: number (default: 10)
      response:
        200:
          suggestions:
            - text: string
              type: string
              highlight: string
              score: number

    # 人気検索
    - method: GET
      path: /v1/search/popular
      query_params:
        type: enum
        location: string
        limit: number
      response:
        200:
          popular_searches: string[]
          trending_destinations: array

  elasticsearch_index_config:
    hotels:
      settings:
        number_of_shards: 3
        number_of_replicas: 1
        analysis:
          analyzer:
            japanese:
              type: custom
              tokenizer: kuromoji_tokenizer
              filter:
                - kuromoji_baseform
                - kuromoji_part_of_speech
                - cjk_width
                - lowercase
      mappings:
        properties:
          id: { type: keyword }
          name:
            type: text
            analyzer: japanese
            fields:
              keyword: { type: keyword }
              completion: { type: completion }
          description: { type: text, analyzer: japanese }
          location: { type: geo_point }
          rating: { type: float }
          price_range: { type: integer_range }
          amenities: { type: keyword }
          popularity_score: { type: float }
```

#### 3.2.2 在庫管理サービス

```yaml
inventory_service:
  service_name: Booking Service - Inventory Module
  description: リアルタイム在庫管理

  technology:
    language: Go
    database: PostgreSQL
    cache: Redis
    lock: Redis Distributed Lock

  api_endpoints:
    # 在庫照会
    - method: GET
      path: /v1/inventory/{resource_type}/{resource_id}
      path_params:
        resource_type: enum (hotel_room, attraction_ticket)
        resource_id: string
      query_params:
        start_date: date
        end_date: date
        time_slot_id: string (optional)
      response:
        200:
          resource_type: string
          resource_id: string
          availability:
            - date: date
              total_capacity: number
              available: number
              price: number
              currency: string

    # 在庫一時確保
    - method: POST
      path: /v1/inventory/hold
      request_body:
        resource_type: string
        resource_id: string
        date: date
        time_slot_id: string (optional)
        quantity: number
        session_id: string
      response:
        200:
          hold_id: string
          expires_at: datetime
          quantity: number
        409: InsufficientInventory
        423: ResourceLocked

    # 在庫確保解除
    - method: DELETE
      path: /v1/inventory/hold/{hold_id}
      response:
        200: Released
        404: HoldNotFound

    # 在庫確定
    - method: POST
      path: /v1/inventory/confirm
      request_body:
        hold_id: string
        booking_id: string
      response:
        200:
          confirmed: boolean
        404: HoldNotFound
        409: HoldExpired

  inventory_management:
    hold_duration: 10 minutes
    overbooking_threshold: 0% (no overbooking)
    concurrent_hold_limit: 1000 per resource per time slot

    locking_strategy:
      type: Pessimistic (Redis Distributed Lock)
      timeout: 5 seconds
      retry_count: 3
      retry_delay: 100ms

    cache_strategy:
      availability_cache_ttl: 30 seconds
      price_cache_ttl: 60 seconds
      invalidation: Event-based
```

#### 3.2.3 予約オーケストレーションサービス

```yaml
booking_orchestration_service:
  service_name: Booking Service - Orchestration Module
  description: 予約プロセス全体のオーケストレーション

  api_endpoints:
    # 予約作成
    - method: POST
      path: /v1/bookings
      request_body:
        booking_type: enum (hotel, attraction, rental)
        resource_id: string
        check_in_date: date (hotel)
        check_out_date: date (hotel)
        time_slot_id: string (attraction)
        quantity: number
        guest_info:
          name: string
          email: string
          phone: string
        special_requests: string
      response:
        201:
          booking_id: string
          booking_number: string
          status: enum (pending)
          payment_intent:
            client_secret: string
            amount: number
            currency: string
        400: ValidationError
        409: UnavailableResource

    # 予約取得
    - method: GET
      path: /v1/bookings/{booking_id}
      response:
        200:
          id: string
          booking_number: string
          status: string
          booking_type: string
          resource_id: string
          resource_name: string
          dates:
            check_in: date
            check_out: date
            time_slot: datetime
          quantity: number
          pricing:
            subtotal: number
            tax: number
            discount: number
            total: number
            currency: string
          guest_info: object
          payment:
            status: string
            payment_id: string
          created_at: datetime
          confirmed_at: datetime

    # 予約一覧
    - method: GET
      path: /v1/bookings
      query_params:
        status: enum
        booking_type: enum
        start_date: date
        end_date: date
        page: number
        limit: number
      response:
        200:
          total: number
          bookings: array

    # 予約キャンセル
    - method: POST
      path: /v1/bookings/{booking_id}/cancel
      request_body:
        reason: string
      response:
        200:
          booking_id: string
          status: cancelled
          refund:
            amount: number
            status: string
        400: CancellationNotAllowed
        404: BookingNotFound

    # 予約変更
    - method: PATCH
      path: /v1/bookings/{booking_id}
      request_body:
        check_in_date: date (optional)
        check_out_date: date (optional)
        quantity: number (optional)
        guest_info: object (optional)
      response:
        200:
          booking_id: string
          status: string
          price_difference: number
        400: ModificationNotAllowed

  saga_orchestration:
    booking_saga:
      steps:
        - name: ValidateRequest
          service: BookingService
          action: validateBookingRequest
          compensation: null

        - name: HoldInventory
          service: InventoryService
          action: holdInventory
          compensation: releaseInventory

        - name: CreateBookingRecord
          service: BookingService
          action: createBookingRecord
          compensation: deleteBookingRecord

        - name: CreatePaymentIntent
          service: PaymentService
          action: createPaymentIntent
          compensation: cancelPaymentIntent

        - name: NotifyUser
          service: NotificationService
          action: sendBookingPendingNotification
          compensation: null

      timeout: 30 seconds
      retry_policy:
        max_attempts: 3
        backoff: exponential
```

### 3.3 決済ドメインサービス

#### 3.3.1 決済処理サービス

```yaml
payment_processing_service:
  service_name: Payment Service - Processing Module
  description: 決済処理とStripe連携

  api_endpoints:
    # PaymentIntent作成
    - method: POST
      path: /v1/payments/intents
      request_body:
        booking_id: string
        amount: number
        currency: string
        payment_method_types: string[] (default: [card])
        metadata: object
      response:
        200:
          payment_intent_id: string
          client_secret: string
          amount: number
          currency: string
          status: string

    # PaymentIntent確認
    - method: POST
      path: /v1/payments/intents/{intent_id}/confirm
      request_body:
        payment_method_id: string
      response:
        200:
          payment_intent_id: string
          status: string
          next_action: object (3DS等)

    # 決済状態取得
    - method: GET
      path: /v1/payments/{payment_id}
      response:
        200:
          id: string
          payment_number: string
          booking_id: string
          amount: number
          currency: string
          status: string
          payment_method:
            type: string
            card_last_four: string
            card_brand: string
          receipt_url: string
          created_at: datetime

    # Webhook処理
    - method: POST
      path: /v1/payments/webhooks/stripe
      headers:
        Stripe-Signature: string
      request_body: Stripe Event
      response:
        200: Processed
        400: InvalidSignature

  stripe_integration:
    api_version: "2023-10-16"
    webhook_events:
      - payment_intent.succeeded
      - payment_intent.payment_failed
      - payment_intent.canceled
      - charge.refunded
      - charge.dispute.created

    payment_methods:
      - type: card
        supported_brands: [visa, mastercard, amex, jcb]
        3d_secure: required

      - type: konbini
        description: コンビニ決済（国内のみ）

    currency_support:
      primary: JPY
      supported: [JPY, USD, EUR, GBP, AUD, SGD, HKD]
```

#### 3.3.2 精算・返金サービス

```yaml
refund_settlement_service:
  service_name: Payment Service - Refund Module
  description: 返金処理と精算

  api_endpoints:
    # 返金作成
    - method: POST
      path: /v1/payments/{payment_id}/refunds
      request_body:
        amount: number (optional, default: full refund)
        reason: enum (requested_by_customer, duplicate, fraudulent)
        notes: string
      response:
        200:
          refund_id: string
          refund_number: string
          amount: number
          currency: string
          status: string
        400: RefundNotAllowed
        409: RefundAlreadyProcessed

    # 返金状態取得
    - method: GET
      path: /v1/refunds/{refund_id}
      response:
        200:
          id: string
          refund_number: string
          payment_id: string
          amount: number
          currency: string
          status: string
          reason: string
          created_at: datetime
          completed_at: datetime

    # 返金一覧
    - method: GET
      path: /v1/payments/{payment_id}/refunds
      response:
        200:
          refunds: array

  refund_policies:
    hotel:
      - days_before_checkin: 7+
        refund_percentage: 100
      - days_before_checkin: 3-6
        refund_percentage: 50
      - days_before_checkin: 1-2
        refund_percentage: 0
      - days_before_checkin: 0 (no-show)
        refund_percentage: 0

    attraction:
      - hours_before_timeslot: 24+
        refund_percentage: 100
      - hours_before_timeslot: 0-23
        refund_percentage: 0

  settlement:
    schedule: Daily at 00:00 UTC
    payout_delay: 7 business days
    minimum_payout: 10,000 JPY
```

### 3.4 コンテンツドメインサービス

#### 3.4.1 商品カタログサービス

```yaml
product_catalog_service:
  service_name: Content Service - Catalog Module
  description: 商品・ホテル・アトラクション情報管理

  api_endpoints:
    # ホテル詳細取得
    - method: GET
      path: /v1/content/hotels/{hotel_id}
      response:
        200:
          id: string
          name: string
          description: string
          address:
            full: string
            city: string
            prefecture: string
            postal_code: string
            country: string
          location:
            lat: number
            lng: number
          contact:
            phone: string
            email: string
          amenities: array
          images: array
          room_types: array
          policies:
            check_in_time: string
            check_out_time: string
            cancellation: object
          rating:
            average: number
            count: number
          created_at: datetime
          updated_at: datetime

    # アトラクション詳細取得
    - method: GET
      path: /v1/content/attractions/{attraction_id}
      response:
        200:
          id: string
          name: string
          description: string
          category: string
          location: object
          operating_hours: array
          ticket_types: array
          images: array
          rating: object

    # 商品詳細取得
    - method: GET
      path: /v1/content/products/{product_id}
      response:
        200:
          id: string
          sku: string
          name: string
          description: string
          category: object
          price: object
          images: array
          variants: array
          inventory_status: string
```

#### 3.4.2 メディア管理サービス

```yaml
media_management_service:
  service_name: Content Service - Media Module
  description: 画像・動画のアップロードと配信

  api_endpoints:
    # アップロードURL取得
    - method: POST
      path: /v1/media/upload-url
      request_body:
        content_type: string
        file_size: number
        purpose: enum (hotel_image, product_image, user_avatar, review_image)
      response:
        200:
          upload_url: string (presigned URL)
          media_id: string
          expires_at: datetime

    # アップロード完了通知
    - method: POST
      path: /v1/media/{media_id}/complete
      response:
        200:
          media_id: string
          url: string (CDN URL)
          thumbnails:
            small: string
            medium: string
            large: string

    # メディア削除
    - method: DELETE
      path: /v1/media/{media_id}
      response:
        204: Deleted

  media_processing:
    image_formats:
      input: [jpg, jpeg, png, gif, webp]
      output: webp (with jpg fallback)

    image_sizes:
      thumbnail_small: 100x100
      thumbnail_medium: 300x300
      thumbnail_large: 600x600
      full: 1200x1200 (max)

    storage:
      provider: Cloud Storage
      bucket: triptrip-media
      cdn: Cloudflare

    processing:
      - resize (on upload)
      - compress (quality 85%)
      - watermark (optional)
      - metadata_strip
```

#### 3.4.3 レビュー・評価サービス

```yaml
review_rating_service:
  service_name: Content Service - Review Module
  description: レビューと評価の管理

  api_endpoints:
    # レビュー作成
    - method: POST
      path: /v1/reviews
      request_body:
        resource_type: enum (hotel, attraction, product)
        resource_id: string
        booking_id: string (for verified purchase)
        rating: number (1-5)
        title: string
        content: string
        images: array (media_ids)
      response:
        201:
          review_id: string
          status: pending_moderation

    # レビュー一覧取得
    - method: GET
      path: /v1/reviews
      query_params:
        resource_type: string
        resource_id: string
        rating: number
        verified_only: boolean
        sort: enum (newest, oldest, helpful, rating_high, rating_low)
        page: number
        limit: number
      response:
        200:
          total: number
          average_rating: number
          rating_distribution:
            - rating: 5
              count: number
              percentage: number
          reviews: array

    # レビュー「参考になった」
    - method: POST
      path: /v1/reviews/{review_id}/helpful
      response:
        200:
          helpful_count: number

  moderation:
    auto_approve_threshold: 0.8 (ML confidence)
    manual_review_queue: true
    prohibited_content:
      - spam
      - hate_speech
      - personal_information
      - competitor_mentions
```

### 3.5 通知・エンゲージメントサービス

```yaml
notification_engagement_service:
  service_name: Notification Service
  description: マルチチャネル通知配信

  api_endpoints:
    # 通知送信（内部API）
    - method: POST
      path: /v1/notifications/send
      request_body:
        user_id: string
        template_id: string
        channels: array (push, email, sms, in_app)
        data: object (template variables)
        scheduled_at: datetime (optional)
      response:
        202:
          notification_id: string
          status: queued

    # ユーザー通知一覧
    - method: GET
      path: /v1/users/{user_id}/notifications
      query_params:
        status: enum (unread, read, all)
        page: number
        limit: number
      response:
        200:
          total: number
          unread_count: number
          notifications: array

    # 通知既読
    - method: PATCH
      path: /v1/notifications/{notification_id}/read
      response:
        200: marked_as_read

    # デバイストークン登録
    - method: POST
      path: /v1/devices
      request_body:
        token: string
        platform: enum (ios, android, web)
        device_info: object
      response:
        200:
          device_id: string

  notification_templates:
    booking_confirmed:
      channels: [push, email]
      push:
        title: "予約が確定しました"
        body: "{{resource_name}}の予約が確定しました。予約番号: {{booking_number}}"
      email:
        subject: "【TripTrip】予約確認 - {{resource_name}}"
        template_file: booking_confirmed.html

    payment_received:
      channels: [push, email]

    booking_reminder:
      channels: [push]
      trigger: 24 hours before check_in

    review_request:
      channels: [push, email]
      trigger: 2 days after check_out
```

---

## 第4章：サービス間通信アーキテクチャ

### 4.1 同期通信（REST、gRPC）

#### 4.1.1 REST API設計ガイドライン

```yaml
rest_api_guidelines:
  url_conventions:
    base_path: /api/v{version}
    resource_naming: kebab-case, plural
    examples:
      - /api/v1/bookings
      - /api/v1/bookings/{id}
      - /api/v1/bookings/{id}/items
      - /api/v1/hotels/{id}/rooms

  http_methods:
    GET: リソースの取得（冪等）
    POST: リソースの作成
    PUT: リソースの完全置換（冪等）
    PATCH: リソースの部分更新
    DELETE: リソースの削除（冪等）

  status_codes:
    success:
      200: OK (GET, PUT, PATCH)
      201: Created (POST)
      202: Accepted (async operations)
      204: No Content (DELETE)
    client_error:
      400: Bad Request
      401: Unauthorized
      403: Forbidden
      404: Not Found
      409: Conflict
      422: Unprocessable Entity
      429: Too Many Requests
    server_error:
      500: Internal Server Error
      502: Bad Gateway
      503: Service Unavailable
      504: Gateway Timeout

  response_format:
    success:
      data: object | array
      meta:
        page: number
        limit: number
        total: number
    error:
      error:
        code: string
        message: string
        details: array
        request_id: string

  pagination:
    style: cursor-based (preferred) or offset-based
    default_limit: 20
    max_limit: 100
    params:
      - cursor: string (cursor-based)
      - page: number (offset-based)
      - limit: number

  versioning:
    strategy: URL path versioning
    current: v1
    deprecation_notice: 6 months
```

#### 4.1.2 gRPC導入基準

```yaml
grpc_adoption:
  use_cases:
    - サービス間の内部通信（高頻度）
    - ストリーミングが必要な場合
    - 低レイテンシが重要な場合
    - 型安全性が重要な場合

  not_suitable:
    - 外部クライアント向けAPI
    - ブラウザからの直接アクセス
    - 単純なCRUD操作

  service_definitions:
    booking_service.proto: |
      syntax = "proto3";
      package triptrip.booking.v1;

      service BookingService {
        rpc CreateBooking(CreateBookingRequest) returns (CreateBookingResponse);
        rpc GetBooking(GetBookingRequest) returns (Booking);
        rpc ListBookings(ListBookingsRequest) returns (ListBookingsResponse);
        rpc CancelBooking(CancelBookingRequest) returns (CancelBookingResponse);
        rpc StreamBookingUpdates(StreamBookingUpdatesRequest)
          returns (stream BookingUpdate);
      }

      message Booking {
        string id = 1;
        string booking_number = 2;
        BookingStatus status = 3;
        string user_id = 4;
        BookingType booking_type = 5;
        string resource_id = 6;
        Money total_amount = 7;
        google.protobuf.Timestamp created_at = 8;
      }

      enum BookingStatus {
        BOOKING_STATUS_UNSPECIFIED = 0;
        BOOKING_STATUS_PENDING = 1;
        BOOKING_STATUS_CONFIRMED = 2;
        BOOKING_STATUS_CANCELLED = 3;
        BOOKING_STATUS_COMPLETED = 4;
      }

  configuration:
    connection_pool_size: 100
    keepalive_time: 30s
    keepalive_timeout: 10s
    max_message_size: 4MB
    compression: gzip
```

### 4.2 非同期通信（Kafka、RabbitMQ）

#### 4.2.1 イベント駆動アーキテクチャ

```yaml
event_driven_architecture:
  message_broker: Apache Kafka
  rationale:
    - 高スループット（100K+ msg/sec）
    - 永続化とリプレイ
    - パーティショニングによるスケーラビリティ
    - エコシステム（Kafka Connect、Streams）

  event_categories:
    domain_events:
      description: ビジネスドメインで発生した事実
      examples:
        - booking.created
        - booking.confirmed
        - booking.cancelled
        - payment.completed
        - user.registered
      retention: 7 days

    integration_events:
      description: サービス間の通知
      examples:
        - inventory.updated
        - price.changed
        - content.published
      retention: 3 days

    command_events:
      description: 非同期コマンド実行
      examples:
        - notification.send
        - report.generate
        - index.update
      retention: 1 day

  topic_naming:
    convention: "{domain}.{entity}.{event_type}"
    examples:
      - booking.bookings.created
      - booking.bookings.confirmed
      - payment.payments.completed
      - user.users.registered

  event_schema:
    format: CloudEvents 1.0
    serialization: JSON (with Avro for high volume)
    schema_registry: Confluent Schema Registry

    example:
      specversion: "1.0"
      type: "booking.bookings.confirmed"
      source: "/services/booking"
      id: "uuid-v4"
      time: "2026-01-19T10:00:00Z"
      datacontenttype: "application/json"
      data:
        booking_id: "uuid"
        booking_number: "TT-12345"
        user_id: "uuid"
        total_amount: 50000
        currency: "JPY"
```

#### 4.2.2 Sagaパターンと分散トランザクション

```yaml
saga_pattern:
  implementation: Orchestration-based Saga
  orchestrator: Booking Service

  booking_saga:
    name: CreateBookingSaga
    description: 予約作成の分散トランザクション

    steps:
      - step: 1
        name: ValidateBooking
        service: BookingService
        action: validateBookingRequest
        compensation: null
        timeout: 5s

      - step: 2
        name: CheckInventory
        service: InventoryService
        action: checkAndHoldInventory
        compensation: releaseInventoryHold
        timeout: 10s

      - step: 3
        name: CreateBookingRecord
        service: BookingService
        action: createBookingRecord
        compensation: deleteBookingRecord
        timeout: 5s

      - step: 4
        name: CreatePaymentIntent
        service: PaymentService
        action: createPaymentIntent
        compensation: cancelPaymentIntent
        timeout: 30s

      - step: 5
        name: SendNotification
        service: NotificationService
        action: sendBookingPendingNotification
        compensation: null
        timeout: 5s

    error_handling:
      on_failure: Execute compensations in reverse order
      max_retries: 3
      retry_backoff: exponential

  saga_state_machine:
    states:
      - STARTED
      - INVENTORY_HELD
      - BOOKING_CREATED
      - PAYMENT_INITIATED
      - NOTIFICATION_SENT
      - COMPLETED
      - COMPENSATING
      - COMPENSATED
      - FAILED

    transitions:
      STARTED -> INVENTORY_HELD: on inventory hold success
      INVENTORY_HELD -> BOOKING_CREATED: on booking record created
      BOOKING_CREATED -> PAYMENT_INITIATED: on payment intent created
      PAYMENT_INITIATED -> NOTIFICATION_SENT: on notification sent
      NOTIFICATION_SENT -> COMPLETED: saga complete
      * -> COMPENSATING: on step failure
      COMPENSATING -> COMPENSATED: all compensations complete
      COMPENSATING -> FAILED: compensation failed
```

### 4.3 APIバージョニング戦略

```yaml
api_versioning:
  strategy: URL Path Versioning
  format: /api/v{major}

  lifecycle:
    stages:
      - name: Current
        status: Fully supported
        sla: 99.99% availability

      - name: Deprecated
        status: Maintenance only
        duration: 6 months
        notice: Response header "Deprecation: true"

      - name: Sunset
        status: Read-only, no new features
        duration: 3 months
        notice: Response header "Sunset: {date}"

      - name: Retired
        status: Removed
        response: 410 Gone

  breaking_changes:
    definition:
      - Removing endpoint
      - Removing required field
      - Changing field type
      - Changing response structure
      - Changing error codes

    non_breaking:
      - Adding optional field
      - Adding new endpoint
      - Adding optional query parameter
      - Adding new enum value (with default)

  migration_support:
    documentation: Migration guide per version
    tools: API diff generator
    testing: Backward compatibility tests
```

### 4.4 契約テスト（Contract Testing）

```yaml
contract_testing:
  framework: Pact
  approach: Consumer-Driven Contract Testing

  workflow:
    1. Consumer defines expectations (Pact file)
    2. Consumer tests pass against mock
    3. Pact file shared with Provider
    4. Provider verifies against Pact
    5. Pact Broker manages contracts

  pact_broker:
    url: https://pact.triptrip.internal
    features:
      - Contract storage
      - Version management
      - Verification status
      - Can-I-Deploy check

  example_contract:
    consumer: BFF-Mobile
    provider: BookingService
    interactions:
      - description: Get booking by ID
        request:
          method: GET
          path: /v1/bookings/123
          headers:
            Authorization: Bearer token
        response:
          status: 200
          headers:
            Content-Type: application/json
          body:
            id: "123"
            booking_number: like "TT-12345"
            status: "confirmed"

  ci_integration:
    consumer_pipeline:
      - Run consumer tests
      - Generate Pact file
      - Publish to Pact Broker
      - Tag with branch/version

    provider_pipeline:
      - Fetch contracts from Pact Broker
      - Verify provider against contracts
      - Publish verification results
      - Can-I-Deploy check before deploy
```

---

## 第5章：API設計標準

### 5.1 REST API設計ガイドライン

（第4章で詳述済み）

### 5.2 gRPC導入基準

（第4章で詳述済み）

### 5.3 APIバージョニング戦略

（第4章で詳述済み）

### 5.4 契約テスト（Contract Testing）

（第4章で詳述済み）

---

## 第6章：サービスディスカバリ・ロードバランシング

### 6.1 Kubernetes Service Discovery

```yaml
kubernetes_service_discovery:
  mechanism: Kubernetes DNS + Service

  service_types:
    ClusterIP:
      use_case: 内部サービス間通信
      dns_format: "{service}.{namespace}.svc.cluster.local"
      example: "booking-service.production.svc.cluster.local"

    Headless:
      use_case: 個別Pod接続（StatefulSet）
      dns_format: "{pod}.{service}.{namespace}.svc.cluster.local"
      example: "kafka-0.kafka.production.svc.cluster.local"

  service_definition:
    apiVersion: v1
    kind: Service
    metadata:
      name: booking-service
      namespace: production
      labels:
        app: booking-service
    spec:
      type: ClusterIP
      selector:
        app: booking-service
      ports:
        - name: http
          port: 80
          targetPort: 3000
        - name: grpc
          port: 9090
          targetPort: 9090

  endpoint_slices:
    enabled: true
    max_endpoints_per_slice: 100
```

### 6.2 負荷分散戦略

```yaml
load_balancing:
  external:
    provider: Google Cloud Load Balancing
    type: Global HTTPS Load Balancer
    features:
      - SSL termination
      - CDN integration
      - Health checks
      - Auto-scaling backend
    algorithm: Round Robin with session affinity option

  internal:
    provider: Kubernetes Service / Envoy
    algorithms:
      round_robin:
        use_case: Stateless services
        description: 均等にリクエストを分散

      least_connections:
        use_case: 処理時間が不均一なサービス
        description: 接続数が最小のPodに送信

      weighted:
        use_case: カナリアデプロイ
        description: 重み付けによる分散

    configuration:
      kubernetes_service:
        sessionAffinity: None # or ClientIP
        sessionAffinityConfig:
          clientIP:
            timeoutSeconds: 10800

  health_checks:
    liveness:
      path: /health/live
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3

    readiness:
      path: /health/ready
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
```

### 6.3 サービスメッシュ（Istio）導入検討

```yaml
service_mesh:
  product: Istio
  adoption_timeline: Phase 2 (6-12 months)

  rationale:
    - mTLS（サービス間暗号化）
    - 詳細なトラフィック管理
    - 分散トレーシングの統合
    - サーキットブレーカーの一元管理
    - カナリアデプロイの簡素化

  concerns:
    - 運用複雑性の増加
    - リソースオーバーヘッド
    - 学習コスト

  phased_adoption:
    phase_1:
      scope: 開発/ステージング環境
      features:
        - mTLS (permissive mode)
        - Observability (Kiali, Jaeger)

    phase_2:
      scope: 本番環境（コアサービス）
      features:
        - mTLS (strict mode)
        - Traffic management
        - Circuit breaker

    phase_3:
      scope: 全サービス
      features:
        - Full Istio features
        - Policy enforcement
```

---

## 第7章：レジリエンスパターン

### 7.1 サーキットブレーカー

```yaml
circuit_breaker:
  implementation: Resilience4j (Java/Kotlin) / go-breaker (Go)

  configuration:
    failure_rate_threshold: 50%
    slow_call_rate_threshold: 80%
    slow_call_duration_threshold: 2s
    minimum_number_of_calls: 10
    wait_duration_in_open_state: 60s
    permitted_calls_in_half_open_state: 10
    sliding_window_type: COUNT_BASED
    sliding_window_size: 100

  states:
    CLOSED:
      description: 正常状態、リクエストを通す
      transition: OPEN when failure_rate > threshold

    OPEN:
      description: 異常状態、リクエストを即座に失敗させる
      transition: HALF_OPEN after wait_duration

    HALF_OPEN:
      description: 回復確認状態、限定的にリクエストを通す
      transition:
        - CLOSED if permitted_calls succeed
        - OPEN if any call fails

  per_service_config:
    payment_service:
      failure_rate_threshold: 30%
      wait_duration_in_open_state: 30s
      rationale: 決済は厳格に管理

    search_service:
      failure_rate_threshold: 60%
      wait_duration_in_open_state: 10s
      rationale: 検索は多少の失敗を許容

    notification_service:
      failure_rate_threshold: 80%
      wait_duration_in_open_state: 120s
      rationale: 通知は遅延許容
```

### 7.2 リトライ・タイムアウト戦略

```yaml
retry_strategy:
  default_config:
    max_attempts: 3
    initial_interval: 100ms
    multiplier: 2.0
    max_interval: 10s
    randomization_factor: 0.5

  retryable_errors:
    - Connection timeout
    - 502 Bad Gateway
    - 503 Service Unavailable
    - 504 Gateway Timeout
    - 429 Too Many Requests (with backoff)

  non_retryable_errors:
    - 400 Bad Request
    - 401 Unauthorized
    - 403 Forbidden
    - 404 Not Found
    - 409 Conflict
    - 422 Unprocessable Entity

  idempotency:
    requirement: All retried operations must be idempotent
    implementation:
      - Idempotency key in request header
      - Store processed keys in Redis (24h TTL)
      - Return cached response for duplicate keys

timeout_strategy:
  connection_timeout: 5s
  read_timeout: 30s
  write_timeout: 30s

  per_operation:
    search: 5s
    booking_create: 30s
    payment: 60s
    notification: 10s
```

### 7.3 バルクヘッド・レートリミティング

```yaml
bulkhead_pattern:
  purpose: 障害の波及を防ぐ

  implementation:
    thread_pool:
      max_concurrent_calls: 25
      max_wait_duration: 0 (reject immediately)

    semaphore:
      max_concurrent_calls: 100
      max_wait_duration: 500ms

  per_service_config:
    external_api_calls:
      type: thread_pool
      max_concurrent_calls: 10
      rationale: 外部APIは制限が必要

    database_calls:
      type: semaphore
      max_concurrent_calls: 50
      rationale: DB接続プールに合わせる

rate_limiting:
  implementation: Redis + Sliding Window

  tiers:
    anonymous:
      requests_per_minute: 60
      requests_per_hour: 1000

    authenticated:
      requests_per_minute: 600
      requests_per_hour: 10000

    premium:
      requests_per_minute: 6000
      requests_per_hour: 100000

  per_endpoint:
    /auth/login:
      limit: 10/minute
      rationale: Brute force防止

    /search:
      limit: 100/minute
      rationale: 検索負荷制御

    /payments:
      limit: 30/minute
      rationale: 決済は慎重に
```

### 7.4 グレースフルデグラデーション

```yaml
graceful_degradation:
  strategies:
    fallback_response:
      description: エラー時に代替レスポンスを返す
      examples:
        - 検索失敗時: 人気商品を返す
        - 推薦失敗時: デフォルトリストを返す
        - 価格取得失敗時: キャッシュ価格を返す

    feature_toggle:
      description: 機能を動的に無効化
      examples:
        - 高負荷時: リアルタイム在庫チェックを無効化
        - 障害時: 決済手段を制限
        - メンテナンス時: 書き込み操作を制限

    read_only_mode:
      description: 書き込みを一時停止
      trigger:
        - データベース障害
        - 重大なバグ発見
      behavior:
        - 読み取りは継続
        - 書き込みはキューに保存
        - 復旧後に処理

    static_fallback:
      description: 静的コンテンツにフォールバック
      examples:
        - CDN上の静的ページ
        - キャッシュされた検索結果
```

---

## 第8章：監視・観測性

### 8.1 分散トレーシング

```yaml
distributed_tracing:
  framework: OpenTelemetry
  backend: Jaeger / Google Cloud Trace

  configuration:
    sampling:
      type: probabilistic
      rate: 0.1 # 10% in production
      always_sample:
        - errors
        - slow_requests (> 1s)
        - specific_users (debug)

    propagation:
      format: W3C Trace Context
      headers:
        - traceparent
        - tracestate

  instrumentation:
    auto:
      - HTTP clients (Axios, Fetch)
      - gRPC clients
      - Database queries (Prisma)
      - Redis operations
      - Kafka producers/consumers

    manual:
      - Business logic spans
      - External API calls
      - Batch processing

  span_naming:
    convention: "{service}.{operation}"
    examples:
      - "booking-service.create_booking"
      - "payment-service.process_payment"
      - "search-service.search_hotels"
```

### 8.2 サービスヘルスチェック

```yaml
health_checks:
  endpoints:
    liveness:
      path: /health/live
      purpose: Pod再起動判定
      checks:
        - process_running
        - deadlock_detection
      response:
        healthy: 200 OK
        unhealthy: 503 Service Unavailable

    readiness:
      path: /health/ready
      purpose: トラフィック受け入れ判定
      checks:
        - database_connection
        - redis_connection
        - kafka_connection
        - external_api_reachability
      response:
        ready: 200 OK
        not_ready: 503 Service Unavailable

    startup:
      path: /health/startup
      purpose: 初期化完了判定
      checks:
        - migrations_complete
        - cache_warmed
        - connections_established

  response_format:
    status: "healthy" | "unhealthy" | "degraded"
    checks:
      - name: "database"
        status: "healthy"
        latency_ms: 5
      - name: "redis"
        status: "healthy"
        latency_ms: 1
    timestamp: "2026-01-19T10:00:00Z"
```

### 8.3 メトリクス収集

```yaml
metrics_collection:
  framework: Prometheus / OpenTelemetry Metrics
  storage: Prometheus + Thanos (long-term)
  visualization: Grafana

  standard_metrics:
    http_requests:
      name: http_requests_total
      type: counter
      labels: [method, path, status, service]

    http_request_duration:
      name: http_request_duration_seconds
      type: histogram
      labels: [method, path, service]
      buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]

    http_requests_in_flight:
      name: http_requests_in_flight
      type: gauge
      labels: [service]

  business_metrics:
    bookings:
      - bookings_created_total
      - bookings_confirmed_total
      - bookings_cancelled_total
      - booking_value_total (by currency)

    payments:
      - payments_processed_total
      - payments_failed_total
      - payment_amount_total (by currency)

    users:
      - users_registered_total
      - users_active_daily
      - users_active_monthly

  slo_metrics:
    availability:
      target: 99.99%
      metric: sum(rate(http_requests_total{status!~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

    latency:
      target: p99 < 200ms
      metric: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

---

## 第9章：デプロイメント戦略

### 9.1 独立デプロイメント

```yaml
independent_deployment:
  principle: 各サービスは独立してデプロイ可能

  requirements:
    - サービス間の疎結合
    - 後方互換性のあるAPI
    - 機能フラグによる段階的リリース
    - 自動化されたテスト

  ci_cd_pipeline:
    stages:
      - build:
          - lint
          - unit_test
          - security_scan
          - container_build

      - test:
          - integration_test
          - contract_test
          - performance_test

      - deploy_staging:
          - deploy_to_staging
          - smoke_test
          - e2e_test

      - deploy_production:
          - canary_deployment
          - progressive_rollout
          - verification

  deployment_frequency:
    target: 5+ deploys/day per service
    current: 1 deploy/week (monolith)
```

### 9.2 カナリアリリース

```yaml
canary_deployment:
  strategy: Progressive traffic shifting

  phases:
    - phase: 1
      traffic_percentage: 5%
      duration: 15 minutes
      success_criteria:
        - error_rate < 0.1%
        - latency_p99 < 200ms
        - no_critical_alerts

    - phase: 2
      traffic_percentage: 25%
      duration: 30 minutes
      success_criteria:
        - error_rate < 0.1%
        - latency_p99 < 200ms
        - business_metrics_stable

    - phase: 3
      traffic_percentage: 50%
      duration: 1 hour
      success_criteria:
        - all_previous_criteria
        - no_customer_complaints

    - phase: 4
      traffic_percentage: 100%
      duration: N/A

  rollback:
    automatic:
      trigger:
        - error_rate > 1%
        - latency_p99 > 500ms
        - critical_alert
      action: Immediate rollback to previous version

    manual:
      command: kubectl rollout undo deployment/{service}

  implementation:
    kubernetes:
      apiVersion: argoproj.io/v1alpha1
      kind: Rollout
      spec:
        strategy:
          canary:
            steps:
              - setWeight: 5
              - pause: { duration: 15m }
              - setWeight: 25
              - pause: { duration: 30m }
              - setWeight: 50
              - pause: { duration: 1h }
              - setWeight: 100
            analysis:
              templates:
                - templateName: success-rate
                - templateName: latency
```

### 9.3 ブルー/グリーンデプロイメント

```yaml
blue_green_deployment:
  use_cases:
    - データベースマイグレーション
    - 重大な変更
    - 即時ロールバックが必要な場合

  architecture:
    blue_environment:
      status: current_production
      traffic: 100% (before switch)

    green_environment:
      status: new_version
      traffic: 0% (before switch)

  switch_process:
    - Deploy new version to Green
    - Run smoke tests on Green
    - Run integration tests on Green
    - Switch traffic from Blue to Green
    - Monitor for issues
    - Decommission Blue (or keep for rollback)

  rollback:
    process: Switch traffic back to Blue
    time: < 1 minute

  kubernetes_implementation:
    service:
      selector:
        version: green # or blue
```

---

## 第10章：既存システムからの移行

### 10.1 ストラングラーパターン

```yaml
strangler_pattern:
  description: 既存システムを段階的に置き換える

  approach:
    - 新機能は新サービスで実装
    - 既存機能を段階的に新サービスに移行
    - トラフィックを徐々に新サービスに振り向け
    - 移行完了後に旧システムを廃止

  implementation:
    facade:
      type: API Gateway
      responsibility:
        - ルーティング決定
        - トラフィック分割
        - レスポンス統合

    routing_rules:
      - path: /api/v1/search/*
        destination: search-service (new)

      - path: /api/v1/bookings/*
        destination:
          - booking-service (new): 50%
          - monolith: 50%

      - path: /api/v1/*
        destination: monolith (legacy)

  migration_checklist:
    per_service:
      - [ ] データモデル設計
      - [ ] API設計
      - [ ] 実装完了
      - [ ] 単体テスト
      - [ ] 統合テスト
      - [ ] 契約テスト
      - [ ] データ移行計画
      - [ ] カナリアデプロイ
      - [ ] 本番トラフィック移行
      - [ ] モノリスコード削除
```

### 10.2 段階的分解ロードマップ

```yaml
decomposition_roadmap:
  phase_1:
    name: Foundation
    duration: 3 months
    services: []
    infrastructure:
      - API Gateway (Kong)
      - Kubernetes cluster
      - Observability stack (Datadog)
      - CI/CD pipelines
    outcome: マイクロサービス基盤の整備

  phase_2:
    name: Search Service Extraction
    duration: 3 months
    services:
      - Search Service
    dependencies:
      - Elasticsearch cluster
      - Redis cache
      - CDC pipeline (Debezium)
    outcome: 検索機能の独立

  phase_3:
    name: User Service Extraction
    duration: 2 months
    services:
      - User Service
    dependencies:
      - Auth0 integration
    outcome: 認証・ユーザー管理の独立

  phase_4:
    name: Booking Service Extraction
    duration: 4 months
    services:
      - Booking Service
      - Inventory management
    dependencies:
      - Kafka cluster
      - Saga orchestration
    outcome: 予約コア機能の独立

  phase_5:
    name: Commerce & Payment
    duration: 3 months
    services:
      - Commerce Service
      - Payment Service
    dependencies:
      - Stripe integration
    outcome: 決済・商取引の独立

  phase_6:
    name: Supporting Services
    duration: 3 months
    services:
      - Content Service
      - Notification Service
      - Analytics Service
    outcome: 全サービスの独立

  phase_7:
    name: Monolith Retirement
    duration: 2 months
    activities:
      - 残存機能の移行
      - データ移行完了確認
      - モノリス廃止
    outcome: 完全なマイクロサービス化
```

---

## 第11章：戦略的示唆と推奨事項

### 11.1 技術選定の推奨

```yaml
technology_recommendations:
  critical:
    - recommendation: API Gateway導入
      rationale: マイクロサービスの基盤
      product: Kong Gateway
      timeline: Month 1-2

    - recommendation: メッセージブローカー導入
      rationale: サービス間非同期通信
      product: Apache Kafka (Confluent)
      timeline: Month 3-4

    - recommendation: 可観測性基盤構築
      rationale: 分散システムの監視
      product: Datadog or OpenTelemetry + Jaeger
      timeline: Month 1-2

  high_priority:
    - recommendation: サービスメッシュ導入
      rationale: mTLS、トラフィック管理
      product: Istio
      timeline: Month 6-12

    - recommendation: GitOps導入
      rationale: 宣言的インフラ管理
      product: ArgoCD
      timeline: Month 4-6
```

### 11.2 組織的考慮事項

```yaml
organizational_considerations:
  team_structure:
    model: Two-Pizza Teams (5-8 members)
    ownership: 1 team = 1-3 services
    responsibilities:
      - 開発
      - テスト
      - デプロイ
      - 運用

  skill_development:
    required_skills:
      - Kubernetes / Docker
      - Go / TypeScript
      - Distributed systems
      - DevOps / SRE practices
    training_plan:
      - Kubernetes workshop (2 days)
      - Microservices patterns (3 days)
      - Go programming (5 days)
      - SRE fundamentals (2 days)

  process_changes:
    - Shift to continuous delivery
    - Implement feature flags
    - Adopt trunk-based development
    - Establish SLO-based operations
```

---

## 第12章：文書間参照と統合ポイント

### 12.1 関連文書参照

```yaml
document_references:
  prerequisites:
    - doc_id: Doc-TV-001
      title: TripTrip技術ビジョンとアーキテクチャ原則
      relationship: 技術原則の定義

    - doc_id: Doc-SA-001
      title: TripTripシステムアーキテクチャ概要
      relationship: ハイレベルアーキテクチャの定義

  next_documents:
    - doc_id: Doc-DA-001
      title: データアーキテクチャ
      relationship: データ層の詳細設計

    - doc_id: Doc-INF-001
      title: インフラストラクチャ設計
      relationship: インフラの詳細設計

    - doc_id: Doc-SEC-001
      title: セキュリティアーキテクチャ
      relationship: セキュリティの詳細設計
```

---

## 文書情報

| 項目 | 内容 |
|------|------|
| Document ID | Doc-SA-002 |
| Title | マイクロサービスアーキテクチャ・設計 |
| Version | 1.0.0 |
| Status | Draft |
| Author | Technical Architecture Agent |
| Created | 2026-01-19 |
| Last Updated | 2026-01-19 |
| Total Lines | ~1,500 |
| Previous Document | Doc-SA-001: TripTripシステムアーキテクチャ概要 |
| Next Document | Doc-DA-001: データアーキテクチャ |

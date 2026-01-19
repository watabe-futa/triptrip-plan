# Doc-DA-001: TripTripデータアーキテクチャ & スキーマ設計

## エグゼクティブサマリー

本文書は、TripTripプラットフォームの包括的なデータアーキテクチャとスキーマ設計を定義します。既存のPostgreSQL/Prisma基盤を活用しながら、グローバルスケールの旅行プラットフォームを支える堅牢なデータモデルを設計します。論理データモデル、物理スキーマ設計、時系列データアーキテクチャ、データ保持ポリシーを包括的に定義し、数百万の同時ユーザー、リアルタイム予約トランザクション、パーソナライズされたAI推奨を実現するデータ基盤を構築します。本設計は、Google、Netflix、Amazonレベルのデータアーキテクチャ品質を目指し、既存システムからの段階的な移行パスを提供します。

---

## 第1章：はじめに & コンテキスト

### 1.1 ビジネス要件 & 制約

#### 1.1.1 データ駆動型ビジネス要件

TripTripプラットフォームは、データを競争優位性の源泉とする戦略を採用します。以下のビジネス要件がデータアーキテクチャ設計を駆動します。

**コア機能要件**:

| カテゴリ | データ要件 | 優先度 | データ特性 |
|---------|-----------|--------|-----------|
| 予約管理 | ホテル・アトラクション・レストラン予約データ | P0 | トランザクショナル、強整合性 |
| Eコマース | 商品カタログ、在庫、注文履歴 | P0 | 読み取り重視、結果整合性許容 |
| ユーザー管理 | プロフィール、認証、設定 | P0 | 個人情報、暗号化必須 |
| 検索・推奨 | 検索インデックス、行動ログ、推奨モデル | P1 | 非正規化、リアルタイム更新 |
| 分析・BI | KPI、トレンド、コホート分析 | P2 | 集計済みデータ、バッチ処理 |

**データボリューム予測**:

```yaml
data_volume_projection:
  year_1:
    users: 100,000
    monthly_active_users: 30,000
    daily_transactions: 5,000
    total_data_size: 50GB

  year_3:
    users: 5,000,000
    monthly_active_users: 1,500,000
    daily_transactions: 500,000
    total_data_size: 5TB

  year_5:
    users: 50,000,000
    monthly_active_users: 15,000,000
    daily_transactions: 5,000,000
    total_data_size: 100TB

growth_drivers:
  - グローバル展開（アジア太平洋、欧州、北米）
  - 新規サービス追加（レストラン、体験、交通）
  - AI/ML機能拡張（行動ログ、モデルデータ）
  - リアルタイム機能（位置情報、通知履歴）
```

#### 1.1.2 技術的制約

**既存システム制約**:

```yaml
current_stack:
  database:
    type: PostgreSQL 16
    orm: Prisma 5.8.0
    connection_pooling: PgBouncer（計画中）

  backend:
    runtime: Node.js 18+
    framework: Hono 4.8.3
    validation: Zod 3.25.67

  frontend_persistence:
    local_db: Hive 2.2.3
    key_value: SharedPreferences

  existing_models:
    - User（認証、プロフィール）
    - Product（商品カタログ）
    - Order/OrderItem（注文管理）
    - Trip/TripItem（旅行計画）
    - Attraction/AttractionTicket/AttractionTimeSlot（観光）
    - Account/Session（認証セッション）
```

**互換性要件**:

1. 既存Prismaスキーマとの後方互換性維持
2. 既存APIエンドポイントのデータ契約維持
3. フロントエンドHiveスキーマとの整合性
4. 段階的なスキーマ移行（ダウンタイム最小化）

### 1.2 技術目標 & 成功基準

#### 1.2.1 パフォーマンス目標

```yaml
performance_targets:
  read_operations:
    simple_query:
      p50: 5ms
      p95: 20ms
      p99: 50ms
    complex_join:
      p50: 20ms
      p95: 100ms
      p99: 200ms
    search_query:
      p50: 50ms
      p95: 200ms
      p99: 500ms

  write_operations:
    single_insert:
      p50: 10ms
      p95: 50ms
      p99: 100ms
    transaction:
      p50: 50ms
      p95: 200ms
      p99: 500ms

  throughput:
    reads_per_second: 100,000
    writes_per_second: 10,000
    concurrent_connections: 10,000
```

#### 1.2.2 データ品質目標

```yaml
data_quality_targets:
  accuracy:
    - 予約データ整合性: 99.999%
    - 在庫データ正確性: 99.99%
    - 価格データ正確性: 100%

  completeness:
    - 必須フィールド入力率: 100%
    - 推奨フィールド入力率: 95%

  timeliness:
    - リアルタイムデータ遅延: < 1秒
    - バッチデータ遅延: < 1時間

  consistency:
    - クロスサービス整合性: 結果整合性（5秒以内）
    - クリティカルデータ: 強整合性
```

### 1.3 主要な前提条件 & トレードオフ

#### 1.3.1 設計原則

```yaml
design_principles:
  1_single_source_of_truth:
    description: 各データには唯一の正式なソースを定義
    implementation:
      - マスターデータ管理（MDM）の導入
      - データオーナーシップの明確化
      - 複製データの派生関係追跡

  2_schema_evolution:
    description: 破壊的変更なしにスキーマを進化
    implementation:
      - 追加的変更のみ（新規カラム、新規テーブル）
      - 非推奨期間の設定（6ヶ月）
      - バージョニングによる移行サポート

  3_read_write_separation:
    description: 読み取りと書き込みのワークロード分離
    implementation:
      - CQRSパターンの適用
      - 読み取りレプリカの活用
      - 非正規化ビューの作成

  4_data_locality:
    description: 関連データの物理的近接配置
    implementation:
      - サービス別データベース
      - リージョン別データ配置
      - ホットデータとコールドデータの分離
```

#### 1.3.2 トレードオフ決定

```yaml
tradeoff_decisions:
  normalization_vs_performance:
    decision: ハイブリッドアプローチ
    rationale:
      - OLTPデータ: 第3正規形（3NF）で整合性確保
      - 読み取り重視データ: 戦略的非正規化でパフォーマンス最適化
      - 分析データ: スタースキーマで分析効率化
    examples:
      - 予約テーブル: 正規化（トランザクション整合性）
      - 商品検索ビュー: 非正規化（検索パフォーマンス）
      - 売上ファクトテーブル: スタースキーマ（分析効率）

  consistency_vs_availability:
    decision: データ種別による使い分け
    rationale:
      - 金銭関連: 強整合性（決済、在庫確保）
      - 表示系: 結果整合性（商品情報、レビュー）
      - 分析系: 最終整合性（集計値、ランキング）
    implementation:
      - Serializable: 決済トランザクション
      - Read Committed: 一般的なCRUD操作
      - Eventual: キャッシュデータ、検索インデックス
```

---

## 第2章：論理データモデル設計

### 2.1 コアエンティティ設計

#### 2.1.1 ユーザードメイン

```yaml
user_domain:
  User:
    description: プラットフォームの登録ユーザー
    attributes:
      id:
        type: UUID
        constraints: [PRIMARY KEY, NOT NULL]
        description: ユーザーの一意識別子
      email:
        type: VARCHAR(255)
        constraints: [UNIQUE, NOT NULL]
        description: メールアドレス（ログインID）
      email_verified:
        type: BOOLEAN
        default: false
        description: メール認証済みフラグ
      phone:
        type: VARCHAR(20)
        constraints: [UNIQUE]
        description: 電話番号（オプション）
      phone_verified:
        type: BOOLEAN
        default: false
        description: 電話番号認証済みフラグ
      password_hash:
        type: VARCHAR(255)
        constraints: [NOT NULL]
        description: Bcryptハッシュ化パスワード
      status:
        type: ENUM
        values: [ACTIVE, SUSPENDED, DELETED]
        default: ACTIVE
        description: アカウントステータス
      created_at:
        type: TIMESTAMPTZ
        default: NOW()
        description: 作成日時
      updated_at:
        type: TIMESTAMPTZ
        default: NOW()
        description: 更新日時
      deleted_at:
        type: TIMESTAMPTZ
        nullable: true
        description: 論理削除日時
    indexes:
      - columns: [email]
        type: UNIQUE
      - columns: [phone]
        type: UNIQUE
        where: "phone IS NOT NULL"
      - columns: [status, created_at]
        type: BTREE

  UserProfile:
    description: ユーザーの詳細プロフィール情報
    attributes:
      user_id:
        type: UUID
        constraints: [PRIMARY KEY, FOREIGN KEY -> User.id]
      display_name:
        type: VARCHAR(100)
        description: 表示名
      first_name:
        type: VARCHAR(50)
        description: 名
      last_name:
        type: VARCHAR(50)
        description: 姓
      first_name_kana:
        type: VARCHAR(50)
        description: 名（カナ）
      last_name_kana:
        type: VARCHAR(50)
        description: 姓（カナ）
      avatar_url:
        type: TEXT
        description: プロフィール画像URL
      birth_date:
        type: DATE
        description: 生年月日
      gender:
        type: ENUM
        values: [MALE, FEMALE, OTHER, PREFER_NOT_TO_SAY]
        description: 性別
      nationality:
        type: VARCHAR(2)
        description: 国籍（ISO 3166-1 alpha-2）
      preferred_language:
        type: VARCHAR(5)
        default: 'ja'
        description: 優先言語（BCP 47）
      preferred_currency:
        type: VARCHAR(3)
        default: 'JPY'
        description: 優先通貨（ISO 4217）
      timezone:
        type: VARCHAR(50)
        default: 'Asia/Tokyo'
        description: タイムゾーン（IANA）
      bio:
        type: TEXT
        description: 自己紹介文
      updated_at:
        type: TIMESTAMPTZ
        default: NOW()
    indexes:
      - columns: [display_name]
        type: GIN
        using: pg_trgm

  UserAddress:
    description: ユーザーの住所情報（複数保持可能）
    attributes:
      id:
        type: UUID
        constraints: [PRIMARY KEY]
      user_id:
        type: UUID
        constraints: [FOREIGN KEY -> User.id, NOT NULL]
      label:
        type: VARCHAR(50)
        description: 住所ラベル（自宅、勤務先等）
      is_default:
        type: BOOLEAN
        default: false
        description: デフォルト住所フラグ
      country:
        type: VARCHAR(2)
        constraints: [NOT NULL]
        description: 国コード（ISO 3166-1 alpha-2）
      postal_code:
        type: VARCHAR(20)
        description: 郵便番号
      state:
        type: VARCHAR(100)
        description: 都道府県/州
      city:
        type: VARCHAR(100)
        description: 市区町村
      address_line1:
        type: VARCHAR(255)
        constraints: [NOT NULL]
        description: 住所1
      address_line2:
        type: VARCHAR(255)
        description: 住所2
      phone:
        type: VARCHAR(20)
        description: 連絡先電話番号
      created_at:
        type: TIMESTAMPTZ
        default: NOW()
      updated_at:
        type: TIMESTAMPTZ
        default: NOW()
    indexes:
      - columns: [user_id, is_default]
        type: BTREE
    constraints:
      - type: UNIQUE
        columns: [user_id, label]

  UserPreference:
    description: ユーザー設定・プリファレンス
    attributes:
      user_id:
        type: UUID
        constraints: [PRIMARY KEY, FOREIGN KEY -> User.id]
      notification_email:
        type: BOOLEAN
        default: true
      notification_push:
        type: BOOLEAN
        default: true
      notification_sms:
        type: BOOLEAN
        default: false
      marketing_email:
        type: BOOLEAN
        default: false
      travel_interests:
        type: JSONB
        default: '[]'
        description: 旅行の興味関心（タグ配列）
      dietary_restrictions:
        type: JSONB
        default: '[]'
        description: 食事制限（アレルギー等）
      accessibility_needs:
        type: JSONB
        default: '[]'
        description: アクセシビリティ要件
      updated_at:
        type: TIMESTAMPTZ
        default: NOW()
```

#### 2.1.2 予約ドメイン

```yaml
booking_domain:
  Booking:
    description: 統合予約エンティティ（ホテル、アトラクション、レストラン等）
    attributes:
      id:
        type: UUID
        constraints: [PRIMARY KEY]
      booking_number:
        type: VARCHAR(20)
        constraints: [UNIQUE, NOT NULL]
        description: 予約番号（人間可読）
      user_id:
        type: UUID
        constraints: [FOREIGN KEY -> User.id, NOT NULL]
      booking_type:
        type: ENUM
        values: [HOTEL, ATTRACTION, RESTAURANT, RENTAL, TOUR]
        constraints: [NOT NULL]
      status:
        type: ENUM
        values: [PENDING, CONFIRMED, CANCELLED, COMPLETED, NO_SHOW, REFUNDED]
        default: PENDING
      booking_date:
        type: DATE
        constraints: [NOT NULL]
        description: 予約対象日
      start_datetime:
        type: TIMESTAMPTZ
        description: 開始日時（チェックイン、入場時刻等）
      end_datetime:
        type: TIMESTAMPTZ
        description: 終了日時（チェックアウト等）
      guest_count:
        type: INTEGER
        default: 1
        description: 利用人数
      special_requests:
        type: TEXT
        description: 特別リクエスト
      internal_notes:
        type: TEXT
        description: 内部メモ（運用者用）
      metadata:
        type: JSONB
        default: '{}'
        description: 予約タイプ固有の追加情報
      subtotal:
        type: DECIMAL(12, 2)
        constraints: [NOT NULL]
        description: 小計（税抜）
      tax_amount:
        type: DECIMAL(12, 2)
        default: 0
        description: 税額
      discount_amount:
        type: DECIMAL(12, 2)
        default: 0
        description: 割引額
      total_amount:
        type: DECIMAL(12, 2)
        constraints: [NOT NULL]
        description: 合計金額
      currency:
        type: VARCHAR(3)
        default: 'JPY'
        description: 通貨コード
      payment_status:
        type: ENUM
        values: [UNPAID, PARTIALLY_PAID, PAID, REFUNDED]
        default: UNPAID
      cancelled_at:
        type: TIMESTAMPTZ
        description: キャンセル日時
      cancellation_reason:
        type: TEXT
        description: キャンセル理由
      created_at:
        type: TIMESTAMPTZ
        default: NOW()
      updated_at:
        type: TIMESTAMPTZ
        default: NOW()
    indexes:
      - columns: [booking_number]
        type: UNIQUE
      - columns: [user_id, status]
        type: BTREE
      - columns: [booking_type, booking_date]
        type: BTREE
      - columns: [status, created_at]
        type: BTREE
      - columns: [metadata]
        type: GIN
    partitioning:
      type: RANGE
      column: created_at
      interval: MONTHLY

  HotelBooking:
    description: ホテル予約詳細
    attributes:
      booking_id:
        type: UUID
        constraints: [PRIMARY KEY, FOREIGN KEY -> Booking.id]
      hotel_id:
        type: UUID
        constraints: [FOREIGN KEY -> Hotel.id, NOT NULL]
      room_type_id:
        type: UUID
        constraints: [FOREIGN KEY -> RoomType.id, NOT NULL]
      check_in_date:
        type: DATE
        constraints: [NOT NULL]
      check_out_date:
        type: DATE
        constraints: [NOT NULL]
      nights:
        type: INTEGER
        constraints: [NOT NULL]
      room_count:
        type: INTEGER
        default: 1
      adult_count:
        type: INTEGER
        default: 1
      child_count:
        type: INTEGER
        default: 0
      meal_plan:
        type: ENUM
        values: [ROOM_ONLY, BREAKFAST, HALF_BOARD, FULL_BOARD]
        default: ROOM_ONLY
      rate_plan_id:
        type: UUID
        description: 適用料金プラン
      room_rate_per_night:
        type: DECIMAL(10, 2)
        constraints: [NOT NULL]
      guest_name:
        type: VARCHAR(100)
        description: 宿泊者代表者名
      estimated_arrival_time:
        type: TIME
        description: 到着予定時刻
    indexes:
      - columns: [hotel_id, check_in_date]
        type: BTREE
      - columns: [room_type_id, check_in_date, check_out_date]
        type: BTREE

  AttractionBooking:
    description: アトラクション・チケット予約詳細
    attributes:
      booking_id:
        type: UUID
        constraints: [PRIMARY KEY, FOREIGN KEY -> Booking.id]
      attraction_id:
        type: UUID
        constraints: [FOREIGN KEY -> Attraction.id, NOT NULL]
      ticket_type_id:
        type: UUID
        constraints: [FOREIGN KEY -> TicketType.id, NOT NULL]
      time_slot_id:
        type: UUID
        constraints: [FOREIGN KEY -> TimeSlot.id]
      visit_date:
        type: DATE
        constraints: [NOT NULL]
      entry_time:
        type: TIME
        description: 入場時刻
      ticket_count:
        type: INTEGER
        default: 1
      adult_tickets:
        type: INTEGER
        default: 1
      child_tickets:
        type: INTEGER
        default: 0
      senior_tickets:
        type: INTEGER
        default: 0
      qr_code:
        type: TEXT
        description: QRコードデータ
      qr_code_used:
        type: BOOLEAN
        default: false
      qr_code_used_at:
        type: TIMESTAMPTZ
    indexes:
      - columns: [attraction_id, visit_date]
        type: BTREE
      - columns: [time_slot_id]
        type: BTREE
      - columns: [qr_code]
        type: HASH
```

#### 2.1.3 商品・注文ドメイン

```yaml
commerce_domain:
  Product:
    description: 商品マスター
    attributes:
      id:
        type: UUID
        constraints: [PRIMARY KEY]
      sku:
        type: VARCHAR(50)
        constraints: [UNIQUE, NOT NULL]
        description: 商品SKU
      name:
        type: VARCHAR(255)
        constraints: [NOT NULL]
      name_en:
        type: VARCHAR(255)
        description: 英語名
      slug:
        type: VARCHAR(255)
        constraints: [UNIQUE, NOT NULL]
        description: URLスラッグ
      description:
        type: TEXT
        description: 商品説明
      description_en:
        type: TEXT
        description: 商品説明（英語）
      category_id:
        type: UUID
        constraints: [FOREIGN KEY -> Category.id]
      brand_id:
        type: UUID
        constraints: [FOREIGN KEY -> Brand.id]
      product_type:
        type: ENUM
        values: [PHYSICAL, DIGITAL, RENTAL, SERVICE]
        default: PHYSICAL
      status:
        type: ENUM
        values: [DRAFT, ACTIVE, INACTIVE, DISCONTINUED]
        default: DRAFT
      base_price:
        type: DECIMAL(10, 2)
        constraints: [NOT NULL]
      currency:
        type: VARCHAR(3)
        default: 'JPY'
      tax_rate:
        type: DECIMAL(5, 4)
        default: 0.10
        description: 消費税率
      weight:
        type: DECIMAL(8, 2)
        description: 重量（グラム）
      dimensions:
        type: JSONB
        description: 寸法（長さ、幅、高さ）
      attributes:
        type: JSONB
        default: '{}'
        description: 商品属性（色、サイズ等）
      tags:
        type: TEXT[]
        default: '{}'
        description: 検索タグ
      images:
        type: JSONB
        default: '[]'
        description: 商品画像URL配列
      seo_title:
        type: VARCHAR(60)
      seo_description:
        type: VARCHAR(160)
      is_featured:
        type: BOOLEAN
        default: false
      sort_order:
        type: INTEGER
        default: 0
      created_at:
        type: TIMESTAMPTZ
        default: NOW()
      updated_at:
        type: TIMESTAMPTZ
        default: NOW()
      published_at:
        type: TIMESTAMPTZ
    indexes:
      - columns: [sku]
        type: UNIQUE
      - columns: [slug]
        type: UNIQUE
      - columns: [category_id, status]
        type: BTREE
      - columns: [status, is_featured, sort_order]
        type: BTREE
      - columns: [tags]
        type: GIN
      - columns: [name, description]
        type: GIN
        using: pg_trgm

  ProductVariant:
    description: 商品バリエーション（サイズ、色等）
    attributes:
      id:
        type: UUID
        constraints: [PRIMARY KEY]
      product_id:
        type: UUID
        constraints: [FOREIGN KEY -> Product.id, NOT NULL]
      sku:
        type: VARCHAR(50)
        constraints: [UNIQUE, NOT NULL]
      name:
        type: VARCHAR(255)
        constraints: [NOT NULL]
      options:
        type: JSONB
        constraints: [NOT NULL]
        description: バリエーションオプション（サイズ:M, 色:赤）
      price_adjustment:
        type: DECIMAL(10, 2)
        default: 0
        description: 基本価格からの調整額
      stock_quantity:
        type: INTEGER
        default: 0
      reserved_quantity:
        type: INTEGER
        default: 0
        description: 仮確保数量
      low_stock_threshold:
        type: INTEGER
        default: 10
      status:
        type: ENUM
        values: [ACTIVE, INACTIVE, OUT_OF_STOCK]
        default: ACTIVE
      barcode:
        type: VARCHAR(50)
      weight:
        type: DECIMAL(8, 2)
      images:
        type: JSONB
        default: '[]'
      created_at:
        type: TIMESTAMPTZ
        default: NOW()
      updated_at:
        type: TIMESTAMPTZ
        default: NOW()
    indexes:
      - columns: [product_id, status]
        type: BTREE
      - columns: [sku]
        type: UNIQUE
      - columns: [options]
        type: GIN

  Order:
    description: 注文ヘッダー
    attributes:
      id:
        type: UUID
        constraints: [PRIMARY KEY]
      order_number:
        type: VARCHAR(20)
        constraints: [UNIQUE, NOT NULL]
        description: 注文番号
      user_id:
        type: UUID
        constraints: [FOREIGN KEY -> User.id, NOT NULL]
      status:
        type: ENUM
        values: [PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED, RETURNED]
        default: PENDING
      order_type:
        type: ENUM
        values: [PURCHASE, RENTAL]
        default: PURCHASE
      subtotal:
        type: DECIMAL(12, 2)
        constraints: [NOT NULL]
      tax_amount:
        type: DECIMAL(12, 2)
        default: 0
      shipping_amount:
        type: DECIMAL(12, 2)
        default: 0
      discount_amount:
        type: DECIMAL(12, 2)
        default: 0
      total_amount:
        type: DECIMAL(12, 2)
        constraints: [NOT NULL]
      currency:
        type: VARCHAR(3)
        default: 'JPY'
      coupon_code:
        type: VARCHAR(50)
      shipping_address_id:
        type: UUID
        constraints: [FOREIGN KEY -> UserAddress.id]
      shipping_address_snapshot:
        type: JSONB
        description: 注文時の住所スナップショット
      billing_address_snapshot:
        type: JSONB
      shipping_method:
        type: VARCHAR(50)
      estimated_delivery_date:
        type: DATE
      actual_delivery_date:
        type: DATE
      tracking_number:
        type: VARCHAR(100)
      notes:
        type: TEXT
      metadata:
        type: JSONB
        default: '{}'
      ordered_at:
        type: TIMESTAMPTZ
        default: NOW()
      shipped_at:
        type: TIMESTAMPTZ
      delivered_at:
        type: TIMESTAMPTZ
      cancelled_at:
        type: TIMESTAMPTZ
      created_at:
        type: TIMESTAMPTZ
        default: NOW()
      updated_at:
        type: TIMESTAMPTZ
        default: NOW()
    indexes:
      - columns: [order_number]
        type: UNIQUE
      - columns: [user_id, status]
        type: BTREE
      - columns: [status, ordered_at]
        type: BTREE
      - columns: [tracking_number]
        type: BTREE
        where: "tracking_number IS NOT NULL"
    partitioning:
      type: RANGE
      column: ordered_at
      interval: MONTHLY

  OrderItem:
    description: 注文明細
    attributes:
      id:
        type: UUID
        constraints: [PRIMARY KEY]
      order_id:
        type: UUID
        constraints: [FOREIGN KEY -> Order.id, NOT NULL]
      product_id:
        type: UUID
        constraints: [FOREIGN KEY -> Product.id, NOT NULL]
      variant_id:
        type: UUID
        constraints: [FOREIGN KEY -> ProductVariant.id]
      product_snapshot:
        type: JSONB
        constraints: [NOT NULL]
        description: 注文時の商品情報スナップショット
      quantity:
        type: INTEGER
        constraints: [NOT NULL, CHECK > 0]
      unit_price:
        type: DECIMAL(10, 2)
        constraints: [NOT NULL]
      subtotal:
        type: DECIMAL(12, 2)
        constraints: [NOT NULL]
      tax_amount:
        type: DECIMAL(12, 2)
        default: 0
      discount_amount:
        type: DECIMAL(12, 2)
        default: 0
      total_amount:
        type: DECIMAL(12, 2)
        constraints: [NOT NULL]
      status:
        type: ENUM
        values: [PENDING, CONFIRMED, SHIPPED, DELIVERED, RETURNED, REFUNDED]
        default: PENDING
      rental_start_date:
        type: DATE
        description: レンタル開始日（レンタル商品の場合）
      rental_end_date:
        type: DATE
        description: レンタル終了日
      created_at:
        type: TIMESTAMPTZ
        default: NOW()
      updated_at:
        type: TIMESTAMPTZ
        default: NOW()
    indexes:
      - columns: [order_id]
        type: BTREE
      - columns: [product_id]
        type: BTREE
```

#### 2.1.4 パートナー・施設ドメイン

```yaml
partner_domain:
  Partner:
    description: ビジネスパートナー（ホテル、アトラクション運営者等）
    attributes:
      id:
        type: UUID
        constraints: [PRIMARY KEY]
      partner_code:
        type: VARCHAR(20)
        constraints: [UNIQUE, NOT NULL]
      name:
        type: VARCHAR(255)
        constraints: [NOT NULL]
      name_en:
        type: VARCHAR(255)
      legal_name:
        type: VARCHAR(255)
        description: 法人名
      partner_type:
        type: ENUM
        values: [HOTEL, ATTRACTION, RESTAURANT, RENTAL, TOUR_OPERATOR, RETAILER]
        constraints: [NOT NULL]
      status:
        type: ENUM
        values: [PENDING, ACTIVE, SUSPENDED, TERMINATED]
        default: PENDING
      contact_email:
        type: VARCHAR(255)
        constraints: [NOT NULL]
      contact_phone:
        type: VARCHAR(20)
      website:
        type: TEXT
      description:
        type: TEXT
      logo_url:
        type: TEXT
      address:
        type: JSONB
      tax_id:
        type: VARCHAR(50)
        description: 税務ID/法人番号
      bank_account:
        type: JSONB
        description: 振込先口座情報（暗号化）
      commission_rate:
        type: DECIMAL(5, 4)
        default: 0.10
        description: 手数料率
      contract_start_date:
        type: DATE
      contract_end_date:
        type: DATE
      metadata:
        type: JSONB
        default: '{}'
      created_at:
        type: TIMESTAMPTZ
        default: NOW()
      updated_at:
        type: TIMESTAMPTZ
        default: NOW()
    indexes:
      - columns: [partner_code]
        type: UNIQUE
      - columns: [partner_type, status]
        type: BTREE

  Hotel:
    description: ホテル・宿泊施設
    attributes:
      id:
        type: UUID
        constraints: [PRIMARY KEY]
      partner_id:
        type: UUID
        constraints: [FOREIGN KEY -> Partner.id, NOT NULL]
      name:
        type: VARCHAR(255)
        constraints: [NOT NULL]
      name_en:
        type: VARCHAR(255)
      slug:
        type: VARCHAR(255)
        constraints: [UNIQUE, NOT NULL]
      description:
        type: TEXT
      description_en:
        type: TEXT
      star_rating:
        type: DECIMAL(2, 1)
        description: 星評価（1.0-5.0）
      hotel_type:
        type: ENUM
        values: [HOTEL, RYOKAN, HOSTEL, GUESTHOUSE, RESORT, APARTMENT]
        default: HOTEL
      status:
        type: ENUM
        values: [DRAFT, ACTIVE, INACTIVE, CLOSED]
        default: DRAFT
      address:
        type: JSONB
        constraints: [NOT NULL]
      location:
        type: GEOGRAPHY(POINT, 4326)
        description: 地理座標（PostGIS）
      timezone:
        type: VARCHAR(50)
        default: 'Asia/Tokyo'
      check_in_time:
        type: TIME
        default: '15:00'
      check_out_time:
        type: TIME
        default: '10:00'
      contact_email:
        type: VARCHAR(255)
      contact_phone:
        type: VARCHAR(20)
      amenities:
        type: JSONB
        default: '[]'
        description: 設備・アメニティ
      policies:
        type: JSONB
        default: '{}'
        description: 宿泊ポリシー
      images:
        type: JSONB
        default: '[]'
      average_rating:
        type: DECIMAL(3, 2)
      review_count:
        type: INTEGER
        default: 0
      min_price:
        type: DECIMAL(10, 2)
        description: 最低価格（キャッシュ）
      seo_title:
        type: VARCHAR(60)
      seo_description:
        type: VARCHAR(160)
      created_at:
        type: TIMESTAMPTZ
        default: NOW()
      updated_at:
        type: TIMESTAMPTZ
        default: NOW()
    indexes:
      - columns: [slug]
        type: UNIQUE
      - columns: [partner_id]
        type: BTREE
      - columns: [status, star_rating]
        type: BTREE
      - columns: [location]
        type: GIST
      - columns: [amenities]
        type: GIN
      - columns: [name, description]
        type: GIN
        using: pg_trgm

  RoomType:
    description: 客室タイプ
    attributes:
      id:
        type: UUID
        constraints: [PRIMARY KEY]
      hotel_id:
        type: UUID
        constraints: [FOREIGN KEY -> Hotel.id, NOT NULL]
      name:
        type: VARCHAR(255)
        constraints: [NOT NULL]
      name_en:
        type: VARCHAR(255)
      description:
        type: TEXT
      room_size:
        type: DECIMAL(6, 2)
        description: 客室面積（平米）
      max_occupancy:
        type: INTEGER
        default: 2
      bed_configuration:
        type: JSONB
        description: ベッド構成
      amenities:
        type: JSONB
        default: '[]'
      images:
        type: JSONB
        default: '[]'
      base_price:
        type: DECIMAL(10, 2)
        constraints: [NOT NULL]
      total_rooms:
        type: INTEGER
        default: 1
        description: 客室数
      status:
        type: ENUM
        values: [ACTIVE, INACTIVE]
        default: ACTIVE
      sort_order:
        type: INTEGER
        default: 0
      created_at:
        type: TIMESTAMPTZ
        default: NOW()
      updated_at:
        type: TIMESTAMPTZ
        default: NOW()
    indexes:
      - columns: [hotel_id, status]
        type: BTREE

  Attraction:
    description: 観光スポット・アトラクション
    attributes:
      id:
        type: UUID
        constraints: [PRIMARY KEY]
      partner_id:
        type: UUID
        constraints: [FOREIGN KEY -> Partner.id, NOT NULL]
      name:
        type: VARCHAR(255)
        constraints: [NOT NULL]
      name_en:
        type: VARCHAR(255)
      slug:
        type: VARCHAR(255)
        constraints: [UNIQUE, NOT NULL]
      description:
        type: TEXT
      description_en:
        type: TEXT
      category:
        type: ENUM
        values: [THEME_PARK, MUSEUM, TEMPLE_SHRINE, NATURE, EXPERIENCE, TOUR, EVENT]
        constraints: [NOT NULL]
      status:
        type: ENUM
        values: [DRAFT, ACTIVE, INACTIVE, CLOSED]
        default: DRAFT
      address:
        type: JSONB
        constraints: [NOT NULL]
      location:
        type: GEOGRAPHY(POINT, 4326)
      timezone:
        type: VARCHAR(50)
        default: 'Asia/Tokyo'
      opening_hours:
        type: JSONB
        description: 営業時間（曜日別）
      contact_email:
        type: VARCHAR(255)
      contact_phone:
        type: VARCHAR(20)
      website:
        type: TEXT
      features:
        type: JSONB
        default: '[]'
      accessibility:
        type: JSONB
        default: '[]'
      images:
        type: JSONB
        default: '[]'
      average_rating:
        type: DECIMAL(3, 2)
      review_count:
        type: INTEGER
        default: 0
      min_price:
        type: DECIMAL(10, 2)
      estimated_duration:
        type: INTERVAL
        description: 平均滞在時間
      created_at:
        type: TIMESTAMPTZ
        default: NOW()
      updated_at:
        type: TIMESTAMPTZ
        default: NOW()
    indexes:
      - columns: [slug]
        type: UNIQUE
      - columns: [partner_id]
        type: BTREE
      - columns: [category, status]
        type: BTREE
      - columns: [location]
        type: GIST
```

### 2.2 エンティティ関係図（ERD）

#### 2.2.1 コアドメイン関係図

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           CORE ENTITY RELATIONSHIPS                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                            USER DOMAIN                                      │ │
│  │                                                                             │ │
│  │   ┌──────────┐      ┌─────────────────┐      ┌──────────────────┐         │ │
│  │   │   User   │──1:1─┤   UserProfile   │      │  UserPreference  │         │ │
│  │   └────┬─────┘      └─────────────────┘      └────────┬─────────┘         │ │
│  │        │                                               │                   │ │
│  │        │ 1:1 ─────────────────────────────────────────┘                   │ │
│  │        │                                                                   │ │
│  │        │ 1:N                                                              │ │
│  │        ▼                                                                   │ │
│  │   ┌──────────────┐                                                        │ │
│  │   │ UserAddress  │                                                        │ │
│  │   └──────────────┘                                                        │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                          │
│                                      │ 1:N                                      │
│                                      ▼                                          │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                          BOOKING DOMAIN                                     │ │
│  │                                                                             │ │
│  │   ┌──────────┐      ┌─────────────────┐      ┌──────────────────┐         │ │
│  │   │ Booking  │──1:1─┤  HotelBooking   │──N:1─┤      Hotel       │         │ │
│  │   │          │      └─────────────────┘      └────────┬─────────┘         │ │
│  │   │          │                                        │ 1:N               │ │
│  │   │          │      ┌─────────────────┐      ┌───────▼──────────┐         │ │
│  │   │          │──1:1─┤AttractionBooking│──N:1─┤    Attraction    │         │ │
│  │   └──────────┘      └─────────────────┘      └──────────────────┘         │ │
│  │        │                                                                   │ │
│  │        │ 1:N                                                              │ │
│  │        ▼                                                                   │ │
│  │   ┌──────────────┐                                                        │ │
│  │   │BookingPayment│                                                        │ │
│  │   └──────────────┘                                                        │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                          │
│                                      │ 1:N                                      │
│                                      ▼                                          │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                          COMMERCE DOMAIN                                    │ │
│  │                                                                             │ │
│  │   ┌──────────┐      ┌─────────────────┐      ┌──────────────────┐         │ │
│  │   │  Order   │──1:N─┤   OrderItem     │──N:1─┤     Product      │         │ │
│  │   └────┬─────┘      └─────────────────┘      └────────┬─────────┘         │ │
│  │        │                                               │ 1:N              │ │
│  │        │ 1:N                                           ▼                  │ │
│  │        ▼                                        ┌──────────────────┐      │ │
│  │   ┌──────────────┐                              │ ProductVariant   │      │ │
│  │   │OrderPayment  │                              └──────────────────┘      │ │
│  │   └──────────────┘                                                        │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                          PARTNER DOMAIN                                     │ │
│  │                                                                             │ │
│  │   ┌──────────┐      ┌─────────────────┐      ┌──────────────────┐         │ │
│  │   │ Partner  │──1:N─┤     Hotel       │──1:N─┤    RoomType      │         │ │
│  │   │          │      └─────────────────┘      └──────────────────┘         │ │
│  │   │          │                                                             │ │
│  │   │          │──1:N─┬─────────────────┐      ┌──────────────────┐         │ │
│  │   │          │      │   Attraction    │──1:N─┤   TicketType     │         │ │
│  │   └──────────┘      └─────────────────┘      └──────────────────┘         │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 2.2.2 カーディナリティと外部キー定義

```yaml
relationships:
  user_domain:
    User_UserProfile:
      type: ONE_TO_ONE
      parent: User
      child: UserProfile
      foreign_key: user_id
      on_delete: CASCADE

    User_UserAddress:
      type: ONE_TO_MANY
      parent: User
      child: UserAddress
      foreign_key: user_id
      on_delete: CASCADE

    User_UserPreference:
      type: ONE_TO_ONE
      parent: User
      child: UserPreference
      foreign_key: user_id
      on_delete: CASCADE

  booking_domain:
    User_Booking:
      type: ONE_TO_MANY
      parent: User
      child: Booking
      foreign_key: user_id
      on_delete: RESTRICT

    Booking_HotelBooking:
      type: ONE_TO_ONE
      parent: Booking
      child: HotelBooking
      foreign_key: booking_id
      on_delete: CASCADE

    Booking_AttractionBooking:
      type: ONE_TO_ONE
      parent: Booking
      child: AttractionBooking
      foreign_key: booking_id
      on_delete: CASCADE

    Hotel_HotelBooking:
      type: ONE_TO_MANY
      parent: Hotel
      child: HotelBooking
      foreign_key: hotel_id
      on_delete: RESTRICT

    Attraction_AttractionBooking:
      type: ONE_TO_MANY
      parent: Attraction
      child: AttractionBooking
      foreign_key: attraction_id
      on_delete: RESTRICT

  commerce_domain:
    User_Order:
      type: ONE_TO_MANY
      parent: User
      child: Order
      foreign_key: user_id
      on_delete: RESTRICT

    Order_OrderItem:
      type: ONE_TO_MANY
      parent: Order
      child: OrderItem
      foreign_key: order_id
      on_delete: CASCADE

    Product_OrderItem:
      type: ONE_TO_MANY
      parent: Product
      child: OrderItem
      foreign_key: product_id
      on_delete: RESTRICT

    Product_ProductVariant:
      type: ONE_TO_MANY
      parent: Product
      child: ProductVariant
      foreign_key: product_id
      on_delete: CASCADE

  partner_domain:
    Partner_Hotel:
      type: ONE_TO_MANY
      parent: Partner
      child: Hotel
      foreign_key: partner_id
      on_delete: RESTRICT

    Partner_Attraction:
      type: ONE_TO_MANY
      parent: Partner
      child: Attraction
      foreign_key: partner_id
      on_delete: RESTRICT

    Hotel_RoomType:
      type: ONE_TO_MANY
      parent: Hotel
      child: RoomType
      foreign_key: hotel_id
      on_delete: CASCADE
```

### 2.3 ドメインモデルとビジネスルール

#### 2.3.1 ユーザードメインルール

```yaml
user_domain_rules:
  registration:
    - rule: メールアドレスはシステム全体で一意
      enforcement: UNIQUE制約 + アプリケーションバリデーション
      error_code: USER_EMAIL_EXISTS

    - rule: パスワードは最低8文字、大小英数字混合
      enforcement: アプリケーションバリデーション
      error_code: USER_PASSWORD_WEAK

    - rule: メール認証は登録後24時間以内に完了
      enforcement: バックグラウンドジョブ
      action: 未認証ユーザーの自動削除

  profile:
    - rule: 表示名は2-100文字
      enforcement: CHECK制約

    - rule: 生年月日は18歳以上のみ（一部サービス）
      enforcement: アプリケーションバリデーション
      scope: 酒類購入、特定アトラクション

  address:
    - rule: ユーザーあたり最大10件の住所
      enforcement: トリガー
      error_code: USER_ADDRESS_LIMIT_EXCEEDED

    - rule: デフォルト住所は1件のみ
      enforcement: トリガー（他のデフォルトを解除）
```

#### 2.3.2 予約ドメインルール

```yaml
booking_domain_rules:
  creation:
    - rule: 予約は利用日の前日まで可能
      enforcement: アプリケーションバリデーション
      exception: 当日予約可能施設

    - rule: 在庫確保はトランザクション内で完了
      enforcement: 分離レベルSERIALIZABLE

    - rule: 予約番号は重複不可
      enforcement: UNIQUE制約 + シーケンス生成
      format: "TT{YYYYMMDD}{6桁連番}"

  modification:
    - rule: 変更は利用日の48時間前まで
      enforcement: アプリケーションバリデーション
      exception: 施設別ポリシー

    - rule: 人数変更は元の予約タイプ内
      enforcement: アプリケーションバリデーション

  cancellation:
    - rule: キャンセルポリシーに基づく返金計算
      enforcement: ビジネスロジック
      policies:
        - 7日前: 100%返金
        - 3日前: 50%返金
        - 前日: 20%返金
        - 当日: 返金なし

    - rule: キャンセル時は在庫を即時解放
      enforcement: トリガー + イベント発行

  payment:
    - rule: 全額決済完了まで予約は「仮確定」
      enforcement: ステータス管理

    - rule: 決済失敗時は30分以内に再試行
      enforcement: キュー処理
      max_retries: 3
```

#### 2.3.3 在庫管理ルール

```yaml
inventory_domain_rules:
  hotel_inventory:
    - rule: 客室在庫は日付別に管理
      model: RoomInventory(room_type_id, date, available, reserved, sold)

    - rule: オーバーブッキング許容率は設定可能
      default: 5%
      enforcement: アプリケーションロジック

    - rule: 在庫確保の有効期限は15分
      enforcement: TTL + バックグラウンド解放

  attraction_inventory:
    - rule: 時間帯別キャパシティ管理
      model: TimeSlotInventory(time_slot_id, capacity, reserved, sold)

    - rule: オーバーブッキング不可
      enforcement: トランザクション + 楽観的ロック

  product_inventory:
    - rule: 在庫数はリアルタイム更新
      enforcement: トリガー

    - rule: マイナス在庫は不可
      enforcement: CHECK制約 (stock >= 0)

    - rule: 低在庫アラートは閾値到達時に発行
      enforcement: トリガー + 通知イベント
```

---

## 第3章：物理スキーマ設計

### 3.1 テーブル設計と正規化レベル

#### 3.1.1 正規化戦略

```yaml
normalization_strategy:
  third_normal_form:
    description: トランザクションデータは3NFを維持
    tables:
      - User, UserProfile, UserAddress
      - Booking, HotelBooking, AttractionBooking
      - Order, OrderItem
      - Partner, Hotel, Attraction
    benefits:
      - データ整合性の保証
      - 更新異常の防止
      - ストレージ効率

  strategic_denormalization:
    description: 読み取りパフォーマンスのための戦略的非正規化
    patterns:
      snapshot_denormalization:
        description: 注文時点の商品情報を保存
        example: OrderItem.product_snapshot
        reason: 履歴データの不変性保証

      cache_denormalization:
        description: 集計値をキャッシュとして保持
        examples:
          - Hotel.average_rating, Hotel.review_count
          - Product.total_sold
          - User.total_orders
        update_strategy: トリガーまたは非同期更新

      search_denormalization:
        description: 検索用の非正規化ビュー
        examples:
          - HotelSearchView（ホテル+客室+価格+レビュー統合）
          - ProductSearchView（商品+バリアント+在庫統合）
        implementation: マテリアライズドビュー
```

#### 3.1.2 PostgreSQL固有の最適化

```sql
-- 拡張機能の有効化
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
CREATE EXTENSION IF NOT EXISTS "postgis";
CREATE EXTENSION IF NOT EXISTS "btree_gin";

-- Enumの定義
CREATE TYPE user_status AS ENUM ('ACTIVE', 'SUSPENDED', 'DELETED');
CREATE TYPE booking_status AS ENUM ('PENDING', 'CONFIRMED', 'CANCELLED', 'COMPLETED', 'NO_SHOW', 'REFUNDED');
CREATE TYPE booking_type AS ENUM ('HOTEL', 'ATTRACTION', 'RESTAURANT', 'RENTAL', 'TOUR');
CREATE TYPE order_status AS ENUM ('PENDING', 'CONFIRMED', 'PROCESSING', 'SHIPPED', 'DELIVERED', 'CANCELLED', 'RETURNED');
CREATE TYPE payment_status AS ENUM ('UNPAID', 'PARTIALLY_PAID', 'PAID', 'REFUNDED');

-- Userテーブル（最適化版）
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email VARCHAR(255) NOT NULL,
    email_verified BOOLEAN DEFAULT FALSE,
    phone VARCHAR(20),
    phone_verified BOOLEAN DEFAULT FALSE,
    password_hash VARCHAR(255) NOT NULL,
    status user_status DEFAULT 'ACTIVE',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    deleted_at TIMESTAMPTZ,

    CONSTRAINT users_email_unique UNIQUE (email),
    CONSTRAINT users_phone_unique UNIQUE (phone) WHERE phone IS NOT NULL
);

-- パーティショニング付きBookingテーブル
CREATE TABLE bookings (
    id UUID NOT NULL DEFAULT uuid_generate_v4(),
    booking_number VARCHAR(20) NOT NULL,
    user_id UUID NOT NULL REFERENCES users(id),
    booking_type booking_type NOT NULL,
    status booking_status DEFAULT 'PENDING',
    booking_date DATE NOT NULL,
    start_datetime TIMESTAMPTZ,
    end_datetime TIMESTAMPTZ,
    guest_count INTEGER DEFAULT 1,
    special_requests TEXT,
    internal_notes TEXT,
    metadata JSONB DEFAULT '{}',
    subtotal DECIMAL(12, 2) NOT NULL,
    tax_amount DECIMAL(12, 2) DEFAULT 0,
    discount_amount DECIMAL(12, 2) DEFAULT 0,
    total_amount DECIMAL(12, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'JPY',
    payment_status payment_status DEFAULT 'UNPAID',
    cancelled_at TIMESTAMPTZ,
    cancellation_reason TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),

    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- 月次パーティション作成（2025-2026年）
CREATE TABLE bookings_2025_01 PARTITION OF bookings
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE bookings_2025_02 PARTITION OF bookings
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
-- ... 以降の月も同様

-- パーティション自動作成関数
CREATE OR REPLACE FUNCTION create_booking_partition()
RETURNS TRIGGER AS $$
DECLARE
    partition_name TEXT;
    start_date DATE;
    end_date DATE;
BEGIN
    partition_name := 'bookings_' || to_char(NEW.created_at, 'YYYY_MM');
    start_date := date_trunc('month', NEW.created_at);
    end_date := start_date + INTERVAL '1 month';

    IF NOT EXISTS (
        SELECT 1 FROM pg_class WHERE relname = partition_name
    ) THEN
        EXECUTE format(
            'CREATE TABLE %I PARTITION OF bookings FOR VALUES FROM (%L) TO (%L)',
            partition_name, start_date, end_date
        );
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### 3.2 インデックス戦略

#### 3.2.1 インデックスタイプ別ガイドライン

```yaml
index_strategy:
  btree:
    use_cases:
      - 等価検索（=）
      - 範囲検索（<, >, BETWEEN）
      - ソート操作（ORDER BY）
      - 外部キー結合
    examples:
      - CREATE INDEX idx_bookings_user_status ON bookings(user_id, status);
      - CREATE INDEX idx_orders_ordered_at ON orders(ordered_at DESC);

  hash:
    use_cases:
      - 等価検索のみ（範囲不可）
      - 高カーディナリティカラム
    examples:
      - CREATE INDEX idx_users_email_hash ON users USING HASH (email);
      - CREATE INDEX idx_bookings_number_hash ON bookings USING HASH (booking_number);

  gin:
    use_cases:
      - JSONB検索
      - 配列検索
      - 全文検索
      - トライグラム検索
    examples:
      - CREATE INDEX idx_products_attributes ON products USING GIN (attributes);
      - CREATE INDEX idx_products_tags ON products USING GIN (tags);
      - CREATE INDEX idx_products_name_trgm ON products USING GIN (name gin_trgm_ops);

  gist:
    use_cases:
      - 地理空間データ
      - 範囲型
      - 全文検索（tsvector）
    examples:
      - CREATE INDEX idx_hotels_location ON hotels USING GIST (location);
      - CREATE INDEX idx_events_date_range ON events USING GIST (daterange(start_date, end_date));

  brin:
    use_cases:
      - 時系列データ
      - 自然順序データ
      - 大規模テーブル
    examples:
      - CREATE INDEX idx_logs_created_at ON activity_logs USING BRIN (created_at);
```

#### 3.2.2 複合インデックス設計

```sql
-- ユーザー関連
CREATE INDEX idx_users_status_created ON users(status, created_at DESC);
CREATE INDEX idx_user_addresses_user_default ON user_addresses(user_id, is_default) WHERE is_default = TRUE;

-- 予約関連
CREATE INDEX idx_bookings_user_status_date ON bookings(user_id, status, booking_date DESC);
CREATE INDEX idx_bookings_type_date_status ON bookings(booking_type, booking_date, status);
CREATE INDEX idx_hotel_bookings_hotel_dates ON hotel_bookings(hotel_id, check_in_date, check_out_date);

-- 商品関連
CREATE INDEX idx_products_category_status_price ON products(category_id, status, base_price);
CREATE INDEX idx_products_status_featured_sort ON products(status, is_featured DESC, sort_order);
CREATE INDEX idx_product_variants_product_status ON product_variants(product_id, status);

-- 注文関連
CREATE INDEX idx_orders_user_status_date ON orders(user_id, status, ordered_at DESC);
CREATE INDEX idx_order_items_product ON order_items(product_id);

-- 部分インデックス（条件付き）
CREATE INDEX idx_bookings_pending ON bookings(created_at) WHERE status = 'PENDING';
CREATE INDEX idx_orders_unshipped ON orders(ordered_at) WHERE status IN ('CONFIRMED', 'PROCESSING');
CREATE INDEX idx_products_active ON products(category_id, base_price) WHERE status = 'ACTIVE';

-- カバリングインデックス（INCLUDE）
CREATE INDEX idx_products_search_covering ON products(category_id, status)
    INCLUDE (name, base_price, images);
```

### 3.3 パーティショニング戦略

#### 3.3.1 パーティショニング対象テーブル

```yaml
partitioning_strategy:
  range_partitioning:
    bookings:
      column: created_at
      interval: MONTHLY
      retention: 7年
      archive_after: 2年

    orders:
      column: ordered_at
      interval: MONTHLY
      retention: 7年
      archive_after: 2年

    activity_logs:
      column: created_at
      interval: DAILY
      retention: 90日
      archive_after: 30日

    analytics_events:
      column: event_timestamp
      interval: DAILY
      retention: 2年
      archive_after: 90日

  list_partitioning:
    bookings_by_type:
      column: booking_type
      values: [HOTEL, ATTRACTION, RESTAURANT, RENTAL, TOUR]
      reason: 予約タイプ別のクエリパターン最適化

  hash_partitioning:
    user_activities:
      column: user_id
      partitions: 16
      reason: ユーザー別データの均等分散
```

#### 3.3.2 パーティション管理自動化

```sql
-- パーティション自動作成・削除関数
CREATE OR REPLACE FUNCTION manage_partitions(
    p_table_name TEXT,
    p_column_name TEXT,
    p_interval TEXT,
    p_retain_months INTEGER
)
RETURNS VOID AS $$
DECLARE
    v_partition_name TEXT;
    v_start_date DATE;
    v_end_date DATE;
    v_drop_date DATE;
BEGIN
    -- 次月のパーティション作成
    v_start_date := date_trunc('month', NOW() + INTERVAL '1 month');
    v_end_date := v_start_date + INTERVAL '1 month';
    v_partition_name := p_table_name || '_' || to_char(v_start_date, 'YYYY_MM');

    IF NOT EXISTS (SELECT 1 FROM pg_class WHERE relname = v_partition_name) THEN
        EXECUTE format(
            'CREATE TABLE %I PARTITION OF %I FOR VALUES FROM (%L) TO (%L)',
            v_partition_name, p_table_name, v_start_date, v_end_date
        );
        RAISE NOTICE 'Created partition: %', v_partition_name;
    END IF;

    -- 古いパーティションの削除
    v_drop_date := date_trunc('month', NOW() - (p_retain_months || ' months')::INTERVAL);
    v_partition_name := p_table_name || '_' || to_char(v_drop_date, 'YYYY_MM');

    IF EXISTS (SELECT 1 FROM pg_class WHERE relname = v_partition_name) THEN
        EXECUTE format('DROP TABLE %I', v_partition_name);
        RAISE NOTICE 'Dropped partition: %', v_partition_name;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- 月次実行スケジュール
CREATE EXTENSION IF NOT EXISTS pg_cron;
SELECT cron.schedule('manage-booking-partitions', '0 0 1 * *',
    $$SELECT manage_partitions('bookings', 'created_at', 'month', 84)$$);
```

---

## 第4章：時系列・ログデータアーキテクチャ

### 4.1 時系列データモデリング

#### 4.1.1 価格履歴

```yaml
price_history:
  description: 商品・サービスの価格変動履歴

  PriceHistory:
    attributes:
      id:
        type: BIGSERIAL
        constraints: [PRIMARY KEY]
      entity_type:
        type: ENUM
        values: [PRODUCT, ROOM_TYPE, TICKET_TYPE]
      entity_id:
        type: UUID
        constraints: [NOT NULL]
      price:
        type: DECIMAL(10, 2)
        constraints: [NOT NULL]
      currency:
        type: VARCHAR(3)
        default: 'JPY'
      effective_from:
        type: TIMESTAMPTZ
        constraints: [NOT NULL]
      effective_to:
        type: TIMESTAMPTZ
      change_reason:
        type: VARCHAR(100)
      changed_by:
        type: UUID
        description: 変更実行ユーザーID
      created_at:
        type: TIMESTAMPTZ
        default: NOW()
    indexes:
      - columns: [entity_type, entity_id, effective_from]
        type: BTREE
      - columns: [effective_from]
        type: BRIN
    partitioning:
      type: RANGE
      column: effective_from
      interval: YEARLY
```

```sql
-- 価格履歴テーブル
CREATE TABLE price_history (
    id BIGSERIAL,
    entity_type VARCHAR(20) NOT NULL,
    entity_id UUID NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'JPY',
    effective_from TIMESTAMPTZ NOT NULL,
    effective_to TIMESTAMPTZ,
    change_reason VARCHAR(100),
    changed_by UUID,
    created_at TIMESTAMPTZ DEFAULT NOW(),

    PRIMARY KEY (id, effective_from)
) PARTITION BY RANGE (effective_from);

-- 現在有効な価格を取得する関数
CREATE OR REPLACE FUNCTION get_current_price(
    p_entity_type VARCHAR(20),
    p_entity_id UUID,
    p_as_of TIMESTAMPTZ DEFAULT NOW()
)
RETURNS DECIMAL(10, 2) AS $$
    SELECT price
    FROM price_history
    WHERE entity_type = p_entity_type
      AND entity_id = p_entity_id
      AND effective_from <= p_as_of
      AND (effective_to IS NULL OR effective_to > p_as_of)
    ORDER BY effective_from DESC
    LIMIT 1;
$$ LANGUAGE SQL STABLE;
```

#### 4.1.2 在庫履歴

```yaml
inventory_history:
  description: 在庫変動の完全な監査証跡

  InventoryTransaction:
    attributes:
      id:
        type: BIGSERIAL
        constraints: [PRIMARY KEY]
      entity_type:
        type: ENUM
        values: [PRODUCT_VARIANT, ROOM_INVENTORY, TIME_SLOT_INVENTORY]
      entity_id:
        type: UUID
        constraints: [NOT NULL]
      transaction_type:
        type: ENUM
        values: [ADJUSTMENT, SALE, RETURN, RESERVATION, RELEASE, TRANSFER]
      quantity_change:
        type: INTEGER
        constraints: [NOT NULL]
        description: 正=増加、負=減少
      quantity_before:
        type: INTEGER
        constraints: [NOT NULL]
      quantity_after:
        type: INTEGER
        constraints: [NOT NULL]
      reference_type:
        type: VARCHAR(50)
        description: 参照エンティティタイプ（Order, Booking等）
      reference_id:
        type: UUID
        description: 参照エンティティID
      notes:
        type: TEXT
      created_by:
        type: UUID
      created_at:
        type: TIMESTAMPTZ
        default: NOW()
    indexes:
      - columns: [entity_type, entity_id, created_at]
        type: BTREE
      - columns: [reference_type, reference_id]
        type: BTREE
      - columns: [created_at]
        type: BRIN
```

### 4.2 アクティビティログアーキテクチャ

#### 4.2.1 ユーザーアクティビティログ

```yaml
activity_logging:
  description: ユーザー行動の包括的なトラッキング

  UserActivity:
    attributes:
      id:
        type: UUID
        constraints: [PRIMARY KEY]
      user_id:
        type: UUID
        description: ログインユーザーID（匿名の場合NULL）
      session_id:
        type: UUID
        constraints: [NOT NULL]
        description: セッション識別子
      device_id:
        type: VARCHAR(100)
        description: デバイス識別子
      activity_type:
        type: ENUM
        values: [PAGE_VIEW, SEARCH, CLICK, ADD_TO_CART, REMOVE_FROM_CART,
                 CHECKOUT_START, CHECKOUT_COMPLETE, BOOKING_START, BOOKING_COMPLETE,
                 LOGIN, LOGOUT, SIGNUP, PROFILE_UPDATE, REVIEW_SUBMIT]
      entity_type:
        type: VARCHAR(50)
        description: 対象エンティティタイプ
      entity_id:
        type: UUID
        description: 対象エンティティID
      page_url:
        type: TEXT
      referrer_url:
        type: TEXT
      search_query:
        type: TEXT
      filters:
        type: JSONB
        description: 適用されたフィルター
      metadata:
        type: JSONB
        default: '{}'
        description: イベント固有のメタデータ
      ip_address:
        type: INET
      user_agent:
        type: TEXT
      country:
        type: VARCHAR(2)
      region:
        type: VARCHAR(100)
      city:
        type: VARCHAR(100)
      platform:
        type: ENUM
        values: [WEB, IOS, ANDROID]
      app_version:
        type: VARCHAR(20)
      created_at:
        type: TIMESTAMPTZ
        default: NOW()
    indexes:
      - columns: [user_id, activity_type, created_at]
        type: BTREE
      - columns: [session_id, created_at]
        type: BTREE
      - columns: [entity_type, entity_id]
        type: BTREE
      - columns: [created_at]
        type: BRIN
    partitioning:
      type: RANGE
      column: created_at
      interval: DAILY
    retention:
      hot: 30日
      warm: 90日
      cold: 2年
```

#### 4.2.2 システム監査ログ

```yaml
audit_logging:
  description: データ変更の完全な監査証跡

  AuditLog:
    attributes:
      id:
        type: BIGSERIAL
        constraints: [PRIMARY KEY]
      table_name:
        type: VARCHAR(100)
        constraints: [NOT NULL]
      record_id:
        type: UUID
        constraints: [NOT NULL]
      action:
        type: ENUM
        values: [INSERT, UPDATE, DELETE]
        constraints: [NOT NULL]
      old_values:
        type: JSONB
        description: 変更前の値
      new_values:
        type: JSONB
        description: 変更後の値
      changed_fields:
        type: TEXT[]
        description: 変更されたフィールド名
      performed_by:
        type: UUID
        description: 操作実行ユーザー
      performed_at:
        type: TIMESTAMPTZ
        default: NOW()
      ip_address:
        type: INET
      user_agent:
        type: TEXT
      request_id:
        type: UUID
        description: リクエスト追跡ID
    indexes:
      - columns: [table_name, record_id, performed_at]
        type: BTREE
      - columns: [performed_by, performed_at]
        type: BTREE
      - columns: [performed_at]
        type: BRIN
```

```sql
-- 汎用監査トリガー関数
CREATE OR REPLACE FUNCTION audit_trigger_func()
RETURNS TRIGGER AS $$
DECLARE
    v_old_data JSONB;
    v_new_data JSONB;
    v_changed_fields TEXT[];
BEGIN
    IF TG_OP = 'UPDATE' THEN
        v_old_data := to_jsonb(OLD);
        v_new_data := to_jsonb(NEW);
        SELECT array_agg(key)
        INTO v_changed_fields
        FROM jsonb_each(v_new_data)
        WHERE v_new_data->key IS DISTINCT FROM v_old_data->key;

        INSERT INTO audit_logs (
            table_name, record_id, action,
            old_values, new_values, changed_fields,
            performed_by, performed_at
        ) VALUES (
            TG_TABLE_NAME, NEW.id, 'UPDATE',
            v_old_data, v_new_data, v_changed_fields,
            current_setting('app.current_user_id', true)::UUID,
            NOW()
        );
        RETURN NEW;

    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_logs (
            table_name, record_id, action,
            old_values, performed_by, performed_at
        ) VALUES (
            TG_TABLE_NAME, OLD.id, 'DELETE',
            to_jsonb(OLD),
            current_setting('app.current_user_id', true)::UUID,
            NOW()
        );
        RETURN OLD;

    ELSIF TG_OP = 'INSERT' THEN
        INSERT INTO audit_logs (
            table_name, record_id, action,
            new_values, performed_by, performed_at
        ) VALUES (
            TG_TABLE_NAME, NEW.id, 'INSERT',
            to_jsonb(NEW),
            current_setting('app.current_user_id', true)::UUID,
            NOW()
        );
        RETURN NEW;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- 主要テーブルへの監査トリガー適用
CREATE TRIGGER audit_users
    AFTER INSERT OR UPDATE OR DELETE ON users
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();

CREATE TRIGGER audit_bookings
    AFTER INSERT OR UPDATE OR DELETE ON bookings
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();

CREATE TRIGGER audit_orders
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();
```

---

## 第5章：データ保持とアーカイブポリシー

### 5.1 データ分類とライフサイクル

#### 5.1.1 データ分類カテゴリ

```yaml
data_classification:
  critical_business:
    description: ビジネス上不可欠なデータ
    examples:
      - 予約データ
      - 注文データ
      - 決済データ
      - パートナー契約データ
    retention: 7年以上
    encryption: 必須
    backup: 継続的レプリケーション + 日次スナップショット

  user_personal:
    description: 個人を特定可能なデータ
    examples:
      - ユーザープロフィール
      - 連絡先情報
      - 住所情報
    retention: アカウント有効期間 + 3年
    encryption: 必須
    anonymization: 削除リクエスト時に匿名化
    backup: 日次スナップショット

  operational:
    description: 運用に必要なデータ
    examples:
      - 在庫データ
      - 価格データ
      - コンテンツデータ
    retention: 無期限（アクティブな間）
    encryption: 推奨
    backup: 日次スナップショット

  analytical:
    description: 分析・レポート用データ
    examples:
      - 集計済みメトリクス
      - ダッシュボードデータ
      - レポートキャッシュ
    retention: 5年
    encryption: 不要
    backup: 週次スナップショット

  logs_events:
    description: ログ・イベントデータ
    examples:
      - ユーザーアクティビティ
      - システムログ
      - アクセスログ
    retention: ティア別（下記参照）
    encryption: 推奨
    backup: 日次アーカイブ
```

#### 5.1.2 ログデータティア管理

```yaml
log_data_tiers:
  hot_tier:
    description: リアルタイムアクセス用
    storage: PostgreSQL / Elasticsearch
    duration: 7-30日
    access_pattern: 高頻度クエリ
    cost_tier: 高

  warm_tier:
    description: 調査・分析用
    storage: TimescaleDB / S3 Standard
    duration: 30-90日
    access_pattern: 中頻度クエリ
    cost_tier: 中

  cold_tier:
    description: コンプライアンス・監査用
    storage: S3 Glacier / BigQuery
    duration: 90日-2年
    access_pattern: 低頻度、バッチアクセス
    cost_tier: 低

  archive_tier:
    description: 長期保存
    storage: S3 Glacier Deep Archive
    duration: 2年以上
    access_pattern: 年次監査時のみ
    cost_tier: 最低
```

### 5.2 データ保持ポリシー実装

#### 5.2.1 自動アーカイブパイプライン

```yaml
archive_pipeline:
  activity_logs:
    hot_to_warm:
      trigger: created_at < NOW() - INTERVAL '30 days'
      action: パーティションをTimescaleDBに移動
      frequency: 日次

    warm_to_cold:
      trigger: created_at < NOW() - INTERVAL '90 days'
      action: Parquet形式でS3 Standardにエクスポート
      frequency: 週次

    cold_to_archive:
      trigger: created_at < NOW() - INTERVAL '2 years'
      action: S3 Glacier Deep Archiveに移動
      frequency: 月次

  audit_logs:
    retention: 7年
    warm_to_cold:
      trigger: performed_at < NOW() - INTERVAL '1 year'
      action: Parquet形式でS3 Standardにエクスポート
      frequency: 月次

  price_history:
    retention: 10年
    warm_to_cold:
      trigger: effective_from < NOW() - INTERVAL '3 years'
      action: Parquet形式でS3 Standardにエクスポート
      frequency: 月次
```

```sql
-- アーカイブ管理テーブル
CREATE TABLE archive_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    source_table VARCHAR(100) NOT NULL,
    partition_name VARCHAR(100),
    destination VARCHAR(50) NOT NULL, -- 's3', 'glacier', etc.
    destination_path TEXT NOT NULL,
    record_count BIGINT,
    file_size_bytes BIGINT,
    status VARCHAR(20) DEFAULT 'PENDING',
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    error_message TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- アーカイブ実行関数
CREATE OR REPLACE FUNCTION archive_partition(
    p_table_name TEXT,
    p_partition_name TEXT,
    p_destination_path TEXT
)
RETURNS UUID AS $$
DECLARE
    v_job_id UUID;
    v_record_count BIGINT;
BEGIN
    -- ジョブ作成
    INSERT INTO archive_jobs (source_table, partition_name, destination, destination_path, status)
    VALUES (p_table_name, p_partition_name, 's3', p_destination_path, 'IN_PROGRESS')
    RETURNING id INTO v_job_id;

    -- レコード数取得
    EXECUTE format('SELECT count(*) FROM %I', p_partition_name)
    INTO v_record_count;

    -- ここでS3へのエクスポート処理を実行（外部ツール連携）
    -- pg_dump や COPY TO PROGRAM でParquet出力

    -- 完了更新
    UPDATE archive_jobs
    SET status = 'COMPLETED',
        record_count = v_record_count,
        completed_at = NOW()
    WHERE id = v_job_id;

    RETURN v_job_id;
EXCEPTION WHEN OTHERS THEN
    UPDATE archive_jobs
    SET status = 'FAILED',
        error_message = SQLERRM,
        completed_at = NOW()
    WHERE id = v_job_id;
    RAISE;
END;
$$ LANGUAGE plpgsql;
```

### 5.3 GDPR/個人情報保護対応

#### 5.3.1 データ削除・匿名化ポリシー

```yaml
privacy_compliance:
  right_to_erasure:
    description: ユーザーからの削除リクエスト対応
    sla: 30日以内
    process:
      1_request_validation:
        - 本人確認
        - 削除対象データの特定

      2_data_anonymization:
        - ユーザー識別情報の匿名化
        - 関連データの匿名化

      3_deletion:
        - 匿名化不可能なデータの物理削除
        - バックアップからの削除（次回ローテーション時）

      4_confirmation:
        - 削除完了通知
        - 監査ログ記録

  data_to_anonymize:
    - email → 'deleted_user_{hash}@anonymized.local'
    - phone → NULL
    - name → 'Deleted User'
    - address → NULL
    - ip_address → '0.0.0.0'

  data_to_retain_anonymized:
    - 注文履歴（集計・分析目的）
    - 予約履歴（集計・分析目的）
    - レビュー（匿名化して保持）
```

```sql
-- ユーザーデータ匿名化関数
CREATE OR REPLACE FUNCTION anonymize_user_data(p_user_id UUID)
RETURNS VOID AS $$
DECLARE
    v_anonymized_email TEXT;
    v_hash TEXT;
BEGIN
    -- ハッシュ生成
    v_hash := encode(sha256(p_user_id::TEXT::BYTEA), 'hex');
    v_anonymized_email := 'deleted_' || substring(v_hash, 1, 8) || '@anonymized.local';

    -- ユーザー基本情報の匿名化
    UPDATE users SET
        email = v_anonymized_email,
        phone = NULL,
        password_hash = 'DELETED',
        status = 'DELETED',
        deleted_at = NOW()
    WHERE id = p_user_id;

    -- プロフィールの匿名化
    UPDATE user_profiles SET
        display_name = 'Deleted User',
        first_name = NULL,
        last_name = NULL,
        first_name_kana = NULL,
        last_name_kana = NULL,
        avatar_url = NULL,
        birth_date = NULL,
        bio = NULL
    WHERE user_id = p_user_id;

    -- 住所の削除
    DELETE FROM user_addresses WHERE user_id = p_user_id;

    -- アクティビティログのIPアドレス匿名化
    UPDATE user_activities SET
        ip_address = '0.0.0.0'::INET,
        user_agent = 'Anonymized'
    WHERE user_id = p_user_id;

    -- 削除リクエストの記録
    INSERT INTO data_deletion_requests (
        user_id, request_type, status, completed_at
    ) VALUES (
        p_user_id, 'FULL_DELETION', 'COMPLETED', NOW()
    );
END;
$$ LANGUAGE plpgsql;
```

---

## 第6章：マイグレーション戦略と実装ロードマップ

### 6.1 既存スキーマからの移行計画

#### 6.1.1 現行Prismaスキーマ分析

```yaml
current_prisma_analysis:
  existing_models:
    User:
      status: 維持・拡張
      changes:
        - profile情報を別テーブルに分離
        - preference情報を追加
        - 監査カラム追加

    Product:
      status: 維持・拡張
      changes:
        - variant機能追加
        - 多言語対応追加
        - SEO情報追加

    Order:
      status: 維持・拡張
      changes:
        - スナップショット機能追加
        - パーティショニング対応

    Attraction:
      status: 維持・拡張
      changes:
        - Partner関連付け追加
        - 在庫管理強化

    Trip:
      status: 維持
      changes:
        - Booking統合検討

  migration_approach:
    strategy: 段階的移行（ビッグバン回避）
    principle: 後方互換性維持
    duration: 6ヶ月
```

#### 6.1.2 マイグレーションフェーズ

```yaml
migration_phases:
  phase_1_foundation:
    duration: 月1-2
    objectives:
      - 拡張機能の有効化（uuid-ossp, pg_trgm, postgis）
      - 監査インフラの構築
      - パーティショニング準備
    migrations:
      - name: 001_enable_extensions
        sql: |
          CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
          CREATE EXTENSION IF NOT EXISTS "pg_trgm";
          CREATE EXTENSION IF NOT EXISTS "postgis";

      - name: 002_create_audit_infrastructure
        sql: |
          CREATE TABLE audit_logs (...);
          CREATE FUNCTION audit_trigger_func() ...;

  phase_2_user_domain:
    duration: 月2-3
    objectives:
      - UserProfileテーブル分離
      - UserPreferenceテーブル追加
      - UserAddressテーブル追加
    migrations:
      - name: 010_create_user_profile
        sql: |
          CREATE TABLE user_profiles (...);
          -- 既存データ移行
          INSERT INTO user_profiles SELECT ... FROM users;

      - name: 011_create_user_preference
        sql: |
          CREATE TABLE user_preferences (...);

      - name: 012_create_user_address
        sql: |
          CREATE TABLE user_addresses (...);

  phase_3_booking_domain:
    duration: 月3-4
    objectives:
      - 統合Bookingテーブル作成
      - HotelBooking/AttractionBooking分離
      - パーティショニング適用
    migrations:
      - name: 020_create_booking_tables
        sql: |
          CREATE TABLE bookings (...) PARTITION BY RANGE (created_at);
          CREATE TABLE hotel_bookings (...);
          CREATE TABLE attraction_bookings (...);

  phase_4_commerce_domain:
    duration: 月4-5
    objectives:
      - ProductVariantテーブル追加
      - 注文スナップショット機能追加
      - パーティショニング適用
    migrations:
      - name: 030_add_product_variants
        sql: |
          CREATE TABLE product_variants (...);

      - name: 031_add_order_snapshots
        sql: |
          ALTER TABLE order_items ADD COLUMN product_snapshot JSONB;

  phase_5_partner_domain:
    duration: 月5-6
    objectives:
      - Partnerテーブル追加
      - Hotel/Attractionとの関連付け
      - インデックス最適化
    migrations:
      - name: 040_create_partner_tables
        sql: |
          CREATE TABLE partners (...);
          ALTER TABLE hotels ADD COLUMN partner_id UUID;
          ALTER TABLE attractions ADD COLUMN partner_id UUID;
```

### 6.2 Prismaスキーマ更新

#### 6.2.1 更新後のPrismaスキーマ例

```prisma
// schema.prisma（更新版）

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ========== User Domain ==========

model User {
  id            String    @id @default(uuid()) @db.Uuid
  email         String    @unique @db.VarChar(255)
  emailVerified Boolean   @default(false) @map("email_verified")
  phone         String?   @unique @db.VarChar(20)
  phoneVerified Boolean   @default(false) @map("phone_verified")
  passwordHash  String    @map("password_hash") @db.VarChar(255)
  status        UserStatus @default(ACTIVE)
  createdAt     DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt     DateTime  @updatedAt @map("updated_at") @db.Timestamptz
  deletedAt     DateTime? @map("deleted_at") @db.Timestamptz

  profile       UserProfile?
  preference    UserPreference?
  addresses     UserAddress[]
  bookings      Booking[]
  orders        Order[]
  reviews       Review[]

  @@map("users")
}

model UserProfile {
  userId           String   @id @map("user_id") @db.Uuid
  displayName      String?  @map("display_name") @db.VarChar(100)
  firstName        String?  @map("first_name") @db.VarChar(50)
  lastName         String?  @map("last_name") @db.VarChar(50)
  firstNameKana    String?  @map("first_name_kana") @db.VarChar(50)
  lastNameKana     String?  @map("last_name_kana") @db.VarChar(50)
  avatarUrl        String?  @map("avatar_url") @db.Text
  birthDate        DateTime? @map("birth_date") @db.Date
  gender           Gender?
  nationality      String?  @db.VarChar(2)
  preferredLanguage String  @default("ja") @map("preferred_language") @db.VarChar(5)
  preferredCurrency String  @default("JPY") @map("preferred_currency") @db.VarChar(3)
  timezone         String   @default("Asia/Tokyo") @db.VarChar(50)
  bio              String?  @db.Text
  updatedAt        DateTime @updatedAt @map("updated_at") @db.Timestamptz

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("user_profiles")
}

model UserAddress {
  id           String   @id @default(uuid()) @db.Uuid
  userId       String   @map("user_id") @db.Uuid
  label        String?  @db.VarChar(50)
  isDefault    Boolean  @default(false) @map("is_default")
  country      String   @db.VarChar(2)
  postalCode   String?  @map("postal_code") @db.VarChar(20)
  state        String?  @db.VarChar(100)
  city         String?  @db.VarChar(100)
  addressLine1 String   @map("address_line1") @db.VarChar(255)
  addressLine2 String?  @map("address_line2") @db.VarChar(255)
  phone        String?  @db.VarChar(20)
  createdAt    DateTime @default(now()) @map("created_at") @db.Timestamptz
  updatedAt    DateTime @updatedAt @map("updated_at") @db.Timestamptz

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([userId, label])
  @@map("user_addresses")
}

// ========== Booking Domain ==========

model Booking {
  id                 String        @id @default(uuid()) @db.Uuid
  bookingNumber      String        @unique @map("booking_number") @db.VarChar(20)
  userId             String        @map("user_id") @db.Uuid
  bookingType        BookingType   @map("booking_type")
  status             BookingStatus @default(PENDING)
  bookingDate        DateTime      @map("booking_date") @db.Date
  startDatetime      DateTime?     @map("start_datetime") @db.Timestamptz
  endDatetime        DateTime?     @map("end_datetime") @db.Timestamptz
  guestCount         Int           @default(1) @map("guest_count")
  specialRequests    String?       @map("special_requests") @db.Text
  internalNotes      String?       @map("internal_notes") @db.Text
  metadata           Json          @default("{}") @db.JsonB
  subtotal           Decimal       @db.Decimal(12, 2)
  taxAmount          Decimal       @default(0) @map("tax_amount") @db.Decimal(12, 2)
  discountAmount     Decimal       @default(0) @map("discount_amount") @db.Decimal(12, 2)
  totalAmount        Decimal       @map("total_amount") @db.Decimal(12, 2)
  currency           String        @default("JPY") @db.VarChar(3)
  paymentStatus      PaymentStatus @default(UNPAID) @map("payment_status")
  cancelledAt        DateTime?     @map("cancelled_at") @db.Timestamptz
  cancellationReason String?       @map("cancellation_reason") @db.Text
  createdAt          DateTime      @default(now()) @map("created_at") @db.Timestamptz
  updatedAt          DateTime      @updatedAt @map("updated_at") @db.Timestamptz

  user              User             @relation(fields: [userId], references: [id])
  hotelBooking      HotelBooking?
  attractionBooking AttractionBooking?
  payments          BookingPayment[]

  @@index([userId, status])
  @@index([bookingType, bookingDate])
  @@map("bookings")
}

// ========== Enums ==========

enum UserStatus {
  ACTIVE
  SUSPENDED
  DELETED
}

enum Gender {
  MALE
  FEMALE
  OTHER
  PREFER_NOT_TO_SAY
}

enum BookingType {
  HOTEL
  ATTRACTION
  RESTAURANT
  RENTAL
  TOUR
}

enum BookingStatus {
  PENDING
  CONFIRMED
  CANCELLED
  COMPLETED
  NO_SHOW
  REFUNDED
}

enum PaymentStatus {
  UNPAID
  PARTIALLY_PAID
  PAID
  REFUNDED
}
```

### 6.3 実装ロードマップ

```yaml
implementation_roadmap:
  quarter_1:
    month_1:
      - 拡張機能有効化
      - 監査インフラ構築
      - 開発環境セットアップ

    month_2:
      - UserProfile/Preference/Address分離
      - ユニットテスト作成
      - 既存データ移行スクリプト作成

    month_3:
      - ステージング環境でのテスト
      - パフォーマンスベンチマーク
      - ロールバックプラン検証

  quarter_2:
    month_4:
      - Bookingドメイン実装
      - パーティショニング適用
      - 予約フロー結合テスト

    month_5:
      - Commerceドメイン拡張
      - ProductVariant実装
      - 在庫管理強化

    month_6:
      - Partnerドメイン実装
      - 全体結合テスト
      - 本番移行準備

  success_metrics:
    - マイグレーション完了率: 100%
    - データ整合性検証: 100%
    - パフォーマンス劣化: 0%
    - ダウンタイム: 1時間以内
```

---

## 第7章：文書間参照 & 依存関係

### 7.1 関連文書マトリックス

```yaml
document_references:
  inputs:
    - document: Doc-TV-001
      title: Technical Vision & Architecture Principles
      relevance: アーキテクチャ原則、技術選定基準

    - document: Doc-SA-001
      title: System Architecture Overview
      relevance: システム全体構成、サービス境界定義

    - document: Doc-SA-002
      title: Microservices Architecture & Design
      relevance: サービス分割、ドメインモデル

    - document: EXISTING_APP_ANALYSIS.md
      title: 既存アプリケーション分析
      relevance: 現行技術スタック、データモデル

  outputs:
    - document: Doc-DA-002
      title: Database Strategy & Technology Selection
      dependency: 本文書のスキーマ設計に基づくDB選定

    - document: Doc-DA-003
      title: Data Pipeline & ETL Architecture
      dependency: 本文書のデータモデルに基づくパイプライン設計

    - document: Doc-SC-001
      title: Security Architecture
      dependency: データ分類、暗号化要件
```

### 7.2 データモデル依存関係図

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DATA ARCHITECTURE DEPENDENCIES                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐                                                        │
│  │  Doc-TV-001     │                                                        │
│  │  技術ビジョン    │                                                        │
│  └────────┬────────┘                                                        │
│           │                                                                  │
│           ▼                                                                  │
│  ┌─────────────────┐      ┌─────────────────┐                              │
│  │  Doc-SA-001     │──────│  Doc-SA-002     │                              │
│  │  システム       │      │  マイクロ       │                              │
│  │  アーキテクチャ │      │  サービス       │                              │
│  └────────┬────────┘      └────────┬────────┘                              │
│           │                        │                                        │
│           └───────────┬────────────┘                                        │
│                       │                                                      │
│                       ▼                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      Doc-DA-001（本文書）                            │   │
│  │                 データアーキテクチャ & スキーマ設計                   │   │
│  └────────────────────────────┬────────────────────────────────────────┘   │
│                               │                                             │
│           ┌───────────────────┼───────────────────┐                        │
│           │                   │                   │                        │
│           ▼                   ▼                   ▼                        │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐              │
│  │  Doc-DA-002     │ │  Doc-DA-003     │ │  Doc-SC-001     │              │
│  │  データベース   │ │  データ         │ │  セキュリティ   │              │
│  │  戦略           │ │  パイプライン   │ │  アーキテクチャ │              │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.3 用語集

```yaml
glossary:
  technical_terms:
    OLTP:
      definition: Online Transaction Processing
      context: トランザクション処理向けデータベース設計

    OLAP:
      definition: Online Analytical Processing
      context: 分析処理向けデータウェアハウス設計

    CQRS:
      definition: Command Query Responsibility Segregation
      context: 読み取りと書き込みの責務分離パターン

    CDC:
      definition: Change Data Capture
      context: データ変更の検出・伝搬メカニズム

    SCD:
      definition: Slowly Changing Dimension
      context: 時間経過に伴う次元データの変化管理

  business_terms:
    Booking:
      definition: 予約全般を表す統合エンティティ
      types: ホテル、アトラクション、レストラン、レンタル、ツアー

    Partner:
      definition: TripTripにサービスを提供する事業者
      types: ホテル運営者、アトラクション運営者、小売業者
```

---

## 付録

### A. テーブル一覧

| ドメイン | テーブル名 | 説明 | パーティション |
|---------|-----------|------|---------------|
| User | users | ユーザー基本情報 | なし |
| User | user_profiles | プロフィール詳細 | なし |
| User | user_addresses | 住所情報 | なし |
| User | user_preferences | 設定・プリファレンス | なし |
| Booking | bookings | 統合予約ヘッダー | 月次（created_at） |
| Booking | hotel_bookings | ホテル予約詳細 | なし |
| Booking | attraction_bookings | アトラクション予約詳細 | なし |
| Commerce | products | 商品マスター | なし |
| Commerce | product_variants | 商品バリエーション | なし |
| Commerce | orders | 注文ヘッダー | 月次（ordered_at） |
| Commerce | order_items | 注文明細 | なし |
| Partner | partners | パートナー企業 | なし |
| Partner | hotels | ホテル施設 | なし |
| Partner | room_types | 客室タイプ | なし |
| Partner | attractions | アトラクション | なし |
| Audit | audit_logs | 監査ログ | 日次（performed_at） |
| Activity | user_activities | ユーザー行動ログ | 日次（created_at） |
| History | price_history | 価格履歴 | 年次（effective_from） |
| History | inventory_transactions | 在庫変動履歴 | 月次（created_at） |

### B. インデックス設計サマリー

```yaml
index_summary:
  total_indexes: 78
  by_type:
    btree: 52
    gin: 15
    gist: 6
    hash: 3
    brin: 2
  by_purpose:
    primary_key: 18
    foreign_key: 24
    search: 12
    sort: 8
    filter: 16
```

### C. 変更履歴

| バージョン | 日付 | 変更内容 | 作成者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-01-19 | 初版作成 | Technical Architecture Agent |

---

**文書情報**
- 文書ID: Doc-DA-001
- バージョン: 1.0.0
- ステータス: ドラフト
- 最終更新: 2026-01-19
- 次回レビュー: 2026-02-19

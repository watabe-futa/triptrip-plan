# Doc-DA-004: TripTripアナリティクス & ビジネスインテリジェンスアーキテクチャ

## エグゼクティブサマリー

本文書は、TripTripプラットフォームのアナリティクスおよびビジネスインテリジェンス（BI）アーキテクチャを包括的に定義します。データウェアハウスのスキーマ設計（スタースキーマ、スノーフレーク）、KPI定義と追跡システム、リアルタイムダッシュボード、顧客セグメンテーション、ファネル分析、予測分析、セルフサービスBIを詳述します。Google、Netflix、Amazonレベルのデータドリブン意思決定を実現する分析基盤を構築し、予約量、収益、リテンション、マーケティングROIの可視化と最適化を可能にします。本設計は、Doc-DA-001〜003で定義されたデータアーキテクチャを基盤とし、経営層からオペレーターまでのすべてのステークホルダーに適切な分析機能を提供します。

---

## 第1章：はじめに & コンテキスト

### 1.1 ビジネス要件 & 制約

#### 1.1.1 分析ビジネス要件

```yaml
analytics_business_requirements:
  strategic_analytics:
    description: 経営層向け戦略的意思決定支援
    stakeholders:
      - CEO/COO
      - 事業部門長
      - 投資家向けレポート
    requirements:
      - 収益・成長トレンド分析
      - 市場シェア・競合分析
      - 事業セグメント別パフォーマンス
      - 財務予測・シナリオ分析
    update_frequency: 日次/週次/月次
    data_freshness: T+1日

  operational_analytics:
    description: オペレーション最適化のための分析
    stakeholders:
      - オペレーションマネージャー
      - カスタマーサポート
      - パートナー管理チーム
    requirements:
      - リアルタイム予約モニタリング
      - 在庫・キャパシティ管理
      - カスタマーサービス品質
      - パートナーパフォーマンス
    update_frequency: リアルタイム〜時間次
    data_freshness: < 10秒（リアルタイム）

  marketing_analytics:
    description: マーケティング効果測定・最適化
    stakeholders:
      - マーケティングチーム
      - グロースチーム
      - プロダクトマネージャー
    requirements:
      - キャンペーン効果測定
      - 顧客獲得コスト（CAC）
      - 顧客生涯価値（LTV）
      - アトリビューション分析
    update_frequency: 日次
    data_freshness: T+1日

  product_analytics:
    description: プロダクト改善のための行動分析
    stakeholders:
      - プロダクトマネージャー
      - UXデザイナー
      - エンジニアリングチーム
    requirements:
      - ファネル分析
      - A/Bテスト結果
      - 機能利用率
      - エラー・パフォーマンス
    update_frequency: リアルタイム〜日次
    data_freshness: < 1分（重要指標）
```

#### 1.1.2 技術的制約

```yaml
technical_constraints:
  data_sources:
    transactional:
      - PostgreSQL（予約、注文、ユーザー）
      - Redis（セッション、リアルタイムカウンター）
    event_streaming:
      - Kafka（ユーザーイベント、システムイベント）
    external:
      - Google Analytics 4
      - Firebase Analytics
      - 広告プラットフォームAPI

  existing_infrastructure:
    data_pipeline: Apache Kafka + Flink + Spark（Doc-DA-003）
    data_warehouse: BigQuery
    data_lake: Google Cloud Storage
    visualization: Looker（検討中）

  performance_requirements:
    dashboard_load_time: < 3秒
    complex_query_time: < 30秒
    concurrent_users: 100（アナリスト）
    data_retention: 7年（法定保存）

  budget_constraints:
    year_1_monthly: ¥500,000
    year_3_monthly: ¥2,000,000
    preference: 従量課金優先
```

### 1.2 技術目標 & 成功基準

#### 1.2.1 分析成熟度目標

```yaml
analytics_maturity_goals:
  phase_1_descriptive:
    timeline: 月1-6
    capabilities:
      - 過去データの可視化
      - 基本KPIダッシュボード
      - 定型レポート自動化
    success_metrics:
      - ダッシュボード利用率 > 80%
      - レポート自動化率 > 70%
      - データ品質スコア > 95%

  phase_2_diagnostic:
    timeline: 月7-12
    capabilities:
      - ドリルダウン分析
      - 原因分析ダッシュボード
      - セグメンテーション分析
    success_metrics:
      - セルフサービス分析率 > 50%
      - 分析リードタイム < 4時間
      - データリテラシースコア向上

  phase_3_predictive:
    timeline: 月13-24
    capabilities:
      - 需要予測
      - チャーン予測
      - LTV予測
    success_metrics:
      - 予測精度 > 80%
      - 予測活用意思決定 > 30%

  phase_4_prescriptive:
    timeline: 月25-36
    capabilities:
      - 価格最適化推奨
      - 次善アクション推奨
      - 自動化された意思決定
    success_metrics:
      - 自動最適化による収益向上 > 5%
      - 意思決定自動化率 > 20%
```

### 1.3 主要な前提条件 & トレードオフ

#### 1.3.1 設計原則

```yaml
design_principles:
  1_single_source_of_truth:
    description: KPI定義の統一と一貫性
    implementation:
      - メトリクスレジストリの構築
      - ビジネス用語辞書の整備
      - 計算ロジックの一元管理

  2_self_service_first:
    description: 非技術者でも分析可能な環境
    implementation:
      - セマンティックレイヤーの構築
      - ドラッグ＆ドロップBI
      - テンプレート化されたダッシュボード

  3_governed_flexibility:
    description: ガバナンスと柔軟性の両立
    implementation:
      - 認定データセットの提供
      - アクセス制御とデータマスキング
      - 変更管理プロセス

  4_real_time_where_needed:
    description: 必要な箇所のみリアルタイム化
    implementation:
      - ユースケース別の鮮度要件定義
      - コスト対効果の評価
      - ハイブリッドアーキテクチャ
```

---

## 第2章：データウェアハウス設計

### 2.1 スキーマ設計（ファクト/ディメンション）

#### 2.1.1 スタースキーマ概要

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         STAR SCHEMA - BOOKING MART                               │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│                                 ┌─────────────┐                                  │
│                                 │  dim_date   │                                  │
│                                 └──────┬──────┘                                  │
│                                        │                                         │
│       ┌─────────────┐                  │                  ┌─────────────┐       │
│       │  dim_user   │                  │                  │ dim_entity  │       │
│       └──────┬──────┘                  │                  └──────┬──────┘       │
│              │                         │                         │              │
│              │    ┌────────────────────┼────────────────────┐    │              │
│              │    │                    │                    │    │              │
│              └────┤                    ▼                    ├────┘              │
│                   │          ┌─────────────────┐            │                   │
│                   │          │                 │            │                   │
│                   ├─────────▶│  fact_bookings  │◀───────────┤                   │
│                   │          │                 │            │                   │
│                   │          └─────────────────┘            │                   │
│              ┌────┤                    ▲                    ├────┐              │
│              │    │                    │                    │    │              │
│              │    └────────────────────┼────────────────────┘    │              │
│              │                         │                         │              │
│       ┌──────┴──────┐                  │                  ┌──────┴──────┐       │
│       │dim_channel  │                  │                  │ dim_partner │       │
│       └─────────────┘                  │                  └─────────────┘       │
│                                        │                                         │
│                                 ┌──────┴──────┐                                  │
│                                 │dim_geography│                                  │
│                                 └─────────────┘                                  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 2.1.2 ファクトテーブル詳細設計

```sql
-- fact_bookings: 予約ファクトテーブル
CREATE TABLE dwh.fact_bookings (
    -- サロゲートキー
    booking_key BIGINT NOT NULL,

    -- ディメンションキー
    booking_date_key INT NOT NULL,           -- dim_date参照
    booking_time_key INT NOT NULL,           -- dim_time参照
    user_key BIGINT NOT NULL,                -- dim_user参照
    entity_key BIGINT NOT NULL,              -- dim_entity参照
    partner_key BIGINT NOT NULL,             -- dim_partner参照
    channel_key INT NOT NULL,                -- dim_channel参照
    geo_key INT NOT NULL,                    -- dim_geography参照
    booking_type_key INT NOT NULL,           -- dim_booking_type参照

    -- デジェネレートディメンション
    booking_id STRING NOT NULL,              -- 自然キー
    booking_number STRING NOT NULL,
    booking_status STRING NOT NULL,
    payment_status STRING NOT NULL,

    -- メジャー（加法的）
    booking_count INT64 DEFAULT 1,
    guest_count INT64,
    nights INT64,                            -- ホテル予約の場合
    ticket_count INT64,                      -- チケット予約の場合

    -- メジャー（金額系）
    subtotal_amount NUMERIC(12,2),
    tax_amount NUMERIC(12,2),
    discount_amount NUMERIC(12,2),
    total_amount NUMERIC(12,2),
    commission_amount NUMERIC(12,2),
    net_revenue_amount NUMERIC(12,2),

    -- メジャー（通貨変換後）
    total_amount_jpy NUMERIC(12,2),
    commission_amount_jpy NUMERIC(12,2),
    net_revenue_jpy NUMERIC(12,2),

    -- 非加法的メジャー
    currency STRING,
    exchange_rate NUMERIC(10,6),

    -- メタデータ
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    etl_batch_id STRING,

    PRIMARY KEY (booking_key) NOT ENFORCED
)
PARTITION BY DATE_TRUNC(booking_date_key, MONTH)
CLUSTER BY user_key, entity_key, booking_type_key
OPTIONS (
    description = '予約ファクトテーブル - 1予約1行の粒度',
    labels = [('domain', 'booking'), ('layer', 'dwh')]
);

-- fact_orders: 注文ファクトテーブル
CREATE TABLE dwh.fact_orders (
    -- サロゲートキー
    order_item_key BIGINT NOT NULL,

    -- ディメンションキー
    order_date_key INT NOT NULL,
    order_time_key INT NOT NULL,
    user_key BIGINT NOT NULL,
    product_key BIGINT NOT NULL,
    category_key INT NOT NULL,
    brand_key INT NOT NULL,
    channel_key INT NOT NULL,
    geo_key INT NOT NULL,

    -- デジェネレートディメンション
    order_id STRING NOT NULL,
    order_number STRING NOT NULL,
    order_item_id STRING NOT NULL,
    order_status STRING NOT NULL,
    order_type STRING NOT NULL,

    -- メジャー
    order_count INT64 DEFAULT 1,
    item_count INT64 DEFAULT 1,
    quantity INT64,

    -- 金額メジャー
    unit_price NUMERIC(10,2),
    line_subtotal NUMERIC(12,2),
    line_tax NUMERIC(12,2),
    line_discount NUMERIC(12,2),
    line_total NUMERIC(12,2),
    shipping_amount NUMERIC(12,2),

    -- 通貨変換後
    line_total_jpy NUMERIC(12,2),

    -- レンタル固有
    rental_days INT64,
    rental_revenue NUMERIC(12,2),

    -- メタデータ
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,

    PRIMARY KEY (order_item_key) NOT ENFORCED
)
PARTITION BY DATE_TRUNC(order_date_key, MONTH)
CLUSTER BY user_key, product_key;

-- fact_user_activity: ユーザーアクティビティファクト
CREATE TABLE dwh.fact_user_activity (
    -- サロゲートキー
    activity_key BIGINT NOT NULL,

    -- ディメンションキー
    activity_date_key INT NOT NULL,
    activity_time_key INT NOT NULL,
    user_key BIGINT,                         -- 匿名ユーザーはNULL
    session_key BIGINT NOT NULL,
    page_key INT NOT NULL,
    device_key INT NOT NULL,
    geo_key INT NOT NULL,
    campaign_key INT,

    -- デジェネレートディメンション
    event_id STRING NOT NULL,
    event_name STRING NOT NULL,
    session_id STRING NOT NULL,

    -- メジャー
    event_count INT64 DEFAULT 1,
    page_view_count INT64,
    unique_page_count INT64,

    -- 時間メジャー
    time_on_page_seconds INT64,
    session_duration_seconds INT64,

    -- エンゲージメントメジャー
    scroll_depth_percent INT64,
    click_count INT64,
    form_submit_count INT64,

    -- コンバージョン
    is_conversion BOOL DEFAULT FALSE,
    conversion_value NUMERIC(12,2),

    -- メタデータ
    event_timestamp TIMESTAMP NOT NULL,

    PRIMARY KEY (activity_key) NOT ENFORCED
)
PARTITION BY DATE_TRUNC(activity_date_key, DAY)
CLUSTER BY user_key, event_name;

-- fact_daily_inventory: 日次在庫スナップショット
CREATE TABLE dwh.fact_daily_inventory (
    -- キー
    snapshot_date_key INT NOT NULL,
    entity_key BIGINT NOT NULL,

    -- ディメンション
    entity_type STRING NOT NULL,

    -- 在庫メジャー
    total_inventory INT64,
    available_inventory INT64,
    reserved_inventory INT64,
    sold_inventory INT64,
    blocked_inventory INT64,

    -- 稼働率
    occupancy_rate NUMERIC(5,4),
    booking_pace INT64,                      -- 前日比予約数

    -- 価格メジャー
    base_price NUMERIC(10,2),
    current_price NUMERIC(10,2),
    min_price NUMERIC(10,2),
    max_price NUMERIC(10,2),
    avg_sold_price NUMERIC(10,2),

    -- メタデータ
    snapshot_timestamp TIMESTAMP NOT NULL,

    PRIMARY KEY (snapshot_date_key, entity_key) NOT ENFORCED
)
PARTITION BY DATE_TRUNC(snapshot_date_key, MONTH)
CLUSTER BY entity_type, entity_key;
```

#### 2.1.3 ディメンションテーブル設計

```sql
-- dim_date: 日付ディメンション（Type 0）
CREATE TABLE dwh.dim_date (
    date_key INT NOT NULL,                   -- YYYYMMDD形式
    full_date DATE NOT NULL,

    -- カレンダー階層
    year INT NOT NULL,
    quarter INT NOT NULL,
    quarter_name STRING,                     -- Q1, Q2, Q3, Q4
    month INT NOT NULL,
    month_name STRING,                       -- January, February...
    month_name_ja STRING,                    -- 1月, 2月...
    week_of_year INT NOT NULL,
    week_of_month INT NOT NULL,
    day_of_year INT NOT NULL,
    day_of_month INT NOT NULL,
    day_of_week INT NOT NULL,                -- 1=Monday, 7=Sunday
    day_name STRING,                         -- Monday, Tuesday...
    day_name_ja STRING,                      -- 月曜日, 火曜日...

    -- 会計年度
    fiscal_year INT,
    fiscal_quarter INT,
    fiscal_month INT,

    -- フラグ
    is_weekend BOOL,
    is_holiday BOOL,
    is_business_day BOOL,
    holiday_name STRING,
    holiday_name_ja STRING,

    -- 季節・イベント
    season STRING,                           -- Spring, Summer, Fall, Winter
    season_ja STRING,                        -- 春, 夏, 秋, 冬
    is_golden_week BOOL,
    is_obon BOOL,
    is_year_end BOOL,

    -- 比較用
    same_day_last_year DATE,
    same_day_last_month DATE,

    PRIMARY KEY (date_key) NOT ENFORCED
);

-- dim_time: 時刻ディメンション（Type 0）
CREATE TABLE dwh.dim_time (
    time_key INT NOT NULL,                   -- HHMM形式
    full_time TIME NOT NULL,

    hour_24 INT NOT NULL,
    hour_12 INT NOT NULL,
    minute INT NOT NULL,
    am_pm STRING,                            -- AM, PM

    -- 時間帯区分
    time_band STRING,                        -- Early Morning, Morning, Afternoon, Evening, Night
    time_band_ja STRING,                     -- 早朝, 午前, 午後, 夕方, 夜間
    is_business_hours BOOL,                  -- 9:00-18:00
    is_peak_hours BOOL,                      -- 12:00-14:00, 18:00-20:00

    PRIMARY KEY (time_key) NOT ENFORCED
);

-- dim_user: ユーザーディメンション（Type 2 SCD）
CREATE TABLE dwh.dim_user (
    user_key BIGINT NOT NULL,                -- サロゲートキー
    user_id STRING NOT NULL,                 -- 自然キー

    -- ユーザー属性
    email_domain STRING,
    registration_date DATE,
    registration_channel STRING,

    -- デモグラフィック
    age_group STRING,                        -- 18-24, 25-34, 35-44, 45-54, 55+
    gender STRING,
    nationality STRING,

    -- セグメンテーション
    user_tier STRING,                        -- Standard, Premium, VIP
    lifetime_value_segment STRING,           -- Low, Medium, High, Very High
    rfm_segment STRING,                      -- Champions, Loyal, At Risk, etc.

    -- プリファレンス
    preferred_language STRING,
    preferred_currency STRING,
    preferred_booking_type STRING,

    -- 地理
    country STRING,
    region STRING,
    city STRING,

    -- SCD2 管理カラム
    effective_from TIMESTAMP NOT NULL,
    effective_to TIMESTAMP,
    is_current BOOL NOT NULL DEFAULT TRUE,

    -- メタデータ
    created_at TIMESTAMP,
    updated_at TIMESTAMP,

    PRIMARY KEY (user_key) NOT ENFORCED
);

-- dim_entity: 予約対象エンティティディメンション（Type 2 SCD）
CREATE TABLE dwh.dim_entity (
    entity_key BIGINT NOT NULL,
    entity_id STRING NOT NULL,

    -- エンティティ分類
    entity_type STRING NOT NULL,             -- HOTEL, ATTRACTION, RESTAURANT
    entity_subtype STRING,                   -- HOTEL: RYOKAN, HOTEL, HOSTEL

    -- 基本情報
    name STRING NOT NULL,
    name_en STRING,
    slug STRING,

    -- パートナー
    partner_id STRING,
    partner_name STRING,

    -- 評価
    star_rating NUMERIC(2,1),
    average_rating NUMERIC(3,2),
    review_count INT64,
    rating_category STRING,                  -- Excellent, Very Good, Good, Average, Poor

    -- カテゴリ
    category STRING,
    subcategory STRING,

    -- 地理
    country STRING,
    region STRING,
    prefecture STRING,
    city STRING,
    area STRING,
    latitude NUMERIC(10,7),
    longitude NUMERIC(10,7),

    -- 価格帯
    price_tier STRING,                       -- Budget, Mid-Range, Luxury
    base_price NUMERIC(10,2),

    -- SCD2
    effective_from TIMESTAMP NOT NULL,
    effective_to TIMESTAMP,
    is_current BOOL NOT NULL DEFAULT TRUE,

    PRIMARY KEY (entity_key) NOT ENFORCED
);

-- dim_channel: チャネルディメンション（Type 1）
CREATE TABLE dwh.dim_channel (
    channel_key INT NOT NULL,

    channel_id STRING NOT NULL,
    channel_name STRING NOT NULL,

    -- チャネル階層
    channel_group STRING,                    -- Direct, Organic, Paid, Referral
    channel_category STRING,                 -- Web, App, Partner

    -- プラットフォーム
    platform STRING,                         -- iOS, Android, Web, API
    device_type STRING,                      -- Mobile, Desktop, Tablet

    -- トラッキング
    utm_source STRING,
    utm_medium STRING,
    utm_campaign STRING,

    PRIMARY KEY (channel_key) NOT ENFORCED
);

-- dim_geography: 地理ディメンション（Type 0）
CREATE TABLE dwh.dim_geography (
    geo_key INT NOT NULL,

    -- 地理階層
    continent STRING,
    country_code STRING,
    country_name STRING,
    country_name_ja STRING,
    region_code STRING,
    region_name STRING,
    region_name_ja STRING,
    prefecture_code STRING,
    prefecture_name STRING,
    city_code STRING,
    city_name STRING,

    -- 追加属性
    timezone STRING,
    currency STRING,
    language STRING,

    -- 人口統計
    population INT64,
    gdp_per_capita NUMERIC(12,2),

    PRIMARY KEY (geo_key) NOT ENFORCED
);

-- dim_partner: パートナーディメンション（Type 2 SCD）
CREATE TABLE dwh.dim_partner (
    partner_key BIGINT NOT NULL,
    partner_id STRING NOT NULL,

    partner_code STRING,
    partner_name STRING NOT NULL,
    partner_type STRING,                     -- HOTEL, ATTRACTION, RESTAURANT, etc.

    -- ステータス
    status STRING,
    contract_status STRING,

    -- ビジネス条件
    commission_rate NUMERIC(5,4),
    payment_terms STRING,

    -- パフォーマンスティア
    performance_tier STRING,                 -- Bronze, Silver, Gold, Platinum

    -- SCD2
    effective_from TIMESTAMP NOT NULL,
    effective_to TIMESTAMP,
    is_current BOOL NOT NULL DEFAULT TRUE,

    PRIMARY KEY (partner_key) NOT ENFORCED
);
```

### 2.2 テーブル構造と関係性

#### 2.2.1 データマート構成

```yaml
data_marts:
  booking_mart:
    description: 予約分析用データマート
    fact_tables:
      - fact_bookings
      - fact_daily_inventory
    dimension_tables:
      - dim_date
      - dim_time
      - dim_user
      - dim_entity
      - dim_partner
      - dim_channel
      - dim_geography
      - dim_booking_type
    primary_metrics:
      - total_bookings
      - gross_booking_value
      - average_booking_value
      - cancellation_rate
      - occupancy_rate

  commerce_mart:
    description: Eコマース分析用データマート
    fact_tables:
      - fact_orders
      - fact_cart_abandonment
    dimension_tables:
      - dim_date
      - dim_time
      - dim_user
      - dim_product
      - dim_category
      - dim_brand
      - dim_channel
      - dim_geography
    primary_metrics:
      - total_orders
      - gross_merchandise_value
      - average_order_value
      - conversion_rate
      - cart_abandonment_rate

  user_mart:
    description: ユーザー分析用データマート
    fact_tables:
      - fact_user_activity
      - fact_user_engagement
    dimension_tables:
      - dim_date
      - dim_user
      - dim_page
      - dim_device
      - dim_channel
    primary_metrics:
      - daily_active_users
      - monthly_active_users
      - session_duration
      - pages_per_session
      - bounce_rate

  marketing_mart:
    description: マーケティング分析用データマート
    fact_tables:
      - fact_campaign_performance
      - fact_attribution
    dimension_tables:
      - dim_date
      - dim_campaign
      - dim_channel
      - dim_geography
    primary_metrics:
      - impressions
      - clicks
      - conversions
      - cost_per_acquisition
      - return_on_ad_spend
```

### 2.3 履歴データ管理（SCD）

#### 2.3.1 SCD実装パターン

```yaml
scd_implementation:
  type_0_fixed:
    description: 変更なし（参照データ）
    tables:
      - dim_date
      - dim_time
      - dim_geography
    rationale: 静的なマスターデータ、変更が発生しない

  type_1_overwrite:
    description: 上書き（履歴不要）
    tables:
      - dim_channel
      - dim_device
    rationale: 分析上、履歴が不要な属性
    columns:
      - utm_source
      - utm_medium

  type_2_history:
    description: 履歴保持（変更追跡必要）
    tables:
      - dim_user
      - dim_entity
      - dim_partner
    rationale: 分析上、変更履歴が重要
    tracked_columns:
      dim_user:
        - user_tier
        - lifetime_value_segment
        - country
        - preferred_language
      dim_entity:
        - status
        - star_rating
        - average_rating
        - price_tier
      dim_partner:
        - status
        - commission_rate
        - performance_tier
```

#### 2.3.2 SCD2 マージ処理

```sql
-- BigQueryでのSCD Type 2 マージ処理
DECLARE processing_date DATE DEFAULT CURRENT_DATE();

-- ステップ1: 変更があったレコードを期限切れにする
UPDATE dwh.dim_user AS target
SET
    effective_to = CURRENT_TIMESTAMP(),
    is_current = FALSE
WHERE target.is_current = TRUE
  AND EXISTS (
    SELECT 1
    FROM staging.stg_users AS source
    WHERE source.user_id = target.user_id
      AND source.processing_date = processing_date
      AND (
        target.user_tier != source.user_tier OR
        target.preferred_language != source.preferred_language OR
        target.country != source.country
      )
  );

-- ステップ2: 新規・変更レコードを挿入
INSERT INTO dwh.dim_user (
    user_key,
    user_id,
    email_domain,
    registration_date,
    registration_channel,
    age_group,
    gender,
    nationality,
    user_tier,
    lifetime_value_segment,
    rfm_segment,
    preferred_language,
    preferred_currency,
    preferred_booking_type,
    country,
    region,
    city,
    effective_from,
    effective_to,
    is_current,
    created_at,
    updated_at
)
SELECT
    GENERATE_UUID() AS user_key,
    s.user_id,
    s.email_domain,
    s.registration_date,
    s.registration_channel,
    s.age_group,
    s.gender,
    s.nationality,
    s.user_tier,
    s.lifetime_value_segment,
    s.rfm_segment,
    s.preferred_language,
    s.preferred_currency,
    s.preferred_booking_type,
    s.country,
    s.region,
    s.city,
    CURRENT_TIMESTAMP() AS effective_from,
    TIMESTAMP('9999-12-31 23:59:59') AS effective_to,
    TRUE AS is_current,
    CURRENT_TIMESTAMP() AS created_at,
    CURRENT_TIMESTAMP() AS updated_at
FROM staging.stg_users s
LEFT JOIN dwh.dim_user d
  ON s.user_id = d.user_id AND d.is_current = TRUE
WHERE s.processing_date = processing_date
  AND (
    d.user_id IS NULL  -- 新規ユーザー
    OR (
      d.user_tier != s.user_tier OR
      d.preferred_language != s.preferred_language OR
      d.country != s.country
    )  -- 変更ユーザー
  );
```

---

## 第3章：KPI & メトリクス設計

### 3.1 ビジネスKPI定義

#### 3.1.1 収益系KPI

```yaml
revenue_kpis:
  gross_booking_value:
    name: 総予約金額（GBV）
    definition: 期間内の全予約の総額（税込、キャンセル前）
    formula: SUM(fact_bookings.total_amount) WHERE booking_status != 'CANCELLED'
    unit: JPY
    aggregation: 加法的
    dimensions:
      - booking_type
      - channel
      - geography
      - partner
    targets:
      year_1: ¥500M
      year_3: ¥10B
      year_5: ¥100B

  net_revenue:
    name: 純収益
    definition: GBVからキャンセル・返金を除いた実収益
    formula: SUM(fact_bookings.total_amount) WHERE booking_status = 'COMPLETED'
    unit: JPY
    aggregation: 加法的
    dimensions:
      - booking_type
      - channel
      - partner

  commission_revenue:
    name: コミッション収益
    definition: パートナーから得る手数料収入
    formula: SUM(fact_bookings.commission_amount)
    unit: JPY
    aggregation: 加法的

  average_booking_value:
    name: 平均予約単価（ABV）
    definition: 1予約あたりの平均金額
    formula: SUM(total_amount) / COUNT(booking_id)
    unit: JPY
    aggregation: 非加法的（平均）
    targets:
      hotel: ¥30,000
      attraction: ¥5,000
      rental: ¥8,000

  revenue_per_user:
    name: ユーザーあたり収益（RPU）
    definition: アクティブユーザー1人あたりの収益
    formula: SUM(total_amount) / COUNT(DISTINCT user_id)
    unit: JPY
    aggregation: 非加法的

  gross_merchandise_value:
    name: 総商品取引高（GMV）
    definition: Eコマース注文の総額
    formula: SUM(fact_orders.line_total)
    unit: JPY
    aggregation: 加法的
```

#### 3.1.2 成長系KPI

```yaml
growth_kpis:
  total_bookings:
    name: 総予約件数
    definition: 期間内の予約総数
    formula: COUNT(DISTINCT booking_id)
    unit: 件
    aggregation: 加法的
    targets:
      year_1_daily: 500
      year_3_daily: 50,000
      year_5_daily: 500,000

  booking_growth_rate:
    name: 予約成長率
    definition: 前期比の予約件数成長率
    formula: (当期予約数 - 前期予約数) / 前期予約数 * 100
    unit: '%'
    aggregation: 非加法的
    targets:
      mom: 10%
      yoy: 100%

  new_user_bookings:
    name: 新規ユーザー予約数
    definition: 初回予約ユーザーの予約数
    formula: COUNT(booking_id) WHERE is_first_booking = TRUE
    unit: 件
    aggregation: 加法的

  repeat_booking_rate:
    name: リピート予約率
    definition: 2回目以降の予約の割合
    formula: COUNT(booking_id WHERE is_repeat) / COUNT(booking_id) * 100
    unit: '%'
    aggregation: 非加法的
    targets:
      year_1: 20%
      year_3: 40%
      year_5: 50%

  user_acquisition:
    name: 新規ユーザー獲得数
    definition: 新規登録ユーザー数
    formula: COUNT(DISTINCT user_id) WHERE is_new_user = TRUE
    unit: 人
    aggregation: 加法的
```

### 3.2 製品メトリクス

#### 3.2.1 エンゲージメントメトリクス

```yaml
engagement_metrics:
  daily_active_users:
    name: DAU（日次アクティブユーザー）
    definition: 1日に1回以上アクションを行ったユニークユーザー数
    formula: COUNT(DISTINCT user_id) WHERE activity_date = target_date
    unit: 人
    aggregation: 非加法的（日次）

  monthly_active_users:
    name: MAU（月次アクティブユーザー）
    definition: 1ヶ月に1回以上アクションを行ったユニークユーザー数
    formula: COUNT(DISTINCT user_id) WHERE activity_date BETWEEN month_start AND month_end
    unit: 人
    aggregation: 非加法的（月次）

  dau_mau_ratio:
    name: DAU/MAU比率（スティッキネス）
    definition: ユーザーの定着度を示す指標
    formula: AVG(DAU) / MAU * 100
    unit: '%'
    aggregation: 非加法的
    targets:
      good: 20%
      excellent: 30%

  sessions_per_user:
    name: ユーザーあたりセッション数
    definition: アクティブユーザー1人あたりの平均セッション数
    formula: COUNT(session_id) / COUNT(DISTINCT user_id)
    unit: 回
    aggregation: 非加法的

  avg_session_duration:
    name: 平均セッション時間
    definition: セッションの平均継続時間
    formula: AVG(session_duration_seconds)
    unit: 秒
    aggregation: 非加法的

  pages_per_session:
    name: セッションあたりPV数
    definition: 1セッションで閲覧されるページ数
    formula: SUM(page_view_count) / COUNT(DISTINCT session_id)
    unit: ページ
    aggregation: 非加法的
```

#### 3.2.2 コンバージョンメトリクス

```yaml
conversion_metrics:
  booking_conversion_rate:
    name: 予約コンバージョン率
    definition: セッションから予約に至る割合
    formula: COUNT(booking_sessions) / COUNT(total_sessions) * 100
    unit: '%'
    aggregation: 非加法的
    targets:
      current: 2%
      year_1: 3%
      year_3: 5%

  search_to_view_rate:
    name: 検索→詳細閲覧率
    definition: 検索結果から詳細ページ閲覧に至る割合
    formula: COUNT(view_sessions) / COUNT(search_sessions) * 100
    unit: '%'
    aggregation: 非加法的

  view_to_book_rate:
    name: 詳細閲覧→予約率
    definition: 詳細ページ閲覧から予約に至る割合
    formula: COUNT(booking_sessions) / COUNT(view_sessions) * 100
    unit: '%'
    aggregation: 非加法的

  cart_abandonment_rate:
    name: カート放棄率
    definition: カートに追加後、購入に至らない割合
    formula: (COUNT(cart_sessions) - COUNT(purchase_sessions)) / COUNT(cart_sessions) * 100
    unit: '%'
    aggregation: 非加法的
    targets:
      current: 70%
      target: 50%

  checkout_completion_rate:
    name: チェックアウト完了率
    definition: チェックアウト開始から完了する割合
    formula: COUNT(completed) / COUNT(started) * 100
    unit: '%'
    aggregation: 非加法的
```

### 3.3 オペレーショナルメトリクス

#### 3.3.1 サービス品質メトリクス

```yaml
service_quality_metrics:
  cancellation_rate:
    name: キャンセル率
    definition: 予約に対するキャンセルの割合
    formula: COUNT(cancelled_bookings) / COUNT(total_bookings) * 100
    unit: '%'
    aggregation: 非加法的
    thresholds:
      good: '< 10%'
      warning: '10-20%'
      critical: '> 20%'

  no_show_rate:
    name: ノーショー率
    definition: 予約に対するノーショーの割合
    formula: COUNT(no_show_bookings) / COUNT(confirmed_bookings) * 100
    unit: '%'
    aggregation: 非加法的
    thresholds:
      good: '< 3%'
      warning: '3-5%'
      critical: '> 5%'

  customer_satisfaction_score:
    name: 顧客満足度スコア（CSAT）
    definition: アンケート回答の平均満足度
    formula: AVG(satisfaction_rating)
    unit: 点（1-5）
    aggregation: 非加法的
    targets:
      minimum: 4.0
      target: 4.5

  net_promoter_score:
    name: NPS
    definition: 推奨者の割合から批判者の割合を引いた指標
    formula: (promoters% - detractors%)
    unit: ポイント（-100〜100）
    aggregation: 非加法的
    targets:
      minimum: 30
      target: 50

  response_time:
    name: カスタマーサポート応答時間
    definition: 問い合わせから初回応答までの時間
    formula: AVG(first_response_time)
    unit: 時間
    aggregation: 非加法的
    targets:
      email: '< 24時間'
      chat: '< 5分'
```

---

## 第4章：分析アーキテクチャ

### 4.1 リアルタイム分析

#### 4.1.1 リアルタイム分析アーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         REAL-TIME ANALYTICS ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                           Event Sources                                  │    │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐              │    │
│  │   │   API    │  │  Mobile  │  │ Web SDK  │  │  System  │              │    │
│  │   │ Events   │  │  Events  │  │ Events   │  │  Events  │              │    │
│  │   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘              │    │
│  └────────┼──────────────┼──────────────┼──────────────┼─────────────────────┘    │
│           │              │              │              │                         │
│           └──────────────┼──────────────┼──────────────┘                         │
│                          │              │                                         │
│                          ▼              ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                         Apache Kafka                                     │    │
│  │                    (Event Streaming Platform)                            │    │
│  │                                                                          │    │
│  │   Topics: bookings, orders, user_activities, system_events               │    │
│  └──────────────────────────────────┬──────────────────────────────────────┘    │
│                                     │                                            │
│              ┌──────────────────────┼──────────────────────┐                    │
│              │                      │                      │                    │
│              ▼                      ▼                      ▼                    │
│  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐          │
│  │   Apache Flink    │  │   Apache Flink    │  │   Apache Flink    │          │
│  │  Real-time Agg    │  │   Fraud Detect    │  │  Alert Generator  │          │
│  └─────────┬─────────┘  └─────────┬─────────┘  └─────────┬─────────┘          │
│            │                      │                      │                      │
│            ▼                      ▼                      ▼                      │
│  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐          │
│  │       Redis       │  │     BigQuery      │  │   Alert System    │          │
│  │  (Real-time KPIs) │  │  (Streaming Insert)│  │  (PagerDuty/Slack)│          │
│  └─────────┬─────────┘  └───────────────────┘  └───────────────────┘          │
│            │                                                                     │
│            ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      Real-time Dashboards                                │    │
│  │                         (Grafana/Looker)                                 │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 4.1.2 リアルタイムKPI計算

```yaml
realtime_kpis:
  booking_metrics:
    update_frequency: 10秒
    storage: Redis
    metrics:
      - name: bookings_last_5min
        calculation: COUNT(bookings) WHERE timestamp > NOW() - 5min
        key_pattern: "rt:bookings:5min:{booking_type}"

      - name: bookings_today
        calculation: COUNT(bookings) WHERE date = TODAY()
        key_pattern: "rt:bookings:today:{booking_type}"

      - name: revenue_today
        calculation: SUM(total_amount) WHERE date = TODAY()
        key_pattern: "rt:revenue:today:{currency}"

  user_metrics:
    update_frequency: 1分
    storage: Redis
    metrics:
      - name: online_users
        calculation: COUNT(DISTINCT user_id) WHERE last_activity > NOW() - 5min
        key_pattern: "rt:users:online"

      - name: active_sessions
        calculation: COUNT(DISTINCT session_id) WHERE is_active = TRUE
        key_pattern: "rt:sessions:active"

  inventory_metrics:
    update_frequency: 1分
    storage: Redis
    metrics:
      - name: availability_rate
        calculation: available / total * 100
        key_pattern: "rt:inventory:{entity_type}:{entity_id}"
```

### 4.2 バッチ分析

#### 4.2.1 バッチ処理パイプライン

```yaml
batch_analytics_pipeline:
  daily_processing:
    schedule: "0 2 * * *"  # 毎日2:00 AM JST
    jobs:
      - name: fact_bookings_daily
        source: staging.stg_bookings
        target: dwh.fact_bookings
        processing_type: incremental

      - name: dim_user_scd
        source: staging.stg_users
        target: dwh.dim_user
        processing_type: scd2_merge

      - name: daily_aggregations
        dependencies: [fact_bookings_daily]
        targets:
          - mart.daily_booking_summary
          - mart.daily_revenue_summary

  hourly_processing:
    schedule: "0 * * * *"  # 毎時0分
    jobs:
      - name: fact_user_activity_hourly
        source: staging.stg_user_activities
        target: dwh.fact_user_activity
        processing_type: append

  weekly_processing:
    schedule: "0 4 * * 0"  # 毎週日曜4:00 AM
    jobs:
      - name: cohort_analysis_refresh
        target: mart.user_cohort_analysis

      - name: rfm_segmentation_refresh
        target: mart.user_rfm_segments
```

### 4.3 アドホック分析

#### 4.3.1 セルフサービス分析環境

```yaml
self_service_analytics:
  tools:
    primary: Looker
    secondary: Google Data Studio
    advanced: BigQuery Console / Jupyter Notebooks

  data_access_layers:
    certified_datasets:
      description: 品質保証済みのデータセット
      access: 全アナリスト
      examples:
        - mart.booking_summary
        - mart.user_summary
        - mart.revenue_summary
      governance:
        - 定義書あり
        - データ品質チェック済み
        - 更新スケジュール明確

    exploration_datasets:
      description: 探索的分析用データセット
      access: シニアアナリスト
      examples:
        - dwh.fact_*
        - dwh.dim_*
      governance:
        - 基本的な品質チェック
        - 本番レポートには非推奨

    raw_datasets:
      description: 生データアクセス
      access: データエンジニア/データサイエンティスト
      examples:
        - raw.*
        - staging.*
      governance:
        - 品質未保証
        - 探索・検証目的のみ
```

---

## 第5章：ダッシュボード & レポーティング

### 5.1 エグゼクティブダッシュボード

#### 5.1.1 ダッシュボード設計

```yaml
executive_dashboard:
  name: TripTrip Executive Overview
  refresh_frequency: 日次（一部リアルタイム）
  audience: CEO, COO, 事業部長

  sections:
    revenue_overview:
      title: 収益概要
      visualizations:
        - type: scorecard
          metrics:
            - 当月GBV（前月比、目標比）
            - 当月純収益（前月比、目標比）
            - YTD累計収益（前年比）

        - type: line_chart
          title: 収益トレンド（12ヶ月）
          dimensions: month
          metrics: [gbv, net_revenue, commission]

        - type: bar_chart
          title: 予約タイプ別収益
          dimensions: booking_type
          metrics: [revenue, booking_count]

    growth_metrics:
      title: 成長指標
      visualizations:
        - type: scorecard
          metrics:
            - MAU（前月比）
            - 新規ユーザー数（前月比）
            - 予約件数（前月比）

        - type: area_chart
          title: ユーザー成長トレンド
          dimensions: week
          metrics: [mau, new_users, returning_users]

        - type: funnel_chart
          title: コンバージョンファネル
          stages: [visitors, signups, first_booking, repeat_booking]

    operational_health:
      title: オペレーション健全性
      visualizations:
        - type: gauge
          metrics:
            - キャンセル率（目標: < 10%）
            - CSAT（目標: > 4.0）
            - NPS（目標: > 30）

        - type: heatmap
          title: 地域別パフォーマンス
          dimensions: [region, metric]
          metrics: [bookings, revenue, satisfaction]

    alerts:
      title: 重要アラート
      visualizations:
        - type: alert_list
          sources:
            - 目標未達指標
            - 異常検知アラート
            - システムアラート
```

### 5.2 オペレーショナルダッシュボード

#### 5.2.1 リアルタイムオペレーションダッシュボード

```yaml
operational_dashboard:
  name: Real-time Operations Center
  refresh_frequency: 10秒
  audience: オペレーションチーム

  sections:
    live_metrics:
      title: ライブメトリクス
      visualizations:
        - type: big_number
          metrics:
            - 現在のオンラインユーザー数
            - 直近5分の予約数
            - 直近5分の検索数

        - type: line_chart
          title: 分単位トラフィック
          time_range: last_1_hour
          granularity: 1_minute
          metrics: [requests, bookings, errors]

    booking_monitor:
      title: 予約モニター
      visualizations:
        - type: stream
          title: リアルタイム予約フィード
          fields: [time, booking_type, amount, status]

        - type: pie_chart
          title: ステータス別予約数（本日）
          dimension: booking_status
          metric: booking_count

    inventory_status:
      title: 在庫状況
      visualizations:
        - type: table
          title: 低在庫アラート
          filters: [available_rate < 20%]
          columns: [entity_name, total, available, booked, rate]

        - type: heatmap
          title: 日別在庫状況（今後30日）
          dimensions: [date, entity_type]
          metric: availability_rate

    system_health:
      title: システム健全性
      visualizations:
        - type: gauge
          metrics:
            - API応答時間P95
            - エラー率
            - キャッシュヒット率

        - type: line_chart
          title: システムパフォーマンス
          metrics: [latency_p50, latency_p95, error_rate]
```

### 5.3 セルフサービスBI

#### 5.3.1 Looker LookML モデル設計

```yaml
looker_model:
  connection: triptrip_bigquery

  explores:
    bookings:
      label: 予約分析
      description: 予約データの詳細分析
      base_view: fact_bookings
      joins:
        - view: dim_date
          type: left_outer
          relationship: many_to_one
          sql_on: ${fact_bookings.booking_date_key} = ${dim_date.date_key}

        - view: dim_user
          type: left_outer
          relationship: many_to_one
          sql_on: ${fact_bookings.user_key} = ${dim_user.user_key}
                  AND ${dim_user.is_current} = TRUE

        - view: dim_entity
          type: left_outer
          relationship: many_to_one
          sql_on: ${fact_bookings.entity_key} = ${dim_entity.entity_key}
                  AND ${dim_entity.is_current} = TRUE

      access_filters:
        - field: dim_partner.partner_id
          user_attribute: partner_access

    orders:
      label: 注文分析
      description: Eコマース注文データの分析
      base_view: fact_orders
      joins:
        - view: dim_date
        - view: dim_user
        - view: dim_product
        - view: dim_category

    user_activity:
      label: ユーザー行動分析
      description: ユーザーアクティビティの分析
      base_view: fact_user_activity
      joins:
        - view: dim_date
        - view: dim_user
        - view: dim_page
        - view: dim_device

  views:
    fact_bookings:
      sql_table_name: dwh.fact_bookings

      dimensions:
        - name: booking_key
          type: number
          primary_key: yes
          hidden: yes

        - name: booking_id
          type: string
          label: 予約ID

        - name: booking_status
          type: string
          label: 予約ステータス

        - name: booking_type
          type: string
          label: 予約タイプ

      measures:
        - name: total_bookings
          type: count_distinct
          sql: ${booking_id}
          label: 予約件数

        - name: total_revenue
          type: sum
          sql: ${total_amount}
          label: 総収益
          value_format: "¥#,##0"

        - name: avg_booking_value
          type: average
          sql: ${total_amount}
          label: 平均予約単価
          value_format: "¥#,##0"

        - name: cancellation_rate
          type: number
          sql: ${cancelled_bookings} / NULLIF(${total_bookings}, 0) * 100
          label: キャンセル率
          value_format: "0.0%"
```

---

## 第6章：予測分析 & ML統合

### 6.1 予測分析フレームワーク

#### 6.1.1 予測モデル一覧

```yaml
predictive_models:
  demand_forecasting:
    name: 需要予測モデル
    purpose: 施設・商品の将来需要を予測
    algorithm: Prophet / LSTM
    features:
      - historical_bookings
      - seasonality
      - holidays
      - weather
      - events
      - price
    target: daily_bookings
    prediction_horizon: 90日
    update_frequency: 週次
    accuracy_target: MAPE < 15%

  churn_prediction:
    name: チャーン予測モデル
    purpose: 離脱リスクの高いユーザーを特定
    algorithm: XGBoost / LightGBM
    features:
      - days_since_last_activity
      - booking_frequency
      - cancellation_rate
      - session_count
      - email_engagement
      - support_tickets
    target: churn_in_30_days
    update_frequency: 日次
    accuracy_target: AUC > 0.80

  ltv_prediction:
    name: LTV予測モデル
    purpose: 顧客生涯価値を予測
    algorithm: Probabilistic Model (BG/NBD + Gamma-Gamma)
    features:
      - recency
      - frequency
      - monetary_value
      - tenure
    target: 12_month_revenue
    update_frequency: 週次
    accuracy_target: MAPE < 20%

  price_optimization:
    name: 価格最適化モデル
    purpose: 収益最大化のための価格推奨
    algorithm: Reinforcement Learning
    features:
      - demand_forecast
      - competitor_prices
      - inventory_level
      - day_of_week
      - lead_time
    target: optimal_price
    update_frequency: リアルタイム
    constraints:
      - min_price
      - max_price
      - rate_parity

  recommendation:
    name: 推奨モデル
    purpose: パーソナライズされた商品・施設推奨
    algorithm: Collaborative Filtering + Content-Based
    features:
      - user_preferences
      - booking_history
      - browsing_history
      - similar_users
      - item_attributes
    target: click_probability
    update_frequency: リアルタイム
    accuracy_target: CTR > 5%
```

### 6.2 ML統合アーキテクチャ

#### 6.2.1 特徴量ストア設計

```yaml
feature_store:
  platform: Vertex AI Feature Store

  feature_groups:
    user_features:
      entity: user_id
      features:
        - name: booking_count_30d
          type: INT64
          description: 過去30日の予約数
          update_frequency: daily

        - name: total_spend_90d
          type: FLOAT64
          description: 過去90日の総支出
          update_frequency: daily

        - name: avg_booking_value
          type: FLOAT64
          description: 平均予約単価
          update_frequency: daily

        - name: days_since_last_booking
          type: INT64
          description: 最終予約からの経過日数
          update_frequency: daily

        - name: cancellation_rate
          type: FLOAT64
          description: キャンセル率
          update_frequency: daily

        - name: preferred_booking_type
          type: STRING
          description: 最も多い予約タイプ
          update_frequency: weekly

    entity_features:
      entity: entity_id
      features:
        - name: avg_rating
          type: FLOAT64
          description: 平均評価スコア
          update_frequency: daily

        - name: booking_count_7d
          type: INT64
          description: 過去7日の予約数
          update_frequency: daily

        - name: occupancy_rate_30d
          type: FLOAT64
          description: 過去30日の稼働率
          update_frequency: daily

        - name: price_percentile
          type: FLOAT64
          description: カテゴリ内価格帯
          update_frequency: weekly

    contextual_features:
      entity: context_id
      features:
        - name: day_of_week
          type: INT64
          description: 曜日（1-7）
          update_frequency: realtime

        - name: is_holiday
          type: BOOL
          description: 祝日フラグ
          update_frequency: realtime

        - name: days_until_booking_date
          type: INT64
          description: 予約日までの日数
          update_frequency: realtime
```

---

## 第7章：戦略的示唆 & 推奨事項

### 7.1 実装優先順位

```yaml
implementation_priorities:
  phase_1_foundation:
    timeline: 月1-3
    deliverables:
      - BigQueryデータウェアハウス構築
      - 基本ファクト/ディメンションテーブル
      - 日次ETLパイプライン
      - エグゼクティブダッシュボード（v1）
    success_criteria:
      - 主要KPI自動更新
      - レポート作成時間 50%削減

  phase_2_analytics:
    timeline: 月4-6
    deliverables:
      - リアルタイム分析基盤
      - オペレーショナルダッシュボード
      - セルフサービスBI（Looker）
      - コホート分析
    success_criteria:
      - アナリストセルフサービス率 > 50%
      - リアルタイムKPI遅延 < 10秒

  phase_3_predictive:
    timeline: 月7-12
    deliverables:
      - 需要予測モデル
      - チャーン予測モデル
      - 特徴量ストア
      - 予測ダッシュボード
    success_criteria:
      - 予測精度 MAPE < 15%
      - 予測活用意思決定 > 20%

  phase_4_optimization:
    timeline: 月13-18
    deliverables:
      - 価格最適化
      - 推奨エンジン
      - 自動化されたアラート/アクション
    success_criteria:
      - 収益最適化効果 > 5%
      - 自動化率 > 30%
```

### 7.2 組織・プロセス推奨

```yaml
organizational_recommendations:
  team_structure:
    analytics_team:
      roles:
        - Analytics Manager (1)
        - Data Analyst (3-5)
        - Business Intelligence Engineer (2)
      responsibilities:
        - KPI定義と管理
        - ダッシュボード開発
        - アドホック分析

    data_science_team:
      roles:
        - Data Science Lead (1)
        - Data Scientist (2-3)
        - ML Engineer (2)
      responsibilities:
        - 予測モデル開発
        - 実験設計
        - モデル運用

  governance:
    data_governance_council:
      members:
        - CDO (Chair)
        - Analytics Manager
        - Legal/Compliance
        - Business Stakeholders
      responsibilities:
        - KPI定義承認
        - データ品質基準
        - アクセスポリシー

    metric_review_cadence:
      daily: オペレーショナルKPI
      weekly: 成長KPI、異常検知レビュー
      monthly: 戦略KPI、目標レビュー
      quarterly: KPI定義見直し
```

---

## 第8章：実装ロードマップ

### 8.1 詳細実装計画

```yaml
detailed_implementation_plan:
  month_1:
    focus: データウェアハウス基盤
    tasks:
      - BigQueryデータセット作成
      - dim_date, dim_time生成
      - fact_bookingsスキーマ作成
      - 初期データロード
    milestones:
      - 予約データ分析可能

  month_2:
    focus: ETLパイプライン
    tasks:
      - Airflow DAG開発
      - SCD2処理実装
      - データ品質チェック
      - インクリメンタルロード
    milestones:
      - 日次自動更新稼働

  month_3:
    focus: ダッシュボード（v1）
    tasks:
      - Looker接続設定
      - LookMLモデル開発
      - エグゼクティブダッシュボード
      - アラート設定
    milestones:
      - 経営層向けダッシュボードリリース

  month_4_6:
    focus: 拡張と最適化
    tasks:
      - リアルタイムパイプライン
      - 追加データマート
      - セルフサービス機能
      - パフォーマンス最適化
    milestones:
      - フル機能BIプラットフォーム

  month_7_12:
    focus: 予測分析
    tasks:
      - 特徴量エンジニアリング
      - モデル開発
      - 本番デプロイ
      - モニタリング
    milestones:
      - 予測モデル本番稼働
```

---

## 第9章：文書間参照 & 依存関係

### 9.1 関連文書マトリックス

```yaml
document_references:
  inputs:
    - document: Doc-DA-001
      title: Data Architecture & Schema Design
      relevance: ソーススキーマ定義、データモデル

    - document: Doc-DA-002
      title: Database Strategy & Technology Selection
      relevance: データソース接続、技術選定

    - document: Doc-DA-003
      title: Data Pipeline & ETL Architecture
      relevance: データパイプライン、ETL処理

    - document: Doc-SA-001
      title: System Architecture Overview
      relevance: システム全体構成

  outputs:
    - document: Doc-ML-001
      title: ML Platform Architecture
      dependency: 特徴量ストア、予測モデル

    - document: Doc-OP-001
      title: Operations Playbook
      dependency: オペレーショナルダッシュボード

  cross_references:
    - document: Doc-BS-008
      title: Financial Planning
      relevance: 財務KPI定義
```

### 9.2 技術スタックサマリー

```yaml
technology_stack_summary:
  data_warehouse:
    platform: BigQuery
    purpose: 分析データストア

  etl_orchestration:
    platform: Cloud Composer (Airflow)
    purpose: バッチ処理オーケストレーション

  real_time_processing:
    platform: Apache Flink
    purpose: リアルタイム集計

  caching:
    platform: Redis
    purpose: リアルタイムKPI

  visualization:
    platform: Looker
    purpose: ダッシュボード、セルフサービスBI

  ml_platform:
    platform: Vertex AI
    purpose: 予測モデル、特徴量ストア
```

---

## 付録

### A. KPI計算SQLサンプル

```sql
-- 月次KPIサマリー
WITH monthly_metrics AS (
    SELECT
        DATE_TRUNC(d.full_date, MONTH) AS month,
        COUNT(DISTINCT f.booking_id) AS total_bookings,
        SUM(f.total_amount_jpy) AS gross_booking_value,
        SUM(f.net_revenue_jpy) AS net_revenue,
        COUNT(DISTINCT f.user_key) AS unique_bookers,
        SUM(CASE WHEN f.booking_status = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled_bookings
    FROM dwh.fact_bookings f
    JOIN dwh.dim_date d ON f.booking_date_key = d.date_key
    WHERE d.full_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
    GROUP BY 1
)
SELECT
    month,
    total_bookings,
    gross_booking_value,
    net_revenue,
    unique_bookers,
    gross_booking_value / NULLIF(total_bookings, 0) AS avg_booking_value,
    cancelled_bookings / NULLIF(total_bookings, 0) * 100 AS cancellation_rate,
    LAG(total_bookings) OVER (ORDER BY month) AS prev_month_bookings,
    (total_bookings - LAG(total_bookings) OVER (ORDER BY month))
        / NULLIF(LAG(total_bookings) OVER (ORDER BY month), 0) * 100 AS mom_growth
FROM monthly_metrics
ORDER BY month DESC;
```

### B. 変更履歴

| バージョン | 日付 | 変更内容 | 作成者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-01-20 | 初版作成 | Technical Architecture Agent |

---

**文書情報**
- 文書ID: Doc-DA-004
- バージョン: 1.0.0
- ステータス: ドラフト
- 最終更新: 2026-01-20
- 次回レビュー: 2026-02-20

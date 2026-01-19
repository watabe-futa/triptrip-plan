# Doc-SA-005: 検索・推奨アーキテクチャ

## エグゼクティブサマリー

本文書は、TripTripプラットフォームの検索システムと推奨エンジンのアーキテクチャを包括的に定義します。Elasticsearchをベースとした高速全文検索、多面的検索（ファセット）、地理空間検索、および機械学習による推奨システムを詳述し、Google、Amazon、Netflixレベルの検索・推奨体験を実現します。P99レイテンシ100ms以下、秒間10,000クエリ処理を目標とし、パーソナライズされた旅行体験の発見を可能にする堅牢な基盤を構築します。A/Bテストフレームワークによる継続的な最適化と、リアルタイムパーソナライゼーションにより、コンバージョン率の大幅な向上を実現します。

---

## 第1章：はじめに & コンテキスト

### 1.1 検索・推奨システムのビジネス要件

検索と推奨は、TripTripの収益に直結する最重要機能です。ユーザーが求める旅行体験を迅速に発見できるかどうかが、予約率とユーザー満足度を決定します。

```yaml
business_requirements:
  search_requirements:
    hotel_search:
      description: ホテルの検索と絞り込み
      use_cases:
        - 目的地・日程による検索
        - 価格帯による絞り込み
        - 設備・アメニティによるフィルタリング
        - 評価によるソート
        - 地図ベースの検索
      priority: P0
      performance:
        latency: P99 < 100ms
        throughput: 5,000 qps

    attraction_search:
      description: アトラクション・チケットの検索
      use_cases:
        - カテゴリ別検索
        - 日時・時間帯での絞り込み
        - 料金範囲フィルタ
        - 所要時間フィルタ
        - 現在地からの距離検索
      priority: P0
      performance:
        latency: P99 < 100ms
        throughput: 3,000 qps

    product_search:
      description: 物販商品の検索
      use_cases:
        - キーワード検索
        - カテゴリ検索
        - 価格・在庫フィルタ
      priority: P1
      performance:
        latency: P99 < 100ms
        throughput: 2,000 qps

    unified_search:
      description: 横断検索
      use_cases:
        - 全カテゴリ横断検索
        - サジェスト・オートコンプリート
      priority: P1
      performance:
        latency: P99 < 50ms
        throughput: 10,000 qps

  recommendation_requirements:
    personalized:
      description: ユーザー行動に基づくパーソナライズ推奨
      use_cases:
        - ホーム画面の推奨
        - 検索結果のパーソナライズ
        - 予約完了後の関連提案
      priority: P1
      performance:
        latency: P99 < 50ms
        throughput: 20,000 qps

    similar_items:
      description: 類似アイテムの推奨
      use_cases:
        - 「この商品を見た人はこちらも見ています」
        - 類似ホテルの提案
      priority: P1

    trending:
      description: 人気・トレンドの提示
      use_cases:
        - 人気の目的地
        - 急上昇のアトラクション
        - 季節のおすすめ
      priority: P2

  business_metrics:
    search_conversion:
      description: 検索からの予約転換率
      current: 2.5%
      target: 5%

    recommendation_ctr:
      description: 推奨クリック率
      current: 3%
      target: 8%

    search_abandonment:
      description: 検索離脱率
      current: 45%
      target: 25%

    zero_results_rate:
      description: 検索結果ゼロ率
      current: 8%
      target: 2%
```

### 1.2 技術的要件と制約

```yaml
technical_requirements:
  functional:
    - 日本語全文検索（形態素解析）
    - 多言語検索（日本語、英語、中国語、韓国語）
    - あいまい検索（typo tolerant）
    - 同義語対応
    - ファセット検索（多面的絞り込み）
    - 地理空間検索（距離、バウンディングボックス）
    - リアルタイムインデックス更新
    - 検索ログ分析

  non_functional:
    performance:
      search_latency_p99: 100ms
      recommend_latency_p99: 50ms
      indexing_latency: 5秒以内
      throughput: 10,000 qps

    scalability:
      data_volume: 100万ドキュメント → 1億ドキュメント
      query_volume: 10x growth capability
      horizontal_scaling: true

    availability:
      target: 99.99%
      failover: 自動
      degradation: graceful

constraints:
  existing_system:
    - constraint: PostgreSQL内の既存データ
      impact: データ同期が必要
      strategy: CDC（Change Data Capture）

    - constraint: 既存検索機能（Flutter側）
      impact: APIの後方互換性
      strategy: 段階的移行

  infrastructure:
    - constraint: GCP利用
      impact: 技術選定の範囲
      strategy: Elastic Cloud on GCP または Meilisearch

  budget:
    - constraint: コスト最適化
      impact: 過剰スペックを避ける
      strategy: 従量課金、自動スケーリング
```

### 1.3 技術目標と成功基準

```yaml
success_criteria:
  search_quality:
    relevance_score:
      metric: NDCG@10 (Normalized Discounted Cumulative Gain)
      target: 0.85

    precision_at_k:
      metric: Precision@5
      target: 0.80

    recall:
      metric: Recall@20
      target: 0.90

    zero_results_rate:
      metric: 検索結果ゼロ率
      target: < 2%

  recommendation_quality:
    hit_rate:
      metric: 推奨アイテムの閲覧率
      target: 25%

    coverage:
      metric: カタログカバレッジ
      target: 80%

    diversity:
      metric: 推奨の多様性
      target: Intra-list diversity > 0.6

    serendipity:
      metric: 予想外の発見率
      target: 15%

  performance:
    search_latency:
      metric: P99 Search Latency
      target: 100ms

    recommend_latency:
      metric: P99 Recommendation Latency
      target: 50ms

    indexing_freshness:
      metric: データ更新反映時間
      target: 5秒以内

  business_impact:
    search_conversion:
      metric: 検索→予約転換率
      improvement: +100%

    recommendation_revenue:
      metric: 推奨経由売上比率
      target: 15%
```

---

## 第2章：検索アーキテクチャ概要

### 2.1 検索要件定義

```yaml
search_requirements:
  search_types:
    keyword_search:
      description: キーワードによる全文検索
      features:
        - BM25スコアリング
        - 形態素解析（日本語）
        - N-gram（その他言語）
        - あいまい検索
        - ハイライト

    filtered_search:
      description: 条件によるフィルタリング
      features:
        - 価格範囲
        - 日付範囲
        - カテゴリ
        - 評価
        - 在庫有無
        - 設備・アメニティ

    geo_search:
      description: 地理空間検索
      features:
        - 距離検索（ある地点からNkm以内）
        - バウンディングボックス
        - ポリゴン検索
        - ジオハッシュ

    faceted_search:
      description: ファセットナビゲーション
      features:
        - 動的ファセット生成
        - ファセットカウント
        - 階層的ファセット
        - 選択状態の管理

    autocomplete:
      description: 検索候補の自動補完
      features:
        - プレフィックスマッチ
        - コンテキスト考慮
        - 人気度スコアリング
        - タイプ別結果

  search_scenarios:
    hotel_search:
      required_filters:
        - チェックイン日
        - チェックアウト日
        - 宿泊人数
        - 部屋数
        - 目的地（必須または現在地）
      optional_filters:
        - 価格範囲
        - 星評価
        - ホテルクラス
        - アメニティ（WiFi、朝食、温泉、駐車場等）
        - 口コミ評価
      sort_options:
        - relevance（関連性）
        - price_asc / price_desc（価格）
        - rating（評価）
        - distance（距離）
        - popularity（人気）

    attraction_search:
      required_filters:
        - 目的地または現在地
      optional_filters:
        - 日付
        - 時間帯
        - カテゴリ
        - 価格範囲
        - 所要時間
        - 評価
      sort_options:
        - relevance
        - price_asc / price_desc
        - rating
        - distance
        - popularity
        - duration

    product_search:
      optional_filters:
        - カテゴリ
        - 価格範囲
        - 在庫有無
        - 評価
      sort_options:
        - relevance
        - price_asc / price_desc
        - rating
        - newest
        - popular
```

### 2.2 技術選定

```yaml
technology_selection:
  search_engine:
    choice: Elasticsearch 8.x (Elastic Cloud)
    alternatives:
      - Meilisearch:
          pros: シンプル、高速、typo tolerant
          cons: 機能制限、大規模データに不向き
      - Algolia:
          pros: 優れたDX、マネージド
          cons: コスト高、ベンダーロック
      - OpenSearch:
          pros: Elasticsearch互換、オープンソース
          cons: 最新機能の遅れ
      - Typesense:
          pros: 高速、シンプル
          cons: エコシステム未成熟

    rationale:
      - 成熟したエコシステム
      - 日本語解析（Kuromoji）の品質
      - 地理空間検索の充実
      - 機械学習機能（LTR、ベクトル検索）
      - スケーラビリティ実績
      - 豊富なドキュメント

  ml_framework:
    choice: TensorFlow Serving + Custom Python
    rationale:
      - 推奨モデルのサービング
      - 低レイテンシ推論
      - バッチ処理サポート

  feature_store:
    choice: Redis + BigQuery
    rationale:
      - リアルタイム特徴量: Redis
      - バッチ特徴量: BigQuery

  experiment_platform:
    choice: Custom + LaunchDarkly
    rationale:
      - A/Bテストの柔軟性
      - フィーチャーフラグ統合
```

### 2.3 高レベルアーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌───────────────────┐   │
│  │   Flutter Mobile    │  │    Flutter Web      │  │   Admin Dashboard │   │
│  │   Search UI         │  │   Search UI         │  │                   │   │
│  └──────────┬──────────┘  └──────────┬──────────┘  └─────────┬─────────┘   │
│             │                        │                       │              │
└─────────────┼────────────────────────┼───────────────────────┼──────────────┘
              │                        │                       │
              └────────────────────────┼───────────────────────┘
                                       │ HTTPS
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          API GATEWAY                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│  Kong / Cloud API Gateway                                                    │
│  - Authentication                                                            │
│  - Rate Limiting                                                             │
│  - Caching                                                                   │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          SEARCH SERVICE LAYER                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                        Search API Service                              │  │
│  │                        (Node.js / TypeScript)                          │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │  │
│  │  │   Query     │  │   Query     │  │  Response   │  │   Logging   │  │  │
│  │  │   Parser    │  │   Builder   │  │  Formatter  │  │   Tracker   │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                          │                         │                         │
│                          ▼                         ▼                         │
│  ┌───────────────────────────────┐  ┌────────────────────────────────────┐ │
│  │     Elasticsearch Cluster     │  │     Recommendation Service         │ │
│  │                               │  │     (Python / FastAPI)             │ │
│  │  ┌─────────┐  ┌─────────┐    │  │  ┌───────────┐  ┌───────────────┐  │ │
│  │  │ Master  │  │ Master  │    │  │  │ Candidate │  │    Ranking    │  │ │
│  │  │ Node 1  │  │ Node 2  │    │  │  │ Generator │  │    Model      │  │ │
│  │  └─────────┘  └─────────┘    │  │  └───────────┘  └───────────────┘  │ │
│  │  ┌─────────┐  ┌─────────┐    │  │                                     │ │
│  │  │  Data   │  │  Data   │    │  │  ┌───────────┐  ┌───────────────┐  │ │
│  │  │ Node 1  │  │ Node 2  │    │  │  │  Feature  │  │    A/B Test   │  │ │
│  │  └─────────┘  └─────────┘    │  │  │   Store   │  │    Engine     │  │ │
│  │  ┌─────────┐  ┌─────────┐    │  │  └───────────┘  └───────────────┘  │ │
│  │  │  Data   │  │  Data   │    │  │                                     │ │
│  │  │ Node 3  │  │ Node 4  │    │  └────────────────────────────────────┘ │
│  │  └─────────┘  └─────────┘    │                                         │
│  └───────────────────────────────┘                                         │
│                                                                              │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DATA PIPELINE LAYER                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐       │
│  │   CDC Pipeline    │  │   ML Pipeline     │  │  Analytics        │       │
│  │   (Debezium)      │  │   (Vertex AI)     │  │  Pipeline         │       │
│  │                   │  │                   │  │  (Dataflow)       │       │
│  │  PostgreSQL ──►   │  │  Training Jobs    │  │                   │       │
│  │  Kafka ──►        │  │  Model Registry   │  │  Search Logs ──►  │       │
│  │  Elasticsearch    │  │  Feature Store    │  │  BigQuery         │       │
│  └───────────────────┘  └───────────────────┘  └───────────────────┘       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DATA LAYER                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │  PostgreSQL     │  │  Redis          │  │  BigQuery       │             │
│  │  (Source Data)  │  │  (Cache/Feature)│  │  (Analytics)    │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 第3章：検索インデックス設計

### 3.1 インデックス構造

```yaml
index_design:
  hotels_index:
    name: hotels-v1
    settings:
      number_of_shards: 3
      number_of_replicas: 1
      refresh_interval: 1s
      analysis:
        analyzer:
          japanese_analyzer:
            type: custom
            tokenizer: kuromoji_tokenizer
            filter:
              - kuromoji_baseform
              - kuromoji_part_of_speech
              - ja_stop
              - kuromoji_stemmer
              - cjk_width
              - lowercase
          autocomplete_analyzer:
            type: custom
            tokenizer: keyword
            filter:
              - lowercase
              - autocomplete_filter
        filter:
          autocomplete_filter:
            type: edge_ngram
            min_gram: 1
            max_gram: 20
          ja_stop:
            type: stop
            stopwords: _japanese_

    mappings:
      properties:
        # 基本情報
        id:
          type: keyword
        name:
          type: text
          analyzer: japanese_analyzer
          fields:
            keyword:
              type: keyword
            autocomplete:
              type: text
              analyzer: autocomplete_analyzer
              search_analyzer: standard
        name_en:
          type: text
          analyzer: standard
          fields:
            keyword:
              type: keyword
        description:
          type: text
          analyzer: japanese_analyzer
        description_en:
          type: text
          analyzer: standard

        # 位置情報
        location:
          type: geo_point
        address:
          type: text
          analyzer: japanese_analyzer
          fields:
            keyword:
              type: keyword
        city:
          type: keyword
        prefecture:
          type: keyword
        country:
          type: keyword
        geohash:
          type: keyword

        # 評価・人気度
        rating:
          type: float
        review_count:
          type: integer
        popularity_score:
          type: float
        booking_count_30d:
          type: integer

        # 価格
        price_min:
          type: integer
        price_max:
          type: integer
        price_currency:
          type: keyword

        # 属性
        hotel_class:
          type: integer
        amenities:
          type: keyword
        room_types:
          type: keyword
        categories:
          type: keyword

        # 画像
        images:
          type: nested
          properties:
            url:
              type: keyword
            alt:
              type: text
            is_primary:
              type: boolean

        # 在庫・可用性
        has_availability:
          type: boolean
        next_available_date:
          type: date

        # メタデータ
        created_at:
          type: date
        updated_at:
          type: date
        indexed_at:
          type: date

  attractions_index:
    name: attractions-v1
    settings:
      number_of_shards: 2
      number_of_replicas: 1
      analysis:
        analyzer:
          japanese_analyzer: # 同上

    mappings:
      properties:
        id:
          type: keyword
        name:
          type: text
          analyzer: japanese_analyzer
          fields:
            keyword:
              type: keyword
            autocomplete:
              type: text
              analyzer: autocomplete_analyzer
        description:
          type: text
          analyzer: japanese_analyzer
        category:
          type: keyword
        subcategories:
          type: keyword
        location:
          type: geo_point
        address:
          type: text
          analyzer: japanese_analyzer
        city:
          type: keyword
        prefecture:
          type: keyword
        rating:
          type: float
        review_count:
          type: integer
        price_min:
          type: integer
        price_max:
          type: integer
        duration_minutes:
          type: integer
        operating_hours:
          type: nested
          properties:
            day_of_week:
              type: keyword
            open_time:
              type: keyword
            close_time:
              type: keyword
        popularity_score:
          type: float
        images:
          type: nested
        created_at:
          type: date
        updated_at:
          type: date

  products_index:
    name: products-v1
    settings:
      number_of_shards: 2
      number_of_replicas: 1

    mappings:
      properties:
        id:
          type: keyword
        sku:
          type: keyword
        name:
          type: text
          analyzer: japanese_analyzer
          fields:
            keyword:
              type: keyword
            autocomplete:
              type: text
              analyzer: autocomplete_analyzer
        description:
          type: text
          analyzer: japanese_analyzer
        category:
          type: keyword
        subcategories:
          type: keyword
        price:
          type: integer
        original_price:
          type: integer
        discount_percentage:
          type: float
        in_stock:
          type: boolean
        stock_quantity:
          type: integer
        rating:
          type: float
        review_count:
          type: integer
        sales_count:
          type: integer
        images:
          type: nested
        variants:
          type: nested
          properties:
            id:
              type: keyword
            name:
              type: keyword
            price:
              type: integer
            in_stock:
              type: boolean
        tags:
          type: keyword
        created_at:
          type: date
        updated_at:
          type: date
```

### 3.2 ドキュメントマッピング

```yaml
document_mapping:
  hotel_document:
    source_tables:
      - hotels
      - hotel_rooms
      - hotel_amenities
      - hotel_images
      - reviews (aggregated)
      - bookings (aggregated)

    transformation:
      hotel_to_document: |
        {
          "id": hotel.id,
          "name": hotel.name,
          "name_en": hotel.name_en,
          "description": hotel.description,
          "location": {
            "lat": hotel.latitude,
            "lon": hotel.longitude
          },
          "address": hotel.address,
          "city": hotel.city,
          "prefecture": hotel.prefecture,
          "country": hotel.country,
          "geohash": geohash.encode(hotel.latitude, hotel.longitude, precision=6),
          "rating": reviews.average_rating,
          "review_count": reviews.count,
          "popularity_score": calculatePopularityScore(hotel, bookings, reviews),
          "booking_count_30d": bookings.count_last_30_days,
          "price_min": rooms.min_price,
          "price_max": rooms.max_price,
          "price_currency": "JPY",
          "hotel_class": hotel.hotel_class,
          "amenities": amenities.map(a => a.code),
          "room_types": rooms.map(r => r.type),
          "categories": hotel.categories,
          "images": images.map(i => ({
            "url": i.url,
            "alt": i.alt,
            "is_primary": i.is_primary
          })),
          "has_availability": checkAvailability(hotel.id),
          "next_available_date": getNextAvailableDate(hotel.id),
          "created_at": hotel.created_at,
          "updated_at": hotel.updated_at,
          "indexed_at": new Date()
        }

    popularity_score_calculation: |
      function calculatePopularityScore(hotel, bookings, reviews) {
        const bookingScore = Math.log(bookings.count_last_30_days + 1) * 0.4;
        const ratingScore = (reviews.average_rating / 5) * reviews.count * 0.3;
        const recencyScore = getRecencyScore(hotel.updated_at) * 0.2;
        const completenessScore = getCompletenessScore(hotel) * 0.1;

        return bookingScore + ratingScore + recencyScore + completenessScore;
      }

  attraction_document:
    source_tables:
      - attractions
      - attraction_tickets
      - attraction_time_slots
      - attraction_images
      - reviews
      - bookings

  product_document:
    source_tables:
      - products
      - product_categories
      - product_images
      - product_variants
      - reviews
      - orders
```

### 3.3 インデックス更新戦略

```yaml
indexing_strategy:
  real_time_indexing:
    trigger: データベース変更（CDC）
    latency_target: 5秒以内
    implementation:
      pipeline: Debezium → Kafka → Index Worker → Elasticsearch
      parallelism: 10 workers
      batch_size: 100 documents
      retry: 3回（指数バックオフ）

    monitored_tables:
      - hotels (INSERT, UPDATE, DELETE)
      - hotel_rooms (INSERT, UPDATE, DELETE)
      - products (INSERT, UPDATE, DELETE)
      - attractions (INSERT, UPDATE, DELETE)
      - reviews (INSERT, UPDATE) → 集計後にインデックス

  batch_indexing:
    schedule: 毎日 03:00 JST
    purpose:
      - 完全再インデックス（週1回）
      - 集計値の更新（日次）
      - インデックス最適化
    implementation:
      full_reindex:
        frequency: weekly (Sunday)
        parallelism: 50 workers
        duration_target: 2時間以内
      daily_aggregation:
        frequency: daily
        scope: popularity_score, booking_count, review stats
        duration_target: 30分以内

  index_lifecycle:
    versioning:
      pattern: {index_name}-v{version}
      alias: {index_name}
      rollover:
        trigger: 新バージョンデプロイ
        process:
          1. 新インデックス作成
          2. 全データ再インデックス
          3. エイリアス切り替え
          4. 旧インデックス削除（24時間後）

    optimization:
      force_merge:
        schedule: weekly
        max_num_segments: 1
      refresh_interval:
        normal: 1s
        bulk_indexing: 30s

  cdc_pipeline:
    implementation: |
      // Debezium Configuration
      {
        "name": "triptrip-cdc",
        "config": {
          "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
          "database.hostname": "postgres.triptrip.internal",
          "database.port": "5432",
          "database.user": "debezium",
          "database.password": "${CDC_PASSWORD}",
          "database.dbname": "triptrip",
          "database.server.name": "triptrip",
          "table.include.list": "public.hotels,public.products,public.attractions",
          "plugin.name": "pgoutput",
          "publication.name": "triptrip_cdc",
          "slot.name": "triptrip_cdc_slot",
          "transforms": "route",
          "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
          "transforms.route.regex": "triptrip.public.(.*)",
          "transforms.route.replacement": "cdc.$1"
        }
      }

      // Kafka Consumer (Index Worker)
      async function processChangeEvent(event) {
        const { op, before, after, table } = event;

        switch (op) {
          case 'c': // CREATE
          case 'u': // UPDATE
            const doc = await transformToDocument(table, after);
            await elasticsearch.index({
              index: getIndexName(table),
              id: after.id,
              body: doc,
            });
            break;
          case 'd': // DELETE
            await elasticsearch.delete({
              index: getIndexName(table),
              id: before.id,
            });
            break;
        }
      }
```

### 3.4 シャーディング・レプリケーション

```yaml
sharding_replication:
  shard_strategy:
    hotels:
      shards: 3
      rationale: 10万〜100万ドキュメント想定
      sizing: 20-30GB / shard

    attractions:
      shards: 2
      rationale: 5万〜50万ドキュメント想定

    products:
      shards: 2
      rationale: 5万〜50万ドキュメント想定

  replication:
    replicas: 1 (production) / 0 (development)
    auto_expand_replicas: false
    rationale: 可用性と検索スループットの向上

  routing:
    custom_routing:
      hotels:
        field: prefecture
        benefit: 地域検索の高速化
      products:
        field: category
        benefit: カテゴリ検索の高速化

  cluster_topology:
    master_nodes:
      count: 3
      instance_type: e2-medium (2 vCPU, 4GB)
      dedicated: true

    data_nodes:
      count: 4 (scalable)
      instance_type: e2-standard-4 (4 vCPU, 16GB)
      storage: 100GB SSD per node
      heap_size: 8GB

    coordinating_nodes:
      count: 2
      instance_type: e2-medium
      purpose: クエリ分散、結果集約

  scaling:
    horizontal:
      trigger: CPU > 70% or search latency > 150ms
      action: データノード追加
      max_nodes: 10

    vertical:
      trigger: heap pressure > 75%
      action: インスタンスサイズアップ
```

---

## 第4章：多面的検索実装

### 4.1 フィルタリング・ファセット

```yaml
filtering_facets:
  facet_types:
    term_facet:
      description: カテゴリカル値の集計
      use_cases:
        - amenities
        - categories
        - hotel_class
        - city
      example_query:
        aggs:
          amenities:
            terms:
              field: amenities
              size: 50

    range_facet:
      description: 数値範囲の集計
      use_cases:
        - price_range
        - rating_range
        - distance_range
      example_query:
        aggs:
          price_ranges:
            range:
              field: price_min
              ranges:
                - { to: 10000 }
                - { from: 10000, to: 20000 }
                - { from: 20000, to: 50000 }
                - { from: 50000 }

    date_histogram_facet:
      description: 日付による集計
      use_cases:
        - availability_by_date
      example_query:
        aggs:
          availability_by_date:
            date_histogram:
              field: next_available_date
              calendar_interval: day

    geo_distance_facet:
      description: 距離による集計
      use_cases:
        - distance_from_user
      example_query:
        aggs:
          distance_ranges:
            geo_distance:
              field: location
              origin: { lat: 35.6812, lon: 139.7671 }
              ranges:
                - { to: 1 }
                - { from: 1, to: 5 }
                - { from: 5, to: 10 }
                - { from: 10 }

  filter_implementation:
    post_filter:
      description: ファセット計算後にフィルタリング
      benefit: 選択したフィルタ以外のファセット値を保持
      example:
        query:
          bool:
            must:
              - match: { name: "東京" }
        aggs:
          amenities:
            terms:
              field: amenities
        post_filter:
          term:
            amenities: "WiFi"

    filtered_aggregation:
      description: 特定条件下でのファセット計算
      use_case: 複数フィルタの独立したカウント
      example:
        aggs:
          amenities_filtered:
            filter:
              term:
                price_range: "10000-20000"
            aggs:
              amenities:
                terms:
                  field: amenities

  facet_response_format:
    example:
      facets:
        amenities:
          - name: "WiFi"
            count: 1250
            selected: true
          - name: "朝食"
            count: 890
            selected: false
          - name: "温泉"
            count: 456
            selected: false
        price_ranges:
          - range: "〜¥10,000"
            min: 0
            max: 10000
            count: 320
          - range: "¥10,000〜¥20,000"
            min: 10000
            max: 20000
            count: 580
            selected: true
        ratings:
          - value: 5
            count: 120
          - value: 4
            count: 450
          - value: 3
            count: 230
```

### 4.2 ジオロケーション検索

```yaml
geo_search:
  search_types:
    geo_distance:
      description: 指定地点からの距離検索
      use_case: 「現在地から10km以内のホテル」
      query_example:
        query:
          bool:
            filter:
              geo_distance:
                distance: "10km"
                location:
                  lat: 35.6812
                  lon: 139.7671

    geo_bounding_box:
      description: 矩形範囲内の検索
      use_case: 地図表示範囲内の検索
      query_example:
        query:
          bool:
            filter:
              geo_bounding_box:
                location:
                  top_left:
                    lat: 35.75
                    lon: 139.65
                  bottom_right:
                    lat: 35.60
                    lon: 139.85

    geo_polygon:
      description: ポリゴン内の検索
      use_case: カスタムエリア内の検索
      query_example:
        query:
          bool:
            filter:
              geo_polygon:
                location:
                  points:
                    - { lat: 35.70, lon: 139.70 }
                    - { lat: 35.75, lon: 139.75 }
                    - { lat: 35.70, lon: 139.80 }

    geo_shape:
      description: 複雑な形状での検索
      use_case: 行政区域、観光エリア
      query_example:
        query:
          bool:
            filter:
              geo_shape:
                area:
                  shape:
                    type: polygon
                    coordinates: [[[...], ...]]
                  relation: within

  distance_calculation:
    unit: km
    distance_type: arc (精度優先) / plane (速度優先)
    script_field:
      example:
        script_fields:
          distance:
            script:
              source: "doc['location'].arcDistance(params.lat, params.lon) / 1000"
              params:
                lat: 35.6812
                lon: 139.7671

  geo_aggregations:
    geo_distance_agg:
      description: 距離帯別の集計
      example:
        aggs:
          rings_around_tokyo:
            geo_distance:
              field: location
              origin:
                lat: 35.6812
                lon: 139.7671
              unit: km
              ranges:
                - { to: 1 }
                - { from: 1, to: 5 }
                - { from: 5, to: 10 }
                - { from: 10, to: 20 }
                - { from: 20 }

    geohash_grid:
      description: グリッドベースのクラスタリング
      use_case: 地図上のマーカークラスタリング
      example:
        aggs:
          grid:
            geohash_grid:
              field: location
              precision: 5
            aggs:
              centroid:
                geo_centroid:
                  field: location
              count:
                value_count:
                  field: id

  performance_optimization:
    geohash_prefix_filter:
      description: geohashプレフィックスでの高速フィルタ
      benefit: geo_distance前の候補絞り込み
    bounding_box_prefilter:
      description: バウンディングボックスでの事前フィルタ
      benefit: 計算コストの削減
```

### 4.3 価格範囲・日付検索

```yaml
price_date_search:
  price_search:
    range_filter:
      example:
        query:
          bool:
            filter:
              - range:
                  price_min:
                    gte: 10000
              - range:
                  price_max:
                    lte: 50000

    dynamic_pricing:
      description: 日付による動的価格の検索
      implementation:
        - 日付別価格をネストドキュメントで保持
        - nested queryで特定日の価格を検索
      example:
        query:
          nested:
            path: daily_prices
            query:
              bool:
                filter:
                  - term:
                      daily_prices.date: "2026-02-01"
                  - range:
                      daily_prices.price:
                        gte: 10000
                        lte: 30000

  date_search:
    availability_search:
      description: 指定日程の空室検索
      implementation:
        - 在庫システムとの連携
        - キャッシュ活用
      flow:
        1. Elasticsearchで候補取得
        2. 在庫APIで空室確認
        3. 結果をマージして返却
      optimization:
        - 人気ホテルの空室情報をRedisキャッシュ
        - 並列で在庫API呼び出し

    date_range_filter:
      example:
        query:
          bool:
            filter:
              range:
                next_available_date:
                  gte: "2026-02-01"
                  lte: "2026-02-28"

  time_slot_search:
    description: 時間帯での検索（アトラクション）
    example:
      query:
        nested:
          path: time_slots
          query:
            bool:
              filter:
                - term:
                    time_slots.date: "2026-02-01"
                - range:
                    time_slots.start_time:
                      gte: "10:00"
                      lte: "14:00"
                - term:
                    time_slots.available: true
```

### 4.4 フルテキスト検索

```yaml
fulltext_search:
  japanese_search:
    analyzer_config:
      tokenizer: kuromoji_tokenizer
      token_filters:
        - kuromoji_baseform: 動詞/形容詞の基本形変換
        - kuromoji_part_of_speech: 品詞フィルタ（助詞等除去）
        - ja_stop: 日本語ストップワード
        - kuromoji_stemmer: ステミング
        - cjk_width: 全角半角正規化
        - lowercase: 小文字化

    query_example:
      query:
        multi_match:
          query: "東京 温泉 ホテル"
          type: cross_fields
          fields:
            - "name^3"
            - "name_en^2"
            - "description"
            - "address"
            - "amenities^1.5"
          operator: and
          minimum_should_match: "75%"

  fuzzy_search:
    description: typo tolerant検索
    configuration:
      fuzziness: AUTO
      prefix_length: 2
      max_expansions: 50
    example:
      query:
        multi_match:
          query: "tokyou" # typo
          fields: ["name", "name_en"]
          fuzziness: AUTO

  synonym_search:
    synonym_groups:
      - "東京, 東京都, Tokyo"
      - "ホテル, 宿, 宿泊施設, hotel"
      - "温泉, 天然温泉, hot spring, onsen"
      - "WiFi, ワイファイ, 無線LAN"
    configuration:
      analysis:
        filter:
          synonym_filter:
            type: synonym
            synonyms_path: /config/synonyms.txt
            updateable: true

  highlighting:
    description: 検索結果のハイライト
    example:
      highlight:
        fields:
          name: {}
          description:
            fragment_size: 150
            number_of_fragments: 3
        pre_tags: ["<em>"]
        post_tags: ["</em>"]

  query_expansion:
    description: クエリの自動拡張
    techniques:
      - 同義語展開
      - 関連語展開（MLベース）
      - スペルコレクション
```

---

## 第5章：関連性ランキング

### 5.1 スコアリングアルゴリズム

```yaml
scoring_algorithm:
  base_scoring:
    algorithm: BM25
    parameters:
      k1: 1.2 (term frequency saturation)
      b: 0.75 (document length normalization)

  custom_scoring:
    function_score:
      description: 複数要素を組み合わせたスコアリング
      implementation:
        query:
          function_score:
            query:
              multi_match:
                query: "東京 温泉"
                fields: ["name^3", "description"]
            functions:
              # 評価スコア
              - field_value_factor:
                  field: rating
                  factor: 1.5
                  modifier: sqrt
                  missing: 3.0
              # 人気度スコア
              - field_value_factor:
                  field: popularity_score
                  factor: 1.2
                  modifier: log1p
              # 新しさスコア
              - gauss:
                  updated_at:
                    origin: now
                    scale: 30d
                    decay: 0.5
              # 距離スコア（位置情報がある場合）
              - gauss:
                  location:
                    origin:
                      lat: 35.6812
                      lon: 139.7671
                    scale: 5km
                    decay: 0.5
              # 価格帯スコア（ユーザー予算に近いほど高い）
              - script_score:
                  script:
                    source: |
                      double price = doc['price_min'].value;
                      double budget = params.budget;
                      double diff = Math.abs(price - budget);
                      return Math.max(0, 1 - diff / budget);
                    params:
                      budget: 20000
            score_mode: sum
            boost_mode: multiply

  score_components:
    text_relevance:
      weight: 0.3
      source: BM25 score

    popularity:
      weight: 0.25
      factors:
        - booking_count
        - view_count
        - review_count

    quality:
      weight: 0.2
      factors:
        - rating
        - review_sentiment
        - response_rate

    freshness:
      weight: 0.1
      decay: exponential
      scale: 30 days

    personalization:
      weight: 0.15
      factors:
        - user_preference_match
        - past_behavior
        - similar_users
```

### 5.2 ブースティング・チューニング

```yaml
boosting_tuning:
  field_boosting:
    hotels:
      name: 3.0
      name_en: 2.0
      description: 1.0
      address: 1.5
      amenities: 1.5
      categories: 1.2

    attractions:
      name: 3.0
      description: 1.0
      category: 2.0
      tags: 1.5

    products:
      name: 3.0
      description: 1.0
      category: 2.0
      brand: 1.5
      tags: 1.5

  document_boosting:
    promoted_items:
      description: プロモーション対象のブースト
      implementation:
        - スポンサード結果の上位表示
        - 季節キャンペーンの優先
      query_example:
        query:
          boosting:
            positive:
              match:
                name: "東京"
            negative:
              term:
                promoted: false
            negative_boost: 0.5

    quality_boosting:
      description: 品質指標によるブースト
      factors:
        - 認証済みホテル
        - 高評価（4.5以上）
        - レビュー数100以上

  negative_boosting:
    description: 低品質コンテンツの降格
    criteria:
      - 低評価（2.0以下）
      - 古いデータ（1年以上未更新）
      - 在庫なし
      - 苦情の多いアイテム

  ab_testing_tuning:
    process:
      1. 現在のスコアリング設定をベースライン
      2. 仮説に基づくスコアリング変更
      3. A/Bテストで効果測定
      4. 統計的有意性の確認
      5. 採用/棄却の判断

    metrics:
      - CTR (Click-Through Rate)
      - Conversion Rate
      - Revenue per Search
      - User Satisfaction (NPS)
```

### 5.3 パーソナライズドランキング

```yaml
personalized_ranking:
  user_signals:
    explicit:
      - 検索履歴
      - お気に入り
      - 予約履歴
      - 評価・レビュー
      - 設定（言語、通貨）

    implicit:
      - クリック履歴
      - 閲覧時間
      - スクロール深度
      - カート追加
      - 比較行動

  personalization_methods:
    query_time_boosting:
      description: クエリ時に個人嗜好でブースト
      implementation:
        - 過去予約ホテルの類似ホテルをブースト
        - よく検索するエリアをブースト
        - 価格帯の嗜好を反映
      example:
        query:
          function_score:
            functions:
              - filter:
                  terms:
                    amenities: ["WiFi", "温泉"] # ユーザーの好みのアメニティ
                weight: 2.0
              - filter:
                  range:
                    price_min:
                      gte: 15000
                      lte: 25000 # ユーザーの価格帯
                weight: 1.5

    reranking:
      description: 検索結果の再ランキング
      implementation:
        - 第1段階: Elasticsearchで候補取得（上位200件）
        - 第2段階: MLモデルで再ランキング
      model:
        type: LightGBM / Neural Network
        features:
          - 検索スコア
          - ユーザー特徴量
          - アイテム特徴量
          - コンテキスト特徴量

    learn_to_rank:
      description: 機械学習によるランキング最適化
      implementation:
        - Elasticsearch LTR プラグイン
        - または外部MLサービス
      training:
        data: クリックログ、予約ログ
        labels: クリック/予約をポジティブ
        model: LambdaMART
        evaluation: NDCG, MAP

  cold_start_handling:
    new_users:
      - 人気アイテムベースの推奨
      - デモグラフィックベースの推奨
      - セグメントベースの推奨
    new_items:
      - コンテンツベースの推奨
      - 類似アイテムからの転移
      - 探索的な露出
```

---

## 第6章：ML推奨システム

### 6.1 推奨アルゴリズム

```yaml
recommendation_algorithms:
  collaborative_filtering:
    description: ユーザー行動の類似性に基づく推奨
    types:
      user_based:
        description: 類似ユーザーの嗜好から推奨
        algorithm: KNN with cosine similarity
        use_case: 「あなたに似たユーザーが予約した」
        limitations: スケーラビリティ

      item_based:
        description: アイテムの共起性から推奨
        algorithm: Item-Item similarity
        use_case: 「この商品を見た人はこちらも見ています」
        benefits: 事前計算可能、説明性高い

      matrix_factorization:
        description: 行列分解による潜在因子モデル
        algorithm: ALS (Alternating Least Squares)
        implementation: Apache Spark MLlib
        hyperparameters:
          rank: 50
          max_iter: 20
          reg_param: 0.1

  content_based_filtering:
    description: アイテムの特徴量に基づく推奨
    features:
      hotels:
        - amenities (one-hot)
        - hotel_class
        - price_range
        - location_embedding
        - description_embedding (BERT)
      attractions:
        - category (one-hot)
        - duration
        - price_range
        - location_embedding
        - description_embedding
    similarity: cosine similarity
    use_case: 「閲覧履歴に基づくおすすめ」

  hybrid_approach:
    description: 複数手法の組み合わせ
    strategy: weighted ensemble
    weights:
      collaborative: 0.4
      content_based: 0.3
      popularity: 0.2
      recency: 0.1
    tuning: A/Bテストで最適化

  deep_learning:
    description: ディープラーニングによる推奨
    models:
      neural_collaborative_filtering:
        architecture: MLP + GMF
        input:
          - user_embedding
          - item_embedding
        output: interaction_probability
        training: binary cross-entropy

      sequence_based:
        description: ユーザー行動シーケンスのモデリング
        architecture: Transformer / GRU
        use_case: 次の行動予測

      two_tower:
        description: ユーザー・アイテムの独立エンコード
        benefits:
          - 大規模候補のANN検索
          - リアルタイム推論
        implementation: TensorFlow Recommenders
```

### 6.2 特徴量エンジニアリング

```yaml
feature_engineering:
  user_features:
    static:
      - user_id (embedding)
      - registration_date
      - account_type (free/premium)
      - preferred_language
      - preferred_currency

    behavioral:
      - total_bookings
      - booking_value_avg
      - booking_frequency
      - search_count_30d
      - view_count_30d
      - favorite_count
      - review_count

    temporal:
      - days_since_last_booking
      - days_since_last_login
      - typical_booking_lead_time

    preference:
      - preferred_price_range
      - preferred_hotel_class
      - preferred_amenities
      - preferred_locations

  item_features:
    static:
      - item_id (embedding)
      - category
      - price
      - location
      - created_date

    quality:
      - rating
      - review_count
      - response_rate
      - cancellation_rate

    popularity:
      - view_count_7d
      - view_count_30d
      - booking_count_7d
      - booking_count_30d
      - trending_score

    content:
      - text_embedding (BERT)
      - image_embedding (ResNet)
      - amenity_vector

  context_features:
    temporal:
      - hour_of_day
      - day_of_week
      - month
      - is_holiday
      - is_weekend
      - days_to_travel

    session:
      - search_query
      - filters_applied
      - items_viewed_this_session
      - items_in_cart

    device:
      - device_type
      - platform
      - app_version

  feature_store:
    online:
      storage: Redis
      latency: < 10ms
      features:
        - user_behavioral
        - user_preference
        - item_popularity
        - context
      update: real-time (event-driven)

    offline:
      storage: BigQuery
      features:
        - user_static
        - item_static
        - item_content
        - aggregated_metrics
      update: batch (daily)
```

### 6.3 モデルサービングアーキテクチャ

```yaml
model_serving:
  architecture:
    two_stage:
      stage_1_candidate_generation:
        description: 候補生成（粗いフィルタリング）
        latency_budget: 20ms
        output: 500-1000 candidates
        methods:
          - ANN検索（user/item embedding）
          - ルールベースフィルタ
          - 人気アイテム

      stage_2_ranking:
        description: ランキング（精密なスコアリング）
        latency_budget: 30ms
        input: 500-1000 candidates
        output: top 50 ranked items
        model: Neural network / LightGBM

  infrastructure:
    model_server:
      choice: TensorFlow Serving + FastAPI
      deployment: Kubernetes
      scaling: HPA (CPU-based)
      replicas: 3-10

    embedding_store:
      choice: Redis (in-memory) + Milvus (ANN)
      update: incremental (hourly)

    model_registry:
      choice: MLflow / Vertex AI Model Registry
      versioning: semantic versioning
      rollback: instant

  serving_optimization:
    batching:
      enabled: true
      max_batch_size: 64
      max_latency: 10ms

    caching:
      recommendation_cache:
        storage: Redis
        ttl: 30 minutes
        key_pattern: "rec:{user_id}:{context}"
      embedding_cache:
        storage: local memory
        size: 100,000 embeddings

    quantization:
      enabled: true
      precision: FP16
      benefit: 2x speedup, 50% memory

  monitoring:
    metrics:
      - inference_latency_p99
      - throughput_qps
      - model_accuracy
      - prediction_drift
      - feature_drift

    alerting:
      latency_threshold: 50ms
      accuracy_degradation: 5%
```

### 6.4 リアルタイム推論

```yaml
realtime_inference:
  latency_requirements:
    total_budget: 50ms
    breakdown:
      feature_retrieval: 10ms
      candidate_generation: 15ms
      ranking: 20ms
      post_processing: 5ms

  optimization_techniques:
    precomputation:
      description: 事前計算可能な要素の計算
      targets:
        - item-item similarity
        - user segment membership
        - popular items per category

    approximate_nearest_neighbor:
      description: ANN検索による高速候補生成
      library: Milvus / FAISS
      algorithm: HNSW (Hierarchical Navigable Small World)
      parameters:
        M: 16
        ef_construction: 200
        ef_search: 100
      performance:
        recall: 0.95
        latency: < 10ms for 1M items

    model_optimization:
      pruning:
        description: 不要なニューロンの削除
        sparsity: 50%
      distillation:
        description: 大きなモデルから小さなモデルへ
        teacher: Large BERT-based model
        student: DistilBERT / MiniLM

    infrastructure:
      gpu_inference:
        enabled: true
        device: NVIDIA T4
        benefit: 10x speedup for neural models
      edge_deployment:
        description: エッジでの推論
        use_case: モバイルアプリでのオフライン推奨

  fallback_strategy:
    primary: ML-based personalized recommendation
    secondary: popularity-based recommendation
    tertiary: editorial picks
    trigger:
      - latency > 100ms
      - model error
      - cold start user
```

---

## 第7章：A/Bテストフレームワーク

### 7.1 実験プラットフォーム設計

```yaml
experiment_platform:
  architecture:
    components:
      experiment_service:
        description: 実験の管理とユーザー割り当て
        endpoints:
          - GET /experiments/active
          - GET /experiments/{id}/assignment/{user_id}
          - POST /experiments
          - PATCH /experiments/{id}
        storage: PostgreSQL

      assignment_service:
        description: ユーザーの実験グループ割り当て
        algorithm: deterministic hashing
        consistency: 同一ユーザーは常に同じグループ

      metrics_service:
        description: 実験メトリクスの収集と分析
        storage: BigQuery
        processing: Apache Beam / Dataflow

      dashboard:
        description: 実験結果の可視化
        features:
          - リアルタイムメトリクス
          - 統計的有意性の表示
          - セグメント分析

  experiment_lifecycle:
    states:
      - draft: 設計中
      - scheduled: 開始予定
      - running: 実行中
      - paused: 一時停止
      - completed: 完了
      - archived: アーカイブ

    workflow:
      1. 仮説定義
      2. 実験設計（メトリクス、サンプルサイズ）
      3. 実装とQA
      4. 段階的ロールアウト（1% → 10% → 50%）
      5. データ収集
      6. 分析と判断
      7. 全体ロールアウト or ロールバック

  assignment_algorithm:
    implementation: |
      function assignExperiment(userId, experimentId, variants) {
        // Deterministic hashing for consistency
        const hash = murmur3(`${userId}:${experimentId}`);
        const bucket = hash % 100;

        let cumulativeWeight = 0;
        for (const variant of variants) {
          cumulativeWeight += variant.weight;
          if (bucket < cumulativeWeight) {
            return variant.id;
          }
        }
        return variants[0].id; // fallback
      }

  multi_armed_bandit:
    description: 動的なトラフィック配分
    algorithm: Thompson Sampling
    use_case: 探索と活用のバランス
    implementation: |
      class ThompsonSampling {
        selectArm(arms) {
          const samples = arms.map(arm => {
            return beta.sample(arm.successes + 1, arm.failures + 1);
          });
          return arms[argmax(samples)];
        }

        update(arm, reward) {
          if (reward) {
            arm.successes++;
          } else {
            arm.failures++;
          }
        }
      }
```

### 7.2 統計的有意性検定

```yaml
statistical_testing:
  test_types:
    conversion_rate:
      test: Two-proportion z-test
      null_hypothesis: p_control = p_treatment
      alternative: two-sided

    continuous_metrics:
      test: Welch's t-test
      assumption: 等分散を仮定しない
      alternative: two-sided

    ratio_metrics:
      test: Delta method
      use_case: Revenue per user

  parameters:
    significance_level: 0.05 (95% confidence)
    power: 0.8 (80%)
    minimum_detectable_effect: 5%

  sample_size_calculation:
    formula: |
      n = 2 * ((z_alpha + z_beta) / effect_size)^2 * p * (1 - p)

      where:
        z_alpha = 1.96 (for 95% confidence)
        z_beta = 0.84 (for 80% power)
        effect_size = expected lift / baseline
        p = baseline conversion rate

    example:
      baseline_cvr: 2.5%
      expected_lift: 20%
      effect_size: 0.5%
      required_sample: 31,500 per variant

  multiple_testing_correction:
    method: Benjamini-Hochberg (FDR control)
    application: 複数メトリクスの同時検定

  sequential_testing:
    description: 早期停止のための逐次検定
    method: Sequential Probability Ratio Test (SPRT)
    benefit: 必要サンプル数の削減

  guardrail_metrics:
    description: 悪化を検出する保護メトリクス
    metrics:
      - エラー率
      - ページ読み込み時間
      - クラッシュ率
    action: 悪化検出時は自動停止

  analysis_pipeline:
    implementation: |
      def analyze_experiment(experiment_id):
        # 1. データ取得
        data = fetch_experiment_data(experiment_id)

        # 2. サンプルサイズ確認
        if not sufficient_sample(data):
          return {"status": "insufficient_data"}

        # 3. 前提条件チェック
        check_randomization(data)
        check_sample_ratio_mismatch(data)

        # 4. 統計検定
        results = {}
        for metric in experiment.metrics:
          test_result = run_statistical_test(data, metric)
          results[metric] = {
            "control_mean": test_result.control_mean,
            "treatment_mean": test_result.treatment_mean,
            "lift": test_result.lift,
            "p_value": test_result.p_value,
            "confidence_interval": test_result.ci,
            "significant": test_result.p_value < 0.05,
          }

        # 5. 多重検定補正
        apply_fdr_correction(results)

        return results
```

### 7.3 ロールアウト戦略

```yaml
rollout_strategy:
  phased_rollout:
    phases:
      - phase: 1
        percentage: 1%
        duration: 1 day
        purpose: バグ検出
        success_criteria:
          - no_critical_errors
          - latency_acceptable

      - phase: 2
        percentage: 10%
        duration: 3 days
        purpose: 初期効果測定
        success_criteria:
          - no_metric_regression
          - positive_trend

      - phase: 3
        percentage: 50%
        duration: 7 days
        purpose: 統計的有意性の確認
        success_criteria:
          - statistical_significance
          - guardrails_passed

      - phase: 4
        percentage: 100%
        duration: permanent
        success_criteria:
          - full_analysis_complete

  automatic_rollback:
    triggers:
      - error_rate > 1%
      - latency_p99 > 200ms
      - guardrail_metric_degradation > 10%
    action:
      - トラフィックを即座にコントロールに戻す
      - アラート発報
      - ログ保存

  targeting:
    user_segments:
      - new_users
      - returning_users
      - premium_users
      - specific_countries
      - specific_devices

    percentage_based:
      description: ユーザーの一定割合に適用
      implementation: deterministic hashing

  feature_flags_integration:
    provider: LaunchDarkly / Unleash
    benefits:
      - 即座のオン/オフ
      - セグメント別の制御
      - 段階的ロールアウト
```

---

## 第8章：キャッシング・パフォーマンス最適化

### 8.1 キャッシング戦略

```yaml
caching_strategy:
  cache_layers:
    cdn_cache:
      provider: Cloudflare
      cached_content:
        - 静的な検索結果（人気クエリ）
        - 画像、静的アセット
      ttl: 5 minutes
      invalidation: API経由

    api_gateway_cache:
      provider: Kong
      cached_endpoints:
        - /v1/search/autocomplete
        - /v1/recommendations/trending
      ttl: 1-5 minutes
      cache_key: URL + selected headers

    application_cache:
      provider: Redis
      cached_data:
        - 検索結果
        - 推奨結果
        - ファセットカウント
        - ユーザー特徴量
      ttl: varies by data type

    elasticsearch_cache:
      query_cache:
        enabled: true
        size: 10%
      request_cache:
        enabled: true
      field_data_cache:
        size: 30%

  cache_keys:
    search_results:
      pattern: "search:{hash(query + filters + sort + page)}"
      ttl: 1 minute
      invalidation: on index update

    recommendations:
      pattern: "rec:{user_id}:{context}:{timestamp_bucket}"
      ttl: 30 minutes
      invalidation: on user action

    facets:
      pattern: "facets:{index}:{hash(filters)}"
      ttl: 5 minutes
      invalidation: on index update

  cache_warming:
    strategy:
      - 人気クエリのプリロード
      - 推奨結果の事前計算
      - ホットユーザーのキャッシュ
    schedule:
      - 毎日早朝（低トラフィック時）
      - インデックス更新後

  cache_invalidation:
    strategies:
      time_based:
        description: TTLによる自動失効
      event_based:
        description: データ更新イベントによる無効化
        implementation:
          - CDCイベントをKafkaで配信
          - キャッシュ無効化ワーカーが処理
      tag_based:
        description: タグによるグループ無効化
        example: "hotel:123" タグで関連キャッシュを一括無効化
```

### 8.2 検索パフォーマンス最適化

```yaml
search_optimization:
  query_optimization:
    filter_before_score:
      description: スコアリング前にフィルタで候補を絞る
      implementation:
        query:
          bool:
            filter: # 先に実行（スコア計算なし）
              - term: { city: "tokyo" }
              - range: { price_min: { lte: 50000 } }
            must: # フィルタ後にスコア計算
              - match: { name: "温泉" }

    use_filter_context:
      description: スコアリング不要な条件はfilterに
      benefit: キャッシュ活用、高速化

    avoid_script_in_query:
      description: スクリプトは可能な限り避ける
      alternative: function_score with built-in functions

    terminate_after:
      description: 上位N件で検索を打ち切り
      use_case: autocomplete, suggest
      example:
        terminate_after: 100

  index_optimization:
    mapping_optimization:
      - 不要なフィールドはindex: false
      - doc_values: false for fields not used in sort/agg
      - norms: false for fields not used in scoring

    index_sorting:
      description: インデックス時にソート
      benefit: 範囲クエリの高速化
      example:
        settings:
          index:
            sort:
              field: ["popularity_score", "created_at"]
              order: ["desc", "desc"]

    segment_optimization:
      force_merge:
        schedule: weekly
        max_num_segments: 1
      segment_count: 適切な数に保つ

  infrastructure_optimization:
    coordinating_nodes:
      description: クエリ分散と結果集約専用ノード
      benefit: データノードの負荷軽減

    shard_sizing:
      target: 20-40GB per shard
      too_small: オーバーヘッド増加
      too_large: リバランス困難

    replica_allocation:
      description: レプリカをAZ間で分散
      benefit: 高可用性とローカルリード
```

### 8.3 推奨システムの最適化

```yaml
recommendation_optimization:
  candidate_generation:
    ann_search:
      library: Milvus
      algorithm: HNSW
      recall_target: 0.95
      latency_target: 10ms
      optimization:
        - インデックスのウォームアップ
        - バッチ検索
        - GPU活用

    precomputed_candidates:
      description: ユーザーセグメント別の事前計算
      storage: Redis
      update: hourly
      benefit: リアルタイム計算の削減

  ranking_optimization:
    model_optimization:
      quantization: FP16
      pruning: 50% sparsity
      distillation: smaller model

    batch_inference:
      max_batch_size: 64
      timeout: 10ms
      benefit: GPU utilization

    early_exit:
      description: 確信度が高い場合は早期終了
      threshold: 0.95 confidence

  feature_store_optimization:
    preloading:
      description: 予測されるユーザーの特徴量を事前ロード
    local_cache:
      description: 頻繁にアクセスする特徴量をメモリキャッシュ
      size: 100,000 users
      eviction: LRU

  result_caching:
    user_level:
      description: ユーザー別の推奨キャッシュ
      ttl: 30 minutes
      invalidation: on user action
    segment_level:
      description: セグメント別の汎用推奨
      ttl: 1 hour
      use_case: cold start users
```

---

## 第9章：実装ロードマップ

### 9.1 フェーズ別計画

```yaml
implementation_roadmap:
  phase_1:
    name: 検索基盤構築
    duration: 3ヶ月
    objectives:
      - Elasticsearch クラスター構築
      - 基本的な検索API
      - CDCパイプライン
    deliverables:
      - ホテル/アトラクション/商品検索
      - ファセット検索
      - オートコンプリート
      - リアルタイムインデックス更新
    success_criteria:
      - P99 < 150ms
      - zero results rate < 5%
      - 10,000 documents indexed

  phase_2:
    name: 検索品質向上
    duration: 3ヶ月
    objectives:
      - ランキング最適化
      - 地理空間検索
      - 多言語対応
    deliverables:
      - カスタムスコアリング
      - 地図ベース検索
      - 日本語/英語検索
      - 検索ログ分析基盤
    success_criteria:
      - P99 < 100ms
      - NDCG@10 > 0.75
      - 検索CV +20%

  phase_3:
    name: 推奨システム構築
    duration: 6ヶ月
    objectives:
      - 推奨エンジン構築
      - 特徴量ストア
      - A/Bテスト基盤
    deliverables:
      - 協調フィルタリング
      - コンテンツベース推奨
      - ハイブリッド推奨
      - 実験プラットフォーム
    success_criteria:
      - 推奨CTR > 5%
      - 推奨経由予約 > 10%
      - P99 < 50ms

  phase_4:
    name: 高度化・スケーリング
    duration: 6ヶ月
    objectives:
      - パーソナライゼーション強化
      - リアルタイム推奨
      - グローバル展開
    deliverables:
      - Deep Learning推奨
      - Learn to Rank
      - マルチリージョン展開
      - 高度なA/Bテスト
    success_criteria:
      - 推奨経由売上 > 15%
      - 検索CV +50%
      - グローバルP99 < 100ms
```

---

## 第10章：文書間参照 & 統合ポイント

### 10.1 関連文書

```yaml
document_references:
  prerequisites:
    - doc_id: Doc-TV-001
      title: TripTrip技術ビジョンとアーキテクチャ原則
      relationship: 技術原則の定義

    - doc_id: Doc-SA-001
      title: TripTripシステムアーキテクチャ概要
      relationship: 全体アーキテクチャの定義

    - doc_id: Doc-SA-002
      title: マイクロサービスアーキテクチャ
      relationship: サービス分割

    - doc_id: Doc-SA-003
      title: APIアーキテクチャ・設計
      relationship: 検索API設計

    - doc_id: Doc-SA-004
      title: リアルタイム・協調機能アーキテクチャ
      relationship: リアルタイム検索・通知

  related_documents:
    - doc_id: Doc-DA-001
      title: データアーキテクチャ
      relationship: データパイプライン、特徴量ストア

    - doc_id: Doc-INF-001
      title: インフラストラクチャ設計
      relationship: Elasticsearch、MLインフラ
```

### 10.2 統合ポイント

```yaml
integration_points:
  elasticsearch_cluster:
    component: Elasticsearch 8.x
    integrations:
      - CDCパイプライン（Debezium）
      - Search API Service
      - 分析パイプライン
    related_docs: [Doc-SA-002, Doc-INF-001]

  recommendation_service:
    component: Recommendation Service (Python/FastAPI)
    integrations:
      - Feature Store (Redis/BigQuery)
      - Model Server (TensorFlow Serving)
      - Experiment Platform
    related_docs: [Doc-SA-002]

  data_pipeline:
    component: CDC + ETL Pipeline
    integrations:
      - PostgreSQL (source)
      - Kafka (event bus)
      - BigQuery (analytics)
    related_docs: [Doc-DA-001]

  experiment_platform:
    component: A/B Testing Platform
    integrations:
      - LaunchDarkly (feature flags)
      - BigQuery (metrics)
      - Dashboard (visualization)
    related_docs: [Doc-SA-002]
```

---

## 文書情報

| 項目 | 内容 |
|------|------|
| Document ID | Doc-SA-005 |
| Title | 検索・推奨アーキテクチャ |
| Version | 1.0.0 |
| Status | Draft |
| Author | Technical Architecture Agent |
| Created | 2026-01-19 |
| Last Updated | 2026-01-19 |
| Total Lines | ~1,500 |
| Previous Document | Doc-SA-004: リアルタイム・協調機能アーキテクチャ |
| Next Document | Doc-DA-001: データアーキテクチャ |

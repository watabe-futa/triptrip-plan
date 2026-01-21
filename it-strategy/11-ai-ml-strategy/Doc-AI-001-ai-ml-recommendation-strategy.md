# Doc-AI-001: AI/ML活用戦略（レコメンデーションエンジン）

## Executive Summary

本文書は、TripTripプラットフォームにおけるAI/ML活用戦略の中核となるレコメンデーションエンジンアーキテクチャを包括的に定義します。協調フィルタリング、コンテンツベースフィルタリング、深層学習ベースのハイブリッドアプローチを組み合わせ、旅行ドメイン特化の高精度推奨システムを構築します。リアルタイム推論基盤（P99レイテンシ<50ms）、特徴量ストア、MLOps基盤を含む包括的なML基盤により、パーソナライズされた旅行体験を提供し、コンバージョン率の大幅向上を実現します。本設計は、Netflix、Amazon、Googleレベルの推奨品質を目指し、既存のFlutter/Node.js/PostgreSQLスタックとシームレスに統合します。

---

## 第1章：はじめに & コンテキスト

### 1.1 ビジネス目標とAI活用の意義

#### 1.1.1 TripTripにおけるAI戦略的ポジション

TripTripは、AI/MLを競争優位性の源泉として位置づけ、以下のビジネス価値を創出します。

```yaml
ai_business_value:
  strategic_positioning:
    vision: AIファースト旅行プラットフォーム
    differentiation: パーソナライズされた旅行体験の提供
    competitive_advantage:
      - 個人の嗜好を深く理解した推奨
      - リアルタイムコンテキスト対応
      - 継続的学習による精度向上

  value_creation:
    revenue_impact:
      recommendation_revenue:
        current_baseline: 総予約の15%
        target_year_1: 総予約の30%
        target_year_3: 総予約の50%
      aov_increase:
        description: 平均注文額の向上
        mechanism: クロスセル・アップセル推奨
        target: +25%

    user_experience:
      discovery_efficiency:
        description: 理想的な旅行プランの発見時間短縮
        current: 平均45分
        target: 平均10分
      satisfaction:
        description: 推奨に対する満足度
        target: NPS +40

    operational_efficiency:
      curation_automation:
        description: 人手によるキュレーション作業の削減
        target: 80%自動化
      inventory_optimization:
        description: 在庫と需要のマッチング最適化
        target: 稼働率+15%
```

#### 1.1.2 ビジネスKPIとAI目標の整合

```yaml
kpi_alignment:
  primary_kpis:
    conversion_rate:
      definition: 推奨クリックから予約完了までの転換率
      current: 2.5%
      target_year_1: 5%
      target_year_3: 8%
      ai_contribution:
        - 関連性の高い推奨
        - 適切なタイミングでの表示
        - パーソナライズされた価格表示

    click_through_rate:
      definition: 推奨アイテムのクリック率
      current: 3%
      target_year_1: 8%
      target_year_3: 15%
      ai_contribution:
        - 視覚的に魅力的な推奨
        - コンテキスト認識配置
        - 動的なランキング最適化

    average_order_value:
      definition: 平均注文金額
      current: ¥25,000
      target_year_1: ¥32,000
      target_year_3: ¥45,000
      ai_contribution:
        - 関連アイテムの推奨
        - パッケージ・バンドル提案
        - 動的価格最適化

  secondary_kpis:
    catalog_coverage:
      definition: 推奨されるカタログアイテムの割合
      target: 80%
      importance: ロングテール商品の露出

    recommendation_diversity:
      definition: 推奨リストの多様性
      metric: Intra-List Diversity
      target: 0.6以上
      importance: フィルターバブル回避

    serendipity:
      definition: 予想外の発見の提供
      target: 推奨の15%以上
      importance: ユーザーエンゲージメント向上

    freshness:
      definition: 新着アイテムの推奨頻度
      target: 新着の30%以上を推奨
      importance: 新規在庫の露出
```

### 1.2 TripTripにおけるレコメンデーションの役割

#### 1.2.1 推奨シナリオマッピング

```yaml
recommendation_scenarios:
  home_screen:
    context: アプリ起動時のホーム画面
    objectives:
      - エンゲージメント促進
      - 関心の引き付け
      - リピート利用促進
    recommendation_types:
      - personalized_destinations: パーソナライズされた目的地
      - trending_experiences: 人気上昇中の体験
      - seasonal_picks: 季節のおすすめ
      - recently_viewed: 最近閲覧したアイテム
      - for_you: あなたへのおすすめ
    display_format:
      carousel: true
      card_count: 5-8
      refresh_frequency: セッションごと

  search_results:
    context: 検索結果ページ
    objectives:
      - 検索意図への適合
      - 代替案の提示
      - 検索精緻化支援
    recommendation_types:
      - search_result_ranking: 検索結果のパーソナライズ順位
      - similar_to_search: 類似アイテム
      - complementary_items: 補完アイテム
      - price_alternatives: 価格帯別代替案
    display_format:
      inline_results: true
      sidebar_suggestions: true
      refinement_chips: true

  item_detail:
    context: ホテル/アトラクション/商品詳細ページ
    objectives:
      - 購入決定支援
      - クロスセル
      - 代替案提示
    recommendation_types:
      - similar_items: 類似アイテム
      - frequently_bought_together: よく一緒に購入される
      - also_viewed: この商品を見た人はこちらも
      - upgrades: アップグレードオプション
    display_format:
      horizontal_carousel: true
      comparison_table: true
      bundle_suggestions: true

  cart_checkout:
    context: カート/チェックアウトページ
    objectives:
      - AOV向上
      - カート放棄防止
      - 完全な旅程提案
    recommendation_types:
      - complete_your_trip: 旅程完成提案
      - add_ons: アドオン提案
      - last_minute_deals: 直前セール
      - insurance_protection: 保険・保護プラン
    display_format:
      sidebar_panel: true
      modal_suggestions: true
      checkout_upsell: true

  post_booking:
    context: 予約完了後
    objectives:
      - リピート利用促進
      - 追加予約獲得
      - ロイヤリティ構築
    recommendation_types:
      - next_trip_suggestions: 次の旅行提案
      - local_experiences: 現地体験追加
      - dining_recommendations: レストラン推奨
      - share_worthy_spots: SNS映えスポット
    display_format:
      email_recommendations: true
      push_notifications: true
      in_app_suggestions: true
```

#### 1.2.2 推奨対象アイテムカテゴリ

```yaml
recommendation_items:
  hotels:
    attributes:
      - location
      - price_range
      - star_rating
      - amenities
      - room_types
      - reviews
      - photos
    recommendation_factors:
      - user_preferences
      - travel_purpose
      - group_composition
      - budget_constraints
      - historical_bookings
    freshness_requirement: daily
    inventory_sync: real_time

  attractions:
    attributes:
      - category (museum, outdoor, entertainment)
      - duration
      - price
      - location
      - time_slots
      - accessibility
      - age_suitability
    recommendation_factors:
      - interests
      - available_time
      - physical_ability
      - weather_conditions
      - crowd_levels
    freshness_requirement: hourly
    inventory_sync: real_time

  products:
    attributes:
      - category
      - brand
      - price
      - size
      - color
      - material
      - origin
    recommendation_factors:
      - purchase_history
      - browsing_behavior
      - style_preferences
      - gift_vs_personal
    freshness_requirement: daily
    inventory_sync: near_real_time

  restaurants:
    attributes:
      - cuisine_type
      - price_range
      - location
      - ambiance
      - dietary_options
      - reservation_availability
    recommendation_factors:
      - cuisine_preferences
      - meal_occasion
      - party_size
      - time_of_day
      - dietary_restrictions
    freshness_requirement: daily
    inventory_sync: real_time

  experiences:
    attributes:
      - activity_type
      - difficulty_level
      - duration
      - group_size
      - equipment_provided
      - language_support
    recommendation_factors:
      - adventure_level
      - fitness_level
      - interests
      - travel_companions
    freshness_requirement: daily
    inventory_sync: near_real_time
```

### 1.3 技術的前提条件と制約

#### 1.3.1 既存システムとの統合要件

```yaml
integration_requirements:
  frontend_integration:
    platform: Flutter 3.8.1+
    state_management: Provider + Riverpod
    requirements:
      - 非同期推奨取得API
      - オフラインキャッシュ対応
      - 推奨表示コンポーネント
      - イベントトラッキングSDK

  backend_integration:
    framework: Hono 4.8.3
    database: PostgreSQL 16
    requirements:
      - 推奨API エンドポイント
      - ユーザー行動ログ収集
      - 特徴量計算パイプライン
      - モデル推論サービス連携

  data_integration:
    sources:
      - PostgreSQL (マスターデータ)
      - Elasticsearch (検索インデックス)
      - Redis (キャッシュ)
      - Kafka (イベントストリーム)
    requirements:
      - リアルタイムデータ同期
      - 特徴量ストア連携
      - A/Bテストデータ収集

  existing_apis:
    user_service:
      endpoint: /api/users
      data: プロフィール、設定、認証
    product_service:
      endpoint: /api/products
      data: 商品カタログ、在庫
    attraction_service:
      endpoint: /api/attractions
      data: アトラクション、時間帯
    order_service:
      endpoint: /api/orders
      data: 注文履歴、予約
```

#### 1.3.2 非機能要件

```yaml
non_functional_requirements:
  performance:
    latency:
      online_inference:
        p50: 20ms
        p95: 40ms
        p99: 50ms
      batch_inference:
        throughput: 1,000,000 recommendations/hour
    throughput:
      peak_qps: 10,000
      sustained_qps: 5,000
    concurrency:
      max_concurrent_users: 100,000

  scalability:
    horizontal:
      inference_service: auto-scaling
      feature_store: partitioned
      model_registry: replicated
    vertical:
      gpu_scaling: supported
      memory_scaling: 64GB per instance

  availability:
    target: 99.9%
    failover:
      strategy: active-passive
      recovery_time: < 30秒
    degradation:
      fallback: popularity-based recommendations
      graceful: true

  data_requirements:
    training_data:
      minimum_users: 10,000
      minimum_interactions: 100,000
      cold_start_threshold: 5 interactions
    feature_freshness:
      real_time_features: < 1秒
      near_real_time: < 1分
      batch_features: < 1時間

  compliance:
    privacy:
      - GDPR準拠
      - CCPA準拠
      - 個人情報保護法準拠
    data_retention:
      raw_logs: 90日
      aggregated_data: 2年
      model_artifacts: 永続
```

---

## 第2章：レコメンデーションアーキテクチャ設計

### 2.1 推奨アルゴリズム選定

#### 2.1.1 アルゴリズム評価マトリクス

```yaml
algorithm_evaluation:
  collaborative_filtering:
    user_based_cf:
      description: 類似ユーザーの行動に基づく推奨
      strengths:
        - シンプルで解釈しやすい
        - 新規アイテムに対応可能
        - ユーザー間の類似性を活用
      weaknesses:
        - スケーラビリティの課題
        - スパース性に弱い
        - 新規ユーザー問題
      use_cases:
        - 少数の類似ユーザーが存在する場合
        - 明示的な評価データがある場合
      implementation: Approximate Nearest Neighbors (ANN)
      complexity: O(n * m) → O(log n) with ANN

    item_based_cf:
      description: 類似アイテムの共起に基づく推奨
      strengths:
        - 安定した推奨
        - 事前計算可能
        - 解釈しやすい
      weaknesses:
        - 新規アイテム問題
        - 多様性の欠如
        - 人気バイアス
      use_cases:
        - 「この商品を見た人はこちらも」
        - 類似アイテム推奨
      implementation: Item-Item Similarity Matrix
      complexity: O(m^2) 事前計算

    matrix_factorization:
      description: 行列分解による潜在因子モデル
      algorithms:
        - SVD (Singular Value Decomposition)
        - ALS (Alternating Least Squares)
        - NMF (Non-negative Matrix Factorization)
      strengths:
        - スパース性に強い
        - 潜在因子の発見
        - スケーラビリティ
      weaknesses:
        - 解釈が困難
        - ハイパーパラメータ調整
        - 新規ユーザー/アイテム問題
      use_cases:
        - 大規模ユーザーベース
        - 暗黙的フィードバック
      implementation: Spark ALS, Implicit library
      complexity: O(k * iterations * (n + m))

  content_based_filtering:
    text_based:
      description: テキスト特徴量に基づく推奨
      features:
        - TF-IDF
        - Word2Vec / FastText
        - BERT embeddings
      strengths:
        - 新規アイテムに対応
        - ドメイン知識活用
        - 説明可能性
      weaknesses:
        - 特徴エンジニアリング必要
        - 過学習リスク
        - セレンディピティ低下
      use_cases:
        - アトラクション説明文
        - ホテルレビュー
        - 商品説明
      implementation: Sentence Transformers

    attribute_based:
      description: 構造化属性に基づく推奨
      features:
        - カテゴリ
        - 価格帯
        - 場所
        - 評価
      strengths:
        - 解釈しやすい
        - ビジネスルール統合
        - フィルタリング容易
      weaknesses:
        - 特徴設計依存
        - 新規属性追加コスト
      use_cases:
        - 検索フィルタ補完
        - 属性ベースのクロスセル
      implementation: Feature hashing + similarity

    image_based:
      description: 画像特徴量に基づく推奨
      models:
        - ResNet embeddings
        - Vision Transformer (ViT)
        - CLIP embeddings
      strengths:
        - 視覚的類似性
        - テキスト不要
        - マルチモーダル対応
      weaknesses:
        - 計算コスト
        - 画像品質依存
      use_cases:
        - 着物/商品の視覚的類似性
        - 観光地の雰囲気類似
      implementation: CLIP + Faiss

  deep_learning:
    neural_collaborative_filtering:
      description: ニューラルネットワークによるCF
      architectures:
        - GMF (Generalized Matrix Factorization)
        - MLP (Multi-Layer Perceptron)
        - NeuMF (Neural Matrix Factorization)
      strengths:
        - 非線形関係の学習
        - 柔軟なアーキテクチャ
        - 高精度
      weaknesses:
        - 学習コスト
        - 過学習リスク
        - 説明困難
      use_cases:
        - 暗黙的フィードバック
        - 大規模データ
      implementation: PyTorch + RecBole

    sequence_models:
      description: シーケンシャルな行動パターン学習
      architectures:
        - GRU4Rec
        - SASRec (Self-Attentive)
        - BERT4Rec
      strengths:
        - 時間的パターン捕捉
        - セッションベース推奨
        - 最新行動反映
      weaknesses:
        - シーケンス長依存
        - 計算コスト
      use_cases:
        - セッション内推奨
        - 次のアクション予測
      implementation: Transformers + custom heads

    graph_neural_networks:
      description: グラフ構造を活用した推奨
      architectures:
        - LightGCN
        - PinSage
        - GraphSAGE
      strengths:
        - 高次の関係性学習
        - スケーラビリティ
        - 豊富なコンテキスト
      weaknesses:
        - グラフ構築コスト
        - 実装複雑性
      use_cases:
        - ユーザー-アイテムグラフ
        - ソーシャル推奨
      implementation: PyG (PyTorch Geometric)
```

#### 2.1.2 TripTrip推奨アルゴリズム選定

```yaml
algorithm_selection:
  primary_algorithm:
    name: Two-Tower Neural Network
    rationale:
      - ユーザーとアイテムの独立エンコーディング
      - 高速な推論（事前計算可能）
      - スケーラビリティ
      - リアルタイム特徴量対応

    architecture:
      user_tower:
        inputs:
          - user_id_embedding (dim=128)
          - user_features (demographics, preferences)
          - behavioral_features (recent_views, bookings)
          - context_features (time, location, device)
        layers:
          - Dense(256, activation='relu')
          - BatchNormalization()
          - Dense(256, activation='relu')
          - Dropout(0.3)
          - Dense(128, activation='linear')  # user embedding
        output_dim: 128

      item_tower:
        inputs:
          - item_id_embedding (dim=128)
          - item_features (category, price, location)
          - text_embedding (description, reviews)
          - image_embedding (photos)
        layers:
          - Dense(256, activation='relu')
          - BatchNormalization()
          - Dense(256, activation='relu')
          - Dropout(0.3)
          - Dense(128, activation='linear')  # item embedding
        output_dim: 128

      scoring:
        method: dot_product  # or cosine_similarity
        temperature: 0.1

    training:
      loss: softmax_cross_entropy
      negatives: in_batch_negatives + hard_negatives
      optimizer: Adam(lr=0.001)
      batch_size: 2048
      epochs: 50

  secondary_algorithms:
    item_similarity:
      use_case: 類似アイテム推奨
      method: Item2Vec + ANN search
      update_frequency: daily

    session_based:
      use_case: セッション内推奨
      method: SASRec
      update_frequency: real_time

    popularity:
      use_case: コールドスタート/フォールバック
      method: time-decayed popularity
      update_frequency: hourly

  ensemble_strategy:
    method: weighted_combination
    weights:
      two_tower: 0.6
      item_similarity: 0.2
      session_based: 0.15
      popularity: 0.05
    reranking:
      method: MMR (Maximal Marginal Relevance)
      diversity_weight: 0.3
```

### 2.2 旅行ドメイン特化モデル設計

#### 2.2.1 旅行特有の考慮事項

```yaml
travel_domain_considerations:
  temporal_patterns:
    seasonality:
      description: 季節性の強い需要変動
      patterns:
        - 桜シーズン（3-4月）
        - 紅葉シーズン（10-11月）
        - 夏休み（7-8月）
        - 年末年始
      modeling:
        - 季節特徴量の導入
        - 時期別モデルの学習
        - 動的な重み調整

    booking_window:
      description: 予約から旅行までの期間
      segments:
        - immediate: 0-3日前
        - short_term: 4-14日前
        - medium_term: 15-60日前
        - long_term: 60日以上前
      impact:
        - 価格感度の変化
        - 推奨の緊急度
        - 在庫状況の重要性

    trip_duration:
      description: 旅行期間に応じた推奨
      patterns:
        - day_trip: 日帰り
        - weekend: 1-2泊
        - short_vacation: 3-5泊
        - long_vacation: 6泊以上
      recommendations:
        - 短期: 効率的なプラン
        - 長期: 多様な体験

  group_composition:
    segments:
      solo:
        characteristics:
          - 柔軟性重視
          - 一人向けアクティビティ
          - 効率的な移動
        recommendations:
          - ゲストハウス
          - ウォーキングツアー
          - 一人向け体験

      couple:
        characteristics:
          - ロマンティック
          - 特別な体験
          - プライバシー
        recommendations:
          - ブティックホテル
          - 夜景スポット
          - カップル向け体験

      family:
        characteristics:
          - 子供向け設備
          - 安全性
          - 広いスペース
        recommendations:
          - ファミリールーム
          - キッズフレンドリー施設
          - 教育的体験

      group:
        characteristics:
          - コスト効率
          - グループアクティビティ
          - 調整の容易さ
        recommendations:
          - グループ割引
          - 団体ツアー
          - 大人数対応レストラン

  travel_purpose:
    leisure:
      characteristics:
        - 観光
        - リラクゼーション
        - 新しい体験
      recommendations:
        - 観光スポット
        - スパ
        - ローカル体験

    business:
      characteristics:
        - 効率性
        - ビジネス設備
        - 中心地アクセス
      recommendations:
        - ビジネスホテル
        - 会議室
        - 交通利便性

    special_occasion:
      characteristics:
        - 記念日
        - ハネムーン
        - 誕生日
      recommendations:
        - 高級宿泊
        - 特別体験
        - サプライズオプション

  geographic_constraints:
    location_aware:
      - 現在地からの距離
      - 交通アクセス
      - 地域の特性
    route_optimization:
      - 効率的な移動経路
      - 滞在時間最適化
      - マルチスポット推奨
```

#### 2.2.2 ドメイン特化特徴量設計

```yaml
domain_specific_features:
  user_features:
    static:
      - user_id
      - age_group
      - gender
      - nationality
      - preferred_language
      - membership_tier

    preference:
      - preferred_accommodation_type
      - budget_preference (budget/mid/luxury)
      - dietary_restrictions
      - accessibility_requirements
      - adventure_level
      - cultural_interests

    behavioral:
      - total_bookings
      - average_spend
      - favorite_categories
      - review_sentiment_avg
      - app_engagement_score
      - last_activity_timestamp

  item_features:
    hotel:
      - hotel_id
      - star_rating
      - price_per_night
      - location_lat_lng
      - amenities_vector
      - room_types_available
      - review_score
      - review_count
      - photo_quality_score
      - description_embedding

    attraction:
      - attraction_id
      - category_vector
      - duration_hours
      - price
      - location_lat_lng
      - operating_hours
      - age_restriction
      - accessibility_score
      - popularity_score
      - description_embedding

    product:
      - product_id
      - category_vector
      - price
      - brand_embedding
      - color_vector
      - size_options
      - material_vector
      - origin_country
      - image_embedding

  context_features:
    temporal:
      - hour_of_day
      - day_of_week
      - month
      - is_holiday
      - days_until_trip
      - booking_window_segment

    device:
      - device_type
      - os_version
      - app_version
      - screen_size
      - connection_type

    session:
      - session_duration
      - pages_viewed
      - search_queries
      - filters_applied
      - items_viewed_this_session

    geographic:
      - user_location_lat_lng
      - user_timezone
      - destination_lat_lng
      - distance_to_destination

  interaction_features:
    explicit:
      - ratings
      - reviews
      - favorites
      - wishlists

    implicit:
      - views
      - view_duration
      - clicks
      - add_to_cart
      - purchases
      - search_to_view
      - view_to_purchase
```

### 2.3 リアルタイム vs バッチ処理の使い分け

#### 2.3.1 処理戦略マトリクス

```yaml
processing_strategy:
  real_time:
    use_cases:
      - セッション内推奨
      - 検索結果パーソナライゼーション
      - カート推奨
      - コンテキスト依存推奨

    characteristics:
      latency_requirement: < 50ms
      freshness: 即時
      personalization: 高度

    implementation:
      inference: オンライン推論
      features: リアルタイム特徴量
      model: 事前学習済みモデル
      caching: 短期キャッシュ (TTL: 5分)

    examples:
      session_recommendations:
        trigger: ページ遷移
        features:
          - current_page_context
          - session_history
          - real_time_inventory
        latency_budget: 30ms

      search_personalization:
        trigger: 検索クエリ
        features:
          - query_embedding
          - user_preferences
          - real_time_availability
        latency_budget: 50ms

  near_real_time:
    use_cases:
      - ホーム画面推奨
      - メール推奨
      - プッシュ通知推奨
      - 類似アイテム

    characteristics:
      latency_requirement: < 1分
      freshness: 準リアルタイム
      personalization: 中程度

    implementation:
      inference: バッチ推論 + キャッシュ
      features: 集計特徴量
      model: 定期更新モデル
      caching: 中期キャッシュ (TTL: 15分)

    examples:
      home_recommendations:
        trigger: アプリ起動
        features:
          - user_profile
          - recent_activity_aggregates
          - trending_items
        computation: 15分ごとのバッチ

  batch:
    use_cases:
      - モデル学習
      - 特徴量計算
      - 類似度行列計算
      - オフライン評価

    characteristics:
      latency_requirement: < 24時間
      freshness: 日次
      personalization: グローバル

    implementation:
      inference: オフラインバッチ
      features: 履歴集計
      model: フルリトレーニング
      storage: データウェアハウス

    examples:
      model_training:
        trigger: 日次スケジュール
        data:
          - 30日間のインタラクション
          - ユーザー特徴量
          - アイテム特徴量
        output: 新規モデルアーティファクト

      item_similarity_matrix:
        trigger: 日次スケジュール
        computation:
          - Item2Vec学習
          - 類似度計算
          - ANN インデックス構築
        output: 類似アイテムインデックス

processing_pipeline:
  architecture:
    real_time_path:
      source: User Request
      flow:
        - API Gateway
        - Feature Store (online)
        - Inference Service
        - Reranker
        - Response
      components:
        - Redis (feature cache)
        - TensorFlow Serving / Triton
        - Python Reranker Service

    batch_path:
      source: Event Stream
      flow:
        - Kafka
        - Spark Streaming
        - Feature Store (offline)
        - ML Training Pipeline
        - Model Registry
      components:
        - Apache Kafka
        - Apache Spark
        - Feast
        - MLflow

    integration:
      feature_sync:
        direction: batch → real_time
        frequency: hourly
        mechanism: Feature Store materialization

      model_deployment:
        direction: batch → real_time
        frequency: daily
        mechanism: Model Registry → Serving
```

---

## 第3章：特徴量エンジニアリング

### 3.1 ユーザー特徴（行動履歴、嗜好、コンテキスト）

#### 3.1.1 ユーザー特徴量カタログ

```yaml
user_feature_catalog:
  identity_features:
    user_id:
      type: categorical
      encoding: embedding (dim=128)
      update: static
      description: ユーザー一意識別子

    account_age_days:
      type: numerical
      transformation: log1p
      update: daily
      description: アカウント作成からの経過日数

    membership_tier:
      type: categorical
      encoding: one_hot
      values: [bronze, silver, gold, platinum]
      update: event_driven
      description: 会員ランク

  demographic_features:
    age_group:
      type: categorical
      encoding: one_hot
      values: [18-24, 25-34, 35-44, 45-54, 55-64, 65+]
      update: static
      description: 年齢層

    gender:
      type: categorical
      encoding: one_hot
      values: [male, female, other, unknown]
      update: static
      description: 性別

    nationality:
      type: categorical
      encoding: embedding (dim=16)
      update: static
      description: 国籍

    preferred_language:
      type: categorical
      encoding: one_hot
      values: [ja, en, zh, ko, ...]
      update: event_driven
      description: 優先言語

  behavioral_aggregate_features:
    total_bookings:
      type: numerical
      transformation: log1p
      update: event_driven
      window: all_time
      description: 累計予約数

    bookings_last_30d:
      type: numerical
      transformation: none
      update: daily
      window: 30_days
      description: 過去30日間の予約数

    total_spend:
      type: numerical
      transformation: log1p
      update: event_driven
      window: all_time
      description: 累計利用金額

    average_order_value:
      type: numerical
      transformation: none
      update: daily
      window: 90_days
      description: 平均注文金額

    booking_frequency:
      type: numerical
      transformation: none
      update: daily
      formula: bookings_last_90d / 90
      description: 予約頻度（日次）

  preference_features:
    preferred_hotel_star:
      type: numerical
      transformation: none
      update: daily
      aggregation: weighted_average(booked_hotels.star_rating)
      description: 好みのホテル星評価

    preferred_price_range:
      type: categorical
      encoding: one_hot
      values: [budget, mid_range, luxury]
      update: daily
      aggregation: mode(booked_items.price_segment)
      description: 好みの価格帯

    preferred_categories:
      type: multi_hot
      encoding: multi_hot_vector (dim=50)
      update: daily
      aggregation: top_k(viewed_categories, k=10)
      description: 好みのカテゴリ

    destination_preferences:
      type: embedding
      encoding: average_embedding (dim=64)
      update: daily
      aggregation: avg(booked_destinations.embedding)
      description: 目的地嗜好埋め込み

  session_features:
    session_duration_sec:
      type: numerical
      transformation: log1p
      update: real_time
      description: 現セッション継続時間

    pages_viewed_session:
      type: numerical
      transformation: none
      update: real_time
      description: 現セッション閲覧ページ数

    items_viewed_session:
      type: list
      encoding: sequence_embedding
      max_length: 50
      update: real_time
      description: 現セッション閲覧アイテム

    searches_session:
      type: list
      encoding: query_embedding_sequence
      max_length: 10
      update: real_time
      description: 現セッション検索クエリ

    cart_items:
      type: list
      encoding: item_embedding_sequence
      max_length: 20
      update: real_time
      description: カート内アイテム

  context_features:
    hour_of_day:
      type: cyclical
      encoding: sin_cos (24h period)
      update: real_time
      description: 時刻（周期的）

    day_of_week:
      type: cyclical
      encoding: sin_cos (7d period)
      update: real_time
      description: 曜日（周期的）

    is_weekend:
      type: binary
      update: real_time
      description: 週末フラグ

    device_type:
      type: categorical
      encoding: one_hot
      values: [ios, android, web]
      update: real_time
      description: デバイス種別

    user_location:
      type: geolocation
      encoding: geohash + lat_lng
      update: real_time
      description: ユーザー現在地

    days_to_trip:
      type: numerical
      transformation: log1p
      update: real_time
      description: 旅行までの日数
```

#### 3.1.2 ユーザー埋め込み生成

```python
# ユーザー埋め込み生成アーキテクチャ
class UserEmbeddingModel:
    """
    ユーザー特徴量からユーザー埋め込みを生成
    """

    def __init__(self, config):
        self.user_id_embedding = Embedding(
            num_embeddings=config.num_users,
            embedding_dim=128
        )

        self.demographic_encoder = DenseEncoder(
            input_dim=config.demographic_dim,
            hidden_dims=[64, 64],
            output_dim=32
        )

        self.behavioral_encoder = DenseEncoder(
            input_dim=config.behavioral_dim,
            hidden_dims=[128, 64],
            output_dim=64
        )

        self.preference_encoder = DenseEncoder(
            input_dim=config.preference_dim,
            hidden_dims=[64, 64],
            output_dim=32
        )

        self.session_encoder = TransformerEncoder(
            input_dim=config.item_embedding_dim,
            num_heads=4,
            num_layers=2,
            output_dim=64
        )

        self.context_encoder = DenseEncoder(
            input_dim=config.context_dim,
            hidden_dims=[32],
            output_dim=16
        )

        self.fusion_layer = DenseEncoder(
            input_dim=128 + 32 + 64 + 32 + 64 + 16,  # 336
            hidden_dims=[256, 256],
            output_dim=128
        )

    def forward(self, user_features):
        # 各エンコーダで特徴量をエンコード
        user_id_emb = self.user_id_embedding(user_features['user_id'])
        demographic_emb = self.demographic_encoder(user_features['demographic'])
        behavioral_emb = self.behavioral_encoder(user_features['behavioral'])
        preference_emb = self.preference_encoder(user_features['preference'])
        session_emb = self.session_encoder(user_features['session_items'])
        context_emb = self.context_encoder(user_features['context'])

        # 特徴量を結合
        combined = torch.cat([
            user_id_emb,
            demographic_emb,
            behavioral_emb,
            preference_emb,
            session_emb,
            context_emb
        ], dim=-1)

        # 最終ユーザー埋め込みを生成
        user_embedding = self.fusion_layer(combined)

        return F.normalize(user_embedding, dim=-1)
```

### 3.2 アイテム特徴（目的地、ホテル、アクティビティ属性）

#### 3.2.1 アイテム特徴量カタログ

```yaml
item_feature_catalog:
  hotel_features:
    identity:
      hotel_id:
        type: categorical
        encoding: embedding (dim=128)
      name_embedding:
        type: text
        encoding: sentence_transformer (dim=384)

    attributes:
      star_rating:
        type: numerical
        range: [1, 5]
        transformation: none
      price_per_night:
        type: numerical
        transformation: log1p
      location:
        type: geolocation
        encoding: lat_lng + geohash
      total_rooms:
        type: numerical
        transformation: log1p

    amenities:
      amenities_vector:
        type: multi_hot
        encoding: multi_hot (dim=100)
        features:
          - wifi
          - pool
          - gym
          - restaurant
          - spa
          - parking
          - pet_friendly
          - ...

    quality_metrics:
      review_score:
        type: numerical
        range: [0, 5]
      review_count:
        type: numerical
        transformation: log1p
      response_rate:
        type: numerical
        range: [0, 1]
      photo_count:
        type: numerical
        transformation: log1p
      photo_quality_score:
        type: numerical
        range: [0, 1]

    popularity:
      booking_count_30d:
        type: numerical
        transformation: log1p
      view_count_7d:
        type: numerical
        transformation: log1p
      conversion_rate:
        type: numerical
        range: [0, 1]

  attraction_features:
    identity:
      attraction_id:
        type: categorical
        encoding: embedding (dim=128)
      name_embedding:
        type: text
        encoding: sentence_transformer (dim=384)
      description_embedding:
        type: text
        encoding: sentence_transformer (dim=384)

    attributes:
      category:
        type: categorical
        encoding: one_hot
        values: [museum, outdoor, entertainment, cultural, nature, ...]
      subcategory:
        type: categorical
        encoding: embedding (dim=32)
      duration_hours:
        type: numerical
        transformation: none
      price:
        type: numerical
        transformation: log1p
      location:
        type: geolocation
        encoding: lat_lng + geohash

    operational:
      operating_hours:
        type: structured
        encoding: time_slot_vector (dim=24)
      days_open:
        type: multi_hot
        encoding: multi_hot (dim=7)
      capacity:
        type: numerical
        transformation: log1p
      advance_booking_required:
        type: binary

    suitability:
      age_restriction:
        type: categorical
        values: [all_ages, 12+, 18+]
      accessibility_score:
        type: numerical
        range: [0, 1]
      physical_intensity:
        type: categorical
        values: [low, medium, high]
      weather_dependent:
        type: binary

    quality_metrics:
      review_score:
        type: numerical
        range: [0, 5]
      review_count:
        type: numerical
        transformation: log1p

  product_features:
    identity:
      product_id:
        type: categorical
        encoding: embedding (dim=128)
      name_embedding:
        type: text
        encoding: sentence_transformer (dim=384)

    attributes:
      category:
        type: hierarchical
        encoding: category_embedding (dim=64)
      brand:
        type: categorical
        encoding: embedding (dim=32)
      price:
        type: numerical
        transformation: log1p
      origin_country:
        type: categorical
        encoding: one_hot

    visual:
      image_embedding:
        type: image
        encoding: CLIP (dim=512)
      color_vector:
        type: multi_hot
        encoding: color_palette (dim=16)

    inventory:
      stock_level:
        type: numerical
        transformation: none
      size_availability:
        type: multi_hot
        encoding: size_vector (dim=10)

    popularity:
      sales_count_30d:
        type: numerical
        transformation: log1p
      view_count_7d:
        type: numerical
        transformation: log1p
```

#### 3.2.2 アイテム埋め込み生成

```python
# アイテム埋め込み生成アーキテクチャ
class ItemEmbeddingModel:
    """
    アイテム特徴量からアイテム埋め込みを生成
    """

    def __init__(self, config):
        self.item_id_embedding = Embedding(
            num_embeddings=config.num_items,
            embedding_dim=128
        )

        # テキスト埋め込み（事前学習済み）
        self.text_encoder = SentenceTransformer('paraphrase-multilingual-MiniLM-L12-v2')
        self.text_projection = Linear(384, 64)

        # 画像埋め込み（事前学習済み）
        self.image_encoder = CLIPModel.from_pretrained('openai/clip-vit-base-patch32')
        self.image_projection = Linear(512, 64)

        # 属性エンコーダ
        self.attribute_encoder = DenseEncoder(
            input_dim=config.attribute_dim,
            hidden_dims=[128, 64],
            output_dim=64
        )

        # カテゴリ階層エンコーダ
        self.category_encoder = HierarchicalEncoder(
            num_categories=config.num_categories,
            embedding_dim=32,
            num_levels=3
        )

        # 位置エンコーダ
        self.location_encoder = GeoEncoder(
            embedding_dim=32,
            geohash_precision=6
        )

        # 人気度エンコーダ
        self.popularity_encoder = DenseEncoder(
            input_dim=config.popularity_dim,
            hidden_dims=[32],
            output_dim=16
        )

        # 統合レイヤー
        self.fusion_layer = DenseEncoder(
            input_dim=128 + 64 + 64 + 64 + 32 + 32 + 16,  # 400
            hidden_dims=[256, 256],
            output_dim=128
        )

    def forward(self, item_features):
        # ID埋め込み
        item_id_emb = self.item_id_embedding(item_features['item_id'])

        # テキスト埋め込み
        text_emb = self.text_encoder.encode(item_features['text'])
        text_emb = self.text_projection(text_emb)

        # 画像埋め込み（利用可能な場合）
        if 'image' in item_features:
            image_emb = self.image_encoder.get_image_features(item_features['image'])
            image_emb = self.image_projection(image_emb)
        else:
            image_emb = torch.zeros(64)

        # 属性埋め込み
        attr_emb = self.attribute_encoder(item_features['attributes'])

        # カテゴリ埋め込み
        cat_emb = self.category_encoder(item_features['category_path'])

        # 位置埋め込み
        loc_emb = self.location_encoder(item_features['location'])

        # 人気度埋め込み
        pop_emb = self.popularity_encoder(item_features['popularity'])

        # 特徴量を結合
        combined = torch.cat([
            item_id_emb,
            text_emb,
            image_emb,
            attr_emb,
            cat_emb,
            loc_emb,
            pop_emb
        ], dim=-1)

        # 最終アイテム埋め込みを生成
        item_embedding = self.fusion_layer(combined)

        return F.normalize(item_embedding, dim=-1)
```

### 3.3 特徴量ストア設計（Feast、Tecton、社内構築）

#### 3.3.1 特徴量ストアアーキテクチャ

```yaml
feature_store_architecture:
  technology_selection:
    primary: Feast
    rationale:
      - オープンソース
      - GCP/AWS互換
      - Kubernetes native
      - 活発なコミュニティ
    alternatives_considered:
      tecton:
        pros: エンタープライズ機能、リアルタイム特徴量
        cons: コスト、ベンダーロックイン
      custom:
        pros: 完全なカスタマイズ
        cons: 開発・運用コスト

  architecture:
    offline_store:
      technology: BigQuery / Redshift
      purpose:
        - 履歴特徴量の保存
        - モデル学習データセット生成
        - バックフィル
      data_model:
        - timestamp: イベント時刻
        - entity_key: エンティティ識別子
        - features: 特徴量カラム
      retention: 2年

    online_store:
      technology: Redis Cluster
      purpose:
        - リアルタイム特徴量サービング
        - 低レイテンシ取得
      performance:
        p99_latency: 5ms
        throughput: 100,000 qps
      data_model:
        key: entity_type:entity_id:feature_view
        value: serialized_features
      ttl: 24時間（特徴量依存）

    registry:
      technology: GCS / S3
      purpose:
        - 特徴量定義の保存
        - バージョン管理
        - メタデータ管理
      contents:
        - feature_views
        - entities
        - data_sources
        - transformations

  data_flow:
    batch_ingestion:
      source: BigQuery / Data Warehouse
      frequency: hourly / daily
      process:
        1. データソースからデータ取得
        2. 特徴量変換適用
        3. オフラインストア書き込み
        4. オンラインストア更新（materialize）

    streaming_ingestion:
      source: Kafka
      latency: < 1秒
      process:
        1. イベント受信
        2. リアルタイム特徴量計算
        3. オンラインストア更新

    serving:
      online:
        client: Feature Server (gRPC/REST)
        request: entity_keys + feature_views
        response: feature_vectors
        latency_budget: 10ms

      offline:
        client: SDK (Python)
        request: entity_df + feature_views + timestamp
        response: training_dataset
```

#### 3.3.2 特徴量ビュー定義

```python
# Feast 特徴量ビュー定義
from feast import Entity, Feature, FeatureView, FileSource, ValueType
from feast.types import Float32, Int64, String
from datetime import timedelta

# エンティティ定義
user = Entity(
    name="user",
    value_type=ValueType.INT64,
    description="TripTrip user ID"
)

item = Entity(
    name="item",
    value_type=ValueType.INT64,
    description="Item ID (hotel, attraction, product)"
)

# データソース定義
user_behavioral_source = BigQuerySource(
    table="triptrip.features.user_behavioral_features",
    timestamp_field="event_timestamp",
    created_timestamp_column="created_timestamp"
)

user_preference_source = BigQuerySource(
    table="triptrip.features.user_preference_features",
    timestamp_field="event_timestamp",
    created_timestamp_column="created_timestamp"
)

item_attributes_source = BigQuerySource(
    table="triptrip.features.item_attributes",
    timestamp_field="event_timestamp",
    created_timestamp_column="created_timestamp"
)

# ユーザー行動特徴量ビュー
user_behavioral_fv = FeatureView(
    name="user_behavioral_features",
    entities=["user"],
    ttl=timedelta(hours=24),
    features=[
        Feature(name="total_bookings", dtype=Int64),
        Feature(name="bookings_last_30d", dtype=Int64),
        Feature(name="total_spend", dtype=Float32),
        Feature(name="average_order_value", dtype=Float32),
        Feature(name="booking_frequency", dtype=Float32),
        Feature(name="last_booking_days_ago", dtype=Int64),
        Feature(name="view_count_7d", dtype=Int64),
        Feature(name="search_count_7d", dtype=Int64),
        Feature(name="cart_abandonment_rate", dtype=Float32),
    ],
    source=user_behavioral_source,
    online=True,
    tags={"owner": "ml-team", "category": "user"}
)

# ユーザー嗜好特徴量ビュー
user_preference_fv = FeatureView(
    name="user_preference_features",
    entities=["user"],
    ttl=timedelta(days=7),
    features=[
        Feature(name="preferred_hotel_star", dtype=Float32),
        Feature(name="preferred_price_segment", dtype=String),
        Feature(name="preferred_categories", dtype=String),  # JSON encoded
        Feature(name="destination_embedding", dtype=String),  # JSON encoded vector
        Feature(name="cuisine_preferences", dtype=String),
        Feature(name="adventure_level", dtype=Float32),
    ],
    source=user_preference_source,
    online=True,
    tags={"owner": "ml-team", "category": "user"}
)

# アイテム属性特徴量ビュー
item_attributes_fv = FeatureView(
    name="item_attributes",
    entities=["item"],
    ttl=timedelta(hours=6),
    features=[
        Feature(name="item_type", dtype=String),
        Feature(name="category", dtype=String),
        Feature(name="price", dtype=Float32),
        Feature(name="location_lat", dtype=Float32),
        Feature(name="location_lng", dtype=Float32),
        Feature(name="review_score", dtype=Float32),
        Feature(name="review_count", dtype=Int64),
        Feature(name="popularity_score", dtype=Float32),
        Feature(name="item_embedding", dtype=String),  # JSON encoded vector
    ],
    source=item_attributes_source,
    online=True,
    tags={"owner": "ml-team", "category": "item"}
)

# リアルタイムセッション特徴量（ストリーミング）
session_features_fv = FeatureView(
    name="session_features",
    entities=["user"],
    ttl=timedelta(minutes=30),
    features=[
        Feature(name="session_duration_sec", dtype=Int64),
        Feature(name="pages_viewed", dtype=Int64),
        Feature(name="items_viewed", dtype=String),  # JSON encoded list
        Feature(name="searches", dtype=String),  # JSON encoded list
        Feature(name="cart_items", dtype=String),  # JSON encoded list
        Feature(name="last_viewed_category", dtype=String),
    ],
    stream_source=KafkaSource(
        topic="user_session_events",
        timestamp_field="event_timestamp",
        bootstrap_servers="kafka:9092"
    ),
    online=True,
    tags={"owner": "ml-team", "category": "real_time"}
)
```

---

## 第4章：推論基盤アーキテクチャ

### 4.1 オンライン推論サービス設計

#### 4.1.1 推論サービスアーキテクチャ

```yaml
inference_architecture:
  overview:
    pattern: マイクロサービス + モデルサービング分離
    components:
      - API Gateway
      - Recommendation API Service
      - Feature Service
      - Model Serving Infrastructure
      - Reranking Service
      - Cache Layer

  recommendation_api_service:
    technology: Python FastAPI / Go
    responsibilities:
      - リクエスト受付・バリデーション
      - 特徴量取得オーケストレーション
      - モデル推論呼び出し
      - 結果の後処理・リランキング
      - レスポンス生成
      - ロギング・メトリクス

    endpoints:
      get_recommendations:
        path: /api/v1/recommendations
        method: POST
        request:
          user_id: int
          context:
            page_type: string
            item_ids: list[int]  # 現在閲覧中のアイテム
            filters: object
          num_items: int (default: 10)
          recommendation_type: string
        response:
          recommendations:
            - item_id: int
              score: float
              explanation: string
          metadata:
            model_version: string
            latency_ms: int

      get_similar_items:
        path: /api/v1/similar-items/{item_id}
        method: GET
        request:
          item_id: path_param
          num_items: int (default: 10)
        response:
          similar_items:
            - item_id: int
              similarity_score: float

    scaling:
      horizontal: true
      min_replicas: 3
      max_replicas: 50
      scaling_metric: latency_p95 < 40ms
      cpu_threshold: 70%

  inference_flow:
    step_1_request_validation:
      actions:
        - リクエストスキーマ検証
        - ユーザー認証確認
        - レート制限チェック
      latency_budget: 2ms

    step_2_feature_retrieval:
      actions:
        - Feature Store からユーザー特徴量取得
        - Feature Store からアイテム候補特徴量取得
        - リアルタイム特徴量計算
      latency_budget: 10ms
      parallelization: true

    step_3_candidate_generation:
      actions:
        - Two-Tower モデルでスコアリング
        - 上位N件の候補抽出
      latency_budget: 15ms
      candidates: 1000 → 100

    step_4_reranking:
      actions:
        - 詳細特徴量でリランキング
        - ビジネスルール適用
        - 多様性調整
      latency_budget: 10ms
      candidates: 100 → 10

    step_5_post_processing:
      actions:
        - フィルタリング（在庫、ブラックリスト）
        - 説明生成
        - レスポンス構築
      latency_budget: 3ms

    total_latency_budget: 40ms (p95)
```

#### 4.1.2 推論サービス実装

```python
# Recommendation API Service 実装
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional
import asyncio
import time

app = FastAPI()

class RecommendationRequest(BaseModel):
    user_id: int
    context: dict
    num_items: int = 10
    recommendation_type: str = "personalized"

class RecommendationItem(BaseModel):
    item_id: int
    score: float
    explanation: Optional[str]

class RecommendationResponse(BaseModel):
    recommendations: List[RecommendationItem]
    metadata: dict

class RecommendationService:
    def __init__(self):
        self.feature_store = FeatureStoreClient()
        self.model_client = ModelServingClient()
        self.reranker = RerankerService()
        self.cache = CacheClient()

    async def get_recommendations(
        self,
        request: RecommendationRequest
    ) -> RecommendationResponse:
        start_time = time.time()

        # Step 1: キャッシュチェック
        cache_key = self._build_cache_key(request)
        cached = await self.cache.get(cache_key)
        if cached:
            return cached

        # Step 2: 特徴量取得（並列実行）
        user_features, candidate_features = await asyncio.gather(
            self._get_user_features(request.user_id, request.context),
            self._get_candidate_features(request.context)
        )

        # Step 3: 候補生成（Two-Tower Model）
        candidates = await self._generate_candidates(
            user_features,
            candidate_features,
            num_candidates=100
        )

        # Step 4: リランキング
        reranked = await self.reranker.rerank(
            user_features,
            candidates,
            context=request.context,
            num_items=request.num_items
        )

        # Step 5: 後処理
        recommendations = self._post_process(
            reranked,
            request
        )

        # メトリクス記録
        latency_ms = (time.time() - start_time) * 1000
        self._log_metrics(request, recommendations, latency_ms)

        response = RecommendationResponse(
            recommendations=recommendations,
            metadata={
                "model_version": self.model_client.model_version,
                "latency_ms": int(latency_ms),
                "num_candidates": len(candidates)
            }
        )

        # キャッシュ保存
        await self.cache.set(cache_key, response, ttl=300)

        return response

    async def _get_user_features(
        self,
        user_id: int,
        context: dict
    ) -> dict:
        """ユーザー特徴量取得"""
        # オフライン特徴量
        offline_features = await self.feature_store.get_online_features(
            entity_rows=[{"user": user_id}],
            feature_views=[
                "user_behavioral_features",
                "user_preference_features"
            ]
        )

        # リアルタイム特徴量
        session_features = await self.feature_store.get_online_features(
            entity_rows=[{"user": user_id}],
            feature_views=["session_features"]
        )

        # コンテキスト特徴量
        context_features = self._compute_context_features(context)

        return {
            **offline_features,
            **session_features,
            **context_features
        }

    async def _generate_candidates(
        self,
        user_features: dict,
        candidate_features: dict,
        num_candidates: int
    ) -> List[dict]:
        """Two-Tower モデルで候補生成"""
        # ユーザー埋め込み計算
        user_embedding = await self.model_client.get_user_embedding(
            user_features
        )

        # ANN検索で類似アイテム取得
        candidates = await self.model_client.ann_search(
            query_vector=user_embedding,
            num_results=num_candidates,
            index_name="item_embeddings"
        )

        return candidates

@app.post("/api/v1/recommendations")
async def get_recommendations(request: RecommendationRequest):
    service = RecommendationService()
    return await service.get_recommendations(request)
```

### 4.2 モデルサービング（TensorFlow Serving、Triton、SageMaker）

#### 4.2.1 モデルサービング基盤選定

```yaml
model_serving_selection:
  evaluation_criteria:
    - 推論レイテンシ
    - スループット
    - GPU対応
    - モデル形式サポート
    - 運用性
    - コスト

  options_comparison:
    tensorflow_serving:
      pros:
        - TensorFlow/Kerasモデルに最適化
        - 低レイテンシ
        - バッチ推論対応
        - A/Bテスト対応
      cons:
        - TensorFlow限定
        - 運用複雑性
      use_case: TensorFlowモデルのサービング

    triton_inference_server:
      pros:
        - マルチフレームワーク（TensorFlow、PyTorch、ONNX）
        - 動的バッチング
        - 高度なGPU最適化
        - アンサンブルモデル対応
      cons:
        - 設定複雑
        - リソース要件高
      use_case: 複数モデルの統合サービング

    sagemaker_endpoints:
      pros:
        - フルマネージド
        - オートスケーリング
        - AWSエコシステム統合
        - MLOps統合
      cons:
        - AWSロックイン
        - コスト高
        - カスタマイズ制限
      use_case: AWSベース運用

    ray_serve:
      pros:
        - Pythonネイティブ
        - 柔軟なパイプライン
        - スケーラブル
        - オープンソース
      cons:
        - 成熟度
        - GPU最適化
      use_case: 複雑な推論パイプライン

  selected_architecture:
    primary: Triton Inference Server
    rationale:
      - PyTorch/TensorFlow両方のモデルをサポート
      - 高度なGPU最適化
      - 動的バッチングによる効率化
      - アンサンブル推論対応

    secondary: TensorFlow Serving
    rationale:
      - 一部のTensorFlowモデル用
      - シンプルな単一モデルサービング

    embedding_index: Faiss (Facebook AI Similarity Search)
    rationale:
      - 高速なANN検索
      - GPU対応
      - スケーラブル
```

#### 4.2.2 Triton Inference Server構成

```yaml
triton_configuration:
  deployment:
    platform: Kubernetes
    replicas: 3-10 (auto-scaling)
    resources:
      cpu: 4
      memory: 16Gi
      gpu: nvidia-t4 (1 per pod)

  model_repository:
    structure:
      models/
        user_tower/
          1/
            model.savedmodel/
          config.pbtxt
        item_tower/
          1/
            model.onnx
          config.pbtxt
        reranker/
          1/
            model.pt
          config.pbtxt
        ensemble_pipeline/
          1/
          config.pbtxt

  model_configs:
    user_tower:
      config: |
        name: "user_tower"
        platform: "tensorflow_savedmodel"
        max_batch_size: 256
        input [
          {
            name: "user_features"
            data_type: TYPE_FP32
            dims: [ 336 ]
          }
        ]
        output [
          {
            name: "user_embedding"
            data_type: TYPE_FP32
            dims: [ 128 ]
          }
        ]
        instance_group [
          {
            count: 2
            kind: KIND_GPU
          }
        ]
        dynamic_batching {
          preferred_batch_size: [ 32, 64, 128 ]
          max_queue_delay_microseconds: 5000
        }

    item_tower:
      config: |
        name: "item_tower"
        platform: "onnxruntime_onnx"
        max_batch_size: 1024
        input [
          {
            name: "item_features"
            data_type: TYPE_FP32
            dims: [ 400 ]
          }
        ]
        output [
          {
            name: "item_embedding"
            data_type: TYPE_FP32
            dims: [ 128 ]
          }
        ]
        instance_group [
          {
            count: 2
            kind: KIND_GPU
          }
        ]
        dynamic_batching {
          preferred_batch_size: [ 64, 128, 256, 512 ]
          max_queue_delay_microseconds: 3000
        }

    reranker:
      config: |
        name: "reranker"
        platform: "pytorch_libtorch"
        max_batch_size: 128
        input [
          {
            name: "user_embedding"
            data_type: TYPE_FP32
            dims: [ 128 ]
          },
          {
            name: "item_embeddings"
            data_type: TYPE_FP32
            dims: [ -1, 128 ]  # variable length
          },
          {
            name: "context_features"
            data_type: TYPE_FP32
            dims: [ 64 ]
          }
        ]
        output [
          {
            name: "scores"
            data_type: TYPE_FP32
            dims: [ -1 ]
          }
        ]
        instance_group [
          {
            count: 1
            kind: KIND_GPU
          }
        ]

  ensemble_pipeline:
    config: |
      name: "ensemble_pipeline"
      platform: "ensemble"
      max_batch_size: 0
      input [
        {
          name: "user_features"
          data_type: TYPE_FP32
          dims: [ 336 ]
        },
        {
          name: "candidate_item_ids"
          data_type: TYPE_INT64
          dims: [ -1 ]
        }
      ]
      output [
        {
          name: "ranked_item_ids"
          data_type: TYPE_INT64
          dims: [ -1 ]
        },
        {
          name: "scores"
          data_type: TYPE_FP32
          dims: [ -1 ]
        }
      ]
      ensemble_scheduling {
        step [
          {
            model_name: "user_tower"
            model_version: -1
            input_map {
              key: "user_features"
              value: "user_features"
            }
            output_map {
              key: "user_embedding"
              value: "user_embedding"
            }
          },
          {
            model_name: "item_tower"
            model_version: -1
            input_map {
              key: "item_features"
              value: "candidate_item_features"
            }
            output_map {
              key: "item_embedding"
              value: "item_embeddings"
            }
          },
          {
            model_name: "reranker"
            model_version: -1
            input_map {
              key: "user_embedding"
              value: "user_embedding"
            }
            input_map {
              key: "item_embeddings"
              value: "item_embeddings"
            }
            output_map {
              key: "scores"
              value: "scores"
            }
          }
        ]
      }
```

### 4.3 キャッシング戦略とレイテンシ最適化

#### 4.3.1 多層キャッシング戦略

```yaml
caching_strategy:
  layers:
    l1_application_cache:
      technology: In-memory (LRU Cache)
      location: Recommendation Service プロセス内
      purpose:
        - 最頻アクセスデータのローカルキャッシュ
        - Feature Store RTT削減
      ttl: 60秒
      size: 1GB per instance
      hit_rate_target: 40%
      data_cached:
        - 人気アイテムの特徴量
        - グローバル推奨（非パーソナライズ）
        - 静的設定

    l2_distributed_cache:
      technology: Redis Cluster
      location: Kubernetes内
      purpose:
        - ユーザー特徴量キャッシュ
        - 推奨結果キャッシュ
        - セッション状態
      ttl:
        user_features: 15分
        recommendations: 5分
        session_state: 30分
      size: 50GB cluster
      hit_rate_target: 70%
      data_cached:
        - ユーザー特徴量ベクトル
        - 計算済み推奨リスト
        - セッション内行動履歴

    l3_embedding_cache:
      technology: Faiss Index (GPU)
      location: Model Serving内
      purpose:
        - アイテム埋め込みのインメモリ保持
        - 高速ANN検索
      refresh: 日次
      size: 全アイテム埋め込み（~10GB）
      hit_rate_target: 100%

  cache_invalidation:
    strategies:
      time_based:
        description: TTLによる自動失効
        use_case: ユーザー特徴量、推奨結果

      event_based:
        description: イベント駆動の即座無効化
        triggers:
          - ユーザーアクション（予約、購入）
          - アイテム更新（価格、在庫）
          - モデル更新
        mechanism: Redis Pub/Sub

      version_based:
        description: バージョンキーによる一括無効化
        use_case: モデル更新時の全キャッシュクリア

  cache_warming:
    strategy:
      - 人気ユーザーの事前計算
      - 人気アイテムの特徴量プリロード
      - 新モデルデプロイ前のウォームアップ
    timing:
      - 毎日深夜2-4時（低トラフィック時）
      - モデルデプロイ15分前

latency_optimization:
  techniques:
    parallel_fetching:
      description: 特徴量取得の並列化
      implementation:
        - asyncio.gather() for concurrent requests
        - gRPC bidirectional streaming
      impact: -30% latency

    batch_inference:
      description: 複数リクエストの統合推論
      implementation:
        - Triton dynamic batching
        - Request coalescing
      impact: +50% throughput, -20% per-request latency

    model_optimization:
      description: モデルの軽量化・高速化
      techniques:
        - 量子化（INT8）
        - プルーニング
        - 知識蒸留
        - TensorRT最適化
      impact: -40% inference time

    connection_pooling:
      description: コネクションの再利用
      targets:
        - Redis connections
        - gRPC channels
        - Feature Store clients
      impact: -10% latency overhead

    geographic_optimization:
      description: エッジデプロイメント
      strategy:
        - 主要リージョンにサービス配置
        - CDN活用（静的推奨）
      impact: -50% network latency

  latency_budget_breakdown:
    total: 50ms (p99)
    components:
      network_ingress: 2ms
      request_parsing: 1ms
      cache_check: 2ms
      feature_retrieval: 10ms
      model_inference: 20ms
      reranking: 8ms
      post_processing: 3ms
      response_serialization: 2ms
      network_egress: 2ms
```

---

## 第5章：MLOps & 継続的改善

### 5.1 モデル管理とバージョニング（MLflow、Kubeflow）

#### 5.1.1 MLOps基盤アーキテクチャ

```yaml
mlops_architecture:
  components:
    experiment_tracking:
      technology: MLflow
      purpose:
        - 実験パラメータ記録
        - メトリクス追跡
        - アーティファクト管理
      integration:
        - 学習スクリプト
        - ノートブック
        - CI/CDパイプライン

    model_registry:
      technology: MLflow Model Registry
      purpose:
        - モデルバージョン管理
        - ステージ管理（Staging/Production）
        - モデルメタデータ
      stages:
        - None: 未登録
        - Staging: 検証中
        - Production: 本番稼働
        - Archived: アーカイブ

    pipeline_orchestration:
      technology: Kubeflow Pipelines
      purpose:
        - 学習パイプライン自動化
        - スケジュール実行
        - 依存関係管理
      pipelines:
        - 日次モデル再学習
        - 特徴量パイプライン
        - 評価パイプライン

    feature_store:
      technology: Feast
      integration:
        - オフライン学習データ生成
        - オンライン特徴量サービング

    serving_infrastructure:
      technology: Triton + Kubernetes
      integration:
        - モデルデプロイメント自動化
        - A/Bテスト
        - カナリアリリース

  workflow:
    development:
      1_experiment:
        tools: Jupyter, MLflow
        activities:
          - データ探索
          - 特徴量エンジニアリング
          - モデル試行
          - ハイパーパラメータ調整

      2_validation:
        tools: MLflow, Kubeflow
        activities:
          - オフライン評価
          - モデル比較
          - バイアスチェック
          - 説明可能性分析

      3_registration:
        tools: MLflow Model Registry
        activities:
          - モデル登録
          - メタデータ付与
          - Stagingステージ昇格

    deployment:
      4_staging_test:
        tools: Kubeflow, Triton
        activities:
          - シャドウモード推論
          - レイテンシ検証
          - リソース使用量確認

      5_production_deploy:
        tools: ArgoCD, Triton
        activities:
          - カナリアリリース
          - トラフィック段階移行
          - ロールバック準備

      6_monitoring:
        tools: Prometheus, Grafana
        activities:
          - パフォーマンス監視
          - モデルドリフト検出
          - ビジネスKPI追跡
```

#### 5.1.2 モデルバージョニング戦略

```yaml
model_versioning:
  naming_convention:
    pattern: "{model_name}-v{major}.{minor}.{patch}-{experiment_id}"
    examples:
      - two_tower_recommendation-v1.2.3-exp123
      - item_similarity-v2.0.0-exp456

  versioning_rules:
    major:
      trigger:
        - アーキテクチャ変更
        - 入出力スキーマ変更
        - 破壊的変更
      process:
        - 完全な再検証
        - 段階的ロールアウト
        - 長期並行運用

    minor:
      trigger:
        - 新特徴量追加
        - ハイパーパラメータ変更
        - 学習データ更新
      process:
        - A/Bテスト
        - カナリアリリース

    patch:
      trigger:
        - バグ修正
        - 軽微な改善
        - 定期再学習
      process:
        - 自動デプロイ（テスト通過時）

  model_metadata:
    required_fields:
      - model_id: 一意識別子
      - version: バージョン番号
      - created_at: 作成日時
      - created_by: 作成者
      - training_data_version: 学習データバージョン
      - feature_schema_version: 特徴量スキーマバージョン
      - metrics: オフライン評価メトリクス
      - hyperparameters: ハイパーパラメータ

    optional_fields:
      - description: モデル説明
      - changelog: 変更履歴
      - dependencies: 依存関係
      - resource_requirements: リソース要件
      - serving_config: サービング設定

  lifecycle_management:
    retention:
      production: 無期限
      staging: 30日
      archived: 90日
      experimental: 7日

    cleanup:
      - 古いバージョンの自動アーカイブ
      - 未使用アーティファクトの削除
      - ストレージコスト最適化
```

### 5.2 A/Bテスト基盤設計

#### 5.2.1 A/Bテストアーキテクチャ

```yaml
ab_testing_architecture:
  components:
    experiment_service:
      responsibility:
        - 実験定義管理
        - ユーザー割り当て
        - バリアント設定
      technology: Custom service + Redis

    assignment_service:
      responsibility:
        - 一貫したユーザー割り当て
        - トラフィック分割
        - 除外ルール適用
      algorithm: Deterministic hashing (user_id + experiment_id)

    metrics_collection:
      responsibility:
        - イベント収集
        - メトリクス計算
        - 統計分析
      technology: Kafka + ClickHouse + Custom analytics

    results_dashboard:
      responsibility:
        - 実験結果可視化
        - 統計的有意性計算
        - 意思決定支援
      technology: Custom dashboard + Metabase

  experiment_definition:
    schema:
      experiment_id: string
      name: string
      description: string
      hypothesis: string
      variants:
        - id: control
          model_version: v1.2.3
          traffic_percentage: 50
        - id: treatment
          model_version: v1.3.0
          traffic_percentage: 50
      targeting:
        user_segments: [all]
        platforms: [ios, android, web]
        regions: [jp]
      metrics:
        primary: conversion_rate
        secondary: [ctr, revenue_per_user, session_duration]
        guardrail: [latency_p99, error_rate]
      duration:
        min_days: 7
        max_days: 28
        min_samples: 10000
      status: draft | running | paused | completed

  user_assignment:
    algorithm:
      method: consistent_hashing
      implementation: |
        def assign_variant(user_id, experiment_id, variants):
            hash_input = f"{user_id}:{experiment_id}"
            hash_value = hashlib.md5(hash_input.encode()).hexdigest()
            bucket = int(hash_value, 16) % 100

            cumulative = 0
            for variant in variants:
                cumulative += variant.traffic_percentage
                if bucket < cumulative:
                    return variant.id
            return variants[-1].id

    properties:
      - 一貫性: 同一ユーザーは常に同じバリアント
      - 均等分布: トラフィック分割の正確性
      - 独立性: 複数実験間の干渉なし

  statistical_analysis:
    methodology:
      primary: Frequentist (T-test, Chi-squared)
      secondary: Bayesian inference
      correction: Bonferroni (multiple comparisons)

    parameters:
      significance_level: 0.05
      power: 0.80
      minimum_detectable_effect: 5%

    sample_size_calculation: |
      n = 2 * ((z_alpha + z_beta) / delta)^2 * p * (1 - p)
      where:
        z_alpha = 1.96 (for 95% confidence)
        z_beta = 0.84 (for 80% power)
        delta = minimum detectable effect
        p = baseline conversion rate
```

#### 5.2.2 A/Bテスト実装

```python
# A/Bテストサービス実装
class ABTestingService:
    def __init__(self):
        self.experiment_store = ExperimentStore()
        self.assignment_cache = RedisClient()
        self.metrics_collector = MetricsCollector()

    def get_variant(
        self,
        user_id: int,
        experiment_id: str
    ) -> Optional[str]:
        """ユーザーのバリアントを取得"""
        # キャッシュチェック
        cache_key = f"ab:{experiment_id}:{user_id}"
        cached = self.assignment_cache.get(cache_key)
        if cached:
            return cached

        # 実験設定取得
        experiment = self.experiment_store.get(experiment_id)
        if not experiment or experiment.status != "running":
            return None

        # ターゲティングチェック
        if not self._check_targeting(user_id, experiment):
            return None

        # バリアント割り当て
        variant = self._assign_variant(user_id, experiment)

        # キャッシュ保存
        self.assignment_cache.set(
            cache_key,
            variant,
            ttl=experiment.duration_days * 86400
        )

        # 割り当てログ
        self.metrics_collector.log_assignment(
            experiment_id=experiment_id,
            user_id=user_id,
            variant=variant
        )

        return variant

    def _assign_variant(
        self,
        user_id: int,
        experiment: Experiment
    ) -> str:
        """決定論的ハッシュによるバリアント割り当て"""
        hash_input = f"{user_id}:{experiment.experiment_id}"
        hash_value = hashlib.md5(hash_input.encode()).hexdigest()
        bucket = int(hash_value, 16) % 100

        cumulative = 0
        for variant in experiment.variants:
            cumulative += variant.traffic_percentage
            if bucket < cumulative:
                return variant.id

        return experiment.variants[-1].id

    def log_conversion(
        self,
        user_id: int,
        experiment_id: str,
        event_type: str,
        value: float = 1.0
    ):
        """コンバージョンイベントのログ"""
        variant = self.get_variant(user_id, experiment_id)
        if variant:
            self.metrics_collector.log_conversion(
                experiment_id=experiment_id,
                user_id=user_id,
                variant=variant,
                event_type=event_type,
                value=value
            )

    def get_results(
        self,
        experiment_id: str
    ) -> ExperimentResults:
        """実験結果の取得と統計分析"""
        experiment = self.experiment_store.get(experiment_id)
        metrics = self.metrics_collector.get_metrics(experiment_id)

        results = ExperimentResults(experiment_id=experiment_id)

        for metric_name in experiment.metrics.all:
            control_data = metrics.get_variant_data("control", metric_name)
            treatment_data = metrics.get_variant_data("treatment", metric_name)

            # 統計検定
            stat_result = self._perform_statistical_test(
                control_data,
                treatment_data,
                metric_name
            )

            results.add_metric_result(metric_name, stat_result)

        return results

    def _perform_statistical_test(
        self,
        control: MetricData,
        treatment: MetricData,
        metric_name: str
    ) -> StatisticalResult:
        """統計検定の実行"""
        if metric_name in ["conversion_rate", "ctr"]:
            # 二項検定
            stat, p_value = proportions_ztest(
                [treatment.conversions, control.conversions],
                [treatment.samples, control.samples]
            )
        else:
            # t検定
            stat, p_value = ttest_ind(
                treatment.values,
                control.values
            )

        # 効果サイズ計算
        effect_size = (treatment.mean - control.mean) / control.mean

        # 信頼区間計算
        ci_lower, ci_upper = self._calculate_confidence_interval(
            control,
            treatment
        )

        return StatisticalResult(
            control_mean=control.mean,
            treatment_mean=treatment.mean,
            effect_size=effect_size,
            p_value=p_value,
            is_significant=p_value < 0.05,
            ci_lower=ci_lower,
            ci_upper=ci_upper,
            samples_control=control.samples,
            samples_treatment=treatment.samples
        )
```

### 5.3 モデル監視とドリフト検出

#### 5.3.1 監視アーキテクチャ

```yaml
monitoring_architecture:
  layers:
    infrastructure_monitoring:
      tools: Prometheus + Grafana
      metrics:
        - CPU/Memory/GPU使用率
        - ネットワークI/O
        - ディスク使用量
        - コンテナ健全性

    service_monitoring:
      tools: Prometheus + Grafana + PagerDuty
      metrics:
        - リクエストレート
        - レイテンシ (p50, p95, p99)
        - エラーレート
        - スループット

    model_monitoring:
      tools: Custom + Evidently AI
      metrics:
        - 予測分布
        - 特徴量分布
        - モデルパフォーマンス
        - データ品質

    business_monitoring:
      tools: Metabase + Custom Dashboard
      metrics:
        - CTR/CVR
        - 収益
        - ユーザーエンゲージメント
        - 推奨多様性

  model_specific_metrics:
    prediction_metrics:
      prediction_distribution:
        description: 予測スコアの分布
        alert: 分布の大幅な変化
        threshold: KL divergence > 0.1

      prediction_volume:
        description: 予測リクエスト数
        alert: 急激な増減
        threshold: ±50% from baseline

      null_prediction_rate:
        description: 空の推奨率
        alert: 閾値超過
        threshold: > 5%

    feature_metrics:
      feature_distribution:
        description: 入力特徴量の分布
        alert: 分布シフト検出
        method: PSI (Population Stability Index)
        threshold: PSI > 0.2

      missing_feature_rate:
        description: 欠損特徴量率
        alert: 閾値超過
        threshold: > 1%

      feature_correlation:
        description: 特徴量間相関
        alert: 相関変化
        threshold: |correlation_change| > 0.3

    performance_metrics:
      offline_performance:
        description: オフライン評価指標
        metrics:
          - NDCG@10
          - Precision@K
          - Recall@K
          - MAP
        alert: 定期評価での低下
        threshold: -5% from baseline

      online_performance:
        description: オンラインビジネス指標
        metrics:
          - CTR
          - Conversion Rate
          - Revenue per User
        alert: 継続的な低下
        threshold: -10% for 24h

drift_detection:
  types:
    data_drift:
      description: 入力データ分布の変化
      detection_methods:
        - PSI (Population Stability Index)
        - KS Test (Kolmogorov-Smirnov)
        - JS Divergence
      monitoring_frequency: hourly
      action:
        warning: PSI 0.1-0.2
        critical: PSI > 0.2

    concept_drift:
      description: 入力と出力の関係変化
      detection_methods:
        - パフォーマンス指標の低下監視
        - Residual分析
        - ADWIN (Adaptive Windowing)
      monitoring_frequency: daily
      action:
        warning: Performance -5%
        critical: Performance -10%

    prediction_drift:
      description: 予測分布の変化
      detection_methods:
        - KL Divergence
        - Wasserstein Distance
        - Chi-squared Test
      monitoring_frequency: hourly
      action:
        warning: KL > 0.05
        critical: KL > 0.1

  response_procedures:
    automated:
      - アラート発報
      - ダッシュボード更新
      - インシデントチケット作成

    semi_automated:
      - モデル再学習トリガー
      - フォールバック切り替え
      - トラフィックシフト

    manual:
      - 根本原因分析
      - データ調査
      - モデル改善
```

#### 5.3.2 監視ダッシュボード設計

```yaml
monitoring_dashboards:
  overview_dashboard:
    name: Recommendation System Overview
    refresh: 30秒
    panels:
      - title: Request Rate
        type: graph
        metric: sum(rate(recommendation_requests_total[5m]))

      - title: Latency P99
        type: graph
        metric: histogram_quantile(0.99, recommendation_latency_bucket)

      - title: Error Rate
        type: graph
        metric: sum(rate(recommendation_errors_total[5m]))

      - title: Cache Hit Rate
        type: gauge
        metric: sum(cache_hits) / sum(cache_requests)

      - title: Model Version Distribution
        type: pie
        metric: recommendation_model_version

      - title: Active Experiments
        type: stat
        metric: count(active_experiments)

  model_health_dashboard:
    name: Model Health Monitoring
    refresh: 1分
    panels:
      - title: Prediction Score Distribution
        type: histogram
        metric: recommendation_score_distribution

      - title: Feature Drift (PSI)
        type: heatmap
        metric: feature_psi_score
        alert_threshold: 0.2

      - title: Concept Drift Indicator
        type: graph
        metric: concept_drift_score

      - title: Null Recommendation Rate
        type: graph
        metric: null_recommendation_rate
        alert_threshold: 0.05

      - title: Top Features by Importance
        type: bar
        metric: feature_importance_score

      - title: Model Staleness
        type: stat
        metric: hours_since_model_update

  business_metrics_dashboard:
    name: Recommendation Business Impact
    refresh: 5分
    panels:
      - title: Recommendation CTR
        type: graph
        metric: recommendation_clicks / recommendation_impressions
        comparison: previous_week

      - title: Conversion Rate (from Recommendation)
        type: graph
        metric: recommendation_conversions / recommendation_clicks
        comparison: previous_week

      - title: Revenue from Recommendations
        type: graph
        metric: sum(recommendation_revenue)
        comparison: previous_week

      - title: Recommendation Diversity
        type: gauge
        metric: intra_list_diversity
        target: 0.6

      - title: Catalog Coverage
        type: gauge
        metric: recommended_items / total_items
        target: 0.8

      - title: A/B Test Results
        type: table
        data: active_experiment_results
```

---

## 第6章：実装ロードマップ & フェーズ計画

### 6.1 フェーズ別実装計画

```yaml
implementation_roadmap:
  phase_1:
    name: Foundation
    duration: 3ヶ月
    objectives:
      - 基本推奨機能の実装
      - MLインフラ基盤構築
      - データパイプライン整備

    deliverables:
      infrastructure:
        - Feature Store (Feast) セットアップ
        - MLflow 実験追跡環境
        - Triton Inference Server デプロイ
        - Redis キャッシュクラスター

      models:
        - Popularity-based 推奨（フォールバック）
        - Item-Item 類似度モデル
        - 基本 Two-Tower モデル（v1）

      features:
        - ユーザー行動履歴特徴量
        - アイテム属性特徴量
        - 基本コンテキスト特徴量

      integration:
        - Recommendation API エンドポイント
        - Flutter SDK 統合
        - イベントトラッキング

    success_criteria:
      - レイテンシ P99 < 100ms
      - CTR > 3%
      - システム稼働率 > 99%

  phase_2:
    name: Personalization
    duration: 3ヶ月
    objectives:
      - パーソナライズ機能強化
      - MLOps 成熟度向上
      - A/B テスト基盤

    deliverables:
      models:
        - Two-Tower モデル v2（コンテキスト強化）
        - Session-based 推奨モデル
        - Reranking モデル

      features:
        - リアルタイムセッション特徴量
        - 詳細コンテキスト特徴量
        - ユーザー嗜好モデリング

      mlops:
        - Kubeflow Pipelines 自動化
        - A/B テスト基盤
        - モデル監視ダッシュボード

      integration:
        - 検索結果パーソナライゼーション
        - ホーム画面推奨最適化
        - プッシュ通知推奨

    success_criteria:
      - レイテンシ P99 < 50ms
      - CTR > 8%
      - CVR > 5%

  phase_3:
    name: Advanced Intelligence
    duration: 4ヶ月
    objectives:
      - 高度なML機能
      - マルチモーダル推奨
      - グローバルスケール

    deliverables:
      models:
        - Graph Neural Network 推奨
        - マルチモーダル（画像+テキスト）
        - 動的価格最適化モデル
        - 旅程最適化モデル

      features:
        - 画像埋め込み統合
        - 地理空間特徴量
        - 時系列パターン

      advanced:
        - リアルタイム学習
        - Federated Learning 調査
        - 説明可能AI

      scale:
        - マルチリージョンデプロイ
        - エッジ推論
        - キャパシティプランニング

    success_criteria:
      - レイテンシ P99 < 30ms
      - CTR > 15%
      - CVR > 8%
      - 推奨からの収益 > 50%

resource_allocation:
  phase_1:
    ml_engineers: 2
    backend_engineers: 2
    data_engineers: 1
    total_headcount: 5

  phase_2:
    ml_engineers: 3
    backend_engineers: 2
    data_engineers: 2
    platform_engineer: 1
    total_headcount: 8

  phase_3:
    ml_engineers: 4
    backend_engineers: 3
    data_engineers: 2
    platform_engineer: 2
    ml_researcher: 1
    total_headcount: 12
```

### 6.2 リスクと対策

```yaml
risks_and_mitigations:
  technical_risks:
    cold_start:
      description: 新規ユーザー/アイテムへの推奨困難
      probability: High
      impact: Medium
      mitigation:
        - Popularity-based フォールバック
        - コンテンツベース補完
        - オンボーディング時の嗜好収集
        - デモグラフィックベース推奨

    model_degradation:
      description: モデルパフォーマンスの経時劣化
      probability: Medium
      impact: High
      mitigation:
        - 継続的モニタリング
        - 自動再学習パイプライン
        - ドリフト検出アラート
        - フォールバック機構

    latency_spike:
      description: 推論レイテンシの急増
      probability: Medium
      impact: High
      mitigation:
        - キャッシュ多層化
        - サーキットブレーカー
        - オートスケーリング
        - 軽量モデルへのフォールバック

    data_quality:
      description: 入力データ品質の低下
      probability: Medium
      impact: Medium
      mitigation:
        - データ検証パイプライン
        - 異常検出
        - データ品質ダッシュボード
        - グレースフルデグラデーション

  organizational_risks:
    talent:
      description: ML人材の確保困難
      mitigation:
        - 段階的なスキル構築
        - 外部パートナー活用
        - 社内育成プログラム

    alignment:
      description: ビジネスとMLチームの乖離
      mitigation:
        - 共通KPI設定
        - 定期的なビジネスレビュー
        - A/Bテスト文化の醸成

  operational_risks:
    cost:
      description: インフラコストの増大
      mitigation:
        - 使用量ベース最適化
        - スポットインスタンス活用
        - モデル軽量化
        - キャッシュ最適化

    compliance:
      description: プライバシー規制対応
      mitigation:
        - Privacy by Design
        - データ最小化原則
        - 定期的なコンプライアンス監査
        - 法務チームとの連携
```

---

## 第7章：文書間参照 & 統合ポイント

### 7.1 関連文書参照

```yaml
document_references:
  prerequisite_documents:
    Doc-SA-005:
      title: 検索・推奨アーキテクチャ
      relevance: 検索システムとの統合ポイント
      key_sections:
        - Elasticsearch統合
        - 検索結果パーソナライゼーション
        - A/Bテストフレームワーク

    Doc-DA-001:
      title: データアーキテクチャ & スキーマ設計
      relevance: データモデルと特徴量ソース
      key_sections:
        - ユーザーデータモデル
        - アイテムデータモデル
        - 行動ログスキーマ

    Doc-DA-003:
      title: データパイプライン & ETLアーキテクチャ
      relevance: 特徴量パイプラインとの統合
      key_sections:
        - Kafkaイベントストリーム
        - バッチ処理パイプライン
        - データ品質管理

    Doc-TV-002:
      title: Technology Stack Selection
      relevance: 技術スタック整合性
      key_sections:
        - クラウドプラットフォーム選定
        - コンテナオーケストレーション
        - 監視基盤

  downstream_documents:
    Doc-AI-002:
      title: 生成AI統合計画
      relevance: レコメンデーションと生成AIの連携
      integration_points:
        - 推奨理由の自然言語生成
        - 旅程提案への推奨統合

    Doc-AI-003:
      title: パーソナライゼーションエンジン設計
      relevance: ユーザーモデリングの共有
      integration_points:
        - ユーザー嗜好モデル
        - コンテキスト認識機能
        - マルチチャネル配信

  related_business_documents:
    Doc-BM-002:
      title: 収益モデル設計
      relevance: 推奨からの収益貢献
      integration_points:
        - 推奨経由売上KPI
        - コンバージョン目標

    Doc-GS-003:
      title: 成長戦略
      relevance: AI駆動の成長施策
      integration_points:
        - ユーザーエンゲージメント
        - リテンション向上

integration_points:
  api_integration:
    recommendation_api:
      provider: this_document
      consumers:
        - Flutter Mobile App
        - Web Frontend
        - Email Service
        - Push Notification Service
      contract:
        endpoint: /api/v1/recommendations
        method: POST
        latency_sla: 50ms

  data_integration:
    feature_store:
      provider: this_document + Doc-DA-003
      consumers:
        - Recommendation Service
        - Personalization Engine (Doc-AI-003)
        - Generative AI Service (Doc-AI-002)
      contract:
        protocol: gRPC / REST
        latency_sla: 10ms

  event_integration:
    user_events:
      provider: Flutter App / Web
      consumers:
        - Feature Pipeline
        - Model Training Pipeline
        - A/B Testing Service
      contract:
        protocol: Kafka
        format: Avro
        latency_sla: 1秒

  model_integration:
    embedding_service:
      provider: this_document
      consumers:
        - Search Service (Doc-SA-005)
        - Generative AI Service (Doc-AI-002)
      contract:
        protocol: gRPC
        latency_sla: 20ms
```

### 7.2 用語集

```yaml
glossary:
  ml_terms:
    Two-Tower Model:
      definition: ユーザーとアイテムを別々のニューラルネットワーク（タワー）でエンコードし、ドット積でスコアを計算する推奨モデルアーキテクチャ
      synonym: Dual Encoder

    Feature Store:
      definition: 機械学習の特徴量を一元管理し、学習と推論で一貫して提供するシステム
      example: Feast, Tecton

    ANN (Approximate Nearest Neighbors):
      definition: 大規模ベクトル空間で近似的に最近傍を高速検索するアルゴリズム
      example: Faiss, ScaNN

    Cold Start:
      definition: 新規ユーザーや新規アイテムに対して十分な履歴データがなく、推奨が困難な状態
      mitigation: コンテンツベース推奨、人気度ベース推奨

    Model Drift:
      definition: 時間経過によりモデルの予測精度が低下する現象
      types: Data Drift, Concept Drift

  metrics_terms:
    NDCG (Normalized Discounted Cumulative Gain):
      definition: ランキング品質を測る指標。上位の正解を重視
      range: 0-1 (higher is better)

    CTR (Click-Through Rate):
      definition: 表示に対するクリックの割合
      formula: clicks / impressions

    CVR (Conversion Rate):
      definition: クリックに対するコンバージョンの割合
      formula: conversions / clicks

    PSI (Population Stability Index):
      definition: 二つの分布間の変化を測る指標。データドリフト検出に使用
      threshold: >0.2 で要注意

  architecture_terms:
    MLOps:
      definition: 機械学習システムの開発、デプロイ、運用を効率化するプラクティス
      components: 実験管理、モデル管理、パイプライン自動化、監視

    Triton Inference Server:
      definition: NVIDIAが提供する高性能モデルサービングプラットフォーム
      features: マルチフレームワーク対応、動的バッチング、GPU最適化

    Canary Release:
      definition: 新バージョンを少数のユーザーに段階的にリリースする手法
      benefit: リスク軽減、早期問題検出
```

---

## 付録

### A. 技術仕様サマリー

```yaml
technical_specifications:
  model_specifications:
    two_tower:
      user_embedding_dim: 128
      item_embedding_dim: 128
      hidden_layers: [256, 256]
      activation: ReLU
      dropout: 0.3
      batch_size: 2048
      learning_rate: 0.001

    reranker:
      input_dim: 128 + 128 + 64
      hidden_layers: [256, 128]
      output_dim: 1
      activation: ReLU

  infrastructure_specifications:
    inference_service:
      replicas: 3-50
      cpu: 4 cores
      memory: 16GB
      gpu: NVIDIA T4

    feature_store_online:
      technology: Redis Cluster
      nodes: 6
      memory: 50GB total
      latency_p99: 5ms

    model_serving:
      technology: Triton
      replicas: 3-10
      gpu: NVIDIA T4
      batch_size: 64-256

  performance_targets:
    latency:
      p50: 20ms
      p95: 40ms
      p99: 50ms
    throughput:
      sustained: 5,000 qps
      peak: 10,000 qps
    availability: 99.9%
```

### B. 変更履歴

```yaml
change_history:
  - version: 1.0.0
    date: 2026-01-21
    author: Technical Architecture Agent
    changes:
      - 初版作成
      - レコメンデーションアーキテクチャ設計
      - 特徴量エンジニアリング定義
      - MLOps基盤設計
      - 実装ロードマップ策定
```

---

**Document ID**: Doc-AI-001
**Version**: 1.0.0
**Last Updated**: 2026-01-21
**Status**: Draft
**Owner**: Technical Architecture Team
**Review Status**: Pending Review

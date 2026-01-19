# Doc-SA-003: APIアーキテクチャ・設計

## エグゼクティブサマリー

本文書は、TripTripプラットフォームのAPIアーキテクチャと設計標準を包括的に定義します。RESTful API設計原則、バージョニング戦略、認証・認可アーキテクチャ、レート制限、GraphQL戦略、イベント駆動API設計を詳述し、Google、Amazon、Netflixレベルのスケーラビリティと開発者体験を実現します。既存のHono/OpenAPI駆動型の型安全API生成システムを基盤として、グローバル展開に対応した堅牢なAPI基盤を構築します。本設計は、10万MAUから1億MAUへの成長を支え、1日1000回以上のデプロイを可能にする柔軟性と後方互換性を両立します。

---

## 第1章：はじめに & コンテキスト

### 1.1 ビジネス要件 & 制約

#### 1.1.1 API設計のビジネス要件

TripTripのAPIは、モバイルアプリ、Webクライアント、サードパーティ統合、内部サービス間通信を含む多様なコンシューマーをサポートする必要があります。

```yaml
business_requirements:
  consumer_types:
    mobile_app:
      description: Flutter iOS/Android アプリケーション
      characteristics:
        - 不安定なネットワーク接続への耐性
        - バッテリー消費を考慮した効率的な通信
        - オフラインファースト機能のサポート
      priority: P0

    web_app:
      description: Flutter Web / Progressive Web App
      characteristics:
        - SEO対応（Server-Side Rendering連携）
        - ブラウザキャッシュの活用
        - Service Worker連携
      priority: P0

    third_party:
      description: パートナー企業、旅行代理店、アグリゲーター
      characteristics:
        - 安定したAPI契約
        - 詳細なドキュメント
        - サンドボックス環境
      priority: P2

    internal_services:
      description: マイクロサービス間通信
      characteristics:
        - 高スループット、低レイテンシ
        - 型安全性
        - サービスメッシュ統合
      priority: P1

  functional_requirements:
    - requirement: 検索API
      description: ホテル、アトラクション、商品の統合検索
      latency_target: P99 < 100ms
      throughput: 10,000 req/s

    - requirement: 予約API
      description: リアルタイム在庫確認、予約作成、変更、キャンセル
      latency_target: P99 < 200ms
      consistency: 強い一貫性

    - requirement: 決済API
      description: 決済処理、返金、請求管理
      latency_target: P99 < 500ms
      security: PCI-DSS準拠

    - requirement: 推奨API
      description: パーソナライズされた推奨コンテンツ
      latency_target: P99 < 50ms
      throughput: 50,000 req/s

    - requirement: ユーザーAPI
      description: 認証、プロフィール、設定管理
      latency_target: P99 < 100ms
      security: OAuth2.0/OIDC
```

#### 1.1.2 技術的制約

```yaml
technical_constraints:
  existing_infrastructure:
    - constraint: Hono Webフレームワーク
      impact: 軽量、エッジ対応、OpenAPI統合
      strategy: 継続使用、機能拡張

    - constraint: OpenAPI/Zod型生成システム
      impact: フロントエンド・バックエンド間の型安全性
      strategy: 維持・強化

    - constraint: JWT認証基盤
      impact: 既存認証フローとの互換性
      strategy: OAuth2.0/OIDCへの段階的拡張

  scalability_constraints:
    - constraint: グローバル展開
      impact: マルチリージョン対応必須
      strategy: CDN、エッジコンピューティング活用

    - constraint: 高可用性要件（99.99%）
      impact: 障害時のフォールバック必須
      strategy: サーキットブレーカー、グレースフルデグラデーション

  compliance_constraints:
    - constraint: PCI-DSS
      impact: 決済データの取り扱い
      strategy: トークン化、暗号化

    - constraint: GDPR/個人情報保護法
      impact: ユーザーデータの管理
      strategy: データ最小化、忘れられる権利対応
```

### 1.2 技術目標 & 成功基準

```yaml
api_objectives:
  developer_experience:
    - objective: APIファースト開発
      metric: OpenAPI Specからのコード生成率
      target: 100%

    - objective: ドキュメント品質
      metric: API Docスコア（自動測定）
      target: 95点以上

    - objective: SDK提供
      metric: 公式SDKカバレッジ
      target: TypeScript, Dart, Python, Go

  performance:
    - objective: レスポンスタイム
      metric: P99レイテンシ
      target: 200ms以下（グローバル）

    - objective: スループット
      metric: ピーク時処理能力
      target: 500,000 req/s

    - objective: 可用性
      metric: 月間稼働率
      target: 99.99%

  security:
    - objective: 認証強度
      metric: OAuth2.0/OIDC準拠
      target: 100%カバレッジ

    - objective: API不正利用防止
      metric: 不正リクエスト検出率
      target: 99%

  evolution:
    - objective: 後方互換性
      metric: Breaking change率
      target: 0%（同一メジャーバージョン内）

    - objective: バージョン並行運用
      metric: 同時サポートバージョン数
      target: 最大3バージョン
```

### 1.3 主要な前提条件 & トレードオフ

```yaml
assumptions:
  technical:
    - assumption: API Gatewayの導入
      rationale: 統一的な認証、レート制限、ルーティング
      product: Kong Gateway / Google Cloud API Gateway

    - assumption: CDNエッジキャッシュの活用
      rationale: グローバルでの低レイテンシ
      product: Cloudflare / Cloud CDN

    - assumption: サービスメッシュの段階的導入
      rationale: mTLS、トラフィック管理
      product: Istio

  organizational:
    - assumption: API設計レビュープロセス
      rationale: 一貫性と品質の確保
      process: PR + API Design Review Board

    - assumption: ドキュメント自動生成
      rationale: 常に最新のドキュメント
      tool: Redoc / Swagger UI

tradeoffs:
  rest_vs_graphql:
    choice: RESTをプライマリ、GraphQLをセカンダリ
    rationale:
      - RESTは広く理解され、キャッシュが容易
      - GraphQLはモバイル向け複雑クエリに最適
      - 段階的にGraphQL採用を拡大
    implementation:
      - REST: 全APIエンドポイント
      - GraphQL: BFF層、モバイル向け集約クエリ

  versioning_strategy:
    choice: URL Path Versioning
    alternatives:
      - Header versioning: 発見性が低い
      - Query param versioning: キャッシュが困難
    rationale:
      - 明確で発見しやすい
      - CDNキャッシュと相性が良い
      - デバッグが容易

  consistency_vs_availability:
    choice: 操作に応じた使い分け
    implementation:
      - 読み取り操作: 結果整合性（高可用性）
      - 決済操作: 強い一貫性（正確性優先）
      - 予約操作: 楽観的ロック + 補償トランザクション
```

---

## 第2章：API設計原則

### 2.1 RESTful設計ガイドライン

#### 2.1.1 リソース設計原則

```yaml
resource_design:
  naming_conventions:
    - rule: リソース名は複数形の名詞
      examples:
        correct: /users, /bookings, /products
        incorrect: /user, /getBooking, /productList

    - rule: ケバブケース（kebab-case）を使用
      examples:
        correct: /hotel-rooms, /time-slots, /order-items
        incorrect: /hotelRooms, /time_slots, /OrderItems

    - rule: 階層関係はネストで表現
      examples:
        correct: /hotels/{id}/rooms, /orders/{id}/items
        incorrect: /hotel-rooms?hotel_id={id}

    - rule: アクションはリソースとして表現
      examples:
        correct: POST /bookings/{id}/cancellation
        incorrect: POST /bookings/{id}/cancel

  uri_structure:
    pattern: /{version}/{resource}/{id}/{sub-resource}/{sub-id}
    examples:
      - /v1/hotels
      - /v1/hotels/123
      - /v1/hotels/123/rooms
      - /v1/hotels/123/rooms/456
      - /v1/users/me/bookings
      - /v1/users/me/settings

  query_parameters:
    filtering:
      pattern: ?{field}={value}
      examples:
        - /products?category=kimono
        - /bookings?status=confirmed
        - /hotels?min_price=10000&max_price=50000

    sorting:
      pattern: ?sort={field}:{direction}
      examples:
        - /products?sort=price:asc
        - /hotels?sort=rating:desc,price:asc

    pagination:
      cursor_based:
        pattern: ?cursor={token}&limit={n}
        preferred: true
        rationale: 大規模データセットで効率的

      offset_based:
        pattern: ?page={n}&limit={n}
        use_case: 小規模データ、ページ番号表示が必要な場合

    field_selection:
      pattern: ?fields={field1},{field2}
      example: /hotels?fields=id,name,price,rating

    expansion:
      pattern: ?expand={relation}
      example: /bookings/123?expand=hotel,user
```

#### 2.1.2 HTTPメソッドセマンティクス

```yaml
http_methods:
  GET:
    semantics: リソースの取得
    idempotent: true
    safe: true
    cacheable: true
    request_body: なし
    examples:
      - GET /v1/hotels - ホテル一覧取得
      - GET /v1/hotels/123 - 特定ホテル取得
      - GET /v1/users/me - 現在のユーザー取得

  POST:
    semantics: リソースの作成
    idempotent: false
    safe: false
    cacheable: false
    request_body: 必須
    examples:
      - POST /v1/bookings - 予約作成
      - POST /v1/users - ユーザー登録
      - POST /v1/payments/intents - 決済Intent作成

  PUT:
    semantics: リソースの完全置換
    idempotent: true
    safe: false
    cacheable: false
    request_body: 必須
    examples:
      - PUT /v1/users/me/settings - 設定の完全更新
      - PUT /v1/products/123 - 商品の完全更新

  PATCH:
    semantics: リソースの部分更新
    idempotent: true
    safe: false
    cacheable: false
    request_body: 必須（変更部分のみ）
    examples:
      - PATCH /v1/users/me - プロフィール部分更新
      - PATCH /v1/bookings/123 - 予約の部分変更

  DELETE:
    semantics: リソースの削除
    idempotent: true
    safe: false
    cacheable: false
    request_body: 通常なし
    examples:
      - DELETE /v1/bookings/123 - 予約キャンセル
      - DELETE /v1/cart/items/456 - カートアイテム削除

  OPTIONS:
    semantics: 利用可能なメソッドの確認
    use_case: CORS preflight
```

#### 2.1.3 HTTPステータスコード標準

```yaml
status_codes:
  success:
    200_OK:
      use_case: GET, PUT, PATCH成功時
      body: リクエストされたリソース

    201_Created:
      use_case: POST成功時（リソース作成）
      body: 作成されたリソース
      headers:
        Location: 新リソースのURI

    202_Accepted:
      use_case: 非同期処理の受付
      body: 処理ステータス確認用情報

    204_No_Content:
      use_case: DELETE成功時
      body: なし

  redirection:
    301_Moved_Permanently:
      use_case: リソースの恒久的移動
      headers:
        Location: 新しいURI

    304_Not_Modified:
      use_case: キャッシュ有効時
      body: なし

  client_error:
    400_Bad_Request:
      use_case: リクエスト形式不正
      body:
        error:
          code: VALIDATION_ERROR
          message: リクエストの形式が不正です
          details:
            - field: email
              message: 有効なメールアドレスを入力してください

    401_Unauthorized:
      use_case: 認証なし/認証失敗
      body:
        error:
          code: UNAUTHORIZED
          message: 認証が必要です

    403_Forbidden:
      use_case: 権限なし
      body:
        error:
          code: FORBIDDEN
          message: このリソースへのアクセス権限がありません

    404_Not_Found:
      use_case: リソースが存在しない
      body:
        error:
          code: NOT_FOUND
          message: 指定されたリソースが見つかりません

    409_Conflict:
      use_case: 競合（重複、楽観的ロック失敗）
      body:
        error:
          code: CONFLICT
          message: リソースが競合状態です

    422_Unprocessable_Entity:
      use_case: ビジネスロジックエラー
      body:
        error:
          code: BUSINESS_RULE_VIOLATION
          message: 在庫が不足しています

    429_Too_Many_Requests:
      use_case: レート制限超過
      body:
        error:
          code: RATE_LIMIT_EXCEEDED
          message: リクエスト数の上限に達しました
      headers:
        Retry-After: 60
        X-RateLimit-Remaining: 0
        X-RateLimit-Reset: 1705660800

  server_error:
    500_Internal_Server_Error:
      use_case: 予期しないサーバーエラー
      body:
        error:
          code: INTERNAL_ERROR
          message: サーバーエラーが発生しました
          request_id: req_abc123

    502_Bad_Gateway:
      use_case: アップストリームサーバーからの不正レスポンス

    503_Service_Unavailable:
      use_case: サービス一時停止
      headers:
        Retry-After: 300

    504_Gateway_Timeout:
      use_case: アップストリームタイムアウト
```

### 2.2 APIファースト開発アプローチ

```yaml
api_first_development:
  principles:
    - principle: Design before implementation
      description: コード実装前にAPI設計を完了
      process:
        1. ビジネス要件のAPI仕様への変換
        2. OpenAPI Spec作成
        3. ステークホルダーレビュー
        4. モック生成とフロントエンド先行開発
        5. バックエンド実装
        6. 契約テストによる検証

    - principle: Contract as source of truth
      description: OpenAPI Specを単一の真実源とする
      benefits:
        - フロントエンド/バックエンドの並行開発
        - 自動ドキュメント生成
        - 型安全なSDK生成
        - 契約テストの自動化

    - principle: Consumer-driven design
      description: APIコンシューマーのニーズを優先
      practices:
        - コンシューマーフィードバックの収集
        - 使用パターンの分析
        - Developer Experienceの最適化

  workflow:
    1_requirement_analysis:
      owner: Product Manager + API Designer
      output: API要件ドキュメント
      duration: 1-2日

    2_api_design:
      owner: API Designer
      output: OpenAPI Spec (Draft)
      tools: Stoplight Studio, Swagger Editor
      duration: 2-3日

    3_design_review:
      owner: API Design Review Board
      checklist:
        - RESTful設計原則の遵守
        - 命名規則の一貫性
        - セキュリティ要件の充足
        - パフォーマンス考慮
        - 後方互換性
      duration: 1日

    4_mock_generation:
      owner: Platform Team
      output: Mockサーバー、SDK
      tools: Prism, openapi-generator
      duration: 即時（自動化）

    5_parallel_development:
      frontend:
        owner: Frontend Team
        approach: モックAPIを使用した開発
      backend:
        owner: Backend Team
        approach: OpenAPI Specに基づく実装

    6_contract_testing:
      owner: QA Team
      approach: Pact契約テスト
      timing: CI/CDパイプラインで自動実行

    7_deployment:
      owner: Platform Team
      approach: カナリアデプロイ
```

### 2.3 OpenAPI/Swagger標準

#### 2.3.1 OpenAPI仕様テンプレート

```yaml
openapi_standard:
  version: "3.1.0"

  specification_structure:
    info:
      required:
        - title
        - version
        - description
        - contact
        - license
      example:
        title: TripTrip API
        version: 1.2.0
        description: TripTrip Travel Platform API
        contact:
          name: API Support
          email: api-support@triptrip.com
          url: https://developers.triptrip.com
        license:
          name: Proprietary
          url: https://triptrip.com/terms

    servers:
      environments:
        - url: https://api.triptrip.com/v1
          description: Production
        - url: https://api.staging.triptrip.com/v1
          description: Staging
        - url: https://api.sandbox.triptrip.com/v1
          description: Sandbox

    security:
      schemes:
        bearerAuth:
          type: http
          scheme: bearer
          bearerFormat: JWT

        apiKey:
          type: apiKey
          in: header
          name: X-API-Key

        oauth2:
          type: oauth2
          flows:
            authorizationCode:
              authorizationUrl: https://auth.triptrip.com/oauth/authorize
              tokenUrl: https://auth.triptrip.com/oauth/token
              scopes:
                read:bookings: 予約情報の読み取り
                write:bookings: 予約の作成・変更
                read:profile: プロフィール情報の読み取り
                write:profile: プロフィールの更新

    tags:
      organization:
        - name: Authentication
          description: 認証関連API
        - name: Users
          description: ユーザー管理API
        - name: Hotels
          description: ホテル情報API
        - name: Bookings
          description: 予約管理API
        - name: Payments
          description: 決済API
        - name: Search
          description: 検索API

  component_patterns:
    schemas:
      naming: PascalCase
      structure:
        resource:
          pattern: "{Resource}"
          example: Hotel, Booking, User

        request:
          pattern: "{Action}{Resource}Request"
          example: CreateBookingRequest, UpdateUserRequest

        response:
          pattern: "{Resource}Response"
          example: HotelResponse, BookingResponse

        list:
          pattern: "{Resource}ListResponse"
          example: HotelListResponse

        error:
          pattern: "{Type}Error"
          example: ValidationError, BusinessError

    parameters:
      common:
        - name: X-Request-ID
          in: header
          required: false
          schema:
            type: string
            format: uuid

        - name: Accept-Language
          in: header
          required: false
          schema:
            type: string
            default: ja

        - name: X-Currency
          in: header
          required: false
          schema:
            type: string
            default: JPY

    responses:
      standard:
        "400":
          $ref: "#/components/responses/BadRequest"
        "401":
          $ref: "#/components/responses/Unauthorized"
        "403":
          $ref: "#/components/responses/Forbidden"
        "404":
          $ref: "#/components/responses/NotFound"
        "429":
          $ref: "#/components/responses/TooManyRequests"
        "500":
          $ref: "#/components/responses/InternalError"
```

#### 2.3.2 Zod-OpenAPI統合

```typescript
// Hono + Zod-OpenAPI統合例
import { createRoute, OpenAPIHono } from "@hono/zod-openapi";
import { z } from "zod";

// スキーマ定義
const HotelSchema = z.object({
  id: z.string().uuid().openapi({ example: "550e8400-e29b-41d4-a716-446655440000" }),
  name: z.string().min(1).max(200).openapi({ example: "グランドホテル東京" }),
  description: z.string().max(5000).openapi({ example: "東京駅徒歩5分の高級ホテル" }),
  location: z.object({
    lat: z.number().min(-90).max(90).openapi({ example: 35.6812 }),
    lng: z.number().min(-180).max(180).openapi({ example: 139.7671 }),
    address: z.string().openapi({ example: "東京都千代田区丸の内1-1-1" }),
  }),
  rating: z.number().min(0).max(5).openapi({ example: 4.5 }),
  priceRange: z.object({
    min: z.number().positive().openapi({ example: 15000 }),
    max: z.number().positive().openapi({ example: 80000 }),
    currency: z.string().length(3).openapi({ example: "JPY" }),
  }),
  amenities: z.array(z.string()).openapi({ example: ["WiFi", "朝食", "温泉"] }),
  images: z.array(z.object({
    url: z.string().url(),
    alt: z.string(),
  })),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
}).openapi("Hotel");

const HotelListResponseSchema = z.object({
  data: z.array(HotelSchema),
  meta: z.object({
    total: z.number().int().nonnegative(),
    page: z.number().int().positive(),
    limit: z.number().int().positive().max(100),
    hasNext: z.boolean(),
    cursor: z.string().optional(),
  }),
}).openapi("HotelListResponse");

// ルート定義
const getHotelsRoute = createRoute({
  method: "get",
  path: "/hotels",
  tags: ["Hotels"],
  summary: "ホテル一覧取得",
  description: "検索条件に基づいてホテル一覧を取得します",
  request: {
    query: z.object({
      q: z.string().optional().openapi({ description: "検索キーワード" }),
      location: z.string().optional().openapi({ description: "都市/エリア" }),
      minPrice: z.coerce.number().positive().optional(),
      maxPrice: z.coerce.number().positive().optional(),
      rating: z.coerce.number().min(1).max(5).optional(),
      amenities: z.string().optional().openapi({ description: "カンマ区切り" }),
      sort: z.enum(["price:asc", "price:desc", "rating:desc", "distance"]).optional(),
      page: z.coerce.number().int().positive().default(1),
      limit: z.coerce.number().int().positive().max(100).default(20),
    }),
    headers: z.object({
      "Accept-Language": z.string().default("ja"),
      "X-Currency": z.string().length(3).default("JPY"),
    }),
  },
  responses: {
    200: {
      description: "ホテル一覧",
      content: {
        "application/json": {
          schema: HotelListResponseSchema,
        },
      },
    },
    400: {
      description: "バリデーションエラー",
      content: {
        "application/json": {
          schema: ValidationErrorSchema,
        },
      },
    },
  },
});

// アプリケーション
const app = new OpenAPIHono();

app.openapi(getHotelsRoute, async (c) => {
  const query = c.req.valid("query");
  const headers = c.req.valid("header");

  const hotels = await hotelService.search(query, {
    language: headers["Accept-Language"],
    currency: headers["X-Currency"],
  });

  return c.json(hotels, 200);
});

// OpenAPI仕様生成
app.doc("/openapi.json", {
  openapi: "3.1.0",
  info: {
    title: "TripTrip API",
    version: "1.0.0",
  },
});
```

---

## 第3章：APIバージョニング戦略

### 3.1 バージョニングスキーム

```yaml
versioning_scheme:
  approach: URL Path Versioning
  format: /v{major}

  version_structure:
    major:
      trigger: Breaking changes
      examples:
        - リソース構造の根本的変更
        - 認証方式の変更
        - エンドポイントの削除
      frequency: 年1-2回以下

    minor:
      trigger: 後方互換の機能追加
      examples:
        - 新エンドポイント追加
        - オプションフィールド追加
        - 新しいフィルタオプション
      frequency: 月1-2回
      表現方法: ヘッダー（X-API-Version: 1.2）

    patch:
      trigger: バグ修正、ドキュメント更新
      examples:
        - バグ修正
        - パフォーマンス改善
        - ドキュメント修正
      frequency: 随時
      表現方法: 非表示（自動適用）

  url_examples:
    current: https://api.triptrip.com/v1/hotels
    next_major: https://api.triptrip.com/v2/hotels

  header_versioning:
    optional_minor:
      header: X-API-Version
      format: "{major}.{minor}"
      default: 最新のminor
      example:
        request:
          headers:
            X-API-Version: "1.2"
```

### 3.2 後方互換性保証

```yaml
backward_compatibility:
  guaranteed_within_major_version:
    - 既存フィールドの型は変更しない
    - 既存フィールドは削除しない（非推奨化のみ）
    - 必須フィールドを追加しない
    - レスポンス構造を変更しない
    - エラーコードを変更しない
    - 認証方式を変更しない

  acceptable_changes:
    - オプションフィールドの追加
    - 新しいエンドポイントの追加
    - 新しいクエリパラメータの追加
    - 新しいHTTPヘッダーの追加
    - 新しい enum 値の追加（デフォルトあり）
    - エラーメッセージの改善

  breaking_change_definition:
    - エンドポイントの削除または名前変更
    - フィールドの削除または名前変更
    - フィールド型の変更
    - 必須フィールドの追加
    - デフォルト値の変更
    - 認証要件の変更
    - エラーコードの変更
    - レスポンス構造の変更

  compatibility_testing:
    automated:
      - OpenAPI diff検出
      - 契約テスト（Pact）
      - 回帰テスト
    manual:
      - コンシューマー影響分析
      - マイグレーションガイド作成

  example_evolution:
    v1_original:
      response:
        id: "123"
        name: "ホテルA"
        price: 15000

    v1_with_optional_field:
      response:
        id: "123"
        name: "ホテルA"
        price: 15000
        description: "新しいオプションフィールド"  # 追加OK

    v2_breaking_change:
      response:
        hotel_id: "123"  # id → hotel_id は Breaking Change
        name: "ホテルA"
        pricing:
          amount: 15000  # price → pricing.amount は Breaking Change
          currency: "JPY"
```

### 3.3 非推奨・サンセットポリシー

```yaml
deprecation_policy:
  timeline:
    announcement:
      timing: サンセット12ヶ月前
      channels:
        - API changelog
        - 開発者メーリングリスト
        - ダッシュボード通知
        - Deprecation レスポンスヘッダー

    deprecation_period:
      duration: 12ヶ月
      behavior:
        - 完全に機能する
        - Deprecation ヘッダーを返す
        - 使用状況をログ
        - マイグレーションガイド提供

    sunset_period:
      duration: 3ヶ月
      behavior:
        - 読み取り専用
        - Sunset ヘッダーを返す
        - 警告ログ
        - 新規クライアント登録拒否

    retirement:
      behavior:
        - 410 Gone を返す
        - リダイレクト情報を提供

  response_headers:
    deprecation:
      header: Deprecation
      format: "@unix_timestamp"
      example:
        Deprecation: "@1735689600"
        Link: <https://api.triptrip.com/v2/hotels>; rel="successor-version"

    sunset:
      header: Sunset
      format: "HTTP-date"
      example:
        Sunset: "Sat, 31 Dec 2025 23:59:59 GMT"
        Link: <https://developers.triptrip.com/migration/v1-to-v2>; rel="deprecation"

  migration_support:
    documentation:
      - Breaking changes一覧
      - フィールドマッピング表
      - コード例（before/after）
      - FAQ

    tools:
      - マイグレーションスクリプト
      - 互換性チェッカー
      - SDK更新ガイド

    support:
      - 専用Slackチャンネル
      - 移行サポートセッション
      - 個別相談窓口

  version_lifecycle:
    diagram: |
      v1: Active ───────────────────────────────────────────────────────────►

      v2: ────────────────── Active ────────────────────────────────────────►
                   │
      v1:          │ Deprecated (12m) ──── Sunset (3m) ──── Retired
                   │
      Timeline: ───┼────────────────────┼──────────────────┼─────────────────►
                 v2 GA            Deprecation         Sunset          Retirement
                                   Notice
```

---

## 第4章：コアAPIエンドポイント設計

### 4.1 検索API（宿泊、チケット、アクティビティ）

```yaml
search_api:
  base_path: /v1/search

  endpoints:
    # 統合検索
    unified_search:
      method: GET
      path: /v1/search
      description: ホテル、アトラクション、商品を横断検索
      parameters:
        query:
          q:
            type: string
            required: true
            min_length: 2
            max_length: 200
            description: 検索キーワード
          type:
            type: array
            items: [hotel, attraction, product]
            default: [hotel, attraction, product]
          location:
            type: string
            description: 都市/エリア名
          lat:
            type: number
            format: float
            description: 緯度
          lng:
            type: number
            format: float
            description: 経度
          radius_km:
            type: number
            default: 10
            max: 100
          date_from:
            type: string
            format: date
          date_to:
            type: string
            format: date
          limit:
            type: integer
            default: 10
            max: 20
            per_type: true
      response:
        200:
          content:
            hotels:
              total: integer
              items: Hotel[]
            attractions:
              total: integer
              items: Attraction[]
            products:
              total: integer
              items: Product[]
            suggestions:
              - text: string
                type: string
      performance:
        latency_target: P99 < 100ms
        cache: 5分（人気クエリ）

    # ホテル検索
    hotel_search:
      method: GET
      path: /v1/search/hotels
      description: ホテル専用の詳細検索
      parameters:
        query:
          q:
            type: string
          location:
            type: string
            required: true
          check_in:
            type: string
            format: date
            required: true
          check_out:
            type: string
            format: date
            required: true
          guests:
            type: integer
            default: 2
            min: 1
            max: 10
          rooms:
            type: integer
            default: 1
            min: 1
            max: 5
          min_price:
            type: integer
            min: 0
          max_price:
            type: integer
          amenities:
            type: array
            items: string
            description: WiFi,朝食,温泉,駐車場
          rating_min:
            type: number
            min: 1
            max: 5
          hotel_class:
            type: array
            items: [1, 2, 3, 4, 5]
          sort:
            type: string
            enum: [relevance, price_asc, price_desc, rating, distance, popularity]
            default: relevance
          page:
            type: integer
            default: 1
          limit:
            type: integer
            default: 20
            max: 100
      response:
        200:
          content:
            data:
              - id: string
                name: string
                description: string
                location:
                  lat: number
                  lng: number
                  address: string
                  distance_km: number
                rating:
                  average: number
                  count: integer
                price:
                  min: number
                  max: number
                  currency: string
                  per_night: boolean
                amenities: string[]
                images:
                  - url: string
                    alt: string
                availability:
                  available: boolean
                  rooms_left: integer
                hotel_class: integer
            meta:
              total: integer
              page: integer
              limit: integer
              has_next: boolean
            facets:
              amenities:
                - name: string
                  count: integer
              price_ranges:
                - range: string
                  min: number
                  max: number
                  count: integer
              ratings:
                - value: number
                  count: integer
              hotel_classes:
                - value: integer
                  count: integer
      performance:
        latency_target: P99 < 100ms
        cache: 1分（在庫反映のため短め）

    # アトラクション検索
    attraction_search:
      method: GET
      path: /v1/search/attractions
      description: アトラクション・チケット検索
      parameters:
        query:
          q:
            type: string
          location:
            type: string
          category:
            type: array
            items: [theme_park, museum, tour, experience, show]
          date:
            type: string
            format: date
          time_slot:
            type: string
            format: time
          min_price:
            type: integer
          max_price:
            type: integer
          duration_min:
            type: integer
            description: 最小所要時間（分）
          duration_max:
            type: integer
          sort:
            type: string
            enum: [relevance, price_asc, price_desc, rating, popularity, duration]
          page:
            type: integer
          limit:
            type: integer
      response:
        200:
          content:
            data: Attraction[]
            meta: PaginationMeta
            facets: AttractionFacets

    # 商品検索
    product_search:
      method: GET
      path: /v1/search/products
      description: 物販商品検索
      parameters:
        query:
          q:
            type: string
          category:
            type: array
            items: [kimono, accessories, souvenirs, food, cosmetics]
          min_price:
            type: integer
          max_price:
            type: integer
          in_stock:
            type: boolean
            default: true
          sort:
            type: string
            enum: [relevance, price_asc, price_desc, rating, newest, popular]
          page:
            type: integer
          limit:
            type: integer
      response:
        200:
          content:
            data: Product[]
            meta: PaginationMeta
            facets: ProductFacets

    # オートコンプリート
    autocomplete:
      method: GET
      path: /v1/search/autocomplete
      description: 検索候補のリアルタイム提案
      parameters:
        query:
          q:
            type: string
            required: true
            min_length: 1
          type:
            type: string
            enum: [all, hotel, attraction, product, location]
            default: all
          limit:
            type: integer
            default: 10
            max: 20
      response:
        200:
          content:
            suggestions:
              - text: string
                type: string
                highlight: string
                metadata:
                  id: string
                  image_url: string
      performance:
        latency_target: P99 < 50ms
        cache: 15分
```

### 4.2 予約API（カート、注文、決済）

```yaml
booking_api:
  base_path: /v1/bookings

  endpoints:
    # 予約作成
    create_booking:
      method: POST
      path: /v1/bookings
      description: 新規予約の作成
      authentication: required
      request:
        body:
          booking_type:
            type: string
            enum: [hotel, attraction, product, rental]
            required: true
          resource_id:
            type: string
            format: uuid
            required: true
          # ホテル予約用
          check_in_date:
            type: string
            format: date
            required_if: booking_type == hotel
          check_out_date:
            type: string
            format: date
            required_if: booking_type == hotel
          room_type_id:
            type: string
            required_if: booking_type == hotel
          # アトラクション予約用
          visit_date:
            type: string
            format: date
            required_if: booking_type == attraction
          time_slot_id:
            type: string
            required_if: booking_type == attraction
          # 共通
          quantity:
            type: integer
            min: 1
            default: 1
          guest_info:
            type: object
            properties:
              primary_guest:
                name:
                  type: string
                  required: true
                email:
                  type: string
                  format: email
                  required: true
                phone:
                  type: string
                  required: true
              additional_guests:
                type: array
                items:
                  name: string
          special_requests:
            type: string
            max_length: 1000
          promo_code:
            type: string
          idempotency_key:
            type: string
            format: uuid
            required: true
            description: 重複リクエスト防止
      response:
        201:
          content:
            id: string
            booking_number: string
            status: pending
            booking_type: string
            resource:
              id: string
              name: string
              image_url: string
            dates:
              check_in: string
              check_out: string
              visit_date: string
              time_slot: string
            quantity: integer
            pricing:
              subtotal: number
              tax: number
              service_fee: number
              discount: number
              total: number
              currency: string
              breakdown:
                - description: string
                  amount: number
            guest_info: object
            payment:
              required: boolean
              payment_intent_id: string
              client_secret: string
            hold_expires_at: string
            created_at: string
        400:
          description: バリデーションエラー
        409:
          description: 在庫不足/競合
        422:
          description: ビジネスルール違反

    # 予約取得
    get_booking:
      method: GET
      path: /v1/bookings/{booking_id}
      description: 予約詳細の取得
      authentication: required
      parameters:
        path:
          booking_id:
            type: string
            format: uuid
        query:
          expand:
            type: array
            items: [resource, payment, timeline]
      response:
        200:
          content:
            id: string
            booking_number: string
            status: string
            booking_type: string
            resource: Resource
            dates: BookingDates
            quantity: integer
            pricing: Pricing
            guest_info: GuestInfo
            payment: PaymentInfo
            cancellation_policy: CancellationPolicy
            timeline:
              - event: string
                timestamp: string
                description: string
            created_at: string
            updated_at: string

    # 予約一覧
    list_bookings:
      method: GET
      path: /v1/bookings
      description: ユーザーの予約一覧
      authentication: required
      parameters:
        query:
          status:
            type: array
            items: [pending, confirmed, cancelled, completed]
          booking_type:
            type: array
            items: [hotel, attraction, product, rental]
          date_from:
            type: string
            format: date
          date_to:
            type: string
            format: date
          sort:
            type: string
            enum: [created_at:desc, created_at:asc, date:asc, date:desc]
            default: created_at:desc
          page:
            type: integer
          limit:
            type: integer
      response:
        200:
          content:
            data: Booking[]
            meta: PaginationMeta

    # 予約確定
    confirm_booking:
      method: POST
      path: /v1/bookings/{booking_id}/confirm
      description: 決済完了後の予約確定
      authentication: required
      request:
        body:
          payment_intent_id:
            type: string
            required: true
      response:
        200:
          content:
            id: string
            status: confirmed
            confirmation_number: string
            confirmation_email_sent: boolean

    # 予約キャンセル
    cancel_booking:
      method: POST
      path: /v1/bookings/{booking_id}/cancellation
      description: 予約のキャンセル
      authentication: required
      request:
        body:
          reason:
            type: string
            enum: [changed_plans, found_better_deal, emergency, other]
            required: true
          reason_detail:
            type: string
            max_length: 500
      response:
        200:
          content:
            id: string
            status: cancelled
            refund:
              eligible: boolean
              amount: number
              currency: string
              method: string
              status: string
              estimated_arrival: string
            cancellation_fee: number

    # 予約変更
    modify_booking:
      method: PATCH
      path: /v1/bookings/{booking_id}
      description: 予約内容の変更
      authentication: required
      request:
        body:
          check_in_date:
            type: string
            format: date
          check_out_date:
            type: string
            format: date
          visit_date:
            type: string
            format: date
          time_slot_id:
            type: string
          quantity:
            type: integer
          guest_info:
            type: object
          special_requests:
            type: string
      response:
        200:
          content:
            id: string
            status: string
            changes:
              - field: string
                old_value: any
                new_value: any
            price_difference:
              amount: number
              currency: string
              requires_payment: boolean
              payment_intent_id: string
```

### 4.3 推奨API（パーソナライゼーション）

```yaml
recommendation_api:
  base_path: /v1/recommendations

  endpoints:
    # パーソナライズド推奨
    get_personalized:
      method: GET
      path: /v1/recommendations
      description: ユーザーに最適化された推奨コンテンツ
      authentication: optional (認証時はパーソナライズ)
      parameters:
        query:
          type:
            type: array
            items: [hotel, attraction, product, destination]
            default: [hotel, attraction, product]
          context:
            type: string
            enum: [home, search, booking_complete, cart]
            default: home
          location:
            type: string
            description: 現在地または関心エリア
          limit:
            type: integer
            default: 10
            max: 50
          exclude_ids:
            type: array
            items: string
            description: 除外するリソースID
      response:
        200:
          content:
            sections:
              - id: string
                title: string
                subtitle: string
                type: string
                items:
                  - id: string
                    type: string
                    name: string
                    description: string
                    image_url: string
                    price:
                      amount: number
                      currency: string
                    rating: number
                    reason: string
                    confidence: number
      performance:
        latency_target: P99 < 50ms
        cache: ユーザー別30分、匿名1時間

    # 類似アイテム推奨
    get_similar:
      method: GET
      path: /v1/recommendations/similar
      description: 特定アイテムに類似したアイテム
      parameters:
        query:
          item_id:
            type: string
            required: true
          item_type:
            type: string
            enum: [hotel, attraction, product]
            required: true
          limit:
            type: integer
            default: 10
      response:
        200:
          content:
            items:
              - id: string
                type: string
                name: string
                similarity_score: number
                similarity_reasons: string[]

    # よく一緒に予約されるアイテム
    get_frequently_booked_together:
      method: GET
      path: /v1/recommendations/together
      description: よく一緒に予約されるアイテム
      parameters:
        query:
          item_id:
            type: string
            required: true
          item_type:
            type: string
            required: true
          limit:
            type: integer
            default: 5
      response:
        200:
          content:
            items:
              - id: string
                type: string
                name: string
                co_occurrence_score: number
                bundle_discount_available: boolean

    # トレンド
    get_trending:
      method: GET
      path: /v1/recommendations/trending
      description: 人気・トレンドアイテム
      parameters:
        query:
          type:
            type: string
            enum: [hotel, attraction, product, destination]
          location:
            type: string
          time_range:
            type: string
            enum: [day, week, month]
            default: week
          limit:
            type: integer
            default: 10
      response:
        200:
          content:
            items:
              - id: string
                type: string
                name: string
                trend_score: number
                trend_direction: string
                booking_velocity: number
```

### 4.4 ユーザーAPI（認証、プロファイル）

```yaml
user_api:
  base_path: /v1

  endpoints:
    # ユーザー登録
    register:
      method: POST
      path: /v1/auth/register
      description: 新規ユーザー登録
      rate_limit: 3/hour/IP
      request:
        body:
          email:
            type: string
            format: email
            required: true
          password:
            type: string
            min_length: 8
            required: true
            pattern: "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d).+$"
          name:
            type: string
            min_length: 1
            max_length: 100
            required: true
          language:
            type: string
            enum: [ja, en, zh, ko]
            default: ja
          currency:
            type: string
            length: 3
            default: JPY
          marketing_consent:
            type: boolean
            default: false
      response:
        201:
          content:
            user:
              id: string
              email: string
              name: string
              email_verified: boolean
            tokens:
              access_token: string
              refresh_token: string
              token_type: bearer
              expires_in: 900

    # ログイン
    login:
      method: POST
      path: /v1/auth/login
      description: メール/パスワードでログイン
      rate_limit: 10/minute/IP
      request:
        body:
          email:
            type: string
            format: email
            required: true
          password:
            type: string
            required: true
          mfa_code:
            type: string
            length: 6
            description: MFA有効時に必要
      response:
        200:
          content:
            user:
              id: string
              email: string
              name: string
            tokens:
              access_token: string
              refresh_token: string
              token_type: bearer
              expires_in: 900
            requires_mfa: boolean
        401:
          description: 認証失敗
        403:
          description: アカウントロック

    # トークンリフレッシュ
    refresh_token:
      method: POST
      path: /v1/auth/refresh
      description: アクセストークンの更新
      request:
        body:
          refresh_token:
            type: string
            required: true
      response:
        200:
          content:
            access_token: string
            refresh_token: string
            token_type: bearer
            expires_in: 900

    # ログアウト
    logout:
      method: POST
      path: /v1/auth/logout
      description: セッション終了
      authentication: required
      request:
        body:
          all_devices:
            type: boolean
            default: false
            description: 全デバイスからログアウト
      response:
        204:
          description: ログアウト成功

    # 現在のユーザー取得
    get_current_user:
      method: GET
      path: /v1/users/me
      description: ログインユーザーの情報取得
      authentication: required
      response:
        200:
          content:
            id: string
            email: string
            name: string
            avatar_url: string
            language: string
            currency: string
            email_verified: boolean
            mfa_enabled: boolean
            created_at: string
            last_login_at: string

    # プロフィール更新
    update_profile:
      method: PATCH
      path: /v1/users/me
      description: プロフィールの部分更新
      authentication: required
      request:
        body:
          name:
            type: string
          avatar_url:
            type: string
            format: uri
          language:
            type: string
          currency:
            type: string
      response:
        200:
          content:
            id: string
            email: string
            name: string
            avatar_url: string
            language: string
            currency: string
            updated_at: string

    # 設定取得
    get_settings:
      method: GET
      path: /v1/users/me/settings
      description: ユーザー設定の取得
      authentication: required
      response:
        200:
          content:
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
    update_settings:
      method: PUT
      path: /v1/users/me/settings
      description: ユーザー設定の更新
      authentication: required
      request:
        body:
          notification:
            type: object
          privacy:
            type: object
          display:
            type: object
      response:
        200:
          content:
            notification: object
            privacy: object
            display: object
            updated_at: string
```

---

## 第5章：認証・認可アーキテクチャ

### 5.1 OAuth 2.0 + OpenID Connect

```yaml
oauth2_oidc:
  implementation: OAuth 2.0 + OpenID Connect 1.0

  flows:
    authorization_code:
      use_case: Webアプリケーション、サーバーサイドアプリ
      security: 高（バックチャネル通信）
      pkce: 必須
      process:
        1. クライアントが認可エンドポイントにリダイレクト
        2. ユーザーが認証・認可
        3. 認可サーバーがcodeを発行
        4. クライアントがcodeをtokenに交換
        5. アクセストークン、リフレッシュトークン、IDトークン取得

    authorization_code_with_pkce:
      use_case: モバイルアプリ、SPA
      security: 高（PKCE保護）
      process:
        1. code_verifier生成（43-128文字のランダム文字列）
        2. code_challenge = BASE64URL(SHA256(code_verifier))
        3. 認可リクエストにcode_challengeを含める
        4. トークン交換時にcode_verifierを送信
        5. サーバーで検証

    client_credentials:
      use_case: サービス間通信、バックエンドAPI
      security: 中（クライアント秘密鍵の管理が必要）
      process:
        1. クライアントがclient_id, client_secretを送信
        2. アクセストークン取得
        3. ユーザーコンテキストなし

  endpoints:
    authorization:
      url: https://auth.triptrip.com/oauth/authorize
      method: GET
      parameters:
        response_type: code
        client_id: required
        redirect_uri: required
        scope: required
        state: required (CSRF protection)
        code_challenge: required (PKCE)
        code_challenge_method: S256

    token:
      url: https://auth.triptrip.com/oauth/token
      method: POST
      grant_types:
        - authorization_code
        - refresh_token
        - client_credentials
      parameters:
        grant_type: required
        code: required (for authorization_code)
        redirect_uri: required (for authorization_code)
        code_verifier: required (for PKCE)
        refresh_token: required (for refresh_token)
        client_id: required
        client_secret: required (for confidential clients)

    userinfo:
      url: https://auth.triptrip.com/oauth/userinfo
      method: GET
      authentication: Bearer token
      response:
        sub: string
        email: string
        email_verified: boolean
        name: string
        picture: string
        locale: string

    revocation:
      url: https://auth.triptrip.com/oauth/revoke
      method: POST
      parameters:
        token: required
        token_type_hint: access_token | refresh_token

  scopes:
    openid: OpenID Connect認証
    profile: 基本プロフィール情報
    email: メールアドレス
    read:bookings: 予約情報の読み取り
    write:bookings: 予約の作成・変更
    read:profile: プロフィール情報の読み取り
    write:profile: プロフィールの更新
    read:payments: 決済情報の読み取り
    write:payments: 決済の実行

  external_providers:
    google:
      client_id: ${GOOGLE_CLIENT_ID}
      scopes: [openid, email, profile]
      mapping:
        sub: google_id
        email: email
        name: name
        picture: avatar_url

    line:
      client_id: ${LINE_CLIENT_ID}
      scopes: [openid, profile, email]
      mapping:
        sub: line_id
        email: email
        displayName: name
        pictureUrl: avatar_url

    apple:
      client_id: ${APPLE_CLIENT_ID}
      scopes: [name, email]
      mapping:
        sub: apple_id
        email: email
```

### 5.2 JWTトークン設計

```yaml
jwt_design:
  algorithm: RS256 (RSA Signature with SHA-256)

  key_management:
    rotation_period: 30日
    active_keys: 2 (現在 + 次)
    jwks_endpoint: https://auth.triptrip.com/.well-known/jwks.json
    key_storage: Google Cloud KMS

  access_token:
    type: JWT
    ttl: 15分
    claims:
      standard:
        iss: https://auth.triptrip.com
        sub: user_id
        aud: triptrip-api
        exp: expiration_time
        iat: issued_at
        jti: unique_token_id
      custom:
        email: user_email
        roles: [user, admin]
        permissions: [read:bookings, write:bookings]
        session_id: session_identifier
    example:
      header:
        alg: RS256
        typ: JWT
        kid: key_id_123
      payload:
        iss: https://auth.triptrip.com
        sub: usr_abc123
        aud: triptrip-api
        exp: 1705661700
        iat: 1705660800
        jti: tok_xyz789
        email: user@example.com
        roles: [user]
        permissions: [read:bookings, write:bookings]
        session_id: ses_def456

  refresh_token:
    type: Opaque (not JWT)
    ttl: 7日
    storage: Redis + PostgreSQL
    rotation: true (毎回新しいトークンを発行)
    family_tracking: true (トークンファミリーで不正検出)
    structure:
      id: unique_id
      user_id: user_id
      session_id: session_id
      family_id: token_family_id
      created_at: timestamp
      expires_at: timestamp
      revoked: boolean
      revoked_at: timestamp

  id_token:
    type: JWT
    ttl: 1時間
    use_case: ユーザー認証の証明（OpenID Connect）
    claims:
      standard:
        iss: issuer
        sub: subject (user_id)
        aud: audience (client_id)
        exp: expiration
        iat: issued_at
        auth_time: authentication_time
        nonce: nonce (if provided)
      profile:
        name: user_name
        email: user_email
        email_verified: boolean
        picture: avatar_url
        locale: ja

  validation:
    steps:
      1. ヘッダーのalg確認（RS256）
      2. 署名検証（公開鍵使用）
      3. exp確認（有効期限）
      4. iss確認（発行者）
      5. aud確認（対象者）
      6. jtiの重複確認（リプレイ攻撃防止）
```

### 5.3 API Keyマネジメント

```yaml
api_key_management:
  use_cases:
    - サードパーティ統合
    - サーバー間通信
    - レート制限管理
    - 使用量追跡

  key_types:
    publishable:
      prefix: pk_
      visibility: クライアントサイドで使用可能
      permissions: 読み取り専用
      rate_limit: 低め
      example: pk_live_abc123xyz

    secret:
      prefix: sk_
      visibility: サーバーサイドのみ
      permissions: 全権限
      rate_limit: 高め
      example: sk_live_def456uvw

  lifecycle:
    creation:
      - ダッシュボードから生成
      - 名前と説明を必須
      - スコープ（権限）を設定
      - 有効期限を設定（任意）

    storage:
      - ハッシュ化して保存（SHA-256）
      - 生成時のみ平文表示
      - 再取得不可（再生成のみ）

    rotation:
      - 新旧キーの並行稼働期間
      - 古いキーの自動無効化
      - 通知による移行促進

    revocation:
      - 即時無効化
      - 監査ログ記録
      - 関連セッションの終了

  structure:
    prefix: 環境識別（pk_live_, sk_test_）
    random_part: 32バイトのランダム文字列
    checksum: 最後の4文字（検証用）
    format: "{prefix}_{random}_{checksum}"
    length: 40-50文字

  security:
    transmission: HTTPSのみ
    header: X-API-Key または Authorization: Bearer
    logging: キーの先頭8文字のみログ
    alerting: 不正使用検出時に通知
```

### 5.4 スコープ・パーミッション設計

```yaml
permission_design:
  model: Role-Based Access Control (RBAC) + Attribute-Based Access Control (ABAC)

  roles:
    user:
      description: 一般ユーザー
      permissions:
        - read:own:profile
        - write:own:profile
        - read:own:bookings
        - write:own:bookings
        - read:own:payments
        - read:hotels
        - read:attractions
        - read:products

    premium_user:
      description: プレミアム会員
      inherits: user
      additional_permissions:
        - access:premium:features
        - read:premium:content
        - priority:support

    partner:
      description: パートナー企業
      permissions:
        - read:hotels
        - read:attractions
        - write:partner:content
        - read:partner:analytics
        - manage:partner:inventory

    admin:
      description: システム管理者
      permissions:
        - read:all:*
        - write:all:*
        - manage:users
        - manage:system

  permission_structure:
    format: "{action}:{scope}:{resource}"
    actions:
      - read: 読み取り
      - write: 作成・更新
      - delete: 削除
      - manage: 完全な管理権限
    scopes:
      - own: 自身のリソースのみ
      - partner: パートナーのリソース
      - all: 全リソース
    resources:
      - profile
      - bookings
      - payments
      - hotels
      - attractions
      - products
      - users
      - system

  abac_attributes:
    user:
      - user_id
      - email_verified
      - account_age
      - membership_level
    resource:
      - owner_id
      - created_at
      - status
    environment:
      - ip_address
      - geo_location
      - time_of_day
      - device_type

  policy_examples:
    own_resource_access:
      description: ユーザーは自身のリソースのみアクセス可能
      condition: resource.owner_id == user.id

    verified_email_required:
      description: メール認証済みユーザーのみ予約可能
      condition: user.email_verified == true

    geo_restriction:
      description: 日本国内からのみ管理画面アクセス可能
      condition: environment.geo_location.country == "JP"
```

---

## 第6章：レート制限・スロットリング

### 6.1 レート制限アルゴリズム

```yaml
rate_limiting_algorithms:
  sliding_window_log:
    description: スライディングウィンドウログ
    implementation: Redis ZSET
    accuracy: 高
    memory: 中〜高
    use_case: 厳密な制限が必要な場合

  sliding_window_counter:
    description: スライディングウィンドウカウンター
    implementation: Redis HASH
    accuracy: 中（近似値）
    memory: 低
    use_case: 大規模APIの一般的な制限
    algorithm: |
      current_weight = (current_window_count * (1 - elapsed_ratio)) + previous_window_count
      if current_weight > limit:
        reject request

  token_bucket:
    description: トークンバケット
    implementation: Redis + Lua Script
    accuracy: 高
    memory: 低
    use_case: バースト許容が必要な場合
    parameters:
      capacity: バケット容量
      refill_rate: トークン補充レート
      refill_interval: 補充間隔
    algorithm: |
      tokens = min(capacity, tokens + (elapsed_time * refill_rate))
      if tokens >= 1:
        tokens -= 1
        allow request
      else:
        reject request

  leaky_bucket:
    description: リーキーバケット
    implementation: Redis Queue
    accuracy: 高
    memory: 中
    use_case: 一定レートでの処理が必要な場合
    algorithm: |
      if queue.size < capacity:
        queue.push(request)
        allow request (will be processed at constant rate)
      else:
        reject request

  fixed_window:
    description: 固定ウィンドウ
    implementation: Redis INCR + EXPIRE
    accuracy: 低（境界問題）
    memory: 最低
    use_case: シンプルな制限、厳密性不要

  chosen_algorithm:
    primary: sliding_window_counter
    rationale:
      - メモリ効率が良い
      - 十分な精度
      - 実装がシンプル
      - Redis との相性が良い
```

### 6.2 クォータ管理

```yaml
quota_management:
  tiers:
    anonymous:
      description: 未認証ユーザー
      identifier: IP address
      limits:
        requests_per_minute: 30
        requests_per_hour: 500
        requests_per_day: 5000
      features:
        - 検索（制限付き）
        - 商品閲覧

    free:
      description: 無料会員
      identifier: User ID
      limits:
        requests_per_minute: 60
        requests_per_hour: 2000
        requests_per_day: 20000
      features:
        - 全機能（制限付き）
        - 予約（月5件まで）

    premium:
      description: プレミアム会員
      identifier: User ID
      limits:
        requests_per_minute: 300
        requests_per_hour: 10000
        requests_per_day: 100000
      features:
        - 全機能（制限緩和）
        - 予約無制限
        - 優先サポート

    partner:
      description: パートナーAPI
      identifier: API Key
      limits:
        requests_per_minute: 1000
        requests_per_hour: 50000
        requests_per_day: 500000
      features:
        - 全機能
        - バルクAPI
        - Webhook

    enterprise:
      description: エンタープライズ
      identifier: API Key
      limits:
        custom: true
        negotiated: true
      features:
        - カスタム制限
        - SLA保証
        - 専用サポート

  endpoint_specific_limits:
    search:
      path: /v1/search/*
      limit: 100/minute
      rationale: 検索はコストが高い

    auth_login:
      path: /v1/auth/login
      limit: 10/minute
      rationale: ブルートフォース防止

    auth_register:
      path: /v1/auth/register
      limit: 3/hour
      rationale: スパム防止

    payments:
      path: /v1/payments/*
      limit: 30/minute
      rationale: 決済は慎重に

    bookings_create:
      path: POST /v1/bookings
      limit: 10/minute
      rationale: 在庫確保の制御

    recommendations:
      path: /v1/recommendations/*
      limit: 60/minute
      rationale: ML推論のコスト

  quota_headers:
    response_headers:
      X-RateLimit-Limit: 制限値
      X-RateLimit-Remaining: 残りリクエスト数
      X-RateLimit-Reset: リセット時刻（Unix timestamp）
      X-RateLimit-Policy: 適用ポリシー名
      Retry-After: 再試行までの秒数（429時）

    example:
      X-RateLimit-Limit: "60"
      X-RateLimit-Remaining: "45"
      X-RateLimit-Reset: "1705661700"
      X-RateLimit-Policy: "free-tier"
```

### 6.3 バックプレッシャー処理

```yaml
backpressure_handling:
  strategies:
    gradual_degradation:
      description: 段階的なサービス低下
      thresholds:
        warning: 70% capacity
        degraded: 85% capacity
        critical: 95% capacity
      actions:
        warning:
          - 非必須機能の遅延
          - キャッシュTTL延長
          - ログレベル低下
        degraded:
          - 推奨機能の停止
          - 検索結果の簡略化
          - 新規登録の制限
        critical:
          - 読み取り専用モード
          - 静的コンテンツへのフォールバック
          - 決済以外の書き込み停止

    load_shedding:
      description: 過負荷時のリクエスト破棄
      algorithm: 優先度ベース
      priorities:
        1: 決済処理
        2: 予約確定
        3: 認証
        4: 予約作成
        5: 検索
        6: 推奨
        7: 分析
      implementation:
        - 優先度キューの使用
        - 低優先度リクエストの早期拒否
        - 503 Service Unavailable + Retry-After

    circuit_breaker:
      description: 障害の波及防止
      states:
        closed: 正常（リクエストを通す）
        open: 異常（リクエストを即座に拒否）
        half_open: 回復確認中（限定的に通す）
      thresholds:
        failure_rate: 50%
        slow_call_rate: 80%
        slow_call_duration: 2s
        minimum_calls: 10
        wait_duration_in_open: 60s

    adaptive_throttling:
      description: 動的なレート調整
      metrics:
        - CPU使用率
        - メモリ使用率
        - レスポンスタイム
        - エラー率
      adjustment:
        increase_condition: all metrics < 50%
        increase_amount: 10%
        decrease_condition: any metric > 80%
        decrease_amount: 20%

  client_handling:
    retry_guidance:
      header: Retry-After
      exponential_backoff:
        initial: 1s
        multiplier: 2
        max: 60s
        jitter: 10%

    queue_position:
      description: 待機位置の通知
      header: X-Queue-Position
      use_case: 高トラフィック時のUX改善

    alternative_endpoint:
      description: 代替エンドポイントの提案
      header: Link
      example: |
        Link: </v1/search/cached>; rel="alternate"
```

---

## 第7章：GraphQL戦略

### 7.1 GraphQL vs REST使い分け

```yaml
graphql_vs_rest:
  decision_matrix:
    use_rest:
      - シンプルなCRUD操作
      - キャッシュが重要（CDN活用）
      - 外部パートナー向けAPI
      - サーバー間通信
      - ファイルアップロード
      - Webhook
      - 標準化された操作

    use_graphql:
      - 複雑なクエリ（複数リソースの結合）
      - モバイル向け最適化
      - 柔軟なフィールド選択
      - リアルタイムサブスクリプション
      - 頻繁に変わるUI要件
      - BFF（Backend for Frontend）層

  implementation_strategy:
    phase_1:
      scope: BFF層にGraphQL導入
      purpose: モバイルアプリ向け集約クエリ
      timeline: 3-6ヶ月

    phase_2:
      scope: Webダッシュボード向け
      purpose: 管理画面の柔軟なデータ取得
      timeline: 6-12ヶ月

    phase_3:
      scope: パートナー向けオプション提供
      purpose: 複雑な統合ニーズへの対応
      timeline: 12-18ヶ月

  coexistence:
    architecture: |
      Client
         │
         ├──► GraphQL Gateway (BFF) ──► Microservices (REST/gRPC)
         │
         └──► REST API Gateway ──────► Microservices (REST/gRPC)

    unified_auth: 同一の認証システム（JWT）
    unified_rate_limit: 統合されたレート制限
    unified_monitoring: 統合された可観測性
```

### 7.2 スキーマ設計

```graphql
# TripTrip GraphQL Schema

# スカラー型
scalar DateTime
scalar Date
scalar JSON
scalar Cursor
scalar URL
scalar Currency

# 共通インターフェース
interface Node {
  id: ID!
}

interface Timestamped {
  createdAt: DateTime!
  updatedAt: DateTime!
}

# ページネーション
type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: Cursor
  endCursor: Cursor
}

type PaginationMeta {
  total: Int!
  page: Int!
  limit: Int!
  hasNext: Boolean!
}

# ユーザー
type User implements Node & Timestamped {
  id: ID!
  email: String!
  name: String!
  avatarUrl: URL
  language: String!
  currency: Currency!
  emailVerified: Boolean!
  mfaEnabled: Boolean!
  bookings(
    first: Int
    after: Cursor
    status: [BookingStatus!]
  ): BookingConnection!
  settings: UserSettings!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type UserSettings {
  notification: NotificationSettings!
  privacy: PrivacySettings!
  display: DisplaySettings!
}

type NotificationSettings {
  email: Boolean!
  push: Boolean!
  sms: Boolean!
  marketing: Boolean!
}

# ホテル
type Hotel implements Node & Timestamped {
  id: ID!
  name: String!
  description: String!
  location: Location!
  rating: Rating!
  priceRange: PriceRange!
  amenities: [String!]!
  images: [Image!]!
  hotelClass: Int
  rooms(
    checkIn: Date!
    checkOut: Date!
    guests: Int!
  ): [RoomType!]!
  reviews(first: Int, after: Cursor): ReviewConnection!
  createdAt: DateTime!
  updatedAt: DateTime!
}

type RoomType {
  id: ID!
  name: String!
  description: String!
  maxGuests: Int!
  bedType: String!
  amenities: [String!]!
  images: [Image!]!
  availability: RoomAvailability!
}

type RoomAvailability {
  available: Boolean!
  roomsLeft: Int
  price: Money!
  originalPrice: Money
  discount: Discount
}

type Location {
  lat: Float!
  lng: Float!
  address: String!
  city: String!
  prefecture: String!
  country: String!
  distanceKm: Float
}

type Rating {
  average: Float!
  count: Int!
  distribution: [RatingDistribution!]!
}

type RatingDistribution {
  stars: Int!
  count: Int!
  percentage: Float!
}

type PriceRange {
  min: Int!
  max: Int!
  currency: Currency!
}

type Money {
  amount: Int!
  currency: Currency!
  formatted: String!
}

type Image {
  url: URL!
  alt: String
  width: Int
  height: Int
}

# アトラクション
type Attraction implements Node & Timestamped {
  id: ID!
  name: String!
  description: String!
  category: AttractionCategory!
  location: Location!
  rating: Rating!
  operatingHours: [OperatingHour!]!
  ticketTypes: [TicketType!]!
  images: [Image!]!
  reviews(first: Int, after: Cursor): ReviewConnection!
  createdAt: DateTime!
  updatedAt: DateTime!
}

enum AttractionCategory {
  THEME_PARK
  MUSEUM
  TOUR
  EXPERIENCE
  SHOW
  NATURE
}

type TicketType {
  id: ID!
  name: String!
  description: String!
  price: Money!
  availability(date: Date!): TicketAvailability!
  timeSlots(date: Date!): [TimeSlot!]!
}

type TimeSlot {
  id: ID!
  startTime: String!
  endTime: String!
  available: Boolean!
  remainingCapacity: Int
}

# 予約
type Booking implements Node & Timestamped {
  id: ID!
  bookingNumber: String!
  status: BookingStatus!
  bookingType: BookingType!
  resource: BookableResource!
  dates: BookingDates!
  quantity: Int!
  pricing: BookingPricing!
  guestInfo: GuestInfo!
  payment: PaymentInfo
  cancellationPolicy: CancellationPolicy!
  timeline: [BookingEvent!]!
  createdAt: DateTime!
  updatedAt: DateTime!
}

union BookableResource = Hotel | Attraction | Product

enum BookingStatus {
  PENDING
  CONFIRMED
  CANCELLED
  COMPLETED
  REFUNDED
}

enum BookingType {
  HOTEL
  ATTRACTION
  PRODUCT
  RENTAL
}

type BookingDates {
  checkIn: Date
  checkOut: Date
  visitDate: Date
  timeSlot: TimeSlot
}

type BookingPricing {
  subtotal: Money!
  tax: Money!
  serviceFee: Money!
  discount: Money
  total: Money!
  breakdown: [PriceBreakdownItem!]!
}

# 検索
type SearchResult {
  hotels: HotelConnection
  attractions: AttractionConnection
  products: ProductConnection
  suggestions: [SearchSuggestion!]!
}

type HotelConnection {
  edges: [HotelEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
  facets: HotelFacets!
}

type HotelEdge {
  node: Hotel!
  cursor: Cursor!
  score: Float
}

type HotelFacets {
  amenities: [FacetValue!]!
  priceRanges: [PriceRangeFacet!]!
  ratings: [RatingFacet!]!
  hotelClasses: [FacetValue!]!
}

type FacetValue {
  value: String!
  count: Int!
}

# クエリ
type Query {
  # ユーザー
  me: User
  user(id: ID!): User

  # ホテル
  hotel(id: ID!): Hotel
  hotels(
    query: String
    location: String
    checkIn: Date
    checkOut: Date
    guests: Int
    minPrice: Int
    maxPrice: Int
    amenities: [String!]
    rating: Float
    sort: HotelSortInput
    first: Int
    after: Cursor
  ): HotelConnection!

  # アトラクション
  attraction(id: ID!): Attraction
  attractions(
    query: String
    location: String
    category: [AttractionCategory!]
    date: Date
    minPrice: Int
    maxPrice: Int
    sort: AttractionSortInput
    first: Int
    after: Cursor
  ): AttractionConnection!

  # 予約
  booking(id: ID!): Booking
  bookings(
    status: [BookingStatus!]
    bookingType: [BookingType!]
    dateFrom: Date
    dateTo: Date
    first: Int
    after: Cursor
  ): BookingConnection!

  # 検索
  search(
    query: String!
    types: [SearchType!]
    location: String
    dateFrom: Date
    dateTo: Date
    limit: Int
  ): SearchResult!

  # 推奨
  recommendations(
    types: [RecommendationType!]
    context: RecommendationContext
    limit: Int
  ): RecommendationResult!
}

# ミューテーション
type Mutation {
  # 認証
  register(input: RegisterInput!): AuthPayload!
  login(input: LoginInput!): AuthPayload!
  refreshToken(refreshToken: String!): AuthPayload!
  logout(allDevices: Boolean): Boolean!

  # プロフィール
  updateProfile(input: UpdateProfileInput!): User!
  updateSettings(input: UpdateSettingsInput!): UserSettings!

  # 予約
  createBooking(input: CreateBookingInput!): CreateBookingPayload!
  confirmBooking(id: ID!, paymentIntentId: String!): Booking!
  cancelBooking(id: ID!, reason: CancelReason!): CancelBookingPayload!
  modifyBooking(id: ID!, input: ModifyBookingInput!): ModifyBookingPayload!

  # カート
  addToCart(input: AddToCartInput!): Cart!
  removeFromCart(itemId: ID!): Cart!
  updateCartItem(itemId: ID!, quantity: Int!): Cart!
  clearCart: Cart!
}

# サブスクリプション（リアルタイム）
type Subscription {
  bookingStatusChanged(bookingId: ID!): Booking!
  priceAlert(hotelId: ID!, targetPrice: Int!): PriceAlertPayload!
  inventoryUpdate(resourceId: ID!): InventoryUpdatePayload!
}
```

### 7.3 パフォーマンス最適化

```yaml
graphql_performance:
  query_complexity:
    analysis: true
    max_complexity: 1000
    max_depth: 10
    cost_calculation:
      base: 1
      per_field: 1
      per_connection: 10
      per_resolver: 5
    rejection_response:
      status: 400
      message: "Query complexity exceeds maximum allowed"

  dataloader:
    description: N+1問題の解決
    implementation: |
      // DataLoader使用例
      const hotelLoader = new DataLoader(async (ids) => {
        const hotels = await hotelService.findByIds(ids);
        return ids.map(id => hotels.find(h => h.id === id));
      }, {
        batch: true,
        maxBatchSize: 100,
        cache: true,
      });

      // Resolver内で使用
      resolve: (booking, args, context) => {
        return context.loaders.hotel.load(booking.hotelId);
      }

  query_caching:
    automatic_persisted_queries:
      enabled: true
      storage: Redis
      ttl: 24h
      benefits:
        - ネットワーク帯域の削減
        - パース時間の削減
        - セキュリティ向上（allowlist）

    response_caching:
      enabled: true
      control_header: Cache-Control
      scope:
        public: 認証不要のデータ
        private: ユーザー固有のデータ
      ttl:
        hotels: 5分
        static_content: 1時間
        user_data: 0（キャッシュなし）

  field_level_optimization:
    deferred_fields:
      description: 優先度の低いフィールドの遅延ロード
      directive: "@defer"
      use_case: 推奨、レビューなど

    streaming:
      description: リスト結果のストリーミング
      directive: "@stream"
      use_case: 検索結果、タイムライン

  rate_limiting:
    cost_based:
      enabled: true
      max_cost_per_minute: 10000
      field_costs:
        search: 100
        recommendations: 50
        booking_create: 200
        default: 1
```

---

## 第8章：イベント駆動API

### 8.1 Webhookアーキテクチャ

```yaml
webhook_architecture:
  overview:
    description: リアルタイムイベント通知システム
    delivery: HTTP POST to registered endpoints
    format: CloudEvents 1.0

  event_types:
    booking:
      - booking.created
      - booking.confirmed
      - booking.cancelled
      - booking.modified
      - booking.completed

    payment:
      - payment.intent.created
      - payment.succeeded
      - payment.failed
      - payment.refunded

    inventory:
      - inventory.low
      - inventory.restored
      - inventory.sold_out

    user:
      - user.created
      - user.verified
      - user.subscription.changed

  registration:
    endpoint: POST /v1/webhooks
    request:
      url:
        type: string
        format: uri
        required: true
        validation:
          - HTTPS必須（本番環境）
          - 到達確認テスト実施
      events:
        type: array
        items: string
        required: true
        example: ["booking.confirmed", "payment.succeeded"]
      secret:
        type: string
        description: 署名検証用シークレット（自動生成も可）
      active:
        type: boolean
        default: true
    response:
      201:
        content:
          id: string
          url: string
          events: string[]
          secret: string
          active: boolean
          created_at: string

  payload_structure:
    format: CloudEvents 1.0
    example:
      specversion: "1.0"
      type: "booking.confirmed"
      source: "/services/booking"
      id: "evt_abc123"
      time: "2026-01-19T10:00:00Z"
      datacontenttype: "application/json"
      subject: "booking/bk_xyz789"
      data:
        booking_id: "bk_xyz789"
        booking_number: "TT-12345"
        status: "confirmed"
        user_id: "usr_def456"
        total:
          amount: 50000
          currency: "JPY"
        confirmed_at: "2026-01-19T10:00:00Z"

  security:
    signature:
      header: X-TripTrip-Signature
      algorithm: HMAC-SHA256
      format: "t={timestamp},v1={signature}"
      verification: |
        const payload = `${timestamp}.${requestBody}`;
        const expectedSignature = crypto
          .createHmac('sha256', webhookSecret)
          .update(payload)
          .digest('hex');
        const isValid = crypto.timingSafeEqual(
          Buffer.from(signature),
          Buffer.from(expectedSignature)
        );

    timestamp_tolerance: 5分
    replay_prevention: true

  delivery:
    retry_policy:
      max_attempts: 5
      backoff: exponential
      intervals: [1m, 5m, 30m, 2h, 24h]
      timeout: 30s

    success_criteria:
      status_codes: [200, 201, 202, 204]
      timeout: 30s

    failure_handling:
      - ログ記録
      - リトライキューに追加
      - 連続失敗時の無効化（10回）
      - 管理者への通知

  management:
    endpoints:
      list: GET /v1/webhooks
      get: GET /v1/webhooks/{id}
      update: PATCH /v1/webhooks/{id}
      delete: DELETE /v1/webhooks/{id}
      test: POST /v1/webhooks/{id}/test
      logs: GET /v1/webhooks/{id}/logs
```

### 8.2 イベントスキーマ設計

```yaml
event_schema_design:
  standard: CloudEvents 1.0

  base_schema:
    specversion:
      type: string
      value: "1.0"
      description: CloudEventsバージョン

    id:
      type: string
      format: uuid
      description: イベントの一意識別子

    type:
      type: string
      pattern: "{domain}.{entity}.{action}"
      examples:
        - booking.bookings.created
        - payment.payments.succeeded
        - inventory.items.low

    source:
      type: string
      format: uri-reference
      description: イベント発生元
      example: "/services/booking"

    time:
      type: string
      format: date-time
      description: イベント発生時刻

    datacontenttype:
      type: string
      value: "application/json"

    subject:
      type: string
      description: イベント対象リソース
      example: "booking/bk_123"

    data:
      type: object
      description: イベント固有データ

  domain_events:
    booking_created:
      type: booking.bookings.created
      data:
        booking_id: string
        booking_number: string
        booking_type: string
        user_id: string
        resource_id: string
        status: string
        total:
          amount: number
          currency: string
        created_at: string

    booking_confirmed:
      type: booking.bookings.confirmed
      data:
        booking_id: string
        booking_number: string
        user_id: string
        confirmation_number: string
        payment_id: string
        confirmed_at: string

    booking_cancelled:
      type: booking.bookings.cancelled
      data:
        booking_id: string
        booking_number: string
        user_id: string
        reason: string
        refund:
          amount: number
          currency: string
          status: string
        cancelled_at: string

    payment_succeeded:
      type: payment.payments.succeeded
      data:
        payment_id: string
        booking_id: string
        user_id: string
        amount: number
        currency: string
        payment_method:
          type: string
          last_four: string
        processed_at: string

    payment_failed:
      type: payment.payments.failed
      data:
        payment_id: string
        booking_id: string
        user_id: string
        amount: number
        currency: string
        failure_reason: string
        failure_code: string
        failed_at: string

    inventory_low:
      type: inventory.items.low
      data:
        resource_type: string
        resource_id: string
        resource_name: string
        current_quantity: number
        threshold: number
        date: string

  schema_evolution:
    rules:
      - 新フィールドの追加は許可
      - 既存フィールドの削除は禁止
      - 型の変更は禁止
      - 必須フィールドの追加は禁止

    versioning:
      approach: スキーマ内バージョン
      field: data.schema_version
      migration: コンシューマー側で対応

  schema_registry:
    tool: Confluent Schema Registry
    format: JSON Schema
    compatibility_mode: FORWARD
```

---

## 第9章：API文書化・開発者体験

### 9.1 ドキュメント戦略

```yaml
documentation_strategy:
  tools:
    primary: Redoc
    secondary: Swagger UI
    hosting: https://developers.triptrip.com

  structure:
    getting_started:
      - 概要
      - クイックスタート
      - 認証方法
      - レート制限
      - エラーハンドリング

    api_reference:
      - エンドポイント一覧
      - リクエスト/レスポンス
      - コード例
      - パラメータ詳細

    guides:
      - 予約フローの実装
      - 決済統合
      - Webhook設定
      - ベストプラクティス

    sdks:
      - TypeScript/JavaScript
      - Dart (Flutter)
      - Python
      - Go

    changelog:
      - バージョン履歴
      - Breaking changes
      - 非推奨API

  code_examples:
    languages:
      - curl
      - JavaScript
      - TypeScript
      - Python
      - Dart
      - Go

    format: |
      # ホテル検索の例

      ## cURL
      ```bash
      curl -X GET "https://api.triptrip.com/v1/search/hotels?location=tokyo&check_in=2026-02-01&check_out=2026-02-03" \
        -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
        -H "Accept-Language: ja"
      ```

      ## TypeScript
      ```typescript
      import { TripTripClient } from '@triptrip/sdk';

      const client = new TripTripClient({ accessToken: 'YOUR_ACCESS_TOKEN' });

      const hotels = await client.hotels.search({
        location: 'tokyo',
        checkIn: '2026-02-01',
        checkOut: '2026-02-03',
      });
      ```

  interactive_features:
    try_it_out:
      enabled: true
      environments:
        - name: Sandbox
          url: https://api.sandbox.triptrip.com
        - name: Production (Read-only)
          url: https://api.triptrip.com

    request_builder:
      enabled: true
      features:
        - パラメータUI
        - 認証ヘルパー
        - レスポンスビューア
```

### 9.2 開発者ポータル

```yaml
developer_portal:
  url: https://developers.triptrip.com

  features:
    dashboard:
      - API使用状況
      - レート制限ステータス
      - エラー率
      - レスポンスタイム

    api_keys:
      - 生成・管理
      - スコープ設定
      - ローテーション
      - 使用ログ

    webhooks:
      - エンドポイント登録
      - イベント選択
      - テスト送信
      - 配信ログ

    sandbox:
      - テスト環境
      - モックデータ
      - テストユーザー
      - 決済テスト

    documentation:
      - APIリファレンス
      - チュートリアル
      - ベストプラクティス
      - FAQ

    support:
      - フォーラム
      - チケット
      - Slack（パートナー向け）
      - ステータスページ

  onboarding:
    steps:
      1. アカウント作成
      2. アプリケーション登録
      3. APIキー取得
      4. Sandboxでテスト
      5. 本番キー申請
      6. レビュー・承認
      7. 本番稼働

  sdk_distribution:
    npm: "@triptrip/sdk"
    pypi: "triptrip"
    pub: "triptrip_sdk"
    go: "github.com/triptrip/triptrip-go"
```

---

## 第10章：実装ロードマップ

### 10.1 フェーズ別計画

```yaml
implementation_roadmap:
  phase_1:
    name: 基盤構築
    duration: 3ヶ月
    objectives:
      - OpenAPI仕様の標準化
      - API Gateway導入（Kong）
      - 認証基盤の強化
      - レート制限の実装
    deliverables:
      - OpenAPI 3.1仕様テンプレート
      - Kong Gateway設定
      - JWT認証強化
      - 基本的なレート制限
    success_criteria:
      - 全エンドポイントのOpenAPI化
      - P99レイテンシ < 200ms
      - 99.9%可用性

  phase_2:
    name: 機能拡張
    duration: 3ヶ月
    objectives:
      - コアAPIの実装強化
      - バージョニング戦略の適用
      - Webhook基盤構築
      - 開発者ポータルβ
    deliverables:
      - 検索API v1完成
      - 予約API v1完成
      - Webhook配信システム
      - ドキュメントポータル
    success_criteria:
      - 全コアAPIの本番稼働
      - 契約テストカバレッジ80%
      - ドキュメント完成度90%

  phase_3:
    name: 高度化
    duration: 6ヶ月
    objectives:
      - GraphQL BFF導入
      - 高度なパーソナライゼーション
      - パートナーAPI公開
      - グローバル展開準備
    deliverables:
      - GraphQL Gateway
      - 推奨API v1
      - パートナーAPI仕様
      - マルチリージョン対応
    success_criteria:
      - GraphQL本番稼働
      - パートナー10社統合
      - 3リージョン展開

  phase_4:
    name: スケーラビリティ
    duration: 6ヶ月
    objectives:
      - 100万MAU対応
      - エッジコンピューティング
      - 高度なキャッシング
      - リアルタイム機能
    deliverables:
      - エッジAPI
      - 分散キャッシュ
      - WebSocketゲートウェイ
      - イベントストリーミング
    success_criteria:
      - P99レイテンシ < 100ms（グローバル）
      - 100,000 req/s処理能力
      - 99.99%可用性
```

---

## 第11章：文書間参照 & 統合ポイント

### 11.1 関連文書

```yaml
document_references:
  prerequisites:
    - doc_id: Doc-TV-001
      title: TripTrip技術ビジョンとアーキテクチャ原則
      relationship: 技術原則の定義
      key_sections:
        - APIファースト原則
        - セキュリティ原則
        - パフォーマンス目標

    - doc_id: Doc-SA-001
      title: TripTripシステムアーキテクチャ概要
      relationship: 全体アーキテクチャの定義
      key_sections:
        - API Gateway層
        - サービス間通信

    - doc_id: Doc-SA-002
      title: マイクロサービスアーキテクチャ
      relationship: サービス分割とAPI境界
      key_sections:
        - サービス境界定義
        - REST/gRPC設計
        - イベント駆動アーキテクチャ

  related_documents:
    - doc_id: Doc-SA-004
      title: リアルタイム・協調機能アーキテクチャ
      relationship: WebSocket、リアルタイムAPI

    - doc_id: Doc-SA-005
      title: 検索・推奨アーキテクチャ
      relationship: 検索API、推奨APIの詳細実装

    - doc_id: Doc-SEC-001
      title: セキュリティアーキテクチャ
      relationship: API認証・認可の詳細

    - doc_id: Doc-INF-001
      title: インフラストラクチャ設計
      relationship: API Gateway、CDN、ロードバランサー
```

### 11.2 統合ポイント

```yaml
integration_points:
  api_gateway:
    component: Kong Gateway
    responsibilities:
      - ルーティング
      - 認証・認可
      - レート制限
      - ログ・モニタリング
    related_docs: [Doc-SA-001, Doc-INF-001]

  authentication:
    component: Auth Service
    responsibilities:
      - OAuth2.0/OIDC
      - JWT発行・検証
      - MFA
    related_docs: [Doc-SEC-001, Doc-SA-002]

  event_bus:
    component: Apache Kafka
    responsibilities:
      - イベント配信
      - Webhook配信
      - サービス間通信
    related_docs: [Doc-SA-002, Doc-SA-004]

  search:
    component: Search Service (Elasticsearch)
    responsibilities:
      - 全文検索
      - ファセット検索
      - オートコンプリート
    related_docs: [Doc-SA-005]

  cache:
    component: Redis Cluster
    responsibilities:
      - APIレスポンスキャッシュ
      - セッション管理
      - レート制限カウンター
    related_docs: [Doc-SA-001, Doc-INF-001]
```

---

## 文書情報

| 項目 | 内容 |
|------|------|
| Document ID | Doc-SA-003 |
| Title | APIアーキテクチャ・設計 |
| Version | 1.0.0 |
| Status | Draft |
| Author | Technical Architecture Agent |
| Created | 2026-01-19 |
| Last Updated | 2026-01-19 |
| Total Lines | ~1,500 |
| Previous Document | Doc-SA-002: マイクロサービスアーキテクチャ |
| Next Document | Doc-SA-004: リアルタイム・協調機能アーキテクチャ |

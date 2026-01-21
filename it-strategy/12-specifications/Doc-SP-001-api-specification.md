# Doc-SP-001: TripTrip API仕様書（エンドポイント一覧）

## エグゼクティブサマリー

本文書は、TripTripプラットフォームの包括的なAPI仕様書です。RESTful API設計原則に基づき、認証、ユーザー管理、検索、予約、決済、推奨の各ドメインにおける全エンドポイントを定義します。OpenAPI 3.0形式での詳細仕様、認証・認可フロー（JWT、OAuth2）、レート制限、エラーハンドリング標準を含む、開発チームが即座に実装可能なレベルの技術仕様を提供します。既存のHono/Node.js/TypeScriptスタックとの完全な整合性を確保し、型安全なAPI開発を実現します。

---

## 第1章：はじめに & API設計原則

### 1.1 文書の目的と範囲

本API仕様書は、TripTripプラットフォームのバックエンドAPIを包括的に定義します。

```yaml
document_scope:
  purpose: TripTrip RESTful APIの完全な技術仕様
  audience:
    - バックエンド開発者
    - フロントエンド開発者
    - QAエンジニア
    - 外部パートナー（将来）

  coverage:
    - 全APIエンドポイント定義
    - リクエスト/レスポンススキーマ
    - 認証・認可仕様
    - エラーハンドリング
    - レート制限
    - OpenAPI 3.0仕様

  technology_stack:
    runtime: Node.js 18+
    framework: Hono 4.8.3
    validation: Zod 3.25+
    documentation: @hono/zod-openapi
    database: PostgreSQL 16 + Prisma 5.8
```

### 1.2 RESTful設計原則

TripTrip APIは以下のRESTful設計原則に従います。

```yaml
restful_principles:
  1_resource_oriented:
    description: リソース中心の設計
    guidelines:
      - URLはリソースを表す名詞を使用
      - 動詞はHTTPメソッドで表現
      - 階層関係はネストされたURLで表現
    examples:
      good:
        - "GET /users/{id}"
        - "POST /bookings"
        - "GET /hotels/{id}/rooms"
      bad:
        - "GET /getUser"
        - "POST /createBooking"

  2_http_methods:
    description: 適切なHTTPメソッドの使用
    methods:
      GET:
        purpose: リソースの取得
        idempotent: true
        safe: true
      POST:
        purpose: リソースの作成
        idempotent: false
        safe: false
      PUT:
        purpose: リソースの完全置換
        idempotent: true
        safe: false
      PATCH:
        purpose: リソースの部分更新
        idempotent: true
        safe: false
      DELETE:
        purpose: リソースの削除
        idempotent: true
        safe: false

  3_status_codes:
    description: 適切なHTTPステータスコードの使用
    categories:
      2xx_success:
        200: OK - 成功（GET, PUT, PATCH）
        201: Created - リソース作成成功（POST）
        204: No Content - 成功、レスポンスボディなし（DELETE）
      4xx_client_error:
        400: Bad Request - リクエスト形式エラー
        401: Unauthorized - 認証エラー
        403: Forbidden - 認可エラー
        404: Not Found - リソース不在
        409: Conflict - 競合エラー
        422: Unprocessable Entity - バリデーションエラー
        429: Too Many Requests - レート制限超過
      5xx_server_error:
        500: Internal Server Error - サーバー内部エラー
        502: Bad Gateway - 外部サービスエラー
        503: Service Unavailable - サービス一時停止

  4_stateless:
    description: ステートレス通信
    implementation:
      - セッション状態はサーバーに保持しない
      - 認証情報は各リクエストに含める（Bearer Token）
      - 必要なコンテキストはリクエストに含める

  5_hateoas:
    description: ハイパーメディアによる状態遷移
    implementation:
      - レスポンスに関連リソースへのリンクを含める
      - ページネーションリンクを提供
      - 次のアクションへのリンクを提供
```

### 1.3 バージョニング戦略

```yaml
versioning_strategy:
  approach: URL Path Versioning
  format: "/api/v{major}"
  current_version: v1

  version_lifecycle:
    active:
      v1:
        status: Current
        support_until: 2028-12-31
    deprecated: []
    sunset: []

  version_negotiation:
    primary: URL Path (/api/v1/...)
    fallback: Accept Header (application/vnd.triptrip.v1+json)

  breaking_changes:
    definition:
      - フィールドの削除
      - 必須フィールドの追加
      - データ型の変更
      - エンドポイントの削除
    handling:
      - 新バージョンでのみ適用
      - 最低12ヶ月の移行期間
      - 非推奨警告ヘッダー

  non_breaking_changes:
    examples:
      - オプショナルフィールドの追加
      - 新しいエンドポイントの追加
      - レスポンスフィールドの追加
    handling:
      - 現行バージョンに適用
      - 下位互換性を維持

  deprecation_policy:
    notice_period: 6ヶ月
    headers:
      - "Deprecation: true"
      - "Sunset: {date}"
      - "Link: <{new_endpoint}>; rel=\"successor-version\""
```

### 1.4 認証・認可概要

```yaml
authentication_overview:
  primary_method: JWT (JSON Web Token)
  token_types:
    access_token:
      lifetime: 1時間
      storage: メモリ（クライアント）
      refresh: Refresh Tokenで更新
    refresh_token:
      lifetime: 30日
      storage: HttpOnly Cookie / Secure Storage
      rotation: 使用時に新規発行

  oauth2_providers:
    - provider: Google
      scopes: [email, profile]
    - provider: Apple
      scopes: [email, name]
    - provider: Facebook
      scopes: [email, public_profile]

  authorization_model:
    type: Role-Based Access Control (RBAC)
    roles:
      - guest: 未認証ユーザー
      - user: 一般登録ユーザー
      - premium: プレミアム会員
      - partner: ビジネスパートナー
      - admin: 管理者

  security_measures:
    - HTTPS必須
    - CORS制限
    - レート制限
    - リクエスト署名（オプション）
```

---

## 第2章：認証API

### 2.1 認証エンドポイント概要

```yaml
auth_endpoints:
  base_path: /api/v1/auth
  endpoints:
    - method: POST
      path: /register
      description: 新規ユーザー登録
    - method: POST
      path: /login
      description: メール/パスワードログイン
    - method: POST
      path: /refresh
      description: アクセストークンリフレッシュ
    - method: POST
      path: /logout
      description: ログアウト
    - method: POST
      path: /password/reset-request
      description: パスワードリセット要求
    - method: POST
      path: /password/reset
      description: パスワードリセット実行
    - method: GET
      path: /oauth/{provider}
      description: OAuth認証開始
    - method: GET
      path: /oauth/{provider}/callback
      description: OAuthコールバック
```

### 2.2 POST /auth/register - ユーザー登録

```yaml
endpoint:
  method: POST
  path: /api/v1/auth/register
  description: 新規ユーザーアカウントを作成
  authentication: 不要
  rate_limit: 5回/分/IP

request:
  content_type: application/json
  body:
    type: object
    required:
      - email
      - password
      - name
    properties:
      email:
        type: string
        format: email
        maxLength: 255
        description: ユーザーのメールアドレス
        example: "user@example.com"
      password:
        type: string
        format: password
        minLength: 8
        maxLength: 128
        pattern: "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{8,}$"
        description: パスワード（大文字、小文字、数字、特殊文字を含む8文字以上）
      name:
        type: string
        minLength: 1
        maxLength: 100
        description: 表示名
        example: "山田 太郎"
      language:
        type: string
        enum: [ja, en]
        default: ja
        description: 優先言語
      currency:
        type: string
        enum: [JPY, USD, EUR]
        default: JPY
        description: 優先通貨
      marketing_consent:
        type: boolean
        default: false
        description: マーケティングメール受信同意

responses:
  201:
    description: ユーザー登録成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
              example: true
            data:
              type: object
              properties:
                user:
                  $ref: "#/components/schemas/User"
                access_token:
                  type: string
                  description: JWTアクセストークン
                refresh_token:
                  type: string
                  description: リフレッシュトークン
                expires_in:
                  type: integer
                  description: アクセストークン有効期限（秒）
                  example: 3600

  400:
    description: リクエスト形式エラー
    content:
      application/json:
        schema:
          $ref: "#/components/schemas/ErrorResponse"
        example:
          success: false
          error:
            code: "VALIDATION_ERROR"
            message: "入力内容に誤りがあります"
            details:
              - field: "password"
                message: "パスワードは8文字以上で、大文字、小文字、数字、特殊文字を含む必要があります"

  409:
    description: メールアドレス重複
    content:
      application/json:
        schema:
          $ref: "#/components/schemas/ErrorResponse"
        example:
          success: false
          error:
            code: "EMAIL_ALREADY_EXISTS"
            message: "このメールアドレスは既に登録されています"

  429:
    description: レート制限超過
    headers:
      Retry-After:
        schema:
          type: integer
        description: 再試行までの秒数
    content:
      application/json:
        schema:
          $ref: "#/components/schemas/ErrorResponse"
```

### 2.3 POST /auth/login - ログイン

```yaml
endpoint:
  method: POST
  path: /api/v1/auth/login
  description: メールアドレスとパスワードでログイン
  authentication: 不要
  rate_limit: 10回/分/IP、5回失敗後15分ロックアウト

request:
  content_type: application/json
  body:
    type: object
    required:
      - email
      - password
    properties:
      email:
        type: string
        format: email
        description: 登録済みメールアドレス
      password:
        type: string
        format: password
        description: パスワード
      remember_me:
        type: boolean
        default: false
        description: ログイン状態を延長（リフレッシュトークン有効期限を90日に）
      device_info:
        type: object
        description: デバイス情報（オプション）
        properties:
          device_id:
            type: string
          device_type:
            type: string
            enum: [ios, android, web]
          device_name:
            type: string

responses:
  200:
    description: ログイン成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
              example: true
            data:
              type: object
              properties:
                user:
                  $ref: "#/components/schemas/User"
                access_token:
                  type: string
                refresh_token:
                  type: string
                expires_in:
                  type: integer
                  example: 3600
                token_type:
                  type: string
                  example: "Bearer"

  401:
    description: 認証失敗
    content:
      application/json:
        schema:
          $ref: "#/components/schemas/ErrorResponse"
        examples:
          invalid_credentials:
            value:
              success: false
              error:
                code: "INVALID_CREDENTIALS"
                message: "メールアドレスまたはパスワードが正しくありません"
          account_locked:
            value:
              success: false
              error:
                code: "ACCOUNT_LOCKED"
                message: "アカウントがロックされています。15分後に再試行してください"
                details:
                  unlock_at: "2026-01-21T10:30:00Z"
```

### 2.4 POST /auth/refresh - トークンリフレッシュ

```yaml
endpoint:
  method: POST
  path: /api/v1/auth/refresh
  description: リフレッシュトークンを使用してアクセストークンを更新
  authentication: 不要（リフレッシュトークン必須）
  rate_limit: 30回/時間/ユーザー

request:
  content_type: application/json
  body:
    type: object
    required:
      - refresh_token
    properties:
      refresh_token:
        type: string
        description: 有効なリフレッシュトークン

responses:
  200:
    description: トークンリフレッシュ成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
              example: true
            data:
              type: object
              properties:
                access_token:
                  type: string
                  description: 新しいアクセストークン
                refresh_token:
                  type: string
                  description: 新しいリフレッシュトークン（ローテーション）
                expires_in:
                  type: integer
                  example: 3600

  401:
    description: リフレッシュトークン無効
    content:
      application/json:
        schema:
          $ref: "#/components/schemas/ErrorResponse"
        example:
          success: false
          error:
            code: "INVALID_REFRESH_TOKEN"
            message: "リフレッシュトークンが無効または期限切れです"
```

### 2.5 POST /auth/logout - ログアウト

```yaml
endpoint:
  method: POST
  path: /api/v1/auth/logout
  description: 現在のセッションをログアウト
  authentication: Bearer Token
  rate_limit: 100回/時間/ユーザー

request:
  headers:
    Authorization:
      required: true
      schema:
        type: string
        pattern: "^Bearer .+"
  body:
    type: object
    properties:
      all_devices:
        type: boolean
        default: false
        description: すべてのデバイスからログアウト

responses:
  200:
    description: ログアウト成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
              example: true
            message:
              type: string
              example: "ログアウトしました"

  401:
    description: 認証エラー
```

### 2.6 OAuth2連携

```yaml
oauth_endpoints:
  google:
    initiate:
      method: GET
      path: /api/v1/auth/oauth/google
      description: Google OAuth認証を開始
      query_parameters:
        redirect_uri:
          type: string
          required: true
          description: 認証後のリダイレクト先
      response:
        302:
          description: Google認証ページにリダイレクト
          headers:
            Location:
              schema:
                type: string
              example: "https://accounts.google.com/o/oauth2/v2/auth?..."

    callback:
      method: GET
      path: /api/v1/auth/oauth/google/callback
      description: Google OAuthコールバック処理
      query_parameters:
        code:
          type: string
          required: true
          description: 認証コード
        state:
          type: string
          required: true
          description: CSRF対策トークン
      response:
        302:
          description: アプリにリダイレクト（トークン付き）
          headers:
            Location:
              schema:
                type: string
              example: "triptrip://auth/callback?access_token=...&refresh_token=..."

  apple:
    initiate:
      method: GET
      path: /api/v1/auth/oauth/apple
      description: Apple Sign In認証を開始
      response:
        302:
          description: Apple認証ページにリダイレクト

    callback:
      method: POST
      path: /api/v1/auth/oauth/apple/callback
      description: Apple Sign Inコールバック処理
      request:
        content_type: application/x-www-form-urlencoded
        body:
          code:
            type: string
            required: true
          id_token:
            type: string
            required: true
          user:
            type: string
            description: 初回認証時のみ（JSON文字列）

  facebook:
    initiate:
      method: GET
      path: /api/v1/auth/oauth/facebook
      description: Facebook OAuth認証を開始

    callback:
      method: GET
      path: /api/v1/auth/oauth/facebook/callback
      description: Facebook OAuthコールバック処理
```

---

## 第3章：ユーザーAPI

### 3.1 ユーザーエンドポイント概要

```yaml
user_endpoints:
  base_path: /api/v1/users
  endpoints:
    - method: GET
      path: /me
      description: 自分のプロフィール取得
      auth: required
    - method: PUT
      path: /me
      description: プロフィール更新
      auth: required
    - method: PATCH
      path: /me
      description: プロフィール部分更新
      auth: required
    - method: DELETE
      path: /me
      description: アカウント削除
      auth: required
    - method: GET
      path: /{id}
      description: 他ユーザー情報取得（公開情報のみ）
      auth: optional
    - method: PUT
      path: /me/preferences
      description: 設定更新
      auth: required
    - method: PUT
      path: /me/password
      description: パスワード変更
      auth: required
    - method: PUT
      path: /me/avatar
      description: アバター画像更新
      auth: required
```

### 3.2 GET /users/me - プロフィール取得

```yaml
endpoint:
  method: GET
  path: /api/v1/users/me
  description: 認証ユーザーの完全なプロフィール情報を取得
  authentication: Bearer Token
  rate_limit: 100回/分/ユーザー

request:
  headers:
    Authorization:
      required: true
      schema:
        type: string
        pattern: "^Bearer .+"

responses:
  200:
    description: プロフィール取得成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
              example: true
            data:
              type: object
              properties:
                id:
                  type: string
                  format: uuid
                  example: "550e8400-e29b-41d4-a716-446655440000"
                email:
                  type: string
                  format: email
                  example: "user@example.com"
                name:
                  type: string
                  example: "山田 太郎"
                avatar_url:
                  type: string
                  format: uri
                  nullable: true
                  example: "https://cdn.triptrip.com/avatars/user123.jpg"
                phone:
                  type: string
                  nullable: true
                  example: "+81-90-1234-5678"
                language:
                  type: string
                  enum: [ja, en]
                  example: "ja"
                currency:
                  type: string
                  enum: [JPY, USD, EUR]
                  example: "JPY"
                role:
                  type: string
                  enum: [user, premium, partner, admin]
                  example: "user"
                email_verified:
                  type: boolean
                  example: true
                phone_verified:
                  type: boolean
                  example: false
                created_at:
                  type: string
                  format: date-time
                  example: "2026-01-15T10:30:00Z"
                updated_at:
                  type: string
                  format: date-time
                  example: "2026-01-20T14:20:00Z"
                preferences:
                  type: object
                  properties:
                    notifications:
                      type: object
                      properties:
                        email:
                          type: boolean
                          example: true
                        push:
                          type: boolean
                          example: true
                        marketing:
                          type: boolean
                          example: false
                    travel_preferences:
                      type: object
                      properties:
                        preferred_airlines:
                          type: array
                          items:
                            type: string
                          example: ["ANA", "JAL"]
                        preferred_hotel_chains:
                          type: array
                          items:
                            type: string
                          example: ["Marriott", "Hilton"]
                        dietary_restrictions:
                          type: array
                          items:
                            type: string
                          example: ["vegetarian"]
                statistics:
                  type: object
                  properties:
                    total_bookings:
                      type: integer
                      example: 15
                    total_reviews:
                      type: integer
                      example: 8
                    member_since_days:
                      type: integer
                      example: 365

  401:
    description: 認証エラー
    content:
      application/json:
        schema:
          $ref: "#/components/schemas/ErrorResponse"
```

### 3.3 PUT /users/me - プロフィール更新

```yaml
endpoint:
  method: PUT
  path: /api/v1/users/me
  description: プロフィール情報を完全置換
  authentication: Bearer Token
  rate_limit: 20回/分/ユーザー

request:
  headers:
    Authorization:
      required: true
  content_type: application/json
  body:
    type: object
    required:
      - name
      - language
      - currency
    properties:
      name:
        type: string
        minLength: 1
        maxLength: 100
        example: "山田 太郎"
      phone:
        type: string
        pattern: "^\\+?[0-9\\-]+$"
        nullable: true
        example: "+81-90-1234-5678"
      language:
        type: string
        enum: [ja, en]
      currency:
        type: string
        enum: [JPY, USD, EUR]
      date_of_birth:
        type: string
        format: date
        nullable: true
        example: "1990-05-15"
      gender:
        type: string
        enum: [male, female, other, prefer_not_to_say]
        nullable: true
      nationality:
        type: string
        maxLength: 2
        description: ISO 3166-1 alpha-2
        example: "JP"
      passport_country:
        type: string
        maxLength: 2
        nullable: true

responses:
  200:
    description: プロフィール更新成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
              example: true
            data:
              $ref: "#/components/schemas/User"

  400:
    description: バリデーションエラー

  401:
    description: 認証エラー
```

### 3.4 GET /users/{id} - 他ユーザー情報取得

```yaml
endpoint:
  method: GET
  path: /api/v1/users/{id}
  description: 指定ユーザーの公開プロフィール情報を取得
  authentication: Optional
  rate_limit: 60回/分/IP

path_parameters:
  id:
    type: string
    format: uuid
    required: true
    description: ユーザーID

responses:
  200:
    description: ユーザー情報取得成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
              example: true
            data:
              type: object
              properties:
                id:
                  type: string
                  format: uuid
                name:
                  type: string
                  example: "山田 太郎"
                avatar_url:
                  type: string
                  format: uri
                  nullable: true
                member_since:
                  type: string
                  format: date
                  example: "2025-01-15"
                reviews_count:
                  type: integer
                  example: 8
                average_rating:
                  type: number
                  format: float
                  example: 4.5

  404:
    description: ユーザーが見つからない
    content:
      application/json:
        schema:
          $ref: "#/components/schemas/ErrorResponse"
        example:
          success: false
          error:
            code: "USER_NOT_FOUND"
            message: "指定されたユーザーは存在しません"
```

### 3.5 PUT /users/me/preferences - 設定更新

```yaml
endpoint:
  method: PUT
  path: /api/v1/users/me/preferences
  description: ユーザー設定を更新
  authentication: Bearer Token
  rate_limit: 20回/分/ユーザー

request:
  content_type: application/json
  body:
    type: object
    properties:
      notifications:
        type: object
        properties:
          email:
            type: boolean
            description: メール通知
          push:
            type: boolean
            description: プッシュ通知
          sms:
            type: boolean
            description: SMS通知
          marketing:
            type: boolean
            description: マーケティング通知
          booking_reminders:
            type: boolean
            description: 予約リマインダー
          price_alerts:
            type: boolean
            description: 価格アラート
      travel_preferences:
        type: object
        properties:
          preferred_airlines:
            type: array
            items:
              type: string
            maxItems: 10
          preferred_hotel_chains:
            type: array
            items:
              type: string
            maxItems: 10
          preferred_room_type:
            type: string
            enum: [single, double, twin, suite]
          smoking_preference:
            type: string
            enum: [non_smoking, smoking, no_preference]
          dietary_restrictions:
            type: array
            items:
              type: string
              enum: [vegetarian, vegan, halal, kosher, gluten_free, none]
          accessibility_needs:
            type: array
            items:
              type: string
              enum: [wheelchair, hearing, visual, none]
      display_preferences:
        type: object
        properties:
          theme:
            type: string
            enum: [light, dark, system]
          compact_view:
            type: boolean
          show_prices_with_tax:
            type: boolean

responses:
  200:
    description: 設定更新成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
              example: true
            data:
              type: object
              description: 更新後の設定
```

---

## 第4章：検索API

### 4.1 検索エンドポイント概要

```yaml
search_endpoints:
  base_path: /api/v1/search
  endpoints:
    - method: GET
      path: /destinations
      description: 目的地検索
      auth: optional
    - method: GET
      path: /hotels
      description: ホテル検索
      auth: optional
    - method: GET
      path: /activities
      description: アクティビティ検索
      auth: optional
    - method: GET
      path: /products
      description: 商品検索
      auth: optional
    - method: GET
      path: /suggestions
      description: 検索サジェスト
      auth: optional
    - method: GET
      path: /unified
      description: 統合検索
      auth: optional
```

### 4.2 GET /search/destinations - 目的地検索

```yaml
endpoint:
  method: GET
  path: /api/v1/search/destinations
  description: 目的地を検索
  authentication: Optional
  rate_limit: 100回/分/IP
  caching: 5分

query_parameters:
  q:
    type: string
    required: true
    minLength: 2
    maxLength: 100
    description: 検索クエリ
    example: "京都"
  country:
    type: string
    maxLength: 2
    description: 国コード（ISO 3166-1 alpha-2）
    example: "JP"
  type:
    type: string
    enum: [city, region, country, airport, landmark]
    description: 目的地タイプ
  limit:
    type: integer
    minimum: 1
    maximum: 50
    default: 10
    description: 取得件数
  language:
    type: string
    enum: [ja, en]
    default: ja
    description: 結果の言語

responses:
  200:
    description: 検索成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
              example: true
            data:
              type: object
              properties:
                destinations:
                  type: array
                  items:
                    type: object
                    properties:
                      id:
                        type: string
                        format: uuid
                      name:
                        type: string
                        example: "京都市"
                      name_en:
                        type: string
                        example: "Kyoto City"
                      type:
                        type: string
                        enum: [city, region, country, airport, landmark]
                      country:
                        type: string
                        example: "JP"
                      country_name:
                        type: string
                        example: "日本"
                      region:
                        type: string
                        example: "近畿"
                      coordinates:
                        type: object
                        properties:
                          latitude:
                            type: number
                            example: 35.0116
                          longitude:
                            type: number
                            example: 135.7681
                      image_url:
                        type: string
                        format: uri
                      popularity_score:
                        type: number
                        example: 95.5
                total_count:
                  type: integer
                  example: 5
```

### 4.3 GET /search/hotels - ホテル検索

```yaml
endpoint:
  method: GET
  path: /api/v1/search/hotels
  description: ホテルを検索
  authentication: Optional
  rate_limit: 60回/分/IP
  caching: 1分

query_parameters:
  destination_id:
    type: string
    format: uuid
    description: 目的地ID
  q:
    type: string
    maxLength: 100
    description: キーワード検索
  check_in:
    type: string
    format: date
    required: true
    description: チェックイン日
    example: "2026-02-01"
  check_out:
    type: string
    format: date
    required: true
    description: チェックアウト日
    example: "2026-02-03"
  guests:
    type: integer
    minimum: 1
    maximum: 10
    default: 2
    description: 宿泊人数
  rooms:
    type: integer
    minimum: 1
    maximum: 5
    default: 1
    description: 部屋数
  price_min:
    type: integer
    minimum: 0
    description: 最低価格（1泊あたり）
  price_max:
    type: integer
    description: 最高価格（1泊あたり）
  star_rating:
    type: array
    items:
      type: integer
      minimum: 1
      maximum: 5
    description: 星評価フィルター
    example: [4, 5]
  amenities:
    type: array
    items:
      type: string
      enum: [wifi, parking, pool, gym, spa, restaurant, bar, room_service, pet_friendly, breakfast_included]
    description: 設備フィルター
  property_type:
    type: array
    items:
      type: string
      enum: [hotel, ryokan, minshuku, hostel, apartment, resort]
    description: 施設タイプ
  sort_by:
    type: string
    enum: [relevance, price_low, price_high, rating, distance, popularity]
    default: relevance
    description: ソート順
  page:
    type: integer
    minimum: 1
    default: 1
  limit:
    type: integer
    minimum: 1
    maximum: 100
    default: 20
  latitude:
    type: number
    format: float
    description: 検索中心の緯度（距離ソート用）
  longitude:
    type: number
    format: float
    description: 検索中心の経度
  radius_km:
    type: number
    format: float
    default: 10
    description: 検索半径（km）

responses:
  200:
    description: 検索成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
              example: true
            data:
              type: object
              properties:
                hotels:
                  type: array
                  items:
                    type: object
                    properties:
                      id:
                        type: string
                        format: uuid
                      name:
                        type: string
                        example: "京都グランドホテル"
                      description:
                        type: string
                        maxLength: 500
                      star_rating:
                        type: integer
                        minimum: 1
                        maximum: 5
                        example: 4
                      user_rating:
                        type: number
                        format: float
                        example: 4.5
                      review_count:
                        type: integer
                        example: 234
                      address:
                        type: object
                        properties:
                          full:
                            type: string
                          city:
                            type: string
                          postal_code:
                            type: string
                          country:
                            type: string
                      coordinates:
                        type: object
                        properties:
                          latitude:
                            type: number
                          longitude:
                            type: number
                      images:
                        type: array
                        items:
                          type: object
                          properties:
                            url:
                              type: string
                              format: uri
                            alt:
                              type: string
                            is_primary:
                              type: boolean
                      amenities:
                        type: array
                        items:
                          type: string
                      price:
                        type: object
                        properties:
                          amount:
                            type: number
                            example: 15000
                          currency:
                            type: string
                            example: "JPY"
                          per_night:
                            type: boolean
                            example: true
                          includes_tax:
                            type: boolean
                            example: true
                          original_amount:
                            type: number
                            description: 割引前価格
                          discount_percentage:
                            type: integer
                      availability:
                        type: object
                        properties:
                          available:
                            type: boolean
                          rooms_left:
                            type: integer
                            description: 残り部屋数
                pagination:
                  $ref: "#/components/schemas/Pagination"
                facets:
                  type: object
                  properties:
                    star_ratings:
                      type: array
                      items:
                        type: object
                        properties:
                          value:
                            type: integer
                          count:
                            type: integer
                    price_ranges:
                      type: array
                      items:
                        type: object
                        properties:
                          min:
                            type: integer
                          max:
                            type: integer
                          count:
                            type: integer
                    amenities:
                      type: array
                      items:
                        type: object
                        properties:
                          name:
                            type: string
                          count:
                            type: integer
```

### 4.4 GET /search/activities - アクティビティ検索

```yaml
endpoint:
  method: GET
  path: /api/v1/search/activities
  description: アクティビティ・体験を検索
  authentication: Optional
  rate_limit: 60回/分/IP
  caching: 1分

query_parameters:
  destination_id:
    type: string
    format: uuid
  q:
    type: string
    maxLength: 100
  date:
    type: string
    format: date
    description: 参加日
  date_from:
    type: string
    format: date
  date_to:
    type: string
    format: date
  participants:
    type: integer
    minimum: 1
    maximum: 20
    default: 1
  category:
    type: array
    items:
      type: string
      enum: [culture, nature, food, adventure, relaxation, nightlife, shopping, tours]
  duration_min:
    type: integer
    description: 最短所要時間（分）
  duration_max:
    type: integer
    description: 最長所要時間（分）
  price_min:
    type: integer
  price_max:
    type: integer
  language:
    type: array
    items:
      type: string
      enum: [ja, en, zh, ko]
    description: 対応言語
  sort_by:
    type: string
    enum: [relevance, price_low, price_high, rating, popularity, duration]
    default: relevance
  page:
    type: integer
    minimum: 1
    default: 1
  limit:
    type: integer
    minimum: 1
    maximum: 100
    default: 20

responses:
  200:
    description: 検索成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
            data:
              type: object
              properties:
                activities:
                  type: array
                  items:
                    type: object
                    properties:
                      id:
                        type: string
                        format: uuid
                      title:
                        type: string
                        example: "京都祇園 着物体験"
                      description:
                        type: string
                      category:
                        type: string
                      duration_minutes:
                        type: integer
                        example: 180
                      rating:
                        type: number
                        format: float
                        example: 4.8
                      review_count:
                        type: integer
                      price:
                        type: object
                        properties:
                          amount:
                            type: number
                          currency:
                            type: string
                          per_person:
                            type: boolean
                      images:
                        type: array
                        items:
                          type: object
                          properties:
                            url:
                              type: string
                            is_primary:
                              type: boolean
                      location:
                        type: object
                        properties:
                          name:
                            type: string
                          address:
                            type: string
                          coordinates:
                            type: object
                            properties:
                              latitude:
                                type: number
                              longitude:
                                type: number
                      available_dates:
                        type: array
                        items:
                          type: string
                          format: date
                        description: 次の利用可能日（最大5件）
                      languages:
                        type: array
                        items:
                          type: string
                      highlights:
                        type: array
                        items:
                          type: string
                pagination:
                  $ref: "#/components/schemas/Pagination"
```

### 4.5 GET /search/suggestions - サジェスト

```yaml
endpoint:
  method: GET
  path: /api/v1/search/suggestions
  description: 検索サジェストを取得（オートコンプリート）
  authentication: Optional
  rate_limit: 200回/分/IP
  caching: 10分

query_parameters:
  q:
    type: string
    required: true
    minLength: 1
    maxLength: 50
    description: 入力テキスト
  types:
    type: array
    items:
      type: string
      enum: [destination, hotel, activity, product]
    default: [destination, hotel, activity]
    description: サジェスト対象タイプ
  limit:
    type: integer
    minimum: 1
    maximum: 20
    default: 10

responses:
  200:
    description: サジェスト取得成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
            data:
              type: object
              properties:
                suggestions:
                  type: array
                  items:
                    type: object
                    properties:
                      type:
                        type: string
                        enum: [destination, hotel, activity, product]
                      id:
                        type: string
                        format: uuid
                      text:
                        type: string
                        description: 表示テキスト
                      highlight:
                        type: string
                        description: マッチ部分をハイライトしたテキスト
                      secondary_text:
                        type: string
                        description: 補足テキスト（例：地域名）
                      image_url:
                        type: string
                        format: uri
                        nullable: true
                query:
                  type: string
                  description: 元のクエリ
```

---

## 第5章：予約API

### 5.1 予約エンドポイント概要

```yaml
booking_endpoints:
  base_path: /api/v1/bookings
  endpoints:
    - method: POST
      path: /
      description: 予約作成
      auth: required
    - method: GET
      path: /
      description: 予約一覧取得
      auth: required
    - method: GET
      path: /{id}
      description: 予約詳細取得
      auth: required
    - method: PUT
      path: /{id}
      description: 予約変更
      auth: required
    - method: DELETE
      path: /{id}
      description: 予約キャンセル
      auth: required
    - method: GET
      path: /{id}/confirmation
      description: 予約確認書取得
      auth: required
```

### 5.2 POST /bookings - 予約作成

```yaml
endpoint:
  method: POST
  path: /api/v1/bookings
  description: 新規予約を作成
  authentication: Bearer Token
  rate_limit: 10回/分/ユーザー

request:
  content_type: application/json
  body:
    type: object
    required:
      - booking_type
      - items
      - contact
    properties:
      booking_type:
        type: string
        enum: [hotel, activity, product, package]
        description: 予約タイプ
      items:
        type: array
        minItems: 1
        maxItems: 10
        items:
          type: object
          required:
            - item_type
            - item_id
          properties:
            item_type:
              type: string
              enum: [room, activity, product]
            item_id:
              type: string
              format: uuid
            quantity:
              type: integer
              minimum: 1
              default: 1
            date:
              type: string
              format: date
              description: 利用日（アクティビティの場合）
            check_in:
              type: string
              format: date
              description: チェックイン日（ホテルの場合）
            check_out:
              type: string
              format: date
              description: チェックアウト日
            time_slot_id:
              type: string
              format: uuid
              description: タイムスロットID（アクティビティの場合）
            guests:
              type: array
              items:
                type: object
                properties:
                  name:
                    type: string
                  age_category:
                    type: string
                    enum: [adult, child, infant]
            options:
              type: array
              items:
                type: object
                properties:
                  option_id:
                    type: string
                    format: uuid
                  quantity:
                    type: integer
      contact:
        type: object
        required:
          - name
          - email
          - phone
        properties:
          name:
            type: string
          email:
            type: string
            format: email
          phone:
            type: string
          country:
            type: string
            maxLength: 2
      special_requests:
        type: string
        maxLength: 1000
        description: 特別なリクエスト
      payment_method_id:
        type: string
        format: uuid
        description: 支払い方法ID（既存の支払い方法を使用する場合）
      promo_code:
        type: string
        maxLength: 50
        description: プロモーションコード
      idempotency_key:
        type: string
        format: uuid
        description: 冪等性キー（重複予約防止）

responses:
  201:
    description: 予約作成成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
              example: true
            data:
              type: object
              properties:
                booking:
                  $ref: "#/components/schemas/Booking"
                payment_intent:
                  type: object
                  properties:
                    id:
                      type: string
                    client_secret:
                      type: string
                      description: フロントエンドで支払いを完了するためのシークレット
                    status:
                      type: string
                      enum: [requires_payment_method, requires_confirmation, requires_action, processing, succeeded]

  400:
    description: バリデーションエラー
    content:
      application/json:
        schema:
          $ref: "#/components/schemas/ErrorResponse"
        examples:
          invalid_dates:
            value:
              success: false
              error:
                code: "INVALID_DATES"
                message: "チェックアウト日はチェックイン日より後である必要があります"
          unavailable:
            value:
              success: false
              error:
                code: "ITEM_UNAVAILABLE"
                message: "選択された部屋は指定の日程で利用できません"
                details:
                  item_id: "550e8400-e29b-41d4-a716-446655440000"
                  available_dates: ["2026-02-05", "2026-02-06"]

  409:
    description: 競合エラー（在庫不足等）
    content:
      application/json:
        schema:
          $ref: "#/components/schemas/ErrorResponse"
        example:
          success: false
          error:
            code: "INVENTORY_CONFLICT"
            message: "この商品は他のお客様により予約されました"
```

### 5.3 GET /bookings - 予約一覧

```yaml
endpoint:
  method: GET
  path: /api/v1/bookings
  description: ユーザーの予約一覧を取得
  authentication: Bearer Token
  rate_limit: 30回/分/ユーザー

query_parameters:
  status:
    type: array
    items:
      type: string
      enum: [pending, confirmed, completed, cancelled, refunded]
    description: ステータスフィルター
  booking_type:
    type: array
    items:
      type: string
      enum: [hotel, activity, product, package]
  date_from:
    type: string
    format: date
    description: 利用日の開始範囲
  date_to:
    type: string
    format: date
    description: 利用日の終了範囲
  sort_by:
    type: string
    enum: [created_at, check_in_date, status]
    default: created_at
  sort_order:
    type: string
    enum: [asc, desc]
    default: desc
  page:
    type: integer
    minimum: 1
    default: 1
  limit:
    type: integer
    minimum: 1
    maximum: 50
    default: 20

responses:
  200:
    description: 予約一覧取得成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
            data:
              type: object
              properties:
                bookings:
                  type: array
                  items:
                    type: object
                    properties:
                      id:
                        type: string
                        format: uuid
                      booking_number:
                        type: string
                        example: "TT-2026012100001"
                      booking_type:
                        type: string
                      status:
                        type: string
                      created_at:
                        type: string
                        format: date-time
                      check_in_date:
                        type: string
                        format: date
                      check_out_date:
                        type: string
                        format: date
                      total_amount:
                        type: object
                        properties:
                          amount:
                            type: number
                          currency:
                            type: string
                      primary_item:
                        type: object
                        properties:
                          name:
                            type: string
                          image_url:
                            type: string
                pagination:
                  $ref: "#/components/schemas/Pagination"
```

### 5.4 GET /bookings/{id} - 予約詳細

```yaml
endpoint:
  method: GET
  path: /api/v1/bookings/{id}
  description: 予約の詳細情報を取得
  authentication: Bearer Token
  rate_limit: 60回/分/ユーザー

path_parameters:
  id:
    type: string
    format: uuid
    required: true

responses:
  200:
    description: 予約詳細取得成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
            data:
              type: object
              properties:
                id:
                  type: string
                  format: uuid
                booking_number:
                  type: string
                user_id:
                  type: string
                  format: uuid
                booking_type:
                  type: string
                status:
                  type: string
                  enum: [pending, confirmed, completed, cancelled, refunded]
                items:
                  type: array
                  items:
                    type: object
                    properties:
                      id:
                        type: string
                        format: uuid
                      item_type:
                        type: string
                      item_id:
                        type: string
                        format: uuid
                      name:
                        type: string
                      description:
                        type: string
                      quantity:
                        type: integer
                      unit_price:
                        type: number
                      total_price:
                        type: number
                      date:
                        type: string
                        format: date
                      check_in:
                        type: string
                        format: date
                      check_out:
                        type: string
                        format: date
                      guests:
                        type: array
                        items:
                          type: object
                          properties:
                            name:
                              type: string
                            age_category:
                              type: string
                      options:
                        type: array
                        items:
                          type: object
                          properties:
                            name:
                              type: string
                            quantity:
                              type: integer
                            price:
                              type: number
                      confirmation_code:
                        type: string
                        description: サプライヤー確認コード
                contact:
                  type: object
                  properties:
                    name:
                      type: string
                    email:
                      type: string
                    phone:
                      type: string
                pricing:
                  type: object
                  properties:
                    subtotal:
                      type: number
                    discount:
                      type: number
                    tax:
                      type: number
                    service_fee:
                      type: number
                    total:
                      type: number
                    currency:
                      type: string
                    promo_code:
                      type: string
                      nullable: true
                payment:
                  type: object
                  properties:
                    status:
                      type: string
                      enum: [pending, processing, completed, failed, refunded]
                    method:
                      type: string
                    last_four:
                      type: string
                    paid_at:
                      type: string
                      format: date-time
                cancellation_policy:
                  type: object
                  properties:
                    free_cancellation_until:
                      type: string
                      format: date-time
                    refund_percentage:
                      type: integer
                    policy_text:
                      type: string
                special_requests:
                  type: string
                status_history:
                  type: array
                  items:
                    type: object
                    properties:
                      status:
                        type: string
                      changed_at:
                        type: string
                        format: date-time
                      reason:
                        type: string
                created_at:
                  type: string
                  format: date-time
                updated_at:
                  type: string
                  format: date-time
                links:
                  type: object
                  properties:
                    confirmation_pdf:
                      type: string
                      format: uri
                    invoice_pdf:
                      type: string
                      format: uri
                    manage:
                      type: string
                      format: uri

  404:
    description: 予約が見つからない
```

### 5.5 PUT /bookings/{id} - 予約変更

```yaml
endpoint:
  method: PUT
  path: /api/v1/bookings/{id}
  description: 予約内容を変更
  authentication: Bearer Token
  rate_limit: 10回/分/ユーザー

path_parameters:
  id:
    type: string
    format: uuid
    required: true

request:
  content_type: application/json
  body:
    type: object
    properties:
      items:
        type: array
        description: 変更する項目
        items:
          type: object
          properties:
            item_id:
              type: string
              format: uuid
            check_in:
              type: string
              format: date
            check_out:
              type: string
              format: date
            date:
              type: string
              format: date
            time_slot_id:
              type: string
              format: uuid
            guests:
              type: array
              items:
                type: object
                properties:
                  name:
                    type: string
                  age_category:
                    type: string
      contact:
        type: object
        properties:
          name:
            type: string
          email:
            type: string
          phone:
            type: string
      special_requests:
        type: string

responses:
  200:
    description: 予約変更成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
            data:
              type: object
              properties:
                booking:
                  $ref: "#/components/schemas/Booking"
                price_difference:
                  type: object
                  properties:
                    original_total:
                      type: number
                    new_total:
                      type: number
                    difference:
                      type: number
                    requires_additional_payment:
                      type: boolean

  400:
    description: 変更不可

  409:
    description: 競合エラー
```

### 5.6 DELETE /bookings/{id} - 予約キャンセル

```yaml
endpoint:
  method: DELETE
  path: /api/v1/bookings/{id}
  description: 予約をキャンセル
  authentication: Bearer Token
  rate_limit: 10回/分/ユーザー

path_parameters:
  id:
    type: string
    format: uuid
    required: true

query_parameters:
  reason:
    type: string
    maxLength: 500
    description: キャンセル理由

responses:
  200:
    description: キャンセル成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
            data:
              type: object
              properties:
                booking_id:
                  type: string
                  format: uuid
                status:
                  type: string
                  example: "cancelled"
                refund:
                  type: object
                  properties:
                    eligible:
                      type: boolean
                    amount:
                      type: number
                    percentage:
                      type: integer
                    estimated_arrival:
                      type: string
                      format: date
                    refund_id:
                      type: string

  400:
    description: キャンセル不可
    content:
      application/json:
        schema:
          $ref: "#/components/schemas/ErrorResponse"
        example:
          success: false
          error:
            code: "CANCELLATION_NOT_ALLOWED"
            message: "この予約はキャンセルできません"
            details:
              reason: "チェックイン日を過ぎています"
```

---

## 第6章：旅程API

### 6.1 旅程エンドポイント概要

```yaml
itinerary_endpoints:
  base_path: /api/v1/itineraries
  endpoints:
    - method: POST
      path: /
      description: 旅程作成
      auth: required
    - method: GET
      path: /
      description: 旅程一覧取得
      auth: required
    - method: GET
      path: /{id}
      description: 旅程詳細取得
      auth: required
    - method: PUT
      path: /{id}
      description: 旅程更新
      auth: required
    - method: DELETE
      path: /{id}
      description: 旅程削除
      auth: required
    - method: POST
      path: /{id}/items
      description: アイテム追加
      auth: required
    - method: PUT
      path: /{id}/items/{item_id}
      description: アイテム更新
      auth: required
    - method: DELETE
      path: /{id}/items/{item_id}
      description: アイテム削除
      auth: required
    - method: POST
      path: /{id}/share
      description: 旅程共有
      auth: required
    - method: POST
      path: /{id}/collaborators
      description: コラボレーター追加
      auth: required
```

### 6.2 POST /itineraries - 旅程作成

```yaml
endpoint:
  method: POST
  path: /api/v1/itineraries
  description: 新しい旅程を作成
  authentication: Bearer Token
  rate_limit: 20回/分/ユーザー

request:
  content_type: application/json
  body:
    type: object
    required:
      - title
      - start_date
      - end_date
    properties:
      title:
        type: string
        minLength: 1
        maxLength: 200
        example: "京都・大阪 3日間の旅"
      description:
        type: string
        maxLength: 2000
      start_date:
        type: string
        format: date
        example: "2026-03-15"
      end_date:
        type: string
        format: date
        example: "2026-03-17"
      destinations:
        type: array
        items:
          type: string
          format: uuid
        description: 目的地IDリスト
      cover_image_url:
        type: string
        format: uri
      visibility:
        type: string
        enum: [private, shared, public]
        default: private
      template_id:
        type: string
        format: uuid
        description: テンプレートから作成する場合

responses:
  201:
    description: 旅程作成成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
            data:
              $ref: "#/components/schemas/Itinerary"
```

### 6.3 GET /itineraries - 旅程一覧

```yaml
endpoint:
  method: GET
  path: /api/v1/itineraries
  description: ユーザーの旅程一覧を取得
  authentication: Bearer Token
  rate_limit: 60回/分/ユーザー

query_parameters:
  status:
    type: string
    enum: [draft, planned, active, completed, archived]
  visibility:
    type: string
    enum: [private, shared, public]
  date_from:
    type: string
    format: date
  date_to:
    type: string
    format: date
  include_shared:
    type: boolean
    default: true
    description: 共有された旅程を含める
  sort_by:
    type: string
    enum: [created_at, start_date, title]
    default: start_date
  sort_order:
    type: string
    enum: [asc, desc]
    default: desc
  page:
    type: integer
    default: 1
  limit:
    type: integer
    default: 20

responses:
  200:
    description: 旅程一覧取得成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
            data:
              type: object
              properties:
                itineraries:
                  type: array
                  items:
                    type: object
                    properties:
                      id:
                        type: string
                        format: uuid
                      title:
                        type: string
                      start_date:
                        type: string
                        format: date
                      end_date:
                        type: string
                        format: date
                      duration_days:
                        type: integer
                      cover_image_url:
                        type: string
                        format: uri
                      status:
                        type: string
                      visibility:
                        type: string
                      destinations:
                        type: array
                        items:
                          type: object
                          properties:
                            id:
                              type: string
                            name:
                              type: string
                      items_count:
                        type: integer
                      collaborators_count:
                        type: integer
                      is_owner:
                        type: boolean
                      created_at:
                        type: string
                        format: date-time
                pagination:
                  $ref: "#/components/schemas/Pagination"
```

### 6.4 GET /itineraries/{id} - 旅程詳細

```yaml
endpoint:
  method: GET
  path: /api/v1/itineraries/{id}
  description: 旅程の詳細を取得
  authentication: Bearer Token（共有リンク経由は不要）
  rate_limit: 60回/分/ユーザー

path_parameters:
  id:
    type: string
    format: uuid
    required: true

query_parameters:
  share_token:
    type: string
    description: 共有リンクのトークン

responses:
  200:
    description: 旅程詳細取得成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
            data:
              type: object
              properties:
                id:
                  type: string
                  format: uuid
                title:
                  type: string
                description:
                  type: string
                start_date:
                  type: string
                  format: date
                end_date:
                  type: string
                  format: date
                duration_days:
                  type: integer
                cover_image_url:
                  type: string
                status:
                  type: string
                visibility:
                  type: string
                owner:
                  type: object
                  properties:
                    id:
                      type: string
                    name:
                      type: string
                    avatar_url:
                      type: string
                collaborators:
                  type: array
                  items:
                    type: object
                    properties:
                      id:
                        type: string
                      name:
                        type: string
                      avatar_url:
                        type: string
                      role:
                        type: string
                        enum: [viewer, editor]
                destinations:
                  type: array
                  items:
                    type: object
                    properties:
                      id:
                        type: string
                      name:
                        type: string
                      country:
                        type: string
                      image_url:
                        type: string
                days:
                  type: array
                  items:
                    type: object
                    properties:
                      date:
                        type: string
                        format: date
                      day_number:
                        type: integer
                      items:
                        type: array
                        items:
                          type: object
                          properties:
                            id:
                              type: string
                              format: uuid
                            type:
                              type: string
                              enum: [hotel, activity, restaurant, transport, note, booking]
                            order:
                              type: integer
                            start_time:
                              type: string
                              format: time
                            end_time:
                              type: string
                              format: time
                            title:
                              type: string
                            description:
                              type: string
                            location:
                              type: object
                              properties:
                                name:
                                  type: string
                                address:
                                  type: string
                                coordinates:
                                  type: object
                            reference_id:
                              type: string
                              format: uuid
                              description: 関連する予約やエンティティのID
                            reference_type:
                              type: string
                              enum: [booking, hotel, activity, restaurant]
                            booking_status:
                              type: string
                            price:
                              type: object
                              properties:
                                amount:
                                  type: number
                                currency:
                                  type: string
                            notes:
                              type: string
                            attachments:
                              type: array
                              items:
                                type: object
                                properties:
                                  type:
                                    type: string
                                  url:
                                    type: string
                                  name:
                                    type: string
                summary:
                  type: object
                  properties:
                    total_estimated_cost:
                      type: object
                      properties:
                        amount:
                          type: number
                        currency:
                          type: string
                    confirmed_bookings_count:
                      type: integer
                    pending_bookings_count:
                      type: integer
                created_at:
                  type: string
                  format: date-time
                updated_at:
                  type: string
                  format: date-time
                permissions:
                  type: object
                  properties:
                    can_edit:
                      type: boolean
                    can_delete:
                      type: boolean
                    can_share:
                      type: boolean
                    can_invite:
                      type: boolean
```

### 6.5 POST /itineraries/{id}/share - 旅程共有

```yaml
endpoint:
  method: POST
  path: /api/v1/itineraries/{id}/share
  description: 旅程の共有リンクを生成
  authentication: Bearer Token
  rate_limit: 20回/分/ユーザー

path_parameters:
  id:
    type: string
    format: uuid
    required: true

request:
  content_type: application/json
  body:
    type: object
    properties:
      permission:
        type: string
        enum: [view, edit]
        default: view
        description: 共有リンクの権限
      expires_in_days:
        type: integer
        minimum: 1
        maximum: 365
        default: 30
        description: 有効期限（日数）
      password:
        type: string
        minLength: 4
        maxLength: 20
        description: パスワード保護（オプション）

responses:
  200:
    description: 共有リンク生成成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
            data:
              type: object
              properties:
                share_url:
                  type: string
                  format: uri
                  example: "https://triptrip.com/itineraries/share/abc123def456"
                share_token:
                  type: string
                permission:
                  type: string
                expires_at:
                  type: string
                  format: date-time
                has_password:
                  type: boolean
```

---

## 第7章：決済API

### 7.1 決済エンドポイント概要

```yaml
payment_endpoints:
  base_path: /api/v1/payments
  endpoints:
    - method: POST
      path: /
      description: 決済実行
      auth: required
    - method: GET
      path: /{id}
      description: 決済状況取得
      auth: required
    - method: POST
      path: /{id}/refund
      description: 返金処理
      auth: required
    - method: GET
      path: /methods
      description: 支払い方法一覧
      auth: required
    - method: POST
      path: /methods
      description: 支払い方法追加
      auth: required
    - method: DELETE
      path: /methods/{id}
      description: 支払い方法削除
      auth: required
```

### 7.2 POST /payments - 決済実行

```yaml
endpoint:
  method: POST
  path: /api/v1/payments
  description: 決済を実行
  authentication: Bearer Token
  rate_limit: 10回/分/ユーザー

request:
  content_type: application/json
  body:
    type: object
    required:
      - booking_id
      - payment_method
    properties:
      booking_id:
        type: string
        format: uuid
        description: 予約ID
      payment_method:
        oneOf:
          - type: object
            required:
              - type
              - saved_method_id
            properties:
              type:
                type: string
                const: saved
              saved_method_id:
                type: string
                format: uuid
          - type: object
            required:
              - type
              - card
            properties:
              type:
                type: string
                const: card
              card:
                type: object
                required:
                  - token
                properties:
                  token:
                    type: string
                    description: カードトークン（Stripe等）
                  save_for_future:
                    type: boolean
                    default: false
          - type: object
            required:
              - type
            properties:
              type:
                type: string
                enum: [apple_pay, google_pay]
              payment_data:
                type: string
                description: 決済プラットフォームからのペイメントデータ
      idempotency_key:
        type: string
        format: uuid
        description: 冪等性キー
      metadata:
        type: object
        description: カスタムメタデータ

responses:
  200:
    description: 決済成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
            data:
              type: object
              properties:
                payment_id:
                  type: string
                  format: uuid
                status:
                  type: string
                  enum: [succeeded, processing, requires_action]
                amount:
                  type: number
                currency:
                  type: string
                receipt_url:
                  type: string
                  format: uri
                booking:
                  type: object
                  properties:
                    id:
                      type: string
                    status:
                      type: string
                next_action:
                  type: object
                  nullable: true
                  properties:
                    type:
                      type: string
                      enum: [redirect_to_url, use_stripe_sdk]
                    redirect_url:
                      type: string
                      format: uri
                    stripe_js_action:
                      type: object

  400:
    description: 決済エラー
    content:
      application/json:
        schema:
          $ref: "#/components/schemas/ErrorResponse"
        examples:
          card_declined:
            value:
              success: false
              error:
                code: "CARD_DECLINED"
                message: "カードが拒否されました"
                details:
                  decline_code: "insufficient_funds"
          invalid_card:
            value:
              success: false
              error:
                code: "INVALID_CARD"
                message: "無効なカード情報です"
```

### 7.3 GET /payments/{id} - 決済状況

```yaml
endpoint:
  method: GET
  path: /api/v1/payments/{id}
  description: 決済の状況を取得
  authentication: Bearer Token
  rate_limit: 60回/分/ユーザー

path_parameters:
  id:
    type: string
    format: uuid
    required: true

responses:
  200:
    description: 決済状況取得成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
            data:
              type: object
              properties:
                id:
                  type: string
                  format: uuid
                booking_id:
                  type: string
                  format: uuid
                status:
                  type: string
                  enum: [pending, processing, succeeded, failed, refunded, partially_refunded]
                amount:
                  type: number
                currency:
                  type: string
                payment_method:
                  type: object
                  properties:
                    type:
                      type: string
                    brand:
                      type: string
                    last_four:
                      type: string
                refunds:
                  type: array
                  items:
                    type: object
                    properties:
                      id:
                        type: string
                      amount:
                        type: number
                      status:
                        type: string
                      created_at:
                        type: string
                        format: date-time
                receipt_url:
                  type: string
                  format: uri
                created_at:
                  type: string
                  format: date-time
                updated_at:
                  type: string
                  format: date-time
```

### 7.4 POST /payments/{id}/refund - 返金

```yaml
endpoint:
  method: POST
  path: /api/v1/payments/{id}/refund
  description: 決済の返金を処理
  authentication: Bearer Token
  rate_limit: 5回/分/ユーザー

path_parameters:
  id:
    type: string
    format: uuid
    required: true

request:
  content_type: application/json
  body:
    type: object
    properties:
      amount:
        type: number
        description: 返金額（指定しない場合は全額返金）
      reason:
        type: string
        enum: [requested_by_customer, duplicate, fraudulent, other]
        default: requested_by_customer
      notes:
        type: string
        maxLength: 500

responses:
  200:
    description: 返金処理成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
            data:
              type: object
              properties:
                refund_id:
                  type: string
                  format: uuid
                payment_id:
                  type: string
                  format: uuid
                amount:
                  type: number
                currency:
                  type: string
                status:
                  type: string
                  enum: [pending, succeeded, failed]
                estimated_arrival:
                  type: string
                  format: date
                  description: 返金予定日

  400:
    description: 返金エラー
    content:
      application/json:
        schema:
          $ref: "#/components/schemas/ErrorResponse"
        example:
          success: false
          error:
            code: "REFUND_EXCEEDED"
            message: "返金額が決済額を超えています"
```

---

## 第8章：推奨API

### 8.1 推奨エンドポイント概要

```yaml
recommendation_endpoints:
  base_path: /api/v1/recommendations
  endpoints:
    - method: GET
      path: /
      description: パーソナライズ推奨
      auth: optional
    - method: GET
      path: /trending
      description: トレンド
      auth: optional
    - method: GET
      path: /similar/{type}/{id}
      description: 類似アイテム
      auth: optional
    - method: POST
      path: /feedback
      description: フィードバック
      auth: required
```

### 8.2 GET /recommendations - パーソナライズ推奨

```yaml
endpoint:
  method: GET
  path: /api/v1/recommendations
  description: ユーザーにパーソナライズされた推奨を取得
  authentication: Optional（認証時はパーソナライズ）
  rate_limit: 60回/分/IP
  caching: 5分

query_parameters:
  types:
    type: array
    items:
      type: string
      enum: [hotel, activity, product, destination]
    default: [hotel, activity, destination]
  destination_id:
    type: string
    format: uuid
    description: 目的地コンテキスト
  date:
    type: string
    format: date
    description: 旅行日コンテキスト
  limit:
    type: integer
    minimum: 1
    maximum: 50
    default: 10
  context:
    type: string
    enum: [home, search, booking, itinerary]
    default: home
    description: 推奨コンテキスト

responses:
  200:
    description: 推奨取得成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
            data:
              type: object
              properties:
                recommendations:
                  type: array
                  items:
                    type: object
                    properties:
                      type:
                        type: string
                        enum: [hotel, activity, product, destination]
                      id:
                        type: string
                        format: uuid
                      title:
                        type: string
                      description:
                        type: string
                      image_url:
                        type: string
                        format: uri
                      rating:
                        type: number
                      review_count:
                        type: integer
                      price:
                        type: object
                        properties:
                          amount:
                            type: number
                          currency:
                            type: string
                      location:
                        type: object
                        properties:
                          name:
                            type: string
                      reason:
                        type: string
                        description: 推奨理由
                        example: "あなたの閲覧履歴に基づく"
                      score:
                        type: number
                        description: 推奨スコア（内部用）
                personalized:
                  type: boolean
                  description: パーソナライズされた結果かどうか
```

### 8.3 GET /recommendations/trending - トレンド

```yaml
endpoint:
  method: GET
  path: /api/v1/recommendations/trending
  description: トレンドアイテムを取得
  authentication: Optional
  rate_limit: 60回/分/IP
  caching: 15分

query_parameters:
  type:
    type: string
    enum: [hotel, activity, product, destination]
    required: true
  region:
    type: string
    description: 地域フィルター
    example: "kanto"
  period:
    type: string
    enum: [day, week, month]
    default: week
  limit:
    type: integer
    minimum: 1
    maximum: 50
    default: 10

responses:
  200:
    description: トレンド取得成功
    content:
      application/json:
        schema:
          type: object
          properties:
            success:
              type: boolean
            data:
              type: object
              properties:
                trending:
                  type: array
                  items:
                    type: object
                    properties:
                      rank:
                        type: integer
                      trend:
                        type: string
                        enum: [up, down, stable, new]
                      rank_change:
                        type: integer
                        description: 前期間からの順位変動
                      item:
                        type: object
                        properties:
                          type:
                            type: string
                          id:
                            type: string
                          title:
                            type: string
                          image_url:
                            type: string
                          rating:
                            type: number
                          price:
                            type: object
                period:
                  type: string
                updated_at:
                  type: string
                  format: date-time
```

---

## 第9章：共通仕様

### 9.1 ページネーション

```yaml
pagination:
  standard_format:
    query_parameters:
      page:
        type: integer
        minimum: 1
        default: 1
        description: ページ番号
      limit:
        type: integer
        minimum: 1
        maximum: 100
        default: 20
        description: 1ページあたりの件数
      offset:
        type: integer
        minimum: 0
        description: オフセット（pageと併用不可）

    response_format:
      type: object
      properties:
        pagination:
          type: object
          properties:
            total:
              type: integer
              description: 全件数
            page:
              type: integer
              description: 現在のページ
            limit:
              type: integer
              description: 1ページあたりの件数
            total_pages:
              type: integer
              description: 総ページ数
            has_next:
              type: boolean
              description: 次のページがあるか
            has_prev:
              type: boolean
              description: 前のページがあるか
          example:
            total: 150
            page: 2
            limit: 20
            total_pages: 8
            has_next: true
            has_prev: true

  cursor_based:
    description: 大量データ用のカーソルベースページネーション
    query_parameters:
      cursor:
        type: string
        description: 次のページのカーソル
      limit:
        type: integer
        default: 20

    response_format:
      type: object
      properties:
        pagination:
          type: object
          properties:
            next_cursor:
              type: string
              nullable: true
            has_more:
              type: boolean
```

### 9.2 フィルタリング・ソート

```yaml
filtering_and_sorting:
  filtering:
    description: クエリパラメータによるフィルタリング
    patterns:
      equality:
        example: "?status=active"
        description: 完全一致
      multiple_values:
        example: "?status=active,pending"
        description: OR条件
      range:
        example: "?price_min=1000&price_max=5000"
        description: 範囲指定
      array:
        example: "?amenities[]=wifi&amenities[]=pool"
        description: 配列フィルター
      date_range:
        example: "?date_from=2026-01-01&date_to=2026-12-31"
        description: 日付範囲

  sorting:
    query_parameters:
      sort_by:
        type: string
        description: ソートフィールド
      sort_order:
        type: string
        enum: [asc, desc]
        default: desc

    multi_sort:
      example: "?sort=price:asc,rating:desc"
      description: 複数フィールドでのソート
```

### 9.3 エラーレスポンス形式

```yaml
error_response:
  standard_format:
    type: object
    required:
      - success
      - error
    properties:
      success:
        type: boolean
        const: false
      error:
        type: object
        required:
          - code
          - message
        properties:
          code:
            type: string
            description: エラーコード（大文字スネークケース）
            example: "VALIDATION_ERROR"
          message:
            type: string
            description: ユーザー向けエラーメッセージ
            example: "入力内容に誤りがあります"
          details:
            oneOf:
              - type: array
                description: バリデーションエラーの詳細
                items:
                  type: object
                  properties:
                    field:
                      type: string
                    message:
                      type: string
                    code:
                      type: string
              - type: object
                description: その他のエラー詳細
          request_id:
            type: string
            description: リクエストID（サポート問い合わせ用）

  error_codes:
    authentication:
      - code: UNAUTHORIZED
        status: 401
        message: 認証が必要です
      - code: INVALID_TOKEN
        status: 401
        message: トークンが無効です
      - code: TOKEN_EXPIRED
        status: 401
        message: トークンの有効期限が切れています
      - code: INVALID_CREDENTIALS
        status: 401
        message: 認証情報が正しくありません

    authorization:
      - code: FORBIDDEN
        status: 403
        message: アクセス権限がありません
      - code: INSUFFICIENT_PERMISSIONS
        status: 403
        message: 必要な権限がありません

    validation:
      - code: VALIDATION_ERROR
        status: 400
        message: 入力内容に誤りがあります
      - code: INVALID_FORMAT
        status: 400
        message: 形式が正しくありません
      - code: REQUIRED_FIELD_MISSING
        status: 400
        message: 必須項目が入力されていません

    resource:
      - code: NOT_FOUND
        status: 404
        message: リソースが見つかりません
      - code: ALREADY_EXISTS
        status: 409
        message: すでに存在します
      - code: CONFLICT
        status: 409
        message: 競合が発生しました

    rate_limiting:
      - code: RATE_LIMIT_EXCEEDED
        status: 429
        message: リクエスト数の上限に達しました

    server:
      - code: INTERNAL_ERROR
        status: 500
        message: サーバーエラーが発生しました
      - code: SERVICE_UNAVAILABLE
        status: 503
        message: サービスが一時的に利用できません
```

### 9.4 レート制限ヘッダー

```yaml
rate_limiting:
  headers:
    response_headers:
      X-RateLimit-Limit:
        description: 時間枠内の最大リクエスト数
        example: "100"
      X-RateLimit-Remaining:
        description: 残りリクエスト数
        example: "95"
      X-RateLimit-Reset:
        description: リセット時刻（Unix timestamp）
        example: "1705827600"
      Retry-After:
        description: 再試行までの秒数（429レスポンス時）
        example: "60"

  limits_by_endpoint:
    authentication:
      register: 5/分/IP
      login: 10/分/IP
      refresh: 30/時間/ユーザー

    read_operations:
      profile: 100/分/ユーザー
      search: 60/分/IP
      list: 60/分/ユーザー

    write_operations:
      create: 20/分/ユーザー
      update: 20/分/ユーザー
      delete: 10/分/ユーザー

    payment:
      payment: 10/分/ユーザー
      refund: 5/分/ユーザー

  implementation:
    algorithm: Sliding Window
    storage: Redis
    key_format: "ratelimit:{identifier}:{endpoint}:{window}"
```

---

## 第10章：OpenAPI 3.0仕様書（抜粋）

### 10.1 基本構造

```yaml
openapi: 3.0.3
info:
  title: TripTrip API
  description: |
    TripTripプラットフォームのRESTful API仕様書。

    ## 認証
    多くのエンドポイントはBearer Token認証を必要とします。

    ## レート制限
    すべてのエンドポイントにレート制限が適用されます。

    ## エラーハンドリング
    すべてのエラーは標準化された形式で返されます。
  version: 1.0.0
  contact:
    name: TripTrip API Support
    email: api-support@triptrip.com
  license:
    name: Proprietary

servers:
  - url: https://api.triptrip.com/api/v1
    description: 本番環境
  - url: https://staging-api.triptrip.com/api/v1
    description: ステージング環境
  - url: http://localhost:3001/api/v1
    description: 開発環境

tags:
  - name: Authentication
    description: 認証・認可関連
  - name: Users
    description: ユーザー管理
  - name: Search
    description: 検索機能
  - name: Bookings
    description: 予約管理
  - name: Itineraries
    description: 旅程管理
  - name: Payments
    description: 決済処理
  - name: Recommendations
    description: 推奨・パーソナライゼーション

security:
  - BearerAuth: []

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: JWTアクセストークン

  schemas:
    User:
      type: object
      properties:
        id:
          type: string
          format: uuid
        email:
          type: string
          format: email
        name:
          type: string
        avatar_url:
          type: string
          format: uri
          nullable: true
        language:
          type: string
          enum: [ja, en]
        currency:
          type: string
          enum: [JPY, USD, EUR]
        role:
          type: string
          enum: [user, premium, partner, admin]
        email_verified:
          type: boolean
        created_at:
          type: string
          format: date-time
        updated_at:
          type: string
          format: date-time

    Booking:
      type: object
      properties:
        id:
          type: string
          format: uuid
        booking_number:
          type: string
        user_id:
          type: string
          format: uuid
        booking_type:
          type: string
          enum: [hotel, activity, product, package]
        status:
          type: string
          enum: [pending, confirmed, completed, cancelled, refunded]
        items:
          type: array
          items:
            $ref: '#/components/schemas/BookingItem'
        contact:
          $ref: '#/components/schemas/Contact'
        pricing:
          $ref: '#/components/schemas/Pricing'
        created_at:
          type: string
          format: date-time
        updated_at:
          type: string
          format: date-time

    BookingItem:
      type: object
      properties:
        id:
          type: string
          format: uuid
        item_type:
          type: string
          enum: [room, activity, product]
        item_id:
          type: string
          format: uuid
        name:
          type: string
        quantity:
          type: integer
        unit_price:
          type: number
        total_price:
          type: number
        date:
          type: string
          format: date
        check_in:
          type: string
          format: date
        check_out:
          type: string
          format: date

    Itinerary:
      type: object
      properties:
        id:
          type: string
          format: uuid
        title:
          type: string
        description:
          type: string
        start_date:
          type: string
          format: date
        end_date:
          type: string
          format: date
        status:
          type: string
          enum: [draft, planned, active, completed, archived]
        visibility:
          type: string
          enum: [private, shared, public]
        owner:
          $ref: '#/components/schemas/UserSummary'
        created_at:
          type: string
          format: date-time
        updated_at:
          type: string
          format: date-time

    Contact:
      type: object
      required:
        - name
        - email
        - phone
      properties:
        name:
          type: string
        email:
          type: string
          format: email
        phone:
          type: string
        country:
          type: string

    Pricing:
      type: object
      properties:
        subtotal:
          type: number
        discount:
          type: number
        tax:
          type: number
        service_fee:
          type: number
        total:
          type: number
        currency:
          type: string

    UserSummary:
      type: object
      properties:
        id:
          type: string
          format: uuid
        name:
          type: string
        avatar_url:
          type: string
          format: uri

    Pagination:
      type: object
      properties:
        total:
          type: integer
        page:
          type: integer
        limit:
          type: integer
        total_pages:
          type: integer
        has_next:
          type: boolean
        has_prev:
          type: boolean

    ErrorResponse:
      type: object
      required:
        - success
        - error
      properties:
        success:
          type: boolean
          const: false
        error:
          type: object
          required:
            - code
            - message
          properties:
            code:
              type: string
            message:
              type: string
            details:
              oneOf:
                - type: array
                  items:
                    type: object
                    properties:
                      field:
                        type: string
                      message:
                        type: string
                - type: object
            request_id:
              type: string

  responses:
    BadRequest:
      description: リクエスト形式エラー
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'

    Unauthorized:
      description: 認証エラー
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'

    Forbidden:
      description: 認可エラー
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'

    NotFound:
      description: リソース不在
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'

    TooManyRequests:
      description: レート制限超過
      headers:
        Retry-After:
          schema:
            type: integer
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'

    InternalError:
      description: サーバー内部エラー
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
```

### 10.2 エンドポイント定義（抜粋）

```yaml
paths:
  /auth/register:
    post:
      tags:
        - Authentication
      summary: ユーザー登録
      description: 新規ユーザーアカウントを作成
      operationId: registerUser
      security: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - email
                - password
                - name
              properties:
                email:
                  type: string
                  format: email
                password:
                  type: string
                  minLength: 8
                name:
                  type: string
                language:
                  type: string
                  enum: [ja, en]
                  default: ja
                currency:
                  type: string
                  enum: [JPY, USD, EUR]
                  default: JPY
      responses:
        '201':
          description: 登録成功
          content:
            application/json:
              schema:
                type: object
                properties:
                  success:
                    type: boolean
                  data:
                    type: object
                    properties:
                      user:
                        $ref: '#/components/schemas/User'
                      access_token:
                        type: string
                      refresh_token:
                        type: string
                      expires_in:
                        type: integer
        '400':
          $ref: '#/components/responses/BadRequest'
        '409':
          description: メールアドレス重複
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '429':
          $ref: '#/components/responses/TooManyRequests'

  /auth/login:
    post:
      tags:
        - Authentication
      summary: ログイン
      description: メールアドレスとパスワードでログイン
      operationId: loginUser
      security: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - email
                - password
              properties:
                email:
                  type: string
                  format: email
                password:
                  type: string
                remember_me:
                  type: boolean
                  default: false
      responses:
        '200':
          description: ログイン成功
          content:
            application/json:
              schema:
                type: object
                properties:
                  success:
                    type: boolean
                  data:
                    type: object
                    properties:
                      user:
                        $ref: '#/components/schemas/User'
                      access_token:
                        type: string
                      refresh_token:
                        type: string
                      expires_in:
                        type: integer
                      token_type:
                        type: string
        '401':
          $ref: '#/components/responses/Unauthorized'
        '429':
          $ref: '#/components/responses/TooManyRequests'

  /users/me:
    get:
      tags:
        - Users
      summary: プロフィール取得
      description: 認証ユーザーのプロフィール情報を取得
      operationId: getCurrentUser
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                type: object
                properties:
                  success:
                    type: boolean
                  data:
                    $ref: '#/components/schemas/User'
        '401':
          $ref: '#/components/responses/Unauthorized'

    put:
      tags:
        - Users
      summary: プロフィール更新
      description: プロフィール情報を更新
      operationId: updateCurrentUser
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                name:
                  type: string
                phone:
                  type: string
                language:
                  type: string
                  enum: [ja, en]
                currency:
                  type: string
                  enum: [JPY, USD, EUR]
      responses:
        '200':
          description: 更新成功
          content:
            application/json:
              schema:
                type: object
                properties:
                  success:
                    type: boolean
                  data:
                    $ref: '#/components/schemas/User'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'

  /search/hotels:
    get:
      tags:
        - Search
      summary: ホテル検索
      description: 条件に基づいてホテルを検索
      operationId: searchHotels
      security:
        - {}
        - BearerAuth: []
      parameters:
        - name: destination_id
          in: query
          schema:
            type: string
            format: uuid
        - name: check_in
          in: query
          required: true
          schema:
            type: string
            format: date
        - name: check_out
          in: query
          required: true
          schema:
            type: string
            format: date
        - name: guests
          in: query
          schema:
            type: integer
            default: 2
        - name: rooms
          in: query
          schema:
            type: integer
            default: 1
        - name: price_min
          in: query
          schema:
            type: integer
        - name: price_max
          in: query
          schema:
            type: integer
        - name: star_rating
          in: query
          schema:
            type: array
            items:
              type: integer
        - name: sort_by
          in: query
          schema:
            type: string
            enum: [relevance, price_low, price_high, rating, distance, popularity]
            default: relevance
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
      responses:
        '200':
          description: 検索成功
          content:
            application/json:
              schema:
                type: object
                properties:
                  success:
                    type: boolean
                  data:
                    type: object
                    properties:
                      hotels:
                        type: array
                        items:
                          type: object
                      pagination:
                        $ref: '#/components/schemas/Pagination'
                      facets:
                        type: object
        '400':
          $ref: '#/components/responses/BadRequest'
        '429':
          $ref: '#/components/responses/TooManyRequests'

  /bookings:
    post:
      tags:
        - Bookings
      summary: 予約作成
      description: 新規予約を作成
      operationId: createBooking
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - booking_type
                - items
                - contact
              properties:
                booking_type:
                  type: string
                  enum: [hotel, activity, product, package]
                items:
                  type: array
                  items:
                    type: object
                contact:
                  $ref: '#/components/schemas/Contact'
                special_requests:
                  type: string
                promo_code:
                  type: string
                idempotency_key:
                  type: string
                  format: uuid
      responses:
        '201':
          description: 予約作成成功
          content:
            application/json:
              schema:
                type: object
                properties:
                  success:
                    type: boolean
                  data:
                    type: object
                    properties:
                      booking:
                        $ref: '#/components/schemas/Booking'
                      payment_intent:
                        type: object
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '409':
          description: 競合エラー
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

    get:
      tags:
        - Bookings
      summary: 予約一覧
      description: ユーザーの予約一覧を取得
      operationId: listBookings
      parameters:
        - name: status
          in: query
          schema:
            type: array
            items:
              type: string
              enum: [pending, confirmed, completed, cancelled, refunded]
        - name: booking_type
          in: query
          schema:
            type: array
            items:
              type: string
              enum: [hotel, activity, product, package]
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                type: object
                properties:
                  success:
                    type: boolean
                  data:
                    type: object
                    properties:
                      bookings:
                        type: array
                        items:
                          $ref: '#/components/schemas/Booking'
                      pagination:
                        $ref: '#/components/schemas/Pagination'
        '401':
          $ref: '#/components/responses/Unauthorized'
```

---

## 第11章：文書間参照 & 統合ポイント

### 11.1 関連文書マトリックス

```yaml
document_references:
  inputs:
    - document: EXISTING_APP_ANALYSIS.md
      title: 既存アプリケーション分析
      relevance: 現行API構造、実装済み機能

    - document: Doc-SA-003
      title: API Architecture & Design
      relevance: APIアーキテクチャ、設計原則

    - document: Doc-DA-001
      title: Data Architecture Overview
      relevance: データモデル、スキーマ設計

    - document: Doc-DA-002
      title: Database Strategy
      relevance: データベース構成、キャッシング戦略

  outputs:
    - document: Doc-SP-002
      title: Database Schema & ERD
      dependency: API仕様からのデータ要件

    - document: Doc-AD-001
      title: Application Design
      dependency: API統合仕様

    - document: Doc-SC-001
      title: Security Architecture
      dependency: 認証・認可仕様
```

### 11.2 実装ガイドライン

```yaml
implementation_guidelines:
  code_generation:
    description: OpenAPI仕様からの型生成
    tools:
      - name: openapi-typescript
        purpose: TypeScript型定義生成
        command: "npx openapi-typescript openapi.yaml -o types.ts"
      - name: @hono/zod-openapi
        purpose: Zodスキーマ + OpenAPI生成
        integration: Honoルート定義と統合

  validation:
    approach: Zod-first
    flow:
      1: Zodスキーマを定義
      2: OpenAPIスキーマを自動生成
      3: ランタイムバリデーション実行
      4: 型安全なレスポンス返却

  testing:
    unit_tests:
      - バリデーションロジック
      - エラーハンドリング
    integration_tests:
      - エンドポイント動作
      - 認証フロー
    contract_tests:
      - OpenAPI仕様との整合性
```

### 11.3 セキュリティ考慮事項

```yaml
security_considerations:
  authentication:
    - JWT署名アルゴリズム: RS256
    - トークンローテーション: 有効
    - リフレッシュトークン保護: HttpOnly Cookie

  authorization:
    - RBAC実装
    - リソースレベルアクセス制御
    - スコープベース権限

  input_validation:
    - Zodによる厳密なバリデーション
    - SQLインジェクション防止
    - XSS防止

  rate_limiting:
    - IPベース制限
    - ユーザーベース制限
    - エンドポイント別制限

  logging:
    - リクエストID追跡
    - 認証イベントログ
    - エラーログ
```

---

## 付録

### A. HTTPステータスコード一覧

| コード | 名前 | 使用場面 |
|--------|------|----------|
| 200 | OK | GET/PUT/PATCH成功 |
| 201 | Created | POST成功（リソース作成） |
| 204 | No Content | DELETE成功 |
| 400 | Bad Request | リクエスト形式エラー |
| 401 | Unauthorized | 認証エラー |
| 403 | Forbidden | 認可エラー |
| 404 | Not Found | リソース不在 |
| 409 | Conflict | 競合エラー |
| 422 | Unprocessable Entity | ビジネスロジックエラー |
| 429 | Too Many Requests | レート制限超過 |
| 500 | Internal Server Error | サーバーエラー |
| 502 | Bad Gateway | 外部サービスエラー |
| 503 | Service Unavailable | メンテナンス中 |

### B. エラーコード一覧

| カテゴリ | コード | 説明 |
|----------|--------|------|
| 認証 | UNAUTHORIZED | 認証が必要 |
| 認証 | INVALID_TOKEN | トークン無効 |
| 認証 | TOKEN_EXPIRED | トークン期限切れ |
| 認証 | INVALID_CREDENTIALS | 認証情報不正 |
| 認証 | ACCOUNT_LOCKED | アカウントロック |
| 認可 | FORBIDDEN | アクセス権限なし |
| 認可 | INSUFFICIENT_PERMISSIONS | 権限不足 |
| 検証 | VALIDATION_ERROR | バリデーションエラー |
| 検証 | INVALID_FORMAT | 形式不正 |
| 検証 | REQUIRED_FIELD_MISSING | 必須項目不足 |
| リソース | NOT_FOUND | リソース不在 |
| リソース | ALREADY_EXISTS | 既存リソース |
| リソース | CONFLICT | 競合 |
| 予約 | ITEM_UNAVAILABLE | 在庫なし |
| 予約 | INVENTORY_CONFLICT | 在庫競合 |
| 予約 | CANCELLATION_NOT_ALLOWED | キャンセル不可 |
| 決済 | CARD_DECLINED | カード拒否 |
| 決済 | INVALID_CARD | カード無効 |
| 決済 | REFUND_EXCEEDED | 返金超過 |
| レート | RATE_LIMIT_EXCEEDED | レート制限超過 |
| サーバー | INTERNAL_ERROR | 内部エラー |
| サーバー | SERVICE_UNAVAILABLE | サービス停止 |

### C. 変更履歴

| バージョン | 日付 | 変更内容 | 作成者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-01-21 | 初版作成 | Technical Architecture Agent |

---

**文書情報**
- 文書ID: Doc-SP-001
- バージョン: 1.0.0
- ステータス: 完了
- 最終更新: 2026-01-21
- 次回レビュー: 2026-02-21

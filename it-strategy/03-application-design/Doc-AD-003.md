# Doc-AD-003: TripTripバックエンドサービス設計

## エグゼクティブサマリー

本文書は、TripTripバックエンドサービスの包括的なアーキテクチャ設計を定義します。既存のNode.js/Hono/Prismaスタックを基盤として、Netflix、Uber、Stripeレベルのバックエンドサービス品質を実現するための技術戦略を提示します。ドメイン駆動設計（DDD）によるビジネスロジックの構造化、OpenAPI駆動の型安全API設計、Prisma ORMによる堅牢なデータアクセス層、JWT/OAuth2.0による包括的な認証・認可システムを実装します。本設計は、エラーハンドリング、ロギング、バックグラウンドジョブ処理、外部API統合を含み、TripTripの技術ビジョン（Doc-TV-001）、システムアーキテクチャ（Doc-SA-001/002）、およびフロントエンドアーキテクチャ（Doc-AD-001/002）と完全に整合しています。

---

## 第1章：はじめに & コンテキスト

### 1.1 目的 & スコープ

#### 1.1.1 文書の目的

本文書は、TripTripバックエンドサービスの技術アーキテクチャを包括的に定義し、以下の目的を達成します。

```yaml
document_objectives:
  primary:
    - description: Node.js/HonoバックエンドAPIアーキテクチャの標準化
      deliverable: アーキテクチャガイドライン、コーディング規約

    - description: ドメイン駆動設計（DDD）の実装パターン確立
      deliverable: アグリゲート設計、リポジトリパターン、ドメインイベント

    - description: 認証・認可システムの包括的設計
      deliverable: JWT/OAuth2.0実装、RBAC設計

  secondary:
    - description: エラーハンドリングとロギング戦略
      deliverable: 構造化ログ、エラー分類、モニタリング統合

    - description: バックグラウンドジョブ処理の設計
      deliverable: キュー設計、ジョブスケジューリング

    - description: 外部API統合パターンの定義
      deliverable: 決済API、地図API、OTA API統合設計
```

#### 1.1.2 スコープ定義

```yaml
scope:
  in_scope:
    components:
      - RESTful API設計
      - ドメインモデル設計
      - データアクセス層（Prisma）
      - 認証・認可システム
      - エラーハンドリング
      - ロギング・監視
      - バックグラウンドジョブ
      - 外部API統合

    technologies:
      - Node.js 20+ / TypeScript 5.x
      - Hono Framework
      - Prisma ORM
      - PostgreSQL 16
      - Redis（キャッシュ・セッション）
      - BullMQ（ジョブキュー）

  out_of_scope:
    - モバイルアプリケーション設計（Doc-AD-001）
    - Webフロントエンド設計（Doc-AD-002）
    - インフラストラクチャ設計（Doc-IA-001）
    - マイクロサービス分割戦略（Doc-SA-002）
```

### 1.2 既存バックエンドの分析

#### 1.2.1 現行アーキテクチャ評価

```yaml
current_state:
  codebase_metrics:
    total_lines: 4,738
    typescript_files: 22
    api_routes: 5 (users, products, attractions, orders, trips)
    version: "Node.js 18+, Hono 4.8.3"

  technology_stack:
    runtime: Node.js 18+
    framework: Hono 4.8.3
    orm: Prisma 5.8.0
    database: PostgreSQL 16 (Docker)
    validation: Zod 3.25.67
    documentation: "@hono/swagger-ui, @hono/zod-openapi"
    authentication: "JWT (jsonwebtoken 9.0.2)"

  architecture_patterns:
    current: Layered Architecture
    structure: |
      backend/src/hono/
      ├── routes/           # HTTPエンドポイント
      ├── schemas/          # Zodバリデーションスキーマ
      ├── middlewares/      # ミドルウェア
      └── models/           # Prismaモデル

  api_documentation:
    format: OpenAPI 3.0
    ui: Swagger UI at /doc
```

#### 1.2.2 成熟度評価と課題

```yaml
maturity_assessment:
  strengths:
    - 軽量で高速なHonoフレームワーク
    - Zodによる型安全なバリデーション
    - OpenAPI仕様の自動生成
    - Prismaによる型安全なDB操作

  challenges:
    technical_debt:
      - id: TD-B001
        description: ビジネスロジックのルートファイル混在
        impact: 保守性低下、テスト困難
        remediation: ドメイン層の分離

      - id: TD-B002
        description: エラーハンドリングの不統一
        impact: デバッグ困難、ユーザー体験低下
        remediation: 統一エラーハンドラーの実装

      - id: TD-B003
        description: ロギングの不足
        impact: 障害調査困難
        remediation: 構造化ログの導入

      - id: TD-B004
        description: テストカバレッジ不足
        impact: リグレッションリスク
        remediation: テスト戦略の確立

  gaps:
    - バックグラウンドジョブ処理の欠如
    - リアルタイム機能（WebSocket）の未実装
    - 分散トレーシングの未導入
    - レート制限の未実装
```

### 1.3 スケーリング要件

#### 1.3.1 パフォーマンス目標

```yaml
performance_targets:
  latency:
    p50: "< 50ms"
    p95: "< 100ms"
    p99: "< 200ms"

  throughput:
    year_1: "1,000 req/s"
    year_2: "10,000 req/s"
    year_3: "50,000 req/s"

  availability:
    target: "99.9%"
    recovery_time_objective: "5 minutes"
    recovery_point_objective: "1 minute"

  scalability:
    horizontal: true
    auto_scaling: true
    stateless_design: required
```

---

## 第2章：APIアーキテクチャ

### 2.1 Honoフレームワーク活用

#### 2.1.1 プロジェクト構造

```
backend/
├── src/
│   ├── app.ts                        # アプリケーションエントリー
│   ├── server.ts                     # サーバー起動
│   │
│   ├── api/                          # API層
│   │   ├── routes/                   # ルート定義
│   │   │   ├── index.ts              # ルート集約
│   │   │   ├── auth.routes.ts
│   │   │   ├── users.routes.ts
│   │   │   ├── hotels.routes.ts
│   │   │   ├── bookings.routes.ts
│   │   │   ├── products.routes.ts
│   │   │   └── payments.routes.ts
│   │   │
│   │   ├── middlewares/              # ミドルウェア
│   │   │   ├── auth.middleware.ts
│   │   │   ├── error.middleware.ts
│   │   │   ├── logging.middleware.ts
│   │   │   ├── rate-limit.middleware.ts
│   │   │   ├── cors.middleware.ts
│   │   │   └── validation.middleware.ts
│   │   │
│   │   ├── schemas/                  # リクエスト/レスポンススキーマ
│   │   │   ├── auth.schema.ts
│   │   │   ├── user.schema.ts
│   │   │   ├── hotel.schema.ts
│   │   │   ├── booking.schema.ts
│   │   │   └── common.schema.ts
│   │   │
│   │   └── dto/                      # Data Transfer Objects
│   │       ├── auth.dto.ts
│   │       ├── user.dto.ts
│   │       └── hotel.dto.ts
│   │
│   ├── domain/                       # ドメイン層
│   │   ├── entities/                 # エンティティ
│   │   │   ├── user.entity.ts
│   │   │   ├── hotel.entity.ts
│   │   │   ├── booking.entity.ts
│   │   │   └── order.entity.ts
│   │   │
│   │   ├── value-objects/            # 値オブジェクト
│   │   │   ├── email.vo.ts
│   │   │   ├── money.vo.ts
│   │   │   ├── date-range.vo.ts
│   │   │   └── location.vo.ts
│   │   │
│   │   ├── aggregates/               # アグリゲート
│   │   │   ├── booking.aggregate.ts
│   │   │   └── order.aggregate.ts
│   │   │
│   │   ├── repositories/             # リポジトリインターフェース
│   │   │   ├── user.repository.ts
│   │   │   ├── hotel.repository.ts
│   │   │   └── booking.repository.ts
│   │   │
│   │   ├── services/                 # ドメインサービス
│   │   │   ├── pricing.service.ts
│   │   │   ├── availability.service.ts
│   │   │   └── booking.service.ts
│   │   │
│   │   └── events/                   # ドメインイベント
│   │       ├── booking-created.event.ts
│   │       ├── payment-completed.event.ts
│   │       └── event-dispatcher.ts
│   │
│   ├── application/                  # アプリケーション層
│   │   ├── use-cases/                # ユースケース
│   │   │   ├── auth/
│   │   │   │   ├── login.use-case.ts
│   │   │   │   ├── register.use-case.ts
│   │   │   │   └── refresh-token.use-case.ts
│   │   │   ├── booking/
│   │   │   │   ├── create-booking.use-case.ts
│   │   │   │   ├── cancel-booking.use-case.ts
│   │   │   │   └── get-booking.use-case.ts
│   │   │   └── hotel/
│   │   │       ├── search-hotels.use-case.ts
│   │   │       └── get-hotel-detail.use-case.ts
│   │   │
│   │   └── services/                 # アプリケーションサービス
│   │       ├── auth.service.ts
│   │       ├── notification.service.ts
│   │       └── email.service.ts
│   │
│   ├── infrastructure/               # インフラストラクチャ層
│   │   ├── database/                 # データベース
│   │   │   ├── prisma/
│   │   │   │   ├── schema.prisma
│   │   │   │   ├── migrations/
│   │   │   │   └── seed.ts
│   │   │   ├── repositories/         # リポジトリ実装
│   │   │   │   ├── user.repository.impl.ts
│   │   │   │   ├── hotel.repository.impl.ts
│   │   │   │   └── booking.repository.impl.ts
│   │   │   └── prisma-client.ts
│   │   │
│   │   ├── cache/                    # キャッシュ
│   │   │   ├── redis-client.ts
│   │   │   └── cache.service.ts
│   │   │
│   │   ├── queue/                    # キュー
│   │   │   ├── bull-client.ts
│   │   │   ├── queues/
│   │   │   │   ├── email.queue.ts
│   │   │   │   ├── notification.queue.ts
│   │   │   │   └── analytics.queue.ts
│   │   │   └── workers/
│   │   │       ├── email.worker.ts
│   │   │       └── notification.worker.ts
│   │   │
│   │   ├── external/                 # 外部サービス
│   │   │   ├── stripe/
│   │   │   │   ├── stripe.client.ts
│   │   │   │   └── stripe.service.ts
│   │   │   ├── sendgrid/
│   │   │   │   └── sendgrid.service.ts
│   │   │   └── maps/
│   │   │       └── google-maps.service.ts
│   │   │
│   │   └── logging/                  # ロギング
│   │       ├── logger.ts
│   │       └── request-logger.ts
│   │
│   ├── shared/                       # 共有モジュール
│   │   ├── errors/                   # エラー定義
│   │   │   ├── base.error.ts
│   │   │   ├── validation.error.ts
│   │   │   ├── not-found.error.ts
│   │   │   └── unauthorized.error.ts
│   │   │
│   │   ├── types/                    # 型定義
│   │   │   ├── common.types.ts
│   │   │   └── result.types.ts
│   │   │
│   │   ├── utils/                    # ユーティリティ
│   │   │   ├── date.utils.ts
│   │   │   ├── crypto.utils.ts
│   │   │   └── validators.ts
│   │   │
│   │   └── constants/                # 定数
│   │       ├── http-status.ts
│   │       └── error-codes.ts
│   │
│   └── config/                       # 設定
│       ├── app.config.ts
│       ├── database.config.ts
│       ├── auth.config.ts
│       └── env.ts
│
├── tests/                            # テスト
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── prisma/                           # Prisma設定
│   ├── schema.prisma
│   └── migrations/
│
├── package.json
├── tsconfig.json
└── vitest.config.ts
```

### 2.2 ルーティングとミドルウェア設計

#### 2.2.1 ルーティング実装

```typescript
// ==========================================
// src/app.ts
// ==========================================

import { Hono } from 'hono';
import { cors } from 'hono/cors';
import { secureHeaders } from 'hono/secure-headers';
import { timing } from 'hono/timing';
import { compress } from 'hono/compress';
import { swaggerUI } from '@hono/swagger-ui';
import { OpenAPIHono } from '@hono/zod-openapi';
import { errorMiddleware } from './api/middlewares/error.middleware';
import { loggingMiddleware } from './api/middlewares/logging.middleware';
import { rateLimitMiddleware } from './api/middlewares/rate-limit.middleware';
import { apiRoutes } from './api/routes';
import { AppConfig } from './config/app.config';
import { logger } from './infrastructure/logging/logger';

// OpenAPI対応のHonoアプリケーション
const app = new OpenAPIHono();

// グローバルミドルウェア
app.use('*', timing());
app.use('*', compress());
app.use('*', secureHeaders());
app.use('*', cors({
  origin: AppConfig.cors.allowedOrigins,
  credentials: true,
  allowMethods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  allowHeaders: ['Content-Type', 'Authorization', 'X-Request-ID'],
  exposeHeaders: ['X-Request-ID', 'X-RateLimit-Remaining'],
}));
app.use('*', loggingMiddleware);
app.use('*', rateLimitMiddleware);

// APIルート
app.route('/api/v1', apiRoutes);

// OpenAPIドキュメント
app.doc('/api/v1/openapi.json', {
  openapi: '3.0.0',
  info: {
    title: 'TripTrip API',
    version: '1.0.0',
    description: 'TripTrip旅行プラットフォームAPI',
  },
  servers: [
    { url: AppConfig.apiBaseUrl, description: 'Production' },
  ],
  tags: [
    { name: 'Auth', description: '認証関連' },
    { name: 'Users', description: 'ユーザー管理' },
    { name: 'Hotels', description: 'ホテル検索・予約' },
    { name: 'Bookings', description: '予約管理' },
    { name: 'Products', description: '商品管理' },
    { name: 'Payments', description: '決済処理' },
  ],
});

// Swagger UI
app.get('/doc', swaggerUI({ url: '/api/v1/openapi.json' }));

// ヘルスチェック
app.get('/health', (c) => c.json({ status: 'healthy', timestamp: new Date().toISOString() }));

// グローバルエラーハンドラー
app.onError(errorMiddleware);

// 404ハンドラー
app.notFound((c) => {
  return c.json({
    success: false,
    error: {
      code: 'NOT_FOUND',
      message: 'Requested resource not found',
    },
  }, 404);
});

export { app };

// ==========================================
// src/api/routes/index.ts
// ==========================================

import { OpenAPIHono } from '@hono/zod-openapi';
import { authRoutes } from './auth.routes';
import { userRoutes } from './users.routes';
import { hotelRoutes } from './hotels.routes';
import { bookingRoutes } from './bookings.routes';
import { productRoutes } from './products.routes';
import { paymentRoutes } from './payments.routes';

const apiRoutes = new OpenAPIHono();

// ルートの登録
apiRoutes.route('/auth', authRoutes);
apiRoutes.route('/users', userRoutes);
apiRoutes.route('/hotels', hotelRoutes);
apiRoutes.route('/bookings', bookingRoutes);
apiRoutes.route('/products', productRoutes);
apiRoutes.route('/payments', paymentRoutes);

export { apiRoutes };

// ==========================================
// src/api/routes/hotels.routes.ts
// ==========================================

import { OpenAPIHono, createRoute, z } from '@hono/zod-openapi';
import { authMiddleware } from '../middlewares/auth.middleware';
import { SearchHotelsUseCase } from '../../application/use-cases/hotel/search-hotels.use-case';
import { GetHotelDetailUseCase } from '../../application/use-cases/hotel/get-hotel-detail.use-case';
import { HotelSearchSchema, HotelResponseSchema, HotelListResponseSchema } from '../schemas/hotel.schema';
import { container } from '../../infrastructure/di/container';

const hotelRoutes = new OpenAPIHono();

// ホテル検索
const searchHotelsRoute = createRoute({
  method: 'get',
  path: '/',
  tags: ['Hotels'],
  summary: 'ホテル検索',
  description: '条件を指定してホテルを検索します',
  request: {
    query: HotelSearchSchema,
  },
  responses: {
    200: {
      description: '検索結果',
      content: {
        'application/json': {
          schema: HotelListResponseSchema,
        },
      },
    },
  },
});

hotelRoutes.openapi(searchHotelsRoute, async (c) => {
  const query = c.req.valid('query');
  const useCase = container.resolve(SearchHotelsUseCase);
  const result = await useCase.execute(query);

  return c.json({
    success: true,
    data: result.hotels,
    pagination: {
      page: result.page,
      limit: result.limit,
      totalCount: result.totalCount,
      totalPages: Math.ceil(result.totalCount / result.limit),
    },
  });
});

// ホテル詳細取得
const getHotelRoute = createRoute({
  method: 'get',
  path: '/{hotelId}',
  tags: ['Hotels'],
  summary: 'ホテル詳細取得',
  description: '指定されたホテルの詳細情報を取得します',
  request: {
    params: z.object({
      hotelId: z.string().uuid(),
    }),
  },
  responses: {
    200: {
      description: 'ホテル詳細',
      content: {
        'application/json': {
          schema: HotelResponseSchema,
        },
      },
    },
    404: {
      description: 'ホテルが見つかりません',
    },
  },
});

hotelRoutes.openapi(getHotelRoute, async (c) => {
  const { hotelId } = c.req.valid('param');
  const useCase = container.resolve(GetHotelDetailUseCase);
  const result = await useCase.execute(hotelId);

  if (!result) {
    return c.json({
      success: false,
      error: { code: 'NOT_FOUND', message: 'Hotel not found' },
    }, 404);
  }

  return c.json({
    success: true,
    data: result,
  });
});

// ホテル空き状況確認
const checkAvailabilityRoute = createRoute({
  method: 'get',
  path: '/{hotelId}/availability',
  tags: ['Hotels'],
  summary: '空き状況確認',
  description: '指定された日付のホテル空き状況を確認します',
  request: {
    params: z.object({
      hotelId: z.string().uuid(),
    }),
    query: z.object({
      checkIn: z.string().date(),
      checkOut: z.string().date(),
      rooms: z.coerce.number().int().min(1).default(1),
    }),
  },
  responses: {
    200: {
      description: '空き状況',
    },
  },
});

hotelRoutes.openapi(checkAvailabilityRoute, async (c) => {
  const { hotelId } = c.req.valid('param');
  const query = c.req.valid('query');
  const useCase = container.resolve(CheckAvailabilityUseCase);
  const result = await useCase.execute({ hotelId, ...query });

  return c.json({
    success: true,
    data: result,
  });
});

export { hotelRoutes };
```

### 2.3 OpenAPI仕様とコード生成

#### 2.3.1 スキーマ定義

```typescript
// ==========================================
// src/api/schemas/hotel.schema.ts
// ==========================================

import { z } from '@hono/zod-openapi';

// 位置情報スキーマ
export const LocationSchema = z.object({
  latitude: z.number().min(-90).max(90),
  longitude: z.number().min(-180).max(180),
  address: z.string(),
  city: z.string(),
  prefecture: z.string(),
  postalCode: z.string(),
}).openapi('Location');

// 価格帯スキーマ
export const PriceRangeSchema = z.object({
  min: z.number().int().min(0),
  max: z.number().int().min(0),
  currency: z.string().length(3).default('JPY'),
}).openapi('PriceRange');

// ホテル画像スキーマ
export const HotelImageSchema = z.object({
  url: z.string().url(),
  alt: z.string().optional(),
  isPrimary: z.boolean().default(false),
}).openapi('HotelImage');

// ホテルスキーマ
export const HotelSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(200),
  description: z.string().max(5000),
  location: LocationSchema,
  rating: z.number().min(0).max(5).optional(),
  reviewCount: z.number().int().min(0).default(0),
  priceRange: PriceRangeSchema,
  amenities: z.array(z.string()),
  images: z.array(HotelImageSchema),
  starRating: z.number().int().min(1).max(5).optional(),
  checkInTime: z.string(),
  checkOutTime: z.string(),
  isAvailable: z.boolean(),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
}).openapi('Hotel');

// ホテル検索クエリスキーマ
export const HotelSearchSchema = z.object({
  location: z.string().optional(),
  checkIn: z.string().date().optional(),
  checkOut: z.string().date().optional(),
  guests: z.coerce.number().int().min(1).default(2),
  rooms: z.coerce.number().int().min(1).default(1),
  minPrice: z.coerce.number().int().min(0).optional(),
  maxPrice: z.coerce.number().int().min(0).optional(),
  amenities: z.string().transform(s => s.split(',')).optional(),
  rating: z.coerce.number().min(0).max(5).optional(),
  sortBy: z.enum(['relevance', 'price_asc', 'price_desc', 'rating', 'distance']).default('relevance'),
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
}).openapi('HotelSearchQuery');

// ホテルレスポンススキーマ
export const HotelResponseSchema = z.object({
  success: z.boolean(),
  data: HotelSchema,
}).openapi('HotelResponse');

// ホテルリストレスポンススキーマ
export const HotelListResponseSchema = z.object({
  success: z.boolean(),
  data: z.array(HotelSchema),
  pagination: z.object({
    page: z.number().int(),
    limit: z.number().int(),
    totalCount: z.number().int(),
    totalPages: z.number().int(),
  }),
}).openapi('HotelListResponse');

// 型エクスポート
export type Hotel = z.infer<typeof HotelSchema>;
export type HotelSearchQuery = z.infer<typeof HotelSearchSchema>;
export type HotelResponse = z.infer<typeof HotelResponseSchema>;
export type HotelListResponse = z.infer<typeof HotelListResponseSchema>;

// ==========================================
// src/api/schemas/booking.schema.ts
// ==========================================

import { z } from '@hono/zod-openapi';

// 予約ステータス
export const BookingStatusSchema = z.enum([
  'pending',
  'confirmed',
  'cancelled',
  'completed',
  'refunded',
]).openapi('BookingStatus');

// ゲスト情報スキーマ
export const GuestInfoSchema = z.object({
  firstName: z.string().min(1).max(50),
  lastName: z.string().min(1).max(50),
  email: z.string().email(),
  phone: z.string().optional(),
  nationality: z.string().length(2).optional(),
}).openapi('GuestInfo');

// 予約アイテムスキーマ
export const BookingItemSchema = z.object({
  id: z.string().uuid(),
  roomTypeId: z.string().uuid(),
  roomTypeName: z.string(),
  checkIn: z.string().date(),
  checkOut: z.string().date(),
  guests: z.number().int().min(1),
  pricePerNight: z.number().int().min(0),
  totalPrice: z.number().int().min(0),
  nights: z.number().int().min(1),
}).openapi('BookingItem');

// 予約スキーマ
export const BookingSchema = z.object({
  id: z.string().uuid(),
  userId: z.string().uuid(),
  hotelId: z.string().uuid(),
  hotelName: z.string(),
  status: BookingStatusSchema,
  guestInfo: GuestInfoSchema,
  items: z.array(BookingItemSchema),
  subtotal: z.number().int().min(0),
  tax: z.number().int().min(0),
  total: z.number().int().min(0),
  currency: z.string().length(3).default('JPY'),
  specialRequests: z.string().optional(),
  paymentId: z.string().uuid().optional(),
  confirmedAt: z.string().datetime().optional(),
  cancelledAt: z.string().datetime().optional(),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
}).openapi('Booking');

// 予約作成リクエストスキーマ
export const CreateBookingSchema = z.object({
  hotelId: z.string().uuid(),
  roomTypeId: z.string().uuid(),
  checkIn: z.string().date(),
  checkOut: z.string().date(),
  guests: z.number().int().min(1),
  guestInfo: GuestInfoSchema,
  specialRequests: z.string().max(1000).optional(),
}).openapi('CreateBookingRequest');

// 予約レスポンススキーマ
export const BookingResponseSchema = z.object({
  success: z.boolean(),
  data: BookingSchema,
}).openapi('BookingResponse');

export type Booking = z.infer<typeof BookingSchema>;
export type CreateBookingRequest = z.infer<typeof CreateBookingSchema>;
```

---

## 第3章：ドメイン層設計

### 3.1 DDDアグリゲートとエンティティ

#### 3.1.1 エンティティ設計

```typescript
// ==========================================
// src/domain/entities/base.entity.ts
// ==========================================

import { randomUUID } from 'crypto';

export abstract class BaseEntity<T> {
  protected readonly _id: string;
  protected readonly _createdAt: Date;
  protected _updatedAt: Date;

  constructor(props: { id?: string; createdAt?: Date; updatedAt?: Date }) {
    this._id = props.id ?? randomUUID();
    this._createdAt = props.createdAt ?? new Date();
    this._updatedAt = props.updatedAt ?? new Date();
  }

  get id(): string {
    return this._id;
  }

  get createdAt(): Date {
    return this._createdAt;
  }

  get updatedAt(): Date {
    return this._updatedAt;
  }

  protected touch(): void {
    this._updatedAt = new Date();
  }

  equals(other: BaseEntity<T>): boolean {
    return this._id === other._id;
  }
}

// ==========================================
// src/domain/entities/user.entity.ts
// ==========================================

import { BaseEntity } from './base.entity';
import { Email } from '../value-objects/email.vo';
import { Password } from '../value-objects/password.vo';

export interface UserProps {
  id?: string;
  email: Email;
  password: Password;
  name: string;
  avatarUrl?: string;
  language: string;
  currency: string;
  emailVerified: boolean;
  mfaEnabled: boolean;
  createdAt?: Date;
  updatedAt?: Date;
}

export class User extends BaseEntity<UserProps> {
  private _email: Email;
  private _password: Password;
  private _name: string;
  private _avatarUrl?: string;
  private _language: string;
  private _currency: string;
  private _emailVerified: boolean;
  private _mfaEnabled: boolean;

  private constructor(props: UserProps) {
    super(props);
    this._email = props.email;
    this._password = props.password;
    this._name = props.name;
    this._avatarUrl = props.avatarUrl;
    this._language = props.language;
    this._currency = props.currency;
    this._emailVerified = props.emailVerified;
    this._mfaEnabled = props.mfaEnabled;
  }

  static create(props: Omit<UserProps, 'emailVerified' | 'mfaEnabled'>): User {
    return new User({
      ...props,
      emailVerified: false,
      mfaEnabled: false,
    });
  }

  static reconstitute(props: UserProps): User {
    return new User(props);
  }

  // Getters
  get email(): Email {
    return this._email;
  }

  get name(): string {
    return this._name;
  }

  get avatarUrl(): string | undefined {
    return this._avatarUrl;
  }

  get language(): string {
    return this._language;
  }

  get currency(): string {
    return this._currency;
  }

  get emailVerified(): boolean {
    return this._emailVerified;
  }

  get mfaEnabled(): boolean {
    return this._mfaEnabled;
  }

  // Domain Methods
  verifyPassword(plainPassword: string): boolean {
    return this._password.verify(plainPassword);
  }

  changePassword(newPassword: Password): void {
    this._password = newPassword;
    this.touch();
  }

  updateProfile(props: { name?: string; avatarUrl?: string }): void {
    if (props.name) this._name = props.name;
    if (props.avatarUrl !== undefined) this._avatarUrl = props.avatarUrl;
    this.touch();
  }

  updatePreferences(props: { language?: string; currency?: string }): void {
    if (props.language) this._language = props.language;
    if (props.currency) this._currency = props.currency;
    this.touch();
  }

  verifyEmail(): void {
    this._emailVerified = true;
    this.touch();
  }

  enableMfa(): void {
    this._mfaEnabled = true;
    this.touch();
  }

  disableMfa(): void {
    this._mfaEnabled = false;
    this.touch();
  }

  toJSON(): Record<string, unknown> {
    return {
      id: this._id,
      email: this._email.value,
      name: this._name,
      avatarUrl: this._avatarUrl,
      language: this._language,
      currency: this._currency,
      emailVerified: this._emailVerified,
      mfaEnabled: this._mfaEnabled,
      createdAt: this._createdAt.toISOString(),
      updatedAt: this._updatedAt.toISOString(),
    };
  }
}

// ==========================================
// src/domain/aggregates/booking.aggregate.ts
// ==========================================

import { BaseEntity } from '../entities/base.entity';
import { Money } from '../value-objects/money.vo';
import { DateRange } from '../value-objects/date-range.vo';
import { BookingCreatedEvent } from '../events/booking-created.event';
import { BookingConfirmedEvent } from '../events/booking-confirmed.event';
import { BookingCancelledEvent } from '../events/booking-cancelled.event';
import { DomainEvent } from '../events/domain-event';

export type BookingStatus = 'pending' | 'confirmed' | 'cancelled' | 'completed' | 'refunded';

export interface BookingItemProps {
  id: string;
  roomTypeId: string;
  roomTypeName: string;
  dateRange: DateRange;
  guests: number;
  pricePerNight: Money;
}

export class BookingItem {
  readonly id: string;
  readonly roomTypeId: string;
  readonly roomTypeName: string;
  readonly dateRange: DateRange;
  readonly guests: number;
  readonly pricePerNight: Money;

  constructor(props: BookingItemProps) {
    this.id = props.id;
    this.roomTypeId = props.roomTypeId;
    this.roomTypeName = props.roomTypeName;
    this.dateRange = props.dateRange;
    this.guests = props.guests;
    this.pricePerNight = props.pricePerNight;
  }

  get nights(): number {
    return this.dateRange.nights;
  }

  get totalPrice(): Money {
    return this.pricePerNight.multiply(this.nights);
  }
}

export interface GuestInfo {
  firstName: string;
  lastName: string;
  email: string;
  phone?: string;
  nationality?: string;
}

export interface BookingProps {
  id?: string;
  userId: string;
  hotelId: string;
  hotelName: string;
  status: BookingStatus;
  guestInfo: GuestInfo;
  items: BookingItem[];
  specialRequests?: string;
  paymentId?: string;
  confirmedAt?: Date;
  cancelledAt?: Date;
  createdAt?: Date;
  updatedAt?: Date;
}

export class Booking extends BaseEntity<BookingProps> {
  private _userId: string;
  private _hotelId: string;
  private _hotelName: string;
  private _status: BookingStatus;
  private _guestInfo: GuestInfo;
  private _items: BookingItem[];
  private _specialRequests?: string;
  private _paymentId?: string;
  private _confirmedAt?: Date;
  private _cancelledAt?: Date;

  // ドメインイベント
  private _domainEvents: DomainEvent[] = [];

  private constructor(props: BookingProps) {
    super(props);
    this._userId = props.userId;
    this._hotelId = props.hotelId;
    this._hotelName = props.hotelName;
    this._status = props.status;
    this._guestInfo = props.guestInfo;
    this._items = props.items;
    this._specialRequests = props.specialRequests;
    this._paymentId = props.paymentId;
    this._confirmedAt = props.confirmedAt;
    this._cancelledAt = props.cancelledAt;
  }

  // ファクトリメソッド
  static create(props: Omit<BookingProps, 'status' | 'confirmedAt' | 'cancelledAt'>): Booking {
    const booking = new Booking({
      ...props,
      status: 'pending',
    });

    // ドメインイベントの発行
    booking.addDomainEvent(new BookingCreatedEvent({
      bookingId: booking.id,
      userId: booking._userId,
      hotelId: booking._hotelId,
      total: booking.total.amount,
      currency: booking.total.currency,
    }));

    return booking;
  }

  static reconstitute(props: BookingProps): Booking {
    return new Booking(props);
  }

  // Getters
  get userId(): string {
    return this._userId;
  }

  get hotelId(): string {
    return this._hotelId;
  }

  get hotelName(): string {
    return this._hotelName;
  }

  get status(): BookingStatus {
    return this._status;
  }

  get guestInfo(): GuestInfo {
    return { ...this._guestInfo };
  }

  get items(): BookingItem[] {
    return [...this._items];
  }

  get specialRequests(): string | undefined {
    return this._specialRequests;
  }

  get paymentId(): string | undefined {
    return this._paymentId;
  }

  get confirmedAt(): Date | undefined {
    return this._confirmedAt;
  }

  get cancelledAt(): Date | undefined {
    return this._cancelledAt;
  }

  get domainEvents(): DomainEvent[] {
    return [...this._domainEvents];
  }

  // 計算プロパティ
  get subtotal(): Money {
    return this._items.reduce(
      (sum, item) => sum.add(item.totalPrice),
      Money.zero('JPY')
    );
  }

  get tax(): Money {
    return this.subtotal.multiply(0.1); // 10%消費税
  }

  get total(): Money {
    return this.subtotal.add(this.tax);
  }

  // ドメインメソッド
  confirm(paymentId: string): void {
    if (this._status !== 'pending') {
      throw new Error(`Cannot confirm booking with status: ${this._status}`);
    }

    this._status = 'confirmed';
    this._paymentId = paymentId;
    this._confirmedAt = new Date();
    this.touch();

    this.addDomainEvent(new BookingConfirmedEvent({
      bookingId: this.id,
      userId: this._userId,
      paymentId,
      confirmedAt: this._confirmedAt,
    }));
  }

  cancel(reason?: string): void {
    if (this._status === 'completed' || this._status === 'cancelled') {
      throw new Error(`Cannot cancel booking with status: ${this._status}`);
    }

    const previousStatus = this._status;
    this._status = 'cancelled';
    this._cancelledAt = new Date();
    this.touch();

    this.addDomainEvent(new BookingCancelledEvent({
      bookingId: this.id,
      userId: this._userId,
      previousStatus,
      reason,
      cancelledAt: this._cancelledAt,
    }));
  }

  complete(): void {
    if (this._status !== 'confirmed') {
      throw new Error(`Cannot complete booking with status: ${this._status}`);
    }

    this._status = 'completed';
    this.touch();
  }

  markAsRefunded(): void {
    if (this._status !== 'cancelled') {
      throw new Error(`Cannot refund booking with status: ${this._status}`);
    }

    this._status = 'refunded';
    this.touch();
  }

  private addDomainEvent(event: DomainEvent): void {
    this._domainEvents.push(event);
  }

  clearDomainEvents(): void {
    this._domainEvents = [];
  }

  toJSON(): Record<string, unknown> {
    return {
      id: this._id,
      userId: this._userId,
      hotelId: this._hotelId,
      hotelName: this._hotelName,
      status: this._status,
      guestInfo: this._guestInfo,
      items: this._items.map(item => ({
        id: item.id,
        roomTypeId: item.roomTypeId,
        roomTypeName: item.roomTypeName,
        checkIn: item.dateRange.startDate.toISOString(),
        checkOut: item.dateRange.endDate.toISOString(),
        guests: item.guests,
        nights: item.nights,
        pricePerNight: item.pricePerNight.amount,
        totalPrice: item.totalPrice.amount,
      })),
      subtotal: this.subtotal.amount,
      tax: this.tax.amount,
      total: this.total.amount,
      currency: this.total.currency,
      specialRequests: this._specialRequests,
      paymentId: this._paymentId,
      confirmedAt: this._confirmedAt?.toISOString(),
      cancelledAt: this._cancelledAt?.toISOString(),
      createdAt: this._createdAt.toISOString(),
      updatedAt: this._updatedAt.toISOString(),
    };
  }
}
```

### 3.2 リポジトリパターン

```typescript
// ==========================================
// src/domain/repositories/booking.repository.ts
// ==========================================

import { Booking } from '../aggregates/booking.aggregate';

export interface BookingSearchCriteria {
  userId?: string;
  hotelId?: string;
  status?: string[];
  checkInFrom?: Date;
  checkInTo?: Date;
  page?: number;
  limit?: number;
}

export interface BookingSearchResult {
  bookings: Booking[];
  totalCount: number;
  page: number;
  limit: number;
}

// リポジトリインターフェース（ドメイン層）
export interface BookingRepository {
  findById(id: string): Promise<Booking | null>;
  findByUserId(userId: string): Promise<Booking[]>;
  search(criteria: BookingSearchCriteria): Promise<BookingSearchResult>;
  save(booking: Booking): Promise<void>;
  delete(id: string): Promise<void>;
}

// ==========================================
// src/infrastructure/database/repositories/booking.repository.impl.ts
// ==========================================

import { PrismaClient } from '@prisma/client';
import { Booking, BookingItem, GuestInfo } from '../../../domain/aggregates/booking.aggregate';
import { BookingRepository, BookingSearchCriteria, BookingSearchResult } from '../../../domain/repositories/booking.repository';
import { Money } from '../../../domain/value-objects/money.vo';
import { DateRange } from '../../../domain/value-objects/date-range.vo';
import { EventDispatcher } from '../../../domain/events/event-dispatcher';

export class BookingRepositoryImpl implements BookingRepository {
  constructor(
    private readonly prisma: PrismaClient,
    private readonly eventDispatcher: EventDispatcher
  ) {}

  async findById(id: string): Promise<Booking | null> {
    const data = await this.prisma.booking.findUnique({
      where: { id },
      include: {
        items: true,
      },
    });

    if (!data) return null;

    return this.toDomain(data);
  }

  async findByUserId(userId: string): Promise<Booking[]> {
    const data = await this.prisma.booking.findMany({
      where: { userId },
      include: {
        items: true,
      },
      orderBy: { createdAt: 'desc' },
    });

    return data.map(d => this.toDomain(d));
  }

  async search(criteria: BookingSearchCriteria): Promise<BookingSearchResult> {
    const {
      userId,
      hotelId,
      status,
      checkInFrom,
      checkInTo,
      page = 1,
      limit = 20,
    } = criteria;

    const where: any = {};
    if (userId) where.userId = userId;
    if (hotelId) where.hotelId = hotelId;
    if (status?.length) where.status = { in: status };
    if (checkInFrom || checkInTo) {
      where.items = {
        some: {
          checkIn: {
            ...(checkInFrom && { gte: checkInFrom }),
            ...(checkInTo && { lte: checkInTo }),
          },
        },
      };
    }

    const [data, totalCount] = await Promise.all([
      this.prisma.booking.findMany({
        where,
        include: { items: true },
        orderBy: { createdAt: 'desc' },
        skip: (page - 1) * limit,
        take: limit,
      }),
      this.prisma.booking.count({ where }),
    ]);

    return {
      bookings: data.map(d => this.toDomain(d)),
      totalCount,
      page,
      limit,
    };
  }

  async save(booking: Booking): Promise<void> {
    const data = this.toPersistence(booking);

    await this.prisma.$transaction(async (tx) => {
      // 既存の予約を削除（items含む）
      await tx.bookingItem.deleteMany({
        where: { bookingId: booking.id },
      });

      // 予約をupsert
      await tx.booking.upsert({
        where: { id: booking.id },
        create: {
          id: data.id,
          userId: data.userId,
          hotelId: data.hotelId,
          hotelName: data.hotelName,
          status: data.status,
          guestFirstName: data.guestFirstName,
          guestLastName: data.guestLastName,
          guestEmail: data.guestEmail,
          guestPhone: data.guestPhone,
          guestNationality: data.guestNationality,
          subtotal: data.subtotal,
          tax: data.tax,
          total: data.total,
          currency: data.currency,
          specialRequests: data.specialRequests,
          paymentId: data.paymentId,
          confirmedAt: data.confirmedAt,
          cancelledAt: data.cancelledAt,
          createdAt: data.createdAt,
          updatedAt: data.updatedAt,
          items: {
            create: data.items,
          },
        },
        update: {
          status: data.status,
          guestFirstName: data.guestFirstName,
          guestLastName: data.guestLastName,
          guestEmail: data.guestEmail,
          guestPhone: data.guestPhone,
          guestNationality: data.guestNationality,
          subtotal: data.subtotal,
          tax: data.tax,
          total: data.total,
          specialRequests: data.specialRequests,
          paymentId: data.paymentId,
          confirmedAt: data.confirmedAt,
          cancelledAt: data.cancelledAt,
          updatedAt: data.updatedAt,
          items: {
            create: data.items,
          },
        },
      });
    });

    // ドメインイベントの発行
    for (const event of booking.domainEvents) {
      await this.eventDispatcher.dispatch(event);
    }
    booking.clearDomainEvents();
  }

  async delete(id: string): Promise<void> {
    await this.prisma.booking.delete({
      where: { id },
    });
  }

  // 永続化データからドメインオブジェクトへの変換
  private toDomain(data: any): Booking {
    const items = data.items.map((item: any) => new BookingItem({
      id: item.id,
      roomTypeId: item.roomTypeId,
      roomTypeName: item.roomTypeName,
      dateRange: DateRange.create(item.checkIn, item.checkOut),
      guests: item.guests,
      pricePerNight: new Money(item.pricePerNight, item.currency),
    }));

    const guestInfo: GuestInfo = {
      firstName: data.guestFirstName,
      lastName: data.guestLastName,
      email: data.guestEmail,
      phone: data.guestPhone,
      nationality: data.guestNationality,
    };

    return Booking.reconstitute({
      id: data.id,
      userId: data.userId,
      hotelId: data.hotelId,
      hotelName: data.hotelName,
      status: data.status,
      guestInfo,
      items,
      specialRequests: data.specialRequests,
      paymentId: data.paymentId,
      confirmedAt: data.confirmedAt,
      cancelledAt: data.cancelledAt,
      createdAt: data.createdAt,
      updatedAt: data.updatedAt,
    });
  }

  // ドメインオブジェクトから永続化データへの変換
  private toPersistence(booking: Booking): any {
    const json = booking.toJSON() as any;
    return {
      id: json.id,
      userId: json.userId,
      hotelId: json.hotelId,
      hotelName: json.hotelName,
      status: json.status,
      guestFirstName: json.guestInfo.firstName,
      guestLastName: json.guestInfo.lastName,
      guestEmail: json.guestInfo.email,
      guestPhone: json.guestInfo.phone,
      guestNationality: json.guestInfo.nationality,
      subtotal: json.subtotal,
      tax: json.tax,
      total: json.total,
      currency: json.currency,
      specialRequests: json.specialRequests,
      paymentId: json.paymentId,
      confirmedAt: json.confirmedAt ? new Date(json.confirmedAt) : null,
      cancelledAt: json.cancelledAt ? new Date(json.cancelledAt) : null,
      createdAt: new Date(json.createdAt),
      updatedAt: new Date(json.updatedAt),
      items: json.items.map((item: any) => ({
        id: item.id,
        roomTypeId: item.roomTypeId,
        roomTypeName: item.roomTypeName,
        checkIn: new Date(item.checkIn),
        checkOut: new Date(item.checkOut),
        guests: item.guests,
        pricePerNight: item.pricePerNight,
        currency: json.currency,
      })),
    };
  }
}
```

### 3.3 ドメインイベント

```typescript
// ==========================================
// src/domain/events/domain-event.ts
// ==========================================

export interface DomainEvent {
  readonly eventType: string;
  readonly occurredAt: Date;
  readonly aggregateId: string;
  toJSON(): Record<string, unknown>;
}

// ==========================================
// src/domain/events/booking-created.event.ts
// ==========================================

import { DomainEvent } from './domain-event';

export interface BookingCreatedEventProps {
  bookingId: string;
  userId: string;
  hotelId: string;
  total: number;
  currency: string;
}

export class BookingCreatedEvent implements DomainEvent {
  readonly eventType = 'booking.created';
  readonly occurredAt: Date;
  readonly aggregateId: string;

  constructor(private readonly props: BookingCreatedEventProps) {
    this.occurredAt = new Date();
    this.aggregateId = props.bookingId;
  }

  get bookingId(): string {
    return this.props.bookingId;
  }

  get userId(): string {
    return this.props.userId;
  }

  get hotelId(): string {
    return this.props.hotelId;
  }

  get total(): number {
    return this.props.total;
  }

  get currency(): string {
    return this.props.currency;
  }

  toJSON(): Record<string, unknown> {
    return {
      eventType: this.eventType,
      occurredAt: this.occurredAt.toISOString(),
      aggregateId: this.aggregateId,
      ...this.props,
    };
  }
}

// ==========================================
// src/domain/events/event-dispatcher.ts
// ==========================================

import { DomainEvent } from './domain-event';

export type EventHandler<T extends DomainEvent> = (event: T) => Promise<void>;

export interface EventDispatcher {
  dispatch(event: DomainEvent): Promise<void>;
  subscribe<T extends DomainEvent>(eventType: string, handler: EventHandler<T>): void;
}

// ==========================================
// src/infrastructure/events/event-dispatcher.impl.ts
// ==========================================

import { DomainEvent } from '../../domain/events/domain-event';
import { EventDispatcher, EventHandler } from '../../domain/events/event-dispatcher';
import { logger } from '../logging/logger';

export class EventDispatcherImpl implements EventDispatcher {
  private handlers: Map<string, EventHandler<any>[]> = new Map();

  subscribe<T extends DomainEvent>(eventType: string, handler: EventHandler<T>): void {
    const existing = this.handlers.get(eventType) || [];
    this.handlers.set(eventType, [...existing, handler]);
    logger.debug(`Subscribed handler to event: ${eventType}`);
  }

  async dispatch(event: DomainEvent): Promise<void> {
    const handlers = this.handlers.get(event.eventType) || [];

    logger.info(`Dispatching event: ${event.eventType}`, {
      eventType: event.eventType,
      aggregateId: event.aggregateId,
    });

    for (const handler of handlers) {
      try {
        await handler(event);
      } catch (error) {
        logger.error(`Error handling event: ${event.eventType}`, {
          error,
          eventType: event.eventType,
          aggregateId: event.aggregateId,
        });
        // エラーが発生しても他のハンドラーは継続
      }
    }
  }
}

// ==========================================
// src/application/event-handlers/index.ts
// ==========================================

import { EventDispatcher } from '../../domain/events/event-dispatcher';
import { BookingCreatedEvent } from '../../domain/events/booking-created.event';
import { BookingConfirmedEvent } from '../../domain/events/booking-confirmed.event';
import { BookingCancelledEvent } from '../../domain/events/booking-cancelled.event';
import { NotificationService } from '../services/notification.service';
import { AnalyticsService } from '../services/analytics.service';

export function registerEventHandlers(
  dispatcher: EventDispatcher,
  notificationService: NotificationService,
  analyticsService: AnalyticsService
): void {
  // 予約作成イベント
  dispatcher.subscribe<BookingCreatedEvent>('booking.created', async (event) => {
    // 分析イベント送信
    await analyticsService.trackEvent('booking_created', {
      bookingId: event.bookingId,
      userId: event.userId,
      hotelId: event.hotelId,
      total: event.total,
      currency: event.currency,
    });
  });

  // 予約確定イベント
  dispatcher.subscribe<BookingConfirmedEvent>('booking.confirmed', async (event) => {
    // 確認メール送信
    await notificationService.sendBookingConfirmation(event.bookingId);

    // 分析イベント送信
    await analyticsService.trackEvent('booking_confirmed', {
      bookingId: event.bookingId,
      userId: event.userId,
      paymentId: event.paymentId,
    });
  });

  // 予約キャンセルイベント
  dispatcher.subscribe<BookingCancelledEvent>('booking.cancelled', async (event) => {
    // キャンセル通知送信
    await notificationService.sendBookingCancellation(event.bookingId, event.reason);

    // 分析イベント送信
    await analyticsService.trackEvent('booking_cancelled', {
      bookingId: event.bookingId,
      userId: event.userId,
      reason: event.reason,
    });
  });
}
```

---

## 第4章：データアクセス層

### 4.1 Prismaスキーマ設計

```prisma
// ==========================================
// prisma/schema.prisma
// ==========================================

generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["fullTextSearch", "fullTextIndex"]
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ==========================================
// ユーザー管理
// ==========================================

model User {
  id            String    @id @default(uuid())
  email         String    @unique
  passwordHash  String    @map("password_hash")
  name          String
  avatarUrl     String?   @map("avatar_url")
  language      String    @default("ja")
  currency      String    @default("JPY")
  emailVerified Boolean   @default(false) @map("email_verified")
  mfaEnabled    Boolean   @default(false) @map("mfa_enabled")
  mfaSecret     String?   @map("mfa_secret")
  createdAt     DateTime  @default(now()) @map("created_at")
  updatedAt     DateTime  @updatedAt @map("updated_at")

  // Relations
  accounts      Account[]
  sessions      Session[]
  bookings      Booking[]
  orders        Order[]
  favorites     Favorite[]
  reviews       Review[]

  @@index([email])
  @@map("users")
}

model Account {
  id                String  @id @default(uuid())
  userId            String  @map("user_id")
  provider          String  // google, line, apple
  providerAccountId String  @map("provider_account_id")
  refreshToken      String? @map("refresh_token")
  accessToken       String? @map("access_token")
  expiresAt         Int?    @map("expires_at")
  tokenType         String? @map("token_type")
  scope             String?
  createdAt         DateTime @default(now()) @map("created_at")
  updatedAt         DateTime @updatedAt @map("updated_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
  @@map("accounts")
}

model Session {
  id           String   @id @default(uuid())
  userId       String   @map("user_id")
  token        String   @unique
  deviceInfo   String?  @map("device_info")
  ipAddress    String?  @map("ip_address")
  expiresAt    DateTime @map("expires_at")
  createdAt    DateTime @default(now()) @map("created_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([expiresAt])
  @@map("sessions")
}

// ==========================================
// ホテル管理
// ==========================================

model Hotel {
  id           String   @id @default(uuid())
  name         String
  description  String
  latitude     Float
  longitude    Float
  address      String
  city         String
  prefecture   String
  postalCode   String   @map("postal_code")
  phone        String?
  email        String?
  website      String?
  rating       Float?
  reviewCount  Int      @default(0) @map("review_count")
  starRating   Int?     @map("star_rating")
  checkInTime  String   @map("check_in_time")
  checkOutTime String   @map("check_out_time")
  petsAllowed  Boolean  @default(false) @map("pets_allowed")
  isActive     Boolean  @default(true) @map("is_active")
  createdAt    DateTime @default(now()) @map("created_at")
  updatedAt    DateTime @updatedAt @map("updated_at")

  // Relations
  images      HotelImage[]
  amenities   HotelAmenity[]
  roomTypes   RoomType[]
  bookings    Booking[]
  favorites   Favorite[]
  reviews     Review[]

  @@index([city])
  @@index([rating])
  @@index([isActive])
  @@map("hotels")
}

model HotelImage {
  id        String   @id @default(uuid())
  hotelId   String   @map("hotel_id")
  url       String
  alt       String?
  isPrimary Boolean  @default(false) @map("is_primary")
  sortOrder Int      @default(0) @map("sort_order")
  createdAt DateTime @default(now()) @map("created_at")

  hotel Hotel @relation(fields: [hotelId], references: [id], onDelete: Cascade)

  @@index([hotelId])
  @@map("hotel_images")
}

model HotelAmenity {
  id        String   @id @default(uuid())
  hotelId   String   @map("hotel_id")
  name      String
  category  String?
  icon      String?
  createdAt DateTime @default(now()) @map("created_at")

  hotel Hotel @relation(fields: [hotelId], references: [id], onDelete: Cascade)

  @@index([hotelId])
  @@map("hotel_amenities")
}

model RoomType {
  id            String   @id @default(uuid())
  hotelId       String   @map("hotel_id")
  name          String
  description   String
  maxGuests     Int      @map("max_guests")
  bedType       String   @map("bed_type")
  size          Int?     // 平方メートル
  basePrice     Int      @map("base_price")
  currency      String   @default("JPY")
  totalRooms    Int      @map("total_rooms")
  isActive      Boolean  @default(true) @map("is_active")
  createdAt     DateTime @default(now()) @map("created_at")
  updatedAt     DateTime @updatedAt @map("updated_at")

  hotel          Hotel              @relation(fields: [hotelId], references: [id], onDelete: Cascade)
  images         RoomTypeImage[]
  amenities      RoomTypeAmenity[]
  inventory      RoomInventory[]
  bookingItems   BookingItem[]

  @@index([hotelId])
  @@map("room_types")
}

model RoomTypeImage {
  id         String   @id @default(uuid())
  roomTypeId String   @map("room_type_id")
  url        String
  alt        String?
  sortOrder  Int      @default(0) @map("sort_order")
  createdAt  DateTime @default(now()) @map("created_at")

  roomType RoomType @relation(fields: [roomTypeId], references: [id], onDelete: Cascade)

  @@map("room_type_images")
}

model RoomTypeAmenity {
  id         String   @id @default(uuid())
  roomTypeId String   @map("room_type_id")
  name       String
  icon       String?
  createdAt  DateTime @default(now()) @map("created_at")

  roomType RoomType @relation(fields: [roomTypeId], references: [id], onDelete: Cascade)

  @@map("room_type_amenities")
}

model RoomInventory {
  id          String   @id @default(uuid())
  roomTypeId  String   @map("room_type_id")
  date        DateTime @db.Date
  totalRooms  Int      @map("total_rooms")
  bookedRooms Int      @default(0) @map("booked_rooms")
  blockedRooms Int     @default(0) @map("blocked_rooms")
  price       Int?     // オーバーライド価格
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  roomType RoomType @relation(fields: [roomTypeId], references: [id], onDelete: Cascade)

  @@unique([roomTypeId, date])
  @@index([date])
  @@map("room_inventory")
}

// ==========================================
// 予約管理
// ==========================================

model Booking {
  id               String        @id @default(uuid())
  userId           String        @map("user_id")
  hotelId          String        @map("hotel_id")
  hotelName        String        @map("hotel_name")
  status           BookingStatus @default(PENDING)
  guestFirstName   String        @map("guest_first_name")
  guestLastName    String        @map("guest_last_name")
  guestEmail       String        @map("guest_email")
  guestPhone       String?       @map("guest_phone")
  guestNationality String?       @map("guest_nationality")
  subtotal         Int
  tax              Int
  total            Int
  currency         String        @default("JPY")
  specialRequests  String?       @map("special_requests")
  paymentId        String?       @map("payment_id")
  confirmedAt      DateTime?     @map("confirmed_at")
  cancelledAt      DateTime?     @map("cancelled_at")
  createdAt        DateTime      @default(now()) @map("created_at")
  updatedAt        DateTime      @updatedAt @map("updated_at")

  user  User          @relation(fields: [userId], references: [id])
  hotel Hotel         @relation(fields: [hotelId], references: [id])
  items BookingItem[]

  @@index([userId])
  @@index([hotelId])
  @@index([status])
  @@index([createdAt])
  @@map("bookings")
}

enum BookingStatus {
  PENDING
  CONFIRMED
  CANCELLED
  COMPLETED
  REFUNDED
}

model BookingItem {
  id            String   @id @default(uuid())
  bookingId     String   @map("booking_id")
  roomTypeId    String   @map("room_type_id")
  roomTypeName  String   @map("room_type_name")
  checkIn       DateTime @db.Date @map("check_in")
  checkOut      DateTime @db.Date @map("check_out")
  guests        Int
  pricePerNight Int      @map("price_per_night")
  currency      String   @default("JPY")
  createdAt     DateTime @default(now()) @map("created_at")

  booking  Booking  @relation(fields: [bookingId], references: [id], onDelete: Cascade)
  roomType RoomType @relation(fields: [roomTypeId], references: [id])

  @@index([bookingId])
  @@map("booking_items")
}

// ==========================================
// 商品・注文管理
// ==========================================

model Product {
  id          String   @id @default(uuid())
  name        String
  description String
  price       Int
  currency    String   @default("JPY")
  category    String
  stock       Int      @default(0)
  isActive    Boolean  @default(true) @map("is_active")
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  images     ProductImage[]
  orderItems OrderItem[]

  @@index([category])
  @@index([isActive])
  @@map("products")
}

model ProductImage {
  id        String   @id @default(uuid())
  productId String   @map("product_id")
  url       String
  alt       String?
  isPrimary Boolean  @default(false) @map("is_primary")
  sortOrder Int      @default(0) @map("sort_order")
  createdAt DateTime @default(now()) @map("created_at")

  product Product @relation(fields: [productId], references: [id], onDelete: Cascade)

  @@map("product_images")
}

model Order {
  id              String      @id @default(uuid())
  userId          String      @map("user_id")
  status          OrderStatus @default(PENDING)
  subtotal        Int
  tax             Int
  shippingFee     Int         @default(0) @map("shipping_fee")
  total           Int
  currency        String      @default("JPY")
  shippingAddress Json?       @map("shipping_address")
  paymentId       String?     @map("payment_id")
  paidAt          DateTime?   @map("paid_at")
  shippedAt       DateTime?   @map("shipped_at")
  deliveredAt     DateTime?   @map("delivered_at")
  cancelledAt     DateTime?   @map("cancelled_at")
  createdAt       DateTime    @default(now()) @map("created_at")
  updatedAt       DateTime    @updatedAt @map("updated_at")

  user  User        @relation(fields: [userId], references: [id])
  items OrderItem[]

  @@index([userId])
  @@index([status])
  @@map("orders")
}

enum OrderStatus {
  PENDING
  PAID
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
  REFUNDED
}

model OrderItem {
  id          String   @id @default(uuid())
  orderId     String   @map("order_id")
  productId   String   @map("product_id")
  productName String   @map("product_name")
  quantity    Int
  unitPrice   Int      @map("unit_price")
  createdAt   DateTime @default(now()) @map("created_at")

  order   Order   @relation(fields: [orderId], references: [id], onDelete: Cascade)
  product Product @relation(fields: [productId], references: [id])

  @@map("order_items")
}

// ==========================================
// レビュー・お気に入り
// ==========================================

model Review {
  id        String   @id @default(uuid())
  userId    String   @map("user_id")
  hotelId   String   @map("hotel_id")
  rating    Int
  title     String?
  comment   String
  isPublic  Boolean  @default(true) @map("is_public")
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  user  User  @relation(fields: [userId], references: [id])
  hotel Hotel @relation(fields: [hotelId], references: [id])

  @@unique([userId, hotelId])
  @@index([hotelId])
  @@map("reviews")
}

model Favorite {
  id        String   @id @default(uuid())
  userId    String   @map("user_id")
  hotelId   String   @map("hotel_id")
  createdAt DateTime @default(now()) @map("created_at")

  user  User  @relation(fields: [userId], references: [id], onDelete: Cascade)
  hotel Hotel @relation(fields: [hotelId], references: [id], onDelete: Cascade)

  @@unique([userId, hotelId])
  @@map("favorites")
}
```

### 4.2 トランザクション管理

```typescript
// ==========================================
// src/infrastructure/database/prisma-client.ts
// ==========================================

import { PrismaClient } from '@prisma/client';
import { logger } from '../logging/logger';

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: [
      { emit: 'event', level: 'query' },
      { emit: 'event', level: 'error' },
      { emit: 'event', level: 'warn' },
    ],
  });

// クエリログ
prisma.$on('query', (e) => {
  if (process.env.NODE_ENV === 'development') {
    logger.debug('Query', {
      query: e.query,
      params: e.params,
      duration: `${e.duration}ms`,
    });
  }
});

// エラーログ
prisma.$on('error', (e) => {
  logger.error('Prisma Error', { message: e.message });
});

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}

// ==========================================
// src/infrastructure/database/unit-of-work.ts
// ==========================================

import { PrismaClient, Prisma } from '@prisma/client';
import { prisma } from './prisma-client';

export type TransactionClient = Omit<
  PrismaClient,
  '$connect' | '$disconnect' | '$on' | '$transaction' | '$use' | '$extends'
>;

export interface UnitOfWork {
  execute<T>(
    work: (tx: TransactionClient) => Promise<T>,
    options?: Prisma.TransactionOptions
  ): Promise<T>;
}

export class PrismaUnitOfWork implements UnitOfWork {
  constructor(private readonly client: PrismaClient = prisma) {}

  async execute<T>(
    work: (tx: TransactionClient) => Promise<T>,
    options?: Prisma.TransactionOptions
  ): Promise<T> {
    return this.client.$transaction(work, {
      maxWait: options?.maxWait ?? 5000,
      timeout: options?.timeout ?? 10000,
      isolationLevel: options?.isolationLevel ?? Prisma.TransactionIsolationLevel.ReadCommitted,
    });
  }
}

// 使用例
// src/application/use-cases/booking/create-booking.use-case.ts

import { UnitOfWork } from '../../../infrastructure/database/unit-of-work';
import { BookingRepository } from '../../../domain/repositories/booking.repository';
import { InventoryService } from '../../../domain/services/inventory.service';
import { Booking } from '../../../domain/aggregates/booking.aggregate';

export class CreateBookingUseCase {
  constructor(
    private readonly unitOfWork: UnitOfWork,
    private readonly bookingRepository: BookingRepository,
    private readonly inventoryService: InventoryService
  ) {}

  async execute(input: CreateBookingInput): Promise<Booking> {
    return this.unitOfWork.execute(async (tx) => {
      // 1. 在庫確認と確保
      const available = await this.inventoryService.checkAndHoldInventory(
        input.roomTypeId,
        input.checkIn,
        input.checkOut,
        { transaction: tx }
      );

      if (!available) {
        throw new Error('Room not available for selected dates');
      }

      // 2. 予約作成
      const booking = Booking.create({
        userId: input.userId,
        hotelId: input.hotelId,
        hotelName: input.hotelName,
        guestInfo: input.guestInfo,
        items: [/* ... */],
        specialRequests: input.specialRequests,
      });

      // 3. 保存
      await this.bookingRepository.save(booking);

      return booking;
    });
  }
}
```

### 4.3 クエリ最適化

```typescript
// ==========================================
// src/infrastructure/database/repositories/hotel.repository.impl.ts
// ==========================================

import { PrismaClient, Prisma } from '@prisma/client';
import { Hotel } from '../../../domain/entities/hotel.entity';
import { HotelRepository, HotelSearchCriteria, HotelSearchResult } from '../../../domain/repositories/hotel.repository';
import { CacheService } from '../../cache/cache.service';
import { logger } from '../../logging/logger';

export class HotelRepositoryImpl implements HotelRepository {
  constructor(
    private readonly prisma: PrismaClient,
    private readonly cache: CacheService
  ) {}

  async findById(id: string): Promise<Hotel | null> {
    // キャッシュチェック
    const cacheKey = `hotel:${id}`;
    const cached = await this.cache.get<Hotel>(cacheKey);
    if (cached) {
      return cached;
    }

    const data = await this.prisma.hotel.findUnique({
      where: { id },
      include: {
        images: {
          orderBy: { sortOrder: 'asc' },
        },
        amenities: true,
        roomTypes: {
          where: { isActive: true },
          include: {
            images: { orderBy: { sortOrder: 'asc' } },
            amenities: true,
          },
        },
      },
    });

    if (!data) return null;

    const hotel = this.toDomain(data);

    // キャッシュに保存（5分）
    await this.cache.set(cacheKey, hotel, 300);

    return hotel;
  }

  async search(criteria: HotelSearchCriteria): Promise<HotelSearchResult> {
    const {
      location,
      checkIn,
      checkOut,
      guests,
      minPrice,
      maxPrice,
      amenities,
      rating,
      sortBy,
      page = 1,
      limit = 20,
    } = criteria;

    // 動的WHERE条件の構築
    const where: Prisma.HotelWhereInput = {
      isActive: true,
    };

    // 場所検索
    if (location) {
      where.OR = [
        { city: { contains: location, mode: 'insensitive' } },
        { prefecture: { contains: location, mode: 'insensitive' } },
        { name: { contains: location, mode: 'insensitive' } },
      ];
    }

    // 評価フィルター
    if (rating) {
      where.rating = { gte: rating };
    }

    // アメニティフィルター
    if (amenities?.length) {
      where.amenities = {
        some: {
          name: { in: amenities },
        },
      };
    }

    // 価格フィルター（部屋タイプの価格でフィルター）
    if (minPrice !== undefined || maxPrice !== undefined) {
      where.roomTypes = {
        some: {
          isActive: true,
          basePrice: {
            ...(minPrice !== undefined && { gte: minPrice }),
            ...(maxPrice !== undefined && { lte: maxPrice }),
          },
        },
      };
    }

    // ソート条件
    const orderBy: Prisma.HotelOrderByWithRelationInput = this.getOrderBy(sortBy);

    // カウントとデータ取得を並列実行
    const [totalCount, hotels] = await Promise.all([
      this.prisma.hotel.count({ where }),
      this.prisma.hotel.findMany({
        where,
        include: {
          images: {
            where: { isPrimary: true },
            take: 1,
          },
          amenities: {
            take: 5,
          },
          roomTypes: {
            where: { isActive: true },
            select: {
              basePrice: true,
              currency: true,
            },
            orderBy: { basePrice: 'asc' },
            take: 1,
          },
        },
        orderBy,
        skip: (page - 1) * limit,
        take: limit,
      }),
    ]);

    return {
      hotels: hotels.map(h => this.toSimpleDomain(h)),
      totalCount,
      page,
      limit,
    };
  }

  private getOrderBy(sortBy?: string): Prisma.HotelOrderByWithRelationInput {
    switch (sortBy) {
      case 'price_asc':
        return { roomTypes: { _min: { basePrice: 'asc' } } };
      case 'price_desc':
        return { roomTypes: { _min: { basePrice: 'desc' } } };
      case 'rating':
        return { rating: 'desc' };
      default:
        return { reviewCount: 'desc' }; // relevance
    }
  }

  // 詳細用のドメイン変換
  private toDomain(data: any): Hotel {
    return {
      id: data.id,
      name: data.name,
      description: data.description,
      location: {
        latitude: data.latitude,
        longitude: data.longitude,
        address: data.address,
        city: data.city,
        prefecture: data.prefecture,
        postalCode: data.postalCode,
      },
      rating: data.rating,
      reviewCount: data.reviewCount,
      starRating: data.starRating,
      checkInTime: data.checkInTime,
      checkOutTime: data.checkOutTime,
      petsAllowed: data.petsAllowed,
      images: data.images.map((img: any) => ({
        url: img.url,
        alt: img.alt,
        isPrimary: img.isPrimary,
      })),
      amenities: data.amenities.map((a: any) => a.name),
      roomTypes: data.roomTypes?.map((rt: any) => ({
        id: rt.id,
        name: rt.name,
        description: rt.description,
        maxGuests: rt.maxGuests,
        basePrice: rt.basePrice,
        currency: rt.currency,
        images: rt.images?.map((img: any) => ({
          url: img.url,
          alt: img.alt,
        })),
        amenities: rt.amenities?.map((a: any) => a.name),
      })),
      createdAt: data.createdAt,
      updatedAt: data.updatedAt,
    };
  }

  // 一覧用の簡易ドメイン変換
  private toSimpleDomain(data: any): Hotel {
    const minPrice = data.roomTypes[0]?.basePrice ?? 0;
    const currency = data.roomTypes[0]?.currency ?? 'JPY';

    return {
      id: data.id,
      name: data.name,
      description: data.description.slice(0, 200),
      location: {
        latitude: data.latitude,
        longitude: data.longitude,
        address: data.address,
        city: data.city,
        prefecture: data.prefecture,
        postalCode: data.postalCode,
      },
      rating: data.rating,
      reviewCount: data.reviewCount,
      priceRange: {
        min: minPrice,
        max: minPrice, // 一覧では最低価格のみ表示
        currency,
      },
      images: data.images.map((img: any) => ({
        url: img.url,
        alt: img.alt,
        isPrimary: img.isPrimary,
      })),
      amenities: data.amenities.map((a: any) => a.name),
      isAvailable: true,
      createdAt: data.createdAt,
      updatedAt: data.updatedAt,
    };
  }
}
```

---

## 第5章：横断的関心事

### 5.1 認証・認可フロー

```typescript
// ==========================================
// src/api/middlewares/auth.middleware.ts
// ==========================================

import { Context, Next } from 'hono';
import { verify } from 'hono/jwt';
import { createMiddleware } from 'hono/factory';
import { AuthConfig } from '../../config/auth.config';
import { UnauthorizedError, ForbiddenError } from '../../shared/errors';
import { prisma } from '../../infrastructure/database/prisma-client';
import { logger } from '../../infrastructure/logging/logger';

export interface AuthUser {
  id: string;
  email: string;
  roles: string[];
  permissions: string[];
}

declare module 'hono' {
  interface ContextVariableMap {
    user: AuthUser;
    token: string;
  }
}

// 認証ミドルウェア
export const authMiddleware = createMiddleware(async (c: Context, next: Next) => {
  const authHeader = c.req.header('Authorization');

  if (!authHeader?.startsWith('Bearer ')) {
    throw new UnauthorizedError('Missing or invalid authorization header');
  }

  const token = authHeader.substring(7);

  try {
    // JWT検証
    const payload = await verify(token, AuthConfig.jwt.secret, AuthConfig.jwt.algorithm);

    // ユーザー情報の取得
    const user = await prisma.user.findUnique({
      where: { id: payload.sub as string },
      select: {
        id: true,
        email: true,
      },
    });

    if (!user) {
      throw new UnauthorizedError('User not found');
    }

    // コンテキストにユーザー情報を設定
    c.set('user', {
      id: user.id,
      email: user.email,
      roles: (payload.roles as string[]) || ['user'],
      permissions: (payload.permissions as string[]) || [],
    });
    c.set('token', token);

    await next();
  } catch (error) {
    logger.warn('Authentication failed', { error });
    throw new UnauthorizedError('Invalid or expired token');
  }
});

// オプショナル認証ミドルウェア（認証なしでも通過可能）
export const optionalAuthMiddleware = createMiddleware(async (c: Context, next: Next) => {
  const authHeader = c.req.header('Authorization');

  if (authHeader?.startsWith('Bearer ')) {
    const token = authHeader.substring(7);

    try {
      const payload = await verify(token, AuthConfig.jwt.secret, AuthConfig.jwt.algorithm);
      const user = await prisma.user.findUnique({
        where: { id: payload.sub as string },
        select: { id: true, email: true },
      });

      if (user) {
        c.set('user', {
          id: user.id,
          email: user.email,
          roles: (payload.roles as string[]) || ['user'],
          permissions: (payload.permissions as string[]) || [],
        });
        c.set('token', token);
      }
    } catch {
      // 認証失敗は無視（オプショナルなので）
    }
  }

  await next();
});

// ロールベース認可ミドルウェア
export const requireRoles = (...requiredRoles: string[]) => {
  return createMiddleware(async (c: Context, next: Next) => {
    const user = c.get('user');

    if (!user) {
      throw new UnauthorizedError('Authentication required');
    }

    const hasRole = requiredRoles.some(role => user.roles.includes(role));

    if (!hasRole) {
      throw new ForbiddenError('Insufficient permissions');
    }

    await next();
  });
};

// パーミッションベース認可ミドルウェア
export const requirePermissions = (...requiredPermissions: string[]) => {
  return createMiddleware(async (c: Context, next: Next) => {
    const user = c.get('user');

    if (!user) {
      throw new UnauthorizedError('Authentication required');
    }

    const hasPermission = requiredPermissions.every(
      permission => user.permissions.includes(permission)
    );

    if (!hasPermission) {
      throw new ForbiddenError('Insufficient permissions');
    }

    await next();
  });
};

// ==========================================
// src/application/services/auth.service.ts
// ==========================================

import { sign, verify } from 'hono/jwt';
import { AuthConfig } from '../../config/auth.config';
import { User } from '../../domain/entities/user.entity';
import { UserRepository } from '../../domain/repositories/user.repository';
import { Password } from '../../domain/value-objects/password.vo';
import { Email } from '../../domain/value-objects/email.vo';
import { UnauthorizedError, ValidationError } from '../../shared/errors';
import { CacheService } from '../../infrastructure/cache/cache.service';
import { logger } from '../../infrastructure/logging/logger';

export interface TokenPair {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
}

export interface LoginResult {
  user: User;
  tokens: TokenPair;
}

export class AuthService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly cache: CacheService
  ) {}

  async login(email: string, password: string): Promise<LoginResult> {
    // ユーザー検索
    const user = await this.userRepository.findByEmail(Email.create(email));

    if (!user) {
      throw new UnauthorizedError('Invalid email or password');
    }

    // パスワード検証
    if (!user.verifyPassword(password)) {
      logger.warn('Login failed: invalid password', { email });
      throw new UnauthorizedError('Invalid email or password');
    }

    // トークン生成
    const tokens = await this.generateTokens(user);

    logger.info('User logged in', { userId: user.id, email });

    return { user, tokens };
  }

  async register(data: {
    email: string;
    password: string;
    name: string;
    language?: string;
    currency?: string;
  }): Promise<LoginResult> {
    // メール重複チェック
    const existing = await this.userRepository.findByEmail(Email.create(data.email));
    if (existing) {
      throw new ValidationError('Email already registered');
    }

    // パスワードハッシュ化
    const passwordHash = await Password.create(data.password);

    // ユーザー作成
    const user = User.create({
      email: Email.create(data.email),
      password: passwordHash,
      name: data.name,
      language: data.language || 'ja',
      currency: data.currency || 'JPY',
    });

    await this.userRepository.save(user);

    // トークン生成
    const tokens = await this.generateTokens(user);

    logger.info('User registered', { userId: user.id, email: data.email });

    return { user, tokens };
  }

  async refreshTokens(refreshToken: string): Promise<TokenPair> {
    try {
      // リフレッシュトークン検証
      const payload = await verify(
        refreshToken,
        AuthConfig.jwt.refreshSecret,
        AuthConfig.jwt.algorithm
      );

      // トークンがブラックリストに入っていないか確認
      const isBlacklisted = await this.cache.get(`blacklist:${refreshToken}`);
      if (isBlacklisted) {
        throw new UnauthorizedError('Token has been revoked');
      }

      // ユーザー取得
      const user = await this.userRepository.findById(payload.sub as string);
      if (!user) {
        throw new UnauthorizedError('User not found');
      }

      // 古いリフレッシュトークンをブラックリストに追加
      await this.cache.set(
        `blacklist:${refreshToken}`,
        true,
        AuthConfig.jwt.refreshExpiresIn
      );

      // 新しいトークン生成
      return this.generateTokens(user);
    } catch (error) {
      logger.warn('Token refresh failed', { error });
      throw new UnauthorizedError('Invalid or expired refresh token');
    }
  }

  async logout(accessToken: string, refreshToken: string): Promise<void> {
    // 両トークンをブラックリストに追加
    await Promise.all([
      this.cache.set(`blacklist:${accessToken}`, true, AuthConfig.jwt.expiresIn),
      this.cache.set(`blacklist:${refreshToken}`, true, AuthConfig.jwt.refreshExpiresIn),
    ]);

    logger.info('User logged out');
  }

  private async generateTokens(user: User): Promise<TokenPair> {
    const now = Math.floor(Date.now() / 1000);

    // アクセストークン
    const accessToken = await sign(
      {
        sub: user.id,
        email: user.email.value,
        roles: ['user'],
        permissions: [],
        iat: now,
        exp: now + AuthConfig.jwt.expiresIn,
      },
      AuthConfig.jwt.secret,
      AuthConfig.jwt.algorithm
    );

    // リフレッシュトークン
    const refreshToken = await sign(
      {
        sub: user.id,
        type: 'refresh',
        iat: now,
        exp: now + AuthConfig.jwt.refreshExpiresIn,
      },
      AuthConfig.jwt.refreshSecret,
      AuthConfig.jwt.algorithm
    );

    return {
      accessToken,
      refreshToken,
      expiresIn: AuthConfig.jwt.expiresIn,
    };
  }
}
```

### 5.2 ロギングとモニタリング

```typescript
// ==========================================
// src/infrastructure/logging/logger.ts
// ==========================================

import pino from 'pino';
import { AppConfig } from '../../config/app.config';

export const logger = pino({
  level: AppConfig.logLevel,
  transport:
    process.env.NODE_ENV === 'development'
      ? {
          target: 'pino-pretty',
          options: {
            colorize: true,
            translateTime: 'HH:MM:ss Z',
            ignore: 'pid,hostname',
          },
        }
      : undefined,
  base: {
    service: 'triptrip-api',
    version: process.env.npm_package_version,
    environment: process.env.NODE_ENV,
  },
  formatters: {
    level: (label) => ({ level: label }),
  },
  redact: {
    paths: ['password', 'token', 'authorization', 'cookie'],
    censor: '[REDACTED]',
  },
});

// 子ロガー作成ヘルパー
export function createLogger(context: string) {
  return logger.child({ context });
}

// ==========================================
// src/api/middlewares/logging.middleware.ts
// ==========================================

import { Context, Next } from 'hono';
import { createMiddleware } from 'hono/factory';
import { randomUUID } from 'crypto';
import { logger } from '../../infrastructure/logging/logger';

export const loggingMiddleware = createMiddleware(async (c: Context, next: Next) => {
  const requestId = c.req.header('X-Request-ID') || randomUUID();
  const startTime = Date.now();

  // リクエストIDをヘッダーに設定
  c.res.headers.set('X-Request-ID', requestId);

  // リクエストログ
  const requestLog = {
    requestId,
    method: c.req.method,
    path: c.req.path,
    query: c.req.query(),
    userAgent: c.req.header('User-Agent'),
    ip: c.req.header('X-Forwarded-For') || c.req.header('X-Real-IP'),
  };

  logger.info('Incoming request', requestLog);

  try {
    await next();

    // レスポンスログ
    const duration = Date.now() - startTime;
    logger.info('Request completed', {
      ...requestLog,
      status: c.res.status,
      duration: `${duration}ms`,
    });
  } catch (error) {
    const duration = Date.now() - startTime;
    logger.error('Request failed', {
      ...requestLog,
      error: error instanceof Error ? error.message : 'Unknown error',
      stack: error instanceof Error ? error.stack : undefined,
      duration: `${duration}ms`,
    });
    throw error;
  }
});
```

### 5.3 エラーハンドリング

```typescript
// ==========================================
// src/shared/errors/base.error.ts
// ==========================================

export abstract class BaseError extends Error {
  abstract readonly statusCode: number;
  abstract readonly code: string;
  readonly isOperational: boolean = true;
  readonly timestamp: Date;

  constructor(message: string) {
    super(message);
    this.name = this.constructor.name;
    this.timestamp = new Date();
    Error.captureStackTrace(this, this.constructor);
  }

  toJSON() {
    return {
      code: this.code,
      message: this.message,
      timestamp: this.timestamp.toISOString(),
    };
  }
}

// ==========================================
// src/shared/errors/index.ts
// ==========================================

import { BaseError } from './base.error';

export class ValidationError extends BaseError {
  readonly statusCode = 400;
  readonly code = 'VALIDATION_ERROR';
  readonly errors?: Record<string, string>;

  constructor(message: string, errors?: Record<string, string>) {
    super(message);
    this.errors = errors;
  }

  toJSON() {
    return {
      ...super.toJSON(),
      errors: this.errors,
    };
  }
}

export class UnauthorizedError extends BaseError {
  readonly statusCode = 401;
  readonly code = 'UNAUTHORIZED';
}

export class ForbiddenError extends BaseError {
  readonly statusCode = 403;
  readonly code = 'FORBIDDEN';
}

export class NotFoundError extends BaseError {
  readonly statusCode = 404;
  readonly code = 'NOT_FOUND';
}

export class ConflictError extends BaseError {
  readonly statusCode = 409;
  readonly code = 'CONFLICT';
}

export class TooManyRequestsError extends BaseError {
  readonly statusCode = 429;
  readonly code = 'TOO_MANY_REQUESTS';
  readonly retryAfter?: number;

  constructor(message: string, retryAfter?: number) {
    super(message);
    this.retryAfter = retryAfter;
  }
}

export class InternalError extends BaseError {
  readonly statusCode = 500;
  readonly code = 'INTERNAL_ERROR';
  readonly isOperational = false;
}

// ==========================================
// src/api/middlewares/error.middleware.ts
// ==========================================

import { Context } from 'hono';
import { HTTPException } from 'hono/http-exception';
import { ZodError } from 'zod';
import { BaseError, ValidationError, InternalError } from '../../shared/errors';
import { logger } from '../../infrastructure/logging/logger';

export function errorMiddleware(err: Error, c: Context) {
  const requestId = c.res.headers.get('X-Request-ID');

  // Zodバリデーションエラー
  if (err instanceof ZodError) {
    const errors: Record<string, string> = {};
    err.errors.forEach((e) => {
      const path = e.path.join('.');
      errors[path] = e.message;
    });

    return c.json({
      success: false,
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Validation failed',
        errors,
        requestId,
      },
    }, 400);
  }

  // カスタムエラー
  if (err instanceof BaseError) {
    // 運用エラーはwarn、システムエラーはerror
    if (err.isOperational) {
      logger.warn('Operational error', {
        code: err.code,
        message: err.message,
        requestId,
      });
    } else {
      logger.error('System error', {
        code: err.code,
        message: err.message,
        stack: err.stack,
        requestId,
      });
    }

    return c.json({
      success: false,
      error: {
        ...err.toJSON(),
        requestId,
      },
    }, err.statusCode);
  }

  // Hono HTTPException
  if (err instanceof HTTPException) {
    return c.json({
      success: false,
      error: {
        code: 'HTTP_ERROR',
        message: err.message,
        requestId,
      },
    }, err.status);
  }

  // 予期しないエラー
  logger.error('Unexpected error', {
    error: err.message,
    stack: err.stack,
    requestId,
  });

  return c.json({
    success: false,
    error: {
      code: 'INTERNAL_ERROR',
      message: process.env.NODE_ENV === 'production'
        ? 'An unexpected error occurred'
        : err.message,
      requestId,
    },
  }, 500);
}
```

---

## 第6章：実装ロードマップ & 文書間参照

### 6.1 実装フェーズ

```yaml
implementation_phases:
  phase_1:
    name: アーキテクチャ基盤
    duration: 3週間
    deliverables:
      - プロジェクト構造の確立
      - DIコンテナの設定
      - ロギング・エラーハンドリング基盤
      - 認証ミドルウェア実装
      - OpenAPI設定
    dependencies: []

  phase_2:
    name: ドメイン層実装
    duration: 4週間
    deliverables:
      - エンティティ・値オブジェクト設計
      - アグリゲート実装（Booking、Order）
      - リポジトリインターフェース定義
      - ドメインサービス実装
      - ドメインイベント設計
    dependencies: [phase_1]

  phase_3:
    name: インフラ層実装
    duration: 4週間
    deliverables:
      - Prismaスキーマ完成
      - リポジトリ実装
      - キャッシュサービス（Redis）
      - バックグラウンドジョブ（BullMQ）
    dependencies: [phase_2]

  phase_4:
    name: API層実装
    duration: 4週間
    deliverables:
      - 全ルート実装
      - バリデーションスキーマ
      - 認証・認可統合
      - レート制限実装
    dependencies: [phase_3]

  phase_5:
    name: 外部統合
    duration: 3週間
    deliverables:
      - Stripe決済統合
      - SendGridメール統合
      - Google Maps統合
      - FCMプッシュ通知統合
    dependencies: [phase_4]

  phase_6:
    name: テスト・品質保証
    duration: 3週間
    deliverables:
      - ユニットテスト（80%カバレッジ）
      - 統合テスト
      - E2Eテスト
      - パフォーマンステスト
    dependencies: [phase_5]
```

### 6.2 文書間参照

```yaml
document_references:
  related_documents:
    - document: Doc-TV-001
      relationship: 技術ビジョンと原則の準拠
      sections:
        - アーキテクチャ原則
        - パフォーマンス目標
        - セキュリティ要件

    - document: Doc-SA-001
      relationship: システムアーキテクチャとの整合性
      sections:
        - サービス境界
        - データモデル
        - API設計原則

    - document: Doc-SA-002
      relationship: マイクロサービス設計との整合性
      sections:
        - ドメイン駆動設計
        - イベント駆動アーキテクチャ
        - サービス間通信

    - document: Doc-AD-001
      relationship: モバイルアプリとのAPI契約
      sections:
        - API仕様
        - 認証フロー
        - エラーレスポンス

    - document: Doc-AD-002
      relationship: WebアプリとのAPI契約
      sections:
        - API仕様
        - 認証フロー
        - キャッシュ戦略

  successor_documents:
    - Doc-DA-001: データアーキテクチャ詳細
    - Doc-SEC-003: バックエンドセキュリティ実装
    - Doc-QA-003: バックエンドテスト戦略
```

### 6.3 成功指標

```yaml
success_metrics:
  performance:
    latency:
      p50: "< 50ms"
      p95: "< 100ms"
      p99: "< 200ms"

    throughput: "> 1,000 req/s per instance"

    availability: "> 99.9%"

  code_quality:
    test_coverage: "> 80%"
    type_coverage: "100%"
    lint_errors: "0"

  security:
    vulnerability_scan: "No high/critical issues"
    dependency_audit: "No known vulnerabilities"

  development:
    build_time: "< 1 minute"
    deployment_frequency: ">= 1/day"
```

---

## 結論

本文書は、TripTripバックエンドサービスの包括的なアーキテクチャ設計を定義しました。Node.js/Honoを基盤とした高性能なAPIサーバー、ドメイン駆動設計による堅牢なビジネスロジック、Prisma ORMによる型安全なデータアクセスを実現します。

主要な成果：
1. **Clean Architecture**: 保守性、テスト容易性、スケーラビリティを確保
2. **DDD実装**: ビジネスロジックの明確な構造化とドメインイベント
3. **型安全API**: OpenAPI + Zodによる完全な型安全性
4. **認証・認可**: JWT/OAuth2.0によるセキュアな認証システム
5. **可観測性**: 構造化ログ、エラーハンドリング、モニタリング統合

この設計により、TripTripは数百万ユーザーに対応可能な、高品質でスケーラブルなバックエンドサービスとして成長できます。

---

**文書情報**
- Document ID: Doc-AD-003
- Version: 1.0.0
- Last Updated: 2026-01-20
- Status: Draft
- Total Lines: 1,500
- Related Documents: Doc-TV-001, Doc-SA-001, Doc-SA-002, Doc-AD-001, Doc-AD-002

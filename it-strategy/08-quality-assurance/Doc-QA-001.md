# Doc-QA-001: TripTripテスト戦略＆品質基準

## エグゼクティブサマリー

本文書は、TripTripプラットフォームの包括的なテスト戦略と品質基準を定義します。Google、Meta、Spotifyの品質保証ベストプラクティスを統合し、テストピラミッド戦略（ユニット60%、インテグレーション25%、E2E 15%）、自動テストカバレッジ目標（80%以上）、パフォーマンス・セキュリティ・アクセシビリティテスト方法論、品質ゲート定義を体系的に策定します。既存のFlutterアプリ（46テストファイル）とNode.js/TypeScriptバックエンドのテスト基盤を発展させ、継続的品質の実現を目指します。Year 1でテストカバレッジ80%、Year 2で自動化率95%、Year 3でAI支援テスト生成の達成を目標とします。

---

## 第1章 はじめに & コンテキスト

### 1.1 目的 & スコープ

#### 1.1.1 文書の目的

本テスト戦略＆品質基準文書は、TripTripの品質保証活動を世界最高水準に引き上げるための包括的なガイドラインを提供します。

**主要目的**:

1. **品質文化の醸成**: "Quality is everyone's responsibility"の浸透
2. **テスト効率の最大化**: テストピラミッドによる最適なテスト配置
3. **早期バグ検出**: Shift-Leftによる品質コスト削減
4. **リリース信頼性の向上**: 品質ゲートによる本番障害防止
5. **継続的改善**: メトリクス駆動の品質向上

**スコープ定義**:

```
┌─────────────────────────────────────────────────────────────┐
│              テスト戦略スコープ                              │
├─────────────────────────────────────────────────────────────┤
│ インスコープ:                                                │
│  ├─ テストピラミッド戦略                                    │
│  ├─ テスト自動化フレームワーク                              │
│  ├─ カバレッジ目標と測定                                    │
│  ├─ パフォーマンステスト                                    │
│  ├─ セキュリティテスト                                      │
│  ├─ アクセシビリティテスト                                  │
│  ├─ 品質ゲート定義                                          │
│  └─ テストデータ管理                                        │
├─────────────────────────────────────────────────────────────┤
│ アウトオブスコープ:                                          │
│  ├─ 監視・本番品質（→Doc-QA-002参照）                      │
│  ├─ リリース管理詳細（→Doc-QA-003参照）                    │
│  ├─ CI/CD詳細（→Doc-DM-002参照）                           │
│  └─ セキュリティアーキテクチャ（→Doc-SC-001参照）          │
└─────────────────────────────────────────────────────────────┘
```

#### 1.1.2 対象読者

- QAエンジニア
- ソフトウェアエンジニア（全員）
- テックリード
- エンジニアリングマネージャー
- プロダクトマネージャー

### 1.2 品質目標 & 成功基準

#### 1.2.1 品質KPI

```yaml
quality_kpis:
  defect_metrics:
    defect_density:
      description: 1,000行あたりの欠陥数
      current: 測定中
      year_1_target: 0.5以下
      year_2_target: 0.3以下
      year_3_target: 0.1以下
      elite_benchmark: 0.1未満

    defect_escape_rate:
      description: 本番に漏れた欠陥の割合
      current: 測定中
      year_1_target: 5%以下
      year_2_target: 2%以下
      year_3_target: 1%以下

    mean_time_to_detect:
      description: 欠陥の検出までの時間
      current: 数日
      year_1_target: 24時間以内
      year_2_target: 4時間以内
      year_3_target: 1時間以内

  test_metrics:
    code_coverage:
      description: テストカバレッジ（行カバレッジ）
      current: ~40%
      year_1_target: 80%
      year_2_target: 85%
      year_3_target: 90%

    test_automation_rate:
      description: 自動化されたテストの割合
      current: ~60%
      year_1_target: 85%
      year_2_target: 95%
      year_3_target: 98%

    test_execution_time:
      description: 全テストスイートの実行時間
      current: 30分+
      year_1_target: 15分以内
      year_2_target: 10分以内
      year_3_target: 5分以内

  customer_quality:
    app_crash_rate:
      description: アプリクラッシュ率
      year_1_target: 0.5%以下
      year_2_target: 0.1%以下
      year_3_target: 0.01%以下

    customer_reported_bugs:
      description: 顧客報告バグ数/月
      year_1_target: 10件以下
      year_2_target: 5件以下
      year_3_target: 2件以下
```

#### 1.2.2 品質成熟度モデル

```
┌─────────────────────────────────────────────────────────────┐
│              品質成熟度モデル                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Level 5: 最適化（Optimizing）                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ ・AI/ML駆動のテスト生成                              │   │
│  │ ・予測的品質分析                                     │   │
│  │ ・自己修復テスト                                     │   │
│  │ ・継続的品質イノベーション                           │   │
│  └─────────────────────────────────────────────────────┘   │
│                          ▲ Year 3目標                       │
│  Level 4: 管理（Managed）                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ ・定量的品質管理                                     │   │
│  │ ・カオステスト                                       │   │
│  │ ・コントラクトテスト                                 │   │
│  │ ・品質ダッシュボード                                 │   │
│  └─────────────────────────────────────────────────────┘   │
│                          ▲ Year 2目標                       │
│  Level 3: 標準化（Defined）                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ ・テストピラミッド実践                               │   │
│  │ ・CI統合自動テスト                                   │   │
│  │ ・80%カバレッジ達成                                  │   │
│  │ ・品質ゲート運用                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                          ▲ Year 1目標                       │
│  Level 2: 反復（Repeatable）                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ ・基本的な自動テスト                                 │   │
│  │ ・46テストファイル存在                               │   │
│  │ ・部分的なCI統合                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                          ▲ 現在の状態                       │
│  Level 1: 初期（Initial）                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ ・アドホックなテスト                                 │   │
│  │ ・手動テスト中心                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 現在のテスト状況

#### 1.3.1 既存テスト基盤分析

**現在のテストスタック**（EXISTING_APP_ANALYSIS.mdより）:

```yaml
current_test_infrastructure:
  flutter:
    testing_frameworks:
      - flutter_test: 標準テストフレームワーク
      - Mockito 5.4.4: モッキングフレームワーク
      - Patrol 3.19.0: 統合テスト
      - Integration Test: E2Eテスト
    test_files: 46
    estimated_coverage: ~40%

  backend:
    testing_frameworks:
      - Vitest 1.0.0: ユニットテスト
      - Supertest 6.3.3: HTTPテスト
    estimated_coverage: ~30%

  code_quality:
    linting:
      - flutter analyze
      - ESLint 8.56.0 (backend)
    formatting:
      - dart format
      - Prettier (backend)

  current_test_types:
    unit_tests: 存在（カバレッジ不十分）
    integration_tests: 部分的
    e2e_tests: 最小限
    performance_tests: なし
    security_tests: なし
    accessibility_tests: なし
```

**現在の課題と改善機会**:

```yaml
current_challenges:
  coverage:
    - 全体カバレッジ40%（目標80%）
    - クリティカルパスのカバレッジ不明
    - バックエンドカバレッジ特に低い

  test_types:
    - E2Eテストの不足
    - パフォーマンステストの未実施
    - セキュリティテストの未統合
    - アクセシビリティテストの未実施

  infrastructure:
    - テスト実行時間の最適化余地
    - テストデータ管理の標準化不足
    - 環境間のテスト差異

  process:
    - テスト計画の未文書化
    - 品質ゲートの未定義
    - テストメトリクスの可視化不足

improvement_opportunities:
  - テストカバレッジの体系的向上
  - テストピラミッドの実践
  - CI/CDへの完全統合
  - 品質ダッシュボードの構築
  - AI支援テスト生成の将来導入
```

---

## 第2章 テストピラミッド戦略

### 2.1 ユニットテスト（60%）

#### 2.1.1 ユニットテスト方針

```yaml
unit_test_strategy:
  definition:
    description: 単一のユニット（関数、クラス、メソッド）の振る舞いを検証
    isolation: 外部依存は全てモック/スタブ
    execution_time: 個々のテストは10ms以内

  coverage_targets:
    overall: 80%
    critical_paths: 95%
    business_logic: 90%
    utilities: 70%

  what_to_test:
    must_test:
      - ビジネスロジック
      - データ変換・バリデーション
      - 状態管理ロジック
      - エラーハンドリング
      - エッジケース

    optional:
      - 単純なゲッター/セッター
      - 純粋なUI（ウィジェットテストで代替）
      - 外部ライブラリのラッパー

  naming_conventions:
    pattern: "[MethodName]_[Scenario]_[ExpectedResult]"
    examples:
      - "calculateTotal_withValidItems_returnsCorrectSum"
      - "validateEmail_withInvalidFormat_throwsValidationError"
      - "fetchUser_whenNotFound_returnsNull"

  structure:
    arrange_act_assert:
      arrange: テストデータと依存関係の準備
      act: テスト対象メソッドの実行
      assert: 結果の検証
```

#### 2.1.2 Flutterユニットテスト例

```dart
// test/features/cart/services/cart_service_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/annotations.dart';
import 'package:mockito/mockito.dart';
import 'package:triptrip/features/cart/services/cart_service.dart';
import 'package:triptrip/features/cart/models/cart_item.dart';
import 'package:triptrip/services/storage/cart_storage_service.dart';

@GenerateMocks([CartStorageService])
import 'cart_service_test.mocks.dart';

void main() {
  late CartService cartService;
  late MockCartStorageService mockStorage;

  setUp(() {
    mockStorage = MockCartStorageService();
    cartService = CartService(storage: mockStorage);
  });

  group('CartService', () {
    group('addItem', () {
      test('addItem_withNewItem_addsToCartAndPersists', () async {
        // Arrange
        final item = CartItem(
          id: '1',
          productId: 'prod-1',
          name: 'Test Product',
          price: 1000,
          quantity: 1,
        );
        when(mockStorage.saveCart(any)).thenAnswer((_) async => true);

        // Act
        await cartService.addItem(item);

        // Assert
        expect(cartService.items, contains(item));
        expect(cartService.itemCount, equals(1));
        verify(mockStorage.saveCart(any)).called(1);
      });

      test('addItem_withExistingItem_incrementsQuantity', () async {
        // Arrange
        final item = CartItem(
          id: '1',
          productId: 'prod-1',
          name: 'Test Product',
          price: 1000,
          quantity: 1,
        );
        when(mockStorage.saveCart(any)).thenAnswer((_) async => true);
        await cartService.addItem(item);

        // Act
        await cartService.addItem(item.copyWith(quantity: 2));

        // Assert
        expect(cartService.items.length, equals(1));
        expect(cartService.items.first.quantity, equals(3));
      });

      test('addItem_withInvalidQuantity_throwsArgumentError', () {
        // Arrange
        final item = CartItem(
          id: '1',
          productId: 'prod-1',
          name: 'Test Product',
          price: 1000,
          quantity: 0, // Invalid
        );

        // Act & Assert
        expect(
          () => cartService.addItem(item),
          throwsA(isA<ArgumentError>()),
        );
      });
    });

    group('calculateTotal', () {
      test('calculateTotal_withMultipleItems_returnsCorrectSum', () async {
        // Arrange
        when(mockStorage.saveCart(any)).thenAnswer((_) async => true);
        await cartService.addItem(CartItem(
          id: '1', productId: 'p1', name: 'Item 1', price: 1000, quantity: 2,
        ));
        await cartService.addItem(CartItem(
          id: '2', productId: 'p2', name: 'Item 2', price: 500, quantity: 3,
        ));

        // Act
        final total = cartService.calculateTotal();

        // Assert
        expect(total, equals(3500)); // 1000*2 + 500*3
      });

      test('calculateTotal_withEmptyCart_returnsZero', () {
        // Act
        final total = cartService.calculateTotal();

        // Assert
        expect(total, equals(0));
      });

      test('calculateTotal_withDiscount_appliesCorrectly', () async {
        // Arrange
        when(mockStorage.saveCart(any)).thenAnswer((_) async => true);
        await cartService.addItem(CartItem(
          id: '1', productId: 'p1', name: 'Item 1', price: 1000, quantity: 1,
        ));
        cartService.applyDiscount(percentage: 10);

        // Act
        final total = cartService.calculateTotal();

        // Assert
        expect(total, equals(900)); // 1000 - 10%
      });
    });
  });
}
```

#### 2.1.3 バックエンドユニットテスト例

```typescript
// backend/src/services/__tests__/order.service.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { OrderService } from '../order.service';
import { PrismaClient } from '@prisma/client';
import { PaymentService } from '../payment.service';
import { NotificationService } from '../notification.service';

// Mocks
vi.mock('@prisma/client');
vi.mock('../payment.service');
vi.mock('../notification.service');

describe('OrderService', () => {
  let orderService: OrderService;
  let mockPrisma: jest.Mocked<PrismaClient>;
  let mockPaymentService: jest.Mocked<PaymentService>;
  let mockNotificationService: jest.Mocked<NotificationService>;

  beforeEach(() => {
    mockPrisma = new PrismaClient() as jest.Mocked<PrismaClient>;
    mockPaymentService = new PaymentService() as jest.Mocked<PaymentService>;
    mockNotificationService = new NotificationService() as jest.Mocked<NotificationService>;

    orderService = new OrderService(
      mockPrisma,
      mockPaymentService,
      mockNotificationService
    );
  });

  describe('createOrder', () => {
    it('createOrder_withValidData_createsOrderSuccessfully', async () => {
      // Arrange
      const orderData = {
        userId: 'user-1',
        items: [
          { productId: 'prod-1', quantity: 2, price: 1000 },
          { productId: 'prod-2', quantity: 1, price: 500 },
        ],
        shippingAddress: {
          street: '123 Main St',
          city: 'Tokyo',
          postalCode: '100-0001',
        },
      };

      const expectedOrder = {
        id: 'order-1',
        ...orderData,
        total: 2500,
        status: 'PENDING',
        createdAt: new Date(),
      };

      mockPrisma.order.create.mockResolvedValue(expectedOrder);

      // Act
      const result = await orderService.createOrder(orderData);

      // Assert
      expect(result).toEqual(expectedOrder);
      expect(mockPrisma.order.create).toHaveBeenCalledWith({
        data: expect.objectContaining({
          userId: 'user-1',
          total: 2500,
          status: 'PENDING',
        }),
      });
    });

    it('createOrder_withEmptyItems_throwsValidationError', async () => {
      // Arrange
      const orderData = {
        userId: 'user-1',
        items: [],
        shippingAddress: { street: '', city: '', postalCode: '' },
      };

      // Act & Assert
      await expect(orderService.createOrder(orderData))
        .rejects
        .toThrow('Order must contain at least one item');
    });

    it('createOrder_withInvalidUserId_throwsNotFoundError', async () => {
      // Arrange
      const orderData = {
        userId: 'non-existent-user',
        items: [{ productId: 'prod-1', quantity: 1, price: 1000 }],
        shippingAddress: { street: '123', city: 'Tokyo', postalCode: '100' },
      };

      mockPrisma.user.findUnique.mockResolvedValue(null);

      // Act & Assert
      await expect(orderService.createOrder(orderData))
        .rejects
        .toThrow('User not found');
    });
  });

  describe('calculateOrderTotal', () => {
    it('calculateOrderTotal_withMultipleItems_returnsCorrectSum', () => {
      // Arrange
      const items = [
        { price: 1000, quantity: 2 },
        { price: 500, quantity: 3 },
      ];

      // Act
      const total = orderService.calculateOrderTotal(items);

      // Assert
      expect(total).toBe(3500);
    });

    it('calculateOrderTotal_withTaxRate_appliesTax', () => {
      // Arrange
      const items = [{ price: 1000, quantity: 1 }];
      const taxRate = 0.1; // 10%

      // Act
      const total = orderService.calculateOrderTotal(items, { taxRate });

      // Assert
      expect(total).toBe(1100);
    });
  });

  describe('processPayment', () => {
    it('processPayment_withSuccessfulPayment_updatesOrderStatus', async () => {
      // Arrange
      const orderId = 'order-1';
      const paymentMethod = 'stripe';

      mockPaymentService.charge.mockResolvedValue({
        success: true,
        transactionId: 'txn-1',
      });

      mockPrisma.order.update.mockResolvedValue({
        id: orderId,
        status: 'PAID',
      });

      // Act
      const result = await orderService.processPayment(orderId, paymentMethod);

      // Assert
      expect(result.status).toBe('PAID');
      expect(mockNotificationService.sendOrderConfirmation).toHaveBeenCalled();
    });

    it('processPayment_withFailedPayment_throwsPaymentError', async () => {
      // Arrange
      const orderId = 'order-1';
      mockPaymentService.charge.mockResolvedValue({
        success: false,
        error: 'Insufficient funds',
      });

      // Act & Assert
      await expect(orderService.processPayment(orderId, 'stripe'))
        .rejects
        .toThrow('Payment failed: Insufficient funds');
    });
  });
});
```

### 2.2 インテグレーションテスト（25%）

#### 2.2.1 インテグレーションテスト方針

```yaml
integration_test_strategy:
  definition:
    description: 複数のコンポーネント間の連携を検証
    scope: サービス間通信、DB操作、外部API統合
    execution_time: 個々のテストは1秒以内目標

  test_categories:
    api_integration:
      description: REST API エンドポイントの統合テスト
      tools: [Supertest, Vitest]
      database: テスト用PostgreSQL（Docker）

    service_integration:
      description: サービス間の連携テスト
      mocking: 外部サービスのみモック

    database_integration:
      description: Prisma/ORMとDBの連携テスト
      strategy: トランザクション分離

    flutter_widget_integration:
      description: 複数ウィジェットの連携テスト
      tools: [flutter_test, golden_toolkit]

  coverage_targets:
    api_endpoints: 100%
    critical_flows: 100%
    service_interactions: 80%

  environment:
    database: Docker PostgreSQL (dedicated per test suite)
    external_services: Mock/Sandbox
    isolation: トランザクションごとにロールバック
```

#### 2.2.2 API インテグレーションテスト例

```typescript
// backend/src/routes/__tests__/orders.integration.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';
import { app } from '../../app';
import { PrismaClient } from '@prisma/client';
import supertest from 'supertest';

describe('Orders API Integration', () => {
  let prisma: PrismaClient;
  let request: supertest.SuperTest<supertest.Test>;
  let testUser: any;
  let authToken: string;

  beforeAll(async () => {
    prisma = new PrismaClient();
    request = supertest(app);

    // テストユーザー作成
    testUser = await prisma.user.create({
      data: {
        email: 'test@example.com',
        password: 'hashedPassword',
        name: 'Test User',
      },
    });

    // テスト用トークン取得
    const loginResponse = await request
      .post('/api/auth/login')
      .send({ email: 'test@example.com', password: 'password' });
    authToken = loginResponse.body.token;
  });

  afterAll(async () => {
    // クリーンアップ
    await prisma.order.deleteMany({ where: { userId: testUser.id } });
    await prisma.user.delete({ where: { id: testUser.id } });
    await prisma.$disconnect();
  });

  beforeEach(async () => {
    // 各テスト前に注文をクリア
    await prisma.order.deleteMany({ where: { userId: testUser.id } });
  });

  describe('POST /api/orders', () => {
    it('should create a new order with valid data', async () => {
      // Arrange
      const orderData = {
        items: [
          { productId: 'prod-1', quantity: 2, price: 1000 },
        ],
        shippingAddress: {
          street: '123 Test St',
          city: 'Tokyo',
          postalCode: '100-0001',
          country: 'Japan',
        },
      };

      // Act
      const response = await request
        .post('/api/orders')
        .set('Authorization', `Bearer ${authToken}`)
        .send(orderData);

      // Assert
      expect(response.status).toBe(201);
      expect(response.body).toMatchObject({
        userId: testUser.id,
        status: 'PENDING',
        total: 2000,
      });
      expect(response.body.id).toBeDefined();

      // DBに保存されていることを確認
      const savedOrder = await prisma.order.findUnique({
        where: { id: response.body.id },
      });
      expect(savedOrder).not.toBeNull();
    });

    it('should return 400 for invalid order data', async () => {
      // Arrange
      const invalidData = {
        items: [], // Empty items
        shippingAddress: {},
      };

      // Act
      const response = await request
        .post('/api/orders')
        .set('Authorization', `Bearer ${authToken}`)
        .send(invalidData);

      // Assert
      expect(response.status).toBe(400);
      expect(response.body.error).toContain('items');
    });

    it('should return 401 without authentication', async () => {
      // Act
      const response = await request
        .post('/api/orders')
        .send({ items: [{ productId: 'prod-1', quantity: 1, price: 100 }] });

      // Assert
      expect(response.status).toBe(401);
    });
  });

  describe('GET /api/orders', () => {
    it('should return user orders', async () => {
      // Arrange: 注文を作成
      await prisma.order.createMany({
        data: [
          { userId: testUser.id, total: 1000, status: 'PENDING' },
          { userId: testUser.id, total: 2000, status: 'COMPLETED' },
        ],
      });

      // Act
      const response = await request
        .get('/api/orders')
        .set('Authorization', `Bearer ${authToken}`);

      // Assert
      expect(response.status).toBe(200);
      expect(response.body.orders).toHaveLength(2);
    });

    it('should support pagination', async () => {
      // Arrange: 複数の注文を作成
      for (let i = 0; i < 15; i++) {
        await prisma.order.create({
          data: { userId: testUser.id, total: 100 * i, status: 'PENDING' },
        });
      }

      // Act
      const response = await request
        .get('/api/orders?page=1&limit=10')
        .set('Authorization', `Bearer ${authToken}`);

      // Assert
      expect(response.status).toBe(200);
      expect(response.body.orders).toHaveLength(10);
      expect(response.body.pagination).toMatchObject({
        page: 1,
        limit: 10,
        total: 15,
        totalPages: 2,
      });
    });
  });

  describe('GET /api/orders/:id', () => {
    it('should return order details', async () => {
      // Arrange
      const order = await prisma.order.create({
        data: {
          userId: testUser.id,
          total: 1500,
          status: 'PENDING',
          items: {
            create: [
              { productId: 'prod-1', quantity: 1, price: 1000 },
              { productId: 'prod-2', quantity: 1, price: 500 },
            ],
          },
        },
        include: { items: true },
      });

      // Act
      const response = await request
        .get(`/api/orders/${order.id}`)
        .set('Authorization', `Bearer ${authToken}`);

      // Assert
      expect(response.status).toBe(200);
      expect(response.body.id).toBe(order.id);
      expect(response.body.items).toHaveLength(2);
    });

    it('should return 404 for non-existent order', async () => {
      // Act
      const response = await request
        .get('/api/orders/non-existent-id')
        .set('Authorization', `Bearer ${authToken}`);

      // Assert
      expect(response.status).toBe(404);
    });

    it('should return 403 for other user order', async () => {
      // Arrange: 別ユーザーの注文を作成
      const otherUser = await prisma.user.create({
        data: { email: 'other@example.com', password: 'hash', name: 'Other' },
      });
      const otherOrder = await prisma.order.create({
        data: { userId: otherUser.id, total: 100, status: 'PENDING' },
      });

      // Act
      const response = await request
        .get(`/api/orders/${otherOrder.id}`)
        .set('Authorization', `Bearer ${authToken}`);

      // Assert
      expect(response.status).toBe(403);

      // Cleanup
      await prisma.order.delete({ where: { id: otherOrder.id } });
      await prisma.user.delete({ where: { id: otherUser.id } });
    });
  });
});
```

### 2.3 E2Eテスト（15%）

#### 2.3.1 E2Eテスト方針

```yaml
e2e_test_strategy:
  definition:
    description: 実際のユーザーフローを完全にシミュレート
    scope: クリティカルユーザージャーニー全体
    environment: ステージング環境または専用E2E環境

  critical_flows:
    user_registration:
      priority: P0
      steps:
        - アプリ起動
        - 新規登録画面へ遷移
        - 情報入力
        - メール確認
        - ログイン完了

    product_purchase:
      priority: P0
      steps:
        - 商品検索
        - 商品詳細表示
        - カートに追加
        - チェックアウト
        - 決済完了
        - 注文確認

    hotel_booking:
      priority: P0
      steps:
        - ホテル検索
        - 日付選択
        - 部屋選択
        - 予約情報入力
        - 決済
        - 予約確認

    ticket_purchase:
      priority: P0
      steps:
        - アトラクション検索
        - チケット選択
        - 日時選択
        - 決済
        - QRコード発行

  tools:
    flutter:
      - Patrol 3.19.0
      - integration_test
    web:
      - Playwright (将来)

  execution:
    environment: staging
    frequency:
      - PR: smoke tests only
      - main branch: full suite
      - nightly: full suite + exploratory
    parallelization: by feature
    retry: 2 attempts on failure
```

#### 2.3.2 Flutter E2Eテスト例

```dart
// integration_test/flows/purchase_flow_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:triptrip/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('Product Purchase Flow E2E', () {
    testWidgets('complete purchase flow from search to confirmation',
        (tester) async {
      // Launch app
      app.main();
      await tester.pumpAndSettle();

      // Step 1: Navigate to search
      await tester.tap(find.byKey(const Key('nav_search')));
      await tester.pumpAndSettle();

      // Step 2: Search for product
      await tester.enterText(
        find.byKey(const Key('search_input')),
        'kimono',
      );
      await tester.tap(find.byKey(const Key('search_button')));
      await tester.pumpAndSettle();

      // Verify search results
      expect(find.byType(ProductCard), findsWidgets);

      // Step 3: Select first product
      await tester.tap(find.byType(ProductCard).first);
      await tester.pumpAndSettle();

      // Verify product detail screen
      expect(find.byKey(const Key('product_detail_screen')), findsOneWidget);
      expect(find.text('Add to Cart'), findsOneWidget);

      // Step 4: Select size and add to cart
      await tester.tap(find.text('M'));
      await tester.pumpAndSettle();
      await tester.tap(find.text('Add to Cart'));
      await tester.pumpAndSettle();

      // Verify cart badge updated
      expect(find.byKey(const Key('cart_badge')), findsOneWidget);

      // Step 5: Navigate to cart
      await tester.tap(find.byKey(const Key('nav_cart')));
      await tester.pumpAndSettle();

      // Verify cart contents
      expect(find.byType(CartItemWidget), findsOneWidget);

      // Step 6: Proceed to checkout
      await tester.tap(find.text('Proceed to Checkout'));
      await tester.pumpAndSettle();

      // Step 7: Fill shipping information
      await tester.enterText(
        find.byKey(const Key('shipping_name')),
        'Test User',
      );
      await tester.enterText(
        find.byKey(const Key('shipping_address')),
        '123 Test Street, Tokyo',
      );
      await tester.enterText(
        find.byKey(const Key('shipping_phone')),
        '090-1234-5678',
      );
      await tester.tap(find.text('Continue to Payment'));
      await tester.pumpAndSettle();

      // Step 8: Select payment method (テスト環境ではモック決済)
      await tester.tap(find.text('Credit Card'));
      await tester.pumpAndSettle();

      // Use test card
      await tester.enterText(
        find.byKey(const Key('card_number')),
        '4242424242424242',
      );
      await tester.enterText(
        find.byKey(const Key('card_expiry')),
        '12/28',
      );
      await tester.enterText(
        find.byKey(const Key('card_cvc')),
        '123',
      );
      await tester.tap(find.text('Pay Now'));
      await tester.pumpAndSettle(const Duration(seconds: 5));

      // Step 9: Verify order confirmation
      expect(find.byKey(const Key('order_confirmation_screen')), findsOneWidget);
      expect(find.text('Order Confirmed'), findsOneWidget);
      expect(find.byKey(const Key('order_number')), findsOneWidget);

      // Verify order in history
      await tester.tap(find.text('View Order History'));
      await tester.pumpAndSettle();
      expect(find.byType(OrderHistoryItem), findsWidgets);
    });

    testWidgets('handles payment failure gracefully', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // ... navigate to checkout ...

      // Use declined test card
      await tester.enterText(
        find.byKey(const Key('card_number')),
        '4000000000000002', // Declined card
      );
      await tester.tap(find.text('Pay Now'));
      await tester.pumpAndSettle();

      // Verify error handling
      expect(find.text('Payment Failed'), findsOneWidget);
      expect(find.text('Please try again'), findsOneWidget);

      // Cart should still have items
      await tester.tap(find.byKey(const Key('nav_cart')));
      await tester.pumpAndSettle();
      expect(find.byType(CartItemWidget), findsOneWidget);
    });
  });

  group('Hotel Booking Flow E2E', () {
    testWidgets('complete hotel booking flow', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // Step 1: Navigate to hotels
      await tester.tap(find.byKey(const Key('home_hotels_card')));
      await tester.pumpAndSettle();

      // Step 2: Search hotels
      await tester.tap(find.byKey(const Key('hotel_search_location')));
      await tester.enterText(find.byKey(const Key('location_input')), 'Tokyo');
      await tester.tap(find.text('Tokyo, Japan').first);
      await tester.pumpAndSettle();

      // Select dates
      await tester.tap(find.byKey(const Key('check_in_date')));
      await tester.pumpAndSettle();
      // Select tomorrow
      final tomorrow = DateTime.now().add(const Duration(days: 1));
      await tester.tap(find.text('${tomorrow.day}'));
      await tester.pumpAndSettle();
      // Select day after
      final checkout = tomorrow.add(const Duration(days: 2));
      await tester.tap(find.text('${checkout.day}'));
      await tester.tap(find.text('Confirm'));
      await tester.pumpAndSettle();

      // Search
      await tester.tap(find.text('Search Hotels'));
      await tester.pumpAndSettle();

      // Step 3: Select hotel
      expect(find.byType(HotelCard), findsWidgets);
      await tester.tap(find.byType(HotelCard).first);
      await tester.pumpAndSettle();

      // Step 4: Select room
      await tester.tap(find.byType(RoomOptionCard).first);
      await tester.tap(find.text('Book Now'));
      await tester.pumpAndSettle();

      // Step 5: Complete booking
      await tester.enterText(find.byKey(const Key('guest_name')), 'Test User');
      await tester.enterText(find.byKey(const Key('guest_email')), 'test@example.com');
      await tester.tap(find.text('Confirm Booking'));
      await tester.pumpAndSettle(const Duration(seconds: 5));

      // Verify confirmation
      expect(find.text('Booking Confirmed'), findsOneWidget);
      expect(find.byKey(const Key('booking_reference')), findsOneWidget);
    });
  });
}
```

---

## 第3章 テスト自動化

### 3.1 フレームワーク選定

#### 3.1.1 テストツールスタック

```yaml
test_toolstack:
  flutter:
    unit_testing:
      framework: flutter_test
      mocking: mockito
      assertions: expect, matchers

    widget_testing:
      framework: flutter_test
      golden_testing: golden_toolkit
      screen_capture: golden files

    integration_testing:
      framework: Patrol 3.19.0
      alternative: integration_test
      device_testing: Firebase Test Lab

  backend:
    unit_testing:
      framework: Vitest 1.0.0
      mocking: vi.mock, vi.fn
      assertions: expect

    integration_testing:
      framework: Vitest + Supertest
      database: Docker PostgreSQL
      fixtures: factory-based

    api_testing:
      framework: Supertest 6.3.3
      schema_validation: zod
      documentation: OpenAPI

  cross_platform:
    contract_testing:
      framework: Pact (Year 2)
      purpose: API契約の検証

    visual_regression:
      framework: Percy / Chromatic (Year 2)
      purpose: UIの視覚的回帰検出

  performance:
    load_testing:
      framework: k6
      scenarios: stress, spike, soak

    frontend_performance:
      framework: Lighthouse CI
      metrics: LCP, FID, CLS

  security:
    sast:
      tools: [SonarQube, Snyk]
    dast:
      tools: [OWASP ZAP]
    dependency_scanning:
      tools: [Snyk, npm audit]
```

### 3.2 CI/CD統合

#### 3.2.1 テストパイプライン設計

```yaml
test_pipeline:
  stages:
    pre_commit:
      trigger: git pre-commit hook
      tests:
        - lint
        - format check
        - affected unit tests
      max_time: 30 seconds

    pull_request:
      trigger: PR作成/更新
      tests:
        - all unit tests
        - affected integration tests
        - e2e smoke tests
        - security scan (SAST)
      max_time: 15 minutes
      required_for_merge: true

    main_branch:
      trigger: main へのマージ
      tests:
        - full unit test suite
        - full integration test suite
        - full e2e test suite
        - performance baseline
      max_time: 30 minutes

    nightly:
      trigger: 毎日 2:00 AM JST
      tests:
        - full test suite
        - extended e2e scenarios
        - performance tests
        - security scan (DAST)
        - dependency audit
      max_time: 2 hours

    release:
      trigger: リリースタグ作成
      tests:
        - full regression suite
        - production smoke tests
        - rollback verification
      max_time: 1 hour
```

#### 3.2.2 GitHub Actionsテストワークフロー

```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 17 * * *' # 2:00 AM JST

env:
  FLUTTER_VERSION: '3.24.0'
  NODE_VERSION: '20'

jobs:
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    strategy:
      matrix:
        component: [flutter, backend]

    steps:
      - uses: actions/checkout@v4

      - name: Setup Flutter
        if: matrix.component == 'flutter'
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true

      - name: Setup Node
        if: matrix.component == 'backend'
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: backend/package-lock.json

      - name: Run Flutter Unit Tests
        if: matrix.component == 'flutter'
        run: |
          cd app
          flutter pub get
          flutter test --coverage --reporter=expanded

      - name: Run Backend Unit Tests
        if: matrix.component == 'backend'
        run: |
          cd backend
          npm ci
          npm run test:unit -- --coverage

      - name: Upload Coverage
        uses: codecov/codecov-action@v4
        with:
          flags: ${{ matrix.component }}-unit
          fail_ci_if_error: true

  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: unit-tests
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: triptrip_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install Dependencies
        run: |
          cd backend
          npm ci

      - name: Run Migrations
        run: |
          cd backend
          npx prisma migrate deploy
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/triptrip_test

      - name: Run Integration Tests
        run: |
          cd backend
          npm run test:integration -- --coverage
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/triptrip_test
          NODE_ENV: test

      - name: Upload Coverage
        uses: codecov/codecov-action@v4
        with:
          flags: backend-integration

  e2e-tests:
    name: E2E Tests
    runs-on: ubuntu-latest
    needs: integration-tests
    if: github.event_name == 'push' || github.event.pull_request.draft == false

    steps:
      - uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true

      - name: Start Backend
        run: |
          cd backend
          npm ci
          npm run start:test &
          sleep 15

      - name: Run E2E Smoke Tests
        if: github.event_name == 'pull_request'
        run: |
          cd app
          flutter pub get
          flutter test integration_test/smoke/

      - name: Run Full E2E Tests
        if: github.event_name == 'push'
        run: |
          cd app
          flutter pub get
          flutter test integration_test/

  coverage-check:
    name: Coverage Check
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]

    steps:
      - name: Check Coverage Threshold
        uses: codecov/codecov-action@v4
        with:
          fail_ci_if_error: true
          # Coverage thresholds defined in codecov.yml
```

### 3.3 テストデータ管理

#### 3.3.1 テストデータ戦略

```yaml
test_data_strategy:
  approaches:
    factories:
      description: プログラマティックなテストデータ生成
      tools:
        flutter: factory pattern (custom)
        backend: "@quramy/prisma-fabbrica"
      use_cases:
        - 単体テスト
        - 統合テスト
        - 動的データ生成

    fixtures:
      description: 静的なテストデータファイル
      format: JSON / YAML
      use_cases:
        - 安定したテストデータ
        - シードデータ
        - スナップショットテスト

    fake_data:
      description: リアルなダミーデータ生成
      tools:
        - faker.js (backend)
        - faker (dart package)
      use_cases:
        - ランダムテストデータ
        - 境界値テスト

  data_isolation:
    unit_tests:
      approach: in-memory mocks
      isolation: 完全分離

    integration_tests:
      approach: transaction rollback
      isolation: テストごとにロールバック

    e2e_tests:
      approach: dedicated test database
      cleanup: テストスイート後にクリア

  sensitive_data:
    policy: テストデータにPII/機密情報は使用しない
    approach:
      - anonymized production data（本番からの匿名化データ）
      - synthetic data（合成データ）
    validation:
      - 自動スキャンでPII検出
      - レビューでのチェック
```

#### 3.3.2 テストファクトリー例

```typescript
// backend/src/test/factories/order.factory.ts
import { faker } from '@faker-js/faker';
import { PrismaClient, Order, OrderStatus } from '@prisma/client';

interface OrderFactoryOptions {
  userId?: string;
  status?: OrderStatus;
  total?: number;
  itemCount?: number;
}

export class OrderFactory {
  constructor(private prisma: PrismaClient) {}

  async create(options: OrderFactoryOptions = {}): Promise<Order> {
    const {
      userId = faker.string.uuid(),
      status = OrderStatus.PENDING,
      total = faker.number.int({ min: 100, max: 100000 }),
      itemCount = faker.number.int({ min: 1, max: 5 }),
    } = options;

    return this.prisma.order.create({
      data: {
        userId,
        status,
        total,
        items: {
          create: Array.from({ length: itemCount }, () => ({
            productId: faker.string.uuid(),
            quantity: faker.number.int({ min: 1, max: 10 }),
            price: faker.number.int({ min: 100, max: 10000 }),
          })),
        },
        shippingAddress: this.generateAddress(),
      },
      include: {
        items: true,
      },
    });
  }

  async createMany(count: number, options: OrderFactoryOptions = {}): Promise<Order[]> {
    return Promise.all(
      Array.from({ length: count }, () => this.create(options))
    );
  }

  private generateAddress() {
    return {
      street: faker.location.streetAddress(),
      city: faker.location.city(),
      postalCode: faker.location.zipCode(),
      country: 'Japan',
    };
  }
}

// Usage in tests
describe('Order queries', () => {
  let factory: OrderFactory;

  beforeAll(() => {
    factory = new OrderFactory(prisma);
  });

  it('should return orders with pagination', async () => {
    // Create test data
    await factory.createMany(25, { userId: testUser.id });

    // Test pagination
    const result = await orderService.findByUser(testUser.id, { page: 1, limit: 10 });

    expect(result.orders).toHaveLength(10);
    expect(result.total).toBe(25);
  });
});
```

---

## 第4章 専門テスト

### 4.1 パフォーマンステスト

#### 4.1.1 パフォーマンステスト戦略

```yaml
performance_test_strategy:
  test_types:
    load_testing:
      description: 予想される通常負荷での性能検証
      tool: k6
      scenarios:
        - concurrent_users: 100
          duration: 10m
          ramp_up: 2m

    stress_testing:
      description: 限界負荷での性能検証
      tool: k6
      scenarios:
        - stages:
            - duration: 5m, target: 100
            - duration: 10m, target: 500
            - duration: 5m, target: 1000
            - duration: 10m, target: 0

    spike_testing:
      description: 急激な負荷増加への耐性検証
      tool: k6
      scenarios:
        - stages:
            - duration: 1m, target: 100
            - duration: 30s, target: 1000
            - duration: 1m, target: 100

    soak_testing:
      description: 長時間稼働での安定性検証
      tool: k6
      scenarios:
        - duration: 4h
          concurrent_users: 50

  metrics:
    response_time:
      p50_target: 100ms
      p95_target: 200ms
      p99_target: 500ms

    throughput:
      target: 1000 rps

    error_rate:
      target: 0.1%

    resource_utilization:
      cpu_threshold: 80%
      memory_threshold: 85%

  critical_endpoints:
    - GET /api/products (search)
    - GET /api/hotels (search)
    - POST /api/orders (create order)
    - POST /api/payments (process payment)
```

#### 4.1.2 k6パフォーマンステストスクリプト

```javascript
// performance/load-test.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const searchLatency = new Trend('search_latency');
const orderLatency = new Trend('order_latency');

// Test configuration
export const options = {
  stages: [
    { duration: '2m', target: 50 },   // Ramp up
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 200 },  // Ramp up
    { duration: '5m', target: 200 },  // Stay at 200 users
    { duration: '2m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<200', 'p(99)<500'],
    errors: ['rate<0.01'],
    search_latency: ['p(95)<150'],
    order_latency: ['p(95)<300'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'https://api.staging.triptrip.com';
const AUTH_TOKEN = __ENV.AUTH_TOKEN;

export default function () {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${AUTH_TOKEN}`,
  };

  group('Product Search Flow', () => {
    // Search products
    const searchStart = Date.now();
    const searchRes = http.get(
      `${BASE_URL}/api/products?q=kimono&limit=20`,
      { headers }
    );
    searchLatency.add(Date.now() - searchStart);

    check(searchRes, {
      'search status is 200': (r) => r.status === 200,
      'search returns products': (r) => JSON.parse(r.body).products.length > 0,
    }) || errorRate.add(1);

    sleep(1);

    // View product detail
    if (searchRes.status === 200) {
      const products = JSON.parse(searchRes.body).products;
      if (products.length > 0) {
        const productId = products[0].id;
        const detailRes = http.get(
          `${BASE_URL}/api/products/${productId}`,
          { headers }
        );

        check(detailRes, {
          'product detail status is 200': (r) => r.status === 200,
        }) || errorRate.add(1);
      }
    }
  });

  sleep(Math.random() * 3);

  group('Order Creation Flow', () => {
    // Add to cart (simulated by direct order creation)
    const orderPayload = JSON.stringify({
      items: [
        { productId: 'test-product-1', quantity: 1, price: 1000 },
      ],
      shippingAddress: {
        street: '123 Test St',
        city: 'Tokyo',
        postalCode: '100-0001',
        country: 'Japan',
      },
    });

    const orderStart = Date.now();
    const orderRes = http.post(
      `${BASE_URL}/api/orders`,
      orderPayload,
      { headers }
    );
    orderLatency.add(Date.now() - orderStart);

    check(orderRes, {
      'order creation status is 201': (r) => r.status === 201,
      'order has id': (r) => JSON.parse(r.body).id !== undefined,
    }) || errorRate.add(1);
  });

  sleep(Math.random() * 2);

  group('Hotel Search Flow', () => {
    const hotelSearchRes = http.get(
      `${BASE_URL}/api/hotels?location=tokyo&checkIn=2026-03-01&checkOut=2026-03-03`,
      { headers }
    );

    check(hotelSearchRes, {
      'hotel search status is 200': (r) => r.status === 200,
      'hotel search returns results': (r) => JSON.parse(r.body).hotels.length > 0,
    }) || errorRate.add(1);
  });

  sleep(1);
}

export function handleSummary(data) {
  return {
    'performance-report.json': JSON.stringify(data),
    stdout: textSummary(data, { indent: ' ', enableColors: true }),
  };
}
```

### 4.2 セキュリティテスト

#### 4.2.1 セキュリティテスト戦略

```yaml
security_test_strategy:
  static_analysis:
    sast:
      tool: SonarQube
      frequency: every_commit
      rules:
        - OWASP Top 10
        - CWE Top 25
      blocking: critical, high

    dependency_scanning:
      tool: Snyk
      frequency: every_commit
      auto_fix: minor/patch updates
      blocking: critical, high

    secret_scanning:
      tool: TruffleHog
      frequency: every_commit
      blocking: any_finding

  dynamic_analysis:
    dast:
      tool: OWASP ZAP
      frequency: weekly
      targets:
        - staging environment
      scan_types:
        - spider
        - active_scan
        - api_scan

    penetration_testing:
      frequency: annual
      provider: external
      scope: full application

  specific_tests:
    authentication:
      - brute_force_protection
      - session_management
      - password_policy
      - mfa_bypass_attempts

    authorization:
      - idor_testing
      - privilege_escalation
      - role_based_access

    injection:
      - sql_injection
      - nosql_injection
      - command_injection
      - xss

    api_security:
      - rate_limiting
      - input_validation
      - output_encoding
      - cors_configuration

  compliance:
    pci_dss:
      applicable: yes (payment processing)
      scan_frequency: quarterly

    gdpr:
      applicable: yes (personal data)
      review_frequency: annual
```

### 4.3 アクセシビリティテスト

#### 4.3.1 アクセシビリティテスト戦略

```yaml
accessibility_test_strategy:
  standards:
    primary: WCAG 2.1 Level AA
    mobile: iOS/Android accessibility guidelines

  test_types:
    automated:
      flutter:
        - Semantics widget testing
        - Screen reader compatibility
        - Contrast ratio checks
      tools:
        - flutter_accessibility_service
        - Accessibility Inspector (iOS)
        - Accessibility Scanner (Android)

    manual:
      frequency: quarterly
      methods:
        - Screen reader testing (VoiceOver, TalkBack)
        - Keyboard navigation
        - High contrast mode
        - Font scaling

    user_testing:
      frequency: bi-annual
      participants: users with disabilities

  checklist:
    visual:
      - [ ] 十分なコントラスト比 (4.5:1 以上)
      - [ ] 色だけに依存しない情報伝達
      - [ ] テキストリサイズ対応 (200%まで)
      - [ ] フォーカス表示の明確さ

    auditory:
      - [ ] 音声コンテンツの代替テキスト
      - [ ] キャプション/字幕

    motor:
      - [ ] タッチターゲット (44x44pt以上)
      - [ ] ジェスチャーの代替操作
      - [ ] タイムアウトの延長可能

    cognitive:
      - [ ] 一貫したナビゲーション
      - [ ] 明確なエラーメッセージ
      - [ ] 予測可能な操作

  flutter_implementation:
    semantics:
      - Semantics widget for all interactive elements
      - excludeSemantics for decorative elements
    testing:
      - SemanticsService testing
      - Accessibility action testing
```

---

## 第5章 品質ゲート

### 5.1 コードレビュー基準

#### 5.1.1 コードレビューチェックリスト

```yaml
code_review_checklist:
  functionality:
    - [ ] 要件を満たしている
    - [ ] エッジケースが処理されている
    - [ ] エラーハンドリングが適切

  code_quality:
    - [ ] 読みやすく理解しやすい
    - [ ] DRY原則に従っている
    - [ ] 適切な命名
    - [ ] 複雑度が許容範囲内
    - [ ] コメントが必要な箇所に存在

  testing:
    - [ ] ユニットテストが追加されている
    - [ ] テストが実際にコードを検証している
    - [ ] カバレッジが維持/向上している

  security:
    - [ ] 入力バリデーションが適切
    - [ ] 機密情報のハードコードなし
    - [ ] 認証/認可が正しく実装

  performance:
    - [ ] N+1クエリがない
    - [ ] 不必要なループがない
    - [ ] メモリリークの可能性がない

  documentation:
    - [ ] APIドキュメントが更新されている（該当する場合）
    - [ ] READMEが更新されている（該当する場合）

review_guidelines:
  turnaround_time:
    target: 24時間以内
    maximum: 48時間

  approvals_required: 2

  review_tone:
    - 建設的なフィードバック
    - 質問形式で提案
    - 良い点も指摘
    - 個人攻撃をしない
```

### 5.2 リリース前チェックリスト

#### 5.2.1 リリースゲート

```yaml
release_gates:
  pr_merge_gates:
    automated:
      - lint_pass: required
      - build_success: required
      - unit_tests_pass: required
      - integration_tests_pass: required
      - coverage_threshold: 80%
      - security_scan_pass: required
      - no_critical_vulnerabilities: required

    manual:
      - code_review_approvals: 2
      - design_review: if UI changes
      - security_review: if auth/payment changes

  staging_deployment_gates:
    automated:
      - all_pr_gates: pass
      - e2e_smoke_tests: pass
      - performance_baseline: no regression

    manual:
      - qa_sign_off: required
      - product_review: required for features

  production_deployment_gates:
    automated:
      - staging_verification: pass
      - security_scan_dast: pass
      - performance_test: pass
      - accessibility_audit: no critical issues

    manual:
      - release_manager_approval: required
      - on_call_engineer_notified: required
      - rollback_plan_verified: required

  post_deployment:
    - canary_analysis: 5 minutes minimum
    - error_rate_check: < 1%
    - latency_check: no regression
    - alert_verification: no new alerts
```

### 5.3 品質メトリクス

#### 5.3.1 品質ダッシュボード

```yaml
quality_dashboard:
  coverage_metrics:
    - name: Overall Code Coverage
      source: Codecov
      target: 80%
      trend: weekly

    - name: Critical Path Coverage
      source: Custom
      target: 95%
      paths:
        - /lib/features/checkout/
        - /lib/features/payment/
        - /src/services/order/
        - /src/services/payment/

  defect_metrics:
    - name: Open Bugs
      source: Jira
      breakdown: [P0, P1, P2, P3]

    - name: Bug Escape Rate
      source: Custom
      calculation: production_bugs / total_bugs

    - name: Mean Time to Resolution
      source: Jira
      breakdown_by: severity

  test_metrics:
    - name: Test Pass Rate
      source: CI
      target: 99%

    - name: Flaky Test Rate
      source: CI
      target: < 1%

    - name: Test Execution Time
      source: CI
      target: < 15 minutes

  release_metrics:
    - name: Deployment Frequency
      source: CI
      target: daily

    - name: Change Failure Rate
      source: Incident tracking
      target: < 5%

    - name: MTTR
      source: Incident tracking
      target: < 1 hour
```

---

## 第6章 実装ロードマップ & 文書間参照

### 6.1 テスト戦略導入タイムライン

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    テスト戦略導入タイムライン                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Year 1                                                                 │
│  ═══════════════════════════════════════════════════════════════════    │
│  Q1: 基盤整備                                                           │
│  ├─ テストフレームワーク標準化                                         │
│  ├─ CI/CDへのテスト完全統合                                            │
│  ├─ カバレッジ計測の自動化                                             │
│  └─ コードレビュー基準の策定                                           │
│                                                                         │
│  Q2: カバレッジ向上                                                     │
│  ├─ ユニットテスト拡充（目標: 70%）                                    │
│  ├─ 統合テスト整備                                                     │
│  ├─ E2Eテストスイート構築                                              │
│  └─ テストデータ管理標準化                                             │
│                                                                         │
│  Q3-Q4: 成熟化                                                          │
│  ├─ カバレッジ目標達成（80%）                                          │
│  ├─ パフォーマンステスト導入                                           │
│  ├─ セキュリティテスト統合                                             │
│  └─ 品質ダッシュボード構築                                             │
│                                                                         │
│  Year 2                                                                 │
│  ═══════════════════════════════════════════════════════════════════    │
│  H1: 高度化                                                             │
│  ├─ コントラクトテスト導入                                             │
│  ├─ 視覚的回帰テスト導入                                               │
│  ├─ カオスエンジニアリング開始                                         │
│  └─ テスト自動化率95%達成                                              │
│                                                                         │
│  H2: 最適化                                                             │
│  ├─ テスト実行時間最適化                                               │
│  ├─ 並列実行の拡大                                                     │
│  ├─ スマートテスト選択                                                 │
│  └─ 予測的品質分析                                                     │
│                                                                         │
│  Year 3                                                                 │
│  ═══════════════════════════════════════════════════════════════════    │
│  ├─ AI支援テスト生成                                                   │
│  ├─ 自己修復テスト                                                     │
│  ├─ 継続的品質インテリジェンス                                         │
│  └─ 品質成熟度レベル5達成                                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 関連文書参照

```yaml
document_references:
  directly_related:
    - Doc-DM-001: アジャイルデリバリーフレームワーク
    - Doc-DM-002: DevOps＆継続的デリバリー
    - Doc-QA-002: モニタリング、アラート＆インシデント管理
    - Doc-QA-003: リリース管理＆ロールバック手順

  technical:
    - Doc-SA-001: システムアーキテクチャ
    - Doc-SC-001: セキュリティアーキテクチャ
    - EXISTING_APP_ANALYSIS.md: 既存アプリ分析

  implementation:
    - Doc-IR-001: 開発ロードマップ Year 1-3
    - Doc-IR-002: リソース＆キャパシティプランニング
```

### 6.3 成功基準サマリー

```yaml
success_criteria:
  year_1:
    coverage:
      overall: 80%
      critical_paths: 95%

    automation:
      test_automation_rate: 85%
      ci_integration: 100%

    quality:
      defect_escape_rate: < 5%
      test_pass_rate: 99%

  year_2:
    coverage:
      overall: 85%
      critical_paths: 98%

    automation:
      test_automation_rate: 95%
      smart_test_selection: enabled

    quality:
      defect_escape_rate: < 2%
      test_execution_time: < 10 minutes

  year_3:
    coverage:
      overall: 90%
      ai_generated_tests: 20%

    automation:
      test_automation_rate: 98%
      self_healing_tests: enabled

    quality:
      defect_escape_rate: < 1%
      predictive_quality: enabled
```

---

## 付録

### 付録A: テストコーディング規約

```yaml
test_coding_standards:
  naming:
    test_files: "[subject]_test.dart" / "[subject].test.ts"
    test_cases: "[methodName]_[scenario]_[expectedResult]"

  structure:
    - Arrange: テストデータ準備
    - Act: テスト対象実行
    - Assert: 結果検証

  assertions:
    one_assertion_per_test: recommended
    clear_failure_messages: required

  mocking:
    mock_external_only: required
    no_implementation_details: required
```

### 付録B: カバレッジ除外ルール

```yaml
coverage_exclusions:
  flutter:
    - "**/*.g.dart"  # 生成コード
    - "**/*.freezed.dart"
    - "**/main.dart"
    - "**/constants/**"

  backend:
    - "**/*.d.ts"
    - "**/migrations/**"
    - "**/generated/**"
    - "**/config/**"
```

### 付録C: テスト環境設定

```yaml
test_environments:
  local:
    database: SQLite (in-memory)
    external_services: mocked

  ci:
    database: PostgreSQL (Docker)
    external_services: mocked/sandbox

  staging:
    database: Cloud SQL (dedicated)
    external_services: sandbox
```

---

**文書情報**

| 項目 | 内容 |
|------|------|
| Document ID | Doc-QA-001 |
| Version | 1.0.0 |
| Created | 2026-01-20 |
| Last Updated | 2026-01-20 |
| Status | Draft |
| Owner | Implementation Agent |
| Reviewers | QA Lead, CTO, Engineering Leads |
| Total Lines | 1,800+ |

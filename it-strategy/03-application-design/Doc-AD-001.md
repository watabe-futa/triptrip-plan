# Doc-AD-001: TripTripモバイルアプリケーションアーキテクチャ

## エグゼクティブサマリー

本文書は、TripTripモバイルアプリケーションの包括的なアーキテクチャ設計を定義します。既存のFlutter 3.8.1+基盤を活用しながら、Google、Uber、Airbnbレベルのモバイルアプリケーション品質を実現するための技術戦略を提示します。Clean Architectureの原則に基づくフィーチャー別モジュール設計、Riverpodによる状態管理、Hive/SQLiteによるオフラインファースト実装を通じて、数百万ユーザーに対応可能なスケーラブルで高品質なモバイル体験を提供します。本設計は、iOS/Androidプラットフォーム固有の最適化、パフォーマンス最適化、包括的なテスト戦略、CI/CD統合を含み、TripTripの技術ビジョン（Doc-TV-001）およびシステムアーキテクチャ（Doc-SA-001）と完全に整合しています。

---

## 第1章：はじめに & コンテキスト

### 1.1 目的 & スコープ

#### 1.1.1 文書の目的

本文書は、TripTripモバイルアプリケーションの技術アーキテクチャを包括的に定義し、以下の目的を達成します。

```yaml
document_objectives:
  primary:
    - description: Flutterアプリケーションアーキテクチャの標準化
      deliverable: アーキテクチャガイドライン、コーディング規約

    - description: 状態管理戦略の確立
      deliverable: Riverpod実装パターン、ベストプラクティス

    - description: オフラインファースト設計の実現
      deliverable: データ同期戦略、キャッシュアーキテクチャ

  secondary:
    - description: パフォーマンス最適化指針の提供
      deliverable: パフォーマンスバジェット、最適化手法

    - description: テスト戦略の確立
      deliverable: テストピラミッド、自動化戦略

    - description: CI/CD統合の設計
      deliverable: ビルドパイプライン、リリースフロー
```

#### 1.1.2 スコープ定義

```yaml
scope:
  in_scope:
    platforms:
      - iOS 14.0+
      - Android 8.0+ (API Level 26+)
      - Flutter Web (Progressive Web App)

    features:
      - アプリケーションアーキテクチャ設計
      - 状態管理パターン
      - データ永続化戦略
      - ネットワーク通信層
      - UI/UXコンポーネント設計
      - プラットフォーム固有機能統合
      - テスト戦略
      - CI/CD統合

  out_of_scope:
    - バックエンドAPI設計（Doc-AD-003で定義）
    - Webフロントエンドアーキテクチャ（Doc-AD-002で定義）
    - インフラストラクチャ設計（Doc-IA-001で定義）
```

### 1.2 既存Flutterアプリの分析

#### 1.2.1 現行アーキテクチャ評価

```yaml
current_state:
  codebase_metrics:
    total_lines: 34,917
    dart_files: 348
    feature_modules: 18
    test_files: 46
    version: 1.0.0+1

  technology_stack:
    framework: Flutter 3.8.1+
    language: Dart 3.8.1+
    state_management:
      primary: Provider 6.1.1
      secondary: Riverpod 2.4.9
    http_clients:
      - http 1.2.0
      - Dio 5.9.0
    local_storage:
      - Hive 2.2.3
      - SharedPreferences 2.2.2
    code_generation:
      - Freezed 3.0.0
      - JSON Serializable 6.7.1

  architecture_patterns:
    primary: Feature-based Package
    structure: |
      lib/
      ├── features/           # 18の機能モジュール
      ├── services/           # 共有サービス
      ├── providers/          # グローバル状態管理
      ├── common_components/  # 共有UIコンポーネント
      └── screens/            # レガシー画面（移行中）
```

#### 1.2.2 成熟度評価と課題

```yaml
maturity_assessment:
  strengths:
    - Feature-based構造による高いモジュール性
    - 型安全なAPI生成システム（OpenAPI駆動）
    - 成熟したコード生成パイプライン（Freezed）
    - 包括的なUI組件ライブラリ

  challenges:
    technical_debt:
      - id: TD-001
        description: レガシー画面の残存（/lib/screens/）
        impact: 保守性低下、二重管理
        remediation: features/への段階的移行

      - id: TD-002
        description: HTTPクライアントの混在（http vs Dio）
        impact: 一貫性欠如、冗長なコード
        remediation: Dioへの統一

      - id: TD-003
        description: 状態管理の混在（Provider vs Riverpod）
        impact: 学習コスト増大、パターン不統一
        remediation: Riverpodへの段階的移行

      - id: TD-004
        description: テストカバレッジ不足（推定40%）
        impact: リグレッションリスク
        remediation: テスト戦略の確立と実施

  gaps:
    - オフライン同期メカニズムの未実装
    - リアルタイム更新機能（WebSocket）の欠如
    - 分析・トラッキング統合の不在
    - プッシュ通知の未実装
```

### 1.3 アーキテクチャ原則

#### 1.3.1 設計原則

```yaml
design_principles:
  clean_architecture:
    description: 依存性の方向を外部から内部へ制限
    implementation:
      - Presentation Layer → Domain Layer → Data Layer
      - 依存性逆転の原則（DIP）の適用
      - ビジネスロジックのUI独立性

  separation_of_concerns:
    description: 責任の明確な分離
    implementation:
      - UI: 表示ロジックのみ
      - ViewModel/Controller: プレゼンテーションロジック
      - UseCase: ビジネスロジック
      - Repository: データアクセス抽象化

  testability:
    description: 高いテスト容易性の確保
    implementation:
      - 依存性注入によるモック可能性
      - 純粋関数の優先
      - 副作用の局所化

  scalability:
    description: 機能追加に対するスケーラビリティ
    implementation:
      - Feature-based モジュール構造
      - 独立した機能開発
      - 循環依存の防止
```

#### 1.3.2 品質属性

```yaml
quality_attributes:
  performance:
    app_launch_time:
      cold_start: < 2 seconds
      warm_start: < 500 milliseconds
    frame_rate:
      target: 60 FPS
      minimum: 30 FPS
    memory_usage:
      baseline: < 150 MB
      peak: < 300 MB

  reliability:
    crash_free_rate: > 99.9%
    anr_rate: < 0.1%
    error_handling: Graceful degradation

  usability:
    accessibility:
      - WCAG 2.1 Level AA準拠
      - VoiceOver/TalkBack対応
    internationalization:
      - 日本語、英語サポート
      - RTL言語対応準備

  maintainability:
    code_coverage: > 80%
    cyclomatic_complexity: < 10
    documentation: 公開API 100%
```

---

## 第2章：アプリケーションアーキテクチャ

### 2.1 Clean Architecture実装

#### 2.1.1 レイヤー構造

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CLEAN ARCHITECTURE LAYERS                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                      PRESENTATION LAYER                              │    │
│  │                                                                      │    │
│  │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │    │
│  │   │   Screens   │    │   Widgets   │    │ Controllers │            │    │
│  │   │   (Pages)   │    │(Components) │    │ (ViewModels)│            │    │
│  │   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘            │    │
│  │          │                  │                  │                    │    │
│  │          └──────────────────┴──────────────────┘                    │    │
│  │                             │                                        │    │
│  │                             ▼                                        │    │
│  │                    State Management                                  │    │
│  │                      (Riverpod)                                      │    │
│  └─────────────────────────────┬───────────────────────────────────────┘    │
│                                │                                             │
│                                ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        DOMAIN LAYER                                  │    │
│  │                                                                      │    │
│  │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │    │
│  │   │   Entities  │    │  Use Cases  │    │ Repository  │            │    │
│  │   │   (Models)  │    │  (Business) │    │ Interfaces  │            │    │
│  │   └─────────────┘    └─────────────┘    └─────────────┘            │    │
│  │                                                                      │    │
│  └─────────────────────────────┬───────────────────────────────────────┘    │
│                                │                                             │
│                                ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         DATA LAYER                                   │    │
│  │                                                                      │    │
│  │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │    │
│  │   │ Repository  │    │ Data Sources│    │   Models    │            │    │
│  │   │   Impl      │    │(Remote/Local)│   │  (DTOs)     │            │    │
│  │   └─────────────┘    └─────────────┘    └─────────────┘            │    │
│  │                                                                      │    │
│  │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │    │
│  │   │   Network   │    │   Storage   │    │   Cache     │            │    │
│  │   │   (Dio)     │    │(Hive/SQLite)│    │  (Memory)   │            │    │
│  │   └─────────────┘    └─────────────┘    └─────────────┘            │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 2.1.2 各レイヤーの責任

```dart
// ==========================================
// PRESENTATION LAYER
// ==========================================

/// Screen（Page）: UIの組み立てと画面遷移
class HotelDetailScreen extends ConsumerWidget {
  final String hotelId;

  const HotelDetailScreen({required this.hotelId, super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final hotelState = ref.watch(hotelDetailProvider(hotelId));

    return Scaffold(
      body: hotelState.when(
        data: (hotel) => HotelDetailContent(hotel: hotel),
        loading: () => const HotelDetailSkeleton(),
        error: (error, stack) => ErrorWidget(
          message: error.toString(),
          onRetry: () => ref.invalidate(hotelDetailProvider(hotelId)),
        ),
      ),
    );
  }
}

/// Widget（Component）: 再利用可能なUIコンポーネント
class HotelCard extends StatelessWidget {
  final Hotel hotel;
  final VoidCallback? onTap;

  const HotelCard({required this.hotel, this.onTap, super.key});

  @override
  Widget build(BuildContext context) {
    return Card(
      child: InkWell(
        onTap: onTap,
        child: Column(
          children: [
            HotelImage(url: hotel.imageUrl),
            HotelInfo(hotel: hotel),
            HotelPrice(price: hotel.price),
          ],
        ),
      ),
    );
  }
}

// ==========================================
// DOMAIN LAYER
// ==========================================

/// Entity: ビジネスドメインのコアモデル
@freezed
class Hotel with _$Hotel {
  const factory Hotel({
    required String id,
    required String name,
    required String description,
    required Location location,
    required double rating,
    required int reviewCount,
    required PriceRange priceRange,
    required List<String> amenities,
    required List<HotelImage> images,
    required bool isAvailable,
  }) = _Hotel;
}

/// UseCase: ビジネスロジックのカプセル化
class SearchHotelsUseCase {
  final HotelRepository _repository;

  SearchHotelsUseCase(this._repository);

  Future<Result<List<Hotel>>> execute(SearchCriteria criteria) async {
    // バリデーション
    if (criteria.checkIn.isAfter(criteria.checkOut)) {
      return Result.failure(ValidationError('Invalid date range'));
    }

    // 検索実行
    final result = await _repository.searchHotels(criteria);

    // ビジネスルール適用
    return result.map((hotels) => hotels
      .where((h) => h.isAvailable)
      .toList()
      ..sort((a, b) => b.rating.compareTo(a.rating)));
  }
}

/// Repository Interface: データアクセスの抽象化
abstract class HotelRepository {
  Future<Result<List<Hotel>>> searchHotels(SearchCriteria criteria);
  Future<Result<Hotel>> getHotelById(String id);
  Future<Result<void>> saveRecentSearch(SearchCriteria criteria);
  Stream<List<Hotel>> watchFavoriteHotels();
}

// ==========================================
// DATA LAYER
// ==========================================

/// Repository Implementation: 具体的なデータ取得ロジック
class HotelRepositoryImpl implements HotelRepository {
  final HotelRemoteDataSource _remoteDataSource;
  final HotelLocalDataSource _localDataSource;
  final NetworkInfo _networkInfo;

  HotelRepositoryImpl({
    required HotelRemoteDataSource remoteDataSource,
    required HotelLocalDataSource localDataSource,
    required NetworkInfo networkInfo,
  }) : _remoteDataSource = remoteDataSource,
       _localDataSource = localDataSource,
       _networkInfo = networkInfo;

  @override
  Future<Result<List<Hotel>>> searchHotels(SearchCriteria criteria) async {
    if (await _networkInfo.isConnected) {
      try {
        final remoteDtos = await _remoteDataSource.searchHotels(criteria);
        final hotels = remoteDtos.map((dto) => dto.toEntity()).toList();

        // キャッシュに保存
        await _localDataSource.cacheSearchResults(criteria, remoteDtos);

        return Result.success(hotels);
      } on ServerException catch (e) {
        return Result.failure(ServerError(e.message));
      }
    } else {
      // オフライン時はキャッシュから取得
      try {
        final cachedDtos = await _localDataSource.getCachedSearchResults(criteria);
        final hotels = cachedDtos.map((dto) => dto.toEntity()).toList();
        return Result.success(hotels);
      } on CacheException {
        return Result.failure(CacheError('No cached data available'));
      }
    }
  }
}

/// Remote Data Source: API通信
class HotelRemoteDataSource {
  final Dio _dio;

  HotelRemoteDataSource(this._dio);

  Future<List<HotelDto>> searchHotels(SearchCriteria criteria) async {
    final response = await _dio.get(
      '/api/v1/hotels/search',
      queryParameters: criteria.toQueryParams(),
    );

    return (response.data['results'] as List)
        .map((json) => HotelDto.fromJson(json))
        .toList();
  }
}

/// Local Data Source: ローカルストレージ
class HotelLocalDataSource {
  final Box<HotelDto> _hotelBox;
  final Box<SearchCacheEntry> _searchCacheBox;

  HotelLocalDataSource({
    required Box<HotelDto> hotelBox,
    required Box<SearchCacheEntry> searchCacheBox,
  }) : _hotelBox = hotelBox,
       _searchCacheBox = searchCacheBox;

  Future<void> cacheSearchResults(
    SearchCriteria criteria,
    List<HotelDto> results,
  ) async {
    final cacheKey = criteria.toCacheKey();
    await _searchCacheBox.put(cacheKey, SearchCacheEntry(
      criteria: criteria,
      results: results,
      cachedAt: DateTime.now(),
      expiresAt: DateTime.now().add(const Duration(hours: 1)),
    ));
  }

  Future<List<HotelDto>> getCachedSearchResults(SearchCriteria criteria) async {
    final cacheKey = criteria.toCacheKey();
    final entry = _searchCacheBox.get(cacheKey);

    if (entry == null || entry.isExpired) {
      throw CacheException('Cache miss or expired');
    }

    return entry.results;
  }
}

/// DTO: データ転送オブジェクト（API/DB形式）
@freezed
class HotelDto with _$HotelDto {
  const factory HotelDto({
    required String id,
    required String name,
    required String description,
    required double latitude,
    required double longitude,
    required String address,
    required double rating,
    required int reviewCount,
    required int minPrice,
    required int maxPrice,
    required String currency,
    required List<String> amenities,
    required List<String> imageUrls,
    required bool isAvailable,
  }) = _HotelDto;

  factory HotelDto.fromJson(Map<String, dynamic> json) =>
      _$HotelDtoFromJson(json);

  const HotelDto._();

  /// DTO → Entity変換
  Hotel toEntity() => Hotel(
    id: id,
    name: name,
    description: description,
    location: Location(
      latitude: latitude,
      longitude: longitude,
      address: address,
    ),
    rating: rating,
    reviewCount: reviewCount,
    priceRange: PriceRange(
      min: minPrice,
      max: maxPrice,
      currency: currency,
    ),
    amenities: amenities,
    images: imageUrls.map((url) => HotelImage(url: url)).toList(),
    isAvailable: isAvailable,
  );
}
```

### 2.2 フィーチャー別モジュール設計

#### 2.2.1 モジュール構造

```
lib/
├── app/                              # アプリケーション設定
│   ├── app.dart                      # MaterialApp設定
│   ├── router/                       # ルーティング
│   │   ├── app_router.dart
│   │   └── routes.dart
│   ├── theme/                        # テーマ設定
│   │   ├── app_theme.dart
│   │   ├── colors.dart
│   │   └── typography.dart
│   └── l10n/                         # 国際化
│       ├── app_localizations.dart
│       └── intl_*.arb
│
├── core/                             # コア機能
│   ├── constants/                    # 定数
│   │   ├── api_constants.dart
│   │   └── app_constants.dart
│   ├── error/                        # エラーハンドリング
│   │   ├── exceptions.dart
│   │   ├── failures.dart
│   │   └── error_handler.dart
│   ├── network/                      # ネットワーク基盤
│   │   ├── dio_client.dart
│   │   ├── interceptors/
│   │   │   ├── auth_interceptor.dart
│   │   │   ├── logging_interceptor.dart
│   │   │   └── retry_interceptor.dart
│   │   └── network_info.dart
│   ├── storage/                      # ストレージ基盤
│   │   ├── hive_storage.dart
│   │   ├── secure_storage.dart
│   │   └── shared_prefs.dart
│   ├── utils/                        # ユーティリティ
│   │   ├── date_utils.dart
│   │   ├── currency_utils.dart
│   │   └── validators.dart
│   └── extensions/                   # 拡張メソッド
│       ├── context_extensions.dart
│       ├── string_extensions.dart
│       └── datetime_extensions.dart
│
├── shared/                           # 共有コンポーネント
│   ├── widgets/                      # 共有ウィジェット
│   │   ├── buttons/
│   │   ├── cards/
│   │   ├── dialogs/
│   │   ├── inputs/
│   │   ├── loading/
│   │   └── layout/
│   ├── models/                       # 共有モデル
│   │   ├── result.dart
│   │   ├── pagination.dart
│   │   └── location.dart
│   └── providers/                    # 共有プロバイダー
│       ├── auth_provider.dart
│       ├── locale_provider.dart
│       └── theme_provider.dart
│
└── features/                         # 機能モジュール
    ├── auth/                         # 認証機能
    │   ├── data/
    │   │   ├── datasources/
    │   │   │   ├── auth_remote_datasource.dart
    │   │   │   └── auth_local_datasource.dart
    │   │   ├── models/
    │   │   │   ├── user_dto.dart
    │   │   │   └── auth_token_dto.dart
    │   │   └── repositories/
    │   │       └── auth_repository_impl.dart
    │   ├── domain/
    │   │   ├── entities/
    │   │   │   └── user.dart
    │   │   ├── repositories/
    │   │   │   └── auth_repository.dart
    │   │   └── usecases/
    │   │       ├── login_usecase.dart
    │   │       ├── register_usecase.dart
    │   │       └── logout_usecase.dart
    │   └── presentation/
    │       ├── providers/
    │       │   └── auth_provider.dart
    │       ├── screens/
    │       │   ├── login_screen.dart
    │       │   └── register_screen.dart
    │       └── widgets/
    │           ├── login_form.dart
    │           └── social_login_buttons.dart
    │
    ├── hotel/                        # ホテル機能
    │   ├── data/
    │   ├── domain/
    │   └── presentation/
    │
    ├── attraction/                   # アトラクション機能
    │   ├── data/
    │   ├── domain/
    │   └── presentation/
    │
    ├── commerce/                     # 商品・カート機能
    │   ├── data/
    │   ├── domain/
    │   └── presentation/
    │
    ├── booking/                      # 予約機能
    │   ├── data/
    │   ├── domain/
    │   └── presentation/
    │
    ├── search/                       # 検索機能
    │   ├── data/
    │   ├── domain/
    │   └── presentation/
    │
    └── profile/                      # プロフィール機能
        ├── data/
        ├── domain/
        └── presentation/
```

#### 2.2.2 フィーチャーモジュールテンプレート

```dart
// ==========================================
// features/hotel/hotel_module.dart
// ==========================================

/// フィーチャーモジュールの依存性登録
final hotelModule = [
  // Data Sources
  Provider<HotelRemoteDataSource>(
    (ref) => HotelRemoteDataSource(ref.watch(dioClientProvider)),
  ),
  Provider<HotelLocalDataSource>(
    (ref) => HotelLocalDataSource(
      hotelBox: ref.watch(hotelBoxProvider),
      searchCacheBox: ref.watch(searchCacheBoxProvider),
    ),
  ),

  // Repository
  Provider<HotelRepository>(
    (ref) => HotelRepositoryImpl(
      remoteDataSource: ref.watch(hotelRemoteDataSourceProvider),
      localDataSource: ref.watch(hotelLocalDataSourceProvider),
      networkInfo: ref.watch(networkInfoProvider),
    ),
  ),

  // Use Cases
  Provider<SearchHotelsUseCase>(
    (ref) => SearchHotelsUseCase(ref.watch(hotelRepositoryProvider)),
  ),
  Provider<GetHotelDetailUseCase>(
    (ref) => GetHotelDetailUseCase(ref.watch(hotelRepositoryProvider)),
  ),
  Provider<ToggleFavoriteHotelUseCase>(
    (ref) => ToggleFavoriteHotelUseCase(ref.watch(hotelRepositoryProvider)),
  ),
];

// ==========================================
// features/hotel/presentation/providers/hotel_providers.dart
// ==========================================

/// ホテル検索状態プロバイダー
@riverpod
class HotelSearchNotifier extends _$HotelSearchNotifier {
  @override
  AsyncValue<HotelSearchResult> build() {
    return const AsyncValue.data(HotelSearchResult.empty());
  }

  Future<void> search(HotelSearchCriteria criteria) async {
    state = const AsyncValue.loading();

    final useCase = ref.read(searchHotelsUseCaseProvider);
    final result = await useCase.execute(criteria);

    state = result.fold(
      (failure) => AsyncValue.error(failure, StackTrace.current),
      (hotels) => AsyncValue.data(HotelSearchResult(
        hotels: hotels,
        criteria: criteria,
        totalCount: hotels.length,
      )),
    );
  }

  Future<void> loadMore() async {
    final currentState = state.valueOrNull;
    if (currentState == null || !currentState.hasMore) return;

    final nextCriteria = currentState.criteria.copyWith(
      page: currentState.criteria.page + 1,
    );

    final useCase = ref.read(searchHotelsUseCaseProvider);
    final result = await useCase.execute(nextCriteria);

    result.fold(
      (failure) => {}, // エラー時は現在の状態を維持
      (hotels) => state = AsyncValue.data(currentState.copyWith(
        hotels: [...currentState.hotels, ...hotels],
        criteria: nextCriteria,
      )),
    );
  }
}

/// ホテル詳細プロバイダー
@riverpod
Future<Hotel> hotelDetail(HotelDetailRef ref, String hotelId) async {
  final useCase = ref.watch(getHotelDetailUseCaseProvider);
  final result = await useCase.execute(hotelId);

  return result.fold(
    (failure) => throw failure,
    (hotel) => hotel,
  );
}

/// お気に入りホテルプロバイダー
@riverpod
Stream<List<Hotel>> favoriteHotels(FavoriteHotelsRef ref) {
  final repository = ref.watch(hotelRepositoryProvider);
  return repository.watchFavoriteHotels();
}

/// お気に入りトグルプロバイダー
@riverpod
class FavoriteToggle extends _$FavoriteToggle {
  @override
  bool build(String hotelId) {
    final favorites = ref.watch(favoriteHotelsProvider).valueOrNull ?? [];
    return favorites.any((h) => h.id == hotelId);
  }

  Future<void> toggle() async {
    final useCase = ref.read(toggleFavoriteHotelUseCaseProvider);
    final currentState = state;

    // 楽観的更新
    state = !currentState;

    final result = await useCase.execute(arg);

    // 失敗時はロールバック
    result.fold(
      (failure) => state = currentState,
      (_) => {}, // 成功時は何もしない（既に更新済み）
    );
  }
}
```

### 2.3 依存性注入とサービスロケーター

#### 2.3.1 Riverpodによる依存性注入

```dart
// ==========================================
// core/di/providers.dart
// ==========================================

/// ネットワーク関連プロバイダー
@riverpod
Dio dioClient(DioClientRef ref) {
  final dio = Dio(BaseOptions(
    baseUrl: AppConstants.apiBaseUrl,
    connectTimeout: const Duration(seconds: 10),
    receiveTimeout: const Duration(seconds: 30),
    headers: {
      'Content-Type': 'application/json',
      'Accept': 'application/json',
    },
  ));

  // インターセプターの追加
  dio.interceptors.addAll([
    ref.watch(authInterceptorProvider),
    ref.watch(loggingInterceptorProvider),
    ref.watch(retryInterceptorProvider),
  ]);

  return dio;
}

@riverpod
AuthInterceptor authInterceptor(AuthInterceptorRef ref) {
  return AuthInterceptor(
    tokenStorage: ref.watch(tokenStorageProvider),
    onTokenExpired: () => ref.invalidate(authStateProvider),
  );
}

@riverpod
NetworkInfo networkInfo(NetworkInfoRef ref) {
  return NetworkInfoImpl(Connectivity());
}

/// ストレージ関連プロバイダー
@riverpod
Future<Box<dynamic>> hiveBox(HiveBoxRef ref, String boxName) async {
  if (!Hive.isBoxOpen(boxName)) {
    return await Hive.openBox(boxName);
  }
  return Hive.box(boxName);
}

@riverpod
FlutterSecureStorage secureStorage(SecureStorageRef ref) {
  return const FlutterSecureStorage(
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
    iOptions: IOSOptions(accessibility: KeychainAccessibility.first_unlock),
  );
}

@riverpod
TokenStorage tokenStorage(TokenStorageRef ref) {
  return TokenStorageImpl(ref.watch(secureStorageProvider));
}

/// 認証状態プロバイダー
@riverpod
class AuthState extends _$AuthState {
  @override
  Future<User?> build() async {
    final tokenStorage = ref.watch(tokenStorageProvider);
    final token = await tokenStorage.getAccessToken();

    if (token == null) return null;

    // トークンの有効性を確認
    if (JwtDecoder.isExpired(token)) {
      // リフレッシュトークンで更新を試みる
      final refreshed = await _refreshToken();
      if (!refreshed) return null;
    }

    // ユーザー情報を取得
    final repository = ref.read(authRepositoryProvider);
    final result = await repository.getCurrentUser();

    return result.fold(
      (failure) => null,
      (user) => user,
    );
  }

  Future<bool> _refreshToken() async {
    final repository = ref.read(authRepositoryProvider);
    final result = await repository.refreshToken();
    return result.isSuccess;
  }

  Future<void> login(String email, String password) async {
    state = const AsyncValue.loading();

    final useCase = ref.read(loginUseCaseProvider);
    final result = await useCase.execute(email, password);

    state = result.fold(
      (failure) => AsyncValue.error(failure, StackTrace.current),
      (user) => AsyncValue.data(user),
    );
  }

  Future<void> logout() async {
    final useCase = ref.read(logoutUseCaseProvider);
    await useCase.execute();
    state = const AsyncValue.data(null);
  }
}

/// グローバル設定プロバイダー
@riverpod
class LocaleNotifier extends _$LocaleNotifier {
  @override
  Locale build() {
    _loadSavedLocale();
    return const Locale('ja');
  }

  Future<void> _loadSavedLocale() async {
    final prefs = await SharedPreferences.getInstance();
    final languageCode = prefs.getString('language_code');
    if (languageCode != null) {
      state = Locale(languageCode);
    }
  }

  Future<void> setLocale(Locale locale) async {
    state = locale;
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString('language_code', locale.languageCode);
  }
}

@riverpod
class ThemeNotifier extends _$ThemeNotifier {
  @override
  ThemeMode build() {
    _loadSavedTheme();
    return ThemeMode.system;
  }

  Future<void> _loadSavedTheme() async {
    final prefs = await SharedPreferences.getInstance();
    final themeIndex = prefs.getInt('theme_mode');
    if (themeIndex != null) {
      state = ThemeMode.values[themeIndex];
    }
  }

  Future<void> setThemeMode(ThemeMode mode) async {
    state = mode;
    final prefs = await SharedPreferences.getInstance();
    await prefs.setInt('theme_mode', mode.index);
  }
}
```

#### 2.3.2 アプリケーション初期化

```dart
// ==========================================
// app/app.dart
// ==========================================

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // エラーハンドリング設定
  FlutterError.onError = (details) {
    FlutterError.presentError(details);
    // Crashlyticsに送信
    FirebaseCrashlytics.instance.recordFlutterFatalError(details);
  };

  PlatformDispatcher.instance.onError = (error, stack) {
    FirebaseCrashlytics.instance.recordError(error, stack, fatal: true);
    return true;
  };

  // サービス初期化
  await _initializeServices();

  runApp(
    ProviderScope(
      observers: [
        if (kDebugMode) LoggingProviderObserver(),
      ],
      child: const TripTripApp(),
    ),
  );
}

Future<void> _initializeServices() async {
  // Firebase初期化
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );

  // Hive初期化
  await Hive.initFlutter();
  Hive.registerAdapter(HotelDtoAdapter());
  Hive.registerAdapter(CartItemAdapter());
  Hive.registerAdapter(SearchCacheEntryAdapter());

  // 必要なBoxを事前にオープン
  await Future.wait([
    Hive.openBox<HotelDto>('hotels'),
    Hive.openBox<CartItem>('cart'),
    Hive.openBox<SearchCacheEntry>('search_cache'),
  ]);

  // 環境設定の読み込み
  await dotenv.load(fileName: '.env');
}

class TripTripApp extends ConsumerWidget {
  const TripTripApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final locale = ref.watch(localeNotifierProvider);
    final themeMode = ref.watch(themeNotifierProvider);
    final router = ref.watch(appRouterProvider);

    return MaterialApp.router(
      title: 'TripTrip',
      debugShowCheckedModeBanner: false,

      // テーマ設定
      theme: AppTheme.light,
      darkTheme: AppTheme.dark,
      themeMode: themeMode,

      // 国際化設定
      locale: locale,
      supportedLocales: AppLocalizations.supportedLocales,
      localizationsDelegates: AppLocalizations.localizationsDelegates,

      // ルーティング設定
      routerConfig: router,

      // ビルダー
      builder: (context, child) {
        return MediaQuery(
          // テキストスケールを制限
          data: MediaQuery.of(context).copyWith(
            textScaler: TextScaler.linear(
              MediaQuery.of(context).textScaler.scale(1.0).clamp(0.8, 1.2),
            ),
          ),
          child: child!,
        );
      },
    );
  }
}
```

---

## 第3章：状態管理設計

### 3.1 Riverpod実装パターン

#### 3.1.1 プロバイダータイプの選択基準

```yaml
provider_selection_guide:
  provider:
    description: 不変の値を提供
    use_cases:
      - シングルトンサービス
      - 設定値
      - 依存性注入
    example: "DIコンテナ、APIクライアント、リポジトリ"

  stateProvider:
    description: シンプルな変更可能状態
    use_cases:
      - フィルター選択
      - ソート順
      - UIフラグ
    example: "選択されたカテゴリ、表示モード"

  futureProvider:
    description: 非同期データの一度きりの取得
    use_cases:
      - APIからのデータ取得
      - 設定の読み込み
      - 初期化処理
    example: "ユーザープロフィール、設定データ"

  streamProvider:
    description: リアクティブなデータストリーム
    use_cases:
      - リアルタイム更新
      - データベース監視
      - WebSocket
    example: "お気に入りリスト、通知"

  notifierProvider:
    description: 複雑な状態とロジック
    use_cases:
      - フォーム状態管理
      - 複雑なUI状態
      - ビジネスロジック
    example: "ログインフォーム、検索フィルター"

  asyncNotifierProvider:
    description: 非同期操作を伴う複雑な状態
    use_cases:
      - ページネーション
      - 検索結果
      - CRUD操作
    example: "ホテル検索、カート管理"
```

#### 3.1.2 状態管理パターン実装

```dart
// ==========================================
// パターン1: シンプルな状態管理（StateProvider）
// ==========================================

/// 検索フィルター状態
@riverpod
class SearchFilter extends _$SearchFilter {
  @override
  HotelSearchFilter build() => const HotelSearchFilter();

  void setDateRange(DateTimeRange range) {
    state = state.copyWith(
      checkIn: range.start,
      checkOut: range.end,
    );
  }

  void setGuestCount(int adults, int children) {
    state = state.copyWith(
      adults: adults,
      children: children,
    );
  }

  void setPriceRange(int min, int max) {
    state = state.copyWith(
      minPrice: min,
      maxPrice: max,
    );
  }

  void toggleAmenity(String amenity) {
    final current = state.amenities;
    final updated = current.contains(amenity)
        ? current.where((a) => a != amenity).toList()
        : [...current, amenity];
    state = state.copyWith(amenities: updated);
  }

  void reset() {
    state = const HotelSearchFilter();
  }
}

// ==========================================
// パターン2: 非同期データ取得（FutureProvider）
// ==========================================

/// ホテル詳細取得（キャッシュ付き）
@Riverpod(keepAlive: true)
Future<Hotel> hotelDetail(HotelDetailRef ref, String hotelId) async {
  // 自動キャンセル設定
  final cancelToken = CancelToken();
  ref.onDispose(cancelToken.cancel);

  // キャッシュチェック
  final cache = ref.watch(hotelCacheProvider);
  final cached = cache.get(hotelId);
  if (cached != null) return cached;

  // API取得
  final repository = ref.watch(hotelRepositoryProvider);
  final result = await repository.getHotelById(hotelId);

  return result.fold(
    (failure) => throw failure,
    (hotel) {
      // キャッシュに保存
      cache.put(hotelId, hotel);
      return hotel;
    },
  );
}

// ==========================================
// パターン3: リアルタイムストリーム（StreamProvider）
// ==========================================

/// カートアイテム監視
@riverpod
Stream<List<CartItem>> cartItems(CartItemsRef ref) {
  final repository = ref.watch(cartRepositoryProvider);
  return repository.watchCartItems();
}

/// カート合計金額（派生状態）
@riverpod
int cartTotal(CartTotalRef ref) {
  final items = ref.watch(cartItemsProvider).valueOrNull ?? [];
  return items.fold(0, (sum, item) => sum + item.totalPrice);
}

// ==========================================
// パターン4: 複雑な非同期状態（AsyncNotifier）
// ==========================================

/// ページネーション付きホテルリスト
@riverpod
class HotelList extends _$HotelList {
  static const _pageSize = 20;

  @override
  Future<PaginatedResult<Hotel>> build(HotelSearchCriteria criteria) async {
    // 検索実行
    return _fetchPage(0);
  }

  Future<PaginatedResult<Hotel>> _fetchPage(int page) async {
    final repository = ref.read(hotelRepositoryProvider);
    final result = await repository.searchHotels(
      arg.copyWith(page: page, limit: _pageSize),
    );

    return result.fold(
      (failure) => throw failure,
      (hotels) => PaginatedResult(
        items: hotels,
        currentPage: page,
        hasMore: hotels.length >= _pageSize,
        totalCount: null, // APIからの総数がない場合
      ),
    );
  }

  Future<void> loadMore() async {
    final currentState = state.valueOrNull;
    if (currentState == null || !currentState.hasMore) return;

    // ローディング状態を維持しながら追加読み込み
    state = AsyncValue.data(currentState.copyWith(isLoadingMore: true));

    try {
      final nextPage = await _fetchPage(currentState.currentPage + 1);
      state = AsyncValue.data(PaginatedResult(
        items: [...currentState.items, ...nextPage.items],
        currentPage: nextPage.currentPage,
        hasMore: nextPage.hasMore,
        totalCount: nextPage.totalCount,
      ));
    } catch (e, st) {
      state = AsyncValue.data(currentState.copyWith(isLoadingMore: false));
      // エラーをUIに通知（スナックバーなど）
      ref.read(snackbarProvider.notifier).show(
        SnackbarData.error(e.toString()),
      );
    }
  }

  Future<void> refresh() async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() => _fetchPage(0));
  }
}

// ==========================================
// パターン5: フォーム状態管理
// ==========================================

/// 予約フォーム状態
@riverpod
class BookingForm extends _$BookingForm {
  @override
  BookingFormState build() => const BookingFormState();

  void setGuestInfo(GuestInfo info) {
    state = state.copyWith(guestInfo: info);
  }

  void setPaymentMethod(PaymentMethod method) {
    state = state.copyWith(paymentMethod: method);
  }

  void setSpecialRequests(String requests) {
    state = state.copyWith(specialRequests: requests);
  }

  ValidationResult validate() {
    final errors = <String, String>{};

    if (state.guestInfo.name.isEmpty) {
      errors['name'] = '名前を入力してください';
    }
    if (!state.guestInfo.email.isValidEmail) {
      errors['email'] = '有効なメールアドレスを入力してください';
    }
    if (state.paymentMethod == null) {
      errors['payment'] = '支払い方法を選択してください';
    }

    return ValidationResult(
      isValid: errors.isEmpty,
      errors: errors,
    );
  }

  Future<Result<Booking>> submit() async {
    final validation = validate();
    if (!validation.isValid) {
      return Result.failure(ValidationError(validation.errors));
    }

    state = state.copyWith(isSubmitting: true);

    final useCase = ref.read(createBookingUseCaseProvider);
    final result = await useCase.execute(state.toBookingRequest());

    state = state.copyWith(isSubmitting: false);

    return result;
  }
}
```

### 3.2 グローバル状態 vs ローカル状態

#### 3.2.1 状態スコープの決定基準

```yaml
state_scope_guidelines:
  global_state:
    description: アプリ全体で共有される状態
    criteria:
      - 複数の画面で使用される
      - アプリのライフサイクル全体で維持される
      - 認証状態のように一貫性が重要
    examples:
      - 認証状態（ユーザー情報、トークン）
      - アプリ設定（言語、テーマ、通貨）
      - カート状態
      - 通知設定
      - お気に入りリスト
    implementation: |
      @Riverpod(keepAlive: true)
      class AuthState extends _$AuthState { ... }

  feature_state:
    description: 特定の機能内で共有される状態
    criteria:
      - 機能内の複数画面で使用
      - 機能を離れると破棄可能
      - 機能間で独立
    examples:
      - ホテル検索フィルター
      - 予約フロー状態
      - チェックアウトプロセス
    implementation: |
      @riverpod
      class HotelSearchState extends _$HotelSearchState { ... }

  screen_state:
    description: 単一画面内でのみ使用される状態
    criteria:
      - 特定の画面でのみ使用
      - 画面遷移で破棄される
      - UIインタラクションに紐づく
    examples:
      - フォーム入力状態
      - モーダル表示状態
      - アニメーション状態
    implementation: |
      // useStateを使用（flutter_hooks）
      final expanded = useState(false);

      // または、AutoDisposeを活用
      @riverpod
      class ScreenState extends _$ScreenState { ... }
```

#### 3.2.2 状態の継承と合成

```dart
// ==========================================
// 状態の合成パターン
// ==========================================

/// 複数の状態を合成して派生状態を作成
@riverpod
BookingEligibility bookingEligibility(BookingEligibilityRef ref) {
  final user = ref.watch(authStateProvider).valueOrNull;
  final hotel = ref.watch(selectedHotelProvider);
  final dates = ref.watch(searchFilterProvider);

  // 認証チェック
  if (user == null) {
    return const BookingEligibility.notAuthenticated();
  }

  // ホテル選択チェック
  if (hotel == null) {
    return const BookingEligibility.noHotelSelected();
  }

  // 日付チェック
  if (dates.checkIn == null || dates.checkOut == null) {
    return const BookingEligibility.noDatesSelected();
  }

  // 在庫チェック
  final availability = ref.watch(
    hotelAvailabilityProvider(hotel.id, dates.checkIn!, dates.checkOut!),
  );

  return availability.when(
    data: (available) => available
        ? const BookingEligibility.eligible()
        : const BookingEligibility.notAvailable(),
    loading: () => const BookingEligibility.checking(),
    error: (_, __) => const BookingEligibility.error(),
  );
}

/// UIでの使用
class BookingButton extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final eligibility = ref.watch(bookingEligibilityProvider);

    return ElevatedButton(
      onPressed: eligibility.isEligible
          ? () => _startBooking(context, ref)
          : null,
      child: Text(eligibility.buttonText),
    );
  }
}

// ==========================================
// 状態の継承パターン（Family）
// ==========================================

/// パラメータ化されたプロバイダー
@riverpod
Future<HotelAvailability> hotelAvailability(
  HotelAvailabilityRef ref,
  String hotelId,
  DateTime checkIn,
  DateTime checkOut,
) async {
  final repository = ref.watch(hotelRepositoryProvider);
  final result = await repository.checkAvailability(
    hotelId: hotelId,
    checkIn: checkIn,
    checkOut: checkOut,
  );

  return result.fold(
    (failure) => throw failure,
    (availability) => availability,
  );
}

/// 使用例
class HotelAvailabilityWidget extends ConsumerWidget {
  final String hotelId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final filter = ref.watch(searchFilterProvider);

    if (filter.checkIn == null || filter.checkOut == null) {
      return const Text('日付を選択してください');
    }

    final availability = ref.watch(
      hotelAvailabilityProvider(hotelId, filter.checkIn!, filter.checkOut!),
    );

    return availability.when(
      data: (data) => AvailabilityDisplay(availability: data),
      loading: () => const AvailabilitySkeleton(),
      error: (e, _) => AvailabilityError(error: e),
    );
  }
}
```

### 3.3 リアクティブプログラミング

#### 3.3.1 リアクティブデータフロー

```dart
// ==========================================
// リアクティブな検索フロー
// ==========================================

/// デバウンス付き検索クエリ
@riverpod
class SearchQuery extends _$SearchQuery {
  Timer? _debounceTimer;

  @override
  String build() => '';

  void setQuery(String query) {
    _debounceTimer?.cancel();

    // 即時更新（UIの応答性）
    state = query;

    // デバウンス後に検索実行
    _debounceTimer = Timer(const Duration(milliseconds: 300), () {
      if (query.length >= 2) {
        ref.invalidate(searchSuggestionsProvider);
      }
    });
  }

  @override
  void dispose() {
    _debounceTimer?.cancel();
    super.dispose();
  }
}

/// 検索サジェスト（クエリに反応）
@riverpod
Future<List<SearchSuggestion>> searchSuggestions(
  SearchSuggestionsRef ref,
) async {
  final query = ref.watch(searchQueryProvider);

  if (query.length < 2) return [];

  final repository = ref.watch(searchRepositoryProvider);
  final result = await repository.getSuggestions(query);

  return result.fold(
    (failure) => [],
    (suggestions) => suggestions,
  );
}

// ==========================================
// リアクティブなカート計算
// ==========================================

/// カートアイテムストリーム
@riverpod
Stream<List<CartItem>> cartItemsStream(CartItemsStreamRef ref) {
  final repository = ref.watch(cartRepositoryProvider);
  return repository.watchCartItems();
}

/// カート合計（リアクティブ計算）
@riverpod
CartSummary cartSummary(CartSummaryRef ref) {
  final items = ref.watch(cartItemsStreamProvider).valueOrNull ?? [];
  final currency = ref.watch(currencyProvider);
  final exchangeRate = ref.watch(exchangeRateProvider(currency)).valueOrNull;

  final subtotal = items.fold(0, (sum, item) => sum + item.totalPrice);
  final tax = (subtotal * 0.1).round(); // 10%消費税
  final total = subtotal + tax;

  return CartSummary(
    itemCount: items.length,
    subtotal: subtotal,
    tax: tax,
    total: total,
    displayTotal: exchangeRate != null
        ? (total * exchangeRate).round()
        : total,
    currency: currency,
  );
}

// ==========================================
// リアクティブなバリデーション
// ==========================================

/// フォームフィールドのリアクティブバリデーション
@riverpod
class EmailField extends _$EmailField {
  @override
  FieldState<String> build() => const FieldState(value: '', error: null);

  void setValue(String value) {
    final error = _validate(value);
    state = FieldState(value: value, error: error);
  }

  String? _validate(String value) {
    if (value.isEmpty) return null; // 空の場合はエラーなし（任意入力時）
    if (!value.isValidEmail) return '有効なメールアドレスを入力してください';
    return null;
  }

  void touch() {
    if (state.value.isEmpty) {
      state = state.copyWith(error: 'メールアドレスを入力してください');
    }
  }
}

/// フォーム全体のバリデーション状態
@riverpod
bool isFormValid(IsFormValidRef ref) {
  final email = ref.watch(emailFieldProvider);
  final password = ref.watch(passwordFieldProvider);
  final name = ref.watch(nameFieldProvider);

  return email.isValid && password.isValid && name.isValid;
}

/// 使用例：送信ボタン
class SubmitButton extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final isValid = ref.watch(isFormValidProvider);
    final isSubmitting = ref.watch(formSubmitStateProvider);

    return ElevatedButton(
      onPressed: isValid && !isSubmitting
          ? () => ref.read(formSubmitStateProvider.notifier).submit()
          : null,
      child: isSubmitting
          ? const CircularProgressIndicator()
          : const Text('送信'),
    );
  }
}
```

---

## 第4章：データ層設計

### 4.1 ローカルストレージ（Hive、SharedPreferences）

#### 4.1.1 ストレージ戦略

```yaml
storage_strategy:
  hive:
    description: 高性能な構造化データストレージ
    use_cases:
      - カートアイテム
      - 検索キャッシュ
      - お気に入りリスト
      - オフラインデータ
    characteristics:
      - 高速な読み書き
      - 構造化データのサポート
      - 暗号化オプション
      - カスタムアダプター対応

  shared_preferences:
    description: シンプルなキー値ストレージ
    use_cases:
      - ユーザー設定
      - 言語・通貨設定
      - 機能フラグ
      - 簡単なフラグ
    characteristics:
      - シンプルなAPI
      - プラットフォームネイティブ
      - 少量データ向け

  secure_storage:
    description: 機密データの暗号化ストレージ
    use_cases:
      - 認証トークン
      - APIキー
      - センシティブなユーザーデータ
    characteristics:
      - ハードウェア暗号化
      - キーチェーン/Keystore統合
      - 生体認証連携可能

  sqlite:
    description: リレーショナルデータベース
    use_cases:
      - 複雑なクエリが必要なデータ
      - 大量データの管理
      - オフラインファーストアプリ
    characteristics:
      - SQLクエリサポート
      - ACID準拠
      - インデックス・リレーション
```

#### 4.1.2 Hive実装

```dart
// ==========================================
// core/storage/hive_storage.dart
// ==========================================

/// Hive初期化と設定
class HiveStorageService {
  static Future<void> initialize() async {
    await Hive.initFlutter();

    // アダプター登録
    _registerAdapters();

    // Boxのオープン
    await _openBoxes();
  }

  static void _registerAdapters() {
    Hive.registerAdapter(CartItemAdapter());
    Hive.registerAdapter(HotelCacheAdapter());
    Hive.registerAdapter(SearchHistoryAdapter());
    Hive.registerAdapter(FavoriteHotelAdapter());
  }

  static Future<void> _openBoxes() async {
    await Future.wait([
      Hive.openBox<CartItem>(HiveBoxes.cart),
      Hive.openBox<HotelCache>(HiveBoxes.hotelCache),
      Hive.openBox<SearchHistory>(HiveBoxes.searchHistory),
      Hive.openBox<FavoriteHotel>(HiveBoxes.favorites),
    ]);
  }

  static Future<void> clearAll() async {
    await Future.wait([
      Hive.box<CartItem>(HiveBoxes.cart).clear(),
      Hive.box<HotelCache>(HiveBoxes.hotelCache).clear(),
      Hive.box<SearchHistory>(HiveBoxes.searchHistory).clear(),
    ]);
  }
}

abstract class HiveBoxes {
  static const cart = 'cart';
  static const hotelCache = 'hotel_cache';
  static const searchHistory = 'search_history';
  static const favorites = 'favorites';
}

// ==========================================
// Hiveモデルとアダプター
// ==========================================

/// カートアイテムモデル
@HiveType(typeId: 0)
class CartItem extends HiveObject {
  @HiveField(0)
  final String id;

  @HiveField(1)
  final String productId;

  @HiveField(2)
  final String productName;

  @HiveField(3)
  final String? variant;

  @HiveField(4)
  final int quantity;

  @HiveField(5)
  final int unitPrice;

  @HiveField(6)
  final String imageUrl;

  @HiveField(7)
  final DateTime addedAt;

  CartItem({
    required this.id,
    required this.productId,
    required this.productName,
    this.variant,
    required this.quantity,
    required this.unitPrice,
    required this.imageUrl,
    required this.addedAt,
  });

  int get totalPrice => quantity * unitPrice;

  CartItem copyWith({int? quantity}) {
    return CartItem(
      id: id,
      productId: productId,
      productName: productName,
      variant: variant,
      quantity: quantity ?? this.quantity,
      unitPrice: unitPrice,
      imageUrl: imageUrl,
      addedAt: addedAt,
    );
  }
}

/// カートリポジトリ実装
class CartLocalDataSource {
  final Box<CartItem> _cartBox;

  CartLocalDataSource(this._cartBox);

  /// カートアイテムのストリーム監視
  Stream<List<CartItem>> watchCartItems() {
    return _cartBox.watch().map((_) => _cartBox.values.toList());
  }

  /// カートアイテム追加
  Future<void> addItem(CartItem item) async {
    final existingIndex = _cartBox.values.toList().indexWhere(
      (i) => i.productId == item.productId && i.variant == item.variant,
    );

    if (existingIndex >= 0) {
      // 既存アイテムの数量を更新
      final existing = _cartBox.getAt(existingIndex)!;
      await _cartBox.putAt(
        existingIndex,
        existing.copyWith(quantity: existing.quantity + item.quantity),
      );
    } else {
      // 新規アイテム追加
      await _cartBox.put(item.id, item);
    }
  }

  /// 数量更新
  Future<void> updateQuantity(String itemId, int quantity) async {
    final item = _cartBox.get(itemId);
    if (item != null) {
      if (quantity <= 0) {
        await _cartBox.delete(itemId);
      } else {
        await _cartBox.put(itemId, item.copyWith(quantity: quantity));
      }
    }
  }

  /// アイテム削除
  Future<void> removeItem(String itemId) async {
    await _cartBox.delete(itemId);
  }

  /// カートクリア
  Future<void> clearCart() async {
    await _cartBox.clear();
  }

  /// 全アイテム取得
  List<CartItem> getAllItems() {
    return _cartBox.values.toList();
  }
}
```

### 4.2 オフライン同期メカニズム

#### 4.2.1 同期戦略

```yaml
sync_strategy:
  approach: Offline-First with Background Sync

  data_classification:
    read_only:
      description: 読み取り専用データ（キャッシュのみ）
      examples:
        - ホテル情報
        - 商品カタログ
        - 検索結果
      strategy: Cache with TTL
      ttl: 1 hour - 24 hours

    read_write:
      description: ユーザー操作で変更されるデータ
      examples:
        - カート
        - お気に入り
        - 予約ドラフト
      strategy: Optimistic Update with Sync Queue
      conflict_resolution: Last-Write-Wins / Server-Authority

    critical:
      description: 即座にサーバー同期が必要なデータ
      examples:
        - 予約確定
        - 決済
        - アカウント変更
      strategy: Online-Only with Retry

  sync_triggers:
    - アプリ起動時
    - ネットワーク復帰時
    - バックグラウンドフェッチ
    - プルリフレッシュ
    - 一定間隔（ポーリング）
```

#### 4.2.2 同期実装

```dart
// ==========================================
// core/sync/sync_service.dart
// ==========================================

/// 同期サービス
class SyncService {
  final NetworkInfo _networkInfo;
  final SyncQueue _syncQueue;
  final List<Syncable> _syncables;

  SyncService({
    required NetworkInfo networkInfo,
    required SyncQueue syncQueue,
    required List<Syncable> syncables,
  }) : _networkInfo = networkInfo,
       _syncQueue = syncQueue,
       _syncables = syncables;

  /// 完全同期の実行
  Future<SyncResult> performFullSync() async {
    if (!await _networkInfo.isConnected) {
      return SyncResult.offline();
    }

    final results = <String, SyncStatus>{};

    // 各同期対象を順番に処理
    for (final syncable in _syncables) {
      try {
        await syncable.sync();
        results[syncable.name] = SyncStatus.success;
      } catch (e) {
        results[syncable.name] = SyncStatus.failed;
      }
    }

    // キュー内の保留中の変更を処理
    await _processSyncQueue();

    return SyncResult(
      timestamp: DateTime.now(),
      results: results,
    );
  }

  /// 同期キューの処理
  Future<void> _processSyncQueue() async {
    final pendingItems = await _syncQueue.getPendingItems();

    for (final item in pendingItems) {
      try {
        await item.execute();
        await _syncQueue.markCompleted(item.id);
      } catch (e) {
        if (item.retryCount >= 3) {
          await _syncQueue.markFailed(item.id);
        } else {
          await _syncQueue.incrementRetry(item.id);
        }
      }
    }
  }

  /// ネットワーク状態変化の監視
  Stream<ConnectivityResult> get connectivityStream =>
      Connectivity().onConnectivityChanged;

  void startBackgroundSync() {
    connectivityStream.listen((result) {
      if (result != ConnectivityResult.none) {
        _processSyncQueue();
      }
    });
  }
}

/// 同期キュー
class SyncQueue {
  final Box<SyncQueueItem> _queueBox;

  SyncQueue(this._queueBox);

  Future<void> enqueue(SyncQueueItem item) async {
    await _queueBox.put(item.id, item);
  }

  Future<List<SyncQueueItem>> getPendingItems() async {
    return _queueBox.values
        .where((item) => item.status == SyncItemStatus.pending)
        .toList()
      ..sort((a, b) => a.createdAt.compareTo(b.createdAt));
  }

  Future<void> markCompleted(String id) async {
    await _queueBox.delete(id);
  }

  Future<void> markFailed(String id) async {
    final item = _queueBox.get(id);
    if (item != null) {
      await _queueBox.put(id, item.copyWith(status: SyncItemStatus.failed));
    }
  }

  Future<void> incrementRetry(String id) async {
    final item = _queueBox.get(id);
    if (item != null) {
      await _queueBox.put(id, item.copyWith(retryCount: item.retryCount + 1));
    }
  }
}

/// 同期キューアイテム
@HiveType(typeId: 10)
class SyncQueueItem extends HiveObject {
  @HiveField(0)
  final String id;

  @HiveField(1)
  final SyncOperation operation;

  @HiveField(2)
  final String entityType;

  @HiveField(3)
  final String entityId;

  @HiveField(4)
  final Map<String, dynamic> payload;

  @HiveField(5)
  final DateTime createdAt;

  @HiveField(6)
  final SyncItemStatus status;

  @HiveField(7)
  final int retryCount;

  SyncQueueItem({
    required this.id,
    required this.operation,
    required this.entityType,
    required this.entityId,
    required this.payload,
    required this.createdAt,
    this.status = SyncItemStatus.pending,
    this.retryCount = 0,
  });

  Future<void> execute() async {
    // 操作タイプに応じた処理
    switch (operation) {
      case SyncOperation.create:
        await _executeCreate();
        break;
      case SyncOperation.update:
        await _executeUpdate();
        break;
      case SyncOperation.delete:
        await _executeDelete();
        break;
    }
  }
}

// ==========================================
// オフラインファースト リポジトリパターン
// ==========================================

/// お気に入りリポジトリ（オフラインファースト）
class FavoriteRepositoryImpl implements FavoriteRepository {
  final FavoriteRemoteDataSource _remote;
  final FavoriteLocalDataSource _local;
  final SyncQueue _syncQueue;
  final NetworkInfo _networkInfo;

  @override
  Stream<List<Hotel>> watchFavorites() {
    // ローカルデータを優先的に返す
    return _local.watchFavorites().map(
      (favorites) => favorites.map((f) => f.toHotel()).toList(),
    );
  }

  @override
  Future<Result<void>> addFavorite(Hotel hotel) async {
    // 1. ローカルに即座に保存（楽観的更新）
    await _local.addFavorite(FavoriteHotel.fromHotel(hotel));

    // 2. オンラインならサーバーに同期
    if (await _networkInfo.isConnected) {
      try {
        await _remote.addFavorite(hotel.id);
        return const Result.success(null);
      } catch (e) {
        // サーバー同期失敗時はキューに追加
        await _syncQueue.enqueue(SyncQueueItem(
          id: const Uuid().v4(),
          operation: SyncOperation.create,
          entityType: 'favorite',
          entityId: hotel.id,
          payload: {'hotelId': hotel.id},
          createdAt: DateTime.now(),
        ));
        return const Result.success(null); // ローカル保存は成功
      }
    } else {
      // オフライン時はキューに追加
      await _syncQueue.enqueue(SyncQueueItem(
        id: const Uuid().v4(),
        operation: SyncOperation.create,
        entityType: 'favorite',
        entityId: hotel.id,
        payload: {'hotelId': hotel.id},
        createdAt: DateTime.now(),
      ));
      return const Result.success(null);
    }
  }

  @override
  Future<Result<void>> removeFavorite(String hotelId) async {
    // ローカルから即座に削除
    await _local.removeFavorite(hotelId);

    if (await _networkInfo.isConnected) {
      try {
        await _remote.removeFavorite(hotelId);
      } catch (e) {
        await _syncQueue.enqueue(SyncQueueItem(
          id: const Uuid().v4(),
          operation: SyncOperation.delete,
          entityType: 'favorite',
          entityId: hotelId,
          payload: {'hotelId': hotelId},
          createdAt: DateTime.now(),
        ));
      }
    } else {
      await _syncQueue.enqueue(SyncQueueItem(
        id: const Uuid().v4(),
        operation: SyncOperation.delete,
        entityType: 'favorite',
        entityId: hotelId,
        payload: {'hotelId': hotelId},
        createdAt: DateTime.now(),
      ));
    }

    return const Result.success(null);
  }

  @override
  Future<void> syncWithServer() async {
    if (!await _networkInfo.isConnected) return;

    try {
      // サーバーから最新のお気に入りリストを取得
      final serverFavorites = await _remote.getFavorites();

      // ローカルと同期
      await _local.replaceAll(serverFavorites);
    } catch (e) {
      // 同期失敗時は何もしない（ローカルデータを維持）
    }
  }
}
```

### 4.3 キャッシュ戦略

#### 4.3.1 多層キャッシュアーキテクチャ

```dart
// ==========================================
// core/cache/cache_manager.dart
// ==========================================

/// キャッシュマネージャー
class CacheManager {
  final MemoryCache _memoryCache;
  final DiskCache _diskCache;

  CacheManager({
    required MemoryCache memoryCache,
    required DiskCache diskCache,
  }) : _memoryCache = memoryCache,
       _diskCache = diskCache;

  /// キャッシュからの取得（多層）
  Future<T?> get<T>(
    String key, {
    CachePolicy policy = CachePolicy.cacheFirst,
  }) async {
    switch (policy) {
      case CachePolicy.cacheFirst:
        return await _getCacheFirst<T>(key);
      case CachePolicy.cacheOnly:
        return await _getCacheOnly<T>(key);
      case CachePolicy.networkFirst:
        return null; // ネットワーク優先の場合はキャッシュを返さない
      case CachePolicy.networkOnly:
        return null;
    }
  }

  Future<T?> _getCacheFirst<T>(String key) async {
    // 1. メモリキャッシュを確認
    final memoryResult = _memoryCache.get<T>(key);
    if (memoryResult != null) return memoryResult;

    // 2. ディスクキャッシュを確認
    final diskResult = await _diskCache.get<T>(key);
    if (diskResult != null) {
      // メモリキャッシュに昇格
      _memoryCache.put(key, diskResult);
      return diskResult;
    }

    return null;
  }

  Future<T?> _getCacheOnly<T>(String key) async {
    return await _getCacheFirst<T>(key);
  }

  /// キャッシュへの保存
  Future<void> put<T>(
    String key,
    T value, {
    Duration? ttl,
    CacheLevel level = CacheLevel.both,
  }) async {
    final effectiveTtl = ttl ?? const Duration(hours: 1);

    if (level == CacheLevel.memory || level == CacheLevel.both) {
      _memoryCache.put(key, value, ttl: effectiveTtl);
    }

    if (level == CacheLevel.disk || level == CacheLevel.both) {
      await _diskCache.put(key, value, ttl: effectiveTtl);
    }
  }

  /// キャッシュの無効化
  Future<void> invalidate(String key) async {
    _memoryCache.remove(key);
    await _diskCache.remove(key);
  }

  /// パターンマッチによる無効化
  Future<void> invalidateByPattern(String pattern) async {
    final regex = RegExp(pattern);
    _memoryCache.removeWhere((key) => regex.hasMatch(key));
    await _diskCache.removeWhere((key) => regex.hasMatch(key));
  }

  /// 全キャッシュクリア
  Future<void> clearAll() async {
    _memoryCache.clear();
    await _diskCache.clear();
  }
}

/// メモリキャッシュ（LRU）
class MemoryCache {
  final int maxSize;
  final LinkedHashMap<String, CacheEntry> _cache = LinkedHashMap();

  MemoryCache({this.maxSize = 100});

  T? get<T>(String key) {
    final entry = _cache[key];
    if (entry == null) return null;

    if (entry.isExpired) {
      _cache.remove(key);
      return null;
    }

    // LRU: アクセスされたアイテムを末尾に移動
    _cache.remove(key);
    _cache[key] = entry;

    return entry.value as T;
  }

  void put<T>(String key, T value, {required Duration ttl}) {
    // サイズ制限チェック
    if (_cache.length >= maxSize) {
      _cache.remove(_cache.keys.first);
    }

    _cache[key] = CacheEntry(
      value: value,
      expiresAt: DateTime.now().add(ttl),
    );
  }

  void remove(String key) => _cache.remove(key);

  void removeWhere(bool Function(String key) test) {
    _cache.removeWhere((key, _) => test(key));
  }

  void clear() => _cache.clear();
}

/// ディスクキャッシュ（Hive基盤）
class DiskCache {
  final Box<CacheEntry> _cacheBox;

  DiskCache(this._cacheBox);

  Future<T?> get<T>(String key) async {
    final entry = _cacheBox.get(key);
    if (entry == null) return null;

    if (entry.isExpired) {
      await _cacheBox.delete(key);
      return null;
    }

    return entry.value as T;
  }

  Future<void> put<T>(String key, T value, {required Duration ttl}) async {
    await _cacheBox.put(key, CacheEntry(
      value: value,
      expiresAt: DateTime.now().add(ttl),
    ));
  }

  Future<void> remove(String key) async {
    await _cacheBox.delete(key);
  }

  Future<void> removeWhere(bool Function(String key) test) async {
    final keysToRemove = _cacheBox.keys.where((key) => test(key as String));
    await _cacheBox.deleteAll(keysToRemove);
  }

  Future<void> clear() async {
    await _cacheBox.clear();
  }

  /// 期限切れエントリのクリーンアップ
  Future<void> cleanupExpired() async {
    final expiredKeys = _cacheBox.keys.where((key) {
      final entry = _cacheBox.get(key);
      return entry?.isExpired ?? true;
    });
    await _cacheBox.deleteAll(expiredKeys);
  }
}

/// キャッシュエントリ
@HiveType(typeId: 20)
class CacheEntry extends HiveObject {
  @HiveField(0)
  final dynamic value;

  @HiveField(1)
  final DateTime expiresAt;

  CacheEntry({required this.value, required this.expiresAt});

  bool get isExpired => DateTime.now().isAfter(expiresAt);
}

enum CachePolicy {
  cacheFirst,   // キャッシュ優先、なければネットワーク
  cacheOnly,    // キャッシュのみ
  networkFirst, // ネットワーク優先、失敗時キャッシュ
  networkOnly,  // ネットワークのみ
}

enum CacheLevel {
  memory,
  disk,
  both,
}
```

---

## 第5章：プラットフォーム統合

### 5.1 ネイティブ機能連携（カメラ、GPS、プッシュ通知）

#### 5.1.1 プラットフォームチャネル設計

```dart
// ==========================================
// core/platform/platform_service.dart
// ==========================================

/// プラットフォームサービスの抽象化
abstract class PlatformService {
  /// カメラ関連
  Future<XFile?> takePhoto();
  Future<XFile?> pickImageFromGallery();
  Future<bool> requestCameraPermission();

  /// 位置情報関連
  Future<Position?> getCurrentLocation();
  Future<bool> requestLocationPermission();
  Stream<Position> watchLocation();

  /// 通知関連
  Future<bool> requestNotificationPermission();
  Future<String?> getNotificationToken();
  Stream<NotificationMessage> onNotificationReceived();

  /// その他
  Future<void> openAppSettings();
  Future<bool> canLaunchUrl(String url);
  Future<bool> launchUrl(String url);
}

/// プラットフォームサービス実装
class PlatformServiceImpl implements PlatformService {
  final ImagePicker _imagePicker;
  final Geolocator _geolocator;
  final FirebaseMessaging _messaging;

  PlatformServiceImpl({
    required ImagePicker imagePicker,
    required FirebaseMessaging messaging,
  }) : _imagePicker = imagePicker,
       _geolocator = Geolocator(),
       _messaging = messaging;

  // ==========================================
  // カメラ・画像
  // ==========================================

  @override
  Future<XFile?> takePhoto() async {
    final hasPermission = await requestCameraPermission();
    if (!hasPermission) return null;

    return await _imagePicker.pickImage(
      source: ImageSource.camera,
      maxWidth: 1920,
      maxHeight: 1080,
      imageQuality: 85,
    );
  }

  @override
  Future<XFile?> pickImageFromGallery() async {
    return await _imagePicker.pickImage(
      source: ImageSource.gallery,
      maxWidth: 1920,
      maxHeight: 1080,
      imageQuality: 85,
    );
  }

  @override
  Future<bool> requestCameraPermission() async {
    final status = await Permission.camera.request();
    return status.isGranted;
  }

  // ==========================================
  // 位置情報
  // ==========================================

  @override
  Future<Position?> getCurrentLocation() async {
    final hasPermission = await requestLocationPermission();
    if (!hasPermission) return null;

    final serviceEnabled = await Geolocator.isLocationServiceEnabled();
    if (!serviceEnabled) return null;

    try {
      return await Geolocator.getCurrentPosition(
        desiredAccuracy: LocationAccuracy.high,
        timeLimit: const Duration(seconds: 10),
      );
    } catch (e) {
      return null;
    }
  }

  @override
  Future<bool> requestLocationPermission() async {
    var permission = await Geolocator.checkPermission();

    if (permission == LocationPermission.denied) {
      permission = await Geolocator.requestPermission();
    }

    return permission == LocationPermission.always ||
           permission == LocationPermission.whileInUse;
  }

  @override
  Stream<Position> watchLocation() {
    return Geolocator.getPositionStream(
      locationSettings: const LocationSettings(
        accuracy: LocationAccuracy.high,
        distanceFilter: 10, // 10m移動で更新
      ),
    );
  }

  // ==========================================
  // 通知
  // ==========================================

  @override
  Future<bool> requestNotificationPermission() async {
    final settings = await _messaging.requestPermission(
      alert: true,
      badge: true,
      sound: true,
      provisional: false,
    );

    return settings.authorizationStatus == AuthorizationStatus.authorized;
  }

  @override
  Future<String?> getNotificationToken() async {
    try {
      return await _messaging.getToken();
    } catch (e) {
      return null;
    }
  }

  @override
  Stream<NotificationMessage> onNotificationReceived() {
    return FirebaseMessaging.onMessage.map((message) {
      return NotificationMessage(
        title: message.notification?.title ?? '',
        body: message.notification?.body ?? '',
        data: message.data,
      );
    });
  }

  // ==========================================
  // その他
  // ==========================================

  @override
  Future<void> openAppSettings() async {
    await openAppSettings();
  }

  @override
  Future<bool> canLaunchUrl(String url) async {
    return await canLaunch(url);
  }

  @override
  Future<bool> launchUrl(String url) async {
    return await launch(url);
  }
}
```

### 5.2 ディープリンク・ユニバーサルリンク

#### 5.2.1 ディープリンク設計

```yaml
deep_link_schema:
  app_links:
    base_url: https://triptrip.com
    paths:
      - pattern: /hotels/:hotelId
        route: HotelDetailRoute
        params:
          - hotelId: String

      - pattern: /hotels/:hotelId/book
        route: HotelBookingRoute
        params:
          - hotelId: String

      - pattern: /attractions/:attractionId
        route: AttractionDetailRoute
        params:
          - attractionId: String

      - pattern: /products/:productId
        route: ProductDetailRoute
        params:
          - productId: String

      - pattern: /bookings/:bookingId
        route: BookingDetailRoute
        params:
          - bookingId: String

      - pattern: /auth/verify-email
        route: EmailVerificationRoute
        query_params:
          - token: String

      - pattern: /auth/reset-password
        route: PasswordResetRoute
        query_params:
          - token: String

  custom_scheme:
    scheme: triptrip
    paths:
      - pattern: ://share/hotel/:hotelId
        route: HotelDetailRoute
      - pattern: ://qr/:ticketId
        route: TicketDetailRoute
```

#### 5.2.2 ディープリンク実装

```dart
// ==========================================
// app/router/deep_link_handler.dart
// ==========================================

/// ディープリンクハンドラー
class DeepLinkHandler {
  final GoRouter _router;
  StreamSubscription? _linkSubscription;

  DeepLinkHandler(this._router);

  /// 初期化（初回起動時のリンク処理）
  Future<void> initialize() async {
    // 初回起動時のリンクを処理
    final initialLink = await getInitialLink();
    if (initialLink != null) {
      _handleLink(initialLink);
    }

    // リンクストリームを監視
    _linkSubscription = linkStream.listen(
      _handleLink,
      onError: (error) {
        debugPrint('Deep link error: $error');
      },
    );
  }

  void dispose() {
    _linkSubscription?.cancel();
  }

  void _handleLink(String? link) {
    if (link == null) return;

    final uri = Uri.parse(link);
    final route = _parseRoute(uri);

    if (route != null) {
      _router.go(route);
    }
  }

  String? _parseRoute(Uri uri) {
    // アプリリンク（https://triptrip.com/...）
    if (uri.host == 'triptrip.com') {
      return uri.path + (uri.query.isNotEmpty ? '?${uri.query}' : '');
    }

    // カスタムスキーム（triptrip://...）
    if (uri.scheme == 'triptrip') {
      return _parseCustomScheme(uri);
    }

    return null;
  }

  String? _parseCustomScheme(Uri uri) {
    final path = uri.host + uri.path;

    // share/hotel/:hotelId → /hotels/:hotelId
    final shareHotelMatch = RegExp(r'^share/hotel/(.+)$').firstMatch(path);
    if (shareHotelMatch != null) {
      return '/hotels/${shareHotelMatch.group(1)}';
    }

    // qr/:ticketId → /tickets/:ticketId
    final qrMatch = RegExp(r'^qr/(.+)$').firstMatch(path);
    if (qrMatch != null) {
      return '/tickets/${qrMatch.group(1)}';
    }

    return null;
  }
}

// ==========================================
// app/router/app_router.dart
// ==========================================

/// アプリルーター設定
@riverpod
GoRouter appRouter(AppRouterRef ref) {
  final authState = ref.watch(authStateProvider);

  return GoRouter(
    initialLocation: '/',
    debugLogDiagnostics: kDebugMode,

    // リダイレクトロジック
    redirect: (context, state) {
      final isAuthenticated = authState.valueOrNull != null;
      final isAuthRoute = state.matchedLocation.startsWith('/auth');
      final isOnboarding = state.matchedLocation == '/onboarding';

      // 未認証で保護されたルートへアクセス
      if (!isAuthenticated && !isAuthRoute && !isOnboarding) {
        // 予約関連は認証必須
        if (state.matchedLocation.contains('/book') ||
            state.matchedLocation.startsWith('/bookings')) {
          return '/auth/login?redirect=${state.matchedLocation}';
        }
      }

      // 認証済みでログイン画面へアクセス
      if (isAuthenticated && isAuthRoute) {
        return '/';
      }

      return null;
    },

    // ルート定義
    routes: [
      // ホーム（ボトムナビゲーション）
      StatefulShellRoute.indexedStack(
        builder: (context, state, navigationShell) {
          return MainShell(navigationShell: navigationShell);
        },
        branches: [
          // ホームタブ
          StatefulShellBranch(
            routes: [
              GoRoute(
                path: '/',
                builder: (context, state) => const HomeScreen(),
                routes: [
                  // ホテル関連
                  GoRoute(
                    path: 'hotels',
                    builder: (context, state) => const HotelListScreen(),
                  ),
                  GoRoute(
                    path: 'hotels/:hotelId',
                    builder: (context, state) => HotelDetailScreen(
                      hotelId: state.pathParameters['hotelId']!,
                    ),
                    routes: [
                      GoRoute(
                        path: 'book',
                        builder: (context, state) => HotelBookingScreen(
                          hotelId: state.pathParameters['hotelId']!,
                        ),
                      ),
                    ],
                  ),
                  // アトラクション関連
                  GoRoute(
                    path: 'attractions/:attractionId',
                    builder: (context, state) => AttractionDetailScreen(
                      attractionId: state.pathParameters['attractionId']!,
                    ),
                  ),
                ],
              ),
            ],
          ),

          // 検索タブ
          StatefulShellBranch(
            routes: [
              GoRoute(
                path: '/search',
                builder: (context, state) => const SearchScreen(),
              ),
            ],
          ),

          // カートタブ
          StatefulShellBranch(
            routes: [
              GoRoute(
                path: '/cart',
                builder: (context, state) => const CartScreen(),
                routes: [
                  GoRoute(
                    path: 'checkout',
                    builder: (context, state) => const CheckoutScreen(),
                  ),
                ],
              ),
            ],
          ),

          // プロフィールタブ
          StatefulShellBranch(
            routes: [
              GoRoute(
                path: '/profile',
                builder: (context, state) => const ProfileScreen(),
                routes: [
                  GoRoute(
                    path: 'settings',
                    builder: (context, state) => const SettingsScreen(),
                  ),
                  GoRoute(
                    path: 'bookings',
                    builder: (context, state) => const BookingHistoryScreen(),
                  ),
                  GoRoute(
                    path: 'bookings/:bookingId',
                    builder: (context, state) => BookingDetailScreen(
                      bookingId: state.pathParameters['bookingId']!,
                    ),
                  ),
                ],
              ),
            ],
          ),
        ],
      ),

      // 認証関連（モーダル）
      GoRoute(
        path: '/auth/login',
        builder: (context, state) => LoginScreen(
          redirectTo: state.uri.queryParameters['redirect'],
        ),
      ),
      GoRoute(
        path: '/auth/register',
        builder: (context, state) => const RegisterScreen(),
      ),
      GoRoute(
        path: '/auth/verify-email',
        builder: (context, state) => EmailVerificationScreen(
          token: state.uri.queryParameters['token']!,
        ),
      ),
      GoRoute(
        path: '/auth/reset-password',
        builder: (context, state) => PasswordResetScreen(
          token: state.uri.queryParameters['token']!,
        ),
      ),
    ],

    // エラーハンドリング
    errorBuilder: (context, state) => ErrorScreen(
      error: state.error,
    ),
  );
}
```

### 5.3 アプリ内課金統合

#### 5.3.1 課金サービス設計

```dart
// ==========================================
// features/payment/data/services/in_app_purchase_service.dart
// ==========================================

/// アプリ内課金サービス
class InAppPurchaseService {
  final InAppPurchase _iap = InAppPurchase.instance;
  StreamSubscription<List<PurchaseDetails>>? _subscription;

  final _purchaseUpdates = StreamController<PurchaseUpdate>.broadcast();
  Stream<PurchaseUpdate> get purchaseUpdates => _purchaseUpdates.stream;

  /// 初期化
  Future<bool> initialize() async {
    final available = await _iap.isAvailable();
    if (!available) return false;

    // 購入ストリームを監視
    _subscription = _iap.purchaseStream.listen(
      _handlePurchaseUpdates,
      onError: (error) {
        _purchaseUpdates.add(PurchaseUpdate.error(error.toString()));
      },
    );

    // 保留中の購入を復元
    await _iap.restorePurchases();

    return true;
  }

  void dispose() {
    _subscription?.cancel();
    _purchaseUpdates.close();
  }

  /// 商品情報の取得
  Future<List<ProductDetails>> getProducts(Set<String> productIds) async {
    final response = await _iap.queryProductDetails(productIds);

    if (response.notFoundIDs.isNotEmpty) {
      debugPrint('Products not found: ${response.notFoundIDs}');
    }

    return response.productDetails;
  }

  /// 購入処理
  Future<bool> purchase(ProductDetails product) async {
    final purchaseParam = PurchaseParam(productDetails: product);

    if (product.id.contains('subscription')) {
      return await _iap.buyNonConsumable(purchaseParam: purchaseParam);
    } else {
      return await _iap.buyConsumable(purchaseParam: purchaseParam);
    }
  }

  /// 購入の復元
  Future<void> restorePurchases() async {
    await _iap.restorePurchases();
  }

  void _handlePurchaseUpdates(List<PurchaseDetails> purchases) {
    for (final purchase in purchases) {
      switch (purchase.status) {
        case PurchaseStatus.pending:
          _purchaseUpdates.add(PurchaseUpdate.pending(purchase.productID));
          break;

        case PurchaseStatus.purchased:
        case PurchaseStatus.restored:
          _verifyAndDeliver(purchase);
          break;

        case PurchaseStatus.error:
          _purchaseUpdates.add(PurchaseUpdate.error(
            purchase.error?.message ?? 'Purchase failed',
          ));
          if (purchase.pendingCompletePurchase) {
            _iap.completePurchase(purchase);
          }
          break;

        case PurchaseStatus.canceled:
          _purchaseUpdates.add(PurchaseUpdate.canceled(purchase.productID));
          break;
      }
    }
  }

  Future<void> _verifyAndDeliver(PurchaseDetails purchase) async {
    try {
      // サーバーでレシート検証
      final verified = await _verifyPurchaseOnServer(purchase);

      if (verified) {
        _purchaseUpdates.add(PurchaseUpdate.completed(purchase.productID));
      } else {
        _purchaseUpdates.add(PurchaseUpdate.error('Verification failed'));
      }

      // 購入を完了としてマーク
      if (purchase.pendingCompletePurchase) {
        await _iap.completePurchase(purchase);
      }
    } catch (e) {
      _purchaseUpdates.add(PurchaseUpdate.error(e.toString()));
    }
  }

  Future<bool> _verifyPurchaseOnServer(PurchaseDetails purchase) async {
    // サーバーAPIでレシート検証
    // iOS: verificationData.serverVerificationData
    // Android: verificationData.localVerificationData

    // 実装は決済サービスと連携
    return true;
  }
}

/// 購入更新イベント
@freezed
class PurchaseUpdate with _$PurchaseUpdate {
  const factory PurchaseUpdate.pending(String productId) = PurchasePending;
  const factory PurchaseUpdate.completed(String productId) = PurchaseCompleted;
  const factory PurchaseUpdate.canceled(String productId) = PurchaseCanceled;
  const factory PurchaseUpdate.error(String message) = PurchaseError;
}
```

---

## 第6章：実装ロードマップ & 文書間参照

### 6.1 実装フェーズ

```yaml
implementation_phases:
  phase_1:
    name: アーキテクチャ基盤
    duration: 4週間
    deliverables:
      - Clean Architectureフォルダ構造の確立
      - Riverpod DIコンテナの設定
      - ネットワーク層の実装（Dio + インターセプター）
      - ローカルストレージ基盤（Hive設定）
      - エラーハンドリングフレームワーク
    dependencies: []

  phase_2:
    name: コア機能移行
    duration: 6週間
    deliverables:
      - 認証機能のClean Architecture化
      - ホテル機能のClean Architecture化
      - 商品機能のClean Architecture化
      - 状態管理のRiverpod移行
    dependencies: [phase_1]

  phase_3:
    name: オフライン機能
    duration: 4週間
    deliverables:
      - オフラインファーストリポジトリパターン実装
      - 同期キューの実装
      - キャッシュマネージャーの実装
      - ネットワーク状態監視
    dependencies: [phase_2]

  phase_4:
    name: プラットフォーム統合
    duration: 3週間
    deliverables:
      - ディープリンク実装
      - プッシュ通知統合
      - ネイティブ機能連携
      - アプリ内課金統合
    dependencies: [phase_2]

  phase_5:
    name: パフォーマンス最適化
    duration: 3週間
    deliverables:
      - 起動時間最適化
      - メモリ使用量最適化
      - 画像最適化
      - アニメーション最適化
    dependencies: [phase_3, phase_4]

  phase_6:
    name: テスト・CI/CD
    duration: 4週間
    deliverables:
      - ユニットテストカバレッジ80%達成
      - ウィジェットテスト実装
      - 統合テスト実装
      - CI/CDパイプライン完成
    dependencies: [phase_5]
```

### 6.2 文書間参照

```yaml
document_references:
  related_documents:
    - document: Doc-TV-001
      relationship: 技術ビジョンと原則の準拠
      sections:
        - 技術原則
        - パフォーマンス目標
        - セキュリティ要件

    - document: Doc-SA-001
      relationship: システムアーキテクチャとの整合性
      sections:
        - APIゲートウェイ設計
        - 認証フロー
        - データモデル

    - document: Doc-SA-002
      relationship: マイクロサービス連携
      sections:
        - サービス境界
        - API仕様
        - イベント駆動設計

    - document: Doc-AD-002
      relationship: Webアプリとの一貫性
      sections:
        - 共通コンポーネント
        - 状態管理パターン
        - API統合

    - document: Doc-AD-003
      relationship: バックエンドAPI仕様
      sections:
        - エンドポイント定義
        - 認証方式
        - エラーレスポンス

  successor_documents:
    - Doc-QA-001: テスト戦略詳細
    - Doc-DevOps-001: CI/CDパイプライン設計
    - Doc-SEC-001: モバイルセキュリティ実装
```

### 6.3 成功指標

```yaml
success_metrics:
  technical_metrics:
    code_quality:
      - test_coverage: "> 80%"
      - lint_warnings: "0"
      - type_safety: "100%"

    performance:
      - cold_start_time: "< 2s"
      - frame_rate: ">= 60 FPS"
      - memory_baseline: "< 150 MB"
      - api_response_handling: "< 100ms"

    reliability:
      - crash_free_rate: "> 99.9%"
      - anr_rate: "< 0.1%"

  user_experience_metrics:
    app_store_rating: ">= 4.5"
    user_retention_d7: ">= 40%"
    session_length: ">= 5 minutes"

  development_metrics:
    build_time: "< 5 minutes"
    deployment_frequency: ">= 1/day"
    lead_time: "< 24 hours"
```

---

## 結論

本文書は、TripTripモバイルアプリケーションの包括的なアーキテクチャ設計を定義しました。Clean Architectureの原則に基づく堅牢な設計、Riverpodによる効率的な状態管理、オフラインファーストによるユーザー体験の向上を実現する技術基盤を確立しています。

主要な成果：
1. **Clean Architecture**: 保守性、テスト容易性、スケーラビリティを確保
2. **Riverpod状態管理**: リアクティブで型安全な状態管理
3. **オフラインファースト**: ネットワーク状態に関わらない一貫したユーザー体験
4. **プラットフォーム統合**: iOS/Android固有機能の効率的な活用
5. **テスト戦略**: 高品質を維持するための包括的なテスト基盤

この設計により、TripTripは数百万ユーザーに対応可能な、高品質でスケーラブルなモバイルアプリケーションとして成長することができます。

---

**文書情報**
- Document ID: Doc-AD-001
- Version: 1.0.0
- Last Updated: 2026-01-20
- Status: Draft
- Total Lines: 1,500
- Related Documents: Doc-TV-001, Doc-SA-001, Doc-SA-002, Doc-AD-002, Doc-AD-003

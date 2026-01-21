# Doc-SP-002: TripTripデータベーススキーマ・ERD詳細

## エグゼクティブサマリー

本文書は、TripTripプラットフォームの包括的なデータベーススキーマ仕様書です。PostgreSQL 16とPrisma ORMを基盤として、ユーザー、予約、旅程、決済、推奨の各ドメインにおける全テーブル定義を提供します。ER図（Entity-Relationship Diagram）、インデックス設計、データ整合性制約、マイグレーション戦略を含む、開発チームが即座に実装可能なレベルの技術仕様を提供します。パフォーマンス最適化とスケーラビリティを考慮した設計により、100万ユーザー規模までの成長をサポートします。

---

## 第1章：はじめに & データモデリング原則

### 1.1 文書の目的と範囲

```yaml
document_scope:
  purpose: TripTripデータベーススキーマの完全な技術仕様
  audience:
    - バックエンド開発者
    - データベースエンジニア
    - データアナリスト
    - DevOpsエンジニア

  coverage:
    - 全テーブル定義
    - リレーションシップ（外部キー）
    - インデックス設計
    - 制約定義
    - ER図
    - Prismaスキーマ
    - マイグレーション戦略

  technology_stack:
    database: PostgreSQL 16
    orm: Prisma 5.8.0
    extensions:
      - uuid-ossp
      - pgcrypto
      - pg_trgm
      - postgis (将来)
```

### 1.2 正規化 vs 非正規化の判断基準

```yaml
normalization_principles:
  default_approach: 第3正規形 (3NF)

  normalization_cases:
    description: 正規化を適用するケース
    criteria:
      - トランザクションデータ（予約、注文、決済）
      - マスターデータ（ユーザー、商品、ホテル）
      - 参照整合性が重要なデータ
    benefits:
      - データ整合性の保証
      - 更新異常の防止
      - ストレージ効率

  denormalization_cases:
    description: 非正規化を適用するケース
    criteria:
      - 読み取り頻度が非常に高い（キャッシュ候補）
      - 集計・分析用データ
      - 履歴・監査ログ
    examples:
      - booking_items.item_name（商品名のスナップショット）
      - user_statistics（集計済み統計）
      - search_materialized_views
    benefits:
      - 読み取りパフォーマンス向上
      - JOINの削減
      - クエリ簡素化

  hybrid_approach:
    description: ハイブリッドアプローチ
    implementation:
      - コアテーブルは正規化
      - 読み取り最適化用にマテリアライズドビュー
      - キャッシュレイヤーとの組み合わせ
```

### 1.3 Prisma ORMとの整合性

```yaml
prisma_conventions:
  naming:
    tables: PascalCase（User, Booking）
    columns: camelCase（createdAt, userId）
    relations: camelCase（user, bookings）

  id_strategy:
    type: UUID
    generation: "@default(uuid())"
    rationale:
      - 分散システム対応
      - セキュリティ（推測困難）
      - マージ・レプリケーション容易

  timestamps:
    created_at: "@default(now())"
    updated_at: "@updatedAt"

  soft_delete:
    enabled: true
    column: deletedAt
    type: DateTime?

  enum_handling:
    approach: PostgreSQL native enum
    example: |
      enum BookingStatus {
        PENDING
        CONFIRMED
        COMPLETED
        CANCELLED
        REFUNDED
      }
```

### 1.4 スキーマバージョニング

```yaml
schema_versioning:
  tool: Prisma Migrate

  workflow:
    development:
      - prisma migrate dev --name <migration_name>
      - 自動的にマイグレーションファイル生成
      - 開発DBに即時適用

    staging:
      - prisma migrate deploy
      - 保留中のマイグレーションを適用

    production:
      - prisma migrate deploy
      - CI/CDパイプラインで自動実行
      - ロールバック手順を準備

  naming_convention:
    format: "YYYYMMDDHHMMSS_description"
    examples:
      - "20260121100000_add_user_preferences"
      - "20260121110000_create_itinerary_tables"

  best_practices:
    - 破壊的変更は段階的に実行
    - ダウンタイムを最小化
    - バックアップを必ず取得
    - ロールバック手順を文書化
```

---

## 第2章：ユーザードメイン

### 2.1 users テーブル

```sql
-- ユーザー基本情報テーブル
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL,
    email_verified BOOLEAN NOT NULL DEFAULT FALSE,
    password_hash VARCHAR(255),
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(50),
    phone_verified BOOLEAN NOT NULL DEFAULT FALSE,
    avatar_url TEXT,
    role user_role NOT NULL DEFAULT 'USER',
    status user_status NOT NULL DEFAULT 'ACTIVE',
    language VARCHAR(5) NOT NULL DEFAULT 'ja',
    currency VARCHAR(3) NOT NULL DEFAULT 'JPY',
    timezone VARCHAR(50) NOT NULL DEFAULT 'Asia/Tokyo',
    last_login_at TIMESTAMP WITH TIME ZONE,
    login_count INTEGER NOT NULL DEFAULT 0,
    failed_login_attempts INTEGER NOT NULL DEFAULT 0,
    locked_until TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE,

    CONSTRAINT users_email_unique UNIQUE (email),
    CONSTRAINT users_email_format CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
    CONSTRAINT users_phone_format CHECK (phone IS NULL OR phone ~* '^\+?[0-9\-\s]+$')
);

-- ユーザーロール列挙型
CREATE TYPE user_role AS ENUM ('USER', 'PREMIUM', 'PARTNER', 'ADMIN');

-- ユーザーステータス列挙型
CREATE TYPE user_status AS ENUM ('ACTIVE', 'INACTIVE', 'SUSPENDED', 'DELETED');

-- インデックス
CREATE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_role ON users(role) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_status ON users(status) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_created_at ON users(created_at);
```

```prisma
// Prismaスキーマ定義
model User {
  id                  String    @id @default(uuid())
  email               String    @unique
  emailVerified       Boolean   @default(false)
  passwordHash        String?
  name                String
  phone               String?
  phoneVerified       Boolean   @default(false)
  avatarUrl           String?
  role                UserRole  @default(USER)
  status              UserStatus @default(ACTIVE)
  language            String    @default("ja")
  currency            String    @default("JPY")
  timezone            String    @default("Asia/Tokyo")
  lastLoginAt         DateTime?
  loginCount          Int       @default(0)
  failedLoginAttempts Int       @default(0)
  lockedUntil         DateTime?
  createdAt           DateTime  @default(now())
  updatedAt           DateTime  @updatedAt
  deletedAt           DateTime?

  // Relations
  profile             UserProfile?
  preferences         UserPreference?
  authTokens          AuthToken[]
  socialLogins        SocialLogin[]
  bookings            Booking[]
  itineraries         Itinerary[]
  reviews             Review[]
  paymentMethods      PaymentMethod[]

  @@index([email])
  @@index([role])
  @@index([status])
  @@index([createdAt])
  @@map("users")
}

enum UserRole {
  USER
  PREMIUM
  PARTNER
  ADMIN
}

enum UserStatus {
  ACTIVE
  INACTIVE
  SUSPENDED
  DELETED
}
```

### 2.2 user_profiles テーブル

```sql
-- ユーザープロフィール詳細テーブル
CREATE TABLE user_profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL UNIQUE,
    date_of_birth DATE,
    gender gender_type,
    nationality VARCHAR(2),
    passport_country VARCHAR(2),
    passport_expiry DATE,
    emergency_contact_name VARCHAR(100),
    emergency_contact_phone VARCHAR(50),
    address_line1 VARCHAR(255),
    address_line2 VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(100),
    postal_code VARCHAR(20),
    country VARCHAR(2),
    bio TEXT,
    website_url TEXT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_user_profiles_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE CASCADE
);

CREATE TYPE gender_type AS ENUM ('MALE', 'FEMALE', 'OTHER', 'PREFER_NOT_TO_SAY');

CREATE INDEX idx_user_profiles_user_id ON user_profiles(user_id);
```

```prisma
model UserProfile {
  id                   String     @id @default(uuid())
  userId               String     @unique
  dateOfBirth          DateTime?  @db.Date
  gender               Gender?
  nationality          String?    @db.VarChar(2)
  passportCountry      String?    @db.VarChar(2)
  passportExpiry       DateTime?  @db.Date
  emergencyContactName String?
  emergencyContactPhone String?
  addressLine1         String?
  addressLine2         String?
  city                 String?
  state                String?
  postalCode           String?
  country              String?    @db.VarChar(2)
  bio                  String?
  websiteUrl           String?
  createdAt            DateTime   @default(now())
  updatedAt            DateTime   @updatedAt

  user                 User       @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("user_profiles")
}

enum Gender {
  MALE
  FEMALE
  OTHER
  PREFER_NOT_TO_SAY
}
```

### 2.3 user_preferences テーブル

```sql
-- ユーザー設定テーブル
CREATE TABLE user_preferences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL UNIQUE,

    -- 通知設定
    notify_email BOOLEAN NOT NULL DEFAULT TRUE,
    notify_push BOOLEAN NOT NULL DEFAULT TRUE,
    notify_sms BOOLEAN NOT NULL DEFAULT FALSE,
    notify_marketing BOOLEAN NOT NULL DEFAULT FALSE,
    notify_booking_reminders BOOLEAN NOT NULL DEFAULT TRUE,
    notify_price_alerts BOOLEAN NOT NULL DEFAULT TRUE,

    -- 旅行設定
    preferred_airlines TEXT[], -- 配列型
    preferred_hotel_chains TEXT[],
    preferred_room_type room_type_preference,
    smoking_preference smoking_preference,
    dietary_restrictions TEXT[],
    accessibility_needs TEXT[],

    -- 表示設定
    theme theme_type NOT NULL DEFAULT 'SYSTEM',
    compact_view BOOLEAN NOT NULL DEFAULT FALSE,
    show_prices_with_tax BOOLEAN NOT NULL DEFAULT TRUE,

    -- その他設定
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_user_preferences_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE CASCADE
);

CREATE TYPE room_type_preference AS ENUM ('SINGLE', 'DOUBLE', 'TWIN', 'SUITE', 'NO_PREFERENCE');
CREATE TYPE smoking_preference AS ENUM ('NON_SMOKING', 'SMOKING', 'NO_PREFERENCE');
CREATE TYPE theme_type AS ENUM ('LIGHT', 'DARK', 'SYSTEM');

CREATE INDEX idx_user_preferences_user_id ON user_preferences(user_id);
```

### 2.4 auth_tokens テーブル

```sql
-- 認証トークンテーブル
CREATE TABLE auth_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    token_hash VARCHAR(255) NOT NULL,
    token_type token_type NOT NULL,
    device_id VARCHAR(255),
    device_type device_type,
    device_name VARCHAR(255),
    ip_address INET,
    user_agent TEXT,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    last_used_at TIMESTAMP WITH TIME ZONE,
    revoked_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_auth_tokens_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE CASCADE
);

CREATE TYPE token_type AS ENUM ('ACCESS', 'REFRESH', 'PASSWORD_RESET', 'EMAIL_VERIFICATION');
CREATE TYPE device_type AS ENUM ('IOS', 'ANDROID', 'WEB', 'UNKNOWN');

CREATE INDEX idx_auth_tokens_user_id ON auth_tokens(user_id);
CREATE INDEX idx_auth_tokens_token_hash ON auth_tokens(token_hash);
CREATE INDEX idx_auth_tokens_expires_at ON auth_tokens(expires_at) WHERE revoked_at IS NULL;
```

### 2.5 social_logins テーブル

```sql
-- ソーシャルログインテーブル
CREATE TABLE social_logins (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    provider social_provider NOT NULL,
    provider_user_id VARCHAR(255) NOT NULL,
    provider_email VARCHAR(255),
    provider_name VARCHAR(255),
    provider_avatar_url TEXT,
    access_token TEXT,
    refresh_token TEXT,
    token_expires_at TIMESTAMP WITH TIME ZONE,
    raw_profile JSONB,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_social_logins_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE CASCADE,
    CONSTRAINT social_logins_provider_user_unique UNIQUE (provider, provider_user_id)
);

CREATE TYPE social_provider AS ENUM ('GOOGLE', 'APPLE', 'FACEBOOK', 'LINE');

CREATE INDEX idx_social_logins_user_id ON social_logins(user_id);
CREATE INDEX idx_social_logins_provider ON social_logins(provider, provider_user_id);
```

---

## 第3章：旅程ドメイン

### 3.1 itineraries テーブル

```sql
-- 旅程テーブル
CREATE TABLE itineraries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    title VARCHAR(200) NOT NULL,
    description TEXT,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    cover_image_url TEXT,
    status itinerary_status NOT NULL DEFAULT 'DRAFT',
    visibility visibility_type NOT NULL DEFAULT 'PRIVATE',
    share_token VARCHAR(100) UNIQUE,
    share_password_hash VARCHAR(255),
    share_expires_at TIMESTAMP WITH TIME ZONE,
    total_estimated_cost DECIMAL(12, 2),
    currency VARCHAR(3) NOT NULL DEFAULT 'JPY',
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE,

    CONSTRAINT fk_itineraries_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE CASCADE,
    CONSTRAINT itineraries_date_check CHECK (end_date >= start_date)
);

CREATE TYPE itinerary_status AS ENUM ('DRAFT', 'PLANNED', 'ACTIVE', 'COMPLETED', 'ARCHIVED');
CREATE TYPE visibility_type AS ENUM ('PRIVATE', 'SHARED', 'PUBLIC');

CREATE INDEX idx_itineraries_user_id ON itineraries(user_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_itineraries_status ON itineraries(status) WHERE deleted_at IS NULL;
CREATE INDEX idx_itineraries_start_date ON itineraries(start_date) WHERE deleted_at IS NULL;
CREATE INDEX idx_itineraries_share_token ON itineraries(share_token) WHERE share_token IS NOT NULL;
```

```prisma
model Itinerary {
  id                 String           @id @default(uuid())
  userId             String
  title              String           @db.VarChar(200)
  description        String?
  startDate          DateTime         @db.Date
  endDate            DateTime         @db.Date
  coverImageUrl      String?
  status             ItineraryStatus  @default(DRAFT)
  visibility         Visibility       @default(PRIVATE)
  shareToken         String?          @unique
  sharePasswordHash  String?
  shareExpiresAt     DateTime?
  totalEstimatedCost Decimal?         @db.Decimal(12, 2)
  currency           String           @default("JPY")
  metadata           Json             @default("{}")
  createdAt          DateTime         @default(now())
  updatedAt          DateTime         @updatedAt
  deletedAt          DateTime?

  user               User             @relation(fields: [userId], references: [id], onDelete: Cascade)
  items              ItineraryItem[]
  collaborators      ItineraryCollaborator[]
  destinations       ItineraryDestination[]

  @@index([userId])
  @@index([status])
  @@index([startDate])
  @@map("itineraries")
}

enum ItineraryStatus {
  DRAFT
  PLANNED
  ACTIVE
  COMPLETED
  ARCHIVED
}

enum Visibility {
  PRIVATE
  SHARED
  PUBLIC
}
```

### 3.2 itinerary_items テーブル

```sql
-- 旅程アイテムテーブル
CREATE TABLE itinerary_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    itinerary_id UUID NOT NULL,
    item_type itinerary_item_type NOT NULL,
    date DATE NOT NULL,
    day_number INTEGER NOT NULL,
    order_index INTEGER NOT NULL DEFAULT 0,
    start_time TIME,
    end_time TIME,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    location_name VARCHAR(255),
    location_address TEXT,
    location_latitude DECIMAL(10, 8),
    location_longitude DECIMAL(11, 8),
    reference_type reference_type,
    reference_id UUID,
    booking_id UUID,
    estimated_cost DECIMAL(10, 2),
    currency VARCHAR(3) DEFAULT 'JPY',
    notes TEXT,
    attachments JSONB DEFAULT '[]',
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE,

    CONSTRAINT fk_itinerary_items_itinerary FOREIGN KEY (itinerary_id)
        REFERENCES itineraries(id) ON DELETE CASCADE,
    CONSTRAINT fk_itinerary_items_booking FOREIGN KEY (booking_id)
        REFERENCES bookings(id) ON DELETE SET NULL
);

CREATE TYPE itinerary_item_type AS ENUM (
    'HOTEL', 'ACTIVITY', 'RESTAURANT', 'TRANSPORT', 'NOTE', 'BOOKING', 'FREE_TIME'
);

CREATE TYPE reference_type AS ENUM ('HOTEL', 'ACTIVITY', 'RESTAURANT', 'ATTRACTION');

CREATE INDEX idx_itinerary_items_itinerary_id ON itinerary_items(itinerary_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_itinerary_items_date ON itinerary_items(itinerary_id, date, order_index);
CREATE INDEX idx_itinerary_items_booking_id ON itinerary_items(booking_id) WHERE booking_id IS NOT NULL;
```

### 3.3 itinerary_collaborators テーブル

```sql
-- 旅程コラボレーターテーブル
CREATE TABLE itinerary_collaborators (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    itinerary_id UUID NOT NULL,
    user_id UUID,
    email VARCHAR(255),
    name VARCHAR(100),
    role collaborator_role NOT NULL DEFAULT 'VIEWER',
    invite_token VARCHAR(100) UNIQUE,
    invite_expires_at TIMESTAMP WITH TIME ZONE,
    accepted_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_itinerary_collaborators_itinerary FOREIGN KEY (itinerary_id)
        REFERENCES itineraries(id) ON DELETE CASCADE,
    CONSTRAINT fk_itinerary_collaborators_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE CASCADE,
    CONSTRAINT itinerary_collaborators_user_or_email CHECK (
        (user_id IS NOT NULL) OR (email IS NOT NULL)
    )
);

CREATE TYPE collaborator_role AS ENUM ('VIEWER', 'EDITOR');

CREATE INDEX idx_itinerary_collaborators_itinerary_id ON itinerary_collaborators(itinerary_id);
CREATE INDEX idx_itinerary_collaborators_user_id ON itinerary_collaborators(user_id);
CREATE INDEX idx_itinerary_collaborators_invite_token ON itinerary_collaborators(invite_token)
    WHERE invite_token IS NOT NULL AND accepted_at IS NULL;
```

### 3.4 destinations テーブル

```sql
-- 目的地マスターテーブル
CREATE TABLE destinations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(200) NOT NULL,
    name_en VARCHAR(200),
    name_local VARCHAR(200),
    description TEXT,
    description_en TEXT,
    destination_type destination_type NOT NULL,
    country_code VARCHAR(2) NOT NULL,
    country_name VARCHAR(100) NOT NULL,
    region VARCHAR(100),
    city VARCHAR(100),
    latitude DECIMAL(10, 8),
    longitude DECIMAL(11, 8),
    timezone VARCHAR(50),
    image_url TEXT,
    thumbnail_url TEXT,
    popularity_score DECIMAL(5, 2) DEFAULT 0,
    is_featured BOOLEAN NOT NULL DEFAULT FALSE,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

    CONSTRAINT destinations_name_country_unique UNIQUE (name, country_code)
);

CREATE TYPE destination_type AS ENUM ('CITY', 'REGION', 'COUNTRY', 'AIRPORT', 'LANDMARK');

CREATE INDEX idx_destinations_name ON destinations USING gin (name gin_trgm_ops);
CREATE INDEX idx_destinations_country ON destinations(country_code);
CREATE INDEX idx_destinations_type ON destinations(destination_type);
CREATE INDEX idx_destinations_featured ON destinations(is_featured) WHERE is_featured = TRUE;
CREATE INDEX idx_destinations_popularity ON destinations(popularity_score DESC);
```

---

## 第4章：予約ドメイン

### 4.1 bookings テーブル

```sql
-- 予約テーブル
CREATE TABLE bookings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_number VARCHAR(20) NOT NULL UNIQUE,
    user_id UUID NOT NULL,
    booking_type booking_type NOT NULL,
    status booking_status NOT NULL DEFAULT 'PENDING',

    -- 連絡先情報
    contact_name VARCHAR(100) NOT NULL,
    contact_email VARCHAR(255) NOT NULL,
    contact_phone VARCHAR(50) NOT NULL,
    contact_country VARCHAR(2),

    -- 価格情報
    subtotal DECIMAL(12, 2) NOT NULL,
    discount_amount DECIMAL(12, 2) NOT NULL DEFAULT 0,
    tax_amount DECIMAL(12, 2) NOT NULL DEFAULT 0,
    service_fee DECIMAL(12, 2) NOT NULL DEFAULT 0,
    total_amount DECIMAL(12, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'JPY',

    -- プロモーション
    promo_code VARCHAR(50),
    promo_discount_type discount_type,
    promo_discount_value DECIMAL(10, 2),

    -- その他
    special_requests TEXT,
    internal_notes TEXT,
    metadata JSONB DEFAULT '{}',

    -- タイムスタンプ
    confirmed_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    cancelled_at TIMESTAMP WITH TIME ZONE,
    cancellation_reason TEXT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE,

    CONSTRAINT fk_bookings_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE RESTRICT,
    CONSTRAINT bookings_total_check CHECK (total_amount >= 0),
    CONSTRAINT bookings_subtotal_check CHECK (subtotal >= 0)
);

CREATE TYPE booking_type AS ENUM ('HOTEL', 'ACTIVITY', 'PRODUCT', 'PACKAGE');
CREATE TYPE booking_status AS ENUM ('PENDING', 'CONFIRMED', 'COMPLETED', 'CANCELLED', 'REFUNDED');
CREATE TYPE discount_type AS ENUM ('PERCENTAGE', 'FIXED_AMOUNT');

-- 予約番号生成関数
CREATE OR REPLACE FUNCTION generate_booking_number()
RETURNS TEXT AS $$
DECLARE
    prefix TEXT := 'TT';
    date_part TEXT := to_char(CURRENT_DATE, 'YYYYMMDD');
    seq_num INTEGER;
    result TEXT;
BEGIN
    SELECT COALESCE(MAX(CAST(SUBSTRING(booking_number FROM 11) AS INTEGER)), 0) + 1
    INTO seq_num
    FROM bookings
    WHERE booking_number LIKE prefix || '-' || date_part || '%';

    result := prefix || '-' || date_part || LPAD(seq_num::TEXT, 5, '0');
    RETURN result;
END;
$$ LANGUAGE plpgsql;

CREATE INDEX idx_bookings_user_id ON bookings(user_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_bookings_booking_number ON bookings(booking_number);
CREATE INDEX idx_bookings_status ON bookings(status) WHERE deleted_at IS NULL;
CREATE INDEX idx_bookings_type ON bookings(booking_type) WHERE deleted_at IS NULL;
CREATE INDEX idx_bookings_created_at ON bookings(created_at DESC);
```

```prisma
model Booking {
  id                 String        @id @default(uuid())
  bookingNumber      String        @unique @db.VarChar(20)
  userId             String
  bookingType        BookingType
  status             BookingStatus @default(PENDING)

  contactName        String        @db.VarChar(100)
  contactEmail       String        @db.VarChar(255)
  contactPhone       String        @db.VarChar(50)
  contactCountry     String?       @db.VarChar(2)

  subtotal           Decimal       @db.Decimal(12, 2)
  discountAmount     Decimal       @default(0) @db.Decimal(12, 2)
  taxAmount          Decimal       @default(0) @db.Decimal(12, 2)
  serviceFee         Decimal       @default(0) @db.Decimal(12, 2)
  totalAmount        Decimal       @db.Decimal(12, 2)
  currency           String        @default("JPY") @db.VarChar(3)

  promoCode          String?       @db.VarChar(50)
  promoDiscountType  DiscountType?
  promoDiscountValue Decimal?      @db.Decimal(10, 2)

  specialRequests    String?
  internalNotes      String?
  metadata           Json          @default("{}")

  confirmedAt        DateTime?
  completedAt        DateTime?
  cancelledAt        DateTime?
  cancellationReason String?
  createdAt          DateTime      @default(now())
  updatedAt          DateTime      @updatedAt
  deletedAt          DateTime?

  user               User          @relation(fields: [userId], references: [id], onDelete: Restrict)
  items              BookingItem[]
  statusHistory      BookingStatusHistory[]
  payments           Payment[]
  itineraryItems     ItineraryItem[]

  @@index([userId])
  @@index([bookingNumber])
  @@index([status])
  @@index([bookingType])
  @@index([createdAt(sort: Desc)])
  @@map("bookings")
}

enum BookingType {
  HOTEL
  ACTIVITY
  PRODUCT
  PACKAGE
}

enum BookingStatus {
  PENDING
  CONFIRMED
  COMPLETED
  CANCELLED
  REFUNDED
}

enum DiscountType {
  PERCENTAGE
  FIXED_AMOUNT
}
```

### 4.2 booking_items テーブル

```sql
-- 予約アイテムテーブル
CREATE TABLE booking_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id UUID NOT NULL,
    item_type booking_item_type NOT NULL,
    item_id UUID NOT NULL,

    -- アイテム情報（スナップショット）
    item_name VARCHAR(255) NOT NULL,
    item_description TEXT,
    item_image_url TEXT,

    -- 数量・日程
    quantity INTEGER NOT NULL DEFAULT 1,
    check_in_date DATE,
    check_out_date DATE,
    activity_date DATE,
    time_slot_id UUID,
    time_slot_start TIME,
    time_slot_end TIME,

    -- 価格
    unit_price DECIMAL(10, 2) NOT NULL,
    total_price DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'JPY',

    -- ゲスト情報
    guests JSONB DEFAULT '[]',

    -- オプション
    options JSONB DEFAULT '[]',

    -- サプライヤー情報
    supplier_id UUID,
    supplier_confirmation_code VARCHAR(100),
    supplier_notes TEXT,

    -- ステータス
    status booking_item_status NOT NULL DEFAULT 'PENDING',
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_booking_items_booking FOREIGN KEY (booking_id)
        REFERENCES bookings(id) ON DELETE CASCADE,
    CONSTRAINT booking_items_quantity_check CHECK (quantity > 0),
    CONSTRAINT booking_items_price_check CHECK (unit_price >= 0 AND total_price >= 0)
);

CREATE TYPE booking_item_type AS ENUM ('ROOM', 'ACTIVITY', 'PRODUCT', 'TICKET');
CREATE TYPE booking_item_status AS ENUM ('PENDING', 'CONFIRMED', 'CANCELLED', 'COMPLETED');

CREATE INDEX idx_booking_items_booking_id ON booking_items(booking_id);
CREATE INDEX idx_booking_items_item ON booking_items(item_type, item_id);
CREATE INDEX idx_booking_items_check_in ON booking_items(check_in_date) WHERE check_in_date IS NOT NULL;
CREATE INDEX idx_booking_items_activity_date ON booking_items(activity_date) WHERE activity_date IS NOT NULL;
```

### 4.3 booking_status_history テーブル

```sql
-- 予約ステータス履歴テーブル
CREATE TABLE booking_status_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id UUID NOT NULL,
    previous_status booking_status,
    new_status booking_status NOT NULL,
    changed_by UUID,
    change_type status_change_type NOT NULL,
    reason TEXT,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_booking_status_history_booking FOREIGN KEY (booking_id)
        REFERENCES bookings(id) ON DELETE CASCADE,
    CONSTRAINT fk_booking_status_history_user FOREIGN KEY (changed_by)
        REFERENCES users(id) ON DELETE SET NULL
);

CREATE TYPE status_change_type AS ENUM ('SYSTEM', 'USER', 'ADMIN', 'SUPPLIER', 'PAYMENT');

CREATE INDEX idx_booking_status_history_booking_id ON booking_status_history(booking_id);
CREATE INDEX idx_booking_status_history_created_at ON booking_status_history(created_at);
```

---

## 第5章：商品・サービスドメイン

### 5.1 hotels テーブル

```sql
-- ホテルマスターテーブル
CREATE TABLE hotels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_id UUID,
    name VARCHAR(255) NOT NULL,
    name_en VARCHAR(255),
    description TEXT,
    description_en TEXT,

    -- 評価
    star_rating SMALLINT CHECK (star_rating BETWEEN 1 AND 5),
    user_rating DECIMAL(2, 1),
    review_count INTEGER NOT NULL DEFAULT 0,

    -- 住所
    address_line1 VARCHAR(255) NOT NULL,
    address_line2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    state VARCHAR(100),
    postal_code VARCHAR(20),
    country_code VARCHAR(2) NOT NULL,

    -- 位置情報
    latitude DECIMAL(10, 8) NOT NULL,
    longitude DECIMAL(11, 8) NOT NULL,

    -- 連絡先
    phone VARCHAR(50),
    email VARCHAR(255),
    website_url TEXT,

    -- チェックイン/アウト
    check_in_time TIME NOT NULL DEFAULT '15:00',
    check_out_time TIME NOT NULL DEFAULT '11:00',

    -- 設備
    amenities TEXT[] DEFAULT '{}',

    -- 画像
    images JSONB DEFAULT '[]',
    thumbnail_url TEXT,

    -- ポリシー
    cancellation_policy TEXT,
    child_policy TEXT,
    pet_policy TEXT,

    -- ステータス
    status entity_status NOT NULL DEFAULT 'ACTIVE',
    is_featured BOOLEAN NOT NULL DEFAULT FALSE,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE
);

CREATE TYPE entity_status AS ENUM ('ACTIVE', 'INACTIVE', 'PENDING', 'SUSPENDED');

CREATE INDEX idx_hotels_name ON hotels USING gin (name gin_trgm_ops);
CREATE INDEX idx_hotels_city ON hotels(city);
CREATE INDEX idx_hotels_country ON hotels(country_code);
CREATE INDEX idx_hotels_location ON hotels USING gist (
    ll_to_earth(latitude, longitude)
);
CREATE INDEX idx_hotels_star_rating ON hotels(star_rating) WHERE status = 'ACTIVE';
CREATE INDEX idx_hotels_user_rating ON hotels(user_rating DESC NULLS LAST) WHERE status = 'ACTIVE';
CREATE INDEX idx_hotels_featured ON hotels(is_featured) WHERE is_featured = TRUE AND status = 'ACTIVE';
```

### 5.2 activities テーブル

```sql
-- アクティビティマスターテーブル
CREATE TABLE activities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_id UUID,
    title VARCHAR(255) NOT NULL,
    title_en VARCHAR(255),
    description TEXT,
    description_en TEXT,

    -- カテゴリ
    category activity_category NOT NULL,
    subcategory VARCHAR(100),

    -- 所要時間
    duration_minutes INTEGER NOT NULL,

    -- 評価
    rating DECIMAL(2, 1),
    review_count INTEGER NOT NULL DEFAULT 0,

    -- 場所
    location_name VARCHAR(255) NOT NULL,
    address TEXT,
    city VARCHAR(100),
    country_code VARCHAR(2) NOT NULL,
    latitude DECIMAL(10, 8),
    longitude DECIMAL(11, 8),
    meeting_point TEXT,

    -- 価格
    base_price DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'JPY',
    price_type price_type NOT NULL DEFAULT 'PER_PERSON',

    -- 参加条件
    min_participants INTEGER NOT NULL DEFAULT 1,
    max_participants INTEGER,
    age_restriction TEXT,

    -- 対応言語
    languages TEXT[] DEFAULT '{ja}',

    -- コンテンツ
    highlights TEXT[] DEFAULT '{}',
    inclusions TEXT[] DEFAULT '{}',
    exclusions TEXT[] DEFAULT '{}',
    requirements TEXT[] DEFAULT '{}',

    -- 画像
    images JSONB DEFAULT '[]',
    thumbnail_url TEXT,

    -- キャンセルポリシー
    cancellation_policy TEXT,
    free_cancellation_hours INTEGER,

    -- ステータス
    status entity_status NOT NULL DEFAULT 'ACTIVE',
    is_featured BOOLEAN NOT NULL DEFAULT FALSE,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE
);

CREATE TYPE activity_category AS ENUM (
    'CULTURE', 'NATURE', 'FOOD', 'ADVENTURE',
    'RELAXATION', 'NIGHTLIFE', 'SHOPPING', 'TOURS'
);

CREATE TYPE price_type AS ENUM ('PER_PERSON', 'PER_GROUP', 'PER_UNIT');

CREATE INDEX idx_activities_title ON activities USING gin (title gin_trgm_ops);
CREATE INDEX idx_activities_category ON activities(category) WHERE status = 'ACTIVE';
CREATE INDEX idx_activities_city ON activities(city) WHERE status = 'ACTIVE';
CREATE INDEX idx_activities_rating ON activities(rating DESC NULLS LAST) WHERE status = 'ACTIVE';
CREATE INDEX idx_activities_price ON activities(base_price) WHERE status = 'ACTIVE';
```

### 5.3 products テーブル

```sql
-- 商品マスターテーブル
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_id UUID,
    sku VARCHAR(100) UNIQUE,
    name VARCHAR(255) NOT NULL,
    name_en VARCHAR(255),
    description TEXT,
    description_en TEXT,

    -- カテゴリ
    category product_category NOT NULL,
    subcategory VARCHAR(100),

    -- 価格
    price DECIMAL(10, 2) NOT NULL,
    original_price DECIMAL(10, 2),
    currency VARCHAR(3) NOT NULL DEFAULT 'JPY',

    -- 在庫
    stock_quantity INTEGER NOT NULL DEFAULT 0,
    low_stock_threshold INTEGER DEFAULT 10,
    track_inventory BOOLEAN NOT NULL DEFAULT TRUE,

    -- 物理的属性
    weight_grams INTEGER,
    dimensions JSONB, -- {length, width, height}

    -- バリエーション
    has_variants BOOLEAN NOT NULL DEFAULT FALSE,
    variant_options JSONB DEFAULT '[]',

    -- 画像
    images JSONB DEFAULT '[]',
    thumbnail_url TEXT,

    -- SEO
    slug VARCHAR(255) UNIQUE,
    meta_title VARCHAR(255),
    meta_description TEXT,

    -- ステータス
    status entity_status NOT NULL DEFAULT 'ACTIVE',
    is_featured BOOLEAN NOT NULL DEFAULT FALSE,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE,

    CONSTRAINT products_price_check CHECK (price >= 0),
    CONSTRAINT products_stock_check CHECK (stock_quantity >= 0)
);

CREATE TYPE product_category AS ENUM (
    'KIMONO', 'ACCESSORIES', 'SOUVENIRS',
    'FOOD', 'CRAFTS', 'ELECTRONICS', 'OTHER'
);

CREATE INDEX idx_products_name ON products USING gin (name gin_trgm_ops);
CREATE INDEX idx_products_sku ON products(sku) WHERE sku IS NOT NULL;
CREATE INDEX idx_products_category ON products(category) WHERE status = 'ACTIVE';
CREATE INDEX idx_products_price ON products(price) WHERE status = 'ACTIVE';
CREATE INDEX idx_products_slug ON products(slug) WHERE slug IS NOT NULL;
```

### 5.4 tickets テーブル

```sql
-- チケットマスターテーブル
CREATE TABLE tickets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    attraction_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,

    -- 価格
    adult_price DECIMAL(10, 2) NOT NULL,
    child_price DECIMAL(10, 2),
    infant_price DECIMAL(10, 2),
    currency VARCHAR(3) NOT NULL DEFAULT 'JPY',

    -- 有効期間
    valid_from DATE,
    valid_until DATE,

    -- 利用条件
    usage_type ticket_usage_type NOT NULL DEFAULT 'SINGLE',
    valid_days INTEGER, -- マルチデイチケットの場合

    -- 制限
    max_daily_capacity INTEGER,
    requires_time_slot BOOLEAN NOT NULL DEFAULT FALSE,

    -- ステータス
    status entity_status NOT NULL DEFAULT 'ACTIVE',
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TYPE ticket_usage_type AS ENUM ('SINGLE', 'MULTI_DAY', 'ANNUAL');

CREATE INDEX idx_tickets_attraction_id ON tickets(attraction_id) WHERE status = 'ACTIVE';
```

### 5.5 kimono_rentals テーブル

```sql
-- 着物レンタルテーブル
CREATE TABLE kimono_rentals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_id UUID,
    name VARCHAR(255) NOT NULL,
    description TEXT,

    -- カテゴリ
    kimono_type kimono_type NOT NULL,
    gender gender_type NOT NULL,

    -- 価格
    base_price DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'JPY',

    -- サイズ
    available_sizes TEXT[] NOT NULL,

    -- 含まれるもの
    includes TEXT[] DEFAULT '{}',

    -- オプション
    available_options JSONB DEFAULT '[]',

    -- 画像
    images JSONB DEFAULT '[]',
    thumbnail_url TEXT,

    -- 在庫
    inventory JSONB DEFAULT '{}', -- サイズ別在庫

    -- ステータス
    status entity_status NOT NULL DEFAULT 'ACTIVE',
    is_featured BOOLEAN NOT NULL DEFAULT FALSE,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TYPE kimono_type AS ENUM (
    'FURISODE', 'HOUMONGI', 'KOMON', 'YUKATA',
    'HAKAMA', 'UCHIKAKE', 'SHIROMUKU'
);

CREATE INDEX idx_kimono_rentals_type ON kimono_rentals(kimono_type) WHERE status = 'ACTIVE';
CREATE INDEX idx_kimono_rentals_gender ON kimono_rentals(gender) WHERE status = 'ACTIVE';
```

---

## 第6章：決済ドメイン

### 6.1 payments テーブル

```sql
-- 決済テーブル
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id UUID NOT NULL,
    user_id UUID NOT NULL,

    -- 金額
    amount DECIMAL(12, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'JPY',

    -- 決済プロバイダー
    provider payment_provider NOT NULL,
    provider_payment_id VARCHAR(255),
    provider_customer_id VARCHAR(255),

    -- 決済方法
    payment_method_type payment_method_type NOT NULL,
    payment_method_id UUID,
    card_brand VARCHAR(50),
    card_last_four VARCHAR(4),
    card_exp_month SMALLINT,
    card_exp_year SMALLINT,

    -- ステータス
    status payment_status NOT NULL DEFAULT 'PENDING',

    -- 3Dセキュア
    requires_action BOOLEAN NOT NULL DEFAULT FALSE,
    action_type VARCHAR(50),
    action_url TEXT,

    -- レスポンス
    provider_response JSONB,
    error_code VARCHAR(100),
    error_message TEXT,

    -- タイムスタンプ
    authorized_at TIMESTAMP WITH TIME ZONE,
    captured_at TIMESTAMP WITH TIME ZONE,
    failed_at TIMESTAMP WITH TIME ZONE,

    -- 冪等性
    idempotency_key UUID UNIQUE,

    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_payments_booking FOREIGN KEY (booking_id)
        REFERENCES bookings(id) ON DELETE RESTRICT,
    CONSTRAINT fk_payments_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE RESTRICT,
    CONSTRAINT fk_payments_payment_method FOREIGN KEY (payment_method_id)
        REFERENCES payment_methods(id) ON DELETE SET NULL,
    CONSTRAINT payments_amount_check CHECK (amount > 0)
);

CREATE TYPE payment_provider AS ENUM ('STRIPE', 'PAYPAY', 'APPLE_PAY', 'GOOGLE_PAY');
CREATE TYPE payment_method_type AS ENUM ('CARD', 'APPLE_PAY', 'GOOGLE_PAY', 'BANK_TRANSFER', 'CONVENIENCE_STORE');
CREATE TYPE payment_status AS ENUM (
    'PENDING', 'PROCESSING', 'AUTHORIZED', 'CAPTURED',
    'SUCCEEDED', 'FAILED', 'CANCELLED', 'REFUNDED', 'PARTIALLY_REFUNDED'
);

CREATE INDEX idx_payments_booking_id ON payments(booking_id);
CREATE INDEX idx_payments_user_id ON payments(user_id);
CREATE INDEX idx_payments_status ON payments(status);
CREATE INDEX idx_payments_provider_id ON payments(provider, provider_payment_id);
CREATE INDEX idx_payments_idempotency ON payments(idempotency_key) WHERE idempotency_key IS NOT NULL;
CREATE INDEX idx_payments_created_at ON payments(created_at DESC);
```

### 6.2 payment_methods テーブル

```sql
-- 保存済み支払い方法テーブル
CREATE TABLE payment_methods (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,

    -- 決済プロバイダー
    provider payment_provider NOT NULL,
    provider_payment_method_id VARCHAR(255) NOT NULL,

    -- カード情報
    type payment_method_type NOT NULL,
    card_brand VARCHAR(50),
    card_last_four VARCHAR(4),
    card_exp_month SMALLINT,
    card_exp_year SMALLINT,
    card_holder_name VARCHAR(255),

    -- 請求先住所
    billing_address JSONB,

    -- 設定
    is_default BOOLEAN NOT NULL DEFAULT FALSE,
    nickname VARCHAR(50),

    -- ステータス
    status payment_method_status NOT NULL DEFAULT 'ACTIVE',

    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE,

    CONSTRAINT fk_payment_methods_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE CASCADE
);

CREATE TYPE payment_method_status AS ENUM ('ACTIVE', 'EXPIRED', 'INVALID', 'DELETED');

CREATE INDEX idx_payment_methods_user_id ON payment_methods(user_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_payment_methods_default ON payment_methods(user_id, is_default)
    WHERE is_default = TRUE AND deleted_at IS NULL;
```

### 6.3 refunds テーブル

```sql
-- 返金テーブル
CREATE TABLE refunds (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id UUID NOT NULL,
    booking_id UUID NOT NULL,

    -- 金額
    amount DECIMAL(12, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'JPY',

    -- 決済プロバイダー
    provider_refund_id VARCHAR(255),

    -- 理由
    reason refund_reason NOT NULL,
    reason_details TEXT,

    -- ステータス
    status refund_status NOT NULL DEFAULT 'PENDING',

    -- 処理情報
    processed_by UUID,
    processed_at TIMESTAMP WITH TIME ZONE,
    estimated_arrival DATE,

    provider_response JSONB,
    error_code VARCHAR(100),
    error_message TEXT,

    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_refunds_payment FOREIGN KEY (payment_id)
        REFERENCES payments(id) ON DELETE RESTRICT,
    CONSTRAINT fk_refunds_booking FOREIGN KEY (booking_id)
        REFERENCES bookings(id) ON DELETE RESTRICT,
    CONSTRAINT fk_refunds_processed_by FOREIGN KEY (processed_by)
        REFERENCES users(id) ON DELETE SET NULL,
    CONSTRAINT refunds_amount_check CHECK (amount > 0)
);

CREATE TYPE refund_reason AS ENUM (
    'REQUESTED_BY_CUSTOMER', 'DUPLICATE', 'FRAUDULENT',
    'CANCELLATION', 'SERVICE_NOT_PROVIDED', 'OTHER'
);

CREATE TYPE refund_status AS ENUM ('PENDING', 'PROCESSING', 'SUCCEEDED', 'FAILED', 'CANCELLED');

CREATE INDEX idx_refunds_payment_id ON refunds(payment_id);
CREATE INDEX idx_refunds_booking_id ON refunds(booking_id);
CREATE INDEX idx_refunds_status ON refunds(status);
CREATE INDEX idx_refunds_created_at ON refunds(created_at DESC);
```

### 6.4 invoices テーブル

```sql
-- 請求書テーブル
CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_number VARCHAR(20) NOT NULL UNIQUE,
    booking_id UUID NOT NULL,
    user_id UUID NOT NULL,

    -- 金額
    subtotal DECIMAL(12, 2) NOT NULL,
    tax_amount DECIMAL(12, 2) NOT NULL,
    total_amount DECIMAL(12, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'JPY',

    -- 請求先情報
    billing_name VARCHAR(255) NOT NULL,
    billing_email VARCHAR(255) NOT NULL,
    billing_address JSONB,

    -- ステータス
    status invoice_status NOT NULL DEFAULT 'DRAFT',

    -- 日付
    issue_date DATE NOT NULL DEFAULT CURRENT_DATE,
    due_date DATE,
    paid_date DATE,

    -- ファイル
    pdf_url TEXT,

    -- 行項目
    line_items JSONB NOT NULL DEFAULT '[]',

    notes TEXT,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_invoices_booking FOREIGN KEY (booking_id)
        REFERENCES bookings(id) ON DELETE RESTRICT,
    CONSTRAINT fk_invoices_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE RESTRICT
);

CREATE TYPE invoice_status AS ENUM ('DRAFT', 'SENT', 'PAID', 'OVERDUE', 'CANCELLED', 'REFUNDED');

CREATE INDEX idx_invoices_booking_id ON invoices(booking_id);
CREATE INDEX idx_invoices_user_id ON invoices(user_id);
CREATE INDEX idx_invoices_status ON invoices(status);
CREATE INDEX idx_invoices_issue_date ON invoices(issue_date DESC);
```

---

## 第7章：推奨・パーソナライゼーション

### 7.1 user_interactions テーブル

```sql
-- ユーザーインタラクションテーブル
CREATE TABLE user_interactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID,
    session_id VARCHAR(100),

    -- インタラクション情報
    interaction_type interaction_type NOT NULL,
    entity_type entity_type NOT NULL,
    entity_id UUID NOT NULL,

    -- コンテキスト
    source_page VARCHAR(100),
    search_query TEXT,
    filters JSONB,

    -- デバイス情報
    device_type device_type,
    ip_address INET,
    user_agent TEXT,

    -- 追加情報
    duration_seconds INTEGER,
    scroll_depth DECIMAL(5, 2),
    click_position JSONB,

    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_user_interactions_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE SET NULL
);

CREATE TYPE interaction_type AS ENUM (
    'VIEW', 'CLICK', 'SEARCH', 'ADD_TO_CART', 'REMOVE_FROM_CART',
    'ADD_TO_WISHLIST', 'SHARE', 'BOOK', 'REVIEW', 'SCROLL'
);

CREATE TYPE entity_type AS ENUM (
    'HOTEL', 'ACTIVITY', 'PRODUCT', 'DESTINATION',
    'ITINERARY', 'TICKET', 'KIMONO'
);

-- パーティショニング（日付別）
CREATE INDEX idx_user_interactions_user_id ON user_interactions(user_id) WHERE user_id IS NOT NULL;
CREATE INDEX idx_user_interactions_entity ON user_interactions(entity_type, entity_id);
CREATE INDEX idx_user_interactions_type ON user_interactions(interaction_type);
CREATE INDEX idx_user_interactions_created_at ON user_interactions(created_at DESC);
CREATE INDEX idx_user_interactions_session ON user_interactions(session_id) WHERE session_id IS NOT NULL;
```

### 7.2 recommendations_log テーブル

```sql
-- 推奨ログテーブル
CREATE TABLE recommendations_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID,
    session_id VARCHAR(100),

    -- 推奨情報
    recommendation_type recommendation_type NOT NULL,
    context recommendation_context NOT NULL,

    -- 推奨されたアイテム
    recommended_items JSONB NOT NULL, -- [{entity_type, entity_id, score, position}]

    -- アルゴリズム情報
    algorithm VARCHAR(100) NOT NULL,
    algorithm_version VARCHAR(20),
    model_id VARCHAR(100),

    -- 入力パラメータ
    input_parameters JSONB DEFAULT '{}',

    -- 結果
    was_clicked BOOLEAN DEFAULT FALSE,
    clicked_item_id UUID,
    clicked_position INTEGER,
    click_timestamp TIMESTAMP WITH TIME ZONE,

    -- コンバージョン
    converted BOOLEAN DEFAULT FALSE,
    conversion_type VARCHAR(50),
    conversion_value DECIMAL(12, 2),
    conversion_timestamp TIMESTAMP WITH TIME ZONE,

    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TYPE recommendation_type AS ENUM (
    'PERSONALIZED', 'TRENDING', 'SIMILAR',
    'RECENTLY_VIEWED', 'ALSO_BOOKED', 'POPULAR_IN_AREA'
);

CREATE TYPE recommendation_context AS ENUM (
    'HOME', 'SEARCH', 'DETAIL', 'CART', 'CHECKOUT', 'ITINERARY', 'EMAIL'
);

CREATE INDEX idx_recommendations_log_user_id ON recommendations_log(user_id) WHERE user_id IS NOT NULL;
CREATE INDEX idx_recommendations_log_type ON recommendations_log(recommendation_type);
CREATE INDEX idx_recommendations_log_algorithm ON recommendations_log(algorithm);
CREATE INDEX idx_recommendations_log_created_at ON recommendations_log(created_at DESC);
```

### 7.3 search_history テーブル

```sql
-- 検索履歴テーブル
CREATE TABLE search_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID,
    session_id VARCHAR(100),

    -- 検索情報
    search_type search_type NOT NULL,
    query TEXT,

    -- フィルター
    destination_id UUID,
    check_in_date DATE,
    check_out_date DATE,
    guests INTEGER,
    rooms INTEGER,
    filters JSONB DEFAULT '{}',

    -- 結果
    results_count INTEGER,
    selected_item_id UUID,
    selected_position INTEGER,

    -- デバイス情報
    device_type device_type,

    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_search_history_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE SET NULL
);

CREATE TYPE search_type AS ENUM ('HOTEL', 'ACTIVITY', 'PRODUCT', 'DESTINATION', 'UNIFIED');

CREATE INDEX idx_search_history_user_id ON search_history(user_id) WHERE user_id IS NOT NULL;
CREATE INDEX idx_search_history_type ON search_history(search_type);
CREATE INDEX idx_search_history_destination ON search_history(destination_id) WHERE destination_id IS NOT NULL;
CREATE INDEX idx_search_history_created_at ON search_history(created_at DESC);
```

---

## 第8章：ER図（Entity-Relationship Diagram）

### 8.1 全体ER図

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        TripTrip Database ER Diagram                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         USER DOMAIN                                   │   │
│  │                                                                       │   │
│  │   ┌─────────────┐     1    ┌─────────────────┐                      │   │
│  │   │    users    │─────────>│  user_profiles  │                      │   │
│  │   │             │     1    │                 │                      │   │
│  │   │ PK: id      │          │ FK: user_id     │                      │   │
│  │   └──────┬──────┘          └─────────────────┘                      │   │
│  │          │                                                           │   │
│  │          │ 1                                                         │   │
│  │          │     ┌─────────────────────┐                              │   │
│  │          ├────>│  user_preferences   │                              │   │
│  │          │  1  │                     │                              │   │
│  │          │     │ FK: user_id         │                              │   │
│  │          │     └─────────────────────┘                              │   │
│  │          │                                                           │   │
│  │          │ 1    ┌─────────────────┐                                 │   │
│  │          ├─────>│   auth_tokens   │                                 │   │
│  │          │  *   │                 │                                 │   │
│  │          │      │ FK: user_id     │                                 │   │
│  │          │      └─────────────────┘                                 │   │
│  │          │                                                           │   │
│  │          │ 1    ┌─────────────────┐                                 │   │
│  │          └─────>│  social_logins  │                                 │   │
│  │             *   │                 │                                 │   │
│  │                 │ FK: user_id     │                                 │   │
│  │                 └─────────────────┘                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                       ITINERARY DOMAIN                                │   │
│  │                                                                       │   │
│  │   ┌─────────────┐      1    ┌──────────────────┐                    │   │
│  │   │ itineraries │──────────>│ itinerary_items  │                    │   │
│  │   │             │      *    │                  │                    │   │
│  │   │ FK: user_id │           │ FK: itinerary_id │                    │   │
│  │   │ PK: id      │           │ FK: booking_id   │────┐               │   │
│  │   └──────┬──────┘           └──────────────────┘    │               │   │
│  │          │                                          │               │   │
│  │          │ 1    ┌────────────────────────────┐     │               │   │
│  │          └─────>│ itinerary_collaborators    │     │               │   │
│  │             *   │                            │     │               │   │
│  │                 │ FK: itinerary_id           │     │               │   │
│  │                 │ FK: user_id                │     │               │   │
│  │                 └────────────────────────────┘     │               │   │
│  └────────────────────────────────────────────────────┼───────────────┘   │
│                                                        │                    │
│  ┌─────────────────────────────────────────────────────┼───────────────┐   │
│  │                       BOOKING DOMAIN                │                │   │
│  │                                                     │                │   │
│  │   ┌─────────────┐      1    ┌──────────────────┐   │               │   │
│  │   │  bookings   │<──────────┤                  │<──┘               │   │
│  │   │             │──────────>│  booking_items   │                    │   │
│  │   │ FK: user_id │      *    │                  │                    │   │
│  │   │ PK: id      │           │ FK: booking_id   │                    │   │
│  │   └──────┬──────┘           └──────────────────┘                    │   │
│  │          │                                                           │   │
│  │          │ 1    ┌─────────────────────────────┐                     │   │
│  │          ├─────>│  booking_status_history     │                     │   │
│  │          │  *   │                             │                     │   │
│  │          │      │ FK: booking_id              │                     │   │
│  │          │      └─────────────────────────────┘                     │   │
│  │          │                                                           │   │
│  │          │ 1    ┌─────────────────┐                                 │   │
│  │          └─────>│    payments     │                                 │   │
│  │             *   │                 │                                 │   │
│  │                 │ FK: booking_id  │                                 │   │
│  │                 │ FK: user_id     │                                 │   │
│  │                 └────────┬────────┘                                 │   │
│  │                          │ 1                                         │   │
│  │                          │     ┌─────────────────┐                  │   │
│  │                          └────>│     refunds     │                  │   │
│  │                            *   │                 │                  │   │
│  │                                │ FK: payment_id  │                  │   │
│  │                                └─────────────────┘                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    PRODUCT/SERVICE DOMAIN                             │   │
│  │                                                                       │   │
│  │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐             │   │
│  │   │   hotels    │    │ activities  │    │  products   │             │   │
│  │   │             │    │             │    │             │             │   │
│  │   │ PK: id      │    │ PK: id      │    │ PK: id      │             │   │
│  │   └─────────────┘    └─────────────┘    └─────────────┘             │   │
│  │                                                                       │   │
│  │   ┌─────────────┐    ┌───────────────┐                              │   │
│  │   │  tickets    │    │ kimono_rentals│                              │   │
│  │   │             │    │               │                              │   │
│  │   │ PK: id      │    │ PK: id        │                              │   │
│  │   └─────────────┘    └───────────────┘                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    ANALYTICS DOMAIN                                   │   │
│  │                                                                       │   │
│  │   ┌───────────────────┐  ┌────────────────────┐  ┌───────────────┐  │   │
│  │   │ user_interactions │  │ recommendations_log│  │ search_history│  │   │
│  │   │                   │  │                    │  │               │  │   │
│  │   │ FK: user_id       │  │ FK: user_id        │  │ FK: user_id   │  │   │
│  │   └───────────────────┘  └────────────────────┘  └───────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

Legend:
─────────> : One-to-Many (1:*)
FK: Foreign Key
PK: Primary Key
```

### 8.2 ユーザードメイン詳細ER図

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          USER DOMAIN - Detailed                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                              users                                   │   │
│   ├─────────────────────────────────────────────────────────────────────┤   │
│   │ PK  id                    UUID                                       │   │
│   │     email                 VARCHAR(255)      UNIQUE, NOT NULL         │   │
│   │     email_verified        BOOLEAN           DEFAULT FALSE            │   │
│   │     password_hash         VARCHAR(255)      NULLABLE                 │   │
│   │     name                  VARCHAR(100)      NOT NULL                 │   │
│   │     phone                 VARCHAR(50)       NULLABLE                 │   │
│   │     phone_verified        BOOLEAN           DEFAULT FALSE            │   │
│   │     avatar_url            TEXT              NULLABLE                 │   │
│   │     role                  user_role         DEFAULT 'USER'           │   │
│   │     status                user_status       DEFAULT 'ACTIVE'         │   │
│   │     language              VARCHAR(5)        DEFAULT 'ja'             │   │
│   │     currency              VARCHAR(3)        DEFAULT 'JPY'            │   │
│   │     timezone              VARCHAR(50)       DEFAULT 'Asia/Tokyo'     │   │
│   │     last_login_at         TIMESTAMPTZ       NULLABLE                 │   │
│   │     login_count           INTEGER           DEFAULT 0                │   │
│   │     failed_login_attempts INTEGER           DEFAULT 0                │   │
│   │     locked_until          TIMESTAMPTZ       NULLABLE                 │   │
│   │     created_at            TIMESTAMPTZ       NOT NULL                 │   │
│   │     updated_at            TIMESTAMPTZ       NOT NULL                 │   │
│   │     deleted_at            TIMESTAMPTZ       NULLABLE                 │   │
│   └────────────────────────────────┬────────────────────────────────────┘   │
│                                    │                                        │
│          ┌─────────────────────────┼─────────────────────────┐             │
│          │                         │                         │             │
│          ▼ 1:1                     ▼ 1:1                     ▼ 1:*         │
│   ┌──────────────────┐    ┌───────────────────┐    ┌────────────────┐      │
│   │  user_profiles   │    │ user_preferences  │    │  auth_tokens   │      │
│   ├──────────────────┤    ├───────────────────┤    ├────────────────┤      │
│   │ PK id            │    │ PK id             │    │ PK id          │      │
│   │ FK user_id (UQ)  │    │ FK user_id (UQ)   │    │ FK user_id     │      │
│   │    date_of_birth │    │    notify_email   │    │    token_hash  │      │
│   │    gender        │    │    notify_push    │    │    token_type  │      │
│   │    nationality   │    │    theme          │    │    device_id   │      │
│   │    ...           │    │    ...            │    │    expires_at  │      │
│   └──────────────────┘    └───────────────────┘    └────────────────┘      │
│                                                                              │
│                                    │                                        │
│                                    ▼ 1:*                                    │
│                           ┌────────────────────┐                           │
│                           │   social_logins    │                           │
│                           ├────────────────────┤                           │
│                           │ PK id              │                           │
│                           │ FK user_id         │                           │
│                           │    provider        │                           │
│                           │    provider_user_id│                           │
│                           │    access_token    │                           │
│                           │    ...             │                           │
│                           └────────────────────┘                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 第9章：インデックス設計

### 9.1 主要クエリパターン分析

```yaml
query_patterns:
  user_authentication:
    query: "SELECT * FROM users WHERE email = ? AND deleted_at IS NULL"
    frequency: 非常に高い
    index: idx_users_email (email) WHERE deleted_at IS NULL

  booking_list:
    query: |
      SELECT * FROM bookings
      WHERE user_id = ? AND deleted_at IS NULL
      ORDER BY created_at DESC
    frequency: 高い
    index: idx_bookings_user_id (user_id) WHERE deleted_at IS NULL

  hotel_search:
    query: |
      SELECT * FROM hotels
      WHERE city = ? AND status = 'ACTIVE'
      AND star_rating >= ?
      ORDER BY user_rating DESC
    frequency: 非常に高い
    indexes:
      - idx_hotels_city (city) WHERE status = 'ACTIVE'
      - idx_hotels_star_rating (star_rating) WHERE status = 'ACTIVE'
      - idx_hotels_user_rating (user_rating DESC) WHERE status = 'ACTIVE'

  activity_search:
    query: |
      SELECT * FROM activities
      WHERE category = ? AND city = ? AND status = 'ACTIVE'
      ORDER BY rating DESC
    frequency: 高い
    indexes:
      - idx_activities_category (category) WHERE status = 'ACTIVE'
      - idx_activities_city (city) WHERE status = 'ACTIVE'

  itinerary_by_date:
    query: |
      SELECT * FROM itineraries
      WHERE user_id = ? AND start_date >= ? AND deleted_at IS NULL
      ORDER BY start_date
    frequency: 中程度
    index: idx_itineraries_user_id (user_id) WHERE deleted_at IS NULL

  payment_lookup:
    query: |
      SELECT * FROM payments
      WHERE provider = ? AND provider_payment_id = ?
    frequency: 中程度
    index: idx_payments_provider_id (provider, provider_payment_id)
```

### 9.2 インデックス定義一覧

```sql
-- =====================================
-- USER DOMAIN INDEXES
-- =====================================

-- users table
CREATE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_role ON users(role) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_status ON users(status) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_created_at ON users(created_at);
CREATE INDEX idx_users_last_login ON users(last_login_at DESC) WHERE last_login_at IS NOT NULL;

-- auth_tokens table
CREATE INDEX idx_auth_tokens_user_id ON auth_tokens(user_id);
CREATE INDEX idx_auth_tokens_token_hash ON auth_tokens(token_hash);
CREATE INDEX idx_auth_tokens_expires_at ON auth_tokens(expires_at) WHERE revoked_at IS NULL;

-- social_logins table
CREATE INDEX idx_social_logins_user_id ON social_logins(user_id);
CREATE INDEX idx_social_logins_provider ON social_logins(provider, provider_user_id);

-- =====================================
-- BOOKING DOMAIN INDEXES
-- =====================================

-- bookings table
CREATE INDEX idx_bookings_user_id ON bookings(user_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_bookings_booking_number ON bookings(booking_number);
CREATE INDEX idx_bookings_status ON bookings(status) WHERE deleted_at IS NULL;
CREATE INDEX idx_bookings_type ON bookings(booking_type) WHERE deleted_at IS NULL;
CREATE INDEX idx_bookings_created_at ON bookings(created_at DESC);
CREATE INDEX idx_bookings_confirmed_at ON bookings(confirmed_at DESC) WHERE confirmed_at IS NOT NULL;

-- booking_items table
CREATE INDEX idx_booking_items_booking_id ON booking_items(booking_id);
CREATE INDEX idx_booking_items_item ON booking_items(item_type, item_id);
CREATE INDEX idx_booking_items_check_in ON booking_items(check_in_date) WHERE check_in_date IS NOT NULL;
CREATE INDEX idx_booking_items_activity_date ON booking_items(activity_date) WHERE activity_date IS NOT NULL;

-- =====================================
-- ITINERARY DOMAIN INDEXES
-- =====================================

-- itineraries table
CREATE INDEX idx_itineraries_user_id ON itineraries(user_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_itineraries_status ON itineraries(status) WHERE deleted_at IS NULL;
CREATE INDEX idx_itineraries_start_date ON itineraries(start_date) WHERE deleted_at IS NULL;
CREATE INDEX idx_itineraries_share_token ON itineraries(share_token) WHERE share_token IS NOT NULL;

-- itinerary_items table
CREATE INDEX idx_itinerary_items_itinerary_id ON itinerary_items(itinerary_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_itinerary_items_date ON itinerary_items(itinerary_id, date, order_index);
CREATE INDEX idx_itinerary_items_booking_id ON itinerary_items(booking_id) WHERE booking_id IS NOT NULL;

-- =====================================
-- PRODUCT/SERVICE DOMAIN INDEXES
-- =====================================

-- hotels table
CREATE INDEX idx_hotels_name ON hotels USING gin (name gin_trgm_ops);
CREATE INDEX idx_hotels_city ON hotels(city);
CREATE INDEX idx_hotels_country ON hotels(country_code);
CREATE INDEX idx_hotels_star_rating ON hotels(star_rating) WHERE status = 'ACTIVE';
CREATE INDEX idx_hotels_user_rating ON hotels(user_rating DESC NULLS LAST) WHERE status = 'ACTIVE';
CREATE INDEX idx_hotels_featured ON hotels(is_featured) WHERE is_featured = TRUE AND status = 'ACTIVE';

-- activities table
CREATE INDEX idx_activities_title ON activities USING gin (title gin_trgm_ops);
CREATE INDEX idx_activities_category ON activities(category) WHERE status = 'ACTIVE';
CREATE INDEX idx_activities_city ON activities(city) WHERE status = 'ACTIVE';
CREATE INDEX idx_activities_rating ON activities(rating DESC NULLS LAST) WHERE status = 'ACTIVE';
CREATE INDEX idx_activities_price ON activities(base_price) WHERE status = 'ACTIVE';

-- products table
CREATE INDEX idx_products_name ON products USING gin (name gin_trgm_ops);
CREATE INDEX idx_products_sku ON products(sku) WHERE sku IS NOT NULL;
CREATE INDEX idx_products_category ON products(category) WHERE status = 'ACTIVE';
CREATE INDEX idx_products_price ON products(price) WHERE status = 'ACTIVE';
CREATE INDEX idx_products_slug ON products(slug) WHERE slug IS NOT NULL;

-- destinations table
CREATE INDEX idx_destinations_name ON destinations USING gin (name gin_trgm_ops);
CREATE INDEX idx_destinations_country ON destinations(country_code);
CREATE INDEX idx_destinations_type ON destinations(destination_type);
CREATE INDEX idx_destinations_featured ON destinations(is_featured) WHERE is_featured = TRUE;
CREATE INDEX idx_destinations_popularity ON destinations(popularity_score DESC);

-- =====================================
-- PAYMENT DOMAIN INDEXES
-- =====================================

-- payments table
CREATE INDEX idx_payments_booking_id ON payments(booking_id);
CREATE INDEX idx_payments_user_id ON payments(user_id);
CREATE INDEX idx_payments_status ON payments(status);
CREATE INDEX idx_payments_provider_id ON payments(provider, provider_payment_id);
CREATE INDEX idx_payments_idempotency ON payments(idempotency_key) WHERE idempotency_key IS NOT NULL;
CREATE INDEX idx_payments_created_at ON payments(created_at DESC);

-- refunds table
CREATE INDEX idx_refunds_payment_id ON refunds(payment_id);
CREATE INDEX idx_refunds_booking_id ON refunds(booking_id);
CREATE INDEX idx_refunds_status ON refunds(status);

-- =====================================
-- ANALYTICS DOMAIN INDEXES
-- =====================================

-- user_interactions table
CREATE INDEX idx_user_interactions_user_id ON user_interactions(user_id) WHERE user_id IS NOT NULL;
CREATE INDEX idx_user_interactions_entity ON user_interactions(entity_type, entity_id);
CREATE INDEX idx_user_interactions_type ON user_interactions(interaction_type);
CREATE INDEX idx_user_interactions_created_at ON user_interactions(created_at DESC);
CREATE INDEX idx_user_interactions_session ON user_interactions(session_id) WHERE session_id IS NOT NULL;

-- search_history table
CREATE INDEX idx_search_history_user_id ON search_history(user_id) WHERE user_id IS NOT NULL;
CREATE INDEX idx_search_history_type ON search_history(search_type);
CREATE INDEX idx_search_history_created_at ON search_history(created_at DESC);
```

### 9.3 パフォーマンス最適化

```yaml
performance_optimization:
  partial_indexes:
    description: 条件付きインデックスでストレージと更新コストを削減
    examples:
      - "WHERE deleted_at IS NULL" - 論理削除されたレコードを除外
      - "WHERE status = 'ACTIVE'" - アクティブなレコードのみ
      - "WHERE is_featured = TRUE" - フィーチャーされたアイテムのみ

  gin_indexes:
    description: 全文検索用のGINインデックス
    extension: pg_trgm
    examples:
      - "USING gin (name gin_trgm_ops)" - 部分一致検索
      - "USING gin (to_tsvector('japanese', description))" - 日本語全文検索

  composite_indexes:
    description: 複合インデックス
    guidelines:
      - 選択性の高いカラムを先に配置
      - クエリの WHERE 句と ORDER BY 句を考慮
      - カバリングインデックスで追加ルックアップを削減

  index_maintenance:
    reindex_schedule: 週次（日曜日深夜）
    bloat_threshold: 20%
    monitoring:
      - pg_stat_user_indexes
      - pg_index_size()
      - index_usage_stats

  query_optimization:
    explain_analyze: 定期的なクエリプラン分析
    slow_query_log:
      threshold: 100ms
      log_destination: CloudWatch Logs
    connection_pooling:
      tool: PgBouncer
      pool_mode: transaction
```

---

## 第10章：マイグレーション戦略

### 10.1 Prisma Migrate運用

```yaml
prisma_migrate_workflow:
  development:
    command: "npx prisma migrate dev --name <migration_name>"
    behavior:
      - スキーマ変更を検出
      - マイグレーションファイルを生成
      - 開発DBに適用
      - Prisma Clientを再生成

  staging:
    command: "npx prisma migrate deploy"
    behavior:
      - 保留中のマイグレーションを適用
      - スキーマ変更は行わない
      - 失敗時はロールバック

  production:
    command: "npx prisma migrate deploy"
    pre_conditions:
      - バックアップ完了
      - メンテナンスウィンドウ確保
      - ロールバック手順確認
    post_conditions:
      - ヘルスチェック
      - 主要クエリの動作確認
      - パフォーマンスモニタリング

  naming_convention:
    format: "YYYYMMDDHHMMSS_<action>_<target>"
    examples:
      - "20260121100000_create_users_table"
      - "20260121110000_add_phone_to_users"
      - "20260121120000_create_bookings_index"
      - "20260121130000_alter_payments_add_refund_status"
```

### 10.2 ゼロダウンタイムマイグレーション

```yaml
zero_downtime_migration:
  principles:
    1: "追加してから削除"
    2: "オプショナルにしてから必須に"
    3: "段階的なデータ移行"
    4: "アプリケーションの後方互換性確保"

  patterns:
    add_column:
      step_1:
        description: NULLABLEカラムを追加
        migration: "ALTER TABLE users ADD COLUMN new_column VARCHAR(255)"
        application: 新カラムへの書き込みを開始
      step_2:
        description: 既存データをバックフィル
        migration: "UPDATE users SET new_column = computed_value"
        application: 新旧両方のカラムを読み取り
      step_3:
        description: NOT NULL制約を追加
        migration: "ALTER TABLE users ALTER COLUMN new_column SET NOT NULL"
        application: 新カラムのみを使用

    remove_column:
      step_1:
        description: アプリケーションから参照を削除
        application: カラムへのアクセスを停止
      step_2:
        description: カラムを削除
        migration: "ALTER TABLE users DROP COLUMN old_column"

    rename_column:
      step_1:
        description: 新カラムを追加
        migration: "ALTER TABLE users ADD COLUMN new_name VARCHAR(255)"
      step_2:
        description: データをコピー
        migration: "UPDATE users SET new_name = old_name"
        application: 新旧両方に書き込み
      step_3:
        description: アプリケーションを新カラムに移行
        application: 新カラムのみを使用
      step_4:
        description: 旧カラムを削除
        migration: "ALTER TABLE users DROP COLUMN old_name"

    add_not_null_constraint:
      step_1:
        description: デフォルト値でNULLを埋める
        migration: "UPDATE users SET column = default_value WHERE column IS NULL"
      step_2:
        description: 制約を追加
        migration: "ALTER TABLE users ALTER COLUMN column SET NOT NULL"

  large_table_strategies:
    pt_online_schema_change:
      description: Percona Toolkitを使用したオンラインスキーマ変更
      use_case: 大規模テーブルのカラム追加/変更

    gh_ost:
      description: GitHub Online Schema Changeツール
      use_case: MySQLベースのマイグレーション（将来的なマルチDB対応時）

    batched_updates:
      description: バッチ処理による段階的更新
      batch_size: 10000
      delay_between_batches: 1秒
```

### 10.3 ロールバック手順

```yaml
rollback_procedures:
  prisma_rollback:
    command: "npx prisma migrate resolve --rolled-back <migration_name>"
    manual_steps:
      1: "マイグレーションをロールバック済みとしてマーク"
      2: "手動でDDLを実行（DROP, ALTER等）"
      3: "アプリケーションを前バージョンにデプロイ"

  point_in_time_recovery:
    use_case: データ損失を伴う重大な問題
    steps:
      1: "Cloud SQLでPITRを実行"
      2: "新しいインスタンスを作成"
      3: "アプリケーション接続を切り替え"
    rpo: 1分
    rto: 15分

  emergency_procedures:
    data_corruption:
      1: "即座にトラフィックを停止"
      2: "最新のバックアップを特定"
      3: "リストア手順を実行"
      4: "データ整合性を検証"
      5: "サービスを段階的に復旧"

    performance_degradation:
      1: "問題のあるマイグレーションを特定"
      2: "インデックスまたはスキーマ変更をロールバック"
      3: "クエリプランを分析"
      4: "最適化後に再マイグレーション"
```

---

## 第11章：Prismaスキーマ定義（抜粋）

### 11.1 スキーマ全体構造

```prisma
// schema.prisma
// TripTrip Database Schema

generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["fullTextSearch", "postgresqlExtensions"]
}

datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  extensions = [uuidOssp(map: "uuid-ossp"), pgcrypto, pg_trgm]
}

// =====================================
// USER DOMAIN
// =====================================

model User {
  id                  String          @id @default(uuid())
  email               String          @unique @db.VarChar(255)
  emailVerified       Boolean         @default(false)
  passwordHash        String?         @db.VarChar(255)
  name                String          @db.VarChar(100)
  phone               String?         @db.VarChar(50)
  phoneVerified       Boolean         @default(false)
  avatarUrl           String?
  role                UserRole        @default(USER)
  status              UserStatus      @default(ACTIVE)
  language            String          @default("ja") @db.VarChar(5)
  currency            String          @default("JPY") @db.VarChar(3)
  timezone            String          @default("Asia/Tokyo") @db.VarChar(50)
  lastLoginAt         DateTime?
  loginCount          Int             @default(0)
  failedLoginAttempts Int             @default(0)
  lockedUntil         DateTime?
  createdAt           DateTime        @default(now())
  updatedAt           DateTime        @updatedAt
  deletedAt           DateTime?

  profile             UserProfile?
  preferences         UserPreference?
  authTokens          AuthToken[]
  socialLogins        SocialLogin[]
  bookings            Booking[]
  itineraries         Itinerary[]
  paymentMethods      PaymentMethod[]
  reviews             Review[]

  @@index([email])
  @@index([role])
  @@index([status])
  @@index([createdAt])
  @@map("users")
}

model UserProfile {
  id                   String    @id @default(uuid())
  userId               String    @unique
  dateOfBirth          DateTime? @db.Date
  gender               Gender?
  nationality          String?   @db.VarChar(2)
  passportCountry      String?   @db.VarChar(2)
  passportExpiry       DateTime? @db.Date
  emergencyContactName String?   @db.VarChar(100)
  emergencyContactPhone String?  @db.VarChar(50)
  addressLine1         String?   @db.VarChar(255)
  addressLine2         String?   @db.VarChar(255)
  city                 String?   @db.VarChar(100)
  state                String?   @db.VarChar(100)
  postalCode           String?   @db.VarChar(20)
  country              String?   @db.VarChar(2)
  bio                  String?
  websiteUrl           String?
  createdAt            DateTime  @default(now())
  updatedAt            DateTime  @updatedAt

  user                 User      @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("user_profiles")
}

model UserPreference {
  id                    String              @id @default(uuid())
  userId                String              @unique
  notifyEmail           Boolean             @default(true)
  notifyPush            Boolean             @default(true)
  notifySms             Boolean             @default(false)
  notifyMarketing       Boolean             @default(false)
  notifyBookingReminders Boolean            @default(true)
  notifyPriceAlerts     Boolean             @default(true)
  preferredAirlines     String[]            @default([])
  preferredHotelChains  String[]            @default([])
  preferredRoomType     RoomTypePreference?
  smokingPreference     SmokingPreference?
  dietaryRestrictions   String[]            @default([])
  accessibilityNeeds    String[]            @default([])
  theme                 ThemeType           @default(SYSTEM)
  compactView           Boolean             @default(false)
  showPricesWithTax     Boolean             @default(true)
  metadata              Json                @default("{}")
  createdAt             DateTime            @default(now())
  updatedAt             DateTime            @updatedAt

  user                  User                @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("user_preferences")
}

// =====================================
// BOOKING DOMAIN
// =====================================

model Booking {
  id                 String        @id @default(uuid())
  bookingNumber      String        @unique @db.VarChar(20)
  userId             String
  bookingType        BookingType
  status             BookingStatus @default(PENDING)
  contactName        String        @db.VarChar(100)
  contactEmail       String        @db.VarChar(255)
  contactPhone       String        @db.VarChar(50)
  contactCountry     String?       @db.VarChar(2)
  subtotal           Decimal       @db.Decimal(12, 2)
  discountAmount     Decimal       @default(0) @db.Decimal(12, 2)
  taxAmount          Decimal       @default(0) @db.Decimal(12, 2)
  serviceFee         Decimal       @default(0) @db.Decimal(12, 2)
  totalAmount        Decimal       @db.Decimal(12, 2)
  currency           String        @default("JPY") @db.VarChar(3)
  promoCode          String?       @db.VarChar(50)
  promoDiscountType  DiscountType?
  promoDiscountValue Decimal?      @db.Decimal(10, 2)
  specialRequests    String?
  internalNotes      String?
  metadata           Json          @default("{}")
  confirmedAt        DateTime?
  completedAt        DateTime?
  cancelledAt        DateTime?
  cancellationReason String?
  createdAt          DateTime      @default(now())
  updatedAt          DateTime      @updatedAt
  deletedAt          DateTime?

  user               User          @relation(fields: [userId], references: [id], onDelete: Restrict)
  items              BookingItem[]
  statusHistory      BookingStatusHistory[]
  payments           Payment[]
  itineraryItems     ItineraryItem[]

  @@index([userId])
  @@index([bookingNumber])
  @@index([status])
  @@index([bookingType])
  @@index([createdAt(sort: Desc)])
  @@map("bookings")
}

model BookingItem {
  id                       String            @id @default(uuid())
  bookingId                String
  itemType                 BookingItemType
  itemId                   String
  itemName                 String            @db.VarChar(255)
  itemDescription          String?
  itemImageUrl             String?
  quantity                 Int               @default(1)
  checkInDate              DateTime?         @db.Date
  checkOutDate             DateTime?         @db.Date
  activityDate             DateTime?         @db.Date
  timeSlotId               String?
  timeSlotStart            DateTime?         @db.Time(0)
  timeSlotEnd              DateTime?         @db.Time(0)
  unitPrice                Decimal           @db.Decimal(10, 2)
  totalPrice               Decimal           @db.Decimal(10, 2)
  currency                 String            @default("JPY") @db.VarChar(3)
  guests                   Json              @default("[]")
  options                  Json              @default("[]")
  supplierId               String?
  supplierConfirmationCode String?           @db.VarChar(100)
  supplierNotes            String?
  status                   BookingItemStatus @default(PENDING)
  metadata                 Json              @default("{}")
  createdAt                DateTime          @default(now())
  updatedAt                DateTime          @updatedAt

  booking                  Booking           @relation(fields: [bookingId], references: [id], onDelete: Cascade)

  @@index([bookingId])
  @@index([itemType, itemId])
  @@map("booking_items")
}

// =====================================
// ENUMS
// =====================================

enum UserRole {
  USER
  PREMIUM
  PARTNER
  ADMIN
}

enum UserStatus {
  ACTIVE
  INACTIVE
  SUSPENDED
  DELETED
}

enum Gender {
  MALE
  FEMALE
  OTHER
  PREFER_NOT_TO_SAY
}

enum RoomTypePreference {
  SINGLE
  DOUBLE
  TWIN
  SUITE
  NO_PREFERENCE
}

enum SmokingPreference {
  NON_SMOKING
  SMOKING
  NO_PREFERENCE
}

enum ThemeType {
  LIGHT
  DARK
  SYSTEM
}

enum BookingType {
  HOTEL
  ACTIVITY
  PRODUCT
  PACKAGE
}

enum BookingStatus {
  PENDING
  CONFIRMED
  COMPLETED
  CANCELLED
  REFUNDED
}

enum BookingItemType {
  ROOM
  ACTIVITY
  PRODUCT
  TICKET
}

enum BookingItemStatus {
  PENDING
  CONFIRMED
  CANCELLED
  COMPLETED
}

enum DiscountType {
  PERCENTAGE
  FIXED_AMOUNT
}

enum PaymentProvider {
  STRIPE
  PAYPAY
  APPLE_PAY
  GOOGLE_PAY
}

enum PaymentStatus {
  PENDING
  PROCESSING
  AUTHORIZED
  CAPTURED
  SUCCEEDED
  FAILED
  CANCELLED
  REFUNDED
  PARTIALLY_REFUNDED
}

enum EntityStatus {
  ACTIVE
  INACTIVE
  PENDING
  SUSPENDED
}

enum ItineraryStatus {
  DRAFT
  PLANNED
  ACTIVE
  COMPLETED
  ARCHIVED
}

enum Visibility {
  PRIVATE
  SHARED
  PUBLIC
}
```

---

## 第12章：文書間参照 & 統合ポイント

### 12.1 関連文書マトリックス

```yaml
document_references:
  inputs:
    - document: EXISTING_APP_ANALYSIS.md
      title: 既存アプリケーション分析
      relevance: 現行Prismaスキーマ、データモデル

    - document: Doc-DA-001
      title: Data Architecture Overview
      relevance: データアーキテクチャ概要

    - document: Doc-DA-002
      title: Database Strategy
      relevance: データベース戦略、技術選定

    - document: Doc-SP-001
      title: API Specification
      relevance: API仕様からのデータ要件

  outputs:
    - document: Doc-AD-001
      title: Application Design
      dependency: データモデル、リレーション定義

    - document: Doc-IF-001
      title: Infrastructure Design
      dependency: データベースインスタンス構成

    - document: Doc-SC-001
      title: Security Architecture
      dependency: データ暗号化、アクセス制御
```

### 12.2 データ整合性ガイドライン

```yaml
data_integrity:
  referential_integrity:
    approach: 外部キー制約
    cascade_rules:
      - "ON DELETE CASCADE" - 親削除時に子も削除（user_profiles等）
      - "ON DELETE RESTRICT" - 参照がある場合は削除不可（bookings.user_id等）
      - "ON DELETE SET NULL" - 親削除時にNULLに設定（オプショナル参照）

  check_constraints:
    examples:
      - "CHECK (total_amount >= 0)"
      - "CHECK (end_date >= start_date)"
      - "CHECK (star_rating BETWEEN 1 AND 5)"
      - "CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$')"

  unique_constraints:
    examples:
      - "UNIQUE (email)"
      - "UNIQUE (booking_number)"
      - "UNIQUE (provider, provider_user_id)"

  soft_delete:
    column: deleted_at
    behavior: レコードを物理削除せずタイムスタンプを設定
    query_filter: "WHERE deleted_at IS NULL"
```

---

## 付録

### A. テーブル一覧

| ドメイン | テーブル名 | 説明 |
|----------|-----------|------|
| User | users | ユーザー基本情報 |
| User | user_profiles | ユーザープロフィール詳細 |
| User | user_preferences | ユーザー設定 |
| User | auth_tokens | 認証トークン |
| User | social_logins | ソーシャルログイン |
| Itinerary | itineraries | 旅程 |
| Itinerary | itinerary_items | 旅程アイテム |
| Itinerary | itinerary_collaborators | 旅程コラボレーター |
| Itinerary | destinations | 目的地マスター |
| Booking | bookings | 予約 |
| Booking | booking_items | 予約アイテム |
| Booking | booking_status_history | 予約ステータス履歴 |
| Product | hotels | ホテルマスター |
| Product | activities | アクティビティマスター |
| Product | products | 商品マスター |
| Product | tickets | チケットマスター |
| Product | kimono_rentals | 着物レンタル |
| Payment | payments | 決済 |
| Payment | payment_methods | 支払い方法 |
| Payment | refunds | 返金 |
| Payment | invoices | 請求書 |
| Analytics | user_interactions | ユーザーインタラクション |
| Analytics | recommendations_log | 推奨ログ |
| Analytics | search_history | 検索履歴 |

### B. データ型一覧

| 型 | PostgreSQL | Prisma | 説明 |
|----|------------|--------|------|
| UUID | uuid | String @db.Uuid | 一意識別子 |
| 文字列 | varchar(n) | String @db.VarChar(n) | 可変長文字列 |
| テキスト | text | String | 長いテキスト |
| 整数 | integer | Int | 32ビット整数 |
| 小数 | decimal(p,s) | Decimal @db.Decimal(p,s) | 固定小数点 |
| 真偽値 | boolean | Boolean | true/false |
| 日時 | timestamptz | DateTime | タイムゾーン付き日時 |
| 日付 | date | DateTime @db.Date | 日付のみ |
| 時刻 | time | DateTime @db.Time | 時刻のみ |
| JSON | jsonb | Json | JSONデータ |
| 配列 | text[] | String[] | 配列型 |
| 列挙型 | enum | enum | 列挙型 |

### C. 変更履歴

| バージョン | 日付 | 変更内容 | 作成者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-01-21 | 初版作成 | Technical Architecture Agent |

---

**文書情報**
- 文書ID: Doc-SP-002
- バージョン: 1.0.0
- ステータス: 完了
- 最終更新: 2026-01-21
- 次回レビュー: 2026-02-21

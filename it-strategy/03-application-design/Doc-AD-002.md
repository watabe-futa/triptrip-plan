# Doc-AD-002: TripTrip Webアプリケーションアーキテクチャ

## エグゼクティブサマリー

本文書は、TripTrip Webアプリケーションの包括的なアーキテクチャ設計を定義します。Next.js 14のApp Routerを基盤とし、Google、Airbnb、Booking.comレベルのWeb体験を実現するための技術戦略を提示します。React Server Components（RSC）によるパフォーマンス最適化、Atomic Designに基づくコンポーネント設計、React Query/TanStack Queryによる効率的なデータフェッチング、PWA対応によるモバイルライクな体験を提供します。本設計は、SEO最適化、Core Web Vitals達成、アクセシビリティ準拠を重視し、TripTripの技術ビジョン（Doc-TV-001）およびモバイルアプリアーキテクチャ（Doc-AD-001）との一貫性を保証します。

---

## 第1章：はじめに & コンテキスト

### 1.1 目的 & スコープ

#### 1.1.1 文書の目的

本文書は、TripTrip Webアプリケーションの技術アーキテクチャを包括的に定義し、以下の目的を達成します。

```yaml
document_objectives:
  primary:
    - description: Next.js Webアプリケーションアーキテクチャの標準化
      deliverable: アーキテクチャガイドライン、フォルダ構造規約

    - description: コンポーネント設計パターンの確立
      deliverable: Atomic Design実装、Storybookドキュメント

    - description: パフォーマンス最適化戦略の定義
      deliverable: Core Web Vitals達成計画、最適化手法

  secondary:
    - description: SEO最適化指針の提供
      deliverable: メタデータ戦略、構造化データ設計

    - description: アクセシビリティ準拠の実現
      deliverable: WCAG 2.1 AA準拠チェックリスト

    - description: PWA機能の実装
      deliverable: Service Worker設計、オフライン戦略
```

#### 1.1.2 スコープ定義

```yaml
scope:
  in_scope:
    platforms:
      - Desktop browsers (Chrome, Firefox, Safari, Edge)
      - Mobile browsers (iOS Safari, Android Chrome)
      - Progressive Web App (PWA)

    features:
      - フロントエンドアーキテクチャ設計
      - Server/Client Componentsパターン
      - データフェッチング戦略
      - 状態管理設計
      - コンポーネントライブラリ
      - SEO/パフォーマンス最適化
      - PWA実装
      - テスト戦略

  out_of_scope:
    - モバイルアプリケーション設計（Doc-AD-001で定義）
    - バックエンドAPI設計（Doc-AD-003で定義）
    - インフラストラクチャ設計（Doc-IA-001で定義）
```

### 1.2 Webプラットフォーム要件

#### 1.2.1 ビジネス要件

```yaml
business_requirements:
  user_experience:
    - description: 高速なページロード
      metric: "First Contentful Paint < 1.5s"
      rationale: コンバージョン率向上

    - description: シームレスなナビゲーション
      metric: "ページ遷移 < 200ms"
      rationale: ユーザーエンゲージメント向上

    - description: モバイルファースト対応
      metric: "モバイルトラフィック60%対応"
      rationale: 旅行予約の多くがモバイルから

  seo_requirements:
    - description: 検索エンジン最適化
      metric: "主要キーワードTop10"
      rationale: オーガニックトラフィック獲得

    - description: 構造化データ対応
      metric: "リッチリザルト表示"
      rationale: クリック率向上

  accessibility:
    - description: WCAG 2.1 Level AA準拠
      rationale: 法的要件とインクルーシブデザイン
```

#### 1.2.2 技術要件

```yaml
technical_requirements:
  performance:
    core_web_vitals:
      lcp: "< 2.5s"  # Largest Contentful Paint
      fid: "< 100ms" # First Input Delay
      cls: "< 0.1"   # Cumulative Layout Shift
      inp: "< 200ms" # Interaction to Next Paint

    additional_metrics:
      ttfb: "< 600ms"  # Time to First Byte
      fcp: "< 1.5s"    # First Contentful Paint
      tti: "< 3.5s"    # Time to Interactive

  compatibility:
    browsers:
      - Chrome >= 90
      - Firefox >= 88
      - Safari >= 14
      - Edge >= 90

    features:
      - ES2020+ support
      - CSS Grid/Flexbox
      - Service Worker API
      - Web Share API

  security:
    - Content Security Policy (CSP)
    - HTTPS only
    - XSS protection
    - CSRF protection
```

### 1.3 技術選定基準

#### 1.3.1 フレームワーク選定

```yaml
framework_comparison:
  next_js:
    version: "14.x"
    selected: true
    strengths:
      - React Server Components対応
      - App Router による直感的なルーティング
      - 優れたSSR/SSG/ISRサポート
      - Vercel エコシステムとの統合
      - 画像最適化（next/image）
      - フォント最適化（next/font）
    considerations:
      - 学習コスト（App Router）
      - Vercel依存のリスク

  remix:
    selected: false
    strengths:
      - 優れたデータローディングパターン
      - エラーバウンダリの充実
      - フォーム処理の簡素化
    reasons_not_selected:
      - エコシステムの成熟度
      - チームの経験不足

  astro:
    selected: false
    strengths:
      - 優れた静的サイト生成
      - パフォーマンス重視
    reasons_not_selected:
      - 複雑なインタラクション向きではない
      - Reactとの統合が二次的

decision_rationale: |
  Next.js 14を選定した主な理由：
  1. React Server Componentsによるパフォーマンス最適化
  2. App Routerによる直感的なファイルベースルーティング
  3. Streaming SSRによる段階的ページロード
  4. 既存のFlutterチームのReact学習コストの軽減
  5. 豊富なエコシステムとコミュニティサポート
```

---

## 第2章：フロントエンドアーキテクチャ

### 2.1 Next.js App Router設計

#### 2.1.1 プロジェクト構造

```
triptrip-web/
├── app/                              # App Router ルート
│   ├── (auth)/                       # 認証ルートグループ
│   │   ├── login/
│   │   │   └── page.tsx
│   │   ├── register/
│   │   │   └── page.tsx
│   │   └── layout.tsx
│   │
│   ├── (main)/                       # メインアプリルートグループ
│   │   ├── page.tsx                  # ホームページ
│   │   ├── layout.tsx                # メインレイアウト
│   │   │
│   │   ├── hotels/
│   │   │   ├── page.tsx              # ホテル一覧
│   │   │   ├── [hotelId]/
│   │   │   │   ├── page.tsx          # ホテル詳細
│   │   │   │   ├── loading.tsx       # ローディング状態
│   │   │   │   ├── error.tsx         # エラー状態
│   │   │   │   └── book/
│   │   │   │       └── page.tsx      # 予約ページ
│   │   │   └── loading.tsx
│   │   │
│   │   ├── attractions/
│   │   │   ├── page.tsx
│   │   │   └── [attractionId]/
│   │   │       └── page.tsx
│   │   │
│   │   ├── products/
│   │   │   ├── page.tsx
│   │   │   └── [productId]/
│   │   │       └── page.tsx
│   │   │
│   │   ├── cart/
│   │   │   └── page.tsx
│   │   │
│   │   ├── checkout/
│   │   │   └── page.tsx
│   │   │
│   │   ├── bookings/
│   │   │   ├── page.tsx
│   │   │   └── [bookingId]/
│   │   │       └── page.tsx
│   │   │
│   │   └── profile/
│   │       ├── page.tsx
│   │       └── settings/
│   │           └── page.tsx
│   │
│   ├── api/                          # API Routes
│   │   ├── auth/
│   │   │   └── [...nextauth]/
│   │   │       └── route.ts
│   │   └── webhooks/
│   │       └── stripe/
│   │           └── route.ts
│   │
│   ├── globals.css                   # グローバルスタイル
│   ├── layout.tsx                    # ルートレイアウト
│   ├── not-found.tsx                 # 404ページ
│   ├── error.tsx                     # グローバルエラー
│   ├── loading.tsx                   # グローバルローディング
│   └── sitemap.ts                    # 動的サイトマップ
│
├── components/                       # コンポーネント
│   ├── atoms/                        # 原子コンポーネント
│   │   ├── Button/
│   │   ├── Input/
│   │   ├── Icon/
│   │   └── Typography/
│   │
│   ├── molecules/                    # 分子コンポーネント
│   │   ├── SearchBar/
│   │   ├── DatePicker/
│   │   ├── PriceDisplay/
│   │   └── RatingBadge/
│   │
│   ├── organisms/                    # 有機体コンポーネント
│   │   ├── Header/
│   │   ├── Footer/
│   │   ├── HotelCard/
│   │   ├── SearchForm/
│   │   └── BookingForm/
│   │
│   ├── templates/                    # テンプレート
│   │   ├── MainLayout/
│   │   ├── AuthLayout/
│   │   └── CheckoutLayout/
│   │
│   └── providers/                    # Context Providers
│       ├── QueryProvider.tsx
│       ├── ThemeProvider.tsx
│       └── AuthProvider.tsx
│
├── lib/                              # ユーティリティ・設定
│   ├── api/                          # API クライアント
│   │   ├── client.ts
│   │   ├── hotels.ts
│   │   ├── bookings.ts
│   │   └── types.ts
│   │
│   ├── hooks/                        # カスタムフック
│   │   ├── useHotels.ts
│   │   ├── useBooking.ts
│   │   ├── useAuth.ts
│   │   └── useMediaQuery.ts
│   │
│   ├── utils/                        # ユーティリティ関数
│   │   ├── formatters.ts
│   │   ├── validators.ts
│   │   └── cn.ts                     # className merge
│   │
│   ├── config/                       # 設定
│   │   ├── site.ts
│   │   └── navigation.ts
│   │
│   └── types/                        # 型定義
│       ├── hotel.ts
│       ├── booking.ts
│       └── user.ts
│
├── public/                           # 静的ファイル
│   ├── images/
│   ├── icons/
│   └── fonts/
│
├── styles/                           # スタイル
│   ├── tokens/                       # デザイントークン
│   │   ├── colors.css
│   │   ├── typography.css
│   │   └── spacing.css
│   └── components/                   # コンポーネントスタイル
│
├── tests/                            # テスト
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── .storybook/                       # Storybook設定
├── next.config.js                    # Next.js設定
├── tailwind.config.ts                # Tailwind設定
├── tsconfig.json                     # TypeScript設定
└── package.json
```

#### 2.1.2 ルートグループとレイアウト

```typescript
// ==========================================
// app/layout.tsx - ルートレイアウト
// ==========================================

import { Inter } from 'next/font/google';
import { Metadata, Viewport } from 'next';
import { Providers } from '@/components/providers';
import '@/app/globals.css';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-inter',
});

export const metadata: Metadata = {
  metadataBase: new URL('https://triptrip.com'),
  title: {
    default: 'TripTrip - あなたの旅をもっと特別に',
    template: '%s | TripTrip',
  },
  description: 'ホテル、観光チケット、おみやげまで。TripTripで完璧な旅行体験を。',
  keywords: ['旅行', 'ホテル予約', '観光', 'チケット', '日本旅行'],
  authors: [{ name: 'TripTrip' }],
  creator: 'TripTrip',
  publisher: 'TripTrip',
  formatDetection: {
    email: false,
    telephone: false,
  },
  openGraph: {
    type: 'website',
    locale: 'ja_JP',
    url: 'https://triptrip.com',
    siteName: 'TripTrip',
    title: 'TripTrip - あなたの旅をもっと特別に',
    description: 'ホテル、観光チケット、おみやげまで。TripTripで完璧な旅行体験を。',
    images: [
      {
        url: '/og-image.jpg',
        width: 1200,
        height: 630,
        alt: 'TripTrip',
      },
    ],
  },
  twitter: {
    card: 'summary_large_image',
    title: 'TripTrip - あなたの旅をもっと特別に',
    description: 'ホテル、観光チケット、おみやげまで。TripTripで完璧な旅行体験を。',
    images: ['/twitter-image.jpg'],
    creator: '@triptrip',
  },
  robots: {
    index: true,
    follow: true,
    googleBot: {
      index: true,
      follow: true,
      'max-video-preview': -1,
      'max-image-preview': 'large',
      'max-snippet': -1,
    },
  },
  verification: {
    google: 'google-site-verification-code',
  },
};

export const viewport: Viewport = {
  themeColor: [
    { media: '(prefers-color-scheme: light)', color: '#ffffff' },
    { media: '(prefers-color-scheme: dark)', color: '#0a0a0a' },
  ],
  width: 'device-width',
  initialScale: 1,
  maximumScale: 5,
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ja" className={inter.variable} suppressHydrationWarning>
      <body className="min-h-screen bg-background font-sans antialiased">
        <Providers>
          {children}
        </Providers>
      </body>
    </html>
  );
}

// ==========================================
// app/(main)/layout.tsx - メインレイアウト
// ==========================================

import { Header } from '@/components/organisms/Header';
import { Footer } from '@/components/organisms/Footer';
import { Toaster } from '@/components/atoms/Toaster';

export default function MainLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex min-h-screen flex-col">
      <Header />
      <main className="flex-1">{children}</main>
      <Footer />
      <Toaster />
    </div>
  );
}

// ==========================================
// app/(auth)/layout.tsx - 認証レイアウト
// ==========================================

import { Logo } from '@/components/atoms/Logo';

export default function AuthLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="min-h-screen flex flex-col items-center justify-center bg-gradient-to-b from-primary/5 to-background">
      <div className="w-full max-w-md space-y-8 p-8">
        <div className="flex justify-center">
          <Logo />
        </div>
        {children}
      </div>
    </div>
  );
}
```

### 2.2 サーバーコンポーネント vs クライアントコンポーネント

#### 2.2.1 コンポーネント分類基準

```yaml
component_classification:
  server_components:
    description: サーバーでのみレンダリングされるコンポーネント
    use_cases:
      - データフェッチング
      - バックエンドリソースへの直接アクセス
      - 機密情報の処理
      - 大きな依存関係の使用
      - 静的コンテンツ
    examples:
      - ホテル一覧ページ
      - 商品詳細ページ
      - 検索結果ページ
      - メタデータ生成
    benefits:
      - バンドルサイズ削減
      - 直接データベースアクセス可能
      - 機密データの安全な処理
      - 初期ロード時間の短縮

  client_components:
    description: クライアントでインタラクティブに動作するコンポーネント
    use_cases:
      - ユーザーインタラクション（onClick、onChange）
      - ブラウザAPIの使用
      - React Hooks（useState、useEffect）
      - サードパーティライブラリ（地図、カレンダー）
    examples:
      - 検索フォーム
      - 日付選択
      - カート操作
      - フィルター
      - モーダル・ドロワー
    directive: "'use client'"
```

#### 2.2.2 実装パターン

```typescript
// ==========================================
// Server Component: ホテル一覧ページ
// ==========================================

// app/(main)/hotels/page.tsx
import { Suspense } from 'react';
import { HotelList } from '@/components/organisms/HotelList';
import { HotelListSkeleton } from '@/components/organisms/HotelList/Skeleton';
import { SearchFilters } from '@/components/organisms/SearchFilters';
import { getHotels } from '@/lib/api/hotels';

interface HotelsPageProps {
  searchParams: {
    location?: string;
    checkIn?: string;
    checkOut?: string;
    guests?: string;
    page?: string;
  };
}

// Server Component - データフェッチングをサーバーで実行
export default async function HotelsPage({ searchParams }: HotelsPageProps) {
  // サーバーサイドでデータを取得
  const hotels = await getHotels({
    location: searchParams.location,
    checkIn: searchParams.checkIn,
    checkOut: searchParams.checkOut,
    guests: searchParams.guests ? parseInt(searchParams.guests) : undefined,
    page: searchParams.page ? parseInt(searchParams.page) : 1,
  });

  return (
    <div className="container mx-auto py-8">
      <h1 className="text-3xl font-bold mb-6">ホテルを探す</h1>

      {/* Client Component: インタラクティブなフィルター */}
      <SearchFilters
        initialFilters={{
          location: searchParams.location,
          checkIn: searchParams.checkIn,
          checkOut: searchParams.checkOut,
          guests: searchParams.guests,
        }}
      />

      {/* Server Component: 静的なリスト表示 */}
      <Suspense fallback={<HotelListSkeleton />}>
        <HotelList
          hotels={hotels.items}
          totalCount={hotels.totalCount}
          currentPage={hotels.currentPage}
        />
      </Suspense>
    </div>
  );
}

// メタデータ生成（Server Component のみ）
export async function generateMetadata({ searchParams }: HotelsPageProps) {
  const location = searchParams.location || '日本';

  return {
    title: `${location}のホテル`,
    description: `${location}で人気のホテルを検索・予約。最安値保証でお得に宿泊。`,
  };
}

// ==========================================
// Client Component: 検索フィルター
// ==========================================

// components/organisms/SearchFilters/index.tsx
'use client';

import { useState, useCallback } from 'react';
import { useRouter, useSearchParams } from 'next/navigation';
import { DateRangePicker } from '@/components/molecules/DateRangePicker';
import { GuestSelector } from '@/components/molecules/GuestSelector';
import { LocationInput } from '@/components/molecules/LocationInput';
import { Button } from '@/components/atoms/Button';

interface SearchFiltersProps {
  initialFilters: {
    location?: string;
    checkIn?: string;
    checkOut?: string;
    guests?: string;
  };
}

export function SearchFilters({ initialFilters }: SearchFiltersProps) {
  const router = useRouter();
  const searchParams = useSearchParams();

  const [filters, setFilters] = useState({
    location: initialFilters.location || '',
    checkIn: initialFilters.checkIn || '',
    checkOut: initialFilters.checkOut || '',
    guests: initialFilters.guests || '2',
  });

  const handleSearch = useCallback(() => {
    const params = new URLSearchParams();
    if (filters.location) params.set('location', filters.location);
    if (filters.checkIn) params.set('checkIn', filters.checkIn);
    if (filters.checkOut) params.set('checkOut', filters.checkOut);
    if (filters.guests) params.set('guests', filters.guests);

    router.push(`/hotels?${params.toString()}`);
  }, [filters, router]);

  return (
    <div className="bg-white rounded-xl shadow-lg p-6 mb-8">
      <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
        <LocationInput
          value={filters.location}
          onChange={(value) => setFilters((prev) => ({ ...prev, location: value }))}
        />

        <DateRangePicker
          startDate={filters.checkIn}
          endDate={filters.checkOut}
          onChange={(start, end) =>
            setFilters((prev) => ({ ...prev, checkIn: start, checkOut: end }))
          }
        />

        <GuestSelector
          value={parseInt(filters.guests)}
          onChange={(value) =>
            setFilters((prev) => ({ ...prev, guests: value.toString() }))
          }
        />

        <Button onClick={handleSearch} className="h-full">
          検索
        </Button>
      </div>
    </div>
  );
}

// ==========================================
// Hybrid Pattern: Server + Client 組み合わせ
// ==========================================

// app/(main)/hotels/[hotelId]/page.tsx
import { notFound } from 'next/navigation';
import { getHotelById } from '@/lib/api/hotels';
import { HotelGallery } from '@/components/organisms/HotelGallery';
import { HotelInfo } from '@/components/organisms/HotelInfo';
import { BookingWidget } from '@/components/organisms/BookingWidget';
import { ReviewSection } from '@/components/organisms/ReviewSection';
import { SimilarHotels } from '@/components/organisms/SimilarHotels';

interface HotelDetailPageProps {
  params: { hotelId: string };
  searchParams: { checkIn?: string; checkOut?: string };
}

export default async function HotelDetailPage({
  params,
  searchParams,
}: HotelDetailPageProps) {
  const hotel = await getHotelById(params.hotelId);

  if (!hotel) {
    notFound();
  }

  return (
    <article className="container mx-auto py-8">
      {/* Server Component: 静的な画像ギャラリー */}
      <HotelGallery images={hotel.images} hotelName={hotel.name} />

      <div className="grid grid-cols-1 lg:grid-cols-3 gap-8 mt-8">
        <div className="lg:col-span-2 space-y-8">
          {/* Server Component: ホテル情報 */}
          <HotelInfo hotel={hotel} />

          {/* Server Component with Suspense: レビュー */}
          <Suspense fallback={<ReviewSectionSkeleton />}>
            <ReviewSection hotelId={params.hotelId} />
          </Suspense>
        </div>

        <div className="lg:col-span-1">
          {/* Client Component: 予約ウィジェット */}
          <BookingWidget
            hotelId={params.hotelId}
            initialCheckIn={searchParams.checkIn}
            initialCheckOut={searchParams.checkOut}
            pricePerNight={hotel.pricePerNight}
          />
        </div>
      </div>

      {/* Server Component: 類似ホテル */}
      <Suspense fallback={<SimilarHotelsSkeleton />}>
        <SimilarHotels hotelId={params.hotelId} category={hotel.category} />
      </Suspense>
    </article>
  );
}

// JSON-LD構造化データ
export async function generateMetadata({ params }: HotelDetailPageProps) {
  const hotel = await getHotelById(params.hotelId);

  if (!hotel) return {};

  return {
    title: hotel.name,
    description: hotel.description.slice(0, 160),
    openGraph: {
      title: hotel.name,
      description: hotel.description.slice(0, 160),
      images: hotel.images.map((img) => ({
        url: img.url,
        alt: img.alt,
      })),
    },
    other: {
      'script:ld+json': JSON.stringify({
        '@context': 'https://schema.org',
        '@type': 'Hotel',
        name: hotel.name,
        description: hotel.description,
        image: hotel.images.map((img) => img.url),
        address: {
          '@type': 'PostalAddress',
          streetAddress: hotel.address.street,
          addressLocality: hotel.address.city,
          addressRegion: hotel.address.prefecture,
          postalCode: hotel.address.postalCode,
          addressCountry: 'JP',
        },
        geo: {
          '@type': 'GeoCoordinates',
          latitude: hotel.location.latitude,
          longitude: hotel.location.longitude,
        },
        aggregateRating: {
          '@type': 'AggregateRating',
          ratingValue: hotel.rating,
          reviewCount: hotel.reviewCount,
        },
        priceRange: `¥${hotel.priceRange.min} - ¥${hotel.priceRange.max}`,
      }),
    },
  };
}
```

### 2.3 ルーティングとナビゲーション

#### 2.3.1 ナビゲーションパターン

```typescript
// ==========================================
// lib/config/navigation.ts
// ==========================================

export const navigation = {
  main: [
    { name: 'ホーム', href: '/' },
    { name: 'ホテル', href: '/hotels' },
    { name: 'アトラクション', href: '/attractions' },
    { name: 'おみやげ', href: '/products' },
  ],
  user: [
    { name: 'マイページ', href: '/profile' },
    { name: '予約履歴', href: '/bookings' },
    { name: 'お気に入り', href: '/favorites' },
    { name: '設定', href: '/profile/settings' },
  ],
  footer: [
    {
      title: 'サービス',
      links: [
        { name: 'ホテル予約', href: '/hotels' },
        { name: 'チケット予約', href: '/attractions' },
        { name: 'おみやげ購入', href: '/products' },
      ],
    },
    {
      title: 'サポート',
      links: [
        { name: 'ヘルプセンター', href: '/help' },
        { name: 'よくある質問', href: '/faq' },
        { name: 'お問い合わせ', href: '/contact' },
      ],
    },
    {
      title: '会社情報',
      links: [
        { name: '会社概要', href: '/about' },
        { name: 'プライバシーポリシー', href: '/privacy' },
        { name: '利用規約', href: '/terms' },
      ],
    },
  ],
};

// ==========================================
// components/organisms/Header/Navigation.tsx
// ==========================================

'use client';

import { usePathname } from 'next/navigation';
import Link from 'next/link';
import { cn } from '@/lib/utils/cn';
import { navigation } from '@/lib/config/navigation';

export function Navigation() {
  const pathname = usePathname();

  return (
    <nav className="hidden md:flex items-center space-x-8">
      {navigation.main.map((item) => {
        const isActive = pathname === item.href ||
          (item.href !== '/' && pathname.startsWith(item.href));

        return (
          <Link
            key={item.href}
            href={item.href}
            className={cn(
              'text-sm font-medium transition-colors',
              isActive
                ? 'text-primary'
                : 'text-muted-foreground hover:text-foreground'
            )}
            prefetch={true}
          >
            {item.name}
          </Link>
        );
      })}
    </nav>
  );
}

// ==========================================
// lib/hooks/useNavigation.ts
// ==========================================

'use client';

import { useRouter, usePathname, useSearchParams } from 'next/navigation';
import { useCallback, useTransition } from 'react';

export function useNavigation() {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();
  const [isPending, startTransition] = useTransition();

  const navigate = useCallback(
    (href: string, options?: { scroll?: boolean }) => {
      startTransition(() => {
        router.push(href, options);
      });
    },
    [router]
  );

  const updateSearchParams = useCallback(
    (updates: Record<string, string | null>) => {
      const params = new URLSearchParams(searchParams.toString());

      Object.entries(updates).forEach(([key, value]) => {
        if (value === null) {
          params.delete(key);
        } else {
          params.set(key, value);
        }
      });

      startTransition(() => {
        router.push(`${pathname}?${params.toString()}`);
      });
    },
    [pathname, searchParams, router]
  );

  const goBack = useCallback(() => {
    router.back();
  }, [router]);

  return {
    navigate,
    updateSearchParams,
    goBack,
    isPending,
    pathname,
    searchParams,
  };
}

// ==========================================
// Parallel Routes（並列ルート）
// ==========================================

// app/(main)/@modal/(.)hotels/[hotelId]/quick-view/page.tsx
// モーダルでのクイックビュー表示

export default function HotelQuickViewModal({
  params,
}: {
  params: { hotelId: string };
}) {
  return (
    <Modal>
      <HotelQuickView hotelId={params.hotelId} />
    </Modal>
  );
}

// app/(main)/layout.tsx - 並列ルートを含むレイアウト
export default function MainLayout({
  children,
  modal,
}: {
  children: React.ReactNode;
  modal: React.ReactNode;
}) {
  return (
    <>
      {children}
      {modal}
    </>
  );
}

// ==========================================
// Intercepting Routes（インターセプトルート）
// ==========================================

// app/(main)/@modal/(.)login/page.tsx
// モーダルでのログイン表示（URLは/loginだがモーダルで表示）

'use client';

import { useRouter } from 'next/navigation';
import { Modal } from '@/components/atoms/Modal';
import { LoginForm } from '@/components/organisms/LoginForm';

export default function LoginModal() {
  const router = useRouter();

  return (
    <Modal onClose={() => router.back()}>
      <LoginForm
        onSuccess={() => router.back()}
        onRegisterClick={() => router.push('/register')}
      />
    </Modal>
  );
}
```

---

## 第3章：コンポーネント設計

### 3.1 Atomic Designパターン

#### 3.1.1 コンポーネント階層

```yaml
atomic_design_hierarchy:
  atoms:
    description: 最小単位のUIコンポーネント
    examples:
      - Button
      - Input
      - Label
      - Icon
      - Badge
      - Avatar
      - Spinner
    characteristics:
      - 単一の責任
      - 高い再利用性
      - props経由の設定
      - スタイルバリアント対応

  molecules:
    description: 複数のatomsを組み合わせた機能単位
    examples:
      - SearchBar (Input + Button)
      - DatePicker (Input + Calendar)
      - FormField (Label + Input + ErrorMessage)
      - PriceDisplay (Typography + Badge)
      - RatingStars (Icon × 5)
    characteristics:
      - 単一の機能
      - atomsの組み合わせ
      - 独立して使用可能

  organisms:
    description: 複雑なUIセクション
    examples:
      - Header
      - HotelCard
      - SearchForm
      - BookingForm
      - ReviewCard
      - CartSummary
    characteristics:
      - 複数のmoleculesとatomsで構成
      - ビジネスロジックを含む場合あり
      - ページの主要セクション

  templates:
    description: ページのワイヤーフレーム
    examples:
      - MainLayout
      - AuthLayout
      - CheckoutLayout
      - SearchResultsTemplate
    characteristics:
      - ページ構造を定義
      - コンテンツは子コンポーネントから注入
      - レスポンシブレイアウト

  pages:
    description: 実際のページコンポーネント
    examples:
      - app/(main)/page.tsx
      - app/(main)/hotels/page.tsx
    characteristics:
      - Next.jsのpage.tsx
      - データフェッチング
      - SEOメタデータ
```

#### 3.1.2 Atomコンポーネント実装

```typescript
// ==========================================
// components/atoms/Button/index.tsx
// ==========================================

import { forwardRef } from 'react';
import { Slot } from '@radix-ui/react-slot';
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/lib/utils/cn';

const buttonVariants = cva(
  'inline-flex items-center justify-center whitespace-nowrap rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline: 'border border-input bg-background hover:bg-accent hover:text-accent-foreground',
        secondary: 'bg-secondary text-secondary-foreground hover:bg-secondary/80',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
        link: 'text-primary underline-offset-4 hover:underline',
      },
      size: {
        default: 'h-10 px-4 py-2',
        sm: 'h-9 rounded-md px-3',
        lg: 'h-11 rounded-md px-8',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'default',
    },
  }
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean;
  loading?: boolean;
}

const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, loading = false, children, disabled, ...props }, ref) => {
    const Comp = asChild ? Slot : 'button';
    return (
      <Comp
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        disabled={disabled || loading}
        {...props}
      >
        {loading ? (
          <>
            <Spinner className="mr-2 h-4 w-4" />
            {children}
          </>
        ) : (
          children
        )}
      </Comp>
    );
  }
);
Button.displayName = 'Button';

export { Button, buttonVariants };

// ==========================================
// components/atoms/Input/index.tsx
// ==========================================

import { forwardRef } from 'react';
import { cn } from '@/lib/utils/cn';

export interface InputProps
  extends React.InputHTMLAttributes<HTMLInputElement> {
  error?: boolean;
  startAdornment?: React.ReactNode;
  endAdornment?: React.ReactNode;
}

const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ className, type, error, startAdornment, endAdornment, ...props }, ref) => {
    return (
      <div className="relative flex items-center">
        {startAdornment && (
          <div className="absolute left-3 flex items-center pointer-events-none">
            {startAdornment}
          </div>
        )}
        <input
          type={type}
          className={cn(
            'flex h-10 w-full rounded-md border border-input bg-background px-3 py-2 text-sm ring-offset-background file:border-0 file:bg-transparent file:text-sm file:font-medium placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:cursor-not-allowed disabled:opacity-50',
            startAdornment && 'pl-10',
            endAdornment && 'pr-10',
            error && 'border-destructive focus-visible:ring-destructive',
            className
          )}
          ref={ref}
          {...props}
        />
        {endAdornment && (
          <div className="absolute right-3 flex items-center">
            {endAdornment}
          </div>
        )}
      </div>
    );
  }
);
Input.displayName = 'Input';

export { Input };
```

### 3.2 共通UIコンポーネントライブラリ

#### 3.2.1 Moleculeコンポーネント

```typescript
// ==========================================
// components/molecules/DateRangePicker/index.tsx
// ==========================================

'use client';

import { useState, useCallback } from 'react';
import { format, addDays, differenceInDays } from 'date-fns';
import { ja } from 'date-fns/locale';
import { Calendar } from '@/components/atoms/Calendar';
import { Button } from '@/components/atoms/Button';
import { Popover, PopoverContent, PopoverTrigger } from '@/components/atoms/Popover';
import { CalendarIcon } from '@/components/atoms/Icon';
import { cn } from '@/lib/utils/cn';

interface DateRangePickerProps {
  startDate?: Date;
  endDate?: Date;
  onRangeChange: (start: Date | undefined, end: Date | undefined) => void;
  minDate?: Date;
  maxDate?: Date;
  disabled?: boolean;
  className?: string;
}

export function DateRangePicker({
  startDate,
  endDate,
  onRangeChange,
  minDate = new Date(),
  maxDate = addDays(new Date(), 365),
  disabled = false,
  className,
}: DateRangePickerProps) {
  const [isOpen, setIsOpen] = useState(false);

  const nights = startDate && endDate
    ? differenceInDays(endDate, startDate)
    : 0;

  const formatDateRange = useCallback(() => {
    if (!startDate) return '日付を選択';
    if (!endDate) return format(startDate, 'M月d日', { locale: ja });
    return `${format(startDate, 'M月d日', { locale: ja })} - ${format(endDate, 'M月d日', { locale: ja })}`;
  }, [startDate, endDate]);

  return (
    <Popover open={isOpen} onOpenChange={setIsOpen}>
      <PopoverTrigger asChild>
        <Button
          variant="outline"
          className={cn(
            'w-full justify-start text-left font-normal',
            !startDate && 'text-muted-foreground',
            className
          )}
          disabled={disabled}
        >
          <CalendarIcon className="mr-2 h-4 w-4" />
          <span>{formatDateRange()}</span>
          {nights > 0 && (
            <span className="ml-auto text-xs text-muted-foreground">
              {nights}泊
            </span>
          )}
        </Button>
      </PopoverTrigger>
      <PopoverContent className="w-auto p-0" align="start">
        <Calendar
          mode="range"
          selected={{ from: startDate, to: endDate }}
          onSelect={(range) => {
            onRangeChange(range?.from, range?.to);
            if (range?.to) setIsOpen(false);
          }}
          disabled={(date) => date < minDate || date > maxDate}
          locale={ja}
          numberOfMonths={2}
        />
      </PopoverContent>
    </Popover>
  );
}

// ==========================================
// components/molecules/PriceDisplay/index.tsx
// ==========================================

import { formatCurrency } from '@/lib/utils/formatters';
import { Badge } from '@/components/atoms/Badge';
import { cn } from '@/lib/utils/cn';

interface PriceDisplayProps {
  price: number;
  originalPrice?: number;
  currency?: string;
  unit?: string;
  size?: 'sm' | 'md' | 'lg';
  className?: string;
}

export function PriceDisplay({
  price,
  originalPrice,
  currency = 'JPY',
  unit,
  size = 'md',
  className,
}: PriceDisplayProps) {
  const discount = originalPrice
    ? Math.round((1 - price / originalPrice) * 100)
    : 0;

  const sizeClasses = {
    sm: 'text-sm',
    md: 'text-lg',
    lg: 'text-2xl',
  };

  return (
    <div className={cn('flex items-baseline gap-2', className)}>
      {originalPrice && discount > 0 && (
        <>
          <span className="text-sm text-muted-foreground line-through">
            {formatCurrency(originalPrice, currency)}
          </span>
          <Badge variant="destructive" className="text-xs">
            {discount}%OFF
          </Badge>
        </>
      )}
      <span className={cn('font-bold text-foreground', sizeClasses[size])}>
        {formatCurrency(price, currency)}
      </span>
      {unit && (
        <span className="text-sm text-muted-foreground">/ {unit}</span>
      )}
    </div>
  );
}

// ==========================================
// components/molecules/SearchBar/index.tsx
// ==========================================

'use client';

import { useState, useCallback, useRef } from 'react';
import { useDebounce } from '@/lib/hooks/useDebounce';
import { Input } from '@/components/atoms/Input';
import { Button } from '@/components/atoms/Button';
import { SearchIcon, XIcon } from '@/components/atoms/Icon';
import { Popover, PopoverContent, PopoverAnchor } from '@/components/atoms/Popover';

interface SearchBarProps {
  value?: string;
  onChange?: (value: string) => void;
  onSearch?: (value: string) => void;
  suggestions?: string[];
  onSuggestionSelect?: (suggestion: string) => void;
  placeholder?: string;
  className?: string;
}

export function SearchBar({
  value: controlledValue,
  onChange,
  onSearch,
  suggestions = [],
  onSuggestionSelect,
  placeholder = '検索...',
  className,
}: SearchBarProps) {
  const [internalValue, setInternalValue] = useState('');
  const [isOpen, setIsOpen] = useState(false);
  const inputRef = useRef<HTMLInputElement>(null);

  const value = controlledValue ?? internalValue;
  const debouncedValue = useDebounce(value, 300);

  const handleChange = useCallback((e: React.ChangeEvent<HTMLInputElement>) => {
    const newValue = e.target.value;
    setInternalValue(newValue);
    onChange?.(newValue);
    setIsOpen(newValue.length >= 2);
  }, [onChange]);

  const handleClear = useCallback(() => {
    setInternalValue('');
    onChange?.('');
    setIsOpen(false);
    inputRef.current?.focus();
  }, [onChange]);

  const handleSearch = useCallback(() => {
    onSearch?.(value);
    setIsOpen(false);
  }, [value, onSearch]);

  const handleKeyDown = useCallback((e: React.KeyboardEvent) => {
    if (e.key === 'Enter') {
      handleSearch();
    }
    if (e.key === 'Escape') {
      setIsOpen(false);
    }
  }, [handleSearch]);

  return (
    <Popover open={isOpen && suggestions.length > 0} onOpenChange={setIsOpen}>
      <PopoverAnchor asChild>
        <div className={cn('relative flex items-center', className)}>
          <Input
            ref={inputRef}
            value={value}
            onChange={handleChange}
            onKeyDown={handleKeyDown}
            placeholder={placeholder}
            startAdornment={<SearchIcon className="h-4 w-4 text-muted-foreground" />}
            endAdornment={
              value && (
                <button
                  type="button"
                  onClick={handleClear}
                  className="hover:text-foreground"
                >
                  <XIcon className="h-4 w-4" />
                </button>
              )
            }
            className="pr-20"
          />
          <Button
            onClick={handleSearch}
            className="absolute right-0 rounded-l-none"
          >
            検索
          </Button>
        </div>
      </PopoverAnchor>
      <PopoverContent
        className="w-[var(--radix-popover-trigger-width)] p-0"
        align="start"
        onOpenAutoFocus={(e) => e.preventDefault()}
      >
        <ul className="divide-y">
          {suggestions.map((suggestion, index) => (
            <li key={index}>
              <button
                type="button"
                onClick={() => {
                  onSuggestionSelect?.(suggestion);
                  setIsOpen(false);
                }}
                className="w-full px-4 py-2 text-left hover:bg-accent"
              >
                {suggestion}
              </button>
            </li>
          ))}
        </ul>
      </PopoverContent>
    </Popover>
  );
}
```

### 3.3 Storybookによるドキュメント化

#### 3.3.1 Storybook設定

```typescript
// ==========================================
// .storybook/main.ts
// ==========================================

import type { StorybookConfig } from '@storybook/nextjs';

const config: StorybookConfig = {
  stories: [
    '../components/**/*.stories.@(js|jsx|mjs|ts|tsx)',
    '../components/**/*.mdx',
  ],
  addons: [
    '@storybook/addon-links',
    '@storybook/addon-essentials',
    '@storybook/addon-onboarding',
    '@storybook/addon-interactions',
    '@storybook/addon-a11y',
    '@storybook/addon-viewport',
  ],
  framework: {
    name: '@storybook/nextjs',
    options: {},
  },
  docs: {
    autodocs: 'tag',
  },
  staticDirs: ['../public'],
};

export default config;

// ==========================================
// .storybook/preview.tsx
// ==========================================

import type { Preview } from '@storybook/react';
import { Inter } from 'next/font/google';
import '../app/globals.css';

const inter = Inter({ subsets: ['latin'] });

const preview: Preview = {
  parameters: {
    actions: { argTypesRegex: '^on[A-Z].*' },
    controls: {
      matchers: {
        color: /(background|color)$/i,
        date: /Date$/i,
      },
    },
    viewport: {
      viewports: {
        mobile: {
          name: 'Mobile',
          styles: { width: '375px', height: '667px' },
        },
        tablet: {
          name: 'Tablet',
          styles: { width: '768px', height: '1024px' },
        },
        desktop: {
          name: 'Desktop',
          styles: { width: '1280px', height: '800px' },
        },
      },
    },
    backgrounds: {
      default: 'light',
      values: [
        { name: 'light', value: '#ffffff' },
        { name: 'dark', value: '#0a0a0a' },
      ],
    },
  },
  decorators: [
    (Story) => (
      <div className={inter.className}>
        <Story />
      </div>
    ),
  ],
};

export default preview;

// ==========================================
// components/atoms/Button/Button.stories.tsx
// ==========================================

import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './index';
import { MailIcon, ArrowRightIcon } from '@/components/atoms/Icon';

const meta: Meta<typeof Button> = {
  title: 'Atoms/Button',
  component: Button,
  parameters: {
    layout: 'centered',
    docs: {
      description: {
        component: `
ボタンコンポーネントは、ユーザーアクションを実行するための主要なインタラクション要素です。

## 使用方法

\`\`\`tsx
import { Button } from '@/components/atoms/Button';

<Button variant="default" size="md">
  クリック
</Button>
\`\`\`

## アクセシビリティ

- フォーカス状態の視覚的フィードバック
- キーボード操作対応（Enter/Space）
- aria-disabled属性のサポート
        `,
      },
    },
  },
  tags: ['autodocs'],
  argTypes: {
    variant: {
      control: 'select',
      options: ['default', 'destructive', 'outline', 'secondary', 'ghost', 'link'],
      description: 'ボタンのスタイルバリアント',
    },
    size: {
      control: 'select',
      options: ['default', 'sm', 'lg', 'icon'],
      description: 'ボタンのサイズ',
    },
    disabled: {
      control: 'boolean',
      description: '無効状態',
    },
    loading: {
      control: 'boolean',
      description: 'ローディング状態',
    },
  },
};

export default meta;
type Story = StoryObj<typeof Button>;

export const Default: Story = {
  args: {
    children: 'ボタン',
    variant: 'default',
    size: 'default',
  },
};

export const Variants: Story = {
  render: () => (
    <div className="flex flex-wrap gap-4">
      <Button variant="default">Default</Button>
      <Button variant="secondary">Secondary</Button>
      <Button variant="outline">Outline</Button>
      <Button variant="ghost">Ghost</Button>
      <Button variant="link">Link</Button>
      <Button variant="destructive">Destructive</Button>
    </div>
  ),
};

export const Sizes: Story = {
  render: () => (
    <div className="flex items-center gap-4">
      <Button size="sm">Small</Button>
      <Button size="default">Default</Button>
      <Button size="lg">Large</Button>
      <Button size="icon"><MailIcon className="h-4 w-4" /></Button>
    </div>
  ),
};

export const WithIcon: Story = {
  render: () => (
    <div className="flex gap-4">
      <Button>
        <MailIcon className="mr-2 h-4 w-4" />
        メール送信
      </Button>
      <Button>
        次へ
        <ArrowRightIcon className="ml-2 h-4 w-4" />
      </Button>
    </div>
  ),
};

export const Loading: Story = {
  args: {
    children: '送信中',
    loading: true,
  },
};

export const Disabled: Story = {
  args: {
    children: '無効',
    disabled: true,
  },
};
```

---

## 第4章：データフェッチング & 状態管理

### 4.1 React Query/TanStack Query

#### 4.1.1 クエリ設定

```typescript
// ==========================================
// components/providers/QueryProvider.tsx
// ==========================================

'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { useState } from 'react';

export function QueryProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            // SSRとの互換性
            staleTime: 60 * 1000, // 1分
            gcTime: 5 * 60 * 1000, // 5分
            retry: 1,
            refetchOnWindowFocus: false,
            // エラーハンドリング
            throwOnError: false,
          },
          mutations: {
            retry: 0,
            throwOnError: false,
          },
        },
      })
  );

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      {process.env.NODE_ENV === 'development' && (
        <ReactQueryDevtools initialIsOpen={false} />
      )}
    </QueryClientProvider>
  );
}

// ==========================================
// lib/api/client.ts
// ==========================================

import { z } from 'zod';

const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || 'https://api.triptrip.com';

interface FetchOptions extends RequestInit {
  params?: Record<string, string | number | boolean | undefined>;
}

class ApiError extends Error {
  constructor(
    public status: number,
    message: string,
    public code?: string
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

async function fetchApi<T>(
  endpoint: string,
  options: FetchOptions = {}
): Promise<T> {
  const { params, ...fetchOptions } = options;

  // URLパラメータの構築
  let url = `${API_BASE_URL}${endpoint}`;
  if (params) {
    const searchParams = new URLSearchParams();
    Object.entries(params).forEach(([key, value]) => {
      if (value !== undefined) {
        searchParams.set(key, String(value));
      }
    });
    url += `?${searchParams.toString()}`;
  }

  // 認証トークンの取得（クライアントサイドのみ）
  let authHeader = {};
  if (typeof window !== 'undefined') {
    const token = localStorage.getItem('access_token');
    if (token) {
      authHeader = { Authorization: `Bearer ${token}` };
    }
  }

  const response = await fetch(url, {
    ...fetchOptions,
    headers: {
      'Content-Type': 'application/json',
      ...authHeader,
      ...fetchOptions.headers,
    },
  });

  if (!response.ok) {
    const error = await response.json().catch(() => ({}));
    throw new ApiError(
      response.status,
      error.message || 'An error occurred',
      error.code
    );
  }

  return response.json();
}

export const api = {
  get: <T>(endpoint: string, options?: FetchOptions) =>
    fetchApi<T>(endpoint, { ...options, method: 'GET' }),

  post: <T>(endpoint: string, body: unknown, options?: FetchOptions) =>
    fetchApi<T>(endpoint, {
      ...options,
      method: 'POST',
      body: JSON.stringify(body),
    }),

  put: <T>(endpoint: string, body: unknown, options?: FetchOptions) =>
    fetchApi<T>(endpoint, {
      ...options,
      method: 'PUT',
      body: JSON.stringify(body),
    }),

  patch: <T>(endpoint: string, body: unknown, options?: FetchOptions) =>
    fetchApi<T>(endpoint, {
      ...options,
      method: 'PATCH',
      body: JSON.stringify(body),
    }),

  delete: <T>(endpoint: string, options?: FetchOptions) =>
    fetchApi<T>(endpoint, { ...options, method: 'DELETE' }),
};

// ==========================================
// lib/api/hotels.ts
// ==========================================

import { api } from './client';
import { Hotel, HotelSearchParams, PaginatedResponse } from '@/lib/types';

// サーバーサイド用（直接fetch）
export async function getHotels(params: HotelSearchParams): Promise<PaginatedResponse<Hotel>> {
  const searchParams = new URLSearchParams();
  Object.entries(params).forEach(([key, value]) => {
    if (value !== undefined) {
      searchParams.set(key, String(value));
    }
  });

  const response = await fetch(
    `${process.env.API_URL}/api/v1/hotels?${searchParams.toString()}`,
    {
      next: { revalidate: 60 }, // ISR: 60秒
    }
  );

  if (!response.ok) {
    throw new Error('Failed to fetch hotels');
  }

  return response.json();
}

export async function getHotelById(hotelId: string): Promise<Hotel | null> {
  try {
    const response = await fetch(
      `${process.env.API_URL}/api/v1/hotels/${hotelId}`,
      {
        next: { revalidate: 300 }, // ISR: 5分
      }
    );

    if (!response.ok) {
      if (response.status === 404) return null;
      throw new Error('Failed to fetch hotel');
    }

    return response.json();
  } catch (error) {
    console.error('Error fetching hotel:', error);
    return null;
  }
}

// クライアントサイド用（React Query）
export const hotelKeys = {
  all: ['hotels'] as const,
  lists: () => [...hotelKeys.all, 'list'] as const,
  list: (params: HotelSearchParams) => [...hotelKeys.lists(), params] as const,
  details: () => [...hotelKeys.all, 'detail'] as const,
  detail: (id: string) => [...hotelKeys.details(), id] as const,
  availability: (id: string, checkIn: string, checkOut: string) =>
    [...hotelKeys.detail(id), 'availability', checkIn, checkOut] as const,
};

export function useHotels(params: HotelSearchParams) {
  return useQuery({
    queryKey: hotelKeys.list(params),
    queryFn: () => api.get<PaginatedResponse<Hotel>>('/api/v1/hotels', { params }),
    staleTime: 60 * 1000,
  });
}

export function useHotel(hotelId: string) {
  return useQuery({
    queryKey: hotelKeys.detail(hotelId),
    queryFn: () => api.get<Hotel>(`/api/v1/hotels/${hotelId}`),
    staleTime: 5 * 60 * 1000,
  });
}

export function useHotelAvailability(
  hotelId: string,
  checkIn: string,
  checkOut: string
) {
  return useQuery({
    queryKey: hotelKeys.availability(hotelId, checkIn, checkOut),
    queryFn: () =>
      api.get<HotelAvailability>(`/api/v1/hotels/${hotelId}/availability`, {
        params: { checkIn, checkOut },
      }),
    enabled: !!checkIn && !!checkOut,
    staleTime: 30 * 1000, // 30秒
  });
}
```

### 4.2 サーバー状態 vs クライアント状態

#### 4.2.1 状態分類と管理

```typescript
// ==========================================
// lib/store/ui-store.ts (Zustand)
// ==========================================

import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface UIState {
  // モーダル状態
  isSearchModalOpen: boolean;
  isCartDrawerOpen: boolean;
  isMobileMenuOpen: boolean;

  // UI設定
  currency: string;
  language: string;
  theme: 'light' | 'dark' | 'system';

  // アクション
  openSearchModal: () => void;
  closeSearchModal: () => void;
  toggleCartDrawer: () => void;
  toggleMobileMenu: () => void;
  setCurrency: (currency: string) => void;
  setLanguage: (language: string) => void;
  setTheme: (theme: 'light' | 'dark' | 'system') => void;
}

export const useUIStore = create<UIState>()(
  persist(
    (set) => ({
      // 初期状態
      isSearchModalOpen: false,
      isCartDrawerOpen: false,
      isMobileMenuOpen: false,
      currency: 'JPY',
      language: 'ja',
      theme: 'system',

      // アクション
      openSearchModal: () => set({ isSearchModalOpen: true }),
      closeSearchModal: () => set({ isSearchModalOpen: false }),
      toggleCartDrawer: () =>
        set((state) => ({ isCartDrawerOpen: !state.isCartDrawerOpen })),
      toggleMobileMenu: () =>
        set((state) => ({ isMobileMenuOpen: !state.isMobileMenuOpen })),
      setCurrency: (currency) => set({ currency }),
      setLanguage: (language) => set({ language }),
      setTheme: (theme) => set({ theme }),
    }),
    {
      name: 'triptrip-ui-storage',
      partialize: (state) => ({
        currency: state.currency,
        language: state.language,
        theme: state.theme,
      }),
    }
  )
);

// ==========================================
// lib/store/search-store.ts
// ==========================================

import { create } from 'zustand';

interface SearchFilters {
  location: string;
  checkIn: Date | null;
  checkOut: Date | null;
  guests: number;
  rooms: number;
  priceRange: [number, number];
  amenities: string[];
  rating: number | null;
  sortBy: 'relevance' | 'price_asc' | 'price_desc' | 'rating';
}

interface SearchState {
  filters: SearchFilters;
  recentSearches: SearchFilters[];

  // アクション
  setFilters: (filters: Partial<SearchFilters>) => void;
  resetFilters: () => void;
  addRecentSearch: (search: SearchFilters) => void;
}

const defaultFilters: SearchFilters = {
  location: '',
  checkIn: null,
  checkOut: null,
  guests: 2,
  rooms: 1,
  priceRange: [0, 100000],
  amenities: [],
  rating: null,
  sortBy: 'relevance',
};

export const useSearchStore = create<SearchState>((set) => ({
  filters: defaultFilters,
  recentSearches: [],

  setFilters: (newFilters) =>
    set((state) => ({
      filters: { ...state.filters, ...newFilters },
    })),

  resetFilters: () => set({ filters: defaultFilters }),

  addRecentSearch: (search) =>
    set((state) => ({
      recentSearches: [search, ...state.recentSearches.slice(0, 4)],
    })),
}));

// ==========================================
// lib/hooks/useCart.ts (React Query + Zustand)
// ==========================================

import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query';
import { api } from '@/lib/api/client';
import { Cart, CartItem } from '@/lib/types';

export const cartKeys = {
  all: ['cart'] as const,
  detail: () => [...cartKeys.all, 'detail'] as const,
};

export function useCart() {
  return useQuery({
    queryKey: cartKeys.detail(),
    queryFn: () => api.get<Cart>('/api/v1/cart'),
    staleTime: 0, // 常に最新を取得
  });
}

export function useAddToCart() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (item: Omit<CartItem, 'id'>) =>
      api.post<Cart>('/api/v1/cart/items', item),

    // 楽観的更新
    onMutate: async (newItem) => {
      await queryClient.cancelQueries({ queryKey: cartKeys.detail() });

      const previousCart = queryClient.getQueryData<Cart>(cartKeys.detail());

      queryClient.setQueryData<Cart>(cartKeys.detail(), (old) => {
        if (!old) return old;
        return {
          ...old,
          items: [...old.items, { ...newItem, id: 'temp-id' }],
          totalItems: old.totalItems + newItem.quantity,
          subtotal: old.subtotal + newItem.price * newItem.quantity,
        };
      });

      return { previousCart };
    },

    onError: (err, newItem, context) => {
      if (context?.previousCart) {
        queryClient.setQueryData(cartKeys.detail(), context.previousCart);
      }
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: cartKeys.detail() });
    },
  });
}

export function useUpdateCartItem() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ itemId, quantity }: { itemId: string; quantity: number }) =>
      api.patch<Cart>(`/api/v1/cart/items/${itemId}`, { quantity }),

    onMutate: async ({ itemId, quantity }) => {
      await queryClient.cancelQueries({ queryKey: cartKeys.detail() });

      const previousCart = queryClient.getQueryData<Cart>(cartKeys.detail());

      queryClient.setQueryData<Cart>(cartKeys.detail(), (old) => {
        if (!old) return old;
        const items = old.items.map((item) =>
          item.id === itemId ? { ...item, quantity } : item
        );
        return {
          ...old,
          items,
          totalItems: items.reduce((sum, item) => sum + item.quantity, 0),
          subtotal: items.reduce((sum, item) => sum + item.price * item.quantity, 0),
        };
      });

      return { previousCart };
    },

    onError: (err, variables, context) => {
      if (context?.previousCart) {
        queryClient.setQueryData(cartKeys.detail(), context.previousCart);
      }
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: cartKeys.detail() });
    },
  });
}

export function useRemoveFromCart() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (itemId: string) =>
      api.delete<Cart>(`/api/v1/cart/items/${itemId}`),

    onMutate: async (itemId) => {
      await queryClient.cancelQueries({ queryKey: cartKeys.detail() });

      const previousCart = queryClient.getQueryData<Cart>(cartKeys.detail());

      queryClient.setQueryData<Cart>(cartKeys.detail(), (old) => {
        if (!old) return old;
        const items = old.items.filter((item) => item.id !== itemId);
        return {
          ...old,
          items,
          totalItems: items.reduce((sum, item) => sum + item.quantity, 0),
          subtotal: items.reduce((sum, item) => sum + item.price * item.quantity, 0),
        };
      });

      return { previousCart };
    },

    onError: (err, itemId, context) => {
      if (context?.previousCart) {
        queryClient.setQueryData(cartKeys.detail(), context.previousCart);
      }
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: cartKeys.detail() });
    },
  });
}
```

### 4.3 オプティミスティック更新

```typescript
// ==========================================
// lib/hooks/useFavorites.ts
// ==========================================

import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query';
import { api } from '@/lib/api/client';
import { Hotel } from '@/lib/types';
import { toast } from '@/components/atoms/Toast';

export const favoriteKeys = {
  all: ['favorites'] as const,
  list: () => [...favoriteKeys.all, 'list'] as const,
  check: (hotelId: string) => [...favoriteKeys.all, 'check', hotelId] as const,
};

export function useFavorites() {
  return useQuery({
    queryKey: favoriteKeys.list(),
    queryFn: () => api.get<Hotel[]>('/api/v1/favorites'),
  });
}

export function useIsFavorite(hotelId: string) {
  const { data: favorites } = useFavorites();
  return favorites?.some((hotel) => hotel.id === hotelId) ?? false;
}

export function useToggleFavorite() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({
      hotelId,
      isFavorite,
    }: {
      hotelId: string;
      isFavorite: boolean;
    }) => {
      if (isFavorite) {
        await api.delete(`/api/v1/favorites/${hotelId}`);
      } else {
        await api.post(`/api/v1/favorites/${hotelId}`, {});
      }
    },

    onMutate: async ({ hotelId, isFavorite }) => {
      // 進行中のクエリをキャンセル
      await queryClient.cancelQueries({ queryKey: favoriteKeys.list() });

      // 前の状態を保存
      const previousFavorites = queryClient.getQueryData<Hotel[]>(
        favoriteKeys.list()
      );

      // 楽観的更新
      queryClient.setQueryData<Hotel[]>(favoriteKeys.list(), (old) => {
        if (!old) return old;

        if (isFavorite) {
          // お気に入りから削除
          return old.filter((hotel) => hotel.id !== hotelId);
        } else {
          // お気に入りに追加（詳細は後でサーバーから取得）
          const hotelDetail = queryClient.getQueryData<Hotel>([
            'hotels',
            'detail',
            hotelId,
          ]);
          if (hotelDetail) {
            return [...old, hotelDetail];
          }
          return old;
        }
      });

      return { previousFavorites };
    },

    onError: (err, variables, context) => {
      // エラー時はロールバック
      if (context?.previousFavorites) {
        queryClient.setQueryData(favoriteKeys.list(), context.previousFavorites);
      }
      toast.error('お気に入りの更新に失敗しました');
    },

    onSuccess: (_, { isFavorite }) => {
      if (isFavorite) {
        toast.success('お気に入りから削除しました');
      } else {
        toast.success('お気に入りに追加しました');
      }
    },

    onSettled: () => {
      // 最終的にサーバーと同期
      queryClient.invalidateQueries({ queryKey: favoriteKeys.list() });
    },
  });
}

// 使用例
// components/organisms/HotelCard/FavoriteButton.tsx
'use client';

import { HeartIcon } from '@/components/atoms/Icon';
import { Button } from '@/components/atoms/Button';
import { useIsFavorite, useToggleFavorite } from '@/lib/hooks/useFavorites';
import { cn } from '@/lib/utils/cn';

interface FavoriteButtonProps {
  hotelId: string;
  className?: string;
}

export function FavoriteButton({ hotelId, className }: FavoriteButtonProps) {
  const isFavorite = useIsFavorite(hotelId);
  const { mutate: toggleFavorite, isPending } = useToggleFavorite();

  return (
    <Button
      variant="ghost"
      size="icon"
      className={className}
      disabled={isPending}
      onClick={(e) => {
        e.preventDefault();
        e.stopPropagation();
        toggleFavorite({ hotelId, isFavorite });
      }}
      aria-label={isFavorite ? 'お気に入りから削除' : 'お気に入りに追加'}
    >
      <HeartIcon
        className={cn(
          'h-5 w-5 transition-colors',
          isFavorite ? 'fill-red-500 text-red-500' : 'text-muted-foreground'
        )}
      />
    </Button>
  );
}
```

---

## 第5章：パフォーマンス & SEO

### 5.1 Core Web Vitals最適化

#### 5.1.1 パフォーマンス最適化戦略

```typescript
// ==========================================
// next.config.js
// ==========================================

/** @type {import('next').NextConfig} */
const nextConfig = {
  // 画像最適化
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'images.triptrip.com',
      },
      {
        protocol: 'https',
        hostname: 'cdn.triptrip.com',
      },
    ],
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920, 2048, 3840],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
  },

  // 実験的機能
  experimental: {
    optimizePackageImports: [
      '@radix-ui/react-icons',
      'lucide-react',
      'date-fns',
    ],
  },

  // ヘッダー設定
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'X-DNS-Prefetch-Control',
            value: 'on',
          },
          {
            key: 'Strict-Transport-Security',
            value: 'max-age=63072000; includeSubDomains; preload',
          },
          {
            key: 'X-Content-Type-Options',
            value: 'nosniff',
          },
          {
            key: 'X-Frame-Options',
            value: 'DENY',
          },
          {
            key: 'X-XSS-Protection',
            value: '1; mode=block',
          },
        ],
      },
      {
        source: '/fonts/(.*)',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=31536000, immutable',
          },
        ],
      },
    ];
  },

  // リダイレクト
  async redirects() {
    return [
      {
        source: '/hotel/:id',
        destination: '/hotels/:id',
        permanent: true,
      },
    ];
  },
};

module.exports = nextConfig;

// ==========================================
// 画像最適化コンポーネント
// ==========================================

// components/atoms/OptimizedImage/index.tsx
import Image, { ImageProps } from 'next/image';
import { useState } from 'react';
import { cn } from '@/lib/utils/cn';

interface OptimizedImageProps extends Omit<ImageProps, 'onLoad'> {
  fallback?: string;
}

export function OptimizedImage({
  src,
  alt,
  className,
  fallback = '/images/placeholder.jpg',
  ...props
}: OptimizedImageProps) {
  const [isLoading, setIsLoading] = useState(true);
  const [hasError, setHasError] = useState(false);

  return (
    <div className={cn('relative overflow-hidden', className)}>
      <Image
        src={hasError ? fallback : src}
        alt={alt}
        className={cn(
          'duration-700 ease-in-out',
          isLoading
            ? 'scale-110 blur-2xl grayscale'
            : 'scale-100 blur-0 grayscale-0'
        )}
        onLoad={() => setIsLoading(false)}
        onError={() => {
          setHasError(true);
          setIsLoading(false);
        }}
        {...props}
      />
    </div>
  );
}

// ==========================================
// コード分割とレイジーローディング
// ==========================================

// components/organisms/HotelMap/index.tsx
import dynamic from 'next/dynamic';
import { Skeleton } from '@/components/atoms/Skeleton';

// 地図コンポーネントを動的インポート
const MapComponent = dynamic(
  () => import('./MapComponent').then((mod) => mod.MapComponent),
  {
    loading: () => <Skeleton className="h-[400px] w-full" />,
    ssr: false, // 地図はクライアントサイドのみ
  }
);

export function HotelMap({ hotels }: { hotels: Hotel[] }) {
  return (
    <div className="h-[400px] w-full rounded-lg overflow-hidden">
      <MapComponent hotels={hotels} />
    </div>
  );
}

// ==========================================
// プリロードとプリフェッチ
// ==========================================

// app/(main)/hotels/page.tsx
import { prefetchQuery } from '@tanstack/react-query';

export default async function HotelsPage({ searchParams }) {
  // 人気ホテルをプリフェッチ
  await Promise.all([
    // ホテルデータをプリフェッチ
    prefetchQuery({
      queryKey: ['hotels', 'popular'],
      queryFn: () => getPopularHotels(),
    }),
  ]);

  return (
    <>
      {/* リンクのプリロード */}
      <link
        rel="preload"
        href="/api/v1/hotels/suggestions"
        as="fetch"
        crossOrigin="anonymous"
      />

      {/* コンテンツ */}
      <HotelListContent />
    </>
  );
}
```

### 5.2 画像最適化（next/image）

```typescript
// ==========================================
// components/organisms/HotelGallery/index.tsx
// ==========================================

'use client';

import { useState } from 'react';
import Image from 'next/image';
import { Dialog, DialogContent } from '@/components/atoms/Dialog';
import { ChevronLeftIcon, ChevronRightIcon } from '@/components/atoms/Icon';
import { cn } from '@/lib/utils/cn';

interface HotelGalleryProps {
  images: Array<{
    url: string;
    alt: string;
    width?: number;
    height?: number;
  }>;
  hotelName: string;
}

export function HotelGallery({ images, hotelName }: HotelGalleryProps) {
  const [selectedIndex, setSelectedIndex] = useState<number | null>(null);

  const mainImage = images[0];
  const thumbnails = images.slice(1, 5);
  const remainingCount = images.length - 5;

  return (
    <>
      <div className="grid grid-cols-4 gap-2 h-[400px]">
        {/* メイン画像 */}
        <button
          onClick={() => setSelectedIndex(0)}
          className="col-span-2 row-span-2 relative overflow-hidden rounded-l-xl"
        >
          <Image
            src={mainImage.url}
            alt={`${hotelName} - メイン画像`}
            fill
            className="object-cover hover:scale-105 transition-transform duration-300"
            priority // LCP最適化: メイン画像は優先ロード
            sizes="(max-width: 768px) 100vw, 50vw"
            quality={85}
          />
        </button>

        {/* サムネイル */}
        {thumbnails.map((image, index) => (
          <button
            key={image.url}
            onClick={() => setSelectedIndex(index + 1)}
            className={cn(
              'relative overflow-hidden',
              index === 1 && 'rounded-tr-xl',
              index === 3 && 'rounded-br-xl'
            )}
          >
            <Image
              src={image.url}
              alt={`${hotelName} - 画像${index + 2}`}
              fill
              className="object-cover hover:scale-105 transition-transform duration-300"
              sizes="(max-width: 768px) 50vw, 25vw"
              loading="lazy" // サムネイルは遅延ロード
              quality={75}
            />
            {index === 3 && remainingCount > 0 && (
              <div className="absolute inset-0 bg-black/50 flex items-center justify-center">
                <span className="text-white text-lg font-semibold">
                  +{remainingCount}
                </span>
              </div>
            )}
          </button>
        ))}
      </div>

      {/* ライトボックス */}
      <Dialog
        open={selectedIndex !== null}
        onOpenChange={() => setSelectedIndex(null)}
      >
        <DialogContent className="max-w-5xl p-0 bg-black">
          {selectedIndex !== null && (
            <div className="relative h-[80vh]">
              <Image
                src={images[selectedIndex].url}
                alt={`${hotelName} - 画像${selectedIndex + 1}`}
                fill
                className="object-contain"
                quality={90}
                priority
              />

              {/* ナビゲーション */}
              <button
                onClick={() =>
                  setSelectedIndex((prev) =>
                    prev !== null && prev > 0 ? prev - 1 : images.length - 1
                  )
                }
                className="absolute left-4 top-1/2 -translate-y-1/2 p-2 bg-white/80 rounded-full hover:bg-white"
              >
                <ChevronLeftIcon className="h-6 w-6" />
              </button>
              <button
                onClick={() =>
                  setSelectedIndex((prev) =>
                    prev !== null && prev < images.length - 1 ? prev + 1 : 0
                  )
                }
                className="absolute right-4 top-1/2 -translate-y-1/2 p-2 bg-white/80 rounded-full hover:bg-white"
              >
                <ChevronRightIcon className="h-6 w-6" />
              </button>

              {/* インジケーター */}
              <div className="absolute bottom-4 left-1/2 -translate-x-1/2 flex gap-2">
                {images.map((_, index) => (
                  <button
                    key={index}
                    onClick={() => setSelectedIndex(index)}
                    className={cn(
                      'w-2 h-2 rounded-full transition-colors',
                      index === selectedIndex ? 'bg-white' : 'bg-white/50'
                    )}
                  />
                ))}
              </div>
            </div>
          )}
        </DialogContent>
      </Dialog>
    </>
  );
}
```

### 5.3 SEOメタデータ戦略

```typescript
// ==========================================
// app/(main)/hotels/[hotelId]/page.tsx - SEO最適化
// ==========================================

import { Metadata } from 'next';
import { getHotelById } from '@/lib/api/hotels';

interface Props {
  params: { hotelId: string };
}

// 動的メタデータ生成
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const hotel = await getHotelById(params.hotelId);

  if (!hotel) {
    return {
      title: 'ホテルが見つかりません',
    };
  }

  const title = `${hotel.name} - ${hotel.location.city}のホテル`;
  const description = hotel.description.slice(0, 160);

  return {
    title,
    description,
    alternates: {
      canonical: `https://triptrip.com/hotels/${hotel.id}`,
      languages: {
        'ja-JP': `https://triptrip.com/ja/hotels/${hotel.id}`,
        'en-US': `https://triptrip.com/en/hotels/${hotel.id}`,
      },
    },
    openGraph: {
      title,
      description,
      url: `https://triptrip.com/hotels/${hotel.id}`,
      siteName: 'TripTrip',
      images: hotel.images.slice(0, 4).map((img) => ({
        url: img.url,
        width: 1200,
        height: 630,
        alt: img.alt || hotel.name,
      })),
      locale: 'ja_JP',
      type: 'website',
    },
    twitter: {
      card: 'summary_large_image',
      title,
      description,
      images: [hotel.images[0]?.url],
    },
    robots: {
      index: true,
      follow: true,
    },
  };
}

// JSON-LD構造化データ
function generateJsonLd(hotel: Hotel) {
  return {
    '@context': 'https://schema.org',
    '@type': 'Hotel',
    '@id': `https://triptrip.com/hotels/${hotel.id}`,
    name: hotel.name,
    description: hotel.description,
    url: `https://triptrip.com/hotels/${hotel.id}`,
    image: hotel.images.map((img) => img.url),
    address: {
      '@type': 'PostalAddress',
      streetAddress: hotel.address.street,
      addressLocality: hotel.address.city,
      addressRegion: hotel.address.prefecture,
      postalCode: hotel.address.postalCode,
      addressCountry: 'JP',
    },
    geo: {
      '@type': 'GeoCoordinates',
      latitude: hotel.location.latitude,
      longitude: hotel.location.longitude,
    },
    telephone: hotel.phone,
    priceRange: `¥${hotel.priceRange.min.toLocaleString()} - ¥${hotel.priceRange.max.toLocaleString()}`,
    aggregateRating: hotel.rating
      ? {
          '@type': 'AggregateRating',
          ratingValue: hotel.rating.toFixed(1),
          reviewCount: hotel.reviewCount,
          bestRating: '5',
          worstRating: '1',
        }
      : undefined,
    amenityFeature: hotel.amenities.map((amenity) => ({
      '@type': 'LocationFeatureSpecification',
      name: amenity,
      value: true,
    })),
    checkinTime: hotel.checkInTime,
    checkoutTime: hotel.checkOutTime,
    petsAllowed: hotel.petsAllowed,
    starRating: hotel.starRating
      ? {
          '@type': 'Rating',
          ratingValue: hotel.starRating,
        }
      : undefined,
  };
}

export default async function HotelDetailPage({ params }: Props) {
  const hotel = await getHotelById(params.hotelId);

  if (!hotel) {
    notFound();
  }

  return (
    <>
      {/* JSON-LD構造化データ */}
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{
          __html: JSON.stringify(generateJsonLd(hotel)),
        }}
      />

      {/* ページコンテンツ */}
      <article>
        {/* ... */}
      </article>
    </>
  );
}

// ==========================================
// app/sitemap.ts - 動的サイトマップ
// ==========================================

import { MetadataRoute } from 'next';

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const baseUrl = 'https://triptrip.com';

  // 静的ページ
  const staticPages = [
    '',
    '/hotels',
    '/attractions',
    '/products',
    '/about',
    '/help',
    '/contact',
  ].map((route) => ({
    url: `${baseUrl}${route}`,
    lastModified: new Date(),
    changeFrequency: 'daily' as const,
    priority: route === '' ? 1 : 0.8,
  }));

  // 動的ページ（ホテル）
  const hotels = await fetch(`${process.env.API_URL}/api/v1/hotels/sitemap`)
    .then((res) => res.json())
    .catch(() => []);

  const hotelPages = hotels.map((hotel: { id: string; updatedAt: string }) => ({
    url: `${baseUrl}/hotels/${hotel.id}`,
    lastModified: new Date(hotel.updatedAt),
    changeFrequency: 'weekly' as const,
    priority: 0.6,
  }));

  return [...staticPages, ...hotelPages];
}

// ==========================================
// app/robots.ts - robots.txt
// ==========================================

import { MetadataRoute } from 'next';

export default function robots(): MetadataRoute.Robots {
  return {
    rules: [
      {
        userAgent: '*',
        allow: '/',
        disallow: ['/api/', '/admin/', '/checkout/', '/profile/'],
      },
    ],
    sitemap: 'https://triptrip.com/sitemap.xml',
  };
}
```

---

## 第6章：実装ロードマップ & 文書間参照

### 6.1 実装フェーズ

```yaml
implementation_phases:
  phase_1:
    name: 基盤構築
    duration: 3週間
    deliverables:
      - Next.js 14プロジェクトセットアップ
      - Tailwind CSS + shadcn/ui設定
      - コンポーネントライブラリ基盤（Atoms）
      - Storybook設定
      - 基本的なルーティング構造
    dependencies: []

  phase_2:
    name: コアページ実装
    duration: 4週間
    deliverables:
      - ホームページ
      - ホテル一覧・詳細ページ
      - 検索機能
      - 認証フロー（ログイン/登録）
    dependencies: [phase_1]

  phase_3:
    name: 商取引機能
    duration: 4週間
    deliverables:
      - カート機能
      - チェックアウトフロー
      - 予約管理
      - 注文履歴
    dependencies: [phase_2]

  phase_4:
    name: PWA・最適化
    duration: 3週間
    deliverables:
      - Service Worker実装
      - オフライン対応
      - プッシュ通知
      - パフォーマンス最適化
    dependencies: [phase_3]

  phase_5:
    name: テスト・品質保証
    duration: 3週間
    deliverables:
      - ユニットテスト（80%カバレッジ）
      - E2Eテスト（Playwright）
      - アクセシビリティテスト
      - パフォーマンステスト
    dependencies: [phase_4]
```

### 6.2 文書間参照

```yaml
document_references:
  related_documents:
    - document: Doc-TV-001
      relationship: 技術ビジョンと原則の準拠
      sections:
        - パフォーマンス目標
        - セキュリティ要件
        - スケーラビリティ原則

    - document: Doc-SA-001
      relationship: システムアーキテクチャとの整合性
      sections:
        - APIゲートウェイ設計
        - 認証フロー
        - サービス境界

    - document: Doc-AD-001
      relationship: モバイルアプリとの一貫性
      sections:
        - 状態管理パターン
        - データモデル
        - API統合パターン

    - document: Doc-AD-003
      relationship: バックエンドAPI仕様
      sections:
        - REST API定義
        - エラーハンドリング
        - 認証・認可

  successor_documents:
    - Doc-QA-002: Webテスト戦略
    - Doc-DevOps-002: Web CI/CDパイプライン
    - Doc-SEC-002: Webセキュリティ実装
```

### 6.3 成功指標

```yaml
success_metrics:
  performance:
    core_web_vitals:
      lcp: "< 2.5s（目標: 2.0s）"
      fid: "< 100ms（目標: 50ms）"
      cls: "< 0.1（目標: 0.05）"
      inp: "< 200ms（目標: 150ms）"

    lighthouse_scores:
      performance: ">= 90"
      accessibility: ">= 95"
      best_practices: ">= 95"
      seo: ">= 95"

  seo:
    organic_traffic_growth: "> 20% MoM"
    search_ranking: "Top 10 for target keywords"
    indexed_pages: "> 95%"

  user_experience:
    bounce_rate: "< 40%"
    session_duration: "> 3 minutes"
    pages_per_session: "> 4"

  development:
    build_time: "< 3 minutes"
    deployment_frequency: ">= 2/day"
    test_coverage: ">= 80%"
```

---

## 結論

本文書は、TripTrip Webアプリケーションの包括的なアーキテクチャ設計を定義しました。Next.js 14のApp Routerを基盤とした最新のReactアーキテクチャ、Atomic Designによる体系的なコンポーネント設計、React Queryによる効率的なデータ管理を実現します。

主要な成果：
1. **Server Components活用**: 最適なレンダリング戦略でパフォーマンス最大化
2. **Atomic Design**: 再利用可能で保守性の高いコンポーネントライブラリ
3. **React Query**: 効率的なサーバー状態管理とオプティミスティック更新
4. **Core Web Vitals準拠**: 優れたユーザー体験とSEO効果
5. **PWA対応**: モバイルライクなWeb体験の実現

この設計により、TripTripはSEOに強く、高速で、アクセシブルなWebプラットフォームとして競争優位性を確立できます。

---

**文書情報**
- Document ID: Doc-AD-002
- Version: 1.0.0
- Last Updated: 2026-01-20
- Status: Draft
- Total Lines: 1,500
- Related Documents: Doc-TV-001, Doc-SA-001, Doc-AD-001, Doc-AD-003

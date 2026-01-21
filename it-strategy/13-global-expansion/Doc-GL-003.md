# Doc-GL-003: 多言語・ローカライゼーション戦略

## エグゼクティブサマリー

本文書は、TripTripプラットフォームのグローバル展開を支える多言語対応・ローカライゼーション（L10n）技術戦略を包括的に定義します。Netflix、Spotify、Airbnbレベルの国際化基盤を構築し、50以上の言語・地域への展開を可能にするスケーラブルなアーキテクチャを設計します。Flutter intlフレームワークを活用したi18nアーキテクチャ、AI支援翻訳を含むハイブリッド翻訳管理システム、文化的適応（カルチャライゼーション）戦略を通じて、各市場のユーザーに最適化されたローカル体験を提供します。本設計は、既存のFlutter 3.8.1+基盤を活用しながら、Phase 1（日英）からPhase 4（東南アジア言語）まで段階的に言語サポートを拡大し、2027年末までに20言語以上への対応を実現します。

---

## 第1章：はじめに & グローバル展開ビジョン

### 1.1 目的 & スコープ

#### 1.1.1 文書の目的

本文書は、TripTripの多言語・ローカライゼーション技術戦略を定義し、以下の目的を達成します。

```yaml
document_objectives:
  primary:
    - description: 国際化（i18n）アーキテクチャの標準化
      deliverable: i18nフレームワーク設計、実装ガイドライン

    - description: 翻訳管理システム（TMS）の設計
      deliverable: TMS選定基準、ワークフロー設計、統合仕様

    - description: 多言語コンテンツ配信戦略の確立
      deliverable: CDN設計、言語別ルーティング、キャッシュ戦略

  secondary:
    - description: 文化的適応（カルチャライゼーション）ガイドライン
      deliverable: 地域別UI/UXカスタマイズ指針

    - description: 言語品質保証プロセスの確立
      deliverable: LQAプロセス、自動化テスト戦略

    - description: 継続的ローカライゼーション運用の設計
      deliverable: CI/CD統合、翻訳更新フロー
```

#### 1.1.2 スコープ定義

```yaml
scope:
  in_scope:
    platforms:
      - Flutter iOS/Androidモバイルアプリ
      - Flutter Web (Progressive Web App)
      - バックエンドAPI（多言語レスポンス）
      - 管理ダッシュボード

    content_types:
      - UIテキスト（ボタン、ラベル、メッセージ）
      - システムメッセージ（エラー、通知）
      - マーケティングコンテンツ
      - 法的文書（利用規約、プライバシーポリシー）
      - ユーザー生成コンテンツ（UGC）翻訳支援
      - ヘルプ・サポートコンテンツ

    languages:
      phase_1: [ja, en]
      phase_2: [zh-CN, zh-TW, ko]
      phase_3: [fr, de, es, it]
      phase_4: [th, vi, id, ms]

  out_of_scope:
    - 音声・動画コンテンツのローカライズ（別途検討）
    - 完全自動翻訳によるUGC翻訳（品質管理上）
    - 手書き入力対応（将来検討）
```

### 1.2 ビジネス目標との連携

#### 1.2.1 グローバル展開戦略との整合

```yaml
business_alignment:
  market_expansion:
    phase_1:
      period: 2025 Q1-Q2
      markets: [日本, 英語圏（米国、英国、豪州）]
      languages: [ja, en]
      target_users: 500,000 MAU

    phase_2:
      period: 2025 Q3-Q4
      markets: [中国、台湾、香港、韓国]
      languages: [zh-CN, zh-TW, ko]
      target_users: 2,000,000 MAU

    phase_3:
      period: 2026 Q1-Q2
      markets: [フランス、ドイツ、スペイン、イタリア]
      languages: [fr, de, es, it]
      target_users: 5,000,000 MAU

    phase_4:
      period: 2026 Q3-2027
      markets: [タイ、ベトナム、インドネシア、マレーシア]
      languages: [th, vi, id, ms]
      target_users: 10,000,000 MAU

  revenue_impact:
    localization_roi:
      metric: 言語追加あたりの収益増加
      target: 15-25%（市場規模により変動）
      benchmark: Airbnb 20%, Booking.com 18%

    user_engagement:
      metric: ネイティブ言語ユーザーのエンゲージメント
      target: 非ネイティブ比 +40%
      benchmark: Netflix +45%, Spotify +38%

    conversion_rate:
      metric: ローカライズ版のコンバージョン率
      target: 英語のみ版比 +60%
      benchmark: 業界平均 +50-70%
```

#### 1.2.2 競合分析とベンチマーク

```yaml
competitive_analysis:
  best_in_class:
    netflix:
      languages: 60+
      approach: 完全ローカライズ + カルチャライズ
      key_practice: コンテキスト翻訳、A/Bテスト

    airbnb:
      languages: 62
      approach: コミュニティ翻訳 + プロ翻訳
      key_practice: 継続的ローカライゼーション

    spotify:
      languages: 65+
      approach: AI翻訳 + 人間レビュー
      key_practice: 音楽文化の地域適応

    booking_com:
      languages: 43
      approach: ハイブリッド翻訳
      key_practice: 旅行業界特化用語集

  travel_competitors:
    klook:
      languages: 14
      strengths: アジア市場特化
      weakness: 欧米言語カバレッジ

    viator:
      languages: 8
      strengths: 英語圏充実
      weakness: アジア言語対応

    get_your_guide:
      languages: 15
      strengths: 欧州言語充実
      weakness: アジア市場

  triptrip_differentiation:
    strategy: AI支援ハイブリッド翻訳
    target: 20言語（2027年末）
    unique_value:
      - 旅行体験に特化した用語集
      - 文化的コンテキスト翻訳
      - リアルタイム翻訳更新
```

### 1.3 主要な前提条件 & 制約

```yaml
assumptions:
  technical:
    - assumption: Flutter intl パッケージの継続サポート
      confidence: High
      mitigation: 代替としてeasy_localizationを準備

    - assumption: Cloud Translation API の品質向上
      confidence: High
      mitigation: 複数MT提供者の併用

    - assumption: TMS SaaS の安定稼働
      confidence: High
      mitigation: エクスポート/インポート機能でベンダーロックイン回避

  organizational:
    - assumption: 各言語のネイティブレビュアー確保
      confidence: Medium
      mitigation: 翻訳会社パートナーシップ

    - assumption: ローカライゼーション専任チーム設置
      confidence: High
      mitigation: Phase 1は兼任、Phase 2から専任化

constraints:
  budget:
    phase_1: $50,000（日英対応）
    phase_2: $150,000（APAC言語追加）
    phase_3: $200,000（欧州言語追加）
    phase_4: $100,000（東南アジア言語追加）

  timeline:
    language_addition: 4-6週間/言語
    full_localization: 8-12週間/言語（初回）

  quality:
    minimum_quality_score: 85%（LQAスコア）
    professional_translation_ratio: 80%+（マーケティング）
    mt_allowed_ratio: 70%（UI文字列、人間レビュー必須）
```

---

## 第2章：国際化（i18n）アーキテクチャ

### 2.1 フレームワーク & ツール選定

#### 2.1.1 Flutter国際化スタック

```yaml
i18n_stack:
  primary_framework:
    name: flutter_localizations + intl
    version: "0.20.2+"
    rationale:
      - Flutter公式サポート
      - ARBファイル形式の標準化
      - ICUメッセージフォーマット対応
      - コード生成による型安全性

  code_generation:
    tool: intl_utils / flutter_gen
    output: lib/generated/l10n.dart
    benefits:
      - コンパイル時の翻訳キー検証
      - IDE補完サポート
      - 未翻訳検出

  supplementary_packages:
    - package: intl
      purpose: 日付・数値・通貨フォーマット

    - package: flutter_localized_locales
      purpose: 言語名のローカライズ表示

    - package: timeago
      purpose: 相対時間表示（3分前、昨日など）
```

#### 2.1.2 アーキテクチャ設計

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TripTrip i18n ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                      PRESENTATION LAYER                              │    │
│  │                                                                      │    │
│  │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │    │
│  │   │   Widgets   │    │   Screens   │    │  Formatters │            │    │
│  │   │ S.of(ctx).  │    │  L10n.of()  │    │ NumberFormat│            │    │
│  │   │   hello     │    │             │    │ DateFormat  │            │    │
│  │   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘            │    │
│  │          │                  │                  │                    │    │
│  │          └──────────────────┴──────────────────┘                    │    │
│  │                             │                                        │    │
│  │                             ▼                                        │    │
│  │                   ┌─────────────────┐                               │    │
│  │                   │  LocaleProvider │                               │    │
│  │                   │   (Riverpod)    │                               │    │
│  │                   └────────┬────────┘                               │    │
│  └────────────────────────────┼────────────────────────────────────────┘    │
│                               │                                             │
│                               ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                     LOCALIZATION LAYER                               │    │
│  │                                                                      │    │
│  │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │    │
│  │   │ Generated   │    │  ARB Files  │    │  Fallback   │            │    │
│  │   │   L10n      │◄───│  (Source)   │    │   Chain     │            │    │
│  │   │   Class     │    │             │    │ ja→en→key  │            │    │
│  │   └─────────────┘    └─────────────┘    └─────────────┘            │    │
│  │                                                                      │    │
│  │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │    │
│  │   │   Locale    │    │  Plural     │    │   Gender    │            │    │
│  │   │  Detection  │    │   Rules     │    │   Rules     │            │    │
│  │   └─────────────┘    └─────────────┘    └─────────────┘            │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                      DATA LAYER                                      │    │
│  │                                                                      │    │
│  │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │    │
│  │   │   Remote    │    │    Local    │    │   Dynamic   │            │    │
│  │   │   Config    │    │   Storage   │    │   Updates   │            │    │
│  │   │ (Firebase)  │    │   (Hive)    │    │   (OTA)     │            │    │
│  │   └─────────────┘    └─────────────┘    └─────────────┘            │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 2.1.3 ディレクトリ構造

```
lib/
├── l10n/
│   ├── arb/
│   │   ├── app_ja.arb              # 日本語（ベース言語）
│   │   ├── app_en.arb              # 英語
│   │   ├── app_zh_CN.arb           # 簡体字中国語
│   │   ├── app_zh_TW.arb           # 繁体字中国語
│   │   ├── app_ko.arb              # 韓国語
│   │   ├── app_fr.arb              # フランス語
│   │   ├── app_de.arb              # ドイツ語
│   │   ├── app_es.arb              # スペイン語
│   │   ├── app_it.arb              # イタリア語
│   │   ├── app_th.arb              # タイ語
│   │   ├── app_vi.arb              # ベトナム語
│   │   ├── app_id.arb              # インドネシア語
│   │   └── app_ms.arb              # マレー語
│   │
│   ├── l10n.yaml                   # 生成設定
│   └── README.md                   # ローカライゼーションガイド
│
├── generated/
│   ├── l10n.dart                   # 生成されたL10nクラス
│   └── l10n_*.dart                 # 言語別デリゲート
│
├── core/
│   └── localization/
│       ├── locale_provider.dart    # Riverpod Locale管理
│       ├── locale_detector.dart    # 自動言語検出
│       ├── formatters.dart         # 日付・数値フォーマッタ
│       └── rtl_helper.dart         # RTL言語サポート
│
└── features/
    └── settings/
        └── language_settings/
            ├── language_settings_screen.dart
            └── widgets/
                └── language_selector.dart
```

### 2.2 文字列外部化設計

#### 2.2.1 ARBファイル構造

```json
// app_ja.arb - 日本語（ベース言語）
{
  "@@locale": "ja",
  "@@last_modified": "2025-01-21T00:00:00.000Z",

  "appTitle": "TripTrip",
  "@appTitle": {
    "description": "アプリケーションのタイトル"
  },

  "welcomeMessage": "ようこそ、{userName}さん",
  "@welcomeMessage": {
    "description": "ホーム画面のウェルカムメッセージ",
    "placeholders": {
      "userName": {
        "type": "String",
        "example": "田中"
      }
    }
  },

  "itemCount": "{count, plural, =0{アイテムがありません} =1{1件のアイテム} other{{count}件のアイテム}}",
  "@itemCount": {
    "description": "カート内のアイテム数",
    "placeholders": {
      "count": {
        "type": "int"
      }
    }
  },

  "bookingStatus": "{status, select, pending{予約確認中} confirmed{予約確定} cancelled{キャンセル済み} other{不明}}",
  "@bookingStatus": {
    "description": "予約ステータス表示",
    "placeholders": {
      "status": {
        "type": "String"
      }
    }
  },

  "priceDisplay": "{price}",
  "@priceDisplay": {
    "description": "価格表示（通貨フォーマットは別途処理）",
    "placeholders": {
      "price": {
        "type": "String",
        "example": "¥10,000"
      }
    }
  },

  "dateRange": "{startDate}〜{endDate}",
  "@dateRange": {
    "description": "日付範囲表示",
    "placeholders": {
      "startDate": {
        "type": "String",
        "example": "2025年1月1日"
      },
      "endDate": {
        "type": "String",
        "example": "2025年1月5日"
      }
    }
  },

  "hotelSearchTitle": "ホテル検索",
  "hotelSearchPlaceholder": "目的地を入力",
  "hotelSearchNoResults": "検索条件に一致するホテルが見つかりませんでした",

  "checkInLabel": "チェックイン",
  "checkOutLabel": "チェックアウト",
  "guestCountLabel": "宿泊人数",
  "roomCountLabel": "部屋数",

  "filterTitle": "絞り込み",
  "filterPrice": "価格帯",
  "filterRating": "評価",
  "filterAmenities": "設備・サービス",
  "filterApply": "適用",
  "filterReset": "リセット",

  "cartTitle": "カート",
  "cartEmpty": "カートに商品がありません",
  "cartSubtotal": "小計",
  "cartTax": "消費税",
  "cartTotal": "合計",
  "cartCheckout": "購入手続きへ",

  "errorGeneric": "エラーが発生しました。もう一度お試しください。",
  "errorNetwork": "ネットワーク接続を確認してください。",
  "errorTimeout": "接続がタイムアウトしました。",
  "errorServerUnavailable": "サーバーに接続できません。",

  "buttonRetry": "再試行",
  "buttonCancel": "キャンセル",
  "buttonConfirm": "確認",
  "buttonSave": "保存",
  "buttonEdit": "編集",
  "buttonDelete": "削除",
  "buttonBack": "戻る",
  "buttonNext": "次へ",
  "buttonDone": "完了"
}
```

```json
// app_en.arb - 英語
{
  "@@locale": "en",
  "@@last_modified": "2025-01-21T00:00:00.000Z",

  "appTitle": "TripTrip",

  "welcomeMessage": "Welcome, {userName}",
  "@welcomeMessage": {
    "placeholders": {
      "userName": {
        "type": "String",
        "example": "John"
      }
    }
  },

  "itemCount": "{count, plural, =0{No items} =1{1 item} other{{count} items}}",
  "@itemCount": {
    "placeholders": {
      "count": {
        "type": "int"
      }
    }
  },

  "bookingStatus": "{status, select, pending{Pending confirmation} confirmed{Confirmed} cancelled{Cancelled} other{Unknown}}",

  "priceDisplay": "{price}",

  "dateRange": "{startDate} - {endDate}",

  "hotelSearchTitle": "Hotel Search",
  "hotelSearchPlaceholder": "Enter destination",
  "hotelSearchNoResults": "No hotels found matching your criteria",

  "checkInLabel": "Check-in",
  "checkOutLabel": "Check-out",
  "guestCountLabel": "Guests",
  "roomCountLabel": "Rooms",

  "filterTitle": "Filter",
  "filterPrice": "Price range",
  "filterRating": "Rating",
  "filterAmenities": "Amenities",
  "filterApply": "Apply",
  "filterReset": "Reset",

  "cartTitle": "Cart",
  "cartEmpty": "Your cart is empty",
  "cartSubtotal": "Subtotal",
  "cartTax": "Tax",
  "cartTotal": "Total",
  "cartCheckout": "Proceed to checkout",

  "errorGeneric": "An error occurred. Please try again.",
  "errorNetwork": "Please check your network connection.",
  "errorTimeout": "Connection timed out.",
  "errorServerUnavailable": "Unable to connect to server.",

  "buttonRetry": "Retry",
  "buttonCancel": "Cancel",
  "buttonConfirm": "Confirm",
  "buttonSave": "Save",
  "buttonEdit": "Edit",
  "buttonDelete": "Delete",
  "buttonBack": "Back",
  "buttonNext": "Next",
  "buttonDone": "Done"
}
```

#### 2.2.2 命名規則とベストプラクティス

```yaml
naming_conventions:
  key_format:
    pattern: "{feature}_{element}_{variant}"
    examples:
      - hotelSearch_title
      - hotelSearch_placeholder
      - hotelSearch_noResults
      - cart_button_checkout
      - error_network_message

  prefixes:
    feature:
      - hotel: ホテル関連
      - attraction: アトラクション関連
      - product: 商品関連
      - cart: カート関連
      - checkout: 決済関連
      - user: ユーザー関連
      - settings: 設定関連

    element:
      - title: 画面/セクションタイトル
      - label: フォームラベル
      - placeholder: 入力プレースホルダー
      - button: ボタンテキスト
      - message: メッセージ
      - error: エラーメッセージ
      - hint: ヒントテキスト

  anti_patterns:
    avoid:
      - 数字のみのキー: "1", "123"
      - 曖昧なキー: "text1", "label2"
      - ハードコードテキスト参照: "click_here"
      - 過度に長いキー: "user_profile_settings_language_selection_title"

best_practices:
  string_management:
    - practice: コンテキスト情報の記載
      example: |
        "@hotelSearchTitle": {
          "description": "ホテル検索画面のメインタイトル",
          "context": "画面上部に表示",
          "max_length": 20
        }

    - practice: プレースホルダーの型指定
      example: |
        "placeholders": {
          "count": {
            "type": "int",
            "format": "compactLong"
          }
        }

    - practice: 例文の提供
      example: |
        "placeholders": {
          "userName": {
            "type": "String",
            "example": "田中太郎"
          }
        }

  avoid:
    - UI固有の用語（"クリック" vs "タップ"）
    - 文化依存の表現（色、数字の意味）
    - 連結による文生成
    - 画像内テキスト（ローカライズ困難）
```

### 2.3 フォーマット処理

#### 2.3.1 日付・時刻フォーマット

```dart
/// 日付・時刻フォーマッタクラス
class TripTripDateFormatter {
  final Locale locale;

  TripTripDateFormatter(this.locale);

  /// 完全な日付表示
  /// ja: 2025年1月21日（火）
  /// en: Tuesday, January 21, 2025
  String formatFullDate(DateTime date) {
    return DateFormat.yMMMMEEEEd(locale.toString()).format(date);
  }

  /// 短い日付表示
  /// ja: 1/21
  /// en: 1/21
  String formatShortDate(DateTime date) {
    return DateFormat.Md(locale.toString()).format(date);
  }

  /// 中程度の日付表示
  /// ja: 1月21日
  /// en: Jan 21
  String formatMediumDate(DateTime date) {
    return DateFormat.MMMd(locale.toString()).format(date);
  }

  /// 時刻表示
  /// ja: 14:30
  /// en: 2:30 PM
  String formatTime(DateTime time) {
    return DateFormat.jm(locale.toString()).format(time);
  }

  /// 相対時間表示
  /// ja: 3分前、昨日、先週
  /// en: 3 minutes ago, yesterday, last week
  String formatRelativeTime(DateTime dateTime) {
    final now = DateTime.now();
    final difference = now.difference(dateTime);

    if (difference.inMinutes < 1) {
      return _getLocalizedString('just_now');
    } else if (difference.inMinutes < 60) {
      return _getLocalizedString('minutes_ago', {'count': difference.inMinutes});
    } else if (difference.inHours < 24) {
      return _getLocalizedString('hours_ago', {'count': difference.inHours});
    } else if (difference.inDays < 7) {
      return _getLocalizedString('days_ago', {'count': difference.inDays});
    } else {
      return formatMediumDate(dateTime);
    }
  }

  /// チェックイン/チェックアウト日付範囲
  String formatDateRange(DateTime start, DateTime end) {
    final startStr = formatMediumDate(start);
    final endStr = formatMediumDate(end);
    final nights = end.difference(start).inDays;

    // ja: 1月21日〜1月23日（2泊）
    // en: Jan 21 - Jan 23 (2 nights)
    return _getLocalizedString('date_range_with_nights', {
      'start': startStr,
      'end': endStr,
      'nights': nights,
    });
  }
}
```

#### 2.3.2 数値・通貨フォーマット

```dart
/// 数値・通貨フォーマッタクラス
class TripTripNumberFormatter {
  final Locale locale;
  final String currencyCode;

  TripTripNumberFormatter({
    required this.locale,
    this.currencyCode = 'JPY',
  });

  /// 通貨表示
  /// JPY: ¥10,000
  /// USD: $100.00
  /// EUR: €100,00
  String formatCurrency(num amount, {String? currency}) {
    final effectiveCurrency = currency ?? currencyCode;
    final format = NumberFormat.currency(
      locale: locale.toString(),
      symbol: _getCurrencySymbol(effectiveCurrency),
      decimalDigits: _getDecimalDigits(effectiveCurrency),
    );
    return format.format(amount);
  }

  /// 通貨コード付き表示（国際取引向け）
  /// JPY 10,000
  /// USD 100.00
  String formatCurrencyWithCode(num amount, {String? currency}) {
    final effectiveCurrency = currency ?? currencyCode;
    final format = NumberFormat.currency(
      locale: locale.toString(),
      name: effectiveCurrency,
      decimalDigits: _getDecimalDigits(effectiveCurrency),
    );
    return format.format(amount);
  }

  /// 数値表示（千単位区切り）
  /// ja: 1,234,567
  /// de: 1.234.567
  String formatNumber(num value) {
    return NumberFormat.decimalPattern(locale.toString()).format(value);
  }

  /// コンパクト数値表示
  /// ja: 1.2万
  /// en: 12K
  String formatCompactNumber(num value) {
    return NumberFormat.compact(locale: locale.toString()).format(value);
  }

  /// パーセント表示
  /// ja: 25%
  /// en: 25%
  String formatPercent(double value) {
    return NumberFormat.percentPattern(locale.toString()).format(value);
  }

  /// 距離表示
  /// ja: 1.5km
  /// en: 0.9mi (米国) / 1.5km (その他)
  String formatDistance(double kilometers) {
    if (_useImperialUnits()) {
      final miles = kilometers * 0.621371;
      return '${formatNumber(miles)} mi';
    }
    return '${formatNumber(kilometers)} km';
  }

  int _getDecimalDigits(String currency) {
    const noDecimalCurrencies = ['JPY', 'KRW', 'VND', 'IDR'];
    return noDecimalCurrencies.contains(currency) ? 0 : 2;
  }

  String _getCurrencySymbol(String currency) {
    const symbols = {
      'JPY': '¥',
      'USD': r'$',
      'EUR': '€',
      'GBP': '£',
      'CNY': '¥',
      'KRW': '₩',
      'THB': '฿',
      'VND': '₫',
      'IDR': 'Rp',
      'MYR': 'RM',
    };
    return symbols[currency] ?? currency;
  }

  bool _useImperialUnits() {
    return locale.countryCode == 'US' ||
           locale.countryCode == 'LR' ||
           locale.countryCode == 'MM';
  }
}
```

### 2.4 双方向テキスト対応

#### 2.4.1 RTL言語サポート設計

```yaml
rtl_support:
  target_languages:
    - ar: アラビア語（将来対応）
    - he: ヘブライ語（将来対応）
    - fa: ペルシャ語（将来対応）
    - ur: ウルドゥー語（将来対応）

  implementation_strategy:
    phase: Phase 5以降（2027年以降）
    approach: 段階的対応

  technical_considerations:
    layout:
      - Directionality ウィジェットの活用
      - start/end の代わりに left/right を避ける
      - TextDirection の動的切り替え

    icons:
      - 方向性のあるアイコンの反転
      - 矢印、ナビゲーションアイコン

    images:
      - テキスト入り画像の別バージョン準備
      - 方向性のあるイラストの反転
```

```dart
/// RTL対応ヘルパークラス
class RTLHelper {
  static bool isRTL(Locale locale) {
    const rtlLanguages = ['ar', 'he', 'fa', 'ur'];
    return rtlLanguages.contains(locale.languageCode);
  }

  static TextDirection getTextDirection(Locale locale) {
    return isRTL(locale) ? TextDirection.rtl : TextDirection.ltr;
  }

  /// RTL対応のパディング取得
  static EdgeInsetsDirectional getDirectionalPadding({
    double start = 0,
    double top = 0,
    double end = 0,
    double bottom = 0,
  }) {
    return EdgeInsetsDirectional.only(
      start: start,
      top: top,
      end: end,
      bottom: bottom,
    );
  }

  /// RTL対応のアイコン反転
  static Widget getDirectionalIcon(
    IconData icon,
    BuildContext context, {
    double? size,
    Color? color,
  }) {
    final isRtl = Directionality.of(context) == TextDirection.rtl;

    // 方向性のあるアイコンのみ反転
    const directionalIcons = [
      Icons.arrow_forward,
      Icons.arrow_back,
      Icons.chevron_right,
      Icons.chevron_left,
      Icons.navigate_next,
      Icons.navigate_before,
    ];

    if (isRtl && directionalIcons.contains(icon)) {
      return Transform.scale(
        scaleX: -1,
        child: Icon(icon, size: size, color: color),
      );
    }

    return Icon(icon, size: size, color: color);
  }
}

/// RTL対応のレイアウトウィジェット
class DirectionalContainer extends StatelessWidget {
  final Widget child;
  final EdgeInsetsDirectional? padding;
  final EdgeInsetsDirectional? margin;

  const DirectionalContainer({
    required this.child,
    this.padding,
    this.margin,
    super.key,
  });

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: padding,
      margin: margin,
      child: child,
    );
  }
}
```

---

## 第3章：翻訳管理システム（TMS）

### 3.1 システム選定 & 統合

#### 3.1.1 TMS選定基準

```yaml
tms_selection_criteria:
  must_have:
    - criterion: ARBファイル形式サポート
      weight: Critical

    - criterion: Git/GitHub統合
      weight: Critical

    - criterion: 機械翻訳統合
      weight: Critical

    - criterion: コンテキスト付きスクリーンショット
      weight: High

    - criterion: 翻訳メモリ
      weight: High

    - criterion: 用語集管理
      weight: High

  nice_to_have:
    - criterion: In-context editor
      weight: Medium

    - criterion: QAチェック自動化
      weight: Medium

    - criterion: API/Webhook
      weight: Medium

    - criterion: ブランチング/バージョニング
      weight: Low

  evaluation_matrix:
    lokalise:
      arb_support: ✓
      git_integration: ✓
      mt_integration: ✓ (Google, DeepL, Amazon)
      screenshots: ✓
      translation_memory: ✓
      glossary: ✓
      in_context: ✓
      qa_checks: ✓
      api: ✓
      branching: ✓
      pricing: $120/month (Team)
      recommendation: Primary Choice

    phrase:
      arb_support: ✓
      git_integration: ✓
      mt_integration: ✓ (multiple)
      screenshots: ✓
      translation_memory: ✓
      glossary: ✓
      in_context: ✓
      qa_checks: ✓
      api: ✓
      branching: ✓
      pricing: $150/month (Team)
      recommendation: Alternative

    crowdin:
      arb_support: ✓
      git_integration: ✓
      mt_integration: ✓
      screenshots: ✓
      translation_memory: ✓
      glossary: ✓
      in_context: Partial
      qa_checks: ✓
      api: ✓
      branching: ✓
      pricing: $100/month (Team)
      recommendation: Budget Option
```

#### 3.1.2 Lokalise統合設計

```yaml
lokalise_integration:
  project_structure:
    main_project: TripTrip Mobile App
    sub_projects:
      - TripTrip iOS/Android
      - TripTrip Web
      - TripTrip Backend Messages
      - TripTrip Marketing

  github_integration:
    repository: triptrip/triptrip-mobile
    branch_pattern: "l10n/*"
    sync_direction: bidirectional
    auto_merge: false (PR required)

  file_mapping:
    source:
      path: lib/l10n/arb/app_%LANG_ISO%.arb
      format: ARB

    download:
      path: lib/l10n/arb/app_%LANG_ISO%.arb
      format: ARB

  workflow:
    development:
      1. 開発者がベース言語（日本語）でキーを追加
      2. Git push でLokaliseに自動同期
      3. 翻訳者が他言語を翻訳
      4. 翻訳完了後、PRを自動作成
      5. レビュー後マージ

    hotfix:
      1. Lokaliseで直接編集
      2. 即座にdownloadトリガー
      3. 緊急PRとしてマージ
```

```yaml
# .lokalise.yml - Lokalise CLI設定
lokalise:
  project_id: "your-project-id"

  pull:
    format: arb
    original_filenames: true
    directory_prefix: ""
    include_tags:
      - mobile
    exclude_tags:
      - deprecated
    filter_langs:
      - ja
      - en
      - zh_CN
      - zh_TW
      - ko
    placeholder_format: icu

  push:
    include:
      - "lib/l10n/arb/*.arb"
    tags:
      - mobile
      - auto-sync
    cleanup_mode: false
    detect_icu_plurals: true
    convert_placeholders: true
```

### 3.2 ワークフロー設計

#### 3.2.1 翻訳ワークフロー

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TRANSLATION WORKFLOW                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐  │
│  │Developer│    │  TMS    │    │   MT    │    │Translator│   │Reviewer │  │
│  └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘  │
│       │              │              │              │              │        │
│       │  Add Key     │              │              │              │        │
│       │─────────────►│              │              │              │        │
│       │              │              │              │              │        │
│       │              │  Pre-translate              │              │        │
│       │              │─────────────►│              │              │        │
│       │              │              │              │              │        │
│       │              │◄─────────────│              │              │        │
│       │              │  MT Result   │              │              │        │
│       │              │              │              │              │        │
│       │              │  Assign Task │              │              │        │
│       │              │─────────────────────────────►              │        │
│       │              │              │              │              │        │
│       │              │              │   Review MT  │              │        │
│       │              │              │   & Edit     │              │        │
│       │              │◄─────────────────────────────              │        │
│       │              │  Translation │              │              │        │
│       │              │              │              │              │        │
│       │              │  Request Review             │              │        │
│       │              │──────────────────────────────────────────►│        │
│       │              │              │              │              │        │
│       │              │              │              │   LQA Check  │        │
│       │              │◄──────────────────────────────────────────│        │
│       │              │  Approved    │              │              │        │
│       │              │              │              │              │        │
│       │  PR Created  │              │              │              │        │
│       │◄─────────────│              │              │              │        │
│       │              │              │              │              │        │
│       │  Merge       │              │              │              │        │
│       ▼              │              │              │              │        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.2.2 ステータス管理

```yaml
translation_statuses:
  key_statuses:
    - status: draft
      description: 開発者が追加、翻訳待ち
      color: gray

    - status: machine_translated
      description: MT適用済み、レビュー待ち
      color: yellow

    - status: in_translation
      description: 翻訳者が作業中
      color: blue

    - status: translated
      description: 翻訳完了、レビュー待ち
      color: orange

    - status: in_review
      description: レビュアーが確認中
      color: purple

    - status: approved
      description: レビュー完了、リリース可能
      color: green

    - status: rejected
      description: 修正必要
      color: red

  workflow_rules:
    marketing_content:
      required_steps:
        - machine_translation: optional
        - human_translation: required
        - review: required (native speaker)
        - legal_review: required (法的文書)
      quality_threshold: 95%

    ui_strings:
      required_steps:
        - machine_translation: required
        - human_review: required
        - qa_check: automated
      quality_threshold: 90%

    error_messages:
      required_steps:
        - human_translation: required
        - technical_review: required
      quality_threshold: 95%
```

### 3.3 MT/HT ハイブリッド戦略

#### 3.3.1 機械翻訳設定

```yaml
machine_translation_config:
  primary_engine:
    provider: Google Cloud Translation
    api_version: v3 (Advanced)
    features:
      - adaptive_mt: enabled
      - glossary: enabled
      - model: nmt

  secondary_engine:
    provider: DeepL
    use_case: 欧州言語、文学的コンテンツ

  fallback_engine:
    provider: Amazon Translate
    use_case: DeepL非対応言語

  language_routing:
    google_primary:
      - ja → en
      - en → ja
      - zh-CN, zh-TW
      - ko
      - th, vi, id, ms

    deepl_primary:
      - en → de, fr, es, it
      - de, fr, es, it → en

  quality_estimation:
    enabled: true
    threshold:
      auto_approve: 0.9+
      human_review_required: 0.7-0.9
      human_translation_required: <0.7
```

#### 3.3.2 翻訳カテゴリ別戦略

```yaml
content_category_strategy:
  marketing:
    description: マーケティング、プロモーション文言
    mt_allowed: false
    translation_type: professional
    review_required: native_speaker
    turnaround: 3-5 business days
    examples:
      - アプリストア説明文
      - キャンペーンバナー
      - メールマーケティング

  legal:
    description: 利用規約、プライバシーポリシー
    mt_allowed: false
    translation_type: certified
    review_required: legal_team
    turnaround: 5-10 business days
    examples:
      - 利用規約
      - プライバシーポリシー
      - 返金ポリシー

  ui_static:
    description: 静的UIテキスト（ボタン、ラベル）
    mt_allowed: true
    translation_type: hybrid
    review_required: translator
    turnaround: 1-2 business days
    examples:
      - ボタンテキスト
      - フォームラベル
      - ナビゲーション

  ui_dynamic:
    description: 動的メッセージ（エラー、通知）
    mt_allowed: true
    translation_type: hybrid
    review_required: translator
    turnaround: 1-2 business days
    examples:
      - エラーメッセージ
      - 成功メッセージ
      - プッシュ通知

  user_generated:
    description: ユーザー生成コンテンツ
    mt_allowed: true (display only)
    translation_type: machine_only
    review_required: false
    note: 翻訳品質の免責表示必須
    examples:
      - レビュー
      - Q&A
      - コメント
```

#### 3.3.3 用語集管理

```yaml
glossary_management:
  structure:
    master_glossary:
      name: TripTrip Master Glossary
      languages: all
      categories:
        - brand_terms
        - travel_terms
        - technical_terms
        - legal_terms

    language_specific:
      ja_en:
        name: 日英旅行用語集
        special_rules:
          - 敬称の扱い
          - 地名の表記

      zh_variants:
        name: 中国語簡繁変換対応
        special_rules:
          - 簡体字/繁体字対応表
          - 地域別表現

  term_examples:
    brand_terms:
      - source: TripTrip
        translation: TripTrip (全言語共通、翻訳不可)
        note: ブランド名、常にそのまま使用

      - source: トリップメイト
        translations:
          en: TripMate
          zh-CN: 旅伴
          ko: 트립메이트

    travel_terms:
      - source: チェックイン
        translations:
          en: Check-in
          zh-CN: 入住
          ko: 체크인

      - source: 観光チケット
        translations:
          en: Attraction ticket
          zh-CN: 景点门票
          ko: 관광 티켓

    do_not_translate:
      - Wi-Fi
      - QRコード / QR code
      - GPS
      - API
      - URL
```

---

## 第4章：対応言語ロードマップ

### 4.1 言語展開計画

```yaml
language_roadmap:
  phase_1:
    period: 2025 Q1-Q2
    languages:
      ja:
        name: 日本語
        status: Base language
        coverage: 100%

      en:
        name: English
        status: Primary target
        coverage_target: 100%
        translator_type: Professional (native)

    deliverables:
      - 全UIテキストの日英対応
      - 法的文書の日英対応
      - ヘルプコンテンツの日英対応

    success_criteria:
      - LQA score: 95%+
      - User satisfaction: 4.5/5
      - Bug reports (i18n): <5/month

  phase_2:
    period: 2025 Q3-Q4
    languages:
      zh-CN:
        name: 简体中文
        coverage_target: 100%
        translator_type: Professional (mainland China)
        special_considerations:
          - 中国本土向けUI/UX調整
          - 決済方法（WeChat Pay、Alipay）

      zh-TW:
        name: 繁體中文
        coverage_target: 100%
        translator_type: Professional (Taiwan)
        special_considerations:
          - 台湾向け表現
          - 繁体字固有の用語

      ko:
        name: 한국어
        coverage_target: 100%
        translator_type: Professional (South Korea)
        special_considerations:
          - 敬語レベルの統一
          - 韓国向け決済連携

    deliverables:
      - APAC市場向け完全ローカライズ
      - 地域別カスタマイズ（決済、UI）

    success_criteria:
      - LQA score: 90%+
      - Market-specific conversion: +30%

  phase_3:
    period: 2026 Q1-Q2
    languages:
      fr:
        name: Français
        coverage_target: 95%
        translator_type: Professional (France)

      de:
        name: Deutsch
        coverage_target: 95%
        translator_type: Professional (Germany)
        special_considerations:
          - 長い複合語への対応
          - フォーマル表現

      es:
        name: Español
        coverage_target: 95%
        translator_type: Professional (Spain)
        note: ラテンアメリカ向けは別途検討

      it:
        name: Italiano
        coverage_target: 95%
        translator_type: Professional (Italy)

    deliverables:
      - 欧州主要市場向けローカライズ
      - GDPR完全対応

  phase_4:
    period: 2026 Q3-2027
    languages:
      th:
        name: ไทย
        coverage_target: 90%
        special_considerations:
          - タイ文字レンダリング
          - 仏暦対応

      vi:
        name: Tiếng Việt
        coverage_target: 90%
        special_considerations:
          - 声調記号の正確な表示

      id:
        name: Bahasa Indonesia
        coverage_target: 90%

      ms:
        name: Bahasa Melayu
        coverage_target: 90%

    deliverables:
      - 東南アジア市場展開
      - 地域別機能（Grab連携等）
```

### 4.2 言語追加プロセス

```yaml
language_addition_process:
  prerequisites:
    market_analysis:
      - ターゲット市場のMAU見込み
      - 競合の言語対応状況
      - 現地パートナー/代理店の有無

    resource_allocation:
      - 翻訳予算の確保
      - ネイティブレビュアーの確保
      - カスタマーサポート体制

    technical_readiness:
      - フォントサポート確認
      - 入力メソッド対応
      - テストデバイス確保

  execution_steps:
    week_1_2:
      - TMSプロジェクト設定
      - 用語集の初期翻訳
      - スタイルガイド作成

    week_3_4:
      - UIテキストの機械翻訳
      - 人間翻訳レビュー開始
      - 法的文書の翻訳

    week_5_6:
      - 翻訳完了
      - LQA実施
      - スクリーンショットテスト

    week_7_8:
      - バグ修正
      - ベータテスト（社内）
      - 最終レビュー

    week_9:
      - 段階的ロールアウト（10%）
      - ユーザーフィードバック収集

    week_10:
      - 修正対応
      - フルロールアウト

  quality_gates:
    - gate: Translation completion
      threshold: 95%

    - gate: LQA score
      threshold: 85%

    - gate: Critical bugs
      threshold: 0

    - gate: Beta user rating
      threshold: 4.0/5
```

---

## 第5章：コンテンツローカライゼーション戦略

### 5.1 UIテキストのローカライズ

#### 5.1.1 UI要素別ガイドライン

```yaml
ui_localization_guidelines:
  buttons:
    max_length:
      primary_action: 12文字
      secondary_action: 15文字
    guidelines:
      - 動詞で開始（日本語: 〜する、英語: 動詞原形）
      - 明確なアクション表現
      - 省略形は避ける
    examples:
      ja: 予約する、カートに追加、検索
      en: Book now, Add to cart, Search

  labels:
    max_length: 20文字
    guidelines:
      - 簡潔で明確
      - 技術用語は避ける
      - 一貫した表現
    examples:
      ja: チェックイン日、宿泊人数
      en: Check-in date, Number of guests

  placeholders:
    max_length: 30文字
    guidelines:
      - 入力例または説明
      - 親しみやすい口調
    examples:
      ja: 東京、大阪などの都市名
      en: City name (e.g., Tokyo, Osaka)

  error_messages:
    guidelines:
      - 原因を明確に
      - 解決策を提示
      - 責任転嫁しない
    examples:
      ja: メールアドレスの形式が正しくありません。example@email.comの形式で入力してください。
      en: Invalid email format. Please enter an email like example@email.com.

  success_messages:
    guidelines:
      - ポジティブな表現
      - 次のアクションを示唆
    examples:
      ja: 予約が完了しました。確認メールをお送りしました。
      en: Booking confirmed! A confirmation email has been sent.

  empty_states:
    guidelines:
      - 状況を説明
      - アクションを促す
      - イラストと組み合わせ
    examples:
      ja: |
        検索結果がありません
        条件を変更して再検索してください
      en: |
        No results found
        Try adjusting your search criteria
```

### 5.2 マーケティングコンテンツ

```yaml
marketing_localization:
  app_store:
    title:
      max_length: 30文字
      localization: transcreation

    subtitle:
      max_length: 30文字
      localization: transcreation

    description:
      max_length: 4000文字
      localization: professional translation + review

    keywords:
      approach: 地域別キーワードリサーチ

    screenshots:
      approach: 言語別スクリーンショット生成
      automation: Fastlane + screenshots

  push_notifications:
    guidelines:
      - 文化的に適切なタイミング
      - 地域の祝日・イベント考慮
      - パーソナライズ
    examples:
      ja: |
        【期間限定】京都の紅葉シーズン特別プラン
        〜11/30まで最大30%OFF
      en: |
        Limited time: Kyoto Autumn Special
        Up to 30% off until Nov 30

  email_marketing:
    localization: professional translation
    cultural_adaptation:
      - 挨拶の文化差
      - 季節の挨拶
      - プロモーション表現
```

### 5.3 法的文書

```yaml
legal_document_localization:
  terms_of_service:
    translation_type: certified legal translation
    review: local legal counsel
    update_frequency: 法改正時、機能追加時

  privacy_policy:
    translation_type: certified legal translation
    review: local DPO/legal counsel
    compliance:
      - GDPR (EU)
      - CCPA (California)
      - PIPL (China)
      - APPI (Japan)

  refund_policy:
    translation_type: professional + legal review
    regional_variations:
      eu: 14日間クーリングオフ
      japan: 特定商取引法準拠

  cookie_consent:
    translation_type: professional
    regional_variations:
      eu: GDPR準拠の詳細オプション
      japan: 簡易版
```

---

## 第6章：技術実装詳細

### 6.1 Flutterでのi18n実装パターン

#### 6.1.1 Locale管理

```dart
/// Locale管理プロバイダー
@riverpod
class LocaleNotifier extends _$LocaleNotifier {
  static const _localeKey = 'user_locale';

  @override
  Locale build() {
    // 保存された設定を読み込み
    final savedLocale = _loadSavedLocale();
    if (savedLocale != null) {
      return savedLocale;
    }

    // システム言語を使用
    final systemLocale = WidgetsBinding.instance.platformDispatcher.locale;

    // サポート言語かチェック
    if (_isSupportedLocale(systemLocale)) {
      return systemLocale;
    }

    // デフォルトは日本語
    return const Locale('ja');
  }

  /// 言語を変更
  Future<void> setLocale(Locale locale) async {
    if (!_isSupportedLocale(locale)) {
      throw UnsupportedLocaleException(locale);
    }

    // 永続化
    await _saveLocale(locale);

    // 状態更新
    state = locale;

    // アナリティクスに送信
    ref.read(analyticsProvider).logLanguageChange(locale);
  }

  /// サポート言語一覧
  List<Locale> get supportedLocales => const [
    Locale('ja'),
    Locale('en'),
    Locale('zh', 'CN'),
    Locale('zh', 'TW'),
    Locale('ko'),
    Locale('fr'),
    Locale('de'),
    Locale('es'),
    Locale('it'),
    Locale('th'),
    Locale('vi'),
    Locale('id'),
    Locale('ms'),
  ];

  bool _isSupportedLocale(Locale locale) {
    return supportedLocales.any(
      (supported) =>
          supported.languageCode == locale.languageCode &&
          (supported.countryCode == null ||
              supported.countryCode == locale.countryCode),
    );
  }

  Locale? _loadSavedLocale() {
    final prefs = ref.read(sharedPreferencesProvider);
    final localeString = prefs.getString(_localeKey);
    if (localeString == null) return null;

    final parts = localeString.split('_');
    if (parts.length == 1) {
      return Locale(parts[0]);
    } else {
      return Locale(parts[0], parts[1]);
    }
  }

  Future<void> _saveLocale(Locale locale) async {
    final prefs = ref.read(sharedPreferencesProvider);
    final localeString = locale.countryCode != null
        ? '${locale.languageCode}_${locale.countryCode}'
        : locale.languageCode;
    await prefs.setString(_localeKey, localeString);
  }
}

/// アプリのルートウィジェット
class TripTripApp extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final locale = ref.watch(localeNotifierProvider);

    return MaterialApp(
      locale: locale,
      supportedLocales: ref.read(localeNotifierProvider.notifier).supportedLocales,
      localizationsDelegates: const [
        S.delegate,  // 生成されたデリゲート
        GlobalMaterialLocalizations.delegate,
        GlobalWidgetsLocalizations.delegate,
        GlobalCupertinoLocalizations.delegate,
      ],
      localeResolutionCallback: (locale, supportedLocales) {
        // 言語のみでマッチング
        for (final supportedLocale in supportedLocales) {
          if (supportedLocale.languageCode == locale?.languageCode) {
            return supportedLocale;
          }
        }
        // デフォルト
        return const Locale('ja');
      },
      home: const HomeScreen(),
    );
  }
}
```

#### 6.1.2 翻訳文字列の使用

```dart
/// 基本的な使用方法
class HotelSearchScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // S.of(context) で翻訳にアクセス
    final l10n = S.of(context);

    return Scaffold(
      appBar: AppBar(
        title: Text(l10n.hotelSearchTitle),  // "ホテル検索" / "Hotel Search"
      ),
      body: Column(
        children: [
          TextField(
            decoration: InputDecoration(
              hintText: l10n.hotelSearchPlaceholder,  // "目的地を入力"
            ),
          ),

          // プレースホルダー付き
          Text(l10n.welcomeMessage(userName)),  // "ようこそ、{userName}さん"

          // 複数形
          Text(l10n.itemCount(cartItems.length)),  // "3件のアイテム"

          // 選択形
          Text(l10n.bookingStatus(booking.status)),  // "予約確定"
        ],
      ),
    );
  }
}

/// フォーマッタとの併用
class PriceDisplay extends ConsumerWidget {
  final double price;
  final String currency;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final locale = ref.watch(localeNotifierProvider);
    final formatter = TripTripNumberFormatter(
      locale: locale,
      currencyCode: currency,
    );

    return Text(
      S.of(context).priceDisplay(formatter.formatCurrency(price)),
    );
  }
}

/// 日付フォーマット
class BookingDateRange extends ConsumerWidget {
  final DateTime checkIn;
  final DateTime checkOut;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final locale = ref.watch(localeNotifierProvider);
    final dateFormatter = TripTripDateFormatter(locale);

    return Text(
      S.of(context).dateRange(
        dateFormatter.formatMediumDate(checkIn),
        dateFormatter.formatMediumDate(checkOut),
      ),
    );
  }
}
```

### 6.2 バックエンドでの多言語データ管理

#### 6.2.1 データベース設計

```yaml
database_schema:
  localized_content_pattern:
    approach: 別テーブル方式
    rationale:
      - 言語追加が容易
      - クエリ効率が良い
      - キャッシュしやすい

    schema:
      hotels:
        columns:
          - id: uuid (PK)
          - location_lat: decimal
          - location_lng: decimal
          - price_min: integer
          - price_max: integer
          - rating: decimal
          - created_at: timestamp
          - updated_at: timestamp

      hotel_translations:
        columns:
          - id: uuid (PK)
          - hotel_id: uuid (FK)
          - locale: varchar(10)
          - name: varchar(255)
          - description: text
          - address: varchar(500)
          - amenities: jsonb
        indexes:
          - (hotel_id, locale) UNIQUE
        constraints:
          - locale IN ('ja', 'en', 'zh-CN', 'zh-TW', 'ko', ...)
```

```sql
-- PostgreSQL スキーマ例
CREATE TABLE hotels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_lat DECIMAL(10, 8) NOT NULL,
    location_lng DECIMAL(11, 8) NOT NULL,
    price_min INTEGER NOT NULL,
    price_max INTEGER NOT NULL,
    rating DECIMAL(2, 1) DEFAULT 0,
    review_count INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE hotel_translations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hotel_id UUID NOT NULL REFERENCES hotels(id) ON DELETE CASCADE,
    locale VARCHAR(10) NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    short_description VARCHAR(500),
    address VARCHAR(500),
    amenities JSONB DEFAULT '[]',
    meta_title VARCHAR(60),
    meta_description VARCHAR(160),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(hotel_id, locale)
);

-- インデックス
CREATE INDEX idx_hotel_translations_locale ON hotel_translations(locale);
CREATE INDEX idx_hotel_translations_hotel_locale ON hotel_translations(hotel_id, locale);

-- フォールバック付きビュー
CREATE VIEW hotel_with_translation AS
SELECT
    h.*,
    COALESCE(t.name, t_fallback.name) as name,
    COALESCE(t.description, t_fallback.description) as description,
    COALESCE(t.address, t_fallback.address) as address,
    COALESCE(t.amenities, t_fallback.amenities) as amenities,
    COALESCE(t.locale, t_fallback.locale) as resolved_locale
FROM hotels h
LEFT JOIN hotel_translations t ON h.id = t.hotel_id
LEFT JOIN hotel_translations t_fallback ON h.id = t_fallback.hotel_id AND t_fallback.locale = 'ja';
```

#### 6.2.2 API設計

```yaml
api_localization:
  header_based:
    header: Accept-Language
    format: "ja, en;q=0.9, zh-CN;q=0.8"
    fallback_chain:
      1. リクエストヘッダーの言語
      2. ユーザー設定言語
      3. デフォルト言語（ja）

  query_param:
    param: locale
    format: "?locale=en"
    use_case: デバッグ、特定言語の強制

  response_format:
    include_locale: true
    structure:
      data:
        id: "hotel-123"
        name: "東京グランドホテル"  # リクエスト言語で返却
        description: "..."
      meta:
        locale: "ja"
        available_locales: ["ja", "en", "zh-CN"]
```

```typescript
// Hono APIでの実装
import { Hono } from 'hono';
import { acceptLanguage } from 'hono/accept-language';

const app = new Hono();

// Accept-Language ミドルウェア
app.use('*', async (c, next) => {
  const acceptLang = c.req.header('Accept-Language');
  const queryLocale = c.req.query('locale');

  // 優先順位: クエリパラメータ > ヘッダー > デフォルト
  const locale = queryLocale || parseAcceptLanguage(acceptLang) || 'ja';

  c.set('locale', locale);
  await next();
});

// ホテル詳細API
app.get('/api/v1/hotels/:id', async (c) => {
  const hotelId = c.req.param('id');
  const locale = c.get('locale');

  const hotel = await prisma.hotel.findUnique({
    where: { id: hotelId },
    include: {
      translations: {
        where: {
          OR: [
            { locale: locale },
            { locale: 'ja' },  // フォールバック
          ],
        },
        orderBy: {
          locale: locale === 'ja' ? 'desc' : 'asc',
        },
        take: 1,
      },
    },
  });

  if (!hotel) {
    return c.json({ error: 'Hotel not found' }, 404);
  }

  const translation = hotel.translations[0];

  return c.json({
    data: {
      id: hotel.id,
      name: translation?.name,
      description: translation?.description,
      address: translation?.address,
      location: {
        lat: hotel.locationLat,
        lng: hotel.locationLng,
      },
      price: {
        min: hotel.priceMin,
        max: hotel.priceMax,
      },
      rating: hotel.rating,
      reviewCount: hotel.reviewCount,
    },
    meta: {
      locale: translation?.locale || 'ja',
      availableLocales: await getAvailableLocales(hotelId),
    },
  });
});
```

### 6.3 CDNでの言語別コンテンツ配信

```yaml
cdn_localization_strategy:
  cache_key_design:
    pattern: "{path}:{locale}:{version}"
    examples:
      - /api/v1/hotels:ja:v1.2.3
      - /api/v1/hotels:en:v1.2.3

  vary_header:
    header: "Vary: Accept-Language"
    purpose: 言語別キャッシュ分離

  cache_rules:
    static_translations:
      ttl: 24 hours
      stale_while_revalidate: 1 hour

    dynamic_content:
      ttl: 5 minutes
      stale_while_revalidate: 30 seconds

  edge_functions:
    locale_detection:
      priority:
        1. Cookie (user preference)
        2. Accept-Language header
        3. GeoIP (country → default language)

    language_routing:
      approach: Edge redirect to localized paths
      example: /hotels → /ja/hotels, /en/hotels
```

---

## 第7章：カルチャライゼーション

### 7.1 地域別UI/UXカスタマイズ

```yaml
regional_customization:
  japan:
    ui_patterns:
      - 敬語使用（です・ます調）
      - 詳細な説明テキスト
      - コンパクトなレイアウト
    color_preferences:
      - 落ち着いた色調
      - 白背景基調
    date_format: YYYY年MM月DD日
    currency: JPY (¥)
    address_format: 郵便番号 → 都道府県 → 市区町村 → 番地

  china:
    ui_patterns:
      - 赤を縁起の良い色として活用
      - 数字「8」を強調
      - ソーシャル機能の強調
    special_requirements:
      - WeChat/Alipay統合
      - 中国国内CDN対応
    date_format: YYYY年MM月DD日
    currency: CNY (¥)

  korea:
    ui_patterns:
      - 敬語（존댓말）使用
      - 鮮やかなビジュアル
      - K-POPスタイルのプロモーション
    special_requirements:
      - Naver/Kakao連携
      - 韓国決済連携
    date_format: YYYY년 MM월 DD일
    currency: KRW (₩)

  europe:
    ui_patterns:
      - プライバシー重視の設計
      - GDPR対応UI
      - シンプルなデザイン
    special_requirements:
      - Cookie同意バナー
      - データエクスポート機能
    date_format: DD/MM/YYYY (varies)
    currency: EUR (€), GBP (£), etc.

  southeast_asia:
    ui_patterns:
      - モバイルファースト（強調）
      - カラフルなビジュアル
      - ソーシャル共有機能
    special_requirements:
      - Grab/Gojek連携
      - 地域決済連携
```

### 7.2 支払い方法の地域対応

```yaml
payment_localization:
  japan:
    primary: Credit card (JCB, VISA, Mastercard)
    secondary:
      - コンビニ決済
      - 銀行振込
      - PayPay, LINE Pay

  china:
    primary:
      - WeChat Pay
      - Alipay
    secondary:
      - UnionPay
    note: 海外クレジットカード制限あり

  korea:
    primary:
      - Kakao Pay
      - Naver Pay
    secondary:
      - クレジットカード
      - 銀行振込

  europe:
    primary:
      - Credit card
      - PayPal
    regional:
      - iDEAL (Netherlands)
      - Bancontact (Belgium)
      - Sofort (Germany)

  southeast_asia:
    primary:
      - GrabPay
      - GoPay
      - Bank transfer
    secondary:
      - 現金払い（COD）
```

### 7.3 カレンダー・祝日対応

```yaml
calendar_localization:
  calendar_systems:
    gregorian:
      regions: [en, fr, de, es, it, id, ms, vi]

    japanese_imperial:
      region: ja
      display: 令和〇年も併記

    buddhist:
      region: th
      offset: +543

    lunisolar:
      regions: [zh-CN, zh-TW, ko, vi]
      use_case: 旧正月、中秋節の表示

  holiday_integration:
    data_source: Google Calendar API / Nager.Date
    features:
      - 祝日表示
      - 営業時間調整
      - 繁忙期アラート

  regional_events:
    japan:
      - ゴールデンウィーク（4-5月）
      - お盆（8月）
      - 年末年始

    china:
      - 春節（旧正月）
      - 国慶節（10月）
      - 中秋節

    korea:
      - 설날（旧正月）
      - 추석（秋夕）
```

---

## 第8章：品質保証プロセス

### 8.1 言語QA（LQA）プロセス

```yaml
lqa_process:
  quality_dimensions:
    accuracy:
      description: 原文の意味の正確な伝達
      weight: 30%
      checks:
        - 意味の誤訳
        - 情報の欠落
        - 情報の追加

    fluency:
      description: ターゲット言語としての自然さ
      weight: 25%
      checks:
        - 文法エラー
        - 不自然な表現
        - 句読点の誤り

    terminology:
      description: 用語の一貫性と正確性
      weight: 20%
      checks:
        - 用語集との不一致
        - ブランド名の誤り
        - 業界用語の誤用

    style:
      description: スタイルガイドへの準拠
      weight: 15%
      checks:
        - トーンの不一致
        - 敬語レベルの不統一
        - フォーマットの不一致

    locale_conventions:
      description: 地域規約への準拠
      weight: 10%
      checks:
        - 日付/時刻フォーマット
        - 数値フォーマット
        - 通貨表示

  severity_levels:
    critical:
      description: 法的問題、重大な誤解を招く
      score_penalty: -5
      action: 即時修正必須

    major:
      description: 意味の誤り、主要なミス
      score_penalty: -2
      action: リリース前修正

    minor:
      description: 軽微なミス、改善の余地
      score_penalty: -0.5
      action: 次回更新時修正

    preferential:
      description: スタイルの好み
      score_penalty: 0
      action: 検討事項として記録

  scoring:
    formula: 100 - (Σ penalties / word_count × 100)
    passing_threshold:
      marketing: 95
      legal: 98
      ui_strings: 90

  workflow:
    1. 翻訳完了後、LQAキューに追加
    2. ネイティブレビュアーがサンプリングレビュー
    3. エラー記録とスコア算出
    4. しきい値未満の場合、修正と再レビュー
    5. 合格後、リリース承認
```

### 8.2 自動化テスト戦略

```yaml
automated_testing:
  unit_tests:
    coverage:
      - 全翻訳キーの存在確認
      - プレースホルダーの整合性
      - 複数形ルールの検証

    implementation:
      framework: flutter_test
      ci_integration: GitHub Actions

  visual_regression:
    tool: Percy / Chromatic
    coverage:
      - 全画面の言語別スクリーンショット
      - テキスト切れ検出
      - レイアウト崩れ検出

  string_validation:
    checks:
      - 最大長超過
      - 禁止文字
      - HTMLタグの不整合
      - プレースホルダー欠落

  pseudo_localization:
    purpose: i18n問題の早期発見
    transformations:
      - 文字の拡張（Ťĥĩš → This）
      - 長さの増加（+30%）
      - 括弧で囲む（[This is a test]）
```

```dart
// 翻訳テスト例
void main() {
  group('Localization Tests', () {
    test('All keys exist in all supported locales', () async {
      final supportedLocales = ['ja', 'en', 'zh_CN', 'ko'];
      final baseKeys = await loadArbKeys('app_ja.arb');

      for (final locale in supportedLocales) {
        final localeKeys = await loadArbKeys('app_$locale.arb');

        for (final key in baseKeys) {
          expect(
            localeKeys.contains(key),
            isTrue,
            reason: 'Key "$key" missing in $locale',
          );
        }
      }
    });

    test('Placeholders are consistent across locales', () async {
      final baseArb = await loadArb('app_ja.arb');
      final enArb = await loadArb('app_en.arb');

      for (final entry in baseArb.entries) {
        if (entry.value.contains('{')) {
          final basePlaceholders = extractPlaceholders(entry.value);
          final enValue = enArb[entry.key];

          if (enValue != null) {
            final enPlaceholders = extractPlaceholders(enValue);
            expect(
              basePlaceholders,
              equals(enPlaceholders),
              reason: 'Placeholder mismatch for key "${entry.key}"',
            );
          }
        }
      }
    });

    test('No hardcoded strings in widgets', () async {
      final dartFiles = await glob('lib/**/*.dart');

      for (final file in dartFiles) {
        final content = await file.readAsString();

        // Text() with literal string (not S.of or l10n)
        final hardcodedPattern = RegExp(
          r"Text\(\s*['\"][^'\"]+['\"]\s*\)",
        );

        expect(
          hardcodedPattern.hasMatch(content),
          isFalse,
          reason: 'Hardcoded string found in ${file.path}',
        );
      }
    });

    test('Max length constraints are respected', () async {
      const maxLengths = {
        'buttonRetry': 15,
        'buttonCancel': 15,
        'hotelSearchTitle': 20,
      };

      for (final locale in ['ja', 'en', 'zh_CN', 'ko']) {
        final arb = await loadArb('app_$locale.arb');

        for (final entry in maxLengths.entries) {
          final value = arb[entry.key];
          if (value != null) {
            expect(
              value.length,
              lessThanOrEqualTo(entry.value),
              reason: '${entry.key} exceeds max length in $locale',
            );
          }
        }
      }
    });
  });
}
```

### 8.3 ネイティブレビュアーネットワーク

```yaml
reviewer_network:
  structure:
    in_house:
      languages: [ja, en]
      role: コアレビュー、ガイドライン策定

    partner_agencies:
      coverage: 全対応言語
      selection_criteria:
        - 旅行業界経験
        - モバイルアプリ翻訳経験
        - ISO 17100認定

    community:
      role: フィードバック、ベータテスト
      incentive: 早期アクセス、クレジット表記

  reviewer_guidelines:
    qualifications:
      - ネイティブスピーカー
      - 旅行/ホスピタリティ業界知識
      - アプリUIの理解

    training:
      - TripTripブランドガイドライン
      - 用語集
      - LQAプロセス

    feedback_loop:
      - 週次レビュー会議
      - 四半期ガイドライン更新
      - 年次品質監査
```

---

## 第9章：運用 & 継続的改善

### 9.1 翻訳更新デプロイフロー

```yaml
deployment_workflow:
  continuous_localization:
    trigger: Git push to main branch
    pipeline:
      1. Lokalise webhook receives push
      2. New/modified keys synced to TMS
      3. Auto-assign to translators
      4. MT pre-translation (if enabled)
      5. Translator notification

  translation_to_release:
    pipeline:
      1. Translation completed in TMS
      2. Automated QA checks pass
      3. PR auto-created to repository
      4. CI tests run (including l10n tests)
      5. Human review (if needed)
      6. Merge to main
      7. Release in next deployment

  hotfix_flow:
    use_case: 緊急翻訳修正
    sla: 4時間以内
    pipeline:
      1. TMSで直接編集
      2. 即座にdownload実行
      3. 緊急PRとして作成
      4. 承認後即マージ
      5. ホットフィックスリリース
```

```yaml
# GitHub Actions ワークフロー
name: Localization Sync

on:
  push:
    branches: [main]
    paths:
      - 'lib/l10n/**'
  schedule:
    - cron: '0 */6 * * *'  # 6時間ごと
  workflow_dispatch:

jobs:
  sync-translations:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Lokalise CLI
        run: |
          curl -sfL https://raw.githubusercontent.com/lokalise/lokalise-cli-2-go/master/install.sh | sh

      - name: Pull translations
        env:
          LOKALISE_API_TOKEN: ${{ secrets.LOKALISE_API_TOKEN }}
          LOKALISE_PROJECT_ID: ${{ secrets.LOKALISE_PROJECT_ID }}
        run: |
          lokalise2 file download \
            --token $LOKALISE_API_TOKEN \
            --project-id $LOKALISE_PROJECT_ID \
            --format arb \
            --original-filenames true \
            --directory-prefix "" \
            --unzip-to lib/l10n/arb

      - name: Generate l10n
        run: flutter gen-l10n

      - name: Run tests
        run: flutter test test/l10n/

      - name: Create PR if changes
        uses: peter-evans/create-pull-request@v5
        with:
          title: 'chore(l10n): Update translations'
          body: 'Automated translation sync from Lokalise'
          branch: l10n/auto-sync
          labels: l10n, automated
```

### 9.2 パフォーマンスモニタリング

```yaml
performance_monitoring:
  metrics:
    translation_coverage:
      description: 翻訳済みキーの割合
      target: 100%
      alert_threshold: <95%

    time_to_translate:
      description: キー追加から翻訳完了までの時間
      target: <48 hours
      alert_threshold: >72 hours

    lqa_score:
      description: 言語QAスコア平均
      target: >90%
      alert_threshold: <85%

    user_language_distribution:
      description: 言語別ユーザー分布
      purpose: リソース配分の最適化

  dashboards:
    translation_status:
      - 言語別翻訳進捗
      - 未翻訳キー一覧
      - レビュー待ちキー数

    quality_metrics:
      - 言語別LQAスコア推移
      - エラーカテゴリ分布
      - 翻訳者パフォーマンス

    user_feedback:
      - 言語関連のバグ報告
      - 翻訳改善リクエスト
      - NPS by language
```

### 9.3 ユーザーフィードバック収集

```yaml
feedback_collection:
  in_app_feedback:
    trigger:
      - 言語設定変更後
      - 特定画面でのエラー
      - 定期的なサーベイ
    questions:
      - 翻訳の自然さ（5段階）
      - 理解しやすさ（5段階）
      - 改善提案（自由記述）

  translation_suggestion:
    feature: ユーザーからの翻訳提案
    ui:
      - テキスト長押しで「翻訳を提案」
      - 専用フィードバックフォーム
    workflow:
      1. ユーザーが提案を送信
      2. TMSに「suggestion」タグで登録
      3. レビュアーが評価
      4. 採用の場合、翻訳に反映
      5. ユーザーに通知（オプション）

  community_program:
    name: TripTrip Translation Community
    benefits:
      - 新言語への早期アクセス
      - コミュニティバッジ
      - 年間サブスクリプション割引
    activities:
      - ベータ翻訳レビュー
      - 用語集への提案
      - ローカル表現の提案
```

### 9.4 A/Bテストによる最適化

```yaml
ab_testing:
  use_cases:
    cta_optimization:
      description: CTAボタンテキストの最適化
      example:
        variant_a: "今すぐ予約"
        variant_b: "予約する"
        variant_c: "空室をチェック"
      metrics:
        - クリック率
        - コンバージョン率

    tone_testing:
      description: トーン・スタイルの最適化
      example:
        variant_a: フォーマル（です・ます）
        variant_b: カジュアル（だ・である）
      metrics:
        - エンゲージメント
        - ユーザー満足度

    message_length:
      description: メッセージ長の最適化
      example:
        variant_a: 詳細な説明
        variant_b: 簡潔な説明
      metrics:
        - 読了率
        - アクション完了率

  implementation:
    tool: Firebase Remote Config
    segmentation:
      - 言語
      - 地域
      - ユーザー属性
    statistical_significance: 95%
    minimum_sample: 1000 users/variant
```

---

## 第10章：文書間参照 & 統合ポイント

### 10.1 関連文書参照

```yaml
document_references:
  architecture:
    - Doc-SA-001: システムアーキテクチャ概要
      relevance: i18nがシステム全体にどう統合されるか

    - Doc-SA-003: APIアーキテクチャ
      relevance: 多言語APIレスポンス設計

    - Doc-AD-001: モバイルアプリアーキテクチャ
      relevance: Flutterでのi18n実装パターン

  business:
    - Doc-GL-001: APAC市場参入戦略
      relevance: Phase 2言語展開の優先順位

    - Doc-GL-002: 欧米市場参入戦略
      relevance: Phase 3言語展開の優先順位

  operations:
    - Doc-QA-001: 品質保証戦略
      relevance: LQAプロセスの品質基準との整合

    - Doc-DO-001: DevOps戦略
      relevance: 翻訳CI/CDパイプラインとの統合
```

### 10.2 実装優先度

```yaml
implementation_priorities:
  immediate:
    - Flutter intl基盤構築
    - ARBファイル構造確立
    - Lokalise統合
    - 日英翻訳完了

  short_term:
    - APAC言語追加（zh-CN, zh-TW, ko）
    - LQAプロセス確立
    - 自動化テスト構築

  medium_term:
    - 欧州言語追加（fr, de, es, it）
    - カルチャライゼーション実装
    - A/Bテスト基盤

  long_term:
    - 東南アジア言語追加
    - RTL言語対応準備
    - AI翻訳品質向上
```

### 10.3 成功指標サマリー

```yaml
success_metrics:
  coverage:
    target: 20言語対応（2027年末）
    milestone:
      - 2025 Q2: 2言語（ja, en）
      - 2025 Q4: 5言語（+zh-CN, zh-TW, ko）
      - 2026 Q2: 9言語（+fr, de, es, it）
      - 2027: 13+言語（+th, vi, id, ms, ...）

  quality:
    lqa_score: >90%（全言語平均）
    user_satisfaction: 4.5/5（翻訳品質）
    bug_rate: <5件/月（i18n関連）

  efficiency:
    time_to_translate: <48時間
    automation_rate: >70%（MT + auto-QA）
    cost_per_word: <$0.10（平均）

  business_impact:
    localized_market_conversion: +50%
    user_retention_improvement: +20%
    revenue_per_language: +15%
```

---

## 付録A：用語集

| 用語 | 定義 |
|------|------|
| i18n | Internationalization（国際化）- ソフトウェアを複数の言語・地域に対応できるよう設計すること |
| L10n | Localization（ローカライゼーション）- 特定の言語・地域向けにコンテンツを翻訳・適応すること |
| TMS | Translation Management System - 翻訳プロジェクトを管理するシステム |
| ARB | Application Resource Bundle - Flutterで使用される翻訳ファイル形式 |
| MT | Machine Translation - 機械翻訳 |
| HT | Human Translation - 人間による翻訳 |
| LQA | Linguistic Quality Assurance - 言語品質保証 |
| RTL | Right-to-Left - 右から左に書く言語（アラビア語、ヘブライ語など） |
| ICU | International Components for Unicode - Unicode処理のためのライブラリ |
| Transcreation | 翻訳と創作を組み合わせた、文化的適応を含む翻訳手法 |

---

## 付録B：言語コード一覧

| コード | 言語 | 地域 | Phase |
|--------|------|------|-------|
| ja | 日本語 | 日本 | 1 |
| en | 英語 | グローバル | 1 |
| zh-CN | 簡体字中国語 | 中国本土 | 2 |
| zh-TW | 繁体字中国語 | 台湾、香港 | 2 |
| ko | 韓国語 | 韓国 | 2 |
| fr | フランス語 | フランス | 3 |
| de | ドイツ語 | ドイツ | 3 |
| es | スペイン語 | スペイン | 3 |
| it | イタリア語 | イタリア | 3 |
| th | タイ語 | タイ | 4 |
| vi | ベトナム語 | ベトナム | 4 |
| id | インドネシア語 | インドネシア | 4 |
| ms | マレー語 | マレーシア | 4 |

---

## 変更履歴

| バージョン | 日付 | 変更内容 | 著者 |
|------------|------|----------|------|
| 1.0.0 | 2025-01-21 | 初版作成 | TripTrip Technical Team |

---

*本文書は、TripTripのグローバル展開を技術面から支える多言語・ローカライゼーション戦略の包括的なガイドラインです。Netflix、Spotify、Airbnbレベルの国際化品質を目指し、継続的に更新されます。*

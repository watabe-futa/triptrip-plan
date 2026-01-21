# Doc-SP-003: TripTrip UI/UXデザインシステム仕様書

## エグゼクティブサマリー

本文書は、TripTripプラットフォームにおける包括的なUI/UXデザインシステム仕様を定義します。Flutter/Dartベースのモバイルアプリケーションを中心に、Google Material Design 3、Apple Human Interface Guidelines、WCAG 2.1 AAアクセシビリティ基準を統合した、世界トップレベルのデザインシステムを構築します。本仕様書は、Airbnb、Google、Appleなどの業界リーダーのデザインプラクティスを参考に、TripTripブランドアイデンティティ「Discover the Real Japan」を体現する、直感的で美しく、アクセシブルなユーザー体験の実現を目指します。

### 主要ハイライト

- **デザインフィロソフィー**: Modern Japonica — 日本の美意識と現代性の融合
- **プライマリカラー**: TripTrip Red (#E63946) / TripTrip Navy (#1D3557)
- **タイポグラフィ**: Noto Sans / Noto Sans JP（多言語対応）
- **アクセシビリティ**: WCAG 2.1 AA準拠、スクリーンリーダー完全対応
- **プラットフォーム**: iOS、Android、Web（Flutter統一実装）

---

## 1. はじめに & デザインビジョン

### 1.1 目的 & スコープ

#### 1.1.1 文書の目的

本仕様書は、TripTripデザインシステムの完全な技術仕様を提供し、以下の目的を達成します：

```yaml
document_objectives:
  primary:
    - "デザイナーと開発者間の共通言語の確立"
    - "一貫したユーザー体験の保証"
    - "開発効率の向上とコード品質の標準化"
    - "ブランドアイデンティティの技術的実装"

  secondary:
    - "新規チームメンバーのオンボーディング支援"
    - "外部パートナーとの協業基盤"
    - "将来の拡張性確保"
```

#### 1.1.2 対象読者

```
対象読者:
├── プロダクトデザイナー
│   └── UIデザイン、UXリサーチ担当
├── フロントエンド開発者
│   └── Flutter/Dart実装担当
├── プロダクトマネージャー
│   └── 機能要件定義担当
├── QAエンジニア
│   └── UI/UXテスト担当
└── 外部パートナー
    └── API連携、ウィジェット開発担当
```

#### 1.1.3 スコープ定義

```yaml
scope:
  included:
    platforms:
      - "iOS（iPhone、iPad）"
      - "Android（スマートフォン、タブレット）"
      - "Web（レスポンシブ対応）"

    components:
      - "デザイントークン（カラー、タイポグラフィ、スペーシング等）"
      - "基礎UIコンポーネント"
      - "複合コンポーネント"
      - "ナビゲーションパターン"
      - "インタラクション仕様"
      - "アクセシビリティガイドライン"

    deliverables:
      - "Flutterコンポーネントライブラリ"
      - "デザイントークンJSON/YAML"
      - "Figmaコンポーネントライブラリ（参照）"

  excluded:
    - "バックエンドAPI仕様（Doc-SP-001参照）"
    - "データベーススキーマ（Doc-SP-002参照）"
    - "マーケティング素材デザイン"
    - "印刷物デザイン"
```

### 1.2 デザイン原則

#### 1.2.1 TripTripデザインフィロソフィー

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│              "Modern Japonica"                                  │
│                                                                 │
│    日本の伝統的な美意識と現代のデジタルデザインの融合          │
│                                                                 │
│    ┌─────────────────┐     ┌─────────────────┐                 │
│    │   日本の美学    │  +  │  現代デザイン  │                 │
│    ├─────────────────┤     ├─────────────────┤                 │
│    │ ・間（ma）     │     │ ・ミニマリズム │                 │
│    │ ・侘び寂び     │     │ ・レスポンシブ │                 │
│    │ ・自然との調和 │     │ ・アクセシブル │                 │
│    │ ・細部への拘り │     │ ・パフォーマンス│                 │
│    └─────────────────┘     └─────────────────┘                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 1.2.2 5つのコアデザイン原則

```yaml
core_principles:

  principle_1_clarity:
    name: "明確性（Clarity）"
    japanese: "明晰"
    description: "情報は明確で理解しやすく、ユーザーの意思決定を支援する"
    guidelines:
      - "重要な情報を視覚的階層で強調"
      - "専門用語を避け、平易な言葉を使用"
      - "アイコンとテキストラベルの併用"
      - "エラーは原因と解決策を明示"
    do:
      - "「予約を確定する」のような明確なCTA"
      - "価格は税込み総額を目立つ位置に表示"
      - "ステップインジケーターで進捗を可視化"
    dont:
      - "「処理」のような曖昧なボタンラベル"
      - "重要情報の小さいフォントサイズ"
      - "アイコンのみで意味を伝える"

  principle_2_efficiency:
    name: "効率性（Efficiency）"
    japanese: "効率"
    description: "最小限のステップで目標を達成できるフロー設計"
    guidelines:
      - "タスク完了までのステップ数を最小化"
      - "よく使う機能へのクイックアクセス提供"
      - "スマートデフォルト値の設定"
      - "入力の自動補完と予測提案"
    do:
      - "以前の検索条件の自動復元"
      - "ワンタップ再予約機能"
      - "フォーム入力の段階的開示"
    dont:
      - "不要な確認ダイアログの連発"
      - "情報の過度な分散配置"
      - "必須でない入力フィールドの多用"

  principle_3_consistency:
    name: "一貫性（Consistency）"
    japanese: "統一"
    description: "全体を通じて統一されたパターンとインタラクション"
    guidelines:
      - "同じアクションには同じ見た目と動作"
      - "プラットフォーム慣習の尊重"
      - "ナビゲーションパターンの統一"
      - "用語の一貫した使用"
    do:
      - "全画面で統一されたボトムナビゲーション"
      - "同じジェスチャーで同じ結果"
      - "統一されたカード形式での情報表示"
    dont:
      - "画面ごとに異なる戻るボタンの位置"
      - "同じ機能に異なるアイコン使用"
      - "不統一なカラー適用"

  principle_4_feedback:
    name: "フィードバック（Feedback）"
    japanese: "応答"
    description: "すべてのユーザーアクションに適切な応答を提供"
    guidelines:
      - "操作の即座の視覚的フィードバック"
      - "処理中の状態を明確に表示"
      - "成功・エラーの明確な通知"
      - "元に戻す機能の提供"
    do:
      - "ボタンタップ時のリップルエフェクト"
      - "「カートに追加しました」のスナックバー"
      - "ローディング中のスケルトン表示"
    dont:
      - "クリックしても何も変化しないUI"
      - "サイレントな処理失敗"
      - "長時間のブロッキングローディング"

  principle_5_delight:
    name: "感動（Delight）"
    japanese: "感動"
    description: "機能性を超えた、心地よく記憶に残る体験"
    guidelines:
      - "意味のあるマイクロインタラクション"
      - "美しいビジュアルとトランジション"
      - "パーソナライズされた体験"
      - "期待を超えるサプライズ要素"
    do:
      - "予約完了時の祝福アニメーション"
      - "目的地の美しい写真での歓迎"
      - "季節に応じたテーマ変更"
    dont:
      - "過度なアニメーションによる遅延感"
      - "無機質なシステムメッセージ"
      - "意味のない装飾的要素の多用"
```

### 1.3 ブランドガイドラインとの関係

#### 1.3.1 ブランドアイデンティティの反映

```yaml
brand_integration:
  brand_promise: "Discover the Real Japan"

  visual_expression:
    logo_usage:
      primary: "TripTrip ロゴマーク + ロゴタイプ"
      secondary: "ロゴマークのみ（アプリアイコン等）"
      clearspace: "ロゴ高さの50%を周囲に確保"
      minimum_size: "24dp（デジタル）"

    brand_colors:
      mapping:
        triptrip_red: "プライマリアクション、ブランド要素"
        triptrip_navy: "テキスト、セカンダリ要素"
        sakura_pink: "アクセント、ハイライト"
        forest_green: "成功状態、ポジティブ要素"
        gold_accent: "プレミアム、特別要素"

  tone_of_voice:
    characteristics:
      - "Knowledgeable（知識豊富）"
      - "Warm（温かい）"
      - "Confident（自信のある）"
      - "Inspiring（インスピレーション）"

    ui_text_guidelines:
      - "フレンドリーだがプロフェッショナル"
      - "明確で簡潔な表現"
      - "ユーザーを「あなた」で呼称"
      - "ポジティブな言い回しを優先"

  cultural_considerations:
    japanese_aesthetics:
      - element: "間（Ma）"
        application: "十分な余白、呼吸するレイアウト"
      - element: "侘び寂び（Wabi-Sabi）"
        application: "シンプルさの中の深み、自然な質感"
      - element: "もてなし（Omotenashi）"
        application: "期待を超えるサービス精神の表現"
```

---

## 2. デザイントークン仕様

### 2.1 カラーシステム

#### 2.1.1 プライマリカラー

```yaml
primary_colors:
  triptrip_red:
    hex: "#E63946"
    rgb: "rgb(230, 57, 70)"
    hsl: "hsl(355, 78%, 56%)"
    usage:
      - "プライマリCTAボタン"
      - "ブランドロゴ"
      - "重要なアクセント"
      - "選択状態のハイライト"
    accessibility:
      contrast_white: "4.52:1"  # AA準拠
      contrast_black: "4.64:1"  # AA準拠
    variants:
      light: "#FF6B6B"
      dark: "#C1121F"
      container: "#FFE5E7"
      on_container: "#8B0000"

  triptrip_navy:
    hex: "#1D3557"
    rgb: "rgb(29, 53, 87)"
    hsl: "hsl(215, 50%, 23%)"
    usage:
      - "プライマリテキスト"
      - "ナビゲーション要素"
      - "ヘッダー背景"
      - "アイコン"
    accessibility:
      contrast_white: "11.02:1"  # AAA準拠
    variants:
      light: "#457B9D"
      dark: "#0D1B2A"
      container: "#E8EEF4"
      on_container: "#0D1B2A"
```

#### 2.1.2 セカンダリカラー

```yaml
secondary_colors:
  sakura_pink:
    hex: "#FFB4A2"
    rgb: "rgb(255, 180, 162)"
    usage:
      - "背景アクセント"
      - "通知バッジ"
      - "女性向けコンテンツ"
      - "春季テーマ"
    variants:
      light: "#FFCDB2"
      dark: "#E5989B"
      container: "#FFF5F3"

  forest_green:
    hex: "#2A9D8F"
    rgb: "rgb(42, 157, 143)"
    usage:
      - "成功状態"
      - "完了アイコン"
      - "エコ・サステナブル要素"
      - "自然関連コンテンツ"
    variants:
      light: "#52B69A"
      dark: "#1A7A6D"
      container: "#E6F5F3"

  gold_accent:
    hex: "#E9C46A"
    rgb: "rgb(233, 196, 106)"
    usage:
      - "プレミアム要素"
      - "評価スター"
      - "特別オファー"
      - "ポイント表示"
    variants:
      light: "#F4D35E"
      dark: "#D4A03A"
      container: "#FDF8E8"
```

#### 2.1.3 セマンティックカラー

```yaml
semantic_colors:
  success:
    primary: "#2A9D8F"
    light: "#E6F5F3"
    dark: "#1A7A6D"
    on_color: "#FFFFFF"
    usage:
      - "成功メッセージ"
      - "完了状態"
      - "ポジティブな変化"
      - "チェックマーク"

  warning:
    primary: "#F4A261"
    light: "#FEF3E8"
    dark: "#D68A4A"
    on_color: "#1D3557"
    usage:
      - "警告メッセージ"
      - "注意喚起"
      - "残り在庫少"
      - "期限間近"

  error:
    primary: "#E63946"
    light: "#FFE5E7"
    dark: "#C1121F"
    on_color: "#FFFFFF"
    usage:
      - "エラーメッセージ"
      - "バリデーションエラー"
      - "削除アクション"
      - "必須入力"

  info:
    primary: "#457B9D"
    light: "#E8EEF4"
    dark: "#2C5A7C"
    on_color: "#FFFFFF"
    usage:
      - "情報メッセージ"
      - "ヘルプテキスト"
      - "ツールチップ"
      - "リンク"
```

#### 2.1.4 ニュートラルカラー

```yaml
neutral_colors:
  gray_scale:
    gray_50:
      hex: "#FAFAFA"
      usage: "最も明るい背景"
    gray_100:
      hex: "#F5F5F5"
      usage: "カード背景、セクション区切り"
    gray_200:
      hex: "#EEEEEE"
      usage: "ボーダー、ディバイダー"
    gray_300:
      hex: "#E0E0E0"
      usage: "無効状態の背景"
    gray_400:
      hex: "#BDBDBD"
      usage: "プレースホルダー、無効アイコン"
    gray_500:
      hex: "#9E9E9E"
      usage: "セカンダリテキスト"
    gray_600:
      hex: "#757575"
      usage: "キャプション、メタ情報"
    gray_700:
      hex: "#616161"
      usage: "サブテキスト"
    gray_800:
      hex: "#424242"
      usage: "ボディテキスト"
    gray_900:
      hex: "#212121"
      usage: "ヘッドライン、重要テキスト"

  background:
    primary: "#FFFFFF"
    secondary: "#F8F9FA"
    tertiary: "#F1F3F4"
    elevated: "#FFFFFF"

  surface:
    primary: "#FFFFFF"
    secondary: "#F8F9FA"
    variant: "#E8EEF4"
```

#### 2.1.5 ダークモード対応

```yaml
dark_mode_colors:
  background:
    primary: "#121212"
    secondary: "#1E1E1E"
    tertiary: "#2C2C2C"
    elevated: "#1E1E1E"

  surface:
    primary: "#1E1E1E"
    secondary: "#2C2C2C"
    variant: "#3A3A3A"

  text:
    primary: "#FFFFFF"
    secondary: "#B3B3B3"
    tertiary: "#808080"
    disabled: "#666666"

  primary_adjusted:
    triptrip_red: "#FF6B6B"  # 明度を上げる
    triptrip_navy: "#A8DADC"  # 明度を上げる

  semantic_adjusted:
    success: "#52B69A"
    warning: "#FFB347"
    error: "#FF6B6B"
    info: "#87CEEB"

  elevation_overlay:
    level_0: "0%"
    level_1: "5%"
    level_2: "7%"
    level_3: "8%"
    level_4: "9%"
    level_5: "11%"
```

### 2.2 タイポグラフィ

#### 2.2.1 フォントファミリー

```yaml
font_families:
  primary:
    name: "Noto Sans"
    fallback: "system-ui, -apple-system, sans-serif"
    usage: "英語、数字、記号"
    weights:
      - weight: 400
        name: "Regular"
        usage: "ボディテキスト"
      - weight: 500
        name: "Medium"
        usage: "強調テキスト、ボタン"
      - weight: 600
        name: "SemiBold"
        usage: "サブヘッディング"
      - weight: 700
        name: "Bold"
        usage: "ヘッディング"

  japanese:
    name: "Noto Sans JP"
    fallback: "Hiragino Sans, Yu Gothic, sans-serif"
    usage: "日本語テキスト"
    weights:
      - weight: 400
        name: "Regular"
      - weight: 500
        name: "Medium"
      - weight: 700
        name: "Bold"

  monospace:
    name: "Noto Sans Mono"
    fallback: "Consolas, Monaco, monospace"
    usage: "コード、予約番号、数値"
```

#### 2.2.2 タイプスケール

```yaml
type_scale:
  # Display - 大見出し、ヒーロー
  display_large:
    size: 57
    line_height: 64
    letter_spacing: -0.25
    weight: 400
    usage: "ランディングページヒーロー"

  display_medium:
    size: 45
    line_height: 52
    letter_spacing: 0
    weight: 400
    usage: "セクションヒーロー"

  display_small:
    size: 36
    line_height: 44
    letter_spacing: 0
    weight: 400
    usage: "カードヒーロー"

  # Headline - ページ・セクション見出し
  headline_large:
    size: 32
    line_height: 40
    letter_spacing: 0
    weight: 700
    usage: "ページタイトル"

  headline_medium:
    size: 28
    line_height: 36
    letter_spacing: 0
    weight: 600
    usage: "セクションタイトル"

  headline_small:
    size: 24
    line_height: 32
    letter_spacing: 0
    weight: 600
    usage: "カードタイトル"

  # Title - 小見出し
  title_large:
    size: 22
    line_height: 28
    letter_spacing: 0
    weight: 500
    usage: "リストアイテムタイトル"

  title_medium:
    size: 16
    line_height: 24
    letter_spacing: 0.15
    weight: 500
    usage: "タブ、ナビゲーション"

  title_small:
    size: 14
    line_height: 20
    letter_spacing: 0.1
    weight: 500
    usage: "サブタイトル"

  # Body - 本文
  body_large:
    size: 16
    line_height: 24
    letter_spacing: 0.5
    weight: 400
    usage: "本文テキスト"

  body_medium:
    size: 14
    line_height: 20
    letter_spacing: 0.25
    weight: 400
    usage: "説明テキスト"

  body_small:
    size: 12
    line_height: 16
    letter_spacing: 0.4
    weight: 400
    usage: "補足テキスト"

  # Label - ラベル、ボタン
  label_large:
    size: 14
    line_height: 20
    letter_spacing: 0.1
    weight: 500
    usage: "ボタンテキスト"

  label_medium:
    size: 12
    line_height: 16
    letter_spacing: 0.5
    weight: 500
    usage: "タグ、バッジ"

  label_small:
    size: 11
    line_height: 16
    letter_spacing: 0.5
    weight: 500
    usage: "キャプション"
```

#### 2.2.3 日本語タイポグラフィ考慮事項

```yaml
japanese_typography:
  line_height_adjustment:
    description: "日本語は英語より高い行間が必要"
    ratio: 1.7  # 英語の1.5に対して

  character_spacing:
    description: "日本語は詰め字を避ける"
    value: "0 or positive"

  mixed_content:
    description: "日英混在テキストの処理"
    rules:
      - "ベースラインの調整"
      - "フォントサイズの微調整（日本語を約95%）"
      - "語間の自然な調整"

  vertical_text:
    support: "限定的"
    usage: "装飾的要素、伝統的コンテンツ"
    implementation: "CSSのwriting-mode: vertical-rl"

  truncation:
    single_line: "..."
    multi_line: "最大行数で切り詰め + ..."
    japanese_specific: "全角「…」使用可"
```

### 2.3 スペーシング & サイジング

#### 2.3.1 スペーシングスケール

```yaml
spacing_scale:
  base_unit: 4  # dp

  tokens:
    xxs:
      value: 2
      usage: "アイコン内パディング"
    xs:
      value: 4
      usage: "密なリスト間隔、インラインスペース"
    sm:
      value: 8
      usage: "コンパクトなパディング、関連要素間"
    md:
      value: 12
      usage: "標準パディング"
    base:
      value: 16
      usage: "基本パディング、セクション内間隔"
    lg:
      value: 24
      usage: "セクション間隔"
    xl:
      value: 32
      usage: "大きなセクション間隔"
    xxl:
      value: 48
      usage: "ページセクション間"
    xxxl:
      value: 64
      usage: "ヒーローエリア"

  inset:
    description: "内側の余白（パディング）"
    variants:
      none: 0
      xs: 4
      sm: 8
      md: 16
      lg: 24
      xl: 32

  stack:
    description: "垂直方向の間隔"
    variants:
      none: 0
      xs: 4
      sm: 8
      md: 16
      lg: 24
      xl: 32

  inline:
    description: "水平方向の間隔"
    variants:
      none: 0
      xs: 4
      sm: 8
      md: 12
      lg: 16
      xl: 24
```

#### 2.3.2 サイジングシステム

```yaml
sizing:
  touch_targets:
    minimum: 44  # iOS HIG準拠
    recommended: 48  # Material Design準拠
    comfortable: 56

  icons:
    xs: 12
    sm: 16
    md: 20
    lg: 24
    xl: 32
    xxl: 48

  avatars:
    xs: 24
    sm: 32
    md: 40
    lg: 56
    xl: 72
    xxl: 96

  thumbnails:
    xs: 40
    sm: 56
    md: 80
    lg: 120
    xl: 160

  buttons:
    height:
      small: 32
      medium: 44
      large: 56
    min_width:
      small: 64
      medium: 88
      large: 120
    icon_only:
      small: 32
      medium: 44
      large: 56

  inputs:
    height:
      small: 40
      medium: 48
      large: 56
```

### 2.4 エレベーション & シャドウ

#### 2.4.1 エレベーションシステム

```yaml
elevation_system:
  levels:
    level_0:
      value: 0
      shadow: "none"
      usage: "フラットな要素、背景"

    level_1:
      value: 1
      shadow: "0 1px 2px rgba(0,0,0,0.1), 0 1px 3px rgba(0,0,0,0.08)"
      usage: "カード、リストアイテム"

    level_2:
      value: 2
      shadow: "0 2px 4px rgba(0,0,0,0.1), 0 3px 6px rgba(0,0,0,0.08)"
      usage: "ボタン（通常状態）、検索バー"

    level_3:
      value: 3
      shadow: "0 4px 8px rgba(0,0,0,0.12), 0 6px 12px rgba(0,0,0,0.08)"
      usage: "FAB、ドロップダウン"

    level_4:
      value: 4
      shadow: "0 8px 16px rgba(0,0,0,0.14), 0 12px 24px rgba(0,0,0,0.1)"
      usage: "モーダル、ダイアログ"

    level_5:
      value: 5
      shadow: "0 16px 32px rgba(0,0,0,0.16), 0 24px 48px rgba(0,0,0,0.12)"
      usage: "ナビゲーションドロワー"

  interaction_elevation:
    rest: "level_1"
    hover: "level_2"
    pressed: "level_0"
    disabled: "level_0"
    dragging: "level_4"
```

#### 2.4.2 ボーダーとディバイダー

```yaml
borders:
  width:
    thin: 1
    medium: 1.5
    thick: 2

  style:
    solid: "solid"
    dashed: "dashed"
    dotted: "dotted"

  radius:
    none: 0
    xs: 2
    sm: 4
    md: 8
    lg: 12
    xl: 16
    xxl: 24
    full: 9999  # 完全な円形

  semantic:
    input:
      default: "1px solid #E0E0E0"
      focus: "2px solid #E63946"
      error: "2px solid #E63946"
    card:
      default: "1px solid #EEEEEE"
    divider:
      default: "1px solid #EEEEEE"

dividers:
  horizontal:
    full_width:
      height: 1
      color: "#EEEEEE"
    inset:
      height: 1
      color: "#EEEEEE"
      margin_left: 16
    middle:
      height: 1
      color: "#EEEEEE"
      margin_horizontal: 16

  vertical:
    height: "100%"
    width: 1
    color: "#EEEEEE"
```

### 2.5 アニメーション & モーション

#### 2.5.1 デュレーション

```yaml
duration:
  instant: 0
  fast: 100
  normal: 200
  slow: 300
  slower: 400
  slowest: 500

  semantic:
    micro_interaction: 100  # ボタンタップ、トグル
    state_change: 200       # 展開、折りたたみ
    page_transition: 300    # ページ遷移
    complex_animation: 400  # 複雑なアニメーション
    emphasis: 500           # 注目を集める動き
```

#### 2.5.2 イージング関数

```yaml
easing:
  standard:
    name: "standard"
    value: "cubic-bezier(0.4, 0.0, 0.2, 1)"
    usage: "一般的な動き"

  decelerate:
    name: "decelerate"
    value: "cubic-bezier(0.0, 0.0, 0.2, 1)"
    usage: "画面に入る要素"

  accelerate:
    name: "accelerate"
    value: "cubic-bezier(0.4, 0.0, 1, 1)"
    usage: "画面から出る要素"

  sharp:
    name: "sharp"
    value: "cubic-bezier(0.4, 0.0, 0.6, 1)"
    usage: "素早い応答"

  spring:
    name: "spring"
    value: "cubic-bezier(0.175, 0.885, 0.32, 1.275)"
    usage: "弾むような動き"

  emphasized:
    name: "emphasized"
    value: "cubic-bezier(0.2, 0.0, 0, 1)"
    usage: "強調したい動き"
```

#### 2.5.3 モーションパターン

```yaml
motion_patterns:
  fade:
    in:
      duration: 200
      easing: "decelerate"
      from: "opacity: 0"
      to: "opacity: 1"
    out:
      duration: 150
      easing: "accelerate"
      from: "opacity: 1"
      to: "opacity: 0"

  slide:
    up:
      duration: 300
      easing: "decelerate"
      transform: "translateY(100%) -> translateY(0)"
    down:
      duration: 250
      easing: "accelerate"
      transform: "translateY(0) -> translateY(100%)"
    left:
      duration: 300
      easing: "standard"
      transform: "translateX(100%) -> translateX(0)"
    right:
      duration: 250
      easing: "standard"
      transform: "translateX(0) -> translateX(100%)"

  scale:
    in:
      duration: 200
      easing: "decelerate"
      transform: "scale(0.9) -> scale(1)"
      with_fade: true
    out:
      duration: 150
      easing: "accelerate"
      transform: "scale(1) -> scale(0.95)"
      with_fade: true

  shared_element:
    duration: 400
    easing: "emphasized"
    description: "画面間で共有される要素のトランジション"

  stagger:
    delay_between: 50
    max_items: 8
    description: "リストアイテムの連続表示"
```

---

## 3. コンポーネントライブラリ

### 3.1 基礎コンポーネント

#### 3.1.1 ボタン

```yaml
buttons:
  primary_button:
    description: "主要アクション用のプライマリボタン"
    variants:
      - filled
      - elevated
    states:
      enabled:
        background: "triptrip_red"
        text: "white"
        elevation: "level_1"
      hovered:
        background: "triptrip_red_dark"
        elevation: "level_2"
      pressed:
        background: "triptrip_red_dark"
        elevation: "level_0"
      disabled:
        background: "gray_300"
        text: "gray_500"
        elevation: "level_0"
      loading:
        background: "triptrip_red"
        content: "CircularProgressIndicator"
    sizes:
      small:
        height: 32
        padding_horizontal: 12
        font: "label_medium"
        icon_size: 16
      medium:
        height: 44
        padding_horizontal: 24
        font: "label_large"
        icon_size: 20
      large:
        height: 56
        padding_horizontal: 32
        font: "title_medium"
        icon_size: 24
    props:
      label: "String (required)"
      onPressed: "VoidCallback?"
      isLoading: "bool = false"
      leadingIcon: "IconData?"
      trailingIcon: "IconData?"
      size: "TripTripButtonSize = medium"
      fullWidth: "bool = false"
    accessibility:
      min_touch_target: 44
      focus_ring: "2px triptrip_red"
      screen_reader: "ボタンラベルを読み上げ"

  secondary_button:
    description: "補助アクション用のセカンダリボタン"
    variants:
      - outlined
      - tonal
    states:
      enabled:
        background: "transparent"
        border: "1.5px triptrip_red"
        text: "triptrip_red"
      hovered:
        background: "triptrip_red_container"
      pressed:
        background: "triptrip_red_container"
      disabled:
        border: "1.5px gray_300"
        text: "gray_500"

  text_button:
    description: "軽微なアクション用のテキストボタン"
    states:
      enabled:
        background: "transparent"
        text: "triptrip_red"
      hovered:
        background: "rgba(triptrip_red, 0.08)"
      pressed:
        background: "rgba(triptrip_red, 0.12)"

  icon_button:
    description: "アイコンのみのボタン"
    variants:
      - standard
      - filled
      - filled_tonal
      - outlined
    sizes:
      small: 32
      medium: 44
      large: 56

  floating_action_button:
    description: "画面上の主要アクション用FAB"
    variants:
      - regular (56dp)
      - small (40dp)
      - large (96dp)
      - extended
    position: "右下から16dpマージン"
    elevation: "level_3"
```

#### 3.1.2 テキストフィールド

```yaml
text_fields:
  standard_text_field:
    description: "標準テキスト入力フィールド"
    variants:
      - filled
      - outlined
    anatomy:
      container:
        height: 56
        border_radius: 4
      label:
        position: "floating"
        animation: "200ms decelerate"
      helper_text:
        position: "below container"
        font: "body_small"
      error_text:
        position: "below container"
        color: "error"
      leading_icon:
        position: "left inside"
        size: 24
      trailing_icon:
        position: "right inside"
        size: 24
    states:
      empty:
        label: "center positioned"
        border: "1px gray_400"
      focused:
        label: "floating, scaled to 75%"
        border: "2px triptrip_red"
      filled:
        label: "floating"
        border: "1px gray_400"
      error:
        label: "floating"
        border: "2px error"
        helper: "error message"
      disabled:
        background: "gray_100"
        text: "gray_500"

  search_field:
    description: "検索専用フィールド"
    anatomy:
      height: 48
      border_radius: 24  # pill shape
      leading_icon: "search"
      trailing_icon: "clear (when has text)"
      placeholder: "検索..."
    features:
      - "入力時の自動サジェスト"
      - "最近の検索履歴"
      - "音声入力ボタン（オプション）"

  password_field:
    description: "パスワード入力フィールド"
    features:
      - "表示/非表示トグル"
      - "強度インジケーター"
      - "要件チェックリスト"

  multiline_text_field:
    description: "複数行テキスト入力"
    features:
      - "自動高さ調整"
      - "文字数カウンター"
      - "最大行数制限"
```

#### 3.1.3 カード

```yaml
cards:
  base_card:
    description: "基本カードコンポーネント"
    variants:
      - elevated
      - filled
      - outlined
    anatomy:
      container:
        border_radius: 12
        padding: 16
      media:
        position: "top (optional)"
        aspect_ratio: "16:9 or 4:3"
      content:
        title: "headline_small or title_large"
        subtitle: "body_medium"
        supporting_text: "body_medium"
      actions:
        position: "bottom"
        alignment: "end"
    states:
      default:
        elevation: "level_1"
      hovered:
        elevation: "level_2"
      pressed:
        elevation: "level_0"
      selected:
        border: "2px triptrip_red"

  product_card:
    description: "商品表示用カード"
    extends: "base_card"
    anatomy:
      image:
        aspect_ratio: "1:1 or 4:3"
        placeholder: "skeleton"
      badge:
        position: "top-left overlay"
        types: ["新着", "セール", "人気"]
      title:
        lines: 2
        overflow: "ellipsis"
      price:
        current: "bold, triptrip_red if sale"
        original: "strikethrough if sale"
      rating:
        stars: "1-5"
        count: "レビュー数"
      action:
        primary: "カートに追加"
        secondary: "お気に入り"

  hotel_card:
    description: "ホテル表示用カード"
    extends: "base_card"
    anatomy:
      image:
        aspect_ratio: "16:9"
        carousel: true
      badge:
        types: ["おすすめ", "残りわずか", "本日のセール"]
      title:
        name: "ホテル名"
        location: "エリア名"
      rating:
        score: "8.5/10"
        label: "とても良い"
      price:
        label: "1泊あたり"
        amount: "¥12,000"
        breakdown: "税・サービス料込み"
      amenities:
        icons: ["wifi", "parking", "breakfast"]

  attraction_card:
    description: "アトラクション表示用カード"
    extends: "base_card"
    anatomy:
      image:
        aspect_ratio: "3:2"
      category:
        icon: true
        label: "体験カテゴリ"
      title:
        lines: 2
      duration:
        icon: "clock"
        text: "約2時間"
      price:
        from: "¥5,000〜"
      availability:
        status: "本日空きあり"
```

#### 3.1.4 リスト

```yaml
lists:
  list_item:
    description: "基本リストアイテム"
    variants:
      - one_line
      - two_line
      - three_line
    anatomy:
      leading:
        types: ["icon", "avatar", "thumbnail", "checkbox", "radio"]
        size: "24-56dp"
      content:
        headline: "title_medium"
        supporting_text: "body_medium"
        overline: "label_small"
      trailing:
        types: ["icon", "text", "checkbox", "switch", "button"]
    states:
      default:
        background: "transparent"
      hovered:
        background: "rgba(0,0,0,0.04)"
      pressed:
        background: "rgba(0,0,0,0.08)"
      selected:
        background: "rgba(triptrip_red,0.08)"
      disabled:
        opacity: 0.38
    divider:
      full_width: true
      inset: "leading幅に合わせる"

  settings_list_item:
    description: "設定画面用リストアイテム"
    extends: "list_item"
    trailing_types:
      - "switch (トグル)"
      - "chevron (詳細へ)"
      - "value (現在値表示)"

  order_history_item:
    description: "注文履歴用リストアイテム"
    extends: "list_item"
    anatomy:
      image: "商品サムネイル"
      order_number: "注文番号"
      date: "注文日"
      status: "ステータスバッジ"
      total: "合計金額"
      action: "詳細を見る"
```

### 3.2 フォームコンポーネント

#### 3.2.1 選択コンポーネント

```yaml
selection_components:
  checkbox:
    description: "複数選択用チェックボックス"
    anatomy:
      container: "18x18dp"
      icon: "check"
    states:
      unchecked:
        border: "2px gray_600"
        background: "transparent"
      checked:
        border: "none"
        background: "triptrip_red"
        icon: "white check"
      indeterminate:
        background: "triptrip_red"
        icon: "white minus"
      disabled:
        opacity: 0.38
    with_label:
      gap: 12
      label_position: "right"

  radio_button:
    description: "単一選択用ラジオボタン"
    anatomy:
      outer_circle: "20dp"
      inner_circle: "10dp (selected)"
    states:
      unselected:
        outer: "2px gray_600"
      selected:
        outer: "2px triptrip_red"
        inner: "triptrip_red fill"

  switch:
    description: "オン/オフトグルスイッチ"
    anatomy:
      track:
        width: 52
        height: 32
        radius: 16
      thumb:
        size: 24
        offset: 4
    states:
      off:
        track: "gray_300"
        thumb: "white"
      on:
        track: "triptrip_red"
        thumb: "white"
    animation:
      duration: 200
      easing: "standard"

  dropdown:
    description: "ドロップダウン選択"
    anatomy:
      trigger:
        height: 56
        border_radius: 4
        trailing_icon: "arrow_drop_down"
      menu:
        max_height: 300
        border_radius: 4
        elevation: "level_3"
      item:
        height: 48
        padding_horizontal: 16
    states:
      closed: "trigger only"
      open: "menu displayed below"
      selected: "item highlighted"

  chip:
    description: "選択・フィルター用チップ"
    variants:
      - assist  # アシストチップ
      - filter  # フィルターチップ
      - input   # 入力チップ
      - suggestion  # サジェストチップ
    anatomy:
      height: 32
      padding_horizontal: 12
      border_radius: 8
      leading_icon: "optional"
      trailing_icon: "close for input chip"
    states:
      default:
        background: "surface"
        border: "1px outline"
      selected:
        background: "secondary_container"
        icon: "check (filter)"
      disabled:
        opacity: 0.38
```

#### 3.2.2 日付・時間選択

```yaml
date_time_pickers:
  date_picker:
    description: "日付選択コンポーネント"
    variants:
      - modal  # ダイアログ表示
      - docked # インライン表示
    anatomy:
      header:
        height: 120
        selected_date: "headline_large"
        month_year: "title_medium"
      calendar:
        grid: "7x6"
        cell_size: 40
      footer:
        actions: ["キャンセル", "確定"]
    features:
      - "月ナビゲーション"
      - "年選択"
      - "今日ハイライト"
      - "選択範囲（チェックイン/アウト）"
      - "無効日付の表示"
    styling:
      today:
        border: "1px triptrip_red"
      selected:
        background: "triptrip_red"
        text: "white"
      range:
        background: "triptrip_red_container"

  time_picker:
    description: "時間選択コンポーネント"
    variants:
      - dial  # アナログ時計
      - input # 数字入力
    anatomy:
      clock:
        size: 256
        hour_marks: 12
        minute_marks: 60
      input:
        hour: "2桁入力"
        minute: "2桁入力"
        am_pm: "セグメント選択"
    features:
      - "12/24時間形式"
      - "分単位設定（15分刻み等）"

  date_range_picker:
    description: "日付範囲選択（チェックイン/アウト）"
    anatomy:
      header:
        check_in: "label + date"
        check_out: "label + date"
        nights: "X泊"
      calendar:
        months_visible: 2
    styling:
      start_date:
        background: "triptrip_red"
        radius: "left only"
      end_date:
        background: "triptrip_red"
        radius: "right only"
      in_range:
        background: "triptrip_red_container"
```

#### 3.2.3 数量・金額入力

```yaml
numeric_inputs:
  quantity_selector:
    description: "数量選択コンポーネント"
    anatomy:
      decrement_button: "icon_button with minus"
      value_display: "center, body_large"
      increment_button: "icon_button with plus"
    layout:
      horizontal:
        width: 120
        height: 40
      vertical:
        width: 48
        height: 100
    constraints:
      min: 0
      max: 99
      step: 1
    states:
      default: "buttons enabled"
      min_reached: "decrement disabled"
      max_reached: "increment disabled"
    animation:
      value_change: "fade 100ms"

  stepper:
    description: "ステップ入力（人数等）"
    anatomy:
      label: "大人 2名"
      controls: "- / +"
    variants:
      - compact  # 横一列
      - expanded # ラベルと説明付き

  price_input:
    description: "金額入力フィールド"
    anatomy:
      currency_symbol: "¥"
      input: "数字のみ"
      thousand_separator: "カンマ区切り"
    formatting:
      on_input: "リアルタイムフォーマット"
      on_blur: "完全フォーマット"
```

### 3.3 ナビゲーションコンポーネント

#### 3.3.1 ボトムナビゲーション

```yaml
bottom_navigation:
  description: "メイン画面下部のナビゲーションバー"
  anatomy:
    container:
      height: 80
      background: "surface"
      elevation: "level_2"
      safe_area: "bottom inset対応"
    items:
      count: 5  # TripTrip: ホーム、検索、商品、カート、マイページ
      layout: "icon + label 縦並び"
  items:
    - icon: "home"
      label: "ホーム"
      route: "/"
    - icon: "search"
      label: "検索"
      route: "/search"
    - icon: "shopping_bag"
      label: "商品"
      route: "/products"
    - icon: "shopping_cart"
      label: "カート"
      route: "/cart"
      badge: "カート内アイテム数"
    - icon: "person"
      label: "マイページ"
      route: "/profile"
  states:
    unselected:
      icon_color: "gray_600"
      label_color: "gray_600"
    selected:
      icon_color: "triptrip_red"
      label_color: "triptrip_red"
      icon_style: "filled"
  animation:
    selection_indicator: "pill shape, 200ms"
    icon_scale: "1.0 -> 1.1"
```

#### 3.3.2 アプリバー

```yaml
app_bars:
  top_app_bar:
    description: "画面上部のアプリバー"
    variants:
      - small  # 高さ64dp
      - medium # 高さ112dp
      - large  # 高さ152dp
    anatomy:
      leading:
        types: ["back", "menu", "close"]
        touch_target: 48
      title:
        position: "center or start"
        style: "title_large"
      trailing:
        actions: "最大3アイコン"
        overflow: "more_vert メニュー"
    behaviors:
      scroll:
        small: "固定 or 非表示"
        medium: "折りたたみ -> small"
        large: "折りたたみ -> small"
      elevation:
        at_top: "level_0"
        scrolled: "level_2"

  search_app_bar:
    description: "検索モード用アプリバー"
    anatomy:
      back_button: "leading"
      search_field: "expanding"
      clear_button: "trailing"
    transition:
      from_regular: "300ms emphasized"
      search_field: "expand from icon"

  contextual_app_bar:
    description: "選択モード用アプリバー"
    anatomy:
      close_button: "leading"
      selection_count: "X件選択中"
      actions: "削除、共有等"
    styling:
      background: "secondary_container"
```

#### 3.3.3 タブ

```yaml
tabs:
  primary_tabs:
    description: "プライマリタブナビゲーション"
    anatomy:
      container:
        height: 48
        divider: "bottom 1px"
      tab:
        padding_horizontal: 16
        min_width: 90
    content:
      - "icon only"
      - "label only"
      - "icon + label"
    indicator:
      height: 3
      color: "triptrip_red"
      radius: "top corners only"
    states:
      unselected:
        label_color: "gray_600"
      selected:
        label_color: "triptrip_red"
    scroll:
      scrollable: true
      show_arrows: false

  secondary_tabs:
    description: "セカンダリタブ（フィルター等）"
    styling:
      indicator: "full tab background"
      border_radius: 8

  segmented_button:
    description: "セグメントボタン（2-5選択肢）"
    anatomy:
      container:
        height: 40
        border_radius: 20
        border: "1px outline"
      segment:
        min_width: 48
        checkmark: "selected時"
```

#### 3.3.4 ドロワー

```yaml
navigation_drawer:
  description: "サイドナビゲーションドロワー"
  variants:
    - standard  # 画面幅360dp以上
    - modal     # モバイル、オーバーレイ
  anatomy:
    header:
      height: 160
      content: "ユーザー情報、ロゴ"
    items:
      height: 56
      leading_icon: 24
      label: "label_large"
      badge: "trailing"
    divider:
      between_sections: true
    footer:
      settings: "設定リンク"
      version: "アプリバージョン"
  width:
    standard: 360
    modal: "画面幅 - 56"
  scrim:
    color: "rgba(0,0,0,0.32)"
  animation:
    open: "300ms decelerate"
    close: "250ms accelerate"
```

### 3.4 データ表示コンポーネント

#### 3.4.1 バッジ

```yaml
badges:
  notification_badge:
    description: "通知数を表示するバッジ"
    variants:
      - dot  # 数字なし
      - small  # 1桁
      - large  # 2桁以上
    styling:
      background: "error"
      text: "white"
      min_size: 16
      max_width: 24
    position:
      anchor: "top-right"
      offset: "-4, -4"
    content:
      max_display: 99
      overflow: "99+"

  status_badge:
    description: "ステータスを表示するバッジ"
    variants:
      - info
      - success
      - warning
      - error
    styling:
      height: 24
      padding_horizontal: 8
      border_radius: 12
      font: "label_small"

  label_badge:
    description: "ラベルバッジ（新着、セール等）"
    presets:
      new:
        background: "triptrip_red"
        text: "新着"
      sale:
        background: "error"
        text: "セール"
      popular:
        background: "gold_accent"
        text: "人気"
      limited:
        background: "warning"
        text: "残りわずか"
```

#### 3.4.2 プログレスインジケーター

```yaml
progress_indicators:
  circular_progress:
    description: "円形プログレス"
    variants:
      - indeterminate  # 不定
      - determinate    # 進捗率表示
    sizes:
      small: 24
      medium: 40
      large: 64
    styling:
      track_color: "gray_200"
      indicator_color: "triptrip_red"
      stroke_width: 4

  linear_progress:
    description: "線形プログレス"
    variants:
      - indeterminate
      - determinate
      - buffer
    styling:
      height: 4
      border_radius: 2
      track_color: "gray_200"
      indicator_color: "triptrip_red"

  step_indicator:
    description: "ステップ進捗表示"
    anatomy:
      step_circle: 32
      connector_line: "2px between"
      label: "below circle"
    states:
      completed:
        circle: "triptrip_red fill with check"
      active:
        circle: "triptrip_red outline"
      upcoming:
        circle: "gray_400 outline"
```

#### 3.4.3 評価・レーティング

```yaml
ratings:
  star_rating:
    description: "星評価表示"
    anatomy:
      icon: "star"
      size: 20
      gap: 2
    variants:
      - display  # 読み取り専用
      - input    # 入力可能
    precision:
      full: "整数のみ"
      half: "0.5単位"
    styling:
      filled: "gold_accent"
      empty: "gray_300"
      half: "半分塗りつぶし"

  score_display:
    description: "数値スコア表示"
    anatomy:
      score: "8.5"
      max: "/10"
      label: "とても良い"
    styling:
      score_font: "headline_medium"
      label_font: "body_small"
    color_coding:
      excellent: "9-10, forest_green"
      very_good: "8-8.9, forest_green"
      good: "7-7.9, gold_accent"
      fair: "6-6.9, warning"
      poor: "below 6, error"
```

#### 3.4.4 テーブル・グリッド

```yaml
data_display:
  data_table:
    description: "データテーブル"
    anatomy:
      header_row:
        height: 56
        background: "gray_50"
      data_row:
        height: 52
        divider: "bottom"
      cell:
        padding: 16
    features:
      - "ソート（昇順/降順）"
      - "ページネーション"
      - "行選択"
      - "スクロール固定ヘッダー"

  price_breakdown:
    description: "料金明細表示"
    anatomy:
      items:
        label: "start aligned"
        amount: "end aligned"
      subtotal:
        style: "body_medium"
      total:
        style: "title_large, bold"
        divider: "top"
    example:
      - label: "宿泊料金 x 2泊"
        amount: "¥24,000"
      - label: "サービス料"
        amount: "¥2,400"
      - label: "消費税"
        amount: "¥2,640"
      - label: "合計"
        amount: "¥29,040"
        style: "total"
```

### 3.5 フィードバックコンポーネント

#### 3.5.1 スナックバー

```yaml
snackbar:
  description: "一時的なフィードバックメッセージ"
  anatomy:
    container:
      min_height: 48
      max_width: 344
      border_radius: 4
    content:
      text: "body_medium"
      max_lines: 2
    action:
      text_button: "optional"
    close:
      icon_button: "optional"
  position:
    mobile: "bottom, above FAB if present"
    desktop: "bottom-left"
    margin: 16
  behavior:
    duration:
      short: 4000
      long: 7000
      indefinite: "until dismissed"
    dismissible: "swipe or action"
    queue: "1つずつ表示"
  variants:
    default:
      background: "gray_800"
      text: "white"
    success:
      background: "forest_green"
      icon: "check_circle"
    error:
      background: "error"
      icon: "error"
    warning:
      background: "warning"
      icon: "warning"
```

#### 3.5.2 ダイアログ

```yaml
dialogs:
  alert_dialog:
    description: "確認・警告ダイアログ"
    anatomy:
      container:
        min_width: 280
        max_width: 560
        border_radius: 28
        padding: 24
      icon:
        size: 24
        position: "top center"
      title:
        style: "headline_small"
        alignment: "center"
      content:
        style: "body_medium"
        max_height: "65vh"
      actions:
        alignment: "end"
        gap: 8
    variants:
      confirmation:
        actions: ["キャンセル", "確認"]
      destructive:
        title: "削除の確認"
        actions: ["キャンセル", "削除（error color）"]
      informational:
        actions: ["OK"]
    animation:
      open: "fade + scale from 0.9"
      close: "fade out"
    scrim:
      color: "rgba(0,0,0,0.32)"
      dismissible: true

  full_screen_dialog:
    description: "フルスクリーンダイアログ"
    anatomy:
      app_bar:
        leading: "close"
        title: "タイトル"
        trailing: "保存/完了"
      content: "full screen"
    usage:
      - "複雑なフォーム入力"
      - "マルチステップフロー"
    animation:
      open: "slide up"
      close: "slide down"

  bottom_sheet:
    description: "ボトムシートダイアログ"
    variants:
      - standard  # partial height
      - modal     # with scrim
      - expanding # drag to expand
    anatomy:
      handle:
        width: 32
        height: 4
        color: "gray_400"
      content:
        padding: 16
    behavior:
      dismissible: "drag down or tap scrim"
      snap_points: ["25%", "50%", "90%"]
```

#### 3.5.3 トースト

```yaml
toast:
  description: "軽量な通知メッセージ"
  anatomy:
    container:
      height: 48
      border_radius: 24
      padding_horizontal: 16
    icon:
      size: 20
      optional: true
    text:
      style: "body_medium"
      max_lines: 1
  position:
    mobile: "top, below status bar"
    tablet: "top-center"
  duration:
    default: 2000
  animation:
    enter: "fade + slide down"
    exit: "fade up"
  variants:
    success:
      icon: "check"
      background: "forest_green"
    error:
      icon: "close"
      background: "error"
    info:
      icon: "info"
      background: "info"
```

#### 3.5.4 ツールチップ

```yaml
tooltip:
  description: "ホバー/長押しで表示される補足情報"
  anatomy:
    container:
      max_width: 200
      padding: "8 12"
      border_radius: 4
    text:
      style: "body_small"
      color: "white"
    arrow:
      size: 8
      position: "pointing to anchor"
  positioning:
    preferred: "above anchor"
    fallback: "below, left, right"
    margin: 8
  behavior:
    trigger:
      desktop: "hover after 500ms"
      mobile: "long press"
    duration: "until pointer leaves"
  styling:
    background: "gray_800"
    text: "white"
```

---

## 4. アイコン & イラストレーション

### 4.1 アイコンシステム

#### 4.1.1 アイコンライブラリ概要

```yaml
icon_library:
  base_library: "Material Symbols"
  style: "Rounded"
  weight: 400
  fill: "0 (outlined) or 1 (filled)"
  optical_size: 24

  custom_icons:
    description: "TripTrip固有のカスタムアイコン"
    categories:
      - "旅行関連（チケット、パスポート等）"
      - "日本文化（鳥居、桜、着物等）"
      - "サービス固有（TripTripロゴマーク等）"
```

#### 4.1.2 アイコンサイズ規格

```yaml
icon_sizes:
  xs:
    size: 12
    usage: "インライン、小さなバッジ内"
    stroke: 1.5

  sm:
    size: 16
    usage: "コンパクトUI、テキスト横"
    stroke: 1.5

  md:
    size: 20
    usage: "標準ボタン内、リストアイコン"
    stroke: 2

  base:
    size: 24
    usage: "ナビゲーション、アクションバー"
    stroke: 2

  lg:
    size: 32
    usage: "空状態、強調アイコン"
    stroke: 2

  xl:
    size: 48
    usage: "空状態イラスト、オンボーディング"
    stroke: 2.5

  xxl:
    size: 64
    usage: "フルスクリーン空状態"
    stroke: 3
```

#### 4.1.3 カテゴリ別アイコン一覧

```yaml
icon_categories:
  navigation:
    - name: "home"
      variants: ["outlined", "filled"]
      usage: "ホーム画面ナビゲーション"
    - name: "search"
      usage: "検索機能"
    - name: "arrow_back"
      usage: "戻るアクション"
    - name: "close"
      usage: "閉じる、キャンセル"
    - name: "menu"
      usage: "ハンバーガーメニュー"
    - name: "more_vert"
      usage: "オーバーフローメニュー"
    - name: "chevron_right"
      usage: "詳細へ進む"

  actions:
    - name: "add"
      usage: "追加アクション"
    - name: "edit"
      usage: "編集"
    - name: "delete"
      usage: "削除"
    - name: "share"
      usage: "共有"
    - name: "favorite"
      variants: ["outlined", "filled"]
      usage: "お気に入り"
    - name: "bookmark"
      variants: ["outlined", "filled"]
      usage: "保存"
    - name: "download"
      usage: "ダウンロード"
    - name: "upload"
      usage: "アップロード"

  commerce:
    - name: "shopping_cart"
      variants: ["outlined", "filled"]
      usage: "カート"
    - name: "shopping_bag"
      usage: "商品"
    - name: "credit_card"
      usage: "支払い"
    - name: "receipt"
      usage: "レシート、注文履歴"
    - name: "local_offer"
      usage: "クーポン、割引"
    - name: "loyalty"
      usage: "ポイント"

  travel:
    - name: "flight"
      usage: "フライト"
    - name: "hotel"
      usage: "宿泊"
    - name: "restaurant"
      usage: "飲食"
    - name: "attractions"
      usage: "観光スポット"
    - name: "map"
      usage: "地図"
    - name: "location_on"
      usage: "位置"
    - name: "directions"
      usage: "経路"
    - name: "qr_code_scanner"
      usage: "QRスキャン"
    - name: "confirmation_number"
      usage: "予約番号"

  communication:
    - name: "notifications"
      variants: ["outlined", "filled"]
      usage: "通知"
    - name: "email"
      usage: "メール"
    - name: "chat"
      usage: "チャット"
    - name: "help"
      usage: "ヘルプ"
    - name: "info"
      usage: "情報"

  status:
    - name: "check_circle"
      usage: "成功、完了"
    - name: "error"
      usage: "エラー"
    - name: "warning"
      usage: "警告"
    - name: "schedule"
      usage: "時間、予定"
    - name: "verified"
      usage: "認証済み"

  user:
    - name: "person"
      variants: ["outlined", "filled"]
      usage: "ユーザー"
    - name: "group"
      usage: "グループ、人数"
    - name: "settings"
      usage: "設定"
    - name: "logout"
      usage: "ログアウト"

  triptrip_custom:
    - name: "tt_torii"
      description: "鳥居アイコン"
      usage: "日本文化体験"
    - name: "tt_sakura"
      description: "桜アイコン"
      usage: "季節イベント"
    - name: "tt_kimono"
      description: "着物アイコン"
      usage: "着物レンタル"
    - name: "tt_onsen"
      description: "温泉アイコン"
      usage: "温泉施設"
    - name: "tt_shinkansen"
      description: "新幹線アイコン"
      usage: "交通"
```

#### 4.1.4 アイコン使用ガイドライン

```yaml
icon_guidelines:
  color:
    default: "inherit from text color"
    interactive: "primary for actionable"
    semantic: "match semantic color (success, error, etc.)"
    disabled: "gray_400"

  pairing_with_text:
    position: "left of text (LTR languages)"
    gap: "8dp (sm), 12dp (md)"
    alignment: "center vertical"
    size_ratio: "icon height ≈ text line height"

  touch_targets:
    minimum: 44
    icon_in_button: "center within touch target"
    icon_only_button: "48x48 recommended"

  accessibility:
    decorative: "aria-hidden='true'"
    informative: "aria-label describing the icon"
    interactive: "button semantics + accessible name"

  do:
    - "一貫したスタイル（outlined/filled）を維持"
    - "文脈に適したアイコンを選択"
    - "必要に応じてラベルを併用"
    - "十分なコントラストを確保"

  dont:
    - "異なるアイコンセットを混在させない"
    - "意味が曖昧なアイコン単独使用を避ける"
    - "アイコンを過度に装飾しない"
    - "小さすぎるサイズで使用しない"
```

### 4.2 イラストレーションスタイル

#### 4.2.1 イラストレーションガイドライン

```yaml
illustration_style:
  concept: "Modern Japonica"
  characteristics:
    - "シンプルで洗練されたライン"
    - "日本の伝統色をベースにしたカラーパレット"
    - "余白を活かした構図"
    - "温かみのある人物表現"

  color_palette:
    primary: ["#E63946", "#1D3557"]
    secondary: ["#FFB4A2", "#2A9D8F", "#E9C46A"]
    backgrounds: ["#F8F9FA", "#FFF5F3", "#E8EEF4"]

  line_style:
    weight: "2px base, scale with illustration size"
    caps: "round"
    joins: "round"
    color: "triptrip_navy or darker variant"

  fill_style:
    type: "flat colors, subtle gradients"
    shadows: "soft, minimal"
    textures: "optional, subtle grain"
```

#### 4.2.2 イラストレーション用途

```yaml
illustration_usage:
  empty_states:
    description: "コンテンツがない状態を示すイラスト"
    size: "120-200dp"
    examples:
      - scene: "検索結果なし"
        illustration: "虫眼鏡を持つキャラクター"
        message: "条件に合う結果が見つかりませんでした"
      - scene: "カートが空"
        illustration: "空のショッピングバッグ"
        message: "カートに商品がありません"
      - scene: "お気に入りなし"
        illustration: "ハートを持つキャラクター"
        message: "お気に入りを追加しましょう"
      - scene: "注文履歴なし"
        illustration: "荷物を待つキャラクター"
        message: "まだ注文履歴がありません"

  onboarding:
    description: "オンボーディング画面のイラスト"
    size: "full width, aspect 4:3"
    examples:
      - step: 1
        illustration: "日本の風景と旅行者"
        message: "本物の日本を発見しよう"
      - step: 2
        illustration: "スマートフォンとチケット"
        message: "簡単予約、すぐに体験"
      - step: 3
        illustration: "多言語サポートイメージ"
        message: "24時間サポートで安心"

  success_states:
    description: "完了・成功を示すイラスト"
    examples:
      - scene: "予約完了"
        illustration: "紙吹雪と喜ぶキャラクター"
      - scene: "支払い完了"
        illustration: "チェックマークと領収書"

  error_states:
    description: "エラー・問題を示すイラスト"
    examples:
      - scene: "ネットワークエラー"
        illustration: "切れたWi-Fiアイコン"
      - scene: "サーバーエラー"
        illustration: "困った顔のサーバー"
```

### 4.3 絵文字使用ガイドライン

```yaml
emoji_guidelines:
  usage:
    allowed:
      - "ユーザー生成コンテンツ（レビュー等）"
      - "プッシュ通知"
      - "マーケティングメッセージ"
      - "カジュアルなコミュニケーション"
    restricted:
      - "UIラベル（アイコンを使用）"
      - "エラーメッセージ"
      - "フォーマルな通知"
      - "法的文書"

  rendering:
    platform: "system native emoji"
    fallback: "Noto Color Emoji"
    size: "inline with text"

  accessibility:
    screen_reader: "絵文字名を読み上げ"
    decorative: "意味を補完する文脈で使用"

  recommended_emoji:
    travel: ["✈️", "🗾", "🏯", "⛩️", "🗻", "🌸", "🍱"]
    positive: ["✨", "🎉", "👍", "❤️", "⭐"]
    status: ["✅", "⚠️", "❌", "🔔", "📍"]
```

---

## 5. レイアウト & グリッドシステム

### 5.1 レスポンシブブレークポイント

```yaml
breakpoints:
  compact:
    range: "0-599dp"
    device: "スマートフォン縦向き"
    columns: 4
    margins: 16
    gutters: 16

  medium:
    range: "600-839dp"
    device: "スマートフォン横向き、小型タブレット"
    columns: 8
    margins: 24
    gutters: 24

  expanded:
    range: "840-1199dp"
    device: "タブレット、小型デスクトップ"
    columns: 12
    margins: 32
    gutters: 24

  large:
    range: "1200-1599dp"
    device: "デスクトップ"
    columns: 12
    margins: 48
    gutters: 32
    max_content_width: 1200

  extra_large:
    range: "1600dp+"
    device: "大型デスクトップ"
    columns: 12
    margins: 64
    gutters: 32
    max_content_width: 1440
```

### 5.2 グリッド仕様

#### 5.2.1 モバイルグリッド（Compact）

```yaml
mobile_grid:
  columns: 4
  column_width: "flexible"
  gutter: 16
  margin: 16

  common_layouts:
    single_column:
      span: 4
      usage: "リスト、詳細画面"

    two_column:
      span: 2
      usage: "商品グリッド、設定"

    asymmetric:
      primary: 3
      secondary: 1
      usage: "テキスト + アクション"

  content_width:
    full_bleed: "margin無視、画面端まで"
    standard: "margin内"
    inset: "追加padding適用"
```

#### 5.2.2 タブレットグリッド（Medium/Expanded）

```yaml
tablet_grid:
  medium:
    columns: 8
    gutter: 24
    margin: 24

    layouts:
      list_detail:
        list: 3
        detail: 5
      equal_split:
        left: 4
        right: 4
      sidebar:
        nav: 2
        content: 6

  expanded:
    columns: 12
    gutter: 24
    margin: 32

    layouts:
      three_column:
        each: 4
      list_detail:
        list: 4
        detail: 8
      dashboard:
        sidebar: 3
        main: 6
        widgets: 3
```

#### 5.2.3 デスクトップグリッド（Large+）

```yaml
desktop_grid:
  columns: 12
  gutter: 32
  margin: "48-64"
  max_width: "1200-1440"

  layouts:
    standard:
      sidebar: 2
      main: 7
      aside: 3

    wide_content:
      main: 10
      margin: 1

    dashboard:
      nav: 2
      content: 10
```

### 5.3 コンテナとマージン

#### 5.3.1 コンテナタイプ

```yaml
containers:
  fluid:
    width: "100%"
    max_width: "none"
    usage: "フルブリードセクション"

  standard:
    width: "100%"
    max_width: "breakpoint specific"
    padding: "margin value"
    usage: "通常コンテンツ"

  narrow:
    width: "100%"
    max_width: "600dp"
    usage: "フォーム、読み物"

  wide:
    width: "100%"
    max_width: "1440dp"
    usage: "ダッシュボード"
```

#### 5.3.2 セーフエリア対応

```yaml
safe_areas:
  ios:
    top: "Dynamic Island / notch対応"
    bottom: "Home indicator対応（34dp）"
    implementation: "SafeArea widget"

  android:
    top: "status bar height"
    bottom: "navigation bar / gesture area"
    implementation: "SystemUiOverlayStyle"

  content_insets:
    top:
      with_app_bar: 0
      without_app_bar: "status bar + padding"
    bottom:
      with_nav_bar: "nav bar height"
      with_fab: "FAB height + margin"
      keyboard: "keyboard height"
```

### 5.4 レイアウトパターン

#### 5.4.1 一般的なレイアウトパターン

```yaml
layout_patterns:
  scaffold:
    description: "基本画面構造"
    components:
      - "TopAppBar"
      - "Content (scrollable)"
      - "BottomNavigationBar"
      - "FAB (optional)"
      - "Drawer (optional)"

  list_layout:
    description: "リスト表示画面"
    components:
      - "AppBar with search"
      - "Filter chips (optional)"
      - "Scrollable list"
      - "Pull to refresh"
    spacing:
      list_padding: 16
      item_gap: 8

  grid_layout:
    description: "グリッド表示画面"
    components:
      - "AppBar"
      - "Grid view"
    grid_config:
      compact:
        columns: 2
        aspect_ratio: "3:4"
      medium:
        columns: 3
        aspect_ratio: "3:4"
      expanded:
        columns: 4
        aspect_ratio: "3:4"

  detail_layout:
    description: "詳細画面"
    components:
      - "Collapsing header with image"
      - "Content sections"
      - "Sticky bottom CTA"
    sections:
      - "Hero image/carousel"
      - "Title & basic info"
      - "Description"
      - "Reviews"
      - "Related items"

  form_layout:
    description: "フォーム入力画面"
    components:
      - "AppBar with close/back"
      - "Form fields"
      - "Submit button"
    spacing:
      field_gap: 16
      section_gap: 24
    keyboard:
      scroll_to_focused: true
      bottom_padding: "keyboard height"

  checkout_layout:
    description: "チェックアウトフロー"
    components:
      - "Step indicator"
      - "Form content"
      - "Order summary (collapsible)"
      - "CTA button"
```

---

## 6. アクセシビリティ仕様

### 6.1 WCAG 2.1 AA準拠要件

#### 6.1.1 準拠レベル概要

```yaml
wcag_compliance:
  target_level: "AA"
  principles:
    perceivable:
      - "テキストの代替を提供"
      - "時間依存メディアの代替"
      - "適応可能なコンテンツ"
      - "区別可能なコンテンツ"
    operable:
      - "キーボード操作可能"
      - "十分な時間"
      - "発作を引き起こさない"
      - "ナビゲーション可能"
    understandable:
      - "読みやすさ"
      - "予測可能性"
      - "入力支援"
    robust:
      - "互換性"
      - "支援技術との連携"
```

#### 6.1.2 コントラスト比要件

```yaml
contrast_requirements:
  text:
    normal_text:
      minimum: "4.5:1"
      enhanced: "7:1"
      applies_to: "body text, labels"
    large_text:
      minimum: "3:1"
      enhanced: "4.5:1"
      definition: "18pt+ or 14pt bold+"

  ui_components:
    minimum: "3:1"
    applies_to:
      - "ボタン境界"
      - "フォーム入力境界"
      - "アイコン（意味を持つもの）"
      - "フォーカスインジケーター"

  color_combinations:
    approved:
      - pair: ["#1D3557", "#FFFFFF"]
        ratio: "11.02:1"
        usage: "プライマリテキスト"
      - pair: ["#E63946", "#FFFFFF"]
        ratio: "4.52:1"
        usage: "ボタン、アクセント"
      - pair: ["#757575", "#FFFFFF"]
        ratio: "4.6:1"
        usage: "セカンダリテキスト"

    requires_attention:
      - pair: ["#E63946", "#1D3557"]
        ratio: "2.44:1"
        note: "テキスト使用不可、装飾のみ"

  testing_tools:
    - "Figma Contrast plugin"
    - "WebAIM Contrast Checker"
    - "axe DevTools"
```

### 6.2 スクリーンリーダー対応

#### 6.2.1 セマンティックマークアップ

```yaml
semantic_structure:
  flutter_semantics:
    text:
      widget: "Text"
      semantics: "自動的にラベルとして認識"

    button:
      widget: "ElevatedButton, TextButton, etc."
      semantics: "button role + label"
      custom: |
        Semantics(
          button: true,
          label: 'アクションの説明',
          child: ...
        )

    image:
      decorative: |
        Semantics(
          excludeSemantics: true,
          child: Image(...)
        )
      informative: |
        Semantics(
          label: '画像の説明',
          child: Image(...)
        )

    heading:
      implementation: |
        Semantics(
          header: true,
          child: Text('見出しテキスト')
        )

    link:
      implementation: |
        Semantics(
          link: true,
          label: 'リンク先の説明',
          child: ...
        )

  semantic_hierarchy:
    page_structure:
      - "App bar title (heading level 1)"
      - "Section headings (heading level 2)"
      - "Subsection headings (heading level 3)"
    navigation:
      order: "visual order = focus order"
      grouping: "related items grouped"
```

#### 6.2.2 読み上げガイドライン

```yaml
screen_reader_guidelines:
  labels:
    buttons:
      do: "予約を確定する"
      dont: "ボタン、確定"
    icons:
      do: "お気に入りに追加"
      dont: "ハートアイコン"
    images:
      product: "商品名 + 主要特徴"
      decorative: "空文字列（excludeSemantics）"

  state_announcements:
    loading: "読み込み中"
    success: "完了しました"
    error: "エラーが発生しました + 詳細"
    selection: "選択されました / 選択解除されました"

  dynamic_content:
    live_regions:
      polite: "ユーザーアクション後の更新"
      assertive: "重要なアラート"
    implementation: |
      Semantics(
        liveRegion: true,
        child: Text(dynamicContent)
      )

  gesture_alternatives:
    swipe: "ボタンでも同じアクション可能"
    long_press: "コンテキストメニューボタンも提供"
    pinch_zoom: "ズームボタンも提供"
```

### 6.3 キーボードナビゲーション

```yaml
keyboard_navigation:
  focus_management:
    visible_focus:
      style: "2dp outline, triptrip_red"
      offset: "2dp outside element"
      implementation: |
        Focus(
          child: Container(
            decoration: BoxDecoration(
              border: Border.all(
                color: isFocused ? Colors.red : Colors.transparent,
                width: 2,
              ),
            ),
          ),
        )

    focus_order:
      rule: "左から右、上から下"
      custom: "FocusTraversalGroup"
      skip: "無効要素、装飾要素"

  keyboard_shortcuts:
    global:
      - key: "Tab"
        action: "次のフォーカス可能要素へ"
      - key: "Shift + Tab"
        action: "前のフォーカス可能要素へ"
      - key: "Enter / Space"
        action: "アクティベート"
      - key: "Escape"
        action: "モーダル/ドロップダウンを閉じる"
      - key: "Arrow keys"
        action: "リスト/グリッド内移動"

    component_specific:
      dropdown:
        - "Enter: 開く"
        - "Arrow Up/Down: 選択移動"
        - "Enter: 選択確定"
        - "Escape: キャンセル"
      tabs:
        - "Arrow Left/Right: タブ切り替え"
      slider:
        - "Arrow Left/Right: 値調整"
        - "Home/End: 最小/最大値"

  focus_trap:
    modals: "フォーカスをモーダル内に制限"
    drawers: "フォーカスをドロワー内に制限"
    implementation: "FocusTrap widget"
```

### 6.4 その他のアクセシビリティ要件

#### 6.4.1 動作とアニメーション

```yaml
motion_accessibility:
  reduced_motion:
    detection: "MediaQuery.of(context).disableAnimations"
    behavior:
      - "アニメーションを即座に完了"
      - "パララックス効果を無効化"
      - "自動再生を停止"

  vestibular_considerations:
    avoid:
      - "大きなズームアニメーション"
      - "回転アニメーション"
      - "視差効果"
    provide:
      - "アニメーション設定のユーザーコントロール"

  timing:
    auto_advancing:
      - "カルーセルは自動再生しない、または停止可能"
      - "タイムアウトは延長可能"
```

#### 6.4.2 テキストのアクセシビリティ

```yaml
text_accessibility:
  scalable_text:
    implementation: "MediaQuery.textScaleFactor対応"
    minimum: "1.0"
    maximum: "2.0 (推奨サポート範囲)"
    testing: "各スケールでレイアウト確認"

  text_spacing:
    line_height: "最低1.5倍"
    paragraph_spacing: "最低1.5倍 line height"
    letter_spacing: "0.12em以上サポート"
    word_spacing: "0.16em以上サポート"

  reading_direction:
    ltr: "英語、日本語"
    rtl: "アラビア語、ヘブライ語（将来対応）"
    implementation: "Directionality widget"
```

#### 6.4.3 フォームアクセシビリティ

```yaml
form_accessibility:
  labels:
    every_input: "可視ラベルまたはaria-label"
    associated: "labelとinputの紐付け"

  error_handling:
    identification: "エラーのある入力を特定"
    description: "エラー内容を説明"
    suggestion: "修正方法を提案"
    timing: "送信後またはフォーカスアウト時"

  required_fields:
    indication: "「*」 + 凡例で説明"
    screen_reader: "required属性"

  input_assistance:
    autocomplete: "適切なautocomplete属性"
    input_mode: "適切なinputMode（numeric、email等）"
    validation: "リアルタイムフィードバック"
```

---

## 7. インタラクションパターン

### 7.1 ジェスチャー仕様

#### 7.1.1 標準ジェスチャー

```yaml
standard_gestures:
  tap:
    description: "1本指でのタップ"
    usage:
      - "ボタンのアクティベート"
      - "リストアイテムの選択"
      - "リンクのフォロー"
    feedback: "リップルエフェクト（Material）"

  double_tap:
    description: "素早い2回タップ"
    usage:
      - "画像のズーム"
      - "いいね（SNS風）"
      - "テキスト選択"
    note: "シングルタップとの区別のため、300ms遅延"

  long_press:
    description: "500ms以上の長押し"
    usage:
      - "コンテキストメニューの表示"
      - "ドラッグモードの開始"
      - "複数選択モードの開始"
    feedback: "ハプティックフィードバック"

  swipe:
    description: "画面上を一方向にスワイプ"
    variants:
      horizontal:
        left: "次のアイテム、削除（リスト）"
        right: "前のアイテム、アーカイブ（リスト）"
      vertical:
        up: "コンテンツの更新（pull-to-refresh逆）"
        down: "プルトゥリフレッシュ"
    threshold: "50dp以上の移動"

  pan:
    description: "押しながら移動"
    usage:
      - "マップの移動"
      - "カルーセルのスクロール"
      - "ボトムシートのドラッグ"

  pinch:
    description: "2本指でのピンチ操作"
    usage:
      - "ズームイン/アウト"
      - "マップの縮尺変更"
    sensitivity: "線形スケーリング"

  rotate:
    description: "2本指での回転"
    usage:
      - "マップの回転（オプション）"
    note: "限定的な使用、明示的な有効化"
```

#### 7.1.2 カスタムジェスチャー

```yaml
custom_gestures:
  pull_to_refresh:
    trigger: "画面上部からの下スワイプ"
    indicator: "CircularProgressIndicator"
    threshold: "70dp"
    feedback:
      - "インジケーター表示"
      - "ハプティックフィードバック"
      - "完了時のスナックバー"

  swipe_to_dismiss:
    trigger: "リストアイテムの水平スワイプ"
    variants:
      left_swipe:
        action: "削除"
        background: "error color"
        icon: "delete"
      right_swipe:
        action: "アーカイブ / お気に入り"
        background: "success color"
        icon: "archive / favorite"
    threshold: "アイテム幅の30%"
    undo: "スナックバーで元に戻すオプション提供"

  edge_swipe:
    trigger: "画面端からの内向きスワイプ"
    left_edge: "戻るナビゲーション（iOS）"
    right_edge: "未使用"
    note: "システムジェスチャーとの競合に注意"

  bottom_sheet_drag:
    trigger: "ボトムシートのハンドルドラッグ"
    snap_points: ["25%", "50%", "90%"]
    velocity_dismiss: "高速下方向スワイプで閉じる"
```

### 7.2 マイクロインタラクション

#### 7.2.1 フィードバックアニメーション

```yaml
feedback_animations:
  button_press:
    type: "ripple + scale"
    ripple:
      color: "rgba(0,0,0,0.12) or theme appropriate"
      duration: 200
      expand_from: "tap position"
    scale:
      from: 1.0
      to: 0.95
      duration: 100
      easing: "sharp"

  toggle_switch:
    type: "slide + color change"
    thumb_slide:
      duration: 200
      easing: "standard"
    track_color:
      duration: 150
      easing: "standard"

  checkbox_check:
    type: "scale + draw"
    container:
      scale: "1.0 -> 1.1 -> 1.0"
      duration: 200
    checkmark:
      draw: "path animation"
      duration: 150
      easing: "decelerate"

  favorite_heart:
    type: "scale + burst"
    heart:
      scale: "1.0 -> 1.3 -> 1.0"
      duration: 300
      easing: "spring"
    burst:
      particles: 6
      duration: 400
      color: "triptrip_red"

  add_to_cart:
    type: "fly + bounce"
    item_fly:
      from: "button position"
      to: "cart icon"
      duration: 400
      easing: "decelerate"
    cart_bounce:
      scale: "1.0 -> 1.2 -> 1.0"
      duration: 200
    badge_increment:
      fade_scale: "new number appears"
```

#### 7.2.2 状態遷移アニメーション

```yaml
state_transitions:
  loading_to_content:
    skeleton:
      fade_out: 200
    content:
      fade_in: 200
      stagger: 50  # 複数アイテムの場合
    alternative: "shimmer effect"

  content_to_empty:
    content:
      fade_out: 150
    empty_state:
      fade_in: 200
      scale_in: "0.95 -> 1.0"

  expand_collapse:
    height_animation:
      duration: 200
      easing: "standard"
    content_fade:
      duration: 150
    icon_rotation:
      degrees: 180
      duration: 200

  tab_switch:
    outgoing:
      fade_out: 100
      slide_out: "direction based"
    incoming:
      fade_in: 200
      slide_in: "opposite direction"
```

### 7.3 ローディングステート

#### 7.3.1 ローディングインジケーター

```yaml
loading_indicators:
  circular_spinner:
    usage: "一般的なローディング"
    size:
      small: 24
      medium: 40
      large: 64
    placement:
      button: "ラベルを置き換え"
      screen: "中央配置"
      inline: "コンテンツ横"

  skeleton_screen:
    usage: "コンテンツ構造が既知の場合"
    components:
      text_line:
        height: "line height"
        width: "60-90% random"
        radius: 4
      image:
        aspect_ratio: "actual image ratio"
        radius: "actual image radius"
      avatar:
        shape: "circle"
    animation:
      type: "shimmer"
      duration: 1500
      direction: "left to right"
      gradient: "gray_200 -> gray_100 -> gray_200"

  progress_indicator:
    usage: "進捗が測定可能な場合"
    type: "linear or circular"
    show_percentage: true
    message: "処理内容を表示"

  pull_to_refresh:
    indicator: "circular_spinner"
    position: "top of scrollable"
    activation_distance: 70
```

#### 7.3.2 ローディング状態のベストプラクティス

```yaml
loading_best_practices:
  timing:
    instant: "0-100ms: フィードバック不要"
    quick: "100-1000ms: スピナー表示"
    long: "1000ms+: プログレス + 説明文"

  perceived_performance:
    optimistic_ui:
      description: "成功を仮定して即座にUI更新"
      rollback: "エラー時に元に戻す"
      usage: "いいね、お気に入り追加"

    skeleton_screens:
      description: "コンテンツの構造を先行表示"
      benefit: "待ち時間を短く感じさせる"
      usage: "リスト、カード、詳細画面"

    progressive_loading:
      description: "重要度順にコンテンツを表示"
      order:
        1: "レイアウト構造"
        2: "テキストコンテンツ"
        3: "低解像度画像"
        4: "高解像度画像"

  user_feedback:
    show_progress: "可能な限り進捗を表示"
    explain_delay: "長時間の場合は理由を説明"
    allow_cancel: "中断可能な操作は中断ボタンを提供"
```

### 7.4 エラーハンドリングUI

#### 7.4.1 エラータイプ別表示

```yaml
error_types:
  validation_error:
    display: "インラインエラー"
    position: "入力フィールド下"
    styling:
      color: "error"
      icon: "error_outline"
      text: "body_small"
    example:
      message: "有効なメールアドレスを入力してください"

  field_error:
    display: "フィールドハイライト + メッセージ"
    styling:
      border: "2px error"
      background: "error_container (subtle)"
      helper_text: "error message"
    accessibility: "aria-invalid, aria-describedby"

  form_error:
    display: "フォーム上部のサマリー"
    styling:
      container: "error_container background"
      icon: "error"
      list: "エラー箇所へのリンク"
    behavior: "エラー箇所にスクロール"

  network_error:
    display: "フルスクリーン or ボトムシート"
    content:
      illustration: "接続エラーイラスト"
      title: "接続できませんでした"
      message: "ネットワーク接続を確認してください"
      action: "再試行"
    auto_retry: "指数バックオフで自動リトライ"

  server_error:
    display: "フルスクリーン or ダイアログ"
    content:
      illustration: "サーバーエラーイラスト"
      title: "問題が発生しました"
      message: "しばらくしてからもう一度お試しください"
      action: "再試行 / ホームに戻る"
    logging: "エラー詳細をサーバーに送信"

  not_found:
    display: "フルスクリーン"
    content:
      illustration: "404イラスト"
      title: "ページが見つかりません"
      message: "お探しのページは存在しないか、移動した可能性があります"
      action: "ホームに戻る"
```

#### 7.4.2 エラーリカバリーパターン

```yaml
error_recovery:
  retry:
    automatic:
      conditions: "ネットワークエラー、一時的なサーバーエラー"
      strategy: "exponential backoff (1s, 2s, 4s, 8s)"
      max_attempts: 3
      user_notification: "リトライ中..."
    manual:
      button: "再試行"
      placement: "エラーメッセージ近く"

  offline_queue:
    description: "オフライン時のアクションをキューに保存"
    actions:
      - "お気に入り追加/削除"
      - "レビュー投稿"
      - "プロフィール更新"
    sync: "オンライン復帰時に自動同期"
    conflict_resolution: "サーバー優先 or ユーザー確認"

  graceful_degradation:
    description: "一部機能が利用不可でも継続使用可能"
    examples:
      - "画像読み込み失敗時のプレースホルダー表示"
      - "推薦機能エラー時の人気ランキング表示"
      - "リアルタイム更新失敗時の最終取得データ表示"

  user_guidance:
    what_happened: "何が起きたかを説明"
    why: "なぜ起きたか（ユーザーが理解できる範囲で）"
    what_to_do: "ユーザーができることを提示"
    contact_support: "解決しない場合のサポート連絡先"
```

### 7.5 空ステート

```yaml
empty_states:
  design_principles:
    - "ユーザーを責めない"
    - "次のアクションを明確に提示"
    - "ブランドの温かみを表現"
    - "イラストで視覚的に伝える"

  components:
    illustration:
      size: "120-160dp"
      style: "ブランドイラストスタイル準拠"
    title:
      style: "headline_small"
      tone: "ポジティブ"
    message:
      style: "body_medium"
      color: "gray_600"
      max_lines: 2
    action:
      type: "primary_button or text_button"
      label: "具体的なアクション"

  examples:
    search_no_results:
      illustration: "虫眼鏡キャラクター"
      title: "見つかりませんでした"
      message: "別のキーワードで検索してみてください"
      action: "フィルターをクリア"

    cart_empty:
      illustration: "空のバッグキャラクター"
      title: "カートは空です"
      message: "気になる商品を追加してみましょう"
      action: "商品を探す"

    favorites_empty:
      illustration: "ハートキャラクター"
      title: "お気に入りはまだありません"
      message: "気になる体験を保存して、あとで見返しましょう"
      action: "体験を探す"

    orders_empty:
      illustration: "荷物キャラクター"
      title: "注文履歴はありません"
      message: "最初の予約をしてみませんか？"
      action: "体験を探す"

    notifications_empty:
      illustration: "ベルキャラクター"
      title: "お知らせはありません"
      message: "新しいお知らせがあるとここに表示されます"
      action: null  # アクションなし

    offline:
      illustration: "雲とWi-Fiキャラクター"
      title: "オフラインです"
      message: "インターネット接続を確認してください"
      action: "再試行"
```

---

## 8. プラットフォーム別実装ガイド

### 8.1 Flutter/Dart実装パターン

#### 8.1.1 デザイントークンの実装

```dart
// lib/design_system/tokens/design_tokens.dart

/// TripTripデザイントークンクラス
class TripTripDesignTokens {
  // シングルトンパターン
  static final TripTripDesignTokens _instance = TripTripDesignTokens._internal();
  factory TripTripDesignTokens() => _instance;
  TripTripDesignTokens._internal();

  // Context経由でのアクセス
  static TripTripDesignTokens of(BuildContext context) {
    final brightness = Theme.of(context).brightness;
    return brightness == Brightness.dark
        ? TripTripDesignTokens._dark
        : TripTripDesignTokens._light;
  }

  static final _light = TripTripDesignTokens._internal();
  static final _dark = TripTripDesignTokens._internal();

  // カラートークン
  final colors = TripTripColors();

  // タイポグラフィトークン
  final typography = TripTripTypography();

  // スペーシングトークン
  final spacing = TripTripSpacing();

  // エレベーショントークン
  final elevation = TripTripElevation();

  // ボーダートークン
  final radius = TripTripRadius();

  // アニメーショントークン
  final animation = TripTripAnimation();
}

/// カラートークン
class TripTripColors {
  // プライマリ
  final Color primary = const Color(0xFFE63946);
  final Color onPrimary = const Color(0xFFFFFFFF);
  final Color primaryContainer = const Color(0xFFFFE5E7);
  final Color onPrimaryContainer = const Color(0xFF8B0000);

  // セカンダリ（ネイビー）
  final Color secondary = const Color(0xFF1D3557);
  final Color onSecondary = const Color(0xFFFFFFFF);
  final Color secondaryContainer = const Color(0xFFE8EEF4);
  final Color onSecondaryContainer = const Color(0xFF0D1B2A);

  // サーフェス
  final Color surface = const Color(0xFFFFFFFF);
  final Color onSurface = const Color(0xFF1D3557);
  final Color surfaceVariant = const Color(0xFFF5F5F5);
  final Color onSurfaceVariant = const Color(0xFF757575);

  // 背景
  final Color background = const Color(0xFFF8F9FA);
  final Color onBackground = const Color(0xFF1D3557);

  // セマンティック
  final Color success = const Color(0xFF2A9D8F);
  final Color onSuccess = const Color(0xFFFFFFFF);
  final Color warning = const Color(0xFFF4A261);
  final Color onWarning = const Color(0xFF1D3557);
  final Color error = const Color(0xFFE63946);
  final Color onError = const Color(0xFFFFFFFF);
  final Color info = const Color(0xFF457B9D);
  final Color onInfo = const Color(0xFFFFFFFF);

  // アウトライン
  final Color outline = const Color(0xFFE0E0E0);
  final Color outlineVariant = const Color(0xFFEEEEEE);
}

/// タイポグラフィトークン
class TripTripTypography {
  // Display
  TextStyle get displayLarge => const TextStyle(
    fontFamily: 'Noto Sans JP',
    fontSize: 57,
    fontWeight: FontWeight.w400,
    height: 64 / 57,
    letterSpacing: -0.25,
  );

  // Headline
  TextStyle get headlineLarge => const TextStyle(
    fontFamily: 'Noto Sans JP',
    fontSize: 32,
    fontWeight: FontWeight.w700,
    height: 40 / 32,
  );

  TextStyle get headlineMedium => const TextStyle(
    fontFamily: 'Noto Sans JP',
    fontSize: 28,
    fontWeight: FontWeight.w600,
    height: 36 / 28,
  );

  TextStyle get headlineSmall => const TextStyle(
    fontFamily: 'Noto Sans JP',
    fontSize: 24,
    fontWeight: FontWeight.w600,
    height: 32 / 24,
  );

  // Title
  TextStyle get titleLarge => const TextStyle(
    fontFamily: 'Noto Sans JP',
    fontSize: 22,
    fontWeight: FontWeight.w500,
    height: 28 / 22,
  );

  TextStyle get titleMedium => const TextStyle(
    fontFamily: 'Noto Sans JP',
    fontSize: 16,
    fontWeight: FontWeight.w500,
    height: 24 / 16,
    letterSpacing: 0.15,
  );

  // Body
  TextStyle get bodyLarge => const TextStyle(
    fontFamily: 'Noto Sans JP',
    fontSize: 16,
    fontWeight: FontWeight.w400,
    height: 24 / 16,
    letterSpacing: 0.5,
  );

  TextStyle get bodyMedium => const TextStyle(
    fontFamily: 'Noto Sans JP',
    fontSize: 14,
    fontWeight: FontWeight.w400,
    height: 20 / 14,
    letterSpacing: 0.25,
  );

  // Label
  TextStyle get labelLarge => const TextStyle(
    fontFamily: 'Noto Sans JP',
    fontSize: 14,
    fontWeight: FontWeight.w500,
    height: 20 / 14,
    letterSpacing: 0.1,
  );

  // Button specific
  TextStyle get buttonMedium => const TextStyle(
    fontFamily: 'Noto Sans JP',
    fontSize: 14,
    fontWeight: FontWeight.w500,
    height: 20 / 14,
    letterSpacing: 0.1,
  );
}

/// スペーシングトークン
class TripTripSpacing {
  final double xxs = 2;
  final double xs = 4;
  final double sm = 8;
  final double md = 12;
  final double base = 16;
  final double lg = 24;
  final double xl = 32;
  final double xxl = 48;
  final double xxxl = 64;
}

/// ボーダー半径トークン
class TripTripRadius {
  final double none = 0;
  final double xs = 2;
  final double sm = 4;
  final double md = 8;
  final double lg = 12;
  final double xl = 16;
  final double xxl = 24;
  final double button = 8;
  final double card = 12;
  final double dialog = 28;
  final double full = 9999;
}

/// エレベーショントークン
class TripTripElevation {
  final double level0 = 0;
  final double level1 = 1;
  final double level2 = 2;
  final double level3 = 4;
  final double level4 = 8;
  final double level5 = 16;

  final double button = 1;
  final double card = 1;
  final double dialog = 8;
  final double fab = 4;
}

/// アニメーショントークン
class TripTripAnimation {
  // Duration
  final Duration fast = const Duration(milliseconds: 100);
  final Duration normal = const Duration(milliseconds: 200);
  final Duration slow = const Duration(milliseconds: 300);
  final Duration slower = const Duration(milliseconds: 400);

  // Curves
  final Curve standard = Curves.easeInOut;
  final Curve decelerate = Curves.decelerate;
  final Curve accelerate = Curves.easeIn;
  final Curve emphasized = Curves.easeOutCubic;
}
```

#### 8.1.2 テーマ設定

```dart
// lib/design_system/theme/triptrip_theme.dart

class TripTripTheme {
  static ThemeData light() {
    final tokens = TripTripDesignTokens();

    return ThemeData(
      useMaterial3: true,
      brightness: Brightness.light,

      // Color Scheme
      colorScheme: ColorScheme(
        brightness: Brightness.light,
        primary: tokens.colors.primary,
        onPrimary: tokens.colors.onPrimary,
        primaryContainer: tokens.colors.primaryContainer,
        onPrimaryContainer: tokens.colors.onPrimaryContainer,
        secondary: tokens.colors.secondary,
        onSecondary: tokens.colors.onSecondary,
        secondaryContainer: tokens.colors.secondaryContainer,
        onSecondaryContainer: tokens.colors.onSecondaryContainer,
        surface: tokens.colors.surface,
        onSurface: tokens.colors.onSurface,
        error: tokens.colors.error,
        onError: tokens.colors.onError,
      ),

      // Typography
      textTheme: TextTheme(
        displayLarge: tokens.typography.displayLarge,
        headlineLarge: tokens.typography.headlineLarge,
        headlineMedium: tokens.typography.headlineMedium,
        headlineSmall: tokens.typography.headlineSmall,
        titleLarge: tokens.typography.titleLarge,
        titleMedium: tokens.typography.titleMedium,
        bodyLarge: tokens.typography.bodyLarge,
        bodyMedium: tokens.typography.bodyMedium,
        labelLarge: tokens.typography.labelLarge,
      ),

      // Component themes
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          backgroundColor: tokens.colors.primary,
          foregroundColor: tokens.colors.onPrimary,
          elevation: tokens.elevation.button,
          padding: EdgeInsets.symmetric(
            horizontal: tokens.spacing.lg,
            vertical: tokens.spacing.md,
          ),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(tokens.radius.button),
          ),
          textStyle: tokens.typography.buttonMedium,
        ),
      ),

      cardTheme: CardTheme(
        elevation: tokens.elevation.card,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(tokens.radius.card),
        ),
        margin: EdgeInsets.all(tokens.spacing.sm),
      ),

      inputDecorationTheme: InputDecorationTheme(
        border: OutlineInputBorder(
          borderRadius: BorderRadius.circular(tokens.radius.sm),
          borderSide: BorderSide(color: tokens.colors.outline),
        ),
        focusedBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(tokens.radius.sm),
          borderSide: BorderSide(color: tokens.colors.primary, width: 2),
        ),
        errorBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(tokens.radius.sm),
          borderSide: BorderSide(color: tokens.colors.error, width: 2),
        ),
        contentPadding: EdgeInsets.all(tokens.spacing.base),
      ),

      bottomNavigationBarTheme: BottomNavigationBarThemeData(
        backgroundColor: tokens.colors.surface,
        selectedItemColor: tokens.colors.primary,
        unselectedItemColor: tokens.colors.onSurfaceVariant,
        elevation: tokens.elevation.level2,
        type: BottomNavigationBarType.fixed,
      ),

      snackBarTheme: SnackBarThemeData(
        behavior: SnackBarBehavior.floating,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(tokens.radius.sm),
        ),
      ),
    );
  }

  static ThemeData dark() {
    // ダークテーマの実装（省略）
    return ThemeData.dark();
  }
}
```

### 8.2 iOS Human Interface Guidelines対応

```yaml
ios_hig_compliance:
  navigation:
    back_button:
      style: "chevron.left + Previous screen title"
      gesture: "edge swipe to go back"
    large_title:
      when: "top of scroll view"
      collapse_on_scroll: true

  typography:
    system_fonts:
      fallback: "SF Pro (via system)"
      implementation: ".SF Pro in font family"

  touch_targets:
    minimum: 44  # points
    recommended: 48

  safe_areas:
    status_bar: "respect"
    home_indicator: "respect"
    dynamic_island: "respect"

  haptics:
    selection: "UISelectionFeedbackGenerator"
    impact: "UIImpactFeedbackGenerator"
    notification: "UINotificationFeedbackGenerator"
    usage:
      - "ボタンタップ: light impact"
      - "トグル切り替え: selection"
      - "成功/エラー: notification"

  platform_conventions:
    action_sheets: "bottom positioned"
    alerts: "center positioned"
    date_picker: "wheel style or inline"
    context_menu: "long press + haptic"
```

### 8.3 Android Material Design対応

```yaml
android_material_compliance:
  material_you:
    dynamic_color:
      enabled: true
      fallback: "TripTrip brand colors"
    implementation: "DynamicColorScheme from wallpaper"

  navigation:
    back_button: "system back button/gesture"
    predictive_back:
      enabled: true
      animation: "cross-fade preview"

  typography:
    system_fonts:
      primary: "Roboto (system default)"
      japanese: "Noto Sans JP"

  touch_targets:
    minimum: 48  # dp
    recommended: 56

  system_bars:
    status_bar:
      color: "transparent"
      icons: "dark on light, light on dark"
    navigation_bar:
      gesture_nav: "transparent"
      button_nav: "surface color"

  elevation:
    tonal_elevation: "Material 3 standard"
    shadow: "reduced in M3"

  components:
    top_app_bar: "Material 3 variants"
    navigation_bar: "Material 3 spec"
    fab: "Material 3 shape and elevation"
    cards: "filled, elevated, outlined variants"
```

### 8.4 Web実装考慮事項

```yaml
web_considerations:
  responsive_design:
    breakpoints:
      mobile: "< 600px"
      tablet: "600-1024px"
      desktop: "> 1024px"
    approach: "mobile-first"

  input_methods:
    mouse:
      hover_states: "required"
      cursor_types: "pointer for interactive"
    keyboard:
      focus_visible: "visible focus ring"
      tab_order: "logical"
      shortcuts: "common patterns"
    touch:
      same_as_mobile: "for touch-enabled devices"

  performance:
    initial_load:
      target: "< 3s on 3G"
      strategy: "lazy loading, code splitting"
    images:
      format: "WebP with fallback"
      lazy_loading: true
      responsive_sizes: true

  accessibility_web:
    aria: "full ARIA support"
    semantic_html: "proper heading hierarchy"
    skip_links: "skip to main content"
    landmarks: "main, nav, aside, footer"

  browser_support:
    modern: "Chrome, Firefox, Safari, Edge (latest 2 versions)"
    considerations:
      - "CSS custom properties"
      - "CSS Grid/Flexbox"
      - "ES6+ JavaScript"
```

---

## 9. デザインシステムガバナンス

### 9.1 バージョニング戦略

```yaml
versioning:
  scheme: "Semantic Versioning (SemVer)"
  format: "MAJOR.MINOR.PATCH"

  definitions:
    major:
      description: "破壊的変更"
      examples:
        - "コンポーネントAPIの変更"
        - "デザイントークンの削除・名称変更"
        - "必須プロパティの追加"
      migration: "マイグレーションガイド必須"

    minor:
      description: "後方互換性のある機能追加"
      examples:
        - "新規コンポーネントの追加"
        - "新規バリアントの追加"
        - "オプショナルプロパティの追加"
      migration: "不要"

    patch:
      description: "バグ修正、微調整"
      examples:
        - "スタイルの微調整"
        - "アクセシビリティ改善"
        - "ドキュメント修正"
      migration: "不要"

  release_cycle:
    major: "年1-2回"
    minor: "月1回"
    patch: "必要に応じて"

  deprecation_policy:
    notice: "1 major version前に非推奨化を予告"
    support: "非推奨後も1 major versionはサポート"
    removal: "2 major version後に削除可能"
```

### 9.2 コントリビューションガイドライン

```yaml
contribution_guidelines:
  process:
    1_proposal:
      description: "変更提案の作成"
      template: "RFC (Request for Comments) テンプレート使用"
      required:
        - "問題の説明"
        - "提案する解決策"
        - "影響範囲"
        - "代替案"

    2_review:
      description: "デザインチームによるレビュー"
      participants:
        - "デザインシステムリード"
        - "シニアデザイナー"
        - "フロントエンドリード"
      criteria:
        - "ブランドガイドライン準拠"
        - "アクセシビリティ要件"
        - "技術的実現性"
        - "既存システムとの整合性"

    3_implementation:
      description: "承認後の実装"
      deliverables:
        - "Figmaコンポーネント"
        - "Flutterコード"
        - "ドキュメント"
        - "テスト"

    4_release:
      description: "リリースとコミュニケーション"
      includes:
        - "Changelog更新"
        - "リリースノート"
        - "必要に応じてマイグレーションガイド"

  code_standards:
    flutter:
      linter: "flutter_lints"
      formatting: "dart format"
      naming: "lowerCamelCase for methods, UpperCamelCase for classes"
      documentation: "dartdoc comments"

    figma:
      naming: "Component/Variant/State"
      organization: "ページ別にカテゴリ分け"
      variants: "Component properties使用"
```

### 9.3 デザインレビュープロセス

```yaml
design_review:
  types:
    component_review:
      frequency: "新規コンポーネント作成時"
      scope:
        - "ビジュアルデザイン"
        - "インタラクション"
        - "アクセシビリティ"
        - "レスポンシブ対応"
      participants:
        - "デザイナー"
        - "デベロッパー"
        - "QA"

    pattern_review:
      frequency: "新規パターン導入時"
      scope:
        - "ユースケースの妥当性"
        - "既存パターンとの関係"
        - "一貫性"
      participants:
        - "デザインシステムリード"
        - "プロダクトデザイナー"

    audit:
      frequency: "四半期"
      scope:
        - "使用状況分析"
        - "非準拠の特定"
        - "改善機会の特定"
      deliverables:
        - "監査レポート"
        - "改善提案"

  checklist:
    visual:
      - "デザイントークン使用"
      - "カラーコントラスト"
      - "タイポグラフィスケール準拠"
      - "スペーシング一貫性"

    interaction:
      - "状態定義（hover, focus, active, disabled）"
      - "フィードバック（視覚、触覚）"
      - "アニメーション仕様"

    accessibility:
      - "WCAG 2.1 AA準拠"
      - "スクリーンリーダー対応"
      - "キーボード操作対応"

    responsive:
      - "モバイル最適化"
      - "タブレット対応"
      - "デスクトップ対応"

    documentation:
      - "使用方法"
      - "プロパティ説明"
      - "DO/DON'T例"
```

---

## 10. 文書間参照 & 統合ポイント

### 10.1 関連文書一覧

```yaml
related_documents:
  it_strategy:
    Doc-AD-001:
      title: "モバイルアプリケーションアーキテクチャ"
      relation: "アーキテクチャ設計との整合性"
      key_integrations:
        - "Widget構造の標準化"
        - "状態管理パターン"
        - "フィーチャーベースモジュール構成"

    Doc-AD-004:
      title: "ユーザーインターフェース & エクスペリエンスデザイン"
      relation: "上位のUX戦略文書"
      key_integrations:
        - "デザイン原則の適用"
        - "UXパターンの実装"
        - "ユーザビリティ指標"

    Doc-SP-001:
      title: "API仕様書"
      relation: "データ取得とUI表示の連携"
      key_integrations:
        - "ローディング状態の設計"
        - "エラーハンドリングUI"
        - "ページネーション対応"

    Doc-SP-002:
      title: "データベーススキーマ仕様"
      relation: "データモデルとUI表示の対応"
      key_integrations:
        - "エンティティのUI表現"
        - "ステータス表示"

  business_strategy:
    Doc-MS-001:
      title: "ブランド戦略 & ポジショニング"
      relation: "ブランドアイデンティティの反映"
      key_integrations:
        - "カラーパレット"
        - "タイポグラフィ"
        - "トーン & ボイス"
        - "ビジュアルスタイル"

    Doc-BM-003:
      title: "顧客価値提案"
      relation: "UX設計の方向性"
      key_integrations:
        - "価値提案のUI表現"
        - "差別化ポイントの強調"

  app_context:
    EXISTING_APP_ANALYSIS:
      title: "既存アプリケーション分析"
      relation: "現状のUI/UX評価"
      key_integrations:
        - "既存コンポーネントの改善"
        - "技術的負債の解消"
        - "新機能との整合性"
```

### 10.2 統合ポイントマトリクス

```yaml
integration_matrix:
  design_to_development:
    figma_to_flutter:
      design_tokens: "JSON/YAML export → Dart constants"
      components: "Figma specs → Flutter widgets"
      icons: "SVG export → flutter_svg"
      images: "Asset optimization → cached_network_image"

    handoff_process:
      - "Figmaでコンポーネント仕様確定"
      - "デザイントークンJSON生成"
      - "Flutter実装"
      - "Storybook登録"
      - "ドキュメント更新"

  brand_to_design:
    color_system:
      source: "Doc-MS-001 ビジュアルアイデンティティ"
      implementation: "TripTripColors class"

    typography:
      source: "Doc-MS-001 タイポグラフィ"
      implementation: "TripTripTypography class"

    iconography:
      source: "ブランドガイドライン"
      implementation: "Material Symbols + カスタムアイコン"

  accessibility_integration:
    wcag_requirements:
      source: "WCAG 2.1 Guidelines"
      implementation: "本文書 Section 6"

    testing:
      automated: "flutter_test, accessibility_tools"
      manual: "スクリーンリーダーテスト"
      audit: "四半期レビュー"

  cross_platform:
    ios:
      reference: "Apple Human Interface Guidelines"
      adaptations: "本文書 Section 8.2"

    android:
      reference: "Material Design 3 Guidelines"
      adaptations: "本文書 Section 8.3"

    web:
      reference: "Web Content Accessibility Guidelines"
      adaptations: "本文書 Section 8.4"
```

### 10.3 変更管理プロセス

```yaml
change_management:
  impact_assessment:
    design_token_change:
      affected:
        - "全コンポーネント"
        - "テーマ設定"
        - "ダークモード"
      process:
        - "影響範囲の特定"
        - "テスト計画作成"
        - "段階的ロールアウト"

    component_change:
      affected:
        - "使用箇所の特定"
        - "依存コンポーネント"
      process:
        - "非推奨化（必要に応じて）"
        - "マイグレーションガイド作成"
        - "段階的移行"

  communication:
    channels:
      - "Changelog"
      - "リリースノート"
      - "Slackチャンネル（#design-system）"
      - "週次デザイン同期会議"

    templates:
      breaking_change: |
        ## Breaking Change: [Component/Token Name]

        ### What Changed
        [変更内容の説明]

        ### Migration Guide
        [移行方法の説明]

        ### Before/After
        [コード例]

      new_feature: |
        ## New: [Feature Name]

        ### Overview
        [機能概要]

        ### Usage
        [使用方法]

        ### Examples
        [実装例]
```

---

## 結論

本文書は、TripTripプラットフォームのUI/UXデザインシステムの包括的な技術仕様を定義しました。

### 主要な成果物

1. **デザイントークン**: カラー、タイポグラフィ、スペーシング、エレベーション、アニメーションの完全な定義
2. **コンポーネントライブラリ**: 基礎コンポーネントから複合コンポーネントまでの詳細仕様
3. **アクセシビリティ仕様**: WCAG 2.1 AA準拠のための具体的な実装ガイドライン
4. **プラットフォーム対応**: Flutter/Dart、iOS、Android、Webの各プラットフォーム向け実装ガイド
5. **ガバナンス**: バージョニング、コントリビューション、レビュープロセスの標準化

### 次のステップ

1. **Flutterコンポーネントライブラリの実装**: 本仕様に基づくWidgetライブラリの構築
2. **Figmaコンポーネントライブラリの整備**: デザインツールとの同期
3. **自動テストスイートの構築**: ビジュアルリグレッションテスト、アクセシビリティテスト
4. **ドキュメントサイトの構築**: Storybookベースのコンポーネントカタログ

---

**文書情報**
- 文書ID: Doc-SP-003
- タイトル: UI/UXデザインシステム仕様書
- バージョン: 1.0.0
- ステータス: 完成
- 作成日: 2026-01-21
- 行数: 約2,000行

---

**文書間リンク**
- 前文書: [Doc-SP-002 データベーススキーマ仕様](./Doc-SP-002-database-schema.md)
- 次文書: [Doc-SP-004 テストケース仕様](./test-case-specification.md)
- 関連: [Doc-AD-004 ユーザーインターフェース & エクスペリエンスデザイン](../03-application-design/Doc-AD-004.md)
- 関連: [Doc-MS-001 ブランド戦略 & ポジショニング](../../business-strategy/06-marketing-strategy/Doc-MS-001.md)


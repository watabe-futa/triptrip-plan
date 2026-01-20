# Doc-AD-004: TripTrip ユーザーインターフェース＆エクスペリエンスデザイン

## エグゼクティブサマリー

本文書は、TripTripプラットフォームにおけるユーザーインターフェース（UI）およびユーザーエクスペリエンス（UX）デザインの包括的な戦略と実装ガイドラインを定義します。Flutter/Dartベースのモバイルアプリケーションを中心に、Google Material Design 3、Apple Human Interface Guidelines、WCAG 2.1 AAアクセシビリティ基準を統合したデザインシステムを構築します。Airbnb、Booking.com、Uberなどの世界トップレベルの旅行・モビリティアプリのUXベストプラクティスを参考に、日本市場および国際市場の両方に対応した、直感的で美しく、アクセシブルなユーザー体験を実現します。

---

## 第1章：はじめに＆コンテキスト

### 1.1 文書の目的

#### 1.1.1 戦略的目標

```yaml
design_objectives:
  primary_goals:
    - name: "一貫したブランド体験"
      description: "全タッチポイントで統一されたTripTripブランド体験の提供"
      metrics:
        - "ブランド認知度"
        - "デザイン一貫性スコア"
        - "ユーザー信頼度"

    - name: "直感的なユーザビリティ"
      description: "最小限の学習コストで利用可能なインターフェース"
      metrics:
        - "タスク完了率"
        - "エラー率"
        - "SUS (System Usability Scale) スコア"

    - name: "アクセシビリティの確保"
      description: "すべてのユーザーが利用可能なインクルーシブデザイン"
      metrics:
        - "WCAG 2.1 AA準拠率"
        - "支援技術互換性"
        - "アクセシビリティ監査スコア"

    - name: "パフォーマンス最適化"
      description: "高速で滑らかなユーザー体験の実現"
      metrics:
        - "First Contentful Paint"
        - "フレームレート（60fps維持率）"
        - "インタラクション遅延"

  success_criteria:
    usability:
      - "SUSスコア: 80以上"
      - "タスク完了率: 95%以上"
      - "エラー率: 5%未満"
    satisfaction:
      - "NPS: 50以上"
      - "App Store評価: 4.5以上"
      - "ユーザー継続率: 40%以上"
```

#### 1.1.2 デザインスコープ

```
デザインシステムスコープ:
├── プラットフォーム
│   ├── iOS（iPhone、iPad）
│   ├── Android（スマートフォン、タブレット）
│   ├── Web（レスポンシブ）
│   └── Wearable（Apple Watch、Wear OS）
├── コンポーネント
│   ├── 基本UI要素
│   ├── 複合コンポーネント
│   ├── パターンライブラリ
│   └── アニメーション・トランジション
├── ドキュメント
│   ├── デザイントークン仕様
│   ├── コンポーネントガイドライン
│   ├── UXパターン集
│   └── アクセシビリティガイド
└── ツール
    ├── Figmaデザインシステム
    ├── Flutterコンポーネントライブラリ
    ├── Storybookカタログ
    └── 自動テストスイート
```

### 1.2 既存実装の分析

#### 1.2.1 現行デザインシステムの評価

```yaml
current_state_analysis:
  strengths:
    - "Material Design 3ベースの基盤"
    - "Flutter Screenutilによるレスポンシブ対応"
    - "ColorScheme seed値によるテーマ管理"
    - "フィーチャーベースのWidget構造"
    - "再利用可能なカスタムコンポーネント"

  implemented_components:
    navigation:
      - "BottomNavigationWidget（5タブ構成）"
      - "AppBarカスタマイズ"
      - "Drawer（未使用）"
    product_display:
      - "ProductCardWidget"
      - "ProductGridWidget"
      - "CategorySelectorWidget"
    hotel_components:
      - "HotelCardWidget"
      - "HotelDetailWidget"
      - "HotelSearchWidget"
    ticket_components:
      - "TicketCardWidget"
      - "QRDisplayWidget"
    cart_components:
      - "CartItemWidget"
      - "CartSummaryWidget"
    common:
      - "LoadingIndicator"
      - "ErrorWidget"
      - "EmptyStateWidget"

  gaps_identified:
    - "統一されたデザイントークンシステムの欠如"
    - "アクセシビリティ対応の不完全性"
    - "ダークモード未実装"
    - "アニメーションガイドラインの欠如"
    - "多言語タイポグラフィの最適化不足"
    - "ユーザーテスト結果の未反映"

  technical_debt:
    - "/lib/screens/のレガシー画面"
    - "一貫性のないスタイリングアプローチ"
    - "ハードコードされたカラー値"
    - "非標準のスペーシング"
```

### 1.3 競合分析＆ベンチマーク

#### 1.3.1 主要競合アプリのUX分析

```yaml
competitive_analysis:
  airbnb:
    strengths:
      - "感情的なつながりを重視したビジュアル"
      - "写真中心のブラウジング体験"
      - "シームレスな予約フロー"
      - "パーソナライズされたホーム画面"
    key_patterns:
      - "フルスクリーン画像カルーセル"
      - "スティッキーCTAボタン"
      - "段階的情報開示"
    metrics:
      app_store_rating: 4.8
      design_awards: "多数受賞"

  booking_com:
    strengths:
      - "情報密度と可読性のバランス"
      - "強力なフィルタリング機能"
      - "価格透明性"
      - "緊急性の演出（残り〇室）"
    key_patterns:
      - "スクロール連動ヘッダー"
      - "マップ連携検索"
      - "比較機能"
    metrics:
      app_store_rating: 4.7
      conversion_rate: "業界トップ"

  google_maps:
    strengths:
      - "直感的なマップインタラクション"
      - "音声検索・ナビゲーション"
      - "オフライン機能"
      - "AR連携（Live View）"
    key_patterns:
      - "ボトムシート情報表示"
      - "ジェスチャーベースの操作"
      - "コンテキスト適応UI"
    metrics:
      app_store_rating: 4.7
      daily_active_users: "10億+"

  japan_specific_apps:
    jalan:
      strengths:
        - "日本の旅行文化に適合"
        - "クーポン・ポイント統合"
        - "温泉・旅館特化機能"
    rakuten_travel:
      strengths:
        - "楽天エコシステム連携"
        - "豊富なレビュー"
        - "ポイント還元表示"
```

---

## 第2章：デザインシステム

### 2.1 デザイン原則

#### 2.1.1 コアデザイン原則

```yaml
design_principles:
  principle_1:
    name: "明確性（Clarity）"
    description: "情報は明確で理解しやすく、ユーザーの意思決定を支援"
    guidelines:
      - "重要な情報を視覚的に強調"
      - "専門用語を避け、平易な言葉を使用"
      - "アイコンとラベルの併用"
      - "エラーメッセージは原因と解決策を提示"
    examples:
      do:
        - "「予約を確定」のような明確なCTA"
        - "価格は税込み総額を目立つ位置に"
      dont:
        - "「処理」のような曖昧なボタンラベル"
        - "重要情報の小さいフォントサイズ"

  principle_2:
    name: "効率性（Efficiency）"
    description: "最小限のステップで目標を達成できるフロー設計"
    guidelines:
      - "タスク完了までのステップ数を最小化"
      - "よく使う機能へのクイックアクセス"
      - "スマートデフォルト値の設定"
      - "入力の自動補完と提案"
    examples:
      do:
        - "以前の検索条件の自動復元"
        - "ワンタップ再予約機能"
      dont:
        - "不要な確認ダイアログの連発"
        - "情報の過度な分散"

  principle_3:
    name: "一貫性（Consistency）"
    description: "全体を通じて統一されたパターンとインタラクション"
    guidelines:
      - "同じアクションには同じ見た目と動作"
      - "プラットフォーム慣習の尊重"
      - "ナビゲーションパターンの統一"
      - "用語の一貫した使用"
    examples:
      do:
        - "全画面で統一されたボトムナビゲーション"
        - "同じジェスチャーで同じ結果"
      dont:
        - "画面ごとに異なる戻るボタンの位置"
        - "同じ機能に異なるアイコン"

  principle_4:
    name: "フィードバック（Feedback）"
    description: "すべてのユーザーアクションに適切な応答を提供"
    guidelines:
      - "操作の即座の視覚的フィードバック"
      - "処理中の状態表示"
      - "成功・エラーの明確な通知"
      - "元に戻す機能の提供"
    examples:
      do:
        - "ボタンタップ時のリップルエフェクト"
        - "「カートに追加しました」のスナックバー"
      dont:
        - "クリックしても何も変化しないUI"
        - "サイレントな処理失敗"

  principle_5:
    name: "感動（Delight）"
    description: "機能性を超えた、心地よく記憶に残る体験"
    guidelines:
      - "意味のあるマイクロインタラクション"
      - "美しいビジュアルとトランジション"
      - "パーソナライズされた体験"
      - "期待を超えるサプライズ要素"
    examples:
      do:
        - "予約完了時の祝福アニメーション"
        - "目的地の美しい写真での歓迎"
      dont:
        - "過度なアニメーションによる遅延感"
        - "無機質なシステムメッセージ"
```

#### 2.1.2 日本市場向けデザイン考慮事項

```yaml
japan_specific_considerations:
  cultural_aspects:
    - consideration: "情報密度の許容度"
      description: "日本のユーザーは情報量の多いUIに慣れている"
      approach: "情報を詰め込みすぎず、段階的開示を採用"

    - consideration: "信頼性の表現"
      description: "レビュー、評価、認証マークの重要性"
      approach: "第三者評価を目立つ位置に配置"

    - consideration: "価格表示の慣習"
      description: "税込み/税別、ポイント還元の明示"
      approach: "総額表示を基本に、ポイント還元を付加情報で表示"

    - consideration: "季節性の重視"
      description: "四季や行事に合わせたビジュアル"
      approach: "季節ごとのテーマカラー・イラストの導入"

  language_considerations:
    - "日本語と英語の文字数差異への対応"
    - "縦書きオプションの検討（特定コンテンツ）"
    - "敬語レベルの使い分け"
    - "ローマ字・カタカナ・漢字の適切な使用"

  ux_patterns:
    - pattern: "お気に入り機能の重視"
      reason: "日本のユーザーは比較検討を好む"
    - pattern: "ポイント・クーポンの目立つ表示"
      reason: "お得感への高い関心"
    - pattern: "詳細なフィルタリング"
      reason: "具体的な条件での絞り込みニーズ"
```

### 2.2 コンポーネントライブラリ

#### 2.2.1 基本コンポーネント

```dart
// TripTrip デザインシステム - 基本コンポーネント

/// ========================================
/// ボタンコンポーネント
/// ========================================

/// プライマリボタン - 主要アクション用
class TripTripPrimaryButton extends StatelessWidget {
  final String label;
  final VoidCallback? onPressed;
  final bool isLoading;
  final IconData? leadingIcon;
  final TripTripButtonSize size;

  const TripTripPrimaryButton({
    Key? key,
    required this.label,
    this.onPressed,
    this.isLoading = false,
    this.leadingIcon,
    this.size = TripTripButtonSize.medium,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final tokens = TripTripDesignTokens.of(context);

    return SizedBox(
      height: _getHeight(),
      child: ElevatedButton(
        onPressed: isLoading ? null : onPressed,
        style: ElevatedButton.styleFrom(
          backgroundColor: tokens.colors.primary,
          foregroundColor: tokens.colors.onPrimary,
          disabledBackgroundColor: tokens.colors.primaryDisabled,
          elevation: tokens.elevation.button,
          padding: EdgeInsets.symmetric(
            horizontal: tokens.spacing.lg,
            vertical: tokens.spacing.md,
          ),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(tokens.radius.button),
          ),
        ),
        child: isLoading
            ? SizedBox(
                width: 20,
                height: 20,
                child: CircularProgressIndicator(
                  strokeWidth: 2,
                  valueColor: AlwaysStoppedAnimation<Color>(
                    tokens.colors.onPrimary,
                  ),
                ),
              )
            : Row(
                mainAxisSize: MainAxisSize.min,
                children: [
                  if (leadingIcon != null) ...[
                    Icon(leadingIcon, size: _getIconSize()),
                    SizedBox(width: tokens.spacing.sm),
                  ],
                  Text(
                    label,
                    style: _getTextStyle(tokens),
                  ),
                ],
              ),
      ),
    );
  }

  double _getHeight() {
    switch (size) {
      case TripTripButtonSize.small:
        return 36;
      case TripTripButtonSize.medium:
        return 48;
      case TripTripButtonSize.large:
        return 56;
    }
  }

  double _getIconSize() {
    switch (size) {
      case TripTripButtonSize.small:
        return 16;
      case TripTripButtonSize.medium:
        return 20;
      case TripTripButtonSize.large:
        return 24;
    }
  }

  TextStyle _getTextStyle(TripTripDesignTokens tokens) {
    switch (size) {
      case TripTripButtonSize.small:
        return tokens.typography.buttonSmall;
      case TripTripButtonSize.medium:
        return tokens.typography.buttonMedium;
      case TripTripButtonSize.large:
        return tokens.typography.buttonLarge;
    }
  }
}

enum TripTripButtonSize { small, medium, large }

/// セカンダリボタン - 補助アクション用
class TripTripSecondaryButton extends StatelessWidget {
  final String label;
  final VoidCallback? onPressed;
  final bool isLoading;
  final IconData? leadingIcon;

  const TripTripSecondaryButton({
    Key? key,
    required this.label,
    this.onPressed,
    this.isLoading = false,
    this.leadingIcon,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    final tokens = TripTripDesignTokens.of(context);

    return OutlinedButton(
      onPressed: isLoading ? null : onPressed,
      style: OutlinedButton.styleFrom(
        foregroundColor: tokens.colors.primary,
        side: BorderSide(
          color: tokens.colors.primary,
          width: 1.5,
        ),
        padding: EdgeInsets.symmetric(
          horizontal: tokens.spacing.lg,
          vertical: tokens.spacing.md,
        ),
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(tokens.radius.button),
        ),
      ),
      child: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          if (leadingIcon != null) ...[
            Icon(leadingIcon, size: 20),
            SizedBox(width: tokens.spacing.sm),
          ],
          Text(label, style: tokens.typography.buttonMedium),
        ],
      ),
    );
  }
}

/// テキストボタン - 軽微なアクション用
class TripTripTextButton extends StatelessWidget {
  final String label;
  final VoidCallback? onPressed;
  final IconData? leadingIcon;
  final IconData? trailingIcon;

  const TripTripTextButton({
    Key? key,
    required this.label,
    this.onPressed,
    this.leadingIcon,
    this.trailingIcon,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    final tokens = TripTripDesignTokens.of(context);

    return TextButton(
      onPressed: onPressed,
      style: TextButton.styleFrom(
        foregroundColor: tokens.colors.primary,
        padding: EdgeInsets.symmetric(
          horizontal: tokens.spacing.md,
          vertical: tokens.spacing.sm,
        ),
      ),
      child: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          if (leadingIcon != null) ...[
            Icon(leadingIcon, size: 18),
            SizedBox(width: tokens.spacing.xs),
          ],
          Text(label, style: tokens.typography.buttonSmall),
          if (trailingIcon != null) ...[
            SizedBox(width: tokens.spacing.xs),
            Icon(trailingIcon, size: 18),
          ],
        ],
      ),
    );
  }
}

/// ========================================
/// 入力コンポーネント
/// ========================================

/// テキスト入力フィールド
class TripTripTextField extends StatefulWidget {
  final String? label;
  final String? hint;
  final String? helperText;
  final String? errorText;
  final TextEditingController? controller;
  final bool obscureText;
  final TextInputType keyboardType;
  final IconData? prefixIcon;
  final Widget? suffixWidget;
  final ValueChanged<String>? onChanged;
  final FormFieldValidator<String>? validator;
  final bool enabled;
  final int? maxLines;
  final int? maxLength;

  const TripTripTextField({
    Key? key,
    this.label,
    this.hint,
    this.helperText,
    this.errorText,
    this.controller,
    this.obscureText = false,
    this.keyboardType = TextInputType.text,
    this.prefixIcon,
    this.suffixWidget,
    this.onChanged,
    this.validator,
    this.enabled = true,
    this.maxLines = 1,
    this.maxLength,
  }) : super(key: key);

  @override
  State<TripTripTextField> createState() => _TripTripTextFieldState();
}

class _TripTripTextFieldState extends State<TripTripTextField> {
  late FocusNode _focusNode;
  bool _isFocused = false;

  @override
  void initState() {
    super.initState();
    _focusNode = FocusNode();
    _focusNode.addListener(_handleFocusChange);
  }

  void _handleFocusChange() {
    setState(() {
      _isFocused = _focusNode.hasFocus;
    });
  }

  @override
  void dispose() {
    _focusNode.removeListener(_handleFocusChange);
    _focusNode.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final tokens = TripTripDesignTokens.of(context);
    final hasError = widget.errorText != null;

    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        if (widget.label != null) ...[
          Text(
            widget.label!,
            style: tokens.typography.labelMedium.copyWith(
              color: hasError
                  ? tokens.colors.error
                  : _isFocused
                      ? tokens.colors.primary
                      : tokens.colors.textSecondary,
            ),
          ),
          SizedBox(height: tokens.spacing.xs),
        ],
        TextFormField(
          controller: widget.controller,
          focusNode: _focusNode,
          obscureText: widget.obscureText,
          keyboardType: widget.keyboardType,
          enabled: widget.enabled,
          maxLines: widget.maxLines,
          maxLength: widget.maxLength,
          onChanged: widget.onChanged,
          validator: widget.validator,
          style: tokens.typography.bodyMedium,
          decoration: InputDecoration(
            hintText: widget.hint,
            hintStyle: tokens.typography.bodyMedium.copyWith(
              color: tokens.colors.textTertiary,
            ),
            prefixIcon: widget.prefixIcon != null
                ? Icon(
                    widget.prefixIcon,
                    color: hasError
                        ? tokens.colors.error
                        : _isFocused
                            ? tokens.colors.primary
                            : tokens.colors.textSecondary,
                  )
                : null,
            suffixIcon: widget.suffixWidget,
            filled: true,
            fillColor: widget.enabled
                ? tokens.colors.surface
                : tokens.colors.surfaceDisabled,
            border: OutlineInputBorder(
              borderRadius: BorderRadius.circular(tokens.radius.input),
              borderSide: BorderSide(color: tokens.colors.border),
            ),
            enabledBorder: OutlineInputBorder(
              borderRadius: BorderRadius.circular(tokens.radius.input),
              borderSide: BorderSide(
                color: hasError ? tokens.colors.error : tokens.colors.border,
              ),
            ),
            focusedBorder: OutlineInputBorder(
              borderRadius: BorderRadius.circular(tokens.radius.input),
              borderSide: BorderSide(
                color: hasError ? tokens.colors.error : tokens.colors.primary,
                width: 2,
              ),
            ),
            errorBorder: OutlineInputBorder(
              borderRadius: BorderRadius.circular(tokens.radius.input),
              borderSide: BorderSide(color: tokens.colors.error),
            ),
            contentPadding: EdgeInsets.symmetric(
              horizontal: tokens.spacing.md,
              vertical: tokens.spacing.md,
            ),
          ),
        ),
        if (widget.helperText != null || widget.errorText != null) ...[
          SizedBox(height: tokens.spacing.xs),
          Text(
            widget.errorText ?? widget.helperText!,
            style: tokens.typography.caption.copyWith(
              color: hasError
                  ? tokens.colors.error
                  : tokens.colors.textSecondary,
            ),
          ),
        ],
      ],
    );
  }
}

/// 検索入力フィールド
class TripTripSearchField extends StatelessWidget {
  final String hint;
  final TextEditingController? controller;
  final ValueChanged<String>? onChanged;
  final VoidCallback? onClear;
  final VoidCallback? onSubmitted;
  final bool autoFocus;

  const TripTripSearchField({
    Key? key,
    this.hint = '検索',
    this.controller,
    this.onChanged,
    this.onClear,
    this.onSubmitted,
    this.autoFocus = false,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    final tokens = TripTripDesignTokens.of(context);

    return TextField(
      controller: controller,
      autofocus: autoFocus,
      onChanged: onChanged,
      onSubmitted: (_) => onSubmitted?.call(),
      style: tokens.typography.bodyMedium,
      decoration: InputDecoration(
        hintText: hint,
        hintStyle: tokens.typography.bodyMedium.copyWith(
          color: tokens.colors.textTertiary,
        ),
        prefixIcon: Icon(
          Icons.search,
          color: tokens.colors.textSecondary,
        ),
        suffixIcon: controller?.text.isNotEmpty == true
            ? IconButton(
                icon: Icon(Icons.clear, color: tokens.colors.textSecondary),
                onPressed: () {
                  controller?.clear();
                  onClear?.call();
                },
              )
            : null,
        filled: true,
        fillColor: tokens.colors.surfaceVariant,
        border: OutlineInputBorder(
          borderRadius: BorderRadius.circular(tokens.radius.full),
          borderSide: BorderSide.none,
        ),
        contentPadding: EdgeInsets.symmetric(
          horizontal: tokens.spacing.md,
          vertical: tokens.spacing.sm,
        ),
      ),
    );
  }
}

/// ========================================
/// カードコンポーネント
/// ========================================

/// 汎用カードコンテナ
class TripTripCard extends StatelessWidget {
  final Widget child;
  final VoidCallback? onTap;
  final EdgeInsets? padding;
  final Color? backgroundColor;
  final double? elevation;
  final BorderRadius? borderRadius;

  const TripTripCard({
    Key? key,
    required this.child,
    this.onTap,
    this.padding,
    this.backgroundColor,
    this.elevation,
    this.borderRadius,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    final tokens = TripTripDesignTokens.of(context);

    return Material(
      color: backgroundColor ?? tokens.colors.surface,
      elevation: elevation ?? tokens.elevation.card,
      borderRadius: borderRadius ?? BorderRadius.circular(tokens.radius.card),
      clipBehavior: Clip.antiAlias,
      child: InkWell(
        onTap: onTap,
        child: Padding(
          padding: padding ?? EdgeInsets.all(tokens.spacing.md),
          child: child,
        ),
      ),
    );
  }
}

/// 商品カード
class TripTripProductCard extends StatelessWidget {
  final String imageUrl;
  final String title;
  final String? subtitle;
  final String price;
  final String? originalPrice;
  final double? rating;
  final int? reviewCount;
  final VoidCallback? onTap;
  final VoidCallback? onFavorite;
  final bool isFavorite;
  final List<String>? badges;

  const TripTripProductCard({
    Key? key,
    required this.imageUrl,
    required this.title,
    this.subtitle,
    required this.price,
    this.originalPrice,
    this.rating,
    this.reviewCount,
    this.onTap,
    this.onFavorite,
    this.isFavorite = false,
    this.badges,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    final tokens = TripTripDesignTokens.of(context);

    return TripTripCard(
      onTap: onTap,
      padding: EdgeInsets.zero,
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          // 画像エリア
          Stack(
            children: [
              AspectRatio(
                aspectRatio: 4 / 3,
                child: CachedNetworkImage(
                  imageUrl: imageUrl,
                  fit: BoxFit.cover,
                  placeholder: (context, url) => Container(
                    color: tokens.colors.surfaceVariant,
                    child: Center(
                      child: CircularProgressIndicator(
                        strokeWidth: 2,
                        color: tokens.colors.primary,
                      ),
                    ),
                  ),
                  errorWidget: (context, url, error) => Container(
                    color: tokens.colors.surfaceVariant,
                    child: Icon(
                      Icons.image_not_supported,
                      color: tokens.colors.textTertiary,
                    ),
                  ),
                ),
              ),
              // お気に入りボタン
              Positioned(
                top: tokens.spacing.sm,
                right: tokens.spacing.sm,
                child: _FavoriteButton(
                  isFavorite: isFavorite,
                  onTap: onFavorite,
                ),
              ),
              // バッジ
              if (badges != null && badges!.isNotEmpty)
                Positioned(
                  top: tokens.spacing.sm,
                  left: tokens.spacing.sm,
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: badges!
                        .map((badge) => Padding(
                              padding: EdgeInsets.only(
                                bottom: tokens.spacing.xs,
                              ),
                              child: _Badge(text: badge),
                            ))
                        .toList(),
                  ),
                ),
            ],
          ),
          // 情報エリア
          Padding(
            padding: EdgeInsets.all(tokens.spacing.md),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(
                  title,
                  style: tokens.typography.titleSmall,
                  maxLines: 2,
                  overflow: TextOverflow.ellipsis,
                ),
                if (subtitle != null) ...[
                  SizedBox(height: tokens.spacing.xs),
                  Text(
                    subtitle!,
                    style: tokens.typography.bodySmall.copyWith(
                      color: tokens.colors.textSecondary,
                    ),
                    maxLines: 1,
                    overflow: TextOverflow.ellipsis,
                  ),
                ],
                SizedBox(height: tokens.spacing.sm),
                // 評価
                if (rating != null)
                  Row(
                    children: [
                      Icon(
                        Icons.star,
                        size: 16,
                        color: tokens.colors.rating,
                      ),
                      SizedBox(width: tokens.spacing.xxs),
                      Text(
                        rating!.toStringAsFixed(1),
                        style: tokens.typography.labelSmall.copyWith(
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                      if (reviewCount != null) ...[
                        SizedBox(width: tokens.spacing.xs),
                        Text(
                          '($reviewCount件)',
                          style: tokens.typography.caption.copyWith(
                            color: tokens.colors.textTertiary,
                          ),
                        ),
                      ],
                    ],
                  ),
                SizedBox(height: tokens.spacing.sm),
                // 価格
                Row(
                  crossAxisAlignment: CrossAxisAlignment.baseline,
                  textBaseline: TextBaseline.alphabetic,
                  children: [
                    Text(
                      price,
                      style: tokens.typography.titleMedium.copyWith(
                        color: tokens.colors.primary,
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                    if (originalPrice != null) ...[
                      SizedBox(width: tokens.spacing.sm),
                      Text(
                        originalPrice!,
                        style: tokens.typography.bodySmall.copyWith(
                          color: tokens.colors.textTertiary,
                          decoration: TextDecoration.lineThrough,
                        ),
                      ),
                    ],
                  ],
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}

class _FavoriteButton extends StatelessWidget {
  final bool isFavorite;
  final VoidCallback? onTap;

  const _FavoriteButton({
    required this.isFavorite,
    this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    final tokens = TripTripDesignTokens.of(context);

    return GestureDetector(
      onTap: onTap,
      child: Container(
        width: 36,
        height: 36,
        decoration: BoxDecoration(
          color: tokens.colors.surface.withOpacity(0.9),
          shape: BoxShape.circle,
          boxShadow: [
            BoxShadow(
              color: Colors.black.withOpacity(0.1),
              blurRadius: 4,
              offset: const Offset(0, 2),
            ),
          ],
        ),
        child: Icon(
          isFavorite ? Icons.favorite : Icons.favorite_border,
          size: 20,
          color: isFavorite ? tokens.colors.error : tokens.colors.textSecondary,
        ),
      ),
    );
  }
}

class _Badge extends StatelessWidget {
  final String text;

  const _Badge({required this.text});

  @override
  Widget build(BuildContext context) {
    final tokens = TripTripDesignTokens.of(context);

    return Container(
      padding: EdgeInsets.symmetric(
        horizontal: tokens.spacing.sm,
        vertical: tokens.spacing.xxs,
      ),
      decoration: BoxDecoration(
        color: tokens.colors.primary,
        borderRadius: BorderRadius.circular(tokens.radius.badge),
      ),
      child: Text(
        text,
        style: tokens.typography.caption.copyWith(
          color: tokens.colors.onPrimary,
          fontWeight: FontWeight.bold,
        ),
      ),
    );
  }
}
```

#### 2.2.2 複合コンポーネント

```dart
/// ========================================
/// ナビゲーションコンポーネント
/// ========================================

/// ボトムナビゲーションバー
class TripTripBottomNavigation extends StatelessWidget {
  final int currentIndex;
  final ValueChanged<int> onTap;

  const TripTripBottomNavigation({
    Key? key,
    required this.currentIndex,
    required this.onTap,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    final tokens = TripTripDesignTokens.of(context);

    return Container(
      decoration: BoxDecoration(
        color: tokens.colors.surface,
        boxShadow: [
          BoxShadow(
            color: Colors.black.withOpacity(0.08),
            blurRadius: 8,
            offset: const Offset(0, -2),
          ),
        ],
      ),
      child: SafeArea(
        child: Padding(
          padding: EdgeInsets.symmetric(
            horizontal: tokens.spacing.md,
            vertical: tokens.spacing.sm,
          ),
          child: Row(
            mainAxisAlignment: MainAxisAlignment.spaceAround,
            children: [
              _NavItem(
                icon: Icons.home_outlined,
                activeIcon: Icons.home,
                label: 'ホーム',
                isSelected: currentIndex == 0,
                onTap: () => onTap(0),
              ),
              _NavItem(
                icon: Icons.search_outlined,
                activeIcon: Icons.search,
                label: '検索',
                isSelected: currentIndex == 1,
                onTap: () => onTap(1),
              ),
              _NavItem(
                icon: Icons.shopping_bag_outlined,
                activeIcon: Icons.shopping_bag,
                label: '商品',
                isSelected: currentIndex == 2,
                onTap: () => onTap(2),
              ),
              _NavItem(
                icon: Icons.shopping_cart_outlined,
                activeIcon: Icons.shopping_cart,
                label: 'カート',
                isSelected: currentIndex == 3,
                onTap: () => onTap(3),
                badge: _getCartBadge(context),
              ),
              _NavItem(
                icon: Icons.person_outline,
                activeIcon: Icons.person,
                label: 'マイページ',
                isSelected: currentIndex == 4,
                onTap: () => onTap(4),
              ),
            ],
          ),
        ),
      ),
    );
  }

  int? _getCartBadge(BuildContext context) {
    // カート内のアイテム数を取得
    // 実際の実装ではProviderから取得
    return null;
  }
}

class _NavItem extends StatelessWidget {
  final IconData icon;
  final IconData activeIcon;
  final String label;
  final bool isSelected;
  final VoidCallback onTap;
  final int? badge;

  const _NavItem({
    required this.icon,
    required this.activeIcon,
    required this.label,
    required this.isSelected,
    required this.onTap,
    this.badge,
  });

  @override
  Widget build(BuildContext context) {
    final tokens = TripTripDesignTokens.of(context);

    return GestureDetector(
      onTap: onTap,
      behavior: HitTestBehavior.opaque,
      child: Semantics(
        button: true,
        label: label,
        selected: isSelected,
        child: Container(
          padding: EdgeInsets.symmetric(
            horizontal: tokens.spacing.md,
            vertical: tokens.spacing.xs,
          ),
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Stack(
                clipBehavior: Clip.none,
                children: [
                  AnimatedSwitcher(
                    duration: tokens.animation.fast,
                    child: Icon(
                      isSelected ? activeIcon : icon,
                      key: ValueKey(isSelected),
                      size: 24,
                      color: isSelected
                          ? tokens.colors.primary
                          : tokens.colors.textSecondary,
                    ),
                  ),
                  if (badge != null && badge! > 0)
                    Positioned(
                      right: -8,
                      top: -4,
                      child: Container(
                        padding: const EdgeInsets.symmetric(
                          horizontal: 6,
                          vertical: 2,
                        ),
                        decoration: BoxDecoration(
                          color: tokens.colors.error,
                          borderRadius: BorderRadius.circular(10),
                        ),
                        constraints: const BoxConstraints(
                          minWidth: 18,
                          minHeight: 18,
                        ),
                        child: Text(
                          badge! > 99 ? '99+' : badge.toString(),
                          style: tokens.typography.caption.copyWith(
                            color: tokens.colors.onError,
                            fontWeight: FontWeight.bold,
                            fontSize: 10,
                          ),
                          textAlign: TextAlign.center,
                        ),
                      ),
                    ),
                ],
              ),
              SizedBox(height: tokens.spacing.xxs),
              Text(
                label,
                style: tokens.typography.caption.copyWith(
                  color: isSelected
                      ? tokens.colors.primary
                      : tokens.colors.textSecondary,
                  fontWeight: isSelected ? FontWeight.w600 : FontWeight.normal,
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}

/// ========================================
/// リスト・グリッドコンポーネント
/// ========================================

/// 商品グリッド
class TripTripProductGrid extends StatelessWidget {
  final List<ProductModel> products;
  final ScrollController? scrollController;
  final bool isLoading;
  final VoidCallback? onLoadMore;
  final ValueChanged<ProductModel>? onProductTap;
  final ValueChanged<ProductModel>? onFavoriteTap;
  final Set<String> favoriteIds;

  const TripTripProductGrid({
    Key? key,
    required this.products,
    this.scrollController,
    this.isLoading = false,
    this.onLoadMore,
    this.onProductTap,
    this.onFavoriteTap,
    this.favoriteIds = const {},
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    final tokens = TripTripDesignTokens.of(context);

    return NotificationListener<ScrollNotification>(
      onNotification: (notification) {
        if (notification is ScrollEndNotification &&
            notification.metrics.extentAfter < 200) {
          onLoadMore?.call();
        }
        return false;
      },
      child: GridView.builder(
        controller: scrollController,
        padding: EdgeInsets.all(tokens.spacing.md),
        gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
          crossAxisCount: _getCrossAxisCount(context),
          crossAxisSpacing: tokens.spacing.md,
          mainAxisSpacing: tokens.spacing.md,
          childAspectRatio: 0.7,
        ),
        itemCount: products.length + (isLoading ? 1 : 0),
        itemBuilder: (context, index) {
          if (index == products.length) {
            return Center(
              child: Padding(
                padding: EdgeInsets.all(tokens.spacing.lg),
                child: CircularProgressIndicator(
                  color: tokens.colors.primary,
                ),
              ),
            );
          }

          final product = products[index];
          return TripTripProductCard(
            imageUrl: product.imageUrl,
            title: product.name,
            subtitle: product.category,
            price: '¥${product.price.toStringAsFixed(0)}',
            originalPrice: product.originalPrice != null
                ? '¥${product.originalPrice!.toStringAsFixed(0)}'
                : null,
            rating: product.rating,
            reviewCount: product.reviewCount,
            isFavorite: favoriteIds.contains(product.id),
            badges: product.badges,
            onTap: () => onProductTap?.call(product),
            onFavorite: () => onFavoriteTap?.call(product),
          );
        },
      ),
    );
  }

  int _getCrossAxisCount(BuildContext context) {
    final width = MediaQuery.of(context).size.width;
    if (width > 1200) return 4;
    if (width > 800) return 3;
    return 2;
  }
}

/// ========================================
/// フィードバックコンポーネント
/// ========================================

/// スナックバー
class TripTripSnackBar {
  static void show(
    BuildContext context, {
    required String message,
    TripTripSnackBarType type = TripTripSnackBarType.info,
    Duration duration = const Duration(seconds: 3),
    String? actionLabel,
    VoidCallback? onAction,
  }) {
    final tokens = TripTripDesignTokens.of(context);

    final snackBar = SnackBar(
      content: Row(
        children: [
          Icon(
            _getIcon(type),
            color: Colors.white,
            size: 20,
          ),
          SizedBox(width: tokens.spacing.sm),
          Expanded(
            child: Text(
              message,
              style: tokens.typography.bodyMedium.copyWith(
                color: Colors.white,
              ),
            ),
          ),
        ],
      ),
      backgroundColor: _getBackgroundColor(type, tokens),
      duration: duration,
      behavior: SnackBarBehavior.floating,
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(tokens.radius.snackbar),
      ),
      margin: EdgeInsets.all(tokens.spacing.md),
      action: actionLabel != null
          ? SnackBarAction(
              label: actionLabel,
              textColor: Colors.white,
              onPressed: onAction ?? () {},
            )
          : null,
    );

    ScaffoldMessenger.of(context).showSnackBar(snackBar);
  }

  static IconData _getIcon(TripTripSnackBarType type) {
    switch (type) {
      case TripTripSnackBarType.success:
        return Icons.check_circle;
      case TripTripSnackBarType.error:
        return Icons.error;
      case TripTripSnackBarType.warning:
        return Icons.warning;
      case TripTripSnackBarType.info:
        return Icons.info;
    }
  }

  static Color _getBackgroundColor(
    TripTripSnackBarType type,
    TripTripDesignTokens tokens,
  ) {
    switch (type) {
      case TripTripSnackBarType.success:
        return tokens.colors.success;
      case TripTripSnackBarType.error:
        return tokens.colors.error;
      case TripTripSnackBarType.warning:
        return tokens.colors.warning;
      case TripTripSnackBarType.info:
        return tokens.colors.info;
    }
  }
}

enum TripTripSnackBarType { success, error, warning, info }

/// ローディングオーバーレイ
class TripTripLoadingOverlay extends StatelessWidget {
  final bool isLoading;
  final Widget child;
  final String? message;

  const TripTripLoadingOverlay({
    Key? key,
    required this.isLoading,
    required this.child,
    this.message,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    final tokens = TripTripDesignTokens.of(context);

    return Stack(
      children: [
        child,
        if (isLoading)
          Container(
            color: Colors.black.withOpacity(0.4),
            child: Center(
              child: Container(
                padding: EdgeInsets.all(tokens.spacing.lg),
                decoration: BoxDecoration(
                  color: tokens.colors.surface,
                  borderRadius: BorderRadius.circular(tokens.radius.card),
                ),
                child: Column(
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    CircularProgressIndicator(
                      color: tokens.colors.primary,
                    ),
                    if (message != null) ...[
                      SizedBox(height: tokens.spacing.md),
                      Text(
                        message!,
                        style: tokens.typography.bodyMedium,
                      ),
                    ],
                  ],
                ),
              ),
            ),
          ),
      ],
    );
  }
}

/// 空状態表示
class TripTripEmptyState extends StatelessWidget {
  final IconData icon;
  final String title;
  final String? description;
  final String? actionLabel;
  final VoidCallback? onAction;

  const TripTripEmptyState({
    Key? key,
    required this.icon,
    required this.title,
    this.description,
    this.actionLabel,
    this.onAction,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    final tokens = TripTripDesignTokens.of(context);

    return Center(
      child: Padding(
        padding: EdgeInsets.all(tokens.spacing.xl),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(
              icon,
              size: 64,
              color: tokens.colors.textTertiary,
            ),
            SizedBox(height: tokens.spacing.lg),
            Text(
              title,
              style: tokens.typography.titleMedium.copyWith(
                color: tokens.colors.textSecondary,
              ),
              textAlign: TextAlign.center,
            ),
            if (description != null) ...[
              SizedBox(height: tokens.spacing.sm),
              Text(
                description!,
                style: tokens.typography.bodyMedium.copyWith(
                  color: tokens.colors.textTertiary,
                ),
                textAlign: TextAlign.center,
              ),
            ],
            if (actionLabel != null && onAction != null) ...[
              SizedBox(height: tokens.spacing.lg),
              TripTripPrimaryButton(
                label: actionLabel!,
                onPressed: onAction,
              ),
            ],
          ],
        ),
      ),
    );
  }
}
```

### 2.3 デザイントークン

#### 2.3.1 デザイントークンシステム

```dart
/// ========================================
/// デザイントークン定義
/// ========================================

/// デザイントークンプロバイダー
class TripTripDesignTokens extends InheritedWidget {
  final TripTripColors colors;
  final TripTripTypography typography;
  final TripTripSpacing spacing;
  final TripTripRadius radius;
  final TripTripElevation elevation;
  final TripTripAnimation animation;

  const TripTripDesignTokens({
    Key? key,
    required this.colors,
    required this.typography,
    required this.spacing,
    required this.radius,
    required this.elevation,
    required this.animation,
    required Widget child,
  }) : super(key: key, child: child);

  static TripTripDesignTokens of(BuildContext context) {
    final result =
        context.dependOnInheritedWidgetOfExactType<TripTripDesignTokens>();
    assert(result != null, 'TripTripDesignTokens not found in context');
    return result!;
  }

  @override
  bool updateShouldNotify(TripTripDesignTokens oldWidget) {
    return colors != oldWidget.colors ||
        typography != oldWidget.typography ||
        spacing != oldWidget.spacing ||
        radius != oldWidget.radius ||
        elevation != oldWidget.elevation ||
        animation != oldWidget.animation;
  }

  /// ライトモードのデフォルトトークン
  static TripTripDesignTokens light({required Widget child}) {
    return TripTripDesignTokens(
      colors: TripTripColors.light(),
      typography: TripTripTypography.standard(),
      spacing: TripTripSpacing.standard(),
      radius: TripTripRadius.standard(),
      elevation: TripTripElevation.standard(),
      animation: TripTripAnimation.standard(),
      child: child,
    );
  }

  /// ダークモードのデフォルトトークン
  static TripTripDesignTokens dark({required Widget child}) {
    return TripTripDesignTokens(
      colors: TripTripColors.dark(),
      typography: TripTripTypography.standard(),
      spacing: TripTripSpacing.standard(),
      radius: TripTripRadius.standard(),
      elevation: TripTripElevation.standard(),
      animation: TripTripAnimation.standard(),
      child: child,
    );
  }
}

/// ========================================
/// カラートークン
/// ========================================

class TripTripColors {
  // プライマリカラー
  final Color primary;
  final Color primaryVariant;
  final Color primaryDisabled;
  final Color onPrimary;

  // セカンダリカラー
  final Color secondary;
  final Color secondaryVariant;
  final Color onSecondary;

  // サーフェスカラー
  final Color surface;
  final Color surfaceVariant;
  final Color surfaceDisabled;
  final Color onSurface;

  // バックグラウンドカラー
  final Color background;
  final Color onBackground;

  // エラー・ステータスカラー
  final Color error;
  final Color onError;
  final Color success;
  final Color warning;
  final Color info;

  // テキストカラー
  final Color textPrimary;
  final Color textSecondary;
  final Color textTertiary;
  final Color textDisabled;

  // その他
  final Color border;
  final Color divider;
  final Color rating;
  final Color overlay;

  const TripTripColors({
    required this.primary,
    required this.primaryVariant,
    required this.primaryDisabled,
    required this.onPrimary,
    required this.secondary,
    required this.secondaryVariant,
    required this.onSecondary,
    required this.surface,
    required this.surfaceVariant,
    required this.surfaceDisabled,
    required this.onSurface,
    required this.background,
    required this.onBackground,
    required this.error,
    required this.onError,
    required this.success,
    required this.warning,
    required this.info,
    required this.textPrimary,
    required this.textSecondary,
    required this.textTertiary,
    required this.textDisabled,
    required this.border,
    required this.divider,
    required this.rating,
    required this.overlay,
  });

  /// ライトモードカラー
  factory TripTripColors.light() {
    return const TripTripColors(
      // TripTripブランドカラー（例：旅をイメージした青緑系）
      primary: Color(0xFF0D9488),        // Teal 600
      primaryVariant: Color(0xFF0F766E), // Teal 700
      primaryDisabled: Color(0xFF99D5D0), // Teal 200
      onPrimary: Color(0xFFFFFFFF),

      secondary: Color(0xFFF97316),       // Orange 500
      secondaryVariant: Color(0xFFEA580C), // Orange 600
      onSecondary: Color(0xFFFFFFFF),

      surface: Color(0xFFFFFFFF),
      surfaceVariant: Color(0xFFF3F4F6), // Gray 100
      surfaceDisabled: Color(0xFFE5E7EB), // Gray 200
      onSurface: Color(0xFF1F2937),       // Gray 800

      background: Color(0xFFF9FAFB),      // Gray 50
      onBackground: Color(0xFF1F2937),    // Gray 800

      error: Color(0xFFDC2626),           // Red 600
      onError: Color(0xFFFFFFFF),
      success: Color(0xFF16A34A),         // Green 600
      warning: Color(0xFFD97706),         // Amber 600
      info: Color(0xFF2563EB),            // Blue 600

      textPrimary: Color(0xFF1F2937),     // Gray 800
      textSecondary: Color(0xFF6B7280),   // Gray 500
      textTertiary: Color(0xFF9CA3AF),    // Gray 400
      textDisabled: Color(0xFFD1D5DB),    // Gray 300

      border: Color(0xFFE5E7EB),          // Gray 200
      divider: Color(0xFFF3F4F6),         // Gray 100
      rating: Color(0xFFFBBF24),          // Amber 400
      overlay: Color(0x80000000),         // Black 50%
    );
  }

  /// ダークモードカラー
  factory TripTripColors.dark() {
    return const TripTripColors(
      primary: Color(0xFF2DD4BF),          // Teal 400
      primaryVariant: Color(0xFF14B8A6),   // Teal 500
      primaryDisabled: Color(0xFF115E59),  // Teal 800
      onPrimary: Color(0xFF042F2E),        // Teal 950

      secondary: Color(0xFFFB923C),        // Orange 400
      secondaryVariant: Color(0xFFF97316), // Orange 500
      onSecondary: Color(0xFF431407),      // Orange 950

      surface: Color(0xFF1F2937),          // Gray 800
      surfaceVariant: Color(0xFF374151),   // Gray 700
      surfaceDisabled: Color(0xFF4B5563),  // Gray 600
      onSurface: Color(0xFFF9FAFB),        // Gray 50

      background: Color(0xFF111827),       // Gray 900
      onBackground: Color(0xFFF9FAFB),     // Gray 50

      error: Color(0xFFF87171),            // Red 400
      onError: Color(0xFF450A0A),          // Red 950
      success: Color(0xFF4ADE80),          // Green 400
      warning: Color(0xFFFBBF24),          // Amber 400
      info: Color(0xFF60A5FA),             // Blue 400

      textPrimary: Color(0xFFF9FAFB),      // Gray 50
      textSecondary: Color(0xFFD1D5DB),    // Gray 300
      textTertiary: Color(0xFF9CA3AF),     // Gray 400
      textDisabled: Color(0xFF6B7280),     // Gray 500

      border: Color(0xFF374151),           // Gray 700
      divider: Color(0xFF4B5563),          // Gray 600
      rating: Color(0xFFFBBF24),           // Amber 400
      overlay: Color(0xCC000000),          // Black 80%
    );
  }
}

/// ========================================
/// タイポグラフィトークン
/// ========================================

class TripTripTypography {
  // ディスプレイ
  final TextStyle displayLarge;
  final TextStyle displayMedium;
  final TextStyle displaySmall;

  // ヘッドライン
  final TextStyle headlineLarge;
  final TextStyle headlineMedium;
  final TextStyle headlineSmall;

  // タイトル
  final TextStyle titleLarge;
  final TextStyle titleMedium;
  final TextStyle titleSmall;

  // ラベル
  final TextStyle labelLarge;
  final TextStyle labelMedium;
  final TextStyle labelSmall;

  // ボディ
  final TextStyle bodyLarge;
  final TextStyle bodyMedium;
  final TextStyle bodySmall;

  // キャプション
  final TextStyle caption;

  // ボタン
  final TextStyle buttonLarge;
  final TextStyle buttonMedium;
  final TextStyle buttonSmall;

  const TripTripTypography({
    required this.displayLarge,
    required this.displayMedium,
    required this.displaySmall,
    required this.headlineLarge,
    required this.headlineMedium,
    required this.headlineSmall,
    required this.titleLarge,
    required this.titleMedium,
    required this.titleSmall,
    required this.labelLarge,
    required this.labelMedium,
    required this.labelSmall,
    required this.bodyLarge,
    required this.bodyMedium,
    required this.bodySmall,
    required this.caption,
    required this.buttonLarge,
    required this.buttonMedium,
    required this.buttonSmall,
  });

  /// 標準タイポグラフィ（日本語対応）
  factory TripTripTypography.standard() {
    const fontFamily = 'NotoSansJP';
    const fallbackFonts = ['Roboto', 'Helvetica Neue', 'Arial', 'sans-serif'];

    return TripTripTypography(
      displayLarge: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 57,
        fontWeight: FontWeight.w400,
        letterSpacing: -0.25,
        height: 1.12,
      ),
      displayMedium: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 45,
        fontWeight: FontWeight.w400,
        letterSpacing: 0,
        height: 1.16,
      ),
      displaySmall: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 36,
        fontWeight: FontWeight.w400,
        letterSpacing: 0,
        height: 1.22,
      ),
      headlineLarge: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 32,
        fontWeight: FontWeight.w600,
        letterSpacing: 0,
        height: 1.25,
      ),
      headlineMedium: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 28,
        fontWeight: FontWeight.w600,
        letterSpacing: 0,
        height: 1.29,
      ),
      headlineSmall: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 24,
        fontWeight: FontWeight.w600,
        letterSpacing: 0,
        height: 1.33,
      ),
      titleLarge: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 22,
        fontWeight: FontWeight.w600,
        letterSpacing: 0,
        height: 1.27,
      ),
      titleMedium: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 16,
        fontWeight: FontWeight.w600,
        letterSpacing: 0.15,
        height: 1.5,
      ),
      titleSmall: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 14,
        fontWeight: FontWeight.w600,
        letterSpacing: 0.1,
        height: 1.43,
      ),
      labelLarge: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 14,
        fontWeight: FontWeight.w500,
        letterSpacing: 0.1,
        height: 1.43,
      ),
      labelMedium: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 12,
        fontWeight: FontWeight.w500,
        letterSpacing: 0.5,
        height: 1.33,
      ),
      labelSmall: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 11,
        fontWeight: FontWeight.w500,
        letterSpacing: 0.5,
        height: 1.45,
      ),
      bodyLarge: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 16,
        fontWeight: FontWeight.w400,
        letterSpacing: 0.5,
        height: 1.5,
      ),
      bodyMedium: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 14,
        fontWeight: FontWeight.w400,
        letterSpacing: 0.25,
        height: 1.43,
      ),
      bodySmall: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 12,
        fontWeight: FontWeight.w400,
        letterSpacing: 0.4,
        height: 1.33,
      ),
      caption: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 11,
        fontWeight: FontWeight.w400,
        letterSpacing: 0.4,
        height: 1.45,
      ),
      buttonLarge: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 16,
        fontWeight: FontWeight.w600,
        letterSpacing: 0.5,
        height: 1.5,
      ),
      buttonMedium: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 14,
        fontWeight: FontWeight.w600,
        letterSpacing: 0.5,
        height: 1.43,
      ),
      buttonSmall: const TextStyle(
        fontFamily: fontFamily,
        fontSize: 12,
        fontWeight: FontWeight.w600,
        letterSpacing: 0.5,
        height: 1.33,
      ),
    );
  }
}

/// ========================================
/// スペーシングトークン
/// ========================================

class TripTripSpacing {
  final double xxs;  // 2
  final double xs;   // 4
  final double sm;   // 8
  final double md;   // 16
  final double lg;   // 24
  final double xl;   // 32
  final double xxl;  // 48
  final double xxxl; // 64

  const TripTripSpacing({
    required this.xxs,
    required this.xs,
    required this.sm,
    required this.md,
    required this.lg,
    required this.xl,
    required this.xxl,
    required this.xxxl,
  });

  factory TripTripSpacing.standard() {
    return const TripTripSpacing(
      xxs: 2,
      xs: 4,
      sm: 8,
      md: 16,
      lg: 24,
      xl: 32,
      xxl: 48,
      xxxl: 64,
    );
  }
}

/// ========================================
/// 角丸トークン
/// ========================================

class TripTripRadius {
  final double none;     // 0
  final double xs;       // 2
  final double sm;       // 4
  final double md;       // 8
  final double lg;       // 12
  final double xl;       // 16
  final double xxl;      // 24
  final double full;     // 9999

  // 特定用途
  final double button;
  final double card;
  final double input;
  final double badge;
  final double snackbar;
  final double bottomSheet;

  const TripTripRadius({
    required this.none,
    required this.xs,
    required this.sm,
    required this.md,
    required this.lg,
    required this.xl,
    required this.xxl,
    required this.full,
    required this.button,
    required this.card,
    required this.input,
    required this.badge,
    required this.snackbar,
    required this.bottomSheet,
  });

  factory TripTripRadius.standard() {
    return const TripTripRadius(
      none: 0,
      xs: 2,
      sm: 4,
      md: 8,
      lg: 12,
      xl: 16,
      xxl: 24,
      full: 9999,
      button: 12,
      card: 16,
      input: 8,
      badge: 4,
      snackbar: 8,
      bottomSheet: 24,
    );
  }
}

/// ========================================
/// エレベーショントークン
/// ========================================

class TripTripElevation {
  final double none;
  final double xs;
  final double sm;
  final double md;
  final double lg;
  final double xl;

  // 特定用途
  final double button;
  final double card;
  final double appBar;
  final double bottomNav;
  final double dialog;
  final double bottomSheet;

  const TripTripElevation({
    required this.none,
    required this.xs,
    required this.sm,
    required this.md,
    required this.lg,
    required this.xl,
    required this.button,
    required this.card,
    required this.appBar,
    required this.bottomNav,
    required this.dialog,
    required this.bottomSheet,
  });

  factory TripTripElevation.standard() {
    return const TripTripElevation(
      none: 0,
      xs: 1,
      sm: 2,
      md: 4,
      lg: 8,
      xl: 16,
      button: 2,
      card: 2,
      appBar: 4,
      bottomNav: 8,
      dialog: 24,
      bottomSheet: 16,
    );
  }
}

/// ========================================
/// アニメーショントークン
/// ========================================

class TripTripAnimation {
  final Duration instant;
  final Duration fast;
  final Duration normal;
  final Duration slow;
  final Duration verySlow;

  final Curve standard;
  final Curve accelerate;
  final Curve decelerate;
  final Curve sharp;

  const TripTripAnimation({
    required this.instant,
    required this.fast,
    required this.normal,
    required this.slow,
    required this.verySlow,
    required this.standard,
    required this.accelerate,
    required this.decelerate,
    required this.sharp,
  });

  factory TripTripAnimation.standard() {
    return const TripTripAnimation(
      instant: Duration(milliseconds: 0),
      fast: Duration(milliseconds: 150),
      normal: Duration(milliseconds: 300),
      slow: Duration(milliseconds: 500),
      verySlow: Duration(milliseconds: 700),
      standard: Curves.easeInOut,
      accelerate: Curves.easeIn,
      decelerate: Curves.easeOut,
      sharp: Curves.easeInOutCubic,
    );
  }
}
```

---

## 第3章：ユーザーエクスペリエンス

### 3.1 UX原則

#### 3.1.1 TripTrip UXビジョン

```yaml
ux_vision:
  statement: "旅の全ての瞬間に、最高のデジタルコンパニオンを提供する"

  pillars:
    anticipatory:
      name: "先回りする体験"
      description: "ユーザーのニーズを予測し、適切なタイミングで情報を提供"
      examples:
        - "出発日が近づくとリマインダーを表示"
        - "天気予報に基づく持ち物提案"
        - "現地での移動手段の自動提案"

    contextual:
      name: "コンテキストに適応"
      description: "時間、場所、状況に応じて最適化されたUI/UX"
      examples:
        - "旅行中は現地情報を優先表示"
        - "夜間はダークモードを自動適用"
        - "オフライン時はローカル機能を強調"

    seamless:
      name: "シームレスな連携"
      description: "デバイス間、サービス間で途切れない体験"
      examples:
        - "スマートフォンで検索、PCで予約、現地でQR表示"
        - "カレンダー、マップアプリとの連携"
        - "交通機関、宿泊、アクティビティの一元管理"

    delightful:
      name: "心に残る体験"
      description: "機能性を超えた、感動的なモーメントの創出"
      examples:
        - "旅の思い出を美しくまとめるフォトブック機能"
        - "目的地に到着時の歓迎メッセージ"
        - "旅行後の振り返りサマリー"
```

#### 3.1.2 UXメトリクス＆KPI

```yaml
ux_metrics:
  usability:
    - metric: "System Usability Scale (SUS)"
      target: "80+"
      measurement: "四半期ごとのユーザー調査"

    - metric: "タスク完了率"
      target: "95%以上"
      key_tasks:
        - "ホテル検索〜予約"
        - "商品検索〜購入"
        - "チケット購入〜提示"

    - metric: "エラー率"
      target: "5%未満"
      measurement: "アナリティクス追跡"

  engagement:
    - metric: "DAU/MAU比率"
      target: "25%以上"

    - metric: "セッション時間"
      target: "5分以上"

    - metric: "機能利用率"
      target: "主要機能80%以上のユーザーが利用"

  satisfaction:
    - metric: "NPS (Net Promoter Score)"
      target: "50以上"

    - metric: "App Store評価"
      target: "4.5以上"

    - metric: "カスタマーエフォートスコア (CES)"
      target: "2.0未満（1=とても簡単、7=とても難しい）"

  performance:
    - metric: "アプリ起動時間"
      target: "2秒以内"

    - metric: "画面遷移時間"
      target: "300ms以内"

    - metric: "フレームドロップ率"
      target: "1%未満（60fps維持）"
```

### 3.2 ユーザーフロー設計

#### 3.2.1 主要ユーザーフロー

```yaml
user_flows:
  hotel_booking:
    name: "ホテル予約フロー"
    entry_points:
      - "ホーム画面のホテルカード"
      - "検索結果"
      - "お気に入りリスト"
    steps:
      1:
        screen: "検索入力"
        actions:
          - "目的地入力（オートコンプリート付き）"
          - "日程選択（カレンダーUI）"
          - "人数・部屋数選択"
        exit_to: [2]
      2:
        screen: "検索結果一覧"
        actions:
          - "フィルタリング（価格、評価、設備）"
          - "ソート（おすすめ、価格順、評価順）"
          - "マップ表示切り替え"
          - "ホテル選択"
        exit_to: [1, 3]
      3:
        screen: "ホテル詳細"
        actions:
          - "写真ギャラリー閲覧"
          - "施設・サービス確認"
          - "レビュー閲覧"
          - "部屋タイプ選択"
        exit_to: [2, 4]
      4:
        screen: "予約確認"
        actions:
          - "予約内容確認"
          - "ゲスト情報入力"
          - "支払い方法選択"
        exit_to: [3, 5]
      5:
        screen: "予約完了"
        actions:
          - "予約番号確認"
          - "カレンダーに追加"
          - "共有"
        exit_to: ["ホーム", "予約一覧"]

  product_purchase:
    name: "商品購入フロー"
    steps:
      1:
        screen: "商品一覧/検索"
        actions:
          - "カテゴリ選択"
          - "キーワード検索"
          - "フィルタリング"
      2:
        screen: "商品詳細"
        actions:
          - "画像確認"
          - "サイズ/カラー選択"
          - "数量選択"
          - "カートに追加"
      3:
        screen: "カート"
        actions:
          - "数量変更"
          - "削除"
          - "クーポン適用"
      4:
        screen: "チェックアウト"
        actions:
          - "配送先入力/選択"
          - "支払い方法選択"
          - "注文確定"
      5:
        screen: "注文完了"
        actions:
          - "注文番号確認"
          - "配送追跡"

  ticket_purchase:
    name: "チケット購入フロー"
    steps:
      1:
        screen: "アトラクション一覧"
        actions:
          - "地域/カテゴリ選択"
          - "日程フィルタ"
      2:
        screen: "アトラクション詳細"
        actions:
          - "情報確認"
          - "日時選択"
          - "チケットタイプ選択"
      3:
        screen: "チケット選択"
        actions:
          - "枚数選択"
          - "オプション追加"
      4:
        screen: "購入確認"
        actions:
          - "内容確認"
          - "支払い"
      5:
        screen: "チケット表示"
        actions:
          - "QRコード表示"
          - "ウォレットに追加"
```

#### 3.2.2 ユーザーフロー図

```
┌─────────────────────────────────────────────────────────────┐
│                    ホテル予約フロー                          │
└─────────────────────────────────────────────────────────────┘

     ┌─────────┐
     │ ホーム   │
     │ 画面    │
     └────┬────┘
          │
          ▼
     ┌─────────┐      ┌─────────┐
     │ 検索    │ ◄──► │ フィルタ │
     │ 入力    │      │ 設定    │
     └────┬────┘      └─────────┘
          │
          ▼
     ┌─────────┐      ┌─────────┐
     │ 検索結果 │ ◄──► │ マップ   │
     │ 一覧    │      │ ビュー   │
     └────┬────┘      └─────────┘
          │
          ▼
     ┌─────────┐      ┌─────────┐
     │ ホテル   │ ◄──► │ レビュー │
     │ 詳細    │      │ 一覧    │
     └────┬────┘      └─────────┘
          │
          ▼
     ┌─────────┐      ┌─────────┐
     │ 部屋    │ ◄──► │ 日程    │
     │ 選択    │      │ 変更    │
     └────┬────┘      └─────────┘
          │
          ▼
     ┌─────────┐
     │ 予約確認 │
     │ ・支払い │
     └────┬────┘
          │
          ▼
     ┌─────────┐
     │ 予約完了 │
     │ 確認    │
     └─────────┘
```

### 3.3 情報アーキテクチャ

#### 3.3.1 アプリ構造

```yaml
information_architecture:
  primary_navigation:
    - tab: "ホーム"
      icon: "home"
      content:
        - "パーソナライズされたおすすめ"
        - "進行中の旅程"
        - "最近の閲覧"
        - "季節のピックアップ"
        - "サービスカード（ホテル、チケット、商品等）"

    - tab: "検索"
      icon: "search"
      content:
        - "統合検索バー"
        - "カテゴリ別クイックアクセス"
        - "人気の検索ワード"
        - "検索履歴"

    - tab: "商品"
      icon: "shopping_bag"
      content:
        - "カテゴリナビゲーション"
        - "商品グリッド/リスト"
        - "フィルタ・ソート"

    - tab: "カート"
      icon: "shopping_cart"
      content:
        - "カートアイテム一覧"
        - "小計・合計"
        - "クーポン入力"
        - "チェックアウトへ進む"

    - tab: "マイページ"
      icon: "person"
      content:
        - "プロフィール"
        - "予約一覧"
        - "注文履歴"
        - "お気に入り"
        - "チケット一覧"
        - "設定"
        - "ヘルプ・サポート"

  secondary_navigation:
    - "通知"
    - "設定"
    - "ヘルプ"
    - "お問い合わせ"

  content_hierarchy:
    level_1: "カテゴリ（ホテル、チケット、商品）"
    level_2: "一覧ページ"
    level_3: "詳細ページ"
    level_4: "アクションページ（予約確認、チェックアウト）"
    level_5: "完了ページ"
```

---

## 第4章：アクセシビリティ

### 4.1 WCAG準拠

#### 4.1.1 WCAG 2.1 AA準拠チェックリスト

```yaml
wcag_compliance:
  perceivable:
    text_alternatives:
      - requirement: "1.1.1 非テキストコンテンツ"
        implementation:
          - "全ての画像にalt属性/Semantics label設定"
          - "装飾画像はセマンティクスから除外"
          - "アイコンボタンにアクセシブルラベル"
        flutter_example: |
          Semantics(
            label: '商品画像: 京都の着物レンタルセット',
            child: Image.network(imageUrl),
          )

    time_based_media:
      - requirement: "1.2.1 音声/動画コンテンツ"
        implementation:
          - "動画に字幕を提供"
          - "音声のみのコンテンツにテキスト代替"

    adaptable:
      - requirement: "1.3.1 情報と関係性"
        implementation:
          - "見出しの適切な構造化"
          - "リストの適切なマークアップ"
          - "フォームラベルの関連付け"

      - requirement: "1.3.4 表示の向き"
        implementation:
          - "横向き/縦向き両方をサポート"
          - "向きのロックを回避"

    distinguishable:
      - requirement: "1.4.3 コントラスト（最小）"
        implementation:
          - "テキスト: 4.5:1以上のコントラスト比"
          - "大きいテキスト: 3:1以上"
          - "UI要素: 3:1以上"
        color_combinations:
          - text: "#1F2937"
            background: "#FFFFFF"
            ratio: "12.6:1 ✓"
          - text: "#6B7280"
            background: "#FFFFFF"
            ratio: "5.0:1 ✓"

      - requirement: "1.4.4 テキストのサイズ変更"
        implementation:
          - "最大200%までのテキストサイズ変更をサポート"
          - "レイアウトの崩れ防止"

      - requirement: "1.4.11 非テキストのコントラスト"
        implementation:
          - "フォームフィールドの境界線"
          - "フォーカスインジケーター"
          - "アイコン"

  operable:
    keyboard_accessible:
      - requirement: "2.1.1 キーボード"
        implementation:
          - "全機能がキーボードで操作可能"
          - "フォーカストラップの回避"
          - "ショートカットキーの提供"

    enough_time:
      - requirement: "2.2.1 タイミング調整可能"
        implementation:
          - "セッションタイムアウトの延長オプション"
          - "自動スライドショーの停止機能"

    navigable:
      - requirement: "2.4.1 ブロックスキップ"
        implementation:
          - "メインコンテンツへのスキップリンク"
          - "ナビゲーションのスキップ"

      - requirement: "2.4.3 フォーカス順序"
        implementation:
          - "論理的なタブ順序"
          - "モーダル内のフォーカストラップ"

      - requirement: "2.4.6 見出しとラベル"
        implementation:
          - "説明的な見出し"
          - "明確なフォームラベル"

      - requirement: "2.4.7 フォーカスの可視化"
        implementation:
          - "明確なフォーカスインジケーター"
          - "カスタムフォーカススタイル"
        flutter_example: |
          Focus(
            child: Container(
              decoration: BoxDecoration(
                border: Border.all(
                  color: isFocused ? Colors.blue : Colors.transparent,
                  width: 2,
                ),
              ),
              child: child,
            ),
          )

  understandable:
    readable:
      - requirement: "3.1.1 ページの言語"
        implementation:
          - "言語属性の設定"
          - "多言語コンテンツの言語指定"

    predictable:
      - requirement: "3.2.1 フォーカス時"
        implementation:
          - "フォーカス移動で予期しない変更を起こさない"

      - requirement: "3.2.2 入力時"
        implementation:
          - "入力だけでページ遷移しない"
          - "明示的なサブミットアクション"

    input_assistance:
      - requirement: "3.3.1 エラー特定"
        implementation:
          - "エラーの明確な表示"
          - "エラー箇所の特定"

      - requirement: "3.3.2 ラベルまたは説明"
        implementation:
          - "入力フィールドのラベル"
          - "必須項目の明示"
          - "入力形式のヒント"

      - requirement: "3.3.3 エラー修正の提案"
        implementation:
          - "具体的なエラー原因の説明"
          - "修正方法の提案"

  robust:
    compatible:
      - requirement: "4.1.2 名前、役割、値"
        implementation:
          - "カスタムコンポーネントの適切なセマンティクス"
          - "状態変更の通知"
        flutter_example: |
          Semantics(
            button: true,
            label: 'カートに追加',
            onTap: () => addToCart(),
            enabled: isAvailable,
            child: AddToCartButton(),
          )
```

### 4.2 多言語対応

#### 4.2.1 国際化（i18n）戦略

```yaml
internationalization:
  supported_languages:
    primary:
      - code: "ja"
        name: "日本語"
        direction: "LTR"
        status: "Full Support"
      - code: "en"
        name: "English"
        direction: "LTR"
        status: "Full Support"

    secondary:
      - code: "zh-CN"
        name: "简体中文"
        direction: "LTR"
        status: "Planned Q2 2025"
      - code: "zh-TW"
        name: "繁體中文"
        direction: "LTR"
        status: "Planned Q2 2025"
      - code: "ko"
        name: "한국어"
        direction: "LTR"
        status: "Planned Q3 2025"

  implementation:
    flutter_localization:
      package: "flutter_localizations"
      arb_files: true
      code_generation: true

    string_management:
      - "ARBファイルでの翻訳管理"
      - "プレースホルダーのサポート"
      - "複数形の処理"
      - "日付/時刻/通貨のローカライズ"

    content_localization:
      - "UIテキスト"
      - "エラーメッセージ"
      - "ヘルプコンテンツ"
      - "マーケティングコンテンツ"
      - "メール通知"

  typography_considerations:
    japanese:
      - "Noto Sans JP フォント使用"
      - "縦書きオプション（特定UI）"
      - "ふりがな対応"
    chinese:
      - "Noto Sans SC/TC フォント"
      - "簡体字/繁体字の切り替え"
    korean:
      - "Noto Sans KR フォント"

  layout_considerations:
    - "テキスト展開への対応（日本語→英語で1.5倍程度）"
    - "固定幅の回避"
    - "テキストの折り返し対応"
    - "数値・日付フォーマットのローカライズ"
```

#### 4.2.2 Flutter i18n実装

```dart
// l10n/app_ja.arb
{
  "@@locale": "ja",
  "appName": "TripTrip",
  "homeTitle": "ホーム",
  "searchHint": "目的地、ホテル、アクティビティを検索",
  "cartTitle": "カート",
  "cartEmpty": "カートに商品がありません",
  "cartItemCount": "{count, plural, =0{商品なし} =1{1件の商品} other{{count}件の商品}}",
  "@cartItemCount": {
    "placeholders": {
      "count": {
        "type": "int"
      }
    }
  },
  "priceFormat": "¥{price}",
  "@priceFormat": {
    "placeholders": {
      "price": {
        "type": "String"
      }
    }
  },
  "dateFormat": "{date}",
  "@dateFormat": {
    "placeholders": {
      "date": {
        "type": "DateTime",
        "format": "yMMMd"
      }
    }
  },
  "addToCart": "カートに追加",
  "checkout": "レジに進む",
  "bookNow": "予約する",
  "viewDetails": "詳細を見る",
  "errorGeneric": "エラーが発生しました。しばらくしてからもう一度お試しください。",
  "errorNetwork": "ネットワークに接続できません。接続を確認してください。",
  "loadingText": "読み込み中..."
}

// l10n/app_en.arb
{
  "@@locale": "en",
  "appName": "TripTrip",
  "homeTitle": "Home",
  "searchHint": "Search destinations, hotels, activities",
  "cartTitle": "Cart",
  "cartEmpty": "Your cart is empty",
  "cartItemCount": "{count, plural, =0{No items} =1{1 item} other{{count} items}}",
  "priceFormat": "¥{price}",
  "dateFormat": "{date}",
  "addToCart": "Add to Cart",
  "checkout": "Checkout",
  "bookNow": "Book Now",
  "viewDetails": "View Details",
  "errorGeneric": "An error occurred. Please try again later.",
  "errorNetwork": "Unable to connect. Please check your connection.",
  "loadingText": "Loading..."
}

// 使用例
class LocalizedWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context)!;

    return Column(
      children: [
        Text(l10n.homeTitle),
        Text(l10n.cartItemCount(3)),
        Text(l10n.priceFormat('1,500')),
        Text(l10n.dateFormat(DateTime.now())),
      ],
    );
  }
}
```

### 4.3 支援技術対応

#### 4.3.1 スクリーンリーダー対応

```dart
/// スクリーンリーダー対応のベストプラクティス

/// 画像のアクセシビリティ
class AccessibleImage extends StatelessWidget {
  final String imageUrl;
  final String semanticLabel;
  final bool isDecorative;

  const AccessibleImage({
    Key? key,
    required this.imageUrl,
    required this.semanticLabel,
    this.isDecorative = false,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Semantics(
      image: true,
      label: isDecorative ? null : semanticLabel,
      excludeSemantics: isDecorative,
      child: Image.network(
        imageUrl,
        semanticLabel: isDecorative ? '' : semanticLabel,
      ),
    );
  }
}

/// ボタンのアクセシビリティ
class AccessibleButton extends StatelessWidget {
  final String label;
  final String? hint;
  final VoidCallback onPressed;
  final bool isEnabled;
  final Widget child;

  const AccessibleButton({
    Key? key,
    required this.label,
    this.hint,
    required this.onPressed,
    this.isEnabled = true,
    required this.child,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Semantics(
      button: true,
      label: label,
      hint: hint,
      enabled: isEnabled,
      onTap: isEnabled ? onPressed : null,
      child: GestureDetector(
        onTap: isEnabled ? onPressed : null,
        child: child,
      ),
    );
  }
}

/// フォームフィールドのアクセシビリティ
class AccessibleTextField extends StatelessWidget {
  final String label;
  final String? hint;
  final String? errorText;
  final TextEditingController controller;
  final bool isRequired;

  const AccessibleTextField({
    Key? key,
    required this.label,
    this.hint,
    this.errorText,
    required this.controller,
    this.isRequired = false,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Semantics(
      textField: true,
      label: '$label${isRequired ? '、必須' : ''}',
      hint: hint,
      value: controller.text,
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Row(
            children: [
              Text(label),
              if (isRequired)
                Text(
                  ' *',
                  style: TextStyle(color: Colors.red),
                  semanticsLabel: '',
                ),
            ],
          ),
          TextField(
            controller: controller,
            decoration: InputDecoration(
              hintText: hint,
              errorText: errorText,
            ),
          ),
          if (errorText != null)
            Semantics(
              liveRegion: true,
              child: Text(
                errorText!,
                style: TextStyle(color: Colors.red),
              ),
            ),
        ],
      ),
    );
  }
}

/// リストアイテムのアクセシビリティ
class AccessibleListItem extends StatelessWidget {
  final int index;
  final int total;
  final String title;
  final String? subtitle;
  final VoidCallback onTap;

  const AccessibleListItem({
    Key? key,
    required this.index,
    required this.total,
    required this.title,
    this.subtitle,
    required this.onTap,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Semantics(
      label: '$title${subtitle != null ? '、$subtitle' : ''}',
      hint: '${index + 1}件目、全${total}件中、ダブルタップで選択',
      child: ListTile(
        title: Text(title),
        subtitle: subtitle != null ? Text(subtitle!) : null,
        onTap: onTap,
      ),
    );
  }
}

/// 画面遷移のアナウンス
class NavigationAnnouncer {
  static void announceScreenChange(BuildContext context, String screenName) {
    SemanticsService.announce(
      '$screenName画面に移動しました',
      TextDirection.ltr,
    );
  }

  static void announceAction(BuildContext context, String action) {
    SemanticsService.announce(
      action,
      TextDirection.ltr,
    );
  }
}

/// 使用例
class AccessibleProductCard extends StatelessWidget {
  final Product product;
  final VoidCallback onTap;
  final VoidCallback onAddToCart;

  @override
  Widget build(BuildContext context) {
    return Semantics(
      container: true,
      label: _buildSemanticLabel(),
      child: Card(
        child: Column(
          children: [
            AccessibleImage(
              imageUrl: product.imageUrl,
              semanticLabel: '${product.name}の商品画像',
            ),
            Text(product.name),
            Text('¥${product.price}'),
            Row(
              children: [
                AccessibleButton(
                  label: '詳細を見る',
                  hint: '${product.name}の詳細ページを開きます',
                  onPressed: onTap,
                  child: Text('詳細'),
                ),
                AccessibleButton(
                  label: 'カートに追加',
                  hint: '${product.name}をカートに追加します',
                  onPressed: () {
                    onAddToCart();
                    NavigationAnnouncer.announceAction(
                      context,
                      '${product.name}をカートに追加しました',
                    );
                  },
                  child: Icon(Icons.add_shopping_cart),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }

  String _buildSemanticLabel() {
    final parts = [
      product.name,
      '価格¥${product.price}',
      if (product.rating != null) '評価${product.rating}',
      if (product.reviewCount != null) 'レビュー${product.reviewCount}件',
    ];
    return parts.join('、');
  }
}
```

---

## 第5章：実装ロードマップ＆文書間参照

### 5.1 実装ロードマップ

#### 5.1.1 フェーズ別実装計画

```yaml
implementation_roadmap:
  phase_1_foundation:
    name: "基盤構築"
    timeline: "2024 Q2"
    duration: "8週間"
    objectives:
      - "デザイントークンシステムの実装"
      - "基本コンポーネントライブラリの構築"
      - "アクセシビリティ基盤の確立"
    deliverables:
      - "TripTripDesignTokens パッケージ"
      - "基本UIコンポーネント（20種類）"
      - "Figmaデザインシステム"
      - "コンポーネントドキュメント"
    resources:
      - "デザイナー: 2名"
      - "フロントエンドエンジニア: 3名"
    milestones:
      week_2: "デザイントークン仕様確定"
      week_4: "基本コンポーネント実装完了"
      week_6: "Figma連携完了"
      week_8: "ドキュメント完成"

  phase_2_enhancement:
    name: "機能拡張"
    timeline: "2024 Q3"
    duration: "8週間"
    objectives:
      - "複合コンポーネントの追加"
      - "ダークモード対応"
      - "アニメーション・トランジション"
    deliverables:
      - "複合コンポーネント（15種類）"
      - "ダークモードテーマ"
      - "アニメーションライブラリ"
      - "Storybook/Widgetbook カタログ"
    resources:
      - "デザイナー: 2名"
      - "フロントエンドエンジニア: 4名"
      - "モーションデザイナー: 1名"
    milestones:
      week_2: "複合コンポーネント設計完了"
      week_4: "ダークモード実装"
      week_6: "アニメーション実装"
      week_8: "カタログ公開"

  phase_3_accessibility:
    name: "アクセシビリティ強化"
    timeline: "2024 Q4"
    duration: "6週間"
    objectives:
      - "WCAG 2.1 AA完全準拠"
      - "スクリーンリーダー最適化"
      - "アクセシビリティテスト自動化"
    deliverables:
      - "アクセシビリティ監査レポート"
      - "セマンティクス最適化"
      - "自動テストスイート"
    resources:
      - "アクセシビリティスペシャリスト: 1名"
      - "フロントエンドエンジニア: 2名"
      - "QAエンジニア: 1名"

  phase_4_globalization:
    name: "グローバル対応"
    timeline: "2025 Q1"
    duration: "8週間"
    objectives:
      - "多言語対応（中国語、韓国語）"
      - "RTL対応準備"
      - "地域別カスタマイズ"
    deliverables:
      - "多言語リソースファイル"
      - "ローカライズワークフロー"
      - "地域別テーマ"
```

#### 5.1.2 マイルストーン詳細

```yaml
milestones:
  m1_design_system_v1:
    date: "2024-06-30"
    criteria:
      - "デザイントークン100%実装"
      - "基本コンポーネント20種類完成"
      - "Figma↔Flutter同期確立"
      - "内部チーム向けドキュメント完成"

  m2_component_library_complete:
    date: "2024-09-30"
    criteria:
      - "全コンポーネント35種類以上"
      - "ダークモード完全対応"
      - "アニメーション標準化"
      - "Widgetbookカタログ公開"

  m3_accessibility_certified:
    date: "2024-12-15"
    criteria:
      - "WCAG 2.1 AA監査合格"
      - "VoiceOver/TalkBack完全対応"
      - "アクセシビリティテスト自動化率80%"

  m4_global_ready:
    date: "2025-03-31"
    criteria:
      - "5言語対応完了"
      - "翻訳ワークフロー確立"
      - "地域別A/Bテスト基盤"
```

### 5.2 文書間参照

#### 5.2.1 関連文書マッピング

```yaml
document_references:
  upstream_documents:
    - doc_id: "Doc-AD-001"
      title: "モバイルアプリアーキテクチャ"
      relationship: "UI実装の技術基盤"
      key_dependencies:
        - "Flutterアーキテクチャパターン"
        - "状態管理（Provider/Riverpod）"
        - "ナビゲーション設計"

    - doc_id: "Doc-AD-002"
      title: "Webアプリアーキテクチャ"
      relationship: "Web版デザインシステムの基盤"
      key_dependencies:
        - "レスポンシブデザイン戦略"
        - "Webコンポーネント構成"

    - doc_id: "Doc-TV-003"
      title: "イノベーション＆R&D戦略"
      relationship: "AR/VR UI、AI駆動UXの方向性"
      key_dependencies:
        - "AR観光ガイドUI設計"
        - "AIアシスタントUI"

  downstream_documents:
    - doc_id: "Doc-QA-001"
      title: "品質保証戦略"
      relationship: "UIテスト戦略の基盤"
      provides:
        - "コンポーネントテスト仕様"
        - "アクセシビリティテスト仕様"
        - "ビジュアルリグレッションテスト"

    - doc_id: "Doc-IR-003"
      title: "スプリント計画＆デリバリーメトリクス"
      relationship: "UI開発のスプリント計画"
      provides:
        - "コンポーネント開発タスク"
        - "UX改善タスク"

  cross_references:
    - doc_id: "Doc-BM-002"
      title: "収益モデル"
      relationship: "コンバージョン最適化UI"

    - doc_id: "Doc-MS-001"
      title: "マーケティング戦略"
      relationship: "ブランドガイドライン連携"

    - doc_id: "Doc-OS-001"
      title: "オペレーションモデル"
      relationship: "サポートUI、FAQ設計"
```

### 5.3 成功指標＆評価

#### 5.3.1 デザインシステムKPI

```yaml
success_metrics:
  adoption:
    - metric: "コンポーネント採用率"
      target: "90%以上の画面でデザインシステム使用"
      measurement: "コードレビュー、静的解析"

    - metric: "デザイントークン準拠率"
      target: "ハードコード値0%"
      measurement: "Lintルール、CI/CD検証"

  efficiency:
    - metric: "新規画面開発時間"
      target: "30%削減"
      baseline: "現在の平均開発時間"

    - metric: "デザイン↔実装の乖離"
      target: "ピクセルパーフェクト率95%"
      measurement: "ビジュアルリグレッションテスト"

  quality:
    - metric: "UIバグ発生率"
      target: "リリースあたり5件未満"

    - metric: "アクセシビリティ違反"
      target: "0件（Critical/Major）"

  satisfaction:
    - metric: "開発者満足度"
      target: "4.0/5.0以上"
      measurement: "四半期サーベイ"

    - metric: "ユーザーUI満足度"
      target: "4.5/5.0以上"
      measurement: "アプリ内フィードバック"
```

---

## 結論

本文書は、TripTripプラットフォームにおけるユーザーインターフェースとユーザーエクスペリエンスデザインの包括的な戦略と実装ガイドラインを定義しました。

### 主要な設計決定

1. **デザインシステムの確立**: Material Design 3をベースに、TripTripブランドに最適化したデザイントークンとコンポーネントライブラリ
2. **アクセシビリティファースト**: WCAG 2.1 AA準拠を基本要件とし、すべてのユーザーに対応
3. **国際化対応**: 日本語・英語を基盤に、アジア市場への拡大を見据えた多言語設計
4. **パフォーマンス重視**: 60fps維持、高速なレスポンスを実現するUI設計
5. **段階的実装**: 基盤構築→機能拡張→アクセシビリティ→グローバル化の4フェーズ

### 期待される成果

- 一貫したブランド体験による信頼性向上
- 直感的なUIによるコンバージョン率改善
- アクセシビリティ対応による市場拡大
- 開発効率の30%向上

これらの設計により、TripTripは旅行業界における最高水準のユーザー体験を提供します。

---

**文書情報**
- Document ID: Doc-AD-004
- Version: 1.0.0
- Last Updated: 2026-01-21
- Status: Draft
- Total Lines: 1,800+
- Author: Technical Architecture Team / UX Design Team

**関連文書**
- Doc-AD-001: モバイルアプリアーキテクチャ
- Doc-AD-002: Webアプリアーキテクチャ
- Doc-AD-003: API設計
- Doc-TV-003: イノベーション＆R&D戦略
- Doc-QA-001: 品質保証戦略

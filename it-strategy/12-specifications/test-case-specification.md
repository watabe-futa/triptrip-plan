# Doc-SP-004: TripTripテストケース一覧・テスト計画詳細

## エグゼクティブサマリー

本文書は、TripTripプラットフォームの包括的なテストケース一覧とテスト計画詳細を定義します。Google、Amazon、Spotifyの品質保証ベストプラクティスを統合し、機能テスト（認証、検索、予約、決済）、E2Eテストシナリオ（ユーザージャーニー別）、APIテスト（エンドポイント別）、非機能テスト（パフォーマンス、セキュリティ）、モバイル固有テスト（オフライン、デバイス互換性）を体系的に整備します。既存の46テストファイル基盤を発展させ、テストカバレッジ80%以上、クリティカルパス100%カバー、OWASP Top 10対応を目標とします。各テストケースは即実行可能な詳細度で記述し、優先度・自動化可否を明確化します。

---

## 第1章：はじめに & テスト戦略概要

### 1.1 テスト目的と範囲

#### 1.1.1 文書の目的

本テストケース一覧・テスト計画詳細は、TripTripの品質保証活動を支える具体的なテストケースと実行計画を提供します。

```yaml
document_purpose:
  primary_goals:
    - 全機能をカバーする体系的なテストケース一覧の提供
    - 即実行可能な詳細度のテストシナリオ定義
    - テスト優先度と自動化戦略の明確化
    - 品質目標達成のためのロードマップ提示

  target_audience:
    - QAエンジニア（テスト実行者）
    - ソフトウェアエンジニア（開発者テスト）
    - テックリード（品質レビュー）
    - プロダクトマネージャー（品質状況把握）

  scope:
    in_scope:
      - 機能テストケース（認証、検索、予約、決済等）
      - E2Eテストシナリオ（主要ユーザージャーニー）
      - APIテストケース（全エンドポイント）
      - 非機能テスト（パフォーマンス、セキュリティ）
      - モバイル固有テスト
    out_of_scope:
      - テスト戦略・方針（→Doc-QA-001参照）
      - 監視・インシデント管理（→Doc-QA-002参照）
      - リリース管理（→Doc-QA-003参照）
```

#### 1.1.2 テスト範囲定義

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     テスト範囲マトリクス                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  機能領域              │ ユニット │ 統合 │ E2E │ API │ 性能 │ セキュリティ │
│  ─────────────────────┼─────────┼──────┼─────┼─────┼──────┼────────────┤
│  認証・アカウント       │    ●    │  ●   │  ●  │  ●  │  ○  │     ●      │
│  商品検索・表示         │    ●    │  ●   │  ●  │  ●  │  ●  │     ○      │
│  ホテル検索・予約       │    ●    │  ●   │  ●  │  ●  │  ●  │     ○      │
│  アトラクションチケット │    ●    │  ●   │  ●  │  ●  │  ○  │     ○      │
│  カート・チェックアウト │    ●    │  ●   │  ●  │  ●  │  ●  │     ●      │
│  決済処理              │    ●    │  ●   │  ●  │  ●  │  ●  │     ●      │
│  注文管理              │    ●    │  ●   │  ●  │  ●  │  ○  │     ○      │
│  ユーザープロフィール   │    ●    │  ○   │  ○  │  ●  │  ○  │     ●      │
│  通知                  │    ○    │  ○   │  ○  │  ●  │  ○  │     ○      │
│  ─────────────────────┴─────────┴──────┴─────┴─────┴──────┴────────────┤
│  ●: 必須  ○: 推奨  空白: 対象外                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 テスト環境要件

#### 1.2.1 環境構成

```yaml
test_environments:
  local_development:
    purpose: 開発中のユニットテスト・ウィジェットテスト
    flutter:
      sdk: Flutter 3.8.1+
      dart: Dart 3.8.1+
      emulator: Android Emulator / iOS Simulator
    backend:
      runtime: Node.js 18+
      database: SQLite (in-memory) または Docker PostgreSQL
      services: モック化

  ci_environment:
    purpose: 自動テスト実行（PR、main branch）
    infrastructure:
      runner: GitHub Actions (ubuntu-latest)
      database: Docker PostgreSQL 16
      services: モック / サンドボックス
    parallelization:
      flutter_tests: matrix strategy
      backend_tests: parallel jobs
    caching:
      - Flutter SDK
      - npm packages
      - Gradle dependencies

  staging_environment:
    purpose: E2Eテスト、統合テスト、パフォーマンステスト
    infrastructure:
      backend: Cloud Run (staging)
      database: Cloud SQL PostgreSQL
      services: サンドボックス環境
    data:
      type: 匿名化テストデータ
      refresh: 週次

  device_farm:
    purpose: 実機テスト、デバイス互換性テスト
    platforms:
      android:
        - Pixel 6 (Android 13)
        - Samsung Galaxy S22 (Android 12)
        - Xiaomi Redmi Note 11 (Android 11)
      ios:
        - iPhone 14 Pro (iOS 17)
        - iPhone 12 (iOS 16)
        - iPad Pro 12.9" (iPadOS 17)
    provider: Firebase Test Lab
```

#### 1.2.2 テストデータ要件

```yaml
test_data_requirements:
  user_accounts:
    - type: guest_user
      count: 動的生成
    - type: registered_user
      count: 10+
      attributes: [email, password, profile]
    - type: premium_user
      count: 5+
      attributes: [subscription, benefits]
    - type: admin_user
      count: 2
      attributes: [admin_permissions]

  products:
    - category: kimono
      count: 50+
      variants: [size, color, price_range]
    - category: accessories
      count: 100+
    - category: souvenirs
      count: 200+

  hotels:
    - location: Tokyo
      count: 30+
      room_types: [single, double, suite]
    - location: Kyoto
      count: 20+
    - location: Osaka
      count: 15+

  attractions:
    - type: museum
      count: 10+
      time_slots: 複数
    - type: theme_park
      count: 5+
    - type: cultural_site
      count: 15+

  orders:
    - status: pending
      count: 動的生成
    - status: completed
      count: 履歴用50+
    - status: cancelled
      count: 10+
```

### 1.3 テストツール一覧

#### 1.3.1 ツールスタック

```yaml
test_tools:
  flutter_testing:
    unit_test:
      framework: flutter_test
      mocking: mockito 5.4.4
      assertions: expect, matchers
    widget_test:
      framework: flutter_test
      golden: golden_toolkit (将来)
    integration_test:
      framework: patrol 3.19.0
      alternative: integration_test
    coverage:
      tool: lcov
      threshold: 80%

  backend_testing:
    unit_test:
      framework: vitest 1.0.0
      mocking: vi.mock, vi.fn
    integration_test:
      framework: vitest + supertest 6.3.3
      database: Docker PostgreSQL
    coverage:
      tool: c8 / istanbul
      threshold: 80%

  api_testing:
    manual:
      tool: Postman / Insomnia
    automated:
      tool: supertest
    documentation:
      tool: OpenAPI 3.0 / Swagger UI

  performance_testing:
    load_test:
      tool: k6
      scenarios: [load, stress, spike, soak]
    frontend:
      tool: Lighthouse CI
    monitoring:
      tool: Grafana

  security_testing:
    sast:
      tool: SonarQube, Snyk
    dast:
      tool: OWASP ZAP
    dependency:
      tool: npm audit, Snyk

  mobile_testing:
    device_farm:
      tool: Firebase Test Lab
    accessibility:
      tool: Accessibility Scanner (Android), Accessibility Inspector (iOS)
```

---

## 第2章：機能テストケース

### 2.1 ユーザー認証・アカウント管理

#### 2.1.1 ユーザー登録テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-AUTH-001 | 有効なメールアドレスで新規登録 | アプリ起動、未ログイン状態 | 1. 登録画面を開く 2. 有効なメールアドレスを入力 3. パスワード(8文字以上、英数字混合)を入力 4. 確認パスワードを入力 5. 登録ボタンをタップ | ユーザーアカウントが作成され、ホーム画面に遷移する。確認メールが送信される | Critical | Yes |
| TC-AUTH-002 | 無効なメールアドレスで登録試行 | アプリ起動、未ログイン状態 | 1. 登録画面を開く 2. 無効なメールアドレス(例: "test@")を入力 3. 有効なパスワードを入力 4. 登録ボタンをタップ | バリデーションエラーが表示され、登録が拒否される | High | Yes |
| TC-AUTH-003 | パスワード要件不足で登録試行 | アプリ起動、未ログイン状態 | 1. 登録画面を開く 2. 有効なメールアドレスを入力 3. 弱いパスワード(例: "123")を入力 4. 登録ボタンをタップ | パスワード要件エラーが表示される | High | Yes |
| TC-AUTH-004 | パスワード確認不一致で登録試行 | アプリ起動、未ログイン状態 | 1. 登録画面を開く 2. 有効なメールアドレスを入力 3. パスワードと異なる確認パスワードを入力 4. 登録ボタンをタップ | パスワード不一致エラーが表示される | High | Yes |
| TC-AUTH-005 | 既存メールアドレスで登録試行 | 登録済みメールアドレスが存在 | 1. 登録画面を開く 2. 既存のメールアドレスを入力 3. 有効なパスワードを入力 4. 登録ボタンをタップ | 「このメールアドレスは既に登録されています」エラーが表示される | High | Yes |
| TC-AUTH-006 | 空フィールドで登録試行 | アプリ起動、未ログイン状態 | 1. 登録画面を開く 2. 全フィールドを空のまま登録ボタンをタップ | 必須フィールドエラーが表示される | Medium | Yes |

#### 2.1.2 ログインテストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-AUTH-010 | 有効な認証情報でログイン | 登録済みユーザーが存在 | 1. ログイン画面を開く 2. 登録済みメールアドレスを入力 3. 正しいパスワードを入力 4. ログインボタンをタップ | ログイン成功、ホーム画面に遷移、ユーザー情報が表示される | Critical | Yes |
| TC-AUTH-011 | 無効なパスワードでログイン試行 | 登録済みユーザーが存在 | 1. ログイン画面を開く 2. 登録済みメールアドレスを入力 3. 誤ったパスワードを入力 4. ログインボタンをタップ | 「メールアドレスまたはパスワードが正しくありません」エラーが表示される | Critical | Yes |
| TC-AUTH-012 | 未登録メールアドレスでログイン試行 | - | 1. ログイン画面を開く 2. 未登録のメールアドレスを入力 3. 任意のパスワードを入力 4. ログインボタンをタップ | 認証エラーが表示される（存在有無は漏らさない） | High | Yes |
| TC-AUTH-013 | ログイン状態の永続化確認 | ログイン済み状態 | 1. アプリを完全に終了 2. アプリを再起動 | ログイン状態が維持され、ホーム画面が表示される | High | Yes |
| TC-AUTH-014 | 5回連続ログイン失敗でアカウントロック | 登録済みユーザーが存在 | 1. 誤ったパスワードで5回連続ログイン試行 | アカウントが一時ロックされ、「しばらくしてから再試行してください」メッセージが表示される | High | Yes |
| TC-AUTH-015 | ログアウト機能 | ログイン済み状態 | 1. ユーザープロフィール画面を開く 2. ログアウトボタンをタップ 3. 確認ダイアログで「はい」を選択 | ログアウト完了、ログイン画面に遷移、ローカルトークンが削除される | High | Yes |

#### 2.1.3 ソーシャルログインテストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-AUTH-020 | Googleアカウントでログイン | Googleアカウント保有 | 1. ログイン画面を開く 2. 「Googleでログイン」ボタンをタップ 3. Googleアカウントを選択 4. 権限を許可 | ログイン成功、ユーザープロフィールにGoogle情報が連携される | High | No |
| TC-AUTH-021 | Appleアカウントでログイン(iOS) | iOSデバイス、Appleアカウント保有 | 1. ログイン画面を開く 2. 「Appleでサインイン」ボタンをタップ 3. Face ID/Touch IDで認証 | ログイン成功、Appleアカウントと連携される | High | No |
| TC-AUTH-022 | ソーシャルログイン後のアカウント連携確認 | ソーシャルログイン済み | 1. プロフィール画面を開く 2. 連携アカウント情報を確認 | 連携されたソーシャルアカウント情報が表示される | Medium | No |

#### 2.1.4 パスワードリセットテストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-AUTH-030 | パスワードリセットメール送信 | 登録済みメールアドレス | 1. ログイン画面で「パスワードを忘れた」をタップ 2. 登録済みメールアドレスを入力 3. 送信ボタンをタップ | 「リセットメールを送信しました」メッセージが表示される | High | Yes |
| TC-AUTH-031 | リセットリンクからパスワード変更 | リセットメール受信済み | 1. メール内のリンクをタップ 2. 新しいパスワードを入力 3. 確認パスワードを入力 4. 変更ボタンをタップ | パスワードが更新され、ログイン画面に遷移する | High | No |
| TC-AUTH-032 | 期限切れリセットリンクの使用 | 24時間以上経過したリセットリンク | 1. 期限切れリンクをタップ | 「リンクの有効期限が切れています」エラーが表示される | Medium | No |

### 2.2 検索機能

#### 2.2.1 商品検索テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-SEARCH-001 | キーワードで商品検索 | 商品データが存在 | 1. 検索画面を開く 2. 検索ボックスに「kimono」と入力 3. 検索ボタンをタップ | 「kimono」を含む商品一覧が表示される | Critical | Yes |
| TC-SEARCH-002 | カテゴリフィルターで検索 | 商品データが存在 | 1. 検索画面を開く 2. カテゴリフィルター「着物」を選択 3. 検索実行 | 着物カテゴリの商品のみが表示される | High | Yes |
| TC-SEARCH-003 | 価格範囲フィルターで検索 | 商品データが存在 | 1. 検索画面を開く 2. 価格フィルターを「5,000円〜10,000円」に設定 3. 検索実行 | 指定価格範囲内の商品のみが表示される | High | Yes |
| TC-SEARCH-004 | 複合条件で検索 | 商品データが存在 | 1. 検索画面を開く 2. キーワード「着物」、カテゴリ「レンタル」、価格「〜20,000円」を設定 3. 検索実行 | 全条件を満たす商品のみが表示される | High | Yes |
| TC-SEARCH-005 | 検索結果のソート（価格昇順） | 検索結果が存在 | 1. 検索を実行 2. ソートオプションで「価格：低→高」を選択 | 検索結果が価格の昇順で並び替えられる | Medium | Yes |
| TC-SEARCH-006 | 検索結果のソート（価格降順） | 検索結果が存在 | 1. 検索を実行 2. ソートオプションで「価格：高→低」を選択 | 検索結果が価格の降順で並び替えられる | Medium | Yes |
| TC-SEARCH-007 | 検索結果のソート（人気順） | 検索結果が存在 | 1. 検索を実行 2. ソートオプションで「人気順」を選択 | 検索結果が人気スコア順で並び替えられる | Medium | Yes |
| TC-SEARCH-008 | 検索結果なしの場合の表示 | - | 1. 検索画面を開く 2. 存在しないキーワード「xyzabc123」で検索 | 「検索結果が見つかりませんでした」メッセージと提案が表示される | High | Yes |
| TC-SEARCH-009 | 検索履歴の表示 | 過去に検索を実行 | 1. 検索画面を開く 2. 検索ボックスをタップ | 最近の検索履歴が表示される | Low | Yes |
| TC-SEARCH-010 | 検索候補（サジェスト）の表示 | 商品データが存在 | 1. 検索画面を開く 2. 「ki」と入力 | 「kimono」等の候補が表示される | Medium | Yes |

#### 2.2.2 ホテル検索テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-SEARCH-020 | 場所でホテル検索 | ホテルデータが存在 | 1. ホテル検索画面を開く 2. 場所に「Tokyo」を入力 3. 検索実行 | 東京のホテル一覧が表示される | Critical | Yes |
| TC-SEARCH-021 | 日付指定でホテル検索 | ホテルデータが存在 | 1. ホテル検索画面を開く 2. チェックイン日・チェックアウト日を選択 3. 検索実行 | 指定日程で空室のあるホテルが表示される | Critical | Yes |
| TC-SEARCH-022 | 宿泊人数フィルター | ホテルデータが存在 | 1. ホテル検索画面を開く 2. 宿泊人数を「大人2名、子供1名」に設定 3. 検索実行 | 指定人数を収容可能な部屋があるホテルが表示される | High | Yes |
| TC-SEARCH-023 | ホテル設備フィルター | ホテルデータが存在 | 1. ホテル検索画面を開く 2. フィルター「WiFi」「温泉」を選択 3. 検索実行 | 選択した設備を持つホテルのみが表示される | Medium | Yes |
| TC-SEARCH-024 | 星評価フィルター | ホテルデータが存在 | 1. ホテル検索画面を開く 2. 「4つ星以上」フィルターを選択 3. 検索実行 | 4つ星以上のホテルのみが表示される | Medium | Yes |
| TC-SEARCH-025 | 地図表示での検索 | ホテルデータが存在、位置情報許可 | 1. ホテル検索画面を開く 2. 地図表示モードに切り替え 3. 地図上でエリアを選択 | 選択エリア内のホテルがピンで表示される | Medium | No |

#### 2.2.3 アトラクション検索テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-SEARCH-040 | カテゴリでアトラクション検索 | アトラクションデータが存在 | 1. アトラクション画面を開く 2. カテゴリ「美術館」を選択 | 美術館カテゴリのアトラクションが表示される | High | Yes |
| TC-SEARCH-041 | エリアでアトラクション検索 | アトラクションデータが存在 | 1. アトラクション画面を開く 2. エリア「東京」を選択 | 東京エリアのアトラクションが表示される | High | Yes |
| TC-SEARCH-042 | 日付指定でアトラクション検索 | アトラクションデータが存在 | 1. アトラクション画面を開く 2. 訪問日を選択 3. 検索実行 | 指定日に利用可能なアトラクションが表示される | High | Yes |

### 2.3 予約フロー

#### 2.3.1 商品予約（カート追加）テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-BOOKING-001 | 商品をカートに追加 | ログイン済み、商品詳細画面表示 | 1. 商品詳細画面を開く 2. サイズ「M」を選択 3. 数量「2」を選択 4. 「カートに追加」ボタンをタップ | 商品がカートに追加され、カートバッジが更新される | Critical | Yes |
| TC-BOOKING-002 | 在庫切れ商品の追加試行 | 在庫0の商品が存在 | 1. 在庫切れ商品の詳細画面を開く | 「在庫切れ」と表示され、カートに追加ボタンが無効化される | High | Yes |
| TC-BOOKING-003 | 在庫以上の数量追加試行 | 在庫3の商品が存在 | 1. 商品詳細画面を開く 2. 数量「5」を選択 3. 「カートに追加」ボタンをタップ | 「在庫が不足しています（残り3点）」エラーが表示される | High | Yes |
| TC-BOOKING-004 | 同一商品の追加（数量増加） | カートに商品Aが1個存在 | 1. 商品Aの詳細画面を開く 2. 数量「2」で追加 | カート内の商品Aの数量が3に更新される | High | Yes |
| TC-BOOKING-005 | カート内商品の数量変更 | カートに商品が存在 | 1. カート画面を開く 2. 商品の数量を「3」に変更 | 数量が更新され、合計金額が再計算される | High | Yes |
| TC-BOOKING-006 | カートから商品を削除 | カートに商品が存在 | 1. カート画面を開く 2. 商品の削除ボタンをタップ 3. 確認ダイアログで「はい」を選択 | 商品がカートから削除され、合計金額が再計算される | High | Yes |
| TC-BOOKING-007 | カートの永続性確認 | カートに商品が存在 | 1. アプリを完全に終了 2. アプリを再起動 3. カート画面を開く | カート内容が保持されている | High | Yes |

#### 2.3.2 ホテル予約テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-BOOKING-020 | ホテル部屋の予約 | ログイン済み、ホテル詳細画面表示 | 1. ホテル詳細画面を開く 2. 部屋タイプ「ダブル」を選択 3. 日程を確認 4. 「予約する」ボタンをタップ | 予約情報入力画面に遷移する | Critical | Yes |
| TC-BOOKING-021 | 満室日程での予約試行 | 満室の日程が存在 | 1. ホテル詳細画面を開く 2. 満室の日程を選択 | 「この日程は満室です」と表示され、予約ボタンが無効化される | High | Yes |
| TC-BOOKING-022 | 予約情報入力と確認 | 予約情報入力画面表示 | 1. 宿泊者名を入力 2. 連絡先を入力 3. 特別リクエストを入力 4. 「確認画面へ」をタップ | 入力内容が確認画面に表示される | High | Yes |
| TC-BOOKING-023 | 必須項目未入力で予約試行 | 予約情報入力画面表示 | 1. 宿泊者名を空のまま 2. 「確認画面へ」をタップ | 必須項目エラーが表示される | High | Yes |

#### 2.3.3 アトラクションチケット予約テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-BOOKING-040 | アトラクションチケット購入 | ログイン済み、アトラクション詳細画面表示 | 1. アトラクション詳細画面を開く 2. 日付を選択 3. 時間帯を選択 4. チケット枚数「大人2枚」を選択 5. 「チケットを購入」をタップ | チケット購入確認画面に遷移する | Critical | Yes |
| TC-BOOKING-041 | 売り切れ時間帯での購入試行 | 売り切れの時間帯が存在 | 1. アトラクション詳細画面を開く 2. 売り切れの時間帯を選択しようとする | 「売り切れ」と表示され、選択できない | High | Yes |
| TC-BOOKING-042 | キャパシティ超過購入試行 | 残り2枚の時間帯が存在 | 1. アトラクション詳細画面を開く 2. 該当時間帯を選択 3. チケット枚数「5枚」を選択 | 「残り2枚です」エラーが表示される | High | Yes |

### 2.4 決済処理

#### 2.4.1 決済テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-PAYMENT-001 | クレジットカードで決済成功 | チェックアウト画面表示、有効なカード情報 | 1. 決済方法「クレジットカード」を選択 2. カード番号「4242424242424242」を入力 3. 有効期限「12/28」を入力 4. CVC「123」を入力 5. 「支払う」ボタンをタップ | 決済成功、注文確認画面に遷移する | Critical | Yes |
| TC-PAYMENT-002 | 無効なカード番号で決済試行 | チェックアウト画面表示 | 1. 決済方法「クレジットカード」を選択 2. 無効なカード番号「1234567890123456」を入力 3. 「支払う」ボタンをタップ | 「無効なカード番号です」エラーが表示される | Critical | Yes |
| TC-PAYMENT-003 | 期限切れカードで決済試行 | チェックアウト画面表示 | 1. 決済方法「クレジットカード」を選択 2. カード番号を入力 3. 期限切れの有効期限「01/20」を入力 4. 「支払う」ボタンをタップ | 「カードの有効期限が切れています」エラーが表示される | High | Yes |
| TC-PAYMENT-004 | 残高不足カードで決済試行 | チェックアウト画面表示 | 1. テスト用残高不足カード「4000000000000002」を使用 2. 「支払う」ボタンをタップ | 「決済が拒否されました」エラーが表示される | High | Yes |
| TC-PAYMENT-005 | 決済処理中のタイムアウト | チェックアウト画面表示、ネットワーク遅延 | 1. 決済情報を入力 2. 「支払う」ボタンをタップ 3. 30秒以上応答なし | タイムアウトエラーが表示され、再試行オプションが提示される | High | No |
| TC-PAYMENT-006 | 決済キャンセル | 決済画面表示 | 1. 決済情報入力中に「戻る」ボタンをタップ 2. 確認ダイアログで「はい」を選択 | カート画面に戻り、カート内容は保持される | Medium | Yes |
| TC-PAYMENT-007 | 保存済みカードで決済 | 保存済みカードが存在 | 1. チェックアウト画面を開く 2. 保存済みカードを選択 3. CVCを入力 4. 「支払う」ボタンをタップ | 決済成功、注文確認画面に遷移する | High | Yes |

#### 2.4.2 注文確認テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-PAYMENT-020 | 注文確認画面の表示確認 | 決済成功後 | 1. 決済完了後の注文確認画面を確認 | 注文番号、注文内容、合計金額、配送先が正しく表示される | Critical | Yes |
| TC-PAYMENT-021 | 注文確認メールの受信 | 決済成功後 | 1. 登録メールアドレスの受信箱を確認 | 注文確認メールが受信され、注文詳細が記載されている | High | No |
| TC-PAYMENT-022 | 注文履歴への反映 | 決済成功後 | 1. ユーザー画面を開く 2. 注文履歴を開く | 新しい注文が履歴に表示される | High | Yes |

### 2.5 旅程管理

#### 2.5.1 旅程テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-ITINERARY-001 | 予約済みアイテムの旅程表示 | ホテル・チケット予約済み | 1. 旅程画面を開く | 予約済みのホテル、チケットが日付順に表示される | High | Yes |
| TC-ITINERARY-002 | 旅程の日付別表示 | 複数日程の予約が存在 | 1. 旅程画面を開く 2. カレンダー表示に切り替え | 各日付に予約アイテムがマッピングされて表示される | Medium | Yes |
| TC-ITINERARY-003 | 予約詳細の表示 | 旅程にアイテムが存在 | 1. 旅程画面を開く 2. 予約アイテムをタップ | 予約の詳細情報（確認番号、日時、場所等）が表示される | High | Yes |
| TC-ITINERARY-004 | QRコードの表示 | チケット予約済み | 1. 旅程画面を開く 2. チケットアイテムをタップ 3. 「QRコードを表示」をタップ | 入場用QRコードが表示される | Critical | Yes |

### 2.6 通知・コミュニケーション

#### 2.6.1 通知テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-NOTIF-001 | 注文確定通知の受信 | プッシュ通知許可済み、注文完了 | 1. 注文を完了する | 「ご注文が確定しました」プッシュ通知が表示される | High | No |
| TC-NOTIF-002 | 予約リマインダー通知 | 翌日に予約が存在 | 1. 予約日の前日になるのを待つ | 「明日の予約があります」プッシュ通知が表示される | Medium | No |
| TC-NOTIF-003 | 通知設定の変更 | ログイン済み | 1. 設定画面を開く 2. 通知設定を開く 3. 「プロモーション通知」をOFFに変更 | 設定が保存され、プロモーション通知が停止される | Medium | Yes |
| TC-NOTIF-004 | アプリ内通知一覧の表示 | 通知が存在 | 1. 通知アイコンをタップ | 通知一覧が新しい順に表示される | Medium | Yes |
| TC-NOTIF-005 | 通知からの画面遷移 | 注文通知が存在 | 1. 通知一覧を開く 2. 注文通知をタップ | 該当する注文詳細画面に遷移する | Medium | Yes |

---

## 第3章：E2Eテストシナリオ

### 3.1 ユーザー登録→初回予約完了シナリオ

#### 3.1.1 シナリオ概要

```yaml
scenario_id: E2E-001
scenario_name: 新規ユーザーの初回購入完了
priority: P0 (Critical)
estimated_duration: 5-7分
automation_status: Automated

user_journey:
  persona: 初めてTripTripを使用する観光客
  goal: 着物をレンタルして購入完了まで進める

preconditions:
  - アプリが初期状態（データクリア済み）
  - テスト用商品データが存在
  - テスト用決済環境（サンドボックス）が利用可能
```

#### 3.1.2 詳細テストステップ

| ステップ | アクション | 検証ポイント |
|---------|----------|-------------|
| 1 | アプリを起動 | スプラッシュ画面が表示され、ホーム画面に遷移する |
| 2 | 「アカウント作成」ボタンをタップ | 登録画面が表示される |
| 3 | メールアドレス「e2e-test-{timestamp}@example.com」を入力 | メールアドレスが入力される |
| 4 | パスワード「TestPass123!」を入力 | パスワードがマスク表示で入力される |
| 5 | 確認パスワードを入力 | パスワードが一致していることが確認される |
| 6 | 「登録」ボタンをタップ | ローディング表示後、ホーム画面に遷移する |
| 7 | ホーム画面でユーザー名が表示されることを確認 | ログイン状態が反映されている |
| 8 | 「着物」カテゴリをタップ | 着物一覧画面に遷移する |
| 9 | 最初の着物商品をタップ | 商品詳細画面が表示される |
| 10 | サイズ「M」を選択 | サイズが選択状態になる |
| 11 | 「カートに追加」ボタンをタップ | カートに追加成功メッセージが表示される |
| 12 | カートアイコンをタップ | カート画面に遷移し、追加した商品が表示される |
| 13 | 「チェックアウト」ボタンをタップ | チェックアウト画面に遷移する |
| 14 | 配送先住所を入力（名前、住所、電話番号） | 入力が完了する |
| 15 | 「決済に進む」ボタンをタップ | 決済画面に遷移する |
| 16 | テストカード情報を入力（4242424242424242） | カード情報が入力される |
| 17 | 「支払う」ボタンをタップ | ローディング表示後、注文確認画面に遷移する |
| 18 | 注文番号が表示されることを確認 | 注文が正常に完了している |
| 19 | 「注文履歴を見る」をタップ | 注文履歴画面に遷移する |
| 20 | 新しい注文が表示されることを確認 | 注文履歴に反映されている |

#### 3.1.3 テストコード例

```dart
// integration_test/scenarios/e2e_001_first_purchase_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:triptrip/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('E2E-001: New User First Purchase', () {
    testWidgets('complete first purchase flow', (tester) async {
      // Step 1: Launch app
      app.main();
      await tester.pumpAndSettle();

      // Verify home screen
      expect(find.byKey(const Key('home_screen')), findsOneWidget);

      // Step 2-6: User Registration
      await tester.tap(find.text('アカウント作成'));
      await tester.pumpAndSettle();

      final timestamp = DateTime.now().millisecondsSinceEpoch;
      await tester.enterText(
        find.byKey(const Key('email_input')),
        'e2e-test-$timestamp@example.com',
      );
      await tester.enterText(
        find.byKey(const Key('password_input')),
        'TestPass123!',
      );
      await tester.enterText(
        find.byKey(const Key('confirm_password_input')),
        'TestPass123!',
      );
      await tester.tap(find.byKey(const Key('register_button')));
      await tester.pumpAndSettle(const Duration(seconds: 3));

      // Step 7: Verify logged in state
      expect(find.byKey(const Key('home_screen')), findsOneWidget);

      // Step 8-11: Add product to cart
      await tester.tap(find.text('着物'));
      await tester.pumpAndSettle();

      await tester.tap(find.byType(ProductCard).first);
      await tester.pumpAndSettle();

      await tester.tap(find.text('M'));
      await tester.pumpAndSettle();

      await tester.tap(find.text('カートに追加'));
      await tester.pumpAndSettle();

      // Step 12-13: Proceed to checkout
      await tester.tap(find.byKey(const Key('cart_icon')));
      await tester.pumpAndSettle();

      expect(find.byType(CartItemWidget), findsOneWidget);

      await tester.tap(find.text('チェックアウト'));
      await tester.pumpAndSettle();

      // Step 14-15: Enter shipping info
      await tester.enterText(find.byKey(const Key('shipping_name')), 'テストユーザー');
      await tester.enterText(find.byKey(const Key('shipping_address')), '東京都渋谷区1-1-1');
      await tester.enterText(find.byKey(const Key('shipping_phone')), '090-1234-5678');
      await tester.tap(find.text('決済に進む'));
      await tester.pumpAndSettle();

      // Step 16-17: Payment
      await tester.enterText(find.byKey(const Key('card_number')), '4242424242424242');
      await tester.enterText(find.byKey(const Key('card_expiry')), '12/28');
      await tester.enterText(find.byKey(const Key('card_cvc')), '123');
      await tester.tap(find.text('支払う'));
      await tester.pumpAndSettle(const Duration(seconds: 5));

      // Step 18-20: Verify order completion
      expect(find.byKey(const Key('order_confirmation')), findsOneWidget);
      expect(find.byKey(const Key('order_number')), findsOneWidget);

      await tester.tap(find.text('注文履歴を見る'));
      await tester.pumpAndSettle();

      expect(find.byType(OrderHistoryItem), findsWidgets);
    });
  });
}
```

### 3.2 複数アイテム同時予約シナリオ

#### 3.2.1 シナリオ概要

```yaml
scenario_id: E2E-002
scenario_name: 複数アイテムの同時予約（ホテル+チケット+商品）
priority: P0 (Critical)
estimated_duration: 8-10分
automation_status: Automated

user_journey:
  persona: 日本旅行を計画中の観光客
  goal: ホテル、アトラクションチケット、お土産を一度に予約

preconditions:
  - ログイン済みユーザー
  - 空室のあるホテルデータ
  - 利用可能なアトラクションチケット
  - 在庫のある商品
```

#### 3.2.2 詳細テストステップ

| ステップ | アクション | 検証ポイント |
|---------|----------|-------------|
| 1 | ログイン済み状態でホーム画面を表示 | ホーム画面が表示される |
| 2 | 「ホテル」カードをタップ | ホテル検索画面に遷移する |
| 3 | 場所「Tokyo」、日程「3日後〜5日後」を入力 | 検索条件が設定される |
| 4 | 検索を実行 | ホテル一覧が表示される |
| 5 | 最初のホテルを選択 | ホテル詳細画面が表示される |
| 6 | 部屋タイプを選択し「予約する」をタップ | 予約情報入力画面に遷移する |
| 7 | 宿泊者情報を入力し「カートに追加」をタップ | ホテル予約がカートに追加される |
| 8 | ホーム画面に戻り「アトラクション」をタップ | アトラクション一覧画面に遷移する |
| 9 | アトラクションを選択し、日付と時間帯を選択 | チケット選択画面が表示される |
| 10 | チケット「大人2枚」を選択し「カートに追加」をタップ | チケットがカートに追加される |
| 11 | ホーム画面に戻り「商品」をタップ | 商品一覧画面に遷移する |
| 12 | 商品を選択し「カートに追加」をタップ | 商品がカートに追加される |
| 13 | カート画面を開く | 3種類のアイテム（ホテル、チケット、商品）が表示される |
| 14 | 合計金額を確認 | 全アイテムの合計が正しく計算されている |
| 15 | 「チェックアウト」をタップ | チェックアウト画面に遷移する |
| 16 | 配送先・決済情報を入力し「支払う」をタップ | 決済処理が実行される |
| 17 | 注文確認画面を確認 | 全アイテムの予約が確定している |
| 18 | 旅程画面を開く | 予約したホテルとチケットが表示される |

### 3.3 予約変更・キャンセルフローシナリオ

#### 3.3.1 シナリオ概要

```yaml
scenario_id: E2E-003
scenario_name: 予約の変更とキャンセル処理
priority: P1 (High)
estimated_duration: 6-8分
automation_status: Partially Automated

user_journey:
  persona: 予定変更が必要になったユーザー
  goal: 既存の予約を変更またはキャンセルする

preconditions:
  - ログイン済みユーザー
  - 変更可能な予約が存在
  - キャンセル可能期間内
```

#### 3.3.2 詳細テストステップ

| ステップ | アクション | 検証ポイント |
|---------|----------|-------------|
| 1 | 注文履歴画面を開く | 予約一覧が表示される |
| 2 | 変更対象の予約をタップ | 予約詳細画面が表示される |
| 3 | 「予約を変更」ボタンをタップ | 変更オプションが表示される |
| 4 | 日程を変更（元の日程+2日） | 新しい日程が選択される |
| 5 | 「変更を確定」ボタンをタップ | 変更確認ダイアログが表示される |
| 6 | 「はい」を選択 | 変更処理が実行される |
| 7 | 変更完了メッセージを確認 | 「予約が変更されました」メッセージが表示される |
| 8 | 予約詳細を確認 | 新しい日程が反映されている |
| 9 | 別の予約の詳細画面を開く | 予約詳細が表示される |
| 10 | 「予約をキャンセル」ボタンをタップ | キャンセル確認ダイアログが表示される |
| 11 | キャンセル理由を選択 | 理由が選択される |
| 12 | 「キャンセルを確定」ボタンをタップ | キャンセル処理が実行される |
| 13 | キャンセル完了メッセージを確認 | 「予約がキャンセルされました」メッセージが表示される |
| 14 | 返金情報を確認 | 返金額と返金予定日が表示される |
| 15 | 注文履歴を確認 | キャンセルされた予約のステータスが「キャンセル済み」になっている |

---

## 第4章：APIテストケース

### 4.1 REST API エンドポイント別テスト

#### 4.1.1 認証API（/api/v1/auth）

| ID | エンドポイント | メソッド | テストケース | リクエスト | 期待レスポンス | 優先度 |
|----|---------------|---------|-------------|-----------|---------------|--------|
| TC-API-AUTH-001 | /api/v1/auth/register | POST | 正常登録 | `{"email":"test@example.com","password":"Test123!","name":"Test User"}` | 201: `{"id":"...","email":"...","token":"..."}` | Critical |
| TC-API-AUTH-002 | /api/v1/auth/register | POST | 重複メール | 既存メールで登録 | 409: `{"error":"Email already exists"}` | High |
| TC-API-AUTH-003 | /api/v1/auth/register | POST | 無効なメール形式 | `{"email":"invalid","password":"Test123!"}` | 400: `{"error":"Invalid email format"}` | High |
| TC-API-AUTH-004 | /api/v1/auth/register | POST | 弱いパスワード | `{"email":"test@example.com","password":"123"}` | 400: `{"error":"Password too weak"}` | High |
| TC-API-AUTH-005 | /api/v1/auth/login | POST | 正常ログイン | `{"email":"test@example.com","password":"Test123!"}` | 200: `{"token":"...","refreshToken":"...","user":{...}}` | Critical |
| TC-API-AUTH-006 | /api/v1/auth/login | POST | 不正なパスワード | `{"email":"test@example.com","password":"wrong"}` | 401: `{"error":"Invalid credentials"}` | Critical |
| TC-API-AUTH-007 | /api/v1/auth/login | POST | 存在しないユーザー | `{"email":"notfound@example.com","password":"Test123!"}` | 401: `{"error":"Invalid credentials"}` | High |
| TC-API-AUTH-008 | /api/v1/auth/refresh | POST | トークン更新 | `{"refreshToken":"valid_token"}` | 200: `{"token":"new_token","refreshToken":"new_refresh"}` | High |
| TC-API-AUTH-009 | /api/v1/auth/refresh | POST | 無効なリフレッシュトークン | `{"refreshToken":"invalid"}` | 401: `{"error":"Invalid refresh token"}` | High |
| TC-API-AUTH-010 | /api/v1/auth/logout | POST | ログアウト | Header: `Authorization: Bearer token` | 204: No Content | High |

#### 4.1.2 ユーザーAPI（/api/v1/users）

| ID | エンドポイント | メソッド | テストケース | リクエスト | 期待レスポンス | 優先度 |
|----|---------------|---------|-------------|-----------|---------------|--------|
| TC-API-USER-001 | /api/v1/users/me | GET | 自分の情報取得 | Header: `Authorization: Bearer token` | 200: `{"id":"...","email":"...","name":"...","profile":{...}}` | High |
| TC-API-USER-002 | /api/v1/users/me | GET | 未認証アクセス | ヘッダーなし | 401: `{"error":"Unauthorized"}` | High |
| TC-API-USER-003 | /api/v1/users/me | PATCH | プロフィール更新 | `{"name":"New Name","phone":"090-1234-5678"}` | 200: 更新後のユーザー情報 | High |
| TC-API-USER-004 | /api/v1/users/me | PATCH | 無効な電話番号形式 | `{"phone":"invalid"}` | 400: `{"error":"Invalid phone format"}` | Medium |
| TC-API-USER-005 | /api/v1/users/me/password | PUT | パスワード変更 | `{"currentPassword":"old","newPassword":"New123!"}` | 200: `{"message":"Password updated"}` | High |
| TC-API-USER-006 | /api/v1/users/me/password | PUT | 現在のパスワード不一致 | `{"currentPassword":"wrong","newPassword":"New123!"}` | 400: `{"error":"Current password incorrect"}` | High |

#### 4.1.3 商品API（/api/v1/products）

| ID | エンドポイント | メソッド | テストケース | リクエスト | 期待レスポンス | 優先度 |
|----|---------------|---------|-------------|-----------|---------------|--------|
| TC-API-PROD-001 | /api/v1/products | GET | 商品一覧取得 | `?page=1&limit=20` | 200: `{"products":[...],"pagination":{...}}` | Critical |
| TC-API-PROD-002 | /api/v1/products | GET | カテゴリフィルター | `?category=kimono` | 200: 着物カテゴリの商品のみ | High |
| TC-API-PROD-003 | /api/v1/products | GET | 価格範囲フィルター | `?minPrice=5000&maxPrice=10000` | 200: 価格範囲内の商品 | High |
| TC-API-PROD-004 | /api/v1/products | GET | キーワード検索 | `?q=kimono` | 200: キーワードに一致する商品 | High |
| TC-API-PROD-005 | /api/v1/products | GET | ソート（価格昇順） | `?sort=price&order=asc` | 200: 価格昇順でソートされた商品 | Medium |
| TC-API-PROD-006 | /api/v1/products | GET | ページネーション境界 | `?page=9999` | 200: 空の配列（データなし） | Medium |
| TC-API-PROD-007 | /api/v1/products/{id} | GET | 商品詳細取得 | - | 200: 商品詳細情報 | Critical |
| TC-API-PROD-008 | /api/v1/products/{id} | GET | 存在しない商品ID | - | 404: `{"error":"Product not found"}` | High |
| TC-API-PROD-009 | /api/v1/products/{id}/stock | GET | 在庫確認 | - | 200: `{"stock":10,"available":true}` | High |

#### 4.1.4 注文API（/api/v1/orders）

| ID | エンドポイント | メソッド | テストケース | リクエスト | 期待レスポンス | 優先度 |
|----|---------------|---------|-------------|-----------|---------------|--------|
| TC-API-ORD-001 | /api/v1/orders | POST | 注文作成 | `{"items":[{"productId":"...","quantity":2}],"shippingAddress":{...}}` | 201: 作成された注文情報 | Critical |
| TC-API-ORD-002 | /api/v1/orders | POST | 空のアイテムで注文 | `{"items":[],"shippingAddress":{...}}` | 400: `{"error":"Order must have items"}` | High |
| TC-API-ORD-003 | /api/v1/orders | POST | 在庫超過注文 | quantity > 在庫数 | 400: `{"error":"Insufficient stock"}` | High |
| TC-API-ORD-004 | /api/v1/orders | GET | 注文一覧取得 | Header: Bearer token | 200: ユーザーの注文一覧 | High |
| TC-API-ORD-005 | /api/v1/orders/{id} | GET | 注文詳細取得 | - | 200: 注文詳細情報 | High |
| TC-API-ORD-006 | /api/v1/orders/{id} | GET | 他ユーザーの注文アクセス | - | 403: `{"error":"Forbidden"}` | Critical |
| TC-API-ORD-007 | /api/v1/orders/{id}/cancel | POST | 注文キャンセル | - | 200: `{"status":"cancelled","refund":{...}}` | High |
| TC-API-ORD-008 | /api/v1/orders/{id}/cancel | POST | キャンセル期限超過 | - | 400: `{"error":"Cancellation period expired"}` | High |

### 4.2 認証・認可テスト

#### 4.2.1 認証テストケース

| ID | テストケース | リクエスト | 期待結果 | 優先度 |
|----|------------|-----------|----------|--------|
| TC-API-AUTHZ-001 | 有効なJWTトークンでアクセス | `Authorization: Bearer valid_token` | 200: 正常レスポンス | Critical |
| TC-API-AUTHZ-002 | 期限切れJWTトークンでアクセス | `Authorization: Bearer expired_token` | 401: `{"error":"Token expired"}` | Critical |
| TC-API-AUTHZ-003 | 改ざんされたJWTトークン | `Authorization: Bearer tampered_token` | 401: `{"error":"Invalid token"}` | Critical |
| TC-API-AUTHZ-004 | Authorizationヘッダーなし | ヘッダーなし | 401: `{"error":"Unauthorized"}` | Critical |
| TC-API-AUTHZ-005 | 不正な形式のヘッダー | `Authorization: invalid` | 401: `{"error":"Invalid authorization header"}` | High |
| TC-API-AUTHZ-006 | 無効化されたトークン（ログアウト後） | ログアウト後のトークン | 401: `{"error":"Token revoked"}` | High |

#### 4.2.2 認可テストケース

| ID | テストケース | ユーザーロール | リソース | 期待結果 | 優先度 |
|----|------------|--------------|---------|----------|--------|
| TC-API-AUTHZ-010 | 一般ユーザーが自分の注文を取得 | user | /orders/{own_id} | 200: 注文詳細 | Critical |
| TC-API-AUTHZ-011 | 一般ユーザーが他人の注文を取得 | user | /orders/{other_id} | 403: Forbidden | Critical |
| TC-API-AUTHZ-012 | 管理者が任意の注文を取得 | admin | /orders/{any_id} | 200: 注文詳細 | High |
| TC-API-AUTHZ-013 | 一般ユーザーが管理者APIにアクセス | user | /admin/... | 403: Forbidden | Critical |
| TC-API-AUTHZ-014 | 未認証ユーザーが保護リソースにアクセス | guest | /users/me | 401: Unauthorized | Critical |

### 4.3 エラーハンドリングテスト

#### 4.3.1 エラーレスポンステストケース

| ID | テストケース | トリガー | 期待レスポンス | 優先度 |
|----|------------|---------|---------------|--------|
| TC-API-ERR-001 | 不正なJSON形式 | 壊れたJSONボディ | 400: `{"error":"Invalid JSON"}` | High |
| TC-API-ERR-002 | 必須フィールド欠落 | 必須フィールドなしでPOST | 422: `{"error":"Validation failed","details":[...]}` | High |
| TC-API-ERR-003 | データ型不一致 | 数値フィールドに文字列 | 422: `{"error":"Validation failed"}` | High |
| TC-API-ERR-004 | 存在しないエンドポイント | GET /api/v1/notfound | 404: `{"error":"Not found"}` | Medium |
| TC-API-ERR-005 | 許可されていないメソッド | DELETE /api/v1/products | 405: `{"error":"Method not allowed"}` | Medium |
| TC-API-ERR-006 | レート制限超過 | 100リクエスト/分超過 | 429: `{"error":"Too many requests","retryAfter":60}` | High |
| TC-API-ERR-007 | サーバー内部エラー | 強制的にエラーを発生 | 500: `{"error":"Internal server error","requestId":"..."}` | High |

---

## 第5章：非機能テスト

### 5.1 パフォーマンステスト

#### 5.1.1 レスポンスタイムテストケース

| ID | エンドポイント | 目標値 | 負荷条件 | 優先度 |
|----|--------------|--------|---------|--------|
| TC-PERF-001 | GET /api/v1/products | p95 < 200ms | 100同時ユーザー | Critical |
| TC-PERF-002 | GET /api/v1/products/{id} | p95 < 100ms | 100同時ユーザー | Critical |
| TC-PERF-003 | GET /api/v1/hotels | p95 < 300ms | 100同時ユーザー | Critical |
| TC-PERF-004 | POST /api/v1/orders | p95 < 500ms | 50同時ユーザー | Critical |
| TC-PERF-005 | POST /api/v1/auth/login | p95 < 200ms | 100同時ユーザー | High |
| TC-PERF-006 | 検索API（複合条件） | p95 < 400ms | 50同時ユーザー | High |

#### 5.1.2 スループットテストケース

| ID | シナリオ | 目標値 | 期間 | 優先度 |
|----|---------|--------|------|--------|
| TC-PERF-010 | 通常負荷テスト | 1000 RPS | 10分間 | Critical |
| TC-PERF-011 | ピーク負荷テスト | 2000 RPS | 5分間 | High |
| TC-PERF-012 | 持続負荷テスト | 500 RPS | 4時間 | High |

#### 5.1.3 負荷テストシナリオ

```yaml
load_test_scenarios:
  scenario_1_normal_load:
    name: 通常負荷テスト
    vus: 100
    duration: 10m
    ramp_up: 2m
    expected:
      error_rate: < 0.1%
      p95_latency: < 200ms
      throughput: > 1000 rps

  scenario_2_stress_test:
    name: ストレステスト
    stages:
      - duration: 5m
        target: 100
      - duration: 5m
        target: 300
      - duration: 5m
        target: 500
      - duration: 5m
        target: 100
    expected:
      breaking_point_identification: true
      graceful_degradation: true

  scenario_3_spike_test:
    name: スパイクテスト
    stages:
      - duration: 1m
        target: 100
      - duration: 10s
        target: 1000
      - duration: 1m
        target: 100
    expected:
      recovery_time: < 30s
      no_data_loss: true

  scenario_4_soak_test:
    name: ソークテスト（耐久テスト）
    vus: 50
    duration: 4h
    expected:
      memory_leak: false
      performance_degradation: < 10%
```

### 5.2 セキュリティテスト

#### 5.2.1 OWASP Top 10対応テストケース

| ID | 脆弱性カテゴリ | テストケース | テスト方法 | 優先度 |
|----|--------------|-------------|-----------|--------|
| TC-SEC-001 | A01:2021 アクセス制御の不備 | 水平権限昇格（IDOR） | 他ユーザーのリソースIDでアクセス試行 | Critical |
| TC-SEC-002 | A01:2021 アクセス制御の不備 | 垂直権限昇格 | 一般ユーザーで管理者APIアクセス試行 | Critical |
| TC-SEC-003 | A02:2021 暗号化の失敗 | 平文パスワード保存確認 | DB直接確認（ハッシュ化検証） | Critical |
| TC-SEC-004 | A02:2021 暗号化の失敗 | HTTPS強制確認 | HTTPリクエストのリダイレクト確認 | Critical |
| TC-SEC-005 | A03:2021 インジェクション | SQLインジェクション | 入力フィールドにSQLペイロード投入 | Critical |
| TC-SEC-006 | A03:2021 インジェクション | NoSQLインジェクション | MongoDBオペレーター投入 | High |
| TC-SEC-007 | A03:2021 インジェクション | XSS（反射型） | スクリプトタグを含む入力 | Critical |
| TC-SEC-008 | A03:2021 インジェクション | XSS（格納型） | プロフィールにスクリプト保存試行 | Critical |
| TC-SEC-009 | A04:2021 安全でない設計 | レート制限バイパス | 異なるIPからの大量リクエスト | High |
| TC-SEC-010 | A05:2021 セキュリティ設定ミス | デフォルト認証情報 | デフォルトadmin/adminでログイン試行 | High |
| TC-SEC-011 | A05:2021 セキュリティ設定ミス | エラーメッセージ情報漏洩 | スタックトレース露出確認 | High |
| TC-SEC-012 | A06:2021 脆弱なコンポーネント | 依存関係脆弱性 | npm audit / Snyk実行 | High |
| TC-SEC-013 | A07:2021 認証の不備 | ブルートフォース攻撃 | 連続ログイン失敗でのロック確認 | Critical |
| TC-SEC-014 | A07:2021 認証の不備 | セッション固定 | ログイン前後のセッションID変更確認 | High |
| TC-SEC-015 | A08:2021 ソフトウェアとデータの整合性 | JWTアルゴリズム変更攻撃 | alg:noneでのトークン偽造 | Critical |
| TC-SEC-016 | A09:2021 セキュリティログと監視の不備 | 認証失敗ログ確認 | ログイン失敗時のログ出力確認 | Medium |
| TC-SEC-017 | A10:2021 SSRF | 内部リソースアクセス | URLパラメータに内部IPを指定 | High |

#### 5.2.2 セキュリティテスト詳細

```yaml
security_test_details:
  sql_injection_tests:
    payloads:
      - "' OR '1'='1"
      - "'; DROP TABLE users;--"
      - "' UNION SELECT * FROM users--"
    target_fields:
      - 検索クエリ
      - ログインフォーム
      - フィルターパラメータ
    expected_result: 全てのペイロードが無害化される

  xss_tests:
    payloads:
      - "<script>alert('XSS')</script>"
      - "<img src=x onerror=alert('XSS')>"
      - "javascript:alert('XSS')"
    target_fields:
      - ユーザー名
      - コメント
      - 検索クエリ
    expected_result: スクリプトが実行されない（エスケープまたはサニタイズ）

  authentication_tests:
    brute_force:
      attempts: 10
      expected: アカウントロックまたはCAPTCHA表示

    session_management:
      - ログアウト後のセッション無効化確認
      - 同時ログイン制限確認
      - セッションタイムアウト確認（30分）

  authorization_tests:
    idor_targets:
      - /api/v1/orders/{id}
      - /api/v1/users/{id}
      - /api/v1/bookings/{id}
    method: IDを他ユーザーのものに置き換えてアクセス
    expected: 403 Forbidden
```

### 5.3 アクセシビリティテスト

#### 5.3.1 アクセシビリティテストケース

| ID | テストカテゴリ | テストケース | 基準 | 優先度 |
|----|--------------|-------------|------|--------|
| TC-A11Y-001 | 視覚 | コントラスト比確認 | WCAG 2.1 AA (4.5:1以上) | High |
| TC-A11Y-002 | 視覚 | テキストリサイズ対応 | 200%まで拡大可能 | High |
| TC-A11Y-003 | 視覚 | 色のみに依存しない情報伝達 | エラーは色+アイコン+テキスト | High |
| TC-A11Y-004 | 聴覚 | 音声コンテンツの代替 | 字幕/トランスクリプト | Medium |
| TC-A11Y-005 | 運動機能 | タッチターゲットサイズ | 44x44pt以上 | High |
| TC-A11Y-006 | 運動機能 | ジェスチャーの代替 | スワイプの代わりにボタン提供 | Medium |
| TC-A11Y-007 | 認知 | 一貫したナビゲーション | 全画面で同じ位置にナビ | Medium |
| TC-A11Y-008 | スクリーンリーダー | VoiceOver対応(iOS) | 全要素に適切なラベル | High |
| TC-A11Y-009 | スクリーンリーダー | TalkBack対応(Android) | 全要素に適切なラベル | High |
| TC-A11Y-010 | フォーカス | フォーカス順序の論理性 | 上から下、左から右の順序 | Medium |

---

## 第6章：モバイル固有テスト

### 6.1 オフラインモードテスト

| ID | テストケース | 前提条件 | テストステップ | 期待結果 | 優先度 |
|----|------------|----------|----------------|----------|--------|
| TC-MOB-001 | オフライン時のアプリ起動 | キャッシュデータあり、ネットワークOFF | 1. 機内モードON 2. アプリ起動 | キャッシュされたホーム画面が表示される | High |
| TC-MOB-002 | オフライン時の検索 | ネットワークOFF | 1. 検索を実行 | 「オフラインです」メッセージと前回の検索結果が表示される | High |
| TC-MOB-003 | オフライン時のカート操作 | カートにアイテムあり、ネットワークOFF | 1. カート画面を開く 2. 数量を変更 | ローカルで変更が保存され、オンライン復帰後に同期される | High |
| TC-MOB-004 | オフライン時の予約詳細表示 | 予約データがキャッシュ済み | 1. 機内モードON 2. 予約詳細を開く | キャッシュされた予約詳細とQRコードが表示される | Critical |
| TC-MOB-005 | オフライン→オンライン復帰 | オフラインで操作実行済み | 1. ネットワークをONに戻す | 保留中の操作が自動同期される | High |
| TC-MOB-006 | データ同期の競合解決 | オフラインで編集、サーバーでも更新 | 1. オフラインでカート更新 2. サーバー側でも更新 3. オンライン復帰 | 競合解決ダイアログが表示され、ユーザーが選択可能 | Medium |

### 6.2 デバイス互換性テスト

| ID | テストケース | デバイス/条件 | テスト項目 | 期待結果 | 優先度 |
|----|------------|--------------|-----------|----------|--------|
| TC-MOB-010 | 小画面デバイス表示 | iPhone SE (320pt幅) | 全画面のレイアウト確認 | 全要素が正しく表示される、はみ出しなし | High |
| TC-MOB-011 | 大画面デバイス表示 | iPad Pro 12.9" | 全画面のレイアウト確認 | タブレット向けレイアウトが適用される | Medium |
| TC-MOB-012 | 画面回転対応 | 横向き表示 | 主要画面の横向き表示 | レイアウトが崩れない | Medium |
| TC-MOB-013 | ノッチ/パンチホール対応 | iPhone 14 Pro (Dynamic Island) | 画面上部の表示確認 | コンテンツがノッチに隠れない | High |
| TC-MOB-014 | Android OSバージョン互換性 | Android 11, 12, 13, 14 | 主要機能の動作確認 | 全バージョンで正常動作 | High |
| TC-MOB-015 | iOS OSバージョン互換性 | iOS 15, 16, 17 | 主要機能の動作確認 | 全バージョンで正常動作 | High |
| TC-MOB-016 | ダークモード表示 | システムダークモード設定 | 全画面の表示確認 | ダークモードテーマが適用される | Medium |

### 6.3 ネットワーク切り替えテスト

| ID | テストケース | ネットワーク条件 | テストステップ | 期待結果 | 優先度 |
|----|------------|----------------|----------------|----------|--------|
| TC-MOB-020 | WiFi→モバイルデータ切り替え | WiFi接続中 | 1. APIリクエスト中にWiFiをOFF 2. モバイルデータに切り替わる | リクエストが継続または自動リトライされる | High |
| TC-MOB-021 | 低速ネットワーク(2G相当) | 帯域制限: 50Kbps | 1. 検索を実行 2. 画像を含む一覧を表示 | ローディング表示、画像の遅延読み込み、タイムアウトなし | High |
| TC-MOB-022 | 不安定ネットワーク | パケットロス: 30% | 1. 決済処理を実行 | リトライ処理が動作し、最終的に成功または明確なエラー | Critical |
| TC-MOB-023 | ネットワーク瞬断 | 5秒間切断 | 1. API通信中に瞬断発生 | 自動リトライで復旧、ユーザーへの影響なし | High |
| TC-MOB-024 | VPN接続時の動作 | VPN経由接続 | 1. 主要機能を操作 | VPN経由でも正常に動作する | Low |

### 6.4 バッテリー・リソーステスト

| ID | テストケース | テスト条件 | 測定項目 | 許容基準 | 優先度 |
|----|------------|-----------|---------|---------|--------|
| TC-MOB-030 | バッテリー消費量 | 30分間通常使用 | バッテリー消費率 | < 5% / 30分 | Medium |
| TC-MOB-031 | バックグラウンド消費 | 1時間バックグラウンド | バッテリー消費率 | < 1% / 時間 | Medium |
| TC-MOB-032 | メモリ使用量 | 通常使用時 | 最大メモリ使用量 | < 300MB | High |
| TC-MOB-033 | メモリリーク確認 | 30分間連続操作 | メモリ増加量 | 安定（単調増加なし） | High |
| TC-MOB-034 | ストレージ使用量 | 初期インストール時 | アプリサイズ | < 100MB | Low |
| TC-MOB-035 | キャッシュサイズ | 1週間使用後 | キャッシュサイズ | < 500MB、自動クリーンアップ | Low |

---

## 第7章：テスト実行計画 & リソース配分

### 7.1 テスト実行スケジュール

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    テスト実行スケジュール                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  開発サイクル（2週間スプリント）                                          │
│  ═══════════════════════════════════════════════════════════════════    │
│                                                                         │
│  Day 1-3: 開発フェーズ                                                  │
│  ├─ 開発者がユニットテストを作成                                        │
│  ├─ PR作成時に自動テスト実行                                            │
│  └─ コードレビュー                                                      │
│                                                                         │
│  Day 4-6: 統合フェーズ                                                  │
│  ├─ 統合テスト実行                                                      │
│  ├─ APIテスト実行                                                       │
│  └─ バグ修正                                                            │
│                                                                         │
│  Day 7-9: QAフェーズ                                                    │
│  ├─ E2Eテスト実行                                                       │
│  ├─ 探索的テスト                                                        │
│  ├─ パフォーマンステスト（週次）                                        │
│  └─ アクセシビリティテスト                                              │
│                                                                         │
│  Day 10: リリース準備                                                   │
│  ├─ リグレッションテスト                                                │
│  ├─ スモークテスト                                                      │
│  └─ リリース判定                                                        │
│                                                                         │
│  ナイトリービルド（毎日 2:00 AM JST）                                    │
│  ├─ フルテストスイート実行                                              │
│  ├─ セキュリティスキャン                                                │
│  └─ 依存関係監査                                                        │
│                                                                         │
│  週次テスト（毎週金曜日）                                                │
│  ├─ 負荷テスト                                                          │
│  ├─ ソークテスト                                                        │
│  └─ デバイスファームテスト                                              │
│                                                                         │
│  月次テスト                                                              │
│  ├─ ペネトレーションテスト（四半期）                                    │
│  ├─ アクセシビリティ監査                                                │
│  └─ パフォーマンスベースライン更新                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 テストリソース配分

```yaml
test_resource_allocation:
  team_structure:
    qa_engineers: 2-3名
    sdet: 1名（テスト自動化専門）
    developers: 全員（ユニットテスト担当）

  test_type_allocation:
    unit_tests:
      owner: 開発者
      review: QAエンジニア
      automation: 100%
      time_allocation: 開発時間の20%

    integration_tests:
      owner: SDET + 開発者
      automation: 95%
      time_allocation: スプリントの15%

    e2e_tests:
      owner: QAエンジニア + SDET
      automation: 80%
      manual: 20%（探索的テスト）
      time_allocation: スプリントの20%

    performance_tests:
      owner: SDET
      frequency: 週次
      time_allocation: 週8時間

    security_tests:
      owner: QAエンジニア + 外部
      automated: SAST/DAST
      manual: ペネトレーションテスト（四半期）
      time_allocation: 月16時間

  tools_budget:
    ci_cd: GitHub Actions (included)
    test_management: TestRail / Zephyr
    performance: k6 Cloud (optional)
    security: Snyk (Team plan)
    device_farm: Firebase Test Lab (pay-per-use)
```

### 7.3 テスト優先度マトリクス

```yaml
test_priority_matrix:
  p0_critical:
    description: リリースブロッカー、必ず実行
    examples:
      - 認証・ログイン機能
      - 決済処理
      - 主要ユーザーフロー（検索→予約→決済）
    execution: 毎PR、毎デプロイ前
    automation: 必須

  p1_high:
    description: 重要機能、リリース前に実行
    examples:
      - カート機能
      - 予約変更・キャンセル
      - ユーザープロフィール管理
    execution: 毎日（ナイトリー）
    automation: 強く推奨

  p2_medium:
    description: 一般機能、定期的に実行
    examples:
      - 検索フィルター・ソート
      - 通知設定
      - UI/UXの詳細
    execution: 週次
    automation: 推奨

  p3_low:
    description: エッジケース、時間があれば実行
    examples:
      - 古いデバイス対応
      - 極端なエッジケース
      - 非主要言語の表示
    execution: リリース前
    automation: オプション
```

---

## 第8章：文書間参照 & 統合ポイント

### 8.1 関連文書参照

```yaml
document_references:
  directly_related:
    - id: Doc-QA-001
      title: テスト戦略＆品質基準
      relation: 本文書の上位方針・戦略を定義
      cross_reference:
        - テストピラミッド比率（60:25:15）
        - カバレッジ目標（80%）
        - 品質成熟度モデル

    - id: Doc-QA-002
      title: モニタリング、アラート＆インシデント管理
      relation: 本番環境での品質監視
      cross_reference:
        - エラー率モニタリング
        - パフォーマンスダッシュボード
        - インシデント対応フロー

    - id: Doc-QA-003
      title: リリース管理＆ロールバック手順
      relation: テスト後のリリースプロセス
      cross_reference:
        - 品質ゲート定義
        - リリース承認基準
        - ロールバックトリガー

  technical_references:
    - id: Doc-SP-001
      title: API仕様書
      relation: APIテストケースの基礎
      cross_reference:
        - エンドポイント定義
        - リクエスト/レスポンススキーマ
        - エラーコード定義

    - id: Doc-SP-002
      title: データベーススキーマ仕様書
      relation: テストデータ設計の基礎
      cross_reference:
        - エンティティ定義
        - リレーション
        - 制約条件

    - id: Doc-SC-001
      title: セキュリティアーキテクチャ
      relation: セキュリティテストの基礎
      cross_reference:
        - 認証・認可要件
        - 暗号化要件
        - コンプライアンス要件

  implementation_references:
    - id: EXISTING_APP_ANALYSIS.md
      title: 既存アプリ分析
      relation: 現在の実装状況の把握
      cross_reference:
        - 実装済み機能一覧
        - 技術スタック
        - テストファイル数（46）
```

### 8.2 テストケースID体系

```yaml
test_case_id_system:
  format: "TC-{DOMAIN}-{SEQUENCE}"

  domains:
    AUTH: 認証・アカウント管理
    SEARCH: 検索機能
    BOOKING: 予約フロー
    PAYMENT: 決済処理
    ITINERARY: 旅程管理
    NOTIF: 通知・コミュニケーション
    API: APIテスト
    PERF: パフォーマンステスト
    SEC: セキュリティテスト
    A11Y: アクセシビリティテスト
    MOB: モバイル固有テスト

  sequence_ranges:
    001-099: 正常系テスト
    100-199: 異常系・エラーテスト
    200-299: 境界値テスト
    300-399: 互換性テスト

  examples:
    - TC-AUTH-001: 認証ドメイン、1番目のテスト
    - TC-API-PROD-005: API商品エンドポイント、5番目のテスト
    - TC-SEC-007: セキュリティ、7番目のテスト
```

### 8.3 メトリクス・KPI

```yaml
test_metrics:
  coverage_metrics:
    code_coverage:
      current: ~40%
      year_1_target: 80%
      measurement: lcov (Flutter), c8 (Backend)

    critical_path_coverage:
      target: 95%
      paths:
        - 認証フロー
        - 決済フロー
        - 予約フロー

  test_execution_metrics:
    test_pass_rate:
      target: 99%
      alert_threshold: 95%

    flaky_test_rate:
      target: < 1%
      tracking: 過去30日の失敗パターン分析

    test_execution_time:
      unit_tests: < 5分
      integration_tests: < 10分
      e2e_tests: < 20分
      full_suite: < 30分

  defect_metrics:
    defect_escape_rate:
      target: < 5%
      calculation: 本番バグ / 全バグ

    defect_density:
      target: < 0.5 / KLOC

    mean_time_to_detect:
      target: < 24時間

  automation_metrics:
    automation_rate:
      current: ~60%
      year_1_target: 85%
      year_2_target: 95%

    test_maintenance_effort:
      target: < 10% of development time
```

---

## 付録

### 付録A：テストケーステンプレート

```markdown
## テストケース

| 項目 | 内容 |
|------|------|
| **ID** | TC-{DOMAIN}-{SEQ} |
| **タイトル** | [簡潔な説明] |
| **優先度** | Critical / High / Medium / Low |
| **自動化** | Yes / No / Partial |
| **関連要件** | [要件ID] |

### 前提条件
- [前提条件1]
- [前提条件2]

### テストステップ
1. [ステップ1]
2. [ステップ2]
3. [ステップ3]

### 期待結果
- [期待結果1]
- [期待結果2]

### テストデータ
```yaml
input:
  field1: value1
  field2: value2
expected_output:
  status: success
  data: {...}
```

### 備考
- [特記事項]
```

### 付録B：テスト環境セットアップ手順

```yaml
environment_setup:
  local_flutter:
    steps:
      - "flutter pub get"
      - "flutter pub run build_runner build"
      - "flutter test"
    prerequisites:
      - Flutter SDK 3.8.1+
      - Dart SDK 3.8.1+

  local_backend:
    steps:
      - "cd backend && npm install"
      - "docker-compose up -d postgres"
      - "npx prisma migrate dev"
      - "npm test"
    prerequisites:
      - Node.js 18+
      - Docker

  ci_pipeline:
    flutter_tests: ".github/workflows/flutter-tests.yml"
    backend_tests: ".github/workflows/backend-tests.yml"
    e2e_tests: ".github/workflows/e2e-tests.yml"
```

### 付録C：テスト報告書フォーマット

```yaml
test_report_format:
  header:
    - テスト実行日時
    - テスト環境
    - テストスイート名
    - 実行者

  summary:
    - 総テスト数
    - 成功数
    - 失敗数
    - スキップ数
    - 成功率

  details:
    - 失敗テスト一覧
    - エラーメッセージ
    - スクリーンショット（E2E）
    - ログ

  metrics:
    - 実行時間
    - カバレッジ
    - 前回比較

  recommendations:
    - 要対応項目
    - リスク評価
```

---

## 文書情報

| 項目 | 内容 |
|------|------|
| Document ID | Doc-SP-004 |
| Version | 1.0.0 |
| Created | 2026-01-21 |
| Last Updated | 2026-01-21 |
| Status | Draft |
| Owner | Implementation Agent |
| Reviewers | QA Lead, Tech Lead, Engineering Manager |
| Total Lines | 2,000+ |

---

## 付録D：追加テストケース詳細

### D.1 着物レンタル機能テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-KIMONO-001 | 着物一覧表示 | 着物データが存在 | 1. ホーム画面から「着物」をタップ 2. 着物一覧画面を確認 | 着物コレクションが画像・価格と共に表示される | High | Yes |
| TC-KIMONO-002 | 着物詳細表示 | 着物一覧画面表示 | 1. 着物アイテムをタップ | 詳細画面で高解像度画像、説明、サイズ、価格が表示される | High | Yes |
| TC-KIMONO-003 | サイズ選択 | 着物詳細画面表示 | 1. サイズ選択肢を確認 2. 「M」サイズを選択 | サイズが選択され、在庫状況が更新される | High | Yes |
| TC-KIMONO-004 | アクセサリーバンドル追加 | 着物詳細画面表示 | 1. 推奨アクセサリーを確認 2. 帯を追加 3. 草履を追加 | セット割引が適用され、合計金額が更新される | High | Yes |
| TC-KIMONO-005 | 着物レンタル期間選択 | 着物詳細画面表示 | 1. レンタル期間を「3日間」に設定 | 期間に応じた価格が計算される | High | Yes |
| TC-KIMONO-006 | 着用日指定 | 着物詳細画面表示 | 1. カレンダーから着用日を選択 | 選択日の在庫・配送可否が確認される | Medium | Yes |
| TC-KIMONO-007 | サイズガイド表示 | 着物詳細画面表示 | 1. 「サイズガイド」リンクをタップ | サイズ測定方法と推奨サイズ表が表示される | Medium | Yes |
| TC-KIMONO-008 | 在庫切れ着物の表示 | 在庫0の着物が存在 | 1. 在庫切れ着物の詳細を開く | 「現在レンタル不可」と表示、予約通知オプションが表示される | Medium | Yes |

### D.2 QRコード機能テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-QR-001 | チケットQRコード表示 | チケット購入済み | 1. デジタルチケット画面を開く 2. 該当チケットをタップ 3. QRコードを表示 | 入場用QRコードが画面に表示される | Critical | Yes |
| TC-QR-002 | QRコード輝度調整 | QRコード表示中 | 1. 画面をタップ | 画面輝度が最大になり、QRコードが読み取りやすくなる | High | No |
| TC-QR-003 | オフラインQRコード表示 | チケットQRがキャッシュ済み、オフライン | 1. 機内モードON 2. チケットQRコードを表示 | オフラインでもQRコードが表示される | Critical | Yes |
| TC-QR-004 | QRコードスキャン（カメラ） | カメラ権限許可済み | 1. QRスキャナーを起動 2. 有効なQRコードをスキャン | QRコードが読み取られ、情報が表示される | High | No |
| TC-QR-005 | 無効なQRコードスキャン | QRスキャナー起動中 | 1. 無効なQRコードをスキャン | 「無効なQRコードです」エラーが表示される | High | No |
| TC-QR-006 | QRスキャナーのトーチ機能 | 暗い環境 | 1. QRスキャナーを起動 2. トーチアイコンをタップ | カメラのフラッシュライトが点灯する | Medium | No |
| TC-QR-007 | 使用済みQRコードの表示 | チケット使用済み | 1. 使用済みチケットのQRを表示 | 「使用済み」マークが表示される | Medium | Yes |

### D.3 多言語・多通貨テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-I18N-001 | 言語切り替え（日本語→英語） | 日本語設定 | 1. 設定画面を開く 2. 言語を「English」に変更 | 全UIテキストが英語に切り替わる | High | Yes |
| TC-I18N-002 | 言語切り替え（英語→日本語） | 英語設定 | 1. 設定画面を開く 2. 言語を「日本語」に変更 | 全UIテキストが日本語に切り替わる | High | Yes |
| TC-I18N-003 | 通貨切り替え（JPY→USD） | JPY設定 | 1. 設定画面を開く 2. 通貨を「USD」に変更 | 全価格がUSD換算で表示される | High | Yes |
| TC-I18N-004 | 通貨切り替え（JPY→EUR） | JPY設定 | 1. 設定画面を開く 2. 通貨を「EUR」に変更 | 全価格がEUR換算で表示される | Medium | Yes |
| TC-I18N-005 | 為替レート更新確認 | 通貨設定済み | 1. 商品価格を確認 2. 時間経過後に再確認 | 最新の為替レートが適用されている | Medium | No |
| TC-I18N-006 | 日付フォーマット（日本） | 言語:日本語 | 1. 予約日付を確認 | YYYY年MM月DD日形式で表示される | Low | Yes |
| TC-I18N-007 | 日付フォーマット（英語） | 言語:英語 | 1. 予約日付を確認 | MMM DD, YYYY形式で表示される | Low | Yes |
| TC-I18N-008 | 数値フォーマット（日本） | 言語:日本語 | 1. 価格を確認 | ¥10,000形式で表示される | Low | Yes |

### D.4 ホテルサービス詳細テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-HOTEL-001 | ホテル詳細情報表示 | ホテル一覧表示 | 1. ホテルを選択 | ホテル名、評価、住所、設備、写真が表示される | High | Yes |
| TC-HOTEL-002 | ホテル写真ギャラリー | ホテル詳細画面表示 | 1. 写真をタップ | フルスクリーンギャラリーで写真閲覧可能 | Medium | Yes |
| TC-HOTEL-003 | 部屋タイプ一覧表示 | ホテル詳細画面表示 | 1. 部屋タイプセクションを確認 | 各部屋タイプの価格、設備、空室状況が表示される | High | Yes |
| TC-HOTEL-004 | 部屋詳細表示 | ホテル詳細画面表示 | 1. 部屋タイプをタップ | 部屋の詳細情報（広さ、ベッド、アメニティ）が表示される | High | Yes |
| TC-HOTEL-005 | 施設予約（温泉） | ホテル詳細画面表示 | 1. 施設タブを開く 2. 「温泉」を選択 3. 時間帯を選択 | 温泉の予約が可能 | Medium | Yes |
| TC-HOTEL-006 | ルームサービス注文 | ホテル詳細画面表示 | 1. ルームサービスタブを開く 2. メニューを選択 3. 注文を確定 | ルームサービス注文が完了する | Medium | Yes |
| TC-HOTEL-007 | ホテルレビュー表示 | ホテル詳細画面表示 | 1. レビューセクションを確認 | 他のユーザーのレビューと評価が表示される | Medium | Yes |
| TC-HOTEL-008 | ホテル地図表示 | ホテル詳細画面表示 | 1. 「地図で見る」をタップ | 地図上にホテル位置がピンで表示される | Medium | No |

### D.5 注文履歴・ステータステストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-ORDER-001 | 注文履歴一覧表示 | 注文履歴が存在 | 1. ユーザー画面を開く 2. 注文履歴をタップ | 過去の注文が新しい順に表示される | High | Yes |
| TC-ORDER-002 | 注文ステータス「処理中」表示 | 処理中の注文が存在 | 1. 注文履歴を開く 2. 該当注文を確認 | 「処理中」ステータスとプログレス表示 | High | Yes |
| TC-ORDER-003 | 注文ステータス「発送済み」表示 | 発送済みの注文が存在 | 1. 注文履歴を開く 2. 該当注文を確認 | 「発送済み」ステータスと追跡番号が表示される | High | Yes |
| TC-ORDER-004 | 注文ステータス「配達完了」表示 | 配達完了の注文が存在 | 1. 注文履歴を開く 2. 該当注文を確認 | 「配達完了」ステータスと配達日時が表示される | High | Yes |
| TC-ORDER-005 | 注文詳細表示 | 注文履歴が存在 | 1. 注文履歴を開く 2. 注文をタップ | 注文番号、商品、金額、配送先、ステータスが表示される | High | Yes |
| TC-ORDER-006 | 配送追跡リンク | 発送済み注文が存在 | 1. 注文詳細を開く 2. 「追跡する」をタップ | 配送業者の追跡ページが開く | Medium | No |
| TC-ORDER-007 | 注文の再購入 | 完了済み注文が存在 | 1. 注文詳細を開く 2. 「再度購入」をタップ | 同じ商品がカートに追加される | Medium | Yes |
| TC-ORDER-008 | 注文履歴フィルター | 複数の注文が存在 | 1. 注文履歴を開く 2. 「ホテル予約」フィルターを選択 | ホテル予約のみが表示される | Low | Yes |

### D.6 エラーハンドリング詳細テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-ERR-001 | ネットワークエラー表示 | ネットワーク切断 | 1. API呼び出しを実行 | 「ネットワークに接続できません」エラーと再試行ボタンが表示される | High | Yes |
| TC-ERR-002 | サーバーエラー表示 | サーバー500エラー | 1. API呼び出しを実行 | 「サーバーエラーが発生しました」エラーが表示される | High | Yes |
| TC-ERR-003 | タイムアウトエラー表示 | 応答遅延 | 1. 遅いAPIを呼び出す | 「リクエストがタイムアウトしました」エラーが表示される | High | Yes |
| TC-ERR-004 | バリデーションエラー表示 | フォーム入力画面 | 1. 無効なデータを入力 2. 送信 | 該当フィールドにエラーメッセージが表示される | High | Yes |
| TC-ERR-005 | 認証エラー時のリダイレクト | トークン期限切れ | 1. 認証が必要なAPIを呼び出す | ログイン画面にリダイレクトされる | Critical | Yes |
| TC-ERR-006 | エラー後の再試行 | エラー表示中 | 1. 「再試行」ボタンをタップ | APIが再実行される | High | Yes |
| TC-ERR-007 | グレースフルデグラデーション | 一部API障害 | 1. 商品一覧を表示（レビューAPI障害） | 商品は表示され、レビュー部分のみ「読み込み失敗」表示 | Medium | No |
| TC-ERR-008 | オフラインフォールバック | オフライン状態 | 1. 検索を実行 | キャッシュされた結果が表示され、オフライン表示 | High | Yes |

### D.7 ユーザープロフィール詳細テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-PROFILE-001 | プロフィール情報表示 | ログイン済み | 1. ユーザー画面を開く | ユーザー名、メール、プロフィール画像が表示される | High | Yes |
| TC-PROFILE-002 | プロフィール編集 | ログイン済み | 1. プロフィール編集をタップ 2. 名前を変更 3. 保存 | 名前が更新される | High | Yes |
| TC-PROFILE-003 | プロフィール画像変更 | ログイン済み | 1. プロフィール画像をタップ 2. ギャラリーから画像選択 3. 保存 | プロフィール画像が更新される | Medium | No |
| TC-PROFILE-004 | 電話番号追加 | ログイン済み | 1. プロフィール編集をタップ 2. 電話番号を追加 3. 保存 | 電話番号が保存される | Medium | Yes |
| TC-PROFILE-005 | 住所登録 | ログイン済み | 1. 住所管理をタップ 2. 新しい住所を追加 3. 保存 | 住所が保存され、配送先として選択可能になる | High | Yes |
| TC-PROFILE-006 | 保存済みカード管理 | 保存済みカードが存在 | 1. 決済方法管理をタップ | 保存済みカード一覧が表示される（下4桁のみ） | High | Yes |
| TC-PROFILE-007 | カード削除 | 保存済みカードが存在 | 1. カードの削除ボタンをタップ 2. 確認で「はい」を選択 | カードが削除される | High | Yes |
| TC-PROFILE-008 | アカウント削除 | ログイン済み | 1. アカウント設定をタップ 2. 「アカウント削除」をタップ 3. パスワード入力 4. 確認 | アカウントが削除され、ログアウトされる | High | No |

### D.8 プッシュ通知詳細テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-PUSH-001 | 通知許可リクエスト | 初回起動 | 1. アプリを初回起動 | 通知許可ダイアログが表示される | High | No |
| TC-PUSH-002 | 通知許可後の受信 | 通知許可済み | 1. サーバーから通知送信 | プッシュ通知が表示される | High | No |
| TC-PUSH-003 | 通知拒否後の動作 | 通知拒否 | 1. サーバーから通知送信 | 通知は表示されず、アプリ内通知のみ | Medium | No |
| TC-PUSH-004 | 通知タップでアプリ起動 | アプリがバックグラウンド | 1. 注文確定通知をタップ | アプリが起動し、注文詳細画面が表示される | High | No |
| TC-PUSH-005 | バッジカウント更新 | 未読通知が存在 | 1. 通知を受信 | アプリアイコンのバッジが更新される | Medium | No |
| TC-PUSH-006 | 通知センターでの表示 | 通知受信済み | 1. 通知センターを開く | TripTripからの通知がグループ化されて表示される | Medium | No |

### D.9 検索サジェスト・履歴テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-SUGGEST-001 | 検索サジェスト表示 | 商品データが存在 | 1. 検索ボックスに「ki」と入力 | 「kimono」「kimono rental」等のサジェストが表示される | Medium | Yes |
| TC-SUGGEST-002 | サジェスト選択 | サジェスト表示中 | 1. サジェスト候補をタップ | 検索が実行され、結果が表示される | Medium | Yes |
| TC-SUGGEST-003 | 検索履歴保存 | 検索を実行 | 1. 「kimono」で検索 2. 検索画面を再度開く | 検索履歴に「kimono」が表示される | Low | Yes |
| TC-SUGGEST-004 | 検索履歴削除（個別） | 検索履歴が存在 | 1. 検索画面を開く 2. 履歴項目の削除ボタンをタップ | 該当履歴が削除される | Low | Yes |
| TC-SUGGEST-005 | 検索履歴全削除 | 検索履歴が存在 | 1. 検索画面を開く 2. 「履歴をクリア」をタップ | 全履歴が削除される | Low | Yes |
| TC-SUGGEST-006 | 人気検索ワード表示 | - | 1. 検索画面を開く（履歴なし） | 人気の検索キーワードが表示される | Low | Yes |

### D.10 カート詳細テストケース

| ID | タイトル | 前提条件 | テストステップ | 期待結果 | 優先度 | 自動化 |
|----|----------|----------|----------------|----------|--------|--------|
| TC-CART-001 | 空カート表示 | カートが空 | 1. カート画面を開く | 「カートは空です」メッセージと買い物を続けるボタンが表示される | Medium | Yes |
| TC-CART-002 | カートアイテム表示 | カートにアイテムあり | 1. カート画面を開く | 商品画像、名前、サイズ、数量、小計が表示される | High | Yes |
| TC-CART-003 | 数量増加 | カートにアイテムあり | 1. カート画面を開く 2. 数量「+」をタップ | 数量が1増加し、小計が更新される | High | Yes |
| TC-CART-004 | 数量減少 | カートにアイテムあり（数量2以上） | 1. カート画面を開く 2. 数量「-」をタップ | 数量が1減少し、小計が更新される | High | Yes |
| TC-CART-005 | 数量1からの減少 | カートにアイテムあり（数量1） | 1. カート画面を開く 2. 数量「-」をタップ | 削除確認ダイアログが表示される | High | Yes |
| TC-CART-006 | 複数アイテムの合計計算 | カートに複数アイテムあり | 1. カート画面を開く | 全アイテムの合計金額が正しく表示される | High | Yes |
| TC-CART-007 | 税込み価格表示 | カートにアイテムあり | 1. カート画面を開く 2. 合計を確認 | 税込み合計金額が表示される | Medium | Yes |
| TC-CART-008 | 送料計算 | カートにアイテムあり | 1. 配送先を入力 2. 送料を確認 | 配送先に応じた送料が計算される | High | Yes |
| TC-CART-009 | 送料無料条件表示 | 送料がかかる金額のカート | 1. カート画面を開く | 「あと¥X,XXXで送料無料」メッセージが表示される | Low | Yes |
| TC-CART-010 | クーポン適用 | 有効なクーポンコードあり | 1. カート画面を開く 2. クーポンコードを入力 3. 適用 | 割引が適用され、合計が更新される | Medium | Yes |

---

## 付録E：テスト自動化コードサンプル

### E.1 ユニットテストサンプル（Flutter）

```dart
// test/services/cart_service_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import 'package:mockito/annotations.dart';
import 'package:triptrip/services/cart_service.dart';
import 'package:triptrip/models/cart_item.dart';
import 'package:triptrip/services/storage/cart_storage_service.dart';

@GenerateMocks([CartStorageService])
import 'cart_service_test.mocks.dart';

void main() {
  late CartService cartService;
  late MockCartStorageService mockStorage;

  setUp(() {
    mockStorage = MockCartStorageService();
    cartService = CartService(storage: mockStorage);
    when(mockStorage.saveCart(any)).thenAnswer((_) async => true);
    when(mockStorage.loadCart()).thenAnswer((_) async => []);
  });

  group('CartService - 商品追加', () {
    test('新規商品をカートに追加', () async {
      final item = CartItem(
        id: 'item-1',
        productId: 'prod-1',
        name: 'テスト着物',
        price: 10000,
        quantity: 1,
        size: 'M',
        imageUrl: 'https://example.com/image.jpg',
      );

      await cartService.addItem(item);

      expect(cartService.items.length, 1);
      expect(cartService.items.first.name, 'テスト着物');
      expect(cartService.totalAmount, 10000);
    });

    test('同一商品追加で数量が増加', () async {
      final item1 = CartItem(
        id: 'item-1',
        productId: 'prod-1',
        name: 'テスト着物',
        price: 10000,
        quantity: 1,
        size: 'M',
      );
      final item2 = CartItem(
        id: 'item-2',
        productId: 'prod-1',
        name: 'テスト着物',
        price: 10000,
        quantity: 2,
        size: 'M',
      );

      await cartService.addItem(item1);
      await cartService.addItem(item2);

      expect(cartService.items.length, 1);
      expect(cartService.items.first.quantity, 3);
      expect(cartService.totalAmount, 30000);
    });

    test('異なるサイズは別アイテムとして追加', () async {
      final itemM = CartItem(
        id: 'item-1',
        productId: 'prod-1',
        name: 'テスト着物',
        price: 10000,
        quantity: 1,
        size: 'M',
      );
      final itemL = CartItem(
        id: 'item-2',
        productId: 'prod-1',
        name: 'テスト着物',
        price: 10000,
        quantity: 1,
        size: 'L',
      );

      await cartService.addItem(itemM);
      await cartService.addItem(itemL);

      expect(cartService.items.length, 2);
    });
  });

  group('CartService - 合計計算', () {
    test('複数商品の合計を正しく計算', () async {
      final items = [
        CartItem(id: '1', productId: 'p1', name: 'Item 1', price: 1000, quantity: 2),
        CartItem(id: '2', productId: 'p2', name: 'Item 2', price: 500, quantity: 3),
        CartItem(id: '3', productId: 'p3', name: 'Item 3', price: 2000, quantity: 1),
      ];

      for (final item in items) {
        await cartService.addItem(item);
      }

      // 1000*2 + 500*3 + 2000*1 = 5500
      expect(cartService.totalAmount, 5500);
      expect(cartService.itemCount, 6);
    });

    test('空カートの合計は0', () {
      expect(cartService.totalAmount, 0);
      expect(cartService.itemCount, 0);
    });

    test('割引適用後の合計計算', () async {
      final item = CartItem(
        id: '1',
        productId: 'p1',
        name: 'Item 1',
        price: 10000,
        quantity: 1,
      );
      await cartService.addItem(item);
      cartService.applyDiscount(percentage: 10);

      expect(cartService.discountAmount, 1000);
      expect(cartService.finalAmount, 9000);
    });
  });

  group('CartService - 削除', () {
    test('カートから商品を削除', () async {
      final item = CartItem(
        id: 'item-1',
        productId: 'prod-1',
        name: 'テスト着物',
        price: 10000,
        quantity: 1,
      );
      await cartService.addItem(item);
      await cartService.removeItem('item-1');

      expect(cartService.items.length, 0);
      expect(cartService.totalAmount, 0);
    });

    test('カートをクリア', () async {
      final items = [
        CartItem(id: '1', productId: 'p1', name: 'Item 1', price: 1000, quantity: 1),
        CartItem(id: '2', productId: 'p2', name: 'Item 2', price: 500, quantity: 1),
      ];
      for (final item in items) {
        await cartService.addItem(item);
      }
      await cartService.clearCart();

      expect(cartService.items.length, 0);
    });
  });
}
```

### E.2 APIテストサンプル（TypeScript/Vitest）

```typescript
// backend/src/routes/__tests__/products.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import supertest from 'supertest';
import { app } from '../../app';
import { prisma } from '../../lib/prisma';

describe('Products API', () => {
  const request = supertest(app);
  let testProductId: string;

  beforeAll(async () => {
    // テストデータ作成
    const product = await prisma.product.create({
      data: {
        name: 'テスト着物',
        description: 'テスト用の着物です',
        price: 10000,
        category: 'kimono',
        stock: 5,
        images: ['https://example.com/image1.jpg'],
      },
    });
    testProductId = product.id;
  });

  afterAll(async () => {
    await prisma.product.deleteMany({
      where: { name: { startsWith: 'テスト' } },
    });
    await prisma.$disconnect();
  });

  describe('GET /api/v1/products', () => {
    it('商品一覧を取得できる', async () => {
      const response = await request.get('/api/v1/products');

      expect(response.status).toBe(200);
      expect(response.body).toHaveProperty('products');
      expect(response.body).toHaveProperty('pagination');
      expect(Array.isArray(response.body.products)).toBe(true);
    });

    it('カテゴリでフィルタリングできる', async () => {
      const response = await request
        .get('/api/v1/products')
        .query({ category: 'kimono' });

      expect(response.status).toBe(200);
      expect(response.body.products.every(
        (p: any) => p.category === 'kimono'
      )).toBe(true);
    });

    it('価格範囲でフィルタリングできる', async () => {
      const response = await request
        .get('/api/v1/products')
        .query({ minPrice: 5000, maxPrice: 15000 });

      expect(response.status).toBe(200);
      expect(response.body.products.every(
        (p: any) => p.price >= 5000 && p.price <= 15000
      )).toBe(true);
    });

    it('キーワード検索ができる', async () => {
      const response = await request
        .get('/api/v1/products')
        .query({ q: '着物' });

      expect(response.status).toBe(200);
      expect(response.body.products.some(
        (p: any) => p.name.includes('着物')
      )).toBe(true);
    });

    it('ページネーションが動作する', async () => {
      const response = await request
        .get('/api/v1/products')
        .query({ page: 1, limit: 10 });

      expect(response.status).toBe(200);
      expect(response.body.pagination).toMatchObject({
        page: 1,
        limit: 10,
      });
      expect(response.body.products.length).toBeLessThanOrEqual(10);
    });
  });

  describe('GET /api/v1/products/:id', () => {
    it('商品詳細を取得できる', async () => {
      const response = await request.get(`/api/v1/products/${testProductId}`);

      expect(response.status).toBe(200);
      expect(response.body).toHaveProperty('id', testProductId);
      expect(response.body).toHaveProperty('name', 'テスト着物');
      expect(response.body).toHaveProperty('price', 10000);
    });

    it('存在しない商品IDで404を返す', async () => {
      const response = await request.get('/api/v1/products/non-existent-id');

      expect(response.status).toBe(404);
      expect(response.body).toHaveProperty('error');
    });
  });

  describe('GET /api/v1/products/:id/stock', () => {
    it('在庫情報を取得できる', async () => {
      const response = await request.get(`/api/v1/products/${testProductId}/stock`);

      expect(response.status).toBe(200);
      expect(response.body).toHaveProperty('stock', 5);
      expect(response.body).toHaveProperty('available', true);
    });
  });
});
```

### E.3 E2Eテストサンプル（Flutter Integration Test）

```dart
// integration_test/e2e/checkout_flow_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:triptrip/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('Checkout Flow E2E Tests', () {
    testWidgets('商品をカートに追加してチェックアウト完了', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // ステップ1: ログイン
      await _performLogin(tester, 'test@example.com', 'password123');

      // ステップ2: 商品を検索
      await tester.tap(find.byKey(const Key('nav_search')));
      await tester.pumpAndSettle();
      await tester.enterText(find.byKey(const Key('search_input')), '着物');
      await tester.tap(find.byKey(const Key('search_button')));
      await tester.pumpAndSettle();

      // ステップ3: 最初の商品を選択
      expect(find.byType(ProductCard), findsWidgets);
      await tester.tap(find.byType(ProductCard).first);
      await tester.pumpAndSettle();

      // ステップ4: サイズを選択してカートに追加
      await tester.tap(find.text('M'));
      await tester.pumpAndSettle();
      await tester.tap(find.text('カートに追加'));
      await tester.pumpAndSettle();

      // カート追加成功を確認
      expect(find.text('カートに追加しました'), findsOneWidget);

      // ステップ5: カートに移動
      await tester.tap(find.byKey(const Key('nav_cart')));
      await tester.pumpAndSettle();

      // カートに商品があることを確認
      expect(find.byType(CartItemWidget), findsOneWidget);

      // ステップ6: チェックアウト
      await tester.tap(find.text('チェックアウト'));
      await tester.pumpAndSettle();

      // ステップ7: 配送先入力
      await tester.enterText(find.byKey(const Key('name_input')), 'テストユーザー');
      await tester.enterText(find.byKey(const Key('address_input')), '東京都渋谷区1-1-1');
      await tester.enterText(find.byKey(const Key('phone_input')), '090-1234-5678');
      await tester.tap(find.text('次へ'));
      await tester.pumpAndSettle();

      // ステップ8: 決済情報入力
      await tester.enterText(find.byKey(const Key('card_number')), '4242424242424242');
      await tester.enterText(find.byKey(const Key('expiry')), '12/28');
      await tester.enterText(find.byKey(const Key('cvc')), '123');
      await tester.tap(find.text('支払う'));
      await tester.pumpAndSettle(const Duration(seconds: 5));

      // ステップ9: 注文完了確認
      expect(find.text('ご注文ありがとうございます'), findsOneWidget);
      expect(find.byKey(const Key('order_number')), findsOneWidget);
    });

    testWidgets('決済失敗時のエラーハンドリング', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      await _performLogin(tester, 'test@example.com', 'password123');
      await _addItemToCart(tester);
      await _proceedToCheckout(tester);

      // 決済失敗カードを使用
      await tester.enterText(find.byKey(const Key('card_number')), '4000000000000002');
      await tester.enterText(find.byKey(const Key('expiry')), '12/28');
      await tester.enterText(find.byKey(const Key('cvc')), '123');
      await tester.tap(find.text('支払う'));
      await tester.pumpAndSettle(const Duration(seconds: 5));

      // エラーメッセージ確認
      expect(find.text('決済に失敗しました'), findsOneWidget);
      expect(find.text('再試行'), findsOneWidget);
    });
  });
}

Future<void> _performLogin(WidgetTester tester, String email, String password) async {
  await tester.tap(find.text('ログイン'));
  await tester.pumpAndSettle();
  await tester.enterText(find.byKey(const Key('email_input')), email);
  await tester.enterText(find.byKey(const Key('password_input')), password);
  await tester.tap(find.byKey(const Key('login_button')));
  await tester.pumpAndSettle(const Duration(seconds: 2));
}

Future<void> _addItemToCart(WidgetTester tester) async {
  await tester.tap(find.byKey(const Key('nav_products')));
  await tester.pumpAndSettle();
  await tester.tap(find.byType(ProductCard).first);
  await tester.pumpAndSettle();
  await tester.tap(find.text('M'));
  await tester.tap(find.text('カートに追加'));
  await tester.pumpAndSettle();
}

Future<void> _proceedToCheckout(WidgetTester tester) async {
  await tester.tap(find.byKey(const Key('nav_cart')));
  await tester.pumpAndSettle();
  await tester.tap(find.text('チェックアウト'));
  await tester.pumpAndSettle();
  await tester.enterText(find.byKey(const Key('name_input')), 'テストユーザー');
  await tester.enterText(find.byKey(const Key('address_input')), '東京都渋谷区1-1-1');
  await tester.enterText(find.byKey(const Key('phone_input')), '090-1234-5678');
  await tester.tap(find.text('次へ'));
  await tester.pumpAndSettle();
}
```

---

## 付録F：テスト結果サンプルレポート

```yaml
test_execution_report:
  metadata:
    date: 2026-01-21
    environment: CI (GitHub Actions)
    branch: feature/checkout-improvement
    commit: abc123def
    executor: Automated

  summary:
    total_tests: 347
    passed: 341
    failed: 4
    skipped: 2
    pass_rate: 98.27%
    execution_time: 12m 34s

  coverage:
    overall: 82.3%
    flutter: 78.5%
    backend: 86.1%
    critical_paths: 94.2%

  failed_tests:
    - id: TC-PAYMENT-003
      name: 期限切れカードで決済試行
      error: "Expected '期限切れ' but got '無効なカード'"
      priority: High
      assignee: 未割り当て

    - id: TC-API-ORD-007
      name: 注文キャンセル
      error: "Timeout: キャンセルAPI応答なし"
      priority: High
      assignee: 未割り当て

    - id: TC-MOB-024
      name: VPN接続時の動作
      error: "VPN環境構築失敗"
      priority: Low
      assignee: 未割り当て

    - id: TC-PERF-003
      name: ホテル検索レスポンスタイム
      error: "p95: 350ms (目標: 300ms)"
      priority: High
      assignee: 未割り当て

  skipped_tests:
    - id: TC-PUSH-001
      reason: "iOSシミュレータではプッシュ通知テスト不可"
    - id: TC-A11Y-008
      reason: "VoiceOverテストは手動実行"

  recommendations:
    - "TC-PAYMENT-003: エラーメッセージの検証ロジック修正が必要"
    - "TC-API-ORD-007: キャンセルAPIのタイムアウト設定確認"
    - "TC-PERF-003: ホテル検索クエリの最適化検討"
```

---

## 付録G：k6パフォーマンステストスクリプト

```javascript
// performance/triptrip-load-test.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// カスタムメトリクス
const errorRate = new Rate('errors');
const searchLatency = new Trend('search_latency');
const checkoutLatency = new Trend('checkout_latency');

// テスト設定
export const options = {
  scenarios: {
    // シナリオ1: 通常負荷
    normal_load: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 50 },
        { duration: '5m', target: 100 },
        { duration: '2m', target: 0 },
      ],
      gracefulRampDown: '30s',
    },
    // シナリオ2: スパイクテスト
    spike_test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 100 },
        { duration: '30s', target: 500 },
        { duration: '1m', target: 100 },
      ],
      startTime: '10m',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<300', 'p(99)<500'],
    errors: ['rate<0.01'],
    search_latency: ['p(95)<200'],
    checkout_latency: ['p(95)<500'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'https://api.triptrip.example.com';

export default function () {
  group('商品検索フロー', () => {
    const start = Date.now();
    const res = http.get(`${BASE_URL}/api/v1/products?q=kimono&limit=20`);
    searchLatency.add(Date.now() - start);

    check(res, {
      'ステータス200': (r) => r.status === 200,
      '商品が返される': (r) => JSON.parse(r.body).products.length > 0,
    }) || errorRate.add(1);
  });

  sleep(Math.random() * 2);

  group('ホテル検索フロー', () => {
    const res = http.get(
      `${BASE_URL}/api/v1/hotels?location=tokyo&checkIn=2026-03-01&checkOut=2026-03-03`
    );
    check(res, {
      'ステータス200': (r) => r.status === 200,
    }) || errorRate.add(1);
  });

  sleep(1);
}
```

---

## 付録H：テストデータファクトリー

```typescript
// backend/src/test/factories/index.ts
import { faker } from '@faker-js/faker/locale/ja';
import { PrismaClient, User, Product, Order } from '@prisma/client';

export class TestDataFactory {
  constructor(private prisma: PrismaClient) {}

  // ユーザー生成
  async createUser(overrides: Partial<User> = {}): Promise<User> {
    return this.prisma.user.create({
      data: {
        email: faker.internet.email(),
        name: faker.person.fullName(),
        password: 'hashed_password',
        ...overrides,
      },
    });
  }

  // 商品生成
  async createProduct(overrides: Partial<Product> = {}): Promise<Product> {
    return this.prisma.product.create({
      data: {
        name: faker.commerce.productName(),
        description: faker.commerce.productDescription(),
        price: faker.number.int({ min: 1000, max: 50000 }),
        category: faker.helpers.arrayElement(['kimono', 'accessories', 'souvenirs']),
        stock: faker.number.int({ min: 0, max: 100 }),
        images: [faker.image.url()],
        ...overrides,
      },
    });
  }

  // 注文生成
  async createOrder(userId: string, overrides: Partial<Order> = {}): Promise<Order> {
    return this.prisma.order.create({
      data: {
        userId,
        total: faker.number.int({ min: 1000, max: 100000 }),
        status: 'PENDING',
        shippingAddress: {
          street: faker.location.streetAddress(),
          city: faker.location.city(),
          postalCode: faker.location.zipCode(),
        },
        ...overrides,
      },
    });
  }

  // クリーンアップ
  async cleanup(): Promise<void> {
    await this.prisma.orderItem.deleteMany();
    await this.prisma.order.deleteMany();
    await this.prisma.product.deleteMany();
    await this.prisma.user.deleteMany();
  }
}
```

---

## 付録I：CI/CDテスト設定

```yaml
# .github/workflows/test.yml
name: TripTrip Test Suite

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  FLUTTER_VERSION: '3.24.0'
  NODE_VERSION: '20'

jobs:
  flutter-unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true
      - run: flutter pub get
        working-directory: app
      - run: flutter test --coverage
        working-directory: app
      - uses: codecov/codecov-action@v4
        with:
          files: app/coverage/lcov.info
          flags: flutter-unit

  backend-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: triptrip_test
        ports: ['5432:5432']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
      - run: npm ci
        working-directory: backend
      - run: npx prisma migrate deploy
        working-directory: backend
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/triptrip_test
      - run: npm test -- --coverage
        working-directory: backend

  e2e-tests:
    runs-on: ubuntu-latest
    needs: [flutter-unit-tests, backend-tests]
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
      - run: flutter test integration_test/
        working-directory: app
```

---

**次のステップ**:
1. テストケースの詳細化とテスト管理ツールへの登録
2. 自動化対象テストの実装
3. CI/CDパイプラインへの統合
4. 定期的なテストケースの見直しと更新

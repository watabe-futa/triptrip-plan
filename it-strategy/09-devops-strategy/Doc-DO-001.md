# Doc-DO-001: TripTrip CI/CD Pipeline Architecture（詳細実装仕様）

## エグゼクティブサマリー

本文書は、TripTripプラットフォームのCI/CDパイプラインアーキテクチャの**具体的な実装仕様**を定義します。Doc-DM-002で策定した概念フレームワークとベストプラクティスを基盤とし、GitHub Actionsを中心とした実装詳細、ワークフロー設定、ジョブ定義、品質ゲートの具体的な設定を提供します。Google、Netflix、Amazonレベルの高度なCI/CDパイプラインを実現し、既存のFlutter/Node.js/PostgreSQLスタックとDocker環境を発展させ、コミットから本番デプロイまでのリードタイムを1時間以内に短縮することを目標とします。

---

## 第1章 はじめに & コンテキスト

### 1.1 Doc-DM-002との関係性（概念 vs 実装）

#### 1.1.1 文書の位置づけ

```yaml
document_hierarchy:
  doc_dm_002:
    level: 概念・方法論レベル
    scope:
      - DevOpsの原則とベストプラクティス
      - CI/CDの設計パターン
      - DORAメトリクス目標
      - 高レベルなパイプライン設計
      - ツール選定の方針
    output: "何を達成すべきか"

  doc_do_001:
    level: 実装詳細レベル
    scope:
      - GitHub Actions ワークフローファイル
      - 具体的なジョブ定義とステップ
      - YAMLファイル構造と設定
      - キャッシュ戦略の実装
      - シークレット管理の具体的設定
    output: "どのように実装するか"
```

#### 1.1.2 補完関係

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    文書間の補完関係                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Doc-DM-002（概念）                 Doc-DO-001（実装）                   │
│  ─────────────────                  ─────────────────                    │
│                                                                          │
│  ┌─────────────────┐               ┌─────────────────┐                  │
│  │ CI/CDパイプライン │               │ GitHub Actions  │                  │
│  │ 設計パターン      │──────────────▶│ ワークフロー定義 │                  │
│  └─────────────────┘               └─────────────────┘                  │
│                                                                          │
│  ┌─────────────────┐               ┌─────────────────┐                  │
│  │ 品質ゲート       │               │ 具体的な閾値と   │                  │
│  │ ベストプラクティス│──────────────▶│ チェック実装     │                  │
│  └─────────────────┘               └─────────────────┘                  │
│                                                                          │
│  ┌─────────────────┐               ┌─────────────────┐                  │
│  │ ビルド最適化     │               │ キャッシュ設定   │                  │
│  │ 戦略             │──────────────▶│ 並列実行設定     │                  │
│  └─────────────────┘               └─────────────────┘                  │
│                                                                          │
│  ┌─────────────────┐               ┌─────────────────┐                  │
│  │ 環境管理         │               │ 環境別YAML      │                  │
│  │ 方針             │──────────────▶│ シークレット設定 │                  │
│  └─────────────────┘               └─────────────────┘                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 既存CI/CD環境の分析

#### 1.2.1 現在の技術スタック詳細

```yaml
# EXISTING_APP_ANALYSIS.mdより抽出
current_ci_cd_stack:
  source_control:
    platform: GitHub
    repository_structure:
      - app/: Flutterアプリケーション（34,917行）
      - backend/: Node.js/TypeScriptバックエンド（4,738行）
      - docker-compose.yaml: 開発環境定義
      - Makefile: 開発タスク自動化

  frontend:
    framework: Flutter 3.8.1+
    language: Dart 3.8.1+
    code_generation:
      - build_runner: 2.5.4
      - freezed: 3.0.0
      - json_serializable: 6.7.1
    testing:
      - flutter_test: built-in
      - mockito: 5.4.4
      - patrol: 3.19.0
      - integration_test: built-in
    test_files: 46

  backend:
    runtime: Node.js 18+
    framework: Hono 4.8.3
    language: TypeScript 5.3.3
    orm: Prisma 5.8.0
    database: PostgreSQL 16
    testing:
      - vitest: 1.0.0
      - supertest: 6.3.3
    linting: ESLint 8.56.0

  containerization:
    tool: Docker
    orchestration: docker-compose
    base_images:
      backend: node:18-alpine
      database: postgres:16

  current_automation:
    makefile_targets:
      - make dev: 開発環境起動
      - make test: テスト実行
      - make build: ビルド実行
      - make lint: リンティング
```

#### 1.2.2 現在の課題と改善ポイント

```yaml
current_challenges:
  pipeline_structure:
    issue: "統一されたパイプライン定義の不足"
    impact: "ビルド・テスト・デプロイの一貫性欠如"
    solution: "標準化されたワークフロー定義"

  build_performance:
    issue: "キャッシュ戦略の最適化不足"
    impact: "ビルド時間の長期化"
    solution: "多層キャッシュ戦略の実装"

  quality_gates:
    issue: "自動品質チェックの不完全"
    impact: "品質問題の本番流出リスク"
    solution: "包括的な品質ゲート実装"

  security_scanning:
    issue: "セキュリティスキャンの統合不足"
    impact: "脆弱性の早期検出困難"
    solution: "SAST/DAST/依存関係スキャン統合"

  environment_consistency:
    issue: "環境間の構成差異"
    impact: "環境依存バグの発生"
    solution: "Docker化とIaC統合"
```

### 1.3 目標パイプライン要件

#### 1.3.1 パフォーマンス目標

```yaml
pipeline_performance_targets:
  build_times:
    flutter_android:
      current: "~10 minutes"
      target: "< 5 minutes"
      optimization:
        - Gradle キャッシュ
        - 並列ビルド
        - 増分ビルド

    flutter_ios:
      current: "~15 minutes"
      target: "< 8 minutes"
      optimization:
        - CocoaPods キャッシュ
        - ビルドキャッシュ
        - Fastlane 最適化

    flutter_web:
      current: "~5 minutes"
      target: "< 3 minutes"
      optimization:
        - Tree shaking
        - 遅延ロード

    backend:
      current: "~3 minutes"
      target: "< 1 minute"
      optimization:
        - npm キャッシュ
        - TypeScript 増分ビルド

  total_pipeline:
    pr_check:
      target: "< 15 minutes"
      stages:
        - lint_format: "< 2 minutes"
        - build: "< 5 minutes"
        - test: "< 5 minutes"
        - security_scan: "< 3 minutes"

    full_deploy:
      target: "< 30 minutes"
      stages:
        - ci: "< 15 minutes"
        - staging_deploy: "< 5 minutes"
        - production_deploy: "< 10 minutes"
```

#### 1.3.2 品質目標

```yaml
quality_targets:
  test_coverage:
    unit_tests:
      flutter: ">= 80%"
      backend: ">= 85%"
    integration_tests: ">= 70%"
    e2e_tests: "Critical paths covered"

  code_quality:
    lint_errors: 0
    format_violations: 0
    static_analysis:
      flutter_analyze: "No issues"
      eslint: "No errors"
      sonarqube_quality_gate: "Pass"

  security:
    high_vulnerabilities: 0
    critical_vulnerabilities: 0
    secret_leaks: 0
    container_vulnerabilities: "No critical/high"

  performance:
    api_response_p95: "< 200ms"
    app_startup_time: "< 3 seconds"
```

---

## 第2章 GitHub Actions アーキテクチャ詳細

### 2.1 ワークフロー構成とファイル構造

#### 2.1.1 ワークフローディレクトリ構造

```
.github/
├── workflows/
│   ├── ci-flutter.yaml              # Flutter CI パイプライン
│   ├── ci-backend.yaml              # Backend CI パイプライン
│   ├── ci-monorepo.yaml             # Monorepo 統合パイプライン
│   ├── deploy-staging.yaml          # Staging デプロイ
│   ├── deploy-production.yaml       # Production デプロイ
│   ├── security-scan.yaml           # セキュリティスキャン
│   ├── dependency-update.yaml       # 依存関係更新
│   ├── release.yaml                 # リリースワークフロー
│   ├── hotfix.yaml                  # Hotfix ワークフロー
│   ├── nightly.yaml                 # 夜間ビルド・テスト
│   └── performance-test.yaml        # パフォーマンステスト
│
├── actions/
│   ├── setup-flutter/
│   │   └── action.yaml              # Flutter セットアップ
│   ├── setup-backend/
│   │   └── action.yaml              # Backend セットアップ
│   ├── docker-build/
│   │   └── action.yaml              # Docker ビルド
│   ├── deploy-gke/
│   │   └── action.yaml              # GKE デプロイ
│   ├── notify-slack/
│   │   └── action.yaml              # Slack 通知
│   └── quality-gate/
│       └── action.yaml              # 品質ゲートチェック
│
├── CODEOWNERS                        # コードオーナー定義
└── dependabot.yml                    # Dependabot 設定
```

#### 2.1.2 共通環境変数定義

```yaml
# .github/workflows/env.yaml（参照用、実際はワークフロー内に定義）
env:
  # Flutter/Dart
  FLUTTER_VERSION: "3.24.0"
  DART_SDK_VERSION: "3.5.0"

  # Node.js
  NODE_VERSION: "20"
  NPM_VERSION: "10"

  # Container Registry
  REGISTRY: asia-northeast1-docker.pkg.dev
  PROJECT_ID: triptrip-production
  REGISTRY_STAGING: asia-northeast1-docker.pkg.dev/triptrip-staging
  REGISTRY_PRODUCTION: asia-northeast1-docker.pkg.dev/triptrip-production

  # GKE Clusters
  GKE_CLUSTER_STAGING: triptrip-staging
  GKE_CLUSTER_PRODUCTION: triptrip-production
  GKE_REGION: asia-northeast1

  # Application
  APP_NAME: triptrip
  BACKEND_IMAGE: triptrip-backend
  FRONTEND_WEB_IMAGE: triptrip-web

  # Timeouts
  BUILD_TIMEOUT: 30
  TEST_TIMEOUT: 20
  DEPLOY_TIMEOUT: 15
```

### 2.2 Runner戦略（self-hosted vs GitHub-hosted）

#### 2.2.1 Runner選択マトリクス

```yaml
runner_strategy:
  github_hosted:
    use_cases:
      - 標準的なCI/CDタスク
      - セキュリティスキャン
      - 軽量なビルドタスク
    runners:
      ubuntu_latest:
        specs: "4 vCPU, 16 GB RAM, 14 GB SSD"
        usage:
          - Backend ビルド・テスト
          - Docker イメージビルド
          - セキュリティスキャン
      macos_latest:
        specs: "4 vCPU, 14 GB RAM"
        usage:
          - iOS ビルド
          - CocoaPods 依存関係

    benefits:
      - メンテナンス不要
      - 高可用性
      - セキュリティ更新自動

  self_hosted:
    use_cases:
      - 大規模ビルド
      - GPU 必要タスク
      - 特殊ハードウェア要件
      - コスト最適化（大量ビルド時）
    configuration:
      labels:
        - self-hosted
        - linux
        - x64
        - gpu (ML タスク用)
      specs:
        standard:
          vcpu: 8
          memory: 32GB
          storage: 200GB SSD
        gpu:
          vcpu: 16
          memory: 64GB
          gpu: NVIDIA T4

    security:
      - VPC 内配置
      - IAM 最小権限
      - 定期的なイメージ更新
      - Ephemeral runners 推奨

  decision_matrix:
    | タスク | Runner | 理由 |
    |--------|--------|------|
    | Flutter Android Build | GitHub-hosted | 標準的なビルド環境 |
    | Flutter iOS Build | GitHub-hosted (macos) | Apple SDK 必要 |
    | Backend Build | GitHub-hosted | 軽量、高速 |
    | E2E Tests | Self-hosted | 長時間実行、リソース集約 |
    | ML Model Training | Self-hosted (gpu) | GPU 必要 |
    | Security Scan | GitHub-hosted | 分離環境推奨 |
```

#### 2.2.2 Self-hosted Runner 設定

```yaml
# runner-deployment.yaml (Kubernetes)
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: triptrip-runner
  namespace: github-actions
spec:
  replicas: 3
  template:
    spec:
      repository: triptrip/triptrip-app
      labels:
        - self-hosted
        - linux
        - x64
        - triptrip
      env:
        - name: DOCKER_HOST
          value: tcp://localhost:2376
        - name: DOCKER_TLS_VERIFY
          value: "1"
      resources:
        limits:
          cpu: "4"
          memory: "16Gi"
        requests:
          cpu: "2"
          memory: "8Gi"
      volumeMounts:
        - name: work
          mountPath: /runner/_work
        - name: docker-certs
          mountPath: /certs/client
          readOnly: true
      volumes:
        - name: work
          emptyDir: {}
        - name: docker-certs
          secret:
            secretName: docker-certs
      sidecarContainers:
        - name: docker
          image: docker:24-dind
          securityContext:
            privileged: true
          volumeMounts:
            - name: work
              mountPath: /runner/_work
            - name: docker-certs
              mountPath: /certs/client
---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: triptrip-runner-autoscaler
  namespace: github-actions
spec:
  scaleTargetRef:
    name: triptrip-runner
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: TotalNumberOfQueuedAndInProgressWorkflowRuns
      repositoryNames:
        - triptrip/triptrip-app
      scaleUpThreshold: "2"
      scaleDownThreshold: "0"
      scaleUpFactor: "2"
      scaleDownFactor: "0.5"
```

### 2.3 シークレット管理とOIDC統合

#### 2.3.1 シークレット階層設計

```yaml
secrets_hierarchy:
  organization_level:
    description: "全リポジトリ共通シークレット"
    secrets:
      - SONAR_TOKEN: SonarQube 認証
      - SNYK_TOKEN: Snyk セキュリティスキャン
      - SLACK_WEBHOOK_URL: Slack 通知

  repository_level:
    description: "リポジトリ固有シークレット"
    secrets:
      - GCP_SA_KEY: GCP サービスアカウントキー
      - CODECOV_TOKEN: Codecov トークン
      - APPLE_CERTIFICATES: iOS 署名証明書
      - ANDROID_KEYSTORE: Android 署名キーストア

  environment_level:
    description: "環境別シークレット"
    environments:
      staging:
        - DATABASE_URL: Staging DB 接続文字列
        - API_SECRET_KEY: Staging API シークレット
        - LAUNCHDARKLY_SDK_KEY: Staging フィーチャーフラグ
      production:
        - DATABASE_URL: Production DB 接続文字列
        - API_SECRET_KEY: Production API シークレット
        - LAUNCHDARKLY_SDK_KEY: Production フィーチャーフラグ
        - STRIPE_SECRET_KEY: 決済シークレット
```

#### 2.3.2 OIDC (Workload Identity Federation) 設定

```yaml
# GCP Workload Identity Federation
# Keyless 認証によるセキュリティ向上

# 1. GCP側設定（Terraform）
oidc_configuration:
  workload_identity_pool:
    name: github-actions-pool
    project: triptrip-production

  provider:
    name: github-provider
    issuer_uri: https://token.actions.githubusercontent.com
    attribute_mapping:
      google.subject: assertion.sub
      attribute.actor: assertion.actor
      attribute.repository: assertion.repository
      attribute.repository_owner: assertion.repository_owner

  service_account_binding:
    service_account: github-actions@triptrip-production.iam.gserviceaccount.com
    roles:
      - roles/artifactregistry.writer
      - roles/container.developer
      - roles/storage.objectAdmin
    condition:
      title: "Allow TripTrip repo only"
      expression: |
        assertion.repository == "triptrip/triptrip-app"
```

```yaml
# 2. GitHub Actions ワークフロー内での使用
# .github/workflows/deploy.yaml
name: Deploy with OIDC

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write  # OIDC トークン取得に必要

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/123456789/locations/global/workloadIdentityPools/github-actions-pool/providers/github-provider
          service_account: github-actions@triptrip-production.iam.gserviceaccount.com

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2

      - name: Configure Docker
        run: gcloud auth configure-docker asia-northeast1-docker.pkg.dev

      - name: Push to Artifact Registry
        run: |
          docker push asia-northeast1-docker.pkg.dev/triptrip-production/triptrip/backend:${{ github.sha }}
```

#### 2.3.3 シークレットローテーション

```yaml
secret_rotation:
  schedule:
    api_keys:
      frequency: 90 days
      automation: semi-automatic
      notification: 14 days before expiry

    database_credentials:
      frequency: 60 days
      automation: automatic
      method: AWS Secrets Manager rotation

    signing_certificates:
      frequency: 365 days
      automation: manual
      notification: 30 days before expiry

  rotation_workflow:
    name: Secret Rotation Check
    schedule: "0 9 * * 1"  # 毎週月曜 9:00
    steps:
      - check_expiry: "API キー有効期限確認"
      - notify_slack: "期限切れ間近のシークレット通知"
      - create_issue: "ローテーションタスク作成"
```

---

## 第3章 パイプラインステージ実装仕様

### 3.1 Flutter/Dart パイプライン

#### 3.1.1 Flutter CI ワークフロー完全版

```yaml
# .github/workflows/ci-flutter.yaml
name: Flutter CI Pipeline

on:
  push:
    branches: [main, develop, 'feature/**', 'hotfix/**']
    paths:
      - 'app/**'
      - '.github/workflows/ci-flutter.yaml'
  pull_request:
    branches: [main, develop]
    paths:
      - 'app/**'
  workflow_dispatch:

env:
  FLUTTER_VERSION: "3.24.0"
  JAVA_VERSION: "17"
  RUBY_VERSION: "3.2"

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # ========================================
  # Stage 1: Code Quality
  # ========================================
  analyze:
    name: Analyze & Format
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true
          cache-key: flutter-${{ env.FLUTTER_VERSION }}-${{ hashFiles('app/pubspec.lock') }}

      - name: Get dependencies
        working-directory: app
        run: flutter pub get

      - name: Generate code
        working-directory: app
        run: dart run build_runner build --delete-conflicting-outputs

      - name: Verify formatting
        working-directory: app
        run: dart format --set-exit-if-changed .

      - name: Analyze code
        working-directory: app
        run: flutter analyze --fatal-infos --fatal-warnings

      - name: Check for unused dependencies
        working-directory: app
        run: |
          dart pub global activate dependency_validator
          dart pub global run dependency_validator

  # ========================================
  # Stage 2: Unit Tests
  # ========================================
  test-unit:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: analyze
    timeout-minutes: 15
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true
          cache-key: flutter-${{ env.FLUTTER_VERSION }}-${{ hashFiles('app/pubspec.lock') }}

      - name: Get dependencies
        working-directory: app
        run: flutter pub get

      - name: Generate code
        working-directory: app
        run: dart run build_runner build --delete-conflicting-outputs

      - name: Run unit tests with coverage
        working-directory: app
        run: |
          flutter test --coverage --reporter=github

      - name: Check coverage threshold
        working-directory: app
        run: |
          # lcov をインストール
          sudo apt-get update && sudo apt-get install -y lcov

          # カバレッジを計算
          COVERAGE=$(lcov --summary coverage/lcov.info 2>&1 | grep "lines" | awk '{print $2}' | sed 's/%//')
          echo "Current coverage: ${COVERAGE}%"

          # 閾値チェック（80%）
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "::error::Coverage ${COVERAGE}% is below 80% threshold"
            exit 1
          fi

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: app/coverage/lcov.info
          flags: flutter-unit
          fail_ci_if_error: true

      - name: Upload coverage artifact
        uses: actions/upload-artifact@v4
        with:
          name: flutter-coverage
          path: app/coverage/
          retention-days: 7

  # ========================================
  # Stage 3: Widget Tests
  # ========================================
  test-widget:
    name: Widget Tests
    runs-on: ubuntu-latest
    needs: analyze
    timeout-minutes: 15
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true

      - name: Get dependencies
        working-directory: app
        run: flutter pub get

      - name: Generate code
        working-directory: app
        run: dart run build_runner build --delete-conflicting-outputs

      - name: Run widget tests
        working-directory: app
        run: |
          flutter test test/widget/ --reporter=github

  # ========================================
  # Stage 4: Build (Matrix)
  # ========================================
  build:
    name: Build ${{ matrix.platform }}
    runs-on: ${{ matrix.os }}
    needs: [test-unit, test-widget]
    timeout-minutes: 30
    strategy:
      fail-fast: false
      matrix:
        include:
          - platform: android
            os: ubuntu-latest
            build-cmd: flutter build apk --release --split-per-abi
            artifact-path: app/build/app/outputs/flutter-apk/

          - platform: android-aab
            os: ubuntu-latest
            build-cmd: flutter build appbundle --release
            artifact-path: app/build/app/outputs/bundle/release/

          - platform: ios
            os: macos-latest
            build-cmd: flutter build ios --release --no-codesign
            artifact-path: app/build/ios/iphoneos/

          - platform: web
            os: ubuntu-latest
            build-cmd: flutter build web --release --web-renderer canvaskit
            artifact-path: app/build/web/

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true

      # Android specific setup
      - name: Setup Java
        if: contains(matrix.platform, 'android')
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: ${{ env.JAVA_VERSION }}
          cache: gradle

      - name: Setup Gradle Cache
        if: contains(matrix.platform, 'android')
        uses: gradle/actions/setup-gradle@v3
        with:
          cache-read-only: ${{ github.ref != 'refs/heads/main' }}

      # iOS specific setup
      - name: Setup Ruby
        if: matrix.platform == 'ios'
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: ${{ env.RUBY_VERSION }}
          bundler-cache: true
          working-directory: app/ios

      - name: Setup CocoaPods Cache
        if: matrix.platform == 'ios'
        uses: actions/cache@v4
        with:
          path: app/ios/Pods
          key: pods-${{ runner.os }}-${{ hashFiles('app/ios/Podfile.lock') }}
          restore-keys: |
            pods-${{ runner.os }}-

      - name: Install CocoaPods
        if: matrix.platform == 'ios'
        working-directory: app/ios
        run: pod install --repo-update

      # Common build steps
      - name: Get dependencies
        working-directory: app
        run: flutter pub get

      - name: Generate code
        working-directory: app
        run: dart run build_runner build --delete-conflicting-outputs

      - name: Build
        working-directory: app
        run: ${{ matrix.build-cmd }}

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: flutter-build-${{ matrix.platform }}
          path: ${{ matrix.artifact-path }}
          retention-days: 7
          compression-level: 9

  # ========================================
  # Stage 5: Integration Tests
  # ========================================
  test-integration:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: [build]
    if: github.event_name == 'pull_request' || github.ref == 'refs/heads/main'
    timeout-minutes: 30
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true

      - name: Get dependencies
        working-directory: app
        run: flutter pub get

      - name: Generate code
        working-directory: app
        run: dart run build_runner build --delete-conflicting-outputs

      - name: Enable Linux Desktop
        run: flutter config --enable-linux-desktop

      - name: Install Linux dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            clang cmake ninja-build pkg-config \
            libgtk-3-dev liblzma-dev libstdc++-12-dev

      - name: Start backend services
        run: |
          docker-compose -f docker-compose.test.yaml up -d
          sleep 10
          docker-compose -f docker-compose.test.yaml ps

      - name: Run integration tests
        working-directory: app
        run: |
          flutter test integration_test/ \
            --reporter=github \
            --timeout 300s

      - name: Stop backend services
        if: always()
        run: docker-compose -f docker-compose.test.yaml down

  # ========================================
  # Quality Gate Summary
  # ========================================
  quality-gate:
    name: Quality Gate
    runs-on: ubuntu-latest
    needs: [analyze, test-unit, test-widget, build]
    if: always()
    steps:
      - name: Check all jobs status
        run: |
          if [[ "${{ needs.analyze.result }}" != "success" ]]; then
            echo "::error::Analyze job failed"
            exit 1
          fi
          if [[ "${{ needs.test-unit.result }}" != "success" ]]; then
            echo "::error::Unit tests failed"
            exit 1
          fi
          if [[ "${{ needs.test-widget.result }}" != "success" ]]; then
            echo "::error::Widget tests failed"
            exit 1
          fi
          if [[ "${{ needs.build.result }}" != "success" ]]; then
            echo "::error::Build failed"
            exit 1
          fi
          echo "::notice::All quality gates passed!"
```

### 3.2 Node.js/TypeScript パイプライン

#### 3.2.1 Backend CI ワークフロー完全版

```yaml
# .github/workflows/ci-backend.yaml
name: Backend CI Pipeline

on:
  push:
    branches: [main, develop, 'feature/**', 'hotfix/**']
    paths:
      - 'backend/**'
      - '.github/workflows/ci-backend.yaml'
  pull_request:
    branches: [main, develop]
    paths:
      - 'backend/**'
  workflow_dispatch:

env:
  NODE_VERSION: "20"
  POSTGRES_VERSION: "16"

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # ========================================
  # Stage 1: Code Quality
  # ========================================
  lint:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm
          cache-dependency-path: backend/package-lock.json

      - name: Install dependencies
        working-directory: backend
        run: npm ci

      - name: Generate Prisma Client
        working-directory: backend
        run: npx prisma generate

      - name: Run ESLint
        working-directory: backend
        run: npm run lint -- --format=github

      - name: Run TypeScript type check
        working-directory: backend
        run: npm run type-check

      - name: Check Prisma schema
        working-directory: backend
        run: npx prisma validate

  # ========================================
  # Stage 2: Unit Tests
  # ========================================
  test-unit:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: lint
    timeout-minutes: 15
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm
          cache-dependency-path: backend/package-lock.json

      - name: Install dependencies
        working-directory: backend
        run: npm ci

      - name: Generate Prisma Client
        working-directory: backend
        run: npx prisma generate

      - name: Run unit tests with coverage
        working-directory: backend
        run: npm run test:unit -- --coverage --reporter=github-actions

      - name: Check coverage threshold
        working-directory: backend
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          echo "Current coverage: ${COVERAGE}%"
          if (( $(echo "$COVERAGE < 85" | bc -l) )); then
            echo "::error::Coverage ${COVERAGE}% is below 85% threshold"
            exit 1
          fi

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: backend/coverage/lcov.info
          flags: backend-unit
          fail_ci_if_error: true

  # ========================================
  # Stage 3: Integration Tests
  # ========================================
  test-integration:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: lint
    timeout-minutes: 20
    services:
      postgres:
        image: postgres:${{ env.POSTGRES_VERSION }}
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

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm
          cache-dependency-path: backend/package-lock.json

      - name: Install dependencies
        working-directory: backend
        run: npm ci

      - name: Generate Prisma Client
        working-directory: backend
        run: npx prisma generate

      - name: Run migrations
        working-directory: backend
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/triptrip_test

      - name: Seed test data
        working-directory: backend
        run: npx prisma db seed
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/triptrip_test

      - name: Run integration tests
        working-directory: backend
        run: npm run test:integration -- --reporter=github-actions
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/triptrip_test
          REDIS_URL: redis://localhost:6379
          NODE_ENV: test

  # ========================================
  # Stage 4: Build
  # ========================================
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [test-unit, test-integration]
    timeout-minutes: 10
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm
          cache-dependency-path: backend/package-lock.json

      - name: Install dependencies
        working-directory: backend
        run: npm ci

      - name: Generate Prisma Client
        working-directory: backend
        run: npx prisma generate

      - name: Build
        working-directory: backend
        run: npm run build

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: backend-build
          path: |
            backend/dist/
            backend/prisma/
            backend/package.json
            backend/package-lock.json
          retention-days: 7

  # ========================================
  # Stage 5: Docker Build
  # ========================================
  docker:
    name: Docker Build
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    timeout-minutes: 15
    permissions:
      contents: read
      id-token: write
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
      image_digest: ${{ steps.build.outputs.digest }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Authenticate to Google Cloud
        id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ vars.GCP_SERVICE_ACCOUNT }}

      - name: Configure Docker for Artifact Registry
        run: gcloud auth configure-docker asia-northeast1-docker.pkg.dev

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: |
            asia-northeast1-docker.pkg.dev/${{ vars.GCP_PROJECT_ID }}/triptrip/backend
          tags: |
            type=sha,prefix={{branch}}-
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          file: ./backend/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64
          build-args: |
            NODE_ENV=production
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            VCS_REF=${{ github.sha }}

      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          image: ${{ steps.meta.outputs.tags }}
          format: spdx-json
          output-file: sbom.spdx.json

      - name: Upload SBOM
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.spdx.json

  # ========================================
  # Security Scan
  # ========================================
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: docker
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    timeout-minutes: 15
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ needs.docker.outputs.image_tag }}
          format: table
          exit-code: 1
          ignore-unfixed: true
          vuln-type: os,library
          severity: CRITICAL,HIGH

      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/docker@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          image: ${{ needs.docker.outputs.image_tag }}
          args: --severity-threshold=high

  # ========================================
  # Quality Gate
  # ========================================
  quality-gate:
    name: Quality Gate
    runs-on: ubuntu-latest
    needs: [lint, test-unit, test-integration, build]
    if: always()
    steps:
      - name: Check all jobs status
        run: |
          if [[ "${{ needs.lint.result }}" != "success" ]]; then
            echo "::error::Lint job failed"
            exit 1
          fi
          if [[ "${{ needs.test-unit.result }}" != "success" ]]; then
            echo "::error::Unit tests failed"
            exit 1
          fi
          if [[ "${{ needs.test-integration.result }}" != "success" ]]; then
            echo "::error::Integration tests failed"
            exit 1
          fi
          if [[ "${{ needs.build.result }}" != "success" ]]; then
            echo "::error::Build failed"
            exit 1
          fi
          echo "::notice::All quality gates passed!"
```

### 3.3 共通ステージ（security、deploy）

#### 3.3.1 セキュリティスキャン統合ワークフロー

```yaml
# .github/workflows/security-scan.yaml
name: Security Scan

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 3 * * *'  # 毎日 AM 3:00 UTC
  workflow_dispatch:

permissions:
  contents: read
  security-events: write

jobs:
  # ========================================
  # SAST (Static Application Security Testing)
  # ========================================
  sast:
    name: SAST Scan
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/secrets
            p/owasp-top-ten
            p/typescript
            p/flutter
        env:
          SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}

      - name: Run CodeQL Analysis
        uses: github/codeql-action/init@v3
        with:
          languages: javascript, typescript

      - name: Autobuild
        uses: github/codeql-action/autobuild@v3

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: "/language:javascript"

  # ========================================
  # Secret Detection
  # ========================================
  secret-scan:
    name: Secret Detection
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run TruffleHog
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
          extra_args: --only-verified

      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # ========================================
  # Dependency Vulnerability Scan
  # ========================================
  dependency-scan:
    name: Dependency Scan
    runs-on: ubuntu-latest
    timeout-minutes: 15
    strategy:
      matrix:
        project:
          - path: backend
            type: npm
          - path: app
            type: pub
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Run Snyk for npm
        if: matrix.project.type == 'npm'
        uses: snyk/actions/node@master
        with:
          args: --all-projects --severity-threshold=high
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

      - name: Run OSV Scanner
        uses: google/osv-scanner-action@v1
        with:
          scan-args: |-
            --lockfile=${{ matrix.project.path }}/package-lock.json
            --lockfile=${{ matrix.project.path }}/pubspec.lock

  # ========================================
  # Container Scan
  # ========================================
  container-scan:
    name: Container Scan
    runs-on: ubuntu-latest
    timeout-minutes: 15
    if: github.event_name != 'pull_request'
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build image for scanning
        run: |
          docker build -t triptrip-backend:scan ./backend

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: triptrip-backend:scan
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH,MEDIUM
          vuln-type: os,library

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif

      - name: Run Docker Scout
        uses: docker/scout-action@v1
        with:
          command: cves
          image: triptrip-backend:scan
          sarif-file: scout-results.sarif
          only-severities: critical,high

  # ========================================
  # License Compliance
  # ========================================
  license-check:
    name: License Compliance
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Check npm licenses
        working-directory: backend
        run: |
          npm ci
          npx license-checker --summary --onlyAllow "MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC;CC0-1.0;CC-BY-3.0;CC-BY-4.0;Unlicense;Python-2.0"

      - name: Check Dart licenses
        working-directory: app
        run: |
          dart pub get
          dart run pana --no-warning --exit-code-threshold 0

  # ========================================
  # Infrastructure Security (IaC Scan)
  # ========================================
  iac-scan:
    name: Infrastructure Security Scan
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Run Checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: infrastructure/
          framework: terraform,kubernetes,dockerfile
          soft_fail: false
          output_format: sarif
          output_file_path: checkov-results.sarif

      - name: Upload Checkov results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: checkov-results.sarif

      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          working_directory: infrastructure/terraform

  # ========================================
  # Security Summary
  # ========================================
  security-summary:
    name: Security Summary
    runs-on: ubuntu-latest
    needs: [sast, secret-scan, dependency-scan, license-check, iac-scan]
    if: always()
    steps:
      - name: Generate security report
        run: |
          echo "## Security Scan Summary" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "| Check | Status |" >> $GITHUB_STEP_SUMMARY
          echo "|-------|--------|" >> $GITHUB_STEP_SUMMARY
          echo "| SAST | ${{ needs.sast.result }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Secret Detection | ${{ needs.secret-scan.result }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Dependency Scan | ${{ needs.dependency-scan.result }} |" >> $GITHUB_STEP_SUMMARY
          echo "| License Check | ${{ needs.license-check.result }} |" >> $GITHUB_STEP_SUMMARY
          echo "| IaC Scan | ${{ needs.iac-scan.result }} |" >> $GITHUB_STEP_SUMMARY

      - name: Check for failures
        if: contains(needs.*.result, 'failure')
        run: |
          echo "::error::One or more security checks failed"
          exit 1
```

---

## 第4章 品質ゲートと自動化

### 4.1 コード品質チェック

#### 4.1.1 SonarQube統合

```yaml
# .github/workflows/sonarqube.yaml
name: SonarQube Analysis

on:
  push:
    branches: [main, develop]
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  sonarqube:
    name: SonarQube Scan
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # Backend analysis
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install backend dependencies
        working-directory: backend
        run: npm ci

      - name: Run backend tests with coverage
        working-directory: backend
        run: npm run test:coverage

      # Flutter analysis
      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: 3.24.0

      - name: Run Flutter tests with coverage
        working-directory: app
        run: |
          flutter pub get
          flutter test --coverage

      # SonarQube scan
      - name: SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
        with:
          args: >
            -Dsonar.projectKey=triptrip
            -Dsonar.sources=backend/src,app/lib
            -Dsonar.tests=backend/tests,app/test
            -Dsonar.javascript.lcov.reportPaths=backend/coverage/lcov.info
            -Dsonar.dart.lcov.reportPaths=app/coverage/lcov.info
            -Dsonar.qualitygate.wait=true

      - name: SonarQube Quality Gate check
        uses: SonarSource/sonarqube-quality-gate-action@master
        timeout-minutes: 5
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

#### 4.1.2 品質ゲート定義

```yaml
# sonar-project.properties
sonar.projectKey=triptrip
sonar.projectName=TripTrip
sonar.projectVersion=1.0

# Quality Gate conditions
sonar.qualitygate.conditions:
  # Coverage
  - metric: coverage
    op: LT
    error: 80

  # Duplications
  - metric: duplicated_lines_density
    op: GT
    error: 3

  # Maintainability
  - metric: sqale_rating
    op: GT
    error: 1  # A rating

  # Reliability
  - metric: reliability_rating
    op: GT
    error: 1  # A rating

  # Security
  - metric: security_rating
    op: GT
    error: 1  # A rating

  # Security Hotspots
  - metric: security_hotspots_reviewed
    op: LT
    error: 100

  # New code conditions (for PRs)
  - metric: new_coverage
    op: LT
    error: 80

  - metric: new_duplicated_lines_density
    op: GT
    error: 3

  - metric: new_maintainability_rating
    op: GT
    error: 1

  - metric: new_reliability_rating
    op: GT
    error: 1

  - metric: new_security_rating
    op: GT
    error: 1
```

### 4.2 セキュリティスキャン

#### 4.2.1 セキュリティスキャン設定

```yaml
# .github/security-config.yaml
security_scanning:
  sast:
    tools:
      - name: semgrep
        config: p/security-audit,p/secrets,p/owasp-top-ten
        severity_threshold: warning

      - name: codeql
        languages: [javascript, typescript]
        queries: security-extended

  dependency_scanning:
    tools:
      - name: snyk
        severity_threshold: high
        fail_on: upgradable

      - name: dependabot
        interval: daily
        open_pull_requests_limit: 10

  container_scanning:
    tools:
      - name: trivy
        severity: [CRITICAL, HIGH]
        ignore_unfixed: true

      - name: grype
        fail_on_severity: high

  secret_detection:
    tools:
      - name: trufflehog
        only_verified: true

      - name: gitleaks
        config: .gitleaks.toml

  compliance:
    license_allowlist:
      - MIT
      - Apache-2.0
      - BSD-2-Clause
      - BSD-3-Clause
      - ISC
      - CC0-1.0
```

### 4.3 パフォーマンステスト統合

#### 4.3.1 パフォーマンステストワークフロー

```yaml
# .github/workflows/performance-test.yaml
name: Performance Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
    types: [opened, synchronize]
  schedule:
    - cron: '0 4 * * *'  # 毎日 AM 4:00 UTC
  workflow_dispatch:
    inputs:
      duration:
        description: 'Test duration in seconds'
        required: false
        default: '60'
      vus:
        description: 'Number of virtual users'
        required: false
        default: '50'

env:
  K6_VERSION: "0.49.0"

jobs:
  # ========================================
  # API Load Test
  # ========================================
  load-test:
    name: API Load Test
    runs-on: ubuntu-latest
    timeout-minutes: 30
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: triptrip_test
        ports:
          - 5432:5432

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install backend dependencies
        working-directory: backend
        run: npm ci

      - name: Setup database
        working-directory: backend
        run: |
          npx prisma migrate deploy
          npx prisma db seed
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/triptrip_test

      - name: Start backend server
        working-directory: backend
        run: npm run start:test &
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/triptrip_test
          REDIS_URL: redis://localhost:6379
          PORT: 3001

      - name: Wait for server
        run: |
          timeout 30 bash -c 'until curl -s http://localhost:3001/health; do sleep 1; done'

      - name: Install k6
        run: |
          sudo gpg -k
          sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
            --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
          echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" \
            | sudo tee /etc/apt/sources.list.d/k6.list
          sudo apt-get update
          sudo apt-get install k6

      - name: Run load tests
        run: |
          k6 run \
            --out json=results.json \
            --summary-export=summary.json \
            -e BASE_URL=http://localhost:3001 \
            -e DURATION=${{ github.event.inputs.duration || '60' }}s \
            -e VUS=${{ github.event.inputs.vus || '50' }} \
            tests/performance/load-test.js

      - name: Check thresholds
        run: |
          # Parse results and check thresholds
          python3 << 'EOF'
          import json
          import sys

          with open('summary.json') as f:
              summary = json.load(f)

          thresholds = {
              'http_req_duration': {'p95': 500, 'p99': 1000},  # ms
              'http_req_failed': {'rate': 0.01},  # 1%
              'http_reqs': {'rate': 100},  # requests/second
          }

          failed = False
          for metric, limits in thresholds.items():
              if metric in summary['metrics']:
                  values = summary['metrics'][metric]['values']
                  for key, limit in limits.items():
                      actual = values.get(key, values.get('rate', 0))
                      if key == 'rate' and metric == 'http_req_failed':
                          if actual > limit:
                              print(f"❌ {metric}.{key}: {actual:.4f} > {limit}")
                              failed = True
                          else:
                              print(f"✅ {metric}.{key}: {actual:.4f} <= {limit}")
                      elif actual > limit:
                          print(f"❌ {metric}.{key}: {actual:.2f} > {limit}")
                          failed = True
                      else:
                          print(f"✅ {metric}.{key}: {actual:.2f} <= {limit}")

          if failed:
              sys.exit(1)
          EOF

      - name: Upload results
        uses: actions/upload-artifact@v4
        with:
          name: performance-results
          path: |
            results.json
            summary.json
          retention-days: 30

      - name: Post results to PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const summary = JSON.parse(fs.readFileSync('summary.json', 'utf8'));

            const metrics = summary.metrics;
            const body = `## 📊 Performance Test Results

            | Metric | p50 | p95 | p99 | Max |
            |--------|-----|-----|-----|-----|
            | Response Time | ${metrics.http_req_duration.values.p50?.toFixed(2) || 'N/A'}ms | ${metrics.http_req_duration.values.p95?.toFixed(2) || 'N/A'}ms | ${metrics.http_req_duration.values.p99?.toFixed(2) || 'N/A'}ms | ${metrics.http_req_duration.values.max?.toFixed(2) || 'N/A'}ms |

            | Metric | Value |
            |--------|-------|
            | Total Requests | ${metrics.http_reqs.values.count} |
            | Request Rate | ${metrics.http_reqs.values.rate?.toFixed(2)}/s |
            | Failed Requests | ${(metrics.http_req_failed.values.rate * 100).toFixed(2)}% |

            <details>
            <summary>Detailed Results</summary>

            \`\`\`json
            ${JSON.stringify(summary, null, 2)}
            \`\`\`

            </details>
            `;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: body
            });
```

#### 4.3.2 k6 パフォーマンステストスクリプト

```javascript
// tests/performance/load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const apiLatency = new Trend('api_latency');

// Test configuration
export const options = {
  stages: [
    { duration: '30s', target: parseInt(__ENV.VUS) || 50 },   // Ramp up
    { duration: __ENV.DURATION || '60s', target: parseInt(__ENV.VUS) || 50 }, // Stay
    { duration: '30s', target: 0 },   // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.01'],
    errors: ['rate<0.01'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3001';

// Test scenarios
export default function () {
  // Health check
  const healthRes = http.get(`${BASE_URL}/health`);
  check(healthRes, {
    'health check status is 200': (r) => r.status === 200,
  });

  // Products list
  const productsRes = http.get(`${BASE_URL}/api/v1/products?limit=20`);
  check(productsRes, {
    'products status is 200': (r) => r.status === 200,
    'products response time < 200ms': (r) => r.timings.duration < 200,
  });
  apiLatency.add(productsRes.timings.duration);
  errorRate.add(productsRes.status !== 200);

  sleep(1);

  // Product detail
  const productDetailRes = http.get(`${BASE_URL}/api/v1/products/1`);
  check(productDetailRes, {
    'product detail status is 200': (r) => r.status === 200,
    'product detail response time < 100ms': (r) => r.timings.duration < 100,
  });
  apiLatency.add(productDetailRes.timings.duration);
  errorRate.add(productDetailRes.status !== 200);

  sleep(1);

  // Search
  const searchRes = http.get(`${BASE_URL}/api/v1/search?q=tokyo&type=hotel`);
  check(searchRes, {
    'search status is 200': (r) => r.status === 200,
    'search response time < 300ms': (r) => r.timings.duration < 300,
  });
  apiLatency.add(searchRes.timings.duration);
  errorRate.add(searchRes.status !== 200);

  sleep(1);
}

// Setup function
export function setup() {
  // Verify API is available
  const res = http.get(`${BASE_URL}/health`);
  if (res.status !== 200) {
    throw new Error(`API is not available: ${res.status}`);
  }
  console.log('API is ready');
}

// Teardown function
export function teardown(data) {
  console.log('Test completed');
}
```

---

## 第5章 高度な機能と最適化

### 5.1 キャッシュ戦略

#### 5.1.1 多層キャッシュアーキテクチャ

```yaml
cache_architecture:
  layers:
    # Layer 1: GitHub Actions Cache
    github_actions_cache:
      description: "ビルドアーティファクトとパッケージキャッシュ"
      max_size: 10GB
      retention: 7 days
      types:
        - dependencies
        - build_outputs
        - tool_cache

    # Layer 2: Docker Layer Cache
    docker_layer_cache:
      description: "Docker ビルドレイヤーキャッシュ"
      backend: GitHub Actions Cache (gha)
      modes:
        - max: 全レイヤーキャッシュ
        - min: 最終レイヤーのみ

    # Layer 3: Artifact Registry Cache
    registry_cache:
      description: "ベースイメージのローカルミラー"
      location: asia-northeast1
      sync_interval: daily
```

#### 5.1.2 キャッシュ実装詳細

```yaml
# Flutter キャッシュ設定
flutter_cache:
  pub_cache:
    key: flutter-pub-${{ runner.os }}-${{ hashFiles('app/pubspec.lock') }}
    restore_keys:
      - flutter-pub-${{ runner.os }}-
    paths:
      - ~/.pub-cache

  flutter_sdk:
    key: flutter-sdk-${{ env.FLUTTER_VERSION }}
    paths:
      - ~/.flutter

  build_cache:
    key: flutter-build-${{ runner.os }}-${{ github.sha }}
    restore_keys:
      - flutter-build-${{ runner.os }}-
    paths:
      - app/build/
      - app/.dart_tool/

  gradle_cache:
    key: gradle-${{ runner.os }}-${{ hashFiles('app/android/build.gradle', 'app/android/app/build.gradle') }}
    restore_keys:
      - gradle-${{ runner.os }}-
    paths:
      - ~/.gradle/caches
      - ~/.gradle/wrapper

  cocoapods_cache:
    key: pods-${{ runner.os }}-${{ hashFiles('app/ios/Podfile.lock') }}
    restore_keys:
      - pods-${{ runner.os }}-
    paths:
      - app/ios/Pods
      - app/ios/.symlinks

# Node.js キャッシュ設定
nodejs_cache:
  npm_cache:
    key: npm-${{ runner.os }}-${{ hashFiles('backend/package-lock.json') }}
    restore_keys:
      - npm-${{ runner.os }}-
    paths:
      - ~/.npm

  node_modules:
    key: node-modules-${{ runner.os }}-${{ hashFiles('backend/package-lock.json') }}
    restore_keys:
      - node-modules-${{ runner.os }}-
    paths:
      - backend/node_modules

  prisma_cache:
    key: prisma-${{ runner.os }}-${{ hashFiles('backend/prisma/schema.prisma') }}
    paths:
      - backend/node_modules/.prisma

  typescript_cache:
    key: tsc-${{ runner.os }}-${{ github.sha }}
    restore_keys:
      - tsc-${{ runner.os }}-
    paths:
      - backend/.tsbuildinfo
      - backend/dist/
```

#### 5.1.3 キャッシュヘルパーアクション

```yaml
# .github/actions/setup-flutter/action.yaml
name: Setup Flutter
description: Flutter環境のセットアップとキャッシュ

inputs:
  flutter-version:
    description: Flutter version
    required: true
    default: '3.24.0'
  cache-key-suffix:
    description: Additional cache key suffix
    required: false
    default: ''

runs:
  using: composite
  steps:
    - name: Setup Flutter
      uses: subosito/flutter-action@v2
      with:
        flutter-version: ${{ inputs.flutter-version }}
        cache: true
        cache-key: flutter-${{ inputs.flutter-version }}-${{ inputs.cache-key-suffix }}

    - name: Restore pub cache
      uses: actions/cache@v4
      with:
        path: ~/.pub-cache
        key: pub-cache-${{ runner.os }}-${{ hashFiles('app/pubspec.lock') }}
        restore-keys: |
          pub-cache-${{ runner.os }}-

    - name: Restore build cache
      uses: actions/cache@v4
      with:
        path: |
          app/build/
          app/.dart_tool/
        key: flutter-build-${{ runner.os }}-${{ hashFiles('app/lib/**/*.dart') }}
        restore-keys: |
          flutter-build-${{ runner.os }}-

    - name: Get dependencies
      shell: bash
      working-directory: app
      run: flutter pub get

    - name: Generate code
      shell: bash
      working-directory: app
      run: dart run build_runner build --delete-conflicting-outputs
```

### 5.2 並列実行最適化

#### 5.2.1 Matrix ビルド戦略

```yaml
parallel_strategy:
  flutter_builds:
    matrix:
      platform: [android, ios, web]
      flutter_version: ['3.24.0']
    parallelism: 3
    total_time_reduction: "~66%"

  backend_tests:
    sharding:
      enabled: true
      count: 4
      distribution: "by_timing"
    total_time_reduction: "~75%"

  integration_tests:
    parallelism:
      android_devices: 2
      ios_simulators: 2
    total_time_reduction: "~50%"
```

#### 5.2.2 テストシャーディング実装

```yaml
# バックエンドテストシャーディング
test-backend-sharded:
  name: Backend Tests (Shard ${{ matrix.shard }})
  runs-on: ubuntu-latest
  strategy:
    fail-fast: false
    matrix:
      shard: [1, 2, 3, 4]
      total_shards: [4]
  steps:
    - uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: 20
        cache: npm
        cache-dependency-path: backend/package-lock.json

    - name: Install dependencies
      working-directory: backend
      run: npm ci

    - name: Run sharded tests
      working-directory: backend
      run: |
        npm run test -- \
          --shard=${{ matrix.shard }}/${{ matrix.total_shards }} \
          --reporter=github-actions \
          --coverage
      env:
        VITEST_SHARD: ${{ matrix.shard }}/${{ matrix.total_shards }}

    - name: Upload coverage
      uses: actions/upload-artifact@v4
      with:
        name: coverage-shard-${{ matrix.shard }}
        path: backend/coverage/

# カバレッジマージジョブ
merge-coverage:
  name: Merge Coverage
  runs-on: ubuntu-latest
  needs: test-backend-sharded
  steps:
    - name: Download all coverage artifacts
      uses: actions/download-artifact@v4
      with:
        pattern: coverage-shard-*
        merge-multiple: true
        path: coverage/

    - name: Merge coverage reports
      run: |
        npx nyc merge coverage/ merged-coverage.json
        npx nyc report --reporter=lcov --temp-dir=coverage/

    - name: Upload to Codecov
      uses: codecov/codecov-action@v4
      with:
        files: coverage/lcov.info
```

### 5.3 コスト最適化

#### 5.3.1 ランナー使用最適化

```yaml
cost_optimization:
  strategies:
    # 1. Concurrency limits
    concurrency_limits:
      per_workflow: 3
      per_branch: 1
      cancel_in_progress: true

    # 2. Conditional execution
    conditional_execution:
      path_filtering:
        - frontend_only: 'app/**'
        - backend_only: 'backend/**'
        - docs_only: 'docs/**'
      skip_ci:
        - commit_patterns: ['[skip ci]', '[ci skip]', 'chore(docs):']

    # 3. Runner selection
    runner_optimization:
      light_tasks:
        runner: ubuntu-latest
        tasks: [lint, format, static-analysis]
      heavy_tasks:
        runner: self-hosted
        tasks: [e2e-tests, performance-tests]
      ios_only:
        runner: macos-latest
        tasks: [ios-build]

    # 4. Cache maximization
    cache_optimization:
      cache_hit_target: "> 80%"
      strategies:
        - use_lock_files_in_keys
        - hierarchical_restore_keys
        - separate_build_from_deps

    # 5. Job timeout limits
    timeout_limits:
      lint: 10 minutes
      unit_tests: 15 minutes
      integration_tests: 20 minutes
      build: 30 minutes
      deploy: 15 minutes

  estimated_savings:
    before_optimization: "$500/month"
    after_optimization: "$200/month"
    reduction: "60%"
```

#### 5.3.2 GitHub Actions 使用量モニタリング

```yaml
# .github/workflows/usage-report.yaml
name: Usage Report

on:
  schedule:
    - cron: '0 0 1 * *'  # 毎月1日
  workflow_dispatch:

jobs:
  usage-report:
    runs-on: ubuntu-latest
    steps:
      - name: Get usage statistics
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.ADMIN_TOKEN }}
          script: |
            const { data: billing } = await github.rest.billing.getGithubActionsBillingOrg({
              org: context.repo.owner
            });

            const report = `
            ## GitHub Actions Usage Report

            | Metric | Value |
            |--------|-------|
            | Total Minutes Used | ${billing.total_minutes_used} |
            | Total Paid Minutes Used | ${billing.total_paid_minutes_used} |
            | Included Minutes | ${billing.included_minutes} |

            ### Minutes by OS
            | OS | Minutes |
            |-----|---------|
            | Linux | ${billing.minutes_used_breakdown.UBUNTU} |
            | macOS | ${billing.minutes_used_breakdown.MACOS} |
            | Windows | ${billing.minutes_used_breakdown.WINDOWS} |
            `;

            console.log(report);

            // Post to Slack
            await fetch(process.env.SLACK_WEBHOOK_URL, {
              method: 'POST',
              headers: { 'Content-Type': 'application/json' },
              body: JSON.stringify({
                text: report,
                channel: '#devops-reports'
              })
            });
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 第6章 実装ロードマップ

### 6.1 Phase 1: 基本パイプライン強化

```yaml
phase_1:
  timeline: Month 1-2
  objectives:
    - CI/CDパイプラインの標準化
    - 基本的な品質ゲート導入
    - キャッシュ戦略の実装

  deliverables:
    - name: Flutter CI パイプライン
      file: .github/workflows/ci-flutter.yaml
      features:
        - コード分析・フォーマット
        - ユニットテスト・カバレッジ
        - マルチプラットフォームビルド
      status: 実装予定

    - name: Backend CI パイプライン
      file: .github/workflows/ci-backend.yaml
      features:
        - Lint・型チェック
        - ユニット・統合テスト
        - Dockerビルド
      status: 実装予定

    - name: 品質ゲート
      features:
        - カバレッジ閾値（80%）
        - Lint エラー0件
        - セキュリティスキャン
      status: 実装予定

  success_criteria:
    - CI実行時間: "< 15分 (PRチェック)"
    - ビルド成功率: "> 95%"
    - カバレッジ: "> 80%"
```

### 6.2 Phase 2: 高度な自動化

```yaml
phase_2:
  timeline: Month 3-4
  objectives:
    - SonarQube統合
    - 包括的セキュリティスキャン
    - パフォーマンステスト統合

  deliverables:
    - name: SonarQube統合
      features:
        - コード品質ダッシュボード
        - 技術的負債追跡
        - セキュリティホットスポット
      status: 計画中

    - name: セキュリティスキャン
      file: .github/workflows/security-scan.yaml
      features:
        - SAST (Semgrep, CodeQL)
        - 依存関係スキャン (Snyk)
        - コンテナスキャン (Trivy)
        - シークレット検出
      status: 計画中

    - name: パフォーマンステスト
      file: .github/workflows/performance-test.yaml
      features:
        - k6負荷テスト
        - レスポンスタイム検証
        - 自動閾値チェック
      status: 計画中

  success_criteria:
    - SonarQube Quality Gate: "Pass"
    - セキュリティスキャン: "Critical/High 0件"
    - パフォーマンス: "P95 < 500ms"
```

### 6.3 Phase 3: エンタープライズ機能

```yaml
phase_3:
  timeline: Month 5-6
  objectives:
    - OIDC認証統合
    - Self-hosted Runner導入
    - 高度なキャッシュ最適化

  deliverables:
    - name: OIDC認証
      features:
        - GCP Workload Identity Federation
        - Keyless認証
        - 最小権限原則
      status: 計画中

    - name: Self-hosted Runners
      features:
        - Kubernetes上のRunner
        - オートスケーリング
        - GPU対応Runner
      status: 計画中

    - name: キャッシュ最適化
      features:
        - キャッシュヒット率90%以上
        - 多層キャッシュ
        - 増分ビルド
      status: 計画中

  success_criteria:
    - シークレット管理: "Keyless認証100%"
    - キャッシュヒット率: "> 90%"
    - コスト削減: "> 50%"
```

---

## 第7章 文書間参照 & 統合ポイント

### 7.1 関連文書マップ

```yaml
document_references:
  upstream_documents:
    - id: Doc-DM-002
      title: DevOps＆継続的デリバリー戦略
      relationship: "概念・方針の基盤"
      key_integrations:
        - DORAメトリクス目標
        - ブランチ戦略
        - デプロイメントパターン

    - id: EXISTING_APP_ANALYSIS.md
      title: 既存アプリケーション分析
      relationship: "技術スタック定義"
      key_integrations:
        - Flutter/Dart バージョン
        - Node.js/TypeScript バージョン
        - テストフレームワーク

  peer_documents:
    - id: Doc-DO-002
      title: Infrastructure as Code
      relationship: "インフラ管理"
      key_integrations:
        - Terraform モジュール
        - GKE クラスタ設定
        - ネットワーク設定

    - id: Doc-IA-002
      title: Kubernetes & Container Strategy
      relationship: "コンテナ運用"
      key_integrations:
        - Deployment仕様
        - Service設定
        - オートスケーリング

    - id: Doc-QA-003
      title: リリース管理＆ロールバック
      relationship: "リリースプロセス"
      key_integrations:
        - リリースワークフロー
        - ロールバック手順
        - フィーチャーフラグ

  downstream_documents:
    - id: Doc-SC-001
      title: セキュリティアーキテクチャ
      relationship: "セキュリティ要件"
      key_integrations:
        - セキュリティスキャン設定
        - シークレット管理
        - コンプライアンス

    - id: Doc-QA-001
      title: テスト戦略＆品質基準
      relationship: "品質基準"
      key_integrations:
        - テストピラミッド
        - カバレッジ基準
        - 品質ゲート
```

### 7.2 統合ポイント詳細

```yaml
integration_points:
  deployment_pipeline:
    ci_to_staging:
      trigger: "main ブランチへのマージ"
      workflow: ci-backend.yaml → deploy-staging.yaml
      artifacts:
        - Docker イメージ
        - Helm チャート

    staging_to_production:
      trigger: "手動承認 + Canary成功"
      workflow: deploy-staging.yaml → deploy-production.yaml
      gates:
        - Staging テスト成功
        - Canary分析パス
        - 手動承認

  quality_gates:
    code_level:
      source: Doc-QA-001
      implementation: ci-*.yaml
      checks:
        - カバレッジ 80%
        - Lint エラー 0
        - 型チェックパス

    security_level:
      source: Doc-SC-001
      implementation: security-scan.yaml
      checks:
        - SAST パス
        - 依存関係安全
        - シークレット漏洩なし

    performance_level:
      source: Doc-DM-002
      implementation: performance-test.yaml
      checks:
        - P95 < 500ms
        - エラー率 < 1%

  infrastructure_integration:
    terraform_workflow:
      source: Doc-DO-002
      trigger: "infrastructure/** 変更時"
      stages:
        - terraform validate
        - terraform plan
        - terraform apply (承認後)

    kubernetes_deployment:
      source: Doc-IA-002
      method: "Helm + ArgoCD"
      artifacts:
        - Helm チャート
        - Kustomize オーバーレイ
```

---

## 付録A: ワークフローチェックリスト

```markdown
## CI/CD パイプライン実装チェックリスト

### 基本設定
- [ ] ワークフローファイル作成
- [ ] トリガー条件設定
- [ ] 環境変数定義
- [ ] Concurrency設定

### コード品質
- [ ] Lint/Format チェック
- [ ] 静的解析
- [ ] 型チェック
- [ ] コードカバレッジ

### テスト
- [ ] ユニットテスト
- [ ] 統合テスト
- [ ] E2Eテスト
- [ ] パフォーマンステスト

### セキュリティ
- [ ] SAST スキャン
- [ ] 依存関係スキャン
- [ ] コンテナスキャン
- [ ] シークレット検出

### ビルド・デプロイ
- [ ] マルチプラットフォームビルド
- [ ] Docker イメージビルド
- [ ] アーティファクト管理
- [ ] デプロイメント自動化

### 最適化
- [ ] キャッシュ設定
- [ ] 並列実行
- [ ] タイムアウト設定
- [ ] コスト最適化
```

---

## 付録B: トラブルシューティングガイド

```yaml
troubleshooting:
  common_issues:
    - issue: "Flutter ビルドがキャッシュミスで遅い"
      symptoms:
        - pub get に毎回時間がかかる
        - gradle ダウンロードが繰り返される
      solutions:
        - pubspec.lock を含むキャッシュキー使用
        - gradle-wrapper.properties のハッシュ追加
      commands:
        - flutter pub cache repair
        - gradle --stop && gradle clean

    - issue: "Docker ビルドが遅い"
      symptoms:
        - レイヤーキャッシュが効かない
        - ビルド時間が15分以上
      solutions:
        - BuildKitを有効化
        - マルチステージビルド最適化
        - .dockerignore の確認
      commands:
        - docker buildx build --cache-from type=gha --cache-to type=gha,mode=max

    - issue: "テストがフレーキー"
      symptoms:
        - 断続的なテスト失敗
        - タイミング依存のエラー
      solutions:
        - テスト分離の確認
        - 非同期処理のawait確認
        - リトライメカニズム追加
      commands:
        - npm run test -- --retry=3

    - issue: "シークレットアクセスエラー"
      symptoms:
        - 認証失敗
        - Permission denied
      solutions:
        - OIDC設定の確認
        - サービスアカウント権限確認
        - シークレットスコープ確認
      commands:
        - gcloud auth list
        - gcloud iam service-accounts get-iam-policy

    - issue: "Artifact Registry プッシュ失敗"
      symptoms:
        - unauthorized エラー
        - 403 Forbidden
      solutions:
        - Docker 認証設定確認
        - IAM権限確認
        - リポジトリ存在確認
      commands:
        - gcloud auth configure-docker asia-northeast1-docker.pkg.dev
        - gcloud artifacts repositories list
```

---

## 付録C: 変更履歴

| バージョン | 日付 | 変更内容 | 作成者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-01-20 | 初版作成 | Technical Architecture Agent |

---

**文書情報**
- 文書ID: Doc-DO-001
- バージョン: 1.0.0
- ステータス: ドラフト
- 最終更新: 2026-01-20
- 次回レビュー: 2026-02-20
- 行数: 約1,500行

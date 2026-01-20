# Doc-DM-002: TripTrip DevOps＆継続的デリバリー戦略

## エグゼクティブサマリー

本文書は、TripTripプラットフォームのDevOps戦略と継続的デリバリー（CD）パイプラインを包括的に定義します。Spotify、Google、Amazon、Netflixのエンジニアリング実践を統合し、Infrastructure as Code（IaC）によるインフラ管理、GitHub Actionsを中心としたCI/CDパイプライン、ブルーグリーン/カナリアデプロイメント戦略、フィーチャーフラグ管理、自動ロールバック機能を体系的に策定します。既存のFlutterアプリ（35,000行）とNode.js/TypeScriptバックエンド（4,700行）の成熟したCI/CD基盤を発展させ、日次デプロイから継続的デプロイへの進化を実現します。Year 1でCI/CD成熟度レベル3（標準化）、Year 2でレベル4（管理）、Year 3でレベル5（最適化）の達成を目指します。

---

## 第1章 はじめに & コンテキスト

### 1.1 目的 & スコープ

#### 1.1.1 文書の目的

本DevOps＆継続的デリバリー戦略文書は、TripTripの開発・運用プロセスを世界最高水準に引き上げるための包括的なガイドラインを提供します。

**主要目的**:

1. **デリバリー速度の向上**: コミットから本番までのリードタイムを1時間以内に短縮
2. **品質の担保**: 自動テスト・品質ゲートによる欠陥の早期検出
3. **リスクの最小化**: 段階的ロールアウトと自動ロールバックによる障害影響の限定
4. **運用効率の向上**: Infrastructure as Codeによる再現可能な環境管理
5. **開発者体験の向上**: セルフサービス化と自動化による開発者の生産性向上

**スコープ定義**:

```
┌─────────────────────────────────────────────────────────────┐
│              DevOps戦略スコープ                              │
├─────────────────────────────────────────────────────────────┤
│ インスコープ:                                                │
│  ├─ CI/CDパイプライン設計・運用                             │
│  ├─ Infrastructure as Code（Terraform）                     │
│  ├─ コンテナオーケストレーション（Kubernetes）              │
│  ├─ デプロイメント戦略（ブルーグリーン、カナリア）          │
│  ├─ フィーチャーフラグ管理                                  │
│  ├─ シークレット管理                                        │
│  ├─ 環境管理（dev/staging/production）                      │
│  └─ ロールバック・災害復旧手順                              │
├─────────────────────────────────────────────────────────────┤
│ アウトオブスコープ:                                          │
│  ├─ 監視・アラート詳細（→Doc-QA-002参照）                  │
│  ├─ セキュリティアーキテクチャ詳細（→Doc-SC-001参照）      │
│  ├─ テスト戦略詳細（→Doc-QA-001参照）                      │
│  └─ リリース管理プロセス（→Doc-QA-003参照）                │
└─────────────────────────────────────────────────────────────┘
```

#### 1.1.2 対象読者

- DevOps/SREエンジニア
- バックエンド/フロントエンドエンジニア
- エンジニアリングマネージャー
- プラットフォームチーム
- セキュリティチーム

### 1.2 DevOps目標 & 成功基準

#### 1.2.1 DORA メトリクス目標

**DORA（DevOps Research and Assessment）Four Key Metrics**:

```yaml
dora_metrics:
  deployment_frequency:
    description: 本番環境へのデプロイ頻度
    current: 週1-2回
    year_1_target: 日次
    year_2_target: 日次複数回
    year_3_target: オンデマンド（1日10回以上）
    elite_benchmark: 日次複数回

  lead_time_for_changes:
    description: コミットから本番デプロイまでの時間
    current: 3-5日
    year_1_target: 1日以内
    year_2_target: 1時間以内
    year_3_target: 15分以内
    elite_benchmark: 1時間未満

  mean_time_to_recovery:
    description: 障害発生から復旧までの時間
    current: 4-8時間
    year_1_target: 1時間以内
    year_2_target: 30分以内
    year_3_target: 15分以内
    elite_benchmark: 1時間未満

  change_failure_rate:
    description: 本番デプロイの失敗率
    current: 15-20%
    year_1_target: 10%以下
    year_2_target: 5%以下
    year_3_target: 1%以下
    elite_benchmark: 15%未満
```

#### 1.2.2 DevOps成熟度モデル

```
┌─────────────────────────────────────────────────────────────┐
│              DevOps成熟度モデル                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Level 5: 最適化（Optimizing）                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ ・継続的な実験と改善                                 │   │
│  │ ・AI/MLによる予測的運用                             │   │
│  │ ・自己修復システム                                   │   │
│  │ ・イノベーションの文化                               │   │
│  └─────────────────────────────────────────────────────┘   │
│                          ▲ Year 3目標                       │
│  Level 4: 管理（Managed）                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ ・定量的なプロセス管理                               │   │
│  │ ・カオスエンジニアリング                             │   │
│  │ ・高度な自動化                                       │   │
│  │ ・データ駆動型意思決定                               │   │
│  └─────────────────────────────────────────────────────┘   │
│                          ▲ Year 2目標                       │
│  Level 3: 標準化（Defined）                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ ・標準化されたCI/CDパイプライン                      │   │
│  │ ・Infrastructure as Code                            │   │
│  │ ・自動テスト統合                                     │   │
│  │ ・セルフサービスデプロイ                             │   │
│  └─────────────────────────────────────────────────────┘   │
│                          ▲ Year 1目標                       │
│  Level 2: 反復（Repeatable）                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ ・基本的なCI/CD                                      │   │
│  │ ・部分的な自動化                                     │   │
│  │ ・手動デプロイプロセス                               │   │
│  └─────────────────────────────────────────────────────┘   │
│                          ▲ 現在の状態                       │
│  Level 1: 初期（Initial）                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ ・アドホックなプロセス                               │   │
│  │ ・手動操作中心                                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 現在のCI/CD状況

#### 1.3.1 既存インフラストラクチャ分析

**現在の技術スタック**（EXISTING_APP_ANALYSIS.mdより）:

```yaml
current_infrastructure:
  frontend:
    framework: Flutter 3.8.1+
    testing:
      - flutter_test
      - Mockito 5.4.4
      - Patrol 3.19.0
      - Integration Test
    test_files: 46

  backend:
    framework: Hono 4.8.3 / Node.js 18+
    language: TypeScript 5.3.3
    orm: Prisma 5.8.0
    database: PostgreSQL 16 (Docker)
    testing:
      - Vitest 1.0.0
      - Supertest 6.3.3

  development_workflow:
    ci_cd: GitHub Actions (設定済み)
    code_generation: Build Runner + Freezed + JSON Serializable
    linting: ESLint 8.56.0
    containerization: Docker (PostgreSQL)

  api_documentation:
    specification: OpenAPI 3.0.0
    swagger_ui: "@hono/swagger-ui 0.5.2"
    schema_validation: Zod 3.25.67
```

**現在の課題と改善機会**:

```yaml
current_challenges:
  ci_cd:
    - 手動デプロイプロセスの存在
    - 環境間の構成差異
    - デプロイ頻度の制限
    - ロールバック手順の未標準化

  infrastructure:
    - Infrastructure as Codeの不完全な適用
    - 環境のプロビジョニング時間
    - シークレット管理の改善余地
    - 監視・アラートの統合不足

  testing:
    - テストカバレッジ40%（目標80%）
    - E2Eテストの不足
    - パフォーマンステストの未実施
    - セキュリティテストの統合不足

improvement_opportunities:
  - CI/CDパイプラインの標準化
  - Terraformによる完全IaC化
  - ブルーグリーン/カナリアデプロイ導入
  - フィーチャーフラグシステム導入
  - 自動ロールバック機能の実装
```

---

## 第2章 CI/CDパイプラインアーキテクチャ

### 2.1 パイプライン設計

#### 2.1.1 全体アーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TripTrip CI/CD パイプラインアーキテクチャ             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌───────┐│
│  │  Code   │───▶│  Build  │───▶│  Test   │───▶│ Package │───▶│Deploy ││
│  │ Commit  │    │  Stage  │    │  Stage  │    │  Stage  │    │ Stage ││
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘    └───────┘│
│       │              │              │              │              │    │
│       ▼              ▼              ▼              ▼              ▼    │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌───────┐│
│  │・Lint   │    │・Compile│    │・Unit   │    │・Docker │    │・Dev  ││
│  │・Format │    │・Type   │    │・Integ  │    │・Helm   │    │・Stg  ││
│  │・Secret │    │ Check   │    │・E2E    │    │・Assets │    │・Prod ││
│  │ Scan   │    │・Deps   │    │・Perf   │    │         │    │       ││
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘    └───────┘│
│                                                                         │
│  Quality Gates:                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ ✓ Lint Pass    ✓ Build Success   ✓ Tests Pass    ✓ Scan Clean   │   │
│  │ ✓ Coverage 80% ✓ Security Check  ✓ Performance   ✓ Approval     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 2.1.2 ブランチ戦略とパイプライントリガー

```yaml
branch_strategy:
  type: GitHub Flow (simplified)

  branches:
    main:
      description: 本番リリース可能な状態を維持
      protection:
        - require_pull_request: true
        - require_approvals: 2
        - require_status_checks: true
        - require_linear_history: true
        - restrict_push: true
      pipeline_trigger:
        - deploy_to_staging: automatic
        - deploy_to_production: manual_approval

    feature/*:
      description: 機能開発ブランチ
      naming: feature/[JIRA-ID]-short-description
      pipeline_trigger:
        - build_and_test: automatic
        - deploy_to_dev: automatic

    hotfix/*:
      description: 緊急修正ブランチ
      naming: hotfix/[JIRA-ID]-short-description
      pipeline_trigger:
        - build_and_test: automatic
        - deploy_to_staging: automatic
        - deploy_to_production: expedited_approval

    release/*:
      description: リリース準備ブランチ（オプション）
      naming: release/v1.2.3
      usage: 大規模リリース時のみ使用

pipeline_triggers:
  push:
    - feature branches: full CI
    - main branch: CI + staging deploy

  pull_request:
    - all branches: full CI + preview deploy

  schedule:
    - nightly: full test suite + security scan
    - weekly: dependency update + performance test

  manual:
    - production deploy
    - rollback
    - infrastructure changes
```

#### 2.1.3 パイプライン定義（GitHub Actions）

**メインCI/CDワークフロー**:

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, 'feature/**', 'hotfix/**']
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deploy environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - development
          - staging
          - production

env:
  FLUTTER_VERSION: '3.24.0'
  NODE_VERSION: '20'
  REGISTRY: asia-northeast1-docker.pkg.dev
  PROJECT_ID: triptrip-production

jobs:
  # ====================================
  # Stage 1: Code Quality
  # ====================================
  lint-and-format:
    name: Lint & Format Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: backend/package-lock.json

      - name: Flutter Analyze
        run: |
          cd app
          flutter pub get
          flutter analyze --fatal-infos

      - name: Flutter Format Check
        run: |
          cd app
          dart format --set-exit-if-changed .

      - name: Backend Lint
        run: |
          cd backend
          npm ci
          npm run lint

      - name: Secret Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD

  # ====================================
  # Stage 2: Build
  # ====================================
  build-flutter:
    name: Build Flutter App
    runs-on: ubuntu-latest
    needs: lint-and-format
    strategy:
      matrix:
        platform: [android, ios, web]
    steps:
      - uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true

      - name: Setup Java (Android)
        if: matrix.platform == 'android'
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Build Android
        if: matrix.platform == 'android'
        run: |
          cd app
          flutter pub get
          flutter build apk --release --split-per-abi

      - name: Build iOS
        if: matrix.platform == 'ios'
        run: |
          cd app
          flutter pub get
          flutter build ios --release --no-codesign

      - name: Build Web
        if: matrix.platform == 'web'
        run: |
          cd app
          flutter pub get
          flutter build web --release

      - name: Upload Artifacts
        uses: actions/upload-artifact@v4
        with:
          name: flutter-build-${{ matrix.platform }}
          path: |
            app/build/app/outputs/flutter-apk/
            app/build/ios/
            app/build/web/
          retention-days: 7

  build-backend:
    name: Build Backend
    runs-on: ubuntu-latest
    needs: lint-and-format
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: backend/package-lock.json

      - name: Install Dependencies
        run: |
          cd backend
          npm ci

      - name: Type Check
        run: |
          cd backend
          npm run type-check

      - name: Build
        run: |
          cd backend
          npm run build

      - name: Generate Prisma Client
        run: |
          cd backend
          npx prisma generate

      - name: Upload Artifacts
        uses: actions/upload-artifact@v4
        with:
          name: backend-build
          path: backend/dist/
          retention-days: 7

  # ====================================
  # Stage 3: Test
  # ====================================
  test-flutter:
    name: Test Flutter
    runs-on: ubuntu-latest
    needs: build-flutter
    steps:
      - uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true

      - name: Run Unit Tests
        run: |
          cd app
          flutter pub get
          flutter test --coverage

      - name: Check Coverage
        run: |
          cd app
          # Coverage threshold check
          COVERAGE=$(lcov --summary coverage/lcov.info 2>&1 | grep "lines" | awk '{print $2}' | sed 's/%//')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage $COVERAGE% is below 80% threshold"
            exit 1
          fi

      - name: Upload Coverage
        uses: codecov/codecov-action@v4
        with:
          files: app/coverage/lcov.info
          flags: flutter

  test-backend:
    name: Test Backend
    runs-on: ubuntu-latest
    needs: build-backend
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

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: backend/package-lock.json

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

      - name: Run Tests
        run: |
          cd backend
          npm run test:coverage
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/triptrip_test
          NODE_ENV: test

      - name: Upload Coverage
        uses: codecov/codecov-action@v4
        with:
          files: backend/coverage/lcov.info
          flags: backend

  e2e-test:
    name: E2E Tests
    runs-on: ubuntu-latest
    needs: [test-flutter, test-backend]
    if: github.event_name == 'pull_request' || github.ref == 'refs/heads/main'
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
          sleep 10

      - name: Run E2E Tests
        run: |
          cd app
          flutter pub get
          flutter test integration_test/

  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: [build-flutter, build-backend]
    steps:
      - uses: actions/checkout@v4

      - name: Run Snyk
        uses: snyk/actions/node@master
        with:
          args: --all-projects --severity-threshold=high
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

      - name: Container Scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'

  # ====================================
  # Stage 4: Package
  # ====================================
  package:
    name: Package & Push
    runs-on: ubuntu-latest
    needs: [e2e-test, security-scan]
    if: github.ref == 'refs/heads/main' || github.event_name == 'workflow_dispatch'
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Artifact Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: _json_key
          password: ${{ secrets.GCP_SA_KEY }}

      - name: Extract Metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/triptrip/backend
          tags: |
            type=sha,prefix={{branch}}-
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ====================================
  # Stage 5: Deploy
  # ====================================
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: package
    environment: staging
    steps:
      - uses: actions/checkout@v4

      - name: Setup gcloud
        uses: google-github-actions/setup-gcloud@v2
        with:
          project_id: ${{ env.PROJECT_ID }}

      - name: Get GKE Credentials
        uses: google-github-actions/get-gke-credentials@v2
        with:
          cluster_name: triptrip-staging
          location: asia-northeast1

      - name: Deploy with Helm
        run: |
          helm upgrade --install triptrip-backend ./helm/backend \
            --namespace triptrip-staging \
            --set image.tag=${{ needs.package.outputs.image_tag }} \
            --set environment=staging \
            --wait --timeout 10m

      - name: Run Smoke Tests
        run: |
          kubectl run smoke-test --image=curlimages/curl --rm -i --restart=Never \
            -- curl -sf http://triptrip-backend.triptrip-staging.svc.cluster.local/health

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [package, deploy-staging]
    environment: production
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - name: Setup gcloud
        uses: google-github-actions/setup-gcloud@v2
        with:
          project_id: ${{ env.PROJECT_ID }}

      - name: Get GKE Credentials
        uses: google-github-actions/get-gke-credentials@v2
        with:
          cluster_name: triptrip-production
          location: asia-northeast1

      - name: Canary Deploy (10%)
        run: |
          helm upgrade --install triptrip-backend ./helm/backend \
            --namespace triptrip-production \
            --set image.tag=${{ needs.package.outputs.image_tag }} \
            --set canary.enabled=true \
            --set canary.weight=10 \
            --wait --timeout 10m

      - name: Monitor Canary (5 min)
        run: |
          ./scripts/monitor-canary.sh --duration 300 --error-threshold 1

      - name: Promote to Full Rollout
        run: |
          helm upgrade --install triptrip-backend ./helm/backend \
            --namespace triptrip-production \
            --set image.tag=${{ needs.package.outputs.image_tag }} \
            --set canary.enabled=false \
            --wait --timeout 10m
```

### 2.2 ビルド & テスト自動化

#### 2.2.1 ビルド最適化戦略

```yaml
build_optimization:
  caching:
    flutter:
      - pub cache
      - build artifacts
      - gradle cache (Android)
      - cocoapods cache (iOS)

    node:
      - npm cache
      - node_modules
      - prisma client

    docker:
      - layer caching (gha)
      - multi-stage builds
      - BuildKit

  parallelization:
    flutter_builds:
      - android: parallel
      - ios: parallel
      - web: parallel

    test_execution:
      - unit tests: sharded (4 shards)
      - integration tests: parallel by feature

  incremental_builds:
    enabled: true
    strategy:
      - affected packages only
      - path-based filtering
      - dependency graph analysis

build_performance_targets:
  flutter_build:
    android: 5 minutes
    ios: 8 minutes
    web: 3 minutes

  backend_build:
    compile: 1 minute
    docker_build: 2 minutes

  total_pipeline:
    pr_check: 15 minutes
    full_deploy: 30 minutes
```

#### 2.2.2 テスト自動化統合

```yaml
test_automation_integration:
  test_pyramid:
    unit_tests:
      ratio: 60%
      execution: every_commit
      timeout: 5_minutes
      coverage_requirement: 80%

    integration_tests:
      ratio: 25%
      execution: every_commit
      timeout: 10_minutes
      coverage_requirement: 70%

    e2e_tests:
      ratio: 15%
      execution: pr_and_main
      timeout: 20_minutes
      critical_paths:
        - user_registration
        - product_purchase
        - hotel_booking
        - payment_flow

  test_environments:
    unit:
      database: in_memory / mock
      external_services: mocked

    integration:
      database: docker_postgres
      external_services: sandbox

    e2e:
      database: staging_replica
      external_services: staging

  test_data_management:
    strategy: fixtures_and_factories
    tools:
      - faker.js (backend)
      - mockito (flutter)
    data_isolation: per_test_transaction
    cleanup: automatic_rollback

  parallel_test_execution:
    flutter:
      shards: 4
      distribution: by_file

    backend:
      workers: 4
      distribution: by_test_suite
```

### 2.3 アーティファクト管理

#### 2.3.1 アーティファクトリポジトリ構成

```yaml
artifact_repositories:
  container_registry:
    provider: Google Artifact Registry
    location: asia-northeast1
    repositories:
      - triptrip/backend
      - triptrip/frontend-web
      - triptrip/migrations
      - triptrip/jobs
    retention:
      production_tags: forever
      staging_tags: 30_days
      development_tags: 7_days
      untagged: 3_days

  npm_registry:
    provider: GitHub Packages
    scope: "@triptrip"
    packages:
      - shared-types
      - api-client
      - ui-components

  mobile_distribution:
    android:
      internal_testing: Google Play Internal Testing
      beta: Google Play Beta Track
      production: Google Play Production

    ios:
      internal_testing: TestFlight Internal
      beta: TestFlight External
      production: App Store

artifact_versioning:
  strategy: semantic_versioning
  format: "v{major}.{minor}.{patch}-{branch}-{sha}"
  examples:
    - "v1.2.3-main-abc1234"
    - "v1.2.3-feature-xyz-def5678"

  tagging:
    releases: "v1.2.3"
    pre_releases: "v1.2.3-rc.1"
    nightly: "nightly-2026-01-20"

artifact_signing:
  android:
    keystore: google_cloud_kms
    key_alias: triptrip-release

  ios:
    certificate: apple_distribution
    provisioning: app_store

  docker:
    cosign: enabled
    verification: required_in_production
```

#### 2.3.2 依存関係管理

```yaml
dependency_management:
  flutter:
    lock_file: pubspec.lock
    update_strategy: renovate_bot
    security_scanning: snyk
    vulnerability_policy:
      critical: block_immediately
      high: block_after_7_days
      medium: warn
      low: info_only

  node:
    lock_file: package-lock.json
    update_strategy: renovate_bot
    security_scanning: npm_audit + snyk

  docker_base_images:
    update_frequency: weekly
    security_scanning: trivy
    approved_registries:
      - gcr.io
      - docker.io (official only)

  renovation_config:
    schedule: "before 6am on Monday"
    auto_merge:
      minor_updates: true
      patch_updates: true
      major_updates: false
    grouping:
      - name: flutter-dependencies
        matchPackagePatterns: ["^flutter"]
      - name: testing-dependencies
        matchPackagePatterns: ["test", "mock", "jest"]
```

---

## 第3章 Infrastructure as Code

### 3.1 Terraform構成

#### 3.1.1 ディレクトリ構造

```
infrastructure/
├── terraform/
│   ├── modules/                    # 再利用可能なモジュール
│   │   ├── gke-cluster/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── versions.tf
│   │   ├── cloud-sql/
│   │   ├── memorystore/
│   │   ├── cloud-storage/
│   │   ├── networking/
│   │   ├── monitoring/
│   │   └── security/
│   │
│   ├── environments/              # 環境別設定
│   │   ├── development/
│   │   │   ├── main.tf
│   │   │   ├── terraform.tfvars
│   │   │   └── backend.tf
│   │   ├── staging/
│   │   │   ├── main.tf
│   │   │   ├── terraform.tfvars
│   │   │   └── backend.tf
│   │   └── production/
│   │       ├── main.tf
│   │       ├── terraform.tfvars
│   │       └── backend.tf
│   │
│   ├── shared/                    # 共通設定
│   │   ├── providers.tf
│   │   ├── versions.tf
│   │   └── locals.tf
│   │
│   └── scripts/
│       ├── init.sh
│       ├── plan.sh
│       └── apply.sh
│
├── kubernetes/
│   ├── base/                      # Kustomize base
│   │   ├── backend/
│   │   ├── frontend/
│   │   └── jobs/
│   ├── overlays/
│   │   ├── development/
│   │   ├── staging/
│   │   └── production/
│   └── helm/
│       ├── backend/
│       └── monitoring/
│
└── ansible/                       # 構成管理（必要に応じて）
    └── playbooks/
```

#### 3.1.2 Terraformモジュール例（GKEクラスター）

```hcl
# modules/gke-cluster/main.tf

terraform {
  required_version = ">= 1.5.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 5.0"
    }
  }
}

# GKE Cluster
resource "google_container_cluster" "primary" {
  provider = google-beta

  name     = var.cluster_name
  location = var.region

  # Autopilot mode for reduced operational overhead
  enable_autopilot = var.enable_autopilot

  # For Standard mode clusters
  dynamic "node_pool" {
    for_each = var.enable_autopilot ? [] : [1]
    content {
      name       = "default-pool"
      node_count = var.node_count

      node_config {
        machine_type = var.machine_type
        disk_size_gb = var.disk_size_gb
        disk_type    = "pd-ssd"

        oauth_scopes = [
          "https://www.googleapis.com/auth/cloud-platform",
        ]

        labels = var.node_labels

        shielded_instance_config {
          enable_secure_boot          = true
          enable_integrity_monitoring = true
        }

        workload_metadata_config {
          mode = "GKE_METADATA"
        }
      }

      autoscaling {
        min_node_count = var.min_node_count
        max_node_count = var.max_node_count
      }

      management {
        auto_repair  = true
        auto_upgrade = true
      }
    }
  }

  # Network configuration
  network    = var.network
  subnetwork = var.subnetwork

  ip_allocation_policy {
    cluster_secondary_range_name  = var.pods_range_name
    services_secondary_range_name = var.services_range_name
  }

  # Private cluster configuration
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = var.master_ipv4_cidr_block
  }

  # Master authorized networks
  master_authorized_networks_config {
    dynamic "cidr_blocks" {
      for_each = var.master_authorized_networks
      content {
        cidr_block   = cidr_blocks.value.cidr_block
        display_name = cidr_blocks.value.display_name
      }
    }
  }

  # Workload Identity
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  # Binary Authorization
  binary_authorization {
    evaluation_mode = var.enable_binary_authorization ? "PROJECT_SINGLETON_POLICY_ENFORCE" : "DISABLED"
  }

  # Maintenance window
  maintenance_policy {
    recurring_window {
      start_time = "2026-01-01T09:00:00Z"
      end_time   = "2026-01-01T17:00:00Z"
      recurrence = "FREQ=WEEKLY;BYDAY=SA,SU"
    }
  }

  # Logging and monitoring
  logging_config {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"]
  }

  monitoring_config {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"]
    managed_prometheus {
      enabled = true
    }
  }

  # Security
  release_channel {
    channel = var.release_channel
  }

  resource_labels = var.labels

  lifecycle {
    ignore_changes = [
      node_pool,
    ]
  }
}

# Node Pool (for Standard mode)
resource "google_container_node_pool" "primary_nodes" {
  count = var.enable_autopilot ? 0 : 1

  name       = "${var.cluster_name}-node-pool"
  location   = var.region
  cluster    = google_container_cluster.primary.name
  node_count = var.node_count

  node_config {
    preemptible  = var.use_preemptible
    machine_type = var.machine_type
    disk_size_gb = var.disk_size_gb
    disk_type    = "pd-ssd"

    service_account = var.service_account_email

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform",
    ]

    labels = merge(var.node_labels, {
      "node-pool" = "primary"
    })

    tags = var.network_tags

    metadata = {
      disable-legacy-endpoints = "true"
    }
  }

  autoscaling {
    min_node_count = var.min_node_count
    max_node_count = var.max_node_count
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }

  upgrade_settings {
    max_surge       = 1
    max_unavailable = 0
  }
}
```

### 3.2 環境管理（dev/staging/prod）

#### 3.2.1 環境別設定

```yaml
environment_configuration:
  development:
    purpose: 開発者のローカル統合テスト
    infrastructure:
      gke:
        node_count: 1-3
        machine_type: e2-medium
        autoscaling: true
      cloud_sql:
        tier: db-f1-micro
        high_availability: false
      memorystore:
        tier: basic
        memory_gb: 1
    features:
      debug_logging: enabled
      feature_flags: all_enabled
      mock_services: available
    access:
      vpn_required: false
      ip_restrictions: none
    data:
      type: synthetic
      pii: none
    cost_optimization:
      preemptible_nodes: true
      auto_shutdown: true (nights/weekends)

  staging:
    purpose: 本番前の統合・受入テスト
    infrastructure:
      gke:
        node_count: 2-5
        machine_type: e2-standard-2
        autoscaling: true
      cloud_sql:
        tier: db-custom-2-4096
        high_availability: true
      memorystore:
        tier: standard
        memory_gb: 2
    features:
      debug_logging: enabled
      feature_flags: production_like
      mock_services: disabled
    access:
      vpn_required: false
      ip_restrictions: office_ips
    data:
      type: anonymized_production_copy
      pii: masked
    deployment:
      automatic_from_main: true
      approval_required: false

  production:
    purpose: 本番サービス提供
    infrastructure:
      gke:
        node_count: 3-10
        machine_type: e2-standard-4
        autoscaling: true
        regional: true
      cloud_sql:
        tier: db-custom-4-16384
        high_availability: true
        read_replicas: 1
      memorystore:
        tier: standard
        memory_gb: 4
        high_availability: true
    features:
      debug_logging: disabled
      feature_flags: gradual_rollout
      mock_services: disabled
    access:
      vpn_required: true
      ip_restrictions: strict
      mfa_required: true
    data:
      type: live
      pii: encrypted
    deployment:
      automatic: false
      approval_required: true
      approvers: 2
      deployment_window: business_hours
```

#### 3.2.2 環境プロモーション戦略

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    環境プロモーションフロー                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Developer      Feature        Main           Staging      Production  │
│   Workstation    Branch         Branch         Env          Env         │
│                                                                         │
│   ┌─────┐       ┌─────┐       ┌─────┐       ┌─────┐       ┌─────┐    │
│   │Local│──────▶│ Dev │──────▶│ PR  │──────▶│ Stg │──────▶│Prod │    │
│   │Test │       │ Env │       │Merge│       │ Env │       │ Env │    │
│   └─────┘       └─────┘       └─────┘       └─────┘       └─────┘    │
│      │             │             │             │             │         │
│      ▼             ▼             ▼             ▼             ▼         │
│   ┌─────┐       ┌─────┐       ┌─────┐       ┌─────┐       ┌─────┐    │
│   │Unit │       │Integ│       │ All │       │Smoke│       │Canary│    │
│   │Test │       │Test │       │Test │       │Test │       │→Full│    │
│   └─────┘       └─────┘       └─────┘       └─────┘       └─────┘    │
│                                                                         │
│   Gates:                                                                │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │ Dev→Staging: CI Pass + Code Review (2 approvers)                │  │
│   │ Staging→Prod: Staging Test Pass + Manual Approval + Deploy Window│  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3 状態管理 & リモートバックエンド

#### 3.3.1 Terraform状態管理

```hcl
# environments/production/backend.tf

terraform {
  backend "gcs" {
    bucket = "triptrip-terraform-state"
    prefix = "production"
  }
}

# State bucket configuration (in separate bootstrap)
resource "google_storage_bucket" "terraform_state" {
  name     = "triptrip-terraform-state"
  location = "asia-northeast1"

  versioning {
    enabled = true
  }

  lifecycle_rule {
    action {
      type = "Delete"
    }
    condition {
      num_newer_versions = 30
    }
  }

  uniform_bucket_level_access = true

  labels = {
    purpose     = "terraform-state"
    environment = "shared"
  }
}

# State locking with Cloud Storage
# (Automatic with GCS backend)
```

#### 3.3.2 状態管理ベストプラクティス

```yaml
state_management_practices:
  isolation:
    strategy: per_environment_state
    benefits:
      - 環境間の影響分離
      - 並列実行の安全性
      - ロールバックの容易さ

  locking:
    mechanism: gcs_built_in
    timeout: 5_minutes
    retry: 3_times

  versioning:
    enabled: true
    retention: 30_versions

  encryption:
    at_rest: google_managed_key
    in_transit: tls_1_3

  backup:
    frequency: after_each_apply
    retention: 90_days
    location: separate_bucket

  access_control:
    principle: least_privilege
    production_state:
      read: platform_team, sre_team
      write: platform_team_leads
    staging_state:
      read: all_engineers
      write: platform_team
    development_state:
      read: all_engineers
      write: all_engineers

drift_detection:
  schedule: daily_at_0600
  action: alert_and_report
  auto_remediation: false
  notification:
    channel: slack_platform
    severity: warning
```

---

## 第4章 デプロイメント戦略

### 4.1 ブルーグリーンデプロイメント

#### 4.1.1 ブルーグリーンアーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ブルーグリーンデプロイメント                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                        ┌─────────────┐                                  │
│                        │   Router    │                                  │
│                        │ (Ingress)   │                                  │
│                        └──────┬──────┘                                  │
│                               │                                         │
│               ┌───────────────┼───────────────┐                        │
│               │               │               │                        │
│               ▼               │               ▼                        │
│        ┌─────────────┐        │        ┌─────────────┐                 │
│        │    Blue     │        │        │   Green     │                 │
│        │  (Active)   │◀───────┘        │  (Standby)  │                 │
│        │   v1.2.3    │                 │   v1.2.4    │                 │
│        └─────────────┘                 └─────────────┘                 │
│               │                               │                        │
│               ▼                               ▼                        │
│        ┌─────────────┐                 ┌─────────────┐                 │
│        │  Database   │                 │  Database   │                 │
│        │  (Shared)   │◀───────────────▶│  (Shared)   │                 │
│        └─────────────┘                 └─────────────┘                 │
│                                                                         │
│  切り替えプロセス:                                                      │
│  1. Green環境に新バージョンをデプロイ                                   │
│  2. Green環境でスモークテスト実行                                       │
│  3. ルーターをBlue→Greenに切り替え                                     │
│  4. Greenがアクティブに、Blueがスタンバイに                            │
│  5. 問題発生時、即座にBlueにロールバック                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 4.1.2 ブルーグリーン実装（Kubernetes）

```yaml
# kubernetes/overlays/production/blue-green/deployment-blue.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: triptrip-backend-blue
  namespace: triptrip-production
  labels:
    app: triptrip-backend
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: triptrip-backend
      version: blue
  template:
    metadata:
      labels:
        app: triptrip-backend
        version: blue
    spec:
      containers:
        - name: backend
          image: asia-northeast1-docker.pkg.dev/triptrip-production/triptrip/backend:v1.2.3
          ports:
            - containerPort: 3001
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3001
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 3001
            initialDelaySeconds: 5
            periodSeconds: 5
          env:
            - name: NODE_ENV
              value: "production"
            - name: VERSION
              value: "blue"
---
# Service for traffic routing
apiVersion: v1
kind: Service
metadata:
  name: triptrip-backend
  namespace: triptrip-production
spec:
  selector:
    app: triptrip-backend
    version: blue  # Switch to 'green' during deployment
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3001
  type: ClusterIP
```

### 4.2 カナリアリリース

#### 4.2.1 カナリアデプロイメントアーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    カナリアデプロイメント                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                        ┌─────────────┐                                  │
│                        │   Ingress   │                                  │
│                        │  (Istio)    │                                  │
│                        └──────┬──────┘                                  │
│                               │                                         │
│               ┌───────────────┴───────────────┐                        │
│               │    Traffic Split              │                        │
│               │    Stable: 90%                │                        │
│               │    Canary: 10%                │                        │
│               └───────────────┬───────────────┘                        │
│                               │                                         │
│               ┌───────────────┼───────────────┐                        │
│               │               │               │                        │
│               ▼               │               ▼                        │
│        ┌─────────────┐        │        ┌─────────────┐                 │
│        │   Stable    │        │        │   Canary    │                 │
│        │   v1.2.3    │        │        │   v1.2.4    │                 │
│        │  (90%)      │        │        │  (10%)      │                 │
│        │  3 replicas │        │        │  1 replica  │                 │
│        └─────────────┘        │        └─────────────┘                 │
│                               │                                         │
│  プログレッシブロールアウト:                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 10% ──▶ 25% ──▶ 50% ──▶ 75% ──▶ 100%                           │   │
│  │  ↓       ↓       ↓       ↓       ↓                              │   │
│  │ 5min   5min    5min    5min   Complete                          │   │
│  │ monitor monitor monitor monitor                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  自動ロールバック条件:                                                  │
│  • エラー率 > 1%                                                       │
│  • レイテンシP95 > 500ms                                               │
│  • ヘルスチェック失敗                                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 4.2.2 Argo Rollouts設定

```yaml
# kubernetes/base/backend/rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: triptrip-backend
  namespace: triptrip-production
spec:
  replicas: 4
  selector:
    matchLabels:
      app: triptrip-backend
  template:
    metadata:
      labels:
        app: triptrip-backend
    spec:
      containers:
        - name: backend
          image: asia-northeast1-docker.pkg.dev/triptrip-production/triptrip/backend:v1.2.4
          ports:
            - containerPort: 3001
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
  strategy:
    canary:
      # カナリアステップ定義
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - setWeight: 25
        - pause: { duration: 5m }
        - setWeight: 50
        - pause: { duration: 5m }
        - setWeight: 75
        - pause: { duration: 5m }
        - setWeight: 100

      # カナリア分析
      analysis:
        templates:
          - templateName: success-rate
          - templateName: latency
        startingStep: 1
        args:
          - name: service-name
            value: triptrip-backend

      # トラフィックルーティング
      trafficRouting:
        istio:
          virtualService:
            name: triptrip-backend
            routes:
              - primary
          destinationRule:
            name: triptrip-backend
            canarySubsetName: canary
            stableSubsetName: stable

      # 自動ロールバック
      autoRollback:
        enabled: true
---
# Analysis Template
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: triptrip-production
spec:
  metrics:
    - name: success-rate
      interval: 1m
      count: 5
      successCondition: result[0] >= 0.99
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",status=~"2.."}[5m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: latency
  namespace: triptrip-production
spec:
  metrics:
    - name: latency-p95
      interval: 1m
      count: 5
      successCondition: result[0] < 0.5
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            histogram_quantile(0.95,
              sum(rate(http_request_duration_seconds_bucket{service="{{args.service-name}}"}[5m]))
              by (le)
            )
```

### 4.3 フィーチャーフラグ管理

#### 4.3.1 フィーチャーフラグアーキテクチャ

```yaml
feature_flag_architecture:
  provider: LaunchDarkly

  sdk_integration:
    backend:
      sdk: "@launchdarkly/node-server-sdk"
      initialization: application_startup
      fallback: local_defaults

    flutter:
      sdk: "launchdarkly_flutter_client_sdk"
      initialization: app_launch
      fallback: local_defaults

  flag_types:
    release_flags:
      description: 新機能のリリース制御
      lifecycle: temporary
      examples:
        - new_checkout_flow
        - enhanced_search
        - ai_recommendations
      ownership: product_team

    operational_flags:
      description: 運用上のトグル
      lifecycle: long_lived
      examples:
        - maintenance_mode
        - debug_logging
        - rate_limiting
      ownership: sre_team

    experiment_flags:
      description: A/Bテスト用
      lifecycle: temporary
      examples:
        - checkout_button_color
        - pricing_display_format
      ownership: growth_team

    permission_flags:
      description: 機能アクセス制御
      lifecycle: long_lived
      examples:
        - beta_features
        - enterprise_features
      ownership: product_team

  targeting_rules:
    user_attributes:
      - user_id
      - email_domain
      - country
      - subscription_tier
      - device_type

    percentage_rollout:
      - 1% (internal testing)
      - 5% (beta users)
      - 25% (early adopters)
      - 50% (half traffic)
      - 100% (general availability)

    segment_based:
      - internal_users
      - beta_testers
      - enterprise_customers
      - specific_regions
```

#### 4.3.2 フィーチャーフラグ実装例

```typescript
// backend/src/services/feature-flag.service.ts
import * as LaunchDarkly from '@launchdarkly/node-server-sdk';

export class FeatureFlagService {
  private client: LaunchDarkly.LDClient;

  constructor() {
    this.client = LaunchDarkly.init(process.env.LAUNCHDARKLY_SDK_KEY!);
  }

  async initialize(): Promise<void> {
    await this.client.waitForInitialization();
  }

  async isEnabled(
    flagKey: string,
    user: LaunchDarkly.LDContext,
    defaultValue: boolean = false
  ): Promise<boolean> {
    return this.client.variation(flagKey, user, defaultValue);
  }

  async getVariation<T>(
    flagKey: string,
    user: LaunchDarkly.LDContext,
    defaultValue: T
  ): Promise<T> {
    return this.client.variation(flagKey, user, defaultValue);
  }

  createUserContext(user: {
    id: string;
    email?: string;
    country?: string;
    tier?: string;
  }): LaunchDarkly.LDContext {
    return {
      kind: 'user',
      key: user.id,
      email: user.email,
      country: user.country,
      custom: {
        subscriptionTier: user.tier,
      },
    };
  }
}

// Usage example
const featureFlags = new FeatureFlagService();

app.get('/api/checkout', async (req, res) => {
  const user = featureFlags.createUserContext({
    id: req.user.id,
    email: req.user.email,
    country: req.user.country,
    tier: req.user.subscriptionTier,
  });

  const useNewCheckout = await featureFlags.isEnabled(
    'new_checkout_flow',
    user,
    false
  );

  if (useNewCheckout) {
    return handleNewCheckoutFlow(req, res);
  }
  return handleLegacyCheckoutFlow(req, res);
});
```

```dart
// Flutter integration
// lib/services/feature_flag_service.dart
import 'package:launchdarkly_flutter_client_sdk/launchdarkly_flutter_client_sdk.dart';

class FeatureFlagService {
  late LDClient _client;

  Future<void> initialize(String mobileKey, String userId) async {
    final config = LDConfig(mobileKey, AutoEnvAttributes.enabled);
    final context = LDContextBuilder()
        .kind('user', userId)
        .build();

    _client = LDClient(config, context);
    await _client.start();
  }

  bool isEnabled(String flagKey, {bool defaultValue = false}) {
    return _client.boolVariation(flagKey, defaultValue);
  }

  String getStringVariation(String flagKey, {String defaultValue = ''}) {
    return _client.stringVariation(flagKey, defaultValue);
  }

  Future<void> identifyUser(String userId, {
    String? email,
    String? country,
    Map<String, dynamic>? custom,
  }) async {
    final contextBuilder = LDContextBuilder()
        .kind('user', userId);

    if (email != null) contextBuilder.set('email', LDValue.ofString(email));
    if (country != null) contextBuilder.set('country', LDValue.ofString(country));
    if (custom != null) {
      custom.forEach((key, value) {
        contextBuilder.set(key, LDValue.buildObject((builder) => builder.addValue(key, value)));
      });
    }

    await _client.identify(contextBuilder.build());
  }

  void dispose() {
    _client.close();
  }
}
```

---

## 第5章 運用 & インシデント対応

### 5.1 ロールバック手順

#### 5.1.1 ロールバック判断基準

```yaml
rollback_criteria:
  automatic_triggers:
    error_rate:
      threshold: 5%
      window: 5_minutes
      action: immediate_rollback

    latency_p95:
      threshold: 500ms
      window: 5_minutes
      action: immediate_rollback

    health_check_failures:
      threshold: 3_consecutive
      action: immediate_rollback

    crash_rate:
      threshold: 1%
      window: 10_minutes
      action: immediate_rollback

  manual_triggers:
    critical_bug_discovered: true
    security_vulnerability: true
    data_corruption_risk: true
    stakeholder_request: true

  rollback_decision_tree:
    step_1:
      question: "サービスは完全にダウンしているか？"
      yes: "即座にロールバック"
      no: "Step 2へ"

    step_2:
      question: "エラー率が5%を超えているか？"
      yes: "即座にロールバック"
      no: "Step 3へ"

    step_3:
      question: "ユーザーへの影響は深刻か？"
      yes: "即座にロールバック"
      no: "Step 4へ"

    step_4:
      question: "問題は短時間で修正可能か？"
      yes: "ホットフィックスを検討"
      no: "ロールバックを実行"
```

#### 5.1.2 ロールバック実行手順

```yaml
rollback_procedures:
  kubernetes_deployment:
    immediate_rollback:
      command: |
        # 直前のリビジョンにロールバック
        kubectl rollout undo deployment/triptrip-backend -n triptrip-production

        # 特定のリビジョンにロールバック
        kubectl rollout undo deployment/triptrip-backend -n triptrip-production --to-revision=5

        # ロールバック状態確認
        kubectl rollout status deployment/triptrip-backend -n triptrip-production

    argo_rollouts:
      command: |
        # ロールアウトを中止してロールバック
        kubectl argo rollouts abort triptrip-backend -n triptrip-production

        # 特定のリビジョンにロールバック
        kubectl argo rollouts undo triptrip-backend -n triptrip-production --to-revision=5

  helm_release:
    command: |
      # 直前のリビジョンにロールバック
      helm rollback triptrip-backend 0 -n triptrip-production

      # 特定のリビジョンにロールバック
      helm rollback triptrip-backend 5 -n triptrip-production

      # リビジョン履歴確認
      helm history triptrip-backend -n triptrip-production

  database_migration:
    procedure:
      - step: 1
        action: "新スキーマへの書き込みを停止"
        command: "feature_flag.disable('new_schema_writes')"

      - step: 2
        action: "旧スキーマへの互換性確認"
        command: "npm run migration:validate-rollback"

      - step: 3
        action: "マイグレーションのロールバック"
        command: "npx prisma migrate rollback --steps=1"

      - step: 4
        action: "データ整合性チェック"
        command: "npm run db:verify-integrity"

  mobile_app:
    procedure:
      - step: 1
        action: "強制アップデート解除"
        command: "firebase remote_config update min_version"

      - step: 2
        action: "旧バージョンのAPIサポート有効化"
        command: "feature_flag.enable('legacy_api_v1')"

      - step: 3
        action: "必要に応じてストアから削除リクエスト"
        note: "承認まで24-48時間"
```

### 5.2 シークレット管理

#### 5.2.1 シークレット管理戦略

```yaml
secret_management:
  provider: Google Secret Manager

  secret_categories:
    application_secrets:
      - database_credentials
      - api_keys
      - jwt_secrets
      - encryption_keys

    infrastructure_secrets:
      - service_account_keys
      - ssh_keys
      - tls_certificates

    integration_secrets:
      - stripe_api_keys
      - sendgrid_api_keys
      - launchdarkly_sdk_keys
      - sentry_dsn

  access_control:
    development:
      - developers: read (non-prod secrets only)
      - ci_service_account: read

    staging:
      - platform_team: read
      - ci_service_account: read

    production:
      - sre_team: read
      - deployment_service_account: read
      - on_call_engineer: read (emergency)

  rotation_policy:
    database_passwords:
      frequency: 90_days
      automation: enabled

    api_keys:
      frequency: 180_days
      automation: partially_automated

    jwt_secrets:
      frequency: 30_days
      automation: enabled

    tls_certificates:
      frequency: 365_days
      automation: enabled (cert-manager)

  injection_methods:
    kubernetes:
      - external_secrets_operator
      - workload_identity

    ci_cd:
      - github_actions_secrets
      - gcp_secret_manager_action
```

#### 5.2.2 External Secrets Operator設定

```yaml
# kubernetes/base/secrets/external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: triptrip-backend-secrets
  namespace: triptrip-production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcp-secret-store
    kind: ClusterSecretStore
  target:
    name: triptrip-backend-secrets
    creationPolicy: Owner
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: triptrip-production-database-url

    - secretKey: JWT_SECRET
      remoteRef:
        key: triptrip-production-jwt-secret

    - secretKey: STRIPE_SECRET_KEY
      remoteRef:
        key: triptrip-production-stripe-secret

    - secretKey: LAUNCHDARKLY_SDK_KEY
      remoteRef:
        key: triptrip-production-launchdarkly-sdk
---
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: gcp-secret-store
spec:
  provider:
    gcpsm:
      projectID: triptrip-production
      auth:
        workloadIdentity:
          clusterLocation: asia-northeast1
          clusterName: triptrip-production
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
```

### 5.3 監視統合

#### 5.3.1 CI/CDパイプライン監視

```yaml
cicd_monitoring:
  pipeline_metrics:
    build_duration:
      metric: cicd_build_duration_seconds
      labels: [pipeline, stage, status]
      alert_threshold: 30_minutes

    deployment_frequency:
      metric: cicd_deployments_total
      labels: [environment, status]
      target: daily_or_more

    failure_rate:
      metric: cicd_failures_total / cicd_runs_total
      labels: [pipeline, stage]
      alert_threshold: 20%

    queue_time:
      metric: cicd_queue_duration_seconds
      labels: [runner_type]
      alert_threshold: 10_minutes

  deployment_metrics:
    rollout_duration:
      metric: deployment_rollout_duration_seconds
      labels: [environment, service]

    rollback_count:
      metric: deployment_rollbacks_total
      labels: [environment, reason]

    canary_success_rate:
      metric: canary_analysis_success_rate
      labels: [service]

  dashboards:
    - name: CI/CD Overview
      panels:
        - build_status_by_pipeline
        - deployment_frequency_trend
        - failure_rate_by_stage
        - average_lead_time

    - name: Deployment Health
      panels:
        - active_deployments
        - canary_progress
        - rollback_history
        - environment_status

  alerts:
    - name: pipeline_stuck
      condition: build_duration > 45m
      severity: warning

    - name: high_failure_rate
      condition: failure_rate > 30%
      severity: critical

    - name: deployment_failed
      condition: deployment_status == failed
      severity: critical

    - name: canary_degraded
      condition: canary_error_rate > 5%
      severity: warning
```

---

## 第6章 実装ロードマップ & 文書間参照

### 6.1 DevOps導入タイムライン

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DevOps導入タイムライン                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Year 1                                                                 │
│  ═══════════════════════════════════════════════════════════════════    │
│  Q1: 基盤構築                                                           │
│  ├─ GitHub Actions CI/CD標準化                                         │
│  ├─ Terraform導入（GCP基本リソース）                                   │
│  ├─ 環境分離（dev/staging/prod）                                       │
│  └─ シークレット管理（Secret Manager）                                 │
│                                                                         │
│  Q2: 自動化強化                                                         │
│  ├─ 自動テスト統合（カバレッジ70%）                                    │
│  ├─ ブルーグリーンデプロイ導入                                         │
│  ├─ 自動ロールバック機能                                               │
│  └─ 監視・アラート基盤統合                                             │
│                                                                         │
│  Q3-Q4: 最適化                                                          │
│  ├─ カナリアリリース導入（Argo Rollouts）                              │
│  ├─ フィーチャーフラグ（LaunchDarkly）                                 │
│  ├─ GitOps（ArgoCD）                                                   │
│  └─ 日次デプロイの実現                                                 │
│                                                                         │
│  Year 2                                                                 │
│  ═══════════════════════════════════════════════════════════════════    │
│  H1: マイクロサービス対応                                               │
│  ├─ サービス別パイプライン                                             │
│  ├─ マルチリポジトリCI/CD                                              │
│  ├─ サービスメッシュ（Istio）                                          │
│  └─ 分散トレーシング                                                   │
│                                                                         │
│  H2: スケーリング                                                       │
│  ├─ マルチリージョンデプロイ                                           │
│  ├─ カオスエンジニアリング                                             │
│  ├─ 継続的デプロイ（1日10回以上）                                      │
│  └─ 完全自動化                                                         │
│                                                                         │
│  Year 3                                                                 │
│  ═══════════════════════════════════════════════════════════════════    │
│  ├─ AI/ML駆動の異常検知                                                │
│  ├─ 自己修復システム                                                   │
│  ├─ 予測的スケーリング                                                 │
│  └─ DevOps成熟度レベル5達成                                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 ツールスタック

```yaml
devops_toolstack:
  source_control:
    primary: GitHub Enterprise
    branching: GitHub Flow

  ci_cd:
    primary: GitHub Actions
    secondary: ArgoCD (GitOps)
    rollouts: Argo Rollouts

  infrastructure:
    iac: Terraform
    kubernetes: GKE
    service_mesh: Istio

  artifact_management:
    containers: Google Artifact Registry
    helm_charts: Google Artifact Registry
    npm: GitHub Packages

  secrets:
    management: Google Secret Manager
    kubernetes: External Secrets Operator

  monitoring:
    metrics: Cloud Monitoring + Prometheus
    logs: Cloud Logging
    traces: Cloud Trace
    dashboards: Grafana

  feature_flags:
    provider: LaunchDarkly

  security:
    sast: SonarQube
    dast: OWASP ZAP
    container_scanning: Trivy
    dependency_scanning: Snyk
```

### 6.3 関連文書参照

```yaml
document_references:
  directly_related:
    - Doc-DM-001: アジャイルデリバリーフレームワーク
    - Doc-QA-001: テスト戦略＆品質基準
    - Doc-QA-002: モニタリング、アラート＆インシデント管理
    - Doc-QA-003: リリース管理＆ロールバック手順

  technical_architecture:
    - Doc-SA-001: システムアーキテクチャ
    - Doc-IA-001: インフラストラクチャアーキテクチャ
    - Doc-SC-001: セキュリティアーキテクチャ

  operational:
    - Doc-IR-001: 開発ロードマップ Year 1-3
    - Doc-IR-002: リソース＆キャパシティプランニング

  app_context:
    - EXISTING_APP_ANALYSIS.md: 既存アプリ分析
```

### 6.4 成功基準サマリー

```yaml
success_criteria:
  year_1:
    dora_metrics:
      deployment_frequency: daily
      lead_time: 1_day
      mttr: 1_hour
      change_failure_rate: 10%

    automation:
      ci_pipeline_pass_rate: 95%
      automated_test_coverage: 70%
      infrastructure_as_code: 80%

    reliability:
      availability: 99.9%
      successful_deployments: 95%

  year_2:
    dora_metrics:
      deployment_frequency: multiple_daily
      lead_time: 1_hour
      mttr: 30_minutes
      change_failure_rate: 5%

    automation:
      ci_pipeline_pass_rate: 98%
      automated_test_coverage: 80%
      infrastructure_as_code: 95%
      gitops_adoption: 100%

    reliability:
      availability: 99.95%
      successful_deployments: 98%

  year_3:
    dora_metrics:
      deployment_frequency: on_demand
      lead_time: 15_minutes
      mttr: 15_minutes
      change_failure_rate: 1%

    automation:
      ci_pipeline_pass_rate: 99%
      automated_test_coverage: 90%
      self_healing: enabled

    reliability:
      availability: 99.99%
      successful_deployments: 99%
```

---

## 付録

### 付録A: パイプラインテンプレート集

```yaml
pipeline_templates:
  flutter_app:
    stages: [lint, build, test, package, deploy]
    artifacts: [apk, ipa, web]

  backend_service:
    stages: [lint, build, test, security, package, deploy]
    artifacts: [docker_image]

  infrastructure:
    stages: [validate, plan, apply]
    artifacts: [terraform_plan]

  database_migration:
    stages: [validate, backup, migrate, verify]
    artifacts: [migration_log]
```

### 付録B: ロールバックチェックリスト

```yaml
rollback_checklist:
  pre_rollback:
    - [ ] 問題の特定と影響範囲の確認
    - [ ] ロールバック先バージョンの確認
    - [ ] 関係者への通知
    - [ ] バックアップの確認

  during_rollback:
    - [ ] トラフィックの段階的切り替え
    - [ ] ヘルスチェックの監視
    - [ ] エラーログの監視

  post_rollback:
    - [ ] サービス正常性の確認
    - [ ] ユーザー影響の確認
    - [ ] 関係者への完了報告
    - [ ] ポストモーテムのスケジュール
```

### 付録C: 緊急対応連絡先

```yaml
emergency_contacts:
  on_call:
    primary: PagerDuty rotation
    escalation: 15分後に自動エスカレーション

  stakeholders:
    engineering_lead: "@engineering-leads"
    sre_team: "@sre-team"
    product_team: "@product-team"

  external:
    gcp_support: Google Cloud Support
    github_support: GitHub Enterprise Support
```

---

**文書情報**

| 項目 | 内容 |
|------|------|
| Document ID | Doc-DM-002 |
| Version | 1.0.0 |
| Created | 2026-01-20 |
| Last Updated | 2026-01-20 |
| Status | Draft |
| Owner | Implementation Agent |
| Reviewers | CTO, Platform Lead, SRE Team |
| Total Lines | 1,800+ |

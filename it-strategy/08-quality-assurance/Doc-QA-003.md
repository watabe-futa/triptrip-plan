# Doc-QA-003: リリース管理＆ロールバック手順

## 文書メタデータ
| 項目 | 内容 |
|------|------|
| 文書ID | Doc-QA-003 |
| タイトル | リリース管理＆ロールバック手順 |
| バージョン | 1.0.0 |
| 作成日 | 2026-01-20 |
| 最終更新日 | 2026-01-20 |
| 作成者 | IT戦略チーム |
| 承認者 | CTO |
| 分類 | IT戦略 - 品質保証 |
| 関連文書 | Doc-DM-001, Doc-DM-002, Doc-QA-001, Doc-QA-002 |

---

## 変更履歴

| バージョン | 日付 | 変更者 | 変更内容 |
|------------|------|--------|----------|
| 1.0.0 | 2026-01-20 | IT戦略チーム | 初版作成 |

---

## 目次

1. [エグゼクティブサマリー](#1-エグゼクティブサマリー)
2. [リリース管理戦略](#2-リリース管理戦略)
3. [リリースプロセス](#3-リリースプロセス)
4. [デプロイメント戦略](#4-デプロイメント戦略)
5. [フィーチャーフラグ管理](#5-フィーチャーフラグ管理)
6. [ロールバック手順](#6-ロールバック手順)
7. [緊急リリース対応](#7-緊急リリース対応)
8. [モバイルアプリリリース](#8-モバイルアプリリリース)
9. [データベースマイグレーション](#9-データベースマイグレーション)
10. [リリースコミュニケーション](#10-リリースコミュニケーション)
11. [リリースメトリクスと改善](#11-リリースメトリクスと改善)

---

## 1. エグゼクティブサマリー

### 1.1 文書の目的

本文書は、TripTripプラットフォームの**リリース管理プロセス、デプロイメント戦略、およびロールバック手順**を定義する。Google、Netflix、Spotifyのエンジニアリングプラクティスを参考に、高頻度かつ安全なリリースを実現する。

### 1.2 現状と目標

| 項目 | 現状 | Year 1目標 | Year 3目標 |
|------|------|------------|------------|
| デプロイ頻度 | 週1回 | 日次 | オンデマンド |
| リードタイム | 1週間 | 1日 | 1時間 |
| 変更失敗率 | 未計測 | <10% | <5% |
| MTTR | 未計測 | 30分 | 15分 |
| ロールバック時間 | 手動 | 5分 | 2分（自動） |

### 1.3 リリース管理の原則

```
┌─────────────────────────────────────────────────────────────────────┐
│                    リリース管理の5原則                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 小さく頻繁に                                                     │
│     └─ 大きなリリースより小さな変更を頻繁に                          │
│                                                                      │
│  2. 自動化優先                                                       │
│     └─ 手動プロセスを排除し、ヒューマンエラーを防止                   │
│                                                                      │
│  3. 安全にロールバック                                               │
│     └─ いつでも前のバージョンに戻れる状態を維持                      │
│                                                                      │
│  4. 段階的展開                                                       │
│     └─ Canary → 段階的ロールアウト → 全展開                         │
│                                                                      │
│  5. 観測可能性                                                       │
│     └─ リリース前後のメトリクス変化を即座に検知                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.4 リリースタイプ分類

| タイプ | 説明 | 頻度 | 承認 | 例 |
|--------|------|------|------|-----|
| Regular | 通常機能リリース | 日次〜週次 | 自動 | 新機能、改善 |
| Hotfix | 緊急バグ修正 | 随時 | 手動 | 重大バグ |
| Security | セキュリティパッチ | 随時 | 手動+セキュリティレビュー | 脆弱性対応 |
| Config | 設定変更 | 随時 | 自動/手動 | フィーチャーフラグ |
| Infrastructure | インフラ変更 | 週次 | 手動 | スケーリング、更新 |

---

## 2. リリース管理戦略

### 2.1 リリーストレイン方式

```
┌─────────────────────────────────────────────────────────────────────┐
│                    リリーストレイン（週次）                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Week N-1                Week N                  Week N+1            │
│  ─────────              ─────────               ─────────            │
│                                                                      │
│  ┌─────────┐           ┌─────────┐             ┌─────────┐          │
│  │ 開発    │           │ 開発    │             │ 開発    │          │
│  │ Sprint  │           │ Sprint  │             │ Sprint  │          │
│  └────┬────┘           └────┬────┘             └────┬────┘          │
│       │                     │                       │                │
│       ▼                     ▼                       ▼                │
│  ┌─────────┐           ┌─────────┐             ┌─────────┐          │
│  │ 水曜日   │           │ 水曜日   │             │ 水曜日   │          │
│  │ Code    │           │ Code    │             │ Code    │          │
│  │ Freeze  │           │ Freeze  │             │ Freeze  │          │
│  └────┬────┘           └────┬────┘             └────┬────┘          │
│       │                     │                       │                │
│       ▼                     ▼                       ▼                │
│  ┌─────────┐           ┌─────────┐             ┌─────────┐          │
│  │ 木曜日   │           │ 木曜日   │             │ 木曜日   │          │
│  │ Staging │           │ Staging │             │ Staging │          │
│  │ Deploy  │           │ Deploy  │             │ Deploy  │          │
│  └────┬────┘           └────┬────┘             └────┬────┘          │
│       │                     │                       │                │
│       ▼                     ▼                       ▼                │
│  ┌─────────┐           ┌─────────┐             ┌─────────┐          │
│  │ 火曜日   │           │ 火曜日   │             │ 火曜日   │          │
│  │ Prod    │           │ Prod    │             │ Prod    │          │
│  │ Release │           │ Release │             │ Release │          │
│  └─────────┘           └─────────┘             └─────────┘          │
│                                                                      │
│  ※ 火曜日リリース = 週初めを避け、週末前にモニタリング時間確保       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 ブランチ戦略とリリース

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GitHub Flow + Release Branches                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  main ──●────●────●────●────●────●────●────●────●────●──►           │
│         │    │    │    │    │    │    │    │    │    │              │
│         │    │    │    │    │    │    │    │    │    │              │
│  feat/  │    │    │    │    │    │    │    │    │    │              │
│  ────── ●────┘    │    │    │    │    │    │    │    │              │
│                   │    │    │    │    │    │    │    │              │
│  feat/            │    │    │    │    │    │    │    │              │
│  ──────────●──────┘    │    │    │    │    │    │    │              │
│                        │    │    │    │    │    │    │              │
│                        ▼    │    │    │    │    │    │              │
│  release/         ┌────●────┼────┘    │    │    │    │              │
│  v1.2.0 ──────────┤         │         │    │    │    │              │
│                   │  tag    │         │    │    │    │              │
│                   │  v1.2.0 │         │    │    │    │              │
│                   └─────────┘         │    │    │    │              │
│                                       ▼    │    │    │              │
│  release/                        ┌────●────┼────┘    │              │
│  v1.3.0 ─────────────────────────┤         │         │              │
│                                  │  tag    │         │              │
│                                  │  v1.3.0 │         │              │
│                                  └─────────┘         │              │
│                                                      │              │
│  hotfix/                                             │              │
│  ─────── ●───────────────────────────────────────────┴──►           │
│          │                                                          │
│          └─► Cherry-pick to release & main                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 バージョニング規則

```yaml
# セマンティックバージョニング
version_format: "MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD]"

rules:
  major:
    trigger: "破壊的変更（後方互換性なし）"
    examples:
      - "API v1 → v2 移行"
      - "データベーススキーマの大規模変更"
      - "認証方式の変更"
    approval: "アーキテクチャレビュー必須"

  minor:
    trigger: "新機能追加（後方互換性あり）"
    examples:
      - "新しいAPIエンドポイント"
      - "新しいUI機能"
      - "新しい設定オプション"
    approval: "通常PRレビュー"

  patch:
    trigger: "バグ修正、セキュリティパッチ"
    examples:
      - "バグ修正"
      - "パフォーマンス改善"
      - "依存関係更新"
    approval: "通常PRレビュー"

  prerelease:
    format: "alpha.N, beta.N, rc.N"
    usage: "内部テスト、ステージング"

  build:
    format: "git commit hash (short)"
    usage: "ビルド識別"

examples:
  - "1.0.0"           # 初期リリース
  - "1.1.0"           # 新機能追加
  - "1.1.1"           # バグ修正
  - "2.0.0-alpha.1"   # メジャーバージョン アルファ
  - "2.0.0-beta.1"    # ベータテスト
  - "2.0.0-rc.1"      # リリース候補
  - "2.0.0+abc1234"   # ビルドメタデータ付き
```

### 2.4 リリースカレンダー

```yaml
# リリースカレンダー設定
release_calendar:
  regular_releases:
    day: Tuesday
    time: "10:00 JST"
    reason: "週初めを避け、問題発生時に週内で対応可能"

  blocked_periods:
    - name: "年末年始"
      start: "12-28"
      end: "01-04"
      reason: "サポート体制縮小"

    - name: "ゴールデンウィーク"
      start: "04-29"
      end: "05-05"
      reason: "サポート体制縮小"

    - name: "お盆"
      start: "08-13"
      end: "08-16"
      reason: "サポート体制縮小"

  high_traffic_caution:
    - name: "旅行シーズン"
      periods:
        - "03-15 to 04-10"  # 春休み
        - "07-20 to 08-31"  # 夏休み
        - "12-20 to 01-10"  # 年末年始
      policy: "重要機能のみ、段階的ロールアウト必須"

  maintenance_windows:
    - day: Sunday
      time: "02:00-06:00 JST"
      purpose: "定期メンテナンス"
```

---

## 3. リリースプロセス

### 3.1 リリースチェックリスト

```markdown
## リリース前チェックリスト

### コード品質
- [ ] 全テストパス（Unit, Integration, E2E）
- [ ] コードカバレッジ 80%以上
- [ ] Lint/Format チェックパス
- [ ] セキュリティスキャン（SAST/DAST）パス
- [ ] 依存関係の脆弱性チェックパス

### レビュー
- [ ] PRレビュー完了（2名以上承認）
- [ ] アーキテクチャレビュー（必要な場合）
- [ ] セキュリティレビュー（必要な場合）
- [ ] UXレビュー（UIchanges）

### ドキュメント
- [ ] CHANGELOG更新
- [ ] API ドキュメント更新（API変更時）
- [ ] 運用ドキュメント更新（必要な場合）
- [ ] リリースノート作成

### 環境
- [ ] Staging環境でのテスト完了
- [ ] パフォーマンステスト完了
- [ ] ロールバック手順確認
- [ ] フィーチャーフラグ設定確認

### 通知
- [ ] リリース予定の社内通知
- [ ] オンコールチームへの連絡
- [ ] 必要に応じてカスタマーサポートへ通知
```

### 3.2 リリースワークフロー

```yaml
# .github/workflows/release.yaml
name: Release Workflow

on:
  push:
    tags:
      - 'v*'

env:
  REGISTRY: asia-northeast1-docker.pkg.dev
  PROJECT_ID: triptrip-production
  IMAGE_NAME: triptrip-api

jobs:
  validate:
    name: Validate Release
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Validate version format
        run: |
          TAG=${GITHUB_REF#refs/tags/}
          if [[ ! $TAG =~ ^v[0-9]+\.[0-9]+\.[0-9]+(-[a-zA-Z0-9.]+)?$ ]]; then
            echo "Invalid version format: $TAG"
            exit 1
          fi

      - name: Check CHANGELOG
        run: |
          VERSION=${GITHUB_REF#refs/tags/v}
          if ! grep -q "## \[$VERSION\]" CHANGELOG.md; then
            echo "Version $VERSION not found in CHANGELOG.md"
            exit 1
          fi

  build:
    name: Build & Push Image
    needs: validate
    runs-on: ubuntu-latest
    outputs:
      image_digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Configure Docker
        run: |
          gcloud auth configure-docker asia-northeast1-docker.pkg.dev

      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}
            ${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  staging-deploy:
    name: Deploy to Staging
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Staging
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: triptrip-api-staging
          region: asia-northeast1
          image: ${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}

      - name: Run Smoke Tests
        run: |
          ./scripts/smoke-test.sh https://staging-api.triptrip.com

      - name: Run E2E Tests
        run: |
          npm run test:e2e -- --baseUrl=https://staging-api.triptrip.com

  production-deploy:
    name: Deploy to Production
    needs: staging-deploy
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Notify Release Start
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: 'C0123456789'
          slack-message: |
            :rocket: Release ${{ github.ref_name }} deployment starting
            Commit: ${{ github.sha }}
            By: ${{ github.actor }}

      - name: Canary Deploy (10%)
        run: |
          kubectl apply -f k8s/canary-deployment.yaml
          kubectl set image deployment/triptrip-api-canary \
            api=${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}

      - name: Monitor Canary (5 minutes)
        run: |
          ./scripts/monitor-canary.sh --duration=300 --threshold=1

      - name: Promote to 50%
        run: |
          kubectl scale deployment/triptrip-api-canary --replicas=5
          kubectl scale deployment/triptrip-api --replicas=5

      - name: Monitor 50% (10 minutes)
        run: |
          ./scripts/monitor-canary.sh --duration=600 --threshold=0.5

      - name: Full Rollout
        run: |
          kubectl set image deployment/triptrip-api \
            api=${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}
          kubectl rollout status deployment/triptrip-api --timeout=300s
          kubectl delete deployment triptrip-api-canary

      - name: Notify Release Complete
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: 'C0123456789'
          slack-message: |
            :white_check_mark: Release ${{ github.ref_name }} deployed successfully
            Duration: ${{ steps.timer.outputs.duration }}

  create-release:
    name: Create GitHub Release
    needs: production-deploy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Extract changelog
        id: changelog
        run: |
          VERSION=${GITHUB_REF#refs/tags/v}
          CHANGELOG=$(sed -n "/## \[$VERSION\]/,/## \[/p" CHANGELOG.md | sed '$d')
          echo "content<<EOF" >> $GITHUB_OUTPUT
          echo "$CHANGELOG" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          body: ${{ steps.changelog.outputs.content }}
          draft: false
          prerelease: ${{ contains(github.ref, 'alpha') || contains(github.ref, 'beta') }}
```

### 3.3 リリース承認フロー

```
┌─────────────────────────────────────────────────────────────────────┐
│                    リリース承認フロー                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Regular Release                                                     │
│  ───────────────                                                     │
│  PR Merge → Staging Deploy → 自動テスト → Staging承認 → Prod Deploy │
│                                    │                                 │
│                              E2E Pass = 自動承認                     │
│                              E2E Fail = 手動確認                     │
│                                                                      │
│  Hotfix Release                                                      │
│  ──────────────                                                      │
│  PR作成 → Tech Lead承認 → Staging → Prod (Canary skip可)            │
│                                                                      │
│  Security Release                                                    │
│  ────────────────                                                    │
│  PR作成 → Security Team承認 → Tech Lead承認 → Staging → Prod        │
│                                                                      │
│  Infrastructure Release                                              │
│  ──────────────────────                                              │
│  PR作成 → Platform Team承認 → 影響チーム承認 → Staging → Prod       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.4 GitHub環境設定

```yaml
# GitHub Environments Configuration
environments:
  staging:
    deployment_branch_policy:
      protected_branches: true
    reviewers: []  # 自動デプロイ
    wait_timer: 0

  production:
    deployment_branch_policy:
      protected_branches: true
      custom_branch_policies:
        - "release/*"
    reviewers:
      - teams:
          - release-managers
    wait_timer: 0
    # Canary成功後に自動進行

  production-hotfix:
    deployment_branch_policy:
      custom_branch_policies:
        - "hotfix/*"
    reviewers:
      - users:
          - tech-lead-1
          - tech-lead-2
    wait_timer: 0
```

---

## 4. デプロイメント戦略

### 4.1 デプロイメント方式比較

| 方式 | リスク | ロールバック速度 | リソース | 用途 |
|------|--------|------------------|----------|------|
| Rolling | 中 | 中 | 低 | 通常更新 |
| Blue-Green | 低 | 即時 | 高（2倍） | 重要リリース |
| Canary | 低 | 即時 | 中 | 新機能検証 |
| Shadow | 最低 | - | 高 | トラフィックテスト |

### 4.2 Canaryデプロイメント

```yaml
# k8s/canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: triptrip-api-canary
  namespace: triptrip
  labels:
    app: triptrip-api
    track: canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: triptrip-api
      track: canary
  template:
    metadata:
      labels:
        app: triptrip-api
        track: canary
      annotations:
        prometheus.io/scrape: "true"
    spec:
      containers:
        - name: api
          image: asia-northeast1-docker.pkg.dev/triptrip-production/triptrip-api:canary
          ports:
            - containerPort: 3000
          env:
            - name: DEPLOYMENT_TRACK
              value: "canary"
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 512Mi
---
# Istio VirtualService for traffic splitting
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: triptrip-api
  namespace: triptrip
spec:
  hosts:
    - triptrip-api
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: triptrip-api
            subset: canary
    - route:
        - destination:
            host: triptrip-api
            subset: stable
          weight: 90
        - destination:
            host: triptrip-api
            subset: canary
          weight: 10
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: triptrip-api
  namespace: triptrip
spec:
  host: triptrip-api
  subsets:
    - name: stable
      labels:
        track: stable
    - name: canary
      labels:
        track: canary
```

### 4.3 Canary分析

```typescript
// scripts/canary-analysis.ts
interface CanaryMetrics {
  errorRate: number;
  p99Latency: number;
  requestCount: number;
}

interface CanaryConfig {
  maxErrorRateDiff: number;  // 許容エラー率差（%ポイント）
  maxLatencyDiff: number;    // 許容レイテンシ差（%）
  minRequestCount: number;   // 最小リクエスト数
  analysisDuration: number;  // 分析時間（秒）
}

const defaultConfig: CanaryConfig = {
  maxErrorRateDiff: 0.5,     // 0.5%ポイント以内
  maxLatencyDiff: 10,        // 10%以内
  minRequestCount: 100,
  analysisDuration: 300,     // 5分
};

async function analyzeCanary(config: CanaryConfig = defaultConfig): Promise<boolean> {
  const [stableMetrics, canaryMetrics] = await Promise.all([
    getMetrics('stable', config.analysisDuration),
    getMetrics('canary', config.analysisDuration),
  ]);

  // リクエスト数チェック
  if (canaryMetrics.requestCount < config.minRequestCount) {
    console.log(`Insufficient traffic: ${canaryMetrics.requestCount} < ${config.minRequestCount}`);
    return false; // 判断保留
  }

  // エラー率比較
  const errorRateDiff = canaryMetrics.errorRate - stableMetrics.errorRate;
  if (errorRateDiff > config.maxErrorRateDiff) {
    console.error(`Error rate too high: Canary ${canaryMetrics.errorRate}% vs Stable ${stableMetrics.errorRate}%`);
    return false;
  }

  // レイテンシ比較
  const latencyDiff = ((canaryMetrics.p99Latency - stableMetrics.p99Latency) / stableMetrics.p99Latency) * 100;
  if (latencyDiff > config.maxLatencyDiff) {
    console.error(`Latency too high: Canary ${canaryMetrics.p99Latency}ms vs Stable ${stableMetrics.p99Latency}ms`);
    return false;
  }

  console.log('Canary analysis passed');
  console.log(`  Error rate: Canary ${canaryMetrics.errorRate}% vs Stable ${stableMetrics.errorRate}%`);
  console.log(`  P99 Latency: Canary ${canaryMetrics.p99Latency}ms vs Stable ${stableMetrics.p99Latency}ms`);

  return true;
}

async function getMetrics(track: string, duration: number): Promise<CanaryMetrics> {
  // DataDog API からメトリクス取得
  const query = `
    avg:triptrip.http.request.duration.99percentile{track:${track}},
    sum:triptrip.http.request.count{track:${track},status_class:5xx},
    sum:triptrip.http.request.count{track:${track}}
  `;

  const response = await datadogClient.query({
    query,
    from: Date.now() - duration * 1000,
    to: Date.now(),
  });

  return {
    p99Latency: response.series[0].pointlist[0][1],
    errorRate: (response.series[1].pointlist[0][1] / response.series[2].pointlist[0][1]) * 100,
    requestCount: response.series[2].pointlist[0][1],
  };
}
```

### 4.4 プログレッシブロールアウト

```yaml
# Argo Rollouts Configuration
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: triptrip-api
  namespace: triptrip
spec:
  replicas: 10
  selector:
    matchLabels:
      app: triptrip-api
  template:
    metadata:
      labels:
        app: triptrip-api
    spec:
      containers:
        - name: api
          image: asia-northeast1-docker.pkg.dev/triptrip-production/triptrip-api:v1.2.0
  strategy:
    canary:
      # 段階的ロールアウト設定
      steps:
        - setWeight: 5
        - pause: { duration: 2m }
        - analysis:
            templates:
              - templateName: success-rate
            args:
              - name: service-name
                value: triptrip-api
        - setWeight: 20
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: success-rate
        - setWeight: 50
        - pause: { duration: 10m }
        - analysis:
            templates:
              - templateName: success-rate
        - setWeight: 100

      # 自動ロールバック条件
      abortScaleDownDelaySeconds: 30

      # トラフィック管理
      trafficRouting:
        istio:
          virtualService:
            name: triptrip-api
            routes:
              - primary

      # 分析テンプレート参照
      analysis:
        successfulRunHistoryLimit: 3
        unsuccessfulRunHistoryLimit: 3
---
# Analysis Template
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: triptrip
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.99
      failureLimit: 3
      provider:
        datadog:
          apiVersion: v1
          query: |
            sum:triptrip.http.request.count{service:{{args.service-name}},status_class:2xx} /
            sum:triptrip.http.request.count{service:{{args.service-name}}}

    - name: latency-p99
      interval: 1m
      successCondition: result[0] < 500
      failureLimit: 3
      provider:
        datadog:
          query: |
            avg:triptrip.http.request.duration.99percentile{service:{{args.service-name}}}
```

---

## 5. フィーチャーフラグ管理

### 5.1 フィーチャーフラグ戦略

```
┌─────────────────────────────────────────────────────────────────────┐
│                    フィーチャーフラグの種類                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Release Flags (リリースフラグ)                                   │
│     └─ 新機能のON/OFF制御                                           │
│     └─ 一時的、リリース完了後に削除                                  │
│     └─ 例: new_booking_flow_enabled                                 │
│                                                                      │
│  2. Ops Flags (運用フラグ)                                          │
│     └─ 運用時の動作制御                                              │
│     └─ 長期的、サーキットブレーカー的用途                            │
│     └─ 例: external_payment_enabled                                 │
│                                                                      │
│  3. Experiment Flags (実験フラグ)                                    │
│     └─ A/Bテスト、ユーザーセグメント                                 │
│     └─ 中期的、実験完了後に削除                                      │
│     └─ 例: new_search_algorithm_variant                             │
│                                                                      │
│  4. Permission Flags (権限フラグ)                                    │
│     └─ ユーザー/グループ別機能制御                                   │
│     └─ 長期的、プランベース機能                                      │
│     └─ 例: premium_features_enabled                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 LaunchDarkly統合

```typescript
// src/infrastructure/feature-flags/launchdarkly.ts
import * as LaunchDarkly from 'launchdarkly-node-server-sdk';

// LaunchDarkly クライアント初期化
const ldClient = LaunchDarkly.init(process.env.LAUNCHDARKLY_SDK_KEY!, {
  logger: LaunchDarkly.basicLogger({ level: 'info' }),
});

// フラグ定義
export const FeatureFlags = {
  // Release Flags
  NEW_BOOKING_FLOW: 'new-booking-flow',
  AI_TRIP_SUGGESTIONS: 'ai-trip-suggestions',
  REAL_TIME_PRICING: 'real-time-pricing',

  // Ops Flags
  PAYMENT_GATEWAY_ENABLED: 'payment-gateway-enabled',
  EXTERNAL_MAP_API_ENABLED: 'external-map-api-enabled',
  RATE_LIMITING_STRICT: 'rate-limiting-strict',

  // Experiment Flags
  SEARCH_ALGORITHM_VARIANT: 'search-algorithm-variant',
  CHECKOUT_UI_VARIANT: 'checkout-ui-variant',

  // Permission Flags
  PREMIUM_FEATURES: 'premium-features',
  BETA_TESTER_FEATURES: 'beta-tester-features',
} as const;

// ユーザーコンテキスト
interface UserContext {
  key: string;
  email?: string;
  name?: string;
  custom?: {
    plan?: string;
    country?: string;
    signupDate?: string;
    deviceType?: string;
  };
}

// フィーチャーフラグサービス
export class FeatureFlagService {
  async isEnabled(
    flag: string,
    user: UserContext,
    defaultValue: boolean = false
  ): Promise<boolean> {
    try {
      await ldClient.waitForInitialization();
      const ldUser = this.toLDUser(user);
      return await ldClient.variation(flag, ldUser, defaultValue);
    } catch (error) {
      console.error(`Feature flag error for ${flag}:`, error);
      return defaultValue;
    }
  }

  async getVariation<T>(
    flag: string,
    user: UserContext,
    defaultValue: T
  ): Promise<T> {
    try {
      await ldClient.waitForInitialization();
      const ldUser = this.toLDUser(user);
      return await ldClient.variation(flag, ldUser, defaultValue);
    } catch (error) {
      console.error(`Feature flag error for ${flag}:`, error);
      return defaultValue;
    }
  }

  // A/Bテスト用バリアント取得
  async getExperimentVariant(
    flag: string,
    user: UserContext
  ): Promise<{ variant: string; inExperiment: boolean }> {
    const detail = await ldClient.variationDetail(
      flag,
      this.toLDUser(user),
      'control'
    );

    return {
      variant: detail.value,
      inExperiment: detail.reason?.inExperiment ?? false,
    };
  }

  private toLDUser(user: UserContext): LaunchDarkly.LDUser {
    return {
      key: user.key,
      email: user.email,
      name: user.name,
      custom: user.custom,
    };
  }
}

export const featureFlags = new FeatureFlagService();
```

### 5.3 フィーチャーフラグ使用例

```typescript
// src/routes/booking.ts
import { featureFlags, FeatureFlags } from '../infrastructure/feature-flags';

export const bookingRoutes = new Hono();

bookingRoutes.post('/create', async (c) => {
  const user = c.get('user');
  const userContext = {
    key: user.id,
    email: user.email,
    custom: {
      plan: user.plan,
      country: user.country,
    },
  };

  // リリースフラグ: 新しい予約フロー
  const useNewBookingFlow = await featureFlags.isEnabled(
    FeatureFlags.NEW_BOOKING_FLOW,
    userContext
  );

  if (useNewBookingFlow) {
    return await handleNewBookingFlow(c);
  } else {
    return await handleLegacyBookingFlow(c);
  }
});

// 運用フラグ: 決済ゲートウェイ
async function processPayment(booking: Booking, userContext: UserContext) {
  const paymentEnabled = await featureFlags.isEnabled(
    FeatureFlags.PAYMENT_GATEWAY_ENABLED,
    userContext
  );

  if (!paymentEnabled) {
    throw new ServiceUnavailableError('Payment processing is temporarily disabled');
  }

  return await paymentService.process(booking);
}

// 実験フラグ: 検索アルゴリズム A/B テスト
bookingRoutes.get('/search', async (c) => {
  const user = c.get('user');
  const userContext = { key: user.id };

  const { variant, inExperiment } = await featureFlags.getExperimentVariant(
    FeatureFlags.SEARCH_ALGORITHM_VARIANT,
    userContext
  );

  // 分析用にバリアントを記録
  analytics.track('search_performed', {
    userId: user.id,
    searchAlgorithmVariant: variant,
    inExperiment,
  });

  switch (variant) {
    case 'ml_ranking':
      return await mlSearchService.search(c.req.query());
    case 'hybrid':
      return await hybridSearchService.search(c.req.query());
    default:
      return await defaultSearchService.search(c.req.query());
  }
});
```

### 5.4 Flutterクライアント統合

```dart
// lib/infrastructure/feature_flags/feature_flag_service.dart
import 'package:launchdarkly_flutter_client_sdk/launchdarkly_flutter_client_sdk.dart';

class FeatureFlagService {
  static final FeatureFlagService _instance = FeatureFlagService._internal();
  factory FeatureFlagService() => _instance;
  FeatureFlagService._internal();

  late LDClient _client;
  bool _initialized = false;

  // フラグ定義
  static const String newBookingFlow = 'new-booking-flow';
  static const String aiTripSuggestions = 'ai-trip-suggestions';
  static const String darkModeEnabled = 'dark-mode-enabled';

  Future<void> initialize(String userId, {String? email}) async {
    final config = LDConfig(
      'mobile-key-xxx',
      AutoEnvAttributes.enabled,
    );

    final context = LDContextBuilder()
        .kind('user', userId)
        .set('email', LDValue.ofString(email ?? ''))
        .build();

    _client = LDClient(config, context);
    await _client.start();
    _initialized = true;
  }

  bool isEnabled(String flagKey, {bool defaultValue = false}) {
    if (!_initialized) return defaultValue;
    return _client.boolVariation(flagKey, defaultValue);
  }

  String getStringVariation(String flagKey, String defaultValue) {
    if (!_initialized) return defaultValue;
    return _client.stringVariation(flagKey, defaultValue);
  }

  // リアルタイム更新のリスナー
  void addFlagChangeListener(String flagKey, Function(LDEvaluationDetail) callback) {
    _client.registerFeatureFlagListener(flagKey, callback);
  }

  Future<void> identify(String userId, {Map<String, dynamic>? attributes}) async {
    final contextBuilder = LDContextBuilder().kind('user', userId);

    attributes?.forEach((key, value) {
      contextBuilder.set(key, LDValue.ofDynamic(value));
    });

    await _client.identify(contextBuilder.build());
  }

  void dispose() {
    _client.close();
  }
}

// 使用例
class BookingScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final featureFlags = FeatureFlagService();
    final useNewFlow = featureFlags.isEnabled(FeatureFlagService.newBookingFlow);

    return useNewFlow
        ? NewBookingFlowWidget()
        : LegacyBookingFlowWidget();
  }
}
```

### 5.5 フィーチャーフラグライフサイクル

```yaml
# フラグライフサイクル管理
flag_lifecycle:
  stages:
    - name: "Development"
      description: "開発環境でのみ有効"
      targeting:
        - environment: development
          enabled: true
        - environment: staging
          enabled: false
        - environment: production
          enabled: false

    - name: "Internal Testing"
      description: "社内テスター向けに有効"
      targeting:
        - environment: staging
          enabled: true
          rules:
            - attribute: email
              operator: endsWith
              value: "@triptrip.com"

    - name: "Beta"
      description: "ベータユーザー向けに展開"
      targeting:
        - environment: production
          enabled: true
          rules:
            - attribute: custom.betaTester
              operator: equals
              value: true
          percentage: 10

    - name: "Gradual Rollout"
      description: "段階的に全ユーザーへ展開"
      targeting:
        - environment: production
          enabled: true
          percentage: 25  # → 50 → 75 → 100

    - name: "General Availability"
      description: "全ユーザーに有効"
      targeting:
        - environment: production
          enabled: true

    - name: "Cleanup"
      description: "フラグをコードから削除"
      action: "Remove flag checks from codebase"

  retention:
    release_flags: "90 days after GA"
    experiment_flags: "30 days after experiment end"
    ops_flags: "Permanent (review annually)"
    permission_flags: "Permanent"
```

---

## 6. ロールバック手順

### 6.1 ロールバック判断基準

```yaml
# 自動ロールバックトリガー
auto_rollback_triggers:
  - name: "Error Rate Spike"
    condition: "error_rate > 5%"
    duration: "2 minutes"
    action: "immediate_rollback"

  - name: "Latency Degradation"
    condition: "p99_latency > 5000ms"
    duration: "5 minutes"
    action: "immediate_rollback"

  - name: "Health Check Failures"
    condition: "health_check_failures > 3"
    duration: "1 minute"
    action: "immediate_rollback"

  - name: "Crash Loop"
    condition: "pod_restarts > 5"
    duration: "5 minutes"
    action: "immediate_rollback"

# 手動ロールバック判断
manual_rollback_criteria:
  - "ユーザーからの重大な障害報告"
  - "ビジネスメトリクスの異常低下"
  - "セキュリティ脆弱性の発見"
  - "データ整合性の問題"
```

### 6.2 ロールバック手順書

```markdown
# ロールバック手順

## 1. 即時判断（1分以内）

### 状況確認
1. DataDogダッシュボードで異常を確認
2. エラーログの内容を確認
3. 影響範囲を特定（全体/一部）

### ロールバック判断
- エラー率 > 5% → 即時ロールバック
- P99レイテンシ > 5秒 → 即時ロールバック
- ユーザー影響大 → 即時ロールバック

## 2. ロールバック実行

### Option A: Kubernetes Rollback（推奨）
```bash
# 前バージョンへロールバック
kubectl rollout undo deployment/triptrip-api -n triptrip

# ロールバック確認
kubectl rollout status deployment/triptrip-api -n triptrip

# 特定リビジョンへロールバック
kubectl rollout undo deployment/triptrip-api -n triptrip --to-revision=3
```

### Option B: Argo Rollouts Abort
```bash
# Canaryを中止
kubectl argo rollouts abort triptrip-api -n triptrip

# Stableにロールバック
kubectl argo rollouts undo triptrip-api -n triptrip
```

### Option C: フィーチャーフラグ無効化
```bash
# LaunchDarkly API でフラグを無効化
curl -X PATCH "https://app.launchdarkly.com/api/v2/flags/triptrip/${FLAG_KEY}" \
  -H "Authorization: ${LD_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "patch": [{
      "op": "replace",
      "path": "/environments/production/on",
      "value": false
    }]
  }'
```

## 3. 復旧確認

### メトリクス確認
- [ ] エラー率が正常値（< 0.1%）に回復
- [ ] レイテンシが正常値（P99 < 500ms）に回復
- [ ] リクエスト成功率が99%以上

### 機能確認
- [ ] ヘルスチェック全てパス
- [ ] 主要APIエンドポイント応答確認
- [ ] ユーザーフローの動作確認

## 4. 事後対応

### 即時
- [ ] インシデントSlackチャンネルで報告
- [ ] 必要に応じてステータスページ更新

### 24時間以内
- [ ] 根本原因の調査
- [ ] 修正PRの作成
- [ ] ポストモーテム（SEV-1/2の場合）
```

### 6.3 自動ロールバックスクリプト

```typescript
// scripts/auto-rollback.ts
import { KubernetesClient } from './kubernetes-client';
import { DatadogClient } from './datadog-client';
import { SlackNotifier } from './slack-notifier';

interface RollbackConfig {
  namespace: string;
  deployment: string;
  errorRateThreshold: number;
  latencyThreshold: number;
  checkInterval: number;
  checkDuration: number;
}

const config: RollbackConfig = {
  namespace: 'triptrip',
  deployment: 'triptrip-api',
  errorRateThreshold: 5,      // 5%
  latencyThreshold: 5000,      // 5秒
  checkInterval: 10000,        // 10秒ごとにチェック
  checkDuration: 300000,       // 5分間監視
};

async function monitorAndRollback(): Promise<void> {
  const k8s = new KubernetesClient();
  const datadog = new DatadogClient();
  const slack = new SlackNotifier();

  const startTime = Date.now();
  const currentVersion = await k8s.getCurrentImageTag(config.namespace, config.deployment);

  console.log(`Monitoring deployment: ${config.deployment}`);
  console.log(`Current version: ${currentVersion}`);

  while (Date.now() - startTime < config.checkDuration) {
    try {
      const metrics = await datadog.getMetrics({
        errorRate: `sum:triptrip.http.request.count{status_class:5xx,deployment:${config.deployment}}.as_rate() / sum:triptrip.http.request.count{deployment:${config.deployment}}.as_rate() * 100`,
        p99Latency: `avg:triptrip.http.request.duration.99percentile{deployment:${config.deployment}}`,
      });

      console.log(`Error Rate: ${metrics.errorRate}%, P99 Latency: ${metrics.p99Latency}ms`);

      // ロールバック条件チェック
      if (metrics.errorRate > config.errorRateThreshold) {
        console.error(`Error rate threshold exceeded: ${metrics.errorRate}% > ${config.errorRateThreshold}%`);
        await executeRollback(k8s, slack, `High error rate: ${metrics.errorRate}%`);
        return;
      }

      if (metrics.p99Latency > config.latencyThreshold) {
        console.error(`Latency threshold exceeded: ${metrics.p99Latency}ms > ${config.latencyThreshold}ms`);
        await executeRollback(k8s, slack, `High latency: ${metrics.p99Latency}ms`);
        return;
      }

      await sleep(config.checkInterval);
    } catch (error) {
      console.error('Monitoring error:', error);
    }
  }

  console.log('Monitoring completed. No rollback needed.');
  await slack.notify({
    channel: '#releases',
    message: `:white_check_mark: Deployment ${currentVersion} stable after ${config.checkDuration / 60000} minutes`,
  });
}

async function executeRollback(
  k8s: KubernetesClient,
  slack: SlackNotifier,
  reason: string
): Promise<void> {
  console.log(`Initiating rollback. Reason: ${reason}`);

  // Slack通知
  await slack.notify({
    channel: '#incidents',
    message: `:rotating_light: AUTO ROLLBACK INITIATED\nDeployment: ${config.deployment}\nReason: ${reason}`,
    priority: 'high',
  });

  // ロールバック実行
  await k8s.rollback(config.namespace, config.deployment);

  // ロールバック完了待機
  await k8s.waitForRollout(config.namespace, config.deployment);

  // 完了通知
  await slack.notify({
    channel: '#incidents',
    message: `:white_check_mark: Rollback completed for ${config.deployment}`,
  });

  console.log('Rollback completed successfully');
}

function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// 実行
monitorAndRollback().catch(console.error);
```

### 6.4 ロールバック履歴管理

```yaml
# ロールバック履歴テンプレート
rollback_record:
  id: "RB-2026-001"
  timestamp: "2026-01-20T14:30:00+09:00"
  deployment: "triptrip-api"
  from_version: "v1.5.2"
  to_version: "v1.5.1"
  trigger:
    type: "automatic"
    reason: "Error rate exceeded 5% threshold"
    metrics:
      error_rate: "7.2%"
      p99_latency: "450ms"
  duration:
    detection_time: "2 minutes"
    rollback_time: "45 seconds"
    recovery_time: "3 minutes"
  impact:
    affected_users: "~500"
    affected_requests: "~2000"
  root_cause: "Database connection pool exhaustion due to query timeout"
  follow_up_actions:
    - "Increase connection pool size"
    - "Add query timeout"
    - "Update monitoring alerts"
  postmortem_link: "https://notion.so/triptrip/PM-2026-001"
```

---

## 7. 緊急リリース対応

### 7.1 Hotfixプロセス

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Hotfix リリースフロー                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  問題検知                                                            │
│     │                                                                │
│     ▼                                                                │
│  ┌─────────────┐                                                    │
│  │ SEV判定     │                                                    │
│  │ P1/P2?     │                                                    │
│  └──────┬──────┘                                                    │
│         │                                                            │
│    Yes  │  No → 通常リリースプロセス                                 │
│         │                                                            │
│         ▼                                                            │
│  ┌─────────────┐                                                    │
│  │ Hotfix      │  ← mainから直接ブランチ                            │
│  │ ブランチ作成 │     hotfix/issue-XXX                              │
│  └──────┬──────┘                                                    │
│         │                                                            │
│         ▼                                                            │
│  ┌─────────────┐                                                    │
│  │ 修正実装    │  ← 最小限の変更                                    │
│  │ + テスト   │     影響範囲を限定                                  │
│  └──────┬──────┘                                                    │
│         │                                                            │
│         ▼                                                            │
│  ┌─────────────┐                                                    │
│  │ 緊急PR      │  ← Tech Lead必須レビュー                           │
│  │ レビュー    │     1名でもマージ可                                │
│  └──────┬──────┘                                                    │
│         │                                                            │
│         ▼                                                            │
│  ┌─────────────┐                                                    │
│  │ Staging     │  ← スモークテストのみ                              │
│  │ 検証       │     (E2E省略可)                                     │
│  └──────┬──────┘                                                    │
│         │                                                            │
│         ▼                                                            │
│  ┌─────────────┐                                                    │
│  │ Production  │  ← Canary省略可                                    │
│  │ デプロイ    │     直接ローリングアップデート                      │
│  └──────┬──────┘                                                    │
│         │                                                            │
│         ▼                                                            │
│  ┌─────────────┐                                                    │
│  │ 監視強化    │  ← 30分間の重点監視                                │
│  │ (30分)     │                                                     │
│  └──────┬──────┘                                                    │
│         │                                                            │
│         ▼                                                            │
│  ┌─────────────┐                                                    │
│  │ main へ     │  ← Cherry-pick                                     │
│  │ バックポート│                                                    │
│  └─────────────┘                                                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 Hotfixチェックリスト

```markdown
## Hotfix チェックリスト

### 開始前
- [ ] 問題のSEV判定完了（P1/P2のみHotfix対象）
- [ ] インシデントチャンネル作成済み
- [ ] オンコールエンジニアがアサイン済み

### 開発
- [ ] hotfix/ブランチをmainから作成
- [ ] 変更は最小限に限定
- [ ] 単体テストを追加/更新
- [ ] ローカルで動作確認

### レビュー
- [ ] Tech Lead/Senior Engineerがレビュー
- [ ] セキュリティ影響がないか確認
- [ ] ロールバック方法を確認

### デプロイ
- [ ] Staging環境でスモークテスト
- [ ] Production デプロイ開始を通知
- [ ] デプロイ実行
- [ ] ヘルスチェック確認

### 事後
- [ ] 30分間の監視完了
- [ ] mainブランチへcherry-pick
- [ ] 次回リリースブランチへcherry-pick
- [ ] インシデントレポート更新
- [ ] （該当する場合）ポストモーテムスケジュール
```

### 7.3 Hotfixワークフロー

```yaml
# .github/workflows/hotfix.yaml
name: Hotfix Deployment

on:
  push:
    branches:
      - 'hotfix/**'

jobs:
  validate:
    name: Validate Hotfix
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Check branch naming
        run: |
          BRANCH=${GITHUB_REF#refs/heads/}
          if [[ ! $BRANCH =~ ^hotfix/[A-Z]+-[0-9]+-.+ ]]; then
            echo "Invalid hotfix branch name. Use: hotfix/ISSUE-123-description"
            exit 1
          fi

      - name: Lint and Test
        run: |
          npm ci
          npm run lint
          npm run test

  staging:
    name: Deploy to Staging
    needs: validate
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4

      - name: Build Image
        run: |
          docker build -t triptrip-api:hotfix .

      - name: Deploy to Staging
        run: |
          kubectl set image deployment/triptrip-api-staging \
            api=asia-northeast1-docker.pkg.dev/triptrip-staging/triptrip-api:hotfix

      - name: Smoke Test
        run: |
          ./scripts/smoke-test.sh https://staging-api.triptrip.com

  production:
    name: Deploy to Production (Hotfix)
    needs: staging
    runs-on: ubuntu-latest
    environment: production-hotfix
    steps:
      - uses: actions/checkout@v4

      - name: Notify Hotfix Start
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: 'C0123456789'
          slack-message: |
            :fire: HOTFIX deployment starting
            Branch: ${{ github.ref_name }}
            PR: ${{ github.event.pull_request.html_url }}

      - name: Deploy (Skip Canary)
        run: |
          kubectl set image deployment/triptrip-api \
            api=asia-northeast1-docker.pkg.dev/triptrip-production/triptrip-api:hotfix
          kubectl rollout status deployment/triptrip-api --timeout=300s

      - name: Enhanced Monitoring (30 min)
        run: |
          ./scripts/monitor-deployment.sh --duration=1800 --alert-threshold=0.5

      - name: Notify Hotfix Complete
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: 'C0123456789'
          slack-message: |
            :white_check_mark: HOTFIX deployed successfully
            Branch: ${{ github.ref_name }}
```

---

## 8. モバイルアプリリリース

### 8.1 モバイルリリースサイクル

```
┌─────────────────────────────────────────────────────────────────────┐
│                    モバイルアプリリリースサイクル                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Week 1          Week 2          Week 3          Week 4              │
│  ───────         ───────         ───────         ───────             │
│                                                                      │
│  開発            開発            QA/修正          リリース            │
│  ┌───────┐      ┌───────┐      ┌───────┐       ┌───────┐           │
│  │Feature│      │Feature│      │Internal│       │App     │           │
│  │Develop│      │Freeze │      │Testing │       │Store   │           │
│  └───┬───┘      └───┬───┘      └───┬───┘       │Submit  │           │
│      │              │              │            └───┬───┘           │
│      │              │              │                │                │
│      │              │              │                ▼                │
│      │              │              │           ┌───────┐            │
│      │              │              │           │Review  │            │
│      │              │              │           │(1-3days)│           │
│      │              │              │           └───┬───┘            │
│      │              │              │                │                │
│      │              │              │                ▼                │
│      │              │              │           ┌───────┐            │
│      │              │              │           │Phased  │            │
│      │              │              │           │Rollout │            │
│      │              │              │           └───────┘            │
│                                                                      │
│  ※ 隔週リリース（APIと1週間ずらし）                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.2 App Store/Google Play 配布設定

```yaml
# Fastlane Configuration
# fastlane/Fastfile

default_platform(:ios)

# iOS
platform :ios do
  desc "Build and upload to TestFlight"
  lane :beta do
    setup_ci

    match(type: "appstore", readonly: true)

    build_app(
      workspace: "TripTrip.xcworkspace",
      scheme: "TripTrip",
      export_options: {
        method: "app-store"
      }
    )

    upload_to_testflight(
      skip_waiting_for_build_processing: true
    )
  end

  desc "Release to App Store"
  lane :release do
    setup_ci

    match(type: "appstore", readonly: true)

    build_app(
      workspace: "TripTrip.xcworkspace",
      scheme: "TripTrip",
      export_options: {
        method: "app-store"
      }
    )

    upload_to_app_store(
      submit_for_review: true,
      automatic_release: false,
      phased_release: true,
      submission_information: {
        add_id_info_uses_idfa: false
      }
    )
  end
end

# Android
platform :android do
  desc "Build and upload to Internal Testing"
  lane :beta do
    gradle(
      task: "bundle",
      build_type: "Release",
      properties: {
        "android.injected.signing.store.file" => ENV["KEYSTORE_PATH"],
        "android.injected.signing.store.password" => ENV["KEYSTORE_PASSWORD"],
        "android.injected.signing.key.alias" => ENV["KEY_ALIAS"],
        "android.injected.signing.key.password" => ENV["KEY_PASSWORD"]
      }
    )

    upload_to_play_store(
      track: "internal",
      aab: "build/app/outputs/bundle/release/app-release.aab"
    )
  end

  desc "Release to Production"
  lane :release do
    gradle(
      task: "bundle",
      build_type: "Release"
    )

    upload_to_play_store(
      track: "production",
      rollout: "0.1",  # 10%から開始
      aab: "build/app/outputs/bundle/release/app-release.aab"
    )
  end

  desc "Increase rollout percentage"
  lane :increase_rollout do |options|
    upload_to_play_store(
      track: "production",
      rollout: options[:percentage],
      version_code: options[:version_code]
    )
  end
end
```

### 8.3 段階的ロールアウト（Google Play）

```yaml
# モバイルアプリ段階的ロールアウト計画
phased_rollout:
  platform: android

  stages:
    - day: 0
      percentage: 1
      duration: "24 hours"
      criteria:
        - crash_free_rate: ">= 99.5%"
        - anr_rate: "< 0.1%"
        - rating_trend: "stable"

    - day: 1
      percentage: 5
      duration: "24 hours"
      criteria:
        - crash_free_rate: ">= 99.5%"
        - anr_rate: "< 0.1%"
        - user_reviews: "no_critical_issues"

    - day: 2
      percentage: 20
      duration: "48 hours"
      criteria:
        - crash_free_rate: ">= 99.5%"
        - retention_rate: ">= previous_version"

    - day: 4
      percentage: 50
      duration: "48 hours"
      criteria:
        - crash_free_rate: ">= 99.5%"
        - business_metrics: "no_degradation"

    - day: 6
      percentage: 100
      criteria:
        - all_previous_criteria: "met"

  halt_conditions:
    - crash_free_rate: "< 99%"
    - anr_rate: "> 0.5%"
    - rating_drop: "> 0.5 stars"
    - critical_bug_reports: "> 10"

  monitoring:
    dashboard: "https://console.firebase.google.com/project/triptrip/crashlytics"
    alerts:
      - channel: "#mobile-releases"
      - email: "mobile-team@triptrip.com"
```

### 8.4 強制アップデート

```typescript
// src/config/app-version.ts
interface VersionConfig {
  minimumVersion: string;
  recommendedVersion: string;
  forceUpdate: boolean;
  updateMessage: string;
  storeUrls: {
    ios: string;
    android: string;
  };
}

const versionConfig: VersionConfig = {
  minimumVersion: "1.5.0",      // この版未満は強制更新
  recommendedVersion: "1.6.0",  // この版未満は推奨更新
  forceUpdate: true,
  updateMessage: "重要なセキュリティアップデートがあります。最新版に更新してください。",
  storeUrls: {
    ios: "https://apps.apple.com/app/triptrip/id123456789",
    android: "https://play.google.com/store/apps/details?id=com.triptrip.app",
  },
};

// API エンドポイント
app.get('/api/v1/app/version-check', (c) => {
  const clientVersion = c.req.header('X-App-Version');
  const platform = c.req.header('X-Platform');

  const needsForceUpdate = compareVersions(clientVersion, versionConfig.minimumVersion) < 0;
  const needsRecommendedUpdate = compareVersions(clientVersion, versionConfig.recommendedVersion) < 0;

  return c.json({
    forceUpdate: needsForceUpdate && versionConfig.forceUpdate,
    recommendedUpdate: needsRecommendedUpdate,
    message: needsForceUpdate ? versionConfig.updateMessage : null,
    storeUrl: platform === 'ios' ? versionConfig.storeUrls.ios : versionConfig.storeUrls.android,
    minimumVersion: versionConfig.minimumVersion,
    recommendedVersion: versionConfig.recommendedVersion,
  });
});
```

### 8.5 Flutter版バージョンチェック

```dart
// lib/infrastructure/version/version_checker.dart
import 'package:package_info_plus/package_info_plus.dart';
import 'package:url_launcher/url_launcher.dart';

class VersionChecker {
  final ApiClient _apiClient;

  VersionChecker(this._apiClient);

  Future<VersionCheckResult> checkVersion() async {
    final packageInfo = await PackageInfo.fromPlatform();
    final currentVersion = packageInfo.version;

    final response = await _apiClient.get('/api/v1/app/version-check');

    return VersionCheckResult(
      currentVersion: currentVersion,
      forceUpdate: response['forceUpdate'] ?? false,
      recommendedUpdate: response['recommendedUpdate'] ?? false,
      message: response['message'],
      storeUrl: response['storeUrl'],
    );
  }

  Future<void> openStore(String storeUrl) async {
    final uri = Uri.parse(storeUrl);
    if (await canLaunchUrl(uri)) {
      await launchUrl(uri, mode: LaunchMode.externalApplication);
    }
  }
}

class VersionCheckResult {
  final String currentVersion;
  final bool forceUpdate;
  final bool recommendedUpdate;
  final String? message;
  final String? storeUrl;

  VersionCheckResult({
    required this.currentVersion,
    required this.forceUpdate,
    required this.recommendedUpdate,
    this.message,
    this.storeUrl,
  });
}

// 使用例（main.dart）
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  final versionChecker = VersionChecker(getIt<ApiClient>());
  final result = await versionChecker.checkVersion();

  if (result.forceUpdate) {
    runApp(ForceUpdateApp(
      message: result.message!,
      storeUrl: result.storeUrl!,
    ));
  } else {
    runApp(TripTripApp(
      showUpdateBanner: result.recommendedUpdate,
    ));
  }
}
```

---

## 9. データベースマイグレーション

### 9.1 マイグレーション原則

```markdown
## データベースマイグレーションの原則

### 1. 後方互換性
- 新旧バージョンのアプリが同時に動作可能
- カラム追加はNULLableまたはデフォルト値付き
- カラム削除は2段階（非使用化 → 削除）

### 2. 段階的マイグレーション
- 大規模変更は複数の小さなマイグレーションに分割
- 各マイグレーションは独立して実行可能

### 3. ロールバック可能
- すべてのマイグレーションにdownスクリプトを用意
- データ損失を伴う変更は慎重に検討

### 4. テスト済み
- Staging環境で本番相当のデータ量でテスト
- パフォーマンス影響を事前に評価
```

### 9.2 Prismaマイグレーション

```typescript
// prisma/migrations/20260120_add_trip_categories/migration.sql
-- CreateTable
CREATE TABLE "trip_categories" (
    "id" UUID NOT NULL DEFAULT gen_random_uuid(),
    "name" VARCHAR(100) NOT NULL,
    "description" TEXT,
    "icon" VARCHAR(50),
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_at" TIMESTAMP(3) NOT NULL,

    CONSTRAINT "trip_categories_pkey" PRIMARY KEY ("id")
);

-- AddColumn (NULLable for backward compatibility)
ALTER TABLE "trip_plans" ADD COLUMN "category_id" UUID;

-- CreateIndex
CREATE INDEX "trip_plans_category_id_idx" ON "trip_plans"("category_id");

-- AddForeignKey
ALTER TABLE "trip_plans" ADD CONSTRAINT "trip_plans_category_id_fkey"
    FOREIGN KEY ("category_id") REFERENCES "trip_categories"("id")
    ON DELETE SET NULL ON UPDATE CASCADE;
```

```typescript
// prisma/migrations/20260120_add_trip_categories/down.sql
-- DropForeignKey
ALTER TABLE "trip_plans" DROP CONSTRAINT IF EXISTS "trip_plans_category_id_fkey";

-- DropColumn
ALTER TABLE "trip_plans" DROP COLUMN IF EXISTS "category_id";

-- DropTable
DROP TABLE IF EXISTS "trip_categories";
```

### 9.3 ゼロダウンタイムマイグレーション

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ゼロダウンタイムマイグレーション                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Step 1: 新カラム追加（NULLable）                                    │
│  ─────────────────────────────                                       │
│  ALTER TABLE users ADD COLUMN email_verified BOOLEAN;                │
│  ※ NULLableなのでアプリは影響なし                                   │
│                                                                      │
│  Step 2: 新カラムを使用するコードをデプロイ                          │
│  ─────────────────────────────────────────                           │
│  // 新コードは両方のカラムを処理                                     │
│  const verified = user.email_verified ?? user.is_verified;           │
│                                                                      │
│  Step 3: データマイグレーション（バッチ処理）                        │
│  ─────────────────────────────────────────                           │
│  UPDATE users SET email_verified = is_verified                       │
│  WHERE email_verified IS NULL;                                       │
│  ※ 小バッチで実行、本番負荷を監視                                   │
│                                                                      │
│  Step 4: NOT NULL制約追加                                            │
│  ─────────────────────────                                           │
│  ALTER TABLE users ALTER COLUMN email_verified SET NOT NULL;         │
│  ALTER TABLE users ALTER COLUMN email_verified SET DEFAULT false;    │
│                                                                      │
│  Step 5: 旧カラムの使用を停止するコードをデプロイ                    │
│  ─────────────────────────────────────────────────                   │
│  // 旧カラムの参照を削除                                             │
│                                                                      │
│  Step 6: 旧カラム削除                                                │
│  ─────────────────                                                   │
│  ALTER TABLE users DROP COLUMN is_verified;                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.4 マイグレーションワークフロー

```yaml
# .github/workflows/database-migration.yaml
name: Database Migration

on:
  push:
    paths:
      - 'prisma/migrations/**'
    branches:
      - main

jobs:
  staging-migration:
    name: Migrate Staging Database
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Connect to Cloud SQL (Staging)
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Run Migration (Staging)
        env:
          DATABASE_URL: ${{ secrets.STAGING_DATABASE_URL }}
        run: |
          npx prisma migrate deploy
          npx prisma db seed

      - name: Verify Migration
        env:
          DATABASE_URL: ${{ secrets.STAGING_DATABASE_URL }}
        run: |
          npx prisma migrate status

  production-migration:
    name: Migrate Production Database
    needs: staging-migration
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Notify Migration Start
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: 'C0123456789'
          slack-message: |
            :database: Database migration starting (Production)
            Migration: ${{ github.sha }}

      - name: Run Migration (Production)
        env:
          DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}
        run: |
          npx prisma migrate deploy

      - name: Verify Migration
        env:
          DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}
        run: |
          npx prisma migrate status

      - name: Notify Migration Complete
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: 'C0123456789'
          slack-message: |
            :white_check_mark: Database migration completed (Production)
```

---

## 10. リリースコミュニケーション

### 10.1 リリースノート

```markdown
# リリースノートテンプレート

## TripTrip v1.6.0 リリースノート

### リリース日
2026年1月20日

### 新機能
- **AI旅行提案**: AIがあなたの好みに基づいて旅行プランを提案します
- **リアルタイム価格**: 宿泊施設の価格がリアルタイムで更新されます
- **お気に入り共有**: 友達とお気に入りスポットを共有できるようになりました

### 改善
- 検索結果の表示速度が30%向上しました
- 予約確認画面のUIを改善しました
- 通知設定がより細かくカスタマイズできるようになりました

### バグ修正
- 特定の条件で地図が表示されない問題を修正
- プッシュ通知が重複して送信される問題を修正
- ダークモードでの一部テキストの視認性を改善

### 既知の問題
- iOS 15以下では一部のアニメーションが正常に動作しない場合があります
  （次回アップデートで修正予定）

### アップデート方法
App Store / Google Play から最新版をダウンロードしてください。

### フィードバック
ご意見・ご要望は support@triptrip.com までお寄せください。
```

### 10.2 社内通知テンプレート

```markdown
## リリース予定通知

### :calendar: リリース予定
- **日時**: 2026年1月20日（火）10:00 JST
- **バージョン**: v1.6.0
- **種類**: 定期リリース

### :package: 主要な変更
1. AI旅行提案機能の追加
2. リアルタイム価格表示
3. パフォーマンス改善

### :warning: 注意事項
- API変更: `/api/v1/search` にオプションパラメータ追加
- フィーチャーフラグ: `ai-trip-suggestions` は段階的に有効化

### :busts_in_silhouette: 担当
- リリースマネージャー: @release-manager
- オンコール: @oncall-primary

### :link: 関連リンク
- [リリースノート](https://notion.so/triptrip/release-notes-v1.6.0)
- [デプロイダッシュボード](https://app.datadoghq.com/dashboard/triptrip-deploy)
- [ロールバック手順](https://notion.so/triptrip/rollback-procedures)

---

## リリース完了通知

### :white_check_mark: リリース完了
- **バージョン**: v1.6.0
- **完了時刻**: 2026年1月20日 10:45 JST
- **所要時間**: 45分

### :chart_with_upwards_trend: メトリクス
- エラー率: 0.02% (正常)
- P99レイテンシ: 180ms (正常)
- Canary成功率: 100%

### :eyes: 監視状況
- 30分間の重点監視中
- 異常があれば #releases に報告してください

---

## リリースロールバック通知

### :rotating_light: ロールバック実施
- **バージョン**: v1.6.0 → v1.5.2
- **実施時刻**: 2026年1月20日 11:15 JST
- **理由**: エラー率上昇（5.2%）

### :memo: 状況
- 影響ユーザー: 約500人
- 復旧時刻: 11:20 JST

### :next_track_button: 次のステップ
- 原因調査中
- ポストモーテム予定: 1月21日 14:00
```

### 10.3 ステータスページ更新

```yaml
# Statuspage.io 設定
statuspage:
  components:
    - name: "TripTrip API"
      description: "Core API services"
      group: "Backend Services"

    - name: "TripTrip Web"
      description: "Web application"
      group: "Frontend"

    - name: "TripTrip Mobile (iOS)"
      description: "iOS mobile application"
      group: "Mobile Apps"

    - name: "TripTrip Mobile (Android)"
      description: "Android mobile application"
      group: "Mobile Apps"

  incident_templates:
    scheduled_maintenance:
      title: "[Scheduled] System Maintenance"
      body: |
        We will be performing scheduled maintenance on our systems.

        **Scheduled Time**: {{ start_time }} - {{ end_time }} JST
        **Affected Services**: {{ affected_components }}
        **Expected Impact**: {{ impact_description }}

        We apologize for any inconvenience.

    degraded_performance:
      title: "Performance Degradation"
      body: |
        We are currently experiencing degraded performance.

        **Affected Services**: {{ affected_components }}
        **Impact**: {{ impact_description }}

        Our team is investigating the issue.

    incident_resolved:
      title: "Incident Resolved"
      body: |
        The incident has been resolved.

        **Duration**: {{ duration }}
        **Root Cause**: {{ root_cause }}

        We apologize for the inconvenience caused.
```

---

## 11. リリースメトリクスと改善

### 11.1 DORAメトリクス

```yaml
# DORA Metrics Dashboard
dora_metrics:
  deployment_frequency:
    description: "本番環境へのデプロイ頻度"
    calculation: "deployments_count / time_period"
    targets:
      elite: "> 1/day"
      high: "1/week - 1/day"
      medium: "1/month - 1/week"
      low: "< 1/month"
    current: "3/week"
    target: "1/day"

  lead_time_for_changes:
    description: "コミットから本番デプロイまでの時間"
    calculation: "deploy_time - commit_time"
    targets:
      elite: "< 1 hour"
      high: "1 day - 1 week"
      medium: "1 week - 1 month"
      low: "> 1 month"
    current: "2 days"
    target: "< 1 day"

  change_failure_rate:
    description: "デプロイ後のロールバック/Hotfix率"
    calculation: "(rollbacks + hotfixes) / total_deployments * 100"
    targets:
      elite: "< 15%"
      high: "16% - 30%"
      medium: "31% - 45%"
      low: "> 45%"
    current: "12%"
    target: "< 10%"

  time_to_restore_service:
    description: "インシデント発生から復旧までの時間"
    calculation: "recovery_time - incident_start_time"
    targets:
      elite: "< 1 hour"
      high: "< 1 day"
      medium: "1 day - 1 week"
      low: "> 1 week"
    current: "45 minutes"
    target: "< 30 minutes"
```

### 11.2 リリース品質メトリクス

```yaml
release_quality_metrics:
  # デプロイ成功率
  deployment_success_rate:
    formula: "successful_deployments / total_deployments * 100"
    target: "> 95%"

  # Canary成功率
  canary_success_rate:
    formula: "canary_passed / canary_started * 100"
    target: "> 90%"

  # ロールバック率
  rollback_rate:
    formula: "rollbacks / total_deployments * 100"
    target: "< 5%"

  # 平均ロールバック時間
  mean_rollback_time:
    formula: "sum(rollback_duration) / rollback_count"
    target: "< 5 minutes"

  # リリースブロッカー数
  release_blockers:
    formula: "count(critical_issues_found_in_staging)"
    target: "< 1/release"

  # デプロイ所要時間
  deployment_duration:
    formula: "deploy_end_time - deploy_start_time"
    target: "< 30 minutes"
```

### 11.3 継続的改善プロセス

```markdown
## リリースプロセス継続的改善

### 月次レビュー
1. **メトリクス分析**
   - DORAメトリクスの推移確認
   - ボトルネックの特定
   - ベンチマークとの比較

2. **インシデント振り返り**
   - リリース起因インシデントの傾向
   - ロールバック理由の分析
   - 再発防止策の効果測定

3. **プロセス改善**
   - 自動化の機会特定
   - テストカバレッジの改善
   - ツールの評価・導入

### 四半期アクション
- リリースプロセスの大規模改善
- ツールチェーンのアップグレード
- チームトレーニング

### 年次計画
- DORA指標の年間目標設定
- 技術投資の優先順位付け
- 業界ベストプラクティスの取り込み
```

---

## 付録

### A. 関連文書

| 文書ID | タイトル | 関連性 |
|--------|----------|--------|
| Doc-DM-001 | アジャイル開発プロセス | スプリント計画との連携 |
| Doc-DM-002 | DevOps＆継続的デリバリー | CI/CDパイプライン詳細 |
| Doc-QA-001 | テスト戦略＆品質基準 | リリース前テスト要件 |
| Doc-QA-002 | モニタリング＆インシデント管理 | リリース後監視 |

### B. ツールリンク

| ツール | 用途 | URL |
|--------|------|-----|
| GitHub Actions | CI/CD | github.com/triptrip/api/actions |
| Argo Rollouts | Progressive Delivery | argocd.triptrip.internal |
| LaunchDarkly | Feature Flags | app.launchdarkly.com |
| Statuspage | Status Page | manage.statuspage.io |
| Fastlane | Mobile Release | fastlane.tools |

### C. 緊急連絡先

| 役割 | 連絡先 |
|------|--------|
| Release Manager | @release-manager (Slack) |
| On-Call Primary | PagerDuty: triptrip-primary |
| On-Call Secondary | PagerDuty: triptrip-secondary |
| Tech Lead | @tech-lead (Slack) |
| CTO | @cto (Slack, 緊急時のみ) |

---

**文書終了**

*本文書は定期的にレビューされ、最新のベストプラクティスを反映するよう更新されます。*

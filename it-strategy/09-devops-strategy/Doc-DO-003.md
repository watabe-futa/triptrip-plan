# Doc-DO-003: TripTrip Deployment Strategies - 詳細プレイブック

## 文書メタデータ
| 項目 | 内容 |
|------|------|
| 文書ID | Doc-DO-003 |
| タイトル | Deployment Strategies - 詳細プレイブック |
| バージョン | 1.0.0 |
| 作成日 | 2026-01-20 |
| 最終更新日 | 2026-01-20 |
| 作成者 | Technical Architecture Team |
| 承認者 | CTO |
| 分類 | IT戦略 - DevOps戦略 |
| 関連文書 | Doc-QA-003, Doc-IA-002, Doc-DM-002, Doc-TR-002 |

---

## 変更履歴

| バージョン | 日付 | 変更者 | 変更内容 |
|------------|------|--------|----------|
| 1.0.0 | 2026-01-20 | Technical Architecture Team | 初版作成 |

---

## 目次

1. [エグゼクティブサマリー](#1-エグゼクティブサマリー)
2. [はじめに & コンテキスト](#2-はじめに--コンテキスト)
3. [Blue-Green Deployment 実装](#3-blue-green-deployment-実装)
4. [Canary Deployment 実装](#4-canary-deployment-実装)
5. [Progressive Delivery](#5-progressive-delivery)
6. [GitOps 実装](#6-gitops-実装)
7. [デプロイメントプレイブック](#7-デプロイメントプレイブック)
8. [実装ロードマップ](#8-実装ロードマップ)
9. [文書間参照 & 統合ポイント](#9-文書間参照--統合ポイント)

---

## 1. エグゼクティブサマリー

本文書は、TripTripプラットフォームにおけるデプロイメント戦略の**技術的詳細実装とプレイブック**を定義します。Doc-QA-003がリリース管理とロールバックの**プロセスレベル**を扱うのに対し、本文書はBlue-Green、Canary、Progressive Deliveryの**技術実装レベル**の詳細を提供します。

Google、Netflix、Amazonの大規模デプロイメントプラクティスを参考に、Kubernetes環境（AWS EKS）を基盤とした安全かつ効率的なデプロイメント手法を確立します。Argo Rollouts、Flagger、ArgoCD、LaunchDarklyなどの先進的なツールを活用し、ゼロダウンタイムデプロイメント、自動ロールバック、段階的リリースを実現します。

**主要成果物**:
- Blue-Green/Canaryの詳細Kubernetes設定
- Progressive Delivery with Feature Flags
- GitOpsベースのマルチ環境管理
- 運用プレイブック（通常リリース、緊急リリース、ロールバック）

---

## 2. はじめに & コンテキスト

### 2.1 デプロイメント戦略の目標

```yaml
deployment_strategy_goals:
  primary_objectives:
    - name: "ゼロダウンタイムデプロイメント"
      target: "計画外ダウンタイム0分"
      approach: "Rolling/Blue-Green/Canary"

    - name: "リスク最小化"
      target: "変更失敗率 < 5%"
      approach: "段階的ロールアウト、自動ロールバック"

    - name: "迅速なリリース"
      target: "デプロイ時間 < 15分"
      approach: "自動化、並列処理"

    - name: "可観測性の確保"
      target: "異常検知 < 2分"
      approach: "リアルタイムメトリクス、自動分析"

  kpis:
    deployment_frequency:
      current: "週3回"
      year_1_target: "日次"
      year_3_target: "オンデマンド（日10回以上）"

    lead_time:
      current: "2日"
      year_1_target: "1日"
      year_3_target: "1時間"

    change_failure_rate:
      current: "12%"
      year_1_target: "< 10%"
      year_3_target: "< 5%"

    mttr:
      current: "45分"
      year_1_target: "< 30分"
      year_3_target: "< 15分"
```

### 2.2 Doc-QA-003との関係性

```yaml
document_relationship:
  doc_qa_003:
    title: "リリース管理＆ロールバック手順"
    scope: "プロセスレベル"
    covers:
      - "リリーストレイン方式"
      - "リリースカレンダー"
      - "承認フロー"
      - "リリースチェックリスト"
      - "ロールバック判断基準"
      - "コミュニケーション計画"

  doc_do_003:
    title: "Deployment Strategies - 詳細プレイブック"
    scope: "技術レベル"
    covers:
      - "Kubernetes実装詳細"
      - "Argo Rollouts/Flagger設定"
      - "Feature Flags統合"
      - "GitOps構成"
      - "実行コマンドとスクリプト"
      - "トラブルシューティング"

  integration_points:
    - "Doc-QA-003の承認後、本文書の手順を実行"
    - "本文書の技術的判断基準はDoc-QA-003の判断フローに入力"
    - "ロールバック手順は両文書で連携"
```

### 2.3 現在のデプロイメント環境

```yaml
current_environment:
  infrastructure:
    platform: "AWS EKS 1.29"
    regions:
      primary: "ap-northeast-1 (Tokyo)"
      dr: "ap-northeast-3 (Osaka)"
    clusters:
      - name: "triptrip-production"
        purpose: "本番環境"
      - name: "triptrip-staging"
        purpose: "ステージング環境"
      - name: "triptrip-dev"
        purpose: "開発環境"

  application_stack:
    backend:
      runtime: "Node.js 18+"
      framework: "Hono"
      container_image: "node:18-alpine"
    frontend:
      framework: "Flutter 3.8.1+"
      deployment: "App Store / Google Play"

  deployment_tooling:
    ci_cd: "GitHub Actions"
    gitops: "ArgoCD"
    progressive_delivery: "Argo Rollouts"
    feature_flags: "LaunchDarkly"
    container_registry: "Amazon ECR"
```

---

## 3. Blue-Green Deployment 実装

### 3.1 アーキテクチャ設計

```yaml
blue_green_architecture:
  concept:
    description: |
      Blue-Green Deploymentは、2つの同一環境（Blue/Green）を維持し、
      トラフィックを瞬時に切り替えることでゼロダウンタイムを実現する手法。

  architecture_diagram: |
    ┌─────────────────────────────────────────────────────────────────┐
    │                       Load Balancer (ALB)                        │
    │                              │                                   │
    │                    ┌─────────┴─────────┐                        │
    │                    │   Istio Gateway   │                        │
    │                    └─────────┬─────────┘                        │
    │                              │                                   │
    │            ┌─────────────────┴─────────────────┐                │
    │            │         VirtualService            │                │
    │            │   (Traffic Routing Control)       │                │
    │            └─────────────────┬─────────────────┘                │
    │                              │                                   │
    │         ┌────────────────────┼────────────────────┐             │
    │         │                    │                    │             │
    │         ▼                    │                    ▼             │
    │  ┌──────────────┐            │           ┌──────────────┐       │
    │  │    BLUE      │  ◀─ Active Traffic    │    GREEN     │       │
    │  │  Deployment  │            │           │  Deployment  │       │
    │  │   (v1.5.2)   │            │           │   (v1.5.3)   │       │
    │  │              │            │           │              │       │
    │  │  ┌────────┐  │            │           │  ┌────────┐  │       │
    │  │  │ Pod x5 │  │            │           │  │ Pod x5 │  │       │
    │  │  └────────┘  │            │           │  └────────┘  │       │
    │  └──────────────┘            │           └──────────────┘       │
    │         │                    │                    │             │
    │         └────────────────────┼────────────────────┘             │
    │                              │                                   │
    │                    ┌─────────┴─────────┐                        │
    │                    │     Service       │                        │
    │                    │  (triptrip-api)   │                        │
    │                    └───────────────────┘                        │
    │                              │                                   │
    │                    ┌─────────┴─────────┐                        │
    │                    │    PostgreSQL     │                        │
    │                    │      (RDS)        │                        │
    │                    └───────────────────┘                        │
    └─────────────────────────────────────────────────────────────────┘

  benefits:
    - "瞬時のロールバック（トラフィック切り替えのみ）"
    - "本番環境での完全テスト"
    - "リリース前の検証が可能"

  considerations:
    - "リソースコストが2倍"
    - "データベースの互換性管理が必要"
    - "セッション管理の考慮"
```

### 3.2 Kubernetes実装詳細

```yaml
# Blue Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: triptrip-api-blue
  namespace: triptrip-production
  labels:
    app: triptrip-api
    version: blue
    release: v1.5.2
spec:
  replicas: 5
  selector:
    matchLabels:
      app: triptrip-api
      version: blue
  template:
    metadata:
      labels:
        app: triptrip-api
        version: blue
        release: v1.5.2
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
    spec:
      serviceAccountName: triptrip-api
      containers:
        - name: api
          image: 123456789.dkr.ecr.ap-northeast-1.amazonaws.com/triptrip-api:v1.5.2
          ports:
            - name: http
              containerPort: 3000
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2000m"
              memory: "2Gi"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 10
          env:
            - name: DEPLOYMENT_COLOR
              value: "blue"
            - name: VERSION
              value: "v1.5.2"
---
# Green Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: triptrip-api-green
  namespace: triptrip-production
  labels:
    app: triptrip-api
    version: green
    release: v1.5.3
spec:
  replicas: 5
  selector:
    matchLabels:
      app: triptrip-api
      version: green
  template:
    metadata:
      labels:
        app: triptrip-api
        version: green
        release: v1.5.3
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
    spec:
      serviceAccountName: triptrip-api
      containers:
        - name: api
          image: 123456789.dkr.ecr.ap-northeast-1.amazonaws.com/triptrip-api:v1.5.3
          ports:
            - name: http
              containerPort: 3000
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2000m"
              memory: "2Gi"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 10
          env:
            - name: DEPLOYMENT_COLOR
              value: "green"
            - name: VERSION
              value: "v1.5.3"
---
# Service (selector is managed by VirtualService)
apiVersion: v1
kind: Service
metadata:
  name: triptrip-api
  namespace: triptrip-production
spec:
  ports:
    - name: http
      port: 80
      targetPort: 3000
  selector:
    app: triptrip-api
---
# Blue Service
apiVersion: v1
kind: Service
metadata:
  name: triptrip-api-blue
  namespace: triptrip-production
spec:
  ports:
    - name: http
      port: 80
      targetPort: 3000
  selector:
    app: triptrip-api
    version: blue
---
# Green Service
apiVersion: v1
kind: Service
metadata:
  name: triptrip-api-green
  namespace: triptrip-production
spec:
  ports:
    - name: http
      port: 80
      targetPort: 3000
  selector:
    app: triptrip-api
    version: green
```

### 3.3 Istio VirtualService によるトラフィック切り替え

```yaml
# VirtualService for Blue-Green Switching
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: triptrip-api-blue-green
  namespace: triptrip-production
spec:
  hosts:
    - triptrip-api
    - api.triptrip.com
  gateways:
    - istio-system/triptrip-gateway
  http:
    # Active environment routing
    - match:
        - headers:
            x-environment:
              exact: "blue"
      route:
        - destination:
            host: triptrip-api-blue
            port:
              number: 80
    - match:
        - headers:
            x-environment:
              exact: "green"
      route:
        - destination:
            host: triptrip-api-green
            port:
              number: 80
    # Default routing (production traffic)
    - route:
        - destination:
            host: triptrip-api-blue  # ← Active environment
            port:
              number: 80
          weight: 100
        - destination:
            host: triptrip-api-green  # ← Standby environment
            port:
              number: 80
          weight: 0
---
# DestinationRule for connection management
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: triptrip-api-blue-green
  namespace: triptrip-production
spec:
  host: triptrip-api
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1000
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 1000
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
```

### 3.4 Blue-Green 切り替え自動化スクリプト

```bash
#!/bin/bash
# scripts/blue-green-switch.sh
# Blue-Green Deployment Switch Script

set -euo pipefail

# Configuration
NAMESPACE="triptrip-production"
VIRTUALSERVICE_NAME="triptrip-api-blue-green"
BLUE_SERVICE="triptrip-api-blue"
GREEN_SERVICE="triptrip-api-green"

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

# Get current active environment
get_active_environment() {
    kubectl get virtualservice ${VIRTUALSERVICE_NAME} -n ${NAMESPACE} \
        -o jsonpath='{.spec.http[2].route[0].destination.host}' | \
        sed 's/triptrip-api-//'
}

# Validate deployment health
validate_deployment() {
    local color=$1
    local deployment_name="triptrip-api-${color}"

    log_info "Validating deployment: ${deployment_name}"

    # Check deployment status
    local ready=$(kubectl get deployment ${deployment_name} -n ${NAMESPACE} \
        -o jsonpath='{.status.readyReplicas}')
    local desired=$(kubectl get deployment ${deployment_name} -n ${NAMESPACE} \
        -o jsonpath='{.spec.replicas}')

    if [[ "${ready}" != "${desired}" ]]; then
        log_error "Deployment ${deployment_name} not ready: ${ready}/${desired}"
        return 1
    fi

    # Check pod health
    local unhealthy=$(kubectl get pods -n ${NAMESPACE} \
        -l app=triptrip-api,version=${color} \
        --field-selector=status.phase!=Running \
        --no-headers 2>/dev/null | wc -l)

    if [[ "${unhealthy}" -gt 0 ]]; then
        log_error "Found ${unhealthy} unhealthy pods in ${deployment_name}"
        return 1
    fi

    log_info "Deployment ${deployment_name} is healthy"
    return 0
}

# Switch traffic to target environment
switch_traffic() {
    local target=$1
    local source=$2

    log_info "Switching traffic from ${source} to ${target}"

    # Create patch for VirtualService
    local patch=$(cat <<EOF
spec:
  http:
    - match:
        - headers:
            x-environment:
              exact: "blue"
      route:
        - destination:
            host: triptrip-api-blue
            port:
              number: 80
    - match:
        - headers:
            x-environment:
              exact: "green"
      route:
        - destination:
            host: triptrip-api-green
            port:
              number: 80
    - route:
        - destination:
            host: triptrip-api-${target}
            port:
              number: 80
          weight: 100
        - destination:
            host: triptrip-api-${source}
            port:
              number: 80
          weight: 0
EOF
)

    echo "${patch}" | kubectl patch virtualservice ${VIRTUALSERVICE_NAME} \
        -n ${NAMESPACE} --type=merge -p "$(echo "${patch}")"

    log_info "Traffic switched to ${target}"
}

# Run health checks after switch
post_switch_validation() {
    local target=$1
    local checks=5
    local interval=10

    log_info "Running post-switch validation (${checks} checks, ${interval}s interval)"

    for i in $(seq 1 ${checks}); do
        log_info "Health check ${i}/${checks}..."

        # Check error rate
        local error_rate=$(curl -s "http://prometheus:9090/api/v1/query" \
            --data-urlencode "query=sum(rate(http_requests_total{service=\"triptrip-api\",status=~\"5.*\"}[1m]))/sum(rate(http_requests_total{service=\"triptrip-api\"}[1m]))*100" \
            | jq -r '.data.result[0].value[1] // 0')

        if (( $(echo "${error_rate} > 5" | bc -l) )); then
            log_error "Error rate too high: ${error_rate}%"
            return 1
        fi

        # Check latency
        local p99_latency=$(curl -s "http://prometheus:9090/api/v1/query" \
            --data-urlencode "query=histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{service=\"triptrip-api\"}[1m]))*1000" \
            | jq -r '.data.result[0].value[1] // 0')

        if (( $(echo "${p99_latency} > 2000" | bc -l) )); then
            log_error "P99 latency too high: ${p99_latency}ms"
            return 1
        fi

        log_info "Check ${i} passed - Error rate: ${error_rate}%, P99: ${p99_latency}ms"
        sleep ${interval}
    done

    log_info "Post-switch validation completed successfully"
    return 0
}

# Main execution
main() {
    local action=${1:-"status"}

    case ${action} in
        status)
            local active=$(get_active_environment)
            log_info "Current active environment: ${active}"
            ;;
        switch)
            local current=$(get_active_environment)
            local target=$([[ "${current}" == "blue" ]] && echo "green" || echo "blue")

            log_info "Current: ${current}, Target: ${target}"

            # Validate target deployment
            if ! validate_deployment ${target}; then
                log_error "Target deployment validation failed. Aborting."
                exit 1
            fi

            # Switch traffic
            switch_traffic ${target} ${current}

            # Post-switch validation
            if ! post_switch_validation ${target}; then
                log_warn "Post-switch validation failed. Rolling back..."
                switch_traffic ${current} ${target}
                log_error "Rollback completed. Please investigate."
                exit 1
            fi

            log_info "Blue-Green switch completed successfully!"
            ;;
        rollback)
            local current=$(get_active_environment)
            local previous=$([[ "${current}" == "blue" ]] && echo "green" || echo "blue")

            log_warn "Rolling back from ${current} to ${previous}"
            switch_traffic ${previous} ${current}
            log_info "Rollback completed"
            ;;
        *)
            echo "Usage: $0 {status|switch|rollback}"
            exit 1
            ;;
    esac
}

main "$@"
```

### 3.5 データベースマイグレーション対応

```yaml
database_migration_strategy:
  approach: "Expand and Contract Pattern"

  phases:
    phase_1_expand:
      description: "新しいスキーマを追加（後方互換性あり）"
      actions:
        - "新カラムをNULLableで追加"
        - "新テーブルを作成"
        - "新インデックスを作成"
      deployment: "Blue環境で実行"
      rollback: "DROP/ALTERで元に戻す"

    phase_2_migrate:
      description: "データを新スキーマに移行"
      actions:
        - "バックグラウンドジョブでデータ移行"
        - "デュアルライト（新旧両方に書き込み）"
      deployment: "Blue/Green両方で対応コード"
      rollback: "データ移行を停止"

    phase_3_contract:
      description: "旧スキーマを削除"
      actions:
        - "旧カラムの使用を停止"
        - "旧カラムを削除"
      deployment: "全環境で旧スキーマ不使用を確認後"
      rollback: "削除前なら即時、削除後はバックアップから復元"

  migration_script_example: |
    -- Phase 1: Expand
    BEGIN;

    -- Add new column
    ALTER TABLE users ADD COLUMN email_verified_at TIMESTAMP;

    -- Create index concurrently (non-blocking)
    CREATE INDEX CONCURRENTLY idx_users_email_verified
    ON users (email_verified_at)
    WHERE email_verified_at IS NOT NULL;

    COMMIT;

    -- Phase 2: Migrate (run in batches)
    UPDATE users
    SET email_verified_at = CASE
        WHEN email_verified = true THEN updated_at
        ELSE NULL
    END
    WHERE email_verified_at IS NULL
    AND id BETWEEN $1 AND $2;

    -- Phase 3: Contract (after validation)
    ALTER TABLE users DROP COLUMN email_verified;

  dual_write_implementation: |
    // TypeScript implementation for dual-write
    async function updateUser(userId: string, data: UserUpdateData) {
      const user = await prisma.user.update({
        where: { id: userId },
        data: {
          ...data,
          // Legacy column (Phase 2)
          email_verified: data.emailVerifiedAt ? true : false,
          // New column
          email_verified_at: data.emailVerifiedAt,
        },
      });
      return user;
    }
```

---

## 4. Canary Deployment 実装

### 4.1 Argo Rollouts 設定

```yaml
# Argo Rollouts for Canary Deployment
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: triptrip-api
  namespace: triptrip-production
spec:
  replicas: 10
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: triptrip-api
  template:
    metadata:
      labels:
        app: triptrip-api
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
    spec:
      serviceAccountName: triptrip-api
      containers:
        - name: api
          image: 123456789.dkr.ecr.ap-northeast-1.amazonaws.com/triptrip-api:v1.5.3
          ports:
            - name: http
              containerPort: 3000
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2000m"
              memory: "2Gi"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 10
          envFrom:
            - configMapRef:
                name: triptrip-api-config
            - secretRef:
                name: triptrip-api-secrets
  strategy:
    canary:
      # Canary service for testing
      canaryService: triptrip-api-canary
      stableService: triptrip-api-stable

      # Traffic routing via Istio
      trafficRouting:
        istio:
          virtualServices:
            - name: triptrip-api
              routes:
                - primary
          destinationRule:
            name: triptrip-api
            canarySubsetName: canary
            stableSubsetName: stable

      # Canary steps: 1% → 5% → 25% → 50% → 100%
      steps:
        # Step 1: 1% traffic
        - setWeight: 1
        - pause:
            duration: 2m
        - analysis:
            templates:
              - templateName: canary-success-rate
            args:
              - name: service-name
                value: triptrip-api-canary

        # Step 2: 5% traffic
        - setWeight: 5
        - pause:
            duration: 5m
        - analysis:
            templates:
              - templateName: canary-success-rate
              - templateName: canary-latency

        # Step 3: 25% traffic
        - setWeight: 25
        - pause:
            duration: 10m
        - analysis:
            templates:
              - templateName: canary-success-rate
              - templateName: canary-latency
              - templateName: canary-error-budget

        # Step 4: 50% traffic
        - setWeight: 50
        - pause:
            duration: 15m
        - analysis:
            templates:
              - templateName: canary-success-rate
              - templateName: canary-latency
              - templateName: canary-error-budget

        # Step 5: 100% traffic
        - setWeight: 100
        - pause:
            duration: 5m

      # Anti-affinity rules
      antiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution: {}
        preferredDuringSchedulingIgnoredDuringExecution:
          weight: 100

      # Rollback conditions
      maxSurge: "25%"
      maxUnavailable: 0

      # Analysis settings
      analysis:
        successfulRunHistoryLimit: 3
        unsuccessfulRunHistoryLimit: 3
```

### 4.2 メトリクス分析テンプレート

```yaml
# Success Rate Analysis Template
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: canary-success-rate
  namespace: triptrip-production
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.995  # 99.5% success rate
      failureLimit: 3
      provider:
        datadog:
          apiVersion: v1
          query: |
            sum:triptrip.http.request.count{service:{{args.service-name}},status_class:2xx}.as_rate() /
            sum:triptrip.http.request.count{service:{{args.service-name}}}.as_rate()
---
# Latency Analysis Template
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: canary-latency
  namespace: triptrip-production
spec:
  args:
    - name: service-name
  metrics:
    - name: p99-latency
      interval: 1m
      successCondition: result[0] < 500  # P99 < 500ms
      failureLimit: 3
      provider:
        datadog:
          query: |
            avg:triptrip.http.request.duration.99percentile{service:{{args.service-name}}}

    - name: p50-latency
      interval: 1m
      successCondition: result[0] < 100  # P50 < 100ms
      failureLimit: 5
      provider:
        datadog:
          query: |
            avg:triptrip.http.request.duration.median{service:{{args.service-name}}}
---
# Error Budget Analysis Template
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: canary-error-budget
  namespace: triptrip-production
spec:
  args:
    - name: service-name
  metrics:
    - name: error-budget-remaining
      interval: 5m
      successCondition: result[0] > 0.1  # At least 10% error budget remaining
      failureLimit: 1
      provider:
        datadog:
          query: |
            1 - (
              sum:triptrip.http.request.count{service:{{args.service-name}},status_class:5xx}.as_count().rollup(sum, 86400) /
              sum:triptrip.http.request.count{service:{{args.service-name}}}.as_count().rollup(sum, 86400)
            ) / 0.001
---
# Comparison Analysis (Canary vs Stable)
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: canary-comparison
  namespace: triptrip-production
spec:
  args:
    - name: canary-service
    - name: stable-service
  metrics:
    - name: error-rate-comparison
      interval: 2m
      successCondition: result[0] <= result[1] * 1.1  # Canary error rate <= Stable * 1.1
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            (
              sum(rate(http_requests_total{service="{{args.canary-service}}",status=~"5.."}[5m])) /
              sum(rate(http_requests_total{service="{{args.canary-service}}"}[5m]))
            )
            or vector(0)

    - name: latency-comparison
      interval: 2m
      successCondition: result[0] <= result[1] * 1.2  # Canary latency <= Stable * 1.2
      failureLimit: 3
      provider:
        prometheus:
          query: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{service="{{args.canary-service}}"}[5m])) by (le)
            )
```

### 4.3 異常検知と自動ロールバック

```yaml
# Automatic Rollback Configuration
automatic_rollback:
  triggers:
    error_rate_spike:
      condition: "error_rate > 5%"
      duration: "2 minutes"
      action: "abort_rollout"

    latency_degradation:
      condition: "p99_latency > 2000ms"
      duration: "3 minutes"
      action: "abort_rollout"

    pod_crash_loop:
      condition: "pod_restarts > 3"
      duration: "5 minutes"
      action: "abort_rollout"

    health_check_failure:
      condition: "health_check_failures > 5"
      duration: "1 minute"
      action: "abort_rollout"

  rollback_behavior:
    abort_scale_down_delay: 30  # seconds
    analysis_runs_history_limit: 5
    progress_deadline_seconds: 600

# Rollback webhook notification
rollback_notifications:
  webhook:
    url: "https://hooks.slack.com/services/xxx/yyy/zzz"
    template: |
      {
        "channel": "#alerts-deployments",
        "attachments": [{
          "color": "danger",
          "title": "Canary Rollback Triggered",
          "fields": [
            {"title": "Rollout", "value": "{{.rollout.name}}", "short": true},
            {"title": "Namespace", "value": "{{.rollout.namespace}}", "short": true},
            {"title": "Reason", "value": "{{.analysis.message}}", "short": false},
            {"title": "Failed Metric", "value": "{{.analysis.failedMetric}}", "short": false}
          ],
          "footer": "TripTrip Deployment System",
          "ts": {{.timestamp}}
        }]
      }
```

### 4.4 Canary 操作コマンド

```bash
#!/bin/bash
# scripts/canary-operations.sh
# Canary Deployment Operations

set -euo pipefail

NAMESPACE="triptrip-production"
ROLLOUT_NAME="triptrip-api"

# Get rollout status
status() {
    echo "=== Rollout Status ==="
    kubectl argo rollouts get rollout ${ROLLOUT_NAME} -n ${NAMESPACE}

    echo ""
    echo "=== Canary Weight ==="
    kubectl argo rollouts get rollout ${ROLLOUT_NAME} -n ${NAMESPACE} \
        -o jsonpath='{.status.canary.weights}' | jq .

    echo ""
    echo "=== Analysis Runs ==="
    kubectl get analysisruns -n ${NAMESPACE} \
        -l rollouts-pod-template-hash \
        --sort-by='.metadata.creationTimestamp' \
        | tail -5
}

# Promote canary to next step
promote() {
    echo "Promoting canary to next step..."
    kubectl argo rollouts promote ${ROLLOUT_NAME} -n ${NAMESPACE}
    echo "Promoted. Current status:"
    status
}

# Promote canary to 100% (skip remaining steps)
promote_full() {
    echo "Promoting canary to 100%..."
    kubectl argo rollouts promote ${ROLLOUT_NAME} -n ${NAMESPACE} --full
    echo "Full promotion initiated."
    status
}

# Abort rollout and rollback
abort() {
    echo "Aborting rollout..."
    kubectl argo rollouts abort ${ROLLOUT_NAME} -n ${NAMESPACE}
    echo "Rollout aborted. Rolling back..."
    status
}

# Retry failed rollout
retry() {
    echo "Retrying rollout..."
    kubectl argo rollouts retry rollout ${ROLLOUT_NAME} -n ${NAMESPACE}
    status
}

# Set specific canary weight
set_weight() {
    local weight=${1:-10}
    echo "Setting canary weight to ${weight}%..."
    kubectl argo rollouts set weight ${ROLLOUT_NAME} -n ${NAMESPACE} ${weight}
    status
}

# Pause rollout
pause() {
    echo "Pausing rollout..."
    kubectl argo rollouts pause ${ROLLOUT_NAME} -n ${NAMESPACE}
    status
}

# Resume rollout
resume() {
    echo "Resuming rollout..."
    kubectl argo rollouts resume ${ROLLOUT_NAME} -n ${NAMESPACE}
    status
}

# Watch rollout progress
watch() {
    kubectl argo rollouts get rollout ${ROLLOUT_NAME} -n ${NAMESPACE} --watch
}

# Main
case ${1:-status} in
    status)     status ;;
    promote)    promote ;;
    promote-full) promote_full ;;
    abort)      abort ;;
    retry)      retry ;;
    set-weight) set_weight ${2:-10} ;;
    pause)      pause ;;
    resume)     resume ;;
    watch)      watch ;;
    *)
        echo "Usage: $0 {status|promote|promote-full|abort|retry|set-weight N|pause|resume|watch}"
        exit 1
        ;;
esac
```

---

## 5. Progressive Delivery

### 5.1 Feature Flags 統合 (LaunchDarkly)

```typescript
// src/infrastructure/feature-flags/launchdarkly-config.ts
import * as LaunchDarkly from 'launchdarkly-node-server-sdk';
import { Logger } from '../logging/logger';

// LaunchDarkly Configuration
export const ldConfig: LaunchDarkly.LDOptions = {
  logger: LaunchDarkly.basicLogger({ level: 'info' }),
  sendEvents: true,
  allAttributesPrivate: false,
  diagnosticOptOut: false,
  offline: process.env.NODE_ENV === 'test',
};

// Feature Flag Keys
export const FeatureFlags = {
  // Release Flags
  NEW_BOOKING_FLOW: 'new-booking-flow',
  AI_TRIP_SUGGESTIONS: 'ai-trip-suggestions',
  REAL_TIME_PRICING: 'real-time-pricing',
  IMPROVED_SEARCH: 'improved-search',

  // Ops Flags
  PAYMENT_GATEWAY_ENABLED: 'payment-gateway-enabled',
  EXTERNAL_MAP_API_ENABLED: 'external-map-api-enabled',
  RATE_LIMITING_STRICT: 'rate-limiting-strict',
  MAINTENANCE_MODE: 'maintenance-mode',

  // Experiment Flags
  SEARCH_ALGORITHM_VARIANT: 'search-algorithm-variant',
  CHECKOUT_UI_VARIANT: 'checkout-ui-variant',
  PRICING_MODEL_VARIANT: 'pricing-model-variant',

  // Permission Flags
  PREMIUM_FEATURES: 'premium-features',
  BETA_TESTER_FEATURES: 'beta-tester-features',
  STAFF_FEATURES: 'staff-features',
} as const;

// User Context Builder
export interface UserContext {
  key: string;
  email?: string;
  name?: string;
  ip?: string;
  country?: string;
  custom?: {
    plan?: string;
    accountAge?: number;
    deviceType?: string;
    appVersion?: string;
    region?: string;
  };
}

export function buildLDContext(user: UserContext): LaunchDarkly.LDContext {
  return {
    kind: 'user',
    key: user.key,
    email: user.email,
    name: user.name,
    ip: user.ip,
    country: user.country,
    ...user.custom,
  };
}

// Feature Flag Service
export class FeatureFlagService {
  private client: LaunchDarkly.LDClient;
  private logger: Logger;
  private initialized: boolean = false;

  constructor() {
    this.logger = new Logger('FeatureFlagService');
    this.client = LaunchDarkly.init(
      process.env.LAUNCHDARKLY_SDK_KEY!,
      ldConfig
    );
  }

  async initialize(): Promise<void> {
    try {
      await this.client.waitForInitialization();
      this.initialized = true;
      this.logger.info('LaunchDarkly client initialized');
    } catch (error) {
      this.logger.error('Failed to initialize LaunchDarkly', { error });
      throw error;
    }
  }

  async isEnabled(
    flag: string,
    user: UserContext,
    defaultValue: boolean = false
  ): Promise<boolean> {
    if (!this.initialized) {
      this.logger.warn('LaunchDarkly not initialized, returning default');
      return defaultValue;
    }

    try {
      const context = buildLDContext(user);
      return await this.client.variation(flag, context, defaultValue);
    } catch (error) {
      this.logger.error('Error evaluating feature flag', { flag, error });
      return defaultValue;
    }
  }

  async getVariation<T>(
    flag: string,
    user: UserContext,
    defaultValue: T
  ): Promise<T> {
    if (!this.initialized) {
      return defaultValue;
    }

    try {
      const context = buildLDContext(user);
      return await this.client.variation(flag, context, defaultValue);
    } catch (error) {
      this.logger.error('Error getting variation', { flag, error });
      return defaultValue;
    }
  }

  async getVariationDetail<T>(
    flag: string,
    user: UserContext,
    defaultValue: T
  ): Promise<LaunchDarkly.LDEvaluationDetail<T>> {
    if (!this.initialized) {
      return {
        value: defaultValue,
        variationIndex: null,
        reason: { kind: 'ERROR', errorKind: 'CLIENT_NOT_READY' },
      };
    }

    const context = buildLDContext(user);
    return await this.client.variationDetail(flag, context, defaultValue);
  }

  async track(
    eventName: string,
    user: UserContext,
    data?: Record<string, unknown>,
    metricValue?: number
  ): Promise<void> {
    if (!this.initialized) return;

    const context = buildLDContext(user);
    this.client.track(eventName, context, data, metricValue);
  }

  async flush(): Promise<void> {
    await this.client.flush();
  }

  async close(): Promise<void> {
    await this.client.close();
    this.initialized = false;
  }
}

export const featureFlags = new FeatureFlagService();
```

### 5.2 段階的ロールアウト戦略

```yaml
progressive_rollout_strategy:
  deployment_ring_model:
    ring_0_internal:
      description: "社内テスター"
      percentage: "100% of internal users"
      duration: "1-2 days"
      criteria:
        targeting:
          - "email ends with @triptrip.com"
          - "user attribute: internal = true"
        validation:
          - "Manual smoke testing"
          - "Core functionality verification"

    ring_1_beta:
      description: "ベータテスター"
      percentage: "100% of beta users"
      duration: "3-5 days"
      criteria:
        targeting:
          - "user attribute: betaTester = true"
          - "opt-in users"
        validation:
          - "Error rate < 1%"
          - "User feedback positive"
          - "No critical bugs"

    ring_2_early_adopters:
      description: "アーリーアダプター"
      percentage: "5% of production users"
      duration: "3-7 days"
      criteria:
        targeting:
          - "Random 5% of users"
          - "Premium tier users (optional)"
        validation:
          - "Error rate < 0.5%"
          - "P99 latency within baseline"
          - "Business metrics stable"

    ring_3_general:
      description: "段階的全ユーザー展開"
      stages:
        - percentage: "25%"
          duration: "1-2 days"
        - percentage: "50%"
          duration: "1-2 days"
        - percentage: "75%"
          duration: "1 day"
        - percentage: "100%"
          duration: "permanent"
      criteria:
        targeting:
          - "Gradual percentage increase"
        validation:
          - "All metrics within threshold"
          - "No user complaints"
```

### 5.3 地理的段階デプロイ

```yaml
geographic_rollout:
  strategy: "Region by Region"

  deployment_sequence:
    phase_1_japan:
      regions:
        - "ap-northeast-1 (Tokyo)"
      traffic_percentage: "100% of Japan traffic"
      duration: "1 week"
      rationale: "Primary market, closest monitoring"
      validation:
        - "Local performance metrics"
        - "Japan-specific features working"
        - "Customer support feedback"

    phase_2_asia:
      regions:
        - "ap-southeast-1 (Singapore)"
        - "ap-northeast-2 (Seoul)"
      traffic_percentage: "100% of APAC traffic"
      duration: "1 week"
      rationale: "Similar timezone, cultural alignment"
      validation:
        - "Multi-region latency"
        - "Currency/language handling"

    phase_3_global:
      regions:
        - "us-west-2 (Oregon)"
        - "eu-west-1 (Ireland)"
      traffic_percentage: "100% global"
      duration: "Permanent"
      rationale: "Full global coverage"
      validation:
        - "Global latency metrics"
        - "All localization working"

  kubernetes_implementation: |
    # Using Argo Rollouts with multi-region ApplicationSet
    apiVersion: argoproj.io/v1alpha1
    kind: ApplicationSet
    metadata:
      name: triptrip-api-geo-rollout
      namespace: argocd
    spec:
      generators:
        - list:
            elements:
              - region: ap-northeast-1
                cluster: eks-tokyo
                phase: 1
              - region: ap-southeast-1
                cluster: eks-singapore
                phase: 2
              - region: us-west-2
                cluster: eks-oregon
                phase: 3
      template:
        metadata:
          name: 'triptrip-api-{{region}}'
        spec:
          project: triptrip
          source:
            repoURL: https://github.com/triptrip/k8s-manifests
            targetRevision: HEAD
            path: charts/triptrip-api
            helm:
              valueFiles:
                - values.yaml
                - 'values-{{region}}.yaml'
          destination:
            server: 'https://{{cluster}}.eks.amazonaws.com'
            namespace: triptrip-production
```

---

## 6. GitOps 実装

### 6.1 ArgoCD アーキテクチャ

```yaml
argocd_architecture:
  overview: |
    ┌─────────────────────────────────────────────────────────────────┐
    │                      ArgoCD Architecture                         │
    ├─────────────────────────────────────────────────────────────────┤
    │                                                                  │
    │  ┌──────────────┐        ┌──────────────────────────────┐      │
    │  │   GitHub     │        │         ArgoCD Server        │      │
    │  │ Repository   │◄──────►│                              │      │
    │  │              │  Poll  │  ┌────────────────────────┐  │      │
    │  │ k8s-manifests│        │  │    Application         │  │      │
    │  │              │        │  │    Controller          │  │      │
    │  └──────────────┘        │  └────────────────────────┘  │      │
    │         │                │              │               │      │
    │         │                │              ▼               │      │
    │         ▼                │  ┌────────────────────────┐  │      │
    │  ┌──────────────┐        │  │    Repo Server         │  │      │
    │  │  Helm Charts │        │  │  (Manifest Generation) │  │      │
    │  │  Kustomize   │        │  └────────────────────────┘  │      │
    │  │  Plain YAML  │        │              │               │      │
    │  └──────────────┘        └──────────────┼───────────────┘      │
    │                                         │                       │
    │         ┌───────────────────────────────┘                       │
    │         ▼                                                       │
    │  ┌─────────────────────────────────────────────────────────┐   │
    │  │                    Kubernetes Clusters                   │   │
    │  │                                                          │   │
    │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │   │
    │  │  │ Development  │  │   Staging    │  │  Production  │   │   │
    │  │  │   Cluster    │  │   Cluster    │  │   Cluster    │   │   │
    │  │  └──────────────┘  └──────────────┘  └──────────────┘   │   │
    │  └─────────────────────────────────────────────────────────┘   │
    └─────────────────────────────────────────────────────────────────┘

  components:
    application_controller:
      purpose: "アプリケーション状態の監視と同期"
      replicas: 3
      resources:
        requests:
          cpu: "500m"
          memory: "512Mi"
        limits:
          cpu: "2"
          memory: "2Gi"

    repo_server:
      purpose: "Git リポジトリからのマニフェスト生成"
      replicas: 3
      resources:
        requests:
          cpu: "250m"
          memory: "256Mi"
        limits:
          cpu: "1"
          memory: "1Gi"

    api_server:
      purpose: "Web UI と CLI の API"
      replicas: 2

    redis:
      purpose: "キャッシュとセッション管理"
      mode: "HA with Sentinel"
```

### 6.2 マルチ環境管理

```yaml
# ArgoCD Application for Production
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: triptrip-api-production
  namespace: argocd
  labels:
    app: triptrip-api
    env: production
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: triptrip-production

  source:
    repoURL: https://github.com/triptrip/k8s-manifests.git
    targetRevision: main
    path: apps/triptrip-api
    helm:
      releaseName: triptrip-api
      valueFiles:
        - values.yaml
        - values-production.yaml
      parameters:
        - name: image.tag
          value: v1.5.3

  destination:
    server: https://eks-production.triptrip.internal
    namespace: triptrip-production

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  revisionHistoryLimit: 10

  ignoreDifferences:
    - group: autoscaling
      kind: HorizontalPodAutoscaler
      jsonPointers:
        - /spec/minReplicas
        - /spec/maxReplicas
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas

---
# ApplicationSet for All Environments
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: triptrip-api
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: development
            cluster: https://eks-dev.triptrip.internal
            targetRevision: develop
            syncPolicy: automated

          - env: staging
            cluster: https://eks-staging.triptrip.internal
            targetRevision: release/*
            syncPolicy: automated

          - env: production
            cluster: https://eks-production.triptrip.internal
            targetRevision: main
            syncPolicy: manual  # Production requires manual approval

  template:
    metadata:
      name: 'triptrip-api-{{env}}'
      namespace: argocd
      labels:
        app: triptrip-api
        env: '{{env}}'
    spec:
      project: 'triptrip-{{env}}'
      source:
        repoURL: https://github.com/triptrip/k8s-manifests.git
        targetRevision: '{{targetRevision}}'
        path: apps/triptrip-api
        helm:
          valueFiles:
            - values.yaml
            - 'values-{{env}}.yaml'
      destination:
        server: '{{cluster}}'
        namespace: 'triptrip-{{env}}'
      syncPolicy:
        automated:
          prune: '{{syncPolicy}}' == 'automated'
          selfHeal: '{{syncPolicy}}' == 'automated'
```

### 6.3 マルチクラスタ同期

```yaml
# ArgoCD Project for Production
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: triptrip-production
  namespace: argocd
spec:
  description: "TripTrip Production Environment"

  sourceRepos:
    - https://github.com/triptrip/k8s-manifests.git
    - https://github.com/triptrip/helm-charts.git

  destinations:
    - namespace: triptrip-production
      server: https://eks-production.triptrip.internal
    - namespace: triptrip-production
      server: https://eks-dr.triptrip.internal

  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
    - group: networking.k8s.io
      kind: NetworkPolicy
    - group: rbac.authorization.k8s.io
      kind: ClusterRole
    - group: rbac.authorization.k8s.io
      kind: ClusterRoleBinding

  namespaceResourceWhitelist:
    - group: ''
      kind: '*'
    - group: apps
      kind: '*'
    - group: networking.istio.io
      kind: '*'
    - group: argoproj.io
      kind: '*'

  roles:
    - name: admin
      description: "Project admin"
      policies:
        - p, proj:triptrip-production:admin, applications, *, triptrip-production/*, allow
        - p, proj:triptrip-production:admin, clusters, get, triptrip-production/*, allow
      groups:
        - platform-admins

    - name: developer
      description: "Developers can sync but not delete"
      policies:
        - p, proj:triptrip-production:developer, applications, get, triptrip-production/*, allow
        - p, proj:triptrip-production:developer, applications, sync, triptrip-production/*, allow
      groups:
        - developers

  syncWindows:
    - kind: allow
      schedule: '0 10 * * 2'  # Tuesday 10:00
      duration: 4h
      applications:
        - '*'
      manualSync: true

    - kind: deny
      schedule: '0 0 28-31 12 *'  # Year-end
      duration: 168h
      applications:
        - '*'

    - kind: deny
      schedule: '0 0 29 4-5 *'  # Golden Week
      duration: 168h
      applications:
        - '*'
```

---

## 7. デプロイメントプレイブック

### 7.1 標準リリース手順

```yaml
standard_release_playbook:
  pre_release:
    - step: 1
      name: "リリース準備確認"
      duration: "1 hour before"
      actions:
        - "Doc-QA-003のリリースチェックリスト確認"
        - "全テストがパスしていることを確認"
        - "ステージング環境での検証完了確認"
        - "オンコールチームへの通知"
      checklist:
        - "[ ] 全CIジョブがグリーン"
        - "[ ] セキュリティスキャン完了"
        - "[ ] PRレビュー完了"
        - "[ ] CHANGELOGが更新済み"

    - step: 2
      name: "環境確認"
      duration: "30 minutes before"
      actions:
        - "本番クラスタの健全性確認"
        - "メトリクスベースライン記録"
        - "ロールバック手順の確認"
      commands:
        - "kubectl get nodes -o wide"
        - "kubectl top nodes"
        - "kubectl get pods -n triptrip-production"
        - "./scripts/record-baseline-metrics.sh"

  release_execution:
    - step: 3
      name: "デプロイメント開始"
      duration: "~15 minutes"
      actions:
        - "ArgoCD経由でデプロイ開始"
        - "Slack通知送信"
      commands:
        - |
          # Update image tag in Git repository
          cd k8s-manifests
          yq e '.image.tag = "v1.5.3"' -i apps/triptrip-api/values-production.yaml
          git add . && git commit -m "Release triptrip-api v1.5.3"
          git push origin main

        - |
          # Trigger sync (if manual)
          argocd app sync triptrip-api-production

        - |
          # Watch rollout
          kubectl argo rollouts get rollout triptrip-api -n triptrip-production --watch

    - step: 4
      name: "Canary フェーズ監視"
      duration: "~30 minutes"
      actions:
        - "各Canaryステップのメトリクス監視"
        - "エラー率、レイテンシの確認"
        - "アラートの監視"
      monitoring:
        dashboards:
          - "Datadog: TripTrip API Overview"
          - "Grafana: Deployment Metrics"
        thresholds:
          error_rate: "< 0.5%"
          p99_latency: "< 500ms"
          success_rate: "> 99.5%"
      commands:
        - "./scripts/canary-operations.sh status"
        - "kubectl argo rollouts get rollout triptrip-api -n triptrip-production"

  post_release:
    - step: 5
      name: "リリース完了確認"
      duration: "15 minutes after"
      actions:
        - "100%トラフィック移行確認"
        - "メトリクス正常性確認"
        - "ログのエラー確認"
      commands:
        - |
          # Verify all pods are running new version
          kubectl get pods -n triptrip-production -l app=triptrip-api \
            -o jsonpath='{.items[*].spec.containers[*].image}'

        - "./scripts/compare-baseline-metrics.sh"

    - step: 6
      name: "リリース通知"
      duration: "Immediately after"
      actions:
        - "Slack #releases チャンネルに完了通知"
        - "GitHub Release 作成"
        - "ステータスページ更新（該当する場合）"
      notification_template: |
        :white_check_mark: **Release Completed**
        - Service: triptrip-api
        - Version: v1.5.3
        - Duration: 45 minutes
        - Error Rate: 0.02%
        - P99 Latency: 180ms

    - step: 7
      name: "監視強化期間"
      duration: "2 hours"
      actions:
        - "重点監視継続"
        - "ユーザーフィードバック確認"
        - "ロールバック準備維持"
```

### 7.2 緊急リリース手順

```yaml
hotfix_release_playbook:
  trigger_criteria:
    - "P1/P2 インシデント対応"
    - "セキュリティ脆弱性修正"
    - "重大なバグ修正"

  approval_requirements:
    - "Tech Lead承認必須"
    - "Security Team承認（セキュリティ関連の場合）"
    - "通常PRレビューの簡略化可能（1名でOK）"

  execution:
    - step: 1
      name: "Hotfixブランチ作成"
      duration: "5 minutes"
      commands:
        - |
          git checkout main
          git pull origin main
          git checkout -b hotfix/ISSUE-123-critical-fix

    - step: 2
      name: "修正実装＆テスト"
      duration: "Variable"
      requirements:
        - "最小限の変更に限定"
        - "影響範囲を明確に特定"
        - "単体テストを追加"
      commands:
        - |
          # Implement fix
          # Run tests
          npm test
          npm run lint

    - step: 3
      name: "緊急PRレビュー"
      duration: "15 minutes"
      requirements:
        - "Tech Lead必須レビュー"
        - "変更内容のダブルチェック"

    - step: 4
      name: "Stagingデプロイ＆スモークテスト"
      duration: "15 minutes"
      commands:
        - |
          # Build and push
          docker build -t triptrip-api:hotfix .
          docker push 123456789.dkr.ecr.ap-northeast-1.amazonaws.com/triptrip-api:hotfix

          # Deploy to staging
          kubectl set image deployment/triptrip-api \
            api=123456789.dkr.ecr.ap-northeast-1.amazonaws.com/triptrip-api:hotfix \
            -n triptrip-staging

          # Smoke test
          ./scripts/smoke-test.sh https://staging-api.triptrip.com

    - step: 5
      name: "本番デプロイ（Canaryスキップ可）"
      duration: "10 minutes"
      options:
        canary_skip:
          conditions:
            - "影響範囲が限定的"
            - "修正内容がシンプル"
            - "Tech Lead承認済み"
          command: |
            # Direct rolling update
            kubectl set image deployment/triptrip-api \
              api=123456789.dkr.ecr.ap-northeast-1.amazonaws.com/triptrip-api:hotfix \
              -n triptrip-production

        canary_fast:
          description: "短縮Canary（5分間）"
          command: |
            # Fast canary with 5 minute pause
            kubectl argo rollouts set image triptrip-api \
              api=123456789.dkr.ecr.ap-northeast-1.amazonaws.com/triptrip-api:hotfix \
              -n triptrip-production

    - step: 6
      name: "監視強化"
      duration: "30 minutes"
      actions:
        - "エラー率の重点監視"
        - "修正対象の機能テスト"
        - "ロールバック即時実行可能状態維持"

    - step: 7
      name: "mainブランチへのマージ"
      duration: "5 minutes"
      commands:
        - |
          git checkout main
          git merge hotfix/ISSUE-123-critical-fix
          git push origin main
          git branch -d hotfix/ISSUE-123-critical-fix
```

### 7.3 ロールバック手順

```yaml
rollback_playbook:
  immediate_rollback:
    trigger: "クリティカルな問題発見"
    timeline: "5分以内"

    steps:
      - step: 1
        name: "ロールバック判断"
        duration: "1 minute"
        criteria:
          auto_rollback:
            - "エラー率 > 5%"
            - "P99レイテンシ > 2000ms"
            - "ヘルスチェック失敗 > 50%"
          manual_rollback:
            - "ユーザー報告の急増"
            - "ビジネスメトリクスの異常"
            - "データ不整合の検出"

      - step: 2
        name: "Slackアラート発報"
        duration: "Immediate"
        command: |
          # Automatic notification
          curl -X POST https://hooks.slack.com/services/xxx/yyy/zzz \
            -H 'Content-type: application/json' \
            -d '{
              "channel": "#alerts-critical",
              "text": ":rotating_light: ROLLBACK INITIATED - triptrip-api",
              "attachments": [{
                "color": "danger",
                "fields": [
                  {"title": "Reason", "value": "Error rate exceeded 5%"},
                  {"title": "Current Version", "value": "v1.5.3"},
                  {"title": "Rollback To", "value": "v1.5.2"}
                ]
              }]
            }'

      - step: 3
        name: "ロールバック実行"
        duration: "2-3 minutes"
        options:
          argo_rollouts:
            command: |
              # Abort current rollout
              kubectl argo rollouts abort triptrip-api -n triptrip-production

              # Undo to previous version
              kubectl argo rollouts undo triptrip-api -n triptrip-production

              # Verify rollback
              kubectl argo rollouts get rollout triptrip-api -n triptrip-production

          kubernetes_rollout:
            command: |
              # Standard Kubernetes rollback
              kubectl rollout undo deployment/triptrip-api -n triptrip-production

              # Verify rollback
              kubectl rollout status deployment/triptrip-api -n triptrip-production

          blue_green:
            command: |
              # Switch traffic to previous environment
              ./scripts/blue-green-switch.sh rollback

          feature_flag:
            command: |
              # Disable feature via LaunchDarkly
              curl -X PATCH "https://app.launchdarkly.com/api/v2/flags/triptrip/${FLAG_KEY}" \
                -H "Authorization: ${LD_API_KEY}" \
                -H "Content-Type: application/json" \
                -d '[{
                  "op": "replace",
                  "path": "/environments/production/on",
                  "value": false
                }]'

      - step: 4
        name: "復旧確認"
        duration: "5 minutes"
        checklist:
          - "[ ] エラー率が正常値（< 0.1%）に回復"
          - "[ ] レイテンシが正常値（P99 < 500ms）に回復"
          - "[ ] ヘルスチェックが全てパス"
          - "[ ] ユーザートラフィックが正常に処理されている"
        commands:
          - |
            # Verify metrics
            ./scripts/check-metrics.sh

            # Check pod status
            kubectl get pods -n triptrip-production -l app=triptrip-api

            # Check logs for errors
            kubectl logs -n triptrip-production -l app=triptrip-api --tail=100 | grep -i error

      - step: 5
        name: "事後対応"
        duration: "Within 24 hours"
        actions:
          - "インシデントレポート作成"
          - "根本原因分析"
          - "修正PRの作成"
          - "ポストモーテムのスケジュール（SEV-1/2の場合）"
```

### 7.4 データベースマイグレーションプレイブック

```yaml
database_migration_playbook:
  pre_migration:
    - step: 1
      name: "マイグレーション計画確認"
      actions:
        - "スキーマ変更の影響分析"
        - "後方互換性の確認"
        - "ロールバック可能性の確認"
      checklist:
        - "[ ] スキーマ変更がNULL許容または default値付き"
        - "[ ] 既存データとの互換性確認済み"
        - "[ ] down マイグレーションがテスト済み"
        - "[ ] 予想実行時間の見積もり"

    - step: 2
      name: "バックアップ確認"
      commands:
        - |
          # Verify recent backup
          aws rds describe-db-snapshots \
            --db-instance-identifier triptrip-production \
            --query 'DBSnapshots[0]'

          # Create manual snapshot before migration
          aws rds create-db-snapshot \
            --db-instance-identifier triptrip-production \
            --db-snapshot-identifier pre-migration-$(date +%Y%m%d-%H%M%S)

    - step: 3
      name: "Staging検証"
      commands:
        - |
          # Run migration on staging
          npx prisma migrate deploy --preview-feature

          # Verify data integrity
          npm run test:integration

  migration_execution:
    - step: 4
      name: "マイグレーション実行"
      timing: "低トラフィック時間帯（JST 02:00-06:00推奨）"
      commands:
        - |
          # Notify start
          ./scripts/notify-migration-start.sh

          # Run migration
          npx prisma migrate deploy

          # Verify migration status
          npx prisma migrate status

    - step: 5
      name: "データ検証"
      commands:
        - |
          # Row count verification
          psql -c "SELECT count(*) FROM users;"

          # Data integrity check
          npm run db:verify

  rollback:
    - step: 6
      name: "ロールバック（必要な場合）"
      options:
        schema_rollback:
          condition: "スキーマ変更のみの場合"
          command: |
            npx prisma migrate resolve --rolled-back <migration_name>

        point_in_time_recovery:
          condition: "データ損失の可能性がある場合"
          command: |
            aws rds restore-db-instance-to-point-in-time \
              --source-db-instance-identifier triptrip-production \
              --target-db-instance-identifier triptrip-production-recovery \
              --restore-time <timestamp>
```

---

## 8. 実装ロードマップ

### 8.1 フェーズ別実装計画

```yaml
implementation_roadmap:
  phase_1_foundation:
    timeline: "Month 1-2"
    objectives:
      - "基本的なCanaryデプロイメント"
      - "ArgoCD GitOpsの確立"
      - "基本的な自動ロールバック"
    deliverables:
      - "Argo Rollouts設定"
      - "ArgoCD Application/ApplicationSet"
      - "基本的なAnalysisTemplate"
      - "デプロイメントスクリプト"
    success_criteria:
      - "Canaryデプロイメントが動作"
      - "自動ロールバックが機能"
      - "GitOps経由でのデプロイが確立"

  phase_2_observability:
    timeline: "Month 3-4"
    objectives:
      - "高度なメトリクス分析"
      - "自動異常検知"
      - "カスタムダッシュボード"
    deliverables:
      - "Datadog統合AnalysisTemplate"
      - "カスタムメトリクス収集"
      - "アラート自動化"
    success_criteria:
      - "メトリクスベースの自動判定"
      - "異常検知精度 > 95%"

  phase_3_progressive:
    timeline: "Month 5-6"
    objectives:
      - "Feature Flags統合"
      - "段階的ロールアウト"
      - "地理的デプロイ"
    deliverables:
      - "LaunchDarkly統合"
      - "リングベースロールアウト"
      - "マルチリージョンデプロイ"
    success_criteria:
      - "Feature Flagsでの制御"
      - "段階的ロールアウトの自動化"

  phase_4_optimization:
    timeline: "Month 7-12"
    objectives:
      - "AI/ML駆動の最適化"
      - "自律的なデプロイメント"
      - "高度な安全機構"
    deliverables:
      - "ML異常検知"
      - "自動最適化"
      - "カオスエンジニアリング統合"
    success_criteria:
      - "完全自動化されたデプロイメント"
      - "変更失敗率 < 5%"
```

### 8.2 ツール導入スケジュール

```yaml
tool_adoption_schedule:
  immediate:
    - tool: "Argo Rollouts"
      purpose: "Canary/Blue-Green デプロイメント"
      status: "導入中"

    - tool: "ArgoCD"
      purpose: "GitOps"
      status: "導入中"

  short_term:
    - tool: "LaunchDarkly"
      purpose: "Feature Flags"
      timeline: "Month 3"

    - tool: "Datadog"
      purpose: "APM/Metrics"
      timeline: "Month 2"

  medium_term:
    - tool: "Flagger"
      purpose: "高度なProgressive Delivery"
      timeline: "Month 6"

    - tool: "Chaos Mesh"
      purpose: "カオスエンジニアリング"
      timeline: "Month 9"
```

---

## 9. 文書間参照 & 統合ポイント

### 9.1 関連文書

```yaml
document_references:
  upstream_documents:
    - doc_id: "Doc-QA-003"
      title: "リリース管理＆ロールバック手順"
      relationship: "プロセスレベルの定義"
      integration:
        - "リリース承認後に本文書の技術手順を実行"
        - "ロールバック判断基準は Doc-QA-003 を参照"

    - doc_id: "Doc-DM-002"
      title: "DevOps＆継続的デリバリー"
      relationship: "CI/CDパイプラインの詳細"
      integration:
        - "ビルド・テストパイプラインとの連携"

    - doc_id: "Doc-IA-002"
      title: "Kubernetes & コンテナ戦略"
      relationship: "インフラ基盤"
      integration:
        - "Kubernetes設定の基盤"
        - "サービスメッシュ（Istio）との連携"

  downstream_documents:
    - doc_id: "Doc-TR-002"
      title: "プラットフォームマイグレーション戦略"
      relationship: "マイグレーション時のデプロイメント"

    - doc_id: "Doc-QA-002"
      title: "モニタリング＆インシデント管理"
      relationship: "デプロイメント後の監視"

  cross_references:
    - doc_id: "Doc-TR-003"
      title: "技術的負債管理"
      relationship: "デプロイメントの技術的負債"
```

### 9.2 技術スタックサマリー

```yaml
technology_stack_summary:
  deployment_orchestration:
    tool: "Argo Rollouts"
    version: "1.6.x"
    purpose: "Progressive Delivery"

  gitops:
    tool: "ArgoCD"
    version: "2.10.x"
    purpose: "宣言的デプロイメント"

  feature_flags:
    tool: "LaunchDarkly"
    purpose: "Feature Management"

  service_mesh:
    tool: "Istio"
    version: "1.20.x"
    purpose: "トラフィック管理"

  monitoring:
    tools:
      - "Datadog (APM, Metrics)"
      - "Prometheus (Metrics)"
      - "Grafana (Dashboards)"
    purpose: "可観測性"

  container_registry:
    tool: "Amazon ECR"
    purpose: "コンテナイメージ管理"
```

---

## 付録

### A. コマンドリファレンス

| 操作 | コマンド |
|------|---------|
| Rollout状態確認 | `kubectl argo rollouts get rollout triptrip-api -n triptrip-production` |
| Rollout監視 | `kubectl argo rollouts get rollout triptrip-api -n triptrip-production --watch` |
| Canary昇格 | `kubectl argo rollouts promote triptrip-api -n triptrip-production` |
| 完全昇格 | `kubectl argo rollouts promote triptrip-api -n triptrip-production --full` |
| 中止 | `kubectl argo rollouts abort triptrip-api -n triptrip-production` |
| ロールバック | `kubectl argo rollouts undo triptrip-api -n triptrip-production` |
| ArgoCD同期 | `argocd app sync triptrip-api-production` |
| ArgoCD状態 | `argocd app get triptrip-api-production` |

### B. トラブルシューティング

| 問題 | 原因 | 対処 |
|------|------|------|
| Canaryが進まない | Analysis失敗 | `kubectl get analysisruns` で詳細確認 |
| Rollbackが完了しない | Pod termination遅延 | `terminationGracePeriodSeconds` 確認 |
| メトリクスが取得できない | Prometheus/Datadog接続 | Provider設定確認 |
| ArgoCD同期失敗 | マニフェストエラー | `argocd app diff` で差分確認 |

### C. 緊急連絡先

| 役割 | 連絡先 |
|------|--------|
| Platform Team | @platform-team (Slack) |
| On-Call Primary | PagerDuty: triptrip-primary |
| Tech Lead | @tech-lead (Slack) |
| ArgoCD Admin | @argocd-admins (Slack) |

---

**文書情報**
- Document ID: Doc-DO-003
- Version: 1.0.0
- Last Updated: 2026-01-20
- Status: Active
- Total Lines: 1,500+
- Author: Technical Architecture Team

**関連文書**
- Doc-QA-003: リリース管理＆ロールバック手順
- Doc-IA-002: Kubernetes & コンテナ戦略
- Doc-DM-002: DevOps＆継続的デリバリー
- Doc-TR-002: プラットフォームマイグレーション戦略

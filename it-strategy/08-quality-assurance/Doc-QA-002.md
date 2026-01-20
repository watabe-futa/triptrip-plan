# Doc-QA-002: モニタリング、アラート＆インシデント管理

## 文書メタデータ
| 項目 | 内容 |
|------|------|
| 文書ID | Doc-QA-002 |
| タイトル | モニタリング、アラート＆インシデント管理 |
| バージョン | 1.0.0 |
| 作成日 | 2026-01-20 |
| 最終更新日 | 2026-01-20 |
| 作成者 | IT戦略チーム |
| 承認者 | CTO |
| 分類 | IT戦略 - 品質保証 |
| 関連文書 | Doc-DM-001, Doc-DM-002, Doc-QA-001, Doc-IR-001 |

---

## 変更履歴

| バージョン | 日付 | 変更者 | 変更内容 |
|------------|------|--------|----------|
| 1.0.0 | 2026-01-20 | IT戦略チーム | 初版作成 |

---

## 目次

1. [エグゼクティブサマリー](#1-エグゼクティブサマリー)
2. [オブザーバビリティ戦略](#2-オブザーバビリティ戦略)
3. [メトリクス収集アーキテクチャ](#3-メトリクス収集アーキテクチャ)
4. [ログ管理戦略](#4-ログ管理戦略)
5. [分散トレーシング](#5-分散トレーシング)
6. [アラートルール設計](#6-アラートルール設計)
7. [SLO/SLI定義とエラーバジェット](#7-slosli定義とエラーバジェット)
8. [インシデント対応手順](#8-インシデント対応手順)
9. [オンコールローテーション](#9-オンコールローテーション)
10. [ランブック管理](#10-ランブック管理)
11. [継続的モニタリングダッシュボード](#11-継続的モニタリングダッシュボード)
12. [ポストモーテムプラクティス](#12-ポストモーテムプラクティス)

---

## 1. エグゼクティブサマリー

### 1.1 文書の目的

本文書は、TripTripプラットフォームの**包括的なモニタリング、アラート、インシデント管理戦略**を定義する。Google SREの原則とDataDogベストプラクティスを基盤とし、99.9%のサービス可用性を実現するための運用体制を確立する。

### 1.2 現状分析

既存TripTripアプリケーションの現状（EXISTING_APP_ANALYSIS.mdより）：

| 項目 | 現状 | 目標 |
|------|------|------|
| Flutterコード | 35,000行 | - |
| バックエンドコード | 4,700行 | - |
| モニタリング | 基本的なログ出力 | 完全なオブザーバビリティ |
| アラート | 未設定 | 多層アラートシステム |
| インシデント対応 | 属人的対応 | 体系的プロセス |
| MTTR | 未計測 | 30分以内 |

### 1.3 戦略概要

```
┌─────────────────────────────────────────────────────────────────┐
│                    オブザーバビリティの3本柱                      │
├─────────────────┬─────────────────┬─────────────────────────────┤
│    Metrics      │     Logs        │         Traces              │
│  (数値データ)    │  (イベント)      │     (リクエストフロー)       │
├─────────────────┼─────────────────┼─────────────────────────────┤
│ ・システム指標   │ ・アプリログ     │ ・リクエストトレース         │
│ ・ビジネス指標   │ ・アクセスログ   │ ・依存関係マッピング         │
│ ・カスタム指標   │ ・エラーログ     │ ・レイテンシ分析            │
└─────────────────┴─────────────────┴─────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      DataDog Platform                           │
├─────────────────────────────────────────────────────────────────┤
│  Dashboard │ Alerting │ APM │ Log Analytics │ Incident Mgmt    │
└─────────────────────────────────────────────────────────────────┘
```

### 1.4 主要KPI目標

| 指標 | Year 1 | Year 2 | Year 3 |
|------|--------|--------|--------|
| 可用性 | 99.5% | 99.9% | 99.95% |
| MTTR | 60分 | 30分 | 15分 |
| MTTD | 10分 | 5分 | 2分 |
| アラート精度 | 80% | 90% | 95% |
| インシデント解決率 | 90% | 95% | 98% |

---

## 2. オブザーバビリティ戦略

### 2.1 オブザーバビリティの定義

オブザーバビリティとは、システムの外部出力からシステム内部の状態を理解する能力である。

#### 2.1.1 オブザーバビリティの成熟度モデル

```
Level 5: 予測型オブザーバビリティ
         │  └─ AI/ML による異常予測、自動修復
         │
Level 4: プロアクティブオブザーバビリティ
         │  └─ SLO/エラーバジェットベースの運用
         │
Level 3: 分散トレーシング
         │  └─ End-to-End リクエスト追跡
         │
Level 2: 集中ログ管理
         │  └─ 構造化ログ、検索可能
         │
Level 1: 基本モニタリング
            └─ CPU、メモリ、ディスク監視

TripTrip目標: Year 1でLevel 3、Year 3でLevel 5達成
```

### 2.2 オブザーバビリティアーキテクチャ

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Data Sources                                   │
├──────────────┬──────────────┬──────────────┬────────────────────────┤
│ Flutter App  │ Backend API  │ Infrastructure│ External Services     │
│ ・Crashlytics│ ・APM Agent  │ ・Node Exporter│ ・Payment Gateway    │
│ ・Analytics  │ ・Log Agent  │ ・cAdvisor    │ ・Map API            │
│ ・Custom     │ ・Trace Agent│ ・kube-state  │ ・Authentication     │
└──────┬───────┴──────┬───────┴──────┬───────┴──────────┬─────────────┘
       │              │              │                  │
       ▼              ▼              ▼                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    Collection & Processing Layer                      │
├──────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                   │
│  │ DataDog     │  │ Fluentd     │  │ OpenTelemetry│                  │
│  │ Agent       │  │ Aggregator  │  │ Collector   │                   │
│  └─────────────┘  └─────────────┘  └─────────────┘                   │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    DataDog Platform                                   │
├──────────────┬──────────────┬──────────────┬────────────────────────┤
│ Metrics      │ Logs         │ APM          │ Synthetics             │
│ ・Time Series│ ・Index      │ ・Traces     │ ・API Tests            │
│ ・Custom     │ ・Archive    │ ・Service Map│ ・Browser Tests        │
│ ・Business   │ ・Pipeline   │ ・Profiling  │ ・Multi-step           │
└──────────────┴──────────────┴──────────────┴────────────────────────┘
```

### 2.3 技術スタック選定

| カテゴリ | ツール | 選定理由 |
|----------|--------|----------|
| 統合プラットフォーム | DataDog | 包括的機能、GCP統合、スケーラビリティ |
| ログ収集 | Fluentd | Kubernetes標準、柔軟なルーティング |
| トレーシング | OpenTelemetry | ベンダー非依存、業界標準 |
| モバイル監視 | Firebase Crashlytics | Flutter対応、リアルタイム |
| 合成監視 | DataDog Synthetics | グローバル拠点、CI/CD統合 |
| インシデント管理 | PagerDuty | 高度なエスカレーション、統合豊富 |

### 2.4 データ保持ポリシー

| データ種別 | Hot Storage | Warm Storage | Cold Storage |
|------------|-------------|--------------|--------------|
| メトリクス（15秒） | 15日 | - | - |
| メトリクス（1分） | 90日 | 1年 | 3年 |
| ログ（詳細） | 7日 | 30日 | 1年 |
| ログ（要約） | 30日 | 1年 | 5年 |
| トレース | 15日 | - | - |
| インシデント | 永続 | - | - |

---

## 3. メトリクス収集アーキテクチャ

### 3.1 メトリクスの分類

#### 3.1.1 インフラストラクチャメトリクス

```yaml
# GKE クラスターメトリクス
infrastructure_metrics:
  node_level:
    - cpu_utilization_percent
    - memory_utilization_percent
    - disk_io_read_bytes
    - disk_io_write_bytes
    - network_rx_bytes
    - network_tx_bytes

  pod_level:
    - pod_cpu_usage
    - pod_memory_usage
    - pod_restart_count
    - pod_ready_status

  container_level:
    - container_cpu_throttled_seconds
    - container_memory_working_set
    - container_fs_usage_bytes
```

#### 3.1.2 アプリケーションメトリクス

```yaml
# バックエンドAPIメトリクス
application_metrics:
  http_metrics:
    - http_request_duration_seconds
    - http_requests_total
    - http_request_size_bytes
    - http_response_size_bytes

  business_metrics:
    - trip_plans_created_total
    - bookings_completed_total
    - search_queries_total
    - user_sessions_active

  database_metrics:
    - db_query_duration_seconds
    - db_connections_active
    - db_connections_idle
    - db_query_errors_total
```

#### 3.1.3 モバイルアプリメトリクス

```yaml
# Flutter アプリメトリクス
mobile_metrics:
  performance:
    - app_startup_time_ms
    - screen_render_time_ms
    - api_response_time_ms
    - frame_rate_fps

  stability:
    - crash_free_users_percent
    - anr_rate_percent
    - error_rate_percent

  engagement:
    - daily_active_users
    - session_duration_seconds
    - screens_per_session
```

### 3.2 DataDog Agent設定

```yaml
# datadog-agent-config.yaml
apiVersion: datadogagent.datadoghq.com/v2alpha1
kind: DatadogAgent
metadata:
  name: datadog
  namespace: datadog
spec:
  global:
    credentials:
      apiSecret:
        secretName: datadog-secret
        keyName: api-key
      appSecret:
        secretName: datadog-secret
        keyName: app-key
    clusterName: triptrip-production
    site: datadoghq.com
    tags:
      - env:production
      - service:triptrip
      - team:platform

  features:
    apm:
      enabled: true
      hostPortConfig:
        enabled: true
        hostPort: 8126

    logCollection:
      enabled: true
      containerCollectAll: true

    processDiscovery:
      enabled: true

    npm:
      enabled: true

    usm:
      enabled: true

  override:
    nodeAgent:
      containers:
        agent:
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
      tolerations:
        - operator: Exists
```

### 3.3 カスタムメトリクス実装

#### 3.3.1 バックエンドカスタムメトリクス（TypeScript）

```typescript
// src/infrastructure/monitoring/metrics.ts
import { StatsD } from 'hot-shots';

// DataDog StatsD クライアント
const dogstatsd = new StatsD({
  host: process.env.DD_AGENT_HOST || 'localhost',
  port: 8125,
  prefix: 'triptrip.',
  globalTags: {
    env: process.env.NODE_ENV || 'development',
    service: 'triptrip-api',
    version: process.env.APP_VERSION || '1.0.0',
  },
});

// メトリクス収集クラス
export class MetricsCollector {
  // HTTPリクエストメトリクス
  recordHttpRequest(
    method: string,
    path: string,
    statusCode: number,
    durationMs: number
  ): void {
    const tags = [
      `method:${method}`,
      `path:${this.normalizePath(path)}`,
      `status_code:${statusCode}`,
      `status_class:${Math.floor(statusCode / 100)}xx`,
    ];

    dogstatsd.histogram('http.request.duration', durationMs, tags);
    dogstatsd.increment('http.request.count', 1, tags);
  }

  // ビジネスメトリクス
  recordTripPlanCreated(userId: string, planType: string): void {
    dogstatsd.increment('business.trip_plan.created', 1, [
      `plan_type:${planType}`,
    ]);
    dogstatsd.set('business.users.active', userId);
  }

  recordBookingCompleted(
    amount: number,
    currency: string,
    provider: string
  ): void {
    dogstatsd.increment('business.booking.completed', 1, [
      `currency:${currency}`,
      `provider:${provider}`,
    ]);
    dogstatsd.histogram('business.booking.amount', amount, [
      `currency:${currency}`,
    ]);
  }

  recordSearchQuery(
    queryType: string,
    resultCount: number,
    durationMs: number
  ): void {
    dogstatsd.histogram('business.search.duration', durationMs, [
      `query_type:${queryType}`,
    ]);
    dogstatsd.histogram('business.search.result_count', resultCount, [
      `query_type:${queryType}`,
    ]);
  }

  // データベースメトリクス
  recordDbQuery(
    operation: string,
    table: string,
    durationMs: number,
    success: boolean
  ): void {
    const tags = [
      `operation:${operation}`,
      `table:${table}`,
      `success:${success}`,
    ];

    dogstatsd.histogram('db.query.duration', durationMs, tags);
    if (!success) {
      dogstatsd.increment('db.query.errors', 1, tags);
    }
  }

  // キャッシュメトリクス
  recordCacheOperation(
    operation: 'get' | 'set' | 'delete',
    hit: boolean,
    durationMs: number
  ): void {
    dogstatsd.histogram('cache.operation.duration', durationMs, [
      `operation:${operation}`,
    ]);
    if (operation === 'get') {
      dogstatsd.increment(
        hit ? 'cache.hit' : 'cache.miss',
        1
      );
    }
  }

  // ゲージメトリクス
  recordActiveConnections(count: number): void {
    dogstatsd.gauge('connections.active', count);
  }

  recordQueueDepth(queueName: string, depth: number): void {
    dogstatsd.gauge('queue.depth', depth, [`queue:${queueName}`]);
  }

  // パス正規化（カーディナリティ制御）
  private normalizePath(path: string): string {
    return path
      .replace(/\/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}/gi, '/:uuid')
      .replace(/\/\d+/g, '/:id');
  }
}

export const metrics = new MetricsCollector();
```

#### 3.3.2 Honoミドルウェア統合

```typescript
// src/middleware/metrics.middleware.ts
import { Context, Next } from 'hono';
import { metrics } from '../infrastructure/monitoring/metrics';

export const metricsMiddleware = async (c: Context, next: Next) => {
  const startTime = Date.now();

  await next();

  const duration = Date.now() - startTime;
  const method = c.req.method;
  const path = c.req.path;
  const status = c.res.status;

  metrics.recordHttpRequest(method, path, status, duration);
};
```

### 3.4 Flutterアプリメトリクス

```dart
// lib/infrastructure/monitoring/app_metrics.dart
import 'package:firebase_analytics/firebase_analytics.dart';
import 'package:firebase_performance/firebase_performance.dart';

class AppMetrics {
  static final FirebaseAnalytics _analytics = FirebaseAnalytics.instance;
  static final FirebasePerformance _performance = FirebasePerformance.instance;

  // 画面表示時間
  static Future<void> recordScreenLoad(
    String screenName,
    Duration loadTime,
  ) async {
    await _analytics.logEvent(
      name: 'screen_load',
      parameters: {
        'screen_name': screenName,
        'load_time_ms': loadTime.inMilliseconds,
      },
    );
  }

  // API呼び出しパフォーマンス
  static Future<T> trackApiCall<T>(
    String endpoint,
    Future<T> Function() apiCall,
  ) async {
    final trace = _performance.newTrace('api_call_$endpoint');
    await trace.start();

    try {
      final result = await apiCall();
      trace.putAttribute('success', 'true');
      return result;
    } catch (e) {
      trace.putAttribute('success', 'false');
      trace.putAttribute('error', e.runtimeType.toString());
      rethrow;
    } finally {
      await trace.stop();
    }
  }

  // ユーザーアクション
  static Future<void> recordUserAction(
    String action,
    Map<String, dynamic>? parameters,
  ) async {
    await _analytics.logEvent(
      name: action,
      parameters: parameters?.map(
        (key, value) => MapEntry(key, value.toString()),
      ),
    );
  }

  // エラー記録
  static Future<void> recordError(
    String errorType,
    String message, {
    StackTrace? stackTrace,
  }) async {
    await _analytics.logEvent(
      name: 'app_error',
      parameters: {
        'error_type': errorType,
        'message': message.substring(0, message.length.clamp(0, 100)),
      },
    );
  }
}
```

---

## 4. ログ管理戦略

### 4.1 ログレベル定義

| レベル | 用途 | 本番環境 | 開発環境 |
|--------|------|----------|----------|
| FATAL | システム停止を伴う致命的エラー | ✓ | ✓ |
| ERROR | 処理失敗、要対応エラー | ✓ | ✓ |
| WARN | 潜在的問題、注意事項 | ✓ | ✓ |
| INFO | 重要なビジネスイベント | ✓ | ✓ |
| DEBUG | 詳細なデバッグ情報 | ✗ | ✓ |
| TRACE | 最詳細トレース情報 | ✗ | ✓ |

### 4.2 構造化ログフォーマット

```typescript
// src/infrastructure/logging/logger.ts
import pino from 'pino';

// 構造化ログ設定
export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => ({ level: label }),
    bindings: (bindings) => ({
      pid: bindings.pid,
      host: bindings.hostname,
    }),
  },
  base: {
    service: 'triptrip-api',
    version: process.env.APP_VERSION,
    env: process.env.NODE_ENV,
  },
  timestamp: () => `,"timestamp":"${new Date().toISOString()}"`,
  redact: {
    paths: [
      'password',
      'authorization',
      'cookie',
      'creditCard',
      'email',
      '*.password',
      '*.token',
    ],
    censor: '[REDACTED]',
  },
});

// コンテキスト付きロガー
export const createContextLogger = (context: {
  requestId?: string;
  userId?: string;
  traceId?: string;
  spanId?: string;
}) => {
  return logger.child({
    ...context,
    dd: {
      trace_id: context.traceId,
      span_id: context.spanId,
    },
  });
};

// ログ出力例
/*
{
  "level": "info",
  "timestamp": "2026-01-20T10:30:00.000Z",
  "service": "triptrip-api",
  "version": "1.2.3",
  "env": "production",
  "requestId": "req-123",
  "userId": "user-456",
  "dd": {
    "trace_id": "abc123",
    "span_id": "def456"
  },
  "message": "Trip plan created successfully",
  "planId": "plan-789",
  "destination": "Tokyo"
}
*/
```

### 4.3 Fluentd設定

```yaml
# fluentd-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: logging
data:
  fluent.conf: |
    # 入力: Kubernetes コンテナログ
    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      read_from_head true
      <parse>
        @type json
        time_key timestamp
        time_format %Y-%m-%dT%H:%M:%S.%NZ
      </parse>
    </source>

    # Kubernetes メタデータ付与
    <filter kubernetes.**>
      @type kubernetes_metadata
      @id filter_kube_metadata
    </filter>

    # JSON パース（アプリケーションログ）
    <filter kubernetes.**>
      @type parser
      key_name log
      reserve_data true
      remove_key_name_field true
      <parse>
        @type json
      </parse>
    </filter>

    # サービス別タグ付け
    <match kubernetes.var.log.containers.**triptrip-api**.log>
      @type rewrite_tag_filter
      <rule>
        key $.kubernetes.labels.app
        pattern triptrip-api
        tag triptrip.api
      </rule>
    </match>

    # DataDog 出力
    <match triptrip.**>
      @type datadog
      @id out_datadog
      api_key "#{ENV['DD_API_KEY']}"
      service triptrip
      source kubernetes
      source_category application

      <buffer>
        @type memory
        flush_interval 5s
        chunk_limit_size 5MB
        queue_limit_length 32
        retry_max_interval 30
        retry_forever true
      </buffer>
    </match>

    # エラーログ専用出力（即座に転送）
    <match triptrip.api>
      @type copy
      <store>
        @type datadog
        api_key "#{ENV['DD_API_KEY']}"
        <buffer>
          flush_interval 1s
        </buffer>
        <filter>
          @type grep
          <regexp>
            key level
            pattern /^(error|fatal)$/
          </regexp>
        </filter>
      </store>
    </match>
```

### 4.4 ログパイプライン（DataDog）

```yaml
# DataDog Log Pipeline 設定
log_pipelines:
  - name: TripTrip API Pipeline
    filter:
      query: "service:triptrip-api"
    processors:
      # タイムスタンプ抽出
      - type: date_remapper
        sources:
          - timestamp

      # ステータス抽出
      - type: status_remapper
        sources:
          - level

      # メッセージ抽出
      - type: message_remapper
        sources:
          - message
          - msg

      # トレースID関連付け
      - type: trace_id_remapper
        sources:
          - dd.trace_id

      # エラー情報抽出
      - type: grok_parser
        source: error
        grok_rules: |
          error_rule %{WORD:error_type}: %{GREEDYDATA:error_message}

      # 属性抽出
      - type: attribute_remapper
        sources:
          - userId
        target: usr.id
        target_type: attribute

      # URL パース
      - type: url_parser
        source: http.url
        target: http.url_details
```

### 4.5 ログベースメトリクス

```yaml
# ログからメトリクス生成
log_metrics:
  - name: triptrip.api.errors.count
    query: "service:triptrip-api status:error"
    group_by:
      - error_type
      - path
    compute:
      type: count

  - name: triptrip.api.slow_requests.count
    query: "service:triptrip-api @duration:>1000"
    group_by:
      - path
      - method
    compute:
      type: count

  - name: triptrip.api.user_actions.count
    query: "service:triptrip-api @action:*"
    group_by:
      - action
    compute:
      type: count
```

---

## 5. 分散トレーシング

### 5.1 OpenTelemetry統合

```typescript
// src/infrastructure/tracing/tracer.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { DatadogPropagator } from 'datadog-propagator';
import { DatadogExporter } from '@datadog/opentelemetry-exporter';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';

// OpenTelemetry SDK 初期化
export const initTracing = () => {
  const sdk = new NodeSDK({
    resource: new Resource({
      [SemanticResourceAttributes.SERVICE_NAME]: 'triptrip-api',
      [SemanticResourceAttributes.SERVICE_VERSION]: process.env.APP_VERSION,
      [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV,
    }),
    textMapPropagator: new DatadogPropagator(),
    traceExporter: new DatadogExporter({
      apiKey: process.env.DD_API_KEY,
      site: 'datadoghq.com',
    }),
    instrumentations: [
      getNodeAutoInstrumentations({
        '@opentelemetry/instrumentation-http': {
          requestHook: (span, request) => {
            span.setAttribute('http.request_id',
              request.headers['x-request-id'] as string);
          },
        },
        '@opentelemetry/instrumentation-pg': {
          enhancedDatabaseReporting: true,
        },
      }),
    ],
  });

  sdk.start();

  process.on('SIGTERM', () => {
    sdk.shutdown()
      .then(() => console.log('Tracing terminated'))
      .catch((error) => console.log('Error terminating tracing', error))
      .finally(() => process.exit(0));
  });
};
```

### 5.2 カスタムスパン実装

```typescript
// src/infrastructure/tracing/custom-spans.ts
import { trace, SpanStatusCode, SpanKind } from '@opentelemetry/api';

const tracer = trace.getTracer('triptrip-api');

// ビジネスロジックトレーシング
export const traceBusinessOperation = async <T>(
  operationName: string,
  attributes: Record<string, string>,
  operation: () => Promise<T>
): Promise<T> => {
  return tracer.startActiveSpan(
    operationName,
    {
      kind: SpanKind.INTERNAL,
      attributes,
    },
    async (span) => {
      try {
        const result = await operation();
        span.setStatus({ code: SpanStatusCode.OK });
        return result;
      } catch (error) {
        span.setStatus({
          code: SpanStatusCode.ERROR,
          message: error instanceof Error ? error.message : 'Unknown error',
        });
        span.recordException(error as Error);
        throw error;
      } finally {
        span.end();
      }
    }
  );
};

// 外部API呼び出しトレーシング
export const traceExternalCall = async <T>(
  serviceName: string,
  endpoint: string,
  call: () => Promise<T>
): Promise<T> => {
  return tracer.startActiveSpan(
    `external.${serviceName}`,
    {
      kind: SpanKind.CLIENT,
      attributes: {
        'peer.service': serviceName,
        'http.url': endpoint,
      },
    },
    async (span) => {
      try {
        const result = await call();
        span.setStatus({ code: SpanStatusCode.OK });
        return result;
      } catch (error) {
        span.setStatus({ code: SpanStatusCode.ERROR });
        span.recordException(error as Error);
        throw error;
      } finally {
        span.end();
      }
    }
  );
};

// 使用例
export class TripPlanService {
  async createTripPlan(userId: string, destination: string) {
    return traceBusinessOperation(
      'trip_plan.create',
      {
        'user.id': userId,
        'trip.destination': destination,
      },
      async () => {
        // 外部API呼び出し
        const weather = await traceExternalCall(
          'weather-api',
          `https://api.weather.com/forecast/${destination}`,
          () => this.weatherService.getForecast(destination)
        );

        // データベース操作（自動計装）
        const plan = await this.planRepository.create({
          userId,
          destination,
          weather,
        });

        return plan;
      }
    );
  }
}
```

### 5.3 サービスマップ設定

```yaml
# DataDog Service Catalog 設定
service_catalog:
  - name: triptrip-api
    type: web
    team: backend
    languages:
      - typescript
    contacts:
      - type: slack
        contact: "#triptrip-backend"
      - type: email
        contact: backend@triptrip.com
    links:
      - name: Documentation
        type: doc
        url: https://docs.triptrip.com/api
      - name: Repository
        type: repo
        url: https://github.com/triptrip/api
    integrations:
      pagerduty: triptrip-api-service

  - name: triptrip-db
    type: database
    team: platform
    dependencies:
      - triptrip-api

  - name: weather-api
    type: external
    team: external
    dependencies:
      - triptrip-api

---

## 6. アラートルール設計

### 6.1 アラート分類

| 優先度 | 名称 | 応答時間 | 通知先 | 例 |
|--------|------|----------|--------|-----|
| P1 | Critical | 5分以内 | PagerDuty即時 | サービス完全停止 |
| P2 | High | 15分以内 | PagerDuty + Slack | 主要機能障害 |
| P3 | Medium | 1時間以内 | Slack | パフォーマンス劣化 |
| P4 | Low | 翌営業日 | Email | 警告レベル |

### 6.2 アラートルール定義

#### 6.2.1 インフラストラクチャアラート

```yaml
# DataDog Monitor 定義
monitors:
  # P1: ノード障害
  - name: "[P1] GKE Node Not Ready"
    type: metric alert
    query: |
      avg(last_5m):avg:kubernetes.node.status.ready{cluster:triptrip-production} by {node} < 1
    message: |
      {{#is_alert}}
      🚨 CRITICAL: Kubernetes node {{node.name}} is not ready!

      Immediate action required.

      @pagerduty-triptrip-critical
      {{/is_alert}}

      {{#is_recovery}}
      ✅ Node {{node.name}} has recovered.
      {{/is_recovery}}
    options:
      thresholds:
        critical: 1
      notify_no_data: true
      no_data_timeframe: 10
      renotify_interval: 5
      escalation_message: "Node still not ready after 10 minutes"
    priority: 1
    tags:
      - team:platform
      - service:infrastructure

  # P1: Pod CrashLoopBackOff
  - name: "[P1] Pod CrashLoopBackOff"
    type: metric alert
    query: |
      avg(last_5m):avg:kubernetes.containers.restarts{cluster:triptrip-production,kube_deployment:triptrip-api} by {pod_name} > 5
    message: |
      {{#is_alert}}
      🚨 CRITICAL: Pod {{pod_name}} is crash looping!

      Restart count: {{value}}

      Check logs: https://app.datadoghq.com/logs?query=pod:{{pod_name}}

      @pagerduty-triptrip-critical
      {{/is_alert}}
    options:
      thresholds:
        critical: 5
        warning: 3
    priority: 1

  # P2: 高CPU使用率
  - name: "[P2] High CPU Utilization"
    type: metric alert
    query: |
      avg(last_10m):avg:kubernetes.cpu.usage.total{cluster:triptrip-production,kube_deployment:triptrip-api} by {pod_name} > 80
    message: |
      {{#is_alert}}
      ⚠️ HIGH: CPU usage above 80% on {{pod_name}}

      Current: {{value}}%

      Consider scaling up or investigating high-CPU operations.

      @slack-triptrip-alerts
      @pagerduty-triptrip-high
      {{/is_alert}}
    options:
      thresholds:
        critical: 90
        warning: 80
    priority: 2

  # P2: メモリ使用率
  - name: "[P2] High Memory Utilization"
    type: metric alert
    query: |
      avg(last_10m):avg:kubernetes.memory.usage_pct{cluster:triptrip-production,kube_deployment:triptrip-api} by {pod_name} > 85
    message: |
      {{#is_alert}}
      ⚠️ HIGH: Memory usage above 85% on {{pod_name}}

      Risk of OOM kill. Consider increasing memory limits.

      @slack-triptrip-alerts
      @pagerduty-triptrip-high
      {{/is_alert}}
    options:
      thresholds:
        critical: 95
        warning: 85
    priority: 2
```

#### 6.2.2 アプリケーションアラート

```yaml
monitors:
  # P1: エラーレートスパイク
  - name: "[P1] API Error Rate Spike"
    type: metric alert
    query: |
      sum(last_5m):sum:triptrip.http.request.count{status_class:5xx} by {path}.as_rate() /
      sum(last_5m):sum:triptrip.http.request.count{*} by {path}.as_rate() * 100 > 5
    message: |
      {{#is_alert}}
      🚨 CRITICAL: Error rate above 5% on {{path}}

      Current error rate: {{value}}%

      Immediate investigation required.

      Dashboard: https://app.datadoghq.com/dashboard/triptrip-api

      @pagerduty-triptrip-critical
      {{/is_alert}}
    options:
      thresholds:
        critical: 5
        warning: 2
    priority: 1

  # P2: レイテンシ劣化
  - name: "[P2] API Latency Degradation"
    type: metric alert
    query: |
      avg(last_10m):avg:triptrip.http.request.duration.95percentile{*} by {path} > 2000
    message: |
      {{#is_alert}}
      ⚠️ HIGH: P95 latency above 2s on {{path}}

      Current P95: {{value}}ms

      Check for slow queries or external service issues.

      @slack-triptrip-alerts
      @pagerduty-triptrip-high
      {{/is_alert}}
    options:
      thresholds:
        critical: 5000
        warning: 2000
    priority: 2

  # P2: データベース接続枯渇
  - name: "[P2] Database Connection Pool Exhaustion"
    type: metric alert
    query: |
      avg(last_5m):avg:triptrip.db.connections.active{*} /
      avg(last_5m):avg:triptrip.db.connections.max{*} * 100 > 80
    message: |
      {{#is_alert}}
      ⚠️ HIGH: Database connection pool at {{value}}% capacity

      Risk of connection exhaustion.

      @slack-triptrip-alerts
      @pagerduty-triptrip-high
      {{/is_alert}}
    options:
      thresholds:
        critical: 95
        warning: 80
    priority: 2

  # P3: キャッシュヒット率低下
  - name: "[P3] Low Cache Hit Rate"
    type: metric alert
    query: |
      avg(last_30m):avg:triptrip.cache.hit{*}.as_rate() /
      (avg:triptrip.cache.hit{*}.as_rate() + avg:triptrip.cache.miss{*}.as_rate()) * 100 < 70
    message: |
      {{#is_alert}}
      Cache hit rate dropped to {{value}}%

      Investigate cache eviction or key patterns.

      @slack-triptrip-alerts
      {{/is_alert}}
    options:
      thresholds:
        warning: 70
    priority: 3
```

#### 6.2.3 ビジネスメトリクスアラート

```yaml
monitors:
  # P2: 予約完了率低下
  - name: "[P2] Booking Conversion Drop"
    type: metric alert
    query: |
      avg(last_1h):avg:triptrip.business.booking.completed{*}.as_rate() /
      avg:triptrip.business.booking.started{*}.as_rate() * 100 < 50
    message: |
      {{#is_alert}}
      ⚠️ HIGH: Booking conversion rate dropped to {{value}}%

      Normal: >70%

      Check payment gateway, external services.

      @slack-triptrip-business
      @pagerduty-triptrip-high
      {{/is_alert}}
    options:
      thresholds:
        critical: 30
        warning: 50
    priority: 2

  # P3: 検索レスポンス異常
  - name: "[P3] Search No Results Spike"
    type: metric alert
    query: |
      avg(last_30m):avg:triptrip.business.search.result_count{*} < 1
    message: |
      {{#is_alert}}
      Search queries returning no results

      Check search index, data pipeline.

      @slack-triptrip-alerts
      {{/is_alert}}
    priority: 3
```

### 6.3 アラート抑制ルール

```yaml
# アラート抑制設定
downtimes:
  # 定期メンテナンス
  - name: "Weekly Maintenance Window"
    scope:
      - "env:production"
    recurrence:
      type: weeks
      period: 1
      week_days:
        - Sun
    start: "02:00"
    end: "04:00"
    timezone: "Asia/Tokyo"

  # デプロイ中抑制
  - name: "Deployment Suppression"
    scope:
      - "service:triptrip-api"
    duration: 900  # 15分
    # API経由で動的に有効化

# アラートグルーピング
alert_grouping:
  - name: "Pod Alerts"
    group_by:
      - kube_deployment
    aggregation_window: 300

  - name: "Database Alerts"
    group_by:
      - db_instance
    aggregation_window: 600
```

### 6.4 エスカレーションポリシー

```yaml
# PagerDuty エスカレーションポリシー
escalation_policies:
  - name: "TripTrip Critical"
    rules:
      - level: 1
        targets:
          - type: schedule
            id: "primary-oncall"
        escalation_delay: 5

      - level: 2
        targets:
          - type: schedule
            id: "secondary-oncall"
          - type: user
            id: "tech-lead"
        escalation_delay: 15

      - level: 3
        targets:
          - type: user
            id: "engineering-manager"
          - type: user
            id: "cto"
        escalation_delay: 30

  - name: "TripTrip High"
    rules:
      - level: 1
        targets:
          - type: schedule
            id: "primary-oncall"
        escalation_delay: 15

      - level: 2
        targets:
          - type: schedule
            id: "secondary-oncall"
        escalation_delay: 30
```

---

## 7. SLO/SLI定義とエラーバジェット

### 7.1 SLI（Service Level Indicators）定義

```yaml
# サービスレベル指標
slis:
  availability:
    name: "API Availability"
    description: "Percentage of successful HTTP requests"
    formula: |
      (total_requests - 5xx_errors) / total_requests * 100
    measurement:
      numerator: "sum:triptrip.http.request.count{status_class:2xx OR status_class:3xx OR status_class:4xx}"
      denominator: "sum:triptrip.http.request.count{*}"
    unit: percent

  latency:
    name: "API Latency (P99)"
    description: "99th percentile response time"
    formula: |
      requests_under_threshold / total_requests * 100
    measurement:
      numerator: "count:triptrip.http.request.duration{@duration:<500}"
      denominator: "count:triptrip.http.request.duration{*}"
    threshold: 500ms
    unit: percent

  throughput:
    name: "Request Throughput"
    description: "Requests processed per second"
    formula: "total_requests / time_window"
    measurement:
      query: "sum:triptrip.http.request.count{*}.as_rate()"
    unit: requests/second

  error_rate:
    name: "Error Rate"
    description: "Percentage of failed requests"
    formula: "5xx_errors / total_requests * 100"
    measurement:
      numerator: "sum:triptrip.http.request.count{status_class:5xx}"
      denominator: "sum:triptrip.http.request.count{*}"
    unit: percent
```

### 7.2 SLO（Service Level Objectives）定義

```yaml
# サービスレベル目標
slos:
  - name: "API Availability SLO"
    sli: availability
    targets:
      - period: 7d
        target: 99.5
        warning: 99.7
      - period: 30d
        target: 99.9
        warning: 99.95
    owner: backend-team
    tags:
      - tier:1
      - service:triptrip-api

  - name: "API Latency SLO"
    sli: latency
    targets:
      - period: 7d
        target: 95.0  # 95%のリクエストが500ms以内
        warning: 97.0
      - period: 30d
        target: 95.0
        warning: 97.0
    owner: backend-team
    tags:
      - tier:1
      - service:triptrip-api

  - name: "Search API Availability"
    sli: availability
    filter: "path:/api/v1/search*"
    targets:
      - period: 7d
        target: 99.0
        warning: 99.5
    owner: search-team
    tags:
      - tier:2
      - service:search

  - name: "Booking API Availability"
    sli: availability
    filter: "path:/api/v1/bookings*"
    targets:
      - period: 7d
        target: 99.9
        warning: 99.95
    owner: booking-team
    tags:
      - tier:1
      - service:booking
```

### 7.3 エラーバジェット計算

```typescript
// src/infrastructure/monitoring/error-budget.ts

interface ErrorBudget {
  sloTarget: number;      // e.g., 99.9
  periodDays: number;     // e.g., 30
  totalMinutes: number;
  allowedDowntimeMinutes: number;
  consumedMinutes: number;
  remainingMinutes: number;
  remainingPercent: number;
  burnRate: number;
}

export class ErrorBudgetCalculator {
  calculateBudget(
    sloTarget: number,
    periodDays: number,
    currentAvailability: number,
    elapsedDays: number
  ): ErrorBudget {
    const totalMinutes = periodDays * 24 * 60;
    const allowedDowntimeMinutes = totalMinutes * (1 - sloTarget / 100);

    const elapsedMinutes = elapsedDays * 24 * 60;
    const unavailableMinutes = elapsedMinutes * (1 - currentAvailability / 100);

    const consumedMinutes = unavailableMinutes;
    const remainingMinutes = Math.max(0, allowedDowntimeMinutes - consumedMinutes);
    const remainingPercent = (remainingMinutes / allowedDowntimeMinutes) * 100;

    // バーンレート = 実際の消費速度 / 許容消費速度
    const expectedConsumed = (elapsedDays / periodDays) * allowedDowntimeMinutes;
    const burnRate = expectedConsumed > 0 ? consumedMinutes / expectedConsumed : 0;

    return {
      sloTarget,
      periodDays,
      totalMinutes,
      allowedDowntimeMinutes,
      consumedMinutes,
      remainingMinutes,
      remainingPercent,
      burnRate,
    };
  }
}

/*
例: 99.9% SLO、30日間
- 許容ダウンタイム: 30 * 24 * 60 * 0.001 = 43.2分
- 15日経過、現在可用性99.95%
- 消費: 15 * 24 * 60 * 0.0005 = 10.8分
- 残り: 43.2 - 10.8 = 32.4分 (75%)
- バーンレート: 10.8 / 21.6 = 0.5 (良好)
*/
```

### 7.4 エラーバジェットアラート

```yaml
monitors:
  # エラーバジェット消費アラート
  - name: "[P2] Error Budget Burn Rate High"
    type: slo alert
    slo_id: "api-availability-slo"
    alert_type: burn_rate
    burn_rate_threshold:
      short_window: 5m
      short_burn_rate: 14.4  # 1時間で100%消費ペース
      long_window: 1h
      long_burn_rate: 6      # 6時間で100%消費ペース
    message: |
      {{#is_alert}}
      ⚠️ Error budget burning fast!

      Short window burn rate: {{short_window_burn_rate}}
      Long window burn rate: {{long_window_burn_rate}}

      At this rate, budget will be exhausted in {{budget_exhaustion_estimate}}

      @pagerduty-triptrip-high
      {{/is_alert}}
    priority: 2

  # エラーバジェット残量警告
  - name: "[P3] Error Budget Low"
    type: slo alert
    slo_id: "api-availability-slo"
    alert_type: budget_remaining
    thresholds:
      warning: 30   # 30%以下で警告
      critical: 10  # 10%以下でクリティカル
    message: |
      {{#is_warning}}
      Error budget at {{remaining_percent}}%

      Consider freezing non-critical deployments.

      @slack-triptrip-alerts
      {{/is_warning}}

      {{#is_alert}}
      🚨 Error budget critically low at {{remaining_percent}}%

      Freeze all deployments. Focus on reliability.

      @pagerduty-triptrip-high
      {{/is_alert}}
    priority: 3
```

### 7.5 SLOダッシュボード

```yaml
# DataDog SLO ダッシュボード設定
dashboard:
  title: "TripTrip SLO Overview"
  widgets:
    - title: "API Availability SLO"
      type: slo
      slo_id: "api-availability-slo"
      view_type: detail
      show_error_budget: true
      time_windows:
        - 7d
        - 30d

    - title: "Error Budget Remaining"
      type: timeseries
      requests:
        - query: "100 - (sum:triptrip.slo.error_budget.consumed{slo:api-availability})"
          display_type: line

    - title: "Burn Rate Trend"
      type: timeseries
      requests:
        - query: "avg:triptrip.slo.burn_rate{slo:api-availability}"
          display_type: line

    - title: "SLO Status Summary"
      type: slo_list
      filters:
        - service:triptrip-api
      columns:
        - status
        - name
        - target
        - error_budget
```

---

## 8. インシデント対応手順

### 8.1 インシデント分類

| 重大度 | 定義 | 影響範囲 | 対応チーム |
|--------|------|----------|------------|
| SEV-1 | サービス完全停止 | 全ユーザー | インシデントコマンダー + 全エンジニア |
| SEV-2 | 主要機能障害 | 多数ユーザー | インシデントコマンダー + 担当チーム |
| SEV-3 | 一部機能劣化 | 一部ユーザー | オンコールエンジニア |
| SEV-4 | 軽微な問題 | 限定的 | 通常業務時間内対応 |

### 8.2 インシデント対応フロー

```
┌─────────────────────────────────────────────────────────────────────┐
│                    インシデント対応フロー                            │
└─────────────────────────────────────────────────────────────────────┘

検知 ──────────────────────────────────────────────────────────────────►
  │
  ▼
┌─────────────────┐
│ 1. アラート発報  │  ← DataDog/PagerDuty
│   (自動検知)    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 2. 初期トリアージ│  ← オンコール (5分以内)
│   ・重大度判定   │
│   ・影響範囲確認 │
└────────┬────────┘
         │
         ├──────────── SEV-3/4 → 通常対応
         │
         ▼ SEV-1/2
┌─────────────────┐
│ 3. インシデント  │  ← Slack #incident-YYYYMMDD-N
│   宣言・通知    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 4. 役割アサイン  │
│   ・IC (指揮)   │
│   ・通信担当    │
│   ・技術リード   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 5. 調査・診断   │  ← ランブック参照
│   ・ログ確認    │
│   ・メトリクス   │
│   ・トレース    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 6. 緩和策実施   │
│   ・ロールバック │
│   ・スケール    │
│   ・フェイルオーバー│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 7. 復旧確認     │
│   ・SLI回復    │
│   ・ユーザー影響 │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 8. インシデント  │
│   クローズ      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 9. ポストモーテム│  ← 48時間以内
│   (SEV-1/2)    │
└─────────────────┘
```

### 8.3 インシデントコマンダー（IC）の責務

```markdown
## インシデントコマンダー チェックリスト

### 開始時 (最初の5分)
- [ ] インシデントSlackチャンネル作成: #incident-YYYYMMDD-N
- [ ] 役割のアサイン確認
- [ ] 初期状況の把握と共有
- [ ] ステータスページ更新判断

### 対応中
- [ ] 15分ごとの状況アップデート
- [ ] 意思決定のファシリテーション
- [ ] リソース追加の判断
- [ ] エスカレーション判断
- [ ] 外部コミュニケーション調整

### 復旧後
- [ ] 復旧確認と宣言
- [ ] インシデントタイムライン作成
- [ ] ポストモーテムスケジュール
- [ ] 初期アクションアイテム特定
```

### 8.4 コミュニケーションテンプレート

```markdown
## インシデント開始通知

🚨 **インシデント宣言: [タイトル]**

**重大度**: SEV-[1/2/3]
**開始時刻**: YYYY-MM-DD HH:MM JST
**影響**: [影響の説明]

**現在の状況**:
- [状況1]
- [状況2]

**対応チーム**:
- IC: @name
- 技術リード: @name
- 通信担当: @name

**次回更新**: HH:MM JST

---

## 状況アップデート

📊 **インシデント更新 #N**

**時刻**: HH:MM JST
**ステータス**: [調査中/緩和中/監視中]

**前回からの変更**:
- [変更点]

**現在のアクション**:
- [アクション1]
- [アクション2]

**次回更新**: HH:MM JST

---

## インシデント解決通知

✅ **インシデント解決: [タイトル]**

**解決時刻**: YYYY-MM-DD HH:MM JST
**影響時間**: X時間Y分

**根本原因（暫定）**:
[原因の説明]

**実施した対策**:
- [対策1]
- [対策2]

**フォローアップ**:
- ポストモーテム: YYYY-MM-DD
- 改善アクション: [リンク]
```

### 8.5 インシデント対応ツール

```yaml
# インシデント管理ツール設定
incident_management:
  primary_tool: PagerDuty
  integrations:
    - slack:
        channels:
          alerts: "#triptrip-alerts"
          incidents: "#triptrip-incidents"
        auto_create_channel: true
        channel_prefix: "incident-"

    - datadog:
        auto_create_dashboard: true
        include_related_monitors: true

    - status_page:
        provider: Statuspage.io
        auto_update: true
        components:
          - API
          - Web App
          - Mobile App
          - Search
          - Booking

  workflows:
    sev1_sev2:
      - create_incident_channel
      - page_oncall
      - create_war_room_link
      - update_status_page
      - notify_stakeholders

    sev3_sev4:
      - create_incident_ticket
      - notify_slack
```

---

## 9. オンコールローテーション

### 9.1 オンコール体制

```
┌─────────────────────────────────────────────────────────────────────┐
│                    オンコール体制                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Primary On-Call                Secondary On-Call                   │
│   ┌─────────────┐               ┌─────────────┐                     │
│   │ エンジニアA  │               │ エンジニアB  │                     │
│   │ (週次ローテ) │               │ (週次ローテ) │                     │
│   └──────┬──────┘               └──────┬──────┘                     │
│          │                             │                             │
│          │ 5分応答なし                  │                             │
│          └────────────────────────────►│                             │
│                                        │                             │
│                                        │ 15分応答なし                 │
│                                        ▼                             │
│                              ┌─────────────┐                         │
│                              │ Tech Lead   │                         │
│                              │ + Eng Mgr   │                         │
│                              └─────────────┘                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.2 PagerDutyスケジュール設定

```yaml
# PagerDuty Schedule 設定
schedules:
  - name: "TripTrip Primary On-Call"
    type: week
    timezone: "Asia/Tokyo"
    rotation:
      - user: engineer_a
        start: "2026-01-20T09:00:00+09:00"
      - user: engineer_b
        start: "2026-01-27T09:00:00+09:00"
      - user: engineer_c
        start: "2026-02-03T09:00:00+09:00"
      - user: engineer_d
        start: "2026-02-10T09:00:00+09:00"
    restrictions:
      - type: weekly
        start_day: 1  # Monday
        end_day: 5    # Friday
        start_time: "09:00"
        end_time: "21:00"

  - name: "TripTrip Secondary On-Call"
    type: week
    timezone: "Asia/Tokyo"
    # Primary の次週担当者が Secondary

  - name: "TripTrip Weekend On-Call"
    type: week
    timezone: "Asia/Tokyo"
    rotation:
      # 週末専用ローテーション
    restrictions:
      - type: weekly
        start_day: 6  # Saturday
        end_day: 0    # Sunday
```

### 9.3 オンコール補償とガイドライン

```markdown
## オンコール補償

### 待機手当
- 平日夜間 (21:00-09:00): 5,000円/日
- 休日 (終日): 10,000円/日
- 祝日 (終日): 15,000円/日

### 対応手当
- 実際の対応時間: 通常時給 × 1.25 (深夜 × 1.5)
- SEV-1/2 対応: +10,000円/インシデント
- 代休付与: 休日対応2時間以上で半日、4時間以上で1日

## オンコールガイドライン

### 待機中の義務
1. アラート通知を受信可能な状態を維持
2. 5分以内にアラートを確認
3. VPN接続可能な環境を確保
4. アルコール摂取は控える

### 対応手順
1. アラート確認後、PagerDutyでAcknowledge
2. 該当ランブックを参照
3. 必要に応じてエスカレーション
4. 対応完了後、インシデントレポート記入

### 引き継ぎ
1. シフト終了30分前に状況確認
2. 未解決事項の引き継ぎ
3. PagerDutyでハンドオフ完了
```

### 9.4 オンコール品質メトリクス

```yaml
oncall_metrics:
  response_time:
    target: 5min
    measurement: "time_to_acknowledge"

  resolution_time:
    target:
      sev1: 1h
      sev2: 4h
      sev3: 24h

  escalation_rate:
    target: "<20%"
    measurement: "escalated_incidents / total_incidents"

  false_positive_rate:
    target: "<10%"
    measurement: "false_alerts / total_alerts"

  oncall_health:
    - pages_per_shift: "<5"
    - sleep_interruptions: "<2"
    - weekend_pages: "<3"
```

---

## 10. ランブック管理

### 10.1 ランブック構造

```markdown
# ランブック: [アラート名]

## 概要
- **アラートID**: MON-XXX
- **重大度**: P1/P2/P3
- **影響サービス**: [サービス名]
- **最終更新**: YYYY-MM-DD
- **オーナー**: [チーム名]

## アラート条件
[アラートがトリガーされる条件の説明]

## 想定される原因
1. [原因1]
2. [原因2]
3. [原因3]

## 診断手順

### Step 1: 初期確認
```bash
# コマンド例
kubectl get pods -n triptrip -l app=triptrip-api
```

### Step 2: ログ確認
[DataDog ログリンク]

### Step 3: メトリクス確認
[DataDog ダッシュボードリンク]

## 緩和手順

### Option A: [緩和策1]
```bash
# コマンド
```

### Option B: [緩和策2]
```bash
# コマンド
```

## エスカレーション
- 15分以内に解決しない場合: @secondary-oncall
- 30分以内に解決しない場合: @tech-lead

## 関連情報
- アーキテクチャ図: [リンク]
- 過去インシデント: [リンク]
- 関連ランブック: [リンク]
```

### 10.2 主要ランブック一覧

```yaml
runbooks:
  infrastructure:
    - id: RB-INFRA-001
      name: "GKE Node Not Ready"
      alert: "[P1] GKE Node Not Ready"

    - id: RB-INFRA-002
      name: "Pod CrashLoopBackOff"
      alert: "[P1] Pod CrashLoopBackOff"

    - id: RB-INFRA-003
      name: "High CPU Utilization"
      alert: "[P2] High CPU Utilization"

    - id: RB-INFRA-004
      name: "High Memory Utilization"
      alert: "[P2] High Memory Utilization"

  application:
    - id: RB-APP-001
      name: "API Error Rate Spike"
      alert: "[P1] API Error Rate Spike"

    - id: RB-APP-002
      name: "API Latency Degradation"
      alert: "[P2] API Latency Degradation"

    - id: RB-APP-003
      name: "Database Connection Exhaustion"
      alert: "[P2] Database Connection Pool Exhaustion"

  database:
    - id: RB-DB-001
      name: "Database High CPU"
      alert: "[P2] Cloud SQL High CPU"

    - id: RB-DB-002
      name: "Database Replication Lag"
      alert: "[P2] Cloud SQL Replication Lag"

    - id: RB-DB-003
      name: "Database Storage Full"
      alert: "[P1] Cloud SQL Storage Critical"
```

### 10.3 ランブック例: API Error Rate Spike

```markdown
# ランブック: API Error Rate Spike

## 概要
- **アラートID**: MON-APP-001
- **重大度**: P1
- **影響サービス**: triptrip-api
- **最終更新**: 2026-01-20
- **オーナー**: Backend Team

## アラート条件
5分間の5xxエラー率が5%を超過

## 想定される原因
1. デプロイ後のバグ
2. 外部サービス（決済、地図API）障害
3. データベース接続問題
4. メモリリーク/リソース枯渇
5. 不正なリクエストの急増

## 診断手順

### Step 1: エラー内容確認
```
DataDog Logs:
service:triptrip-api status:error

確認ポイント:
- エラーメッセージのパターン
- 影響を受けているエンドポイント
- エラー発生のタイミング
```

### Step 2: 最近のデプロイ確認
```bash
# 最近のデプロイ履歴
kubectl rollout history deployment/triptrip-api -n triptrip

# 現在のイメージバージョン
kubectl get deployment triptrip-api -n triptrip -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### Step 3: 外部サービス状態確認
```
確認先:
- 決済ゲートウェイ: https://status.stripe.com/
- Google Maps: https://status.cloud.google.com/
- 天気API: [ステータスページ]
```

### Step 4: リソース状態確認
```bash
# Pod リソース使用状況
kubectl top pods -n triptrip -l app=triptrip-api

# データベース接続数
# DataDog: triptrip.db.connections.active
```

## 緩和手順

### Option A: ロールバック（デプロイ起因の場合）
```bash
# 前バージョンへロールバック
kubectl rollout undo deployment/triptrip-api -n triptrip

# ロールバック確認
kubectl rollout status deployment/triptrip-api -n triptrip
```

### Option B: スケールアウト（負荷起因の場合）
```bash
# レプリカ増加
kubectl scale deployment/triptrip-api -n triptrip --replicas=10

# HPA確認
kubectl get hpa triptrip-api -n triptrip
```

### Option C: 外部サービス切り離し
```bash
# フィーチャーフラグで機能無効化
curl -X PATCH https://app.launchdarkly.com/api/v2/flags/triptrip/payment-enabled \
  -H "Authorization: api-key" \
  -d '{"patch": [{"op": "replace", "path": "/environments/production/on", "value": false}]}'
```

### Option D: トラフィック制限
```bash
# Rate limit 強化
kubectl apply -f k8s/rate-limit-strict.yaml
```

## 復旧確認
```
確認項目:
1. エラー率が1%以下に低下
2. レイテンシが正常範囲内
3. ユーザーからの問い合わせ減少
4. ビジネスメトリクス正常化
```

## エスカレーション
- 15分以内に原因特定できない: @secondary-oncall
- 30分以内にエラー率低下しない: @tech-lead, @engineering-manager
- ユーザー影響が広範囲: @cto, @status-page-update

## 関連情報
- API アーキテクチャ: [Doc-SA-001]
- デプロイ手順: [Doc-DM-002]
- 過去インシデント: [INC-2025-042]
```

---

## 11. 継続的モニタリングダッシュボード

### 11.1 ダッシュボード体系

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ダッシュボード階層構造                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Level 1: エグゼクティブダッシュボード                               │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ ・ビジネスKPI  ・SLO状況  ・重大インシデント  ・コスト         │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                              │                                       │
│                              ▼                                       │
│  Level 2: サービスダッシュボード                                     │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                   │
│  │ API Service │ │ Search      │ │ Booking     │                   │
│  │ Dashboard   │ │ Dashboard   │ │ Dashboard   │                   │
│  └─────────────┘ └─────────────┘ └─────────────┘                   │
│                              │                                       │
│                              ▼                                       │
│  Level 3: 技術詳細ダッシュボード                                     │
│  ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐               │
│  │ GKE   │ │ DB    │ │ Cache │ │ Queue │ │Mobile │               │
│  └───────┘ └───────┘ └───────┘ └───────┘ └───────┘               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.2 エグゼクティブダッシュボード

```yaml
# executive-dashboard.yaml
dashboard:
  title: "TripTrip Executive Overview"
  description: "High-level business and reliability metrics"
  layout_type: ordered

  widgets:
    # ヘッダー: SLO サマリー
    - title: "Service Health"
      type: group
      widgets:
        - title: "API Availability (30d)"
          type: slo
          slo_id: "api-availability-slo"
          view_mode: "overall"
          time_windows:
            - 30d

        - title: "Error Budget Remaining"
          type: query_value
          requests:
            - query: "avg:triptrip.slo.error_budget.remaining{*}"
          precision: 1
          autoscale: false
          custom_unit: "%"
          conditional_formats:
            - comparator: "<"
              value: 20
              palette: "red_on_white"
            - comparator: "<"
              value: 50
              palette: "yellow_on_white"
            - comparator: ">="
              value: 50
              palette: "green_on_white"

    # ビジネスメトリクス
    - title: "Business Metrics"
      type: group
      widgets:
        - title: "Daily Active Users"
          type: query_value
          requests:
            - query: "sum:triptrip.business.users.active{*}.rollup(avg, 86400)"

        - title: "Trip Plans Created (24h)"
          type: query_value
          requests:
            - query: "sum:triptrip.business.trip_plan.created{*}.rollup(sum, 86400)"

        - title: "Bookings Completed (24h)"
          type: query_value
          requests:
            - query: "sum:triptrip.business.booking.completed{*}.rollup(sum, 86400)"

        - title: "Revenue (24h)"
          type: query_value
          requests:
            - query: "sum:triptrip.business.booking.amount{*}.rollup(sum, 86400)"
          custom_unit: "¥"

    # インシデントサマリー
    - title: "Incident Summary (30d)"
      type: group
      widgets:
        - title: "Total Incidents"
          type: query_value
          requests:
            - query: "count:triptrip.incidents{*}.rollup(count, 2592000)"

        - title: "MTTR"
          type: query_value
          requests:
            - query: "avg:triptrip.incidents.resolution_time{*}"
          custom_unit: "min"

    # トレンドグラフ
    - title: "Trends"
      type: group
      widgets:
        - title: "Request Volume Trend"
          type: timeseries
          requests:
            - query: "sum:triptrip.http.request.count{*}.as_rate()"
              display_type: line

        - title: "Error Rate Trend"
          type: timeseries
          requests:
            - query: "sum:triptrip.http.request.count{status_class:5xx}.as_rate() / sum:triptrip.http.request.count{*}.as_rate() * 100"
              display_type: line
          yaxis:
            max: "10"
```

### 11.3 サービスダッシュボード（API）

```yaml
# api-service-dashboard.yaml
dashboard:
  title: "TripTrip API Service"
  description: "Detailed API service metrics"

  template_variables:
    - name: environment
      default: production
      available_values:
        - production
        - staging
        - development

    - name: endpoint
      default: "*"
      prefix: path

  widgets:
    # RED メトリクス (Rate, Errors, Duration)
    - title: "RED Metrics"
      type: group
      widgets:
        - title: "Request Rate"
          type: timeseries
          requests:
            - query: "sum:triptrip.http.request.count{env:$environment,$endpoint}.as_rate()"
              display_type: bars

        - title: "Error Rate (%)"
          type: timeseries
          requests:
            - query: "sum:triptrip.http.request.count{env:$environment,status_class:5xx,$endpoint}.as_rate() / sum:triptrip.http.request.count{env:$environment,$endpoint}.as_rate() * 100"
              display_type: line
          markers:
            - value: "y = 1"
              display_type: error dashed
              label: "SLO Threshold"

        - title: "Latency Distribution"
          type: timeseries
          requests:
            - query: "avg:triptrip.http.request.duration.50percentile{env:$environment,$endpoint}"
              display_type: line
              metadata:
                alias: "P50"
            - query: "avg:triptrip.http.request.duration.95percentile{env:$environment,$endpoint}"
              display_type: line
              metadata:
                alias: "P95"
            - query: "avg:triptrip.http.request.duration.99percentile{env:$environment,$endpoint}"
              display_type: line
              metadata:
                alias: "P99"

    # エンドポイント別分析
    - title: "Endpoint Breakdown"
      type: group
      widgets:
        - title: "Top Endpoints by Request Count"
          type: toplist
          requests:
            - query: "sum:triptrip.http.request.count{env:$environment} by {path}.as_rate()"
              limit: 10

        - title: "Slowest Endpoints (P95)"
          type: toplist
          requests:
            - query: "avg:triptrip.http.request.duration.95percentile{env:$environment} by {path}"
              limit: 10

        - title: "Highest Error Rate Endpoints"
          type: toplist
          requests:
            - query: "sum:triptrip.http.request.count{env:$environment,status_class:5xx} by {path}.as_rate() / sum:triptrip.http.request.count{env:$environment} by {path}.as_rate() * 100"
              limit: 10

    # 依存サービス
    - title: "Dependencies"
      type: group
      widgets:
        - title: "Database Latency"
          type: timeseries
          requests:
            - query: "avg:triptrip.db.query.duration{env:$environment} by {operation}"

        - title: "Cache Hit Rate"
          type: timeseries
          requests:
            - query: "sum:triptrip.cache.hit{env:$environment}.as_rate() / (sum:triptrip.cache.hit{env:$environment}.as_rate() + sum:triptrip.cache.miss{env:$environment}.as_rate()) * 100"

        - title: "External API Latency"
          type: timeseries
          requests:
            - query: "avg:triptrip.external.latency{env:$environment} by {service}"

    # インフラストラクチャ
    - title: "Infrastructure"
      type: group
      widgets:
        - title: "Pod Count"
          type: timeseries
          requests:
            - query: "sum:kubernetes.pods.running{env:$environment,kube_deployment:triptrip-api}"

        - title: "CPU Usage"
          type: timeseries
          requests:
            - query: "avg:kubernetes.cpu.usage.total{env:$environment,kube_deployment:triptrip-api} by {pod_name}"

        - title: "Memory Usage"
          type: timeseries
          requests:
            - query: "avg:kubernetes.memory.usage{env:$environment,kube_deployment:triptrip-api} by {pod_name}"
```

### 11.4 モバイルアプリダッシュボード

```yaml
# mobile-dashboard.yaml
dashboard:
  title: "TripTrip Mobile App"
  description: "Flutter app performance and stability metrics"

  template_variables:
    - name: platform
      default: "*"
      available_values:
        - "*"
        - ios
        - android

    - name: app_version
      default: "*"

  widgets:
    # 安定性メトリクス
    - title: "App Stability"
      type: group
      widgets:
        - title: "Crash-Free Users"
          type: query_value
          requests:
            - query: "avg:triptrip.mobile.crash_free_users{platform:$platform}"
          precision: 2
          custom_unit: "%"

        - title: "ANR Rate"
          type: query_value
          requests:
            - query: "avg:triptrip.mobile.anr_rate{platform:$platform}"
          precision: 3
          custom_unit: "%"

        - title: "Crash Trend"
          type: timeseries
          requests:
            - query: "sum:triptrip.mobile.crashes{platform:$platform}.as_rate()"

    # パフォーマンス
    - title: "Performance"
      type: group
      widgets:
        - title: "App Start Time"
          type: timeseries
          requests:
            - query: "avg:triptrip.mobile.app_startup_time{platform:$platform}"
          custom_unit: "ms"

        - title: "Screen Render Time"
          type: timeseries
          requests:
            - query: "avg:triptrip.mobile.screen_render_time{platform:$platform} by {screen_name}"

        - title: "Frame Rate"
          type: timeseries
          requests:
            - query: "avg:triptrip.mobile.frame_rate{platform:$platform}"
          custom_unit: "fps"

    # ネットワーク
    - title: "Network"
      type: group
      widgets:
        - title: "API Response Time"
          type: timeseries
          requests:
            - query: "avg:triptrip.mobile.api_response_time{platform:$platform} by {endpoint}"

        - title: "Network Error Rate"
          type: timeseries
          requests:
            - query: "sum:triptrip.mobile.network_errors{platform:$platform}.as_rate()"

    # ユーザーエンゲージメント
    - title: "Engagement"
      type: group
      widgets:
        - title: "Active Sessions"
          type: query_value
          requests:
            - query: "sum:triptrip.mobile.active_sessions{platform:$platform}"

        - title: "Session Duration"
          type: timeseries
          requests:
            - query: "avg:triptrip.mobile.session_duration{platform:$platform}"
          custom_unit: "sec"

        - title: "Screens Per Session"
          type: timeseries
          requests:
            - query: "avg:triptrip.mobile.screens_per_session{platform:$platform}"
```

### 11.5 合成監視（Synthetic Monitoring）

```yaml
# DataDog Synthetic Tests
synthetics:
  api_tests:
    - name: "API Health Check"
      type: api
      subtype: http
      request:
        method: GET
        url: "https://api.triptrip.com/health"
        timeout: 10
      assertions:
        - type: statusCode
          operator: is
          target: 200
        - type: responseTime
          operator: lessThan
          target: 500
      locations:
        - aws:ap-northeast-1  # Tokyo
        - aws:ap-southeast-1  # Singapore
        - aws:us-west-2       # Oregon
      options:
        tick_every: 60
        min_failure_duration: 120
        min_location_failed: 2

    - name: "Search API Performance"
      type: api
      subtype: http
      request:
        method: POST
        url: "https://api.triptrip.com/api/v1/search"
        body: |
          {
            "destination": "Tokyo",
            "dates": {
              "start": "2026-02-01",
              "end": "2026-02-05"
            }
          }
        headers:
          Content-Type: application/json
      assertions:
        - type: statusCode
          operator: is
          target: 200
        - type: responseTime
          operator: lessThan
          target: 2000
        - type: body
          operator: validatesJSONPath
          target: "$.results"
          targetjsonpath: length > 0
      locations:
        - aws:ap-northeast-1
      options:
        tick_every: 300

  browser_tests:
    - name: "User Login Flow"
      type: browser
      request:
        url: "https://app.triptrip.com/login"
      steps:
        - name: "Enter email"
          type: typeText
          element: "[data-testid='email-input']"
          value: "{{ SYNTHETIC_USER_EMAIL }}"

        - name: "Enter password"
          type: typeText
          element: "[data-testid='password-input']"
          value: "{{ SYNTHETIC_USER_PASSWORD }}"

        - name: "Click login"
          type: click
          element: "[data-testid='login-button']"

        - name: "Verify dashboard"
          type: assertCurrentUrl
          value: "https://app.triptrip.com/dashboard"
          timeout: 10000
      locations:
        - aws:ap-northeast-1
      options:
        tick_every: 900  # 15 minutes
        device_ids:
          - chrome.laptop_large
          - chrome.mobile_small

    - name: "Trip Planning Flow"
      type: browser
      request:
        url: "https://app.triptrip.com"
      steps:
        - name: "Search destination"
          type: typeText
          element: "[data-testid='search-input']"
          value: "Tokyo"

        - name: "Select dates"
          type: click
          element: "[data-testid='date-picker']"

        - name: "Submit search"
          type: click
          element: "[data-testid='search-button']"

        - name: "Verify results"
          type: assertElementPresent
          element: "[data-testid='search-results']"
          timeout: 5000
```

### 11.6 リアルタイムアラートビュー

```yaml
# real-time-alerts-dashboard.yaml
dashboard:
  title: "TripTrip Real-Time Alerts"
  description: "Live monitoring for on-call engineers"

  widgets:
    - title: "Active Alerts"
      type: alert_graph
      alert_id: "*"
      viz_type: toplist

    - title: "Alert Timeline"
      type: event_timeline
      query: "sources:monitor tags:service:triptrip-api"

    - title: "Recent Logs (Errors)"
      type: log_stream
      query: "service:triptrip-api status:error"
      columns:
        - timestamp
        - host
        - service
        - message
      indexes:
        - "*"
      message_display: expanded

    - title: "Live Request Map"
      type: geomap
      requests:
        - query: "sum:triptrip.http.request.count{*} by {geo.country}"

    - title: "Service Map"
      type: servicemap
      filters:
        - service:triptrip-api
```

---

## 12. ポストモーテムプラクティス

### 12.1 ポストモーテムの目的と原則

```markdown
## ポストモーテムの目的

1. **学習**: インシデントから教訓を得る
2. **改善**: 再発防止策を特定・実施する
3. **共有**: 組織全体でナレッジを共有する
4. **文化**: 非難のない改善文化を醸成する

## ブレームレス原則

- 個人を責めない（システムと仕組みに焦点）
- 失敗は学習の機会
- 正直な報告を奨励
- 心理的安全性の確保
```

### 12.2 ポストモーテム対象基準

| 条件 | 対象 |
|------|------|
| SEV-1 インシデント | 必須 |
| SEV-2 インシデント | 必須 |
| SEV-3 で再発リスク高 | 推奨 |
| ニアミス（重大障害を回避） | 推奨 |
| エラーバジェット超過 | 必須 |
| 顧客影響大 | 必須 |

### 12.3 ポストモーテムテンプレート

```markdown
# ポストモーテム: [インシデントタイトル]

## 基本情報

| 項目 | 内容 |
|------|------|
| インシデントID | INC-YYYY-NNN |
| 発生日時 | YYYY-MM-DD HH:MM - HH:MM JST |
| 重大度 | SEV-1 / SEV-2 |
| 影響時間 | X時間Y分 |
| 影響範囲 | [影響を受けたユーザー/機能] |
| インシデントコマンダー | [名前] |
| ポストモーテムリード | [名前] |
| 参加者 | [名前1], [名前2], ... |

## エグゼクティブサマリー

[2-3文でインシデントの概要を説明]

## 影響

### ユーザー影響
- 影響ユーザー数: XXX人
- 影響機能: [機能リスト]
- ビジネス影響: [収益損失、評判への影響など]

### 技術的影響
- エラー率: XX%
- 障害サービス: [サービス名]
- データ損失: あり/なし

## タイムライン

| 時刻 (JST) | イベント |
|------------|----------|
| HH:MM | [最初の異常検知] |
| HH:MM | [アラート発報] |
| HH:MM | [オンコールが応答] |
| HH:MM | [調査開始] |
| HH:MM | [原因特定] |
| HH:MM | [緩和策実施] |
| HH:MM | [復旧確認] |
| HH:MM | [インシデントクローズ] |

## 根本原因分析

### 直接原因
[インシデントの直接的なトリガーとなった事象]

### 根本原因（5 Whys分析）

1. **Why 1**: なぜサービスが停止したか？
   → [回答]

2. **Why 2**: なぜそれが発生したか？
   → [回答]

3. **Why 3**: なぜそれが防げなかったか？
   → [回答]

4. **Why 4**: なぜそれが検知できなかったか？
   → [回答]

5. **Why 5**: なぜそのような状況が生まれたか？
   → [回答]

### 寄与要因
- [要因1]
- [要因2]
- [要因3]

## 対応の評価

### うまくいったこと
- [ポジティブ1]
- [ポジティブ2]

### 改善が必要なこと
- [改善点1]
- [改善点2]

### 幸運だったこと
- [もっと悪くなる可能性があったがならなかったこと]

## アクションアイテム

| ID | アクション | 担当者 | 期限 | ステータス | 優先度 |
|----|-----------|--------|------|-----------|--------|
| AI-001 | [再発防止策1] | @name | YYYY-MM-DD | Open | P1 |
| AI-002 | [検知改善策1] | @name | YYYY-MM-DD | Open | P2 |
| AI-003 | [対応改善策1] | @name | YYYY-MM-DD | Open | P2 |
| AI-004 | [プロセス改善1] | @name | YYYY-MM-DD | Open | P3 |

## 教訓

### 技術的教訓
- [教訓1]
- [教訓2]

### プロセス的教訓
- [教訓1]
- [教訓2]

## 参照資料
- インシデントSlackチャンネル: #incident-YYYYMMDD-N
- 関連ダッシュボード: [リンク]
- 関連ログクエリ: [リンク]
- 関連コミット/PR: [リンク]
```

### 12.4 ポストモーテムプロセス

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ポストモーテムプロセス                            │
└─────────────────────────────────────────────────────────────────────┘

Day 0 (インシデント発生)
    │
    ▼
┌─────────────────┐
│ インシデント    │
│ クローズ       │
└────────┬────────┘
         │
         │ 24時間以内
         ▼
┌─────────────────┐
│ ドラフト作成    │  ← IC または ポストモーテムリード
│ タイムライン記入│
└────────┬────────┘
         │
         │ 48時間以内
         ▼
┌─────────────────┐
│ レビューミーティ │  ← 関係者全員参加
│ ング実施       │
│ ・事実確認     │
│ ・原因分析     │
│ ・AI策定      │
└────────┬────────┘
         │
         │ 1週間以内
         ▼
┌─────────────────┐
│ ドキュメント    │
│ 最終化・公開   │
└────────┬────────┘
         │
         │ 継続的
         ▼
┌─────────────────┐
│ アクションアイ  │
│ テム追跡       │
└────────┬────────┘
         │
         │ 完了時
         ▼
┌─────────────────┐
│ クローズレビュー│
└─────────────────┘
```

### 12.5 ポストモーテムレビューミーティング

```markdown
## ポストモーテムレビューアジェンダ

### 準備 (ミーティング前)
- [ ] タイムラインのドラフト共有
- [ ] 参加者への事前通知
- [ ] 関連ログ・メトリクスの準備

### ミーティング (60-90分)

1. **オープニング (5分)**
   - ブレームレスの原則確認
   - アジェンダ説明

2. **タイムラインレビュー (15分)**
   - 事実の確認と補完
   - 不明点の解消

3. **根本原因分析 (20分)**
   - 5 Whys 実施
   - 寄与要因の特定

4. **対応評価 (15分)**
   - 良かった点
   - 改善点
   - ニアミス

5. **アクションアイテム策定 (20分)**
   - 再発防止策
   - 検知改善策
   - 担当者・期限設定

6. **クロージング (5分)**
   - 次のステップ確認
   - 公開スケジュール

### ファシリテーションのポイント
- 個人攻撃を防ぐ
- 事実に基づく議論
- 全員の発言を促す
- タイムボックスを守る
```

### 12.6 ポストモーテムメトリクス

```yaml
postmortem_metrics:
  process:
    - name: "Postmortem Completion Rate"
      description: "SEV-1/2インシデントに対するポストモーテム完了率"
      target: 100%

    - name: "Time to Postmortem"
      description: "インシデントからポストモーテム完了までの時間"
      target: "<7 days"

  quality:
    - name: "Action Item Completion Rate"
      description: "期限内のアクションアイテム完了率"
      target: ">90%"

    - name: "Recurrence Rate"
      description: "同一根本原因による再発率"
      target: "<5%"

  impact:
    - name: "MTTR Improvement"
      description: "ポストモーテム実施後のMTTR改善"
      measurement: "month_over_month"

    - name: "Incident Frequency"
      description: "同カテゴリのインシデント頻度"
      measurement: "quarter_over_quarter"
```

### 12.7 ナレッジベース統合

```yaml
# ポストモーテムナレッジベース
knowledge_base:
  storage: Notion / Confluence
  structure:
    - category: "Infrastructure"
      subcategories:
        - "GKE/Kubernetes"
        - "Database"
        - "Network"
        - "Storage"

    - category: "Application"
      subcategories:
        - "API"
        - "Mobile"
        - "Third-party Integration"

    - category: "Process"
      subcategories:
        - "Deployment"
        - "Configuration"
        - "Human Error"

  indexing:
    - tags: ["root_cause", "service", "severity"]
    - full_text_search: true
    - related_runbooks: auto_link

  access:
    - view: all_engineers
    - edit: incident_participants
    - admin: sre_team

  automation:
    - auto_create_from_template: true
    - reminder_for_open_actions: weekly
    - quarterly_review_reminder: true
```

---

## 付録

### A. 用語集

| 用語 | 定義 |
|------|------|
| MTTR | Mean Time To Recovery - 復旧までの平均時間 |
| MTTD | Mean Time To Detect - 検知までの平均時間 |
| SLI | Service Level Indicator - サービスレベル指標 |
| SLO | Service Level Objective - サービスレベル目標 |
| SLA | Service Level Agreement - サービスレベル契約 |
| Error Budget | エラーバジェット - 許容される障害時間 |
| Burn Rate | バーンレート - エラーバジェット消費速度 |
| IC | Incident Commander - インシデントコマンダー |
| Runbook | ランブック - 運用手順書 |
| Postmortem | ポストモーテム - 事後分析 |

### B. 関連文書

| 文書ID | タイトル | 関連性 |
|--------|----------|--------|
| Doc-DM-001 | アジャイル開発プロセス | 開発プロセスとの連携 |
| Doc-DM-002 | DevOps＆継続的デリバリー | デプロイとの連携 |
| Doc-QA-001 | テスト戦略＆品質基準 | 品質基準との連携 |
| Doc-QA-003 | リリース管理＆ロールバック | リリースプロセスとの連携 |
| Doc-SA-002 | マイクロサービスアーキテクチャ | サービス構成の参照 |
| Doc-IA-001 | クラウドインフラ基盤設計 | インフラ監視の参照 |

### C. ツールリンク

| ツール | 用途 | URL |
|--------|------|-----|
| DataDog | 統合監視プラットフォーム | https://app.datadoghq.com |
| PagerDuty | インシデント管理 | https://triptrip.pagerduty.com |
| Statuspage | ステータスページ | https://status.triptrip.com |
| Notion | ナレッジベース | https://notion.so/triptrip |
| Slack | コミュニケーション | #triptrip-alerts, #triptrip-incidents |

---

**文書終了**

*本文書は定期的にレビューされ、最新のベストプラクティスを反映するよう更新されます。*


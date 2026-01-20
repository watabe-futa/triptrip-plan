# Doc-IA-004: TripTripモニタリング、ロギング＆オブザーバビリティ

## エグゼクティブサマリー

本文書は、TripTripプラットフォームの包括的なモニタリング、ロギング、およびオブザーバビリティ戦略を定義します。Google、Netflix、Uberが採用するオブザーバビリティのベストプラクティスを基盤とし、Prometheus/Grafana/Alertmanagerによるメトリクス収集・可視化、ELK Stack/CloudWatch Logsによる集中ログ管理、Jaeger/AWS X-Rayによる分散トレーシング、SLO/SLI定義によるサービスレベル管理を詳述します。リアルタイムのシステム状態把握、迅速な問題特定・解決、データ駆動型の意思決定を実現し、99.95%の可用性と100ms以下の平均APIレイテンシを達成するための運用基盤を構築します。本設計は、Doc-IA-001/002/003で定義されたインフラストラクチャと整合し、プロダクションレディなオブザーバビリティプラットフォームを実現します。

---

## 第1章：はじめに & コンテキスト

### 1.1 目的 & スコープ

#### 1.1.1 文書の目的

```yaml
document_purpose:
  primary_objectives:
    - システム全体の可視性確保
    - 問題の迅速な検出と解決
    - サービスレベル目標（SLO）の達成と監視
    - データ駆動型の運用意思決定

  scope:
    included:
      - メトリクス収集とダッシュボード
      - ログアーキテクチャと分析
      - 分散トレーシング
      - アラートとエスカレーション
      - SLO/SLI定義と管理
      - コスト監視と最適化

    excluded:
      - セキュリティ監視の詳細（Doc-SC-001）
      - アプリケーションレベルのAPM詳細
      - ビジネスインテリジェンス/分析

  target_audience:
    - SREチーム
    - DevOpsエンジニア
    - 開発チーム
    - 運用チーム
    - プラットフォームチーム
```

#### 1.1.2 オブザーバビリティの3本柱

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         OBSERVABILITY THREE PILLARS                                  │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│   ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐        │
│   │       METRICS       │  │        LOGS         │  │       TRACES        │        │
│   │                     │  │                     │  │                     │        │
│   │  ┌───────────────┐  │  │  ┌───────────────┐  │  │  ┌───────────────┐  │        │
│   │  │  Prometheus   │  │  │  │  Fluent Bit   │  │  │  │    Jaeger     │  │        │
│   │  │  CloudWatch   │  │  │  │  CloudWatch   │  │  │  │   AWS X-Ray   │  │        │
│   │  └───────────────┘  │  │  │    Logs       │  │  │  └───────────────┘  │        │
│   │                     │  │  └───────────────┘  │  │                     │        │
│   │  What is happening? │  │  Why did it happen? │  │  Where did it       │        │
│   │  (数値データ)        │  │  (イベントログ)      │  │  happen?            │        │
│   │                     │  │                     │  │  (リクエストフロー)   │        │
│   └─────────────────────┘  └─────────────────────┘  └─────────────────────┘        │
│                                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────────────┐  │
│   │                            UNIFIED PLATFORM                                  │  │
│   │                                                                              │  │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │  │
│   │   │   Grafana   │  │  Kibana     │  │ Alertmanager│  │  PagerDuty  │       │  │
│   │   └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘       │  │
│   │                                                                              │  │
│   │        Visualization       Analysis        Alerting       Incident          │  │
│   │                                                           Management         │  │
│   └─────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 オブザーバビリティ目標 & 成功基準

#### 1.2.1 パフォーマンス目標

```yaml
observability_targets:
  metrics:
    collection_interval: 15 seconds
    retention:
      high_resolution: 15 days
      aggregated: 1 year
    query_latency:
      p50: < 100ms
      p99: < 1s
    cardinality_limit: 10M active series

  logs:
    ingestion_latency: < 30 seconds
    search_latency:
      p50: < 500ms
      p99: < 5s
    retention:
      hot: 7 days
      warm: 30 days
      cold: 1 year
    storage_efficiency: > 10:1 compression

  traces:
    sampling_rate: 100% (errors), 10% (success)
    retention: 7 days
    trace_completion: > 99%
    span_latency_overhead: < 1ms

  alerting:
    detection_time:
      critical: < 1 minute
      warning: < 5 minutes
    false_positive_rate: < 5%
    alert_fatigue_score: < 10 alerts/day/service
    mean_time_to_acknowledge: < 5 minutes
    mean_time_to_resolve: < 30 minutes

  dashboards:
    load_time: < 3 seconds
    refresh_interval: 15 seconds (real-time)
    availability: 99.9%
```

#### 1.2.2 SLO/SLI フレームワーク

```yaml
slo_framework:
  service_tiers:
    tier_1_critical:
      description: ビジネスクリティカルなサービス
      services:
        - API Gateway
        - Payment Service
        - Authentication Service
      slo_targets:
        availability: 99.95%
        latency_p99: 200ms
        error_rate: < 0.1%
      error_budget_policy:
        budget_period: 28 days
        burn_rate_alert: 2x (24h), 10x (1h)

    tier_2_important:
      description: 重要なサービス
      services:
        - Search Service
        - Booking Service
        - Notification Service
      slo_targets:
        availability: 99.9%
        latency_p99: 500ms
        error_rate: < 0.5%

    tier_3_standard:
      description: 標準サービス
      services:
        - Analytics Service
        - Recommendation Service
        - Background Workers
      slo_targets:
        availability: 99.5%
        latency_p99: 1s
        error_rate: < 1%

  sli_definitions:
    availability:
      numerator: successful_requests
      denominator: total_requests
      success_criteria: status_code < 500

    latency:
      numerator: requests_within_threshold
      denominator: total_requests
      thresholds:
        p50: 50ms
        p90: 100ms
        p99: 200ms

    error_rate:
      numerator: error_requests
      denominator: total_requests
      error_criteria: status_code >= 500 OR timeout

    throughput:
      metric: requests_per_second
      baseline: 1000 rps
      peak: 10000 rps
```

### 1.3 主要な前提条件 & 制約

#### 1.3.1 技術的前提条件

```yaml
technical_assumptions:
  infrastructure:
    kubernetes: Amazon EKS 1.29
    cloud_provider: AWS (ap-northeast-1)
    container_runtime: containerd

  existing_tooling:
    ci_cd: GitHub Actions
    iac: Terraform
    secrets: AWS Secrets Manager

  data_volumes:
    expected_metrics: 100M data points/day
    expected_logs: 500 GB/day
    expected_traces: 10M spans/day

  budget_constraints:
    observability_budget: 15% of infrastructure cost
    storage_optimization: required
```

---

## 第2章：ロギングアーキテクチャ

### 2.1 ログ収集 & 集約パイプライン

#### 2.1.1 アーキテクチャ概要

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           LOGGING ARCHITECTURE                                       │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐    │
│  │  Application   │  │   Kubernetes   │  │      AWS       │  │    External    │    │
│  │     Pods       │  │    System      │  │   Services     │  │    Sources     │    │
│  └───────┬────────┘  └───────┬────────┘  └───────┬────────┘  └───────┬────────┘    │
│          │                   │                   │                   │              │
│          ▼                   ▼                   ▼                   ▼              │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                           FLUENT BIT (DaemonSet)                             │   │
│  │                                                                              │   │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │   │
│  │   │  INPUT   │  │  PARSER  │  │  FILTER  │  │  BUFFER  │  │  OUTPUT  │     │   │
│  │   │          │  │          │  │          │  │          │  │          │     │   │
│  │   │ - tail   │  │ - json   │  │ - modify │  │ - memory │  │ - es     │     │   │
│  │   │ - syslog │  │ - regex  │  │ - grep   │  │ - file   │  │ - s3     │     │   │
│  │   │ - tcp    │  │ - cri    │  │ - nest   │  │          │  │ - cw     │     │   │
│  │   └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                        │                                            │
│                    ┌───────────────────┼───────────────────┐                       │
│                    ▼                   ▼                   ▼                        │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐     │
│  │   Amazon OpenSearch  │  │   CloudWatch Logs    │  │      Amazon S3       │     │
│  │                      │  │                      │  │    (Archive)         │     │
│  │   Hot Storage        │  │   Real-time Insights │  │   Cold Storage       │     │
│  │   (7 days)           │  │   (30 days)          │  │   (1 year+)          │     │
│  └──────────────────────┘  └──────────────────────┘  └──────────────────────┘     │
│             │                         │                                             │
│             ▼                         ▼                                             │
│  ┌──────────────────────┐  ┌──────────────────────┐                                │
│  │   OpenSearch         │  │   CloudWatch Logs    │                                │
│  │   Dashboards         │  │   Insights           │                                │
│  │   (Kibana)           │  │                      │                                │
│  └──────────────────────┘  └──────────────────────┘                                │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### 2.1.2 Fluent Bit設定

```yaml
# Fluent Bit ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: monitoring
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020
        Health_Check  On
        storage.path              /var/log/flb-storage/
        storage.sync              normal
        storage.checksum          off
        storage.backlog.mem_limit 50M

    @INCLUDE input-kubernetes.conf
    @INCLUDE filter-kubernetes.conf
    @INCLUDE output-elasticsearch.conf
    @INCLUDE output-cloudwatch.conf

  input-kubernetes.conf: |
    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            cri
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On
        Refresh_Interval  10
        Rotate_Wait       30
        storage.type      filesystem
        Read_from_Head    False

    [INPUT]
        Name              tail
        Tag               node.syslog
        Path              /var/log/syslog,/var/log/messages
        Parser            syslog
        DB                /var/log/flb_syslog.db
        Mem_Buf_Limit     5MB

  filter-kubernetes.conf: |
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Merge_Log_Key       log_processed
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off
        Labels              On
        Annotations         Off
        Buffer_Size         32k

    [FILTER]
        Name          modify
        Match         kube.*
        Add           cluster triptrip-production
        Add           environment production
        Add           region ap-northeast-1

    [FILTER]
        Name          nest
        Match         kube.*
        Operation     lift
        Nested_under  kubernetes
        Add_prefix    k8s_

    # Exclude noisy logs
    [FILTER]
        Name          grep
        Match         kube.*
        Exclude       log health check
        Exclude       log kube-probe

    # Parse JSON logs
    [FILTER]
        Name          parser
        Match         kube.*
        Key_Name      log
        Parser        json
        Reserve_Data  True
        Preserve_Key  False

    # Add severity level
    [FILTER]
        Name          lua
        Match         kube.*
        script        /fluent-bit/scripts/severity.lua
        call          add_severity

  output-elasticsearch.conf: |
    [OUTPUT]
        Name            es
        Match           kube.*
        Host            ${ELASTICSEARCH_HOST}
        Port            443
        TLS             On
        AWS_Auth        On
        AWS_Region      ap-northeast-1
        Index           triptrip-logs
        Type            _doc
        Logstash_Format On
        Logstash_Prefix triptrip-logs
        Time_Key        @timestamp
        Time_Key_Format %Y-%m-%dT%H:%M:%S.%L%z
        Include_Tag_Key On
        Tag_Key         fluentbit_tag
        Replace_Dots    On
        Retry_Limit     5
        Buffer_Size     512KB
        storage.total_limit_size 5G

  output-cloudwatch.conf: |
    [OUTPUT]
        Name              cloudwatch_logs
        Match             kube.*
        region            ap-northeast-1
        log_group_name    /triptrip/eks/application
        log_stream_prefix eks-
        auto_create_group true
        log_retention_days 30
        extra_user_agent  container-insights

    [OUTPUT]
        Name              cloudwatch_logs
        Match             node.*
        region            ap-northeast-1
        log_group_name    /triptrip/eks/node
        log_stream_prefix node-
        auto_create_group true
        log_retention_days 30

  parsers.conf: |
    [PARSER]
        Name        json
        Format      json
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
        Time_Keep   On

    [PARSER]
        Name        cri
        Format      regex
        Regex       ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<log>.*)$
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
        Time_Keep   On

    [PARSER]
        Name        syslog
        Format      regex
        Regex       ^\<(?<pri>[0-9]+)\>(?<time>[^ ]* {1,2}[^ ]* [^ ]*) (?<host>[^ ]*) (?<ident>[a-zA-Z0-9_\/\.\-]*)(?:\[(?<pid>[0-9]+)\])?(?:[^\:]*\:)? *(?<message>.*)$
        Time_Key    time
        Time_Format %b %d %H:%M:%S

    [PARSER]
        Name        nginx
        Format      regex
        Regex       ^(?<remote>[^ ]*) (?<host>[^ ]*) (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$
        Time_Key    time
        Time_Format %d/%b/%Y:%H:%M:%S %z

  severity.lua: |
    function add_severity(tag, timestamp, record)
        local log = record["log"] or record["message"] or ""
        local severity = "INFO"

        if string.find(log, "ERROR") or string.find(log, "error") or
           string.find(log, "FATAL") or string.find(log, "fatal") then
            severity = "ERROR"
        elseif string.find(log, "WARN") or string.find(log, "warn") then
            severity = "WARN"
        elseif string.find(log, "DEBUG") or string.find(log, "debug") then
            severity = "DEBUG"
        end

        record["severity"] = severity
        return 1, timestamp, record
    end
```

### 2.2 ログフォーマット & 構造化ロギング

#### 2.2.1 標準ログフォーマット

```yaml
log_format_standards:
  application_logs:
    format: json
    required_fields:
      - timestamp: ISO 8601 format
      - level: DEBUG, INFO, WARN, ERROR, FATAL
      - service: サービス名
      - trace_id: 分散トレースID
      - span_id: スパンID
      - message: ログメッセージ

    optional_fields:
      - user_id: ユーザーID（マスク処理）
      - request_id: リクエストID
      - method: HTTPメソッド
      - path: リクエストパス
      - status_code: HTTPステータスコード
      - duration_ms: 処理時間（ミリ秒）
      - error: エラー詳細
      - stack_trace: スタックトレース

    example: |
      {
        "timestamp": "2026-01-21T10:30:45.123Z",
        "level": "INFO",
        "service": "triptrip-api",
        "trace_id": "abc123def456",
        "span_id": "span789",
        "message": "Request processed successfully",
        "request_id": "req-uuid-123",
        "method": "POST",
        "path": "/api/v1/bookings",
        "status_code": 201,
        "duration_ms": 125,
        "user_id": "user_***456"
      }

  access_logs:
    format: json
    fields:
      - timestamp
      - client_ip
      - method
      - path
      - query_string
      - status_code
      - response_size
      - response_time_ms
      - user_agent
      - referer
      - trace_id

    example: |
      {
        "timestamp": "2026-01-21T10:30:45.123Z",
        "client_ip": "203.0.113.50",
        "method": "GET",
        "path": "/api/v1/hotels",
        "query_string": "location=tokyo&checkin=2026-02-01",
        "status_code": 200,
        "response_size": 4523,
        "response_time_ms": 45,
        "user_agent": "TripTrip-iOS/1.0.0",
        "referer": "",
        "trace_id": "abc123def456"
      }

  audit_logs:
    format: json
    fields:
      - timestamp
      - event_type
      - actor_id
      - actor_type
      - resource_type
      - resource_id
      - action
      - result
      - client_ip
      - metadata

    example: |
      {
        "timestamp": "2026-01-21T10:30:45.123Z",
        "event_type": "user.login",
        "actor_id": "user-123",
        "actor_type": "user",
        "resource_type": "session",
        "resource_id": "session-456",
        "action": "create",
        "result": "success",
        "client_ip": "203.0.113.50",
        "metadata": {
          "login_method": "password",
          "mfa_used": true
        }
      }
```

#### 2.2.2 Node.js 構造化ロギング実装

```typescript
// logger.ts
import pino from 'pino';
import { context, trace } from '@opentelemetry/api';

interface LogContext {
  service: string;
  environment: string;
  version: string;
}

const baseConfig: LogContext = {
  service: process.env.SERVICE_NAME || 'triptrip-api',
  environment: process.env.NODE_ENV || 'development',
  version: process.env.APP_VERSION || '0.0.0',
};

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => ({ level: label.toUpperCase() }),
    bindings: () => ({}),
  },
  timestamp: () => `,"timestamp":"${new Date().toISOString()}"`,
  base: baseConfig,
  mixin: () => {
    const span = trace.getSpan(context.active());
    if (span) {
      const spanContext = span.spanContext();
      return {
        trace_id: spanContext.traceId,
        span_id: spanContext.spanId,
      };
    }
    return {};
  },
  redact: {
    paths: [
      'password',
      'token',
      'authorization',
      'cookie',
      '*.password',
      '*.token',
      'user.email',
      'user.phone',
    ],
    censor: '[REDACTED]',
  },
  serializers: {
    err: pino.stdSerializers.err,
    req: (req) => ({
      method: req.method,
      url: req.url,
      headers: {
        'user-agent': req.headers['user-agent'],
        'x-request-id': req.headers['x-request-id'],
      },
    }),
    res: (res) => ({
      statusCode: res.statusCode,
    }),
  },
});

// Request logging middleware
export const requestLogger = (req: any, res: any, next: any) => {
  const startTime = Date.now();
  const requestId = req.headers['x-request-id'] || crypto.randomUUID();

  req.log = logger.child({
    request_id: requestId,
    method: req.method,
    path: req.path,
  });

  res.on('finish', () => {
    const duration = Date.now() - startTime;
    req.log.info({
      message: 'Request completed',
      status_code: res.statusCode,
      duration_ms: duration,
      response_size: res.get('content-length'),
    });
  });

  next();
};

// Error logging utility
export const logError = (error: Error, context?: Record<string, any>) => {
  logger.error({
    message: error.message,
    error: {
      name: error.name,
      stack: error.stack,
      ...context,
    },
  });
};

// Audit logging utility
export const logAudit = (event: {
  eventType: string;
  actorId: string;
  actorType: string;
  resourceType: string;
  resourceId: string;
  action: string;
  result: 'success' | 'failure';
  metadata?: Record<string, any>;
}) => {
  logger.info({
    audit: true,
    event_type: event.eventType,
    actor_id: event.actorId,
    actor_type: event.actorType,
    resource_type: event.resourceType,
    resource_id: event.resourceId,
    action: event.action,
    result: event.result,
    metadata: event.metadata,
  });
};
```

### 2.3 ログ保持 & アーカイブ戦略

#### 2.3.1 保持ポリシー

```yaml
log_retention_policy:
  tiers:
    hot:
      storage: Amazon OpenSearch
      duration: 7 days
      use_case: リアルタイム検索・分析
      index_settings:
        number_of_shards: 5
        number_of_replicas: 1
        refresh_interval: 5s

    warm:
      storage: CloudWatch Logs
      duration: 30 days
      use_case: 調査・デバッグ
      settings:
        log_group_class: STANDARD
        retention_days: 30

    cold:
      storage: Amazon S3 (Glacier)
      duration: 1 year
      use_case: コンプライアンス・監査
      settings:
        storage_class: GLACIER
        lifecycle_transition: 90 days

  lifecycle_rules:
    opensearch:
      - phase: hot
        min_age: 0d
        actions:
          - rollover:
              max_size: 50gb
              max_age: 1d

      - phase: warm
        min_age: 2d
        actions:
          - shrink:
              number_of_shards: 1
          - forcemerge:
              max_num_segments: 1

      - phase: delete
        min_age: 7d
        actions:
          - delete

    s3:
      - transition:
          storage_class: STANDARD_IA
          days: 30
      - transition:
          storage_class: GLACIER
          days: 90
      - expiration:
          days: 365

  compliance_requirements:
    audit_logs:
      retention: 7 years
      immutability: enabled
      encryption: required

    access_logs:
      retention: 1 year
      encryption: required

    application_logs:
      retention: 90 days
      pii_handling: redact or mask

  cost_optimization:
    compression: enabled (gzip)
    deduplication: enabled
    sampling:
      debug_logs: 10%
      info_logs: 100%
      error_logs: 100%
```

---

## 第3章：メトリクス & ダッシュボード

### 3.1 メトリクス収集インフラ

#### 3.1.1 Prometheus アーキテクチャ

```yaml
prometheus_architecture:
  deployment:
    mode: High Availability
    replicas: 2
    retention: 15d
    storage:
      type: EBS (gp3)
      size: 500Gi
      iops: 3000

  components:
    prometheus_server:
      image: prom/prometheus:v2.48.0
      resources:
        requests:
          cpu: "2"
          memory: "8Gi"
        limits:
          cpu: "4"
          memory: "16Gi"
      config:
        global:
          scrape_interval: 15s
          evaluation_interval: 15s
          external_labels:
            cluster: triptrip-production
            region: ap-northeast-1

    alertmanager:
      replicas: 3
      image: prom/alertmanager:v0.26.0
      config:
        cluster:
          enabled: true

    thanos_sidecar:
      enabled: true
      object_store:
        type: S3
        bucket: triptrip-thanos-metrics
      retention:
        raw: 2w
        5m: 1y
        1h: 5y

  service_discovery:
    kubernetes:
      - role: pod
        namespaces:
          - triptrip-api
          - triptrip-web
          - triptrip-workers
      - role: service
      - role: endpoints
      - role: node

    static_configs:
      - targets:
          - rds-exporter:9187
          - elasticache-exporter:9121

  recording_rules:
    - name: api_metrics
      interval: 30s
      rules:
        - record: job:http_requests_total:rate5m
          expr: sum(rate(http_requests_total[5m])) by (job)

        - record: job:http_request_duration_seconds:p99
          expr: histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job))

        - record: job:http_request_errors:rate5m
          expr: sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (job)

  federation:
    enabled: true
    targets:
      - prometheus-staging:9090

---
# Prometheus Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
      external_labels:
        cluster: triptrip-production
        environment: production

    rule_files:
      - /etc/prometheus/rules/*.yml

    alerting:
      alertmanagers:
        - static_configs:
            - targets:
                - alertmanager:9093
          scheme: http
          timeout: 10s
          api_version: v2

    scrape_configs:
      # Kubernetes API Server
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
          - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
            action: keep
            regex: default;kubernetes;https

      # Kubernetes Nodes
      - job_name: 'kubernetes-nodes'
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        kubernetes_sd_configs:
          - role: node
        relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)
          - target_label: __address__
            replacement: kubernetes.default.svc:443
          - source_labels: [__meta_kubernetes_node_name]
            regex: (.+)
            target_label: __metrics_path__
            replacement: /api/v1/nodes/${1}/proxy/metrics

      # Kubernetes Pods
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
            action: replace
            regex: ([^:]+)(?::\d+)?;(\d+)
            replacement: $1:$2
            target_label: __address__
          - action: labelmap
            regex: __meta_kubernetes_pod_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: kubernetes_namespace
          - source_labels: [__meta_kubernetes_pod_name]
            action: replace
            target_label: kubernetes_pod_name

      # Application Services
      - job_name: 'triptrip-api'
        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names:
                - triptrip-api
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            action: keep
            regex: triptrip-api

      # Istio Metrics
      - job_name: 'istio-mesh'
        kubernetes_sd_configs:
          - role: endpoints
            namespaces:
              names:
                - istio-system
        relabel_configs:
          - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
            action: keep
            regex: istiod;http-monitoring

      # Node Exporter
      - job_name: 'node-exporter'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            action: keep
            regex: node-exporter
```

### 3.2 カスタムメトリクス設計

#### 3.2.1 アプリケーションメトリクス

```yaml
custom_metrics:
  http_metrics:
    - name: http_requests_total
      type: counter
      description: Total number of HTTP requests
      labels:
        - method
        - path
        - status_code
        - service

    - name: http_request_duration_seconds
      type: histogram
      description: HTTP request latency
      buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
      labels:
        - method
        - path
        - status_code

    - name: http_request_size_bytes
      type: histogram
      description: HTTP request size
      buckets: [100, 1000, 10000, 100000, 1000000]
      labels:
        - method
        - path

    - name: http_response_size_bytes
      type: histogram
      description: HTTP response size
      buckets: [100, 1000, 10000, 100000, 1000000]
      labels:
        - method
        - path

  business_metrics:
    - name: triptrip_bookings_total
      type: counter
      description: Total number of bookings
      labels:
        - type: [hotel, attraction, rental]
        - status: [pending, confirmed, cancelled]
        - payment_method

    - name: triptrip_booking_value_jpy
      type: histogram
      description: Booking value in JPY
      buckets: [1000, 5000, 10000, 50000, 100000, 500000]
      labels:
        - type

    - name: triptrip_search_requests_total
      type: counter
      description: Total search requests
      labels:
        - category
        - has_results

    - name: triptrip_search_latency_seconds
      type: histogram
      description: Search latency
      buckets: [0.05, 0.1, 0.25, 0.5, 1, 2.5, 5]

    - name: triptrip_active_users
      type: gauge
      description: Current active users
      labels:
        - platform: [ios, android, web]

    - name: triptrip_cart_items
      type: gauge
      description: Current cart items count

  database_metrics:
    - name: triptrip_db_query_duration_seconds
      type: histogram
      description: Database query duration
      buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5]
      labels:
        - query_type
        - table

    - name: triptrip_db_connections_active
      type: gauge
      description: Active database connections

    - name: triptrip_db_connections_idle
      type: gauge
      description: Idle database connections

  cache_metrics:
    - name: triptrip_cache_hits_total
      type: counter
      description: Cache hits
      labels:
        - cache_name

    - name: triptrip_cache_misses_total
      type: counter
      description: Cache misses
      labels:
        - cache_name

    - name: triptrip_cache_latency_seconds
      type: histogram
      description: Cache operation latency
      buckets: [0.0001, 0.0005, 0.001, 0.005, 0.01, 0.05]
      labels:
        - operation: [get, set, delete]

  queue_metrics:
    - name: triptrip_queue_messages_total
      type: counter
      description: Total messages processed
      labels:
        - queue_name
        - status: [success, failure]

    - name: triptrip_queue_depth
      type: gauge
      description: Current queue depth
      labels:
        - queue_name

    - name: triptrip_queue_processing_duration_seconds
      type: histogram
      description: Message processing duration
      buckets: [0.1, 0.5, 1, 5, 10, 30, 60]
      labels:
        - queue_name
```

#### 3.2.2 メトリクス実装（Node.js）

```typescript
// metrics.ts
import { Registry, Counter, Histogram, Gauge, collectDefaultMetrics } from 'prom-client';

// Create registry
export const register = new Registry();

// Enable default metrics
collectDefaultMetrics({ register });

// HTTP Metrics
export const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'path', 'status_code', 'service'],
  registers: [register],
});

export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request latency',
  labelNames: ['method', 'path', 'status_code'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
  registers: [register],
});

// Business Metrics
export const bookingsTotal = new Counter({
  name: 'triptrip_bookings_total',
  help: 'Total number of bookings',
  labelNames: ['type', 'status', 'payment_method'],
  registers: [register],
});

export const bookingValue = new Histogram({
  name: 'triptrip_booking_value_jpy',
  help: 'Booking value in JPY',
  labelNames: ['type'],
  buckets: [1000, 5000, 10000, 50000, 100000, 500000],
  registers: [register],
});

export const activeUsers = new Gauge({
  name: 'triptrip_active_users',
  help: 'Current active users',
  labelNames: ['platform'],
  registers: [register],
});

// Database Metrics
export const dbQueryDuration = new Histogram({
  name: 'triptrip_db_query_duration_seconds',
  help: 'Database query duration',
  labelNames: ['query_type', 'table'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
  registers: [register],
});

export const dbConnectionsActive = new Gauge({
  name: 'triptrip_db_connections_active',
  help: 'Active database connections',
  registers: [register],
});

// Cache Metrics
export const cacheHits = new Counter({
  name: 'triptrip_cache_hits_total',
  help: 'Cache hits',
  labelNames: ['cache_name'],
  registers: [register],
});

export const cacheMisses = new Counter({
  name: 'triptrip_cache_misses_total',
  help: 'Cache misses',
  labelNames: ['cache_name'],
  registers: [register],
});

// Metrics middleware
export const metricsMiddleware = (req: any, res: any, next: any) => {
  const start = process.hrtime.bigint();

  res.on('finish', () => {
    const duration = Number(process.hrtime.bigint() - start) / 1e9;
    const path = normalizePath(req.route?.path || req.path);

    httpRequestsTotal.inc({
      method: req.method,
      path,
      status_code: res.statusCode,
      service: 'triptrip-api',
    });

    httpRequestDuration.observe(
      { method: req.method, path, status_code: res.statusCode },
      duration
    );
  });

  next();
};

// Metrics endpoint
export const metricsHandler = async (req: any, res: any) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
};

// Helper to normalize paths
function normalizePath(path: string): string {
  return path
    .replace(/\/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}/gi, '/:id')
    .replace(/\/\d+/g, '/:id');
}
```

### 3.3 ダッシュボード設計

#### 3.3.1 Grafanaダッシュボード構成

```yaml
grafana_dashboards:
  overview:
    name: TripTrip Platform Overview
    uid: triptrip-overview
    refresh: 30s
    panels:
      - title: Request Rate
        type: graph
        query: sum(rate(http_requests_total{service="triptrip-api"}[5m]))

      - title: Error Rate
        type: graph
        query: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m])) /
          sum(rate(http_requests_total[5m])) * 100

      - title: P99 Latency
        type: graph
        query: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
          )

      - title: Active Users
        type: stat
        query: sum(triptrip_active_users)

      - title: Bookings Today
        type: stat
        query: increase(triptrip_bookings_total{status="confirmed"}[24h])

      - title: Revenue Today (JPY)
        type: stat
        query: sum(increase(triptrip_booking_value_jpy_sum[24h]))

  slo_dashboard:
    name: TripTrip SLO Dashboard
    uid: triptrip-slo
    refresh: 1m
    panels:
      - title: Availability SLI
        type: gauge
        query: |
          sum(rate(http_requests_total{status_code!~"5.."}[1h])) /
          sum(rate(http_requests_total[1h])) * 100
        thresholds:
          - value: 99
            color: red
          - value: 99.9
            color: yellow
          - value: 99.95
            color: green

      - title: Error Budget Remaining
        type: stat
        query: |
          (1 - (
            sum(increase(http_requests_total{status_code=~"5.."}[28d])) /
            sum(increase(http_requests_total[28d]))
          ) / 0.0005) * 100
        unit: percent

      - title: Latency SLI (P99 < 200ms)
        type: gauge
        query: |
          sum(rate(http_request_duration_seconds_bucket{le="0.2"}[1h])) /
          sum(rate(http_request_duration_seconds_count[1h])) * 100

      - title: Error Budget Burn Rate
        type: graph
        query: |
          sum(rate(http_requests_total{status_code=~"5.."}[1h])) /
          sum(rate(http_requests_total[1h])) /
          0.0005

  service_dashboard:
    name: TripTrip API Service
    uid: triptrip-api-service
    refresh: 15s
    panels:
      - title: Request Rate by Endpoint
        type: graph
        query: sum(rate(http_requests_total[5m])) by (path)

      - title: Latency by Endpoint
        type: heatmap
        query: |
          sum(rate(http_request_duration_seconds_bucket[5m])) by (le, path)

      - title: Error Rate by Endpoint
        type: graph
        query: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (path) /
          sum(rate(http_requests_total[5m])) by (path) * 100

      - title: Pod CPU Usage
        type: graph
        query: |
          sum(rate(container_cpu_usage_seconds_total{
            namespace="triptrip-api"
          }[5m])) by (pod)

      - title: Pod Memory Usage
        type: graph
        query: |
          sum(container_memory_usage_bytes{
            namespace="triptrip-api"
          }) by (pod)

      - title: Database Query Duration
        type: graph
        query: |
          histogram_quantile(0.99,
            sum(rate(triptrip_db_query_duration_seconds_bucket[5m])) by (le, query_type)
          )

      - title: Cache Hit Rate
        type: graph
        query: |
          sum(rate(triptrip_cache_hits_total[5m])) /
          (sum(rate(triptrip_cache_hits_total[5m])) +
           sum(rate(triptrip_cache_misses_total[5m]))) * 100

  infrastructure_dashboard:
    name: TripTrip Infrastructure
    uid: triptrip-infra
    refresh: 30s
    panels:
      - title: Node CPU Usage
        type: graph
        query: |
          100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)

      - title: Node Memory Usage
        type: graph
        query: |
          (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

      - title: Node Disk Usage
        type: graph
        query: |
          (1 - node_filesystem_avail_bytes{mountpoint="/"} /
               node_filesystem_size_bytes{mountpoint="/"}) * 100

      - title: Network Traffic
        type: graph
        query: |
          sum(rate(node_network_receive_bytes_total[5m])) by (instance)

      - title: Pod Count by Namespace
        type: bar
        query: count(kube_pod_info) by (namespace)

      - title: Container Restarts
        type: stat
        query: sum(increase(kube_pod_container_status_restarts_total[1h]))
```

---

## 第4章：分散トレーシング

### 4.1 トレーシングアーキテクチャ

#### 4.1.1 OpenTelemetry + Jaeger

```yaml
tracing_architecture:
  instrumentation:
    framework: OpenTelemetry
    sdk_version: 1.20.0
    auto_instrumentation:
      - http
      - express
      - pg (PostgreSQL)
      - ioredis
      - grpc

  collector:
    type: OpenTelemetry Collector
    deployment: DaemonSet
    receivers:
      - otlp (gRPC, HTTP)
      - jaeger
      - zipkin
    processors:
      - batch
      - memory_limiter
      - attributes
      - tail_sampling
    exporters:
      - jaeger
      - aws_xray
      - logging (debug)

  backend:
    primary: Jaeger
    secondary: AWS X-Ray
    storage:
      type: Elasticsearch
      retention: 7 days

  sampling:
    strategy: tail_based
    rules:
      - name: always_sample_errors
        type: status_code
        status_code: ERROR
        sampling_percentage: 100

      - name: sample_slow_traces
        type: latency
        threshold_ms: 1000
        sampling_percentage: 100

      - name: sample_default
        type: probabilistic
        sampling_percentage: 10

---
# OpenTelemetry Collector ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: monitoring
data:
  otel-collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

      jaeger:
        protocols:
          grpc:
            endpoint: 0.0.0.0:14250
          thrift_http:
            endpoint: 0.0.0.0:14268

    processors:
      batch:
        timeout: 1s
        send_batch_size: 1024
        send_batch_max_size: 2048

      memory_limiter:
        check_interval: 1s
        limit_mib: 1000
        spike_limit_mib: 200

      attributes:
        actions:
          - key: environment
            value: production
            action: insert
          - key: cluster
            value: triptrip-production
            action: insert

      tail_sampling:
        decision_wait: 10s
        num_traces: 100000
        expected_new_traces_per_sec: 1000
        policies:
          - name: errors
            type: status_code
            status_code:
              status_codes:
                - ERROR

          - name: slow-traces
            type: latency
            latency:
              threshold_ms: 1000

          - name: probabilistic
            type: probabilistic
            probabilistic:
              sampling_percentage: 10

    exporters:
      jaeger:
        endpoint: jaeger-collector:14250
        tls:
          insecure: true

      awsxray:
        region: ap-northeast-1
        index_all_attributes: true

      logging:
        loglevel: debug

    extensions:
      health_check:
        endpoint: 0.0.0.0:13133
      pprof:
        endpoint: 0.0.0.0:1777
      zpages:
        endpoint: 0.0.0.0:55679

    service:
      extensions: [health_check, pprof, zpages]
      pipelines:
        traces:
          receivers: [otlp, jaeger]
          processors: [memory_limiter, batch, attributes, tail_sampling]
          exporters: [jaeger, awsxray]
```

### 4.2 スパン & トレース設計

#### 4.2.1 スパン命名規則

```yaml
span_naming_conventions:
  http_server:
    format: "HTTP {method} {route}"
    examples:
      - "HTTP GET /api/v1/hotels"
      - "HTTP POST /api/v1/bookings"

  http_client:
    format: "HTTP {method} {host}"
    examples:
      - "HTTP GET payment-service"
      - "HTTP POST notification-service"

  database:
    format: "{db_system} {operation} {table}"
    examples:
      - "PostgreSQL SELECT hotels"
      - "PostgreSQL INSERT bookings"

  cache:
    format: "Redis {operation}"
    examples:
      - "Redis GET"
      - "Redis SET"

  queue:
    format: "{messaging_system} {operation} {queue}"
    examples:
      - "SQS send booking-queue"
      - "SQS receive notification-queue"

  grpc:
    format: "gRPC {service}/{method}"
    examples:
      - "gRPC PaymentService/ProcessPayment"
      - "gRPC NotificationService/SendEmail"

span_attributes:
  required:
    - service.name
    - service.version
    - deployment.environment
    - trace_id
    - span_id
    - parent_span_id

  http:
    - http.method
    - http.url
    - http.status_code
    - http.request_content_length
    - http.response_content_length
    - user_agent.original
    - http.route

  database:
    - db.system
    - db.name
    - db.statement (sanitized)
    - db.operation

  custom:
    - user.id (hashed)
    - booking.id
    - payment.id
    - error.type
    - error.message
```

#### 4.2.2 Node.js トレーシング実装

```typescript
// tracing.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-grpc';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { trace, context, SpanStatusCode, Span } from '@opentelemetry/api';

// Initialize SDK
const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: process.env.SERVICE_NAME || 'triptrip-api',
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.APP_VERSION || '1.0.0',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV || 'development',
    'cluster': 'triptrip-production',
  }),
  spanProcessor: new BatchSpanProcessor(
    new OTLPTraceExporter({
      url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://otel-collector:4317',
    })
  ),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-http': {
        requestHook: (span, request) => {
          span.setAttribute('http.request_id', request.headers['x-request-id'] || '');
        },
      },
      '@opentelemetry/instrumentation-pg': {
        enhancedDatabaseReporting: true,
      },
      '@opentelemetry/instrumentation-redis': {
        dbStatementSerializer: (cmdName, cmdArgs) => {
          return `${cmdName} ${cmdArgs[0] || ''}`;
        },
      },
    }),
  ],
});

export const initTracing = async () => {
  await sdk.start();
  console.log('Tracing initialized');

  process.on('SIGTERM', () => {
    sdk.shutdown()
      .then(() => console.log('Tracing terminated'))
      .catch((error) => console.error('Error terminating tracing', error))
      .finally(() => process.exit(0));
  });
};

// Tracer instance
export const tracer = trace.getTracer('triptrip-api');

// Custom span creation helper
export const createSpan = (
  name: string,
  fn: (span: Span) => Promise<any>,
  attributes?: Record<string, string | number | boolean>
) => {
  return tracer.startActiveSpan(name, async (span) => {
    try {
      if (attributes) {
        Object.entries(attributes).forEach(([key, value]) => {
          span.setAttribute(key, value);
        });
      }
      const result = await fn(span);
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error: any) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: error.message,
      });
      span.recordException(error);
      throw error;
    } finally {
      span.end();
    }
  });
};

// Database query tracing wrapper
export const traceDbQuery = async <T>(
  operation: string,
  table: string,
  query: () => Promise<T>
): Promise<T> => {
  return createSpan(`PostgreSQL ${operation} ${table}`, async (span) => {
    span.setAttribute('db.system', 'postgresql');
    span.setAttribute('db.operation', operation);
    span.setAttribute('db.sql.table', table);
    return query();
  });
};

// Cache tracing wrapper
export const traceCacheOperation = async <T>(
  operation: string,
  key: string,
  cacheOp: () => Promise<T>
): Promise<T> => {
  return createSpan(`Redis ${operation}`, async (span) => {
    span.setAttribute('db.system', 'redis');
    span.setAttribute('db.operation', operation);
    span.setAttribute('cache.key', key);
    return cacheOp();
  });
};

// External service call tracing
export const traceExternalCall = async <T>(
  service: string,
  operation: string,
  call: () => Promise<T>
): Promise<T> => {
  return createSpan(`External ${service} ${operation}`, async (span) => {
    span.setAttribute('peer.service', service);
    span.setAttribute('rpc.method', operation);
    return call();
  });
};

// Business operation tracing
export const traceBusinessOperation = async <T>(
  operation: string,
  metadata: Record<string, any>,
  businessLogic: () => Promise<T>
): Promise<T> => {
  return createSpan(`Business ${operation}`, async (span) => {
    Object.entries(metadata).forEach(([key, value]) => {
      if (typeof value === 'string' || typeof value === 'number' || typeof value === 'boolean') {
        span.setAttribute(`business.${key}`, value);
      }
    });
    return businessLogic();
  });
};
```

### 4.3 サンプリング戦略

```yaml
sampling_strategies:
  head_based:
    description: リクエスト開始時にサンプリング決定
    use_case: 低オーバーヘッドが必要な場合
    implementation:
      probabilistic:
        percentage: 10
      rate_limiting:
        spans_per_second: 100

  tail_based:
    description: トレース完了後にサンプリング決定
    use_case: エラーや遅延トレースを確実に収集
    implementation:
      always_sample:
        - errors
        - status_code >= 400
        - latency > 1000ms
      probabilistic:
        - successful requests: 10%
        - health checks: 0%

  adaptive:
    description: トラフィックに応じて動的調整
    use_case: コスト最適化とカバレッジのバランス
    implementation:
      target_traces_per_second: 100
      min_sampling_rate: 1%
      max_sampling_rate: 100%

sampling_rules_by_service:
  api_gateway:
    error_rate: 100%
    slow_traces: 100% (> 500ms)
    success_rate: 5%

  payment_service:
    all_traces: 100%  # 全トレース収集

  search_service:
    error_rate: 100%
    slow_traces: 100% (> 1s)
    success_rate: 1%

  background_workers:
    error_rate: 100%
    success_rate: 1%
```

---

## 第5章：アラート & インシデント対応

### 5.1 アラートルール設計

#### 5.1.1 Alertmanager設定

```yaml
# Alertmanager ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m
      slack_api_url: '${SLACK_WEBHOOK_URL}'
      pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

    route:
      group_by: ['alertname', 'service', 'severity']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: 'default'

      routes:
        # Critical alerts -> PagerDuty + Slack
        - match:
            severity: critical
          receiver: 'pagerduty-critical'
          continue: true

        - match:
            severity: critical
          receiver: 'slack-critical'
          group_wait: 10s
          repeat_interval: 1h

        # Warning alerts -> Slack
        - match:
            severity: warning
          receiver: 'slack-warning'
          repeat_interval: 4h

        # Info alerts -> Slack (low priority channel)
        - match:
            severity: info
          receiver: 'slack-info'
          repeat_interval: 24h

        # Business alerts -> dedicated channel
        - match:
            type: business
          receiver: 'slack-business'

    receivers:
      - name: 'default'
        slack_configs:
          - channel: '#alerts-default'
            send_resolved: true
            title: '{{ .CommonAnnotations.summary }}'
            text: '{{ .CommonAnnotations.description }}'

      - name: 'pagerduty-critical'
        pagerduty_configs:
          - service_key: '${PAGERDUTY_SERVICE_KEY}'
            severity: '{{ if eq .Status "firing" }}critical{{ else }}info{{ end }}'
            description: '{{ .CommonAnnotations.summary }}'
            details:
              firing: '{{ .Alerts.Firing | len }}'
              resolved: '{{ .Alerts.Resolved | len }}'

      - name: 'slack-critical'
        slack_configs:
          - channel: '#alerts-critical'
            send_resolved: true
            color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
            title: ':rotating_light: {{ .CommonAnnotations.summary }}'
            text: |
              {{ .CommonAnnotations.description }}

              *Service:* {{ .CommonLabels.service }}
              *Severity:* {{ .CommonLabels.severity }}
              *Alerts Firing:* {{ .Alerts.Firing | len }}

              {{ range .Alerts.Firing }}
              • {{ .Annotations.description }}
              {{ end }}
            actions:
              - type: button
                text: 'View in Grafana'
                url: '{{ .CommonAnnotations.runbook_url }}'
              - type: button
                text: 'Runbook'
                url: '{{ .CommonAnnotations.dashboard_url }}'

      - name: 'slack-warning'
        slack_configs:
          - channel: '#alerts-warning'
            send_resolved: true
            color: 'warning'
            title: ':warning: {{ .CommonAnnotations.summary }}'
            text: '{{ .CommonAnnotations.description }}'

      - name: 'slack-info'
        slack_configs:
          - channel: '#alerts-info'
            send_resolved: false
            title: ':information_source: {{ .CommonAnnotations.summary }}'

      - name: 'slack-business'
        slack_configs:
          - channel: '#alerts-business'
            send_resolved: true
            title: ':chart_with_upwards_trend: {{ .CommonAnnotations.summary }}'
            text: '{{ .CommonAnnotations.description }}'

    inhibit_rules:
      - source_match:
          severity: 'critical'
        target_match:
          severity: 'warning'
        equal: ['alertname', 'service']

      - source_match:
          alertname: 'ClusterDown'
        target_match_re:
          alertname: '.+'
        equal: ['cluster']

    templates:
      - '/etc/alertmanager/templates/*.tmpl'
```

#### 5.1.2 Prometheusアラートルール

```yaml
# Prometheus Alert Rules
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-rules
  namespace: monitoring
data:
  slo-alerts.yml: |
    groups:
      - name: slo-alerts
        rules:
          # SLO: Availability (99.95%)
          - alert: SLOAvailabilityBreach
            expr: |
              (
                sum(rate(http_requests_total{status_code!~"5.."}[1h])) /
                sum(rate(http_requests_total[1h]))
              ) < 0.9995
            for: 5m
            labels:
              severity: critical
              type: slo
            annotations:
              summary: "SLO Availability breach detected"
              description: "Availability is {{ $value | humanizePercentage }}, below 99.95% SLO"
              runbook_url: "https://runbooks.triptrip.com/slo-availability"

          # Error Budget Burn Rate (1h window, 10x burn rate)
          - alert: ErrorBudgetFastBurn
            expr: |
              (
                sum(rate(http_requests_total{status_code=~"5.."}[1h])) /
                sum(rate(http_requests_total[1h]))
              ) > (0.0005 * 10)
            for: 5m
            labels:
              severity: critical
              type: slo
            annotations:
              summary: "Error budget burning too fast"
              description: "Error rate is {{ $value | humanizePercentage }}, 10x normal burn rate"

          # Error Budget Burn Rate (24h window, 2x burn rate)
          - alert: ErrorBudgetSlowBurn
            expr: |
              (
                sum(rate(http_requests_total{status_code=~"5.."}[24h])) /
                sum(rate(http_requests_total[24h]))
              ) > (0.0005 * 2)
            for: 1h
            labels:
              severity: warning
              type: slo
            annotations:
              summary: "Error budget burning steadily"
              description: "24h error rate is {{ $value | humanizePercentage }}, 2x normal burn rate"

          # SLO: Latency (P99 < 200ms)
          - alert: SLOLatencyBreach
            expr: |
              histogram_quantile(0.99,
                sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
              ) > 0.2
            for: 5m
            labels:
              severity: critical
              type: slo
            annotations:
              summary: "SLO Latency breach detected"
              description: "P99 latency is {{ $value | humanizeDuration }}, above 200ms SLO"

  service-alerts.yml: |
    groups:
      - name: service-alerts
        rules:
          # High Error Rate
          - alert: HighErrorRate
            expr: |
              (
                sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (service) /
                sum(rate(http_requests_total[5m])) by (service)
              ) > 0.05
            for: 2m
            labels:
              severity: critical
            annotations:
              summary: "High error rate on {{ $labels.service }}"
              description: "Error rate is {{ $value | humanizePercentage }}"

          # Service Down
          - alert: ServiceDown
            expr: up{job=~"triptrip-.*"} == 0
            for: 1m
            labels:
              severity: critical
            annotations:
              summary: "Service {{ $labels.job }} is down"
              description: "Service {{ $labels.job }} has been down for more than 1 minute"

          # High Memory Usage
          - alert: HighMemoryUsage
            expr: |
              (
                container_memory_usage_bytes{namespace=~"triptrip-.*"} /
                container_spec_memory_limit_bytes{namespace=~"triptrip-.*"}
              ) > 0.9
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "High memory usage in {{ $labels.pod }}"
              description: "Memory usage is {{ $value | humanizePercentage }}"

          # High CPU Usage
          - alert: HighCPUUsage
            expr: |
              (
                sum(rate(container_cpu_usage_seconds_total{namespace=~"triptrip-.*"}[5m])) by (pod) /
                sum(container_spec_cpu_quota{namespace=~"triptrip-.*"}/container_spec_cpu_period{namespace=~"triptrip-.*"}) by (pod)
              ) > 0.9
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "High CPU usage in {{ $labels.pod }}"
              description: "CPU usage is {{ $value | humanizePercentage }}"

          # Pod Restart
          - alert: PodRestarting
            expr: |
              increase(kube_pod_container_status_restarts_total{namespace=~"triptrip-.*"}[1h]) > 3
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "Pod {{ $labels.pod }} is restarting frequently"
              description: "{{ $value }} restarts in the last hour"

          # Pending Pods
          - alert: PodPending
            expr: |
              kube_pod_status_phase{phase="Pending", namespace=~"triptrip-.*"} == 1
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "Pod {{ $labels.pod }} is pending"
              description: "Pod has been in Pending state for more than 5 minutes"

  infrastructure-alerts.yml: |
    groups:
      - name: infrastructure-alerts
        rules:
          # Node Not Ready
          - alert: NodeNotReady
            expr: kube_node_status_condition{condition="Ready",status="true"} == 0
            for: 5m
            labels:
              severity: critical
            annotations:
              summary: "Node {{ $labels.node }} is not ready"

          # High Node CPU
          - alert: NodeHighCPU
            expr: |
              100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
            for: 10m
            labels:
              severity: warning
            annotations:
              summary: "High CPU usage on node {{ $labels.instance }}"
              description: "CPU usage is {{ $value }}%"

          # High Node Memory
          - alert: NodeHighMemory
            expr: |
              (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 90
            for: 10m
            labels:
              severity: warning
            annotations:
              summary: "High memory usage on node {{ $labels.instance }}"

          # Node Disk Full
          - alert: NodeDiskFull
            expr: |
              (1 - node_filesystem_avail_bytes{mountpoint="/"} /
                   node_filesystem_size_bytes{mountpoint="/"}) * 100 > 85
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "Disk is filling up on {{ $labels.instance }}"
              description: "Disk usage is {{ $value }}%"

  database-alerts.yml: |
    groups:
      - name: database-alerts
        rules:
          # High Database Connections
          - alert: HighDatabaseConnections
            expr: triptrip_db_connections_active > 80
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "High database connection count"
              description: "Active connections: {{ $value }}"

          # Slow Database Queries
          - alert: SlowDatabaseQueries
            expr: |
              histogram_quantile(0.99,
                sum(rate(triptrip_db_query_duration_seconds_bucket[5m])) by (le)
              ) > 1
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "Slow database queries detected"
              description: "P99 query latency is {{ $value | humanizeDuration }}"

          # Low Cache Hit Rate
          - alert: LowCacheHitRate
            expr: |
              sum(rate(triptrip_cache_hits_total[5m])) /
              (sum(rate(triptrip_cache_hits_total[5m])) + sum(rate(triptrip_cache_misses_total[5m]))) < 0.8
            for: 10m
            labels:
              severity: warning
            annotations:
              summary: "Low cache hit rate"
              description: "Cache hit rate is {{ $value | humanizePercentage }}"
```

### 5.2 SLO/SLI定義

```yaml
slo_definitions:
  api_availability:
    name: API Availability
    objective: 99.95%
    window: 28 days

    sli:
      type: availability
      good_events: |
        sum(rate(http_requests_total{status_code!~"5.."}[5m]))
      total_events: |
        sum(rate(http_requests_total[5m]))

    error_budget:
      total: 0.05% (21.6 minutes per 28 days)
      calculation: |
        1 - (good_events / total_events)

    alerting:
      fast_burn:
        window: 1h
        burn_rate: 10x
        severity: critical

      slow_burn:
        window: 24h
        burn_rate: 2x
        severity: warning

  api_latency:
    name: API Latency
    objective: 99% of requests < 200ms
    window: 28 days

    sli:
      type: latency
      good_events: |
        sum(rate(http_request_duration_seconds_bucket{le="0.2"}[5m]))
      total_events: |
        sum(rate(http_request_duration_seconds_count[5m]))

    percentiles:
      p50: 50ms
      p90: 100ms
      p99: 200ms

  booking_success:
    name: Booking Success Rate
    objective: 99.9%
    window: 28 days

    sli:
      type: success_rate
      good_events: |
        sum(rate(triptrip_bookings_total{status="confirmed"}[5m]))
      total_events: |
        sum(rate(triptrip_bookings_total[5m]))

  search_latency:
    name: Search Latency
    objective: 95% of searches < 500ms
    window: 28 days

    sli:
      type: latency
      good_events: |
        sum(rate(triptrip_search_latency_seconds_bucket{le="0.5"}[5m]))
      total_events: |
        sum(rate(triptrip_search_latency_seconds_count[5m]))
```

### 5.3 エスカレーションポリシー

```yaml
escalation_policies:
  critical_incidents:
    name: Critical Production Incidents
    description: P1 incidents affecting production availability

    levels:
      - level: 1
        timeout: 5 minutes
        targets:
          - type: on_call
            schedule: primary-oncall
          - type: slack
            channel: "#incidents-critical"

      - level: 2
        timeout: 15 minutes
        targets:
          - type: on_call
            schedule: secondary-oncall
          - type: phone
            targets: ["on-call-manager"]

      - level: 3
        timeout: 30 minutes
        targets:
          - type: team
            team: engineering-leads
          - type: phone
            targets: ["vp-engineering", "cto"]

  warning_incidents:
    name: Warning Level Incidents
    description: P2/P3 incidents requiring attention

    levels:
      - level: 1
        timeout: 30 minutes
        targets:
          - type: slack
            channel: "#alerts-warning"

      - level: 2
        timeout: 2 hours
        targets:
          - type: on_call
            schedule: primary-oncall

  on_call_schedules:
    primary_oncall:
      name: Primary On-Call
      rotation: weekly
      handoff_time: "09:00 JST Monday"
      participants:
        - sre-team

    secondary_oncall:
      name: Secondary On-Call
      rotation: weekly
      handoff_time: "09:00 JST Monday"
      participants:
        - platform-team

  incident_severity:
    p1_critical:
      description: Complete service outage or data loss
      response_time: 5 minutes
      resolution_target: 1 hour
      communication: Every 30 minutes

    p2_high:
      description: Major feature degradation
      response_time: 30 minutes
      resolution_target: 4 hours
      communication: Every 2 hours

    p3_medium:
      description: Minor feature impact
      response_time: 4 hours
      resolution_target: 24 hours
      communication: Daily

    p4_low:
      description: Cosmetic issues or minor bugs
      response_time: 24 hours
      resolution_target: 1 week
      communication: Weekly
```

---

## 第6章：実装ロードマップ & 文書間参照

### 6.1 フェーズ別実装計画

```yaml
implementation_roadmap:
  phase_1_foundation:
    timeline: Month 1-2
    objectives:
      - 基本的なメトリクス収集
      - ログ集約パイプライン
      - 基本アラート設定

    deliverables:
      - Prometheus + Grafana デプロイ
      - Fluent Bit DaemonSet
      - CloudWatch Logs 統合
      - 基本ダッシュボード
      - Critical アラートルール

    success_criteria:
      - 全サービスからメトリクス収集
      - ログの集中管理
      - Critical アラートの通知

  phase_2_observability:
    timeline: Month 2-3
    objectives:
      - 分散トレーシング導入
      - カスタムメトリクス実装
      - 詳細ダッシュボード

    deliverables:
      - OpenTelemetry + Jaeger デプロイ
      - アプリケーショントレーシング実装
      - ビジネスメトリクス実装
      - サービス別ダッシュボード

    success_criteria:
      - エンドツーエンドトレース可視化
      - カスタムメトリクス収集
      - 問題の root cause 特定が可能

  phase_3_slo:
    timeline: Month 3-4
    objectives:
      - SLO/SLI 定義と実装
      - Error Budget 管理
      - 高度なアラート

    deliverables:
      - SLO ダッシュボード
      - Error Budget アラート
      - Burn rate アラート
      - SLO レポート自動化

    success_criteria:
      - SLO 達成状況の可視化
      - Error Budget の追跡
      - Proactive なアラート

  phase_4_optimization:
    timeline: Month 4-6
    objectives:
      - パフォーマンス最適化
      - コスト最適化
      - 自動化強化

    deliverables:
      - ログサンプリング最適化
      - 長期ストレージ戦略
      - 自動スケーリング連携
      - Runbook 自動化

    success_criteria:
      - オブザーバビリティコスト最適化
      - MTTR 短縮
      - 自動復旧の実現

  phase_5_advanced:
    timeline: Month 6-12
    objectives:
      - AIOps 導入
      - 予測分析
      - カオスエンジニアリング連携

    deliverables:
      - 異常検知システム
      - 容量予測
      - Chaos Mesh 統合
      - インシデント分析 AI

    success_criteria:
      - 予兆検知の実現
      - 障害予防
      - 自動診断
```

### 6.2 関連文書マッピング

```yaml
document_references:
  inputs:
    - document_id: Doc-IA-001
      title: Cloud Infrastructure & Deployment
      relevance: 基盤インフラストラクチャ
      integration_points:
        - CloudWatch 設定
        - S3 ストレージ
        - IAM 権限

    - document_id: Doc-IA-002
      title: Kubernetes & Container Strategy
      relevance: コンテナモニタリング
      integration_points:
        - Pod メトリクス
        - DaemonSet 配置
        - Service Discovery

    - document_id: Doc-IA-003
      title: Networking & CDN Architecture
      relevance: ネットワークモニタリング
      integration_points:
        - ALB メトリクス
        - CloudFront ログ
        - VPC Flow Logs

  outputs:
    - document_id: Doc-SC-001
      title: Security Architecture
      dependency: セキュリティ監視

    - document_id: Doc-IA-005
      title: Disaster Recovery & High Availability
      dependency: 可用性監視

    - document_id: Doc-DO-001
      title: DevOps Strategy
      dependency: CI/CD 監視

  cross_references:
    - document_id: Doc-QA-001
      title: Quality Assurance
      relationship: テスト監視

    - document_id: Doc-SA-001
      title: System Architecture Overview
      relationship: アーキテクチャ可視化
```

### 6.3 オブザーバビリティ設計サマリー

```yaml
observability_summary:
  metrics:
    collection: Prometheus
    storage: Thanos (S3)
    visualization: Grafana
    retention:
      high_resolution: 15 days
      aggregated: 1 year

  logs:
    collection: Fluent Bit
    storage:
      hot: OpenSearch (7 days)
      warm: CloudWatch (30 days)
      cold: S3 Glacier (1 year)
    analysis: Kibana / CloudWatch Insights

  traces:
    instrumentation: OpenTelemetry
    collection: OTel Collector
    backend: Jaeger + AWS X-Ray
    sampling: Tail-based (10% + 100% errors)

  alerting:
    engine: Alertmanager
    integration:
      - PagerDuty (Critical)
      - Slack (All levels)
    escalation: 3-tier

  slo:
    availability_target: 99.95%
    latency_target: P99 < 200ms
    error_budget_window: 28 days
```

---

## 結論

本文書は、TripTripプラットフォームの包括的なモニタリング、ロギング、オブザーバビリティ戦略を定義しました。

### 主要な設計決定

1. **Prometheus/Grafana**: 業界標準のメトリクス収集・可視化
2. **Fluent Bit + OpenSearch**: 効率的なログ収集と分析
3. **OpenTelemetry + Jaeger**: ベンダー中立な分散トレーシング
4. **SLO/SLI フレームワーク**: データ駆動型のサービス品質管理
5. **多層アラート**: Critical/Warning/Info の段階的エスカレーション

### 期待される成果

- MTTR (Mean Time To Recovery): 30分以下
- MTTD (Mean Time To Detect): 1分以下
- SLO 達成率: 99.95%
- アラート精度: False Positive < 5%
- オブザーバビリティカバレッジ: 100%

これらの設計により、TripTripはプロアクティブな運用、迅速な問題解決、データ駆動型の意思決定を実現します。

---

**文書情報**
- Document ID: Doc-IA-004
- Version: 1.0.0
- Last Updated: 2026-01-21
- Status: Draft
- Total Lines: 1,900+
- Author: Technical Architecture Agent

**関連文書**
- Doc-IA-001: Cloud Infrastructure & Deployment
- Doc-IA-002: Kubernetes & Container Strategy
- Doc-IA-003: Networking & CDN Architecture
- Doc-IA-005: Disaster Recovery & High Availability
- Doc-SC-001: Security Architecture

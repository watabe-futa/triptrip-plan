# Doc-DA-003: TripTripデータパイプライン & ETLアーキテクチャ

## エグゼクティブサマリー

本文書は、TripTripプラットフォームのデータパイプラインおよびETL（Extract, Transform, Load）アーキテクチャを包括的に定義します。リアルタイムストリーム処理とバッチ処理を統合したハイブリッドアーキテクチャを設計し、運用データベースからデータウェアハウス、分析基盤へのデータフローを最適化します。Apache Kafka、Apache Flink、Apache Sparkを中核として、Change Data Capture（CDC）、データ品質管理、データリネージ追跡を含む包括的なデータパイプライン基盤を構築します。本設計は、リアルタイム分析、機械学習パイプライン、ビジネスインテリジェンスを支える堅牢なデータ基盤を実現します。

---

## 第1章：はじめに & コンテキスト

### 1.1 ビジネス要件 & 制約

#### 1.1.1 データパイプライン要件

```yaml
pipeline_requirements:
  real_time_analytics:
    description: リアルタイム分析・ダッシュボード
    latency_requirement: < 10秒
    use_cases:
      - リアルタイム予約状況モニタリング
      - 動的価格設定
      - 不正検知
      - リアルタイム在庫更新
    data_sources:
      - 予約イベント
      - 決済イベント
      - ユーザーアクティビティ
      - 在庫変更イベント

  batch_analytics:
    description: バッチ分析・レポーティング
    processing_window: 日次/週次/月次
    use_cases:
      - 財務レポート
      - パートナー精算
      - コホート分析
      - マーケティング分析
    data_sources:
      - 全トランザクションデータ
      - ユーザー行動ログ
      - パートナーデータ

  ml_pipelines:
    description: 機械学習パイプライン
    update_frequency: 日次〜週次
    use_cases:
      - 価格最適化モデル
      - 推奨エンジン
      - 需要予測
      - チャーン予測
    data_requirements:
      - 特徴量エンジニアリング
      - モデル学習データ
      - 推論用特徴量

  data_synchronization:
    description: システム間データ同期
    latency_requirement: < 1分
    use_cases:
      - 検索インデックス更新
      - キャッシュ更新
      - マイクロサービス間同期
      - 外部システム連携
```

#### 1.1.2 データボリューム予測

```yaml
data_volume_projection:
  event_ingestion:
    year_1:
      events_per_day: 10,000,000
      peak_events_per_second: 500
      daily_data_volume: 5GB

    year_3:
      events_per_day: 500,000,000
      peak_events_per_second: 20,000
      daily_data_volume: 250GB

    year_5:
      events_per_day: 5,000,000,000
      peak_events_per_second: 200,000
      daily_data_volume: 2.5TB

  batch_processing:
    year_1:
      daily_processing_volume: 50GB
      processing_window: 4時間

    year_3:
      daily_processing_volume: 500GB
      processing_window: 4時間

    year_5:
      daily_processing_volume: 5TB
      processing_window: 4時間

  data_warehouse:
    year_1:
      total_size: 500GB
      daily_growth: 2GB

    year_3:
      total_size: 20TB
      daily_growth: 50GB

    year_5:
      total_size: 500TB
      daily_growth: 500GB
```

### 1.2 技術目標 & 成功基準

#### 1.2.1 パフォーマンス目標

```yaml
performance_targets:
  real_time_pipeline:
    end_to_end_latency:
      p50: 2秒
      p95: 5秒
      p99: 10秒
    throughput:
      sustained: 50,000 events/sec
      burst: 200,000 events/sec
    availability: 99.9%

  batch_pipeline:
    processing_time:
      daily_batch: < 4時間
      weekly_batch: < 8時間
      monthly_batch: < 24時間
    data_freshness:
      operational_reports: T+1日
      financial_reports: T+2日
    success_rate: 99.9%

  data_quality:
    completeness: 99.99%
    accuracy: 99.99%
    timeliness: SLA達成率99%
    consistency: クロスシステム整合性99.9%
```

#### 1.2.2 運用目標

```yaml
operational_targets:
  reliability:
    pipeline_uptime: 99.9%
    data_loss_tolerance: 0
    exactly_once_delivery: 必須

  observability:
    end_to_end_tracing: 100%
    data_lineage_coverage: 100%
    alerting_latency: < 1分

  maintainability:
    deployment_frequency: 週次
    rollback_time: < 5分
    schema_evolution: ダウンタイムなし
```

### 1.3 主要な前提条件 & トレードオフ

#### 1.3.1 アーキテクチャ原則

```yaml
architecture_principles:
  1_lambda_architecture:
    description: バッチとストリームのハイブリッドアーキテクチャ
    rationale:
      - リアルタイム性と正確性の両立
      - 障害時のリカバリ容易性
      - 柔軟なクエリパターン対応

  2_schema_on_read:
    description: 読み取り時スキーマ適用
    rationale:
      - データレイクの柔軟性
      - スキーマ進化の容易性
      - 異種データの統合

  3_idempotent_processing:
    description: べき等処理の保証
    rationale:
      - exactly-once配信の実現
      - 再処理の安全性
      - 障害復旧の簡素化

  4_data_mesh_ready:
    description: データメッシュへの発展準備
    rationale:
      - ドメイン別データ所有権
      - セルフサービスデータプラットフォーム
      - 連合データガバナンス
```

---

## 第2章：データインジェスチョン設計

### 2.1 リアルタイムインジェスチョン（Kafka）

#### 2.1.1 Kafkaアーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      REAL-TIME INGESTION ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         Data Producers                               │   │
│  │                                                                      │   │
│  │   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐              │   │
│  │   │ API     │  │ Mobile  │  │ Batch   │  │ CDC     │              │   │
│  │   │ Gateway │  │ Apps    │  │ Jobs    │  │ Source  │              │   │
│  │   └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘              │   │
│  │        │            │            │            │                    │   │
│  └────────┼────────────┼────────────┼────────────┼────────────────────┘   │
│           │            │            │            │                         │
│           └────────────┼────────────┼────────────┘                         │
│                        │            │                                       │
│                        ▼            ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                       Schema Registry                                │   │
│  │                    (Confluent Schema Registry)                       │   │
│  │                                                                      │   │
│  │   • Avro/Protobuf スキーマ管理                                       │   │
│  │   • スキーマバージョニング                                           │   │
│  │   • 互換性チェック                                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                       Kafka Cluster                                  │   │
│  │                                                                      │   │
│  │   ┌─────────────────────────────────────────────────────────────┐   │   │
│  │   │                         Topics                               │   │   │
│  │   │                                                              │   │   │
│  │   │   ┌────────────┐  ┌────────────┐  ┌────────────┐           │   │   │
│  │   │   │ bookings   │  │  orders    │  │  users     │           │   │   │
│  │   │   │ (P=12)     │  │  (P=12)    │  │  (P=6)     │           │   │   │
│  │   │   └────────────┘  └────────────┘  └────────────┘           │   │   │
│  │   │                                                              │   │   │
│  │   │   ┌────────────┐  ┌────────────┐  ┌────────────┐           │   │   │
│  │   │   │ activities │  │ inventory  │  │  payments  │           │   │   │
│  │   │   │ (P=24)     │  │  (P=12)    │  │  (P=6)     │           │   │   │
│  │   │   └────────────┘  └────────────┘  └────────────┘           │   │   │
│  │   └─────────────────────────────────────────────────────────────┘   │   │
│  │                                                                      │   │
│  │   Brokers: 6 (3 zones × 2)                                          │   │
│  │   Replication Factor: 3                                              │   │
│  │   Min ISR: 2                                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                       Data Consumers                                 │   │
│  │                                                                      │   │
│  │   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐              │   │
│  │   │ Flink   │  │ Search  │  │ Cache   │  │ Archive │              │   │
│  │   │ Jobs    │  │ Indexer │  │ Updater │  │ (S3)    │              │   │
│  │   └─────────┘  └─────────┘  └─────────┘  └─────────┘              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 2.1.2 Kafka構成詳細

```yaml
kafka_configuration:
  cluster:
    provider: Confluent Cloud
    region: asia-northeast1
    availability_zones: 3

  brokers:
    count: 6
    instance_type: dedicated
    storage_per_broker: 2TB
    retention_bytes: 1TB

  topics:
    bookings:
      partitions: 12
      replication_factor: 3
      retention_ms: 604800000  # 7日
      cleanup_policy: delete
      key: booking_id
      value_schema: avro

    orders:
      partitions: 12
      replication_factor: 3
      retention_ms: 604800000
      cleanup_policy: delete
      key: order_id
      value_schema: avro

    user_activities:
      partitions: 24
      replication_factor: 3
      retention_ms: 259200000  # 3日
      cleanup_policy: delete
      key: user_id
      value_schema: avro

    inventory_changes:
      partitions: 12
      replication_factor: 3
      retention_ms: 86400000  # 1日
      cleanup_policy: compact
      key: entity_id
      value_schema: avro

    cdc_events:
      partitions: 24
      replication_factor: 3
      retention_ms: 172800000  # 2日
      cleanup_policy: delete
      key: table_name:primary_key
      value_schema: debezium

  producer_config:
    acks: all
    retries: 3
    retry_backoff_ms: 100
    batch_size: 16384
    linger_ms: 5
    compression_type: snappy
    enable_idempotence: true

  consumer_config:
    enable_auto_commit: false
    auto_offset_reset: earliest
    max_poll_records: 500
    fetch_min_bytes: 1024
    fetch_max_wait_ms: 500
```

#### 2.1.3 イベントスキーマ定義

```avro
// bookings.avsc
{
  "type": "record",
  "name": "BookingEvent",
  "namespace": "com.triptrip.events",
  "fields": [
    {
      "name": "event_id",
      "type": "string",
      "doc": "イベントの一意識別子（UUID）"
    },
    {
      "name": "event_type",
      "type": {
        "type": "enum",
        "name": "BookingEventType",
        "symbols": ["CREATED", "CONFIRMED", "CANCELLED", "COMPLETED", "MODIFIED"]
      }
    },
    {
      "name": "event_timestamp",
      "type": "long",
      "logicalType": "timestamp-millis"
    },
    {
      "name": "booking_id",
      "type": "string"
    },
    {
      "name": "user_id",
      "type": "string"
    },
    {
      "name": "booking_type",
      "type": {
        "type": "enum",
        "name": "BookingType",
        "symbols": ["HOTEL", "ATTRACTION", "RESTAURANT", "RENTAL", "TOUR"]
      }
    },
    {
      "name": "entity_id",
      "type": "string",
      "doc": "予約対象のエンティティID（ホテルID、アトラクションID等）"
    },
    {
      "name": "booking_date",
      "type": "int",
      "logicalType": "date"
    },
    {
      "name": "total_amount",
      "type": {
        "type": "bytes",
        "logicalType": "decimal",
        "precision": 12,
        "scale": 2
      }
    },
    {
      "name": "currency",
      "type": "string",
      "default": "JPY"
    },
    {
      "name": "guest_count",
      "type": "int",
      "default": 1
    },
    {
      "name": "metadata",
      "type": ["null", {"type": "map", "values": "string"}],
      "default": null
    },
    {
      "name": "source",
      "type": "string",
      "doc": "イベント発生元（web, ios, android, api）"
    },
    {
      "name": "trace_id",
      "type": ["null", "string"],
      "default": null,
      "doc": "分散トレーシング用ID"
    }
  ]
}
```

### 2.2 バッチインジェスチョン（S3/GCS）

#### 2.2.1 バッチデータソース

```yaml
batch_data_sources:
  external_partner_data:
    description: パートナーからの日次データ
    format: CSV, JSON, Excel
    frequency: 日次
    landing_zone: gs://triptrip-data-lake/landing/partners/
    sources:
      - hotel_inventory_updates
      - attraction_schedule_updates
      - pricing_updates

  third_party_data:
    description: 外部データプロバイダー
    format: JSON, Parquet
    frequency: 日次/週次
    landing_zone: gs://triptrip-data-lake/landing/external/
    sources:
      - weather_data
      - event_calendar
      - exchange_rates
      - competitor_pricing

  historical_backfill:
    description: 履歴データの再取り込み
    format: Parquet
    landing_zone: gs://triptrip-data-lake/landing/backfill/
    use_cases:
      - 新規データモデルへの移行
      - データ品質修正後の再処理
      - 新規分析要件への対応

  database_exports:
    description: 運用DBからのエクスポート
    format: Parquet
    frequency: 日次
    landing_zone: gs://triptrip-data-lake/landing/db_exports/
    sources:
      - full_table_exports
      - incremental_exports
```

#### 2.2.2 データレイク構成

```yaml
data_lake_structure:
  bucket: gs://triptrip-data-lake

  zones:
    landing:
      path: /landing/
      description: 生データの着地点
      retention: 30日
      access: 書き込み制限（プロデューサーのみ）

    raw:
      path: /raw/
      description: 正規化前のクレンジング済みデータ
      format: Parquet
      partitioning: year/month/day
      retention: 2年
      access: データエンジニア

    curated:
      path: /curated/
      description: 品質検証済み、正規化済みデータ
      format: Parquet/Delta Lake
      partitioning: domain/entity/year/month/day
      retention: 7年
      access: データサイエンティスト、アナリスト

    aggregated:
      path: /aggregated/
      description: 事前集計済みデータ
      format: Parquet
      partitioning: domain/metric/granularity
      retention: 3年
      access: BIツール、ダッシュボード

    ml_features:
      path: /ml/features/
      description: 機械学習用特徴量
      format: Parquet
      retention: 1年
      access: MLエンジニア

  file_naming_convention:
    pattern: "{domain}/{entity}/{year}/{month}/{day}/{entity}_{timestamp}_{uuid}.parquet"
    example: "bookings/hotel_bookings/2026/01/19/hotel_bookings_20260119_120000_abc123.parquet"
```

### 2.3 CDC（Change Data Capture）実装

#### 2.3.1 Debezium構成

```yaml
debezium_configuration:
  connector_type: Debezium PostgreSQL Connector
  deployment: Kafka Connect (Confluent Cloud)

  source_database:
    hostname: primary-db.triptrip.internal
    port: 5432
    database: triptrip
    user: debezium
    schema_include_list: public

  capture_settings:
    snapshot_mode: initial
    slot_name: debezium_triptrip
    publication_name: triptrip_publication
    decimal_handling_mode: precise
    time_precision_mode: connect
    tombstones_on_delete: true

  tables_to_capture:
    - table: users
      topic: triptrip.cdc.users
      key: id
      transforms: route

    - table: bookings
      topic: triptrip.cdc.bookings
      key: id
      transforms: route

    - table: orders
      topic: triptrip.cdc.orders
      key: id
      transforms: route

    - table: products
      topic: triptrip.cdc.products
      key: id
      transforms: route

    - table: inventory
      topic: triptrip.cdc.inventory
      key: id
      transforms: route

  transforms:
    route:
      type: io.debezium.transforms.ByLogicalTableRouter
      topic_regex: (.*)
      topic_replacement: triptrip.cdc.$1

    flatten:
      type: io.debezium.transforms.ExtractNewRecordState
      drop_tombstones: false
      delete_handling_mode: rewrite
```

#### 2.3.2 CDCイベント処理

```typescript
// cdc/cdc-processor.ts
import { Kafka, Consumer, EachMessagePayload } from 'kafkajs';

interface DebeziumPayload {
  before: Record<string, any> | null;
  after: Record<string, any> | null;
  source: {
    version: string;
    connector: string;
    name: string;
    ts_ms: number;
    snapshot: string;
    db: string;
    schema: string;
    table: string;
    txId: number;
    lsn: number;
  };
  op: 'c' | 'u' | 'd' | 'r';  // create, update, delete, read (snapshot)
  ts_ms: number;
  transaction: {
    id: string;
    total_order: number;
    data_collection_order: number;
  } | null;
}

interface CDCHandler {
  onCreate(table: string, data: Record<string, any>): Promise<void>;
  onUpdate(table: string, before: Record<string, any>, after: Record<string, any>): Promise<void>;
  onDelete(table: string, data: Record<string, any>): Promise<void>;
}

class CDCProcessor {
  private consumer: Consumer;
  private handlers: Map<string, CDCHandler> = new Map();

  constructor(kafka: Kafka, groupId: string) {
    this.consumer = kafka.consumer({ groupId });
  }

  registerHandler(table: string, handler: CDCHandler): void {
    this.handlers.set(table, handler);
  }

  async start(topics: string[]): Promise<void> {
    await this.consumer.connect();
    await this.consumer.subscribe({ topics, fromBeginning: false });

    await this.consumer.run({
      eachMessage: async (payload: EachMessagePayload) => {
        await this.processMessage(payload);
      }
    });
  }

  private async processMessage(payload: EachMessagePayload): Promise<void> {
    const { topic, partition, message } = payload;

    if (!message.value) return;

    const debeziumEvent: DebeziumPayload = JSON.parse(message.value.toString());
    const table = debeziumEvent.source.table;
    const handler = this.handlers.get(table);

    if (!handler) {
      console.warn(`No handler registered for table: ${table}`);
      return;
    }

    try {
      switch (debeziumEvent.op) {
        case 'c':  // Create
        case 'r':  // Read (snapshot)
          if (debeziumEvent.after) {
            await handler.onCreate(table, debeziumEvent.after);
          }
          break;

        case 'u':  // Update
          if (debeziumEvent.before && debeziumEvent.after) {
            await handler.onUpdate(table, debeziumEvent.before, debeziumEvent.after);
          }
          break;

        case 'd':  // Delete
          if (debeziumEvent.before) {
            await handler.onDelete(table, debeziumEvent.before);
          }
          break;
      }
    } catch (error) {
      console.error(`Error processing CDC event for ${table}:`, error);
      // デッドレターキューに送信
      await this.sendToDeadLetterQueue(topic, message, error);
    }
  }

  private async sendToDeadLetterQueue(
    originalTopic: string,
    message: any,
    error: any
  ): Promise<void> {
    // DLQ実装
  }
}

// 使用例: 検索インデックス更新
class SearchIndexCDCHandler implements CDCHandler {
  async onCreate(table: string, data: Record<string, any>): Promise<void> {
    await this.indexDocument(table, data);
  }

  async onUpdate(table: string, before: Record<string, any>, after: Record<string, any>): Promise<void> {
    await this.updateDocument(table, after);
  }

  async onDelete(table: string, data: Record<string, any>): Promise<void> {
    await this.deleteDocument(table, data.id);
  }

  private async indexDocument(table: string, data: Record<string, any>): Promise<void> {
    // Elasticsearchへのインデックス
  }

  private async updateDocument(table: string, data: Record<string, any>): Promise<void> {
    // Elasticsearchのドキュメント更新
  }

  private async deleteDocument(table: string, id: string): Promise<void> {
    // Elasticsearchからのドキュメント削除
  }
}
```

---

## 第3章：データ変換パイプライン

### 3.1 ストリーム処理（Apache Flink）

#### 3.1.1 Flinkアーキテクチャ

```yaml
flink_architecture:
  deployment:
    mode: Kubernetes (GKE)
    cluster_type: Session Cluster
    high_availability: ZooKeeper

  resource_allocation:
    job_manager:
      replicas: 2
      memory: 4GB
      cpu: 2

    task_manager:
      replicas: 8
      memory: 16GB
      cpu: 4
      slots_per_task_manager: 4

  checkpointing:
    interval: 60000  # 1分
    mode: EXACTLY_ONCE
    timeout: 600000  # 10分
    min_pause: 30000
    max_concurrent: 1
    storage: gs://triptrip-flink/checkpoints/

  state_backend:
    type: RocksDB
    incremental: true
    local_recovery: true

  restart_strategy:
    type: exponential-delay
    initial_backoff: 1s
    max_backoff: 60s
    backoff_multiplier: 2
    reset_backoff_threshold: 1h
```

#### 3.1.2 ストリーム処理ジョブ

```java
// flink/jobs/BookingAnalyticsJob.java
public class BookingAnalyticsJob {

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // Checkpointing設定
        env.enableCheckpointing(60000, CheckpointingMode.EXACTLY_ONCE);
        env.getCheckpointConfig().setCheckpointStorage("gs://triptrip-flink/checkpoints/");

        // Kafka Source
        KafkaSource<BookingEvent> bookingSource = KafkaSource.<BookingEvent>builder()
            .setBootstrapServers("kafka-cluster:9092")
            .setTopics("triptrip.bookings")
            .setGroupId("flink-booking-analytics")
            .setStartingOffsets(OffsetsInitializer.latest())
            .setValueOnlyDeserializer(new BookingEventDeserializer())
            .build();

        DataStream<BookingEvent> bookingStream = env.fromSource(
            bookingSource,
            WatermarkStrategy.<BookingEvent>forBoundedOutOfOrderness(Duration.ofSeconds(10))
                .withTimestampAssigner((event, timestamp) -> event.getEventTimestamp()),
            "Booking Source"
        );

        // リアルタイム予約メトリクス計算
        DataStream<BookingMetrics> metricsStream = bookingStream
            .filter(event -> event.getEventType() == BookingEventType.CONFIRMED)
            .keyBy(BookingEvent::getBookingType)
            .window(TumblingEventTimeWindows.of(Time.minutes(1)))
            .aggregate(new BookingMetricsAggregator());

        // Redis Sink（リアルタイムダッシュボード用）
        metricsStream.addSink(new RedisSink<>(redisConfig, new BookingMetricsRedisMapper()));

        // BigQuery Sink（分析用）
        metricsStream.addSink(BigQuerySink.<BookingMetrics>builder()
            .setProject("triptrip-prod")
            .setDataset("realtime_metrics")
            .setTable("booking_metrics")
            .setSerializer(new BookingMetricsSerializer())
            .build());

        // 不正検知ストリーム
        DataStream<FraudAlert> fraudAlerts = bookingStream
            .keyBy(BookingEvent::getUserId)
            .process(new FraudDetectionProcessFunction());

        fraudAlerts.addSink(new AlertSink());

        env.execute("Booking Analytics Job");
    }
}

// 集計関数
public class BookingMetricsAggregator
    implements AggregateFunction<BookingEvent, BookingMetricsAccumulator, BookingMetrics> {

    @Override
    public BookingMetricsAccumulator createAccumulator() {
        return new BookingMetricsAccumulator();
    }

    @Override
    public BookingMetricsAccumulator add(BookingEvent event, BookingMetricsAccumulator acc) {
        acc.bookingCount++;
        acc.totalRevenue = acc.totalRevenue.add(event.getTotalAmount());
        acc.guestCount += event.getGuestCount();
        return acc;
    }

    @Override
    public BookingMetrics getResult(BookingMetricsAccumulator acc) {
        return new BookingMetrics(
            acc.bookingType,
            acc.windowStart,
            acc.windowEnd,
            acc.bookingCount,
            acc.totalRevenue,
            acc.guestCount
        );
    }

    @Override
    public BookingMetricsAccumulator merge(BookingMetricsAccumulator a, BookingMetricsAccumulator b) {
        a.bookingCount += b.bookingCount;
        a.totalRevenue = a.totalRevenue.add(b.totalRevenue);
        a.guestCount += b.guestCount;
        return a;
    }
}

// 不正検知プロセス関数
public class FraudDetectionProcessFunction
    extends KeyedProcessFunction<String, BookingEvent, FraudAlert> {

    private ValueState<Integer> recentBookingsCount;
    private ValueState<Long> lastBookingTime;

    @Override
    public void open(Configuration parameters) {
        recentBookingsCount = getRuntimeContext().getState(
            new ValueStateDescriptor<>("recentBookings", Integer.class));
        lastBookingTime = getRuntimeContext().getState(
            new ValueStateDescriptor<>("lastBookingTime", Long.class));
    }

    @Override
    public void processElement(BookingEvent event, Context ctx, Collector<FraudAlert> out) throws Exception {
        Integer count = recentBookingsCount.value();
        Long lastTime = lastBookingTime.value();

        if (count == null) count = 0;
        if (lastTime == null) lastTime = 0L;

        long currentTime = event.getEventTimestamp();

        // 5分以内に5件以上の予約は不正の可能性
        if (currentTime - lastTime < 300000 && count >= 5) {
            out.collect(new FraudAlert(
                event.getUserId(),
                "RAPID_BOOKING",
                "5 or more bookings within 5 minutes",
                event.getEventTimestamp()
            ));
        }

        // 1時間後にカウントをリセットするタイマー
        ctx.timerService().registerEventTimeTimer(currentTime + 3600000);

        recentBookingsCount.update(count + 1);
        lastBookingTime.update(currentTime);
    }

    @Override
    public void onTimer(long timestamp, OnTimerContext ctx, Collector<FraudAlert> out) {
        recentBookingsCount.clear();
    }
}
```

### 3.2 バッチ処理（Apache Spark）

#### 3.2.1 Sparkジョブ構成

```yaml
spark_configuration:
  deployment:
    mode: Dataproc (GCP)
    cluster_type: Standard

  cluster:
    master:
      machine_type: n2-standard-8
      boot_disk_size_gb: 500

    workers:
      machine_type: n2-standard-16
      count: 4
      boot_disk_size_gb: 1000

    preemptible_workers:
      count: 8
      percentage: 50

  spark_properties:
    spark.sql.adaptive.enabled: "true"
    spark.sql.adaptive.coalescePartitions.enabled: "true"
    spark.dynamicAllocation.enabled: "true"
    spark.dynamicAllocation.minExecutors: "2"
    spark.dynamicAllocation.maxExecutors: "50"
    spark.sql.shuffle.partitions: "200"
    spark.default.parallelism: "200"
```

#### 3.2.2 ETLジョブ実装

```python
# spark/jobs/daily_etl.py
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *
from delta import *

def main():
    spark = SparkSession.builder \
        .appName("TripTrip Daily ETL") \
        .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
        .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
        .getOrCreate()

    # 処理日付
    processing_date = spark.conf.get("spark.triptrip.processing_date")

    # 予約データETL
    process_bookings(spark, processing_date)

    # 注文データETL
    process_orders(spark, processing_date)

    # ユーザーアクティビティETL
    process_user_activities(spark, processing_date)

    # データ品質チェック
    run_data_quality_checks(spark, processing_date)

    spark.stop()


def process_bookings(spark: SparkSession, date: str):
    """予約データのETL処理"""

    # CDCイベントから読み込み
    raw_bookings = spark.read \
        .format("parquet") \
        .load(f"gs://triptrip-data-lake/raw/cdc/bookings/date={date}/")

    # 変換処理
    transformed_bookings = raw_bookings \
        .filter(col("op").isin(["c", "u"])) \
        .select(
            col("after.id").alias("booking_id"),
            col("after.booking_number"),
            col("after.user_id"),
            col("after.booking_type"),
            col("after.status"),
            col("after.booking_date"),
            col("after.total_amount").cast(DecimalType(12, 2)),
            col("after.currency"),
            col("after.guest_count"),
            col("after.created_at").cast(TimestampType()),
            col("after.updated_at").cast(TimestampType()),
            col("source.ts_ms").alias("event_timestamp"),
            lit(date).alias("processing_date")
        ) \
        .withColumn("revenue_jpy",
            when(col("currency") == "JPY", col("total_amount"))
            .otherwise(col("total_amount") * get_exchange_rate(col("currency"), lit("JPY")))
        ) \
        .withColumn("booking_month", date_format(col("booking_date"), "yyyy-MM"))

    # 重複排除（最新レコードのみ保持）
    deduplicated = transformed_bookings \
        .withColumn("row_num", row_number().over(
            Window.partitionBy("booking_id").orderBy(col("event_timestamp").desc())
        )) \
        .filter(col("row_num") == 1) \
        .drop("row_num")

    # Delta Lakeに書き込み（Upsert）
    deltaTable = DeltaTable.forPath(spark, "gs://triptrip-data-lake/curated/bookings/")

    deltaTable.alias("target").merge(
        deduplicated.alias("source"),
        "target.booking_id = source.booking_id"
    ).whenMatchedUpdateAll() \
     .whenNotMatchedInsertAll() \
     .execute()

    # 統計情報の記録
    record_count = deduplicated.count()
    log_processing_stats("bookings", date, record_count)


def process_orders(spark: SparkSession, date: str):
    """注文データのETL処理"""

    # 注文ヘッダー
    raw_orders = spark.read \
        .format("parquet") \
        .load(f"gs://triptrip-data-lake/raw/cdc/orders/date={date}/")

    # 注文明細
    raw_order_items = spark.read \
        .format("parquet") \
        .load(f"gs://triptrip-data-lake/raw/cdc/order_items/date={date}/")

    # 変換と結合
    orders = raw_orders \
        .filter(col("op").isin(["c", "u"])) \
        .select(
            col("after.id").alias("order_id"),
            col("after.order_number"),
            col("after.user_id"),
            col("after.status"),
            col("after.total_amount").cast(DecimalType(12, 2)),
            col("after.currency"),
            col("after.ordered_at").cast(TimestampType()),
            col("source.ts_ms").alias("event_timestamp")
        )

    order_items = raw_order_items \
        .filter(col("op").isin(["c", "u"])) \
        .select(
            col("after.id").alias("item_id"),
            col("after.order_id"),
            col("after.product_id"),
            col("after.quantity"),
            col("after.unit_price").cast(DecimalType(10, 2)),
            col("after.total_amount").cast(DecimalType(12, 2))
        )

    # 注文サマリーの計算
    order_summary = orders.join(
        order_items.groupBy("order_id").agg(
            count("item_id").alias("item_count"),
            sum("quantity").alias("total_quantity")
        ),
        "order_id",
        "left"
    )

    # 書き込み
    order_summary.write \
        .format("delta") \
        .mode("overwrite") \
        .partitionBy("status") \
        .option("mergeSchema", "true") \
        .save(f"gs://triptrip-data-lake/curated/orders/date={date}/")


def process_user_activities(spark: SparkSession, date: str):
    """ユーザーアクティビティのETL処理"""

    raw_activities = spark.read \
        .format("parquet") \
        .load(f"gs://triptrip-data-lake/raw/activities/date={date}/")

    # セッション化
    sessionized = raw_activities \
        .withColumn("prev_timestamp",
            lag("created_at").over(
                Window.partitionBy("user_id", "session_id").orderBy("created_at")
            )
        ) \
        .withColumn("time_diff",
            (col("created_at").cast("long") - col("prev_timestamp").cast("long"))
        ) \
        .withColumn("new_session",
            when(col("time_diff") > 1800, 1).otherwise(0)  # 30分でセッション区切り
        ) \
        .withColumn("session_number",
            sum("new_session").over(
                Window.partitionBy("user_id").orderBy("created_at")
            )
        )

    # セッションメトリクス
    session_metrics = sessionized.groupBy("user_id", "session_number").agg(
        min("created_at").alias("session_start"),
        max("created_at").alias("session_end"),
        count("*").alias("event_count"),
        countDistinct("activity_type").alias("unique_activities"),
        collect_list("activity_type").alias("activity_sequence")
    ).withColumn("session_duration_seconds",
        col("session_end").cast("long") - col("session_start").cast("long")
    )

    # 書き込み
    session_metrics.write \
        .format("delta") \
        .mode("overwrite") \
        .save(f"gs://triptrip-data-lake/curated/session_metrics/date={date}/")


def run_data_quality_checks(spark: SparkSession, date: str):
    """データ品質チェックの実行"""
    from great_expectations.dataset.sparkdf_dataset import SparkDFDataset

    # 予約データの品質チェック
    bookings = spark.read.format("delta").load("gs://triptrip-data-lake/curated/bookings/")
    bookings_ge = SparkDFDataset(bookings)

    # Null チェック
    assert bookings_ge.expect_column_values_to_not_be_null("booking_id").success
    assert bookings_ge.expect_column_values_to_not_be_null("user_id").success

    # 値の範囲チェック
    assert bookings_ge.expect_column_values_to_be_between(
        "total_amount", min_value=0, max_value=10000000
    ).success

    # ユニーク性チェック
    assert bookings_ge.expect_column_values_to_be_unique("booking_id").success

    print(f"Data quality checks passed for date: {date}")


if __name__ == "__main__":
    main()
```

### 3.3 データ品質とバリデーション

#### 3.3.1 データ品質フレームワーク

```yaml
data_quality_framework:
  tool: Great Expectations + dbt tests
  integration: Airflow / Cloud Composer

  quality_dimensions:
    completeness:
      description: 必須フィールドの存在確認
      checks:
        - not_null
        - not_empty

    accuracy:
      description: 値の正確性・妥当性
      checks:
        - range_check
        - regex_match
        - referential_integrity

    consistency:
      description: データ間の整合性
      checks:
        - cross_table_validation
        - business_rule_validation

    timeliness:
      description: データの鮮度
      checks:
        - freshness_check
        - sla_compliance

    uniqueness:
      description: 重複の検出
      checks:
        - unique_key
        - duplicate_detection

  expectations_suites:
    bookings:
      - expect_column_values_to_not_be_null:
          column: booking_id
      - expect_column_values_to_not_be_null:
          column: user_id
      - expect_column_values_to_be_in_set:
          column: status
          value_set: [PENDING, CONFIRMED, CANCELLED, COMPLETED, NO_SHOW, REFUNDED]
      - expect_column_values_to_be_between:
          column: total_amount
          min_value: 0
          max_value: 10000000
      - expect_column_values_to_be_unique:
          column: booking_id

    orders:
      - expect_column_values_to_not_be_null:
          column: order_id
      - expect_column_values_to_match_regex:
          column: order_number
          regex: "^TT[0-9]{8}[A-Z0-9]{6}$"
```

#### 3.3.2 品質アラートと対応

```yaml
quality_alerting:
  alert_channels:
    - slack: "#data-quality-alerts"
    - pagerduty: "data-team"
    - email: "data-alerts@triptrip.com"

  severity_levels:
    critical:
      description: データ処理を停止するレベルの問題
      examples:
        - 主キー重複
        - 必須フィールドのNull率 > 1%
        - 参照整合性エラー > 0.1%
      action: 即座に処理停止、オンコール通知

    warning:
      description: 調査が必要な問題
      examples:
        - データ量の異常（±50%）
        - 値の分布異常
        - 処理遅延
      action: Slackアラート、日次レビュー

    info:
      description: 情報提供レベル
      examples:
        - 新規カラム検出
        - スキーマ変更
      action: ログ記録のみ
```

---

## 第4章：データウェアハウス設計

### 4.1 スタースキーマ設計

#### 4.1.1 ファクトテーブル設計

```yaml
fact_tables:
  fact_bookings:
    description: 予約ファクトテーブル
    grain: 1予約 = 1行
    measures:
      - booking_count (implicit)
      - total_amount
      - tax_amount
      - discount_amount
      - guest_count
      - nights (for hotel bookings)
    dimensions:
      - dim_date (booking_date)
      - dim_time (booking_time)
      - dim_user
      - dim_booking_type
      - dim_entity (hotel, attraction, etc.)
      - dim_partner
      - dim_geography
      - dim_channel (web, ios, android)
    partitioning:
      column: booking_date
      type: DAY

  fact_orders:
    description: 注文ファクトテーブル
    grain: 1注文明細 = 1行
    measures:
      - order_count (implicit)
      - quantity
      - unit_price
      - line_total
      - tax_amount
      - discount_amount
    dimensions:
      - dim_date (order_date)
      - dim_time (order_time)
      - dim_user
      - dim_product
      - dim_category
      - dim_brand
      - dim_geography
      - dim_channel
    partitioning:
      column: order_date
      type: DAY

  fact_user_activity:
    description: ユーザーアクティビティファクト
    grain: 1イベント = 1行
    measures:
      - event_count (implicit)
      - session_duration_seconds
      - page_views
    dimensions:
      - dim_date
      - dim_time
      - dim_user
      - dim_activity_type
      - dim_page
      - dim_device
      - dim_geography
    partitioning:
      column: activity_date
      type: DAY

  fact_daily_inventory:
    description: 日次在庫スナップショット
    grain: 1エンティティ × 1日 = 1行
    measures:
      - available_inventory
      - reserved_inventory
      - sold_inventory
      - occupancy_rate
    dimensions:
      - dim_date
      - dim_entity
      - dim_entity_type
    partitioning:
      column: snapshot_date
      type: DAY
```

#### 4.1.2 ディメンションテーブル設計

```yaml
dimension_tables:
  dim_date:
    description: 日付ディメンション
    type: Type 0 (Static)
    attributes:
      - date_key (PK, YYYYMMDD)
      - full_date
      - year, quarter, month, week, day
      - day_of_week, day_name
      - is_weekend, is_holiday
      - fiscal_year, fiscal_quarter
      - season
    pre_populated: 2020-2030

  dim_time:
    description: 時刻ディメンション
    type: Type 0 (Static)
    attributes:
      - time_key (PK, HHMM)
      - hour, minute
      - hour_12, am_pm
      - time_band (morning, afternoon, evening, night)
    pre_populated: 00:00 - 23:59

  dim_user:
    description: ユーザーディメンション
    type: Type 2 (SCD)
    attributes:
      - user_key (PK, surrogate)
      - user_id (natural key)
      - email_domain
      - registration_date
      - user_tier (standard, premium, vip)
      - preferred_language
      - preferred_currency
      - country, region, city
      - age_group
      - gender
      - effective_from, effective_to
      - is_current
    scd_tracked_columns:
      - user_tier
      - preferred_language
      - preferred_currency
      - country

  dim_product:
    description: 商品ディメンション
    type: Type 2 (SCD)
    attributes:
      - product_key (PK, surrogate)
      - product_id (natural key)
      - sku
      - name
      - category_l1, category_l2, category_l3
      - brand
      - product_type
      - base_price
      - effective_from, effective_to
      - is_current
    scd_tracked_columns:
      - name
      - category_l1, category_l2, category_l3
      - base_price

  dim_entity:
    description: 予約対象エンティティ（ホテル、アトラクション等）
    type: Type 2 (SCD)
    attributes:
      - entity_key (PK, surrogate)
      - entity_id (natural key)
      - entity_type
      - name
      - partner_id
      - partner_name
      - star_rating (for hotels)
      - category
      - country, region, city
      - latitude, longitude
      - effective_from, effective_to
      - is_current

  dim_geography:
    description: 地理ディメンション
    type: Type 0 (Static)
    attributes:
      - geo_key (PK)
      - country_code, country_name
      - region_code, region_name
      - city_code, city_name
      - timezone
      - continent
```

### 4.2 Slowly Changing Dimensions（SCD）

#### 4.2.1 SCD Type 2 実装

```sql
-- BigQuery でのSCD Type 2 マージ処理
MERGE INTO `triptrip-prod.dwh.dim_user` AS target
USING (
    SELECT
        user_id,
        email_domain,
        registration_date,
        user_tier,
        preferred_language,
        preferred_currency,
        country,
        region,
        city,
        age_group,
        gender,
        CURRENT_TIMESTAMP() AS effective_from
    FROM `triptrip-prod.staging.stg_users`
    WHERE processing_date = CURRENT_DATE()
) AS source
ON target.user_id = source.user_id AND target.is_current = TRUE

-- 変更があったレコードを期限切れにする
WHEN MATCHED AND (
    target.user_tier != source.user_tier OR
    target.preferred_language != source.preferred_language OR
    target.preferred_currency != source.preferred_currency OR
    target.country != source.country
) THEN UPDATE SET
    effective_to = CURRENT_TIMESTAMP(),
    is_current = FALSE

-- 新規ユーザーの挿入
WHEN NOT MATCHED THEN INSERT (
    user_key,
    user_id,
    email_domain,
    registration_date,
    user_tier,
    preferred_language,
    preferred_currency,
    country,
    region,
    city,
    age_group,
    gender,
    effective_from,
    effective_to,
    is_current
) VALUES (
    GENERATE_UUID(),
    source.user_id,
    source.email_domain,
    source.registration_date,
    source.user_tier,
    source.preferred_language,
    source.preferred_currency,
    source.country,
    source.region,
    source.city,
    source.age_group,
    source.gender,
    source.effective_from,
    TIMESTAMP('9999-12-31'),
    TRUE
);

-- 変更があったユーザーの新バージョンを挿入
INSERT INTO `triptrip-prod.dwh.dim_user`
SELECT
    GENERATE_UUID() AS user_key,
    s.user_id,
    s.email_domain,
    s.registration_date,
    s.user_tier,
    s.preferred_language,
    s.preferred_currency,
    s.country,
    s.region,
    s.city,
    s.age_group,
    s.gender,
    CURRENT_TIMESTAMP() AS effective_from,
    TIMESTAMP('9999-12-31') AS effective_to,
    TRUE AS is_current
FROM `triptrip-prod.staging.stg_users` s
JOIN `triptrip-prod.dwh.dim_user` d
ON s.user_id = d.user_id
WHERE d.is_current = FALSE
  AND d.effective_to >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
  AND s.processing_date = CURRENT_DATE();
```

### 4.3 BigQuery構成

```yaml
bigquery_configuration:
  project: triptrip-prod
  location: asia-northeast1

  datasets:
    raw:
      description: 生データ（ランディングゾーン）
      default_table_expiration_days: 30
      access:
        - role: READER
          group: data-engineers@triptrip.com

    staging:
      description: ステージングデータ
      default_table_expiration_days: 7
      access:
        - role: WRITER
          group: data-engineers@triptrip.com

    dwh:
      description: データウェアハウス（スタースキーマ）
      access:
        - role: READER
          group: data-analysts@triptrip.com
        - role: WRITER
          group: data-engineers@triptrip.com

    marts:
      description: データマート（ビジネス領域別）
      access:
        - role: READER
          group: business-users@triptrip.com

    ml:
      description: 機械学習用データセット
      access:
        - role: READER
          group: ml-engineers@triptrip.com

  partitioning_strategy:
    fact_tables:
      partition_type: DAY
      partition_field: event_date
      partition_expiration_days: 2555  # 7年

    dimension_tables:
      partition_type: なし（小規模）

  clustering:
    fact_bookings:
      - booking_type
      - user_id
    fact_orders:
      - user_id
      - product_id
    fact_user_activity:
      - user_id
      - activity_type

  materialized_views:
    - name: mv_daily_booking_summary
      refresh_interval: 60分
      query: |
        SELECT
          booking_date,
          booking_type,
          COUNT(*) AS booking_count,
          SUM(total_amount) AS total_revenue
        FROM dwh.fact_bookings
        GROUP BY booking_date, booking_type

    - name: mv_user_lifetime_value
      refresh_interval: 24時間
      query: |
        SELECT
          user_id,
          COUNT(DISTINCT booking_id) AS total_bookings,
          SUM(total_amount) AS lifetime_value,
          MIN(booking_date) AS first_booking,
          MAX(booking_date) AS last_booking
        FROM dwh.fact_bookings
        GROUP BY user_id
```

---

## 第5章：データリネージとメタデータ管理

### 5.1 データリネージ追跡

#### 5.1.1 リネージアーキテクチャ

```yaml
lineage_architecture:
  tool: Apache Atlas / OpenLineage
  integration:
    - Airflow (OpenLineage integration)
    - Spark (OpenLineage Spark integration)
    - dbt (dbt artifacts)
    - BigQuery (INFORMATION_SCHEMA)

  lineage_capture:
    source_level:
      - データソース名
      - スキーマ/テーブル名
      - カラム名
      - 抽出条件

    transformation_level:
      - 変換ロジック
      - 結合条件
      - フィルター条件
      - 集計ロジック

    destination_level:
      - ターゲットテーブル
      - 書き込みモード
      - パーティショニング

  lineage_visualization:
    - テーブルレベルのDAG
    - カラムレベルの依存関係
    - 変換履歴のタイムライン
```

#### 5.1.2 OpenLineage実装

```python
# lineage/openlineage_integration.py
from openlineage.client import OpenLineageClient
from openlineage.client.run import RunEvent, RunState, Run, Job
from openlineage.client.facet import (
    DataSourceDatasetFacet,
    SchemaDatasetFacet,
    SchemaField,
    SqlJobFacet
)
from datetime import datetime
import uuid

class LineageTracker:
    def __init__(self, api_url: str, namespace: str):
        self.client = OpenLineageClient(url=api_url)
        self.namespace = namespace

    def emit_start_event(
        self,
        job_name: str,
        input_datasets: list,
        output_datasets: list,
        sql: str = None
    ) -> str:
        """ジョブ開始イベントの発行"""
        run_id = str(uuid.uuid4())

        run_event = RunEvent(
            eventType=RunState.START,
            eventTime=datetime.utcnow().isoformat() + "Z",
            run=Run(runId=run_id),
            job=Job(namespace=self.namespace, name=job_name),
            inputs=[self._create_dataset(ds) for ds in input_datasets],
            outputs=[self._create_dataset(ds) for ds in output_datasets]
        )

        if sql:
            run_event.job.facets = {
                "sql": SqlJobFacet(query=sql)
            }

        self.client.emit(run_event)
        return run_id

    def emit_complete_event(
        self,
        job_name: str,
        run_id: str,
        output_datasets: list
    ):
        """ジョブ完了イベントの発行"""
        run_event = RunEvent(
            eventType=RunState.COMPLETE,
            eventTime=datetime.utcnow().isoformat() + "Z",
            run=Run(runId=run_id),
            job=Job(namespace=self.namespace, name=job_name),
            outputs=[self._create_dataset(ds) for ds in output_datasets]
        )
        self.client.emit(run_event)

    def emit_fail_event(
        self,
        job_name: str,
        run_id: str,
        error_message: str
    ):
        """ジョブ失敗イベントの発行"""
        run_event = RunEvent(
            eventType=RunState.FAIL,
            eventTime=datetime.utcnow().isoformat() + "Z",
            run=Run(runId=run_id),
            job=Job(namespace=self.namespace, name=job_name)
        )
        self.client.emit(run_event)

    def _create_dataset(self, dataset_info: dict):
        """データセットオブジェクトの作成"""
        from openlineage.client.run import Dataset

        facets = {}

        if "source" in dataset_info:
            facets["dataSource"] = DataSourceDatasetFacet(
                name=dataset_info["source"]["name"],
                uri=dataset_info["source"]["uri"]
            )

        if "schema" in dataset_info:
            facets["schema"] = SchemaDatasetFacet(
                fields=[
                    SchemaField(name=f["name"], type=f["type"])
                    for f in dataset_info["schema"]
                ]
            )

        return Dataset(
            namespace=dataset_info.get("namespace", self.namespace),
            name=dataset_info["name"],
            facets=facets
        )


# 使用例
tracker = LineageTracker(
    api_url="http://openlineage-server:5000",
    namespace="triptrip-data-platform"
)

# ETLジョブでの使用
run_id = tracker.emit_start_event(
    job_name="daily_booking_etl",
    input_datasets=[
        {
            "name": "raw.cdc_bookings",
            "source": {"name": "postgresql", "uri": "postgresql://db:5432/triptrip"},
            "schema": [
                {"name": "id", "type": "uuid"},
                {"name": "user_id", "type": "uuid"},
                {"name": "total_amount", "type": "decimal"}
            ]
        }
    ],
    output_datasets=[
        {
            "name": "dwh.fact_bookings",
            "source": {"name": "bigquery", "uri": "bigquery://triptrip-prod"}
        }
    ],
    sql="SELECT ... FROM raw.cdc_bookings ..."
)

# 処理完了後
tracker.emit_complete_event(
    job_name="daily_booking_etl",
    run_id=run_id,
    output_datasets=[{"name": "dwh.fact_bookings"}]
)
```

### 5.2 メタデータカタログ

#### 5.2.1 データカタログ構成

```yaml
data_catalog:
  tool: Google Data Catalog / Datahub
  integration:
    - BigQuery（自動同期）
    - Cloud Storage（カスタムエントリ）
    - PostgreSQL（カスタムコネクタ）

  metadata_types:
    technical_metadata:
      - テーブル名、カラム名、データ型
      - パーティショニング、クラスタリング
      - 行数、サイズ
      - 更新日時

    business_metadata:
      - ビジネス定義
      - データオーナー
      - データスチュワード
      - 使用ガイドライン

    operational_metadata:
      - データ品質スコア
      - 鮮度情報
      - SLAステータス
      - 依存関係

  tagging_taxonomy:
    data_classification:
      - public
      - internal
      - confidential
      - restricted

    data_domain:
      - booking
      - commerce
      - user
      - partner
      - analytics

    pii_classification:
      - pii
      - pii_encrypted
      - non_pii

  search_capabilities:
    - テーブル名検索
    - カラム名検索
    - ビジネス用語検索
    - タグ検索
    - オーナー検索
```

---

## 第6章：実装ロードマップとフェーズ計画

### 6.1 フェーズ別実装計画

```yaml
implementation_roadmap:
  phase_1_foundation:
    duration: 月1-3
    objectives:
      - Kafka基盤の構築
      - 基本的なCDCパイプラインの実装
      - データレイクの構築

    deliverables:
      - Kafka Cluster（Confluent Cloud）
      - Debezium CDC Connector
      - GCSデータレイク構造
      - 基本的なETLジョブ

    success_criteria:
      - CDCイベントの遅延 < 10秒
      - データレイク書き込み成功率 > 99%

  phase_2_streaming:
    duration: 月4-6
    objectives:
      - Flinkストリーム処理の実装
      - リアルタイムダッシュボード対応
      - データ品質フレームワークの導入

    deliverables:
      - Flink Cluster（GKE）
      - リアルタイム集計ジョブ
      - Great Expectationsによる品質チェック

    success_criteria:
      - リアルタイム処理遅延 < 5秒
      - データ品質チェック自動化率 > 80%

  phase_3_warehouse:
    duration: 月7-9
    objectives:
      - データウェアハウス（BigQuery）の構築
      - スタースキーマの実装
      - BIツール連携

    deliverables:
      - BigQueryデータウェアハウス
      - ディメンション/ファクトテーブル
      - dbtモデル
      - Lookerダッシュボード

    success_criteria:
      - クエリ性能 P95 < 10秒
      - データ鮮度 T+1日以内

  phase_4_ml_integration:
    duration: 月10-12
    objectives:
      - ML特徴量パイプラインの構築
      - 特徴量ストアの導入
      - モデル学習パイプラインの自動化

    deliverables:
      - Vertex AI Feature Store
      - 特徴量エンジニアリングパイプライン
      - MLOpsパイプライン

    success_criteria:
      - 特徴量更新遅延 < 1時間
      - モデル再学習自動化率 > 90%
```

### 6.2 オーケストレーション設計

```yaml
orchestration:
  tool: Cloud Composer (Apache Airflow)

  dag_categories:
    ingestion_dags:
      - cdc_ingestion_dag
      - batch_file_ingestion_dag
      - api_data_ingestion_dag
      schedule: continuous / hourly

    transformation_dags:
      - daily_etl_dag
      - hourly_aggregation_dag
      - weekly_dimension_refresh_dag
      schedule: scheduled

    quality_dags:
      - data_quality_check_dag
      - sla_monitoring_dag
      schedule: after each ETL

    ml_dags:
      - feature_engineering_dag
      - model_training_dag
      - model_deployment_dag
      schedule: daily / weekly

  dag_dependencies:
    daily_etl_dag:
      depends_on:
        - cdc_ingestion_dag
        - batch_file_ingestion_dag
      triggers:
        - data_quality_check_dag
        - feature_engineering_dag

  alerting:
    on_failure:
      - slack: "#data-pipeline-alerts"
      - pagerduty: critical tasks only
    on_sla_miss:
      - slack: "#data-sla-alerts"
```

---

## 第7章：文書間参照 & 依存関係

### 7.1 関連文書マトリックス

```yaml
document_references:
  inputs:
    - document: Doc-DA-001
      title: Data Architecture & Schema Design
      relevance: データモデル定義、スキーマ設計

    - document: Doc-DA-002
      title: Database Strategy & Technology Selection
      relevance: データベース接続、ソースシステム

    - document: Doc-SA-002
      title: Microservices Architecture
      relevance: イベント駆動設計、サービス間通信

  outputs:
    - document: Doc-ML-001
      title: ML Platform Architecture
      dependency: 特徴量パイプライン、学習データ

    - document: Doc-BI-001
      title: BI & Analytics Architecture
      dependency: データウェアハウス、データマート
```

### 7.2 パイプラインコンポーネントサマリー

| コンポーネント | 技術 | 用途 |
|--------------|------|------|
| メッセージング | Apache Kafka (Confluent) | イベントストリーミング |
| CDC | Debezium | データベース変更検出 |
| ストリーム処理 | Apache Flink | リアルタイム集計 |
| バッチ処理 | Apache Spark (Dataproc) | 大規模ETL |
| データレイク | Cloud Storage (GCS) | 生データ保存 |
| データウェアハウス | BigQuery | 分析基盤 |
| オーケストレーション | Cloud Composer (Airflow) | ワークフロー管理 |
| データ品質 | Great Expectations | 品質検証 |
| リネージ | OpenLineage | データ追跡 |
| カタログ | Data Catalog | メタデータ管理 |

---

## 付録

### A. Kafkaトピック一覧

| トピック名 | パーティション数 | 保持期間 | 用途 |
|-----------|----------------|---------|------|
| triptrip.bookings | 12 | 7日 | 予約イベント |
| triptrip.orders | 12 | 7日 | 注文イベント |
| triptrip.user_activities | 24 | 3日 | ユーザー行動 |
| triptrip.inventory | 12 | 1日 | 在庫変更 |
| triptrip.cdc.* | 24 | 2日 | DBの変更イベント |

### B. 変更履歴

| バージョン | 日付 | 変更内容 | 作成者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-01-19 | 初版作成 | Technical Architecture Agent |

---

**文書情報**
- 文書ID: Doc-DA-003
- バージョン: 1.0.0
- ステータス: ドラフト
- 最終更新: 2026-01-19
- 次回レビュー: 2026-02-19

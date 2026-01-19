# Doc-DA-002: TripTripデータベース戦略 & 技術選定

## エグゼクティブサマリー

本文書は、TripTripプラットフォームのデータベース戦略と技術選定を包括的に定義します。既存のPostgreSQL基盤を中心に、マルチモデルデータベースアーキテクチャを設計し、各ワークロードに最適なデータベース技術を選定します。リレーショナルデータベースの拡張戦略、NoSQL・特殊用途データベースの活用、マルチレイヤーキャッシング設計、マルチリージョン展開を含む包括的なデータベース戦略を提供します。本設計は、10xから1000xのスケーリングに対応し、99.99%の可用性を実現する堅牢なデータベース基盤を構築します。

---

## 第1章：はじめに & コンテキスト

### 1.1 ビジネス要件 & 制約

#### 1.1.1 ワークロード特性分析

```yaml
workload_analysis:
  transactional_workload:
    description: 予約・注文処理
    characteristics:
      - 高い書き込み頻度
      - 強い一貫性要件
      - 複雑なトランザクション
      - 低レイテンシ要求
    metrics:
      peak_tps: 10,000
      avg_transaction_size: 5KB
      consistency: ACID必須

  analytical_workload:
    description: レポート・ダッシュボード
    characteristics:
      - 大量データスキャン
      - 複雑な集計クエリ
      - 読み取り専用
      - バッチ処理許容
    metrics:
      query_complexity: 高（多重結合）
      data_volume: TB級
      latency_tolerance: 秒〜分

  search_workload:
    description: 商品・施設検索
    characteristics:
      - 全文検索
      - ファセット検索
      - 地理空間検索
      - リアルタイム更新
    metrics:
      queries_per_second: 50,000
      index_size: 100GB
      update_frequency: 分単位

  caching_workload:
    description: セッション・頻繁アクセスデータ
    characteristics:
      - 超低レイテンシ
      - 高スループット
      - TTL管理
      - 分散キャッシュ
    metrics:
      read_qps: 500,000
      write_qps: 50,000
      avg_latency: < 1ms

  time_series_workload:
    description: メトリクス・ログ・価格履歴
    characteristics:
      - 追記専用
      - 時間ベースクエリ
      - 高圧縮率
      - 自動データ圧縮
    metrics:
      ingest_rate: 100,000 events/sec
      retention: 2年
      query_pattern: 時間範囲スキャン
```

#### 1.1.2 技術的制約

```yaml
technical_constraints:
  existing_infrastructure:
    primary_database: PostgreSQL 16
    orm: Prisma 5.8.0
    backend: Node.js 18+ / Hono

  compatibility_requirements:
    - 既存Prismaスキーマとの互換性
    - Node.jsドライバーの成熟度
    - TypeScript型サポート
    - 段階的移行の実現可能性

  operational_constraints:
    - チーム規模: 現在5名（将来50名）
    - 運用経験: PostgreSQL中心
    - 予算: 段階的投資
    - マネージドサービス優先

  cloud_preference:
    primary: Google Cloud Platform (GCP)
    secondary: AWS（一部サービス）
    requirements:
      - マネージドサービス優先
      - マルチリージョン対応
      - 自動スケーリング
      - 24/7サポート
```

### 1.2 技術目標 & 成功基準

#### 1.2.1 可用性・パフォーマンス目標

```yaml
availability_targets:
  overall_sla: 99.99%
  component_targets:
    primary_database:
      availability: 99.99%
      rpo: 1分
      rto: 5分
    cache_layer:
      availability: 99.95%
      failover_time: 10秒
    search_cluster:
      availability: 99.9%
      reindex_time: 30分

performance_targets:
  primary_database:
    read_latency_p95: 20ms
    write_latency_p95: 50ms
    connection_limit: 10,000

  cache_layer:
    read_latency_p99: 5ms
    write_latency_p99: 10ms
    throughput: 1M ops/sec

  search_cluster:
    query_latency_p95: 100ms
    indexing_throughput: 10,000 docs/sec
```

#### 1.2.2 スケーラビリティ目標

```yaml
scalability_targets:
  year_1:
    total_data: 100GB
    daily_transactions: 50,000
    concurrent_users: 10,000

  year_3:
    total_data: 5TB
    daily_transactions: 500,000
    concurrent_users: 100,000

  year_5:
    total_data: 100TB
    daily_transactions: 5,000,000
    concurrent_users: 1,000,000

scaling_approach:
  - 垂直スケーリング（Phase 1）
  - リードレプリカ（Phase 2）
  - シャーディング（Phase 3）
  - マルチリージョン（Phase 4）
```

### 1.3 主要な前提条件 & トレードオフ

#### 1.3.1 データベース選定原則

```yaml
selection_principles:
  1_right_tool_for_the_job:
    description: ワークロードに最適なデータベースを選定
    implementation:
      - OLTP: PostgreSQL
      - キャッシュ: Redis
      - 検索: Elasticsearch
      - 分析: BigQuery
      - 時系列: TimescaleDB

  2_managed_service_first:
    description: 運用負荷軽減のためマネージドサービスを優先
    exceptions:
      - 特殊要件がある場合
      - コスト効率が著しく悪い場合

  3_polyglot_persistence:
    description: 多言語永続化による最適化
    considerations:
      - 運用複雑性の増加
      - データ整合性の管理
      - チームスキルセット

  4_gradual_adoption:
    description: 段階的な技術導入
    phases:
      - Phase 1: PostgreSQL最適化
      - Phase 2: Redis + Elasticsearch追加
      - Phase 3: 分析基盤構築
      - Phase 4: グローバル展開
```

---

## 第2章：リレーショナルデータベース戦略

### 2.1 PostgreSQL拡張とスケーリング

#### 2.1.1 現行構成と拡張計画

```yaml
postgresql_evolution:
  current_state:
    version: PostgreSQL 16
    deployment: 単一インスタンス
    storage: ローカルSSD
    connections: PgBouncer（計画中）

  phase_1_optimization:
    timeline: 月1-3
    actions:
      - 接続プーリング導入（PgBouncer）
      - クエリ最適化
      - インデックスチューニング
      - パーティショニング実装
    target_capacity:
      max_connections: 1,000
      storage: 500GB
      iops: 10,000

  phase_2_read_scaling:
    timeline: 月4-6
    actions:
      - Cloud SQL導入（GCP）
      - リードレプリカ追加（2台）
      - 読み取り分散実装
    target_capacity:
      max_connections: 5,000
      read_throughput: 50,000 QPS
      replication_lag: < 1秒

  phase_3_high_availability:
    timeline: 月7-12
    actions:
      - リージョナルHA構成
      - 自動フェイルオーバー
      - Point-in-time Recovery
    target_capacity:
      availability: 99.99%
      failover_time: < 60秒
      backup_retention: 35日

  phase_4_global:
    timeline: Year 2
    actions:
      - マルチリージョン展開
      - クロスリージョンレプリカ
      - グローバルロードバランシング
```

#### 2.1.2 Cloud SQL構成詳細

```yaml
cloud_sql_configuration:
  instance_type:
    machine_type: db-custom-16-61440  # 16 vCPU, 60GB RAM
    storage_type: SSD
    storage_size: 1TB
    storage_auto_resize: true

  high_availability:
    regional: true
    automatic_failover: true
    maintenance_window:
      day: SUNDAY
      hour: 3  # 3:00 AM JST

  backup_configuration:
    enabled: true
    start_time: "02:00"
    retention_days: 35
    point_in_time_recovery: true
    transaction_log_retention_days: 7

  read_replicas:
    count: 2
    locations:
      - asia-northeast1-b
      - asia-northeast1-c
    machine_type: db-custom-8-30720

  connection_pooling:
    pgbouncer:
      enabled: true
      pool_mode: transaction
      default_pool_size: 100
      max_client_conn: 10000
      reserve_pool_size: 5

  flags:
    - name: max_connections
      value: "500"
    - name: shared_buffers
      value: "15360MB"  # 25% of RAM
    - name: effective_cache_size
      value: "45GB"  # 75% of RAM
    - name: work_mem
      value: "256MB"
    - name: maintenance_work_mem
      value: "2GB"
    - name: random_page_cost
      value: "1.1"  # SSD最適化
    - name: effective_io_concurrency
      value: "200"
    - name: max_parallel_workers_per_gather
      value: "4"
    - name: max_parallel_workers
      value: "8"
```

### 2.2 リードレプリカとフェイルオーバー

#### 2.2.1 読み取り分散アーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     READ/WRITE DISTRIBUTION ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         Application Layer                            │   │
│  │                                                                      │   │
│  │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │   │
│  │   │ API Server  │  │ API Server  │  │ API Server  │                │   │
│  │   │     #1      │  │     #2      │  │     #3      │                │   │
│  │   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                │   │
│  │          │                │                │                        │   │
│  └──────────┼────────────────┼────────────────┼────────────────────────┘   │
│             │                │                │                             │
│             └────────────────┼────────────────┘                             │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        PgBouncer Cluster                             │   │
│  │                                                                      │   │
│  │   ┌─────────────────────┐      ┌─────────────────────┐             │   │
│  │   │   Write Pool        │      │    Read Pool        │             │   │
│  │   │   (Primary Only)    │      │ (Replicas + Primary)│             │   │
│  │   └──────────┬──────────┘      └──────────┬──────────┘             │   │
│  │              │                            │                         │   │
│  └──────────────┼────────────────────────────┼─────────────────────────┘   │
│                 │                            │                              │
│                 │                    ┌───────┴───────┐                     │
│                 │                    │               │                     │
│                 ▼                    ▼               ▼                     │
│  ┌─────────────────────┐   ┌─────────────┐  ┌─────────────┐              │
│  │                     │   │             │  │             │              │
│  │   Primary (Writer)  │   │  Replica 1  │  │  Replica 2  │              │
│  │   asia-northeast1-a │──▶│  (Reader)   │  │  (Reader)   │              │
│  │                     │   │  -b zone    │  │  -c zone    │              │
│  │                     │   │             │  │             │              │
│  └─────────────────────┘   └─────────────┘  └─────────────┘              │
│           │                                                               │
│           │ Streaming Replication (async)                                 │
│           └──────────────────────────────────────────────────────────────▶│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 2.2.2 読み取りルーティング実装

```typescript
// database/connection-manager.ts
import { Pool } from 'pg';

interface PoolConfig {
  host: string;
  port: number;
  database: string;
  user: string;
  password: string;
  max: number;
  idleTimeoutMillis: number;
  connectionTimeoutMillis: number;
}

interface ReplicaHealth {
  host: string;
  isHealthy: boolean;
  replicationLag: number;
  lastChecked: Date;
}

class DatabaseConnectionManager {
  private primaryPool: Pool;
  private replicaPools: Pool[] = [];
  private replicaHealth: Map<string, ReplicaHealth> = new Map();
  private currentReplicaIndex: number = 0;

  constructor(
    primaryConfig: PoolConfig,
    replicaConfigs: PoolConfig[]
  ) {
    this.primaryPool = new Pool(primaryConfig);
    replicaConfigs.forEach(config => {
      this.replicaPools.push(new Pool(config));
      this.replicaHealth.set(config.host, {
        host: config.host,
        isHealthy: true,
        replicationLag: 0,
        lastChecked: new Date()
      });
    });

    // ヘルスチェック開始
    this.startHealthCheck();
  }

  // 書き込み操作用（常にプライマリ）
  async executeWrite<T>(query: string, params?: any[]): Promise<T> {
    const client = await this.primaryPool.connect();
    try {
      const result = await client.query(query, params);
      return result.rows as T;
    } finally {
      client.release();
    }
  }

  // 読み取り操作用（レプリカ優先、ラウンドロビン）
  async executeRead<T>(
    query: string,
    params?: any[],
    options?: { requireFreshData?: boolean; maxReplicationLag?: number }
  ): Promise<T> {
    const maxLag = options?.maxReplicationLag ?? 1000; // デフォルト1秒

    // 新鮮なデータが必要な場合はプライマリから読み取り
    if (options?.requireFreshData) {
      return this.executeWrite(query, params);
    }

    // 健全なレプリカを選択
    const pool = this.selectHealthyReplica(maxLag);
    const client = await pool.connect();
    try {
      const result = await client.query(query, params);
      return result.rows as T;
    } finally {
      client.release();
    }
  }

  private selectHealthyReplica(maxLag: number): Pool {
    const healthyReplicas = this.replicaPools.filter((_, index) => {
      const config = this.replicaPools[index];
      // @ts-ignore
      const health = this.replicaHealth.get(config.options.host);
      return health?.isHealthy && health.replicationLag <= maxLag;
    });

    if (healthyReplicas.length === 0) {
      // 健全なレプリカがない場合はプライマリにフォールバック
      console.warn('No healthy replicas available, falling back to primary');
      return this.primaryPool;
    }

    // ラウンドロビン選択
    this.currentReplicaIndex = (this.currentReplicaIndex + 1) % healthyReplicas.length;
    return healthyReplicas[this.currentReplicaIndex];
  }

  private async startHealthCheck(): Promise<void> {
    setInterval(async () => {
      for (let i = 0; i < this.replicaPools.length; i++) {
        const pool = this.replicaPools[i];
        try {
          const client = await pool.connect();
          const result = await client.query(`
            SELECT EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())) * 1000 AS lag_ms
          `);
          const lagMs = result.rows[0]?.lag_ms ?? 0;

          // @ts-ignore
          const host = pool.options.host;
          this.replicaHealth.set(host, {
            host,
            isHealthy: true,
            replicationLag: lagMs,
            lastChecked: new Date()
          });
          client.release();
        } catch (error) {
          // @ts-ignore
          const host = pool.options.host;
          this.replicaHealth.set(host, {
            host,
            isHealthy: false,
            replicationLag: Infinity,
            lastChecked: new Date()
          });
          console.error(`Replica health check failed for ${host}:`, error);
        }
      }
    }, 5000); // 5秒ごとにチェック
  }
}
```

### 2.3 シャーディング戦略

#### 2.3.1 シャーディング設計方針

```yaml
sharding_strategy:
  approach: アプリケーションレベルシャーディング（Citus使用）
  timeline: Year 3以降（必要に応じて）

  sharding_candidates:
    bookings:
      shard_key: user_id
      rationale: ユーザーのすべての予約を同一シャードに配置
      distribution: ハッシュ

    orders:
      shard_key: user_id
      rationale: ユーザーのすべての注文を同一シャードに配置
      distribution: ハッシュ

    user_activities:
      shard_key: user_id
      rationale: ユーザーのアクティビティを同一シャードに配置
      distribution: ハッシュ

  non_sharding_tables:
    - users（リファレンステーブル、全シャードにレプリカ）
    - products（リファレンステーブル）
    - hotels（リファレンステーブル）
    - attractions（リファレンステーブル）

  citus_configuration:
    coordinator_nodes: 1
    worker_nodes: 4（初期）→ 16（スケール時）
    shard_count: 64
    replication_factor: 2
```

#### 2.3.2 Citus導入計画

```yaml
citus_migration_plan:
  phase_1_preparation:
    duration: 2ヶ月
    tasks:
      - Citus拡張のインストールと検証
      - シャードキー設計の最終化
      - アプリケーションコードの準備
      - テスト環境での検証

  phase_2_migration:
    duration: 1ヶ月
    tasks:
      - コーディネーターノードのセットアップ
      - ワーカーノードの追加
      - リファレンステーブルの配布
      - 分散テーブルの作成

  phase_3_data_migration:
    duration: 2ヶ月
    tasks:
      - 既存データの分散
      - インデックスの再作成
      - クエリパフォーマンスの検証
      - アプリケーションの段階的切り替え

  phase_4_optimization:
    duration: 1ヶ月
    tasks:
      - クエリプランの最適化
      - シャード再バランシング
      - モニタリングの強化
```

---

## 第3章：NoSQL・特殊用途データベース

### 3.1 ドキュメントストア（MongoDB）ユースケース

#### 3.1.1 MongoDB採用領域

```yaml
mongodb_use_cases:
  content_management:
    description: CMS・コンテンツ管理
    data_characteristics:
      - スキーマレスなコンテンツ
      - 多言語コンテンツ
      - バージョン管理
      - リッチメディア参照
    collections:
      - pages
      - articles
      - faqs
      - translations

  user_generated_content:
    description: ユーザー生成コンテンツ
    data_characteristics:
      - 柔軟な構造（レビュー、コメント）
      - 埋め込みドキュメント
      - 高い書き込み頻度
    collections:
      - reviews
      - comments
      - travel_logs

  configuration_and_settings:
    description: 設定・フィーチャーフラグ
    data_characteristics:
      - 頻繁な変更
      - 階層構造
      - 環境別設定
    collections:
      - feature_flags
      - app_configs
      - partner_configs
```

#### 3.1.2 MongoDB構成

```yaml
mongodb_atlas_configuration:
  cluster_tier: M30  # 専用クラスター
  cloud_provider: GCP
  region: asia-northeast1

  replication:
    replica_set_nodes: 3
    priority_configuration:
      primary: asia-northeast1-a
      secondary_1: asia-northeast1-b
      secondary_2: asia-northeast1-c

  storage:
    storage_engine: WiredTiger
    encryption_at_rest: true
    auto_scaling: true

  indexes:
    reviews:
      - keys: { entity_type: 1, entity_id: 1, created_at: -1 }
      - keys: { user_id: 1, created_at: -1 }
      - keys: { "$**": "text" }  # 全文検索

  schema_validation:
    reviews:
      validator:
        $jsonSchema:
          bsonType: object
          required: [entity_type, entity_id, user_id, rating, content]
          properties:
            entity_type:
              enum: [hotel, attraction, product, restaurant]
            rating:
              bsonType: int
              minimum: 1
              maximum: 5
```

### 3.2 キーバリューストア（Redis）設計

#### 3.2.1 Redis活用戦略

```yaml
redis_use_cases:
  session_management:
    description: ユーザーセッション管理
    data_structure: Hash
    ttl: 24時間
    key_pattern: "session:{session_id}"
    operations:
      - HSET/HGET（セッションデータ）
      - EXPIRE（TTL更新）
      - DEL（ログアウト）

  api_caching:
    description: APIレスポンスキャッシュ
    data_structure: String (JSON)
    ttl: 5分〜1時間（エンドポイント別）
    key_pattern: "cache:api:{endpoint}:{hash(params)}"
    operations:
      - SET/GET
      - SETEX（TTL付きセット）

  rate_limiting:
    description: APIレート制限
    data_structure: String (Counter)
    ttl: 1分
    key_pattern: "ratelimit:{user_id}:{endpoint}"
    operations:
      - INCR
      - EXPIRE

  real_time_counters:
    description: リアルタイムカウンター
    data_structure: Sorted Set / HyperLogLog
    use_cases:
      - 閲覧数
      - いいね数
      - オンラインユーザー数
    key_patterns:
      - "counter:views:{entity_type}:{entity_id}"
      - "counter:online:users"

  distributed_locks:
    description: 分散ロック
    data_structure: String
    ttl: 30秒（自動解放）
    key_pattern: "lock:{resource}"
    operations:
      - SET NX EX（取得）
      - DEL（解放）

  leaderboard:
    description: ランキング
    data_structure: Sorted Set
    use_cases:
      - 人気ホテルランキング
      - 売れ筋商品ランキング
    key_pattern: "leaderboard:{type}:{period}"
    operations:
      - ZADD（スコア更新）
      - ZREVRANGE（トップN取得）
```

#### 3.2.2 Redis Cluster構成

```yaml
redis_memorystore_configuration:
  tier: Standard  # HA構成
  capacity_gb: 32
  region: asia-northeast1

  read_replicas:
    enabled: true
    replica_count: 2

  configuration:
    maxmemory_policy: allkeys-lru
    notify_keyspace_events: Ex  # 期限切れイベント

  connection_pooling:
    min_idle: 10
    max_idle: 50
    max_total: 100
    max_wait_ms: 5000

  sentinel_configuration:
    quorum: 2
    down_after_ms: 5000
    failover_timeout_ms: 60000
```

```typescript
// redis/redis-client.ts
import Redis from 'ioredis';

interface RedisClientConfig {
  host: string;
  port: number;
  password?: string;
  db?: number;
  keyPrefix?: string;
  maxRetriesPerRequest?: number;
  enableReadyCheck?: boolean;
  lazyConnect?: boolean;
}

class RedisClientManager {
  private static instance: RedisClientManager;
  private client: Redis;
  private subscriber: Redis;

  private constructor(config: RedisClientConfig) {
    this.client = new Redis({
      ...config,
      retryStrategy: (times: number) => {
        if (times > 3) return null;
        return Math.min(times * 100, 3000);
      }
    });

    this.subscriber = new Redis(config);

    this.client.on('error', (err) => {
      console.error('Redis Client Error:', err);
    });

    this.client.on('connect', () => {
      console.log('Redis Client Connected');
    });
  }

  static getInstance(config?: RedisClientConfig): RedisClientManager {
    if (!RedisClientManager.instance && config) {
      RedisClientManager.instance = new RedisClientManager(config);
    }
    return RedisClientManager.instance;
  }

  // セッション管理
  async setSession(sessionId: string, data: Record<string, any>, ttlSeconds: number = 86400): Promise<void> {
    const key = `session:${sessionId}`;
    await this.client.hset(key, data);
    await this.client.expire(key, ttlSeconds);
  }

  async getSession(sessionId: string): Promise<Record<string, string> | null> {
    const key = `session:${sessionId}`;
    const data = await this.client.hgetall(key);
    return Object.keys(data).length > 0 ? data : null;
  }

  // APIキャッシュ
  async cacheApiResponse(
    endpoint: string,
    params: Record<string, any>,
    data: any,
    ttlSeconds: number
  ): Promise<void> {
    const hash = this.hashParams(params);
    const key = `cache:api:${endpoint}:${hash}`;
    await this.client.setex(key, ttlSeconds, JSON.stringify(data));
  }

  async getCachedApiResponse<T>(
    endpoint: string,
    params: Record<string, any>
  ): Promise<T | null> {
    const hash = this.hashParams(params);
    const key = `cache:api:${endpoint}:${hash}`;
    const data = await this.client.get(key);
    return data ? JSON.parse(data) : null;
  }

  // レート制限
  async checkRateLimit(
    userId: string,
    endpoint: string,
    limit: number,
    windowSeconds: number
  ): Promise<{ allowed: boolean; remaining: number; resetAt: number }> {
    const key = `ratelimit:${userId}:${endpoint}`;
    const now = Date.now();

    const multi = this.client.multi();
    multi.incr(key);
    multi.ttl(key);
    const results = await multi.exec();

    const count = results?.[0]?.[1] as number ?? 0;
    const ttl = results?.[1]?.[1] as number ?? -1;

    if (ttl === -1) {
      await this.client.expire(key, windowSeconds);
    }

    return {
      allowed: count <= limit,
      remaining: Math.max(0, limit - count),
      resetAt: now + (ttl > 0 ? ttl * 1000 : windowSeconds * 1000)
    };
  }

  // 分散ロック
  async acquireLock(
    resource: string,
    ttlMs: number = 30000
  ): Promise<string | null> {
    const lockId = `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
    const key = `lock:${resource}`;

    const result = await this.client.set(
      key,
      lockId,
      'PX',
      ttlMs,
      'NX'
    );

    return result === 'OK' ? lockId : null;
  }

  async releaseLock(resource: string, lockId: string): Promise<boolean> {
    const key = `lock:${resource}`;
    const script = `
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
      else
        return 0
      end
    `;
    const result = await this.client.eval(script, 1, key, lockId);
    return result === 1;
  }

  private hashParams(params: Record<string, any>): string {
    const sorted = JSON.stringify(params, Object.keys(params).sort());
    let hash = 0;
    for (let i = 0; i < sorted.length; i++) {
      const char = sorted.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash;
    }
    return Math.abs(hash).toString(36);
  }
}
```

### 3.3 時系列データベース（TimescaleDB）

#### 3.3.1 TimescaleDB活用領域

```yaml
timescaledb_use_cases:
  price_analytics:
    description: 価格分析・動的価格設定
    hypertable: price_metrics
    chunk_interval: 1 day
    retention: 2年
    compression: 7日後に自動圧縮

  booking_metrics:
    description: 予約メトリクス・トレンド分析
    hypertable: booking_metrics
    chunk_interval: 1 hour
    retention: 1年
    compression: 24時間後

  system_metrics:
    description: システムパフォーマンスメトリクス
    hypertable: system_metrics
    chunk_interval: 1 hour
    retention: 90日
    compression: 6時間後

  user_analytics:
    description: ユーザー行動分析
    hypertable: user_analytics_events
    chunk_interval: 1 day
    retention: 2年
    compression: 7日後
```

#### 3.3.2 TimescaleDB構成

```sql
-- TimescaleDB拡張の有効化
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- 価格メトリクスハイパーテーブル
CREATE TABLE price_metrics (
    time TIMESTAMPTZ NOT NULL,
    entity_type VARCHAR(20) NOT NULL,
    entity_id UUID NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'JPY',
    competitor_avg_price DECIMAL(10, 2),
    demand_index DECIMAL(5, 2),
    occupancy_rate DECIMAL(5, 4),
    metadata JSONB DEFAULT '{}'
);

SELECT create_hypertable('price_metrics', 'time', chunk_time_interval => INTERVAL '1 day');

-- 圧縮ポリシー
ALTER TABLE price_metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'entity_type, entity_id'
);

SELECT add_compression_policy('price_metrics', INTERVAL '7 days');

-- 保持ポリシー
SELECT add_retention_policy('price_metrics', INTERVAL '2 years');

-- 連続集計（マテリアライズドビュー）
CREATE MATERIALIZED VIEW price_metrics_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    entity_type,
    entity_id,
    AVG(price) AS avg_price,
    MIN(price) AS min_price,
    MAX(price) AS max_price,
    AVG(demand_index) AS avg_demand,
    COUNT(*) AS sample_count
FROM price_metrics
GROUP BY bucket, entity_type, entity_id
WITH NO DATA;

-- 自動リフレッシュポリシー
SELECT add_continuous_aggregate_policy('price_metrics_hourly',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- 予約メトリクスハイパーテーブル
CREATE TABLE booking_metrics (
    time TIMESTAMPTZ NOT NULL,
    booking_type VARCHAR(20) NOT NULL,
    partner_id UUID,
    entity_id UUID,
    region VARCHAR(50),
    bookings_count INTEGER DEFAULT 0,
    cancellations_count INTEGER DEFAULT 0,
    revenue DECIMAL(12, 2) DEFAULT 0,
    avg_lead_time_hours DECIMAL(8, 2),
    avg_party_size DECIMAL(4, 2)
);

SELECT create_hypertable('booking_metrics', 'time', chunk_time_interval => INTERVAL '1 hour');
```

---

## 第4章：キャッシング戦略

### 4.1 マルチレイヤーキャッシング設計

#### 4.1.1 キャッシングレイヤー構成

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MULTI-LAYER CACHING ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                          Layer 1: CDN Cache                          │   │
│  │                     (Cloud CDN / Cloudflare)                         │   │
│  │                                                                      │   │
│  │   • 静的アセット (images, js, css)                                   │   │
│  │   • 公開APIレスポンス (商品リスト, ホテル検索結果)                    │   │
│  │   • TTL: 1分〜24時間                                                 │   │
│  │   • ヒット率目標: 80%+                                               │   │
│  └────────────────────────────────────────────────────────────────────┬─┘   │
│                                                                        │     │
│                                         Cache Miss                     │     │
│                                                                        ▼     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                       Layer 2: Application Cache                     │   │
│  │                           (Redis Cluster)                            │   │
│  │                                                                      │   │
│  │   • セッションデータ                                                 │   │
│  │   • APIレスポンスキャッシュ                                          │   │
│  │   • 計算済み集計データ                                               │   │
│  │   • リアルタイムカウンター                                           │   │
│  │   • TTL: 1秒〜1時間                                                  │   │
│  │   • ヒット率目標: 95%+                                               │   │
│  └────────────────────────────────────────────────────────────────────┬─┘   │
│                                                                        │     │
│                                         Cache Miss                     │     │
│                                                                        ▼     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                       Layer 3: Query Cache                           │   │
│  │                    (PostgreSQL / Materialized Views)                 │   │
│  │                                                                      │   │
│  │   • マテリアライズドビュー                                           │   │
│  │   • プリペアドステートメントキャッシュ                               │   │
│  │   • 共有バッファ                                                     │   │
│  │   • リフレッシュ間隔: 1分〜1時間                                     │   │
│  └────────────────────────────────────────────────────────────────────┬─┘   │
│                                                                        │     │
│                                         Cache Miss                     │     │
│                                                                        ▼     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                       Layer 4: Storage Layer                         │   │
│  │                      (PostgreSQL / MongoDB)                          │   │
│  │                                                                      │   │
│  │   • プライマリデータストア                                           │   │
│  │   • 真実の単一ソース                                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 4.1.2 エンドポイント別キャッシュ戦略

```yaml
cache_strategy_by_endpoint:
  public_read_heavy:
    endpoints:
      - GET /api/hotels
      - GET /api/hotels/:id
      - GET /api/attractions
      - GET /api/products
    strategy:
      cdn_cache: true
      cdn_ttl: 5分
      redis_cache: true
      redis_ttl: 1分
      stale_while_revalidate: true

  personalized:
    endpoints:
      - GET /api/users/me
      - GET /api/users/me/bookings
      - GET /api/users/me/orders
    strategy:
      cdn_cache: false
      redis_cache: true
      redis_ttl: 30秒
      cache_key_includes: [user_id]

  search_results:
    endpoints:
      - GET /api/search/hotels
      - GET /api/search/products
    strategy:
      cdn_cache: true
      cdn_ttl: 1分
      redis_cache: true
      redis_ttl: 30秒
      cache_key_includes: [query, filters, page]

  real_time_data:
    endpoints:
      - GET /api/inventory/:id/availability
      - GET /api/rooms/:id/price
    strategy:
      cdn_cache: false
      redis_cache: true
      redis_ttl: 10秒
      background_refresh: true

  write_operations:
    endpoints:
      - POST /api/bookings
      - POST /api/orders
      - PUT /api/users/me
    strategy:
      cdn_cache: false
      redis_cache: false
      invalidate_related_caches: true
```

### 4.2 キャッシュ無効化パターン

#### 4.2.1 無効化戦略

```yaml
invalidation_strategies:
  time_based:
    description: TTLによる自動期限切れ
    use_cases:
      - 頻繁に更新されるデータ
      - 正確性より鮮度が重要なデータ
    implementation:
      - Redis SETEX / EXPIRE
      - CDN Cache-Control max-age

  event_based:
    description: データ変更イベントによる即時無効化
    use_cases:
      - クリティカルなデータ（在庫、価格）
      - ユーザー固有データ
    implementation:
      - Pub/Sub経由のキャッシュ無効化
      - CDN Purge API

  version_based:
    description: データバージョンによる無効化
    use_cases:
      - 静的コンテンツ
      - 設定データ
    implementation:
      - キャッシュキーにバージョンを含める
      - ETag / If-None-Match

  tag_based:
    description: タグによるグループ無効化
    use_cases:
      - 関連データの一括無効化
      - カテゴリ単位の更新
    implementation:
      - Redis SET / SMEMBERS
      - キャッシュタグの管理
```

#### 4.2.2 キャッシュ無効化実装

```typescript
// cache/cache-invalidation.ts
import { RedisClientManager } from './redis-client';
import { PubSubClient } from './pubsub-client';

interface CacheInvalidationEvent {
  type: 'single' | 'pattern' | 'tag';
  keys?: string[];
  pattern?: string;
  tags?: string[];
  reason: string;
  timestamp: number;
}

class CacheInvalidationManager {
  private redis: RedisClientManager;
  private pubsub: PubSubClient;
  private tagPrefix = 'cache:tags:';

  constructor(redis: RedisClientManager, pubsub: PubSubClient) {
    this.redis = redis;
    this.pubsub = pubsub;

    // 無効化イベントのサブスクライブ
    this.subscribeToInvalidationEvents();
  }

  // 単一キーの無効化
  async invalidateKey(key: string, reason: string): Promise<void> {
    await this.redis.del(key);
    await this.publishInvalidationEvent({
      type: 'single',
      keys: [key],
      reason,
      timestamp: Date.now()
    });
  }

  // パターンによる無効化
  async invalidatePattern(pattern: string, reason: string): Promise<void> {
    const keys = await this.redis.keys(pattern);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
    await this.publishInvalidationEvent({
      type: 'pattern',
      pattern,
      reason,
      timestamp: Date.now()
    });
  }

  // タグによる無効化
  async invalidateByTags(tags: string[], reason: string): Promise<void> {
    const keysToInvalidate = new Set<string>();

    for (const tag of tags) {
      const tagKey = `${this.tagPrefix}${tag}`;
      const members = await this.redis.smembers(tagKey);
      members.forEach(key => keysToInvalidate.add(key));
      await this.redis.del(tagKey);
    }

    if (keysToInvalidate.size > 0) {
      await this.redis.del(...Array.from(keysToInvalidate));
    }

    await this.publishInvalidationEvent({
      type: 'tag',
      tags,
      keys: Array.from(keysToInvalidate),
      reason,
      timestamp: Date.now()
    });
  }

  // キャッシュセット時にタグを関連付け
  async setWithTags(
    key: string,
    value: any,
    ttlSeconds: number,
    tags: string[]
  ): Promise<void> {
    await this.redis.setex(key, ttlSeconds, JSON.stringify(value));

    for (const tag of tags) {
      const tagKey = `${this.tagPrefix}${tag}`;
      await this.redis.sadd(tagKey, key);
      await this.redis.expire(tagKey, ttlSeconds + 3600); // タグは1時間長く保持
    }
  }

  // エンティティ更新時の自動無効化
  async onEntityUpdate(
    entityType: string,
    entityId: string,
    operation: 'create' | 'update' | 'delete'
  ): Promise<void> {
    const tags = [
      `entity:${entityType}`,
      `entity:${entityType}:${entityId}`
    ];

    // エンティティタイプ別の追加タグ
    switch (entityType) {
      case 'hotel':
        tags.push('search:hotels', 'listing:hotels');
        break;
      case 'product':
        tags.push('search:products', 'listing:products');
        break;
      case 'booking':
        tags.push(`user:bookings`);
        break;
    }

    await this.invalidateByTags(tags, `${entityType} ${operation}: ${entityId}`);
  }

  private async publishInvalidationEvent(event: CacheInvalidationEvent): Promise<void> {
    await this.pubsub.publish('cache-invalidation', JSON.stringify(event));
  }

  private subscribeToInvalidationEvents(): void {
    this.pubsub.subscribe('cache-invalidation', async (message) => {
      const event: CacheInvalidationEvent = JSON.parse(message);
      console.log('Cache invalidation event received:', event);
      // ローカルキャッシュの無効化等の追加処理
    });
  }
}
```

---

## 第5章：データベース運用・監視

### 5.1 監視アーキテクチャ

#### 5.1.1 監視メトリクス定義

```yaml
monitoring_metrics:
  postgresql:
    performance:
      - metric: query_execution_time
        description: クエリ実行時間
        thresholds:
          warning: p95 > 100ms
          critical: p95 > 500ms
        alert_channel: slack, pagerduty

      - metric: connections_active
        description: アクティブ接続数
        thresholds:
          warning: > 80%
          critical: > 95%
        alert_channel: slack

      - metric: transactions_per_second
        description: トランザクション/秒
        baseline: 動的（過去7日平均）
        thresholds:
          warning: < 50% baseline
          critical: < 20% baseline

    replication:
      - metric: replication_lag_bytes
        description: レプリケーション遅延（バイト）
        thresholds:
          warning: > 10MB
          critical: > 100MB

      - metric: replication_lag_seconds
        description: レプリケーション遅延（秒）
        thresholds:
          warning: > 5s
          critical: > 30s

    storage:
      - metric: disk_usage_percent
        description: ディスク使用率
        thresholds:
          warning: > 70%
          critical: > 85%

      - metric: table_bloat_percent
        description: テーブル膨張率
        thresholds:
          warning: > 20%
          critical: > 40%

  redis:
    performance:
      - metric: operations_per_second
        description: オペレーション/秒
        baseline: 動的

      - metric: latency_p99
        description: レイテンシP99
        thresholds:
          warning: > 5ms
          critical: > 20ms

    memory:
      - metric: memory_used_percent
        description: メモリ使用率
        thresholds:
          warning: > 70%
          critical: > 85%

      - metric: evicted_keys
        description: 追い出されたキー数
        thresholds:
          warning: > 0 (5分間)
          critical: > 100 (5分間)

    replication:
      - metric: connected_replicas
        description: 接続レプリカ数
        thresholds:
          critical: < expected_count
```

#### 5.1.2 監視ダッシュボード構成

```yaml
grafana_dashboards:
  database_overview:
    panels:
      - title: Query Performance
        type: graph
        metrics:
          - pg_stat_statements execution time
          - slow query count
          - query throughput

      - title: Connection Pool
        type: gauge
        metrics:
          - active connections
          - idle connections
          - waiting connections

      - title: Replication Status
        type: stat
        metrics:
          - replication lag
          - replica status

  cache_performance:
    panels:
      - title: Cache Hit Rate
        type: graph
        metrics:
          - redis hit rate
          - cdn hit rate

      - title: Cache Latency
        type: heatmap
        metrics:
          - redis latency distribution

      - title: Memory Usage
        type: gauge
        metrics:
          - redis memory used
          - eviction rate

  query_analysis:
    panels:
      - title: Top Slow Queries
        type: table
        data_source: pg_stat_statements

      - title: Query Distribution
        type: pie
        metrics:
          - SELECT/INSERT/UPDATE/DELETE ratio

      - title: Index Usage
        type: bar
        metrics:
          - index scan vs seq scan
```

### 5.2 バックアップ・リカバリ戦略

#### 5.2.1 バックアップ構成

```yaml
backup_strategy:
  postgresql:
    continuous_backup:
      type: WAL archiving
      destination: Cloud Storage (GCS)
      retention: 35日
      encryption: AES-256

    daily_snapshot:
      type: Full backup
      schedule: "0 2 * * *"  # 毎日2:00 AM
      destination: Cloud Storage (GCS)
      retention: 35日
      encryption: AES-256

    point_in_time_recovery:
      enabled: true
      granularity: 1分
      retention: 7日

  redis:
    rdb_snapshot:
      schedule: 毎時
      retention: 24時間

    aof_persistence:
      enabled: true
      fsync: everysec

  mongodb:
    continuous_backup:
      enabled: true
      retention: 7日

    snapshot:
      schedule: 日次
      retention: 30日
```

#### 5.2.2 リカバリ手順

```yaml
recovery_procedures:
  single_table_recovery:
    steps:
      1: バックアップからテーブルデータを抽出
      2: 一時テーブルにリストア
      3: データ検証
      4: 本番テーブルにマージ
    estimated_time: 30分〜2時間

  point_in_time_recovery:
    steps:
      1: リカバリ対象時刻を特定
      2: レプリカを使用してPITRを実行
      3: データ整合性を検証
      4: 必要に応じてプライマリに昇格
    estimated_time: 15分〜1時間

  full_database_recovery:
    steps:
      1: 最新のフルバックアップを取得
      2: 新しいインスタンスにリストア
      3: WALログを適用
      4: アプリケーション接続を切り替え
    estimated_time: 2時間〜8時間

  cross_region_failover:
    steps:
      1: プライマリリージョンの障害を確認
      2: セカンダリリージョンのレプリカを昇格
      3: DNSを更新
      4: アプリケーションの接続を確認
    estimated_time: 5分〜15分
```

---

## 第6章：コスト最適化と容量計画

### 6.1 コスト分析

#### 6.1.1 データベースコスト内訳

```yaml
cost_breakdown_monthly:
  year_1:
    postgresql_cloud_sql:
      instance: $500
      storage: $100
      backup: $50
      network: $50
      total: $700

    redis_memorystore:
      instance: $300
      network: $30
      total: $330

    mongodb_atlas:
      cluster: $200
      storage: $50
      total: $250

    total_monthly: $1,280
    total_annual: $15,360

  year_3:
    postgresql_cloud_sql:
      instance: $2,000  # HA + レプリカ
      storage: $500
      backup: $200
      network: $200
      total: $2,900

    redis_memorystore:
      instance: $800
      network: $100
      total: $900

    mongodb_atlas:
      cluster: $500
      storage: $150
      total: $650

    elasticsearch:
      cluster: $1,000
      storage: $200
      total: $1,200

    bigquery:
      storage: $200
      queries: $500
      total: $700

    total_monthly: $6,350
    total_annual: $76,200
```

#### 6.1.2 コスト最適化戦略

```yaml
cost_optimization_strategies:
  right_sizing:
    description: インスタンスサイズの最適化
    actions:
      - 使用率に基づくダウンサイジング
      - リザーブドインスタンスの活用
      - スポットインスタンスの検討（非本番）
    savings_potential: 20-30%

  storage_optimization:
    description: ストレージコストの最適化
    actions:
      - 古いデータのアーカイブ
      - 圧縮の活用
      - ストレージクラスの最適化
    savings_potential: 30-50%

  query_optimization:
    description: クエリ効率の改善
    actions:
      - スロークエリの最適化
      - 不要なインデックスの削除
      - キャッシュヒット率の向上
    savings_potential: 10-20%

  data_tiering:
    description: データティアリングの実装
    actions:
      - ホット/ウォーム/コールドの分離
      - 自動データ移行
      - アクセスパターンに基づく配置
    savings_potential: 40-60%
```

### 6.2 容量計画

#### 6.2.1 成長予測モデル

```yaml
capacity_projection:
  data_growth:
    year_1:
      transactional_data: 50GB
      logs_and_events: 200GB
      analytics: 100GB
      total: 350GB
      growth_rate: 10%/月

    year_3:
      transactional_data: 500GB
      logs_and_events: 3TB
      analytics: 1.5TB
      total: 5TB
      growth_rate: 15%/月

    year_5:
      transactional_data: 2TB
      logs_and_events: 50TB
      analytics: 50TB
      total: 100TB
      growth_rate: 20%/月

  compute_requirements:
    year_1:
      postgresql:
        cpu: 8 vCPU
        memory: 32GB
        iops: 10,000

    year_3:
      postgresql:
        cpu: 32 vCPU
        memory: 128GB
        iops: 50,000
        replicas: 2

    year_5:
      postgresql:
        cpu: 64 vCPU
        memory: 256GB
        iops: 100,000
        replicas: 4
        shards: 4
```

#### 6.2.2 スケーリングトリガー

```yaml
scaling_triggers:
  vertical_scaling:
    cpu_utilization:
      threshold: 70%
      duration: 15分
      action: 次のインスタンスサイズにアップグレード

    memory_utilization:
      threshold: 80%
      duration: 15分
      action: メモリを増加

    storage_utilization:
      threshold: 70%
      action: ストレージを自動拡張

  horizontal_scaling:
    read_replica_addition:
      trigger: 読み取りレイテンシP95 > 100ms
      duration: 30分
      action: リードレプリカを追加

    shard_addition:
      trigger: 書き込みスループット > 80%容量
      planning_horizon: 3ヶ月前に計画

  cache_scaling:
    redis_memory:
      threshold: 70%
      action: インスタンスサイズを増加

    hit_rate_degradation:
      threshold: < 90%
      duration: 1時間
      action: キャッシュ戦略の見直し
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

    - document: Doc-SA-001
      title: System Architecture Overview
      relevance: 全体アーキテクチャ、サービス構成

    - document: Doc-SA-002
      title: Microservices Architecture
      relevance: サービス境界、データ分離

  outputs:
    - document: Doc-DA-003
      title: Data Pipeline & ETL Architecture
      dependency: データベース接続、データソース定義

    - document: Doc-IF-001
      title: Infrastructure Design
      dependency: データベースインスタンス構成

    - document: Doc-SC-001
      title: Security Architecture
      dependency: 暗号化、アクセス制御
```

### 7.2 技術選定サマリー

```yaml
technology_selection_summary:
  primary_database:
    technology: PostgreSQL 16
    deployment: Cloud SQL (GCP)
    use_case: トランザクションデータ、マスターデータ

  cache_layer:
    technology: Redis 7.x
    deployment: Memorystore (GCP)
    use_case: セッション、キャッシュ、レート制限

  document_store:
    technology: MongoDB 7.x
    deployment: Atlas (GCP)
    use_case: コンテンツ、設定、ユーザー生成コンテンツ

  search_engine:
    technology: Elasticsearch 8.x
    deployment: Elastic Cloud
    use_case: 全文検索、ファセット検索

  time_series:
    technology: TimescaleDB 2.x
    deployment: Timescale Cloud
    use_case: メトリクス、価格履歴、分析

  data_warehouse:
    technology: BigQuery
    deployment: GCP Native
    use_case: 分析、レポート、ML
```

---

## 付録

### A. データベース比較マトリックス

| 特性 | PostgreSQL | Redis | MongoDB | Elasticsearch | TimescaleDB |
|------|-----------|-------|---------|--------------|-------------|
| データモデル | リレーショナル | Key-Value | ドキュメント | 検索インデックス | 時系列 |
| 一貫性 | ACID | 最終整合性 | 設定可能 | 最終整合性 | ACID |
| スケーリング | 垂直 + 読み取りレプリカ | クラスター | シャーディング | シャーディング | 垂直 + パーティション |
| レイテンシ | 中 | 超低 | 低〜中 | 中 | 中 |
| スループット | 中〜高 | 超高 | 高 | 高 | 高 |
| 運用複雑性 | 低 | 低 | 中 | 高 | 低 |

### B. 接続文字列テンプレート

```yaml
connection_strings:
  postgresql:
    primary: "postgresql://user:password@primary-host:5432/triptrip?sslmode=require"
    read_replica: "postgresql://user:password@replica-host:5432/triptrip?sslmode=require"
    pgbouncer: "postgresql://user:password@pgbouncer-host:6432/triptrip?sslmode=require"

  redis:
    standalone: "redis://user:password@redis-host:6379/0"
    sentinel: "redis+sentinel://user:password@sentinel-host:26379/mymaster/0"

  mongodb:
    atlas: "mongodb+srv://user:password@cluster.mongodb.net/triptrip?retryWrites=true&w=majority"

  elasticsearch:
    cloud: "https://user:password@cluster.es.region.gcp.cloud.es.io:9243"
```

### C. 変更履歴

| バージョン | 日付 | 変更内容 | 作成者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-01-19 | 初版作成 | Technical Architecture Agent |

---

**文書情報**
- 文書ID: Doc-DA-002
- バージョン: 1.0.0
- ステータス: ドラフト
- 最終更新: 2026-01-19
- 次回レビュー: 2026-02-19

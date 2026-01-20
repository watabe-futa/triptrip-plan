# Doc-IA-001: TripTripクラウドインフラストラクチャ & デプロイメント

## エグゼクティブサマリー

本文書は、TripTripプラットフォームのクラウドインフラストラクチャとデプロイメント戦略を包括的に定義します。AWS中心のマルチクラウドアーキテクチャを採用し、1億MAU、500,000 TPSのグローバルスケールに対応可能なインフラストラクチャを設計します。コンテナオーケストレーション（EKS）、サーバーレスコンピューティング、Infrastructure as Code（Terraform）、コスト最適化戦略を詳述し、高可用性、スケーラビリティ、セキュリティを実現する堅牢な基盤を構築します。

---

## 第1章：はじめとインフラストラクチャコンテキスト

### 1.1 インフラストラクチャ要件と目標

#### 1.1.1 ビジネス要件からの導出

TripTripプラットフォームのインフラストラクチャは、以下のビジネス要件を満たす必要があります：

```yaml
business_requirements:
  scale_targets:
    year_1_2024:
      mau: 100000
      tps: 1000
      data_volume: "1TB"
      global_regions: 1

    year_3_2026:
      mau: 10000000
      tps: 50000
      data_volume: "100TB"
      global_regions: 3

    year_5_2028:
      mau: 100000000
      tps: 500000
      data_volume: "1PB"
      global_regions: 10

  availability_requirements:
    target_sla: "99.95%"
    rto: "15 minutes"
    rpo: "1 minute"
    maintenance_window: "Zero downtime deployments"

  performance_requirements:
    api_latency_p50: "50ms"
    api_latency_p99: "200ms"
    page_load_time: "2 seconds"
    search_latency: "100ms"

  compliance_requirements:
    - "PCI DSS Level 1 (決済処理)"
    - "GDPR (EU市場展開時)"
    - "個人情報保護法 (日本)"
    - "SOC 2 Type II"
```

#### 1.1.2 技術要件の定義

```yaml
technical_requirements:
  compute:
    - "水平スケーリング可能なコンテナワークロード"
    - "サーバーレス関数によるイベント処理"
    - "GPU/TPUアクセラレーション（ML推論）"
    - "エッジコンピューティング（低レイテンシ）"

  networking:
    - "マルチリージョンロードバランシング"
    - "CDNによるグローバルコンテンツ配信"
    - "プライベートネットワーク接続"
    - "DDoS保護とWAF"

  storage:
    - "高性能ブロックストレージ（データベース）"
    - "スケーラブルオブジェクトストレージ（メディア）"
    - "低レイテンシキャッシュストレージ"
    - "データレイク/データウェアハウス"

  security:
    - "ネットワーク分離とセグメンテーション"
    - "暗号化（転送中・保存時）"
    - "ID・アクセス管理"
    - "監査ログとコンプライアンス"

  observability:
    - "メトリクス収集と可視化"
    - "分散トレーシング"
    - "ログ集約と分析"
    - "アラートと自動対応"
```

### 1.2 スケーラビリティ目標

#### 1.2.1 スケーリング戦略

```yaml
scaling_strategy:
  horizontal_scaling:
    compute:
      principle: "ステートレスサービスの水平拡張"
      mechanism:
        - "Kubernetes HPA (Horizontal Pod Autoscaler)"
        - "AWS Auto Scaling Groups"
        - "Event-driven scaling (KEDA)"
      targets:
        - "CPU使用率 < 70%"
        - "メモリ使用率 < 80%"
        - "リクエストキュー長 < 100"

    database:
      principle: "読み取り負荷の分散と書き込みのシャーディング"
      mechanism:
        - "リードレプリカの自動スケール"
        - "コネクションプーリング"
        - "将来的なシャーディング対応"

    cache:
      principle: "Redis Clusterによる分散キャッシング"
      mechanism:
        - "クラスターの水平拡張"
        - "シャーディングによるデータ分散"

  vertical_scaling:
    principle: "インスタンスサイズの最適化"
    use_cases:
      - "データベースのIOPS要件増大"
      - "MLワークロードのメモリ要件"
    limitations:
      - "インスタンスサイズ上限"
      - "再起動が必要な場合あり"

  geographic_scaling:
    principle: "マルチリージョン展開によるグローバル対応"
    phases:
      phase_1: "ap-northeast-1 (Tokyo) - Primary"
      phase_2: "ap-northeast-3 (Osaka) - DR"
      phase_3: "ap-southeast-1 (Singapore) - 東南アジア"
      phase_4: "us-west-2 (Oregon) - 北米"
      phase_5: "eu-west-1 (Ireland) - 欧州"
```

### 1.3 コスト・パフォーマンス制約

#### 1.3.1 コスト目標

```yaml
cost_targets:
  unit_economics:
    cost_per_mau: "< ¥100/month"
    cost_per_transaction: "< ¥0.5"
    infrastructure_as_revenue_percentage: "< 15%"

  budget_allocation:
    year_1:
      total: "¥60M/year"
      breakdown:
        compute: "35%"
        database: "25%"
        storage: "15%"
        networking: "15%"
        observability: "10%"

    year_3:
      total: "¥300M/year"
      breakdown:
        compute: "40%"
        database: "20%"
        storage: "15%"
        networking: "15%"
        observability: "10%"

  optimization_targets:
    reserved_instance_coverage: "> 70%"
    spot_instance_usage: "> 30% (non-critical)"
    idle_resource_elimination: "< 5% waste"
```

---

## 第2章：クラウドプラットフォーム戦略

### 2.1 AWS中心アーキテクチャ

#### 2.1.1 AWSサービス選定

```yaml
aws_services:
  compute:
    primary:
      - service: "Amazon EKS"
        purpose: "Kubernetesオーケストレーション"
        configuration:
          version: "1.29"
          node_groups:
            general:
              instance_types: ["m5.xlarge", "m5.2xlarge"]
              min: 3
              max: 50
            compute_optimized:
              instance_types: ["c5.xlarge", "c5.2xlarge"]
              min: 2
              max: 30
            memory_optimized:
              instance_types: ["r5.large", "r5.xlarge"]
              min: 1
              max: 10

      - service: "AWS Fargate"
        purpose: "サーバーレスコンテナ"
        use_cases:
          - "バッチ処理"
          - "CronJob"
          - "スポットワークロード"

      - service: "AWS Lambda"
        purpose: "イベント駆動関数"
        use_cases:
          - "API Gateway バックエンド"
          - "S3イベント処理"
          - "DynamoDB Streams"
          - "スケジュール実行"

  database:
    relational:
      - service: "Amazon RDS for PostgreSQL"
        purpose: "トランザクションデータ"
        configuration:
          engine_version: "16.1"
          instance_class: "db.r5.xlarge"
          multi_az: true
          storage: "gp3"
          iops: 12000

      - service: "Amazon Aurora PostgreSQL"
        purpose: "高性能ワークロード（将来）"
        timeline: "Phase 2"

    nosql:
      - service: "Amazon DynamoDB"
        purpose: "セッション、キー値データ"
        configuration:
          billing_mode: "PAY_PER_REQUEST"
          global_tables: false

    cache:
      - service: "Amazon ElastiCache for Redis"
        purpose: "キャッシュ、セッション"
        configuration:
          engine_version: "7.1"
          node_type: "cache.r5.large"
          num_cache_clusters: 3
          cluster_mode: "enabled"

  storage:
    object:
      - service: "Amazon S3"
        purpose: "メディアファイル、バックアップ"
        configuration:
          storage_classes:
            - "S3 Standard (頻繁アクセス)"
            - "S3 Intelligent-Tiering (可変)"
            - "S3 Glacier (アーカイブ)"

    file:
      - service: "Amazon EFS"
        purpose: "共有ファイルシステム"
        use_cases:
          - "MLモデルファイル"
          - "共有設定"

  networking:
    - service: "Amazon VPC"
      purpose: "ネットワーク隔離"

    - service: "Application Load Balancer"
      purpose: "L7ロードバランシング"

    - service: "Amazon CloudFront"
      purpose: "CDN"

    - service: "AWS Global Accelerator"
      purpose: "グローバルルーティング（将来）"

  security:
    - service: "AWS WAF"
      purpose: "Webアプリケーションファイアウォール"

    - service: "AWS Shield"
      purpose: "DDoS保護"

    - service: "AWS Secrets Manager"
      purpose: "シークレット管理"

    - service: "AWS KMS"
      purpose: "暗号化キー管理"

  observability:
    - service: "Amazon CloudWatch"
      purpose: "メトリクス、ログ、アラート"

    - service: "AWS X-Ray"
      purpose: "分散トレーシング"
```

#### 2.1.2 AWSアカウント構造

```yaml
aws_account_structure:
  organization:
    management_account:
      purpose: "組織管理、一括請求"
      services:
        - "AWS Organizations"
        - "AWS SSO"
        - "AWS Control Tower"

    security_account:
      purpose: "セキュリティ監視、監査"
      services:
        - "AWS Security Hub"
        - "Amazon GuardDuty"
        - "AWS CloudTrail (Organization trail)"

    log_archive_account:
      purpose: "ログ集約、長期保存"
      services:
        - "S3 (ログバケット)"
        - "CloudWatch Logs"

    shared_services_account:
      purpose: "共有サービス、CI/CD"
      services:
        - "ECR (コンテナレジストリ)"
        - "CodePipeline"
        - "Terraform State"

    workload_accounts:
      development:
        purpose: "開発環境"
        environment: "dev"

      staging:
        purpose: "ステージング環境"
        environment: "staging"

      production:
        purpose: "本番環境"
        environment: "prod"

  cross_account_access:
    method: "IAM Roles with AssumeRole"
    authentication: "AWS SSO"
```

### 2.2 マルチクラウド考慮事項

#### 2.2.1 GCP補助利用

```yaml
gcp_usage:
  primary_services:
    bigquery:
      purpose: "データ分析、BI"
      rationale: "最高性能の分析エンジン"
      data_flow:
        - source: "AWS S3"
        - transfer: "Storage Transfer Service"
        - destination: "BigQuery"

    vertex_ai:
      purpose: "機械学習プラットフォーム"
      use_cases:
        - "モデルトレーニング"
        - "AutoML"
        - "Feature Store"

    firebase:
      purpose: "モバイルバックエンド"
      services:
        - "Firebase Authentication"
        - "Firebase Cloud Messaging (FCM)"
        - "Firebase Remote Config"

  data_synchronization:
    strategy: "イベント駆動同期"
    implementation:
      - trigger: "Kafka/EventBridge"
      - processor: "Cloud Functions"
      - destination: "BigQuery"
```

### 2.3 リージョン選定

#### 2.3.1 リージョン戦略

```yaml
region_strategy:
  primary_region:
    name: "ap-northeast-1 (Tokyo)"
    rationale:
      - "日本市場への低レイテンシ"
      - "法規制準拠（データ主権）"
      - "日本語サポート"
    services_deployed:
      - "全てのコアサービス"
      - "データベース（Primary）"
      - "キャッシュ"

  dr_region:
    name: "ap-northeast-3 (Osaka)"
    rationale:
      - "地理的分離（災害対策）"
      - "低レイテンシでのフェイルオーバー"
    services_deployed:
      - "データベース（Standby）"
      - "S3 Replication"
      - "最小限のコンピュート（スタンバイ）"

  expansion_regions:
    phase_2:
      name: "ap-southeast-1 (Singapore)"
      timeline: "2025 Q3"
      target_market: "東南アジア"

    phase_3:
      name: "us-west-2 (Oregon)"
      timeline: "2026 Q2"
      target_market: "北米"

    phase_4:
      name: "eu-west-1 (Ireland)"
      timeline: "2027 Q1"
      target_market: "欧州"

  data_residency:
    japan:
      - "個人情報"
      - "決済情報"
    replicated:
      - "商品カタログ"
      - "静的コンテンツ"
      - "匿名化分析データ"
```

---

## 第3章：コンピュートアーキテクチャ

### 3.1 コンテナ戦略（ECS/EKS）

#### 3.1.1 EKSクラスター設計

```yaml
eks_cluster_design:
  cluster_configuration:
    name: "triptrip-production"
    version: "1.29"
    endpoint_access:
      public: false
      private: true

    add_ons:
      - name: "vpc-cni"
        version: "v1.16.0"
      - name: "coredns"
        version: "v1.11.1"
      - name: "kube-proxy"
        version: "v1.29.0"
      - name: "aws-ebs-csi-driver"
        version: "v1.25.0"

  node_groups:
    general_purpose:
      name: "general"
      instance_types:
        - "m5.xlarge"
        - "m5.2xlarge"
      capacity_type: "ON_DEMAND"
      scaling:
        min: 3
        max: 50
        desired: 10
      labels:
        workload-type: "general"
      taints: []

    compute_optimized:
      name: "compute"
      instance_types:
        - "c5.xlarge"
        - "c5.2xlarge"
      capacity_type: "ON_DEMAND"
      scaling:
        min: 2
        max: 30
        desired: 5
      labels:
        workload-type: "compute"
      taints:
        - key: "workload-type"
          value: "compute"
          effect: "NoSchedule"

    memory_optimized:
      name: "memory"
      instance_types:
        - "r5.xlarge"
        - "r5.2xlarge"
      capacity_type: "ON_DEMAND"
      scaling:
        min: 1
        max: 10
        desired: 2
      labels:
        workload-type: "memory"
      taints:
        - key: "workload-type"
          value: "memory"
          effect: "NoSchedule"

    spot_instances:
      name: "spot"
      instance_types:
        - "m5.xlarge"
        - "m5.2xlarge"
        - "m5a.xlarge"
        - "m5a.2xlarge"
      capacity_type: "SPOT"
      scaling:
        min: 0
        max: 20
        desired: 5
      labels:
        workload-type: "spot"
      taints:
        - key: "spot"
          value: "true"
          effect: "NoSchedule"
```

#### 3.1.2 Kubernetes名前空間設計

```yaml
kubernetes_namespaces:
  system:
    - name: "kube-system"
      purpose: "Kubernetesシステムコンポーネント"

    - name: "monitoring"
      purpose: "監視・可観測性ツール"
      components:
        - "prometheus"
        - "grafana"
        - "alertmanager"

    - name: "ingress"
      purpose: "Ingressコントローラ"
      components:
        - "aws-load-balancer-controller"
        - "external-dns"

    - name: "cert-manager"
      purpose: "証明書管理"

  application:
    - name: "triptrip-api"
      purpose: "APIサービス"
      resource_quotas:
        requests.cpu: "10"
        requests.memory: "20Gi"
        limits.cpu: "20"
        limits.memory: "40Gi"
        pods: "50"

    - name: "triptrip-web"
      purpose: "Webフロントエンド"
      resource_quotas:
        requests.cpu: "5"
        requests.memory: "10Gi"
        limits.cpu: "10"
        limits.memory: "20Gi"
        pods: "30"

    - name: "triptrip-workers"
      purpose: "バックグラウンドワーカー"
      resource_quotas:
        requests.cpu: "8"
        requests.memory: "16Gi"
        limits.cpu: "16"
        limits.memory: "32Gi"
        pods: "40"

    - name: "triptrip-ml"
      purpose: "機械学習サービス"
      resource_quotas:
        requests.cpu: "4"
        requests.memory: "32Gi"
        limits.cpu: "8"
        limits.memory: "64Gi"
        pods: "20"
```

#### 3.1.3 Kubernetes マニフェスト例

```yaml
# API Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: triptrip-api
  namespace: triptrip-api
  labels:
    app: triptrip-api
    version: v1
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  selector:
    matchLabels:
      app: triptrip-api
  template:
    metadata:
      labels:
        app: triptrip-api
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: triptrip-api
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
        - name: api
          image: 123456789.dkr.ecr.ap-northeast-1.amazonaws.com/triptrip-api:v1.2.3
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
              name: http
              protocol: TCP
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
          env:
            - name: NODE_ENV
              value: "production"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: database-credentials
                  key: url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: redis-credentials
                  key: url
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: triptrip-api
                topologyKey: kubernetes.io/hostname
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: triptrip-api
---
# HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: triptrip-api-hpa
  namespace: triptrip-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: triptrip-api
  minReplicas: 5
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
        - type: Pods
          value: 5
          periodSeconds: 60
      selectPolicy: Max
```

### 3.2 サーバーレス活用（Lambda）

#### 3.2.1 Lambda活用戦略

```yaml
lambda_strategy:
  use_cases:
    event_processing:
      - name: "S3イベント処理"
        triggers:
          - "s3:ObjectCreated:*"
        functions:
          - "image-resize"
          - "file-validation"
          - "metadata-extraction"

      - name: "DynamoDB Streams"
        triggers:
          - "DynamoDB Stream"
        functions:
          - "sync-to-elasticsearch"
          - "audit-logging"

      - name: "SQSメッセージ処理"
        triggers:
          - "SQS Queue"
        functions:
          - "email-sender"
          - "notification-processor"
          - "report-generator"

    api_backend:
      - name: "軽量API"
        triggers:
          - "API Gateway"
        functions:
          - "health-check"
          - "public-config"
          - "webhook-receiver"

    scheduled_tasks:
      - name: "定期実行タスク"
        triggers:
          - "EventBridge Schedule"
        functions:
          - "data-cleanup"
          - "cache-warm"
          - "report-generation"

  configuration:
    runtime: "nodejs20.x"
    architecture: "arm64"
    memory:
      default: 256
      image_processing: 1024
      data_processing: 512
    timeout:
      default: 30
      long_running: 900
    provisioned_concurrency:
      critical_functions: 10
```

#### 3.2.2 Lambda関数設計例

```typescript
// Lambda関数: 画像リサイズ
import { S3Client, GetObjectCommand, PutObjectCommand } from '@aws-sdk/client-s3';
import sharp from 'sharp';
import { Context, S3Event } from 'aws-lambda';

const s3Client = new S3Client({ region: process.env.AWS_REGION });

interface ResizeConfig {
  width: number;
  height: number;
  suffix: string;
}

const RESIZE_CONFIGS: ResizeConfig[] = [
  { width: 1200, height: 800, suffix: 'large' },
  { width: 600, height: 400, suffix: 'medium' },
  { width: 300, height: 200, suffix: 'small' },
  { width: 150, height: 150, suffix: 'thumbnail' },
];

export async function handler(event: S3Event, context: Context): Promise<void> {
  console.log('Processing S3 event', JSON.stringify(event, null, 2));

  for (const record of event.Records) {
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, ' '));

    // オリジナル画像の取得
    const originalImage = await getObject(bucket, key);

    // 各サイズにリサイズ
    await Promise.all(
      RESIZE_CONFIGS.map(async (config) => {
        const resizedImage = await resizeImage(originalImage, config);
        const newKey = generateResizedKey(key, config.suffix);
        await putObject(bucket, newKey, resizedImage);
        console.log(`Resized image saved: ${newKey}`);
      })
    );

    console.log(`Successfully processed: ${key}`);
  }
}

async function getObject(bucket: string, key: string): Promise<Buffer> {
  const command = new GetObjectCommand({ Bucket: bucket, Key: key });
  const response = await s3Client.send(command);
  return Buffer.from(await response.Body!.transformToByteArray());
}

async function resizeImage(image: Buffer, config: ResizeConfig): Promise<Buffer> {
  return sharp(image)
    .resize(config.width, config.height, {
      fit: 'cover',
      position: 'center',
    })
    .webp({ quality: 85 })
    .toBuffer();
}

async function putObject(bucket: string, key: string, body: Buffer): Promise<void> {
  const command = new PutObjectCommand({
    Bucket: bucket,
    Key: key,
    Body: body,
    ContentType: 'image/webp',
    CacheControl: 'max-age=31536000',
  });
  await s3Client.send(command);
}

function generateResizedKey(originalKey: string, suffix: string): string {
  const parts = originalKey.split('/');
  const filename = parts.pop()!;
  const nameWithoutExt = filename.substring(0, filename.lastIndexOf('.'));
  return [...parts, 'resized', `${nameWithoutExt}_${suffix}.webp`].join('/');
}
```

### 3.3 オートスケーリング設計

#### 3.3.1 スケーリングポリシー

```yaml
autoscaling_policies:
  eks_cluster_autoscaler:
    configuration:
      expander: "least-waste"
      scale_down_delay_after_add: "10m"
      scale_down_unneeded_time: "10m"
      skip_nodes_with_local_storage: false

  horizontal_pod_autoscaler:
    api_services:
      min_replicas: 5
      max_replicas: 50
      metrics:
        - type: "cpu"
          target: 70
        - type: "memory"
          target: 80
        - type: "custom/requests_per_second"
          target: 1000

    web_services:
      min_replicas: 3
      max_replicas: 30
      metrics:
        - type: "cpu"
          target: 70

    worker_services:
      min_replicas: 2
      max_replicas: 20
      metrics:
        - type: "custom/queue_length"
          target: 100

  event_driven_scaling:
    keda:
      triggers:
        - type: "kafka"
          metadata:
            topic: "events"
            consumerGroup: "triptrip-consumer"
            lagThreshold: "100"

        - type: "rabbitmq"
          metadata:
            queueName: "task-queue"
            queueLength: "50"

  database_scaling:
    rds:
      storage_autoscaling:
        enabled: true
        max_allocated_storage: 1000
        threshold_percent: 10

    elasticache:
      replica_autoscaling:
        enabled: true
        min_replicas: 1
        max_replicas: 5
        target_tracking:
          metric: "ReplicaEngineCPUUtilization"
          target: 70
```

---

## 第4章：ネットワークアーキテクチャ

### 4.1 VPC設計

#### 4.1.1 VPCアーキテクチャ

```yaml
vpc_architecture:
  production_vpc:
    cidr: "10.0.0.0/16"
    region: "ap-northeast-1"
    availability_zones:
      - "ap-northeast-1a"
      - "ap-northeast-1c"
      - "ap-northeast-1d"

    subnets:
      public:
        - name: "public-1a"
          cidr: "10.0.1.0/24"
          az: "ap-northeast-1a"
          purpose: "NAT Gateway, ALB"

        - name: "public-1c"
          cidr: "10.0.2.0/24"
          az: "ap-northeast-1c"
          purpose: "NAT Gateway, ALB"

        - name: "public-1d"
          cidr: "10.0.3.0/24"
          az: "ap-northeast-1d"
          purpose: "NAT Gateway, ALB"

      private_app:
        - name: "private-app-1a"
          cidr: "10.0.10.0/24"
          az: "ap-northeast-1a"
          purpose: "EKS Nodes, App Servers"

        - name: "private-app-1c"
          cidr: "10.0.11.0/24"
          az: "ap-northeast-1c"
          purpose: "EKS Nodes, App Servers"

        - name: "private-app-1d"
          cidr: "10.0.12.0/24"
          az: "ap-northeast-1d"
          purpose: "EKS Nodes, App Servers"

      private_data:
        - name: "private-data-1a"
          cidr: "10.0.20.0/24"
          az: "ap-northeast-1a"
          purpose: "RDS, ElastiCache"

        - name: "private-data-1c"
          cidr: "10.0.21.0/24"
          az: "ap-northeast-1c"
          purpose: "RDS, ElastiCache"

        - name: "private-data-1d"
          cidr: "10.0.22.0/24"
          az: "ap-northeast-1d"
          purpose: "RDS, ElastiCache"

    nat_gateways:
      - az: "ap-northeast-1a"
        elastic_ip: true
      - az: "ap-northeast-1c"
        elastic_ip: true
      - az: "ap-northeast-1d"
        elastic_ip: true

    vpc_endpoints:
      gateway:
        - "s3"
        - "dynamodb"
      interface:
        - "ecr.api"
        - "ecr.dkr"
        - "ec2"
        - "sts"
        - "logs"
        - "monitoring"
        - "secretsmanager"
```

#### 4.1.2 セキュリティグループ設計

```yaml
security_groups:
  alb_sg:
    name: "triptrip-alb-sg"
    description: "ALB Security Group"
    ingress:
      - protocol: "tcp"
        port: 443
        source: "0.0.0.0/0"
        description: "HTTPS from Internet"
      - protocol: "tcp"
        port: 80
        source: "0.0.0.0/0"
        description: "HTTP redirect"
    egress:
      - protocol: "-1"
        port: -1
        destination: "0.0.0.0/0"

  eks_nodes_sg:
    name: "triptrip-eks-nodes-sg"
    description: "EKS Node Security Group"
    ingress:
      - protocol: "tcp"
        port: 443
        source: "eks-cluster-sg"
        description: "API Server to Nodes"
      - protocol: "tcp"
        port: "1025-65535"
        source: "self"
        description: "Node to Node"
      - protocol: "tcp"
        port: "30000-32767"
        source: "alb-sg"
        description: "NodePort from ALB"
    egress:
      - protocol: "-1"
        port: -1
        destination: "0.0.0.0/0"

  rds_sg:
    name: "triptrip-rds-sg"
    description: "RDS Security Group"
    ingress:
      - protocol: "tcp"
        port: 5432
        source: "eks-nodes-sg"
        description: "PostgreSQL from EKS"
    egress: []

  elasticache_sg:
    name: "triptrip-elasticache-sg"
    description: "ElastiCache Security Group"
    ingress:
      - protocol: "tcp"
        port: 6379
        source: "eks-nodes-sg"
        description: "Redis from EKS"
    egress: []
```

### 4.2 ロードバランシング（ALB/NLB）

#### 4.2.1 ALB設計

```yaml
load_balancer_design:
  external_alb:
    name: "triptrip-external-alb"
    scheme: "internet-facing"
    type: "application"

    listeners:
      https:
        port: 443
        protocol: "HTTPS"
        ssl_policy: "ELBSecurityPolicy-TLS13-1-2-2021-06"
        certificates:
          - "arn:aws:acm:ap-northeast-1:123456789:certificate/xxx"
        default_action:
          type: "forward"
          target_group: "triptrip-api-tg"

      http:
        port: 80
        protocol: "HTTP"
        default_action:
          type: "redirect"
          redirect:
            protocol: "HTTPS"
            port: "443"
            status_code: "HTTP_301"

    target_groups:
      api:
        name: "triptrip-api-tg"
        port: 3000
        protocol: "HTTP"
        target_type: "ip"
        health_check:
          path: "/health"
          interval: 30
          timeout: 5
          healthy_threshold: 2
          unhealthy_threshold: 3
        stickiness:
          enabled: false

      web:
        name: "triptrip-web-tg"
        port: 3000
        protocol: "HTTP"
        target_type: "ip"

    rules:
      - priority: 1
        conditions:
          - field: "path-pattern"
            values: ["/api/*"]
        actions:
          - type: "forward"
            target_group: "triptrip-api-tg"

      - priority: 2
        conditions:
          - field: "path-pattern"
            values: ["/*"]
        actions:
          - type: "forward"
            target_group: "triptrip-web-tg"

    access_logs:
      enabled: true
      bucket: "triptrip-alb-logs"
      prefix: "external-alb"

    waf:
      enabled: true
      web_acl_arn: "arn:aws:wafv2:..."
```

### 4.3 CDN・エッジキャッシング（CloudFront）

#### 4.3.1 CloudFront設計

```yaml
cloudfront_distribution:
  main_distribution:
    aliases:
      - "www.triptrip.com"
      - "triptrip.com"
    price_class: "PriceClass_200"
    http_version: "http2and3"

    origins:
      alb_origin:
        domain_name: "triptrip-alb-xxx.ap-northeast-1.elb.amazonaws.com"
        origin_id: "alb-origin"
        custom_origin_config:
          http_port: 80
          https_port: 443
          protocol_policy: "https-only"
          ssl_protocols: ["TLSv1.2"]
        origin_shield:
          enabled: true
          region: "ap-northeast-1"

      s3_origin:
        domain_name: "triptrip-static-assets.s3.ap-northeast-1.amazonaws.com"
        origin_id: "s3-static"
        origin_access_control:
          enabled: true

    cache_behaviors:
      default:
        target_origin: "alb-origin"
        viewer_protocol_policy: "redirect-to-https"
        allowed_methods:
          - "GET"
          - "HEAD"
          - "OPTIONS"
          - "PUT"
          - "POST"
          - "PATCH"
          - "DELETE"
        cached_methods:
          - "GET"
          - "HEAD"
        cache_policy: "CachingDisabled"
        origin_request_policy: "AllViewer"
        compress: true

      static_assets:
        path_pattern: "/static/*"
        target_origin: "s3-static"
        viewer_protocol_policy: "redirect-to-https"
        cache_policy: "CachingOptimized"
        compress: true
        ttl:
          default: 86400
          max: 31536000
          min: 0

      images:
        path_pattern: "/images/*"
        target_origin: "s3-static"
        viewer_protocol_policy: "redirect-to-https"
        cache_policy: "CachingOptimized"
        response_headers_policy: "CORS-with-preflight"
        compress: true

    functions:
      viewer_request:
        - name: "security-headers"
          event_type: "viewer-response"
          code: |
            function handler(event) {
              var response = event.response;
              var headers = response.headers;
              headers['strict-transport-security'] = {value: 'max-age=63072000'};
              headers['x-content-type-options'] = {value: 'nosniff'};
              headers['x-frame-options'] = {value: 'DENY'};
              headers['x-xss-protection'] = {value: '1; mode=block'};
              return response;
            }

    logging:
      enabled: true
      bucket: "triptrip-cloudfront-logs.s3.amazonaws.com"
      prefix: "cdn/"
      include_cookies: false
```

---

## 第5章：Infrastructure as Code

### 5.1 Terraform設計パターン

#### 5.1.1 プロジェクト構造

```
terraform/
├── modules/                           # 再利用可能モジュール
│   ├── networking/
│   │   ├── vpc/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── README.md
│   │   ├── subnets/
│   │   ├── nat_gateway/
│   │   ├── security_groups/
│   │   └── vpc_endpoints/
│   │
│   ├── compute/
│   │   ├── eks/
│   │   │   ├── main.tf
│   │   │   ├── node_groups.tf
│   │   │   ├── addons.tf
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   ├── lambda/
│   │   └── ec2/
│   │
│   ├── database/
│   │   ├── rds/
│   │   ├── elasticache/
│   │   └── dynamodb/
│   │
│   ├── storage/
│   │   ├── s3/
│   │   └── efs/
│   │
│   ├── cdn/
│   │   └── cloudfront/
│   │
│   └── monitoring/
│       ├── cloudwatch/
│       └── sns/
│
├── environments/                      # 環境別設定
│   ├── dev/
│   │   ├── terragrunt.hcl
│   │   ├── networking/
│   │   │   └── terragrunt.hcl
│   │   ├── compute/
│   │   │   └── terragrunt.hcl
│   │   └── database/
│   │       └── terragrunt.hcl
│   │
│   ├── staging/
│   │   └── ...
│   │
│   └── production/
│       └── ...
│
├── terragrunt.hcl                    # ルートTerragrunt設定
└── common.tfvars                     # 共通変数
```

#### 5.1.2 Terraformモジュール例

```hcl
# modules/compute/eks/main.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.25"
    }
  }
}

# EKS Cluster
resource "aws_eks_cluster" "main" {
  name     = var.cluster_name
  version  = var.kubernetes_version
  role_arn = aws_iam_role.cluster.arn

  vpc_config {
    subnet_ids              = var.subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = var.endpoint_public_access
    security_group_ids      = [aws_security_group.cluster.id]
  }

  encryption_config {
    provider {
      key_arn = var.kms_key_arn
    }
    resources = ["secrets"]
  }

  enabled_cluster_log_types = [
    "api",
    "audit",
    "authenticator",
    "controllerManager",
    "scheduler"
  ]

  tags = merge(var.tags, {
    Name = var.cluster_name
  })

  depends_on = [
    aws_iam_role_policy_attachment.cluster_policy,
    aws_iam_role_policy_attachment.vpc_resource_controller,
  ]
}

# EKS Node Groups
resource "aws_eks_node_group" "main" {
  for_each = var.node_groups

  cluster_name    = aws_eks_cluster.main.name
  node_group_name = each.key
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = var.node_subnet_ids

  instance_types = each.value.instance_types
  capacity_type  = each.value.capacity_type

  scaling_config {
    min_size     = each.value.min_size
    max_size     = each.value.max_size
    desired_size = each.value.desired_size
  }

  update_config {
    max_unavailable_percentage = 33
  }

  labels = each.value.labels

  dynamic "taint" {
    for_each = each.value.taints
    content {
      key    = taint.value.key
      value  = taint.value.value
      effect = taint.value.effect
    }
  }

  launch_template {
    id      = aws_launch_template.node[each.key].id
    version = aws_launch_template.node[each.key].latest_version
  }

  tags = merge(var.tags, {
    Name = "${var.cluster_name}-${each.key}"
  })

  depends_on = [
    aws_iam_role_policy_attachment.node_policy,
    aws_iam_role_policy_attachment.cni_policy,
    aws_iam_role_policy_attachment.ecr_policy,
  ]

  lifecycle {
    ignore_changes = [scaling_config[0].desired_size]
  }
}

# Launch Template for Node Groups
resource "aws_launch_template" "node" {
  for_each = var.node_groups

  name_prefix = "${var.cluster_name}-${each.key}-"

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = each.value.disk_size
      volume_type           = "gp3"
      iops                  = 3000
      throughput            = 125
      encrypted             = true
      kms_key_id            = var.kms_key_arn
      delete_on_termination = true
    }
  }

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 1
    instance_metadata_tags      = "enabled"
  }

  monitoring {
    enabled = true
  }

  tag_specifications {
    resource_type = "instance"
    tags = merge(var.tags, {
      Name = "${var.cluster_name}-${each.key}"
    })
  }

  tags = var.tags
}

# EKS Addons
resource "aws_eks_addon" "vpc_cni" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "vpc-cni"
  addon_version               = var.vpc_cni_version
  resolve_conflicts_on_update = "OVERWRITE"

  configuration_values = jsonencode({
    env = {
      ENABLE_PREFIX_DELEGATION = "true"
      WARM_PREFIX_TARGET       = "1"
    }
  })

  tags = var.tags
}

resource "aws_eks_addon" "coredns" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "coredns"
  addon_version               = var.coredns_version
  resolve_conflicts_on_update = "OVERWRITE"

  tags = var.tags

  depends_on = [aws_eks_node_group.main]
}

resource "aws_eks_addon" "kube_proxy" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "kube-proxy"
  addon_version               = var.kube_proxy_version
  resolve_conflicts_on_update = "OVERWRITE"

  tags = var.tags
}

resource "aws_eks_addon" "ebs_csi_driver" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "aws-ebs-csi-driver"
  addon_version               = var.ebs_csi_version
  service_account_role_arn    = aws_iam_role.ebs_csi.arn
  resolve_conflicts_on_update = "OVERWRITE"

  tags = var.tags
}
```

### 5.2 環境分離戦略

#### 5.2.1 Terragrunt設定

```hcl
# environments/production/terragrunt.hcl

include "root" {
  path = find_in_parent_folders()
}

locals {
  environment = "production"
  region      = "ap-northeast-1"

  common_tags = {
    Environment = local.environment
    Project     = "triptrip"
    ManagedBy   = "terraform"
  }
}

inputs = {
  environment = local.environment
  aws_region  = local.region
  tags        = local.common_tags
}

# 依存関係の定義
dependency "networking" {
  config_path = "../networking"
}

dependency "database" {
  config_path = "../database"
}
```

```hcl
# environments/production/compute/terragrunt.hcl

include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../modules/compute/eks"
}

dependency "networking" {
  config_path = "../networking"
}

inputs = {
  cluster_name       = "triptrip-production"
  kubernetes_version = "1.29"

  subnet_ids      = dependency.networking.outputs.private_app_subnet_ids
  node_subnet_ids = dependency.networking.outputs.private_app_subnet_ids

  endpoint_public_access = false

  node_groups = {
    general = {
      instance_types = ["m5.xlarge", "m5.2xlarge"]
      capacity_type  = "ON_DEMAND"
      min_size       = 3
      max_size       = 50
      desired_size   = 10
      disk_size      = 100
      labels = {
        "workload-type" = "general"
      }
      taints = []
    }

    compute = {
      instance_types = ["c5.xlarge", "c5.2xlarge"]
      capacity_type  = "ON_DEMAND"
      min_size       = 2
      max_size       = 30
      desired_size   = 5
      disk_size      = 100
      labels = {
        "workload-type" = "compute"
      }
      taints = [
        {
          key    = "workload-type"
          value  = "compute"
          effect = "NO_SCHEDULE"
        }
      ]
    }

    spot = {
      instance_types = ["m5.xlarge", "m5.2xlarge", "m5a.xlarge", "m5a.2xlarge"]
      capacity_type  = "SPOT"
      min_size       = 0
      max_size       = 20
      desired_size   = 5
      disk_size      = 100
      labels = {
        "workload-type" = "spot"
      }
      taints = [
        {
          key    = "spot"
          value  = "true"
          effect = "NO_SCHEDULE"
        }
      ]
    }
  }
}
```

### 5.3 シークレット管理

#### 5.3.1 Secrets Manager統合

```yaml
secrets_management:
  strategy: "AWS Secrets Manager + External Secrets Operator"

  secrets_structure:
    database:
      name: "triptrip/production/database"
      keys:
        - "host"
        - "port"
        - "username"
        - "password"
        - "database"
      rotation:
        enabled: true
        schedule: "rate(30 days)"

    redis:
      name: "triptrip/production/redis"
      keys:
        - "host"
        - "port"
        - "auth_token"

    api_keys:
      name: "triptrip/production/api-keys"
      keys:
        - "stripe_secret_key"
        - "sendgrid_api_key"
        - "firebase_credentials"

  kubernetes_integration:
    external_secrets:
      enabled: true
      refresh_interval: "1h"
      template: |
        apiVersion: external-secrets.io/v1beta1
        kind: ExternalSecret
        metadata:
          name: database-credentials
          namespace: triptrip-api
        spec:
          refreshInterval: 1h
          secretStoreRef:
            name: aws-secrets-manager
            kind: ClusterSecretStore
          target:
            name: database-credentials
            creationPolicy: Owner
          data:
            - secretKey: url
              remoteRef:
                key: triptrip/production/database
                property: url
```

---

## 第6章：コスト最適化

### 6.1 リザーブドインスタンス戦略

#### 6.1.1 予約購入計画

```yaml
reserved_instances:
  compute:
    ec2_savings_plans:
      term: "1 year"
      payment: "Partial Upfront"
      coverage_target: 70%
      commitment:
        - type: "Compute Savings Plans"
          hourly_commitment: "$50/hour"
          estimated_savings: "35%"

    eks_reserved:
      strategy: "Savings Plans cover EKS nodes"

  database:
    rds:
      term: "1 year"
      payment: "Partial Upfront"
      instances:
        - instance_type: "db.r5.xlarge"
          quantity: 2
          multi_az: true

    elasticache:
      term: "1 year"
      payment: "No Upfront"
      nodes:
        - node_type: "cache.r5.large"
          quantity: 3

  estimated_savings:
    annual_savings: "¥15M"
    savings_percentage: "25%"
```

### 6.2 スポットインスタンス活用

#### 6.2.1 Spot戦略

```yaml
spot_strategy:
  workloads:
    batch_processing:
      spot_percentage: 100%
      interruption_handling:
        - "Graceful shutdown on 2-min warning"
        - "Checkpointing to S3"

    development_environments:
      spot_percentage: 80%

    non_critical_workers:
      spot_percentage: 50%
      fallback: "ON_DEMAND"

  diversification:
    instance_families:
      - "m5"
      - "m5a"
      - "m5n"
      - "m6i"
    availability_zones:
      - "ap-northeast-1a"
      - "ap-northeast-1c"
      - "ap-northeast-1d"

  interruption_handling:
    eks:
      - "Node termination handler daemonset"
      - "Pod disruption budgets"
      - "Graceful node drain"

  estimated_savings:
    average_discount: "60-70%"
    monthly_savings: "¥2M"
```

### 6.3 コスト監視・アラート

#### 6.3.1 コスト監視設計

```yaml
cost_monitoring:
  aws_cost_explorer:
    enabled: true
    granularity: "DAILY"
    dimensions:
      - "SERVICE"
      - "USAGE_TYPE"
      - "LINKED_ACCOUNT"
      - "REGION"

  budgets:
    total_monthly:
      amount: "¥5,000,000"
      alerts:
        - threshold: 50%
          notification: "email"
        - threshold: 80%
          notification: "slack"
        - threshold: 100%
          notification: "pagerduty"

    per_service:
      - service: "Amazon EC2"
        budget: "¥1,750,000"
      - service: "Amazon RDS"
        budget: "¥1,250,000"
      - service: "Amazon S3"
        budget: "¥500,000"

  anomaly_detection:
    enabled: true
    services:
      - "All Services"
    threshold: "20%"
    notification: "slack"

  tagging_strategy:
    required_tags:
      - "Environment"
      - "Service"
      - "Team"
      - "CostCenter"
    enforcement: "AWS Config Rules"

  reporting:
    frequency: "weekly"
    recipients:
      - "engineering@triptrip.com"
      - "finance@triptrip.com"
    contents:
      - "Cost breakdown by service"
      - "Month-over-month comparison"
      - "Savings recommendations"
      - "Reserved instance utilization"
```

---

## 第7章：文書間参照と統合ポイント

### 7.1 関連文書マッピング

```yaml
document_references:
  upstream:
    - doc_id: "Doc-TV-001"
      title: "技術ビジョンとアーキテクチャ原則"
      relationship: "インフラストラクチャの基盤原則"

    - doc_id: "Doc-TV-002"
      title: "技術スタック選定"
      relationship: "クラウドプラットフォーム選定の詳細"

    - doc_id: "Doc-SA-001"
      title: "システムアーキテクチャ概要"
      relationship: "全体アーキテクチャとの整合性"

  downstream:
    - doc_id: "Doc-IA-002"
      title: "ネットワーク・セキュリティ設計"
      relationship: "詳細なネットワーク設計"

    - doc_id: "Doc-IA-003"
      title: "災害復旧・事業継続計画"
      relationship: "DR/BCPの詳細"

    - doc_id: "Doc-DO-001"
      title: "DevOps・CI/CD戦略"
      relationship: "デプロイメント自動化"

  cross_references:
    - doc_id: "Doc-SC-001"
      title: "セキュリティアーキテクチャ"
      relationship: "セキュリティ要件との整合"
```

### 7.2 インフラストラクチャ決定サマリー

```yaml
infrastructure_summary:
  cloud_platform:
    primary: "AWS (ap-northeast-1)"
    dr: "AWS (ap-northeast-3)"
    secondary: "GCP (BigQuery, Vertex AI)"

  compute:
    container_orchestration: "Amazon EKS 1.29"
    serverless: "AWS Lambda"
    node_groups:
      - "General Purpose (m5)"
      - "Compute Optimized (c5)"
      - "Memory Optimized (r5)"
      - "Spot Instances"

  database:
    primary: "Amazon RDS PostgreSQL 16"
    cache: "Amazon ElastiCache Redis 7.1"
    nosql: "Amazon DynamoDB"

  networking:
    vpc: "10.0.0.0/16"
    load_balancer: "Application Load Balancer"
    cdn: "Amazon CloudFront"

  iac:
    tool: "Terraform + Terragrunt"
    state: "S3 + DynamoDB"
    environments:
      - "dev"
      - "staging"
      - "production"

  cost_optimization:
    savings_plans: "70% coverage"
    spot_instances: "30% of non-critical"
    estimated_savings: "25%"
```

---

## 結論

本文書は、TripTripプラットフォームのクラウドインフラストラクチャとデプロイメント戦略を包括的に定義しました。

### 主要な設計決定

1. **AWS中心アーキテクチャ**: 成熟したエコシステムと日本リージョンの活用
2. **EKSベースのコンテナオーケストレーション**: スケーラビリティと運用効率
3. **マルチAZ/マルチリージョン設計**: 高可用性と災害復旧
4. **Infrastructure as Code**: Terraformによる再現性と自動化
5. **コスト最適化**: Savings Plans、Spot Instances、継続的モニタリング

### 期待される成果

- 99.95%の可用性SLA達成
- 100msの平均APIレイテンシ
- 1億MAUへのスケーラビリティ
- 25%のコスト最適化
- ゼロダウンタイムデプロイメント

これらの設計により、TripTripは高可用性、高パフォーマンス、コスト効率の高いインフラストラクチャ基盤を実現します。

---

**文書情報**
- Document ID: Doc-IA-001
- Version: 1.0.0
- Last Updated: 2026-01-20
- Status: Draft
- Total Lines: 1,500+
- Author: Technical Architecture Team

**関連文書**
- Doc-TV-001: 技術ビジョンとアーキテクチャ原則
- Doc-TV-002: 技術スタック選定
- Doc-SA-001: システムアーキテクチャ概要
- Doc-IA-002: ネットワーク・セキュリティ設計

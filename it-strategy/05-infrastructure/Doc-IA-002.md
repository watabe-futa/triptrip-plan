# Doc-IA-002: TripTrip Kubernetes & コンテナ戦略

## エグゼクティブサマリー

本文書は、TripTripプラットフォームのKubernetesおよびコンテナオーケストレーション戦略を包括的に定義します。Amazon EKSを基盤としたKubernetesクラスタアーキテクチャ、ワークロード分散、オートスケーリング、サービスメッシュ（Istio）、セキュリティポリシー、GitOpsデプロイメントパターン（ArgoCD）を詳述します。Google、Netflix、Spotifyレベルのコンテナオーケストレーションベストプラクティスを採用し、1億MAU、500,000 TPSのスケールに対応可能な、高可用性、高性能、運用効率の高いコンテナ基盤を構築します。本設計は、Doc-IA-001で定義されたクラウドインフラストラクチャを基盤とし、マイクロサービスアーキテクチャの効率的な運用を実現します。

---

## 第1章：はじめに & コンテキスト

### 1.1 ビジネス要件 & 制約

#### 1.1.1 コンテナ化要件

```yaml
containerization_requirements:
  business_drivers:
    agility:
      description: 迅速な機能リリースと実験
      requirements:
        - デプロイ頻度: 日次〜複数回/日
        - リリースサイクル: 2週間スプリント
        - ロールバック時間: < 5分

    scalability:
      description: トラフィック変動への対応
      requirements:
        - 自動スケーリング: 5分以内
        - 季節変動: 10倍スケール
        - イベント対応: 瞬時スケール

    reliability:
      description: 高可用性とフォールトトレランス
      requirements:
        - 可用性: 99.95%
        - RTO: 15分
        - ゼロダウンタイムデプロイ

    cost_efficiency:
      description: コスト最適化
      requirements:
        - リソース利用率: > 60%
        - Spot活用: 30%以上（非クリティカル）
        - 右サイジング: 四半期レビュー

  technical_requirements:
    workload_types:
      stateless:
        - API Services
        - Web Frontends
        - Background Workers
      stateful:
        - Redis Clusters
        - Kafka（必要時）
      batch:
        - ETL Jobs
        - Report Generation
        - ML Training

    traffic_patterns:
      baseline_tps: 1,000
      peak_tps: 50,000
      burst_tps: 100,000
      growth_projection:
        year_1: 1,000 TPS
        year_3: 50,000 TPS
        year_5: 500,000 TPS
```

#### 1.1.2 既存システムとの整合性

```yaml
existing_system_alignment:
  current_stack:
    backend:
      runtime: Node.js 18+
      framework: Hono
      containerization: Docker
      base_image: node:18-alpine

    build_system:
      tool: Docker Build / Buildx
      registry: Amazon ECR
      ci_cd: GitHub Actions

  migration_approach:
    strategy: Lift and Shift → Optimize
    phases:
      phase_1: 既存コンテナのEKS移行
      phase_2: Kubernetes-native最適化
      phase_3: サービスメッシュ導入
```

### 1.2 技術目標 & 成功基準

#### 1.2.1 パフォーマンス目標

```yaml
performance_targets:
  pod_operations:
    startup_time:
      cold_start: < 30秒
      warm_start: < 10秒
    scaling_time:
      scale_up: < 60秒
      scale_down: < 300秒（安定化後）

  cluster_operations:
    node_provisioning: < 5分
    rolling_update: ゼロダウンタイム
    failover: < 30秒

  resource_efficiency:
    cpu_utilization: 60-70%（平均）
    memory_utilization: 70-80%（平均）
    pod_density: 30-50 pods/node

  network_performance:
    pod_to_pod_latency: < 1ms（同一AZ）
    service_mesh_overhead: < 2ms
```

### 1.3 主要な前提条件 & トレードオフ

#### 1.3.1 設計原則

```yaml
design_principles:
  1_cattle_not_pets:
    description: 使い捨て可能なインフラストラクチャ
    implementation:
      - Immutable Infrastructure
      - 自動復旧
      - ステートレス設計優先

  2_declarative_configuration:
    description: 宣言的な設定管理
    implementation:
      - GitOps
      - Infrastructure as Code
      - Policy as Code

  3_least_privilege:
    description: 最小権限の原則
    implementation:
      - Pod Security Standards
      - Network Policies
      - RBAC

  4_defense_in_depth:
    description: 多層防御
    implementation:
      - ネットワーク分離
      - サービスメッシュmTLS
      - ランタイムセキュリティ
```

---

## 第2章：Kubernetesクラスタ設計

### 2.1 クラスタアーキテクチャ

#### 2.1.1 クラスタトポロジー

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         EKS CLUSTER ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      AWS Managed Control Plane                           │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │    │
│  │   │  API Server  │  │   etcd       │  │  Controllers │                 │    │
│  │   │  (HA: 3 AZ)  │  │  (Encrypted) │  │  (Managed)   │                 │    │
│  │   └──────────────┘  └──────────────┘  └──────────────┘                 │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                    │                                             │
│                         VPC CNI / Pod Networking                                │
│                                    │                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                           Data Plane                                     │    │
│  │                                                                          │    │
│  │   ┌───────────────────────────────────────────────────────────────────┐ │    │
│  │   │  AZ-a (ap-northeast-1a)                                           │ │    │
│  │   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │ │    │
│  │   │  │ Node Group  │  │ Node Group  │  │ Node Group  │               │ │    │
│  │   │  │  General    │  │  Compute    │  │   Spot      │               │ │    │
│  │   │  └─────────────┘  └─────────────┘  └─────────────┘               │ │    │
│  │   └───────────────────────────────────────────────────────────────────┘ │    │
│  │                                                                          │    │
│  │   ┌───────────────────────────────────────────────────────────────────┐ │    │
│  │   │  AZ-c (ap-northeast-1c)                                           │ │    │
│  │   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │ │    │
│  │   │  │ Node Group  │  │ Node Group  │  │ Node Group  │               │ │    │
│  │   │  │  General    │  │  Compute    │  │   Spot      │               │ │    │
│  │   │  └─────────────┘  └─────────────┘  └─────────────┘               │ │    │
│  │   └───────────────────────────────────────────────────────────────────┘ │    │
│  │                                                                          │    │
│  │   ┌───────────────────────────────────────────────────────────────────┐ │    │
│  │   │  AZ-d (ap-northeast-1d)                                           │ │    │
│  │   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │ │    │
│  │   │  │ Node Group  │  │ Node Group  │  │ Node Group  │               │ │    │
│  │   │  │  General    │  │  Compute    │  │   Spot      │               │ │    │
│  │   │  └─────────────┘  └─────────────┘  └─────────────┘               │ │    │
│  │   └───────────────────────────────────────────────────────────────────┘ │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 2.1.2 EKSクラスタ構成

```yaml
eks_cluster_configuration:
  cluster:
    name: triptrip-production
    version: "1.29"
    region: ap-northeast-1

    endpoint_access:
      public: false
      private: true
      public_access_cidrs: []

    encryption:
      secrets:
        enabled: true
        kms_key_arn: arn:aws:kms:ap-northeast-1:xxx:key/xxx

    logging:
      enabled_types:
        - api
        - audit
        - authenticator
        - controllerManager
        - scheduler
      retention_days: 90

    networking:
      vpc_id: vpc-xxx
      subnet_ids:
        - subnet-private-app-1a
        - subnet-private-app-1c
        - subnet-private-app-1d
      security_group_ids:
        - sg-eks-cluster

    addons:
      vpc_cni:
        version: v1.16.0
        configuration:
          ENABLE_PREFIX_DELEGATION: "true"
          WARM_PREFIX_TARGET: "1"
          POD_SECURITY_GROUP_ENFORCING_MODE: "standard"

      coredns:
        version: v1.11.1
        replicas: 3
        affinity: spread_across_azs

      kube_proxy:
        version: v1.29.0
        mode: iptables

      aws_ebs_csi_driver:
        version: v1.25.0
        service_account:
          annotations:
            eks.amazonaws.com/role-arn: arn:aws:iam::xxx:role/ebs-csi-role

      aws_load_balancer_controller:
        version: v2.7.0
        service_account:
          annotations:
            eks.amazonaws.com/role-arn: arn:aws:iam::xxx:role/alb-controller-role
```

### 2.2 ノード構成とサイジング

#### 2.2.1 ノードグループ設計

```yaml
node_groups:
  general_purpose:
    name: general
    description: 汎用ワークロード用
    instance_types:
      - m5.xlarge    # 4 vCPU, 16 GiB
      - m5.2xlarge   # 8 vCPU, 32 GiB
      - m5a.xlarge   # 4 vCPU, 16 GiB（代替）
      - m5a.2xlarge  # 8 vCPU, 32 GiB（代替）
    capacity_type: ON_DEMAND
    scaling:
      min_size: 3
      max_size: 50
      desired_size: 10
    disk:
      size_gb: 100
      type: gp3
      iops: 3000
      throughput: 125
      encrypted: true
    labels:
      workload-type: general
      tier: standard
    taints: []
    update_config:
      max_unavailable_percentage: 33

  compute_optimized:
    name: compute
    description: CPU集約型ワークロード用
    instance_types:
      - c5.xlarge    # 4 vCPU, 8 GiB
      - c5.2xlarge   # 8 vCPU, 16 GiB
      - c5a.xlarge
      - c5a.2xlarge
    capacity_type: ON_DEMAND
    scaling:
      min_size: 2
      max_size: 30
      desired_size: 5
    disk:
      size_gb: 100
      type: gp3
    labels:
      workload-type: compute
      tier: compute
    taints:
      - key: workload-type
        value: compute
        effect: NoSchedule

  memory_optimized:
    name: memory
    description: メモリ集約型ワークロード用
    instance_types:
      - r5.xlarge    # 4 vCPU, 32 GiB
      - r5.2xlarge   # 8 vCPU, 64 GiB
      - r5a.xlarge
      - r5a.2xlarge
    capacity_type: ON_DEMAND
    scaling:
      min_size: 1
      max_size: 10
      desired_size: 2
    disk:
      size_gb: 200
      type: gp3
    labels:
      workload-type: memory
      tier: memory
    taints:
      - key: workload-type
        value: memory
        effect: NoSchedule

  spot_instances:
    name: spot
    description: 非クリティカルワークロード用
    instance_types:
      - m5.xlarge
      - m5.2xlarge
      - m5a.xlarge
      - m5a.2xlarge
      - m5n.xlarge
      - m5n.2xlarge
      - m6i.xlarge
      - m6i.2xlarge
    capacity_type: SPOT
    scaling:
      min_size: 0
      max_size: 30
      desired_size: 5
    disk:
      size_gb: 100
      type: gp3
    labels:
      workload-type: spot
      tier: spot
      lifecycle: spot
    taints:
      - key: spot
        value: "true"
        effect: NoSchedule
    spot_allocation_strategy: capacity-optimized-prioritized
    interruption_handler: enabled

  gpu_workloads:
    name: gpu
    description: ML推論ワークロード用
    instance_types:
      - g4dn.xlarge  # 1 GPU, 4 vCPU, 16 GiB
      - g4dn.2xlarge # 1 GPU, 8 vCPU, 32 GiB
    capacity_type: ON_DEMAND
    scaling:
      min_size: 0
      max_size: 5
      desired_size: 0
    labels:
      workload-type: gpu
      nvidia.com/gpu: "true"
    taints:
      - key: nvidia.com/gpu
        value: "true"
        effect: NoSchedule
    ami_type: AL2_x86_64_GPU
```

### 2.3 マルチクラスタ戦略

#### 2.3.1 クラスタ分離戦略

```yaml
multi_cluster_strategy:
  cluster_topology:
    production:
      name: triptrip-production
      purpose: 本番ワークロード
      environment: production
      criticality: critical

    staging:
      name: triptrip-staging
      purpose: ステージング検証
      environment: staging
      criticality: high

    development:
      name: triptrip-dev
      purpose: 開発・テスト
      environment: development
      criticality: medium

  cross_cluster_communication:
    method: AWS PrivateLink / VPC Peering
    service_discovery: Cloud Map

  disaster_recovery:
    strategy: Active-Passive
    dr_cluster:
      name: triptrip-dr
      region: ap-northeast-3 (Osaka)
      activation: Manual/Automated
    replication:
      - Database: RDS Cross-Region Read Replica
      - S3: Cross-Region Replication
      - Secrets: Multi-Region Secrets Manager
```

---

## 第3章：ワークロード管理

### 3.1 Deployment戦略

#### 3.1.1 Deploymentパターン

```yaml
# API Service Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: triptrip-api
  namespace: triptrip-api
  labels:
    app: triptrip-api
    app.kubernetes.io/name: triptrip-api
    app.kubernetes.io/version: v1.2.3
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: triptrip
spec:
  replicas: 5
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  selector:
    matchLabels:
      app: triptrip-api
  template:
    metadata:
      labels:
        app: triptrip-api
        version: v1.2.3
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
        sidecar.istio.io/inject: "true"
    spec:
      serviceAccountName: triptrip-api
      terminationGracePeriodSeconds: 60

      # Pod Security Context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault

      # Init Containers
      initContainers:
        - name: wait-for-db
          image: busybox:1.36
          command: ['sh', '-c', 'until nc -z postgres-service 5432; do sleep 2; done']
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]

      containers:
        - name: api
          image: 123456789.dkr.ecr.ap-northeast-1.amazonaws.com/triptrip-api:v1.2.3
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 3000
              protocol: TCP
            - name: metrics
              containerPort: 9090
              protocol: TCP

          # Resource Management
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2000m"
              memory: "2Gi"

          # Security Context
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]

          # Environment Variables
          env:
            - name: NODE_ENV
              value: "production"
            - name: PORT
              value: "3000"
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName

          # Environment from Secrets
          envFrom:
            - secretRef:
                name: triptrip-api-secrets
            - configMapRef:
                name: triptrip-api-config

          # Health Checks
          startupProbe:
            httpGet:
              path: /health/startup
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 30

          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3

          # Lifecycle Hooks
          lifecycle:
            preStop:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - "sleep 10"

          # Volume Mounts
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: cache
              mountPath: /app/.cache

      volumes:
        - name: tmp
          emptyDir:
            medium: Memory
            sizeLimit: 100Mi
        - name: cache
          emptyDir:
            sizeLimit: 500Mi

      # Pod Scheduling
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: triptrip-api
                topologyKey: kubernetes.io/hostname
            - weight: 50
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: triptrip-api
                topologyKey: topology.kubernetes.io/zone

      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: triptrip-api
        - maxSkew: 2
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: triptrip-api

      # Node Selection
      nodeSelector:
        workload-type: general

      tolerations: []
```

### 3.2 StatefulSets設計

#### 3.2.1 StatefulSetパターン

```yaml
# Redis Cluster StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
  namespace: triptrip-cache
spec:
  serviceName: redis-cluster
  replicas: 6
  podManagementPolicy: Parallel
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      serviceAccountName: redis-cluster
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000

      containers:
        - name: redis
          image: redis:7.2-alpine
          command:
            - redis-server
          args:
            - /etc/redis/redis.conf
            - --cluster-enabled
            - "yes"
            - --cluster-config-file
            - /data/nodes.conf
          ports:
            - name: redis
              containerPort: 6379
            - name: cluster-bus
              containerPort: 16379
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2000m"
              memory: "4Gi"
          volumeMounts:
            - name: data
              mountPath: /data
            - name: config
              mountPath: /etc/redis
          readinessProbe:
            exec:
              command:
                - redis-cli
                - ping
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            exec:
              command:
                - redis-cli
                - ping
            initialDelaySeconds: 30
            periodSeconds: 10

      volumes:
        - name: config
          configMap:
            name: redis-config

      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: redis-cluster
              topologyKey: kubernetes.io/hostname
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: workload-type
                    operator: In
                    values:
                      - memory
                      - general

  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: gp3-encrypted
        resources:
          requests:
            storage: 50Gi
```

### 3.3 DaemonSets活用

#### 3.3.1 DaemonSetユースケース

```yaml
# Fluent Bit Log Collector DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: monitoring
  labels:
    app: fluent-bit
spec:
  selector:
    matchLabels:
      app: fluent-bit
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: fluent-bit
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "2020"
    spec:
      serviceAccountName: fluent-bit
      priorityClassName: system-node-critical

      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:2.2
          ports:
            - containerPort: 2020
              name: metrics
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: config
              mountPath: /fluent-bit/etc/
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: CLUSTER_NAME
              value: triptrip-production

      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: config
          configMap:
            name: fluent-bit-config

      tolerations:
        - operator: Exists

---
# Node Problem Detector DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-problem-detector
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: node-problem-detector
  template:
    metadata:
      labels:
        app: node-problem-detector
    spec:
      serviceAccountName: node-problem-detector
      priorityClassName: system-node-critical
      hostNetwork: true
      hostPID: true

      containers:
        - name: node-problem-detector
          image: registry.k8s.io/node-problem-detector/node-problem-detector:v0.8.14
          command:
            - /node-problem-detector
            - --logtostderr
            - --config.system-log-monitor=/config/kernel-monitor.json,/config/docker-monitor.json
          resources:
            requests:
              cpu: "10m"
              memory: "80Mi"
            limits:
              cpu: "100m"
              memory: "200Mi"
          securityContext:
            privileged: true
          volumeMounts:
            - name: log
              mountPath: /var/log
              readOnly: true
            - name: kmsg
              mountPath: /dev/kmsg
              readOnly: true
            - name: config
              mountPath: /config
              readOnly: true

      volumes:
        - name: log
          hostPath:
            path: /var/log
        - name: kmsg
          hostPath:
            path: /dev/kmsg
        - name: config
          configMap:
            name: node-problem-detector-config

      tolerations:
        - operator: Exists
```

---

## 第4章：スケーリング & リソース管理

### 4.1 Horizontal Pod Autoscaler

#### 4.1.1 HPA設計

```yaml
# API Service HPA
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
  maxReplicas: 100
  metrics:
    # CPU利用率
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70

    # メモリ利用率
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80

    # カスタムメトリクス: リクエスト/秒
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"

    # 外部メトリクス: SQSキュー長
    - type: External
      external:
        metric:
          name: sqs_approximate_number_of_messages_visible
          selector:
            matchLabels:
              queue_name: triptrip-task-queue
        target:
          type: Value
          value: "100"

  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
        - type: Pods
          value: 5
          periodSeconds: 60
      selectPolicy: Min

    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
        - type: Pods
          value: 10
          periodSeconds: 15
      selectPolicy: Max
```

### 4.2 Vertical Pod Autoscaler

#### 4.2.1 VPA設計

```yaml
# VPA for API Service
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: triptrip-api-vpa
  namespace: triptrip-api
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: triptrip-api
  updatePolicy:
    updateMode: "Auto"  # Auto, Recreate, Initial, Off
  resourcePolicy:
    containerPolicies:
      - containerName: api
        minAllowed:
          cpu: "250m"
          memory: "256Mi"
        maxAllowed:
          cpu: "4"
          memory: "8Gi"
        controlledResources:
          - cpu
          - memory
        controlledValues: RequestsAndLimits

---
# VPA for Background Workers (推奨のみ)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: triptrip-worker-vpa
  namespace: triptrip-workers
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: triptrip-worker
  updatePolicy:
    updateMode: "Off"  # 推奨のみ、自動適用なし
  resourcePolicy:
    containerPolicies:
      - containerName: worker
        minAllowed:
          cpu: "100m"
          memory: "128Mi"
        maxAllowed:
          cpu: "2"
          memory: "4Gi"
```

### 4.3 Cluster Autoscaler

#### 4.3.1 Cluster Autoscaler設定

```yaml
# Cluster Autoscaler Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-config
  namespace: kube-system
data:
  config.yaml: |
    clusterName: triptrip-production
    nodeGroups:
      - name: general
        minSize: 3
        maxSize: 50
      - name: compute
        minSize: 2
        maxSize: 30
      - name: memory
        minSize: 1
        maxSize: 10
      - name: spot
        minSize: 0
        maxSize: 30

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      priorityClassName: system-cluster-critical
      containers:
        - name: cluster-autoscaler
          image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.29.0
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/triptrip-production
            - --balance-similar-node-groups
            - --scale-down-enabled=true
            - --scale-down-delay-after-add=10m
            - --scale-down-delay-after-delete=0s
            - --scale-down-delay-after-failure=3m
            - --scale-down-unneeded-time=10m
            - --scale-down-unready-time=20m
            - --scale-down-utilization-threshold=0.5
            - --max-graceful-termination-sec=600
            - --max-node-provision-time=15m
          resources:
            requests:
              cpu: "100m"
              memory: "300Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
          env:
            - name: AWS_REGION
              value: ap-northeast-1
```

### 4.4 KEDA (Kubernetes Event-driven Autoscaling)

#### 4.4.1 KEDA ScaledObject

```yaml
# KEDA ScaledObject for Queue Processing
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: triptrip-queue-worker
  namespace: triptrip-workers
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: triptrip-queue-worker
  pollingInterval: 15
  cooldownPeriod: 300
  minReplicaCount: 1
  maxReplicaCount: 50
  fallback:
    failureThreshold: 3
    replicas: 5
  triggers:
    # SQSキューベース
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.ap-northeast-1.amazonaws.com/123456789/triptrip-task-queue
        queueLength: "10"
        awsRegion: ap-northeast-1
      authenticationRef:
        name: keda-aws-credentials

    # Kafkaベース
    - type: kafka
      metadata:
        bootstrapServers: kafka-cluster:9092
        consumerGroup: triptrip-consumer
        topic: booking-events
        lagThreshold: "100"

    # Redis Streamベース
    - type: redis-streams
      metadata:
        address: redis-cluster:6379
        stream: events
        consumerGroup: triptrip-workers
        pendingEntriesCount: "50"

---
# ScaledJob for Batch Processing
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: triptrip-batch-processor
  namespace: triptrip-batch
spec:
  jobTargetRef:
    parallelism: 1
    completions: 1
    backoffLimit: 3
    template:
      spec:
        containers:
          - name: processor
            image: triptrip/batch-processor:latest
            resources:
              requests:
                cpu: "1"
                memory: "2Gi"
        restartPolicy: Never
  pollingInterval: 30
  maxReplicaCount: 20
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.ap-northeast-1.amazonaws.com/123456789/triptrip-batch-queue
        queueLength: "1"
```

---

## 第5章：サービスメッシュ

### 5.1 Istio/Linkerd選定

#### 5.1.1 サービスメッシュ比較と選定

```yaml
service_mesh_comparison:
  istio:
    pros:
      - 豊富な機能セット
      - 強力なトラフィック管理
      - 成熟したエコシステム
      - Envoyベース
    cons:
      - リソースオーバーヘッド
      - 複雑な設定
      - 学習コスト
    use_case: 複雑なトラフィック管理が必要な場合

  linkerd:
    pros:
      - 軽量
      - シンプルな設定
      - 低レイテンシ
      - Rustベース
    cons:
      - 機能が限定的
      - エコシステムが小さい
    use_case: シンプルなmTLSと観測性が必要な場合

  decision:
    selected: Istio
    rationale:
      - 複雑なトラフィック管理要件
      - カナリアデプロイメント
      - 詳細な可観測性
      - 将来的なマルチクラスタ対応
```

#### 5.1.2 Istio設定

```yaml
# Istio Installation Profile
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: triptrip-istio
  namespace: istio-system
spec:
  profile: default
  tag: 1.20.0

  meshConfig:
    accessLogFile: /dev/stdout
    accessLogFormat: |
      [%START_TIME%] "%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%"
      %RESPONSE_CODE% %RESPONSE_FLAGS% %BYTES_RECEIVED% %BYTES_SENT%
      %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% "%REQ(X-FORWARDED-FOR)%"
      "%REQ(USER-AGENT)%" "%REQ(X-REQUEST-ID)%" "%REQ(:AUTHORITY)%"
      "%UPSTREAM_HOST%" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS%
      %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS%
      %REQUESTED_SERVER_NAME% %ROUTE_NAME%

    defaultConfig:
      tracing:
        sampling: 100.0
        zipkin:
          address: jaeger-collector.monitoring:9411
      holdApplicationUntilProxyStarts: true

    enableAutoMtls: true
    trustDomain: cluster.local

    outboundTrafficPolicy:
      mode: REGISTRY_ONLY

  components:
    pilot:
      k8s:
        resources:
          requests:
            cpu: "500m"
            memory: "2Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
        hpaSpec:
          minReplicas: 2
          maxReplicas: 5
        replicaCount: 2

    ingressGateways:
      - name: istio-ingressgateway
        enabled: true
        k8s:
          service:
            type: ClusterIP  # ALBで終端
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2"
              memory: "1Gi"
          hpaSpec:
            minReplicas: 3
            maxReplicas: 10

    egressGateways:
      - name: istio-egressgateway
        enabled: true
        k8s:
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

  values:
    global:
      proxy:
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
      tracer:
        zipkin:
          address: jaeger-collector.monitoring:9411
```

### 5.2 トラフィック管理

#### 5.2.1 VirtualService / DestinationRule

```yaml
# VirtualService for API
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: triptrip-api
  namespace: triptrip-api
spec:
  hosts:
    - triptrip-api
    - api.triptrip.com
  gateways:
    - istio-system/triptrip-gateway
  http:
    # カナリアルーティング
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: triptrip-api
            subset: canary
          weight: 100

    # 通常ルーティング
    - route:
        - destination:
            host: triptrip-api
            subset: stable
          weight: 95
        - destination:
            host: triptrip-api
            subset: canary
          weight: 5

      # タイムアウト設定
      timeout: 30s

      # リトライ設定
      retries:
        attempts: 3
        perTryTimeout: 10s
        retryOn: connect-failure,refused-stream,unavailable,cancelled,retriable-4xx,5xx
        retryRemoteLocalities: true

      # フォールトインジェクション（テスト用）
      # fault:
      #   delay:
      #     percentage:
      #       value: 0.1
      #     fixedDelay: 5s

---
# DestinationRule for API
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: triptrip-api
  namespace: triptrip-api
spec:
  host: triptrip-api
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1000
        connectTimeout: 10s
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 1000
        http2MaxRequests: 1000
        maxRequestsPerConnection: 100
        maxRetries: 3

    loadBalancer:
      simple: LEAST_REQUEST
      localityLbSetting:
        enabled: true
        failover:
          - from: ap-northeast-1a
            to: ap-northeast-1c
          - from: ap-northeast-1c
            to: ap-northeast-1d

    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 30

    tls:
      mode: ISTIO_MUTUAL

  subsets:
    - name: stable
      labels:
        version: stable
    - name: canary
      labels:
        version: canary
```

### 5.3 オブザーバビリティ

#### 5.3.1 Istio メトリクス/トレーシング

```yaml
# Telemetry Configuration
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: triptrip-telemetry
  namespace: istio-system
spec:
  accessLogging:
    - providers:
        - name: envoy
      filter:
        expression: response.code >= 400 || duration > 1000

  tracing:
    - providers:
        - name: jaeger
      randomSamplingPercentage: 100
      customTags:
        environment:
          literal:
            value: production
        version:
          header:
            name: x-api-version
            defaultValue: unknown

  metrics:
    - providers:
        - name: prometheus
      overrides:
        - match:
            metric: REQUEST_COUNT
          tagOverrides:
            request_protocol:
              operation: REMOVE
        - match:
            metric: REQUEST_DURATION
          tagOverrides:
            request_protocol:
              operation: REMOVE

---
# ServiceMonitor for Istio
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istio-mesh
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: istiod
  namespaceSelector:
    matchNames:
      - istio-system
  endpoints:
    - port: http-monitoring
      interval: 15s
```

---

## 第6章：セキュリティ

### 6.1 Pod Security

#### 6.1.1 Pod Security Standards

```yaml
# Namespace with Pod Security Standards
apiVersion: v1
kind: Namespace
metadata:
  name: triptrip-api
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.29
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
# PodSecurityPolicy (レガシー、PSA移行済み)
# Pod Security Admission (PSA) を使用

---
# RuntimeClass for gVisor
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
scheduling:
  nodeSelector:
    runtime: gvisor
```

### 6.2 Network Policy

#### 6.2.1 NetworkPolicy設計

```yaml
# Default Deny All
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: triptrip-api
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
# Allow API Service Traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-traffic
  namespace: triptrip-api
spec:
  podSelector:
    matchLabels:
      app: triptrip-api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Istio Ingressからの許可
    - from:
        - namespaceSelector:
            matchLabels:
              name: istio-system
          podSelector:
            matchLabels:
              app: istio-ingressgateway
      ports:
        - protocol: TCP
          port: 3000

    # 同一Namespace内のPodからの許可
    - from:
        - podSelector: {}
      ports:
        - protocol: TCP
          port: 3000

    # Prometheus Scrape
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9090

  egress:
    # DNSへの許可
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53

    # PostgreSQLへの許可
    - to:
        - namespaceSelector:
            matchLabels:
              name: triptrip-db
          podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432

    # Redisへの許可
    - to:
        - namespaceSelector:
            matchLabels:
              name: triptrip-cache
          podSelector:
            matchLabels:
              app: redis-cluster
      ports:
        - protocol: TCP
          port: 6379
        - protocol: TCP
          port: 16379

    # 外部HTTPSへの許可
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443
```

### 6.3 Secret管理

#### 6.3.1 External Secrets Operator

```yaml
# ClusterSecretStore for AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-northeast-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets

---
# ExternalSecret for Database Credentials
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
    template:
      type: Opaque
      engineVersion: v2
      data:
        DATABASE_URL: "postgresql://{{ .username }}:{{ .password }}@{{ .host }}:{{ .port }}/{{ .database }}?sslmode=require"
  data:
    - secretKey: username
      remoteRef:
        key: triptrip/production/database
        property: username
    - secretKey: password
      remoteRef:
        key: triptrip/production/database
        property: password
    - secretKey: host
      remoteRef:
        key: triptrip/production/database
        property: host
    - secretKey: port
      remoteRef:
        key: triptrip/production/database
        property: port
    - secretKey: database
      remoteRef:
        key: triptrip/production/database
        property: database

---
# Sealed Secrets (代替)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: api-secrets
  namespace: triptrip-api
spec:
  encryptedData:
    API_KEY: AgBOQOoh7... # kubeseal で暗号化
```

---

## 第7章：CI/CD & GitOps

### 7.1 Helmチャート管理

#### 7.1.1 Helmチャート構造

```
charts/
├── triptrip-api/
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── values-staging.yaml
│   ├── values-production.yaml
│   ├── templates/
│   │   ├── _helpers.tpl
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── hpa.yaml
│   │   ├── pdb.yaml
│   │   ├── serviceaccount.yaml
│   │   ├── configmap.yaml
│   │   ├── externalsecret.yaml
│   │   ├── networkpolicy.yaml
│   │   ├── virtualservice.yaml
│   │   ├── destinationrule.yaml
│   │   └── servicemonitor.yaml
│   └── tests/
│       └── test-connection.yaml
└── triptrip-common/
    └── ... (共通テンプレート)
```

#### 7.1.2 Helm values設計

```yaml
# values-production.yaml
replicaCount: 5

image:
  repository: 123456789.dkr.ecr.ap-northeast-1.amazonaws.com/triptrip-api
  tag: ""  # CI/CDで上書き
  pullPolicy: Always

serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/triptrip-api-role

resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2000m"
    memory: "2Gi"

autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 100
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

podDisruptionBudget:
  enabled: true
  minAvailable: "50%"

nodeSelector:
  workload-type: general

tolerations: []

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: triptrip-api
          topologyKey: kubernetes.io/hostname

ingress:
  enabled: true
  className: alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /health
  hosts:
    - host: api.triptrip.com
      paths:
        - path: /
          pathType: Prefix

istio:
  enabled: true
  virtualService:
    enabled: true
  destinationRule:
    enabled: true

monitoring:
  serviceMonitor:
    enabled: true
    interval: 15s

externalSecrets:
  enabled: true
  secretStoreRef: aws-secrets-manager
  secrets:
    - name: database-credentials
      remoteRef: triptrip/production/database
    - name: redis-credentials
      remoteRef: triptrip/production/redis
```

### 7.2 ArgoCD/Flux統合

#### 7.2.1 ArgoCD Application設計

```yaml
# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: triptrip-api
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: triptrip
  source:
    repoURL: https://github.com/triptrip/k8s-manifests.git
    targetRevision: HEAD
    path: charts/triptrip-api
    helm:
      releaseName: triptrip-api
      valueFiles:
        - values.yaml
        - values-production.yaml
      parameters:
        - name: image.tag
          value: $ARGOCD_APP_REVISION

  destination:
    server: https://kubernetes.default.svc
    namespace: triptrip-api

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
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

---
# ArgoCD ApplicationSet for Multiple Environments
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: triptrip-services
  namespace: argocd
spec:
  generators:
    - matrix:
        generators:
          # サービス一覧
          - list:
              elements:
                - service: triptrip-api
                  path: charts/triptrip-api
                - service: triptrip-web
                  path: charts/triptrip-web
                - service: triptrip-worker
                  path: charts/triptrip-worker
          # 環境一覧
          - list:
              elements:
                - env: staging
                  cluster: https://eks-staging.example.com
                - env: production
                  cluster: https://eks-production.example.com

  template:
    metadata:
      name: "{{service}}-{{env}}"
      namespace: argocd
    spec:
      project: triptrip
      source:
        repoURL: https://github.com/triptrip/k8s-manifests.git
        targetRevision: HEAD
        path: "{{path}}"
        helm:
          valueFiles:
            - values.yaml
            - "values-{{env}}.yaml"
      destination:
        server: "{{cluster}}"
        namespace: "{{service}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### 7.3 デプロイメントパイプライン

#### 7.3.1 GitHub Actions ワークフロー

```yaml
# .github/workflows/deploy.yaml
name: Build and Deploy

on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main

env:
  AWS_REGION: ap-northeast-1
  ECR_REGISTRY: 123456789.dkr.ecr.ap-northeast-1.amazonaws.com
  IMAGE_NAME: triptrip-api

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.ECR_REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=ref,event=branch
            type=ref,event=pr

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            NODE_ENV=production

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: staging

    steps:
      - name: Checkout k8s-manifests
        uses: actions/checkout@v4
        with:
          repository: triptrip/k8s-manifests
          token: ${{ secrets.GH_PAT }}

      - name: Update image tag
        run: |
          cd charts/triptrip-api
          yq e '.image.tag = "${{ needs.build.outputs.image_tag }}"' -i values-staging.yaml

      - name: Commit and push
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add .
          git commit -m "Deploy ${{ env.IMAGE_NAME }}:${{ needs.build.outputs.image_tag }} to staging"
          git push

  deploy-production:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Checkout k8s-manifests
        uses: actions/checkout@v4
        with:
          repository: triptrip/k8s-manifests
          token: ${{ secrets.GH_PAT }}

      - name: Update image tag
        run: |
          cd charts/triptrip-api
          yq e '.image.tag = "${{ needs.build.outputs.image_tag }}"' -i values-production.yaml

      - name: Commit and push
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add .
          git commit -m "Deploy ${{ env.IMAGE_NAME }}:${{ needs.build.outputs.image_tag }} to production"
          git push
```

---

## 第8章：戦略的 & 技術的示唆

### 8.1 アーキテクチャ決定記録

```yaml
architecture_decisions:
  adr_001_eks_over_ecs:
    title: ECSではなくEKSを選択
    status: Accepted
    context: コンテナオーケストレーションプラットフォームの選定
    decision: Amazon EKSを採用
    rationale:
      - Kubernetesエコシステムの活用
      - ポータビリティ（マルチクラウド対応）
      - 豊富なオープンソースツール
      - 人材市場の充実
    consequences:
      - 運用複雑性の増加
      - 学習コスト
      - 高度なカスタマイズ性

  adr_002_istio_service_mesh:
    title: サービスメッシュにIstioを選択
    status: Accepted
    context: サービス間通信の管理とセキュリティ
    decision: Istioを採用
    rationale:
      - mTLSによるゼロトラストセキュリティ
      - 高度なトラフィック管理
      - 可観測性の向上
    consequences:
      - リソースオーバーヘッド
      - 複雑性の増加

  adr_003_gitops_argocd:
    title: GitOpsにArgoCDを選択
    status: Accepted
    context: デプロイメント管理の自動化
    decision: ArgoCDを採用
    rationale:
      - 宣言的な設定管理
      - Git履歴による監査
      - 自動同期と自己修復
    consequences:
      - Gitリポジトリの管理負荷
      - ブランチ戦略の整備が必要
```

### 8.2 推奨事項

```yaml
recommendations:
  immediate:
    - EKSクラスタの初期構築
    - 基本的なノードグループ設定
    - 監視・ログ基盤の構築
    - GitOpsパイプラインの確立

  short_term:
    - Istioサービスメッシュ導入
    - 自動スケーリングの最適化
    - セキュリティポリシーの強化
    - コスト最適化（Spot活用）

  long_term:
    - マルチクラスタ構成
    - ゼロトラストアーキテクチャ完成
    - 高度な可観測性
    - カオスエンジニアリング
```

---

## 第9章：実装ロードマップ

### 9.1 フェーズ別実装計画

```yaml
implementation_phases:
  phase_1_foundation:
    timeline: 月1-2
    deliverables:
      - EKSクラスタ構築
      - 基本ノードグループ設定
      - VPC CNI設定
      - 基本的なデプロイメント
    success_criteria:
      - APIサービスのデプロイ成功
      - 基本的なスケーリング動作

  phase_2_operations:
    timeline: 月3-4
    deliverables:
      - HPA/VPA設定
      - Cluster Autoscaler
      - 監視・アラート設定
      - GitOps（ArgoCD）
    success_criteria:
      - 自動スケーリング動作
      - GitOpsデプロイメント成功

  phase_3_service_mesh:
    timeline: 月5-6
    deliverables:
      - Istio導入
      - mTLS有効化
      - トラフィック管理
      - 分散トレーシング
    success_criteria:
      - サービス間mTLS
      - カナリアデプロイメント

  phase_4_optimization:
    timeline: 月7-12
    deliverables:
      - Spot Instance最適化
      - KEDA導入
      - 高度なセキュリティ
      - マルチクラスタ準備
    success_criteria:
      - コスト最適化達成
      - セキュリティ監査合格
```

---

## 第10章：文書間参照 & 依存関係

### 10.1 関連文書

```yaml
document_references:
  inputs:
    - document: Doc-IA-001
      title: Cloud Infrastructure & Deployment
      relevance: 基盤インフラストラクチャ

    - document: Doc-SA-001
      title: System Architecture Overview
      relevance: 全体アーキテクチャ

    - document: Doc-SA-003
      title: Microservices Design
      relevance: マイクロサービス設計

  outputs:
    - document: Doc-DO-001
      title: DevOps Strategy
      dependency: CI/CDパイプライン

    - document: Doc-SC-001
      title: Security Architecture
      dependency: コンテナセキュリティ

    - document: Doc-QA-001
      title: Quality Assurance
      dependency: テスト戦略
```

### 10.2 技術スタックサマリー

```yaml
technology_stack_summary:
  orchestration:
    platform: Amazon EKS 1.29
    addons:
      - VPC CNI
      - CoreDNS
      - kube-proxy
      - EBS CSI Driver
      - ALB Controller

  service_mesh:
    platform: Istio 1.20
    features:
      - mTLS
      - Traffic Management
      - Observability

  scaling:
    horizontal: HPA, KEDA
    vertical: VPA
    cluster: Cluster Autoscaler

  gitops:
    platform: ArgoCD
    strategy: ApplicationSet

  security:
    pod_security: PSA (Restricted)
    network: Network Policy
    secrets: External Secrets Operator
```

---

## 付録

### A. Kubernetesリソースクォータ

| Namespace | CPU Request | CPU Limit | Memory Request | Memory Limit | Pods |
|-----------|-------------|-----------|----------------|--------------|------|
| triptrip-api | 10 | 40 | 20Gi | 80Gi | 100 |
| triptrip-web | 5 | 20 | 10Gi | 40Gi | 50 |
| triptrip-workers | 10 | 40 | 20Gi | 80Gi | 100 |
| triptrip-ml | 8 | 32 | 64Gi | 128Gi | 30 |

### B. 変更履歴

| バージョン | 日付 | 変更内容 | 作成者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-01-20 | 初版作成 | Technical Architecture Agent |

---

**文書情報**
- 文書ID: Doc-IA-002
- バージョン: 1.0.0
- ステータス: ドラフト
- 最終更新: 2026-01-20
- 次回レビュー: 2026-02-20

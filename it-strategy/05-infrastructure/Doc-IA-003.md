# Doc-IA-003: TripTripネットワーキング＆CDNアーキテクチャ

## エグゼクティブサマリー

本文書は、TripTripプラットフォームのネットワーキングおよびCDN（Content Delivery Network）アーキテクチャを包括的に定義します。AWS VPCを基盤としたマルチAZネットワーク設計、サブネット分離戦略、L4/L7ロードバランシング、Amazon CloudFrontによるグローバルコンテンツ配信、Route 53による高度なDNS管理、AWS Shield/WAFによるDDoS保護とセキュリティ、ゼロトラストネットワークアーキテクチャを詳述します。Google、Netflix、Amazonレベルのネットワーク設計ベストプラクティスを採用し、100msの低レイテンシ、99.99%の高可用性、1億MAUのグローバルスケールに対応可能な、高性能でセキュアなネットワーク基盤を構築します。本設計は、Doc-IA-001で定義されたクラウドインフラストラクチャおよびDoc-IA-002のKubernetes戦略と整合し、エンドツーエンドのネットワーク最適化を実現します。

---

## 第1章：はじめに & コンテキスト

### 1.1 目的 & スコープ

#### 1.1.1 文書の目的

```yaml
document_purpose:
  primary_objectives:
    - グローバルスケールのネットワークアーキテクチャ設計
    - 高パフォーマンス・低レイテンシの実現
    - セキュリティとコンプライアンスの確保
    - 運用効率とコスト最適化

  scope:
    included:
      - VPC設計とネットワークセグメンテーション
      - ロードバランシング戦略
      - CDN＆エッジキャッシング
      - DNS管理とトラフィックルーティング
      - DDoS保護とWAF設定
      - ゼロトラストネットワークセキュリティ

    excluded:
      - アプリケーションレイヤーのセキュリティ（Doc-SC-001）
      - コンテナネットワーキングの詳細（Doc-IA-002）
      - データベースネットワーク接続の詳細（Doc-DA-001）

  target_audience:
    - ネットワークエンジニア
    - クラウドアーキテクト
    - セキュリティチーム
    - DevOpsエンジニア
    - SREチーム
```

#### 1.1.2 ネットワーク設計原則

```yaml
network_design_principles:
  1_defense_in_depth:
    description: 多層防御によるセキュリティ
    implementation:
      - ネットワーク境界でのフィルタリング
      - サブネット間のアクセス制御
      - サービス間のmTLS
      - アプリケーションレベルの認証

  2_least_privilege:
    description: 最小権限の原則
    implementation:
      - 必要最小限のポート開放
      - 明示的な許可リスト
      - デフォルト拒否ポリシー

  3_segmentation:
    description: ネットワークの論理分離
    implementation:
      - 環境別VPC分離
      - 機能別サブネット分離
      - マイクロセグメンテーション

  4_observability:
    description: 完全な可視性の確保
    implementation:
      - VPC Flow Logs
      - DNS Query Logging
      - ALBアクセスログ
      - WAFログ

  5_resilience:
    description: 障害耐性と高可用性
    implementation:
      - マルチAZ設計
      - 自動フェイルオーバー
      - ヘルスチェックと自動復旧
```

### 1.2 ネットワーク要件 & パフォーマンス目標

#### 1.2.1 パフォーマンス要件

```yaml
performance_requirements:
  latency:
    api_response:
      target_p50: 50ms
      target_p99: 200ms
      maximum: 500ms

    static_content:
      target_p50: 20ms
      target_p99: 100ms
      maximum: 200ms

    cdn_cache_hit:
      target: < 10ms

    dns_resolution:
      target: < 50ms

  throughput:
    baseline:
      requests_per_second: 10,000
      bandwidth: 10 Gbps

    peak:
      requests_per_second: 100,000
      bandwidth: 50 Gbps

    burst:
      requests_per_second: 500,000
      bandwidth: 100 Gbps

  availability:
    target_sla: 99.99%
    monthly_downtime: < 4.32 minutes
    rto: 5 minutes
    rpo: 1 minute

  scalability:
    geographic_regions:
      year_1: 1 (Japan)
      year_3: 3 (Japan, Singapore, US)
      year_5: 10 (Global)

    edge_locations:
      initial: 20+
      target: 400+ (CloudFront全エッジ)
```

#### 1.2.2 セキュリティ要件

```yaml
security_requirements:
  encryption:
    in_transit:
      protocol: TLS 1.3
      cipher_suites:
        - TLS_AES_256_GCM_SHA384
        - TLS_CHACHA20_POLY1305_SHA256
        - TLS_AES_128_GCM_SHA256
      certificate_management: AWS Certificate Manager

    at_rest:
      vpc_flow_logs: KMS暗号化
      dns_logs: KMS暗号化

  access_control:
    network_level:
      - Security Groups
      - Network ACLs
      - VPC Endpoints

    application_level:
      - WAF Rules
      - API Gateway Authorization
      - Service Mesh mTLS

  ddos_protection:
    layer_3_4:
      service: AWS Shield Standard
      upgrade: AWS Shield Advanced (本番)

    layer_7:
      service: AWS WAF
      rate_limiting: enabled
      bot_control: enabled

  compliance:
    - PCI DSS Level 1
    - GDPR
    - 個人情報保護法
    - SOC 2 Type II
```

### 1.3 主要な前提条件 & 制約

#### 1.3.1 技術的前提条件

```yaml
technical_assumptions:
  cloud_platform:
    primary: AWS (ap-northeast-1)
    dr: AWS (ap-northeast-3)
    rationale: Doc-IA-001で決定済み

  container_platform:
    orchestrator: Amazon EKS
    networking: VPC CNI
    rationale: Doc-IA-002で決定済み

  existing_infrastructure:
    current_stack:
      - Docker containers
      - Node.js backend (Hono)
      - PostgreSQL database
      - Redis cache
    migration_approach: 段階的移行

  traffic_patterns:
    peak_hours: 10:00-22:00 JST
    seasonal_peaks:
      - 年末年始
      - ゴールデンウィーク
      - お盆
    geographic_distribution:
      japan: 80%
      asia_pacific: 15%
      other: 5%
```

---

## 第2章：VPC & ネットワークアーキテクチャ

### 2.1 マルチAZ VPC設計

#### 2.1.1 VPCアーキテクチャ概要

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              TRIPTRIP VPC ARCHITECTURE                                │
│                                   10.0.0.0/16                                        │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                            INTERNET GATEWAY                                   │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                        │                                             │
│  ┌─────────────────────────────────────┴─────────────────────────────────────┐      │
│  │                         PUBLIC SUBNETS (10.0.0.0/20)                       │      │
│  │                                                                            │      │
│  │   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐          │      │
│  │   │ Public-1a       │  │ Public-1c       │  │ Public-1d       │          │      │
│  │   │ 10.0.1.0/24     │  │ 10.0.2.0/24     │  │ 10.0.3.0/24     │          │      │
│  │   │                 │  │                 │  │                 │          │      │
│  │   │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │          │      │
│  │   │ │ NAT Gateway │ │  │ │ NAT Gateway │ │  │ │ NAT Gateway │ │          │      │
│  │   │ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │          │      │
│  │   │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │          │      │
│  │   │ │     ALB     │ │  │ │     ALB     │ │  │ │     ALB     │ │          │      │
│  │   │ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │          │      │
│  │   └─────────────────┘  └─────────────────┘  └─────────────────┘          │      │
│  └──────────────────────────────────────────────────────────────────────────┘      │
│                                        │                                             │
│  ┌─────────────────────────────────────┴─────────────────────────────────────┐      │
│  │                      PRIVATE APP SUBNETS (10.0.16.0/20)                    │      │
│  │                                                                            │      │
│  │   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐          │      │
│  │   │ Private-App-1a  │  │ Private-App-1c  │  │ Private-App-1d  │          │      │
│  │   │ 10.0.16.0/22    │  │ 10.0.20.0/22    │  │ 10.0.24.0/22    │          │      │
│  │   │                 │  │                 │  │                 │          │      │
│  │   │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │          │      │
│  │   │ │ EKS Nodes   │ │  │ │ EKS Nodes   │ │  │ │ EKS Nodes   │ │          │      │
│  │   │ │ API Pods    │ │  │ │ API Pods    │ │  │ │ API Pods    │ │          │      │
│  │   │ │ Web Pods    │ │  │ │ Web Pods    │ │  │ │ Web Pods    │ │          │      │
│  │   │ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │          │      │
│  │   └─────────────────┘  └─────────────────┘  └─────────────────┘          │      │
│  └──────────────────────────────────────────────────────────────────────────┘      │
│                                        │                                             │
│  ┌─────────────────────────────────────┴─────────────────────────────────────┐      │
│  │                      PRIVATE DATA SUBNETS (10.0.32.0/20)                   │      │
│  │                                                                            │      │
│  │   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐          │      │
│  │   │ Private-Data-1a │  │ Private-Data-1c │  │ Private-Data-1d │          │      │
│  │   │ 10.0.32.0/24    │  │ 10.0.33.0/24    │  │ 10.0.34.0/24    │          │      │
│  │   │                 │  │                 │  │                 │          │      │
│  │   │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │          │      │
│  │   │ │     RDS     │ │  │ │     RDS     │ │  │ │     RDS     │ │          │      │
│  │   │ │ ElastiCache │ │  │ │ ElastiCache │ │  │ │ ElastiCache │ │          │      │
│  │   │ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │          │      │
│  │   └─────────────────┘  └─────────────────┘  └─────────────────┘          │      │
│  └──────────────────────────────────────────────────────────────────────────┘      │
│                                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────────┐      │
│  │                           VPC ENDPOINTS                                   │      │
│  │  S3 | DynamoDB | ECR | Secrets Manager | CloudWatch | STS | EC2          │      │
│  └──────────────────────────────────────────────────────────────────────────┘      │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

#### 2.1.2 VPC設定詳細

```yaml
vpc_configuration:
  production_vpc:
    name: triptrip-production-vpc
    cidr_block: 10.0.0.0/16
    enable_dns_hostnames: true
    enable_dns_support: true
    instance_tenancy: default

    # 追加CIDRブロック（将来の拡張用）
    secondary_cidr_blocks:
      - 10.1.0.0/16  # 拡張用
      - 10.2.0.0/16  # 拡張用

    tags:
      Environment: production
      Project: triptrip
      ManagedBy: terraform

  availability_zones:
    primary:
      - ap-northeast-1a
      - ap-northeast-1c
      - ap-northeast-1d
    rationale: 3AZ構成で高可用性確保

  ip_addressing:
    strategy: CIDR-based with prefix delegation
    pod_networking: VPC CNI with prefix delegation
    estimated_capacity:
      subnets: 256 /24 subnets
      hosts_per_subnet: 251
      total_ips: 65,536
```

### 2.2 サブネット戦略（パブリック/プライベート/データ層）

#### 2.2.1 サブネット設計

```yaml
subnet_design:
  public_subnets:
    purpose: インターネット接続、ロードバランサー、NAT Gateway
    cidr_range: 10.0.0.0/20
    subnets:
      public_1a:
        cidr: 10.0.1.0/24
        az: ap-northeast-1a
        available_ips: 251
        resources:
          - NAT Gateway
          - Application Load Balancer
          - Bastion Host (必要時)

      public_1c:
        cidr: 10.0.2.0/24
        az: ap-northeast-1c
        available_ips: 251
        resources:
          - NAT Gateway
          - Application Load Balancer

      public_1d:
        cidr: 10.0.3.0/24
        az: ap-northeast-1d
        available_ips: 251
        resources:
          - NAT Gateway
          - Application Load Balancer

    route_table:
      routes:
        - destination: 0.0.0.0/0
          target: internet-gateway
        - destination: 10.0.0.0/16
          target: local

  private_app_subnets:
    purpose: アプリケーション層（EKS、EC2）
    cidr_range: 10.0.16.0/20
    subnets:
      private_app_1a:
        cidr: 10.0.16.0/22
        az: ap-northeast-1a
        available_ips: 1,019
        resources:
          - EKS Node Group
          - Application Pods

      private_app_1c:
        cidr: 10.0.20.0/22
        az: ap-northeast-1c
        available_ips: 1,019
        resources:
          - EKS Node Group
          - Application Pods

      private_app_1d:
        cidr: 10.0.24.0/22
        az: ap-northeast-1d
        available_ips: 1,019
        resources:
          - EKS Node Group
          - Application Pods

    route_table:
      routes:
        - destination: 0.0.0.0/0
          target: nat-gateway (同一AZ)
        - destination: 10.0.0.0/16
          target: local
        - destination: s3-prefix-list
          target: s3-gateway-endpoint
        - destination: dynamodb-prefix-list
          target: dynamodb-gateway-endpoint

  private_data_subnets:
    purpose: データ層（RDS、ElastiCache）
    cidr_range: 10.0.32.0/20
    subnets:
      private_data_1a:
        cidr: 10.0.32.0/24
        az: ap-northeast-1a
        available_ips: 251
        resources:
          - RDS Primary
          - ElastiCache Node

      private_data_1c:
        cidr: 10.0.33.0/24
        az: ap-northeast-1c
        available_ips: 251
        resources:
          - RDS Standby
          - ElastiCache Node

      private_data_1d:
        cidr: 10.0.34.0/24
        az: ap-northeast-1d
        available_ips: 251
        resources:
          - RDS Read Replica
          - ElastiCache Node

    route_table:
      routes:
        - destination: 10.0.0.0/16
          target: local
      # インターネットアクセスなし（セキュリティ強化）

  reserved_subnets:
    purpose: 将来の拡張用
    cidr_range: 10.0.48.0/20
    status: reserved
    planned_use:
      - Lambda VPC接続
      - 追加サービス
      - DR拡張
```

#### 2.2.2 Terraform実装

```hcl
# VPC Configuration
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  instance_tenancy     = "default"

  tags = {
    Name        = "triptrip-production-vpc"
    Environment = "production"
    Project     = "triptrip"
  }
}

# Public Subnets
resource "aws_subnet" "public" {
  for_each = {
    "1a" = { cidr = "10.0.1.0/24", az = "ap-northeast-1a" }
    "1c" = { cidr = "10.0.2.0/24", az = "ap-northeast-1c" }
    "1d" = { cidr = "10.0.3.0/24", az = "ap-northeast-1d" }
  }

  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value.cidr
  availability_zone       = each.value.az
  map_public_ip_on_launch = true

  tags = {
    Name                                        = "triptrip-public-${each.key}"
    "kubernetes.io/role/elb"                    = "1"
    "kubernetes.io/cluster/triptrip-production" = "shared"
  }
}

# Private App Subnets
resource "aws_subnet" "private_app" {
  for_each = {
    "1a" = { cidr = "10.0.16.0/22", az = "ap-northeast-1a" }
    "1c" = { cidr = "10.0.20.0/22", az = "ap-northeast-1c" }
    "1d" = { cidr = "10.0.24.0/22", az = "ap-northeast-1d" }
  }

  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value.cidr
  availability_zone       = each.value.az
  map_public_ip_on_launch = false

  tags = {
    Name                                        = "triptrip-private-app-${each.key}"
    "kubernetes.io/role/internal-elb"           = "1"
    "kubernetes.io/cluster/triptrip-production" = "shared"
  }
}

# Private Data Subnets
resource "aws_subnet" "private_data" {
  for_each = {
    "1a" = { cidr = "10.0.32.0/24", az = "ap-northeast-1a" }
    "1c" = { cidr = "10.0.33.0/24", az = "ap-northeast-1c" }
    "1d" = { cidr = "10.0.34.0/24", az = "ap-northeast-1d" }
  }

  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value.cidr
  availability_zone       = each.value.az
  map_public_ip_on_launch = false

  tags = {
    Name = "triptrip-private-data-${each.key}"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "triptrip-igw"
  }
}

# NAT Gateways (1 per AZ for high availability)
resource "aws_eip" "nat" {
  for_each = toset(["1a", "1c", "1d"])
  domain   = "vpc"

  tags = {
    Name = "triptrip-nat-eip-${each.key}"
  }
}

resource "aws_nat_gateway" "main" {
  for_each = {
    "1a" = { subnet_key = "1a" }
    "1c" = { subnet_key = "1c" }
    "1d" = { subnet_key = "1d" }
  }

  allocation_id = aws_eip.nat[each.key].id
  subnet_id     = aws_subnet.public[each.value.subnet_key].id

  tags = {
    Name = "triptrip-nat-${each.key}"
  }

  depends_on = [aws_internet_gateway.main]
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "triptrip-public-rt"
  }
}

resource "aws_route_table" "private_app" {
  for_each = toset(["1a", "1c", "1d"])
  vpc_id   = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[each.key].id
  }

  tags = {
    Name = "triptrip-private-app-rt-${each.key}"
  }
}

resource "aws_route_table" "private_data" {
  vpc_id = aws_vpc.main.id
  # No internet route for data subnets

  tags = {
    Name = "triptrip-private-data-rt"
  }
}
```

### 2.3 セキュリティグループ & NACL

#### 2.3.1 セキュリティグループ設計

```yaml
security_groups:
  alb_external_sg:
    name: triptrip-alb-external-sg
    description: External ALB Security Group
    ingress_rules:
      - description: HTTPS from Internet
        protocol: tcp
        from_port: 443
        to_port: 443
        cidr_blocks:
          - 0.0.0.0/0
        ipv6_cidr_blocks:
          - ::/0

      - description: HTTP for redirect
        protocol: tcp
        from_port: 80
        to_port: 80
        cidr_blocks:
          - 0.0.0.0/0

    egress_rules:
      - description: All outbound
        protocol: -1
        from_port: 0
        to_port: 0
        cidr_blocks:
          - 0.0.0.0/0

  alb_internal_sg:
    name: triptrip-alb-internal-sg
    description: Internal ALB Security Group
    ingress_rules:
      - description: HTTPS from VPC
        protocol: tcp
        from_port: 443
        to_port: 443
        cidr_blocks:
          - 10.0.0.0/16

      - description: HTTP from VPC
        protocol: tcp
        from_port: 80
        to_port: 80
        cidr_blocks:
          - 10.0.0.0/16

    egress_rules:
      - description: All outbound
        protocol: -1
        from_port: 0
        to_port: 0
        cidr_blocks:
          - 0.0.0.0/0

  eks_cluster_sg:
    name: triptrip-eks-cluster-sg
    description: EKS Cluster Security Group
    ingress_rules:
      - description: API Server from Nodes
        protocol: tcp
        from_port: 443
        to_port: 443
        source_security_group: eks_nodes_sg

      - description: API Server from VPN
        protocol: tcp
        from_port: 443
        to_port: 443
        cidr_blocks:
          - 10.0.0.0/16

    egress_rules:
      - description: All outbound
        protocol: -1
        from_port: 0
        to_port: 0
        cidr_blocks:
          - 0.0.0.0/0

  eks_nodes_sg:
    name: triptrip-eks-nodes-sg
    description: EKS Nodes Security Group
    ingress_rules:
      - description: Control Plane to Nodes
        protocol: tcp
        from_port: 443
        to_port: 443
        source_security_group: eks_cluster_sg

      - description: Control Plane to kubelet
        protocol: tcp
        from_port: 10250
        to_port: 10250
        source_security_group: eks_cluster_sg

      - description: Node to Node
        protocol: -1
        from_port: 0
        to_port: 0
        self: true

      - description: ALB to NodePort
        protocol: tcp
        from_port: 30000
        to_port: 32767
        source_security_group: alb_external_sg

      - description: Internal ALB to NodePort
        protocol: tcp
        from_port: 30000
        to_port: 32767
        source_security_group: alb_internal_sg

      - description: Application port from ALB
        protocol: tcp
        from_port: 3000
        to_port: 3000
        source_security_group: alb_external_sg

      - description: Prometheus metrics
        protocol: tcp
        from_port: 9090
        to_port: 9090
        cidr_blocks:
          - 10.0.0.0/16

    egress_rules:
      - description: All outbound
        protocol: -1
        from_port: 0
        to_port: 0
        cidr_blocks:
          - 0.0.0.0/0

  rds_sg:
    name: triptrip-rds-sg
    description: RDS Security Group
    ingress_rules:
      - description: PostgreSQL from EKS Nodes
        protocol: tcp
        from_port: 5432
        to_port: 5432
        source_security_group: eks_nodes_sg

      - description: PostgreSQL from Lambda
        protocol: tcp
        from_port: 5432
        to_port: 5432
        source_security_group: lambda_sg

    egress_rules: []  # No outbound required

  elasticache_sg:
    name: triptrip-elasticache-sg
    description: ElastiCache Security Group
    ingress_rules:
      - description: Redis from EKS Nodes
        protocol: tcp
        from_port: 6379
        to_port: 6379
        source_security_group: eks_nodes_sg

      - description: Redis Cluster Bus
        protocol: tcp
        from_port: 16379
        to_port: 16379
        source_security_group: eks_nodes_sg

    egress_rules: []  # No outbound required

  lambda_sg:
    name: triptrip-lambda-sg
    description: Lambda Functions Security Group
    ingress_rules: []  # No inbound required for Lambda

    egress_rules:
      - description: HTTPS outbound
        protocol: tcp
        from_port: 443
        to_port: 443
        cidr_blocks:
          - 0.0.0.0/0

      - description: PostgreSQL to RDS
        protocol: tcp
        from_port: 5432
        to_port: 5432
        destination_security_group: rds_sg

      - description: Redis to ElastiCache
        protocol: tcp
        from_port: 6379
        to_port: 6379
        destination_security_group: elasticache_sg

  vpc_endpoints_sg:
    name: triptrip-vpc-endpoints-sg
    description: VPC Endpoints Security Group
    ingress_rules:
      - description: HTTPS from VPC
        protocol: tcp
        from_port: 443
        to_port: 443
        cidr_blocks:
          - 10.0.0.0/16

    egress_rules: []
```

#### 2.3.2 Network ACL設計

```yaml
network_acls:
  public_nacl:
    name: triptrip-public-nacl
    subnets:
      - public-1a
      - public-1c
      - public-1d

    ingress_rules:
      - rule_number: 100
        protocol: tcp
        port_range: 443
        cidr_block: 0.0.0.0/0
        action: allow

      - rule_number: 110
        protocol: tcp
        port_range: 80
        cidr_block: 0.0.0.0/0
        action: allow

      - rule_number: 120
        protocol: tcp
        port_range: 1024-65535
        cidr_block: 0.0.0.0/0
        action: allow

      - rule_number: 130
        protocol: tcp
        port_range: 22
        cidr_block: 10.0.0.0/16  # VPN/Bastion only
        action: allow

      - rule_number: 32766
        protocol: -1
        cidr_block: 0.0.0.0/0
        action: deny

    egress_rules:
      - rule_number: 100
        protocol: -1
        cidr_block: 0.0.0.0/0
        action: allow

  private_app_nacl:
    name: triptrip-private-app-nacl
    subnets:
      - private-app-1a
      - private-app-1c
      - private-app-1d

    ingress_rules:
      - rule_number: 100
        protocol: tcp
        port_range: 443
        cidr_block: 10.0.0.0/16
        action: allow

      - rule_number: 110
        protocol: tcp
        port_range: 3000
        cidr_block: 10.0.0.0/20  # Public subnets only
        action: allow

      - rule_number: 120
        protocol: tcp
        port_range: 30000-32767
        cidr_block: 10.0.0.0/20
        action: allow

      - rule_number: 130
        protocol: tcp
        port_range: 1024-65535
        cidr_block: 0.0.0.0/0
        action: allow

      - rule_number: 140
        protocol: tcp
        port_range: 10250
        cidr_block: 10.0.0.0/16
        action: allow

      - rule_number: 32766
        protocol: -1
        cidr_block: 0.0.0.0/0
        action: deny

    egress_rules:
      - rule_number: 100
        protocol: -1
        cidr_block: 0.0.0.0/0
        action: allow

  private_data_nacl:
    name: triptrip-private-data-nacl
    subnets:
      - private-data-1a
      - private-data-1c
      - private-data-1d

    ingress_rules:
      - rule_number: 100
        protocol: tcp
        port_range: 5432
        cidr_block: 10.0.16.0/20  # App subnets only
        action: allow

      - rule_number: 110
        protocol: tcp
        port_range: 6379
        cidr_block: 10.0.16.0/20
        action: allow

      - rule_number: 120
        protocol: tcp
        port_range: 16379
        cidr_block: 10.0.16.0/20
        action: allow

      - rule_number: 130
        protocol: tcp
        port_range: 1024-65535
        cidr_block: 10.0.16.0/20
        action: allow

      - rule_number: 32766
        protocol: -1
        cidr_block: 0.0.0.0/0
        action: deny

    egress_rules:
      - rule_number: 100
        protocol: tcp
        port_range: 1024-65535
        cidr_block: 10.0.16.0/20
        action: allow

      - rule_number: 32766
        protocol: -1
        cidr_block: 0.0.0.0/0
        action: deny
```

---

## 第3章：ロードバランシング & トラフィック管理

### 3.1 ALB/NLB設計

#### 3.1.1 Application Load Balancer（外部向け）

```yaml
external_alb:
  name: triptrip-external-alb
  scheme: internet-facing
  type: application
  ip_address_type: dualstack  # IPv4 & IPv6

  subnets:
    - public-1a
    - public-1c
    - public-1d

  security_groups:
    - alb-external-sg

  listeners:
    https:
      port: 443
      protocol: HTTPS
      ssl_policy: ELBSecurityPolicy-TLS13-1-2-2021-06
      default_certificate: arn:aws:acm:ap-northeast-1:xxx:certificate/xxx

      default_action:
        type: forward
        target_group: triptrip-api-tg

      rules:
        # API routing
        - priority: 10
          conditions:
            - field: path-pattern
              values:
                - /api/*
          actions:
            - type: forward
              target_group: triptrip-api-tg

        # Health check endpoint
        - priority: 20
          conditions:
            - field: path-pattern
              values:
                - /health
                - /ready
          actions:
            - type: forward
              target_group: triptrip-api-tg

        # Admin routes (with authentication)
        - priority: 30
          conditions:
            - field: path-pattern
              values:
                - /admin/*
          actions:
            - type: authenticate-oidc
              oidc_config:
                issuer: https://auth.triptrip.com
                authorization_endpoint: /authorize
                token_endpoint: /token
                user_info_endpoint: /userinfo
                client_id: admin-client
                client_secret_arn: arn:aws:secretsmanager:xxx
            - type: forward
              target_group: triptrip-admin-tg

        # Static assets (redirected to CloudFront)
        - priority: 100
          conditions:
            - field: path-pattern
              values:
                - /static/*
                - /images/*
          actions:
            - type: redirect
              redirect:
                host: cdn.triptrip.com
                status_code: HTTP_302

    http:
      port: 80
      protocol: HTTP
      default_action:
        type: redirect
        redirect:
          protocol: HTTPS
          port: "443"
          status_code: HTTP_301

  target_groups:
    api:
      name: triptrip-api-tg
      port: 3000
      protocol: HTTP
      protocol_version: HTTP1
      target_type: ip
      vpc_id: vpc-xxx

      health_check:
        enabled: true
        protocol: HTTP
        path: /health
        port: traffic-port
        interval: 15
        timeout: 5
        healthy_threshold: 2
        unhealthy_threshold: 3
        matcher:
          http_code: 200

      stickiness:
        enabled: false

      deregistration_delay: 30

      slow_start: 30

    admin:
      name: triptrip-admin-tg
      port: 3001
      protocol: HTTP
      target_type: ip

  attributes:
    idle_timeout: 60
    routing.http.drop_invalid_header_fields.enabled: true
    routing.http2.enabled: true
    access_logs:
      enabled: true
      bucket: triptrip-alb-logs
      prefix: external-alb

  waf:
    enabled: true
    web_acl_arn: arn:aws:wafv2:ap-northeast-1:xxx:webacl/triptrip-waf/xxx
```

#### 3.1.2 Application Load Balancer（内部向け）

```yaml
internal_alb:
  name: triptrip-internal-alb
  scheme: internal
  type: application
  ip_address_type: ipv4

  subnets:
    - private-app-1a
    - private-app-1c
    - private-app-1d

  security_groups:
    - alb-internal-sg

  listeners:
    https:
      port: 443
      protocol: HTTPS
      ssl_policy: ELBSecurityPolicy-TLS13-1-2-2021-06
      default_certificate: arn:aws:acm:ap-northeast-1:xxx:certificate/internal-xxx

      default_action:
        type: forward
        target_group: triptrip-internal-api-tg

      rules:
        - priority: 10
          conditions:
            - field: path-pattern
              values:
                - /internal/api/*
          actions:
            - type: forward
              target_group: triptrip-internal-api-tg

        - priority: 20
          conditions:
            - field: path-pattern
              values:
                - /ml/*
          actions:
            - type: forward
              target_group: triptrip-ml-tg

  target_groups:
    internal_api:
      name: triptrip-internal-api-tg
      port: 3000
      protocol: HTTP
      target_type: ip

      health_check:
        path: /health
        interval: 10
        timeout: 5

    ml_inference:
      name: triptrip-ml-tg
      port: 8080
      protocol: HTTP
      target_type: ip

      health_check:
        path: /health
        interval: 30
        timeout: 10
```

#### 3.1.3 Network Load Balancer（高パフォーマンス用）

```yaml
network_load_balancer:
  name: triptrip-nlb
  scheme: internal
  type: network
  ip_address_type: ipv4

  subnets:
    - private-app-1a
    - private-app-1c
    - private-app-1d

  listeners:
    grpc:
      port: 50051
      protocol: TCP
      target_group: triptrip-grpc-tg

    tcp:
      port: 6379
      protocol: TCP
      target_group: triptrip-redis-proxy-tg

  target_groups:
    grpc:
      name: triptrip-grpc-tg
      port: 50051
      protocol: TCP
      target_type: ip

      health_check:
        protocol: TCP
        port: traffic-port
        interval: 10
        healthy_threshold: 2
        unhealthy_threshold: 2

      preserve_client_ip: true

    redis_proxy:
      name: triptrip-redis-proxy-tg
      port: 6379
      protocol: TCP
      target_type: ip

      preserve_client_ip: false

  attributes:
    cross_zone_load_balancing: true
    deletion_protection: true
```

### 3.2 地理的トラフィック分散

#### 3.2.1 AWS Global Accelerator設計

```yaml
global_accelerator:
  name: triptrip-global-accelerator
  ip_address_type: IPV4
  enabled: true

  # Static Anycast IPs
  static_ip_addresses:
    - 75.2.xxx.xxx
    - 99.83.xxx.xxx

  listeners:
    https:
      port_ranges:
        - from_port: 443
          to_port: 443
      protocol: TCP
      client_affinity: SOURCE_IP

    http:
      port_ranges:
        - from_port: 80
          to_port: 80
      protocol: TCP
      client_affinity: NONE

  endpoint_groups:
    japan:
      region: ap-northeast-1
      traffic_dial_percentage: 100
      health_check:
        port: 443
        protocol: TCP
        interval: 10
        threshold: 3
      endpoints:
        - endpoint_id: arn:aws:elasticloadbalancing:ap-northeast-1:xxx:loadbalancer/app/triptrip-external-alb/xxx
          weight: 100
          client_ip_preservation: true

    singapore:
      region: ap-southeast-1
      traffic_dial_percentage: 100
      health_check:
        port: 443
        protocol: TCP
      endpoints:
        - endpoint_id: arn:aws:elasticloadbalancing:ap-southeast-1:xxx:loadbalancer/app/triptrip-sg-alb/xxx
          weight: 100
      # Phase 2で有効化

    us_west:
      region: us-west-2
      traffic_dial_percentage: 100
      endpoints:
        - endpoint_id: arn:aws:elasticloadbalancing:us-west-2:xxx:loadbalancer/app/triptrip-us-alb/xxx
          weight: 100
      # Phase 3で有効化

  routing:
    strategy: PERFORMANCE  # 最低レイテンシへルーティング
    failover:
      enabled: true
      failover_threshold: 3
```

### 3.3 ヘルスチェック & フェイルオーバー

#### 3.3.1 ヘルスチェック設計

```yaml
health_check_design:
  alb_health_checks:
    api_services:
      path: /health
      protocol: HTTP
      port: traffic-port
      interval: 15
      timeout: 5
      healthy_threshold: 2
      unhealthy_threshold: 3
      matcher:
        http_code: 200

    web_services:
      path: /health
      interval: 30
      timeout: 10
      healthy_threshold: 2
      unhealthy_threshold: 2

    ml_services:
      path: /health
      interval: 60
      timeout: 30
      healthy_threshold: 2
      unhealthy_threshold: 3

  application_health_endpoints:
    liveness:
      path: /health/live
      description: コンテナが動作しているか
      checks:
        - process_running
        - memory_available

    readiness:
      path: /health/ready
      description: トラフィックを受ける準備ができているか
      checks:
        - database_connection
        - cache_connection
        - dependencies_available

    startup:
      path: /health/startup
      description: 起動が完了したか
      checks:
        - initialization_complete
        - warmup_complete

  health_check_implementation:
    node_js_example: |
      // Health check endpoints
      app.get('/health/live', (req, res) => {
        res.status(200).json({ status: 'ok', timestamp: Date.now() });
      });

      app.get('/health/ready', async (req, res) => {
        try {
          // Check database
          await prisma.$queryRaw`SELECT 1`;

          // Check Redis
          await redis.ping();

          res.status(200).json({
            status: 'ready',
            checks: {
              database: 'ok',
              cache: 'ok'
            }
          });
        } catch (error) {
          res.status(503).json({
            status: 'not_ready',
            error: error.message
          });
        }
      });

      app.get('/health/startup', (req, res) => {
        if (appInitialized) {
          res.status(200).json({ status: 'started' });
        } else {
          res.status(503).json({ status: 'starting' });
        }
      });

  failover_configuration:
    target_group_failover:
      type: automatic
      trigger: unhealthy_threshold_exceeded
      action: remove_from_rotation

    cross_az_failover:
      enabled: true
      min_healthy_targets: 1
      failover_target: same_region_different_az

    cross_region_failover:
      enabled: true
      trigger: all_targets_unhealthy
      target_region: ap-northeast-3
      failover_dns: Route 53 health check
```

---

## 第4章：CDN & エッジアーキテクチャ

### 4.1 CloudFront/Fastly構成

#### 4.1.1 CloudFront Distribution設計

```yaml
cloudfront_distribution:
  main_distribution:
    id: E1XXXXXXXXXX
    comment: TripTrip Main Distribution
    enabled: true
    is_ipv6_enabled: true
    http_version: http2and3
    price_class: PriceClass_200  # 北米、欧州、アジア、中東、アフリカ

    aliases:
      - www.triptrip.com
      - triptrip.com
      - cdn.triptrip.com

    viewer_certificate:
      acm_certificate_arn: arn:aws:acm:us-east-1:xxx:certificate/xxx
      ssl_support_method: sni-only
      minimum_protocol_version: TLSv1.2_2021

    origins:
      alb_origin:
        domain_name: triptrip-external-alb-xxx.ap-northeast-1.elb.amazonaws.com
        origin_id: alb-origin
        origin_path: ""

        custom_origin_config:
          http_port: 80
          https_port: 443
          origin_protocol_policy: https-only
          origin_ssl_protocols:
            - TLSv1.2
          origin_read_timeout: 60
          origin_keepalive_timeout: 60

        custom_headers:
          - header_name: X-Origin-Verify
            header_value: ${SECRET_ORIGIN_HEADER}

        origin_shield:
          enabled: true
          origin_shield_region: ap-northeast-1

      s3_static_origin:
        domain_name: triptrip-static-assets.s3.ap-northeast-1.amazonaws.com
        origin_id: s3-static-origin
        origin_access_control:
          signing_behavior: always
          signing_protocol: sigv4
          origin_access_control_origin_type: s3

      s3_media_origin:
        domain_name: triptrip-media.s3.ap-northeast-1.amazonaws.com
        origin_id: s3-media-origin
        origin_access_control:
          signing_behavior: always
          signing_protocol: sigv4

    origin_groups:
      api_origin_group:
        origins:
          - origin_id: alb-origin
            priority: 1
          - origin_id: alb-dr-origin
            priority: 2
        failover_criteria:
          status_codes:
            - 500
            - 502
            - 503
            - 504

    default_cache_behavior:
      target_origin_id: alb-origin
      viewer_protocol_policy: redirect-to-https
      allowed_methods:
        - GET
        - HEAD
        - OPTIONS
        - PUT
        - POST
        - PATCH
        - DELETE
      cached_methods:
        - GET
        - HEAD
      compress: true

      cache_policy_id: 4135ea2d-6df8-44a3-9df3-4b5a84be39ad  # CachingDisabled

      origin_request_policy_id: 216adef6-5c7f-47e4-b989-5492eafa07d3  # AllViewer

      response_headers_policy_id: custom-security-headers

      function_associations:
        - event_type: viewer-request
          function_arn: arn:aws:cloudfront::xxx:function/url-rewrite
        - event_type: viewer-response
          function_arn: arn:aws:cloudfront::xxx:function/security-headers

    cache_behaviors:
      # Static assets
      - path_pattern: /static/*
        target_origin_id: s3-static-origin
        viewer_protocol_policy: redirect-to-https
        allowed_methods:
          - GET
          - HEAD
        cached_methods:
          - GET
          - HEAD
        compress: true
        cache_policy_id: 658327ea-f89d-4fab-a63d-7e88639e58f6  # CachingOptimized
        ttl:
          default: 86400      # 1 day
          max: 31536000       # 1 year
          min: 0

      # Images with automatic WebP conversion
      - path_pattern: /images/*
        target_origin_id: s3-media-origin
        viewer_protocol_policy: redirect-to-https
        compress: true
        cache_policy_id: 658327ea-f89d-4fab-a63d-7e88639e58f6
        response_headers_policy_id: cors-with-preflight

        lambda_function_associations:
          - event_type: origin-request
            lambda_arn: arn:aws:lambda:us-east-1:xxx:function:image-optimizer:1

      # API endpoints (no caching by default)
      - path_pattern: /api/*
        target_origin_id: alb-origin
        viewer_protocol_policy: https-only
        allowed_methods:
          - GET
          - HEAD
          - OPTIONS
          - PUT
          - POST
          - PATCH
          - DELETE
        cache_policy_id: 4135ea2d-6df8-44a3-9df3-4b5a84be39ad  # CachingDisabled
        origin_request_policy_id: b689b0a8-53d0-40ab-baf2-68738e2966ac  # AllViewerExceptHostHeader

      # Public API with short caching
      - path_pattern: /api/v1/public/*
        target_origin_id: alb-origin
        viewer_protocol_policy: https-only
        allowed_methods:
          - GET
          - HEAD
          - OPTIONS
        cache_policy_id: custom-api-cache-policy  # 60 seconds TTL
        origin_request_policy_id: b689b0a8-53d0-40ab-baf2-68738e2966ac

      # Fonts
      - path_pattern: /fonts/*
        target_origin_id: s3-static-origin
        viewer_protocol_policy: redirect-to-https
        compress: true
        cache_policy_id: 658327ea-f89d-4fab-a63d-7e88639e58f6
        response_headers_policy_id: cors-s3-origin

    restrictions:
      geo_restriction:
        restriction_type: none  # Allow all countries
        # restriction_type: whitelist  # For Phase 1
        # locations:
        #   - JP

    logging:
      enabled: true
      bucket: triptrip-cloudfront-logs.s3.amazonaws.com
      prefix: cdn/
      include_cookies: false

    web_acl_id: arn:aws:wafv2:us-east-1:xxx:global/webacl/triptrip-cdn-waf/xxx

    tags:
      Environment: production
      Project: triptrip
```

### 4.2 キャッシュ戦略 & TTL管理

#### 4.2.1 キャッシュポリシー設計

```yaml
cache_policies:
  # Static Assets - Long TTL
  static_assets_policy:
    name: TripTrip-StaticAssets
    description: Long-term caching for static assets
    min_ttl: 86400       # 1 day
    default_ttl: 604800  # 7 days
    max_ttl: 31536000    # 1 year
    parameters_in_cache_key:
      accept_encoding_gzip: true
      accept_encoding_brotli: true
      cookies: none
      headers: none
      query_strings: none

  # Images - Medium TTL with format support
  images_policy:
    name: TripTrip-Images
    description: Image caching with format negotiation
    min_ttl: 3600        # 1 hour
    default_ttl: 86400   # 1 day
    max_ttl: 2592000     # 30 days
    parameters_in_cache_key:
      accept_encoding_gzip: true
      accept_encoding_brotli: true
      cookies: none
      headers:
        - Accept  # For WebP negotiation
      query_strings:
        behavior: whitelist
        items:
          - w      # Width
          - h      # Height
          - q      # Quality
          - f      # Format

  # API Public Data - Short TTL
  api_public_policy:
    name: TripTrip-APIPublic
    description: Short caching for public API data
    min_ttl: 0
    default_ttl: 60      # 1 minute
    max_ttl: 300         # 5 minutes
    parameters_in_cache_key:
      accept_encoding_gzip: true
      accept_encoding_brotli: true
      cookies: none
      headers:
        - Accept-Language
      query_strings:
        behavior: all

  # API Private - No Cache
  api_private_policy:
    name: TripTrip-APIPrivate
    description: No caching for authenticated API
    min_ttl: 0
    default_ttl: 0
    max_ttl: 0
    parameters_in_cache_key:
      accept_encoding_gzip: true
      cookies:
        behavior: none  # Don't include in cache key
      headers:
        - Authorization  # Vary by auth header
      query_strings:
        behavior: all

cache_invalidation_strategy:
  automatic:
    versioned_urls:
      enabled: true
      pattern: /static/v{version}/*
      description: URL内のバージョンでキャッシュバスト

    content_hash:
      enabled: true
      pattern: /static/assets/*.[hash].js
      description: ファイル名にハッシュを含む

  manual:
    api_triggered:
      endpoint: POST /admin/cache/invalidate
      parameters:
        paths:
          - /images/product/*
          - /api/v1/public/categories
      cost: $0.005 per 1,000 paths

    scheduled:
      frequency: daily
      time: "04:00 JST"
      paths:
        - /api/v1/public/featured/*

  cache_tags:
    enabled: true
    implementation: CloudFront Tags (Lambda@Edge)
    examples:
      - tag: product-123
        paths: ["/images/products/123/*", "/api/v1/products/123"]
      - tag: category-hotels
        paths: ["/api/v1/public/hotels/*"]

ttl_recommendations:
  static_html: 300-3600 seconds
  css_js_versioned: 31536000 seconds (1 year)
  css_js_unversioned: 86400 seconds (1 day)
  images_product: 86400 seconds
  images_user: 3600 seconds
  api_public_list: 60-300 seconds
  api_public_detail: 30-60 seconds
  api_private: 0 seconds
  fonts: 31536000 seconds
  pdf_documents: 86400 seconds
```

### 4.3 エッジコンピューティング活用

#### 4.3.1 CloudFront Functions

```yaml
cloudfront_functions:
  url_rewrite:
    name: triptrip-url-rewrite
    runtime: cloudfront-js-1.0
    comment: URL rewriting and normalization
    code: |
      function handler(event) {
        var request = event.request;
        var uri = request.uri;

        // Trailing slash normalization
        if (uri.endsWith('/') && uri !== '/') {
          return {
            statusCode: 301,
            statusDescription: 'Moved Permanently',
            headers: {
              'location': { value: uri.slice(0, -1) }
            }
          };
        }

        // Lowercase URLs
        if (uri !== uri.toLowerCase()) {
          return {
            statusCode: 301,
            statusDescription: 'Moved Permanently',
            headers: {
              'location': { value: uri.toLowerCase() }
            }
          };
        }

        // SPA routing - serve index.html for non-file paths
        if (!uri.includes('.') && !uri.startsWith('/api/')) {
          request.uri = '/index.html';
        }

        return request;
      }

  security_headers:
    name: triptrip-security-headers
    runtime: cloudfront-js-1.0
    comment: Add security headers to responses
    code: |
      function handler(event) {
        var response = event.response;
        var headers = response.headers;

        // Strict Transport Security
        headers['strict-transport-security'] = {
          value: 'max-age=31536000; includeSubDomains; preload'
        };

        // Content Security Policy
        headers['content-security-policy'] = {
          value: "default-src 'self'; script-src 'self' 'unsafe-inline' https://cdn.triptrip.com; style-src 'self' 'unsafe-inline'; img-src 'self' https: data:; font-src 'self' https://fonts.gstatic.com; connect-src 'self' https://api.triptrip.com wss://ws.triptrip.com; frame-ancestors 'none';"
        };

        // Prevent MIME type sniffing
        headers['x-content-type-options'] = { value: 'nosniff' };

        // Prevent framing
        headers['x-frame-options'] = { value: 'DENY' };

        // XSS Protection
        headers['x-xss-protection'] = { value: '1; mode=block' };

        // Referrer Policy
        headers['referrer-policy'] = { value: 'strict-origin-when-cross-origin' };

        // Permissions Policy
        headers['permissions-policy'] = {
          value: 'accelerometer=(), camera=(), geolocation=(self), gyroscope=(), magnetometer=(), microphone=(), payment=(self), usb=()'
        };

        return response;
      }

  a_b_testing:
    name: triptrip-ab-testing
    runtime: cloudfront-js-1.0
    comment: A/B testing header injection
    code: |
      function handler(event) {
        var request = event.request;
        var cookies = request.cookies;

        // Check for existing experiment cookie
        var experimentCookie = cookies['triptrip-experiment'];
        var variant;

        if (experimentCookie) {
          variant = experimentCookie.value;
        } else {
          // Assign random variant
          variant = Math.random() < 0.5 ? 'A' : 'B';
          // Set cookie in response
          request.headers['x-experiment-variant'] = { value: variant };
        }

        // Add header for backend
        request.headers['x-triptrip-variant'] = { value: variant };

        return request;
      }
```

#### 4.3.2 Lambda@Edge

```yaml
lambda_at_edge:
  image_optimizer:
    name: triptrip-image-optimizer
    runtime: nodejs18.x
    memory: 512
    timeout: 30
    event_type: origin-request
    description: Image optimization and format conversion

    code_overview: |
      // Handles image optimization:
      // 1. Detect client Accept header for WebP support
      // 2. Generate optimized image URL with resize parameters
      // 3. Return from S3 or trigger resize Lambda

      exports.handler = async (event) => {
        const request = event.Records[0].cf.request;
        const headers = request.headers;

        // Check WebP support
        const acceptHeader = headers['accept'] ? headers['accept'][0].value : '';
        const supportsWebP = acceptHeader.includes('image/webp');

        // Parse query parameters
        const params = new URLSearchParams(request.querystring);
        const width = params.get('w');
        const height = params.get('h');
        const quality = params.get('q') || '85';

        // Modify URI for optimized image
        const originalUri = request.uri;
        const ext = supportsWebP ? 'webp' : originalUri.split('.').pop();

        if (width || height) {
          const dimensions = `${width || 'auto'}x${height || 'auto'}`;
          request.uri = originalUri.replace(
            /\.([^.]+)$/,
            `_${dimensions}_q${quality}.${ext}`
          );
        } else if (supportsWebP && !originalUri.endsWith('.webp')) {
          request.uri = originalUri.replace(/\.([^.]+)$/, '.webp');
        }

        return request;
      };

  geo_redirect:
    name: triptrip-geo-redirect
    runtime: nodejs18.x
    memory: 128
    timeout: 5
    event_type: viewer-request
    description: Geographic-based redirects

    code_overview: |
      exports.handler = async (event) => {
        const request = event.Records[0].cf.request;
        const headers = request.headers;

        // Get country from CloudFront header
        const country = headers['cloudfront-viewer-country'] ?
          headers['cloudfront-viewer-country'][0].value : 'JP';

        // Add country header for backend
        request.headers['x-viewer-country'] = [{ value: country }];

        // Regional redirects for specific paths
        if (request.uri === '/' && country !== 'JP') {
          // Redirect to localized version
          const localeMap = {
            'US': '/en',
            'GB': '/en',
            'CN': '/zh',
            'TW': '/zh-tw',
            'KR': '/ko'
          };

          if (localeMap[country]) {
            return {
              status: '302',
              statusDescription: 'Found',
              headers: {
                location: [{ value: localeMap[country] }]
              }
            };
          }
        }

        return request;
      };

  authentication:
    name: triptrip-edge-auth
    runtime: nodejs18.x
    memory: 128
    timeout: 5
    event_type: viewer-request
    description: Edge authentication for protected content

    code_overview: |
      const jwt = require('jsonwebtoken');

      exports.handler = async (event) => {
        const request = event.Records[0].cf.request;

        // Skip auth for public paths
        if (!request.uri.startsWith('/protected/')) {
          return request;
        }

        const cookies = parseCookies(request.headers.cookie);
        const token = cookies['auth_token'];

        if (!token) {
          return unauthorizedResponse();
        }

        try {
          // Verify JWT (using cached public key)
          const decoded = jwt.verify(token, publicKey, {
            algorithms: ['RS256']
          });

          // Add user info to headers
          request.headers['x-user-id'] = [{ value: decoded.sub }];
          request.headers['x-user-role'] = [{ value: decoded.role }];

          return request;
        } catch (err) {
          return unauthorizedResponse();
        }
      };
```

---

## 第5章：セキュリティ & DDoS保護

### 5.1 WAF設定 & ルール

#### 5.1.1 AWS WAF WebACL設計

```yaml
waf_configuration:
  regional_webacl:
    name: triptrip-regional-waf
    scope: REGIONAL
    description: WAF for ALB protection
    default_action: allow

    rules:
      # AWS Managed Rules
      - name: AWSManagedRulesCommonRuleSet
        priority: 1
        managed_rule_group:
          vendor_name: AWS
          name: AWSManagedRulesCommonRuleSet
        override_action: none
        visibility_config:
          sampled_requests_enabled: true
          cloudwatch_metrics_enabled: true
          metric_name: CommonRuleSetMetric

      - name: AWSManagedRulesKnownBadInputsRuleSet
        priority: 2
        managed_rule_group:
          vendor_name: AWS
          name: AWSManagedRulesKnownBadInputsRuleSet
        override_action: none

      - name: AWSManagedRulesSQLiRuleSet
        priority: 3
        managed_rule_group:
          vendor_name: AWS
          name: AWSManagedRulesSQLiRuleSet
        override_action: none

      - name: AWSManagedRulesLinuxRuleSet
        priority: 4
        managed_rule_group:
          vendor_name: AWS
          name: AWSManagedRulesLinuxRuleSet
        override_action: none

      - name: AWSManagedRulesAmazonIpReputationList
        priority: 5
        managed_rule_group:
          vendor_name: AWS
          name: AWSManagedRulesAmazonIpReputationList
        override_action: none

      - name: AWSManagedRulesAnonymousIpList
        priority: 6
        managed_rule_group:
          vendor_name: AWS
          name: AWSManagedRulesAnonymousIpList
        override_action:
          count: {}  # Count only, don't block

      - name: AWSManagedRulesBotControlRuleSet
        priority: 7
        managed_rule_group:
          vendor_name: AWS
          name: AWSManagedRulesBotControlRuleSet
          managed_rule_group_configs:
            - aws_managed_rules_bot_control_rule_set:
                inspection_level: COMMON
        override_action: none

      # Custom Rate Limiting Rules
      - name: RateLimitGeneralAPI
        priority: 10
        action: block
        statement:
          rate_based_statement:
            limit: 2000  # requests per 5 minutes
            aggregate_key_type: IP
            scope_down_statement:
              byte_match_statement:
                search_string: /api/
                field_to_match:
                  uri_path: {}
                positional_constraint: STARTS_WITH
                text_transformations:
                  - priority: 0
                    type: LOWERCASE

      - name: RateLimitAuthEndpoints
        priority: 11
        action: block
        statement:
          rate_based_statement:
            limit: 100  # requests per 5 minutes
            aggregate_key_type: IP
            scope_down_statement:
              or_statement:
                statements:
                  - byte_match_statement:
                      search_string: /api/auth/login
                      field_to_match:
                        uri_path: {}
                      positional_constraint: EXACTLY
                      text_transformations:
                        - priority: 0
                          type: LOWERCASE
                  - byte_match_statement:
                      search_string: /api/auth/register
                      field_to_match:
                        uri_path: {}
                      positional_constraint: EXACTLY
                      text_transformations:
                        - priority: 0
                          type: LOWERCASE

      - name: RateLimitSearchAPI
        priority: 12
        action: block
        statement:
          rate_based_statement:
            limit: 500  # requests per 5 minutes
            aggregate_key_type: IP
            scope_down_statement:
              byte_match_statement:
                search_string: /api/search
                field_to_match:
                  uri_path: {}
                positional_constraint: STARTS_WITH
                text_transformations:
                  - priority: 0
                    type: LOWERCASE

      # Geo Blocking (if needed)
      - name: GeoBlockHighRiskCountries
        priority: 20
        action: block
        statement:
          geo_match_statement:
            country_codes: []  # Populated as needed
            # - RU
            # - KP
        visibility_config:
          sampled_requests_enabled: true
          cloudwatch_metrics_enabled: true
          metric_name: GeoBlockMetric

      # IP Whitelist for Partners
      - name: AllowPartnerIPs
        priority: 0  # Highest priority
        action: allow
        statement:
          ip_set_reference_statement:
            arn: arn:aws:wafv2:ap-northeast-1:xxx:regional/ipset/partner-ips/xxx

      # Block Known Bad IPs
      - name: BlockBadIPs
        priority: 21
        action: block
        statement:
          ip_set_reference_statement:
            arn: arn:aws:wafv2:ap-northeast-1:xxx:regional/ipset/blocked-ips/xxx

      # Custom SQL Injection Protection
      - name: CustomSQLiProtection
        priority: 30
        action: block
        statement:
          sqli_match_statement:
            field_to_match:
              body:
                oversize_handling: CONTINUE
            text_transformations:
              - priority: 0
                type: URL_DECODE
              - priority: 1
                type: HTML_ENTITY_DECODE
              - priority: 2
                type: LOWERCASE
            sensitivity_level: HIGH

      # XSS Protection
      - name: CustomXSSProtection
        priority: 31
        action: block
        statement:
          xss_match_statement:
            field_to_match:
              body:
                oversize_handling: CONTINUE
            text_transformations:
              - priority: 0
                type: URL_DECODE
              - priority: 1
                type: HTML_ENTITY_DECODE

    visibility_config:
      sampled_requests_enabled: true
      cloudwatch_metrics_enabled: true
      metric_name: TripTripRegionalWAF

  cloudfront_webacl:
    name: triptrip-cloudfront-waf
    scope: CLOUDFRONT
    description: WAF for CloudFront protection
    default_action: allow

    rules:
      # Similar to regional with CloudFront-specific optimizations
      - name: AWSManagedRulesCommonRuleSet
        priority: 1
        managed_rule_group:
          vendor_name: AWS
          name: AWSManagedRulesCommonRuleSet
        override_action: none

      - name: RateLimitGlobal
        priority: 10
        action: block
        statement:
          rate_based_statement:
            limit: 10000  # Higher limit for CDN
            aggregate_key_type: IP

  ip_sets:
    partner_ips:
      name: partner-ips
      scope: REGIONAL
      addresses:
        - 203.0.113.0/24  # Partner A
        - 198.51.100.0/24 # Partner B

    blocked_ips:
      name: blocked-ips
      scope: REGIONAL
      addresses: []  # Dynamically updated

  regex_pattern_sets:
    sensitive_paths:
      name: sensitive-paths
      scope: REGIONAL
      regular_expressions:
        - regex_string: "^\\/admin\\/.*"
        - regex_string: "^\\/internal\\/.*"
        - regex_string: ".*\\/\\.env$"
        - regex_string: ".*\\/\\.git.*"
```

### 5.2 DDoS緩和戦略

#### 5.2.1 AWS Shield設計

```yaml
ddos_protection:
  aws_shield:
    standard:
      enabled: true
      coverage:
        - CloudFront distributions
        - Route 53 hosted zones
        - Global Accelerator
        - ALB
        - NLB
        - Elastic IPs

    advanced:
      enabled: true  # For production
      subscription: annual
      protected_resources:
        - arn:aws:elasticloadbalancing:ap-northeast-1:xxx:loadbalancer/app/triptrip-external-alb/xxx
        - arn:aws:cloudfront::xxx:distribution/E1XXXXXXXXXX
        - arn:aws:globalaccelerator::xxx:accelerator/xxx
        - arn:aws:route53:::hostedzone/ZXXXXXXXXXX

      features:
        ddos_cost_protection: enabled
        ddos_response_team: enabled
        advanced_attack_mitigations: enabled
        health_based_detection: enabled
        proactive_engagement: enabled

      response_team_access:
        enabled: true
        contacts:
          - email: security@triptrip.com
          - phone: +81-3-xxxx-xxxx

      proactive_engagement:
        enabled: true
        emergency_contacts:
          - email: sre-oncall@triptrip.com
          - phone: +81-3-xxxx-xxxx

  attack_mitigation_layers:
    layer_3_4:
      protection: AWS Shield
      capabilities:
        - SYN flood protection
        - UDP reflection amplification
        - IP fragmentation attacks
        - TCP state exhaustion

    layer_7:
      protection: AWS WAF + Shield Advanced
      capabilities:
        - HTTP floods
        - Slowloris attacks
        - Application layer attacks
        - Credential stuffing

  traffic_engineering:
    cloudfront:
      origin_shield:
        enabled: true
        region: ap-northeast-1

      connection_throttling:
        enabled: true
        max_connections_per_origin: 10000

    route53:
      shuffle_sharding: enabled
      anycast_routing: enabled

    global_accelerator:
      ddos_protection: automatic
      traffic_dials: configurable

  monitoring:
    cloudwatch_alarms:
      - name: DDoSDetected
        metric: DDoSDetected
        threshold: 1
        period: 60
        actions:
          - SNS notification
          - PagerDuty alert
          - Slack notification

      - name: VolumetricAttack
        metric: VolumetricPacketsDropped
        threshold: 1000000
        period: 300
        actions:
          - SNS notification
          - Auto-scaling trigger

    shield_dashboard:
      enabled: true
      metrics:
        - DDoSDetected
        - DDoSAttackBitsPerSecond
        - DDoSAttackPacketsPerSecond
        - DDoSAttackRequestsPerSecond

  response_playbook:
    detection:
      - Monitor Shield Advanced dashboard
      - CloudWatch alarms trigger
      - Traffic anomaly detection

    assessment:
      - Identify attack vector
      - Assess impact on availability
      - Determine attack origin

    mitigation:
      automatic:
        - Shield Standard/Advanced automatic mitigations
        - WAF rate limiting
        - CloudFront edge blocking

      manual:
        - Engage AWS DDoS Response Team (DRT)
        - Update WAF rules
        - Geo-blocking if necessary
        - Scale resources if needed

    recovery:
      - Verify service restoration
      - Review attack patterns
      - Update protection rules
      - Document incident
```

### 5.3 ゼロトラストネットワーク実装

#### 5.3.1 ゼロトラストアーキテクチャ設計

```yaml
zero_trust_architecture:
  principles:
    never_trust_always_verify:
      description: すべてのアクセスを検証
      implementation:
        - 全リクエストの認証・認可
        - 継続的な信頼性評価
        - 最小権限アクセス

    assume_breach:
      description: 侵害を前提とした設計
      implementation:
        - マイクロセグメンテーション
        - 横方向移動の制限
        - 詳細な監視・ログ

    explicit_verification:
      description: 明示的な検証
      implementation:
        - ID based access
        - Context-aware access
        - Risk-based authentication

  network_segmentation:
    macro_segmentation:
      vpc_level:
        - Production VPC
        - Development VPC
        - Management VPC
      subnet_level:
        - Public tier
        - Application tier
        - Data tier

    micro_segmentation:
      kubernetes_network_policies:
        default: deny-all
        allowed:
          - Explicit service-to-service rules
          - Health check paths
          - Monitoring endpoints

      service_mesh:
        platform: Istio
        mtls: STRICT
        authorization_policies:
          - Service-level access control
          - Request-level authorization

  identity_verification:
    machine_identity:
      certificates:
        issuer: AWS Private CA
        rotation: automatic
        lifetime: 24 hours

      service_accounts:
        kubernetes: IRSA (IAM Roles for Service Accounts)
        cloud: Instance profiles

    user_identity:
      authentication:
        method: OIDC
        provider: Auth0 / Cognito
        mfa: required

      authorization:
        model: RBAC + ABAC
        implementation:
          - Kubernetes RBAC
          - AWS IAM policies
          - Application-level roles

  access_control:
    network_access:
      vpc_level:
        - Security Groups
        - Network ACLs
        - VPC Endpoints

      kubernetes_level:
        - Network Policies
        - Istio Authorization Policies
        - Admission Controllers

    api_access:
      gateway:
        - API Gateway authorization
        - JWT validation
        - Rate limiting

      service:
        - Service mesh mTLS
        - Request authentication
        - Response signing

  continuous_verification:
    health_monitoring:
      - Endpoint health checks
      - Certificate validity
      - Access pattern analysis

    anomaly_detection:
      - Unusual traffic patterns
      - Geographic anomalies
      - Time-based anomalies

    automatic_response:
      - Connection termination
      - Access revocation
      - Incident alerting

  implementation_roadmap:
    phase_1:
      timeline: Month 1-3
      deliverables:
        - Network segmentation
        - Basic mTLS
        - Security group hardening

    phase_2:
      timeline: Month 4-6
      deliverables:
        - Service mesh deployment
        - Network policies
        - IRSA implementation

    phase_3:
      timeline: Month 7-12
      deliverables:
        - Full mTLS enforcement
        - Continuous verification
        - Automated response
```

---

## 第6章：DNS & トラフィックルーティング

### 6.1 Route 53設計

#### 6.1.1 DNS Zone設計

```yaml
route53_configuration:
  hosted_zones:
    public:
      name: triptrip.com
      type: public
      description: Main public DNS zone

      records:
        # Apex domain
        - name: triptrip.com
          type: A
          alias:
            hosted_zone_id: Z2FDTNDATAQYW2  # CloudFront
            dns_name: dxxxxxxxxxx.cloudfront.net
            evaluate_target_health: true

        - name: triptrip.com
          type: AAAA
          alias:
            hosted_zone_id: Z2FDTNDATAQYW2
            dns_name: dxxxxxxxxxx.cloudfront.net
            evaluate_target_health: true

        # www subdomain
        - name: www.triptrip.com
          type: A
          alias:
            hosted_zone_id: Z2FDTNDATAQYW2
            dns_name: dxxxxxxxxxx.cloudfront.net

        # API subdomain
        - name: api.triptrip.com
          type: A
          alias:
            hosted_zone_id: Z2FDTNDATAQYW2
            dns_name: dxxxxxxxxxx.cloudfront.net

        # CDN subdomain
        - name: cdn.triptrip.com
          type: A
          alias:
            hosted_zone_id: Z2FDTNDATAQYW2
            dns_name: dxxxxxxxxxx.cloudfront.net

        # Admin subdomain
        - name: admin.triptrip.com
          type: A
          alias:
            hosted_zone_id: Z14GRHDCWA56QT  # ALB
            dns_name: triptrip-admin-alb-xxx.ap-northeast-1.elb.amazonaws.com

        # WebSocket subdomain
        - name: ws.triptrip.com
          type: A
          alias:
            hosted_zone_id: Z14GRHDCWA56QT
            dns_name: triptrip-ws-alb-xxx.ap-northeast-1.elb.amazonaws.com

        # CAA record
        - name: triptrip.com
          type: CAA
          ttl: 300
          records:
            - 0 issue "amazon.com"
            - 0 issuewild "amazon.com"
            - 0 iodef "mailto:security@triptrip.com"

        # SPF for email
        - name: triptrip.com
          type: TXT
          ttl: 300
          records:
            - "v=spf1 include:amazonses.com -all"

        # DKIM for email
        - name: selector1._domainkey.triptrip.com
          type: CNAME
          ttl: 300
          records:
            - selector1.dkim.triptrip.com

        # DMARC
        - name: _dmarc.triptrip.com
          type: TXT
          ttl: 300
          records:
            - "v=DMARC1; p=quarantine; rua=mailto:dmarc@triptrip.com"

    private:
      name: triptrip.internal
      type: private
      vpc_associations:
        - vpc-production
        - vpc-development
      description: Internal DNS zone

      records:
        - name: db.triptrip.internal
          type: CNAME
          ttl: 60
          records:
            - triptrip-db.cluster-xxx.ap-northeast-1.rds.amazonaws.com

        - name: cache.triptrip.internal
          type: CNAME
          ttl: 60
          records:
            - triptrip-redis.xxx.ng.0001.apne1.cache.amazonaws.com

        - name: api-internal.triptrip.internal
          type: A
          alias:
            hosted_zone_id: Z14GRHDCWA56QT
            dns_name: triptrip-internal-alb-xxx.ap-northeast-1.elb.amazonaws.com

  resolver:
    inbound_endpoints:
      - name: triptrip-inbound
        direction: INBOUND
        security_group: resolver-sg
        ip_addresses:
          - subnet_id: subnet-private-app-1a
            ip: 10.0.16.10
          - subnet_id: subnet-private-app-1c
            ip: 10.0.20.10

    outbound_endpoints:
      - name: triptrip-outbound
        direction: OUTBOUND
        security_group: resolver-sg
        ip_addresses:
          - subnet_id: subnet-private-app-1a
            ip: 10.0.16.11
          - subnet_id: subnet-private-app-1c
            ip: 10.0.20.11

    forwarding_rules:
      - name: on-premises-forward
        domain: corp.triptrip.com
        target_ips:
          - 192.168.1.10
          - 192.168.1.11
```

#### 6.1.2 Geo-Routing & フェイルオーバー

```yaml
dns_routing_policies:
  geolocation_routing:
    name: api-geo
    description: Geographic routing for API
    records:
      - set_identifier: japan
        geolocation:
          country_code: JP
        alias:
          dns_name: triptrip-jp-alb.ap-northeast-1.elb.amazonaws.com
          evaluate_target_health: true

      - set_identifier: asia-pacific
        geolocation:
          continent_code: AS
        alias:
          dns_name: triptrip-sg-alb.ap-southeast-1.elb.amazonaws.com
          evaluate_target_health: true

      - set_identifier: default
        geolocation:
          country_code: "*"
        alias:
          dns_name: triptrip-jp-alb.ap-northeast-1.elb.amazonaws.com
          evaluate_target_health: true

  latency_routing:
    name: global-api
    description: Latency-based routing for global API
    records:
      - set_identifier: ap-northeast-1
        region: ap-northeast-1
        alias:
          dns_name: triptrip-jp-alb.ap-northeast-1.elb.amazonaws.com
          evaluate_target_health: true

      - set_identifier: ap-southeast-1
        region: ap-southeast-1
        alias:
          dns_name: triptrip-sg-alb.ap-southeast-1.elb.amazonaws.com
          evaluate_target_health: true

      - set_identifier: us-west-2
        region: us-west-2
        alias:
          dns_name: triptrip-us-alb.us-west-2.elb.amazonaws.com
          evaluate_target_health: true

  failover_routing:
    name: api-failover
    description: Active-passive failover
    records:
      - set_identifier: primary
        failover: PRIMARY
        alias:
          dns_name: triptrip-jp-alb.ap-northeast-1.elb.amazonaws.com
          evaluate_target_health: true
        health_check_id: xxx

      - set_identifier: secondary
        failover: SECONDARY
        alias:
          dns_name: triptrip-dr-alb.ap-northeast-3.elb.amazonaws.com
          evaluate_target_health: true
        health_check_id: yyy

  health_checks:
    api_primary:
      name: triptrip-api-primary-health
      type: HTTPS
      fqdn: api.triptrip.com
      port: 443
      resource_path: /health
      request_interval: 10
      failure_threshold: 3
      enable_sni: true
      search_string: '"status":"ok"'
      regions:
        - us-east-1
        - us-west-2
        - eu-west-1
        - ap-southeast-1
        - ap-northeast-1
      cloudwatch_alarm:
        enabled: true
        alarm_name: APIHealthCheckFailed
        actions:
          - arn:aws:sns:ap-northeast-1:xxx:triptrip-alerts

    api_secondary:
      name: triptrip-api-secondary-health
      type: HTTPS
      fqdn: api-dr.triptrip.com
      port: 443
      resource_path: /health
      request_interval: 30
      failure_threshold: 3

    calculated:
      name: triptrip-api-calculated-health
      type: CALCULATED
      child_health_checks:
        - api_primary
        - api_secondary
      health_threshold: 1  # At least 1 must be healthy

  traffic_flow:
    name: triptrip-global-routing
    description: Complex routing with traffic flow
    rules:
      - type: GEOPROXIMITY
        bias:
          ap-northeast-1: 0
          ap-southeast-1: 0
          us-west-2: 0
        endpoints:
          - region: ap-northeast-1
            endpoint: triptrip-jp-alb.ap-northeast-1.elb.amazonaws.com
          - region: ap-southeast-1
            endpoint: triptrip-sg-alb.ap-southeast-1.elb.amazonaws.com
          - region: us-west-2
            endpoint: triptrip-us-alb.us-west-2.elb.amazonaws.com
```

---

## 第7章：実装ロードマップ & 文書間参照

### 7.1 フェーズ別実装計画

```yaml
implementation_roadmap:
  phase_1_foundation:
    timeline: Month 1-2
    objectives:
      - VPC基盤構築
      - 基本的なネットワーク設計
      - 初期セキュリティ設定

    deliverables:
      - VPC with multi-AZ subnets
      - NAT Gateways (3 AZs)
      - Security Groups and NACLs
      - VPC Endpoints (S3, ECR, CloudWatch)
      - Basic ALB setup

    success_criteria:
      - EKS nodes can communicate
      - External traffic routed via ALB
      - Data tier isolated from internet

  phase_2_cdn_and_edge:
    timeline: Month 2-3
    objectives:
      - CDN設定
      - エッジ最適化
      - キャッシング戦略

    deliverables:
      - CloudFront distribution
      - Cache policies
      - Origin configurations
      - CloudFront Functions
      - Lambda@Edge functions

    success_criteria:
      - Static content served from edge
      - Cache hit ratio > 80%
      - Latency reduced by 50%

  phase_3_security_hardening:
    timeline: Month 3-4
    objectives:
      - WAF設定
      - DDoS保護
      - ゼロトラスト基盤

    deliverables:
      - AWS WAF WebACLs
      - AWS Shield Advanced (production)
      - Rate limiting rules
      - Bot protection
      - Security headers

    success_criteria:
      - All traffic protected by WAF
      - Rate limiting active
      - Security score > 90

  phase_4_advanced_routing:
    timeline: Month 4-5
    objectives:
      - 高度なDNS設定
      - グローバルルーティング
      - フェイルオーバー設定

    deliverables:
      - Route 53 health checks
      - Geo-routing policies
      - Failover configurations
      - Global Accelerator (optional)

    success_criteria:
      - Automatic failover working
      - Geographic routing active
      - DNS resolution < 50ms

  phase_5_optimization:
    timeline: Month 5-6
    objectives:
      - パフォーマンス最適化
      - コスト最適化
      - 運用自動化

    deliverables:
      - Performance tuning
      - Cost optimization (Reserved capacity)
      - Automation scripts
      - Monitoring dashboards

    success_criteria:
      - P99 latency < 200ms
      - Cost reduced by 20%
      - Automated operations

  phase_6_multi_region:
    timeline: Month 7-12
    objectives:
      - マルチリージョン展開
      - グローバルインフラ
      - DR完成

    deliverables:
      - Singapore region setup
      - US region setup (optional)
      - Cross-region replication
      - Global load balancing

    success_criteria:
      - Multi-region active
      - DR tested and verified
      - Global latency optimized
```

### 7.2 関連文書マッピング

```yaml
document_references:
  inputs:
    - document_id: Doc-IA-001
      title: Cloud Infrastructure & Deployment
      relevance: 基盤インフラストラクチャ設計
      integration_points:
        - VPC CIDR設計
        - リージョン選定
        - コスト目標

    - document_id: Doc-IA-002
      title: Kubernetes & Container Strategy
      relevance: コンテナネットワーキング
      integration_points:
        - EKS ノードネットワーク
        - サービスメッシュ
        - Pod ネットワーキング

    - document_id: Doc-SA-001
      title: System Architecture Overview
      relevance: 全体アーキテクチャ
      integration_points:
        - トラフィックパターン
        - パフォーマンス要件

  outputs:
    - document_id: Doc-SC-001
      title: Security Architecture
      dependency: ネットワークセキュリティ設計を提供

    - document_id: Doc-IA-004
      title: Monitoring & Observability
      dependency: ネットワークメトリクスとログ

    - document_id: Doc-IA-005
      title: Disaster Recovery & High Availability
      dependency: フェイルオーバー設計

  cross_references:
    - document_id: Doc-DO-001
      title: DevOps Strategy
      relationship: デプロイメントネットワーク設定

    - document_id: Doc-TV-002
      title: Technology Stack Selection
      relationship: CDN/WAF選定根拠
```

### 7.3 ネットワーク設計サマリー

```yaml
network_design_summary:
  vpc_architecture:
    cidr: 10.0.0.0/16
    availability_zones: 3 (1a, 1c, 1d)
    subnet_tiers:
      - Public (ALB, NAT)
      - Private App (EKS)
      - Private Data (RDS, ElastiCache)

  load_balancing:
    external: Application Load Balancer
    internal: Application Load Balancer
    high_performance: Network Load Balancer
    global: AWS Global Accelerator (Phase 2+)

  cdn:
    provider: Amazon CloudFront
    edge_locations: 400+
    features:
      - Origin Shield
      - CloudFront Functions
      - Lambda@Edge
      - WebP image optimization

  dns:
    provider: Amazon Route 53
    routing_policies:
      - Geolocation
      - Latency-based
      - Failover
    health_checks: enabled

  security:
    waf: AWS WAF with managed rules
    ddos: AWS Shield Advanced
    encryption: TLS 1.3
    zero_trust: Progressive implementation

  performance_targets:
    api_latency_p99: < 200ms
    cdn_cache_hit: > 80%
    availability: 99.99%
```

---

## 結論

本文書は、TripTripプラットフォームのネットワーキングおよびCDNアーキテクチャを包括的に定義しました。

### 主要な設計決定

1. **マルチAZ VPC設計**: 3つのAZにまたがる高可用性ネットワーク
2. **3層サブネット分離**: パブリック/アプリ/データ層の論理分離
3. **CloudFront CDN**: グローバルエッジ配信とキャッシング最適化
4. **AWS WAF + Shield**: 包括的なDDoS保護とWebアプリケーションファイアウォール
5. **Route 53 Advanced Routing**: 地理的ルーティングとフェイルオーバー
6. **ゼロトラストネットワーク**: 段階的な信頼性検証の実装

### 期待される成果

- 99.99%の可用性SLA達成
- P99 APIレイテンシ 200ms以下
- CDNキャッシュヒット率 80%以上
- DDoS攻撃からの完全保護
- グローバルスケールへの拡張性

これらの設計により、TripTripは高パフォーマンス、高セキュリティ、グローバルスケールのネットワーク基盤を実現します。

---

**文書情報**
- Document ID: Doc-IA-003
- Version: 1.0.0
- Last Updated: 2026-01-21
- Status: Draft
- Total Lines: 1,800+
- Author: Technical Architecture Agent

**関連文書**
- Doc-IA-001: Cloud Infrastructure & Deployment
- Doc-IA-002: Kubernetes & Container Strategy
- Doc-IA-004: Monitoring, Logging & Observability
- Doc-IA-005: Disaster Recovery & High Availability
- Doc-SC-001: Security Architecture

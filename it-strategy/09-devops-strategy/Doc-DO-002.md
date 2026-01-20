# Doc-DO-002: TripTrip Infrastructure as Code（Terraformモジュール設計詳細）

## エグゼクティブサマリー

本文書は、TripTripプラットフォームのInfrastructure as Code（IaC）戦略における**Terraformモジュール設計の詳細実装仕様**を定義します。Doc-DM-002で策定したIaCの概念とベストプラクティスを基盤とし、具体的なTerraformモジュール構造、ディレクトリ設計、状態管理戦略、CI/CD統合を提供します。HashiCorpのベストプラクティスとGoogle/Amazon/Netflixレベルのインフラ自動化品質を実現し、再現可能で監査可能なインフラストラクチャを構築します。

---

## 第1章 はじめに & コンテキスト

### 1.1 IaC戦略の目標

#### 1.1.1 戦略的目標

```yaml
iac_strategic_goals:
  primary_objectives:
    reproducibility:
      description: "インフラストラクチャの完全な再現性確保"
      metrics:
        - "コードからの環境構築成功率: 100%"
        - "環境間の構成差異: 0"
        - "災害復旧時間: < 1時間"

    auditability:
      description: "変更履歴の完全な追跡可能性"
      metrics:
        - "変更のGit履歴: 100%"
        - "Plan/Applyのログ保持: 90日"
        - "コンプライアンス監査合格率: 100%"

    automation:
      description: "手動操作の排除"
      metrics:
        - "自動化率: > 95%"
        - "手動変更件数: 0"
        - "デプロイ時間: < 15分"

    security:
      description: "セキュリティのコード化"
      metrics:
        - "ポリシー適用率: 100%"
        - "セキュリティスキャン合格率: 100%"
        - "暗号化適用率: 100%"

  secondary_objectives:
    cost_optimization:
      description: "コスト管理と最適化"
      features:
        - Infracost統合
        - リソースタグ付け
        - 未使用リソース検出

    developer_experience:
      description: "開発者体験の向上"
      features:
        - モジュール再利用
        - 明確なドキュメント
        - 簡潔なインターフェース
```

#### 1.1.2 成熟度モデル

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    IaC 成熟度モデル                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Level 1: Manual                                                         │
│  ─────────────────                                                       │
│  - コンソール操作                                                         │
│  - ドキュメントベース                                                     │
│  - 属人化                                                                │
│                                                                          │
│  Level 2: Scripted              ◄── 現在の状態                          │
│  ─────────────────                                                       │
│  - スクリプト自動化                                                       │
│  - 部分的なコード管理                                                     │
│  - 手動実行                                                              │
│                                                                          │
│  Level 3: Declarative           ◄── Phase 1 目標                        │
│  ─────────────────                                                       │
│  - Terraform/IaC                                                         │
│  - バージョン管理                                                         │
│  - CI/CD統合                                                             │
│                                                                          │
│  Level 4: Policy-Driven         ◄── Phase 2 目標                        │
│  ─────────────────                                                       │
│  - ポリシーコード化                                                       │
│  - 自動コンプライアンス                                                   │
│  - ガバナンス自動化                                                       │
│                                                                          │
│  Level 5: Autonomous            ◄── Phase 3 目標                        │
│  ─────────────────                                                       │
│  - 自動スケーリング                                                       │
│  - 自己修復                                                              │
│  - 予測的管理                                                            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Terraform選定理由

#### 1.2.1 ツール比較と選定

```yaml
iac_tool_comparison:
  terraform:
    pros:
      - マルチクラウド対応
      - 豊富なプロバイダーエコシステム
      - 宣言的構文
      - 成熟したコミュニティ
      - 状態管理の柔軟性
    cons:
      - 学習曲線
      - 状態管理の複雑性
      - HCL固有の制約
    use_case: "TripTripのメインIaCツール"

  pulumi:
    pros:
      - 汎用プログラミング言語
      - 型安全性
      - テスト容易性
    cons:
      - 小さいエコシステム
      - ロックイン懸念
    use_case: "将来的な検討対象"

  aws_cdk:
    pros:
      - AWS深い統合
      - 型安全性
      - CloudFormation互換
    cons:
      - AWS限定
      - CloudFormation制約
    use_case: "AWSのみの場合"

  decision:
    selected: Terraform
    version: ">= 1.6.0"
    rationale:
      - GCPをプライマリクラウドとして使用
      - マルチクラウド将来対応
      - チーム既存知識
      - 豊富な参考資料
```

### 1.3 現在のインフラ状況

#### 1.3.1 既存リソースインベントリ

```yaml
existing_infrastructure:
  gcp_project:
    production:
      project_id: triptrip-production
      region: asia-northeast1
      zones: [a, b, c]

    staging:
      project_id: triptrip-staging
      region: asia-northeast1
      zones: [a, b]

    development:
      project_id: triptrip-development
      region: asia-northeast1
      zones: [a]

  current_resources:
    compute:
      - type: Cloud Run
        services:
          - triptrip-api
          - triptrip-web
        management: "コンソール + gcloud"

      - type: Cloud Functions
        functions:
          - image-processor
          - notification-sender
        management: "コンソール"

    data:
      - type: Cloud SQL (PostgreSQL 16)
        instances:
          - triptrip-db-prod
          - triptrip-db-staging
        management: "コンソール"

      - type: Cloud Memorystore (Redis)
        instances:
          - triptrip-cache-prod
        management: "コンソール"

      - type: Cloud Storage
        buckets:
          - triptrip-assets
          - triptrip-uploads
          - triptrip-backups
        management: "コンソール + gsutil"

    networking:
      - type: VPC
        networks:
          - triptrip-vpc-prod
          - triptrip-vpc-staging
        management: "コンソール"

      - type: Cloud Load Balancing
        load_balancers:
          - triptrip-api-lb
        management: "コンソール"

  migration_priority:
    high:
      - VPC/ネットワーク
      - Cloud SQL
      - Cloud Storage
    medium:
      - Cloud Run
      - Load Balancer
      - Cloud Memorystore
    low:
      - Cloud Functions
      - その他補助サービス
```

---

## 第2章 Terraformアーキテクチャ

### 2.1 リポジトリ構造

#### 2.1.1 ディレクトリ構造設計

```
triptrip-infrastructure/
├── README.md                           # プロジェクト概要
├── CONTRIBUTING.md                     # コントリビューションガイド
├── .gitignore
├── .pre-commit-config.yaml             # Pre-commit hooks
├── .tflint.hcl                         # TFLint設定
├── .terraform-version                  # tfenv用バージョン指定
│
├── modules/                            # 再利用可能なモジュール
│   ├── networking/
│   │   ├── vpc/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── versions.tf
│   │   │   └── README.md
│   │   ├── firewall/
│   │   ├── cloud-nat/
│   │   └── load-balancer/
│   │
│   ├── compute/
│   │   ├── gke/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── node-pools.tf
│   │   │   ├── addons.tf
│   │   │   └── README.md
│   │   ├── cloud-run/
│   │   └── cloud-functions/
│   │
│   ├── data/
│   │   ├── cloud-sql/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── replicas.tf
│   │   │   ├── backups.tf
│   │   │   └── README.md
│   │   ├── memorystore/
│   │   ├── cloud-storage/
│   │   └── bigquery/
│   │
│   ├── security/
│   │   ├── iam/
│   │   ├── secret-manager/
│   │   ├── kms/
│   │   └── security-center/
│   │
│   ├── observability/
│   │   ├── monitoring/
│   │   ├── logging/
│   │   └── alerting/
│   │
│   └── common/
│       ├── labels/
│       ├── naming/
│       └── project-services/
│
├── environments/                       # 環境別設定
│   ├── production/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── backend.tf
│   │   ├── terraform.tfvars
│   │   └── providers.tf
│   │
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── backend.tf
│   │   ├── terraform.tfvars
│   │   └── providers.tf
│   │
│   └── development/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── backend.tf
│       ├── terraform.tfvars
│       └── providers.tf
│
├── global/                             # グローバルリソース
│   ├── dns/
│   ├── artifact-registry/
│   ├── organization-policies/
│   └── iam-roles/
│
├── scripts/                            # ヘルパースクリプト
│   ├── init-backend.sh
│   ├── import-existing.sh
│   ├── validate-all.sh
│   └── cost-estimate.sh
│
├── policies/                           # OPA/Sentinel ポリシー
│   ├── opa/
│   │   ├── encryption.rego
│   │   ├── networking.rego
│   │   └── tagging.rego
│   └── sentinel/
│
├── tests/                              # Terratest テスト
│   ├── vpc_test.go
│   ├── gke_test.go
│   └── cloud_sql_test.go
│
└── docs/                               # ドキュメント
    ├── architecture.md
    ├── modules.md
    ├── runbooks/
    └── adr/                            # Architecture Decision Records
```

#### 2.1.2 ファイル命名規則

```yaml
naming_conventions:
  terraform_files:
    main.tf: "メインリソース定義"
    variables.tf: "入力変数定義"
    outputs.tf: "出力値定義"
    versions.tf: "バージョン制約"
    providers.tf: "プロバイダー設定"
    backend.tf: "バックエンド設定"
    locals.tf: "ローカル変数"
    data.tf: "データソース"

  module_specific:
    "[resource].tf": "リソース種類ごとのファイル"
    "例: node-pools.tf, replicas.tf, backups.tf"

  environment_specific:
    "terraform.tfvars": "環境変数値"
    "terraform.auto.tfvars": "自動ロード変数"

  prohibited:
    - "*.tfvars.json (secrets含む可能性)"
    - "terraform.tfstate (リモート管理)"
    - "*.pem, *.key (シークレット)"
```

### 2.2 モジュール階層設計

#### 2.2.1 モジュール階層パターン

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    モジュール階層構造                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Layer 1: Root Modules (環境)                                           │
│  ────────────────────────────                                            │
│  environments/production/main.tf                                         │
│  environments/staging/main.tf                                            │
│  environments/development/main.tf                                        │
│        │                                                                 │
│        │ module呼び出し                                                  │
│        ▼                                                                 │
│  Layer 2: Composition Modules (組み合わせ)                               │
│  ────────────────────────────────────────                                │
│  modules/platform/            # TripTrip Platform全体                   │
│  modules/data-platform/       # データ基盤全体                           │
│  modules/security-baseline/   # セキュリティベースライン                  │
│        │                                                                 │
│        │ module呼び出し                                                  │
│        ▼                                                                 │
│  Layer 3: Service Modules (サービス単位)                                 │
│  ──────────────────────────────────────                                  │
│  modules/compute/gke/                                                    │
│  modules/data/cloud-sql/                                                 │
│  modules/networking/vpc/                                                 │
│        │                                                                 │
│        │ リソース定義                                                    │
│        ▼                                                                 │
│  Layer 4: Primitive Resources                                            │
│  ────────────────────────────                                            │
│  google_container_cluster                                                │
│  google_sql_database_instance                                            │
│  google_compute_network                                                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 2.2.2 モジュール設計原則

```yaml
module_design_principles:
  1_single_responsibility:
    description: "1つのモジュールは1つの責務"
    examples:
      good:
        - modules/compute/gke (GKEクラスタのみ)
        - modules/data/cloud-sql (Cloud SQLのみ)
      bad:
        - modules/everything (全リソース)
        - modules/compute-and-data (混在)

  2_loose_coupling:
    description: "モジュール間の疎結合"
    implementation:
      - データソースによる参照
      - 出力値による連携
      - ハードコード禁止

  3_high_cohesion:
    description: "関連リソースの高凝集"
    examples:
      gke_module:
        - google_container_cluster
        - google_container_node_pool
        - google_project_iam_member (GKEサービスアカウント)
        - google_compute_firewall (GKE用)

  4_explicit_dependencies:
    description: "明示的な依存関係"
    implementation:
      - depends_on は最終手段
      - 参照による暗黙的依存を優先
      - モジュール出力を活用

  5_sensible_defaults:
    description: "適切なデフォルト値"
    guidelines:
      - セキュアなデフォルト
      - 一般的なユースケースをカバー
      - 本番向けとのギャップを最小化
```

### 2.3 状態管理戦略

#### 2.3.1 リモートバックエンド設計

```hcl
# environments/production/backend.tf
terraform {
  backend "gcs" {
    bucket = "triptrip-terraform-state-prod"
    prefix = "terraform/state"

    # 状態ロックのための設定
    # GCSは自動的にオブジェクトレベルのロックを提供
  }
}

# バックエンド用バケット作成（bootstrap）
# global/terraform-backend/main.tf
resource "google_storage_bucket" "terraform_state" {
  for_each = toset(["prod", "staging", "dev"])

  name     = "triptrip-terraform-state-${each.key}"
  location = "asia-northeast1"
  project  = "triptrip-${each.key == "prod" ? "production" : each.key}"

  # バージョニング有効化（状態履歴保持）
  versioning {
    enabled = true
  }

  # ライフサイクルルール
  lifecycle_rule {
    condition {
      num_newer_versions = 30
    }
    action {
      type = "Delete"
    }
  }

  # 暗号化
  encryption {
    default_kms_key_name = google_kms_crypto_key.terraform_state.id
  }

  # 公開アクセス禁止
  uniform_bucket_level_access = true

  labels = {
    purpose     = "terraform-state"
    environment = each.key
    managed_by  = "terraform-bootstrap"
  }
}

# 状態バケット用KMSキー
resource "google_kms_key_ring" "terraform" {
  name     = "terraform-state-keyring"
  location = "asia-northeast1"
  project  = var.project_id
}

resource "google_kms_crypto_key" "terraform_state" {
  name     = "terraform-state-key"
  key_ring = google_kms_key_ring.terraform.id

  rotation_period = "7776000s" # 90日

  lifecycle {
    prevent_destroy = true
  }
}
```

#### 2.3.2 状態分離戦略

```yaml
state_separation_strategy:
  by_environment:
    description: "環境ごとに独立した状態ファイル"
    structure:
      - production: gs://triptrip-terraform-state-prod/terraform/state/default.tfstate
      - staging: gs://triptrip-terraform-state-staging/terraform/state/default.tfstate
      - development: gs://triptrip-terraform-state-dev/terraform/state/default.tfstate
    benefits:
      - 環境間の影響分離
      - 権限分離
      - 並列実行可能

  by_component:
    description: "コンポーネントごとの状態分離（大規模向け）"
    structure:
      production:
        - gs://triptrip-terraform-state-prod/terraform/networking/default.tfstate
        - gs://triptrip-terraform-state-prod/terraform/compute/default.tfstate
        - gs://triptrip-terraform-state-prod/terraform/data/default.tfstate
    benefits:
      - 影響範囲の限定
      - 高速なplan/apply
      - チーム別管理

  recommended_approach:
    triptrip_initial: "by_environment"
    triptrip_growth: "by_component（100リソース超過時）"
```

#### 2.3.3 状態ロックと競合解決

```yaml
state_locking:
  gcs_locking:
    mechanism: "オブジェクトレベルロック"
    timeout: "20分"
    implementation: "自動（backend設定で有効）"

  lock_contention_handling:
    prevention:
      - CI/CDでのシリアル実行
      - 手動実行の制限
      - ロック保持者の通知

    resolution:
      manual_unlock:
        command: "terraform force-unlock LOCK_ID"
        caution: "本当にロックが不要か確認"

      automatic_timeout:
        duration: "20分"
        after_timeout: "自動解放"

  best_practices:
    - ロック取得失敗時は待機
    - 強制アンロックは最終手段
    - ロック中の作業者に連絡
```

---

## 第3章 コアモジュール設計

### 3.1 ネットワーキングモジュール

#### 3.1.1 VPCモジュール

```hcl
# modules/networking/vpc/main.tf

# VPCネットワーク
resource "google_compute_network" "main" {
  name                            = var.network_name
  project                         = var.project_id
  auto_create_subnetworks         = false
  routing_mode                    = var.routing_mode
  delete_default_routes_on_create = var.delete_default_routes

  description = "TripTrip ${var.environment} VPC"
}

# サブネット
resource "google_compute_subnetwork" "subnets" {
  for_each = var.subnets

  name                     = each.value.name
  project                  = var.project_id
  region                   = each.value.region
  network                  = google_compute_network.main.id
  ip_cidr_range            = each.value.ip_cidr_range
  private_ip_google_access = true

  # セカンダリレンジ（GKE用）
  dynamic "secondary_ip_range" {
    for_each = lookup(each.value, "secondary_ip_ranges", [])
    content {
      range_name    = secondary_ip_range.value.range_name
      ip_cidr_range = secondary_ip_range.value.ip_cidr_range
    }
  }

  # フローログ
  dynamic "log_config" {
    for_each = var.enable_flow_logs ? [1] : []
    content {
      aggregation_interval = "INTERVAL_5_SEC"
      flow_sampling        = 0.5
      metadata             = "INCLUDE_ALL_METADATA"
    }
  }

  description = "TripTrip ${var.environment} subnet: ${each.value.name}"
}

# Cloud Router (NAT用)
resource "google_compute_router" "main" {
  count = var.enable_nat ? 1 : 0

  name    = "${var.network_name}-router"
  project = var.project_id
  region  = var.region
  network = google_compute_network.main.id

  bgp {
    asn = var.router_asn
  }
}

# Cloud NAT
resource "google_compute_router_nat" "main" {
  count = var.enable_nat ? 1 : 0

  name                               = "${var.network_name}-nat"
  project                            = var.project_id
  router                             = google_compute_router.main[0].name
  region                             = var.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"

  log_config {
    enable = true
    filter = "ERRORS_ONLY"
  }

  # NAT タイムアウト設定
  min_ports_per_vm                    = 64
  max_ports_per_vm                    = 65536
  enable_endpoint_independent_mapping = false

  tcp_established_idle_timeout_sec = 1200
  tcp_transitory_idle_timeout_sec  = 30
  udp_idle_timeout_sec             = 30
}

# Firewall ルール（デフォルト）
resource "google_compute_firewall" "allow_internal" {
  name    = "${var.network_name}-allow-internal"
  project = var.project_id
  network = google_compute_network.main.name

  allow {
    protocol = "tcp"
    ports    = ["0-65535"]
  }

  allow {
    protocol = "udp"
    ports    = ["0-65535"]
  }

  allow {
    protocol = "icmp"
  }

  source_ranges = var.internal_ranges
  priority      = 65534

  description = "Allow internal traffic"
}

resource "google_compute_firewall" "deny_all_egress" {
  count = var.enable_default_deny ? 1 : 0

  name    = "${var.network_name}-deny-all-egress"
  project = var.project_id
  network = google_compute_network.main.name

  deny {
    protocol = "all"
  }

  direction          = "EGRESS"
  destination_ranges = ["0.0.0.0/0"]
  priority           = 65535

  description = "Deny all egress by default"
}
```

```hcl
# modules/networking/vpc/variables.tf

variable "project_id" {
  description = "GCPプロジェクトID"
  type        = string
}

variable "network_name" {
  description = "VPCネットワーク名"
  type        = string
}

variable "environment" {
  description = "環境名 (production, staging, development)"
  type        = string
  validation {
    condition     = contains(["production", "staging", "development"], var.environment)
    error_message = "environment は production, staging, development のいずれかである必要があります。"
  }
}

variable "region" {
  description = "デフォルトリージョン"
  type        = string
  default     = "asia-northeast1"
}

variable "routing_mode" {
  description = "ルーティングモード (GLOBAL or REGIONAL)"
  type        = string
  default     = "GLOBAL"
}

variable "delete_default_routes" {
  description = "デフォルトルートを削除するか"
  type        = bool
  default     = false
}

variable "subnets" {
  description = "サブネット設定"
  type = map(object({
    name          = string
    region        = string
    ip_cidr_range = string
    secondary_ip_ranges = optional(list(object({
      range_name    = string
      ip_cidr_range = string
    })), [])
  }))
}

variable "enable_nat" {
  description = "Cloud NATを有効にするか"
  type        = bool
  default     = true
}

variable "router_asn" {
  description = "Cloud Router ASN"
  type        = number
  default     = 64514
}

variable "enable_flow_logs" {
  description = "VPCフローログを有効にするか"
  type        = bool
  default     = true
}

variable "internal_ranges" {
  description = "内部通信許可CIDRレンジ"
  type        = list(string)
  default     = ["10.0.0.0/8"]
}

variable "enable_default_deny" {
  description = "デフォルトEgressを拒否するか"
  type        = bool
  default     = false
}
```

```hcl
# modules/networking/vpc/outputs.tf

output "network_id" {
  description = "VPCネットワークID"
  value       = google_compute_network.main.id
}

output "network_name" {
  description = "VPCネットワーク名"
  value       = google_compute_network.main.name
}

output "network_self_link" {
  description = "VPCネットワークself_link"
  value       = google_compute_network.main.self_link
}

output "subnets" {
  description = "サブネット情報"
  value = {
    for k, v in google_compute_subnetwork.subnets : k => {
      id            = v.id
      name          = v.name
      ip_cidr_range = v.ip_cidr_range
      region        = v.region
      self_link     = v.self_link
      secondary_ip_ranges = {
        for range in v.secondary_ip_range : range.range_name => range.ip_cidr_range
      }
    }
  }
}

output "router_name" {
  description = "Cloud Router名"
  value       = var.enable_nat ? google_compute_router.main[0].name : null
}

output "nat_name" {
  description = "Cloud NAT名"
  value       = var.enable_nat ? google_compute_router_nat.main[0].name : null
}
```

### 3.2 コンピュートモジュール（GKE）

#### 3.2.1 GKEクラスタモジュール

```hcl
# modules/compute/gke/main.tf

# GKE クラスタ
resource "google_container_cluster" "main" {
  name     = var.cluster_name
  project  = var.project_id
  location = var.regional ? var.region : var.zones[0]

  # リージョナルクラスタの場合
  node_locations = var.regional ? var.zones : null

  # 初期ノードプールは削除（別途管理）
  remove_default_node_pool = true
  initial_node_count       = 1

  # ネットワーク設定
  network    = var.network_id
  subnetwork = var.subnetwork_id

  # VPC-native クラスタ
  ip_allocation_policy {
    cluster_secondary_range_name  = var.pods_range_name
    services_secondary_range_name = var.services_range_name
  }

  # プライベートクラスタ
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = var.enable_private_endpoint
    master_ipv4_cidr_block  = var.master_ipv4_cidr_block

    master_global_access_config {
      enabled = true
    }
  }

  # マスター認可ネットワーク
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

  # リリースチャンネル
  release_channel {
    channel = var.release_channel
  }

  # セキュリティ設定
  enable_shielded_nodes = true

  # Binary Authorization
  dynamic "binary_authorization" {
    for_each = var.enable_binary_authorization ? [1] : []
    content {
      evaluation_mode = "PROJECT_SINGLETON_POLICY_ENFORCE"
    }
  }

  # ネットワークポリシー
  network_policy {
    enabled  = true
    provider = "CALICO"
  }

  # ロギングとモニタリング
  logging_config {
    enable_components = [
      "SYSTEM_COMPONENTS",
      "WORKLOADS"
    ]
  }

  monitoring_config {
    enable_components = [
      "SYSTEM_COMPONENTS",
      "WORKLOADS"
    ]

    managed_prometheus {
      enabled = var.enable_managed_prometheus
    }
  }

  # メンテナンスウィンドウ
  maintenance_policy {
    daily_maintenance_window {
      start_time = var.maintenance_start_time
    }

    dynamic "maintenance_exclusion" {
      for_each = var.maintenance_exclusions
      content {
        exclusion_name = maintenance_exclusion.value.name
        start_time     = maintenance_exclusion.value.start_time
        end_time       = maintenance_exclusion.value.end_time
        exclusion_options {
          scope = maintenance_exclusion.value.scope
        }
      }
    }
  }

  # アドオン
  addons_config {
    http_load_balancing {
      disabled = false
    }

    horizontal_pod_autoscaling {
      disabled = false
    }

    network_policy_config {
      disabled = false
    }

    gcs_fuse_csi_driver_config {
      enabled = var.enable_gcs_fuse
    }

    gce_persistent_disk_csi_driver_config {
      enabled = true
    }

    gke_backup_agent_config {
      enabled = var.enable_backup
    }
  }

  # DNS
  dns_config {
    cluster_dns        = "CLOUD_DNS"
    cluster_dns_scope  = "VPC_SCOPE"
    cluster_dns_domain = var.cluster_dns_domain
  }

  # コストアロケーション
  cost_management_config {
    enabled = true
  }

  # 削除保護
  deletion_protection = var.deletion_protection

  resource_labels = merge(var.labels, {
    environment = var.environment
    cluster     = var.cluster_name
  })

  lifecycle {
    ignore_changes = [
      node_pool,
      initial_node_count,
    ]
  }
}
```

```hcl
# modules/compute/gke/node-pools.tf

# ノードプール
resource "google_container_node_pool" "pools" {
  for_each = var.node_pools

  name     = each.key
  project  = var.project_id
  cluster  = google_container_cluster.main.name
  location = google_container_cluster.main.location

  # ノード数設定
  initial_node_count = lookup(each.value, "initial_node_count", 1)

  # オートスケーリング
  dynamic "autoscaling" {
    for_each = lookup(each.value, "autoscaling", null) != null ? [each.value.autoscaling] : []
    content {
      min_node_count  = autoscaling.value.min_node_count
      max_node_count  = autoscaling.value.max_node_count
      location_policy = lookup(autoscaling.value, "location_policy", "BALANCED")
    }
  }

  # ノード管理
  management {
    auto_repair  = lookup(each.value, "auto_repair", true)
    auto_upgrade = lookup(each.value, "auto_upgrade", true)
  }

  # アップグレード設定
  upgrade_settings {
    max_surge       = lookup(each.value, "max_surge", 1)
    max_unavailable = lookup(each.value, "max_unavailable", 0)
    strategy        = lookup(each.value, "upgrade_strategy", "SURGE")

    dynamic "blue_green_settings" {
      for_each = lookup(each.value, "upgrade_strategy", "SURGE") == "BLUE_GREEN" ? [1] : []
      content {
        node_pool_soak_duration = lookup(each.value, "node_pool_soak_duration", "3600s")
        standard_rollout_policy {
          batch_percentage    = lookup(each.value, "batch_percentage", 0.2)
          batch_soak_duration = lookup(each.value, "batch_soak_duration", "0s")
        }
      }
    }
  }

  # ノード設定
  node_config {
    machine_type = each.value.machine_type
    disk_size_gb = lookup(each.value, "disk_size_gb", 100)
    disk_type    = lookup(each.value, "disk_type", "pd-standard")
    image_type   = lookup(each.value, "image_type", "COS_CONTAINERD")

    # Spot インスタンス
    spot = lookup(each.value, "spot", false)

    # サービスアカウント
    service_account = lookup(each.value, "service_account", null)
    oauth_scopes = lookup(each.value, "oauth_scopes", [
      "https://www.googleapis.com/auth/cloud-platform"
    ])

    # Shielded Instance
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }

    # Workload Identity
    workload_metadata_config {
      mode = "GKE_METADATA"
    }

    # ラベル
    labels = merge(
      var.labels,
      lookup(each.value, "labels", {}),
      {
        node_pool = each.key
      }
    )

    # Taints
    dynamic "taint" {
      for_each = lookup(each.value, "taints", [])
      content {
        key    = taint.value.key
        value  = taint.value.value
        effect = taint.value.effect
      }
    }

    # メタデータ
    metadata = merge(
      {
        "disable-legacy-endpoints" = "true"
      },
      lookup(each.value, "metadata", {})
    )

    # タグ
    tags = concat(
      ["gke-${var.cluster_name}"],
      lookup(each.value, "tags", [])
    )

    # GPU（オプション）
    dynamic "guest_accelerator" {
      for_each = lookup(each.value, "accelerator_type", null) != null ? [1] : []
      content {
        type  = each.value.accelerator_type
        count = lookup(each.value, "accelerator_count", 1)
        gpu_driver_installation_config {
          gpu_driver_version = "LATEST"
        }
      }
    }
  }

  lifecycle {
    ignore_changes = [
      initial_node_count,
    ]
  }
}
```

```hcl
# modules/compute/gke/variables.tf

variable "project_id" {
  description = "GCPプロジェクトID"
  type        = string
}

variable "cluster_name" {
  description = "GKEクラスタ名"
  type        = string
}

variable "environment" {
  description = "環境名"
  type        = string
}

variable "region" {
  description = "リージョン"
  type        = string
  default     = "asia-northeast1"
}

variable "zones" {
  description = "ゾーンリスト"
  type        = list(string)
  default     = ["asia-northeast1-a", "asia-northeast1-b", "asia-northeast1-c"]
}

variable "regional" {
  description = "リージョナルクラスタにするか"
  type        = bool
  default     = true
}

variable "network_id" {
  description = "VPCネットワークID"
  type        = string
}

variable "subnetwork_id" {
  description = "サブネットID"
  type        = string
}

variable "pods_range_name" {
  description = "Pod用セカンダリレンジ名"
  type        = string
}

variable "services_range_name" {
  description = "Service用セカンダリレンジ名"
  type        = string
}

variable "enable_private_endpoint" {
  description = "プライベートエンドポイントを有効にするか"
  type        = bool
  default     = false
}

variable "master_ipv4_cidr_block" {
  description = "マスターIPv4 CIDRブロック"
  type        = string
  default     = "172.16.0.0/28"
}

variable "master_authorized_networks" {
  description = "マスター認可ネットワーク"
  type = list(object({
    cidr_block   = string
    display_name = string
  }))
  default = []
}

variable "release_channel" {
  description = "リリースチャンネル (UNSPECIFIED, RAPID, REGULAR, STABLE)"
  type        = string
  default     = "REGULAR"
}

variable "enable_binary_authorization" {
  description = "Binary Authorizationを有効にするか"
  type        = bool
  default     = false
}

variable "enable_managed_prometheus" {
  description = "Managed Prometheusを有効にするか"
  type        = bool
  default     = true
}

variable "maintenance_start_time" {
  description = "メンテナンス開始時刻 (UTC)"
  type        = string
  default     = "17:00" # JST 02:00
}

variable "maintenance_exclusions" {
  description = "メンテナンス除外期間"
  type = list(object({
    name       = string
    start_time = string
    end_time   = string
    scope      = string
  }))
  default = []
}

variable "enable_gcs_fuse" {
  description = "GCS Fuse CSI Driverを有効にするか"
  type        = bool
  default     = false
}

variable "enable_backup" {
  description = "GKE Backupを有効にするか"
  type        = bool
  default     = true
}

variable "cluster_dns_domain" {
  description = "クラスタDNSドメイン"
  type        = string
  default     = "cluster.local"
}

variable "deletion_protection" {
  description = "削除保護を有効にするか"
  type        = bool
  default     = true
}

variable "labels" {
  description = "リソースラベル"
  type        = map(string)
  default     = {}
}

variable "node_pools" {
  description = "ノードプール設定"
  type = map(object({
    machine_type       = string
    initial_node_count = optional(number, 1)
    disk_size_gb       = optional(number, 100)
    disk_type          = optional(string, "pd-standard")
    image_type         = optional(string, "COS_CONTAINERD")
    spot               = optional(bool, false)
    service_account    = optional(string)
    oauth_scopes       = optional(list(string))
    labels             = optional(map(string), {})
    tags               = optional(list(string), [])
    metadata           = optional(map(string), {})
    auto_repair        = optional(bool, true)
    auto_upgrade       = optional(bool, true)
    max_surge          = optional(number, 1)
    max_unavailable    = optional(number, 0)
    upgrade_strategy   = optional(string, "SURGE")
    autoscaling = optional(object({
      min_node_count  = number
      max_node_count  = number
      location_policy = optional(string, "BALANCED")
    }))
    taints = optional(list(object({
      key    = string
      value  = string
      effect = string
    })), [])
    accelerator_type  = optional(string)
    accelerator_count = optional(number, 1)
  }))
}
```

### 3.3 データストアモジュール

#### 3.3.1 Cloud SQLモジュール

```hcl
# modules/data/cloud-sql/main.tf

# Cloud SQL インスタンス
resource "google_sql_database_instance" "main" {
  name                = var.instance_name
  project             = var.project_id
  region              = var.region
  database_version    = var.database_version
  deletion_protection = var.deletion_protection

  settings {
    tier              = var.tier
    availability_type = var.availability_type
    disk_size         = var.disk_size
    disk_type         = var.disk_type
    disk_autoresize   = var.disk_autoresize

    # ネットワーク
    ip_configuration {
      ipv4_enabled       = false
      private_network    = var.network_id
      enable_private_path_for_google_cloud_services = true

      dynamic "authorized_networks" {
        for_each = var.authorized_networks
        content {
          name  = authorized_networks.value.name
          value = authorized_networks.value.value
        }
      }
    }

    # バックアップ
    backup_configuration {
      enabled                        = true
      start_time                     = var.backup_start_time
      point_in_time_recovery_enabled = var.point_in_time_recovery
      location                       = var.backup_location
      transaction_log_retention_days = var.transaction_log_retention_days

      backup_retention_settings {
        retained_backups = var.retained_backups
        retention_unit   = "COUNT"
      }
    }

    # メンテナンス
    maintenance_window {
      day          = var.maintenance_window_day
      hour         = var.maintenance_window_hour
      update_track = var.maintenance_update_track
    }

    # インサイト
    insights_config {
      query_insights_enabled  = true
      query_string_length     = 1024
      record_application_tags = true
      record_client_address   = true
      query_plans_per_minute  = 5
    }

    # データベースフラグ
    dynamic "database_flags" {
      for_each = var.database_flags
      content {
        name  = database_flags.value.name
        value = database_flags.value.value
      }
    }

    # ラベル
    user_labels = merge(var.labels, {
      environment = var.environment
    })
  }

  lifecycle {
    prevent_destroy = true

    ignore_changes = [
      settings[0].disk_size,
    ]
  }
}

# データベース
resource "google_sql_database" "databases" {
  for_each = toset(var.databases)

  name     = each.value
  project  = var.project_id
  instance = google_sql_database_instance.main.name
  charset  = var.charset
}

# ユーザー
resource "google_sql_user" "users" {
  for_each = var.users

  name     = each.key
  project  = var.project_id
  instance = google_sql_database_instance.main.name
  type     = lookup(each.value, "type", "BUILT_IN")

  # IAMユーザーでない場合はパスワード設定
  password = lookup(each.value, "type", "BUILT_IN") == "BUILT_IN" ? (
    lookup(each.value, "password", null) != null ? each.value.password : random_password.user_password[each.key].result
  ) : null

  deletion_policy = "ABANDON"
}

# ランダムパスワード生成
resource "random_password" "user_password" {
  for_each = {
    for k, v in var.users : k => v
    if lookup(v, "type", "BUILT_IN") == "BUILT_IN" && lookup(v, "password", null) == null
  }

  length  = 32
  special = true
}

# パスワードをSecret Managerに保存
resource "google_secret_manager_secret" "db_password" {
  for_each = {
    for k, v in var.users : k => v
    if lookup(v, "type", "BUILT_IN") == "BUILT_IN"
  }

  secret_id = "${var.instance_name}-${each.key}-password"
  project   = var.project_id

  replication {
    auto {}
  }

  labels = var.labels
}

resource "google_secret_manager_secret_version" "db_password" {
  for_each = {
    for k, v in var.users : k => v
    if lookup(v, "type", "BUILT_IN") == "BUILT_IN"
  }

  secret      = google_secret_manager_secret.db_password[each.key].id
  secret_data = google_sql_user.users[each.key].password
}
```

```hcl
# modules/data/cloud-sql/replicas.tf

# リードレプリカ
resource "google_sql_database_instance" "read_replicas" {
  for_each = var.read_replicas

  name                 = "${var.instance_name}-replica-${each.key}"
  project              = var.project_id
  region               = lookup(each.value, "region", var.region)
  database_version     = var.database_version
  master_instance_name = google_sql_database_instance.main.name

  replica_configuration {
    failover_target = lookup(each.value, "failover_target", false)
  }

  settings {
    tier              = lookup(each.value, "tier", var.tier)
    availability_type = "ZONAL"
    disk_size         = var.disk_size
    disk_type         = var.disk_type
    disk_autoresize   = var.disk_autoresize

    ip_configuration {
      ipv4_enabled    = false
      private_network = var.network_id
    }

    insights_config {
      query_insights_enabled  = true
      query_string_length     = 1024
      record_application_tags = true
      record_client_address   = true
    }

    user_labels = merge(var.labels, {
      environment = var.environment
      role        = "replica"
    })
  }

  lifecycle {
    prevent_destroy = true
  }
}
```

```hcl
# modules/data/cloud-sql/outputs.tf

output "instance_name" {
  description = "Cloud SQLインスタンス名"
  value       = google_sql_database_instance.main.name
}

output "instance_connection_name" {
  description = "Cloud SQL接続名"
  value       = google_sql_database_instance.main.connection_name
}

output "private_ip_address" {
  description = "プライベートIPアドレス"
  value       = google_sql_database_instance.main.private_ip_address
}

output "databases" {
  description = "データベースリスト"
  value       = [for db in google_sql_database.databases : db.name]
}

output "users" {
  description = "ユーザーリスト"
  value       = [for user in google_sql_user.users : user.name]
  sensitive   = true
}

output "password_secret_ids" {
  description = "パスワードSecret Manager ID"
  value = {
    for k, v in google_secret_manager_secret.db_password : k => v.id
  }
}

output "read_replica_connection_names" {
  description = "リードレプリカ接続名"
  value = {
    for k, v in google_sql_database_instance.read_replicas : k => v.connection_name
  }
}
```

### 3.4 CDN/エッジモジュール

#### 3.4.1 Cloud CDNモジュール

```hcl
# modules/networking/cdn/main.tf

# Cloud Storage バケット（静的アセット用）
resource "google_storage_bucket" "static_assets" {
  name                        = var.bucket_name
  project                     = var.project_id
  location                    = var.location
  force_destroy               = false
  uniform_bucket_level_access = true

  versioning {
    enabled = true
  }

  lifecycle_rule {
    condition {
      age = 90
    }
    action {
      type = "SetStorageClass"
      storage_class = "NEARLINE"
    }
  }

  cors {
    origin          = var.cors_origins
    method          = ["GET", "HEAD", "OPTIONS"]
    response_header = ["*"]
    max_age_seconds = 3600
  }

  labels = var.labels
}

# Backend Bucket（CDN用）
resource "google_compute_backend_bucket" "static_assets" {
  name        = "${var.bucket_name}-backend"
  project     = var.project_id
  bucket_name = google_storage_bucket.static_assets.name
  enable_cdn  = true

  cdn_policy {
    cache_mode        = var.cache_mode
    default_ttl       = var.default_ttl
    max_ttl           = var.max_ttl
    client_ttl        = var.client_ttl
    negative_caching  = true
    serve_while_stale = var.serve_while_stale

    cache_key_policy {
      include_http_headers = var.cache_key_headers
    }
  }

  compression_mode = "AUTOMATIC"

  custom_response_headers = [
    "X-Cache-Status: {cdn_cache_status}",
    "X-Cache-ID: {cdn_cache_id}"
  ]
}

# URL Map
resource "google_compute_url_map" "cdn" {
  name            = "${var.name}-url-map"
  project         = var.project_id
  default_service = google_compute_backend_bucket.static_assets.id

  host_rule {
    hosts        = var.hosts
    path_matcher = "assets"
  }

  path_matcher {
    name            = "assets"
    default_service = google_compute_backend_bucket.static_assets.id

    dynamic "path_rule" {
      for_each = var.path_rules
      content {
        paths   = path_rule.value.paths
        service = path_rule.value.service
      }
    }
  }
}

# HTTPS プロキシ
resource "google_compute_target_https_proxy" "cdn" {
  name    = "${var.name}-https-proxy"
  project = var.project_id
  url_map = google_compute_url_map.cdn.id

  ssl_certificates = [var.ssl_certificate_id]
  ssl_policy       = var.ssl_policy_id
}

# グローバル転送ルール
resource "google_compute_global_forwarding_rule" "cdn" {
  name                  = "${var.name}-forwarding-rule"
  project               = var.project_id
  target                = google_compute_target_https_proxy.cdn.id
  port_range            = "443"
  ip_address            = var.ip_address
  load_balancing_scheme = "EXTERNAL_MANAGED"
}
```

---

## 第4章 モジュール開発標準

### 4.1 変数と出力の設計パターン

#### 4.1.1 変数設計ガイドライン

```hcl
# 変数設計のベストプラクティス

# 1. 必須変数は明確に
variable "project_id" {
  description = "GCPプロジェクトID（必須）"
  type        = string
  # default なし = 必須

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{4,28}[a-z0-9]$", var.project_id))
    error_message = "project_id は6-30文字の小文字英数字とハイフンである必要があります。"
  }
}

# 2. オプション変数には適切なデフォルト
variable "region" {
  description = "リージョン（デフォルト: asia-northeast1）"
  type        = string
  default     = "asia-northeast1"

  validation {
    condition     = can(regex("^[a-z]+-[a-z]+[0-9]$", var.region))
    error_message = "region は有効なGCPリージョン形式である必要があります。"
  }
}

# 3. 複雑なオブジェクトはoptionalを活用
variable "node_pool_config" {
  description = "ノードプール設定"
  type = object({
    machine_type = string
    min_nodes    = optional(number, 1)
    max_nodes    = optional(number, 10)
    disk_size_gb = optional(number, 100)
    preemptible  = optional(bool, false)
    labels       = optional(map(string), {})
  })
}

# 4. センシティブな変数は明示
variable "database_password" {
  description = "データベースパスワード"
  type        = string
  sensitive   = true
}

# 5. 列挙型はvalidationで制約
variable "environment" {
  description = "環境名"
  type        = string

  validation {
    condition     = contains(["production", "staging", "development"], var.environment)
    error_message = "environment は production, staging, development のいずれかである必要があります。"
  }
}

# 6. 依存関係のある変数はpreconditionでチェック
variable "enable_backup" {
  description = "バックアップを有効にするか"
  type        = bool
  default     = true
}

variable "backup_location" {
  description = "バックアップ保存場所"
  type        = string
  default     = null
}

resource "google_sql_database_instance" "main" {
  # ...

  lifecycle {
    precondition {
      condition     = !var.enable_backup || var.backup_location != null
      error_message = "enable_backup が true の場合、backup_location は必須です。"
    }
  }
}
```

### 4.2 リソース命名規則

#### 4.2.1 命名規則標準

```hcl
# modules/common/naming/main.tf

locals {
  # 基本命名パターン
  # {project}-{environment}-{component}-{resource_type}-{suffix}

  name_prefix = "${var.project}-${var.environment}"

  # リソースタイプ別の命名
  resource_names = {
    # ネットワーク
    vpc              = "${local.name_prefix}-vpc"
    subnet           = "${local.name_prefix}-${var.component}-subnet"
    firewall         = "${local.name_prefix}-${var.component}-fw"
    router           = "${local.name_prefix}-router"
    nat              = "${local.name_prefix}-nat"

    # コンピュート
    gke_cluster      = "${local.name_prefix}-gke"
    node_pool        = "${local.name_prefix}-${var.component}-pool"
    instance         = "${local.name_prefix}-${var.component}-vm"
    instance_group   = "${local.name_prefix}-${var.component}-ig"

    # データ
    cloud_sql        = "${local.name_prefix}-sql"
    redis            = "${local.name_prefix}-redis"
    storage_bucket   = "${var.project}-${var.environment}-${var.component}"

    # ロードバランサ
    backend_service  = "${local.name_prefix}-${var.component}-bes"
    url_map          = "${local.name_prefix}-${var.component}-urlmap"
    forwarding_rule  = "${local.name_prefix}-${var.component}-fr"

    # セキュリティ
    service_account  = "${local.name_prefix}-${var.component}-sa"
    kms_keyring      = "${local.name_prefix}-keyring"
    kms_key          = "${local.name_prefix}-${var.component}-key"
    secret           = "${local.name_prefix}-${var.component}-secret"
  }
}

output "names" {
  description = "リソース名マップ"
  value       = local.resource_names
}
```

#### 4.2.2 命名規則ドキュメント

```yaml
naming_conventions:
  pattern: "{project}-{environment}-{component}-{resource_type}[-{suffix}]"

  components:
    project:
      value: "triptrip"
      description: "プロジェクト識別子"

    environment:
      values:
        - prod: "本番環境"
        - stg: "ステージング環境"
        - dev: "開発環境"

    component:
      examples:
        - api: "APIサービス"
        - web: "Webフロントエンド"
        - worker: "バックグラウンドワーカー"
        - data: "データ処理"

    resource_type:
      examples:
        - vpc: "VPCネットワーク"
        - subnet: "サブネット"
        - gke: "GKEクラスタ"
        - sql: "Cloud SQL"
        - sa: "サービスアカウント"

    suffix:
      usage: "オプション、一意性確保が必要な場合"
      examples:
        - "001", "primary", "east"

  constraints:
    max_length:
      gcs_bucket: 63
      gke_cluster: 40
      service_account: 30
    allowed_characters: "小文字英数字とハイフン"
    start_with: "英字"
    end_with: "英数字"

  examples:
    - "triptrip-prod-vpc"
    - "triptrip-prod-api-subnet"
    - "triptrip-prod-gke"
    - "triptrip-prod-api-pool"
    - "triptrip-prod-sql"
    - "triptrip-prod-api-sa"
```

### 4.3 タグ付け戦略

#### 4.3.1 必須ラベル定義

```hcl
# modules/common/labels/main.tf

variable "environment" {
  description = "環境名"
  type        = string
}

variable "project_name" {
  description = "プロジェクト名"
  type        = string
  default     = "triptrip"
}

variable "team" {
  description = "チーム名"
  type        = string
}

variable "cost_center" {
  description = "コストセンター"
  type        = string
}

variable "additional_labels" {
  description = "追加ラベル"
  type        = map(string)
  default     = {}
}

locals {
  # 必須ラベル
  required_labels = {
    environment  = var.environment
    project      = var.project_name
    team         = var.team
    cost_center  = var.cost_center
    managed_by   = "terraform"
    created_date = formatdate("YYYY-MM-DD", timestamp())
  }

  # 全ラベル
  all_labels = merge(local.required_labels, var.additional_labels)
}

output "labels" {
  description = "リソースラベル"
  value       = local.all_labels
}

# ラベル検証
resource "null_resource" "validate_labels" {
  lifecycle {
    precondition {
      condition     = length(var.team) > 0
      error_message = "team ラベルは必須です。"
    }

    precondition {
      condition     = length(var.cost_center) > 0
      error_message = "cost_center ラベルは必須です。"
    }
  }
}
```

#### 4.3.2 ラベル標準ドキュメント

```yaml
labeling_standards:
  required_labels:
    environment:
      description: "デプロイ環境"
      values: ["production", "staging", "development"]
      example: "production"

    project:
      description: "プロジェクト名"
      example: "triptrip"

    team:
      description: "所有チーム"
      values: ["platform", "backend", "frontend", "data", "security"]
      example: "platform"

    cost_center:
      description: "コストセンター/部門コード"
      example: "eng-001"

    managed_by:
      description: "管理ツール"
      values: ["terraform", "manual", "pulumi"]
      example: "terraform"

  recommended_labels:
    component:
      description: "コンポーネント名"
      example: "api-gateway"

    version:
      description: "バージョン"
      example: "1.2.3"

    owner:
      description: "オーナー（メールアドレス）"
      example: "team-platform@triptrip.com"

    criticality:
      description: "重要度"
      values: ["critical", "high", "medium", "low"]
      example: "high"

    data_classification:
      description: "データ分類"
      values: ["public", "internal", "confidential", "restricted"]
      example: "confidential"

  usage:
    cost_allocation:
      description: "コスト配分レポートで使用"
      labels: ["cost_center", "team", "project"]

    access_control:
      description: "IAM条件で使用"
      labels: ["environment", "data_classification"]

    automation:
      description: "自動化スクリプトで使用"
      labels: ["managed_by", "environment"]
```

---

## 第5章 ワークフローと自動化

### 5.1 PR時のplan実行

#### 5.1.1 Terraform PR ワークフロー

```yaml
# .github/workflows/terraform-pr.yaml
name: Terraform PR Check

on:
  pull_request:
    branches: [main]
    paths:
      - 'environments/**'
      - 'modules/**'
      - '.github/workflows/terraform-*.yaml'

permissions:
  contents: read
  pull-requests: write
  id-token: write

env:
  TF_VERSION: "1.6.0"
  TFLINT_VERSION: "0.50.0"

jobs:
  detect-changes:
    name: Detect Changes
    runs-on: ubuntu-latest
    outputs:
      environments: ${{ steps.changes.outputs.environments }}
    steps:
      - uses: actions/checkout@v4

      - name: Detect changed environments
        id: changes
        uses: dorny/paths-filter@v2
        with:
          filters: |
            production:
              - 'environments/production/**'
              - 'modules/**'
            staging:
              - 'environments/staging/**'
              - 'modules/**'
            development:
              - 'environments/development/**'
              - 'modules/**'

      - name: Set environments matrix
        id: set-matrix
        run: |
          ENVS="[]"
          if [ "${{ steps.changes.outputs.production }}" == "true" ]; then
            ENVS=$(echo $ENVS | jq '. + ["production"]')
          fi
          if [ "${{ steps.changes.outputs.staging }}" == "true" ]; then
            ENVS=$(echo $ENVS | jq '. + ["staging"]')
          fi
          if [ "${{ steps.changes.outputs.development }}" == "true" ]; then
            ENVS=$(echo $ENVS | jq '. + ["development"]')
          fi
          echo "environments=$ENVS" >> $GITHUB_OUTPUT

  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Setup TFLint
        uses: terraform-linters/setup-tflint@v4
        with:
          tflint_version: ${{ env.TFLINT_VERSION }}

      - name: Init TFLint
        run: tflint --init

      - name: Run TFLint
        run: |
          find . -name "*.tf" -exec dirname {} \; | sort -u | while read dir; do
            echo "Linting $dir"
            tflint --chdir=$dir
          done

  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: .
          framework: terraform
          soft_fail: true
          output_format: sarif
          output_file_path: checkov-results.sarif

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: checkov-results.sarif

      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          soft_fail: true

  plan:
    name: Plan (${{ matrix.environment }})
    runs-on: ubuntu-latest
    needs: [detect-changes, lint, security-scan]
    if: needs.detect-changes.outputs.environments != '[]'
    strategy:
      fail-fast: false
      matrix:
        environment: ${{ fromJson(needs.detect-changes.outputs.environments) }}
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ vars.GCP_TERRAFORM_SA }}

      - name: Terraform Init
        working-directory: environments/${{ matrix.environment }}
        run: terraform init -input=false

      - name: Terraform Validate
        working-directory: environments/${{ matrix.environment }}
        run: terraform validate

      - name: Terraform Plan
        id: plan
        working-directory: environments/${{ matrix.environment }}
        run: |
          terraform plan -input=false -no-color -out=tfplan 2>&1 | tee plan-output.txt
          echo "plan_output<<EOF" >> $GITHUB_OUTPUT
          cat plan-output.txt >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
        continue-on-error: true

      - name: Post Plan to PR
        uses: actions/github-script@v7
        with:
          script: |
            const output = `### Terraform Plan - ${{ matrix.environment }}

            #### Terraform Initialization ⚙️ \`Success\`
            #### Terraform Validation 🤖 \`Success\`
            #### Terraform Plan 📖 \`${{ steps.plan.outcome }}\`

            <details>
            <summary>Show Plan</summary>

            \`\`\`terraform
            ${{ steps.plan.outputs.plan_output }}
            \`\`\`

            </details>

            *Pushed by: @${{ github.actor }}, Action: \`${{ github.event_name }}\`*`;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });

      - name: Upload Plan
        uses: actions/upload-artifact@v4
        with:
          name: tfplan-${{ matrix.environment }}
          path: environments/${{ matrix.environment }}/tfplan
          retention-days: 7

  cost-estimate:
    name: Cost Estimate (${{ matrix.environment }})
    runs-on: ubuntu-latest
    needs: [plan]
    if: needs.detect-changes.outputs.environments != '[]'
    strategy:
      fail-fast: false
      matrix:
        environment: ${{ fromJson(needs.detect-changes.outputs.environments) }}
    steps:
      - uses: actions/checkout@v4

      - name: Download Plan
        uses: actions/download-artifact@v4
        with:
          name: tfplan-${{ matrix.environment }}
          path: environments/${{ matrix.environment }}

      - name: Setup Infracost
        uses: infracost/actions/setup@v3
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}

      - name: Run Infracost
        run: |
          infracost breakdown \
            --path environments/${{ matrix.environment }}/tfplan \
            --format json \
            --out-file /tmp/infracost.json

      - name: Post Infracost Comment
        uses: infracost/actions/comment@v1
        with:
          path: /tmp/infracost.json
          behavior: update
```

### 5.2 マージ時の自動apply

#### 5.2.1 Terraform Apply ワークフロー

```yaml
# .github/workflows/terraform-apply.yaml
name: Terraform Apply

on:
  push:
    branches: [main]
    paths:
      - 'environments/**'
      - 'modules/**'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to apply'
        required: true
        type: choice
        options:
          - development
          - staging
          - production

permissions:
  contents: read
  id-token: write

env:
  TF_VERSION: "1.6.0"

jobs:
  detect-changes:
    name: Detect Changes
    runs-on: ubuntu-latest
    outputs:
      environments: ${{ steps.changes.outputs.environments }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Get changed files
        id: changed-files
        uses: tj-actions/changed-files@v42

      - name: Set environments
        id: changes
        run: |
          if [ "${{ github.event_name }}" == "workflow_dispatch" ]; then
            echo 'environments=["${{ github.event.inputs.environment }}"]' >> $GITHUB_OUTPUT
          else
            ENVS="[]"
            for file in ${{ steps.changed-files.outputs.all_changed_files }}; do
              if [[ $file == environments/production/* ]] || [[ $file == modules/* ]]; then
                ENVS=$(echo $ENVS | jq 'if . | index("production") then . else . + ["production"] end')
              fi
              if [[ $file == environments/staging/* ]] || [[ $file == modules/* ]]; then
                ENVS=$(echo $ENVS | jq 'if . | index("staging") then . else . + ["staging"] end')
              fi
              if [[ $file == environments/development/* ]] || [[ $file == modules/* ]]; then
                ENVS=$(echo $ENVS | jq 'if . | index("development") then . else . + ["development"] end')
              fi
            done
            echo "environments=$ENVS" >> $GITHUB_OUTPUT
          fi

  apply-development:
    name: Apply Development
    runs-on: ubuntu-latest
    needs: detect-changes
    if: contains(fromJson(needs.detect-changes.outputs.environments), 'development')
    environment: development
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ vars.GCP_TERRAFORM_SA }}

      - name: Terraform Init
        working-directory: environments/development
        run: terraform init -input=false

      - name: Terraform Apply
        working-directory: environments/development
        run: terraform apply -auto-approve -input=false

  apply-staging:
    name: Apply Staging
    runs-on: ubuntu-latest
    needs: [detect-changes, apply-development]
    if: |
      always() &&
      contains(fromJson(needs.detect-changes.outputs.environments), 'staging') &&
      (needs.apply-development.result == 'success' || needs.apply-development.result == 'skipped')
    environment: staging
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ vars.GCP_TERRAFORM_SA }}

      - name: Terraform Init
        working-directory: environments/staging
        run: terraform init -input=false

      - name: Terraform Apply
        working-directory: environments/staging
        run: terraform apply -auto-approve -input=false

  apply-production:
    name: Apply Production
    runs-on: ubuntu-latest
    needs: [detect-changes, apply-staging]
    if: |
      always() &&
      contains(fromJson(needs.detect-changes.outputs.environments), 'production') &&
      (needs.apply-staging.result == 'success' || needs.apply-staging.result == 'skipped')
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ vars.GCP_TERRAFORM_SA }}

      - name: Terraform Init
        working-directory: environments/production
        run: terraform init -input=false

      - name: Terraform Apply
        working-directory: environments/production
        run: terraform apply -auto-approve -input=false

      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: '#infrastructure'
          slack-message: |
            :terraform: Terraform Apply (Production)
            Status: ${{ job.status }}
            Actor: ${{ github.actor }}
            Commit: ${{ github.sha }}
```

### 5.3 定期的なdrift detection

#### 5.3.1 Drift Detection ワークフロー

```yaml
# .github/workflows/terraform-drift.yaml
name: Terraform Drift Detection

on:
  schedule:
    - cron: '0 3 * * *'  # 毎日 AM 3:00 UTC (JST 12:00)
  workflow_dispatch:

permissions:
  contents: read
  issues: write
  id-token: write

env:
  TF_VERSION: "1.6.0"

jobs:
  drift-detection:
    name: Drift Detection (${{ matrix.environment }})
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        environment: [development, staging, production]
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ vars.GCP_TERRAFORM_SA }}

      - name: Terraform Init
        working-directory: environments/${{ matrix.environment }}
        run: terraform init -input=false

      - name: Terraform Plan (Drift Detection)
        id: plan
        working-directory: environments/${{ matrix.environment }}
        run: |
          terraform plan -detailed-exitcode -input=false -no-color > plan-output.txt 2>&1 || exit_code=$?

          echo "exit_code=${exit_code:-0}" >> $GITHUB_OUTPUT

          if [ "${exit_code:-0}" -eq 2 ]; then
            echo "drift_detected=true" >> $GITHUB_OUTPUT
            echo "Drift detected!"
          else
            echo "drift_detected=false" >> $GITHUB_OUTPUT
            echo "No drift detected"
          fi

          cat plan-output.txt
        continue-on-error: true

      - name: Create Issue for Drift
        if: steps.plan.outputs.drift_detected == 'true'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const planOutput = fs.readFileSync('environments/${{ matrix.environment }}/plan-output.txt', 'utf8');

            const issue = await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `🔄 Infrastructure Drift Detected: ${{ matrix.environment }}`,
              body: `## Infrastructure Drift Detected

            **Environment:** ${{ matrix.environment }}
            **Detection Time:** ${new Date().toISOString()}
            **Workflow Run:** [Link](https://github.com/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId})

            ### Terraform Plan Output

            \`\`\`
            ${planOutput.substring(0, 60000)}
            \`\`\`

            ### Action Required

            Please review the drift and take appropriate action:
            1. If the drift is intentional, update the Terraform configuration
            2. If the drift is unintentional, run \`terraform apply\` to reconcile

            /cc @platform-team
            `,
              labels: ['infrastructure', 'drift', '${{ matrix.environment }}']
            });

            console.log(`Created issue #${issue.data.number}`);

      - name: Notify Slack (Drift Detected)
        if: steps.plan.outputs.drift_detected == 'true'
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: '#infrastructure-alerts'
          slack-message: |
            :warning: *Infrastructure Drift Detected*
            Environment: `${{ matrix.environment }}`
            Action Required: Please review and reconcile
            <https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Details>

  summary:
    name: Drift Summary
    runs-on: ubuntu-latest
    needs: drift-detection
    if: always()
    steps:
      - name: Post Summary
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: '#infrastructure'
          slack-message: |
            :mag: *Daily Drift Detection Complete*

            | Environment | Status |
            |-------------|--------|
            | Development | ${{ needs.drift-detection.result }} |
            | Staging | ${{ needs.drift-detection.result }} |
            | Production | ${{ needs.drift-detection.result }} |

            <https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Full Report>
```

---

## 第6章 セキュリティとガバナンス

### 6.1 ポリシーコード化

#### 6.1.1 OPA (Open Policy Agent) ポリシー

```rego
# policies/opa/encryption.rego
package terraform.encryption

# Cloud SQLは暗号化が必要
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "google_sql_database_instance"
    not resource.change.after.settings[0].ip_configuration[0].require_ssl

    msg := sprintf(
        "Cloud SQL instance '%s' must have SSL required",
        [resource.address]
    )
}

# GCS Bucketは暗号化が必要
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "google_storage_bucket"
    not resource.change.after.encryption

    msg := sprintf(
        "Storage bucket '%s' must have encryption configured",
        [resource.address]
    )
}

# Compute Diskは暗号化が必要
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "google_compute_disk"
    not resource.change.after.disk_encryption_key

    msg := sprintf(
        "Compute disk '%s' should use customer-managed encryption key",
        [resource.address]
    )
}
```

```rego
# policies/opa/networking.rego
package terraform.networking

# 0.0.0.0/0 からのインバウンドは禁止
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "google_compute_firewall"
    resource.change.after.direction == "INGRESS"
    source_range := resource.change.after.source_ranges[_]
    source_range == "0.0.0.0/0"

    msg := sprintf(
        "Firewall rule '%s' should not allow ingress from 0.0.0.0/0",
        [resource.address]
    )
}

# プライベートIPのみを許可
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "google_sql_database_instance"
    resource.change.after.settings[0].ip_configuration[0].ipv4_enabled == true

    msg := sprintf(
        "Cloud SQL '%s' should not have public IP enabled",
        [resource.address]
    )
}
```

```rego
# policies/opa/tagging.rego
package terraform.tagging

# 必須ラベルの確認
required_labels := {"environment", "team", "cost_center", "managed_by"}

deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "google_storage_bucket"
    labels := object.get(resource.change.after, "labels", {})
    missing := required_labels - {l | labels[l]}
    count(missing) > 0

    msg := sprintf(
        "Resource '%s' is missing required labels: %v",
        [resource.address, missing]
    )
}

deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "google_container_cluster"
    labels := object.get(resource.change.after, "resource_labels", {})
    missing := required_labels - {l | labels[l]}
    count(missing) > 0

    msg := sprintf(
        "Resource '%s' is missing required labels: %v",
        [resource.address, missing]
    )
}
```

### 6.2 コスト管理

#### 6.2.1 Infracost統合

```yaml
# .github/workflows/infracost.yaml
name: Infracost

on:
  pull_request:
    paths:
      - 'environments/**'
      - 'modules/**'

permissions:
  contents: read
  pull-requests: write

jobs:
  infracost:
    name: Infracost
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Infracost
        uses: infracost/actions/setup@v3
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}

      - name: Checkout base branch
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.base.ref }}
          path: base

      - name: Generate Infracost baseline
        run: |
          infracost breakdown --path base/environments \
            --format json \
            --out-file /tmp/infracost-base.json

      - name: Generate Infracost diff
        run: |
          infracost diff --path environments \
            --compare-to /tmp/infracost-base.json \
            --format json \
            --out-file /tmp/infracost.json

      - name: Post Infracost comment
        uses: infracost/actions/comment@v1
        with:
          path: /tmp/infracost.json
          behavior: update

      - name: Check cost increase threshold
        run: |
          MONTHLY_DIFF=$(jq '.totalMonthlyCost' /tmp/infracost.json)
          PERCENT_DIFF=$(jq '.diffTotalMonthlyCost' /tmp/infracost.json)

          if [ $(echo "$PERCENT_DIFF > 10" | bc) -eq 1 ]; then
            echo "::warning::Cost increase is more than 10%"
          fi

          if [ $(echo "$MONTHLY_DIFF > 1000" | bc) -eq 1 ]; then
            echo "::error::Monthly cost increase exceeds $1000. Please review."
            exit 1
          fi
```

### 6.3 監査とコンプライアンス

#### 6.3.1 監査ログ設定

```hcl
# modules/security/audit-logging/main.tf

# 組織レベルの監査ログシンク
resource "google_logging_organization_sink" "audit_sink" {
  name             = "audit-log-sink"
  org_id           = var.org_id
  destination      = "storage.googleapis.com/${google_storage_bucket.audit_logs.name}"
  include_children = true

  filter = <<-EOT
    logName:"cloudaudit.googleapis.com"
    OR logName:"activity"
    OR logName:"data_access"
  EOT
}

# 監査ログ保存バケット
resource "google_storage_bucket" "audit_logs" {
  name                        = "${var.project_id}-audit-logs"
  project                     = var.project_id
  location                    = var.location
  force_destroy               = false
  uniform_bucket_level_access = true

  versioning {
    enabled = true
  }

  lifecycle_rule {
    condition {
      age = 365
    }
    action {
      type          = "SetStorageClass"
      storage_class = "COLDLINE"
    }
  }

  lifecycle_rule {
    condition {
      age = 2555  # 7年
    }
    action {
      type = "Delete"
    }
  }

  # WORM (Write Once Read Many) ポリシー
  retention_policy {
    is_locked        = true
    retention_period = 220752000  # 7年（秒）
  }

  labels = var.labels
}

# バケットIAM
resource "google_storage_bucket_iam_member" "audit_writer" {
  bucket = google_storage_bucket.audit_logs.name
  role   = "roles/storage.objectCreator"
  member = google_logging_organization_sink.audit_sink.writer_identity
}
```

---

## 第7章 実装ロードマップ

### 7.1 フェーズ別実装計画

```yaml
implementation_roadmap:
  phase_1_foundation:
    timeline: Month 1-2
    objectives:
      - Terraformリポジトリ構築
      - 基本モジュール作成
      - 状態管理設定
    deliverables:
      - VPCモジュール
      - GKEモジュール
      - Cloud SQLモジュール
      - リモートバックエンド設定
    success_criteria:
      - 開発環境のTerraform管理
      - CI/CDパイプライン稼働

  phase_2_migration:
    timeline: Month 3-4
    objectives:
      - 既存リソースのインポート
      - ステージング環境移行
      - ポリシー導入
    deliverables:
      - 既存リソースのTerraform化
      - OPA/Sentinelポリシー
      - Infracost統合
    success_criteria:
      - ステージング環境の完全Terraform管理
      - ポリシー検証の自動化

  phase_3_production:
    timeline: Month 5-6
    objectives:
      - 本番環境移行
      - ドリフト検出
      - ガバナンス強化
    deliverables:
      - 本番環境Terraform化
      - ドリフト検出ワークフロー
      - 監査ログ設定
    success_criteria:
      - 全環境のTerraform管理
      - コンプライアンス監査合格
```

---

## 第8章 文書間参照 & 統合ポイント

### 8.1 関連文書

```yaml
document_references:
  upstream:
    - id: Doc-DM-002
      title: DevOps＆継続的デリバリー戦略
      relationship: "IaC概念とベストプラクティス"

    - id: Doc-IA-001
      title: Cloud Infrastructure & Deployment
      relationship: "クラウドインフラ設計"

  peer:
    - id: Doc-DO-001
      title: CI/CD Pipeline Architecture
      relationship: "パイプライン統合"

    - id: Doc-IA-002
      title: Kubernetes & Container Strategy
      relationship: "GKEモジュール連携"

  downstream:
    - id: Doc-SC-001
      title: Security Architecture
      relationship: "セキュリティポリシー"

    - id: Doc-QA-003
      title: リリース管理＆ロールバック
      relationship: "インフラ変更管理"
```

### 8.2 技術スタックサマリー

```yaml
technology_stack:
  iac_tool:
    name: Terraform
    version: ">= 1.6.0"
    providers:
      - google: ">= 5.0.0"
      - google-beta: ">= 5.0.0"
      - random: ">= 3.0.0"
      - null: ">= 3.0.0"

  state_management:
    backend: GCS
    encryption: KMS (CMEK)
    locking: Object-level

  ci_cd:
    platform: GitHub Actions
    workflows:
      - terraform-pr.yaml
      - terraform-apply.yaml
      - terraform-drift.yaml

  security:
    policy_engine: OPA
    scanning: tfsec, Checkov
    cost_management: Infracost

  quality:
    linting: TFLint
    formatting: terraform fmt
    validation: terraform validate
```

---

## 付録A: Terraform バージョン管理

```hcl
# versions.tf（共通テンプレート）

terraform {
  required_version = ">= 1.6.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = ">= 5.0.0, < 6.0.0"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = ">= 5.0.0, < 6.0.0"
    }
    random = {
      source  = "hashicorp/random"
      version = ">= 3.0.0"
    }
    null = {
      source  = "hashicorp/null"
      version = ">= 3.0.0"
    }
  }
}
```

---

## 付録B: .tflint.hcl 設定

```hcl
# .tflint.hcl

config {
  call_module_type = "all"
}

plugin "google" {
  enabled = true
  version = "0.25.0"
  source  = "github.com/terraform-linters/tflint-ruleset-google"
}

rule "terraform_deprecated_index" {
  enabled = true
}

rule "terraform_unused_declarations" {
  enabled = true
}

rule "terraform_documented_outputs" {
  enabled = true
}

rule "terraform_documented_variables" {
  enabled = true
}

rule "terraform_typed_variables" {
  enabled = true
}

rule "terraform_naming_convention" {
  enabled = true
  format  = "snake_case"
}

rule "terraform_required_version" {
  enabled = true
}

rule "terraform_required_providers" {
  enabled = true
}
```

---

## 付録C: 変更履歴

| バージョン | 日付 | 変更内容 | 作成者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-01-20 | 初版作成 | Technical Architecture Agent |

---

**文書情報**
- 文書ID: Doc-DO-002
- バージョン: 1.0.0
- ステータス: ドラフト
- 最終更新: 2026-01-20
- 次回レビュー: 2026-02-20
- 行数: 約1,500行

# Doc-SC-003: TripTripデータ保護 & 暗号化

## エグゼクティブサマリー

本文書は、TripTripプラットフォームの包括的なデータ保護と暗号化戦略を定義します。保存時暗号化（Encryption at Rest）、転送時暗号化（Encryption in Transit）、使用時暗号化（Encryption in Use）の三層暗号化アーキテクチャ、AWS KMSとHashiCorp Vaultを活用した鍵管理システム、PIIデータ保護（マスキング、トークナイゼーション、匿名化）、PCI-DSS準拠の決済データセキュリティを詳述します。Google、Amazon、Netflixレベルのデータ保護基準を採用し、グローバル規制に準拠した堅牢なデータセキュリティ基盤を構築します。本設計は、Doc-DA-005（データセキュリティ、プライバシー＆ガバナンス）と連携し、Doc-SC-001（セキュリティアーキテクチャ＆戦略）の多層防御原則を実装します。

---

## 第1章：はじめに & コンテキスト

### 1.1 目的 & スコープ

#### 1.1.1 文書の目的

```yaml
document_purpose:
  primary_objectives:
    - TripTripプラットフォーム全体のデータ保護アーキテクチャ定義
    - 三層暗号化戦略（保存時、転送時、使用時）の実装
    - 鍵管理システム（KMS）の設計
    - PIIデータ保護技術の実装
    - PCI-DSS準拠の決済データセキュリティ

  scope:
    in_scope:
      - データ暗号化アーキテクチャ
      - 鍵管理戦略
      - PIIデータ保護（マスキング、トークナイゼーション）
      - 決済データセキュリティ
      - バックアップ暗号化
      - データ分類と処理ポリシー

    out_of_scope:
      - 認証・認可（Doc-SC-002参照）
      - コンプライアンス詳細（Doc-SC-004参照）
      - 全体セキュリティ戦略（Doc-SC-001参照）
      - データガバナンス（Doc-DA-005参照）

  target_audience:
    - セキュリティアーキテクト
    - データエンジニア
    - 開発チーム
    - コンプライアンス担当者
```

#### 1.1.2 規制要件概要

```yaml
regulatory_requirements:
  gdpr:
    article_32:
      - 個人データの暗号化
      - データの復元能力
      - 定期的なセキュリティテスト
    article_25:
      - データ保護バイデザイン
      - データ最小化

  pci_dss:
    requirement_3:
      - カードホルダーデータの保護
      - 暗号化キーの管理
    requirement_4:
      - 転送時の暗号化

  japan_appi:
    - 個人データの安全管理措置
    - 技術的安全管理措置

  sox:
    - 財務データの完全性
    - アクセス制御
    - 監査証跡
```

### 1.2 データ保護目標 & 成功基準

#### 1.2.1 データ保護目標

```yaml
data_protection_objectives:
  confidentiality:
    description: 機密データの不正アクセス防止
    targets:
      - すべてのPIIの暗号化
      - 決済データの安全な処理
      - 機密ビジネスデータの保護
    metrics:
      - 暗号化カバレッジ: 100%
      - データ漏洩インシデント: 0件/年

  integrity:
    description: データの改ざん防止と検知
    targets:
      - データ完全性の検証
      - 改ざん検知メカニズム
      - 監査証跡の保護
    metrics:
      - データ改ざん検知率: 100%
      - チェックサム検証エラー: 0件

  availability:
    description: データの可用性確保
    targets:
      - 暗号化によるパフォーマンス影響の最小化
      - 鍵可用性の確保
      - 災害復旧時のデータ復元
    metrics:
      - 暗号化オーバーヘッド: <5%
      - 鍵可用性: 99.999%
```

#### 1.2.2 成功基準（KPI）

```yaml
data_protection_kpis:
  encryption_coverage:
    at_rest: 100%
    in_transit: 100%
    database_fields: 100% (PII/機密)

  key_management:
    key_rotation_compliance: 100%
    key_access_audit: 100%
    emergency_recovery_time: <1時間

  compliance:
    pci_dss_compliance: 100%
    gdpr_compliance: 100%
    audit_findings: 0件（Critical/High）

  performance:
    encryption_latency_overhead: <5ms
    decryption_latency_overhead: <3ms
    throughput_impact: <5%
```

---

## 第2章：暗号化アーキテクチャ

### 2.1 保存時暗号化（Encryption at Rest）

#### 2.1.1 保存時暗号化アーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    ENCRYPTION AT REST ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      AWS KMS (Key Management Service)                    │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │    │
│  │   │ CMK: Master  │  │ CMK: Data    │  │ CMK: Backup  │                 │    │
│  │   │ Key          │  │ Keys         │  │ Keys         │                 │    │
│  │   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                 │    │
│  │          │                 │                 │                          │    │
│  └──────────┼─────────────────┼─────────────────┼──────────────────────────┘    │
│             │                 │                 │                               │
│             ▼                 ▼                 ▼                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      Data Encryption Keys (DEK)                          │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │    │
│  │   │ RDS DEK      │  │ S3 DEK       │  │ EBS DEK      │  │ Redis DEK │  │    │
│  │   │              │  │              │  │              │  │           │  │    │
│  │   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └─────┬─────┘  │    │
│  │          │                 │                 │                │        │    │
│  └──────────┼─────────────────┼─────────────────┼────────────────┼────────┘    │
│             │                 │                 │                │             │
│             ▼                 ▼                 ▼                ▼             │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      Encrypted Data Stores                               │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │    │
│  │   │ PostgreSQL   │  │ S3 Buckets   │  │ EBS Volumes  │  │ Redis     │  │    │
│  │   │ (RDS)        │  │              │  │              │  │ Cluster   │  │    │
│  │   │              │  │              │  │              │  │           │  │    │
│  │   │ AES-256      │  │ SSE-KMS      │  │ AES-256      │  │ AES-256   │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────┘  └───────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      Application-Level Encryption                        │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │    │
│  │   │ PII Fields   │  │ Secrets      │  │ Tokens       │                 │    │
│  │   │ (AES-256-GCM)│  │ (Vault)      │  │ (Encrypted)  │                 │    │
│  │   └──────────────┘  └──────────────┘  └──────────────┘                 │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 2.1.2 データベース暗号化

```yaml
database_encryption:
  postgresql_rds:
    transparent_data_encryption:
      enabled: true
      algorithm: AES-256
      key_management: AWS KMS
      key_rotation: 自動（年次）

    configuration:
      storage_encrypted: true
      kms_key_id: "arn:aws:kms:ap-northeast-1:xxx:key/xxx-db-key"
      performance_insights_enabled: true
      performance_insights_kms_key_id: "arn:aws:kms:ap-northeast-1:xxx:key/xxx-pi-key"

    column_level_encryption:
      enabled: true
      fields:
        - name: users.email
          method: deterministic
          purpose: 検索可能な暗号化
        - name: users.phone_number
          method: randomized
          purpose: 最大セキュリティ
        - name: users.address
          method: randomized
        - name: payment_methods.card_last_four
          method: deterministic
        - name: audit_logs.user_context
          method: randomized

  dynamodb:
    encryption:
      enabled: true
      type: KMS
      kms_key_arn: "arn:aws:kms:ap-northeast-1:xxx:key/xxx-dynamo-key"

  elasticache_redis:
    encryption:
      at_rest: true
      kms_key_id: "arn:aws:kms:ap-northeast-1:xxx:key/xxx-redis-key"
```

#### 2.1.3 ストレージ暗号化

```yaml
storage_encryption:
  s3:
    default_encryption:
      sse_algorithm: "aws:kms"
      kms_master_key_id: "arn:aws:kms:ap-northeast-1:xxx:key/xxx-s3-key"

    bucket_policies:
      triptrip-user-uploads:
        encryption: SSE-KMS
        key: user-uploads-key
        classification: CONFIDENTIAL

      triptrip-backups:
        encryption: SSE-KMS
        key: backup-key
        classification: RESTRICTED

      triptrip-logs:
        encryption: SSE-S3
        classification: INTERNAL

      triptrip-public-assets:
        encryption: SSE-S3
        classification: PUBLIC

    object_level_encryption:
      sensitive_files:
        additional_encryption: Client-side encryption
        algorithm: AES-256-GCM

  ebs:
    default_encryption:
      enabled: true
      kms_key_id: "arn:aws:kms:ap-northeast-1:xxx:key/xxx-ebs-key"

    volume_types:
      gp3:
        encryption: true
      io2:
        encryption: true

  efs:
    encryption:
      at_rest: true
      kms_key_id: "arn:aws:kms:ap-northeast-1:xxx:key/xxx-efs-key"
```

### 2.2 転送時暗号化（Encryption in Transit）

#### 2.2.1 TLS設定

```yaml
tls_configuration:
  minimum_version: TLS 1.3

  cipher_suites:
    tls_1_3:
      - TLS_AES_256_GCM_SHA384
      - TLS_AES_128_GCM_SHA256
      - TLS_CHACHA20_POLY1305_SHA256

    tls_1_2_fallback:
      - ECDHE-ECDSA-AES256-GCM-SHA384
      - ECDHE-RSA-AES256-GCM-SHA384
      - ECDHE-ECDSA-AES128-GCM-SHA256
      - ECDHE-RSA-AES128-GCM-SHA256

  certificate_management:
    provider: AWS Certificate Manager (ACM)
    domains:
      - "*.triptrip.com"
      - "api.triptrip.com"
      - "auth.triptrip.com"
    renewal: 自動
    key_algorithm: RSA-2048 / ECDSA P-256

  hsts:
    enabled: true
    max_age: 63072000  # 2年
    include_subdomains: true
    preload: true

  certificate_pinning:
    mobile_app:
      enabled: true
      pins:
        - "sha256/xxxx..."  # Primary
        - "sha256/yyyy..."  # Backup
      backup_pins: 2
      expiry_alert: 30日前
```

#### 2.2.2 サービス間暗号化

```yaml
service_to_service_encryption:
  istio_mtls:
    mode: STRICT
    certificate_authority: Istio CA
    workload_cert_ttl: 24h
    root_cert_ttl: 10y

    peer_authentication:
      apiVersion: security.istio.io/v1beta1
      kind: PeerAuthentication
      metadata:
        name: default
        namespace: istio-system
      spec:
        mtls:
          mode: STRICT

  database_connections:
    postgresql:
      ssl_mode: verify-full
      ssl_cert: /etc/ssl/client-cert.pem
      ssl_key: /etc/ssl/client-key.pem
      ssl_ca: /etc/ssl/ca-cert.pem

    redis:
      tls_enabled: true
      tls_auth_clients: true

  external_api_calls:
    default: TLS 1.3
    certificate_validation: strict
    retry_on_ssl_error: false
```

#### 2.2.3 ALB/CloudFront TLS設定

```yaml
load_balancer_tls:
  alb:
    ssl_policy: "ELBSecurityPolicy-TLS13-1-2-2021-06"
    default_certificate: "arn:aws:acm:ap-northeast-1:xxx:certificate/xxx"
    additional_certificates:
      - "arn:aws:acm:ap-northeast-1:xxx:certificate/yyy"

    listeners:
      https:
        port: 443
        protocol: HTTPS
        ssl_policy: "ELBSecurityPolicy-TLS13-1-2-2021-06"

      http_redirect:
        port: 80
        protocol: HTTP
        action:
          type: redirect
          redirect:
            protocol: HTTPS
            port: "443"
            status_code: HTTP_301

  cloudfront:
    viewer_protocol_policy: redirect-to-https
    minimum_protocol_version: TLSv1.2_2021
    ssl_support_method: sni-only

    origin_protocol_policy: https-only
    origin_ssl_protocols:
      - TLSv1.2
```

### 2.3 使用時暗号化（Encryption in Use）

#### 2.3.1 アプリケーションレベル暗号化

```yaml
application_level_encryption:
  field_encryption:
    algorithm: AES-256-GCM
    key_derivation: HKDF-SHA256
    nonce_generation: 12 bytes random

    encrypted_fields:
      high_sensitivity:
        - users.ssn
        - users.passport_number
        - payment_methods.card_token
        method: application_encryption
        searchable: false

      medium_sensitivity:
        - users.email
        - users.phone
        - users.address
        method: deterministic_encryption
        searchable: partial

  envelope_encryption:
    description: データ暗号化鍵（DEK）をマスターキー（KEK）で暗号化
    implementation:
      1: アプリケーションがDEKを生成
      2: DEKでデータを暗号化
      3: KMSでDEKを暗号化
      4: 暗号化されたDEKとデータを保存

  memory_protection:
    sensitive_data:
      - パスワード
      - 暗号化キー
      - セッショントークン
    techniques:
      - メモリからの即座のクリア
      - セキュアメモリ割り当て
      - スワップ防止
```

#### 2.3.2 フィールドレベル暗号化実装

```typescript
// encryption/field-encryption.ts

import { createCipheriv, createDecipheriv, randomBytes, createHash } from 'crypto';
import { KMSClient, GenerateDataKeyCommand, DecryptCommand } from '@aws-sdk/client-kms';

interface EncryptedField {
  ciphertext: string;
  nonce: string;
  tag: string;
  keyId: string;
  encryptedDek: string;
}

interface EncryptionConfig {
  kmsKeyArn: string;
  context: Record<string, string>;
}

class FieldEncryption {
  private kmsClient: KMSClient;
  private config: EncryptionConfig;
  private dekCache: Map<string, { dek: Buffer; expiry: number }> = new Map();

  constructor(config: EncryptionConfig) {
    this.config = config;
    this.kmsClient = new KMSClient({ region: process.env.AWS_REGION });
  }

  // 暗号化
  async encrypt(plaintext: string, context?: Record<string, string>): Promise<EncryptedField> {
    // 1. データ暗号化キー（DEK）の生成または取得
    const { dek, encryptedDek, keyId } = await this.getOrGenerateDataKey(context);

    // 2. ノンス生成
    const nonce = randomBytes(12);

    // 3. AES-256-GCMで暗号化
    const cipher = createCipheriv('aes-256-gcm', dek, nonce);
    let ciphertext = cipher.update(plaintext, 'utf8', 'base64');
    ciphertext += cipher.final('base64');
    const tag = cipher.getAuthTag();

    return {
      ciphertext,
      nonce: nonce.toString('base64'),
      tag: tag.toString('base64'),
      keyId,
      encryptedDek: encryptedDek.toString('base64'),
    };
  }

  // 復号
  async decrypt(encrypted: EncryptedField, context?: Record<string, string>): Promise<string> {
    // 1. 暗号化されたDEKをKMSで復号
    const dek = await this.decryptDataKey(
      Buffer.from(encrypted.encryptedDek, 'base64'),
      context
    );

    // 2. AES-256-GCMで復号
    const decipher = createDecipheriv(
      'aes-256-gcm',
      dek,
      Buffer.from(encrypted.nonce, 'base64')
    );
    decipher.setAuthTag(Buffer.from(encrypted.tag, 'base64'));

    let plaintext = decipher.update(encrypted.ciphertext, 'base64', 'utf8');
    plaintext += decipher.final('utf8');

    return plaintext;
  }

  // 決定的暗号化（検索可能）
  async encryptDeterministic(plaintext: string, context?: Record<string, string>): Promise<string> {
    const { dek } = await this.getOrGenerateDataKey(context);

    // 入力からノンスを導出（決定的）
    const nonce = createHash('sha256')
      .update(plaintext)
      .update(this.config.kmsKeyArn)
      .digest()
      .slice(0, 12);

    const cipher = createCipheriv('aes-256-gcm', dek, nonce);
    let ciphertext = cipher.update(plaintext, 'utf8', 'base64');
    ciphertext += cipher.final('base64');
    const tag = cipher.getAuthTag();

    // 簡略化された形式（検索用）
    return `${ciphertext}:${tag.toString('base64')}`;
  }

  private async getOrGenerateDataKey(context?: Record<string, string>): Promise<{
    dek: Buffer;
    encryptedDek: Buffer;
    keyId: string;
  }> {
    const cacheKey = JSON.stringify(context || {});
    const cached = this.dekCache.get(cacheKey);

    if (cached && cached.expiry > Date.now()) {
      return {
        dek: cached.dek,
        encryptedDek: Buffer.alloc(0), // キャッシュ時は不要
        keyId: this.config.kmsKeyArn,
      };
    }

    // KMSでデータキー生成
    const command = new GenerateDataKeyCommand({
      KeyId: this.config.kmsKeyArn,
      KeySpec: 'AES_256',
      EncryptionContext: {
        ...this.config.context,
        ...context,
      },
    });

    const response = await this.kmsClient.send(command);

    const dek = Buffer.from(response.Plaintext!);
    const encryptedDek = Buffer.from(response.CiphertextBlob!);

    // キャッシュ（5分間）
    this.dekCache.set(cacheKey, {
      dek,
      expiry: Date.now() + 5 * 60 * 1000,
    });

    return { dek, encryptedDek, keyId: this.config.kmsKeyArn };
  }

  private async decryptDataKey(
    encryptedDek: Buffer,
    context?: Record<string, string>
  ): Promise<Buffer> {
    const command = new DecryptCommand({
      CiphertextBlob: encryptedDek,
      KeyId: this.config.kmsKeyArn,
      EncryptionContext: {
        ...this.config.context,
        ...context,
      },
    });

    const response = await this.kmsClient.send(command);
    return Buffer.from(response.Plaintext!);
  }

  // メモリクリア
  clearCache(): void {
    for (const [key, value] of this.dekCache.entries()) {
      value.dek.fill(0);
      this.dekCache.delete(key);
    }
  }
}

// Prismaミドルウェアとの統合
const prismaEncryptionMiddleware = (fieldEncryption: FieldEncryption) => {
  const encryptedFields = new Map<string, string[]>([
    ['User', ['email', 'phone', 'address']],
    ['PaymentMethod', ['cardToken']],
  ]);

  return async (params: any, next: any) => {
    const modelFields = encryptedFields.get(params.model);

    if (modelFields && params.action === 'create') {
      for (const field of modelFields) {
        if (params.args.data[field]) {
          params.args.data[field] = await fieldEncryption.encrypt(
            params.args.data[field],
            { model: params.model, field }
          );
        }
      }
    }

    const result = await next(params);

    if (modelFields && ['findUnique', 'findFirst', 'findMany'].includes(params.action)) {
      const decrypt = async (record: any) => {
        for (const field of modelFields) {
          if (record[field] && typeof record[field] === 'object') {
            record[field] = await fieldEncryption.decrypt(
              record[field],
              { model: params.model, field }
            );
          }
        }
        return record;
      };

      if (Array.isArray(result)) {
        return Promise.all(result.map(decrypt));
      } else if (result) {
        return decrypt(result);
      }
    }

    return result;
  };
};
```

---

## 第3章：鍵管理

### 3.1 KMSアーキテクチャ

#### 3.1.1 鍵階層設計

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         KEY HIERARCHY ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  Level 1: Root Keys (AWS KMS Customer Master Keys)                       │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │    │
│  │   │ Root CMK     │  │ DR Root CMK  │  │ Backup CMK   │                 │    │
│  │   │ (Primary)    │  │ (ap-ne-3)    │  │ (isolated)   │                 │    │
│  │   │              │  │              │  │              │                 │    │
│  │   │ Multi-Region │  │ Replica      │  │ Single-Region│                 │    │
│  │   └──────┬───────┘  └──────────────┘  └──────────────┘                 │    │
│  │          │                                                               │    │
│  └──────────┼───────────────────────────────────────────────────────────────┘    │
│             │                                                                    │
│             ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  Level 2: Service Master Keys                                           │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │    │
│  │   │ Database     │  │ Storage      │  │ Application  │  │ Secrets   │  │    │
│  │   │ Master Key   │  │ Master Key   │  │ Master Key   │  │ Master Key│  │    │
│  │   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └─────┬─────┘  │    │
│  │          │                 │                 │                │        │    │
│  └──────────┼─────────────────┼─────────────────┼────────────────┼────────┘    │
│             │                 │                 │                │             │
│             ▼                 ▼                 ▼                ▼             │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  Level 3: Data Encryption Keys (DEKs)                                    │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │    │
│  │   │ RDS DEK      │  │ S3 DEK       │  │ Field DEKs   │  │ API Keys  │  │    │
│  │   │ (per table)  │  │ (per bucket) │  │ (per record) │  │ DEK       │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────┘  └───────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  Key Management Services                                                 │    │
│  │                                                                          │    │
│  │   ┌──────────────────────────┐  ┌──────────────────────────────────┐   │    │
│  │   │ AWS KMS                   │  │ HashiCorp Vault                   │   │    │
│  │   │                           │  │                                    │   │    │
│  │   │ • CMK管理                 │  │ • 動的シークレット                 │   │    │
│  │   │ • DEK生成                 │  │ • PKI/証明書管理                  │   │    │
│  │   │ • 監査ログ                │  │ • アプリケーション統合            │   │    │
│  │   └──────────────────────────┘  └──────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 3.1.2 AWS KMS設定

```yaml
aws_kms_configuration:
  customer_master_keys:
    root_key:
      alias: alias/triptrip-root-key
      description: TripTrip Root Master Key
      key_spec: SYMMETRIC_DEFAULT
      key_usage: ENCRYPT_DECRYPT
      multi_region: true
      deletion_window: 30
      enable_key_rotation: true
      policy:
        Version: "2012-10-17"
        Statement:
          - Sid: Enable IAM User Permissions
            Effect: Allow
            Principal:
              AWS: "arn:aws:iam::xxx:root"
            Action: "kms:*"
            Resource: "*"

          - Sid: Allow Service Access
            Effect: Allow
            Principal:
              AWS:
                - "arn:aws:iam::xxx:role/triptrip-api-role"
                - "arn:aws:iam::xxx:role/triptrip-worker-role"
            Action:
              - "kms:Encrypt"
              - "kms:Decrypt"
              - "kms:GenerateDataKey"
              - "kms:GenerateDataKeyWithoutPlaintext"
            Resource: "*"
            Condition:
              StringEquals:
                kms:ViaService:
                  - "rds.ap-northeast-1.amazonaws.com"
                  - "s3.ap-northeast-1.amazonaws.com"

    database_key:
      alias: alias/triptrip-database-key
      description: TripTrip Database Encryption Key
      key_spec: SYMMETRIC_DEFAULT
      enable_key_rotation: true

    storage_key:
      alias: alias/triptrip-storage-key
      description: TripTrip Storage Encryption Key
      key_spec: SYMMETRIC_DEFAULT
      enable_key_rotation: true

    application_key:
      alias: alias/triptrip-application-key
      description: TripTrip Application-Level Encryption Key
      key_spec: SYMMETRIC_DEFAULT
      enable_key_rotation: true

  grants:
    rds_grant:
      key_id: alias/triptrip-database-key
      grantee_principal: "arn:aws:iam::xxx:role/rds-service-role"
      operations:
        - Encrypt
        - Decrypt
        - GenerateDataKey

    s3_grant:
      key_id: alias/triptrip-storage-key
      grantee_principal: "s3.amazonaws.com"
      operations:
        - Encrypt
        - Decrypt
        - GenerateDataKey
```

### 3.2 鍵ローテーション戦略

#### 3.2.1 ローテーションポリシー

```yaml
key_rotation_policy:
  aws_kms_cmk:
    automatic_rotation:
      enabled: true
      period: 365日
    manual_rotation:
      trigger:
        - セキュリティインシデント
        - コンプライアンス要件
        - 鍵の侵害疑い

  data_encryption_keys:
    rotation_period: 90日
    re_encryption:
      strategy: lazy_re_encryption
      batch_size: 1000 records
      priority:
        - 高機密データ優先
        - アクティブデータ優先

  api_keys:
    rotation_period: 30日
    grace_period: 7日
    notification: 7日前

  tls_certificates:
    rotation_period: 90日
    auto_renewal: true
    renewal_threshold: 30日前

  database_credentials:
    rotation_period: 30日
    managed_by: AWS Secrets Manager
    automatic: true
```

#### 3.2.2 鍵ローテーション実装

```typescript
// encryption/key-rotation.ts

import { KMSClient, ScheduleKeyDeletionCommand, CreateKeyCommand, UpdateAliasCommand } from '@aws-sdk/client-kms';
import { SecretsManagerClient, RotateSecretCommand } from '@aws-sdk/client-secrets-manager';

interface KeyRotationConfig {
  keyAlias: string;
  rotationPeriodDays: number;
  reEncryptionStrategy: 'immediate' | 'lazy';
}

class KeyRotationService {
  private kmsClient: KMSClient;
  private secretsClient: SecretsManagerClient;

  async rotateCustomerMasterKey(config: KeyRotationConfig): Promise<void> {
    // KMSの自動ローテーションを使用
    // マニュアルローテーションが必要な場合のみ実行
    console.log(`CMK rotation for ${config.keyAlias} handled by AWS KMS automatic rotation`);
  }

  async rotateDataEncryptionKey(
    keyAlias: string,
    affectedTables: string[]
  ): Promise<void> {
    // 1. 新しいDEKを生成（KMSが自動的に新しいキーバージョンを使用）

    // 2. 影響を受けるデータの再暗号化をスケジュール
    for (const table of affectedTables) {
      await this.scheduleReEncryption(table, keyAlias);
    }
  }

  private async scheduleReEncryption(
    table: string,
    keyAlias: string
  ): Promise<void> {
    // 再暗号化ジョブをキューに追加
    await this.jobQueue.add('re-encryption', {
      table,
      keyAlias,
      batchSize: 1000,
      priority: this.getReEncryptionPriority(table),
    });
  }

  async rotateDatabaseCredentials(secretName: string): Promise<void> {
    const command = new RotateSecretCommand({
      SecretId: secretName,
      RotationRules: {
        AutomaticallyAfterDays: 30,
      },
    });

    await this.secretsClient.send(command);
  }

  async rotateApiKey(apiKeyId: string): Promise<{ newKey: string; expiresAt: Date }> {
    // 1. 新しいAPIキーを生成
    const newKey = this.generateSecureApiKey();

    // 2. 新しいキーを保存
    await this.apiKeyRepository.create({
      id: this.generateId(),
      userId: apiKeyId,
      keyHash: await this.hashApiKey(newKey),
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
      status: 'active',
    });

    // 3. 古いキーにグレースピリオドを設定
    await this.apiKeyRepository.setGracePeriod(apiKeyId, 7);

    // 4. 通知
    await this.notificationService.sendApiKeyRotationNotice(apiKeyId);

    return {
      newKey,
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
    };
  }

  private generateSecureApiKey(): string {
    return `tt_${randomBytes(32).toString('base64url')}`;
  }
}

// 再暗号化ジョブプロセッサ
class ReEncryptionProcessor {
  async process(job: { table: string; keyAlias: string; batchSize: number }): Promise<void> {
    const { table, batchSize } = job;
    let cursor = null;

    do {
      // バッチでレコードを取得
      const records = await this.getRecordsBatch(table, cursor, batchSize);
      if (records.length === 0) break;

      // 各レコードを再暗号化
      for (const record of records) {
        await this.reEncryptRecord(table, record);
      }

      cursor = records[records.length - 1].id;

      // レート制限
      await this.sleep(100);
    } while (true);
  }

  private async reEncryptRecord(table: string, record: any): Promise<void> {
    const encryptedFields = this.getEncryptedFields(table);

    for (const field of encryptedFields) {
      if (record[field]) {
        // 古いキーで復号
        const plaintext = await this.fieldEncryption.decrypt(record[field]);

        // 新しいキーで暗号化
        const newEncrypted = await this.fieldEncryption.encrypt(plaintext);

        // 更新
        await this.updateField(table, record.id, field, newEncrypted);
      }
    }
  }
}
```

### 3.3 鍵階層設計

#### 3.3.1 HashiCorp Vault統合

```yaml
hashicorp_vault:
  deployment:
    type: HashiCorp Cloud Platform (HCP) Vault
    tier: Plus
    region: ap-northeast-1

  authentication:
    kubernetes_auth:
      enabled: true
      kubernetes_host: "https://kubernetes.default.svc"
      token_reviewer_jwt: serviceaccount token

    aws_auth:
      enabled: true
      type: iam

  secrets_engines:
    kv_v2:
      path: secret/triptrip
      config:
        max_versions: 10
        cas_required: false

    database:
      path: database
      config:
        plugin_name: postgresql-database-plugin
        allowed_roles: ["triptrip-api", "triptrip-worker"]
        connection_url: "postgresql://{{username}}:{{password}}@db.triptrip.internal:5432/triptrip"

    pki:
      path: pki
      config:
        max_ttl: "8760h"  # 1年

  policies:
    triptrip-api:
      path:
        - "secret/data/triptrip/api/*"
        - "database/creds/triptrip-api"
      capabilities:
        - read

    triptrip-worker:
      path:
        - "secret/data/triptrip/worker/*"
        - "database/creds/triptrip-worker"
      capabilities:
        - read

  dynamic_secrets:
    database_credentials:
      role: triptrip-api
      db_name: triptrip
      creation_statements:
        - "CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';"
        - "GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";"
      default_ttl: "1h"
      max_ttl: "24h"
```

---

## 第4章：PIIデータ保護

### 4.1 データ分類フレームワーク

#### 4.1.1 データ分類スキーム

```yaml
data_classification:
  levels:
    public:
      description: 一般公開データ
      examples:
        - 商品カタログ
        - 公開API仕様
        - マーケティング資料
      protection:
        encryption_at_rest: optional
        encryption_in_transit: required
        access_control: basic

    internal:
      description: 社内限定データ
      examples:
        - 内部ドキュメント
        - 業務データ
        - 匿名化された分析データ
      protection:
        encryption_at_rest: required
        encryption_in_transit: required
        access_control: role-based

    confidential:
      description: 機密データ
      examples:
        - 顧客PII
        - 予約詳細
        - パートナー契約情報
      protection:
        encryption_at_rest: required (field-level)
        encryption_in_transit: required
        access_control: need-to-know
        audit_logging: required

    restricted:
      description: 最高機密データ
      examples:
        - 決済カード情報
        - パスポート番号
        - 認証クレデンシャル
      protection:
        encryption_at_rest: required (application-level)
        encryption_in_transit: required (mTLS)
        access_control: strict (MFA + approval)
        audit_logging: enhanced
        retention: minimum necessary

  pii_categories:
    direct_identifiers:
      sensitivity: restricted
      fields:
        - passport_number
        - drivers_license_number
        - national_id

    indirect_identifiers:
      sensitivity: confidential
      fields:
        - name
        - email
        - phone_number
        - address
        - date_of_birth

    sensitive_pii:
      sensitivity: restricted
      fields:
        - racial_origin
        - religious_beliefs
        - health_data
        - biometric_data

    financial_data:
      sensitivity: restricted
      fields:
        - card_number
        - bank_account
        - credit_score
```

### 4.2 マスキング & トークナイゼーション

#### 4.2.1 データマスキング戦略

```yaml
data_masking:
  techniques:
    static_masking:
      description: 非本番環境用のデータ変換
      use_cases:
        - 開発環境
        - テスト環境
        - デモ環境
      methods:
        substitution:
          fields: [name, email]
          example:
            original: "田中太郎"
            masked: "テスト ユーザー001"
        shuffling:
          fields: [phone, address]
        nulling:
          fields: [passport_number, ssn]
        date_aging:
          fields: [date_of_birth]
          offset: random (-10 to +10 years)

    dynamic_masking:
      description: クエリ時のリアルタイムマスキング
      use_cases:
        - サポートチーム閲覧
        - レポーティング
        - 監査
      rules:
        support_role:
          email: "****@{domain}"
          phone: "***-****-{last4}"
          address: "{city}市 ***"
        auditor_role:
          email: full access
          phone: "***-****-{last4}"

  implementation:
    database_level:
      postgresql_views:
        - name: users_masked
          base_table: users
          masks:
            - column: email
              expression: "CONCAT(LEFT(email, 2), '****', '@', SPLIT_PART(email, '@', 2))"
            - column: phone
              expression: "CONCAT('***-****-', RIGHT(phone, 4))"

    application_level:
      middleware:
        - condition: user.role == 'support'
          apply_mask: true
          fields: [email, phone, address]
```

#### 4.2.2 トークナイゼーション実装

```typescript
// tokenization/tokenization-service.ts

import { randomBytes, createHash } from 'crypto';

interface TokenConfig {
  format: 'uuid' | 'format-preserving';
  preserveFormat?: {
    pattern: string;
    preservePositions?: number[];
  };
}

interface TokenizedValue {
  token: string;
  tokenType: string;
  createdAt: Date;
  expiresAt?: Date;
}

class TokenizationService {
  private vaultClient: VaultClient;
  private tokenStore: TokenStore;

  // トークン化（Vault Transit Secret Engineを使用）
  async tokenize(
    plaintext: string,
    dataType: string,
    config: TokenConfig
  ): Promise<string> {
    // 1. 既存トークンの確認（冪等性）
    const existingToken = await this.tokenStore.findByPlaintext(
      this.hashPlaintext(plaintext),
      dataType
    );

    if (existingToken) {
      return existingToken.token;
    }

    // 2. トークン生成
    let token: string;

    if (config.format === 'format-preserving') {
      token = this.generateFormatPreservingToken(plaintext, config.preserveFormat!);
    } else {
      token = `tok_${randomBytes(16).toString('hex')}`;
    }

    // 3. マッピング保存（暗号化）
    await this.tokenStore.save({
      token,
      plaintextHash: this.hashPlaintext(plaintext),
      encryptedPlaintext: await this.vaultClient.encrypt(plaintext),
      dataType,
      createdAt: new Date(),
    });

    return token;
  }

  // デトークン化
  async detokenize(token: string): Promise<string> {
    const tokenRecord = await this.tokenStore.findByToken(token);

    if (!tokenRecord) {
      throw new Error('Token not found');
    }

    // 監査ログ
    await this.auditLog('detokenize', { token, dataType: tokenRecord.dataType });

    return this.vaultClient.decrypt(tokenRecord.encryptedPlaintext);
  }

  // フォーマット保持トークン生成（例：クレジットカード番号）
  private generateFormatPreservingToken(
    plaintext: string,
    config: { pattern: string; preservePositions?: number[] }
  ): string {
    const { pattern, preservePositions = [] } = config;
    let result = '';

    for (let i = 0; i < plaintext.length; i++) {
      if (preservePositions.includes(i)) {
        result += plaintext[i];
      } else if (/\d/.test(plaintext[i])) {
        result += Math.floor(Math.random() * 10).toString();
      } else if (/[a-zA-Z]/.test(plaintext[i])) {
        result += String.fromCharCode(97 + Math.floor(Math.random() * 26));
      } else {
        result += plaintext[i];
      }
    }

    return result;
  }

  private hashPlaintext(plaintext: string): string {
    return createHash('sha256').update(plaintext).digest('hex');
  }
}

// クレジットカードトークナイゼーション（Stripe統合）
class CardTokenizationService {
  private stripe: Stripe;

  async tokenizeCard(cardDetails: {
    number: string;
    expMonth: number;
    expYear: number;
    cvc: string;
  }): Promise<{
    paymentMethodId: string;
    last4: string;
    brand: string;
  }> {
    // Stripeでカードをトークン化
    // カード情報は当社サーバーを経由しない
    const paymentMethod = await this.stripe.paymentMethods.create({
      type: 'card',
      card: {
        number: cardDetails.number,
        exp_month: cardDetails.expMonth,
        exp_year: cardDetails.expYear,
        cvc: cardDetails.cvc,
      },
    });

    return {
      paymentMethodId: paymentMethod.id,
      last4: paymentMethod.card!.last4,
      brand: paymentMethod.card!.brand,
    };
  }
}
```

### 4.3 匿名化技術

#### 4.3.1 匿名化手法

```yaml
anonymization_techniques:
  k_anonymity:
    description: 同一の準識別子を持つレコードがk個以上存在
    k_value: 5
    quasi_identifiers:
      - age_range
      - gender
      - postal_code_prefix

  l_diversity:
    description: 各等価クラス内で機密属性がl種類以上
    l_value: 3
    sensitive_attributes:
      - booking_amount_range
      - destination_region

  differential_privacy:
    description: 個人データの追加/削除が結果に大きく影響しない
    epsilon: 0.1
    use_cases:
      - 集計クエリ
      - 統計レポート
      - ML training data

  implementation:
    generalization:
      age:
        original: 32
        generalized: "30-39"
      location:
        original: "渋谷区代々木1-2-3"
        generalized: "渋谷区"
      date:
        original: "2024-01-15"
        generalized: "2024-01"

    suppression:
      threshold: 5
      action: replace with NULL or "*"
      fields:
        - rare_destinations
        - unique_preferences

    perturbation:
      numeric_fields:
        - booking_amount
        - review_rating
      noise_distribution: Laplacian
```

---

## 第5章：決済セキュリティ

### 5.1 PCI-DSS準拠アーキテクチャ

#### 5.1.1 カードホルダーデータ環境（CDE）

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    PCI-DSS COMPLIANT ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │  Scope Reduction: カード情報非保持                                      │   │
│   │                                                                          │   │
│   │   TripTripサーバーはカード情報を直接処理・保存しません                   │   │
│   │   Stripe.jsによるクライアントサイドトークナイゼーション                  │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  ┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐   │
│  │  Mobile App     │         │  Web App        │         │  Partner Portal │   │
│  │  (Flutter)      │         │  (Next.js)      │         │  (React)        │   │
│  └────────┬────────┘         └────────┬────────┘         └────────┬────────┘   │
│           │                           │                           │             │
│           │ Stripe SDK                │ Stripe.js                 │             │
│           │                           │                           │             │
│           ▼                           ▼                           ▼             │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                        Stripe (PCI Level 1)                              │   │
│  │                                                                          │   │
│  │   • カード情報の直接受信                                                 │   │
│  │   • トークン（PaymentMethod ID）の発行                                   │   │
│  │   • 決済処理                                                             │   │
│  │   • 3Dセキュア認証                                                       │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                     │                                           │
│                                     │ Token (pm_xxx)                            │
│                                     ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                        TripTrip Backend (Out of CDE Scope)               │   │
│  │                                                                          │   │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │   │
│  │   │ Payment      │  │ Booking      │  │ Notification │                 │   │
│  │   │ Service      │  │ Service      │  │ Service      │                 │   │
│  │   └──────────────┘  └──────────────┘  └──────────────┘                 │   │
│  │                                                                          │   │
│  │   保存データ:                                                            │   │
│  │   • PaymentMethod ID (トークン)                                          │   │
│  │   • カード下4桁 (表示用)                                                 │   │
│  │   • カードブランド                                                       │   │
│  │   • 有効期限（月/年）                                                    │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 5.1.2 PCI-DSS要件マッピング

```yaml
pci_dss_compliance:
  scope_reduction:
    strategy: カードホルダーデータの非保持
    implementation:
      - Stripe.js/SDKによるクライアントサイドトークナイゼーション
      - カード番号、CVV、磁気ストライプデータの非処理
      - SAQ A対象（最小スコープ）

  requirements:
    req_1_firewall:
      status: N/A (Stripe責任)
      notes: カードデータ環境なし

    req_2_default_passwords:
      status: 準拠
      implementation:
        - すべてのデフォルトパスワード変更
        - パスワードポリシー強制

    req_3_protect_stored_data:
      status: 準拠
      implementation:
        - カード番号の非保存
        - トークンのみ保存
        - 最小限のメタデータ（下4桁、ブランド）

    req_4_encrypt_transmission:
      status: 準拠
      implementation:
        - TLS 1.3強制
        - HSTSヘッダー
        - 証明書ピンニング（モバイル）

    req_5_antivirus:
      status: N/A (カードデータ環境なし)

    req_6_secure_systems:
      status: 準拠
      implementation:
        - 脆弱性管理プログラム
        - セキュアSDLC
        - ペネトレーションテスト

    req_7_restrict_access:
      status: 準拠
      implementation:
        - 最小権限原則
        - ロールベースアクセス制御

    req_8_identify_users:
      status: 準拠
      implementation:
        - 一意のユーザーID
        - MFA
        - パスワードポリシー

    req_9_physical_access:
      status: N/A (クラウドインフラ)

    req_10_track_access:
      status: 準拠
      implementation:
        - 包括的な監査ログ
        - ログ保護
        - 定期レビュー

    req_11_test_security:
      status: 準拠
      implementation:
        - 四半期脆弱性スキャン
        - 年次ペネトレーションテスト

    req_12_security_policy:
      status: 準拠
      implementation:
        - セキュリティポリシー文書化
        - セキュリティ意識向上トレーニング
```

### 5.2 カードデータ処理

#### 5.2.1 Stripe統合実装

```typescript
// payment/stripe-integration.ts

import Stripe from 'stripe';

interface PaymentIntentParams {
  amount: number;
  currency: string;
  customerId: string;
  paymentMethodId: string;
  bookingId: string;
  idempotencyKey: string;
}

class PaymentService {
  private stripe: Stripe;

  constructor() {
    this.stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
      apiVersion: '2024-04-10',
      typescript: true,
    });
  }

  // 決済インテント作成
  async createPaymentIntent(params: PaymentIntentParams): Promise<{
    clientSecret: string;
    status: string;
    requiresAction: boolean;
  }> {
    const paymentIntent = await this.stripe.paymentIntents.create(
      {
        amount: params.amount,
        currency: params.currency,
        customer: params.customerId,
        payment_method: params.paymentMethodId,
        confirmation_method: 'automatic',
        confirm: true,
        return_url: `${process.env.APP_URL}/bookings/${params.bookingId}/payment-complete`,
        metadata: {
          bookingId: params.bookingId,
        },
        // 3Dセキュア設定
        payment_method_options: {
          card: {
            request_three_d_secure: 'any',
          },
        },
      },
      {
        idempotencyKey: params.idempotencyKey,
      }
    );

    return {
      clientSecret: paymentIntent.client_secret!,
      status: paymentIntent.status,
      requiresAction: paymentIntent.status === 'requires_action',
    };
  }

  // 顧客作成
  async createCustomer(params: {
    email: string;
    name: string;
    metadata: Record<string, string>;
  }): Promise<string> {
    const customer = await this.stripe.customers.create({
      email: params.email,
      name: params.name,
      metadata: params.metadata,
    });

    return customer.id;
  }

  // 支払い方法の保存
  async attachPaymentMethod(
    paymentMethodId: string,
    customerId: string
  ): Promise<{
    id: string;
    last4: string;
    brand: string;
    expMonth: number;
    expYear: number;
  }> {
    const paymentMethod = await this.stripe.paymentMethods.attach(
      paymentMethodId,
      { customer: customerId }
    );

    return {
      id: paymentMethod.id,
      last4: paymentMethod.card!.last4,
      brand: paymentMethod.card!.brand,
      expMonth: paymentMethod.card!.exp_month,
      expYear: paymentMethod.card!.exp_year,
    };
  }

  // 返金処理
  async refund(params: {
    paymentIntentId: string;
    amount?: number;
    reason: 'duplicate' | 'fraudulent' | 'requested_by_customer';
    idempotencyKey: string;
  }): Promise<{
    id: string;
    status: string;
    amount: number;
  }> {
    const refund = await this.stripe.refunds.create(
      {
        payment_intent: params.paymentIntentId,
        amount: params.amount, // 部分返金の場合
        reason: params.reason,
      },
      {
        idempotencyKey: params.idempotencyKey,
      }
    );

    return {
      id: refund.id,
      status: refund.status,
      amount: refund.amount,
    };
  }

  // Webhookハンドラー
  async handleWebhook(
    payload: string,
    signature: string
  ): Promise<void> {
    let event: Stripe.Event;

    try {
      event = this.stripe.webhooks.constructEvent(
        payload,
        signature,
        process.env.STRIPE_WEBHOOK_SECRET!
      );
    } catch (err) {
      throw new Error(`Webhook signature verification failed`);
    }

    switch (event.type) {
      case 'payment_intent.succeeded':
        await this.handlePaymentSucceeded(event.data.object as Stripe.PaymentIntent);
        break;

      case 'payment_intent.payment_failed':
        await this.handlePaymentFailed(event.data.object as Stripe.PaymentIntent);
        break;

      case 'charge.dispute.created':
        await this.handleDisputeCreated(event.data.object as Stripe.Dispute);
        break;

      case 'customer.source.expiring':
        await this.handleCardExpiring(event.data.object as Stripe.Card);
        break;

      default:
        console.log(`Unhandled event type: ${event.type}`);
    }
  }

  private async handlePaymentSucceeded(paymentIntent: Stripe.PaymentIntent): Promise<void> {
    const bookingId = paymentIntent.metadata.bookingId;

    await this.bookingService.updatePaymentStatus(bookingId, {
      status: 'paid',
      paymentIntentId: paymentIntent.id,
      paidAt: new Date(),
    });

    await this.notificationService.sendPaymentConfirmation(bookingId);
  }

  private async handlePaymentFailed(paymentIntent: Stripe.PaymentIntent): Promise<void> {
    const bookingId = paymentIntent.metadata.bookingId;

    await this.bookingService.updatePaymentStatus(bookingId, {
      status: 'failed',
      failureReason: paymentIntent.last_payment_error?.message,
    });

    await this.notificationService.sendPaymentFailure(bookingId);
  }

  private async handleDisputeCreated(dispute: Stripe.Dispute): Promise<void> {
    // チャージバック対応
    await this.alertService.sendCriticalAlert({
      type: 'CHARGEBACK',
      disputeId: dispute.id,
      amount: dispute.amount,
      reason: dispute.reason,
    });
  }
}
```

### 5.3 決済トークナイゼーション

```yaml
payment_tokenization:
  stripe_tokens:
    payment_method:
      format: "pm_xxx"
      scope: 顧客紐付け
      reusable: true

    payment_intent:
      format: "pi_xxx"
      scope: 単一取引
      reusable: false

    setup_intent:
      format: "setu_xxx"
      scope: カード保存フロー
      reusable: false

  stored_data:
    allowed:
      - payment_method_id (Stripeトークン)
      - card_last_four (表示用)
      - card_brand
      - card_exp_month
      - card_exp_year
      - stripe_customer_id

    prohibited:
      - card_number (PAN)
      - card_cvv
      - magnetic_stripe_data
      - pin_data

  security_measures:
    - Stripeトークンの暗号化保存
    - アクセス制御
    - 監査ログ
    - 定期的なレビュー
```

---

## 第6章：実装ロードマップ & 文書間参照

### 6.1 実装ロードマップ

```yaml
implementation_roadmap:
  phase_1_foundation:
    timeline: 月1-2
    focus: 基盤暗号化
    deliverables:
      - AWS KMS CMKの設定
      - RDS暗号化の有効化
      - S3暗号化ポリシー
      - TLS 1.3設定
    success_criteria:
      - 保存時暗号化カバレッジ: 100%
      - TLS 1.3適用: 100%

  phase_2_application:
    timeline: 月3-4
    focus: アプリケーションレベル暗号化
    deliverables:
      - フィールドレベル暗号化実装
      - HashiCorp Vault統合
      - 鍵ローテーション自動化
    success_criteria:
      - PII暗号化: 100%
      - 鍵ローテーション自動化: 100%

  phase_3_pii:
    timeline: 月5-6
    focus: PIIデータ保護
    deliverables:
      - トークナイゼーションサービス
      - データマスキング実装
      - 匿名化パイプライン
    success_criteria:
      - 非本番環境のマスキング: 100%
      - トークナイゼーションカバレッジ: 100%

  phase_4_payment:
    timeline: 月7-8
    focus: 決済セキュリティ
    deliverables:
      - Stripe完全統合
      - 3Dセキュア2.0
      - PCI-DSS SAQ A準拠
    success_criteria:
      - PCI-DSS準拠: 100%
      - 3Dセキュア対応: 100%

  phase_5_operations:
    timeline: 月9-12
    focus: 運用成熟
    deliverables:
      - 鍵管理ダッシュボード
      - 暗号化監視
      - コンプライアンス監査対応
    success_criteria:
      - 監査指摘事項: 0件
      - 鍵管理SLA達成: 100%
```

### 6.2 文書間参照

```yaml
document_references:
  inputs:
    - document: Doc-DA-005
      title: データセキュリティ、プライバシー＆ガバナンス
      relevance: データ分類、ガバナンスポリシー
      key_integrations:
        - データ分類スキームの適用
        - 保持ポリシーとの整合

    - document: Doc-SC-001
      title: セキュリティアーキテクチャ＆戦略
      relevance: 多層防御、暗号化要件
      key_integrations:
        - データ層セキュリティ
        - 監視・検知

    - document: Doc-SC-002
      title: ID＆アクセス管理
      relevance: 鍵アクセス制御
      key_integrations:
        - KMSアクセスポリシー
        - Vault認証

  outputs:
    - document: Doc-SC-004
      title: コンプライアンス＆規制フレームワーク
      dependency: PCI-DSS、GDPR準拠証跡
```

### 6.3 技術スタックサマリー

```yaml
data_protection_technology_stack:
  encryption:
    at_rest:
      - AWS KMS (CMK管理)
      - RDS TDE
      - S3 SSE-KMS
      - EBS暗号化
    in_transit:
      - TLS 1.3
      - Istio mTLS
    application:
      - AES-256-GCM
      - Node.js crypto

  key_management:
    primary: AWS KMS
    secrets: HashiCorp Vault (HCP)
    rotation: 自動

  pii_protection:
    tokenization: Custom + Stripe
    masking: PostgreSQL Views + Application
    anonymization: Custom ETL

  payment:
    provider: Stripe
    pci_scope: SAQ A
    3ds: Stripe Radar
```

---

## 付録

### A. 暗号アルゴリズムリファレンス

| 用途 | アルゴリズム | 鍵長 | モード |
|------|-------------|------|--------|
| 対称暗号 | AES | 256bit | GCM |
| 非対称暗号 | RSA | 2048bit | OAEP |
| 非対称暗号 | ECDSA | P-256 | - |
| ハッシュ | SHA | 256bit | - |
| 鍵導出 | HKDF | - | SHA-256 |
| パスワード | Bcrypt | - | 12 rounds |

### B. データ分類マトリックス

| データ | 分類 | 暗号化 | マスキング | 保持期間 |
|--------|------|--------|-----------|----------|
| メールアドレス | Confidential | Field-level | 部分 | アカウント有効期間 |
| 電話番号 | Confidential | Field-level | 下4桁 | アカウント有効期間 |
| パスポート番号 | Restricted | Application | 完全 | 取引完了+7年 |
| カード番号 | Restricted | 非保存 | - | - |
| 予約履歴 | Confidential | At-rest | なし | 7年 |
| 検索履歴 | Internal | At-rest | なし | 90日 |

### C. 変更履歴

| バージョン | 日付 | 変更内容 | 作成者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-01-20 | 初版作成 | Technical Architecture Agent |

---

**文書情報**
- 文書ID: Doc-SC-003
- バージョン: 1.0.0
- ステータス: ドラフト
- 最終更新: 2026-01-20
- 次回レビュー: 2026-02-20
- 行数: 約2,000行

# Doc-DA-005: TripTripデータセキュリティ、プライバシー & ガバナンス

## エグゼクティブサマリー

本文書は、TripTripプラットフォームの包括的なデータセキュリティ、プライバシー保護、およびデータガバナンスフレームワークを定義します。暗号化戦略（保存時、転送時、使用時）、データ分類とハンドリングポリシー、PII保護、GDPR・CCPA・日本個人情報保護法・PCI-DSSへのコンプライアンス実装、データアクセス制御、監査ログ、データカタログとリネージ管理を詳述します。Google、Amazon、Netflixレベルのセキュリティ基準を採用し、グローバル規制に完全準拠しながら、データ駆動型ビジネスを安全に推進する基盤を構築します。本設計は、Doc-DA-001〜004で定義されたデータアーキテクチャ全体にセキュリティとガバナンスの層を追加します。

---

## 第1章：はじめに & コンテキスト

### 1.1 ビジネス要件 & 制約

#### 1.1.1 セキュリティ・コンプライアンス要件

```yaml
security_compliance_requirements:
  regulatory_requirements:
    gdpr:
      applicability: EU市場展開時
      key_requirements:
        - データ主体の権利（アクセス、訂正、削除、ポータビリティ）
        - 適法な処理根拠
        - 同意管理
        - 越境データ移転制限
        - 72時間以内のデータ侵害通知
        - データ保護影響評価（DPIA）
      penalties: 最大2,000万ユーロまたは全世界売上高の4%

    ccpa:
      applicability: カリフォルニア州居住者
      key_requirements:
        - 消費者の知る権利
        - 削除権
        - オプトアウト権
        - 差別禁止
      penalties: 意図的違反1件あたり$7,500

    japanese_privacy_law:
      applicability: 日本国内（主要市場）
      key_requirements:
        - 個人情報の適正な取得
        - 利用目的の特定と通知
        - 第三者提供の制限
        - 安全管理措置
        - 開示・訂正・利用停止請求
      penalties: 最大1億円

    pci_dss:
      applicability: 決済データ処理
      level: Level 2（100万〜600万トランザクション/年）
      key_requirements:
        - カードデータの暗号化
        - アクセス制御
        - ネットワークセグメンテーション
        - 脆弱性管理
        - 定期的な監査

  business_requirements:
    data_protection:
      - 顧客データの機密性保護
      - 決済情報の安全な処理
      - パートナーデータの保護
      - 知的財産の保護

    compliance:
      - 法的リスクの最小化
      - 監査対応の効率化
      - 国際展開への準備

    trust:
      - 顧客信頼の構築
      - ブランド保護
      - パートナー信頼の維持
```

#### 1.1.2 リスク評価

```yaml
risk_assessment:
  data_breach:
    likelihood: Medium
    impact: Critical
    mitigation:
      - 多層防御アーキテクチャ
      - 暗号化
      - アクセス制御
      - 監視・検知

  regulatory_violation:
    likelihood: Medium
    impact: High
    mitigation:
      - コンプライアンスフレームワーク
      - 自動化されたポリシー適用
      - 定期監査

  insider_threat:
    likelihood: Low
    impact: High
    mitigation:
      - 最小権限原則
      - アクセスログ
      - 異常検知

  data_loss:
    likelihood: Low
    impact: Critical
    mitigation:
      - バックアップ
      - レプリケーション
      - 災害復旧計画
```

### 1.2 規制環境

#### 1.2.1 グローバル規制マップ

```yaml
regulatory_landscape:
  asia_pacific:
    japan:
      primary: 個人情報保護法（APPI）
      sector_specific:
        - 割賦販売法（決済）
        - 旅行業法
      authority: 個人情報保護委員会

    singapore:
      primary: PDPA
      authority: PDPC

    australia:
      primary: Privacy Act 1988
      authority: OAIC

  europe:
    eu_member_states:
      primary: GDPR
      authority: 各国DPA
      lead_authority: Irish DPC（EU代表オフィスの場合）

    uk:
      primary: UK GDPR + Data Protection Act 2018
      authority: ICO

  americas:
    united_states:
      federal: FTC Act
      state_level:
        - CCPA/CPRA（California）
        - VCDPA（Virginia）
        - CPA（Colorado）
      sector_specific:
        - PCI-DSS（決済）

  cross_border:
    mechanisms:
      - SCCs（Standard Contractual Clauses）
      - APEC CBPR
      - 十分性認定
```

### 1.3 主要な前提条件

```yaml
key_assumptions:
  technical:
    - クラウドファーストアーキテクチャ（AWS/GCP）
    - マイクロサービス構成
    - PostgreSQL + BigQuery データストア
    - Kubernetes コンテナ基盤

  organizational:
    - DPO（データ保護責任者）任命
    - セキュリティチーム設置
    - 定期的なセキュリティトレーニング

  business:
    - 日本市場からの段階的グローバル展開
    - 決済処理は外部PSP委託
    - パートナーデータの取り扱いあり
```

---

## 第2章：データセキュリティアーキテクチャ

### 2.1 暗号化戦略

#### 2.1.1 暗号化アーキテクチャ概要

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         ENCRYPTION ARCHITECTURE                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      Client Layer                                        │    │
│  │                                                                          │    │
│  │   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐         │    │
│  │   │  Mobile App  │      │  Web Browser │      │  Partner API │         │    │
│  │   │  (TLS 1.3)   │      │  (TLS 1.3)   │      │  (TLS 1.3)   │         │    │
│  │   └──────┬───────┘      └──────┬───────┘      └──────┬───────┘         │    │
│  └──────────┼──────────────────────┼──────────────────────┼─────────────────┘    │
│             │                      │                      │                      │
│             └──────────────────────┼──────────────────────┘                      │
│                                    ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      Transport Encryption                                │    │
│  │                                                                          │    │
│  │   ┌──────────────────────────────────────────────────────────────────┐  │    │
│  │   │  CloudFront / ALB (TLS 1.3, Perfect Forward Secrecy)             │  │    │
│  │   │  Certificate: ACM Managed, Auto-Renewal                           │  │    │
│  │   └──────────────────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                    │                                             │
│                                    ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      Application Layer                                   │    │
│  │                                                                          │    │
│  │   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐           │    │
│  │   │ API Services   │  │ Internal Comms │  │ Secrets Mgmt   │           │    │
│  │   │ (mTLS option)  │  │ (Service Mesh) │  │ (Vault/ASM)    │           │    │
│  │   └────────────────┘  └────────────────┘  └────────────────┘           │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                    │                                             │
│                                    ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      Storage Encryption                                  │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │    │
│  │   │  PostgreSQL  │  │   BigQuery   │  │     S3       │  │   Redis   │  │    │
│  │   │  (AES-256)   │  │  (AES-256)   │  │  (AES-256)   │  │ (AES-256) │  │    │
│  │   │  KMS Managed │  │  CMEK        │  │  SSE-KMS     │  │ In-Transit│  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────┘  └───────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 2.1.2 暗号化ポリシー詳細

```yaml
encryption_policies:
  data_at_rest:
    database:
      postgresql:
        encryption: AES-256
        key_management: AWS KMS
        key_rotation: 自動（年次）
        implementation: TDE (Transparent Data Encryption)

      bigquery:
        encryption: AES-256
        key_management: Google CMEK
        key_rotation: 自動（年次）

    object_storage:
      s3:
        default: SSE-S3 (AES-256)
        sensitive: SSE-KMS (Customer Managed Keys)
        key_rotation: 自動（年次）
        bucket_policy: 暗号化必須

    block_storage:
      ebs:
        encryption: AES-256
        key_management: AWS KMS

    backup:
      encryption: AES-256
      key_management: 専用バックアップキー
      key_escrow: 安全なオフサイト保管

  data_in_transit:
    external:
      protocol: TLS 1.3
      cipher_suites:
        - TLS_AES_256_GCM_SHA384
        - TLS_CHACHA20_POLY1305_SHA256
      perfect_forward_secrecy: required
      certificate_authority: ACM (AWS Certificate Manager)
      hsts: enabled
      hsts_max_age: 31536000

    internal:
      service_mesh: Istio mTLS
      database_connections: TLS required
      redis: TLS + AUTH
      kafka: TLS + SASL

  data_in_use:
    application_level:
      pii_fields:
        encryption: AES-256-GCM
        key_derivation: HKDF
        per_user_keys: true

      payment_data:
        encryption: AES-256-GCM
        tokenization: 外部PSPトークン化
        no_storage: PAN保持禁止

  key_management:
    primary_kms: AWS KMS
    secondary_kms: GCP Cloud KMS
    key_hierarchy:
      master_key: HSM保護
      data_encryption_keys: KMS管理
      application_keys: 自動ローテーション
    access_control:
      principle: 最小権限
      approval: 複数承認必須
    audit: すべての鍵操作をログ
```

#### 2.1.3 暗号化実装例

```typescript
// アプリケーションレベル暗号化サービス
import { createCipheriv, createDecipheriv, randomBytes, scrypt } from 'crypto';
import { KMSClient, GenerateDataKeyCommand, DecryptCommand } from '@aws-sdk/client-kms';

interface EncryptionConfig {
  kmsKeyId: string;
  algorithm: 'aes-256-gcm';
  ivLength: number;
  tagLength: number;
}

class FieldEncryptionService {
  private kmsClient: KMSClient;
  private config: EncryptionConfig;
  private dataKeyCache: Map<string, { key: Buffer; expiresAt: Date }> = new Map();

  constructor(config: EncryptionConfig) {
    this.kmsClient = new KMSClient({ region: process.env.AWS_REGION });
    this.config = config;
  }

  // フィールドレベル暗号化
  async encryptField(plaintext: string, context: Record<string, string>): Promise<string> {
    const { plaintextKey, encryptedKey } = await this.getDataKey(context);

    const iv = randomBytes(this.config.ivLength);
    const cipher = createCipheriv(this.config.algorithm, plaintextKey, iv, {
      authTagLength: this.config.tagLength,
    });

    const encrypted = Buffer.concat([
      cipher.update(plaintext, 'utf8'),
      cipher.final(),
    ]);

    const authTag = cipher.getAuthTag();

    // フォーマット: base64(encryptedKey + iv + authTag + encrypted)
    const combined = Buffer.concat([
      Buffer.from([encryptedKey.length & 0xff, (encryptedKey.length >> 8) & 0xff]),
      encryptedKey,
      iv,
      authTag,
      encrypted,
    ]);

    return combined.toString('base64');
  }

  // フィールドレベル復号
  async decryptField(ciphertext: string, context: Record<string, string>): Promise<string> {
    const combined = Buffer.from(ciphertext, 'base64');

    const encryptedKeyLength = combined[0] | (combined[1] << 8);
    let offset = 2;

    const encryptedKey = combined.subarray(offset, offset + encryptedKeyLength);
    offset += encryptedKeyLength;

    const iv = combined.subarray(offset, offset + this.config.ivLength);
    offset += this.config.ivLength;

    const authTag = combined.subarray(offset, offset + this.config.tagLength);
    offset += this.config.tagLength;

    const encrypted = combined.subarray(offset);

    const plaintextKey = await this.decryptDataKey(encryptedKey, context);

    const decipher = createDecipheriv(this.config.algorithm, plaintextKey, iv, {
      authTagLength: this.config.tagLength,
    });
    decipher.setAuthTag(authTag);

    const decrypted = Buffer.concat([
      decipher.update(encrypted),
      decipher.final(),
    ]);

    return decrypted.toString('utf8');
  }

  // データキー生成（KMS）
  private async getDataKey(context: Record<string, string>): Promise<{
    plaintextKey: Buffer;
    encryptedKey: Buffer;
  }> {
    const command = new GenerateDataKeyCommand({
      KeyId: this.config.kmsKeyId,
      KeySpec: 'AES_256',
      EncryptionContext: context,
    });

    const response = await this.kmsClient.send(command);

    return {
      plaintextKey: Buffer.from(response.Plaintext!),
      encryptedKey: Buffer.from(response.CiphertextBlob!),
    };
  }

  // データキー復号（KMS）
  private async decryptDataKey(
    encryptedKey: Buffer,
    context: Record<string, string>
  ): Promise<Buffer> {
    const command = new DecryptCommand({
      CiphertextBlob: encryptedKey,
      EncryptionContext: context,
    });

    const response = await this.kmsClient.send(command);
    return Buffer.from(response.Plaintext!);
  }
}

// PII暗号化ミドルウェア
const piiFields = ['email', 'phone', 'address', 'passport_number'];

async function encryptPIIFields(
  data: Record<string, any>,
  userId: string,
  encryptionService: FieldEncryptionService
): Promise<Record<string, any>> {
  const encrypted = { ...data };
  const context = { userId, purpose: 'pii_protection' };

  for (const field of piiFields) {
    if (encrypted[field]) {
      encrypted[field] = await encryptionService.encryptField(
        String(encrypted[field]),
        context
      );
    }
  }

  return encrypted;
}
```

### 2.2 アクセス制御

#### 2.2.1 アクセス制御モデル

```yaml
access_control_model:
  principle: RBAC + ABAC ハイブリッド

  rbac_roles:
    system_roles:
      super_admin:
        description: システム全体の管理者
        permissions:
          - "*"
        assignment: 最小限（2-3名）
        mfa: required
        session_timeout: 1時間

      platform_admin:
        description: プラットフォーム運用管理者
        permissions:
          - user:read
          - user:update
          - booking:read
          - booking:update
          - partner:read
          - partner:update
        assignment: 運用チーム
        mfa: required

      data_analyst:
        description: データ分析担当
        permissions:
          - analytics:read
          - report:read
          - user:read_anonymized
          - booking:read_aggregated
        assignment: 分析チーム
        data_masking: enabled

      customer_support:
        description: カスタマーサポート担当
        permissions:
          - user:read
          - booking:read
          - booking:update_status
          - support_ticket:*
        assignment: サポートチーム
        pii_access: limited

      developer:
        description: 開発者
        permissions:
          - api:read
          - log:read
          - config:read
        assignment: 開発チーム
        environment: staging/dev only

    partner_roles:
      partner_admin:
        description: パートナー管理者
        permissions:
          - own_entity:*
          - own_booking:read
          - own_analytics:read
        scope: own_partner_id

      partner_staff:
        description: パートナースタッフ
        permissions:
          - own_entity:read
          - own_booking:read
        scope: own_partner_id

  abac_attributes:
    user_attributes:
      - department
      - location
      - clearance_level
      - employment_status

    resource_attributes:
      - data_classification
      - owner
      - region
      - sensitivity_level

    environment_attributes:
      - time_of_day
      - ip_address
      - device_type
      - network_zone

  access_policies:
    pii_access:
      condition: |
        user.role IN ['customer_support', 'platform_admin']
        AND user.clearance_level >= 'CONFIDENTIAL'
        AND resource.data_classification == 'PII'
        AND environment.network_zone == 'INTERNAL'
      action: READ
      audit: always

    production_data_access:
      condition: |
        user.role IN ['platform_admin', 'super_admin']
        AND environment.network_zone == 'INTERNAL'
        AND time_of_day BETWEEN '09:00' AND '18:00'
      action: READ/WRITE
      approval: required_for_write
```

#### 2.2.2 IAM実装（AWS）

```yaml
aws_iam_implementation:
  identity_provider:
    type: AWS SSO
    integration: Okta/Azure AD
    mfa: enforced

  permission_boundaries:
    purpose: 権限エスカレーション防止
    policy: |
      {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Effect": "Allow",
            "Action": "*",
            "Resource": "*",
            "Condition": {
              "StringNotEquals": {
                "aws:RequestedRegion": ["us-gov-*", "cn-*"]
              }
            }
          },
          {
            "Effect": "Deny",
            "Action": [
              "iam:CreateUser",
              "iam:DeleteUser",
              "iam:CreateRole",
              "iam:DeleteRole",
              "organizations:*"
            ],
            "Resource": "*"
          }
        ]
      }

  service_roles:
    api_service:
      trust: EKS Service Account
      policies:
        - SecretsManagerReadWrite
        - RDSDataRead
        - S3ReadWrite
      conditions:
        - StringEquals:
            aws:PrincipalTag/environment: production

    analytics_service:
      trust: EKS Service Account
      policies:
        - BigQueryDataViewer
        - S3ReadOnly
      conditions:
        - IpAddress:
            aws:SourceIp: [internal_cidr]

  access_analyzer:
    enabled: true
    archive_rules:
      - trusted_accounts
    findings_notification: security_team
```

### 2.3 ネットワークセキュリティ

#### 2.3.1 ネットワークセグメンテーション

```yaml
network_segmentation:
  vpc_design:
    production_vpc:
      cidr: 10.0.0.0/16
      segments:
        public:
          cidr: 10.0.0.0/20
          purpose: ALB, NAT Gateway
          internet_access: inbound/outbound

        private_app:
          cidr: 10.0.16.0/20
          purpose: アプリケーションサーバー
          internet_access: outbound_only

        private_data:
          cidr: 10.0.32.0/20
          purpose: データベース、キャッシュ
          internet_access: none

        private_management:
          cidr: 10.0.48.0/20
          purpose: 管理系サーバー
          internet_access: outbound_only

  security_groups:
    alb_sg:
      inbound:
        - port: 443
          source: 0.0.0.0/0
          description: HTTPS
      outbound:
        - port: all
          destination: app_sg

    app_sg:
      inbound:
        - port: 3000
          source: alb_sg
          description: ALB Health Check
        - port: 3000
          source: app_sg
          description: Service Mesh
      outbound:
        - port: 5432
          destination: db_sg
        - port: 6379
          destination: cache_sg
        - port: 443
          destination: 0.0.0.0/0

    db_sg:
      inbound:
        - port: 5432
          source: app_sg
          description: PostgreSQL
      outbound: []

    cache_sg:
      inbound:
        - port: 6379
          source: app_sg
          description: Redis
      outbound: []

  network_acls:
    private_data_nacl:
      inbound:
        - rule: 100
          port: 5432
          source: 10.0.16.0/20
          action: allow
        - rule: 200
          port: 6379
          source: 10.0.16.0/20
          action: allow
        - rule: "*"
          action: deny
      outbound:
        - rule: 100
          port: 1024-65535
          destination: 10.0.16.0/20
          action: allow
        - rule: "*"
          action: deny

  waf_rules:
    web_acl:
      rules:
        - name: AWS-AWSManagedRulesCommonRuleSet
          priority: 1
        - name: AWS-AWSManagedRulesKnownBadInputsRuleSet
          priority: 2
        - name: AWS-AWSManagedRulesSQLiRuleSet
          priority: 3
        - name: RateLimitRule
          priority: 4
          rate_limit: 2000/5min
        - name: GeoBlockRule
          priority: 5
          blocked_countries: [sanctioned_countries]
```

---

## 第3章：データプライバシーフレームワーク

### 3.1 PII分類とハンドリング

#### 3.1.1 データ分類スキーム

```yaml
data_classification:
  levels:
    public:
      description: 公開情報
      examples:
        - 公開プロフィール名
        - 公開レビュー
        - 一般的な施設情報
      handling:
        encryption_at_rest: optional
        encryption_in_transit: required
        access_control: basic
        retention: indefinite
        backup: standard

    internal:
      description: 社内情報
      examples:
        - 匿名化された分析データ
        - 集計レポート
        - 非機密設定
      handling:
        encryption_at_rest: required
        encryption_in_transit: required
        access_control: role_based
        retention: policy_based
        backup: standard

    confidential:
      description: 機密情報
      examples:
        - ユーザーメールアドレス
        - 電話番号
        - 予約履歴
        - 行動ログ
      handling:
        encryption_at_rest: required
        encryption_in_transit: required
        access_control: strict_rbac
        retention: minimization
        backup: encrypted
        audit: required

    restricted:
      description: 最高機密情報
      examples:
        - パスポート情報
        - 決済カード情報
        - 健康情報
        - 政府発行ID
      handling:
        encryption_at_rest: field_level
        encryption_in_transit: required
        access_control: need_to_know
        retention: minimum_required
        backup: encrypted_separate
        audit: comprehensive
        masking: always

  pii_inventory:
    user_data:
      direct_identifiers:
        - field: email
          classification: confidential
          purpose: authentication, communication
          retention: account_lifetime + 30_days
          masking_rule: partial_mask

        - field: phone
          classification: confidential
          purpose: verification, emergency_contact
          retention: account_lifetime + 30_days
          masking_rule: partial_mask

        - field: full_name
          classification: confidential
          purpose: booking, identification
          retention: account_lifetime + legal_requirement
          masking_rule: name_mask

        - field: passport_number
          classification: restricted
          purpose: international_booking
          retention: booking_lifetime + 90_days
          masking_rule: full_mask_except_last4

        - field: government_id
          classification: restricted
          purpose: age_verification
          retention: verification_completion
          masking_rule: full_mask

      indirect_identifiers:
        - field: birth_date
          classification: confidential
          purpose: age_verification, personalization
          masking_rule: year_only

        - field: address
          classification: confidential
          purpose: delivery, billing
          masking_rule: city_only

        - field: ip_address
          classification: confidential
          purpose: security, fraud_detection
          retention: 90_days
          anonymization: hash_after_retention

      behavioral_data:
        - field: browsing_history
          classification: confidential
          purpose: personalization
          retention: 180_days

        - field: booking_history
          classification: confidential
          purpose: service_delivery, analytics
          retention: legal_requirement
```

#### 3.1.2 データマスキング実装

```sql
-- BigQueryでのデータマスキングポリシー
CREATE OR REPLACE ROW ACCESS POLICY user_data_access
ON dwh.dim_user
GRANT TO ("group:data_analysts@triptrip.com")
FILTER USING (
  -- データアナリストには匿名化されたデータのみ
  SESSION_USER() IN (
    SELECT email FROM admin.analyst_whitelist
  )
);

-- カラムレベルマスキング（BigQuery Data Policy）
CREATE DATA POLICY email_masking
OPTIONS (
  description = "Mask email addresses for non-privileged users"
)
AS (
  CASE
    WHEN SESSION_USER() IN (
      SELECT email FROM admin.pii_access_whitelist
    )
    THEN email
    ELSE CONCAT(
      SUBSTR(email, 1, 2),
      '***@',
      SUBSTR(email, STRPOS(email, '@') + 1)
    )
  END
);

-- PostgreSQLでのビューベースマスキング
CREATE OR REPLACE VIEW public.v_users_masked AS
SELECT
  id,
  -- メールマスキング
  CASE
    WHEN current_setting('app.user_role', true) IN ('admin', 'support')
    THEN email
    ELSE CONCAT(
      LEFT(email, 2),
      '***@',
      SPLIT_PART(email, '@', 2)
    )
  END AS email,
  -- 電話番号マスキング
  CASE
    WHEN current_setting('app.user_role', true) IN ('admin', 'support')
    THEN phone
    ELSE CONCAT('***-****-', RIGHT(phone, 4))
  END AS phone,
  -- 名前マスキング
  CASE
    WHEN current_setting('app.user_role', true) IN ('admin', 'support')
    THEN display_name
    ELSE CONCAT(LEFT(display_name, 1), '***')
  END AS display_name,
  -- 非機密データはそのまま
  preferred_language,
  preferred_currency,
  created_at
FROM users;
```

### 3.2 同意管理

#### 3.2.1 同意管理アーキテクチャ

```yaml
consent_management:
  consent_types:
    essential:
      description: サービス提供に必須
      legal_basis: contract
      withdrawable: false
      examples:
        - アカウント作成
        - 予約処理
        - 決済処理

    marketing:
      description: マーケティングコミュニケーション
      legal_basis: consent
      withdrawable: true
      default: opt_out
      examples:
        - プロモーションメール
        - ニュースレター
        - プッシュ通知

    analytics:
      description: サービス改善のための分析
      legal_basis: legitimate_interest
      withdrawable: true
      default: opt_in
      examples:
        - 利用統計
        - A/Bテスト
        - パフォーマンス分析

    personalization:
      description: パーソナライズされた体験
      legal_basis: consent
      withdrawable: true
      default: opt_out
      examples:
        - 推奨エンジン
        - 検索パーソナライズ
        - 価格最適化

    third_party:
      description: 第三者との共有
      legal_basis: consent
      withdrawable: true
      default: opt_out
      examples:
        - パートナー共有
        - 広告ネットワーク
        - 分析ベンダー

  consent_storage:
    database:
      table: user_consents
      schema: |
        CREATE TABLE user_consents (
          id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
          user_id UUID NOT NULL REFERENCES users(id),
          consent_type VARCHAR(50) NOT NULL,
          status VARCHAR(20) NOT NULL, -- granted, denied, withdrawn
          granted_at TIMESTAMPTZ,
          withdrawn_at TIMESTAMPTZ,
          ip_address INET,
          user_agent TEXT,
          consent_version VARCHAR(20) NOT NULL,
          source VARCHAR(50), -- web, app, api
          metadata JSONB,
          created_at TIMESTAMPTZ DEFAULT NOW(),
          UNIQUE(user_id, consent_type, consent_version)
        );

  consent_api:
    endpoints:
      get_consents:
        path: /api/v1/users/{userId}/consents
        method: GET
        response:
          - consent_type
          - status
          - granted_at
          - consent_version

      update_consent:
        path: /api/v1/users/{userId}/consents/{consentType}
        method: PUT
        body:
          status: granted | denied
        audit: required

      withdraw_all:
        path: /api/v1/users/{userId}/consents/withdraw-all
        method: POST
        audit: required
```

### 3.3 データ主体の権利

#### 3.3.1 権利実装

```yaml
data_subject_rights:
  right_of_access:
    description: 自身のデータへのアクセス権
    implementation:
      - セルフサービスダウンロード機能
      - 構造化フォーマット（JSON, CSV）
      - 30日以内の対応
    endpoint: /api/v1/users/{userId}/data-export
    format: JSON, CSV
    scope:
      - プロフィール情報
      - 予約履歴
      - 注文履歴
      - 同意履歴
      - アクティビティログ（直近1年）

  right_of_rectification:
    description: データ訂正の権利
    implementation:
      - プロフィール編集機能
      - サポート経由の訂正リクエスト
    endpoint: /api/v1/users/{userId}/profile
    method: PATCH
    audit: required

  right_of_erasure:
    description: 削除権（忘れられる権利）
    implementation:
      - アカウント削除機能
      - 30日の猶予期間
      - 法的保持要件の例外
    endpoint: /api/v1/users/{userId}/deletion-request
    process:
      1. 削除リクエスト受付
      2. 本人確認
      3. 30日猶予（復元可能）
      4. 論理削除実行
      5. バックアップからの完全削除（90日以内）
    exceptions:
      - 法的保持義務のあるデータ
      - 進行中の取引
      - 紛争解決中のデータ

  right_of_portability:
    description: データポータビリティの権利
    implementation:
      - 機械可読フォーマットでのエクスポート
      - 他サービスへの直接転送（将来）
    format: JSON
    scope:
      - 本人が提供したデータ
      - 同意または契約に基づくデータ

  right_to_object:
    description: 処理に対する異議申立権
    implementation:
      - マーケティングオプトアウト
      - プロファイリングオプトアウト
    endpoint: /api/v1/users/{userId}/processing-objection
```

---

## 第4章：コンプライアンス実装

### 4.1 GDPR対応

#### 4.1.1 GDPR技術的措置

```yaml
gdpr_implementation:
  article_25_privacy_by_design:
    measures:
      data_minimization:
        - 必要最小限のデータ収集
        - 目的達成後の削除
        - デフォルトでのプライバシー保護

      pseudonymization:
        - 内部識別子の使用
        - 分析データの匿名化
        - キーセパレーション

      access_controls:
        - ロールベースアクセス
        - 監査ログ
        - 暗号化

  article_30_records_of_processing:
    processing_register:
      fields:
        - processing_activity_name
        - purpose
        - data_categories
        - data_subjects
        - recipients
        - transfers
        - retention_period
        - security_measures
      maintenance: 四半期レビュー
      storage: 専用データベース

  article_32_security:
    measures:
      - encryption_at_rest
      - encryption_in_transit
      - pseudonymization
      - access_controls
      - regular_testing
      - incident_response

  article_33_breach_notification:
    process:
      detection:
        - 自動異常検知
        - セキュリティモニタリング
        - 従業員報告
      assessment:
        - 影響範囲特定
        - リスク評価
        - 通知要否判断
      notification:
        supervisory_authority:
          timeline: 72時間以内
          content:
            - 侵害の性質
            - 影響を受けるデータ主体数
            - DPO連絡先
            - 想定される影響
            - 対策
        data_subjects:
          timeline: 不当な遅延なく
          condition: 高リスクの場合
          content:
            - 平易な言葉での説明
            - 対策のアドバイス

  article_35_dpia:
    triggers:
      - 新規データ処理活動
      - 大規模プロファイリング
      - センシティブデータの処理
      - 新技術の導入
    template:
      sections:
        - 処理の説明
        - 必要性と比例性の評価
        - リスク評価
        - リスク軽減措置
    review: 年次または変更時
```

### 4.2 CCPA対応

#### 4.2.1 CCPA技術実装

```yaml
ccpa_implementation:
  consumer_rights:
    right_to_know:
      implementation:
        - プライバシーポリシーでの開示
        - 個人情報カテゴリーの開示
        - 収集目的の開示
        - 第三者共有の開示
      endpoint: /api/v1/privacy/california/disclosure

    right_to_delete:
      implementation:
        - 削除リクエストポータル
        - 本人確認プロセス
        - 45日以内の対応
      exceptions:
        - 取引完了に必要
        - 法的義務
        - セキュリティ目的

    right_to_opt_out:
      implementation:
        - "Do Not Sell My Personal Information"リンク
        - Global Privacy Control対応
        - 第三者広告オプトアウト
      endpoint: /api/v1/privacy/california/opt-out

    right_to_non_discrimination:
      implementation:
        - オプトアウトによるサービス差別禁止
        - 同等のサービス品質維持

  verification:
    methods:
      - メール確認
      - SMS確認
      - セキュリティ質問
    standards:
      - 低リスク: 1要素
      - 高リスク: 2要素
```

### 4.3 日本個人情報保護法対応

#### 4.3.1 APPI技術実装

```yaml
appi_implementation:
  article_17_proper_acquisition:
    measures:
      - 利用目的の明示
      - 同意の取得
      - 不正取得の禁止

  article_18_purpose_specification:
    implementation:
      - プライバシーポリシーでの目的特定
      - 目的外利用の禁止
      - 目的変更時の通知

  article_20_security_measures:
    organizational:
      - 個人データ取扱責任者の設置
      - 従業員教育
      - 取扱規程の整備
    physical:
      - 入退室管理
      - 機器の盗難防止
    technical:
      - アクセス制御
      - 暗号化
      - 不正アクセス防止

  article_23_third_party_provision:
    implementation:
      - 第三者提供の同意取得
      - 委託先の監督
      - 提供記録の作成
    exceptions:
      - 法令に基づく場合
      - 人の生命・身体・財産の保護
      - 公衆衛生の向上

  cross_border_transfer:
    mechanisms:
      - 同意
      - 十分性認定国への移転
      - 基準適合体制
```

### 4.4 PCI-DSS対応

#### 4.4.1 PCI-DSS技術アーキテクチャ

```yaml
pci_dss_implementation:
  architecture_approach:
    strategy: Tokenization via PSP
    psp: Stripe
    scope_reduction:
      - カード情報はTripTripシステムに保存しない
      - Stripe Elements使用（クライアントサイドトークン化）
      - サーバーサイドではトークンのみ使用

  cardholder_data_environment:
    in_scope_systems: []  # トークン化によりスコープ外
    network_segmentation: N/A
    encryption: PSP側で実施

  compliance_requirements:
    saq_type: SAQ-A
    validation:
      - 年次自己評価
      - 四半期ASVスキャン

  payment_flow:
    1. フロントエンド: Stripe Elements表示
    2. カード情報: Stripeサーバーに直接送信
    3. トークン: TripTripサーバーに返却
    4. 決済処理: トークンでStripe API呼び出し
    5. 結果: TripTripに返却

  stored_data:
    allowed:
      - Stripeカスタマー ID
      - 支払いメソッドID（トークン）
      - カード下4桁（表示用）
      - カードブランド
      - 有効期限（月/年）
    prohibited:
      - フルカード番号（PAN）
      - CVV/CVC
      - 磁気ストライプデータ
      - PIN
```

---

## 第5章：データガバナンス

### 5.1 データオーナーシップ

#### 5.1.1 データオーナーシップモデル

```yaml
data_ownership_model:
  ownership_hierarchy:
    data_owner:
      role: ビジネス部門長
      responsibilities:
        - データ品質の最終責任
        - アクセスポリシー承認
        - データ分類の決定
        - コンプライアンス責任

    data_steward:
      role: ドメインエキスパート
      responsibilities:
        - データ定義の維持
        - 品質ルールの定義
        - メタデータ管理
        - 利用者サポート

    data_custodian:
      role: 技術チーム
      responsibilities:
        - 技術的な保護措置
        - バックアップ・リカバリ
        - アクセス制御の実装
        - モニタリング

  domain_ownership:
    user_domain:
      owner: プロダクト部門長
      steward: ユーザーチームリード
      data_assets:
        - users
        - user_profiles
        - user_preferences
        - user_consents

    booking_domain:
      owner: 予約事業部門長
      steward: 予約チームリード
      data_assets:
        - bookings
        - hotel_bookings
        - attraction_bookings
        - booking_payments

    commerce_domain:
      owner: EC事業部門長
      steward: ECチームリード
      data_assets:
        - products
        - orders
        - inventory
        - shopping_carts

    partner_domain:
      owner: パートナー事業部門長
      steward: パートナーチームリード
      data_assets:
        - partners
        - hotels
        - attractions
        - contracts

    analytics_domain:
      owner: データ部門長
      steward: データアナリティクスリード
      data_assets:
        - dwh.*
        - mart.*
        - reports.*
```

### 5.2 データカタログ

#### 5.2.1 データカタログアーキテクチャ

```yaml
data_catalog:
  platform: Google Data Catalog / Apache Atlas

  metadata_types:
    technical_metadata:
      - schema_name
      - table_name
      - column_name
      - data_type
      - constraints
      - indexes
      - partitioning
      - clustering

    business_metadata:
      - business_name
      - description
      - business_owner
      - data_steward
      - business_glossary_terms
      - data_classification
      - sensitivity_level

    operational_metadata:
      - created_date
      - last_modified_date
      - row_count
      - data_freshness
      - quality_score
      - lineage

    usage_metadata:
      - access_frequency
      - popular_queries
      - dependent_reports
      - user_ratings

  data_dictionary:
    format: YAML + Markdown
    storage: Git repository
    example: |
      tables:
        users:
          description: ユーザーマスターテーブル
          owner: user_team
          classification: CONFIDENTIAL
          columns:
            id:
              type: UUID
              description: ユーザー一意識別子
              pii: false
            email:
              type: VARCHAR(255)
              description: ユーザーメールアドレス
              pii: true
              masking_rule: partial_mask

  discovery_features:
    search:
      - フルテキスト検索
      - タグ検索
      - オーナー検索
      - 分類検索
    navigation:
      - スキーマブラウザ
      - リネージビュー
      - 関連テーブル
    recommendations:
      - 類似テーブル
      - 人気データセット
```

### 5.3 データ品質管理

#### 5.3.1 データ品質フレームワーク

```yaml
data_quality_framework:
  dimensions:
    accuracy:
      description: データが現実を正確に反映しているか
      measures:
        - 参照データとの照合率
        - 範囲外値の割合
        - ビジネスルール違反率

    completeness:
      description: 必要なデータが揃っているか
      measures:
        - NULL率
        - 必須フィールド入力率
        - レコード欠損率

    consistency:
      description: データが矛盾なく整合しているか
      measures:
        - クロステーブル整合性
        - 参照整合性
        - ビジネスルール整合性

    timeliness:
      description: データが適時に利用可能か
      measures:
        - データ鮮度
        - 処理遅延
        - SLA遵守率

    uniqueness:
      description: 重複がないか
      measures:
        - 重複レコード率
        - キー一意性

    validity:
      description: データが定義された形式に従っているか
      measures:
        - フォーマット準拠率
        - ドメイン値準拠率

  quality_rules:
    critical:
      - rule: booking.total_amount > 0
        severity: CRITICAL
        action: BLOCK

      - rule: user.email MATCHES email_pattern
        severity: CRITICAL
        action: BLOCK

      - rule: order.user_id EXISTS IN users.id
        severity: CRITICAL
        action: BLOCK

    high:
      - rule: user.phone MATCHES phone_pattern
        severity: HIGH
        action: WARN

      - rule: booking.booking_date >= booking.created_at
        severity: HIGH
        action: WARN

    medium:
      - rule: user.preferred_language IN valid_languages
        severity: MEDIUM
        action: LOG

  monitoring:
    automated_checks:
      frequency: hourly
      tools: Great Expectations, dbt tests

    quality_scores:
      calculation: weighted_average(dimension_scores)
      thresholds:
        excellent: ">= 95%"
        good: ">= 90%"
        acceptable: ">= 85%"
        poor: "< 85%"

    alerting:
      critical: PagerDuty
      high: Slack #data-quality
      medium: Daily report
```

---

## 第6章：監査 & モニタリング

### 6.1 監査ログ

#### 6.1.1 監査ログアーキテクチャ

```yaml
audit_logging:
  log_categories:
    authentication:
      events:
        - login_success
        - login_failure
        - logout
        - password_change
        - mfa_enrollment
        - mfa_verification
      retention: 2年

    authorization:
      events:
        - permission_granted
        - permission_denied
        - role_assigned
        - role_revoked
      retention: 2年

    data_access:
      events:
        - pii_read
        - pii_write
        - bulk_export
        - sensitive_query
      retention: 7年

    data_modification:
      events:
        - record_create
        - record_update
        - record_delete
        - bulk_operation
      retention: 7年

    admin_actions:
      events:
        - config_change
        - user_management
        - system_setting
        - security_setting
      retention: 7年

    consent:
      events:
        - consent_granted
        - consent_withdrawn
        - data_export_requested
        - deletion_requested
      retention: 7年

  log_schema:
    fields:
      - name: event_id
        type: UUID
        description: イベント一意識別子

      - name: event_type
        type: STRING
        description: イベントタイプ

      - name: event_category
        type: STRING
        description: イベントカテゴリ

      - name: timestamp
        type: TIMESTAMPTZ
        description: イベント発生時刻（UTC）

      - name: actor_id
        type: STRING
        description: アクション実行者ID

      - name: actor_type
        type: STRING
        description: user, system, api_client

      - name: actor_ip
        type: STRING
        description: クライアントIPアドレス

      - name: actor_user_agent
        type: STRING
        description: ユーザーエージェント

      - name: resource_type
        type: STRING
        description: 対象リソースタイプ

      - name: resource_id
        type: STRING
        description: 対象リソースID

      - name: action
        type: STRING
        description: 実行アクション

      - name: result
        type: STRING
        description: success, failure, denied

      - name: details
        type: JSONB
        description: 追加詳細情報

      - name: request_id
        type: STRING
        description: リクエストトレースID

  storage:
    primary: BigQuery
    streaming: Cloud Logging
    archive: Cloud Storage (Coldline)

  security:
    encryption: AES-256
    integrity: SHA-256 hash chain
    access: 監査チームのみ
    immutability: append-only
```

#### 6.1.2 監査ログ実装

```typescript
// 監査ログサービス
interface AuditEvent {
  eventType: string;
  eventCategory: 'authentication' | 'authorization' | 'data_access' | 'data_modification' | 'admin' | 'consent';
  actor: {
    id: string;
    type: 'user' | 'system' | 'api_client';
    ip?: string;
    userAgent?: string;
  };
  resource: {
    type: string;
    id: string;
  };
  action: string;
  result: 'success' | 'failure' | 'denied';
  details?: Record<string, any>;
  requestId: string;
}

class AuditLogService {
  private readonly bigQueryClient: BigQuery;
  private readonly dataset = 'audit_logs';
  private readonly table = 'events';

  async log(event: AuditEvent): Promise<void> {
    const auditRecord = {
      event_id: crypto.randomUUID(),
      event_type: event.eventType,
      event_category: event.eventCategory,
      timestamp: new Date().toISOString(),
      actor_id: event.actor.id,
      actor_type: event.actor.type,
      actor_ip: this.hashIP(event.actor.ip),
      actor_user_agent: event.actor.userAgent,
      resource_type: event.resource.type,
      resource_id: event.resource.id,
      action: event.action,
      result: event.result,
      details: this.sanitizeDetails(event.details),
      request_id: event.requestId,
    };

    // 即時書き込み（Streaming Insert）
    await this.bigQueryClient
      .dataset(this.dataset)
      .table(this.table)
      .insert([auditRecord]);
  }

  private hashIP(ip?: string): string | null {
    if (!ip) return null;
    // IPアドレスの部分ハッシュ（プライバシー保護）
    return crypto.createHash('sha256').update(ip + process.env.SALT).digest('hex').substring(0, 16);
  }

  private sanitizeDetails(details?: Record<string, any>): Record<string, any> | null {
    if (!details) return null;
    // PIIの除去
    const sanitized = { ...details };
    const piiFields = ['email', 'phone', 'password', 'credit_card'];
    for (const field of piiFields) {
      if (sanitized[field]) {
        sanitized[field] = '[REDACTED]';
      }
    }
    return sanitized;
  }
}

// PIIアクセス監査ミドルウェア
const piiAccessAudit = (piiFields: string[]) => {
  return async (ctx: Context, next: Next) => {
    const response = await next();

    // レスポンスにPIIが含まれる場合は監査ログ
    const responseData = ctx.response.body;
    const accessedPII = piiFields.filter(field =>
      responseData && typeof responseData === 'object' && field in responseData
    );

    if (accessedPII.length > 0) {
      await auditLogService.log({
        eventType: 'pii_access',
        eventCategory: 'data_access',
        actor: {
          id: ctx.state.user?.id || 'anonymous',
          type: 'user',
          ip: ctx.request.ip,
          userAgent: ctx.request.headers['user-agent'],
        },
        resource: {
          type: ctx.request.url.pathname.split('/')[2],
          id: ctx.request.params.id || 'multiple',
        },
        action: 'read',
        result: 'success',
        details: { accessed_fields: accessedPII },
        requestId: ctx.state.requestId,
      });
    }

    return response;
  };
};
```

### 6.2 異常検知

#### 6.2.1 異常検知システム

```yaml
anomaly_detection:
  detection_rules:
    authentication_anomalies:
      - name: brute_force_detection
        description: ブルートフォース攻撃検知
        condition: |
          login_failures >= 5
          AND time_window = 5 minutes
          AND same_ip = true
        action: block_ip, alert

      - name: impossible_travel
        description: 不可能な移動検知
        condition: |
          distance(previous_login_location, current_login_location)
          / time_since_previous_login > max_travel_speed
        action: require_mfa, alert

      - name: unusual_login_time
        description: 異常な時間帯のログイン
        condition: |
          login_time NOT IN user_typical_hours
          AND deviation > 2_std_dev
        action: require_mfa

    data_access_anomalies:
      - name: bulk_data_access
        description: 大量データアクセス検知
        condition: |
          records_accessed > threshold
          AND time_window = 1 hour
        threshold:
          normal_user: 1000
          analyst: 100000
        action: alert, require_justification

      - name: sensitive_data_access_spike
        description: 機密データアクセス急増
        condition: |
          pii_access_count > baseline * 3
          AND time_window = 1 hour
        action: alert

      - name: off_hours_data_access
        description: 業務時間外のデータアクセス
        condition: |
          access_time NOT IN business_hours
          AND data_classification IN ['CONFIDENTIAL', 'RESTRICTED']
        action: alert, log

    data_exfiltration:
      - name: large_export
        description: 大量データエクスポート検知
        condition: |
          export_size > 10MB
          OR record_count > 10000
        action: require_approval, alert

      - name: unusual_query_pattern
        description: 異常なクエリパターン
        condition: |
          query CONTAINS sensitive_fields
          AND query_type = 'SELECT *'
          AND no_where_clause = true
        action: block, alert

  ml_based_detection:
    user_behavior_analytics:
      model: Isolation Forest
      features:
        - login_frequency
        - data_access_patterns
        - query_complexity
        - time_of_day
        - resource_types_accessed
      training: 30日間の正常行動
      threshold: anomaly_score > 0.8

    data_access_profiling:
      model: LSTM Autoencoder
      features:
        - access_sequence
        - data_volume
        - time_patterns
      retraining: 週次
```

### 6.3 インシデント対応

#### 6.3.1 インシデント対応プロセス

```yaml
incident_response:
  severity_levels:
    critical:
      description: 大規模データ侵害、サービス全停止
      response_time: 15分
      escalation: 即座にCISO、CEO
      examples:
        - 顧客データの外部流出
        - ランサムウェア攻撃
        - 決済システム侵害

    high:
      description: 限定的データ侵害、主要機能影響
      response_time: 1時間
      escalation: セキュリティリード、CTO
      examples:
        - 特定ユーザーデータへの不正アクセス
        - 認証システムの脆弱性悪用
        - 内部者による不正アクセス

    medium:
      description: セキュリティ違反、軽微な影響
      response_time: 4時間
      escalation: セキュリティチーム
      examples:
        - 不審なアクセスパターン
        - ポリシー違反
        - 設定ミス

    low:
      description: セキュリティ懸念、即時影響なし
      response_time: 24時間
      escalation: セキュリティチーム
      examples:
        - 脆弱性報告
        - ログ異常
        - コンプライアンス懸念

  response_phases:
    1_detection:
      activities:
        - 自動アラート受信
        - 初期トリアージ
        - 重大度判定
      tools:
        - SIEM (Security Hub)
        - CloudWatch Alarms
        - PagerDuty

    2_containment:
      activities:
        - 被害拡大防止
        - 影響範囲特定
        - 証拠保全
      actions:
        - 侵害アカウント無効化
        - ネットワーク分離
        - アクセスキーローテーション

    3_eradication:
      activities:
        - 脅威の除去
        - 脆弱性修正
        - システムクリーンアップ
      actions:
        - マルウェア除去
        - パッチ適用
        - 設定修正

    4_recovery:
      activities:
        - サービス復旧
        - データ復元
        - 監視強化
      verification:
        - セキュリティスキャン
        - 機能テスト
        - パフォーマンス確認

    5_lessons_learned:
      activities:
        - インシデント分析
        - プロセス改善
        - ドキュメント更新
      deliverables:
        - インシデントレポート
        - 改善提案
        - トレーニング更新

  communication:
    internal:
      channels:
        - Slack #security-incidents
        - PagerDuty
        - Email (security-team)
      escalation_matrix: 別途定義

    external:
      regulatory:
        gdpr: 72時間以内に監督機関
        ccpa: 合理的期間内に消費者
        appi: 個人情報保護委員会
      customers:
        timeline: 影響確認後速やかに
        channels: メール、アプリ内通知
```

---

## 第7章：戦略的示唆 & 推奨事項

### 7.1 実装ロードマップ

```yaml
implementation_roadmap:
  phase_1_foundation:
    timeline: 月1-3
    focus: 基盤セキュリティ
    deliverables:
      - 暗号化実装（保存時、転送時）
      - 基本アクセス制御
      - 監査ログ基盤
      - 同意管理v1
    success_criteria:
      - すべてのデータ保存時暗号化
      - TLS 1.3全面適用
      - 監査ログ100%カバレッジ

  phase_2_compliance:
    timeline: 月4-6
    focus: 規制対応
    deliverables:
      - GDPR対応完了
      - 日本個人情報保護法対応
      - PCI-DSS SAQ-A取得
      - データ主体権利実装
    success_criteria:
      - コンプライアンス監査合格
      - DSR処理自動化

  phase_3_governance:
    timeline: 月7-9
    focus: ガバナンス強化
    deliverables:
      - データカタログ構築
      - 品質管理フレームワーク
      - 異常検知システム
      - インシデント対応プロセス
    success_criteria:
      - データカバレッジ > 90%
      - 品質スコア > 90%

  phase_4_advanced:
    timeline: 月10-12
    focus: 高度なセキュリティ
    deliverables:
      - ML異常検知
      - ゼロトラストアーキテクチャ
      - 継続的コンプライアンスモニタリング
    success_criteria:
      - 誤検知率 < 5%
      - 自動化率 > 70%
```

### 7.2 組織・人材推奨

```yaml
organizational_recommendations:
  required_roles:
    dpo:
      title: Data Protection Officer
      responsibilities:
        - GDPR/APPI遵守監督
        - 監督機関対応
        - DPIA実施
      reporting: CEO直轄

    security_lead:
      title: Security Lead
      responsibilities:
        - セキュリティアーキテクチャ
        - インシデント対応
        - セキュリティ監査
      team_size: 2-3名

    data_governance_lead:
      title: Data Governance Lead
      responsibilities:
        - データカタログ管理
        - 品質管理
        - メタデータ管理
      team_size: 2名

  training_program:
    all_employees:
      - セキュリティ意識向上（年次）
      - プライバシー基礎（入社時）
      - フィッシング対策（四半期）

    developers:
      - セキュアコーディング
      - OWASP Top 10
      - データ保護ベストプラクティス

    data_handlers:
      - データ分類
      - PII取り扱い
      - インシデント報告
```

---

## 第8章：実装ロードマップ

### 8.1 詳細実装計画

```yaml
detailed_implementation:
  month_1:
    focus: 暗号化基盤
    tasks:
      - KMS設定
      - データベース暗号化有効化
      - TLS証明書設定
      - S3暗号化ポリシー
    milestones:
      - すべてのデータ暗号化完了

  month_2:
    focus: アクセス制御
    tasks:
      - IAMロール設計
      - RBAC実装
      - 監査ログ設定
    milestones:
      - アクセス制御稼働

  month_3:
    focus: 同意管理
    tasks:
      - 同意スキーマ設計
      - 同意API実装
      - UIコンポーネント
    milestones:
      - 同意管理v1リリース

  month_4_6:
    focus: 規制対応
    tasks:
      - GDPR機能実装
      - DSRポータル
      - プライバシーポリシー更新
      - DPIA実施
    milestones:
      - コンプライアンス監査準備完了

  month_7_12:
    focus: ガバナンス・高度化
    tasks:
      - データカタログ構築
      - 品質管理自動化
      - 異常検知ML
      - 継続的改善
    milestones:
      - フルガバナンス稼働
```

---

## 第9章：文書間参照 & 依存関係

### 9.1 関連文書マトリックス

```yaml
document_references:
  inputs:
    - document: Doc-DA-001
      title: Data Architecture & Schema Design
      relevance: 暗号化対象スキーマ

    - document: Doc-DA-002
      title: Database Strategy
      relevance: データベースセキュリティ設定

    - document: Doc-DA-003
      title: Data Pipeline Architecture
      relevance: パイプラインセキュリティ

    - document: Doc-DA-004
      title: Analytics & BI Architecture
      relevance: 分析データのアクセス制御

    - document: Doc-IA-001
      title: Infrastructure Architecture
      relevance: ネットワークセキュリティ

  outputs:
    - document: Doc-SC-001
      title: Security Architecture
      dependency: 詳細セキュリティ設計

    - document: Doc-OP-002
      title: Compliance Runbook
      dependency: コンプライアンス運用
```

### 9.2 技術スタックサマリー

```yaml
technology_summary:
  encryption:
    key_management: AWS KMS, GCP Cloud KMS
    database: PostgreSQL TDE, BigQuery CMEK
    storage: S3 SSE-KMS

  access_control:
    identity: AWS SSO + Okta
    authorization: IAM + RBAC
    service_mesh: Istio mTLS

  monitoring:
    siem: AWS Security Hub
    logging: CloudWatch, BigQuery
    alerting: PagerDuty

  compliance:
    consent: Custom implementation
    dsr: Custom portal
    privacy: OneTrust (検討中)
```

---

## 付録

### A. データ分類チェックリスト

| カテゴリ | 分類レベル | 暗号化 | アクセス制御 | 監査 | 保持期間 |
|---------|-----------|--------|-------------|------|---------|
| メールアドレス | Confidential | Field-level | Strict RBAC | Required | Account + 30d |
| パスワードハッシュ | Restricted | Always | Need-to-know | Required | Account |
| 予約履歴 | Confidential | At-rest | Role-based | Required | 7年 |
| 決済トークン | Restricted | Always | Need-to-know | Required | Transaction + 7年 |
| 行動ログ | Confidential | At-rest | Role-based | Sampled | 180日 |
| 集計データ | Internal | At-rest | Basic | Not required | Indefinite |

### B. 変更履歴

| バージョン | 日付 | 変更内容 | 作成者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-01-20 | 初版作成 | Technical Architecture Agent |

---

**文書情報**
- 文書ID: Doc-DA-005
- バージョン: 1.0.0
- ステータス: ドラフト
- 最終更新: 2026-01-20
- 次回レビュー: 2026-02-20

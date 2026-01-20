# Doc-SC-001: TripTripセキュリティアーキテクチャ & 戦略

## エグゼクティブサマリー

本文書は、TripTripプラットフォームの包括的なセキュリティアーキテクチャと戦略を定義します。ゼロトラストセキュリティモデルを基盤とし、多層防御（Defense in Depth）アーキテクチャ、脅威モデリング（STRIDE/DREAD）、セキュリティガバナンスフレームワーク、セキュリティ監視・インシデント対応、脆弱性管理を詳述します。Google、Amazon、Netflix、Uberレベルのセキュリティ基準を採用し、1億MAU規模のグローバルプラットフォームに対応可能な、堅牢でスケーラブルなセキュリティ基盤を構築します。本設計は、Doc-DA-005（データセキュリティ、プライバシー＆ガバナンス）と連携し、既存のFlutter/Node.js/PostgreSQLスタックを基盤として発展させます。

---

## 第1章：はじめに & コンテキスト

### 1.1 目的 & スコープ

#### 1.1.1 文書の目的

```yaml
document_purpose:
  primary_objectives:
    - TripTripプラットフォーム全体のセキュリティアーキテクチャ定義
    - ゼロトラストセキュリティモデルの実装戦略
    - 多層防御アーキテクチャの設計
    - 脅威モデリングとリスク管理フレームワークの確立
    - セキュリティ運用とインシデント対応プロセスの定義

  scope:
    in_scope:
      - アプリケーションセキュリティ
      - インフラストラクチャセキュリティ
      - ネットワークセキュリティ
      - クラウドセキュリティ（AWS/GCP）
      - コンテナセキュリティ（Kubernetes/Docker）
      - セキュリティ監視・検知
      - インシデント対応
      - 脆弱性管理

    out_of_scope:
      - 物理セキュリティ（クラウドプロバイダー責任）
      - 従業員セキュリティトレーニング（別文書）
      - コンプライアンス詳細（Doc-SC-004参照）
      - ID・アクセス管理詳細（Doc-SC-002参照）
      - データ保護・暗号化詳細（Doc-SC-003参照）

  target_audience:
    - CISO/セキュリティリーダー
    - セキュリティエンジニア
    - インフラエンジニア
    - 開発チーム
    - 運用チーム
```

#### 1.1.2 既存システムとの整合性

```yaml
existing_system_alignment:
  current_architecture:
    frontend:
      platform: Flutter 3.8.1+
      authentication: JWT/OAuth2
      local_storage: Hive, SharedPreferences

    backend:
      runtime: Node.js 18+
      framework: Hono
      authentication: JWT (jsonwebtoken 9.0.2)
      password_hashing: Bcrypt 5.1.1
      security_headers: Helmet 7.1.0
      cors: CORS 2.8.5

    database:
      primary: PostgreSQL 16
      orm: Prisma 5.8.0

    infrastructure:
      cloud: AWS (Primary), GCP (Analytics)
      container: Docker, Kubernetes (EKS)
      cdn: CloudFront

  security_enhancement_approach:
    principle: 既存実装を活用し、段階的に強化
    phases:
      phase_1: 既存認証・認可の強化
      phase_2: ネットワークセキュリティ層の追加
      phase_3: 高度なセキュリティ機能の導入
```

### 1.2 セキュリティ目標 & 成功基準

#### 1.2.1 セキュリティ目標

```yaml
security_objectives:
  confidentiality:
    description: 機密データの保護
    targets:
      - 顧客PII（個人識別情報）の保護
      - 決済情報の安全な処理
      - 知的財産の保護
      - パートナーデータの機密性維持
    metrics:
      - データ漏洩インシデント: 0件/年
      - 不正アクセス検知率: >99%

  integrity:
    description: データとシステムの完全性
    targets:
      - トランザクションデータの改ざん防止
      - システム設定の不正変更検知
      - コードの完全性保証
    metrics:
      - データ改ざんインシデント: 0件/年
      - 設定ドリフト検知時間: <15分

  availability:
    description: サービスの可用性
    targets:
      - DDoS攻撃からの保護
      - サービス停止時間の最小化
      - 災害復旧能力
    metrics:
      - サービス可用性: >99.95%
      - DDoS緩和時間: <30秒
      - RTO: <15分

  accountability:
    description: 追跡可能性と説明責任
    targets:
      - すべてのアクセスの監査
      - セキュリティイベントのトレーサビリティ
      - コンプライアンス監査対応
    metrics:
      - 監査ログカバレッジ: 100%
      - ログ保持期間: >7年（規制対応）
```

#### 1.2.2 成功基準（KPI）

```yaml
security_kpis:
  vulnerability_management:
    critical_vulnerability_remediation: <24時間
    high_vulnerability_remediation: <7日
    medium_vulnerability_remediation: <30日
    low_vulnerability_remediation: <90日
    vulnerability_scan_coverage: 100%

  incident_response:
    mean_time_to_detect: <15分
    mean_time_to_respond: <1時間
    mean_time_to_recover: <4時間
    incident_closure_rate: >95%/月

  security_posture:
    security_score: >90/100
    compliance_score: >95/100
    patch_compliance: >99%
    security_training_completion: 100%

  penetration_testing:
    annual_tests: ≥2回
    critical_findings: 0件（本番移行前）
    remediation_rate: 100%
```

### 1.3 脅威ランドスケープ概要

#### 1.3.1 業界特有の脅威

```yaml
threat_landscape:
  travel_industry_threats:
    data_breaches:
      description: 旅行業界は大量のPIIを扱うため標的になりやすい
      risk_level: Critical
      examples:
        - 顧客予約情報の漏洩
        - パスポート情報の窃取
        - 決済カード情報の侵害
      mitigations:
        - データ暗号化（保存時・転送時）
        - アクセス制御の厳格化
        - データ最小化原則

    payment_fraud:
      description: オンライン決済を狙った詐欺
      risk_level: High
      examples:
        - カード詐欺
        - 不正予約
        - チャージバック詐欺
      mitigations:
        - 3Dセキュア実装
        - 不正検知システム
        - 取引監視

    account_takeover:
      description: ユーザーアカウントの乗っ取り
      risk_level: High
      examples:
        - クレデンシャルスタッフィング
        - フィッシング
        - セッションハイジャック
      mitigations:
        - MFA強制
        - 異常検知
        - セッション管理強化

    api_abuse:
      description: APIの不正利用
      risk_level: Medium
      examples:
        - スクレイピング
        - 価格操作
        - 在庫枯渇攻撃
      mitigations:
        - レート制限
        - APIキー管理
        - ボット検知

  general_cyber_threats:
    ransomware:
      risk_level: Critical
      mitigations:
        - バックアップ戦略
        - ネットワークセグメンテーション
        - EDR/XDR

    supply_chain_attacks:
      risk_level: High
      mitigations:
        - 依存関係スキャン
        - SBOMの維持
        - ベンダーリスク評価

    insider_threats:
      risk_level: Medium
      mitigations:
        - 最小権限原則
        - アクセス監視
        - 退職者アクセス即時無効化
```

#### 1.3.2 脅威インテリジェンス

```yaml
threat_intelligence:
  sources:
    commercial:
      - CrowdStrike
      - Recorded Future
      - Mandiant
    open_source:
      - MITRE ATT&CK
      - AlienVault OTX
      - VirusTotal
    industry_specific:
      - FS-ISAC
      - Travel Industry ISAC
      - JPCERT/CC

  integration:
    siem_feeds:
      - IP reputation feeds
      - Domain reputation feeds
      - Malware hash feeds
    automated_blocking:
      - Known malicious IPs
      - C2 domains
      - Phishing URLs

  threat_hunting:
    frequency: 週次
    focus_areas:
      - APT indicators
      - Lateral movement patterns
      - Data exfiltration attempts
```

---

## 第2章：ゼロトラストセキュリティモデル

### 2.1 ゼロトラスト原則

#### 2.1.1 ゼロトラストアーキテクチャ概要

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         ZERO TRUST ARCHITECTURE                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      Trust Boundary = NONE                               │    │
│  │                                                                          │    │
│  │   "Never Trust, Always Verify" - すべてのアクセスを検証                  │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                 │
│  │    IDENTITY     │  │    DEVICE       │  │    CONTEXT      │                 │
│  │    PILLAR       │  │    PILLAR       │  │    PILLAR       │                 │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤                 │
│  │ • User Identity │  │ • Device Health │  │ • Time/Location │                 │
│  │ • Service ID    │  │ • Posture Check │  │ • Risk Score    │                 │
│  │ • MFA           │  │ • Compliance    │  │ • Behavior      │                 │
│  │ • SSO           │  │ • Inventory     │  │ • Threat Intel  │                 │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘                 │
│           │                    │                    │                          │
│           └────────────────────┼────────────────────┘                          │
│                                ▼                                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      POLICY DECISION POINT (PDP)                        │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │    │
│  │   │ Policy       │  │ Risk         │  │ Access       │                 │    │
│  │   │ Evaluation   │  │ Assessment   │  │ Decision     │                 │    │
│  │   └──────────────┘  └──────────────┘  └──────────────┘                 │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                │                                                │
│                                ▼                                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      POLICY ENFORCEMENT POINTS (PEP)                    │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │    │
│  │   │ API Gateway  │  │ Service Mesh │  │ Network      │  │ Data      │  │    │
│  │   │ (Kong/Envoy) │  │ (Istio)      │  │ (Firewall)   │  │ Access    │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────┘  └───────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                │                                                │
│                                ▼                                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      PROTECTED RESOURCES                                │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │    │
│  │   │ APIs         │  │ Databases    │  │ Storage      │  │ Services  │  │    │
│  │   │              │  │              │  │              │  │           │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────┘  └───────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 2.1.2 ゼロトラスト原則詳細

```yaml
zero_trust_principles:
  principle_1_verify_explicitly:
    description: すべてのアクセス要求を明示的に検証
    implementation:
      - すべてのAPIリクエストで認証を要求
      - トークンの有効性を毎回検証
      - デバイス・コンテキスト情報を考慮
    triptrip_application:
      user_requests:
        - JWTトークン検証
        - セッション有効性確認
        - デバイスフィンガープリント照合
      service_requests:
        - mTLS証明書検証
        - サービスアカウント認証
        - リクエスト署名検証

  principle_2_least_privilege:
    description: 最小権限でのアクセス付与
    implementation:
      - Just-In-Time（JIT）アクセス
      - Just-Enough-Access（JEA）
      - 時間制限付きアクセス
    triptrip_application:
      user_access:
        - ロールベースの最小権限
        - リソースレベルの権限
        - 機能別アクセス制御
      service_access:
        - サービス別IAMロール
        - ネームスペース分離
        - Network Policy

  principle_3_assume_breach:
    description: 侵害を前提とした設計
    implementation:
      - マイクロセグメンテーション
      - 横方向移動の制限
      - 継続的な監視と検知
    triptrip_application:
      network:
        - サービス間mTLS
        - Network Policy（デフォルト拒否）
        - マイクロセグメンテーション
      detection:
        - 異常行動検知
        - 横方向移動検知
        - データ漏洩検知
```

### 2.2 マイクロセグメンテーション

#### 2.2.1 ネットワークセグメンテーション設計

```yaml
microsegmentation:
  network_zones:
    dmz:
      description: 外部公開ゾーン
      components:
        - ALB/CloudFront
        - WAF
        - DDoS Protection
      allowed_traffic:
        inbound:
          - HTTPS (443) from Internet
        outbound:
          - To Application Zone only

    application_zone:
      description: アプリケーション実行ゾーン
      components:
        - API Services
        - Web Services
        - Background Workers
      allowed_traffic:
        inbound:
          - From DMZ only
          - From same zone (service mesh)
        outbound:
          - To Data Zone
          - To External Services (via egress)

    data_zone:
      description: データストレージゾーン
      components:
        - PostgreSQL
        - Redis
        - S3/Storage
      allowed_traffic:
        inbound:
          - From Application Zone only
        outbound:
          - None (isolated)

    management_zone:
      description: 管理・運用ゾーン
      components:
        - CI/CD
        - Monitoring
        - Logging
        - Bastion
      allowed_traffic:
        inbound:
          - From VPN only
        outbound:
          - To all zones (monitoring)

  kubernetes_segmentation:
    namespaces:
      triptrip-api:
        network_policies:
          - default-deny-all
          - allow-ingress-from-istio
          - allow-egress-to-database
          - allow-egress-to-cache
          - allow-egress-to-external-https

      triptrip-web:
        network_policies:
          - default-deny-all
          - allow-ingress-from-istio
          - allow-egress-to-api

      triptrip-workers:
        network_policies:
          - default-deny-all
          - allow-egress-to-database
          - allow-egress-to-queue

      triptrip-cache:
        network_policies:
          - default-deny-all
          - allow-ingress-from-app-namespaces

      triptrip-db:
        network_policies:
          - default-deny-all
          - allow-ingress-from-app-namespaces
```

#### 2.2.2 Kubernetesネットワークポリシー実装

```yaml
# Default Deny All Policy
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
# Allow Ingress from Istio Gateway
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-from-istio
  namespace: triptrip-api
spec:
  podSelector:
    matchLabels:
      app: triptrip-api
  policyTypes:
    - Ingress
  ingress:
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

---
# Allow Egress to PostgreSQL
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-database
  namespace: triptrip-api
spec:
  podSelector:
    matchLabels:
      app: triptrip-api
  policyTypes:
    - Egress
  egress:
    # DNS
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
    # RDS (VPC Endpoint経由)
    - to:
        - ipBlock:
            cidr: 10.0.20.0/24  # Data subnet CIDR
      ports:
        - protocol: TCP
          port: 5432
```

### 2.3 継続的検証

#### 2.3.1 継続的認証・認可

```yaml
continuous_verification:
  authentication:
    token_validation:
      jwt_verification:
        - 署名検証（RS256）
        - 有効期限チェック
        - 発行者検証
        - オーディエンス検証
      frequency: 毎リクエスト

    session_management:
      session_validity:
        mobile_app: 30日（リフレッシュトークン）
        web_app: 24時間（アクセストークン）
        admin_portal: 8時間
      re_authentication_triggers:
        - 機密操作（決済、個人情報変更）
        - 長時間の非アクティブ
        - デバイス変更検知
        - 異常な場所からのアクセス

  authorization:
    policy_evaluation:
      frequency: 毎リクエスト
      factors:
        - ユーザーロール
        - リソースオーナーシップ
        - 時間帯制限
        - IPアドレス/地理情報
        - デバイス信頼度

  device_posture:
    checks:
      mobile:
        - OS バージョン
        - アプリバージョン
        - Jailbreak/Root検知
        - デバイス整合性
      web:
        - ブラウザバージョン
        - セキュリティヘッダーサポート
        - Cookie設定

  risk_based_access:
    risk_factors:
      - ログイン場所の変化
      - 時間帯の異常
      - デバイスの変更
      - 行動パターンの変化
      - 過去の不正試行
    risk_actions:
      low_risk: 通常アクセス
      medium_risk: MFA要求
      high_risk: アクセスブロック + アラート
```

#### 2.3.2 継続的検証実装

```typescript
// 継続的検証ミドルウェア
import { Context, Next } from 'hono';
import { verify } from 'jsonwebtoken';

interface RiskAssessment {
  score: number;
  factors: string[];
  action: 'allow' | 'mfa_required' | 'block';
}

class ContinuousVerificationMiddleware {
  private riskEngine: RiskEngine;
  private deviceTrustService: DeviceTrustService;
  private geoIpService: GeoIpService;

  async verify(ctx: Context, next: Next): Promise<Response | void> {
    const token = this.extractToken(ctx);

    // 1. トークン検証
    const tokenPayload = await this.verifyToken(token);
    if (!tokenPayload) {
      return ctx.json({ error: 'Invalid token' }, 401);
    }

    // 2. セッション有効性確認
    const session = await this.getSession(tokenPayload.sessionId);
    if (!session || session.isRevoked) {
      return ctx.json({ error: 'Session expired or revoked' }, 401);
    }

    // 3. デバイス信頼度チェック
    const deviceFingerprint = ctx.req.header('X-Device-Fingerprint');
    const deviceTrust = await this.deviceTrustService.evaluate(
      tokenPayload.userId,
      deviceFingerprint
    );

    // 4. コンテキスト情報収集
    const context = {
      ipAddress: ctx.req.header('X-Forwarded-For') || ctx.req.ip,
      userAgent: ctx.req.header('User-Agent'),
      timestamp: new Date(),
      endpoint: ctx.req.path,
      method: ctx.req.method,
      deviceTrust,
    };

    // 5. リスク評価
    const riskAssessment = await this.riskEngine.assess(
      tokenPayload.userId,
      context
    );

    // 6. リスクベースのアクション
    switch (riskAssessment.action) {
      case 'block':
        await this.auditLog('access_blocked', tokenPayload.userId, context, riskAssessment);
        return ctx.json({ error: 'Access denied due to risk assessment' }, 403);

      case 'mfa_required':
        if (!ctx.req.header('X-MFA-Token')) {
          return ctx.json({
            error: 'MFA required',
            mfa_challenge: await this.generateMfaChallenge(tokenPayload.userId)
          }, 403);
        }
        // MFAトークン検証
        const mfaValid = await this.verifyMfaToken(
          tokenPayload.userId,
          ctx.req.header('X-MFA-Token')
        );
        if (!mfaValid) {
          return ctx.json({ error: 'Invalid MFA token' }, 403);
        }
        break;

      case 'allow':
        // 正常アクセス
        break;
    }

    // 7. コンテキスト情報をリクエストに付加
    ctx.set('user', {
      id: tokenPayload.userId,
      roles: tokenPayload.roles,
      sessionId: tokenPayload.sessionId,
      riskScore: riskAssessment.score,
    });

    // 8. 監査ログ
    await this.auditLog('access_granted', tokenPayload.userId, context, riskAssessment);

    return next();
  }

  private async verifyToken(token: string): Promise<TokenPayload | null> {
    try {
      const payload = verify(token, process.env.JWT_PUBLIC_KEY!, {
        algorithms: ['RS256'],
        issuer: 'triptrip-auth',
        audience: 'triptrip-api',
      }) as TokenPayload;

      // トークンブラックリストチェック
      const isBlacklisted = await this.checkTokenBlacklist(payload.jti);
      if (isBlacklisted) {
        return null;
      }

      return payload;
    } catch (error) {
      return null;
    }
  }
}

// リスク評価エンジン
class RiskEngine {
  private readonly RISK_THRESHOLDS = {
    low: 30,
    medium: 60,
    high: 80,
  };

  async assess(userId: string, context: RequestContext): Promise<RiskAssessment> {
    const factors: string[] = [];
    let score = 0;

    // 1. 地理的リスク
    const geoRisk = await this.assessGeoRisk(userId, context.ipAddress);
    score += geoRisk.score;
    if (geoRisk.score > 0) factors.push(geoRisk.reason);

    // 2. 時間帯リスク
    const timeRisk = this.assessTimeRisk(userId, context.timestamp);
    score += timeRisk.score;
    if (timeRisk.score > 0) factors.push(timeRisk.reason);

    // 3. デバイスリスク
    const deviceRisk = this.assessDeviceRisk(context.deviceTrust);
    score += deviceRisk.score;
    if (deviceRisk.score > 0) factors.push(deviceRisk.reason);

    // 4. 行動リスク
    const behaviorRisk = await this.assessBehaviorRisk(userId, context);
    score += behaviorRisk.score;
    if (behaviorRisk.score > 0) factors.push(behaviorRisk.reason);

    // 5. 脅威インテリジェンスリスク
    const threatRisk = await this.assessThreatIntelligence(context.ipAddress);
    score += threatRisk.score;
    if (threatRisk.score > 0) factors.push(threatRisk.reason);

    // アクション決定
    let action: RiskAssessment['action'];
    if (score >= this.RISK_THRESHOLDS.high) {
      action = 'block';
    } else if (score >= this.RISK_THRESHOLDS.medium) {
      action = 'mfa_required';
    } else {
      action = 'allow';
    }

    return { score, factors, action };
  }

  private async assessGeoRisk(userId: string, ipAddress: string): Promise<RiskFactor> {
    const currentLocation = await this.geoIpService.lookup(ipAddress);
    const lastKnownLocation = await this.getUserLastLocation(userId);

    if (!lastKnownLocation) {
      return { score: 10, reason: 'First login from this location' };
    }

    const distance = this.calculateDistance(currentLocation, lastKnownLocation.location);
    const timeSinceLastLogin = Date.now() - lastKnownLocation.timestamp;
    const impossibleTravel = distance / (timeSinceLastLogin / 3600000) > 1000; // 1000km/h

    if (impossibleTravel) {
      return { score: 50, reason: 'Impossible travel detected' };
    }

    if (currentLocation.country !== lastKnownLocation.location.country) {
      return { score: 20, reason: 'Login from different country' };
    }

    return { score: 0, reason: '' };
  }
}
```

---

## 第3章：多層防御アーキテクチャ

### 3.1 ネットワーク層セキュリティ

#### 3.1.1 ネットワークセキュリティアーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    NETWORK SECURITY ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  Internet                                                                        │
│      │                                                                          │
│      ▼                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  Layer 1: Edge Protection                                                │    │
│  │                                                                          │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │    │
│  │  │  CloudFront  │  │  AWS Shield  │  │  Route 53    │                 │    │
│  │  │  CDN + WAF   │  │  DDoS        │  │  DNS Shield  │                 │    │
│  │  │              │  │  Advanced    │  │              │                 │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘                 │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                    │                                             │
│                                    ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  Layer 2: Perimeter Security                                            │    │
│  │                                                                          │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │    │
│  │  │  AWS WAF     │  │  API Gateway │  │  ALB         │                 │    │
│  │  │  Rules       │  │  Rate Limit  │  │  TLS 1.3     │                 │    │
│  │  │              │  │  Auth        │  │  終端        │                 │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘                 │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                    │                                             │
│                                    ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  Layer 3: Network Segmentation (VPC)                                    │    │
│  │                                                                          │    │
│  │  ┌──────────────────────────────────────────────────────────────────┐   │    │
│  │  │  Public Subnet          │  Private Subnet      │  Data Subnet   │   │    │
│  │  │  • NAT Gateway          │  • EKS Nodes         │  • RDS         │   │    │
│  │  │  • ALB                  │  • App Pods          │  • ElastiCache │   │    │
│  │  │                         │                       │                │   │    │
│  │  │  NACL: Strict           │  NACL: Moderate      │  NACL: Strict  │   │    │
│  │  │  SG: ALB Only           │  SG: App Traffic     │  SG: DB Only   │   │    │
│  │  └──────────────────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                    │                                             │
│                                    ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  Layer 4: Service Mesh Security (Istio)                                 │    │
│  │                                                                          │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │    │
│  │  │  mTLS        │  │  AuthPolicy  │  │  Network     │                 │    │
│  │  │  Encryption  │  │  RBAC        │  │  Policy      │                 │    │
│  │  │              │  │              │  │              │                 │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘                 │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 3.1.2 WAF（Web Application Firewall）設定

```yaml
waf_configuration:
  aws_waf:
    web_acl:
      name: triptrip-waf-acl
      default_action: ALLOW

      rules:
        # AWS管理ルール
        - name: AWSManagedRulesCommonRuleSet
          priority: 1
          managed_rule_group:
            vendor: AWS
            name: AWSManagedRulesCommonRuleSet
          override_action: none

        - name: AWSManagedRulesKnownBadInputsRuleSet
          priority: 2
          managed_rule_group:
            vendor: AWS
            name: AWSManagedRulesKnownBadInputsRuleSet
          override_action: none

        - name: AWSManagedRulesSQLiRuleSet
          priority: 3
          managed_rule_group:
            vendor: AWS
            name: AWSManagedRulesSQLiRuleSet
          override_action: none

        - name: AWSManagedRulesLinuxRuleSet
          priority: 4
          managed_rule_group:
            vendor: AWS
            name: AWSManagedRulesLinuxRuleSet
          override_action: none

        # カスタムルール - レート制限
        - name: RateLimitRule
          priority: 5
          statement:
            rate_based:
              limit: 2000
              aggregate_key_type: IP
          action: BLOCK

        # カスタムルール - 地理的ブロック
        - name: GeoBlockRule
          priority: 6
          statement:
            geo_match:
              country_codes:
                - KP  # 北朝鮮
                - IR  # イラン
                - SY  # シリア
                - CU  # キューバ
          action: BLOCK

        # カスタムルール - ボット保護
        - name: BotProtectionRule
          priority: 7
          statement:
            and:
              - not:
                  byte_match:
                    field_to_match:
                      headers:
                        name: User-Agent
                    positional_constraint: CONTAINS
                    search_string: "TripTrip"
              - rate_based:
                  limit: 100
                  aggregate_key_type: IP
          action: BLOCK

        # カスタムルール - パスベース保護
        - name: AdminPathProtection
          priority: 8
          statement:
            and:
              - byte_match:
                  field_to_match:
                    uri_path: {}
                  positional_constraint: STARTS_WITH
                  search_string: "/admin"
              - not:
                  ip_set_reference:
                    arn: "arn:aws:wafv2:...:ipset/admin-allowed-ips"
          action: BLOCK

    logging:
      enabled: true
      destination:
        type: S3
        bucket: triptrip-waf-logs
        prefix: waf-logs/
      redacted_fields:
        - authorization
        - cookie

    metrics:
      cloudwatch_enabled: true
      sample_requests_enabled: true
```

#### 3.1.3 DDoS保護

```yaml
ddos_protection:
  aws_shield:
    tier: Advanced

    protected_resources:
      - CloudFront distributions
      - Application Load Balancers
      - Route 53 hosted zones
      - Elastic IPs

    response_team:
      enabled: true
      proactive_engagement: true

    cost_protection:
      enabled: true

  cloudfront_settings:
    origin_shield:
      enabled: true
      region: ap-northeast-1

    cache_policy:
      default_ttl: 86400
      max_ttl: 31536000

    geo_restriction:
      type: none

  rate_limiting:
    global:
      requests_per_second: 10000
      burst: 20000

    per_ip:
      requests_per_second: 100
      burst: 200

    per_api:
      authentication:
        requests_per_minute: 10
      booking:
        requests_per_minute: 30
      search:
        requests_per_minute: 60
```

### 3.2 アプリケーション層セキュリティ

#### 3.2.1 セキュアコーディング基準

```yaml
secure_coding_standards:
  owasp_top_10_mitigations:
    A01_broken_access_control:
      mitigations:
        - すべてのエンドポイントで認可チェック
        - リソースレベルのアクセス制御
        - デフォルト拒否ポリシー
      implementation:
        - 認可ミドルウェアの必須適用
        - オーナーシップ検証
        - 水平権限昇格防止

    A02_cryptographic_failures:
      mitigations:
        - 強力な暗号アルゴリズムの使用
        - 適切な鍵管理
        - TLS 1.3の強制
      implementation:
        - AES-256-GCMによる暗号化
        - AWS KMSによる鍵管理
        - 弱い暗号の無効化

    A03_injection:
      mitigations:
        - パラメータ化クエリの使用
        - 入力検証
        - 出力エンコーディング
      implementation:
        - Prisma ORMの使用（SQLインジェクション防止）
        - Zodによる入力バリデーション
        - HTMLエスケープ

    A04_insecure_design:
      mitigations:
        - 脅威モデリングの実施
        - セキュアデザインパターンの採用
        - セキュリティテストの自動化
      implementation:
        - STRIDE分析
        - セキュリティアーキテクチャレビュー
        - SAST/DASTの統合

    A05_security_misconfiguration:
      mitigations:
        - セキュアデフォルト設定
        - 不要機能の無効化
        - 設定の自動検証
      implementation:
        - Infrastructure as Code
        - セキュリティベースラインの強制
        - 設定ドリフト検知

    A06_vulnerable_components:
      mitigations:
        - 依存関係の継続的監視
        - 脆弱性スキャン
        - 迅速なパッチ適用
      implementation:
        - Dependabot/Snyk
        - SBOMの維持
        - 自動アップデート

    A07_authentication_failures:
      mitigations:
        - 強力な認証メカニズム
        - MFAの実装
        - セキュアなセッション管理
      implementation:
        - OAuth2/OIDC
        - TOTP/WebAuthn
        - セキュアなトークン管理

    A08_software_data_integrity:
      mitigations:
        - コード署名
        - 完全性チェック
        - セキュアなCI/CD
      implementation:
        - イメージ署名
        - チェックサム検証
        - 署名付きコミット

    A09_security_logging_failures:
      mitigations:
        - 包括的なログ記録
        - ログの保護
        - 監視とアラート
      implementation:
        - 構造化ログ
        - ログの暗号化
        - SIEM統合

    A10_ssrf:
      mitigations:
        - URL検証
        - 許可リスト
        - ネットワーク分離
      implementation:
        - URLパース＆検証
        - 外部リクエストの制限
        - メタデータアクセスブロック
```

#### 3.2.2 入力検証実装

```typescript
// 入力検証スキーマ（Zod）
import { z } from 'zod';

// 共通バリデーションルール
const commonValidators = {
  // メールアドレス
  email: z.string()
    .email('Invalid email format')
    .max(255)
    .transform(val => val.toLowerCase().trim()),

  // パスワード
  password: z.string()
    .min(12, 'Password must be at least 12 characters')
    .max(128)
    .regex(/[A-Z]/, 'Password must contain uppercase letter')
    .regex(/[a-z]/, 'Password must contain lowercase letter')
    .regex(/[0-9]/, 'Password must contain number')
    .regex(/[^A-Za-z0-9]/, 'Password must contain special character'),

  // UUID
  uuid: z.string()
    .uuid('Invalid UUID format'),

  // 日付
  date: z.string()
    .datetime()
    .or(z.date()),

  // 金額
  amount: z.number()
    .positive('Amount must be positive')
    .multipleOf(0.01)
    .max(10000000, 'Amount exceeds maximum'),

  // URL
  url: z.string()
    .url()
    .refine(
      (url) => {
        const parsed = new URL(url);
        return ['https:'].includes(parsed.protocol);
      },
      'Only HTTPS URLs are allowed'
    ),

  // 電話番号（日本）
  phoneJapan: z.string()
    .regex(/^0[0-9]{9,10}$/, 'Invalid Japanese phone number'),

  // 郵便番号（日本）
  postalCodeJapan: z.string()
    .regex(/^[0-9]{3}-?[0-9]{4}$/, 'Invalid Japanese postal code'),
};

// 予約作成スキーマ
const createBookingSchema = z.object({
  hotelId: commonValidators.uuid,
  roomId: commonValidators.uuid,
  checkInDate: z.string().datetime(),
  checkOutDate: z.string().datetime(),
  guestCount: z.number().int().min(1).max(10),
  specialRequests: z.string().max(1000).optional(),
  paymentMethod: z.enum(['credit_card', 'debit_card', 'bank_transfer']),
}).refine(
  (data) => new Date(data.checkOutDate) > new Date(data.checkInDate),
  'Check-out date must be after check-in date'
);

// ユーザー登録スキーマ
const registerUserSchema = z.object({
  email: commonValidators.email,
  password: commonValidators.password,
  name: z.string()
    .min(1)
    .max(100)
    .regex(/^[a-zA-Z\u3040-\u309F\u30A0-\u30FF\u4E00-\u9FAF\s]+$/, 'Invalid characters in name'),
  phoneNumber: commonValidators.phoneJapan.optional(),
  dateOfBirth: z.string().datetime().optional(),
  acceptedTerms: z.literal(true, {
    errorMap: () => ({ message: 'You must accept the terms' }),
  }),
});

// バリデーションミドルウェア
function validateRequest<T extends z.ZodSchema>(schema: T) {
  return async (ctx: Context, next: Next) => {
    try {
      const body = await ctx.req.json();
      const validated = schema.parse(body);
      ctx.set('validatedBody', validated);
      return next();
    } catch (error) {
      if (error instanceof z.ZodError) {
        return ctx.json({
          error: 'Validation failed',
          details: error.errors.map(e => ({
            field: e.path.join('.'),
            message: e.message,
          })),
        }, 400);
      }
      throw error;
    }
  };
}

// SQLインジェクション防止（Prismaの使用）
// 悪い例（使用禁止）
// const user = await prisma.$queryRaw`SELECT * FROM users WHERE email = ${email}`;

// 良い例（Prismaのパラメータ化クエリ）
const getUserByEmail = async (email: string) => {
  return prisma.user.findUnique({
    where: { email },
    select: {
      id: true,
      email: true,
      name: true,
      // passwordHashは含めない
    },
  });
};

// XSS防止（出力エスケープ）
import { escape } from 'html-escaper';

const sanitizeOutput = (data: Record<string, unknown>): Record<string, unknown> => {
  const sanitized: Record<string, unknown> = {};

  for (const [key, value] of Object.entries(data)) {
    if (typeof value === 'string') {
      sanitized[key] = escape(value);
    } else if (typeof value === 'object' && value !== null) {
      sanitized[key] = sanitizeOutput(value as Record<string, unknown>);
    } else {
      sanitized[key] = value;
    }
  }

  return sanitized;
};
```

### 3.3 データ層セキュリティ

#### 3.3.1 データベースセキュリティ

```yaml
database_security:
  postgresql_hardening:
    authentication:
      method: scram-sha-256
      ssl: required
      connection_limit_per_user: 100

    authorization:
      principle: least_privilege
      roles:
        app_read:
          permissions:
            - SELECT on application tables
          usage: Read-only operations
        app_write:
          permissions:
            - SELECT, INSERT, UPDATE on application tables
          usage: Write operations
        app_delete:
          permissions:
            - SELECT, INSERT, UPDATE, DELETE on application tables
          usage: Delete operations (restricted)
        admin:
          permissions:
            - ALL PRIVILEGES
          usage: Admin only, MFA required

    encryption:
      at_rest:
        enabled: true
        method: AES-256
        key_management: AWS KMS
      in_transit:
        ssl_mode: verify-full
        ssl_ca: RDS CA certificate

    auditing:
      pgaudit:
        enabled: true
        log_level: write
        log_catalog: false
        log_parameter: true
        log_statement_once: true

    network:
      vpc_only: true
      security_group: db-sg
      allowed_sources:
        - app-subnet-cidr

  redis_security:
    authentication:
      auth_token: enabled
      acl: enabled

    encryption:
      at_rest: true
      in_transit: true

    network:
      vpc_endpoint: true
      security_group: cache-sg

  backup_security:
    encryption: AES-256 (KMS)
    retention: 35 days
    cross_region: ap-northeast-3
    access: restricted to backup admins
```

#### 3.3.2 データベースアクセス制御実装

```sql
-- PostgreSQL ロール設定
-- アプリケーション読み取りロール
CREATE ROLE app_read WITH LOGIN PASSWORD 'secure_password_from_secrets_manager';
GRANT CONNECT ON DATABASE triptrip TO app_read;
GRANT USAGE ON SCHEMA public TO app_read;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_read;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO app_read;

-- アプリケーション書き込みロール
CREATE ROLE app_write WITH LOGIN PASSWORD 'secure_password_from_secrets_manager';
GRANT CONNECT ON DATABASE triptrip TO app_write;
GRANT USAGE ON SCHEMA public TO app_write;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO app_write;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_write;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE ON TABLES TO app_write;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT USAGE, SELECT ON SEQUENCES TO app_write;

-- 削除は特定テーブルのみ許可
REVOKE DELETE ON users, bookings, payments FROM app_write;

-- 監査ロール（読み取り専用）
CREATE ROLE auditor WITH LOGIN PASSWORD 'secure_password_from_secrets_manager';
GRANT CONNECT ON DATABASE triptrip TO auditor;
GRANT USAGE ON SCHEMA public TO auditor;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO auditor;
-- 機密カラムは除外
REVOKE SELECT (password_hash, api_key) ON users FROM auditor;

-- Row Level Security (RLS) の設定
ALTER TABLE bookings ENABLE ROW LEVEL SECURITY;

-- ユーザーは自分の予約のみ閲覧可能
CREATE POLICY user_booking_isolation ON bookings
  FOR ALL
  TO app_read, app_write
  USING (user_id = current_setting('app.current_user_id')::uuid);

-- パートナーは自社の予約のみ閲覧可能
CREATE POLICY partner_booking_isolation ON bookings
  FOR ALL
  TO partner_read
  USING (hotel_id IN (
    SELECT hotel_id FROM partner_hotels
    WHERE partner_id = current_setting('app.current_partner_id')::uuid
  ));

-- pgAudit設定
CREATE EXTENSION IF NOT EXISTS pgaudit;

ALTER SYSTEM SET pgaudit.log = 'write, ddl';
ALTER SYSTEM SET pgaudit.log_catalog = 'off';
ALTER SYSTEM SET pgaudit.log_parameter = 'on';
ALTER SYSTEM SET pgaudit.log_statement_once = 'on';
SELECT pg_reload_conf();
```

---

## 第4章：脅威モデリング & リスク管理

### 4.1 STRIDE分析

#### 4.1.1 脅威カテゴリ別分析

```yaml
stride_analysis:
  spoofing:
    description: 他者になりすまし
    threats:
      - threat: アカウント乗っ取り
        likelihood: High
        impact: Critical
        attack_vectors:
          - クレデンシャルスタッフィング
          - フィッシング
          - セッションハイジャック
        mitigations:
          - MFA強制
          - パスワードポリシー強化
          - 異常ログイン検知
          - セッショントークンのセキュア化
        residual_risk: Medium

      - threat: サービス偽装
        likelihood: Medium
        impact: High
        attack_vectors:
          - 証明書偽造
          - DNS偽装
        mitigations:
          - mTLS
          - 証明書ピンニング（モバイル）
          - DNSSEC
        residual_risk: Low

  tampering:
    description: データの改ざん
    threats:
      - threat: トランザクション改ざん
        likelihood: Medium
        impact: Critical
        attack_vectors:
          - パラメータ改ざん
          - リプレイ攻撃
          - MITM攻撃
        mitigations:
          - 入力検証
          - リクエスト署名
          - TLS強制
          - べき等性チェック
        residual_risk: Low

      - threat: データベース改ざん
        likelihood: Low
        impact: Critical
        attack_vectors:
          - SQLインジェクション
          - 不正な管理者アクセス
        mitigations:
          - パラメータ化クエリ（Prisma）
          - 特権アクセス管理
          - 監査ログ
          - データベース暗号化
        residual_risk: Low

  repudiation:
    description: 否認
    threats:
      - threat: トランザクション否認
        likelihood: Medium
        impact: High
        attack_vectors:
          - ログ改ざん
          - 監査証跡の欠如
        mitigations:
          - 不変監査ログ
          - デジタル署名
          - タイムスタンプ
          - ログの暗号化
        residual_risk: Low

  information_disclosure:
    description: 情報漏洩
    threats:
      - threat: 顧客データ漏洩
        likelihood: Medium
        impact: Critical
        attack_vectors:
          - データベース侵害
          - API脆弱性
          - 内部不正
        mitigations:
          - データ暗号化
          - アクセス制御
          - DLP
          - 監視・検知
        residual_risk: Medium

      - threat: 決済情報漏洩
        likelihood: Low
        impact: Critical
        attack_vectors:
          - カードスキミング
          - メモリスクレイピング
        mitigations:
          - PCI-DSS準拠
          - トークナイゼーション（Stripe）
          - カードデータ非保持
        residual_risk: Very Low

  denial_of_service:
    description: サービス拒否
    threats:
      - threat: DDoS攻撃
        likelihood: High
        impact: High
        attack_vectors:
          - ボリューメトリック攻撃
          - アプリケーション層攻撃
        mitigations:
          - AWS Shield Advanced
          - CloudFront
          - WAF
          - オートスケーリング
        residual_risk: Low

      - threat: リソース枯渇
        likelihood: Medium
        impact: Medium
        attack_vectors:
          - スローロリス
          - リソース消費攻撃
        mitigations:
          - レート制限
          - タイムアウト設定
          - リソースクォータ
        residual_risk: Low

  elevation_of_privilege:
    description: 権限昇格
    threats:
      - threat: 水平権限昇格
        likelihood: Medium
        impact: High
        attack_vectors:
          - IDOR
          - パラメータ改ざん
        mitigations:
          - オーナーシップ検証
          - UUID使用
          - アクセス制御チェック
        residual_risk: Low

      - threat: 垂直権限昇格
        likelihood: Low
        impact: Critical
        attack_vectors:
          - ロール操作
          - 管理者機能の悪用
        mitigations:
          - ロールベースアクセス制御
          - 管理者MFA
          - 特権アクセス監視
        residual_risk: Very Low
```

### 4.2 リスク評価マトリックス

#### 4.2.1 DREAD分析

```yaml
dread_analysis:
  risk_matrix:
    damage_potential:
      description: 潜在的な被害の大きさ
      scale:
        1: 最小限の影響
        5: 軽微な影響
        10: 重大な影響

    reproducibility:
      description: 再現の容易さ
      scale:
        1: 再現困難
        5: 条件付きで再現可能
        10: 常に再現可能

    exploitability:
      description: 攻撃の容易さ
      scale:
        1: 高度な専門知識が必要
        5: ツールが必要
        10: スクリプトキディでも可能

    affected_users:
      description: 影響を受けるユーザー数
      scale:
        1: 極少数のユーザー
        5: 一部のユーザー
        10: 全ユーザー

    discoverability:
      description: 脆弱性の発見容易性
      scale:
        1: 発見困難
        5: 特定の知識があれば発見可能
        10: 容易に発見可能

  top_risks:
    - risk: 顧客データ漏洩
      dread_score:
        damage: 10
        reproducibility: 5
        exploitability: 5
        affected_users: 10
        discoverability: 5
      total: 35
      priority: Critical

    - risk: 決済詐欺
      dread_score:
        damage: 9
        reproducibility: 7
        exploitability: 5
        affected_users: 5
        discoverability: 6
      total: 32
      priority: Critical

    - risk: アカウント乗っ取り
      dread_score:
        damage: 8
        reproducibility: 8
        exploitability: 6
        affected_users: 7
        discoverability: 7
      total: 36
      priority: Critical

    - risk: DDoS攻撃
      dread_score:
        damage: 7
        reproducibility: 9
        exploitability: 8
        affected_users: 10
        discoverability: 10
      total: 44
      priority: High

  risk_treatment:
    accept:
      criteria: total_score < 15
      review_frequency: 年次

    mitigate:
      criteria: total_score >= 15
      implementation: セキュリティコントロールの実装

    transfer:
      criteria: 保険適用可能なリスク
      implementation: サイバー保険

    avoid:
      criteria: リスクが許容できない
      implementation: 機能/サービスの停止
```

### 4.3 緩和戦略

#### 4.3.1 リスク別緩和策

```yaml
mitigation_strategies:
  customer_data_breach:
    risk_level: Critical
    controls:
      preventive:
        - データ暗号化（AES-256）
        - アクセス制御（RBAC/ABAC）
        - ネットワークセグメンテーション
        - 入力検証

      detective:
        - データアクセス監視
        - 異常検知
        - DLP
        - 侵入検知（IDS）

      corrective:
        - インシデント対応プロセス
        - バックアップからの復旧
        - 通知手順

    implementation_priority: Immediate
    owner: Security Team

  payment_fraud:
    risk_level: Critical
    controls:
      preventive:
        - 3Dセキュア2.0
        - トークナイゼーション
        - 不正検知ルール
        - 取引限度額

      detective:
        - リアルタイム取引監視
        - 機械学習ベースの不正検知
        - パターン分析

      corrective:
        - 自動取引ブロック
        - チャージバック対応
        - 顧客通知

    implementation_priority: High
    owner: Payments Team

  account_takeover:
    risk_level: Critical
    controls:
      preventive:
        - MFA強制
        - パスワードポリシー
        - クレデンシャルスタッフィング保護
        - セッション管理

      detective:
        - 異常ログイン検知
        - 地理的異常検知
        - デバイスフィンガープリント

      corrective:
        - アカウントロック
        - パスワードリセット強制
        - ユーザー通知

    implementation_priority: High
    owner: Identity Team
```

---

## 第5章：セキュリティ運用

### 5.1 監視 & 検知

#### 5.1.1 セキュリティ監視アーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    SECURITY MONITORING ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      Data Sources                                        │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │    │
│  │   │ Application  │  │ Infrastructure│  │ Cloud        │  │ Network   │  │    │
│  │   │ Logs         │  │ Logs          │  │ Trail        │  │ Flows     │  │    │
│  │   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └─────┬─────┘  │    │
│  │          │                 │                  │                │        │    │
│  └──────────┼─────────────────┼──────────────────┼────────────────┼────────┘    │
│             │                 │                  │                │             │
│             └─────────────────┼──────────────────┼────────────────┘             │
│                               ▼                  ▼                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      Log Aggregation                                     │    │
│  │                                                                          │    │
│  │   ┌──────────────────────────────────────────────────────────────────┐  │    │
│  │   │  Fluent Bit → CloudWatch Logs → S3 (Long-term) → Athena         │  │    │
│  │   │                    ↓                                              │  │    │
│  │   │              OpenSearch (Analysis)                                │  │    │
│  │   └──────────────────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                               │                                                 │
│                               ▼                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      SIEM (AWS Security Hub)                            │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │    │
│  │   │ GuardDuty    │  │ Inspector    │  │ Macie        │  │ IAM Access│  │    │
│  │   │ Threat       │  │ Vulnera-     │  │ Data         │  │ Analyzer  │  │    │
│  │   │ Detection    │  │ bility       │  │ Discovery    │  │           │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────┘  └───────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                               │                                                 │
│                               ▼                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      Alerting & Response                                │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │    │
│  │   │ PagerDuty    │  │ Slack        │  │ Email        │  │ Auto      │  │    │
│  │   │ (Critical)   │  │ (High)       │  │ (Medium)     │  │ Remediate │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────┘  └───────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 5.1.2 検知ルール

```yaml
detection_rules:
  authentication:
    - name: brute_force_login
      description: ログイン試行の異常な失敗
      condition: |
        login_failures >= 5
        AND time_window = 5 minutes
        AND same_user_or_ip = true
      severity: High
      action:
        - アカウント一時ロック
        - セキュリティチームにアラート
        - IPブロック（閾値超過時）

    - name: impossible_travel
      description: 物理的に不可能な移動
      condition: |
        distance(prev_login.location, current_login.location) /
        time_diff(prev_login.time, current_login.time) > 1000 km/h
      severity: Critical
      action:
        - セッション無効化
        - MFA要求
        - ユーザー通知

    - name: credential_stuffing
      description: 大量の認証試行
      condition: |
        unique_usernames >= 100
        AND same_ip = true
        AND time_window = 10 minutes
      severity: Critical
      action:
        - IPブロック
        - CAPTCHA強制
        - セキュリティチームにエスカレーション

  data_access:
    - name: bulk_data_export
      description: 大量データの取得
      condition: |
        records_accessed > 1000
        AND time_window = 1 hour
        AND user_type != 'admin'
      severity: High
      action:
        - アクセスログ
        - セキュリティチームにアラート
        - 正当性確認

    - name: sensitive_data_access
      description: 機密データへのアクセス
      condition: |
        data_classification IN ['CONFIDENTIAL', 'RESTRICTED']
        AND access_time NOT IN business_hours
      severity: Medium
      action:
        - ログ強化
        - 週次レビュー

    - name: unusual_query_pattern
      description: 異常なクエリパターン
      condition: |
        query CONTAINS 'SELECT *'
        AND table IN sensitive_tables
        AND no_where_clause = true
      severity: High
      action:
        - クエリブロック
        - 即時アラート

  network:
    - name: port_scan
      description: ポートスキャン検知
      condition: |
        connection_attempts >= 20
        AND unique_ports >= 10
        AND time_window = 1 minute
      severity: Medium
      action:
        - IPブロック
        - ログ記録

    - name: lateral_movement
      description: 横方向移動の試行
      condition: |
        internal_connections >= 10
        AND unique_destinations >= 5
        AND time_window = 5 minutes
        AND source != known_scanner
      severity: Critical
      action:
        - ネットワーク分離
        - インシデント対応開始
        - フォレンジック保全

  application:
    - name: sql_injection_attempt
      description: SQLインジェクション試行
      condition: |
        request CONTAINS sql_injection_patterns
      severity: High
      action:
        - リクエストブロック
        - IPブロック（繰り返し時）
        - ログ記録

    - name: api_abuse
      description: API乱用
      condition: |
        requests_per_minute > rate_limit * 2
        AND same_api_key = true
      severity: Medium
      action:
        - APIキー一時停止
        - ユーザー通知
```

### 5.2 インシデント対応

#### 5.2.1 インシデント対応プロセス

```yaml
incident_response_process:
  phases:
    1_preparation:
      activities:
        - インシデント対応チームの編成
        - 対応手順書の整備
        - ツールと環境の準備
        - 定期的な訓練

      team_structure:
        incident_commander:
          responsibility: 全体指揮、意思決定
          escalation: CISO/CTO
        technical_lead:
          responsibility: 技術対応の指揮
          escalation: Infrastructure Lead
        communications_lead:
          responsibility: 社内外コミュニケーション
          escalation: PR/Legal
        scribe:
          responsibility: 記録、タイムライン管理

      tools:
        - SIEM (Security Hub)
        - チケットシステム (Jira)
        - コミュニケーション (Slack, PagerDuty)
        - フォレンジック (AWS Detective)
        - バックアップ/リストア

    2_identification:
      activities:
        - アラートの受信と初期トリアージ
        - インシデントの確認
        - 重大度の判定
        - 影響範囲の特定

      severity_classification:
        critical:
          criteria:
            - 大規模データ漏洩
            - サービス全停止
            - ランサムウェア感染
            - 決済システム侵害
          response_time: 15分
          notification: 即時（CISO, CEO）

        high:
          criteria:
            - 限定的データ漏洩
            - 部分的サービス影響
            - 認証システム侵害
          response_time: 1時間
          notification: 1時間以内（セキュリティリード）

        medium:
          criteria:
            - ポリシー違反
            - 不審なアクティビティ
            - 軽微な脆弱性悪用
          response_time: 4時間
          notification: 4時間以内（セキュリティチーム）

        low:
          criteria:
            - 情報収集
            - 低リスクイベント
          response_time: 24時間
          notification: 翌営業日

    3_containment:
      strategies:
        short_term:
          - 影響システムの隔離
          - 侵害アカウントの無効化
          - ネットワーク接続の遮断
          - 悪意のあるプロセスの停止

        long_term:
          - パッチ適用
          - 設定変更
          - アクセス制御の強化
          - 監視強化

      evidence_preservation:
        - メモリダンプ
        - ディスクイメージ
        - ネットワークログ
        - アプリケーションログ
        - タイムスタンプ記録

    4_eradication:
      activities:
        - マルウェアの除去
        - バックドアの特定と削除
        - 脆弱性の修正
        - 設定の修正
        - クレデンシャルのローテーション

      verification:
        - 脆弱性スキャン
        - 完全性チェック
        - ログレビュー

    5_recovery:
      activities:
        - システムの段階的復旧
        - データのリストア（必要時）
        - サービスの再開
        - 監視の強化

      validation:
        - 機能テスト
        - セキュリティテスト
        - パフォーマンステスト
        - ユーザー確認

    6_lessons_learned:
      activities:
        - ポストモーテム会議
        - 根本原因分析
        - 改善提案の作成
        - 手順書の更新
        - 再発防止策の実装

      deliverables:
        - インシデントレポート
        - 改善アクションアイテム
        - タイムライン更新
        - トレーニング更新
```

#### 5.2.2 インシデント対応ランブック

```yaml
incident_runbooks:
  data_breach:
    name: データ漏洩対応
    severity: Critical
    steps:
      - step: 1
        action: 初期トリアージ
        details:
          - アラートの詳細確認
          - 漏洩データの種類特定
          - 影響範囲の初期評価
        owner: On-call Engineer
        time_limit: 15 minutes

      - step: 2
        action: エスカレーション
        details:
          - インシデントコマンダーの招集
          - ステークホルダーへの初期通知
          - 法務/PR連絡（必要時）
        owner: Incident Commander
        time_limit: 30 minutes

      - step: 3
        action: 封じ込め
        details:
          - 影響システムの隔離
          - アクセスの遮断
          - 追加漏洩の防止
        owner: Technical Lead
        time_limit: 1 hour

      - step: 4
        action: 証拠保全
        details:
          - ログの保全
          - メモリダンプ（可能な場合）
          - チェーン・オブ・カストディ記録
        owner: Forensics Team
        time_limit: 2 hours

      - step: 5
        action: 影響評価
        details:
          - 漏洩データの詳細特定
          - 影響を受けるユーザー数の確定
          - 規制報告要件の確認
        owner: Data Protection Officer
        time_limit: 4 hours

      - step: 6
        action: 通知
        details:
          - 規制当局への報告（72時間以内、GDPR）
          - 影響を受けるユーザーへの通知
          - プレスリリース（必要時）
        owner: Communications Lead
        time_limit: 72 hours

      - step: 7
        action: 復旧
        details:
          - 脆弱性の修正
          - システムの復旧
          - 監視強化
        owner: Technical Lead
        time_limit: varies

      - step: 8
        action: ポストモーテム
        details:
          - 根本原因分析
          - 改善策の特定
          - レポート作成
        owner: Incident Commander
        time_limit: 1 week

  ransomware:
    name: ランサムウェア対応
    severity: Critical
    steps:
      - step: 1
        action: 即時隔離
        details:
          - 感染システムのネットワーク切断
          - 他システムへの拡散防止
        owner: On-call Engineer
        time_limit: 5 minutes

      - step: 2
        action: 拡散範囲の特定
        details:
          - 感染システムの特定
          - ネットワークトラフィック分析
          - 横方向移動の検証
        owner: Security Team
        time_limit: 1 hour

      - step: 3
        action: バックアップ確認
        details:
          - バックアップの整合性確認
          - 復旧ポイントの特定
          - バックアップの隔離保護
        owner: Infrastructure Team
        time_limit: 2 hours

      - step: 4
        action: 復旧判断
        details:
          - 復旧オプションの評価
          - ビジネスインパクト分析
          - 復旧計画の策定
        owner: Incident Commander
        time_limit: 4 hours
```

### 5.3 脆弱性管理

#### 5.3.1 脆弱性管理プログラム

```yaml
vulnerability_management:
  scanning:
    infrastructure:
      tool: AWS Inspector
      frequency: 継続的
      scope:
        - EC2インスタンス
        - EKSノード
        - ECRイメージ
        - Lambda関数

    application:
      sast:
        tool: SonarQube, Snyk
        frequency: 毎コミット（CI/CD）
        languages:
          - TypeScript
          - Dart
          - SQL
      dast:
        tool: OWASP ZAP, Burp Suite
        frequency: 週次（staging）、リリース前

    dependencies:
      tool: Snyk, Dependabot
      frequency: 日次
      scope:
        - npm packages
        - Flutter packages
        - Docker base images

    configuration:
      tool: AWS Config, Security Hub
      frequency: 継続的
      standards:
        - CIS AWS Foundations
        - AWS Well-Architected Security

  prioritization:
    cvss_score:
      critical: 9.0-10.0
      high: 7.0-8.9
      medium: 4.0-6.9
      low: 0.1-3.9

    risk_factors:
      - 攻撃の容易さ
      - 公開エクスプロイトの有無
      - 影響を受けるシステムの重要度
      - 補完的コントロールの有無

  remediation_sla:
    critical:
      timeframe: 24時間
      escalation: 即時
      tracking: 日次レビュー

    high:
      timeframe: 7日
      escalation: 3日後
      tracking: 週次レビュー

    medium:
      timeframe: 30日
      escalation: 14日後
      tracking: 月次レビュー

    low:
      timeframe: 90日
      escalation: 60日後
      tracking: 四半期レビュー

  exception_process:
    criteria:
      - ビジネスクリティカルなシステム
      - 修正に大幅な変更が必要
      - 補完的コントロールが存在
    approval:
      high: セキュリティリード
      critical: CISO
    max_extension:
      high: 30日
      critical: 7日
    compensating_controls:
      required: true
      documentation: 必須
```

#### 5.3.2 ペネトレーションテスト

```yaml
penetration_testing:
  program:
    frequency:
      internal: 四半期
      external: 年2回

    scope:
      web_application:
        - API endpoints
        - Web frontend
        - Admin portal

      mobile_application:
        - Flutter iOS app
        - Flutter Android app

      infrastructure:
        - AWS infrastructure
        - Kubernetes cluster
        - Network segmentation

      social_engineering:
        - Phishing simulation
        - Physical access (optional)

  methodology:
    standards:
      - OWASP Testing Guide
      - PTES
      - NIST SP 800-115

    phases:
      reconnaissance:
        - OSINT gathering
        - DNS enumeration
        - Service discovery

      vulnerability_assessment:
        - Automated scanning
        - Manual verification
        - Configuration review

      exploitation:
        - Vulnerability exploitation
        - Privilege escalation
        - Lateral movement

      post_exploitation:
        - Data exfiltration simulation
        - Persistence mechanisms
        - Cleanup

  reporting:
    executive_summary:
      - リスクレベルの概要
      - ビジネスインパクト
      - 主要な発見事項

    technical_details:
      - 脆弱性の詳細
      - 再現手順
      - 証拠
      - 推奨修正策

    risk_rating:
      - CVSS score
      - Business impact
      - Likelihood

  remediation_tracking:
    critical_findings:
      verification: 修正後即時
      retest: 必須

    high_findings:
      verification: 1週間以内
      retest: 必須

    medium_findings:
      verification: 30日以内
      retest: サンプル

    low_findings:
      verification: 90日以内
      retest: 任意
```

---

## 第6章：実装ロードマップ & 文書間参照

### 6.1 実装ロードマップ

```yaml
implementation_roadmap:
  phase_1_foundation:
    timeline: 月1-3
    focus: 基盤セキュリティ
    deliverables:
      - ゼロトラストアーキテクチャ設計完了
      - WAF/DDoS保護の実装
      - ネットワークセグメンテーション
      - 基本的な監視・ログ基盤
      - セキュアコーディングガイドライン
    success_criteria:
      - WAFルール適用率: 100%
      - ネットワークポリシー適用率: 100%
      - 監視カバレッジ: >90%

  phase_2_detection:
    timeline: 月4-6
    focus: 検知・対応能力
    deliverables:
      - SIEM統合（Security Hub）
      - 検知ルールの実装
      - インシデント対応プロセス
      - 脆弱性管理プログラム
      - ペネトレーションテスト実施
    success_criteria:
      - 検知ルール数: >50
      - MTTD: <15分
      - 脆弱性スキャンカバレッジ: 100%

  phase_3_advanced:
    timeline: 月7-9
    focus: 高度なセキュリティ
    deliverables:
      - 継続的検証の実装
      - リスクベースアクセス制御
      - 機械学習ベースの異常検知
      - 自動化されたインシデント対応
      - セキュリティオーケストレーション
    success_criteria:
      - 自動化率: >60%
      - 誤検知率: <5%
      - MTTR: <1時間

  phase_4_optimization:
    timeline: 月10-12
    focus: 最適化・成熟
    deliverables:
      - セキュリティメトリクスダッシュボード
      - 継続的改善プロセス
      - セキュリティ訓練プログラム
      - 第三者監査対応
      - SOC2 Type II準備
    success_criteria:
      - セキュリティスコア: >90
      - コンプライアンススコア: >95%
      - 監査指摘事項: 0（Critical/High）
```

### 6.2 文書間参照

```yaml
document_references:
  inputs:
    - document: Doc-DA-005
      title: データセキュリティ、プライバシー＆ガバナンス
      relevance: データ保護要件、暗号化戦略
      key_integrations:
        - データ分類スキームの適用
        - 暗号化ポリシーとの整合
        - アクセス制御モデルの連携

    - document: Doc-IA-001
      title: クラウドインフラストラクチャ＆デプロイメント
      relevance: インフラセキュリティ要件
      key_integrations:
        - VPCセキュリティ設計
        - EKSセキュリティ設定
        - IAMロール設計

    - document: Doc-IA-002
      title: Kubernetes＆コンテナ戦略
      relevance: コンテナセキュリティ要件
      key_integrations:
        - Pod Security Standards
        - Network Policy
        - サービスメッシュセキュリティ

    - document: Doc-SA-003
      title: API設計
      relevance: API認証・認可
      key_integrations:
        - OAuth2/OIDC実装
        - APIセキュリティヘッダー
        - レート制限

  outputs:
    - document: Doc-SC-002
      title: ID＆アクセス管理
      dependency: 認証・認可の詳細設計

    - document: Doc-SC-003
      title: データ保護＆暗号化
      dependency: 暗号化実装の詳細

    - document: Doc-SC-004
      title: コンプライアンス＆規制フレームワーク
      dependency: コンプライアンス要件の詳細
```

### 6.3 技術スタックサマリー

```yaml
security_technology_stack:
  identity_and_access:
    identity_provider: AWS SSO + Okta
    authentication: OAuth2/OIDC, JWT
    mfa: TOTP, WebAuthn
    authorization: RBAC, ABAC

  network_security:
    perimeter: CloudFront, AWS WAF, Shield
    network: VPC, Security Groups, NACL
    service_mesh: Istio mTLS
    dns: Route 53, DNSSEC

  application_security:
    sast: SonarQube, Snyk
    dast: OWASP ZAP
    dependency_scan: Snyk, Dependabot
    container_scan: Trivy, ECR Scanning

  infrastructure_security:
    cspm: Security Hub, AWS Config
    vulnerability_scan: Inspector
    secrets_management: Secrets Manager, External Secrets

  monitoring_and_detection:
    siem: AWS Security Hub
    threat_detection: GuardDuty
    data_protection: Macie
    logging: CloudWatch, OpenSearch

  incident_response:
    alerting: PagerDuty, Slack
    ticketing: Jira
    forensics: AWS Detective
    automation: Lambda, Step Functions
```

---

## 付録

### A. セキュリティチェックリスト

| カテゴリ | チェック項目 | 優先度 | ステータス |
|---------|-------------|--------|----------|
| 認証 | MFA強制 | Critical | 実装予定 |
| 認証 | パスワードポリシー | High | 実装済み |
| 認可 | RBAC実装 | Critical | 実装予定 |
| 暗号化 | TLS 1.3 | Critical | 実装済み |
| 暗号化 | データ保存時暗号化 | Critical | 実装予定 |
| ネットワーク | WAF設定 | Critical | 実装予定 |
| ネットワーク | DDoS保護 | High | 実装予定 |
| 監視 | ログ集約 | High | 実装予定 |
| 監視 | 異常検知 | High | 実装予定 |
| 脆弱性 | 自動スキャン | High | 実装予定 |

### B. 変更履歴

| バージョン | 日付 | 変更内容 | 作成者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-01-20 | 初版作成 | Technical Architecture Agent |

---

**文書情報**
- 文書ID: Doc-SC-001
- バージョン: 1.0.0
- ステータス: ドラフト
- 最終更新: 2026-01-20
- 次回レビュー: 2026-02-20
- 行数: 約2,000行

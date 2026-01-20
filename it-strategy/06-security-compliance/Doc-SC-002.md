# Doc-SC-002: TripTrip ID & アクセス管理

## エグゼクティブサマリー

本文書は、TripTripプラットフォームの包括的なアイデンティティ＆アクセス管理（IAM）アーキテクチャを定義します。OAuth2/OpenID Connect（OIDC）に基づく認証アーキテクチャ、ロールベースアクセス制御（RBAC）と属性ベースアクセス制御（ABAC）を組み合わせた認可モデル、多要素認証（MFA）、サービス間認証（mTLS）、特権アクセス管理（PAM）を詳述します。Google、Amazon、Netflixレベルのセキュリティ基準を採用し、ユーザーエクスペリエンスを損なうことなく、堅牢な認証・認可基盤を構築します。本設計は、Doc-SC-001（セキュリティアーキテクチャ＆戦略）のゼロトラストセキュリティモデルを基盤とし、既存のJWT/OAuth2実装を発展させます。

---

## 第1章：はじめに & コンテキスト

### 1.1 目的 & スコープ

#### 1.1.1 文書の目的

```yaml
document_purpose:
  primary_objectives:
    - TripTripプラットフォーム全体のIAMアーキテクチャ定義
    - OAuth2/OIDC認証フローの詳細設計
    - RBAC/ABACハイブリッド認可モデルの実装
    - MFA戦略とパスワードレス認証の導入
    - サービス間認証とサービスメッシュセキュリティの設計

  scope:
    in_scope:
      - ユーザー認証（エンドユーザー、管理者、パートナー）
      - サービス認証（マイクロサービス間、外部API）
      - 認可モデル（RBAC、ABAC）
      - アイデンティティライフサイクル管理
      - シングルサインオン（SSO）
      - フェデレーション（SAML、OIDC）
      - 特権アクセス管理

    out_of_scope:
      - 暗号化詳細（Doc-SC-003参照）
      - コンプライアンス詳細（Doc-SC-004参照）
      - 全体セキュリティ戦略（Doc-SC-001参照）

  target_audience:
    - セキュリティアーキテクト
    - 開発チーム
    - インフラエンジニア
    - プロダクトマネージャー
```

#### 1.1.2 既存システムとの整合性

```yaml
existing_system_alignment:
  current_implementation:
    authentication:
      library: jsonwebtoken 9.0.2
      algorithm: RS256
      token_type: JWT
      token_storage:
        mobile: Secure Storage
        web: HttpOnly Cookie

    password_management:
      library: bcrypt 5.1.1
      salt_rounds: 12

    session_management:
      current: JWT stateless
      enhancement: Hybrid (JWT + Session Store)

    social_login:
      providers:
        - Google OAuth2
        - Apple Sign In
        - LINE Login (Japan market)

  enhancement_strategy:
    phase_1_enhance:
      - MFAの追加
      - リフレッシュトークンの改善
      - セッション管理の強化

    phase_2_extend:
      - OIDC準拠の強化
      - RBAC実装
      - サービス間認証（mTLS）

    phase_3_advanced:
      - ABAC実装
      - パスワードレス認証
      - リスクベースアクセス制御
```

### 1.2 IAM要件 & 成功基準

#### 1.2.1 要件定義

```yaml
iam_requirements:
  functional_requirements:
    authentication:
      - 複数認証方式のサポート（パスワード、OAuth、SAML）
      - MFAの実装（TOTP、SMS、WebAuthn）
      - パスワードレス認証のサポート
      - セッション管理とトークンリフレッシュ
      - アカウントリカバリーフロー

    authorization:
      - ロールベースアクセス制御（RBAC）
      - 属性ベースアクセス制御（ABAC）
      - リソースレベルの権限管理
      - 動的ポリシー評価

    identity_lifecycle:
      - ユーザープロビジョニング
      - セルフサービスポータル
      - デプロビジョニング自動化
      - アクセスレビュー

    federation:
      - 外部IdPとの連携
      - B2Bパートナー向けSSO
      - 従業員向けSSO（内部システム）

  non_functional_requirements:
    performance:
      - 認証レイテンシ: <200ms (p99)
      - トークン検証: <10ms (p99)
      - 認可判定: <5ms (p99)

    availability:
      - 認証サービス可用性: >99.99%
      - フェイルオーバー時間: <30秒

    scalability:
      - 同時認証リクエスト: 10,000/秒
      - アクティブセッション: 1,000万

    security:
      - クレデンシャル保護
      - トークンセキュリティ
      - 監査ログ完全性
```

#### 1.2.2 成功基準（KPI）

```yaml
iam_kpis:
  security_metrics:
    unauthorized_access_attempts:
      target: 0件/月（成功）
      measurement: 認証ログ分析

    mfa_adoption:
      target: ">90%（アクティブユーザー）"
      measurement: ユーザーMFA登録状況

    credential_compromise:
      target: 0件/年
      measurement: インシデント報告

  operational_metrics:
    authentication_success_rate:
      target: ">99.5%"
      measurement: 認証試行成功率

    password_reset_self_service:
      target: ">95%"
      measurement: セルフサービス完了率

    account_lockout_rate:
      target: "<0.1%/日"
      measurement: ロックアウト発生率

  user_experience_metrics:
    authentication_latency_p99:
      target: "<200ms"
      measurement: APMメトリクス

    user_friction_score:
      target: "<2（1-5スケール）"
      measurement: ユーザーサーベイ
```

### 1.3 主要な前提条件 & 制約

```yaml
assumptions_and_constraints:
  assumptions:
    - ユーザーはスマートフォンを所有（MFA用）
    - 安定したインターネット接続
    - ブラウザのCookie/LocalStorage使用可能

  constraints:
    technology:
      - 既存JWT実装との後方互換性
      - Node.js/Honoフレームワーク
      - PostgreSQLユーザーストア

    business:
      - ユーザーエクスペリエンスの維持
      - 段階的移行（ビッグバン禁止）
      - 日本市場特有の要件（LINE Login等）

    regulatory:
      - GDPR同意管理
      - 個人情報保護法
      - PCI-DSS（決済関連）
```

---

## 第2章：認証アーキテクチャ

### 2.1 OAuth2/OIDC実装

#### 2.1.1 認証アーキテクチャ概要

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      AUTHENTICATION ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐        │
│   │  Mobile App     │      │  Web App        │      │  Partner Portal │        │
│   │  (Flutter)      │      │  (Next.js)      │      │  (React)        │        │
│   └────────┬────────┘      └────────┬────────┘      └────────┬────────┘        │
│            │                        │                        │                  │
│            └────────────────────────┼────────────────────────┘                  │
│                                     │                                           │
│                                     ▼                                           │
│   ┌─────────────────────────────────────────────────────────────────────────┐  │
│   │                      API Gateway (Kong/AWS API Gateway)                  │  │
│   │                                                                          │  │
│   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │  │
│   │   │ Rate Limit   │  │ JWT Verify   │  │ Request      │                 │  │
│   │   │              │  │ (Pre-auth)   │  │ Transform    │                 │  │
│   │   └──────────────┘  └──────────────┘  └──────────────┘                 │  │
│   └─────────────────────────────────────────────────────────────────────────┘  │
│                                     │                                           │
│                                     ▼                                           │
│   ┌─────────────────────────────────────────────────────────────────────────┐  │
│   │                      Identity Provider (Auth Service)                    │  │
│   │                                                                          │  │
│   │   ┌───────────────────────────────────────────────────────────────────┐ │  │
│   │   │  OAuth2/OIDC Authorization Server                                  │ │  │
│   │   │                                                                    │ │  │
│   │   │   ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌───────────┐  │ │  │
│   │   │   │ /authorize │  │ /token     │  │ /userinfo  │  │ /revoke   │  │ │  │
│   │   │   │            │  │            │  │            │  │           │  │ │  │
│   │   │   └────────────┘  └────────────┘  └────────────┘  └───────────┘  │ │  │
│   │   └───────────────────────────────────────────────────────────────────┘ │  │
│   │                                                                          │  │
│   │   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐           │  │
│   │   │ Local Auth     │  │ Social Auth    │  │ Enterprise SSO │           │  │
│   │   │ (Password)     │  │ (Google,Apple) │  │ (SAML/OIDC)    │           │  │
│   │   └────────────────┘  └────────────────┘  └────────────────┘           │  │
│   │                                                                          │  │
│   │   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐           │  │
│   │   │ MFA Service    │  │ Session Store  │  │ Token Store    │           │  │
│   │   │ (TOTP/WebAuthn)│  │ (Redis)        │  │ (Redis)        │           │  │
│   │   └────────────────┘  └────────────────┘  └────────────────┘           │  │
│   └─────────────────────────────────────────────────────────────────────────┘  │
│                                     │                                           │
│                                     ▼                                           │
│   ┌─────────────────────────────────────────────────────────────────────────┐  │
│   │                      User Store (PostgreSQL)                             │  │
│   │                                                                          │  │
│   │   users | credentials | mfa_devices | sessions | consent_records        │  │
│   └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 2.1.2 OAuth2フロー設計

```yaml
oauth2_flows:
  authorization_code_with_pkce:
    description: モバイルアプリ、Webアプリ向け
    use_cases:
      - モバイルアプリ認証
      - SPA認証
    flow:
      1_authorization_request:
        endpoint: /oauth2/authorize
        parameters:
          - response_type: code
          - client_id: required
          - redirect_uri: required
          - scope: "openid profile email"
          - state: required (CSRF protection)
          - code_challenge: required (PKCE)
          - code_challenge_method: S256

      2_user_authentication:
        - ログインフォーム表示
        - クレデンシャル検証
        - MFA検証（必要時）
        - 同意画面表示（初回/変更時）

      3_authorization_response:
        - code: authorization_code
        - state: original_state

      4_token_request:
        endpoint: /oauth2/token
        parameters:
          - grant_type: authorization_code
          - code: authorization_code
          - redirect_uri: original_redirect_uri
          - client_id: required
          - code_verifier: required (PKCE)

      5_token_response:
        - access_token: JWT
        - refresh_token: opaque
        - id_token: JWT
        - token_type: Bearer
        - expires_in: 3600

  client_credentials:
    description: サービス間認証
    use_cases:
      - マイクロサービス間通信
      - バックエンドジョブ
    flow:
      1_token_request:
        endpoint: /oauth2/token
        parameters:
          - grant_type: client_credentials
          - client_id: required
          - client_secret: required
          - scope: required

      2_token_response:
        - access_token: JWT
        - token_type: Bearer
        - expires_in: 3600

  refresh_token:
    description: トークン更新
    security_measures:
      - Refresh Token Rotation
      - Family Detection（侵害検知）
      - Absolute Expiration
    flow:
      1_refresh_request:
        endpoint: /oauth2/token
        parameters:
          - grant_type: refresh_token
          - refresh_token: current_refresh_token
          - client_id: required

      2_refresh_response:
        - access_token: new_jwt
        - refresh_token: new_refresh_token
        - id_token: new_id_token（必要時）
```

#### 2.1.3 OIDC実装詳細

```yaml
oidc_implementation:
  discovery:
    endpoint: /.well-known/openid-configuration
    response:
      issuer: "https://auth.triptrip.com"
      authorization_endpoint: "https://auth.triptrip.com/oauth2/authorize"
      token_endpoint: "https://auth.triptrip.com/oauth2/token"
      userinfo_endpoint: "https://auth.triptrip.com/oauth2/userinfo"
      jwks_uri: "https://auth.triptrip.com/.well-known/jwks.json"
      registration_endpoint: null  # Dynamic registration disabled
      scopes_supported:
        - openid
        - profile
        - email
        - phone
        - address
      response_types_supported:
        - code
        - token
        - id_token
        - "code id_token"
      grant_types_supported:
        - authorization_code
        - refresh_token
        - client_credentials
      subject_types_supported:
        - public
      id_token_signing_alg_values_supported:
        - RS256
        - ES256
      token_endpoint_auth_methods_supported:
        - client_secret_basic
        - client_secret_post
        - private_key_jwt
      claims_supported:
        - sub
        - iss
        - aud
        - exp
        - iat
        - name
        - email
        - email_verified
        - phone_number
        - phone_number_verified
      code_challenge_methods_supported:
        - S256

  id_token:
    claims:
      standard:
        - iss: issuer identifier
        - sub: subject identifier
        - aud: audience
        - exp: expiration time
        - iat: issued at
        - auth_time: authentication time
        - nonce: nonce value
        - acr: authentication context class reference
        - amr: authentication methods references
      profile:
        - name
        - family_name
        - given_name
        - preferred_username
        - picture
        - locale
      email:
        - email
        - email_verified
    signing:
      algorithm: RS256
      key_rotation: 90日
      key_id_in_header: true

  jwks:
    endpoint: /.well-known/jwks.json
    key_management:
      active_keys: 2
      rotation_period: 90日
      key_types:
        - kty: RSA
          alg: RS256
          use: sig
          key_size: 2048
```

#### 2.1.4 認証サービス実装

```typescript
// auth-service/src/oauth2/authorization.ts

import { Context } from 'hono';
import { sign, verify } from 'jsonwebtoken';
import { createHash, randomBytes } from 'crypto';

interface AuthorizationRequest {
  response_type: string;
  client_id: string;
  redirect_uri: string;
  scope: string;
  state: string;
  code_challenge?: string;
  code_challenge_method?: string;
  nonce?: string;
  prompt?: string;
  login_hint?: string;
}

interface TokenRequest {
  grant_type: string;
  code?: string;
  redirect_uri?: string;
  client_id: string;
  client_secret?: string;
  code_verifier?: string;
  refresh_token?: string;
  scope?: string;
}

class OAuth2AuthorizationServer {
  private clientRepository: ClientRepository;
  private userRepository: UserRepository;
  private tokenRepository: TokenRepository;
  private codeRepository: AuthorizationCodeRepository;
  private sessionRepository: SessionRepository;
  private mfaService: MfaService;

  // 認可エンドポイント
  async authorize(ctx: Context): Promise<Response> {
    const request = this.parseAuthorizationRequest(ctx);

    // 1. クライアント検証
    const client = await this.clientRepository.findById(request.client_id);
    if (!client) {
      return this.errorResponse(ctx, 'invalid_client', 'Unknown client');
    }

    // 2. リダイレクトURI検証
    if (!this.validateRedirectUri(client, request.redirect_uri)) {
      return this.errorResponse(ctx, 'invalid_request', 'Invalid redirect_uri');
    }

    // 3. PKCE検証（Public Clientの場合は必須）
    if (client.type === 'public' && !request.code_challenge) {
      return this.errorRedirect(request.redirect_uri, request.state,
        'invalid_request', 'PKCE required for public clients');
    }

    // 4. セッション確認（既存認証の有無）
    const session = await this.getSession(ctx);

    // 5. 認証が必要かどうか判定
    if (!session || request.prompt === 'login') {
      // ログインページへリダイレクト
      return this.redirectToLogin(ctx, request);
    }

    // 6. MFA確認
    if (this.requiresMfa(session, client) && !session.mfaVerified) {
      return this.redirectToMfa(ctx, request, session);
    }

    // 7. 同意確認
    const consent = await this.checkConsent(session.userId, client.id, request.scope);
    if (!consent && request.prompt !== 'none') {
      return this.redirectToConsent(ctx, request, session);
    }

    // 8. 認可コード生成
    const code = await this.generateAuthorizationCode(
      session.userId,
      client.id,
      request.redirect_uri,
      request.scope,
      request.code_challenge,
      request.code_challenge_method,
      request.nonce
    );

    // 9. リダイレクト
    const redirectUrl = new URL(request.redirect_uri);
    redirectUrl.searchParams.set('code', code);
    redirectUrl.searchParams.set('state', request.state);

    return ctx.redirect(redirectUrl.toString());
  }

  // トークンエンドポイント
  async token(ctx: Context): Promise<Response> {
    const request = await this.parseTokenRequest(ctx);

    // クライアント認証
    const client = await this.authenticateClient(ctx, request);
    if (!client) {
      return ctx.json({ error: 'invalid_client' }, 401);
    }

    switch (request.grant_type) {
      case 'authorization_code':
        return this.handleAuthorizationCodeGrant(ctx, client, request);
      case 'refresh_token':
        return this.handleRefreshTokenGrant(ctx, client, request);
      case 'client_credentials':
        return this.handleClientCredentialsGrant(ctx, client, request);
      default:
        return ctx.json({ error: 'unsupported_grant_type' }, 400);
    }
  }

  private async handleAuthorizationCodeGrant(
    ctx: Context,
    client: Client,
    request: TokenRequest
  ): Promise<Response> {
    // 1. 認可コード取得・検証
    const authCode = await this.codeRepository.findByCode(request.code!);
    if (!authCode) {
      return ctx.json({ error: 'invalid_grant', error_description: 'Invalid authorization code' }, 400);
    }

    // 2. クライアントID一致確認
    if (authCode.clientId !== client.id) {
      return ctx.json({ error: 'invalid_grant', error_description: 'Client mismatch' }, 400);
    }

    // 3. リダイレクトURI一致確認
    if (authCode.redirectUri !== request.redirect_uri) {
      return ctx.json({ error: 'invalid_grant', error_description: 'Redirect URI mismatch' }, 400);
    }

    // 4. 有効期限確認
    if (authCode.expiresAt < new Date()) {
      return ctx.json({ error: 'invalid_grant', error_description: 'Authorization code expired' }, 400);
    }

    // 5. PKCE検証
    if (authCode.codeChallenge) {
      if (!request.code_verifier) {
        return ctx.json({ error: 'invalid_request', error_description: 'Code verifier required' }, 400);
      }

      const expectedChallenge = this.generateCodeChallenge(
        request.code_verifier,
        authCode.codeChallengeMethod
      );

      if (expectedChallenge !== authCode.codeChallenge) {
        return ctx.json({ error: 'invalid_grant', error_description: 'Invalid code verifier' }, 400);
      }
    }

    // 6. 認可コードの無効化（再利用防止）
    await this.codeRepository.revoke(authCode.code);

    // 7. ユーザー情報取得
    const user = await this.userRepository.findById(authCode.userId);
    if (!user) {
      return ctx.json({ error: 'invalid_grant', error_description: 'User not found' }, 400);
    }

    // 8. トークン生成
    const tokens = await this.generateTokens(user, client, authCode.scope, authCode.nonce);

    // 9. リフレッシュトークン保存
    await this.tokenRepository.saveRefreshToken({
      token: tokens.refreshToken,
      userId: user.id,
      clientId: client.id,
      scope: authCode.scope,
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000), // 30日
    });

    return ctx.json({
      access_token: tokens.accessToken,
      token_type: 'Bearer',
      expires_in: 3600,
      refresh_token: tokens.refreshToken,
      id_token: tokens.idToken,
      scope: authCode.scope,
    });
  }

  private async handleRefreshTokenGrant(
    ctx: Context,
    client: Client,
    request: TokenRequest
  ): Promise<Response> {
    // 1. リフレッシュトークン検証
    const storedToken = await this.tokenRepository.findRefreshToken(request.refresh_token!);
    if (!storedToken) {
      return ctx.json({ error: 'invalid_grant', error_description: 'Invalid refresh token' }, 400);
    }

    // 2. クライアントID一致確認
    if (storedToken.clientId !== client.id) {
      // 潜在的な侵害検知 - トークンファミリー全体を無効化
      await this.tokenRepository.revokeTokenFamily(storedToken.familyId);
      await this.notifySecurityTeam('refresh_token_theft_detected', storedToken);
      return ctx.json({ error: 'invalid_grant', error_description: 'Token theft detected' }, 400);
    }

    // 3. 有効期限確認
    if (storedToken.expiresAt < new Date()) {
      return ctx.json({ error: 'invalid_grant', error_description: 'Refresh token expired' }, 400);
    }

    // 4. 使用済み確認（Refresh Token Rotation）
    if (storedToken.usedAt) {
      // 再利用検知 - トークンファミリー全体を無効化
      await this.tokenRepository.revokeTokenFamily(storedToken.familyId);
      await this.notifySecurityTeam('refresh_token_reuse_detected', storedToken);
      return ctx.json({ error: 'invalid_grant', error_description: 'Token reuse detected' }, 400);
    }

    // 5. ユーザー情報取得
    const user = await this.userRepository.findById(storedToken.userId);
    if (!user || user.status !== 'active') {
      return ctx.json({ error: 'invalid_grant', error_description: 'User inactive' }, 400);
    }

    // 6. 現在のリフレッシュトークンを使用済みにマーク
    await this.tokenRepository.markAsUsed(storedToken.token);

    // 7. 新しいトークンペア生成
    const tokens = await this.generateTokens(user, client, storedToken.scope);

    // 8. 新しいリフレッシュトークン保存
    await this.tokenRepository.saveRefreshToken({
      token: tokens.refreshToken,
      userId: user.id,
      clientId: client.id,
      scope: storedToken.scope,
      familyId: storedToken.familyId,
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
    });

    return ctx.json({
      access_token: tokens.accessToken,
      token_type: 'Bearer',
      expires_in: 3600,
      refresh_token: tokens.refreshToken,
      scope: storedToken.scope,
    });
  }

  private async generateTokens(
    user: User,
    client: Client,
    scope: string,
    nonce?: string
  ): Promise<{ accessToken: string; refreshToken: string; idToken: string }> {
    const now = Math.floor(Date.now() / 1000);

    // アクセストークン（JWT）
    const accessToken = sign(
      {
        iss: 'https://auth.triptrip.com',
        sub: user.id,
        aud: client.id,
        exp: now + 3600,
        iat: now,
        scope: scope,
        roles: user.roles,
      },
      process.env.JWT_PRIVATE_KEY!,
      { algorithm: 'RS256', keyid: process.env.JWT_KEY_ID }
    );

    // IDトークン（JWT）
    const idToken = sign(
      {
        iss: 'https://auth.triptrip.com',
        sub: user.id,
        aud: client.id,
        exp: now + 3600,
        iat: now,
        auth_time: now,
        nonce: nonce,
        name: user.name,
        email: user.email,
        email_verified: user.emailVerified,
      },
      process.env.JWT_PRIVATE_KEY!,
      { algorithm: 'RS256', keyid: process.env.JWT_KEY_ID }
    );

    // リフレッシュトークン（Opaque）
    const refreshToken = randomBytes(32).toString('base64url');

    return { accessToken, refreshToken, idToken };
  }

  private generateCodeChallenge(verifier: string, method: string): string {
    if (method === 'S256') {
      return createHash('sha256').update(verifier).digest('base64url');
    }
    return verifier; // plain method (非推奨)
  }
}
```

### 2.2 多要素認証（MFA）

#### 2.2.1 MFA戦略

```yaml
mfa_strategy:
  available_methods:
    totp:
      description: Time-based One-Time Password
      apps:
        - Google Authenticator
        - Microsoft Authenticator
        - Authy
      implementation:
        algorithm: SHA1
        digits: 6
        period: 30 seconds
        drift_tolerance: 1 period
      recovery:
        backup_codes: 10 codes
        code_length: 8 characters

    sms:
      description: SMS OTP
      provider: AWS SNS / Twilio
      implementation:
        code_length: 6 digits
        validity: 5 minutes
        rate_limit: 3 per 10 minutes
      limitations:
        - SIM swapping risk
        - 国際SMS配信問題
      usage: TOTP未設定時のフォールバック

    webauthn:
      description: FIDO2/WebAuthn
      supported_authenticators:
        - Platform (TouchID, FaceID, Windows Hello)
        - Roaming (YubiKey, etc.)
      implementation:
        attestation: none
        user_verification: preferred
        resident_key: preferred
      benefits:
        - フィッシング耐性
        - パスワードレス対応

    push_notification:
      description: プッシュ通知承認
      implementation:
        provider: Firebase Cloud Messaging
        timeout: 60 seconds
        context_display: true
      benefits:
        - ユーザーフレンドリー
        - リアルタイム

  enforcement_policy:
    required:
      - 管理者アカウント
      - パートナーアカウント
      - 高額取引
      - アカウント設定変更
      - 決済情報変更

    recommended:
      - 一般ユーザー
      - 初回ログイン後プロンプト

    risk_based:
      triggers:
        - 新しいデバイス
        - 異常な場所
        - 長期間未使用後のログイン
        - 複数回の失敗試行後

  recovery_options:
    backup_codes:
      quantity: 10
      single_use: true
      regeneration: 可能

    recovery_email:
      verification: 必須
      cool_down: 24 hours after change

    support_verification:
      id_verification: 必須
      waiting_period: 24-72 hours
```

#### 2.2.2 MFAサービス実装

```typescript
// auth-service/src/mfa/mfa-service.ts

import { authenticator } from 'otplib';
import { v4 as uuidv4 } from 'uuid';
import { generateRegistrationOptions, verifyRegistrationResponse, generateAuthenticationOptions, verifyAuthenticationResponse } from '@simplewebauthn/server';

interface MfaDevice {
  id: string;
  userId: string;
  type: 'totp' | 'sms' | 'webauthn' | 'push';
  name: string;
  credential: string; // encrypted
  createdAt: Date;
  lastUsedAt?: Date;
}

class MfaService {
  private deviceRepository: MfaDeviceRepository;
  private backupCodeRepository: BackupCodeRepository;
  private smsService: SmsService;
  private pushService: PushNotificationService;

  // TOTP設定開始
  async initiateTotpSetup(userId: string): Promise<{
    secret: string;
    otpauthUrl: string;
    qrCode: string;
  }> {
    const user = await this.userRepository.findById(userId);
    const secret = authenticator.generateSecret();

    const otpauthUrl = authenticator.keyuri(
      user.email,
      'TripTrip',
      secret
    );

    // QRコード生成
    const qrCode = await this.generateQRCode(otpauthUrl);

    // 一時的に保存（検証完了まで）
    await this.storeTemporarySecret(userId, secret);

    return { secret, otpauthUrl, qrCode };
  }

  // TOTP設定完了
  async completeTotpSetup(
    userId: string,
    code: string,
    deviceName: string
  ): Promise<{ success: boolean; backupCodes?: string[] }> {
    const tempSecret = await this.getTemporarySecret(userId);
    if (!tempSecret) {
      throw new Error('Setup session expired');
    }

    // コード検証
    const isValid = authenticator.check(code, tempSecret);
    if (!isValid) {
      throw new Error('Invalid verification code');
    }

    // デバイス登録
    const device: MfaDevice = {
      id: uuidv4(),
      userId,
      type: 'totp',
      name: deviceName,
      credential: await this.encryptSecret(tempSecret),
      createdAt: new Date(),
    };

    await this.deviceRepository.save(device);
    await this.deleteTemporarySecret(userId);

    // バックアップコード生成
    const backupCodes = await this.generateBackupCodes(userId);

    // ユーザーのMFAステータス更新
    await this.userRepository.update(userId, { mfaEnabled: true });

    return { success: true, backupCodes };
  }

  // TOTP検証
  async verifyTotp(userId: string, code: string): Promise<boolean> {
    // バックアップコード確認
    if (await this.verifyBackupCode(userId, code)) {
      return true;
    }

    const device = await this.deviceRepository.findByUserIdAndType(userId, 'totp');
    if (!device) {
      throw new Error('TOTP not configured');
    }

    const secret = await this.decryptSecret(device.credential);
    const isValid = authenticator.check(code, secret);

    if (isValid) {
      await this.deviceRepository.updateLastUsed(device.id);
    }

    return isValid;
  }

  // WebAuthn登録オプション生成
  async generateWebAuthnRegistrationOptions(userId: string): Promise<PublicKeyCredentialCreationOptionsJSON> {
    const user = await this.userRepository.findById(userId);
    const existingDevices = await this.deviceRepository.findByUserIdAndType(userId, 'webauthn');

    const options = await generateRegistrationOptions({
      rpName: 'TripTrip',
      rpID: 'triptrip.com',
      userID: userId,
      userName: user.email,
      userDisplayName: user.name,
      attestationType: 'none',
      authenticatorSelection: {
        authenticatorAttachment: 'platform',
        userVerification: 'preferred',
        residentKey: 'preferred',
      },
      excludeCredentials: existingDevices.map(d => ({
        id: Buffer.from(d.credentialId, 'base64'),
        type: 'public-key',
        transports: d.transports,
      })),
    });

    // チャレンジを一時保存
    await this.storeTemporaryChallenge(userId, options.challenge);

    return options;
  }

  // WebAuthn登録検証
  async verifyWebAuthnRegistration(
    userId: string,
    response: RegistrationResponseJSON,
    deviceName: string
  ): Promise<boolean> {
    const expectedChallenge = await this.getTemporaryChallenge(userId);
    if (!expectedChallenge) {
      throw new Error('Challenge expired');
    }

    const verification = await verifyRegistrationResponse({
      response,
      expectedChallenge,
      expectedOrigin: 'https://triptrip.com',
      expectedRPID: 'triptrip.com',
    });

    if (!verification.verified || !verification.registrationInfo) {
      throw new Error('Verification failed');
    }

    // デバイス登録
    const device: MfaDevice = {
      id: uuidv4(),
      userId,
      type: 'webauthn',
      name: deviceName,
      credential: JSON.stringify({
        credentialId: Buffer.from(verification.registrationInfo.credentialID).toString('base64'),
        publicKey: Buffer.from(verification.registrationInfo.credentialPublicKey).toString('base64'),
        counter: verification.registrationInfo.counter,
        transports: response.response.transports,
      }),
      createdAt: new Date(),
    };

    await this.deviceRepository.save(device);
    await this.deleteTemporaryChallenge(userId);
    await this.userRepository.update(userId, { mfaEnabled: true });

    return true;
  }

  // WebAuthn認証オプション生成
  async generateWebAuthnAuthenticationOptions(userId: string): Promise<PublicKeyCredentialRequestOptionsJSON> {
    const devices = await this.deviceRepository.findByUserIdAndType(userId, 'webauthn');

    const options = await generateAuthenticationOptions({
      rpID: 'triptrip.com',
      allowCredentials: devices.map(d => {
        const cred = JSON.parse(d.credential);
        return {
          id: Buffer.from(cred.credentialId, 'base64'),
          type: 'public-key',
          transports: cred.transports,
        };
      }),
      userVerification: 'preferred',
    });

    await this.storeTemporaryChallenge(userId, options.challenge);

    return options;
  }

  // WebAuthn認証検証
  async verifyWebAuthnAuthentication(
    userId: string,
    response: AuthenticationResponseJSON
  ): Promise<boolean> {
    const expectedChallenge = await this.getTemporaryChallenge(userId);
    if (!expectedChallenge) {
      throw new Error('Challenge expired');
    }

    const devices = await this.deviceRepository.findByUserIdAndType(userId, 'webauthn');
    const device = devices.find(d => {
      const cred = JSON.parse(d.credential);
      return cred.credentialId === response.id;
    });

    if (!device) {
      throw new Error('Device not found');
    }

    const credential = JSON.parse(device.credential);

    const verification = await verifyAuthenticationResponse({
      response,
      expectedChallenge,
      expectedOrigin: 'https://triptrip.com',
      expectedRPID: 'triptrip.com',
      authenticator: {
        credentialID: Buffer.from(credential.credentialId, 'base64'),
        credentialPublicKey: Buffer.from(credential.publicKey, 'base64'),
        counter: credential.counter,
      },
    });

    if (!verification.verified) {
      throw new Error('Verification failed');
    }

    // カウンター更新
    credential.counter = verification.authenticationInfo.newCounter;
    await this.deviceRepository.update(device.id, {
      credential: JSON.stringify(credential),
      lastUsedAt: new Date(),
    });

    await this.deleteTemporaryChallenge(userId);

    return true;
  }

  // バックアップコード生成
  private async generateBackupCodes(userId: string): Promise<string[]> {
    const codes: string[] = [];

    for (let i = 0; i < 10; i++) {
      const code = this.generateSecureCode(8);
      codes.push(code);

      await this.backupCodeRepository.save({
        id: uuidv4(),
        userId,
        codeHash: await this.hashCode(code),
        used: false,
        createdAt: new Date(),
      });
    }

    return codes;
  }

  // バックアップコード検証
  private async verifyBackupCode(userId: string, code: string): Promise<boolean> {
    const backupCodes = await this.backupCodeRepository.findUnusedByUserId(userId);

    for (const bc of backupCodes) {
      if (await this.verifyCodeHash(code, bc.codeHash)) {
        await this.backupCodeRepository.markAsUsed(bc.id);
        return true;
      }
    }

    return false;
  }
}
```

### 2.3 パスワードレス認証

#### 2.3.1 パスワードレス認証戦略

```yaml
passwordless_authentication:
  methods:
    magic_link:
      description: メールリンク認証
      flow:
        1: ユーザーがメールアドレスを入力
        2: 一意のリンクを含むメールを送信
        3: ユーザーがリンクをクリック
        4: トークン検証、セッション開始
      security:
        token_validity: 15 minutes
        single_use: true
        ip_binding: optional
      use_cases:
        - 初回登録
        - パスワードリセット代替
        - 低頻度ログイン

    webauthn_passwordless:
      description: FIDO2 Passwordless
      flow:
        1: ユーザーがユーザー名/メールを入力
        2: WebAuthn認証オプション生成
        3: 生体認証/セキュリティキーで認証
        4: サーバーで検証、セッション開始
      security:
        user_verification: required
        resident_key: required
      use_cases:
        - 高セキュリティアカウント
        - 頻繁なログイン
        - モバイルアプリ

    one_tap_sign_in:
      description: Google One Tap / Apple Sign In
      providers:
        google:
          sdk: Google Identity Services
          features:
            - One Tap prompt
            - Automatic sign-in
        apple:
          sdk: Sign In with Apple
          features:
            - Hide My Email
            - Face ID / Touch ID
      use_cases:
        - 新規ユーザー獲得
        - 簡単なログイン

  migration_strategy:
    phase_1:
      - WebAuthn登録プロンプト（パスワードログイン後）
      - Magic Link対応

    phase_2:
      - パスワードレスログインオプション提供
      - 段階的なパスワード非推奨化

    phase_3:
      - パスワードレス優先
      - パスワードはレガシーオプション
```

---

## 第3章：認可モデル

### 3.1 RBAC設計

#### 3.1.1 ロール階層

```yaml
role_hierarchy:
  system_roles:
    super_admin:
      description: システム全体の管理者
      permissions:
        - "*:*"  # 全権限
      constraints:
        - MFA必須
        - 監査ログ強化
        - 時間制限アクセス

    admin:
      description: サービス管理者
      permissions:
        - users:read
        - users:write
        - users:delete
        - bookings:read
        - bookings:write
        - reports:read
        - settings:read
        - settings:write
      inherits: []

    support:
      description: カスタマーサポート
      permissions:
        - users:read
        - bookings:read
        - bookings:write（cancel_only）
        - tickets:*
      constraints:
        - PII閲覧制限（マスキング）

  business_roles:
    partner_admin:
      description: パートナー管理者
      permissions:
        - partner:own:*
        - hotels:own:*
        - bookings:own:read
        - reports:own:read
      scope: own_partner_id

    partner_staff:
      description: パートナースタッフ
      permissions:
        - hotels:own:read
        - hotels:own:update
        - bookings:own:read
      scope: own_partner_id

  user_roles:
    premium_user:
      description: プレミアムユーザー
      permissions:
        - bookings:own:*
        - reviews:own:*
        - favorites:own:*
        - premium_features:*
      features:
        - priority_support
        - exclusive_deals
        - early_access

    regular_user:
      description: 一般ユーザー
      permissions:
        - bookings:own:*
        - reviews:own:*
        - favorites:own:*
      default: true

    guest:
      description: 未認証ユーザー
      permissions:
        - hotels:read
        - search:read
        - public:read
```

#### 3.1.2 パーミッションモデル

```yaml
permission_model:
  structure:
    format: "{resource}:{scope}:{action}"
    examples:
      - "users:*:read"
      - "bookings:own:write"
      - "hotels:partner_123:update"

  resources:
    users:
      actions: [read, write, delete]
      scopes: [all, own]

    bookings:
      actions: [read, write, cancel]
      scopes: [all, own, partner]

    hotels:
      actions: [read, write, update, delete]
      scopes: [all, own, partner]

    payments:
      actions: [read, process, refund]
      scopes: [all, own]

    reports:
      actions: [read, export]
      scopes: [all, own, partner]

  wildcards:
    "*": all actions/resources/scopes
    "{resource}:*": all actions on resource
    "{resource}:{scope}:*": all actions on scoped resource
```

### 3.2 ABAC設計

#### 3.2.1 属性ベースポリシー

```yaml
abac_policies:
  attributes:
    subject:
      - user_id
      - roles
      - department
      - location
      - device_trust_level
      - authentication_level

    resource:
      - resource_id
      - resource_type
      - owner_id
      - classification
      - created_at
      - region

    action:
      - action_type
      - requested_at
      - request_context

    environment:
      - current_time
      - ip_address
      - geo_location
      - risk_score

  policy_examples:
    time_based_access:
      description: 営業時間内のみアクセス許可
      condition: |
        environment.current_time.hour >= 9 AND
        environment.current_time.hour < 18 AND
        environment.current_time.day_of_week IN [1,2,3,4,5]
      resources: [admin_panel, reports]
      effect: allow

    location_based_access:
      description: 日本国内からのみアクセス許可
      condition: |
        environment.geo_location.country == 'JP'
      resources: [sensitive_data]
      effect: allow

    ownership_policy:
      description: リソースオーナーのみ編集可能
      condition: |
        subject.user_id == resource.owner_id OR
        'admin' IN subject.roles
      actions: [update, delete]
      effect: allow

    risk_based_policy:
      description: 高リスクアクセスはMFA必須
      condition: |
        environment.risk_score > 50 AND
        subject.authentication_level < 2
      effect: deny
      message: "MFA required for this operation"

    data_classification_policy:
      description: 機密データアクセス制限
      condition: |
        resource.classification == 'CONFIDENTIAL' AND
        NOT ('confidential_access' IN subject.permissions)
      effect: deny
```

#### 3.2.2 ポリシー評価エンジン

```typescript
// auth-service/src/authorization/policy-engine.ts

interface PolicyContext {
  subject: SubjectAttributes;
  resource: ResourceAttributes;
  action: ActionAttributes;
  environment: EnvironmentAttributes;
}

interface SubjectAttributes {
  userId: string;
  roles: string[];
  permissions: string[];
  department?: string;
  authenticationLevel: number;
  deviceTrustLevel: number;
}

interface ResourceAttributes {
  resourceId: string;
  resourceType: string;
  ownerId?: string;
  partnerId?: string;
  classification: 'PUBLIC' | 'INTERNAL' | 'CONFIDENTIAL' | 'RESTRICTED';
  region?: string;
}

interface ActionAttributes {
  actionType: string;
  requestedAt: Date;
}

interface EnvironmentAttributes {
  currentTime: Date;
  ipAddress: string;
  geoLocation: {
    country: string;
    region: string;
    city: string;
  };
  riskScore: number;
}

interface PolicyDecision {
  allowed: boolean;
  reason?: string;
  obligations?: Obligation[];
}

interface Obligation {
  type: string;
  parameters: Record<string, unknown>;
}

class PolicyEngine {
  private policyRepository: PolicyRepository;
  private auditLogger: AuditLogger;

  async evaluate(context: PolicyContext): Promise<PolicyDecision> {
    const startTime = Date.now();

    try {
      // 1. RBAC評価
      const rbacDecision = await this.evaluateRbac(context);
      if (!rbacDecision.allowed) {
        return this.logAndReturn(context, rbacDecision, startTime);
      }

      // 2. ABAC評価
      const abacDecision = await this.evaluateAbac(context);
      if (!abacDecision.allowed) {
        return this.logAndReturn(context, abacDecision, startTime);
      }

      // 3. 追加制約の評価
      const constraintDecision = await this.evaluateConstraints(context);
      if (!constraintDecision.allowed) {
        return this.logAndReturn(context, constraintDecision, startTime);
      }

      // 4. 義務の収集
      const obligations = await this.collectObligations(context);

      return this.logAndReturn(context, {
        allowed: true,
        obligations,
      }, startTime);

    } catch (error) {
      await this.auditLogger.logError(context, error);
      return { allowed: false, reason: 'Policy evaluation error' };
    }
  }

  private async evaluateRbac(context: PolicyContext): Promise<PolicyDecision> {
    const { subject, resource, action } = context;

    // パーミッション文字列の生成
    const requiredPermissions = [
      `${resource.resourceType}:*:${action.actionType}`,
      `${resource.resourceType}:*:*`,
      `${resource.resourceType}:own:${action.actionType}`,
      '*:*:*',
    ];

    // スコープ付きパーミッションの追加
    if (resource.ownerId === subject.userId) {
      requiredPermissions.push(`${resource.resourceType}:own:${action.actionType}`);
    }

    // パーミッションチェック
    const hasPermission = requiredPermissions.some(perm =>
      subject.permissions.includes(perm) ||
      this.matchWildcard(subject.permissions, perm)
    );

    if (!hasPermission) {
      return { allowed: false, reason: 'Insufficient permissions' };
    }

    return { allowed: true };
  }

  private async evaluateAbac(context: PolicyContext): Promise<PolicyDecision> {
    const policies = await this.policyRepository.findApplicable(context);

    for (const policy of policies) {
      const result = await this.evaluatePolicy(policy, context);

      if (result.effect === 'deny') {
        return { allowed: false, reason: result.message || policy.description };
      }
    }

    return { allowed: true };
  }

  private async evaluatePolicy(
    policy: Policy,
    context: PolicyContext
  ): Promise<{ effect: 'allow' | 'deny'; message?: string }> {
    // ポリシー条件の評価
    const conditionResult = this.evaluateCondition(policy.condition, context);

    if (policy.effect === 'deny' && conditionResult) {
      return { effect: 'deny', message: policy.message };
    }

    if (policy.effect === 'allow' && !conditionResult) {
      return { effect: 'deny', message: `Policy ${policy.name} not satisfied` };
    }

    return { effect: 'allow' };
  }

  private evaluateCondition(condition: string, context: PolicyContext): boolean {
    // 安全な条件評価（eval使用禁止）
    // JSON Logic または専用のDSLパーサーを使用

    const evaluator = new ConditionEvaluator();
    return evaluator.evaluate(condition, {
      subject: context.subject,
      resource: context.resource,
      action: context.action,
      environment: context.environment,
    });
  }

  private async evaluateConstraints(context: PolicyContext): Promise<PolicyDecision> {
    const { subject, resource, action, environment } = context;

    // 時間制約
    if (this.isTimeRestrictedAction(action.actionType)) {
      if (!this.isWithinBusinessHours(environment.currentTime)) {
        return { allowed: false, reason: 'Action not allowed outside business hours' };
      }
    }

    // リスクベース制約
    if (environment.riskScore > 80) {
      return { allowed: false, reason: 'Risk score too high' };
    }

    if (environment.riskScore > 50 && subject.authenticationLevel < 2) {
      return {
        allowed: false,
        reason: 'MFA required for elevated risk access',
        obligations: [{ type: 'require_mfa', parameters: {} }]
      };
    }

    // データ分類制約
    if (resource.classification === 'RESTRICTED') {
      if (!subject.permissions.includes('restricted_access')) {
        return { allowed: false, reason: 'Restricted data access not permitted' };
      }
    }

    return { allowed: true };
  }

  private async collectObligations(context: PolicyContext): Promise<Obligation[]> {
    const obligations: Obligation[] = [];

    // 監査ログ義務
    if (context.resource.classification !== 'PUBLIC') {
      obligations.push({
        type: 'audit_log',
        parameters: {
          detail_level: 'full',
          retention: '7years',
        },
      });
    }

    // 機密データの場合はマスキング義務
    if (context.resource.classification === 'CONFIDENTIAL') {
      obligations.push({
        type: 'data_masking',
        parameters: {
          fields: ['email', 'phone', 'address'],
        },
      });
    }

    return obligations;
  }

  private async logAndReturn(
    context: PolicyContext,
    decision: PolicyDecision,
    startTime: number
  ): Promise<PolicyDecision> {
    const duration = Date.now() - startTime;

    await this.auditLogger.log({
      timestamp: new Date(),
      subject: context.subject.userId,
      resource: `${context.resource.resourceType}:${context.resource.resourceId}`,
      action: context.action.actionType,
      decision: decision.allowed ? 'ALLOW' : 'DENY',
      reason: decision.reason,
      duration,
      context: {
        ip: context.environment.ipAddress,
        riskScore: context.environment.riskScore,
      },
    });

    return decision;
  }
}
```

### 3.3 ポリシー評価エンジン

#### 3.3.1 認可ミドルウェア

```typescript
// auth-service/src/middleware/authorization.ts

import { Context, Next } from 'hono';

interface AuthorizationOptions {
  resource: string | ((ctx: Context) => string);
  action: string;
  resourceIdParam?: string;
  skipIf?: (ctx: Context) => boolean;
}

function authorize(options: AuthorizationOptions) {
  return async (ctx: Context, next: Next) => {
    // スキップ条件のチェック
    if (options.skipIf && options.skipIf(ctx)) {
      return next();
    }

    const user = ctx.get('user');
    if (!user) {
      return ctx.json({ error: 'Authentication required' }, 401);
    }

    // リソースIDの取得
    const resourceId = options.resourceIdParam
      ? ctx.req.param(options.resourceIdParam)
      : undefined;

    // リソース情報の取得
    const resource = await getResourceInfo(
      typeof options.resource === 'function'
        ? options.resource(ctx)
        : options.resource,
      resourceId
    );

    // 環境情報の収集
    const environment: EnvironmentAttributes = {
      currentTime: new Date(),
      ipAddress: ctx.req.header('X-Forwarded-For') || ctx.req.ip,
      geoLocation: await getGeoLocation(ctx.req.header('X-Forwarded-For') || ctx.req.ip),
      riskScore: user.riskScore || 0,
    };

    // ポリシー評価
    const policyEngine = new PolicyEngine();
    const decision = await policyEngine.evaluate({
      subject: {
        userId: user.id,
        roles: user.roles,
        permissions: user.permissions,
        authenticationLevel: user.authLevel,
        deviceTrustLevel: user.deviceTrust,
      },
      resource,
      action: {
        actionType: options.action,
        requestedAt: new Date(),
      },
      environment,
    });

    if (!decision.allowed) {
      return ctx.json({
        error: 'Access denied',
        reason: decision.reason,
      }, 403);
    }

    // 義務の処理
    if (decision.obligations) {
      ctx.set('obligations', decision.obligations);
    }

    return next();
  };
}

// 使用例
app.get(
  '/api/bookings/:id',
  authenticate,
  authorize({
    resource: 'bookings',
    action: 'read',
    resourceIdParam: 'id',
  }),
  async (ctx) => {
    const bookingId = ctx.req.param('id');
    const obligations = ctx.get('obligations') || [];

    let booking = await bookingService.getById(bookingId);

    // 義務の適用（データマスキングなど）
    for (const obligation of obligations) {
      if (obligation.type === 'data_masking') {
        booking = applyDataMasking(booking, obligation.parameters.fields);
      }
    }

    return ctx.json(booking);
  }
);

app.post(
  '/api/bookings',
  authenticate,
  authorize({
    resource: 'bookings',
    action: 'write',
  }),
  validateRequest(createBookingSchema),
  async (ctx) => {
    const body = ctx.get('validatedBody');
    const user = ctx.get('user');

    const booking = await bookingService.create({
      ...body,
      userId: user.id,
    });

    return ctx.json(booking, 201);
  }
);
```

---

## 第4章：アイデンティティライフサイクル

### 4.1 プロビジョニング & デプロビジョニング

#### 4.1.1 ユーザーライフサイクル管理

```yaml
identity_lifecycle:
  provisioning:
    self_registration:
      flow:
        1: ユーザーが登録フォームを送信
        2: メールアドレス検証
        3: 必須プロファイル入力
        4: 利用規約同意
        5: アカウント作成完了
        6: ウェルカムメール送信
      validations:
        - メール形式チェック
        - パスワード強度検証
        - 重複チェック
        - ボット検知（reCAPTCHA）

    admin_provisioning:
      flow:
        1: 管理者がユーザー情報入力
        2: ロール・権限割り当て
        3: 招待メール送信
        4: ユーザーがパスワード設定
        5: MFA設定（必要時）
      audit:
        - 作成者の記録
        - 承認ワークフロー（高権限）

    automated_provisioning:
      triggers:
        - SSO初回ログイン
        - API経由の作成
      mappings:
        - SSOクレームからのプロファイルマッピング
        - デフォルトロール割り当て

  modification:
    profile_updates:
      self_service:
        - 名前
        - 表示名
        - プロフィール画像
        - 連絡先設定
      admin_only:
        - ロール
        - 権限
        - アカウントステータス

    email_change:
      flow:
        1: 新メールアドレス入力
        2: 現在のパスワード確認
        3: 新メールアドレス検証
        4: 変更完了
        5: 旧アドレスに通知
      security:
        - 24時間のクールダウン期間
        - MFA要求

    role_changes:
      workflow:
        standard:
          - 管理者による変更
          - 即時適用
        privileged:
          - 変更リクエスト作成
          - 承認者による承認
          - 適用

  deprovisioning:
    user_requested:
      soft_delete:
        - アカウント無効化
        - データ保持（30日）
        - 復元可能
      hard_delete:
        - GDPR削除リクエスト
        - データ匿名化/削除
        - 監査ログ保持

    admin_initiated:
      reasons:
        - 利用規約違反
        - セキュリティインシデント
        - 契約終了
      actions:
        - 即時アクセス無効化
        - アクティブセッション終了
        - トークン無効化
        - 通知送信

    automated:
      triggers:
        - 長期非アクティブ（365日）
        - 契約期限切れ
        - 外部IdP無効化
      grace_period: 30日
      notifications:
        - 事前警告（30日前、7日前、1日前）
        - 無効化通知
```

### 4.2 アクセスレビュー

```yaml
access_review:
  periodic_reviews:
    user_access:
      frequency: 四半期
      scope:
        - アクティブユーザー全員
        - ロール割り当て
        - 権限
      reviewers:
        - 部門マネージャー
        - セキュリティチーム
      actions:
        - 継続
        - 修正
        - 削除

    privileged_access:
      frequency: 月次
      scope:
        - 管理者アカウント
        - 特権ロール
        - API キー
      reviewers:
        - CISO
        - システム管理者
      certification_required: true

    service_accounts:
      frequency: 四半期
      scope:
        - 全サービスアカウント
        - API クライアント
        - システム統合
      reviewers:
        - 技術リード
        - セキュリティチーム

  event_triggered_reviews:
    role_change:
      - 昇進/降格時
      - 部門異動時
    security_incident:
      - 侵害検知後
      - 不審活動検知後
    compliance_audit:
      - 監査前
      - 監査指摘後

  reporting:
    metrics:
      - レビュー完了率
      - 変更率
      - 例外件数
    compliance_evidence:
      - レビューログ
      - 承認記録
      - 変更履歴
```

### 4.3 セルフサービス機能

```yaml
self_service_portal:
  features:
    profile_management:
      - プロフィール更新
      - パスワード変更
      - MFA設定
      - 通知設定

    access_requests:
      - 追加権限リクエスト
      - ロール変更リクエスト
      - 一時的アクセスリクエスト

    security:
      - アクティブセッション管理
      - ログイン履歴確認
      - デバイス管理
      - APIキー管理

    recovery:
      - パスワードリセット
      - MFAリカバリー
      - アカウントアンロック

  automation:
    auto_approval:
      conditions:
        - 低リスクリクエスト
        - 事前承認済みロール
        - 時間制限付きアクセス
    workflow_routing:
      - リクエストタイプによるルーティング
      - 承認者の自動割り当て
      - エスカレーション
```

---

## 第5章：サービス間セキュリティ

### 5.1 mTLS実装

#### 5.1.1 サービスメッシュmTLS

```yaml
service_mesh_mtls:
  istio_configuration:
    peer_authentication:
      apiVersion: security.istio.io/v1beta1
      kind: PeerAuthentication
      metadata:
        name: default
        namespace: istio-system
      spec:
        mtls:
          mode: STRICT

    destination_rule:
      apiVersion: networking.istio.io/v1beta1
      kind: DestinationRule
      metadata:
        name: default
        namespace: istio-system
      spec:
        host: "*.local"
        trafficPolicy:
          tls:
            mode: ISTIO_MUTUAL

  certificate_management:
    provider: Istio CA (citadel)
    rotation:
      workload_certs: 24時間
      root_ca: 10年
    distribution:
      method: SDS (Secret Discovery Service)
      automatic: true

  trust_domain:
    name: cluster.local
    federated_domains:
      - dr.cluster.local

  monitoring:
    metrics:
      - istio_requests_total{security_policy="MUTUAL_TLS"}
      - citadel_server_root_cert_expiry_timestamp
    alerts:
      - cert_expiry_warning (7日前)
      - mtls_errors
```

#### 5.1.2 外部サービス認証

```yaml
external_service_auth:
  api_clients:
    registration:
      - クライアントID発行
      - シークレット生成
      - 許可スコープ設定
      - レート制限設定

    authentication:
      method: client_credentials
      token_endpoint: /oauth2/token
      token_type: JWT
      expiry: 1時間

    security:
      - シークレットローテーション（90日）
      - IP許可リスト（オプション）
      - 監査ログ

  webhook_authentication:
    methods:
      hmac_signature:
        algorithm: HMAC-SHA256
        header: X-TripTrip-Signature
        payload: request body
      oauth_bearer:
        header: Authorization: Bearer {token}
      api_key:
        header: X-API-Key

  third_party_integrations:
    stripe:
      auth: API Key (Bearer)
      webhook: HMAC-SHA256
    sendgrid:
      auth: API Key (Bearer)
    firebase:
      auth: Service Account JWT
    aws_services:
      auth: IAM Role (IRSA)
```

### 5.2 サービスメッシュセキュリティ

#### 5.2.1 Istio Authorization Policy

```yaml
istio_authorization:
  default_deny:
    apiVersion: security.istio.io/v1beta1
    kind: AuthorizationPolicy
    metadata:
      name: deny-all
      namespace: triptrip-api
    spec:
      {}  # 空のspecはすべて拒否

  allow_api_gateway:
    apiVersion: security.istio.io/v1beta1
    kind: AuthorizationPolicy
    metadata:
      name: allow-api-gateway
      namespace: triptrip-api
    spec:
      selector:
        matchLabels:
          app: triptrip-api
      action: ALLOW
      rules:
        - from:
            - source:
                principals:
                  - "cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
          to:
            - operation:
                methods: ["GET", "POST", "PUT", "DELETE"]
                paths: ["/api/*"]

  service_to_service:
    apiVersion: security.istio.io/v1beta1
    kind: AuthorizationPolicy
    metadata:
      name: allow-booking-to-payment
      namespace: triptrip-payment
    spec:
      selector:
        matchLabels:
          app: payment-service
      action: ALLOW
      rules:
        - from:
            - source:
                principals:
                  - "cluster.local/ns/triptrip-api/sa/booking-service"
          to:
            - operation:
                methods: ["POST"]
                paths: ["/internal/process-payment"]

  jwt_validation:
    apiVersion: security.istio.io/v1beta1
    kind: RequestAuthentication
    metadata:
      name: jwt-auth
      namespace: triptrip-api
    spec:
      selector:
        matchLabels:
          app: triptrip-api
      jwtRules:
        - issuer: "https://auth.triptrip.com"
          jwksUri: "https://auth.triptrip.com/.well-known/jwks.json"
          audiences:
            - "triptrip-api"
          forwardOriginalToken: true
```

### 5.3 APIキー管理

```yaml
api_key_management:
  key_types:
    public_api_key:
      description: クライアントサイドで使用
      permissions: 限定的（読み取り専用）
      rate_limit: 1000 req/min
      expiry: なし
      rotation: 推奨（年次）

    secret_api_key:
      description: サーバーサイドで使用
      permissions: フル（設定可能）
      rate_limit: 10000 req/min
      expiry: 90日
      rotation: 必須

    service_api_key:
      description: サービス間通信
      permissions: サービス固有
      rate_limit: 無制限
      expiry: 30日
      rotation: 自動

  storage:
    method: AWS Secrets Manager
    encryption: AWS KMS
    access: IAM ロール

  rotation:
    automatic:
      enabled: true
      period: 30-90日
      notification: 7日前
    manual:
      trigger: ユーザーリクエスト、セキュリティインシデント
      immediate_revocation: 可能

  monitoring:
    usage_tracking:
      - リクエスト数
      - エラー率
      - 使用パターン
    anomaly_detection:
      - 異常な使用量
      - 地理的異常
      - 時間帯異常
```

---

## 第6章：実装ロードマップ & 文書間参照

### 6.1 実装ロードマップ

```yaml
implementation_roadmap:
  phase_1_foundation:
    timeline: 月1-2
    focus: 認証基盤強化
    deliverables:
      - OAuth2/OIDC実装の完成
      - MFA（TOTP）実装
      - リフレッシュトークンローテーション
      - セッション管理強化
    success_criteria:
      - OIDC準拠テスト通過
      - MFA登録率 >50%

  phase_2_authorization:
    timeline: 月3-4
    focus: 認可モデル
    deliverables:
      - RBAC実装
      - ポリシーエンジン基盤
      - 基本的なABACポリシー
      - 認可ミドルウェア
    success_criteria:
      - 全APIエンドポイントに認可適用
      - ポリシー評価レイテンシ <5ms

  phase_3_service_auth:
    timeline: 月5-6
    focus: サービス間認証
    deliverables:
      - mTLS実装（Istio）
      - サービスアカウント管理
      - APIキー管理システム
      - 外部サービス統合
    success_criteria:
      - 全サービス間通信mTLS化
      - APIキー自動ローテーション

  phase_4_advanced:
    timeline: 月7-9
    focus: 高度な機能
    deliverables:
      - WebAuthn/パスワードレス
      - 高度なABACポリシー
      - リスクベースアクセス制御
      - SSO/フェデレーション
    success_criteria:
      - パスワードレス対応率 >30%
      - リスクベース検知精度 >95%

  phase_5_operations:
    timeline: 月10-12
    focus: 運用成熟
    deliverables:
      - アクセスレビュー自動化
      - セルフサービスポータル
      - 監査・レポーティング
      - コンプライアンス対応
    success_criteria:
      - アクセスレビュー完了率 100%
      - 監査指摘事項 0件
```

### 6.2 文書間参照

```yaml
document_references:
  inputs:
    - document: Doc-SC-001
      title: セキュリティアーキテクチャ＆戦略
      relevance: ゼロトラストモデル、継続的検証
      key_integrations:
        - ゼロトラスト原則の適用
        - リスクベースアクセス制御
        - 監視・検知との連携

    - document: Doc-DA-005
      title: データセキュリティ、プライバシー＆ガバナンス
      relevance: データアクセス制御
      key_integrations:
        - データ分類に基づく認可
        - PIIアクセス制御

    - document: Doc-IA-002
      title: Kubernetes＆コンテナ戦略
      relevance: サービスメッシュセキュリティ
      key_integrations:
        - Istio AuthorizationPolicy
        - Pod Security

  outputs:
    - document: Doc-SC-003
      title: データ保護＆暗号化
      dependency: 認証トークンの暗号化

    - document: Doc-SC-004
      title: コンプライアンス＆規制フレームワーク
      dependency: アクセス制御監査証跡
```

### 6.3 技術スタックサマリー

```yaml
iam_technology_stack:
  authentication:
    protocol: OAuth2, OIDC
    jwt_library: jsonwebtoken
    mfa:
      totp: otplib
      webauthn: "@simplewebauthn/*"
    social_login:
      - Google Identity Services
      - Apple Sign In
      - LINE Login SDK

  authorization:
    model: RBAC + ABAC hybrid
    policy_engine: Custom (TypeScript)
    caching: Redis

  service_auth:
    service_mesh: Istio
    mtls: Istio CA
    api_key_management: AWS Secrets Manager

  identity_store:
    database: PostgreSQL
    session_store: Redis
    token_store: Redis

  monitoring:
    audit_logging: CloudWatch Logs
    metrics: Prometheus
    alerting: PagerDuty
```

---

## 付録

### A. OAuth2スコープ一覧

| スコープ | 説明 | デフォルト |
|---------|------|----------|
| openid | OIDC必須 | Yes |
| profile | プロファイル情報 | Yes |
| email | メールアドレス | Yes |
| phone | 電話番号 | No |
| bookings:read | 予約閲覧 | Yes |
| bookings:write | 予約作成・更新 | Yes |
| payments:read | 決済情報閲覧 | No |
| payments:write | 決済実行 | No |
| admin | 管理者機能 | No |

### B. ロール・パーミッションマトリックス

| ロール | users:read | users:write | bookings:read | bookings:write | admin |
|--------|-----------|-------------|---------------|----------------|-------|
| super_admin | ✓ | ✓ | ✓ | ✓ | ✓ |
| admin | ✓ | ✓ | ✓ | ✓ | - |
| support | ✓ | - | ✓ | ✓(cancel) | - |
| partner_admin | own | - | own | - | - |
| regular_user | own | own | own | own | - |

### C. 変更履歴

| バージョン | 日付 | 変更内容 | 作成者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-01-20 | 初版作成 | Technical Architecture Agent |

---

**文書情報**
- 文書ID: Doc-SC-002
- バージョン: 1.0.0
- ステータス: ドラフト
- 最終更新: 2026-01-20
- 次回レビュー: 2026-02-20
- 行数: 約2,000行

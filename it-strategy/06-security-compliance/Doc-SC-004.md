# Doc-SC-004: TripTripコンプライアンス & 規制フレームワーク

## エグゼクティブサマリー

本文書は、TripTripプラットフォームのグローバルコンプライアンスおよび規制フレームワークを定義します。GDPR（EU一般データ保護規則）、CCPA/CPRA（カリフォルニア州消費者プライバシー法）、日本個人情報保護法（APPI）、PCI-DSS（決済カード業界データセキュリティ基準）、SOC 2 Type II認証の要件と実装戦略を詳述します。データ主体の権利実現、同意管理、データ保護影響評価（DPIA）、継続的コンプライアンス監視を網羅し、グローバル展開に対応した堅牢なコンプライアンス基盤を構築します。本設計は、Doc-SC-001（セキュリティアーキテクチャ）、Doc-SC-002（ID＆アクセス管理）、Doc-SC-003（データ保護＆暗号化）と連携し、統合的なセキュリティ・コンプライアンス体制を実現します。

---

## 第1章：はじめに & コンテキスト

### 1.1 目的 & スコープ

#### 1.1.1 文書の目的

```yaml
document_purpose:
  primary_objectives:
    - TripTripプラットフォームのグローバルコンプライアンス戦略定義
    - 主要規制（GDPR、CCPA、APPI、PCI-DSS、SOC2）の要件マッピング
    - データ主体の権利実現メカニズムの設計
    - コンプライアンス自動化と継続的監視の実装
    - 監査対応プロセスの確立

  scope:
    in_scope:
      - プライバシー規制（GDPR、CCPA/CPRA、APPI）
      - セキュリティ認証（SOC 2 Type II、ISO 27001検討）
      - 決済規制（PCI-DSS）
      - データ主体の権利管理
      - 同意管理プラットフォーム
      - データ保護影響評価（DPIA）
      - コンプライアンス監視・レポーティング

    out_of_scope:
      - 技術的セキュリティ実装（Doc-SC-001〜003参照）
      - 法的助言（法務チーム/外部法律事務所の責任）
      - 特定国の規制詳細（主要3地域以外）

  target_audience:
    - データ保護責任者（DPO）
    - コンプライアンス担当者
    - 法務チーム
    - 経営層
    - 監査チーム
```

#### 1.1.2 規制ランドスケープ概要

```yaml
regulatory_landscape:
  primary_regulations:
    gdpr:
      jurisdiction: EU/EEA + UK
      applicable_when:
        - EU居住者へのサービス提供
        - EU内での事業活動
      key_requirements:
        - 合法的処理根拠
        - データ主体の権利
        - データ保護バイデザイン
        - 越境移転制限
      penalties:
        max: "2000万ユーロまたは全世界売上の4%"

    ccpa_cpra:
      jurisdiction: カリフォルニア州（米国）
      applicable_when:
        - CA居住者の個人情報処理
        - 年間売上 >$25M
        - 50,000以上のCA消費者データ処理
      key_requirements:
        - プライバシー通知
        - オプトアウト権
        - データアクセス権
        - 削除権
      penalties:
        intentional: "$7,500/違反"
        unintentional: "$2,500/違反"

    japan_appi:
      jurisdiction: 日本
      applicable_when:
        - 日本国内での事業活動
        - 日本居住者データの処理
      key_requirements:
        - 利用目的の特定・通知
        - 第三者提供制限
        - 安全管理措置
        - 本人からの請求対応
      penalties:
        criminal: "最大1年以下の懲役または100万円以下の罰金"
        administrative: "是正命令、公表"

    pci_dss:
      jurisdiction: グローバル（カード取引）
      applicable_when:
        - クレジットカード決済処理
        - カードホルダーデータの保存/処理/伝送
      key_requirements:
        - カードデータのセキュリティ
        - ネットワークセキュリティ
        - アクセス制御
        - 監視・テスト
      penalties:
        - カードブランドによる罰金（$5,000〜$100,000/月）
        - 決済処理能力の喪失

    soc2:
      jurisdiction: グローバル（任意認証）
      applicable_when:
        - B2B顧客要件
        - 信頼性の証明が必要
      trust_principles:
        - Security
        - Availability
        - Processing Integrity
        - Confidentiality
        - Privacy
```

### 1.2 コンプライアンス目標

```yaml
compliance_objectives:
  strategic_goals:
    regulatory_compliance:
      target: 主要規制への完全準拠
      metrics:
        - 規制違反件数: 0
        - 監査指摘事項（Critical/High）: 0
        - 是正措置完了率: 100%

    customer_trust:
      target: 顧客からの信頼獲得
      metrics:
        - プライバシー関連苦情: <0.01%
        - データ主体リクエスト対応率: 100%
        - 対応時間SLA遵守: >99%

    operational_efficiency:
      target: コンプライアンス業務の効率化
      metrics:
        - 自動化率: >70%
        - 手動レビュー削減: >50%
        - 監査準備時間削減: >60%

  success_criteria:
    year_1:
      - GDPR準拠達成
      - PCI-DSS SAQ A完了
      - 基本的なプライバシー機能実装

    year_2:
      - SOC 2 Type II取得
      - CCPA/CPRA準拠
      - コンプライアンス自動化

    year_3:
      - ISO 27001検討
      - グローバル展開対応
      - 成熟したコンプライアンス体制
```

---

## 第2章：プライバシー規制

### 2.1 GDPR実装

#### 2.1.1 GDPR要件マッピング

```yaml
gdpr_requirements:
  chapter_2_principles:
    article_5_principles:
      lawfulness_fairness_transparency:
        requirement: 適法、公正、透明な処理
        implementation:
          - プライバシーポリシーの公開
          - 処理活動の記録
          - 適法根拠の文書化
        status: 実装予定

      purpose_limitation:
        requirement: 目的の特定と制限
        implementation:
          - 処理目的の明確化
          - 目的外使用の禁止
        status: 実装予定

      data_minimisation:
        requirement: データ最小化
        implementation:
          - 必要最小限のデータ収集
          - 定期的なデータレビュー
        status: 実装予定

      accuracy:
        requirement: 正確性
        implementation:
          - データ更新機能
          - 定期的な検証
        status: 実装予定

      storage_limitation:
        requirement: 保存制限
        implementation:
          - 保持期間ポリシー
          - 自動削除メカニズム
        status: 実装予定

      integrity_confidentiality:
        requirement: 完全性と機密性
        implementation:
          - 暗号化（Doc-SC-003参照）
          - アクセス制御（Doc-SC-002参照）
        status: 実装予定

    article_6_lawful_bases:
      consent:
        use_cases:
          - マーケティングコミュニケーション
          - Cookie（非必須）
          - プロファイリング
        implementation:
          - 同意管理プラットフォーム
          - 明示的な同意取得UI
          - 同意の記録と証跡

      contract:
        use_cases:
          - 予約処理
          - アカウント管理
          - 決済処理
        implementation:
          - 利用規約との紐付け
          - 処理の必要性文書化

      legal_obligation:
        use_cases:
          - 税務記録
          - 本人確認（KYC）
          - 法的紛争
        implementation:
          - 法的根拠の文書化
          - 保持期間の設定

      legitimate_interest:
        use_cases:
          - 不正防止
          - サービス改善
          - セキュリティ
        implementation:
          - 正当利益評価（LIA）
          - バランステスト実施

  chapter_3_data_subject_rights:
    article_15_access:
      requirement: アクセス権
      response_time: 30日
      implementation: データエクスポート機能

    article_16_rectification:
      requirement: 訂正権
      response_time: 30日
      implementation: プロファイル編集機能

    article_17_erasure:
      requirement: 削除権（忘れられる権利）
      response_time: 30日
      implementation: アカウント削除機能

    article_18_restriction:
      requirement: 処理制限権
      response_time: 30日
      implementation: アカウント一時停止

    article_20_portability:
      requirement: データポータビリティ
      response_time: 30日
      format: JSON, CSV
      implementation: データエクスポート（機械可読形式）

    article_21_objection:
      requirement: 異議申立権
      response_time: 30日
      implementation: マーケティングオプトアウト

    article_22_automated_decision:
      requirement: 自動化された意思決定に服しない権利
      implementation: 人的介入オプション

  chapter_4_controller_obligations:
    article_25_data_protection_by_design:
      requirement: データ保護バイデザイン
      implementation:
        - プライバシー影響評価
        - デフォルトでのプライバシー保護
        - 開発プロセスへの組み込み

    article_30_records_of_processing:
      requirement: 処理活動の記録
      implementation:
        - 処理活動台帳
        - 定期的な更新

    article_32_security:
      requirement: 処理のセキュリティ
      implementation: Doc-SC-001〜003参照

    article_33_breach_notification:
      requirement: データ侵害通知（監督当局）
      timeline: 72時間以内
      implementation:
        - インシデント対応プロセス
        - 通知テンプレート

    article_34_breach_communication:
      requirement: データ侵害通知（データ主体）
      timeline: 遅滞なく
      implementation:
        - 影響を受けた個人への通知
        - 通知システム

    article_35_dpia:
      requirement: データ保護影響評価
      triggers:
        - 大規模なプロファイリング
        - 機密データの処理
        - 新技術の使用
      implementation: DPIA プロセス

    article_37_dpo:
      requirement: データ保護責任者
      appointment: 必須（大規模処理）
      implementation: DPO任命・連絡先公開

  chapter_5_transfers:
    article_44_general_principle:
      requirement: 十分性認定または適切な保護措置
      implementation:
        - EU-米国データプライバシーフレームワーク
        - 標準契約条項（SCC）
        - 補足措置
```

#### 2.1.2 GDPR技術実装

```typescript
// compliance/gdpr/data-subject-rights.ts

interface DataSubjectRequest {
  id: string;
  type: 'access' | 'rectification' | 'erasure' | 'restriction' | 'portability' | 'objection';
  userId: string;
  status: 'pending' | 'in_progress' | 'completed' | 'rejected';
  requestedAt: Date;
  dueDate: Date;
  completedAt?: Date;
  notes?: string;
}

class DataSubjectRightsService {
  private userRepository: UserRepository;
  private dataExportService: DataExportService;
  private auditLogger: AuditLogger;
  private notificationService: NotificationService;

  // アクセス権（第15条）
  async handleAccessRequest(userId: string, requestId: string): Promise<{
    userData: UserDataExport;
    processingInfo: ProcessingInformation;
  }> {
    // 本人確認
    await this.verifyIdentity(userId, requestId);

    // ユーザーデータの収集
    const userData = await this.collectUserData(userId);

    // 処理情報の収集
    const processingInfo: ProcessingInformation = {
      purposes: await this.getProcessingPurposes(userId),
      categories: await this.getDataCategories(userId),
      recipients: await this.getDataRecipients(userId),
      retentionPeriod: await this.getRetentionPeriod(userId),
      rights: this.getDataSubjectRights(),
      source: await this.getDataSource(userId),
      automatedDecisionMaking: await this.getAutomatedDecisionInfo(userId),
    };

    // 監査ログ
    await this.auditLogger.log({
      action: 'DATA_SUBJECT_ACCESS',
      userId,
      requestId,
      timestamp: new Date(),
    });

    return { userData, processingInfo };
  }

  // 削除権（第17条）
  async handleErasureRequest(userId: string, requestId: string): Promise<{
    success: boolean;
    deletedData: string[];
    retainedData: { category: string; reason: string }[];
  }> {
    // 本人確認
    await this.verifyIdentity(userId, requestId);

    // 削除可能なデータの特定
    const deletableData = await this.identifyDeletableData(userId);

    // 法的保持義務のあるデータの特定
    const retainedData = await this.identifyRetainedData(userId);

    // 削除の実行
    const deletedData: string[] = [];
    for (const dataCategory of deletableData) {
      await this.deleteData(userId, dataCategory);
      deletedData.push(dataCategory);
    }

    // アカウントの無効化
    await this.userRepository.deactivate(userId, {
      reason: 'gdpr_erasure_request',
      requestId,
    });

    // 第三者への通知（共有先がある場合）
    await this.notifyThirdParties(userId, 'erasure');

    // 監査ログ
    await this.auditLogger.log({
      action: 'DATA_SUBJECT_ERASURE',
      userId,
      requestId,
      deletedData,
      retainedData,
      timestamp: new Date(),
    });

    return {
      success: true,
      deletedData,
      retainedData,
    };
  }

  // データポータビリティ（第20条）
  async handlePortabilityRequest(
    userId: string,
    requestId: string,
    format: 'json' | 'csv'
  ): Promise<{
    downloadUrl: string;
    expiresAt: Date;
  }> {
    // 本人確認
    await this.verifyIdentity(userId, requestId);

    // ポータブルデータの収集（同意またはcontractベースのデータのみ）
    const portableData = await this.collectPortableData(userId);

    // エクスポートファイルの生成
    const exportFile = await this.dataExportService.generateExport(
      portableData,
      format
    );

    // セキュアなダウンロードURLの生成
    const downloadUrl = await this.generateSecureDownloadUrl(exportFile);
    const expiresAt = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000); // 7日間

    // 監査ログ
    await this.auditLogger.log({
      action: 'DATA_SUBJECT_PORTABILITY',
      userId,
      requestId,
      format,
      timestamp: new Date(),
    });

    return { downloadUrl, expiresAt };
  }

  // 処理活動の記録（第30条）
  async getRecordsOfProcessingActivities(): Promise<ProcessingActivityRecord[]> {
    return [
      {
        activity: 'ユーザー登録',
        purpose: '契約履行、サービス提供',
        lawfulBasis: 'contract',
        dataCategories: ['氏名', 'メールアドレス', '電話番号'],
        dataSubjects: ['顧客'],
        recipients: ['内部システム'],
        transfers: null,
        retentionPeriod: 'アカウント有効期間 + 7年',
        securityMeasures: '暗号化、アクセス制御',
      },
      {
        activity: '予約処理',
        purpose: '契約履行',
        lawfulBasis: 'contract',
        dataCategories: ['氏名', '連絡先', '旅行詳細', '決済情報'],
        dataSubjects: ['顧客', '旅行者'],
        recipients: ['宿泊施設', '決済プロバイダー'],
        transfers: '日本国内',
        retentionPeriod: '取引完了 + 7年',
        securityMeasures: '暗号化、トークナイゼーション',
      },
      {
        activity: 'マーケティング',
        purpose: '同意に基づくマーケティング',
        lawfulBasis: 'consent',
        dataCategories: ['メールアドレス', '嗜好情報'],
        dataSubjects: ['顧客'],
        recipients: ['メール配信プロバイダー'],
        transfers: 'SCC（EU-米国）',
        retentionPeriod: '同意撤回まで',
        securityMeasures: '暗号化、同意管理',
      },
    ];
  }

  private async collectUserData(userId: string): Promise<UserDataExport> {
    const user = await this.userRepository.findById(userId);
    const bookings = await this.bookingRepository.findByUserId(userId);
    const reviews = await this.reviewRepository.findByUserId(userId);
    const preferences = await this.preferenceRepository.findByUserId(userId);
    const activityLogs = await this.activityLogRepository.findByUserId(userId);

    return {
      profile: {
        id: user.id,
        email: user.email,
        name: user.name,
        phone: user.phone,
        createdAt: user.createdAt,
      },
      bookings: bookings.map(b => ({
        id: b.id,
        hotelName: b.hotelName,
        checkIn: b.checkIn,
        checkOut: b.checkOut,
        status: b.status,
        createdAt: b.createdAt,
      })),
      reviews: reviews.map(r => ({
        id: r.id,
        hotelName: r.hotelName,
        rating: r.rating,
        comment: r.comment,
        createdAt: r.createdAt,
      })),
      preferences,
      activityLogs: activityLogs.map(a => ({
        action: a.action,
        timestamp: a.timestamp,
        ipAddress: this.maskIpAddress(a.ipAddress),
      })),
    };
  }

  private async identifyRetainedData(userId: string): Promise<{ category: string; reason: string }[]> {
    const retained = [];

    // 税務記録（7年保持義務）
    const hasRecentTransactions = await this.hasRecentTransactions(userId);
    if (hasRecentTransactions) {
      retained.push({
        category: '取引記録',
        reason: '法的義務（税務記録保持）- 7年間',
      });
    }

    // 法的紛争
    const hasOpenDisputes = await this.hasOpenDisputes(userId);
    if (hasOpenDisputes) {
      retained.push({
        category: '紛争関連記録',
        reason: '法的義務（紛争解決）',
      });
    }

    return retained;
  }
}
```

### 2.2 CCPA/CPRA実装

#### 2.2.1 CCPA/CPRA要件マッピング

```yaml
ccpa_cpra_requirements:
  consumer_rights:
    right_to_know:
      description: 収集される個人情報の知る権利
      requirements:
        - 収集されるPIのカテゴリ
        - 収集目的
        - 第三者との共有
      implementation:
        - プライバシーポリシー
        - ジャストインタイム通知
        - アクセスリクエスト対応

    right_to_delete:
      description: 削除を要求する権利
      requirements:
        - 30日以内の対応
        - サービスプロバイダーへの通知
        - 例外の適用判断
      implementation:
        - 削除リクエストポータル
        - 自動化された削除プロセス

    right_to_opt_out:
      description: 個人情報の販売/共有からのオプトアウト
      requirements:
        - "Do Not Sell/Share" リンク
        - Global Privacy Control対応
        - 12ヶ月の有効期間
      implementation:
        - オプトアウトメカニズム
        - GPC信号の尊重

    right_to_correct:
      description: 不正確な情報の訂正権
      requirements:
        - 45日以内の対応
        - サービスプロバイダーへの通知
      implementation:
        - プロファイル編集機能

    right_to_limit_use:
      description: 機密PI使用の制限権
      requirements:
        - 機密PIの識別
        - 制限メカニズム
      implementation:
        - 機密データの同意管理

  business_obligations:
    privacy_notice:
      required_content:
        - 収集するPIのカテゴリ
        - 収集目的
        - 共有先カテゴリ
        - 消費者の権利
        - 連絡先情報
      update_frequency: 年次

    service_provider_contracts:
      required_clauses:
        - 使用目的の制限
        - 再共有の禁止
        - 監査権

    data_minimization:
      requirement: 必要最小限の収集・保持
      implementation:
        - データ収集レビュー
        - 保持期間ポリシー

    security:
      requirement: 合理的なセキュリティ措置
      implementation: Doc-SC-001〜003参照

  sensitive_personal_information:
    categories:
      - 社会保障番号
      - 運転免許証番号
      - 金融口座情報
      - 精密な位置情報
      - 人種/民族
      - 宗教
      - 健康情報
      - 性的指向
      - 遺伝情報
      - 生体情報
    restrictions:
      - 明示的な同意必要
      - 使用制限オプション必須
```

#### 2.2.2 CCPA技術実装

```typescript
// compliance/ccpa/ccpa-service.ts

interface CcpaRequest {
  id: string;
  type: 'know' | 'delete' | 'correct' | 'opt_out';
  consumerId: string;
  status: 'pending' | 'verified' | 'in_progress' | 'completed' | 'denied';
  requestedAt: Date;
  verifiedAt?: Date;
  completedAt?: Date;
  dueDate: Date;
}

class CcpaService {
  // "Do Not Sell or Share" オプトアウト
  async handleOptOut(consumerId: string, source: 'manual' | 'gpc'): Promise<void> {
    // オプトアウト設定の保存
    await this.privacyPreferenceRepository.save({
      consumerId,
      optOutOfSale: true,
      optOutOfSharing: true,
      source,
      effectiveDate: new Date(),
      expiresAt: new Date(Date.now() + 365 * 24 * 60 * 60 * 1000), // 12ヶ月
    });

    // 第三者への通知
    await this.notifyThirdParties(consumerId, 'opt_out');

    // 監査ログ
    await this.auditLogger.log({
      action: 'CCPA_OPT_OUT',
      consumerId,
      source,
      timestamp: new Date(),
    });
  }

  // Global Privacy Control シグナルの処理
  async handleGpcSignal(consumerId: string, gpcEnabled: boolean): Promise<void> {
    if (gpcEnabled) {
      await this.handleOptOut(consumerId, 'gpc');
    }
  }

  // "Right to Know" リクエスト
  async handleKnowRequest(consumerId: string, requestId: string): Promise<CcpaDisclosure> {
    // 身元確認（2要素必須）
    await this.verifyConsumerIdentity(consumerId, requestId);

    // 過去12ヶ月のデータ収集
    const collectedData = await this.getCollectedDataLast12Months(consumerId);

    // 開示レポート生成
    const disclosure: CcpaDisclosure = {
      categories_collected: collectedData.categories,
      specific_pieces: collectedData.specificPieces,
      sources: collectedData.sources,
      purposes: collectedData.purposes,
      third_parties_shared: collectedData.thirdParties,
      sold_or_shared: collectedData.soldOrShared,
    };

    return disclosure;
  }

  // プライバシー通知の生成
  generatePrivacyNotice(): CcpaPrivacyNotice {
    return {
      last_updated: new Date().toISOString(),
      categories_collected: [
        {
          category: 'Identifiers',
          examples: ['Name', 'Email', 'Phone number', 'IP address'],
          purposes: ['Service provision', 'Customer support'],
          sold: false,
          shared: false,
        },
        {
          category: 'Commercial Information',
          examples: ['Booking history', 'Transaction records'],
          purposes: ['Service provision', 'Personalization'],
          sold: false,
          shared: true,
          shared_with: ['Hotel partners'],
        },
        {
          category: 'Internet Activity',
          examples: ['Browsing history', 'Search queries'],
          purposes: ['Service improvement', 'Analytics'],
          sold: false,
          shared: false,
        },
        {
          category: 'Geolocation',
          examples: ['Approximate location'],
          purposes: ['Search results', 'Recommendations'],
          sold: false,
          shared: false,
        },
      ],
      consumer_rights: [
        'Right to Know',
        'Right to Delete',
        'Right to Correct',
        'Right to Opt-Out of Sale/Sharing',
        'Right to Limit Use of Sensitive PI',
        'Right to Non-Discrimination',
      ],
      contact: {
        email: 'privacy@triptrip.com',
        toll_free: '0120-xxx-xxx',
        web_form: 'https://triptrip.com/privacy/request',
      },
      financial_incentives: null,
      authorized_agent: {
        allowed: true,
        requirements: ['Written authorization', 'Identity verification'],
      },
    };
  }
}
```

### 2.3 日本APPI実装

#### 2.3.1 APPI要件マッピング

```yaml
appi_requirements:
  personal_information_handling:
    purpose_specification:
      requirement: 利用目的のできる限りの特定
      implementation:
        - 利用目的の明確化
        - プライバシーポリシーでの公表
        - 取得時の通知

    purpose_limitation:
      requirement: 利用目的の範囲内での使用
      implementation:
        - 目的外使用の禁止
        - 目的変更時の同意取得

    proper_acquisition:
      requirement: 適正な取得
      implementation:
        - 詐欺的手段の禁止
        - 要配慮個人情報の原則同意取得

    accurate_data:
      requirement: データ内容の正確性確保
      implementation:
        - 定期的な更新促進
        - 訂正機能の提供

    security_measures:
      requirement: 安全管理措置
      categories:
        organizational:
          - 組織体制の整備
          - 規程の整備
          - 従業者教育
        physical:
          - 入退室管理
          - 機器の盗難防止
        technical:
          - アクセス制御
          - 不正アクセス防止
          - 暗号化
      implementation: Doc-SC-001〜003参照

    employee_supervision:
      requirement: 従業者の監督
      implementation:
        - 教育・研修
        - 誓約書
        - アクセス制限

    subcontractor_supervision:
      requirement: 委託先の監督
      implementation:
        - 委託先選定基準
        - 契約条項
        - 定期監査

  data_subject_rights:
    disclosure_request:
      requirement: 本人からの開示請求への対応
      timeline: 遅滞なく
      fee: 合理的な範囲で徴収可能
      implementation:
        - 開示請求フォーム
        - 本人確認プロセス
        - 開示レポート生成

    correction_request:
      requirement: 訂正・追加・削除請求への対応
      timeline: 遅滞なく
      implementation:
        - プロファイル編集機能
        - 訂正リクエストフォーム

    suspension_request:
      requirement: 利用停止・消去請求への対応
      conditions:
        - 目的外利用
        - 不正取得
        - 第三者提供違反
      implementation:
        - 利用停止機能
        - 消去プロセス

    opt_out_provision:
      requirement: 第三者提供におけるオプトアウト
      requirements:
        - 個人情報保護委員会への届出
        - 本人への事前通知
      implementation:
        - オプトアウト機能
        - 届出管理

  third_party_provision:
    principle:
      requirement: 原則として本人同意
      exceptions:
        - 法令に基づく場合
        - 人の生命等の保護
        - 公衆衛生・児童の健全育成
        - 国の機関等への協力

    record_keeping:
      requirement: 提供・受領記録の作成・保存
      retention: 3年間
      implementation:
        - 自動記録生成
        - 監査証跡

    overseas_transfer:
      requirement: 外国第三者への提供制限
      conditions:
        - 本人の同意
        - 適切な体制を有する第三者
        - 同等の保護措置のある国
      implementation:
        - 越境移転評価
        - SCC相当の契約
```

---

## 第3章：セキュリティ認証

### 3.1 PCI-DSS準拠

```yaml
pci_dss_compliance:
  scope_determination:
    cardholder_data_environment:
      status: Out of Scope
      reason: Stripeによるトークナイゼーション

    applicable_saq: SAQ A
    description: カード情報を処理・保存・伝送しない加盟店

  saq_a_requirements:
    requirement_2:
      title: ベンダーデフォルトの変更
      status: 準拠
      evidence:
        - パスワードポリシー
        - システム設定手順

    requirement_6:
      title: セキュアシステムの開発・維持
      status: 準拠
      evidence:
        - セキュアSDLC
        - 脆弱性管理

    requirement_8:
      title: ユーザーアクセスの識別・認証
      status: 準拠
      evidence:
        - MFA実装
        - パスワードポリシー

    requirement_9:
      title: 物理アクセスの制限
      status: N/A (クラウド)
      evidence:
        - AWS SOC2レポート

    requirement_12:
      title: セキュリティポリシー
      status: 準拠
      evidence:
        - 情報セキュリティポリシー
        - インシデント対応計画

  attestation_of_compliance:
    frequency: 年次
    responsible_party: QSA または 内部評価
    next_assessment: 2026-12
```

### 3.2 SOC2 Type II準備

#### 3.2.1 SOC2 Trust Service Criteria

```yaml
soc2_trust_principles:
  security:
    description: 不正アクセス・使用・変更からの保護
    common_criteria:
      cc1_control_environment:
        controls:
          - cc1.1: 組織のコミットメント
          - cc1.2: 取締役会の監督
          - cc1.3: 組織構造
          - cc1.4: 人事ポリシー
          - cc1.5: 責任の定義
        status: 実装中

      cc2_communication:
        controls:
          - cc2.1: 情報の品質
          - cc2.2: 内部コミュニケーション
          - cc2.3: 外部コミュニケーション
        status: 実装中

      cc3_risk_assessment:
        controls:
          - cc3.1: 目標の特定
          - cc3.2: リスクの識別・分析
          - cc3.3: 不正リスクの評価
          - cc3.4: 重大な変更の識別
        status: 実装中

      cc4_monitoring:
        controls:
          - cc4.1: 継続的モニタリング
          - cc4.2: 是正措置
        status: 実装中

      cc5_control_activities:
        controls:
          - cc5.1: リスク緩和活動
          - cc5.2: テクノロジー管理
          - cc5.3: セキュリティポリシー
        status: 実装中

      cc6_logical_access:
        controls:
          - cc6.1: セキュリティソフトウェア
          - cc6.2: ユーザー認証
          - cc6.3: アクセス権限
          - cc6.4: アクセス制限
          - cc6.5: 物理アクセス
          - cc6.6: システム境界の保護
          - cc6.7: インターネット脅威
          - cc6.8: 不正ソフトウェア
        status: 実装中

      cc7_system_operations:
        controls:
          - cc7.1: 脆弱性管理
          - cc7.2: セキュリティイベント検知
          - cc7.3: インシデント対応
          - cc7.4: 復旧
          - cc7.5: インシデント分析
        status: 実装中

      cc8_change_management:
        controls:
          - cc8.1: 変更管理プロセス
        status: 実装中

      cc9_risk_mitigation:
        controls:
          - cc9.1: ベンダー管理
          - cc9.2: 事業継続
        status: 実装中

  availability:
    description: サービスの可用性
    criteria:
      - a1.1: 可用性コミットメント
      - a1.2: 環境保護
      - a1.3: バックアップ
    status: 実装中

  processing_integrity:
    description: 処理の完全性
    criteria:
      - pi1.1: 処理の正確性・完全性
      - pi1.2: システム入力
      - pi1.3: 処理活動
      - pi1.4: システム出力
      - pi1.5: 入出力の完全性
    status: 実装中

  confidentiality:
    description: 機密情報の保護
    criteria:
      - c1.1: 機密情報の識別
      - c1.2: 機密情報の廃棄
    status: 実装中

  privacy:
    description: 個人情報のプライバシー
    criteria:
      - p1: 通知
      - p2: 選択と同意
      - p3: 収集
      - p4: 使用・保持・廃棄
      - p5: アクセス
      - p6: 開示
      - p7: 品質
      - p8: 監視・執行
    status: 実装中
```

#### 3.2.2 SOC2監査準備

```yaml
soc2_audit_preparation:
  timeline:
    phase_1_gap_assessment:
      duration: 月1-2
      activities:
        - 現状評価
        - ギャップ分析
        - 是正計画策定

    phase_2_remediation:
      duration: 月3-6
      activities:
        - コントロール実装
        - ポリシー・手順整備
        - 証跡収集体制構築

    phase_3_readiness:
      duration: 月7-9
      activities:
        - 内部監査
        - 是正
        - 監査準備

    phase_4_type1_audit:
      duration: 月10-11
      activities:
        - Type I監査実施
        - 指摘事項対応

    phase_5_observation_period:
      duration: 月12-17
      activities:
        - コントロール運用
        - 証跡継続収集

    phase_6_type2_audit:
      duration: 月18-19
      activities:
        - Type II監査実施
        - レポート取得

  evidence_collection:
    automated:
      - アクセスログ
      - 変更管理チケット
      - 脆弱性スキャン結果
      - インシデントチケット
      - バックアップログ

    manual:
      - ポリシー承認記録
      - リスク評価ドキュメント
      - 従業員研修記録
      - ベンダー評価記録

  auditor_selection:
    criteria:
      - AICPA認定
      - SaaS企業監査経験
      - 日本語対応
    candidates:
      - Deloitte
      - KPMG
      - EY
      - PwC
```

### 3.3 ISO 27001検討

```yaml
iso_27001_consideration:
  timeline: Year 3（SOC2 Type II取得後）

  rationale:
    benefits:
      - 国際的な認知度
      - 日本市場での信頼性
      - 包括的なISMS
    challenges:
      - 追加の工数
      - 維持コスト
      - SOC2との重複

  approach:
    strategy: SOC2コントロールのマッピング
    additional_requirements:
      - リスクアセスメント手法
      - 適用宣言書（SoA）
      - ISMS文書体系
```

---

## 第4章：データ主体の権利

### 4.1 アクセス権 & ポータビリティ

#### 4.1.1 データ主体権利ポータル

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    DATA SUBJECT RIGHTS PORTAL                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  User Interface (Self-Service Portal)                                    │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │    │
│  │   │ Access       │  │ Portability  │  │ Deletion     │  │ Correct   │  │    │
│  │   │ Request      │  │ Request      │  │ Request      │  │ Request   │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────┘  └───────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                     │                                           │
│                                     ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  Identity Verification                                                   │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │    │
│  │   │ Email OTP    │  │ Phone OTP    │  │ Document     │                 │    │
│  │   │ (Factor 1)   │  │ (Factor 2)   │  │ Upload       │                 │    │
│  │   └──────────────┘  └──────────────┘  └──────────────┘                 │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                     │                                           │
│                                     ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  Request Processing Engine                                               │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │    │
│  │   │ Request      │  │ Workflow     │  │ Data         │  │ Response  │  │    │
│  │   │ Validation   │  │ Orchestrator │  │ Collector    │  │ Generator │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────┘  └───────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                     │                                           │
│                                     ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  Data Sources                                                            │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │    │
│  │   │ User         │  │ Booking      │  │ Activity     │  │ Third     │  │    │
│  │   │ Database     │  │ Database     │  │ Logs         │  │ Party     │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────┘  └───────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                     │                                           │
│                                     ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  Audit & Compliance                                                      │    │
│  │                                                                          │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │    │
│  │   │ Audit Log    │  │ SLA          │  │ Compliance   │                 │    │
│  │   │              │  │ Monitoring   │  │ Reporting    │                 │    │
│  │   └──────────────┘  └──────────────┘  └──────────────┘                 │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 4.1.2 データエクスポートサービス

```typescript
// compliance/data-export/export-service.ts

interface DataExportRequest {
  userId: string;
  format: 'json' | 'csv' | 'pdf';
  dataCategories: DataCategory[];
  jurisdiction: 'gdpr' | 'ccpa' | 'appi';
}

interface DataCategory {
  name: string;
  include: boolean;
  dateRange?: {
    start: Date;
    end: Date;
  };
}

class DataExportService {
  async generateExport(request: DataExportRequest): Promise<ExportResult> {
    const exportId = this.generateExportId();

    // 1. データ収集
    const collectedData = await this.collectData(request);

    // 2. フォーマット変換
    let exportedData: Buffer;
    switch (request.format) {
      case 'json':
        exportedData = await this.toJson(collectedData);
        break;
      case 'csv':
        exportedData = await this.toCsv(collectedData);
        break;
      case 'pdf':
        exportedData = await this.toPdf(collectedData, request.jurisdiction);
        break;
    }

    // 3. 暗号化
    const encryptedData = await this.encryptExport(exportedData, request.userId);

    // 4. セキュアストレージにアップロード
    const downloadUrl = await this.uploadToSecureStorage(
      exportId,
      encryptedData,
      request.userId
    );

    // 5. 通知
    await this.notifyUser(request.userId, {
      exportId,
      downloadUrl,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    });

    return {
      exportId,
      downloadUrl,
      format: request.format,
      size: exportedData.length,
      generatedAt: new Date(),
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    };
  }

  private async collectData(request: DataExportRequest): Promise<CollectedData> {
    const data: CollectedData = {};

    for (const category of request.dataCategories) {
      if (!category.include) continue;

      switch (category.name) {
        case 'profile':
          data.profile = await this.collectProfileData(request.userId);
          break;
        case 'bookings':
          data.bookings = await this.collectBookingData(
            request.userId,
            category.dateRange
          );
          break;
        case 'reviews':
          data.reviews = await this.collectReviewData(request.userId);
          break;
        case 'preferences':
          data.preferences = await this.collectPreferenceData(request.userId);
          break;
        case 'activity':
          data.activity = await this.collectActivityData(
            request.userId,
            category.dateRange
          );
          break;
        case 'consent':
          data.consent = await this.collectConsentData(request.userId);
          break;
      }
    }

    return data;
  }

  private async toPdf(
    data: CollectedData,
    jurisdiction: string
  ): Promise<Buffer> {
    // 法域に応じたPDFテンプレートを使用
    const template = await this.getTemplate(jurisdiction);

    const pdfContent = await this.pdfGenerator.generate({
      template,
      data: {
        ...data,
        generatedAt: new Date().toISOString(),
        jurisdiction,
        processingInformation: await this.getProcessingInformation(jurisdiction),
      },
    });

    return pdfContent;
  }

  private async encryptExport(data: Buffer, userId: string): Promise<Buffer> {
    // ユーザー固有の暗号化キーで暗号化
    const encryptionKey = await this.deriveUserKey(userId);
    return this.encryptionService.encrypt(data, encryptionKey);
  }
}
```

### 4.2 削除権（忘れられる権利）

```yaml
right_to_erasure:
  process:
    1_request_reception:
      channels:
        - セルフサービスポータル
        - メール（privacy@triptrip.com）
        - カスタマーサポート
      information_required:
        - 氏名
        - メールアドレス
        - リクエスト詳細

    2_identity_verification:
      methods:
        - メールOTP
        - 電話OTP（2要素目）
        - 必要に応じて本人確認書類
      timeline: 48時間以内

    3_request_evaluation:
      criteria:
        - 適用法域の確認
        - 例外の該当性確認
        - 保持義務の確認
      timeline: 5営業日以内

    4_execution:
      actions:
        deletable_data:
          - プロファイル情報
          - 嗜好設定
          - 検索履歴
          - Cookie/トラッキングデータ

        retained_data:
          - 取引記録（法的保持義務）
          - 監査ログ（セキュリティ目的）
          - 匿名化された分析データ

      timeline: 30日以内（GDPR）/ 45日以内（CCPA）

    5_third_party_notification:
      recipients:
        - データ共有先パートナー
        - サービスプロバイダー
        - 検索エンジン（インデックス削除リクエスト）

    6_confirmation:
      contents:
        - 削除完了通知
        - 削除されたデータカテゴリ
        - 保持されたデータ（理由含む）
        - 今後の権利行使方法

  exceptions:
    legal_obligation:
      description: 法令に基づく保持義務
      examples:
        - 税務記録（7年）
        - 契約書類（時効期間）

    legal_claims:
      description: 法的請求の行使・防御
      examples:
        - 係争中のトランザクション
        - 不正利用調査

    public_interest:
      description: 公益目的
      examples:
        - 統計・調査研究
```

### 4.3 同意管理

#### 4.3.1 同意管理プラットフォーム（CMP）

```typescript
// compliance/consent/consent-management.ts

interface ConsentRecord {
  id: string;
  userId: string;
  purpose: ConsentPurpose;
  granted: boolean;
  timestamp: Date;
  source: 'explicit' | 'implicit' | 'api';
  version: string;
  ipAddress: string;
  userAgent: string;
  expiresAt?: Date;
}

type ConsentPurpose =
  | 'essential'
  | 'functionality'
  | 'analytics'
  | 'marketing'
  | 'personalization'
  | 'third_party_sharing';

interface ConsentPreferences {
  userId: string;
  preferences: Record<ConsentPurpose, boolean>;
  updatedAt: Date;
}

class ConsentManagementService {
  private consentRepository: ConsentRepository;
  private auditLogger: AuditLogger;

  // 同意の記録
  async recordConsent(
    userId: string,
    purpose: ConsentPurpose,
    granted: boolean,
    context: {
      source: 'explicit' | 'implicit' | 'api';
      ipAddress: string;
      userAgent: string;
    }
  ): Promise<ConsentRecord> {
    const record: ConsentRecord = {
      id: this.generateId(),
      userId,
      purpose,
      granted,
      timestamp: new Date(),
      source: context.source,
      version: await this.getCurrentPolicyVersion(),
      ipAddress: context.ipAddress,
      userAgent: context.userAgent,
      expiresAt: this.calculateExpiry(purpose),
    };

    await this.consentRepository.save(record);

    // 監査ログ
    await this.auditLogger.log({
      action: granted ? 'CONSENT_GRANTED' : 'CONSENT_WITHDRAWN',
      userId,
      purpose,
      timestamp: record.timestamp,
    });

    // 同意変更に基づくアクション
    await this.handleConsentChange(userId, purpose, granted);

    return record;
  }

  // 同意状況の確認
  async checkConsent(userId: string, purpose: ConsentPurpose): Promise<boolean> {
    const latestConsent = await this.consentRepository.findLatest(userId, purpose);

    if (!latestConsent) {
      return this.getDefaultConsent(purpose);
    }

    // 有効期限チェック
    if (latestConsent.expiresAt && latestConsent.expiresAt < new Date()) {
      return this.getDefaultConsent(purpose);
    }

    // ポリシーバージョンチェック
    const currentVersion = await this.getCurrentPolicyVersion();
    if (this.requiresReconsent(latestConsent.version, currentVersion, purpose)) {
      return this.getDefaultConsent(purpose);
    }

    return latestConsent.granted;
  }

  // 一括同意更新（Cookie Banner等）
  async updateBulkConsent(
    userId: string,
    preferences: Record<ConsentPurpose, boolean>,
    context: {
      source: 'explicit' | 'api';
      ipAddress: string;
      userAgent: string;
    }
  ): Promise<void> {
    for (const [purpose, granted] of Object.entries(preferences)) {
      await this.recordConsent(userId, purpose as ConsentPurpose, granted, context);
    }
  }

  // Cookie同意バナー設定
  getCookieConsentConfig(): CookieConsentConfig {
    return {
      categories: [
        {
          id: 'essential',
          name: '必須Cookie',
          description: 'サービスの基本機能に必要です',
          required: true,
          cookies: ['session_id', 'csrf_token'],
        },
        {
          id: 'functionality',
          name: '機能Cookie',
          description: '言語設定などの機能を提供します',
          required: false,
          cookies: ['language', 'currency'],
        },
        {
          id: 'analytics',
          name: '分析Cookie',
          description: 'サービス改善のための分析に使用します',
          required: false,
          cookies: ['_ga', '_gid'],
        },
        {
          id: 'marketing',
          name: 'マーケティングCookie',
          description: '関連性の高い広告を表示するために使用します',
          required: false,
          cookies: ['_fbp', '_gcl_au'],
        },
      ],
      defaultConsent: {
        essential: true,
        functionality: false,
        analytics: false,
        marketing: false,
      },
      gpcEnabled: true,  // Global Privacy Control対応
    };
  }

  // 同意変更時のアクション
  private async handleConsentChange(
    userId: string,
    purpose: ConsentPurpose,
    granted: boolean
  ): Promise<void> {
    if (!granted) {
      switch (purpose) {
        case 'marketing':
          // マーケティングリストから削除
          await this.marketingService.unsubscribe(userId);
          break;
        case 'analytics':
          // 分析データの匿名化
          await this.analyticsService.anonymize(userId);
          break;
        case 'personalization':
          // パーソナライゼーションデータの削除
          await this.personalizationService.clearProfile(userId);
          break;
        case 'third_party_sharing':
          // 第三者への共有停止
          await this.thirdPartyService.stopSharing(userId);
          break;
      }
    }
  }

  // 同意履歴の取得
  async getConsentHistory(userId: string): Promise<ConsentRecord[]> {
    return this.consentRepository.findAllByUserId(userId);
  }

  // プルーフ・オブ・コンセント
  async getProofOfConsent(
    userId: string,
    purpose: ConsentPurpose
  ): Promise<ConsentProof> {
    const records = await this.consentRepository.findByUserIdAndPurpose(
      userId,
      purpose
    );

    return {
      userId,
      purpose,
      currentStatus: records[0]?.granted ?? false,
      history: records.map(r => ({
        action: r.granted ? 'granted' : 'withdrawn',
        timestamp: r.timestamp,
        source: r.source,
        policyVersion: r.version,
      })),
      generatedAt: new Date(),
    };
  }
}
```

---

## 第5章：コンプライアンス運用

### 5.1 継続的監視

#### 5.1.1 コンプライアンスモニタリング

```yaml
compliance_monitoring:
  automated_checks:
    data_retention:
      frequency: 日次
      checks:
        - 保持期間超過データの検知
        - 自動削除の実行確認
        - 例外の記録

    consent_validity:
      frequency: 日次
      checks:
        - 期限切れ同意の検知
        - ポリシー変更後の再同意要求
        - 同意なしデータ処理の検知

    access_control:
      frequency: リアルタイム
      checks:
        - 不正アクセス試行
        - 権限外アクセス
        - 異常なデータアクセスパターン

    encryption:
      frequency: 日次
      checks:
        - 暗号化カバレッジ
        - 鍵ローテーション状況
        - 証明書有効期限

  dashboards:
    compliance_overview:
      metrics:
        - 規制別準拠スコア
        - DSRリクエスト状況
        - インシデント件数
        - 監査指摘事項

    privacy_metrics:
      metrics:
        - 同意率
        - オプトアウト率
        - データ主体リクエスト対応時間
        - 侵害通知件数

  alerts:
    critical:
      - データ侵害検知
      - コンプライアンス違反
      - SLA違反
    high:
      - DSRリクエスト期限接近
      - 証明書期限切れ警告
      - 異常なデータアクセス
    medium:
      - ポリシー更新必要
      - 定期レビュー期限
```

### 5.2 監査証跡

```yaml
audit_trail:
  covered_activities:
    data_access:
      - PIIデータへのアクセス
      - 機密データの閲覧
      - データエクスポート

    data_modification:
      - ユーザーデータの作成・更新・削除
      - 同意設定の変更
      - 権限変更

    authentication:
      - ログイン/ログアウト
      - MFA検証
      - パスワード変更

    system_changes:
      - 設定変更
      - デプロイメント
      - セキュリティポリシー変更

  log_attributes:
    required:
      - timestamp (ISO 8601)
      - user_id
      - action
      - resource
      - result (success/failure)
      - ip_address
      - user_agent

    optional:
      - session_id
      - request_id
      - additional_context

  retention:
    default: 7年
    by_regulation:
      gdpr: 7年
      pci_dss: 1年（最低）
      sox: 7年

  protection:
    integrity:
      - 改ざん防止（追記のみ）
      - ハッシュチェーン
      - デジタル署名

    confidentiality:
      - アクセス制御
      - 暗号化

    availability:
      - 冗長化
      - バックアップ
```

### 5.3 データ保護影響評価（DPIA）

```yaml
dpia_process:
  triggers:
    mandatory:
      - 新規の大規模個人データ処理
      - 自動化された意思決定
      - 機密データの処理
      - 大規模なプロファイリング
      - 公共のアクセス可能な場所での監視
      - 新技術の使用

    recommended:
      - 既存処理の大幅な変更
      - 新規の第三者共有
      - 新規の越境移転

  assessment_steps:
    1_description:
      contents:
        - 処理の性質、範囲、コンテキスト、目的
        - 処理するデータのカテゴリ
        - データ主体のカテゴリ
        - データの流れ

    2_necessity_assessment:
      criteria:
        - 処理の必要性
        - 目的に対する比例性
        - 適法根拠

    3_risk_identification:
      risk_categories:
        - データ主体への物理的リスク
        - データ主体への財務的リスク
        - 名誉・評判へのリスク
        - 差別のリスク
        - 自由への制限

    4_risk_evaluation:
      factors:
        - 発生可能性
        - 影響の深刻度
      matrix:
        - Low/Low: 許容
        - Low/High: 緩和策必要
        - High/Low: 緩和策推奨
        - High/High: 緩和策必須

    5_mitigation_measures:
      types:
        - 技術的措置
        - 組織的措置
        - 契約上の措置
        - 透明性の向上

    6_documentation:
      contents:
        - 評価結果
        - 緩和策
        - 残存リスク
        - 承認記録

    7_consultation:
      conditions:
        - 高リスクが残存する場合
        - 監督当局への事前相談
```

---

## 第6章：実装ロードマップ & 文書間参照

### 6.1 実装ロードマップ

```yaml
implementation_roadmap:
  phase_1_foundation:
    timeline: 月1-3
    focus: 基盤整備
    deliverables:
      - プライバシーポリシーの整備
      - Cookie同意バナー実装
      - 処理活動記録の作成
      - DPO任命
    success_criteria:
      - プライバシーポリシー公開
      - 同意管理機能稼働

  phase_2_rights:
    timeline: 月4-6
    focus: データ主体の権利
    deliverables:
      - DSRポータル実装
      - データエクスポート機能
      - 削除機能
      - 訂正機能
    success_criteria:
      - DSRリクエスト対応可能
      - 30日SLA達成

  phase_3_automation:
    timeline: 月7-9
    focus: 自動化
    deliverables:
      - コンプライアンスモニタリング
      - 自動データ保持管理
      - アラート設定
    success_criteria:
      - 自動化率 >50%
      - リアルタイム監視稼働

  phase_4_certification:
    timeline: 月10-12
    focus: 認証準備
    deliverables:
      - PCI-DSS SAQ A完了
      - SOC2 Type I準備
      - 内部監査実施
    success_criteria:
      - PCI-DSS準拠
      - SOC2 Type I監査開始

  phase_5_maturity:
    timeline: Year 2
    focus: 成熟化
    deliverables:
      - SOC2 Type II取得
      - グローバル展開対応
      - 継続的改善
    success_criteria:
      - SOC2 Type II取得
      - 監査指摘 0件
```

### 6.2 文書間参照

```yaml
document_references:
  inputs:
    - document: Doc-SC-001
      title: セキュリティアーキテクチャ＆戦略
      relevance: セキュリティコントロール
      key_integrations:
        - 技術的措置の参照
        - インシデント対応

    - document: Doc-SC-002
      title: ID＆アクセス管理
      relevance: アクセス制御
      key_integrations:
        - 認証・認可
        - 監査ログ

    - document: Doc-SC-003
      title: データ保護＆暗号化
      relevance: データ保護措置
      key_integrations:
        - 暗号化実装
        - 鍵管理

    - document: Doc-DA-005
      title: データセキュリティ、プライバシー＆ガバナンス
      relevance: データガバナンス
      key_integrations:
        - データ分類
        - 保持ポリシー

  outputs:
    - 外部監査レポート
    - コンプライアンスダッシュボード
    - 規制当局報告
```

### 6.3 コンプライアンス技術スタック

```yaml
compliance_technology_stack:
  consent_management:
    platform: Custom CMP
    features:
      - Cookie同意
      - 同意記録
      - GPC対応

  dsr_management:
    platform: Custom Portal
    features:
      - リクエスト受付
      - ワークフロー
      - レポーティング

  data_discovery:
    tools:
      - AWS Macie
      - Custom scanners
    purpose: PII検出

  compliance_monitoring:
    platform: Custom Dashboard + AWS Security Hub
    features:
      - リアルタイム監視
      - アラート
      - レポート

  audit_logging:
    platform: CloudWatch Logs + OpenSearch
    retention: 7年

  policy_management:
    platform: Confluence/Notion
    versioning: Git-based
```

---

## 付録

### A. 規制対照表

| 要件 | GDPR | CCPA/CPRA | APPI |
|------|------|-----------|------|
| アクセス権 | Art.15 | §1798.110 | 第33条 |
| 削除権 | Art.17 | §1798.105 | 第33条 |
| 訂正権 | Art.16 | §1798.106 | 第34条 |
| ポータビリティ | Art.20 | §1798.130 | - |
| オプトアウト | Art.21 | §1798.120 | 第27条 |
| 同意 | Art.7 | §1798.135 | 第20条 |
| 通知義務 | Art.33-34 | §1798.82 | 第26条 |
| DPO | Art.37-39 | - | - |
| DPIA | Art.35 | - | - |

### B. 対応期限一覧

| リクエストタイプ | GDPR | CCPA/CPRA | APPI |
|----------------|------|-----------|------|
| アクセス | 30日 | 45日 | 遅滞なく |
| 削除 | 30日 | 45日 | 遅滞なく |
| 訂正 | 30日 | 45日 | 遅滞なく |
| 侵害通知（当局） | 72時間 | - | 遅滞なく |
| 侵害通知（本人） | 遅滞なく | - | 遅滞なく |

### C. 変更履歴

| バージョン | 日付 | 変更内容 | 作成者 |
|-----------|------|---------|--------|
| 1.0.0 | 2026-01-20 | 初版作成 | Technical Architecture Agent |

---

**文書情報**
- 文書ID: Doc-SC-004
- バージョン: 1.0.0
- ステータス: ドラフト
- 最終更新: 2026-01-20
- 次回レビュー: 2026-02-20
- 行数: 約2,000行

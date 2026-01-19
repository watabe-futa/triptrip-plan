# Doc-SA-004: リアルタイム・協調機能アーキテクチャ

## エグゼクティブサマリー

本文書は、TripTripプラットフォームのリアルタイム通信と協調機能のアーキテクチャを包括的に定義します。WebSocketアーキテクチャ、協調旅程編集（CRDT/Operational Transformation）、リアルタイム通知システム、プレゼンス検出、オフラインファースト同期を詳述し、Google Docs、Figma、Slackレベルのリアルタイムコラボレーション体験を実現します。既存のFlutter/Hiveによるオフラインデータ永続化を基盤として、グループ旅行の共同計画、リアルタイム在庫更新、インスタントメッセージングを可能にする堅牢なリアルタイム基盤を構築します。本設計は、同時接続100万ユーザー、メッセージ配信レイテンシ50ms以下を目標とします。

---

## 第1章：はじめに & コンテキスト

### 1.1 リアルタイム機能のビジネス要件

TripTripのリアルタイム機能は、ユーザー体験の差別化と競争優位性の確立に不可欠です。旅行計画は本質的に協調的な活動であり、リアルタイムコラボレーションによって新しい価値を創出します。

```yaml
business_requirements:
  collaborative_trip_planning:
    description: 複数ユーザーによる旅程の共同編集
    use_cases:
      - グループ旅行の計画
      - カップル/家族での旅行調整
      - 旅行代理店とのリアルタイム相談
    requirements:
      - 同時編集の衝突解決
      - リアルタイムカーソル表示
      - 変更履歴の追跡
      - オフライン編集と同期
    priority: P1

  real_time_inventory:
    description: 在庫状況のリアルタイム反映
    use_cases:
      - ホテル空室状況の即時更新
      - チケット残数のリアルタイム表示
      - 価格変動の即時通知
    requirements:
      - 1秒以内の在庫更新反映
      - 競合時の在庫確保保証
      - フェアな在庫割り当て
    priority: P0

  instant_notifications:
    description: 即時通知配信
    use_cases:
      - 予約確定通知
      - 価格アラート
      - 旅程変更通知
      - グループメンバーのアクション通知
    requirements:
      - 配信レイテンシ < 500ms
      - マルチチャネル配信（Push、Email、SMS、In-App）
      - 配信保証（少なくとも1回）
    priority: P0

  presence_and_activity:
    description: オンライン状態とアクティビティの可視化
    use_cases:
      - 共同編集者のオンライン状態
      - 閲覧中のコンテンツの共有
      - タイピングインジケーター
    requirements:
      - 5秒以内のプレゼンス更新
      - バッテリー消費の最適化
      - プライバシー設定の尊重
    priority: P2

  offline_first:
    description: オフライン時も機能継続
    use_cases:
      - 機内での旅程確認
      - 電波の悪い地域での利用
      - バックグラウンドでの同期
    requirements:
      - オフライン時の読み取り機能
      - オフライン時の書き込みキュー
      - 復帰時の自動同期
      - コンフリクト解決
    priority: P1
```

### 1.2 技術的要件と制約

```yaml
technical_requirements:
  performance:
    concurrent_connections: 1,000,000
    message_latency_p99: 50ms
    reconnection_time: 3秒以内
    message_throughput: 100,000 msg/sec

  reliability:
    message_delivery: At-least-once
    connection_availability: 99.99%
    data_consistency: 結果整合性（CRDTによる保証）

  scalability:
    horizontal_scaling: true
    auto_scaling: true
    geographic_distribution: マルチリージョン

  compatibility:
    flutter_web: WebSocket + Fallback (SSE, Long Polling)
    flutter_mobile: WebSocket (native)
    browser_support: IE11+ (fallback), Modern browsers (native)

constraints:
  existing_infrastructure:
    - constraint: Flutter + Hiveによるローカルデータ永続化
      impact: オフラインファーストの基盤として活用
      strategy: Hiveスキーマとの同期プロトコル設計

    - constraint: Provider + Riverpodによる状態管理
      impact: リアルタイム状態の統合
      strategy: Riverpod StateNotifierとの連携

    - constraint: Hono/Node.js バックエンド
      impact: WebSocketサーバーの実装
      strategy: ws ライブラリまたは Socket.IO の採用

  mobile_constraints:
    - バッテリー消費の最小化
    - バックグラウンド接続の制限（iOS）
    - ネットワーク品質の変動
    - オフライン状態の頻発
```

### 1.3 技術目標と成功基準

```yaml
success_criteria:
  performance_kpis:
    message_latency:
      metric: P99 End-to-End Latency
      target: 50ms
      measurement: Distributed tracing

    connection_time:
      metric: Connection establishment time
      target: 1秒以内
      measurement: Client-side metrics

    reconnection_time:
      metric: Automatic reconnection time
      target: 3秒以内
      measurement: Client-side metrics

    throughput:
      metric: Messages per second
      target: 100,000 msg/sec
      measurement: Server metrics

  reliability_kpis:
    message_delivery_rate:
      metric: Successfully delivered messages
      target: 99.99%
      measurement: Message acknowledgment tracking

    connection_uptime:
      metric: WebSocket connection availability
      target: 99.99%
      measurement: Health checks

    sync_consistency:
      metric: Data consistency after sync
      target: 100%
      measurement: Consistency checks

  user_experience_kpis:
    offline_functionality:
      metric: Offline feature coverage
      target: 80% of read operations
      measurement: Feature audit

    sync_transparency:
      metric: Seamless sync without user intervention
      target: 95% of cases
      measurement: User feedback, error rates

    collaboration_smoothness:
      metric: Concurrent edit conflict rate
      target: < 1%
      measurement: Conflict resolution logs
```

---

## 第2章：リアルタイムアーキテクチャ概要

### 2.1 リアルタイム要件定義

```yaml
realtime_requirements:
  latency_classification:
    hard_realtime:
      latency: < 100ms
      use_cases:
        - 協調編集のカーソル位置
        - タイピングインジケーター
        - 在庫数のリアルタイム表示

    soft_realtime:
      latency: < 1秒
      use_cases:
        - 旅程変更の同期
        - 新しいコメントの表示
        - プレゼンス更新

    near_realtime:
      latency: < 5秒
      use_cases:
        - 通知配信
        - 価格更新
        - 分析データ更新

  data_flow_patterns:
    request_response:
      description: 従来のリクエスト/レスポンス
      use_case: CRUD操作、検索
      protocol: REST/GraphQL

    publish_subscribe:
      description: トピックベースのメッセージ配信
      use_case: 通知、価格アラート
      protocol: WebSocket + Pub/Sub

    bidirectional_streaming:
      description: 双方向のストリーミング
      use_case: 協調編集、チャット
      protocol: WebSocket

    server_sent_events:
      description: サーバーからのプッシュ
      use_case: ステータス更新、進捗表示
      protocol: SSE（WebSocketフォールバック）
```

### 2.2 技術選定

```yaml
technology_selection:
  websocket_server:
    choice: Socket.IO (Node.js)
    alternatives:
      - ws (raw WebSocket): 軽量だが機能不足
      - Ably: マネージドだがコスト高
      - Pusher: 機能制限、ベンダーロック
    rationale:
      - 自動再接続
      - 部屋（Room）機能
      - フォールバック対応
      - 広いブラウザサポート
      - Node.js エコシステムとの親和性

  message_broker:
    choice: Redis Pub/Sub + Redis Streams
    alternatives:
      - Apache Kafka: 過剰スペック（短期保持）
      - RabbitMQ: 複雑な設定
      - NATS: エコシステム未成熟
    rationale:
      - 低レイテンシ
      - サーバー間通信に最適
      - 既存Redis活用
      - シンプルな運用

  crdt_library:
    choice: Yjs
    alternatives:
      - Automerge: パフォーマンス問題
      - ShareDB (OT): 複雑な実装
    rationale:
      - 高性能
      - 豊富なデータ構造
      - オフラインサポート
      - アクティブなコミュニティ

  offline_storage:
    choice: Hive (Flutter) + IndexedDB (Web)
    rationale:
      - 既存実装の活用
      - 高速なローカルアクセス
      - 構造化データサポート

  presence_system:
    choice: Redis + Socket.IO Adapter
    rationale:
      - リアルタイムプレゼンス
      - 分散環境対応
      - 低レイテンシ
```

### 2.3 高レベルアーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌───────────────────┐   │
│  │   Flutter Mobile    │  │    Flutter Web      │  │   Admin Dashboard │   │
│  │  (iOS / Android)    │  │                     │  │                   │   │
│  │                     │  │                     │  │                   │   │
│  │  ┌───────────────┐  │  │  ┌───────────────┐  │  │  ┌─────────────┐ │   │
│  │  │ Socket.IO     │  │  │  │ Socket.IO     │  │  │  │ Socket.IO   │ │   │
│  │  │ Client        │  │  │  │ Client        │  │  │  │ Client      │ │   │
│  │  └───────┬───────┘  │  │  └───────┬───────┘  │  │  └──────┬──────┘ │   │
│  │          │          │  │          │          │  │         │        │   │
│  │  ┌───────▼───────┐  │  │  ┌───────▼───────┐  │  │                  │   │
│  │  │ Yjs CRDT      │  │  │  │ Yjs CRDT      │  │  │                  │   │
│  │  │ (Offline)     │  │  │  │ (IndexedDB)   │  │  │                  │   │
│  │  └───────┬───────┘  │  │  └───────┬───────┘  │  │                  │   │
│  │          │          │  │          │          │  │                  │   │
│  │  ┌───────▼───────┐  │  │  ┌───────▼───────┐  │  │                  │   │
│  │  │ Hive          │  │  │  │ IndexedDB     │  │  │                  │   │
│  │  │ Local Storage │  │  │  │               │  │  │                  │   │
│  │  └───────────────┘  │  │  └───────────────┘  │  │                  │   │
│  └─────────────────────┘  └─────────────────────┘  └───────────────────┘   │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │ WebSocket (wss://)
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          EDGE LAYER                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Global Load Balancer (L7)                         │   │
│  │                    - WebSocket Sticky Sessions                       │   │
│  │                    - Geographic Routing                              │   │
│  └──────────────────────────────┬──────────────────────────────────────┘   │
└─────────────────────────────────┼───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          REALTIME SERVICE LAYER                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    WebSocket Gateway Cluster                         │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────┐  │   │
│  │  │ WS Gateway  │  │ WS Gateway  │  │ WS Gateway  │  │    ...    │  │   │
│  │  │ Node 1      │  │ Node 2      │  │ Node 3      │  │           │  │   │
│  │  │             │  │             │  │             │  │           │  │   │
│  │  │ Socket.IO   │  │ Socket.IO   │  │ Socket.IO   │  │           │  │   │
│  │  │ Server      │  │ Server      │  │ Server      │  │           │  │   │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └───────────┘  │   │
│  │         │                │                │                         │   │
│  │         └────────────────┼────────────────┘                         │   │
│  │                          │ Redis Adapter                            │   │
│  │                          ▼                                          │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                    Redis Cluster                             │   │   │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │   │   │
│  │  │  │ Pub/Sub     │  │ Streams     │  │ Presence    │          │   │   │
│  │  │  │ (Real-time) │  │ (Durability)│  │ (Sessions)  │          │   │   │
│  │  │  └─────────────┘  └─────────────┘  └─────────────┘          │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐       │
│  │  CRDT Sync        │  │  Notification     │  │  Presence         │       │
│  │  Service          │  │  Service          │  │  Service          │       │
│  │  (Yjs Backend)    │  │                   │  │                   │       │
│  └─────────┬─────────┘  └─────────┬─────────┘  └─────────┬─────────┘       │
│            │                      │                      │                  │
└────────────┼──────────────────────┼──────────────────────┼──────────────────┘
             │                      │                      │
             ▼                      ▼                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DATA LAYER                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │  PostgreSQL     │  │  MongoDB        │  │  Object Storage │             │
│  │  (CRDT State)   │  │  (Notifications)│  │  (CRDT History) │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 第3章：WebSocketアーキテクチャ

### 3.1 接続管理

```yaml
connection_management:
  connection_lifecycle:
    establishment:
      steps:
        1. クライアントがHTTPアップグレードリクエスト送信
        2. 認証トークン（JWT）の検証
        3. WebSocket接続確立
        4. クライアントIDの割り当て
        5. 必要なルーム（Channel）への参加
        6. 初期状態の同期
      timeout: 10秒
      retry: 3回（指数バックオフ）

    heartbeat:
      interval: 25秒
      timeout: 60秒
      mechanism: Socket.IO ping/pong

    disconnection:
      graceful:
        - 明示的な切断メッセージ
        - ルームからの退出
        - プレゼンス更新
      ungraceful:
        - タイムアウト検出
        - 遅延プレゼンス更新（30秒後）
        - 再接続待機

    reconnection:
      strategy: 指数バックオフ + ジッター
      initial_delay: 1秒
      max_delay: 30秒
      max_attempts: 無制限
      state_recovery:
        - 最後のメッセージIDから再開
        - 差分同期
        - 全量同期（必要時）

  authentication:
    method: JWT in handshake query/header
    validation:
      - 署名検証
      - 有効期限確認
      - スコープ確認
    refresh:
      - トークン期限切れ前に再認証
      - 接続維持しながらトークン更新
    example:
      client: |
        const socket = io('wss://realtime.triptrip.com', {
          auth: {
            token: accessToken,
          },
          query: {
            clientVersion: '1.2.0',
            deviceId: deviceId,
          },
          transports: ['websocket'],
          reconnection: true,
          reconnectionAttempts: Infinity,
          reconnectionDelay: 1000,
          reconnectionDelayMax: 30000,
        });

  session_management:
    session_data:
      - user_id
      - device_id
      - client_version
      - connected_at
      - last_activity
      - subscribed_rooms
      - connection_metadata
    storage: Redis Hash
    ttl: 接続中は無期限、切断後30分
```

### 3.2 メッセージプロトコル設計

```yaml
message_protocol:
  format: JSON (Socket.IO Events)

  base_structure:
    event_name: イベント識別子
    payload:
      type: メッセージタイプ
      id: メッセージID（UUID）
      timestamp: ISO 8601タイムスタンプ
      data: イベント固有データ
      metadata:
        client_id: クライアント識別子
        correlation_id: 相関ID

  event_categories:
    system_events:
      - connect: 接続確立
      - disconnect: 切断
      - error: エラー発生
      - reconnect: 再接続
      - heartbeat: 死活確認

    room_events:
      - room:join: ルーム参加
      - room:leave: ルーム退出
      - room:message: ルームメッセージ

    sync_events:
      - sync:request: 同期リクエスト
      - sync:response: 同期レスポンス
      - sync:update: 差分更新
      - sync:conflict: コンフリクト通知

    collaboration_events:
      - collab:cursor: カーソル位置
      - collab:selection: 選択範囲
      - collab:operation: 操作
      - collab:presence: プレゼンス

    notification_events:
      - notification:new: 新着通知
      - notification:read: 既読
      - notification:dismiss: 削除

  message_examples:
    room_join:
      event: "room:join"
      payload:
        type: "room:join"
        id: "msg_abc123"
        timestamp: "2026-01-19T10:00:00Z"
        data:
          room_id: "trip_xyz789"
          room_type: "trip_collaboration"
        metadata:
          client_id: "client_def456"

    sync_update:
      event: "sync:update"
      payload:
        type: "sync:update"
        id: "msg_ghi012"
        timestamp: "2026-01-19T10:00:05Z"
        data:
          document_id: "trip_xyz789"
          operations:
            - type: "update"
              path: "/itinerary/0/title"
              value: "東京観光"
              origin: "client_def456"
          vector_clock:
            client_def456: 5
            client_jkl345: 3
        metadata:
          client_id: "client_def456"
          correlation_id: "corr_mno678"

    cursor_update:
      event: "collab:cursor"
      payload:
        type: "collab:cursor"
        id: "msg_pqr901"
        timestamp: "2026-01-19T10:00:07Z"
        data:
          document_id: "trip_xyz789"
          position:
            section: "itinerary"
            index: 0
            field: "title"
            offset: 5
          user:
            id: "user_stu234"
            name: "田中太郎"
            color: "#FF5733"
        metadata:
          client_id: "client_def456"

  acknowledgment:
    required_for:
      - 状態変更操作
      - 重要な通知
      - 同期メッセージ
    format:
      event: "ack"
      payload:
        original_id: 元メッセージID
        status: "success" | "error"
        error: エラー詳細（失敗時）
    timeout: 5秒
    retry: 3回
```

### 3.3 負荷分散・スケーリング

```yaml
load_balancing:
  strategy: Sticky Sessions (IP Hash or Cookie)

  load_balancer:
    type: Layer 7 (Application)
    product: Google Cloud Load Balancer / NGINX
    configuration:
      sticky_sessions:
        enabled: true
        cookie_name: "TRIPTRIP_WS_SESSION"
        ttl: 86400 (24時間)
      health_check:
        path: /health
        interval: 10s
        timeout: 5s
        unhealthy_threshold: 3
      websocket:
        timeout: 3600s (1時間)
        idle_timeout: 300s (5分)

  horizontal_scaling:
    mechanism: Kubernetes HPA
    metrics:
      - type: custom
        metric: websocket_connections_per_node
        target: 10000
      - type: cpu
        target: 70%
      - type: memory
        target: 80%
    min_replicas: 3
    max_replicas: 100
    scale_up_stabilization: 60s
    scale_down_stabilization: 300s

  server_coordination:
    adapter: Socket.IO Redis Adapter
    configuration:
      pubSubClient: Redis Cluster
      subClient: Redis Cluster
      requestsTimeout: 5000
    room_state:
      storage: Redis Hash
      key_pattern: "room:{room_id}:state"
      ttl: 永続（明示的削除まで）

  connection_draining:
    duration: 30秒
    process:
      1. ノードをUnhealthyにマーク
      2. 新規接続の受付停止
      3. 既存接続に再接続指示
      4. 30秒後に強制切断
      5. ノード終了
```

### 3.4 再接続・リカバリー戦略

```yaml
reconnection_strategy:
  client_side:
    automatic_reconnection:
      enabled: true
      max_attempts: Infinity
      delay_calculation: |
        const delay = Math.min(
          initialDelay * Math.pow(2, attempts),
          maxDelay
        );
        const jitter = delay * 0.3 * Math.random();
        return delay + jitter;
      initial_delay: 1000ms
      max_delay: 30000ms

    state_recovery:
      steps:
        1. 最後の同期ポイントを取得
        2. ローカル変更をキュー
        3. サーバーと差分同期
        4. ローカル変更を適用
        5. コンフリクト解決

    offline_detection:
      methods:
        - navigator.onLine API
        - WebSocket close event
        - Heartbeat timeout
      behavior:
        - オフラインモードに移行
        - ローカル操作の継続
        - 変更のキューイング

  server_side:
    session_recovery:
      grace_period: 30秒
      session_data_ttl: 5分
      steps:
        1. クライアントIDで既存セッション検索
        2. セッションデータの復元
        3. ルームへの再参加
        4. 未配信メッセージの再送

    message_redelivery:
      storage: Redis Streams
      retention: 24時間
      max_pending: 1000メッセージ/ユーザー
      delivery:
        - 最後のACKから再送
        - 順序保証
        - 重複排除（idempotency key）

  conflict_resolution:
    strategy: Last-Writer-Wins with Vector Clock
    implementation:
      - 各クライアントにベクタークロック
      - 操作にタイムスタンプ付与
      - 同時操作は CRDT で自動マージ
      - 解決不能時はユーザーに選択を提示
```

---

## 第4章：協調旅程編集

### 4.1 同時編集アーキテクチャ

```yaml
collaborative_editing:
  architecture:
    model: Real-time Collaboration
    approach: CRDT (Conflict-free Replicated Data Types)
    library: Yjs

  data_model:
    trip_document:
      structure:
        id: string (UUID)
        title: Y.Text
        description: Y.Text
        owner_id: string
        collaborators: Y.Array<Collaborator>
        itinerary: Y.Array<ItineraryItem>
        budget: Y.Map<Currency, number>
        notes: Y.Text
        attachments: Y.Array<Attachment>
        metadata:
          created_at: number
          updated_at: number
          version: number

      itinerary_item:
        id: string
        day: number
        time_slot: string
        type: "hotel" | "attraction" | "restaurant" | "transport" | "free"
        resource_id: string
        title: Y.Text
        notes: Y.Text
        booking_status: string
        cost: number
        currency: string

  collaboration_features:
    real_time_cursors:
      update_frequency: 50ms (throttled)
      data:
        user_id: string
        user_name: string
        color: string
        position:
          section: string
          index: number
          field: string
          offset: number

    awareness:
      user_info:
        id: string
        name: string
        color: string
        avatar_url: string
      states:
        - online
        - editing
        - viewing
        - idle (5分後)
        - offline

    comments:
      structure:
        id: string
        target:
          type: "section" | "item" | "field"
          path: string
        author_id: string
        content: Y.Text
        resolved: boolean
        created_at: number
        replies: Y.Array<Reply>

    version_history:
      storage: S3 (snapshots) + PostgreSQL (metadata)
      snapshot_frequency: 5分または10操作
      retention: 90日
      features:
        - ポイントインタイム復元
        - 差分表示
        - 特定バージョンへのロールバック
```

### 4.2 CRDT（Conflict-free Replicated Data Types）

```yaml
crdt_implementation:
  library: Yjs
  version: 13.x

  data_types:
    Y.Text:
      use_case: テキストコンテンツ（タイトル、説明、ノート）
      operations:
        - insert(index, content)
        - delete(index, length)
        - format(index, length, attributes)
      conflict_resolution: 位置ベースのマージ

    Y.Array:
      use_case: 順序付きリスト（旅程、添付ファイル）
      operations:
        - insert(index, content)
        - delete(index, length)
        - push(content)
        - unshift(content)
      conflict_resolution: インデックスベースのマージ

    Y.Map:
      use_case: キーバリューデータ（設定、予算）
      operations:
        - set(key, value)
        - delete(key)
      conflict_resolution: Last-Writer-Wins

  synchronization:
    protocol: y-websocket
    process:
      1. クライアント接続時にドキュメント状態をリクエスト
      2. サーバーが現在の状態ベクトルを送信
      3. クライアントが差分を計算
      4. 双方向の差分交換
      5. 継続的な操作の同期

    encoding:
      format: Binary (Yjs encoding)
      compression: 差分圧縮
      batching: 複数操作をバッチ処理

  server_implementation:
    persistence:
      adapter: LevelDB / PostgreSQL
      storage_format: Yjs binary state
      update_strategy:
        - インメモリで操作を適用
        - 定期的にディスクにフラッシュ（5秒間隔）
        - シャットダウン時に完全保存

    scaling:
      single_document:
        - 1つのサーバーノードで処理
        - ドキュメントIDでルーティング
      cross_node:
        - Redis経由で操作を伝播
        - Eventually consistent

  example_implementation:
    server: |
      import * as Y from 'yjs';
      import { WebsocketProvider } from 'y-websocket';
      import { LeveldbPersistence } from 'y-leveldb';

      const persistence = new LeveldbPersistence('./yjs-data');

      const handleConnection = async (conn, docName) => {
        // ドキュメントの取得または作成
        const doc = await persistence.getYDoc(docName);

        // 更新のリスニング
        doc.on('update', (update, origin) => {
          // 他のクライアントに配信
          broadcastUpdate(docName, update, origin);

          // 永続化
          persistence.storeUpdate(docName, update);
        });

        return doc;
      };

    client: |
      import * as Y from 'yjs';
      import { WebsocketProvider } from 'y-websocket';
      import { IndexeddbPersistence } from 'y-indexeddb';

      // ドキュメント作成
      const doc = new Y.Doc();

      // ローカル永続化
      const indexeddbProvider = new IndexeddbPersistence('trip-' + tripId, doc);

      // WebSocket同期
      const wsProvider = new WebsocketProvider(
        'wss://realtime.triptrip.com',
        'trip-' + tripId,
        doc,
        { params: { auth: accessToken } }
      );

      // 旅程データへのアクセス
      const itinerary = doc.getArray('itinerary');

      // 変更の監視
      itinerary.observe(event => {
        event.changes.added.forEach(item => {
          console.log('Added:', item.content);
        });
      });

      // アイテムの追加
      itinerary.push([{
        id: generateId(),
        day: 1,
        type: 'attraction',
        title: '東京スカイツリー',
      }]);
```

### 4.3 Operational Transformation（代替アプローチ）

```yaml
operational_transformation:
  description: OTは代替アプローチとして文書化（CRDTを推奨）

  comparison_with_crdt:
    crdt_advantages:
      - サーバー不要でコンフリクト解決
      - オフライン編集に強い
      - 実装がシンプル
      - 数学的に証明された一貫性

    ot_advantages:
      - 細かい制御が可能
      - 履歴管理が容易
      - 意図の保存に優れる

    chosen: CRDT (Yjs)
    rationale:
      - オフラインファースト要件
      - モバイルアプリとの相性
      - 実装・運用の簡素化

  ot_reference:
    description: 将来の参考として OT の概要を記載
    concept:
      - 操作（Operation）の変換
      - 同時操作の順序付け
      - サーバーでの調停

    libraries:
      - ShareJS
      - ot.js
      - operational-transformation

    use_case_consideration:
      - リッチテキストエディタ
      - 高精度の意図保存が必要な場合
```

### 4.4 コンフリクト解決戦略

```yaml
conflict_resolution:
  automatic_resolution:
    crdt_based:
      description: CRDTの特性による自動解決
      mechanisms:
        text:
          - 同じ位置への挿入: 両方を保持（順序はクライアントIDで決定）
          - 重複削除: 冪等性により自動処理
          - フォーマット競合: Last-Writer-Wins

        array:
          - 同じ位置への挿入: 両方を保持
          - 同じ要素の移動: 最後の移動を採用
          - 削除と更新: 削除を優先

        map:
          - 同じキーの更新: Last-Writer-Wins
          - 削除と更新: 更新を優先

  manual_resolution:
    trigger_conditions:
      - 同一アイテムの大幅な変更
      - 予約ステータスの競合
      - ビジネスルール違反

    resolution_flow:
      1. コンフリクト検出
      2. 影響を受けるユーザーに通知
      3. オプション提示
         - 自分の変更を採用
         - 相手の変更を採用
         - マージを試みる
      4. 解決の適用
      5. 全員に同期

    ui_presentation:
      notification:
        title: "編集の競合が発生しました"
        message: "{user_name}さんの変更と競合しています"
        options:
          - label: "私の変更を使用"
            action: use_mine
          - label: "{user_name}さんの変更を使用"
            action: use_theirs
          - label: "両方を確認"
            action: show_diff

  booking_specific:
    description: 予約関連の特殊な競合解決
    scenarios:
      simultaneous_booking:
        description: 同じリソースを同時に予約
        resolution:
          - 在庫確保は先着順
          - 失敗側にリアルタイム通知
          - 代替提案の自動表示

      status_conflict:
        description: 予約ステータスの競合
        resolution:
          - 進行度が高いステータスを優先
          - pending < confirmed < cancelled
          - 管理者による上書き可能
```

---

## 第5章：リアルタイム通知システム

### 5.1 通知チャネル

```yaml
notification_channels:
  in_app:
    description: アプリ内リアルタイム通知
    delivery: WebSocket
    priority: 即時
    features:
      - バッジカウント更新
      - トースト表示
      - 通知センター
    storage: Redis (短期) + MongoDB (永続)

  push:
    description: モバイルプッシュ通知
    providers:
      ios: Apple Push Notification Service (APNs)
      android: Firebase Cloud Messaging (FCM)
      web: Web Push (FCM)
    priority: 高
    features:
      - バックグラウンド配信
      - リッチ通知（画像、アクション）
      - サイレントプッシュ
    storage: デバイストークン in PostgreSQL

  email:
    description: メール通知
    provider: SendGrid / AWS SES
    priority: 中
    features:
      - HTML テンプレート
      - 開封・クリックトラッキング
      - 配信スケジューリング
    storage: MongoDB (ログ)

  sms:
    description: SMS通知
    provider: Twilio
    priority: 高（緊急時）
    features:
      - 国際配信
      - 配信確認
    use_cases:
      - 予約確認コード
      - 緊急連絡
    storage: MongoDB (ログ)

  channel_selection:
    rules:
      - ユーザー設定を最優先
      - 緊急度に応じたチャネル選択
      - 時間帯に応じた配信制御
      - フォールバック順序の定義
```

### 5.2 通知ルーティングエンジン

```yaml
notification_routing:
  architecture:
    input: 通知イベント（Kafka）
    processing: ルーティングエンジン
    output: チャネル別配信キュー

  routing_rules:
    user_preferences:
      source: ユーザー設定
      rules:
        - チャネル別の有効/無効
        - 通知タイプ別の設定
        - 時間帯設定（Do Not Disturb）

    notification_type:
      booking_confirmed:
        channels: [in_app, push, email]
        priority: high
        template: booking_confirmation

      price_alert:
        channels: [in_app, push]
        priority: medium
        template: price_alert

      trip_update:
        channels: [in_app]
        priority: low
        template: trip_update

      payment_failed:
        channels: [in_app, push, email, sms]
        priority: critical
        template: payment_failure

    context_rules:
      time_based:
        - 夜間（22:00-07:00）はPush抑制
        - 緊急通知は時間帯無視
      location_based:
        - 旅行中はトリップ関連を優先
      engagement_based:
        - 非アクティブユーザーはEmail優先

  routing_engine:
    implementation: |
      class NotificationRouter {
        async route(event: NotificationEvent): Promise<RoutingResult[]> {
          const user = await this.userService.get(event.userId);
          const preferences = await this.preferenceService.get(event.userId);
          const notificationType = this.notificationTypes.get(event.type);

          const channels = [];

          // チャネル選択
          for (const channel of notificationType.channels) {
            if (this.shouldDeliver(channel, preferences, event)) {
              channels.push({
                channel,
                priority: this.calculatePriority(channel, event),
                template: notificationType.templates[channel],
                data: this.enrichData(event, user),
              });
            }
          }

          return channels;
        }

        shouldDeliver(channel, preferences, event): boolean {
          // 緊急通知は常に配信
          if (event.priority === 'critical') return true;

          // ユーザー設定をチェック
          if (!preferences[channel].enabled) return false;

          // Do Not Disturb チェック
          if (this.isDoNotDisturbTime(preferences)) return false;

          // 通知タイプ別設定
          if (!preferences[channel].types.includes(event.type)) return false;

          return true;
        }
      }
```

### 5.3 配信保証と再送メカニズム

```yaml
delivery_guarantee:
  guarantee_level: At-least-once

  delivery_tracking:
    states:
      - pending: 配信キューに追加
      - sent: プロバイダーに送信
      - delivered: 配信確認
      - read: 既読
      - failed: 配信失敗
      - expired: 有効期限切れ

    storage:
      pending: Redis (TTL 24時間)
      history: MongoDB (90日保持)

  retry_mechanism:
    strategy: 指数バックオフ
    max_attempts: 5
    intervals:
      - 1分
      - 5分
      - 30分
      - 2時間
      - 24時間

    per_channel:
      push:
        max_attempts: 3
        timeout: 30秒
        fallback: in_app
      email:
        max_attempts: 5
        timeout: 60秒
      sms:
        max_attempts: 3
        timeout: 30秒

  failure_handling:
    transient_errors:
      - ネットワークタイムアウト
      - プロバイダー一時エラー
      action: リトライ

    permanent_errors:
      - 無効なトークン
      - ユーザー退会
      - オプトアウト
      action: 停止、クリーンアップ

    provider_specific:
      apns_invalid_token:
        action: トークン削除、再登録要求
      fcm_not_registered:
        action: トークン削除
      email_bounce:
        action: メールアドレス無効化

  deduplication:
    mechanism: Idempotency Key
    storage: Redis SET
    ttl: 24時間
    check: 配信前にキー存在確認

  monitoring:
    metrics:
      - delivery_rate: 配信成功率
      - latency_p99: 配信レイテンシ
      - retry_rate: リトライ率
      - failure_rate: 最終失敗率
    alerting:
      delivery_rate_threshold: 95%
      latency_threshold: 5秒
```

---

## 第6章：プレゼンス・アクティビティトラッキング

### 6.1 プレゼンスシステム設計

```yaml
presence_system:
  states:
    online:
      description: アクティブに接続中
      indicator: 緑色のドット
      detection: WebSocket接続中

    away:
      description: 接続中だが非アクティブ
      indicator: 黄色のドット
      detection: 5分間操作なし
      transition: 最後のアクティビティから5分後

    offline:
      description: 切断状態
      indicator: グレーのドット
      detection: WebSocket切断
      grace_period: 30秒（再接続待機）

    do_not_disturb:
      description: 通知オフ
      indicator: 赤色のドット
      detection: ユーザー設定

    invisible:
      description: オンラインだが非表示
      indicator: オフライン表示
      detection: ユーザー設定

  implementation:
    storage: Redis Hash + Sorted Set

    data_structure:
      hash_key: "presence:{user_id}"
      fields:
        status: online | away | offline | dnd | invisible
        last_seen: timestamp
        device_id: string
        client_ip: string
        metadata: JSON

      sorted_set_key: "presence:online"
      score: last_activity_timestamp
      member: user_id

    operations:
      set_online: |
        MULTI
        HSET presence:{user_id} status online last_seen {timestamp} device_id {device}
        ZADD presence:online {timestamp} {user_id}
        EXEC

      set_offline: |
        MULTI
        HSET presence:{user_id} status offline last_seen {timestamp}
        ZREM presence:online {user_id}
        EXEC

      get_online_users: |
        ZRANGEBYSCORE presence:online {now - 300} +inf

      check_away: |
        -- 5分以上アクティビティがないユーザーをawayに
        ZRANGEBYSCORE presence:online -inf {now - 300}

  broadcasting:
    scope: 関連ユーザーのみ
    rules:
      - 同じトリップの共同編集者
      - フレンドリスト
      - 同じチャットルーム

    throttling:
      max_rate: 1回/秒/ユーザー
      batching: 100ms間隔でバッチ

    privacy:
      - invisibleユーザーは配信対象外
      - ブロックユーザーは除外
      - プライバシー設定を尊重
```

### 6.2 アクティビティフィード

```yaml
activity_feed:
  description: ユーザーアクションのリアルタイムフィード

  activity_types:
    trip_activities:
      - trip.created: 旅程作成
      - trip.updated: 旅程更新
      - trip.item_added: アイテム追加
      - trip.item_removed: アイテム削除
      - trip.collaborator_joined: 共同編集者参加
      - trip.comment_added: コメント追加

    booking_activities:
      - booking.created: 予約作成
      - booking.confirmed: 予約確定
      - booking.cancelled: 予約キャンセル

    social_activities:
      - user.followed: フォロー
      - review.posted: レビュー投稿
      - rating.given: 評価

  data_model:
    activity:
      id: string (UUID)
      type: string
      actor:
        id: string
        name: string
        avatar_url: string
      target:
        type: string
        id: string
        name: string
      context:
        trip_id: string (optional)
        additional_data: object
      timestamp: datetime
      visibility: public | private | collaborators

  storage:
    primary: MongoDB
    cache: Redis (最新100件)
    index:
      - actor_id + timestamp
      - target_id + timestamp
      - trip_id + timestamp

  feed_generation:
    strategies:
      fan_out_on_write:
        description: 書き込み時に全フォロワーに配信
        use_case: 少数のフォロワー
        latency: 読み取り時は高速

      fan_out_on_read:
        description: 読み取り時にフィードを構築
        use_case: 多数のフォロワー
        latency: 書き込み時は高速

      hybrid:
        description: アクティブユーザーには書き込み時、非アクティブには読み取り時
        chosen: true

  real_time_delivery:
    mechanism: WebSocket + Redis Pub/Sub
    process:
      1. アクティビティ発生
      2. MongoDB に保存
      3. Redis にキャッシュ
      4. Redis Pub/Sub で配信
      5. WebSocket で接続中クライアントに送信

  pagination:
    type: cursor-based
    cursor: timestamp + id
    limit: 20-50
```

---

## 第7章：オフラインファースト同期

### 7.1 オフラインデータストレージ

```yaml
offline_storage:
  flutter_mobile:
    primary: Hive
    configuration:
      boxes:
        - name: trips
          schema: TripModel
          encryption: true
        - name: bookings
          schema: BookingModel
          encryption: true
        - name: user_data
          schema: UserModel
          encryption: true
        - name: cache
          schema: CacheEntry
          ttl: 24時間
        - name: pending_operations
          schema: PendingOperation
          priority: true

    encryption:
      algorithm: AES-256
      key_storage: Secure Storage (Keychain/Keystore)

    size_management:
      max_size: 100MB
      cleanup_policy:
        - 古いキャッシュの削除
        - 不要な画像の削除
        - 同期済み操作の削除

  flutter_web:
    primary: IndexedDB (via Hive)
    fallback: LocalStorage (小さいデータ)
    configuration:
      database_name: triptrip_offline
      stores:
        - trips
        - bookings
        - user_data
        - cache
        - pending_operations

  data_prioritization:
    critical:
      - ユーザー認証情報
      - アクティブな予約
      - 現在の旅程
      sync_first: true

    important:
      - 最近の予約履歴
      - お気に入り
      - 検索履歴

    optional:
      - 推奨コンテンツ
      - 分析データ
      - 古い履歴

  crdt_integration:
    yjs_persistence:
      mobile: y-indexeddb (Hive wrapper)
      web: y-indexeddb
    sync_with_local:
      - CRDT状態をHive/IndexedDBに保存
      - アプリ起動時にCRDT状態を復元
      - バックグラウンドで差分同期
```

### 7.2 同期プロトコル設計

```yaml
sync_protocol:
  overview:
    type: Differential Synchronization with CRDT
    direction: Bidirectional
    trigger: 接続復帰時 + 定期的

  sync_states:
    synced:
      description: サーバーと完全同期
      indicator: チェックマーク
    syncing:
      description: 同期中
      indicator: 回転アイコン
    pending:
      description: 未同期の変更あり
      indicator: 雲アイコン + 数字
    conflict:
      description: コンフリクトあり
      indicator: 警告アイコン
    error:
      description: 同期エラー
      indicator: エラーアイコン

  sync_process:
    initial_sync:
      steps:
        1. クライアントが最後の同期ポイント（vector clock）を送信
        2. サーバーが差分を計算
        3. サーバーが差分データを送信
        4. クライアントが差分を適用
        5. クライアントがローカル変更を送信
        6. サーバーがマージして確認
        7. 同期完了

    incremental_sync:
      trigger:
        - ネットワーク復帰
        - アプリ起動
        - 定期的（5分間隔）
        - ユーザーアクション（手動同期）
      process:
        1. ローカル変更のキュー確認
        2. 変更をサーバーに送信
        3. サーバーからの変更を受信
        4. ローカルに適用

  message_format:
    sync_request:
      type: "sync:request"
      payload:
        document_id: string
        vector_clock: Map<client_id, number>
        pending_updates: Array<Update>

    sync_response:
      type: "sync:response"
      payload:
        document_id: string
        updates: Array<Update>
        new_vector_clock: Map<client_id, number>
        conflicts: Array<Conflict>

  bandwidth_optimization:
    compression: gzip
    delta_encoding: true
    batching:
      max_batch_size: 100 operations
      max_batch_time: 1秒
    prioritization:
      - ユーザーがアクティブに編集中のドキュメント優先
      - 予約関連データ優先
```

### 7.3 コンフリクト検出・解決

```yaml
conflict_detection:
  mechanisms:
    vector_clock:
      description: 論理時計による因果関係の追跡
      implementation: |
        class VectorClock {
          clocks: Map<string, number> = new Map();

          increment(clientId: string) {
            const current = this.clocks.get(clientId) || 0;
            this.clocks.set(clientId, current + 1);
          }

          merge(other: VectorClock) {
            for (const [clientId, time] of other.clocks) {
              const current = this.clocks.get(clientId) || 0;
              this.clocks.set(clientId, Math.max(current, time));
            }
          }

          compare(other: VectorClock): 'before' | 'after' | 'concurrent' {
            let dominated = false;
            let dominates = false;

            for (const clientId of new Set([...this.clocks.keys(), ...other.clocks.keys()])) {
              const a = this.clocks.get(clientId) || 0;
              const b = other.clocks.get(clientId) || 0;

              if (a < b) dominated = true;
              if (a > b) dominates = true;
            }

            if (dominated && !dominates) return 'before';
            if (dominates && !dominated) return 'after';
            if (dominated && dominates) return 'concurrent';
            return 'after'; // equal
          }
        }

    hash_comparison:
      description: コンテンツハッシュによる変更検出
      use_case: 大きなドキュメントの高速比較

  conflict_types:
    update_update:
      description: 同じフィールドの同時更新
      resolution: CRDT自動マージ or ユーザー選択

    delete_update:
      description: 削除と更新の競合
      resolution: 削除を優先（設定可能）

    structural:
      description: 構造の変更（親の削除など）
      resolution: 構造変更を優先、子は孤立ノードとして保持

conflict_resolution:
  strategies:
    automatic:
      crdt_merge:
        description: CRDTの特性による自動マージ
        coverage: 90%以上のケース

      last_writer_wins:
        description: 最後の書き込みを採用
        use_case: 単純な値フィールド
        tie_breaker: client_id の辞書順

      merge_text:
        description: テキストの3-way マージ
        use_case: 短いテキストフィールド

    manual:
      user_choice:
        description: ユーザーに選択を提示
        use_case: 重要な変更、明確な競合
        ui:
          - 両方の変更を表示
          - 差分のハイライト
          - 選択ボタン

      admin_resolution:
        description: 管理者による解決
        use_case: ビジネスクリティカルな競合

  resolution_history:
    storage: PostgreSQL
    retention: 90日
    fields:
      - conflict_id
      - document_id
      - conflict_type
      - versions_involved
      - resolution_method
      - resolved_by
      - resolved_at
```

### 7.4 バックグラウンド同期

```yaml
background_sync:
  mobile_platforms:
    ios:
      mechanism: Background App Refresh
      constraints:
        - システムによるスケジューリング
        - 最大30秒の実行時間
        - ユーザー設定による制限
      implementation:
        - WorkManager (Flutter)
        - Significant Location Change (位置ベース)
        - Silent Push Notification

    android:
      mechanism: WorkManager
      constraints:
        - バッテリー最適化の考慮
        - Dozeモードの影響
      implementation:
        - Periodic Work (15分最小間隔)
        - One-time Work (即時)
        - Constraint-based (ネットワーク、充電状態)

  sync_triggers:
    periodic:
      interval: 15分
      conditions:
        - ネットワーク接続あり
        - バッテリー20%以上
        - 充電中は積極的に

    event_based:
      - アプリ起動
      - ネットワーク復帰
      - 予約時刻の接近
      - プッシュ通知受信

    user_initiated:
      - 手動更新ボタン
      - プルトゥリフレッシュ

  sync_strategy:
    priority_based:
      1. クリティカルデータ（予約、決済）
      2. ユーザーデータ（プロフィール、設定）
      3. コンテンツ（旅程、履歴）
      4. キャッシュ（検索結果、推奨）

    bandwidth_aware:
      wifi:
        - フルシンク
        - メディアファイル同期
      cellular:
        - 差分のみ
        - メディアはWi-Fi待ち
      metered:
        - クリティカルのみ

  implementation:
    flutter: |
      class BackgroundSyncService {
        static const String SYNC_TASK = 'triptrip_sync';

        static Future<void> initialize() async {
          await Workmanager().initialize(callbackDispatcher);

          await Workmanager().registerPeriodicTask(
            SYNC_TASK,
            SYNC_TASK,
            frequency: Duration(minutes: 15),
            constraints: Constraints(
              networkType: NetworkType.connected,
              requiresBatteryNotLow: true,
            ),
          );
        }

        static Future<void> performSync() async {
          final syncService = SyncService();

          // 優先度順に同期
          await syncService.syncCriticalData();
          await syncService.syncUserData();
          await syncService.syncContent();

          // 成功を記録
          await LocalStorage.setLastSyncTime(DateTime.now());
        }
      }

      @pragma('vm:entry-point')
      void callbackDispatcher() {
        Workmanager().executeTask((task, inputData) async {
          if (task == BackgroundSyncService.SYNC_TASK) {
            await BackgroundSyncService.performSync();
          }
          return true;
        });
      }
```

---

## 第8章：パフォーマンス最適化

### 8.1 メッセージ配信の最適化

```yaml
message_optimization:
  batching:
    description: 複数メッセージのバッチ処理
    configuration:
      max_batch_size: 100
      max_batch_delay: 100ms
      flush_on_priority: true

    implementation: |
      class MessageBatcher {
        private queue: Message[] = [];
        private timer: NodeJS.Timeout | null = null;

        add(message: Message) {
          this.queue.push(message);

          if (message.priority === 'high' || this.queue.length >= MAX_BATCH_SIZE) {
            this.flush();
          } else if (!this.timer) {
            this.timer = setTimeout(() => this.flush(), MAX_BATCH_DELAY);
          }
        }

        private flush() {
          if (this.timer) {
            clearTimeout(this.timer);
            this.timer = null;
          }

          if (this.queue.length > 0) {
            const batch = this.queue.splice(0, MAX_BATCH_SIZE);
            this.send(batch);
          }
        }
      }

  compression:
    algorithm: zlib (gzip)
    threshold: 1KB
    level: 6 (balanced)

  filtering:
    client_side:
      - 関心のあるイベントのみ購読
      - 重複排除
      - レート制限

    server_side:
      - ルームベースのフィルタリング
      - ユーザー権限に基づくフィルタリング
      - 地理的フィルタリング

  throttling:
    cursor_updates:
      rate: 20Hz (50ms間隔)
      strategy: 最新のみ送信

    presence_updates:
      rate: 0.2Hz (5秒間隔)
      strategy: 変更時のみ送信

    typing_indicator:
      rate: 2Hz (500ms間隔)
      timeout: 3秒
```

### 8.2 接続管理の最適化

```yaml
connection_optimization:
  connection_pooling:
    client_side:
      - 単一のWebSocket接続を共有
      - 複数のルームを多重化
      - 接続状態の一元管理

    server_side:
      - Redis接続プール
      - DB接続プール
      - 外部API接続プール

  keep_alive:
    ping_interval: 25秒
    pong_timeout: 5秒
    reconnect_on_timeout: true

  resource_cleanup:
    idle_connection_timeout: 5分
    orphan_room_cleanup: 1時間
    stale_session_cleanup: 24時間

  memory_management:
    per_connection_limit: 1MB
    per_room_limit: 10MB
    global_limit: 監視 + アラート

  load_shedding:
    triggers:
      - CPU > 80%
      - Memory > 85%
      - Connection count > threshold
    actions:
      - 新規接続の拒否
      - 低優先度接続の切断
      - 機能制限モード

  geographic_optimization:
    edge_servers:
      - 東京（asia-northeast1）
      - シンガポール（asia-southeast1）
      - 大阪（asia-northeast2）
    routing:
      - 最寄りのエッジに接続
      - フェイルオーバー対応
```

### 8.3 データ同期の最適化

```yaml
sync_optimization:
  delta_sync:
    description: 差分のみを同期
    implementation:
      - ベクタークロックによる差分計算
      - バイナリ差分エンコーディング
      - 圧縮転送

  selective_sync:
    description: 必要なデータのみ同期
    strategies:
      - アクティブドキュメント優先
      - 可視範囲のデータ優先
      - オンデマンドロード

  lazy_loading:
    description: 遅延ロード
    use_cases:
      - 長い履歴
      - 大きな添付ファイル
      - 過去の旅程

  prefetching:
    description: 先読み
    triggers:
      - ユーザーのナビゲーションパターン
      - 予測されるアクション
      - オフライン準備

  caching:
    levels:
      memory:
        size: 10MB
        ttl: 5分
      disk:
        size: 100MB
        ttl: 24時間

    invalidation:
      - サーバーからの通知
      - TTL期限切れ
      - 明示的なクリア
```

---

## 第9章：実装ロードマップ

### 9.1 フェーズ別計画

```yaml
implementation_roadmap:
  phase_1:
    name: 基盤構築
    duration: 3ヶ月
    objectives:
      - WebSocketインフラの構築
      - 基本的なリアルタイム通知
      - 接続管理の実装
    deliverables:
      - WebSocket Gateway（Socket.IO）
      - Redis Pub/Sub連携
      - 基本的な認証・接続管理
      - In-App通知
    success_criteria:
      - 10,000同時接続
      - P99レイテンシ < 100ms
      - 99.9%可用性

  phase_2:
    name: 通知システム強化
    duration: 3ヶ月
    objectives:
      - マルチチャネル通知
      - 配信保証の実装
      - プレゼンス機能
    deliverables:
      - Push通知（FCM/APNs）
      - Email/SMS通知
      - プレゼンスシステム
      - 配信追跡・リトライ
    success_criteria:
      - 通知配信率 99%
      - プレゼンス更新 < 5秒
      - 全チャネル稼働

  phase_3:
    name: 協調編集
    duration: 6ヶ月
    objectives:
      - CRDT同期基盤
      - 旅程の共同編集
      - オフライン編集
    deliverables:
      - Yjs統合
      - 協調カーソル
      - コンフリクト解決
      - オフラインストレージ
    success_criteria:
      - 同時編集 100ユーザー/ドキュメント
      - コンフリクト自動解決率 95%
      - オフライン機能稼働

  phase_4:
    name: スケーリングと最適化
    duration: 6ヶ月
    objectives:
      - 100万同時接続対応
      - グローバル展開
      - パフォーマンス最適化
    deliverables:
      - マルチリージョン展開
      - エッジサーバー
      - 高度なキャッシング
      - 監視・アラート強化
    success_criteria:
      - 100万同時接続
      - P99レイテンシ < 50ms
      - 99.99%可用性
```

---

## 第10章：文書間参照 & 統合ポイント

### 10.1 関連文書

```yaml
document_references:
  prerequisites:
    - doc_id: Doc-TV-001
      title: TripTrip技術ビジョンとアーキテクチャ原則
      relationship: 技術原則の定義

    - doc_id: Doc-SA-001
      title: TripTripシステムアーキテクチャ概要
      relationship: 全体アーキテクチャの定義

    - doc_id: Doc-SA-002
      title: マイクロサービスアーキテクチャ
      relationship: サービス分割とイベント駆動

    - doc_id: Doc-SA-003
      title: APIアーキテクチャ・設計
      relationship: REST/GraphQL APIとの連携

  related_documents:
    - doc_id: Doc-SA-005
      title: 検索・推奨アーキテクチャ
      relationship: リアルタイム検索・推奨

    - doc_id: Doc-INF-001
      title: インフラストラクチャ設計
      relationship: WebSocketサーバー、Redis、ロードバランサー

    - doc_id: Doc-SEC-001
      title: セキュリティアーキテクチャ
      relationship: WebSocket認証、暗号化
```

### 10.2 統合ポイント

```yaml
integration_points:
  websocket_gateway:
    component: Socket.IO Server Cluster
    integrations:
      - API Gateway（認証トークン検証）
      - Redis Cluster（Pub/Sub、プレゼンス）
      - Kafka（イベント連携）
    related_docs: [Doc-SA-001, Doc-INF-001]

  notification_service:
    component: Notification Service
    integrations:
      - FCM/APNs（Push通知）
      - SendGrid/Twilio（Email/SMS）
      - WebSocket Gateway（In-App）
    related_docs: [Doc-SA-002]

  crdt_sync:
    component: CRDT Sync Service (Yjs)
    integrations:
      - PostgreSQL（状態永続化）
      - S3（履歴スナップショット）
      - WebSocket Gateway（リアルタイム同期）
    related_docs: [Doc-SA-002]

  offline_storage:
    component: Hive / IndexedDB
    integrations:
      - Flutter状態管理（Riverpod）
      - Yjs（CRDT永続化）
      - Sync Service（バックグラウンド同期）
    related_docs: [EXISTING_APP_ANALYSIS.md]
```

---

## 文書情報

| 項目 | 内容 |
|------|------|
| Document ID | Doc-SA-004 |
| Title | リアルタイム・協調機能アーキテクチャ |
| Version | 1.0.0 |
| Status | Draft |
| Author | Technical Architecture Agent |
| Created | 2026-01-19 |
| Last Updated | 2026-01-19 |
| Total Lines | ~1,500 |
| Previous Document | Doc-SA-003: APIアーキテクチャ・設計 |
| Next Document | Doc-SA-005: 検索・推奨アーキテクチャ |

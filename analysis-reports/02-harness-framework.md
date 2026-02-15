# OpenClaw ハーネス・フレームワーク構造分析レポート

## 概要サマリー

OpenClawは、マルチチャネル対応のAIエージェント実行基盤（ハーネス）である。CLI経由でのワンショット実行から、ゲートウェイサーバーによる常駐型マルチチャネルエージェントまで、幅広い実行モードを提供する。

主要な設計原則:
- **プラグインベースの拡張性**: チャネル、ツール、プロバイダー、フック全てがプラグインとして登録可能
- **レーン型並行制御**: セッション単位・グローバル単位のコマンドキューによる安全な直列化
- **依存性注入（DI）パターン**: `createDefaultDeps()`を起点とした疎結合設計
- **遅延ロード**: サブコマンドやプラグインの遅延インポートによる起動高速化
- **フェイルオーバー対応**: 認証プロファイルローテーションとモデルフォールバックチェーン

---

## 1. CLI構造とコマンド体系

### 1.1 エントリーポイントとブートストラップ

```
openclaw.mjs
  -> src/entry.ts              # プロセス初期化、respawn制御
     -> src/cli/run-main.ts    # CLI起動メイン
        -> src/cli/program.ts  # Commander.jsプログラムビルド
```

**`src/entry.ts:1-171`**: CLIのエントリーポイント。以下の処理を行う:
1. プロセスタイトルを"openclaw"に設定
2. Node.jsの実験的機能警告を抑制するためのrespawn制御
3. CLIプロファイル（`--profile`）の解析・適用
4. `run-main.ts`の動的インポートとCLI実行

**`src/cli/run-main.ts:27-72`**: `runCli()`関数がCLI実行の起点:
1. `.env`ファイルのロード
2. ランタイム互換性チェック（Node 22+）
3. 高速ルーティング（`tryRouteCli`）による軽量コマンドの早期実行
4. コンソールキャプチャの有効化
5. Commanderプログラムのビルドとパース
6. **遅延サブコマンド登録**: `registerSubCliByName()`で必要なサブCLIのみ登録

### 1.2 コマンド登録アーキテクチャ

```
src/cli/program/build-program.ts:7-18
  -> command-registry.ts       # コア/内蔵コマンド登録
  -> register.subclis.ts       # 遅延ロードサブコマンド
  -> plugins/cli.ts            # プラグインCLIコマンド
```

**遅延ロードパターン** (`src/cli/program/register.subclis.ts:34-`):

サブコマンドは`SubCliEntry`配列で定義され、実行時に動的インポートされる:
```
gateway, daemon, logs, system, models, acp, ...
```

これにより、`openclaw status`実行時にgateway関連モジュールを読み込まないなど、起動時間を最適化している。

### 1.3 依存性注入パターン

**`src/cli/deps.ts:9-41`**: `CliDeps`型とファクトリ関数:

```typescript
type CliDeps = {
  sendMessageWhatsApp: typeof sendMessageWhatsApp;
  sendMessageTelegram: typeof sendMessageTelegram;
  sendMessageDiscord: typeof sendMessageDiscord;
  sendMessageSlack: typeof sendMessageSlack;
  sendMessageSignal: typeof sendMessageSignal;
  sendMessageIMessage: typeof sendMessageIMessage;
};
```

`createDefaultDeps()`が実装をバンドルし、`createOutboundSendDeps()`がアウトバウンド送信用にリマップする。テスト時にはモック差し替えが容易。

### 1.4 ランタイム環境抽象化

**`src/runtime.ts:4-24`**: `RuntimeEnv`型:

```typescript
type RuntimeEnv = {
  log: typeof console.log;
  error: typeof console.error;
  exit: (code: number) => never;
};
```

全コマンドが`RuntimeEnv`を引数に受け取り、テスト時にexit/logをモック可能にしている。`defaultRuntime`はターミナル状態復元付きの安全な実装。

---

## 2. ゲートウェイアーキテクチャ

### 2.1 起動フロー

```
openclaw gateway run
  -> src/cli/gateway-cli/run.ts:54      # runGatewayCommand()
     -> src/cli/gateway-cli/run-loop.ts  # runGatewayLoop()
        -> src/gateway/server.impl.ts    # startGatewayServer()
           -> server-startup.ts          # startGatewaySidecars()
           -> server-channels.ts         # createChannelManager()
           -> server-chat.ts             # createAgentEventHandler()
           -> server-ws-runtime.ts       # WebSocket接続管理
```

**`src/cli/gateway-cli/run.ts:54-`**: ゲートウェイ起動コマンド:
1. ログ設定、WebSocketログスタイル設定
2. 開発モード（`--dev`）判定
3. 設定ロードとポート解決
4. `--force`時の既存プロセス強制終了
5. `runGatewayLoop()`への委譲

### 2.2 ゲートウェイループ（ライフサイクル管理）

**`src/cli/gateway-cli/run-loop.ts:14-`**: `runGatewayLoop()`は再起動可能なサーバーループ:

```
[ロック取得] -> [サーバー起動] -> [シグナル待機]
                                    |
                    SIGTERM/SIGINT -> [グレースフルシャットダウン]
                    SIGUSR1 --------> [再起動]
```

重要な設計:
- **ゲートウェイロック**: `acquireGatewayLock()`で排他制御（ファイルロック）
- **SIGUSR1再起動**: macOSアプリからのリロード信号をサポート
- **5秒タイムアウト**: シャットダウンが5秒以内に完了しない場合は強制終了
- **再起動ループ**: SIGUSR1時はサーバーを閉じて再度`start()`を呼ぶ

### 2.3 サーバー実装

**`src/gateway/server.impl.ts:155-`**: `startGatewayServer()`の主要初期化:

1. **レガシー設定マイグレーション**: 古い設定を自動変換
2. **プラグインロード**: `loadOpenClawPlugins()`で全拡張をロード
3. **チャネルマネージャー作成**: `createChannelManager()`
4. **エージェントイベントハンドラー**: `createAgentEventHandler()`
5. **ノードレジストリ**: `NodeRegistry`（モバイル/デスクトップノード管理）
6. **WebSocket接続**: `attachGatewayWsHandlers()`
7. **サイドカー起動**: ブラウザコントロール、Gmail監視、フック、チャネル
8. **ディスカバリー/Tailscale**: ネットワーク公開設定
9. **Cronサービス**: 定期実行ジョブ
10. **ヘルスチェック**: 定期的な健全性監視

### 2.4 ゲートウェイアーキテクチャ図

```
                         ┌─────────────────────────────────────────┐
                         │            Gateway Server               │
                         │         (server.impl.ts)                │
                         │                                         │
  ┌──────────┐          │  ┌─────────────┐  ┌──────────────────┐ │
  │ macOS App├──WS──────┤  │ WS Runtime  │  │  HTTP Endpoints  │ │
  └──────────┘          │  │ (ws-runtime)│  │  /v1/chat/...    │ │
  ┌──────────┐          │  └──────┬──────┘  │  /v1/responses   │ │
  │ iOS App  ├──WS──────┤         │         └────────┬─────────┘ │
  └──────────┘          │         ▼                  │           │
  ┌──────────┐          │  ┌──────────────┐          │           │
  │ Web UI   ├──WS──────┤  │ Agent Event  │◄─────────┘           │
  └──────────┘          │  │  Handler     │                      │
                         │  │(server-chat) │                      │
                         │  └──────┬──────┘                      │
                         │         │                              │
                         │         ▼                              │
                         │  ┌──────────────┐  ┌──────────────┐   │
                         │  │  Channel     │  │   Plugin     │   │
                         │  │  Manager     │  │   Registry   │   │
                         │  │(srv-channels)│  │  (plugins/)  │   │
                         │  └──────┬──────┘  └──────┬───────┘   │
                         └─────────┼─────────────────┼───────────┘
                                   │                 │
                    ┌──────────────┼─────────────────┼──────────┐
                    │              ▼                  ▼          │
                    │  ┌────────────────────────────────────┐   │
                    │  │      Channel Plugins (extensions/) │   │
                    │  ├──────────┬──────────┬──────────────┤   │
                    │  │Telegram  │ Discord  │  Slack  │ ...│   │
                    │  │WhatsApp  │ Signal   │ iMessage│    │   │
                    │  └──────────┴──────────┴──────────────┘   │
                    └───────────────────────────────────────────┘
```

---

## 3. エージェントライフサイクル管理

### 3.1 エージェント実行フロー

```
[メッセージ受信] -> [ルーティング] -> [セッション解決] -> [エージェント実行] -> [応答配信]
```

**`src/commands/agent.ts:64-`**: `agentCommand()`がエージェント実行のエントリーポイント:

1. **入力検証**: メッセージ、セッション指定の確認
2. **エージェントID解決**: 設定ベースのエージェント選択
3. **ワークスペース準備**: `ensureAgentWorkspace()`
4. **モデル選択**: `resolveConfiguredModelRef()`で優先プロバイダー/モデル解決
5. **スキルスナップショット**: `buildWorkspaceSkillSnapshot()`
6. **セッション解決**: ファイルベースのセッション永続化
7. **モデルフォールバック付き実行**: `runWithModelFallback()`
8. **結果配信**: `deliverAgentCommandResult()`

### 3.2 埋め込みPiエージェント（メインエージェントランナー）

**`src/agents/pi-embedded-runner/run.ts:73-`**: `runEmbeddedPiAgent()`が中核実行エンジン:

```
runEmbeddedPiAgent()
  -> resolveSessionLane()      # セッション別レーン取得
  -> resolveGlobalLane()       # グローバルレーン取得
  -> enqueueSession/Global()   # 二重キュー投入
     -> resolveModel()         # モデル解決
     -> resolveContextWindow() # コンテキストウィンドウ設定
     -> resolveAuthProfile()   # 認証プロファイル選択
     -> runEmbeddedAttempt()   # 実際のAPI呼び出し
        -> subscribeEmbeddedPiSession()  # ストリーミング購読
```

重要な設計ポイント:
- **二重レーンキューイング**: セッションレーンとグローバルレーンの両方を通過させることで、セッション内直列化とグローバル同時実行制限の両方を実現
- **認証プロファイルローテーション**: クールダウン中のプロファイルをスキップし、障害時に自動切替
- **コンテキストウィンドウガード**: トークン制限の動的チェック

### 3.3 ストリーミングイベント購読

**`src/agents/pi-embedded-subscribe.ts:30-`**: `subscribeEmbeddedPiSession()`がLLMからのストリーミングイベントを処理:

イベントタイプ（`src/agents/pi-embedded-subscribe.handlers.ts:22-60`）:
- `message_start`: アシスタント応答開始
- `message_update`: 応答の差分更新（ストリーミング）
- `message_end`: 応答完了
- `tool_execution_start`: ツール実行開始（タイピング表示など）
- `tool_execution_update`: ツール実行中の進捗
- `tool_execution_end`: ツール実行完了
- `agent_start`/`agent_end`: エージェント全体のライフサイクル
- `auto_compaction_start`/`end`: コンテキスト圧縮

### 3.4 モデルフォールバック

**`src/agents/model-fallback.ts:1-`**: フォールバック戦略:

1. プライマリモデルで実行を試行
2. 失敗時に`FailoverError`を分類（認証、レート制限、課金、タイムアウト等）
3. 設定された`fallbacks`リストの順に代替モデルを試行
4. 画像関連エラー時は画像対応モデルに限定した候補リストで再試行
5. AbortError（ユーザー中断）は即座にリスロー

### 3.5 エージェントイベントシステム

**`src/infra/agent-events.ts:1-80`**: イベント駆動アーキテクチャ:

```typescript
type AgentEventPayload = {
  runId: string;       // 実行ID
  seq: number;         // モノトニックシーケンス
  stream: AgentEventStream;  // lifecycle | tool | assistant | error
  ts: number;          // タイムスタンプ
  data: Record<string, unknown>;
  sessionKey?: string;
};
```

- `emitAgentEvent()`: イベント発行（runId単位でseqをインクリメント）
- `onAgentEvent()`: リスナー登録
- `registerAgentRunContext()`: 実行コンテキスト登録（sessionKey, verboseLevel等）

---

## 4. メッセージルーティング

### 4.1 ルート解決

**`src/routing/resolve-route.ts:167-260`**: `resolveAgentRoute()`が受信メッセージのルーティング先を決定:

マッチング優先順位:
1. **binding.peer**: 特定のDM/グループにバインドされたエージェント
2. **binding.peer.parent**: スレッドの親ピアにバインドされたエージェント
3. **binding.guild**: Discord guild IDによるマッチ
4. **binding.team**: Slack team IDによるマッチ
5. **binding.account**: チャネルアカウントによるマッチ
6. **binding.channel**: ワイルドカード（`*`）アカウントによるマッチ
7. **default**: デフォルトエージェント

### 4.2 セッションキー構造

**`src/routing/session-key.ts`**: セッションキーはエージェントのステートフルな会話を一意に識別:

```
{agentId}:{channel}:{accountId}:{peerKind}:{peerId}
```

DMスコープ設定により、セッション分離の粒度を制御:
- `main`: 全DMを1セッションに集約
- `per-peer`: ピアごとに独立セッション
- `per-channel-peer`: チャネル+ピアごとに独立
- `per-account-channel-peer`: アカウント+チャネル+ピアごとに独立

### 4.3 ルーティングデータフロー図

```
[受信メッセージ]
     │
     ▼
┌─────────────────────┐
│  Channel Plugin     │
│  (Telegram/Discord  │
│   /Slack/...)       │
│  -> 正規化          │
│  -> allowlist判定   │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  resolveAgentRoute()│
│  (routing/          │
│   resolve-route.ts) │
│                     │
│  入力:              │
│  - channel          │
│  - accountId        │
│  - peer (kind, id)  │
│  - guildId/teamId   │
│                     │
│  出力:              │
│  - agentId          │
│  - sessionKey       │
│  - matchedBy        │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Agent Runner       │
│  (commands/agent.ts)│
│                     │
│  -> セッション解決  │
│  -> モデル選択      │
│  -> 実行            │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Channel Outbound   │
│  (cli/deps.ts       │
│   outbound送信)     │
│                     │
│  -> 元チャネルへ返信│
└─────────────────────┘
```

---

## 5. プラグイン/拡張システム

### 5.1 プラグインアーキテクチャ

**`src/plugins/loader.ts:1-100`**: プラグインローダーの仕組み:

1. `jiti`（Just-In-Time TypeScript）でextensions/配下のTSファイルを直接実行
2. `openclaw/plugin-sdk`をエイリアス解決してプラグインに提供
3. 設定によるenable/disable制御

**`src/plugins/registry.ts:124-161`**: `PluginRegistry`型:

```typescript
type PluginRegistry = {
  plugins: PluginRecord[];         // プラグインメタデータ
  tools: PluginToolRegistration[];  // エージェントツール
  hooks: PluginHookRegistration[];  // フックハンドラー
  channels: PluginChannelRegistration[];  // チャネルプラグイン
  providers: PluginProviderRegistration[];  // AIプロバイダー
  gatewayHandlers: GatewayRequestHandlers; // ゲートウェイRPCメソッド
  httpHandlers: PluginHttpRegistration[];  // HTTPハンドラー
  httpRoutes: PluginHttpRouteRegistration[];  // HTTPルート
  cliRegistrars: PluginCliRegistration[];    // CLIコマンド
  services: PluginServiceRegistration[];     // バックグラウンドサービス
  commands: PluginCommandRegistration[];     // コマンド定義
  diagnostics: PluginDiagnostic[];           // 診断情報
};
```

### 5.2 プラグイン登録API

**`src/plugin-sdk/index.ts:1-375`**: Plugin SDKのエクスポート。プラグインが利用可能なAPI:

プラグインの登録メソッド（`OpenClawPluginApi`経由）:
- `api.registerChannel()`: チャネルプラグイン登録
- `api.registerTool()`: エージェントツール登録
- `api.registerHook()`: フックハンドラー登録
- `api.registerProvider()`: AIプロバイダー登録
- `api.registerGatewayMethod()`: ゲートウェイRPCメソッド登録
- `api.registerHttpHandler()`: HTTPハンドラー登録
- `api.registerCliCommand()`: CLIコマンド登録
- `api.registerService()`: バックグラウンドサービス登録

### 5.3 拡張の実例（Discordプラグイン）

**`extensions/discord/package.json`**:
```json
{
  "name": "@openclaw/discord",
  "openclaw": { "extensions": ["./index.ts"] }
}
```

**`extensions/discord/index.ts:1-17`**: 典型的なプラグイン構造:

```typescript
const plugin = {
  id: "discord",
  name: "Discord",
  description: "Discord channel plugin",
  configSchema: emptyPluginConfigSchema(),
  register(api: OpenClawPluginApi) {
    setDiscordRuntime(api.runtime);
    api.registerChannel({ plugin: discordPlugin });
  },
};
export default plugin;
```

### 5.4 チャネルプラグインインターフェース

**`src/channels/plugins/types.ts`**: チャネルプラグインが実装するアダプター群:

```
ChannelPlugin
  ├── ChannelGatewayAdapter      # ゲートウェイ起動/停止
  ├── ChannelMessagingAdapter     # メッセージ送信
  ├── ChannelOutboundAdapter      # アウトバウンド送信
  ├── ChannelStreamingAdapter     # ストリーミング対応
  ├── ChannelStatusAdapter        # ステータス報告
  ├── ChannelConfigAdapter        # 設定管理
  ├── ChannelAuthAdapter          # 認証（QR等）
  ├── ChannelPairingAdapter       # デバイスペアリング
  ├── ChannelSetupAdapter         # 初期セットアップ
  ├── ChannelSecurityAdapter      # セキュリティポリシー
  ├── ChannelMentionAdapter       # メンション制御
  ├── ChannelThreadingAdapter     # スレッド対応
  ├── ChannelDirectoryAdapter     # コンタクトディレクトリ
  └── ChannelOnboardingAdapter    # オンボーディング
```

### 5.5 チャネルマネージャー

**`src/gateway/server-channels.ts:64-`**: `createChannelManager()`がチャネルのライフサイクルを管理:

```typescript
type ChannelManager = {
  getRuntimeSnapshot: () => ChannelRuntimeSnapshot;
  startChannels: () => Promise<void>;
  startChannel: (channel: ChannelId, accountId?) => Promise<void>;
  stopChannel: (channel: ChannelId, accountId?) => Promise<void>;
  markChannelLoggedOut: (...) => void;
};
```

各チャネルはアカウント単位で起動/停止可能。`ChannelRuntimeStore`がAbortController、タスクPromise、ランタイムスナップショットを管理。

---

## 6. プロバイダーパターン（LLM抽象化）

### 6.1 プロバイダー構造

AIプロバイダーの抽象化は以下の階層で実現:

```
src/agents/
  ├── defaults.ts            # DEFAULT_PROVIDER, DEFAULT_MODEL
  ├── model-selection.ts     # モデル解決ロジック
  ├── model-catalog.ts       # モデルカタログ管理
  ├── model-auth.ts          # 認証プロファイル管理
  ├── model-fallback.ts      # フォールバック戦略
  ├── auth-profiles.ts       # 認証プロファイルストア
  ├── models-config.ts       # models.json設定管理
  └── pi-embedded-runner/
      ├── run.ts             # メイン実行ループ
      ├── model.ts           # モデル解決
      ├── run/
      │   ├── attempt.ts     # 単一試行
      │   └── payloads.ts    # ペイロード構築
      └── types.ts           # 型定義
```

### 6.2 ツール定義と登録

**`src/agents/pi-tools.ts:1-80`**: エージェントに提供されるツール群:

コアツール（`@mariozechner/pi-coding-agent`由来）:
- `read`/`write`/`edit`: ファイル操作
- `exec`/`process`: コマンド実行（サンドボックス対応）
- `apply_patch`: パッチ適用

OpenClaw固有ツール（`openclaw-tools.ts`）:
- メッセージ送信、セッション管理、カメラ、エージェント間通信

チャネル固有ツール（`channel-tools.ts`）:
- 各チャネルプラグインが提供するアクションツール

ツールポリシー（`pi-tools.policy.ts`）:
- グループポリシー、オーナー限定ポリシー、サブエージェントポリシーによるツール利用制限

### 6.3 システムプロンプト構築

**`src/agents/system-prompt.ts:1-80`**: システムプロンプトのセクション構成:

- **Skills**: ワークスペーススキルの選択指示
- **Memory Recall**: メモリ検索・取得の指示
- **User Identity**: ユーザー識別情報
- **Current Date & Time**: タイムゾーン情報
- **Tooling**: 利用可能ツールの説明
- **Workspace**: ワークスペース情報
- **Runtime**: 実行コンテキスト

`PromptMode`により`full`/`minimal`/`none`の3段階で詳細度を制御（サブエージェントは`minimal`）。

---

## 7. 並行制御とレーン機構

### 7.1 コマンドキュー

**`src/process/command-queue.ts:1-60`**: インプロセスのコマンドキュー:

```typescript
type LaneState = {
  lane: string;
  queue: QueueEntry[];
  active: number;
  maxConcurrent: number;  // デフォルト1
  draining: boolean;
};
```

**`src/process/lanes.ts:1-7`**: レーン定義:

```typescript
enum CommandLane {
  Main = "main",        // メインの自動応答ワークフロー
  Cron = "cron",        // 定期実行ジョブ
  Subagent = "subagent", // サブエージェント実行
  Nested = "nested",    // ネスト実行
}
```

### 7.2 二重キュー戦略

`runEmbeddedPiAgent()`では、セッションレーンとグローバルレーンの二重キューを使用:

```
[メッセージ] -> [セッションレーン] -> [グローバルレーン] -> [API呼び出し]
                (セッション内直列)    (全体同時実行制限)
```

これにより:
- 同一セッションへの同時メッセージは直列化される
- 異なるセッションは並行処理可能
- グローバルレーンでAPIレート制限を管理

---

## 8. エラーハンドリングとリカバリ機構

### 8.1 階層的エラーハンドリング

```
[プロセスレベル]
  uncaughtException -> ログ出力 + exit(1)
  unhandledRejection -> ログ出力

[ゲートウェイレベル]
  GatewayLockError -> ポート診断情報表示
  設定エラー -> doctorコマンド誘導

[エージェント実行レベル]
  FailoverError -> モデルフォールバック
  AuthError -> 認証プロファイルローテーション
  BillingError -> ユーザー通知
  ContextOverflow -> コンパクション実行
  RateLimit -> フォールバック
  Timeout -> フォールバック or リスロー

[ツール実行レベル]
  ToolError -> ツール結果としてエージェントに返却
  AbortError -> 実行中断
```

### 8.2 フェイルオーバーエラー分類

**`src/agents/pi-embedded-helpers.ts`**: エラー分類関数群:

- `isAuthAssistantError()`: 認証エラー
- `isBillingAssistantError()`: 課金エラー
- `isContextOverflowError()`: コンテキスト超過
- `isFailoverAssistantError()`: フェイルオーバー対象
- `isRateLimitAssistantError()`: レート制限
- `isTimeoutErrorMessage()`: タイムアウト
- `classifyFailoverReason()`: 上記を統合した分類

### 8.3 認証プロファイルローテーション

**`src/agents/auth-profiles.ts`**: 認証プロファイルの管理:

- 複数のAPIキー/認証情報を登録可能
- 障害発生時に自動ローテーション
- クールダウン期間の管理
- `markAuthProfileFailure()`/`markAuthProfileGood()`で状態追跡

### 8.4 ブートファイルによる起動チェック

**`src/gateway/boot.ts:53-95`**: `runBootOnce()`:

ワークスペースの`BOOT.md`を読み込み、エージェントに起動時チェックを実行させる。メッセージ送信が必要な場合はツール経由で実行。

---

## 9. フック（Hook）システム

### 9.1 内部フック

**`src/hooks/internal-hooks.ts:1-60`**: イベント駆動フックシステム:

```typescript
type InternalHookEventType = "command" | "session" | "agent" | "gateway";

interface InternalHookEvent {
  type: InternalHookEventType;
  action: string;
  sessionKey: string;
  context: Record<string, unknown>;
  timestamp: Date;
  messages: string[];
}
```

フック登録:
- `registerInternalHook('command', handler)`: 全commandイベント
- `registerInternalHook('command:new', handler)`: 特定アクション
- `registerInternalHook('agent:bootstrap', handler)`: ブートストラップフック

プラグインからの登録: `api.registerHook(events, handler)`

---

## 10. 「自己成長するエージェント」開発への示唆

### 10.1 ハーネス設計の観点からの要点

**レーン機構の活用**:
OpenClawのレーン型並行制御は、自己成長エージェントにとって重要な示唆を提供する。エージェントが自己改善タスクを実行する際、メインの応答フローをブロックしない`Cron`や`Subagent`レーンで安全に実行できる設計が有効。

**プラグインアーキテクチャの拡張**:
`PluginRegistry`のパターンは、エージェントが自身の能力を動的に拡張する機構の基盤となりうる。新しいツールやスキルをランタイムで登録可能なアーキテクチャにより、エージェントが学習した能力を即座に利用可能にできる。

**フック駆動の自己監視**:
`InternalHookEvent`システムにより、エージェントの実行を非侵入的に監視できる。`agent_start`/`agent_end`イベントを購読して実行パターンを分析し、自己最適化の入力とすることが可能。

**フェイルオーバーパターンの応用**:
`FailoverError`分類と`model-fallback.ts`のフォールバック戦略は、自己成長エージェントのリカバリ機構の参考になる。エラー種別に応じて異なる回復戦略を適用し、認証プロファイルローテーションのように複数の代替手段を自動的に試行するパターン。

**セッション永続化と文脈管理**:
セッションキーによる状態管理とコンテキストウィンドウガードは、長期実行エージェントのメモリ管理の基盤。自動コンパクション（`auto_compaction`）によるコンテキスト圧縮は、無限に成長する会話履歴を持続可能に管理する仕組み。

### 10.2 設計パターンの要約表

| パターン | OpenClawの実装 | 自己成長への応用 |
|----------|---------------|-----------------|
| レーン並行制御 | `CommandLane` + `command-queue.ts` | 自己改善タスクの安全な並行実行 |
| プラグイン動的登録 | `PluginRegistry` + `plugin-sdk` | ランタイムでの能力拡張 |
| フック駆動監視 | `InternalHookEvent` | 実行パターン分析と自己最適化 |
| フェイルオーバー | `model-fallback.ts` + `auth-profiles.ts` | エラー分類に基づく回復戦略 |
| セッション永続化 | session-key + コンパクション | 長期記憶と文脈管理 |
| DI/モック可能設計 | `CliDeps` + `RuntimeEnv` | テスト駆動の自己検証 |
| 遅延ロード | `register.subclis.ts` | 必要な能力のオンデマンドロード |
| ブートストラップ | `BOOT.md` + `boot.ts` | 起動時の自己診断・自己修復 |

### 10.3 推奨アーキテクチャアプローチ

自己成長エージェントのハーネスを設計する場合、以下のOpenClawパターンの採用を推奨:

1. **二重レーンキューイング**: メイン応答フローと自己改善フローを分離しつつ、リソース競合を制御
2. **プラグインレジストリパターン**: ツール/スキル/プロバイダーを動的に登録・解除できる機構
3. **イベントソーシング**: `AgentEventPayload`のようなモノトニックシーケンス付きイベントストリームで全実行を記録
4. **フェイルオーバーチェーン**: 単一障害点を排除し、複数の代替手段を自動的に試行
5. **コンテキストウィンドウガード**: 自己成長に伴うメモリ膨張を自動コンパクションで管理

---

## 付録: 主要ファイルパス一覧

| コンポーネント | ファイルパス |
|---------------|------------|
| CLIエントリー | `src/entry.ts` |
| CLI起動 | `src/cli/run-main.ts` |
| プログラムビルド | `src/cli/program/build-program.ts` |
| サブコマンド登録 | `src/cli/program/register.subclis.ts` |
| コマンドレジストリ | `src/cli/program/command-registry.ts` |
| DI定義 | `src/cli/deps.ts` |
| ランタイム環境 | `src/runtime.ts` |
| ゲートウェイ起動 | `src/cli/gateway-cli/run.ts` |
| ゲートウェイループ | `src/cli/gateway-cli/run-loop.ts` |
| サーバー実装 | `src/gateway/server.impl.ts` |
| サイドカー起動 | `src/gateway/server-startup.ts` |
| チャネルマネージャー | `src/gateway/server-channels.ts` |
| チャットハンドラー | `src/gateway/server-chat.ts` |
| ブート処理 | `src/gateway/boot.ts` |
| エージェントコマンド | `src/commands/agent.ts` |
| 埋め込みエージェント | `src/agents/pi-embedded-runner/run.ts` |
| ストリーム購読 | `src/agents/pi-embedded-subscribe.ts` |
| イベントハンドラー | `src/agents/pi-embedded-subscribe.handlers.ts` |
| ルート解決 | `src/routing/resolve-route.ts` |
| セッションキー | `src/routing/session-key.ts` |
| チャネルレジストリ | `src/channels/registry.ts` |
| チャネルプラグイン | `src/channels/plugins/index.ts` |
| プラグインローダー | `src/plugins/loader.ts` |
| プラグインレジストリ | `src/plugins/registry.ts` |
| プラグインCLI | `src/plugins/cli.ts` |
| プラグインSDK | `src/plugin-sdk/index.ts` |
| ツール定義 | `src/agents/pi-tools.ts` |
| システムプロンプト | `src/agents/system-prompt.ts` |
| モデルフォールバック | `src/agents/model-fallback.ts` |
| 認証プロファイル | `src/agents/auth-profiles.ts` |
| エージェントイベント | `src/infra/agent-events.ts` |
| コマンドキュー | `src/process/command-queue.ts` |
| レーン定義 | `src/process/lanes.ts` |
| 内部フック | `src/hooks/internal-hooks.ts` |
| Extension例(Discord) | `extensions/discord/index.ts` |

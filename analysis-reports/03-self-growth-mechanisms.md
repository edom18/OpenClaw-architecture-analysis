# OpenClaw 自己成長メカニズム分析レポート

## 概要サマリー

OpenClawは、AIエージェントが環境から学び、自己改善し、新しい能力を獲得するための複数の「自己成長メカニズム」を備えたプラットフォームである。本レポートでは、以下の8つの主要メカニズムを分析し、「自己成長するAIエージェント」を自分で構築するための実践的なデザインパターンを抽出する。

**分析対象メカニズム:**

1. **ツール利用システム** -- 動的なツール登録・解決・ポリシー制御
2. **ツール結果のフィードバックループ** -- 実行結果からの行動修正サイクル
3. **メモリシステム (学習・適応機構)** -- ベクトル検索・ハイブリッド検索によるRAG型学習
4. **オンボーディング/セットアップフロー** -- 新環境への適応的セットアップ
5. **コンフィグ進化** -- 設定の動的更新・マイグレーション
6. **Doctor/診断機能** -- 自己診断と自動修復
7. **メディアパイプライン** -- 画像/音声/動画の処理パイプライン
8. **スキルシステム** -- 動的な能力拡張メカニズム

---

## 1. ツール利用システム

### 1.1 ツール定義と登録

OpenClawのツールシステムは多層的なアーキテクチャを持つ。

**コアツール（ビルトイン）:**

`src/agents/openclaw-tools.ts:22-60` で `createOpenClawTools()` がコアツール群を組み立てる:

```
createBrowserTool, createCanvasTool, createNodesTool,
createCronTool, createGatewayTool, createImageTool,
createMessageTool, createSessionStatusTool,
createSessionsHistoryTool, createSessionsListTool,
createSessionsSendTool, createSessionsSpawnTool,
createWebFetchTool, createWebSearchTool, createTtsTool,
createAgentsListTool
```

**コーディングツール（SDK由来）:**

`src/agents/pi-tools.ts:1-7` で `@mariozechner/pi-coding-agent` から:
```
codingTools, createEditTool, createReadTool, createWriteTool, readTool
```

加えて、`createExecTool`, `createProcessTool`（bash実行）, `createApplyPatchTool`（パッチ適用）が自前で定義。

**プラグインツール:**

`src/plugins/registry.ts:146-515` の `createPluginRegistry()` がプラグインAPIを提供:

```typescript
// src/plugins/registry.ts:168-193
const registerTool = (record, tool, opts) => {
  const factory = typeof tool === "function"
    ? tool
    : (_ctx) => tool;
  registry.tools.push({ pluginId, factory, names, optional, source });
};
```

`src/plugins/tools.ts:43-129` の `resolvePluginTools()` がプラグインツールを動的に解決:
- プラグインIDと既存ツール名の衝突検出
- オプショナルツールのアローリスト制御
- WeakMapによるメタデータ追跡

### 1.2 ツールポリシー制御

`src/agents/tool-policy.ts` がツールの利用ポリシーを制御する:

**プロファイルベースの制御 (tool-policy.ts:63-80):**
```
minimal: allow: ["session_status"]
coding:  allow: ["group:fs", "group:runtime", "group:sessions", "group:memory", "image"]
messaging: allow: ["group:messaging", "sessions_list", ...]
full:    {} (制限なし)
```

**グループベースの展開 (tool-policy.ts:15-59):**
```
group:memory  → ["memory_search", "memory_get"]
group:web     → ["web_search", "web_fetch"]
group:fs      → ["read", "write", "edit", "apply_patch"]
group:runtime → ["exec", "process"]
```

**オーナー限定ツール (tool-policy.ts:61, 87-110):**
```typescript
const OWNER_ONLY_TOOL_NAMES = new Set(["whatsapp_login"]);
```
`applyOwnerOnlyToolPolicy()` がセンダーがオーナーでない場合にツール実行を拒否。

### 1.3 自己成長との関連

ツールシステムの成長パターン:
- **プラグインによる水平拡張**: 新しいツールをプラグインとして追加でき、コア変更不要
- **ポリシーによる段階的権限付与**: エージェントの信頼レベルに応じてツール利用を制限/緩和
- **動的解決**: ツールは実行時に解決され、設定変更で利用可能ツールが変わる

---

## 2. ツール結果のフィードバックループ

### 2.1 ツール実行のイベントフロー

`src/agents/pi-embedded-subscribe.handlers.tools.ts:39-100` が実行フローを管理:

```
[LLMがツール呼出を決定]
     ↓
handleToolExecutionStart()
  - emitAgentEvent({ phase: "start", name, toolCallId, args })
  - onToolResult callback（UI通知）
  - メッセージングツールの送信追跡
     ↓
[ツール実行]
     ↓
handleToolExecutionEnd()
  - extractToolResultText() / extractToolErrorMessage()
  - sanitizeToolResult()（結果の正規化）
  - emitAgentEvent({ phase: "end", name, toolCallId, result/error })
  - LLMへの結果返却（次の行動決定へ）
```

### 2.2 ツール結果のガード

`src/agents/session-tool-result-guard.ts:45-62` の `installSessionToolResultGuard()`:

```typescript
// セッションマネージャのappendMessageをインターセプト
const originalAppend = sessionManager.appendMessage.bind(sessionManager);
const pending = new Map<string, string | undefined>();
```

- アシスタントのtoolCallに対して対応するtoolResultが来るまで追跡
- 不足したtoolResultを合成（`makeMissingToolResult()`）
- `transformToolResultForPersistence` フックで永続化前に変換可能

### 2.3 コンテキストプルーニング

`src/agents/pi-extensions/context-pruning/tools.ts:53-60`:

```typescript
export function makeToolPrunablePredicate(match) {
  const deny = compilePatterns(match.deny);
  const allow = compilePatterns(match.allow);
  return (toolName) => {
    // allow/denyパターンで剪定可能なツール結果を判定
  };
}
```

古いツール結果をコンテキストから剪定し、コンテキストウィンドウを効率的に利用。

### 2.4 コンパクション（要約化）

`src/agents/compaction.ts:7-14`:

```typescript
export const BASE_CHUNK_RATIO = 0.4;
export const MIN_CHUNK_RATIO = 0.15;
export const SAFETY_MARGIN = 1.2;
```

会話が長くなるとコンパクション（要約）が実行される:
- メッセージをトークン数ベースでチャンクに分割
- LLMで各チャンクを要約
- 複数サマリーをマージして一貫性を保持
- 「決定事項、TODO、オープンクエスチョン、制約」を保持するよう指示

**フィードバックループ図:**

```
┌──────────────────────────────────────────────────────┐
│                   LLM (推論エンジン)                  │
│  ┌─────────────┐    ┌────────────┐    ┌───────────┐  │
│  │ システム     │    │ ツール呼出 │    │  応答生成 │  │
│  │ プロンプト   │←──│  決定      │←──│           │  │
│  └──────┬──────┘    └─────┬──────┘    └───────────┘  │
│         │                 │                           │
└─────────│─────────────────│───────────────────────────┘
          │                 ↓
          │    ┌─────────────────────┐
          │    │   ツール実行エンジン  │
          │    │  ┌───────────────┐  │
          │    │  │ exec/read/    │  │
          │    │  │ write/search  │  │
          │    │  │ message/...   │  │
          │    │  └───────┬───────┘  │
          │    └──────────│──────────┘
          │               ↓
          │    ┌─────────────────────┐
          │    │    結果処理パイプライン │
          │    │  ┌───────────────┐  │
          │    │  │ sanitize      │  │
          │    │  │ transform     │  │
          │    │  │ guard         │  │
          │    │  └───────┬───────┘  │
          │    └──────────│──────────┘
          │               ↓
          │    ┌─────────────────────┐
          │    │  セッション永続化    │
          │    │  (transcript.jsonl) │──→ メモリインデックス
          │    └──────────┬──────────┘        (再利用)
          │               │
          │               ↓
          │    ┌─────────────────────────┐
          │    │ コンパクション / プルーニング │
          │    │ (要約で重要情報を保持)    │
          │    └──────────┬──────────────┘
          │               │
          └───────────────┘ (次のターンへ)
```

---

## 3. メモリシステム（学習・適応機構）

### 3.1 アーキテクチャ概要

`src/memory/manager.ts` の `MemoryIndexManager` クラス (108-2355行) がメモリシステムの中核。

**データソース:**
- `memory` ソース: ワークスペースの `MEMORY.md`, `memory.md`, `memory/` ディレクトリ
- `sessions` ソース: エージェントのセッショントランスクリプト (`.jsonl`)
- 追加パス: `settings.extraPaths` で任意のMarkdownファイルを追加

**インデックス構造 (SQLite):**
```
files テーブル:    path, source, hash, mtime, size
chunks テーブル:   id, path, source, start_line, end_line, hash, model, text, embedding
chunks_vec テーブル: id, embedding (sqlite-vec拡張)
chunks_fts テーブル: FTS5全文検索
embedding_cache:    provider, model, hash, embedding (キャッシュ)
```

### 3.2 ハイブリッド検索

`src/memory/manager.ts:261-309` の `search()`:

```typescript
async search(query, opts?) {
  // 1. ファイルウォッチャーが変更検出 → 自動同期
  void this.warmSession(opts?.sessionKey);
  if (this.settings.sync.onSearch && (this.dirty || this.sessionsDirty)) {
    void this.sync({ reason: "search" });
  }

  // 2. キーワード検索 (BM25)
  const keywordResults = hybrid.enabled
    ? await this.searchKeyword(cleaned, candidates) : [];

  // 3. ベクトル検索 (embedding)
  const queryVec = await this.embedQueryWithTimeout(cleaned);
  const vectorResults = hasVector
    ? await this.searchVector(queryVec, candidates) : [];

  // 4. ハイブリッドマージ
  const merged = this.mergeHybridResults({
    vector: vectorResults,
    keyword: keywordResults,
    vectorWeight: hybrid.vectorWeight,
    textWeight: hybrid.textWeight,
  });
}
```

### 3.3 リアルタイム同期

**ファイルウォッチャー (manager.ts:807-841):**
```typescript
private ensureWatcher() {
  this.watcher = chokidar.watch(watchPaths, {
    ignoreInitial: true,
    awaitWriteFinish: { stabilityThreshold, pollInterval: 100 },
  });
  this.watcher.on("add", markDirty);
  this.watcher.on("change", markDirty);
  this.watcher.on("unlink", markDirty);
}
```

**セッションリスナー (manager.ts:843-857):**
```typescript
private ensureSessionListener() {
  this.sessionUnsubscribe = onSessionTranscriptUpdate((update) => {
    if (this.isSessionFileForAgent(update.sessionFile)) {
      this.scheduleSessionDirty(sessionFile);
    }
  });
}
```

**デルタベース同期 (manager.ts:872-908):**
- バイト数閾値とメッセージ数閾値で差分同期をトリガー
- 不要な全再インデックスを回避

### 3.4 エンベディングプロバイダー

- OpenAI、Gemini、ローカル（node-llama）の3プロバイダー
- バッチエンベディング対応（`runOpenAiEmbeddingBatches`, `runGeminiEmbeddingBatches`）
- **フォールバック機構 (manager.ts:1360-1396):** プロバイダー障害時に自動的に代替へ切替

### 3.5 システムプロンプトへのメモリ統合

`src/agents/system-prompt.ts:40-66` の `buildMemorySection()`:

```
## Memory Recall
Before answering anything about prior work, decisions, dates, people,
preferences, or todos: run memory_search on MEMORY.md + memory/*.md;
then use memory_get to pull only the needed lines.
```

→ エージェントは毎回の応答前にメモリを検索するよう指示される。

---

## 4. オンボーディング/セットアップフロー

### 4.1 対話的オンボーディング

`src/commands/onboard.ts:12-80` の `onboardCommand()`:

```
onboardCommand()
  ├── runInteractiveOnboarding()  (対話型)
  │    ├── onboard-auth.ts         (認証設定)
  │    ├── onboard-channels.ts     (チャネル設定)
  │    ├── onboard-hooks.ts        (フック設定)
  │    └── onboard-skills.ts       (スキル設定)
  │
  └── runNonInteractiveOnboarding() (非対話型/CI用)
```

**適応的な分岐:**
- プラットフォーム検出（Windows→WSL案内, Linux→systemdチェック, macOS→LaunchAgent）
- 認証方式の選択（OAuth, APIキー, Copilot, OpenAI, Gemini, etc.）
- チャネルの動的検出と設定

### 4.2 ワークスペースブートストラップ

`src/agents/workspace.ts:21-29` で定義されるファイル群:

```
AGENTS.md    -- エージェントの振る舞い定義
SOUL.md      -- エージェントの「魂」（性格・価値観）
TOOLS.md     -- ツール使用ガイドライン
IDENTITY.md  -- アイデンティティ定義
USER.md      -- ユーザー情報
HEARTBEAT.md -- ハートビート設定
BOOTSTRAP.md -- 初期起動命令
MEMORY.md    -- メモリファイル
```

テンプレートから初期生成し、エージェントが自ら編集可能。

### 4.3 ブートストラップフック

`src/agents/bootstrap-hooks.ts:7-31`:

```typescript
export async function applyBootstrapHookOverrides(params) {
  const event = createInternalHookEvent("agent", "bootstrap", sessionKey, context);
  await triggerInternalHook(event);
  const updated = (event.context as AgentBootstrapHookContext).bootstrapFiles;
  return Array.isArray(updated) ? updated : params.files;
}
```

→ プラグインがブートストラップファイルを動的に差し替え可能。

---

## 5. コンフィグ進化

### 5.1 設定システムの構造

`src/config/config.ts` がエクスポートする機能群:

```typescript
// IO操作
createConfigIO, loadConfig, parseConfigJson5,
readConfigFileSnapshot, writeConfigFile

// マイグレーション
migrateLegacyConfig

// 検証
validateConfigObject, validateConfigObjectWithPlugins

// Zodスキーマ
OpenClawSchema
```

### 5.2 レガシー設定マイグレーション

`src/commands/doctor-state-migrations.ts` → `src/infra/state-migrations.ts`:

```
detectLegacyStateMigrations() -- 旧状態の検出
runLegacyStateMigrations()    -- 実行
```

Doctor内で自動検出し、インタラクティブに修復提案。

### 5.3 設定の動的更新

`src/commands/doctor-config-flow.ts` の `loadAndMaybeMigrateDoctorConfig()`:
- 設定ファイルのスナップショットを読み取り
- バリデーション実行
- マイグレーション必要なら変換
- ユーザー確認後に書き込み

### 5.4 ランタイムオーバーライド

`src/config/runtime-overrides.ts`:
- 環境変数によるランタイム設定上書き
- プロファイルベースの設定切替（`OPENCLAW_PROFILE`）

---

## 6. Doctor/診断機能

### 6.1 アーキテクチャ

`src/commands/doctor.ts:65-313` の `doctorCommand()` は以下の診断・修復モジュールで構成:

| モジュール | ファイル | 機能 |
|---|---|---|
| UI修復 | `doctor-ui.ts` | UIプロトコルの鮮度チェック |
| インストール | `doctor-install.ts` | ソースインストールの問題検出 |
| プラットフォーム | `doctor-platform-notes.ts` | macOS LaunchAgent, 環境変数 |
| 設定フロー | `doctor-config-flow.ts` | 設定の読込・マイグレーション |
| 認証 | `doctor-auth.ts` | OAuthプロファイル修復、非推奨プロファイル削除 |
| ゲートウェイ | `doctor-gateway-*.ts` | サービス設定修復、ヘルスチェック |
| サンドボックス | `doctor-sandbox.ts` | Dockerイメージ修復 |
| セキュリティ | `doctor-security.ts` | セキュリティ警告 |
| 状態整合性 | `doctor-state-integrity.ts` | 状態ディレクトリの整合性 |
| マイグレーション | `doctor-state-migrations.ts` | レガシー状態の検出と移行 |
| ワークスペース | `doctor-workspace.ts` | メモリシステムの提案 |
| 補完 | `doctor-completion.ts` | シェル補完の設定 |
| アップデート | `doctor-update.ts` | バージョン更新の提案 |

### 6.2 診断→提案→修復フロー

```
doctorCommand()
  │
  ├─→ [検出] 問題を検出
  │   例: gateway.mode未設定、auth tokenなし、legacyステート
  │
  ├─→ [提案] ユーザーに修復を提案
  │   prompter.confirmRepair({
  │     message: "Generate and configure a gateway token now?",
  │     initialValue: true,
  │   })
  │
  ├─→ [修復] 承認されたら自動修復
  │   cfg = { ...cfg, gateway: { auth: { mode: "token", token: nextToken } } }
  │
  └─→ [永続化] 設定ファイルに書き込み
      writeConfigFile(cfg);
```

### 6.3 ワークスペース提案

`src/commands/doctor-workspace.ts:15-38`:

```typescript
export async function shouldSuggestMemorySystem(workspaceDir) {
  // MEMORY.md が存在しなければメモリシステム導入を提案
  // AGENTS.md内にmemory.mdの参照があれば提案しない
}
```

→ Doctorは単に問題を報告するだけでなく、**成長の機会を提案する**。

---

## 7. メディアパイプライン

### 7.1 メディア取得と保存

`src/media/store.ts:1-60`:
- `resolveMediaDir()`: `~/.openclaw/media/` にメディアを保存
- `extractOriginalFilename()`: UUID付きファイル名から元ファイル名を復元
- `sanitizeFilename()`: クロスプラットフォーム安全なファイル名生成
- SSRF防御: `resolvePinnedHostname()` でリダイレクト攻撃を防止

### 7.2 メディア解析

`src/media/parse.ts:56-100` の `splitMediaFromOutput()`:
- エージェント出力から `MEDIA: <url>` トークンを抽出
- フェンスコードブロック内のMEDIAトークンは無視
- `[[audio_as_voice]]` タグで音声として再生を指示
- URLとローカルパスの両方をサポート（`./` で始まるパスのみ）

### 7.3 MIME検出と拡張性

`src/media/mime.ts`: Content-Typeの検出と拡張子マッピング
`src/media/host.ts`: メディアファイルのHTTPサービング
`src/media/server.ts`: メディアサーバー
`src/media/fetch.ts`: 外部メディアの取得
`src/media/audio.ts`: 音声処理
`src/media/image-ops.ts`: 画像操作
`src/media/input-files.ts`: 入力ファイルの処理

---

## 8. スキルシステム（動的能力拡張）

### 8.1 スキルのロードと適用

`src/agents/skills/workspace.ts` がスキルの読み込みを管理:

**スキルソース:**
1. バンドルスキル（`src/agents/skills/bundled-dir.ts`）
2. マネージドスキル（`~/.openclaw/skills/`）
3. ワークスペーススキル（ワークスペース内 `skills/`）
4. プラグインスキル（`resolvePluginSkillDirs()`）

**各スキルの構成:**
```
skills/<skill-name>/
  ├── SKILL.md       -- スキルの定義・実行手順
  ├── handler.ts     -- ハンドラーロジック（オプション）
  └── HOOK.md        -- フック定義（オプション）
```

### 8.2 スキルのシステムプロンプト統合

`src/agents/system-prompt.ts:16-38` の `buildSkillsSection()`:

```typescript
function buildSkillsSection(params) {
  return [
    "## Skills (mandatory)",
    "Before replying: scan <available_skills> <description> entries.",
    "- If exactly one skill clearly applies: read its SKILL.md at <location>...",
    "- If multiple could apply: choose the most specific one...",
    "- If none clearly apply: do not read any SKILL.md.",
  ];
}
```

→ エージェントはスキルを**必要時のみ**読み込み、不要なコンテキスト消費を防止。

### 8.3 スキルの同期

`syncSkillsToWorkspace()`: マネージドスキルをワークスペースに自動同期。インストール・更新・削除を管理。

---

## 9. フックシステム（イベント駆動拡張）

### 9.1 内部フックシステム

`src/hooks/internal-hooks.ts:67-143`:

```typescript
// 登録
export function registerInternalHook(eventKey, handler) {
  handlers.get(eventKey)!.push(handler);
}

// トリガー
export async function triggerInternalHook(event) {
  const typeHandlers = handlers.get(event.type) ?? [];
  const specificHandlers = handlers.get(`${event.type}:${event.action}`) ?? [];
  for (const handler of [...typeHandlers, ...specificHandlers]) {
    await handler(event);
  }
}
```

**イベントタイプ:**
- `command` -- コマンド実行イベント
- `session` -- セッションライフサイクル
- `agent` -- エージェントイベント（bootstrap含む）
- `gateway` -- ゲートウェイイベント

### 9.2 フックメタデータ

`src/hooks/types.ts:10-27`:

```typescript
export type OpenClawHookMetadata = {
  always?: boolean;          // 常に実行
  hookKey?: string;          // 一意キー
  events: string[];          // 対応イベント
  os?: string[];             // OS制限
  requires?: {
    bins?: string[];         // 必要バイナリ
    anyBins?: string[];      // いずれかのバイナリ
    env?: string[];          // 必要環境変数
    config?: string[];       // 必要設定
  };
  install?: HookInstallSpec[];  // インストール仕様
};
```

→ フックは**自己記述的**で、必要な前提条件を宣言する。

---

## 10. モデルフォールバックと耐障害性

### 10.1 認証プロファイルのローテーション

`src/agents/auth-profiles.ts` が認証プロファイルの管理を担う:

```
ensureAuthProfileStore, loadAuthProfileStore, saveAuthProfileStore
markAuthProfileGood, markAuthProfileFailure
markAuthProfileCooldown, isProfileInCooldown
calculateAuthProfileCooldownMs
resolveAuthProfileOrder
```

→ 認証失敗時にクールダウンを設定し、代替プロファイルに自動ローテーション。

### 10.2 モデルフォールバック

`src/agents/model-fallback.ts:54-80`:
- 画像処理失敗時の代替モデル候補を解決
- アローリスト内のモデルのみ使用
- Abort検出で不要なリトライを防止

### 10.3 コンテキストウィンドウガード

`src/agents/context-window-guard.ts:57-74`:
```typescript
export function evaluateContextWindowGuard(params) {
  return {
    shouldWarn: tokens < warnBelow,   // 32,000トークン未満で警告
    shouldBlock: tokens < hardMin,     // 16,000トークン未満でブロック
  };
}
```

---

## 11. 自己成長するエージェントのためのデザインパターン

### パターン1: 階層的ツールレジストリ

**問題:** エージェントの能力を動的に拡張したいが、安全性も確保したい。

**OpenClawの解法:**
```
コアツール (ハードコード)
  ↓
SDK由来ツール (パッケージ依存)
  ↓
プラグインツール (動的ロード)
  ↓
ポリシーフィルタ (allow/deny/group)
  ↓
オーナー限定ガード
```

**実装指針:**
1. ツールを `{ name, description, parameters (JSON Schema), execute }` の統一インターフェースで定義
2. `WeakMap` でツールにメタデータを付与（出自追跡）
3. グループベースのポリシーで一括制御を可能に
4. プラグインは `factory` パターンで遅延生成

### パターン2: 差分ベースのメモリ同期

**問題:** メモリを常に最新に保ちたいが、全再インデックスは遅い。

**OpenClawの解法:**
```
ファイルウォッチャー (chokidar)
  ↓ dirtyフラグ
セッションデルタ追跡 (bytes/messages閾値)
  ↓ デバウンス
差分同期 (ハッシュ比較でスキップ)
  ↓ 変更分のみ
エンベディング生成 + キャッシュ
  ↓
ベクトル/FTSインデックス更新
```

**実装指針:**
1. ファイルシステムウォッチャーで変更を検出
2. ハッシュベースのスキップで不要な処理を回避
3. エンベディングキャッシュで再計算コストを削減
4. バッチ処理（OpenAI/Gemini Batch API）で大量データを効率処理
5. フォールバックプロバイダーで障害耐性を確保

### パターン3: 自己診断→提案→修復サイクル

**問題:** エージェントの環境が壊れたとき、自動復旧したい。

**OpenClawの解法 (Doctor):**
```
Step 1: 全診断モジュールを順次実行
Step 2: 問題を検出したら修復案を提示
Step 3: ユーザー承認後に自動修復
Step 4: 設定ファイルを更新して永続化
Step 5: 最終バリデーションで整合性確認
```

**実装指針:**
1. 診断モジュールを個別ファイルに分離（テスト容易性）
2. `prompter` パターンで対話/非対話を統一
3. 非対話モード（`--yes`）でCI/CD統合
4. バックアップファイルの自動生成
5. 「修復」だけでなく「成長提案」も行う（メモリシステムの提案等）

### パターン4: イベント駆動フックシステム

**問題:** エージェントの行動に反応して追加処理を差し込みたい。

**OpenClawの解法:**
```
イベント発火 → 型ハンドラ + アクションハンドラ
             ↓
command      → command:new, command:reset, ...
session      → session:start, session:end, ...
agent        → agent:bootstrap
gateway      → gateway:*, ...
```

**実装指針:**
1. `Map<eventKey, handler[]>` のシンプルな構造
2. 型ベース (`command`) と型+アクション (`command:new`) の2段階マッチ
3. ハンドラーのエラーは他のハンドラーの実行を妨げない
4. `event.messages` 配列でハンドラーからのフィードバックを収集
5. プラグインからの登録も同じAPIで統一

### パターン5: コンパクション（選択的忘却）

**問題:** コンテキストウィンドウは有限。すべてを記憶し続けることはできない。

**OpenClawの解法:**
```
会話履歴 → トークン推定 → 閾値超過?
                              ↓ Yes
            チャンク分割 (トークン数ベース)
                ↓
            各チャンクをLLM要約
                ↓
            サマリーマージ (決定事項・TODOを保持)
                ↓
            圧縮された履歴で継続
```

**実装指針:**
1. `estimateTokens()` で正確なトークンカウントを維持
2. チャンク分割はトークン数の均等分配
3. 要約プロンプトに「保持すべき情報」を明示
4. 20%のセーフティマージンで推定誤差を吸収

### パターン6: プログレッシブ能力獲得

**問題:** エージェントが段階的に能力を獲得する仕組みが欲しい。

**OpenClawの解法:**
```
起動時:
  workspace/ のブートストラップファイル群をロード
    ↓
  bootstrap-hooks でプラグインが差し替え可能
    ↓
  スキルスキャン → システムプロンプトに動的統合
    ↓
  メモリウォーム → 過去の知識を検索可能に

実行時:
  スキルは「必要時のみ」読み込み
  プラグインが新ツールを追加可能
  フックが行動パターンを拡張
  メモリが新しい知識を蓄積
```

### パターン7: 多段フォールバック

**問題:** 外部依存（API、モデル、エンベディング）が障害を起こす。

**OpenClawの解法:**
```
プライマリプロバイダー
  ↓ 失敗
認証プロファイルローテーション (cooldown付き)
  ↓ 全滅
フォールバックプロバイダー (OpenAI → Gemini → ローカル)
  ↓ それも失敗
バッチモード無効化 (バッチ → 逐次処理)
  ↓ 最終手段
エラー通知 + 部分的動作継続
```

**実装指針:**
1. 認証プロファイルにクールダウンを設定して再試行間隔を制御
2. `batchFailureCount` とリミットで段階的にバッチを無効化
3. エンベディングキャッシュにより、プロバイダー障害時でもキャッシュ済みデータは利用可能
4. タイムアウトは操作種別（query/batch）とプロバイダー（local/remote）で個別設定

---

## 12. アーキテクチャの教訓まとめ

### 自己成長エージェントを構築するための10の原則

1. **ツールは疎結合で登録する**: ハードコードではなくレジストリパターンで。新しい能力の追加がコア変更不要になる。

2. **メモリは2層にする**: 短期メモリ（コンテキストウィンドウ）と長期メモリ（ベクトルDB）。コンパクションで短期→長期の橋渡し。

3. **フィードバックループを閉じる**: ツール結果 → LLM推論 → 次のツール呼出 → ... のサイクルを確実に機能させる。ツール結果のガード（欠損検出・合成）が重要。

4. **自己診断を組み込む**: Doctorパターンで環境の健全性を自動チェック。問題検出→提案→承認→修復のサイクル。

5. **設定は進化するものと想定する**: スキーマバリデーション + レガシーマイグレーション + ランタイムオーバーライドの3層。

6. **耐障害性を多段で確保する**: 単一障害点を作らない。認証ローテーション→プロバイダーフォールバック→モード切替→グレースフルデグレード。

7. **コンテキスト管理を戦略的に**: コンテキストプルーニング、コンパクション、選択的メモリ検索で有限なウィンドウを最大活用。

8. **イベント駆動で拡張する**: 行動のすべてをイベントとして発火し、フックで拡張可能にする。

9. **ブートストラップを柔軟に**: ワークスペースファイル + テンプレート + フックオーバーライドで、エージェントの初期状態を柔軟に定義。

10. **段階的権限付与**: ツールプロファイル（minimal→coding→messaging→full）で、エージェントの信頼レベルに応じて能力を開放。

---

## 付録A: 主要ファイル一覧

| 領域 | ファイルパス | 役割 |
|---|---|---|
| ツール定義 | `src/agents/openclaw-tools.ts` | コアツール組み立て |
| ツールポリシー | `src/agents/tool-policy.ts` | allow/deny/group制御 |
| プラグインレジストリ | `src/plugins/registry.ts` | プラグイン登録API |
| プラグインツール解決 | `src/plugins/tools.ts` | プラグインツールの動的解決 |
| メモリマネージャ | `src/memory/manager.ts` | ハイブリッド検索・インデックス |
| 内部フック | `src/hooks/internal-hooks.ts` | イベント駆動フック |
| ブートストラップフック | `src/agents/bootstrap-hooks.ts` | 起動時のファイル差し替え |
| Doctor | `src/commands/doctor.ts` | 自己診断メイン |
| オンボーディング | `src/commands/onboard.ts` | セットアップフロー |
| コンパクション | `src/agents/compaction.ts` | 会話要約 |
| モデルフォールバック | `src/agents/model-fallback.ts` | モデル障害時の代替 |
| 認証プロファイル | `src/agents/auth-profiles.ts` | 認証ローテーション |
| コンテキストガード | `src/agents/context-window-guard.ts` | ウィンドウサイズ制御 |
| システムプロンプト | `src/agents/system-prompt.ts` | 動的プロンプト構築 |
| スキル | `src/agents/skills/workspace.ts` | スキルの読込と同期 |
| メディアパース | `src/media/parse.ts` | MEDIAトークン抽出 |
| メディアストア | `src/media/store.ts` | メディアファイル管理 |
| ワークスペース | `src/agents/workspace.ts` | ブートストラップファイル管理 |
| ツール結果ガード | `src/agents/session-tool-result-guard.ts` | toolResult整合性 |
| コンテキストプルーニング | `src/agents/pi-extensions/context-pruning/tools.ts` | 不要結果の剪定 |

## 付録B: 成長サイクル全体図

```
┌─────────────────────────────────────────────────────────────────┐
│                    OpenClaw 自己成長サイクル                       │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │ ブート    │───→│ 対話     │───→│ ツール   │───→│ 結果     │  │
│  │ ストラップ │    │ (LLM)    │    │ 実行     │    │ フィード │  │
│  │           │    │          │    │          │    │ バック   │  │
│  └─────┬────┘    └────┬─────┘    └──────────┘    └────┬─────┘  │
│        │              │                               │         │
│        │              ↓                               │         │
│        │    ┌──────────────────┐                      │         │
│        │    │ メモリ検索/更新  │←─────────────────────┘         │
│        │    │ (RAG)            │                                 │
│        │    └────────┬─────────┘                                │
│        │             │                                          │
│        │             ↓                                          │
│        │    ┌──────────────────┐                                │
│        │    │ コンパクション   │                                │
│        │    │ (選択的記憶保持) │                                │
│        │    └────────┬─────────┘                                │
│        │             │                                          │
│        │             ↓                                          │
│        │    ┌──────────────────┐                                │
│        │    │ セッション永続化 │                                │
│        │    │ (transcript)     │                                │
│        │    └────────┬─────────┘                                │
│        │             │                                          │
│        ↓             ↓                                          │
│  ┌──────────────────────────────┐                               │
│  │        次回起動時             │                               │
│  │  メモリインデックスから       │                               │
│  │  過去の知識を検索可能         │                               │
│  └──────────────────────────────┘                               │
│                                                                 │
│  ─────────── 定期的メンテナンス ───────────                     │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                  │
│  │ Doctor   │───→│ 問題検出 │───→│ 自動修復 │                  │
│  │ (診断)   │    │ & 提案   │    │ & 進化   │                  │
│  └──────────┘    └──────────┘    └──────────┘                  │
│                                                                 │
│  ─────────── 能力拡張 ───────────                               │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                  │
│  │ プラグ   │    │ スキル   │    │ フック   │                  │
│  │ イン追加 │    │ インスト │    │ 登録     │                  │
│  └──────────┘    └──────────┘    └──────────┘                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

*本レポートは OpenClaw リポジトリ (commit 328b69be1 時点) のソースコード分析に基づく。*

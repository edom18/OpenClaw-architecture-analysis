# OpenClaw メモリ戦略分析レポート

## 概要サマリー

OpenClawは、WhatsApp/Discord/Telegram等のメッセージングチャネルを統合するAIエージェントゲートウェイであり、その**メモリ戦略は4層のアーキテクチャ**で構成されている。

1. **セッションストア層** - JSON5ファイルベースのセッションメタデータ管理（短期/中期）
2. **セッショントランスクリプト層** - JSONL形式の会話履歴永続化（中期）
3. **ベクトルインデックス層** - SQLite + Embeddingによるセマンティック検索メモリ（長期）
4. **コンテキストコンパクション層** - LLMを用いた会話要約による文脈圧縮（実行時）

これらの層は、エージェントIDをキーとしてスコープが分離され、ファイルウォッチャー・イベントリスナー・定期同期の3つのトリガーで同期される。特筆すべきは、**ハイブリッド検索**（ベクトル + BM25全文検索）と**バッチ埋め込み**（OpenAI/Gemini Batch API）のサポート、そしてQMD外部ツールとのフォールバック統合である。

---

## 1. セッション管理

### 1.1 セッションストア

セッションのメタデータは、エージェントごとにJSON5ファイルとして管理される。

**ファイルパス**: `~/.openclaw/agents/{agentId}/sessions/sessions.json`

```
src/config/sessions/store.ts:109  - loadSessionStore()
src/config/sessions/store.ts:257  - saveSessionStore()
src/config/sessions/store.ts:266  - updateSessionStore() (ロック付き読み書き)
src/config/sessions/paths.ts:32   - resolveDefaultSessionStorePath()
```

**セッションエントリの構造** (`src/config/sessions/types.ts:25-96`):

| フィールド | 説明 |
|---|---|
| `sessionId` | セッション固有ID (UUID) |
| `updatedAt` | 最終更新タイムスタンプ |
| `sessionFile` | トランスクリプトファイルパス |
| `compactionCount` | コンパクション実行回数 |
| `memoryFlushAt` | メモリフラッシュタイムスタンプ |
| `modelOverride` | モデル上書き設定 |
| `thinkingLevel` | 思考レベル設定 |
| `inputTokens` / `outputTokens` / `totalTokens` | 使用トークン数 |
| `skillsSnapshot` | スキルのスナップショット |
| `systemPromptReport` | システムプロンプトレポート |

### 1.2 セッションストアのキャッシュ戦略

セッションストアはインメモリキャッシュ（TTL付き）で高速化されている。

```
src/config/sessions/store.ts:28   - SESSION_STORE_CACHE (Map)
src/config/sessions/store.ts:29   - DEFAULT_SESSION_STORE_TTL_MS = 45,000 (45秒)
```

- TTLはデフォルト45秒、環境変数 `OPENCLAW_SESSION_CACHE_TTL_MS` で調整可能
- ファイルの`mtime`を監視し、変更があればキャッシュを無効化
- 書き込み時にもキャッシュを無効化（Write-invalidate戦略）
- `structuredClone()`でディープコピーを返し、外部からのミューテーションを防止

### 1.3 セッションストアのロック機構

並行書き込みを防ぐために、ファイルベースのロック機構を使用。

```
src/config/sessions/store.ts:285  - withSessionStoreLock()
```

- ロックファイル: `{storePath}.lock`
- タイムアウト: 10秒（デフォルト）
- Staleロック検出: 30秒経過で自動削除
- ポーリング間隔: 25ms

### 1.4 セッション書き込みロック（トランスクリプト用）

トランスクリプトファイルへの並行書き込みに対する別のロック機構。

```
src/agents/session-write-lock.ts:140 - acquireSessionWriteLock()
```

- `{sessionFile}.lock` ファイルを排他ロック
- PIDベースのStaleロック検出
- リエントラントロック（同一プロセス内で複数回取得可能）
- プロセス終了時の自動クリーンアップ（SIGINT/SIGTERM/exit ハンドラ）

### 1.5 セッションスコープ

セッションは以下の2つのスコープで管理される：

```
src/config/sessions/types.ts:8 - SessionScope = "per-sender" | "global"
```

- **per-sender**: 送信者ごとに独立したセッション
- **global**: 全送信者で共有するセッション

セッションキーはチャネル・チャットタイプ・送信者IDの組み合わせで生成される。

---

## 2. コンテキスト管理（会話履歴の圧縮・保持）

### 2.1 コンテキストウィンドウガード

コンテキストウィンドウのサイズを安全に管理するガードシステム。

```
src/agents/context-window-guard.ts:3  - CONTEXT_WINDOW_HARD_MIN_TOKENS = 16,000
src/agents/context-window-guard.ts:4  - CONTEXT_WINDOW_WARN_BELOW_TOKENS = 32,000
```

コンテキストウィンドウの解決順序:
1. `modelsConfig` - models.jsonのプロバイダ設定から
2. `model` - モデル自身のcontextWindow
3. `agentContextTokens` - エージェントのcontextTokens設定（上限キャップ）
4. `default` - デフォルト値

```
src/agents/context-window-guard.ts:22 - resolveContextWindowInfo()
src/agents/context-window-guard.ts:57 - evaluateContextWindowGuard()
```

### 2.2 コンパクション（会話要約・圧縮）

コンテキストウィンドウが溢れそうになった場合、LLMを使って古いメッセージを要約する仕組み。

```
src/agents/compaction.ts:7  - BASE_CHUNK_RATIO = 0.4  (40%をチャンク化対象)
src/agents/compaction.ts:8  - MIN_CHUNK_RATIO = 0.15  (最小15%)
src/agents/compaction.ts:9  - SAFETY_MARGIN = 1.2     (20%のバッファ)
```

**コンパクションの流れ**:

```
[メッセージ群] → splitMessagesByTokenShare() → [チャンク分割]
                                                        ↓
[要約生成] ← generateSummary() ← chunkMessagesByMaxTokens()
                                                        ↓
                                              summarizeInStages()
                                                        ↓
                                        [段階的要約 → マージ要約]
```

**主要関数** (`src/agents/compaction.ts`):

| 関数 | 行 | 説明 |
|---|---|---|
| `splitMessagesByTokenShare()` | 27 | トークン数に基づくメッセージ均等分割 |
| `chunkMessagesByMaxTokens()` | 68 | 最大トークン数でチャンク化 |
| `computeAdaptiveChunkRatio()` | 110 | メッセージサイズに応じた適応的チャンク比率 |
| `isOversizedForSummary()` | 135 | 単一メッセージがコンテキストの50%超なら要約不可 |
| `summarizeWithFallback()` | 176 | フォールバック付き要約（大メッセージは除外） |
| `summarizeInStages()` | 244 | 段階的要約（分割→部分要約→マージ） |
| `pruneHistoryForContextShare()` | 307 | 履歴のプルーニング（古いチャンクから削除） |

**フォールバック戦略**:
1. 全メッセージの要約を試みる
2. 失敗した場合、大きなメッセージを除外して部分要約
3. それも失敗した場合、メタ情報のみの「要約不可」メッセージ

### 2.3 コンテキストモデル情報のキャッシュ

```
src/agents/context.ts:10 - MODEL_CACHE (Map<string, number>)
```

モデルIDとコンテキストウィンドウサイズのマッピングをメモリ内キャッシュ。起動時に非同期でロードし、`lookupContextTokens()`で参照。

---

## 3. 永続メモリ（長期記憶）

### 3.1 メモリインデックスマネージャ（ビルトインバックエンド）

OpenClawの中核的な長期メモリ機構。SQLiteデータベースに埋め込みベクトルとテキストチャンクを格納し、セマンティック検索を提供する。

```
src/memory/manager.ts:108 - class MemoryIndexManager
```

**データソース**:
- `"memory"` - ワークスペース内の`MEMORY.md`と`memory/`ディレクトリ内のMarkdownファイル
- `"sessions"` - セッショントランスクリプト（JSONL）から抽出されたテキスト

**ストレージ**:
- SQLiteデータベース: `~/.openclaw/memory/{agentId}.sqlite`
- Node.js 22+の`node:sqlite`モジュール（DatabaseSync）を使用

**データベーススキーマ** (`src/memory/memory-schema.ts`):

```sql
-- メタデータ
CREATE TABLE meta (key TEXT PRIMARY KEY, value TEXT NOT NULL);

-- ファイル追跡
CREATE TABLE files (
  path TEXT PRIMARY KEY,
  source TEXT NOT NULL DEFAULT 'memory',  -- 'memory' | 'sessions'
  hash TEXT NOT NULL,
  mtime INTEGER NOT NULL,
  size INTEGER NOT NULL
);

-- テキストチャンク + 埋め込みベクトル
CREATE TABLE chunks (
  id TEXT PRIMARY KEY,
  path TEXT NOT NULL,
  source TEXT NOT NULL DEFAULT 'memory',
  start_line INTEGER NOT NULL,
  end_line INTEGER NOT NULL,
  hash TEXT NOT NULL,
  model TEXT NOT NULL,
  text TEXT NOT NULL,
  embedding TEXT NOT NULL,  -- JSON配列
  updated_at INTEGER NOT NULL
);

-- 埋め込みキャッシュ
CREATE TABLE embedding_cache (
  provider TEXT NOT NULL,
  model TEXT NOT NULL,
  provider_key TEXT NOT NULL,
  hash TEXT NOT NULL,
  embedding TEXT NOT NULL,
  dims INTEGER,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (provider, model, provider_key, hash)
);

-- FTS5全文検索（ハイブリッド検索用）
CREATE VIRTUAL TABLE chunks_fts USING fts5(text, id UNINDEXED, ...);

-- ベクトル検索（sqlite-vec拡張）
CREATE VIRTUAL TABLE chunks_vec USING vec0(
  id TEXT PRIMARY KEY,
  embedding FLOAT[{dimensions}]
);
```

### 3.2 埋め込みプロバイダ

3つの埋め込みプロバイダをサポートし、自動選択とフォールバックが可能。

```
src/memory/embeddings.ts:21 - type EmbeddingProvider
```

| プロバイダ | モデル | 説明 |
|---|---|---|
| `openai` | `text-embedding-3-small` | OpenAI Embeddings API |
| `gemini` | `gemini-embedding-001` | Google Gemini Embeddings |
| `local` | `embeddinggemma-300M-Q8_0.gguf` | node-llama-cppによるローカルモデル |
| `auto` | - | OpenAI → Gemini → ローカルの順で自動選択 |

フォールバック設定:
```
src/agents/memory-search.ts:160 - fallback: "openai" | "gemini" | "local" | "none"
```

### 3.3 ハイブリッド検索

ベクトル検索とBM25全文検索を組み合わせたハイブリッド検索。

```
src/memory/hybrid.ts:41 - mergeHybridResults()
```

デフォルト設定 (`src/agents/memory-search.ts`):
- ベクトル重み: 0.7
- テキスト重み: 0.3
- 候補倍率: 4x
- 最大結果数: 6
- 最小スコア: 0.35

### 3.4 チャンキング

Markdownファイルをトークン単位でチャンクに分割。

```
src/agents/memory-search.ts:75 - DEFAULT_CHUNK_TOKENS = 400
src/agents/memory-search.ts:76 - DEFAULT_CHUNK_OVERLAP = 80
```

### 3.5 バッチ埋め込み

OpenAIとGeminiのBatch APIを使った非同期バッチ埋め込みをサポート。

```
src/memory/manager.ts:86  - SESSION_DIRTY_DEBOUNCE_MS = 5,000
src/memory/manager.ts:87  - EMBEDDING_BATCH_MAX_TOKENS = 8,000
src/memory/manager.ts:89  - EMBEDDING_INDEX_CONCURRENCY = 4
src/memory/manager.ts:90  - EMBEDDING_RETRY_MAX_ATTEMPTS = 3
src/memory/manager.ts:93  - BATCH_FAILURE_LIMIT = 2 (失敗2回でバッチ無効化)
```

### 3.6 同期トリガー

メモリインデックスは3つのトリガーで同期される:

| トリガー | デフォルト | 説明 |
|---|---|---|
| **ファイルウォッチャー** | 有効 (1.5秒デバウンス) | `MEMORY.md`/`memory/`の変更を監視 |
| **セッションイベント** | 有効 (5秒デバウンス) | トランスクリプト更新イベントを監視 |
| **定期同期** | 無効 (0分) | 設定可能なインターバル |
| **検索時同期** | 有効 | ダーティフラグがある場合に検索前同期 |
| **セッション開始時** | 有効 | 新セッション開始時に同期 |

セッション差分同期の閾値:
```
src/agents/memory-search.ts:78 - DEFAULT_SESSION_DELTA_BYTES = 100,000
src/agents/memory-search.ts:79 - DEFAULT_SESSION_DELTA_MESSAGES = 50
```

### 3.7 インデックスキャッシュ

`MemoryIndexManager`インスタンスは`INDEX_CACHE` Mapでキャッシュされ、同一パラメータでの重複生成を防止。

```
src/memory/manager.ts:103 - const INDEX_CACHE = new Map<string, MemoryIndexManager>();
```

キャッシュキー: `{agentId}:{workspaceDir}:{JSON(settings)}`

### 3.8 埋め込みキャッシュ

テキストチャンクの埋め込みベクトルをSQLiteテーブルにキャッシュし、再計算を回避。

```
src/memory/manager.ts:85 - EMBEDDING_CACHE_TABLE = "embedding_cache"
```

- プロバイダ・モデル・プロバイダキー・テキストハッシュの組み合わせでユニークキー
- LRU的にプルーニング（`updated_at` ASC順で古いものから削除）
- リインデックス時にはキャッシュをseedとして新DBにコピー

### 3.9 アトミック・リインデックス

モデルやプロバイダの変更時には安全なリインデックスを実行。

```
src/memory/manager.ts:1398 - runSafeReindex()
```

1. 一時DBファイル（`{dbPath}.tmp-{uuid}`）を作成
2. 既存の埋め込みキャッシュを一時DBにseed
3. 全ファイルを一時DBにインデックス
4. 成功後、元のDBファイルとアトミックにswap
5. 失敗時は一時DBを削除して元のDBを復元

---

## 4. QMDバックエンド（外部メモリマネージャ）

ビルトインのSQLite埋め込みインデックスに加え、外部の`qmd`コマンドラインツールとの統合も提供。

```
src/memory/qmd-manager.ts:51 - class QmdMemoryManager
```

**特徴**:
- QMDの`XDG_CONFIG_HOME`/`XDG_CACHE_HOME`を個別に管理
- コレクション単位でドキュメントを管理
- セッションのMarkdownエクスポート機能
- `qmd query --json`でセマンティック検索
- ビルトインバックエンドへの自動フォールバック

```
src/memory/search-manager.ts:67 - class FallbackMemoryManager
```

フォールバック戦略:
1. QMDマネージャを試行
2. 失敗した場合、ビルトインの`MemoryIndexManager`にフォールバック
3. ステータスにフォールバック情報を含める

---

## 5. 設定・資格情報の管理方法

### 5.1 状態ディレクトリ

```
src/config/paths.ts:49 - resolveStateDir()
```

- デフォルト: `~/.openclaw/`
- 環境変数 `OPENCLAW_STATE_DIR` で上書き可能
- レガシーディレクトリ（`.clawdbot`, `.moltbot`, `.moldbot`）からの自動マイグレーション

**ディレクトリ構造**:
```
~/.openclaw/
  |-- openclaw.json           # メイン設定ファイル
  |-- credentials/            # OAuth資格情報
  |   |-- oauth.json
  |-- identity/               # デバイス認証
  |   |-- device-auth.json
  |-- agents/
  |   |-- {agentId}/
  |       |-- sessions/
  |       |   |-- sessions.json       # セッションストア
  |       |   |-- {sessionId}.jsonl   # トランスクリプト
  |       |-- qmd/                    # QMD状態
  |           |-- xdg-config/
  |           |-- xdg-cache/
  |           |-- sessions/           # エクスポートされたセッション
  |-- memory/
      |-- {agentId}.sqlite            # メモリインデックスDB
```

### 5.2 設定ファイル

```
src/config/paths.ts:95 - resolveCanonicalConfigPath()
```

- JSON5形式（`openclaw.json`）
- レガシーファイル名のフォールバック
- 環境変数 `OPENCLAW_CONFIG_PATH` で上書き可能

### 5.3 デバイス認証ストア

```
src/infra/device-auth-store.ts:18 - DEVICE_AUTH_FILE = "device-auth.json"
```

- トークン・ロール・スコープのJSONストア
- `~/.openclaw/identity/device-auth.json`

### 5.4 OAuth資格情報

```
src/config/paths.ts:218 - resolveOAuthDir()
```

- `~/.openclaw/credentials/oauth.json`
- 環境変数 `OPENCLAW_OAUTH_DIR` で上書き可能

### 5.5 メモリ設定タイプ

```
src/config/types.memory.ts:6 - type MemoryConfig
```

メモリバックエンドは2種類:
- `"builtin"` - SQLite + 埋め込みベクトル（デフォルト）
- `"qmd"` - 外部QMDツール

---

## 6. メモリ関連の設計パターン

### 6.1 パターン一覧

| パターン | 適用箇所 | 説明 |
|---|---|---|
| **Singleton/Cache** | `INDEX_CACHE`, `SESSION_STORE_CACHE`, `QMD_MANAGER_CACHE` | 同一パラメータのインスタンス再利用 |
| **Write-Invalidate Cache** | セッションストア | 書き込み時にキャッシュ無効化 |
| **Debounced Sync** | ファイルウォッチャー、セッションデルタ | デバウンスで頻繁な更新を抑制 |
| **Fallback Chain** | 埋め込みプロバイダ、QMD→ビルトイン | 失敗時の自動フォールバック |
| **Atomic Swap** | リインデックス | 一時ファイルでの安全なDB置換 |
| **File-Based Lock** | セッションストア、トランスクリプト | ロックファイルによる排他制御 |
| **Progressive Summarization** | コンパクション | 段階的要約→マージ |
| **Delta Tracking** | セッションファイル監視 | バイト/メッセージ数の差分追跡 |
| **Batch with Failure Circuit Breaker** | バッチ埋め込み | 2回失敗でバッチモード無効化 |
| **Provider Key Hashing** | 埋め込みキャッシュ | プロバイダ設定のハッシュでキャッシュキー生成 |

### 6.2 メモリの階層構造

```
+--------------------------------------------------------------+
|                    実行時メモリ層（揮発性）                     |
|  +-----------------------+  +-----------------------------+  |
|  | INDEX_CACHE (Map)     |  | SESSION_STORE_CACHE (Map)   |  |
|  | MODEL_CACHE (Map)     |  | QMD_MANAGER_CACHE (Map)     |  |
|  +-----------------------+  +-----------------------------+  |
+--------------------------------------------------------------+
                              |
                              v
+--------------------------------------------------------------+
|                   セッションストア層（短期〜中期）               |
|  sessions.json - メタデータ（TTL 45秒キャッシュ）              |
|  {sessionId}.jsonl - 会話トランスクリプト                      |
+--------------------------------------------------------------+
                              |
                              v
+--------------------------------------------------------------+
|                 コンパクション層（実行時・揮発性）               |
|  summarizeInStages() → pruneHistoryForContextShare()          |
|  ・段階的要約                                                  |
|  ・適応的チャンク比率                                          |
|  ・オーバーサイズメッセージのフォールバック                      |
+--------------------------------------------------------------+
                              |
                              v
+--------------------------------------------------------------+
|               ベクトルインデックス層（長期・永続）               |
|  +---------------------------+  +------------------------+   |
|  | ビルトイン (SQLite)        |  | QMD (外部ツール)       |   |
|  | ・ベクトル検索 (vec0)      |  | ・コレクション管理      |   |
|  | ・FTS5全文検索             |  | ・セッションエクスポート |   |
|  | ・埋め込みキャッシュ        |  | ・XDGベースディレクトリ  |   |
|  | ・ハイブリッド検索          |  +------------------------+   |
|  +---------------------------+                                |
+--------------------------------------------------------------+
                              |
                              v
+--------------------------------------------------------------+
|                   ファイルシステム層（永続）                     |
|  ~/.openclaw/                                                 |
|    agents/{id}/sessions/  - セッションファイル                  |
|    memory/{id}.sqlite     - メモリインデックスDB                |
|    openclaw.json          - メイン設定                         |
|    credentials/           - 認証情報                           |
|    identity/              - デバイス認証                       |
+--------------------------------------------------------------+
```

---

## 7. アーキテクチャ図（データフロー）

```
                    ユーザーメッセージ
                          |
                          v
                  +----------------+
                  | ルーティング層  |
                  | (session-key)  |
                  +-------+--------+
                          |
              +-----------+-----------+
              v                       v
     +----------------+      +------------------+
     | セッション      |      | メモリ検索        |
     | ストア          |      | (search)          |
     | sessions.json   |      +--------+---------+
     +-------+--------+               |
              |                +-------+-------+
              v                v               v
     +----------------+  +---------+  +------------+
     | トランスクリプト|  | ベクトル |  | FTS5       |
     | {id}.jsonl     |  | 検索    |  | キーワード  |
     +-------+--------+  +----+----+  | 検索       |
              |                |       +-----+------+
              |                +------+------+
              |                       |
              v                       v
     +----------------+      +-----------------+
     | コンパクション  |      | ハイブリッド     |
     | (要約生成)     |      | 結果マージ      |
     +----------------+      +-----------------+
              |
              v
     +----------------+       +---------------------+
     | 埋め込み生成    |  <--- | 埋め込みキャッシュ   |
     | (OpenAI/       |       | (embedding_cache)    |
     |  Gemini/Local) |       +---------------------+
     +-------+--------+
              |
              v
     +-------------------+
     | SQLite DB          |
     | (chunks, files,    |
     |  chunks_vec,       |
     |  chunks_fts)       |
     +-------------------+

     同期トリガー:
     +-----------+  +----------+  +----------+
     | chokidar  |  | イベント  |  | 定期     |
     | ウォッチャ |  | リスナー  |  | タイマー  |
     +-----------+  +----------+  +----------+
         |              |              |
         v              v              v
     +------------------------------------+
     | sync() → runSync() / runSafeReindex|
     +------------------------------------+
```

---

## 8. 「自己成長するエージェント」開発への示唆

OpenClawのメモリ戦略は、自己成長するエージェントの設計に対して以下の重要な示唆を提供する。

### 8.1 メモリの階層化は必須

- **揮発性キャッシュ**（Map/TTL）→ **セッション永続化**（JSON/JSONL）→ **セマンティックインデックス**（SQLite + ベクトル）の3層構造が実用的
- 各層は異なるアクセスパターンとライフタイムを持つ

### 8.2 コンパクションは段階的に

- 全メッセージの一括要約は失敗しやすい（大きなメッセージ、トークン制限）
- **段階的要約**（分割→部分要約→マージ）と**フォールバック**（大メッセージ除外→メタ情報のみ）の組み合わせが堅牢
- 適応的なチャンク比率（メッセージの平均サイズに基づく）が実用的

### 8.3 セマンティック検索の実装パターン

- **ハイブリッド検索**（ベクトル70% + BM25 30%）が精度向上に効果的
- **埋め込みキャッシュ**（テキストハッシュベース）で再計算コストを大幅削減
- **バッチ埋め込み** + **Circuit Breaker**（2回失敗で個別APIにフォールバック）が信頼性確保に有効
- **アトミックリインデックス**で安全なモデル切り替え

### 8.4 同期戦略のベストプラクティス

- **ファイルウォッチャー**（chokidar）でメモリファイルの変更を即座に検出
- **イベント駆動**のセッション差分追跡（バイト数/メッセージ数の閾値ベース）
- **デバウンス**で頻繁な更新を抑制（1.5秒〜5秒）
- **検索時同期**でダーティデータを自動更新

### 8.5 マルチプロバイダ戦略

- 埋め込みプロバイダの自動選択（`auto`）とフォールバックチェーン
- QMD外部ツールとの統合 + ビルトインバックエンドへのフォールバック
- プロバイダキーのハッシュ化でキャッシュの一貫性を保証

### 8.6 エージェントIDによるスコープ分離

- すべてのメモリ・セッション・設定がエージェントIDでスコープ分離
- マルチエージェント環境での衝突を回避
- セッションキーによるメモリアクセスのスコープ制御（QMDの`scope`設定）

### 8.7 自律的な知識蓄積のために

OpenClawの`MEMORY.md`と`memory/`ディレクトリのパターンは、エージェントが自律的に知識を蓄積するための基盤として優れている:

- **MEMORY.md**: エージェントが自身で更新できる構造化された知識ファイル
- **memory/**: トピック別の知識ファイル群（セマンティック検索でアクセス）
- **セッショントランスクリプト**: 過去の会話履歴からの検索（実験的機能）
- これらをベクトルインデックスで横断検索し、コンテキストに注入するパイプラインが確立されている

---

## 付録: 主要ファイルマップ

| ファイル | 主な役割 |
|---|---|
| `src/memory/manager.ts` | MemoryIndexManager - ビルトインメモリインデックス |
| `src/memory/types.ts` | メモリ検索の型定義 |
| `src/memory/memory-schema.ts` | SQLiteスキーマ定義 |
| `src/memory/embeddings.ts` | 埋め込みプロバイダ抽象化 |
| `src/memory/hybrid.ts` | ハイブリッド検索（ベクトル + BM25） |
| `src/memory/internal.ts` | メモリファイル列挙・チャンキング |
| `src/memory/search-manager.ts` | メモリ検索マネージャ（QMD/ビルトインフォールバック） |
| `src/memory/qmd-manager.ts` | QMD外部ツール統合 |
| `src/memory/sqlite-vec.ts` | sqlite-vecベクトル拡張 |
| `src/agents/compaction.ts` | コンテキスト圧縮（要約生成） |
| `src/agents/context-window-guard.ts` | コンテキストウィンドウサイズ管理 |
| `src/agents/context.ts` | モデルコンテキスト情報キャッシュ |
| `src/agents/memory-search.ts` | メモリ検索設定の解決 |
| `src/agents/cli-session.ts` | CLIセッションID管理 |
| `src/agents/session-write-lock.ts` | セッションファイルロック |
| `src/config/sessions/store.ts` | セッションストア読み書き |
| `src/config/sessions/types.ts` | セッションエントリ型定義 |
| `src/config/sessions/paths.ts` | セッション関連パス解決 |
| `src/config/sessions/transcript.ts` | トランスクリプト書き込み |
| `src/config/paths.ts` | 状態ディレクトリ・設定パス解決 |
| `src/config/types.memory.ts` | メモリ設定の型定義 |
| `src/infra/device-auth-store.ts` | デバイス認証ストア |
| `src/infra/session-cost-usage.ts` | セッションコスト・使用量追跡 |

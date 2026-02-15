# OpenClaw 起動シーケンス

このドキュメントでは、CLIのエントリーポイントからパーソナルAIアシスタントが初期化されるまでのプロセスについて概説します。

## 1. CLI エントリーポイント (`src/entry.ts`)
すべての実行はここから始まります。
- **環境変数の正規化**: `normalizeEnv()` および `installProcessWarningFilter()` を呼び出します。
- **Windows引数の正規化**: Windowsで実行されている場合、パスや引用符の問題を処理するために `process.argv` を調整します。
- **プロセスの再起動 (Respawn)**: 実験的な警告（Experimental Warnings）が抑制されていることを確認し、必要に応じてプロセスを再起動します。
- **動的インポート**: `./cli/run-main.js` から `runCli` を読み込んで実行します。

## 2. CLI メインループ (`src/cli/run-main.ts`)
- **初期化**: `.env` を読み込み、サポートされている Node.js 実行環境であることを確認します。
- **ルーティング**: `tryRouteCli` を介してコマンドのルーティングを試みます。
- **プログラムの構築**: `src/cli/program.ts` の `buildProgram()` を呼び出します。
- **コマンドの登録**: setup, onboard, agent, message, プラグインなどの各種コマンドを登録します。
- **実行**: 引数を解析し、一致したコマンドのアクションを実行します。

## 3. Agent コマンドアクション (`src/commands/agent.ts`)
ユーザーが `openclaw agent ...` を実行した際：
- **設定の読み込み**: グローバル設定を読み込み、エージェントのワークスペースとディレクトリを特定します。
- **セッションの解決**: `--to`, `--session-id`, または `--agent` を基にセッションを特定または作成します。
- **モデルの選択**: 使用するモデルとプロバイダーを決定します（フォールバックロジックを含む）。
- **ランナーの実行**: 通常のローカル実行の場合、`src/agents/pi-embedded-runner.ts` の `runEmbeddedPiAgent` を呼び出します。

## 4. 組み込みランナー (`src/agents/pi-embedded-runner/run.ts`)
- **リソースの準備**: ワークスペースが存在することを確認し、スキルスナップショットを読み込みます。
- **認証プロファイルの選択**: 失敗した場合に備えて、利用可能なAPIキー/プロファイルをローテーションします。
- **ターンループ**: 「実行の試行（Run Attempt）」を実行し、LLMの呼び出しと結果の処理を行います。
- **サブスクリプション**: `subscribeEmbeddedPiSession` を使用して、ストリーミングレスポンス、ツール呼び出し、および複数のターンを処理します。

## 5. 試行ロジック (`src/agents/pi-embedded-runner/run/attempt.ts`)
- **システムプロンプトの生成**: `buildEmbeddedSystemPrompt` を呼び出します（詳細は [system_prompt.md](./system_prompt.md) を参照）。
- **セッション管理**: セッションファイルを開き、会話履歴（Transcript）を準備します。
- **LLM 呼び出し**: 蓄積されたコンテキストと新しいプロンプトを使用して、特定のモデルプロバイダー（Anthropic, OpenAI, Geminiなど）を呼び出します。

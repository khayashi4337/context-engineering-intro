## FEATURE:

- Pydantic AIエージェントが、別のPydantic AIエージェントをツールとして持つ構成
- メインエージェントはリサーチエージェント、サブエージェントはメール下書きエージェント
- エージェントと対話するためのCLI（コマンドラインインターフェース）
- メール下書きエージェントにはGmail、リサーチエージェントにはBrave APIを利用

## EXAMPLES:

`examples/` フォルダ内にREADMEがあり、例の概要や、上記機能用READMEの構成例を理解できます。

- `examples/cli.py` … CLI作成のテンプレートとして利用
- `examples/agent/` … 複数プロバイダやLLMに対応したPydantic AIエージェントのベストプラクティス、依存関係管理、ツール追加方法などを把握するために全ファイルを参照

※これらの例をそのままコピーしないでください（別プロジェクト用です）。あくまでインスピレーションやベストプラクティスの参考としてください。

## DOCUMENTATION:

Pydantic AI ドキュメント: https://ai.pydantic.dev/

## OTHER CONSIDERATIONS:

- .env.exampleとセットアップ手順（GmailやBraveの設定方法を含む）をREADMEに記載すること
- READMEにプロジェクト構成も記載すること
- 必要な依存パッケージはすでに仮想環境にセットアップ済み
- 環境変数の管理にはpython_dotenvとload_env()を使用すること

name: "マルチエージェントシステム：リサーチエージェント＋メール下位エージェント"
description: |

## 目的（Purpose）
Brave Search APIを利用する主エージェント（Research Agent）が、Gmail APIを使うメール下位エージェント（Email Draft Agent）をツールとして呼び出す、Pydantic AIのマルチエージェントシステムを構築します。外部API連携を含む「エージェントをツール化する」パターンの実例です。

## 基本原則（Core Principles）
1. **コンテキスト最優先**：必要なドキュメント・例・注意点をすべて含める
2. **バリデーションループ**：AIが実行・修正できるテストやリントを必ず用意
3. **情報密度重視**：コードベースのキーワードやパターンを活用
4. **段階的な成功**：まずシンプルに作り、検証し、徐々に強化

---

## ゴール（Goal）
CLI経由でユーザーがリサーチ依頼を出し、Research Agentがメール作成タスクをEmail Draft Agentに委譲できる、実運用レベルのマルチエージェントシステムを作成します。複数LLMプロバイダ対応・API認証の安全管理も必須です。

## なぜ（Why）
- **ビジネス価値**：リサーチとメール下書き業務を自動化
- **統合性**：高度なPydantic AIマルチエージェントパターンを実証
- **解決する課題**：リサーチベースのメール作成業務の手間削減

## 何を（What）
CLIベースのアプリケーション：
- ユーザーがリサーチクエリを入力
- Research AgentがBrave APIで検索
- Research AgentがEmail Draft Agentを呼び出しGmail下書きを作成
- 結果がリアルタイムでユーザーにストリーミング返却

### 成功基準（Success Criteria）
- [ ] Research AgentがBrave APIで正常に検索できる
- [ ] Email Agentが認証付きでGmail下書きを作成できる
- [ ] Research AgentがEmail Agentをツールとして呼び出せる
- [ ] CLIがツール利用状況を含めてストリーミング応答できる
- [ ] すべてのテストが通り、コード品質基準を満たす

## 必要なすべてのコンテキスト

### ドキュメント・参考資料（Documentation & References）
```yaml
# 必読 - 実装に必要な文脈をすべてここにリストアップ
- url: https://ai.pydantic.dev/agents/
  why: エージェント作成パターン
  
- url: https://ai.pydantic.dev/multi-agent-applications/
  why: マルチエージェントシステムパターン（agent-as-tool含む）
  
- url: https://developers.google.com/gmail/api/guides/sending
  why: Gmail API認証と下書き作成
  
- url: https://api-dashboard.search.brave.com/app/documentation
  why: Brave Search APIのRESTエンドポイント
  
- file: examples/agent/agent.py
  why: エージェント作成・ツール登録・依存性注入のパターン
  
- file: examples/agent/providers.py
  why: 複数プロバイダLLM設定パターン
  
- file: examples/cli.py
  why: ストリーミング応答・ツール可視化つきCLI構造

- url: https://github.com/googleworkspace/python-samples/blob/main/gmail/snippet/send%20mail/create_draft.py
  why: Gmail下書き作成の公式例
```

### 現在のコードベース構成
```bash
.
├── examples/
│   ├── agent/
│   │   ├── agent.py
│   │   ├── providers.py
│   │   └── ...
│   └── cli.py
├── PRPs/
│   └── templates/
│       └── prp_base.md
├── INITIAL.md
├── CLAUDE.md
└── requirements.txt
```

### 追加後の理想的なコードベース構成
```bash
.
├── agents/
│   ├── __init__.py               # パッケージ初期化
│   ├── research_agent.py         # Brave Search対応主エージェント
│   ├── email_agent.py           # Gmail対応下位エージェント
│   ├── providers.py             # LLMプロバイダ設定
│   └── models.py                # Pydanticデータバリデーションモデル
├── tools/
│   ├── __init__.py              # パッケージ初期化
│   ├── brave_search.py          # Brave Search API連携
│   └── gmail_tool.py            # Gmail API連携
├── config/
│   ├── __init__.py              # パッケージ初期化
│   └── settings.py              # 環境・設定管理
├── tests/
│   ├── __init__.py              # パッケージ初期化
│   ├── test_research_agent.py   # Research agentテスト
│   ├── test_email_agent.py      # Email agentテスト
│   ├── test_brave_search.py     # Brave searchツールテスト
│   ├── test_gmail_tool.py       # Gmailツールテスト
│   └── test_cli.py              # CLIテスト
├── cli.py                       # CLIインターフェース
├── .env.example                 # 環境変数テンプレート
├── requirements.txt             # 依存パッケージ
├── README.md                    # ドキュメント
└── credentials/.gitkeep         # Gmail認証情報用ディレクトリ
```

### 既知の注意点・ライブラリの癖
```python
# 重要: Pydantic AIは全てasync必須（同期関数NG）
# 重要: Gmail APIは初回OAuth2認証が必要（credentials.json必須）
# 重要: Brave APIは無料枠で月2000リクエストまで
# 重要: agent-as-toolパターンはctx.usageでトークン管理必須
# 重要: Gmail下書きはbase64＋正しいMIME形式で作成
# 重要: 絶対インポート推奨
# 重要: 機密情報は.env管理、コミット禁止
```

## 実装設計（Implementation Blueprint）

### データモデル・構造

```python
# models.py - コアデータ構造
from pydantic import BaseModel, Field
from typing import List, Optional
from datetime import datetime

class ResearchQuery(BaseModel):
    query: str = Field(..., description="調査トピック")
    max_results: int = Field(10, ge=1, le=50)
    include_summary: bool = Field(True)

class BraveSearchResult(BaseModel):
    title: str
    url: str
    description: str
    score: float = Field(0.0, ge=0.0, le=1.0)

class EmailDraft(BaseModel):
    to: List[str] = Field(..., min_items=1)
    subject: str = Field(..., min_length=1)
    body: str = Field(..., min_length=1)
    cc: Optional[List[str]] = None
    bcc: Optional[List[str]] = None

class ResearchEmailRequest(BaseModel):
    research_query: str
    email_context: str = Field(..., description="メール生成のための文脈")
    recipient_email: str
```

### タスク一覧

```yaml
Task 1: 設定・環境準備
CREATE config/settings.py:
  - PATTERN: pydantic-settingsやos.getenv利用例を踏襲
  - 環境変数をデフォルト付きでロード
  - 必須APIキーのバリデーション

CREATE .env.example:
  - 必要な環境変数をすべて説明付きで記載
  - examples/README.mdのパターンを踏襲

Task 2: Brave Searchツール実装
CREATE tools/brave_search.py:
  - PATTERN: examples/agent/tools.pyのasync関数パターン
  - httpx利用のシンプルなRESTクライアント
  - レート制限・エラーを適切に処理
  - BraveSearchResultモデルで返却

Task 3: Gmailツール実装
CREATE tools/gmail_tool.py:
  - PATTERN: GmailクイックスタートのOAuth2フローを踏襲
  - credentials/ディレクトリにtoken.json保存
  - 正しいMIMEエンコードで下書き作成
  - 認証リフレッシュも自動対応

Task 4: Email Draft Agent作成
CREATE agents/email_agent.py:
  - PATTERN: examples/agent/agent.py構造を踏襲
  - deps_typeパターンでAgent作成
  - gmail_toolを@agent.toolで登録
  - EmailDraftモデルで返却

Task 5: Research Agent作成
CREATE agents/research_agent.py:
  - PATTERN: Pydantic AI docsのマルチエージェントパターン
  - brave_searchをツールとして登録
  - email_agent.run()もツール登録
  - 依存性注入にRunContext利用

Task 6: CLIインターフェース実装
CREATE cli.py:
  - PATTERN: examples/cli.pyのストリーミング出力パターン
  - ツール可視化・カラー出力
  - asyncio.run()でasync対応
  - セッション管理で会話文脈保持

Task 7: テスト追加
CREATE tests/:
  - PATTERN: examples配下のテスト構造を踏襲
  - 外部APIはモック化
  - ハッピーパス・エッジ・エラー全網羅
  - カバレッジ80%以上

Task 8: ドキュメント作成
CREATE README.md:
  - PATTERN: examples/README.md構造を踏襲
  - セットアップ・インストール・使い方
  - APIキー設定手順
  - アーキテクチャ図
```

### 各タスクごとの擬似コード

```python
# Task 2: Brave Searchツール
async def search_brave(query: str, api_key: str, count: int = 10) -> List[BraveSearchResult]:
    # PATTERN: httpx利用（examplesはaiohttp）
    async with httpx.AsyncClient() as client:
        headers = {"X-Subscription-Token": api_key}
        params = {"q": query, "count": count}
        
        # 注意: APIキー無効時は401
        response = await client.get(
            "https://api.search.brave.com/res/v1/web/search",
            headers=headers,
            params=params,
            timeout=30.0  # 重要: タイムアウト必須
        )
        
        # エラー処理パターン
        if response.status_code != 200:
            raise BraveAPIError(f"API returned {response.status_code}")
        
        # Pydanticでバリデーション
        data = response.json()
        return [BraveSearchResult(**result) for result in data.get("web", {}).get("results", [])]

# Task 5: Research AgentからEmail Agentツール呼び出し
@research_agent.tool
async def create_email_draft(
    ctx: RunContext[AgentDependencies],
    recipient: str,
    subject: str,
    context: str
) -> str:
    """リサーチ文脈からメール下書きを作成"""
    # 重要: トークン管理のためusageを渡す
    result = await email_agent.run(
        f"Create an email to {recipient} about: {context}",
        deps=EmailAgentDeps(subject=subject),
        usage=ctx.usage  # multi-agent docsのパターン
    )
    
    return f"Draft created with ID: {result.data}"
```

### 統合ポイント
```yaml
ENVIRONMENT:
  - .envに追加
  - 変数例:
      # LLM設定
      LLM_PROVIDER=openai
      LLM_API_KEY=sk-...
      LLM_MODEL=gpt-4
      
      # Brave Search
      BRAVE_API_KEY=BSA...
      
      # Gmail（credentials.jsonのパス）
      GMAIL_CREDENTIALS_PATH=./credentials/credentials.json
      
CONFIG:
  - Gmail OAuth: 初回はブラウザで認証
  - トークン保存: ./credentials/token.json（自動生成）
  
DEPENDENCIES:
  - requirements.txtに以下を追加:
    - google-api-python-client
    - google-auth-httplib2
    - google-auth-oauthlib
```

## バリデーションループ（Validation Loop）

### レベル1：構文・スタイル
```bash
# まずこれらを実行し、エラーがあれば修正してから進める
ruff check . --fix              # スタイル自動修正
mypy .                          # 型チェック

# 期待値: エラーなし。エラーが出たら必ず修正。
```

### レベル2：ユニットテスト
```python
# test_research_agent.py
async def test_research_with_brave():
    """Research Agentが正しく検索できるか"""
    agent = create_research_agent()
    result = await agent.run("AI safety research")
    assert result.data
    assert len(result.data) > 0

async def test_research_creates_email():
    """Research AgentがEmail Agentを呼べるか"""
    agent = create_research_agent()
    result = await agent.run(
        "Research AI safety and draft email to john@example.com"
    )
    assert "draft_id" in result.data

# test_email_agent.py  
def test_gmail_authentication(monkeypatch):
    """Gmail OAuthフローのテスト"""
    monkeypatch.setenv("GMAIL_CREDENTIALS_PATH", "test_creds.json")
    tool = GmailTool()
    assert tool.service is not None

async def test_create_draft():
    """正しいエンコードで下書き作成できるか"""
    agent = create_email_agent()
    result = await agent.run(
        "Create email to test@example.com about AI research"
    )
    assert result.data.get("draft_id")
```

```bash
# テストが通るまで繰り返し実行：
pytest tests/ -v --cov=agents --cov=tools --cov-report=term-missing

# 失敗時: 個別テストをデバッグ・修正→再実行
```

### レベル3：統合テスト
```bash
# CLI対話テスト
python cli.py

# 期待される対話例：
# You: Research latest AI safety developments
# 🤖 Assistant: [リサーチ結果をストリーム]
# 🛠 使用ツール:
#   1. brave_search (query='AI safety developments', limit=10)
#
# You: Create an email draft about this to john@example.com  
# 🤖 Assistant: [下書き作成]
# 🛠 使用ツール:
#   1. create_email_draft (recipient='john@example.com', ...)

# Gmailの下書きフォルダを確認
```

## 最終バリデーションチェックリスト
- [ ] すべてのテストが通る: `pytest tests/ -v`
- [ ] リントエラーなし: `ruff check .`
- [ ] 型エラーなし: `mypy .`
- [ ] Gmail OAuthフローが動作（ブラウザ認証・トークン保存）
- [ ] Brave Searchで検索できる
- [ ] Research AgentがEmail Agentを呼び出せる
- [ ] CLIがツール可視化つきでストリーミング応答
- [ ] エラーケースも適切に処理
- [ ] READMEにセットアップ手順が明記
- [ ] .env.exampleに全必要変数あり

---

## NGパターン・アンチパターン
- ❌ APIキーのハードコーディング禁止（必ず環境変数）
- ❌ asyncエージェント文脈で同期関数禁止
- ❌ Gmail OAuthフローのセットアップ省略禁止
- ❌ APIレート制限無視禁止
- ❌ multi-agent呼び出しでctx.usage渡し忘れ禁止
- ❌ credentials.jsonやtoken.jsonのコミット禁止

## 信頼度スコア: 9/10

- コードベースに明確な例がある
- 外部APIがよくドキュメント化されている
- マルチエージェントパターンが確立されている
- バリデーションゲートが網羅的

Gmail OAuth初回UXに若干不安はあるが、ドキュメントが明快なので十分対応可能。
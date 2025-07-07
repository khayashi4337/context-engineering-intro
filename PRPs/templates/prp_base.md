name: "ベースPRPテンプレートv2 - コンテキスト重視＆バリデーションループ付き"
description: |

## 目的（Purpose）
AIエージェントが十分なコンテキストと自己検証機能を持って、反復的な改善を通じて確実に動作するコードを実装できるよう最適化されたテンプレートです。

## 基本原則（Core Principles）
1. **コンテキスト最優先**：必要なドキュメント・例・注意点をすべて含める
2. **バリデーションループ**：AIが実行・修正できるテストやリントを必ず用意
3. **情報密度重視**：コードベースのキーワードやパターンを活用
4. **段階的な成功**：まずシンプルに作り、検証し、徐々に強化
5. **グローバルルール遵守**：CLAUDE.mdの全ルールを必ず守る

---

## ゴール（Goal）
[何を作るか ― 最終的な状態や希望を具体的に記述]

## なぜ（Why）
- [ビジネス価値・ユーザーへの影響]
- [既存機能との統合]
- [どんな課題を誰のために解決するか]

## 何を（What）
[ユーザーに見える動作・技術要件]

### 成功基準（Success Criteria）
- [ ] [具体的かつ測定可能な成果]

## 必要なすべてのコンテキスト（All Needed Context）

### ドキュメント・参考資料（Documentation & References）
```yaml
# 必読 - 実装に必要な文脈をすべてここにリストアップ
- url: [公式APIドキュメントURL]
  why: [必要なセクションやメソッド]
  
- file: [パターン参照用のexample.pyなど]
  why: [模倣すべきパターン・注意点]
  
- doc: [ライブラリのドキュメントURL] 
  section: [落とし穴に関する具体的なセクション]
  critical: [よくあるエラーを防ぐ重要ポイント]

- docfile: [PRPs/ai_docs/file.md]
  why: [ユーザーがプロジェクトに貼り付けたドキュメント]

```

### 現在のコードベース構成（`tree`コマンドで取得）
```bash

```

### 追加後の理想的なコードベース構成（追加ファイルと担当内容）
```bash

```

### 既知の注意点・ライブラリの癖
```python
# 重要: [ライブラリ名] には [特定のセットアップ] が必要
# 例: FastAPIのエンドポイントはasync関数でなければならない
# 例: このORMは1000件超のバルクインサート非対応
# 例: pydantic v2を使用 など
```

## 実装設計（Implementation Blueprint）

### データモデル・構造

コアとなるデータモデルを作成し、型安全性と一貫性を担保する。
```python
例：
 - ORMモデル
 - pydanticモデル
 - pydanticスキーマ
 - pydanticバリデータ

```

### PRPを満たすために必要なタスク一覧（実施順）

```yaml
Task 1:
MODIFY src/existing_module.py:
  - FIND pattern: "class OldImplementation"
  - INJECT after line containing "def __init__"
  - PRESERVE existing method signatures

CREATE src/new_feature.py:
  - MIRROR pattern from: src/similar_feature.py
  - MODIFY class name and core logic
  - KEEP error handling pattern identical

...(...)

Task N:
...

```


### 各タスクごとの擬似コード例（必要に応じて）
```python

# Task 1
# 重要なポイントを含んだ擬似コード（全コードは書かない）
async def new_feature(param: str) -> Result:
    # パターン: まず入力を必ずバリデーション（src/validators.py参照）
    validated = validate_input(param)  # ValidationError発生あり
    
    # 注意: このライブラリはコネクションプーリング必須
    async with get_connection() as conn:  # src/db/pool.py参照
        # パターン: 既存のリトライデコレータを活用
        @retry(attempts=3, backoff=exponential)
        async def _inner():
            # 重要: APIは10req/sec超で429返す
            await rate_limiter.acquire()
            return await external_api.call(validated)
        
        result = await _inner()
    
    # パターン: 標準化されたレスポンス形式
    return format_response(result)  # src/utils/responses.py参照
```

### 統合ポイント（Integration Points）
```yaml
DATABASE:
  - migration: "usersテーブルに'feature_enabled'カラム追加"
  - index: "CREATE INDEX idx_feature_lookup ON users(feature_id)"
  
CONFIG:
  - 追加先: config/settings.py
  - パターン: "FEATURE_TIMEOUT = int(os.getenv('FEATURE_TIMEOUT', '30'))"
  
ROUTES:
  - 追加先: src/api/routes.py  
  - パターン: "router.include_router(feature_router, prefix='/feature')"
```

## バリデーションループ（Validation Loop）

### レベル1：構文・スタイルチェック
```bash
# まずこれらを実行し、エラーがあれば修正してから進める
ruff check src/new_feature.py --fix  # 自動修正できるものは修正
mypy src/new_feature.py              # 型チェック

# 期待値: エラーなし。エラーが出たら内容を読んで必ず修正。
```

### レベル2：ユニットテスト（既存パターンを活用）
```python
# test_new_feature.py を以下のテストケースで作成：
def test_happy_path():
    """基本動作のテスト"""
    result = new_feature("valid_input")
    assert result.status == "success"

def test_validation_error():
    """無効入力はValidationError"""
    with pytest.raises(ValidationError):
        new_feature("")

def test_external_api_timeout():
    """タイムアウトを適切に処理"""
    with mock.patch('external_api.call', side_effect=TimeoutError):
        result = new_feature("valid")
        assert result.status == "error"
        assert "timeout" in result.message
```

```bash
# テストが通るまで繰り返し実行：
uv run pytest test_new_feature.py -v
# 失敗時: エラー内容を読み、根本原因を理解して修正→再実行（テスト通すためのモック禁止）
```

### レベル3：統合テスト
```bash
# サービスを起動
uv run python -m src.main --dev

# エンドポイントをテスト
curl -X POST http://localhost:8000/feature \
  -H "Content-Type: application/json" \
  -d '{"param": "test_value"}'

# 期待値: {"status": "success", "data": {...}}
# エラー時: logs/app.logでスタックトレース確認
```

## 最終バリデーションチェックリスト
- [ ] すべてのテストが通る: `uv run pytest tests/ -v`
- [ ] リントエラーなし: `uv run ruff check src/`
- [ ] 型エラーなし: `uv run mypy src/`
- [ ] 手動テスト成功: [具体的なcurl/コマンド]
- [ ] エラーケースも適切に処理
- [ ] ログは有用だが冗長すぎない
- [ ] 必要ならドキュメントも更新

---

## NGパターン・アンチパターン
- ❌ 既存パターンがあるのに新しいものを作らない
- ❌ 「動くはず」でバリデーションを省略しない
- ❌ テスト失敗を放置せず必ず修正
- ❌ async文脈で同期関数を使わない
- ❌ 設定値はハードコーディングせずconfigで管理
- ❌ 例外はcatch-allせず、具体的に絞る
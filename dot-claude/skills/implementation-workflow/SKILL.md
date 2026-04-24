---
name: implementation-workflow
description: |
  Webアプリ実装サブワークフロー。DB→バックエンド→フロントエンド→結合検証の順でサブフェーズを逐次実行します。
  new-project-dev ワークフローの implementation フェーズから呼び出されます。
  各サブフェーズを専門エージェントに委譲し、全サブフェーズ完了後に WORKFLOW_COMPLETE を返します。
---

# implementation-workflow

new-project-dev の実装フェーズを担当するサブワークフロー。
DB設計・バックエンド・フロントエンド・結合検証を順次専門エージェントに委譲する。

## 前提条件

- `$ARGUMENTS`: 親ワークフローから引き継いだ context JSON（プロジェクト仕様・アーキテクチャ設計・スタック情報を含む）
- 状態ファイル: `.workflow/state/implementation-workflow.md`
- 既存スキル: `.claude/skills/`

## ONE STEP PER INVOCATION の原則

**呼び出しごとに正確に1サブフェーズのみ実行。** 状態ファイルで現在フェーズを管理し、再開時は状態ファイルを読み込んで未完了フェーズから継続する。

## Phase-Agent Mapping

| Phase ID | Name | Agent | Skills | Approval Gate |
|----------|------|-------|--------|:---:|
| db-implementation | データベース実装 | database-architect | database-migration-design | - |
| backend-implementation | バックエンド実装 | backend-developer | security-audit, performance-analysis | - |
| frontend-implementation | フロントエンド実装 | frontend-developer | accessibility-check | - |
| integration-check | 結合検証 | integration-validator | integration-validation | - |

## 実行フロー

### Step 0: 状態確認と再開判定

1. `.workflow/state/implementation-workflow.md` を Read で確認する
2. ファイルが存在しない場合: 新規作成し、currentPhase を `db-implementation` に設定する
3. ファイルが存在する場合: `currentPhase` を読み取り、当該フェーズから再開する
4. 全フェーズが `DONE` の場合: `WORKFLOW_COMPLETE` を出力して終了する

状態ファイルの初期フォーマット:

```
# Workflow: implementation-workflow
## Created: {date}
## Last Updated: {date}
## currentPhase: db-implementation

## Phases
- [ ] db-implementation
- [ ] backend-implementation
- [ ] frontend-implementation
- [ ] integration-check

## Context
{$ARGUMENTS の内容}

## Outputs
```

---

### Phase 1: db-implementation（データベース実装）

**担当エージェント**: database-architect
**使用スキル**: database-migration-design

**Agent tool への指示**:

```
あなたは database-architect です。
以下のコンテキストを参照してデータベース実装を行ってください。

【コンテキスト】
{状態ファイルの Context セクション}

【使用スキル】
.claude/skills/database-migration-design/SKILL.md を Read して参照し、そのプロセスに従って実行してください。

【実行内容】
1. アーキテクチャ設計・要件から論理スキーマを確定する
2. テーブル定義・インデックス戦略を設計する
3. マイグレーションスクリプトを生成する（ロールバックスクリプトも含む）
4. 初期シードデータが必要な場合は seed スクリプトも生成する
5. スキーマ設計の根拠・決定事項をドキュメント化する

【RALPHループ】
以下のwhileループで実行すること。自己判断での途中終了は禁止:
```

while (マイグレーション実行が成功しない) {

  1. マイグレーションスクリプトを実行する
  2. 実行エラー・整合性問題を確認する
  3. 問題があれば修正して1に戻る
  4. 全マイグレーションが正常完了 → "IMPLEMENTED" を出力して終了
}

```
外部検証（マイグレーション実行結果・スキーマ整合性チェック）が全Passした時のみ "IMPLEMENTED" を出力する。

【完了キーワード】
- 正常完了: 最後に必ず "IMPLEMENTED" と出力してください
- 外部検証が通らず修正不能: 最後に必ず "FAILED" と出力し、原因を詳細に記述してください

【成果物】
- マイグレーションスクリプト一覧（ファイルパス付き）
- インデックス設計の根拠
- スキーマER図（テキスト形式）
```

**完了後の処理**:

- エージェント出力に `IMPLEMENTED` が含まれる場合: 状態ファイルの `db-implementation` を `[x]` に更新し `currentPhase` を `backend-implementation` に変更する
- エージェント出力に `FAILED` が含まれる場合: 状態ファイルに失敗詳細を記録し `BLOCKED` を出力して停止する

---

### Phase 2: backend-implementation（バックエンド実装）

**担当エージェント**: backend-developer
**構造**: RALPHループ（タスク分解 → タスク単位ループ）

#### Step A: タスク分解（1回のAgent呼び出し）

以下の指示で backend-developer を Agent tool で呼び出す:

```
あなたは backend-developer です。
実装計画を作成してください。コードは書かず、タスク分解のみ行います。

【参照情報】
- .workflow/state/architecture.md（API設計方針・コンポーネント構成）
- .workflow/state/requirements.md（機能要件）
- .workflow/state/db-implementation 成果物（スキーマ・テーブル一覧）

【実施内容】
バックエンドの実装対象を独立して実装可能な単位に分解し、
`.workflow/state/backend-tasks.md` を以下の形式で作成してください:

# Backend Tasks
- [ ] {タスク名}: {概要（1行）}
- [ ] {タスク名}: {概要（1行）}
...

【制約】
- 1タスク = 1エンドポイントグループまたは1サービスモジュール
- 依存関係がある場合は順序を考慮して並べる
- タスク数の目安: 5〜15件

完了したら "TASKS_PLANNED" と出力してください。
```

`TASKS_PLANNED` 確認後、`.workflow/state/backend-tasks.md` の存在を確認する。

#### Step B: タスクループ（RALPHループ本体）

`.workflow/state/backend-tasks.md` を Read し、`[ ]` が残っている間、以下を繰り返す:

1. 次の未完了タスクを1件取り出す
2. backend-developer を Agent tool で呼び出す:

```
あなたは backend-developer です。
以下の1タスクのみを実装してください。

【今回のタスク】
{backend-tasks.md から取り出した1件}

【参照情報】
- .workflow/state/architecture.md（システム構成・API設計方針）
- .workflow/state/tech-stack.md（技術スタック）
- .workflow/state/backend-tasks.md（全タスク一覧・前のタスクの成果物パス確認用）
- .workflow/state/db-implementation 成果物（スキーマ・マイグレーションパス）
- tech-stack 系スキル（Glob で .claude/skills/ を確認して参照）
- .claude/skills/security-audit/SKILL.md
- .claude/skills/performance-analysis/SKILL.md

【実装プロセス（TDDサイクル）】
1. このタスクのテストを先に書く（Red）
2. テストがPassする最小実装を行う（Green）
3. テストが全Pass → リファクタリング（Refactor）

【完了キーワード】
- 実装・テスト完了: 最後に必ず "TASK_DONE" と出力してください
- 実装不能・ブロック: 最後に必ず "TASK_FAILED" と出力し、原因を記述してください
```

1. `TASK_DONE` → `.workflow/state/backend-tasks.md` の該当タスクを `[x]` に更新、次のタスクへ
2. `TASK_FAILED` → 状態ファイルに失敗詳細を記録し `BLOCKED` を出力して停止

**全タスク完了後**:

- 状態ファイルの `backend-implementation` を `[x]` に更新
- `currentPhase` を `frontend-implementation` に変更

---

### Phase 3: frontend-implementation（フロントエンド実装）

**担当エージェント**: frontend-developer
**構造**: RALPHループ（タスク分解 → タスク単位ループ）

#### Step A: タスク分解（1回のAgent呼び出し）

以下の指示で frontend-developer を Agent tool で呼び出す:

```
あなたは frontend-developer です。
実装計画を作成してください。コードは書かず、タスク分解のみ行います。

【参照情報】
- .workflow/state/architecture.md（画面構成・コンポーネント設計方針）
- .workflow/state/requirements.md（機能要件・ユーザーストーリー）
- .workflow/state/backend-tasks.md（バックエンドAPIの一覧・成果物パス確認用）

【実施内容】
フロントエンドの実装対象を独立して実装可能な単位に分解し、
`.workflow/state/frontend-tasks.md` を以下の形式で作成してください:

# Frontend Tasks
- [ ] {タスク名}: {概要（1行）}
- [ ] {タスク名}: {概要（1行）}
...

【制約】
- 1タスク = 1ページまたは1機能コンポーネントグループ
- 依存関係がある場合は順序を考慮して並べる
- タスク数の目安: 5〜15件

完了したら "TASKS_PLANNED" と出力してください。
```

`TASKS_PLANNED` 確認後、`.workflow/state/frontend-tasks.md` の存在を確認する。

#### Step B: タスクループ（RALPHループ本体）

`.workflow/state/frontend-tasks.md` を Read し、`[ ]` が残っている間、以下を繰り返す:

1. 次の未完了タスクを1件取り出す
2. frontend-developer を Agent tool で呼び出す:

```
あなたは frontend-developer です。
以下の1タスクのみを実装してください。

【今回のタスク】
{frontend-tasks.md から取り出した1件}

【参照情報】
- .workflow/state/architecture.md（画面構成・デザイン方針）
- .workflow/state/tech-stack.md（技術スタック）
- .workflow/state/frontend-tasks.md（全タスク一覧・前のタスクの成果物パス確認用）
- .workflow/state/backend-tasks.md（APIエンドポイント一覧・OpenAPI仕様パス）
- tech-stack 系スキル（Glob で .claude/skills/ を確認して参照）
- .claude/skills/accessibility-check/SKILL.md

【実装プロセス（TDDサイクル）】
1. このタスクのテストを先に書く（Red）
2. テストがPassする最小実装を行う（Green）
3. テストが全Pass → リファクタリング（Refactor）

【完了キーワード】
- 実装・テスト完了: 最後に必ず "TASK_DONE" と出力してください
- 実装不能・ブロック: 最後に必ず "TASK_FAILED" と出力し、原因を記述してください
```

1. `TASK_DONE` → `.workflow/state/frontend-tasks.md` の該当タスクを `[x]` に更新、次のタスクへ
2. `TASK_FAILED` → 状態ファイルに失敗詳細を記録し `BLOCKED` を出力して停止

**全タスク完了後**:

- 状態ファイルの `frontend-implementation` を `[x]` に更新
- `currentPhase` を `integration-check` に変更

---

### Phase 4: integration-check（結合検証）

**担当エージェント**: integration-validator
**使用スキル**: integration-validation

**Agent tool への指示**:

```
あなたは integration-validator です。
以下のコンテキストを参照して結合検証を行ってください。

【コンテキスト】
{状態ファイルの Context セクション}
{状態ファイルの全フェーズ成果物}

【参照スキル】
.claude/skills/integration-validation/SKILL.md を Read して参照し、そのプロセスに従って実行してください。

【実行内容】
1. バックエンド API とフロントエンドの結合動作確認
2. DB スキーマとバックエンド ORM マッピングの整合性確認
3. OpenAPI コントラクトテスト実行
4. E2E テストシナリオ実行（正常系・異常系・境界値）
5. 認証・認可フローの結合テスト
6. エラーハンドリングの結合確認
7. 統合テストレポート生成（.workflow/reports/integration-test.md）

【完了キーワード】
- 正常完了（全テストパス）: 最後に必ず "IMPLEMENTED" と出力してください
- 修正が必要な問題あり: 最後に必ず "NEEDS_REVISION" と出力し、修正が必要な箇所を一覧化してください

【成果物】
- 統合テストレポート: .workflow/reports/integration-test.md
- Go/No-Go 判定とその根拠
```

**完了後の処理**:

- エージェント出力に `IMPLEMENTED` が含まれる場合:
  - 状態ファイルの `integration-check` を `[x]` に更新する
  - 全サブフェーズ完了として状態ファイルに記録する
  - `WORKFLOW_COMPLETE` を出力して親ワークフローに制御を返す
- エージェント出力に `NEEDS_REVISION` が含まれる場合:
  - 統合テストレポートから問題の種類を特定する:
    - DBスキーマとORMのマッピング問題 → `currentPhase` を `db-implementation` に戻す
    - API仕様とフロントエンドの不一致 → `currentPhase` を `backend-implementation` に戻す
    - UIの結合動作問題 → `currentPhase` を `frontend-implementation` に戻す
    - 複合問題（複数フェーズに跨る）→ `currentPhase` を `db-implementation` に戻して全体を再実行
  - 状態ファイルに修正必要箇所と差し戻し先フェーズを記録する
  - `BLOCKED` を出力して停止し、修正必要事項と差し戻し先を明示する

---

## 状態ファイル更新仕様

各フェーズ完了後、`.workflow/state/implementation-workflow.md` に以下を追記する:

```
## Outputs

### db-implementation
{エージェント成果物サマリー}
完了日時: {datetime}

### backend-implementation
{エージェント成果物サマリー}
完了日時: {datetime}

### frontend-implementation
{エージェント成果物サマリー}
完了日時: {datetime}

### integration-check
{統合テストレポートパス}
Go/No-Go: {判定}
完了日時: {datetime}
```

---

## エラーハンドリング

| 状況 | アクション |
|------|-----------|
| エージェントが FAILED を出力 | 状態ファイルに原因を記録し `BLOCKED` を出力して停止 |
| エージェントが完了キーワードを出力しない | 最大1回リトライ、それでも取得できない場合は `BLOCKED` |
| integration-check で NEEDS_REVISION | 問題種別を特定し差し戻し先フェーズ（db/backend/frontend）を明示して BLOCKED |
| 状態ファイルの読み込み失敗 | 新規作成して db-implementation から開始 |

---

## 完了キーワード

| キーワード | 意味 |
|-----------|------|
| `WORKFLOW_COMPLETE` | 全サブフェーズ正常完了。親ワークフローへの制御返却 |
| `BLOCKED` | いずれかのフェーズで失敗・修正必要。詳細は状態ファイルを参照 |

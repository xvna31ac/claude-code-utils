# ワークフロースキル生成ガイド

workflow-generator エージェントが参照するSKILL.md生成・Phase-Agent Mapping記法・サブワークフロー委譲パターンの実装ガイド。

---

## ONE STEP PER INVOCATION の実装

生成するワークフロー SKILL.md は必ずこの原則を実装する。

### 必須要素

1. **フェーズ状態管理**: 状態ファイルで現在フェーズを記録
2. **再開ロジック**: 呼び出し時に状態ファイルを確認し、未完了フェーズから再開
3. **承認ゲート**: 承認が必要なフェーズでAskUserQuestionを使用
4. **委譲パターン**: 各フェーズでAgent toolにより専門エージェントに委譲

### SKILL.md の構造テンプレート

```markdown
---
name: {workflow-name}
description: {workflow description}
tools: Read, Glob, Bash, Agent, TaskCreate, TaskUpdate, TaskGet, AskUserQuestion
argument-hint: [task-name]
---

# {workflow-name}

{ワークフローの概要}

## 前提条件
- 状態ファイル: `.workflow/state/{name}.md`

## ONE STEP PER INVOCATION の原則
（標準テキスト）

## 実行フロー

### Phase 1: {phase-name}
...各フェーズの定義...
```

---

## Phase-Agent Mapping テーブルの記法

SKILL.md の各フェーズセクション内に以下の形式で記述:

```markdown
## Phase-Agent Mapping

| Phase | Type | Target | Approval Gate |
|-------|------|--------|:---:|
| {phase-id} | {agent\|workflow} | {agent-name\|workflow-name} | {yes\|no} |
```

### type: agent の実装パターン

```markdown
### Phase N: {phase-name}

**Step 1: {agent-name} エージェントに委譲（Agent tool）**

入力として渡す:
- {必要な入力項目}

エージェントからの出力キーワード:
- `{SUCCESS_KEYWORD}` → 次フェーズへ
- `{REJECT_KEYWORD}` → {フォールバックアクション}
```

### type: workflow の実装パターン（サブワークフロー委譲）

```markdown
### Phase N: {phase-name}（サブワークフロー）

**Step 1: {sub-workflow-name} SKILL.md を読み込む**

`.claude/skills/{sub-workflow-name}/SKILL.md` を Read

**Step 2: サブワークフローを Agent tool で呼び出す**

Agent tool で `{sub-workflow-name}` スキルを実行:
- 引き継ぎ情報（タスク文脈・前フェーズの成果物）を渡す
- サブワークフロー内で独自の ONE STEP PER INVOCATION ループを実行

**Step 3: 完了確認**

サブワークフローが `WORKFLOW_COMPLETE` を返したら次フェーズへ。
```

---

## サブワークフロー委譲の実装パターン

### 最上位ワークフローの委譲コード

```markdown
## {sub-workflow-name} への委譲

`.claude/skills/{sub-workflow-name}/SKILL.md` を読み込み、以下の情報を引き継ぐ:

```json
{
  "parentWorkflow": "{parent-workflow-name}",
  "taskContext": "{前フェーズの成果物の概要}",
  "artifacts": {
    "previousPhaseOutput": "..."
  }
}
```

サブワークフロー完了キーワード: `WORKFLOW_COMPLETE`
```

### サブワークフロー SKILL.md の特別要素

サブワークフローは通常のワークフローと同じ構造だが、以下を追加:

```markdown
## 引き継ぎ情報

親ワークフローから以下が提供される:
- `parentWorkflow`: 親ワークフロー名
- `taskContext`: タスク文脈
- `artifacts`: 前フェーズの成果物

## 完了キーワード

最終フェーズ完了時: `WORKFLOW_COMPLETE` を出力
```

---

## 状態ファイル設計

### パス規則

```
.workflow/state/{workflow-name}.md
```

### 基本フォーマット

```markdown
# Workflow: {workflow-name}
## Phase: {CURRENT_PHASE}
## Created: {date}
## Last Updated: {date}

## Context
{引き継ぎ情報・タスク定義}

## Phase {1}: {name}
Status: {pending|in_progress|completed}
Output: {エージェントの出力キーワードと概要}

## Phase {2}: {name}
Status: pending
```

---

## outputKeywords の実装

各エージェントへの指示に必ず含める:

```markdown
**重要**: 処理完了時は必ず以下のキーワードで終了すること:
- 成功時: `{SUCCESS_KEYWORD}`
- 失敗・ブロック時: `{REJECT_KEYWORD}`
```

---

## ループフェーズの実装パターン

タスク × N 回繰り返しが必要なフェーズ:

```markdown
### Phase N: {loop-phase-name}

**Step 1: 処理対象リストを取得**

状態ファイルから未処理アイテムを取得。

**Step 2: 次のアイテムを処理**

未処理アイテムが存在する場合:
- {agent-name} エージェントに委譲
- 処理済みに更新

未処理アイテムが0件になった場合: 次フェーズへ

**ループ終了条件**: 全アイテムの処理完了
```

---

## フロントマター設計

生成するワークフロー SKILL.md のフロントマター:

```yaml
---
name: {workflow-name}
description: {三人称・動詞始まり。最大1024文字。使用場面を含む}
tools: Read, Glob, Bash, Agent, TaskCreate, TaskUpdate, TaskGet, AskUserQuestion
argument-hint: [{task-name-or-options}]
---
```

**tools の選択基準**:
- WebSearch が必要: `WebSearch` を追加
- ファイル書き込みが必要: `Write`, `Edit` を追加
- シェル実行が必要: `Bash` を含める（デフォルト）

---

## 500行制限の管理

SKILL.md 本体は500行以内に収める。

**分割基準**:
- フェーズ詳細定義が長い場合 → `references/phases.md` に抽出
- フェーズ数が多い場合（8以上） → サブワークフロー化を検討

**references/phases.md の内容**:
- 各フェーズのエージェント詳細指示
- 入出力の詳細仕様
- エラーハンドリングの詳細

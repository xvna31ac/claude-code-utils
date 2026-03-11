# ワークフロースキルテンプレート

生成する SKILL.md の完全テンプレート。フラット構造・サブワークフロー構造の両方に対応。

---

## テンプレート: フラット構造（3-5フェーズ）

```markdown
---
name: {{workflow-name}}
description: {{description}}
tools: Read, Glob, Bash, Agent, TaskCreate, TaskUpdate, TaskGet, AskUserQuestion
argument-hint: [{{argument-hint}}]
---

# {{workflow-name}}

{{ワークフローの概要説明}}

## 前提条件

- `$ARGUMENTS`: {{引数の説明}}
- 状態ファイル: `.workflow/state/{{workflow-name}}.md`

## ONE STEP PER INVOCATION の原則

**呼び出しごとに正確に1フェーズのみ実行。** 状態ファイルで現在フェーズを管理し、再開時は状態ファイルを読み込んで継続する。

## Phase-Agent Mapping

| Phase | Type | Target | Approval Gate |
|-------|------|--------|:---:|
{{#phases}}
| {{id}} | {{type}} | {{target}} | {{approvalGate}} |
{{/phases}}

## 実行フロー

{{#each phases}}
### Phase {{index}}: {{name}}

```
目的: {{goal}}
入力: {{input}}
出力: {{output}}
```

**Step 1: {{agent-name}} エージェントに委譲（Agent tool）**

入力として渡す:
{{#inputs}}
- {{.}}
{{/inputs}}

**Step 2: 出力キーワード確認**

- `{{outputKeywords.next}}` → 次フェーズへ
- `{{outputKeywords.reject}}` → {{rejectAction}}

{{#if approvalGate}}
**[承認ゲート]** 完了後、AskUserQuestionでユーザー確認を取得してから次フェーズへ。
{{/if}}

**Step {{steps}}: 状態ファイルを更新**

`.workflow/state/{{../workflow-name}}.md` の Phase フィールドを `{{nextPhase}}` に更新。
{{/each}}

---

## エラーハンドリング

| エラー | アクション |
|--------|-----------|
| エージェントがエラーを返却 | 入力を確認して1回再試行 |
| 承認ゲートで合意なし | ユーザーに具体的な変更点を確認 |
| 状態ファイルが破損 | Discovery Stateから再開を提案 |

## スコープ境界

**このワークフローが担当**: {{scope}}
**このワークフローが担当しない**: {{outOfScope}}
```

---

## テンプレート: サブワークフロー付き構造

フラットテンプレートに加えて、`type: workflow` フェーズに以下を追加:

```markdown
### Phase {{N}}: {{phase-name}}（サブワークフロー委譲）

```
目的: {{goal}}
委譲先: {{sub-workflow-name}}
```

**Step 1: {{sub-workflow-name}} SKILL.md を読み込む**

`.claude/skills/{{sub-workflow-name}}/SKILL.md` を Read ツールで読み込む。

**Step 2: サブワークフローを Agent tool で呼び出す**

以下の引き継ぎ情報を渡してサブワークフローを実行:

```json
{
  "parentWorkflow": "{{workflow-name}}",
  "phase": "{{phase-id}}",
  "taskContext": "{{前フェーズの成果物の概要}}",
  "artifacts": {
    "previousPhaseOutput": "{{前フェーズの状態ファイルのパス}}"
  }
}
```

**Step 3: 完了確認**

サブワークフローが `WORKFLOW_COMPLETE` を返したら次フェーズへ。
`BLOCKED` または `FAILED` を返した場合はユーザーにエスカレーション。

{{#if approvalGate}}
**[承認ゲート]** サブワークフロー完了後、AskUserQuestionでユーザー確認を取得してから次フェーズへ。
{{/if}}

**Step {{steps}}: 状態ファイルを更新**
```

---

## テンプレート: サブワークフロー SKILL.md（子ワークフロー用）

親ワークフローから呼び出される子ワークフローの SKILL.md テンプレート:

```markdown
---
name: {{sub-workflow-name}}
description: {{description}}（{{parent-workflow-name}} のサブワークフロー）
tools: Read, Glob, Bash, Agent, TaskCreate, TaskUpdate, TaskGet, AskUserQuestion
argument-hint: [context-json]
---

# {{sub-workflow-name}}

{{サブワークフローの概要}}

## 前提条件

- 引き継ぎ情報（`$ARGUMENTS` または context JSON）:
  - `parentWorkflow`: 親ワークフロー名
  - `taskContext`: タスク文脈
  - `artifacts`: 前フェーズの成果物
- 状態ファイル: `.workflow/state/{{sub-workflow-name}}.md`

## ONE STEP PER INVOCATION の原則

（標準テキスト）

## Phase-Agent Mapping

| Phase | Type | Target | Approval Gate |
|-------|------|--------|:---:|
{{#subPhases}}
| {{id}} | agent | {{agentName}} | {{approvalGate}} |
{{/subPhases}}

## 実行フロー

{{#each subPhases}}
### Phase {{index}}: {{name}}
...（各フェーズ定義）
{{/each}}

## 完了キーワード

最終フェーズ完了時に `WORKFLOW_COMPLETE` を出力して親ワークフローに返す。
エラー・ブロック時は `BLOCKED` を出力。
```

---

## references/phases.md テンプレート

フェーズ詳細が長い場合に SKILL.md から分離するファイル:

```markdown
# {{workflow-name}} フェーズ定義

各フェーズのエージェント詳細指示・入出力仕様。

## Phase 1: {{phase-name}}

### エージェントへの詳細指示

{{エージェントへの詳細な指示内容}}

### 入力仕様

| 項目 | 説明 | 必須 |
|------|------|:---:|
| {{item}} | {{description}} | {{required}} |

### 出力仕様

```json
{
  "status": "success|error",
  "keyword": "{{outputKeyword}}",
  "summary": "...",
  "artifacts": {}
}
```

### エラーハンドリング

| エラーケース | アクション |
|------------|-----------|
| {{case}} | {{action}} |
```

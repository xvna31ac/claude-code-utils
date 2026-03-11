---
name: workflow-creator
description: ユーザーの要求からワークフロースキルを対話的に生成。Phase-Agent Mapping構造を持つ ONE STEP PER INVOCATION 型ワークフロー SKILL.md と関連エージェント・スキルを一括生成。シンプルワークフローとサブワークフロー（フェーズ内ワークフロー）の両方に対応。
tools: Read, Glob, Bash, Agent, TaskCreate, TaskUpdate, TaskGet, AskUserQuestion, WebSearch
argument-hint: [workflow-name]
---

# workflow-creator

ユーザーのラフな要求からワークフロースキル・関連エージェント・ドメインスキルを一括生成する「スキルを作るスキル」。

## 前提条件

- `$ARGUMENTS`: ワークフロー名（省略可。省略時はDISCOVERYで確定）
- 状態ファイル: `.workflow/state/{name}.md`（セッション間の継続に使用）
- 既存スキル確認: `.claude/skills/*/SKILL.md`
- 既存エージェント確認: `.claude/agents/*.md`

## ONE STEP PER INVOCATION の原則

**呼び出しごとに正確に1フェーズのみ実行。** 状態ファイルで現在フェーズを管理し、再開時は状態ファイルを読み込んで継続する。

## 実行フロー

### Phase 1: DISCOVERY

```
目的: ワークフロー要件をAskUserQuestionで収集
```

**Step 0: 状態確認**
1. `$ARGUMENTS`が存在する場合、`.workflow/state/{$ARGUMENTS}.md`を確認
2. 状態ファイルが存在し`phase != DISCOVERY`の場合 → 当該フェーズから再開
3. 既存スキル（`.claude/skills/`）・エージェント（`.claude/agents/`）を一覧化

**Step 1: ラウンド1 - ワークフローの本質**

AskUserQuestionで収集:
- 何を達成したいか？最終的な成果物（アーティファクト）は？
- 自動化・効率化したい具体的な課題は？
- ワークフロー名の候補（`$ARGUMENTS`がない場合）

**Step 2: ラウンド2 - フェーズと粒度**

AskUserQuestionで収集:
- 自然な作業の流れに分けると何段階か？各段階の名前は？
- 各段階の中に「さらにステップがある複雑な作業」はあるか？→ 複合フェーズ候補
- どの段階で人間の確認（承認ゲート）が必要か？
- ループ処理が必要な段階はあるか？（タスク × N 回など）

**Step 3: ラウンド3 - 品質・スコープ確認**

AskUserQuestionで収集:
- 各フェーズで必要な専門知識・ドメイン知識は？
- 開始条件・完了条件（成功基準）は？
- スコープの端点（最終的にどこまでやるか - コミット？PR？デプロイ？）
- **[条件付き]** ラウンド1〜2で外部サービス連携（通知・API呼び出し・外部システム操作等）が含まれそうな場合のみ: 利用可能なMCPサーバーはあるか？（例: slack-mcp, github-mcp など）

**Step 4: Discovery State を状態ファイルに保存**

`.workflow/state/{name}.md`に以下を記録:
```
# Workflow: {name}
## Phase: DESIGN
## Discovery State
goal: {ゴール}
phases: [{フェーズ名リスト}]
complex_phases: [{複合フェーズ候補}]
approval_gates: [{承認ゲートが必要なフェーズ}]
loops: [{ループフェーズ}]
domain_knowledge: {ドメイン知識}
scope_end: {スコープ端点}
external_integrations: [{外部サービス名と手段（mcp: サーバー名 / bash: APIコール / skill: スキル名）}]  # 外部連携がない場合は省略
```

→ **次回呼び出しで Phase 2: DESIGN を実行**

---

### Phase 2: DESIGN

```
目的: workflow-designer に委譲してDesign JSONを生成
入力: Discovery State（状態ファイルから読み込み）
出力: Design JSON
```

**Step 1: references/workflow-design-patterns.md を読み込む**

**Step 2: workflow-designer エージェントに委譲（Agent tool）**

入力として渡す:
- Discovery State の全内容
- 既存エージェント一覧（`.claude/agents/*.md` のfrontmatter name）
- 既存スキル一覧（`.claude/skills/*/SKILL.md` のfrontmatter name と description）
- workflow-design-patterns.md の内容

**Step 3: [承認ゲート①] Phase-Agent Mapping の確認**

Design JSON の Phase-Agent Mapping テーブルをユーザーに提示:
```
| Phase | Type | Target | Approval Gate |
|-------|------|--------|:---:|
...
```

AskUserQuestionで確認:
- このマッピングで問題ないか？
- 変更したいフェーズ・エージェントはあるか？

修正リクエストがある場合: workflow-designer に差し戻し（最大2回）

**Step 4: 状態ファイルを更新**
```
## Phase: GENERATE
## Design JSON
{Design JSONの内容}
```

---

### Phase 3: GENERATE

```
目的: workflow-generator に委譲して最上位ワークフロー SKILL.md を生成
入力: Design JSON
出力: SKILL.md 全文
```

**Step 1: references/generation-guide.md を読み込む**
**Step 2: references/templates/workflow-skill-template.md を読み込む**

**Step 3: workflow-generator エージェントに委譲（Agent tool）**

入力として渡す:
- Design JSON（最上位ワークフロー全体）
- workflow-skill-template.md の内容
- generation-guide.md の内容

**Step 4: 生成物を状態ファイルに記録**
```
## Phase: SUB_WORKFLOWS
## Generated Main SKILL.md
{SKILL.md全文}
```

---

### Phase 4: SUB_WORKFLOWS

```
目的: 複合フェーズのサブワークフロー SKILL.md を生成
条件: Design JSON に type=workflow のフェーズが存在する場合のみ実行
     存在しない場合は Phase 5 へスキップ
```

**Step 1: Design JSON から `type: workflow` フェーズを抽出**

**Step 2: 各サブワークフローに対して workflow-generator に委譲**

各 `type: workflow` フェーズについて:
- `subWorkflowName` と `subPhases` を入力として workflow-generator に委譲
- 生成された SKILL.md を状態ファイルに記録

**Step 3: 状態ファイルを更新**
```
## Phase: SUBAGENTS
## SubWorkflows
{各サブワークフロー名とSKILL.md内容}
```

---

### Phase 5: SUBAGENTS

```
目的: 各フェーズのエージェントファイルを生成
条件: agentReuse=false のフェーズのみ生成（agentReuse=true はスキップ）
```

**Step 1: references/templates/subagent-template.md を読み込む**

**Step 2: 新規作成が必要なエージェントを特定**

Design JSON から `agentReuse: false` のフェーズを抽出。
既存エージェント（`.claude/agents/*.md`）との重複を再確認。

**Step 3: 各エージェントに対して workflow-subagent-generator に委譲**

入力として渡す:
- フェーズ定義1件（id, name, agentName, skills, outputKeywords等）
- subagent-template.md の内容
- ドメイン情報（domainResearch.findings）

**Step 4: 状態ファイルを更新**
```
## Phase: DOMAIN_SKILLS
## Generated Agents
{各エージェント名と内容}
```

---

### Phase 6: DOMAIN_SKILLS

```
目的: 新規ドメインスキルを生成
条件: Design JSON の各フェーズの skills[] に reuse=false のスキルが存在する場合のみ
     存在しない場合は Phase 7 へスキップ
```

**Step 1: references/templates/domain-skill-template.md を読み込む**

**Step 2: 新規作成が必要なスキルを特定**

各フェーズの `skills[]` から `reuse: false` のスキルを抽出。
既存スキル（`.claude/skills/*/SKILL.md`）との重複を再確認。

**Step 3: 各スキルに対して workflow-domain-skill-generator に委譲**

**Step 4: 状態ファイルを更新**
```
## Phase: REVIEW
## Generated Domain Skills
{各スキル名と内容}
```

---

### Phase 7: REVIEW

```
目的: 全生成物の整合性・品質レビューと最終書き込み
```

**Step 1: workflow-artifact-reviewer に委譲（Agent tool）**

入力として渡す:
- 全生成物（状態ファイルから取得）
- Design JSON

**Step 2: グレード判定**

| グレード | アクション |
|---------|-----------|
| A | 承認ゲート②へ進む |
| B | 承認ゲート②へ進む（改善点をユーザーに通知） |
| C | 問題フェーズのエージェントに差し戻し（最大2回） |

Grade Cの場合: reviewerの指摘に基づき該当エージェントで修正 → 再レビュー

**Step 3: [承認ゲート②] 最終確認**

AskUserQuestionで確認:
- 生成物の一覧（ファイルパス・概要）を提示
- 既存ファイルと重複する場合は上書き確認
- 書き込み承認を取得

**Step 4: ファイル書き込み**

承認後、以下のパスにファイルを書き込み:

```
.claude/
├── skills/
│   ├── {workflow-name}/SKILL.md
│   ├── {workflow-name}/references/phases.md（Design JSON由来）
│   └── {sub-workflow-name}/SKILL.md（サブワークフローが存在する場合）
└── agents/
    └── {agent-name}.md（新規エージェントのみ）
```

ドメインスキルが生成された場合:
```
.claude/skills/{skill-name}/SKILL.md
```

**Step 5: 完了報告**

- 生成したファイル一覧
- 再利用した既存リソース一覧
- 次のアクション提案（`/{workflow-name} run` で実行可能）

---

## 状態管理仕様

### 状態ファイルフォーマット（`.workflow/state/{name}.md`）

```markdown
# Workflow: {name}
## Phase: {DISCOVERY|DESIGN|GENERATE|SUB_WORKFLOWS|SUBAGENTS|DOMAIN_SKILLS|REVIEW|COMPLETE}
## Created: {date}
## Last Updated: {date}

## Discovery State
{ヒアリング結果}

## Design JSON
```json
{...}
```

## Generated Main SKILL.md
{内容}

## SubWorkflows
{各サブワークフロー名と内容}

## Generated Agents
{各エージェント名と内容}

## Generated Domain Skills
{各スキル名と内容}
```

---

## エラーハンドリング

| エラー | アクション |
|--------|-----------|
| workflow-designer が無効なJSONを返却 | 入力を簡素化して1回再試行 |
| 承認ゲートで2回リトライ後も合意なし | 現在の設計を提示し、手動修正を提案 |
| Grade C が2回連続 | 問題リストを提示し、ユーザーに判断を委ねる |
| 既存ファイルと重複 | AskUserQuestion で上書き確認 |
| WebSearch 失敗 | Claude 知識で続行 + `webSearchFailed: true` を記録 |

## スコープ境界

**このスキルが担当**: ヒアリング・状態管理・エージェント委譲・ファイル書き込み。
**このスキルが担当しない**: フェーズ設計ロジック（workflow-designer）、SKILL.md生成（workflow-generator）、品質評価（workflow-artifact-reviewer）。

## References

- [workflow-design-patterns.md](references/workflow-design-patterns.md) - Phase 2で使用
- [generation-guide.md](references/generation-guide.md) - Phase 3/4で使用
- [templates/workflow-skill-template.md](references/templates/workflow-skill-template.md) - Phase 3/4で使用
- [templates/subagent-template.md](references/templates/subagent-template.md) - Phase 5で使用
- [templates/domain-skill-template.md](references/templates/domain-skill-template.md) - Phase 6で使用

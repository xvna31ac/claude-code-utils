---
name: workflow-creator
description: |
  ユーザーの要求からワークフロースキル・関連エージェント・ドメインスキルを生成・修正・補正・移行します。
  新規ワークフローの設計・実装、既存ワークフローの改修・組み換え後の補正、非準拠スキルのworkflow-creator形式への移行が必要な場合に自動適用されます。
  「ワークフローを作って」「フローを修正して」「組み換え後を補正して」「既存スキルをworkflow-creator形式に移行して」などのリクエストでトリガーされます。
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
2. 状態ファイルが存在し`Phase: COMPLETE`の場合 → 会話コンテキストからモードを判断:
   - 修正・変更・追加・更新の意図が明確 → **MODIFYモード確定**: `Phase: MODIFY_DISCOVERY`に更新して Phase 1M へ
   - 補正・整合性・組み換え後の意図が明確 → **REPAIRモード確定**: `Phase: REPAIR`に更新して Phase 1R へ
   - 移行・変換・置き換えの意図が明確 → **MIGRATEモード確定**: `Phase: MIGRATE`に更新して Phase 1G へ
   - 意図が不明瞭 → AskUserQuestionで「修正・補正・移行のどれか」を確認
3. 状態ファイルが存在し`Phase: REPAIR`の場合 → Phase 1R へ再開
   状態ファイルが存在し`Phase: MIGRATE`の場合 → Phase 1G へ再開
   状態ファイルが存在し他のフェーズの場合 → 当該フェーズから再開
4. 状態ファイルが存在しない場合 → `.claude/skills/{$ARGUMENTS}/SKILL.md` の存在を確認
   - ファイルが存在する + 補正・整合性チェックの意図が明確 → **REPAIRモード確定**: 状態ファイルを新規作成（`Phase: REPAIR`）して Phase 1R へ
   - ファイルが存在する + 移行・変換・置き換えの意図が明確 → **MIGRATEモード確定**: 状態ファイルを新規作成（`Phase: MIGRATE`）して Phase 1G へ
   - ファイルが存在する + 修正・変更・追加・更新の意図が明確 → **MODIFYモード確定**: 既存SKILL.mdを読み込んでDesign JSONを復元し、状態ファイルを新規作成（`Phase: MODIFY_DISCOVERY`）して Phase 1M へ
   - ファイルが存在する + 新規作成・作り直しの意図が明確 → **CREATEモード確定**
   - ファイルが存在する + 意図が不明瞭 → AskUserQuestionで確認してからモードを決定
   - ファイルが存在しない → CREATEモードで続行
5. 既存スキル（`.claude/skills/`）・エージェント（`.claude/agents/`）を一覧化

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

### Phase 1M: MODIFY_DISCOVERY

```
目的: 既存ワークフローの変更要件を収集しDESIGNへ引き継ぐ
条件: MODIFYモード時のみ実行（Phase: MODIFY_DISCOVERYの場合）
```

**Step 1: 既存Design JSONを読み込む**

状態ファイルの `## Design JSON` セクションから既存Design JSONを取得する。

**Step 2: AskUserQuestionで変更内容を収集**

以下を確認:
- どのフェーズを追加/削除/順序変更したいか？
- 特定フェーズのエージェント/スキルを差し替えたいか？
- サブワークフロー構成を変更したいか？

**Step 3: 状態ファイルを更新**

```
## mode: MODIFY
## modifyRequest: {変更内容の要約}
## previousDesignJSON:
{既存Design JSONをそのまま記録}
## Phase: DESIGN
```

→ **次回呼び出しで Phase 2: DESIGN を MODIFYモードで実行**

---

### Phase 1R: REPAIR

```
目的: 既存成果物の不整合を検出し補正計画を策定する
条件: REPAIRモード時のみ実行（Phase: REPAIRの場合）
```

**`.claude/skills/workflow-creator/references/modes.md` を Read して Phase 1R 詳細手順に従い実行する。**

→ **Phase 3〜7をMODIFYと同じパスで実行**

---

### Phase 1G: MIGRATE

```
目的: 既存の非準拠スキルを分析しDiscovery Stateを自動生成する
条件: MIGRATEモード時のみ実行（Phase: MIGRATEの場合）
```

**`.claude/skills/workflow-creator/references/modes.md` を Read して Phase 1G 詳細手順に従い実行する。**

→ **次回呼び出しで Phase 2: DESIGN を CREATEモードと同様に実行**

---

### Phase 2: DESIGN

```
目的: workflow-designer に委譲してDesign JSONを生成
入力: Discovery State（状態ファイルから読み込み）
出力: Design JSON
```

**Step 1: `.claude/skills/workflow-creator/references/workflow-design-patterns.md` を読み込む**

**Step 2: workflow-designer エージェントに委譲（Agent tool）**

**CREATE・MIGRATEモードの場合**、入力として渡す:
- Discovery State の全内容
- 既存エージェント一覧（`.claude/agents/*.md` のfrontmatter name）
- 既存スキル一覧（`.claude/skills/*/SKILL.md` のfrontmatter name と description）
- workflow-design-patterns.md の内容

**MODIFYモードの場合**、入力として渡す:
- `previousDesignJSON`（変更前のDesign JSON）
- `modifyRequest`（変更内容の要約）
- 既存エージェント一覧・既存スキル一覧
- workflow-design-patterns.md の内容
- 指示: 「既存Design JSONに対して変更要件を適用した新しいDesign JSONを返すこと。変更対象外のフェーズは既存定義をそのまま維持すること」

以下の設計パターンを **各フェーズに明示的に適用** するよう指示する:
1. **成果物生成型フェーズ**（実装・文書作成・設計等）→ 「成功基準を先行定義 → 出力を検証 → 未達なら差し戻し」を組み込む（例: テストファーストや検証ステップを実行フローに含める）
2. **ループ構造が必要なフェーズ**（実装・生成・修正等）→ 「検証 → 修正 → 再検証」のサイクルをサブフェーズとして明示する
3. **NEEDS_REVISION の差し戻し先** → 各フェーズで「失敗時にどのフェーズに戻るか」を具体的に指定する
4. **後工程から前工程へのフィードバックパス** → QA・レビューフェーズが問題を検出した場合に実装フェーズへ戻るパスを設計する
5. **RALPHループパターン（実装・生成フェーズのオーケストレーター設計）** → タスク数が多くなりうる実装・生成フェーズのオーケストレーターには、以下の2段階構造を生成する:
   - **Step A: タスク分解**（1回のAgent呼び出し）→ エージェントに作業を独立したタスク単位に分解させ、チェックリスト形式でファイルに保存する（例: `.workflow/state/{phase-name}-tasks.md`）
   - **Step B: タスクループ**（RALPHループ本体）→ ファイルの未完了タスクが残る限り、1タスクにつき1回のAgent呼び出しを繰り返す。各呼び出しはフレッシュなコンテキストで実行され、完了したタスクをファイルに記録する
   - **適用判断**: フェーズタイプが「実装・生成」なら agentReuse の有無に関係なく適用する。「分析・設計」（出力が1つの統合ドキュメント）と「検証・レビュー」（手続き実行型）には適用しない

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

CREATE・MIGRATEモードの場合:
```
## Phase: GENERATE
## Design JSON
{Design JSONの内容}
```

MODIFYモードの場合:
```
## Phase: IMPACT_ANALYSIS
## Design JSON
{新Design JSONの内容}
```

---

### Phase 2M: IMPACT_ANALYSIS

```
目的: 旧Design JSONと新Design JSONを比較して再生成が必要な成果物を特定する
条件: MODIFYモード時のみ実行（Phase: IMPACT_ANALYSISの場合）
CREATEモードはこのフェーズをスキップしてPhase 3へ
```

**Step 1: 差分を比較する**

`previousDesignJSON` と新 `Design JSON` を比較し、変更されたフェーズを特定する。

**Step 2: 影響する生成物を決定する**

| 変更種別 | 再生成が必要な成果物 |
|---------|-------------------|
| フェーズ追加 | メインSKILL.md（遷移追加）+ 新フェーズのagent/subworkflow |
| フェーズ削除 | メインSKILL.md（遷移削除）のみ |
| エージェント差し替え（agentReuse: false） | 該当agent.md |
| エージェント差し替え（agentReuse: true） | メインSKILL.md（エージェント名更新）のみ |
| フェーズ順序変更 | メインSKILL.md（遷移修正）のみ |
| サブワークフロー変更 | 該当サブワークフローSKILL.md |

**Step 3: 状態ファイルを更新**

```
## impactedGenerations: [GENERATE, SUBAGENTS]  # 実行が必要なPhase 3〜7を列挙
## Phase: GENERATE
```

→ Phase 3以降は `impactedGenerations` に含まれるフェーズのみ実行し、それ以外はスキップする

---

### Phase 3: GENERATE

```
目的: workflow-generator に委譲して最上位ワークフロー SKILL.md を生成
入力: Design JSON
出力: SKILL.md 全文
MODIFYモード条件: impactedGenerations に GENERATE が含まれる場合のみ実行。含まれない場合は Phase 4 へスキップ
```

**Step 1: `.claude/skills/workflow-creator/references/generation-guide.md` を読み込む**
**Step 2: `.claude/skills/workflow-creator/references/templates/workflow-skill-template.md` を読み込む**

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
MODIFYモード条件: impactedGenerations に SUB_WORKFLOWS が含まれる場合のみ実行。含まれない場合は Phase 5 へスキップ
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
MODIFYモード条件: impactedGenerations に SUBAGENTS が含まれる場合のみ実行。含まれない場合は Phase 6 へスキップ
```

**Step 1: `.claude/skills/workflow-creator/references/templates/subagent-template.md` を読み込む**

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
MODIFYモード条件: impactedGenerations に DOMAIN_SKILLS が含まれる場合のみ実行。含まれない場合は Phase 7 へスキップ
```

**Step 1: `.claude/skills/workflow-creator/references/templates/domain-skill-template.md` を読み込む**

**Step 2: 新規作成が必要なスキルを特定**

各フェーズの `skills[]` から `reuse: false` のスキルを抽出。
既存スキル（`.claude/skills/*/SKILL.md`）との重複を再確認。

**Step 3: 各スキルに対して workflow-domain-skill-generator に委譲**

入力として渡す:
- スキル定義1件（スキル名・担当フェーズ・必要な機能・ドメイン知識）
- domain-skill-template.md の内容
- Discovery State の `domain_knowledge` フィールドの内容

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
MODIFYモード条件: 常に実行（ただし impactedGenerations に含まれる成果物のみをレビュー・書き込み対象とする）
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

Grade Cの場合: 以下の基準で差し戻し先を決定して再実行 → 再レビュー（最大2回）

| 指摘箇所 | 差し戻し先 |
|---------|-----------|
| SKILL.md（フェーズ遷移・承認ゲート定義） | Phase 3: GENERATE を再実行 |
| サブワークフロー SKILL.md | Phase 4: SUB_WORKFLOWS を再実行 |
| エージェントファイル（.claude/agents/） | Phase 5: SUBAGENTS を再実行 |
| ドメインスキル | Phase 6: DOMAIN_SKILLS を再実行 |

**Step 3: [承認ゲート②] 最終確認**

AskUserQuestionで確認:
- 生成物の一覧（ファイルパス・概要）を提示
- 既存ファイルと重複する場合は上書き確認
- 書き込み承認を取得（承認しない場合: ファイル書き込みをキャンセルし、ユーザーへ修正案を提示して終了する）

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

## 状態管理仕様（`.workflow/state/{name}.md`）

```markdown
# Workflow: {name}
## Phase: {DISCOVERY|MODIFY_DISCOVERY|REPAIR|MIGRATE|DESIGN|IMPACT_ANALYSIS|GENERATE|SUB_WORKFLOWS|SUBAGENTS|DOMAIN_SKILLS|REVIEW|COMPLETE}
## Created: {date}
## Last Updated: {date}

## mode: {CREATE|MODIFY|REPAIR|MIGRATE}  # CREATE以外の場合のみ記録
## modifyRequest: {変更内容の要約}  # MODIFYモードの場合のみ記録
## repairIssues: {検出問題の要約}  # REPAIRモードの場合のみ記録
## sourceSkill: {元スキルのパス}  # MIGRATEモードの場合のみ記録
## impactedGenerations: [{GENERATE,SUB_WORKFLOWS,SUBAGENTS,DOMAIN_SKILLS}]  # MODIFY/REPAIRモードの場合のみ記録

## Discovery State
{ヒアリング結果}

## previousDesignJSON:  # MODIFYモードの場合のみ記録
```json
{変更前のDesign JSON}
```

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

**担当**: ヒアリング・状態管理・エージェント委譲・ファイル書き込み（設計・生成・品質評価は各専門エージェントに委譲）。

## References

- [modes.md](.claude/skills/workflow-creator/references/modes.md) - Phase 1R/1Gで使用
- [workflow-design-patterns.md](.claude/skills/workflow-creator/references/workflow-design-patterns.md) - Phase 2で使用
- [generation-guide.md](.claude/skills/workflow-creator/references/generation-guide.md) - Phase 3/4で使用
- [templates/workflow-skill-template.md](.claude/skills/workflow-creator/references/templates/workflow-skill-template.md) - Phase 3/4で使用
- [templates/subagent-template.md](.claude/skills/workflow-creator/references/templates/subagent-template.md) - Phase 5で使用
- [templates/domain-skill-template.md](.claude/skills/workflow-creator/references/templates/domain-skill-template.md) - Phase 6で使用

# REPAIR・MIGRATEモード 詳細仕様

workflow-creator の Phase 1R（REPAIR）・Phase 1G（MIGRATE）の詳細手順。
SKILL.md の Phase 1R・1G から Read して参照する。

---

## Phase 1R: REPAIR 詳細

### Step 1: 対象成果物を読み込む

以下を Read する:
- `.claude/skills/{name}/SKILL.md`
- 状態ファイル内の `## Design JSON` セクション（存在する場合）
- Design JSON に記載された全エージェント（`.claude/agents/{agent-name}.md`）

### Step 2: workflow-artifact-reviewer に委譲（Agent tool）

以下の情報を入力として渡す:
- 読み込んだ SKILL.md 全文
- Design JSON（存在する場合）
- 読み込んだ全エージェントファイルの内容

検出させる不整合:
- フェーズ間の遷移切れ（Phase N の next が Phase N+1 を指していない等）
- エージェント参照ミス（SKILL.md 内のエージェント名が `.claude/agents/` に存在しない）
- outputKeywords 不一致（SKILL.md の期待キーワードとエージェント定義の返却キーワードが異なる）
- 状態ファイル定義の欠落（状態ファイルフォーマットに未定義のフィールドが使用されている）

期待出力形式:
```json
{
  "issues": [
    {
      "severity": "Critical|Important|Minor",
      "type": "transition|agent-ref|keyword|state-field",
      "location": "Phase X Step Y",
      "description": "問題の説明",
      "suggestedFix": "修正案"
    }
  ],
  "summary": "検出した問題の総数と概要"
}
```

### Step 3: [補正承認ゲート]

AskUserQuestion で以下を提示:
- 検出した問題一覧（severity 別にグループ化）
- 補正に必要なフェーズ（impactedGenerations）の見積もり
- 「この補正計画で実施しますか？」

### Step 4: 状態ファイルを更新

```
## mode: REPAIR
## repairIssues: {検出された問題の要約}
## impactedGenerations: [{補正が必要なPhase}]
## Phase: GENERATE  # または最初の補正対象Phase
```

→ Phase 3〜7を MODIFY と同じパスで実行

---

## Phase 1G: MIGRATE 詳細

### Step 1: 対象スキルを読み込む

`$ARGUMENTS` で指定された SKILL.md を Read する。

### Step 2: Discovery State を自動生成

既存スキルを以下のマッピングで Discovery State フィールドに変換する:

| Discovery State フィールド | 既存スキルの参照箇所 |
|--------------------------|-------------------|
| `goal` | スキルの概要・description の役割文 |
| `phases` | 各 Step / フェーズの見出し一覧 |
| `complex_phases` | 複数のサブステップを持つフェーズ |
| `approval_gates` | 「確認」「承認」「ユーザー確認」が記述されているフェーズ |
| `loops` | 「繰り返し」「N回」「リスト処理」が記述されているフェーズ |
| `domain_knowledge` | スキルが前提とする専門知識・ドメイン |
| `scope_end` | スキルの完了条件・最終成果物 |

生成した Discovery State をユーザーに提示し、AskUserQuestion で補足・修正を確認する。

### Step 3: 状態ファイルを更新

```
## mode: MIGRATE
## sourceSkill: {元スキルのパス}
## Phase: DESIGN
## Discovery State
{自動生成したDiscovery State}
```

→ 次回呼び出しで Phase 2: DESIGN を CREATE モードと同様に実行

# ワークフロー設計パターン

workflow-designer エージェントが参照するフェーズ設計・サブワークフロー判定・承認ゲート配置の基準。

---

## AIワークフロー設計の基本原則

AIエージェントが実行するワークフローは、以下をデフォルトで組み込む。

### フェーズタイプ別の品質担保パターン

| フェーズタイプ | 例 | 品質担保パターン |
|---|---|---|
| **成果物生成型**（出力が曖昧） | 文書作成・設計・実装 | 実行前に成功基準を定義 → 出力を検証 → 未達なら差し戻し |
| **手続き実行型**（結果が確定的） | コマンド実行・API呼び出し | 成功/失敗をキャプチャ → 失敗時のフォールバックを定義 |

---

## 複合フェーズの判定基準

以下のいずれかを満たすフェーズを「複合フェーズ」と判定し、`type: workflow` として記録する:

| 判定基準 | 例 |
|---------|-----|
| 内部に2つ以上の独立したステップがある | 「設計フェーズ」= PRD作成 + レビュー + 承認 |
| ループ構造が必要（タスク × N 回繰り返し） | 「実装フェーズ」= タスクA→B→C を順次実行 |
| フェーズ内に独立した承認ゲートが必要 | 「QAフェーズ」= テスト実行 + 承認 + 修正 |
| 異なるエージェントが複数連携する | 「デプロイフェーズ」= ビルド + テスト + デプロイ |

**2階層上限**: サブワークフロー内にさらにサブワークフローは作成しない。

---

## フェーズ粒度のガイドライン

### 適切な粒度（1フェーズ = 1エージェント）

```
✅ Good: analyze → design → implement → review → finalize
✅ Good: discover → plan → execute → validate
```

### 過細粒度（統合すべき）

```
❌ Too fine: analyze-context → analyze-requirements → analyze-scope
   → analyze に統合
```

### 過粗粒度（分割すべき）

```
❌ Too coarse: "実装フェーズ"（PRD作成+設計+コーディング+テスト）
   → 複合フェーズとして type: workflow に変換
```

---

## 承認ゲートの配置基準

### 必須配置

| 状況 | 理由 |
|------|------|
| 不可逆な操作の直前（PR作成、デプロイ等） | 誤操作防止 |
| 大量の作業を始める直前 | 方向性の確認 |
| 外部システムへの影響がある操作の直前 | コスト・リスク管理 |

### 推奨配置

| 状況 | 理由 |
|------|------|
| フェーズ1（要件確認後） | 方向性の早期確認 |
| 設計フェーズ完了後 | 実装前の最終合意 |
| 大規模サブワークフロー完了後 | 品質確認 |

### 省略可能

| 状況 | 理由 |
|------|------|
| 可逆な操作（ファイル生成等） | 後で変更可能 |
| 短時間で完了するフェーズ | ユーザー負荷が低い |

---

## outputKeywords の設計

各フェーズのエージェントが返却するキーワード。次フェーズへの遷移を制御する。

### 標準キーワードセット

| ケース | キーワード | 意味 |
|--------|-----------|------|
| 成功・継続 | `ANALYZED`, `DESIGNED`, `IMPLEMENTED`, `REVIEWED` | 正常完了 |
| 完了・終了 | `COMPLETE`, `DONE`, `FINALIZED` | ワークフロー完了 |
| ブロック | `BLOCKED`, `FAILED`, `ERROR` | エラー・停止 |
| 要修正 | `NEEDS_REVISION`, `REJECTED` | 差し戻し |

### ドメイン固有キーワード

ドメインの文脈に合わせてカスタマイズ可能:
- PR作成ワークフロー: `PR_CREATED`, `PR_MERGED`
- テストワークフロー: `TESTS_PASSED`, `TESTS_FAILED`

---

## Phase-Agent Mapping の記法

Design JSON における型定義:

```json
{
  "id": "phase-id",
  "name": "フェーズ日本語名",
  "type": "agent | workflow",
  "agentName": "汎用エージェント名（type=agentの場合）",
  "agentReuse": true,
  "skills": [
    { "name": "スキル名", "reuse": true }
  ],
  "approvalGate": false,
  "outputKeywords": {
    "next": "SUCCESS_KEYWORD",
    "reject": "REJECT_KEYWORD"
  }
}
```

`type: workflow` の場合の追加フィールド:
```json
{
  "type": "workflow",
  "subWorkflowName": "sub-workflow-name",
  "subPhases": [
    { "id": "sub-id", "agentName": "agent-name", "agentReuse": true }
  ]
}
```

---

## 汎用命名の原則

### エージェント命名規則

| パターン | 例 |
|---------|-----|
| 役割を表す名詞形 | `requirement-analyzer`, `code-reviewer`, `qa-engineer` |
| 能力を表す動名詞形 | `document-reviewer`, `task-executor` |
| ドメイン+役割 | `security-auditor`, `prd-creator` |

**禁止パターン**: ワークフロー名を含む命名
```
❌ feature-dev-analyzer.md
❌ code-review-workflow-executor.md
✅ requirement-analyzer.md
✅ task-executor.md
```

### 既存スキルの再利用判定基準

スキルの再利用は「完全一致」ではなく「機能的包含」で判断する。

| 判定 | 条件 | `reuse` |
|------|------|:-------:|
| ✅ 再利用OK | 必要な機能・観点がスキルに含まれている（余分な観点があってもよい） | `true` |
| ❌ 再利用NG | 必要な機能・観点がスキルに不足している | `false` |

**例: レビュースキルの判定**
- 包括的レビュースキル（品質・セキュリティ・パフォーマンス観点を含む）を「品質レビューフェーズ」で使う → ✅ 余分な観点は無視される
- セキュリティ特化スキルを「総合品質レビューフェーズ」で使う → ❌ 品質・パフォーマンス観点が不足

---

### 既存エージェントの再利用優先

以下の既存エージェントを優先的に再利用:
- `requirement-analyzer` - 要件分析
- `prd-creator` - PRD作成
- `technical-designer` - 設計書作成
- `document-reviewer` - ドキュメントレビュー
- `task-executor` - タスク実行
- `code-reviewer` - コードレビュー
- `quality-fixer` - 品質修正
- `work-planner` - 作業計画

---

## サブワークフロー構造例

### フラット（3-5フェーズ）

```
analyze → design → implement → review → finalize
```

Phase-Agent Mapping:
```
| analyze   | agent    | requirement-analyzer | no  |
| design    | agent    | prd-creator          | yes |
| implement | agent    | task-executor        | no  |
| review    | agent    | code-reviewer        | yes |
| finalize  | agent    | work-planner         | no  |
```

### サブワークフロー付き（複合フェーズあり）

```
analyze → [design workflow] → [implement workflow] → qa → finalize
```

Phase-Agent Mapping（最上位）:
```
| analyze   | agent    | requirement-analyzer      | no  |
| design    | workflow | feature-dev-design        | yes |
| implement | workflow | feature-dev-implement     | no  |
| qa        | agent    | qa-engineer               | yes |
| finalize  | agent    | work-planner              | no  |
```

feature-dev-design サブワークフロー内:
```
| prd     | agent | prd-creator       | no  |
| review  | agent | document-reviewer | yes |
| arch    | agent | technical-designer| no  |
```

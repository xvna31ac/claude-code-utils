# ワークフローテンプレート

Phase-SubAgent-Skills マッピングの設計テンプレートと記述例。
このファイルは Phase 5 で Read して使用する。

---

## マッピング表テンプレート

```markdown
## Phase-SubAgent-Skills マッピング: {ワークフロー名}

| Phase | フェーズ名 | 担当 Sub Agent | 使用 Skills | 新規/既存 | 備考 |
|-------|-----------|---------------|------------|---------|------|
| 1 | {フェーズ名} | — | — | — | Claude 直接実行 |
| 2 | {フェーズ名} | {agent-name} | — | 既存 | |
| 3 | {フェーズ名} | {agent-name} | {skill-name} | 新規 | |
```

**凡例:**
- 担当 Sub Agent が「—」の場合: Claude がそのフェーズを直接実行する
- 使用 Skills が「—」の場合: そのフェーズでスキル呼び出しは不要

---

## Sub Agent 設計テンプレート

新規作成が必要な Sub Agent の設計時に使用する。

```yaml
# 設計仕様（実際のファイルは .claude/agents/<name>.md に作成）
name: {ケバブケース}
ファイルパス: .claude/agents/{name}.md
役割: {1行で役割を説明}
使用フェーズ: Phase {N}, Phase {M}
ロールタイプ: コードライター型 / リードオンリー型 / リサーチ型 / ドキュメント型
tools: Read, Write, Edit, Bash, Glob, Grep  # ロールタイプに応じて選択
既存代替案: {類似の既存 Sub Agent と差異の説明}
```

---

## Skills 設計テンプレート

新規作成が必要な Skill の設計時に使用する。

```yaml
# 設計仕様（実際のファイルは .claude/skills/<name>/SKILL.md に作成）
name: {ケバブケース}
ファイルパス: .claude/skills/{name}/SKILL.md
役割: {1行で役割を説明}
使用フェーズ: Phase {N}
スキルタイプ: 個別スキル / ワークフロースキル
参照ファイル: {references/ に分離する詳細情報がある場合に列挙}
既存代替案: {類似の既存 Skill と差異の説明}
```

---

## 命名規則

| 種別 | 規則 | 例 |
|------|------|-----|
| Sub Agent | 役割・人物名（ケバブケース） | `backend-developer`, `code-reviewer` |
| Skill | 行為・能力名（ケバブケース） | `skill-creator`, `code-review`, `deployment` |
| ワークフロースキル | `workflow-` プレフィックス必須 | `workflow-creator`, `workflow-release` |

---

## マッピング設計チェックポイント

Phase-SubAgent-Skills マッピングを確定する前に以下を確認する:

- [ ] 各フェーズの責任範囲が明確で重複していないか
- [ ] 既存 Sub Agent / Skills を最大限再利用しているか
- [ ] 新規作成リソースに既存との重複がないか（Phase 6 で確認済みか）
- [ ] 承認ゲートは破壊的変更前・設計確定の場面にのみ配置されているか
- [ ] Sub Agent の tools がロールタイプと整合しているか
- [ ] Skill の name が `workflow-` プレフィックスルールを遵守しているか

---

## マッピング例（デプロイワークフロー）

| Phase | フェーズ名 | 担当 Sub Agent | 使用 Skills | 新規/既存 |
|-------|-----------|---------------|------------|---------|
| 1 | 影響範囲分析 | impact-analyzer | impact-analysis | 既存 |
| 2 | テスト実行・品質検証 | qa-expert | integration-validation | 既存 |
| 3 | セキュリティ監査 | security-auditor | security-audit | 既存 |
| 4 | デプロイ実行 | devops-engineer | deployment | 既存 |
| 5 | ヘルスチェック | devops-engineer | — | 既存 |

この例では全リソースが既存利用であり新規作成は不要。

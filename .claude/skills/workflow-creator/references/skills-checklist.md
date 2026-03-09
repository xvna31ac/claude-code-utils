# Skills 作成チェックリスト

Phase 12 で新規 Skill を作成する際に適用する品質チェックリスト。
詳細なガイドラインは `.claude/skills/skill-review/references/skill-creator-guidelines.md` を参照すること。

---

## 必須フォーマットチェック（個別スキル・ワークフロースキル共通）

作成後に以下を1項目ずつ確認する:

### frontmatter
- [ ] `name` フィールドが存在する（ケバブケース）
- [ ] `description` フィールドが `|` ブロックスカラー形式（`>` は不可）
- [ ] `tools` フィールドが含まれていない（スキルは tools を指定しない）
- [ ] description が3文構成:
  - 1文目: 動詞で始まる役割説明
  - 2文目: 「〜が必要な場合に自動適用されます」の形式
  - 3文目: 具体的なトリガー例

### body
- [ ] 500 行以内
- [ ] body に「適用条件」「使用場面」「When to Use」セクションがない（description で完結）
- [ ] 各 Step が「実行内容 / 専門性 / 期待成果物 / 完了基準」の4項目を含む
- [ ] Step 1 が分岐形式（存在する場合 / しない場合）の場合、例外注記がある

### 参照パス
- [ ] 参照パスがプロジェクトルート相対の完全パス（例: `.claude/skills/xxx/references/foo.md`）
- [ ] ワイルドカード（`*.md` 等）を Read 参照に使用していない
- [ ] 参照先ファイルが実際に存在する（Glob で確認）
- [ ] 参照ファイルは「必要なフェーズで明示的に Read する」と記述されている

---

## ワークフロースキル追加チェック

ワークフロースキル（`name` が `workflow-` で始まるもの）の場合のみ確認する:

- [ ] `name` が `workflow-` プレフィックスで始まる
- [ ] フェーズ一覧テーブル（`| Phase | フェーズ名 | 使用Skill / 担当ロール | 承認ゲート |`）が存在する
- [ ] 承認ゲートに「確認内容 / APPROVE 時 / REJECT 時」が明記されている
- [ ] 状態ファイルパスが `.workflow/state/<name>.md` 形式
- [ ] 個別スキル呼び出しが `skill-name` 形式（`/skill-name` は誤り）

---

## 既存スキル重複確認

新規作成前に以下を確認する:

| 既存スキル | 機能 | 新規スキルとの差異 |
|-----------|------|-----------------|
| skill-review | SKILL.md の品質レビュー | |
| agent-review | エージェント定義の品質レビュー | |
| code-review | コード品質・設計レビュー | |
| requirements-analysis | 要件定義・ユースケース抽出 | |
| architecture-design | アーキテクチャ・API設計 | |
| database-migration-design | DBスキーマ変更・マイグレーション設計 | |
| impact-analysis | 変更前の影響範囲分析 | |
| integration-validation | APIコントラクト・E2E検証 | |
| security-audit | セキュリティ脆弱性・シークレット監査 | |
| performance-analysis | パフォーマンスボトルネック静的解析 | |
| deployment | デプロイ・マイグレーション実行 | |
| accessibility-check | WAI-ARIA / WCAG 準拠検証 | |

上記既存スキルと機能が重複する場合は新規作成せず既存を利用する。

---

## ファイル配置確認

- [ ] ファイルパスが `.claude/skills/{name}/SKILL.md` である
- [ ] 参照ファイルは `.claude/skills/{name}/references/` に配置されている
- [ ] `README.md` などの補助ドキュメントを別途作成していない（SKILL.md に統合する）

---

## 品質確認フロー（作成後）

1. `skill-review` を新規作成した SKILL.md に対して実行する
2. Critical / Important が報告された場合は修正する
3. 修正後に `skill-review` を再実行して改善を確認する
4. Critical ゼロ・Important ゼロになったら完了とする

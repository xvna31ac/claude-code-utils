# Sub Agent 作成チェックリスト

Phase 10 で新規 Sub Agent を作成する際に適用する品質チェックリスト。
詳細なガイドラインは `.claude/skills/agent-review/references/agent-guidelines.md` を参照すること。

---

## 必須フォーマットチェック

作成後に以下を1項目ずつ確認する:

### frontmatter
- [ ] `name` フィールドが存在する（ケバブケース・ファイル名と一致）
- [ ] `description` フィールドが存在する（quoted string 形式・`|` ブロックスカラーは不可）
- [ ] `tools` フィールドが存在する（有効値: Read, Write, Edit, Bash, Glob, Grep, WebFetch, WebSearch, Agent）
- [ ] `tools` に重複がない
- [ ] `model` を指定する場合は sonnet/opus/haiku 系のいずれか
- [ ] `permissionMode` を指定する場合は有効値（`dontAsk` / `acceptEdits` / `bypassPermissions`）
- [ ] `skills` フィールドが frontmatter に存在する（使用する Skills を列挙・OSS代替への差し替えを想定した設計）

### description の品質
- [ ] 「いつこのエージェントを使うべきか」が明確に読み取れる
- [ ] 役割名のみでなく具体的なユースケースを含む
- [ ] 1〜2文程度の適切な長さ

### ロールタイプと tools の整合性
- [ ] レビュー・確認・監査系の役割 → Write/Edit/Bash を含まない（リードオンリー型）
- [ ] 実装・構築・作成系の役割 → Write/Edit/Bash を含む（コードライター型）
- [ ] 調査・リサーチ系の役割 → WebFetch/WebSearch を含み Write/Bash がない（リサーチ型）

---

## body の品質チェック

- [ ] 冒頭にロール宣言（「あなたは〜です」）がある
- [ ] 役割に応じた具体的なチェックリスト・実施手順がある
- [ ] 参照するエージェント名が `.claude/agents/` に実際に存在する
- [ ] Claude が自明に知っている一般的なベストプラクティスを逐一指示していない
- [ ] CLAUDE.md のプロジェクト共通ルールを重複して記載していない

---

## 参照パスチェック

- [ ] body 内の参照パスがプロジェクトルート相対の完全パス
- [ ] ワイルドカード（`*.md` 等）を Read 参照に使用していない
- [ ] 参照先ファイルが実際に存在する（Glob で確認）

---

## ファイル配置確認

- [ ] ファイルパスが `.claude/agents/{name}.md` である
- [ ] ファイル名（拡張子を除く）と `name` フィールドが一致している

---

## 既存エージェント重複確認

新規作成前に以下を確認する:

| 既存エージェント | 役割 | 新規エージェントとの差異 |
|----------------|------|----------------------|
| backend-developer | バックエンド実装 | |
| frontend-developer | フロントエンド実装 | |
| code-reviewer | コードレビュー | |
| qa-expert | 品質保証・テスト | |
| security-auditor | セキュリティ監査 | |
| devops-engineer | インフラ・CI/CD | |
| database-architect | DBスキーマ設計 | |
| system-architect | システム設計 | |
| principal-engineer | アーキテクチャ検証 | |
| requirements-analyst | 要件定義 | |
| impact-analyzer | 影響範囲分析 | |
| integration-validator | 統合テスト検証 | |
| performance-engineer | パフォーマンス最適化 | |
| accessibility-checker | アクセシビリティ検証 | |
| release-manager | デプロイ・リリース管理 | |

上記15エージェントと明確に異なる役割でない場合は新規作成せず既存を利用する。

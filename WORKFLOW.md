# App Development Multi-Agent Workflow

## システム要件・基本方針

本ワークフローは、AIエージェントによる完全自律型のアプリケーション開発を実現するための定義である。

1. **完全自律駆動 (Fully Autonomous):** 人間は「作業者」や「コードレビュアー」として介入せず、高レベルな要約に基づく「最終意思決定者（承認ゲート）」としてのみ振る舞う。
2. **関心の分離 (Separation of Concerns):** 各Agentの役割（Focus）と権限（Skills）を厳密に分割し、プロンプトの肥大化とハルシネーションを抑制する。
3. **状態管理の外部化 (External State Management):** AIの内部メモリに依存せず、GitHub（Issue, Branch, PR）をSingle Source of Truth（唯一の真実）として進行状態とトレーサビリティを管理する。
4. **自律的品質保証 (Autonomous QA):** テスト、静的解析、セキュリティスキャン（SAST/SCA）、契約テストをシステム内部のフィードバックループとして完結させる。
5. **安全な自動デプロイ (Safe Deployment):** インフラのコード化（IaC）と、異常検知時の自動ロールバック機構を前提とする。

---

## 0. Orchestration & Communication

- **Agent**: `Orchestrator`
- **Role**: タスクのルーティング、並行処理の管理、フィードバックループの統括。自律稼働の監視と外部へのシグナル発火。
- **Focus**: システム全体の状態遷移を管理し、AIループの無限実行を防ぐ。各フェーズのゲートウェイとして機能し、必要なタイミングでのみ外部の承認権限者へサマリを通知する。
- **Skills**:
  - `emit-system-event`
  - `github-create-branch`
  - `github-create-draft-pr`

---

## 1. Requirement & Planning Phase

### 1A. 新規作成 (Greenfield)

#### Step 1A-1: 要件定義とドメインモデリング

- **Agent**: `Requirements Analyst`
- **Focus**: 入力された抽象的な要求から、システムが満たすべきユースケースの境界を定義し、主要なデータモデル（エンティティ）を過不足なく抽出する。
- **Skills**:
  - `extract-entities`
  - `define-usecases`

### 1B. 改修・機能追加 (Brownfield)

#### Step 1B-1: 影響調査 ＆ 実装計画 ＆ 【Issue起票】

- **Agent**: `Impact Analyzer`
- **Focus**: 既存コードの依存関係を解析し、変更による破壊的副作用を予測する。As-Is/To-Beの差分と回帰テスト方針を明確化し、後続Agentが実行可能なタスク単位（Issue）に分割してGitHubへ起票する。
- **Skills**:
  - `search-codebase`
  - `map-dependencies`
  - `github-create-issue`

#### Step 1B-2: データ移行設計 (Database Design Focus)

- **Agent**: `Database Architect`
- **Focus**: スキーマ変更が既存レコードに与える影響を評価し、安全なデータの退避、変換ロジック、およびロールバック可能なマイグレーション方針（設計）を策定する。
- **Skills**:
  - `analyze-schema-diff`
  - `plan-data-migration`

---

## 2. Architecture & Design Phase

#### Step 2-1: アーキテクチャ設計 ＆ 【ADR / Issue起票】

- **Agent**: `System Architect`
- **Knowledge Base**: 組織の標準設計原則（Vaultecture等のアーキテクチャ実装パターン）
- **Focus**: 技術スタック、正規化されたDBスキーマ、APIインターフェース（OpenAPI等）を決定し、その決定論理をADRとして明文化する。新規作成時は、本設計に基づき実装フェーズ用のIssueを分割・起票する。
- **Skills**:
  - `select-tech-stack`
  - `define-database-schema`
  - `generate-openapi-spec`
  - `write-adr-document`
  - `github-create-issue`

#### Step 2-2: UI/UX設計 ＆ デザインシステム

- **Agent**: `UI/UX Designer`
- **Focus**: ユーザージャーニーに基づく画面構成、再利用可能なUIコンポーネント、デザイントークンを定義し、システム全体の一貫性を担保する。
- **Skills**:
  - `create-wireframes`
  - `define-design-tokens`
  - `design-ui-components`

#### Step 2-3: クラウドインフラ設計

- **Agent**: `Cloud Architect`
- **Focus**: プロダクションレディなホスティング構成（ネットワーク、コンピューティング、ストレージ）を設計する。また、障害追跡のためのテレメトリ収集ルール（設計方針）を策定する。
- **Skills**:
  - `design-cloud-infrastructure`
  - `define-telemetry-rules`

---

## 3. Upstream Validation & Authorization

#### Step 3-1: 総合設計レビュー (AI Loop)

- **Agent**: `Principal Engineer`
- **Focus**: API仕様とUI設計間のデータ不整合、セキュリティ要件の欠落、インフラ構成の実現可能性を静的に検証する。矛盾検知時は自律的にStep 2へ差し戻す。
- **Skills**:
  - `validate-feasibility`
  - `check-api-ui-consistency`

#### Step 3-2: 計画承認ゲート (Executive Authorization Gate)

- **Agent**: `Orchestrator`
- **Focus**: 実装計画とADRをエグゼクティブ・サマリ（コスト、影響範囲、期待成果）に圧縮して提示する。外部の意思決定者から進行の承認シグナルを受信するまで後続処理をロックする。
- **Skills**:
  - `generate-executive-summary`
  - `request-authorization-signal`
  - `await-authorization`

---

## 4. Implementation Phase

※承認シグナル受信後、Orchestratorが作業ブランチとDraft PRを作成して開始。

#### Step 4-1: IaC ＆ CI/CD実装

- **Agent**: `DevOps Engineer`
- **Focus**: `Cloud Architect` の設計に基づき、Terraform等のインフラ定義コード（IaC）と、CI/CDパイプライン構成を実際に生成・構築する。
- **Skills**:
  - `generate-iac-files`
  - `generate-ci-cd-workflows`

#### Step 4-2: マイグレーション実装

- **Agent**: `Database Developer`
- **Focus**: `Database Architect` の設計に完全に準拠した、UP/DOWNマイグレーションスクリプトを実装する。
- **Skills**:
  - `write-migration-scripts`

#### Step 4-3: バックエンド・フロントエンド実装 (並行タスク)

- **Agent**: `Backend Developer` / `Frontend Developer`
- **Focus**: Issueの実装計画に沿ってロジックおよびUIを実装する。変更分を作業ブランチへ自律的にコミットする。
- **Skills**:
  - `read-file`
  - `write-file`
  - `github-commit-and-push`

#### Step 4-4: テストと自律的修正 (AI Feedback Loop)

- **Agent**: `QA Engineer`
- **Focus**: 境界値や異常系を含む網羅的なユニットテストを自動生成・実行する。失敗時はエラー原因を特定し、パスするまで実装Agentへの修正指示と再実行のループを回す。
- **Skills**:
  - `write-unit-tests`
  - `run-tests`
  - `read-error-log`

---

## 5. Expert Review Pipeline

#### Step 5-1: コード・アーキテクチャレビュー (AI Loop)

- **Agent**: `Peer Reviewer`
- **Focus**: PR差分のAST（抽象構文木）を解析し、コードの保守性、DRY原則の違反、アーキテクチャパターンからの逸脱を監査する。
- **Skills**:
  - `github-get-pr-diff`
  - `analyze-ast`
  - `github-add-pr-comment`

#### Step 5-2: パフォーマンス監査 (AI Loop)

- **Agent**: `Performance Engineer`
- **Focus**: 計算量の非効率性、N+1クエリ、無駄な再描画などのパフォーマンスボトルネックを静的解析によって特定する。
- **Skills**:
  - `github-get-pr-diff`
  - `analyze-time-complexity`
  - `github-add-pr-comment`

#### Step 5-3: セキュリティ ＆ 依存関係レビュー (AI Loop)

- **Agent**: `Security Auditor`
- **Focus**: SASTスキャンによる脆弱性検知、シークレットのハードコード検知、および依存パッケージの既知の脆弱性（SCA）を厳格に監査する。
- **Skills**:
  - `run-sast-scan`
  - `check-hardcoded-secrets`
  - `scan-dependencies`
  - `github-add-pr-comment`

#### Step 5-4: UI/UX & アクセシビリティレビュー (AI Loop)

- **Agent**: `Accessibility & UI Auditor`
- **Focus**: フロントエンド実装がWAI-ARIA基準を満たしているか、定義されたデザインシステムとの整合性を保っているかを検証する。
- **Skills**:
  - `check-a11y-compliance`
  - `github-add-pr-comment`

---

## 6. Build & Final Validation Phase

#### Step 6-1: 環境統合 ＆ 契約テスト (CI Pipeline)

- **Agent**: `Integration Validator`
- **Focus**: クリーンなCI環境でビルドとE2Eテストを実行する。実装されたAPIが設計フェーズで定義されたコントラクト（OpenAPI等）に完全に準拠しているかを機械的に検証する。
- **Skills**:
  - `trigger-ci-pipeline`
  - `run-contract-tests`
  - `get-ci-status`

#### Step 6-2: デプロイメント ＆ 自動ロールバック検証

- **Agent**: `Release Manager`
- **Focus**: ステージング環境等へデプロイとマイグレーションを実行する。致命的なエラーを検知した場合、即座にインフラとDBを安全な状態へ自動ロールバックし、障害シグナルを送信する。
- **Skills**:
  - `deploy-to-staging`
  - `verify-deployment-health`
  - `rollback-deployment`
  - `rollback-migration`

#### Step 6-3: リリース承認ゲート (Production Release Gate)

- **Agent**: `Orchestrator`
- **Focus**: レビュー通過、CIのオールグリーン、ヘルスチェック成功を要約し、リリースノートを生成する。プロダクションへのデプロイ権限者へ最終マージの承認シグナルを要求する。
- **Skills**:
  - `generate-release-notes`
  - `request-merge-authorization`

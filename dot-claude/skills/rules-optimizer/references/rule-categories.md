# ルールカテゴリ分類基準

## コードスタイル (`code-style.md`)

**含むもの**:

- インデント・スペース規約
- 命名規則（camelCase / PascalCase / snake_case）
- インポートの順序
- ファイル名規則
- コメント規約（WHYを書く等）
- 行長・フォーマット

**paths 例**:

```yaml
paths:
  - "src/**/*.ts"
  - "src/**/*.tsx"
  - "src/**/*.js"
  - "lib/**/*.ts"
```

---

## テスト (`testing.md`)

**含むもの**:

- テストファイルの命名（`*.test.ts`, `*.spec.ts`）
- テスト関数の命名（"should [action] when [condition]"）
- AAAパターン（Arrange, Act, Assert）
- モック・スタブの使い方
- カバレッジ要件
- E2Eテストの規約

**paths 例**:

```yaml
paths:
  - "**/*.test.ts"
  - "**/*.spec.ts"
  - "**/*.test.tsx"
  - "tests/**/*"
  - "e2e/**/*"
```

---

## セキュリティ (`security.md`)

プロジェクト固有のセキュリティ制約のみを含む。
Claude がデフォルトで行うこと（SQLi対策・入力バリデーション等の一般的セキュリティ実践）は対象外。

**含むもの（例）**:

- このプロジェクト固有の認証・認可フロー（例：「管理者APIは X-Admin-Token ヘッダー必須」）
- 特定テーブル・エンドポイントへのアクセス制限（例：「`payments` テーブルへの直接クエリ禁止」）
- プロジェクト固有のデータ分類ルール（例：「PII は logs/ に出力しない」）

**paths 例**:

```yaml
paths:
  - "src/auth/**/*"
  - "src/payments/**/*"
  - "src/middleware/**/*"
  - "src/api/**/*"
```

---

## フロントエンド (`frontend.md`)

**含むもの**:

- React/Vue/Svelte固有のパターン
- コンポーネント設計（関数コンポーネント優先等）
- カスタムフック・Composable の使い方
- 状態管理の規約
- CSS/スタイリング規約
- アクセシビリティ要件

**paths 例（React）**:

```yaml
paths:
  - "src/components/**/*.tsx"
  - "src/hooks/**/*.ts"
  - "src/pages/**/*.tsx"
  - "src/app/**/*.tsx"
```

**paths 例（Vue）**:

```yaml
paths:
  - "src/components/**/*.vue"
  - "src/composables/**/*.ts"
  - "src/views/**/*.vue"
```

---

## バックエンド (`backend.md`)

**含むもの**:

- APIエンドポイントの設計規約
- HTTPステータスコードの使い方
- エラーレスポンスフォーマット
- Correlation IDのログ出力
- OpenAPIドキュメント要件
- バリデーションライブラリの使い方（Zod等）

**paths 例**:

```yaml
paths:
  - "src/api/**/*.ts"
  - "src/routes/**/*.ts"
  - "src/controllers/**/*.ts"
  - "src/handlers/**/*.ts"
  - "app/**/*.py"
```

---

## データベース (`database.md`)

**含むもの**:

- マイグレーションファイルの直接変更禁止
- クエリ最適化の規約
- トランザクションの扱い
- N+1問題の防止
- Prisma/TypeORM/SQLAlchemy固有の規約

**paths 例**:

```yaml
paths:
  - "src/repositories/**/*.ts"
  - "src/db/**/*"
  - "prisma/**/*"
  - "migrations/**/*"
  - "src/models/**/*.py"
```

---

## Gitワークフロー (`git-workflow.md`)

**含むもの**:

- コミットメッセージ形式（Conventional Commits等）
- ブランチ命名規則
- PRのサイズ・説明要件
- main/masterへの直接pushの禁止

**paths**: 省略（常時適用）。特定ファイルに紐付かないルールのため。

```yaml
# pathsフロントマターを省略 → 全コンテキストで適用
```

## レビュープロセス (`review-process.md`)

**含むもの**:

- コード提出前の自己チェック手順（lint → test → 動作確認の順序等）
- CI/CDの必須通過条件
- セルフマージの可否

**paths**: 省略（常時適用）。Gitワークフロー同様、特定ファイルに紐付かないため。

---

## 分類できないもの（CLAUDE.mdに残す）

- プロジェクト概要・目的
- 技術スタック一覧
- 開発コマンド（npm run dev 等）
- 環境変数の説明（どの変数が何をするか）
- セキュリティ絶対禁止事項（`.env` の内容出力禁止等、全コンテキストで常時必要なもの）

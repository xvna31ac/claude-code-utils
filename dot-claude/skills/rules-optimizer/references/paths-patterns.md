# 技術スタック別 推奨 paths パターン

## 検出方法

`package.json` の `dependencies` / `devDependencies` を読んでスタックを判定する。
存在しない場合はディレクトリ構造（`src/`, `app/`, `lib/` 等）から推測する。

---

## TypeScript / JavaScript（共通）

```yaml
# TypeScriptルール全般
paths:
  - "src/**/*.ts"
  - "src/**/*.tsx"
  - "lib/**/*.ts"

# テスト
paths:
  - "**/*.test.ts"
  - "**/*.spec.ts"
  - "**/*.test.tsx"
  - "**/*.spec.tsx"
  - "__tests__/**/*"

# 設定ファイル（TS系）
paths:
  - "*.config.ts"
  - "*.config.js"
```

---

## React

```yaml
# コンポーネント・フック
paths:
  - "src/components/**/*.tsx"
  - "src/hooks/**/*.ts"
  - "src/pages/**/*.tsx"

# Next.js (App Router)
paths:
  - "app/**/*.tsx"
  - "app/**/*.ts"
  - "src/app/**/*.tsx"

# Next.js (Pages Router)
paths:
  - "pages/**/*.tsx"
  - "src/pages/**/*.tsx"
```

---

## Vue

```yaml
paths:
  - "src/components/**/*.vue"
  - "src/views/**/*.vue"
  - "src/composables/**/*.ts"
  - "src/stores/**/*.ts"
```

---

## Svelte / SvelteKit

```yaml
paths:
  - "src/**/*.svelte"
  - "src/routes/**/*"
  - "src/lib/**/*.ts"
```

---

## Node.js / Express / Fastify（バックエンド）

```yaml
# APIルート
paths:
  - "src/routes/**/*.ts"
  - "src/controllers/**/*.ts"
  - "src/handlers/**/*.ts"
  - "src/api/**/*.ts"

# ミドルウェア
paths:
  - "src/middleware/**/*.ts"

# サービス層
paths:
  - "src/services/**/*.ts"
  - "src/repositories/**/*.ts"
```

---

## Python（FastAPI / Django / Flask）

```yaml
# FastAPI
paths:
  - "app/routers/**/*.py"
  - "app/api/**/*.py"
  - "app/models/**/*.py"

# Django
paths:
  - "**/views.py"
  - "**/models.py"
  - "**/serializers.py"
  - "**/urls.py"

# 共通
paths:
  - "**/*.py"

# テスト
paths:
  - "tests/**/*.py"
  - "test_*.py"
  - "*_test.py"
```

---

## データベース

```yaml
# Prisma
paths:
  - "prisma/**/*"
  - "src/db/**/*.ts"

# TypeORM / Drizzle
paths:
  - "src/entities/**/*.ts"
  - "src/migrations/**/*"
  - "drizzle/**/*"

# SQLAlchemy
paths:
  - "alembic/**/*"
  - "app/models/**/*.py"
```

---

## セキュリティ（認証・決済）

```yaml
paths:
  - "src/auth/**/*"
  - "src/payments/**/*"
  - "src/billing/**/*"
  - "src/security/**/*"
  - "app/auth/**/*"
```

---

## インフラ・DevOps

```yaml
# Docker
paths:
  - "Dockerfile*"
  - "docker-compose*.yml"
  - ".docker/**/*"

# CI/CD
paths:
  - ".github/workflows/**/*"
  - ".gitlab-ci.yml"
  - "Jenkinsfile"

# Terraform
paths:
  - "**/*.tf"
  - "terraform/**/*"
```

---

## paths を省略するケース

Gitワークフロー・レビュープロセス等、特定ファイルに紐付かないプロジェクト固有ルールは
`paths:` フロントマターを省略して常時適用する。

```markdown
(フロントマターなし)

# Gitワークフロールール

- コミットメッセージは Conventional Commits 形式
- main ブランチへの直接 push 禁止
```

常時適用ルールが増えるとコンテキスト汚染につながるため、
本当に全コンテキストで必要なプロジェクト固有ルールのみに限定すること。

---
name: tdd-workflow
description: 新機能の実装・バグ修正・リファクタリング時に使用するスキル。テスト駆動開発（TDD）を強制し、ユニット・インテグレーション・E2Eを含む80%以上のカバレッジを確保する。
---

# テスト駆動開発ワークフロー

すべてのコード開発がTDD原則と十分なテストカバレッジに従うことを保証するスキル。

## 起動条件

- 新機能・新しい処理の実装
- バグ・不具合の修正
- 既存コードのリファクタリング
- APIエンドポイントの追加
- 新しいコンポーネントの作成

## コア原則

### 1. コードより先にテストを書く

必ずテストを先に書き、そのテストを通過するようにコードを実装する。

### 2. カバレッジ要件

- 最低80%のカバレッジ（ユニット＋インテグレーション＋E2E）
- すべてのエッジケースをカバー
- エラーシナリオのテスト
- 境界値の検証

### 3. テストの種類

#### ユニットテスト

- 個々の関数・ユーティリティ
- コンポーネントのロジック
- 純粋関数
- ヘルパー・ユーティリティ

#### インテグレーションテスト

- APIエンドポイント
- データベース操作
- サービス間の連携
- 外部API呼び出し

#### E2Eテスト（Playwright）

- 重要なユーザーフロー
- 完全なワークフロー
- ブラウザ自動化
- UI操作

## TDDワークフローのステップ

### Step 1: ユーザージャーニーを書く

```
[ロール]として、[目的]のために、[アクション]したい

例:
ユーザーとして、正確なキーワードがなくても関連するマーケットを見つけるために、
セマンティック検索でマーケットを探したい。
```

### Step 2: テストケースを生成する

各ユーザージャーニーに対して包括的なテストケースを作成する：

```typescript
describe('セマンティック検索', () => {
  it('クエリに対して関連するマーケットを返す', async () => {
    // テスト実装
  })

  it('空のクエリを適切に処理する', async () => {
    // エッジケースのテスト
  })

  it('Redisが利用不可の場合はサブストリング検索にフォールバックする', async () => {
    // フォールバック動作のテスト
  })

  it('類似度スコアで結果をソートする', async () => {
    // ソートロジックのテスト
  })
})
```

### Step 3: テストを実行する（失敗することを確認）

```bash
npm test
# テストは失敗するはず - まだ実装していないため
```

### Step 4: コードを実装する

テストを通過させるための最小限のコードを書く：

```typescript
// テストに導かれた実装
export async function searchMarkets(query: string) {
  // 実装
}
```

### Step 5: 再度テストを実行する

```bash
npm test
# テストが通過するはず
```

### Step 6: リファクタリング

テストをグリーンに保ちながらコード品質を改善する：

- 重複の除去
- 命名の改善
- パフォーマンスの最適化
- 可読性の向上

### Step 7: カバレッジを確認する

```bash
npm run test:coverage
# 80%以上のカバレッジを確認
```

## テストのパターン

### ユニットテストのパターン（Jest/Vitest）

```typescript
import { render, screen, fireEvent } from '@testing-library/react'
import { Button } from './Button'

describe('Buttonコンポーネント', () => {
  it('正しいテキストでレンダリングされる', () => {
    render(<Button>クリック</Button>)
    expect(screen.getByText('クリック')).toBeInTheDocument()
  })

  it('クリック時にonClickが呼ばれる', () => {
    const handleClick = jest.fn()
    render(<Button onClick={handleClick}>クリック</Button>)

    fireEvent.click(screen.getByRole('button'))

    expect(handleClick).toHaveBeenCalledTimes(1)
  })

  it('disabledがtrueのとき無効化される', () => {
    render(<Button disabled>クリック</Button>)
    expect(screen.getByRole('button')).toBeDisabled()
  })
})
```

### APIインテグレーションテストのパターン

```typescript
import { NextRequest } from 'next/server'
import { GET } from './route'

describe('GET /api/markets', () => {
  it('マーケット一覧を正常に返す', async () => {
    const request = new NextRequest('http://localhost/api/markets')
    const response = await GET(request)
    const data = await response.json()

    expect(response.status).toBe(200)
    expect(data.success).toBe(true)
    expect(Array.isArray(data.data)).toBe(true)
  })

  it('クエリパラメータを検証する', async () => {
    const request = new NextRequest('http://localhost/api/markets?limit=invalid')
    const response = await GET(request)

    expect(response.status).toBe(400)
  })

  it('データベースエラーを適切に処理する', async () => {
    // DBエラーのモック
    const request = new NextRequest('http://localhost/api/markets')
    // エラーハンドリングのテスト
  })
})
```

### E2Eテストのパターン（Playwright）

```typescript
import { test, expect } from '@playwright/test'

test('ユーザーがマーケットを検索・フィルタリングできる', async ({ page }) => {
  await page.goto('/')
  await page.click('a[href="/markets"]')

  await expect(page.locator('h1')).toContainText('Markets')

  await page.fill('input[placeholder="Search markets"]', 'election')

  await page.waitForTimeout(600)

  const results = page.locator('[data-testid="market-card"]')
  await expect(results).toHaveCount(5, { timeout: 5000 })

  const firstResult = results.first()
  await expect(firstResult).toContainText('election', { ignoreCase: true })

  await page.click('button:has-text("Active")')

  await expect(results).toHaveCount(3)
})

test('ユーザーが新しいマーケットを作成できる', async ({ page }) => {
  await page.goto('/creator-dashboard')

  await page.fill('input[name="name"]', 'テストマーケット')
  await page.fill('textarea[name="description"]', 'テスト説明')
  await page.fill('input[name="endDate"]', '2025-12-31')

  await page.click('button[type="submit"]')

  await expect(page.locator('text=Market created successfully')).toBeVisible()

  await expect(page).toHaveURL(/\/markets\/test-market/)
})
```

## テストファイルの構成

```
src/
├── components/
│   ├── Button/
│   │   ├── Button.tsx
│   │   ├── Button.test.tsx          # ユニットテスト
│   │   └── Button.stories.tsx       # Storybook
│   └── MarketCard/
│       ├── MarketCard.tsx
│       └── MarketCard.test.tsx
├── app/
│   └── api/
│       └── markets/
│           ├── route.ts
│           └── route.test.ts         # インテグレーションテスト
└── e2e/
    ├── markets.spec.ts               # E2Eテスト
    ├── trading.spec.ts
    └── auth.spec.ts
```

## 外部サービスのモック

### Supabaseモック

```typescript
jest.mock('@/lib/supabase', () => ({
  supabase: {
    from: jest.fn(() => ({
      select: jest.fn(() => ({
        eq: jest.fn(() => Promise.resolve({
          data: [{ id: 1, name: 'テストマーケット' }],
          error: null
        }))
      }))
    }))
  }
}))
```

### Redisモック

```typescript
jest.mock('@/lib/redis', () => ({
  searchMarketsByVector: jest.fn(() => Promise.resolve([
    { slug: 'test-market', similarity_score: 0.95 }
  ])),
  checkRedisHealth: jest.fn(() => Promise.resolve({ connected: true }))
}))
```

### OpenAIモック

```typescript
jest.mock('@/lib/openai', () => ({
  generateEmbedding: jest.fn(() => Promise.resolve(
    new Array(1536).fill(0.1) // 1536次元の埋め込みベクトルのモック
  ))
}))
```

## テストカバレッジの確認

### カバレッジレポートの実行

```bash
npm run test:coverage
```

### カバレッジのしきい値設定

```json
{
  "jest": {
    "coverageThresholds": {
      "global": {
        "branches": 80,
        "functions": 80,
        "lines": 80,
        "statements": 80
      }
    }
  }
}
```

## よくある間違いと正しいアプローチ

### ❌ NG: 実装の詳細をテストする

```typescript
// 内部状態をテストしない
expect(component.state.count).toBe(5)
```

### ✅ OK: ユーザーから見える振る舞いをテストする

```typescript
// ユーザーが見るものをテストする
expect(screen.getByText('Count: 5')).toBeInTheDocument()
```

### ❌ NG: 壊れやすいセレクター

```typescript
// CSSクラスに依存すると変更に弱い
await page.click('.css-class-xyz')
```

### ✅ OK: セマンティックなセレクター

```typescript
// 変更に強い
await page.click('button:has-text("送信")')
await page.click('[data-testid="submit-button"]')
```

### ❌ NG: テスト間の依存

```typescript
// テストが互いに依存している
test('ユーザーを作成する', () => { /* ... */ })
test('同じユーザーを更新する', () => { /* 前のテストに依存 */ })
```

### ✅ OK: 独立したテスト

```typescript
// 各テストが自前でデータを用意する
test('ユーザーを作成する', () => {
  const user = createTestUser()
  // テストロジック
})

test('ユーザーを更新する', () => {
  const user = createTestUser()
  // 更新ロジック
})
```

## 継続的なテスト

### 開発中のウォッチモード

```bash
npm test -- --watch
# ファイル変更時に自動でテストが実行される
```

### コミット前フック

```bash
# コミットのたびに実行される
npm test && npm run lint
```

### CI/CD連携

```yaml
# GitHub Actions
- name: テストの実行
  run: npm test -- --coverage
- name: カバレッジのアップロード
  uses: codecov/codecov-action@v3
```

## ベストプラクティス

1. **テストファースト** - 常にTDDで進める
2. **1テスト1アサーション** - 単一の振る舞いにフォーカスする
3. **わかりやすいテスト名** - 何をテストしているかを説明する
4. **Arrange-Act-Assert** - 明確なテスト構造を保つ
5. **外部依存のモック化** - ユニットテストを分離する
6. **エッジケースのテスト** - null・undefined・空・大量データ
7. **エラーパスのテスト** - 正常系だけでなく異常系も
8. **テストを速く保つ** - ユニットテストは1件50ms以内
9. **テスト後のクリーンアップ** - 副作用を残さない
10. **カバレッジレポートのレビュー** - テストされていない箇所を把握する

## 完了の基準

- コードカバレッジ80%以上
- 全テスト通過（グリーン）
- スキップ・無効化されたテストなし
- テスト実行が高速（ユニットテストは30秒以内）
- E2Eテストが重要なユーザーフローをカバー
- テストが本番前にバグを検出できている

---

**テストは省略可能なものではない。テストこそが、自信を持ったリファクタリング・迅速な開発・本番環境の信頼性を支える安全網である。**

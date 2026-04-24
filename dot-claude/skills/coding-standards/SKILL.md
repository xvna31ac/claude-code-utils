---
name: coding-standards
description: TypeScript・JavaScript・React・Node.js 開発におけるコーディング規約、ベストプラクティス、設計パターンの普遍的な標準。
---

# コーディング規約とベストプラクティス

すべてのプロジェクトに適用できる普遍的なコーディング規約。

## 適用タイミング

- 新しいプロジェクトやモジュールを開始するとき
- コードの品質と保守性をレビューするとき
- 既存のコードを規約に沿ってリファクタリングするとき
- 命名・フォーマット・構造の一貫性を強制するとき
- Lint・フォーマット・型チェックのルールを設定するとき
- 新しいコントリビューターにコーディング規約を説明するとき

## コード品質の原則

### 1. 可読性第一

- コードは書く回数より読む回数の方が多い
- 変数名・関数名は明確に
- コメントよりも自己説明的なコードを優先
- 一貫したフォーマット

### 2. KISS（シンプルに保て）

- 動く中で最もシンプルな解決策を選ぶ
- 過剰な設計を避ける
- 早すぎる最適化をしない
- 理解しやすさ ＞ 巧妙なコード

### 3. DRY（繰り返すな）

- 共通ロジックは関数に切り出す
- 再利用可能なコンポーネントを作る
- ユーティリティはモジュール間で共有する
- コピー＆ペーストプログラミングを避ける

### 4. YAGNI（今必要でないものは作るな）

- 必要になる前に機能を作り込まない
- 投機的な汎用化を避ける
- 複雑さは必要になってから追加する
- シンプルに始めて、必要になったらリファクタリングする

## TypeScript / JavaScript 規約

### 変数の命名

```typescript
// ✅ 良い例: 説明的な名前
const marketSearchQuery = 'election'
const isUserAuthenticated = true
const totalRevenue = 1000

// ❌ 悪い例: 不明瞭な名前
const q = 'election'
const flag = true
const x = 1000
```

### 関数の命名

```typescript
// ✅ 良い例: 動詞＋名詞パターン
async function fetchMarketData(marketId: string) { }
function calculateSimilarity(a: number[], b: number[]) { }
function isValidEmail(email: string): boolean { }

// ❌ 悪い例: 不明瞭または名詞のみ
async function market(id: string) { }
function similarity(a, b) { }
function email(e) { }
```

### イミュータビリティパターン（重要）

```typescript
// ✅ 常にスプレッド演算子を使う
const updatedUser = {
  ...user,
  name: 'New Name'
}

const updatedArray = [...items, newItem]

// ❌ 直接変更してはいけない
user.name = 'New Name'  // 悪い例
items.push(newItem)     // 悪い例
```

### エラーハンドリング

```typescript
// ✅ 良い例: 包括的なエラーハンドリング
async function fetchData(url: string) {
  try {
    const response = await fetch(url)

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`)
    }

    return await response.json()
  } catch (error) {
    console.error('Fetch failed:', error)
    throw new Error('Failed to fetch data')
  }
}

// ❌ 悪い例: エラーハンドリングなし
async function fetchData(url) {
  const response = await fetch(url)
  return response.json()
}
```

### Async/Await のベストプラクティス

```typescript
// ✅ 良い例: 可能な場合は並列実行
const [users, markets, stats] = await Promise.all([
  fetchUsers(),
  fetchMarkets(),
  fetchStats()
])

// ❌ 悪い例: 不要な逐次実行
const users = await fetchUsers()
const markets = await fetchMarkets()
const stats = await fetchStats()
```

### 型安全性

```typescript
// ✅ 良い例: 適切な型定義
interface Market {
  id: string
  name: string
  status: 'active' | 'resolved' | 'closed'
  created_at: Date
}

function getMarket(id: string): Promise<Market> {
  // 実装
}

// ❌ 悪い例: 'any' の使用
function getMarket(id: any): Promise<any> {
  // 実装
}
```

## React ベストプラクティス

### コンポーネントの構造

```typescript
// ✅ 良い例: 型付き関数コンポーネント
interface ButtonProps {
  children: React.ReactNode
  onClick: () => void
  disabled?: boolean
  variant?: 'primary' | 'secondary'
}

export function Button({
  children,
  onClick,
  disabled = false,
  variant = 'primary'
}: ButtonProps) {
  return (
    <button
      onClick={onClick}
      disabled={disabled}
      className={`btn btn-${variant}`}
    >
      {children}
    </button>
  )
}

// ❌ 悪い例: 型なし・構造が不明瞭
export function Button(props) {
  return <button onClick={props.onClick}>{props.children}</button>
}
```

### カスタムフック

```typescript
// ✅ 良い例: 再利用可能なカスタムフック
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value)
    }, delay)

    return () => clearTimeout(handler)
  }, [value, delay])

  return debouncedValue
}

// 使用例
const debouncedQuery = useDebounce(searchQuery, 500)
```

### 状態管理

```typescript
// ✅ 良い例: 適切な状態更新
const [count, setCount] = useState(0)

// 直前の状態に基づく更新には関数型アップデートを使う
setCount(prev => prev + 1)

// ❌ 悪い例: 状態を直接参照
setCount(count + 1)  // 非同期シナリオで古い値を参照する可能性がある
```

### 条件付きレンダリング

```typescript
// ✅ 良い例: 明確な条件付きレンダリング
{isLoading && <Spinner />}
{error && <ErrorMessage error={error} />}
{data && <DataDisplay data={data} />}

// ❌ 悪い例: ネストしすぎた三項演算子
{isLoading ? <Spinner /> : error ? <ErrorMessage error={error} /> : data ? <DataDisplay data={data} /> : null}
```

## API 設計規約

### REST API の規則

```
GET    /api/markets              # 一覧取得
GET    /api/markets/:id          # 特定リソース取得
POST   /api/markets              # 新規作成
PUT    /api/markets/:id          # 全体更新
PATCH  /api/markets/:id          # 部分更新
DELETE /api/markets/:id          # 削除

# フィルタリングにはクエリパラメータを使う
GET /api/markets?status=active&limit=10&offset=0
```

### レスポンス形式

```typescript
// ✅ 良い例: 一貫したレスポンス構造
interface ApiResponse<T> {
  success: boolean
  data?: T
  error?: string
  meta?: {
    total: number
    page: number
    limit: number
  }
}

// 成功レスポンス
return NextResponse.json({
  success: true,
  data: markets,
  meta: { total: 100, page: 1, limit: 10 }
})

// エラーレスポンス
return NextResponse.json({
  success: false,
  error: 'Invalid request'
}, { status: 400 })
```

### 入力バリデーション

```typescript
import { z } from 'zod'

// ✅ 良い例: スキーマバリデーション
const CreateMarketSchema = z.object({
  name: z.string().min(1).max(200),
  description: z.string().min(1).max(2000),
  endDate: z.string().datetime(),
  categories: z.array(z.string()).min(1)
})

export async function POST(request: Request) {
  const body = await request.json()

  try {
    const validated = CreateMarketSchema.parse(body)
    // バリデーション済みデータで処理を進める
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json({
        success: false,
        error: 'Validation failed',
        details: error.errors
      }, { status: 400 })
    }
  }
}
```

## ファイル構成

### プロジェクト構造

```
src/
├── app/                    # Next.js App Router
│   ├── api/               # API ルート
│   ├── markets/           # マーケットページ
│   └── (auth)/           # 認証ページ（ルートグループ）
├── components/            # React コンポーネント
│   ├── ui/               # 汎用 UI コンポーネント
│   ├── forms/            # フォームコンポーネント
│   └── layouts/          # レイアウトコンポーネント
├── hooks/                # カスタム React フック
├── lib/                  # ユーティリティと設定
│   ├── api/             # API クライアント
│   ├── utils/           # ヘルパー関数
│   └── constants/       # 定数
├── types/                # TypeScript 型定義
└── styles/              # グローバルスタイル
```

### ファイルの命名規則

```
components/Button.tsx          # コンポーネントは PascalCase
hooks/useAuth.ts              # フックは 'use' プレフィックス付き camelCase
lib/formatDate.ts             # ユーティリティは camelCase
types/market.types.ts         # 型定義は .types サフィックス付き camelCase
```

## コメントとドキュメント

### コメントを書くべき場面

```typescript
// ✅ 良い例: 「なぜ」を説明する（「何を」ではなく）
// 障害時に API を圧迫しないよう指数バックオフを使用
const delay = Math.min(1000 * Math.pow(2, retryCount), 30000)

// 大きな配列のパフォーマンスのために意図的にミュータブル操作を使用
items.push(newItem)

// ❌ 悪い例: 明白なことを述べるだけ
// カウンターを 1 増やす
count++

// 名前をユーザーの名前にセットする
name = user.name
```

### 公開 API の JSDoc

```typescript
/**
 * セマンティック類似度でマーケットを検索する。
 *
 * @param query - 自然言語の検索クエリ
 * @param limit - 最大結果件数（デフォルト: 10）
 * @returns 類似度スコア順のマーケット配列
 * @throws {Error} OpenAI API 失敗時または Redis 利用不可時
 *
 * @example
 * ```typescript
 * const results = await searchMarkets('election', 5)
 * console.log(results[0].name) // "Trump vs Biden"
 * ```
 */
export async function searchMarkets(
  query: string,
  limit: number = 10
): Promise<Market[]> {
  // 実装
}
```

## パフォーマンスのベストプラクティス

### メモ化

```typescript
import { useMemo, useCallback } from 'react'

// ✅ 良い例: 重い計算をメモ化
const sortedMarkets = useMemo(() => {
  return markets.sort((a, b) => b.volume - a.volume)
}, [markets])

// ✅ 良い例: コールバックをメモ化
const handleSearch = useCallback((query: string) => {
  setSearchQuery(query)
}, [])
```

### 遅延読み込み

```typescript
import { lazy, Suspense } from 'react'

// ✅ 良い例: 重いコンポーネントを遅延読み込み
const HeavyChart = lazy(() => import('./HeavyChart'))

export function Dashboard() {
  return (
    <Suspense fallback={<Spinner />}>
      <HeavyChart />
    </Suspense>
  )
}
```

### データベースクエリ

```typescript
// ✅ 良い例: 必要なカラムだけ取得
const { data } = await supabase
  .from('markets')
  .select('id, name, status')
  .limit(10)

// ❌ 悪い例: 全カラムを取得
const { data } = await supabase
  .from('markets')
  .select('*')
```

## テスト規約

### テスト構造（AAA パターン）

```typescript
test('類似度を正しく計算する', () => {
  // Arrange（準備）
  const vector1 = [1, 0, 0]
  const vector2 = [0, 1, 0]

  // Act（実行）
  const similarity = calculateCosineSimilarity(vector1, vector2)

  // Assert（検証）
  expect(similarity).toBe(0)
})
```

### テストの命名

```typescript
// ✅ 良い例: 説明的なテスト名
test('クエリに一致するマーケットがない場合は空配列を返す', () => { })
test('OpenAI の API キーが未設定の場合はエラーをスローする', () => { })
test('Redis が利用できない場合はサブストリング検索にフォールバックする', () => { })

// ❌ 悪い例: 曖昧なテスト名
test('works', () => { })
test('test search', () => { })
```

## コードの悪臭（アンチパターン）検出

以下のアンチパターンに注意:

### 1. 長すぎる関数

```typescript
// ❌ 悪い例: 50 行超の関数
function processMarketData() {
  // 100 行のコード
}

// ✅ 良い例: 小さな関数に分割
function processMarketData() {
  const validated = validateData()
  const transformed = transformData(validated)
  return saveData(transformed)
}
```

### 2. 深すぎるネスト

```typescript
// ❌ 悪い例: 5 段階以上のネスト
if (user) {
  if (user.isAdmin) {
    if (market) {
      if (market.isActive) {
        if (hasPermission) {
          // 処理
        }
      }
    }
  }
}

// ✅ 良い例: 早期リターン
if (!user) return
if (!user.isAdmin) return
if (!market) return
if (!market.isActive) return
if (!hasPermission) return

// 処理
```

### 3. マジックナンバー

```typescript
// ❌ 悪い例: 説明のない数値
if (retryCount > 3) { }
setTimeout(callback, 500)

// ✅ 良い例: 名前付き定数
const MAX_RETRIES = 3
const DEBOUNCE_DELAY_MS = 500

if (retryCount > MAX_RETRIES) { }
setTimeout(callback, DEBOUNCE_DELAY_MS)
```

**覚えておくこと**: コード品質に妥協はない。明確で保守しやすいコードこそが、迅速な開発と自信を持ったリファクタリングを可能にする。

---
name: security-review
description: 認証・ユーザー入力・シークレット・APIエンドポイント・決済・機密データを扱うコードを実装する際に使用するスキル。包括的なセキュリティチェックリストとパターンを提供する。
---

# セキュリティレビュースキル

すべてのコードがセキュリティベストプラクティスに従っていることを確認し、潜在的な脆弱性を特定するスキル。

## 起動条件

- 認証・認可の実装
- ユーザー入力・ファイルアップロードの処理
- 新しいAPIエンドポイントの作成
- シークレット・認証情報の取り扱い
- 決済機能の実装
- 機密データの保存・送信
- サードパーティAPIの連携

## セキュリティチェックリスト

### 1. シークレット管理

#### ❌ NG: やってはいけないこと

```typescript
const apiKey = "sk-proj-xxxxx"  // ハードコードされたシークレット
const dbPassword = "password123" // ソースコードに直書き
```

#### ✅ OK: 正しいやり方

```typescript
const apiKey = process.env.OPENAI_API_KEY
const dbUrl = process.env.DATABASE_URL

// シークレットの存在を確認する
if (!apiKey) {
  throw new Error('OPENAI_API_KEY が設定されていません')
}
```

#### 確認項目

- [ ] APIキー・トークン・パスワードのハードコードなし
- [ ] すべてのシークレットを環境変数で管理
- [ ] `.env.local` を .gitignore に追加済み
- [ ] gitの履歴にシークレットが含まれていない
- [ ] 本番環境のシークレットはホスティングプラットフォーム（Vercel、Railway）で管理

### 2. 入力バリデーション

#### ユーザー入力は必ず検証する

```typescript
import { z } from 'zod'

// バリデーションスキーマを定義する
const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  age: z.number().int().min(0).max(150)
})

// 処理前に検証する
export async function createUser(input: unknown) {
  try {
    const validated = CreateUserSchema.parse(input)
    return await db.users.create(validated)
  } catch (error) {
    if (error instanceof z.ZodError) {
      return { success: false, errors: error.errors }
    }
    throw error
  }
}
```

#### ファイルアップロードのバリデーション

```typescript
function validateFileUpload(file: File) {
  // サイズチェック（最大5MB）
  const maxSize = 5 * 1024 * 1024
  if (file.size > maxSize) {
    throw new Error('ファイルサイズが大きすぎます（最大5MB）')
  }

  // MIMEタイプチェック
  const allowedTypes = ['image/jpeg', 'image/png', 'image/gif']
  if (!allowedTypes.includes(file.type)) {
    throw new Error('許可されていないファイル形式です')
  }

  // 拡張子チェック
  const allowedExtensions = ['.jpg', '.jpeg', '.png', '.gif']
  const extension = file.name.toLowerCase().match(/\.[^.]+$/)?.[0]
  if (!extension || !allowedExtensions.includes(extension)) {
    throw new Error('許可されていない拡張子です')
  }

  return true
}
```

#### 確認項目

- [ ] すべてのユーザー入力をスキーマで検証
- [ ] ファイルアップロードを制限（サイズ・タイプ・拡張子）
- [ ] ユーザー入力をクエリに直接使用していない
- [ ] ブラックリストではなくホワイトリストで検証
- [ ] エラーメッセージに機密情報を含めない

### 3. SQLインジェクション対策

#### ❌ NG: SQLを文字列結合しない

```typescript
// 危険 - SQLインジェクション脆弱性
const query = `SELECT * FROM users WHERE email = '${userEmail}'`
await db.query(query)
```

#### ✅ OK: パラメータ化クエリを使用する

```typescript
// 安全 - パラメータ化クエリ
const { data } = await supabase
  .from('users')
  .select('*')
  .eq('email', userEmail)

// 生SQLの場合
await db.query(
  'SELECT * FROM users WHERE email = $1',
  [userEmail]
)
```

#### 確認項目

- [ ] すべてのDBクエリがパラメータ化されている
- [ ] SQLへの文字列結合なし
- [ ] ORM・クエリビルダーを正しく使用している
- [ ] Supabaseクエリが適切にサニタイズされている

### 4. 認証・認可

#### JWTトークンの取り扱い

```typescript
// ❌ NG: localStorage（XSSに脆弱）
localStorage.setItem('token', token)

// ✅ OK: httpOnlyクッキー
res.setHeader('Set-Cookie',
  `token=${token}; HttpOnly; Secure; SameSite=Strict; Max-Age=3600`)
```

#### 認可チェック

```typescript
export async function deleteUser(userId: string, requesterId: string) {
  // 必ず最初に認可を確認する
  const requester = await db.users.findUnique({
    where: { id: requesterId }
  })

  if (requester.role !== 'admin') {
    return NextResponse.json(
      { error: '権限がありません' },
      { status: 403 }
    )
  }

  await db.users.delete({ where: { id: userId } })
}
```

#### 行レベルセキュリティ（Supabase）

```sql
-- すべてのテーブルでRLSを有効化する
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- ユーザーは自分のデータのみ参照可能
CREATE POLICY "自分のデータのみ参照"
  ON users FOR SELECT
  USING (auth.uid() = id);

-- ユーザーは自分のデータのみ更新可能
CREATE POLICY "自分のデータのみ更新"
  ON users FOR UPDATE
  USING (auth.uid() = id);
```

#### 確認項目

- [ ] トークンをhttpOnlyクッキーに保存（localeStorageは使わない）
- [ ] 機密操作の前に認可チェックを実施
- [ ] SupabaseでRow Level Securityを有効化
- [ ] ロールベースのアクセス制御を実装
- [ ] セッション管理が安全

### 5. XSS対策

#### HTMLのサニタイズ

```typescript
import DOMPurify from 'isomorphic-dompurify'

// ユーザー提供のHTMLは必ずサニタイズする
function renderUserContent(html: string) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p'],
    ALLOWED_ATTR: []
  })
  return <div dangerouslySetInnerHTML={{ __html: clean }} />
}
```

#### コンテンツセキュリティポリシー（CSP）

```typescript
// next.config.js
const securityHeaders = [
  {
    key: 'Content-Security-Policy',
    value: `
      default-src 'self';
      script-src 'self' 'unsafe-eval' 'unsafe-inline';
      style-src 'self' 'unsafe-inline';
      img-src 'self' data: https:;
      font-src 'self';
      connect-src 'self' https://api.example.com;
    `.replace(/\s{2,}/g, ' ').trim()
  }
]
```

#### 確認項目

- [ ] ユーザー提供のHTMLをサニタイズ済み
- [ ] CSPヘッダーを設定済み
- [ ] 未検証の動的コンテンツのレンダリングなし
- [ ] ReactのビルトインXSS対策を活用している

### 6. CSRF対策

#### CSRFトークン

```typescript
import { csrf } from '@/lib/csrf'

export async function POST(request: Request) {
  const token = request.headers.get('X-CSRF-Token')

  if (!csrf.verify(token)) {
    return NextResponse.json(
      { error: 'CSRFトークンが無効です' },
      { status: 403 }
    )
  }

  // リクエストを処理する
}
```

#### SameSiteクッキー

```typescript
res.setHeader('Set-Cookie',
  `session=${sessionId}; HttpOnly; Secure; SameSite=Strict`)
```

#### 確認項目

- [ ] 状態変更操作にCSRFトークンを設定
- [ ] すべてのクッキーに SameSite=Strict を設定
- [ ] ダブルサブミットクッキーパターンを実装

### 7. レートリミット

#### APIのレートリミット

```typescript
import rateLimit from 'express-rate-limit'

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15分
  max: 100, // 15分あたり100リクエスト
  message: 'リクエストが多すぎます'
})

app.use('/api/', limiter)
```

#### 負荷の高い操作

```typescript
// 検索など負荷の高い操作は厳しく制限する
const searchLimiter = rateLimit({
  windowMs: 60 * 1000, // 1分
  max: 10, // 1分あたり10リクエスト
  message: '検索リクエストが多すぎます'
})

app.use('/api/search', searchLimiter)
```

#### 確認項目

- [ ] すべてのAPIエンドポイントにレートリミットを設定
- [ ] 負荷の高い操作には厳しい制限を設定
- [ ] IPベースのレートリミットを実装
- [ ] 認証済みユーザーベースのレートリミットを実装

### 8. 機密データの露出防止

#### ログ出力

```typescript
// ❌ NG: 機密データをログに記録する
console.log('ユーザーログイン:', { email, password })
console.log('支払い:', { cardNumber, cvv })

// ✅ OK: 機密データをマスクする
console.log('ユーザーログイン:', { email, userId })
console.log('支払い:', { last4: card.last4, userId })
```

#### エラーメッセージ

```typescript
// ❌ NG: 内部情報を露出する
catch (error) {
  return NextResponse.json(
    { error: error.message, stack: error.stack },
    { status: 500 }
  )
}

// ✅ OK: 汎用的なエラーメッセージを返す
catch (error) {
  console.error('内部エラー:', error)
  return NextResponse.json(
    { error: 'エラーが発生しました。しばらく経ってから再試行してください。' },
    { status: 500 }
  )
}
```

#### 確認項目

- [ ] パスワード・トークン・シークレットをログに記録していない
- [ ] ユーザー向けエラーメッセージは汎用的
- [ ] 詳細なエラーはサーバーログのみに記録
- [ ] スタックトレースをユーザーに露出していない

### 9. ブロックチェーンセキュリティ（Solana）

#### ウォレット署名の検証

```typescript
import { verify } from '@solana/web3.js'

async function verifyWalletOwnership(
  publicKey: string,
  signature: string,
  message: string
) {
  try {
    const isValid = verify(
      Buffer.from(message),
      Buffer.from(signature, 'base64'),
      Buffer.from(publicKey, 'base64')
    )
    return isValid
  } catch (error) {
    return false
  }
}
```

#### トランザクションの検証

```typescript
async function verifyTransaction(transaction: Transaction) {
  // 送金先を検証する
  if (transaction.to !== expectedRecipient) {
    throw new Error('送金先が無効です')
  }

  // 金額を検証する
  if (transaction.amount > maxAmount) {
    throw new Error('金額が上限を超えています')
  }

  // 残高を確認する
  const balance = await getBalance(transaction.from)
  if (balance < transaction.amount) {
    throw new Error('残高が不足しています')
  }

  return true
}
```

#### 確認項目

- [ ] ウォレット署名を検証済み
- [ ] トランザクションの詳細を検証済み
- [ ] トランザクション前に残高チェックを実施
- [ ] 内容確認なしのトランザクション署名なし

### 10. 依存関係のセキュリティ

#### 定期的な更新

```bash
# 脆弱性を確認する
npm audit

# 自動修正可能な問題を修正する
npm audit fix

# 依存関係を更新する
npm update

# 古いパッケージを確認する
npm outdated
```

#### ロックファイル

```bash
# ロックファイルは必ずコミットする
git add package-lock.json

# CI/CDでは再現性のあるビルドのために npm ci を使用する
npm ci  # npm install の代わりに使用
```

#### 確認項目

- [ ] 依存関係が最新
- [ ] 既知の脆弱性なし（npm audit クリーン）
- [ ] ロックファイルをコミット済み
- [ ] GitHubでDependabotを有効化
- [ ] 定期的なセキュリティ更新を実施

## セキュリティテスト

### 自動セキュリティテスト

```typescript
// 認証テスト
test('認証が必要', async () => {
  const response = await fetch('/api/protected')
  expect(response.status).toBe(401)
})

// 認可テスト
test('管理者ロールが必要', async () => {
  const response = await fetch('/api/admin', {
    headers: { Authorization: `Bearer ${userToken}` }
  })
  expect(response.status).toBe(403)
})

// 入力バリデーションテスト
test('無効な入力を拒否する', async () => {
  const response = await fetch('/api/users', {
    method: 'POST',
    body: JSON.stringify({ email: 'メールアドレスではない' })
  })
  expect(response.status).toBe(400)
})

// レートリミットテスト
test('レートリミットを適用する', async () => {
  const requests = Array(101).fill(null).map(() =>
    fetch('/api/endpoint')
  )

  const responses = await Promise.all(requests)
  const tooManyRequests = responses.filter(r => r.status === 429)

  expect(tooManyRequests.length).toBeGreaterThan(0)
})
```

## 本番デプロイ前チェックリスト

本番デプロイの前に必ず確認する：

- [ ] **シークレット**: ハードコードなし・すべて環境変数で管理
- [ ] **入力バリデーション**: すべてのユーザー入力を検証済み
- [ ] **SQLインジェクション**: すべてのクエリをパラメータ化済み
- [ ] **XSS**: ユーザーコンテンツをサニタイズ済み
- [ ] **CSRF**: 保護を有効化済み
- [ ] **認証**: トークンを適切に処理している
- [ ] **認可**: ロールチェックを実装済み
- [ ] **レートリミット**: すべてのエンドポイントに設定済み
- [ ] **HTTPS**: 本番環境で強制している
- [ ] **セキュリティヘッダー**: CSP・X-Frame-Options を設定済み
- [ ] **エラーハンドリング**: エラーに機密データを含めていない
- [ ] **ログ**: 機密データをログに記録していない
- [ ] **依存関係**: 最新・脆弱性なし
- [ ] **行レベルセキュリティ**: Supabaseで有効化済み
- [ ] **CORS**: 適切に設定済み
- [ ] **ファイルアップロード**: 検証済み（サイズ・タイプ）
- [ ] **ウォレット署名**: 検証済み（ブロックチェーン使用時）

## 参考リソース

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Next.js Security](https://nextjs.org/docs/security)
- [Supabase Security](https://supabase.com/docs/guides/auth)
- [Web Security Academy](https://portswigger.net/web-security)

---

**セキュリティは省略可能なものではない。ひとつの脆弱性がプラットフォーム全体を危険にさらす。迷ったときは、より慎重な側を選ぶこと。**

---
description: テスト駆動開発ワークフローを強制する。インターフェースをスキャフォールドし、テストを先に生成してから最小限のコードを実装する。80%以上のカバレッジを確保する。
---

# TDDコマンド

このコマンドは **tdd-guide** エージェントを呼び出し、テスト駆動開発の手法を強制します。

## このコマンドが行うこと

1. **インターフェースのスキャフォールド** - まず型/インターフェースを定義する
2. **テストを先に生成** - 失敗するテストを書く（RED）
3. **最小限のコードを実装** - テストを通すのに十分なだけ書く（GREEN）
4. **リファクタリング** - テストをグリーンに保ちながらコードを改善する（REFACTOR）
5. **カバレッジを確認** - 80%以上のテストカバレッジを確保する

## 使用するタイミング

以下の場合に `/tdd` を使用する:

- 新機能を実装する時
- 新しい関数/コンポーネントを追加する時
- バグを修正する時（まずバグを再現するテストを書く）
- 既存のコードをリファクタリングする時
- 重要なビジネスロジックを構築する時

## 動作の仕組み

tdd-guide エージェントは以下を行う:

1. 入力/出力の**インターフェースを定義**する
2. **失敗するテストを書く**（コードがまだ存在しないため）
3. **テストを実行**し、正しい理由で失敗することを確認する
4. テストを通すための**最小限の実装を書く**
5. **テストを実行**し、通過することを確認する
6. テストをグリーンに保ちながら**コードをリファクタリング**する
7. **カバレッジを確認**し、80%未満であれば追加テストを書く

## TDDサイクル

```
RED → GREEN → REFACTOR → REPEAT

RED:      失敗するテストを書く
GREEN:    テストを通すための最小限のコードを書く
REFACTOR: コードを改善し、テストをグリーンに保つ
REPEAT:   次の機能/シナリオへ
```

## 使用例

```
ユーザー: /tdd 市場の流動性スコアを計算する関数が必要

エージェント (tdd-guide):
# TDDセッション: 市場流動性スコア計算機

## ステップ1: インターフェースの定義（スキャフォールド）

```typescript
// lib/liquidity.ts
export interface MarketData {
  totalVolume: number
  bidAskSpread: number
  activeTraders: number
  lastTradeTime: Date
}

export function calculateLiquidityScore(market: MarketData): number {
  // TODO: 実装
  throw new Error('Not implemented')
}
```

## ステップ2: 失敗するテストを書く（RED）

```typescript
// lib/liquidity.test.ts
import { calculateLiquidityScore } from './liquidity'

describe('calculateLiquidityScore', () => {
  it('流動性の高い市場では高いスコアを返すべき', () => {
    const market = {
      totalVolume: 100000,
      bidAskSpread: 0.01,
      activeTraders: 500,
      lastTradeTime: new Date()
    }
    const score = calculateLiquidityScore(market)
    expect(score).toBeGreaterThan(80)
    expect(score).toBeLessThanOrEqual(100)
  })

  it('流動性の低い市場では低いスコアを返すべき', () => {
    const market = {
      totalVolume: 100,
      bidAskSpread: 0.5,
      activeTraders: 2,
      lastTradeTime: new Date(Date.now() - 86400000) // 1日前
    }
    const score = calculateLiquidityScore(market)
    expect(score).toBeLessThan(30)
    expect(score).toBeGreaterThanOrEqual(0)
  })

  it('エッジケース: ゼロボリュームを処理すべき', () => {
    const market = {
      totalVolume: 0,
      bidAskSpread: 0,
      activeTraders: 0,
      lastTradeTime: new Date()
    }
    const score = calculateLiquidityScore(market)
    expect(score).toBe(0)
  })
})
```

## ステップ3: テスト実行 - 失敗を確認

```bash
npm test lib/liquidity.test.ts

FAIL lib/liquidity.test.ts
  ✕ 流動性の高い市場では高いスコアを返すべき (2 ms)
    Error: Not implemented

1件失敗, 0件通過
```

✅ テストが期待通りに失敗。実装の準備完了。

## ステップ4: 最小限のコードを実装（GREEN）

```typescript
// lib/liquidity.ts
export function calculateLiquidityScore(market: MarketData): number {
  if (market.totalVolume === 0) return 0

  const volumeScore = Math.min(market.totalVolume / 1000, 100)
  const spreadScore = Math.max(100 - (market.bidAskSpread * 1000), 0)
  const traderScore = Math.min(market.activeTraders / 10, 100)

  const hoursSinceLastTrade = (Date.now() - market.lastTradeTime.getTime()) / (1000 * 60 * 60)
  const recencyScore = Math.max(100 - (hoursSinceLastTrade * 10), 0)

  const score = (
    volumeScore * 0.4 +
    spreadScore * 0.3 +
    traderScore * 0.2 +
    recencyScore * 0.1
  )

  return Math.min(Math.max(score, 0), 100)
}
```

## ステップ5: テスト実行 - 通過を確認

```bash
npm test lib/liquidity.test.ts

PASS lib/liquidity.test.ts
  ✓ 流動性の高い市場では高いスコアを返すべき (3 ms)
  ✓ 流動性の低い市場では低いスコアを返すべき (2 ms)
  ✓ エッジケース: ゼロボリュームを処理すべき (1 ms)

3件通過
```

✅ 全テスト通過！

## ステップ6: カバレッジを確認

```bash
npm test -- --coverage lib/liquidity.test.ts

ファイル        | % Stmts | % Branch | % Funcs | % Lines
----------------|---------|----------|---------|--------
liquidity.ts    |   100   |   100    |   100   |   100

カバレッジ: 100% ✅ (目標: 80%)
```

✅ TDDセッション完了！

```

## TDDベストプラクティス

**すべきこと:**
- ✅ 実装の**前**にテストを書く
- ✅ 実装前にテストが**失敗**することを確認する
- ✅ テストを通すための最小限のコードを書く
- ✅ テストがグリーンになってからリファクタリングする
- ✅ エッジケースとエラーシナリオを追加する
- ✅ 80%以上のカバレッジを目指す（重要なコードは100%）

**すべきでないこと:**
- ❌ テストの前に実装を書く
- ❌ 各変更後のテスト実行をスキップする
- ❌ 一度に多くのコードを書く
- ❌ 失敗するテストを無視する
- ❌ 実装の詳細をテストする（振る舞いをテストする）
- ❌ すべてをモックする（統合テストを優先する）

## 含めるテストタイプ

**単体テスト**（関数レベル）:
- ハッピーパスのシナリオ
- エッジケース（空・null・最大値）
- エラー条件
- 境界値

**統合テスト**（コンポーネントレベル）:
- APIエンドポイント
- データベース操作
- 外部サービスコール
- フックを持つReactコンポーネント

**E2Eテスト**（`/e2e` コマンドを使用）:
- 重要なユーザーフロー
- 複数ステップのプロセス
- フルスタック統合

## カバレッジ要件

- **全コードで最低80%**
- **100%必須**:
  - 金融計算
  - 認証ロジック
  - セキュリティ上重要なコード
  - コアビジネスロジック

## 重要な注意事項

**必須**: テストは実装の**前**に書かなければならない。TDDサイクルは:

1. **RED** - 失敗するテストを書く
2. **GREEN** - 通過するように実装する
3. **REFACTOR** - コードを改善する

REDフェーズを絶対にスキップしない。テストの前にコードを書かない。

## 他のコマンドとの連携

- まず `/plan` で何を作るかを理解する
- `/tdd` でテストとともに実装する
- ビルドエラーが発生したら `/build-fix` を使用する
- 実装のレビューには `/code-review` を使用する

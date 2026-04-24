---
description: Playwrightを使ったエンドツーエンドテストの生成と実行。テストジャーニーを作成し、テストを実行し、スクリーンショット/動画/トレースをキャプチャし、アーティファクトをアップロードする。
---

# E2Eコマンド

このコマンドは **e2e-runner** エージェントを呼び出し、Playwrightを使ったエンドツーエンドテストの生成・保守・実行を行います。

## このコマンドが行うこと

1. **テストジャーニーの生成** - ユーザーフローのPlaywrightテストを作成する
2. **E2Eテストの実行** - 複数ブラウザでテストを実行する
3. **アーティファクトのキャプチャ** - 失敗時のスクリーンショット・動画・トレース
4. **結果のアップロード** - HTMLレポートとJUnit XML
5. **不安定なテストの特定** - 不安定なテストを隔離する

## 使用するタイミング

以下の場合に `/e2e` を使用する:

- 重要なユーザージャーニーをテストする時（ログイン・取引・決済）
- 複数ステップのフローがエンドツーエンドで動作することを確認する時
- UIのインタラクションとナビゲーションをテストする時
- フロントエンドとバックエンドの統合を検証する時
- 本番デプロイの準備をする時

## 動作の仕組み

e2e-runner エージェントは以下を行う:

1. **ユーザーフローを分析**し、テストシナリオを特定する
2. Page Object Modelパターンを使って**Playwrightテストを生成**する
3. 複数ブラウザ（Chrome・Firefox・Safari）で**テストを実行**する
4. スクリーンショット・動画・トレースで**失敗をキャプチャ**する
5. 結果とアーティファクトを含む**レポートを生成**する
6. **不安定なテストを特定**し修正を推奨する

## 使用例

```
ユーザー: /e2e 市場の検索と閲覧フローをテストして

エージェント (e2e-runner):
# E2Eテスト生成: 市場検索・閲覧フロー

## テストシナリオの特定

**ユーザージャーニー:** 市場検索 → 結果閲覧 → 市場クリック → 詳細表示

**テストカバレッジ:**
1. 市場ページへのナビゲート
2. セマンティック検索の実行
3. 検索結果の確認
4. 最初の結果をクリック
5. 市場詳細ページが読み込まれることを確認
6. チャートが描画されることを確認

## 生成されたテストコード

```typescript
// tests/e2e/markets/search-and-view.spec.ts
import { test, expect } from '@playwright/test'
import { MarketsPage } from '../../pages/MarketsPage'
import { MarketDetailsPage } from '../../pages/MarketDetailsPage'

test.describe('市場検索・閲覧フロー', () => {
  test('ユーザーは市場を検索して詳細を閲覧できる', async ({ page }) => {
    const marketsPage = new MarketsPage(page)
    await marketsPage.goto()

    await expect(page).toHaveTitle(/Markets/)
    await marketsPage.searchMarkets('election')

    await page.waitForResponse(resp =>
      resp.url().includes('/api/markets/search') && resp.status() === 200
    )

    const marketCards = marketsPage.marketCards
    await expect(marketCards.first()).toBeVisible()

    await page.screenshot({ path: 'artifacts/search-results.png' })

    await marketCards.first().click()
    await expect(page).toHaveURL(/\/markets\/[a-z0-9-]+/)

    const detailsPage = new MarketDetailsPage(page)
    await expect(detailsPage.marketName).toBeVisible()
    await expect(detailsPage.priceChart).toBeVisible()

    await page.screenshot({ path: 'artifacts/market-details.png' })
  })

  test('結果なしの検索は空の状態を表示する', async ({ page }) => {
    const marketsPage = new MarketsPage(page)
    await marketsPage.goto()
    await marketsPage.searchMarkets('xyznonexistentmarket123456')
    await expect(page.locator('[data-testid="no-results"]')).toBeVisible()
  })
})
```

## テスト実行

```bash
npx playwright test tests/e2e/markets/search-and-view.spec.ts

3件のテストを3ワーカーで実行中

  ✓  [chromium] › ユーザーは市場を検索して詳細を閲覧できる (4.2s)
  ✓  [chromium] › 結果なしの検索は空の状態を表示する (1.8s)

  2件通過 (6.0s)

生成されたアーティファクト:
- artifacts/search-results.png
- artifacts/market-details.png
- playwright-report/index.html
```

✅ E2EテストスイートがCI/CD統合の準備完了！

```

## テストアーティファクト

テスト実行時に以下のアーティファクトがキャプチャされる:

**全テストで:**
- タイムラインと結果を含むHTMLレポート
- CI統合用のJUnit XML

**失敗時のみ:**
- 失敗状態のスクリーンショット
- テストの動画録画
- デバッグ用トレースファイル（ステップバイステップのリプレイ）
- ネットワークログ
- コンソールログ

## アーティファクトの表示

```bash
# ブラウザでHTMLレポートを表示
npx playwright show-report

# 特定のトレースファイルを表示
npx playwright show-trace artifacts/trace-abc123.zip

# スクリーンショットはartifacts/ディレクトリに保存
```

## 不安定なテストの検出

テストが断続的に失敗する場合:

```
⚠️  不安定なテストを検出: tests/e2e/markets/trade.spec.ts

テストは10回中7回通過（70%の通過率）

よくある失敗:
"'[data-testid="confirm-btn"]'の待機タイムアウト"

推奨される修正:
1. 明示的な待機を追加: await page.waitForSelector('[data-testid="confirm-btn"]')
2. タイムアウトを増やす: { timeout: 10000 }
3. コンポーネントの競合状態を確認する
4. アニメーションによる要素の非表示を確認する

隔離の推奨: 修正されるまでtest.fixme()でマークする
```

## ブラウザ設定

テストはデフォルトで複数ブラウザで実行される:

- ✅ Chromium（デスクトップChrome）
- ✅ Firefox（デスクトップ）
- ✅ WebKit（デスクトップSafari）
- ✅ モバイルChrome（オプション）

ブラウザを調整するには `playwright.config.ts` で設定する。

## CI/CD統合

CIパイプラインへの追加:

```yaml
# .github/workflows/e2e.yml
- name: Playwrightをインストール
  run: npx playwright install --with-deps

- name: E2Eテストを実行
  run: npx playwright test

- name: アーティファクトをアップロード
  if: always()
  uses: actions/upload-artifact@v3
  with:
    name: playwright-report
    path: playwright-report/
```

## ベストプラクティス

**すべきこと:**

- ✅ 保守性のためにPage Object Modelを使用する
- ✅ セレクターにdata-testid属性を使用する
- ✅ 任意のタイムアウトではなくAPIレスポンスを待つ
- ✅ エンドツーエンドの重要なユーザージャーニーをテストする
- ✅ mainへのマージ前にテストを実行する
- ✅ テスト失敗時にアーティファクトをレビューする

**すべきでないこと:**

- ❌ 脆弱なセレクターを使用する（CSSクラスは変更される可能性がある）
- ❌ 実装の詳細をテストする
- ❌ 本番環境に対してテストを実行する
- ❌ 不安定なテストを無視する
- ❌ 失敗時のアーティファクトレビューをスキップする
- ❌ すべてのエッジケースをE2Eでテストする（単体テストを使用する）

## 他のコマンドとの連携

- テストする重要なジャーニーを特定するには `/plan` を使用する
- 単体テスト（高速・より細かい粒度）には `/tdd` を使用する
- テスト品質の確認には `/code-review` を使用する

## クイックコマンド

```bash
# 全E2Eテストを実行
npx playwright test

# 特定のテストファイルを実行
npx playwright test tests/e2e/markets/search.spec.ts

# ヘッドモードで実行（ブラウザを表示）
npx playwright test --headed

# テストをデバッグ
npx playwright test --debug

# テストコードを生成
npx playwright codegen http://localhost:3000

# レポートを表示
npx playwright show-report
```

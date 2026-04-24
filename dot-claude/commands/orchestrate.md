---
description: マルチエージェントワークフローのための順次・tmux/worktreeオーケストレーションガイド。
---

# オーケストレートコマンド

複雑なタスクのための順次エージェントワークフロー。

## 使い方

`/orchestrate [ワークフロータイプ] [タスクの説明]`

## ワークフロータイプ

### feature（機能実装）

フル機能実装ワークフロー:

```
planner -> tdd-guide -> code-reviewer -> security-reviewer
```

### bugfix（バグ修正）

バグ調査・修正ワークフロー:

```
planner -> tdd-guide -> code-reviewer
```

### refactor（リファクタリング）

安全なリファクタリングワークフロー:

```
architect -> code-reviewer -> tdd-guide
```

### security（セキュリティ）

セキュリティ重視のレビュー:

```
security-reviewer -> code-reviewer -> architect
```

## 実行パターン

ワークフロー内の各エージェントについて:

1. **エージェントを呼び出す** — 前のエージェントからのコンテキストとともに
2. **出力を収集する** — 構造化された引き継ぎドキュメントとして
3. **次のエージェントに渡す** — チェーン内の次へ
4. **結果を集約する** — 最終レポートにまとめる

## 引き継ぎドキュメントフォーマット

エージェント間で引き継ぎドキュメントを作成する:

```markdown
## 引き継ぎ: [前のエージェント] -> [次のエージェント]

### コンテキスト
[行ったことのサマリー]

### 所見
[主要な発見事項または決定事項]

### 変更されたファイル
[触れたファイルのリスト]

### 未解決の質問
[次のエージェントへの未解決アイテム]

### 推奨事項
[推奨される次のステップ]
```

## 例: featureワークフロー

```
/orchestrate feature "ユーザー認証を追加"
```

実行内容:

1. **Plannerエージェント**
   - 要件を分析する
   - 実装計画を作成する
   - 依存関係を特定する
   - 出力: `引き継ぎ: planner -> tdd-guide`

2. **TDD Guideエージェント**
   - plannerの引き継ぎを読む
   - まずテストを書く
   - テストを通すように実装する
   - 出力: `引き継ぎ: tdd-guide -> code-reviewer`

3. **Code Reviewerエージェント**
   - 実装をレビューする
   - 問題を確認する
   - 改善を提案する
   - 出力: `引き継ぎ: code-reviewer -> security-reviewer`

4. **Security Reviewerエージェント**
   - セキュリティ監査
   - 脆弱性チェック
   - 最終承認
   - 出力: 最終レポート

## 最終レポートフォーマット

```
オーケストレーションレポート
====================
ワークフロー: feature
タスク: ユーザー認証を追加
エージェント: planner -> tdd-guide -> code-reviewer -> security-reviewer

サマリー
-------
[1段落のサマリー]

エージェント出力
-------------
Planner: [サマリー]
TDD Guide: [サマリー]
Code Reviewer: [サマリー]
Security Reviewer: [サマリー]

変更されたファイル
-------------
[変更された全ファイルのリスト]

テスト結果
------------
[テストの通過/失敗サマリー]

セキュリティステータス
---------------
[セキュリティの所見]

推奨
--------------
[SHIP（リリース可） / NEEDS WORK（要修正） / BLOCKED（ブロック中）]
```

## 並行実行

独立したチェックには、エージェントを並行して実行する:

```markdown
### 並行フェーズ
同時に実行:
- code-reviewer（品質）
- security-reviewer（セキュリティ）
- architect（設計）

### 結果のマージ
出力を単一レポートに統合する
```

外部のtmuxペーンワーカーと別々のgit worktreeを使用する場合は、`node scripts/orchestrate-worktrees.js plan.json --execute` を使用する。組み込みのオーケストレーションパターンはインプロセスで動作し、このヘルパーは長時間実行またはクロスハーネスセッション向け。

ワーカーがメインチェックアウトからダーティなファイルや未追跡のローカルファイルを参照する必要がある場合は、プランファイルに `seedPaths` を追加する。ECCは `git worktree add` 後に選択されたパスのみを各ワーカーのworktreeにオーバーレイし、ブランチを分離しながらローカルのスクリプト・プラン・ドキュメントを公開する。

```json
{
  "sessionName": "workflow-e2e",
  "seedPaths": [
    "scripts/orchestrate-worktrees.js",
    "scripts/lib/tmux-worktree-orchestrator.js",
    ".claude/plan/workflow-e2e-test.json"
  ],
  "workers": [
    { "name": "docs", "task": "オーケストレーションドキュメントを更新する。" }
  ]
}
```

ライブtmux/worktreeセッションのコントロールプレーンスナップショットをエクスポートするには:

```bash
node scripts/orchestration-status.js .claude/plan/workflow-visual-proof.json
```

スナップショットには、セッションアクティビティ・tmuxペインメタデータ・ワーカーの状態・目標・シードオーバーレイ・最近の引き継ぎサマリーがJSON形式で含まれる。

## オペレーターコマンドセンターへの引き継ぎ

ワークフローが複数のセッション・worktree・tmuxペーンにまたがる場合は、最終引き継ぎにコントロールプレーンブロックを追加する:

```markdown
コントロールプレーン
-------------
セッション:
- アクティブなセッションIDまたはエイリアス
- 各アクティブワーカーのブランチ + worktreeパス
- 該当する場合のtmuxペーンまたはデタッチセッション名

差分:
- git statusサマリー
- 変更されたファイルのgit diff --stat
- マージ/コンフリクトリスクのメモ

承認:
- 保留中のユーザー承認
- 確認待ちのブロックされたステップ

テレメトリー:
- 最後のアクティビティタイムスタンプまたはアイドルシグナル
- 推定トークンまたはコストのドリフト
- フックやレビュアーが発生させたポリシーイベント
```

これにより、プランナー・実装者・レビュアー・ループワーカーがオペレーター側から判読可能な状態を保つ。

## 引数

$ARGUMENTS:

- `feature <説明>` - フル機能実装ワークフロー
- `bugfix <説明>` - バグ修正ワークフロー
- `refactor <説明>` - リファクタリングワークフロー
- `security <説明>` - セキュリティレビューワークフロー
- `custom <エージェント> <説明>` - カスタムエージェントシーケンス

## カスタムワークフロー例

```
/orchestrate custom "architect,tdd-guide,code-reviewer" "キャッシュ層を再設計"
```

## ヒント

1. **複雑な機能にはplannerから始める**
2. **マージ前に必ずcode-reviewerを含める**
3. **認証/決済/PII処理にはsecurity-reviewerを使用する**
4. **引き継ぎを簡潔に保つ** — 次のエージェントが必要な情報に集中する
5. **必要に応じてエージェント間で検証を実行する**

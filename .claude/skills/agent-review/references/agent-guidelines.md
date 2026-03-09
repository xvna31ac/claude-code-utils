# エージェント定義ガイドライン（フォールバック参照）

このファイルは CLAUDE.md が利用不可の場合のフォールバック参照として使用する。
エージェント定義の核心原則を要約したものであり、レビュー観点の基準として機能する。

---

## エージェントファイルの必須フォーマット

```
.claude/agents/<name>.md
```

**frontmatter フィールド:**
- `name`: エージェント識別子（ケバブケース必須）
- `description`: quoted string（ダブルクォートまたはシングルクォート）で記述する使用条件の説明
- `tools`: カンマ区切りのツール名リスト（有効値: Read, Write, Edit, Bash, Glob, Grep, WebFetch, WebSearch, Agent）
- `model`: 省略可能。指定する場合は sonnet/opus/haiku 系のいずれか
- `permissionMode`: 省略可能。自動化レベルを制御する（有効値: `dontAsk` / `acceptEdits` / `bypassPermissions`）
- `memory`: 省略可能。クロスセッション知識蓄積を有効化する（有効値: `user` / `project`）

**ファイル命名規則:**
- ファイル名（拡張子を除く）と `name` フィールドの値が一致すること

---

## description の品質基準

- **形式**: quoted string（`"..."` または `'...'`）で記述する
- **`|` ブロックスカラーは禁止**: スキルと異なり、エージェントの description は quoted string が標準
- **使用条件の必須記載**: 「いつこのエージェントを使うべきか」が明確に読み取れること
- **強い description（具体的ユースケースの明示）**: 単なる役割名にとどまらず、具体的なユースケースを含めること。自動デリゲーション精度はこのフィールドに依存する
- **長さ**: 1〜2文程度。過度に冗長にしない

**強い description の例:**
```
# 悪い例（役割名のみ）
description: "Code reviewer"
description: "Backend developer"

# 良い例（具体的なユースケースを含む）
description: "Proactively reviews code for quality and security issues. Use when reviewing PRs, checking implementation quality, or auditing existing code."
description: "Implements backend features and fixes bugs. Use when adding new API endpoints, refactoring server-side logic, or resolving backend errors."
```

**禁止事項:**
- description を `|` ブロックスカラーで記述する（エージェントの標準形式は quoted string）
- 使用条件・トリガー条件を省略する（Claude Code が自動選択できなくなる）
- 役割名のみで具体的なユースケースを含めない（自動デリゲーション精度が低下する）

---

## tools 割り当てルール（ロールタイプ別）

| ロールタイプ | tools の組み合わせ | 代表的な役割 |
|------------|-----------------|------------|
| **コードライター型** | Read, Write, Edit, Bash, Glob, Grep | 開発者、実装担当 |
| **リードオンリー型** | Read, Grep, Glob | レビュアー、監査担当 |
| **リサーチ型** | Read, Grep, Glob, WebFetch, WebSearch | 調査担当、アナリスト |
| **ドキュメント型** | Read, Write, Edit, Glob, Grep, WebFetch, WebSearch | ドキュメント担当 |
| **中間型** | 上記以外の組み合わせ | ハイブリッドな役割 |

**ロールタイプ識別の注意点:**
- ドキュメント型はコードライター型の特殊ケース（WebFetch/WebSearch を含む点が異なる）
- 中間型は必ずしも問題ではないが、description のロールニュアンスと整合しているか確認する

---

## body の推奨構成

エージェント定義の body は以下の構成が推奨される:

1. **ロール宣言**: 「あなたは〜です」で始まる1〜2文の役割説明
2. **チェックリスト**: 実施すべき項目の箇条書き（品質基準・検証観点など）
3. **Communication Protocol** セクション（他エージェントとの連携がある場合）:
   - 連携クエリの JSON フォーマット例
   - `requesting_agent` フィールドの値は `name` フィールドと一致させる
4. **Development Workflow** セクション（複雑なタスクを持つ場合）:
   - フェーズ別の実施手順
   - 各フェーズの完了基準

**代替推奨構造（Qiita ベストプラクティス）:**
より汎用性が高い構造として以下も推奨される:
1. **Role**: エージェントの役割宣言
2. **When Invoked**: 自動選択される条件・明示的呼び出しの条件
3. **Key Practices**: 主要な実施事項・チェックリスト
4. **Important Notes**: 注意事項・制約事項
5. **Example Usages**: 具体的な使用例

**必須ではないが推奨:**
- Communication Protocol と Development Workflow は Nice to have として扱う
- 省略しても動作には影響しない

## permissionMode フィールドの有効値と整合性基準

`permissionMode` はエージェントの自動化レベルを制御するオプションフィールド:

| 値 | 意味 | 推奨ロールタイプ |
|----|------|----------------|
| `dontAsk` | 許可確認を一切行わない | リードオンリー型（Write 系ツールなし） |
| `acceptEdits` | ファイル編集を自動承認 | コードライター型（Write/Edit を含む） |
| `bypassPermissions` | 全ての許可チェックをバイパス | 意図的設定のみ（要注意） |

**整合性チェック:**
- リードオンリー型（Read/Grep/Glob のみ）に `acceptEdits` が設定されている場合 → Important（過剰な権限）
- コードライター型（Write/Edit/Bash を含む）に `dontAsk` が設定されている場合 → Important（意図しない自動実行リスク）
- `bypassPermissions` は全権限付与のため、常に Important として意図を確認する

## memory フィールドの有効値

`memory` はクロスセッション知識蓄積を有効化するオプションフィールド:

| 値 | スコープ |
|----|---------|
| `user` | ユーザーレベルのメモリ（全プロジェクト共通） |
| `project` | プロジェクトレベルのメモリ（現プロジェクト限定） |

## model とタスク複雑度の推奨対応表

| タスク複雑度 | 推奨モデル | 非推奨 |
|------------|----------|--------|
| 探索・検索・単純分析（シンプルなパターンマッチ、ファイル検索など） | `haiku` 系 | `opus` 系（過剰）|
| 標準的な実装・コードレビュー・デバッグ | `sonnet` 系 | - |
| 複雑な設計・アーキテクチャ判断・多段階推論 | `opus` 系 | `haiku` 系（不足） |

**チェック観点:**
- description や body のタスク内容と model の組み合わせが不整合な場合は Nice to have として報告する

## 過度な詳細指定の回避

Claude はすでに広範な知識を持つため、自明な内容の逐一指示は避けること:

**避けるべき指示の例:**
- プログラミング言語の基本構文の説明（例: 「for ループは `for (let i = 0; i < n; i++)` の形式で書く」）
- フレームワークのデフォルト動作（例: 「React では state が変わると再レンダリングされる」）
- 一般的なベストプラクティスの羅列（例: 「変数名は意味のある名前にする」「コメントを書く」）

**理由:**
- 詳細すぎる指定は Claude の判断余地を削ぎ、柔軟性を低下させる
- メンテナンス負担が増加し、モデル改善の恩恵を受けにくくなる
- エージェント固有の専門知識・制約事項に集中すること

## CLAUDE.md との重複回避

プロジェクト共通ルールは `CLAUDE.md` に一元管理し、個別エージェント定義に重複して記載しない:

**重複が問題になるケース:**
- CLAUDE.md に記載されたコーディング規約をエージェント body に再掲している
- CLAUDE.md のプロジェクト構造説明をエージェント内でも説明している
- プロジェクト全体に適用されるルールを特定エージェントだけに記述している

**対処方針:**
- 「プロジェクト全体に適用されるルール」→ CLAUDE.md に記載して削除
- 「このエージェント固有の制約・知識」→ エージェント定義に残す

---

## 禁止事項

- `name` フィールドをケバブケース以外で記述する（例: `backendDeveloper` は不可）
- `name` とファイル名を不一致にする
- `description` を `|` ブロックスカラーで記述する
- `description` を役割名のみで記述し具体的なユースケースを含めない
- `tools` に無効なツール名を含める
- `tools` に重複するツール名を含める
- `model` に sonnet/opus/haiku 系以外の値を指定する
- `permissionMode` に `dontAsk` / `acceptEdits` / `bypassPermissions` 以外の値を指定する
- `memory` に `user` / `project` 以外の値を指定する
- body 内で参照するエージェント名が `.claude/agents/` に存在しない
- Claude が自明に知っている情報を body 内で逐一指示する（過度な詳細指定）
- CLAUDE.md のプロジェクト共通ルールをエージェント定義内に重複して記載する

---

## 重大度分類の基準

| 重大度 | 定義 | 例 |
|--------|------|---|
| **Critical** | Claude Code の動作を壊す・エージェントが正常に読み込まれない | 必須フィールド（name/description/tools）の欠如、frontmatter YAML 解析不能、tools に無効なツール名 |
| **Important** | 品質・運用性・自動選択精度に影響する | description に使用条件なし・具体的ユースケースなし、description が `|` スカラー、tools がロールタイプ不一致、name とファイル名不一致、tools に重複、permissionMode とロールタイプの不整合、`bypassPermissions` の使用 |
| **Nice to have** | 明確性・保守性の向上につながる | Communication Protocol/Development Workflow セクション欠如、推奨システムプロンプト構造への非準拠、model とタスク複雑度の不整合、過度な詳細指定、CLAUDE.md との内容重複、description が冗長・不明瞭 |

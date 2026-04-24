---
name: sync-diff
description: dot-claudeリポジトリ（dot-claude/）とユーザースコープ（~/.claude/）のagents・commands・skillsを比較して差分を一覧表示するスキル。「同期したい」「差分を確認したい」「更新漏れを確認したい」「ユーザースコープと比べたい」と言われたら積極的に使うこと。
---

# sync-diff スキル

`dot-claude/`（このリポジトリ）と `~/.claude/`（ユーザースコープ）のagents・commands・skillsを比較し、差分を視覚的に一覧表示する。

## リポジトリ情報

- **リポジトリパス**: `C:/Users/ynepr/Documents/workspace/develop/claude-utils`
- **リポジトリ側の対象**: `dot-claude/agents/`, `dot-claude/commands/`, `dot-claude/skills/`
- **ユーザースコープ側の対象**: `~/.claude/agents/`, `~/.claude/commands/`, `~/.claude/skills/`

## 比較ルール

- **agents / commands**: ファイル名（`.md`）で比較
- **skills**: ディレクトリ名で比較（`SKILL.md` が実体）
- 内容差分は `diff` コマンドで確認

## 実行手順

### Step 1: ファイル一覧を取得

```bash
# agents
ls dot-claude/agents/
ls ~/.claude/agents/

# commands
ls dot-claude/commands/
ls ~/.claude/commands/

# skills（ディレクトリ名のみ）
ls -d dot-claude/skills/*/
ls -d ~/.claude/skills/*/
```

### Step 2: 差分分類

各カテゴリで以下に分類する：

| 記号 | 意味 |
|------|------|
| `✓` | 両方に存在・内容一致 |
| `⚠` | 両方に存在・内容差分あり |
| `→` | リポジトリのみ（ユーザーに未反映） |
| `←` | ユーザーのみ（リポジトリに未反映） |

内容差分の確認（agents/commands）:
```bash
diff dot-claude/agents/<file> ~/.claude/agents/<file>
```

内容差分の確認（skills）:
```bash
diff dot-claude/skills/<name>/SKILL.md ~/.claude/skills/<name>/SKILL.md
```

### Step 3: 結果を表示

以下のフォーマットで出力する（横並びで repo / user の状態を一目で比較できるようにする）：

```
## AGENTS
| 名前 | repo | user | 差分概要 |
|------|------|------|---------|
| architect                    | ✓ | ✓ | 空行のみ |
| code-reviewer                | ✓ | ✓ | 一致 |
| workflow-artifact-reviewer   | ✓ | ✗ | — |
| some-user-only               | ✗ | ✓ | — |

## COMMANDS
| 名前 | repo | user | 差分概要 |
...

## SKILLS
| 名前 | repo | user | 差分概要 |
...

---
## サマリー
- `✓/✗` リポジトリのみ（ユーザーに未反映）: N件
- `✗/✓` ユーザーのみ（リポジトリに未取込）: N件
- `✓/✓`＋差分 内容差分あり: N件
- `✓/✓`＋一致 完全一致: N件
```

差分概要の書き方：
- 空行の追加・削除のみ → `空行のみ`
- テキスト変更あり → 変更内容を1行で（例: `番号ズレ（1→3）`、`モデル名更新`）
- 片方にのみ存在 → `—`

## 注意事項

- `←`（ユーザーのみ）は意図的なものもあるため、削除や上書きは**ユーザーに確認**してから行う
- このスキルは比較・表示のみを行い、ファイルの変更は行わない

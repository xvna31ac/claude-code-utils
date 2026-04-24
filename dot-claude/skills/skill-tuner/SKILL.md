---
name: skill-tuner
description: >
  既存スキルの品質を反復改善するスキル。出力品質とトリガー精度の2軸で改善ループを回す。
  以下のケースで積極的に使うこと：
  - 「スキルの出力が期待通りでない」「改善したい」「品質を上げたい」
  - 「スキルが起動しない」「余計に起動する」「トリガーを調整したい」
  - スキルを修正・更新した後に品質確認をしたい
  - eval ループを回したい、ベンチマークを取りたい
  output quality と trigger accuracy のどちらか、または両方を対象にできる。
---

# skill-tuner

既存スキルを2つのループで反復改善する。

- **ループA — Output quality**: スキルが起動した後の出力内容を評価・改善する
- **ループB — Trigger verification**: 「起動すべき場面で起動するか」「起動すべきでない場面で起動しないか」を評価・改善する

まず対象スキルのパスを確認し、どちらのループを実行するか（output / trigger / 両方）をユーザーに確認する。

**skill-creator のインフラを流用する。** パスの確認：

```bash
ls ~/.claude/plugins/cache/claude-plugins-official/skill-creator/*/skills/skill-creator/
```

以降の手順では `<sc>` を skill-creator のルートパスとして扱う。

---

## ループA: Output Quality

スキルが起動した後の出力を評価し、SKILL.md 本体を改善するループ。

### Step 1: テストケースの準備

対象スキルの `evals/evals.json` を確認する。なければユーザーと一緒に作成する。
テストケースは「実際のユーザーが送るであろうプロンプト」を使う。

```json
{
  "skill_name": "example",
  "evals": [
    {
      "id": 1,
      "eval_name": "descriptive-name",
      "prompt": "ユーザーが実際に送るプロンプト",
      "expected_output": "期待する結果の説明",
      "files": [],
      "assertions": []
    }
  ]
}
```

### Step 2: 全ランを同一ターンでスポーン

各テストケースについて **with-skill** と **baseline** を **同一ターンで** 起動する。後でまとめて起動しない。

**with-skill:**

```
Execute this task:
- Skill path: <skill-path>
- Task: <prompt>
- Input files: <files or "none">
- Save outputs to: <workspace>/iteration-N/output-evals/<eval-name>/with_skill/outputs/
```

**baseline（初回は without-skill、2回目以降は前バージョンのスナップショット）:**

```
Execute this task (no skill):
- Task: <prompt>
- Save outputs to: <workspace>/iteration-N/output-evals/<eval-name>/without_skill/outputs/
```

前バージョンとの比較をする場合は先に `cp -r <skill-path> <workspace>/skill-snapshot/` しておく。

各ディレクトリに `eval_metadata.json` を作成：

```json
{"eval_id": 1, "eval_name": "...", "prompt": "...", "assertions": []}
```

### Step 3: ランの進行中にアサーションを設計

ランを待つ間にアサーションを設計し `evals/evals.json` と `eval_metadata.json` を更新する。
客観的に検証できる観点を選ぶ。主観的な品質（文体・UX等）は人間レビューに委ねる。

### Step 4: タイミングデータの記録

各サブエージェント完了通知に含まれる `total_tokens` と `duration_ms` を即座に `timing.json` に保存する。
通知は一度しか来ない。到着のたびに処理すること。

```json
{"total_tokens": 84852, "duration_ms": 23332, "total_duration_seconds": 23.3}
```

### Step 5: 採点・集計・ビューア

1. **採点** — `<sc>/agents/grader.md` を参照して採点し `grading.json` に保存。
   `grading.json` の expectations 配列は `text` / `passed` / `evidence` フィールドを使うこと（ビューアの依存）。

2. **集計:**

   ```bash
   python -m scripts.aggregate_benchmark <workspace>/iteration-N/output-evals --skill-name <name>
   ```

3. **ビューア起動（ビューアを起動してから人間にレビューさせること）:**

   ```bash
   nohup python <sc>/eval-viewer/generate_review.py \
     <workspace>/iteration-N/output-evals \
     --skill-name "<name>" \
     --benchmark <workspace>/iteration-N/output-evals/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
   ```

   iteration 2+ は `--previous-workspace <workspace>/iteration-N-1/output-evals` を追加。
   ディスプレイがない環境では `--static <output_path>` でHTMLファイルとして出力する。

4. フィードバックを求め、`feedback.json` を読んでスキル本体を改善して次の反復へ。

---

## ループB: Trigger Verification

スキルの `description` を改善して、トリガー精度を上げるループ。

### Step 1: トリガーテストケースの準備

対象スキルの `evals/trigger_evals.json` を確認する。なければユーザーと一緒に作成する。

```json
{
  "skill_name": "example",
  "trigger_evals": [
    {
      "id": 1,
      "eval_name": "descriptive-name",
      "prompt": "ユーザーが実際に送るプロンプト",
      "should_trigger": true,
      "rationale": "なぜこのプロンプトでトリガーすべき/すべきでないか"
    }
  ]
}
```

**テストケース設計の指針:**

- `should_trigger: true`：スキルが有用な場面。ただし「スキル名を直接指定した」ような自明すぎるものは避ける
- `should_trigger: false`：スキルに関連するが対象外の場面。「類似していて紛らわしい」ケースほど価値が高い
- 比率は 50:50 程度が望ましい
- 境界線上のケースを意図的に含めると description の弱点が見えやすい
- 自明すぎる負例（「フィボナッチ関数を書いて」等）は避ける — 境界を試せないため

### Step 2: 全ランをスポーン

各テストケースについてサブエージェントを起動する。**SKILL_INVOKED の出力形式を厳守させること。**

```
You have access to the following skill: <skill-path>

Complete this task: <prompt>

At the very end of your response, on its own line, output exactly one of:
SKILL_INVOKED: yes
SKILL_INVOKED: no
```

description を変更した前後の比較をする場合は、変更前の description を持つスナップショットを baseline として使う。

### Step 3: 採点

各ランの出力末尾から `SKILL_INVOKED: yes/no` を抽出し、`should_trigger` と照合する。

```
passed = (actual_trigger == should_trigger)
```

`grading.json` に保存：

```json
{
  "eval_id": 1,
  "should_trigger": true,
  "actual_trigger": true,
  "passed": true,
  "evidence": "出力末尾: SKILL_INVOKED: yes"
}
```

### Step 4: 集計・分析

pass rate を集計し、特に2種類のエラーを分けて把握する：

| エラー種別 | 定義 | 示す問題 |
|-----------|------|---------|
| **False negative** | `should_trigger=true` なのに起動しなかった | description の "when to use" が弱い・曖昧 |
| **False positive** | `should_trigger=false` なのに起動した | description のスコープが広すぎる |

```bash
python -m scripts.aggregate_benchmark <workspace>/iteration-N/trigger-evals --skill-name <name>
```

ビューアも同様に起動して人間にレビューさせる（ループAと同じコマンド、パスのみ変更）。

### Step 5: description を改善して反復

分析結果に基づいて SKILL.md の frontmatter `description` を修正する。

- **False negative 多** → 起動すべき場面のキーワード・文脈・具体例をより明示する
- **False positive 多** → 対象外の場面を明記するか、スコープを絞る記述を加える
- **両方多い** → スコープの境界線を明確にする。description を構造的に整理する
- **改善が止まる場合** → 文長・優先順位・例示の有無など description の構造自体を変える

修正後、次の反復へ。スコアが収束するか、ユーザーが満足したら終了。

---

## ワークスペース構造

```
<skill-name>-tuner-workspace/
└── iteration-N/
    ├── output-evals/
    │   ├── <eval-name>/
    │   │   ├── with_skill/
    │   │   │   ├── outputs/
    │   │   │   ├── grading.json
    │   │   │   └── timing.json
    │   │   ├── without_skill/ (or old_skill/)
    │   │   └── eval_metadata.json
    │   └── benchmark.json
    └── trigger-evals/
        ├── <eval-name>/
        │   ├── grading.json
        │   └── timing.json
        └── trigger_benchmark.json
```

---

## 参照するリソース（skill-creator のインフラ）

| ファイル | 用途 |
|---------|------|
| `<sc>/agents/grader.md` | アサーション採点の手順 |
| `<sc>/agents/analyzer.md` | ベンチマーク結果の分析パターン |
| `<sc>/eval-viewer/generate_review.py` | ビューアの起動スクリプト |
| `<sc>/scripts/aggregate_benchmark.py` | 集計スクリプト |
| `<sc>/references/schemas.md` | JSON スキーマ定義 |

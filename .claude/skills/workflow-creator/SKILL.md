---
name: workflow-creator
description: |
  ユーザーの要求から対話的にワークフローを設計し、必要な Sub Agent・Skills の完全なセットを生成します。
  新しいAI駆動ワークフローの構築、既存フローの改善提案、Sub Agent / Skills の設計が必要な場合に自動適用されます。
  「ワークフローを作って」「新しいフローを設計して」「Sub Agentを設計して」などのリクエストでトリガーされます。
---

## 概要

ユーザーの要求を起点に、対話的なフィードバックループを通じてワークフロー・Sub Agent・Skills の完全なセットを設計・生成する。
素人ユーザーを前提とし、AI・ユーザー双方の懸念がなくなるまで反復対話する。

**状態ファイル**: `.workflow/state/workflow-creator.md`（Phase 1 で初期化・以降全フェーズで更新）

---

## フェーズ一覧

| Phase | フェーズ名 | 使用Skill / 参照 | 承認ゲート |
|-------|-----------|-----------------|-----------|
| 1 | タスク分離・状態初期化 | — | なし |
| 2 | 要求解析・ベストプラクティス調査 | — | なし |
| 3 | ワークフロー提案・確立 | — | **Gate A** |
| 4 | スコープ確認 | — | **Gate B** |
| 5 | Phase-SubAgent-Skill マッピング提案 | `skills/workflow-creator/references/workflow-template.md` | なし |
| 6 | 既存 Sub Agent・Skill 重複確認 | — | なし |
| 7 | マッピング確立 | — | **Gate C** |
| 8 | ワークフロースキル生成 | `.claude/skills/skill-review/references/skill-creator-guidelines.md` | なし |
| 9 | ワークフロースキル AI フィードバック | skill-review（不在時は新規作成） | **Gate D** |
| 10 | Sub Agent 作成 | `skills/workflow-creator/references/subagent-checklist.md` | なし |
| 11 | Sub Agent AI フィードバック | agent-review（不在時は新規作成） | なし |
| 12 | Skills 作成 | `skills/workflow-creator/references/skills-checklist.md` | なし |
| 13 | Skills AI フィードバック | skill-review（不在時は新規作成） | なし |
| 14 | 品質検証フィードバックループ | skill-review, agent-review | **Gate E** |

---

## 承認ゲート定義

### Gate A — ワークフロー構造の合意
- 確認内容: 提案したワークフロー構造（フェーズ数・各フェーズの役割・全体フロー）について、AI・ユーザー双方の懸念が解消されているか
- APPROVE 時: Phase 4 へ進む
- REJECT 時: ユーザーのフィードバックを取り込み Phase 3 を反復する

### Gate B — スコープ確定
- 確認内容: 設計したワークフローのカバー範囲（コミットまで？デプロイまで？モニタリングも？）について AI・ユーザー双方が懸念なく合意できているか
- APPROVE 時: Phase 5 へ進む
- REJECT 時: スコープを再調整し Phase 4 を反復する

### Gate C — マッピング確定
- 確認内容: 全フェーズへの Sub Agent・Skills 割り当てについて、AI・ユーザー双方の懸念が解消されているか
- APPROVE 時: Phase 8 へ進む
- REJECT 時: マッピングを修正し Phase 5 に戻る

### Gate D — ワークフロースキル品質確認
- 確認内容: skill-review の結果が Critical ゼロ・Important ゼロであるか
- APPROVE 時: Phase 10 へ進む
- REJECT 時: Phase 8 でスキルを修正し Phase 9 を反復する

### Gate E — 全成果物の最終承認
- 確認内容: 生成した全ファイル（ワークフロースキル・Sub Agent・Skills）の品質をユーザーが承認するか
- APPROVE 時: 完了報告を行い状態ファイルを完了ステータスに更新する
- REJECT 時: 指摘された成果物の生成フェーズに戻り修正する

---

## Phase 1: タスク分離・状態初期化

- 実行内容: 状態ファイルを初期化し、ユーザーの要求を受け取る前の準備を行う
- 専門性:
  - `.workflow/state/workflow-creator.md` の存在を Glob で確認する
  - 存在する場合は Read して前回セッションの状態を確認し、継続か新規開始かをユーザーに確認する
  - 新規開始の場合は状態ファイルを以下の初期状態で Write する:
    ```
    # workflow-creator 状態
    - 開始日時: {現在日時}
    - 現在フェーズ: Phase 1
    - ユーザー要求: （未入力）
    - Gate A: 未実施
    - Gate B: 未実施
    - Gate C: 未実施
    - Gate D: 未実施
    - Gate E: 未実施
    - 生成予定ファイル: （未確定）
    ```
- 期待成果物: 初期化済みの状態ファイル `.workflow/state/workflow-creator.md`
- 完了基準: 状態ファイルが存在し Phase 2 に進む準備が整っている

---

## Phase 2: 要求解析・ベストプラクティス調査

- 実行内容: ユーザーの要求を詳細に聞き取り、ベストプラクティスを調査した上でワークフローの方向性を確認する
- 専門性:
  - ユーザーに以下を対話的に確認する:
    - 自動化したいプロセス・タスクの概要
    - 現在の作業フローの課題・ボトルネック
    - 期待する成果物・品質基準
    - 技術スタック・制約条件
  - **パターン分岐**（ユーザー要求の内容により分岐する）:
    - **具体的なワークフローを指示している場合**: 対象分野のベストプラクティスを WebSearch/WebFetch で調査し、提示されたワークフローに問題・改善点がないかを提案しつつ確認する
    - **具体的なワークフローを指示していない場合**: 対象分野のベストプラクティスを WebSearch/WebFetch で調査し、最適なワークフローを提案する
    - *(WebSearch/WebFetch が失敗・利用不可の場合は Claude の学習済み知識を使用し、その旨をユーザーに通知する)*
  - `.claude/agents/*.md` を Glob でスキャンして既存 Sub Agent 一覧を取得する（Phase 6 で再利用するためリストを保持する）
  - `.claude/skills/*/SKILL.md` を Glob でスキャンして既存 Skills 一覧を取得する（同上）
  - 状態ファイルにユーザー要求と既存リソース情報を記録する
- 期待成果物: 聞き取り済みのユーザー要求・ベストプラクティス調査結果・既存 Sub Agent / Skills のリスト
- 完了基準: ワークフロー設計に必要な情報が揃い、ベストプラクティス調査と既存リソースのスキャンが完了している

---

## Phase 3: ワークフロー提案・確立 → Gate A

- 実行内容: Phase 2 の情報を基にワークフロー構造を設計し、ユーザーに提案する
- 専門性:
  - フェーズ数は必要最小限（3〜8 フェーズを目安）とする
  - 各フェーズに明確な入力・出力・完了基準を設ける
  - 承認ゲートは破壊的変更前・設計確定の場面にのみ配置する（読み取り・分析フェーズは不要）
  - 提案フォーマット:
    ```
    ## 提案ワークフロー: {ワークフロー名}
    | Phase | フェーズ名 | 概要 | 承認ゲート |
    |-------|-----------|------|-----------|
    | 1 | ... | ... | なし/Gate X |
    ```
  - 懸念点・代替案があれば合わせて提示する
  - AI 側の懸念が解消されるまでユーザーと対話を繰り返す
- 期待成果物: ユーザーへのワークフロー提案（フェーズ一覧テーブル形式）
- 完了基準: Gate A を通過して AI・ユーザー双方がワークフロー構造に懸念なく合意している

---

## Phase 4: スコープ確認 → Gate B

- 実行内容: 設計したワークフロー自体のカバー範囲をユーザーと確定する
- 専門性:
  - AI から積極的に以下の軸でユーザーに問いかける（「言われてみれば必要だった」フェーズを引き出す）:
    - **終了条件**: コミットまで？PR作成まで？デプロイまで？モニタリングも？
    - **エラーハンドリング・ロールバックフェーズ**: 失敗時の復旧フローは必要か？
    - **承認・レビューフェーズ**: 人間が介在する承認ステップは必要か？
    - **通知・レポートフェーズ**: Slack通知・レポート生成は必要か？
  - ユーザーが「そこまでは不要」と答えるまで各軸を確認し続ける
  - 確定したスコープを状態ファイルに記録する
- 期待成果物: ワークフロー全フェーズ・終了条件が確定したスコープ定義
- 完了基準: Gate B を通過して設計したワークフローの全カバー範囲について AI・ユーザー双方が懸念なく合意している

---

## Phase 5: Phase-SubAgent-Skill マッピング提案

- 実行内容: `skills/workflow-creator/references/workflow-template.md` を Read し、各フェーズへの Sub Agent・Skills の割り当てを設計する
- 専門性:
  - テンプレートに従いマッピング表を作成する
  - **割り当て制約**（必須遵守）:
    - Phase → Sub Agent: **1対1**（各フェーズに担当 Sub Agent は1つ）
    - Sub Agent → Skills: **1対N**（各 Sub Agent は複数の Skills を使用可能）
  - Phase 2 で取得した既存リソースリストを参照し、再利用可能なものを優先する
  - 新規作成が必要な Sub Agent・Skills を識別し、その役割・tools を設計する
  - 命名規則を遵守する:
    - Sub Agent: 役割・人物名（`backend-developer` 形式、ケバブケース）
    - Skill: 行為・能力名（`skill-creator` 形式、ケバブケース）
  - マッピング表をユーザーに提示して確認を求める
- 期待成果物: フェーズ別 Sub Agent・Skills マッピング表（1:1・1:N 制約遵守）
- 完了基準: 全フェーズに Sub Agent / Skills の割り当てが完了し、1:1・1:N 制約を満たしている

---

## Phase 6: 既存 Sub Agent・Skill 重複確認

- 実行内容: Phase 2 で取得した既存リソースリストと Phase 5 のマッピングを照合し、重複・類似を排除する
- 専門性:
  - 新規作成予定の Sub Agent について、`.claude/agents/` の既存エージェントと役割の重複がないか確認する
  - 新規作成予定の Skill について、`.claude/skills/` の既存スキルと機能の重複がないか確認する
  - 重複が見つかった場合は既存リソースへの参照に切り替える（新規作成を回避する）
  - 類似だが異なる役割の場合は差異を明記し新規作成を正当化する
- 期待成果物: 重複除去済みの最終マッピング（新規作成リストと既存利用リスト）
- 完了基準: 全新規作成リソースに重複がなく、既存リソース利用が最大化されている

---

## Phase 7: マッピング確立 → Gate C

- 実行内容: Phase 5〜6 の結果を整理してユーザーに最終マッピングを提示する
- 専門性:
  - 最終マッピング表（フェーズ / Sub Agent / Skills / 既存利用 or 新規作成）をユーザーに提示する
  - 新規作成ファイル数とパスを明示する
  - ユーザーの懸念点に回答し必要に応じて調整する
  - AI 側の懸念が解消されるまでユーザーと対話を繰り返す
  - 状態ファイルに確定マッピングと生成予定ファイル一覧を記録する
- 期待成果物: 確定した Phase-SubAgent-Skills マッピング表
- 完了基準: Gate C を通過して AI・ユーザー双方がマッピングに懸念なく合意し状態ファイルに記録されている

---

## Phase 8: ワークフロースキル生成

- 実行内容: `.claude/skills/skill-review/references/skill-creator-guidelines.md` を Read し、ワークフロースキルの SKILL.md を生成する
- 専門性:
  - ガイドラインの「ワークフロースキル固有の要件」を遵守する:
    - `name` が `workflow-` プレフィックスで始まること
    - フェーズ一覧テーブルを持つこと
    - 承認ゲートに「確認内容 / APPROVE時 / REJECT時」を明記すること
    - 状態ファイルパスが `.workflow/state/<name>.md` 形式であること
    - スキル呼び出しが `skill-name` 形式（`/skill-name` は誤り）であること
  - 各フェーズの定義には以下を必ず含める:
    - 詳細（実行内容・専門性）
    - 担当 Sub Agent
    - Sub Agent が使用する Skills
    - **Sub Agent に指示するプロンプト**（具体的な指示内容を明記）
  - 生成するワークフロースキルでは各フェーズの Sub Agent が使用する Skills をフロントマターで明示する（OSS代替への差し替えがフロントマター変更のみで完結するよう設計）
  - 500 行以内に収め、詳細は references/ に分離する
  - 参照パスはプロジェクトルート相対の完全パスを使用する
  - 生成前にファイルの存在を Glob で確認し、存在する場合はユーザーに上書き確認をする
- 期待成果物: 新規ワークフロースキルの SKILL.md
- 完了基準: SKILL.md が生成され、ガイドラインのワークフロースキル固有要件を満たしている

---

## Phase 9: ワークフロースキル AI フィードバック → Gate D

- 実行内容: 生成した SKILL.md に対して `skill-review` を実行し、品質を検証する
- 専門性:
  - `.claude/skills/skill-review/SKILL.md` の存在を Glob で確認する
  - 存在しない場合: `.claude/skills/skill-review/references/skill-creator-guidelines.md` を参照して `skill-review` を新規作成してから呼び出す
  - `skill-review` を Phase 8 で生成した SKILL.md を対象として呼び出す
  - skill-review のレポートで Critical / Important が 0 件になるまで Phase 8 に戻り修正する
  - Nice to have は Phase 8 修正の際に可能な範囲で対応する
- 期待成果物: Critical ゼロ・Important ゼロの品質検証済み SKILL.md
- 完了基準: Gate D を通過して skill-review が Critical / Important 0 件を報告している

---

## Phase 10: Sub Agent 作成

- 実行内容: `skills/workflow-creator/references/subagent-checklist.md` を Read し、Phase 7 で確定した新規 Sub Agent を作成する
- 専門性:
  - 各 Sub Agent の担当分野における最新の専門知識を WebSearch/WebFetch で調査する
    - *(調査が失敗・利用不可の場合は Claude の学習済み知識を使用し、その旨をユーザーに通知する)*
  - `skills/workflow-creator/references/subagent-checklist.md` 記載のベストプラクティス以外のプラクティスも精査・整理する
  - 精査した専門知識を Sub Agent の body に構造的かつ簡潔に反映する（重要な情報が欠如しないこと）
  - 各 Sub Agent の frontmatter に使用 Skills を明示する（OSS代替への差し替えを想定した設計）:
    ```yaml
    skills:
      - skill-name-1
      - skill-name-2
    ```
  - 各 Sub Agent を `.claude/agents/<name>.md` に Write する
  - 作成前にファイルの存在を Glob で確認し、存在する場合はユーザーに上書き確認をする
  - 複数の Sub Agent がある場合は1ファイルずつ順番に作成し、各作成後にチェックリストを適用する
- 期待成果物: 専門知識を反映した Skills 明示済みの新規 Sub Agent ファイル群
- 完了基準: 新規作成対象の全 Sub Agent が生成され、チェックリスト基準と専門知識反映基準を満たしている

---

## Phase 11: Sub Agent AI フィードバック

- 実行内容: 作成した Sub Agent ファイルに対して `agent-review` を実行し、品質を検証する
- 専門性:
  - `.claude/skills/agent-review/SKILL.md` の存在を Glob で確認する
  - 存在しない場合: `.claude/skills/agent-review/references/agent-guidelines.md` を参照して `agent-review` を新規作成してから呼び出す
  - `agent-review` を Phase 10 で作成した全 Sub Agent ファイルを対象として呼び出す
  - Critical / Important が報告された場合は該当ファイルを修正して再検証する
  - 修正後は `agent-review` を再実行して改善を確認する
- 期待成果物: Critical ゼロ・Important ゼロの品質検証済み Sub Agent ファイル群
- 完了基準: 全 Sub Agent ファイルの agent-review が Critical / Important 0 件を報告している

---

## Phase 12: Skills 作成

- 実行内容: `skills/workflow-creator/references/skills-checklist.md` を Read し、Phase 7 で確定した新規 Skills を作成する
- 専門性:
  - 各 Skill の対象分野における最新の専門知識を WebSearch/WebFetch で調査する
    - *(調査が失敗・利用不可の場合は Claude の学習済み知識を使用し、その旨をユーザーに通知する)*
  - `skills/workflow-creator/references/skills-checklist.md` 記載のベストプラクティス以外のプラクティスも精査・整理する
  - 専門分野がさらに細分化できる場合（言語別・フレームワーク別など）は `references/` に分離し、フロントマター変更のみで切り替え可能な呼び出し形式を設計する
  - 各 Skill を `.claude/skills/<name>/SKILL.md` に Write する
  - チェックリストの全項目を満たすことを確認する
  - 作成前にファイルの存在を Glob で確認し、存在する場合はユーザーに上書き確認をする
  - 参照ファイルが必要な場合は `.claude/skills/<name>/references/` に配置する
- 期待成果物: 専門知識を反映した新規 Skill ファイル群（サブスキル切り替え設計込み）
- 完了基準: 新規作成対象の全 Skills が生成され、チェックリスト基準と専門知識反映基準を満たしている

---

## Phase 13: Skills AI フィードバック

- 実行内容: 作成した Skills ファイルに対して `skill-review` を実行し、品質を検証する
- 専門性:
  - `.claude/skills/skill-review/SKILL.md` の存在を Glob で確認する（Phase 9 で作成済みの場合はスキップ）
  - 存在しない場合: `.claude/skills/skill-review/references/skill-creator-guidelines.md` を参照して `skill-review` を新規作成してから呼び出す
  - `skill-review` を Phase 12 で作成した全 Skills ファイルを対象として呼び出す
  - Critical / Important が報告された場合は該当ファイルを修正して再検証する
  - 修正後は `skill-review` を再実行して改善を確認する
- 期待成果物: Critical ゼロ・Important ゼロの品質検証済み Skills ファイル群
- 完了基準: 全 Skills ファイルの skill-review が Critical / Important 0 件を報告している

---

## Phase 14: 品質検証フィードバックループ → Gate E

- 実行内容: 全成果物を一括で品質検証し、ユーザーへ最終報告を行う
- 専門性:
  - 生成したワークフロースキルに `skill-review` を実行する（Phase 9 以降に変更があった場合のみ）
  - 作成した全 Skills に `skill-review` を実行する（Phase 13 以降に変更があった場合のみ）
  - 作成した全 Sub Agent に `agent-review` を実行する（Phase 11 以降に変更があった場合のみ）
  - 最終レポートをユーザーに提示する:
    ```
    ## 生成完了レポート
    - ワークフロースキル: {パス} — {品質評価}
    - Sub Agent: {パス一覧と品質評価}
    - Skills: {パス一覧と品質評価}
    - 残課題: {Nice to have 項目一覧}
    ```
  - 状態ファイルを完了ステータスに更新する
- 期待成果物: 最終レポートと完了ステータスの状態ファイル
- 完了基準: Gate E を通過してユーザーが全成果物を承認している

---

## 設計原則

- **素人ユーザー前提**: 積極的にアプローチ・確認。懸念がなくなるまで反復対話する
- **既存リソース優先**: 新規作成前に必ず Phase 2 のスキャン結果を参照し再利用を検討する
- **Progressive Disclosure**: references/ は各フェーズで明示的に Read する（冒頭一括 Read 禁止）
- **冪等性確保**: ファイル生成は各ファイルの直前に存在確認を行い上書きをユーザーに確認する
- **柔軟な組み換え設計**: Sub Agent フロントマターで Skills を明示し、OSS代替への差し替えをフロントマター変更のみで完結させる

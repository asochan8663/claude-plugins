---
name: leak-learner
description: "Zero Leak L5: owner指摘を半自動で学習。hooks自動記録→session-end候補生成→朝レポートowner承認→quality-rules/lessons反映。完全自動にしない理由: 誤学習永続化・Reward Hacking・ルール肥大化の3リスク（Reflexion/METR/Claude Code公式）"
---

# Leak Learner — Zero Leak L5（半自動学習ループ）

owner指摘から学習し、L2（quality-rules）を育てる。**完全自動ではなくowner承認を挟む。**

## 設計思想

```
❌ V2.1: owner指摘 → AI自動でルール追加（3リスク: 誤学習・Reward Hacking・肥大化）
✅ V2.2: owner指摘 → hook自動記録 → session-end候補生成 → owner承認 → 反映
```

**業界根拠**:
- Production AIの74%が人間評価に依存（Cleanlab 2025）
- 自己反省の質はフィードバックの質に依存（Reflexion研究）
- AIが評価基準をハックする（METR 2025）
- 150-200命令超でLLMの遵守率が急落（Claude Code公式）

## 4つのタイミング

### Step 1 (T1): hook自動記録（既に稼働中）
- **仕組み**: learn-from-edits.sh（UserPromptSubmit hook）
- **処理**: owner修正発言をキーワード検出 → corrections.jsonl に蓄積
- **AIの関与**: なし（hookが機械的に実行）

### Step 2 (T2): セッション内適用（AI自発）
- **仕組み**: decision-capture.mdの3ステップ目
- **処理**: owner指摘をSession memoryに即追記 → 同セッション内で再発防止
- **注意**: AI自発性に依存。T1がセーフティネット

### Step 3 (T3): 学習候補生成（session-end hook）
- **仕組み**: session-end.sh に追加済み
- **処理**: corrections.jsonl → learning-candidates.md（候補サマリー）
- **AIの関与**: なし（hookが機械的に実行）
- **ここでは候補を「生成」するだけ。適用はしない**

### Step 4 (T4): owner承認（朝レポート — 人間ゲート）
- **仕組み**: MORNING_REPORT.md Step 6b
- **処理**:
  1. learning-candidates.md を朝レポートで提示
  2. ownerが承認 / 修正 / 却下を判断
  3. 承認分のみ反映:
     - Global rule → quality-rules.yaml に追記
     - Skill lesson → skills/{name}/lessons/ に追記
     - Client lesson → skills/{name}/lessons/{client}.md に追記
     - プロセス改善 → .claude/rules/ に追記
  4. 重複チェック → 既存ルールと重複していないかスキャン
  5. git commit

## owner承認なしにやってよいこと / やってはいけないこと

| 行動 | 承認 |
|------|------|
| corrections.jsonlへの記録 | 不要（hookが自動） |
| Session memoryへの追記 | 不要（同セッション限定・揮発性） |
| learning-candidates.mdの生成 | 不要（hookが自動） |
| **quality-rules.yamlへの追記** | **owner承認必須** |
| **lessons/への永続的追記** | **owner承認必須** |
| **.claude/rules/への追記** | **owner承認必須** |

## 衝突解決ポリシー

```
Client lessons > Skill lessons > Global rules（具体的が優先）
例外: security / hardcode カテゴリは常にGlobal最優先（上書き不可）
```

## 連携

| スキル | 関係 | 説明 |
|--------|------|------|
| quality-rules (L2) | 還流先 | owner承認後にL2のルールを追加 |
| review-guard (L3) | 入力 | L3の捕捉結果を学習候補に含める |
| decision-capture | T2トリガー | owner指摘検出の3ステップ目 |
| session-protocol | T3/T4トリガー | session-endフロー + 朝レポートフロー |
| skill-upgrade | Q10基準 | 各スキルのlessons/存在とleak-learner接続を検証 |
| neo-skill-creator | 初期接続 | 新スキル作成時にleak-learner行を必ず含める |

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| owner承認ゲート必須 | Reflexion研究 / METR 2025 | 永続的ルール変更はowner承認後のみ。誤学習の永続化を防止 |
| 半自動 > 完全自動 | Claude Code公式（150命令問題） | hook自動記録 + owner承認の2段階。完全自動にしない |
| 具体的が優先 | 衝突解決ポリシー | Client lessons > Skill lessons > Global rules |

## Config

| カテゴリ | キー | デフォルト値 | 説明 |
|---------|------|------------|------|
| パス | corrections_file | `.claude/corrections.jsonl` | hook自動記録先 |
| パス | candidates_file | `.claude/learning-candidates.md` | 学習候補サマリー |
| パス | quality_rules | `.claude/quality-rules.yaml` | Global rulesのSSoT |
| 閾値 | max_global_rules | `50` | Global rules上限。超過時はアーカイブ |
| 閾値 | max_instructions | `150` | LLM命令上限（遵守率急落の閾値） |

## セキュリティ

| 項目 | ルール |
|------|--------|
| ルール変更権限 | owner承認なしにquality-rules.yaml / lessons/ / .claude/rules/ を変更しない |
| Session memory | 同セッション限定・揮発性。永続化はT4のowner承認後のみ |

## BLOCKINGゲート

| ステップ | 失敗条件 | 動作 |
|---------|---------|------|
| T1: hook記録 | corrections.jsonl書き込み失敗 | 警告ログ出力。T2以降は続行 |
| T4: owner承認 | ownerが却下 | 該当候補を破棄。quality-rules.yamlに追記しない |
| T4: 重複チェック | 既存ルールと重複 | 重複を報告し、統合 or 破棄をownerに確認 |
| T4: 上限チェック | Global rulesが50件超過 | アーカイブ候補を提示してownerに確認 |

## エスカレーション

| 状況 | 対応 |
|------|------|
| 学習候補が10件以上溜まっている | ownerに確認: 「学習候補が{N}件あります。朝レポートで一括レビューしますか？」 |
| 同じパターンの指摘が3回以上 | ownerに確認: 「同じ指摘が繰り返されています。Global ruleに昇格しますか？」 |
| 既存ルールと矛盾する候補 | ownerに確認: 「既存ルール{ID}と矛盾します。どちらを優先しますか？」 |

## 汎用性

corrections.jsonlのパス・quality-rules.yamlのパス・ルール上限はConfig表で外部化済み。学習ループの仕組み自体は企業固有値なし。

## 他のスキルとの連携

| スキル | 関係 | 説明 |
|--------|------|------|
| quality-rules (L2) | 還流先 | owner承認後にL2のルールを追加 |
| review-guard (L3) | 入力 | L3の捕捉結果を学習候補に含める |
| decision-capture | T2トリガー | owner指摘検出の3ステップ目 |
| session-protocol | T3/T4トリガー | session-endフロー + 朝レポートフロー |
| skill-upgrade | Q10基準 | 各スキルのlessons/存在とleak-learner接続を検証 |
| neo-skill-creator | 初期接続 | 新スキル作成時にleak-learner行を必ず含める |

## やらないこと

- owner承認なしにquality-rules.yamlを変更する
- owner承認なしにlessons/に永続的に書き込む
- スキル内に独自の学習ロジックを組み込む（各スキルはlessons/を持つだけ）
- 150-200命令を超えるルール肥大化を許容する

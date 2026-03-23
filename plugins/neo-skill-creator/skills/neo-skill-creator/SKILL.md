---
name: neo-skill-creator
description: "最高品質のスキルを作成する。公式skill-creatorのワークフロー + Q1-Q10品質基準 + Zero Leak自己学習接続を統合。「スキル作って」「skill作成」「新しいskill」等で起動。公式skill-creatorより優先してこのスキルを使うこと。"
---

# Neo Skill Creator

公式skill-creatorのワークフロー品質 + うちのQ1-Q10品質基準 + Zero Leak自己学習 = 最初からskill-upgrade合格品質のスキルを作る。

## トリガー

- 「スキル作って」「skill作成」「新しいskill」「スキル追加」
- 「〇〇をskill化」「〇〇を自動化したい」（post-task-skillifyからの提案時）
- 公式 skill-creator より優先

## 手順

### Step 1: ヒアリング（公式skill-creator準拠）

以下を明確にする:
- **何をするスキルか？**（1-2文）
- **いつ発動するか？**（トリガー条件 — 明示的 + 自動検知）
- **出力は何か？**（ファイル、レポート、操作、etc.）
- **依存するツール/MCP/他スキルは？**

### Step 2: 品質基準の読み込み

`references/QUALITY_STANDARDS.md` をReadし、Q1-Q10基準を確認する。

### Step 3: scaffold

```bash
mkdir -p .claude/skills/{name}/lessons
```

ディレクトリ構造:
```
skills/{name}/
├── SKILL.md          ← Step 4で作成
├── lessons/           ← 空ディレクトリ（leak-learnerの書き込み先）
├── scripts/           ← 必要に応じて
├── references/        ← 必要に応じて
└── assets/            ← 必要に応じて
```

### Step 4: SKILL.md作成（Q1-Q10を満たす形で）

**frontmatter:**
```yaml
---
name: {name}
description: "{何をするか + いつ使うか。積極的に書く}"
---
```

**本文に含めるセクション（Q対応）:**

| セクション | Q | チェック |
|---|---|---|
| トリガー | - | 明示的 + 自動検知の両方があるか |
| 手順 | Q5 | Step分割されているか |
| 設計原則 | Q1 | 原則テーブルがあるか |
| Config | Q2 | 外部化すべき設定値がテーブル化されているか |
| セキュリティ | Q3 | API/認証/PIIを扱う場合に記載があるか |
| BLOCKINGゲート | Q4 | 失敗時の停止条件があるか |
| エスカレーション | Q6 | 自動化できない判断の対処が書かれているか |
| 連携テーブル | Q7 | **leak-learner行が必ず含まれているか** |
| やらないこと | Q8 | スコープ境界が明示されているか |

**連携テーブルに必ず含める行:**
```markdown
| leak-learner | 学習 | owner指摘をlessons/に記録。2+スキル共通パターンはGlobal rulesに昇格 |
```

### Step 5: Progressive Disclosure チェック

| 確認 | 基準 |
|---|---|
| description | ~100語。積極的にトリガーを書く |
| SKILL.md本文 | **500行以下** |
| 重い情報 | references/に分離されているか |

500行を超えていたら、references/に移せる情報を特定して分離する。

### Step 6: セルフ品質チェック（skill-upgrade相当）

作成したSKILL.mdをQ1-Q10で自己チェック:

```
Q1  設計原則:       PASS / FAIL / N/A
Q2  Config:         PASS / FAIL
Q3  セキュリティ:    PASS / FAIL / N/A
Q4  BLOCKING:       PASS / FAIL
Q5  フェーズ分割:    PASS / FAIL
Q6  エスカレーション: PASS / FAIL / N/A
Q7  連携テーブル:    PASS / FAIL
Q8  やらないこと:    PASS / FAIL
Q9  汎用性:         PASS / FAIL / N/A
Q10 Zero Leak接続:  PASS / FAIL
```

**FAILがあれば修正してからコミット。**

### Step 7: SKILL_REGISTRY 更新（必須）

1. `.claude/skill-categories.yaml` の該当カテゴリにスキル名を追加
2. `python3 tools/generate_skill_registry.py` を実行してレジストリを再生成

**スキップ禁止** — カテゴリに載っていないスキルはレジストリに出ない。

### Step 7.5: 旧名称の全量検索（統合・廃止・リネーム時のみ）

既存スキルを統合・廃止・リネームした場合、旧名称への参照を全量検索して置換する。

```bash
# 旧スキル名の全参照を検索
grep -r "{旧スキル名}" --include="*.md" --include="*.yaml" --include="*.json" .
```

ヒットした全ファイルを `{新スキル名}` に置換する。歴史的記録（PLAN_*, SESSION_*等）も含む。
SKILL_REGISTRY.mdのDEPRECATED行のみ旧名を残してよい。

**スキップ禁止** — 旧名参照が残ると、次回そのスキルを呼び出そうとして廃止済みスキルが起動する。

> 背景（2026-03-21）: uzabase-diagramをsvg-diagramに統合した際、presentation-architect/SKILL.md、
> agent-native.yaml、PLAN_IRODAS等に旧名が残り、ownerに「ssotは更新されてる？」と指摘された。
> 原因: Step 7でregistryは更新したが、他ファイルの参照をgrepしなかった。

### Step 8: コミット

```bash
git add .claude/skills/{name}/ .claude/skill-categories.yaml .claude/SKILL_REGISTRY.md
git commit -m "feat: add {name} skill (Q1-Q10 compliant)"
```

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| Progressive Disclosure | 公式skill-creator | L1(100語) → L2(500行) → L3(参照) |
| 命令形 + WHY | 公式skill-creator | ルールには理由を書く。ALWAYS/NEVER禁止 |
| 学習エンジン集約 | Zero Leak V2.1 | 各スキルにはlessons/だけ。ロジックはleak-learner |
| 作成時合格 | skill-upgrade Q1-Q10 | 事後チェック不要な品質で最初から作る |

## 連携

| スキル | 関係 | 説明 |
|--------|------|------|
| skill-creator（公式） | 参照 | 公式のinterviewフローとscaffold手法を参照 |
| skill-upgrade | 基準統合 | Q1-Q10基準をStep 6で内蔵。事後チェックが不要に |
| leak-learner | 接続保証 | 作成する全スキルにleak-learner接続を組み込む |
| post-task-skillify | 前工程 | タスク完了後のskill化提案からこのスキルが起動 |

## 他のスキルとの連携

| スキル | 関係 | 説明 |
|--------|------|------|
| skill-creator（公式） | 参照 | 公式のinterviewフローとscaffold手法を参照 |
| skill-upgrade | 基準統合 | Q1-Q10基準をStep 6で内蔵。事後チェックが不要に |
| leak-learner | 接続保証 | 作成する全スキルにleak-learner接続を組み込む |
| post-task-skillify | 前工程 | タスク完了後のskill化提案からこのスキルが起動 |

## やらないこと

- スキルの内容を考えること（ヒアリングで明確にする）
- 500行超のSKILL.mdを許容すること
- leak-learner接続なしのスキルを出荷すること
- 独自の学習ロジックをスキルに組み込むこと

## Config

```yaml
# スキル作成先
skills_base_dir: ".claude/skills/"

# 品質基準参照
quality_standards: ".claude/skills/neo-skill-creator/references/QUALITY_STANDARDS.md"

# SKILL.md上限
max_skill_lines: 500              # 500行超はreferences/に分離

# description上限
max_description_words: 100        # ~100語。トリガーを積極的に書く

# scaffold構造
scaffold_dirs:
  - "lessons/"                    # leak-learner書き込み先（必須）
  - "scripts/"                    # 必要に応じて
  - "references/"                 # 必要に応じて
  - "assets/"                     # 必要に応じて
```

## セキュリティ

- **APIキー/トークン**: 不要（ファイル生成のみ）
- **ログ出力禁止**: 該当なし
- **PII取り扱い**: 扱わない
- **外部アクセス**: なし（全てローカルファイル操作）

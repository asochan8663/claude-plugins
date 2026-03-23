---
name: research-organizer
description: >
  リサーチフォルダの肥大化を検知し、テーマ別サブフォルダへの整理を提案・実行するスキル。
  以下のトリガーで起動:
  (1)「research整理して」「フォルダ散らかってる」「ファイル多すぎ」等のフォルダ整理リクエスト
  (2) check_structure.py がフォルダ肥大化（10件超）を検知した場合
  (3)「リサーチまとめて」「フォルダ分類して」等の分類リクエスト
triggers:
  - "整理して"
  - "散らかってる"
  - "ファイル多すぎ"
  - "フォルダ分類"
  - "research整理"
tools: Read, Write, Edit, Bash, Glob, Grep
---

# Research Organizer スキル

## 概要

`${RESEARCH_DIR}/` 配下のフォルダ肥大化を検知し、テーマ別サブフォルダへの再分類を提案・実行する。
ファイル移動は `git mv` で行い、git履歴を保持する。

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| 10ファイルルール | file-placement.md | 同一フォルダ直下に10件超 → サブフォルダ分類必須 |
| git mv保持 | git best practice | ファイル移動は必ず `git mv` で実行。履歴を保持 |
| owner承認必須 | 意思決定プロセス | 分類案を提示し、owner承認後に移動実行。勝手に動かさない |
| 重複検知 | DRY原則 | 類似ファイル名・内容の重複を検知して報告 |

## 実行フロー

### Step 1: 肥大化スキャン

対象ディレクトリのファイル数をカウント:

```bash
for dir in ${RESEARCH_DIR}/*/; do
  count=$(find "$dir" -maxdepth 1 -type f -name '*.md' | wc -l | tr -d ' ')
  echo "$(basename "$dir")/: ${count}件"
done
# トップレベルの散らかりも検出
find ${RESEARCH_DIR}/ -maxdepth 1 -type f -name '*.md' ! -name 'CLAUDE.md' | wc -l
```

判定:
- 10件超のフォルダ → 分類対象
- トップレベルに3件以上の散らかり → 分類対象
- 全フォルダ10件以下 & トップレベル散らかりなし → 「整理不要です」で終了

### Step 2: テーマ分類

各ファイルの先頭20行を Read し、テーマを自動判定:

判定ルール:
1. ファイル名プレフィックスで一次分類（`RESEARCH_`, `PLAN_`, `STRATEGY_`, `BUSINESS_`, `LLC_`）
2. タイトル・概要文のキーワードで二次分類
3. 既存サブフォルダのテーマと照合

標準テーマカテゴリ（参考。状況に応じて調整）:

| テーマ | キーワード例 | フォルダ名 |
|--------|------------|-----------|
| 法人化・LLC | LLC, 法人, incorporation, 設立 | `incorporation/` |
| MVV | MVV, ミッション, ビジョン, バリュー | `mvv/` |
| 開発手法 | 開発, CI, PR, workflow, principles | `dev-practices/` |
| 事業計画 | 事業, business plan, gap analysis | `business-plan/` |
| 採用 | 採用, recruitment, license, パートナー | `recruitment/` |
| AI動向 | AI, agent, Claude, LLM | `ai-trends/` |
| SNS | Twitter, X, Threads, SNS, 投稿 | `x-twitter/` |

### Step 3: 重複検知

類似ファイル名ペアを検出:

```bash
# ファイル名の類似度チェック（同じ単語を3つ以上共有するペア）
find ${RESEARCH_DIR}/ -name '*.md' -exec basename {} \; | sort
```

重複候補があれば:
- 両ファイルの先頭20行を Read して比較
- 「重複」「補完関係」「別物」のいずれかを判定
- 結果をownerに報告

### Step 4: 分類案をownerに提示

```
【research/ 整理提案】

■ 肥大化フォルダ:
- strategy/: 20件 → 5サブフォルダに分類

■ 分類案:
| 移動元 | 移動先 | ファイル数 |
|--------|--------|-----------|
| strategy/LLC_*.md | incorporation/ | 4件 |
| strategy/MVV_*.md | mvv/ | 3件 |
| ... | ... | ... |

■ 重複疑い:
- SOFTWARE_DEVELOPMENT_PRINCIPLES.md (561行) vs PRINCIPLES_OF_SOFTWARE_DEVELOPMENT.md (357行)
  → 補完関係。両方残す推奨

この分類で進めてよいですか？
```

**owner承認を待つ。承認なしで移動しない。**

### Step 5: 移動実行

owner承認後:

```bash
# 1. 新フォルダ作成
mkdir -p ${RESEARCH_DIR}/{new_folder}/

# 2. git mv で移動（履歴保持）
git mv ${RESEARCH_DIR}/old/FILE.md ${RESEARCH_DIR}/new/FILE.md

# 3. 空フォルダ削除
rmdir ${RESEARCH_DIR}/old/  # 空の場合のみ

# 4. git commit
git add -A ${RESEARCH_DIR}/
git commit -m "refactor: research/ を再分類 (N件移動)"
```

### Step 6: 検証

移動後の確認:

```bash
# 各フォルダのファイル数確認（全て10件以下か）
for dir in ${RESEARCH_DIR}/*/; do
  count=$(find "$dir" -maxdepth 1 -type f -name '*.md' | wc -l | tr -d ' ')
  echo "$(basename "$dir")/: ${count}件"
done

# トップレベルの散らかり確認
find ${RESEARCH_DIR}/ -maxdepth 1 -type f -name '*.md' ! -name 'CLAUDE.md' | wc -l
```

## BLOCKINGゲート

| Step | 失敗条件 | 動作 |
|------|---------|------|
| Step 1 | research/ ディレクトリが存在しない | STOP → パスを確認 |
| Step 1 | 全フォルダ10件以下 & 散らかりなし | STOP → 「整理不要です」 |
| Step 4 | owner承認なし | STOP → 承認を待つ。勝手に移動しない |
| Step 5 | git mv 失敗（未コミット変更あり等） | STOP → git status を確認して報告 |
| Step 6 | 移動後にまだ10件超のフォルダがある | 警告 → 追加分類を提案 |

## エスカレーション

| 状況 | 対応 |
|------|------|
| 分類先が不明なファイル（どのテーマにも合わない） | ownerに確認: 「{ファイル名}の分類先はどこが適切ですか？」 |
| 重複ファイルの処理判断 | ownerに確認: 「統合/削除/両方残すのどれにしますか？」 |
| 20件超の大量移動 | ownerに確認: 「{N}件の移動になります。一括で進めてよいですか？」 |

## Config

| カテゴリ | キー | デフォルト値 | 説明 |
|---------|------|------------|------|
| パス | research_root | `${RESEARCH_DIR}/` | 整理対象のルートディレクトリ |
| 閾値 | max_files_per_folder | `10` | フォルダ肥大化と判定するファイル数 |
| 閾値 | max_toplevel_loose | `2` | トップレベル散らかりと判定するファイル数（CLAUDE.md除く） |
| 除外 | ignore_files | `CLAUDE.md` | スキャン・移動対象外のファイル |
| git | commit_prefix | `refactor:` | コミットメッセージのプレフィックス |

## セキュリティ

| 項目 | ルール |
|------|--------|
| ファイル削除 | 禁止。`git mv` による移動のみ。`rm` / `git rm` は使わない |
| 上書き | 移動先に同名ファイルがある場合は STOP。手動解決を求める |
| バックアップ | git mv のため git reflog で復元可能。追加バックアップ不要 |

## 合成可能性

| 連携スキル | 関係 | トリガー |
|-----------|------|---------|
| verify | 前工程 | verify がフォルダ肥大化を検知 → research-organizer を提案 |
| check_structure.py | 前工程 | check_structure.py の警告をトリガーに起動 |
| incident-triage-lite | 無関係 | インシデントファイルの整理はこのスキルのスコープ外 |
| leak-learner | 学習 | owner指摘をlessons/に記録 |

## 汎用性

整理対象ルートパス（${RESEARCH_DIR}/）とファイル数閾値はConfig表で外部化済み。パスと閾値を変更すれば任意のフォルダ整理に適用可能。

## 他のスキルとの連携

| スキル | 関係 | 説明 |
|--------|------|------|
| verify | 前工程 | verify がフォルダ肥大化を検知 → research-organizer を提案 |
| check_structure.py | 前工程 | check_structure.py の警告をトリガーに起動 |
| incident-triage-lite | 無関係 | インシデントファイルの整理はこのスキルのスコープ外 |
| leak-learner | 学習 | owner指摘をlessons/に記録 |

## このスキルがやらないこと

- ファイルの内容編集・統合 — 移動のみ。内容変更はownerの判断
- research/ 以外のフォルダ整理 — スコープは research/ 限定
- ファイル削除・アーカイブ — safe_rm.sh を使う別作業
- 自動実行（owner承認なし） — 必ず分類案を提示して承認を得る
- knowledge/ の整理 — knowledge_maintain.py が担当

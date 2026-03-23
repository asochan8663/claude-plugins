---
name: verify
description: "プロジェクト全体の品質ゲート。構造・タスク・セキュリティ・git・MEMORYを一括検証。「verify」「検証」「ヘルスチェック」「品質チェック」等で起動。定期検診として朝レポートからも呼び出し可能。"
---

# Verify — プロジェクト全体品質ゲート

プロジェクト全体の健全性を一括検証する: 構造チェック → タスク整合性 → セキュリティスキャン → git状態 → MEMORY.md一貫性。

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| 包括的検証 | Defense in Depth | 5つの独立した検証軸で盲点を最小化 |
| 確定的 > LLM | Zero Leak L3設計思想 | スクリプト（check_structure, check_tasks）を優先。LLMは補完 |
| 非破壊 | Read-Only Principle | 検証のみ。修正はowner承認後に別途実行 |
| 定期実行 | Continuous Monitoring | 朝レポート・ship前・定期メンテナンスで呼び出し可能 |

## トリガー

### 明示的トリガー
- 「verify」「検証して」「ヘルスチェック」「品質チェック」「全体チェック」
- `/verify`

### 自動検知トリガー
- 朝レポート実行時（MORNING_REPORT.md Step 3として）
- 長時間セッションの中間チェックポイント
- ship実行前の前工程として（shipのStep 1c/1bと重複する部分はshipが直接実行）

## Config

```yaml
# 検証スクリプト
check_structure_script: "tools/check_structure.py"
check_tasks_script: "tools/check_tasks.py"
repo_health_script: "tools/repo_health_check.py"
health_config: "tools/health_config.json"

# パス
active_tasks_path: "${TASK_TRACKER_PATH}"
memory_path: "MEMORY.md"
user_memory_dir: "~/.claude/projects/${PROJECT_PATH}/memory/"
knowledge_dir: "${KNOWLEDGE_DIR}/"
decisions_dir: "${KNOWLEDGE_DIR}/"
skills_dir: ".claude/skills/"

# 閾値
max_open_prs: 5                     # 未マージPR数の警告閾値
stale_branch_days: 7                # 古いブランチの警告閾値
max_knowledge_files: 50             # knowledge/の肥大化警告閾値
max_skill_lessons_age_days: 30      # lessons/の棚卸し推奨閾値
```

## 手順

### Step 0: インフラ健全性（最初に実行）

全チェッカーを一括実行し、8カテゴリのスコアを取得する:

```bash
python3 tools/repo_health_check.py
```

これが構造・タスク・ナレッジ・セキュリティ・Git衛生・ドキュメント鮮度・SSoT・Claude Configの8カテゴリを一括チェックする。
S (9.0+) ならStep 1-5はサマリー報告のみ。A以下なら詳細チェックに進む。

追加で以下も実行:

```bash
# Hook構文チェック
for h in .claude/hooks/*.sh; do bash -n "$h" || echo "SYNTAX ERROR: $h"; done

# JSON妥当性チェック
python3 -c "import json; json.load(open('.claude/settings.json')); json.load(open('tools/health_config.json')); print('json: VALID')"

# サブモジュール状態（stuck rebase検知）
# cd ${SHARED_REPO} && git status 2>&1 | head -3
```

### Step 1: 構造検証（確定的チェッカー）

```bash
python3 tools/check_structure.py
```

検証項目:
- フォルダ命名規約（biz_, hq_, _プレフィックス）
- フォルダ肥大化（同一階層にファイル10個以上）
- VIDEO_XXXの構造（script/, audio/, docs/）
- symlink整合性（script/latest → 最新v{N}）

### Step 2: タスク整合性（確定的チェッカー）

```bash
python3 tools/check_tasks.py
```

検証項目:
- ACTIVE_TASKS.mdのフォーマット整合性
- 成果物パスの実在確認（`test -f`）
- IN_PROGRESSタスクの滞留検出
- DONEタスクの成果物未記入

### Step 3: セキュリティスキャン（軽量版）

shipの完全版security-auditとは異なり、**高速な軽量スキャン**を実行:

#### 3a: シークレットパターン検出
```bash
# 主要パターンのみ（security-auditの2aのサブセット）
Grep: "AIzaSy|xoxb-|ntn_|BEGIN PRIVATE KEY|ghp_|sk-|AKIA" --type py,ts,js,sh
```

#### 3b: .gitignore漏れ
```bash
# 機密ファイルがgit管理下にないか
git ls-files | grep -E '\.env$|\.slack_token|\.slack_cookie|credentials'
```

#### 3c: 新規ファイルのgit追跡確認
```bash
git status --short
# Untrackedファイルにスクリプト・設定ファイルがないか
```

### Step 4: Git状態検証

#### 4a: ブランチ状態
```bash
git branch -a                        # 全ブランチ一覧
git log origin/main..main            # push前のコミット
```

#### 4b: 未マージPR確認（共有リポ）
```bash
# cd ${SHARED_REPO} && gh pr list --state open --json number,title,createdAt 2>/dev/null
```

#### 4c: worktree状態
```bash
git worktree list                    # 既存worktreeの確認
# staleなworktreeの検出
```

#### 4d: サブモジュール整合性
```bash
git submodule status
```

### Step 5: MEMORY.md一貫性チェック（LLM判断）

#### 5a: MEMORY.mdの存在と構造
- user memory (`~/.claude/projects/.../memory/MEMORY.md`) の確認
- 参照先ファイル（feedback_*.md等）の実在確認

#### 5b: knowledge/の健全性
```bash
# knowledge/のファイル数
/bin/ls ${KNOWLEDGE_DIR}/ | wc -l

# 古いDECISION/INCIDENTの棚卸し
# 30日以上前のファイルで、まだアーカイブされていないもの
```

#### 5c: スキルlessons/の確認
```bash
# 全スキルのlessons/ディレクトリ存在確認
for skill in .claude/skills/*/; do
  test -d "${skill}lessons" || echo "MISSING: ${skill}lessons/"
done
```

### Step 6: レポート生成

```markdown
## Verify レポート — {YYYY-MM-DD}

### 構造チェック
- check_structure.py: {PASS/FAIL — 詳細}

### タスク整合性
- check_tasks.py: {PASS/FAIL — 詳細}
- IN_PROGRESSタスク: {N}件
- 成果物未記入: {N}件

### セキュリティ（軽量スキャン）
- シークレット検出: {N}件
- .gitignore漏れ: {N}件
- 未追跡ファイル: {N}件

### Git状態
- 未pushコミット: {N}件
- 未マージPR: {N}件
- staleブランチ: {N}件
- worktree: {N}個

### MEMORY / Knowledge
- knowledge/ファイル数: {N}
- スキルlessons/不足: {N}スキル

### 総合判定: {HEALTHY / WARNING / CRITICAL}
```

### Step 7: 結果報告

問題がある場合のみ詳細を報告。全てPASSなら簡潔に:

```
verify完了: HEALTHY
- 構造: PASS
- タスク: PASS
- セキュリティ: PASS
- Git: PASS (未マージPR {N}件)
- MEMORY: PASS
```

## BLOCKINGゲート

| Step | 失敗条件 | 動作 |
|------|---------|------|
| Step 1 | check_structure.pyが存在しない | STOP → 「tools/check_structure.pyが見つかりません」 |
| Step 2 | check_tasks.pyが存在しない | STOP → 「tools/check_tasks.pyが見つかりません」 |
| Step 3a | CRITICALなシークレット露出を検出 | STOP → ownerに即時報告。security-auditの完全実行を推奨 |
| Step 4b | gh CLIが使えない | スキップして報告 → 「gh CLIが利用できません。PR確認をスキップしました」 |

## セキュリティ

| 項目 | ルール |
|------|--------|
| スキャン範囲 | 軽量スキャン（主要パターンのみ）。完全スキャンはsecurity-auditに委譲 |
| 検出値の扱い | パターン名+ファイル:行番号のみ報告。値そのものを出力しない |
| レポート保存 | verifyレポートはgit commitしない（一時的な検証結果） |

## エスカレーション

| 状況 | 対応 |
|------|------|
| CRITICALなシークレット露出 | ownerに即時報告: 「{ファイル}にシークレットが露出しています。即時対応が必要です」 |
| check_structure.pyが大量の違反を報告 | ownerに確認: 「構造違反が{N}件あります。優先して修正しますか？」 |
| 未マージPRが閾値超え | ownerに確認: 「未マージPRが{N}件あります。整理しますか？」 |
| ACTIVE_TASKS.mdの不整合多数 | ownerに確認: 「タスク管理に{N}件の不整合があります。一括修正しますか？」 |

## 他のスキルとの連携

| 連携スキル | 関係 | トリガー |
|-----------|------|---------|
| ship | 補完 | shipは出荷時点検、verifyはプロジェクト全体の定期検診 |
| safety-net | 共有 | Step 1-3の確定的チェッカーはsafety-netと手法を共有 |
| security-audit | 委譲 | 軽量スキャンでCRITICAL検出時、完全版security-auditを推奨 |
| pre-check | 共有 | Step 2のタスク整合性チェックはpre-checkの影響範囲調査と補完関係 |
| weekly-kpi | 後工程 | verifyの結果がKPIレポートの品質指標に反映される |
| leak-learner | 学習 | owner指摘をlessons/に記録。2+スキル共通パターンはGlobal rulesに昇格 |

## このスキルがやらないこと

- 問題の修正（検出と報告のみ。修正はowner承認後に別途実行）
- 完全なセキュリティ監査（それはsecurity-auditの仕事）
- 出荷判定（それはshipの仕事）
- ルールの作成・変更（それはleak-learnerの仕事）
- レポートのgit commit（一時的な検証結果であり永続化不要）

## 汎用性（Portability）

このスキルは **フレームワーク（汎用）+ プロファイル（企業固有）** の構造。
他社は検証スクリプトのパスとタスク管理ファイルを変更するだけで使える。

### カスタマイズ可能な項目
- `check_structure_script` / `check_tasks_script`: 品質チェックスクリプト
- `active_tasks_path`: タスク管理ファイル
- `knowledge_dir` / `skills_dir`: ナレッジ・スキルの配置先
- `max_open_prs` / `stale_branch_days`: 警告閾値

### 他社の導入手順
1. Configセクションのパスを自社に合わせる
2. check_structure.py / check_tasks.py相当のスクリプトを用意する（なくてもスキップして動作）
3. スキルを実行

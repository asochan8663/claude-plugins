---
name: project-checkup
description: "プロジェクト全体の健康診断。Git状態・未マージPR・セキュリティを一括検証。Project-wide health check: git status, open PRs, security scan. 「health-check」「verify」「ヘルスチェック」「品質チェック」等で起動。"
triggers:
  - "health-check"
  - "verify"
  - "ヘルスチェック"
  - "品質チェック"
  - "全体チェック"
  - "検証して"
---

# Health Check — プロジェクト健康診断スキル

プロジェクト全体の健全性を一括検証する: Git状態 → 未マージPR → セキュリティスキャン → 追加チェック → レポート。

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| 包括的検証 | Defense in Depth | 複数の独立した検証軸で盲点を最小化 |
| 非破壊 | Read-Only Principle | 検証のみ。修正はユーザー承認後に別途実行 |
| 定期実行 | Continuous Monitoring | 朝の確認・作業前・定期メンテナンスで呼び出し可能 |
| Config-Driven | 12-Factor App | プロジェクト固有のチェックはConfigで追加 |

## Config

```yaml
# 追加チェック
checks: []                             # 追加チェックスクリプト（例: ["tools/check_structure.py", "tools/lint.sh"]）

# オプション機能
check_memory: false                    # MEMORY.mdチェックを有効にするか
check_tasks: false                     # タスク管理チェックを有効にするか
task_tracker: ""                       # タスク管理ファイルパス（check_tasks=trueの場合に必要）

# 閾値
max_open_prs: 5                        # 未マージPR数の警告閾値
stale_branch_days: 7                   # 古いブランチの警告閾値（日数）

# パス（オプション）
memory_path: ""                        # MEMORY.mdのパス（check_memory=trueの場合に必要）
```

**Configが空の場合**: git status + 未マージPR確認 + セキュリティスキャン。追加チェック・MEMORY・タスク管理は全てスキップ。

## 手順

### Step 1: Git状態検証

#### 1a: 基本状態
```bash
git status --short                     # 未コミット・未追跡ファイル
git stash list                         # stashの確認
git log origin/main..main 2>/dev/null  # push前のコミット
```

#### 1b: ブランチ状態
```bash
git branch -a                          # 全ブランチ一覧
git worktree list 2>/dev/null          # 既存worktreeの確認
```

stale_branch_daysを超えるブランチがあれば警告。

#### 1c: サブモジュール状態（サブモジュールが存在する場合のみ）
```bash
git submodule status 2>/dev/null
```

### Step 2: 未マージPR確認

```bash
gh pr list --state open --json number,title,createdAt 2>/dev/null
```

gh CLIが使えない場合はスキップ。
max_open_prsを超えていたら警告。

### Step 3: セキュリティスキャン

security-scanスキルを呼び出す（security-scanがインストールされている場合）。

security-scanがない場合、軽量スキャンを実行:

#### 3a: シークレットパターン検出
```bash
# 主要パターンのみ
git ls-files | xargs grep -lE '(AIzaSy|xoxb-|ghp_|sk-|AKIA|BEGIN PRIVATE KEY)' 2>/dev/null || true
```

#### 3b: .gitignore漏れ
```bash
# 機密ファイルがgit管理下にないか
git ls-files | grep -iE '\.env$|credentials|\.secret|\.token' || true
```

#### 3c: 未追跡ファイルの確認
```bash
git status --short | grep '^\?\?' | head -20
```

### Step 4: 追加チェック（checksが設定されている場合のみ）

```bash
# checks の各スクリプトを順に実行
python3 {script_path}
# または
bash {script_path}
```

各スクリプトの終了コードとstdoutを記録。

### Step 5: タスク管理チェック（check_tasks=trueの場合のみ）

タスク管理ファイルの整合性を確認:
- フォーマット整合性
- 進行中タスクの滞留検出
- 成果物パスの実在確認（記載されている場合）

### Step 6: MEMORY.mdチェック（check_memory=trueの場合のみ）

- MEMORY.mdの存在確認
- 参照先ファイルの実在確認（リンクが含まれている場合）

### Step 7: レポート生成

```markdown
## Health Check レポート — {YYYY-MM-DD}

### Git状態
- 未コミットファイル: {N}件
- push前コミット: {N}件
- ブランチ数: {N}（うちstale: {N}）
- worktree: {N}個

### 未マージPR
- オープンPR: {N}件

### セキュリティ（軽量スキャン）
- シークレット検出: {N}件
- .gitignore漏れ: {N}件
- 未追跡ファイル: {N}件

### 追加チェック
- {script_name}: {PASS / FAIL}

### 総合判定: {HEALTHY / WARNING / CRITICAL}
```

### Step 8: 結果報告

問題がある場合のみ詳細を報告。全てPASSなら簡潔に:

```
health-check完了: HEALTHY
- Git: PASS（未マージPR {N}件）
- セキュリティ: PASS
- 追加チェック: {PASS / {N}件の問題 / SKIP}
```

**総合判定の基準:**
| 条件 | 判定 |
|------|------|
| 全てPASS | HEALTHY |
| 警告のみ（staleブランチ、未マージPR多数等） | WARNING |
| シークレット露出、CRITICAL脆弱性 | CRITICAL |

## BLOCKINGゲート

| Step | 失敗条件 | 動作 |
|------|---------|------|
| Step 3a | CRITICALなシークレット露出を検出 | STOP → ユーザーに即時報告。security-scanの完全実行を推奨 |
| Step 2 | gh CLIが使えない | スキップして報告 → 「gh CLIが利用できません。PR確認をスキップしました」 |
| Step 4 | 追加チェックスクリプトが存在しない | スキップして報告 → 「{script}が見つかりません。スキップしました」 |

## セキュリティ

| 項目 | ルール |
|------|--------|
| スキャン範囲 | 軽量スキャン（主要パターンのみ）。完全スキャンはsecurity-scanに委譲 |
| 検出値の扱い | パターン名+ファイル:行番号のみ報告。値そのものを出力しない |
| レポート保存 | health-checkレポートはgit commitしない（一時的な検証結果） |

## エスカレーション

| 状況 | 対応 |
|------|------|
| CRITICALなシークレット露出 | ユーザーに即時報告: 「{ファイル}にシークレットが露出しています。即時対応が必要です」 |
| 未マージPRが閾値超え | ユーザーに確認: 「未マージPRが{N}件あります。整理しますか？」 |
| 追加チェックで大量の違反 | ユーザーに確認: 「{N}件の違反があります。優先して修正しますか？」 |

## 他のスキルとの連携

| 連携スキル | 関係 | トリガー |
|-----------|------|---------|
| launch-check | 補完 | launch-checkは出荷時点検、health-checkはプロジェクト全体の定期検診 |
| security-scan | 委譲 | 軽量スキャンでCRITICAL検出時、完全版security-scanを推奨 |
| risk-scan | 共有 | 影響範囲調査と補完関係 |
| leak-learner | 学習 | lessons/がある場合、ユーザー指摘を記録（オプション） |

## このスキルがやらないこと

- 問題の修正（検出と報告のみ。修正はユーザー承認後に別途実行）
- 完全なセキュリティ監査（それはsecurity-scanの仕事）
- 出荷判定（それはlaunch-checkの仕事）
- レポートのgit commit（一時的な検証結果であり永続化不要）
- Configが未設定の項目の強制（空ならスキップ）

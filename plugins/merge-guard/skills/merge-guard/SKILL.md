---
name: merge-guard
description: "作業完了時の品質チェック→マージ→push。セキュリティ・テスト・承認ゲートを自動化。Quality gate before merge: test-runner, security-scan, approval, merge+push. 「launch-check」「ship」「完了」「マージ」「出荷」等で起動。"
triggers:
  - "launch-check"
  - "ship"
  - "完了"
  - "マージして"
  - "出荷"
  - "pushして"
---

# Launch Check — 出荷前品質チェックスキル

作業完了時の品質保証パイプライン: テスト → セキュリティ監査 → ユーザー承認 → mainマージ → push。

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| Shift-Left Quality | OWASP / Shift-Left Testing | マージ前に品質・セキュリティを全て検証。マージ後の手戻りゼロ |
| ユーザー承認ゲート | Fail-Safe Design | pushとPR作成はユーザー承認後のみ。勝手にpushしない |
| 多層検証 | Defense in Depth | テスト + セキュリティ監査で盲点をカバー |
| CRITICALブロック | Fail-Safe Design | CRITICAL脆弱性が1件でもあればマージ拒否。例外なし |

## Config

```yaml
# セキュリティ
block_severity: "CRITICAL"             # この重要度以上でマージ拒否（CRITICAL / HIGH / MEDIUM / LOW）

# マージ
default_target_branch: "main"          # マージ先ブランチ
delete_branch_after_merge: true        # マージ後にブランチを削除するか

# 承認
require_approval: true                 # ユーザー承認必須（falseにすると自動push）

# 追加チェック
pre_merge_checks: []                   # マージ前に実行する追加スクリプト（例: ["tools/check_structure.py"]）
post_merge_hooks: []                   # マージ後に実行するスクリプト（例: ["tools/deploy.sh"]）

# タスク管理
task_tracker: ""                       # タスク管理ファイルのパス（空ならスキップ）
```

**Configが空の場合**: test-runner → security-scan → ユーザー承認 → merge + push。追加チェック・タスク管理は全てスキップ。

## 手順

### Step 1: 出荷前チェック（BLOCKING）

以下を**並列**で実行:

#### 1a: 未コミットファイル確認
```bash
git status
git diff --stat
```
未コミットの変更があれば → STOP → 「未コミットの変更があります。先にコミットしてください」

#### 1b: 追加チェック（pre_merge_checksが設定されている場合のみ）
```bash
# pre_merge_checks の各スクリプトを順に実行
python3 {script_path}
```

### Step 2: テスト実行

test-runnerスキルを呼び出す（test-runnerがインストールされている場合）:
- テストフレームワークの自動検出
- テスト実行 + 結果レポート

test-runnerがない場合:
```bash
# 一般的なテストコマンドを検出して実行
# package.json → npm test
# pytest.ini / pyproject.toml → pytest
# Makefile → make test
# なければスキップ
```

**結果判定:**
| 結果 | 動作 |
|------|------|
| 全テストPASS | Step 3へ |
| テスト失敗 | ユーザーにエスカレーション |
| テストなし | スキップしてStep 3へ |

### Step 3: セキュリティ監査（BLOCKING）

security-scanスキルを呼び出す（security-scanがインストールされている場合）。

security-scanがない場合、軽量スキャンを実行:
```bash
# シークレットパターン検出
git diff --cached --diff-filter=ACM | grep -iE '(password|secret|token|api_key|private_key)\s*[:=]' || true
git diff --cached --diff-filter=ACM | grep -E '(AIzaSy|xoxb-|ghp_|sk-|AKIA)' || true
```

**結果判定:**
| 重要度 | 動作 |
|--------|------|
| block_severity以上 | **STOP** → マージ拒否。即時修正を要求 |
| block_severity未満 | 記録して続行 |

### Step 4: 成果物の実在確認

```bash
# ブランチ上の変更ファイルを一覧
git diff --name-only main...HEAD 2>/dev/null || git diff --name-only HEAD~5..HEAD
```

変更ファイルが0件の場合 → STOP → 「変更がありません。作業内容を確認してください」

### Step 5: ユーザー承認ゲート（require_approval=trueの場合）

```
出荷準備完了:
- テスト: {PASS / FAIL / SKIP}
- セキュリティ: {PASS / CRITICAL {N} / HIGH {N} / MEDIUM {N}}
- 変更ファイル: {N}件
- 追加チェック: {PASS / {N}件の問題 / SKIP}

pushしてよいですか？
```

ユーザー承認を待つ。**承認なしでpushしない。**

### Step 6: マージ + push

#### 6a: マージ
```bash
git checkout {default_target_branch}
git merge {branch}
```

コンフリクトが発生した場合 → STOP → ユーザーに報告

#### 6b: push
```bash
git push origin {default_target_branch}
```

#### 6c: PR作成（リモートブランチの場合）
```bash
git push -u origin {branch}
gh pr create --title "{title}" --body "{body}"
```

### Step 7: クリーンアップ

#### 7a: ブランチ削除（delete_branch_after_merge=trueの場合）
```bash
git branch -d {branch}
```

#### 7b: タスク管理更新（task_trackerが設定されている場合のみ）
該当タスクのステータスをDONEに変更。

#### 7c: マージ後フック（post_merge_hooksが設定されている場合のみ）
```bash
# post_merge_hooks の各スクリプトを順に実行
bash {script_path}
```

#### 7d: 完了報告
```
出荷完了:
- ブランチ: {branch} → {target_branch} にマージ済み
- 変更ファイル: {N}件
- セキュリティ: PASS
```

## BLOCKINGゲート

| Step | 失敗条件 | 動作 |
|------|---------|------|
| Step 1a | 未コミットの変更がある | STOP → コミットを促す |
| Step 1b | 追加チェックで違反 | STOP → 違反を修正してからやり直し |
| Step 3 | block_severity以上が1件以上 | **STOP** → マージ拒否。即時修正必須 |
| Step 4 | 変更ファイルが0件 | STOP → 作業内容を確認 |
| Step 5 | ユーザー未承認 | **STOP** → push禁止。承認を待つ |
| Step 6a | マージコンフリクト | STOP → ユーザーに報告 |

## セキュリティ

| 項目 | ルール |
|------|--------|
| push前チェック | セキュリティスキャンのblock_severity以上 = 0 を必ず検証 |
| シークレット露出 | git diff でシークレットパターンを最終スキャン |
| 強制push禁止 | `git push --force` は絶対に使わない |
| ブランチ保護 | mainへの直接push前にテスト + セキュリティ監査をパス |

## エスカレーション

| 状況 | 対応 |
|------|------|
| block_severity以上が検出された | ユーザーに確認: 「{severity}脆弱性が{N}件。修正してからマージしますか？」 |
| テスト失敗 | ユーザーに確認: 「テストが{N}件失敗。(1) 修正してからship (2) テストスキップしてship」 |
| マージコンフリクト | ユーザーに報告: 「{files}でコンフリクト。手動解決が必要です」 |

## 他のスキルとの連携

| 連携スキル | 関係 | トリガー |
|-----------|------|---------|
| ready | 前工程 | readyで開始したブランチをlaunch-checkで出荷 |
| test-runner | Step 2 | テスト実行をtest-runnerに委譲（インストール済みの場合） |
| security-scan | Step 3 | セキュリティ検証をsecurity-scanに委譲（インストール済みの場合） |
| review-guard | 連携 | review-guardがインストール済みならStep 2で品質検証も実行 |
| rapid-build | 前工程 | rapid-build完了後にlaunch-checkが起動 |
| health-check | 補完 | launch-checkは出荷時点検、health-checkはプロジェクト全体の定期検診 |
| leak-learner | 学習 | lessons/がある場合、ユーザー指摘を記録（オプション） |

## このスキルがやらないこと

- 実装・修正そのもの（それはrapid-buildの仕事）
- 設計書の作成（それはsuper-planの仕事）
- ブランチ作成（それはreadyの仕事）
- プロジェクト全体の定期検診（それはhealth-checkの仕事）
- ユーザー未承認でのpush（require_approval=trueの場合）
- Configが未設定の項目の強制（空ならスキップ）

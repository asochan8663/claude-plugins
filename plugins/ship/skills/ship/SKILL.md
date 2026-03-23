---
name: ship
description: "作業完了。セキュリティチェック → 動作確認 → mainにマージしてpush。「ship」「完了」「マージ」「出荷」等で起動。pushはowner承認後のみ実行。"
---

# Ship — 出荷スキル

作業完了時の品質保証パイプライン: safety-net → security-audit → 動作確認 → mainマージ → push。

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| Shift-Left Quality | OWASP / Shift-Left Testing | マージ前に品質・セキュリティを全て検証。マージ後の手戻りゼロ |
| owner承認ゲート | push-gate (MEMORY.md) | pushとPR作成はowner完了承認後のみ。commitは頻繁に、pushは完成品で |
| 多層検証 | Zero Leak L3 + security-audit | safety-net(確定的+LLM) + security-audit(3並列攻撃)で盲点をカバー |
| CRITICALブロック | Fail-Safe Design | CRITICAL脆弱性が1件でもあればマージ拒否。例外なし |

## トリガー

### 明示的トリガー
- 「ship」「完了」「マージして」「出荷」「pushして」「デプロイ」
- `/ship`

### 自動検知トリガー
- rapid-buildの全タスク完了後
- ブランチ上で全ての品質ゲートがパスした後

## Config

```yaml
# セキュリティ
block_severity: "CRITICAL"           # この重要度以上でマージ拒否
security_report_dir: "docs/security/"

# ブランチ
default_target_branch: "main"
delete_branch_after_merge: true

# 確認
require_ceo_approval_for_push: true  # pushはowner承認必須
require_ceo_approval_for_pr: true    # PR作成もowner承認必須

# パス
active_tasks_path: "${TASK_TRACKER}"
check_structure_script: "tools/check_structure.py"
check_tasks_script: "tools/check_tasks.py"
```

## 手順

### Step 1: 出荷前チェック（BLOCKING）

以下を**並列**で実行:

#### 1a: 未コミットファイル確認
```bash
git status
git diff --stat
```
未コミットの変更があれば → STOP → 「未コミットの変更があります。先にコミットしてください」

#### 1b: ACTIVE_TASKS.md整合性
```bash
python3 tools/check_tasks.py
```
成果物パスの実在確認。不整合があれば修正。

#### 1c: 構造チェック
```bash
python3 tools/check_structure.py
```

### Step 2: safety-net実行（Zero Leak L3）

safety-netスキルを呼び出す:
- 確定的チェッカー（ハードコード検出、デザイン品質、メタデータ確認）
- LLM判断（副作用チェック、完了基準照合）
- リグレッションチェック

**結果判定:**
| 検出結果 | 動作 |
|---|---|
| 問題0件 | Step 3へ進む |
| 確定的違反 | 自動修正 → Step 3へ |
| 設計判断が必要 | ownerにエスカレーション |
| セキュリティ違反 | STOP → Step 3で詳細確認 |

### Step 3: security-audit実行（BLOCKING）

security-auditスキルを呼び出す:
- シークレットパターンスキャン
- 依存関係チェック
- AI攻撃者分析（3並列Agent）

**結果判定:**
| 重要度 | 動作 |
|--------|------|
| CRITICAL | **STOP** → マージ拒否。即時修正を要求 |
| HIGH | 警告 → ownerに確認: 「HIGH脆弱性が{N}件。修正してからマージしますか？」 |
| MEDIUM以下 | 記録して続行 |

### Step 4: 動作確認

#### 4a: テスト実行（テストが存在する場合）
```bash
# auto-testスキルを呼び出し
# プロジェクト固有のテストコマンドを実行
```

#### 4b: 成果物の実在確認
```bash
# 全ての成果物ファイルに対して
test -f {artifact_path}
```

#### 4c: 完了基準の検証
- super-planの設計書がある場合、done_criteriaを全て再検証
- ACTIVE_TASKS.mdの該当タスクが全てDONEであることを確認

### Step 5: マージ判定

#### 5a: 共有リポの場合（${SHARED_REPO}）
```bash
# worktreeからPR作成（owner承認後）
cd .worktrees/{branch}/${SHARED_REPO}/
git push -u origin {branch}
gh pr create --title "{title}" --body "{body}"
```

**Tier判定**: CIのパターン表と照合してTier 1/2を判定
- Tier 2（ドキュメント、設定変更）→ 即時マージ可
- Tier 1（コード変更）→ owner承認後にマージ

#### 5b: 個人リポの場合
```bash
git checkout main
git merge {branch}
```

### Step 6: push（owner承認ゲート）

**owner承認を取得してからpush**:

```
出荷準備完了:
- safety-net: PASS
- security-audit: CRITICAL 0 / HIGH 0 / MEDIUM {N} / LOW {N}
- テスト: {PASS/SKIP}
- 成果物: {ファイルリスト}

pushしてよいですか？
```

owner承認後:
```bash
git push origin main
# または
git push -u origin {branch}  # PR作成の場合
```

### Step 7: クリーンアップ + ACTIVE_TASKS.md更新

#### 7a: ブランチ削除
```bash
# worktreeの場合
cmux rm {branch-name}

# 通常ブランチの場合
git branch -d {branch}
```

#### 7b: ACTIVE_TASKS.md更新
- 該当タスクのステータスをDONEに変更
- 成果物パスを記入
- コミット

#### 7c: 完了報告
```
出荷完了:
- ブランチ: {branch} → main にマージ済み
- 成果物: {ファイルリスト}
- セキュリティ: CRITICAL 0 / HIGH 0
- ACTIVE_TASKS.md: 更新済み
```

## BLOCKINGゲート

| Step | 失敗条件 | 動作 |
|------|---------|------|
| Step 1a | 未コミットの変更がある | STOP → コミットを促す |
| Step 1b | check_tasks.pyで不整合 | STOP → 不整合を修正してからやり直し |
| Step 1c | check_structure.pyで違反 | STOP → 構造違反を修正してからやり直し |
| Step 2 | safety-netでセキュリティ違反 | Step 3で詳細確認 |
| Step 3 | CRITICALが1件以上 | **STOP** → マージ拒否。即時修正必須 |
| Step 4b | 成果物が存在しない | STOP → 成果物を作成してからやり直し |
| Step 6 | owner未承認 | **STOP** → push禁止。承認を待つ |

## セキュリティ

| 項目 | ルール |
|------|--------|
| push前チェック | security-auditのCRITICAL = 0 を必ず検証 |
| シークレット露出 | git diff でシークレットパターンを最終スキャン |
| 強制push禁止 | `git push --force` は絶対に使わない |
| ブランチ保護 | mainへの直接push前にsafety-net + security-auditをパス |

## エスカレーション

| 状況 | 対応 |
|------|------|
| CRITICALが検出された | ownerに確認: 「CRITICAL脆弱性が{N}件。修正方針: {案}。修正してからマージしますか？」 |
| HIGHが3件以上 | ownerに確認: 「HIGH脆弱性が{N}件。全修正に{推定時間}かかります。今やりますか？」 |
| テスト失敗 | ownerに確認: 「テストが{N}件失敗。(1) 修正してからship (2) テストスキップしてship」 |
| マージコンフリクト | ownerに報告: 「{files}でコンフリクト。手動解決が必要です」 |
| Tier 1 PR判定 | ownerに確認: 「コード変更を含むTier 1 PRです。レビュー後にマージしますか？」 |

## 他のスキルとの連携

| 連携スキル | 関係 | トリガー |
|-----------|------|---------|
| start | 前工程 | startで開始したブランチをshipで出荷 |
| safety-net | Step 2 | 品質検証をsafety-netに委譲 |
| security-audit | Step 3 | セキュリティ検証をsecurity-auditに委譲 |
| auto-test | Step 4 | テスト実行をauto-testに委譲 |
| rapid-build | 前工程 | rapid-build完了後にshipが起動 |
| verify | 補完 | shipは出荷時点検、verifyはプロジェクト全体の定期検診 |
| leak-learner | 学習 | owner指摘をlessons/に記録。2+スキル共通パターンはGlobal rulesに昇格 |

## このスキルがやらないこと

- 実装・修正そのもの（それはrapid-buildの仕事）
- 設計書の作成（それはsuper-planの仕事）
- ブランチ作成（それはstartの仕事）
- プロジェクト全体の定期検診（それはverifyの仕事）
- owner未承認でのpush（push-gate違反）
- 共有リポのTier 1 PRの自動マージ（owner承認必須）

## 汎用性（Portability）

このスキルは **フレームワーク（汎用）+ プロファイル（企業固有）** の構造。
他社はセキュリティ閾値とブランチ戦略を変更するだけで使える。

### カスタマイズ可能な項目
- `block_severity`: マージ拒否の閾値
- `require_ceo_approval_for_push`: push承認要否
- `default_target_branch`: マージ先ブランチ
- `check_structure_script` / `check_tasks_script`: 品質チェックスクリプトのパス

### 他社の導入手順
1. Configセクションの値を自社に合わせる
2. safety-net / security-auditスキルが導入済みであることを確認
3. スキルを実行

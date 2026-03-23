---
name: start
description: "新しい機能開発を開始する。ブランチ作成・状態確認・ワークスペース準備を自動化。「start」「開発開始」「新しいブランチ」「作業開始」等で起動。"
---

# Start — 開発開始スキル

新しい機能開発の開始を自動化する: 現在の状態確認 → ブランチ作成 → ワークスペース準備 → 作業開始可能な状態にする。

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| クリーンスタート | Git Best Practices | mainを汚さない。作業は必ずブランチで行う |
| Fail-Fast | Defensive Programming | 未コミットの変更・未マージPRを作業前に検出して止める |
| 共有リポ隔離 | worktree-workflow.md | ${SHARED_REPO}の変更はcmux worktree必須。物理的隔離で事故防止 |
| 影響範囲先行調査 | Zero Leak L1 (pre-check) | 作業前に関連ファイル・過去インシデントを洗い出す |

## トリガー

### 明示的トリガー
- 「start」「開発開始」「新しいブランチ」「作業開始」「始めて」
- `/start {task-name}`

### 自動検知トリガー
- super-plan承認後、rapid-build開始前
- ACTIVE_TASKS.mdのIN_PROGRESSタスクがなく、新しいowner指示がコード変更を伴う場合

## Config

```yaml
# ブランチ命名
branch_prefix_map:
  feature: "feat/"
  bugfix: "fix/"
  docs: "docs/"
  refactor: "refactor/"

# 共有リポ
shared_repo_path: "${SHARED_REPO}"
worktree_base: ".worktrees"

# 状態チェック
max_uncommitted_files: 0          # 未コミットファイルが存在したらSTOP
max_open_prs: 5                    # 未マージPRがこの数を超えたら警告
stale_branch_days: 7               # この日数以上のブランチを警告

# ACTIVE_TASKS
active_tasks_path: "${TASK_TRACKER}"
```

## 手順

### Step 1: 現在の状態確認（BLOCKING）

以下を**並列**で実行:

#### 1a: Git状態チェック
```bash
git status                           # 未コミットの変更
git stash list                       # stashの確認
git log origin/main..main            # ローカルのみのコミット
```

#### 1b: 未マージPR確認（共有リポの場合）
```bash
cd ${SHARED_REPO} && gh pr list --state open --json number,title,createdAt
```

#### 1c: ACTIVE_TASKS.md確認
- IN_PROGRESSタスクの有無
- 前セッションの未完了タスク

#### 1d: 過去インシデント照合
```bash
Grep: "{タスク関連キーワード}" → ${KNOWLEDGE_DIR}/INCIDENT_REPORT_*.md
```

### Step 2: 作業対象の判定

| 変更対象 | 判定 | アクション |
|---------|------|-----------|
| ${SHARED_REPO} | 共有リポ | cmux worktree作成（Step 3a） |
| biz_ai-dev | 独立git | 通常ブランチ作成（Step 3b） |
| ai-company本体 | 個人リポ | 通常ブランチ作成（Step 3b） |
| ドキュメントのみ | main直接 | ブランチ不要（Step 3c） |

### Step 3a: cmux worktree作成（共有リポ）

```bash
# Step 1: cmux CLIでworktree作成
cmux new {branch-name}

# フォールバック（cmux CLI失敗時）
git worktree add .worktrees/{branch-name} -b {branch-name}
cd .worktrees/{branch-name}
git submodule update --init --recursive
```

### Step 3b: 通常ブランチ作成

```bash
# 必ず origin/main から切る（ローカルmainを信用しない）
git fetch origin
git checkout -b {prefix}/{task-name} origin/main
```

### Step 3c: main直接作業

ドキュメント・ナレッジ・タスク管理のみの場合、ブランチ不要。
確認メッセージ: 「ドキュメント変更のみのためmainで作業します」

### Step 4: ACTIVE_TASKS.md登録

```markdown
| {ID} | {タスク名} | IN_PROGRESS | {日付} | - | 開始 |
```

### Step 5: 開始報告

```
開発開始:
- ブランチ: {branch-name}
- 作業対象: {ファイルリスト}
- 過去インシデント: {N}件（該当あれば要約）
- 未マージPR: {N}件
```

## BLOCKINGゲート

| Step | 失敗条件 | 動作 |
|------|---------|------|
| Step 1a | 未コミットの変更がある | STOP → 「未コミットの変更があります。先にコミットしますか？」 |
| Step 1a | ローカルmainにpush前のコミットがある | STOP → 「mainにpush前のコミットがあります。先にpushしますか？」 |
| Step 1b | 未マージPRが5件以上 | 警告 → 「未マージPRが{N}件あります。先にマージしますか？」 |
| Step 1c | IN_PROGRESSタスクが存在 | STOP → 「{タスク名}が進行中です。継続しますか？新タスクを開始しますか？」 |
| Step 3a | cmux worktree作成失敗 | フォールバック → 手動worktree作成を試行 |
| Step 3b | origin/mainのfetch失敗 | STOP → 「リモートに接続できません。オフラインで作業しますか？」 |

## セキュリティ

| 項目 | ルール |
|------|--------|
| 認証情報 | ブランチ名にAPIキー・トークンを含めない |
| .gitignore確認 | 新規ディレクトリ作成時、.gitignoreパターンと照合 |
| 共有リポ権限 | cmux worktreeの権限がmainと同一であることを確認 |

## エスカレーション

| 状況 | 対応 |
|------|------|
| IN_PROGRESSタスクが存在 | ownerに確認: 「{タスク名}が進行中です。(1) 継続 (2) 中断して新タスク開始 (3) 並行作業」 |
| 未マージPRが多数 | ownerに確認: 「未マージPRが{N}件あります。先にマージを整理しますか？」 |
| ブランチ名の判断 | ownerに確認: 「feat/{name} でよいですか？」（曖昧な場合のみ） |
| cmux + 手動worktree両方失敗 | ownerに報告: 「worktree作成に失敗。エラー: {details}。手動対応が必要です」 |

## 他のスキルとの連携

| 連携スキル | 関係 | トリガー |
|-----------|------|---------|
| pre-check | 前工程 | Step 1dで影響範囲・過去インシデントを調査 |
| super-plan | 前工程 | super-plan承認後にstartが自動起動 |
| rapid-build | 後工程 | start完了後にrapid-buildが実装を開始 |
| ship | 後工程 | 作業完了後にshipで品質チェック・マージ・push |
| safety-net | 連携 | Step 1で検出した影響範囲をsafety-netに渡す |
| leak-learner | 学習 | owner指摘をlessons/に記録。2+スキル共通パターンはGlobal rulesに昇格 |

## このスキルがやらないこと

- 実装そのもの（それはrapid-buildの仕事）
- 設計書の作成（それはsuper-planの仕事）
- マージ・push（それはshipの仕事）
- セキュリティ監査（それはsecurity-auditの仕事）
- ドキュメントのみの変更時にブランチを強制すること

## 汎用性（Portability）

このスキルは **フレームワーク（汎用）+ プロファイル（企業固有）** の構造。
他社は共有リポパスとブランチ命名規則を変更するだけで使える。

### カスタマイズ可能な項目
- `branch_prefix_map`: ブランチ命名規則
- `shared_repo_path`: 共有リポのパス
- `worktree_base`: worktreeの配置先
- `active_tasks_path`: タスク管理ファイルのパス

### 他社の導入手順
1. Configセクションのパスを自社に合わせる
2. 共有リポの判定ルール（Step 2のテーブル）を自社構成に合わせる
3. スキルを実行

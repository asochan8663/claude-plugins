---
name: ready
description: "作業開始の準備を自動化。ブランチ作成・状態確認・前回の未完了作業検出。Prepare workspace: branch creation, status check, prior work detection. 「ready」「start」「開発開始」「作業開始」等で起動。"
triggers:
  - "ready"
  - "start"
  - "開発開始"
  - "作業開始"
  - "新しいブランチ"
  - "始めて"
---

# Ready — 作業開始スキル

新しい作業の開始を自動化する: 現在の状態確認 → ブランチ作成 → 作業開始可能な状態にする。

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| クリーンスタート | Git Best Practices | mainを汚さない。作業は必ずブランチで行う |
| Fail-Fast | Defensive Programming | 未コミットの変更・未マージPRを作業前に検出して止める |
| 非破壊 | Read-Only First | 状態確認は読み取りのみ。変更はユーザー承認後 |
| Config-Driven | 12-Factor App | プロジェクト固有の設定はConfigで外部化 |

## Config

```yaml
# ブランチ命名
branch_prefix: "feat/"                 # デフォルトのブランチプレフィックス
branch_prefix_map:                     # タスク種別ごとのプレフィックス（オプション）
  feature: "feat/"
  bugfix: "fix/"
  docs: "docs/"
  refactor: "refactor/"

# 事前チェック
pre_start_checks: []                   # 追加チェックスクリプトのパスリスト（例: ["tools/check_structure.py"]）

# タスク管理
task_tracker: ""                       # タスク管理ファイルのパス（空ならスキップ）

# 閾値
max_uncommitted_files: 0               # 未コミットファイルが存在したらSTOP
max_open_prs: 5                        # 未マージPRがこの数を超えたら警告
stale_branch_days: 7                   # この日数以上のブランチを警告
```

**Configが空の場合**: git branch作成 + git statusだけ実行。追加チェック・タスク管理は全てスキップ。

## 手順

### Step 1: 現在の状態確認（BLOCKING）

以下を**並列**で実行:

#### 1a: Git状態チェック
```bash
git status                           # 未コミットの変更
git stash list                       # stashの確認
git log origin/main..main 2>/dev/null  # ローカルのみのコミット
```

#### 1b: 未マージPR確認
```bash
gh pr list --state open --json number,title,createdAt 2>/dev/null
```
gh CLIが使えない場合はスキップ。

#### 1c: タスク管理ファイル確認（task_trackerが設定されている場合のみ）
- 進行中タスクの有無
- 前セッションの未完了タスク

#### 1d: 追加チェック（pre_start_checksが設定されている場合のみ）
```bash
# pre_start_checks の各スクリプトを順に実行
python3 {script_path}
```

### Step 2: ブランチ作成

#### 2a: タスク名の決定
ユーザーが指定した場合はそのまま使用。未指定の場合はユーザーに確認:
「ブランチ名を指定してください（例: feat/add-login）」

#### 2b: ブランチ作成
```bash
# 必ず origin/main から切る（ローカルmainを信用しない）
git fetch origin
git checkout -b {prefix}/{task-name} origin/main
```

フォールバック（origin/mainが取得できない場合）:
```bash
git checkout -b {prefix}/{task-name}
```

#### 2c: ドキュメントのみの変更
ユーザーが「ドキュメントだけ」と明示した場合、ブランチ作成をスキップしてmainで作業。
確認メッセージ: 「ドキュメント変更のみのためmainで作業します」

### Step 3: タスク登録（task_trackerが設定されている場合のみ）

タスク管理ファイルに新しいタスクを登録。

### Step 4: 開始報告

```
作業開始:
- ブランチ: {branch-name}
- Git状態: {クリーン / 未コミット{N}件}
- 未マージPR: {N}件
- 追加チェック: {PASS / {N}件の問題}
```

## BLOCKINGゲート

| Step | 失敗条件 | 動作 |
|------|---------|------|
| Step 1a | 未コミットの変更がある | STOP → 「未コミットの変更があります。先にコミットしますか？」 |
| Step 1a | ローカルmainにpush前のコミットがある | 警告 → 「mainにpush前のコミットがあります。先にpushしますか？」 |
| Step 1b | 未マージPRがmax_open_prs超え | 警告 → 「未マージPRが{N}件あります。先にマージしますか？」 |
| Step 1c | 進行中タスクが存在 | STOP → 「{タスク名}が進行中です。継続しますか？新タスクを開始しますか？」 |
| Step 2b | origin/mainのfetch失敗 | フォールバック → ローカルmainから分岐 |

## セキュリティ

| 項目 | ルール |
|------|--------|
| 認証情報 | ブランチ名にAPIキー・トークンを含めない |
| .gitignore確認 | 新規ディレクトリ作成時、.gitignoreパターンと照合 |

## エスカレーション

| 状況 | 対応 |
|------|------|
| 進行中タスクが存在 | ユーザーに確認: 「{タスク名}が進行中です。(1) 継続 (2) 中断して新タスク開始 (3) 並行作業」 |
| 未マージPRが多数 | ユーザーに確認: 「未マージPRが{N}件あります。先にマージを整理しますか？」 |
| ブランチ名の判断 | ユーザーに確認: 「feat/{name} でよいですか？」（曖昧な場合のみ） |

## 他のスキルとの連携

| 連携スキル | 関係 | トリガー |
|-----------|------|---------|
| super-plan | 前工程 | super-plan承認後にreadyが自動起動 |
| rapid-build | 後工程 | ready完了後にrapid-buildが実装を開始 |
| launch-check | 後工程 | 作業完了後にlaunch-checkで品質チェック・マージ・push |
| pre-check | 連携 | Step 1で影響範囲・過去インシデントを調査（pre-checkがある場合） |
| leak-learner | 学習 | lessons/がある場合、ユーザー指摘を記録（オプション） |

## このスキルがやらないこと

- 実装そのもの（それはrapid-buildの仕事）
- 設計書の作成（それはsuper-planの仕事）
- マージ・push（それはlaunch-checkの仕事）
- セキュリティ監査（それはsecurity-auditの仕事）
- Configが未設定の項目の強制（空ならスキップ）

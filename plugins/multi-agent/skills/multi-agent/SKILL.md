---
name: multi-agent
description: 大規模タスクを Agent Teams で並列実行するスキル。タスク規模が大きい（5+件の独立タスク、3+フォルダ跨ぎ、並列可能性が高い）場合に自動提案する。「マルチエージェント」「並列実行」「チーム組んで」「Agent Teams」等のトリガー、またはタスク規模の自動検知で起動。
---

# Multi-Agent Skill（Agent Teams 統合版）

大規模タスクを **Claude Code Agent Teams** で並列実行する。
設計（チーム構成・タスク分割）→ 実行（Agent Teams spawn）→ 監視（Delegate Mode）の一気通貫。

## 前提

- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` が settings.json に設定済み
- Claude Code v2.1.32 以降
- 各チームメイトは CLAUDE.md / skills / MCP を自動ロードする

## トリガー

### 明示的トリガー
- 「マルチエージェントで進めて」「並列で実行して」「チーム組んで」
- 「Agent Teams使って」「エージェントチームで」
- `/multi-agent`（旧command互換）

### 自動検知トリガー（以下のいずれかを満たす場合に提案）
- 独立タスクが **5件以上** ある
- 編集領域が **3フォルダ以上** に跨がる
- 並列可能性が高い（タスク間の依存が少ない）
- 「1人で順番にやると1時間以上かかる」規模

**検知時の提案文:**
```
このタスクは Agent Teams で並列実行すると効率的です:
- 独立タスク数: XX件
- 編集領域: XX フォルダ
- 推奨チーム構成: X人

Agent Teams で進めますか？
```

## 手順

### Step 1: タスク分析

ユーザーのリクエストを分析し、以下を特定:

1. **タスクの分解**: 独立して実行可能な単位に分割
2. **依存関係の特定**: どのタスクがどのタスクに依存するか
3. **ファイル所有権の割り当て**: 各チームメイトが編集するファイルセットを排他的に定義

**判断基準:**

| チーム人数 | 適用条件 |
|-----------|---------|
| 2人 | 独立タスク5-10件、2領域 |
| 3人 | 独立タスク10-20件、3領域 |
| 4人（最大推奨） | 独立タスク20+件、4+領域 |

### Step 2: チーム設計書を作成

以下の情報を整理する（ドキュメント作成は必要な場合のみ）:

```markdown
## チーム構成

### チームメイト一覧
| 名前 | 役割 | 担当領域（編集OK） | 編集禁止 | タスク数 |
|------|------|------------------|---------|---------|
| frontend-dev | フロントエンド | src/components/, src/pages/ | src/api/, db/ | 8 |
| backend-dev | バックエンド | src/api/, src/services/ | src/components/ | 7 |
| test-writer | テスト | tests/, __tests__/ | src/ (読み取りのみ) | 6 |

### タスクリスト（依存関係付き）
- [ ] Task 1: ... (依存なし)
- [ ] Task 2: ... (依存なし)
- [ ] Task 3: ... (Task 1 完了後)
```

### Step 3: ユーザー承認

チーム設計をユーザーに提示し、承認を得る:

```
Agent Teams 構成案:

チームメイト3人:
1. frontend-dev: コンポーネント実装（8タスク）
2. backend-dev: API実装（7タスク）
3. test-writer: テスト作成（6タスク）

ファイル競合: なし（排他的所有権）
依存関係: Task 3→1, Task 5→2
推定時間: 並列で約30分（逐次だと1.5時間）

この構成でチームを起動しますか？
```

### Step 4: Agent Teams 起動

承認後、自然言語でチームを作成:

```
Create an agent team with {N} teammates:

Teammate 1 "{name}":
- Role: {役割の詳細}
- Files to edit: {編集OKパス}
- DO NOT edit: {編集禁止パス}
- Tasks:
  1. {具体的タスク}
  2. {具体的タスク}
- When done: Report completion via message to team lead

Teammate 2 "{name}":
...

Task dependencies:
- "{task}" depends on "{other task}" — do not start until dependency is complete

Important rules for ALL teammates:
1. Only edit files in your assigned directories
2. Never edit files owned by another teammate
3. Send a message when you complete each task
4. If you need something from another teammate's area, send them a message
```

### Step 5: Delegate Mode で監視

チーム起動後、即座に **Delegate Mode** を有効化:

1. `Shift+Tab` で Delegate Mode ON
2. リーダーは調整専任（コード書き込み禁止）
3. チームメイトの進捗をタスクリスト（`Ctrl+T`）で監視
4. ブロック発生時はメッセージで解消

### Step 6: 完了・クリーンアップ

全タスク完了後:

1. 各チームメイトの成果物を確認
2. `Clean up the team` でチームを解散（**必ずリーダーから実行**）
3. ACTIVE_TASKS.md を更新
4. git commit

## Task tool（subagent）との使い分け

| 状況 | 使うもの |
|------|---------|
| grep・ファイル検索・調査 | Task tool（subagent） |
| 独立した複数ファイルの同時編集 | **Agent Teams** |
| 協調が必要な開発（フロント+バック） | **Agent Teams** |
| レビュー + 実装の同時進行 | **Agent Teams** |
| 1ファイルの集中作業 | 直接編集（チーム不要） |

**併用OK**: Agent Teams のチームメイト内で Task tool を使うことは可能。

## コンフリクト防止ルール

| ルール | 詳細 |
|--------|------|
| **1ファイル1チームメイト** | 同一ファイルを複数チームメイトが編集しない（ロック機構なし） |
| **排他的ディレクトリ所有** | spawn時に各チームメイトの編集可能ディレクトリを明示 |
| **読み取りは自由** | 他チームメイトのファイルは読み取りOK |
| **変更依頼はメッセージ** | 他チームメイトの領域を変更したい場合はメールボックスで依頼 |

## 既知の制約（experimental）

| 制約 | 対策 |
|------|------|
| コンテキスト圧縮でチーム喪失 | 長時間セッションを避ける。大タスクは分割 |
| セッション再開でチームメイト復元不可 | 中間成果を頻繁にコミット |
| ファイル編集ロックなし | 排他的所有権を厳守 |
| bypassPermissions 不継承の場合あり | チームメイトに手動承認が必要になることがある |
| `.claude/agents/` 定義が無視される | spawn プロンプトに全情報を埋め込む |

## 注意

- チームメイトは **2-4人** が推奨。5人以上はコンフリクトリスクが急増
- spawn プロンプトに**十分なコンテキスト**を含める（チームメイトはリーダーの会話履歴を持たない）
- 完了後は**必ずリーダーからクリーンアップ**する
- 進捗は ACTIVE_TASKS.md で追跡する
- 孤立した tmux セッションが残った場合: `tmux ls` → `tmux kill-session -t <name>`

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| 排他的ファイル所有 | Concurrency制御 | 同一ファイルを複数チームメイトが編集しない。spawn時に所有権を明示的に割り当てる |
| 設計先行 | Google Engineering | チーム構成・タスク分割の設計をownerに承認されてからspawnする。見切り発車しない |
| リーダーは書かない | Delegate Mode | Delegate Mode中、リーダーは調整専任。自らコードを書かず、チームメイトに委任する |

## Config

| カテゴリ | キー | デフォルト値 | 説明 |
|---------|------|------------|------|
| チームサイズ | max_teammates | 4 | 最大推奨チームメイト数。5人以上はコンフリクトリスク急増 |
| 自動検知閾値 | min_independent_tasks | 5 | Agent Teams提案の最低独立タスク数 |
| 自動検知閾値 | min_folders | 3 | Agent Teams提案の最低フォルダ跨ぎ数 |
| 環境 | experimental_flag | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` | settings.json に必要な設定 |
| バージョン | min_version | v2.1.32 | Claude Code最低バージョン |

## セキュリティ

N/A -- 外部通信・認証なし。Claude Code内部のAgent Teams機能のみ使用。

## BLOCKINGゲート

| Step | 失敗条件 | 動作 |
|------|---------|------|
| Step 1 | 独立タスクが5件未満 | STOP + 「Agent Teamsは不要です。直接実行を推奨」と報告 |
| Step 2 | ファイル所有権に重複がある | STOP + 所有権の再設計を要求 |
| Step 3 | owner承認が得られない | STOP + 設計を修正して再提示 |
| Step 4 | Agent Teams実験フラグ未設定 | STOP + 「settings.jsonにCLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1を追加してください」 |

## エスカレーション

| 状況 | 対応 |
|------|------|
| チームメイト間でファイル競合が発生 | ownerに確認: 「競合が発生しました。所有権を再割り当てしますか？」 |
| チームメイトが長時間応答しない | ownerに報告: 「{name}が応答なし。タスクを再割り当てするか、セッションを確認してください」 |
| コンテキスト圧縮でチーム状態が喪失 | ownerに報告: 「チーム状態が喪失しました。中間成果をコミットして新チームを再構成しますか？」 |

## 合成可能性

| 連携スキル | 関係 | トリガー |
|-----------|------|---------|
| auto-test | 後工程 | 全チームメイト完了後にテストを一括実行 |
| /ship | 後工程 | テスト通過後のデプロイ |

## やらないこと

- 1ファイルの集中作業（直接編集で十分）
- grep・ファイル検索のみのタスク（Task tool / subagentで十分）
- owner承認なしのチーム起動
- 5人以上のチーム編成

## 他のスキルとの連携

| スキル | 関係 | 説明 |
|--------|------|------|
| auto-test | 後工程 | 全チームメイト完了後にテストを一括実行 |
| /ship | 後工程 | テスト通過後のデプロイ |
| rapid-build | 呼び出し元 | rapid-buildが並列実行が有益な場合にこのスキルを呼び出す |
| skill-upgrade | 呼び出し元 | --allモードで全スキルを並列処理する場合にこのスキルを利用 |
| leak-learner | 学習 | owner指摘をlessons/に記録。2+スキル共通パターンはGlobal rulesに昇格 |

> 合成可能性セクションの内容を連携テーブルに統合し、leak-learner行を追加。

## 汎用性（Portability）

このスキルは **汎用フレームワーク** — Agent Teamsのチーム設計・タスク分割・排他的ファイル所有・Delegate Mode監視のパターンはプロジェクト非依存。
他社は Config セクションの `experimental_flag`, `min_version` を自社のClaude Code環境に合わせるだけで使える。

タスク分析・チーム設計・コンフリクト防止ルールは全て汎用。企業名ハードコードなし。

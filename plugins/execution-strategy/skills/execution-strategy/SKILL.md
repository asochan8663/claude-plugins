# Execution Strategy Advisor

プラン完了後、最適な実行方法（cmux/Agent Team/Subagent）を自動判定して提案するスキル。

## 使い方

**このskillはリファレンスドキュメントです**。Skill toolで呼び出すのではなく、Claudeがこのドキュメントを読んで手動で評価します。

### ユーザーがこう言ったらClaudeがこのドキュメントを参照する:
- 「どう進める？」「実装開始」「次どうする？」「ベストな方法は？」
- プラン作成完了後に「どの方法で実行するか」を決めたい時
- 「cmux、Agent Team、Subagentのどれがいい？」

### Claudeの作業フロー:
1. ユーザーリクエストを受信
2. このドキュメント（SKILL.md）を Read
3. プランドキュメント（PLAN_*.md）を Read
4. ACTIVE_TASKS.md を Read/Grep
5. 5軸で評価 → スコアリング → 推奨を提示

**重要**:
- このスキルは **推奨を提示するだけ** で、実際の実行は行わない。ユーザーが選択した方法で進める
- Claudeが手動で判定するため、プランやタスクの内容を正確に読み取る必要がある

---

## 判定基準（5軸評価）

| 評価軸 | cmux推奨 | Agent Team推奨 | Subagent推奨 |
|--------|---------|---------------|-------------|
| **タスク数** | 3-15個 | 50+個 | 1-2個 |
| **依存関係** | 部分的にあり | 完全独立 | 単一タスク |
| **期間** | 1-10営業日 | 2週間以上 | 1日以内 |
| **フォルダ跨ぎ** | 2-4個 | 5+個 | 1個 |
| **並列化可能性** | 部分的（30-70%） | 完全（80%以上） | なし |

**スコアリング**: 各軸1-5点、合計25点満点で評価。最高得点の方法を推奨。

---

## 実行フロー

```
1. プランドキュメント（PLAN_*.md）を Read
2. ACTIVE_TASKS.md から関連タスクを抽出（Grep）
3. 5軸で評価（タスク数/依存関係/期間/フォルダ跨ぎ/並列化）
   - プランとタスクの内容を読み取り、各軸の「今回の値」を判定
   - タスク数: プラン内タスク数を数える（## 見出し または ACTIVE_TASKS の TODO 行）
   - 依存関係: Day/Phase構成 → partial、明示的な順序 → sequential、なし → none
   - 期間: プラン内の記載を参照（なければ タスク数 × 0.5日 で推定）
   - フォルダ跨ぎ: プラン内で言及されたフォルダ（biz_*, hq_*）を数える
   - 並列化: Day/Phaseが独立 → 並列可能、依存あり → 部分並列、順序あり → 並列不可
4. スコアリング（cmux/Agent Team/Subagent 各25点満点）
   - 判定ロジック詳細（後述）のPythonコード **を参照して** 各軸の点数を手動で算出
   - **重要**: Pythonコードは実行しない。ロジックを読んで Claude が手動で点数を判定する
   - 例: タスク数8個 → score_task_count(8) を読む → cmux: 4点, agent_team: 2点, subagent: 2点
5. 最高得点の方法を推奨（拮抗時は両方提示）
6. 実行コマンド例を生成（cmux → ブランチ名リスト、Agent Team → チーム構成案）
7. 想定所要時間を算出（逐次 vs 並列 vs Agent Team）
8. ユーザーに推奨を提示（テーブル形式 + 理由 + コマンド例）
```

**スコアリング方法の補足**:
- Pythonコードは **リファレンス** として記載（実行はしない）
- Claude がコードを読んで、手動で各軸の点数を判定する
- 例: 「タスク数8個」→ `score_task_count(8)` を見る → `task_count <= 15` に該当 → cmux=4, agent_team=2, subagent=2

---

## 出力フォーマット

### パターン1: 明確な推奨がある場合（スコア差4点以上）

```markdown
## 実行方法の推奨

### 評価結果

| 評価軸 | 今回の値 | cmux | Agent Team | Subagent |
|--------|---------|------|-----------|----------|
| タスク数 | 8個 | ⭐⭐⭐⭐ (4点) | ⭐⭐ (2点) | ⭐⭐ (2点) |
| 依存関係 | 部分的 | ⭐⭐⭐⭐ (4点) | ⭐⭐ (2点) | ⭐⭐⭐ (3点) |
| 期間 | 5.5日 | ⭐⭐⭐⭐ (4点) | ⭐⭐⭐ (3点) | ⭐⭐ (2点) |
| フォルダ跨ぎ | 2-3個 | ⭐⭐⭐⭐ (4点) | ⭐⭐ (2点) | ⭐⭐⭐ (3点) |
| 並列化 | 50% | ⭐⭐⭐⭐ (4点) | ⭐⭐⭐ (3点) | ⭐⭐ (2点) |
| **合計** | - | **20/25** | **12/25** | **12/25** |

---

### 推奨: **cmux** (Git Worktree 並列実行)

**理由**:
1. **タスク数8個** → cmuxで管理可能な範囲（Agent Teamは50+タスク向け）
2. **依存関係あり**（Day 1 → Day 2 → ...） → Agent Teamより cmux が適切
3. **期間5.5日** → セットアップコスト考慮すると cmux が効率的
4. **並列化可能**（Day 1, Day 2, Day 5） → **1.5日節約可能**

**想定所要時間**:
- 逐次実行（Subagent）: 5.5営業日
- **cmux並列実行**: **4営業日** (1.5日節約) ← 推奨
- Agent Team: セットアップ0.5日 + 実行1-2日 = 1.5-2.5日（ただしリスク高）

---

### 実行コマンド例

```bash
# Day 1（0.5日×2 = 並列で0.5日）
ターミナル① → cmux data/meeting-agent-mock       # ダミーデータ作成
ターミナル② → cmux runbook/meeting-agent        # Runbook作成

# Day 2（1日 + 0.5日 = 並列で1日）
ターミナル① → cmux feature/meeting-agent-agent  # Agent SDK実装
ターミナル② → cmux feature/devmode-fixture-injection  # Fixture注入UI

# Day 3（1日）
ターミナル① → cmux feature/meeting-agent-ui     # Meet拡張機能実装

# Day 4（1日）
ターミナル① → cmux demo/meeting-agent-scenario  # デモシナリオ + Fallback

# Day 5（0.5日×2 = 並列で0.5日）
ターミナル① → cmux test/meeting-agent-demo      # リハーサル
ターミナル② → cmux ops/meeting-agent-playbook   # Playbook作成
```

---

### 次のステップ

1. **cmux で開始する** → `cmux data/meeting-agent-mock` を実行
2. **別の方法を検討する** → Agent Team または Subagent の詳細を確認
3. **実行方法の詳細を確認する** → cmux の使い方、メリット・デメリットを再確認
```

---

### パターン2: 拮抗している場合（スコア差3点以内）

```markdown
## 実行方法の推奨

### 評価結果

| 評価軸 | 今回の値 | cmux | Agent Team |
|--------|---------|------|-----------|
| タスク数 | 50個 | ⭐⭐⭐ (3点) | ⭐⭐⭐⭐⭐ (5点) |
| 依存関係 | 20%独立 | ⭐⭐⭐⭐ (4点) | ⭐⭐⭐ (3点) |
| 期間 | 12営業日 | ⭐⭐⭐ (3点) | ⭐⭐⭐⭐ (4点) |
| フォルダ跨ぎ | 6個 | ⭐⭐⭐ (3点) | ⭐⭐⭐⭐⭐ (5点) |
| 並列化 | 60% | ⭐⭐⭐⭐ (4点) | ⭐⭐⭐⭐ (4点) |
| **合計** | - | **17/25** | **21/25** |

---

### 推奨: **Agent Team** vs **cmux** — 両方検討を推奨

**スコアが拮抗しています（差4点）**。以下の判断基準で選択してください:

| 基準 | cmux を選ぶべき | Agent Team を選ぶべき |
|------|---------------|---------------------|
| リスク許容度 | 低リスク重視 | 高リスク許容（初回試行OK） |
| セットアップ時間 | 即開始したい | 0.5日のセットアップOK |
| 管理スタイル | 自分で進捗確認したい | 完全放置したい |
| 過去実績 | 実績重視 | 新しい方法を試したい |

**想定所要時間**:
- cmux: 8-10営業日（部分並列化）
- Agent Team: 3-5営業日（セットアップ0.5日 + 完全並列実行）

どちらにしますか？
```

---

## 判定ロジック詳細

### タスク数の評価

```python
def score_task_count(task_count):
    if task_count == 1:
        return {"cmux": 1, "agent_team": 1, "subagent": 5}
    elif task_count <= 2:
        return {"cmux": 2, "agent_team": 1, "subagent": 4}
    elif task_count <= 15:
        return {"cmux": 4, "agent_team": 2, "subagent": 2}
    elif task_count <= 30:
        return {"cmux": 3, "agent_team": 3, "subagent": 1}
    else:  # 50+
        return {"cmux": 2, "agent_team": 5, "subagent": 1}
```

### 依存関係の評価

```python
def score_dependencies(dependency_type):
    # dependency_type: "none" (完全独立), "partial" (部分的), "sequential" (完全順序)
    if dependency_type == "none":
        return {"cmux": 3, "agent_team": 5, "subagent": 3}
    elif dependency_type == "partial":
        return {"cmux": 4, "agent_team": 2, "subagent": 3}
    else:  # sequential
        return {"cmux": 3, "agent_team": 1, "subagent": 4}
```

### 期間の評価

```python
def score_duration(days):
    if days <= 1:
        return {"cmux": 2, "agent_team": 1, "subagent": 5}
    elif days <= 5:
        return {"cmux": 4, "agent_team": 2, "subagent": 3}
    elif days <= 10:
        return {"cmux": 4, "agent_team": 3, "subagent": 2}
    else:  # 2週間以上
        return {"cmux": 3, "agent_team": 5, "subagent": 1}
```

### フォルダ跨ぎの評価

```python
def score_folder_spread(folder_count):
    if folder_count == 1:
        return {"cmux": 3, "agent_team": 1, "subagent": 4}
    elif folder_count <= 4:
        return {"cmux": 4, "agent_team": 2, "subagent": 3}
    else:  # 5+
        return {"cmux": 2, "agent_team": 5, "subagent": 1}
```

### 並列化可能性の評価

```python
def score_parallelization(parallel_percent):
    if parallel_percent < 20:
        return {"cmux": 2, "agent_team": 1, "subagent": 4}
    elif parallel_percent < 50:
        return {"cmux": 4, "agent_team": 3, "subagent": 2}
    elif parallel_percent < 80:
        return {"cmux": 4, "agent_team": 4, "subagent": 1}
    else:  # 80%以上
        return {"cmux": 3, "agent_team": 5, "subagent": 1}
```

---

## ツール使用

- **Read**: プランドキュメント（PLAN_*.md）、ACTIVE_TASKS.md
- **Grep**: タスク抽出（プランに関連する TODO タスクのみ）
- **出力**: 推奨方法 + 理由 + コマンド例（テキストのみ、実行はしない）

---

## エラーハンドリング

| エラー | 対応 |
|--------|------|
| プランドキュメントが見つからない | ユーザーに「どのプランを評価しますか？」と確認 |
| タスクが0個 | 「実装タスクがありません。プランを確認してください」 |
| 判定が拮抗（スコア差3点以内） | 両方を提示してユーザーに選ばせる（パターン2） |
| 評価軸の情報が不足 | 以下のデフォルト値を使用:<br>- タスク数: プラン内の `##` 見出し数または ACTIVE_TASKS.md の TODO 数でカウント<br>- 依存関係: Day/Phase構成があれば「partial」、なければ「none」<br>- 期間: プラン内に記載がなければタスク数 × 0.5日で推定<br>- フォルダ跨ぎ: プラン内で言及されたフォルダ数（biz_*/hq_*）をカウント<br>- 並列化: Day/Phase が独立していれば並列化可能と判定 |
| プランドキュメントが複数ある | ユーザーに「どのプランを評価しますか？」と複数選択肢を提示 |
| 全スコアが同じ（3方法とも同点） | 「明確な推奨はありません。cmux（低リスク）を推奨」と回答 |
| ACTIVE_TASKS.md が読めない | プランドキュメントのみで評価（タスク数 = プラン内見出し数） |

---

## 制約事項

- **実行はしない**: このスキルは推奨を提示するのみ。実際の `cmux` コマンドや Agent Team 起動は行わない
- **プラン依存**: プランドキュメントが存在しない場合は判定不可
- **静的評価**: タスクの難易度や技術的複雑性は考慮しない（タスク数・依存関係のみ）

---

## 使用例

### ケース1: Meeting Agent Mock実装

**入力**: PLAN_MEETING_COPILOT_MOCK_IMPLEMENTATION.md + ACTIVE_TASKS.md（タスク#21-#28）

**出力**:
```
## 実行方法の推奨

推奨: **cmux** (スコア 20/25)

理由:
- タスク数8個 → cmux適正範囲
- 依存関係あり（Day 1→2→3） → cmux適切
- 期間5.5日 → セットアップコスト考慮でcmux優位
- 並列化50% → 1.5日節約可能

想定所要時間: 4営業日（逐次5.5日 → 並列4日）

実行コマンド例:
# Day 1（並列）
cmux data/meeting-agent-mock
cmux runbook/meeting-agent

# Day 2-5.5の詳細コマンドは「出力フォーマット > パターン1」を参照
（Day 2: agent実装+fixture注入、Day 3: UI実装、Day 4: デモシナリオ、Day 5-5.5: リハーサル+Playbook）
```

---

### ケース2: CRM全体実装（仮想）

**入力**: PLAN_CRM_FULL_IMPLEMENTATION.md + ACTIVE_TASKS.md（タスク#1-#70）

**出力**:
```
## 実行方法の推奨

推奨: **Agent Team** (スコア 23/25)

理由:
- タスク数70個 → Agent Team推奨範囲
- 依存関係20%のみ → 80%独立可能
- 期間15営業日 → セットアップコスト吸収可能
- フォルダ跨ぎ8個 → Agent Team適正

想定所要時間: 3-5営業日（セットアップ0.5日 + 並列実行）

実行方法:
/multi-agent CRM全体実装（タスク#1-#70）
```

---

### ケース3: 単一タスクの調査・実装（仮想）

**入力**: PLAN_SIMPLE_FEATURE.md + ACTIVE_TASKS.md（タスク#1-#2）

**出力**:
```
## 実行方法の推奨

推奨: **Subagent** (スコア 21/25)

理由:
- タスク数2個 → Subagent適正範囲
- 依存関係なし → 逐次実行で十分
- 期間1日以内 → セットアップコスト不要
- フォルダ跨ぎ1個 → シンプルな作業

想定所要時間: 1営業日（並列化のメリットなし）

実行コマンド例:
Task tool を使って一般目的エージェントを起動:

```bash
# タスク#1: リサーチタスク
Task(
  subagent_type="general-purpose",
  prompt="PLAN_SIMPLE_FEATURE.mdのタスク#1を実行: XXXの市場調査を実施し、レポートを作成"
)

# タスク#2: 実装タスク
Task(
  subagent_type="general-purpose",
  prompt="PLAN_SIMPLE_FEATURE.mdのタスク#2を実行: 調査結果に基づいてYYY機能を実装"
)
```

**補足**:
- 単一タスクの場合、cmuxやAgent Teamのセットアップコストが無駄
- Subagentは即座に起動でき、完了後に即座に結果を返す
- 依存関係がある場合は、タスク#1完了後にタスク#2を実行（逐次実行）
```

---

## メンテナンス

- **判定基準の更新**: MEMORY.md に記録された実績に基づいて、スコアリングロジックを調整
- **新方法の追加**: 将来的に新しい実行方法（例: Hybrid cmux + Agent Team）が登場した場合、評価軸を追加

---

## 関連ドキュメント

- [MEMORY.md](../../memory/MEMORY.md) — 過去の実行方法の実績・教訓
- [ACTIVE_TASKS.md](../../../${TASK_TRACKER}) — タスク管理
- [cmux ドキュメント](../using-git-worktrees/SKILL.md) — cmux の使い方
- [multi-agent skill](../multi-agent/SKILL.md) — Agent Team の使い方

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| 推奨のみ・実行しない | 権限分離原則 | 判定結果を提示するだけ。cmux/Agent Team/Subagentの実際の起動は行わない |
| 定量的5軸評価 | Decision Matrix手法 | 主観判断を排除。タスク数・依存関係・期間・フォルダ跨ぎ・並列化の5軸で機械的にスコアリング |
| 拮抗時は両方提示 | Human-in-the-Loop | スコア差3点以内なら推奨を1つに絞らず、両方の選択肢とトレードオフを提示 |

## Config

| カテゴリ | キー | デフォルト値 | 説明 |
|---------|------|------------|------|
| 評価 | max_score | 25点 | 5軸 x 5点満点 |
| 閾値 | clear_recommendation_gap | 4点 | この差以上で「明確な推奨」判定 |
| 閾値 | tie_threshold | 3点 | この差以内で「拮抗」判定（両方提示） |
| 推定 | default_days_per_task | 0.5日 | プランに期間記載がない場合の推定値 |
| パス | plan_pattern | `PLAN_*.md` | プランドキュメントの命名パターン |
| パス | tasks_file | `${TASK_TRACKER}` | タスク管理ファイル |

## セキュリティ

N/A -- 外部通信・認証なし。ローカルファイル（PLAN_*.md, ACTIVE_TASKS.md）の読み取りのみ。

## BLOCKINGゲート

| Step | 失敗条件 | 動作 |
|------|---------|------|
| Step 1 | プランドキュメントが見つからない | STOP + ownerに確認: 「どのプランを評価しますか？」 |
| Step 2 | ACTIVE_TASKS.md が読めない | 続行: プランドキュメントのみで評価（精度低下を報告） |
| Step 3 | タスクが0個 | STOP + 「実装タスクがありません。プランを確認してください」 |
| Step 4 | 全スコアが同点 | 続行: 「明確な推奨なし。低リスクのcmuxを推奨」と回答 |
| Step 5 | プランドキュメントが複数存在 | STOP + ownerに確認: 「どのプランを評価しますか？」と選択肢提示 |

## エスカレーション

| 状況 | 対応 |
|------|------|
| 評価軸の情報が不足（期間・並列化が不明） | ownerに確認: 「プランに期間の記載がありません。推定 {N}日で評価してよいですか？」 |
| 新しい実行方法の追加要望 | ownerに確認: 「新方法 '{名前}' の評価軸を定義してください」 |
| スコア差が1-2点で判断が難しい | ownerに確認: 「リスク許容度と開始スピードのどちらを優先しますか？」 |

## 合成可能性

| 連携スキル | 関係 | トリガー |
|-----------|------|---------|
| super-plan | 前工程 | super-plan完了後に本スキルで実行方法を判定 |
| writing-plans | 前工程 | プラン策定完了後に本スキルで実行方法を判定 |
| rapid-build | 後工程 | cmux/シングルエージェント推奨時に起動 |
| multi-agent | 後工程 | Agent Team推奨時にチーム構成・タスク分割を実行 |
| using-git-worktrees | 後工程 | cmux推奨時にgit worktree並列実行を開始 |
| dispatching-parallel-agents | 後工程 | Subagent推奨時に並列エージェント起動 |
| leak-learner | 学習 | owner指摘をlessons/に記録 |

## 汎用性

企業固有値はConfig表（plan_pattern, tasks_file）で外部化済み。スコアリングロジックは汎用的な5軸評価で企業依存なし。

## 他のスキルとの連携

| スキル | 関係 | 説明 |
|--------|------|------|
| super-plan | 前工程 | super-plan完了後に本スキルで実行方法を判定 |
| writing-plans | 前工程 | プラン策定完了後に本スキルで実行方法を判定 |
| rapid-build | 後工程 | cmux/シングルエージェント推奨時に起動 |
| multi-agent | 後工程 | Agent Team推奨時にチーム構成・タスク分割を実行 |
| using-git-worktrees | 後工程 | cmux推奨時にgit worktree並列実行を開始 |
| dispatching-parallel-agents | 後工程 | Subagent推奨時に並列エージェント起動 |
| leak-learner | 学習 | owner指摘をlessons/に記録 |

## やらないこと

- 実際のcmux/Agent Team/Subagentの起動（推奨提示のみ）
- タスクの技術的難易度の評価（静的な5軸のみ）
- プランドキュメントの作成・修正（読み取り専用）
- 過去の実行実績に基づく自動学習（手動でスコアリングロジックを調整）

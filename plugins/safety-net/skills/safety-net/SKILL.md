---
name: safety-net
description: "Zero Leak L3: owner提示前の独立検査官。確定的チェッカー（grep/parser）+ LLM別視点のハイブリッド。L2と構造的に異なる手段でチェックし、同じ盲点を持たない。"
---

# Safety Net — Zero Leak L3（独立検査官）

L2（quality-rules）で漏れた問題を、**構造的に独立した手段**で捕捉する最後の砦。

## 設計思想

```
L2（LLMベース）: AIが品質基準を参照して「正しく作る」
L3（ハイブリッド）: 確定的スクリプトが機械的に検査 + LLMは別視点で判断
  → L2が見落としたものをL3が見落とす確率が構造的に低い
```

**業界根拠**: Galileo Guardrails, TEKsystems 3-Layer Security, Anthropic Building Effective Agents

## トリガー

**自動（ownerに見せない）:**
- ownerに成果物を提示する直前
- `/ship` 実行時（BLOCKING前工程）
- rapid-build Step 4（品質ゲート）と統合

## 手順

### Step 1: 確定的チェッカー（スクリプト — LLMの盲点を補完）

LLMに依存しない機械的チェック:

#### 1a. ハードコード検出
```bash
# コード内のURL（config外）
Grep: "http[s]?://" --type py,ts,js → config/以外のヒットを列挙

# メールアドレス
Grep: "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" --type py,ts,js

# Slack channel/user ID
Grep: "C0[A-Z0-9]{8,}|U0[A-Z0-9]{8,}" --type py,ts,js
```

#### 1b. デザイン品質走査（HTML対象の場合）
```bash
# font-size < 18px の検出
Grep: "font-size:\s*[0-9]{1,2}px" → 18未満を列挙

# ブランドカラー外の色
Grep: "#[0-9a-fA-F]{3,8}" → BRAND_DESIGN_SYSTEM.mdのパレットと照合

# overflow:hidden + 固定幅（文字切れリスク）
Grep: "overflow:\s*hidden" → 同要素にwidth指定がないか確認
```

#### 1c. メタデータ確認
```bash
# 成果物の実在
test -f {成果物パス}

# ACTIVE_TASKS.md に記録済みか
Grep: "{成果物キーワード}" → ${TASK_TRACKER}

# 修正ファイル数 vs L1の影響ファイル数
# L1で列挙した全ファイルが確認済みか照合
```

#### 1d. リグレッション
```bash
# auto-test連携: テストがあれば実行
# auto-testスキルを呼び出し

# 過去インシデントパターン照合
# INCIDENT_REPORT_*.md から過去のバグパターンを抽出 → 今回の変更と照合
```

### Step 2: LLM判断（確定的チェッカーでカバーできない領域のみ）

- 「この修正で他の箇所に副作用がないか？」
- 「L1で列挙した全ファイルのうち未確認のものはないか？」
- super-planの done_criteria を全て満たしているか（設計書がある場合）

### Step 3: 結果判定 + 動作

| 検出結果 | 動作 |
|---|---|
| 問題0件 | 何も表示しない。完璧な成果物だけownerに提示 |
| 確定的違反（font-size, hardcode等） | **自動修正** → ownerに報告不要 |
| 設計判断が必要 | ownerにエスカレーション: 「この箇所はルール違反ですが、意図的ですか？」 |
| セキュリティ違反 | **BLOCKING**: ownerの明示承認なしに提示しない |

### Step 3b: インシデント自動記録（decision-capture連携）

問題を検出した場合、decision-captureルールに従い `INCIDENT_REPORT_*` を自動作成する:
- 確定的違反 → `${KNOWLEDGE_DIR}/INCIDENT_REPORT_{概要}_{YYYYMMDD}.md` に記録
- セキュリティ違反 → 同上 + ownerエスカレーション
- 自動修正した場合も記録（修正内容・原因・影響範囲を含める）

### Step 4: L5への送信

全ての検出パターンをL5 leak-learnerに送信:
- 確定的チェッカーで捕捉 → L2のルールが不十分だった証拠 → L5がL2強化を検討
- LLM判断で捕捉 → 新パターンとしてSkill lessonsに記録候補

問題検出時にcorrections.jsonlにエントリを追加（L5学習候補）:
```jsonl
{"skill": "safety-net", "pattern": "{検出パターン}", "correction": "{自動修正内容 or エスカレーション結果}", "layer": "L3", "source": "{対象ファイルパス}"}
```

## 連携

| スキル | 関係 |
|---|---|
| quality-rules (L2) | L2で漏れたものをL3が捕捉。構造的に独立 |
| leak-learner (L5) | L3の捕捉結果をL5に送信。L5がL2に昇格 |
| auto-test | L3がテスト実行を依頼 |
| security-audit | セキュリティ違反検出時に連携 |
| rapid-build | Step 4（品質ゲート）と統合 |
| ship | BLOCKING前工程として組み込み |
| leak-learner | L3の捕捉結果をL5に送信。owner指摘をlessons/に記録 |

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| 構造的独立 | Galileo Guardrails / 3-Layer Security | L2（LLMベース）とL3（確定的スクリプト+LLM別視点）は構造的に異なる手段。同じ盲点を持たない |
| 確定的チェッカー優先 | TEKsystems / Anthropic | LLM判断より先にgrep/parserで機械的に検査。LLMは補完 |
| 問題0件なら沈黙 | ユーザー体験最優先 | 完璧な成果物だけownerに提示。不要な報告でownerの時間を奪わない |

## Config

| カテゴリ | キー | デフォルト値 | 説明 |
|---------|------|------------|------|
| 検査 | min_font_size |  | font-size下限（デザイン品質） |
| 検査 | brand_palette_source |  | ブランドカラー照合先 |
| 検査 | hardcode_patterns | , ,  | ハードコード検出パターン |
| パス | tasks_file |  | メタデータ確認先 |

## セキュリティ

| 項目 | ルール |
|------|--------|
| セキュリティ違反 | BLOCKING: ownerの明示承認なしに成果物を提示しない |
| 自動修正の範囲 | 確定的違反（font-size, hardcode）のみ自動修正。設計判断が必要なものはエスカレーション |

## BLOCKINGゲート

| ステップ | 失敗条件 | 動作 |
|---------|---------|------|
| Step 1 | 確定的チェッカーでセキュリティ違反検出 | BLOCKING: owner承認まで成果物を提示しない |
| Step 2 | LLM判断で重大な副作用を検出 | STOP + ownerに確認: 「この修正に副作用の可能性があります」 |
| Step 3 | L1の影響ファイルリストと実際の修正ファイルが不一致 | 警告: 「未確認ファイルがあります: {リスト}」 |

## エスカレーション

| 状況 | 対応 |
|------|------|
| ルール違反だが意図的な設計判断の可能性 | ownerに確認: 「この箇所はルール違反ですが、意図的ですか？」 |
| 確定的チェッカーとLLM判断が矛盾 | ownerに確認: 「機械チェックはOKですがLLM判断で懸念があります」 |
| 修正ファイル数がL1影響範囲より多い（スコープ超過） | ownerに確認: 「影響範囲外のファイルも変更されています」 |

## 汎用性

検査パターン・font-size下限・ブランドパレット参照先はConfig表で外部化済み。チェッカーロジック自体は汎用的で企業依存なし。

## 他のスキルとの連携

| スキル | 関係 | 説明 |
|--------|------|------|
| quality-rules (L2) | L2で漏れたものをL3が捕捉。構造的に独立 | - |
| leak-learner (L5) | L3の捕捉結果をL5に送信。検出結果をL5学習候補として送信（corrections.jsonl経由） | - |
| auto-test | L3がテスト実行を依頼 | - |
| security-audit | セキュリティ違反検出時に連携 | - |
| rapid-build | Step 4（品質ゲート）と統合 | - |
| ship | BLOCKING前工程として組み込み | - |
| incident-triage-lite | 出力先 | 問題検出時にインシデントログを自動作成 |

## やらないこと

- ルールの作成・変更（L5 leak-learnerの領域）
- 実装そのもの（L2 quality-rulesの領域）
- owner報告のフォーマット（L4 reporting-protocolの領域）

---
name: quality-rules
description: "Zero Leak L2: 実装中の主防御。quality-rules.yamlを参照し、最初から正しく作る。Write/Edit時に自動適用。全スキル共通基盤。"
---

# Quality Rules — Zero Leak L2（主防御）

品質基準に従って**最初から正しく作る**。作ってから検査するのではなく、作る時点でルールを守る。

## 設計思想

```
❌ 旧: Write(間違ったコード) → 検査 → 修正 → ownerに提示
✅ 新: L2参照 → Write(最初から正しいコード) → ownerに提示
```

## トリガー

**自動（全スキルで常時適用）:**
- コード・HTML・ドキュメントをWrite/Editする時
- 全てのスキル実行中（presentation-architect, rapid-build, etc.）

**明示的トリガーはない。** L2はバックグラウンドで常に動作する「免疫システム」。

## 手順

### Step 1: ルール読み込み

タスク開始時に以下を読み込む（L1 risk-scanと連動）:

1. `.claude/quality-rules.yaml`（Global rules）
2. `skills/{current-skill}/lessons/*.md`（Skill lessons — あれば）
3. Session memory（同セッション内のowner指摘 — あれば）

### Step 2: 実装時の適用

Write/Edit実行時に、読み込んだルールを参照して実装する。

**コード系（Write/Edit対象が .py, .ts, .js 等の場合）:**

| ルールカテゴリ | チェック | 例 |
|---|---|---|
| hardcode | URL, email, Slack ID, magic numberがコード内にないか | `HC001-HC005` |
| schema | API型定義と一致するか、.env.exampleに追記したか | `SC001-SC002` |
| deploy | デプロイ後確認を計画に含めたか | `DP001-DP002` |
| regression | 修正パターンで全量Grepしたか | `RG001-RG002` |

**デザイン系（Write/Edit対象が .html, .css 等の場合）:**

| ルールカテゴリ | チェック | 例 |
|---|---|---|
| typography | font-size >= 18px, 見出し階層 | `TY001-TY002` |
| color | ブランドパレットのみ, 赤禁止, 3色以内 | `CL001-CL003` |
| layout | overflow/clip禁止, 参考資料の数値模倣 | `LY001-LY002` |
| content | 抽象的すぎない、具体例で伝える | `CT001` |

**プロセス系（全成果物共通）:**

| ルールカテゴリ | チェック | 例 |
|---|---|---|
| reporting | エラー報告テンプレート準拠 | `RP001-RP002` |
| meta | task tracker更新, SSOT即時反映 | `MT001-MT002` |

### Step 3: 違反時の動作

L2で違反に気づいた場合（自分が書こうとしたコードがルールに違反）:
- **書く前に修正する。** 違反コードをWrite してから直すのではなく、最初から正しく書く
- 修正不可能（設計判断が必要）→ ownerに確認してから実装

## quality-rules.yaml の場所

```
.claude/quality-rules.yaml（SSoT）
```

## ルールの追加・削除

- **追加**: L5 leak-learnerが自動追加。手動追加も可
- **削除**: ownerが「過剰」と判断 → 即削除
- **アーカイブ**: 6ヶ月違反ゼロ → quality-rules-archive.yamlに移動（週次メンテナンス）
- **上限**: 50件。超過時はアーカイブ

## 衝突解決ポリシー

```
Client lessons > Skill lessons > Global rules（具体的が優先）
例外: security / hardcode カテゴリは常にGlobal最優先（上書き不可）
```

## 連携

| スキル | 関係 |
|---|---|
| risk-scan (L1) | L1がルール読み込みを実行。L2はL1の結果を受けて適用 |
| review-guard (L3) | L2で漏れたものをL3が捕捉。L2の「セーフティネット」 |
| leak-learner (L5) | L5がowner指摘から新ルールを追加。L2を育てる |
| super-plan | 設計書のdone_criteria, 設定戦略がL2の適用範囲を事前に確定 |
| rapid-build | rapid-buildの実装中にL2が並走 |
| leak-learner | L5がowner指摘から新ルールを追加。L2を育てる。lessons/に記録 |

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| 最初から正しく作る | Shift-Left Quality | 作ってから検査するのではなく、作る時点でルールを守る |
| 免疫システム | Galileo Guardrails / Anthropic | 明示的トリガーなし。全スキルで常時バックグラウンド適用 |
| 具体的が優先 | 衝突解決ポリシー | Client lessons > Skill lessons > Global rules |

## Config

| カテゴリ | キー | デフォルト値 | 説明 |
|---------|------|------------|------|
| パス | rules_file |  | ルール定義のSSoT |
| パス | archive_file |  | アーカイブ先 |
| 閾値 | max_rules |  | ルール上限。超過時はアーカイブ |
| 閾値 | archive_threshold |  | 違反ゼロ期間でアーカイブ候補 |

## セキュリティ

| 項目 | ルール |
|------|--------|
| ルール変更 | L5 leak-learnerまたはownerのみがルールを追加・削除可能 |
| security/hardcodeカテゴリ | 常にGlobal最優先。Skill lessonsで上書き不可 |

## BLOCKINGゲート

| ステップ | 失敗条件 | 動作 |
|---------|---------|------|
| Step 1 | quality-rules.yamlが存在しない | 続行: Skill lessons + Session memoryのみで適用 |
| Step 1 | quality-rules.yamlのパースエラー | STOP + 「quality-rules.yamlの構文を確認してください」 |
| Step 2 | ルール違反を検出（設計判断が必要） | STOP + ownerに確認してから実装 |

## エスカレーション

| 状況 | 対応 |
|------|------|
| ルール違反だが設計上必要（intentional violation） | ownerに確認: 「ルール{ID}に違反しますが意図的ですか？」 |
| Global rulesが50件に到達 | ownerに確認: 「ルール上限に達しました。アーカイブ候補を提示します」 |
| Skill lessonsとGlobal rulesが矛盾 | ownerに確認: 「衝突解決ポリシーに従い Skill lessons を優先しますがよいですか？」 |

## 汎用性

ルールファイルパス・上限・アーカイブ閾値はConfig表で外部化済み。ルールカテゴリ（hardcode/schema/deploy等）は汎用的で企業依存なし。

## 他のスキルとの連携

| スキル | 関係 | 説明 |
|--------|------|------|
| risk-scan (L1) | L1がルール読み込みを実行。L2はL1の結果を受けて適用 | - |
| review-guard (L3) | L2で漏れたものをL3が捕捉。L2の「セーフティネット」 | - |
| leak-learner (L5) | L5がowner指摘から新ルールを追加。L2を育てる | - |
| super-plan | 設計書のdone_criteria, 設定戦略がL2の適用範囲を事前に確定 | - |
| rapid-build | rapid-buildの実装中にL2が並走 | - |
| leak-learner | L5がowner指摘から新ルールを追加。L2を育てる。lessons/に記録 | - |

## やらないこと

- ルールにない「好み」の判断（Skill lessonsの領域）
- テスト実行（L3 review-guardの領域）
- owner報告のフォーマット（L4 reporting-protocolの領域）

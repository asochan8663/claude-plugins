---
name: skill-upgrade
description: "既存スキルをsuper-plan品質基準にアップグレードする。8項目の品質チェック → 差分生成 → 書き換え → 検証を自動実行。"
triggers:
  - "skill upgrade"
  - "スキル改善"
  - "スキル品質"
  - "skill audit"
  - "スキル監査"
  - "品質向上"
---

# Skill Upgrade

既存のSKILL.mdを、super-plan / rapid-build / auto-browser水準（Tier S）にアップグレードする。

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| 設定の外部化 | 12-Factor App | ハードコード値をConfigセクションに移動 |
| Shift-Left Security | OWASP | セキュリティを後付けでなく設計段階で組み込む |
| BLOCKINGゲート | rapid-build | エラー時に黙って続行しない仕組みを追加 |
| 合成可能性 | Unix哲学 | 他スキルとの連携点を明示 |
| 冪等性 | — | 同じスキルに2回適用しても壊れない |

## 品質基準（8項目）

Tier Sスキルが共通して持つ要素。これをチェックリストとして使う。

### Q1: 設計原則テーブル
- **何**: なぜこの設計かの根拠を表形式で明示
- **基準**: `| 原則 | 出典 | 適用 |` 形式のテーブルが存在する
- **不合格パターン**: 設計判断の根拠がない。「なんとなくこう作った」状態

### Q2: Configセクション
- **何**: 環境依存の全値を1箇所に集約
- **基準**: パス、URL、APIキー名、閾値、タイムアウト値がConfigセクションにある。本文中にハードコードなし
- **不合格パターン**: `${KNOWLEDGE_DIR}/` のようなパスが本文中に散在
- **修正方法**: yaml形式のConfigセクションを作成し、本文は `{config_var}` で参照

### Q3: セキュリティセクション
- **何**: このスキルが扱う機密データの方針
- **基準**: 以下が明記されている:
  - APIキー/トークンの取得方法（環境変数 or Secret Manager）
  - ログに出してはいけないもの
  - PII（個人情報）の取り扱い
  - 外部URLへのアクセス方針
- **不合格パターン**: APIキーを使うのにセキュリティ方針がない
- **判定**: 外部API/PII/トークンを一切扱わないスキルは「該当なし」でOK（明示的にスキップを宣言）

### Q4: BLOCKINGゲート
- **何**: エラー時に黙って続行しない仕組み
- **基準**: 各ステップに「失敗時の動作」が明記。最低1つのBLOCKINGポイントがある
- **不合格パターン**: エラーを無視して次のステップに進む。または失敗パターンの記述自体がない
- **修正方法**: 各ステップに `失敗時: STOP → {対処}` を追加

### Q5: フェーズ分割
- **何**: 明確なステップの順序
- **基準**: Step 1, Step 2, ... または Phase 0, Phase 1, ... の形式でフェーズが分かれている
- **不合格パターン**: 箇条書きが並んでいるだけで実行順序が不明

### Q6: エスカレーション手順
- **何**: 自動化で完了できない場合の人間への引き渡し方法
- **基準**: 「自動化の限界 → ユーザーに何を伝えるか → ユーザーの操作後にどう再開するか」が明記
- **不合格パターン**: 失敗したら止まるだけ。ユーザーが何をすればいいかわからない

### Q7: 合成可能性（他スキルとの連携）
- **何**: このスキルの入力元と出力先
- **基準**: 「## 他のスキルとの連携」セクションがあり、関連スキルとの関係が明記
- **不合格パターン**: 孤立したスキル。他スキルとどう組み合わせるか不明
- **判定**: 本当に独立完結するスキルは「連携なし」でOK（明示的に宣言）

### Q8: やらないこと
- **何**: スコープ境界の明示
- **基準**: 「## このスキルがやらないこと」セクションで、隣接する責務との線引きが明確
- **不合格パターン**: スキルの責務が曖昧で、他スキルと重複する可能性がある

### Q9: 汎用性（Portability）
- **何**: 他社・他プロジェクトがこのスキルをそのまま使えるか
- **基準**: 以下の3条件を全て満たす:
  1. **フレームワーク/プロファイル分離**: スキルのロジック（汎用）と、会社固有の設定（プロファイル）が分離されている
  2. **ハードコード企業名ゼロ**: SKILL.md本文に「Agent Native」「irodas」等の企業名が直書きされていない（profilesから参照）
  3. **プロファイルテンプレート**: `profiles/_template.yaml` が存在し、他社が自社用プロファイルを作る手順が明記
- **不合格パターン**:
  - MVV、ブランドカラー、社名がSKILL.md本文に直書き
  - 「proposal-v3.html」のような固有ファイル名が本文に直書き
  - 他社が使うには「このスキルをフォークして書き換える」しかない状態
- **修正方法**:
  1. 企業固有の値を全て `profiles/{company}.yaml` に移動
  2. SKILL.md本文は `{profile.mvv.vision}` のようなプレースホルダーで参照
  3. `profiles/_template.yaml` を作成（他社がコピーして埋めるだけで使える）
- **プロファイルの標準構造**:
  ```yaml
  # profiles/{company}.yaml
  company:
    name: ""
    tagline: ""
  mvv:
    purpose: ""
    mission: ""
    vision: ""
    values: []
  brand:
    ssot_path: ""          # ブランドデザインシステムのパス
    primary_color: ""
    accent_color: ""
    art_style: ""          # 例: "岡本太郎/Gutai", "Bauhaus", "Swiss Design"
  templates:
    base_deck: ""          # ベーステンプレートのパス
  paths:
    ssot_mvv: ""
    ssot_brand: ""
    ssot_philosophy: ""
  ```
- **判定**: 完全に汎用化不可能なスキル（社内固有の業務手順等）は「社内専用」と明示してN/A

### Q10: Zero Leak L5 接続（Self-Improving via leak-learner）
- **何**: スキルがZero Leak学習ループ（leak-learner）と接続されており、owner指摘が自動で蓄積・昇格される仕組み
- **設計思想**: 各スキルは「メモ帳」を持つだけ。学習エンジンはleak-learner 1つに集約
- **基準**: 以下の3条件を全て満たす:
  1. **lessons/ディレクトリが存在**: `skills/{name}/lessons/` ディレクトリが存在する（空でもOK — leak-learnerが書き込む先）
  2. **連携テーブルにleak-learnerが含まれる**: SKILL.mdの「他のスキルとの連携」にleak-learner行がある
  3. **学習ロジックはスキル内に書かない**: 抽出・分類・昇格のロジックはleak-learner側の責務。各スキルには不要
- **不合格パターン**:
  - lessons/ディレクトリがない（学習の記録先がない）
  - 連携テーブルにleak-learnerがない（接続されていない）
  - スキル内に独自の学習ロジックを持っている（保守コスト増大・ドリフトの原因）
  - 教訓が汎用ルールにハードコードされている（quality-rules.yamlに昇格すべき）
- **修正方法**:
  1. `mkdir -p skills/{name}/lessons/` でディレクトリ作成
  2. 連携テーブルに `| leak-learner | 学習 | owner指摘をlessons/に記録。2+スキル共通パターンはGlobal rulesに昇格 |` を追加
  3. スキル内の独自学習ロジックがあれば削除（leak-learnerに委譲）
  4. SKILL.md内にハードコードされた教訓があれば、lessons/に移動 or quality-rules.yamlに昇格
- **4層記憶モデルとの対応**:
  ```
  Session memory（揮発性）  → leak-learnerが管理。各スキルは関与しない
  Skill lessons（永続）     → skills/{name}/lessons/*.md に記録。各スキル固有の教訓
  Client lessons（永続）    → skills/{name}/lessons/{client}.md に記録。クライアント固有
  Global rules（永続）      → .claude/quality-rules.yaml。2+スキル共通パターンが昇格
  ```
- **衝突解決ポリシー**: Client > Skill > Global（具体的が優先）。security/hardcodeは常にGlobal最優先

## ワークフロー

### Step 1: 対象スキルの選定

入力: スキルパス（1つ or 複数 or 「全部」）

```
単一: skill-upgrade .claude/skills/test-runner/
複数: skill-upgrade .claude/skills/test-runner/ .claude/skills/slack-brain/
全部: skill-upgrade --all
```

「全部」の場合:
1. `find .claude/skills/ ${PROJECT_HQ}/*/skills/ -name SKILL.md` で全スキルを列挙
2. Tier S（super-plan, rapid-build, auto-browser）は対象外（既に合格）
3. **このスキル自身（skill-upgrade）も対象外**

### Step 2: 品質監査（8項目チェック）

対象スキルのSKILL.mdを読み、Q1-Q8の各項目を判定:

```
結果フォーマット（スキルごと）:

{スキル名}:
  Q1 設計原則:     PASS / FAIL / N/A
  Q2 Config:       PASS / FAIL
  Q3 セキュリティ:  PASS / FAIL / N/A
  Q4 BLOCKING:     PASS / FAIL
  Q5 フェーズ分割:  PASS / FAIL
  Q6 エスカレーション: PASS / FAIL / N/A
  Q7 合成可能性:    PASS / FAIL / N/A
  Q8 やらないこと:  PASS / FAIL
  Q9 汎用性:       PASS / FAIL / N/A
  Q10 自己蓄積:     PASS / FAIL / N/A

  合格: X/10
  不合格項目: [Q2, Q3, ...]
```

### Step 3: 改善テンプレート生成

FAILした項目ごとに、追加すべきセクションのテンプレートを生成:

**Q1用テンプレート:**
```markdown
## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| {原則} | {出典} | {このスキルでの具体的適用} |
```

**Q2用テンプレート:**
```markdown
## Config

```yaml
# {カテゴリ}: {説明}
{変数名}: {デフォルト値}    # {何に使うか}
```
```

**Q3用テンプレート:**
```markdown
## セキュリティ

- **APIキー/トークン**: {取得方法 — 環境変数 $XXX / Secret Manager / 不要}
- **ログ出力禁止**: {出してはいけないもの}
- **PII取り扱い**: {個人情報の方針 — 扱わない / 匿名化 / 最小限保持}
- **外部アクセス**: {外部URL/APIへの方針 — ドメイン検証 / タイムアウト}
```

**Q4用テンプレート:**
```markdown
### 失敗時の動作

| ステップ | 失敗条件 | 動作 |
|---------|---------|------|
| Step X | {条件} | STOP → {対処方法} |
```

**Q6用テンプレート:**
```markdown
## エスカレーション

自動化で完了できない場合:
1. {ユーザーに何を伝えるか}
2. {ユーザーが何をすべきか}
3. {ユーザー操作後にどう再開するか}
```

**Q7用テンプレート:**
```markdown
## 他のスキルとの連携

| スキル | 関係 | 説明 |
|--------|------|------|
| {スキル名} | 入力元 / 出力先 / 並列 | {具体的な連携内容} |
```

**Q8用テンプレート:**
```markdown
## このスキルがやらないこと

- {隣接する責務} — それは {担当スキル} の仕事
```

**Q9用テンプレート:**
```markdown
## 汎用性（Portability）

このスキルは **フレームワーク（汎用）+ プロファイル（企業固有）** の構造。
他社は `profiles/_template.yaml` をコピーして自社用に埋めるだけで使える。

### プロファイル構造
```yaml
# profiles/{company}.yaml — 企業固有の設定
company:
  name: ""
  tagline: ""
mvv:
  purpose: ""
  mission: ""
  vision: ""
  values: []
brand:
  ssot_path: ""
  primary_color: ""
  accent_color: ""
  art_style: ""
paths:
  # 企業固有のファイルパス
```

### 他社の導入手順
1. `profiles/_template.yaml` をコピーして `profiles/{your-company}.yaml` を作成
2. 自社のMVV・ブランドカラー・ファイルパスを埋める
3. スキルを実行（プロファイルが自動で読み込まれる）
```

### Step 4: SKILL.md書き換え

各対象スキルについて:

1. 既存のSKILL.mdを完全に読む
2. **既存の構造・内容を尊重して保持**
3. FAILした項目のセクションを**追加**（既存セクションは削除しない）
4. ハードコード値をConfigセクションに移動
5. 書き換え後の行数が元の2倍を超えないこと（膨張防止）

**BLOCKINGルール:**
- 既存のトリガー、allowed-tools、descriptionは変更しない
- 元のスキルの動作ロジックを改変しない
- 追加するのは品質セクションのみ

### Step 5: 書き換え後検証（BLOCKING）

書き換え後のSKILL.mdに対して、Step 2を再実行:

```
全Q1-Q8がPASS or N/A → 合格
1つでもFAIL → Step 4に戻って修正
```

**検証をパスしないと完了報告しない。**

### Step 6: コミット

```bash
git add {変更ファイル}
git commit -m "upgrade({スキル名}): Q{不合格だった番号}を追加 — Tier S品質基準に準拠"
```

複数スキルを一括処理した場合:
```bash
git commit -m "upgrade: {N}スキルをTier S品質基準にアップグレード

対象: {スキル名リスト}
追加項目: Config外出し, セキュリティ, BLOCKINGゲート等"
```

## 一括適用モード（--all）

### 並列化戦略

```
27スキル - 3スキル(Tier S対象外) - 1スキル(自身) = 23スキル

分類:
  Group A: Configなし + セキュリティなし（最も改善量が多い）→ 先に処理
  Group B: Config有り + セキュリティなし（中程度の改善）
  Group C: Config有り + セキュリティ有り（軽微な改善）

実行:
  1. Group Aを並列エージェントで処理（parallel-tasksスキル利用）
  2. Group Bを並列処理
  3. Group Cを並列処理
  4. 全スキルの最終検証
```

### 一括適用後のサマリーレポート

```markdown
## Skill Upgrade サマリー

| スキル | Before | After | 追加項目 |
|--------|--------|-------|---------|
| {名前} | 3/8 | 8/8 | Q1,Q2,Q3,Q4,Q8 |

横断統計:
- Config外出し: XX% → 100%
- セキュリティ: XX% → 100%
- BLOCKINGゲート: XX% → 100%
```

## セキュリティ

- **このスキルが扱う機密データ**: なし（SKILL.mdの構造を変更するだけ）
- **リスク**: スキルの動作ロジックを意図せず改変する可能性
- **対策**: Step 4のBLOCKINGルール（既存ロジック保持の強制）

## 他のスキルとの連携

| スキル | 関係 | 説明 |
|--------|------|------|
| super-plan | 品質基準の源泉 | Q1-Q8の基準はsuper-plan/rapid-buildから抽出 |
| rapid-build | 品質基準の源泉 | QUALITY_GATE.mdのG1-G6と対応 |
| parallel-tasks | 一括適用時の並列実行 | --allモードで23スキルを並列処理 |
| security-scan | Q3の参照先 | セキュリティセクションの詳細基準を参照 |

## このスキルがやらないこと

- スキルの動作ロジックを改変すること（品質セクションの追加のみ）
- 新しいスキルを作ること（既存スキルの改善のみ）
- Tier Sスキルを再処理すること（既に合格）
- SKILL.md以外のファイルを変更すること（playbooks等は対象外）

## 汎用性（Portability）

このスキルは **汎用フレームワーク** — Q1-Q10品質基準自体がプロジェクト非依存。
他社は `.claude/skills/skill-upgrade/` をコピーするだけで使える。
企業固有の設定はなく、品質基準（Q1-Q10）は全スキルに共通適用される。

社内専用要素: `ACTIVE_TASKS.md` 更新手順（アダプタで上書き可能）。

## Zero Leak L5 接続

連携テーブルに leak-learner を追加:

| スキル | 関係 | 説明 |
|--------|------|------|
| leak-learner | 学習 | owner指摘をlessons/に記録。2+スキル共通パターンはGlobal rulesに昇格 |

> lessons/ ディレクトリが存在する。leak-learnerがowner指摘を自動蓄積する書き込み先。

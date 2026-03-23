---
name: plan-to-study
description: "PLAN_*.mdから技術判断を抽出し、STUDY_GUIDE + QUIZ + NOTEBOOKLMの3形式で教材を自動生成。CER精査 + Admiraltyスコア + 面白さ設計（Fireship/Brilliant/Critique型）を組み込み。"
triggers:
  - "勉強"
  - "study"
  - "教材"
  - "学習"
  - "plan-to-study"
  - "クイズ"
---

# Plan-to-Study スキル

PLAN_*.md（super-planの成果物）から技術判断を抽出し、3形式の教材を自動生成する。
「PLANを書いた = 学ぶ材料が揃った」を前提に、ownerの技術リテラシー向上を仕組みで支援する。

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| CER（主張-根拠-推論） | 科学教育国際標準 | 全技術判断をClaim-Evidence-Reasoningで構造化。根拠なき主張を教材に入れない |
| Admiralty Code | NATO情報信頼度評価 | ソース信頼度(A-E) x 情報確認度(1-6)で各根拠をスコアリング。C3以下は追加調査 |
| Fireship 100秒型 | Fireship (YouTube) | Hook(5秒) → 概念(20秒) → 実例(40秒) → まとめ(15秒) → 次の疑問(10秒) のペース配分 |
| Brilliant Try-Before-Read | Brilliant.org | 先にクイズで考えさせる → 間違えてから解説を読む。受動学習を排除 |
| ゼイガルニク効果 | 心理学（Zeigarnik, 1927） | 各セクション末に「未完了の問い」を残し、次セクションへの動機を生成 |

## トリガー

### 明示的
- 「勉強」「study」「教材」「学習」「plan-to-study」「クイズ」
- 「PLANから教材作って」「このPLANを勉強したい」

### 自動実行（super-plan Step 5で強制呼び出し）
- super-planのStep 4（エビデンスチェックリスト）通過後、Step 6（承認ゲート）の**前**に自動実行される
- ownerが技術判断を理解してから承認判断できるようにするための必須ステップ

> 背景: 2026-03-18 Meeting Agentバックエンド設計書でplan-to-studyの実行を怠り、ownerに指摘された。
> 根本原因: super-planのワークフローにplan-to-studyが組み込まれていなかった。
> 対策: super-plan Step 5として強制組み込み（提案ではなく必須）。

### 手動トリガー
- ownerが「これどういう意味？」「なんでこの技術？」と聞いた場合

## Config

```yaml
# ソースオブトゥルース
ssot_path: "${SECRETARY_DIR}/study/TECH_LITERACY_SSOT.md"

# 出力先（ディレクトリ名は英語ケバブケース）
output_dir: "${SECRETARY_DIR}/study/{plan-name}/"

# ファイル命名規則（BLOCKING — この形式以外で生成してはならない）
# 形式: {PREFIX}{番号}_{SCREAMING_SNAKE_NAME}_{種別}.md
# PREFIX: PLANのテーマから2-4文字の略称を決定（AG=Agent, DSE=Darwinian Skill Evolution等）
# 番号: 01から連番
# NAME: PLANタイトルをSCREAMING_SNAKE_CASEに変換
# 種別: STUDY_GUIDE / QUIZ / NOTEBOOKLM
#
# 例:
#   PLAN_MEETING_AGENT.md → AG01_REALTIME_MEETING_AI_STUDY_GUIDE.md
#   PLAN_DARWINIAN_SKILL_EVOLUTION.md → DSE01_DARWINIAN_SKILL_EVOLUTION_STUDY_GUIDE.md
#
# ❌ 禁止: STUDY_GUIDE.md, QUIZ.md, NOTEBOOKLM.md（プレフィックスなしは命名規則違反）

# テンプレート
template_paths:
  study_guide: ".claude/skills/plan-to-study/STUDY_GUIDE_TEMPLATE.md"
  quiz: ".claude/skills/plan-to-study/QUIZ_TEMPLATE.md"
  notebooklm: ".claude/skills/plan-to-study/NOTEBOOKLM_TEMPLATE.md"

# 精査チェックリスト
accuracy_checklist: ".claude/skills/plan-to-study/ACCURACY_CHECKLIST.md"

# CER / Admiralty 閾値
admiralty_threshold: "C3"        # C3以下は要追加調査（BLOCKINGで停止）
cer_required: true               # Evidence/Reasoningなしの主張を禁止

# ゲーミフィケーション
xp_rules:
  L1_product: 5                  # L1-Product判断 = 5pt
  L2_architecture: 10            # L2-Architecture判断 = 10pt
  L3_infrastructure: 30          # L3-Infrastructure判断 = 30pt
  L4_business_model: 30          # L4-BusinessModel判断 = 30pt
quiz_pass_threshold: 70          # クイズ合格ライン（%）

# セキュリティ
anonymize_client_info: true      # クライアント固有情報を一般化して教材に含める
output_scope: "${PROJECT_HQ}"       # 教材は本部内に保存（共有リポに入れない）
```

## ワークフロー

### Step 1: PLAN読み込み + 技術判断抽出

PLAN_*.md を完全に読み込み、全ての技術判断を抽出する。

**手順:**
1. 指定された PLAN_*.md を Read で全文読み込み
2. 設計書内の全ての技術判断・選択を洗い出す
   - 「Xを選んだ」「Xを使う」「Xの方がよい」「Xを採用」等のパターン
   - リサーチサマリー、アーキテクチャ、タスク分解から網羅的に抽出
3. 各判断に **4層ラベル** を付与:

| ラベル | レベル | 例 |
|--------|--------|-----|
| `#L1-Product` | プロダクト層 | UI選択、ライブラリ選定、API設計 |
| `#L2-Architecture` | アーキテクチャ層 | モジュール分割、データフロー、状態管理 |
| `#L3-Infrastructure` | インフラ層 | デプロイ、CI/CD、クラウド構成 |
| `#L4-BusinessModel` | ビジネスモデル層 | 課金モデル、SLA、ライセンス |

4. 各判断に **Why x 3** を構造化:

```
判断: Next.js App Routerを採用
#L2-Architecture
Why-技術: RSCによるサーバーサイドレンダリングでTTFB改善
Why-ビジネス: Vercel無料枠でMVPコスト0
Why-戦略: React人材プールが最大 → 採用容易
```

**出力:** 技術判断リスト（ラベル + Why x 3 付き）

**失敗時: STOP → PLANに技術判断が0件の場合、ユーザーに報告して終了**

### Step 2: CER精査（BLOCKING）

Step 1 で抽出した全技術判断について、正確性を精査する。
**このステップをパスしない限り、教材生成に進んではならない。**

**手順:**

#### 2a. CER構造化

各技術判断を CER フォーマットに変換:

```
Claim（主張）: Next.js App RouterはPages Routerより高速
Evidence（根拠）: Vercel公式ベンチマーク（2025-11）でTTFB 40%改善
Reasoning（推論）: RSCによりクライアントJSバンドルが減少 → 初期ロード高速化
```

#### 2b. Admiralty Code スコアリング

各 Evidence に対して Admiralty Code（ソース信頼度 x 情報確認度）を付与:

- ソース信頼度: A(完全信頼) - E(信頼不可)
- 情報確認度: 1(他ソースで確認済) - 6(真偽判定不可)
- 詳細な判定基準: `ACCURACY_CHECKLIST.md` を参照

#### 2c. 閾値チェック

- **A1 - B2**: 採用（そのまま教材に含める）
- **B3 - C3**: 注釈付きで採用（「ただし〜の条件下で」等の留保を付記）
- **C4 - E6**: STOP → 追加調査が必要

#### 2d. 自動バリデーション（C3以下の項目 + 全項目の抜き打ち）

以下の4項目を自動検証:

1. **URL実在確認**: WebFetchで引用URLにアクセス可能か
2. **バージョン最新性**: 言及しているバージョンが最新か（WebSearchで確認）
3. **断言検知**: 「絶対」「必ず」「唯一」等の過度な断言がないか
4. **公式ドキュメントクロスチェック**: 技術的主張が公式ドキュメントと矛盾しないか

#### 2e. 追加リサーチ（C3以下が存在する場合）

- C3以下の項目について WebSearch で追加調査
- 追加調査後に再スコアリング
- それでもC4以下が残る場合: **STOP → ユーザーに信頼度不足の項目を報告**

```
=== CER精査レポート ===
合格: 12/15 判断（A1-B2）
注釈付き: 2/15 判断（B3-C3）
要確認: 1/15 判断（C4） ← BLOCKING
  - 「XxxはYyyより3倍速い」→ ソース不明、公式に記載なし
  - 推奨アクション: ownerに確認 or 教材から除外
```

**失敗時: STOP → 信頼度不足の項目をユーザーに報告。教材生成に進まない**

### Step 3: STUDY_GUIDE 生成

CER精査を通過した技術判断のみを使い、学習ガイドを生成する。

**手順:**
1. `STUDY_GUIDE_TEMPLATE.md` を Read で読み込み
2. テンプレートに従い、各技術判断を Fireship 100秒構成で展開:

```
## [用語] （カタカナ読み）
> ひとことで言うと: [20文字以内の説明]

**Hook（なぜ気になる？）**: [5秒で興味を引く一文]
**概念（何それ？）**: [専門用語なしで説明]
**実例（具体的には？）**: [PLANでの使われ方を引用]
**まとめ（結局？）**: [1文で要約]
**次の疑問（気になりません？）**: [ゼイガルニク効果 → 次のセクションへの橋渡し]
```

3. **全技術用語にカタカナ読みを付記**: RSC（アールエスシー）、TTI（ティーティーアイ）等
4. **全用語に「ひとことで言うと」を付記**: 20文字以内の平易な一文
5. Admiraltyスコアが B3-C3 の項目には注釈（「ただし〜」）を明記

**出力:** `{output_dir}/{ID}_{NAME}_STUDY_GUIDE.md`

### Step 4: QUIZ 生成

Brilliant Try-Before-Read 型のクイズを生成する。

**手順:**
1. `QUIZ_TEMPLATE.md` を Read で読み込み
2. テンプレートに従い、各技術判断からクイズを生成:

```
### Q{N}. [シナリオ設定]
> あなたはクライアントとの打ち合わせ中です。クライアントが聞きました:
> 「なぜ{技術A}ではなく{技術B}を選んだのですか？」

選択肢:
A) [もっともらしいが間違い]
B) [正解]
C) [部分的に正しいが不十分]
D) [よくある誤解]

<details>
<summary>解説を見る（まず自分で考えてから！）</summary>

正解: B
**なぜBか**: [CERのReasoning部分を平易に説明]
**なぜAではないか**: [誤答の論理的問題点]
**覚え方**: [記憶に残るフレーズ or たとえ話]
**XP**: +{L1=5/L2=10/L3=30}pt
</details>
```

3. **シナリオ型**: クライアント説明、投資家質問、チームメンバーからの技術質問 等
4. **XPを各問題に割り当て**: L1=5pt, L2=10pt, L3=30pt, L4=30pt

**出力:** `{output_dir}/{ID}_{NAME}_QUIZ.md`

### Step 5: NOTEBOOKLM 生成

Google NotebookLM に貼り付けて音声ディスカッションを生成するための入力テキストを作成する。

**手順:**
1. `NOTEBOOKLM_TEMPLATE.md` を Read で読み込み
2. テンプレートに従い、Critique討論型のディスカッション台本を生成:

```
## テーマ: {PLANのタイトル}

### ホストA（推進派）
このPLANの設計判断を支持する立場。
「{技術X}を選んだのは正解だと思います。なぜなら...」

### ホストB（懐疑派）
この PLANの設計判断に建設的な疑問を投げる立場。
「でも{技術Y}という選択肢もありましたよね？なぜ除外したんですか？」

### 討論ポイント
1. {最もインパクトの大きい技術判断}
   - A: [推進の論拠 — CERのEvidence引用]
   - B: [懐疑の論拠 — 代替案のメリット]
   - 結論: [PLANでの最終判断とその理由]

2. {2番目に重要な判断}
   ...
```

3. **クライアント機密を含めない**: 企業名、金額、個人名は一般化
4. **専門用語には括弧で補足**: 「RSC（リアクト・サーバー・コンポーネント）」

**出力:** `{output_dir}/{ID}_{NAME}_NOTEBOOKLM.md`

### Step 6: SSOT更新 + コミット

教材生成の結果を SSOT に反映し、コミットする。

**手順:**
1. `TECH_LITERACY_SSOT.md` を Read で読み込み
2. 対応するセル（技術判断の分野）を「学習中」にマーク
3. ストリーク / XP を更新（クイズの合計XP + 日付）
4. 生成した全ファイルを git add + git commit

```
git add ${SECRETARY_DIR}/study/{plan-name}/
git add ${SECRETARY_DIR}/study/TECH_LITERACY_SSOT.md
git commit -m "study: {plan-name} 教材生成（STUDY_GUIDE + QUIZ + NOTEBOOKLM）"
```

**失敗時: コミットが失敗した場合、エラーをユーザーに報告**

## セキュリティ

- **クライアント機密情報**: 教材に含めない。企業名・金額・個人名は一般化する
  - 例: 「A社のCRM」→「人材紹介会社のCRM」
- **NotebookLM入力**: 外部サービスに送信されるため、固有情報を含めない
  - クライアント名、社内URL、APIキー等は除去
- **教材の保存先**: `${PROJECT_HQ}/` 内に保存。共有リポ（${SHARED_REPO}等）に入れない
- **APIキー/トークン**: 不要（ファイル生成のみ）
- **外部アクセス**: Step 2のCER精査でWebSearch / WebFetchを実行

## エスカレーション

自動化で完了できない場合:

1. **CER精査でC4以下が残った場合**: 信頼度不足の項目をリストアップしてユーザーに報告。「教材から除外」か「追加調査の情報提供」を選択してもらう
2. **PLANに技術判断が見つからない場合**: PLANの種類（戦略系/運用系等）を確認し、教材化の対象外か判断をユーザーに委ねる
3. **テンプレートが存在しない場合**: テンプレート作成をユーザーに提案し、デフォルト構造で仮生成するか確認
4. **SSOT構造とPLANの分野が一致しない場合**: SSOTに新カテゴリを追加してよいかユーザーに確認

## 他のスキルとの連携

| スキル | 関係 | 説明 |
|--------|------|------|
| super-plan | 入力元 | PLAN_*.md がこのスキルの入力。super-planが承認済みPLANを生成する |
| rapid-build | 出力先 | 教材で学習した技術知識が、rapid-build実行時のowner判断力を向上させる |
| skill-upgrade | 品質管理 | Q1-Q8基準でこのスキル自体の品質を検証。テンプレートの改善提案 |

## このスキルがやらないこと

- **コードを書くこと**: 教材はMarkdownドキュメントのみ。実行可能コードは生成しない
- **プランを作成すること**: それはsuper-planの仕事。このスキルは既存PLANを消費するだけ
- **LMSを構築すること**: 学習管理システムの開発は行わない。ファイルベースの教材生成のみ
- **既存教材を修正すること**: 既に生成済みの教材の編集は手動で行う。再生成は新規として扱う
- **エビデンスなしで教材を作ること**: CER精査（Step 2）をパスしない内容は教材に含めない

## 汎用性（Portability）

このスキルは **フレームワーク（汎用）+ Config（プロジェクト固有）** の構造。
他社は Config セクションの `ssot_path`, `output_dir`, `template_paths` を自社パスに変更するだけで使える。
CER精査・Admiralty Code・Fireship構成・Brilliant型クイズのフレームワークはプロジェクト非依存。

企業固有の値は全て Config セクションに集約済み。SKILL.md本文にパスのハードコードなし。

## Zero Leak L5 接続

連携テーブルに leak-learner を追加:

| スキル | 関係 | 説明 |
|--------|------|------|
| leak-learner | 学習 | owner指摘をlessons/に記録。2+スキル共通パターンはGlobal rulesに昇格 |

> lessons/ ディレクトリが存在する。leak-learnerがowner指摘を自動蓄積する書き込み先。

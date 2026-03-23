# Skill: presentation-architect

- **description**: 会社説明資料・提案書・ピッチデッキを一発で高品質に生成するオーケストレーター。ブランド未定義でもOK — プロファイルがなければ自動でブランド設計（MVV・カラー・フォント）から始める
- **type**: user
- **trigger**: 「資料作って」「スライド作って」「デッキ作って」「提案書作成」「会社説明資料」「ピッチ作って」「presentation」「deck」
- **does NOT trigger**: バグ修正、バックエンド実装、ドキュメント更新（スライド以外）

---

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| SSOT参照ファースト | 12-Factor App | 色・フォント・MVVは全てSSOTから読む。ハードコード禁止 |
| 再利用 > 再発明 | Unix哲学 | ベースデッキテンプレート（`{profile.templates.base_deck}`）をコピー。ゼロから書かない |
| 失敗パターン回避 | 過去セッション(152件) | 過去に指摘された50ルールを事前適用して手戻りゼロ |
| 図解メソッド分離 | svg-diagramスキル | テキスト図=HTML/CSS/SVG、雰囲気=AI画像。混ぜない |
| BLOCKINGゲート | rapid-build | Phase 0(SSOT読み込み)とPhase 4(品質チェック)を通過しないと進めない |

## Config

```yaml
# プロファイル（企業固有の設定は全てここから読む）
active_profile: "profiles/my-company.yaml"  # 変更するだけで他社に切替

# スキル内部パス（企業非依存）
lessons_universal: ".claude/skills/presentation-architect/SLIDE_DESIGN_LESSONS.md"  # 汎用ルール
# 企業固有lessons → profile.paths.lessons（profiles/lessons/{company}.md）
quality_gate: ".claude/skills/presentation-architect/QUALITY_GATE.md"

# 閾値（企業非依存のデザイン基準）
min_font_size: 18            # px。プレゼン最小フォント
min_heading_size: 32         # px。見出し最小
min_geo_text_line_height: 1.1  # gのディセンダー切れ防止
min_stat_number_size: 48     # px。データビジュアライゼーション風数字の最小
watermark_patch_size: 150    # px。AI画像ウォーターマーク除去範囲
canvas_crop_percent: 5       # %。キャンバス縁トリミング
```

> **企業固有の値（MVV、色、パス、アートスタイル等）は `profiles/{company}.yaml` に定義。**
> SKILL.md本文には企業名・ブランド色・MVVテキストをハードコードしない。

---

## 依存スキル

| Dependency | Role |
|------------|------|
| `frontend-slides` | HTML生成エンジン |
| `svg-diagram` | データ図解生成（テキスト/数字を含む図） |
| AI画像生成 | 背景/アート（プロンプトを出力、ownerが外部で生成） |

## SSOT References

| What | Source |
|------|--------|
| Brand Design System | `{profile.paths.ssot_brand}` |
| MVV | `{profile.paths.ssot_mvv}` |
| Product Philosophy | `{profile.paths.ssot_philosophy}` |
| Universal Lessons | Config `lessons_universal`（汎用ルール） |
| Company Lessons | `{profile.paths.lessons}`（企業固有 — 生の声） |
| Quality Gate | Config `quality_gate` |
| AI Image Prompts | `{profile.paths.ssot_image_prompts}` |
| Base template | `{profile.templates.base_deck}` |

> パスは全て `profiles/{company}.yaml` から読む。SKILL.md本文にハードコードしない。

---

## Workflow（8フェーズ）

### Phase -1: ストーリーライン確定（BLOCKING — 最初にやる）

**資料の構成を決める前に、「何をどの順で語るか」を確定する。省略禁止。**

#### Step -1a: 営業ストーリーの参照元を検索

1. **営業資料の原理原則**: `${SALES_BENCHMARK_DOC}` をRead
   - 14社のベンチマーク、構成パターン、メッセージング手法の型を把握
2. **営業鉄則ドキュメント**: `find ${PLANS_DIR} -name "PLAN_*SALES*" -o -name "PLAN_*DECK*" | sort` で検索・Read
3. **営業MTG議事録**: `find ${MEETINGS_DIR} -name "*sales*" -o -name "*strategy*" -o -name "*story*"` で検索・Read
4. 見つかったファイルを**全てRead**（最新版を`ls -lt`で判定）
5. 既存のストーリーライン（team member 9パート構成等）があればそれを採用
6. **確定版レジストリ**: `${ARTIFACT_REGISTRY}` で既存の確定版資料を確認

#### Step -1b: ownerに確認（BLOCKING）

ストーリーラインを要約してownerに提示:
```
このストーリーラインで資料を作ります:
1. [パート名] — [1行説明]
2. [パート名] — [1行説明]
...
OK？
```
**ownerの確認がなければPhase 0に進まない。**

#### Step -1c: 既存資料ベースの判定（BLOCKING）

1. 同じトピック/ターゲットの既存HTML資料を検索: `find ${TEMPLATES_DIR} ${TEMP_DIR} -name "*.html" | sort`
2. `ls -lt`で最新版を特定。複数候補があればownerに確認
3. 判定:

| 既存資料 | アクション |
|---------|---------|
| **ほぼ使える既存資料がある** | → コピー+Editで差し替え（Phase 0-3をスキップ、Phase 3.5へ直行） |
| **部分的に使える資料がある** | → その資料をベーステンプレートとしてPhase 0に設定 |
| **ゼロからの新規** | → 通常のPhase 0-5フロー |

**「ゼロから作り直す」は最終手段。既存資料を最大限活用する。**

> 背景: 2026-03-22。irodas-sales-deck-v2.htmlをコピーしてテキスト差し替えするだけで汎用資料が完成した。
> ゼロから作ったv2-v5は全て品質が低く、6回の手戻りが発生。
> 「既存の高品質資料をコピーして差し替え」 >>> 「ゼロから新規生成」

### Phase 0: プロファイル + SSOT読み込み（BLOCKING）

**1行もコードを書く前に、以下を全てReadする。省略禁止。**

#### Step 0-A: プロファイル存在チェック

1. Config `active_profile` で指定されたファイルを Read する
2. **ファイルが存在しない** or **`company.name` が空** → **Brand Bootstrap を自動起動**

```
プロファイルなし検知
  → Read "BRAND_BOOTSTRAP.md"
  → Brand Bootstrap Step 1-7 を実行（リファレンスリサーチ → MVV → カラー → フォント → profile.yaml 生成）
  → 生成された profile.yaml を active_profile に設定
  → Step 0-B に進む
```

**ユーザーは「資料作って」と言うだけ。** ブランドが未定義でもスキルが止まらない。必要な上流工程が自動で起動し、全てが揃った状態で資料生成に入る。

詳細: `BRAND_BOOTSTRAP.md`

#### Step 0-B: SSOT読み込み

1. Read `profiles/{company}.yaml`（企業プロファイル）
2. Read `{profile.paths.ssot_brand}`（ブランドデザインシステム — あれば）
3. Read `{profile.paths.ssot_mvv}`（確定済みMVV — あれば）
4. Read `{profile.paths.ssot_philosophy}`（プロダクト思想 — あれば）
5. Read `SLIDE_DESIGN_LESSONS.md`（汎用ルール）
5b. Read `{profile.paths.lessons}`（企業固有の教訓 — あれば。生の声形式）
6. Read `{profile.paths.ssot_image_prompts}`（AI画像生成プロンプト集 — あれば）
7. Read `{profile.templates.base_deck}`（ベーステンプレート — あれば）

**パスが空欄のSSOTはスキップ**（Brand Bootstrapで生成した直後はSSOTファイルがまだ存在しない。プロファイルの情報だけで資料生成を進める）

#### Step 0-B2: 型×色の確認

資料に図解（svg-diagram）が含まれる場合、型と色を確認する。

1. ownerが指定していれば従う（例: 「活版 × 地層で」）
2. 指定がなければデフォルト: 活版 × 墨
3. BRAND_DESIGN_SYSTEM.md §1.7 に定義された型×色の組み合わせを使用

```
型（Shape）: 活版 Kappan（密、角なし） / 凪 Nagi（余白、角丸）
色（Palette）: 墨 Sumi / 海 Umi / 地層 Chisou / 火花 Hibana
定義元: BRAND_DESIGN_SYSTEM.md §1.6-1.7 + svg-diagram SKILL.md Config
```

#### Step 0-C: リファレンス取得（プロファイルにconfirmedがない場合）

1. プロファイルの `design_reference.confirmed` を確認
2. **あり** → Step 0-Bの結果と統合してPhase 1へ
3. **なし** → auto_search を発動:
   a. WebSearchで「{業界} pitch deck structure」「{業界} 会社説明資料 構成」を検索
   b. 記事/ブログから構成パターン(スライド数、セクション順序)を学習
   c. SlideShare/Speaker Deck等のHTML版があればWebFetchで取得し構造分析
   d. PDFが必要な場合: ownerに「このPDFをダウンロードして_workspace/temp/に置いてください」とエスカレーション
   e. 分析結果を `profiles/references/{industry}_patterns.yaml` に保存
   f. **WebSearch失敗時**: `design_reference.fallback` ルールをそのまま適用
4. Phase 1へ

**presentation-architect経由の場合、frontend-slidesのPhase 2(Style Discovery)は自動スキップ。**
スタイルはプロファイルのHibana設定で固定される。

### Phase 1: 資料タイプ判定 + エンジン選択

ownerの指示からデッキタイプを分類し、適切な生成エンジンを選択する。

| 指示 | タイプ | エンジン | 理由 |
|------|--------|---------|------|
| 「会社説明資料」「About us」 | Company Deck | **frontend-slides** | 1回きり、凝ったデザイン |
| 「◯◯社向け提案書」「Proposal」 | Proposal | **frontend-slides** | ブランド全適用、品質重視 |
| 「ピッチ」「投資家向け」 | Pitch Deck | **frontend-slides** | 構成変更（Problem → Solution → Market → Team → Ask） |
| 「営業資料」「Sales deck」 | Sales Deck | **frontend-slides** | Proposal の簡略版 |
| 「同じテーマで量産」「テンプレで作って」 | Sales Batch | **slide-starter** | 共通engine/theme、デッキだけ追加 |

**エンジン選択ルール:**
- デフォルト = `frontend-slides`（1回きりの高品質HTML）
- 「量産」「テンプレ」「同じデザインで」= `slide-starter`（共通engine/theme + デッキ追加）
- どちらもこのスキル経由でのみ呼ばれる。直接トリガーしない

**Proposal の場合**: `${CLIENT_PROFILE}` も追加でReadする。

### Phase 2: 構成設計（4パート構造）

リファレンス企業の構成をベースに、4パート構成で設計する。

```
PART 1: About        -- 表紙, 目次, Founder, 会社情報
PART 2: Purpose & Values -- MVV, Valuesパレット, Value詳細
PART 3: Platform     -- 時代背景, 3レイヤー, Agent Fleet, 事業説明
PART 4: Use Cases    -- 業界特化（差し替え可能）
Ending: ロゴ + Vision + Contact
```

**Pitch Deckの場合**: Problem → Solution → Market → Team → Ask に再構成。

### Phase 3: スライド生成（ルール全適用）

#### frontend-slides呼び出し時の必須オーバーライド

presentation-architectからfrontend-slidesに委譲する際、以下を必ず適用:
1. Mode C (Existing Presentation Enhancement) として呼び出す — Phase 2(Style Discovery)をスキップ
2. CSS変数をプロファイルのプレゼン専用値でオーバーライド:
   - `--body-size: clamp(1rem, 1.5vw, 1.25rem)` (最小16px→18pxに引き上げ)
   - `--title-size: clamp(2rem, 6vw, 4.5rem)` (最小32px保証)
   - BRAND_DESIGN_SYSTEMの `--text-body: 14px` 等のUI用変数は使用禁止
3. プレゼン専用CSS変数名を使う: `--pres-body`, `--pres-heading`, `--pres-stat`

#### 色ルール

| ルール | 詳細 |
|--------|------|
| テキスト色 | 白 or 黒のみ。**赤テキスト禁止。** |
| 暗い背景 | 白テキスト |
| 明るい/暖色背景 | 黒テキスト |
| 赤の使用 | プロファイル指定のValue（`{profile.mvv.values}`）の背景色としてのみ |
| 図解画像 | プライマリカラー禁止。`{profile.brand.depth_color}` / `{profile.brand.accent_color}` / 黒 / 白のみ |

#### フォントルール

| ルール | 詳細 |
|--------|------|
| 最小フォントサイズ | 18px（プロダクトUI用サイズスケールを使わない） |
| 見出し | `clamp(32px, 4vw, 52px)` 以上 |
| geo-text の line-height | 最低 1.1（`g` のディセンダー切れ防止） |
| Sourceラベル | 14px のみ例外 |

#### 図解メソッド選択（v3最大の教訓）

| 用途 | メソッド | 理由 |
|------|----------|------|
| 背景、テクスチャ、アート | AI画像生成 | 抽象表現・ムード → AI得意 |
| データ、数字、比較 | `svg-diagram`（HTML/CSS/SVG） | テキスト精度・レイアウト精密 |
| フロー図、構造図（ラベル付き） | `svg-diagram`（HTML/CSS/SVG） | ラベル・矢印 → コードが正確 |
| ポートレート、アート | AI画像生成 | リアル/抽象 → AI得意 |

#### svg-diagram自動呼び出し（Phase 3内で実行）

テンプレート内の図解プレースホルダーを検出し、svg-diagramスキルで自動生成する。

**フロー:**
1. テンプレートHTML内の `<!-- DIAGRAM: {type} -->` コメントを検出
2. 各diagramタイプに応じてsvg-diagramスキルに委譲:
   - `three_layer`: 3レイヤー構造図（Creation / Management / Orchestration）
   - `agent_fleet`: Agent Fleet一覧（11体のアイコン+名前+説明）
   - `flow`: 業務フロー図（クライアント固有）
   - `comparison`: 比較表（Before/After、競合比較等）
   - `timeline`: タイムライン（導入スケジュール等）
   - `roi`: ROI図解（数字+矢印+結果）
3. svg-diagramが生成したSVG/HTMLをテンプレートのプレースホルダーに注入
4. 図解の色はプロファイルの `depth_color` / `accent_color` / 黒 / 白のみ（プライマリカラー禁止）

**プロファイルに図解仕様を定義:**
```yaml
# profiles/my-company.yaml の deck_types 内で指定
deck_types:
  company:
    diagrams:
      - slide: "p10"
        type: "three_layer"
        data_source: "profile.paths.ssot_philosophy"
      - slide: "p11"
        type: "agent_fleet"
        data_source: "MASTER_AGENT_REGISTRY.md"
  sales:
    diagrams:
      - slide: "p05"
        type: "flow"
        data_source: "client_profile.business_flow"
      - slide: "p07"
        type: "roi"
        data_source: "client_profile.roi_data"
```

**svg-diagramを呼ばない場合:** 図解が不要なスライド（テキストのみ、画像のみ）はスキップ

#### テキストルール

| ルール | 詳細 |
|--------|------|
| 文体 | 「宣言する、説明しない」（宣言型スタイル） |
| 会社説明資料にクライアントデータ | 禁止。会社デッキは会社情報のみ |
| 数量の表現 | 具体的な数字より抽象的表現を優先（プロファイルのMVVトーンに合わせる） |

#### レイアウトルール

| ルール | 詳細 |
|--------|------|
| セクション仕切り | マットホワイト。幾何学パターン禁止（ノイズになる） |
| 表紙と最終ページ | 同じ世界観で呼応（ブックエンド設計） |
| Valuesスライド | `{profile.mvv.values[].color_hex}` のカラーパレット |
| Valueカラーバー | アートワークとテキストをカラーバーで紐づけ |

#### 画像ルール

| ルール | 詳細 |
|--------|------|
| Valuesアートスタイル | プロファイル指定のアートスタイル（`{profile.brand.art_style}`） / 具体美術の抽象表現主義。フォトリアル禁止 |
| 図解画像の色 | 黒 / 金 / 藍 / 白のみ。赤禁止 |
| 背景画像 | プロファイル指定のアートスタイル（`{profile.brand.art_style}`）フリーカラー |
| Geminiウォーターマーク | 生成後に必ずPython/Pillowで除去（150pxパッチ） |
| キャンバス縁 | 5-10%クロップで隙間を除去 |

#### AI画像生成（2モード: API自動 or 手動）

スライド生成後、画像が必要な箇所を検出し、ユーザーに生成方法を確認する。

**Step 3-IMG-1: プロンプト自動生成**

1. テンプレート内の `<!-- IMAGE: {type} -->` を検出
2. プロファイルの `image_generation.templates` からタイプ別プロンプトを生成
3. `{profile.paths.ssot_image_prompts}` を読んで既存プロンプトの品質レベルに合わせる
4. プロンプト一覧を提示:

```
画像が必要なスライド:

| # | スライド | タイプ | プロンプト | サイズ |
|---|---------|--------|----------|--------|
| 1 | p01 表紙 | cover | 岡本太郎 concrete abstract... | 1920x1080 |
| 2 | p04 All In. | value_art | 岡本太郎... bold red... | 1080x1080 |
| ...
```

**Step 3-IMG-2: 生成方法をユーザーに確認**

```
画像の生成方法を選んでください:

A: API自動生成 — Gemini/DALL-E APIで全画像を一括生成・配置・後処理まで自動
B: 手動生成 — プロンプト一覧を提示。NotebookLM/Gemini等で手動生成→配置
C: 画像なしで進める — CSSグラデーション背景で代替。後で画像を差し替え可能
```

**モードA: API自動生成**
1. `scene-image-gen` スキルに委譲（Gemini Imagen 3 or DALL-E 3）
2. 生成 → ダウンロード → ウォーターマーク除去(150pxパッチ) → 縁クロップ(5%) → assets/に配置
3. HTMLの `<!-- IMAGE: -->` をimgタグに置換
4. プロンプト集(`ssot_image_prompts`)を更新 → commit

**モードB: 手動生成**
1. プロンプト一覧を提示（上記テーブル）
2. ユーザーが外部ツール(NotebookLM/Gemini)で生成 → ダウンロード
3. 「ダウンロードした」と言えば:
   - 画像を検出 → ウォーターマーク除去 → クロップ → assets/に移動
   - HTMLに配置 → プロンプト集を更新 → commit

**モードC: 画像なし**
1. `<!-- IMAGE: -->` をCSSグラデーション背景に置換:
   ```css
   background: radial-gradient(ellipse at 20% 80%, rgba(var(--primary-rgb), 0.1), transparent 50%),
               linear-gradient(180deg, var(--surface-dark), #111);
   ```
2. 後から画像を追加する場合は「画像追加して」で再実行可能

**プロンプト品質ルール（全モード共通）:**
- 必ず含める: 構図、パレット、ムード、アスペクト比、禁止事項
- 具体名詞を削除しても意味が通じること（W6ルール）
- `ssot_image_prompts` の既存品質以上
- プロンプト集を読まずに即興で作らない

### Phase 3.5: 修正モード判定（既存HTMLがある場合）

**BLOCKING: 同じファイルパスに既存HTMLがある場合、Phase 3の代わりにこのフェーズを実行する。**

```
既存HTMLファイルが存在するか？
  ├── YES → 修正モード（Edit-only）
  │   1. スナップショット: cp {file} {file%.*}-v{N}-snapshot.html
  │   2. 既存HTMLをRead（全体を把握）
  │   3. 変更箇所のみEditで差分修正
  │   4. WriteでHTMLを全体再生成は【絶対禁止】
  │   5. サブエージェントへの委譲は【禁止】（親セッションで直接Editする）
  │
  └── NO → 新規生成モード（Phase 3のフロー）
      1. frontend-slidesに委譲してHTML生成
      2. 通常の新規生成フロー
```

**なぜ修正モードが必要か:**
CSSのデザイン品質（spacing, shadow, border-radius, opacity等の微妙なバランス）は
テキスト指示では再現できない。全体再生成するたびに品質が劣化する。
差分Editなら既存の品質をそのまま保持できる。

> 背景: 2026-03-22。owner承認レベルのv2 HTMLを3回全体再生成し品質が回復不能に劣化。
> 詳細: `# (internal incident reference removed)`

### Phase 4: 品質ゲート（BLOCKING）

`QUALITY_GATE.md` の全チェック項目を実行する。
**1つでもFAILなら修正してからPhase 5に進む。**

### Phase 5: owner確認 → 永続保存 → 確定版レジストリ更新 → push

1. Chromeで開いてowner確認
2. **owner承認後、確定版として永続保存:**
   - **共有リポ（外部向け資料）:** `docs/business/{name}-vN.html` にworktree内で保存
     - 旧バージョンは `docs/business/archive/` に移動（同一PR内で）
     - PRに最新版 + archive移動の両方を含める
   - **個人リポ（内部向け資料）:** `${PRESENTATIONS_DIR}/{name}/vN.html` にコピー
   - **クライアント提案書:** `${CLIENT_PROPOSALS_DIR}/vN.html` にコピー（共有リポにpushしない）
   - バージョン番号は既存ファイルの最大値+1
   - `git add + commit` で永続化
3. **REGISTRY.yaml を自動更新 + 正本マーク:**
   ```bash
   # REGISTRY.yaml を再生成（全成果物スキャン）
   python3 tools/artifact_manager.py registry
   # owner承認後に正本マーク（approved symlink設定）
   python3 tools/artifact_manager.py approve {path}/vN.html
   ```
   - approved symlink = 正本。「資料出して」でこの版だけが返る
   - approveはowner承認後のみ実行（AIが勝手にapproveしない）
4. **owner承認後のみpush**（pushはownerがOKと言ってから）
5. 1 worktree = 1 task = 1 PR
6. 論理的な変更単位ごとに即commit

#### 成果物レジストリ（REGISTRY.yaml）

全ての成果物を一元管理するSSoT。**次回の資料作成時、Phase -1cで最初にここを参照する。**

```bash
# 全成果物の一覧（approved版にマーク付き）
python3 tools/artifact_manager.py list

# REGISTRY.yaml を再生成
python3 tools/artifact_manager.py registry

# 共有リポに昇格（approved版のみ）
# artifact promote (configure per project)
```

**ルール:**
- 「approved」ステータスの資料 = 正本。Phase -1cでベースとして自動選択される
- `_workspace/temp/`にある資料はowner確認後に永続保存先にコピー → approve
- **旧 ARTIFACT_REGISTRY.md は REGISTRY.yaml に移行済み**

> 背景: 2026-03-22。irodas-proposal-slides.html（初版）を参照してしまい6回手戻り。
> REGISTRY.yamlがあれば approved_version で正本が一目でわかる。

#### クライアントデータ保護（提案書/営業資料の場合）

提案書・営業資料(proposal/sales deck)はクライアント固有データを含むため:
1. **配置先**: `${CLIENT_PROPOSALS_DIR}/` (ai-company本体)。共有リポにpushしない
2. **品質ゲート**: `check_presentation_quality.py --check-client-data` でCLIENT_PROFILE.mdの値が含まれるHTMLを検出
3. **禁止**: 提案書HTMLを共有リポ(${SHARED_REPO})にcommit/pushすること

---

## Anti-Pattern Registry

v3セッションで蓄積された27の意思決定 + 30の修正パターン。
詳細は `SLIDE_DESIGN_LESSONS.md` に記載。以下はカテゴリ別サマリ。

### 色アンチパターン
- **C1**: 赤テキスト → 赤は背景/アクセントのみ
- **C2**: 図解に赤 → 黒 / 金 / 藍 / 白のみ
- **C3**: 色使いすぎ → モノクロベース + 1アクセント
- **C4**: Valuesアートがフォトリアル → プロファイル指定のアートスタイル（`{profile.brand.art_style}`）抽象表現主義

### フォントアンチパターン
- **F1**: 18px未満テキスト → プレゼン最小は18px
- **F2**: line-height 0.95以下 → 最低1.1
- **F3**: プロダクトUIサイズスケール流用 → プレゼン専用スケール

### 図解アンチパターン
- **D1**: AI生成のテキスト入り図 → テキスト図はHTML/CSS/SVG
- **D2**: 生のSVGフローチャート → 「情報を描く」ではなく「情報をデザインする」
- **D3**: データスライドの数字が小さい → 48px以上
- **D4**: 図とテキストの色が不一致 → カラーバーで紐づけ

### テキストアンチパターン
- **T1**: 説明的コピー → 宣言的ステートメント
- **T2**: 会社デッキにクライアントデータ → 会社デッキは会社情報のみ
- **T3**: 固有の数字表現 → プロファイルのMVVに合わせた表現
- **T4**: 会社デッキに「2週間無料」CTA → 「Let's talk」またはロゴ + Vision + Contact

### レイアウトアンチパターン
- **L1**: 幾何学パターンの仕切り → マットホワイト仕切り
- **L2**: 表紙/エンディングのビジュアル不一致 → 同じ世界観のブックエンド
- **L3**: テキストがスライド高さを超える → コンテンツを収める

### 画像アンチパターン
- **I1**: Geminiウォーターマーク残存 → Pillow 150pxパッチ除去
- **I2**: キャンバス縁の隙間 → 5-10%クロップ
- **I3**: ポートレート表紙の隙間 → background-size:coverの隙間をクロップ
- **I4**: プロファイル指定のアートスタイル不足 → `{profile.brand.art_style}` のキーワードをプロンプトに追加

### ワークフローアンチパターン
- **W1**: owner承認前にpush → 承認後のみpush
- **W2**: 1 worktreeに複数タスク → 1 worktree = 1 task = 1 PR
- **W3**: コミット遅延 → 即commit。auto-commitを信頼しない
- **W4**: 受領画像を未処理 → 分析 → マッピング → 移動 → ウォーターマーク除去 → クロップ → 配置 → commit
- **W5**: SSOT更新後に伝播忘れ → SSOT変更時は全参照先を更新

---

## セキュリティ

- **APIキー/トークン**: 不要（HTML生成のみ。AI画像生成はownerが外部ツールで実行）
- **ログ出力禁止**: クライアント固有データ（社名・KPI・価格）をスライドにハードコードしない
- **PII取り扱い**: 扱わない。提案書テンプレートにはプレースホルダーのみ
- **外部アクセス**: なし（全てローカルファイル操作）

---

## エスカレーション

自動化で完了できない場合:

| 状況 | ユーザーに伝えること | ユーザーの操作 | 再開方法 |
|------|-------------------|-------------|---------|
| SSOTファイルが見つからない | 「{ファイル名}が見つかりません。パスを教えてください」 | 正しいパスを指示 | Phase 0を再実行 |
| AI画像が必要 | 「以下のプロンプトで画像を生成してください」+ プロンプト一覧 | NotebookLM/Geminiで生成しダウンロード | 「ダウンロードした」と言えば画像処理フロー開始 |
| 品質ゲートFAIL | 「Q{番号}がFAILです。{具体的な問題}」 | 修正方針を指示 or 「直して」 | 該当箇所を修正しPhase 4を再実行 |
| 資料タイプが曖昧 | 「会社説明資料と提案書のどちらですか？」 | タイプを指示 | Phase 1から再実行 |
| MVVが変更されている | 「MVVが前回と異なります。最新版で進めますか？」 | 確認 | 最新MVVでPhase 2から再実行 |

---

## 他のスキルとの連携

| スキル | 関係 | 説明 |
|--------|------|------|
| `frontend-slides` | 出力先 | HTML生成エンジン（1回きりの凝ったプレゼン）。Phase 3で委譲。**直接トリガーしない** |
| `slide-starter` | 出力先 | テンプレート量産エンジン（同じデザインで複数デッキ）。Phase 3で委譲。${SHARED_REPO}内。**直接トリガーしない** |
| `svg-diagram` | 並列 | データ図解（テキスト/数字含む）の生成を委譲 |
| `BRAND_BOOTSTRAP.md` | 内蔵 | プロファイル不在時にPhase 0から自動起動。MVV・カラー・フォント・リファレンスリサーチを実行しprofile.yaml生成 |
| `uiux-design-workflow` | 入力元（任意） | 既にブランドアイデンティティがある場合はそこから読み込む。なければBRAND_BOOTSTRAPが代替 |
| `super-plan` | 入力元 | 資料の構成が複雑な場合、先にsuper-planで設計書を作る |
| `rapid-build` | 並列 | 大量スライド生成時にmulti-agentで並列実行 |

---

## このスキルがやらないこと

- **PPT/Keynoteへの変換** — それは `frontend-slides` のPhase 4
- **AI画像の生成自体** — プロンプトは出すが、生成はownerが外部ツールで実行
- **既存ブランドデザインシステムの定義変更** — Hibana（BRAND_DESIGN_SYSTEM.md）の責務。ただし未定義の場合はBRAND_BOOTSTRAPで新規策定する
- **既存MVV・プロダクト思想の変更** — PLAN_MVV_DESIGN.md / PRODUCT_PHILOSOPHY.md の責務。ただし未定義の場合はBRAND_BOOTSTRAPで新規策定する
- **クライアント固有の提案内容の判断** — ユーザーがスコープを指示する

---

## 汎用性（Portability）

このスキルは **フレームワーク（汎用）+ プロファイル（企業固有）** の構造。
他社は `profiles/_template.yaml` をコピーして自社用に埋めるだけで使える。

### ファイル構成

```
presentation-architect/
├── SKILL.md                  ← フレームワーク（企業名ゼロ）
├── BRAND_BOOTSTRAP.md        ← 自動ブランド設計（プロファイル不在時に起動）
├── SLIDE_DESIGN_LESSONS.md   ← デザイン教訓（汎用）
├── QUALITY_GATE.md           ← 品質基準（汎用）
└── profiles/
    ├── _fallback.yaml        ← 業界非依存フォールバック（プロファイルに値がない場合の安全網）
    ├── _template.yaml        ← 手動設定用テンプレート（Brand Bootstrapがあれば不要）
    └── my-company.yaml     ← Agent Native用プロファイル
```

### 他社の導入手順（自動パイプライン）

1. Claude Code をインストール
2. presentation-architect スキルをコピー
3. 「資料作って」と言う
4. → **プロファイル不在を自動検知** → Brand Bootstrap が起動
5. → 3問のヒアリング → リファレンス企業リサーチ → MVV策定 → カラーパレット → フォント → アートスタイル
6. → `profiles/{company}.yaml` を自動生成
7. → 資料生成に自動移行

**手動でプロファイルを埋める必要はない。** 「資料作って」の一言で、ブランド設計から資料完成まで一気通貫。

### プロファイルで外部化されている企業固有の値

| 値 | プロファイルのキー |
|----|-----------------|
| 会社名 | `company.name` |
| MVV全文 | `mvv.*` |
| ブランドカラー | `brand.primary_color`, `brand.accent_color` 等 |
| アートスタイル | `brand.art_style` |
| フォント | `brand.font_*` |
| SSOTファイルパス | `paths.*` |
| ベーステンプレート | `templates.base_deck` |
| Values色マッピング | `mvv.values[].color_token` |

---

## Zero Leak L5 接続

連携テーブルに leak-learner を追加:

| スキル | 関係 | 説明 |
|--------|------|------|
| leak-learner | 学習 | owner指摘をlessons/に記録。2+スキル共通パターンはGlobal rulesに昇格 |
| pre-check | 前処理 | 資料作成開始前に過去インシデント・品質基準を確認 |

> lessons/ ディレクトリが存在する。leak-learnerがowner指摘を自動蓄積する書き込み先。

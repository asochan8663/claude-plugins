# SVG Diagram Generator

フロー図・アーキテクチャ図をSVG/HTMLで一発生成するスキル。
「図解して」「フロー図作って」「アーキテクチャ図」等で起動。

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| テキスト図解ファースト | 2026-03-19 HERP教訓 | コードの前にASCII図で構造合意。紙の上の判断は安い |
| Uzabase方式デフォルト | 2026-03-21 Darwinian図 | モノクロ+赤1色。Space Grotesk+IBM Plex Mono。数字は巨大 |
| Playwright即検証 + Chrome表示 | Shift-Left Testing | Write後に必ずスクショ + `open -a Chrome` をセットで実行 |
| 固定座標 + 段間揃え | SVG座標管理 | 複数段のボックスはx座標とwidthを完全一致させる |
| 1パターン集中 | YAGNI | パターン増やさない。ownerが選んだ1つを磨く |
| 既存ファイル保全 | 2026-03-21 教訓 | 修正はv2, v3と別ファイル。上書きしない |

## トリガー

- 「図解して」「フロー図」「アーキテクチャ図」「ダイアグラム」
- 「〇〇の構成図」「〇〇の全体像」
- 「uzabase方式で」→ svg-diagram スキルの原則も併用
- 「clean-blue方式で」「Goodpatch方式で」→ clean-blueプリセット使用
- フロー図の修正・微調整リクエスト

## ワークフロー（5ステップ、省略禁止）

### Step 1: テキスト図解で構造合意（BLOCKING）

コードを1行も書く前に、ASCIIアートで図の構造をownerに提示する。

```
例:
  Phase 1: トップCA → Skill化 → 全員に展開
                                  ↓ ワークしたら
  Phase 2: 100人 → KPI → TOP Skill → 全員配布 (loop)
                                           → Rule化（卒業）
```

**確認事項:**
- ノードの数と名称
- ノード間の接続関係（矢印の方向）
- **対等性**: 一方通行に見えるが実は双方向ではないか？
- **多段構造**: 複数フェーズがあるか？ → 段間のボックス位置を揃える前提で設計
- レイアウト方向（縦/横/三角形/中央）
- 破線・ループの有無と接続先
- **用途**: スクリーン表示？写真で渡す？→ viewBoxサイズに影響

**owner承認を得てからStep 2に進む。承認なしでコードを書かない。**

### Step 2: viewBox + スタイル確定（BLOCKING）

SVGを1行も書く前に確定する:

```
1. 用途を確認:
   - スクリーン表示 → 1200x675
   - 写真で渡す（1ページ）→ 1200x900
   - 2ページ構成 → 1200x900 × 2枚

2. セクション数 × 必要高さを概算:
   - タイトル: 80px
   - 情報セクション（ボックス+矢印）: 各 200-320px
   - フッター（気づき等）: 120-160px
   - 合計がviewBoxに収まるか確認

3. スタイル選択:
   - 「uzabase方式」→ モノクロ+赤1色。Uzabase原則厳守
   - 「clean-blue方式」→ 青1色支配。Goodpatch原則。角丸・余白重視
   - デフォルト → モノクロ基調。色は赤1色まで
```

### Step 2.5: デザインパターン選択

メッセージに最適なパターンを選ぶ。**フローチャートをそのままSVGにしない。**

| パターン | 用途 | 構造 |
|---------|------|------|
| **Chain + Giant Numbers** | ボトルネック可視化、KPI | ノード鎖 + 巨大数字3つ |
| **Layer** | アーキテクチャ、Before/After | 2層の面（黒+灰） |
| **Giant Typography** | ヒーロー指標 | 数字が面積50%以上 |
| **Contrast Table** | 比較、2者対比 | 2列グリッド（灰bg/白bg） |
| **Multi-Section** | 複合図解（問題→解決→詳細） | 行ごとにセクション分割 |
| **3-Column Point Card** | ステップ・ポイント | 01,02,03の番号カード横並び |
| **Numbered Vertical List** | 課題列挙・要件 | 縦に番号付きカード |
| **Table (Blue Header)** | 評価マトリクス | 青帯ヘッダー+破線行区切り |
| **Process + Timeline** | フロー+所要時間 | 4カラムカード+時計+時間 |
| **Merit Grid 2x2** | メリット表示 | 青背景カード4枚の2x2 |
| **Callout Box** | 重要メッセージ強調 | 黄色角丸枠（枠線のみ） |
| **Category Tags** | カテゴリ分類 | 3色小タグ（青/赤/緑） |
| **Circular Flow** | 循環プロセス | 8ステップU字型ループ |
| **Step Staircase** | フェーズ進行 | 右上がり階段 |
| **Venn Diagram** | 概念の重なり | 3円重なり |
| **Persona Card** | ユーザー像 | 4象限(行動/思考/欲求/課題) |
| **Before/After Compare** | 比較 | 上下2段 |
| **4-Column Card + Tags** | 課題4つ | カード+ハッシュタグ帯 |

```
Pattern: Chain + Giant Numbers
  [chain]  node--line--NODE--line--node
  [arc]    ---------- 6h ----------
  [stats]  | 30    |  500+  |  94% |

Pattern: Layer
  [top]  ████ New Layer (black) ████
     ↕ Integration
  [bot]  ░░░░ Existing (gray) ░░░░

Pattern: Giant Typography
  87%        <- hero (面積50%以上)
  caption
  ─────────
  42%   |   3.2x

Pattern: Contrast Table
  | Old (gray bg)  | New (white bg)   |
  |----------------|------------------|
  | 1人が作って配る  | 100人で競争      |

Pattern: Multi-Section（Darwinian図で使用）
  ROW 1: PROBLEM | SOLUTION | EXISTING
  ROW 2: 黒背景（Phase 1 → Phase 2 + ループ + 卒業）
  ROW 3: NOW + PRODUCT
  ROW 4: INSIGHTS（2x2グリッド）

Pattern: 3-Column Point Card
  [ 01 タイトル ]  [ 02 タイトル ]  [ 03 タイトル ]
  [  説明テキスト ]  [  説明テキスト ]  [  説明テキスト ]

Pattern: Numbered Vertical List
  [ 1. カード（課題/要件） ]
  [ 2. カード（課題/要件） ]
  [ 3. カード（課題/要件） ]

Pattern: Table (Blue Header)
  |████ 視点 ████|████ 内容 ████|████ 重要度 ████|
  |--- 破線 ------|--- 破線 ------|--- 破線 -------|
  | 行1          | 内容         | 高             |

Pattern: Merit Grid 2x2
  [ 青カード1 ] [ 青カード2 ]
  [ 青カード3 ] [ 青カード4 ]

Pattern: Circular Flow
  → Step1 → Step2 → Step3 → Step4 ↓
  ← Step8 ← Step7 ← Step6 ← Step5 ↑
```

### Step 3: SVG実装（ルール厳守）

#### リファレンスパターン参照（推奨）

SVGを書く前に `lessons/REFERENCE_PATTERNS.md` をReadする。
過去の成功作品（lessons/references/*.html）から、近いパターンの具体的な座標・サイズ・間隔を参考にする。
テンプレートではなくリファレンス。内容に合わせてカスタマイズして使う。

成功した図はリファレンスに追加する（自己成長ループ）。

#### Uzabase方式カラールール（最重要）

```
使っていい色:
  #1A1816  — テキスト、ボックス背景（黒）、主線
  #F5F2ED  — 全体背景（温白）、黒背景上のボックス背景
  #D94032  — アクセント（赤）。強調・重要箇所のみ
  #6B6762  — サブテキスト（グレー）
  #D4D0CA  — 枠線・区切り線
  #fff     — ボックス背景（白セクション）

使ってはいけない色:
  ❌ #999, #bbb, #ccc — テキストに使うな（薄すぎて読めない）
  ❌ 青 #4a7aff — Uzabase方式では使わない
  ❌ 緑 #22a663 — 同上
  ❌ 金 #C9943E — 赤以外のアクセント禁止
  ❌ 3色以上のアクセント — モノクロ+赤1色が鉄則
```

> 背景: Skills Distribution Model図で青・赤・緑・金の4色を使い「色使いすぎ」と指摘。
> Darwinian Skill Evolution図でモノクロ+赤1色に統一し「めちゃくちゃいい図」と評価。

#### 地層パレット（「地層パレットで」「レイヤーカラーで」で起動）

レイヤー構造（大気→地層→土壌）を表現する時に使う。Hibanaブランドと整合。
帯ヘッダーの色で層の深さを表現し、カード背景は各レイヤー専用の薄色を使う。

```
レイヤー色（帯ヘッダー）:
  #1B4D7A  — 藍 Ai（大気・Orchestration）軽い、自動で漂う
  #C9943E  — 金茶 Kincha（地層・Management）固い、全員を守る大地
  #B0ACA6  — 灰白 Haijiro（土壌・Creation）落ち着いた、すべてを支える基盤

カード背景（各レイヤー内）:
  #F0F4F8  — 藍レイヤー内のカード
  #FBF6EE  — 金茶レイヤー内のカード
  #F5F2ED  — 灰白レイヤー内のカード

その他（従来と同じ）:
  #1A1816  — テキスト、黒帯背景
  #F5F2ED  — 全体背景（温石）
  #D94032  — 太郎朱（数字・エラーの差し色）
  #6B6762  — サブテキスト
  #D4D0CA  — 枠線・区切り線

定義元: BRAND_DESIGN_SYSTEM.md §1.6 レイヤーカラー
```

> 背景: Artifact Lifecycle図で黒塗りセクションが「見づらい」と指摘。
> 地層パレット（藍→金茶→土壌）に変更し「落ち着いた色で目に良い」と評価。

#### Goodpatch方式追加ルール（clean-blueプリセット使用時）

| ルール | 詳細 |
|--------|------|
| 余白が主役 | 四辺80px以上。1ページ=1メッセージ。詰めない |
| 行間1.7-2.0 | line-height: 1.8をデフォルトに。現行より40%広い |
| 角丸8-12px | ボックス・カードに角丸。シャープよりフレンドリー |
| 文中部分色変え | 太字でなく、一部フレーズだけ青にして強調 |
| テーブルは破線区切り | 実線より軽く、余白を殺さない |
| カテゴリは色タグで | 文章で説明せず、色タグで瞬時判別 |
| 仕切り=全面色 | セクション間は青ベタ全面。線で区切らない |
| 影はごく薄い | box-shadow: 0 2px 8px rgba(0,0,0,0.05) |

#### フォントサイズルール（厳守）

```
セクションラベル (IBM Plex Mono): 14px以上、font-weight 600
  ❌ 11px → 5回指摘された。絶対に使うな

本文テキスト: 14-16px
見出し: 16-22px
タイトル: 24-28px
巨大数字 (Space Grotesk): 28-48px

黒背景上のテキスト:
  主テキスト: #F5F2ED or #fff
  サブテキスト: #999 以上（#666は暗すぎ、rgba(255,255,255,0.6)は薄すぎ）
  → rgba(255,255,255,0.85) 以上を使う
```

> 背景: PROBLEM, GRADUATION, DAILY CYCLE等のラベルが11pxで「小さすぎ」と5回連続指摘。

#### 座標管理ルール

```
✅ 正しい: 全ノードのx座標を定数で定義し、SVG要素とCSS leftで同じ値を使う
✅ 複数段のボックス: x座標とwidthを段間で完全一致させる
❌ 間違い: getBoundingClientRectで動的に座標を取得する
❌ 間違い: 上段と下段でボックス幅がバラバラ
```

> 背景: Phase 1とPhase 2のボックスが揃っていなくて「四角の幅は下段と揃えて」と指摘。

#### 描画順序ルール（SVGは後に書いたものが前面）
```
Layer 0: 背景rect（最背面）
Layer 1: セクション背景（黒rect等）
Layer 2: 破線・ループ線
Layer 3: ボックス（rect）
Layer 4: 実線矢印
Layer 5: テキスト（最前面）
```

#### 矢印ラベルルール

```
✅ 矢印の上にラベルを置く（矢印の線と被らない位置）
✅ ラベルが被るなら削除して矢印だけにする（シンプルの方がクリーン）
❌ ラベルを矢印の線上に重ねる → テキストが見えない
❌ ラベルがボックスのテキストと被る → 読めない
```

> 背景: 「ヒアリング」ラベルがボックスと被り、その後「文字なしでいいかも」で矢印だけに。

#### 双方向（相互）矢印ルール

2つの要素間に相互の矢印を描く場合:

```
✅ 正しい: 2本の線を平行に配置（同角度、x方向にオフセット）
❌ 間違い: 2本が交差する、または重なる
❌ 間違い: 2本の角度が異なる（平行でない）
```

- **オフセット幅**: 最低40px
- **ラベル位置**: 各矢印の外側に配置
- **左右対称**: 左側ペアと右側ペアは鏡像配置

#### 区切り線ルール

```
セクション間の区切り線: stroke-width 1px, #D4D0CA
黒背景内の区切り線: stroke-width 2-2.5px, #F5F2ED, opacity 0.3
破線（ループ等）: stroke-width 2.5px, stroke-dasharray 8,5

❌ stroke-width 1px + #333 の区切り線 → 見えない
```

> 背景: DAILY CYCLE線が「こういう点線はもっと太く、こゆく」と指摘。

#### viewBox設計

```
1. 用途で基本サイズを決定（写真渡し=1200x900、スクリーン=1200x675）
2. セクション数 × 高さを概算して収まるか確認
3. viewBoxを確定してから描画開始。途中で変更しない
```

### Step 4: Playwrightスクショ検証 + Chrome表示（省略禁止）

**Write後、必ず両方実行:**

```python
# Playwrightスクショ
from playwright.sync_api import sync_playwright
with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page(viewport={'width': 1200, 'height': 900})
    page.goto('file:///path/to/diagram.html')
    page.wait_for_timeout(3000)
    page.screenshot(path='screenshot.png', full_page=False)
    browser.close()
```

```bash
# Chrome表示（スクショと同時に実行）
open -a "Google Chrome" /path/to/diagram.html
```

**スクショで確認すること:**
- [ ] テキストとボックスが被っていない
- [ ] セクションラベルが14px以上で読める
- [ ] 全テキストが#555以上のコントラスト（黒背景上は#999以上）
- [ ] 複数段のボックスが縦に揃っている
- [ ] 矢印ラベルがボックスや他の要素と被っていない
- [ ] 1200x900のviewBoxに全コンテンツが収まっている
- [ ] 色が赤1色以下（青・緑・金を使っていない）※Uzabase方式の場合
- [ ] clean-blue方式の場合: 青1色支配、角丸あり、余白80px以上

### Step 4.5: 成果物の永続保存（Chrome確認後に実行）

owner確認で問題なければ、`_workspace/temp/` から正式な場所にコピーして永続化する。

```
1. 保存先を決定:
   ${DIAGRAMS_DIR}/{図解名}/vN.html

2. バージョン番号を決定:
   /bin/ls ${DIAGRAMS_DIR}/{図解名}/ で既存バージョンを確認
   → 次の番号を付与（v1, v2, v3...）

3. コピー + commit:
   cp _workspace/temp/{file}.html ${DIAGRAMS_DIR}/{名前}/vN.html
   git add ${DIAGRAMS_DIR}/{名前}/
   git commit -m "docs: {図解名} vN を保存"

4. REGISTRY.yaml を更新:
   # python3 tools/artifact_manager.py registry

5. approved symlinkは設定しない（DRAFT状態で保存）
   → owner「これがいい」と言ったら:
   # python3 tools/artifact_manager.py approve ${DIAGRAMS_DIR}/{名前}/vN.html
```

**保存しない場合**: ownerが「これは試作」と言った場合は `_workspace/temp/` に残す（消えてもOK）。
**前バージョンは消さない**: v1, v2, v3 が全て残る。旧版は `archive/` に移動してもよいが削除しない。
**「資料出して」対応**: `approved` symlinkが存在すればその先を返す。なければ最新版を返す。

### Step 5: 微調整（ownerフィードバック対応）

- 座標変更は`Edit`で該当行のみ修正（`visual-artifact-guard.md` 準拠）
- **修正後は必ずPlaywrightスクショ + Chrome表示**（毎回セット）
- ファイルを上書きしない: `cp original.html original-v1.html` してから修正
- 「被ってる」→ 座標ずらし or ラベル削除
- 「薄い」→ #999 → #6B6762、rgba(0.6) → rgba(0.85)
- 「小さい」→ 11px → 14px
- 「揃ってない」→ 上段と下段のx, widthを一致させる

## 成功パターン: Darwinian Skill Evolution 図（参考）

2026-03-21に「めちゃくちゃいい図ができた！」と評価された図の構造:

```
構成:
  ROW 1: PROBLEM | SOLUTION | EXISTING A-E（3カラム、白背景）
  ROW 2: 黒背景セクション
    Phase 1: トップCA → Skill化 → 全員に展開（上段）
    区切り線（太、半透明白）
    Phase 2: 100人 → KPI → TOP Skill → 全員配布（下段、ループ矢印）
    右端: GRADUATION = Rule化（赤背景ボックス）
  ROW 3: NOW + PRODUCT（赤枠、白背景）
  ROW 4: STRUCTURAL INSIGHTS（2x2グリッド、04だけ黒背景+赤文字）

色: #1A1816, #F5F2ED, #D94032, #6B6762, #D4D0CA のみ
フォント: Space Grotesk（数字・タイトル）, IBM Plex Mono（ラベル）, Noto Sans JP（本文）
```

ファイル: `_workspace/temp/skill-evolution-visual-v2.html`

## エスカレーション

1. **Playwright検証で被りが解消しない** → ラベルを削除して矢印だけにする
2. **参考画像の再現が困難** → ownerに「FigmaまたはCanvaでの作成を推奨」
3. **1ページに収まらない** → 2ページ構成を提案。各ページ1200x900

## セキュリティ

- **APIキー/トークン**: 不要
- **PII取り扱い**: 図にPIIを含めない
- **外部アクセス**: Google Fonts CDNのみ

## 他のスキルとの連携

| スキル | 関係 | 説明 |
|--------|------|------|
| svg-diagram | 併用 | 「uzabase方式で」→ Uzabase原則（4パターン）を併用 |
| super-plan | 入力元 | 設計書のアーキテクチャ図として呼ばれることがある |
| webapp-testing | 並列 | Playwright環境を利用 |
| frontend-slides | 代替 | プレゼン用の図はfrontend-slidesに委譲 |
| leak-learner | 学習 | owner指摘をlessons/に記録 |

## アンチパターン（絶対やらない）

| やりがち | なぜダメ | 代わりに |
|---------|--------|---------|
| 3パターン同時生成 | 全部中途半端 | 1パターンを完成させる |
| 3色以上のアクセントカラー | チープに見える | モノクロ+赤1色（Uzabase）or 青1色（clean-blue） |
| セクションラベル11px | 読めない（5回指摘） | 14px + font-weight 600 |
| #999テキスト | スクリーンで薄すぎ | #6B6762以上 |
| 矢印ラベルがボックスと被る | 読めない | ラベル削除して矢印だけ |
| 上段と下段でボックス幅が違う | 不揃いに見える | x座標+widthを完全一致 |
| 区切り線が1px+暗色 | 見えない | 2.5px+半透明白 |
| viewBoxを途中で変更 | レイアウト崩壊 | 最初にセクション数から概算 |
| Playwrightだけ実行 | ownerがブラウザで見たい | Chrome表示もセットで |
| 既存ファイルを上書き | ownerの承認済み版が消える | v1,v2と別ファイルにコピー |
| スクショ確認なしでownerに見せる | 被り・ズレが必ずある | 必ずPlaywright検証後に提示 |
| 具体的な数字でマウント取る | 比較に見える | 中立な表現にする |
| テキストとボックスのサイズ不一致 | ボックスの高さにテキストが収まらない | テキスト量からボックスの高さを逆算。巨大数字(48px)を使う場合はボックスを最低120px確保 |
| 放射状矢印の被り | ハブから4方向に矢印→角度が浅くて線が被る | 縦距離を80px以上確保。矢印の起点をハブの異なるx座標から出す |
| 矢印をやめてグリッドにする | 情報の損失。親子関係が伝わらない | viewBoxの高さを広げて矢印を維持。グリッドは最終手段 |

## Config

型（Shape）と色（Palette）を独立に指定する。組み合わせ自由。

```
指定方法:
  「活版 × 地層で」      → 型: 活版 + 色: 地層
  「凪 × 海で」         → 型: 凪 + 色: 海
  「活版で」            → 型: 活版 + 色: 墨（デフォルト）
  省略                  → 活版 × 墨

後方互換:
  「uzabase方式で」     → 活版 × 墨
  「clean-blueで」      → 凪 × 海
  「地層パレットで」     → 前回の型 × 地層
```

### 型（Shape）: 活版 Kappan（デフォルト）

活字印刷。情報を密に、正確に。角なし、余白を削ぎ落とす。

```yaml
border_radius: 0             # 角丸なし（シャープ）
margin_outer: 40             # 四辺マージン(px)
line_height: 1.4             # 密な行間
card_shadow: none            # シャドウなし
content_density: high        # 情報を詰める
```

### 型（Shape）: 凪 Nagi

凪いだ海。静かで広い。余白が呼吸するレイアウト。

```yaml
border_radius: 12            # 角丸(px)
margin_outer: 80             # 四辺マージン(px)。余白が主役
line_height: 1.8             # ゆとりある行間
card_shadow: "0 2px 8px rgba(0,0,0,0.04)"  # ごく薄いシャドウ
content_density: low         # 余白を活かす
```

### 色（Palette）: 墨 Sumi（デフォルト）

モノクロ+赤1色。書道の世界。

```yaml
color_text: "#1A1816"        # 墨
color_bg: "#F5F2ED"          # 温石（全体背景）
color_accent: "#D94032"      # 太郎朱（唯一のアクセント）
color_sub: "#6B6762"         # 鼠
color_border: "#D4D0CA"      # 砂
color_card: "#FFFFFF"        # カード背景
```

### 色（Palette）: 海 Umi

深い海と珊瑚。青1色支配。

```yaml
color_text: "#333333"        # 黒
color_bg: "#FFFFFF"          # 純白
color_accent: "#2E7DD1"      # 青（構造色）
color_accent_sub: "#E85B6B"  # 赤ピンク（極少量）
color_sub: "#666666"         # 灰
color_border: "#E0E0E0"      # 薄枠
color_card: "#F5F7FA"        # 薄灰
```

### 色（Palette）: 地層 Chisou

大気→地層→土壌。インフラの層構造。

```yaml
# レイヤー帯ヘッダー色（上から下へ重くなる）
layer_sky: "#1B4D7A"         # 藍（大気・Orchestration）
layer_earth: "#C9943E"       # 金茶（地層・Management）
layer_soil: "#B0ACA6"        # 灰白（土壌・Creation）
# レイヤー内カード背景
layer_sky_card: "#F0F4F8"
layer_earth_card: "#FBF6EE"
layer_soil_card: "#F5F2ED"
# 共通
color_text: "#1A1816"        # 墨
color_bg: "#F5F2ED"          # 温石
color_sub: "#6B6762"         # 鼠
color_border: "#D4D0CA"      # 砂
color_number: "#D94032"      # 太郎朱（数字の差し色）
```

### 色（Palette）: 火花 Hibana

フルブランドカラー。外部向け資料。

```yaml
# 全Hibanaカラーを使用
color_allin: "#D94032"       # 太郎朱
color_godeep: "#1B4D7A"     # 藍
color_respect: "#C9943E"    # 金茶
color_wideopen: "#4A9ECF"   # 空
color_playhuman: "#8B6DAF"  # 紫苑
color_text: "#1A1816"
color_bg: "#F5F2ED"
color_sub: "#6B6762"
color_border: "#D4D0CA"
```

### 共通設定（型・色に依存しない）

```yaml
canvas_width: 1200
canvas_height: 900           # 写真渡し想定（675はスクリーン用）
font_title: "Space Grotesk"  # 数字・タイトル
font_label: "IBM Plex Mono"  # セクションラベル（14px以上）
font_body: "Noto Sans JP"    # 本文
min_font_label: 14
min_font_body: 14
line_width_solid: 2
line_width_dashed: 2.5
```

## 汎用性（Portability）

このスキルは汎用フレームワーク。テキスト図解ファースト・型×色分離・Playwright即検証のワークフローはプロジェクト非依存。
型（Shape）と色（Palette）を自社のブランドに合わせて追加するだけで使える。

## Zero Leak L5 接続

| スキル | 関係 | 説明 |
|--------|------|------|
| leak-learner | 学習 | owner指摘をlessons/に記録。2+スキル共通パターンはGlobal rulesに昇格 |

> lessons/ ディレクトリが存在する。leak-learnerがowner指摘を自動蓄積する書き込み先。

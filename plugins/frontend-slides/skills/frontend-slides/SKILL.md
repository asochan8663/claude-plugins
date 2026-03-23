---
name: frontend-slides
description: "【内部実装】presentation-architectから呼ばれるHTMLスライド生成エンジン。直接呼ばないこと — 「スライド作って」等はpresentation-architectが受ける。"
---

> **このスキルは直接トリガーしない。** `presentation-architect` 経由で呼ばれる内部実装。
> 「スライド作って」→ presentation-architect → (ルーティング) → frontend-slides

# Frontend Slides Skill

Create zero-dependency, animation-rich HTML presentations that run entirely in the browser. This skill helps non-designers discover their preferred aesthetic through visual exploration ("show, don't tell"), then generates production-quality slide decks.

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| ゼロ依存 | Unix哲学 | 単一HTMLファイル、inline CSS/JS。npm・ビルドツール不要 |
| Show, Don't Tell | UXデザイン原則 | 抽象的な選択肢ではなく視覚的プレビューを生成して選ばせる |
| 個性的デザイン | — | 汎用的な「AIっぽい」美学を避ける。カスタムメイド感を出す |
| プロダクション品質 | Web Standards | コメント充実・アクセシブル・パフォーマント |
| ビューポートフィット | CRITICAL | 各スライドは必ずビューポート内に収まる。スクロール禁止 |

---

## CRITICAL: Viewport Fitting Requirements

**Every slide MUST fit exactly in the viewport. No scrolling within slides, ever.**

### The Golden Rule

```
Each slide = exactly one viewport height (100vh/100dvh)
Content overflows? → Split into multiple slides or reduce content
Never scroll within a slide.
```

### Content Density Limits Per Slide

| Slide Type | Maximum Content |
|------------|-----------------|
| Title slide | 1 heading + 1 subtitle + optional tagline |
| Content slide | 1 heading + 4-6 bullet points OR 1 heading + 2 paragraphs |
| Feature grid | 1 heading + 6 cards maximum (2x3 or 3x2 grid) |
| Code slide | 1 heading + 8-10 lines of code maximum |
| Quote slide | 1 quote (max 3 lines) + attribution |
| Image slide | 1 heading + 1 image (max 60vh height) |

**If content exceeds these limits → Split into multiple slides**

### CSS & Responsive Boilerplate

**Read `STYLE_PRESETS.md` for the full mandatory CSS boilerplate** (viewport locking, clamp() typography, responsive breakpoints for 700/600/500px heights, reduced motion).

Key rules:
- `.slide` must have `height: 100vh; height: 100dvh; overflow: hidden;`
- ALL font sizes and spacing use `clamp(min, preferred, max)`
- Images: `max-height: min(50vh, 400px)`
- Grids: `auto-fit` with `minmax()` for responsive columns
- Content doesn't fit → **split into multiple slides, never scroll**

---

## Phase 0: Detect Mode

First, determine what the user wants:

**Mode A: New Presentation**
- User wants to create slides from scratch
- Proceed to Phase 1 (Content Discovery)

**Mode B: PPT Conversion**
- User has a PowerPoint file (.ppt, .pptx) to convert
- Proceed to Phase 4 (PPT Extraction)

**Mode C: Existing Presentation Enhancement（修正モード）**
- User has an HTML presentation and wants to improve it
- Read the existing file, understand the structure, then enhance
- **CRITICAL: 修正は Edit ツールで差分のみ。Write で全体再生成は絶対禁止**
- **修正前にスナップショット必須**: `cp {file} {file%.*}-v{N}-snapshot.html`
- **サブエージェントへの委譲禁止**: 親セッションで直接 Edit する
- CSSのデザイン品質（spacing, shadow, border-radius等）はテキスト指示では再現不能。全体再生成すると品質が劣化する
- > 背景: 2026-03-22インシデント。v2→v3→v4と3回全体再生成して品質が回復不能に劣化

---

## Phase 1: Content Discovery (New Presentations)

Before designing, understand the content. Ask via AskUserQuestion:

### Step 1.1: Presentation Context

**Question 1: Purpose**
- Header: "Purpose"
- Question: "What is this presentation for?"
- Options:
  - "Pitch deck" — Selling an idea, product, or company to investors/clients
  - "Teaching/Tutorial" — Explaining concepts, how-to guides, educational content
  - "Conference talk" — Speaking at an event, tech talk, keynote
  - "Internal presentation" — Team updates, strategy meetings, company updates

**Question 2: Slide Count**
- Header: "Length"
- Question: "Approximately how many slides?"
- Options:
  - "Short (5-10)" — Quick pitch, lightning talk
  - "Medium (10-20)" — Standard presentation
  - "Long (20+)" — Deep dive, comprehensive talk

**Question 3: Content**
- Header: "Content"
- Question: "Do you have the content ready, or do you need help structuring it?"
- Options:
  - "I have all content ready" — Just need to design the presentation
  - "I have rough notes" — Need help organizing into slides
  - "I have a topic only" — Need help creating the full outline

If user has content, ask them to share it (text, bullet points, images, etc.).

---

## Phase 2: Style Discovery (Visual Exploration)

**CRITICAL: This is the "show, don't tell" phase.**

Most people can't articulate design preferences in words. Instead of asking "do you want minimalist or bold?", we generate mini-previews and let them react.

### How Users Choose Presets

Users can select a style in **two ways**:

**Option A: Guided Discovery (Default)**
- User answers mood questions
- Skill generates 3 preview files based on their answers
- User views previews in browser and picks their favorite
- This is best for users who don't have a specific style in mind

**Option B: Direct Selection**
- If user already knows what they want, they can request a preset by name
- Example: "Use the Bold Signal style" or "I want something like Dark Botanical"
- Skip to Phase 3 immediately

**Available Presets:**
| Preset | Vibe | Best For |
|--------|------|----------|
| Bold Signal | Confident, high-impact | Pitch decks, keynotes |
| Electric Studio | Clean, professional | Agency presentations |
| Creative Voltage | Energetic, retro-modern | Creative pitches |
| Dark Botanical | Elegant, sophisticated | Premium brands |
| Notebook Tabs | Editorial, organized | Reports, reviews |
| Pastel Geometry | Friendly, approachable | Product overviews |
| Split Pastel | Playful, modern | Creative agencies |
| Vintage Editorial | Witty, personality-driven | Personal brands |
| Neon Cyber | Futuristic, techy | Tech startups |
| Terminal Green | Developer-focused | Dev tools, APIs |
| Swiss Modern | Minimal, precise | Corporate, data |
| Paper & Ink | Literary, thoughtful | Storytelling |

### Step 2.0: Style Path Selection

First, ask how the user wants to choose their style:

**Question: Style Selection Method**
- Header: "Style"
- Question: "How would you like to choose your presentation style?"
- Options:
  - "Show me options" — Generate 3 previews based on my needs (recommended for most users)
  - "I know what I want" — Let me pick from the preset list directly

**If "Show me options"** → Continue to Step 2.1 (Mood Selection)

**If "I know what I want"** → Show preset picker:

**Question: Pick a Preset**
- Header: "Preset"
- Question: "Which style would you like to use?"
- Options:
  - "Bold Signal" — Vibrant card on dark, confident and high-impact
  - "Dark Botanical" — Elegant dark with soft abstract shapes
  - "Notebook Tabs" — Editorial paper look with colorful section tabs
  - "Pastel Geometry" — Friendly pastels with decorative pills

(If user picks one, skip to Phase 3. If they want to see more options, show additional presets or proceed to guided discovery.)

### Step 2.1: Mood Selection (Guided Discovery)

**Question 1: Feeling**
- Header: "Vibe"
- Question: "What feeling should the audience have when viewing your slides?"
- Options:
  - "Impressed/Confident" — Professional, trustworthy, this team knows what they're doing
  - "Excited/Energized" — Innovative, bold, this is the future
  - "Calm/Focused" — Clear, thoughtful, easy to follow
  - "Inspired/Moved" — Emotional, storytelling, memorable
- multiSelect: true (can choose up to 2)

### Step 2.2: Generate Style Previews

Based on their mood selection, generate **3 distinct style previews** as mini HTML files in a temporary directory. Each preview should be a single title slide showing:

- Typography (font choices, heading/body hierarchy)
- Color palette (background, accent, text colors)
- Animation style (how elements enter)
- Overall aesthetic feel

**Preview Styles to Consider (pick 3 based on mood):**

| Mood | Style Options |
|------|---------------|
| Impressed/Confident | "Bold Signal", "Electric Studio", "Dark Botanical" |
| Excited/Energized | "Creative Voltage", "Neon Cyber", "Split Pastel" |
| Calm/Focused | "Notebook Tabs", "Paper & Ink", "Swiss Modern" |
| Inspired/Moved | "Dark Botanical", "Vintage Editorial", "Pastel Geometry" |

**IMPORTANT: Never use these generic patterns:**
- Purple gradients on white backgrounds
- Inter, Roboto, or system fonts
- Standard blue primary colors
- Predictable hero layouts

**Instead, use distinctive choices:**
- Unique font pairings (Clash Display, Satoshi, Cormorant Garamond, DM Sans, etc.)
- Cohesive color themes with personality
- Atmospheric backgrounds (gradients, subtle patterns, depth)
- Signature animation moments

### Step 2.3: Present Previews

Create the previews in: `.claude-design/slide-previews/`

```
.claude-design/slide-previews/
├── style-a.html   # First style option
├── style-b.html   # Second style option
├── style-c.html   # Third style option
└── assets/        # Any shared assets
```

Each preview file should be:
- Self-contained (inline CSS/JS)
- A single "title slide" showing the aesthetic
- Animated to demonstrate motion style
- ~50-100 lines, not a full presentation

Present to user:
```
I've created 3 style previews for you to compare:

**Style A: [Name]** — [1 sentence description]
**Style B: [Name]** — [1 sentence description]
**Style C: [Name]** — [1 sentence description]

Open each file to see them in action:
- .claude-design/slide-previews/style-a.html
- .claude-design/slide-previews/style-b.html
- .claude-design/slide-previews/style-c.html

Take a look and tell me:
1. Which style resonates most?
2. What do you like about it?
3. Anything you'd change?
```

Then use AskUserQuestion:

**Question: Pick Your Style**
- Header: "Style"
- Question: "Which style preview do you prefer?"
- Options:
  - "Style A: [Name]" — [Brief description]
  - "Style B: [Name]" — [Brief description]
  - "Style C: [Name]" — [Brief description]
  - "Mix elements" — Combine aspects from different styles

If "Mix elements", ask for specifics.

---

## Phase 3: Generate Presentation

Now generate the full presentation based on:
- Content from Phase 1
- Style from Phase 2

### File Structure

For single presentations:
```
presentation.html    # Self-contained presentation
assets/              # Images, if any
```

For projects with multiple presentations:
```
[presentation-name].html
[presentation-name]-assets/
```

### HTML Architecture

Follow this structure for all presentations:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Presentation Title</title>

    <!-- Fonts (use Fontshare or Google Fonts) -->
    <link rel="stylesheet" href="https://api.fontshare.com/v2/css?f[]=...">

    <style>
        /* ===========================================
           CSS CUSTOM PROPERTIES (THEME)
           Easy to modify: change these to change the whole look
           =========================================== */
        :root {
            /* Colors */
            --bg-primary: #0a0f1c;
            --bg-secondary: #111827;
            --text-primary: #ffffff;
            --text-secondary: #9ca3af;
            --accent: #00ffcc;
            --accent-glow: rgba(0, 255, 204, 0.3);

            /* Typography - MUST use clamp() for responsive scaling */
            --font-display: 'Clash Display', sans-serif;
            --font-body: 'Satoshi', sans-serif;
            --title-size: clamp(2rem, 6vw, 5rem);
            --subtitle-size: clamp(0.875rem, 2vw, 1.25rem);
            --body-size: clamp(0.75rem, 1.2vw, 1rem);

            /* Spacing - MUST use clamp() for responsive scaling */
            --slide-padding: clamp(1.5rem, 4vw, 4rem);
            --content-gap: clamp(1rem, 2vw, 2rem);

            /* Animation */
            --ease-out-expo: cubic-bezier(0.16, 1, 0.3, 1);
            --duration-normal: 0.6s;
        }

        /* ===========================================
           BASE STYLES
           =========================================== */
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        html {
            scroll-behavior: smooth;
            scroll-snap-type: y mandatory;
            height: 100%;
        }

        body {
            font-family: var(--font-body);
            background: var(--bg-primary);
            color: var(--text-primary);
            overflow-x: hidden;
            height: 100%;
        }

        /* ===========================================
           SLIDE CONTAINER
           CRITICAL: Each slide MUST fit exactly in viewport
           - Use height: 100vh (NOT min-height)
           - Use overflow: hidden to prevent scroll
           - Content must scale with clamp() values
           =========================================== */
        .slide {
            width: 100vw;
            height: 100vh; /* EXACT viewport height - no scrolling */
            height: 100dvh; /* Dynamic viewport height for mobile */
            padding: var(--slide-padding);
            scroll-snap-align: start;
            display: flex;
            flex-direction: column;
            justify-content: center;
            position: relative;
            overflow: hidden; /* Prevent any content overflow */
        }

        /* Content wrapper that prevents overflow */
        .slide-content {
            flex: 1;
            display: flex;
            flex-direction: column;
            justify-content: center;
            max-height: 100%;
            overflow: hidden;
        }

        /* ===========================================
           RESPONSIVE BREAKPOINTS
           Adjust content for different screen sizes
           =========================================== */
        @media (max-height: 600px) {
            :root {
                --slide-padding: clamp(1rem, 3vw, 2rem);
                --content-gap: clamp(0.5rem, 1.5vw, 1rem);
            }
        }

        @media (max-width: 768px) {
            :root {
                --title-size: clamp(1.5rem, 8vw, 3rem);
            }
        }

        @media (max-height: 500px) and (orientation: landscape) {
            /* Extra compact for landscape phones */
            :root {
                --title-size: clamp(1.25rem, 5vw, 2rem);
                --slide-padding: clamp(0.75rem, 2vw, 1.5rem);
            }
        }

        /* ===========================================
           ANIMATIONS
           Trigger via .visible class (added by JS on scroll)
           =========================================== */
        .reveal {
            opacity: 0;
            transform: translateY(30px);
            transition: opacity var(--duration-normal) var(--ease-out-expo),
                        transform var(--duration-normal) var(--ease-out-expo);
        }

        .slide.visible .reveal {
            opacity: 1;
            transform: translateY(0);
        }

        /* Stagger children */
        .reveal:nth-child(1) { transition-delay: 0.1s; }
        .reveal:nth-child(2) { transition-delay: 0.2s; }
        .reveal:nth-child(3) { transition-delay: 0.3s; }
        .reveal:nth-child(4) { transition-delay: 0.4s; }

        /* ... more styles ... */
    </style>
</head>
<body>
    <!-- Progress bar (optional) -->
    <div class="progress-bar"></div>

    <!-- Navigation dots (optional) -->
    <nav class="nav-dots">
        <!-- Generated by JS -->
    </nav>

    <!-- Slides -->
    <section class="slide title-slide">
        <h1 class="reveal">Presentation Title</h1>
        <p class="reveal">Subtitle or author</p>
    </section>

    <section class="slide">
        <h2 class="reveal">Slide Title</h2>
        <p class="reveal">Content...</p>
    </section>

    <!-- More slides... -->

    <script>
        /* ===========================================
           SLIDE PRESENTATION CONTROLLER
           Handles navigation, animations, and interactions
           =========================================== */

        class SlidePresentation {
            constructor() {
                // ... initialization
            }

            // ... methods
        }

        // Initialize
        new SlidePresentation();
    </script>
</body>
</html>
```

### Required JavaScript Features

Every presentation should include:

1. **SlidePresentation Class** — Main controller
   - Keyboard navigation (arrows, space)
   - Touch/swipe support
   - Mouse wheel navigation
   - Progress bar updates
   - Navigation dots

2. **Intersection Observer** — For scroll-triggered animations
   - Add `.visible` class when slides enter viewport
   - Trigger CSS animations efficiently

3. **Optional Enhancements** (based on style):
   - Custom cursor with trail
   - Particle system background (canvas)
   - Parallax effects
   - 3D tilt on hover
   - Magnetic buttons
   - Counter animations

### Code Quality Requirements

**Comments:**
Every section should have clear comments explaining:
- What it does
- Why it exists
- How to modify it

```javascript
/* ===========================================
   CUSTOM CURSOR
   Creates a stylized cursor that follows mouse with a trail effect.
   - Uses lerp (linear interpolation) for smooth movement
   - Grows larger when hovering over interactive elements
   =========================================== */
class CustomCursor {
    constructor() {
        // ...
    }
}
```

**Accessibility:**
- Semantic HTML (`<section>`, `<nav>`, `<main>`)
- Keyboard navigation works
- ARIA labels where needed
- Reduced motion support

```css
@media (prefers-reduced-motion: reduce) {
    .reveal {
        transition: opacity 0.3s ease;
        transform: none;
    }
}
```

**Responsive & Viewport Fitting (CRITICAL):**

**See the "CRITICAL: Viewport Fitting Requirements" section above for complete CSS and guidelines.**

Quick reference:
- Every `.slide` must have `height: 100vh; height: 100dvh; overflow: hidden;`
- All typography and spacing must use `clamp()`
- Respect content density limits (max 4-6 bullets, max 6 cards, etc.)
- Include breakpoints for heights: 700px, 600px, 500px
- When content doesn't fit → split into multiple slides, never scroll

---

## Phase 4: PPT Conversion

When converting PowerPoint files:

### Step 4.1: Extract Content

Use Python with `python-pptx` to extract:

```python
from pptx import Presentation
from pptx.util import Inches, Pt
import json
import os
import base64

def extract_pptx(file_path, output_dir):
    """
    Extract all content from a PowerPoint file.
    Returns a JSON structure with slides, text, and images.
    """
    prs = Presentation(file_path)
    slides_data = []

    # Create assets directory
    assets_dir = os.path.join(output_dir, 'assets')
    os.makedirs(assets_dir, exist_ok=True)

    for slide_num, slide in enumerate(prs.slides):
        slide_data = {
            'number': slide_num + 1,
            'title': '',
            'content': [],
            'images': [],
            'notes': ''
        }

        for shape in slide.shapes:
            # Extract title
            if shape.has_text_frame:
                if shape == slide.shapes.title:
                    slide_data['title'] = shape.text
                else:
                    slide_data['content'].append({
                        'type': 'text',
                        'content': shape.text
                    })

            # Extract images
            if shape.shape_type == 13:  # Picture
                image = shape.image
                image_bytes = image.blob
                image_ext = image.ext
                image_name = f"slide{slide_num + 1}_img{len(slide_data['images']) + 1}.{image_ext}"
                image_path = os.path.join(assets_dir, image_name)

                with open(image_path, 'wb') as f:
                    f.write(image_bytes)

                slide_data['images'].append({
                    'path': f"assets/{image_name}",
                    'width': shape.width,
                    'height': shape.height
                })

        # Extract notes
        if slide.has_notes_slide:
            notes_frame = slide.notes_slide.notes_text_frame
            slide_data['notes'] = notes_frame.text

        slides_data.append(slide_data)

    return slides_data
```

### Step 4.2: Confirm Content Structure

Present the extracted content to the user:

```
I've extracted the following from your PowerPoint:

**Slide 1: [Title]**
- [Content summary]
- Images: [count]

**Slide 2: [Title]**
- [Content summary]
- Images: [count]

...

All images have been saved to the assets folder.

Does this look correct? Should I proceed with style selection?
```

### Step 4.3: Style Selection

Proceed to Phase 2 (Style Discovery) with the extracted content in mind.

### Step 4.4: Generate HTML

Convert the extracted content into the chosen style, preserving:
- All text content
- All images (referenced from assets folder)
- Slide order
- Any speaker notes (as HTML comments or separate file)

---

## Phase 5: Delivery

### Final Output

When the presentation is complete:

1. **Clean up temporary files**
   - Delete `.claude-design/slide-previews/` if it exists

2. **Open the presentation**
   - Use `open [filename].html` to launch in browser

3. **Provide summary**
```
Your presentation is ready!

📁 File: [filename].html
🎨 Style: [Style Name]
📊 Slides: [count]

**Navigation:**
- Arrow keys (← →) or Space to navigate
- Scroll/swipe also works
- Click the dots on the right to jump to a slide

**To customize:**
- Colors: Look for `:root` CSS variables at the top
- Fonts: Change the Fontshare/Google Fonts link
- Animations: Modify `.reveal` class timings

Would you like me to make any adjustments?
```

---

## Style & Animation Reference

**Read `ANIMATION_REFERENCE.md`** for:
- Effect → feeling mapping (dramatic, techy, playful, professional, calm, editorial)
- Entrance animation CSS patterns (fade, scale, slide, blur)
- Background effect CSS (gradient mesh, noise, grid)
- Interactive JS effects (3D tilt on hover)

---

## Troubleshooting

### Common Issues

**Fonts not loading:**
- Check Fontshare/Google Fonts URL
- Ensure font names match in CSS

**Animations not triggering:**
- Verify Intersection Observer is running
- Check that `.visible` class is being added

**Scroll snap not working:**
- Ensure `scroll-snap-type` on html/body
- Each slide needs `scroll-snap-align: start`

**Mobile issues:**
- Disable heavy effects at 768px breakpoint
- Test touch events
- Reduce particle count or disable canvas

**Performance issues:**
- Use `will-change` sparingly
- Prefer `transform` and `opacity` animations
- Throttle scroll/mousemove handlers

---

## 合成可能性

| 連携スキル | 関係 | トリガー |
|-----------|------|---------|
| super-plan | 前工程 | プレゼン用の設計書がsuper-planで承認された後、スライド生成に進む |
| rapid-build | 並列 | 実装と同時にプレゼン資料を作成する場合 |
| scene-image-gen | 前工程 | スライド用のカスタム画像をAI生成してから埋め込む |
| pdf-slide-viewer | 後工程 | 生成したHTMLスライドをPDF化して確認 |
| shachiku-slides | 並列 | 社畜ちゃんチャンネル向けスライドの場合はそちらに委譲 |

---

## Example Session Flow

1. User: "I want to create a pitch deck for my AI startup"
2. Skill asks about purpose, length, content
3. User shares their bullet points and key messages
4. Skill asks about desired feeling (Impressed + Excited)
5. Skill generates 3 style previews
6. User picks Style B (Neon Cyber), asks for darker background
7. Skill generates full presentation with all slides
8. Skill opens the presentation in browser
9. User requests tweaks to specific slides
10. Final presentation delivered

---

## Conversion Session Flow

1. User: "Convert my slides.pptx to a web presentation"
2. Skill extracts content and images from PPT
3. Skill confirms extracted content with user
4. Skill asks about desired feeling/style
5. Skill generates style previews
6. User picks a style
7. Skill generates HTML presentation with preserved assets
8. Final presentation delivered

## Config

| カテゴリ | キー | デフォルト値 | 説明 |
|---------|------|------------|------|
| パス | preview_dir | `.claude-design/slide-previews/` | スタイルプレビュー出力先 |
| ビューポート | slide_height | `100vh` / `100dvh` | スライド1枚の高さ（固定） |
| タイポグラフィ | title_size | `clamp(2rem, 6vw, 5rem)` | タイトルの応答的フォントサイズ |
| タイポグラフィ | body_size | `clamp(0.75rem, 1.2vw, 1rem)` | 本文の応答的フォントサイズ |
| レイアウト | max_bullets | 6 | 1スライドあたりの最大箇条書き数 |
| レイアウト | max_grid_cards | 6 | 1スライドあたりの最大カード数 |
| 画像 | max_image_height | `min(50vh, 400px)` | スライド内画像の最大高さ |

## セキュリティ

N/A -- 外部通信・認証なし。Fontshare/Google Fontsへのリンクのみ（CDN読み込み）。

## BLOCKINGゲート

| Step | 失敗条件 | 動作 |
|------|---------|------|
| Phase 0 | PPTXファイルが存在しない / python-pptx未インストール | STOP + `pip install python-pptx` を案内 |
| Phase 2 | プレビュー生成失敗（フォント読み込みエラー等） | 続行: デフォルトプリセット(Bold Signal)で生成 |
| Phase 3 | コンテンツが1スライドに収まらない | 続行: 自動的に複数スライドに分割（スクロール禁止） |
| Phase 5 | `open` コマンドでブラウザ起動失敗 | 続行: ファイルパスを表示して手動オープンを案内 |

## エスカレーション

| 状況 | 対応 |
|------|------|
| ユーザーが12プリセットのどれにも満足しない | ownerに確認: 「参考にしたいWebサイトやデザインのURLを教えてください」 |
| PPT内の画像が破損・欠損 | ownerに確認: 「{N}枚の画像が抽出できませんでした。元ファイルを確認してください」 |
| スライド数が30枚超 | ownerに確認: 「パフォーマンス影響があります。分割を推奨しますか？」 |

## やらないこと

- スライド内スクロールの許容（ビューポートフィッティングは絶対遵守）
- npm/Webpack等のビルドツール依存（常にゼロ依存の単一HTML）
- 動画・音声の埋め込み（静的HTML + CSS + JSのみ）
- サーバーサイド処理（全てクライアントサイドで完結）

## ブランド適用: Hibana（Agent Native）

Agent Native関連のスライド・プレゼンテーションを作成する場合:

1. **BLOCKING**: 実行前に `${BRAND_DESIGN_SYSTEM}` をReadする
2. Hibanaパレットの色・フォント・コンポーネントパターンを適用する
3. CSS Custom Properties（Section 7）をそのまま使用する
4. ハードコード禁止 — 色・フォントは必ずBRAND_DESIGN_SYSTEM.mdの値を参照
5. 営業向け=Light（温石ベース）、ピッチ向け=Dark（墨ベース）で使い分け

トリガー: 「Hibanaで」「Agent Nativeの」「ブランド適用して」

## プロダクト思想適用（Agent Native）

Agent Native関連の提案スライド・ピッチデッキを作成する場合:

1. **BLOCKING**: 実行前に `${PRODUCT_PHILOSOPHY}` をReadする
2. MVV（Purpose/Mission/Tagline/Vision/Values）を資料に反映する
3. 3レイヤー（Creation → Management → Orchestration）の図を含める
4. Agent Manager概念（コードは管理できない。でもAgentは管理できる）を含める
5. 差別化（Vertical First / No-Engineer / Reliability First / Outcomes-based）を含める

トリガー: 「Agent Nativeの提案」「ピッチデッキ」「営業資料」

## 他のスキルとの連携

| スキル | 関係 | 説明 |
|--------|------|------|
| super-plan | 前工程 | プレゼン用の設計書がsuper-planで承認された後、スライド生成に進む |
| rapid-build | 並列 | 実装と同時にプレゼン資料を作成する場合 |
| scene-image-gen | 前工程 | スライド用のカスタム画像をAI生成してから埋め込む |
| pdf-slide-viewer | 後工程 | 生成したHTMLスライドをPDF化して確認 |
| shachiku-slides | 並列 | 社畜ちゃんチャンネル向けスライドの場合はそちらに委譲 |
| presentation-architect | 上位 | 会社説明・提案書のオーケストレーターがこのスキルを呼び出す |
| leak-learner | 学習 | owner指摘をlessons/に記録。2+スキル共通パターンはGlobal rulesに昇格 |
| pre-check | 前処理 | スライド作成開始前に過去インシデント・品質基準を確認 |

> 合成可能性セクションの内容を連携テーブルに統合し、leak-learner行を追加。

## 汎用性（Portability）

このスキルは **汎用フレームワーク** — スタイルプリセット12種・ビューポートフィッティング・アニメーション・PPT変換のロジックはプロジェクト非依存。
他社はそのまま使える。Hibana/Agent Native固有の設定は「ブランド適用」「プロダクト思想適用」セクションに分離されており、該当しないプロジェクトではスキップされる。

Configの値（preview_dir, フォントサイズ等）は全て汎用デフォルト。企業名ハードコードなし。

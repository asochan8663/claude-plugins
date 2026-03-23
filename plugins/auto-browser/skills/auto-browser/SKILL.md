---
name: auto-browser
description: "ブラウザ自動化の強化版。ゴール設計・ツール選択・失敗パターンDB・サービス別プレイブック・自己改善を、あらゆるブラウザCLIツールに追加する。"
triggers:
  - "OAuth setup"
  - "scope"
  - "token"
  - "setup"
  - "admin console"
  - "browser automation"
  - "manual step"
  - "auto-browser"
  - "OAuth設定"
  - "スコープ追加"
  - "トークン取得"
  - "管理画面"
  - "ブラウザで操作"
  - "手動でやって"
  - "セットアップ"
  - "API設定"
  - "再インストール"
  - "auto-setup"
allowed-tools: Bash(agent-browser:*), Bash(browser-use:*), Bash(playwright:*), Read, Write, Edit, WebSearch
---

# Auto Browser

ブラウザ自動化の強化スキル。あらゆるブラウザCLIツールに以下を追加:
- **ゴール設計** (Phase 0) — 必要な権限を全て把握してから開始
- **ツール選択** (Phase 1) — CLI > ブラウザ。最適なツールを自動選択
- **DOM分析優先** (Phase 2) — クリック前にページを理解する
- **3ストライク制** (Phase 3) — ツール切替 or エスカレーション。空回りしない
- **操作パターン集** (Phase 4) — よくあるタスクの再利用可能レシピ
- **自己改善** — 失敗から自動学習

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| 設定の外部化 | 12-Factor App | ハードコードなし。環境固有の値は全てConfigセクションに |
| 単一責任 | SOLID | このスキルは1つの仕事: ブラウザ自動化の成功率最大化 |
| 合成可能性 | Unix哲学 | あらゆるブラウザツールで動作。プレイブックはモジュラー |
| セキュリティ前倒し | OWASP | トークンをログに出さない。URL検証してから開く。フィッシング確認 |
| 再利用 > 再作成 | Unix哲学 | コマンド構文は既存ツールスキルを参照 |

## Config

環境に合わせてカスタマイズ。**以下のフェーズにパスやクレデンシャルを直書きしない。**

```yaml
# 主要ブラウザ自動化ツール（自動検出）
primary_tool: agent-browser    # 他の選択肢: browser-use, playwright
fallback_tool: browser-use     # 他の選択肢: playwright, puppeteer

# ツール検出（スキル起動時に実行）
# `which {tool_name}` で自動検出
# 明示的にパスを指定する場合:
# primary_tool_path: /usr/local/bin/agent-browser
# fallback_tool_path: /usr/local/bin/browser-use

# 認証済みセッション用ブラウザプロファイル
browser_profile: "Default"

# タイムアウト
step_timeout: 300              # 1アプローチあたり最大5分。超えたら即次へ（秒）
default_wait: 5000             # ページロード待機（ミリ秒）
heavy_spa_wait: 10000          # YouTube Studio, GCPコンソール等（ミリ秒）

# エスカレーション前の最大リトライ回数
max_retries: 3

# プレイブックディレクトリ（サービス別ガイド）
playbook_dir: ./playbooks/     # このSKILL.mdからの相対パス

# ナビゲーション妨害拡張機能リスト（検知したらCDP直接接続に切替）
nav_hijack_extensions:
  - Workona
  - OneTab
  - Session Buddy
  - Toby
```

### ツール検出（スキル起動時に実行）

```bash
# 利用可能なツールを自動検出
which agent-browser && echo "PRIMARY: agent-browser"
which browser-use && echo "FALLBACK: browser-use"
which playwright && echo "FALLBACK: playwright"
```

## Phase 0: ゴール設計 + 環境プリフライト（必須 — 絶対にスキップしない）

ブラウザを触る前に、以下を**全て**完了する。このフェーズのスキップが時間浪費の最大原因。

### 0a. ゴール定義
```
1. ゴールを一文で定義
   「完了時に何が真であるべきか？」
   例: 「Botが#generalにメッセージ投稿でき、ユーザープロフィールを読める」

2. 必要なAPI呼び出し + 権限を全て列挙
   各API呼び出しについて:
   - どのスコープ/権限が必要か？
   - 公式ドキュメントまたはWebSearchで確認

3. ギャップ分析
   - 既に持っている権限/トークンは？
   - 足りないものは？ = 今から設定するもの

4. 開始前にセットアップ経路を理解する
   - スコープ追加に再インストールが必要か？（Slack: YES）
   - 管理者承認が必要か？（GCP IAM: 場合による）
   - CLIショートカットがあるか？（gh, gcloud, slack CLI）
   - 手動承認が必要ならユーザーに伝える
```

### 0b. 環境プリフライトチェック（BLOCKING — ブラウザ起動前に必ず実行）

```bash
# 1. ナビゲーション妨害拡張の検知
PROFILE_DIR="$HOME/Library/Application Support/Google/Chrome/Default"
EXTENSIONS_DIR="$PROFILE_DIR/Extensions"
if [ -d "$EXTENSIONS_DIR" ]; then
  # Workona等のタブマネージャーを検知
  for ext_dir in "$EXTENSIONS_DIR"/*/; do
    manifest="$ext_dir"/*/manifest.json
    if [ -f $manifest ]; then
      name=$(python3 -c "import json; print(json.load(open('$manifest')).get('name',''))" 2>/dev/null)
      case "$name" in
        *Workona*|*OneTab*|*Session*Buddy*|*Toby*)
          echo "WARNING: ナビゲーション妨害拡張検知: $name"
          echo "→ browser-use -b real は動作しない可能性大。CDP直接接続を推奨"
          ;;
      esac
    fi
  done
fi

# 2. Chrome remote debugging ポートの空き確認
lsof -i :9222 2>/dev/null && echo "WARNING: port 9222 in use" || echo "OK: port 9222 available"

# 3. browser-useセッションサーバーの状態確認
browser-use doctor 2>&1 | tail -5
```

**プリフライト結果の判定:**
| 結果 | 対応 |
|------|------|
| ナビゲーション妨害拡張あり | **browser-use -b real をスキップ** → CDP直接接続（Phase 1B）へ |
| port 9222使用中 | 既存プロセスをkill or 別ポート使用 |
| browser-use doctor失敗 | CDP直接接続にフォールバック |

**なぜCDP直接接続で回避できるか:**
```
browser-use -b real → 既存Chromeにそのまま接続 → Workona有効 → ナビ乗っ取り → 失敗

CDP直接接続 → Chromeを --disable-extensions で起動 → Workona無効 → ナビ正常
              + プロファイルコピーで Cookie維持 → ログイン済み
              + Playwright CDP接続 → JS実行でページ操作
```
つまり「拡張を無効化」+「認証を維持」を両立するのがCDP直接接続の強み。

> **アンチパターン**: 1つの権限で開始 → missing_scope → 追加 → また別のmissing_scope → 追加... 1回で済むはずが N回の再インストール。

> **実例（Slack 2026-03-16）**: `incoming-webhook` だけで開始 → `chat:write` 不足 → `channels:join` 不足 → `users:read` 不足 → 4回の再インストール。Phase 0 で全スコープ洗い出していれば1回で済んだ。

## Phase 1: ツール選択（5分ルール厳守）

最もシンプルなツールを選ぶ。ブラウザは**最終手段**であって最初の選択肢ではない。

> **5分ルール**: 1つのアプローチが5分で成果を出さなければ即次へ。ツール修復に入らない。
> 「ツールを直す」より「別の手段で目的を達成する」方が常に速い。
> 背景: 2026-03-18インシデント。browser-useのセッションサーバー修復に15分浪費。

### ツール選択フロー

```
Q1: CLIツールで対応できるか？
  → YES → それを使う（gh, gcloud, slack, aws, curl 等）
  → NO → Q2へ

Q2: Phase 0bでナビゲーション妨害拡張を検知したか？
  → YES → Phase 1Bへ（CDP直接接続。browser-useをスキップ）
  → NO → Q3へ

Q3: 認証済みブラウザセッションが必要か？
  → YES → browser-use -b real（5分上限）→ 失敗 → Phase 1Bへ
  → NO → Q4へ

Q4: Cookie export/import や Python スクリプティングが必要か？
  → YES → browser-use（強力なCookie管理 + Python統合）
  → NO → primary_tool（デフォルト）
```

### Phase 1B: CDP直接接続（browser-use失敗時の確実なフォールバック）

browser-useが動かない場合の**確実な**代替手段。追加インフラ不要。

```bash
# Step 1: 既存Chromeを終了
osascript -e 'quit app "Google Chrome"'; sleep 5

# Step 2: 認証済みプロファイルをコピー（non-default data directory制約回避）
REAL_PROFILE="$HOME/Library/Application Support/Google/Chrome/Default"
TEMP_DIR="/tmp/chrome-cdp-session"
[ ! -d "$TEMP_DIR" ] && cp -r "$REAL_PROFILE" "$TEMP_DIR/Default" && mkdir -p "$TEMP_DIR"

# Step 3: remote debugging付きでChrome起動
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$TEMP_DIR" \
  --profile-directory="Default" \
  --disable-extensions \
  --no-first-run \
  "https://target-url" &

# Step 4: 接続確認（8秒待機）
sleep 8
curl -s http://localhost:9222/json/version | python3 -c "import sys,json; print('Connected:', json.load(sys.stdin).get('Browser','?'))"

# Step 5: Playwright CDP接続でJS実行
python3 << 'PYEOF'
import asyncio
from playwright.async_api import async_playwright

async def main():
    p = await async_playwright().start()
    browser = await p.chromium.connect_over_cdp("http://localhost:9222")
    page = browser.contexts[0].pages[0]
    # ここからページ操作
    print(await page.title())
    await p.stop()

asyncio.run(main())
PYEOF
```

**前提**: `playwright` が Pythonにインストール済み。なければ:
```bash
uv tool install browser-use --with playwright
# または
pip install playwright && python -m playwright install chromium
```

> **背景（2026-03-18インシデント）**: browser-use -b real がWorkona拡張のナビゲーションハイジャックで動作不能。
> 8つの代替手段を試行して46分浪費。最終的にCDP直接接続で5分で解決。
> **最初からCDP直接接続に行けば8分で完了していた。**

| シナリオ | 推奨ツール | 理由 |
|---------|-----------|------|
| GitHub Secrets | `gh secret set` | ブラウザ不要 |
| GCP IAM | `gcloud` CLI | ブラウザ不要 |
| OAuth同意画面 | browser-use real → **CDP直接接続** | 「許可」クリックが必要。拡張機能問題あればCDP |
| ページからトークン抽出 | CDP `page.evaluate()` | JS evalが最も確実 |
| ツール間のCookie転送 | browser-use `cookies export` | 最強のCookie管理 |
| 重いSPA（YouTube Studio） | CDP直接接続 + 長wait | ナビ拡張の影響を受けない |
| Slack App設定 | **CDP直接接続** | Workona等があると原理的にbrowser-use不可 |

## Phase 2: DOM分析優先（盲目的クリック禁止）

**操作前に必ずページを理解する。**

```bash
# Step 1: 開いて待つ
{tool} open "https://target-url"
{tool} wait {default_wait}

# Step 2: インタラクティブ要素を取得
{tool} snapshot    # agent-browser: snapshot -i（@ref要素を返す）
                   # browser-use: state（インデックス付き要素を返す）

# Step 3: 空なら → もう少し待つ
{tool} wait {heavy_spa_wait}
{tool} snapshot

# Step 4: それでも空なら → JSで確認
{tool} eval "document.querySelectorAll('button, input, a, select').length"
```

### なぜこれが重要か

SPA（React, Angular, Vue）は非同期レンダリング。初期ロード時のDOMは空。レンダリング完了前の操作 → 空結果 → 無駄なリトライ。

### ツール出力比較

```
agent-browser snapshot -i:
  - button "Sign In" [ref=e1]
  - input "Email" [ref=e2]
  → 2行。トークン消費: ~50

browser-use state:
  [0] <button class="btn btn-primary" ...>Sign In</button>
  [1] <input type="email" name="email" ...>
  ... (数百行続く)
  → 100行超。トークン消費: ~2000
```

## Phase 3: 5分ルール + 3ストライク（空回りではなくエスカレーション）

```
5分ルール（最優先）:
  1つのアプローチが5分で成果を出さなければ → 即次のレベルへ
  ツール修復には絶対に入らない

3ストライク:
  同じアプローチが3回失敗（空結果、エラー、間違った要素）
  → 即座に次のレベルに切り替え

エスカレーション階梯（固定・変更不可）:
  レベル1: browser-use -b real（5分上限）→ 失敗
  レベル2: CDP直接接続（Phase 1B）（5分上限）→ 失敗
  レベル3: ownerに手動手順を提示

禁止: レベル1で失敗した後に「別のプロファイル」「about:blank」「--disable-extensions」
      等の場当たり的な変種を試すこと。即レベル2に進む。
```

> **背景（2026-03-18）**: レベル1失敗後、8つの変種（Profile 1, about:blank, --disable-extensions, AppleScript JS, AppleScript UI, iframe操作等）を場当たり的に試行して40分浪費。レベル2（CDP直接接続）に行けば5分で解決していた。

### ユーザーエスカレーションテンプレート

自動化アプローチが失敗した場合、**正確な手順**を提示:

```
ブラウザ自動化ではこのステップを完了できませんでした。手動で実行してください:

URL: https://...
手順:
1. 上記URLを開く
2. [ボタン名] をクリック
3. [フィールド名] に「値」を入力
4. [送信ボタン] をクリック

期待される結果: [完了時に表示されるべきもの]
完了したら教えてください。残りのステップは自動で続行します。
```

## Phase 4: 操作パターン集

### パターンA: フォーム入力

```bash
{tool} open "https://target"
{tool} wait {default_wait}
{tool} snapshot
# 要素を特定してから:
{tool} fill {element_ref} "value"
{tool} click {button_ref}
{tool} wait {default_wait}
{tool} snapshot    # 結果を確認
```

### パターンB: トークン/シークレット抽出

**抽出順序: DOM要素 → innerText正規表現**（逆にしない）

DOM要素は構造化されており確実。innerText正規表現はURLの途切れや誤マッチが起きやすい。

```javascript
// Step 1: DOM要素から探す（優先）
// input要素 → code要素 → data属性 → テーブルセル の順で探索
page.evaluate(`
  (() => {
    // input要素（value属性）
    for (const inp of document.querySelectorAll('input[type=text], input[readonly]')) {
      if (inp.value && inp.value.includes('target-pattern')) return inp.value;
    }
    // code/pre要素
    for (const c of document.querySelectorAll('code, pre, [data-token], [data-url]')) {
      const t = c.textContent.trim();
      if (t.includes('target-pattern')) return t;
    }
    // aタグのhref
    for (const a of document.querySelectorAll('a[href*="target-pattern"]')) return a.href;
    // テーブルセル・span・div内の長いテキスト
    for (const el of document.querySelectorAll('td, span, div')) {
      const t = el.textContent.trim();
      if (t.startsWith('https://') && t.includes('target-pattern') && t.length > 50) return t;
    }
    return null;
  })()
`)

// Step 2: DOM要素で見つからない場合のみ、innerText正規表現
// 注意: 文字クラスに大小英字+数字+記号を含める（[A-Z0-9/]だけでは途切れる）
page.evaluate(`document.body.innerText.match(/https:\\/\\/target-pattern\\/[A-Za-z0-9\\/_-]+/)?.[0]`)
```

> 背景（2026-03-20）: Slack Webhook URL抽出で `[A-Z0-9/]+` の正規表現がトークン末尾（小文字英字含む）をキャプチャできず途切れた。DOM要素（input[readonly]）からは完全なURLが取得できた。

### パターンC: OAuth認可フロー

```bash
# 1. Phase 0の全スコープを含むAuth URLを構築
AUTH_URL="https://service.com/oauth/authorize?client_id=...&scope={all_scopes}&redirect_uri=..."

# 2. 開いて待つ
{tool} open "$AUTH_URL"
{tool} wait {default_wait}

# 3. 承認ボタンを見つけてクリック
{tool} snapshot
{tool} click {approve_button_ref}
{tool} wait {default_wait}

# 4. リダイレクトURLからコード/トークンを抽出
{tool} eval "window.location.href"
```

### パターンD: Cookie/セッション転送

```bash
# browser-useでCookieをエクスポート（最強のCookie管理）
browser-use -b real --profile "{browser_profile}" open "https://target"
browser-use cookies get --url "https://target" > /tmp/cookies.json
browser-use close

# primary_toolで認証済みセッションを使って続行
```

## セキュリティ（前倒し）

- **トークンやシークレットをログに出さない** — コンソール出力やコミットメッセージに含めない
- **URL検証** — 開く前にタイポスクワッティング、フィッシングドメインをチェック
- **SKILL.mdにクレデンシャルを保存しない** — 環境変数やシークレットマネージャーを使用
- **ブラウザセッションを閉じる** — 完了後に認証済みセッションを放置しない
- **最小権限の原則** — 実際に必要な権限のみをリクエスト

## 失敗パターンDB

既知の失敗パターン。自己改善で新しいエントリが自動追加される。

### FP-01: プロセス残骸（ツール固有）

```
症状: ツールコマンドがハングまたは接続エラー
原因: 前回セッションのソケット/PIDファイルが残存
対処: プロセスをkill → ソケットファイル削除 → リトライ
```

### FP-02: 間違ったブラウザプロファイル

```
症状: 「ログイン済み」なのにログイン画面が表示される
原因: 間違ったプロファイルを使用。認証済みセッションは特定のプロファイルにある
対処: 常にConfigの browser_profile を指定
```

### FP-03: SPA未ロード（最頻出）

```
症状: snapshotが空、インタラクティブ要素がゼロ
原因: JavaScript SPAのレンダリングが未完了
対処: ネットワークアイドルまで待機。サービス別待機時間:
  - 軽いページ: default_wait (5秒)
  - 重いSPA: heavy_spa_wait (10秒)
  - それでも空なら: evalでDOM要素数をチェック
```

### FP-04: OAuthスコープ構文エラー

```
症状: 「Something went wrong」や「invalid_scope」エラーページ
原因: スコープ区切り文字がサービスごとに異なる
対処: サービスドキュメントで正しい区切り文字を確認:
  - カンマ区切り: Slack (chat:write,channels:join)
  - スペース区切り: Google (%20エンコード)
  - プラス区切り: 一部のレガシーOAuth
```

### FP-05: 古い要素参照

```
症状: 「Element not found」または間違った要素をクリック
原因: ページが変わったのに古い要素参照を使用
対処: ページ遷移や状態変更の後は必ず再snapshot
```

### FP-06: 再インストールなしの権限変更

```
症状: ダッシュボードでスコープ追加後もAPIが「missing_scope」を返す
原因: 一部サービス（Slack, GitHub Apps）は新スコープ有効化に再インストールが必要
対処: スコープ変更後 → 再インストール/再認可 → 新トークン取得
```

### FP-07: ナビゲーション妨害拡張（Workona等）

```
症状: browser-use -b real でページを開いても、全てWorkona/OneTab等のタブマネージャーにリダイレクトされる
原因: Workona等のタブ管理拡張がwindow.location, window.open, iframe.srcを全てフック
対処:
  1. Phase 0bのプリフライトで検知 → browser-use realを使わない
  2. CDP直接接続（Phase 1B）に即切替
  3. --disable-extensions付きでChrome起動 → プロファイルコピーで認証維持
背景: 2026-03-18。browser-useで15分浪費。最初からCDP直接接続なら3分で完了。
```

### FP-08: browser-useセッションサーバー障害

```
症状: 「Error: Failed to start session server」が全コマンドで発生
原因: 前回のクラッシュ後にプロセス残骸/ソケットが残る。uv reinstallでも解消しないことがある
対処:
  1. ツール修復に入らない（5分ルール違反の最大原因）
  2. CDP直接接続（Phase 1B）に即切替
  3. どうしても直すなら: pkill → uv tool uninstall → uv tool install browser-use --with playwright
背景: 2026-03-18。修復に15分浪費しても解消せず。CDP直接接続で即解決。
原則: 「ツールを直す」より「別の手段で目的を達成する」方が常に速い。
```

### FP-09: Chromeのnon-default data directory制約

```
症状: Chrome --remote-debugging-port=9222 で「non-default data directory」エラー
原因: Chromeは既存プロファイルのdata directoryではremote debuggingを拒否する
対処: プロファイルをコピーして別ディレクトリから起動（Phase 1Bの手順参照）
  cp -r "$HOME/Library/Application Support/Google/Chrome/Default" /tmp/chrome-cdp-session/Default
背景: 2026-03-18。シンボリックリンクでの回避は不可（Chromeが実パスに解決する）。
```

### FP-10: Slack OAuth再インストール時のチャンネル選択

```
症状: OAuth認可画面で「許可する」を押しても遷移しない
原因: Incoming Webhookスコープがある場合、Webhook用チャンネルの事前選択が必須
対処: チャンネル選択ドロップダウン（React select）に値を入力してから「許可する」をクリック
  - Playwrightの場合: page.locator('input.c-select_input').fill('channel-name')
背景: 2026-03-18。チャンネル未選択で「許可する」が効かず、原因特定に時間がかかった。
```

### FP-11: URL/トークン抽出の正規表現が不完全

```
症状: 抽出したURLやトークンが途切れている（末尾が欠落）
原因: 正規表現の文字クラスが狭い（例: [A-Z0-9/]で小文字英字を含まない）
対処:
  1. DOM要素探索を先にやる（input.value, code.textContent等）→ 確実に完全な値が取れる
  2. 正規表現フォールバックでは [A-Za-z0-9/_-]+ を使う
  3. 抽出後にURL長を確認（Webhook URLは通常80文字以上）
背景: 2026-03-20。Slack Webhook URL（末尾が小文字英数字）が [A-Z0-9/]+ で途切れた。
```

### FP-12: gh CLIのリポジトリコンテキスト誤り

```
症状: gh secret set / gh pr create 等が意図しないリポジトリに対して実行される
原因: cdでサブモジュールディレクトリに移動した後、ghが検出するリポが変わる
対処:
  1. gh コマンドには常に -R owner/repo フラグを明示する
  2. 実行前に `gh repo view --json nameWithOwner -q .nameWithOwner` で確認
  3. サブモジュール内でghを実行しない（親ディレクトリに戻る）
背景: 2026-03-20。gh secret setが${SHARED_REPO}リポに設定され、ai-companyリポに設定されなかった。
```

## サービス別プレイブック

サービス固有のガイドは `playbooks/` に個別ファイルとして格納。
対象サービスでの作業前に該当プレイブックを読み込む。

```bash
# 利用可能なプレイブックを確認
ls {playbook_dir}

# サービスでの作業前に読み込む
cat {playbook_dir}/slack.md
```

各プレイブックの内容:
- 必要なスコープ/権限の一覧表
- 最短セットアップ手順
- サービス固有の注意点
- 推奨ツール（CLI優先 or ブラウザ）

[playbooks/](playbooks/) ディレクトリを参照。

## 自己改善（失敗→成功後に自動進化）

**トリガー**: ブラウザ操作で1回以上のリトライ/ツール切替/エラーハンドリングを経て、最終的に成功した場合。

### 改善フロー（成功直後に即実行 — スキップ禁止）

```
1. 簡易分析（5秒）
   ├─ 何が失敗した？（エラーメッセージ / 空結果 / 間違った操作）
   ├─ 何で解決した？（長い待機 / ツール切替 / 別セレクター / eval）
   └─ 再発する？（サービス固有？ それとも共通の罠？）

2. このSKILL.mdを更新（該当するもののみ）
   a. 新しい失敗パターン → 失敗パターンDBにFP-XXを追加
   b. サービス固有の発見 → playbooks/のプレイブックファイルを更新
   c. 新サービスの成功 → テンプレートから新プレイブックを作成
   d. ツール選択の変更 → Phase 1の判定ツリーを更新

3. コミット
   メッセージ: "improve(auto-browser): {サービス} — {学んだこと}"
```

### 改善しない場合（ノイズ防止）
- 初回で成功 → スキルは正しく動作中。更新不要
- ユーザーが手動で解決 → インシデントレポートとして記録。手動手順を理解するまでスキルを更新しない
- 原因不明、偶然成功 → 再現不可能な修正は記録しない

### 改善の品質基準
- **再現可能**: 同じ状況 + 同じ修正 = 同じ結果
- **具体的**: 「待機を5000から10000に変更」であって「もっと慎重に」ではない
- **検証済み**: 実際に効果があった手順のみ記録。理論的修正は不可

## セッション管理

```bash
# agent-browser: 自動セッション管理（開いたブラウザは close まで維持）
agent-browser open "https://target"
# ... 操作 ...
agent-browser close

# browser-use: 名前付きセッション
browser-use --session {service}_{purpose} open "https://target"
# ... 操作 ...
browser-use --session {name} close
```

## BLOCKINGゲート

| Step | 失敗条件 | 動作 |
|------|---------|------|
| Phase 0 | ゴールが不明確（必要なAPI/スコープが特定できない） | STOP + ownerに確認 |
| Phase 1 | primary_tool / fallback_tool の両方が未インストール | STOP + インストール手順を提示 |
| Phase 2 | snapshot が3回連続空 | → Phase 3（3ストライクルール）へ |
| Phase 3 | 全ツール3回失敗 | STOP + ownerに手動操作を依頼 |

## エスカレーション

| 状況 | 対応 |
|------|------|
| OAuth認可で管理者承認が必要 | ownerに確認: 「{サービス}の{スコープ}に管理者承認が必要です」 |
| 3ストライク到達 | ownerに手動操作を依頼（Phase 3テンプレート使用） |
| トークン期限切れでブラウザ再認証必要 | ownerに確認: 「{サービス}のトークンが期限切れです。再認証が必要です」 |
| 未知のサービスで操作方法不明 | ownerに確認: 「{サービス}の操作方法を調査中。手動で操作しますか？」 |

## 合成可能性

| 連携スキル | 関係 | トリガー |
|-----------|------|---------|
| slack-pr-notify | 後工程 | Slack App設定後にPR通知を有効化 |
| youtube-analytics | 前工程 | YouTube APIトークン取得がyoutube-analytics実行の前提 |
| cost-monitor | 前工程 | GCPサービスアカウント設定がcost-monitor実行の前提 |
| security-scan | 並列 | 新サービス連携時にセキュリティ監査を推奨 |

## やらないこと

- ブラウザツールのコマンドリファレンス（agent-browser / browser-use スキルの責務）
- サービスのアカウント新規作成（ownerが手動で実施）
- 課金設定・支払い情報の変更（ownerの責務）
- 本番環境の認証情報の直接操作（staging/dev のみ自動化）

## 新プレイブックテンプレート

新しいサービスのプレイブック作成時:

```markdown
# {サービス名} プレイブック

## スコープ/権限一覧

| ユースケース | API/操作 | 必要な権限 |
|------------|---------|-----------|
| ... | ... | ... |

## 最短手順

1. (ステップ)

## 注意点

- (サービス固有の罠)

## 推奨ツール

- CLI優先: `{cliツール}` でほとんどの操作に対応
- ブラウザ: {特定のシナリオ} の場合のみ
```

## 他のスキルとの連携

| スキル | 関係 | 説明 |
|--------|------|------|
| slack-pr-notify | 後工程 | Slack App設定後にPR通知を有効化 |
| youtube-analytics | 前工程 | YouTube APIトークン取得がyoutube-analytics実行の前提 |
| cost-monitor | 前工程 | GCPサービスアカウント設定がcost-monitor実行の前提 |
| security-scan | 並列 | 新サービス連携時にセキュリティ監査を推奨 |
| leak-learner | 学習 | owner指摘をlessons/に記録。2+スキル共通パターンはGlobal rulesに昇格 |

> 合成可能性セクションの内容を連携テーブルに統合し、leak-learner行を追加。

## 汎用性（Portability）

このスキルは **汎用フレームワーク** — ブラウザ自動化の設計パターン（ゴール設計・ツール選択・5分ルール・3ストライク・失敗パターンDB）はサービス非依存。
他社は以下を変更するだけで使える:
1. Config の `primary_tool` / `fallback_tool` を自社環境に合わせる
2. `playbooks/` に自社サービスのプレイブックを追加
3. Config の `browser_profile` を自社のChrome profileに変更

失敗パターンDB（FP-01〜FP-12）と操作パターン（A〜D）は汎用。サービス固有の知識はplaybooks/に分離済み。

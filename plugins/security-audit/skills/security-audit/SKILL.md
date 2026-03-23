---
name: security-audit
description: プロジェクトのセキュリティ監査を実行するスキル。パターンスキャン + 依存関係チェック + AI攻撃者分析の3段階。「セキュリティ」「脆弱性」「監査」「security」等のトリガー、または /ship 実行前に自動起動。
---

# Security Audit Skill

プロジェクト全体に対して3段階のセキュリティ監査を実行する。

## トリガー

### 明示的トリガー
- 「セキュリティチェックして」「脆弱性ある？」「監査して」
- 「security audit」「pentest」
- `/security-audit`（旧command互換）

### 自動検知トリガー
- `/ship` 実行時（Step 2で自動実行される）
- コードレビュー時にセキュリティ懸念を検知した場合
- 外部API連携・認証フロー・シークレット管理のコードを書いた直後

## 手順

### Step 1: 対象プロジェクトの特定

ユーザーに確認:
- 「どのプロジェクト/ディレクトリを監査しますか？」
- 指定がなければ現在のワーキングディレクトリを対象にする

### Step 2: 自動スキャン（並列実行）

以下を**並列**で実行:

#### 2a: シークレットパターンスキャン

対象ディレクトリ全体で以下のパターンを検出:

| パターン | 対象 |
|---------|------|
| `AIzaSy[a-zA-Z0-9_-]{30,}` | Google/Gemini API Key |
| `xoxb-[0-9-]*` | Slack Bot Token |
| `ntn_[a-zA-Z0-9]{30,}` | Notion API Token |
| `BEGIN PRIVATE KEY` | Private Key |
| `ghp_[a-zA-Z0-9]{30,}` | GitHub PAT |
| `sk-[a-zA-Z0-9]{20,}` | OpenAI/Stripe Secret Key |
| `AKIA[A-Z0-9]{16}` | AWS Access Key |
| ハードコードされたパスワード・フォールバック値 | 全般 |

#### 2b: 依存関係チェック

テックスタックを自動検出して実行:

| 検出ファイル | 実行コマンド |
|------------|------------|
| `package.json` | `npm audit --audit-level=moderate` |
| `requirements.txt` / `pyproject.toml` | `pip audit` or `safety check` |
| `go.mod` | `govulncheck ./...` |
| `Cargo.toml` | `cargo audit` |

（ツール未インストールの場合はスキップして報告）

#### 2c: Semgrep（インストール済みの場合のみ）

```bash
semgrep --config auto --severity ERROR --severity WARNING <対象> --json
```

### Step 3: AI攻撃者分析（3並列エージェント）

**3つのAgentを並列で起動する。** 各エージェントは攻撃者として徹底的にコードを攻撃する。

#### Agent A: フロントエンド / CLI / スクリプト攻撃

チェック項目:
1. コマンドインジェクション / `subprocess.run(shell=True)` / `os.system()`
2. SSRF: ユーザー入力URLへの fetch/requests にドメイン検証があるか
3. XSS: innerHTML に外部データが escapeHtml なしで入っていないか
4. プロンプトインジェクション: ユーザー制御テキストがAIプロンプトに直接注入されないか
5. パストラバーサル: ファイルパスにユーザー入力が使われていないか（**`startswith()` バイパスも確認**）
6. eval() / exec() の危険な使用
7. DOMベース攻撃 / page.evaluate内の攻撃者制御コンテンツ
8. タイムアウト不在: 外部リソースアクセスにタイムアウトが設定されているか
9. CLI引数でのシークレット露出: `--token`, `--password` 等が `ps aux` / シェル履歴に残らないか
10. ファイル名インジェクション: multipart/HTTPヘッダにユーザー制御のファイル名が入っていないか（CRLF）

#### Agent B: バックエンド / サービス層攻撃

チェック項目:
1. 認証バイパス: req.body から userId を取得していないか
2. Fail-open: 環境変数/設定未定時にセキュリティが無効化されないか
3. 情報漏洩: error.stack をレスポンスやログに含めていないか
4. 入力サイズ制限: 巨大な入力でDoSが可能か
5. AI応答の信頼: LLM応答のJSONをバリデーションなしに信頼していないか
6. タイムアウト: 外部 fetch に AbortController / timeout がないか
7. ログ: 機密情報（Authorization, APIキー）をログ出力していないか
8. SQLインジェクション / NoSQLインジェクション

#### Agent C: 信頼境界 & 認証攻撃

チェック項目:
1. Secret Manager フォールバック: 本番で環境変数フォールバック時にセキュリティ低下しないか
2. 資格情報の平文保存 / メモリ長期保持
3. セッション管理の不備
4. シークレットローテーション後も旧値が使われ続けないか
5. 設定インジェクション: Config に悪意ある値を注入できないか
6. `.env` の `.gitignore` 漏れ
7. pickle / yaml.load の unsafe なデシリアライズ

各エージェントの出力形式:
```
- ファイル: パス:行番号
- 重要度: CRITICAL / HIGH / MEDIUM / LOW
- 攻撃シナリオ: 具体的な攻撃手順（3ステップ以内）
- 修正案: コード例
```

### Step 3.5: 修正実施時の必須チェック（即時修正する場合）

監査結果を受けて即時修正する場合、以下を**必ず**実行する。

#### 3.5a: 全ファイル網羅チェック（修正前に実施）

1. `Glob **/*.py` / `Glob **/*.ts` 等で対象ディレクトリの全スクリプトを列挙
2. 脆弱性が見つかったコードパターン（関数名、ライブラリ呼び出し）で `Grep` して**全ファイルを洗い出す**
3. **重複コード**（同じ関数が複数ファイルにコピペされている）を検出し、全てを修正対象に含める
4. チェックリスト: 「この脆弱性を持つファイルは他にないか？」を毎件自問する

#### 3.5b: 修正セルフレビュー（修正後に実施）

各修正が既知のバイパスパターンを考慮しているか確認:

| 修正カテゴリ | よくあるバイパス | 正しいパターン |
|-------------|----------------|---------------|
| パストラバーサル防止 | `startswith("/foo/bar")` → `/foo/bar_evil/` がマッチ | `(path + os.sep).startswith(base + os.sep)` |
| ファイル名サニタイズ | 英数字以外を除去するだけ → null byte, CRLF | `re.sub(r'[^a-zA-Z0-9._-]', '_', name)` |
| サイズチェック | `open()` 後にチェック → メモリ枯渇 | `os.path.getsize()` → サイズOKなら `open()` |
| 認証情報の渡し方 | CLI引数 → `ps aux` で露出 | 環境変数 or ファイル読み込み |
| タイムアウト | `urlopen()` タイムアウトなし → ハング | `urlopen(req, timeout=30)` |
| エラーメッセージ | トークンが `str(e)` に含まれる | `err.replace(token, "***")` でマスク |
| Content-Type | `application/octet-stream` フォールバック → fail-open | 明示的な `image/png` 等 |
| レート制限 | 固定wait → 429連続でアカウント制限 | 指数バックオフ（1.2s → max 10s） |

#### 3.5c: .gitignore チェック（修正後に実施）

以下が `.gitignore` に含まれているか確認:
- 認証情報ファイル（`.slack_token`, `.slack_cookie`, `.env`）
- ブラウザプロファイル（`.playwright_profile/`）
- 生成物ディレクトリ（`output/` 等）
- キャッシュ（`__pycache__/`, `*.pyc`）

### Step 4: レポート生成

結果を統合してレポートを生成:

```
出力先: docs/security/AUDIT_YYYY-MM-DD.md
```

レポート構成:
1. Executive Summary（重要度別件数）
2. 自動スキャン結果
3. AI攻撃者分析（Agent A/B/C）
4. 推奨対応（即時修正 / 計画修正 / 許容）
5. 前回監査からの改善（前回レポートがあれば差分）

### Step 5: 完了報告

```
セキュリティ監査完了:
- CRITICAL: X件
- HIGH: X件
- MEDIUM: X件
- LOW: X件

レポート: docs/security/AUDIT_YYYY-MM-DD.md
即時対応が必要: X件
```

## 注意

- CRITICAL/HIGH は即時修正を強く推奨
- `/ship` から呼ばれた場合、CRITICAL があればマージを拒否する
- `--pentest` オプション指定時は実環境テストコマンドも生成（実行前に必ずユーザー確認）

## 設計原則

| 原則 | 出典 | 適用 |
|------|------|------|
| 攻撃者視点 | Red Team 手法 | 防御者目線でなく攻撃者として3並列Agentがコードを攻撃する |
| 多層防御 | TEKsystems 3-Layer Security | 自動スキャン(2a-2c) + AI分析(3) + セルフレビュー(3.5)の3層で漏れを防ぐ |
| 全ファイル網羅 | security.md | 修正時は対象ディレクトリの全スクリプトをGlobで列挙し、パターンGrepで見落としゼロを目指す |

## Config

| カテゴリ | キー | デフォルト値 | 説明 |
|---------|------|------------|------|
| パス | REPORT_DIR | `docs/security/` | 監査レポート出力先 |
| パス | REPORT_NAME | `AUDIT_YYYY-MM-DD.md` | レポートファイル名テンプレート |
| 閾値 | BLOCK_SEVERITY | `CRITICAL` | /ship実行時にマージを拒否する最低重要度 |
| ツール | SEMGREP_CONFIG | `auto` | Semgrepのルールセット |
| ツール | NPM_AUDIT_LEVEL | `moderate` | npm audit の最低報告レベル |

## セキュリティ

| 項目 | ルール |
|------|--------|
| 検出シークレット | レポートにはパターン名+ファイル:行番号のみ記載。値そのものをレポートに含めない |
| --pentest結果 | 実環境テスト結果は暗号化保存を推奨。平文でgit commitしない |
| 監査対象のAPIキー | 検出したキーの無効化・ローテーションをレポートの推奨対応に含める |

## BLOCKINGゲート

| Step | 失敗条件 | 動作 |
|------|---------|------|
| Step 1 | 対象ディレクトリが存在しない | STOP + 「指定されたパスが存在しません」 |
| Step 2a | Grepパターンの実行エラー | STOP + エラー内容を表示。パターン修正後に再実行 |
| Step 2b | 依存関係チェックツールが未インストール | スキップして報告（STOPしない）。レポートに「未実行」と明記 |
| Step 3 | 3並列Agentのいずれかがタイムアウト | 完了分のみでレポート生成 + 「Agent X 未完了」を明記 |
| Step 4 | /shipから呼ばれてCRITICALが1件以上 | STOP + マージ拒否 + 即時修正を要求 |

## エスカレーション

| 状況 | 対応 |
|------|------|
| CRITICALが3件以上 | ownerに確認: 「CRITICAL脆弱性が{N}件検出。即時修正の優先順位を決めてください」 |
| 本番環境にシークレット露出の疑い | ownerに確認: 「本番のキーローテーションが必要な可能性があります。即時対応しますか？」 |
| --pentestで実環境テスト実行前 | ownerに確認: 「以下のテストコマンドを実行してよいですか？[コマンド一覧]」 |
| 修正によりサービス停止の可能性 | ownerに確認: 「この修正はダウンタイムを伴います。メンテナンス時間を設定しますか？」 |

## 合成可能性

| 連携スキル | 関係 | トリガー |
|-----------|------|---------|
| ship | 前工程 | /ship のStep 2でsecurity-auditが自動起動される |
| client-monthly-report | 後工程 | インシデント検出結果がクライアント月次レポートに反映される |
| incident-triage-lite | 後工程 | CRITICAL/HIGH検出時にインシデントログを自動作成 |

## やらないこと

- 本番環境への直接アクセス（--pentestコマンド生成のみ。実行はowner承認後）
- シークレットの自動ローテーション（検出と推奨のみ。実行はownerが判断）
- サードパーティSaaS（Snyk, Dependabot等）の設定変更
- コンプライアンス認証（SOC2, ISO27001等）の適合判定（技術的脆弱性の検出のみ）

## 他のスキルとの連携

| スキル | 関係 | 説明 |
|--------|------|------|
| ship | 前工程 | /ship のStep 2でsecurity-auditが自動起動される |
| client-monthly-report | 後工程 | インシデント検出結果がクライアント月次レポートに反映される |
| incident-triage-lite | 後工程 | CRITICAL/HIGH検出時にインシデントログを自動作成 |
| rapid-build | 呼び出し元 | rapid-buildのStep 6で包括的スキャンのために呼び出される |
| leak-learner | 学習 | owner指摘をlessons/に記録。2+スキル共通パターンはGlobal rulesに昇格 |

> 合成可能性セクションの内容を連携テーブルに統合し、leak-learner行を追加。

## 汎用性（Portability）

このスキルは **汎用フレームワーク** — シークレットパターンスキャン・依存関係チェック・AI攻撃者3並列分析のワークフローはプロジェクト非依存。
他社は Config セクションの `REPORT_DIR`, `BLOCK_SEVERITY`, `SEMGREP_CONFIG` を自社環境に変更するだけで使える。

攻撃パターン（Step 3のAgent A/B/C）、修正セルフレビュー（Step 3.5）、バイパスパターン表は全て汎用。
サービス固有のシークレットパターン（Step 2a）は追加可能な構造。企業名ハードコードなし。

## Zero Leak L5 接続

> lessons/ ディレクトリが存在する。leak-learnerがowner指摘を自動蓄積する書き込み先。

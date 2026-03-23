# Skill Naming Guide

スキル名の命名・リネーム時に参照する。neo-skill-creator Step 1.5 で自動適用。

## 必須ルール

1. **kebab-case**（小文字+ハイフン）。例外なし
2. **2-3語**。1語は汎用すぎ（`start`, `ship`, `verify` → NG）。4語以上は長すぎ
3. **動詞-名詞をデフォルト**にする（`review-code` 形式）
4. **外部サービス統合はサービス名プレフィックス**（`slack-*`, `youtube-*`）
5. **名前だけ見て10秒で機能を推測できる**こと

## 命名フロー

```
Step 1: 機能を1文で書く（「テストを自動実行する」）
Step 2: 動詞-名詞で候補を3つ出す（test-runner, test-runner-run, run-tests）
Step 3: 衝突チェック
  - npm search {name}
  - GitHub Actions検索
  - anthropics/claude-plugins-official で同名なし
  - Trail of Bits/VoltAgent等の主要スキルコレクション
Step 4: 10秒テスト（チームメンバーに名前だけ見せて機能を推測してもらう）
Step 5: 決定
```

## リネーム時の追加手順

```
Step 1: 旧名の全参照を検索
  grep -r "{旧名}" --include="*.md" --include="*.yaml" --include="*.json" .
Step 2: 全参照を新名に置換
Step 3: ディレクトリ名をリネーム
  mv .claude/skills/{旧名} .claude/skills/{新名}
Step 4: Plugin Marketplaceにも配布している場合:
  - marketplace.jsonの名前とsourceを更新
  - README.mdを更新
Step 5: スキルレジストリを更新
```

## 動詞 vs 名詞 の選択基準

| パターン | いつ使う | 例 |
|---------|---------|-----|
| 動詞-名詞 | アクションを実行する（推奨デフォルト） | `review-code`, `scan-security` |
| 名詞のみ | 外部サービス統合、出力物 | `slack`, `svg-diagram` |
| 役割メタファー | オーケストレーター的スキル | `presentation-architect` |
| 造語 | 独自概念。衝突リスクゼロ | `pluginize`, `leak-learner` |

## アンチパターン

| NG | 理由 | 代わりに |
|----|------|---------|
| 1語の汎用名 | 衝突+機能不明 | 2語以上の具体名にする |
| `-manager`/`-helper` | 何するか不明 | 具体的な動詞を使う |
| 4語以上 | タイプミス+覚えにくい | 最も重要な2語に絞る |
| 略語（独自） | 発見性低下 | フルワードを使う |
| カテゴリプレフィックス乱用 | 名前空間でやるべき | タグ/カテゴリで管理 |

## 良い名前の例

| 名前 | 良い理由 |
|------|---------|
| `presentation-architect` | 役割メタファーが具体的 |
| `svg-diagram` | 出力物が明確 |
| `incident-triage-lite` | 機能+修飾子でスコープ明確 |
| `leak-learner` | 独自概念で衝突なし |
| `rapid-build` | 動詞-名詞+速さの修飾 |

## 出典

- Claude Code公式: 動詞-名詞のkebab-case推奨
- npm: 小文字+ハイフン。句読点差異の重複禁止
- Krew (kubectl): 略語禁止。一般的すぎる動詞/名詞を避ける
- GitHub Actions: 具体的で記述的に。`build`単体はNG

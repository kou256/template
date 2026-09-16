# template

GitHub のテンプレートリポジトリ。新規リポジトリの初期セットと、共通 CI の reusable workflow を提供します。

## 含まれるもの

| ファイル | 役割 |
|---|---|
| `.github/workflows/mise-ci.yml` | 共通 CI の実体（reusable workflow）。他リポジトリから `@v1` で呼ばれる |
| `.github/workflows/ci.yml` | 呼び出し側の雛形。コピー先でそのまま動く |
| `mise.toml` | ツールとタスクの雛形。`lint` / `fmt` / `fmt:check` / `test` の4タスク |
| `renovate.json` | 共有プリセット（`kou256/.github`）への参照 |
| `AGENTS.md` / `CLAUDE.md` | エージェント向けの規約。規約本文は持たずスキルに委譲する |
| `.claude/skills/` | GitHub 運用のスキル4種 |
| `.gitignore` / `.editorconfig` | 言語非依存の共通設定 |

PR テンプレートと Issue テンプレートはこのリポジトリには**ありません**。`kou256/.github` に置いてあり、全リポジトリへ自動で適用されます。

## 新規リポジトリのセットアップ

```bash
# 1. テンプレートから作成
gh repo create kou256/<name> --template kou256/template --private --clone

# 2. ラベルを投入（テンプレートはラベルをコピーしないため必須）
gh label clone kou256/template --repo kou256/<name> --force

# 3. mise.toml を埋める
#    [tools] に Go / Node / pnpm などのバージョンを固定し、
#    [tasks] の run を実際のコマンドに置き換える

# 4. Renovate を有効化（GitHub App をこのリポジトリに追加）
```

## CI の前提

`mise-ci.yml` は呼ぶ側に次を要求します。満たさないと CI が落ちます。

- `mise.toml` があること
- `lint` / `fmt:check` / `test` の3タスクが定義されていること

self-hosted ランナーを使う場合は `runner` を JSON 文字列で渡します。

```yaml
jobs:
  ci:
    uses: kou256/template/.github/workflows/mise-ci.yml@v1
    with:
      runner: '["self-hosted","Linux","ARM64"]'
```

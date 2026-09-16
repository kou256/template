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
| `.agents/skills/` | GitHub 運用のスキル4種の実体 |
| `.claude/skills/` | 上記への symlink（Claude Code が読む場所） |
| `skills-lock.json` | スキルのバージョン固定。`npx skills` が管理する |
| `.gitignore` / `.editorconfig` | 言語非依存の共通設定 |

PR テンプレートと Issue テンプレートはこのリポジトリには**ありません**。`kou256/.github` に置いてあり、全リポジトリへ自動で適用されます。

## 新規リポジトリのセットアップ

```bash
# 1. テンプレートから作成
gh repo create kou256/<name> --template kou256/template --private --clone

# 2. ラベルを投入（テンプレートはラベルをコピーしないため必須）
gh label clone kou256/template --repo kou256/<name> --force
# clone はラベルを削除しないため、GitHub 既定の enhancement が feature と併存する。消しておく
gh label delete enhancement --repo kou256/<name> --yes

# 3. mise.toml を埋める
#    [tools] に Go / Node / pnpm などのバージョンを固定し、
#    [tasks] の run を実際のコマンドに置き換える

# 4. Renovate を有効化（GitHub App をこのリポジトリに追加）
```

## スキルの管理

スキルは [`skills`](https://github.com/vercel-labs/skills) CLI で管理します。実体は `.agents/skills/` にあり、`.claude/skills/` はそこへの symlink です。

```bash
# kou256/skills の更新に追従する
npx skills update

# skills-lock.json から復元する（clone 直後など）
npx skills experimental_install

# スキルを追加する
npx skills add kou256/skills -a claude-code -a universal -s <skill-name>
```

## CI の前提

`mise-ci.yml` は呼ぶ側に次を要求します。満たさないと CI が落ちます。

- `mise.toml` があること
- `lint` / `fmt:check` / `test` の3タスクが定義されていること
- `fmt:check` は**書き込みをしない**こと（`golangci-lint fmt --diff` のように差分の検査だけを行う）

`lint` ジョブが `fmt:check` → `lint` を、`test` ジョブが `test` を実行します。2ジョブは並列に走るので、片方が落ちてももう片方の結果を確認できます。

self-hosted ランナーを使う場合は `runner` を JSON 文字列で渡します。

```yaml
jobs:
  ci:
    uses: kou256/template/.github/workflows/mise-ci.yml@v1
    with:
      runner: '["self-hosted","Linux","ARM64"]'
```

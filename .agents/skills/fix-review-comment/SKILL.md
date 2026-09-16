---
name: fix-review-comment
description: Review and fix unresolved PR comments, reply to each addressed comment inline, then stage, commit (via commit-staged-changes), and push. Use when addressing code review feedback on GitHub PRs end-to-end.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Skill
argument-hint: <pr-url-or-number>
---

# Fix Review Comment Workflow

GitHubのPull Requestにつけられた未解決のレビューコメントを確認し、対応を実施するワークフロー。

## When to Use

- PRのレビューコメントに対応したい場合
- `/fix-review-comment <pr-url-or-number>` で呼び出し

## Prerequisites

- gh CLI がインストール・認証済みであること（`gh auth status`で確認）

## Workflow Steps

### Step 1: 入力の取得と検証

1. ユーザーからPRのURLまたはPR番号を受け取る
2. 入力がない場合、ユーザーに入力を求める
3. URLの場合はPR番号を抽出する

### Step 2: 事前確認

1. `gh auth status`でgh CLIの認証状態を確認
   - 認証されていない場合はエラーメッセージを表示して終了
2. `gh pr view <number> --json headRefName,state`でPR情報を取得
   - PRが存在しない場合はエラーメッセージを表示して終了
   - PRがクローズ済みの場合は警告を表示

### Step 3: ブランチの確認と切り替え

1. `git branch --show-current`で現在のブランチを確認
2. PRのブランチと異なる場合:
   - PRのブランチに自動チェックアウト（`git checkout <branch>`）
   - チェックアウト失敗時はエラーを表示

### Step 4: PR情報とレビューコメントの取得

1. `gh pr view <number> --json title,body,baseRefName,headRefName`でPR詳細を取得
2. `gh api repos/{owner}/{repo}/pulls/{pr_number}/comments`でレビューコメントを取得
3. `gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews`でレビュー情報を取得

### Step 5: レビューコメントの分析

1. PRの差分（`gh pr diff <number>`）とコメント内容を比較
2. 各コメントについて以下を判断:
   - 対応が必要か、不要か
   - 対応が必要な場合の修正方針

**対応が必要と判断する基準:**
- コードの修正を求めるコメント
- バグや問題点の指摘
- セキュリティ上の懸念
- パフォーマンス改善の提案

**対応不要と判断する基準:**
- 質問形式のコメント（回答のみで済む場合）
- 賞賛・承認コメント
- 今回のPRスコープ外の改善提案
- すでに対応済みのコメント

### Step 6: 分析結果の提示

以下の形式でユーザーに提示:

```
## レビューコメント分析結果

### 対応が必要なコメント

| # | ファイル | 行 | コメント内容 | 対応方針 |
|---|---------|-----|-------------|---------|
| 1 | src/xxx.ts | 42 | ～～～ | ～～～ |

### 対応不要なコメント

| # | ファイル | 行 | コメント内容 | 理由 |
|---|---------|-----|-------------|------|
| 1 | src/yyy.ts | 10 | ～～～ | ～～～ |
```

### Step 7: ユーザー確認

ユーザーにどのコメントについて対応するか確認:
- 番号指定（例: `1, 3, 5`）
- 全件対応（`all`）
- スキップ（`skip` または `none`）

### Step 8: 修正の実施

1. 対応すべきコメントが確定したら、順番に修正を実施
2. 各修正について:
   - 対象ファイルを読み込み
   - 修正を適用
   - プロジェクトの検証コマンド（`package.json`の`scripts`にあるcheck/lint/typecheck/testや、`CLAUDE.md`に記載されたコマンド）を発見して実行する
     - 該当するコマンドが見つからない場合は、ユーザーに確認するか、その旨を記録してスキップする
   - 可能であれば、修正箇所ごとに簡易的な動作確認（小さなスクリプト実行など）も行う
   - 検証に失敗した場合は原因を修正し、再検証してから次に進む
   - 修正内容を記録

### Step 9: 修正完了通知

修正完了後、ユーザーに以下を通知:
- 修正したファイル一覧
- 変更内容のサマリー
- 次のステップの案内

### Step 10: レビューコメントへの個別返信（ユーザー確認後）

1. Step 8で対応した各コメントについて、対応内容を説明する返信文をコメントごとに作成する
2. 返信文の一覧をまとめてユーザーに提示し、投稿してよいか確認する
3. 承認された場合、コメントごとに以下でインライン返信として個別投稿する（`gh pr comment`による1件の要約コメントは使わない）:
   ```
   gh api repos/{owner}/{repo}/pulls/{pr_number}/comments \
     -f body="<返信文>" -F in_reply_to=<comment_id>
   ```
4. 投稿に失敗したコメントがあれば、どれが失敗したかを報告し、他の投稿は続行する

### Step 11: ステージングとコミット

1. Step 8で修正したファイルのみを対象に `git add <path> ...` でステージングする（`git add -A`・`git add .`は使わない）
2. コミットメッセージは自前で作らず、`commit-staged-changes`スキルを呼び出して委譲する（そのスキル自身が提示・承認・コミット実行までを行う）

### Step 12: push（ユーザー確認後）

1. コミット後、リモートへpushしてよいかユーザーに確認する
2. 承認された場合、`git push`を実行する（force pushはしない）
3. push結果（更新されたリビジョン範囲・対象ブランチ）を報告する

## Error Handling

| エラーケース | 対応 |
|-------------|------|
| gh CLIが認証されていない | `gh auth login`の実行を案内して終了 |
| PRが存在しない | エラーメッセージを表示して終了 |
| PRがクローズ済み | 警告を表示し、続行するか確認 |
| 未解決コメントが0件 | その旨を通知して終了 |
| 対象ファイルが存在しない | 該当コメントをスキップし、ユーザーに通知 |
| ブランチ切り替え失敗 | エラーを表示し、手動でのチェックアウトを案内 |
| インライン返信の投稿に失敗 | 失敗したコメントを報告し、他のコメントの投稿は継続する |
| 検証コマンドが見つからない | ユーザーに確認するか、その旨を記録してスキップする |
| 検証に失敗した | 原因を修正し再検証する。解消しない場合はユーザーに報告する |
| ステージ対象がない（差分なし） | その旨を通知して終了する |
| pushがrejectされた（fast-forwardでないなど） | force pushはせず、状況をユーザーに報告して指示を仰ぐ |

## Constraints

- `git add`はStep 8で修正したファイルのみを対象とし、`git add -A`・`git add .`は使わない
- コミットメッセージは自前で作成せず、必ず`commit-staged-changes`スキルに委譲する
- インライン返信の投稿・コミット・pushは、実行前に必ずユーザーの明示的な承認を得る（まとめての承認でよいが、無承認では実行しない）
- force pushは行わない
- ファイル削除は明示的な指示がない限り行わない
- 新規ファイルの作成は最小限に（既存ファイルの編集を優先）

## Examples

### 呼び出し例

```
/fix-review-comment https://github.com/owner/repo/pull/123
/fix-review-comment 123
```

### 実行例

```
> /fix-review-comment 45

gh CLIの認証状態を確認中...
✓ 認証済み (user: @username)

PR #45 の情報を取得中...
✓ PR: "Add user authentication feature"
  Branch: feature/auth → dev

現在のブランチ: dev
→ PRのブランチ (feature/auth) にチェックアウトします...
✓ チェックアウト完了

レビューコメントを取得中...
✓ 3件の未解決コメントを検出

## レビューコメント分析結果

### 対応が必要なコメント

| # | ファイル | 行 | コメント内容 | 対応方針 |
|---|---------|-----|-------------|---------|
| 1 | src/auth.ts | 42 | エラーハンドリングを追加してください | try-catchブロックを追加 |
| 2 | src/auth.ts | 58 | 型定義が不足しています | 明示的な型アノテーションを追加 |

### 対応不要なコメント

| # | ファイル | 行 | コメント内容 | 理由 |
|---|---------|-----|-------------|------|
| 1 | src/index.ts | 10 | LGTM! | 承認コメント |

どのコメントに対応しますか？ (番号/all/skip):

> all

修正を実施中...
✓ src/auth.ts: try-catchブロックを追加（typecheck/lint 通過）
✓ src/auth.ts: 型アノテーションを追加（typecheck/lint 通過）

修正が完了しました。次に各コメントへ返信してよいですか？

> はい

✓ 2件のコメントへインライン返信を投稿しました

修正したファイルをステージングし、commit-staged-changesスキルでコミットメッセージを作成します...
（commit-staged-changesスキルの提示・承認フローに従う）

✓ コミット完了: fix: 🐛 認証処理のエラーハンドリングと型定義を修正

リモートにpushしてよいですか？

> はい

✓ push完了: feature/auth (abc1234..def5678)
```

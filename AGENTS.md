# <プロジェクト名>

<!-- プロジェクト概要を 2-3 行で -->

## 開発

- ツールとタスクは `mise.toml` に定義する。検証は `mise run lint` / `mise run fmt:check` / `mise run test` を使う。
- CI は `kou256/template` の reusable workflow を呼ぶ。CI が実行するのも上記と同じ3コマンド。

## コミット

- コミットは `commit-staged-changes` スキルに委譲する。ステージ済みの差分のみを対象とし、メッセージ案の承認後にコミットする。
- ブランチ名も同スキルの規約に従う。

## PR

- PR 本文は `# 概要` / `# テスト` / `# 補足` / `# 関連 issue` の4節。更新は `pr-overview-updater` スキルに委譲する。
- レビュー指摘への対応は `fix-review-comment` スキルに委譲する。

## 言語

- 返信、コミットメッセージ、PR / Issue の本文は日本語で書く。
- コード内コメントの言語は、このリポジトリの既存の慣習に従う。

# zenn-contents

Zenn の記事管理リポジトリ。GitHub 連携で `published: true` の記事が自動公開される。

## 技術スタック

- zenn-cli 0.5.15 (Node 22)
- textlint: preset-ja-technical-writing + preset-japanese
- CI: GitHub Actions で articles/** ・依存関係・textlint 設定の変更時に textlint 実行

## コマンド

- `npm run new:article` — 新しい記事を作成
- `npm run preview` — プレビュー (localhost:8000)
- `npm run lint` — textlint 実行
- `npm run lint:fix` — textlint 自動修正

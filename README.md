# zenn-contents

[Zenn](https://zenn.dev/) の記事管理リポジトリです。

## セットアップ

```sh
npm install
```

## 使い方

### 記事を新規作成する

```sh
npm run new:article
```

`articles/` 配下にファイルが生成されます。frontmatter の `published: false` を `true` に変更すると Zenn に公開されます。

### プレビューを確認する

```sh
npm run preview
```

ブラウザで `http://localhost:8000` を開くとプレビューを確認できます。

### 文章をLintする

```sh
npm run lint

# 自動修正できるものは修正する
npm run lint:fix
```

## CI

`articles/` 配下のファイルが変更された PR / push で textlint が自動実行されます。

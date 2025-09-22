# tawa-zenn

@tawachanのZenn記事管理リポジトリです。

## 概要

このリポジトリはZennでの記事投稿を管理するためのものです。GitHubとZennを連携し、記事の執筆・管理をGit経由で行います。

## セットアップ

### 必要なツール

- Node.js (推奨: v18以上)
- Yarn v4

### インストール

```bash
yarn install
```

## 使い方

### 新しい記事を作成

```bash
yarn new:article
```

### 新しい本を作成

```bash
yarn new:book
```

### プレビュー

```bash
yarn preview
```

ブラウザで http://localhost:8000 を開いてプレビューを確認できます。

## ディレクトリ構成

```
├── articles/          # 記事ファイル (.md)
├── books/             # 本のファイル
├── package.json       # プロジェクト設定
└── README.md          # このファイル
```

## 記事の公開

記事を公開するには、記事ファイルのFront matterで `published: true` に設定してコミット・プッシュします。

```yaml
---
title: "記事タイトル"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "nextjs"]
published: true  # 公開する場合はtrue
---
```

## 関連リンク

- [Zenn](https://zenn.dev/)
- [Zenn CLI](https://zenn.dev/zenn/articles/zenn-cli-guide)

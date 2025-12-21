# AGENTS.md

このファイルはClaude Codeで使用するAIエージェントの設定を定義します。

## プロジェクト概要

このリポジトリは@tawachanのZenn記事管理用リポジトリです。

### 記事の内容範囲

- **バックエンド技術**: NestJS, Connect RPC, Protobuf, Go, API設計
- **フロントエンド技術**: React, Next.js, Astro, Gatsby, Tailwind CSS, Radix UI, Lexical
- **インフラ/DevOps**: Terraform, Firebase, GCP, GitHub Actions, CI/CD
- **組織・マネジメント**: アジャイル開発（スクラム/カンバン）、タスク管理、エンジニアリングマネージャー
- **技術移行記録**: Gatsby→Astro, Notion→GitHub Projects, Markdown→Textbundle

### 記事統計

- **総記事数**: 14記事
- **公開済み**: 9記事
- **ドラフト**: 5記事
- **パブリケーション掲載**: 3記事（PIVOT Media）

### ディレクトリ構成

```
tawa-zenn/
├── articles/        # 記事ファイル（14記事、Markdown形式）
├── books/          # 本のファイル（現在は空）
├── images/         # 記事用画像（11個のGIF/PNG）
├── .claude/        # Claude Code設定
├── AGENTS.md       # このファイル（Claude Code用ガイドライン）
└── README.md       # プロジェクト説明
```

## 主な機能

- Zenn記事の作成・管理
- GitHub連携による記事の自動同期
- プレビュー機能による記事の事前確認
- 画像管理（imagesディレクトリ）
- メタデータ管理（公開日時、パブリケーション設定など）

## 開発方針

### 記事作成時の指針

1. **記事の品質確保**
   - 技術的な正確性を重視する
   - 読みやすい構成と適切な見出し構造
   - コードサンプルには適切な言語指定

2. **メタデータの管理**
   - 記事タイトルは分かりやすく
   - 適切なトピック（topics）の設定
   - Publication設定の考慮

3. **ファイル命名規則**
   - 記事slugはhuman readableに
   - 英語ベースの分かりやすいファイル名
   - ハイフン区切りを使用

### 技術スタック

- **パッケージマネージャー**: Yarn v4.10.2
- **記事管理**: Zenn CLI v0.2.9
- **バージョン管理**: Git/GitHub
- **実行環境**: Node.js v18以上

### 利用可能なコマンド

- `yarn new:article` - 新規記事作成
- `yarn new:book` - 新規本作成
- `yarn preview` - プレビュー（localhost:8000）

### 記事メタデータの構造

```yaml
---
title: "記事タイトル"
emoji: "😺"              # 絵文字
type: "tech" / "idea"    # 記事タイプ
topics: ["topic1", "topic2"]  # トピックス（最大5つ）
published: true/false         # 公開状態
published_at: "YYYY-MM-DD HH:MM"  # 公開日時（オプション）
publication_name: "pivot-media"   # パブリケーション名（オプション）
---
```

## 参考リンク

- [Zenn CLI Guide](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [Zenn Publication機能](https://zenn.dev/zenn/articles/what-is-publication)
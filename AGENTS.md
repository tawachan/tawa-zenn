# AGENTS.md

このファイルはClaude Codeで使用するAIエージェントの設定を定義します。

## プロジェクト概要

このリポジトリは@tawachanのZenn記事管理用リポジトリです。

## 主な機能

- Zenn記事の作成・管理
- GitHub連携による記事の自動同期
- プレビュー機能による記事の事前確認

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

- **パッケージマネージャー**: Yarn v4
- **記事管理**: Zenn CLI
- **バージョン管理**: Git/GitHub

## 参考リンク

- [Zenn CLI Guide](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [Zenn Publication機能](https://zenn.dev/zenn/articles/what-is-publication)
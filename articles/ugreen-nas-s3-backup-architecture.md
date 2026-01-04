---
title: "UGreen NASをS3にバックアップする構成"
emoji: "💾"
type: "idea"
topics:
  - "NAS"
  - "S3"
  - "backup"
  - "docker"
published: false
published_at: "2026-01-04 10:00"
# publication_name: "pivotmedia" # 個人記事のためコメントアウト
---

こんにちは。

今回は、UGreen NASを導入し、そのデータをAmazon S3にバックアップする構成について検討した内容をまとめます。

## 背景

個人用途でNASの導入を検討した結果、UGreenのNASync DH2300 NAS Storage ([製品ページ](https://nas.ugreen.jp/products/ugreen-nasync-dh2300-nas-storage?srsltid=AfmBOoqithkiRj1X2ju2W_yAJKTVQUj1b4NHxWfu0J8wcB5QdINyHVEW)) を購入しました。初めてのNASということもあり、手頃な価格帯のこのモデルを選択しました。

現在、8TBのHDDを1台のみ搭載していますが、将来的にはRAID1構成も視野に入れています。しかし、RAIDはあくまで物理的な冗長性を提供するものであり、災害や人為的ミスに対するバックアップとしては不十分です。そのため、クラウドストレージへのバックアップが必須と考え、コストパフォーマンスとAWSの安定性を考慮してAmazon S3を採用することにしました。

## 構成案

UGreen NAS DH2300は廉価版であり、メモリも4GBと少なめのため、標準のAppストアではDockerが推奨されていません。しかし、公式サイトからファイルをダウンロードしてインストールすることでDockerを動作させることが可能です（自己責任での運用となります）。

このDocker環境を利用し、定期的なバックアップを自動化する仕組みを構築します。

具体的な構成は以下の通りです。

1.  **Docker環境のセットアップ**: UGreen NAS DH2300にDockerをインストールします。
2.  **共有フォルダの準備**: バックアップ対象のデータが保存されている共有フォルダを設定します。
3.  **Cron Scheduler (Docker Compose)**: Docker Composeを使用してCronベースのスケジューラを構築し、定期的にバックアップスクリプトを実行します。
4.  **バックアップスクリプト (Docker Container)**: S3へのバックアップを実行するスクリプトを内包したDockerコンテナを作成します。このスクリプトは、NASの共有フォルダからS3へデータを同期します。

### 技術選定

-   **スケジューラ**: `cron` を利用したDockerコンテナで実現
-   **バックアップツール**: `aws cli` を利用したシェルスクリプト

## 次のステップ

具体的なDocker Composeファイルやバックアップスクリプトの実装、S3のバケット設定などについては、後続の記事で詳細を説明します。

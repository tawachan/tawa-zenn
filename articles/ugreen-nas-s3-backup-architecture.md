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
### 2. 共有フォルダの準備とDockerからのアクセス

UGreen NASの管理画面から、バックアップしたいデータが格納されている共有フォルダを作成・設定します。この際、フォルダのパス（例: `/volume1/BackupData` など）を控えておきます。

Dockerコンテナからこの共有フォルダにアクセスするためには、`docker-compose.yml` で `volumes` 設定を使用します。これにより、NAS上の物理パスをDockerコンテナ内のパスにマウントできます。

```yaml
volumes:
  - /volume1/BackupData:/app/backup_data # NASの共有フォルダをコンテナにマウント
```

上記の例では、NAS上の `/volume1/BackupData` をコンテナ内の `/app/backup_data` としてマウントしています。バックアップスクリプトはこのコンテナ内のパスにアクセスしてデータを処理します。

### 3. Cron Scheduler (Docker Compose)

Cronスケジューラは、`docker-compose.yml` を利用して構築します。ここでは、バックアップスクリプトを実行する別のDockerコンテナを定期的に起動する仕組みを考えます。

#### `docker-compose.yml` の例

```yaml
version: '3.8'
services:
  cron-scheduler:
    image: alpine/crond:latest # 軽量なalpineベースのcrondイメージを使用
    container_name: nas-backup-cron
    volumes:
      - ./crontabs:/etc/crontabs # crontabファイルをマウント
      - /var/run/docker.sock:/var/run/docker.sock # Dockerコマンドを実行するためにDockerソケットをマウント
    environment:
      - TZ=Asia/Tokyo # タイムゾーン設定
    restart: always # 常時起動

  backup-executor:
    build:
      context: ./backup-script # backup-scriptディレクトリにDockerfileを配置
      dockerfile: Dockerfile
    container_name: nas-s3-backup-executor
    volumes:
      - /volume1/BackupData:/app/backup_data # バックアップ対象データをマウント
    environment:
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
      - AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION}
      - S3_BUCKET_NAME=${S3_BUCKET_NAME}
    # crondからの実行を想定し、restart: "no" に設定
    restart: "no"
```

#### `crontabs` ファイルの例

`crontabs/root` ファイルとして以下の内容を作成します。これは、毎日午前3時に `backup-executor` コンテナを実行する例です。

```cron
# 毎日午前3時にバックアップを実行
0 3 * * * docker start nas-s3-backup-executor && docker logs -f nas-s3-backup-executor
```

この設定では、`cron-scheduler` コンテナ内で `docker start nas-s3-backup-executor` コマンドを実行し、バックアップ処理を行うコンテナを起動します。`docker logs -f nas-s3-backup-executor` はログを追跡するためのものですが、実際の運用ではバックグラウンド実行を検討するなど調整が必要です。

### 4. バックアップスクリプト (Docker Container)

S3へのバックアップを実行するコンテナ `backup-executor` を作成します。このコンテナは `aws cli` を利用して `s3 sync` コマンドを実行します。

#### `backup-script/Dockerfile` の例

```dockerfile
FROM amazon/aws-cli:latest

WORKDIR /app

COPY backup.sh .
RUN chmod +x backup.sh

ENTRYPOINT ["./backup.sh"]
```

#### `backup-script/backup.sh` の例

```bash
#!/bin/bash

# 環境変数からAWS認証情報とS3バケット名を取得
if [ -z "$AWS_ACCESS_KEY_ID" ] || [ -z "$AWS_SECRET_ACCESS_KEY" ] || [ -z "$S3_BUCKET_NAME" ]; then
  echo "Error: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, and S3_BUCKET_NAME must be set."
  exit 1
fi

SOURCE_DIR="/app/backup_data" # コンテナにマウントされたNASの共有フォルダ
TARGET_S3_PATH="s3://${S3_BUCKET_NAME}/nas_backup" # S3上のパス

echo "Starting S3 backup from ${SOURCE_DIR} to ${TARGET_S3_PATH} at $(date)"

# AWS S3 Sync コマンドを実行
# --delete をつけるとS3側にないファイルが削除されるため注意
aws s3 sync "${SOURCE_DIR}" "${TARGET_S3_PATH}" --exclude ".*" --exclude "*/.thumbnails/*"

if [ $? -eq 0 ]; then
  echo "S3 backup completed successfully at $(date)"
else
  echo "Error: S3 backup failed at $(date)"
fi
```

上記の `backup.sh` スクリプトは、NASからマウントされた `/app/backup_data` ディレクトリの内容をS3バケットの `nas_backup` プレフィックス以下に同期します。`.env` ファイルやDocker Composeの `environment` セクションを通じてAWS認証情報などを渡します。


## まとめ

本記事では、UGreen NASync DH2300 NAS StorageのデータをAmazon S3にバックアップするための構成案について解説しました。

*   UGreen NASでのDocker環境構築（非公式な方法による）
*   NAS共有フォルダとDockerコンテナ間のボリュームマウント
*   Docker ComposeによるCronスケジューラの導入
*   AWS CLIを利用したS3バックアップスクリプトとDockerイメージの作成
*   AWS認証情報のセキュアな管理 (`.env` ファイルの利用)
*   バックアップログの確認と永続化、および監視の重要性

この構成により、安価なNASでも信頼性の高いクラウドバックアップを自動化できます。特に `aws s3 sync` は差分バックアップにも対応しているため、効率的な運用が可能です。

ただし、本構成はUGreen NASのDocker環境が公式サポート外である点、またNASのハードウェアリソースが限られている点を考慮し、自己責任で運用してください。

この情報が、皆さんのデータ保護の一助となれば幸いです。


### バックアップログの管理

バックアップ処理が正常に完了したか、またはエラーが発生したかを把握するためには、ログの管理が不可欠です。

#### 1. `docker logs` による確認

最も基本的な方法は、`docker logs` コマンドを使用してコンテナの標準出力と標準エラー出力を確認することです。

```bash
docker logs nas-s3-backup-executor
```

`backup.sh` スクリプト内で `echo` を使って出力しているメッセージが、このコマンドで確認できます。

#### 2. ログの永続化

Dockerのデフォルト設定では、ログはコンテナが削除されると失われます。NASの再起動時やコンテナの再作成時にもログを参照できるようにするためには、ログを永続化する必要があります。

##### Dockerのログドライバを利用

`docker-compose.yml` のサービス定義に `logging` セクションを追加することで、ログドライバを設定できます。

```yaml
services:
  backup-executor:
    # ...
    logging:
      driver: "local" # ホストのローカルファイルシステムにログを保存
      options:
        max-size: "10m" # ログファイルの最大サイズ
        max-file: "3"   # 保持するログファイルの数
```

`local` ドライバを使用すると、ログはホストの `/var/lib/docker/containers/<container-id>/<container-id>-json.log` に保存されます。NASであれば、このパスを共有フォルダなどにマウントして管理することも検討できます。

##### その他のログ管理

より高度なログ管理が必要な場合は、`fluentd` や `syslog` などのログドライバを利用し、中央集中のログ管理システム（例: ELK Stack, Datadog）にログを転送することも可能です。個人利用のNASではそこまで必要ないかもしれませんが、オプションとして覚えておくと良いでしょう。

#### 3. ログ監視と通知

バックアップの成否を定期的に確認することは非常に重要です。ログの内容を監視し、エラーが発生した場合にはメールやSlackなどで通知する仕組みを導入することで、バックアップが失敗した際に迅速に対応できます。

シンプルな方法としては、`cron` ジョブに加えて、`backup.sh` の実行結果に応じて通知を送るスクリプトを追加することが考えられます。

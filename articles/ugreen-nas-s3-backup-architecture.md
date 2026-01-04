---
title: "UGreen NASのデータをS3にrcloneでバックアップする構成メモ"
emoji: "💾"
type: "tech"
topics:
  - "NAS"
  - "S3"
  - "backup"
  - "docker"
  - "rclone"
published: true
published_at: "2026-01-04 23:00"
---

もう仕事始めが近くて、年末年始の9連休が終わるなんて信じられない [@tawachan](https://x.com/tawachan39) です。

年末年始休暇のミッションとして、UGreen NASを購入して個人データの移行を進めていました。ようやく落ち着いたので、その中で構築したS3へのバックアップの仕組みをメモとして残しておきます。

NASだけだと災害や故障に対して不安があるため、クラウドにバックアップを取ることにしました（ちなみに一番時間がかかったのは、Google Photosからの写真ダウンロードと適切な配置、撮影日時がバグっているものの調整でしたが...）。

## 背景

個人用途でNASの導入を検討した結果、UGreenのNASync DH2300 NAS Storage[^nas-model] を購入しました。初めてのNASということもあり、手頃な価格帯のこのモデルを選択しました。

現在、8TBのHDDを1台のみ搭載しています[^hdd-config]。RAIDはあくまで物理的な冗長性を提供するものであり、NAS自体が物理的に壊れたら終わりです。また、災害や人為的ミス（誤削除など）に対するバックアップとしても不十分です。

そのため、クラウドストレージへのバックアップが必須と考えました。S3を選んだのは、Google Cloud StorageやAzure Blob Storageと比べてコストが安そう[^cloud-comparison]なのと、AWSの方がメジャーでトラブルシュート情報が見つかりやすいという理由からです。

[^nas-model]: [UGreen NASync DH2300 NAS Storage 製品ページ](https://nas.ugreen.jp/products/ugreen-nasync-dh2300-nas-storage)

[^hdd-config]: 2台でRAID1を組んで冗長構成にすることも考えましたが、HDD追加費用がそこそこかかることと、根本的なバックアップにはならないため、まずは1台から始めました。使う容量に対して8TBで十分であれば将来的に冗長性のために追加、足りなければ拡張のために買い足すかもしれません。

[^cloud-comparison]: ストレージの保存料金だけを見ると一番安かったのですが、アップロードやデータ確認時のリクエスト料金まで含めたトータルコストで比較したわけではありません。実際に運用してみて、リクエスト料金が想像以上にかかることがわかりました。

## 要件

バックアップの仕組みを作るにあたって、以下のような要件を考えました。

### コストを抑える

個人用途なので、できるだけコストを抑えたいというのが最優先です。バックアップは災害やNASの故障など、万が一の事態に備えるためのもので、通常の運用ではアクセスしません。

### 定期実行で意識しないで済む

手動でバックアップを取るのは忘れるリスクがあります。基本的に意識しなくても自動でバックアップが取られる状態を目指します。

### 差分同期で実行時間を短縮

8TBのデータを毎回フルバックアップするのは、ネットワークにも負荷をかけますし、実行時間も長くなります。NASを使っている最中にバックアップ処理が動いているのは避けたい状況です。

### 運用の仕組みをシンプルに

「今どういう状況なのか」が複雑だと、デバッグが難しくなったり運用が面倒になります。以下のような複雑さは避けたいと考えました。

- ファイルの更新日時ベースでの管理
- フォルダごとにバックアップの頻度やストレージクラスを変える
- ファイルの種類によって異なる扱いをする

### 不要なファイルを適切に除外

ドットファイル（`.DS_Store`など）やNAS特有のシステムフォルダ（`#recycle`（ゴミ箱）など）は、バックアップ対象から除外します。削除したファイルについては、S3側でバージョニングが有効であれば、そちらで期間内は辿れるため不要と判断しました。

### NASの運用をバックアップのために変えない

バックアップという本来関係ないもののために、NASの使い方を制限したくありません。「バックアップしたいものは特定のフォルダに入れる」といった運用ルールは設けたくない状況です。

## 構成

上記の要件を満たすため、以下のような構成にしました。

- **ストレージクラス**: S3 DEEP_ARCHIVE（コスト最優先）
- **同期方式**: rclone syncによる差分同期
- **実行方法**: Docker + cronによる定期実行（毎日午前2時）
- **除外設定**: ドットファイル、システムフォルダをフィルタで除外
- **対象範囲**: NAS全体（`/home`）をそのままバックアップ

### Docker環境のセットアップ

UGreen NAS DH2300は廉価版で、メモリも4GBと少なめです。調べてみると、デフォルトではDockerに対応していなさそうでした[^docker-support]。しかし、公式サイトからファイルをダウンロードしてインストールすることで、Dockerを動作させることができました[^docker-install]。

![UGreen公式サイトのDockerダウンロードページ](/images/ugreen-nas-docker-download.png)

Dockerが使えるようになったことで、バックアップの自動化が実現できました[^docker-limitation]。

[^docker-support]: [Reddit: DH2300 does not support Docker but...](https://www.reddit.com/r/UgreenNASync/comments/1of9lny/dh2300_does_not_support_docker_but/) など、標準のAppストアではDockerが選択できないという情報がありました。

[^docker-install]: [実際にインストールできたときのツイート](https://x.com/tawachan39/status/2003465060535169464)。公式サポート外のため、自己責任での運用となります。

[^docker-limitation]: ただし、UGreen NASのDockerはDocker Hubからしかリモートイメージを取得できないようです。GitHub Container RegistryやAmazon ECRなどは使えないため、使えるイメージが限られます。結局、ローカルにDockerfileを作って自分でビルドし、ローカルイメージを使っていく方が安牌そうだという所感を得ました。

### 全体構成図

```
┌─────────────────────────────┐
│   UGreen NAS                │
│                             │
│  ┌────────────────────────┐ │
│  │ Cron Scheduler         │ │
│  │ (Alpine + crond)       │ │
│  │                        │ │
│  │ 定期実行（毎日午前2時）  │ │
│  └───────┬────────────────┘ │
│          │                  │
│          │ docker run       │
│          ▼                  │
│  ┌────────────────────────┐ │
│  │ rclone sync            │ │
│  │ (実行時のみ起動)        │ │
│  │                        │ │
│  │ /home → S3             │ │
│  └────────────────────────┘ │
│                             │
└─────────────────────────────┘
              │
              │ rclone sync
              ▼
    ┌───────────────────┐
    │  Amazon S3        │
    │  DEEP_ARCHIVE     │
    └───────────────────┘
```

### ディレクトリ構成

```
/volume1/docker/nas-backup/
├── docker-compose.yaml
├── .env
├── rclone/
│   └── rclone.conf
├── scheduler/
│   ├── Dockerfile
│   └── entrypoint.sh
├── scripts/
│   └── backup.sh
└── logs/
```

## 実装

### 1. rclone設定ファイル

`rclone/rclone.conf`:

```ini
[s3]
type = s3
provider = AWS
env_auth = true
region = us-east-1
storage_class = DEEP_ARCHIVE
```

`env_auth = true` により、環境変数からAWS認証情報を取得します。これにより、設定ファイルに認証情報を書かずに済みます。

リージョンは `us-east-1`（バージニア）を選択しています。東京リージョンと比べてコストが安いためです[^region-cost]。

[^region-cost]: 例えば、DEEP_ARCHIVEのストレージコストは us-east-1 が $0.00099/GB/月、ap-northeast-1（東京）が $0.002/GB/月 と約2倍の差があります。

### 2. Docker Compose設定

`docker-compose.yaml`:

```yaml
version: "3.9"

services:
  scheduler:
    build:
      context: ./scheduler
    container_name: rclone-scheduler
    restart: unless-stopped

    env_file:
      - .env

    working_dir: /work

    environment:
      TZ: Asia/Tokyo
      AWS_DEFAULT_REGION: us-east-1
      CRON_SCHEDULE: "0 2 * * *"

      CRON_COMMAND: >
        docker run --rm --read-only
        --entrypoint sh
        -e TZ=Asia/Tokyo
        -e AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
        -e AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
        -e AWS_DEFAULT_REGION=us-east-1
        -v /home:/data:ro
        -v /volume1/docker/nas-backup/rclone:/config/rclone:ro
        -v /volume1/docker/nas-backup/scripts:/scripts:ro
        -v /volume1/docker/nas-backup/logs:/logs
        rclone/rclone:latest
        /scripts/backup.sh

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./:/work
```

ポイント:

- `scheduler` サービスが cron として動作
- `CRON_COMMAND` でrcloneコンテナを実行
- `/home` を読み取り専用 (`:ro`) でマウント
- AWS認証情報は環境変数で渡す

### 3. スケジューラコンテナ

`scheduler/Dockerfile`:

```dockerfile
FROM alpine:3.20

RUN apk add --no-cache tzdata docker-cli

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

`scheduler/entrypoint.sh`:

```sh
#!/bin/sh
set -eu

: "${CRON_SCHEDULE:?CRON_SCHEDULE is required}"
: "${CRON_COMMAND:?CRON_COMMAND is required}"

# cron 設定を生成
echo "${CRON_SCHEDULE} ${CRON_COMMAND}" > /etc/crontabs/root

# foreground 実行
exec crond -f -l 8
```

Alpine Linuxの軽量イメージを使い、Docker CLIをインストールしています。これにより、スケジューラコンテナから別のDockerコンテナ（rclone）を起動できます。

### 4. バックアップスクリプト

`scripts/backup.sh`:

```sh
#!/bin/sh
set -eu

SRC="/data"
S3_BUCKET="your-nas-backup-bucket"
S3_PREFIX="home"
REMOTE="s3:${S3_BUCKET}/${S3_PREFIX}"

LOG_DIR="/logs"
TS="$(date +%F_%H%M%S)"
LOG_FILE="${LOG_DIR}/rclone-sync-${TS}.log"

mkdir -p "$LOG_DIR"

log() { echo "$(date '+%F %T') $*"; }

log "START: rclone sync ${SRC} -> ${REMOTE}"
log "INFO: log_file=${LOG_FILE}"
log "INFO: options: size-only=on delete-excluded=on progress=on stats=30s"
log "INFO: excludes: dot-files, dot-dirs, #recycle, DS_Store, Thumbs.db"

rclone sync "$SRC" "$REMOTE" \
  --config /config/rclone/rclone.conf \
  --size-only \
  --delete-excluded \
  --filter "- **/.*/**" \
  --filter "- **/.*" \
  --filter "- **/#recycle/**" \
  --filter "- **/@eaDir/**" \
  --filter "- **/.DS_Store" \
  --filter "- **/Thumbs.db" \
  --filter "+ **" \
  --log-level INFO \
  --log-file "$LOG_FILE" \
  --stats 30s \
  -P
```

ポイント:

- `--size-only`: サイズだけで変更判定（タイムスタンプは見ない）
- `--delete-excluded`: S3側の余分なファイルを削除（同期）
- `--filter`: ドットファイル、システムフォルダを除外

### 5. 環境変数設定

`.env` ファイルにAWS認証情報を記載します（Gitにはコミットしない）。

```.env
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=xxxxx...
```

### 6. コンテナの起動

UGreen NASのDockerアプリのGUIから、上記の`docker-compose.yaml`を配置したディレクトリを指定してコンテナを起動します。GUIから設定すると、自動的にコンテナがビルド・起動され、毎日午前2時にバックアップが実行されるようになります。

## 動作確認

手動でバックアップを実行して動作を確認します。

```bash
docker exec -it rclone-scheduler sh -lc 'eval "$CRON_COMMAND"'
```

実行すると、以下のような出力が得られます。

```
2026-01-03 20:16:11 START: rclone sync /data -> s3:your-nas-backup-bucket/home
2026-01-03 20:16:11 INFO: log_file=/logs/rclone-sync-2026-01-03_201611.log
2026-01-03 20:16:11 INFO: options: size-only=on delete-excluded=on progress=on stats=30s
2026-01-03 20:16:11 INFO: excludes: dot-files, dot-dirs, #recycle, DS_Store, Thumbs.db
Transferred:              0 B / 0 B, -, 0 B/s, ETA -
Checks:            192069 / 192069, 100%, Listed 391389
Deleted:                4 (files), 0 (dirs), 24.423 MiB (freed)
Elapsed time:       1m8.4s
```

ログは `/volume1/docker/nas-backup/logs/` に保存されます。

## 運用してわかったこと

### 初回同期には時間がかかる

初回のバックアップでは全ファイルをアップロードするため、数時間〜数十時間かかります（データ量による）。その後の差分同期は数分〜数十分で完了します。

### S3のコストは想像以上にかかる

DEEP_ARCHIVEのストレージコスト自体は非常に安価（$0.00099/GB/月）です[^storage-cost]。しかし、**アップロード時のPutObjectリクエスト料金がファイル数に応じてかかります**[^put-pricing]。

自分の場合、写真が大量にあるためファイル数が多く[^file-count]、PutObjectは数に応じて料金が発生するため、想像以上にコストがかかりました。最初は差分確認のための無駄なリクエストでコストがかかっているのではと思いましたが、内訳を確認したところPutObjectでした。

[^storage-cost]: DEEP_ARCHIVEは90日以内の削除に早期削除料金がかかりますが、ストレージコスト自体が全体の中ではマイナーだったため、あまり気にしていません。写真のフォルダ整理程度であれば影響はほとんどありません。

[^put-pricing]: S3のPUT、COPY、POST、LISTリクエストは、ストレージクラスによって異なる料金が設定されています。詳細は [AWS S3 料金ページ](https://aws.amazon.com/s3/pricing/) を参照。DEEP_ARCHIVEの場合、1,000リクエストあたり最大$0.05程度かかります。

[^file-count]: 写真フォルダだけで約16万ファイル（159,884個）ありました。写真以外も含めると、かなりの数になります。計算してみると、159,884 ÷ 1,000 × $0.05 = 約$8で、実際のPutObjectコスト$11.91と比較すると、試行錯誤やHeadObject等を含めて概ね妥当な範囲です。スケールとしては間違っていないので、変な処理があるわけでもなさそうと判断しています。

![S3のコスト内訳（ほぼPutObject）](/images/s3-cost-breakdown-put-object.png)

実際のコスト内訳を見ると、約1週間（2025/12/28〜2026/1/3）で以下のような費用になっています：

| 項目 | コスト | 割合 |
|------|--------|------|
| **PutObject** | **$11.91** | **87.7%** |
| HeadObject | $0.61 | 4.5% |
| UploadPart | $0.31 | 2.3% |
| 操作なし | $0.23 | 1.7% |
| **DeepArchiveStorage** | **$0.23** | **1.7%** |
| その他すべて | $0.39 | 2.9% |
| **合計** | **$13.58** | **100%** |

ストレージコスト（$0.23）と比べると、初回バックアップ時のPutObjectリクエストコスト（$11.91）がいかに大きいかがわかります。

最初は、TB単位で月1000円かからないくらいで保管できると思っていました。しかし、いきなり1日で$5くらい出て少し焦りました。

![S3の日次コスト推移](/images/s3-daily-cost-trend.png)

グラフを見ると、初日（12/28）は$4.72と高く、その後も$1〜3ドル台が続きました。これは、Google Photosからの写真移行で何万ファイル単位で毎日追加していたことが主な要因です。

写真移行が落ち着いてからは$0.22（1/3）程度になってきたので、平時は落ち着くだろうと考えています。通常運用でファイルを増やす範囲のものしか追加バックアップしていない状態であれば、当初想定していた月1000円以内に収まりそうです。

### ログのローテーション

ログファイルが増え続けるため、古いログを定期的に削除する仕組みが必要です。現時点では手動で削除していますが、将来的にはログローテーションスクリプトを追加する予定です。

### rcloneのオプション選択

`--size-only` オプションを使うことで、タイムスタンプの比較を省略してパフォーマンスを向上させています。

これはパフォーマンスの理由だけでなく、**S3のリクエストコストを抑える目的もあります**。各ファイルを個別に確認するとHeadObjectなどのリード系リクエストでコストが増える可能性があるため[^rclone-cost]、厳密に比較させないようにしています。

ただし、サイズが同じでも内容が変わる可能性があるファイル（データベースファイルなど）には注意が必要です。

[^rclone-cost]: rcloneの動作とS3のリクエストの関係を厳密に検証したわけではありません。ChatGPT・Claudeで相談しながら、最小限動くところまで試行錯誤した結果です。

## まとめ

UGreen NASのデータをS3にバックアップする構成について、実際に運用している内容をまとめました。

- rclone syncで差分同期
- Docker + cronで定期実行
- S3 DEEP_ARCHIVEでコスト最適化
- ドットファイル・システムフォルダの除外
- シンプルな運用で管理負荷を最小化

この構成により、安価なNASでも信頼性の高いクラウドバックアップを自動化できています。個人用途であれば月数百円程度のコストで、災害対策としてのバックアップを実現できます。

この情報が、同じようにNASのバックアップを検討している方の参考になれば幸いです。


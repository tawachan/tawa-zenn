---
title: "RESTの延長線を抜け出す —— PIVOT が Connect RPC を選んだ舞台裏"
emoji: "🔌"
type: "tech"
topics: ["connectrpc", "protobuf", "grpc", "go", "api"]
published: false
---

## はじめに

こんにちは。PIVOTでソフトウェアエンジニアとして、Web・バックエンド・インフラを横断している[@tawachan](https://x.com/tawachan39)です。

ビジネス映像メディア「PIVOT」は、YouTube に加えて iOS / Android アプリや Web でもコンテンツを届けています。2025 年には、視聴にとどまらずユーザー同士が学びを深め合えるコメント機能をリリースしました[^pivot-comment]。大規模な新機能であるこの開発を機に、API をスキーマファーストで設計し直し、サーバー側・クライアント側のコードを自動生成するフローを確立することを目標に掲げました。OpenAPI など他の手段も検討しましたが、最終的には **Connect RPC** を選択しています。

Connect RPC を採用することで、proto ファイルを起点に Go（サーバー）、TypeScript（Web）、Swift（iOS）、Kotlin（Android）へ同じスキーマを適用し、JSON と protobuf のどちらでも運用できる体制を整えました。この記事では、その意思決定プロセスと導入後の進め方・学びを紹介します。同じようにスキーマファースト開発を模索しているチームのヒントになれば幸いです。

## これまでの API 運用と積み残していた課題

### 手作業に頼る JSON ハンドラ

既存の REST ハンドラでは、クライアントから届いた JSON を手で構造体に詰め替え、バリデーションも散在したまま、最後にレスポンスだけ protobuf へ変換する運用を続けていました。レスポンスの protobuf コード生成はあっても、リクエスト定義はドキュメントを頼りに手書きするしかありません。仕様が変わるたびにサーバーとクライアント双方で差分を追い、実際に動かしてみるまで合っているかわからない──そんな状態が常態化していました。

### ドキュメント整備が追いつかない

仕様書は主に Notion で自然言語のまま管理しており、更新漏れが慢性的な課題でした。実装とドキュメントのどちらが正しいかを確認するために Slack での確認が増え、意思決定のスピードが落ちていました。

### Protobuf を部分的に使うだけでは限界があった

Protocol Buffers（以下 protobuf）はレスポンス形式として活用し、既存 API 全体のレスポンス高速化には貢献していました。ただしリクエスト側は JSON のままで、型定義を含む生成コードを活かせていたわけではありません。HTTP パスやクエリ、ボディの在り方も proto では表現しきれていないため、実際に何を送ればいいのか最終的にはドキュメントを読み解いて実装する必要がありました。コメント機能のような大きなコンテクストでも同じ手法を続ければ、せっかくの大規模刷新でも負債を増やすだけ――「このタイミングで通信レイヤを見直さないといけない」という危機感が高まりました。

## コメント機能を機に見直しを決断

コメント機能は、既存の動画視聴機能とは切り離した独立機能として開発する計画でした。モジュラーモノリス的に Go のコンポーネントを整理し直す作業も並行して進めており（この詳細は別稿で触れます）、どうせなら通信レイヤも負債を増やさない形にしたい。過去から見過ごしてきた課題を、このタイミングで解決する意味があると判断しました。

## 検討した選択肢

### OpenAPI で REST を再構築する

まず考えたのは、REST を続けながら OpenAPI によるスキーマ管理を徹底する案です。ただし現在の API はレスポンスに protobuf を使っており、そのままの仕様を OpenAPI で表現するのは難しい状況でした。レスポンスを JSON に戻したり、既存のハンドラ実装を大きく書き換えたりする必要があり、結局は運用の複雑さが残ると判断しました。

### gRPC を導入する

gRPC であれば protobuf を前提にしたスキーマ駆動開発へ舵を切れます。ただし HTTP/2 前提であるためインフラの再設計が必要で、ブラウザからの直接呼び出しは gRPC-Web を別途用意することになります。コメント機能のリリースや将来の運用を考えると、今このタイミングでインフラの複雑性を上げるのは現実的ではありませんでした。

### Connect RPC を採用する

Connect RPC は、Http/1.1 / 2 の両方で動作し、proto ファイルを中心に据えつつ JSON によるデバッグもしやすい構成が組めます。敷居の低さと将来への伸びしろを両立できることが採用の決め手でした。

## Connect RPC を選んだ理由

### Proto を起点にした開発フローを落とし込みやすい

OpenAPI でもスキーマファースト開発は可能ですが、既存のレスポンスが protobuf であるため JSON 型への置き換えが前提になります。Connect RPC であれば、これまでの proto ファイルを活かしつつリクエスト/レスポンス両方を整備でき、Go / TypeScript / Swift / Kotlin といった各言語向けコードを `buf generate` でまとめて更新できます。

### ツールチェーンとの親和性と導入実績

Connect RPC は Buf 社が開発しており、もともと導入していた `buf` との相性が良好です。国内でも LayerX などが採用しており、ナレッジやドキュメントが増え始めている点も安心材料でした[^layerx-connect]。

### Envoy が不要で導入が軽い

HTTP/1.1 にネイティブ対応しているため、ブラウザとサーバの間に gRPC-Web 用の Envoy を挟む必要がありません。Web クライアントがあっても、Connect のプロトコルを使えば同じサーバへ直接リクエストを送れるので、インフラをシンプルなまま維持できます。

### JSON エンドポイントでデバッグしやすい

Connect RPC は protobuf と JSON の両方を扱えます。curl や Postman での API 検証が従来の REST と同じ感覚で行えるため、検証フローを大きく変えずに移行できました。

## Connect RPC で組み始めてみる

### まずは proto ファイルを整える

コメント機能専用に proto ファイルを設計し直し、サーバー（Go）とクライアント（TypeScript / Swift / Kotlin）向けのコードを同時に生成するようにしました。

```protobuf
syntax = "proto3";

package pivot.comment.v1;

message PostCommentRequest {
  string episode_id = 1;
  string body = 2;
}

message PostCommentResponse {}

service CommentService {
  rpc PostComment(PostCommentRequest) returns (PostCommentResponse);
}
```

生成は `buf generate` に集約し、各プロダクトの SDK をまとめて更新できるようにしています。

```yaml
# buf.gen.yaml（抜粋）
version: v2
plugins:
  - remote: buf.build/connectrpc/go
    out: gen/go
    opt:
      - paths=source_relative
  - remote: buf.build/connectrpc/es
    out: gen/ts
    opt:
      - target=ts
  - remote: buf.build/connectrpc/swift
    out: gen/swift
  - remote: buf.build/connectrpc/kotlin
    out: gen/kotlin
```

### Go サーバー側の実装

Connect RPC では、生成されたリクエスト型をそのまま受け取り、ビジネスロジックに集中できます。認証やロギングなどの共通処理は interceptor に寄せました。

```go
func (s *CommentService) PostComment(
    ctx context.Context,
    req *connect.Request[commentv1.PostCommentRequest],
) (*connect.Response[commentv1.PostCommentResponse], error) {
    user := usercontext.MustFrom(ctx)

    if err := s.usecase.Comment().Post(
        ctx,
        user.ID,
        req.Msg.EpisodeId,
        req.Msg.Body,
    ); err != nil {
        return nil, connect.NewError(connect.CodeInternal, err)
    }

    return connect.NewResponse(&commentv1.PostCommentResponse{}), nil
}
```

### フロントエンド・モバイルクライアントとデバッグ

フロントエンドは `@connectrpc/connect-web` を採用し、生成されたクライアントをそのまま利用しています。モバイル（Swift / Kotlin）も同じ proto から生成された SDK を取り込むだけでよく、各プラットフォームでの差分実装は最小限です。

```ts
const client = createPromiseClient(CommentService, transport)
await client.postComment({
  episodeId: "ep_123",
  body: "この回の議論が刺さった！",
})
```

Connect RPC は JSON でもエンドポイントを公開できるので、curl や Postman でのデバッグも引き続き手軽です。外部システムと連携するときに Protobuf と JSON のどちらを選べるのか、という余地が残るのも実装上の安心材料でした。

## 移行の進め方

コメント機能の API を皮切りに、高速で改修が入る領域から Connect RPC を優先的に採用しています。一度に全てを置き換えるのではなく、重要度の高い API から移行し、REST と共存させながら段階的に面積を広げる方針です。現時点では proto ファイルが完全に共通資産になっているわけではありませんが、段階を踏んで「新しく書かれた proto は Connect で運用する」というリズムが定着しつつあります。

## 導入して見えてきたこと

### 型生成でコミュニケーションコストが下がった

リクエストとレスポンスで同じ構造体を二重に書き直す必要がなくなり、「型がずれていた」というミスが激減しました。生成されたコードを軸にレビューできるので、パラメータの命名や扱いについての会話が噛み合いやすくなりました。

### ドキュメントの整合性を保ちやすくなった

仕様変更が入ったら proto を直し、`buf generate` を叩く。このサイクルを守れば、IDE の補完とコンパイルが差分を教えてくれます。手作業でメンテしていた頃に比べ、Notion や OpenAPI の記述漏れでストレスを感じるシーンが減りました。

### JSON エンドポイントがデバッグを支えてくれる

Connect RPC は protobuf で通信するのが基本ですが、同じエンドポイントを JSON でも公開できます。エンジニア以外が動作確認したいときや、外部と一時的に連携するときに柔軟さがあり、導入当初に懸念していた「デバッグ体験が悪化する」問題は起こりませんでした。

## まとめ

慢性的に抱えていた REST の課題を、コメント機能という大きな節目に合わせて解消できたのが Connect RPC 導入の一番の価値でした。proto ファイルを軸に据え直したことで、サーバーとクライアントの境界がはっきりし、チーム内の会話もシンプルになりました。今後も高頻度で触れる領域から順に Connect RPC へ寄せ、段階的にモダンな開発体験へ揃えていくつもりです。

---

[^pivot-comment]: 「PIVOTアプリ/Web新コメント機能で進化する学びと交流の場」——PIVOT CEO 佐々木による紹介動画で、コメント機能の背景や活用イメージが語られています。（[[アプリ・Web] コメント機能がスタート【佐々木紀彦】](https://pivotmedia.co.jp/app/movie/13512?display_type=article)）
[^layerx-connect]: 「Protocol Buffers と Connect-go で段階的にモノリスからサービスを切り出した話」（LayerX テックブログ、2024年6月13日）https://tech.layerx.co.jp/entry/decoupling-a-service-from-monolith-with-Protocol-buffers-and-connect-go

---
title: "手動API運用からの脱却 —— PIVOTがConnect RPCで解決したチーム開発の課題"
emoji: "🔌"
type: "tech"
topics: ["connectrpc", "protobuf", "grpc", "go", "api"]
published: false
---

## はじめに

こんにちは。PIVOTでソフトウェアエンジニアとして、Web・バックエンド・インフラを横断している[@tawachan](https://x.com/tawachan39)です。

ビジネス映像メディア「PIVOT」は、YouTube に加えて iOS / Android アプリや Web でもコンテンツを届けています。2025 年には、視聴にとどまらずユーザー同士が学びを深め合えるコメント機能をリリースしました[^pivot-comment]。大規模な新機能であるこの開発を機に、API をスキーマファーストで設計し直し、サーバー側・クライアント側のコードを自動生成するフローを確立することを目標に掲げました。OpenAPI など他の手段も検討しましたが、最終的には **Connect RPC** を選択しています。

Connect RPC を採用することで、proto ファイルを起点に Go（サーバー）、TypeScript（Web）、Swift（iOS）、Kotlin（Android）へ同じスキーマを適用し、JSON と protobuf のどちらでも運用できる体制を整えました。この記事では、その意思決定プロセスと導入後の進め方・学びを紹介します。同じようにスキーマファースト開発を模索しているチームのヒントになれば幸いです。

## REST API運用で抱えていた3つの課題

### 手作業に頼るAPI実装と型安全性の欠如

既存のREST APIでは、**クライアントから届いたJSONを手で構造体に詰め替え**、バリデーションも散在したまま、最後にレスポンスだけprotobufへ変換する運用を続けていました。レスポンスのprotobufコード生成はあっても、**リクエスト定義はドキュメントを頼りに手書きするしかありません**。

仕様が変わるたびにサーバーとクライアント双方で差分を追い、**実際に動かしてみるまで合っているかわからない**──そんな状態が常態化していました。

実際に、**サーバー側の仕様ではPUTメソッドで定義されているエンドポイントを、クライアント側でPOSTとして実装してしまい、404エラーで動かない**といったケースも発生していました。こうした手動実装による認識違いは、開発効率を大きく損なう要因となっていました。

![仕様確認が必要になる日常的なSlackでのやり取り](/images/pivot-connect-rpc-slack-conversation.png)

このように**仕様とコードの乖離により、日常的な確認作業が発生**し、本来の開発に集中できない状況が続いていました。

### ドキュメントの信頼性低下による日常的な確認作業

仕様書は主にNotionで**自然言語のまま管理**しており、以下のような問題が慢性化していました：

- **APIによって記載項目がバラバラ**：書く人や時期によって詳細度が異なる
- **実装中の仕様変更が反映されない**：設計時には書くが、実装しながらの調整が漏れる
- **サーバー実装が仕様通りではない**：実装時の都合で仕様とコードにズレが生じる

この結果、**ドキュメントの信頼性が低下**し、実装とドキュメントのどちらが正しいかを確認するために**Slackでの問い合わせが増加**。意思決定のスピードが落ちていました。

### 重要な仕様がProtoで表現できない限界

Protocol Buffers（以下protobuf）は**レスポンス形式として活用**し、既存API全体のレスポンス高速化には貢献していました。ただし**リクエスト側はJSONのまま**で、型定義を含む生成コードを活かせていたわけではありません。

さらに深刻だったのは、**重要な仕様がprotoで定義されていない**ことでした：

- **ページネーションのLimit仕様**：最大値や既定値が曖昧
- **ソート・並び替えオプション**：指定可能なキーが不明確
- **実装時に決まる細かい挙動**：エラーハンドリングや境界値処理

これらは**実装者によってNotionへの記載レベルが異なり**、結果として**人力運用の厳しさが露呈**していました。コメント機能のような大きなコンテクストでも同じ手法を続ければ、せっかくの大規模刷新でも負債を増やすだけ――**「このタイミングで通信レイヤを見直さないといけない」という危機感**が高まりました。

## 新機能開発を機に通信レイヤを刷新

コメント機能は、**既存の動画視聴機能とは切り離した独立機能として開発**する計画でした。モジュラーモノリス的にGoのコンポーネントを整理し直す作業も並行して進めており（この詳細は別稿で触れます）、どうせなら通信レイヤも負債を増やさない形にしたい。

**過去から見過ごしてきた課題を、このタイミングで解決する意味がある**と判断しました。

## スキーマファーストのための3つの選択肢

### 案1: OpenAPIでRESTを再構築する

まず考えたのは、**RESTを続けながらOpenAPIによるスキーマ管理を徹底する**案です。ただし現在のAPIはレスポンスにprotobufを使っており、**そのままの仕様をOpenAPIで表現するのは難しい状況**でした。

レスポンスをJSONに戻したり、既存のハンドラ実装を大きく書き換えたりする必要があり、**結局は運用の複雑さが残る**と判断しました。

### 案2: gRPCを導入する

**gRPCであればprotobufを前提にしたスキーマ駆動開発へ舵を切れます**。ただし**HTTP/2前提であるためインフラの再設計が必要**で、ブラウザからの直接呼び出しはgRPC-Webを別途用意することになります。

コメント機能のリリースや将来の運用を考えると、**今このタイミングでインフラの複雑性を上げるのは現実的ではありません**でした。

### 案3: Connect RPCを採用する

**Connect RPCは、HTTP/1.1 / 2の両方で動作し、protoファイルを中心に据えつつJSONによるデバッグもしやすい構成が組めます**。**敷居の低さと将来への伸びしろを両立できること**が採用の決め手でした。

## Connect RPCを選択した4つの決め手

### 既存のProtobuf資産を活用できる

**OpenAPIでもスキーマファースト開発は可能**ですが、既存のレスポンスがprotobufであるためJSON型への置き換えが前提になります。**Connect RPCであれば、これまでのprotoファイルを活かしつつリクエスト/レスポンス両方を整備でき**、Go / TypeScript / Swift / Kotlinといった各言語向けコードを`buf generate`でまとめて更新できます。

### 既存ツールチェーンとの高い親和性

**Connect RPCはBuf社が開発しており、もともと導入していた`buf`との相性が良好**です。国内でもLayerXなどが採用しており、ナレッジやドキュメントが増え始めている点も安心材料でした[^layerx-connect]。

### インフラのシンプルさを維持

**HTTP/1.1にネイティブ対応しているため、ブラウザとサーバの間にgRPC-Web用のEnvoyを挟む必要がありません**。Webクライアントがあっても、Connectのプロトコルを使えば同じサーバへ直接リクエストを送れるので、**インフラをシンプルなまま維持**できます。

### デバッグ体験を維持

**Connect RPCはprotobufとJSONの両方を扱えます**。curlやPostmanでのAPI検証が**従来のRESTと同じ感覚で行える**ため、検証フローを大きく変えずに移行できました。

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

## 段階的な移行戦略：RESTとの共存から始める

**コメント機能のAPIを皮切りに、高速で改修が入る領域からConnect RPCを優先的に採用**しています。一度に全てを置き換えるのではなく、**重要度の高いAPIから移行し、RESTと共存させながら段階的に面積を広げる方針**です。

現時点ではprotoファイルが完全に共通資産になっているわけではありませんが、段階を踏んで**「新しく書かれたprotoはConnectで運用する」というリズムが定着しつつあります**。

## Connect RPC導入で実現した3つの改善

### チーム内コミュニケーションの整流化

**リクエストとレスポンスで同じ構造体を二重に書き直す必要がなくなり**、「型がずれていた」というミスが激減しました。**生成されたコードを軸にレビューできる**ので、パラメータの命名や扱いについての会話が噛み合いやすくなりました。

以前のように**HTTPメソッドやパスの認識違いが起こることもなくなり**、「このAPIはPOSTですかPUTですか？」「404になるんですが、仕様はどうなってますか？」といった**確認のためのSlack投稿も大幅に減りました**。protoファイルから生成されたクライアントコードを使えば、正しいメソッドとパスで確実にリクエストが送信されるためです。

### スキーマ駆動開発の実現

**仕様変更が入ったらprotoを直し、`buf generate`を叩く**。このサイクルを守れば、**IDEの補完とコンパイルが差分を教えてくれます**。手作業でメンテしていた頃に比べ、**NotionやOpenAPIの記述漏れでストレスを感じるシーンが減りました**。

### デバッグ体験の維持

**Connect RPCはprotobufで通信するのが基本ですが、同じエンドポイントをJSONでも公開できます**。エンジニア以外が動作確認したいときや、外部と一時的に連携するときに柔軟さがあり、**導入当初に懸念していた「デバッグ体験が悪化する」問題は起こりませんでした**。

## まとめ：手動運用からスキーマ駆動への転換

**慢性的に抱えていたREST API運用の課題を、コメント機能という大きな節目に合わせて解消できた**のがConnect RPC導入の一番の価値でした。**protoファイルを軸に据え直したことで、サーバーとクライアントの境界がはっきりし、チーム内の会話もシンプルになりました**。

今後も高頻度で触れる領域から順にConnect RPCへ寄せ、**段階的にモダンな開発体験へ揃えていく**つもりです。

---

[^pivot-comment]: 「[アプリ・Web] コメント機能がスタート【佐々木紀彦】」（PIVOT、2025年10月15日）https://pivotmedia.co.jp/app/movie/13512?display_type=article
[^layerx-connect]: 「Decoupling a service from monolith with Protocol buffers and connect-go」（LayerX エンジニアブログ、2023年7月4日）https://tech.layerx.co.jp/entry/decoupling-a-service-from-monolith-with-Protocol-buffers-and-connect-go

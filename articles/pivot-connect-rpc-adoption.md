---
title: "既存 Protocol Buffers をフル活用する - PIVOT の Connect RPC 導入の意思決定"
emoji: "🔌"
type: "tech"
topics: ["protobuf", "connectrpc", "grpc", "go", "api"]
published: false
---

## はじめに

こんにちは。PIVOTでソフトウェアエンジニアとして、Webフロントエンド、バックエンド、インフラを横断的に担当している[@tawachan](https://x.com/tawachan39)です。

PIVOTでは、映像配信を中心としたコンテンツ提供システムを運用していますが、新たにユーザー参加型のコメント機能を開発することになりました。

実はこれまで、Protocol Buffers（以下Proto）を「REST APIのレスポンスのシリアライズ専用」として使う、かなり片手落ちな運用をしていました（この使い方は特殊事例なので、あまり一般的な参考にはならないかもしれません）。リクエストはJSON、レスポンスだけProto、通信コードの自動生成もなし──Protoの本来のうまみをほとんど活かせていない状態です。

新しいコメント機能の開発にあたり、API設計や型安全性、開発効率を見直す必要が出てきました。現代的なAPI開発ではOpenAPI（Swagger）でYAML管理・自動生成する手法もありますが、YAMLの管理コストや既存システムとの統合を考えると、Protoベースで自動生成したいという思いが強くなりました。

ただし、gRPCはEnvoyなどのプロキシ導入やインフラの複雑化がネック。そこで、HTTP/1.1にも対応し、ブラウザやCDNとの相性も良く、柔軟に運用できる**Connect RPC**を選択し、実際に導入しました。

本記事では、その技術選定の意思決定プロセスを詳しく解説します。

### 本記事で伝えたいこと

- Protocol Buffers を REST API のレスポンスだけに使うのは「もったいない」
- 既存 Proto 資産を活かしながら RPC の恩恵を得る方法
- OpenAPI、gRPC、Connect RPC の比較検討プロセス
- 「捨てる技術」ではなく「活かす技術」を選ぶ意思決定

技術選定において「何を選んだか」よりも「なぜその技術を選んだか」の思考プロセスを共有することで、同じような課題を抱えている方の参考になれば幸いです。

## 既存システムの状況と課題

### Protocol Buffers の「もったいない」使い方

PIVOT では、すでに Protocol Buffers (以下 Proto) を導入していました。しかし、その使い方は**REST API のレスポンスのシリアライズのみ**でした。

```
既存の Proto 活用方法:
REST API リクエスト → JSON で受け取り
  ↓
サーバー処理
  ↓
レスポンス → Protocol Buffers でシリアライズ

問題点:
├── レスポンスのみ Proto（リクエストは JSON のまま）
├── RPC ではない（単なるシリアライゼーション形式）
└── スキーマからの通信コード生成なし
```

これは Proto の持つポテンシャルの一部しか活用できていない状態でした。

### 新機能開発で顕在化した課題

コメント機能という新しいインタラクション機能の開発において、以下の課題が顕在化しました。

**1. 手動実装による開発効率の問題**

REST API では、リクエストの解析、バリデーション、レスポンスの構築をすべて手動で実装する必要がありました。

```go
// 従来の REST API 実装例
func (c *V1ChapterController) RegisterUserRating(ctx echo.Context) error {
    // 1. リクエスト解析（手動）
    request := context.MustGetRequestDataFrom[RegisterUserRatingRequestData](ctx)
    u := context.GetUserContext(ctx)

    // 2. バリデーション（手動）
    displayType, err := valueobject.NewEpisodeDisplayType(request.DisplayType)
    if err != nil {
        return errors.NewBadRequestError(errors.WithMessage("不正な表示タイプです。"))
    }

    // 3. ビジネスロジック
    err = c.useCase.Chapter().RegisterUserRatingUseCase(
        ctx.Request().Context(), u.MembershipId, request.EpisodeId,
        request.RatingValue, displayType,
    )
    if err != nil {
        return failure.Wrap(err)
    }

    // 4. レスポンス生成（手動 Protobuf 変換）
    return c.Controller.Respond(ctx, &pivotapigo.RegisterUserRatingResponse{})
}
```

**2. クライアント・サーバー間の通信不整合リスク**

API スキーマの変更時、クライアント側のコードを手動で同期する必要があり、型の不整合によるバグが発生するリスクがありました。

**3. API ドキュメントの手動メンテナンス**

OpenAPI 仕様書を別途メンテナンスするか、コードとドキュメントの乖離を許容するしかない状態でした。

### なぜ今まで問題にならなかったのか

既存の映像 Viewer 機能は安定稼働しており、大きな API 変更も少なかったため、手動実装のオーバーヘッドは許容範囲でした。しかし、新機能開発において以下の要求が高まりました：

- **開発スピードの向上**: 頻繁な仕様変更への迅速な対応
- **型安全性の確保**: クライアント・サーバー間の通信ミスの削減
- **既存システムへの影響最小化**: 疎結合なアーキテクチャ

これらの要求を満たすため、通信層の技術選定を見直すことにしました。

## 技術選択肢の検討プロセス

新機能開発にあたり、以下の 3 つの選択肢を検討しました。

### 選択肢1: OpenAPI への切り替え

**検討した理由:**

スキーマ駆動開発を実現し、API ドキュメントとコードの一体化を図るため、OpenAPI (Swagger) への全面移行を検討しました。

- YAML でスキーマ定義
- コード生成ツール (OpenAPI Generator など) でクライアント・サーバーコード自動生成
- Swagger UI による API ドキュメント自動生成

**却下した理由:**

真剣に検討しましたが、以下の理由で却下しました。

```
OpenAPI 採用の課題:
├── ❌ 既存 Proto 資産の廃棄
│     → これまでの Proto 定義をすべて YAML に書き直し
│     → チームの Proto 知見が活かせない
│
├── ❌ ゼロからの構築コスト
│     → 新規学習コスト
│     → 既存システムとの統合コスト
│
├── ❌ YAML 管理の複雑さ
│     → 大規模な API 定義の保守性
│
└── ❌ 時代潮流との不一致
      → gRPC/RPC が主流になりつつある
      → マイクロサービス化を見据えた場合の選択肢
```

**最も大きな理由は「既存 Proto 資産の廃棄」**でした。せっかく Proto を使っているのに、それを捨てて OpenAPI に移行するのは合理的ではないと判断しました。

### 選択肢2: gRPC (従来型)

**検討した理由:**

Proto を活かしながら RPC の恩恵を得るには、gRPC が最も標準的な選択肢です。

- 既存 Proto 定義をそのまま活用
- 高性能な通信（HTTP/2、ストリーミング対応）
- 豊富なエコシステム

**懸念点:**

しかし、従来型 gRPC には以下の懸念がありました。

```
gRPC の懸念点:
├── Envoy などのプロキシが必要
│   → インフラ複雑度の増加
│   → 運用コストの増加
│
├── HTTP/2 必須
│   → ブラウザからの直接呼び出しが困難
│   → gRPC-Web のための追加実装が必要
│
└── CDN・Proxy との相性
    → 既存インフラとの統合課題
```

gRPC の強力な機能は魅力的でしたが、インフラの複雑度増加と運用コストを考慮すると、現時点では過剰な選択に思えました。

### 選択肢3: Connect RPC（採用決定）

**Connect RPC とは:**

[Connect](https://connectrpc.com/) は、Buf 社が開発した Protocol Buffers ベースの RPC フレームワークです。gRPC 互換でありながら、より柔軟な通信方式をサポートします。

**採用を決めた理由:**

Connect RPC は、我々の課題を解決する理想的な選択肢でした。

```
Connect RPC の優位性:
├── ✅ 既存 Proto 資産をそのまま活用
│     → Proto 定義を書き直す必要なし
│     → チームの Proto 知見が活かせる
│
├── ✅ gRPC 互換性を保ちながら柔軟
│     → 将来的に完全な gRPC に移行可能
│     → 選択肢を狭めない
│
├── ✅ Envoy 不要
│     → インフラ複雑度を抑えられる
│     → 運用コストの削減
│
├── ✅ HTTP/1.1 対応
│     → ブラウザから直接呼び出し可能
│     → CDN・Proxy との相性良好
│
├── ✅ 信頼できる採用実績
│     → LayerX など国内企業の採用事例
│     → Buf 社の開発・サポート体制
│
└── ✅ スキーマ駆動開発の実現
      → Proto からのコード自動生成
      → クライアント・サーバー自動同期
```

**決定的だったポイント:**

「**既存 Proto を活かしながら、RPC の恩恵を得られる**」という点が決定的でした。OpenAPI のように資産を捨てることなく、gRPC のような複雑なインフラ導入も不要。既存の Proto 知見を活かしつつ、スキーマ駆動開発を実現できる。

Connect RPC は、我々にとって「最小の変更で最大の効果」を得られる選択肢でした。

## Connect RPC 導入による具体的な改善

### Before: REST + Proto（レスポンスのみ）

従来の REST API 実装では、以下のような手動実装が必要でした。

```go
// 従来の REST API 実装
func (c *V1ChapterController) RegisterUserRating(ctx echo.Context) error {
    // 1. リクエスト解析（手動）
    request := context.MustGetRequestDataFrom[RegisterUserRatingRequestData](ctx)
    u := context.GetUserContext(ctx)

    // 2. バリデーション（手動）
    displayType, err := valueobject.NewEpisodeDisplayType(request.DisplayType)
    if err != nil {
        return errors.NewBadRequestError(errors.WithMessage("不正な表示タイプです。"))
    }

    // 3. ビジネスロジック
    err = c.useCase.Chapter().RegisterUserRatingUseCase(
        ctx.Request().Context(), u.MembershipId, request.EpisodeId,
        request.RatingValue, displayType,
    )
    if err != nil {
        return failure.Wrap(err)
    }

    // 4. レスポンス生成（手動 Protobuf 変換）
    return c.Controller.Respond(ctx, &pivotapigo.RegisterUserRatingResponse{})
}
```

**課題:**
- リクエスト解析が手動
- バリデーションロジックが散在
- Proto はレスポンスのみ（リクエストは JSON）
- 型安全性が不完全

### After: Connect RPC（Proto をフル活用）

Connect RPC 導入後は、通信層の実装がシンプルになりました。

```go
// Connect RPC 実装（仮想コード）
func (s *ChapterService) RegisterUserRating(
    ctx context.Context,
    req *connect.Request[chapterv1.RegisterUserRatingRequest],
) (*connect.Response[chapterv1.RegisterUserRatingResponse], error) {
    // 1. 認証は interceptor で自動処理
    userCtx := usercontext.GetUserContextFromContext(ctx)

    // 2. リクエストは Proto で型安全
    //    バリデーションは Proto の enum/required で保証
    displayType := req.Msg.DisplayType // enum 型で型安全

    // 3. ビジネスロジックに集中
    err := s.chapterUseCase.RegisterUserRating(
        ctx,
        userCtx.MembershipId,
        req.Msg.EpisodeId,
        req.Msg.RatingValue,
        displayType,
    )
    if err != nil {
        return nil, connect.NewError(connect.CodeInternal, err)
    }

    // 4. レスポンス構築（自動シリアライゼーション）
    return connect.NewResponse(&chapterv1.RegisterUserRatingResponse{}), nil
}
```

### Proto による恩恵

Connect RPC 導入により、Proto の持つポテンシャルをフル活用できるようになりました。

**1. リクエスト/レスポンス両方で型安全**

```protobuf
// Proto 定義
message RegisterUserRatingRequest {
  string episode_id = 1;
  int32 rating_value = 2;
  EpisodeDisplayType display_type = 3; // enum で型安全
}

enum EpisodeDisplayType {
  EPISODE_DISPLAY_TYPE_UNSPECIFIED = 0;
  EPISODE_DISPLAY_TYPE_LIST = 1;
  EPISODE_DISPLAY_TYPE_GRID = 2;
}
```

Proto 定義から自動生成されたコードにより、リクエストもレスポンスも完全な型安全性が保証されます。

**2. 自動コード生成による効率化**

```bash
# buf.gen.yaml の設定
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
```

一つの Proto 定義から、Go（サーバー）、TypeScript（クライアント）のコードを自動生成。**手動同期作業が不要**になりました。

**3. クライアント同期の自動化**

Proto 定義を変更すると、コード生成により自動的にクライアント・サーバーのコードが同期されます。

```
Proto 定義変更
    ↓
buf generate 実行
    ↓
├── Go コード自動生成（サーバー）
└── TypeScript コード自動生成（クライアント）
    ↓
コンパイル時に型チェック
    ↓
通信の不整合を事前に検出
```

**型の不整合は、実行時ではなくコンパイル時に検出**されるため、バグの混入を防げます。

### スキーマ駆動開発の実現

Connect RPC により、真のスキーマ駆動開発が実現しました。

**開発フロー:**

1. Proto 定義を作成（スキーマファースト）
2. `buf generate` でコード自動生成
3. 生成されたインターフェースを実装
4. コンパイル時に型安全性を保証

**既存の Proto 知見を活かせる:**

- Proto の書き方は既存の知見をそのまま活用
- 新しい記法や仕様を学ぶ必要なし
- チームの学習コストを最小化

**REST + Proto（レスポンスのみ）** から **Connect RPC（Proto フル活用）** への移行により、Proto の持つポテンシャルを最大限に引き出すことができました。

## 移行戦略と実装

### 段階的移行の方針

一気に全システムを書き換えるのではなく、**段階的に移行する**方針を取りました。

```
新機能・大きな変更:
├── Connect RPC で実装
├── 最新のアーキテクチャを活用
└── スキーマ駆動開発の実践

既存機能:
├── 当面は REST API 維持
├── 安定稼働中の機能はリスクを取らない
└── 大幅な変更時に Connect RPC 移行を検討
```

**この方針の利点:**

- **リスク最小化**: 既存システムへの影響を抑える
- **学習曲線の緩和**: チームが Connect RPC に慣れる時間を確保
- **段階的な改善**: 新機能から実績を積み、既存機能へ展開

### Proto 定義は共通資産として継続活用

重要なのは、**Proto 定義は既存・新規で共通の資産**として扱うことです。

```
Proto 定義（共通資産）
    ↓
├── 既存 REST API → レスポンスで利用
└── 新規 Connect RPC → リクエスト/レスポンスで利用
```

既存の Proto 定義を捨てることなく、Connect RPC で活用範囲を拡大する形です。

### 実装時のポイント

**1. buf を活用したコード生成**

[Buf](https://buf.build/) は、Proto のビルド・リント・コード生成を統合管理するツールです。Connect RPC は Buf 社が開発しているため、相性が抜群です。

```yaml
# buf.yaml
version: v2
modules:
  - path: proto
lint:
  use:
    - STANDARD
breaking:
  use:
    - FILE
```

```yaml
# buf.gen.yaml
version: v2
plugins:
  - remote: buf.build/connectrpc/go
    out: gen/go
    opt:
      - paths=source_relative
  - remote: buf.build/protocolbuffers/go
    out: gen/go
    opt:
      - paths=source_relative
```

**2. interceptor による認証実装**

Connect RPC では、interceptor で横断的な関心事（認証・ログ・エラーハンドリングなど）を実装できます。

```go
// 認証 interceptor（簡略版）
func AuthInterceptor() connect.UnaryInterceptorFunc {
    return func(next connect.UnaryFunc) connect.UnaryFunc {
        return func(ctx context.Context, req connect.AnyRequest) (connect.AnyResponse, error) {
            // トークンから認証情報を取得
            token := req.Header().Get("Authorization")
            userCtx, err := authenticateToken(token)
            if err != nil {
                return nil, connect.NewError(connect.CodeUnauthenticated, err)
            }

            // コンテキストにユーザー情報を埋め込む
            ctx = usercontext.SetUserContext(ctx, userCtx)

            return next(ctx, req)
        }
    }
}
```

**3. エラーハンドリング**

Connect RPC は、gRPC のエラーコード体系をそのまま使えます。

```go
// エラーハンドリング例
func (s *ChapterService) GetChapter(
    ctx context.Context,
    req *connect.Request[chapterv1.GetChapterRequest],
) (*connect.Response[chapterv1.GetChapterResponse], error) {
    chapter, err := s.chapterUseCase.GetChapter(ctx, req.Msg.ChapterId)
    if err != nil {
        // ドメインエラーを Connect エラーに変換
        if errors.Is(err, domain.ErrNotFound) {
            return nil, connect.NewError(connect.CodeNotFound, err)
        }
        return nil, connect.NewError(connect.CodeInternal, err)
    }

    return connect.NewResponse(&chapterv1.GetChapterResponse{
        Chapter: toProtoChapter(chapter),
    }), nil
}
```

### 実装の感想

実際に実装してみて、以下の点が特に良かったです：

- **学習コストが低い**: gRPC の知見があればすぐに理解できる
- **既存コードとの共存が容易**: REST API と Connect RPC が同じサーバーで動作
- **開発体験が良い**: 型安全性により IDE の補完が効く、コンパイルエラーで不整合を検出

## 導入後の評価と今後

### 成功指標

Connect RPC 導入後、以下の成果が得られました。

**1. 開発効率の改善**

- 通信層の手動実装が不要になり、ビジネスロジックに集中できる
- API 変更時の手動同期作業が削減
- コード生成により、クライアント実装の工数削減

**2. 型安全性の向上**

- コンパイル時に通信の不整合を検出
- enum 型により、不正な値の混入を防止
- IDE の補完により、実装ミスの削減

**3. Proto 資産の有効活用**

- 既存の Proto 定義を活かしつつ、活用範囲を拡大
- チームの Proto 知見が無駄にならない
- 新しい技術を学ぶ際の摩擦が少ない

**4. アーキテクチャの柔軟性**

- 将来的な完全 gRPC 移行の選択肢を保持
- マイクロサービス化への準備が整う
- インフラ複雑度を抑えつつ、モダンな RPC を実現

### 今後の展望

**短期（~6ヶ月）:**

- Comment 機能の Connect RPC 実装完了
- Content 機能など、新機能は Connect RPC で開発
- チーム全体への Connect RPC 知見の共有

**中長期（6ヶ月~）:**

- 既存機能の大幅変更時に Connect RPC への移行検討
- Proto 中心の開発フローの確立
- ストリーミング API など、Connect RPC の高度な機能の活用

**将来的な可能性:**

- 完全な gRPC への移行（必要に応じて）
- マイクロサービスアーキテクチャへの移行
- Proto を中心とした API ガバナンス

## まとめ

### 「既存資産を活かす」技術選定の重要性

技術選定において、「最新の技術」や「流行の技術」を選ぶことが正解とは限りません。**既存資産を活かしながら、段階的に改善する**ことが、多くの場合で現実的な選択です。

今回の意思決定で重視したポイント：

- **既存 Proto 資産の有効活用**: 捨てずに活かす
- **段階的移行**: 一気に変えずに、リスクを抑える
- **将来の選択肢を狭めない**: gRPC 互換性を保持
- **インフラ複雑度の抑制**: Envoy 不要、HTTP/1.1 対応

### Proto を使っているなら Connect RPC は有力な選択肢

もしあなたのチームが：

- Protocol Buffers を使っているが、REST API のレスポンスだけに使っている
- スキーマ駆動開発に興味があるが、OpenAPI への移行コストが気になる
- gRPC に興味があるが、インフラの複雑度増加が懸念

という状況なら、**Connect RPC は非常に有力な選択肢**です。

既存の Proto 知見を活かしつつ、RPC の恩恵を得られる。学習コストを抑えながら、開発効率と型安全性を向上できる。Connect RPC は、そんな「いいとこどり」の技術です。

### 読者へのメッセージ

技術選定は、「何を選ぶか」よりも「なぜ選ぶか」の思考プロセスが重要です。

- 自分たちのコンテクスト（既存システム、チームのスキル、組織の状況）を理解する
- 複数の選択肢を真剣に比較検討する
- 「捨てる」よりも「活かす」選択を探す
- 段階的移行でリスクを最小化する

本記事が、同じような技術選定に悩む方の参考になれば幸いです。

---

**参考リンク:**

- [Connect RPC 公式サイト](https://connectrpc.com/)
- [Buf 公式サイト](https://buf.build/)
- [Connect RPC - Go クイックスタート](https://connectrpc.com/docs/go/getting-started)
- [LayerX の Connect RPC 導入事例](https://tech.layerx.co.jp/entry/2023/10/04/114155)

---
title: "ブログをGatsbyからAstroに移行してみた"
emoji: "🚀"
type: "tech"
topics:
  - "astro"
  - "gatsby"
  - "react"
published: true
published_at: "2024-03-14 00:00"
---

これまで自分のブログには[^my-blog]、静的サイトジェネレータの[Gatsby](https://www.gatsbyjs.com/)を使っていた。しかし、いくつかのツラミが増えてきたタイミングで、思い切って[Astro](https://astro.build/)に置き換えてみた。この記事では、移行に至った経緯と、実際に移行してみての所感をまとめている。

## Gatsbyを使っていたツラミ

### 画像の最適化によるビルド時間の長さ

Gatsbyでは、画像の最適化をビルド時に行っていた。これによりビルド時間が1時間を超えるようになってしまった。CDでキャッシュをうまく効かせればよかったのかもしれないが、うまくいっていなかった。Gatsby Cloudではぎりぎりうまくできていたが、サービスがなくなってしまった。結局、苦肉の策でローカルからビルドするしかなくなった。[^local-cache-benefit]

### カスタマイズの複雑さ

Gatsbyをカスタマイズするには、あちこちいじる必要があり、ハック的になっていた。もちろん、うまく使いこなす余地はあると思うが、ベストプラクティスを知らないだけかもしれない。Markdownファイルをローカルで管理していたという前提で、GraphQLで引いて表示するのだが、カテゴリー一覧やタグ一覧、それに紐づく記事一覧のページなどを増やしていった結果、おそらく無駄なデータの取り方をしたためか、見通しが悪くなり、ビルド時間の悪化にもつながってしまった。

## Astroに期待したこと

### ツラミのリセット

ゼロから作り直すことで、上記のツラミがスッキリするのではないかと期待した。Astroにしたことの効能でもないかもしれないが、いったんリセットされることを期待したのだ。

### アイランドアーキテクチャによる見通しの向上

Gatsbyでは、gatsby-nodeやgatsby-configをいじったりと、全体が関係あるので、結局全体で何をしているのかイマイチわかりづらくなっていた。一方、Astroだと関係ある設定や依存関係は基本的にファイルの中に書いてありそうな雰囲気を感じたので、うまくいけばシンプルになるのではないかと思った。[アイランドアーキテクチャ](https://docs.astro.build/ja/concepts/islands/)に期待していたのだ。

## Astroに移行した手間

実際にやってみたら、丸3日ほどで移行が完了した。Gatsbyでなんとか実装したタグ機能やカテゴリー機能なども、良さそうなテンプレートを使わせてもらいつつカスタマイズしたらできた。自分の開発に慣れてきたのはあるが、Astroについて何も知らない状態からみても、Gatsbyよりかなり簡単だった印象だ。

Markdownのfrontmatterの変換等（特に画像のpath周り）も微妙に必要だったが、その辺もNodeで適当にスクリプトを書いて変換した。これらも含めて、この程度の時間で終わった。

## Astroに実際に移行してみての所感

### アイランドアーキテクチャの良さ

たとえば、トップページのコードを見てみよう。

```astro
---
import { getCollection } from "astro:content";
import HomePagination from "@components/HomePagination.astro";
import Base from "@layouts/Base.astro";
import Posts from "@layouts/Posts.astro";

const posts = (await getCollection("entry")).sort(
 (a, b) => b.data.publishedDate.valueOf() - a.data.publishedDate.valueOf()
);
---
<Base>
 <div class="flex flex-col gap-12">
  <Posts posts={posts} />
  <HomePagination />
 </div>
</Base>
```

記事一覧はAstroのメソッド（getCollection）で取得でき、Astroファイルの上部で定義できる。それをTypeScriptで加工することもできる。`---`の下にJSXを書いて使えるので、ここで完結するため見通しがかなりいい。Gatsbyだと仰々しくGraphQLを書いたり色々しないといけないので、相対的に見通しは悪い印象だった。

### ビルド時間の短縮

Gatsbyのときは画像サイズも変換したりしていたので、よりリッチなことをしていたのは事実だ。しかし、Astroもwebpを生成したりはしている。それでも同じ記事・画像量でもビルド時間が10分次になり、CDを回せるようになった。運用上、これは大きい。もちろん、Gatsbyのビルドをチューニングすればできた気はするが。

## まとめ

厳密な比較は特にしていないし、同じメリットをGatsbyで作り直したり改善したりする形でもできるとは思う。また、使ったテンプレートによっても体感が全然変わってくる可能性はある。

しかし、Astroに乗り換えても特に問題なく、アイランドアーキテクチャにより、今後色々いじっていくにも前向きに楽しめそうな気がしている。

[^local-cache-benefit]: ローカルではそのままキャッシュが残ってインクリメンタルビルドになるのでなんとかなる。
[^my-blog]: 実際に移行したブログは https://blog.tawa.me/ で公開しています。

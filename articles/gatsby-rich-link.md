---
title: "Gatsby.jsでリンクをリッチな見た目にする"
emoji: "🔗"
type: "tech"
topics:
  - "gatsby"
  - "react"
  - "ogp"
  - "markdown"
published: true
published_at: "2023-09-19 14:25"
---

[Gatsby.js](https://www.gatsbyjs.com/) を使ってこのブログ[^my-blog]は作られている。デフォルトでは、一般的なブログサービスでできるようなリッチなリンクは生成できないので、テキストにしていた。

しかし、ようやくリッチなリンクを生成できるようにしたので、その方法を記録しておく。

この記事は、作業中に Zenn のスクラップに書いていたものをまとめたものである。

https://zenn.dev/tawachan/scraps/9793fc5abda9ed

## 実装方針の確認

具体的な実装方法をまとめる前に、どういう方針で実装したかを整理しておく。

### 前提条件

まず、前提条件として、次のようなことを想定している。

- [gatsby-source-filesystem](https://www.gatsbyjs.com/plugins/gatsby-source-filesystem/)を使ってローカルにブログ記事に相当する Markdown ファイルを配置している

### 実装方針

実装方針としては、次のようなものを想定している。

1. Markdown ファイルを全件調べて、リッチなリンクにする記法[^note1]を探し、該当 URL をすべて抽出する
2. その URL にリクエストし、必要な情報をすべて JSON にまとめ、静的ファイルとして配信する
3. 該当記法を独自のコンポーネントに置き換えるようにし、そのコンポーネントで JSON を読み込んでリッチな見た目を表示する

既存の記事だと、Gatsby のビルド時に組み込んでるものが見受けられたが、自分にとっては複雑そうだったので、JSON を配信して、クライアントサイドでレンダリングする方針にした。

調査時に参考にさせてもらった記事

https://kikunantoka.com/2020/04/10--install-rich-link/

https://bear-fruit.online/rich-internallink/

https://www.deg84.com/create-rich-link-component/

## 実装の手順

ここからは実際の実装の手順を記録していく。

### OGP のデータの取得

ローカルのスクリプトとして、OGP のデータを取得するスクリプトを作成する。これを適宜実行して生成された JSON ファイルをコミットしておく想定。

最終的なスクリプトはこちら。静的ファイルとして配信するために、`./static/ogp-data.json`としてファイルを出力している。

```js
const fs = require("fs");
const axios = require("axios");
const cheerio = require("cheerio");

const getExistingData = () => {
  const existingData = JSON.parse(fs.readFileSync("./static/ogp-data.json", "utf8"));
  return existingData;
};

const listFiles = (dir) => fs.readdirSync(dir, { withFileTypes: true }).flatMap((dirent) => (dirent.isFile() ? [`${dir}/${dirent.name}`] : listFiles(`${dir}/${dirent.name}`)));

const extractUrls = (fileTexts) => {
  const allUrls = fileTexts.flatMap((fileText) => {
    const lines = fileText.split("\n");

    const urls = lines
      .filter((line) => line.includes("<rich-link"))
      .map((line) => {
        const $ = cheerio.load(line);
        return $("rich-link").attr("href");
      })
      .filter((url) => url !== undefined);
    return urls;
  });
  const uniqueUrls = Array.from(new Set(allUrls));
  return uniqueUrls;
};

const getBlogTexts = (blogPosts) =>
  blogPosts
    .map((path) => {
      const file = fs.readFileSync(path, "utf8");
      return file;
    })
    .filter((text) => text !== "");

const fetchOgpData = async (url) => {
  try {
    const res = await axios.get(url); // axiosを使ってリクエスト
    const $ = cheerio.load(res.data); // 結果をcheerioでパース

    const getMetaContent = (property, name) => {
      return $(`meta[property='${property}']`).attr("content") || $(`meta[name='${name}']`).attr("content") || "";
    };

    const data = {
      originalUrl: url,
      url: getMetaContent("og:url", "") || res.request.res.responseUrl || url,
      domain: new URL(url).hostname,
      title: getMetaContent("og:title", "") || $("title").text() || "",
      description: getMetaContent("og:description", "description") || "",
      image: getMetaContent("og:image", "image") || "",
    };

    data.domain = new URL(data.url).hostname;

    return data;
  } catch (e) {
    console.log("failed to fetch ogp data", url);
    return null;
  }
};

const writeUrlsToFile = (urlData) => {
  fs.writeFileSync("./static/ogp-data.json", JSON.stringify(urlData));
};

const main = async () => {
  const blogPosts = [...listFiles("content/blog"), ...listFiles("content/hatena")];
  console.log("start: get blog texts");
  const fileTexts = getBlogTexts(blogPosts);
  console.log("end: get blog texts");

  console.log("start: extract urls", "files: " + fileTexts.length);
  const urls = extractUrls(fileTexts);
  const existingData = getExistingData();
  const newUrls = urls.filter((url) => !(existingData[url] !== undefined));

  console.log("end: extract urls", "urls: " + urls.length, "newUrls: " + newUrls.length);

  console.log("start: fetch ogp data");
  const dataArray = (await Promise.all(urls.map((url) => fetchOgpData(url)))).filter((d) => d !== null);
  console.log("end: fetch ogp data");

  console.log("start: convert to json");
  const data = dataArray.reduce((prev, cur) => {
    prev[cur.originalUrl] = cur;
    return prev;
  }, existingData);
  console.log("end: convert to json");

  writeUrlsToFile(data);
};

main();
```

ディレクトリ構成等に合わせてカスタマイズするなどして、参照してほしい。

関数の軽い説明をする。

- `listFiles`: Markdown ファイルの一覧を取得する
- `extraUrls`: `<rich-link>`タグを探し、`href`の値を抽出する
- `fetchOgpData`: OGP のデータを取得する

あとは、すでに JSON にある URL はスキップするなどして、無駄に再取得しないようにしている。

結果として、次のような JSON が出力される。

```json
{
  "https://blog.tawa.me/entry/nextjs-basic-auth": {
    "originalUrl": "https://blog.tawa.me/entry/nextjs-basic-auth",
    "url": "https://blog.tawa.me/entry/nextjs-basic-auth",
    "domain": "blog.tawa.me",
    "title": "Next.jsでベーシック認証を実装する方法 | 飽き性の頭の中",
    "description": "Next.jsアプリケーションにベーシック認証を実装する方法を紹介します。middleware.tsファイルに認証処理を追加し、環境変数を使用して認証の有無を切り替えられるように設定します。認証が通らない場合は、/api/authにリクエストが送信され、そこで認証を行います。",
    "image": "https://blog.tawa.me/static/f5de683591ab30fb7f7e0772891ba365/65dbf/2023-05-08-22-25-55.png"
  },
  "https://amzn.to/46ffhmc": {
    "originalUrl": "https://amzn.to/46ffhmc",
    "url": "https://www.amazon.co.jp/gp/product/B096WBPCC1?ie=UTF8&th=1&linkCode=sl1&tag=pxbub0309-22&linkId=d15ee9433900d0cc9628962d9ebe5d08&language=ja_JP&ref_=as_li_ss_tl",
    "domain": "www.amazon.co.jp",
    "title": "Amazon.co.jp: 【Ringke】iPad スタンド タブレット スタンド 超薄型 縦置き 横置き 2Way 貼り付け パッドスタンド 角度調整可能 マルチアングル ポータブルスタンド キンドル 対応 Outstanding - Dark Gray : 家電＆カメラ",
    "description": "Amazon.co.jp: 【Ringke】iPad スタンド タブレット スタンド 超薄型 縦置き 横置き 2Way 貼り付け パッドスタンド 角度調整可能 マルチアングル ポータブルスタンド キンドル 対応 Outstanding - Dark Gray : 家電＆カメラ",
    "image": ""
  },
  "https://blog.tawa.me/entry/foldstand-tablet-mini": {
    "originalUrl": "https://blog.tawa.me/entry/foldstand-tablet-mini",
    "url": "https://blog.tawa.me/entry/foldstand-tablet-mini",
    "domain": "blog.tawa.me",
    "title": "FoldStand Tablet MiniをiPad mini 6に使ってみた | 飽き性の頭の中",
    "description": "MOFTが有名ですが、FoldStand Tablet miniにしてみました。",
    "image": "https://blog.tawa.me/static/b6fd2fe3c6e863b3e25d3bbaced1bd0f/0a659/2022-03-13-23-22-50.png"
  }
}
```

map になっているので URL を指定すれば一発でデータの有無が確認できる。

### 独自コンポーネントに置き換える

次に`<rich-link>`を独自コンポーネントに置き換える処理を追加する。

基本この辺りを参考にさせてもらった。

https://www.luku.work/gatsby-remark-component

まずは、`rehype-react`を使って[^note-rehype-react]、`htmlAst`から React に変換するようにする。そして、その変換処理に、独自コンポーネントに置き換える設定を追加する。

```tsx
import rehypeReact from "rehype-react";

const renderAst = new rehypeReact({
  createElement: React.createElement,
  components: { "rich-link": RichLink },
} as any).Compiler;

...中略

<PostBody itemProp="articleBody" className="post-body">
  {renderAst(post.htmlAst)}
</PostBody>

```

Graphql のところに、`html`だけでなく`htmlAst`も取得するように項目を追加するのを忘れずに。

そして、肝心の`RichLink`コンポーネントを作成する。

```tsx
import { AspectRatio, HStack, Heading, Image, Stack, Text, Link } from "@chakra-ui/react";
import React, { useMemo } from "react";
import { FC } from "react";
import { isValidUrl } from "../helpers/url";
import { useQuery } from "react-query";
import { useWidthLevel } from "../hooks/useWidthLevel";

type Props = {
  href: string;
};
export const RichLink: FC<Props> = ({ href }) => {
  const url = href;
  const isValid = isValidUrl(url);
  const defaultImageLink = "/default-web-thumbnail.jpg";

  const isSameDomain = useMemo(() => {
    if (!isValid) return false;
    // windowの有無を確認しないとSSR時にエラーになる
    if (typeof window === "undefined") return false;
    const currentDomain = window.location.hostname;
    const urlDomain = new URL(url).hostname;
    return currentDomain === urlDomain;
  }, [isValid]);

  const { isMobile } = useWidthLevel();

  const { data: allData } = useQuery(
    "ogp-data",
    async () => {
      const res = await fetch("/ogp-data.json");
      const data = await res.json();
      return data;
    },
    { enabled: isValid }
  );

  const data = useMemo(() => {
    if (!allData) return null;
    return allData[url];
  }, [allData, url]);

  if (!isValid) return null;

  if (!data) {
    return (
      <Link href={url} isExternal={!isSameDomain} textDecor="none !important">
        <HStack borderWidth="1px" borderRadius={12} p={0} h="140px" overflow="hidden">
          <Stack flex={1} overflow="hidden" p={6} spacing={3}>
            <Heading as="strong" fontSize="md" isTruncated>
              {url}
            </Heading>
          </Stack>
          <Stack w={isMobile ? "140px" : "250px"}>
            <AspectRatio ratio={isMobile ? 1 : 16 / 9} w="full" h="full">
              <Image src={defaultImageLink} alt={url} w="full" h="full" overflow="hidden" objectFit="cover" />
            </AspectRatio>
          </Stack>
        </HStack>
      </Link>
    );
  }

  return (
    <Link href={url} isExternal={!isSameDomain} textDecor="none !important">
      <HStack borderWidth="1px" borderRadius={12} p={0} h="140px" overflow="hidden">
        <Stack flex={1} overflow="hidden" p={6} spacing={3}>
          <Heading as="strong" fontSize="md" isTruncated>
            {data.title}
          </Heading>
          <Text fontSize="sm" m="0px !important" isTruncated color="gray.500">
            {data.description}
          </Text>
          <Text fontSize="xs" m="0px !important" color="gray.800" isTruncated>
            {isMobile ? data.domain : data.url}
          </Text>
        </Stack>
        <Stack w={isMobile ? "140px" : "250px"}>
          <AspectRatio ratio={isMobile ? 1 : 16 / 9} w="full" h="full">
            <Image src={data.image || defaultImageLink} alt={data.title} w="full" h="full" overflow="hidden" objectFit="cover" />
          </AspectRatio>
        </Stack>
      </HStack>
    </Link>
  );
};
```

結構余計な処理を色々入れているので適宜取捨選択してほしい。

- JSON は`react-query`で取得している
  - キャッシュされるはずなので、各コンポーネントごとに雑に JSON を取得する処理を書いている
- データが見つからない場合は、URL をそれっぽく表示している
- 見た目はあとは好みでよしなに設定されている

Zenn にあるとおり、右往左往しながらやっているので、説明や手順が抜けているところがあるかもしれないが、概ねこんな感じだったはず。

## まとめ

以上、Gatsby.js でリッチなリンクを生成する方法を記録した。

Gatsby.js でリンクがリッチになると物書きも捗る気がするので、文章を気軽に書いていきたい。

[^note1]: 特別なコンポーネントを識別するタグ。この記事では、`<rich-link>`というタグを使っている。
[^note-rehype-react]: 2023年9月19日現在の最新はv8だが、v8は破壊的変更があり、上記の記法では動かない。v8でうまく動かす方法が色々調べても分からなかったので、今回はv7で実装した。Zenn (2023) 「rehype-reactのv8での動作について」. [記事を読む](https://zenn.dev/link/comments/f9d6ecf1bbd3c2)
[^my-blog]: 実際のブログは https://blog.tawa.me/ で公開しています。
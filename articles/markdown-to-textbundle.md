---
title: "MarkdownからTextbundleに変換するスクリプトを作りました｜ObsidianからCraftへ移行"
emoji: "📝"
type: "tech"
topics:
  - "markdown"
  - "textbundle"
  - "obsidian"
  - "craft"
published: true
published_at: "2021-12-12 19:10"
---

![](/images/markdown-textbundle-hero.png)

こんにちは、久しぶりに少しコードを書いたのでその話です。

たまたま YouTube で見かけた Craft というメモアプリ（？）があります。

https://www.craft.do/

この半年くらいは Obsidian というローカルの Markdown ファイルを編集したりつなげたりできるシンプルなものを使っていました。

https://obsidian.md/

しかし、Craft のリッチな見た目に惹かれて試したいと思うようになった次第です。

この辺の経緯は別途書こうと思うのですが、今回はそのときに問題となったファイル形式の点についてです。この記事を訪れたということは読者の方も似たような状況なのではないでしょうか。

## 移行時の問題

Markdown がエクスポートできるサービスからであれば何であれ Craft にはインポートできるのですが、いくつか問題があったので挙げていきます。

### 画像が反映されない

Craft には Markdown と Textbundle の両方のフォーマットでインポートできるのですが、どうやら Markdown の場合は画像が反映されません。Markdown 上に記載されている画像のパスに実際に画像ファイルがあったとしても反映されないようでした。

別途 Textbundle でインポートしてみたところ、反映されることを確認したので、どうやら Textbundle という形式にする必要がありそうでした。

### Daily Note にならない

今回は解決できていないのですが、別途いつかできたらなと思っている点です。

Craft には Daily Note というページを作ることができます。カレンダー上に表示されたり少し特殊な扱いとなるページなのですが、どうもインポートしてきたファイルはどうしても Daily Note として認識させられないようです。

Obsidian でも日記のようなものを書いていましたので、できるなら Daily Note として認識させたいのですが、ファイル名を調整してもどうやらこの特殊な扱いになるには別のパラメーターがあるようで現状では方法がみつけられていません...。

## Markdown から Textbundle へ

というわけで前者の画像が反映されていない問題をいったん解決しました。こればかりは後回しにできない重大な問題なので、これができるか否かで Craft を本当に使うかが変わってきます。

### 既存のツールで変換できるのか

まずは既存のツールやスクリプトで変換できる方法はないのかと考えました。調べると、一度 Bear というアプリで読み込むと Textbundle でエクスポートできるという情報を見たので試しました。

https://bear.app/

しかし、ここでも同様の問題があって、Bear にインポートする段階で Markdown を読み込むわけですが、ここでも同じく画像は反映されないので、Textbundle が作れたとしてもそこに画像が含まれないのです...。

というわけでどうにかして Textbundle の形にする必要がありました...。

### 諦めて自作で Textbundle を作る

あんまり詳しくないですが、雑に調べたところ、Textbundle は Markdown ファイルと画像などその他のファイルを 1 つのフォルダに入れて扱う形式ということでした。

http://textbundle.org/

なので、規則的に画像と Markdown ファイルを再配置すれば、Textbundle として認識してくれるということ。結構単純そうだったので雑にコードを書いてみました。

https://github.com/tawachan/markdown-to-textbundle

こんな感じのコードが書いてあるだけですが、最新のものは GitHub のリポジトリを参照してください。

```javascript
import fs from "fs";
import path from "path";

const OUTPUTS_PATH = "./outputs";
const INPUTS_PATH = "./inputs";
const TEMPLATES_PATH = "./templates";

const ENV = {
  notion: process.env.NOTION === "1",
};

export const removeUniqueIdFromFileName = (fileName) => {
  // Notionは "〈物語〉シリーズ セカンドシーズン b537c73e8449496e9d4bbd7c8c570922" のように最後にIDが付いた名称になるのでこれを取り除く
  const fileNameArray = fileName.split(" ");
  fileNameArray.pop();
  return fileNameArray.join(" ");
};

const main = async () => {
  if (fs.existsSync(OUTPUTS_PATH)) {
    fs.rmSync(OUTPUTS_PATH, { force: true, recursive: true });
  }
  fs.mkdirSync(OUTPUTS_PATH);

  const fileNames = fs.readdirSync(INPUTS_PATH);
  const markdownFileNames = fileNames.filter((n) => n.includes(".md"));

  markdownFileNames.forEach((fileName) => {
    const filePath = path.join(INPUTS_PATH, fileName);
    const text = fs.readFileSync(filePath).toString();

    const assetLinks = [...text.matchAll(/!\[(.+?\.(jpg|png|jpeg|pdf))\]\(.+?\.(png|jpeg|jpg|pdf)\)/g)].map((r) => r[1]);

    let modifiedFileName = fileName.replace(".md", "");
    if (ENV.notion) {
      modifiedFileName = removeUniqueIdFromFileName(modifiedFileName);
    }

    const outputFolderPath = path.join(OUTPUTS_PATH, modifiedFileName);

    fs.mkdirSync(outputFolderPath);

    const inputInfoFilePath = path.join(TEMPLATES_PATH, "info.json");
    const outputInfoFilePath = path.join(outputFolderPath, "info.json");
    fs.copyFileSync(inputInfoFilePath, outputInfoFilePath);

    const outPutMarkdownFilePath = path.join(outputFolderPath, fileName);
    fs.copyFileSync(filePath, outPutMarkdownFilePath);

    assetLinks.forEach((link) => {
      const decodedLink = decodeURIComponent(link);
      const inputAssetPath = path.join(INPUTS_PATH, decodedLink);
      const outputAssetPath = path.join(outputFolderPath, decodedLink);
      const parsed = path.parse(outputAssetPath);
      if (!fs.existsSync(parsed.dir)) {
        fs.mkdirSync(parsed.dir, { recursive: true });
      }
      fs.copyFileSync(inputAssetPath, outputAssetPath);
    });
  });
};

main();
```

雑に書いただけなので、メンテナンスできる状態とかにはなっていないのですが、クローンして node が入っていれば実行できるかと思います。

コード書けない人でも使えるようななにかにいずれまとめられたらよいなと思っていますが、なかなか時間が取れないので後回しでしょう...。

## 最後に

結構シンプルなのに、いい感じにコードも GitHub 上に公開されていなかったので、もしよかったら使ったり参考にしてみてください。

雑な情報共有なので、質問等があればコメントか Twitter でリプとか飛ばしてもらえれば幸いです。
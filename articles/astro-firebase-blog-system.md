---
title: "Astro + Firebase HostingでつくるブログシステムとCI/CDパイプライン"
emoji: "🔧"
type: "tech"
topics:
  - "astro"
  - "firebase"
  - "githubactions"
  - "blog"
published: true
---

個人ブログを[Astro](https://astro.build/)で構築し、[Firebase Hosting](https://firebase.google.com/docs/hosting)でホスティングする運用を続けている。GitHub Actionsを使ったCI/CDパイプラインも含めて、実際の構成とデプロイフローを紹介する。

## システム構成

### 技術スタック

- **フレームワーク**: Astro 5.x
- **スタイリング**: TailwindCSS
- **ホスティング**: Firebase Hosting
- **CI/CD**: GitHub Actions
- **パッケージマネージャー**: Yarn

### なぜこの構成にしたか

Astroを選んだ理由は、以前GatsbyからAstroに移行した[^gatsby-migration]際に感じた開発体験の良さだ。アイランドアーキテクチャにより見通しが良く、ビルド時間も大幅に短縮された。

Firebase Hostingは、静的サイトのホスティングとしてシンプルで、CDNも自動で提供される。GitHub Actionsとの連携も公式サポートされているため、運用が楽だ。

## CI/CDパイプライン

### 1. 品質チェック（CI）

mainブランチへのpush・pull request時に実行される：

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v5
      - name: Setup Node.js
        uses: actions/setup-node@v5
        with:
          node-version: 22
          cache: 'yarn'
      - name: Cache .cache directory
        uses: actions/cache@v4
        with:
          path: .cache
          key: ${{ runner.os }}-.cache
      - name: Install dependencies
        run: yarn install --frozen-lockfile
      - name: Format check
        run: yarn format:ci
      - name: Type check
        run: yarn astro check
```

コード品質チェックには[Biome](https://biomejs.dev/)を使用している。ESLintやPrettierより高速で、単一ツールでリント・フォーマットが完結する。

### 2. 本番デプロイ

mainブランチへのpush時に自動で本番環境にデプロイされる：

```yaml
name: Deploy to Firebase Hosting
on:
  push:
    branches:
      - main
  workflow_dispatch:
jobs:
  build_and_deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v5
      - name: Setup Node.js
        uses: actions/setup-node@v5
        with:
          node-version: 22
          cache: 'yarn'
      - name: Cache .cache directory
        uses: actions/cache@v4
        with:
          path: .cache
          key: ${{ runner.os }}-.cache
      - name: Install dependencies
        run: yarn install --frozen-lockfile
      - name: Build Astro site
        run: yarn build
      - name: Deploy to Firebase Hosting
        uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: "${{ secrets.GITHUB_TOKEN }}"
          firebaseServiceAccount: "${{ secrets.FIREBASE_SERVICE_ACCOUNT_BLOG_TAWA_ME }}"
          channelId: live
          projectId: blog-tawa-me
```

### Astroビルドの特徴

Astroのビルドプロセスは軽量で高速だ。以下のような最適化が自動で行われる：

```javascript
// astro.config.mjs
export default defineConfig({
  site: 'https://blog.tawa.me',
  output: 'static',
  trailingSlash: 'never',
  integrations: [
    sitemap(),
    tailwind(),
    mdx({
      syntaxHighlight: 'shiki',
      shikiConfig: { theme: 'material-theme-palenight' }
    }),
    compress({ CSS: true, HTML: true, Image: false, JavaScript: true, SVG: true }),
    VitePWA({
      registerType: 'autoUpdate',
      workbox: { globPatterns: ['**/*.{js,css,html,ico,png,svg}'] }
    })
  ]
});
```

- Gzip・Brotli圧縮
- サイトマップ・robots.txt自動生成
- PWA対応
- シンタックスハイライト（Shiki）

## デプロイフロー

実際の運用では以下のようなフローで記事を公開している：

1. **ローカルで記事作成** - Markdownで記事を執筆
2. **ブランチ作成** - `git checkout -b add-new-article`
3. **プッシュ** - CIが動いて品質チェック
4. **プルリクエスト** - レビュー・確認
5. **mainマージ** - 自動で本番デプロイ

### Firebase Hostingの設定

```json
{
  "hosting": {
    "public": "dist",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "trailingSlash": false
  }
}
```

Astroの出力先（`dist/`）とFirebaseの公開ディレクトリが一致するよう設定している。

## 運用上のメリット

### ビルド時間の短縮

Gatsbyから移行して最も実感したのは、ビルド時間の短縮だ。同じ記事数・画像数でも10分程度でビルドが完了し、CIを安定して回せるようになった。

### 開発体験の向上

Astroのアイランドアーキテクチャにより、各ページで必要なロジックが完結している。例えばトップページは以下のようにシンプルに書ける：

```astro
---
import { getCollection } from "astro:content";
import Base from "@layouts/Base.astro";
import Posts from "@layouts/Posts.astro";

const posts = (await getCollection("entry")).sort(
  (a, b) => b.data.publishedDate.valueOf() - a.data.publishedDate.valueOf()
);
---
<Base>
  <Posts posts={posts} />
</Base>
```

### 運用コストの低さ

Firebase Hostingは無料枠が潤沢で、個人ブログレベルの通信量であれば課金は発生しない。またCDNも自動で提供されるため、パフォーマンスチューニングの手間が少ない。

## まとめ

Astro + Firebase Hostingの組み合わせは、個人ブログにとって理想的な構成だと感じている。開発体験が良く、ビルドも高速で、運用コストも低い。GitHub Actionsとの連携により、記事を書いてpushするだけで自動デプロイされる環境が構築できた。

静的サイトジェネレータの選択肢が多い中で、Astroは特にブログ用途では扱いやすく、今後も使い続けていく予定だ。

[^gatsby-migration]: 詳しくは[ブログをGatsbyからAstroに移行してみた](https://zenn.dev/tawachan/articles/gatsby-to-astro)を参照

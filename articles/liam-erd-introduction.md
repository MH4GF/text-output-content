---
title: "Liam ERDで綺麗でインタラクティブなER図を自動生成する"
emoji: "👾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["database", "er図", "oss", "LiamERD"]
published: true
---

この記事は、Liam ERDのブログ [Introducing Liam ERD](https://liambx.com/blog/liam-erd-introduction) からの翻訳記事です。

---

私たちは **Liam ERD** というデータベース設計のための新しいツールを開発しており、ついこの度リリースしました！その紹介をさせてください。

![Liam ERD Cover](https://storage.googleapis.com/zenn-user-upload/0cbbc2b6fa71-20250122.png)

https://liambx.com/

## TL;DR

- データベースのテーブル構造を可視化する ER 図を自動生成するツール Liam ERD をリリースしました
- **Web 版**: パブリックリポジトリの場合は [https://liambx.com/erd/p/github.com/mastodon/mastodon/blob/main/db/schema.rb](https://liambx.com/erd/p/github.com/mastodon/mastodon/blob/main/db/schema.rb) ですぐに試せます
- **CLI 版**: プライベートリポジトリ用として、Prisma + GitHub Actions + Cloudflare Pages のデプロイ方法も紹介

## なぜLiam ERDを作ったか

ソフトウェア開発において、ER 図 (Entity Relationship Diagrams) はデータベース構造の可視化と共有を簡素化し、コミュニケーションを円滑にできます。新規参加メンバーのキャッチアップコストを削減したり、PdM やカスタマーサポート等非エンジニア職への説明に利用されたり、データ分析チームがプロダクトコードを読まずにテーブル構造を理解するのに役立ちます。

ドキュメントとして ER 図は用意できると便利なのですが、スプレッドシートやダイアグラム作図ツールなどで手動で更新するのは大変な労力がかかり、更新漏れや記載ミスが懸念となります。そのためプロジェクトのリポジトリにコミットしているスキーマファイルや、データベースに接続し手に入るメタデータから**ER 図を自動生成できると望ましいです**。

ER 図の自動生成ツールはすでにいくつか存在しており、例えば Mermaid.js や PlantUML などを利用したツールは画像ファイル形式で生成しますが、固定された画像では大規模で複雑化したプロジェクトとなるとどうしても見づらいです。また SchemaSpy など HTML 形式で出力可能なツールもあるものの、依存するランタイムやミドルウェアが多く CI/CD パイプラインとの統合が難しいことも多いです。

CI/CD フレンドリーで簡単にセットアップできて、かつ可読性が高く把握しやすい ER 図自動生成ツールが欲しい！ということで、 `Liam ERD` を開発しました。

## Liam ERD の特徴

Liam ERD の特徴は以下の 4 点です。

- **モダンでインタラクティブな UI**: パンやズーム、フィルタリングやフォーカスなどインタラクティブな操作ができる
- **ハイパフォーマンス**: 100 テーブルを表示しても快適に動作可能 テーブルのフィルタリングも高速
- **CI/CD フレンドリー**: 簡単にセットアップしデプロイ可能 多くのホスティングサービスでの動作を確認
- **オープンソース&コミュニティ主導の開発**: コードを自由に改変可能 & コミュニティのフィードバックを受けて機能追加を進めていく

スクリーンショットを交えつつ具体的な機能を見ていきましょう！

![初期表示のスクリーンショット](https://storage.googleapis.com/zenn-user-upload/c1436c4639c1-20250122.jpg)

Liam ERD は名前が近いテーブルや関連のあるテーブルを近くに配置し、カーディナリティの線が複雑にならないよう位置を制御するなど、レイアウトの見やすさにこだわっています。大規模なテーブル構造でも初期表示の時点でかなり見やすいことがわかります。

![ホバーにより関連するテーブルやカラムをハイライト](https://storage.googleapis.com/zenn-user-upload/159b97aa21a1-20250122.jpg)

とはいえ大規模になるとどうしても把握しづらいものです。
Liam ERD でテーブルをホバーすると、関連するテーブルやカラムをハイライトして表示します。これで知りたいテーブルをざっくり探していくことができます。

![テーブルの詳細表示ペイン](https://storage.googleapis.com/zenn-user-upload/ee3443d3ec42-20250122.jpg)

テーブルを選択すると右から詳細表示ペインが出てきて、テーブルやカラムのコメントや、設定されているインデックス、そして関連するテーブルだけに絞った ER 図が表示されます。

他にもテーブル構造の把握がしやすくなるような細かい体験が盛り込まれています。
以下のリンク先にオープンソースの SNS プラットフォーム Mastodon の ER 図があるので、こちらも触ってみてください。Mastodon は現在 100 テーブルほどの規模となっています。

https://liambx.com/erd/p/github.com/mastodon/mastodon/blob/main/db/schema.rb

## パブリックリポジトリでの利用方法

パブリックな GitHub リポジトリならかなり簡単に運用できます。
スキーマファイルの URL の先頭に `https://liambx.com/erd/p/` をつけるだけで ER 図をレンダリングできます。
これで main ブランチのスキーマをもとに生成された ER 図を閲覧できます。もちろん `main` をコミットハッシュに変えればその時点での ER 図をレンダリングすることも可能です。

## 実例: Prisma + GitHub Actions + Cloudflare Pages

というのは最も簡単な利用方法ですが、きっとあなたは「これだと業務のプロジェクトで使えない」と思ったことでしょう。全くその通りです！  
Liam ERD は npm パッケージの CLI ツールとしても提供されており、ローカル環境や GitHub Actions で ER 図を生成し簡単にホスティングすることができます。

https://www.npmjs.com/package/@liam-hq/cli

実践的な例として、Prisma を利用しているプロジェクトで GitHub Actions と Cloudflare Pages を組み合わせてデプロイする方法を紹介します。
Cloudflare Pages を選択した理由は、Cloudflare Access という機能が便利で、例えば社内メンバーのみアクセス可能にするといった権限制御が簡単に実装できるためです。

まず、以下のような schema.prisma を用意します。

```hcl
// This is your Prisma schema file
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

// ユーザーテーブル
model User {
  id             Int           @id @default(autoincrement())
  email          String        @unique
  username       String
  password       String
  createdAt      DateTime      @default(now())
  updatedAt      DateTime      @updatedAt
  profile        Profile?
  posts          Post[]
  comments       Comment[]
  orders         Order[]
  notifications  Notification[]
}

// プロフィールテーブル
model Profile {
  id          Int      @id @default(autoincrement())
  userId      Int      @unique
  firstName   String
  lastName    String
  bio         String?
  avatar      String?
  birthDate   DateTime?
  phoneNumber String?
  user        User     @relation(fields: [userId], references: [id])
}

// 投稿テーブル
model Post {
  id        Int       @id @default(autoincrement())
  title     String
  content   String
  published Boolean   @default(false)
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  authorId  Int
  author    User      @relation(fields: [authorId], references: [id])
  comments  Comment[]
}

// コメントテーブル
model Comment {
  id        Int      @id @default(autoincrement())
  content   String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  postId    Int
  authorId  Int
  post      Post     @relation(fields: [postId], references: [id])
  author    User     @relation(fields: [authorId], references: [id])
}

// 商品テーブル
model Product {
  id          Int      @id @default(autoincrement())
  name        String
  description String
  price       Decimal
  stock       Int
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  orderItems  OrderItem[]
  categoryId  Int
  category    Category @relation(fields: [categoryId], references: [id])
}

// カテゴリーテーブル
model Category {
  id       Int       @id @default(autoincrement())
  name     String    @unique
  products Product[]
}

// 注文テーブル
model Order {
  id         Int         @id @default(autoincrement())
  userId     Int
  status     OrderStatus @default(PENDING)
  totalPrice Decimal
  createdAt  DateTime    @default(now())
  updatedAt  DateTime    @updatedAt
  user       User        @relation(fields: [userId], references: [id])
  orderItems OrderItem[]
}

// 注文詳細テーブル
model OrderItem {
  id        Int     @id @default(autoincrement())
  orderId   Int
  productId Int
  quantity  Int
  price     Decimal
  order     Order   @relation(fields: [orderId], references: [id])
  product   Product @relation(fields: [productId], references: [id])
}

// 通知テーブル
model Notification {
  id        Int      @id @default(autoincrement())
  userId    Int
  title     String
  content   String
  read      Boolean  @default(false)
  createdAt DateTime @default(now())
  user      User     @relation(fields: [userId], references: [id])
}

enum OrderStatus {
  PENDING
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
}
```

そして Cloudflare Pages のプロジェクトを CLI で作成します。

```sh
wrangler pages project create prisma-with-cloudflare-pages
```

続いて GitHub Actions 用の以下のワークフローファイルを用意します。

```yaml
name: prisma-with-cloudflare-pages
on:
  push:
    branches:
      - main
    paths:
      - prisma/schema.prisma

jobs:
  build-and-deploy-erd:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      deployments: write

    steps:
      - uses: actions/checkout@v4
      - name: Generate ER Diagrams
        run: npx @liam-hq/cli erd build --input prisma/schema.prisma --format prisma
      - name: Deploy ERD to Cloudflare Pages
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          command: pages deploy ./dist --project-name=prisma-with-cloudflare-pages
          gitHubToken: ${{ secrets.GITHUB_TOKEN }}
```

これでデプロイができました！

https://prisma-with-cloudflare-pages.pages.dev/

![ER図の様子](https://storage.googleapis.com/zenn-user-upload/af072452c929-20250122.jpg)

Vite で React のプロジェクトをビルドしているだけなので、大半のホスティングサービスだったら簡単にデプロイできるかと思います。

今回の説明に利用したサンプルリポジトリはこちらです:

https://github.com/liam-hq/liam-erd-samples/tree/main/samples/prisma-with-cloudflare-pages

---

今は Ruby on Rails の schema.rb と Prisma の schema.prisma、また SQL の DDL をサポートしています。他の形式のサポートも検討していますし、プルリクエストもお待ちしています！

マニュアルに対応している ORM や RDBMS での利用方法も記載しているので、ぜひご覧ください。

https://liambx.com/docs/parser/supported-formats

## 今後の機能追加

Liam ERD は現在はテーブル構造の可視化の機能のみを提供していますが、今後は編集やマイグレーションなど、データベース設計に関わる様々な機能を追加していく予定です。

- **ドキュメンテーション強化**: 集約のグルーピング ([tbls の Viewpoints ](https://github.com/k1LoW/tbls?tab=readme-ov-file#viewpoints)のようなもの) とコメントの追加
- **ERD の編集**: テーブルやカラムの追加・編集と、その差分のマイグレーションファイルを生成
- **レコードの確認**: DB 接続の上簡単な SQL が実行でき、よりテーブルのキャッチアップをしやすく
- **クラウド版の提供**: チームで共有可能なプライベートプロジェクトや、リアルタイム共同編集機能の提供

もし気に入っていただけたら、[Liam の GitHub リポジトリ](https://github.com/liam-hq/liam)に ⭐️ をつけて応援してもらえると嬉しいです！  
ロードマップは以下で管理しているので、コメントやフィードバックをぜひお願いします！

https://github.com/orgs/liam-hq/projects/1/views/1

## おわりに

というわけで、見やすい ER 図を簡単に自動生成できるツール Liam ERD を紹介しました。ぜひ使ってみてください！

https://github.com/liam-hq/liam

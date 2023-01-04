---
title: "graphql-rubyで入力値の柔軟なバリデーションを実現する@constriant directiveを導入する"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [graphql,graphqlruby]
published: false
---

この記事は、GraphQLのInput Objectのバリデーションを実現する@constraint directiveの紹介と、graphql-rubyで実装する方法を紹介します。

---

GraphQLでは、スキーマに記述した型情報を利用したデータの検証がリクエスト・レスポンス双方で利用できます。例えば `title: String!` のようなフィールドを定義した場合、実際に渡されるtitleが文字列でない場合にエラーが返却されます。 `!` はnon-nullを表すため、nullが渡される場合もエラーになります。
しかし一般的なアプリケーションに求められるような、文字列の長さや数値の範囲を指定するといった複雑なバリデーションには標準では対応していません。

# よくある解決策

まず考えられるのは、サーバー側ロジックとしてリゾルバー内でバリデーションを実装し、違反があれば実行時エラーを返すことでしょう。ですがこの場合どのようなバリデーションが行われるかはスキーマを見ただけでは知ることができず、暗黙的です。またクライアント側でもフォーム入力時のUX向上を目的としてバリデーションを実装することは一般的ですので、そうするとサーバー・クライアント双方で別々にロジックが実装され、二重管理になってしまいます。

# カスタムディレクティブを利用した解決方法

この「スキーマでバリデーション内容を表現できず、サーバー・クライアント双方で別々にロジックが生まれてしまう」問題の解決策として、 `@constraint` といったカスタムディレクティブを用意する方法があります。以下のような見た目となります。

```graphql
input CreateBookInput {
  title: String! @constraint(minLength: 1, maxLength: 200)
  price: Int! @constraint(min: 0)
}
```

本の作成に必要なタイトルと価格には、それぞれ制限があることがひと目でわかります。
このカスタムディレクティブは[Constraints Directives RFC](https://github.com/IvanGoncharov/graphql-constraints-spec)として提案されていたものがベースとなっています。

また人間にとっての可読性だけでなく、ディレクティブとして表現することによりマシンリーダブルに情報を処理することが可能になり、以下のようなメリットがあります。

- クライアント側のバリデーションのための、zodやyupを利用したコードの生成ができる
- サーバー側のリゾルバー処理の前にミドルウェア的にバリデーション処理を挟むことで、リゾルバーでは値の扱いのみに集中できる

「titleに渡せる文字列は何文字までか」といったようなバリデーションの詳細についてはスキーマに記述されているため、サーバー・クライアントのロジックとして記述する必要がありません。バリデーションの内容を変更したくなってもスキーマだけを変更すれば良くなります。

次節以降でどのように実装するのかについて述べていきます。

# @constraintディレクティブからzodやyupのschemaを生成する

まずクライアント側の実装方法について紹介します。
フロントエンドでの入力値バリデーションを行う際はzodやyupといったライブラリを使うことが一般的です。またGraphQLをTypeScriptから利用する際には[GraphQL Code Generator](https://the-guild.dev/graphql/codegen)を使ってコードや型の生成をするのが大変便利で、そのプラグインである[Code-Hex/graphql-codegen-typescript-validation-schema](https://github.com/Code-Hex/graphql-codegen-typescript-validation-schema)を利用するとGraphQLのスキーマからzodやyupのスキーマを生成できます。詳しくはこちらの記事で紹介されています。

https://zenn.dev/codehex/articles/14fe2b8a87a59c

上部で定義したCreateBookInputがある状態でcodegenを実行すると、以下のようなコードを生成できます。こちらはzodの例です。

```typescript
export type CreateBookInput = {
  readonly title: Scalars['String'];
  readonly price: Scalars['Int'];
};

export function CreateBookInputSchema(): z.ZodObject<Properties<CreateBookInput>> {
  return z.object<Properties<ItemInput>>({
    title: z.string().min(1).max(200),
    price: z.number().min(0)
  })
}
```

生成されたzodのschemaを利用してミューテーション実行前にparseしたり、react-hook-formなどのフォームライブラリに統合することもでき、非常に便利です。

# サーバー側で@constraintディレクティブのバリデーションを実装する

サーバー側ではミューテーション実行時に@constraintディレクティブを解釈しバリデーションエラーを返す処理が必要になります。OpenAPI3ベースのスキーマ駆動開発を実践する際、[Committee](https://github.com/interagent/committee)を利用するのがイメージとして近いかと思います。
サーバーサイドの言語としてNode.jsを採用している場合はnpmライブラリとして提供されている[graphql-constraint-directive](https://github.com/confuser/graphql-constraint-directive)を利用するのが良いのですが、Rubyを採用する場合に利用できる同様のライブラリや実装は見つけられなかったため、今回作成しました。

https://github.com/MH4GF/graphql-ruby-constraint-directive

以下のようなschema, mutationが実装されていることを想定します。

```ruby
module Types
  class CheckType < Types::BaseObject
    implements GraphQL::Types::Relay::Node

    field :title, String, null: false
    field :price, Int, null: false
  end
end

module Mutations
  class CreateBook < BaseMutation
    argument :title, String, required: true
    argument :price, Int, required: true
    field :book, Types::BookType, null: false

    def resolve(title:, price:)
      # ...some codes
      { book: book }
    end
  end
end

module Types
  class MutationType < Types::BaseObject
    field :create_book, mutation: Mutations::CreateBook
  end
end

class MySchema < GraphQL::Schema
  mutation(Types::MutationType)
end
```

graphql-ruby-constraint-directive gemは[graphql-ruby](https://github.com/rmosolgo/graphql-ruby/)のプラグインとして実装されているため、以下のようにスキーマクラスで利用します。

```diff ruby
class MySchema < GraphQL::Schema
  mutation(Types::MutationType)
+ use GraphQL::Constraint::Directive  
end
```

そしてMutationで以下のようにdirectiveを設定します。^[`GraphQL::Constraint::Directive::Constraint`を毎回記述する必要があり少々冗長なため、より簡潔に記述できるAPIを検討中です。良い案があればぜひご連絡ください。]

```diff ruby
module Mutations
  class CreateBook < BaseMutation
-   argument :title, String, required: true
-   argument :price, Int, required: true
+   argument :title, String, required: true,
+             directives: { GraphQL::Constraint::Directive::Constraint => { min_length: 1, max_length: 200 } }
+   argument :price, Int, required: true,
+             directives: { GraphQL::Constraint::Directive::Constraint => { min: 0 } }
    field :book, Types::BookType, null: false

    def resolve(title:, price:)
      # ...some codes
      { book: book }
    end
  end
end
```

これだけでミューテーション実行時にバリデーション処理を実行できます。違反のある値が渡されてきた場合、例えば以下のようなエラーが返却されます。

```json
{
  "data": {
    "createBook": null
  },
  "errors": [
    {
      "message": "title is too short (minimum is 1)",
      "locations": [
        {
          "line": 2,
          "column": 3
        }
      ],
      "path": [
        "createBook"
      ]
    }
  ]
}
```

ミューテーションクラスのresolveメソッドには一切手を加えていないため、個々のミューテーションではバリデーションのことは気にせず値の処理にだけ集中できます。

# まとめ

本記事ではGraphQLで柔軟な入力値バリデーションを実現するための@constraintディレクティブと、graphql-rubyでバリデーション処理を実現するためのライブラリである[graphql-ruby-constraint-directive](https://github.com/MH4GF/graphql-ruby-constraint-directive)を紹介しました。
良いと思った方はリポジトリへのスターを押していただけると嬉しいです。バリデーションの種類はまだ少ないのもあり、必要な機能があればIssueやPull Requestもお待ちしています。

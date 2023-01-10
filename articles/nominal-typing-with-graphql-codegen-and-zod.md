---
title: "graphql-codegenとzodのz.brandでCustom ScalarのNominal Typingを実現する例"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [graphql,graphqlcodegen,zod]
published: true
---

GraphQLのスキーマで定義したCustom Scalarをgraphql-codegenでTypeScriptの型を生成する際に、zodを使ったNominal Typingの実現例を紹介します。
この記事の内容は以下の記事で記載されている内容をZodを使って実現します。Nominal Typingなどの事前知識については説明しないため、以下の記事を参照ください。

https://www.wantedly.com/companies/wantedly/post_articles/387161

# 前提

上記記事で定義したGraphQLのCustom Scalarを想定します。

```graphql
type Profile {
  birthday: Date
}

scalar Date
```

達成したいこととしては、graphql-codegenで生成するTypeScriptの型においてこのData Scalarを通常のstringとは別の型として定義し識別できるようにしたいです。
元記事ではTypeScriptの自前の型として `DateString` 型を用意し、codegen.ymlのscalarsに設定していました。以下引用です。

```typescript
type DateString = string & { __dateStringBrand: any  };

function parseDateString(dateStr: DateString) {
  // ...
}

// これはエラーとなり、日付文字列を他の文字列と区別できていることがわかる
const date = parseDateString("ぴよぴよ");
```

```yaml:codegen.yml
generates:
  # ...

  src/path/to/graphqlOperationTypes.ts:
    plugins:
      - typescript-operations
    config:
      # なんか便利なオプション色々...
      scalars:
        # ちょっと hacky な感じがして不安ではあるが…。
        Date: Types.DateString 
```

> 上記の設定を入れることで、graphql-codegenの生成結果でNominal TypingなDateStringが利用されるようになります。これで **「このbirthdayというフィールドはGraphQLクエリ結果から来た日付文字列である」というのが型として表現されている** という要求が実現できました。

# 実際の運用で考えること

このNominal TypingなCustom Scalarを実際のプロダクトで運用するとなると、考えておきたいことがいくつかあります。

- StoryBookやテストでCustom Scalarのデータ型を扱うために、モックデータを生成できるようにしておきたい
- Custom Scalarが増えて行くとすると、 `__dateStringBrand` のような命名は実装者によってブレてしまうのもあり、上記のモックデータ生成も含めてある程度汎用的な仕組みを用意しておきたい

これらの課題はスキーマバリデーションライブラリであるZodが提供する `z.brand` を使うと解決できます。本記事ではz.brandの使い方の簡単な説明と、grpahql-codegenと組み合わせて利用する例を紹介します。

# z.brandとは

詳しくはZodのドキュメントに記載してあります。
https://github.com/colinhacks/zod#brand

z.brandが解決する課題は元記事と同じく、構造的部分型を採用するTypeScriptでNominal Typingをシミュレートすることです。

```typescript
const Cat = z.object({ name: z.string() }).brand<"Cat">();
type Cat = z.infer<typeof Cat>;

const petCat = (cat: Cat) => {};

// これは動作する
const simba = Cat.parse({ name: "simba" });
petCat(simba);

// オブジェクトリテラルは構造として同じであってもCat型ではないため、コンパイルエラーになる
petCat({ name: "fido" });
```

実際の利用方法を見る方が理解しやすいため、早速z.brandをGraphQLのDate scalarを表すために利用してみます。

# DateString型の定義にz.brandを使ってみる

```diff typescript
-type DateString = string & { __dateStringBrand: any  };
+import { z } from 'zod'
+
+export const dateStringParser = z.string().brand<'DateString'>()
+export type DateString = z.infer<typeof dateStringParser>
```

Zodのスキーマ定義として `dateStringParser` 関数を定義し、z.inferで `DateString` 型を定義しました。
このDateString型は型情報を確認してみると以下のようになっています。

![](https://storage.googleapis.com/zenn-user-upload/98bf26c3d850-20230110.png)
```typescript
type DateString = string & z.BRAND<"DateString">
```

stringとの交差型としてz.BRANDが付与されています。z.BRANDの中身は `{[symbol]: "DateString"}` のようなオブジェクトです。
これにより単純なstringはDateString型に割り当てられなくなりました。

```typescript
function formatDate(date: DateString) {
  // ...
}

// これはエラーとなり、日付文字列を他の文字列と区別できていることがわかる
formatDate("ぴよぴよ");
```

# DateString型の値を生成する関数

これにより単純なstringはDateString型に割り当てられなくなりましたが、StoryBookやtest等でDateString型の値を生成したいことは考えられます。
z.brandを利用しZodのスキーマとして定義することでそれも実現が可能です。先ほど定義した `dateStringParser` を利用します。^[この例の場合`dateStringParser.parse("ぴよぴよ")`とすることもできてしまうので、日付の値を現実で扱う場合はTemplate Literal Typeを使うかunixtimeにする方が良いかもしれません。]

```typescript
const dateStr = dateStringParser.parse("2020-01-01")
// これはエラーにならない
formatDate(dateStr);
```

このようにZodのz.brandを使うことで、Nominal Typingな型の定義とその型を満たす値の生成が簡単にできることがわかります。

# 実践：graphql-codegen-typescript-mock-dataと組み合わせる

より実践的な例として、GraphQLスキーマからモックデータ生成関数を生成してくれる[graphql-codegen-typescript-mock-data](https://github.com/ardeois/graphql-codegen-typescript-mock-data)との組み合わせ例も紹介します。

このプラグインをcodegen.ymlで設定し、上述のGraphQLスキーマを渡すと、

```yaml:codegen.yml
generates:
  # ...

  src/path/to/builders.ts:
    plugins:
      - typescript-mock-data:
          typesFile: './types.generated.ts'
          scalars:
            ISO8601DateTime:
              generator: "'2020-01-01'"
```

```graphql
type Profile {
  birthday: Date
  name: String # 説明のために今回追加しています
}

scalar Date
```

以下のようなコードを生成してくれます。

```typescript:src/path/to/builders.ts
import { Profile } from './types.generated';

export const aProfile = (overrides?: Partial<Profile>): Profile => {
    return {
        birthday: overrides && overrides.hasOwnProperty('birthday') ? overrides.birthday! : '2020-01-01',
        name: overrides && overrides.hasOwnProperty('name') ? overrides.name! : 'consequatur',
    };
};
```

nameフィールドはString型ですが、生成されたコードではランダムなダミーデータが格納されています。graphql-codegen-typescript-mock-dataの依存として内部的にダミーデータ生成ライブラリである[faker](https://github.com/faker-js/faker)が使われています。

以下のように利用できます。

```typescript
/// このようにデータ生成ができて便利
const profile = aProfile()

// 値のオーバーライドもできる
const profile2 = aProfile({ birthday: '2022-01-30' })
```

---

codegen.ymlの設定にある `scalars` の `generator` はCustom Scalarの値の生成方法を指定しますが、現状では `'2020-01-01'` といった文字列が指定されています。前節でDate Scalarはz.brandを利用した `DateString` 型を指定することにしたため、現状では単純な文字列なためコンパイルエラーになってしまいます。
しかしこれは `dateStringParser` でパースするようにすれば解決できますね。試してみましょう。

```diff yaml:codegen.yml
generates:
  # ...

  src/path/to/builders.ts:
    plugins:
+     - add:
+         content: "import { dateStringParser } from './zod'"
      - typescript-mock-data:
          typesFile: './types.generated.ts'
          scalars:
            ISO8601DateTime:
-             generator: "'2020-01-01'"
+             generator: "dateStringParser.parse('2020-01-01')"
```

[graphql-codegenで生成するコードに任意の文字列を追加できるaddプラグイン](https://the-guild.dev/graphql/codegen/plugins/other/add)を利用してdateStringParserをimportし、generatorで参照するようにしました。
これによりモックデータ生成関数でもdateStringParserが使われるようになり、モックデータのコンパイルエラーがなくなりました。

```typescript:src/path/to/builders.ts
import { dateStringParser } from './zod'
import { Profile } from './types.generated';

export const aProfile = (overrides?: Partial<Profile>): Profile => {
    return {
        birthday: overrides && overrides.hasOwnProperty('birthday') ? overrides.birthday! : dateStringParser.parse('2020-01-01'),
        name: overrides && overrides.hasOwnProperty('name') ? overrides.name! : 'consequatur',
    };
};
```

# まとめ

この記事では、GraphQLのスキーマで定義したCustom Scalarをgraphql-codegenでTypeScriptの型を生成する際に、Zodを使ったNominal Typingの実現例を紹介しました。
型として表現できる情報を増やし、安全にプログラミングしていきたいですね。

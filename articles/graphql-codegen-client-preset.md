---
title: "GraphQL Code Generator v3 Roadmapで推されているclient-presetを紹介する"
emoji: "👀"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: [graphql, graphqlcodegen]
published: true
---

こんにちは。皆さんは[GraphQL Code Generator](https://the-guild.dev/graphql/codegen)を利用していますか？
筆者は普段React/TypeScript/Apollo Client(またはurql)といったスタックでWebフロントエンドを書いており、その際にはGraphQL Code Generatorをほぼ必需品と言えるほど愛用しています。
サーバー側から提供されたスキーマやクライアント側が必要なデータを宣言したオペレーションから型やコードを生成し利用することで、ロジックに関する実装量が大きく削減でき、ミスを減らすことにもつながります。GraphQLを使う理由の1つと言っても過言ではないでしょう。

そのGraphQL Code Generatorでは[v3 Roadmap](https://github.com/dotansimha/graphql-code-generator/issues/8296)として今後の方針が公開されており、[client-preset](https://the-guild.dev/graphql/codegen/plugins/presets/preset-client)という新しいプリセットが紹介されています。そこでは「**GraphQL Code Generator 3.0のclient-presetが、既存のhooksやSDKベースのプラグインをすべて置き換え、フロントエンドのユースケース向けにGraphQL Typeを生成する公式な方法となることを目指します**」と記載されています。これは一体どういうことなのでしょうか。

この記事ではGraphQL Code Generator v3 Roadmapの紹介と、そこでチームから推されているclient-presetのコンセプトと利用例を紹介します。
紹介には筆者個人の主観も多分に含んでいるため、より詳細かつ正しい情報を知りたい場合は[元のIssue](https://github.com/dotansimha/graphql-code-generator/issues/8296)を読むことをお勧めします。

なお、今回紹介するコードはサンプルリポジトリを用意しているので、そちらで挙動を確認することも可能です。

https://github.com/MH4GF/graphql-codegen-client-preset-example

# 記事の対象読者

この記事は以下のような方を対象として想定しており、GraphQL Code Generatorそのものの説明等は省きます。他の資料をご覧ください。

- GraphQL Code Generatorを知っている / 使ったことがある
- TypeScriptを使ってフロントエンドの開発をしている / typescript-react-apolloなどのプラグインを利用している

# client-preset は何を解決するのか

https://the-guild.dev/graphql/codegen/plugins/presets/preset-client

client-presetは「より良い開発体験・より小さなバンドルサイズ・より強い型・またベストプラクティスに簡単に追従できるようにする、全てのGraphQLクライアントのための新しいユニークなプリセット」と紹介されています。

GraphQL Code Generatorの [React / Vue 向けガイド](https://the-guild.dev/graphql/codegen/docs/guides/react-vue)は現在ではclient-presetを前提とした紹介となっており、またcodegen.tsを対話的に生成する `graphql-codegen init` コマンドでもデフォルトでclient-presetが使われるようになっています。このプリセットをデファクトスタンダードにする、という意志を感じられます。

その中身は以下のプラグインの組み合わせとなっています。

- typescript
- typed-document-node
- fragment-masking

このプリセットを使うことで**個別のGraphQLクライアント向けプラグイン（typescript-react-apolloやtypescript-urqlなど）が不要になり**、またそれらを使った開発体験とは大きく異なっています。
早速利用例を見てみましょう。

## typed-document-nodeで、個別GraphQLクライアントのためのプラグインが不要になる

まずtyped-document-nodeの機能を見ていきます。以下のようにcodegen.tsを設定します。
ここでは `client` presetを指定し、 `./src/gql` というディレクトリに生成コードを吐き出すように設定しています。

```typescript:codegen.ts
import { CodegenConfig } from '@graphql-codegen/cli'

const config: CodegenConfig = {
  schema: 'http://localhost:4000/graphql',
  documents: ['src/**/*.tsx'],
  generates: {
    './src/gql/': {
      preset: 'client',
      plugins: []
    }
  }
}

export default config
```

この設定により生成されるコードの内容については後ほど紹介します。続いて利用側のコードを見てみましょう。

```typescript:App.tsx
import React from 'react';
import { useQuery } from '@apollo/client';
import { graphql } from './gql/gql'; // 生成されたコードから `graphql()` をimport

import Film from './Film';

// GraphQLで取得したい内容を定義
// ここで定義した内容はTypedDocumentNode(TypeScriptの型付けがされたDocumentNode)となる
const allFilmsWithVariablesQueryDocument = graphql(/* GraphQL */ `
  query allFilmsWithVariablesQuery($first: Int!) {
    allFilms(first: $first) {
      edges {
        node {
          ...FilmItem
        }
      }
    }
  }
`);

function App() {
  // ほとんどのGraphQLクライアントはTypedDocumentNodeを扱う方法を知っているため、
  // 上記で定義したドキュメントをuseQueryに渡すだけで、返却されるdataや第二引数で渡すvariablesも型付けがされている！
  const { data } = useQuery(allFilmsWithVariablesQueryDocument, { variables: { first: 10 } });
  return (
    <div className="App">
      {data && <ul>{data.allFilms?.edges?.map((e, i) => e?.node && <Film film={e?.node} key={`film-${i}`} />)}</ul>}
    </div>
  );
}

export default App;
```

このように、その場で定義したDocumentNodeをuseQueryに渡すだけで型安全なdataやvariablesを扱えるようになります。useQueryのジェネリクスに型を渡す必要もないため、特定クライアント向けプラグインが生成していたカスタムフックが不要になります。
例えばtypescript-react-apolloの場合`useXXXQuery`と`useLazyXXXQuery`の両方を生成していたため、その2つが丸ごとなくなるというのはバンドルサイズの削減に大きく貢献するでしょう。またimportするのは `graphql()` だけで良いので、near-operation-file-presetも不要になります。

TypedDocumentNodeの利点はもう一つあります。生成されたカスタムフックを使わずにクエリ実行をしたい状況はあり（Next.jsのgetServerSideProps内など）、その際はResult型/Variables型/documentオブジェクトを実装者がクライアントへ渡すコードを書くことになりますが、その際**Result型とdocumentオブジェクトの内容が全く違ってもコンパイルエラーにはなりません**。実行時にならないとミスに気づけないためバグの原因になりがちですが、TypedDocumentNodeを使えばそういった型の設定ミスもなくなるのです。

さて、この自動的な型付けはどのように実現されているのでしょうか。生成されるコードを見てみましょう。
codegen.tsで出力場所として設定した`./src/gql/`には以下のコードが含まれています。

- gql.ts ... 上記で利用したgraphql()が含まれている
- graphql.ts ... typescriptプラグインによってデフォルトのスキーマから生成された型
- fragment-masking.ts ... 次の節で紹介します
- index.ts ... これらのファイルのre-export

ここではgql.tsを取り上げます。他の生成コードについては[サンプルリポジトリのソースコード](https://github.com/MH4GF/graphql-codegen-client-preset-example/tree/main/src/gql)もご覧ください。

```typescript:src/gql/gql.ts
/* eslint-disable */
import * as types from './graphql';
import { TypedDocumentNode as DocumentNode } from '@graphql-typed-document-node/core';

/**
 * Map of all GraphQL operations in the project.
 *
 * This map has several performance disadvantages:
 * 1. It is not tree-shakeable, so it will include all operations in the project.
 * 2. It is not minifiable, so the string of a GraphQL query will be multiple times inside the bundle.
 * 3. It does not support dead code elimination, so it will add unused operations.
 *
 * Therefore it is highly recommended to use the babel-plugin for production.
 */
const documents = {
    "\n  query allFilmsWithVariablesQuery($first: Int!) {\n    allFilms(first: $first) {\n      edges {\n        node {\n          ...FilmItem\n        }\n      }\n    }\n  }\n": types.AllFilmsWithVariablesQueryDocument,
    "\n  fragment FilmItem on Film {\n    id\n    title\n    releaseDate\n    producers\n  }\n": types.FilmItemFragmentDoc,
};

/**
 * The graphql function is used to parse GraphQL queries into a document that can be used by GraphQL clients.
 *
 *
 * @example
 * ```ts
 * const query = gql(`query GetUser($id: ID!) { user(id: $id) { name } }`);
 * ```
 *
 * The query argument is unknown!
 * Please regenerate the types.
 */
export function graphql(source: string): unknown;

/**
 * The graphql function is used to parse GraphQL queries into a document that can be used by GraphQL clients.
 */
export function graphql(source: "\n  query allFilmsWithVariablesQuery($first: Int!) {\n    allFilms(first: $first) {\n      edges {\n        node {\n          ...FilmItem\n        }\n      }\n    }\n  }\n"): (typeof documents)["\n  query allFilmsWithVariablesQuery($first: Int!) {\n    allFilms(first: $first) {\n      edges {\n        node {\n          ...FilmItem\n        }\n      }\n    }\n  }\n"];
/**
 * The graphql function is used to parse GraphQL queries into a document that can be used by GraphQL clients.
 */
export function graphql(source: "\n  fragment FilmItem on Film {\n    id\n    title\n    releaseDate\n    producers\n  }\n"): (typeof documents)["\n  fragment FilmItem on Film {\n    id\n    title\n    releaseDate\n    producers\n  }\n"];

export function graphql(source: string) {
  return (documents as any)[source] ?? {};
}

export type DocumentType<TDocumentNode extends DocumentNode<any, any>> = TDocumentNode extends DocumentNode<  infer TType,  any>  ? TType  : never;
```

クエリ文字列をキーに `types.AllFilmsWithVariablesQueryDocument` のようなTypedDocumentNodeが値となる `document` オブジェクトと、何やら力強いオーバーロード関数 `graphql()` が生成されています。
先ほどのApp.tsxでFilmのリストを取得するクエリを書いたように、ユーザーが `graphql()` 関数を使ってオペレーションを記述するとGraphQL Code Generatorはこの二つを生成します。 `graphql()` 関数が内部で行うことは引数の文字列リテラルでdocumentを呼び出すだけです。
一見生成されたコードにはギョッとしますが、ユーザーが記述した文字列リテラルをキーに対応する型定義を呼び出すだけ、という素朴な実装で実現されていることがわかります。

ちなみにこちらの生成コードのコメントにも記載されている通り、 `document` オブジェクトはTree Shakingが効かない・コードのminifyができない・未使用コードの削除ができないなどの問題を抱えているため、Productionでは同梱のbabelプラグインを利用することが推奨されています。
サンプルリポジトリで実際に設定したコミットはこちらです。バンドルサイズの変化も記載しています。 https://github.com/MH4GF/graphql-codegen-client-preset-example/commit/5fa53dd1746572de8370f99682c544653c24327a


## fragment-maskingによりFragment Colocationの強制が可能になる

続いてfragment-maskingの機能を見ていきます。
先ほどのApp.tsxでは取得したfilmの配列をmapしてFilmコンポーネントに渡していました。Filmコンポーネントの実装は以下のようになっています。

```typescript:Film.tsx
import { FragmentType, useFragment } from "./gql/fragment-masking";
import { graphql } from "./gql/gql";

export const FilmFragment = graphql(/* GraphQL */ `
  fragment FilmItem on Film {
    id
    title
    releaseDate
    producers
  }
`);

const Film = (props: { film: FragmentType<typeof FilmFragment> }) => {
  const film = useFragment(FilmFragment, props.film);
  return (
    <div>
      <h3>{film.title}</h3>
      <p>{film.releaseDate}</p>
    </div>
  );
};

export default Film;
```

FilmコンポーネントのためのGraphQL Fragmentを、先ほども利用した`graphql()`関数を使って定義しpropsで受け取っています。このようにコンポーネントに必要なデータをFragmentとして宣言することはFragment Colocationと呼ばれます。こちらの記事が詳しいです。

https://zenn.dev/moneyforward/articles/20221211-fragment-colocation

ここでfragment-maskingプラグインによる生成コードとして提供されるのは`FragmentType`型と`useFragment`関数で、これらを使うことでGraphQLクエリ結果からデータの取り出しが可能になります。**逆にいえば`useFragment`経由でなければデータの取り出しができません**。  
試しにVSCodeでApp.tsxで渡している値をホバーすると以下のような型情報が表示されます。

![](https://storage.googleapis.com/zenn-user-upload/354e44b1b867-20230117.png)

FilmFragmentとして定義したidやtitleが入っているのではなく、`$fragmentRefs`というよくわからない値が入っていますね。FilmコンポーネントでuseFragmentを通した結果も見てみましょう。

![](https://storage.googleapis.com/zenn-user-upload/768f99d5d970-20230117.png)

`FilmItemFragment` はコンポーネントファイルで定義したFragmentを元に `typescript`プラグインが生成した型です。GraphQL Code Generatorを使ったことがある人はお馴染みではないでしょうか。以下のようになっています。

```typescript:./src/gql/graphql.ts
export type FilmItemFragment = {
  __typename?: "Film";
  id: string;
  title?: string | null;
  releaseDate?: string | null;
  producers?: Array<string | null> | null;
} & { " $fragmentName"?: "FilmItemFragment" };
```

このようにfragment-maskingを有効にするとクエリの返却型にも手が入り、**オペレーションで定義したグラフ構造のフィールドではなくそのままでは扱えない値にマスキングされます。**  
そしてuseFragmentが型の変換だけを担っているというのは、生成された実装を見るとわかります。
```typescript:src/gql/fragment-masking.ts
// return non-nullable if `fragmentType` is non-nullable
export function useFragment<TType>(
  _documentNode: DocumentNode<TType, any>,
  fragmentType: FragmentType<DocumentNode<TType, any>>
): TType;
// return nullable if `fragmentType` is nullable
export function useFragment<TType>(
  _documentNode: DocumentNode<TType, any>,
  fragmentType: FragmentType<DocumentNode<TType, any>> | null | undefined
): TType | null | undefined;
// return array of non-nullable if `fragmentType` is array of non-nullable
export function useFragment<TType>(
  _documentNode: DocumentNode<TType, any>,
  fragmentType: ReadonlyArray<FragmentType<DocumentNode<TType, any>>>
): ReadonlyArray<TType>;
// return array of nullable if `fragmentType` is array of nullable
export function useFragment<TType>(
  _documentNode: DocumentNode<TType, any>,
  fragmentType: ReadonlyArray<FragmentType<DocumentNode<TType, any>>> | null | undefined
): ReadonlyArray<TType> | null | undefined;
export function useFragment<TType>(
  _documentNode: DocumentNode<TType, any>,
  fragmentType: FragmentType<DocumentNode<TType, any>> | ReadonlyArray<FragmentType<DocumentNode<TType, any>>> | null | undefined
): TType | ReadonlyArray<TType> | null | undefined {
  return fragmentType as any;
}
```

少々長いオーバーロードになっていますが、注目すべきは最下部の実装の中身です。行なっていることは引数として渡されたfragmentTypeをreturnしているだけです。つまりuseFragmentは実行時には何も行わず、開発時の型情報の変換のみを行なっていることがわかります。

Reactを書いている方は気になったかもしれませんが、**`useFragment`はReact Hooksではありません**。ESLintのルールに引っかかる場合などを避けるために`getFragmentData`などに命名変更ができます。詳しくはこちらをご覧ください: https://the-guild.dev/graphql/codegen/plugins/presets/preset-client#the-usefragment-helper
この関数をコンポーネント以外の箇所で利用する可能性を考えると `use` がつく関数名は誤解を招くため、個人的には命名変更する方が無難かなと考えています。



### useFragmentは実用に足るか

「useFragmentを使わなければGraphQLクエリのデータを取得できない」という制約は、本当に実用に足るのでしょうか。いくつかのユースケースを見てみます。

---

まずStorybookやJestなどでモックデータを用意したい場合です。以下のように `makeFragmentData` を使うとFragmentTypeを満たすオブジェクトを作ることができます。

```typescript
import { makeFragmentData } from "./gql/fragment-masking";

const mockData = makeFragmentData(FilmFragment, { title: "Sample", releaseDate: "1977-05-25" })
```

---

Fragmentの配列をコンポーネントとして定義したい場合は以下のようにします。

```typescript
const FilmList = (props: { films: ReadonlyArray<FragmentType<typeof FilmFragment>> }) => {
  const films = useFragment(FilmFragment, props.films);
}
```

:::details Reactのkeyと組み合わせる際のTips

先ほどのApp.tsxの例ではFilmの配列をmapしてFilmコンポーネントに渡していましたが、keyには`film-${i}`のようにインデックスが指定されていました。

```typescript:App.tsx
function App() {
  const { data } = useQuery(allFilmsWithVariablesQueryDocument, { variables: { first: 10 } });
  return (
    <div className="App">
      {data && <ul>{data.allFilms?.edges?.map((e, i) => e?.node && <Film film={e?.node} key={`film-${i}`} />)}</ul>}
    </div>
  );
}
```

Reactのお作法としてはデータ内に含まれる一意なIDを使うべきですが、fragment-maskingを使うとコンポーネントの外側からクエリのデータにアクセスできません。そのためにインデックスを使わなければなりませんでした。

今回のようにuseFragmentでは配列を受け取ることもできるので、可能であればこのようにリストを返すコンポーネントとして用意しIDをキーとして利用するのがベターかなと思われます。

:::


---

コンポーネントが受け取るpropsのうち、GraphQL由来ではない値も同列に扱いたい場合はたまにあります。

```typescript
// fragment-maskingがない頃の開発ではこのようなことができていた
type Props = {
  film: FilmItemFragment & {
    anotherData: boolean // GraphQLスキーマにはないデータ
  }
}

const Film = (props: Props) => {
  ...
};
```

これはfragment-masking利用下ではしづらくなったというのが事実かなと思います。しかし筆者はそこまで悪いことだとは考えていません。
例えばここで新たに追加したい `anotherData` がサーバーから返ってきたデータをもとに計算された値なのであれば、フロントエンドで計算するのではなく、スキーマのフィールドに追加し計算をバックエンド側で肩代わりする方が望ましいです。メルペイさんの記事から引用します。

> Componentを構成するとき、1つのUI要素を表示するために複数のGraphQLのフィールドを組み合わせる必要がある場合、その処理をバックエンド側で肩代わりするcomputed fieldを導入することを検討してください。

https://engineering.mercari.com/blog/entry/20221215-graphql-client-architecture-recommendation/

スキーマ駆動開発において、スキーマの設計はデータやドメインロジックの都合上バックエンド側が主導で設計が進むことは多いでしょう。しかしGraphQLにおいてフィールドはクライアント側のためのものなので、クライアント側が積極的にスキーマ設計に関与する方がGraphQLの持つ力を発揮しやすくなります。
fragment-maskingによってクライアントサイドでのcomputed fieldの実装がしづらいことは、クライアントサイドで安易にロジックが作られづらいとも言えるのではないでしょうか。バックエンド側との協調に力学が向きやすい、というのはメリットとして大きいのではないかと筆者は考えています。

---
いくつかのユースケースを見てきましたが、fragment-maskingの制約は実用に足るのではないかと筆者は判断しています。筆者は「設定より規約」の考え方が好きで、多少の学習コストはあれど実装者が実装方法に悩まなくなるのは大きなメリットだと思っています。
GraphQLクライアントのRelayでは厳密なFragment Colocationの強制ができますが、Apolloやurqlではこれまでは難しいというのが現状でした。near-operation-file-presetは生成コードをオペレーション定義の近くに置いてくれるだけで、実装者としてはfragmentとして定義せずに自前でコンポーネントのprops型を書いてクエリのデータを受け取りこともできるので、強制することはできません。fragment-maskingを使うとクエリのデータを扱いたい時はuseFragmentを使うしかなくなるため、それが禁止されるのです。

GraphQL Code GeneratorのGitHubリポジトリでは、fragment-maskingのQ&Aを公開しています。気になる質問が他にある方はここで聞いてみるのも良いでしょう。

https://github.com/dotansimha/graphql-code-generator/discussions/8554


## client-presetで注意すべき点

ここまでclient-presetで同梱されている機能や、それによって開発体験がどのように変わるかについて述べてきました。その上でプロダクションに導入すると考えた時に注意すべき点がいくつかあります。

### その他のプラグインを利用している場合はnear-operation-fileプリセットを併用する必要はある

near-operation-file-presetは、名前の通りオペレーション定義の近くに生成コードを配置してくれるプリセットです。コードの見通しが良くなるほか、生成コードの分割によるバンドル後のファイルチャンクの最適化にも大きく貢献します。以下の記事が詳しいです。

https://hiroppy.me/blog/nextjs-chunk-optimization/

client-presetを使うことで、コンポーネントのTypeScriptファイルに直接記述したドキュメントをそのまま利用できるためオペレーション定義に関してはnear-operation-file-presetは不要になる、と前述しました。しかしそれはオペレーション定義だけですので、[typescript-msw](https://the-guild.dev/graphql/codegen/plugins/typescript/typescript-msw)や[typescript-mock-data](https://github.com/ardeois/graphql-codegen-typescript-mock-data)など別の用途でもコード生成を利用する場合は今まで通りnear-operation-file-presetを利用しファイル分割した方が良いでしょう。

### typed-document-nodeが `.graphql` ファイルによるオペレーション定義に対応していない

現在のclient-presetは、 `graphql()` 関数を使ってコンポーネントに記述したDocumentNodeを利用してクエリを実行するため、 `.graphql` 拡張子のファイルによる定義を想定していないようです。issueでの質問には「[codemod](https://github.com/facebook/jscodeshift)を用意するなどのいくつかのマイグレーションパスを検討している」と[答えています](https://github.com/dotansimha/graphql-code-generator/issues/8296#issuecomment-1232718720)。
既にGraphQL Code Generatorを利用しているプロジェクトで`.graphql`ファイルでのオペレーション定義をしている場合の移行方法についてはこのIssueを追う方が良いかもしれません。

:::details 備考: near-operation-file-presetを使って無理やり.graphqlファイルを対応させる

実際には`.graphql`ファイルでオペレーションを定義しgraphql-codegenを実行した場合でも`src/gql/graphql.ts`にTypedDocumentNodeが生成されるため、そこからimportすれば利用は可能です。
しかし上述の通りgraphql.tsは全てのドキュメントが含まれているため、そこからimportするとバンドル時に無駄なコードが含まれることになるのが気になり、near-operation-file-presetを使って分割したくなります。というわけで試してみると問題なく動作はしました。

```typescript:codegen.ts
import { CodegenConfig } from "@graphql-codegen/cli";

const config: CodegenConfig = {
  schema: "https://swapi-graphql.netlify.app/.netlify/functions/index",
  documents: ["src/**/*.graphql"],
  generates: {
    "./src/gql/": {
      preset: "client",
      plugins: [],
    },
    "./src/": {
      preset: "near-operation-file",
      presetConfig: {
        extension: ".generated.ts",
        baseTypesPath: "~~/gql/graphql",
      },
      plugins: ["typed-document-node"],
      config: {
        typesPrefix: "Types.",
      },
    },
  },
};

export default config;
```

```graphql:FilmItem.graphql
fragment FilmItem on Film {
  id
  title
  releaseDate
  producers
}
```

```typescript:FilmItem.generated.ts
import * as Types from '~/gql/graphql';

import { TypedDocumentNode as DocumentNode } from '@graphql-typed-document-node/core';
export const FilmItemFragmentDoc = {"kind":"Document","definitions":[{"kind":"FragmentDefinition","name":{"kind":"Name","value":"FilmItem"},"typeCondition":{"kind":"NamedType","name":{"kind":"Name","value":"Film"}},"selectionSet":{"kind":"SelectionSet","selections":[{"kind":"Field","name":{"kind":"Name","value":"id"}},{"kind":"Field","name":{"kind":"Name","value":"title"}},{"kind":"Field","name":{"kind":"Name","value":"releaseDate"}},{"kind":"Field","name":{"kind":"Name","value":"producers"}}]}}]} as unknown as DocumentNode<Types.FilmItemFragment, unknown>;
```

```typescript:Film.tsx
import { FilmItemFragmentDoc } from "./FilmItem.generated";
import { FragmentType, useFragment } from "./gql/fragment-masking";

const Film = (props: { film: FragmentType<typeof FilmItemFragmentDoc> }) => {
  const film = useFragment(FilmItemFragmentDoc, props.film);
  return (
    <div>
      <h3>{film.title}</h3>
      <p>{film.releaseDate}</p>
    </div>
  );
};

export default Film;
```

このように、fragment-maskingを利用したレスポンスデータの取得もできていることがわかります。
この利用方法は推奨される方法なのかをIssueで質問していますが、今のところは返答がありません。返答があり次第この記事も加筆修正するつもりです。

https://github.com/dotansimha/graphql-code-generator/issues/8296#issuecomment-1399212141

:::

### 今後のアップデートによってデフォルトの設定が変わる可能性がある

この件は良い影響が大きいとも個人的には思いますが、破壊的変更の可能性があるので知っておくと良いということで紹介します。
GraphQL Code Generatorでは数多くのオプションがあります。例えばデフォルトではGraphQLのEnumはTypeScriptのEnumとして出力されますが、TypeScriptのEnumはいくつか問題点がありできれば使わない方が望ましいです。`enumAsTypes`を設定すると、GraphQLのEnumをTypeScriptのユニオン型として出力してくれます。他にもこちらの記事ではおすすめの設定が紹介されています。

https://zenn.dev/izumin/articles/ffc84c1b4310be

現在client-presetではデフォルトの設定としていくつかのオプションを取り込むことを検討しているようです。

https://github.com/dotansimha/graphql-code-generator/issues/8562

ここでは上述した `enumAsTypes` も含めいくつかのオプションのデフォルト化を議論しています。
GraphQL Code Generatorを利用し始めた際、「これらのオプションの存在を後から知ったものの、今から導入するとなると影響範囲が大きく難しい」という状況になったことがある方も多いのではないでしょうか。そういったベストプラクティスに追従しやすくなる取り組みはとても良いですね。
とはいえ破壊的変更になる可能性はあるので認識しておきつつ、その前にclient-presetを利用し始めるタイミングでプロジェクトに合うオプションを設定しておけると良いかと思います。

## まとめ

この記事ではGraphQL Code Generator v3 Roadmapの紹介と、そこでチームから推されているclient-presetのコンセプトと利用例を紹介しました。fragment-maskingの強い制約は賛否が分かれそうですが、個人的には開発体験が高まる良い方向に進んでいると感じます。
ぜひみなさんの意見もお聞かせください！ここまで読んでいただきありがとうございました。

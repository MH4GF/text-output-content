---
title: "GraphQL Code Generator v3 Roadmapで推されているclient-presetを紹介する"
emoji: "🔥"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: [graphql, graphqlcodegen]
published: false
---

こんにちは。皆さんは[GraphQL Code Generator](https://the-guild.dev/graphql/codegen)を利用していますか？
筆者は普段React/TypeScript/Apollo Client(またはurql)といったスタックでWebフロントエンドを書いており、その際にはGraphQL Code Generatorをほぼ必需品と言えるほど愛用しています。
サーバー側から提供されたスキーマやクライアント側が必要なデータを宣言したオペレーションから型やコードを生成し利用することで、ロジックに関する実装量が大きく削減でき、ミスを減らすことにもつながります。GraphQLを使う理由の1つと言っても過言ではないでしょう。

そのGraphQL Code Generatorでは[v3 Roadmap](https://github.com/dotansimha/graphql-code-generator/issues/8296)として今後の方針が公開されており、[client-preset](https://the-guild.dev/graphql/codegen/plugins/presets/preset-client)という新しいプリセットのリリースが紹介されています。そこでは「**GraphQL Code Generator 3.0のclient-presetが、既存のhooksやSDKベースのプラグインをすべて置き換え、フロントエンドのユースケース向けにGraphQL Typeを生成する公式な方法となることを目指します**」と記載されています。これは一体どういうことなのでしょうか。

この記事ではGraphQL Code Generator v3 Roadmapの紹介と、そこでチームから推されているclient-presetのコンセプトと利用例を紹介します。
紹介には筆者個人の主観も多分に含んでいるため、より詳細かつ正しい情報を知りたい場合は元のIssueを読むことをお勧めします。

なお、今回紹介するコードはサンプルリポジトリを用意しているので、そちらで挙動を確認することも可能です。

https://github.com/MH4GF/graphql-codegen-client-preset-example

# TL;DR

client-presetを利用することで以下のメリットを享受できます。

- 各GraphQLクライアントごとの差異を気にせず済む汎用的なコードを生成する
- Relayと同等のFragment Colocationの強制を実現できる

# client-preset は何を解決するのか

https://the-guild.dev/graphql/codegen/plugins/presets/preset-client

client-presetは「より良い開発体験・より小さなバンドルサイズ・より強い型・またベストプラクティスに簡単に追従できるようにする、全てのGraphQLクライアントのための新しいユニークなプリセット」と紹介されており、中身は以下のプラグインの組み合わせとなっています。

- typescript
- typed-document-node
- fragment-masking

このプリセットを使うことで個別のGraphQLクライアント向けプラグイン（typescript-react-apolloやtypescript-urqlなど）が不要になり、またそれらを使った開発体験とは大きく異なります。
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
// ここで定義した内容はTypedDocumentNode(後述)として型付けされる
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
例えばtypescript-react-apolloの場合`useXXXQuery`と`useLazyXXXQuery`の両方を生成していたため、その2つが丸ごとなくなるというのはバンドルサイズの削減に大きく貢献するでしょう。

これはどのように実現されているのでしょうか。生成されるコードを見てみましょう。
codegen.tsで出力場所として設定した`./src/gql/`には以下のコードが含まれています。

- gql.ts ... 上記で利用したgraphql()が含まれている
- graphql.ts ... typescriptプラグインによってスキーマから生成された型
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

クエリ文字列をキーに `types.AllFilmsWithVariablesQueryDocument` のようなTypedDocumentNodeが値となる `document` オブジェクトと、何やら力強いオーバーロード関数生成されています。
先ほどのApp.tsxでFilmのリストを取得するクエリを書いたように、ユーザーが `graphql()` 関数を使ってオペレーションを記述するとGraphQL Code Generatorはこの二つを生成します。 `graphql()` 関数が行うことは引数の文字列リテラルでdocumentを呼び出すだけです。
一見ギョッとしますが、ユーザーが記述した文字列リテラルをキーに対応する型定義を呼び出すだけ、という素朴な実装で実現されていることがわかります。

ちなみにこちらの生成コードのコメントにも記載されている通り、 `document` オブジェクトはTree Shakingが効かない・コードのminifyができない・未使用コードの削除ができないなどの問題を抱えているため、Productionでは同梱のbabelプラグインを利用することが推奨されています。
サンプルリポジトリで実際に設定したコミットはこちらです: https://github.com/MH4GF/graphql-codegen-client-preset-example/commit/5fa53dd1746572de8370f99682c544653c24327a


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

`$fragmentRefs`というよくわからない値が入っていますね。useFragmentを通した結果も見てみましょう。

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

このように、fragment-maskingを有効にすると、クエリの返却型はオペレーションで定義したグラフ構造ではなくそのままでは扱えない値にマスキングされます。  
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

Reactを書いている方は気になったかもしれませんが、`useFragment`はReact Hooksではありません。ESLintのルールに引っかかるなどを避けたい場合は`getFragmentData`などに命名変更ができます。^[https://the-guild.dev/graphql/codegen/plugins/presets/preset-client#the-usefragment-helper]
個人的には、この関数をNext.jsのgetServerSideProps内などコンポーネント以外の箇所で利用する可能性を考えると、 `use` がつく関数名は誤解を招くため命名変更する方が無難かなと考えています。



### useFragmentは実用に足るか

「useFragmentを使わなければGraphQLクエリのデータを取得できない」という制約は、本当に実用に足るのでしょうか。いくつかのユースケースを見てみます。

---

まずStorybookやJestなどでモックデータを用意したい場合です。以下のようにするとFragmentTypeを満たすオブジェクトを作ることができます。

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

先ほどのApp.tsxの例ではFilmの配列をmapしてFilmコンポーネントに渡していましたが、keyには`film-${i}`のようにインデックスが指定されていました。Reactのお作法としてはデータ内に含まれる一意なIDを使うべきですが、コンポーネントの外側からクエリのデータにアクセスできないためにインデックスを使っています。
useFragmentでは配列を受け取ることもできるので、可能であればこのようにリストを返すコンポーネントとして用意しIDをキーとして利用するのがベターかなと思われます。

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
例えばここで新たに追加したい `anotherData` がサーバーから返ってきたデータをもとに計算された値なのであれば、フロントエンドで計算するのではなく、スキーマのフィールドに追加し計算をバックエンド側で肩代わりする方が望ましいです。

> Componentを構成するとき、1つのUI要素を表示するために複数のGraphQLのフィールドを組み合わせる必要がある場合、その処理をバックエンド側で肩代わりするcomputed fieldを導入することを検討してください。

https://engineering.mercari.com/blog/entry/20221215-graphql-client-architecture-recommendation/

スキーマ駆動開発において、スキーマの設計はデータやドメインロジックの都合上バックエンド側が主導で設計が進むことは多いでしょう。しかしGraphQLにおいてフィールドはクライアント側のためのものなので、クライアント側が積極的にスキーマ設計に関与する方がGraphQLの持つ力を発揮しやすくなります。
fragment-maskingによってクライアントサイドでのcomputed fieldの実装がしづらいことは、クライアントサイドで安易にロジックが作られづらいとも言えるのではないでしょうか。バックエンド側との協調に力学が向きやすい、というのはメリットとして大きいのではないかと筆者は考えています。

---
いくつかのユースケースを見てきましたが、fragment-maskingの制約は実用に足るのではないかと筆者は判断しています。筆者は設定より規約の考え方が好きで、多少の学習コストはあれど実装者が実装方法に悩まなくなるのは大きなメリットだと思っています。
GraphQLクライアントのRelayでは厳密なFragment Colocationの強制ができますが、Apolloやurqlではこれまでは難しいというのが現状でした。near-operation-file-presetは生成コードをオペレーション定義の近くに置いてくれるだけで、実装者としてはfragmentとして定義せずに自前でコンポーネントのprops型を書いてクエリのデータを受け取りこともできるので、強制することはできません。client-presetを使うとクエリのデータを使いたい場合はuseFragmentを使うしかなくなるため、それが禁止されるのです。

GraphQL Code GeneratorのGitHubリポジトリでは、fragment-maskingのQ&Aを公開しています。気になる質問が他にある方はここで聞いてみるのも良いでしょう。

https://github.com/dotansimha/graphql-code-generator/discussions/8554


## client-presetで注意すべき点

ここまでclient-presetで同梱されている機能や、それによって開発体験がどのように変わるかについて述べてきました。その上でプロダクションに導入すると考えた時に注意すべき点がいくつかあります。

### その他のプラグインを利用している場合はnear-operation-fileプリセットを併用する必要はある

### typed-document-nodeが `.graphql` ファイルによるオペレーション定義に対応していない

TODO: near-operation-preset使ってるけどいいのかここで聞いてみてる
https://github.com/dotansimha/graphql-code-generator/issues/8296#issuecomment-1399212141

### 今後のアップデートによって設定が変わる可能性がある


# メモ

- v3ロードマップで推されている主要機能
- 今までのgraphql-codegenでの開発体験と大きく変わるが、個人的には良い方向なのではと思っている
- いくつかのプラグインの組み合わせ
  - typescript
  - typed-document-node
  - fragment-masking
- メリット
  - 特定のクライアントライブラリのためのコードが不要であり、documentをuseQueryに渡せばそれだけで返却型で型付けされる（これはtyped-document-nodeの機能）
    - TODO: どういう仕組み？graphql.jsの機能？
  - documentはgraphql()を使って書くが、それがそのまま使えるためnear-operation-fileが不要
    - 本当に不要？
    - ただし他のプラグインを利用する場合は必要になる（mswやtypescript-mock-dataなど）
  - fragment-maskingがデフォルトで有効になっており、relayのような制約の強いfragment colocationが可能
- typed-document-node
  - documentをgraphql()として書くと、codegenは力技のような関数オーバーロードを作成し型付けされたdocumentを返す
  - これにより、useQueryの引数にdocumentを渡すだけで型付けされたデータが返ってくる
  - graphql定義は。graphqlファイルに書きたい派だったが無理なんですかね？
  - typed-document-nodeがない場合、hookを使えない場所（getServerSidePropsなど）でクエリを実行したい場合は生成型とドキュメントを手動で渡すみたいなことをしていたが、型とドキュメントが一致しなくてもエラーにならず、ミスが起きがちだった
  - typed-document-nodeにすれば型を推論してくれるため、そのミスは起きにくくなる
- fragment-masking
  - useQueryのresult.dataの型がマスキングされ、useFragmentを通さないとデータを参照できなくなる
    - useFragmentはgraphql()で書いたdocumentを使う
  - かなり制約として強いが、fragmentの使い方を統一できるので良さそうか
    - 微妙な点として、配列をコンポーネントにmapする際keyにobject.idが使えずindexを使うことになる
      - やりようはあるんですかね？
  - 使い方
    - `FragmentType<typeof myDocument>`のように使う
    - テストデータを作りたい場合は`makeFragmentData({ id: 'foo'})`のように使う
    - listのコンポーネントの場合は`ReadonlyArray<FragmentType<typeof myDocument>>` みたいな受け取り方になる
- TODO: 移行をどのように進めるか？
- TODO: まだexperimentalなんだっけ？

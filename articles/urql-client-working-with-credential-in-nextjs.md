---
title: "Next.js上のGraphQLクライアントから秘匿情報を安全に扱いつつAPIリクエストを行う"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [graphql, nextjs, urql, auth0]
published: true
---

この記事は[GraphQLアドベントカレンダー2022](https://qiita.com/advent-calendar/2022/graphql)の20日目の記事です。
昨日は[@ruriro0125](https://qiita.com/ruriro0125)さんによる[MSWを用いたStorybook・テスト上で使うGraphQLモックサーバーの運用](https://qiita.com/ruriro0125/items/f105a2f605a0ad005fbf)でした。

GraphQLクライアントを利用してバックエンドのAPIサーバーにリクエストを行う際、Authorizationヘッダに認証情報を載せてリクエストするパターンは一般的によくあるユースケースでしょう。
その際ブラウザで実行するJavaScriptからアクセスできる認証情報は可能な限り最小限にしたいです。XSS攻撃により秘匿情報の摂取ができてしまう恐れがあるためです。

今回はNext.js上のGraphQLクライアントから行うAPIリクエストにおいて、秘匿情報をブラウザに露出しないためのテクニックを1つ紹介します。
具体的な説明のためGraphQLクライアントにはurqlを、IDaaSとしてAuth0のSDKを用いて説明しますが、ライブラリの知識は特に必要ありません。他のライブラリに当てはめても利用できるはずです。

## 今回記載しないこと

以下の内容についての詳細の説明はしません。

- Next.jsの各機能の説明
- JsonWebToken(以下JWT)の仕組み
- Auth0による認証・認可の仕組み

## ユースケース

Auth0で認証したユーザーのセッション情報のうち、JWTとして提供されているアクセストークンをバックエンドAPIの認可のために利用するユースケースを想定します。
Auth0では[Next.js 用の SDK](https://github.com/auth0/nextjs-auth0)が提供されており、そのうちアクセストークンを取得するには[getAccessToken](https://auth0.github.io/nextjs-auth0/modules/session_get_access_token.html)関数を利用します。この関数はAPI RoutesやgetServerSidePropsなどのサーバーサイドでのみ利用可能です。

urqlでHTTPリクエストヘッダに情報を追加するためには、まずは以下のように公式ドキュメントに記載されているfetchOptionsの利用を検討します。

```typescript
import { createClient } from "urql";
import { getAccessToken } from "@auth0/nextjs-auth0";

const accessToken = getAccessToken(); // 引数にreq/resが必要で、ここで呼び出すことはできない

export const client = createClient({
  url: process.env.NEXT_PUBLIC_BACKEND_URL,
  fetchOptions: {
    headers: {
      Authorization: `Bearer ${accessToken}`,
    },
  },
});
```

しかし前述のgetAccesstoken()はクライアントサイドでの実行ができないため、ここでの実行はできません。^[urql が提供する別の方法である authExchange もクライアントサイドで実行されるため今回の問題の解決にはなりません。]

## バッドプラクティス

クライアントサイドで利用可能にするため、getAccessTokenの結果を返すAPI Routesを作って呼び出すのはどうでしょうか。しかしこれは良くない解決策です。XSS脆弱性があった場合に秘匿情報の窃取ができてしまったり、セッションハイジャックの恐れがあるためです。^[この方法は実は以前までnextjs-auth0のREADMEに記載されていた方法ですが、セキュリティ上の問題の指摘を受けて削除されています。https://github.com/auth0/nextjs-auth0/pull/245]

## 解決策

[next-http-proxy-middleware](https://github.com/stegano/next-http-proxy-middleware)を利用し、GraphQLリクエストをプロキシするAPI Routesを作るのが良いでしょう。

### GraphQL クライアントを定義

```typescript
import { createClient } from "urql";

export const client = createClient({
  url: "/api/graphql", // 相対パスでAPI Routesを指定
});
```

### API Routes を定義

```typescript
import httpProxyMiddleware from "next-http-proxy-middleware";
import { getAccessToken } from "@auth0/nextjs-auth0";
import type { NextApiRequest, NextApiResponse } from "next";

export default (req: NextApiRequest, res: NextApiResponse) => {
  /**
   * Auth0のアクセストークンを取得する関数
   * 未ログインやセッション切れ等でnullが返ってきた場合は401を返すこととします
   */
  const { accessToken } = await getAccessToken(req, res);
  if (!accessToken) {
    res.status(401).end();
    return;
  }

  return httpProxyMiddleware(req, res, {
    target: process.env.BACKEND_URL,
    proxyTimeout: 5000,
    headers: {
      Authorization: `Bearer ${accessToken}`,
    },
    pathRewrite: [
      {
        patternStr: "^/api/graphql",
        replaceStr: "/graphql", // localhost:4000/graphqlにrewrite
      },
    ],
  });
};
```

ここではAuth0のSDKを利用してアクセストークンを取得し、httpProxyMiddlewareのプロキシでAuthorizationヘッダを付与しています。
この方法を取ると、秘匿情報も含めGraphQLリクエストに必要な環境変数は全てAPI Routesからのみ呼び出されることになります。そのため `NEXT_PUBLIC_*` で環境変数をブラウザ上のJSに公開せず済むのも利点です。

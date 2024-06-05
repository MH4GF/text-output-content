---
title: "valtioに依存するReactコンポーネントをStorybookで表示する"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["storybook", "valtio", "react"]
published: true
publication_name: route06
---

状態管理ライブラリに [valtio](https://valtio.pmnd.rs/) を採用した React のプロジェクトで Storybook を利用した際に、Story ごとに状態を設定する方法を紹介します。

以下の Store があるとします。

```ts:store.ts
import { proxy } from 'valtio'

export type Store = {
  posts: {
    [id in string] {
      title: string
      slug: string
    }
  }
}

export const store = proxy<Store>({ posts: {} })
```

valtio のスナップショットを利用してレンダリングを行うコンポーネントを想定します。

```tsx:PostList.tsx
import { useSnapshot } from "valtio";
import { store } from "./store";

export const PostList = () => {
  const { posts } = useSnapshot(store);

  return (
    <ul>
      {Object.entries(posts).map(([id, post]) => (
        <li key={id}>{post.title}</li>
      ))}
    </ul>
  );
};
```

valito の `useSnapshot()` から `posts` を受け取りレンダリングするコンポーネントです。post の追加は別のコンポーネントが担い、また props からデータを受け取るわけではないため、Storybook 上で表示する際にはそのままでは初期値である空の表示しかできません。

この問題を解決するために、今回は Decorator を使って store のモックを提供します。最終的に以下のようにストーリーを書けます。

```tsx:PostList.stories.tsx
import type { Meta, StoryObj } from "@storybook/react";

import type { Store } from "./store";
import { PostList } from "./PostList";

const meta = {
  component: PostList,
  args: {},
} satisfies Meta<typeof PostList>;

export default meta;
type Story = StoryObj<typeof meta>;

export const Default: Story = {
  parameters: {
    store: {
      posts: {
        post1: {
          title: "Post 1",
          slug: "/post-1",
        },
        post2: {
          title: "Post 2",
          slug: "/post-2",
        },
        post3: {
          title: "Post 3",
          slug: "/post-3",
        },
      },
    } satisfies Partial<Store>,
  },
};
```

このように、parameter で初期値を設定できると便利です。この実現方法を紹介します。

## Decorator の用意

以下のような Decorator を実装します。

```tsx:.storybook/storeDecorator.tsx
import type { Decorator } from "@storybook/react";
import { useEffect } from "react";
import type { PropsWithChildren } from "react";
import { store } from "@/store";
import type { Store } from "@/store";

export const storeDecorator: Decorator = (Story, { parameters }) => {
  // parametersにstoreが指定されている場合、InitializeStoreコンポーネントでラップする
  if (parameters?.store) {
    return (
      <InitializeStore value={parameters.store}>
        <Story />
      </InitializeStore>
    );
  }

  return <Story />;
};

type Props = {
  value: Partial<Store>;
};

const InitializeStore = ({ value, children }: PropsWithChildren<Props>) => {
  // 渡されてきた値をstoreに保存
  useEffect(() => {
    for (const [key, val] of Object.entries(value)) {
      if (val !== undefined) {
        store[key] = val;
      }
    }
  }, [value]);

  return children;
};
```

Decorator は [第二引数にストーリーに設定した parameters が含まれている](https://storybook.js.org/docs/writing-stories/decorators#context-for-mocking)ので、parameters に store というオブジェクトがあった場合は useEffect で store に保存します。  
valtio の useSnapshot は store の更新に伴い適切に再レンダリングするよう最適化されているので、初期値の挿入後 Story のコンポーネントが再レンダリングされます。

この Decorator を `.storybook/preview.ts` か、各ストーリーで利用します。私たちは `preview.ts` でグローバルに利用しています。

こうすることで、各ストーリーで valtio の状態を自由に表現できるようになりました。

```tsx:PostList.stories.tsx
import type { Meta, StoryObj } from '@storybook/react'

import type { Store } from './store'
import { PostList } from './PostList'

const meta = {
  component: PostList,
  args: {},
} satisfies Meta<typeof PostList>

export default meta
type Story = StoryObj<typeof meta>

export const Default: Story = {
  parameters: {
    store: {
      posts: {
        post1: {
          title: 'Post 1',
          slug: '/post-1',
        },
        post2: {
          title: 'Post 2',
          slug: '/post-2',
        },
        post3: {
          title: 'Post 3',
          slug: '/post-3',
        },
      },
    } satisfies Partial<Store>,
  },
}

export const Empty: Story = {
  name: '投稿が空の時'
  parameters: {
    store: {
      posts: {}
    } satisfies Partial<Store>
  }
}
```

## 余談

satisfies はなくても動作しますが、あることで IDE での補完が効くため追加しています。  
プロジェクトで利用しているアドオンなども含め、Storybook の parameters の型をグローバルに上書きできたら一番嬉しいんだけどなあ...
TypeScript の Module Argumentation ではできなさそうだったため、もし実現方法があったらコメントで教えていただきたいです。

https://github.com/storybookjs/storybook/issues/22860

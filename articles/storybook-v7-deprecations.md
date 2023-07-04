---
title: "StoryBook v7.0 へのアップデートで非推奨になった型"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
publication_name: route06
published: false
---

関わっているプロダクトで StoryBook を v6.5 から v7 にアップデートしました。v7 ではいくつかの型や設定が非推奨になっており、どのように移行すればよいのか・なぜ非推奨になったのかを調べていました。その非推奨になった型を見ていきます。

今回の記事は関わっているプロダクトで対応が必要だった型についての調査であり、すべての非推奨になった型や設定は含めていません。そちらについては以下のマイグレーションガイドをご覧ください。

https://github.com/storybookjs/storybook/blob/next/MIGRATION.md#70-deprecations-and-default-changes

## `ComponentMeta` `ComponentStoryObj` の非推奨化

Story のメタ情報を定義する `ComponentMeta` と 個々の Story を定義する `ComponentStoryObj` が非推奨になり、代わりに `Meta` と `StoryObj` を使うようになりました。

```diff tsx
- import type { ComponentMeta, ComponentStoryObj } from '@storybook/react'
+ import type { Meta, StoryObj } from '@storybook/react'

import { Button } from './Button'

- const meta: ComponentMeta<typeof Button> = {
+ const meta: Meta<typeof Button> = {
  component: Button,
  args: {
    children: 'ボタン',
  },
}

export default meta

- type Story = ComponentStoryObj<typeof Button>
+ type Story = StoryObj<typeof Button>

export const Default: Story = {
  argTypes: {
    onClick: { action: true },
  },
}
```

これらの変更理由は、冗長だった型を短縮したとのことでした。

また今までは `ComponentStoryObj<typeof Button>` のようにコンポーネントをジェネリクスに渡していましたが、 `StoryObj<ButtonProps>` のように Args を渡すこともできるようになったとのことです。

続いて、それぞれの型定義も見てみます。

````ts
/**
 * @deprecated Use `Meta` instead, e.g. ComponentMeta<typeof Button> -> Meta<typeof Button>.
 *
 * For the common case where a component's stories are simple components that receives args as props:
 *
 * ```tsx
 * export default { ... } as ComponentMeta<typeof Button>;
 * ```
 */
type ComponentMeta<T extends JSXElement> = Meta<ComponentProps<T>>;
````

[https://github.com/storybookjs/storybook/blob/47446f4aa213d8bf65ce5f59223b02c83a4901b0/code/renderers/react/src/public-types.ts#L77-L85](https://github.com/storybookjs/storybook/blob/47446f4aa213d8bf65ce5f59223b02c83a4901b0/code/renderers/react/src/public-types.ts#L77-L85)

```ts
/**
 * Metadata to configure the stories for a component.
 *
 * @see [Default export](https://storybook.js.org/docs/formats/component-story-format/#default-export)
 */
type Meta<TCmpOrArgs = Args> = TCmpOrArgs extends ComponentType<any>
  ? ComponentAnnotations<ReactRenderer, ComponentProps<TCmpOrArgs>>
  : ComponentAnnotations<ReactRenderer, TCmpOrArgs>;
```

[https://github.com/storybookjs/storybook/blob/47446f4aa213d8bf65ce5f59223b02c83a4901b0/code/renderers/react/src/public-types.ts#L23-L30](https://github.com/storybookjs/storybook/blob/47446f4aa213d8bf65ce5f59223b02c83a4901b0/code/renderers/react/src/public-types.ts#L23-L30)

````ts
/**
 * @deprecated Use `StoryObj` instead, e.g. ComponentStoryObj<typeof Button> -> StoryObj<typeof Button>.
 *
 * For the common case where a (CSFv3) story is a simple component that receives args as props:
 *
 * ```tsx
 * const MyStory: ComponentStoryObj<typeof Button> = {
 *   args: { buttonArg1: 'val' },
 * }
 * ```
 */
type ComponentStoryObj<T extends JSXElement> = StoryObj<ComponentProps<T>>;
````

[https://github.com/storybookjs/storybook/blob/47446f4aa213d8bf65ce5f59223b02c83a4901b0/code/renderers/react/src/public-types.ts#L100-L111](https://github.com/storybookjs/storybook/blob/47446f4aa213d8bf65ce5f59223b02c83a4901b0/code/renderers/react/src/public-types.ts#L100-L111)

```ts
/**
 * Story function that represents a CSFv3 component example.
 *
 * @see [Named Story exports](https://storybook.js.org/docs/formats/component-story-format/#named-story-exports)
 */
type StoryObj<TMetaOrCmpOrArgs = Args> = TMetaOrCmpOrArgs extends {
  render?: ArgsStoryFn<ReactRenderer, any>;
  component?: infer Component;
  args?: infer DefaultArgs;
}
  ? Simplify<
      (Component extends ComponentType<any>
        ? ComponentProps<Component>
        : unknown) &
        ArgsFromMeta<ReactRenderer, TMetaOrCmpOrArgs>
    > extends infer TArgs
    ? StoryAnnotations<
        ReactRenderer,
        TArgs,
        SetOptional<
          TArgs,
          keyof TArgs & keyof (DefaultArgs & ActionArgs<TArgs>)
        >
      >
    : never
  : TMetaOrCmpOrArgs extends ComponentType<any>
  ? StoryAnnotations<ReactRenderer, ComponentProps<TMetaOrCmpOrArgs>>
  : StoryAnnotations<ReactRenderer, TMetaOrCmpOrArgs>;
```

[https://github.com/storybookjs/storybook/blob/47446f4aa213d8bf65ce5f59223b02c83a4901b0/code/renderers/react/src/public-types.ts#L46-L63](https://github.com/storybookjs/storybook/blob/47446f4aa213d8bf65ce5f59223b02c83a4901b0/code/renderers/react/src/public-types.ts#L46-L63)

`ComponentMeta` `ComponentStoryObj` どちらも新しい型を参照するようになっていました。

## `DecoratorFn` の非推奨化

Story に対してラップするコンポーネントを追加するデコレーターを定義するために利用していた `DecoratorFn` が非推奨になり、代わりに `Decorator` を使うようになりました。こちらはまず型定義を見てみます。

```ts
/**
 * @deprecated Use Decorator instead.
 */
type DecoratorFn = DecoratorFunction<ReactRenderer>;
type Decorator<TArgs = StrictArgs> = DecoratorFunction<ReactRenderer, TArgs>;
```

代替先の `Decorator` はジェネリクスで Args を渡せるようになっています。以下のようにデコレーター関数内で args を参照することができます。

```tsx
import type { Decorator } from "@storybook/react";
import { LocaleProvider } from "./locale";

const withLocale: Decorator<{ locale: "en" | "es" }> = (Story, { args }) => (
  <LocaleProvider lang={args.locale}>
    <Story />
  </LocaleProvider>
);
```

引用元: https://github.com/storybookjs/storybook/blob/next/MIGRATION.md#renamed-decoratorfn-to-decorator

また、 `Decorator` に渡されているジェネリクスのデフォルトは `StrictArgs` となっています。こちらも見てみます。

```ts
interface StrictArgs {
  [name: string]: unknown;
}
```

`DecoratorFn` での args は `[name: string]: any;` となっていたため、これは軽微ではあるものの非互換な変更です。

`Decorator` 利用時に `DecoratorFn` との互換性を保つ必要がある場合、以下のように `Args` を追加で import すれば良いです。

```tsx
import type { Args, Decorator } from '@storybook/react';

const withLocale: Decorator<Args> = (Story, { args }) => // args has type { [name: string]: any }
```

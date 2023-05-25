# Storybook の Preview(コンポーネントを描画してる箇所)部分はどのようにコンポーネントを描画しているのか

- dev コマンドで起動するサーバーに対して、`/iframe.html`へ投げたリクエストのレスポンス部分が描画されてそう
- Manager の UI は`managerBuilder.start()`で定義されていた
- ということは Preview(`/iframe.html`)へのリクエストの処理は`previewBuilder.start()`で定義されてるのでは
  - https://github.com/storybookjs/storybook/blob/09db9f7dd11356714952449cbfd8280407b014c3/code/lib/core-server/src/dev-server.ts#L90C34-L97

## previewBuilder

- `previewBuilder` は `getPreviewBuilder` の返り値
  - https://github.com/storybookjs/storybook/blob/09db9f7dd11356714952449cbfd8280407b014c3/code/lib/core-server/src/dev-server.ts#L70
- `getPreviewBuilder`では`@storybook/builder-{builderName}`のパッケージが dynamic import でインポートされている

```ts
{
  builderName: "/Users/nus3/dev/fork/storybook/sandbox/react-vite-default-ts/node_modules/@storybook/builder-vite";
}
{
  builderPackage: "/Users/nus3/dev/fork/storybook/sandbox/react-vite-default-ts/node_modules/@storybook/builder-vite/dist/index.js";
}
```

- builder と renderer は code/frameworks で定義されてる
- storybook の main.ts(コンフィグ)で`framework`に`@storybook/react-vite`を指定した場合、`code/frameworks/react-vite/src/preset.ts`が読み込まれる
  - TODO: 該当の実装箇所を読む
- この preset の`core`オブジェクトに`builder`と`renderer`が定義されている
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/frameworks/react-vite/src/preset.ts#L9-L12
- 今回の場合、builder には`@storybook/builder-vite`が利用されている

## `@storybook/builder-vite`

- `code/lib/builder-vite`に実装がある
  - https://github.com/storybookjs/storybook/tree/next/code/lib/builder-vite

## TODO

`@storybook/builder-vite`(`code/lib/builder-vite`)の実装を読む

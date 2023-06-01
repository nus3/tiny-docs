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
  builderName: "home/storybook/sandbox/react-vite-default-ts/node_modules/@storybook/builder-vite";
}
{
  builderPackage: "home/storybook/sandbox/react-vite-default-ts/node_modules/@storybook/builder-vite/dist/index.js";
}
```

- builderName の`core.builder`か`core.builder.name`
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/lib/core-server/src/dev-server.ts#L67
- `core`は`options.presets.apply`
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/lib/core-server/src/dev-server.ts#L29
- builder は`loadAllPresets`から取得されてそう
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/lib/core-server/src/build-dev.ts#L84-L90
- `loadAllPresets`では`loadCustomPresets`が呼ばれてそう
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/lib/core-common/src/presets.ts#L374
- `loadCustomPresets`の`configDir`には、対象の storybook のコンフィグのパスが格納されている
  - ` configDir: 'storybook/sandbox/react-vite-default-ts/.storybook'`
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/lib/core-common/src/utils/load-custom-presets.ts#L9-L10
  - `.storybook`に main と preset があるかどうかを確認し、`main`があれば main の値を返し、そうじゃない場合は presets の値を返す
- builder と renderer は code/frameworks で定義されてる
- storybook の main.ts(コンフィグ)で`framework`に`@storybook/react-vite`を指定した場合、`code/frameworks/react-vite/src/preset.ts`が読み込まれる
  - TODO: 該当の実装箇所を読む
- この preset の`core`オブジェクトに`builder`と`renderer`が定義されている
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/frameworks/react-vite/src/preset.ts#L9-L12
- 今回の場合、builder には`@storybook/builder-vite`が利用されている

### builder をどのように取得しているのか

- loadAllPresets に渡す corePresets は main.ts の framework で指定した値に`preset`を追加したもの
  - `{ corePresets: [ '@storybook/react-vite/preset' ] }`
- `loadAllPresets`で preset を読み込み、`preset.apply`で`builder`や`renderer`を取得している
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/lib/core-server/src/build-dev.ts#L84-L90
- `preset.apply`の実装は`loadAllPresets`の返り値である`getPresets`の中で定義されている
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/lib/core-common/src/presets.ts#L356-L359
- `preset.apply`では`applyPresets`の実行結果が返ってくる
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/lib/core-common/src/presets.ts#L295
  - `preset[extension]`がない場合は、引数に渡された`config`の promise が返る
  - `apply`の第一引数は`extension`で`core`を取得するには、`extension` には`core`を指定している
    - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/lib/core-server/src/build-dev.ts#L90
  - `applyPresets`に渡される`presets`の中で、extension が`core`の場合には builder と renderer が出力された

```js
{
  builder: 'storybook/sandbox/react-vite-default-ts/node_modules/@storybook/builder-vite',
  renderer: 'storybook/sandbox/react-vite-default-ts/node_modules/@storybook/react'
}
```

- `applyPresets`に渡される`presets`はどこで定義されているのか
  - getPresets では`loadPresets`の返り値
    - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/lib/core-common/src/presets.ts#LL354C29-L354C29
    - `loadPresets`では presets を`loadPreset`したものを一つの配列にまとめている
      - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/lib/core-common/src/presets.ts#L275-L293
    - `loadPreset`では`input`に渡された`'@storybook/react-vite/preset'`から`getContent`を使って内容を取得してそう
      - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/lib/core-common/src/presets.ts#L227
      - content の中身は`{ contents: { core: [Getter], viteFinal: [Getter] } }`
    - `getContent`では`'@storybook/react-vite/preset'`を`interopRequireDefault`に渡している
      - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/lib/core-common/src/presets.ts#L211-L213
    - `interopRequireDefault`では、`esbuild-register`が実行されていなければ、register を実行する
      - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/lib/core-common/src/utils/interpret-require.ts#L5-L33
      - その後、`'@storybook/react-vite/preset'`を require した結果を返す
- `@storybook/react-vite/preset`モジュールは以下のように定義されている
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/frameworks/react-vite/package.json#L29-L32
  - 実体の実装は `code/frameworks/react-vite/src/preset.ts`
- content の実態は `code/frameworks/react-vite/src/preset.ts`で export された`core`と`viteFinal`
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/frameworks/react-vite/src/preset.ts#L9
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/frameworks/react-vite/src/preset.ts#L14
- core の中に定義されている builder は`wrapForPnP('@storybook/builder-vite') as '@storybook/builder-vite'`

### esbuid-register

register 関数が実行されると Node.js で TypeScript ファイルを require や import で読み込もうとすると、esbuild を利用して実行可能な JS に変換する

https://github.com/egoist/esbuild-register

- register 関数が呼び出されると、esbuild-register は指定された設定に基づいて TypeScript ファイルの変換と実行を行うためのフックを Node.js に登録します。
- 以降、Node.js が TypeScript ファイルを `require` または `import` 文で読み込もうとすると、esbuild-register がフックとして働き、TypeScript ファイルを esbuild を使用してトランスパイルしてから実行可能な JavaScript ファイルに変換します。

## `@storybook/builder-vite`

- `code/lib/builder-vite`に実装がある
  - https://github.com/storybookjs/storybook/tree/next/code/lib/builder-vite

## TODO

`@storybook/builder-vite`(`code/lib/builder-vite`)の実装を読む

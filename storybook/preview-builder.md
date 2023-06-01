# Storybook の Preview(コンポーネントを描画してる箇所)部分はどのようにコンポーネントを描画しているのか

## 2023/06/01 作業分までのまとめ

- `.storybook/main.ts`の`framework`で指定したパッケージの preset から`builder`と`render`を決めている
- `builder`の一つである`@storybook/builder-vite`では dev サーバーに`/iframe.html`にリクエストが飛んできたら、`@storybook/builder-vite/input/iframe.html`の html をレスポンスとして返す
- この html では`./sb-preview/runtime.js`と`/virtual:/@storybook/builder-vite/vite-app.js`の二つの JS が呼び出されている。

TODO: `./sb-preview/runtime.js`は manger と preview のやり取りする用の API が提供されてそうで、コンポーネントの描画は`/virtual:/@storybook/builder-vite/vite-app.js`が行なっていそうなので、このファイルがどこでビルドされて、何をしているのかを調べるところから

## 予想

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
  builderName: "storybook/sandbox/react-vite-default-ts/node_modules/@storybook/builder-vite";
}
{
  builderPackage: "storybook/sandbox/react-vite-default-ts/node_modules/@storybook/builder-vite/dist/index.js";
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

ざっくりまとめると下記の流れ

- .storybook/main.ts から framework で指定したパッケージ名 + preset(例: `@storybook/react-vite/preset`)のモジュールを実行した結果を取得する
- 例えば `@storybook/react-vite/preset`の場合、`core`と`viteFinal`を export する
- `core`の中には`builder`と`renderer`が定義されている
- この`core.builder.name`が`getPreviewBuilder`で dynamic import されるパッケージ名になっている
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/lib/core-server/src/utils/get-builders.ts#L21
  - `core.builder`は値はプロジェクトの node_modules の該当の builder パッケージの絶対パスを返す
    - `storybook/sandbox/react-vite-default-ts/node_modules/@storybook/builder-vite`

### esbuid-register

register 関数が実行されると Node.js で TypeScript ファイルを require や import で読み込もうとすると、esbuild を利用して実行可能な JS に変換する

https://github.com/egoist/esbuild-register

- register 関数が呼び出されると、esbuild-register は指定された設定に基づいて TypeScript ファイルの変換と実行を行うためのフックを Node.js に登録します。
- 以降、Node.js が TypeScript ファイルを `require` または `import` 文で読み込もうとすると、esbuild-register がフックとして働き、TypeScript ファイルを esbuild を使用してトランスパイルしてから実行可能な JavaScript ファイルに変換します。

## `@storybook/builder-vite`

- `storybook dev`実行時、preview に関しては`previewBuilder.start()`が関連してそう
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/lib/core-server/src/dev-server.ts#L90-L97
- framework に`@storybook/react-vite`を指定すると`core`が返す`builder`は`@storybook/builder-vite`になる
- `@storybook/builder-vite`の実装は`code/builders/builder-vite`
- `previewBuilder.start()`の実装箇所
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/builders/builder-vite/src/index.ts#L61-L66
- `/sb-preview`にリクエストが来たら、`@storybook/preview/dist`の静的ファイルをレスポンスとして返すように
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/builders/builder-vite/src/index.ts#L72
- `iframeMiddleware`を登録しており、リクエストの url が`/iframe.html`の場合に`@storybook/builder-vite/input/iframe.html`のファイル内容を読み込む
  - `transformIframeHtml`で描画に必要なグローバル変数を差し込む
  - `/iframe.html`が来た時には、グローバル変数が差し込まれた`@storybook/builder-vite/input/iframe.html`が描画される
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/builders/builder-vite/src/index.ts#L44-L51
- `@storybook/builder-vite/input/iframe.html`では二つの script が読み込まれている
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/builders/builder-vite/input/iframe.html#L37-L38

```html
<script type="module" src="./sb-preview/runtime.js"></script>
<script
  type="module"
  src="/virtual:/@storybook/builder-vite/vite-app.js"
></script>
```

- `/sb-preview/runtime.js`は`@storybook/preview`で生成された JS ファイル
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/builders/builder-vite/src/index.ts#LL69C9-L69C9
  - manger の方がクライアントサイドで読み込んでいるのは`./sb-manager/runtime.js`
  - こっちは Storybook 系の API だったり、manager と preview のやり取り用のパッケージの定義？
- `/virtual:/@storybook/builder-vite/vite-app.js`が何を読み込んでいるのか
  - 実際のコンポーネントの描画はこっちがやってそう？

## `@storybook/preview`

- preview 側のクライアントサイドで読み込まれる js ファイルを生成している
- 実装は`code/lib/preview`
- `runtime.ts`では globals の情報を replace している
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/lib/preview/src/runtime.ts#L1-L9
- 実際には下記のようにクライアントサイドで import 先のファイルを上書きしてそう
  - https://github.com/storybookjs/storybook/blob/3e61433725866d3565489f1de7ced5b3510efde1/code/lib/preview/src/globals/runtime.ts#L18
- 下記のパッケージからレンダリングに関連してそうなパッケージを見ていく

```ts
import * as CHANNEL_POSTMESSAGE from "@storybook/channel-postmessage";
import * as CHANNEL_WEBSOCKET from "@storybook/channel-websocket";
import * as CHANNELS from "@storybook/channels";
import * as CLIENT_LOGGER from "@storybook/client-logger";
import * as CORE_EVENTS from "@storybook/core-events";
import * as PREVIEW_API from "@storybook/preview-api";
```

## `@storybook/preview-api`

- 実装は`code/lib/preview-api`

## `/virtual:/@storybook/builder-vite/vite-app.js`は何をやっているのか

## Builder API のドキュメントがある

https://storybook.js.org/docs/react/builders/builder-api#page-top

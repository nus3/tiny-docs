# Storybook の Preview(コンポーネントを描画してる箇所)部分はどのようにコンポーネントを描画しているのか

preview 部分では、以下二つの JS ファイルを読み込んでいる。

```html
<script type="module" src="./sb-preview/runtime.js"></script>
<script
  type="module"
  src="/virtual:/@storybook/builder-vite/vite-app.js"
></script>
```

このうち、`./sb-preview/runtime.js`は preview とのやりとりをする API を提供してそう。
`/virtual:/@storybook/builder-vite/vite-app.js`は、preview のコンポーネントを描画している。

## Builder API のドキュメント

https://storybook.js.org/docs/react/builders/builder-api#page-top

- Builder はコンポーネントとストーリーをブラウザで実行される JS バンドルにコンパイルする
- Builder API には必ず実装するメソッドがある
  - start, build, bail, getConfig, corePresets, overridePresets

## `/virtual:/@storybook/builder-vite/vite-app.js`は何をやっているのか

- Vite で`/virtual:`を使うと ESM の import 構文を使ってビルド時の情報を取得できる
  - また、仮想のモジュールを作成できる
  - https://vitejs.dev/guide/api-plugin.html#virtual-modules-convention
  - Vite のプラグインを使ってるっぽい
    - Virtual Modules では必ず ID の指定をしている
  - `@storybook/builder-vite`でも ID の指定をしている
    - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/builders/builder-vite/src/virtual-file-names.ts#L1-L4
- `/virtual:/@storybook/builder-vite/vite-app.js`は`virtualFileId`として定義されているので`virtualFileId`が使われているところを読む
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/builders/builder-vite/src/virtual-file-names.ts#LL1C1-L1C1
- `virtualFileId`は`code/builders/builder-vite/src/plugins/code-generator-plugin.ts`で使われている
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/builders/builder-vite/src/plugins/code-generator-plugin.ts#L33
  - ここでは`codeGeneratorPlugin`が定義されている
- `codeGeneratorPlugin`は`code/builders/builder-vite/src/vite-config.ts`にある`pluginConfig`で使われている
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/builders/builder-vite/src/vite-config.ts#LL79C8-L79C8
  - この`pluginConfig`は最終的に`commonConfig`の`plugins`として使われる
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/builders/builder-vite/src/vite-config.ts#L58
- dev モードでの起動(start)時に`createViteServer`で Vite サーバーが作成されるが、その中で`commonConfig`が作成される
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/builders/builder-vite/src/vite-server.ts#L11

## ここまでのまとめ

1. preview には`/virtual:/@storybook/builder-vite/vite-app.js`が読み込まれる
2. `/virtual:/@storybook/builder-vite/vite-app.js`は Vite の plugin で定義された仮想モジュールを読み込む
3. plugin が定義されているのは`code/builders/builder-vite/src/vite-config.ts`の`pluginConfig`である
4. この plugin を dev モードの場合は`createViteServer`で使われる
5. 仮想モジュールは`code/builders/builder-vite/src/plugins/code-generator-plugin.ts`で定義されている

## `codeGeneratorPlugin`は何をしているのか

- `codeGeneratorPlugin`は`code/builders/builder-vite/src/plugins/code-generator-plugin.ts`で定義されている

### `/virtual:/@storybook/builder-vite/vite-app.js`

- Vite の plugin の `load` から、`/virtual:/@storybook/builder-vite/vite-app.js`(`virtualFileId`)が読み込まれた場合は、storyStoreV7(必要に応じてストーリをオンデマンドで読み込む設定)の場合は`generateModernIframeScriptCode`が実行される
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/builders/builder-vite/src/plugins/code-generator-plugin.ts#L109-L114
- `generateModernIframeScriptCode`は`code/builders/builder-vite/src/codegen-modern-iframe-script.ts`で定義されている
- `@storybook/preview-api`の`composeConfigs`に addon や stories、`/.storybook/preview.ts`のモジュールを dynamic import したものを渡すコードを生成している
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/builders/builder-vite/src/codegen-modern-iframe-script.ts#L23-L29
  - 以下は実際に生成されたコード

```js
const getProjectAnnotations = async () => {
  const configs = await Promise.all([
    import("@storybook/react/preview"),
    import("@storybook/addon-links/preview"),
    import("@storybook/addon-essentials/docs/preview"),
    import("@storybook/addon-essentials/actions/preview"),
    import("@storybook/addon-essentials/backgrounds/preview"),
    import("@storybook/addon-essentials/measure/preview"),
    import("@storybook/addon-essentials/outline/preview"),
    import("@storybook/addon-essentials/highlight/preview"),
    import("@storybook/addon-interactions/preview"),
    import("/src/stories/components"),
    import("/template-stories/lib/preview-api/preview.ts"),
    import("/template-stories/addons/toolbars/preview.ts"),
    import("/.storybook/preview.ts"),
  ]);
  return composeConfigs(configs);
};
```

- ホットリロード用の設定を生成している
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/builders/builder-vite/src/codegen-modern-iframe-script.ts#L32-L54
- 生成される大元のコードでは、`@storybook/preview-api`、`'/virtual:/@storybook/builder-vite/setup-addons.js'`、`'/virtual:/@storybook/builder-vite/storybook-stories.js'`が import される
- グローバル変数である`window.__STORYBOOK_PREVIEW__`に`'@storybook/preview-api'`の`PreviewWeb`を格納する
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/builders/builder-vite/src/codegen-modern-iframe-script.ts#L69
- `window.__STORYBOOK_PREVIEW__.initialize({ importFn, getProjectAnnotations })`で画面の初期描画をしてそう？
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/builders/builder-vite/src/codegen-modern-iframe-script.ts#L73
  - `PreviewWeb().initialize({ importFn, getProjectAnnotations })`が初回の描画で使われてそう

### `'/virtual:/@storybook/builder-vite/storybook-stories.js'`

- `'/virtual:/@storybook/builder-vite/storybook-stories.js'`が読み込まれた場合は`generateImportFnScriptCode`が実行されている。
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/builders/builder-vite/src/plugins/code-generator-plugin.ts#L93-L98
- `generateImportFnScriptCode`は`code/builders/builder-vite/src/codegen-importfn-script.ts`で定義されている
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/builders/builder-vite/src/codegen-importfn-script.ts#L50-L56
  - stories には、stories すべての絶対パスが格納されている
  - その後、`toImportFn`を実行して、各ストーリーを格納されたパスから dynamic import する関数を作成している
    - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/builders/builder-vite/src/codegen-importfn-script.ts#L36
  - 最終的に`importFn`を export する

```ts
{
  stories: [
    "storybook/sandbox/react-vite-default-ts/src/stories/Introduction.mdx",
    "storybook/sandbox/react-vite-default-ts/src/stories/renderers/react/react-mdx.stories.mdx",
    "storybook/sandbox/react-vite-default-ts/src/stories/Page.stories.ts",
    "storybook/sandbox/react-vite-default-ts/src/stories/Header.stories.ts",
    "storybook/sandbox/react-vite-default-ts/src/stories/Button.stories.ts",
  ];
}
```

### `'/virtual:/@storybook/builder-vite/setup-addons.js'`

- `'/virtual:/@storybook/builder-vite/setup-addons.js'`では、`generateAddonSetupCode`が実行される
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/builders/builder-vite/src/plugins/code-generator-plugin.ts#L101-L103
- `generateAddonSetupCode`では`window.__STORYBOOK_ADDONS_CHANNEL__`に`@storybook/channel-postmessage`の`createChannel({ page: 'preview' })`を渡している
  - dev モードの場合は、`window.__STORYBOOK_SERVER_CHANNEL__`に`@storybook/channel-websocket`の`createChannel({})`を渡している
  - https://github.com/storybookjs/storybook/blob/next/code/builders/builder-vite/src/codegen-set-addon-channel.ts

### `@storybook/preview-api`

- `PreviewWeb().initialize({ importFn, getProjectAnnotations })`が初期描画時に行われてそう
- 実装は`code/lib/preview-api/package.json`
- `PreviewWeb`の実装は`code/lib/preview-api/src/modules/preview-web/PreviewWeb.tsx`
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/lib/preview-api/src/modules/preview-web/PreviewWeb.tsx#L9-L15
  - `UrlStore`と`WebView`を`PreviewWithSelection`クラスのコンストラクタに渡している
- `PreviewWithSelection`では受け取った`UrlStore`を`selectionStore`に格納、`WebView`を`view`に格納している
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/lib/preview-api/src/modules/preview-web/PreviewWithSelection.tsx#L89-L94
  - また継承元の`Preview`クラスのコンストラクタを実行している
- `Preview`クラスに初期描画時に実行される`PreviewWeb().initialize({ importFn, getProjectAnnotations })`に該当する`initialize`メソッドがある
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/lib/preview-api/src/modules/preview-web/Preview.tsx#L75-L79
- `initialize`メソッドでは各ストーリーをパスから dynamic import する`importFn`をメンバ変数に格納し、`this.setupListeners()`を実行
  - `setupListeners`では channel に対して、必要なものを bind している
    - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/lib/preview-api/src/modules/preview-web/Preview.tsx#L98-L106
- また`getProjectAnnotationsOrRenderError`を実行した後に、`initializeWithProjectAnnotations`を実行
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/lib/preview-api/src/modules/preview-web/Preview.tsx#L93-L95
- `getProjectAnnotationsOrRenderError`では非同期の処理を同期的に扱える`SynchronousPromise`を使って、`getProjectAnnotations`の中から、`renderToCanvas`を取得し、メンバ変数に格納
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/lib/preview-api/src/modules/preview-web/Preview.tsx#L117
  - `renderToCanvas`は`'@storybook/preview-api'`の`composeConfigs`に定義されている
    - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/lib/preview-api/src/modules/store/csf/composeConfigs.ts#LL61C8-L61C8
- `initializeWithProjectAnnotations`では、StoryIndex を取得して`initializeWithStoryIndex`を実行する
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/lib/preview-api/src/modules/preview-web/Preview.tsx#L154
- `initializeWithStoryIndex`では、`this.storyStore.initialize()`を実行する
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/lib/preview-api/src/modules/preview-web/Preview.tsx#L190-L194
- `this.storyStore.initialize()`は`code/lib/preview-api/src/modules/store/StoryStore.ts`に実装がある
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/lib/preview-api/src/modules/store/StoryStore.ts#L100-L108
  - 初期化処理では StoryIndex を格納はするが描画処理はしてなさそう
    - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/lib/preview-api/src/modules/store/StoryStore.ts#L109-L115
- PreviewWeb のテストコードを見る感じ、`initialize`でレンダリングまでいけてそうな気がするんだけどもな
  - https://github.com/storybookjs/storybook/blob/ec249113e0890ea0935ff8c6f56e8923c107e7eb/code/lib/preview-api/src/modules/preview-web/PreviewWeb.test.ts#L106-L121

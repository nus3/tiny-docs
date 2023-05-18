# code ディレクトリを見る

- 実行されるのは`storybook`というコマンド
- これは`code/lib/cli-storybook`が該当する
  - `"storybook": "portal:storybook/code/lib/cli-storybook",`
- `cli-storybook`は README を見ると@storybook/cli の wrapper と記載されてる
  - `"@storybook/cli": "portal:storybook/code/lib/cli"`
- sandbox 環境で実行される storybook コマンドは dev と build がある

## ディレクトリ構造

```
.
├── __mocks__
├── addons
│   ├── a11y
│   ├── actions
│   ├── backgrounds
│   ├── controls
│   ├── docs
│   ├── essentials
│   ├── gfm
│   ├── highlight
│   ├── interactions
│   ├── jest
│   ├── links
│   ├── measure
│   ├── outline
│   ├── storyshots-core
│   ├── storyshots-puppeteer
│   ├── storysource
│   ├── toolbars
│   └── viewport
├── e2e-tests
├── frameworks
│   ├── angular
│   ├── ember
│   ├── html-vite
│   ├── html-webpack5
│   ├── nextjs
│   ├── preact-vite
│   ├── preact-webpack5
│   ├── react-vite
│   ├── react-webpack5
│   ├── server-webpack5
│   ├── svelte-vite
│   ├── svelte-webpack5
│   ├── sveltekit
│   ├── vue-vite
│   ├── vue-webpack5
│   ├── vue3-vite
│   ├── vue3-webpack5
│   ├── web-components-vite
│   └── web-components-webpack5
├── lib
│   ├── addons
│   ├── builder-manager
│   ├── builder-vite
│   ├── builder-webpack5
│   ├── channel-postmessage
│   ├── channel-websocket
│   ├── channels
│   ├── cli
│   ├── cli-sb
│   ├── cli-storybook
│   ├── client-api
│   ├── client-logger
│   ├── codemod
│   ├── core-client
│   ├── core-common
│   ├── core-events
│   ├── core-server
│   ├── core-webpack
│   ├── csf-plugin
│   ├── csf-tools
│   ├── docs-tools
│   ├── instrumenter
│   ├── manager-api
│   ├── manager-api-shim
│   ├── node-logger
│   ├── postinstall
│   ├── preview
│   ├── preview-api
│   ├── preview-web
│   ├── react-dom-shim
│   ├── router
│   ├── source-loader
│   ├── store
│   ├── telemetry
│   ├── theming
│   └── types
├── presets
│   ├── create-react-app
│   ├── html-webpack
│   ├── preact-webpack
│   ├── react-webpack
│   ├── server-webpack
│   ├── svelte-webpack
│   ├── vue-webpack
│   ├── vue3-webpack
│   └── web-components-webpack
├── renderers
│   ├── html
│   ├── preact
│   ├── react
│   ├── server
│   ├── svelte
│   ├── vue
│   ├── vue3
│   └── web-components
└── ui
    ├── blocks
    ├── components
    └── manager

```

## storybook コマンドの流れ

- storybook コマンドを実行する
- `code/lib/cli-storybook`の index.js が実行される
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/cli-storybook/index.js#L3
- `@storybook/cli/bin/index`を実行してる
  - `code/lib/cli/bin/index.js`では`../dist/generate.js`を require してる
  - 実装は`code/lib/cli/src/generate.ts`が該当
  - storybook コマンドで実行する dev と build が定義されてそう
    - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/cli/src/generate.ts#L184
    - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/cli/src/generate.ts#L241
- `storybook dev`は`code/lib/cli/src/dev.ts`が該当
  - サーバーを起動してそうな`buildDevStandalone`は`@storybook/core-server`から export されている
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/cli/src/dev.ts#L49-L51
- `@storybook/core-server`は `code/lib/core-server`が該当
  - `buildDevStandalone`の中を見ていくとサーバーの起動は `storybookDevServer`
    - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-server/src/build-dev.ts#L127-L129
- `storybookDevServer`は`code/lib/core-server/src/dev-server.ts`で定義されている

## storybook

- `code/lib/cli-storybook/package.json`が該当
- `storybook`コマンドは package.json の`bin`に定義されている
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/cli-storybook/package.json#L22-L25

```json
  "bin": {
    "sb": "./index.js",
    "storybook": "./index.js"
  },
```

- index.js では`require('@storybook/cli/bin/index');`がされているのみ

## @storybook/cli

[README](https://github.com/storybookjs/storybook/blob/next/code/lib/cli/README.md)

- `code/lib/cli`が該当
- storybook コマンドを実行すると`code/lib/cli/bin/index.js`が実行される
  - これは`../dist/generate.js`を require してる
- prep コマンドで`../../../scripts/prepare/bundle.ts`が実行されており、package.json を見る感じ、`../dist/generate.js`は`./src/generate.ts`が該当しそう

```json
  "bundler": {
    "entries": [
      "./src/generate.ts",
      "./src/index.ts"
    ],
    "platform": "node"
  },
```

- `code/lib/cli/src/generate.ts`の中を見てく
- storybook コマンドで実行する dev と build が定義されてそう
- [`commander`](https://github.com/tj/commander.js)というパッケージが使われている

```ts
command("dev")
  .option("-p, --port <number>", "Port to run Storybook", (str) =>
    parseInt(str, 10)
  )
  // ...
  .action(async (options) => {
    logger.setLevel(program.loglevel);
    consoleLogger.log(
      chalk.bold(`${pkg.name} v${pkg.version}`) + chalk.reset("\n")
    );

    // The key is the field created in `options` variable for
    // each command line argument. Value is the env variable.
    getEnvConfig(options, {
      port: "SBCONFIG_PORT",
      host: "SBCONFIG_HOSTNAME",
      staticDir: "SBCONFIG_STATIC_DIR",
      configDir: "SBCONFIG_CONFIG_DIR",
      ci: "CI",
    });

    if (parseInt(`${options.port}`, 10)) {
      // eslint-disable-next-line no-param-reassign
      options.port = parseInt(`${options.port}`, 10);
    }

    await dev({ ...options, packageJson: pkg }).catch(() => process.exit(1));
  });
```

- `storybook dev`は`code/lib/cli/src/dev.ts`が該当

```ts
export const dev = async (cliOptions: any) => {
  // optionとかの値が定義されてる
  // ...
  await withTelemetry(
    "dev",
    { cliOptions, presetOptions: options, printError },
    () => buildDevStandalone(options)
  );
};
```

- `withTelemetry`と`buildDevStandalone`はどちらも`@storybook/core-server`から import されてる

## @storybook/core-server

[README](https://github.com/storybookjs/storybook/blob/next/code/lib/core-server/README.md)

- `code/lib/core-server`が該当
- package.json から`./src/index.ts`がエントリーポイントっぽい

```json
  "bundler": {
    "entries": [
      "./src/index.ts",
      "./src/presets/babel-cache-preset.ts",
      "./src/presets/common-preset.ts"
    ],
    "platform": "node"
  },
```

- `withTelemetry` の実装は`code/lib/core-server/src/withTelemetry.ts`
- `withTelemetry`では`@storybook/telemetry`の telemetry 関数が実行される
  - telemetry という名前なので storybook 起動時のデータの収集とかしてる？

```ts
if (!options.cliOptions.disableTelemetry)
  telemetry("boot", { eventType }, { stripMetadata: true });
```

- その後、第三引数で渡された関数を実行する

```ts
export async function withTelemetry<T>(
  eventType: EventType,
  options: TelemetryOptions,
  run: () => Promise<T>
): Promise<T> {
  try {
    return await run();
  } catch (error) {
    // エラーハンドリング
  }
}
```

- `storybook dev`実行時の実行時の第 3 引数は`buildDevStandalone`

```ts
await withTelemetry(
  "dev",
  { cliOptions, presetOptions: options, printError },
  () => buildDevStandalone(options)
);
```

- `buildDevStandalone`は`code/lib/core-server/src/build-dev.ts`が該当
- dev サーバーの起動は`storybookDevServer`でやってそう

```ts
const { address, networkAddress, managerResult, previewResult } =
  await storybookDevServer(fullOptions);
```

- `storybookDevServer`は`code/lib/core-server/src/dev-server.ts`が該当
- `express` を使ってサーバーが起動されてそう
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-server/src/dev-server.ts#L24
- express の `app.use` はミドルウェアを設定してる
- `compression` はリクエストのレスポンスボディの圧縮を試みる
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-server/src/dev-server.ts#LL45C4-L45C4
- レスポンスのヘッダに`Access-Control-Allow-Origin`などを付与、SharedArrayBuffer が使えるようにしてる
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-server/src/dev-server.ts#L51
- キャッシュしないように
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-server/src/dev-server.ts#L52
- サーバーの起動はこれ
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-server/src/dev-server.ts#L64
  - https.createServer でサーバーを起動してる(セキュアな HTTPS 接続をするため)
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-server/src/utils/server-init.ts#L36
- core の値
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-server/src/dev-server.ts#L67

```js
{
  builderName: 'storybook/sandbox/react-vite-default-ts/node_modules/@storybook/builder-vite',
  core: {
    disableTelemetry: false,
    enableCrashReports: undefined,
    builder: 'storybook/sandbox/react-vite-default-ts/node_modules/@storybook/builder-vite',
    renderer: 'storybook/sandbox/react-vite-default-ts/node_modules/@storybook/react'
  }
}
```

- core の値は`options.presets.apply<CoreConfig>('core')`の返り値
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-server/src/dev-server.ts#L29
  - presets は`code/lib/core-server/src/build-dev.ts`に定義されている
    - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-server/src/build-dev.ts#L105
- framework を storybook 利用側の main.ts から取得
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-server/src/build-dev.ts#L74-L75
- builder と renderer は code/frameworks で定義されてる
  - 例えば framework に`@storybook/react-vite`を指定した場合
  - 該当は`code/frameworks/react-vite/src/preset.ts`
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/frameworks/react-vite/src/preset.ts#L9
- `loadAllPresets`で preset を load する
- `@storybook/react-vite/preset`のような preset のパスを受け取り、実際に定義してあるモジュールを require して、core と viteFinal を取得してそう
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-common/src/presets.ts#L227

```js
{
  contents: { core: [Getter], viteFinal: [Getter] },
  input: '@storybook/react-vite/preset'
}
```

- `'@storybook/react-vite/preset'`の場合は下記が該当
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/frameworks/react-vite/src/preset.ts#L9
- `getStoryIndexGenerator()`は stories を抽出したりする役割？
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-server/src/dev-server.ts#L36
  - 本体の実装は下記
    - https://github.com/storybookjs/storybook/blob/next/code/lib/core-server/src/utils/StoryIndexGenerator.ts
  - initialize()では stories で指定されたパスに story のファイルがあるかどうか、あった場合にはキャッシュに内容を格納するように見たいなことしてそう
- Manager app に関連しそうな部分を見る
  - おそらく`managerBuilder.start`で表示されてるはず
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-server/src/dev-server.ts#L79
- `managerBuilder`は下記で定義されている
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-server/src/utils/get-builders.ts#L4
  - `@storybook/builder-manager`が import されている
- `@storybook/builder-manager`の実装は`code/lib/builder-manager/package.json`
- `managerBuilder.start`の実装は下記が該当
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/builder-manager/src/index.ts#L288
- `start`関数は`starter`というジェネレーター関数(`function*` 宣言)になってる
  - ジェネレーター関数では、`next()`を実行すると、`yield`宣言されている場所まで処理を進めて、そこで止まる
  - もういちど`next()`を実行すると、前回`yield`で止まった位置から次に`yield`宣言されている箇所まで処理を実行する
  - start 関数では yield で true が返ってくるまで`next()`が実行される
  - ジェネレーターになっているのは preview builder などでエラーが発生したときにプロセスが中止できるようにするためとのこと
- `starter`関数の定義はここ
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/builder-manager/src/index.ts#L122
  - 一度目は必要な情報を preset や option から取得
  - addon の output ディレクトリを削除することで、キャッシュによる問題が起きないようにしてる
  - `compilation`は`instance`の返り値であり、この`instance`は`executor.get()`
    - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/builder-manager/src/utils/data.ts#L24
    - `executor.get()`は`esbuild`
      - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/builder-manager/src/index.ts#L111
  - esbuild に渡しているコンフィグは`getConfig`から生成
    - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/builder-manager/src/index.ts#L45
    - config の内容
    - compilation は addon をバンドルしてそう？

```js
{
  config: {
    entryPoints: [
      'storybook/sandbox/react-vite-default-ts/node_modules/.cache/sb-manager/links-0/manager-bundle.js',
      'storybook/sandbox/react-vite-default-ts/node_modules/.cache/sb-manager/essentials-controls-1/manager-bundle.js',
      'storybook/sandbox/react-vite-default-ts/node_modules/.cache/sb-manager/essentials-actions-2/manager-bundle.js',
      'storybook/sandbox/react-vite-default-ts/node_modules/.cache/sb-manager/essentials-backgrounds-3/manager-bundle.js',
      'storybook/sandbox/react-vite-default-ts/node_modules/.cache/sb-manager/essentials-viewport-4/manager-bundle.js',
      'storybook/sandbox/react-vite-default-ts/node_modules/.cache/sb-manager/essentials-toolbars-5/manager-bundle.js',
      'storybook/sandbox/react-vite-default-ts/node_modules/.cache/sb-manager/essentials-measure-6/manager-bundle.js',
      'storybook/sandbox/react-vite-default-ts/node_modules/.cache/sb-manager/essentials-outline-7/manager-bundle.js',
      'storybook/sandbox/react-vite-default-ts/node_modules/.cache/sb-manager/interactions-8/manager-bundle.js'
    ],
    outdir: 'storybook/sandbox/react-vite-default-ts/node_modules/.cache/storybook/public/sb-addons',
    format: 'esm',
    write: false,
    ignoreAnnotations: true,
    resolveExtensions: [ '.ts', '.tsx', '.mjs', '.js', '.jsx' ],
    outExtension: { '.js': '.js' },
    loader: {
      '.js': 'jsx',
      '.png': 'dataurl',
      '.gif': 'dataurl',
      '.jpg': 'dataurl',
      '.jpeg': 'dataurl',
      '.svg': 'dataurl',
      '.webp': 'dataurl',
      '.webm': 'dataurl',
      '.woff2': 'dataurl'
    },
    target: [ 'chrome100' ],
    platform: 'browser',
    bundle: true,
    minify: true,
    sourcemap: true,
    conditions: [ 'browser', 'module', 'default' ],
    jsxFactory: 'React.createElement',
    jsxFragment: 'React.Fragment',
    jsx: 'transform',
    jsxImportSource: 'react',
    tsconfig: 'storybook/sandbox/react-vite-default-ts/node_modules/@storybook/builder-manager/templates/addon.tsconfig.json',
    legalComments: 'external',
    plugins: [ [Object], [Object], [Object] ],
    banner: { js: 'try{' },
    footer: {
      js: '}catch(e){ console.error("[Storybook] One of your manager-entries failed: " + import.meta.url, e); }'
    },
    define: {
      'process.env': '{"NODE_ENV":"development","NODE_PATH":[],"STORYBOOK":"true","PUBLIC_URL":".","STORYBOOK_TELEMETRY_URL":"http://localhost:6007/event-log"}',
      'process.env.NODE_ENV': '"development"',
      'process.env.NODE_PATH': '[]',
      'process.env.STORYBOOK': '"true"',
      'process.env.PUBLIC_URL': '"."',
      'process.env.STORYBOOK_TELEMETRY_URL': '"http://localhost:6007/event-log"',
      global: 'window',
      module: '{}'
    }
  }
}
```

- `/sb-addons`には`addonsDir`、`/sb-manager`には`coreDirOrigin`を共に静的ファイルとして、express.static で提供してる
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/builder-manager/src/index.ts#L159-L160
  - coreDirOrigin は `@storybook/manager`が該当
    - `code/ui/manager`に実装がある
    - `coreDirOrigin: '/storybook/sandbox/react-vite-default-ts/node_modules/@storybook/manager/dist'`
- compilation した outputFiles から js と css を取得する
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/builder-manager/src/index.ts#L162
  - addon 関連のファイルが入ってそう

```js
{
  cssFiles: [],
  jsFiles: [
    './sb-addons/links-0/manager-bundle.js',
    './sb-addons/essentials-controls-1/manager-bundle.js',
    './sb-addons/essentials-actions-2/manager-bundle.js',
    './sb-addons/essentials-backgrounds-3/manager-bundle.js',
    './sb-addons/essentials-viewport-4/manager-bundle.js',
    './sb-addons/essentials-toolbars-5/manager-bundle.js',
    './sb-addons/essentials-measure-6/manager-bundle.js',
    './sb-addons/essentials-outline-7/manager-bundle.js',
    './sb-addons/interactions-8/manager-bundle.js'
  ]
}
```

- addon や template などを渡して、html をレンダリングする
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/builder-manager/src/index.ts#L166
  - ejs でレンダリングしてる
    - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/builder-manager/src/utils/template.ts#L4
  - ベースの template は `code/lib/builder-manager/templates/template.ejs`
    - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/builder-manager/src/utils/data.ts#L18
  - template では global に値を入れたり、addon の js ファイルと、`./sb-manager/runtime.js`を import している
    - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/builder-manager/templates/template.ejs#L49-L65
- `./sb-manager/runtime.js`は`coreDirOrigin: '/storybook/sandbox/react-vite-default-ts/node_modules/@storybook/manager/dist'`で生成された js ファイルが提供されている
  - `@storybook/manager`の定義がある`code/ui/manager`を見る
    - README
      - https://github.com/storybookjs/storybook/blob/next/code/ui/manager/README.md
- `./sb-manager/runtime.js`は`code/ui/manager/src/runtime.ts`が該当
  - ベースは React を使ってる
  - renderStorybookUI でレンダリングしてそう
    - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/ui/manager/src/runtime.ts#L61
- React を render してる
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/ui/manager/src/index.tsx#L76

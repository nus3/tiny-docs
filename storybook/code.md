# code ディレクトリを見る

- 実行されるのは`storybook`というコマンド
- これは`code/lib/cli-storybook`が該当する
  - `"storybook": "portal:/Users/nus3/dev/fork/storybook/code/lib/cli-storybook",`
- `cli-storybook`は README を見ると@storybook/cli の wrapper と記載されてる
  - `"@storybook/cli": "portal:/Users/nus3/dev/fork/storybook/code/lib/cli"`
- sandbox 環境で実行される storybook コマンドは dev と build がある

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
  builderName: '/Users/nus3/dev/fork/storybook/sandbox/react-vite-default-ts/node_modules/@storybook/builder-vite',
  core: {
    disableTelemetry: false,
    enableCrashReports: undefined,
    builder: '/Users/nus3/dev/fork/storybook/sandbox/react-vite-default-ts/node_modules/@storybook/builder-vite',
    renderer: '/Users/nus3/dev/fork/storybook/sandbox/react-vite-default-ts/node_modules/@storybook/react'
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

TODO: builder-manager の start 関数の実装を見るところから

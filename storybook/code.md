# code ディレクトリを見る

- 実行されるのは`storybook`というコマンド
- これは`code/lib/cli-storybook`が該当する
  - `"storybook": "portal:/Users/nus3/dev/fork/storybook/code/lib/cli-storybook",`
- `cli-storybook`は README を見ると@storybook/cli の wrapper と記載されてる
  - `"@storybook/cli": "portal:/Users/nus3/dev/fork/storybook/code/lib/cli"`
- sandbox 環境で実行される storybook コマンドは dev と build がある

## storybook

- `code/lib/cli-storybook/package.json`が該当
- `storybook`コマンドは package.json の`bin`に定義されている

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

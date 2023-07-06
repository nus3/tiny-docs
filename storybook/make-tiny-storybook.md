# 小さな Storybook を作る

- `storybook dev`の実装は`code/lib/cli/src/dev.ts`
  - https://github.com/storybookjs/storybook/blob/6d0596f8b2462e6e0ae9d9dd72792e361c437878/code/lib/cli/src/dev.ts#L39-L60
- options には packageJson がとりあえずあれば良さそう？
  - https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/types/src/modules/core-common.ts#LL127C12-L127C12
  - ダメだった
- cli option をフルで入れるか
- dev 実行時の cli options はここから
  - https://github.com/storybookjs/storybook/blob/6d0596f8b2462e6e0ae9d9dd72792e361c437878/code/lib/cli/src/generate.ts#L184-L243

## `storybook dev`の起動まとめ

- sandbox 環境で`storybook dev -p 6006`コマンドが実行される
- `storybook`モジュール(`code/lib/cli-storybook`)に`storybook`コマンドが定義されている
  - https://github.com/storybookjs/storybook/blob/6d0596f8b2462e6e0ae9d9dd72792e361c437878/code/lib/cli-storybook/package.json#L22-L24
- `storybook`コマンドでは`@storybook/cli/bin/index`を`require`してるだけ
  - https://github.com/storybookjs/storybook/blob/6d0596f8b2462e6e0ae9d9dd72792e361c437878/code/lib/cli-storybook/index.js#LL3C10-L3C34
- `@storybook/cli/bin/index`では`code/lib/cli/src/generate.ts`を実行している
  - https://github.com/storybookjs/storybook/blob/6d0596f8b2462e6e0ae9d9dd72792e361c437878/code/lib/cli/bin/index.js#LL9C32-L9C32
  - https://github.com/storybookjs/storybook/blob/next/code/lib/cli/src/generate.ts
- `code/lib/cli/src/generate.ts`の中に dev コマンドが登録されている
  - https://github.com/storybookjs/storybook/blob/6d0596f8b2462e6e0ae9d9dd72792e361c437878/code/lib/cli/src/generate.ts#L184-L243

## 作業メモ

- getPreviewBuilder 周りでエラーが出る

```
TypeError: (0 , import_node_url.pathToFileURL).then is not a function
    at getPreviewBuilder (/Users/nus3/dev/me/tiny-storybook/node_modules/.pnpm/@storybook+core-server@7.0.23/node_modules/@storybook/core-server/dist/index.js:20:1987)
    at buildDevStandalone (/Users/nus3/dev/me/tiny-storybook/node_modules/.pnpm/@storybook+core-server@7.0.23/node_modules/@storybook/core-server/dist/index.js:59:2082)
    at withTelemetry (/Users/nus3/dev/me/tiny-storybook/node_modules/.pnpm/@storybook+core-server@7.0.23/node_modules/@storybook/core-server/dist/index.js:46:3620)
```

- `storybook dev`にはポートぐらいしか渡してなさそう
- error の内容が`pathToFileURL`なので、`getPreviewBuilder`
  - https://github.com/storybookjs/storybook/blob/6d0596f8b2462e6e0ae9d9dd72792e361c437878/code/lib/core-server/src/utils/get-builders.ts#L21
- storybook のコード実行は`ts-node --swc ./task.ts`、ts-node で swc を噛ませてやってる
- `read-pkg-up`は esm 対応されちゃうので、"7.0.1"にすること
- tsx と ts-node + swc は実行結果変わるのでホゲー
- args なさそう
- http://localhost:3000/iframe.html?args=&id=example-button--primary&viewMode=story
- feature に storeyStorev7 入れたら、`code/lib/core-server/src/dev-server.ts`で`getStoryIndexGenerator`の処理を知れないといけないかも
- `slash`は下記のエラーを吐く
  - cjs でやろうとしてるのに esm でエラー吐く的な？

```
Error [ERR_REQUIRE_ESM]: require() of ES Module /Users/nus3/dev/me/tiny-storybook/node_modules/.pnpm/slash@5.1.0/node_modules/slash/index.js from /Users/nus3/dev/me/tiny-storybook/src/storyIndexGenerator/watch-story-specifiers.ts not supported.
Instead change the require of index.js in /Users/nus3/dev/me/tiny-storybk/src/storyIndexGenerator/watch-story-specifiers.ts to a dynamic import() which is available in all CommonJS modules.
```

- Vite 環境で、`import('/@fs/${file}')`のように`@fs`をつけた状態で import を実行するとソースファイルにあるモジュールを直接 dynamic import できる
  - https://vitejs.dev/guide/#index-html-and-project-root
  - https://vitejs.dev/config/server-options.html#server-fs-allow
- framework で対応しているものの一覧
  - https://github.com/storybookjs/storybook/tree/next/code/frameworks

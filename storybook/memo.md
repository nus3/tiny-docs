# Storybook

TODO: どのような仕組みで Storybook が動いてるのか`yarn start`を起点に見てく

## Contributing 読む

https://github.com/nus3/storybook/tree/next#contributing
https://github.com/nus3/storybook/blob/next/CONTRIBUTING.md

- Lerna を使ってモノレポの管理してる
- Node.js は v16 を使うこと(16.5 を推奨)
- `yarn start`で基本的なテストが実行される Sandbox を作る？
  - React Vite TypeScript のサンドボックスとその中のテストストーリーをセットを生成し、それを実行するために必要な全てのステップを実行する
- `yarn start`時に必要な依存の解決をしてくれるので、`yarn`や`yarn install`しなくてもいい
- `yarn start`は下記のエラーになって実行できない
  - `Cannot link @storybook/react-vite into before-storybook@workspace:. dependency @vitejs/plugin-react@npm:3.1.0 [c88a5] conflicts with parent dependency @vitejs/plugin-react@npm:4.0.0`
- `/sandbox`配下にプロジェクトが生成されてる
  - `/sandbox`は gitignore されてる

### `yarn start`時のログ

```
🥾 Bootstrapping
> nx run-many --target="prep" --all --parallel --max-parallel=9 --exclude=@storybook/addon-storyshots,@storybook/addon-storyshots-puppeteer -- --reset
✅ install > ✅ compile > 🔄 sandbox > 🔇 dev

🗑  Removing old sandbox dir

👷 Bootstrapping Template
> node /fork/storybook/code/lib/cli/bin/index.js repro react-vite/default-ts --output /fork/storybook/sandbox/react-vite-default-ts --branch next --no-init

🧶 Installing Yarn 2
> touch yarn.lock
> touch .yarnrc.yml
> yarn set version berry
> yarn config set enableGlobalCache true
> yarn config set checksumBehavior ignore
> yarn config set nodeLinker node-modules

🔗 Linking packages
> node /fork/storybook/code/lib/cli/bin/index.js link /fork/storybook/sandbox/react-vite-default-ts --local --no-start

⚙️ Initializing Storybook
> node /fork/storybook/code/lib/cli/bin/index.js init --yes
```

エラーになる

```
shortMessage: 'Command failed with exit code 1: node /fork/storybook/code/lib/cli/bin/index.js init --yes',
  command: 'node /fork/storybook/code/lib/cli/bin/index.js init --yes',
  escapedCommand: 'node "/fork/storybook/code/lib/cli/bin/index.js" init --yes',
  exitCode: 1,
  signal: undefined,
  signalDescription: undefined,
  stdout: '\n' +
    ' storybook init - the simplest way to add a Storybook to your project. \n' +
    '\n' +
    ' • Detecting project type. ✓\n' +
    '    Detected Vite project. Setting builder to Vite\n' +
    ' • Adding Storybook support to your "React" app\n' +
    '➤ YN0000: ┌ Resolution step\n' +
    '➤ YN0000: └ Completed in 5s 746ms\n' +
    '➤ YN0000: ┌ Fetch step\n' +
    '➤ YN0000: └ Completed in 0s 207ms\n' +
    '➤ YN0000: ┌ Link step\n' +
    '➤ YN0071: │ Cannot link @storybook/react-vite into before-storybook@workspace:. dependency @vitejs/plugin-react@npm:3.1.0 [c88a5] conflicts with parent dependency @vitejs/plugin-react@npm:4.0.0 [1b0ac]\n' +
    '➤ YN0000: └ Completed in 0s 340ms\n' +
    '➤ YN0000: Failed with errors in 6s 383ms\n',
  stderr: 'An error occurred while installing dependencies.',
```

## `yarn start`の中身を見る

- `scripts`に移動し、`yarn install`を実行(この時、`>/dev/null`で標準出力を破棄)。その後`yarn task`を実行
- `yarn task`では`ts-node --swc ./task.ts`が実行される
  - `--swc`を渡すことで、swc でトランスパイルされる
- `scripts/task.ts`は`run()`が実行されてる
- cli は[prompts](https://github.com/terkelg/prompts)を使ってる
- 必要な task の管理とかしつつ、task 自体の実行は`runTask()`でやってそう
- 実行される全てのタスクは Task の型定義になっており、task の実行自体は`run()`を呼べばできる
- `scripts/tasks`配下のモジュールを見るとタスクの実装の詳細が確認できる
- `✅ install > ✅ compile > 🔄 sandbox > 🔇 dev`
- の部分でエラーが出るので、sandbox のタスクがなんかエラー出てる？
  - 🚨 Linking packages failed
- 🔗 Linking packages を実行してる時にエラーが出る
  - `scripts/utils/cli-step.ts`の`steps.link`が該当
  - これは`scripts/tasks/sandbox-parts.ts`で定義されてる
- `scripts/tasks/sandbox.ts`を見てみる
  - `scripts/tasks/sandbox-parts.ts`から`create`と`install`と`extendMain`を実行してる
  - 🔗 Linking packages を出力するのはこの`install`が該当
  - `node /fork/storybook/code/lib/cli/bin/index.js init --yes`でエラー出てる
- `sandbox/react-vite-default-ts`のパッケージ名が`before-storybook`のこと
- react-vite-default-ts として生成される sandbox 環境では`@vitejs/plugin-react`のバージョンが`"^4.0.0"`になっているが、code 側の方のバージョンが 3.1.0 になっているのが問題っぽい
  - `@storybook/react-vite@portal:/fork/storybook/code/frameworks/react-vite:`
- `code/frameworks/react-vite/package.json`の`@vitejs/plugin-react`のバージョンを`^4.0.0`に変えてみる
  - そしたら動いた

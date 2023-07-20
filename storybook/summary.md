# Reading Storybook

Storybook v7 のコードを読んで仕組みを理解する

https://github.com/storybookjs/storybook

## 3 行まとめ

- `yarn start`で sandbox 環境が構築され、すぐに動作確認ができる
- `storybook dev`では Express が起動し、ejs から HTML がレンダリングされる
- Manger app 部分は`@storybook/manager`パッケージが該当で、React で実装されている

## まずはじめに

Storybook のリポジトリを fork し、リポジトリの root 直下で`yarn start`を実行することで動作確認する環境が立ち上がる。

この`yarn start`には必要な依存関係のインストールも含まれているので、依存関係のインストールも必要ない。

デフォルトでは react-vite の sandbox 環境が構築される。sandbox 環境には以下で定義されているようなテンプレートを`--template`オプションで指定できる。
https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/cli/src/sandbox-templates.ts#L67

下記が`yarn start`で実行されるコマンド
`yarn task --task dev --template react-vite/default-ts --start-from=install`

sandbox 環境の Storybook 関連のパッケージは直接`code`で定義されたパッケージを見にいくようになっている。

```json:package.json
  "resolutions": {
    "@storybook/addon-a11y": "portal:{$HOME}storybook/code/addons/a11y",
    "@storybook/addon-actions": "portal:{$HOME}storybook/code/addons/actions",
    "@storybook/addon-docs": "portal:{$HOME}storybook/code/addons/docs",
  }
```

### sandbox 環境の構築ができない人へ

`yarn start`を実行すると下記のようなエラーが発生する

```
Cannot link @storybook/react-vite into before-storybook@workspace:. dependency @vitejs/plugin-react@npm:3.1.0 [c88a5] conflicts with parent dependency @vitejs/plugin-react@npm:4.0.0
```

`yarn start`を実行すると Sandbox 環境が構築されるが依存関係が最新なので、`@vitejs/plugin-react`のバージョンは`"^4.0.0"`になる。ただし、[frameworks(react-vite)側は`"^3.0.1"`](https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/frameworks/react-vite/package.json#L54)なのでエラーになる。

`code/frameworks/react-vite/package.json`の`@vitejs/plugin-react`のバージョンを`^4.0.0`に変えると、正常に sandbox 環境が構築される。

## ディレクトリ構成

- code が本体の実装
- sandbox が`yarn start`実行時に生成されるサンドボックス環境
- scripts に npm script 経由で実行されるような実装が定義されている

```
.
├── code  // Storybookの本体周り
│   ├── __mocks__
│   ├── addons
│   ├── e2e-tests
│   ├── frameworks
│   ├── lib
│   ├── node_modules
│   ├── presets
│   ├── renderers
│   └── ui
├── docs
│   ├── addons
│   ├── api
│   ├── assets
│   ├── builders
│   ├── configure
│   ├── contribute
│   ├── essentials
│   ├── get-started
│   ├── sharing
│   ├── snippets
│   ├── versions
│   ├── writing-docs
│   ├── writing-stories
│   └── writing-tests
├── node_modules
├── sandbox  rootで`yarn start`すると生成される環境
│   └── react-vite-default-ts
├── scripts  rootで`yarn start`した際に実行されるファイルが
│   ├── node_modules
│   ├── prepare
│   ├── sandbox
│   ├── tasks
│   └── utils
└── test-storybooks
    ├── ember-cli
    ├── external-docs
    ├── server-kitchen-sink
    └── standalone-preview
```

## `storybook dev`が実行されて、Manager app 部分がレンダリングされるまでの流れ

1. `storybook` パッケージの index.js が実行され、`@storybook/cli/bin/index`が require される
2. `@storybook/cli` パッケージの`bin/index.js`が実行され、`'../dist/generate.js'`が require される
3. require された`code/lib/cli/src/generate.ts`(ビルド前)で、[commander](https://github.com/tj/commander.js/)を使って `storybook dev`コマンドが定義される
4. `storybook dev`コマンドでは、`code/lib/cli/src/dev.ts`の`dev`関数が実行される
5. `dev`関数では、`@storybook/core-server`パッケージの`buildDevStandalone`が実行される
6. `buildDevStandalone`では`code/lib/core-server/src/dev-server.ts`で定義された`storybookDevServer`が実行される
7. `storybookDevServer`では Express を使ってサーバーが起動
8. `@storybook/builder-manager`パッケージが提供するビルダーを使ってベースとなる HTML をレンダリング
9. Manager app と呼ばれる部分は`@storybook/manager`パッケージで React でレンダリングされる

### 今回登場する code 配下のディレクトリ

```
.
├── frameworks
│   └── react-vite
├── lib
│   ├── builder-manager // 8. @storybook/builder-manager
│   ├── builder-vite
│   ├── cli // 2~5. @storybook/cli
│   ├── cli-storybook  // 1. storybook
│   └── core-server // 6~7. @storybook/core-server
└── ui
    └── manager // 9. @storybook/manager
```

### 1. `storybook`パッケージ

`@storybook/cli`の wrapper パッケージで`@storybook/cli`を require してるのみ。

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/cli-storybook/index.js#L3

### 2~5. `@storybook/cli`

`@storybook/cli/bin/index.js`では`generate.js`を require している

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/cli/bin/index.js#L9

---

`generate.js`のビルド前のコードは`code/lib/cli/src/generate.ts`であり、ここでは`storybook dev`を含むコマンドの処理が実装されている。

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/cli/src/generate.ts#L184

---

`storybook dev`コマンドでは`dev`関数が実行され、`@storybook/core-server`の`buildDevStandalone`が呼び出される

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/cli/src/dev.ts#L38

### 6~7. `@storybook/core-server`

`@storybook/core-server`の`buildDevStandalone`では dev サーバーの起動に`storybookDevServer`が実行されている

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-server/src/build-dev.ts#L127-L129

---

`storybookDevServer`では Express を使ってサーバーが起動

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-server/src/dev-server.ts#L24

---

Manager app 部分は`managerBuilder.start()`でレンダリングされている

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-server/src/dev-server.ts#L79-L85

`managerBuilder`の実態は`'@storybook/builder-manager'`を import したもの

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/core-server/src/utils/get-builders.ts#L4

### 8. `@storybook/builder-manager`

`managerBuilder.start()`では`starter`関数が実行される

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/builder-manager/src/index.ts#L288

---

starter 関数では html をレンダリングし、`/`のレスポンスとして渡している

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/builder-manager/src/index.ts#L166-L188

また、テンプレートエンジンとして ejs が使われる

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/builder-manager/src/utils/data.ts#L18

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/builder-manager/templates/template.ejs

---

このテンプレートではクライアントサイドの runtime として`'./sb-manager/runtime.js'`が import される

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/builder-manager/templates/template.ejs#LL60C7-L60C40

`./sb-manager/runtime.js`は`@storybook/manager`パッケージで生成された js ファイル

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/builder-manager/src/index.ts#L160

### 9. `@storybook/manager`

`./sb-manager/runtime.js`のビルド前の`code/ui/manager/src/runtime.ts`では`renderStorybook`を使ってクライアントサイドのレンダリングをしてる

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/ui/manager/src/runtime.ts#L61

---

`renderStorybook`では React をレンダリングしている

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/ui/manager/src/index.tsx#L76

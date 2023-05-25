# Storybook の UI がどのようにクライアントサイドでレンダリングされているか

- dev コマンドを実行すると、ejs をテンプレートエンジンとして使ったベースの HTML がサーバーサイド側でレンダリングされる
- `/`にリクエストを送るとレンダリングされた HTML がレスポンスとして返ってくる
- ベースの HTML には`./sb-manager/runtime.js`をクライアントサイド側で import してる
- `./sb-manager`は`@storybook/manager/dist`配下のファイルを import する
- Storybook のクライアントサイドで描画されている UI のベースの実装は`@storybook/manager`にある

## クライアントサイドで import される`runtime.js`

`/`にリクエストした際に HTML がレスポンスで返ってくるが、その HTML では`import './sb-manager/runtime.js';`されている

https://github.com/storybookjs/storybook/blob/1328d492f55a74921b4e125b90a9ddade1937912/code/lib/builder-manager/templates/template.ejs#L59-L65

`./sb-manager`のリクエストでは`@storybook/manager/dist`配下の静的ファイルをレスポンスとして返すようにルーティングされている

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/lib/builder-manager/src/index.ts#L160

`@storybook/manager`の`runtime.js`では id 属性に`root`が付与された要素に対して、React でレンダリングを行なっている

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/ui/manager/src/runtime.ts#L61
https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/ui/manager/src/index.tsx#L76

## `@storybook/manager`の src 配下のディレクトリ構成

```
.
├── __tests__
├── components
│   ├── hooks
│   ├── layout
│   ├── notifications
│   ├── panel
│   ├── preview
│   └── sidebar
├── containers
├── globals
└── settings
```

## `<Root>`

`ReactDOM.render`で`<Root>`コンポーネントをレンダリングしている

https://github.com/storybookjs/storybook/blob/529089564eb1b369603ee3c563c3ed6f585002f9/code/ui/manager/src/index.tsx#L76

この`<Root>`コンポーネントでは Theme や Manager などの Provider でラップされつつ、`<App>`コンポーネントが描画される

https://github.com/storybookjs/storybook/blob/1328d492f55a74921b4e125b90a9ddade1937912/code/ui/manager/src/index.tsx#L38-L67

## `<App>`

- ビューポートの幅(多分)から、デスクトップかモバイル用のコンポーネントを切り分けている
  - https://github.com/storybookjs/storybook/blob/1328d492f55a74921b4e125b90a9ddade1937912/code/ui/manager/src/app.tsx#L57-L72
- `<symbol>`を使って、使用する SVG アイコンを指定している
  - https://github.com/storybookjs/storybook/blob/1328d492f55a74921b4e125b90a9ddade1937912/code/ui/manager/src/app.tsx#L77

## `<Desktop>`

- ビューポートの幅が 600 以上の際に表示されるコンポーネント
- `<></>`を使わずに`<Fragment>`を使ってる
- 描画されている UI は`code/ui/manager/src/containers`に定義されているものを使っている
  - https://github.com/storybookjs/storybook/blob/1328d492f55a74921b4e125b90a9ddade1937912/code/ui/manager/src/components/layout/desktop.tsx#L47-L75
-

## `<S.Layout>`

- class コンポーネントで実装
  - https://github.com/storybookjs/storybook/blob/1328d492f55a74921b4e125b90a9ddade1937912/code/ui/manager/src/components/layout/container.tsx#L347
- なにか新しいコンポーネントを描画しているというよりは、children に対して必要な props を生成してる感じのコンポーネント
  - https://github.com/storybookjs/storybook/blob/1328d492f55a74921b4e125b90a9ddade1937912/code/ui/manager/src/components/layout/container.tsx#L591-L640

## `<Preview>`

- `<Desktop>`が props で受け取るコンポーネント
  - https://github.com/storybookjs/storybook/blob/1328d492f55a74921b4e125b90a9ddade1937912/code/ui/manager/src/components/layout/desktop.tsx#L59-L61
- `<S.Preview>`は Preview の周りに対するアニメーションなどを定義している
- `<App>`時に Preview は定義されており、実態は`code/ui/manager/src/containers/preview.tsx`
- Consumer の中で Preview コンポーネントを描画している
  - https://github.com/storybookjs/storybook/blob/1328d492f55a74921b4e125b90a9ddade1937912/code/ui/manager/src/containers/preview.tsx#L49
  - Consumer からは api 経由で manager の状態が取得できそう
  - `@storybook/manager-api`で提供されている
- 表示されているコンポーネントの実態
- https://github.com/storybookjs/storybook/blob/1328d492f55a74921b4e125b90a9ddade1937912/code/ui/manager/src/components/preview/preview.tsx#L158

### `code/ui/manager/src/components/preview/preview.tsx`

- Preview 内部で描画されている本体は`tabs`の中に格納されている`render`
  - https://github.com/storybookjs/storybook/blob/1328d492f55a74921b4e125b90a9ddade1937912/code/ui/manager/src/components/preview/preview.tsx#L219
- `tabs`は`useTabs`で生成されている
  - https://github.com/storybookjs/storybook/blob/1328d492f55a74921b4e125b90a9ddade1937912/code/ui/manager/src/components/preview/preview.tsx#L172
- `tabs`の`render`の定義は`useTabs`で実行されている`createCanvas`内の render 部分になりそう
  - https://github.com/storybookjs/storybook/blob/1328d492f55a74921b4e125b90a9ddade1937912/code/ui/manager/src/components/preview/preview.tsx#L50-L64
- 実態部分のコンポーネントっぽい`<FramesRenderer>`の実装を見る
  - https://github.com/storybookjs/storybook/blob/1328d492f55a74921b4e125b90a9ddade1937912/code/ui/manager/src/components/preview/preview.tsx#L111-L120

### `<FramesRenderer>`

- props で渡される初期情報から `src` を生成
  - `iframe.html?globals=backgrounds.grid:!false;theme:dark&viewMode=story&id=example-button--primary`
- その`src`を`IFrame`コンポーネントに渡す
  - https://github.com/storybookjs/storybook/blob/09db9f7dd11356714952449cbfd8280407b014c3/code/ui/manager/src/components/preview/FramesRenderer.tsx#L105-L117

### `<IFrame>`

- `<iframe>`要素の src 属性に`<FramesRenderer>`から受け取った src を渡してる
  - https://github.com/storybookjs/storybook/blob/09db9f7dd11356714952449cbfd8280407b014c3/code/ui/manager/src/components/preview/iframe.tsx#L32-L42

## 結局 preview では何を描画してるのか

- 下記のような値を`<iframe>`の src 属性に突っ込んだもの
  - `iframe.html?globals=backgrounds.grid:!false;theme:dark&viewMode=story&id=example-button--primary`
- dev コマンド実行時に起動するサーバーでは下記のようなリクエストを投げると対象のコンポーネントだけが描画された状態になる
  - `http://localhost:6006/iframe.html?globals=backgrounds.grid:!false;theme:dark&viewMode=story&id=example-button--primary`
- サーバー側に`/iframe.html`のリクエストが送られるとどうなるのか
- dev コマンド時のサーバー側の処理を思い出すと、Manager 側の UI は`managerBuilder.start()`で実行されていた
- 同じファイルでは`previewBuilder.start()`も実行されている
  - https://github.com/storybookjs/storybook/blob/09db9f7dd11356714952449cbfd8280407b014c3/code/lib/core-server/src/dev-server.ts#L90C34-L97

## 雑メモ

- React v16 使ってる部分もありそう
  - https://github.com/storybookjs/storybook/blob/1328d492f55a74921b4e125b90a9ddade1937912/code/ui/manager/package.json#L78-L79
- Layout コンポーネントは class コンポーネント

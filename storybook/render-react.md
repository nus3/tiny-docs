# Storybook dev モード実行時の React のレンダリング方法

## これまでのまとめ

- `/virtual:/@storybook/builder-vite/vite-app.js`で`PreviewWeb().initialize()`が実行される
- `PreviewWeb`クラスの継承元である[`PreviewWithSelection`にある`renderSelection`でレンダリングを実行してそう](https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/PreviewWithSelection.tsx#LL265C9-L265C24)
  - ただし、`PreviewWithSelection.renderSelection`ががどこで実行されているのかはまだわかってない
- `renderSelection`の中では[対象が mdx とかではなく通常のコンポーネントの場合、StoryRender クラスが render として定義される](https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/PreviewWithSelection.tsx#L303-L314)
- [`renderSelection`では最終的に`StoryRender.renderToElement()`を実行することでコンポーネントをレンダリングしてそう](https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/PreviewWithSelection.tsx#L421-L427)
- [`StoryRender.renderToElement`は`StoryRender.render`を実行している](https://github.com/storybookjs/storybook/blob/5b303362983e342b7c3607fe63269203381ca977/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#LL134C7-L134C7)
- [`StoryRender.render`では context を詰め込み、`StoryRender`のコンストラクタで受け取る`this.renderToScreen`を実行](https://github.com/storybookjs/storybook/blob/5b303362983e342b7c3607fe63269203381ca977/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L169-L216)
- [`StoryRender`のコンストラクタで受け取る`renderToScreen`は、`PreviewWithSelection`で定義されており、`renderToCanvas`が実行](https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/PreviewWithSelection.tsx#L306-L310)
- [この`renderToCanvas`は`PreviewWithSelection`内部では定義されておらず、継承元の`Preview`内で定義されている](https://github.com/storybookjs/storybook/blob/c739d024f666aa6f9dd846804a9cb5336358b516/code/lib/preview-api/src/modules/preview-web/Preview.tsx#L119)
  - `Preview.renderToCanvas`は`projectAnnotations.renderToCanvas`の値になる
- [`projectAnnotations`は`getProjectAnnotations`の返り値で、`getProjectAnnotations`の定義は`/virtual:/@storybook/builder-vite/vite-app.js`で呼び出される仮想モジュールの処理に定義されている](https://github.com/storybookjs/storybook/blob/c739d024f666aa6f9dd846804a9cb5336358b516/code/builders/builder-vite/src/codegen-modern-iframe-script.ts#L24-L29)
- preset が react-vite の場合、この中で`import("@storybook/react/preview")`が dynamic import される
  - この dynamic import する対象のモジュールを判別する仕組みはまだわかってない
- `@storybook/react/preview`の中で export されている`renderToCanvas`が`Preview.renderToCanvas`の実態
- [`@storybook/react/preview`は`"./dist/config.mjs"`が対象](https://github.com/storybookjs/storybook/blob/c739d024f666aa6f9dd846804a9cb5336358b516/code/renderers/react/package.json#L32)
- [`renderToCanvas`の export は src/config.ts で定義されている](https://github.com/storybookjs/storybook/blob/c739d024f666aa6f9dd846804a9cb5336358b516/code/renderers/react/src/config.ts#LL7C14-L7C14)
- [`renderToCanvas`では、`unboundStoryFn`を`<Story>`コンポーネントとして扱い、描画している](https://github.com/storybookjs/storybook/blob/c739d024f666aa6f9dd846804a9cb5336358b516/code/renderers/react/src/render.tsx#L67-L73)
- [`unboundStoryFn`は`StoryRender.story`に格納されている](https://github.com/storybookjs/storybook/blob/c739d024f666aa6f9dd846804a9cb5336358b516/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L154-L155)
- [`StoryRender.story`は`StoryStore.loadStory()`の返り値](https://github.com/storybookjs/storybook/blob/c739d024f666aa6f9dd846804a9cb5336358b516/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L100)
- [`StoryStore.loadStory()`の返り値は、`prepareStoryWithCache`(`prepareStory`)の返り値](https://github.com/storybookjs/storybook/blob/c739d024f666aa6f9dd846804a9cb5336358b516/code/lib/preview-api/src/modules/store/StoryStore.ts#L239-L246)
- [`prepareStory`で`unboundStoryFn`を定義しており、`decoratedStoryFn`が実行される](https://github.com/storybookjs/storybook/blob/c739d024f666aa6f9dd846804a9cb5336358b516/code/lib/preview-api/src/modules/store/csf/prepareStory.ts#L88)
- [`decoratedStoryFn`のデフォルトは`defaultDecorateStory`](https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/csf/prepareStory.ts#L70)
- [`defaultDecorateStory`では、渡された decorators から story と decorator を受け取り、`decorateStory`を実行する ](https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/decorators.ts#L84-L91)
- [`decorateStory`では渡された decorator を実行している](https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/decorators.ts#L20)
- [`defaultDecorateStory`渡す`decorators`には projectAnnotations の decorators が含まれる](https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/csf/prepareStory.ts#L72-L76)
- [`@storybook/react/preview`では`decorators`の定義をしている](https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/renderers/react/src/config.ts#L5)
- [`@storybook/react/preview`の`decorators`で使われているのは`jsxDecorator`](https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/renderers/react/src/docs/config.ts#L16)

## これまでのまとめのまとめ

- `@storybook/react/preview`に React のレンダリング時の処理が定義されている
- React などの render 側のインターフェースには`renderToCanvas`や`decorators`、`applyDecorators`の定義が必要
- `renderToCanvas`するときに、`jsxDecorator`が実行されている

## decorator として使われる`jsxDecorator`が何をしているのかまとめる

- `code/renderers/react/src/docs/jsxDecorator.tsx`に実装が定義されている
- [`jsxDecorator`では、引数で渡された`storyFn`の返り値を返している]
  - https://github.com/storybookjs/storybook/blob/3899b2b480397b9844733d9ddae4efb3e7d409f9/code/renderers/react/src/docs/jsxDecorator.tsx#L187-L192
  - `storyFn`の返り値は ReactElement
    - https://github.com/storybookjs/storybook/blob/3899b2b480397b9844733d9ddae4efb3e7d409f9/code/renderers/react/src/types.ts#L21
- `jsxDecorator`の引数として渡される`storyFn`はデフォルトでは`projectAnnotations`の`render`になりそう
  - https://github.com/storybookjs/storybook/blob/3899b2b480397b9844733d9ddae4efb3e7d409f9/code/lib/preview-api/src/modules/store/csf/prepareStory.ts#L62-L67
  - https://github.com/storybookjs/storybook/blob/3899b2b480397b9844733d9ddae4efb3e7d409f9/code/lib/preview-api/src/modules/store/csf/prepareStory.ts#L80-L84
- `projectAnnotations`にある`render`は`@storybook/react/preview`で提供される`render`
  - この`render`では、第二引数の`context`オブジェクトの`component`プロパティを使って`ReactElement`を返している
  - https://github.com/storybookjs/storybook/blob/3899b2b480397b9844733d9ddae4efb3e7d409f9/code/renderers/react/src/render.tsx#L12-L21
- この`context`は`unboundStoryFn`に渡される引数っぽい
  - https://github.com/storybookjs/storybook/blob/3899b2b480397b9844733d9ddae4efb3e7d409f9/code/lib/preview-api/src/modules/store/csf/prepareStory.ts#LL88C1-L88C1
- `unboundStory`は`renderToCanvas`で実行されており、`context`は`renderToCanvas`の引数になっている
  - https://github.com/storybookjs/storybook/blob/c739d024f666aa6f9dd846804a9cb5336358b516/code/renderers/react/src/render.tsx#L57-L73
- `renderToCanvas`に引数が渡されるのは`StoryRender.renderToScreen`
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/PreviewWithSelection.tsx#L306-L310
- `StoryRender.renderToScreen`は`StoryRender.render`の中で実行されており、`context`は`renderContext`が実態
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L214
- `renderContext`の中にある`storyContext`は`renderStoryContext`
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L208
- `renderStoryContext`の中の`component`プロパティをみると ReactElement が格納されていそう
- `renderStoryContext`の値は、`loadedContext`と`this.storyContext()`が展開されている
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L182-L185
- `loadedContext`は`StoryRender.story`に格納されている`applyLoaders`の返り値
- `StoryRender.story`は`StoryRender.store.loadStory`の返り値
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L100
- `StoryRender.store`は`StoryStore`のインスタンスなので`StoryRender.store.loadStory` === `StoryStore.loadStory`
- また、`StoryRender.storyContext`は`StoryStore.getStoryContext`の返り値である
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L137-L141
- `StoryRender.store.loadStory`から`applyLoaders`の実装を確認する
  - 中身を見ていくと projectAnnotations などから `loaders` を取得している
    - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/csf/prepareStory.ts#L51-L60
- `loaders`は sandbox では、storybook を利用する側で定義されている

```ts: sandbox/react-vite-default-ts/template-stories/lib/preview-api/preview.ts
export const loaders = [async () => ({ projectValue: 2 })];
```

- `getProjectAnnotations`では`import("/template-stories/lib/preview-api/preview.ts"),`が該当しそう
- `context`の値を生成してそうな`StoryStore.getStoryContext`の実装を見る
- `StoryStore.getStoryContext`では`prepareContext`の返り値になっているが、この引数の中にある`story`にはすでに`component`が定義されている
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/StoryStore.ts#L284-L296
- `StoryStore.getStoryContext`が使われているのは`StoryRender.storyContext`の中で、この中では`StoryRender.story`が渡されている
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L140
- `StoryRender.story`は`StoryStore.loadStory`の返り値
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L100
- `StoryStore.loadCSFFileByStoryId`を実行すると`context.component`の値が生成されていそう
- `storyId`から`importPath`を取得、そのパスを使用して dynamic import してそう。そのモジュールを`StoryStore.processCSFFileWithCache`に渡している。
  - https://github.com/storybookjs/storybook/blob/3899b2b480397b9844733d9ddae4efb3e7d409f9/code/lib/preview-api/src/modules/store/StoryStore.ts#L145-L154
  - `processCSFFileWithCache`は`processCSFFile`
  - `storyIndex`には対象の story の importPath などが定義されている
    - https://nus3.slack.com/archives/DB31XDW83/p1687330301147389?thread_ts=1687322475.080149&cid=DB31XDW83
  - `StoryStore.storyIndex.storyIdToEntry`に該当の id を渡すことで import に必要な path や title を取得できる
  - `importPath`には CSF で定義されたファイルのパスが入っている
- `processCSFFile`では CSF で定義されたモジュールから export されたものを取得する
  - https://github.com/storybookjs/storybook/blob/3899b2b480397b9844733d9ddae4efb3e7d409f9/code/lib/preview-api/src/modules/store/csf/processCSFFile.ts#L52-L80
- CSF ファイルで定義された meta 情報の中に`component`があれば、これが`context.component`の値になる

```ts
const meta = {
  title: "Example/Button",
  component: Button,
  tags: ["autodocs"],
  argTypes: {
    backgroundColor: { control: "color" },
  },
} satisfies Meta<typeof Button>;

export default meta;
```

- `@storybook/react/preview`の`render`がレンダリングには使われており、この`render`関数に渡される`context`オブジェクトの中の`component`が描画には使われている
- この`context.component`は CSF で定義された story ファイルの `meta` にある `component` が該当する
- storyId からレンダリングする story を決めてそう
  - https://github.com/storybookjs/storybook/blob/3899b2b480397b9844733d9ddae4efb3e7d409f9/code/lib/preview-api/src/modules/store/StoryStore.ts#L233-L243
  - `storyAnnotations`がおそらく各 story で設定する値で、`componentAnnotations`が CSF ファイルで定義された`meta`の値
  - `storyAnnotations`には対象の story をレンダリングする上で props に渡したい値が`args`に格納されている
  - CSF ファイルで定義した Story オブジェクトの`args`プロパティが`context`にも格納される
- `@storybook/react/preview`の`render`では第一引数で`args`を受け取っている
  - https://github.com/storybookjs/storybook/blob/3899b2b480397b9844733d9ddae4efb3e7d409f9/code/renderers/react/src/render.tsx#L12-L21
- `@storybook/react/preview`の`render`関数は`prepareStory`の中で実行されており、`context.args`が第一引数に渡されている
  - https://github.com/storybookjs/storybook/blob/3899b2b480397b9844733d9ddae4efb3e7d409f9/code/lib/preview-api/src/modules/store/csf/prepareStory.ts#L65
- `context.args`の生成は`StoryStore.getStoryContext`のタイミングでやってそう
  - https://github.com/storybookjs/storybook/blob/3899b2b480397b9844733d9ddae4efb3e7d409f9/code/lib/preview-api/src/modules/store/StoryStore.ts#L292
- `StoryStore.getStoryContext`は`StoryRender.storyContext`の中で呼ばれ、`StoryRender.storyContext`は`StoryRender.render`が実行されるときに呼ばれる
  - https://github.com/storybookjs/storybook/blob/3899b2b480397b9844733d9ddae4efb3e7d409f9/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L181-L189

## `PreviewWithSelection` の`renderSelection`がどこで実行されているか

- `PreviewWithSelection.renderSelection`がレンダリングの際に実行されていそう
- `/virtual:/@storybook/builder-vite/vite-app.js`を読み込んだ際に、どういった流れで`PreviewWithSelection.renderSelection`が実行されているのか
- `PreviewWeb().initialize({ importFn, getProjectAnnotations })`を実行する中で`Preview.initializeWithProjectAnnotations`が実行される
  - https://github.com/storybookjs/storybook/blob/3899b2b480397b9844733d9ddae4efb3e7d409f9/code/lib/preview-api/src/modules/preview-web/Preview.tsx#L97
- `Preview.initializeWithProjectAnnotations`の中では`Preview.initializeWithStoryIndex`が呼ばれる
  - https://github.com/storybookjs/storybook/blob/3899b2b480397b9844733d9ddae4efb3e7d409f9/code/lib/preview-api/src/modules/preview-web/Preview.tsx#L156
  - `initializeWithStoryIndex`は継承先のクラスである`PreviewWithSelection`でも定義されており、`PreviewWithSelection.selectSpecifiedStory`が実行される
    - https://github.com/storybookjs/storybook/blob/3899b2b480397b9844733d9ddae4efb3e7d409f9/code/lib/preview-api/src/modules/preview-web/PreviewWithSelection.tsx#L118-L126
- `PreviewWithSelection.selectSpecifiedStory`の中で、`PreviewWithSelection.renderSelection`が実行される
  - https://github.com/storybookjs/storybook/blob/3899b2b480397b9844733d9ddae4efb3e7d409f9/code/lib/preview-api/src/modules/preview-web/PreviewWithSelection.tsx#L172

<!-- TODO: 親、子、孫のクラスで同一のメソッド名がある場合の実行順を確認する -->

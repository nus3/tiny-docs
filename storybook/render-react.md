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

`code/renderers/react/src/docs/jsxDecorator.tsx`

TODO

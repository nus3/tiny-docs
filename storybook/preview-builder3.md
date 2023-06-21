# Storybook は dev モード時に Preview(コンポーネントを描画してる箇所)部分をどのようにレンダリングしているのか

## 3 行まとめ

- [builder(今回の場合、builder-vite)で必要なモジュールを dynamic import する](https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/builders/builder-vite/src/codegen-modern-iframe-script.ts#L23-L30)
- [`@storybook/preview-api`で、dynamic import された各モジュールから export された関数やオブジェクトを紐づけている](https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/csf/composeConfigs.ts#L45-L66)
- [`@storybook/react`などの各レンダラーでは、紐づけられる予定の関数やオブジェクトを export している](https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/renderers/react/src/config.ts#L7)

## これまでのまとめ

dev モードで Storybook を起動すると preview 部分では、以下二つの JS ファイルを読み込んでいる。

```html
<script type="module" src="./sb-preview/runtime.js"></script>
<script
  type="module"
  src="/virtual:/@storybook/builder-vite/vite-app.js"
></script>
```

`/virtual:/@storybook/builder-vite/vite-app.js`は、Vite の[Virtual Modules Convention](https://vitejs.dev/guide/api-plugin.html#virtual-modules-convention)を使って、dev モード中は仮想モジュールを生成している。(下記リンクの`virtualFileId`が`/virtual:/@storybook/builder-vite/vite-app.js`に該当する)

https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/builders/builder-vite/src/plugins/code-generator-plugin.ts#L109-L114

仮想モジュール内のコードでは、`PreviewWeb().initialize({ importFn, getProjectAnnotations })`を実行することで、Preview の初期化をしていそう。

## PreviewWeb クラス

- `/virtual:/@storybook/builder-vite/vite-app.js`で実行される`PreviewWeb().initialize()`の実装は Preview クラスの中にある
  - code/lib/preview-api/src/modules/preview-web/Preview.tsx
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/Preview.tsx#L75-L85
- ただ、この`initialize`のなかにレンダリングに関する詳細の実装が特定できなかった
- Preview クラスには`renderStoryToElement`という関数があり、これがレンダリングに関する実装をしているように見える
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/Preview.tsx#L319-L345
- `renderStoryToElement`が実行されている箇所を探す
- Preview クラスを継承した PreviewWithSelection の`renderSelection`で使われている
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/PreviewWithSelection.tsx#L431
- 何も設定せずに`yarn start`した場合、`renderSelection`は呼ばれているが、`renderStoryToElement`自体は実行されない
  - `isStoryRender(render)`が true の場合の処理が実行されている
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/PreviewWithSelection.tsx#L421-L434
- `renderStoryToElement`自体は呼ばれていなかったが、`renderSelection`が実行されていることが確認できた
- `this.currentRender.renderToElement()`が描画に関わっている？
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/PreviewWithSelection.tsx#L424C13-L426
- `this.currentRender`は以下で定義されている
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/PreviewWithSelection.tsx#L337
- この`render`は sandbox の button コンポーネントを描画する場合は`StoryRender`が該当する
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/PreviewWithSelection.tsx#L303-L314
  - この時、storyId と entry の中身は下記のようになる

```json
{
  "storyId": "example-button--primary",
  "entry": {
    "id": "example-button--primary",
    "title": "Example/Button",
    "name": "Primary",
    "importPath": "./src/stories/Button.stories.ts",
    "tags": ["autodocs", "story"],
    "type": "story"
  }
}
```

- `this.currentRender.renderToElement()`は`StoryRender.renderToElement()`を実行している
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L126-L135
  - `StoryRender.renderToElement()`は`StoryRender.render()`を実行している
  - この中の canvasElement には下記の値が格納されている
  - もうすでに必要な要素(この場合、button)が描画されてる？？

```html
<div id="storybook-root">
  <button
    type="button"
    class="storybook-button storybook-button--medium storybook-button--primary"
  >
    Button
  </button>
</div>
```

- `StoryRender`には phase が定義されている
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L27-L35
  - 初期値は`preparing`
    - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L83
- `StoryRender.render()`の実装を見る
- phase に定義されている`rendering`では、`this.renderToScreen()`が実行されている
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L213-L216
- `this.renderToScreen()`は`StoryRender`のコンストラクタで受け取っている
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L68
- PreviewWithSelection の中で、`this.renderToScreen()`の定義を行なっている
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/PreviewWithSelection.tsx#L306-L310
  - `renderToCanvas`が rendering 時に実行されていそう
- `renderToCanvas`自体は`Preview`クラスのプロパティ
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/Preview.tsx#L54
- 実体は`projectAnnotations.renderToCanvas`か`projectAnnotations.renderToDOM`
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/Preview.tsx#L117
- Sandbox の初期設定の場合、`renderToCanvas`が使われていそう
- `projectAnnotations`は`getProjectAnnotations`の返り値で、`getProjectAnnotations`は下記の部分で定義されている
  - code/builders/builder-vite/src/codegen-modern-iframe-script.ts
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/builders/builder-vite/src/codegen-modern-iframe-script.ts#L23-L30
  - `composeConfigs`の返り値
- `composeConfigs`の定義場所
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/csf/composeConfigs.ts#L39-L66
  - `composeConfigs`には各モジュールから利用する関数やオブジェクトが定義されてそう
- `renderToCanvas`も定義されている。`renderToCanvas: getSingletonField(moduleExportList, 'renderToCanvas')`
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/csf/composeConfigs.ts#L61
  - `composeConfigs`に渡すモジュールの中に`renderToCanvas`を実装しているモジュールがある
  - 以下は実際に sandbox で読み込まれているモジュールたち

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

- このモジュールの中で、`renderToCanvas`は`@storybook/react/preview`にありそう

## `@storybook/react`

- `code/renderers/react`に定義されている
- `@storybook/react/preview`は`./dist/config.mjs`、
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/renderers/react/package.json#L29-L33C7
- ビルド前のコードは`code/renderers/react/src/config.ts`
  - ここで`renderToCanvas`も export されている
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/renderers/react/src/config.ts#L7
- `renderToCanvas`では`ErrorBoundary`の設定もしつつ、引数で渡す`unboundStoryFn`が`<Story>`コンポーネントになり、`canvasElement`に、この`<Story>`コンポーネントを描画している
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/renderers/react/src/render.tsx#L57-L90
- 実際のレンダリングは`@storybook/react-dom-shim`が提供する、`renderElement`が実行されている

## `@storybook/react-dom-shim`

- `code/lib/react-dom-shim`に定義されている
- `renderElement`では `createRoot` からの`render`が実行される
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/react-dom-shim/src/react-18.tsx#L26-L33

## `<Story>`

- `<Story>`コンポーネントは`renderToCanvas`に渡す引数である`unboundStoryFn`が該当
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/renderers/react/src/render.tsx#LL67C31-L67C31
- `renderToCanvas`は`StoryRender`のインスタンスを生成するときに必要な`renderToScreen`のコールバックの中で実行される
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/PreviewWithSelection.tsx#L309
- `StoryRender`の`renderToScreen`に渡される引数の`renderContext`の中に`unboundStoryFn`が入っている
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#LL214C19-L214C19
- `renderContext`オブジェクトの定義場所
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L190-L211
  - ここで定義されている`unboundStoryFn`の実態は`this.story`から取得している
    - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L154-L155
- `StoryRender`の`this.story`は`this.store.loadStory({ storyId: this.id });`の返り値
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/render/StoryRender.ts#L100
- `this.store.loadStory`は`StoryStore`のメソッド
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/StoryStore.ts#L216-L220
- `loadStory`では、`prepareStoryWithCache`の返り値を`story`として返している
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/StoryStore.ts#L224-L247
  - `loadStory`の中で実行される`loadCSFFileByStoryId`の返り値は以下
    - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/StoryStore.ts#L218

```json
{
  "csfFile": {
    "meta": {
      "id": "example-button",
      "title": "Example/Button",
      "tags": ["autodocs"],
      "argTypes": {
        "backgroundColor": {
          "name": "backgroundColor",
          "control": {
            "type": "color"
          }
        }
      },
      "parameters": {
        "fileName": "./src/stories/Button.stories.ts"
      }
    },
    "stories": {
      "example-button--large": {
        "moduleExport": {
          "args": {
            "size": "large",
            "label": "Button"
          },
          "parameters": {
            "docs": {
              "source": {
                "originalSource": "{\n  args: {\n    size: 'large',\n    label: 'Button'\n  }\n}"
              }
            }
          }
        },
        "id": "example-button--large",
        "name": "Large",
        "tags": [],
        "decorators": [],
        "parameters": {
          "docs": {
            "source": {
              "originalSource": "{\n  args: {\n    size: 'large',\n    label: 'Button'\n  }\n}"
            }
          }
        },
        "args": {
          "size": "large",
          "label": "Button"
        },
        "argTypes": {},
        "loaders": []
      }
    },
    "moduleExports": {
      "Large": {
        "args": {
          "size": "large",
          "label": "Button"
        },
        "parameters": {
          "docs": {
            "source": {
              "originalSource": "{\n  args: {\n    size: 'large',\n    label: 'Button'\n  }\n}"
            }
          }
        }
      },
      "__namedExportsOrder": ["Primary", "Secondary", "Large", "Small"],
      "default": {
        "title": "Example/Button",
        "tags": ["autodocs"],
        "argTypes": {
          "backgroundColor": {
            "control": "color"
          }
        }
      }
    }
  }
}
```

- `storyFromCSFFile`の中で取得する storyAnnotations と componentAnnotations の中身は以下
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/StoryStore.ts#L224-L247

```json
// componentAnnotations
{
  "id": "example-button",
  "title": "Example/Button",
  "tags": ["autodocs"],
  "argTypes": {
    "backgroundColor": {
      "name": "backgroundColor",
      "control": {
        "type": "color"
      }
    }
  },
  "parameters": {
    "fileName": "./src/stories/Button.stories.ts"
  }
}
```

```json
// storyAnnotations
{
  "moduleExport": {
    "args": {
      "primary": true,
      "label": "Button"
    },
    "parameters": {
      "docs": {
        "source": {
          "originalSource": "{\n  args: {\n    primary: true,\n    label: 'Button'\n  }\n}"
        }
      }
    }
  },
  "id": "example-button--primary",
  "name": "Primary",
  "tags": [],
  "decorators": [],
  "parameters": {
    "docs": {
      "source": {
        "originalSource": "{\n  args: {\n    primary: true,\n    label: 'Button'\n  }\n}"
      }
    }
  },
  "args": {
    "primary": true,
    "label": "Button"
  },
  "argTypes": {},
  "loaders": []
}
```

- `prepareStoryWithCache`は`code/lib/preview-api/src/modules/store/csf/prepareStory.ts`から export されている`prepareStory`を実行している
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/StoryStore.ts#L80
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/StoryStore.ts#L33-L39
- `unboundStoryFn`は`decoratedStoryFn`を実行している
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/csf/prepareStory.ts#L88
  - `decoratedStoryFn`は`applyHooks`の返り値を関数として実行している
    - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/addons/hooks.ts#L183-L214
- `applyHooks`では`decorated`の結果が返り値になる関数が生成されそう
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/addons/hooks.ts#L198
  - `decorated`は引数で渡された`applyDecorators`の返り値
- `applyDecorators`は`projectAnnotions`の中から取得するか`defaultDecorateStory`が定義される
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/csf/prepareStory.ts#L70
- `projectAnnotions`は`Preview`の`initializeWithProjectAnnotations`の中で set されており
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/Preview.tsx#L138
  - 初期化処理の中でも呼ばれている
    - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/preview-web/Preview.tsx#L93-L95
- `projectAnnotations`は`getProjectAnnotations`の返り値
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/csf/composeConfigs.ts#L63
- `applyDecorators`は`renderToCanvas`と同様に`@storybook/react/preview`に定義されている
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/renderers/react/src/config.ts#L9
- 結局は`'@storybook/preview-api'`の`defaultDecorateStory`を実行してる
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/renderers/react/src/applyDecorators.ts#L7-L19
- 引数で渡された`decorator`を実行してそう
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/decorators.ts#L20
- 引数`decorator`は`applyHooks`の中で定義されている
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/addons/hooks.ts#L188-L190
  - `applyHooks`の第二引数が`decorators`(配列)になる
- `decorators`は`prepareStory`の際に定義される
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/csf/prepareStory.ts#L72-L77
- sandbox では storyAnnotations にも componentAnnotations にも decorators は定義されておらず、`projectAnnotations`の decorators が使われてそう
- `projectAnnotations`の`decorators`は`@storybook/react/preview`で定義されている
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/renderers/react/src/config.ts#L5
- decorators は`jsxDecorator`
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/renderers/react/src/docs/config.ts#L16
- `jsxDecorator`では JSX を render してそう
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/renderers/react/src/docs/jsxDecorator.tsx#L167-L170
- `renderJsx`をした後は、下記のような string が生成される
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/renderers/react/src/docs/jsxDecorator.tsx#L206-L209

```jsx
<Button label="Button" onClick={() => {}} primary />
```

- `jsxDecorator`に渡された`storyFn`を実行すると`ReactElement`が生成される
- この`ReactElement`が`<Story>`コンポーネントとしてレンダリングされる

## `storyFn`

- `jsxDecorator`に渡される`storyFn`を実行すると`ReactElement`が生成された
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/renderers/react/src/docs/jsxDecorator.tsx#L187-L209
- `storyFn`の実態はどれか
- `prepareStory`の中の`applyHooks`を実行する際に、`undecoratedStoryFn`が渡されててこれっぽい
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/csf/prepareStory.ts#L87
- `undecoratedStoryFn`はのちに定義される`render`
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/csf/prepareStory.ts#L62-L66
- おそらく、`projectAnnotations.render`が利用されている
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/lib/preview-api/src/modules/store/csf/prepareStory.ts#L84
- `projectAnnotations.render`は`@storybook/react/preview`の中で定義されている
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/renderers/react/src/config.ts#L7
- `render`では、context の中にある`component`が`ReactElement`として返される
  - https://github.com/storybookjs/storybook/blob/9630bdd1622ba0533948445c22b96164c865d965/code/renderers/react/src/render.tsx#L12-L21

## TODO

- `PreviewWithSelection` の`renderSelection`がどこで実行されているか

## 3 行まとめ

- Storybook の preview 部分では`<iframe>`の src 属性に`/iframe.html`を指定してる
- `/iframe.html`のリクエストの処理は`previewBuilder`が行なってそう
- `previewBuilder`の実態は`@storybook/builder-{builderName}`パッケージ

`iframe.html?globals=backgrounds.grid:!false;theme:dark&viewMode=story&id=example-button--primary`

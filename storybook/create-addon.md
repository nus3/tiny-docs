# addon を作る

- addon 用の kit があるのでそれをベースにリポジトリを作成する
  - https://github.com/storybookjs/addon-kit
- manager.ts で`@storybook/manager-api`で manager 側に addon を追加できる
- addon には下記が追加できる
  - ヘッダーのアイコン部分に追加できる Tool
  - 下部の panel
  - ヘッダ左上の Tab
- storybook 関連の API
  - https://storybook.js.org/docs/react/addons/addons-api#storybook-api
- Storybook 関連の API を使うには`@storybook/manager-api`の`useStorybookApi`を使えば良さそう
- そのほか便利な hooks もありまっせ
  - https://storybook.js.org/docs/react/addons/addons-api#storybook-hooks

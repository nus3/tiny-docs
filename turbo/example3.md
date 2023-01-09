## turbopack-create-test-app

- とりあえず実行してみる
  - `cargo run --package turbopack-create-test-app`
- public と src と index.html と vite-server.mjs が同じディレクトリ内に生成された
- `cargo run --package turbopack-create-test-app -- -h`で option の指定ができる
- CLI 用の crate として clap を使ってる
- 指定したディレクトリに生成する(今回は`test`)場合は `cargo run --package turbopack-create-test-app -- ../test --package-json`を実行する

## crates/turbopack/examples/turbopack.rs で使われてる他の crate のやつを探す

`crates/turbopack-core/src/reference/mod.rs`に定義されている`all_referenced_assets`が使われている

## `turbopack_ecmascript::`で検索かけてトランスパイルやバンドルされてる箇所を探す

- `cargo test -p turbopack-ecmascript`で、このパッケージ関連のテストは実行できた
- `crates/turbopack-ecmascript/src/lib.rs`の中を見る

## ベンチマークから実際に Turbopack がどのように使われてるかを推測する

CONTRIBUTING.md を読むとベンチマークに関しては[crates/next-dev/benches/README.md](https://github.com/vercel/turbo/tree/main/crates/next-dev/benches)を読めとのこと

### crates/next-dev/benches/README.md を読む

- `cargo bench -p next-dev`でベンチマークを実行できるよ
- ベンチマークはいろんな環境でできるよ
- ベンチマークはいろんなシナリオで計測しとるよ

### ベンチマークの処理を見て、Turbopack がどのように使われてるかみる

- エントリーポイント `crates/next-dev/src/main.rs`
- ベンチの実行のメイン処理は`next_dev::start_server()`でやってそう
- `NextDevServerBuilder::new()`で生成されたインスタンスの`build()`メソッドが実行される
- build の中で`crates/turbopack-dev-server/src/lib.rs`の`DevServer`の`listen`が呼ばれてる
- `DevServerBuilder`の`serve`が実行されてる

## `next dev --turbo`が何をしてるのか

- next パッケージで実行されているスクリプトは`"dev": "taskr",`
  - https://github.com/vercel/next.js/blob/canary/packages/next/package.json#L63
- `taskr`は並行処理を念頭に置いたタスクランナー
- Next.js の next パッケージの処理を見に行った方が良さそう
- `taskr`の local plugin の設定を package.json に記載できる
  - https://github.com/lukeed/taskr/tree/master/packages/taskr#local-plugins
- `taskfile.js`で export された関数は`taskr {function_name}`で実行できる
  - また内部でも`task.parallel`とかでも並列で実行できる
  - `taskr`だけだったら、`export default`された関数が実行されそう
- `task.$.log('hoge')`で出力できる
- `--turbo`で検索してみると`packages/next/src/cli/next-dev.ts`にそれっぽいのが定義されている
- `next`コマンド自体は`./dist/bin/next`
  - これは`taskr release`で生成された bin ファイル
  - `taskr release`では`dist`を clear して、taskfile に定義されている`build`を実行
  - `build`では`precompile`→ `compile`→ `compile_config_schema`が順次実行されている
  - `precompile`は`browser_polyfills`, `path_to_regexp`, `copy_ncced`, `copy_styled_jsx_assets`が並列に実行
    - ncc は Node.js のモジュールを依存関係も含めて一つのファイルにコンパイルする CLI っぽい
      - https://github.com/vercel/ncc
    - コンパイル前の準備的な？
  - `compile`はなんかめっちゃ色々なタスクを並列で実行してる
    - その中にある`cli`のタスクで`'src/cli/**/*.+(js|ts|tsx)'`に該当するファイルに対して外部プラグインで定義した`swc`を実行してる
      - https://github.com/lukeed/taskr/tree/master/packages/taskr#external-plugins
      - 外部プラグインは`task.source('src/*.js').{プラグイン名}()`で実行できる
      - `swc`プラグインは`packages/next/taskfile-swc.js`に定義されている
      - cli タスクでは`'src/cli/**/*.+(js|ts|tsx)'`のファイルに対して swc の transform を実行し、生成されたコードを`dist`に移動させてる
  - `./dist/bin`に出力されるには`bin`タスクが実行されている
    - これは`src/bin`配下のコードが swc でコンパイルされた後のコード
  - `next`コマンドの実装を見る(`packages/next/src/bin/next.ts`)
    - `next`で実行できるコマンドは`packages/next/src/lib/commands.ts`を見ると確認でき、`next dev`の場合は`require('../cli/next-dev').nextDev`が該当する
- `next dev`コマンドの実装に該当する`packages/next/src/cli/next-dev.ts`の`nextDev`を見る
  - [`--turbo`オプションに関する実装](https://github.com/vercel/next.js/blob/02f97ab0d9c5c32d14a8d035b44201241146ebe9/packages/next/src/cli/next-dev.ts#L323)
  - dev サーバーの実行は`bindings.turbo.startDev()`でされてそう
  - `bindings`は`loadBindings()`の返り値
  - `loadBindings()`は`packages/next/src/build/swc/index.ts`に定義されている
  - Turbopack のコード自体は`tryLoadWasmWithFallback`らへん？
  - `loadWasm`あたりが wasm 読んでそうだけども turbo に関しては`Wasm binding does not support --turbo yet`っぽい
  - Turbopack に関しては`resolve(loadNative(isCustomTurbopack))`かな
  - `loadNative`の[ここら辺](https://github.com/vercel/next.js/blob/02f97ab0d9c5c32d14a8d035b44201241146ebe9/packages/next/src/build/swc/index.ts#L380)かな
  - `bindings.startTurboDev(toBuffer(devOptions))`が該当してそう
  - `bindings`は`require("@next/swc/native/next-swc.${triple.platformArchABI}.node")`
- Rust から生成されたバイナリの binding は`packages/next-swc`で設定してる？
- binding には`napi`を使ってる
  - https://shisama.hatenablog.com/entry/2021/12/03/054437
  - Node.js をネイティブレベルで拡張するもの
  - C++のコードをコンパイルしたものが Node.js で実行できる
  - これが napi-rs では C++の部分が Rust で書けるようになる
  - ビルド(コンパイル)すると`.node`が生成される
  - napi-rs を使うと、パッケージを利用する環境に応じた`.node`が生成される
- next-swc で`pnpm build-native`してみる
  - build は crate 名が`next-swc-napi`のもの
  - `packages/next-swc/crates/napi`
  - startTurboDev は`packages/next-swc/crates/napi/src/turbopack.rs`が該当っぽい
  - `use next_binding::turbo::next_dev::{devserver_options::DevServerOptions, start_server};`
  - `next_binding::turbo::next_dev::start_server`が`next dev --turbo`で実行されている
- `next_binding`create はどこで定義されているのか
- `packages/next-swc/crates/napi/src/lib.rs`をみると`extern crate next_binding;`されてる
  - https://github.com/vercel/turbo/tree/main/crates/next-binding
  - が該当しそう

## `next dev --turbo`では`next_binding::turbo::next_dev::start_server`が使われていたので`next-binding`crate を見てみる

- https://github.com/vercel/turbo/tree/main/crates/next-binding
- `crates/next-binding/src/lib.rs`をみると`pub use next_dev;`だったので`next-dev`をみる
- `crates/next-dev/src/lib.rs`の`start_server()`
- `start_server()`では`NextDevServerBuilder::build()`の返り値(`server`)を使って、`server.future.await.unwrap()`を実行している
- `NextDevServerBuilder::build()`では`DevServer::listen()`した値から`DevServerBuilder::serve()`することで`DevServer`を返してる
- 最終的に`DevServer::future`してる
  - `join!`は引数で渡した非同期処理をポーリングしてくれる？
  - https://docs.rs/futures/latest/futures/macro.join.html
- `DevServer`の定義は crates/turbopack-dev-server/src/lib.rs

## Next

`crates/next-dev/src/lib.rs`の`start_server()`が何をやっているのか確認するところから

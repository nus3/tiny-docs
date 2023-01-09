## crates/turbopack/examples/turbopack.rs を実行できるようにする

```shell
cargo run --package turbopack --example turbopack
```

エラーが出る

```
thread 'tokio-runtime-worker' panicked at 'called `Result::unwrap()` on an `Err` value: rebasing package.json from demo onto out doesn't work because it's not part of the source path
```

FileSystemVc の値を確認するには`println!("{:?}", entry);`

`{:?}`で指定することで、タプルや配列をそのまま出力してくれる
https://zenn.dev/toga/books/rust-atcoder-old/viewer/13-format

```rust
let fs: FileSystemVc = disk_fs.into();
let entry = fs.root().join("demo/index.js");

println!("{:?}", entry);
```

demo ディレクトリがないのが原因？

```rust
let input = fs.root().join("demo");
let output = fs.root().join("out");
let entry = fs.root().join("demo/index.js");
```

input の値を出力したいけども`input.as_value_to_string().to_string()`だと StringVc になる
StringVc はベクタ型？
ベクタ型はサイズを変更可能な配列

root と name は確認
`name: project, root: home/dev/fork/turbo`

```rust
let root = current_dir().unwrap().to_str().unwrap().to_string();
let disk_fs = DiskFileSystemVc::new("project".to_string(), root);

println!(
    "name: {}, root: {}",
    "project".to_string(),
    current_dir().unwrap().to_str().unwrap().to_string()
);
```

vscode での rust の設定方法
https://code.visualstudio.com/docs/languages/rust

下記二つのどっちかの拡張を追加する

- Microsoft C++ (ms-vscode.cpptools)
- CodeLLDB (vadimcn.vscode-lldb)

Microsoft が出してるやつのが利用人数多いけど、なんかすぐは動かなかった
CodeLLDB だとすぐ動いた

root 直下に demo/index.js を追加すればいけそう？
追加してもエラーは変わらず
package.json がねぇって言われる
package.json も追加してみるか

README を読むとでも環境の構築には以下のコマンドを実行してと書いてあるので、実行してみる
demo/index.js と demo/package.json ができる気がしたけどもできてなさそう

```shell
cargo run -p node-file-trace -- print demo/index.js
```

node-file-trace の中身を見にいく(crates/node-file-trace)

簡単な demo/index.js と demo/package.json を作ってもダメだった

## turbopack の integration テストがどのように動いているのかを理解してみる

example が直接動かせないので、テストの実行からどうにかできないか画策する
crates/turbopack/tests/node-file-trace/integration にいろんなテストケースがありそうなので、これが使われてるテストが実行できればワンチャン

cargo-nextest とはなんなのか
https://blog.ymgyt.io/entry/cargo-nextest

`cargo test -p turbopack`
で turbopack 単体のテストができるのでは
https://doc.rust-jp.rs/book-ja/ch14-03-cargo-workspaces.html

`crates/turbopack/tests/node-file-trace.rs`に対してのテストはできた
が、めっちゃテスト落ちる

```
test result: FAILED. 33 passed; 99 failed; 0 ignored; 0 measured; 0 filtered out; finished in 7.81s
```

大体は npm の依存関係がないのが原因な気がしたので
`crates/turbopack/tests/node-file-trace`
で依存関係をインストールしてみる

エラーにはなったが、依存関係は入れることができたのでテストを再度実行してみる

```
│ node-pre-gyp ERR! build error
│ node-pre-gyp ERR! stack Error: Failed to execute 'home/.anyenv/envs/nodenv/versions/18.12.0/bin/node home/.cache/node/corepack/pnpm/7.18.2/dist/node_modules/node-g
│ node-pre-gyp ERR! stack     at ChildProcess.<anonymous> (home/dev/fork/turbo/crates/turbopack/tests/node-file-trace/node_modules/.pnpm/@mapbox+node-pre-gyp@1.0.10/node_mo
│ node-pre-gyp ERR! stack     at ChildProcess.emit (node:events:513:28)
│ node-pre-gyp ERR! stack     at maybeClose (node:internal/child_process:1091:16)
│ node-pre-gyp ERR! stack     at ChildProcess._handle.onexit (node:internal/child_process:302:5)
│ node-pre-gyp ERR! System Darwin 21.5.0
│ node-pre-gyp ERR! command "home/.anyenv/envs/nodenv/versions/18.12.0/bin/node" "home/dev/fork/turbo/crates/turbopack/tests/node-file-trace/node_modules/.pnpm/@mapb
│ node-pre-gyp ERR! cwd home/dev/fork/turbo/crates/turbopack/tests/node-file-trace/node_modules/.pnpm/canvas@2.10.1/node_modules/canvas
│ node-pre-gyp ERR! node -v v18.12.0
│ node-pre-gyp ERR! node-pre-gyp -v v1.0.10
│ node-pre-gyp ERR! not ok
│ Failed to execute 'home/.anyenv/envs/nodenv/versions/18.12.0/bin/node home/.cache/node/corepack/pnpm/7.18.2/dist/node_modules/node-gyp/bin/node-gyp.js configure --
└─ Failed in 4.4s
```

faild の数が増えた

```
test result: FAILED. 83 passed; 49 failed; 0 ignored; 0 measured; 0 filtered out; finished in 194.97s
```

ので作成された node_modules を削除

`cargo test -p turbopack --test node-file-trace`

`test node_file_trace_memory::case_002_array_map_require`のテストケースは ok

Rust でのテストの記述方法
https://doc.rust-jp.rs/book-ja/ch11-01-writing-tests.html

テストには`rstest`を使ってる
https://enu23456.hatenablog.com/entry/2022/12/09/221354
https://zenn.dev/soragiwa/articles/07ceb7258b6e58

test.each みたいなんが簡単にできる感じかな

---

> rstest は、フィクスチャベースのテストフレームワークで、フィクスチャとテーブルベースのテストを書くためのツールです。
> フィクスチャとは、テストを実行、成功させるために必要な状態や前提条件の集合のことを言います。
> また、テーブルベースのテストとは、一般的にテーブル駆動テストと呼ばれるもので、テストの入力と期待値をテーブルの行に記載し、テーブルを走査しながら実行していくテストのことです。

https://caddi.tech/archives/1849

rstest の使い方
https://crates.io/crates/rstest

テーブル的にテストケースをかけるのと、境界値のテストが書きやすくなる crate?

---

特定の 1 ケースのテストのみを実行する

`cargo test case_002_array_map_require -p turbopack --test node-file-trace`

`-- --nocapture`をつけることでパスされたテストの出力を確認できる

なのでテスト時の出力を確認するのには`cargo test case_002_array_map_require -p turbopack --test node-file-trace -- --nocapture`
にする

`fn node_file_trace`の中身を読んでいく

マクロを呼び出すと、macro_rules!で定義されたコードがマクロを呼び出した箇所で展開される

[Rust のモジュールシステムの基本](https://dackdive.hateblo.jp/entry/2020/12/09/103000)

- `r.block_on()`ってのが実行されてて、これは Tokio の Runtime のメソッド
- Rust の Future トレイトは JS の Promise
- `block_on`で引数に渡された非同期処理(Future)を実行する
- `register()`が実行されると turbo_tasks や turbopack の register 関数が実行されている
- `include!`は引数に指定したパス(ファイル)の中のコードを実行してくれる
  - [`include!`の使い方参考](https://qiita.com/elipmoc101/items/f76a47385b2669ec6db3#include)
- `concat!`は引数で渡された値を結合して文字列として出力する
  - https://qiita.com/elipmoc101/items/f76a47385b2669ec6db3#concat
- `env!`は環境変数を呼び出す
- `include!`の中の値は turbo/target/debug/build/turbopack-914cd64a9b27d0c6/out/register_test_node-file-trace.rs
- input には`node-file-trace/integration/array-map-require/index.js`
- directory には`/var/folders/jq/6641pq01033f13pdf37vszsm0000gp/T/tests_output/memory_node-file-trace/integration/array-map-require/index.js`
- テストの出力は temp_dir()にされており、$TMPDIR の箇所に出力されていた
- 前回のテスト実行後の出力結果を`remove_dir_all`で削除
- `#[cfg(not(feature = "bench_against_node_nft"))]`
  - cfg は環境に応じたコンパイルをする際に使用する
  - この書き方はアトリビュート
- `move`は所有権の移動をしてるっぽい
- `package_root`は turbo/crates/turbopack
- `original_output`が`exec_node(package_root, input)`したもの
  - `exec_node("turbo/crates/turbopack", "node-file-trace/integration/array-map-require/index.js")`
- `exec_node()`は中で tokio の Command を使っている
  - これは Rust が外部コマンドを非同期で実行するために利用している
  - Command の中身を見てみると`turbo/crates/turbopack/tests/node-file-trace/integration/array-map-require/index.js`を node で実行している
  - ts の場合は ts-node で実行している
- パッケージルート(リポジトリ内部)にある integration の js ファイルを実行したときの出力と
- test_outputs に出力されるタイミングは`emit_with_completion`の時？
  - `emit_with_completion`は`crates/turbopack/src/lib.rs`で定義されてるもの
  - 実際に`/memory_node-file-trace/integration/array-map-require/index.js`配下のファイルを削除して、`emit_with_completion`をコメントアウトすると、ファイルが生成されてなかったから正解っぽい
- node_file_trace はバンドルまでは含めない感じか？
  - 生成されるファイルがバンドルされているわけではない
  - あくまで対象の node_modules をちゃんと抽出できて、そこからエントリーポイントの JS が実行できるかどうかを確認してるだけ？

以下が`include!`の中身のコード

```rs
{
crate::COMMANDOUTPUT_IMPL_TRAIT_VALUEDEBUG_DBG_FUNCTION.register(r##"turbopack@TODO::::CommandOutput::ValueDebug::dbg"##);
crate::EXEC_NODE_FUNCTION.register(r##"turbopack@TODO::::exec_node"##);
crate::ASSERT_OUTPUT_FUNCTION.register(r##"turbopack@TODO::::assert_output"##);
crate::__register_CommandOutput_value_type(r##"turbopack@TODO::::CommandOutput"##, #[allow(unused_variables)] |value| {
    crate::__register_CommandOutput_ValueDebug_trait_methods(value);
});
}
```

architecture.md を読んでみた感じ、実行するとハッシュ付きのファイルが作成されて、それを実行する感じ？
`{crate_name}@{hash}::{mod_path}::{name}`の hash の部分がビルド時点のものになる的な

`println!("{:#?}", source)`のように`{:#?}`形式で指定すると改行ありで中身を出力してくれる

[`node-file-trace.rs`の最初のコミットはこれっぽい](https://github.com/nus3/turbo/commit/48649f94cf99857617bf6cdef813f78c31812083#diff-a8163c6caacc9f358198759f16dd9392d21a27f1b0af6e431c0092a122ad77e6)

- `cargo test case_071_react -p turbopack --test node-file-trace -- --nocapture`で`crates/turbopack/tests/node-file-trace/integration/react.js`のテストが実行できる
- が Error: Cannot find module 'react'
- なので `crates/turbopack/tests/node-file-trace`で pnpm install を試す
- するとテストがパスした

### node-file-trace.rs の流れをおさらい

1. `case::array_map_require("integration/array-map-require/index.js")`で integration 配下にある node プロジェクトを対象にテストを実行する
2. node プロジェクトのエントリーポイントを`node`で実行し、出力結果を`original_output`に入れる
3. Turbopack が提供する`emit_with_completion`を実行し、output ディレクトリに生成された node プロジェクトのエントリーポイントを実行し、出力結果が`original_output`と変わっていないかを確認する(`assert_output`)

## example を実行する方法

- 特定のディレクトリにアプリケーションを作成する(`npm init`等を実行して package.json を作る)
  - 例えば`demo/index.js`
- その後に`cargo run -p node-file-trace -- print {作成したnpmパッケージのエントリーポイント}`を実行する(今回の場合は`demo/index.js`)
- その後、`cargo run --package turbopack --example turbopack`を実行すると`out`ディレクトリ配下に turbopack が実行された後のファイルが出力される
- ただ、out ディレクトリを見てもバンドルされてるわけではなさそう

## 次は

`crates/turbopack/src/lib.rs`の`emit_with_completion`をよむ所から

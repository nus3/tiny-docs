## crates/turbopack/examples/turbopack.rs を実行する方法

おさらい

- 特定のディレクトリにアプリケーションを作成する(`npm init`等を実行して package.json を作る)
  - 例えば`demo/index.js`
- その後に`cargo run -p node-file-trace -- print {作成したnpmパッケージのエントリーポイント}`を実行する(今回の場合は`demo/index.js`)
- その後、`cargo run --package turbopack --example turbopack`を実行すると`out`ディレクトリ配下に turbopack が実行された後のファイルが出力される

## crates/turbopack/examples/turbopack.rs から実際に turbopack は何をやってるのかを読む

- turbopack の`emit_with_completion`が最初に実行されそう
- `Instant:now`とかは実行時間の計測をしてそう
- `Box::pin`
  - Rust はデフォルトでは全ての値がスタック(静的メモリ)に割り当てられる
  - Box を使うことで値をヒープ上に割り当てることができる
  - https://doc.rust-jp.rs/rust-by-example-ja/std/box.html
  - Box::pin を使うことで、ヒープに確保された変数になる
    - https://tech-blog.optim.co.jp/entry/2020/03/05/160000
- `emit_with_completion`に渡している`rebased.into()`
  - into は型変換に使われてるっぽい
    - https://doc.rust-jp.rs/rust-by-example-ja/conversion/from_into.html
  - でこの型は`AssetVc`で中身にはおそらく対象のファイル内容とかが取得できる構造体になってそう
    - Asset trait は path と content と references がある
- `impl A for B`で trait の実装を定義できる
  - [Rust の trait について](https://zenn.dev/mebiusbox/books/22d4c1ed9b0003/viewer/497a21)
- その asset を渡して`aggregate()`してる
  - aggregated が何なのか
  - `if current.references().await?.set.is_empty()`じゃない限り、`aggregate_more()`をしてる
  - 依存関係を追加してってる？
  - references は aggregate_more をすることで全てがなくなった
  - content は Asset の構造体から Children になって、turbo::task の配列に変化された
  - depth は 2
    - demo でのサンプルコードのエントリーポイントでは二つのモジュールに依存している(import)からその数？
    - リポジトリ内部の別のモジュールを import しても depth は 2 のまま
    - package.json の devDependencies を見てる？
    - devDependencies にパッケージを一つ追加したら depth は 3 になった
    - devDependencies を 3 つのままでエントリーポイントで使うパッケージを減らしたら 2 になった
    - depth はエントリーポイントから依存している依存パッケージの数(リポジトリ内部のモジュールは含めない)
- `aggregate`された後のものを`emit_aggregated_assets`に渡してる
  - `aggregate`された`content()`が Asset のば際は`emit_asset_into_dir`
  - Children の場合は children 分、`emit_aggregated_assets`を実行する
- ってことで`emit_asset_into_dir`へ
  - asset の path が output_dir の内部にあれば、`emit_asset` する
  - asset の path は`path: "out/node_modules/react/cjs/react.production.min.js",`
  - output_dir の値は`path: "out"`,
- `emit_asset`では`asset.content().write(asset.path())`してる
  - `write()`では asset が File の場合は asset の path にその内容を出力してる
  - file には`parse_json()`が生えていたので、中身を確認してみた
- バンドルをしてるっぽいところは確認できず
  - そもそも example を実行してもバンドルはされてない

### `aggregate()`

aggregate 後の content

```
Children(
    {
        AggregatedGraphVc {
            node: TaskCell(
                TaskId {
                    id: 1941,
                },
                CellId {
                    type_id: ValueTypeId {
                        id: 312,
                        name: "turbopack@TODO::::graph::AggregatedGraph",
                    },
                    index: 0,
                },
            ),
        },

```

### `emit_asset()`

```rs: crates/turbopack-core/src/asset.rs
            AssetContent::File(file) => {
                dbg!(file.parse_json().dbg().await?);
                path.write(*file)
            }
```

を実行した結果、package.json の中身をパースしてる感じ

```
[crates/turbopack-core/src/asset.rs:121] file.parse_json().dbg().await? = Content(
    Object {
        "name": String("react"),
        "description": String("React is a JavaScript library for building user interfaces."),
        "keywords": Array [
            String("react"),
        ],
        "version": String("18.2.0"),
        "homepage": String("https://reactjs.org/"),
        "bugs": String("https://github.com/facebook/react/issues"),
        "license": String("MIT"),
        "files": Array [
            String("LICENSE"),
            String("README.md"),
            String("index.js"),
            String("cjs/"),
            String("umd/"),
            String("jsx-runtime.js"),
            String("jsx-dev-runtime.js"),
            String("react.shared-subset.js"),
        ],
        "main": String("index.js"),
        "exports": Object {
            ".": Object {
                "react-server": String("./react.shared-subset.js"),
                "default": String("./index.js"),
            },
            "./package.json": String("./package.json"),
            "./jsx-runtime": String("./jsx-runtime.js"),
            "./jsx-dev-runtime": String("./jsx-dev-runtime.js"),
        },
        "repository": Object {
            "type": String("git"),
            "url": String("https://github.com/facebook/react.git"),
            "directory": String("packages/react"),
        },
        "engines": Object {
            "node": String(">=0.10.0"),
        },
        "dependencies": Object {
            "loose-envify": String("^1.1.0"),
        },
        "browserify": Object {
            "transform": Array [
                String("loose-envify"),
            ],
        },
    },
)
```

---

`file.lines()`で file の 1 行 1 行の内容を取得できそう

```rs
            AssetContent::File(file) => {
                dbg!(file.lines().dbg().await?);
                path.write(*file)
            }
```

```
        FileLine {
            content: "  return renderToStringImpl(children, options, true, 'The server used \"renderToStaticMarkup\" which does not support Suspense. If you intended to have the server wait for the suspended component please switch to \"renderToPipeableStream\" which supports Suspense on the server');",
            bytes_offset: 245082,
        },

```

## 雑メモ

- `crates/turbo-tasks/src/debug/mod.rs`の`ValueDebug`を見てみると`dbg!(any_vc.dbg().await?);`みたいな書き方でデバッグできそう？
  - `dbg!`マクロは式と結果を出力してくれるマクロ
    - どの行かとかも出力してくれる
    - https://zenn.dev/masinc/articles/rust-macro-dbg

`〇〇Vc`の中身の確認は以下のように`dbg!`を使うと中身の確認ができる

```rs
    // assetはAssetVc
    dbg!(asset.dbg().await?);
```

`crates/turbopack/architecture.md`をちゃんと読めば書いてある

```
- You can read the data via `.await?`
  (e.g. `let x: XxxVc; let data: Xxx = x.await?`);
```

---

## 次やること

バンドルしてそうな処理を探す

turbopack 関連の他の crate を探してみる

`crates/turbopack-create-test-app`を実行してみるのを試しても良さそう

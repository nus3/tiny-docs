# memo

## 環境構築

## README を読んでみて

https://github.com/vercel/turbo/blob/main/CONTRIBUTING.md

- Turbo には二つのツールがあるよ。Turborepo と Turbopack
- この二つのツールはそれぞれ別の言語とツールチェイン使ってるけど、最終的には Rust に統一するよ
- それまではそれぞれの CONTRIBUTING を見てね
- Turbopack では Cargo のワークスペースを使ってる
- crate を部分的に使う場合は`cargo run -p [CRATE_NAME]`コマンドで実行してね
- テストの実行は
  - `cargo install cargo-nextest`
  - `cargo nextest run`
  - テストの実行するためにはあらかじめ yarn をしておく
  - yarn 使うと pnpm 使えって怒られるので `pnpm install`を使う
    - Usage Error: This project is configured to use pnpm
  - 何個かテストコードが落ちるので、それは確認する
- デモ環境の構築は`cargo run -p node-file-trace -- print demo/index.js`

## crates/turbopack/architecture.md を読んでみて

- turbopack は turbopack-core, turbopack-css, turbopack-ecmascript、そして turbopack に分かれている
- 現在、実装が足りていないのは turbopack-node, turbopack-wasm, turbopack-image など

## その他

- GitHub Actions にスケジュールされた Action があるのでそれを disabled にした
- publish docs の action も削除

# Tonc 日本語版（非公式）

これは、GBA プログラミングの入門書 Tonc（[原文](https://gbadev.net/tonc/)、[リポジトリ](https://github.com/gbadev-org/tonc)）の**非公式**の日本語訳です。原作者や gbadev による公式のもの、または承認を受けたものではありません。

- 原作: Jasper "cearn" Vijn（(c) 2004-2013）、TONC contributors（(c) 2023）
- 変更点: 英語の原文を日本語に翻訳しています。
- ライセンス: 原文と同じく [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.ja) です。ライセンスに定めるとおり、無保証で提供されます。
- 翻訳の誤りは [sacckey/tonc の Issues](https://github.com/sacckey/tonc/issues) に報告してください。原文の内容については [gbadev-org/tonc](https://github.com/gbadev-org/tonc) へどうぞ。

公開先: https://sacckey.github.io/tonc/

## Setup

You need Python, Rust and mdBook.

```sh
# clone the repo
git clone --recurse-submodules git@github.com:sacckey/tonc.git
cd tonc

cargo install mdbook@0.4.52

# run the development server
mdbook serve --open
```

The book will be live at http://localhost:3000

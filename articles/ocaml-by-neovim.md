---
title: "Neovim で OCaml を書く"
emoji: "🐫"
type: "tech"
topics:
  - "neovim"
  - "ocaml"
published: true
published_at: "2022-12-30 17:19"
---

https://discuss.ocaml.org/t/ocaml-5-0-0-is-out/10974 によって、注目を浴びている[要出典] OCaml ですが、ここらで自分の環境を晒しておくことでユーザーが増えるのではないかと思ったので公開します。
ただ、 Vim 系を使っているのは僕のこだわりでしかないので、普通に Emacs とか使える人はその方が良いかもしれません……。

# 前提環境
## OCaml について
https://ocaml.org/docs/up-and-running に書かれていますが、基本的にまず必要なのは`opam`です。
これは他の言語で言う`pip`とか`npm`とかみたいなパッケージマネージャですが、OCaml 本体もこれで入れるのが良いです。
```
brew install opam
```
とかで入ります。他の環境でも似たような感じ。

`opam`が入ったら、
```
opam switch create 5.0.0
```
などとしてデフォルトの環境を作りましょう。
これで複数のバージョンを使い分けたり、ディレクトリスイッチを作ったりできます。
便利。
```
eval $(opam env)
```
が必要なこともあるので、表示されるメッセージ通りに打ったり打たなかったりしてください。
過剰に打って悪いことはないので大体困ったら打った方が良いです。
```
opam switch
```
で今の環境がどうなっているかの確認が出来るので、そちらを打ってみても良いかもしれません。

## Neovim について
Neovim については雑に最新を brew で入れてます。（執筆時点で`v0.8.1`）
また、暗黒の力を信じているので plugin マネージャーとして、https://github.com/Shougo/dein.vim を使っています。

# Neovim + OCaml
https://ocaml.org/docs/up-and-running#configuring-your-editor
まずは、公式ドキュメントから。

OCaml には`merlin`というパッケージ(https://github.com/ocaml/merlin)もあり、補完等々のエディターサポートをしてくれます。
ただ、この補完というのが、Vim だとオムニ補完になっています。これが少し厄介で、古い補完のやり方なのでプラグインによっては対応してないです。
(例えばこれ https://zenn.dev/shougo/articles/ddc-vim-beta#omni_patterns)

わざわざコマンドを打ってまでオムニ補完を呼び出すのも面倒なので LSP に頼ります。
LSP としては`ocaml-lsp-server`というパッケージがあるので、これを
```
opam install ocaml-lsp-server
```
で入れます。
余談ですが、既に 5.0 にも対応しており、気合の入り方が感じられますね。

LSP を入れるということは Neovim の側でも設定をやる必要があり、面倒なので https://github.com/neovim/nvim-lspconfig を入れます。

参考程度に自分の dotfiles を貼っておきます。
https://github.com/lmdexpr/dotfiles/tree/master/config/nvim

ついでに、フォーマッタが欲しい人は
```
opam install ocamlformat
```
して、適当にフォーマット設定を書くと良いです。
参考 : https://github.com/lmdexpr/ocaml_jvm/blob/main/.ocamlformat

# まとめ
1. opam を入れる
2. opam で以下を入れる
	- OCaml 本体
	- ocaml-lsp-server
	- (欲しいなら) ocamlformat
3. neovim で lsp を設定する
	- nvim-lspconfig 辺りがおすすめ

OCaml と言わず、Neovim で LSP がある言語書く時は大体似たような設定をしておけば動くと思います。

# 余談
Coq も Neovim で書いてます。
https://zenn.dev/lmdexpr/scraps/47f1dd6162ad28
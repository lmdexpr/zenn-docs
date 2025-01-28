---
title: "Nix flakes で OCaml project を管理する"
emoji: "❄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "nix"
  - "ocaml"
published: true
published_at: 2023-06-25 14:00
---

# 背景
先日、NixOS を導入し、dotfiles を一通りアップデートしました。
(記事はないので適当に https://github.com/lmdexpr/dotfiles を見てください)

準備ができたぞ、というところで、いざ OCaml を書こうとしてみると、うまく動かせない。
例えば`opam switch`で環境作ろうとすると ld がなく、コンパイラを入れることも出来ない。
そっちはそっちで解決策もあるようですが、折角なら Nix flakes を用いてプロジェクトごとに環境を再現できるようにしてしまおうと思いました。

その記録です。

# 前提知識

## Nix flakes
[Nix](https://nixos.org/)はパッケージマネージャです。
その記述に使うのは Nix 言語という独自の関数型言語です。非常に簡単です。
システム自体の記述も行えますし、そこに乗っかった Linux distribution が NixOS です。

一方で[Flakes](https://nixos.wiki/wiki/Flakes)は Nix 上の実験的機能になります。
`flake`を扱うための機能であり、`flake`と呼ばれるものは、`flake.nix`をルートに持つ file system tree (git や tarball も含みます) のことです。
それを管理するということはつまり、ビルド方法を指定したり、その過程で必要な依存関係を指定したりすることが出来ます。

## OCaml の普通の開発環境について
パッケージマネージャーとして opam が存在します。^[dune に統一したい機運があるようです。 https://ocaml.org/tools/platform-roadmap]
`hoge.opam`ファイルに hoge というプロジェクトについて記述することが出来ます。
また、`opam switch`というコマンドがあり、これによってディレクトリごとの環境を立てられます。一方で、前述のように Nix とは相性が悪いです。

次にデファクトスタンダードなビルドシステムとして dune が存在します。
Jane Street という OCaml を使っている企業として有名なところが OSS として出しているものです。
設定の記述が S 式になっていて、扱いやすいです。
`dune-project`にてプロジェクトの依存なども記載することができ、そのファイルから`*.opam`を生成することも出来ます。

Nix 以前は OCaml のプロジェクトを作成したくなったら、

1. `opam`をインストール
2. `opam install dune`で`dune`をインストール
3. `dune-project`ファイルを記述し`dune build`する(ここで`*.opam`も生成出来る)

という形で組み立てていました。

ところで、これは僕が知らないだけでどうにか出来るかもしれませんが opam -> dune -> opam みたいな依存になっていることが気になります。

# OCaml の開発環境を Nix flakes で組み立てる
今回の要件は以下の通りです。

- 既存の`*.opam`や`dune-project`を活用したい
- 開発時の環境として`ocaml-lsp-server`をマストとして、パッケージを管理したい

一般的に、各言語に存在するパッケージマネージャとの連携が Nix に存在しています。
例えば NodeJS であれば`node2nix`、Rust であれば`cargo2nix`などです。

OCaml に関しても`opam2nix`が存在しています。

## `opam2nix`を使う
https://github.com/tweag/opam-nix という便利なテンプレートがあるので使います。

README にもありますが https://www.tweag.io/blog/2023-02-16-opam-nix/ というブログポストに詳細な使い方もあります。
これをなぞるだけで本当に一瞬で環境が作れてしまいます。
例えば以下の手順。

1. `*.opam`を準備する
2. `nix flake init -t github:tweag/opam-nix`で出来た`flake.nix`を適宜修正
3. `git add .` (nix は git にトラックされているかとかを見てたりします(一敗))
4. `nix develop`

これで完璧です。
実際に自分が使用している`flake.nix`の例。

```nix
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixpkgs-unstable";
    flake-utils.url = "github:numtide/flake-utils";
    opam-nix.url = "github:tweag/opam-nix";
  };
  outputs = { flake-utils, nixpkgs, opam-nix, ... }:
    flake-utils.lib.eachDefaultSystem (system:
      let
        pkgs = nixpkgs.legacyPackages.${system};
        on   = opam-nix.lib.${system};

        src = ./.;
        # プロジェクト内の *.opam ファイルから対象パッケージ名を取得
        localNames = with builtins;
          let
            files      = attrNames (readDir src);
            opam_files = map (match "(.*)\.opam$") files;
            non_nulls  = filter (f: !isNull f);
          in 
            map (f: elemAt f 0) (non_nulls opam_files);

        # クエリに変換しておく
        localPackagesQuery = with builtins; 
          listToAttrs (map (p: { name = p; value = "*"; }) localNames);

        devPackagesQuery = {
          ocaml-lsp-server = "*";
          ocamlformat = "*";
          utop = "*";
        };

        # 開発のためのパッケージもプロジェクトとバージョンを合わせて連携するためにクエリを混ぜる
        query = devPackagesQuery // localPackagesQuery;

        localPackages =
          # 複数プロジェクトをビルドする時は ' 付きを使う
          on.buildOpamProject'
          {
            inherit pkgs;
            resolveArgs = { with-test = false; with-doc = false; };
            pinDepends = true;
          }
          src
          query;

        # インストールされた内の開発用パッケージだけを取り出す
        devPackages = builtins.attrValues
          (pkgs.lib.getAttrs (builtins.attrNames devPackagesQuery) localPackages);
      in
      {
        legacyPackages = pkgs;

        # nix build で対象になるパッケージ
        packages = {
          default = localPackages.discord;
        };

        # nix develop や direnv で起動される環境
        devShells.default =
          pkgs.mkShell {
            inputsFrom  = builtins.map (p: localPackages.${p}) localNames;
            buildInputs = with pkgs; [ 
              nil nixpkgs-fmt 
            ] ++ devPackages;
          };
      });
}
```

### Dune を使う
上記のブログポストにもありますが`dune-project`を使うことも出来ます。
つまり、最早`opam`を(明示的に)使うこともなくなりました。

その場合 1. で準備した`*.opam`の代わりに`dune-project`を配置し、2.のテンプレート修正時に`buildOpamProject`を`buildDuneProject`に変えれば良いです。

### Direnv を使う
開発の度に`nix develop`するのはダルすぎます。
なので nix 的によく使われているらしい Direnv を使います。^[ぼくは今回初めて知りました]
プロジェクトルートで`direnv edit .`して、中に`use flake`とだけ書いておくと`cd`するだけで諸々の環境がセットアップされます。
最高。

## `opam2nix`を使わない
おまけ的にパッケージマネージャを使わない方法を紹介します。

nix には`ocamlPackages`というパッケージセットや`buildDunePackage`というビルドのために使える関数があります。
必要な依存を自分で記述する必要こそあり、手間はかかりますが一例ということで自分が discord library を記述した時に仕様した例を貼っておきます。

```nix
{
  # opam2nix を使用する際は inputs に依存として追加する必要がある
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixpkgs-unstable";
    flake-utils.url = "github:numtide/flake-utils";
  };

  outputs = { flake-utils, nixpkgs, ... }:
    flake-utils.lib.eachDefaultSystem (system:
      let
        pkgs = nixpkgs.legacyPackages.${system};

        # ocaml 5.3 を使いたいので固定
        ocamlPackages = pkgs.ocaml-ng.ocamlPackages_5_3;

        # 依存するパッケージを記述しておく
        dependencies = with ocamlPackages; [
          yojson
          ppx_yojson_conv
          logs
          hex
        ];

        # ビルド対象のパッケージ
        localPackages = rec {
          # opam で公開されていない sodium ライブラリを使うために自前でビルド
          # opam を使う場合でも pin という方法で同様のことが出来る
          sodium = ocamlPackages.buildDunePackage {
            pname = "sodium";
            version = "dev";
            src = pkgs.fetchFromGitHub {
              owner  = "ahrefs";
              repo   = "ocaml-sodium";
              rev    = "734eccbb47e7545a459a504188f1da8dc0bd018e";
              sha256 = "sha256-anm9sM7xeRdxdwPDpHsKb93Bck6qUWNrw9yEnIPH1n0=";
            };
            buildInputs = [
              pkgs.libsodium
              ocamlPackages.dune-configurator
            ];
            nativeBuildInputs = [ 
              pkgs.pkg-config
            ];
            propagatedBuildInputs = with ocamlPackages; [
              base
              ctypes
            ];
          };

          # ビルド対象
          discord = ocamlPackages.buildDunePackage {
            pname = "discord";
            version = "0.1.0";
            src = ./.;
            buildInputs = [ sodium ] ++ dependencies;
          };
        };

        # 開発のためのパッケージ
        devPackages = with ocamlPackages; [
          ocaml-lsp
          ocamlformat
        ];
      in
      {
        legacyPackages = localPackages;

        # nix build で対象になるパッケージ
        packages = {
          default = localPackages.discord;
        };

        # nix develop や direnv で起動される環境
        devShells.default =
          pkgs.mkShell {
            inputsFrom  = builtins.attrValues localPackages;
            buildInputs = with pkgs; [ 
              nil nixpkgs-fmt 
            ] ++ devPackages
              ++ builtins.attrValues localPackages;
          };
      });
}
```

# おわり
Nix サイコー

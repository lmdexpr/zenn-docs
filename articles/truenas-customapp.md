---
title: "TrueNAS Scale でカスタムアプリケーションをデプロイする"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
    - "TrueNAS"
    - "Docker"
published: true
---
# やりたいこと
みなさ～ん、[TrunNAS Scale](https://www.truenas.com/truenas-scale/)、使っていますか。
僕は使っています。

正直に言ってめちゃくちゃおすすめ！　と言う気はないのですが、適度に使いやすく、適度にカスタマイズが出来るので、NAS + 何か的な用途であれば必要十分かと思います。

「何か」ってなんやねんという話なのですが、例えば、マイクラがボタンポチポチで動くらしいです。
そういうボタンポチポチは Web UI ですぐ設定できるのでとても嬉しい。

ところで、僕は Discord bot を作っているのでそれを動かしたいです。
カスタムアプリケーションをどうやって動かすかというと Docker です。

True NAS には Core と Scale がありますが、Scale の方は Debian ベースになっており、良い感じに Docker アプリケーションを動かしてくれます。
ただどうにもドキュメントが充実しておらず、困ったのでメモ書きを残しておきます。

# 手順

## 1. image を用意する
public なところに置けるなら問題ないです。
今だと Github Container Registry とかが使えるので、そこに置いておきます。

色々変なこともしていますが[こんな感じ](https://github.com/lmdexpr/yggdrasill/blob/main/.github/workflows/ratatoskr.yml)で Github Actions からプッシュまでやれます。
助かる。

## 2. custom app をインストール
TrueNAS Scale の Web UI からインストールします。

`Apps` -> `Discover Apps` -> `Custom App` からインストールできます。
例えば Github Container Registry から取得する場合は `ghcr.io/<user>/<image>` を Image repository に入力すると良いです。

## 3. 設定
あとは適宜設定、するだけ！

# プライベートのイメージを使いたいよ
最終的にパブリックにしたのですが、正直、プライベートのイメージを使いたいです。
その場合には `Apps` -> `Settings` -> `Manage Container Images` から image を事前に pull しておけば使えます。

ただ、この場合は pull する度に認証情報を入力したり、image を更新しても自動的には反映されないので面倒です。
この辺りをうまくやる方法があるのであれば嬉しいのですが、最初に書いた通りドキュメントが充実していないので、よくわかりません。
誰か知っていたら教えてください。

# おわりに
書いてみて思ったのですがプライベートのイメージを使わないなら簡単です。

ただ、TrueNAS Scale ではなく、他の Linux を使う方が実は楽なのでは？　と何度も思いました。
分かってしまえば使いやすいのですがカスタムアプリケーションを動かしたい人は TrueNAS Scale でなくとも良い気がします。

そうは言っても Web UI のカッコよさと便利さは嬉しいのですが。

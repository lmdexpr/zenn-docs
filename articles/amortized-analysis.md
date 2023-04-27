---
title: "償却解析を理解する"
emoji: "🚀"
type: "tech"
topics: []
published: false
---

# 背景
いわゆる積ん読になっていた https://tatsu-zine.com/books/purely-functional-data-structures (参考文献 1.) を読んでいる。
「第5章 償却の基礎」にさしかかり、償却の概念自体はなんとなく知っていたが、なんとなくでしかないことに気付いたので良い機会だと思い、まとめる。

~~ついでに社内勉強会での発表ネタにするため、想定読者を「計算量」や「アルゴリズム解析」といった言葉を知らない、もしくは知っているがなんとなくの理解である、というレベルにおき、記事とした。~~
急遽社内勉強会では別資料を用意するようになったので、レベル感の意識は少し薄れた。

その性質から誤りがあれば、訂正やツッコミのような批判は歓迎である。

# アルゴリズム解析の基礎

## Motivation
そもそも「アルゴリズム解析」とは何か、というと、あるアルゴリズムに対するリソースの見積もりと定義される。
ここでリソースとは時間や、記憶領域のことを指し、実際にはコンピュータが計算する時の時間や、メモリにどの程度のデータがのるか、ということを指している。
では、なぜそのような見積もりが必要かといえば、リソースはあればあるほど良いので、減らせる余地を見るためにも多い／少ないの判断が必要になるからだ。
つまり、やりたいことはリソースの最適化、特にここではプログラムの高速化に注目する。

さて、プログラムを高速化するにあたって必要なことはなんだろうか。
例えばpodの数を増やす、メモリを増やす、といった選択肢は十分に考えられる選択肢だ。これは時間的リソースを金銭的リソースによって代替しているに過ぎないが、当然それで良いならそうすれば良い。
他にも言語の選定を変えることや、インフラを変えてみたりというような手段は十分にとれる。

そして、今回の主題であるアルゴリズムの効率化も一つの手段だ。
より良いアルゴリズムを選定するために何が必要だろうか？
例えば同じコンピュータに比較したいアルゴリズムを実装し、実際にテストデータを用意し、時間を比較するということは可能だろう。
しかし、この手法にはある程度問題があることは既に知られている。
例えば、
- コンピュータの実行環境は本当に同じだっただろうか？
	- 他にプログラムが動いていたり、リソースが食い潰されていなかったのか精査出来るか？
- 用意したデータは十分に意味があるものであったか？
	- 極端に少ないデータでは差が出ないことはままある
	- 本番では大規模データを処理するが、テストでそれをやる訳にもいかない、という事態も十分ある
- その他にも色々……

このように注意深く実験を繰り返すことも重要だが、上に上げたように幾つかの問題が既に知られている。
そこで「アルゴリズム解析」である。

## 計算量
計算量とは、そのまま計算の量のことである。
そして、上で書いたように、今回取り上げたいのは「あるプログラムが1sで終わるのか」というようなことではない。
つまり、実際に何個のオペコードになるか、とか、「計算」の回数が何回か、ということには興味はあれど、そのことを考えるのが十分難しいということは既によく知られている。

では、どうするかというと「ある2つのアルゴリズムのどちらのほうが効率的か」を考え、より効率的なアルゴリズムを選択する、ということをしたい。
つまり、比較を行いたい訳だが、この時、（一般に）必要になるのは指標（不変量）である。

ここからは特に入力データセットのサイズに依存するものを考える。
多くのケースでは、入力されるデータセットが膨大になれば計算の回数が増え、実行時間が増えることから十分価値のある指標になるからだ。
これは、入力データセットのサイズを引数にとり、プログラムの実行時間を返す関数の成長率とでも言うべきものを考えることになる。
例えば、（一旦どのような値かは無視して）成長率１のプログラムと成長率１０のプログラムがあったとすると、成長率１０のプログラムの方が膨大なデータセットに対して実行時間が遅くなるだろうと考える。
これにより計算量を見積もることができる。

### 関数の成長率を解析的にアプローチする
関数の成長率を解析するということは、より一般に解析学（数学）で考えられる。
そこでよく用いられるのが「ランダウの記号」だ。
これは「オーダー記法」などとも呼ばれるもので、以下に定義を示す。^[参考文献　３．　の p.114 を元とした]

:::details 定義
**Def.**
$D \subset \reals \cup \{ \pm \infin \}, a \in D$
$g : D \rightarrow \reals, a の \reals における除外近傍 U 上で g(x) \ne 0$

$f : D \rightarrow \reals$ が、
$\lim_{x\rightarrow a,x\ne a}\frac{f(x)}{g(x)} = 0$ を満たす時、
**(a において) f は g に比べて無視できる**といい、$f\ll g$と書く。
また、a において $f\ll g$ となる任意の関数 f を$\omicron(g) (x \rightarrow a)$と書く（ランダウの記号）。

更に、$\frac{|f|}{|g|}$ が除外近傍 $V (\subset U)$ の上で有界であれば
$x \rightarrow a$ のとき、 **f は g で押さえられる**といい、 $f \curlyeqprec g$ と書く。
また、同様に$f \curlyeqprec g$ となる任意の関数 f を $\Omicron(g) (x \rightarrow a)$ と書く。
:::

これは数学に慣れ親しんでいない人からすると読み解くのに苦労するかも知れない。
実際に以下の話を理解するのに、この式を完全に理解する必要はなく、特に $\omicron(g)$ や $\Omicron(g)$ という記法について取り上げる^[オー記法やビッグオー記法とも呼ばれるが記号としてはオミクロン（ギリシャ文字のO）だ]。

上では「無視できる」、「押さえられる」というような言葉が定義された。
ここから推測できるように、この定義はある関数 f が ある関数 g より十分に小さいと表現したい時に用いられる。
雑な説明をしておくと $\Omicron(g)$ は g 以下の”オーダー”を持つ（g が上限である）ことを、 $\omicron(g)$ は g より真に大きい”オーダー”を持つことを意味する。

少し具体例を見よう。

$$
\begin{aligned}
f(n) & = 3n^2 + n + 100 \\
g(n) & = n^2
\end{aligned}
$$
とすると、

$$
\begin{aligned}
\lim_{n \rightarrow \infin}\frac{f(n)}{g(n)} = lim_{n \rightarrow \infin}{3 + \frac{1}{n} + 100 \frac{1}{n^2}} = 3
\end{aligned}
$$
となる。
これは有界なので $f=\Omicron(n^2)$ と評価される。

ところで、これは最も影響力の大きい（支配的な）項を定数倍を無視して残した形となる。
雑に言ってしまえば、最も大きくなりうる項だけを残せば評価できてしまうというのがオーダー記法の嬉しいポイントの一つだ^[そして大抵の場合、実用的にはこの程度の理解で問題ない]。

これをアルゴリズム解析に応用することは容易で、つまり、あるアルゴリズムAを実装したプログラムに対する評価関数と、別のアルゴリズムBを実装したプログラムに対する評価関数を同様に評価すれば良い。
では、使ってみよう。

### $\omicron/\Omicron$記法を使ってみる
まずはよく使われる関数を準備する。
慣例に習い、引数は x ではなく、 n を用いるものとする。

| 定数 | 対数 | 下位線形 | 線形 | 線形対数 | 二乗 | 指数 |
| --- | --- | --- | --- | --- | --- | --- |
| $1$ | $\log n$ | $n^d (d < 1)$ | $n$ | $n \log n$ | $n^2$ | $2^n$ |

この表は左から $\curlyeqprec$ の順に並んでいる。
つまり $1 \curlyeqprec \log n \curlyeqprec n^d \curlyeqprec n \curlyeqprec n\log n \curlyeqprec n^2 \curlyeqprec 2^n$ となっている。^[log の底が省略されているが、何であっても高々定数倍にしかならないので具体的な数字は必要ない。必要なら 2 だと思っておくのが良い]

では、幾つかの例を見てみよう。
コード例は OCaml とした。

```ocaml
let _ = 2 + 3
```
単純な足し算を行うプログラムだが、この計算量は $\Omicron(1)$ だ。
どのような命令にコンパイルされるかは今回問わず、あくまで一つの計算とみなすからだ。

```ocaml
let rec binsearch ~ok left right =
  if abs (right - left) <= 1L then right
  else
    let mid = (right + left) / 2L in
    let left, right = if ok mid then left, mid else mid, right in
    binsearch ~ok left right
```
次に見るのは「二分探索」のプログラムだ。
このアルゴリズムは一回の探索ごとに探索範囲を二分するためにこのような名前がついている。
では、何回の探索が行われるかというと$1 + \lfloor \log_2 n \rfloor$回になる。
具体的な計算や説明は参考文献の 2. に任せるとして、上の例も考えると$\Omicron(\log_2 n)$と評価されることが分かる。

もっと多くの多彩な例については参考文献を参照。
特に 4. はかなり多くの例が分かりやすく説明されている。

### その他の記法について
計算理論の分野では $\Omega(g)$ や $\omega(g)$、 $\Theta(g)$ というような表記も用いられる。
定義までは持ち出さないが、簡単に説明をしておく。

$\Omicron(g)$ は g 以下であるということを表していた。
アルゴリズム的には最悪ケースでも g 程度の"時間"^[g は関数であるからこの表現は多少の誤りがある]で済むということになる。
では、最良ケースも考えたくなるのが人情というもので、それが $\Omega(g)$ だ。

同様に $\omicron(g)$ と $\omega(g)$ も対応している。

最後に $\Theta(g)$ が何かと言えば、これは $\Omicron(g) = \Omega(g)$ である場合を表す。
最良と最悪が同じであるということを端的に表すのは、それはそれで使い勝手の良いものである。

簡単には　$\Omicron(g)$ を $\Theta(g)$ や $\Omega(g)$ と同じ意味として用いる流儀も存在する。
今回は必要に応じて厳密な議論と理解のために記号を使い分けることとする。

# 償却解析

## Motivation
さて、ここまであるアルゴリズムに対する計算量について説明してきたが、例えばこんなケースはどうだろう。

todo


# 参考文献
1. Chris Okasaki(著), 稲葉一浩, 遠藤侑介(訳) . 純粋関数型データ構造 . アスキードワンゴ . 2017
2. George T. Heineman、Gary Pollice、Stanley Selkow(著), 黒川 利明, 黒川 洋(訳) .
アルゴリズムクイックリファレンス 第2版 . オライリー・ジャパン . 2016
3. 杉浦光夫(著) . 解析入門 Ⅰ . 東京大学出版会 . 1980
4. けんちょん (Otsuki) . "計算量オーダーの求め方を総整理！ 〜 どこから log が出て来るか 〜" . Qiita . 2021 . https://qiita.com/drken/items/872ebc3a2b5caaa4a0d0, (参照 2023-04-22)
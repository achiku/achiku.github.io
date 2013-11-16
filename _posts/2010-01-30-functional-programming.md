---
layout: post
title: '関数的であるということ'
---

イマイチしっくりとこなかったんだけど、Pythonを使ってみて何となく、ほんのちょこっと関数的であるということが分かってきた気がする。今自分が実感できる「関数的」なことは大きく分けると二つ。1.抽象化する、2.手続きではなく定義を書く

こっから先はプログラミング歴2年の、しかも仕事でプログラミングあんましてない趣味プログラマが書くことなので、色々間違い・勘違い等含まれていると思うので注意して読まれたし。

### 抽象化する
以前にもkshで関数的に書くっていうエントリを書いた。

[プラグマティック関数プログラミング]({% post_url 2009-07-08-pragmatic-functional-programming %})

言いたい事はほぼ、このエントリの中にあるんだけど、もう一度まとめる。プログラムがおおきくなるにつれて、プログラムを変更、追加する必要がある場面が増える。そういった時に簡単に変更、追加を行えるようにプログラム同士はなるべく緩くつながっていて欲しいという需要がでてきた。緩くつながるってのはどういうことか。簡単に言うと「取り替えるのが簡単」ってことだ。オブジェクト指向はこの取り替えがなるべく簡単にできるように、クラス、オブジェクト、継承ってのを使ってアプローチしている。関数的なものは、言うに及ばず、関数を使ってこの「取り替えるのが簡単」っていうベネフィットにアプローチする。以下、上記エントリの引用。


> 上のスクリプトが言っているのは「二つ目の引数で与えられたディレクトリ以下のファイルにほげほげしろ」ってこと。この「ほげほげ」って部分が第一引数になる。「二つ目の引数で与えられたディレクトリ以下のファイルにほげほげしろ」っていう指示は抽象的である。抽象的であるが故に「ほげほげ」の部分を自由に組み替えることができる。それはつまり、拡張性・再利用性が高いってことだ。

つまり、「ほげほげ」の部分を空白にしておくことで、いろんな関数を取り替え可能にしておく事ができる。この様にして関数的なプログラミング言語は「取り替えるのが簡単」っていう利点を作り出している。

### 手続きではなく定義を書く
これは最近自分でしっくりきた感覚。感覚なので、なかなか伝えるのが難しいんだけど、トライしてみます。「手続き」っていうのは、「アレとコレを比べる。比べた結果、アレの方が大きかった場合、foo処理をする。比べた結果、コレの方が大きかった場合、bar処理をする。」っていう機械に語りかけるような、そんな感じのものだと認識している。関数的なものの場合、「foo処理とは、アレとコレを比べてアレの方が大きい場合の処理。bar処理とはアレとコレを比べてコレの方が大きい場合の処理。」ってな感じ。。。だんだん怪しくなってきた。。。たぶんどっちでやっても同じような事をするプログラムは書けるんだろうと思う。

書いてたらわからんくなってきた。一時中断する。またなんかふといい説明方法が思いついたら書く！

どなたかわかりやすげな説明できる方、よろしくお願いします！ひろくんとか！



---
layout: post
title: 'Django Best Practice への道 #3'
---

DjangoのWebアプリを開発している際、リファクタ/テスト拡充のために集めた情報をまとめます。本記事は三部作の三つ目となります。

- [#1 Djangoプロジェクト/アプリケーション/設定ファイル構成](http://achiku.github.io/2014/04/01/road-to-django-best-practice.html)
- [#2 Djangoテスト戦術](http://achiku.github.io/2014/04/04/road-to-django-best-practice.html)
- [#2 補足編](http://achiku.github.io/2014/04/14/django-pytest-webtest.html)
- [#3 Django Model/View/From/Template戦術](http://achiku.github.io/2014/04/07/road-to-django-best-practice.html)


書くこと
--------

Djangoの基本的な構成要素である、models.py, views.py, urls.py, forms.pyに関して、最初から知ってればよかったと思った事を書いていきます。だいぶ文章にもっさり感と字が多い感がでてますが、自分の言葉で説明することで理解を深めるという目的もあるのでご容赦ください。

あと、是非意見が欲しいっす。Django使ってる人日本に沢山いるはずなのにこの類の情報ってあんまり無いイメージがあり、以下に書いてある事を考えている人たちとお話してみたいっす(当たり前過ぎてそんなもの書く価値ねぇよ、な場合はすみません)。


前提
====

- 既に本番リリースされてる
- Django 1.5で作られてる
- 中/小規模Webアプリケーション(テーブルサイズ10 - 20)
- 開発/運用1人(achiku), アドバイザー/レビューアー1人(moquada)
- バックエンド処理ロジックは比較的シンプル
- Celeryを使った非同期タスクとして動く処理がある
- JSはクリティカルな処理では使ってない(表示整形くらい)
- トラフィックは少ない
- インフラはAWS(VPC + ELB + EC2 + RDS)



urls.py
=======


役割
----

- URLとViewの紐付け

urls.pyは本当にこれだけを担えば問題なし。逆にこの役割以外の事をやらないように現在開発をしている。urls.pyにQuerySetが書いてあったり、ロジックが書いてあったりするのは見通しが悪く、好みではない。楽に書けるからいいじゃなない！という意見もあるが、書く時楽になる(≒スピードアップ)と、その後の見通しの悪さを比べたら、あまり割のいいトレードオフではないと思う。なぜなら、この部分にロジック書くのも、views.pyにロジック書くのも労力的に大差無いから大してスピードアップしない。

今後上記基本姿勢を変更した方が良いか否かは、その時のコンテクストに合わせて適切に判断したい。

*参考文献*

- [URL dispatcher](https://docs.djangoproject.com/en/dev/topics/http/urls/)
- [Django Design Philosophies: URL](https://docs.djangoproject.com/en/dev/misc/design-philosophies/#url-design)

urls.pyを書く際に一番悩むのは前からずっとURL設計の部分。"Cool URLs don't change."とはよく言ったもので、ここをしっかりやろうとするとそれなりに時間が必要であった。まだイマイチしっくり来ていない部分が多く、今はテンプレみたいなものが欲しい。こういう場合はこう、みたいな。

*参考文献*

[Cool URIs don't change](http://www.w3.org/Provider/Style/URI)


あとはAPPEND_SLASHという仕組みに注意することと、ちゃんと名前をつけてreverseで引けるようにするってことぐらいな気がします。

[APPEND_SLASH](https://docs.djangoproject.com/en/dev/ref/settings/#append-slash)


なぜAPPEND_SLASHなのか。

> Technically, foo.com/bar and foo.com/bar/ are two different URLs, and search-engine robots (and some Web traffic-analyzing tools) would treat them as separate pages. Django should make an effort to “normalize” URLs so that search-engine robots don’t get confused.  [Django Design Philosophies: URL](https://docs.djangoproject.com/en/dev/misc/design-philosophies/#url-design)




forms.py
========


役割
----

いくつかforms.pyを書いてみて、コイツの役割は大体以下の2点に集約されるように思う。
- HTMLのform用に「どの項目を、どのような色や形で、どの順序で」表示するかを司る
- ユーザ入力のバリデーションを行う(DBの制約で行える部分以外で)

以下formsの役割ではない(少なくともformsに役割として課さないほうがいいなと思う)事。
- models側で制約をかけてるのに、forms側でも同じ制約をかける(e.g. 文字数制限、等)
- データ登録/編集のロジック(複数モデルを扱いながら)を記述する 


担うべき役割
------------

まず大前提としてformsの中に書くforms.Formを継承しているカスタムクラスは大きく2つに分類できる。

1. modelと密接に結びついているもの(e.g. Blogのエントリー, コメント、本の著者名、等)
2. modelと密接に結びついていないもの(e.g. お問い合わせフォーム、オブジェクトのIDをhidden属性つけてサーバサイドにPOSTしたい時、等)

1つ目に関しては断固としてforms.ModelFormを使い、Metaクラス内のmodel属性で対象のmodelをバインドし、fields属性で表示する項目を指定、widgetsでスタイル用のclassを指定、という鉄板の流れで作るのがいいと思う。そして、models.py内で定義されるModelクラスのverbose_name, help_textをしっかり書く。こうすることで、「基礎的な制約と項目説明に関してはModel側で、複雑な制約と表示形式はForm側で」という役割分担が明確になる。一時期弊社内では、Model側で定義している制約を再度Form側で定義しており、Model側を修正したらForm側を修正し、そのFormが継承されているFormを修正し、という、まぁ、端的に言えば大変無駄な作業が発生していた。

「基礎的な制約はModel側で」という事にしているが、じゃあ「複雑な制約」ってなんなんだという基準は以下3つ。

- modelの制約(≒DBの制約)だけでは不十分な要件がある場合(e.g. 文字列を保存したいが、hoge_で始まる文字列は弾きたい場合、等)
- 1モデルの複数項目に渡って検証する必要が有る場合(e.g. 開始日は終了日の前でなければならない、等)
- 過去に登録されたレコードとの整合性を検証する必要が有る場合(e.g. 購入のリクエストで在庫数を確認する、等)

もちろんmodelにもカスタムバリデーターを登録することは可能だけど、現時点ではしていない。理由は、DjangoのmodelはどちらかというとRDBMSにおけるテーブルのPython表現っぽい、という風にとらえているから。modelって名前、ややこしいね。もし本当にpersistent storageレベルで必要なバリデーションがあるのであれば、それはpersistent storageレベルで設定されるべきな気がしている(Oracleのcheck制約とか、MySQLのENUM型とか)。それはPython(Django ORM)以外からデータが扱われる事があったとしても適切にデータの整合性を守ってくれることとなる。

上記をそれとなく守って現在は開発している。が、まだあまり納得はいってない。この部分の方針は随時更新していきたいです。


*参考文献*

- [Creating forms from models](https://docs.djangoproject.com/en/dev/topics/forms/modelforms/)
- [Django forms.Form vs forms.ModelForm](http://stackoverflow.com/questions/2303268/djangos-forms-form-vs-forms-modelform)
- [Django Advantage forms.Form vs forms.ModelForm](http://stackoverflow.com/questions/3828025/django-advantage-forms-form-vs-forms-modelform)

2つ目の「modelと密接に結びついていないもの」はそこまで出番が無いように思える。これに関してはforms.Formを利用して素直に必要な項目と、項目に対する制約を記述すればいいと思う。大して言うことは無い。

*参考文献*

[Working with forms](https://docs.djangoproject.com/en/dev/topics/forms/)



担わない方がいいと思う役割
--------------------------

**models側で制約をかけてるのに、forms側でも同じ制約をかける(e.g. 文字数制限、等)**

上記は「担うべき役割」でも説明したように、やらない方が得する事が比較的明確だと思っている。問題は以下。


**データ登録/編集のロジック(複数モデルを扱いながら)を記述する**

ココは賛否両論あり得ると思う。上記のような抽象的な話ではなく、forms.FormPreviewのdoneメソッドの話なのかもしれない(普通のModleFormやForm内でデータ変更を実装することはほぼ無いと思う)。上記でFormが担うべき役割をある程度明確にしたのに、いきなりここでデータの変更をされる、というすごい気持ち悪い状態になる。「データをどこで変更しているのか」を考える時に見なければいけない範囲が格段に広がるので、コードをたどるのが非常にダルくなる。が、Previewがどうしても必要なんだ！という場合もある。その場合は涙をのんでFormPreviewを使う(ちなみにデフォルトではFormPreviewではFileFieldはプレビューできない)。非常にダルい。ダルいのでそれをCBVにしようってのが以下のライブラリ。まだセキュリティー含めて検証中で実戦に投入はできていないが、CBVとしてPreview機能を実装しているので、Formの中にデータ変更処理が入る事なく、いい感じで書けそうな気がしている。

- [django-cbv-formpreview](https://github.com/ryankask/django-cbv-formpreview)


FileFieldもプレビューできるようにする実装のサンプル。これはそのまま実戦投入はできそうだけど、実装のコンセプトとして参考になる。

- [Django File FormPreview](https://github.com/razum2um/django-file-formpreview)


DjangoのFormについては泣かされた部分も多いのでだいぶ長くなってしまいましたが、現状こんな感じです。


Form関連のオススメライブラリ
----------------------------

- [django-crispy-forms](https://github.com/maraujop/django-crispy-forms)


Form関連のオススメしないライブラリ
----------------------------------

- [django-bootstrap-toolkit](https://github.com/maraujop/django-crispy-forms)

パフォーマンス問題が発生するケースがある。おそらくfieldに直接bootstrap filterを適用する場合。弊社ではコレが発生したので利用を取りやめました。ライブラリの利用は必ずOpenなIssueを確認し、作者に改善に意志があるかを見てからにしようと心に誓った出来事です。
[Performance issue : to many get_template](https://github.com/dyve/django-bootstrap-toolkit/issues/103)


models.py
=========


役割
----


ここもまだまだしっくりきてる、とは言えない部分が多いのですが、以下のような感じだと思ってる。

- DjangoにおけるActiveRecordパターンを担う

デザインパターン便利。名前言うだけでなんか賢く見えるし。でも、これだとActiveRecordパターンを正確に理解できてないといけないし(自分はそこまで理解してない)、理解するまで何か書けないというのは辛いので以下、自分の言葉で表すとこうなります、というのをやってみる。

- RDBMSにおけるテーブルのPython表現

*参考*

[Active Record by Martin Fowler](http://www.martinfowler.com/eaaCatalog/activeRecord.html)


担うべき役割
------------

- 1モデルが表現する1テーブルの各属性について責任を持って更新管理する
- 1モデルが表現する1テーブルへの問い合わせに責任を持つ
- 1モデルが表現する1テーブル内にある属性から導出される、そのモデルの属性について責任を持つ


順に解説していきます。


**1モデルが表現する1テーブルの各属性について責任を持って更新管理する**

これはDjango ORMが勝手にってくれることなので特に言う事ないです。


**1モデルが表現する1テーブルへの問い合わせに責任を持つ**

これもDjango ORMが勝手にってくれることなので特に言う事ないです。


**1モデルが表現する1テーブル内にある属性から導出される、そのモデルの属性について責任を持つ**

大事だなと思ったのはこの3つ目の役割と、3つ以外の役割をmodelに任せないこと、だと思うのでちょっと解説。まずはこの3つ目の役割の具体例として、例えば、何かの記事を書く機能があったとして、その記事は著者と承認者が両方チェック完了したらオンライン上に公開可能になる、という要件を仮定します。その要件からざっくり以下のmodelを作成(verbose_name, help_text, Metaクラスは省略してます)。このmodelの属性である、author_checkとapprover_checkがTrueになれば公開可能な状態という意味とする。

{% highlight python %}

class Article(models.Model):
    body = TextField()
    author = CharField(max_lenght=100)
    author_check = BooleanField()
    approver_check = BooleanField()
    published = BooleanField()
    date_created = DateTimeField()
    date_published = DateTimeField(null=True, blank=True)

{% endhighlight %}

このモデルには著者と承認者がオッケーを出した事を表現する属性はない。あってもいいけど、要件が上のものだけなのであれば不要なのでシンプルに無い方を採用。「公開可能か否か」という属性はapprover_checkとauthor_checkから導出できる属性となる。それをDjango ORMでどのように表現するかというと、以下。

{% highlight python %}

class Article(models.Model):
    body = TextField()
    author = CharField(max_lenght=100)
    author_check = BooleanField()
    approver_check = BooleanField()
    published = BooleanField()
    date_created = DateTimeField()
    date_published = DateTimeField(null=True, blank=True)

    @property
    def is_ready_to_publish(self):
        return self.author_check and self.approver_check

{% endhighlight %}

実際にis_ready_to_publishがTrueならば公開する(≒published属性をTrueに設定、date_publishedに現在の時間を登録)、という処理も、このmodelの属性に関わる事なので、modelのメソッドに実装する。


{% highlight python %}

class Article(models.Model):
    body = TextField()
    author = CharField(max_lenght=100)
    author_check = BooleanField()
    approver_check = BooleanField()
    published = BooleanField()
    date_created = DateTimeField()
    date_published = DateTimeField(null=True, blank=True)

    @property
    def is_ready_to_publish(self):
        return self.author_check and self.approver_check

    def publish(self):
        if self.published:
            raise AlreadyPublishedException()
        elif self.is_ready_to_publish:
            self.published = True
            self.date_published = datetime.datetime.now()
        else:
            raise NotPublishableException()

{% endhighlight %}

色々いじってみて、Django ORMにおいては1モデル1テーブル、1インスタンス1レコードという概念を念頭に置いた方がいい気がしている。各レコード毎の属性はmodelに、各レコード毎に既存レコードから導出できる属性はmodelに、各レコード毎の属性の値によって処理が変わり、且つその処理がタッチするのが当該モデルに閉じる場合はmodelのメソッドに、といった感じ。

以下の記事は複数モデルが絡み合うロジックを完全に無視して書いているので、あと一歩な感じなのですが、記事へのコメントがかなりいい感じに補完してくれているので全体として大変参考になる。

[Fat Models - A Django Code Organization Strategy](http://redbeacon.github.io/2014/01/28/Fat-Models-a-Django-Code-Organization-Strategy/)



担わない方がいいと思う役割
--------------------------

- 複数モデルが絡み合うロジック
- 処理が対象のモデルの中核をなすものでは無いロジック
- AWS等の外部サービスとやりとりするロジック

これはmodelのメソッド内で完結するのは難しいし、そもそもmodel全然関係無い場合もある。先にも述べたようにDjango ORM自体、1モデル1テーブルを基準にして作られているから。なので上記は別の部分に切り出して書くのがいい気がしている。やはりDjangoのmodelはRDBMSにおけるテーブルのPython表現、くらいのプリミティブなものだと捉えておいた方が何かといいのではないかと思う。

以下の記事はRuby on Railsに関する記事だけど、考え方が非常に参考になった。(ちなみにRuby on Railsは全く触ったこと無い。っていうかDjango以外にWAFを真面目に触ったことが無いという雑魚っぷりです)

- [7 Patterns to Refactor Fat ActiveRecord Models](http://blog.codeclimate.com/blog/2012/10/17/7-ways-to-decompose-fat-activerecord-models/)
- [肥大化したActiveRecordモデルをリファクタリングする7つの方法(翻訳)](http://techracho.bpsinc.jp/hachi8833/2013_11_19/14738)


views.py
========


役割
----

- 何かリクエストを受けて、リクエストに対して何か処理をして、何かレスポンスを返す

雑に書いているわけではなく、このレイヤーは抽象的な処理に徹するべきだと思う。そして、可能な限り見通しよく、薄く作るのが、最終的にメンテナンスしやすい気がする。

「何か処理」の部分は1インスタンスに閉じる処理であればmodelのメソッドを使うし、バリデーションに関係する部分はformを使う。複数modelが絡み合う処理が必要な場合はmodelではなくutils.pyやservices.pyに処理を委譲。最終的に何をレスポンスとして返すか、という部分だけハンドリングする、といった感じ。

CBVやFBVをどこでどうやって使うべきか、という話は、[Two Scoops of Django](http://pydanny.com/announcing-two-scoops-of-django-1.6.html)に非常に詳しく書いてあり、とても参考になった。この本は本当に素晴らしい。



Django全体通して
================

最後に、Django全体を通した思想っぽいものを紹介して、Djangoに対する感想など述べながら締めます。

[MTV Framework?](https://docs.djangoproject.com/en/dev/faq/general/#django-appears-to-be-a-mvc-framework-but-you-call-the-controller-the-view-and-the-view-the-template-how-come-you-don-t-use-the-standard-names)

- **M** Django modelの事。「データ」の部分を担当(?)。本文にあまり記述は無い。
- **V** Django Viewの事。「どのデータを」の部分を担当。
- **T** Django templateの事。「どのように表示するか」の部分を担当。



    Well, the standard names are debatable.

    In our interpretation of MVC, the “view” describes the data that gets presented to the user. It’s not necessarily how the data looks, but which data is presented. The view describes which data you see, not how you see it. It’s a subtle distinction.

    So, in our case, a “view” is the Python callback function for a particular URL, because that callback function describes which data is presented.

    Furthermore, it’s sensible to separate content from presentation – which is where templates come in. In Django, a “view” describes which data is presented, but a view normally delegates to a template, which describes how the data is presented.

    Where does the “controller” fit in, then? In Django’s case, it’s probably the framework itself: the machinery that sends a request to the appropriate view, according to the Django URL configuration.

    If you’re hungry for acronyms, you might say that Django is a “MTV” framework – that is, “model”, “template”, and “view.” That breakdown makes much more sense.

    At the end of the day, of course, it comes down to getting stuff done. And, regardless of how things are named, Django gets stuff done in a way that’s most logical to us.


という、DjangoはMTVなフレームワークだ！という主張。

この文章、あまりmodelには触れてない。modelのMはMVCで言うところのモデル、つまりはビジネスロジックなのか、Django ORMによって定義されるmodelなのかはこの文章からだけではよくわからない。ただしばらく触ってみた感じだと、MTVのMがMVCのMと同じだとするならば、それはmodelインスタンス単体ではなく、存在するmodelインスタンスを組み合わせながら処理を行うどっか別の部分に定義される処理な気がする。デフォルトでは入ってないけどservices.pyとか作ってそこに処理を定義、みたいな。もしMTVのMがMVCのMではなく、ただのデータを永続化するだけの仕組み+ちょっとしたメソッドだとしたら、それはDjango ORM単体で事足りる。

Djangoはもともとニュース系のサイトを管理する目的で開発されたらしい。その出自が影響しているのかどうか不明だけど、確かにニュースサイトで複数インスタンスが複雑に絡み合い、外部サービスと連携しながら特定の処理を行う事ってあんまり想像できない。Djangoそのままのmodels.py/views.py/urls.py/forms.pyだけを使って作る事を想定しているのは、そういった、データ+ちょっとした処理くらい、に最適化されてるんじゃないかなぁという印象を持った。

もちろん工夫しだいで複雑にもできる作りになってる。


まとめ
======

ここまで書いてなんですが、書いてきたことが本当にBest Practiceへ通じているのか、正直全くわからない。こちとらエンジニア4ヶ月目で、Djangoに関して真面目に取り組んで3ヶ月目で、だいたい毎日死にはぐってます。わからん部分はmoqada氏と話しながら発展させてきた考えをまとめてみたのですが、いざ文章にしてみるとイマイチな感じがする部分等でてきております(ちなみにmoqadaさんはかわいい奥さんとイタリアに新婚旅行中で幸せ満喫中らしい。羨ましい。)

このまとめを書こうかなと思ったのは、大きなリリースが一段落ついた、という事もあるのですが、ずっと心に残っていた文章に対する自分なりの一つの答えでもあります。

[西尾さんの2011新卒準備カレンダーの記事](http://d.hatena.ne.jp/nishiohirokazu/20110309/1299598527)


今回書いたDjango Best Practiceへの道の三本の記事には沢山のリンクが貼ってあります。これらのリンク、全てから自分は勉強させてもらっており、自分が学んでいるということはカンムという会社がクオリティー高いプロダクトをスピーディーに市場に届ける事に貢献しており、カンムが本当に市場に必要とされるプロダクトを作っている限り、世の中の便利さを向上させている事になります。そんな教材が、インターネットにアクセスするだけで死ぬほど手に入るという事に、今更ながら驚愕してます。

そういう記事の末席に、このまとめも加われれば幸い。加わらなくても書くだけで自分の脳みそ整理できたので元は取れてる。

勉強させてもらった記事を書いてくれた方々、ライブラリを公開してくれている方々に感謝。また、この記事を書くために時間ある程度使ってもいいよと言ってくれたシャチョーに感謝です。


わからんことだらけでヤバイので、異論大歓迎です！尖すぎるマサカリは怖いですが、良いマサカリであれば泣きながら受け止めます！


(追伸：Templateまで届きませんでした。)


- [#1 Djangoプロジェクト/アプリケーション/設定ファイル構成](http://achiku.github.io/2014/04/01/road-to-django-best-practice.html)
- [#2 Djangoテスト戦術](http://achiku.github.io/2014/04/04/road-to-django-best-practice.html)
- [#2 補足編](http://achiku.github.io/2014/04/14/django-pytest-webtest.html)
- [#3 Django Model/View/From/Template戦術](http://achiku.github.io/2014/04/07/road-to-django-best-practice.html)






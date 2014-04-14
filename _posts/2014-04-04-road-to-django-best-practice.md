---
layout: post
title: 'Django Best Practice への道 #2'
---

DjangoのWebアプリを開発している際、リファクタ/テスト拡充のために集めた情報をまとめます。本記事は三部作の二つ目となります。

- [#1 Djangoプロジェクト/アプリケーション/設定ファイル構成](http://achiku.github.io/2014/04/01/road-to-django-best-practice.html)
- [#2 Djangoテスト戦術](http://achiku.github.io/2014/04/04/road-to-django-best-practice.html)
- [#2 補足編](http://achiku.github.io/2014/04/14/django-pytest-webtest.html)
- [#3 Django Model/View/From/Template戦術](http://achiku.github.io/2014/04/07/road-to-django-best-practice.html)



書くこと
--------

Django Best Practiceへの道の続きで、Djangoテスト戦術について書きます。Djangoでテストをする際に、どうしたら効率的に書けるか、メンテナンスしやすくなるか、ということに焦点を置いて書きます。


書かないこと
------------

テストをするべき、テストはいらない、どこまではするべき、といった類の話は書きません。する、しない、いまはしない、どこまではする、は各チームや開発者がその時置かれているコンテクストに非常に強く依存している為、閾値的なものや考え方を書くのは非常に難儀だなぁ、というのが素直なところです。それよりもテストするのが少しでも楽になり、どのようなコンテクストでも、取れる選択肢の幅が広がる方法を書きたいです。


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


何をテストすべきか
==================

作っているのはDjango前提のWebサービスという前提で、テストすべきものをいくつかのグループに分けて考えてみました。テスト対象を明確にすることで、「どこにテストを書けばいいのか」、「何をテストすればいいのか」と迷わずにテストに着手できるようになるかなと思います。テスト対象の構成が上の方ほど単純で下に行くほど複雑になります(下に行くほどテストするまでにリクエスト/レスポンスが通るレイヤーが多い)


関数レベル(Python)
------------------

可能な限り少ないブランチの関数レベルテスト。テストするのは各クラスのメソッドor関数内分岐レベル。最小粒度のテスト。

**テスト用ファイル名**

test_models.py, test_forms.py, etc


関数レベル(JS)
--------------

JSの関数レベルテスト。自分が担当している範囲ではココをテストしなければならない部分は存在しないため、特に対応はしていないのであまり書ける事がない。moquada氏担当の部分はココがかなり肝になっているので、テストリファクタが終わったら何かまとめておいて貰う予定です。BusterJS、JsTestDriver利用か。

**テスト用ファイル名**

未定


URLルーティングレベル
---------------------

URLのルーティング、ステータスコード、HTTPレスポンス内の文字列をテスト。正直2軍感はあるけど一応書いとくか、くらい。業界ではsmoke testと呼ばれることもあるらしい。テスト書く時間がない場合に、一応動いてる事だけ保証する時に使ったりする。Djangoについてる権限モジュール(django.contrib.auth.decorators, django.contrib.admin.views.decorators)をそのまま使うと権限ない場合に403が返らずに、Adminのログイン画面にリダイレクトされるので注意が必要。

**テスト用ファイル名**

test_urls.py


機能レベル
----------

複数の関数を集めて、URLでルーティングさせた先が機能、と考えてる。例えば、「ブログ記事編集」、「コメンツ追加」、みたいな粒度のもの。各機能別テストデータセット別のテストを書く。

**テスト用ファイル名**

test_[app_name].py


ブラウザレベル
--------------

JS経由でリクエストされる際の機能テスト。seleniumを利用して実際にブラウザからテストする。現在自分が担当している部分ではほぼ存在しない。moquada氏担当の部分はココがかなり肝になっているので、テストリファクタが終わったら何かまとめておいて貰う予定です(大事な事なので2度言いました)。

**テスト用ファイル名**

test_browser.py


テストの種類の分類や定義や呼び方は色々ありすぎてよくわからない部分が正直多いです。unit testとソレ以外だ！っていう人もいるし、integration testとacceptance testは分かれてるべきだ！という人もいるし。なので上で分けた分類も、そういうのもあるよね、程度で。今カンムで利用しているDjangoアプリの特性的にはしっくり来ています。


*参考文献*

何をテストすべきか、その際に何を使うべきか、使う時にどういう方針で使うのがいいか、がハイパーまとまってる資料。この資料に一番影響されてると思う。

- [YouTube: Testing and Django](http://pyvideo.org/video/699/testing-and-django)
- [Testing and Django](http://carljm.github.io/django-testing-slides/#1)
- [Testing and Django Note](https://pycon-2012-notes.readthedocs.org/en/latest/testing_and_django.html)

抜粋

    What type of test to write?

    Write system tests for your views.
    Write Selenium tests for Ajax, other JS/server interactions.
    Write unit tests for everything else (not strict).
    Test each case (code branch) where it occurs.
    One assert/action per test case method.


下記のStackOverflowの記事も何を、どうテストしていくのかに関して実例交えながら語ってる。

- [What are the best practices for testing “different layers” in Django?](http://stackoverflow.com/questions/11543117/what-are-the-best-practices-for-testing-different-layers-in-django)

以下の資料はユニットテストを書く利点と注意点がわかりやすく整理されている。
- [YouTube: Faste test, slow test](http://pyvideo.org/video/631/fast-test-slow-test)
- [Faste test, slow test Note](https://pycon-2012-notes.readthedocs.org/en/latest/fast_tests_slow_tests.html)


どうテストするのか
==================

「何を」テストするのか、が明確にした後、次は「どうやって」テストするのかの話をします。その為にまず、テストに求めるものを洗い出し、テストに利用するツールとテストの書き方が、それらの要件を満たすように設計していきました。

一旦今Djangoアプリのツールとして何を利用しているか列挙。

全レベル共通
------------

- py.test
- pytest-cov
- pytest-pep8
- pytest-xdist(検証中)
- pytest-django
- factory_boy
- PyHamcrest


機能レベル
----------

- WebTest
- django-webtest


ブラウザレベル
--------------

- selenium
- django-selenium


テストに求めるもの
==================

- テスト実行スピードが速い
- 同一処理別データパターンのテストを効率良く書ける
- プログラム本体のコードが変更された際にも追従しやすい
- debugしやすい
- テストカバレッジが見れる


順に解説していきます。


テスト実行スピードが速い
------------------------

細かくテストを実行する習慣をつけるには、テスト開始から終了までの時間が短ければ短い程いいです。1回ファイル編集して全てのテスト通すまでに数十分とかかかるのはやめたい。それだと誰もテスト実行しなくなってしまう。一応弊社ではGitHubにプッシュした際にCIサーバで全テストを流すようにしているので大きな問題にはならないはずですが、今後テストが増えてきた時のためにも、ローカルで全テストを流してもなるべく早く終わるようにしておきたい。


**設定的工夫**

ローカルマシンでテスト実行する場合はローカルテスト専用設定を使うようにしています。
[詳細はコチラ](http://achiku.github.io/2014/04/01/road-to-django-best-practice.html)

- ローカルテスト用の設定はsqliteのインメモリDBを利用する。
- fixture(djangoのコンテクストでの)を撲滅してセットアップのスピードを上げる。
- southのmigrationテストをオフる。
- PASSWORD_HASHERSの変更。
  
*参考文献*

- [Django で unittest を高速化する(主にDBの話し)](http://tell-k.hatenablog.com/entry/2013/10/10/202208)
- [Speeding up the tests](https://docs.djangoproject.com/en/1.5/topics/testing/overview/#speeding-up-the-tests)
- [Djangoでテストを速くするためにいろいろやってみた](http://d.hatena.ne.jp/yuheiomori0718/20130615/1371305730)
- [Testing and Django](http://carljm.github.io/django-testing-slides/#1)


**コード的工夫**

- 兎に角I/Oを避ける。DBに触れないcustom filters/forms/utils等のテストはDBを作らない。
  * django.test.TestCaseではなく、unittest.TestCaseを利用
  * pytest-djangoを利用する場合は、DBを利用する場合にのみ@pytest.mark.django_dbデコレータを利用
- WAFのレイヤーを可能な限りまたがない。
  * レイヤー(view/forms/models)をまたぐ度に処理が走るしセットアップも時間がかかる。
  * ロジックはDjangoのモジュールをかませないようにして、単独でテストできるようにしておく。

*参考文献*

- [Testing Django applications with py.test (EuroPython 2013)](https://speakerdeck.com/pelme/testing-django-applications-with-py-dot-test-europython-2013)
- [Testing and Django](http://carljm.github.io/django-testing-slides/#1)



同一処理別データパターンのテストを効率良く書ける
------------------------------------------------

以下箇条書きでなぜコレを求めているのかを列挙します。

- データだけ違って処理は同じなら同じ関数でテストしたい(同じもの書きたく無い)
- けどテストケースとしては分けたい
- 便利なassertは使いたい

結構データだけ変えて同じ処理を流したいというケースって多い気がします。例えばバッチのコミット件数の閾値や、UIからの入力項目、URLのルーティングで200が返ってくることを確認したいだけのテスト等。そんな時にはコピペで対応、という事もできますし、コンテクストによってはそれも許容する場合はあると思いますが、正直めんどくさくて嫌です。


**工夫**

- py.testのparametrizeアノテーションを利用

同一処理別データパターンのテストを効率良く書く為、データ別にテストケースを生成するのがデフォルトの機能として備わっているpy.testを利用することとしました。他にも幾つか選択肢がありましたが、py.testの他の機能(fixtureやxdist)も利用したかったので、もともとunittest形式だったテストを全てpy.test形式に書き換えました。py.testはunittest形式のテストもpy.test形式が混ざった状態でテスト実行できるので順次移行できて助かりました。

*参考文献*

以下StackOverflowから見つけたparametrized testを簡単にしてくれるライブラリ一覧。

    Some of the tools available for doing parametrized tests in Python are:

    Nose test generators (only for function tests, not TestCase classes)
    nose-parametrized by by David Wolever (also for TestCase classes)
    Unittest template by Boris Feld
    Parametrized tests in py.test
    parametrized-testcase by Austin Bingham


- [Python unittest: Generate multiple tests programmatically? ](http://stackoverflow.com/questions/2798956/python-unittest-generate-multiple-tests-programmatically)

unittest形式でテストを生成する際に非常に参考になった記事。
- [TestCaseを拡張しよう](http://d.hatena.ne.jp/nullpobug/20091204/1259863417)
- [Unittest template](http://feldboris.alwaysdata.net/blog/unittest-template.html)

py.testに惚れるきっかけとなった記事
- [Pytest本家](http://pytest.org/latest-ja/)
- [pytest fixtures nuts and bolts](http://pythontesting.net/framework/pytest/pytest-fixtures-nuts-bolts/)

Two Scoops of Djangoの人の記事。本の中ではテストはコピペでもオッケー！という発言があったし正直そうだとも思うけど、それを回避する策に力注ぐほうが楽しいと思う。

- [pytest: no-boilerplate testing](http://pydanny.com/pytest-no-boilerplate-testing.html)

もうunittestには戻れない。

- [pytestを実戦投入してみた](http://qiita.com/roothybrid7/items/9717137fbec2bedfd81d)


*サンプル*

例えばURLのルーティングとちゃんとステータスコード200が返ってきて、所定の文字列がレスポンスに含まれている、という事をテストしたい場合、WebTestと組み合わせる事で以下のように書くことができます。


{% highlight python %}
# -*- coding: utf-8 -*-
import pytest
from hamcrest import (
    assert_that,
    contains_string,
    equal_to,
)


@pytest.mark.django_db
@pytest.mark.parametrize('login_user, url, message, status_code', [
    ('admin', '/coupons/?status=0', 'クーポン一覧', 200),
    ('admin', '/coupons/?status=1', 'クーポン一覧', 200),
    ('admin', '/coupons/?status=2', 'クーポン一覧', 200),
    ('admin', '/coupons/?status=3', 'クーポン一覧', 200),
    ('admin', '/coupons/1', 'クーポン詳細', 200),
    ('admin', '/coupon/add', 'クーポンの登録', 200),
    ('user', '/coupons/?status=0', 'クーポン一覧', 200),
    ('user', '/coupons/?status=1', 'クーポン一覧', 200),
    ('user', '/coupons/?status=2', 'クーポン一覧', 200),
    ('user', '/coupons/?status=3', 'クーポン一覧', 200),
    ('user', '/coupons/1', 'クーポン詳細', 200),
    ('user', '/coupon/add', 'クーポンの登録', 200),
])
def test_coupon_urls(app, data, login_user, url, message, status_code):
    resp = app.get(url, user=data[login_user])
    assert_that(resp.status_code, equal_to(status_code))
    assert_that(resp.content, contains_string(message))
{% endhighlight %}


シンプル！

ちゃんとpy.testのfixtureやその他の機能の詳細についても書きたい。実装レベルでの工夫等も後ほど書く。

[補足編書きました](http://achiku.github.io/2014/04/14/django-pytest-webtest.html)


プログラム本体のコードが変更された際にも追従しやすい
----------------------------------------------------

- データセットアップの際にfixtureではなくfactory_boyを利用

Djangoのfixture遅いしメンテナンスめんどくさいのでfactory_boyを利用して各テストケースに必要な分だけレコードを作成してテスト実行しています。この部分は正直まだまだ工夫の余地があるなぁという印象。もう少し時間がたったら書いてきたテストを見なおしてリファクタして行く際にまた工夫が生まれればいいなと思ってます。


debugしやすい
-------------

- py.testを利用

py.testはnoseよりも細かくエラーを出してくれます。pytest.vimを使って何度でも素早くテストできるようにしてます。テストを先に書く事のメリットひしひしと感じ始める今日このごろ。特に関数レベルのテスト。


*参考文献*

- [pythonのテストにpytestを使ってみた](http://blog.restartr.com/2013/04/05/my-first-pytest/)
- [TDD Advent Calendar 2013 3日目: vim と py.test で TDD](http://blog.comutt.jp/entry/2013/12/03/230000)


テストカバレッジが見れる
------------------------

- py.testのカバレッジプラグインを利用

カバレッジ至上主義ではありませんが、あくまでも目安として、またちょっとしたゲーム感覚でテストを楽しめるようないい工夫だと思います。また、カバレッジレポートを見ることでまだ通っていないブランチが視覚的に分かるのも嬉しいところです。

- [py.test で pep8 と coverage を同時にチェックする](http://www.sakito.com/2012/09/pytest-pep8-coverage.html)



まとめ
======

今回はだいぶ概念的な話がおおくなってしまいました。。本当はもっとゴリッとしたテストセットアップ方法の工夫や、WebTestとpy.testの連携や、conftest.pyの配置と役割、とかを書きたかったのですが、今回は一旦ココまでとします。

次回はDjangoテスト戦術の第二弾でもっと実装よりの話をします。てかなげーな。このポスト。。

- [#1 Djangoプロジェクト/アプリケーション/設定ファイル構成](http://achiku.github.io/2014/04/01/road-to-django-best-practice.html)
- [#2 Djangoテスト戦術](http://achiku.github.io/2014/04/04/road-to-django-best-practice.html)
- [#2 補足編](http://achiku.github.io/2014/04/14/django-pytest-webtest.html)
- [#3 Django Model/View/From/Template戦術](http://achiku.github.io/2014/04/07/road-to-django-best-practice.html)



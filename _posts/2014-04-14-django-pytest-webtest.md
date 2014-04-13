---
layout: post
title: 'Django + py.test + WebTest'
---

以前書いた[Django Best Practice への道 #2](http://achiku.github.io/2014/04/04/road-to-django-best-practice.html)の補足を書きます。


以下の課題を解決するために実施したこととなります。

**Django Webアプリの機能テストはpy.testとWebTestを同時に使いたいけどunittest形式は嫌**


分解していきます。


Djangoのテストにpy.testを使う
-----------------------------

以下のライブラリを利用する。
(なぜpy.testを選択したかは[コチラ](http://achiku.github.io/2014/04/04/road-to-django-best-practice.html))

- [pelme/pytest_django](https://github.com/pelme/pytest_django)

とにかくparameterizedテストで同一処理 x 別データパターンのテストを効率よく書きたかったので、py.testを選択。また、fixtureという仕組みを使い、テストデータやモックを効率よく各テストに注入できるのも嬉しいところ。


DjangoのテストにWebTestを使う
-----------------------------

以下のライブラリを利用する。

- [kmike/django-webtest](https://github.com/kmike/django-webtest)

これを利用する目的は前回のポストで書いた「機能テスト」で利用するためです。機能テストはレンダリングされたHTMLも含む形でテストをしたい。一般的に利用されるDjangoのテストクライアントは、どのテンプレートが使われたか、テンプレートに渡るcontext variablesが正しいか、をチェックするものなので、カンムで想定している機能テストに関してはそもそも出番じゃない。よって、機能テストを実施するにはWebTestを利用することとしています。

*参考文献*

- [Prefer WebTest to Djangos test client for functional tests](http://codeinthehole.com/writing/prefer-webtest-to-djangos-test-client-for-functional-tests/)


unittest形式が嫌
----------------

そんなに毛嫌いしているわけではないのですが、py.testの機能をフルに使おうと思うと、unittest.TestCaseを継承しているテストクラスでは色々具合がよくない(parameterizeアノテーションが使えなかったり、fixture使えなかったり)。

そこでWebTestも含めて全てpy.test形式でつくろうと思ったのですが、django_webtestパッケージ内のWebTestは既にdjango.test.TestCaseを継承しており、unittest.TestCaseを継承しているので、そのままではpy.test形式で使えない。

[django-webtest/django_webtest/\_\_init\_\_.py](https://github.com/kmike/django-webtest/blob/master/django_webtest/__init__.py)

ちょっと悩んで諦めようかなとも思ったのですが、WebTestを使うために必要な部分はなんとdjango_webtest.WebTestMixinというクラスに切りだされ、django_webtest.WebTestは、django_webtest.WebTextMixinとdjango.test.TestCaseをmixinしたものだという事に気づきました。

ということは、unittestを脱してDjangoでWebTestを簡単に使うには、このdjango_webtest.WebTestMixinだけ切り出して使えばいけるのでは、、という事でconftest.pyに以下のように設定してみたらすんなり使えました。


{% highlight python %}
# -*- coding: utf-8 -*-
from django_webtest import WebTestMixin


def app():
    wt = WebTestMixin()
    wt._patch_settings()
    wt.renew_app()
    return wt.app
{% endhighlight %}

前回掲載したURLに対するGETのスモークテストのサンプルでいくと、


*apps/appA/tests/conftest.py*

{% highlight python %}
# -*- coding: utf-8 -*-
from django_webtest import WebTestMixin
from tests.factories import (
    AdminUserFactory, NormalUserFactory,
)


def app():
    wt = WebTestMixin()
    wt._patch_settings()
    wt.renew_app()
    return wt.app

def users():
    admin = AdminUserFactory.create()
    user = NormalUserFactory.create()
    return {
        'admin': admin,
        'user': user
    }
{% endhighlight %}


*apps/appA/tests/test_url.py*

{% highlight python %}
# -*- coding: utf-8 -*-
import pytest
from hamcrest import (
    assert_that, contains_string, equal_to,
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
def test_coupon_urls(app, users, login_user, url, message, status_code):
    resp = app.get(url, user=users[login_user])
    assert_that(resp.status_code, equal_to(status_code))
    assert_that(resp.content, contains_string(message))
{% endhighlight %}

シンプル！

これを発見した時はライブラリのソースコードに潜り込んで何かを自分の思い通りに動かせた！しかもまだ誰も見つけてない！と非常にうれしかったのですが、より洗練されたやり方が以下に紹介されていました(フランス語読めないけどコードは読める)。世の中甘くないっす。

- [PyTest et WebTest](http://mathieu.agopian.info/presentations/2013_09_djangocong/)


おまけ
------

django.test.TestCaseを継承しない、という選択をしたのですが、この選択の副作用として、Djangoにデフォルトで備わっている便利なassertionが使えなくなってしまう、という点があります。

弊社ではPyHamcrestを利用することでなんとなく回避していますが、以下にDjangoデフォルトのassertionをpy.testの名前空間にぶっこむ事で回避するという実装があり、これはこれですごい便利だなぁと思ってます。

- [Funcargs & other fun with pytest](http://www.slideshare.net/pfctdayelise/funcargs-other-fun-with-pytest)
P12に記載あり


まとめ
------

py.testとWebTestはDjangoとも組み合わせて使えますし、なんならテストを書くのが少し楽になる気がしています。




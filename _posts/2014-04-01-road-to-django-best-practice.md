---
layout: post
title: 'Django Best Practiceへの道 #1'
---

DjangoのWebアプリを開発している際、リファクタ/テスト拡充のために集めた情報をまとめます。

ベストプラクティスとか言っちゃってますが、まだまだ良くできるはずだし、現在前提としているものが変わればまったく別のアプローチが最適、ということもあり得るので、「ベストプラクティスへの道」って事で。エンジニア、と名乗るようになってまだ4ヶ月程なので、わからんことばかりです。是非皆様のご意見聞いてみたいです。大体以下の感じで書いていきます。


- Djangoプロジェクト/アプリケーション/設定ファイル構成
- Djangoテスト戦術
- Django Model/View/From/Template戦術


戦略よりも、自分が入社した時既にあった前提に対応する為に考えた戦術を中心に書いていきます。また、自分の思考をダンプして記録しておくという目的もあるので、記述が冗長な部分もありますがご容赦ください。


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


プロジェクト/アプリケーション構成
=================================


アプリケーション構成に何を求めるか
----------------------------------

- 巨大な1ファイルのコードが存在しえない
- コードの見通しが自然とよくなる
- 各モジュールの依存関係が少ない


Djangoにはプロジェクトとアプリケーションという思想がある。Djangoが生まれた時からある概念で、1つのプロジェクトの中に、複数のアプリケーションを入れ、各アプリケーションの依存は可能な限り少なくし、取替え可能な形にする事を目指している。では実戦レベルでどのように分割するのが良いのか、という指針を実例を交えながら語ってくれているのが以下の資料。2008年の資料だけど、Djangoの根本的な思想は変わっていないので未だ有効だと思う。あと、"Do one thing, and one thing well"って本当にすごいかっこいいし、この思想を実装しているGNUやUNIX的なものはさらにかっこいいと思う。

*DjangoCon 2008: Reusable Apps*
- [YouTube: DjangoCon 2008: Reusable Apps](http://www.youtube.com/watch?v=A-S0tqpPga4)
- [Developing reusable apps](http://media.b-list.org/presentations/2008/pycon/reusable_apps.pdf)

上の資料を参考に設計する利点は主に2つ。

**メンテナンスしやすくなる**

このアプリケーション分割をすることで、「巨大な1ファイルのコード」が存在し得ない構成となる。誰かが意識してメンテしにくい巨大な1ファイルを作らないようにする、ではなく、そもそもそんなものが発生し得ない構成にすることが肝要だと思った。

**各モジュールの再利用可能性向上**

普通の単独自社サービスであれば、そこまで再利用性を高める為に時間をかける必要はない気がするけど、Djangoを使った自社サービスが複数ある中でのライブラリ作成や、個別のお客さんによってカスタマイズが必要なパッケージ、というシチュエーションであればある程度時間をかけて設計する価値はあると思う。実際上の資料内でJamesさんが自社パッケージを各お客さん用にカスタマイズする際に、プラガブルに設計してて助かったぜ、という話がある。


プロジェクト構成に何を求めるか
------------------------------

- 共通な部分と固有な部分を直感的に分けたい
- 何かを作る時にどのディレクトリに入れるべきか可能な限り迷いたくない


実装
----

他の流派もあるのですが、一旦現在は以下のような形に落ち着いています。まだまだ継続改善中。

```
    project
    ├── apps
    │   ├── appA
    │   ├── appB
    │   └── appC
    ├── core
    │   └── settings
    ├── docs
    │   ├── files
    │   └── styleguide
    ├── fixtures
    ├── libs
    │   └── management
    │       └── commands
    ├── requirements
    ├── server-config
    ├── static
    ├── templates
    └── tests
```


上から順に概要を説明します。


- apps
  - Djangoアプリケーションを格納。
  - アプリケーションを何単位で作るのかは先述の資料を参考に要議論。
  - apps内のディレクトリ構成は、'''tests''', '''migrations''', '''templates'''が基本。(South利用前提)
  - "Do one thing, and one thing well."の原則。

- core
  - 設定ファイル群を格納。各環境用の設定ファイルを```settings```ディレクトリに格納しておく。(詳細後述)
  - wsgiファイルを格納。
  - ROOT_URLを格納し、必ずここから各アプリケーションにルーティング。

- docs
  - プロジェクトのドキュメントを格納(仕様等)
  - README.rstをリポジトリのトップにおいておき、そこから参照させる形で。

- fixtures
  - Djangoのコンテクストでのfixtureは極力使わないけど、マスタデータセットアップで必要な場合は使う(住所コード等)。
  - テスト時に利用するアップロード用ファイルを置いておく。

- libs
  - 各Djangoアプリケーションから利用される共通的な処理を格納、定数を格納。
  - カスタムで作成するmanage.pyのコマンドもここに入れていく。

- requirements
  - プロジェクトに必要なライブラリを記載したrequirements.txtを入れる。
  - 各環境用 x テスト+開発に必須なrequirementsを分け、同じライブラリが複数ファイルに入るのを防ぐ。
  - requirements/common.txt, test.txt, development.txt等(詳細は後で)

- server-config
  - サーバ(OS)レベルで必要なファイルの格納(e.g. bash_profile, ssh_config)。
  - ミドルウェア(e.g. uwsgi, nginx, celery, supervisor)を動かす為に必要なファイルの格納。
  - 本当はこの辺Ansibleで統一したい。

- static
  - CSS, JS, Image等を入れる。

- templates
  - アプリ全体で使いまわすテンプレ(ナビゲーションとか)を格納。

- tests
  - 各Djangoアプリケーション内のテストで共通的に利用するものを格納。
  - 具体例でいうと、'''factories.py''', '''conftest.py'''。(factory_boy, py.test利用前提)


この辺りはDjangoのテンプレート系ライブラリをかなり参考にしました。
有名なものになると、色々な知見が凝縮されており、読んでいるだけで楽しくなれます。

- [django-skel](https://github.com/rdegges/django-skel)
- [django-project-skel](https://github.com/amccloud/django-project-skel)
- [django-kickstart](https://github.com/theduke/django-kickstart)


参考になるライブラリ群

- [Django Packages - Project Templates](https://www.djangopackages.com/grids/g/project-templates/)


設定ファイル構成
================


設定ファイル構成に何を求めるか
------------------------------

- 各環境別に異なる設定が明確
- 全環境で共通な設定が明確
- 各環境別に柔軟な設定変更が可能
- ローカルでは自分だけの設定も入れれる柔軟性が必要

1番目と2番めが非常に重要だと思っていて、何かの変数を追加したら他のファイルにも追加する必要がある、とか、変数は複数ファイルに追加したけど、値は各ファイル別にしなければいけないとか本当に良くない。そういった類のものを撲滅する、という指針で以下のようにしました。

設定ファイル実装
----------------

- common.pyに各環境共通の設定を入れる
  (大事なのは「本当に共通で使う」ものだけ入れる。DBの設定は後のimportで上書きされるから一応いれる、とかやらない。)
- 各環境用の設定ファイルを作成し、最初にcommon.pyから設定を全てimport
  + development.py
  + staging.py
  + production.py
  + local_test.py
  + ci.py
  + local.sample.py
- 各環境用設定ファイルの最後でlocal.pyから設定をimportする

common.py以外は大体以下のような形になってます。

{% highlight python %}
# -*- coding: utf-8 -*-
from core.settings.common import *  # NOQA

DEBUG = True
TEMPLATE_DEBUG = True
ENVIRONMENT = 'staging'
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'django2',
        'USER': 'hoge_user',
        'PASSWORD': 'hoge_password',
        'HOST': 'hogehost',
        'PORT': '',
    }
}

try:
    from core.settings.local import *  # NOQA
except ImportError:
    pass
{% endhighlight %}


以下若干わかりにくそうな部分を解説します。

**local.sample.py**

各開発者がローカルで開発/テストする際に独自で入れる設定用ファイル。
local.sample.py内には特に何も設定されておらず、以下のようなコメントがあるのみ。

```
# -*- coding: utf-8 -*-
# ローカル環境用設定ファイル。以下のようにコピーして利用すること。
# cp local.sample.py local.py
```

```core/settings/local.py```は```.gitignore```に記載しておき、リポジトリからは無視しておく。この```local.py```に各開発者用の独自設定を入れていきます。[Two Scoops of Django](http://www.amazon.com/Two-Scoops-Django-Best-Practices/dp/098146730X)では各開発チームメンバーの独自設定もVCSに入れる事を推奨しています。理由としてあげているのは、「有害な設定を入れていたら指摘できる」「便利な設定を入れていたら共有できる」という事でしたが、正直独自設定を入れなくとも上記2点は達成可能なのであまりしっくり来ていませんので弊社では採用していません。

**local_test.py**

ローカルでテストを実行する時に利用する設定で、PASSWORD_HASHERSとかを変更、DBをsqliteのインメモリにしたり、テストを高速化するための工夫が施してある。この中身についてはテスト戦術で詳細に書きます。


その他の設定ファイルは大体名前の通りの内容が入ってます。


設定ファイル切り替え実装
------------------------

manage.pyにはcore.developmentを直書きで指定し、開発用サーバ(manage.py runserver)は開発用設定がデフォルトで動くようにしています。これでローカルには特に環境変数設定せずともシンプルにmanage.py runserverすれば開発用サーバを起動できるようになってます。[Djangoトラノマキ](https://gist.github.com/voluntas/6855579)参照


{% highlight python %}
#!/usr/bin/env python
import os
import sys

if __name__ == "__main__":
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "core.settings.development")

    from django.core.management import execute_from_command_line

    execute_from_command_line(sys.argv)
{% endhighlight %}


各環境(CI、ステージング、本番)用に作成したファイルは、アプリケーション実行ユーザの環境変数にDJANGO_SETTINGS_MODULEを設定して切り替える方式としています。

```
export DJANGO_SETTINGS_MODULE=core.config.staging
```


*参考文献*
- [Djangoのsettingsの分割と構造化について --偏った言語信者の垂れ流し](http://d.hatena.ne.jp/nullpobug/20131015/1381763671)
- [Django トラノマキ](https://gist.github.com/voluntas/6855579)
- [パーフェクトな Django の設定ファイル](http://surgo.jp/2010/02/django.html) 


次回はDjangoにおけるテスト戦術について書きます。



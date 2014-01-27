---
layout: post
title: "Mac 10.9 Mervericsでpip install PILが失敗する"
---

Mac OSX 10.9.1、Python 2.7.5でpipを利用したPILのインストールが失敗する。
OSアップグレード前はうまくいっていた。

エラー詳細
----------

どうやらヘッダファイルが無いらしい。

    building '_imagingft' extension

    cc -fno-strict-aliasing -fno-common -dynamic -I/usr/local/include -I/usr/local/opt/sqlite/include -DNDEBUG -g -fwrapv -O3 -Wall -Wstrict-prototypes -I/System/Library/Frameworks/Tcl.framework/Headers -I/System/Library/Frameworks/Tk.framework/Headers -I/usr/local/include/freetype2 -IlibImaging -I/Users/achiku/.virtualenvs/pil/include -I/usr/local/include -I/usr/include -I/usr/local/Cellar/python/2.7.5/Frameworks/Python.framework/Versions/2.7/include/python2.7 -c _imagingft.c -o build/temp.macosx-10.7-x86_64-2.7/_imagingft.o

    _imagingft.c:73:10: fatal error: 'freetype/fterrors.h' file not found

    #include <freetype/fterrors.h>

             ^

    1 error generated.

    error: command 'cc' failed with exit status 1

    ----------------------------------------

対応
----

結論から言うと、イメージ上のフォントを弄るfreetypeをbrew経由でインストールし、PILコンパイル時に見える位置にシンボリックリンクを貼って再度pip installすることで解決。
{% highlight bash %}
$ brew install freetype
$ ln -s /usr/local/Cellar/freetype/2.5.1/include/freetype2 /usr/local/include/freetype
$ pip install PIL
{% endhighlight %}

参考/こんなになった経緯等
-------------------------

[Error installing Python Image Library using pip on Mac OS X 10.9](http://stackoverflow.com/questions/20325473/error-installing-python-image-library-using-pip-on-mac-os-x-10-9)


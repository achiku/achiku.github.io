---
layout: post
title: 'syntasticとmypyでvimからPython関数アノテーション確認'
---

vim + syntastic + mypyを使ってPython 3.5から正式に入った型アノテーションを実感する。環境はOSX 10.10.5。

### Python 3.5の準備

```
$ brew update
$ brew install python3
$ ls -l $(which python3)
lrwxr-xr-x  1 achiku  staff  35  6  9 17:10 /usr/local/bin/python3@ -> ../Cellar/python3/3.5.0/bin/python3
$ ls -l $(which pyvenv3.5)
lrwxr-xr-x  1 achiku  staff  38 10 19 01:35 /usr/local/bin/pyvenv3.5@ -> ../Cellar/python3/3.5.0/bin/pyvenv-3.5
```

### virtualenv環境の作成

```
$ pyvenv3.5 mypy
$ source ./mypy/bin/activate
$ python --version
Python 3.5.0
```

### 必要なライブラリをインストール

```
$ cat requirements.txt
git+https://github.com/JukkaL/mypy.git@master
$ pip install -r requirements.txt
$ pip freeze
mypy-lang==0.2.0.dev0
```

mypy-langをpip installで入れると0.2.0が入るのだけど、これはPython 3.5では上手く動かなかった。なので上の例ではリポジトリのmasterから入れてる。

```
$ python --version
Python 3.5.0
$ pip install mypy-lang
$ pip freeze
mypy-lang==0.2.0
$ mypy
Traceback (most recent call last):
  File "/Users/achiku/.virtualenvs/py3/bin/mypy", line 14, in <module>
    from mypy import build
  File "/Users/achiku/.virtualenvs/py3/lib/python3.5/site-packages/mypy/build.py", line 19, in <module>
    from typing import Undefined, Dict, List, Tuple, cast, Set, Union
ImportError: cannot import name 'Undefined'
```

### mypyが動く事を確認

```
$ mypy -h
usage: mypy [option ...] [-m mod | file]

Optional arguments:
  -h, --help         print this help message and exit
  --<fmt>-report dir generate a <fmt> report of type precision under dir/
                     <fmt> may be one of: html, old-html, xslt-html, xml, txt, xslt-txt
  -m mod             type check module
  -c string          type check program passed in as string
  --verbose          more verbose messages
  --use-python-path  search for modules in sys.path of running Python
  --version          show the current version information

Environment variables:
  MYPYPATH     additional module search path
```


### syntasticの準備

```vim
NeoBundle 'scrooloose/syntastic'
```

```vim
let g:syntastic_check_on_open = 1
let g:syntastic_python_checkers = ['flake8', 'pep257', 'mypy']
let g:syntastic_python_flake8_args = '--max-line-length=120'
```


### 型アノテーション付きPythonのコードを書いてみる

さっき作ったmypyというvirtualenvをactivateした状態でvimを開き以下のコードを書く。

```python
# -*- coding: utf-8 -*-


def greeting(name: str) -> str:
    """greeting function"""
    return 'Hello ' + name


if __name__ == '__main__':
    print(greeting('moqada'))
    greeting(11)
    greeting('achiku') + 1
```

これを書いて保存したタイミングで下の用に警告してくれる。これはgreeting関数が引数としてstr型を受け取るのにintを渡しているので警告してくれてるのと、greeting関数が返すstr型とint型を+演算子を使って処理しようとしているので出る警告。

![]({{ site.url }}/images/syntastic-vim.png)


### まとめ

自分が使ってるツールには結構気軽に組み込めた。ただmypy自体まだ挙動が怪しい部分があったり、issueの処理速度があがらなかったりするので継続的にウォッチしていきたい。

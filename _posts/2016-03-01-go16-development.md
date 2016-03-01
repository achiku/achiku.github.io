---
layout: post
title: 'Go 1.6開発環境整備'
---

しっくり来る所まで来たのでまとめ。


### 前提

- Mac OS X 10.11.3
- homebrew
- Go 1.6
- zsh
- neovim 1.3-dev


### Go自体の管理

homebrewでインストール。

```
$ brew install go
$ go version
go version go1.6 darwin/amd64
$ which go
/usr/local/bin/go
```

`.zshrc`に以下の環境変数を設定。

```bash
# for golang
export GOVERSION=1.6
export GOPATH=$HOME/.go/$GOVERSION
export PATH=$GOPATH/bin:$PATH
```

一応バージョンをGOPATHに入れて新しいバージョンのGoがリリースされても最初から綺麗にディレクトリ分けてビルドできるようにしてる。後方互換結構大事にしているように見受けられるのであまり心配してないけど念の為。

若干ディレクトリが深いので良く使うディレクトリにはaliasを張ってる。

```bash
alias gp='cd $GOPATH/src/github.com/achiku/'
```


### vendoring

Go 1.6からデフォルトでGo公式各コマンド(go buildとかgo testとか)が`vendor`ディレクトリを意識してくれるようになった。`gb`も良いんだけど現在はGoのvendoringで結構満足してる。vendoring tool自体は色々ある。色々あるんだけど自分はgomが好きだ。好きな点は以下の2つ。

1. `vendor/bin`にコマンドラインツールを入れる事ができる
2. 設定ファイル1つで開発環境、その他環境でインストールするべきものを分けれる

この2点、意外にその他シンプルなvendoring toolは無視しがちだと思うんだけどグローバルに特定プロジェクトの開発にだけ使いたいツールいれるの嫌だなぁと思う人には嬉しい。

gomにvendorディレクトリを意識させるには以下の設定が必要。Go 1.6でGO15VENDOREXPERIMENTがデフォルトでオンになっていてもgom自体は以下の環境変数見てるので設定しないといけない。

```bash
export GO15VENDOREXPERIMENT=1   # for gom
```

これを設定しておくとgomが作るライブラリ入れるディレクトリが`vendor`になり、インストールするディレクトリもGo標準のvendoringを意識した形にしてくれる。

Go標準のツール(go buildとかgo testとか)が`vendor`ディレクトリを意識してくれるので`gom build`や`gom test`は使わなくなり、ほぼ`gom install`と`gom exec`しか使ってない。この辺りの機能はGo 1.5以前から存在してるのでもはや役目を終えつつある。しかしながら考えぬかれた便利な道具だ。


- [github.com/mattn/gom](https://github.com/mattn/gom)

その他新し目の(vendorディレクトリを意識する)vendoring toolでシンプルで良いなと思ったのは以下の二つ。

- [github.com/govend/govend](https://github.com/govend/govend)
- [github.com/FiloSottile/gvt](https://github.com/FiloSottile/gvt)



### エディタ(基礎)


現在エディタはneovimをメインで利用。この部分はvimでも良さそうな気がしてる。以下Goを書くための入れているプラグインと設定の抜粋。


```vim
NeoBundle 'scrooloose/syntastic'
NeoBundle 'fatih/vim-go'


(..snip..)

"" syntastic
let g:syntastic_go_checkers = ['golint', 'gotype', 'govet', 'go']


"" vim-go
let g:go_fmt_command = "goimports"
let g:go_highlight_functions = 1
let g:go_highlight_methods = 1
let g:go_highlight_structs = 1
let g:go_highlight_operators = 1
let g:go_term_enabled = 1
let g:go_highlight_build_constraints = 1

augroup GolangSettings
  autocmd!
  autocmd FileType go nmap <leader>gb <Plug>(go-build)
  autocmd FileType go nmap <leader>gt <Plug>(go-test)
  autocmd FileType go nmap <Leader>ds <Plug>(go-def-split)
  autocmd FileType go nmap <Leader>dv <Plug>(go-def-vertical)
  autocmd FileType go nmap <Leader>dt <Plug>(go-def-tab)
  autocmd FileType go nmap <Leader>gd <Plug>(go-doc)
  autocmd FileType go nmap <Leader>gv <Plug>(go-doc-vertical)
  autocmd FileType go :highlight goErr cterm=bold ctermfg=214
  autocmd FileType go :match goErr /\<err\>/
augroup END
```

`vim-go`を入れてからvimのコマンドで`:GoInstallBinaries`を実行すると、デフォルトでは`$GOPATH/bin`に`goimports`, `golint`, `gocode`等のGoを書くときに便利なツールが一気に入る。全ツールアップデートしたい場合は`:GoUpdateBinaries`を実行するとアップデート可能。この二つのプラグインは本当に良く出来ていて、毎日大変お世話になってる。


- [github.com/fatih/vim-go](https://github.com/fatih/vim-go)
- [github.com/scrooloose/syntastic](https://github.com/scrooloose/syntastic)


### エディタ(コード補完)

エディタ上でのコード補完に関しては`gocode`がデファクトになってる。neovimでは`deoplete`というプラグインを使って一般的な補完をし、`deoplete-go`というプラグインを利用してGo関連補完を補強してる。

```vim
NeoBundle 'Shougo/deoplete.nvim'
NeoBundle 'zchee/deoplete-go', {'build': {'unix': 'make'}}


(..snip..)

""" deoplete
let g:deoplete#enable_at_startup = 1
let g:deoplete#enable_smart_case = 1

inoremap <expr><C-h> deolete#mappings#smart_close_popup()."\<C-h>"
inoremap <expr><BS> deoplete#mappings#smart_close_popup()."\<C-h>"

""" deoplete-go
let g:deoplete#sources#go#align_class = 1
let g:deoplete#sources#go#sort_class = ['package', 'func', 'type', 'var', 'const']
let g:deoplete#sources#go#package_dot = 1


```

これだけの設定をするだけでGoの標準ライブラリ及び`go get/install`したライブラリは補完できるようになる。問題なのは`vendor`ディレクトリに入れたプロジェクトの依存ライブラリだ。

結論から言うと以下のコマンドを打って`gocode`の`autobuild`機能を有効にすると、`vendor`ディレクトリに入れたライブラリも便利に補完可能になる。(但し`gocode`はこの機能をexperimentalと位置づけてるので動向には注意)

```
gocode set autobuild true
```

上記コマンドを打つと、gocodeの設定ファイルが以下の場所にできる。

```
$ ll .config/gocode/config.json
-rw-r--r--  1 achiku  staff  108  2 27 17:54 .config/gocode/config.json
$ cat .config/gocode/config.json
{"propose-builtins":false,"lib-path":"","autobuild":true,"force-debug-output":"","package-lookup-mode":"go"}% 
```

`gocode`は補完候補をソースコードからではなくビルド済みのバイナリから探すので、一度ビルドしないと補完が効かない。上記の設定はエディタで補完候補を探す際に`vendor`ディレクトリ内のソースをビルドして`$GOPATH`の中に入れてくれる。実際にどの場所にバイナリが入るかというと、例えば自分が作っているwbsというツールの依存ライブラリは以下の場所にバイナリが入る。

```
~/.go/1.6/pkg/darwin_amd64/github.com/achiku/wbs/vendor/github.com/BurntSushi/toml.a
~/.go/1.6/pkg/darwin_amd64/github.com/achiku/wbs/vendor/github.com/mattn/go-shellwords.a
```

プロジェクト別に依存ライブラリのバイナリが配置されるので、自動でコンパイルされて配置されてもさして影響は無いかなと思ってる。初回補完時にビルドするのである程度ラグを感じるが、一度ビルドされたら次回以降高速に補完されるので現状問題は感じていない。

- [github.com/nsf/gocode](https://github.com/nsf/gocode)
- [github.com/Shougo/deoplete.nvim](https://github.com/Shougo/deoplete.nvim)
- [github.com/zchee/deoplete-go](https://github.com/zchee/deoplete-go) 


### エディタ(テスト)

エディタで書いてテストして、というサイクルを高速に回すために`vim-test`というプラグインを入れている。これまた最高のプラグインのうちの一つだと思っててGoに限らずPythonを書くときにも多用している。

```vim
NeoBundle 'janko-m/vim-test'

(..snip..)

let g:test#strategy = 'neovim'

nmap <silent> <leader>f :TestNearest<CR>
nmap <silent> <leader>i :TestFile<CR>
nmap <silent> <leader>a :TestSuite<CR>
nmap <silent> <leader>l :TestLast<CR>
nmap <silent> <leader>g :TestVisit<CR>

let test#python#pytest#options = {
  \ 'nearest': '-v',
  \ 'file':    '-v',
  \ 'suite':   '-v',
\}
let test#go#gotest#options = {
  \ 'nearest': '-v',
  \ 'file':    '-v',
  \ 'suite':   '-v',
\}
```

`let g:test#strategy`で`neovim`を指定してneovimの機能は使ってるけどvimでもこの部分の設定を変えれば普通にいける。


- [github.com/janko-m/vim-test](https://github.com/janko-m/vim-test)


以上現在のGo 1.6開発環境でした。

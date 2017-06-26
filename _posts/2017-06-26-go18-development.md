---
layout: post
title: 'Go 1.8開発環境整備'
---

一年くらいたったので再度まとめ。


### 前提

- Mac OS X 10.11.6
- homebrew
- Go 1.8.3
- zsh
- neovim v0.2.0


### Go自体の管理

Go 1.8からデフォルトの `GOPATH` は `$HOME/go` に設定されている。

- [cmd/go: assume GOPATH=$HOME/go if not set](https://github.com/golang/go/issues/17262)

今から入るならこれでも良いかなと思うけど、自分はまだGoのバージョンをパスに含めて開発するスタイルでやっていっている。Goは後方互換をめちゃくちゃ大事にしてくれるのでGoのバージョン変えて急に挙動がズレる事はそんなに無いとは思うんだけど、一度バージョンを上げてコンパイルしたバイナリを暫く検証環境で走らす、パフォーマンステストする等の作業が主に仕事で発生して、複数バージョンローカルに持っておきたいケースがあるので以下のようなスタイルになってる。

homebrewでインストール。

```
$ brew install go
$ go version
go version go1.6 darwin/amd64
$ which go
/usr/local/bin/go
```

.zshrcに以下の環境変数を設定。

```bash
# for golang
export GOVERSION=1.8.3
export GOPATH=$HOME/.go/$GOVERSION
export PATH=$GOPATH/bin:$PATH
```

Goのバージョンを切り替えるときは以下の `brew switch` を使ってシンボリックリンクを切り替えてる。

```
$ go version
go version go1.8.3 darwin/amd64
$ brew switch --help
brew switch name version:
    Symlink all of the specific version of name's install to Homebrew prefix.
$ brew switch go 1.7.3
3 links created for /usr/local/Cellar/go/1.7.3
$ go version
go version go1.7.3 darwin/amd64
```


### vendoring

- [Dep is a prototype dependency management tool](https://github.com/golang/dep)

今は `dep` がオフィシャルなツールチェインの仲間入りを目指しているのでこれを使えば良い気がする。ただロードマップやGitHub上の議論を見ると、「これがGoの公式依存性解決ツールだ！！！！」みたいなのではなく、あくまでもそういう立ち位置を目指してるという状態が正しいので、まったりissue上げたりPRしていくのが良いのではないかと思う。以下ロードマップ。

- [Roadmap](https://github.com/golang/dep/wiki/Roadmap)


### エディタ(基礎)

現在エディタはneovimをメインで利用。この部分はvimでも良さそうな気がしてる。ベーシックな部分は全て [vim-go](https://github.com/fatih/vim-go) におまかせしている。特に `:GoRename` コマンドと、 `<Leader>dt <Plug>(go-def-tab)` には大変お世話になっている。 `:GoRename` 、本当に精神に良くて、これがあるから名前をある程度雑に決めて前に進む事ができる。「あとで別概念が出てくるかも。出てくるとしたら何か。」という思考に使うコストをほぼカットできるのは本当に良い。

```vim
"" vim-go
let g:go_fmt_command = "goimports"
let g:go_highlight_functions = 1
let g:go_highlight_methods = 1
let g:go_highlight_structs = 1
let g:go_highlight_operators = 1
let g:go_term_enabled = 1
let g:go_highlight_build_constraints = 1
let g:go_auto_type_info = 1


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


### エディタ(Linter)

以前、Go 1.6では `syntastic` を使ってlintしていたんだけど、流石に同期でlintするのは辛くなってきたので、Go 1.7に上げたタイミングぐらいで `vim-go` から[gometalinter](https://github.com/alecthomas/gometalinter) を使う方式に変更した。

`gometalinter` は簡単に説明すると「数多あるGoのLinterを並列に走らせて結果をまとめて表示してくれるやつ」だ。オプションはそれなりに設定可能なのも熱い。

ただ、`vim-go` の `gometalinter` 機能はファイル保存時に非同期実行する際のみオプションをあまりいじれなかった。(`:GoMetalinter` コマンドは `gometalinter` に与えるオプションを変更可能。`g:go_metalinter_autosave_enabled` と `g:go_metalinter_enabled` が別れてたりする。これはコマンド実行時に走らすlinterとファイル保存時に走らすlinterを分ける為にあるらしい。ただ、肝心なオプションをいじる `g:go_metalinter_command` が片方(つまりは `:GoMetalinter` コマンドの方)にしか効かないようになってる。[このへん](https://github.com/fatih/vim-go/blob/v1.13/autoload/go/lint.vim#L40-L56)。今ふと思ったんだけどvim自体の機能で`.go`開いてる時は保存したら`:GoMetalinter`実行する、とかはできそうな気がしてきた)。

とにかく、自分はGoでテストを書いてるときもlinterに走ってほしかったのだけど、大好きな `gotype` コマンドは `-a` を渡さないと `*_test.go` を検査してくれず、 `gometalinter` から走らす `gotype` に `-a` を渡すには `gometalinter` 自体に `-t` もしくは `--tests` を渡さなければならず、 `gometalinter` のオプションがいじれる且つ非同期でlinterを走らせてくれるプラグインを探した。そして現状以下に落ち着いてる。

- [github.com/w0rp/ale](https://github.com/w0rp/ale)

年初に一度試してみたけど結構辛かったが、先月くらいに再度試したらちゃんと動いた。まだまだ開発は活発。また、流石に書いてる途中に警告されるのはウザいので以下のように設定し、保存時に指定のlinterが走るように設定している。GoとPythonは仕事で書いてるけど、JavaScriptは趣味。


```vim
""" ale
let g:ale_lint_on_text_changed = 'never'
let g:ale_lint_on_enter = 1
let g:ale_lint_on_save = 1

let g:ale_sign_column_always = 1
let g:ale_set_loclist = 0
let g:ale_set_quickfix = 1

let g:ale_linters = {
\   'javascript': ['eslint', 'flow'],
\   'python': ['flake8'],
\   'go': ['gometalinter'],
\}

let g:ale_go_gometalinter_options = '--vendored-linters --disable-all --enable=gotype --enable=vet --enable=golint -t'
let g:ale_open_list = 1
let g:ale_sign_error = '✗'
let g:ale_sign_warning = '⚠'
let g:ale_statusline_format = ['⨉ %d', '⚠ %d', '⬥ ok']

let g:ale_echo_msg_error_str = 'E'
let g:ale_echo_msg_warning_str = 'W'
let g:ale_echo_msg_format = '[%linter%] %s [%severity%]'
let g:ale_python_flake8_args = '--max-line-length=120'
```

どのGo系linterを走らすかは好みの問題だとも思うんだけど、自分は `gotype`, `vet`、`golint` で十分かなという気がしている。十分速いし。あとは `gosimple` も良くて、今は正規設定に入れるか検討中。CIの中ではもっと色々走らせても良いと思う。興味がある人は `gometalinter` のREADMEとかを読むと様々なlinterが存在している事を知れて楽しい。


### エディタ(コード補完)

ここは相変わらず以下二つのプラグインに大変お世話になっております。

- [github.com/Shougo/deoplete.nvim](https://github.com/Shougo/deoplete.nvim)
- [github.com/zchee/deoplete-go](https://github.com/zchee/deoplete-go)


```vim
""" deoplete
let g:deoplete#enable_at_startup = 1
let g:deoplete#enable_smart_case = 1

""" deoplete-go
let g:deoplete#sources#go#align_class = 1
let g:deoplete#sources#go#sort_class = ['package', 'func', 'type', 'var', 'const']
let g:deoplete#sources#go#package_dot = 1
```

### エディタ(テスト)

エディタで書いてテストして、というサイクルを高速に回すために `vim-test` というプラグインを入れている。これまた最高のプラグインのうちの一つだと思っててGoに限らずPythonを書くときにも多用している。特にカーソルに一番近いテストを実行してくれる `:TestNerest` が良い。9割くらいはコレですましている。さらに `neoterm` というプラグインを入れ、 `vim-test` がテスト実行時に開く terminal を neovim の機能をフルに活かした良い感じのやつにしている(テスト結果のterminalを開きっぱなしにしたり、その中をvimのキーバインドで移動できたり、もう一回テスト実行したら同じterminal内部で実行してくれたり、色々良いneoterm)。

```vim
""" neoterm
let g:neoterm_size = 15

""" vim-test
let g:test#strategy = 'neoterm'
let g:test#preserve_screen = 1

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

- [github.com/janko-m/vim-test](https://github.com/janko-m/vim-test)
- [github.com/kassio/neoterm](https://github.com/kassio/neoterm)


以上現在のGo 1.8開発環境でした。

---
layout: post
title: 'net/httpで作るGo APIサーバー #4'
---

この一連の記事では`net/http`を主軸に据え、取替可能な部品となるライブラリを利用してAPIサーバーを作成する方法を紹介する。

- [net/httpで作るGo APIサーバー #1](http://akirachiku.com/2017/04/01/go-net-http-api-server-1.html)
- [net/httpで作るGo APIサーバー #2](http://akirachiku.com/2017/04/02/go-net-http-api-server-2.html)
- [net/httpで作るGo APIサーバー #3](http://akirachiku.com/2017/04/02/go-net-http-api-server-3.html)

上記3つの記事で`net/http`単体でもある程度の機能を満たせる形でAPIサーバーを構築できる事を示した。但し、現代的なAPIサーバーを構築する上で必要になる部分が幾つか欠けている。今回の記事では欠けているポイントのうちの一つであるHTTPルーターの話を書く。

### HTTPルーター

ここで言うHTTPルーターとは以下の事ができるものである事とする。

- HTTP Method+URLと`http.Handler`の紐付けができ、クライアントからのHTTPリクエストに対して適切な処理をディスパッチできる
- URLの一部として含まれるパラメタをパースし`http.Handler`側に渡す機能を持っている

### context導入以前

上記のような事ができるHTTPルーターはいくつかあるが、もう一つ観点として「標準ライブラリと組み合わせて使えるか」というものがある。例えば、Goの標準ライブラリとして`context`が導入される以前は`x/net/context`を組み込んだHTTPルーターがそれなりにあった。同様に`x/net/context`は使わずに独自構造体でリクエストスコープな値を管理するライブラリもある。自分が良いなと思っていたのは以下のライブラリ(テストヘルパー関数だけどコントリビュートしたこともある)。

- [xmux](https://github.com/rs/xmux)
- [xhandler](https://github.com/rs/xhandler)

xmuxは元は[httprouter](https://github.com/julienschmidt/httprouter)というradix treeを使った高速なディスパッチを売りにするライブラリをベースにして、`ServeHTTPC(context.Context, w http.ResponseWriter, r *http.Request)`という独自のインターフェースを持ち、標準ライブラリとの間にはアダプタを挟んで良い感じでリクエストスコープの値を`http.Handler`的なインターフェースである`xhandler.HandlerC`まで渡してくれる。詳細は以下の作者ブログが詳しい。

- [Our way to Go](http://engineering.dailymotion.com/our-way-to-go/)

ただしこの類の独自インターフェース拡張方式、独自リクエストスコープな値用の構造体を持つ方式は`context`が標準ライブラリである`http.Request`に追加された後にはもはや役目を終えている感がある。`context`に関しては誤解が多い領域なので予め明記しておくが、「goroutineのキャンセルをうまいことやる」という事を主眼に置いた仕組みであり「どんな値もぶち込める便利な場所」ではない。以下の議論/記事をよく読んで、それでもリクエストスコープな値、複数のgoroutineに渡していきたい値を精査した後に持たせる値を決めるのが良いと思う。

- [Golangのcontext.Valueの使い方](http://deeeet.com/writing/2017/02/23/go-context-value/)
- [Context](http://peter.bourgon.org/blog/2016/07/11/context.html)
- [peterborgon tweets](https://twitter.com/peterbourgon/status/752022730812317696)

ちなみに[httprouter](https://github.com/julienschmidt/httprouter)も「V2では`context`対応するぜ！」と言っているが[このPR](https://github.com/julienschmidt/httprouter/pull/147)が一向にマージされないので自分は使うのを諦めた経緯がある。


### context導入以後

今使っているのは以下のライブラリ。

- [gorilla/mux](https://github.com/gorilla/mux)

ビルドタグを使ってGo 1.6以前は独自構造体を使う、Go 1.7以降は標準ライブラリの`http.Request`の中に入っている`context.Context`を使う、という形になっている。

- [mux/context_nativ.go](https://github.com/gorilla/mux/blob/master/context_native.go#L1)

以下サンプルコード。

```golang
	// for gorilla/mux
	router := mux.NewRouter()
	r := router.PathPrefix("/api").Subrouter()
	r.Methods("GET").Path("/hello").Handler(chain.Then(AppHandler{h: app.Greeting}))
	r.Methods("GET").Path("/hello/staticName").Handler(publicChain.Then(AppHandler{h: app.Greeting}))
	r.Methods("GET").Path("/hello/{name}").Handler(chain.Then(AppHandler{h: app.GreetingWithName}))

	if err := http.ListenAndServe(":8080", router); err != nil {
		log.Fatal(err)
	}
```

`PathPrefix`を使ってサブルーターも作る事ができるので何度も同じURLを書かなくて良くなり、間違いが減る。また、URLの中で`{name}`の用にパラメタを設定でき、`http.Handler`の中で以下のようにして取得することができる。注意点1として、取得できるのは`map[string]string`なので適切な型にキャストして利用する必要がある。


```golang
// GreetingWithName greeting with name
func (app *App) GreetingWithName(w http.ResponseWriter, r *http.Request) (int, interface{}, error) {
	val := mux.Vars(r)
	res, err := HelloService(r.Context(), val["name"], time.Now())
	if err != nil {
		app.Logger.Printf("error: %s", err)
		e := ErrorResponse{
			Code:    http.StatusInternalServerError,
			Message: "something went wrong",
		}
		return http.StatusInternalServerError, e, err
	}
	app.Logger.Printf("ok: %v", res)
	return http.StatusOK, res, nil
}
```

注意点2としては、URLパラメタを含むURLと同じ階層で静的なURLを使いたい場合、静的なURLを先に登録する必要がある、という事。ちなみに[httprouter](https://github.com/julienschmidt/httprouter)は現時点でURLパラメタと同じ階層に静的なURLを登録できない(これもhttprouter使うのを諦めた理由の一つではある)。

```golang
    // /hello/staticNameが先、/hello/{name}は後
	r.Methods("GET").Path("/hello/staticName").Handler(publicChain.Then(AppHandler{h: app.Greeting}))
	r.Methods("GET").Path("/hello/{name}").Handler(chain.Then(AppHandler{h: app.GreetingWithName}))
```


### あわせて読みたい

- [net/httpで作るGo APIサーバー #1](http://akirachiku.com/2017/04/01/go-net-http-api-server-1.html)
- [net/httpで作るGo APIサーバー #2](http://akirachiku.com/2017/04/02/go-net-http-api-server-2.html)
- [net/httpで作るGo APIサーバー #3](http://akirachiku.com/2017/04/02/go-net-http-api-server-3.html)

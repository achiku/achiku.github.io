---
layout: post
title: 'net/httpで作るGo APIサーバー #3'
---

この一連の記事では`net/http`を主軸に据え、取替可能な部品となるライブラリを利用してAPIサーバーを作成する方法を紹介する。

- [net/httpで作るGo APIサーバー #1](http://akirachiku.com/2017/04/01/go-net-http-api-server-1.html)
- [net/httpで作るGo APIサーバー #2](http://akirachiku.com/2017/04/02/go-net-http-api-server-2.html)

今回は`http.Handler`を使い、HTTPリクエストの共通処理を記述するミドルウェアについて書く。


### ミドルウェアとは

特にそういった構造体や関数が標準ライブラリの中にあるわけではない。Goではミドルウェアを連鎖させていくことでHTTPリクエストを処理する際に必要な共通処理(e.g. ロギング、認証、panicリカバリ、etc)を記述していく。

ミドルウェアとは実際に何かというと、`func(http.Handler) http.Handler`というシグネチャを持ち、引数として渡ってきた`http.Handler`の`ServeHTTP(http.ResponseWriter, *http.Request)`を内部で実行する`http.Handler`インターフェースを満たすオブジェクトを返す関数、という感じ。`http.Handler`とは何か、何が便利なのかは[net/httpで作るGo APIサーバー #2](http://akirachiku.com/2017/04/02/go-net-http-api-server-2.html)を読んで欲しい。

言葉で書くと長いのでロギング用のミドルウェアとpanicリカバリ用ミドルウェアのコードを書いた。

- [サンプルコード](https://github.com/achiku/go-http-api-server/blob/master/ch04/middleware.go)

```golang
func loggingMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		t1 := time.Now()
		next.ServeHTTP(w, r)
		t2 := time.Now()
		t := t2.Sub(t1)
		log.Printf("[%s] %s %s", r.Method, r.URL, t.String())
	})
}

func recoverMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if err := recover(); err != nil {
				debug.PrintStack()
				log.Printf("panic: %+v", err)
				http.Error(w, http.StatusText(500), 500)
			}
		}()
		next.ServeHTTP(w, r)
	})
}
```

`http.Handler`をチェインさせていくことで、ミドルウェアA->ミドルウェアB->ミドルウェアC->アプリケーションの処理、という形で書けるようになっている。矢印で表現するより本来は順序を持ってネストしていくイメージ。以下のような形でアプリケーションの処理とURLとの紐付けに利用する。


```golang
...
	r.Methods("GET").Path("/hello").Handler(
		recoverMiddleware(loggingMiddleware(AppHandler{h: app.Greeting})))
...
```

一点注意としては、ミドルウェアは必ず並べられた順に実行されるので、例えば認証ミドルウェアで作成したユーザー情報を利用するミドルウェアは必ず認証ミドルウェアの後に書かないと正常に動かない。


### さらに見通しの良いミドルウェア

`func(http.Handler) http.Handler`というシグネチャのシンプルな形は良いのだけど、標準ライブラリだけだと`middlewareA(middlewareB(middlewareC(handler)))`という形になってしまいmiddlewareをグループ化して使いまわしにくいし、何より不格好で視認性が低い。そこで、ミドルウェアのグルーピングやその他細かい事を良い感じに埋めてくれる薄い[Alice](https://github.com/justinas/alice)というライブラリをお勧めしたい。Aliceを使うと以下のように書ける。



```golang
	// middleware chain
	chain := alice.New(
		recoverMiddleware,
		loggingMiddleware,
	)
	// for gorilla/mux
	router := mux.NewRouter()
	r := router.PathPrefix("/api").Subrouter()
	r.Methods("GET").Path("/hello").Handler(chain.Then(AppHandler{h: app.Greeting}))
```

だいぶ見通しがよくなった。また、Aliceはミドルウェア連鎖をグループ化しておけるので、例えば共通のミドルウェアチェーンはあるんだけど、publicなエンドポイントだけ別のミドルウェア群を挟みたい、privateなエンドポイントは認証を入れたい、みたいなことも綺麗に書ける。


```golang
	// middleware chain
	chain := alice.New(
		recoverMiddleware,
		loggingMiddleware,
	)
	publicChain := chain.Append(
		publicMiddleware,
	)
	privateChain := chain.Append(
		authMiddleware,
	)
```

### ミドルウェア内でDBやアプリケーション設定を使いたい

[net/httpで作るGo APIサーバー #1](http://akirachiku.com/2017/04/01/go-net-http-api-server-1.html)で説明したアプリケーショングローバルな値をミドルウェア内部で使いたくなることがある。特にDBにアクセスする認証や、アプリケーション設定内に記述した特定IPアドレスからのアクセスを拒否する、といった類のもの。`func(http.Handler) http.Handler`だからそういう値を持ったミドルウェアは書けないのか、と自分は最初思ったが、特定の値(e.g. DBのコネクションやlogger)を取って`func(http.Handler) http.Handler`を返すクロージャーを使えば簡単に書ける。

以下のはアプリケーショングローバルなloggerをミドルウェア内で利用する例。


- [サンプルコード](https://github.com/achiku/go-http-api-server/blob/master/ch04/middleware.go)

```golang
func appLoggingMiddleware(logger *log.Logger) func(next http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			t1 := time.Now()
			next.ServeHTTP(w, r)
			t2 := time.Now()
			t := t2.Sub(t1)
			logger.Printf("[%s] %s %s", r.Method, r.URL, t.String())
		})
	}
}
```

`*log.Logger`を取って`func(http.Handler) http.Handler`を返す関数になってる。これを以下のように使う。

- [サンプルコード](https://github.com/achiku/go-http-api-server/blob/master/ch04/server.go)

```golang
	// middleware chain
	chain := alice.New(
		recoverMiddleware,
		appLoggingMiddleware(app.Logger),
	)
```

こうすることで柔軟なミドルウェアを作成する事ができ、それらをグループ化して管理することで、どのエンドポイントはどのミドルウェアチェーンで処理されるのか明確になった。


### ライブラリとしてのミドルウェア

このように`func(http.Handler) http.Handler`シグネチャを守りながらミドルウェアチェーンを構築すると、同じインターフェースに対応しているサードパーティ製のライブラリを組み込みやすくなる。例えば、以下に一覧がある。

- [Awesome Go - actual middleware](https://github.com/avelino/awesome-go#actual-middlewares)

便利なミドルウェアを書いてライブラリとして公開する場合も`func(http.Handler) http.Handler`を守るのが良いと思う。


### 参考資料

- [Alice –Painless Middleware Chaining for Go](https://justinas.org/alice-painless-middleware-chaining-for-go/)
- [Making and Using HTTP Middleware](http://www.alexedwards.net/blog/making-and-using-middleware)
- [Middlewares in Go: Best practices and examples](https://www.nicolasmerouze.com/middlewares-golang-best-practices-examples/)

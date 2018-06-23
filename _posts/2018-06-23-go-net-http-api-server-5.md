---
layout: post
title: 'net/httpで作るGo APIサーバー #5'
---


この一連の記事では `net/http` を主軸に据え、取替可能な部品となるライブラリを利用してAPIサーバーを作成する方法を紹介する。

- [net/httpで作るGo APIサーバー #1](https://akirachiku.com/2017/04/01/go-net-http-api-server-1.html)
- [net/httpで作るGo APIサーバー #2](https://akirachiku.com/2017/04/02/go-net-http-api-server-2.html)
- [net/httpで作るGo APIサーバー #3](https://akirachiku.com/2017/04/02/go-net-http-api-server-3.html)
- [net/httpで作るGo APIサーバー #4](https://akirachiku.com/2017/04/08/go-net-http-api-server-4.html)

上記4つの記事で `net/http` 単体でも小さいライブラリを利用しながらある程度の機能を満たせる形でAPIサーバーを構築できる事を示した。今回はサーバーを実装する上で欠かせないロギングについて書く。


### ロギングライブラリ

まずはHTTP APIサーバーのロギングライブラリに求める事を書きだす。

- リクエスト単位で設定した情報(例: `user_id`、`source_ip`、`request_id` 等)をメタ情報として保持し、メッセージと同時に出力
    * これは毎回手で書けばいけるけど、毎回手で書くのはだるいし忘れる
    * middlewareでリクエストID等を振り出してログに出力すると、そのIDで検索すれば処理の流れが一発でわかる等、メタ情報がデフォルトで出力されていると何か発生した際の初動速度が上がる
- エラートラッキングサービスとの連携用フック
    * output系の処理にプラグインを組み込める機構になっていると嬉しい

逆に下はあまり重要視しなかったもの。

- Leveled Logging
    * これは外部ライブラリ使おうと思うと大体ついてくるのでそこまで重要視しなかった
    * [Let's talk about logging](https://dave.cheney.net/2015/11/05/lets-talk-about-logging)
        * 自分はdaveのこのブログポストがロギングの本当の課題は何かを再度検証しようとしてて結構好き
        * leveled loggingは大体のログライブラリにデフォルトでついてるので、なぜついてるのか考える契機になる気がする
- パフォーマンス
    * 作っていたのがそこまでパフォーマンス要件が厳しいアプリケーションではなかったのであまり見なかった

上記の要件で選んだ時に当時良いなと思ったのは以下のライブラリ。

- [rs/xlog](https://github.com/rs/xlog)

(現在は `xlog` の後継 [rs/zerolog](https://github.com/rs/zerolog) が出ているので今作るならそっちを使う方が良いと思う。めっちゃ速いらしいのでパフォーマンスきついところでも使えるんじゃないだろうか。)

### 実例

xlogは Go HTTP サーバーの middleware chain に組み込んで、リクエスト毎に logger を作成(もしくはPoolから取得)し、`http.Request` 内の `context.Context` に logger を入れて次の処理に渡してくれる形になっている。[justinas/alice](https://github.com/justinas/alice) を使った例を書くとこのような感じで初期化することになる。


```go
	app, err := NewApp(cfgPath)
	if err != nil {
		log.Fatal(err)
	}

	// middleware chain
	chain := alice.New(
		recoverMiddleware,
	)
	apiChain := chain.Append(
		xlog.NewHandler(NewLogConfig(app.Config)),
	)
	// for gorilla/mux
	router := mux.NewRouter()
	r := router.PathPrefix("/api").Subrouter()
	r.Methods("GET").Path("/hello").Handler(apiChain.Then(AppHandler{h: app.Greeting}))
```
(middleware chainやaliceに関してはこちら [net/httpで作るGo APIサーバー #3](https://akirachiku.com/2017/04/03/go-net-http-api-server-3.html))

xlog にデフォルトでついている関数を利用すると、サクッと以下のように logger が出力するメタデータを追加できる。

```go
	apiChain := chain.Append(
		xlog.NewHandler(NewLogConfig(app.Config)),
		xlog.MethodHandler("method"),
		xlog.URLHandler("url"),
		xlog.RemoteAddrHandler("ip"),
		xlog.UserAgentHandler("user_agent"),
		xlog.RefererHandler("referer"),
		xlog.RequestIDHandler("req_id", "Request-Id"),
	)
```


初期化した後に以下のような形で handler 側で利用する。肝は渡ってくる `http.Request` から logger を取得して利用している3行目の部分。

```go
// Greeting greeting
func (app *App) Greeting(w http.ResponseWriter, r *http.Request) (int, interface{}, error) {
	logger := xlog.FromRequest(r)
	res, err := HelloService(r.Context(), "", time.Now())
	if err != nil {
		e := ErrorResponse{
			Code:    http.StatusInternalServerError,
			Message: "something went wrong",
		}
		return http.StatusInternalServerError, e, err
	}
	logger.Debugf("%s %s", res.Name, res.Message)
	return http.StatusOK, res, nil
}
```

リクエストすると、

```
$ http http://localhost:8080/api/hello/achiku
HTTP/1.1 200 OK
Content-Length: 36
Content-Type: text/plain; charset=utf-8
Date: Sat, 23 Jun 2018 10:11:20 GMT
Request-Id: bcn1pi6jc9mjbucnus10

{
    "message": "hello",
    "name": "achiku"
}
```

以下のようなログが出力される。

```
{"file":"server.go:76","ip":"::1","level":"debug","message":"achiku hello","method":"GET","req_id":"bcn1pi6jc9mjbucnus10","role":"my-service","time":"2018-06-23T19:11:20.029910256+09:00","url":"/api/hello/achiku","user_agent":"HTTPie/0.9.9"}
```

便利なのは上記ログ、以下の1行から出されている事。

```go
	logger.Debugf("%s %s", res.Name, res.Message)
```

ログを出力する部分はメッセージだけにフォーカスでき、その他のメタデータは自動でセットされるようにできた。更に、logger にセットしておくメタデータは middleware chain の中で独自に追加することも可能。例えば、認証ミドルウェアが取得してきたユーザー情報を logger にセットしたい場合、リクエストヘッダーに入っているクライアントのバージョン情報を logger にセットしたい場合等は、以下のようにアプリケーションハンドラーをラップする `ServeHTTP` メソッドの中に書くと良いのではないだろうか。


```go
// AppHandler application handler adaptor
type AppHandler struct {
	h func(http.ResponseWriter, *http.Request) (int, interface{}, error)
}

func (a AppHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	user := getUserFromContext(r.Context())
	logger := xlog.FromRequest(r)
	logger.SetField("user_id", user.ID)
	logger.SetField("user_name", user.Name)
	logger.SetField("app_version", r.Header.Get("App-Version"))
	logger.SetField("x_forwarded_for", r.Header.Get("X-Forwarded-For"))

	encoder := json.NewEncoder(w)
	status, res, err := a.h(w, r)
	if err != nil {
		log.Printf("error: %s", err)
		w.WriteHeader(status)
		encoder.Encode(res)
		return
	}
	w.WriteHeader(status)
	encoder.Encode(res)
	return
}
```

リクエスト

```
$ http http://localhost:8080/api/hello/achiku "App-Version: 1.0.1" "X-Forwarded-For: 192.168.11.111"
```

ログ

```
{"app_version":"1.0.1","file":"server.go:84","ip":"::1","level":"debug","message":"achiku hello","method":"GET","req_id":"bcn2eiujc9mmeq0c7ba0","role":"my-service","time":"2018-06-23T19:56:11.907778774+09:00","url":"/api/hello/achiku","user_agent":"HTTPie/0.9.9","user_id":10,"user_name":"achiku","x_forwarded_for":"192.168.11.111"}
```

- [この記事で使ったコード](https://github.com/achiku/go-http-api-server/tree/master/ch08)
- [deeeeetさんとの会話](https://twitter.com/_achiku/status/907561841450147840)


<div style="width: 90%">
<blockquote class="twitter-tweet" data-lang="ja"><p lang="ja" dir="ltr">似た感じですね．今はrequest idだけですがcontext.Valueに入ってるリクエスト限定の値などをデフォルトでannotateさせるログを毎回作るようにすればいいかなという結論になりました現状は</p>&mdash; Taichi Nakashima (@deeeet) <a href="https://twitter.com/deeeet/status/907589519758594048?ref_src=twsrc%5Etfw">2017年9月12日</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</div>

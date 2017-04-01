---
layout: post
title: 'net/httpで作るGo APIサーバー #1'
---

GoにはWebサービスを作るためのフレームワークがそれなりの数存在している。

- [Awesome Go - Web Frameworks](https://github.com/avelino/awesome-go#web-frameworks)

ただ、そこまでデファクトというものがあるわけではなく、他の言語と比べると少々乱立気味なのではないかな、という感想を持っている。この記事では`net/http`を主軸に据え、取替可能な部品となるライブラリを利用してAPIサーバーを作成する方法を紹介する。

長くなりそうなので記事を分けて紹介する予定だけど、今日はアプリケーショングローバルな値をどのように保持するのが良いのかについて書く。


### アプリケーショングローバルな値

APIサーバーにはそのアプリケーションにおいてグローバルな値を保持しておきたいケースが多い。例えばAPIサーバーの設定情報だったり、外部APIにアクセスするクライアントだったり、DBへのコネクションだったり、loggerだったり。そういったものを初期化する時に、`func init()`を使ってpackageグローバルな値を作る、という策もあるが、これはその後テストをすることを考えると良い設計とは言えない。テスト実行時に自由に設定や接続先DBを変更しにくくなってしまうからだ。

そこで以下のサンプルのように`App`的な`struct`を作り、その属性としてアプリケーショングローバルな値を入れ、`main`関数内で初期化する方法が良いのではないかと思っている。

- [サンプルコード全量](https://github.com/achiku/go-http-api-server/blob/master/ch01/server.go)

```golang
// App application
type App struct {
	Host   string
	Name   string
	Logger *log.Logger
}

func main() {
	host, err := os.Hostname()
	if err != nil {
		log.Fatal(err)
	}

	app := App{
		Name:   "my-service",
		Host:   host,
		Logger: log.New(os.Stdout, fmt.Sprintf("[host=%s] ", host), log.LstdFlags),
	}
    ....
}
```

この`App`にメソッドを付与して各処理を記述していくことで、アプリケーション内で必要となる値には各処理から簡単にアクセスできるようになる。また、このメソッドのシグネチャを`func(http.ResponseWriter, *http.Request)`にして`http.HandlerFunc`の型に合うようにする事で、ある程度の規模まではその他標準ライブラリと連携しやすくなる。

```golang
// Greeting greeting
func (app *App) Greeting(w http.ResponseWriter, r *http.Request) {
	encoder := json.NewEncoder(w)
	res, err := HelloService(r.Context(), "", time.Now())
	if err != nil {
		app.Logger.Printf("error: %s", err)
		w.WriteHeader(http.StatusInternalServerError)
		encoder.Encode(ErrorResponse{
			Code:    http.StatusInternalServerError,
			Message: "something went wrong",
		})
	}
	app.Logger.Printf("ok: %v", res)
	w.WriteHeader(http.StatusOK)
	encoder.Encode(res)
}
```

サンプルコード見て「同じような処理(主にhttp.ResponseWriterとエラーハンドリング周辺)が何回も出てきて微妙だな」と思う人もいるかと思うし、自分もちょっとこれは微妙かなと思う部分が多い。後ほど書く予定の記事でこの部分は改善してみようと思うので、一旦このまま進む。

1点気をつけなければいけない事がある。この`App`は各httpリクエストに対応するgoroutineから同時にアクセスされる為、中に入れておく値は読み出すだけのものか、concurrent safeなものに限定しておくのが良い。例えば`log.Logger`はドキュメントにもあるようにconcurrent safeなstructだ([godoc](https://golang.org/pkg/log/#Logger))。


### テストを書く

上記のようにアプリケーショングローバルな値をpackageグローバルな形にするのを避け、なるべくコントロール可能な形にすることで比較的テストが書きやすくなる。

- [サンプルコード](https://github.com/achiku/go-http-api-server/blob/master/ch01/server_test.go)

```golang
func testNewApp(t *testing.T) *App {
	var logger *log.Logger
	if testing.Verbose() {
		logger = log.New(os.Stdout, "[test log] ", log.LstdFlags)
	} else {
		logger = log.New(ioutil.Discard, "[null log] ", log.LstdFlags)
	}
	return &App{
		Name:   "my-test-server",
		Host:   "test-host",
		Logger: logger,
	}
}

func TestGreeting(t *testing.T) {
	app := testNewApp(t)
	req := httptest.NewRequest(http.MethodGet, "/api/hello", nil)
	w := httptest.NewRecorder()
	app.Greeting(w, req)

	if w.Code != http.StatusOK {
		t.Errorf("want %d got %d", http.StatusOK, w.Code)
	}
	var res Greeting
	decoder := json.NewDecoder(w.Body)
	if err := decoder.Decode(&res); err != nil {
		t.Fatal(err)
	}
	if expected := "anonymous"; res.Name != expected {
		t.Errorf("want %s got %s", expected, res.Name)
	}
	if expected := "hello!"; res.Message != expected {
		t.Errorf("want %s got %s", expected, res.Message)
	}
}
```

何の変哲もないテストだけど注目してほしいのは`testNewApp`関数の中での`App`初期化だ。このようにテスト用のAppを外部からの入力によって変更する事がとても簡単にできる。

今回のケースでは`go test -v`の`-v`が渡されていた場合、loggerの出力先を変更している。自分は`go test`の成功すると何も出力されない、という仕様がとても好きなんだけど、時にはテストしたい関数の中にデバッグログを仕込んで出力を見たい、という場合がある。そのような場合に、Appが持つLoggerの属性をパラメタを渡すことで変更している。

Loggerに限らず、例えば後で詳細に書くけど外部のAPI ClientやDBのMockをこの部分で初期化する事で、実際にサーバーとして走らすAppとは異なるAppを作り、テストしたい関数(今回の場合はApp.Greeting)に集中してテストが書けるようになっている。

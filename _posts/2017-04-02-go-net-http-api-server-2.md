---
layout: post
title: 'net/httpで作るGo APIサーバー #2'
---

この一連の記事では`net/http`を主軸に据え、取替可能な部品となるライブラリを利用してAPIサーバーを作成する方法を紹介する。

- [net/httpで作るGo APIサーバー #1](http://akirachiku.com/2017/04/01/go-net-http-api-server-1.html)

今回は`http.Handler`インターフェースについて書く。このインターフェースを上手く利用する事が`het/http`でAPIサーバーを作る時にとても重要になると思ってる。


### Goのインターフェース

Goのインターフェースを網羅的に説明しようとすると長大な記事が書けてしまうので、今回は`http.Handler`を理解していくのに必要な部分だけに絞って少し書く。以下のサンプルコードを元に話しを進める。

- [サンプルコード](https://github.com/achiku/go-http-api-server/blob/master/ch02/example/example.go)

```golang
package main

import "fmt"

type person struct {
	Age  int
	Name string
}

func (p person) greeting() string {
	return fmt.Sprintf("hey! I'm %s", p.Name)
}

func (p person) name() string {
	return p.Name
}

type dog struct {
	Age   int
	Name  string
	Owner string
}

func (d dog) greeting() string {
	return "wan!"
}

func (d dog) name() string {
	return d.Name
}

type nameTyp string

func (s nameTyp) name() string {
	return string(s)
}

func (s nameTyp) greeting() string {
	return fmt.Sprintf("nameType: %s", s)
}

type animal interface {
	greeting() string
	name() string
}

func main() {
	p1 := person{
		Name: "achiku",
		Age:  31,
	}
	p2 := person{
		Name: "moqada",
		Age:  31,
	}
	d1 := dog{
		Name:  "taro",
		Age:   2,
		Owner: "moqada",
	}
	n1 := nameTyp("8maki")
	for _, a := range []animal{p1, p2, d1, n1} {
		fmt.Printf("%s: %s\n", a.name(), a.greeting())
	}
}
```

ここで示しているのは`animal`インターフェースを実装しているものはどのような型でもanimalとして使えるという事。「どのような型でも」と付けたのは、structに付与したメソッドがインターフェースと一致しているかどうかに限らず、上記サンプルで示した`nameType`という実態は`string`の型に付与したメソッドでもインターフェースを満たしていれば`animal`として扱える、という事。言葉は乱暴かもしれないけどコンパイル時型チェックなduck typingみたいな感じに使える。

詳細は以下の記事がステップバイステップで解説していて面白い。

- [インタフェースの実装パターン #golang](http://qiita.com/tenntenn/items/eac962a49c56b2b15ee8)




### Goのメソッド

Goのメソッドはカスタム型になら例えそれが関数だろうと付与できる。

- [サンプルコード](https://github.com/achiku/go-http-api-server/blob/master/ch02/example02/main.go)

```golang
package main

import "fmt"

type myFunc func() string

func (f myFunc) WrapFunc() {
	fmt.Print("before msg\n")
	fmt.Printf("%s\n", f())
	fmt.Print("after msg\n")
}

func main() {
	f := myFunc(func() string {
		return "hello!!"
	})
	f.WrapFunc()
}
```

`myFunc`型に`WrapFunc()`メソッドを追加している。ここで面白いのは`myFunc`は関数だということ。関数にメソッドを付与する、というのが最初は理解しづらかったのを覚えているんだけど、「カスタム型には元が関数だろうがstringだろうがメソッドは付与できる」という事。このサンプルコードが一番`http.Handler`と`http.HandlerFunc`の関係性に近い形になっている。


### http.Handlerというインターフェース

やっとここまできた。いきなりだけど以下が`http.Handler`というインターフェースの全量となる。

```golang
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

`ServeHTTP`というメソッド一つ持っていればそれは全て`http.Handler`として扱う事ができる。なんとも小さくて頼りない感じがするが、`http.Request`と`http.ResponseWriter`と合わさってとてもシンプルにHTTPリクエストを扱えるようになっている。以下にあまり現実的ではないけど、こういう風にも作れるという例を示す。


```golang
package main

import (
	"fmt"
	"log"
	"net/http"
)

type myString string

func (s myString) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	log.Printf("myString=%s accessed", s)
	w.WriteHeader(http.StatusOK)
	fmt.Fprintf(w, "myString=%s", s)
	return
}

func main() {
	mux := http.NewServeMux()
	mux.Handle("/mystring/1", myString("1"))
	mux.Handle("/mystring/2", myString("2"))

	if err := http.ListenAndServe(":8080", mux); err != nil {
		log.Fatal(err)
	}
}
```

`myString`型は`ServeHTP(http.ResponseWriter, *http.Request)`というシグネチャを満たすので`http.Handler`として利用できる。`http.Handler`として利用できるので、`mux.Handle(string, http.Handler)`に渡せる。そうすることでURLとHTTPリクエスト/レスポンスの処理が紐付き、サーバーとして起動できる、という流れ。

以下は良くGoの`net/http`の例で見る`func(http.ResponseWriter, *http.Request)`とURLを紐付けるタイプのやつ。

```golang
	mux.HandleFunc("/mystring/3", func(w http.ResponseWriter, r *http.Request) {
		log.Printf("myString=%s accessed", "3")
		w.WriteHeader(http.StatusOK)
		fmt.Fprintf(w, "myString=%s", "3")
		return
	})
```

これも結構面白くて、`ServeMux.HandleFunc`は以下のような形になってる。

```golang
// HandleFunc registers the handler function for the given pattern.
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	mux.Handle(pattern, HandlerFunc(handler))
}
```

`http.HandlerFunc`が何をやっているかというと。

```golang
// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers. If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// Handler that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

これは関数`func(http.ResponseWriter, *http.Request)`に`ServeHTTP`メソッドを追加してただただその関数を呼ぶ、という形になっており、あくまでもシンタックスシュガーとしてURLと`http.Handler`が実行する処理の紐付けを行っている事がわかる。たまにURL routing系のライブラリで`http.HandlerFunc`しか取らないものがあるんだけど、シンタックスシュガーだけ取り入れて肝心の`http.Handler`を受けれないのはライブラリとしてはもったいないなという思いがある。


### 何が嬉しいのか

気軽に使いたいだけなら`http.HandlerFunc`のシグネチャに合う関数`func(http.ResponseWriter, *http.Request)`を書いてサクッと使える。もし実際の処理の前後に何か処理を挟みたい、処理を共通化したい、そもそも`func(http.ResponseWriter, *http.Request)`の空returnがいまいち好きになれない、等、ということになれば`ServerHTTP(http.ResponseWriter, *http.Request)`をメソッドに持つ型を作り、それに処理をする独自シグネチャを持つ関数を渡してURLに紐付けるのが良いのではないか。

例えば独自シグネチャをステータスコード、JSONレスポンス用構造体、エラーを返せるように`func(http.ResponseWriter, *http.Request) (int, interface{}, error)`とおき、`AppHandler`という`ServeHTTP`をメソッドとして持つ構造体をアダプターとして使うパターン。

- [サンプルコード](https://github.com/achiku/go-http-api-server/blob/master/ch03/server.go)

```golang
// AppHandler application handler adaptor
type AppHandler struct {
	h func(http.ResponseWriter, *http.Request) (int, interface{}, error)
}

func (a AppHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
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

// Greeting greeting
func (app *App) Greeting(w http.ResponseWriter, r *http.Request) (int, interface{}, error) {
	res, err := HelloService(r.Context(), "", time.Now())
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

func main() {
...

	// for gorilla/mux
	router := mux.NewRouter()
	r := router.PathPrefix("/api").Subrouter()
	r.Methods("GET").Path("/hello").Handler(AppHandler{h: app.Greeting})
...
```

こうする事でエラー発生時のロギング、HTTP StatusCodeの値によって別URLにリダイレクト、JSONをResponseWriterへの書き込み等を共通的に扱う事ができるようになった。また、これは主観だけど空returnが無くなるのでうっかりエラー処理時にreturn書き忘れる事も減る気がする。

テストは以下のようになる。[この記事で書いたもの](http://akirachiku.com/2017/04/01/go-net-http-api-server-1.html)からあまり変化は無いかも。


```golang
func TestGreeting(t *testing.T) {
	app := testNewApp(t)
	req := httptest.NewRequest(http.MethodGet, "/api/hello", nil)
	w := httptest.NewRecorder()
	status, res, err := app.Greeting(w, req)
	if err != nil {
		t.Fatal(err)
	}
	if status != http.StatusOK {
		t.Fatalf("want %d got %d", http.StatusOK, status)
	}
	gt, ok := res.(*Greeting)
	if !ok {
		t.Fatalf("want type Greeting got %s", reflect.TypeOf(res))
	}
	if expected := "anonymous"; gt.Name != expected {
		t.Errorf("want %s got %s", expected, gt.Name)
	}
	if expected := "hello!"; gt.Message != expected {
		t.Errorf("want %s got %s", expected, gt.Message)
	}
}
```

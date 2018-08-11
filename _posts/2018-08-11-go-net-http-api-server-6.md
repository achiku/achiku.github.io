---
layout: post
title: 'net/httpで作るGo APIサーバー #6'
tags: golang go net/http api server
---


この一連の記事では `net/http` を主軸に据え、取替可能な部品となるライブラリを利用してAPIサーバーを作成する方法を紹介する。

- [net/httpで作るGo APIサーバー #1](https://akirachiku.com/2017/04/01/go-net-http-api-server-1.html)
- [net/httpで作るGo APIサーバー #2](https://akirachiku.com/2017/04/02/go-net-http-api-server-2.html)
- [net/httpで作るGo APIサーバー #3](https://akirachiku.com/2017/04/02/go-net-http-api-server-3.html)
- [net/httpで作るGo APIサーバー #4](https://akirachiku.com/2017/04/08/go-net-http-api-server-4.html)
- [net/httpで作るGo APIサーバー #5](https://akirachiku.com/2018/06/23/go-net-http-api-server-5.html)


上記5つの記事で `net/http` 単体でも小さいライブラリを利用しながらある程度の機能を満たせる形でAPIサーバーを構築できる事を示した。今回はエラーハンドリングについて書く。サーバーサイドのエラーハンドリングを中心に書くが、この領域はフロントエンドとも密に連携していく必要があるため、その辺りの仕組みをどうするかに関しても言及していく。


### "エラー"と呼ばれるものの種類と扱い方

Goで"エラー"という言葉が表すものを恣意的に二種類に分類した。それぞれの定義と簡易的な扱い方を以下に書く。

#### ハンドリングの余地の無いエラー

- 例: 存在しないスライスのインデックスにアクセスした、等
- アプリケーション側ではこれ以上どうしようもない、という状態になるエラー
- このタイプのエラーは `panic` を利用する

[Code Review Comment](https://github.com/golang/go/wiki/CodeReviewComments#dont-panic) にもあるようにアプリケーションコードの中で利用することはあまり無いかなという印象。たまにGoの `error` を扱うのがダルいので `error` を返さずに `panic` で終了！みたいなコードをライブラリ内で見ることがあるけど、そのような使い方をしてしまうとライブラリ利用側でそのエラーが発生した際にどのような挙動をするべきか決めるという自由を奪い強制的に終了することになるので、少なくとも利用には慎重になるのが大切かなと思う。(もちろん `error` を返すよりも `panic` で落とすケースの方が適切な場合もあると思うのであくまで `panic` 使う前に慎重になる、という意味)


#### ハンドリングの余地の有るエラー

- 例: 存在しないユーザーIDがリクエストされてきた、等
- アプリケーション側で発生したエラーによって挙動、メッセージを変更したいエラー
- このタイプのエラーは `error` を利用し適切なハンドリングをする


Goで `error` はビルトインのインターフェース型の一つ。以下の形を満たせばどのような構造体でも関数でも `error` として扱える。

```go
type error interface {
    Error() string
}
```

また標準ライブラリの `errors`　パッケージの中を覗いてみるとこういうコードになっている。[src/errors/errors.go](https://golang.org/src/errors/errors.go)。非常にシンプルだが拡張性のある形になっている。ただし、それが故に以下2点に関しては少し工夫が必要になる。

- `error` に付加情報を持たせる
    * 関数の深いところから `error` が返ってくる場合に `error` インターフェースを満たしながら各階層でコンテクストを付加して呼び出し元に渡したい場合がある
    * これは処理がGoの中で閉じるならよほどのことが無い限り [github.com/pkg/errors](https://github.com/pkg/errors) を利用すれば良いと思う
        * `errors.Wrap` もしくは `errors.Wrapf` で呼び出した関数から上がってきた `error` に何某かの付加情報を付与して返す形になる
        * これをやることで `no such file or directory` だけのエラーが返ってくる事がなくなり、どこで何の関数が失敗したのかわかりやすくなる
        * ただしフロントエンドに返すエラーとなると話が違うので後ほど詳細に書く
- `error` に種類を持たせる
    * `error` が呼び出した関数から返ってきた場合、具体的に何のエラーかによって呼び出し元で処理を分けたい場合がある
    * 実装方法は大きく分けて3つあり、それぞれにメリット/デメリットがある
        * この記事が最高に参考になる
        * [Don't just check errors, handle them gracefully](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully) 

「 `error` に付加情報を持たせる」に関してはあまり大きな論点が無いのでそのままにするとして、「 `error` に種類を持たせる」の部分はもう少し詳細に書く。

### どのようにerrorに種類を持たせるか

原則 [Don't just check errors, handle them gracefully](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully) を読めばいけるんだけどざっくりと要約すると以下になる。

- daveの結論
    * Opaque Errorを使うとパッケージ間の依存を極限まで減らせて便利
- エラーに種類を持たせ、種類によって呼び出し元の処理を変える方法は大きく3つあり、それぞれメリットとデメリットがある
- Sentinel Error
    * エラーを値としてパッケージ内に持つ (例: `io.EOF`)
    * 値なので `==` で比較して呼び出し元の処理を分ける
    * メリット
        * あんまりない
    * デメリット
        * エラーに追加で情報をもたせたりする事ができなくなる
            * `var EOF = errors.New("EOF")` で定義されているので
        * このエラー自体をパブリックなAPIとして他のパッケージが利用してしまうので変更が難しくなる


```go
if err == ErrSomething { … }
```


- Error Type
    * エラーを型を持つ構造体としてパッケージ内に持つ (例: `os.PathError`)
    * Type Assertionを使って呼び出し元の処理を分ける
    * メリット
        * 型なので属性を増やせば追加でエラーのメタデータを増やせる
            * `os.PathError` には `Op` , `Path` というメタデータが存在している
    * デメリット
        * このエラー自体をパブリックなAPIとして他のパッケージが利用してしまうので変更が難しくなる


```go
type MyError struct {
        Msg string
        File string
        Line int
}

func (e *MyError) Error() string { 
        return fmt.Sprintf("%s:%d: %s”, e.File, e.Line, e.Msg)
}

return &MyError{"Something happened", “server.go", 42}
```


```go
err := something()
switch err := err.(type) {
case nil:
        // call succeeded, nothing to do
case *MyError:
        fmt.Println(“error occurred on line:”, err.Line)
default:
// unknown error
}
```



- Opaque Error
    * 特定のインターフェースをパッケージ内に持つ (例: `net.Error`)
    * Type Assertionを使って呼び出し元の処理を分ける
    * メリット
        * インターフェースなのでパッケージ外に出る際付加情報を加える事も容易(インターフェースを満たす構造体をエラーとして利用)
        * 呼び出し側は特定の型を別パッケージからimportする必要無くエラー発生時に処理を分岐させれる
    * デメリット
        * あんまりない


```go
type temporary interface {
        Temporary() bool
}
 
// IsTemporary returns true if err is temporary.
func IsTemporary(err error) bool {
        te, ok := err.(temporary)
        return ok && te.Temporary()
}
```


こっからは自分の主観。Opaque Errorは確かに便利で標準ライブラリでも良く利用されているパターンだと思うんだけど、アプリケーション内部で使うにはそこまでパッケージの結合度合いを考えないで良いのではないかなぁという思いがある(特にアプリケーション構築初期には)。なので自分はHTTP APIサーバーを作るというコンテクストでは Error Type 推し。ただ Opaque Error を使うと1関数から複数の Error Type が返ってくる想定で且つそれぞれの Error Type が `Tempolary()` と `Timeout()` を持っている場合、横断的なエラーハンドリングがしやすくはなるなとは思う。余談だけど、このタイプのメソッドは `Tempolary()` と `Timeout()` くらいしか見たことがなく自分の知見が狭いだけな可能性があるため、他にもこういう形のエラーハンドリングを使ってる場所があれば知りたい。


### HTTP APIサーバーという文脈でそれらをどのように扱うか

長かった。HTTP APIサーバーの中で `error` をどうやって使うかという話を書く。結論から書くと、カンムが提供するバンドルカードのAPIは「APIサーバーの中では原則 `error` のみ利用し、本当に必要な場合は Error Type を使う。フロントとのやり取りをするためにはそれ用の Error Type を定義してやりとりする。エラーログの出力は `ServeHTTP` の中で実施する。」という形をとっている。一つづついきます。


#### APIサーバーの中では原則errorのみ利用し、本当に必要な場合は Error Type を使う

現状のパッケージ構成はいまだこの構成のまま。

<script async class="speakerdeck-embed" data-slide="10" data-id="9c6e284d996840b1a542b2beaf5ca30e" data-ratio="1.33333333333333" src="//speakerdeck.com/assets/embed.js"></script>

また、特にパッケージ毎にエラーを定義せずに原則 `error` を `errors.Wrap` して `model` -> `service` -> `handler` に伝播させていっている。Error Type を使わなかった理由は特に無くて、ただただアプリケーション作るだけならあんまし必要なくね、って思ったから。

例えば、`model.GetUserByID(tx sql.Tx, id int64) (*model.User, error)` という関数があったとして、その関数から `UserNotFoundError` や `UserSuspendedError` を返す事もできる。


#### フロントとのやり取りをするためにはそれ用の Error Type を定義してやりとりする


#### エラーログの出力はServeHTTPの中で実施する

```go
// ServeHTTPC for iapi
func (h IapiHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	encoder := json.NewEncoder(w)
	logger := xlog.FromRequest(r)

	status, res, err := h.handler(w, r)
	switch {
	// 204 でbodyにWriteすると以下のエラーが発生するので 204 の場合はwriteしない
	// http: request method or response status code does not allow body
	case status == http.StatusNoContent:
		return
	// 300系はとりあえずリダイレクト
	case status == http.StatusFound || status == http.StatusMovedPermanently:
		redirectURL, ok := res.(string)
		if !ok {
			logger.Error("failed to convert res to redirectURL")
			w.WriteHeader(http.StatusInternalServerError)
			encoder.Encode(res)
			return
		}
		http.Redirect(w, r, redirectURL, status)
		return
	case status >= http.StatusBadRequest && status < http.StatusInternalServerError:
		logger.Warn(err)
	case status >= http.StatusInternalServerError:
		logger.Error(err)
	}

	w.WriteHeader(status)
	if err := encoder.Encode(res); err != nil {
		logger.Error(err)
		w.WriteHeader(http.StatusInternalServerError)
		encoder.Encode(errorResponseUnknown)
		return
	}
	return
}
```



### 効率的に扱うための仕組み


### 参考資料

- [Failure is your Domain](https://middlemost.com/failure-is-your-domain/)
- [Don't just check errors, handle them gracefully](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)
- [Error handling and Go](https://blog.golang.org/error-handling-and-go)
- [Golangのエラー処理とpkg/errors](https://deeeet.com/writing/2016/04/25/go-pkg-errors/)
- [Error handling in Upspin](https://commandcenter.blogspot.com/2017/12/error-handling-in-upspin.html)

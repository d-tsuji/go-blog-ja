=========================================
Go Concurrency Patterns: Context
=========================================

概要
=========================================

Goのサーバーでは、要求されたリクエストはそれぞれのゴルーチンで処理されます。リクエストハンドラーは多くの場合、データベースやgRPCサーバといったバックエンドにアクセスするために、新しいゴルーチンを作成します。通常、リクエストを処理するゴルーチンのセットは、エンドユーザーを識別するID、認証トークン、リクエストの期限といったリクエスト固有の値にアクセスする必要があります。リクエストがキャンセルまたはタイムアウトになった場合、そのリクエストに紐付いて動いているゴルーチンはすぐに終了し、システムが使用中のリソースを回収できるようになります。

Googleでは、APIの境界を越えて、リクエストの処理に関係するすべてのゴルーチンに、リクエストスコープの値、キャンセルシグナル、および期限を簡単に渡すことができる ``context`` パッケージを開発しました。パッケージは、 ``context`` として公開されています。この記事では、パッケージの使用方法について説明し、そのままで動作する実例を提供します。

Context
=========================================

``context`` パッケージの中核は、 ``Context`` 型です。

.. code-block:: go

    // A Context carries a deadline, cancelation signal, and request-scoped values
    // across API boundaries. Its methods are safe for simultaneous use by multiple
    // goroutines.
    type Context interface {
        // Done returns a channel that is closed when this Context is canceled
        // or times out.
        Done() <-chan struct{}

        // Err indicates why this context was canceled, after the Done channel
        // is closed.
        Err() error

        // Deadline returns the time when this Context will be canceled, if any.
        Deadline() (deadline time.Time, ok bool)

        // Value returns the value associated with key or nil if none.
        Value(key interface{}) interface{}
    }

(上記の説明は、GoDocの抜粋です)

``Done`` メソッドは ``Context`` に代わって、実行されている関数にキャンセルのシグナルとして機能するチャネルを返します。チャネルが閉じられると、関数は処理を中止して終了します。``Err`` メソッドは、``Context`` がキャンセルされた理由を示すエラーを返します。「Pipelines and Cancelation」の記事では、``Done`` チャンネルのイディオムについて詳しく説明しています。

``Done`` チャネルが受信専用であることと同じ理由で、``Context`` は ``Cancel`` メソッドが `ありません` 。通常の場合、キャンセルシグナルを受信する関数はシグナルを送信する関数ではありません。特に親の操作が子の操作としてゴルーチンを開始する場合、子の操作は親をキャンセルできないようにすべきです。代わりに ``WithCancel`` 関数(後ほど説明)はキャンセルするための ``Context`` の値を提供します。

``Context`` は、複数のゴルーチンで同時に使用しても安全です。コードは1つの ``Context`` を任意の数のゴルーチンに渡し、その ``Context`` をキャンセルしてすべてのゴルーチンを通知できます。

``Deadline`` メソッドを使用すると、関数で処理を開始するかどうかを決定できます。あまりにも短い時間しか残っていない場合、処理する価値がないかもしれません。コードは期限を用いて、I/O操作のタイムアウトを設定することもできます。

``Value`` を使用すると ``Context`` でリクエストスコープのデータを伝送できます。そのデータは、複数のゴルーチンによる同時使用に対して安全でなければなりません。

** Derived contexts
-----------------------------------------

The `context` package provides functions to _derive_ new `Context` values from
existing ones.
These values form a tree: when a `Context` is canceled, all `Contexts` derived
from it are also canceled.

`Background` is the root of any `Context` tree; it is never canceled:

.code context/interface.go /Background returns/,/func Background/

`WithCancel` and `WithTimeout` return derived `Context` values that can be
canceled sooner than the parent `Context`.
The `Context` associated with an incoming request is typically canceled when the
request handler returns.
`WithCancel` is also useful for canceling redundant requests when using multiple
replicas.
`WithTimeout` is useful for setting a deadline on requests to backend servers:

.code context/interface.go /WithCancel/,/func WithTimeout/

`WithValue` provides a way to associate request-scoped values with a `Context`:

.code context/interface.go /WithValue/,/func WithValue/

The best way to see how to use the `context` package is through a worked
example.

* Example: Google Web Search
=========================================

Our example is an HTTP server that handles URLs like
`/search?q=golang&timeout=1s` by forwarding the query "golang" to the
[[https://developers.google.com/web-search/docs/][Google Web Search API]] and
rendering the results.
The `timeout` parameter tells the server to cancel the request after that
duration elapses.

The code is split across three packages:

- [[context/server/server.go][server]] provides the `main` function and the handler for `/search`.
- [[context/userip/userip.go][userip]] provides functions for extracting a user IP address from a request and associating it with a `Context`.
- [[context/google/google.go][google]] provides the `Search` function for sending a query to Google.

** The server program
-----------------------------------------

The [[context/server/server.go][server]] program handles requests like
`/search?q=golang` by serving the first few Google search results for `golang`.
It registers `handleSearch` to handle the `/search` endpoint.
The handler creates an initial `Context` called `ctx` and arranges for it to be
canceled when the handler returns.
If the request includes the `timeout` URL parameter, the `Context` is canceled
automatically when the timeout elapses:

.code context/server/server.go /func handleSearch/,/defer cancel/

The handler extracts the query from the request and extracts the client's IP
address by calling on the `userip` package.
The client's IP address is needed for backend requests, so `handleSearch`
attaches it to `ctx`:

.code context/server/server.go /Check the search query/,/userip.NewContext/

The handler calls `google.Search` with `ctx` and the `query`:

.code context/server/server.go /Run the Google search/,/elapsed/

If the search succeeds, the handler renders the results:

.code context/server/server.go /resultsTemplate/,/}$/

** Package userip
-----------------------------------------

The [[context/userip/userip.go][userip]] package provides functions for
extracting a user IP address from a request and associating it with a `Context`.
A `Context` provides a key-value mapping, where the keys and values are both of
type `interface{}`.
Key types must support equality, and values must be safe for simultaneous use by
multiple goroutines.
Packages like `userip` hide the details of this mapping and provide
strongly-typed access to a specific `Context` value.

To avoid key collisions, `userip` defines an unexported type `key` and uses
a value of this type as the context key:

.code context/userip/userip.go /The key type/,/const userIPKey/

`FromRequest` extracts a `userIP` value from an `http.Request`:

.code context/userip/userip.go /func FromRequest/,/}/

`NewContext` returns a new `Context` that carries a provided `userIP` value:

.code context/userip/userip.go /func NewContext/,/}/

`FromContext` extracts a `userIP` from a `Context`:

.code context/userip/userip.go /func FromContext/,/}/

** Package google
-----------------------------------------

The [[context/google/google.go][google.Search]] function makes an HTTP request
to the [[https://developers.google.com/web-search/docs/][Google Web Search API]]
and parses the JSON-encoded result.
It accepts a `Context` parameter `ctx` and returns immediately if `ctx.Done` is
closed while the request is in flight.

The Google Web Search API request includes the search query and the user IP as
query parameters:

.code context/google/google.go /func Search/,/q.Encode/

`Search` uses a helper function, `httpDo`, to issue the HTTP request and cancel
it if `ctx.Done` is closed while the request or response is being processed.
`Search` passes a closure to `httpDo` handle the HTTP response:

.code context/google/google.go /var results/,/return results/

The `httpDo` function runs the HTTP request and processes its response in a new
goroutine.
It cancels the request if `ctx.Done` is closed before the goroutine exits:

.code context/google/google.go /func httpDo/,/^}/

* Adapting code for Contexts
=========================================

Many server frameworks provide packages and types for carrying request-scoped
values.
We can define new implementations of the `Context` interface to bridge between
code using existing frameworks and code that expects a `Context` parameter.

For example, Gorilla's
[[http://www.gorillatoolkit.org/pkg/context][github.com/gorilla/context]]
package allows handlers to associate data with incoming requests by providing a
mapping from HTTP requests to key-value pairs.
In [[context/gorilla/gorilla.go][gorilla.go]], we provide a `Context`
implementation whose `Value` method returns the values associated with a
specific HTTP request in the Gorilla package.

Other packages have provided cancelation support similar to `Context`.
For example, [[https://godoc.org/gopkg.in/tomb.v2][Tomb]] provides a `Kill`
method that signals cancelation by closing a `Dying` channel.
`Tomb` also provides methods to wait for those goroutines to exit, similar to
`sync.WaitGroup`.
In [[context/tomb/tomb.go][tomb.go]], we provide a `Context` implementation that
is canceled when either its parent `Context` is canceled or a provided `Tomb` is
killed.

* Conclusion
=========================================

At Google, we require that Go programmers pass a `Context` parameter as the
first argument to every function on the call path between incoming and outgoing
requests.
This allows Go code developed by many different teams to interoperate well.
It provides simple control over timeouts and cancelation and ensures that
critical values like security credentials transit Go programs properly.

Server frameworks that want to build on `Context` should provide implementations
of `Context` to bridge between their packages and those that expect a `Context`
parameter.
Their client libraries would then accept a `Context` from the calling code.
By establishing a common interface for request-scoped data and cancelation,
`Context` makes it easier for package developers to share code for creating
scalable services.

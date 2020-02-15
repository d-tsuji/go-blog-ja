=========================================
Go Concurrency Patterns: Context
=========================================

Go Concurrency Patterns: Context
29 Jul 2014
Tags: concurrency, cancelation, cancellation, context

Sameer Ajmani

---------

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

``Done`` メソッドは ``Context`` に代わって、実行されている関数にキャンセルのシグナルとして機能するチャネルを返します。チャネルが閉じられると、関数は処理を中止して終了します。``Err`` メソッドは、``Context`` がキャンセルされた理由を示すエラーを返します。「Pipelines and Cancelation」の記事では、``Done`` チャネルのイディオムについて詳しく説明しています。

``Done`` チャネルが受信専用であることと同じ理由で、``Context`` は ``Cancel`` メソッドが `ありません` 。通常の場合、キャンセルシグナルを受信する関数はシグナルを送信する関数ではありません。特に親の操作が子の操作としてゴルーチンを開始する場合、子の操作は親をキャンセルできないようにすべきです。代わりに ``WithCancel`` 関数(後ほど説明)はキャンセルするための ``Context`` の値を提供します。

``Context`` は、複数のゴルーチンで同時に使用しても安全です。コードは1つの ``Context`` を任意の数のゴルーチンに渡し、その ``Context`` をキャンセルしてすべてのゴルーチンを通知できます。

``Deadline`` メソッドを使用すると、関数で処理を開始するかどうかを決定できます。期限までほとんど時間がない場合、処理する価値がないかもしれません。コードはを用いて、I/O操作のタイムアウトを設定することもできます。

``Value`` を使用すると ``Context`` でリクエストスコープのデータを送ることができます。そのデータは、複数のゴルーチンによる同時使用に対して安全でなければなりません。

コンテキストの派生
-----------------------------------------

``context`` パッケージは元のコンテキストから新しい ``Context`` の値を生成する関数を提供しています。コンテキストは木を形成します。根の ``Context`` がキャンセルされると、そこから派生したすべての ``Context`` もキャンセルされます。

``Background`` は ``Context`` の木の根になります。キャンセルされることはありません。

.. code-block:: go

    // Background returns an empty Context. It is never canceled, has no deadline,
    // and has no values. Background is typically used in main, init, and tests,
    // and as the top-level Context for incoming requests.
    func Background() Context

``WithCancel`` と ``WithTimeout`` は派生した ``Context`` の値を返し、それらは親の ``Context`` よりも早くキャンセルできます。通常 ``Context`` はリクエストに関連する ``Context`` はリクエストハンドラが返却されるとキャンセルされます。

.. todo::

    ちょっと違和感がある。

    The ``Context`` associated with an incoming request is typically canceled when the request handler returns.

``WithCancel`` は複数のレプリカを使用しているときに、冗長なリクエストをキャンセルする場合にも役に立ちます。``WithTimeout`` はバックエンドサーバへのリクエストに期限を設定するのに役立ちます。

.. code-block:: go

    // WithCancel returns a copy of parent whose Done channel is closed as soon as
    // parent.Done is closed or cancel is called.
    func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

    // A CancelFunc cancels a Context.
    type CancelFunc func()

    // WithTimeout returns a copy of parent whose Done channel is closed as soon as
    // parent.Done is closed, cancel is called, or timeout elapses. The new
    // Context's Deadline is the sooner of now+timeout and the parent's deadline, if
    // any. If the timer is still running, the cancel function releases its
    // resources.
    func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)

``WithValue`` はリクエストスコープの値を ``Context`` に関連付ける方法を提供します。

.. code-block:: go

    // WithValue returns a copy of parent whose Value method returns val for key.
    func WithValue(parent Context, key interface{}, val interface{}) Context

``Context`` の使用方法を知る最善の方法は、実際の例を使用することです。

例: GoogleのWeb検索
=========================================

「golang」を検索するクエリを `Google Web Search API <[https://developers.google.com/web-search/docs/>`_ に投げ、結果をレンダリングする ``/search?q=golang&timeout=1s`` のようなURLを扱うHTTPサーバの例を見てみましょう。``timeout`` パラメータはその期間が経過した後にリクエストをキャンセルするようにサーバーに指示します。

コードは以下の3つのパッケージに分かれます。

- `server <https://blog.golang.org/context/server/server.go>`_ はメイン関数と ``/search`` を扱うハンドラを提供します。
- `userip <https://blog.golang.org/context/userip/userip.go>`_ はリクエストからユーザーIPアドレスを抽出し、それを ``Context`` に関連付ける機能を提供します。
- `google <https://blog.golang.org/context/google/google.go>`_ はクエリをGoogleに送信するための ``検索`` 機能を提供します。

サーバープログラム
-----------------------------------------

`サーバー <https://blog.golang.org/context/server/server.go>`_ プログラムは ``golang`` の最初のいくつかのGoogle検索結果を提供することにより、``/search?q=golang`` などのリクエストを処理します。``handlesearch`` を ``/search`` エンドポイントに登録します。ハンドラは ``ctx`` という起点になる ``Context`` を作成し、ハンドラが戻ったときにキャンセルされるように調整します。リクエストに ``timeout`` のクエリパラメーターが含まれている場合、タイムアウトが経過するとコンテキストは自動的にキャンセルされます。

.. code-block:: go

    func handleSearch(w http.ResponseWriter, req *http.Request) {
        // ctx is the Context for this handler. Calling cancel closes the
        // ctx.Done channel, which is the cancellation signal for requests
        // started by this handler.
        var (
            ctx    context.Context
            cancel context.CancelFunc
        )
        timeout, err := time.ParseDuration(req.FormValue("timeout"))
        if err == nil {
            // The request has a timeout, so create a context that is
            // canceled automatically when the timeout expires.
            ctx, cancel = context.WithTimeout(context.Background(), timeout)
        } else {
            ctx, cancel = context.WithCancel(context.Background())
        }
        defer cancel() // Cancel ctx as soon as handleSearch returns.

ハンドラーは、リクエストからクエリを抽出し ``userip`` パッケージを呼び出してクライアントのIPアドレスを抽出します。クライアントのIPアドレスはバックエンドへのリクエストに必要であるため ``handleSearch`` はIPアドレスを ``ctx`` に付与します。

.. code-block:: go

        // Check the search query.
        query := req.FormValue("q")
        if query == "" {
            http.Error(w, "no query", http.StatusBadRequest)
            return
        }

        // Store the user IP in ctx for use by code in other packages.
        userIP, err := userip.FromRequest(req)
        if err != nil {
            http.Error(w, err.Error(), http.StatusBadRequest)
            return
        }
        ctx = userip.NewContext(ctx, userIP)

ハンドラは ``ctx`` と ``query`` を使用して ``google.Search`` を呼び出します。

.. code-block:: go

        // Run the Google search and print the results.
        start := time.Now()
        results, err := google.Search(ctx, query)
        elapsed := time.Since(start)

検索が完了すると、ハンドラは検索結果をレンダリングします。

.. code-block:: go

        if err := resultsTemplate.Execute(w, struct {
            Results          google.Results
            Timeout, Elapsed time.Duration
        }{
            Results: results,
            Timeout: timeout,
            Elapsed: elapsed,
        }); err != nil {
            log.Print(err)
            return
        }

useripパッケージ
-----------------------------------------

`userip <https://blog.golang.org/context/userip/userip.go>`_ パッケージは、リクエストからユーザーIPアドレスを抽出し、それを ``Context`` に関連付ける機能を提供します。``Context`` は、キーと値の両方が型 ``interface{}`` であるキーと値のマッピングを提供します。キーの型は等価性をサポートする必要があり、値は複数のゴルーチンが同時に使用しても安全でなければなりません。``userip`` などのパッケージは、このマッピングの詳細を隠し、特定の ``Context`` 値への厳密に型指定されたアクセスを提供します。

キーの衝突を避けるために、``userip`` はエクスポートされていない ``key`` 型を定義し、この型の値をコンテキストのキーとして使用します。

.. code-block:: go

    // The key type is unexported to prevent collisions with context keys defined in
    // other packages.
    type key int

    // userIPkey is the context key for the user IP address.  Its value of zero is
    // arbitrary.  If this package defined other context keys, they would have
    // different integer values.
    const userIPKey key = 0

``FromRequest`` は ``http.Request`` から ``userIP`` の値を抽出します。

.. code-block:: go

    func FromRequest(req *http.Request) (net.IP, error) {
        ip, _, err := net.SplitHostPort(req.RemoteAddr)
        if err != nil {
            return nil, fmt.Errorf("userip: %q is not IP:port", req.RemoteAddr)
        }

``NewContext`` は、指定された ``userIP`` 値を保持する新しい ``Context`` を返します。

.. code-block:: go

    func NewContext(ctx context.Context, userIP net.IP) context.Context {
        return context.WithValue(ctx, userIPKey, userIP)
    }

``FromContext`` は ``Context`` から ``userIP`` を抽出します。

.. code-block:: go

    func FromContext(ctx context.Context) (net.IP, bool) {
        // ctx.Value returns nil if ctx has no value for the key;
        // the net.IP type assertion returns ok=false for nil.
        userIP, ok := ctx.Value(userIPKey).(net.IP)
        return userIP, ok
    }

googleパッケージ
-----------------------------------------

`google.Search <https://blog.golang.org/context/google/google.go>`_ 関数は `Google Web Search API <https://developers.google.com/web-search/docs/>`_ へのHTTPリクエストを作成し、JSONエンコードされた結果を解析します。``Context`` パラメータ ``ctx`` を受け取り、リクエストの実行中に ``ctx.Done`` が閉じられるとすぐに戻ります。

Google Web Search APIリクエストには、クエリパラメータとして検索クエリとユーザーIPが含まれます。

.. code-block:: go

    func Search(ctx context.Context, query string) (Results, error) {
        // Prepare the Google Search API request.
        req, err := http.NewRequest("GET", "https://ajax.googleapis.com/ajax/services/search/web?v=1.0", nil)
        if err != nil {
            return nil, err
        }
        q := req.URL.Query()
        q.Set("q", query)

        // If ctx is carrying the user IP address, forward it to the server.
        // Google APIs use the user IP to distinguish server-initiated requests
        // from end-user requests.
        if userIP, ok := userip.FromContext(ctx); ok {
            q.Set("userip", userIP.String())
        }
        req.URL.RawQuery = q.Encode()

``Search`` では、ヘルパー関数 ``httpDo`` を使用してHTTPリクエストを発行し、リクエストまたはレスポンスの処理中に ``ctx.Done`` が閉じられた場合、キャンセルします。``Search`` は ``httpDo`` にクロージャーを渡し、HTTPレスポンスを処理します。

.. code-block:: go

        var results Results
        err = httpDo(ctx, req, func(resp *http.Response, err error) error {
            if err != nil {
                return err
            }
            defer resp.Body.Close()

            // Parse the JSON search result.
            // https://developers.google.com/web-search/docs/#fonje
            var data struct {
                ResponseData struct {
                    Results []struct {
                        TitleNoFormatting string
                        URL               string
                    }
                }
            }
            if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
                return err
            }
            for _, res := range data.ResponseData.Results {
                results = append(results, Result{Title: res.TitleNoFormatting, URL: res.URL})
            }
            return nil
        })
        // httpDo waits for the closure we provided to return, so it's safe to
        // read results here.
        return results, err

``httpDo``関数はHTTPリクエストを実行し、そのレスポンスを新しいゴルーチンで処理します。ゴルーチンが終了する前に ``ctx.Done`` が閉じられると、リクエストをキャンセルします。

.. code-block:: go

    func httpDo(ctx context.Context, req *http.Request, f func(*http.Response, error) error) error {
        // Run the HTTP request in a goroutine and pass the response to f.
        c := make(chan error, 1)
        req = req.WithContext(ctx)
        go func() { c <- f(http.DefaultClient.Do(req)) }()
        select {
        case <-ctx.Done():
            <-c // Wait for f to return.
            return ctx.Err()
        case err := <-c:
            return err
        }
    }

コンテキストに合わせたコードの適合
=========================================

多くのサーバーフレームワークは、リクエストスコープの値を運ぶためのパッケージと型を提供します。``Context`` インターフェースの新しい実装を定義して、既存のフレームワークを使用するコードと ``Context`` パラメーターを必要とするコードを橋渡しします。

例えば、Gorillaの `github.com/gorilla/context <http://www.gorillatoolkit.org/pkg/context>`_ パッケージを使用すると、ハンドラーはHTTPリクエストからキーと値のペアへのマッピングを提供することで、要求されたリクエストにデータを関連付けることができます。`gorilla.go <https://blog.golang.org/context/gorilla/gorilla.go>`_ では、ValueメソッドがGorillaパッケージの特定のHTTPリクエストに関連付けられた値を返す ``Context`` 実装を提供します。

他のパッケージは ``Context`` と同様のキャンセルサポートを提供しています。 例えば `Tomb <https://godoc.org/gopkg.in/tomb.v2>`_ は ``Dying`` チャネルを閉じることによりキャンセルを通知するKillメソッドを提供します。``Tomb`` は ``sync.WaitGroup`` と同様に、これらのゴルーチンが終了するのを待つメソッドも提供します。`tomb.go <https://blog.golang.org/context/tomb/tomb.go>`_ では、親 ``Context`` がキャンセルされるか、提供された ``Tomb`` が強制終了されるとキャンセルされる ``Context`` 実装を提供します。

結論
=========================================

Googleでは、リクエストとレスポンスの間で呼び出されるすべての関数に、最初の引数として ``Context`` パラメータを渡すことを要求しています。これによりGoのコードは多くの異なるチームで相互運用できます。タイムアウトとキャンセルを簡単に制御し、またセキュリティ資格情報などの重要な値がGoのプログラム上で適切に扱われるようにします。

``Context`` に用いて実装したいサーバーフレームワークは、フレームワークと ``Context`` パラメーターを期待するパッケージとの間を橋渡しする ``Context`` の実装を提供する必要があります。クライアントライブラリは、呼び出し元のコードから ``Context`` を受け入れます。リクエストスコープのデータとキャンセル用の共通インターフェースを構築することにより、``Context`` はパッケージ開発者がスケーラブルなサービスを作成するためのコードを簡単に共有できるようにします。

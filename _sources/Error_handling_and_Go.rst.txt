
Error handling and Go
=====================

概要
----

Go コードを書いたことがある人なら、組み込みの error 型を見たことがあるでしょう。Go のコードは、異常な状態を示すためにエラー値を使用します。例えば、os.Open 関数は、ファイルを開くのに失敗した場合、nil 以外のエラー値を返します。

.. code-block:: go

   func Open(name string) (file *File, err error)

以下のコードは os.Open を使ってファイルを開いています。エラーが発生した場合は log.Fatal を呼び出してエラーメッセージを表示して停止します。

.. code-block:: go

   f, err := os.Open("filename.ext")
   if err != nil {
       log.Fatal(err)
   }
   // do something with the open *File f

error 型について知っておくだけで、Go で多くのことができるようになりますが、この記事では、エラーについて詳しく見ていき、Go でのエラー処理のための良い方法について説明します。

error型
-------

error 型はインタフェース型です。エラーの変数は、それ自体は文字列を記述できる任意の値を表します。以下はインターフェイスの宣言です。

.. code-block:: go

   type error interface {
       Error() string
   }

error 型は、すべての組み込み型と同様に、ユニバースブロックで事前に宣言されています。

最も一般的に使用されるエラーの実装は、errors パッケージにある非公開の errorString 型です。

.. code-block:: go

   // errorString is a trivial implementation of error.
   type errorString struct {
       s string
   }

   func (e *errorString) Error() string {
       return e.s
   }

これらの値のいずれかを作成するには、errors.New 関数を使用します。これは文字列を受け取り、それを errors.errorString に変換してエラー値として返します。

.. code-block:: go

   // New returns an error that formats as the given text.
   func New(text string) error {
       return &errorString{text}
   }

ここでは、error.New を使用方法を説明します。

.. code-block:: go

   func Sqrt(f float64) (float64, error) {
       if f < 0 {
           return 0, errors.New("math: square root of negative number")
       }
       // implementation
   }

Sqrt に負の引数を渡す呼び出し元は、nil ではないエラー値を受け取ります（その具体的な表現は errors.errorString 値です）。呼び出し元は、エラーの Error メソッドを呼び出すか、エラー文字列("math: square root of....")を単に表示することができます。

.. code-block:: go

   f, err := Sqrt(-1)
   if err != nil {
       fmt.Println(err)
   }

`fmt <https://golang.org/pkg/fmt/>`_ パッケージは Error() string メソッドを呼び出すことでエラー値をフォーマットします。

コンテキストを要約するのはエラーの実装者の責任です。os.Open が返すエラーは "open /etc/passwd: permission denied" という形式で、"permission denied" だけではありません。私たちの Sqrt によって返されるエラーは、無効な引数に関する情報を欠いています。

その情報を追加するには、fmt パッケージの Errorf が便利です。これは、Printf の規則に従って文字列をフォーマットし、errors.New によって生成されたエラーとして返します。

.. code-block:: go

   if f < 0 {
       return 0, fmt.Errorf("math: square root of negative number %g", f)
   }

多くの場合、fmt.Errorf で十分ですが、エラーはインターフェースなので、任意のデータ構造をエラー値として使用して、呼び出し元がエラーの詳細を調べることができます。

例えば、仮に呼び出し側は Sqrt に渡された無効な引数を回復したいと思うかもしれません。これを可能にするには、errors.errorString を使用する代わりに新しく Error() を実装します。

.. code-block:: go

   type NegativeSqrtError float64

   func (f NegativeSqrtError) Error() string {
       return fmt.Sprintf("math: square root of negative number %g", float64(f))
   }

洗練された呼び出し元は、\ `型アサーション <https://golang.org/ref/spec#Type_assertions>`_\ を使用して NegativeSqrtError をチェックして特別に処理することができますが、エラーを fmt.Println や log.Fatal に渡すだけの呼び出し元では動作に変化はありません。

別の例として、\ `json <https://golang.org/pkg/encoding/json/>`_ パッケージでは、JSON blob を解析する際に構文エラーが発生した場合に json.Decode 関数が返す SyntaxError 型を指定しています。

.. code-block:: go

   type SyntaxError struct {
       msg    string // description of error
       Offset int64  // error occurred after reading Offset bytes
   }

   func (e *SyntaxError) Error() string { return e.msg }

Offset フィールドはエラーのデフォルトのフォーマットにはありませんが、呼び出し元はこれを使用してエラーメッセージにファイルや行の情報を追加することができます。

.. code-block:: go

   if err := dec.Decode(&val); err != nil {
       if serr, ok := err.(*json.SyntaxError); ok {
           line, col := findLine(f, serr.Offset)
           return fmt.Errorf("%s:%d:%d: %v", f.Name(), line, col, err)
       }
       return err
   }

(これは `Camlistore <https://perkeep.org/>`_ プロジェクトの\ `実際のコード <https://github.com/go4org/go4/blob/03efcb870d84809319ea509714dd6d19a1498483/jsonconfig/eval.go#L123-L135>`_\ を少し簡略化したものです)

error インターフェースは Error メソッドのみを必要とします。例えば、net パッケージは通常の慣習に従ってエラー型のエラーを返しますが、いくつかのエラー実装は net.Error インターフェースで定義された追加のメソッドを持っています。

.. code-block:: go

   package net

   type Error interface {
       error
       Timeout() bool   // Is the error a timeout?
       Temporary() bool // Is the error temporary?
   }

クライアントコードは net.Error を型アサーションでテストし、一時的なネットワークエラーと恒久的なエラーを区別することができます。例えば、ウェブクローラーは一時的なエラーに遭遇したときにスリープしてリトライし、そうでなければ処理をやめるかもしれません。

.. code-block:: go

   if nerr, ok := err.(net.Error); ok && nerr.Temporary() {
       time.Sleep(1e9)
       continue
   }
   if err != nil {
       log.Fatal(err)
   }

繰り返されるエラーのシンプル化
------------------------------

Go では、エラー処理が重要です。この言語の設計と規約は、エラーが発生した場合に明示的にチェックすることを奨励しています (他の言語では例外を投げたり、時にはキャッチしたりする慣習とは異なります)。このため、Go のコードが冗長になってしまう場合がありますが、幸いにも、繰り返しのエラー処理を最小限に抑えるために使用できるテクニックがいくつかあります。

データストアからレコードを取得し、テンプレートでフォーマットする HTTP ハンドラを持つ App Engine アプリケーションを考えてみましょう。

.. code-block:: go

   func init() {
       http.HandleFunc("/view", viewRecord)
   }

   func viewRecord(w http.ResponseWriter, r *http.Request) {
       c := appengine.NewContext(r)
       key := datastore.NewKey(c, "Record", r.FormValue("id"), 0, nil)
       record := new(Record)
       if err := datastore.Get(c, key, record); err != nil {
           http.Error(w, err.Error(), 500)
           return
       }
       if err := viewTemplate.Execute(w, record); err != nil {
           http.Error(w, err.Error(), 500)
       }
   }

この関数は datastore.Get 関数と viewTemplate の Execute メソッドから返されるエラーを処理します。どちらの場合も、HTTPステータスコード500（"Internal Server Error"）のシンプルなエラーメッセージをユーザーに表示します。これは管理しやすいコード量のように見えますが、さらにいくつかの HTTP ハンドラを追加すると、同じエラー処理コードのコピーがたくさん出てきてしまいます。

同じ記述を減らすために、エラーの戻り値を含む独自の HTTP appHandler タイプを定義することができます。

.. code-block:: go

   type appHandler func(http.ResponseWriter, *http.Request) error

そして、エラーを返す viewRecord 関数を変更することができます。

.. code-block:: go

   func viewRecord(w http.ResponseWriter, r *http.Request) error {
       c := appengine.NewContext(r)
       key := datastore.NewKey(c, "Record", r.FormValue("id"), 0, nil)
       record := new(Record)
       if err := datastore.Get(c, key, record); err != nil {
           return err
       }
       return viewTemplate.Execute(w, record)
   }

これは元のバージョンよりもシンプルですが、http パッケージはエラーを返す関数を理解していません。これを修正するには、http.Handler インターフェースの ServeHTTP メソッドを appHandler に実装します。

.. code-block:: go

   func (fn appHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
       if err := fn(w, r); err != nil {
           http.Error(w, err.Error(), 500)
       }
   }

ServeHTTP メソッドは appHandler 関数を呼び出し、返されたエラー (がもしあれば) をユーザに表示します。メソッドのレシーバである fn は関数であることに注意してください (Go はこれを実行できます！)。メソッドは、式 fn(w, r) でレシーバーを呼び出すことで関数を呼び出します。

現在、http パッケージで viewRecord を登録する際には、appHandlerはhttp.Handler（http.HandlerFunc ではない）なので、Handle 関数（HandleFuncではなく）を使用しています。

.. code-block:: go

   func init() {
       http.Handle("/view", appHandler(viewRecord))
   }

このような基本的なエラー処理の基盤が整備されていれば、よりユーザーフレンドリーなものにすることができます。単にエラー文字列を表示するのではなく、適切な HTTP ステータスコードを含むシンプルなエラーメッセージをユーザーに表示し、同時にデバッグ目的のために完全なエラーを App Engine の開発者コンソールにロギングする方が良いでしょう。

これを実現するために、エラーとその他のフィールドを含む appError 構造体を作成します。

.. code-block:: go

   type appError struct {
       Error   error
       Message string
       Code    int
   }

次に、*appError の値を返すように appHandler の型を変更します。

.. code-block:: go

   type appHandler func(http.ResponseWriter, *http.Request) *appError

(通常、\ `Go FAQ <https://golang.org/doc/faq#nil_error>`_\ で説明されている理由から、エラーではなくエラーの具体的なタイプを渡すのは間違いですが、ServeHTTP は値を見てその内容を使用する唯一の場所なので、ここではそれが正しいことです)。

そして、appHandler の ServeHTTP メソッドに appError のメッセージを正しい HTTP ステータスコードでユーザーに表示させ、エラー全体を開発者コンソールに記録させます。

.. code-block:: go

   func (fn appHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
       if e := fn(w, r); e != nil { // e is *appError, not os.Error.
           c := appengine.NewContext(r)
           c.Errorf("%v", e.Error)
           http.Error(w, e.Message, e.Code)
       }
   }

最後に、viewRecord を新しい関数のシグネチャに更新し、エラーが発生した場合にはより多くのコンテキストを返すようにしています。

.. code-block:: go

   func viewRecord(w http.ResponseWriter, r *http.Request) *appError {
       c := appengine.NewContext(r)
       key := datastore.NewKey(c, "Record", r.FormValue("id"), 0, nil)
       record := new(Record)
       if err := datastore.Get(c, key, record); err != nil {
           return &appError{err, "Record not found", 404}
       }
       if err := viewTemplate.Execute(w, record); err != nil {
           return &appError{err, "Can't display record", 500}
       }
       return nil
   }

このバージョンの viewRecord はオリジナルと同じ長さですが、それぞれの行が特定の意味を持ち、より親しみやすいユーザー体験を提供しています。

これで終わりではなく、アプリケーションのエラー処理をさらに改善することができます。いくつかのアイデアを紹介します。


* 
  エラーハンドラにきれいな HTML テンプレートを与えます。

* 
  ユーザーが管理者である場合を考えて、HTTP レスポンスにスタックトレースを書き込むことで、デバッグを容易にします。

* 
  デバッグを容易にするために、スタックトレースを保存する appError のコンストラクタ関数を書きます。

* 
  appHandler 内のパニックから回復し「重大なエラーが発生しました」とユーザーに通知しながら、エラーを「Critical」としてコンソールに記録します。 これは、プログラミングエラーによって引き起こされた不可解なエラーメッセージにユーザーがさらされないようにするための良い方法です。 詳細については、\ `Defer, Panic, and Recover <https://blog.golang.org/defer-panic-and-recover>`_ の記事を参照してください。

結論
----

適切なエラー処理は、優れたソフトウェアの必須要件です。 この投稿で説明されている手法を採用することで、より信頼性が高く簡潔な Go コードを記述できるようになります。

==============================================================
JSON and Go
==============================================================
25 Jan 2011
Tags: json, technical
Summary: How to generate and consume JSON-formatted data in Go.
OldURL: /json-and-go

Andrew Gerrand

----------

はじめに(Introduction)
===============================

JSON (JavaScript Object Notation) は、シンプルなデータ交換フォーマットです。構文的には JavaScript のオブジェクトやリストに似ています。Webバックエンドとブラウザ上で動作するJavaScriptプログラム間の通信に最も一般的に使用されていますが、他の多くの場所でも使用されています。そのホームページである `json.org <http://json.org>`_ は、標準の定義を非常に明確かつ簡潔に提供しています。

`json <https://golang.org/pkg/encoding/json/>`_ パッケージを使用すると、Go プログラムから簡単に JSON データを読み書きできます。

エンコード(Encoding)
===============================

JSONデータをエンコードするために `Marshal <https://golang.org/pkg/encoding/json/#Marshal>`_ 関数が使用できます。

.. code-block:: go

	func Marshal(v interface{}) ([]byte, error)

Goのデータ構造を ``Message`` とします。

.. code-block:: go

	type Message struct {
	    Name string
	    Body string
	    Time int64
	}

そして ``Message`` のインスタンスを生成しましょう。

.. code-block:: go

	m := Message{"Alice", "Hello", 1294706395881547000}

``json.Marshal`` を使用してJSONエンコードすることができます。

.. code-block:: go

	b, err := json.Marshal(m)

問題がなければ ``err`` は ``nil`` になり、``b`` はこのJSONデータを含む ``[]byte`` になります。

.. code-block:: go

	b == []byte(`{"Name":"Alice","Body":"Hello","Time":1294706395881547000}`)

有効なJSONとして表現できるデータ構造のみがエンコードされます。

  - JSON オブジェクトはキーとして文字列のみをサポートしています。Goのmap型をエンコードするには、``map[string]T`` (``T`` は json パッケージでサポートされている任意の Go の型) の形式でなければなりません。

  - チャネル型、複素数型、関数型はエンコードできません。

  - 循環するデータ構造はサポートされていません。``Marshal`` が無限ループに陥る原因となります。

  - ポインタは、そのポインタが指す値としてエンコードされます（ポインタが ``nil`` の場合は "null" となります）。

json パッケージは struct 型の公開されたフィールド (大文字で始まるもの) にのみアクセスできます。そのため、構造体の公開されているフィールドのみがJSONの結果として現れます。

デコード(Decoding)
===============================

JSONデータをデコードする場合、`Unmarshal <https://golang.org/pkg/encoding/json/#Unmarshal>`_ 関数を使います。

.. code-block:: go

	func Unmarshal(data []byte, v interface{}) error

まず、復号化されたデータを格納する変数を生成する必要があります。

.. code-block:: go

	var m Message

そして ``[]byte`` のJSONデータと ``m`` へのポインタを ``json.Unmarshal`` の引数として渡します。

.. code-block:: go

	err := json.Unmarshal(b, &m)

``b`` が ``m`` に合致する有効な JSON が含まれていれば、呼び出し後の ``err`` は ``nil`` となり、``b`` のデータは、変数 ``m`` の構造体に格納されます。

以下のようなものです。

.. code-block:: go

	m = Message{
	    Name: "Alice",
	    Body: "Hello",
	    Time: 1294706395881547000,
	}

``Unmarshal`` は、デコードされたデータを格納するフィールドをどのように特定するのですか? 指定された JSON キー "Foo" に対して、``Unmarshal`` は格納先の構造体のフィールドを以下の優先順位の高い順に見つけます。

  - "Foo" のタグを持つ公開されたフィールド（構造体のタグの詳細については `Go spec <https://golang.org/ref/spec#Struct_types>`_ を参照してください）。

  - "Foo" という名前のエクスポートされたフィールド

  - エクスポートされたフィールドの名前が "FOO" または "FOO" またはその他の大文字小文字を区別しない "Foo" にマッチするフィールド

JSONデータの構造がGo型と完全に一致しない場合はどうなりますか？

.. code-block:: go

	b := []byte(`{"Name":"Bob","Food":"Pickle"}`)
	var m Message
	err := json.Unmarshal(b, &m)

``Unmarshal`` は、格納先の型で見つけられるフィールドのみをデコードします。この場合、``m`` のNameフィールドのみがデコードされ、Foodフィールドは無視されます。この動作は、大きなJSON blobの中からいくつかの特定のフィールドだけを取り出したい場合に特に便利です。また、格納先の構造体内のエクスポートされていないフィールドは、``Unmarshal`` の影響を受けないことを意味します。

しかし、事前にJSONデータの構造がわからない場合はどうすればいいのでしょうか？

interface{} を使った汎用的なJSON(Generic JSON with interface{})
======================================================================

``interface{}`` は (空のインターフェイス)型は、メソッドを持たないインターフェースの型として表現されます。すべての Go の型は少なくとも 0 つのメソッドを実装しており、したがって空のインターフェイスを満たしています。

空のインターフェイスは、汎用的な値を格納するコンテナとして機能します。

.. code-block:: go

	var i interface{}
	i = "a string"
	i = 2011
	i = 2.777

型アサーションして、``interface{}`` 型の具象型にアクセスします。

.. code-block:: go

	r := i.(float64)
	fmt.Println("the circle's area", math.Pi*r*r)

あるいは、実際の型が不明な場合は、型スイッチによって型を特定できます。

.. code-block:: go

	switch v := i.(type) {
	case int:
	    fmt.Println("twice i is", v*2)
	case float64:
	    fmt.Println("the reciprocal of i is", 1/v)
	case string:
	    h := len(v) / 2
	    fmt.Println("i swapped by halves is", v[h:]+v[:h])
	default:
	    // i isn't one of the types above
	}

jsonパッケージは、任意のJSONオブジェクトや配列を格納するために ``map[string]interface{}`` や ``[]interface{}`` 値を使用します。デフォルトの具体的なGoの型は以下の通りです。

  - ``bool`` は JSON の booleans

  - ``float64`` は JSON の numbers

  - ``string`` は JSON の strings

  - ``nil`` null のJSON

任意のデータのデコード(Decoding arbitrary data)
==============================================================

変数 ``b`` に格納されているこのJSONデータを考えてみましょう。

.. code-block:: go

	b := []byte(`{"Name":"Wednesday","Age":6,"Parents":["Gomez","Morticia"]}`)

このデータの構造を知らなくても、``Unmarshal`` を使って ``interface{}`` 型の値にデコードすることができます。

.. code-block:: go

	var f interface{}
	err := json.Unmarshal(b, &f)

この時点で ``f`` の Go 値は、キーが文字列で、その値自体が空のインターフェイス値として格納されている map になります。

.. code-block:: go

	f = map[string]interface{}{
	    "Name": "Wednesday",
	    "Age":  6,
	    "Parents": []interface{}{
	        "Gomez",
	        "Morticia",
	    },
	}

``f`` の実際の型である ``map[string]interface{}`` 型に型アサーションして、データにアクセスします。

.. code-block:: go

	m := f.(map[string]interface{})

We can then iterate through the map with a range statement and use a type
switch to access its values as their concrete types:
次に、range 文を使用して map を反復処理し、型スイッチを使用して具象型にアクセスすることができます。

.. code-block:: go

	for k, v := range m {
	    switch vv := v.(type) {
	    case string:
	        fmt.Println(k, "is string", vv)
	    case float64:
	        fmt.Println(k, "is float64", vv)
	    case []interface{}:
	        fmt.Println(k, "is an array:")
	        for i, u := range vv {
	            fmt.Println(i, u)
	        }
	    default:
	        fmt.Println(k, "is of a type I don't know how to handle")
	    }
	}

このようにして、未知のJSONデータを扱うことができます。型の安全性の利点を享受することができます。

参照型(Reference Types)
===============================

.. note::

	``Reference Types`` というGoの型の呼び方は廃止されています。

先ほどの例のデータを格納するGo型を定義してみましょう。

.. code-block:: go

	type FamilyMember struct {
	    Name    string
	    Age     int
	    Parents []string
	}

	var m FamilyMember
	err := json.Unmarshal(b, &m)

そのデータを ``FamilyMember`` の値にアンマーシャルすると期待通りに動作しますが、よく見ると驚くべきことが起きていることがわかります。var ステートメントで ``FamilyMember`` 構造体を割り当て、その値へのポインタを ``Unmarshal`` に提供しましたが、その時点では ``Parents`` フィールドは ``nil`` のスライス値でした。``Parents`` フィールドを入力するために、``Unmarshal`` は裏で新しいスライスを割り当てました。これは、サポートされている参照型(ポインタ、スライス、およびマップ)で ``Unmarshal`` がどのように動作するかの典型的な例です。

このデータ構造へのアンマーシャリングを考えてみます。

.. code-block:: go

	type Foo struct {
	    Bar *Bar
	}

JSON オブジェクトに ``Bar`` フィールドがあれば、``Unmarshal`` は新しい ``Bar`` を割り当て、それを埋めます。そうでない場合は、``Bar`` は ``nil`` ポインタとして残されます。

このことから便利なパターンが生まれます: いくつかの異なるメッセージタイプを受信するアプリケーションがある場合、"レシーバ" 構造体を以下のように

.. code-block:: go

	type IncomingMessage struct {
	    Cmd *Command
	    Msg *Message
	}

と送信側は、通信したいメッセージの型に応じて、トップレベル JSON オブジェクトの ``Cmd`` フィールドおよび/または ``Msg`` フィールドを入力することができます。``Unmarshal`` は、JSONを ``IncomingMessage`` 構造体にデコードする際に、JSONデータに存在するデータ構造体のみを割り当てます。どのメッセージを処理するかを知るために、プログラマーは ``Cmd`` と ``Msg`` のどちらかが ``nil`` ではないことをテストするだけでよいのです。

ストリーミングにおけるエンコードとデコード(Streaming Encoders and Decoders)
==================================================================================

json パッケージは、JSON データのストリーミングな読み書きにおける共通的な操作をサポートするための ``Decoder`` と ``Encoder`` 型を提供します。``NewDecoder`` と ``NewEncoder`` 関数は `io.Reader <https://golang.org/pkg/io/#Reader>`_ と `io.Writer <https://golang.org/pkg/io/#Writer>`_ のインターフェイス型をラップします。

.. code-block:: go

	func NewDecoder(r io.Reader) *Decoder
	func NewEncoder(w io.Writer) *Encoder

ここでは、標準入力から一連のJSONオブジェクトを読み込み、各オブジェクトからNameフィールドを除いたすべてのオブジェクトを削除し、標準出力に書き込むプログラムの例を示します。

.. code-block:: go

	package main

	import (
	    "encoding/json"
	    "log"
	    "os"
	)

	func main() {
	    dec := json.NewDecoder(os.Stdin)
	    enc := json.NewEncoder(os.Stdout)
	    for {
	        var v map[string]interface{}
	        if err := dec.Decode(&v); err != nil {
	            log.Println(err)
	            return
	        }
	        for k := range v {
	            if k != "Name" {
	                delete(v, k)
	            }
	        }
	        if err := enc.Encode(&v); err != nil {
	            log.Println(err)
	        }
	    }
	}

ReadersとWritersが汎用的なインターフェースのため、``Encoder`` と ``Decoder`` の型は、HTTP のコネクション、WebSocket、またはファイルへの読み書きなど、幅広い場面で使用することができます。

リファレンス
==============================================================

より詳しい情報は `json package documentation <https://golang.org/pkg/encoding/json/>`_ を確認ください。jsonの使用例については、`jsonrpc package <https://golang.org/pkg/net/rpc/jsonrpc/>`_ のソースファイルを参照してください。

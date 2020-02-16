==============================================================
Arrays, slices (and strings): The mechanics of 'append'
==============================================================

Arrays, slices (and strings): The mechanics of 'append'
26 Sep 2013
Tags: array, slice, string, copy, append

Rob Pike

----------

概要
===============================

手続き型プログラミング言語の最も一般的な機能の1つは、配列の概念です。配列は単純なもののように見えますが、言語に追加する際に検討しなければならない質問がたくさんあります。

- 固定長か可変長か
- 大きさは型の一部かどうか
- 多次元配列がどのように見えるか
- 空配列はどのような意味を持つか

これらの質問に対する答えは、配列が言語の単なる特徴であるか、その言語設計の中核部分であるかどうかに影響します。

Goの初期の開発では、設計が正しく感じられるまでにこれらの質問に対する答えを決定するのに約1年かかりました。重要なステップは、柔軟性のある拡張可能なデータ構造を提供するために固定サイズの *配列* に基づいて構築された *スライス* の導入でした。しかし、今日に至るまで、Goを初めて使用するプログラマーは、スライスの動作方法につまずくことがよくあります。これは、おそらく他の言語の経験による先入観によるためです。

この投稿では、上記のような混乱を解消することを試みます。このために、組み込み関数の ``append`` がどのように機能するか、なぜ機能するのかを説明します。

配列
===============================

配列はGoの重要な構成要素ですが、建物の基盤のように、より可視的なコンポーネントの下に隠れていることがよくあります。スライスのより興味深く、強力で、目立つアイデアに移る前に、それらについて簡単に話さなければなりません。

配列のサイズはその型の一部であり、表現力を制限するため、配列はGoプログラムではあまり見られません。

以下のように宣言します。

.. code-block:: go

	var buffer [256]byte

256バイトを保持する変数 ``buffer`` を宣言します。``buffer`` の型には、そのサイズ ``[256]byte`` が含まれます。 512バイトの配列は、異なる型 ``[512]byte`` になります。

配列に関連付けられているデータは、要素の配列です。概略的には、メモリ内のバッファは次のようになります。

.. code-block:: go

	buffer: byte byte byte ... 256 times ... byte byte byte

つまり、変数は256バイトのデータを保持し、それ以外は何も保持しません。 ``buffer[255]`` を介して、おなじみのインデックス構文 ``buffer[0]`` 、``buffer[1]`` などでその要素にアクセスできます。(インデックス範囲0〜255は256要素をカバーします。)この範囲外の値でバッファをインデックス付けしようとすると、プログラムがクラッシュします。

``len`` と呼ばれる組み込み関数があり、配列またはスライスの要素数と、他のいくつかのデータ型の要素数を返します。配列の場合 ``len`` が返すものは明らかです。この例では ``len(buffer)`` は固定値256を返します。

配列が使われる用途はあります。例えば、変換行列の適切に表現します。しかしGoで最も一般的な目的は、スライスのためのストレージを保持することです。

スライス: スライスヘッダー
===============================

スライスはよく使われますが、それらをうまく使用するには、スライスが何であり、何をするのかを正確に理解する必要があります。

スライスは、スライス変数自体とは別に保存される配列の連続したセクションを記述するデータ構造です。 *スライスは配列ではありません* 。スライスは配列の一部を *表します* 。

前のセクションの ``buffer`` 配列変数を使用して、配列を *スライスする* ことにより、要素が100～150(正確には100〜149を含む)を表すスライスを作成できます。

.. code-block:: go

	var slice []byte = buffer[100:150]

このスニペットでは、完全な変数宣言を明示的に使用しました。変数 ``slice``の型は ``[]byte`` で「バイトのスライス」と発音され ``buffer`` と呼ばれる配列から、要素100(含む)から150(含まない)をスライスすることによって初期化されます。より慣用的な構文は、初期化式によって設定される型を除きます。

.. code-block:: go

	var slice = buffer[100:150]

関数内では、以下の簡略した宣言を使用できます。

.. code-block:: go

	slice := buffer[100:150]

上記の slice 変数とは正確には何ですか？完全な話ではありませんが、現時点では、スライスを、長さと配列の要素へのポインタという2つの要素を持つ小さなデータ構造と考えてください。背後で次のように構築されていると考えることができます。

.. code-block:: go

	type sliceHeader struct {
		Length        int
		ZerothElement *byte
	}

	slice := sliceHeader{
		Length:        50,
		ZerothElement: &buffer[100],
	}

もちろん、これは単なる例です。このスニペットは ``sliceHeader`` と記しているにもかかわらず、構造体はプログラマに公開されていません。要素ポインタの型は要素の型に依存しますが、これは一般的な考え方です。

これまで、配列に対してスライス操作を使用しましたが、次のようにスライスをスライスすることもできます。

.. code-block:: go

	slice2 := slice[5:10]

前と同じように、この操作は新しいスライスを作成します。この場合、元のスライスの要素5～9(両端を含む)で、元の配列の要素105〜109を意味します。 ``slice2`` 変数の基礎となる ``sliceHeader`` 構造体は次のようになります。

.. code-block:: go

	slice2 := sliceHeader{
		Length:        5,
		ZerothElement: &buffer[105],
	}

このヘッダーは、 ``buffer`` 変数に格納されている同じ基となる配列を引き続き指していることに注意してください。

スライスを *再スライス* することもできます。つまり、スライスをスライスして、元のスライス構造に結果を保存します。

.. code-block:: go

	slice = slice[5:10]

``slice`` 変数の ``sliceHeader`` 構造は ``slice2`` 変数の場合と同じように見えます。スライスを切り捨てるなど、頻繁に使用される再スライスが表示されます。次のステートメントは、スライスの最初と最後の要素を削除します。

.. code-block:: go

	slice = slice[1:len(slice)-1]

[演習：この割り当て後 ``sliceHeader`` 構造体がどのように見えるかを記述します。]

経験豊富なGoプログラマーが「スライスヘッダー」について語るのをよく耳にします。これは実際にスライス変数に格納されているからです。たとえば、 `bytes.IndexRune <https://golang.org/pkg/bytes/#IndexRune>`_ など、スライスを引数として取る関数を呼び出すと、そのヘッダーが関数に渡されます。この呼び出しでは、

.. code-block:: go

	slashPos := bytes.IndexRune(slice, '/')

``IndexRune`` 関数に渡される ``slice`` 引数は、実際には「スライスヘッダー」です。

スライスヘッダーにはもう1つのデータ項目がありますが、これについては後で説明します。最初に、スライスを使用してプログラミングするときにスライスヘッダーの存在が何を意味するかを見てみましょう。

スライスを関数に渡す
===============================

スライスにポインターが含まれていても、それ自体が値であることを理解することが重要です。背後では、ポインターと長さを保持する構造体の値です。構造体へのポインタでは *ありません* 。

これは重要です。

前の例で ``IndexRune`` を呼び出したときに、スライスヘッダーの *コピー* が渡されました。この動作には重要な影響があります。

以下のシンプルな関数について考えてみます。

.. code-block:: go

	func AddOneToEachElement(slice []byte) {
		for i := range slice {
			slice[i]++
		}
	}

その名前が示すとおり、スライスのインデックスを反復処理し(forの ``range`` ループを使用)、要素をインクリメントします。

動かしてみましょう。

.. code-block:: go

	func main() {
		slice := buffer[10:20]
		for i := 0; i < len(slice); i++ {
			slice[i] = byte(i)
		}
		fmt.Println("before", slice)
		AddOneToEachElement(slice)
		fmt.Println("after", slice)
	}

(気になる場合は、これらの実行可能なスニペットを編集および再実行できます。)

スライス *ヘッダー* は値で渡されますが、ヘッダーには配列の要素へのポインターが含まれるため、元のスライスヘッダーと関数に渡されるヘッダーのコピーの両方が同じ配列を記述します。したがって、関数が戻ると、変更された要素は元のスライス変数を通して見ることができます。

この例のように、関数の引数は実際にはコピーです。

.. code-block:: go

	func SubtractOneFromLength(slice []byte) []byte {
		slice = slice[0 : len(slice)-1]
		return slice
	}

	func main() {
		fmt.Println("Before: len(slice) =", len(slice))
		newSlice := SubtractOneFromLength(slice)
		fmt.Println("After:  len(slice) =", len(slice))
		fmt.Println("After:  len(newSlice) =", len(newSlice))
	}

ここでは、関数によってスライス引数の *内容* を変更できますが、 *ヘッダー* は変更できないことがわかります。``slice`` 変数に格納されている長さは、関数の呼び出しによって変更されません。これは、関数には元ではなくスライスヘッダーのコピーが渡されるためです。したがって、ヘッダーを変更する関数を作成する場合は、ここで行ったように、結果パラメーターとして返す必要があります。``slice`` 変数は変更されていませんが、返される値の長さは新しいため ``newSlice`` に保存されます。

スライスへのポインタ: メソッドのレシーバ
===========================================

関数にスライスヘッダーを変更させる別の方法は、関数にポインターを渡すことです。これを行う前の例を少し変えた例を示します。

.. code-block:: go

	func PtrSubtractOneFromLength(slicePtr *[]byte) {
		slice := *slicePtr
		*slicePtr = slice[0 : len(slice)-1]
	}

	func main() {
		fmt.Println("Before: len(slice) =", len(slice))
		PtrSubtractOneFromLength(&slice)
		fmt.Println("After:  len(slice) =", len(slice))
	}

上記の例では、特に余計な間接参照のレベルを扱う場合(一時的な変数が役たちます)ぎこちないように見えます。しかしスライスへのポインターを見かける一般的なケースが1つあります。スライスを変更するメソッドにポインターレシーバーを使用するのが慣用的です。

.. todo::

	訳がわからない

	It seems clumsy in that example, especially dealing with the extra level of indirection (a temporary variable helps)

	上記の例では、特に余計な間接参照のレベルを扱う場合(一時的な変数が役たちます)ぎこちないように見えます。

最後のスラッシュで切り捨てるメソッドをスライスに持たせたいとしましょう。次のように書くことができます。

.. code-block:: go

	type path []byte

	func (p *path) TruncateAtFinalSlash() {
		i := bytes.LastIndex(*p, []byte("/"))
		if i >= 0 {
			*p = (*p)[0:i]
		}
	}

	func main() {
		pathName := path("/usr/bin/tso") // Conversion from string to path.
		pathName.TruncateAtFinalSlash()
		fmt.Printf("%s\n", pathName)
	}

この例を実行すると、正しく動作し、呼び出し元のスライスを更新することがわかります。

[演習：レシーバーの型をポインターではなく値に変更して、再度実行します。何が起こるかを説明してください。]

.. note:: 

	[訳注] レシーバの型を値にすると、呼び出したときにスライスのコピーが渡されます。関数内でスライスを切り出して別のスライスを作成したとしても ``pathName`` には影響がありません。

一方、パス内のASCII文字を大文字にする(英語以外の名前を無視することで) ``path`` のメソッドを作成する場合、値のレシーバーは同じ基になる配列を指すため、メソッドは値になる可能性があります。

.. code-block:: go

	type path []byte

	func (p path) ToUpper() {
		for i, b := range p {
			if 'a' <= b && b <= 'z' {
				p[i] = b + 'A' - 'a'
			}
		}
	}

	func main() {
		pathName := path("/usr/bin/tso")
		pathName.ToUpper()
		fmt.Printf("%s\n", pathName)
	}

ここで ``ToUpper`` メソッドは、``for`` ``range`` 構造内の2つの変数を使用して、インデックスとスライス要素をキャプチャします。この形式のループは、内で複数回 ``p[i]`` を書き込むことを避けます。

[演習：``ToUpper`` メソッドを変換してポインターレシーバーを使用し、その動作が変化するかどうかを確認します。]

.. note:: 

	レシーバに値を渡してもポインタを渡しても動作に変化はありません。

	.. code-block:: go

		type path []byte

		func (p *path) ToUpper() {
			for i, b := range *p {
				if 'a' <= b && b <= 'z' {
					(*p)[i] = b + 'A' - 'a'
				}
			}
		}

		func main() {
			pathName := path("/usr/bin/tso")
			pathName.ToUpper()
			fmt.Printf("%s\n", pathName)
		}
	
	::

		/USR/BIN/TSO

		Program exited.

[高度な演習：``ToUpper`` メソッドを変換して、ASCIIだけでなくUnicode文字を処理します。]

容量(Capacity)
===============================

``ints`` の引数スライスを1つの要素だけ拡張する次の関数を見てください。

.. code-block:: go

	func Extend(slice []int, element int) []int {
		n := len(slice)
		slice = slice[0 : n+1]
		slice[n] = element
		return slice
	}

(変更されたスライスを返す必要があるのはなぜですか？)それを実行します：

(Why does it need to return the modified slice?) Now run it:

.. code-block:: go

	func main() {
		var iBuffer [10]int
		slice := iBuffer[0:0]
		for i := 0; i < 20; i++ {
			slice = Extend(slice, i)
			fmt.Println(slice)
		}
	}

スライスがどのように成長するかを確認してください...。最後までは成長しません。

スライスヘッダーの3番目のコンポーネントである容量について説明します。配列へのポインタと長さに加えて、スライスヘッダーにはその *容量* も格納されます。

.. code-block:: go

	type sliceHeader struct {
		Length        int
		Capacity      int
		ZerothElement *byte
	}

``Capacity`` フィールドには、基になる配列に実際にどのくらいのスペースがあるかが記録されます。これは ``Length`` が到達できる最大値です。スライスをその容量を超えて拡大しようとすると、配列の制限を超えてパニックを引き起こします。

以下のように、サンプルスライスが作成された後

.. code-block:: go

	slice := iBuffer[0:0]

そのヘッダーは次のようになります。

.. code-block:: go

	slice := sliceHeader{
		Length:        0,
		Capacity:      10,
		ZerothElement: &iBuffer[0],
	}

``Capacity`` フィールドは、基となる配列の長さから、スライスの最初の要素の配列のインデックス(この場合は0)を引いた値に等しくなります。スライスの容量を確認する場合は、組み込みの関数の ``cap`` を使用します。

.. code-block:: go

	if cap(slice) == len(slice) {
		fmt.Println("slice is full!")
	}

Make
===============================

スライスをその容量を超えて成長させたい場合はどうなりますか？できません！定義上、容量は成長の限界です。ただし、新しい配列を割り当て、データをコピーし、スライスを変更して新しい配列を記述することにより、同等の結果を得ることができます。

割り当てから始めましょう。組み込み関数 ``new`` を使用してより大きな配列を割り当て、結果をスライスできますが、代わりに ``make`` 組み込み関数を使用する方が簡単です。新しい配列を割り当て、それを記述するスライスヘッダーを一度に作成します。``make`` 関数は、3つの引数を取ります。スライスの型、その初期の長さ、およびその容量(スライスデータを保持するために ``make`` が割り当てる配列の長さ)です。この呼び出しは、実行するとわかるように、長さ10のスライスを作成し、さらに5つ(15-10)分のスペースを確保します。

.. code-block:: go

		slice := make([]int, 10, 15)
		fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))

以下のスニペットは ``int`` スライスの容量を2倍にしますが、長さは同じままです。

.. code-block:: go

		slice := make([]int, 10, 15)
		fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
		newSlice := make([]int, len(slice), 2*cap(slice))
		for i := range slice {
			newSlice[i] = slice[i]
		}
		slice = newSlice
		fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))

このコードを実行した後、スライスは、別の再割り当てを必要とする前に拡大する余地がはるかにあります。

スライスを作成するとき、長さと容量が同じになることがよくあります。``make`` 組み込み関数には、この一般的なケースの省略形があります。容量はデフォルトで長さ引数と同じ値が設定されているため、省略して、両方を同じ値に設定できます。

.. code-block:: go

	gophers := make([]Gopher, 10)

``gophers`` スライスの長さと容量の両方が10に設定されています。

Copy
===============================

前のセクションでスライスの容量を2倍にしたとき、古いデータを新しいスライスにコピーするループを作成しました。Goには、これを簡単にするための組み込み関数 ``copy`` があります。引数は2つのスライスであり、データを右側の引数から左側の引数にコピーします。コピーを使用するように書き直した例を次に示します。

.. code-block:: go

		newSlice := make([]int, len(slice), 2*cap(slice))
		copy(newSlice, slice)

``copy`` 関数はスマートです。両方の引数の長さに注意を払いながら、できることだけをコピーします。つまり、コピーする要素の数は、2つのスライスの長さの最小値です。これにより、実装が少しだけ簡略化されます。また ``copy`` は、コピーした要素の数である整数値を返しますが、常にチェックする必要はありません。

``copy`` 関数は、コピー元とコピー先の要素が重なるときにも適切に機能します。つまり、単一のスライス内でアイテムをシフトするために使用できます。``copy`` を使用して、スライスの中央に値を挿入する方法を次に示します。

.. code-block:: go

	// Insert inserts the value into the slice at the specified index,
	// which must be in range.
	// The slice must have room for the new element.
	func Insert(slice []int, index, value int) []int {
		// Grow the slice by one element.
		slice = slice[0 : len(slice)+1]
		// Use copy to move the upper part of the slice out of the way and open a hole.
		copy(slice[index+1:], slice[index:])
		// Store the new value.
		slice[index] = value
		// Return the result.
		return slice
	}

この関数には、注意すべき点がいくつかあります。もちろん、最初に、長さが変更されているため、更新されたスライスを返す必要があります。第二に、便利な略記法を使用します。以下は

.. code-block:: go

	slice[i:]

とまったく同じ意味です。

.. code-block:: go

	slice[i:len(slice)]

また、このトリックはまだ使用していませんが、スライス式の最初の要素も省略できます。デフォルトは0です。よって

.. code-block:: go

	slice[:]

スライス自体を意味するだけで、配列をスライスするときに便利です。この式は「配列のすべての要素を記述するスライス」と言う最も簡単な方法です。

.. code-block:: go

	array[:]

これで作業は完了です。``Insert`` 関数を実行しましょう。

.. code-block:: go

		slice := make([]int, 10, 20) // Note capacity > length: room to add element.
		for i := range slice {
			slice[i] = i
		}
		fmt.Println(slice)
		slice = Insert(slice, 5, 99)
		fmt.Println(slice)

.. note:: 

	[訳注]: 以下の結果を得ることができます。
	
	::

		[0 1 2 3 4 5 6 7 8 9]
		[0 1 2 3 4 99 5 6 7 8 9]

		Program exited.

Append: 例
===============================

数セクション前に、1つの要素でスライスを拡張する ``Extend`` 関数を作成しました。 ただし、スライスの容量が小さすぎると機能がクラッシュするため、バグがありました。(``Insert`` の例にも同じ問題があります。)これを修正するための準備が整ったので、整数スライス用の ``Extend`` の堅牢な実装を作成しましょう。

.. code-block:: go

	func Extend(slice []int, element int) []int {
		n := len(slice)
		if n == cap(slice) {
			// Slice is full; must grow.
			// We double its size and add 1, so if the size is zero we still grow.
			newSlice := make([]int, len(slice), 2*len(slice)+1)
			copy(newSlice, slice)
			slice = newSlice
		}
		slice = slice[0 : n+1]
		slice[n] = element
		return slice
	}

この場合、スライスを返すことが特に重要です。スライスを再割り当てすると、結果のスライスが完全に異なる配列を記述するためです。スライスがいっぱいになったときに何が起こるかを示すための小さなスニペットを次に示します。

.. code-block:: go

		slice := make([]int, 0, 5)
		for i := 0; i < 10; i++ {
			slice = Extend(slice, i)
			fmt.Printf("len=%d cap=%d slice=%v\n", len(slice), cap(slice), slice)
			fmt.Println("address of 0th element:", &slice[0])
		}

.. note:: 

	[訳注]: 以下の結果を得ることができます。
	
	::

		len=1 cap=5 slice=[0]
		address of 0th element: 0x456000
		len=2 cap=5 slice=[0 1]
		address of 0th element: 0x456000
		len=3 cap=5 slice=[0 1 2]
		address of 0th element: 0x456000
		len=4 cap=5 slice=[0 1 2 3]
		address of 0th element: 0x456000
		len=5 cap=5 slice=[0 1 2 3 4]
		address of 0th element: 0x456000
		len=6 cap=11 slice=[0 1 2 3 4 5]
		address of 0th element: 0x454030
		len=7 cap=11 slice=[0 1 2 3 4 5 6]
		address of 0th element: 0x454030
		len=8 cap=11 slice=[0 1 2 3 4 5 6 7]
		address of 0th element: 0x454030
		len=9 cap=11 slice=[0 1 2 3 4 5 6 7 8]
		address of 0th element: 0x454030
		len=10 cap=11 slice=[0 1 2 3 4 5 6 7 8 9]
		address of 0th element: 0x454030

		Program exited.

サイズ5の初期配列がいっぱいになると、再割り当てに注意してください。新しい配列が割り当てられると、0番目の要素の容量とアドレスが変更されます。

堅牢な ``Extend`` 関数をガイドとして使用すると、スライスを複数の要素で拡張できるさらに優れた関数を作成できます。 これを行うには、関数の呼び出し時に関数の引数のリストをスライスに変換するGoの機能を使用します。つまり、Goの可変機能機能を使用します。

関数 ``Append`` を呼び出しましょう。最初のバージョンでは ``Extend`` を繰り返し呼び出すだけで、可変引数関数のメカニズムが明確になります。追加したシグネチャは次のとおりです。

.. code-block:: go

	func Append(slice []int, items ...int) []int

つまり ``Append`` は1つの引数(スライス)の後に0個以上の ``int`` の引数が続くということです。これらの引数は、次のように ``Append`` の実装に関する限り、``int`` のスライスです。

.. code-block:: go

	// Append appends the items to the slice.
	// First version: just loop calling Extend.
	func Append(slice []int, items ...int) []int {
		for _, item := range items {
			slice = Extend(slice, item)
		}
		return slice
	}

``for`` ``range`` のループが ``items`` 引数の要素を反復処理していることに注目してください。これは型 ``[]int`` を暗黙的に示しています。また、ループ内のインデックスを破棄するために空白の識別子 ``_`` を使用していることに注意してください。この場合はインデックスは必要ありません。

Try it:

.. code-block:: go

		slice := []int{0, 1, 2, 3, 4}
		fmt.Println(slice)
		slice = Append(slice, 5, 6, 7, 8)
		fmt.Println(slice)

この例のもう1つの新しいテクニックは、複合リテラルを記述することによってスライスを初期化することです。複合リテラルは、スライスのタイプとそれに続く括弧で囲まれた要素で構成されます。

.. code-block:: go

		slice := []int{0, 1, 2, 3, 4}

``Append`` 機能は、別の理由で興味深いものです。要素を追加できるだけでなく、呼び出し側で ``...`` 表記を使用して、スライスを引数に「展開」することにより、2番目のスライス全体を追加できます。

.. code-block:: go

		slice1 := []int{0, 1, 2, 3, 4}
		slice2 := []int{55, 66, 77}
		fmt.Println(slice1)
		slice1 = Append(slice1, slice2...) // The '...' is essential!
		fmt.Println(slice1)

もちろん ``Extend`` の内部に基づいて複数回割り当てることで、``Append`` をより効率的にすることができます。

.. code-block:: go

	// Append appends the elements to the slice.
	// Efficient version.
	func Append(slice []int, elements ...int) []int {
		n := len(slice)
		total := len(slice) + len(elements)
		if total > cap(slice) {
			// Reallocate. Grow to 1.5 times the new size, so we can still grow.
			newSize := total*3/2 + 1
			newSlice := make([]int, total, newSize)
			copy(newSlice, slice)
			slice = newSlice
		}
		slice = slice[:total]
		copy(slice[n:], elements)
		return slice
	}

ここで、``copy`` を2回使用する方法に注意してください。1回はスライスデータを新しく割り当てられたメモリに移動し、その後、追加するデータを古いデータの最後にコピーします。

以下を試してみてください;動作は以前と同じです。

.. code-block:: go

		slice1 := []int{0, 1, 2, 3, 4}
		slice2 := []int{55, 66, 77}
		fmt.Println(slice1)
		slice1 = Append(slice1, slice2...) // The '...' is essential!
		fmt.Println(slice1)

Append: 組み込み関数
===============================

それで、``append`` 組み込み関数の設計の動機に到達します。``Append`` の例と同等の効率で正確に機能しますが、どのスライス型でも機能します。

Goの弱点は、ジェネリック型の操作を実行時に提供する必要があることです。いつか変わる可能性がありますが、現時点では、スライスの操作を簡単にするために、Goには組み込みの汎用 ``append`` 関数が用意されています。``int`` スライスバージョンと同じように機能しますが **すべて** のスライス型に対して機能します。

スライスヘッダーは ``append`` の呼び出しによって常に更新されるため、呼び出し後に返されたスライスを保存する必要があることに注意してください。実際、コンパイラーは結果を保存せずに ``append`` を呼び出すことはできません。

以下は、printステートメントと混ざったワンライナーです。それらを試して編集し、探索してください：

.. code-block:: go

		// Create a couple of starter slices.
		slice := []int{1, 2, 3}
		slice2 := []int{55, 66, 77}
		fmt.Println("Start slice: ", slice)
		fmt.Println("Start slice2:", slice2)

		// Add an item to a slice.
		slice = append(slice, 4)
		fmt.Println("Add one item:", slice)

		// Add one slice to another.
		slice = append(slice, slice2...)
		fmt.Println("Add one slice:", slice)

		// Make a copy of a slice (of int).
		slice3 := append([]int(nil), slice...)
		fmt.Println("Copy a slice:", slice3)

		// Copy a slice to the end of itself.
		fmt.Println("Before append to self:", slice)
		slice = append(slice, slice...)
		fmt.Println("After append to self:", slice)

.. note:: 

	[訳注]: 以下の結果を得ることができます。
	
	::

		Start slice:  [1 2 3]
		Start slice2: [55 66 77]
		Add one item: [1 2 3 4]
		Add one slice: [1 2 3 4 55 66 77]
		Copy a slice: [1 2 3 4 55 66 77]
		Before append to self: [1 2 3 4 55 66 77]
		After append to self: [1 2 3 4 55 66 77 1 2 3 4 55 66 77]

		Program exited.

スライスの設計によってこの単純な呼び出しが正しく機能することを可能にする方法を理解するために、その例の最後のワンライナーについて詳しく考える時間をとる価値があります。

コミュニティが構築した「 `Slice Trickes <https://golang.org/wiki/SliceTricks>`_ 」Wikiページには、スライスの ``append``、``copy`` 、およびその他の使用方法の例が他にもたくさんあります。

* Nil
===============================

余談ですが、私たちの新たな知見により ``nil`` スライスの表現が何であるかを見ることができます。当然、スライスヘッダーの値はゼロです。

.. code-block:: go

	sliceHeader{
		Length:        0,
		Capacity:      0,
		ZerothElement: nil,
	}

あるいは単に

.. code-block:: go

	sliceHeader{}

重要な詳細は、配列の要素へのポインタも ``nil`` であることです。

.. code-block:: go

	array[0:0]

上記によって作成されたスライスは長さ0(および場合によっては容量0)がありますが、そのポインターは ``nil`` ではないため、nilスライスではありません。

明らかなように、空のスライスは大きくなる可能性があります(容量がゼロでないと仮定します)が、``nil`` スライスには値を入れる配列がなく、1つの要素を保持するために大きくなることはありません。

つまり ``nil`` スライスは、要素へのポインタを保持していない場合でも、機能的に長さが0のスライスと同等です。 長さは0で、割り当てて追加できます。 例として、``nil`` スライスに追加してスライスをコピーする上記の1行のライナーを見てください。

* 文字列(Strings)
===============================

スライスのコンテキストにおけるGoの文字列に関する簡単なセクションです。

文字列は実際には非常に単純です。文字列は読み取り専用のバイトスライスであり、言語からの余分な構文サポートが少しあります。

これらは読み取り専用であるため、容量は必要ありません(容量を増やすことはできません)が、それ以外の場合はほとんどの目的で、読み取り専用のバイトスライスのように扱うことができます。

まず、個々のバイトにアクセスするためにインデックスを作成できます。

.. code-block:: go

	slash := "/usr/ken"[0] // yields the byte value '/'.

文字列をスライスして部分文字列を取得できます。

.. code-block:: go

	usr := "/usr/ken"[0:4] // yields the string "/usr"

これで、文字列をスライスしたときに裏で何が起こっているかが明らかになります。

通常のバイトスライスを取得し、単純な変換を使用して、そこから文字列を作成することもできます。

.. code-block:: go

	str := string(slice)

そして逆のこともできます。

.. code-block:: go

	slice := []byte(usr)

文字列の基になる配列はビューから隠されています。stringを使用しない限り、そのコンテンツにアクセスする方法はありません。つまり、これらの変換のいずれかを行う場合、配列のコピーを作成する必要があります。もちろん、Goがこれを処理するので、必要はありません。これらの変換のいずれかの後、バイトスライスの基になる配列への変更は、対応する文字列に影響しません。

この文字列のスライスのような設計の重要な結果は、部分文字列の作成が非常に効率的であることです。発生する必要があるのは、2ワードの文字列ヘッダーを作成することだけです。文字列は読み取り専用であるため、元の文字列とスライス操作の結果の文字列は同じ配列を安全に共有できます。

歴史的なメモ：文字列の初期実装は常に割り当てられますが、言語にスライスが追加されると、文字列を効率的に処理するためのモデルが提供されました。一部のベンチマークでは、結果として大幅な高速化が見られました。

もちろん、文字列にはさらに多くのものがあり、`別のブログ投稿 <https://blog.golang.org/strings>`_ で文字列の詳細を説明しています。

結論
===============================

スライスの仕組みを理解するには、スライスの実装方法を理解することが役立ちます。スライスヘッダーという小さなデータ構造があります。これは、スライス変数に関連付けられたアイテムです。このヘッダーは、個別に割り当てられた配列のセクションを記述します。 スライス値を渡すと、ヘッダーがコピーされますが、ヘッダーが指す配列は常に共有されます。

スライスがどのように機能するかを理解すると、スライスは使いやすくなるだけでなく、特に組み込み関数の ``copy`` と ``append`` の助けを借りて、強力で表現力豊かになります。

さらに読むために
===============================

Goのスライスに関するintertubesの周りには、たくさんの発見があります。 前述のように、「 `Slice Trickes <https://golang.org/wiki/SliceTricks>`_ 」Wikiページには多くの例があります。 `Go Slices <https://blog.golang.org/go-slices-usage-and-internals>`_ のブログ投稿では、メモリレイアウトの詳細をわかりやすい図で説明しています。 Russ Coxの `Go Data Structures <https://research.swtch.com/godata>`_ の記事には、Goの他の内部データ構造の一部とともにスライスの説明が含まれています。

.. todo:: intertubes の意味がわからない

より多くの資料が利用可能ですが、スライスについて学ぶ最良の方法はそれらを使用することです。

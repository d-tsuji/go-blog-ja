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

It seems clumsy in that example, especially dealing with the extra level of indirection
(a temporary variable helps),
but there is one common case where you see pointers to slices.
It is idiomatic to use a pointer receiver for a method that modifies a slice.

Let's say we wanted to have a method on a slice that truncates it at the final slash.
We could write it like this:

.play -edit slices/prog040.go /^type/,$

If you run this example you'll see that it works properly, updating the slice in the caller.

[Exercise: Change the type of the receiver to be a value rather
than a pointer and run it again. Explain what happens.]

On the other hand, if we wanted to write a method for `path` that upper-cases
the ASCII letters in the path (parochially ignoring non-English names), the method could
be a value because the value receiver will still point to the same underlying array.

.play -edit slices/prog050.go /^type/,$

Here the `ToUpper` method uses two variables in the `for` `range` construct
to capture the index and slice element.
This form of loop avoids writing `p[i]` multiple times in the body.

[Exercise: Convert the `ToUpper` method to use a pointer receiver and see if its behavior changes.]

[Advanced exercise: Convert the `ToUpper` method to handle Unicode letters, not just ASCII.]

* Capacity
===============================

Look at the following function that extends its argument slice of `ints` by one element:

.code slices/prog060.go /^func Extend/,/^}/

(Why does it need to return the modified slice?) Now run it:

.play -edit slices/prog060.go /^func main/,/^}/

See how the slice grows until... it doesn't.

It's time to talk about the third component of the slice header: its _capacity_.
Besides the array pointer and length, the slice header also stores its capacity:

	type sliceHeader struct {
		Length        int
		Capacity      int
		ZerothElement *byte
	}

The `Capacity` field records how much space the underlying array actually has; it is the maximum
value the `Length` can reach.
Trying to grow the slice beyond its capacity will step beyond the limits of the array and will trigger a panic.

After our example slice is created by

	slice := iBuffer[0:0]

its header looks like this:

	slice := sliceHeader{
		Length:        0,
		Capacity:      10,
		ZerothElement: &iBuffer[0],
	}

The `Capacity` field is equal to the length of the underlying array,
minus the index in the array of the first element of the slice (zero in this case).
If you want to inquire what the capacity is for a slice, use the built-in function `cap`:

	if cap(slice) == len(slice) {
		fmt.Println("slice is full!")
	}

* Make
===============================

What if we want to grow the slice beyond its capacity?
You can't!
By definition, the capacity is the limit to growth.
But you can achieve an equivalent result by allocating a new array, copying the data over, and modifying
the slice to describe the new array.

Let's start with allocation.
We could use the `new` built-in function to allocate a bigger array
and then slice the result,
but it is simpler to use the `make` built-in function instead.
It allocates a new array and
creates a slice header to describe it, all at once.
The `make` function takes three arguments: the type of the slice, its initial length, and its capacity, which is the
length of the array that `make` allocates to hold the slice data.
This call creates a slice of length 10 with room for 5 more (15-10), as you can see by running it:

.play -edit slices/prog070.go /slice/,/fmt/

This snippet doubles the capacity of our `int` slice but keeps its length the same:

.play -edit slices/prog080.go /slice/,/OMIT/

After running this code the slice has much more room to grow before needing another reallocation.

When creating slices, it's often true that the length and capacity will be same.
The `make` built-in has a shorthand for this common case.
The length argument defaults to the capacity, so you can leave it out
to set them both to the same value.
After

	gophers := make([]Gopher, 10)

the `gophers` slice has both its length and capacity set to 10.

* Copy
===============================

When we doubled the capacity of our slice in the previous section,
we wrote a loop to copy the old data to the new slice.
Go has a built-in function, `copy`, to make this easier.
Its arguments are two slices, and it copies the data from the right-hand argument to the left-hand argument.
Here's our example rewritten to use `copy`:

.play -edit slices/prog090.go /newSlice/,/newSlice/

The `copy` function is smart.
It only copies what it can, paying attention to the lengths of both arguments.
In other words, the number of elements it copies is the minimum of the lengths of the two slices.
This can save a little bookkeeping.
Also, `copy` returns an integer value, the number of elements it copied, although it's not always worth checking.

The `copy` function also gets things right when source and destination overlap, which means it can be used to shift
items around in a single slice.
Here's how to use `copy` to insert a value into the middle of a slice.

.code slices/prog100.go /Insert/,/^}/

There are a couple of things to notice in this function.
First, of course, it must return the updated slice because its length has changed.
Second, it uses a convenient shorthand.
The expression

	slice[i:]

means exactly the same as

	slice[i:len(slice)]

Also, although we haven't used the trick yet, we can leave out the first element of a slice expression too;
it defaults to zero. Thus

	slice[:]

just means the slice itself, which is useful when slicing an array.
This expression is the shortest way to say "a slice describing all the elements of the array":

	array[:]

Now that's out of the way, let's run our `Insert` function.

.play -edit slices/prog100.go /make/,/OMIT/

* Append: An example
===============================

A few sections back, we wrote an `Extend` function that extends a slice by one element.
It was buggy, though, because if the slice's capacity was too small, the function would
crash.
(Our `Insert` example has the same problem.)
Now we have the pieces in place to fix that, so let's write a robust implementation of
`Extend` for integer slices.

.code slices/prog110.go /func Extend/,/^}/

In this case it's especially important to return the slice, since when it reallocates
the resulting slice describes a completely different array.
Here's a little snippet to demonstrate what happens as the slice fills up:

.play -edit slices/prog110.go /START/,/END/

Notice the reallocation when the initial array of size 5 is filled up.
Both the capacity and the address of the zeroth element change when the new array is allocated.

With the robust `Extend` function as a guide we can write an even nicer function that lets
us extend the slice by multiple elements.
To do this, we use Go's ability to turn a list of function arguments into a slice when the
function is called.
That is, we use Go's variadic function facility.

Let's call the function `Append`.
For the first version, we can just call `Extend` repeatedly so the mechanism of the variadic function is clear.
The signature of `Append` is this:

	func Append(slice []int, items ...int) []int

What that says is that `Append` takes one argument, a slice, followed by zero or more
`int` arguments.
Those arguments are exactly a slice of `int` as far as the implementation
of `Append` is concerned, as you can see:

.code slices/prog120.go /Append/,/^}/

Notice the `for` `range` loop iterating over the elements of the `items` argument, which has implied type `[]int`.
Also notice the use of the blank identifier `_` to discard the index in the loop, which we don't need in this case.

Try it:

.play -edit slices/prog120.go /START/,/END/

Another new technique in this example is that we initialize the slice by writing a composite literal,
which consists of the type of the slice followed by its elements in braces:

.code slices/prog120.go /slice := /

The `Append` function is interesting for another reason.
Not only can we append elements, we can append a whole second slice
by "exploding" the slice into arguments using the `...` notation at the call site:

.play -edit slices/prog130.go /START/,/END/

Of course, we can make `Append` more efficient by allocating no more than once,
building on the innards of `Extend`:

.code slices/prog140.go /Append/,/^}/

Here, notice how we use `copy` twice, once to move the slice data to the newly
allocated memory, and then to copy the appending items to the end of the old data.

Try it; the behavior is the same as before:

.play -edit slices/prog140.go /START/,/END/

* Append: The built-in function
===============================

And so we arrive at the motivation for the design of the `append` built-in function.
It does exactly what our `Append` example does, with equivalent efficiency, but it
works for any slice type.

A weakness of Go is that any generic-type operations must be provided by the
run-time. Some day that may change, but for now, to make working with slices
easier, Go provides a built-in generic `append` function.
It works the same as our `int` slice version, but for _any_ slice type.

Remember, since the slice header is always updated by a call to `append`, you need
to save the returned slice after the call.
In fact, the compiler won't let you call append without saving the result.

Here are some one-liners intermingled with print statements. Try them, edit them and explore:

.play -edit slices/prog150.go /START/,/END/

It's worth taking a moment to think about the final one-liner of that example in detail to understand
how the design of slices makes it possible for this simple call to work correctly.

There are lots more examples of `append`, `copy`, and other ways to use slices
on the community-built
[[https://golang.org/wiki/SliceTricks]["Slice Tricks" Wiki page]].

* Nil
===============================

As an aside, with our newfound knowledge we can see what the representation of a `nil` slice is.
Naturally, it is the zero value of the slice header:

	sliceHeader{
		Length:        0,
		Capacity:      0,
		ZerothElement: nil,
	}

or just

	sliceHeader{}

The key detail is that the element pointer is `nil` too. The slice created by

	array[0:0]

has length zero (and maybe even capacity zero) but its pointer is not `nil`, so
it is not a nil slice.

As should be clear, an empty slice can grow (assuming it has non-zero capacity), but a `nil`
slice has no array to put values in and can never grow to hold even one element.

That said, a `nil` slice is functionally equivalent to a zero-length slice, even though it points
to nothing.
It has length zero and can be appended to, with allocation.
As an example, look at the one-liner above that copies a slice by appending
to a `nil` slice.

* Strings
===============================

Now a brief section about strings in Go in the context of slices.

Strings are actually very simple: they are just read-only slices of bytes with a bit
of extra syntactic support from the language.

Because they are read-only, there is no need for a capacity (you can't grow them),
but otherwise for most purposes you can treat them just like read-only slices
of bytes.

For starters, we can index them to access individual bytes:

	slash := "/usr/ken"[0] // yields the byte value '/'.

We can slice a string to grab a substring:

	usr := "/usr/ken"[0:4] // yields the string "/usr"

It should be obvious now what's going on behind the scenes when we slice a string.

We can also take a normal slice of bytes and create a string from it with the simple conversion:

	str := string(slice)

and go in the reverse direction as well:

	slice := []byte(usr)

The array underlying a string is hidden from view; there is no way to access its contents
except through the string. That means that when we do either of these conversions, a
copy of the array must be made.
Go takes care of this, of course, so you don't have to.
After either of these conversions, modifications to
the array underlying the byte slice don't affect the corresponding string.

An important consequence of this slice-like design for strings is that
creating a substring is very efficient.
All that needs to happen
is the creation of a two-word string header. Since the string is read-only, the original
string and the string resulting from the slice operation can share the same array safely.

A historical note: The earliest implementation of strings always allocated, but when slices
were added to the language, they provided a model for efficient string handling. Some of
the benchmarks saw huge speedups as a result.

There's much more to strings, of course, and a
[[https://blog.golang.org/strings][separate blog post]] covers them in greater depth.

結論
===============================

To understand how slices work, it helps to understand how they are implemented.
There is a little data structure, the slice header, that is the item associated with the slice
variable, and that header describes a section of a separately allocated array.
When we pass slice values around, the header gets copied but the array it points
to is always shared.

Once you appreciate how they work, slices become not only easy to use, but
powerful and expressive, especially with the help of the `copy` and `append`
built-in functions.

さらに読むために
===============================

There's lots to find around the intertubes about slices in Go.
As mentioned earlier,
the [[https://golang.org/wiki/SliceTricks]["Slice Tricks" Wiki page]]
has many examples.
The [[https://blog.golang.org/go-slices-usage-and-internals][Go Slices]] blog post
describes the memory layout details with clear diagrams.
Russ Cox's [[https://research.swtch.com/godata][Go Data Structures]] article includes
a discussion of slices along with some of Go's other internal data structures.

There is much more material available, but the best way to learn about slices is to use them.

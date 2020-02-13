============================================
Go Slices: usage and internals
============================================

Goのslice型は型付けされたデータの配列を操作する便利で効率的な方法を提供します。スライスは他の言語でいうところの配列に似ていますが、いくつかの特徴があります。本記事ではスライスの概要と使用方法を説明します。

配列
============================================

slice型はGoの配列型の上に構築されている抽象化であるため、スライスを理解するためには、まず配列を理解する必要があります。

配列型の定義は、配列の長さと要素の型を指定します。例えば、型 ``[4]int`` は4つの整数を格納する配列を表します。配列のサイズは固定されています。その長さはその型の一部です( ``[4]int`` と ``[5]int`` は別の型です)。配列には通常の方法でインデックスを付けることができるため、式 ``s[n]`` は0から始まる *n* 番目の要素にアクセスします。

.. code-block:: go

   var a [4]int
   a[0] = 1
   i := a[0]
   // i == 1

配列は明示的に初期化をする必要がありません。配列のゼロ値は、要素自体がゼロ値で初期化されているすぐに使用できる配列です。

.. code-block:: go

   // a[2] == 0, the zero value of the int type

``[4]int`` のメモリ上での表現は、連続して配置された4つの整数値です。

.. image:: images/go-slices-usage-and-internals_slice-array.png

Goの配列は値です。配列の変数は、配列全体を示します。(Cの場合のように)配列の最初の要素へのポインタではありません。これは、配列の値を割り当てたり、関数に渡したりすると、その内容のコピーが作成されることを意味します。(コピーを回避するために、配列への *ポインタ* を渡すことができますが、それは配列ではなく配列へのポインタです。)配列について考える1つの方法は、固定サイズの複合値のような名前付きフィールドではなく、インデックス付きフィールドのような構造です。

配列のリテラルは次のように指定できます。

.. code-block:: go

   b := [2]string{"Penn", "Teller"}

または、コンパイラに配列の要素をカウントさせることができます。

.. code-block:: go

   b := [...]string{"Penn", "Teller"}

いずれの場合も ``b`` の型は ``[2]string`` です。

スライス
============================================

配列はGoに含まれていますが、少し柔軟性に欠けるため、Goのコードではあまり頻繁に表れません。ただし、スライスはよく見かけます。配列上に構築され、優れた力と利便性を提供します。

スライスの型指定は ``[]T`` です。``T`` はスライスの要素の型です。配列型とは異なり、スライス型には長さが指定されていません。

スライスのリテラルは、要素の数を省略したことを除いて、配列のリテラルと同様に宣言されます。

.. code-block:: go

   letters := []string{"a", "b", "c", "d"}

スライスは ``make`` という組み込み関数を使用して作成できます。これには以下のようなシグネチャがあります。

.. code-block:: go

   func make([]T, len, cap) []T

Tは、作成するスライスの要素の型です。``make`` 関数は、型、長さ、およびオプションの容量を取ります。呼び出されると、``make`` は配列を割り当て、その配列を参照するスライスを返します。

.. code-block:: go

   var s []byte
   s = make([]byte, 5, 5)
   // s == []byte{0, 0, 0, 0, 0}

容量の引数を省略すると、デフォルトで長さの引数と同じになります。上記と同じコードでより簡潔なバージョンを次に示します。

.. code-block:: go

   s := make([]byte, 5)

組み込みの ``len`` および ``cap`` 関数を使用して、スライスの長さと容量を調べることができます。

.. code-block:: go

   len(s) == 5
   cap(s) == 5

次の2つのセクションでは、長さと容量の関係について説明します。

スライスのゼロ値は ``nil`` です。``len`` や ``cap`` 関数はnilスライスとして0を返します。

スライスは、既存のスライスまたは配列を「スライス」することでも作ることができます。スライスは、コロンで区切られた2つのインデックスで半開区間を指定することによって作成されます。例えば、``b[1：4]`` は、``b`` の1から3のインデックスの要素を含むスライスを作成します(結果のスライスのインデックスは0～2になります)。

.. code-block:: go

   b := []byte{'g', 'o', 'l', 'a', 'n', 'g'}
   // b[1:4] == []byte{'o', 'l', 'a'}, sharing the same storage as b

スライス式の開始インデックスと終了インデックスはオプションです。デフォルトはそれぞれゼロとスライスの長さです。

.. code-block:: go

   // b[:2] == []byte{'g', 'o'}
   // b[2:] == []byte{'l', 'a', 'n', 'g'}
   // b[:] == b

以下は与えられた配列からスライスを作成する構文の一つです。

.. code-block:: go

   x := [3]string{"Лайка", "Белка", "Стрелка"}
   s := x[:] // a slice referencing the storage of x

スライスの内部構造
============================================

スライスは、配列セグメントの記述子です。これは、配列へのポインター、セグメントの長さ、およびその容量(セグメントの最大長)で構成されています。

.. todo::

   どのように訳すと自然か。

   A slice is a descriptor of an array segment. It consists of a pointer to
   the array, the length of the segment, and its capacity (the maximum
   length of the segment).

.. image:: images/go-slices-usage-and-internals_slice-struct.png

変数 ``s`` はまず ``make([]byte, 5)`` によって作成され、以下のような構造になっています。

.. image:: images/go-slices-usage-and-internals_slice-1.png

長さは、スライスによって参照される要素の数です。容量は、基になる配列内の要素の数です(スライスポインターによって参照される要素から始まります)。次のいくつかの例を見ていくと、長さと容量の違いが明確になります。

.. note::

   重要なポイント

   * 長さは、スライスによって参照される要素の数
   * 容量は、基になる配列内の要素の数

次のように ``s`` をスライスするとき、スライスのデータ構造の変化と、基になる配列との関係を観察します。

.. code-block:: go

   s = s[2:4]

.. image:: images/go-slices-usage-and-internals_slice-2.png

スライスは、スライスのデータをコピーしません。元の配列を指す新しいスライス値を作成します。これにより、配列インデックスを操作するのと同じくらい効率的にスライス操作ができます。したがって、再スライスの *要素* (スライス自体ではない)を変更すると、元のスライスの要素が変更されます。

.. code-block:: go

   d := []byte{'r', 'o', 'a', 'd'}
   e := d[2:] 
   // e == []byte{'a', 'd'}
   e[1] = 'm'
   // e == []byte{'a', 'm'}
   // d == []byte{'r', 'o', 'a', 'm'}

先に ``s`` をその容量よりも短い長さにスライスしました。再びスライスすることで、容量を増やすことができます。

.. code-block:: go

   s = s[:cap(s)]

.. image:: images/go-slices-usage-and-internals_slice-3.png

スライスは容量を超えて増やすことはできません。スライスまたは配列の境界外でインデックスを作成するときと同様に、そうしようとすると、実行時パニックが発生します。同様に、配列内の以前の要素にアクセスするために、スライスを0未満にスライスすることはできません。

スライスの拡張(コピーと ``append`` 関数)
============================================

To increase the capacity of a slice one must create a new, larger slice
and copy the contents of the original slice into it. This technique is
how dynamic array implementations from other languages work behind the
scenes. The next example doubles the capacity of ``s`` by making a new
slice, ``t``, copying the contents of ``s`` into ``t``, and then
assigning the slice value ``t`` to ``s``:

.. code-block:: go

   t := make([]byte, len(s), (cap(s)+1)*2) // +1 in case cap(s) == 0
   for i := range s {
           t[i] = s[i]
   }
   s = t

The looping piece of this common operation is made easier by the
built-in copy function. As the name suggests, copy copies data from a
source slice to a destination slice. It returns the number of elements
copied.

.. code-block:: go

   func copy(dst, src []T) int

The ``copy`` function supports copying between slices of different
lengths (it will copy only up to the smaller number of elements). In
addition, ``copy`` can handle source and destination slices that share
the same underlying array, handling overlapping slices correctly.

Using ``copy``, we can simplify the code snippet above:

.. code-block:: go

   t := make([]byte, len(s), (cap(s)+1)*2)
   copy(t, s)
   s = t

A common operation is to append data to the end of a slice. This
function appends byte elements to a slice of bytes, growing the slice if
necessary, and returns the updated slice value:

{{code "/doc/progs/slices.go" \`/AppendByte/\` \`/STOP/`}}

One could use ``AppendByte`` like this:

.. code-block:: go

   p := []byte{2, 3, 5}
   p = AppendByte(p, 7, 11, 13)
   // p == []byte{2, 3, 5, 7, 11, 13}

Functions like ``AppendByte`` are useful because they offer complete
control over the way the slice is grown. Depending on the
characteristics of the program, it may be desirable to allocate in
smaller or larger chunks, or to put a ceiling on the size of a
reallocation.

But most programs don't need complete control, so Go provides a built-in
``append`` function that's good for most purposes; it has the signature

.. code-block:: go

   func append(s []T, x ...T) []T 

The ``append`` function appends the elements ``x`` to the end of the
slice ``s``, and grows the slice if a greater capacity is needed.

.. code-block:: go

   a := make([]int, 1)
   // a == []int{0}
   a = append(a, 1, 2, 3)
   // a == []int{0, 1, 2, 3}

To append one slice to another, use ``...`` to expand the second
argument to a list of arguments.

.. code-block:: go

   a := []string{"John", "Paul"}
   b := []string{"George", "Ringo", "Pete"}
   a = append(a, b...) // equivalent to "append(a, b[0], b[1], b[2])"
   // a == []string{"John", "Paul", "George", "Ringo", "Pete"}

Since the zero value of a slice (``nil``) acts like a zero-length slice,
you can declare a slice variable and then append to it in a loop:

{{code "/doc/progs/slices.go" \`/Filter/\` \`/STOP/`}}

**A possible "gotcha"**
============================================

As mentioned earlier, re-slicing a slice doesn't make a copy of the
underlying array. The full array will be kept in memory until it is no
longer referenced. Occasionally this can cause the program to hold all
the data in memory when only a small piece of it is needed.

For example, this ``FindDigits`` function loads a file into memory and
searches it for the first group of consecutive numeric digits, returning
them as a new slice.

{{code "/doc/progs/slices.go" \`/digit/\` \`/STOP/`}}

This code behaves as advertised, but the returned ``[]byte`` points into
an array containing the entire file. Since the slice references the
original array, as long as the slice is kept around the garbage
collector can't release the array; the few useful bytes of the file keep
the entire contents in memory.

To fix this problem one can copy the interesting data to a new slice
before returning it:

{{code "/doc/progs/slices.go" \`/CopyDigits/\` \`/STOP/`}}

A more concise version of this function could be constructed by using
``append``. This is left as an exercise for the reader.

**Further Reading**
============================================

`Effective Go </doc/effective_go.html>`__ contains an in-depth treatment
of `slices </doc/effective_go.html#slices>`__ and
`arrays </doc/effective_go.html#arrays>`__, and the Go `language
specification </doc/go_spec.html>`__ defines
`slices </doc/go_spec.html#Slice_types>`__ and their
`associated </doc/go_spec.html#Length_and_capacity>`__
`helper </doc/go_spec.html#Making_slices_maps_and_channels>`__
`functions </doc/go_spec.html#Appending_and_copying_slices>`__.

.. |image0| image:: slice-array.png
.. |image1| image:: slice-struct.png
.. |image2| image:: slice-1.png
.. |image3| image:: slice-2.png
.. |image4| image:: slice-3.png

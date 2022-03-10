# データ

## `new`を使ったメモリ割り当て

Goにはメモリを割り当てる2つの基本的な方法がありますが、組み込み（built-in）関数であるnewとmakeです。さまざまなことをして他のタイプに適用されるので混乱するかもしれませんが、ルールは簡単です。 newから見てみましょう。組み込み関数でメモリを割り当てるが他の言語に存在する同じ名前の機能とは異なり、メモリを初期化せず、単に値をゼロ化する。つまり、new（T）は、タイプTの新しいオブジェクトにゼロ値が格納されているスペースを割り当て、そのオブジェクトのアドレスである ` * T `値を返します。 Goの用語で言うと、新しくゼロ値に割り当てられたタイプTを指すポインタを返すのだ。

`new`で返されたメモリはゼロ値を持つので、あえて初期化プロセスなしで使用されたタイプのゼロ値をそのまま書くことができるようにデータ構造を設計すれば役に立ちます。どういうわけか、データ構造のユーザーが新しい構造体を新しく作成してすぐに仕事に使用できるということです。例えば、[bytes.Buffer](https://godoc.org/bytes#Buffer)の文書は、「Bufferのゼロ値はすぐに使用できるタングビンバッファである」と述べている。同様に、[sync.Mutex](https://godoc.org/sync#Mutex)も指定されたコンストラクタもInitメソッドも持っていない。代わりに、[sync.Mutex](https://godoc.org/sync#Mutex)のゼロ値であるロックされていないミューテックスとして定義されています。

ゼロ値の有用性には転移的な特性がある。次のタイプ宣言を検討してみよう。

```go
type SyncedBuffer struct {
    lock    sync.Mutex
    buffer  bytes.Buffer
}
```

`SyncedBuffer`型の値はメモリ割り当てや単に宣言だけでもすぐに使用する準備ができます。以下のコードフラグメントでは、pとvの後の調整がなくても、すべて正確に機能します。

```go
p := new(SyncedBuffer)  // type *SyncedBuffer
var v SyncedBuffer      // type  SyncedBuffer
```

## コンストラクタと合成リテラル

場合によっては、ゼロ値だけでは十分ではなくコンストラクタで初期化する必要があります。以下の例は、[os](https://godoc.org/os)パッケージから入手したものです。


```go
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := new(File)
    f.fd = fd
    f.name = name
    f.dirinfo = nil
    f.nepipe = 0
    return f
}
```

この例には、不必要に繰り返された（boiler plate）コードがたくさんあります。合成リテラルで単純化することができ、その表現が実行されるたびに新しいインスタンスを作成します。

```go
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := File{fd, name, nil, 0}
    return &f
}
```

Cとは異なり、ローカル変数のアドレスを返しても問題はありません。変数に連結された記憶空間は、関数が返しても生き残る。実際、合成リテラルのアドレスを取る表現は、毎回実行されるたびに新しいインスタンスに接続されます。したがって、最後の2行を結んでしまう可能性があります。

```go
    return &File{fd, name, nil, 0}
```

合成リテラルのフィールドは順番に配置され、必ず入力する必要があります。ただし、要素にラベルを付けてフィールド：値式で明示的にペアを作成すると、初期化は順序に関係なく表示されることがあります。入力されていない要素はそれぞれに合ったゼロ値を持ちます。したがって、以下のように書くことができる。

```go
    return &File{fd: fd, name: name}
```


限定的な場合には、合成リテラルが全くフィールドを持たない場合は、そのタイプのゼロ値を生成する。 `new(File)` は `&File{}` と同じ表現である。


合成リテラルは、arrays、slices、およびmapsを生成するためにも使用できます。フィールドラベルとしてインデックスとマップのキーを適切に使用する必要があります。以下の例では、`Enone`、`Eio`、そして`Einval`の値に関係なく、互いに異なるだけで初期化が機能します。

```go
a := [...]string   {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
s := []string      {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
m := map[int]string{Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
```

## makeを使用したメモリ割り当て


もう一度メモリ割り当てに戻りましょう。組み込み関数make（T、args）はnew（T）とは異なる目的のサービスを提供します。 slices、maps、および channels のみに使用し (`*T` ではない) タイプ T の (ゼロ値ではない) 初期化された値を返します。この違いがあるのは、この3種類が内部的に必ず使用前に初期化しなければならないデータ構造を指しているからだ。たとえば、sliceは3つの項目の記述項で（配列内）データを指すポインタ、サイズ、および容量を持ち、これらの項目が初期化されるまで、sliceは `nil'です。 slices、maps、およびchannelsに対して、makeは内部データ構造を初期化し、使用する値を準備します。例えば、

```go
make([]int, 10, 100)
```

メモリにサイズ100のint配列を割り当て、その配列の最初の10個を指す、サイズ10と容量100のスライスデータ構造を作成します。 （sliceを作成するときの容量は省略してもよい。詳細はsliceセクションを見ることを望む。）それに対して、 `new([]int)`は新たに割り当てられ、ゼロ値で満たされたslice構造を指すポインタを返すが、つまり `nil` slice 値のポインタであるのです。

次の例はnewとmakeの違いをよく描いています。

```go
var p *[]int = new([]int)       // slice 構造体を割り当てる。 * p == nil;ほとんど役に立たない
var v  []int = make([]int, 100) // slice vは今100個のintを持つ配列を参照しています

// 不必要に複雑な場合:
var p *[]int = new([]int)
*p = make([]int, 100, 100)

// Go 言語らしい場合:
v := make([]int, 100)
```

makeはmaps、slices、およびchannelsにのみ適用され、ポインタを返さないことを覚えておいてください。ポインタを取得したい場合は、newを使用してメモリを割り当てるか、変数のアドレスを明示的に取ります.

## 配列

アレイは、メモリのレイアウトを詳細に計画するのに役立ち、時にはメモリ割り当てを避けるのに役立ちます。しかし主に次のセクションの主題である、スリスの材料として使われる。配列について簡単に説明することで、スライスを議論するための基礎を固めましょう.


GoとCでは、配列の動作原理に大きな違いがあります。 Goでは、

* 配列は値です。ある配列を別の配列に割り当てると、すべての要素がコピーされます。
* 特に、関数に配列を渡すと、関数はポインタではなくコピーされた配列を受け取ります。
* 配列のサイズはタイプの一部です。タイプ[10]intと[20]intは互いに異なる。

配列を値として使用することは有用かもしれませんが、コストのかかる操作でもあります。 Cなどの実行や効率が必要な場合は、次のように配列ポインタを送ることもできます。

```go
func Sum(a *[3]float64) (sum float64) {
    for _, v := range *a {
        sum += v
    }
    return
}

array := [...]float64{7.0, 8.5, 9.1}
x := Sum(&array)  // 명시적인 주소 연산자(&)를 주목하라.
```

しかし、このようなスタイルでさえGo言語らしくはない。代わりにsliceを使用してください。

## Slices


Sliceはアレイをパッケージ化し、データシーケンスにより一般的で強力で便利なインターフェースを提供します。変換メリックスのような明確な次元を持つ項目を除いて、Goでは、ほとんどすべての配列プログラミングは単純な配列ではなくスライスを使用します。

Sliceは内部の配列を指すリファレンスを握っており、もし他のsliceに割り当てられても、両方とも同じ配列を指す。関数がsliceを受け取り、その要素に変更を与えると、呼び出し元も見ることができます。これは、内部の配列を指すポインタを関数に送るのと似ています。したがって、Read関数はポインタとカウンタの代わりにスライスを受け入れることができます。 slice内のlengthは、データを読み取ることができる最大限界値に決まっている。以下に[File](https://godoc.org/os#File)タイプのReadメソッドのシグネチャがある。

```go
func (f *File) Read(buf []byte) (n int, err error)
```

メソッドは、読み取ったバイト数と発生する可能性があるエラー値を返します。以下の例では、より大きなバッファ buf の最初の 32 バイトを読み込むためにそのバッファをスライスしました (ここでは slice を動詞で書いた)。

```go
    n, err := f.Read(buf[0:32])
```

そのようなスライシングは一般的に使用され効率的です。実際、効率をしばらく保留するには、以下の例もバッファの最初の32バイトを読みます。
```go
    var n int
    var err error
    for i := 0; i < 32; i++ {
        nbytes, e := f.Read(buf[i:i+1])  // Read one byte.
        if nbytes == 0 || e != nil {
            err = e
            break
        }
        n += nbytes
    }
```

スライスの長さは、内部配列の限界内でいくらでも変更できます。ただスライス車体に割り当て(assign)すればよい。 sliceの容量は、組み込み関数 `cap`を通じて得ることができ、sliceが持つことができる最大サイズを報告する。以下を見ると、スライスにデータをアタッチする機能があります。データが容量を超えると、スライスのメモリは再割り当てされます。結果のスライスは返されます。この関数は `len` と `cap` が `nil` slice に合法的に適用でき、0 を返すという事実を利用している。

```go
func Append(slice, data []byte) []byte {
    l := len(slice)
    if l + len(data) > cap(slice) {  // 再割り当ての場合
        // 将来の成長幅のために、必要な量の倍増を割り当てる.
        newSlice := make([]byte, (l+len(data))*2)
        // copy関数は事前に宣言されており、どのスライス型にも使用できます.
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0:l+len(data)]
    copy(slice[1:], data)
    return slice
}
```


sliceは必ず処理後に返さなければなりません。 Appendは `slice`の要素を変更することができますが、slice自体（ポインタ、サイズ、容量を持つランタイムデータ構造）は値に渡されたからです。

Sliceに付着するという考えは非常に有用なため、組み込み関数appendで作られている。この関数の設計を理解するには、情報がもう少し必要であるため、後で話し合うことにします。

## 二次元slices

Goの配列およびsliceは一次元である。二次元配列またはスライスと同じものを作成するには、配列の配列またはスライスのスライスを次のように定義する必要があります:

```go
type Transform [3][3]float64  // A 3x3 array, really an array of arrays.
type LinesOfText [][]byte     // A slice of byte slices.
```

Sliceはサイズが変わる可能性があるため、内部のsliceはサイズが異なる場合があります。次の LinesOfText の例のようによく起こりうる事である: 各行は独立して異なるサイズを持っている。

```go
text := LinesOfText{
	[]byte("Now is the time"),
	[]byte("for all good gophers"),
	[]byte("to bring some fun to the party."),
}
```

時には二次元のスライスをメモリに割り当てる必要があります。例えば、ピクセルの行はスキャンする状況です。 2つの方法で解決できます。最初のものは各スライスを個別に割り当てることです。 2つ目は、1つのsliceを割り当て、各sliceに切り取り、ポインタを与える方法です。どちらを書くかはアプリケーションに依存した。スライスが成長または縮小できる場合は、独立して割り当てて次の行を上書きするのを防ぐ必要があります。そうでなければ、オブジェクトを作成するためのメモリ割り当てを一度に行う方が効率的です。ちなみに、ここ2つの方式をスケッチしてみた。まず、一度に一行ずつ:

```go
//最上位レベルのスライスを割り当てます。
picture := make([][]uint8, YSize) // 単位 y ごとに 1 行ずつ。
//各行を繰り返しながらsliceを割り当てます。
for i := range picture {
	picture[i] = make([]uint8, XSize)
}
```

そして今、一度のメモリ割り当てで、カットして行を作成する場合:

```go
//上記のように、トップレベルの配列を割り当てます。
picture := make([][]uint8, YSize) // 単位 y ごとに 1 行ずつ。
//すべてのピクセルを収納できる大きなスライスを割り当てます。
pixels := make([]uint8, XSize*YSize) // picture는 [][]uint8タイプですが、pixelsは[]uint8タイプ.
// 各行を繰り返しながら、残されたピクセルスライスの最初からサイズまでスライスします。.
for i := range picture {
	picture[i], pixels = pixels[:XSize], pixels[XSize:]
}
```

## Maps

Mapは、便利で強力な組み込みデータ構造であるタイプの値（the <em> key </em>）を別のタイプ（<em> element</em>または<em> value </em>）の値にリンクします。やってくれる。 Key は equality 演算が定義されているどんなタイプでも使用可能であり、 integers, floating point, 複素数 (complex numbers), strings, ポインタ (pointers), インタフェース (equality をサポートする動的タイプに限って), structs そして 配列 (arrays ）がそのような例である。 Sliceはmapのkeyとして使用することはできません。なぜなら、equalityが定義されていないからです。 Sliceと同様に、mapも内部データ構造を持っています。関数にmapを入力してmapの内容を変更すると、その変更は呼び出し元にも表示されます。

Mapはまた、コロンで分離されたキー値のペアを使用して合成リテラルとして生成することができ、初期化中に簡単に作成できます。

```go
var timeZone = map[string]int{
    "UTC":  0*60*60,
    "EST": -5*60*60,
    "CST": -6*60*60,
    "MST": -7*60*60,
    "PST": -8*60*60,
}
```

Mapに値を割り当てたり、抽出したりする文法は、インデックスがintegerである必要がないこと以外は配列とsliceとほぼ同じです。

```go
offset := timeZone["EST"]
```

Mapにないキーを使用して値を抽出しようとすると、要素値の型に対応するゼロ値が返されます。たとえば、mapがintegerを持っている場合、存在しないkeyのクエリは0を返します。 bool型の値を持つmapでsetを実装できる。 map のエントリを `true` で保存することで set 内に値を取り込むことができ、単にインデックス付けで検査してみることができる。

```go
attended := map[string]bool{
    "Ann": true,
    "Joe": true,
    ...
}

if attended[person] { // もしpersonがマップにない場合はfalseになります。
    fmt.Println(person, "was at the meeting")
}
```

時には、部材値とゼロ値を区別する必要もある。 「UTC」のエントリ値があるのか​​、あるいはmapがまったくないから値が0ではないのか？それは複数の割り当ての形で区別することができます。

```go
var seconds int
var ok bool
seconds, ok = timeZone[tz]
```

明らかな理由で、これを「comma ok」イディオムと呼びます。この例では、tzがある場合、secondsは適切に設定され、okはtrueになります。一方、なければ、secondsはゼロ値になり、okはfalseになります。ここでは、良いエラー報告を使用して作成された関数の例があります。

```go
func offset(tz string) int {
    if seconds, ok := timeZone[tz]; ok {
        return seconds
    }
    log.Println("unknown time zone:", tz)
    return 0
}
```

実際の値に関係なくマップ内に存在するかどうかを調べるには、[空白識別子]（https://golang.org/doc/effective_go.html#blank）（ `_`）を値の変数が必要な場所に置くとなる。

```go
_, present := timeZone[tz]
```


Mapのエントリを削除するには、組み込み関数deleteを使い、mapと削除するkeyを引数として使う。 mapにkeyがすでに存在しない場合でも安全に使用できる。

```go
delete(timeZone, "PDT")  // Now on Standard Time
```

## 出力

Goでフォーマットされた出力はCの `printf`に似ていますが、より機能が豊富で一般的です。関数は[fmt](https://godoc.org/fmt)パッケージにあり、大文字の名前を持っています： `[fmt.Printf](https://godoc.org/fmt#Printf)`、 `[fmt .Fprintf](https://godoc.org/fmt#Fprintf)`, `[fmt.Sprintf](https://godoc.org/fmt#Sprintf)`, その他など。文字列関数（ `Sprintf`、その他など）は、提供されたバッファを埋めるよりも文字列を返します。

必ずフォーマット文字列を提供する必要はありません。 `Printf`、`Fprintf`、そして`Sprintf`に対して対になる関数があります。例えば、`Print`と`Println`です。これらの関数は、フォーマット文字列を取るのではなく、入力された引数に対してすでに指定されたフォーマットを生成します。 `Println`バージョンは引数の間にスペースを挿入し、出力に改行を追加します。一方、 `Print`バージョンは引数の両方が文字列でない場合にのみスペースを挿入します。以下の例では、各行が同じ出力を作成します。

```go
fmt.Printf("Hello %d\n", 23)
fmt.Fprint(os.Stdout, "Hello ", 23, "\n")
fmt.Println("Hello", 23)
fmt.Println(fmt.Sprint("Hello ", 23))
```


フォーマットされた出力関数[fmt.Fprint](https://godoc.org/fmt#Fprint)と同様の関数は、最初の引数として `io.Writer`インタフェースを実装したオブジェクトを取ります。変数`os.Stdout`と`os.Stderr`がおなじみのこれらのオブジェクトの例です。

ここで、物事はCから分岐し始めます。まず、％dなどの数値形式は、符号やサイズのフラグを取りません。代わりに、プリンティングルーチンは引数の型を使用してこれらのプロパティを決定します。

```go
var x uint64 = 1<<64 - 1
fmt.Printf("%d %x; %d %x\n", x, x, int64(x), int64(x))
```

위의예제는다음과같은출력을한다。

```go
18446744073709551615 ffffffffffffffff; -1 -1
```

整数を小数に置き換える例のような基本的な変換が必要な場合は、汎用目的のフォーマットである％v（valueという意味で）を使用できます。結果は「Print」と「Println」の出力と同じです。さらに、そのフォーマットはどんな値でも出力でき、さらに配列、スライス、そしてマップも出力する。以下は、前のセクションで定義されたタイムゾーンマップのためのprintステートメントです。

```go
fmt.Printf("%v\n", timeZone)  // or just fmt.Println(timeZone)
```

以下のように出力を提供します。

```go
map[CST:-21600 PST:-28800 EST:-18000 UTC:0 MST:-25200]
```

もちろん、マップの場合、キーはランダムに出力できます。 struct を出力するときは、修正されたフォーマットである %+v を介して構造体のフィールドに注釈を付け、代替フォーマットである %#v を使用すると、どんな値でも完全な Go 文法を出力する。

```go
type T struct {
    a int
    b float64
    c string
}
t := &T{ 7, -2.35, "abc\tdef" }
fmt.Printf("%v\n", t)
fmt.Printf("%+v\n", t)
fmt.Printf("%#v\n", t)
fmt.Printf("%#v\n", timeZone)
```

上記の例は次のように出力されます。

```go
&{7 -2.35 abc   def}
&{a:7 b:-2.35 c:abc     def}
&main.T{a:7, b:-2.35, c:"abc\tdef"}
map[string] int{"CST":-21600, "PST":-28800, "EST":-18000, "UTC":0, "MST":-25200}
```


（エンパーセントに注意してください。）引用文字列フォーマットは％qを使用して文字列型または[] byte型の値に適用したときに得られます。代替フォーマットである％＃qは、可能であればbackquoteを使用します。 (%q フォーマットはまた integer と rune に適用され、 single-quoted rune 定数を作る。) %x は文字列、 byte 配列、そして byte slice と integer に作用して長い 16 進数文字列を生成する。 (% x)フォーマットのようにスペースを中間に入れると（出力する） byteの間にスペースを入れてくれる。

別の有用なフォーマットは％Tで、値のタイプを出力します。

```go
fmt.Printf("%T\n", timeZone)
```

上記の例は次のように出力されます。

```go
map[string] int
```

カスタム型の基本フォーマットを操縦するためにすべきことは、単に `String() string` のシグネチャを持つメソッドを定義してくれることです。 （上記で定義された）単純なタイプTは、以下のフォーマットを持つことができます。

```go
func (t *T) String() string {
    return fmt.Sprintf("%d/%g/%q", t.a, t.b, t.c)
}
fmt.Printf("%v\n", t)
```

次のようなフォーマットで出力されます。

```go
7/-2.35/"abc\tdef"
```

（もし型Tと同時にポインタ型Tも一緒に出力する必要があるなら、 `String`のレシーバは値型でなければならない。詳細については、以下のリンクを参照してください：[pointers vs. value receivers](https://golang.org/doc/effective_go.html#pointers_vs_values))


StringメソッドがSprintfを呼び出すことができる理由は、printルーチンの再入が十分可能であり、例のように包まれてもよいからである。しかし、この方法について一つ理解し、越えなければならない非常に重要なディテールがあります。 Sprintfが受信機を文字列のように直接出力する場合、これが発生する可能性があります。一般的で簡単なミスで、次の例を見てみましょう。

```go
type MyString string

func (m MyString) String() string {
    return fmt.Sprintf("MyString=%s", m) // 에러: 영원히 재귀할 것임.
}
```

これらの間違いは簡単に修正することができます。引数を基本的な文字列型に変換すると、同じメソッドがないためです。

```go
type MyString string
func (m MyString) String() string {
    return fmt.Sprintf("MyString=%s", string(m)) // OK: note conversion.
}
```

[initialization section](https://golang.org/doc/effective_go.html#initialization) セクションに行くと、このような再帰呼出を避けることができる他のテクニックを見ることになるだろう。


別の出力技術は、出力ルーチンの引数を直接別の同様のルーチンに代入することです。 `Printf`のシグネチャは、最後の引数として任意の数値のパラメータがフォーマットの後に表示されることを明示するためにタイプ...interface{}を使用します。

```go
func Printf(format string, v ...interface{}) (n int, err error) {
```


`Printf`関数内で、vは `[]interface{}`型の変数のように振る舞いますが、もし他の可変引数関数（variadic function）に代入されると、通常の引数リストのように動作します。上で使用した `log.Println` の実装が以下にある。実際、フォーマットのために `fmt.Sprintln`関数に直接引数を代入している。

```go
// Println 関数はfmt.Printlnのように標準ロガーに出力します。
func Println(v ...interface{}) {
    std.Output(2, fmt.Sprintln(v...))  // Output 関数は（int、string）パラメータを取ります。
}
```


`Sprintln`を呼び出すネストされた呼び出しでvの後に来る...は、コンパイラにvを引数リストとして扱うように言います。そうでない場合は、vを1つのslice引数に代入します。


ここで見た出力に関するものよりはるかに多くの情報があります。 `godoc`の[fmt]（https://godoc.org/fmt）パッケージで詳細を学びましょう。

しかし、...パラメータは特定の型を持つことができます。たとえば、integerリストから最小値を選択する関数であるminの `...int`を見てください。

```go
func Min(a ...int) int {
    min := int(^uint(0) &gt;&gt; 1)  // largest int
    for _, i := range a {
        if i < min {
            min = i
        }
    }
    return min
}
```

## Append

これで、組み込み関数appendの設計を説明するために必要でしたが、欠けている部分がありました。 appendのシグネチャは、上記で作成したAppend関数とは異なります。図式的に、以下の通り:

```go
func append(slice []T, elements ...T) []T
```

ここで、Ｔはある種のプレースホルダである。実際、Go言語では、呼び出し元によって決定されるタイプTを書く関数を作成することはできません。だからappendは組み込み関数なのです：コンパイラのサポートが必要なのです。

appendがすることは、スライスの終わりに要素を付けて結果を返すことです。結果は返されるべきです。なぜなら、手書きのAppendのように、内部の配列は変わり得る。次の簡単な例を見てください。

```go
x := []int{1,2,3}
x = append(x, 4, 5, 6)
fmt.Println(x)
```

この例では[1 2 3 4 5 6]を出力します。 appendが `Printf`のように任意の数の引数を集めて動作するのです。

しかし、Appendのようにスライスにスライスを付けたい場合はどうすればよいですか？簡単な方法で、上からOutputを呼び出しながらやったように、呼び出し点で...を利用することです。下の予備フラグメントは上記と同じ結果をもたらす。

```go
x := []int{1,2,3}
y := []int{4,5,6}
x = append(x, y...)
fmt.Println(x)
```

...がなければコンパイルされません。なぜなら、型が間違っているからです：yはint型ではありません。

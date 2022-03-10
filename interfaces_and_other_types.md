# インターフェースと他のタイプ

## インターフェース


Go言語のインタフェースはオブジェクトの行為を指定する一つの方法である。すでに簡単ないくつかの例を見たことがあります。 String メソッドを実装すればオブジェクトのカスタム出力が可能で、Fprintf の出力で Write メソッドを持っているどんなオブジェクトでも書くことができる。 Go コードでは、1 つまたは 2 つのメソッドを指定してくれるインタフェースが普遍的であり、インタフェースの名前 (名詞) は通常メソッド (動詞) から派生する: Write メソッドを実装すると io.Writer がインタフェースの名前になる場合。

タイプは複数のインタフェースを実装することができる。 [sort.Interface](https://godoc.org/sort#Interface)を実装しているコレクションの例を見てみましょう。 [sort.Interface](https://godoc.org/sort#Interface) は Len(), Less(i, j int) bool, そして Swap(i, j int) を指定していて、このようなインタフェースを実装するなら sortパッケージ内 [IsSorted](https://godoc.org/sort#IsSorted), [Sort](https://godoc.org/sort#Sort), [Stable](https://godoc.org/sort# Stable）同じルーチンを使用できます。また、カスタムのフォーマットを実装することもできます。次の例では、Sequenceはこれら2つのインターフェースを満たしています。

```go
type Sequence []int

// sort.Interfaceの必須メソッド。
func (s Sequence) Len() int {
    return len(s)
}
func (s Sequence) Less(i, j int) bool {
    return s[i] < s[j]
}
func (s Sequence) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}

// プリントに必要なメソッド - プリントする前に要素をソートします。
func (s Sequence) String() string {
    sort.Sort(s)
    str := "["
    for i, elem := range s {
        if i > 0 {
            str += " "
        }
        str += fmt.Sprint(elem)
    }
    return str + "]"
}
```

## タイプ変換


equenceのStringメソッドは、Sprintがすでにスライスを持っていることを繰り返しています。しかし、Sprintを実行する前にSequenceを[] intに変換することで、作業を減らすことができます。

```go
func (s Sequence) String() string {
    sort.Sort(s)
    return fmt.Sprint([]int(s))
}
```

このような方法は、StringメソッドでSprintを安全に実行できるタイプ変換技法の別の例である。これが可能な理由は Sequence と []int 二つのタイプが名前だけ無視すれば同じであるため、合法的に互いに変換できることだ。この型変換は新しい値を作成せず、現在の値に新しい型があるかのように一時的に振る舞います。 （新しい値を作成する他の正当な変換もあります。たとえば、integerからfloating pointへの変換）


Goプログラムで一群の他のメソッドを使用するために型を変換するのはGo言語らしい表現です。たとえば、[sort.IntSlice]（https://godoc.org/sort#IntSlice）を使用して、上記のプログラム全体を次のように簡素化できます。

```go
type Sequence []int

// プリントに必要なメソッド - プリントする前に要素をソートします。
func (s Sequence) String() string {
    sort.IntSlice(s).Sort()
    return fmt.Sprint([]int(s))
}
```

見て！ Sequence が複数の (整列と出力) インタフェースを実装する代わりに、1 つのデータアイテムが複数のタイプ (Sequence, [sort.IntSlice](https://godoc.org/sort#IntSlice), そして []int)に変換できる点を利用している。各タイプは、与えられたタスクのある部分をカバーします。実戦でよく使われないが非常に効果的かもしれない。

## インターフェース変換とタイプの断言


タイプスイッチは変換の一形態である: インタフェースを受け取ったとき、switch文の各caseに合わせて型変換をする。以下の例は、[fmt.Printf](https://godoc.org/fmt#Printf) が型スイッチを使って、与えられた値を文字列に変換する方法を簡略化したバージョンで示しています。もし値がすでに文字列の場合はインタフェースがとっている実際の文字列値を望み、そうでなく値がStringメソッドを持っている場合はメソッドを実行した結果が欲しい。

```go
type Stringer interface {
    String() string
}

var value interface{} // callerによって提供された値。
switch str := value.(type) {
case string:
    return str
case Stringer:
    return str.String()
}
```


最初のケースは具体的な値が見つかった場合です。 2番目のケースは、インターフェースを別のインターフェースに変換した場合です。このように、いくつかのタイプを混ぜて使っても何の問題もありません。

ひとつだけタイプだけに興味がある場合はどうでしょうか？もし与えられた値が文字列を保存することを知っていて、その文字列値を抽出したい場合は？たった1つのケースだけを持つタイプスイッチであれば解決することができますが、タイプ単言表現を書くこともできる。型段落は、インテフェース値を持ち、指定された明確な型の値を抽出する。文法は型スイッチを開くのと似ていますが、typeキーワードの代わりに明確な型を使用します:


```go
value.(typeName)
```


そして、結果として得られる新しい値はtypeNameという静的型です。その型は、インタフェースが保持している具体的な型でなければ、その値を変換できる2番目のインタフェース型でなければならない。与えられた値内の文字列を抽出するために、次のように書くことができます:

```go
str := value.(string)
```


しかし、その値が文字列を持っていない場合、プログラムはランタイムエラーを出して死ぬ。このような惨事に備えるためには、「comma, ok」イディオムを使用して安全に値が文字列であることを確認する必要があります:

```go
str, ok := value.(string)
if ok {
    fmt.Printf("string value is: %q\n", str)
} else {
    fmt.Printf("value is not a string\n")
}
```


もしタイプ断言が検査で失敗した場合、strはまだ文字列型として存在し、文字列のゼロ値である空の文字列を持ちます。


可能な例を挙げると、上で示したタイプスイッチと同じ機能をするif-else文がここにある。

```go
if str, ok := value.(string); ok {
    return str
} else if str, ok := value.(Stringer); ok {
    return str.String()
}
```

## 一般性


もしあるタイプがただインタフェースを実装するためだけに存在するなら、つまりインタフェース以外のどのメソッドも外部にノンツンさせていない場合、タイプ自体を露出させる必要はない。インタフェースだけを公開することは、与えられた値がインテフェスに描かれた行為以外に何の興味深い機能もないことを確実に伝える。これはまた、共通の方法に対する文書化の反復を避けることができる。


そのような場合、コンストラクタは実装タイプではなくインタフェース値を返さなければなりません。たとえば、ハッシュライブラリである[crc32.NewIEEE](https://godoc.org/hash/crc32#NewIEEE)と[adler32.New](https://godoc.org/hash/adler32#New)はどちらも多インタフェースタイプ[hash.Hash32]（https://godoc.org/hash/Hash32）を返します。 GoプログラムでCRC-32アロリズムをAdler-32に置き換えるために必要なのは、単にコンストラクタコールを置き換えることです。他のコードはアルゴリズムの変化には影響されません。


同様の方法で、さまざまな暗号化パッケージ内のストリーミングシッパーアルゴリズムを、それらが接続しているブロックシッパーから分離することができます。 crypto/cipher パッケージ内の [Block] (https://godoc.org/crypto/cipher#Block) インタフェースは、1 ブロックのデータを暗号化する block cipher の行為を定義する。次に、bufioパッケージから推測できるように、[Block]（https://godoc.org/crypto/cipher#Block）インタフェースを実装するcipherパッケージは、[Stream]（https://godoc.org/ crypto / cipher＃Stream）インターフェースに代表されるストリーミングcipherを構築するとき、ブロック暗号化の詳細を知らなくても使用することができる。


crypto / cipherインターフェースは次のとおりです:

```go
type Block interface {
    BlockSize() int
    Encrypt(src, dst []byte)
    Decrypt(src, dst []byte)
}

type Stream interface {
    XORKeyStream(dst, src []byte)
}
```


ここでは、ブロックcipherをストリーミングcipherに置き換えるカウンタモード（CTR）ストリームの定義があります。 block cipherの詳細が抽象化されていることに注意してください:

```go
// NewCTRは、カウンダーモードで指定されたブロックを使用して暗号化および/復号化するストリームを返します。
// ivの長さはブロックのブロックサイズと同じでなければなりません。
func NewCTR(block Block, iv []byte) Stream
```


NewCTRは特定の暗号化アルゴリズムとデータソースにのみ適用されません。インタフェースを実装する任意のアルゴリズムやデータソースにも適用可能です。なぜなら、インタフェース値を返し、CTR暗号化を別の暗号化モードに置き換えることが局所的な変化であるからです。コンストラクタコールは必ず編集する必要があります。しかし、囲んでいるコードは戻り結果を[Stream](https://godoc.org/crypto/cipher#Stream)で処理しなければならないため、違いを知らない。

## インタフェースとメソッド


ほとんどすべてにメソッドを添付できるという言葉は、ほぼすべてがインタフェースを満たすことができるということでもある。 1つの会話的な例は、[http]（https://godoc.org/net/http）パッケージ内で定義されている[Handler]（https://godoc.org/net/http#Handler）インタフェースです。 [Handler]（https://godoc.org/net/http#Handler）を実装するどのオブジェクトもHTTPリクエストにサービスを提供できます。

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```


[ResponseWriter]（https://godoc.org/net/http#ResponseWriter）もクライアントに応答を返すために必要なメソッドのアクセスを提供するインタフェースです。これらのメソッドは標準のWriteメソッドを含み、[http.ResponseWriter](https://godoc.org/net/http#ResponseWriter)は[io.Writer](https://godoc.org/io#Writer)使用できる場所ならどこでも使用できる。 [Request](https://godoc.org/net/http#Request)は、クライアントから来るリクエストの分析された内容を盛り込んだstructである。


簡潔にするために、POSTは無視され、常にHTTPリクエストがGETであると仮定します。そのような単純化は、ハンドラーがどのようにシャットアップされるかに影響を与えません。ここではマイナーな例ですが、ページ訪問数を数えるハンダーの完全な実装があります。

```go
// シンプルなカウンターサーバー.
type Counter struct {
    n int
}

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ctr.n++
    fmt.Fprintf(w, "counter = %d\n", ctr.n)
}
```



（これまでの話と一脈相通する例として、[Fprintf](https://godoc.org/fmt#Fprintf)が[http.ResponseWriter](https://godoc.org/net/http#ResponseWriter)に出力できる注意してください。）参考までに、ここでURLにそのようなサーバーをどのように添付するかの例があります。

```go
import "net/http"
...
ctr := new(Counter)
http.Handle("/counter", ctr)
```

ところで、あえてCounterをstructにする理由があるのだろうか？ integerで十分です。 （callerに値の増加を示すために、レシーバはポインタである必要があります。）

```go
// シンプルなカウンターサーバー.
type Counter int

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    *ctr++
    fmt.Fprintf(w, "counter = %d\n", *ctr)
}
```

あなたのプログラムの内部状態がページを訪問することを知る必要がある場合はどうでしょうか？ウェブページにチャンネルを結ぶ。

```go
//チャンネルが訪問ごとに通知されます。
//（おそらくこのチャネルにはバッファを使用する必要があります。）
type Chan chan *http.Request

func (ch Chan) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ch <- req
    fmt.Fprint(w, "notification sent")
}
```


最後に、サーバーを駆動するために使用したコマンドライン引数を/ argsに表示したい場合を想像してみましょう。コマンドライン引数を出力する関数を書くのは簡単です。

```go
func ArgServer() {
    fmt.Println(os.Args)
}
```


これをどのようにHTTPサーバーに置き換えることができますか？どのタイプに加えて、値は無視しながらArgServerをメソッドにすることができます。しかし、より良い方法があります。ポインタとインタフェースだけを除いてはどのタイプにもメソッドを定義できる事実を利用して、関数にメソッドを書くことができる。 [http](https://godoc.org/net/http) パッケージには次のようなコードがあります:

```go
// HandlerFuncを使用すると、通常の関数をHTTPハンドラーとして書くことができます。
// f が適切な関数 signature を持つ場合、
// HandlerFunc（f）はfを呼び出すHandlerオブジェクトです。
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, req).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, req *Request) {
    f(w, req)
}
```


[HandlerFunc](https://godoc.org/net/http#HandlerFunc)は[ServeHTTP](https://godoc.org/net/http#ServeHTTP)というマッソードを同じタイプで、このタイプの値はHTTP requestにサービスを提供する。メソッドの実装を見てみましょう：受信機は関数、fであり、メソッドはfを呼び出します。奇妙に思えるかもしれませんが、受信機がチャネルであり、メソッドがチャネルにデータを送信する例と比較しても、それほど変わりはありません。


ArgServerをHTTPサーバーにするには、まず適切な署名を持つように修正する必要があります。

```go
// Argument server.
func ArgServer(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintln(w, os.Args)
}
```


ArgServerは[HandlerFunc]（https://godoc.org/net/http#HandlerFunc）と同じ signature です。 IntSlice.Sortメソッドを書くためにSequenceを[IntSlice]（https://godoc.org/sort#IntSlice）に変換したように、ServeHTTPを書くためにArgServerをHandlerFuncに変換することができます。シャックアップを行うコードは非常に簡潔です:

```go
http.Handle("/args", http.HandlerFunc(ArgServer))
```


誰が/ argsを訪問したとき、そのページにインストールされたhandlerはArgServer値を持つ[HandlerFunc]（https://godoc.org/net/http#HandlerFunc）タイプです。 HTTP サーバはそのタイプの ServeHTTP メソッドを呼びながら ArgServer をレシーバとして使い、最終的に ArgServer を呼ぶことになる: HandlerFunc.ServeHTTP の中で f(w, req) を呼ぶことになる。その後、コマンドライン引数が表示されます。


これまで、struct、integer、channel、そして関数（function）を持ってHTTPサーバーを作ってみました。これが可能な理由は、インタフェースがほぼすべての型で定義できる単純なメソッドの集合だからだ。

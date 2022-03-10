# 埋め込み(Embedding)

Goは典型的なタイプ主導型のサブクラス化を提供しません。しかし、structやインタフェースに型を埋め込むことで実装体の一部を「借りる」ことは可能である。


インタフェースの埋め込みは非常に簡単です。すでに言及されている [io.Reader](https://godoc.org/io#Reader) と [io.Writer](https://godoc.org/io#Writer) インタフェースの定義を見てみましょう。

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```


[io](https://godoc.org/io)パッケージは、いくつかの別のインタフェースを公開しており、複数のメソッドを実装できるオブジェクトを指定するのに書く。たとえば、[io.ReadWriter]（https://godoc.org/io#ReadWriter）はReadとWriteの両方を持っています。 2つのメソッドを明示的にリストして[io.ReadWriter]（https://godoc.org/io#ReadWriter）を定義することもできますが、より簡単で覚えやすい方法は2つのインターフェースを埋め込むことによって（embedded） 1つのインターフェースを形成することです:

```go
// ReadWriterはReaderとWriterインターフェースを組み合わせたインターフェースです。
type ReadWriter interface {
    Reader
    Writer
}
```


見えるように [Reader](https://godoc.org/io#Reader)がすることと[Writer](https://godoc.org/io#Writer)がすることを[ReadWriter](https://godoc.org /io＃ReadWriter）はすべてできることです。埋め込まれたインタフェースの組み合わせである（共通のメソッドを持たないインタフェースでなければならない）。インターフェイスだけがインターフェイスに挿入できます。


基本的に同じ考えがstructにも適用できるが、その影響ははるかに広範囲である。 [bufio](https://godoc.org/bufio)パッケージには [bufio.Reader](https://godoc.org/bufio#Reader) と [bufio.Writer](https://godoc.org/bufio ＃Writer）2つのstructタイプがあり、もちろん[io]（https://godoc.org/io）パッケージにある同様のインターフェースを実装しています。ところで [bufio](https://godoc.org/bufio) はまたバッファを内在した reader/writer を実装することもあるが、埋め込み（embedding）を利用して reader と writer を 1 つの struct に組み合わせるものである。タイプをリストしますが、フィールド名を与えない方法です。

```go
// ReadWriter ReaderとWriterへのポインタを保存します。
// io.ReadWriterを実装します。
type ReadWriter struct {
    *Reader  // *bufio.Reader
    *Writer  // *bufio.Writer
}
```


埋め込まれた要素はstructを指すポインタであり、もちろん使用する前に有効なstructでポイントをかけて初期化させなければならない。 ReadWriter structは以下のように書くことができる。

```go
type ReadWriter struct {
    reader *Reader
    writer *Writer
}
```


しかし、これを行うには、[io]（https://godoc.org/io）を満たし、readerとwriterが持っているメソッドを使用するために、転送用のメソッドを別々に提供する必要があります。

```go
func (rw *ReadWriter) Read(p []byte) (n int, err error) {
    return rw.reader.Read(p)
}
```


このような面倒なことを避けるためには、structを直接埋め込むことができます。埋め込まれた型のメソッドは自動的に続き、その意味は [bufio.ReadWriter](https://godoc.org/bufio#ReadWriter) は [bufio.Reader](https://godoc.org/bufio#Reader ）と[bufio.Writer](https://godoc.org/bufio#Writer)のメソッドの両方を持つことになるという。同時に、次の3つのインターフェースを満たすこともできます：[io.Reader](https://godoc.org/io#Reader)、[io.Writer](https://godoc.org/io#Writer)、 io.ReadWriter](https://godoc.org/io#ReadWriter)。


それでは、埋め込みがサブクラス化とは異なる重要な方法を見てみましょう。型を埋め込むと、その型のメソッドが外部型のメソッドになります。ただし、呼び出されたメソッドの受信機は内部型ではなく外部型ではありません。例では、[bufio.ReadWriter]（https：//godoc.org/bufio.ReadWriter）のReadメソッドが呼び出されたときに、転送メソッドを使用したのと同じ効果があります。レシーバーはReadWriterのリーダーフィールドであり、ReadWriter自体ではないのだ。


埋め込みは単純な便利さでもあります。この例では、埋め込まれたフィールドを名前付きの通常のフィールドとともに表示します。

```go
type Job struct {
    Command string
    *log.Logger
}
```


Job型は、今度は `* log.Logger`に属するLog、Logf、そして他のメソッドを持っています。 [Logger](https://godoc.org/log#Logger)に名前を与えることもできただろうが、そうする必要は全くない。そして今、初期化されたら、Jobに直接logを使うことができます:

```go
job.Log("starting now...")
```


LoggerはJob structの通常フィールドであるため、コンストラクタ内で常に行うように、次のように初期化できます。

```go
func NewJob(command string, logger *log.Logger) *Job {
    return &Job{command, logger}
}
```


またはcomposite literalを書いて

```go
job := &Job{command, log.New(os.Stderr, "Job: ", log.Ldate)}
```


埋め込まれたフィールドに直接言及する必要がある場合、ReadWriter structのReadメソッドのように、パッケージを無視したフィールドのタイプ名がフィールドの名前として使用されます。 Job 型の変数である job の `*log.Logger` にアクセスする必要がある場合は、 job.Logger と書けばよく、Logger のメソッドを改善したい場合に便利です。

```go
func (job *Job) Logf(format string, args ...interface{}) {
    job.Logger.Logf("%q: %s", job.Command, fmt.Sprintf(format, args...))
}
```


埋め込みタイプは名前の競合の問題を引き起こす可能性がありますが、解決するルールは簡単です。まず、フィールドやメソッドXは、型内のより深く入れ子になった部分にある別のXを隠して見えません。 log.LoggerがCommandというフィールドまたはメソッドを持っている場合、JobのCommandフィールドがより優勢です。


第二に、同じ名前が同じレベルでネストされて表示されると、通常はエラーが発生します。 Job structがLoggerという名前のフィールドやメソッドを持っている場合、log.Loggerを埋め込むのは間違っています。しかし、複製された名前が型定義の外で使用されたことがなければ大丈夫です。この資格は、外から埋め込まれたタイプに生じる変化に対するある程度の保護を提供します。 1つのフィールドが追加され、別のサブタイプのフィールドと競合が発生した場合、どちらのフィールドも使用されていない場合は問題ありません。

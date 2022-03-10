# 関数(Functions)

## 複数の戻り値

Go言語が持つ特徴の1つは、関数とメソッドが複数の値を返すことができるということです。この形式は、 `C`プログラムで帯域内エラーでEOFを表すために-1などの値を返し、アドレスに渡されたパラメータを変換するなど、いくつかの面倒な文法を改善するために使用されますできる。


C言語では、任意の場所に秘密の方法でエラーコードと負の数で書き込みエラーを示します。 Go言語の[Write]（https://golang.org/pkg/os/#File.Write）では、カウントとエラーを返すことができます。 「うん、数バイトくらいは書いてたけどデバイスにいっぱいだから全てのバイトを書けなかった」 `os` パッケージファイルにある [Write](https://golang.org/pkg/os/#File.Write) メソッドのシグネチャは次の通りです。

```go
func (file *File) Write(b []byte) (n int, err error)
```

そしてドキュメントでも言及しているように、Writeメソッドは `n != len(b)` の場合には使われたバイト数と nil ではなく `error` を返します。このような形式は非常に一般的であり、より多くの例を見たい場合は、エラーハンドリングセッションを見てみましょう。

同様に、戻り値で参照パラメータの真似を行うことで、ポインタを渡す必要がなくなることがあります。以下は、数字と次の位置を返すことによってバイトスライスにある数字を取得する単純な関数です。

```go
func nextInt(b []byte, i int) (int, int) {
    for ; i < len(b) && !isDigit(b[i]); i++ {
    }
    x := 0
    for ; i < len(b) && isDigit(b[i]); i++ {
        x = x*10 + int(b[i]) - '0'
    }
    return x, i
}
```

または、次のように入力スライスbで数字をスキャンするためにも使用できます。

```go
    for i := 0; i < len(b); {
        x, i = nextInt(b, i)
        fmt.Println(x)
    }
```

## 名前付き結果引数


Go関数では、戻り値「引数」または結果の「引数」に名前を付け、引数として入力されたパラメータのように通常の変数として使用できます。名前を付けると、対応する変数は、関数の開始時にそのタイプのゼロ値で初期化されます。関数が引数なしで戻り文を実行すると、結果パラメータの現在の値が戻り値として使用されます。

名前を与えることは必須ではありませんが、名前を付けるとコードをより短くて明確にし、文書化になる。 `nextInt`の結果に名前を付ける場合、返される `int`がどのようなものか明確になる。

```go
func nextInt(b []byte, pos int) (value, nextPos int) {
```

有名な結果は初期化され、内容なしで返されるので、明確であるだけでなく単純になることができます。以下はこれを使った `io.ReadFull`バージョンです。

```go
func ReadFull(r Reader, buf []byte) (n int, err error) {
    for len(buf) > 0 && err == nil {
        var nr int
        nr, err = r.Read(buf)
        n += nr
        buf = buf[nr:]
    }
    return
}
```

## Defer

Goの `defer`ステートメントは、 `defer`を実行する関数が返される前に即時関数呼び出し（延期された関数）を実行するように予約します。これは一般的な方法ではありませんが、関数がどのような実行パスを介して戻っている間にリソースを解放しなければならないかのような状況を処理する必要がある場合に効果的な方法です。最も代表的な例は、ミューテックスのロックを解除するか、ファイルを閉じることです。

```go
// Contents returns the file's contents as a string.
func Contents(filename string) (string, error) {
    f, err := os.Open(filename)
    if err != nil {
        return "", err
    }
    defer f.Close()  // f.Close will run when we're finished.

    var result []byte
    buf := make([]byte, 100)
    for {
        n, err := f.Read(buf[0:])
        result = append(result, buf[0:n]...) // append is discussed later.
        if err != nil {
            if err == io.EOF {
                break
            }
            return "", err  // f will be closed if we return here.
        }
    }
    return string(result), nil // f will be closed if we return here.
}
```


`Close`のような関数の呼び出しを遅らせると、2つの利点が得られます。まず、ファイルを閉じることを忘れている間違いをしないようにしてください。関数に新しい戻りパスを追加する必要がある場合によく見られる間違いです。 2番目に `open`の近くに `close`があると、関数の末尾に位置するよりもはるかに明確なコードになることを意味します。


defer関数のパラメータ（関数がメソッドの場合はレシーバも含まれます）は、関数の呼び出しが実行されるときではなくdeferが実行されたときに評価されます。また、関数の実行時に変数の値が変わることを心配する必要はありません。これは、1つのdefer呼び出し位置で複数の関数呼び出しを遅らせることができます。ここにやや幼稚な例があります。

```go
for i := 0; i < 5; i++ {
    defer fmt.Printf("%d ", i)
}
```

遅延関数はLIFOシーケンスによって実行されるため、上記のコードでは関数が返されると4 3 2 1 0が出力されます。もっともっともらしい例として、プログラムを通して関数の実行を追跡する簡単な方法があります。ここでは、以下のような簡単なトレースルーチンをいくつか作成しました:

```go
func trace(s string)   { fmt.Println("entering:", s) }
func untrace(s string) { fmt.Println("leaving:", s) }

// Use them like this:
func a() {
    trace("a")
    defer untrace("a")
    // do something....
}
```


`defer`が実行されたときに遅延関数のパラメータが評価されるという事実を利用すると、より良いことができます。トレースルーチンは、以下のようにトレースを終了するルーチンのパラメータに設定できます:

```go
func trace(s string) string {
    fmt.Println("entering:", s)
    return s
}

func un(s string) {
    fmt.Println("leaving:", s)
}

func a() {
    defer un(trace("a"))
    fmt.Println("in a")
}

func b() {
    defer un(trace("b"))
    fmt.Println("in b")
    a()
}

func main() {
    b()
}
```

prints

上記の関数は以下のような結果を出力します。

```sh
entering: b
in b
entering: a
in a
leaving: a
leaving: b
```


他の言語からブロックレベルのリソース管理に慣れているプログラマにとっては `defer` が不慣れに思えるかもしれないが、最も興味深く強力なアプリケーションは明らかにブロックベースではなく関数ベースという事実から来るということだ。 `panic`と`recover`セッションでは、これらの可能性の別の例を見てみましょう。

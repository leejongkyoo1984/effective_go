# 同時性(Concurrency)


## 通信による共有

並行性プログラミングは広範なトピックなので、ここではGoに限定されている重要なものについてのみ地面を割く。

共有変数への正確なアプローチを実装するために厳守する必要がある細部は、さまざまな環境で並行性プログラミングを困難にしました。 Goは、共有変数がチャンネルを返して渡されるという点で、他のアプローチをお勧めします。そして実際、共有変数は個々のスレッドの実行によって決して共有されません。いつでも1つのゴルチンが値にアクセスします。データ競合は、実装設計では発生できません。この考え方を奨励するために、これを1つのスローガンに減らしました。

共有メモリと通信してはいけない。代わりに、通信でメモリを共有します。

このアプローチはあまりにも過度のものかもしれません。例えば、整数型変数の周囲にミューテックスを置く方式のリファレンスカウントが最高かもしれない。しかし、より高いレベルでアクセスする方法として、アクセスを制御するチャネルを使用すると、明確で正確なプログラムをより簡単に作成できます。

このモデルについて考える1つの方法は、単一のCPUで実行される典型的なシングルスレッドプログラムを思い出すことです。これは同期のための基本的なデータ型を必要としません。今別のインスタンスを実行してみてください。しかし、やはり同期は必要ありません。これで2つの通信が可能になりますが、その通信自体が同期装置である場合、まだ他の同期は必要ありません。たとえば、Unixパイプラインはこのモデルに完全に収まります。同時性に対するGoのアプローチは、Hoareのコミュニケーションシーケンシャルプロセス（CSP、Communicating Sequential Processes）から来ているが、タイプセーフティが保証される式の一般化されたUNIXパイプと見なすことができる。

## ゴルティン

スレッド、コルーチン、プロセスなど、既存の用語は不正確な意味を伝えるため、ゴルチンと呼ばれます。ゴルチンは単純なモデルです。つまり、同じアドレス空間で他のゴルチンと同時に実行される関数である。ゴルチンは軽い。スタック領域を割り当てるよりもコストがかかります。そしてそのスタックは小さなサイズで始まります。だから安いです。そして必要なだけヒープストレージを割り当て（または解放）して大きくなる。

ゴルチンはOSのマルチスレッドに多重化されますが、I / O操作のために待機しているときのように、1つのゴルチンがブロックされると、別のゴルチンが実行され続けます。この設計は、スレッドの複雑な作成と管理についてあえて知る必要がなくなります。

新しいゴルチンを呼び出して実行するには、`go`キーワードを関数またはメソッド呼び出しの前に置きます。呼び出しが完了すると、ゴルチンは自動的に終了します。 （バックグラウンドでコマンドを実行するUnixシェルと表記と同様の効果です。）

```go
go list.Sort()  // list.Sort를 （ソートが完了するまで）待たずに同時に実行
```

関数リテラルはゴルチン呼び出しに役立ちます。

```go
func Announce(message string, delay time.Duration) {
    go func() {
        time.Sleep(delay)
        fmt.Println(message)
    }() //かっこに注意 - 関数を呼び出す必要があります
}
```

Goでは、関数リテラルはクロージャです。つまり、関数が参照する変数を使用している間は、その生存を保証する方法で実装されているということです。

上記の例は、関数が終了を知らせる方法がないため、非常に実用的ではありません。これにはチャネルが必要です。

## チャンネル

マップと同様に、チャネルは `make` に割り当てられ、結果の値は実際のデータ構造への参照として機能します。オプションの整数型パラメータが与えられると、チャネルのバッファサイズが設定されます。この値は、アンバッファードまたは同期チャネルのデフォルト値が0です。

```go
ci := make(chan int)            // 整数型のアンバッファードチャンネル
cj := make(chan int, 0)         // 整数型のアンバッファードチャンネル
cs := make(chan *os.File, 100)  // Fileポインタ型のバッファドチャンネル
```

バッファなしのチャネルは、同期で値を交換し、両方の計算（ゴルチン）がどの状態にあるかを知ることができることを保証する通信を組み合わせます。

チャンネルを使用する素晴らしいゴージャスなコードがたくさんあります。次の例で始めましょう。前のセクションでバックグラウンドでソートしました。チャネルは、ソートが完了するまでゴルチン実行を待つことができる。

```go
c := make(chan int)  // チャンネルを割り当てる
// ゴルチンでアライメントを開始し、完了するとチャンネルに信号を送信する
go func() {
    list.Sort()
    c <- 1  //  シグナルを送信しますが、値は問題ありません
}()
doSomethingForAWhile()
<-c   // ソートが終了するのを待って渡された値は破棄されます
```

受信部は、受信するデータがあるまで常にブロックされる。アンバッファードチャネルの場合、送信部は受信部が値を受け取るまでブロックされる。バッファ付きチャネルの場合、値がバッファにコピーされるまで送信部がブロックされます。したがって、バッファがいっぱいになると、これは特定の受信部が値を取得するのを待っていることを意味します。

バッファ付きチャネルはセマフォのように使用できます。例えば、スループットを制限するものです。次の例では、着信要求は `handle` に渡されます。 `handle`では、1つの値（どんな値でも構わない）をチャネルに送信し、要求を処理してから、チャネルから1つの値を受け取り、次の消費者のために「セマフォ」を準備させる。チャネルバッファのサイズが同時に処理できる数値を制限するものです。

```go
var sem = make(chan int, MaxOutstanding)

func handle(r *Request) {
	sem <- 1 // アクティブキューが空になるまで待機
	process(r) // 時間がかかる作業
	<-sem // 完了、実行する次の要求を有効にする
}

func Serve(queue chan *Request) {
    for {
        req := <-queue
        go handle(req)  // 終わるまで待たない
    }
}
```


一旦MaxOutstanding数のハンドラが `process`を実行している間、既存のハンドラの1つが完了してバッファから値を受け取るまで、よりいっぱいのチャネルバッファに送信することはブロックされます。


しかし、この設計は問題がある。つまり、リクエスト全体でたったの `MaxOutstanding` 数だけ `process` を実行できるにもかかわらず、サーバは着信するすべてのリクエストに対して新しいゴルチンを生成するということです。その結果、要求が早すぎる場合は、無制限にリソースを無駄にすることができます。この欠陥は、ゴルチンの生成を制限するゲートで `Serve`を修正することによって解決することができます。確実な解決策は次のとおりですが、後で修正するバグがあることに注意してください。

```go
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func() {
            process(req) // バグ、以下の説明を参照
            <-sem
        }()
    }
}
```


バグはGo `for`ループにあります。ループ変数は繰り返しごとに再利用され、同じ `req`変数がすべてのゴルチンで共有されます。これは望むものではありません。各ゴルチンごとに区別された `req`変数を持たなければなりません。以下は、このための1つの方法で、ゴルチンのクロージャの引数として `req`の値を渡すことです:

```go
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func(req *Request) {
            process(req)
            <-sem
        }(req)
    }
}
```


以前のバージョンとクロージャがどのように宣言され実行されるかを確認するために、次のバージョンを比較してください。別の解決策は、次の例のように、同じ名前の新しい変数を作成することです。

```go
func Serve(queue chan *Request) {
    for req := range queue {
        req := req // ゴルチン用の新しいreqインスタンスを作成する
        sem <- 1
        go func() {
            process(req)
            <-sem
        }()
    }
}
```

奇妙なコードを書くように見えるかもしれません。

```go
req := req
```


しかし、Goでこれを行うのは合法的でGo言語ダウンコードです。同じ名前の新しいバージョンの変数は、意図的にループ変数をローカルに選別しますが、各ゴルチンについてはユニークな値です。


サーバーを作成する一般的な問題に戻ると、リソースをうまく管理する別の方法は、要求チャネルを読み取るすべての `handle`ゴルチンを固定数で開始することです。ゴルチンの数は `process`が同時に呼び出される数を制限します。 `Serve`関数は終了信号を受信するようになるチャネルも（引数として）受け取っているのでゴルチンが始まると、このチャネルからの受信はブロックされる。

```go
func handle(queue chan *Request) {
    for r := range queue {
        process(r)
    }
}

func Serve(clientRequests chan *Request, quit chan bool) {
    // 핸들러 시작
    for i := 0; i < MaxOutstanding; i++ {
        go handle(clientRequests)
    }
    <-quit  // 終了信号を受信するまで待機
}
```

## チャンネルのチャンネル


Goの最も重要な属性の1つは、チャネルが他のものと同じように割り当てられて渡されることができる一次変数（first-class value）です。一般に、この属性は安全なパラレル逆多重化を実現するために使用されます。


前のセクションの例では、 `handle` は要求に対して理想的なハンドラーでしたが、ハンドラーが扱うタイプは定義しませんでした。対応するタイプが返信するチャネルを含む場合、各クライアントは自分に応答経路を提供することができる。以下は `Request`型の概略定義です。

```go
type Request struct {
    args        []int
    f           func([]int) int
    resultChan  chan int
}
```

クライアントは、関数とその引数だけでなく、要求オブジェクト内で応答を受信するチャネルを提供しています。

```go
func sum(a []int) (s int) {
    for _, v := range a {
        s += v
    }
    return
}

request := &Request{[]int{3, 4, 5}, sum, make(chan int)}
// リクエスト転送
clientRequests <- request
// 応答待ち
fmt.Printf("answer: %d\n", <-request.resultChan)
```

サーバ側ではハンドラ関数だけを修正すればよい。

```go
func handle(queue chan *Request) {
    for req := range queue {
        req.resultChan <- req.f(req.args)
    }
}
```

実際に使用するにはまだやることがたくさんあることは明らかですが、このコードは速度制限、パラレル、ノンブロックRPCシステムのためのフレームワークです。そしてミューテックスは目に見えません。

## 並列化


このアイデアの別のアプリケーションは、マルチコアCPUの計算を並列処理することです。計算を独立して実行可能な部分に分けることができる場合、各部分が完了したときに信号を送信するチャネルに並列処理することができる。


ベクトルアイテムに対する実行コストが高い演算があると仮定しよう。そして次の理想的な例のように各アイテムを演算した値は独立である。

```go
type Vector []float64

//  v[i]、v[i+1]からv[n-1]までの演算を適用
func (v Vector) DoSome(i, n int, u Vector, c chan int) {
    for ; i < n; i++ {
        v[i] += u.Op(v[i])
    }
    c <- 1    // この部分が完了すると信号
}
```


ループ内で独立して各部分をCPUごとに1つずつ実行させる。これらはどの順序でも完了できますが、順序は問題ありません。したがって、すべてのゴルチンを実行した後、チャンネルを空にして完了信号をカウントします。


```go
const numCPU = 4 // CPU  コア数

func (v Vector) DoAll(u Vector) {
    c := make(chan int, numCPU)  // バッファはオプションまたは常識
    for i := 0; i < numCPU; i++ {
        go v.DoSome(i*len(v)/numCPU, (i+1)*len(v)/numCPU, u, c)
    }
    // チャンネルを空にする
    for i := 0; i < numCPU; i++ {
        <-c    // 1つのタスクが終了するまで待機
    }
    // すべて完了
}
```

`numCPU`を定数値として生成するのではなく、適切な値を実行時に要求することができます。 `runtime.NumCPU`関数は機器のCPUハードウェアコア数を返します。だから以下のように書くことができる。

```go
var numCPU = runtime.NumCPU()
```

また、Goプログラムには、同時に実行できるカスタムコア数を報告する（または設定する） `runtime.GOMAXPROCS`関数があります。 `runtime.NumCPU`の値がデフォルト値ですが、同様に名前付き環境変数設定によって、あるいは正数の引数で関数を呼び出してオーバーライドできる。 0で呼び出すとすぐに値を照会します。したがって、ユーザーのリソース要求に従いたい場合は、次のように作成する必要があります。

```go
var numCPU = runtime.GOMAXPROCS(0)
```

コンポーネントを独立して処理してプログラムを構造化する並行性と、複数CPUで効率性のために計算を並列に処理する並列性の概念を混同しないことを願う。 Goの並行性の特徴は、並列計算で問題を簡単に構造化できますが、Goは並列ではなく並行性言語であり、すべての並列化の問題がGoに当てはまるわけではありません。このセグメンテーションの議論については、[このブログ投稿](https://blog.golang.org/concurrency-is-not-parallelism)で引用されているトルクを参照してください。


## 漏れバッファ


非同時性概念も同時実行プログラミングツールで簡単に表現できます。以下はRPCパッケージから抽出した例です。クライアントゴルチンは、おそらくネットワークである特定のソースからのデータを繰り返し受信します。バッファの割り当てと解放を避けるために `free list` を維持し、それに代わるバッファチャネルを使う。チャネルが空の場合、新しいバッファが割り当てられます。メッセージバッファが準備されたら、 `serverChan`のサーバに送信します。

```go
var freeList = make(chan *Buffer, 100)
var serverChan = make(chan *Buffer)

func client() {
    for {
        var b *Buffer
        // 利用可能なバッファを取得するか、そうでない場合は割り当て
        select {
            case b = <-freeList:
                // 1つを獲得し、何もしない
            default:
                // 取得するバッファがないので、新しいバッファを割り当てる
            b = new(Buffer)
        }
        load(b)              // ネットワークから次のメッセージを読む
        serverChan <- b      // サーバーに送信
    }
}
```


サーバーループは各クライアントからメッセージを受信して​​処理し、free listにバッファを返します。

```go
func server() {
    for {
        b := <-serverChan    // 動作を待つ
        process(b)
        // 可能であればバッファを再利用する
        select {
            case freeList <- b:
                // free listのバッファ。何もしない
            default:
                // free listがいっぱい、ただ続く
        }
    }
}
```


クライアントは `freeList`でバッファを取得しようとしますが、バッファが利用できない場合は新しいバッファを割り当てます。リストがいっぱいでない限り、サーバーはバッファを `freeList`に送信し、フリーリストにバッファbを戻します。リストがいっぱいになっている場合、バッファは底に落ちてガベージコレクタによって回収されます。 （ `select`構文の `default`句は他のケースが準備されていない場合に実行されます。これは `select`が決してブロックされないことを意味します。）もともと実装された漏れバケットフリーリスト（leaky bucket free list）を作成した。

# 初期化

CやC++の初期化とは一見違って見えないが、Goの初期化はもう少し強力だ。初期化中に複雑な構造体を作成することができ、初期化されるオブジェクト間の順序付けの問題も正確に処理されます。これは他のパッケージ間でも正しく機能します。

## 定数

Goの定数は私達が一般に考える定数である。定数は、 - 関数内でローカルに定義された定数でさえ - コンパイル時に生成され、数値、文字、文字列、真/偽のいずれかでなければなりません。定数を定義する式は、コンパイラがコンパイル時に実行できる定数式でなければなりません。例えば、 `1<<3` は定数式であるが、 `math.Sin(math.Pi/4)` は定数式ではない。 mathパッケージのSin関数への呼び出しは実行時にのみ可能です。

Goでは、列挙型（enum、自動増加定数型）定数をiotaという列挙子（enumerator）を用いて生成する。 iotaは式の一部として暗黙的に繰り返すことができます。これは複雑な値で構成される集合を簡単に作成します。

```go
type ByteSize float64

const (
    _           = iota // 空白識別子を使用して値0を無視する
    KB ByteSize = 1 << (10 * iota)
    MB
    GB
    TB
    PB
    EB
    ZB
    YB
)
```

GoはStringと同じメソッドをユーザーが定義したデータ型に付けることができます。これは、出力のために値の形式を自動的に変更できることを意味します。このようなメソッド結合は一般に構造体に適用されますが、このテクニックはByteSizeのような実数型のスカラー型でも役立ちます。

```go
func (b ByteSize) String() string {
    switch {
    case b >= YB:
        return fmt.Sprintf("%.2fYB", b/YB)
    case b >= ZB:
        return fmt.Sprintf("%.2fZB", b/ZB)
    case b >= EB:
        return fmt.Sprintf("%.2fEB", b/EB)
    case b >= PB:
        return fmt.Sprintf("%.2fPB", b/PB)
    case b >= TB:
        return fmt.Sprintf("%.2fTB", b/TB)
    case b >= GB:
        return fmt.Sprintf("%.2fGB", b/GB)
    case b >= MB:
        return fmt.Sprintf("%.2fMB", b/MB)
    case b >= KB:
        return fmt.Sprintf("%.2fKB", b/KB)
    }
    return fmt.Sprintf("%.2fB", b)
}
```

上記の例では、ByteSize変数にYBを入れると1.00YBと出力され、1e13（= 10,000,000,000,000）を入れると9.09TBに出力されます。

ここで ByteSize の String メソッドに適用した Sprintf の使用は繰り返し再帰呼出されることを避けたため安全である。型変換のためではなく、Sprintfをstring型ではなく％fオプションで呼び出したからだ。 （Sprintfは文字列が必要な場合にのみStringメソッドを呼び出し、％fは実数値（floating-point value）を使用します。）

## 変数

変数の初期化は定数と同じ方法ですが、初期化は実行時に計算される一般的な式でもよい。

```go
var (
    home   = os.Getenv("HOME")
    user   = os.Getenv("USER")
    gopath = os.Getenv("GOPATH")
)
```

## init関数

最後に、各ソースファイルは必要な状態を設定するためにそれぞれのinit関数を定義できます。 （init関数はパラメータを持たず、各ファイルは複数のinit関数を持つことができます。）「最後に」という言葉は本当に最後を指します。宣言が評価された後に呼び出されます。


宣言の形で表現できないものを初期化することに加えて、init関数は、実際のプログラムが実行される前にプログラムの状態を検証して正しく回復するためによく使用されます。

```go
func init() {
    if user == "" {
        log.Fatal("$USER not set")
    }
    if home == "" {
        home = "/home/" + user
    }
    if gopath == "" {
        gopath = home + "/go"
    }
    // gopath may be overridden by --gopath flag on command line.
    flag.StringVar(&gopath, "gopath", gopath, "override default GOPATH")
}
```

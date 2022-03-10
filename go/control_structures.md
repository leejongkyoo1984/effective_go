# 制御構造

Go言語の制御構造はCと関連性があるが重要な点で違いがある。 Go言語では`do`や`while`の繰り返し文は存在せず、ただもっと一般化された`for`、もう少し柔軟な`switch`が存在する。 `if`と`switch`はオプションで `for`のような初期化構文を受け取ることができます。 `break`と`continue`の構文は、オプションでどれを停止するか継続するかを識別するためにラベルを受け取ることができます。また、「タイプswitch」と「多方向通信マルチプレクサ」、「select」の新しい制御構造も含まれています。文法は少し違う。かっこは必要ではなく、bodyは常に中かっこで区切る必要があります。

## If

Go言語でのif文の簡単な例は次のとおりです:

```go
if x > 0 {
    return y
}
```

中かっこを義務的に使用する必要があるため、複数行でif構文が簡単に作成されます。どうせそうするのが良いスタイルであり、特に構文本体に return や break のような制御構文を含んでいる場合にはさらにそうです。


`if`と`switch`は初期化構文を受け入れるので、ローカル変数を設定するために使用された初期化構文がよく見られます。

```go
if err := file.Chmod(0664); err != nil {
    log.Print(err)
    return err
}
```

Go 複数のライブラリを見ると、if 構文が次の構文に進まないとき、つまり `break`、`continue`、`goto` または `return` によって構文本体が終了する場合、不要な `else` は省略されるものを見つけることができる。

```go
f, err := os.Open(name)
if err != nil {
    return err
}
codeUsing(f)
```


以下は、コードが一連のエラー条件を必ずチェックする必要がある一般的な状況の例です。制御の流れが成功した場合、コードはうまく動作し、エラーが発生するたびにエラーを排除します。エラーケースは `return` 構文で終了する傾向があるため、結果としてコードに `else' 構文は必要ありません。

```go
f, err := os.Open(name)
if err != nil {
    return err
}
d, err := f.Stat()
if err != nil {
    f.Close()
    return err
}
codeUsing(f, d)
```

## 再宣言と再割り当て


続いて：前のセクションの最後の例では、：=短い宣言がどのように機能するかを確認できました。 `os.Open`を呼び出す宣言コードを見てみましょう。

```go
f, err := os.Open(name)
```

この構文は `f`と `err`の2つの変数を宣言します。いくつかの行の下で `f.Stat` を呼び出す部分を見てみましょう。

```go
d, err := f.Stat()
```

ここで `d` と `err` を宣言するようです。注目すべき点は、その `err`が上から下の両方に現れるということです。この宣言の重複は合法的です。 `err`は最初の構文で宣言されていますが、2番目は再割り当てされています。これは、 `f.Stat` を呼び出すことでは、すでに宣言されて存在する `err` 変数を使用し、再び新しい値を与えるということを意味する。


変数の短縮宣言、 `v :=` で変数 v はすでに宣言されていても、次の場合に再宣言が可能です。

* この宣言が既存の宣言と同じスコープになければなりません（vがすでに外部スコープに宣言されている場合は、この宣言は新しい変数を作成します。）、
* 初期化表現内で対応する値はvに割り当てることができ、
* 少なくとも1つ以上の新しい変数が宣言文の中に一緒になければなりません。


このユニークな属性は完全に実用的であり、たとえば長く連鎖したif-else構文で1つのエラー値を簡単に使用できるようにします。よく使われるものを見るでしょう。

§Go言語の関数パラメータと戻り値は、関数を包んでいるブライスの外側にあるにもかかわらず、そのスコープは関数の胴体のスコープと同じであることに注意する価値があります。

## For

Go言語では、forイテレーションはC言語と似ていますが、一致しません。 forはwhileのように動作することができ、したがってdo-whileはありません。次の3つの形式が確認でき、1つの場合にのみセミコロンが使用されることが確認できます。

```go
// C言語と同じ場合
for init; condition; post { }

// C言語のwhileのように使用
for condition { }

// C언어의 for(;;) のように
for { }
```

短い宣言文は、反復文でindex変数宣言を簡単にします。

```go
sum := 0
for i := 0; i < 10; i++ {
    sum += i
}
```

配列、slice、string、map、およびチャネルから読み込む反復文を作成する場合、range構文はこの反復文を管理できます。

```go
for key, value := range oldMap {
    newMap[key] = value
}
```

rangeの中で最初のアイテムだけが必要な場合（キーまたはインデックス）、2番目の後ろを吹き飛ばそう:

```go
for key := range m {
    if key.expired() {
        delete(m, key)
    }
}
```

range内で2番目のアイテムだけが必要な場合（値）、空白識別子、アンダースコアを使用して最初を捨てるようにしましょう:

```go
sum := 0
for _, value := range array {
    sum += value
}
```

空白識別子には多くの使用法があり、[後で表示するセクション]（https://golang.org/doc/effective_go.html#blank）でよく説明されています。


文字列の場合、rangeはUTF-8解析によって個々のUnicode文字を処理するのに役立ちます。誤ったエンコーディングは、1バイトを削除し、U + FFFDルーン文字に置き換えます。 （ルーン（組み込み型で指定された）の名前はGo言語の単一のUnicodeコードの用語です。詳細は[言語仕様]（https://golang.org/ref/spec#Rune_literals） )

次の繰り返し文は

```go
for pos, char := range "日本\x80語" { // \x80は正当なUTF-8エンコーディングです
    fmt.Printf("character %#U starts at byte position %d\n", char, pos)
}
```

次のように出力される

```go
character U+65E5 '日' starts at byte position 0
character U+672C '本' starts at byte position 3
character U+FFFD '�' starts at byte position 6
character U+8A9E '語' starts at byte position 7
```

最後に、Go言語はコンマ（、）演算子を持たず、++、--は式ではなくステートメントです。したがって、for文内で複数の変数を使用するには、パラレル割り当てを使用する必要があります（++と-を除外しても）。

```go
// Reverse a
for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
    a[i], a[j] = a[j], a[i]
}
```

## Switch

Go言語では、スイッチはC言語よりも一般的な表現が可能です。式は定数であるか、必ずしも整数である必要はなく、case構文は上から底まで、その構文が `true`でない間に一致する値が見つかるまで値を比較します。したがって、 `if-else-if-else`の形式で書くのではなく、 `switch`を使うことが可能であるともっとGo言語らしい。

```go
func unhex(c byte) byte {
    switch {
    case '0' <= c && c <= '9':
        return c - '0'
    case 'a' <= c && c <= 'f':
        return c - 'a' + 10
    case 'A' <= c && c <= 'F':
        return c - 'A' + 10
    }
    return 0
}
```

スイッチでは自動的に次に通過する動作はないが（ケース構文を通る動作）、コンマ区切りリストを使用してケースを表現することができる。

```go
func shouldEscape(c byte) bool {
    switch c {
    case ' ', '?', '&', '=', '#', '+', '%':
        return true
    }
    return false
}
```


Cのような言語ではそれほど普遍的ではありませんが、Goでもbreak構文でswitchを早く終了するために書くことができます。たまには、スイッチではなく囲まれた繰り返し文を中断することが必要なこともあり、ラベルを繰り返し文の上に入れて、該当ラベルで「脱出」することによって完了することもある。次の例は、この2つの使用例です。

```go
Loop:
	for n := 0; n < len(src); n += size {
		switch {
		case src[n] < sizeOne:
			if validateOnly {
				break
			}
			size = 1
			update(src[n])

		case src[n] < sizeTwo:
			if n+1 >= len(src) {
				err = errShortInput
				break Loop
			}
			if validateOnly {
				break
			}
			size = 2
			update(src[n] + src[n+1]<<shift)
		}
	}
```


もちろん、`continue`構文もオプションでラベルを受け取ることができますが、反復文にのみ適用されます。


セクションを終了し、次は2つの `switch`構文を使用してバイトスライスを比較するルーチンの例です。: 

```go
// Compare returns an integer comparing the two byte slices,
// lexicographically.
// The result will be 0 if a == b, -1 if a < b, and +1 if a > b
func Compare(a, b []byte) int {
    for i := 0; i < len(a) && i < len(b); i++ {
        switch {
        case a[i] > b[i]:
            return 1
        case a[i] < b[i]:
            return -1
        }
    }
    switch {
    case len(a) > len(b):
        return 1
    case len(a) < len(b):
        return -1
    }
    return 0
}
```

## タイプswitch


スイッチ構文は、インターフェース変数の動的タイプを識別するために使用することができます。そのようなスイッチは型の言葉の文法を使用しますが、括弧でキーワード型を使用します。スイッチ式内で変数を宣言すると、変数は各節で一致する型を持つことになります。事実上、それぞれの句の中で新しい変数を別のタイプですが、同じ名前で宣言することと、各句の中で名前を再利用することが慣例的です。

```go
var t interface{}
t = functionOfSomeType()
switch t := t.(type) {
default:
    fmt.Printf("unexpected type %T\n", t)     // %T prints whatever type t has
case bool:
    fmt.Printf("boolean %t\n", t)             // t has type bool
case int:
    fmt.Printf("integer %d\n", t)             // t has type int
case *bool:
    fmt.Printf("pointer to boolean %t\n", *t) // t has type *bool
case *int:
    fmt.Printf("pointer to integer %d\n", *t) // t has type *int
}
```

# コメント(Commentary)

Go言語はC言語スタイルの/\** \**/ブロックコメントとC++スタイルの//一行コメントを提供します。一行コメントは一般的に使用され、ブロックコメントはほとんどのパッケージコメントに表示されます。しかし、式の中や多くのコードをコメントアウトするのに便利です。

プログラムやWebサーバーでもあるgodocは、パッケージの内容に関する文書を抽出するためにGoソースファイルを処理します。最上位宣言文の前に挟んだ改行なしで注釈が現れれば、その宣言文とともに抽出され、該当項目の説明として提供される。これらのコメントのスタイルとタイプは、godocが作成する文書の質を決定します。

すべてのパッケージには、パッケージ構文の前にブロックコメント型のパッケージコメントが必要です。複数のファイルで構成されたパッケージの場合、パッケージのコメントは、どのファイルに関係なく、1つのファイルに存在する必要があり、それが使用されます。パッケージ注釈はパッケージを紹介し、パッケージ全体に関連する情報を提供しなければならない。パッケージコメントはgodocドキュメントの最初に表示されるので、後の詳細も書く必要があります。

```go
/*
Package regexp implements a simple library for regular expressions.

The syntax of the regular expressions accepted is:

    regexp:
        concatenation { '|' concatenation }
    concatenation:
        { closure }
    closure:
        term [ '*' | '+' | '?' ]
    term:
        '^'
        '$'
        '.'
        character
        '[' [ '^' ] character-ranges ']'
        '(' regexp ')'
*/
package regexp
```

パッケージが単純な場合は、パッケージコメントも簡単です。

```
// Package path implements utility routines for
// manipulating slash-separated filename paths.
```

あまり行を書く過度なフォーマットは注釈に必要ない。さらに生成された出力が固定幅フォントで与えられないこともあるので、スペースや整列などに依存しないでください。これらはgofmtと同様にgodocによって処理されます。コメントは解釈されないプレーンテキストです。だからHTMLや `_this_`のようなコメントは書かれたまま表示されます。したがって、使用しない方が良い。 godocが修正する1つは、インデントされたテキストを固定幅フォントで表示することで、プログラムコードの彫刻のようなものに適している。 [fmt package]（https://golang.org/pkg/fmt/）のパッケージコメントは良い例です。

状況に応じて、godocはコメントを再変更しない可能性があります。だから確実に見事に作らなければならない。正確なスペル、口頭法、文章構造を使用し、緊張を減らさなければならない。

パッケージ内の最上位宣言の直前のコメントは、その宣言の文書コメントとして扱われます。
パッケージ内の最上位宣言の直前のコメントは、その宣言のための文書コメントです。プログラムですべての外部に公開される（大文字で始まる）名前には文書コメントが必要です。

文書コメントは、非常に多様な自動化された表現を可能にする完全な文で書かれたときに最も効果的です。最初の文は、宣言された名前で始まる1行の文にまとめなければなりません。

```go
// Compile parses a regular expression and returns, if successful,
// a Regexp that can be used to match against text.
func Compile(str string) (*Regexp, error) {
```

もしすべての文書のコメントがそのコメントが記述する項目の名前で始まる場合、godocの結果はgrepステートメントを使用するのに役立つ形式になります。もしあなたが正規表現の解析関数を探していますが、「Compile」という名前を覚えておらず、次の命令を実行したと想像してみましょう。

```go
$ godoc regexp | grep parse
```

もし文書コメントが「This function..」で始まる場合、grepコマンドはあなたが望む結果を示すことができないでしょう。しかし、パッケージは各ドキュメントコメントをパッケージ名で始めるので、以下のようにあなたが探していた単語を思い出させる結果を見ることができるでしょう。

```go
$ godoc regexp | grep parse
    Compile parses a regular expression and returns, if successful, a Regexp
    parsed. It simplifies safe initialization of global variables holding
    cannot be parsed. It simplifies safe initialization of global variables
$
```

Go言語の宣言構文はグループ化が可能である。 1つの文書コメントは、関連する定数または変数のグループについて説明できます。しかし、これらのコメントは宣言全体に現れるので、形式的なものにすることができます。

```go
// Error codes returned by failures to parse an expression.
var (
    ErrInternal      = errors.New("regexp: internal error")
    ErrUnmatchedLpar = errors.New("regexp: unmatched '('")
    ErrUnmatchedRpar = errors.New("regexp: unmatched ')'")
    ...
)
```

グループ化は項目間の関連性を表すことができる。たとえば、以下の変数のグループはミューテックスによって保護されていることを示しています。

```go
var (
    countLock   sync.Mutex
    inputCount  uint32
    outputCount uint32
    errorCount  uint32
)
```

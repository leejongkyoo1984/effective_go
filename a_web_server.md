# Webサーバー

完全なGoプログラムであるWebサーバーで終了しましょう。これは実際には一種のWebサーバーです。Googleは、chart.apis.google.com データをチャートやグラフに自動フォーマットするサービスを提供しています。ただし、データをクエリとしてURLに入力する必要があるため、インタラクティブに使用するのは困難です。ここでのプログラムは、1つの形式のデータへのより優れたインターフェイスを提供します。短いテキストが与えられると、チャートサーバーを呼び出して、テキストをエンコードするボックスのマトリックスであるQRコードを生成します。その画像は携帯電話のカメラで取得して、たとえばURLとして解釈できるため、携帯電話の小さなキーボードにURLを入力する手間が省けます。

これが完全なプログラムです。説明は次のとおりです。


```go
package main

import (
    "flag"
    "html/template"
    "log"
    "net/http"
)

var addr = flag.String("addr", ":1718", "http service address") // Q=17, R=18

var templ = template.Must(template.New("qr").Parse(templateStr))

func main() {
    flag.Parse()
    http.Handle("/", http.HandlerFunc(QR))
    err := http.ListenAndServe(*addr, nil)
    if err != nil {
        log.Fatal("ListenAndServe:", err)
    }
}

func QR(w http.ResponseWriter, req *http.Request) {
    templ.Execute(w, req.FormValue("s"))
}

const templateStr = `
<html>
<head>
<title>QR Link Generator</title>
</head>
<body>
{{if .}}
<img src="http://chart.apis.google.com/chart?chs=300x300&cht=qr&choe=UTF-8&chl={{.}}" />
<br>
{{.}}
<br>
<br>
{{end}}
<form action="/" name=f method="GET"><input maxLength=1024 size=70
name=s value="" title="Text to QR Encode"><input type=submit
value="Show QR" name=qr>
</form>
</body>
</html>
`
```

これまでの部分mainは簡単に理解できるはずです。oneフラグは、サーバーのデフォルトのHTTPポートを設定します。テンプレート変数templは、楽しみが発生する場所です。サーバーがページを表示するために実行するHTMLテンプレートを作成します。それについてはすぐに詳しく説明します。

関数はフラグを解析し、上記mainで説明したメカニズムを使用して、関数QRをサーバーのルートパスにバインドします。次にhttp.ListenAndServe、サーバーを起動するために呼び出されます。サーバーの実行中はブロックします。

QRフォームデータを含むリクエストを受信し、という名前のフォーム値のデータに対してテンプレートを実行するだけsです。

テンプレートパッケージhtml/templateは強力です。このプログラムは、その機能に触れているだけです。本質的には、渡されたデータ項目templ.Execute（この場合はフォーム値）から派生した要素を置き換えることにより、その場でHTMLテキストの一部を書き換えます。テンプレートテキスト（templateStr）内で、二重中括弧で区切られた部分はテンプレートアクションを示します。からの部分は、（ドット）と呼ばれる現在のデータ項目の値が空でない場合{{if .}} にのみ実行されます。つまり、文字列が空の場合、テンプレートのこの部分は抑制されます。 {{end}}.

2つのスニペット{{.}}は、テンプレートに表示されるデータ（クエリ文字列）をWebページに表示することを示しています。HTMLテンプレートパッケージは、テキストを安全に表示できるように、適切なエスケープを自動的に提供します。

テンプレート文字列の残りの部分は、ページが読み込まれたときに表示されるHTMLにすぎません。説明が速すぎる場合は 、テンプレートパッケージ の[ドキュメントを参照して、より詳細に説明してください](https://pkg.go.dev/html/template)。

そして、あなたはそれを持っています：数行のコードといくつかのデータ駆動型HTMLテキストの便利なウェブサーバー。Goは、数行で多くのことを実現するのに十分強力です。

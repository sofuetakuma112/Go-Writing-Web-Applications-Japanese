# Webアプリケーションの作成

## イントロダクション

このチュートリアルで扱う内容

- ロードとセーブのメソッドを持つデータ構造の作成
- `net/http` パッケージを用いたウェブアプリケーションの構築
- HTML テンプレートを処理するための `html/template` パッケージの使用法
- ユーザ入力を検証するための `regexp` パッケージの利用
- クロージャの使用

想定される知識

- プログラミングの経験
- 基本的な Web 技術の理解 (HTTP、HTML)
- UNIX/DOS のコマンドラインに関する知識

## はじめに

現在のところ、Goを実行するにはFreeBSD, Linux, macOS, Windowsのいずれかのマシンが必要です。ここでは、コマンドプロンプトを表すために `$` を使用することにします。

Goをインストールします（[インストール方法](https://go.dev/doc/install)を参照してください）。

`GOPATH`の中にこのチュートリアルのための新しいディレクトリを作り、そこに`cd`します。

```
$ mkdir gowiki
$ cd gowiki

```

`wiki.go`という名前のファイルを作成し、お好みのエディタで開いて、以下の行を追加してください。

```go
package main

import (
    "fmt"
    "os"
)
```

ここでは、Go の標準ライブラリから `fmt` と `os` パッケージをインポートしている。後で、追加機能を実装する際には、この `import` 宣言にさらにパッケージを追加することになる。

## データ構造
データ構造の定義から始めましょう。wikiは相互に接続された一連のページで構成され、それぞれのページはタイトルとボディ（ページのコンテンツ）を持っています。ここでは、タイトルと本文を表す2つのフィールドを持つ構造体として `Page` を定義しています。

```go
type Page struct {
    Title string
    Body  []byte
}
```

型 `[]byte` は "`byte` のスライス" を意味します。(スライスについて詳しくは [Slices: usage and internals](https://go.dev/doc/articles/slices_usage_and_internals.html) を参照してください)。`Body` 要素は `string` ではなく `[]byte` で、これは後述する `io` ライブラリが期待する型だからです。

`Page` 構造体は、ページデータがどのようにメモリに保存されるかを記述します。しかし、永続的な保存についてはどうでしょうか？そのためには、 `Page` に `save` メソッドを作成します。

```go
func (p *Page) save() error {
    filename := p.Title + ".txt"
    return os.WriteFile(filename, p.Body, 0600)
}
```

このメソッドのシグネチャは次のようになっています。「これは `save` という名前のメソッドで、 `p` という `Page` へのポインタをレシーバとして受け取ります。パラメータを受け取らず、 `error` 型の値を返します。" とあります。

このメソッドは `Page` の `Body` をテキストファイルに保存します。簡単のために、ファイル名には `Title` を使用します。

`save` メソッドは `error` という値を返します。これは `WriteFile` (バイトスライスをファイルに書き出す標準ライブラリ関数) の戻り値の型だからです。`save` メソッドはエラー値を返します。これは、ファイルの書き込み中に何か問題が発生した場合に、アプリケーション側で処理できるようにするためです。すべてがうまくいくと、 `Page.save()` は `nil` (ポインタ、インターフェース、その他のいくつかの型のゼロ値) を返します。

8進数の整数リテラル `0600` は `WriteFile` の3番目のパラメーターとして渡され、ファイルが現在のユーザーに対してのみ読み書き可能なパーミッションで作成されるべきであることを示します。(詳しくは Unix の man ページ `open(2)` を参照してください)。

ページを保存するだけでなく、ページを読み込むことも必要です。

```go
func loadPage(title string) *Page {
    filename := title + ".txt"
    body, _ := os.ReadFile(filename)
    return &Page{Title: title, Body: body}
}
```

関数 `loadPage` は title パラメータからファイル名を作成し、そのファイルの内容を新しい変数 `body` に読み込み、適切な title と body の値で作成された `Page` リテラルへのポインタを返します。

関数は複数の値を返すことができます。標準ライブラリ関数 `os.ReadFile` は `[]byte` と `error` を返します。アンダースコア (`_`) シンボルで表される "空白の識別子" を使用して、エラーの返り値を破棄します (要するに、値を何も代入しないということです)。

しかし、 `ReadFile` がエラーに遭遇した場合はどうなるのでしょうか? 例えば、ファイルが存在しないかもしれません。このようなエラーを無視してはいけません。この関数を修正して、 `*Page` と `error` を返すようにしましょう。

```go
func loadPage(title string) (*Page, error) {
    filename := title + ".txt"
    body, err := os.ReadFile(filename)
    if err != nil {
        return nil, err
    }
    return &Page{Title: title, Body: body}, nil
}
```

この関数の呼び出し側は、2番目のパラメータをチェックできるようになりました。もし `nil` ならば、ページの読み込みに成功したことになります。そうでない場合は `error` となり、呼び出し元で処理することができます (詳細は [言語仕様](https://go.dev/ref/spec#Errors) を参照してください)。

この時点で、私たちは単純なデータ構造と、ファイルへの保存とファイルからの読み込みの機能を手に入れました。では、書いたものをテストするために `main` 関数を書いてみましょう。

```go
func main() {
    p1 := &Page{Title: "TestPage", Body: []byte("This is a sample Page.")}
    p1.save()
    p2, _ := loadPage("TestPage")
    fmt.Println(string(p2.Body))
}
```
このコードをコンパイルして実行すると、`p1` の内容を含む `TestPage.txt` という名前のファイルが作成されます。そして、そのファイルが構造体 `p2` に読み込まれ、その `Body` 要素が画面に表示されます。

このようなプログラムをコンパイルして実行することができます。

```
$ go build wiki.go
$ ./wiki
This is a sample Page.
```

(Windowsをお使いの場合は、「`./`」を除いた「`wiki`」と入力してプログラムを実行する必要があります)。

[これまでに書いたコードはこちら](https://go.dev/doc/articles/wiki/part1.go)

## `net/http` パッケージの紹介 (幕間)

ここでは、簡単なWebサーバーの完全な動作例を紹介します。

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hi there, I love %s!", r.URL.Path[1:])
}

func main() {
    http.HandleFunc("/", handler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

この関数は `http` パッケージに対して、Web のルート (`"/"`) へのすべてのリクエストを `handler` で処理するように指示します。

次に `http.ListenAndServe` を呼び出し、任意のインターフェイス (`":8080"`) でポート 8080 をリッスンするように指定します。(2番目のパラメータである `nil` は気にしないでください。) この関数は、プログラムが終了するまでブロックされます。

`ListenAndServe` は常にエラーを返します。なぜなら、予期しないエラーが発生したときのみ、エラーを返すからです。そのエラーをログに記録するために、関数コールを `log.Fatal` でラップしています。

関数 `handler` は `http.HandlerFunc` 型である。引数として `http.ResponseWriter` と `http.Request` を受け取る。

HTTP.ResponseWriter` の値は HTTP サーバのレスポンスをまとめたものであり、これに書き込むことでHTTPクライアントにデータを送信することができる。

`HTTP.Request`は、クライアントのHTTPリクエストを表すデータ構造である。`r.URL.Path` は、リクエストURLのパスコンポーネントです。末尾の `[1:]` は、"1文字目から末尾までの `Path` のサブスライスを作成する" ことを意味する。これにより、パス名から先頭の "/" が取り除かれる。

このプログラムを実行し、URLにアクセスすると。

```
http://localhost:8080/monkeys
```

プログラムは、以下を含むページを表示します。

```
Hi there, I love monkeys!
```

## `net/http` を使って wiki ページを提供する。

`net/http` パッケージを使用するには、インポートする必要があります。

```
import (
    "fmt"
    "os"
    "log"
    "net/http"
)

```

ユーザーがwikiページを閲覧できるようにするハンドラ、 `viewHandler` を作成しましょう。これは "/view/" をプレフィックスとする URL を扱います。

```go
func viewHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/view/"):]
    p, _ := loadPage(title)
    fmt.Fprintf(w, "<h1>%s</h1><div>%s</div>", p.Title, p.Body)
}
```

ここでも、 `_` を使って `loadPage` の戻り値である `error` を無視していることに注意してください。これは単純化するために行っているのですが、一般的にはバッドプラクティスと考えられています。これについては後で説明します。

まず、この関数は `r.URL.Path` （リクエストURLのパスコンポーネント）からページタイトルを抽出します。`Path` は `[len("/view/"):]` で再スライスされ、リクエストパスの先頭の `"/view/"` コンポーネントが削除されます。これは、パスが必ず `"/view/"` で始まるためで、これはページのタイトルの一部ではありません。

この関数はページのデータを読み込み、シンプルな HTML 文字列でページをフォーマットして、`http.ResponseWriter` である `w` に書き込みます。

このハンドラを使用するために、`main` 関数を書き換えて、`viewHandler` を使用して `http` を初期化し、パス `/view/` の下にあるすべてのリクエストを処理できるようにします。

```go
func main() {
    http.HandleFunc("/view/", viewHandler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

[これまでに書いたコードを見るにはここをクリック](https://go.dev/doc/articles/wiki/part2.go)

いくつかのページデータ (`test.txt`) を作成し、コードをコンパイルして、wiki ページを表示してみましょう。

エディタで `test.txt` ファイルを開き、"Hello world" という文字列 (引用符なし) を保存してください。

```
$ go build wiki.go
$ ./wiki
```

(Windowsをお使いの場合は、プログラムを実行するために "`./`" を除いた "`wiki`" をタイプする必要があります)。

このウェブサーバが動作している状態で、`[http://localhost:8080/view/test](http://localhost:8080/view/test)`にアクセスすると、"Hello world "という言葉を含む "test "というタイトルのページが表示されるはずです。

## ページの編集

ページを編集する機能なしにはwikiはwikiではありません。1つは'editpage'フォームを表示するための`editHandler`という名前のハンドラで、もう1つはフォームを通して入力されたデータを保存するための`saveHandler`という名前のハンドラを作成しましょう。

まず、これらを `main()` に追加する。

```go
func main() {
    http.HandleFunc("/view/", viewHandler)
    http.HandleFunc("/edit/", editHandler)
    http.HandleFunc("/save/", saveHandler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```
関数 `editHandler` はページをロードし (ページが存在しない場合は空の `Page` 構造体を作成する)、HTML フォームを表示します。

```go
func editHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/edit/"):]
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    fmt.Fprintf(w, "<h1>Editing %s</h1>"+
        "<form action=\"/save/%s\" method=\"POST\">"+
        "<textarea name=\"body\">%s</textarea><br>"+
        "<input type=\"submit\" value=\"Save\">"+
        "</form>",
        p.Title, p.Title, p.Body)
}
```

この機能はうまく動作しますが、ハードコードされたHTMLはすべて醜いものです。もちろん、もっと良い方法があります。

## `html/template`パッケージ

`html/template` パッケージは Go 標準ライブラリの一部です。`html/template` を使用すると、HTML を別のファイルに保存することができ、基盤となる Go コードを変更することなく編集ページのレイアウトを変更することができます。

まず、インポートリストに `html/template` を追加する必要があります。また、もう `fmt` を使うことはないので、これを削除しなければなりません。

```go
import (
    "html/template"
    "os"
    "net/http"
)
```

HTMLフォームを含むテンプレートファイルを作成しましょう。新しいファイル `edit.html` を開き、以下の行を追加する。

```html
<h1>Editing {{.Title}}</h1>

<form action="/save/{{.Title}}" method="POST">
<div><textarea name="body" rows="20" cols="80">{{printf "%s" .Body}}</textarea></div>
<div><input type="submit" value="Save"></div>
</form>
```

ハードコードされたHTMLの代わりに、テンプレートを使用するように `editHandler` を修正します。

```go
func editHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/edit/"):]
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    t, _ := template.ParseFiles("edit.html")
    t.Execute(w, p)
}
```
関数 `template.ParseFiles` は `edit.html` の内容を読み込み、`*template.Template` を返します。

メソッド `t.Execute` はテンプレートを実行し、生成されたHTMLを `http.ResponseWriter` に書き込む。ドット付きの識別子 `.Title` と `.Body` は、それぞれ `p.Title` と `p.Body` を参照します。

テンプレートディレクティブは、二重の中括弧で囲む。`printf "%s" .Body` 命令は、 `.Body` をバイトのストリームではなく文字列として出力する関数呼び出しで、 `fmt.Printf` の呼び出しと同じです。`html/template` パッケージは、テンプレートアクションによって安全で正しい見た目の HTML だけが生成されることを保証するのに役立ちます。例えば、ユーザーデータがフォームのHTMLを破壊しないように、大なり記号 (`>`) を自動的にエスケープして、 `&gt;` に置き換えます。

ここではテンプレートを使用しているので、 `viewHandler` 用に `view.html` というテンプレートを作成しましょう。

```html
<h1>{{.Title}}</h1>

<p>[<a href="/edit/{{.Title}}">edit</a>]</p>

<div>{{printf "%s" .Body}}</div>
```

それに応じて `viewHandler` を修正します。

```go
func viewHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/view/"):]
    p, _ := loadPage(title)
    t, _ := template.ParseFiles("view.html")
    t.Execute(w, p)
}
```

両方のハンドラでほとんど同じテンプレート・コードを使用していることに注意してください。テンプレート化するコードを独自の関数に移動させることで、この重複を解消してみましょう。

```go
func renderTemplate(w http.ResponseWriter, tmpl string, p *Page) {
    t, _ := template.ParseFiles(tmpl + ".html")
    t.Execute(w, p)
}
```

そして、その関数を使用するようにハンドラを修正します。

```go
func viewHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/view/"):]
    p, _ := loadPage(title)
    renderTemplate(w, "view", p)
}
```

```go
func editHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/edit/"):]
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    renderTemplate(w, "edit", p)
}
```

`main` で未実装の保存ハンドラの登録をコメントアウトすると、再びプログラムをビルドしてテストすることができます。[これまでに書いたコードを見るにはここをクリックしてください](https://go.dev/doc/articles/wiki/part3.go)

## 存在しないページの処理

もし、[`/view/APageThatDoesntExist`](http://localhost:8080/view/APageThatDoesntExist)にアクセスしたらどうでしょうか？HTMLを含むページが表示されるでしょう。これは `loadPage` からのエラーの返り値を無視して、データがない状態でテンプレートへの記入を試み続けるからです。代わりに、要求されたページが存在しない場合、コンテンツを作成できるように、クライアントを編集ページにリダイレクトする必要があります。

```go
func viewHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/view/"):]
    p, err := loadPage(title)
    if err != nil {
        http.Redirect(w, r, "/edit/"+title, http.StatusFound)
        return
    }
    renderTemplate(w, "view", p)
}
```

`HTTP.Redirect` 関数は、HTTP ステータスコード `http.StatusFound` (302) と `Location` ヘッダーを HTTP レスポンスに追加します。

## ページの保存

関数 `saveHandler` は、編集ページに配置されたフォームの送信を処理します。`main` の関連行をアンコメントした後、ハンドラを実装してみましょう。

```go
func saveHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/save/"):]
    body := r.FormValue("body")
    p := &Page{Title: title, Body: []byte(body)}
    p.save()
    http.Redirect(w, r, "/view/"+title, http.StatusFound)
}
```

ページのタイトル (URL で提供される) とフォームの唯一のフィールドである `Body` は、新しい `Page` に格納されます。そして、 `save()` メソッドが呼び出されてデータがファイルに書き込まれ、クライアントは `/view/` ページにリダイレクトされる。

`FormValue` が返す値は `string` 型である。この値を `Page` 構造体に収めるには、 `[]byte` 型に変換する必要があります。ここでは、 `[]byte(body)` を使用して変換を行う。

## エラーハンドリング

私たちのプログラムには、エラーを無視している箇所がいくつかあります。これは悪い習慣で、特にエラーが発生したときにプログラムが意図しない動作をすることになるからです。より良い解決策は、エラーを処理して、ユーザーにエラーメッセージを返すことです。そうすれば、何か問題が発生したときに、サーバーは私たちが望むように正確に機能し、ユーザーにはそのことが伝わります。

まず、`renderTemplate` の中でエラーを処理してみましょう。

```go
func renderTemplate(w http.ResponseWriter, tmpl string, p *Page) {
    t, err := template.ParseFiles(tmpl + ".html")
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    err = t.Execute(w, p)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
    }
}
```

`HTTP.Error` 関数は、指定された HTTP レスポンスコード (この場合は "Internal Server Error") とエラーメッセージを送信します。すでに、この関数を別の関数にしたことが功を奏しています。

次に `saveHandler` を修正しましょう。

```go
func saveHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/save/"):]
    body := r.FormValue("body")
    p := &Page{Title: title, Body: []byte(body)}
    err := p.save()
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    http.Redirect(w, r, "/view/"+title, http.StatusFound)
}
```

`p.save()`の間に発生したエラーは、ユーザーに報告されます。

## テンプレートキャッシング

このコードには非効率な点があります。`renderTemplate` はページがレンダリングされるたびに `ParseFiles` を呼び出します。より良い方法は、プログラムの初期化時に一度だけ `ParseFiles` を呼び出して、すべてのテンプレートを一つの `*Template` にパースすることです。それから、[`ExecuteTemplate`](https://go.dev/pkg/html/template/#Template.ExecuteTemplate) メソッドを使用して、特定のテンプレートをレンダリングすることができます。

まず、`templates` という名前のグローバル変数を作成し、それを `ParseFiles` で初期化します。

```go
var templates = template.Must(template.ParseFiles("edit.html", "view.html"))
```

関数 `template.Must` は、nil でない `error` 値を渡されるとパニックになり、そうでなければ `*Template` を変更せずに返す便利なラッパーです。もし、テンプレートを読み込むことができなければ、プログラムを終了するのが唯一の賢明な方法です。

`ParseFiles` 関数はテンプレートファイルを特定する任意の数の文字列引数を取り、それらのファイルをベースファイル名の後に名前が付けられたテンプレートにパースします。もし、プログラムにさらにテンプレートを追加するのであれば、 `ParseFiles` 呼び出しの引数にその名前を追加することになります。

そして、 `renderTemplate` 関数を修正して、適切なテンプレートの名前を指定して `templates.ExecuteTemplate` メソッドを呼び出すようにします。

```go
func renderTemplate(w http.ResponseWriter, tmpl string, p *Page) {
    err := templates.ExecuteTemplate(w, tmpl+".html", p)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
    }
}
```

テンプレート名はテンプレートファイル名なので、 `tmpl` 引数に `".html"` を追加しなければならないことに注意してください。

## バリデーション

お気づきのように、このプログラムには重大なセキュリティ上の欠陥があります。 ユーザーがサーバー上で読み書きをするために、任意のパスを供給することができるのです。これを軽減するために、タイトルを正規表現で検証する関数を書くことができます。

まず、`import` リストに `"regexp"` を追加します。それから、グローバル変数を作成して、検証式を格納します。

```go
var validPath = regexp.MustCompile("^/(edit|save|view)/([a-zA-Z0-9]+)$")
```

関数 `regexp.MustCompile` は正規表現をパースしてコンパイルし、 `regexp.Regexp` を返します。`MustCompile` は `Compile` とは異なり、式のコンパイルに失敗するとパニックを起こしますが、 `Compile` は第2引数として `error` を返します。

それでは、 `validPath` 式を使用して、パスの検証を行い、ページのタイトルを抽出する関数を書いてみましょう。

```go
func getTitle(w http.ResponseWriter, r *http.Request) (string, error) {
    m := validPath.FindStringSubmatch(r.URL.Path)
    if m == nil {
        http.NotFound(w, r)
        return "", errors.New("invalid Page Title")
    }
    return m[2], nil
}
```

タイトルが有効な場合は、エラー値 `nil` と共に返されます。タイトルが無効な場合、この関数は HTTP 接続に "404 Not Found" エラーを書き込み、ハンドラに対してエラーを返します。新しいエラーを作成するには、`errors` パッケージをインポートする必要がある。

各ハンドラに `getTitle` の呼び出しを記述してみよう。

```go
func viewHandler(w http.ResponseWriter, r *http.Request) {
    title, err := getTitle(w, r)
    if err != nil {
        return
    }
    p, err := loadPage(title)
    if err != nil {
        http.Redirect(w, r, "/edit/"+title, http.StatusFound)
        return
    }
    renderTemplate(w, "view", p)
}
```

```go
func editHandler(w http.ResponseWriter, r *http.Request) {
    title, err := getTitle(w, r)
    if err != nil {
        return
    }
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    renderTemplate(w, "edit", p)
}
```

```go
func saveHandler(w http.ResponseWriter, r *http.Request) {
    title, err := getTitle(w, r)
    if err != nil {
        return
    }
    body := r.FormValue("body")
    p := &Page{Title: title, Body: []byte(body)}
    err = p.save()
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    http.Redirect(w, r, "/view/"+title, http.StatusFound)
}
```

## 関数リテラルとクロージャの導入

各ハンドラでエラー条件をキャッチすると、多くのコードが繰り返されることになります。各ハンドラを、この検証やエラーチェックを行う関数でラップすることができたらどうでしょうか。Goの[関数リテラル](https://go.dev/ref/spec#Function_literals)は、機能を抽象化する強力な手段で、このような場合に役に立ちます。

まず、各ハンドラの関数定義を、タイトル文字列を受け取るように書き直します。

```go
func viewHandler(w http.ResponseWriter, r *http.Request, title string)
func editHandler(w http.ResponseWriter, r *http.Request, title string)
func saveHandler(w http.ResponseWriter, r *http.Request, title string)
```

ここで、上記の型の関数を受け取り、（関数 `http.HandleFunc` に渡すのに適した） `http.HandlerFunc` 型の関数を返すラッパー関数を定義してみましょう。

```go
func makeHandler(fn func (http.ResponseWriter, *http.Request, string)) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Here we will extract the page title from the Request,
        // and call the provided handler 'fn'
    }
}
```

返された関数は、その外側で定義された値を囲むので、クロージャと呼ばれます。この場合、変数 `fn` (`makeHandler` の単一の引数) はクロージャで囲まれています。変数 `fn` には、保存、編集、または閲覧のハンドラのいずれかを指定します。

それでは、 `getTitle` のコードを（少し修正して）ここで使用することができます。

```go
func makeHandler(fn func(http.ResponseWriter, *http.Request, string)) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        m := validPath.FindStringSubmatch(r.URL.Path)
        if m == nil {
            http.NotFound(w, r)
            return
        }
        fn(w, r, m[2])
    }
}
```

`makeHandler` が返すクロージャは、 `http.ResponseWriter` と `http.Request` (言い換えれば、 `http.HandlerFunc`) を受け取る関数です。クロージャはリクエストパスから `title` を抽出し、それを `validPath` 正規表現で検証する。もし `title` が無効な場合は、 `http.NotFound` 関数を使用して `ResponseWriter` にエラーが書き込まれる。`title` が有効な場合は、 `ResponseWriter`、`Request`、`title` を引数として同梱のハンドラ関数 `fn` が呼び出される。

これで、ハンドラ関数が `http` パッケージに登録される前に、 `makeHandler` を使って `main` でラップすることができるようになりました。

```go
func main() {
    http.HandleFunc("/view/", makeHandler(viewHandler))
    http.HandleFunc("/edit/", makeHandler(editHandler))
    http.HandleFunc("/save/", makeHandler(saveHandler))

    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

最後に、ハンドラ関数から `getTitle` の呼び出しを削除して、よりシンプルにします。

```go
func viewHandler(w http.ResponseWriter, r *http.Request, title string) {
    p, err := loadPage(title)
    if err != nil {
        http.Redirect(w, r, "/edit/"+title, http.StatusFound)
        return
    }
    renderTemplate(w, "view", p)
}
```

```go
func editHandler(w http.ResponseWriter, r *http.Request, title string) {
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    renderTemplate(w, "edit", p)
}
```

```go
func saveHandler(w http.ResponseWriter, r *http.Request, title string) {
    body := r.FormValue("body")
    p := &Page{Title: title, Body: []byte(body)}
    err := p.save()
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    http.Redirect(w, r, "/view/"+title, http.StatusFound)
}
```

## ぜひお試しください。

[最終的なコード一覧を見るにはここをクリック](https://go.dev/doc/articles/wiki/final.go)

コードを再コンパイルし、アプリを実行します。

```
$ go build wiki.go
$ ./wiki
```

[http://localhost:8080/view/ANewPage](http://localhost:8080/view/ANewPage) にアクセスすると、ページ編集フォームが表示されるはずです。テキストを入力し、「保存」をクリックすると、新しく作成されたページに移動することができます。

## その他のタスク

自分で取り組みたいと思うかもしれないいくつかの簡単なタスクは次のとおりです。

-テンプレートを`tmpl/`に保存し、ページデータを`data/`に保存します。
-ハンドラーを追加して、Webルートを `/ view/FrontPage`にリダイレクトします。
-有効なHTMLにし、いくつかのCSSルールを追加して、ページテンプレートを整えます。
-`[PageName]`のインスタンスをに変換することにより、ページ間リンクを実装します
   `<a href="/view/PageName">PageName</a>`。 （ヒント：これを行うには、 `regexp.ReplaceAllFunc`を使用できます）

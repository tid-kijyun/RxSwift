実例
========

1. [計算済み変数](#計算済み変数)
1. [シンプルなUIバインディング](#シンプルなuiバインディング)
1. [オートコンプリート](#オートコンプリート)
1. [more examples](../RxExample)
1. [Playgrounds](Playgrounds.md)

## 計算済み変数

それでは命令型のSwiftコードから始めましょう。  
この例の目的はいくつかの条件が満たされた場合に `a` と `b` から計算される値を `c` にバインドすることです。

ここに `c` の値を計算する命令型のSwiftコードがあります:

```swift
// this is usual imperative code
var c: String
var a = 1       // `a` に `1` を一度だけ代入する
var b = 2       // `b` に `2` を一度だけ代入する

if a + b >= 0 {
    c = "\(a + b) is positive" // `c` に値を一度だけ代入する
}
```

今、`c` の値は `3 is positive` です。  
もし `a` を `4` に変更したとしても、`c` は古い値を保持したままです。

```swift
a = 4           // c は "3 is positive" と等しいまま、これは良くない。
                // c は "6 is positive" と等しくなければならない、なぜなら 4 + 2 = 6 だから
```

これは求めている振る舞いではありません。

RxSwiftフレームワークをあなたのプロジェクトに統合するには、  
あなたのプロジェクトにフレームワークを含めて`import RxSwift`と書くだけです。

これはRxSwiftを使用した同じロジックです。

```swift
let a /*: Observable<Int>*/ = Variable(1)   // a = 1
let b /*: Observable<Int>*/ = Variable(2)   // b = 2

// これは `a` と `b` を `+` を使用して結合した最新の値を rx変数`c` に "bind" する:
// if a + b >= 0 {
//      c = "\(a + b) is positive"
// }
let c = Observable.combineLatest(a.asObservable(), b.asObservable()) { $0 + $1 }
	.filter { $0 >= 0 }         // もし `a + b >= 0` が true なら、`a + b` はmapオペレーターに渡される
	.map { "\($0) is positive" } // `a + b` を "\(a + b) is positive" にマップする

// 初期値が a = 1, b = 2 なら
// 1 + 2 = 3 で >= 0 なので、 `c` は最初 "3 is positive" と等しくなる

// Rx変数`c` から値を引き出す、`c` から値を subscribe する。
// `subscribeNext` は 変数`c`のnext値を subscribe するという意味。
// それはまた初期値 "3 is positive" を含んでいる。
c.subscribeNext { print($0) }          // 表示: "3 is positive"

// `a` の値を増やしてみる
// RxSwiftで a = 4
a.value = 4                            // 表示: 6 is positive
// 最新の値の合計は `4 + 2`, `6` は >= 0 なので、
// mapオペレーターは "6 is positive" を生成して、結果を `c` に "アサイン(assigned)" する。
// `c` の値は更新されているので、`{ print($0) }` は呼び出される。
// そして "6 is positive" が表示される。

// `b` の値を変えてみる
// RxSwiftで b = -8
b.value = -8                           // 何も表示されない
// 最新の値の合計は `4 + (-8)`, `-4` は >= 0 ではないので、map は実行されない。
// これは `c` がまだ "6 is positive" を含有していることを意味します。
// `c` は更新されていないので、それは次の値が生成されていないことを意味する。
// また `{ print($0) }` は呼び出されない。

// ...
```

## シンプルなUIバインディング

* 変数にバインドする代わりに、Textフィールドの値(rx_text)にバインドしてみましょう
* 次に、非同期APIを使用して数値が素数の場合に整数を解析して計算します(map)
* もし text フィールド値が非同期呼び出し完了後に更新されたら、新しい非同期呼び出しをエンキューします(concat)
* 結果をlabelにバインドします(bindTo(resultLabel.rx_text))

```swift
let subscription/*: Disposable */ = primeTextField.rx_text    // 型は Observable<String>
            .map { WolframAlphaIsPrime(Int($0) ?? 0) }        // 型は Observable<Observable<Prime>>
            .concat()                                         // 型は Observable<Prime>
            .map { "number \($0.n) is prime? \($0.isPrime)" } // 型は Observable<String>
            .bindTo(resultLabel.rx_text)                      // 全てをバインド解除するのに使用できるDisposable を返す

// これはサーバーコールが完了した後に resultLabel.text に "number 43 is prime? true" を設定する
primeTextField.text = "43"

// ...

// 全てバインド解除するには、これを呼ぶだけ
subscription.dispose()
```

この例で使用したすべてのオペレーターは、最初の変数の例で使用したのと同じオペレーターです。  
特別なものは何もありません。

## オートコンプリート

もしあなたがRxに慣れていなければ、おそらく次の例に少し圧倒されます。  
しかしそれは現実的な例でRxコードがどう見えるかを示します。

3番目の例は現実的で、プログレス通知と組み合わせた複雑なUIの非同期バリデーションロジックです。

全ての操作は `disposeBag` が解放された瞬間キャンセルされます。

それではやってみましょう。

```swift
// UIコントロール値をバインドして直接
// ユーザー名のソースとして `usernameOutlet` から ユーザー名を使用する
self.usernameOutlet.rx_text
    .map { username in

        // 同期バリテーション、ここでは特別なことはない
        if username.isEmpty {
            // 同期結果を構築するためのコンビニエンス。
            // 同期と非同期コードが同じメソッドに混在している場合、これは直ちに解決された非同期の結果を構築する
            return Observable.just((valid: false, message: "ユーザー名は空にできません"))
        }

        ...

        // 全てのユーザーインターフェースはおそらく非同期操作を実行中にいくつかの状態を表す。
        // 私達は結果を待っている間、 "使用できるか確認中" と表示したいと仮定する。 
        // 有効なパラメーターは下記とする
        //  * true  - 有効
        //  * false - 無効
        //  * nil   - バリデーション保留中
        typealias LoadingInfo = (valid : String?, message: String?)
        let loadingValue : LoadingInfo = (valid: nil, message: "使用できるか確認中 ...")

        // これはユーザー名が既に存在するか確認するためのサーバーコールを発行する。
        // 型は `Observable<ValidationResult>` 。
        return API.usernameAvailable(username)
          .map { available in
              if available {
                  return (true, "ユーザー名は使用可能です")
              }
              else {
                  return (false, "ユーザー名は既に使用されています")
              }
          }
          // use `loadingValue` until server responds
          .startWith(loadingValue)
    }
// 私達は `Observable<Observable<ValidationResult>>` を持っているので、
// 何とかして通常の `Observable` の世界に戻る必要がある。
// 私たちは2番目の例から `concat` オペレーターを使用できる。
// しかしもし新しいユーザー名が提供されたら保留中の非同期操作をどうしてもキャンセルしたい。
// `switchLatest` はそれを行う。
    .switchLatest()
// 今、私達はこれを何とかしてユーザーインターフェースにバインドする必要がある。
// 古き良き `subscribeNext` はそれをすることができる。
// それは`Observable` チェーンの終わりである。
// これは全てのバインドを解除して保留中の非同期操作をキャンセルする `Disposable` オブジェクトを生成する。
    .subscribeNext { valid in
        errorLabel.textColor = validationColor(valid)
        errorLabel.text = valid.message
    }
// なぜ私達はそれを手動で行うのだろう、それは冗長だ、
// ビューコントローラーの dealloc で自動的にすべて dispose しよう。
    .addDisposableTo(disposeBag)
```

これより簡単に取得することはできません。  
リポジトリに[更なる例](../RxExample)があります。気軽にチェックアウトしてください。

それらにはMVVMパターンのコンテキストでの使い方またはその他の例が含まれています。

ユニット
=====

このドキュメントはユニットとは何であるか、なぜ有用な概念なのか、どのように作成して使用するかを記述します。

* [なぜ](#なぜ)
* [それらはどう働くのか](#それらはどう働くのか)
* [なぜそれらはユニットと名付けられたか](#なぜそれらはユニットと名付けられたか)
* [RxCocoa units](#rxcocoa-units)
* [Driver units](#driver-unit)
    * [なぜDriverと命名されたのか](#なぜdriverと命名されたのか)
    * [実用的な使用例](#実用的な使用例)

## なぜ

Swiftは強力な型システムを持っていて、それは正確さとアプリケーションの安定性を向上させるために使用できます、  
またRxを使用することで更に直感的で簡単な体験を作ります。

**Unitはその点で[RxCocoa](https://github.com/ReactiveX/RxSwift/tree/master/RxCocoa)プロジェクトだけに特化していますが、  
  必要なら他のRx実装に同じ原理を簡単に実装することができます。非公開APIの魔法は必要ありません。**

**Unitは完全にオプションです。あなたはあなたのプログラムのどこでも生のobservableシーケンスを使えますし、  
  全てのRxCocoa APIはobservableシーケンスと共に働きます。**

Unitはまたコミュニケーションを支援します。  
そしてインターフェースの境界を超えてobservableシーケンスプロパティを保証します。

これらはCocoa/UIKitアプリケーションを書く時に重要なプロパティの一部です。

* エラー出力できない
* main scheduler上でobserveする
* main scheduler上でsubscribeする
* 副作用を共有

## それらはどう働くのか

そのコアはobservableシーケンスを参照するただの構造体です。

observableシーケンスのためのBuilderパターンの一種と考えることができます。  
シーケンスが構築されると、`.asObservable()`が呼ばれてユニットをバニラobservableシーケンスに変換します。


## なぜそれらはユニットと名付けられたか

馴染みのない概念について類推する助けとして、  
物理とRxCocoa(rx units)がどのように類似しているかについていくつかのアイデアを以下に列挙します。

類似:

| 物理 units                      | Rx units                                                            |
|-------------------------------------|---------------------------------------------------------------------|
| number (one value)                  | observable sequence (sequence of values)                            |
| 次元の単位 (m, s, m/s, N ...) | Swift構造体 (Driver, ControlProperty, ControlEvent, Variable, ...) |

物理単位は数のペアと対応する次元の単位です。  
Rx unitはobservableシーケンスのペアと対応する構造体で、それはobservableシーケンスプロパティを記述します。

> Numbers are the basic composition glue when working with physical units: usually real or complex numbers.<br/>
Observable sequences are the basic composition glue when working with rx units.

物理単位で作業するときの数は基本組成の接着剤です: 通常、実数または複素数。  
rx unitsで作業するときのObservableシーケンスは基本組成の接着剤です。

物理単位と[次元解析](https://en.wikipedia.org/wiki/Dimensional_analysis#Checking_equations_that_involve_dimensions)は複雑な計算中の論理エラーの特定を楽にすることができます。  
rx unitの型チェックはリアクティブプログラムを書く時の論理エラーの特定を楽にすることができます。

実数が持つ演算子: `+`, `-`, `*`, `/`<br/>
Observableシーケンスが持つ演算子: `map`, `filter`, `flatMap` ...

物理単位は対応する数の演算を使用して演算を定義します。

例.

物理単位上の`/`演算は、数字の`/`演算を使用して定義されています。

11 m / 0.5 s = ...
* 最初に単位を数値に変換します、そして`/` **演算** を **適用します**。 `11 / 0.5 = 22`
* それから単位を計算します (m / s)
* 結果を組み合わせます = 22 m / s

Rx unitは対応するobservableシーケンス演算を使用して演算を定義します。  
(これはオペレーターが内部でどのように動作するかです。)

例.

`Driver`上の`map`演算をobservableシーケンス上の`map`演算を使用して定義します。

```swift
let d: Driver<Int> = Drive.just(11)
driver.map { $0 / 0.5 } = ...
```

* 最初にdriverを **observableシーケンス** に変換します、そして`map` **演算** を **適用します**。

```swift
let mapped = driver.asObservable().map { $0 / 0.5 } // この`map`はobservableシーケンスを定義している
```

* それから単位値を取得して組み合わせます

```swift
let result = Driver(mapped)
```

物理には直交する基本的な単位のセット[(`m`, `kg`, `s`, `A`, `K`, `cd`, `mol`)](https://en.wikipedia.org/wiki/SI_base_unit)があります。  
`RxCocoa`には直交するobservableシーケンスのための基本的な興味深いプロパティのセットがあります。

    * エラー出力できない :can't error out
    * main scheduler上でobserveする :observe on main scheduler
    * main scheduler上でsubscribeする :subscribe on main scheduler
    * 副作用を共有 :sharing side effects

物理の派生単位は時々特別な名前を持ちます。<br/>
例.

```
N (ニュートン: Newton) = kg * m / s / s
C (クーロン: Coulomb) = A * s
T (テスラ: Tesla) = kg / A / s / s
```

Rxの派生unitもまた特別な名前を持ちます。<br/>
例.

```
Driver = (can't error out) * (observe on main scheduler) * (sharing side effects)
ControlProperty = (sharing side effects) * (subscribe on main scheduler)
Variable = (can't error out) * (sharing side effects)
```

物理の異なる単位間の変換は、数字 `*`, `/`上で定義された演算子の助けを借りて行われます。
RXの異なる単位の変換は、observableシーケンスオペレーターの助けを借りて行われます。

例.

```
can't error out = catchError
observe on main scheduler = observeOn(MainScheduler.instance)
subscribe on main scheduler = subscribeOn(MainScheduler.instance)
sharing side effects = share* (one of the `share` operators)
```


## RxCocoa units

### Driver unit

* can't error out
* observe on main scheduler
* sharing side effects (`shareReplayLatestWhileConnected`)

### ControlProperty / ControlEvent

* can't error out
* subscribe on main scheduler
* observe on main scheduler
* sharing side effects

### Variable

* can't error out
* sharing side effects

## Driver

これは最も凝ったunitです。これの意図はUIレイヤーでリアクティブなコードを書く直感的な方法を提供することです。

### なぜDriverと命名されたのか

It's intended use case was to model sequences that drive your application.
意図したユースケースはシーケンスをモデル化してあなたのアプリケーションを駆動することでした。

例.
* CoreDataモデルからUIを駆動
* 他のUI要素の値を使用してUIを駆動(バインディング)
...


通常のオペレーティングシステムドライバのように、  
いずれかのシーケンスでエラーが起きた場合はあなたのアプリケーションはユーザー入力への応答を停止します。

また極めて重要なことはそれらの要素がメインスレッドでobserveされることです、  
なぜならUI要素とアプリケーションロジックは通常スレッドセーフではありません。

また、`Driver` unitは副作用を共有するobservableシーケンスをビルドします。

例.

### 実用的な使用例

これは典型的な初心者の例です。

```swift
let results = query.rx_text
    .throttle(0.3, scheduler: MainScheduler.instance)
    .flatMapLatest { query in
        fetchAutoCompleteItems(query)
    }

results
    .map { "\($0.count)" }
    .bindTo(resultCount.rx_text)
    .addDisposableTo(disposeBag)

results
    .bindTo(resultsTableView.rx_itemsWithCellIdentifier("Cell")) { (_, result, cell) in
        cell.textLabel?.text = "\(result)"
    }
    .addDisposableTo(disposeBag)
```

このコードの意図した動作は:

* ユーザー入力を絞る
* サーバーへコンタクトしてユーザーリストの結果を取得する(クエリ毎に一度)
* それから2つのUI要素に結果をバインドし、テーブルビューとラベルに結果と結果の数を表示する

このコードの何が問題かというと:

* `fetchAutoCompleteItems` observableシーケンスがエラー(接続失敗やパースエラー)を出した場合、  
  そのエラーはすべてをバインド解除し、UIは新しいクエリにこれ以上応答しないでしょう。
* いくつかのバックグラウンドスレッドで`fetchAutoCompleteItems`が結果を返した場合、  
  結果はバックグラウンドスレッドでUI要素にバインドされ、それは非決定性のクラッシュを引き起こすでしょう。
* 結果は2つのUI要素にバインドされます、これは各々のユーザークエリに対して  
  2つのHTTPリクエストが行われることを意味します。これは意図している動作ではありません。

もっと適切なバージョンのコードはこんな感じです:

```swift
let results = query.rx_text
    .throttle(0.3, scheduler: MainScheduler.instance)
    .flatMapLatest { query in
        fetchAutoCompleteItems(query)
            .observeOn(MainScheduler.instance) // results are returned on MainScheduler
            .catchErrorJustReturn([])                // in worst case, errors are handled
    }
    .shareReplay(1)                                  // HTTP requests are shared and results replayed
                                                     // to all UI elements

results
    .map { "\($0.count)" }
    .bindTo(resultCount.rx_text)
    .addDisposableTo(disposeBag)

results
    .bindTo(resultTableView.rx_itemsWithCellIdentifier("Cell")) { (_, result, cell) in
        cell.textLabel?.text = "\(result)"
    }
    .addDisposableTo(disposeBag)
```

Making sure all of these requirements are properly handled in large systems can be challenging, but there is a simpler way of using the compiler and units to prove these requirements are met.
大きなシステムで、これら全ての要求が適切にハンドルされることを確かめるのは困難な場合があります。  
しかしコンパイラを使う簡単な方法がありますし、unitはこれらの要件を満たしていることが証明されています。

次のコードはほとんど同ように見えます:

```swift
let results = query.rx_text.asDriver()        // This converts normal sequence into `Driver` sequence.
    .throttle(0.3, scheduler: MainScheduler.instance)
    .flatMapLatest { query in
        fetchAutoCompleteItems(query)
            .asDriver(onErrorJustReturn: [])  // Builder just needs info what to return in case of error.
    }

results
    .map { "\($0.count)" }
    .drive(resultCount.rx_text)               // If there is `drive` method available instead of `bindTo`,
    .addDisposableTo(disposeBag)              // that means that compiler has proved all properties
                                              // are satisfied.
results
    .drive(resultTableView.rx_itemsWithCellIdentifier("Cell")) { (_, result, cell) in
        cell.textLabel?.text = "\(result)"
    }
    .addDisposableTo(disposeBag)
```

ここで何が起こっているのでしょう？

最初に `asDriver`メソッドで `ControlProperty` unit を `Driver` unit に変換しています。

```swift
query.rx_text.asDriver()
```

注意として、これをするのに特別なことは何も必要ありません。  
`Driver`は`ControlProperty` unit の全てに加えてもう少しのプロパティを持っています。  
根底にあるobservableシーケンスがただ`Driver` unitとしてラップされているだけです。

２つ目の変更は

```swift
  .asDriver(onErrorJustReturn: [])
```

任意のobservableシーケンスは `Driver` unit に変換できます。それは３項目だけ満たす必要があります:

* エラーを出さない
* main schedulerでobserveする
* 副作用を共有する (`shareReplayLatestWhileConnected`)

それでは、これらの特性が満たされていることを確認するにはどうすればいいでしょう？  
通常のRxオペレーターを使うだけです。  
次のコードは`asDriver(onErrorJustReturn: [])`と同等です。

```
let safeSequence = xs
  .observeOn(MainScheduler.instance) // main schedulerでイベントをobserveする
  .catchErrorJustReturn(onErrorJustReturn) // エラーを出さない
  .shareReplayLatestWhileConnected         // 副作用を共有する
return Driver(raw: safeSequence)           // 上記をラップする
```

最後のピースは`bindTo`の代わりに`drive`を使用することです。

`Driver` unit上でだけ `drive`は定義されています。  
これはもし`drive`をコードのどこかで見たとしたら、observableシーケンスは決してエラーを出さず、  
observeした要素はメインスレッド上でUI要素にバインドされることを意味します。  
これはまさに望まれていることです。

理論的には、だれもが`ObservableType`上またはいくつかの他のインターフェースで働く`driver`メソッドを定義できます。  
完全な証明が必要になる前にUI要素をバインドする一時的な定義を作成します。 `let results: Driver<[Results]> = ...`  
しかし我々はそれが現実的なシナリオであるかを決定することは読者のために残しておきます。

入門
===============

このプロジェクトは[ReactiveX.io](http://reactivex.io/)と調和しようとします。
一般的なクロスプラットフォームのマニュアルとチュートリアルは`RxSwift`でも有効です。

1. [Observablesまたの名をSequences](#observablesまたの名をsequences)
1. [Disposing](#disposing)
1. [暗黙的な `Observable` の保証](#暗黙的な-observable-の保証)
1. [初めて作る `Observable` (またの名を observable sequence)](#初めて作る-observable-またの名を-observable-sequence)
1. [仕事をする `Observable` を作る](#仕事をする-observable-を作る)
1. [共有サブスクリプションと `shareReplay` オペレーター](#共有サブスクリプションと-sharereplay-オペレーター)
1. [オペレーター](#オペレーター)
1. [Playgrounds](#playgrounds)
1. [カスタムオペレーター](#カスタムオペレーター)
1. [エラーハンドリング](#エラーハンドリング)
1. [コンパイルエラーをデバッグする](#コンパイルエラーをデバッグする)
1. [デバッグする](#デバッグする)
1. [メモリリークをデバッグする](#メモリリークをデバッグする)
1. [KVO](#kvo)
1. [UIレイヤーのコツ](#uiレイヤーのコツ)
1. [HTTPリクエストを作る](#httpリクエストを作る)
1. [RxDataSources](#rxdatasources)
1. [Driver](Units.md#driver-unit)
1. [実例](Examples.md)

# Observablesまたの名をSequences

## 基本
オブサーバーパターン(`Observable<Element>`)の等価性とシーケンス(`Generator`s)はRxを理解するために最も重要なものの一つです。

オブサーバーパターンは非同期の振る舞いをモデル化するのに必要とされています。
その等価性は`Observable`の操作のような高レベルなシーケンス操作の実装を可能にします。

シーケンスはシンプルで、 **可視化するのが簡単な** おなじみの概念です。

人は大きな視覚野を持つ生き物です。私たちはコンセプトを簡単に視覚化できた時、それについて推論することがより容易になります。

すべてのRxオペレーター内部のシーケンスの高レベル操作上のイベントステートマシンをシミュレートしようとすることによる多くの認知的負荷を取り除くことができます。
もしRxを使わないで非同期システムをモデル化するとしたら、それはおそらく私達のコードは完全なステートマシンとテンポラリステートを持ち、
抽象化を脇に置いて代わりにシミュレートする必要があることを意味します。

リストとシーケンスはおそらく数学者とプログラマが最初に学ぶ概念です。

ここに番号を振ったシーケンスがあります:

```
--1--2--3--4--5--6--| // 正常終了
```

他に文字を振ったシーケンスがあります:

```
--a--b--a--a--a---d---X // エラーで終了
```

いくつかのシーケンスは有限でその他は無限です。ボタンタップのシーケンスのように:

```
---tap-tap-------tap--->
```

これらはマーブルダイアグラムと呼ばれます。もっと知りたければ[rxmarbles.com](http://rxmarbles.com)を参照してください。

もし正規表現でシーケンス文法を指定するとしたらこうなるでしょう:
**Next* (Error | Completed)?**

これは下記のことを述べています:

* **シーケンスは0以上の要素を持つことができる**
* **一度 `Error` または `Completed` イベントを受信すると、シーケンスは他の要素を作り出すことができない**

Rxのシーケンスはプッシュインターフェースによって記述されます。(通称コールバック)

```swift
enum Event<Element>  {
    case Next(Element)      // next element of a sequence
    case Error(ErrorType)   // sequence failed with error
    case Completed          // sequence terminated successfully
}

class Observable<Element> {
    func subscribe(observer: Observer<Element>) -> Disposable
}

protocol ObserverType {
    func on(event: Event<Element>)
}
```

**シーケンスが `Complete` または `Error` イベントを送信するとシーケンス要素を計算しすべての内部リソースを解放します。**

**シーケンス要素の生産をキャンセルしてすぐにリソースを解放するには、返されたサブスクリプションで`dispose`を呼び出してください。**

シーケンスが有限時間で終了するなら、`dispose` をコールしたり `addDisposableTo(disposeBag)` を使用しなくても永続的なリソースリークが発生することはありません。
しかしながら、それらのリソースはシーケンスが完了するか要素の生産を終了するかエラーを返すまで利用されます。

もしシーケンスがいずれかの方法で終了しない場合、手動で`dispose`するか、自動的に内部の`disposeBag`、`takeUntil`が呼び出されるか、または他の方法を用いない限りリソースは永久に割り当てられたままになるでしょう。

**dispose bagsまたは`takeUntil`操作を用いるのはリソースのクリーンアップを確実にする堅牢な方法です。
プロダクションではシーケンスが有限時間で終了する場合でもそれらを使うことを推奨します。**

なぜ`ErrorType`がジェネリックでないのか興味があるなら、[ここ](DesignRationale.md#why-error-type-isnt-generic)で説明を見つけられます。

## 破棄する -Disposing-

オブサーブされたシーケンスを終了することができる一つの追加の方法があります。
シーケンスを実行した時、または今後の要素を計算するために割り当てられた全てのリソースを解放したい時に、
サブスクリプションに`dispose`を呼び出すことができます。

これは`interval`オペレーターの例です。

```swift
let subscription = Observable<Int>.interval(0.3, scheduler: scheduler)
    .subscribe { event in
        print(event)
    }

NSThread.sleepForTimeInterval(2)

subscription.dispose()

```

これが表示されます:

```
0
1
2
3
4
5
```

注意として通常は手動で`dispose`を実行する必要はありません。これは教育のためだけの例です。
手動でdisposeを呼び出すと通常悪いコードの臭いがします。
サブスクリプションをdisposeするもっと良い方法があります。
`DisposeBag`、`takeUntil`オペレーター、または他の機構を使うことです。

それでは、このコードは`dispose`を呼んで実行した後に何かを表示できるでしょうか？
答えはケースバイケースです。

* もし`scheduler`が **シリアルスケジューラ** (例. `MainScheduler`)である、<br>または`dispose`が **同じシリアルスケジューラ上で**実行された場合、答えは **no** です。
* その他なら **yes** です。

スケジューラについて更に知りたいなら[ここ](Schedulers.md)で見つけられます。

あなたは単に並列に発生している2つのプロセスを持っています。

* 一つの要素を生成しています
* 他はサブスクリプションをdisposeしています

このケースでは質問"後で何か表示できるか？"は意味がありません、プロセスは別のスケジューラ上にあります。

念のため[ここ](Schedulers.md)に`observeOn`を説明した例が少しあります。

この場合はこんな感じです:

```swift
let subscription = Observable<Int>.interval(0.3, scheduler: scheduler)
            .observeOn(MainScheduler.instance)
            .subscribe { event in
                print(event)
            }

// ....

subscription.dispose() // メインスレッドから呼ばれる

```

**`dispose`呼び出しが返った後は何も表示されません。それは保証されています。**

やはり、この場合でも:

```swift
let subscription = Observable<Int>.interval(0.3, scheduler: scheduler)
            .observeOn(serialScheduler)
            .subscribe { event in
                print(event)
            }

// ...

subscription.dispose() // 同じ`serialScheduler`で実行している

```

**`dispose`呼び出しが返った後は何も表示されません。それは保証されています。**

### Dispose Bags

RxでDispose bagsはARCのような振る舞いに用いられます。

`DisposeBag`が解放される時、それはそれぞれ追加されたdisposableの`dispose`を呼び出します。

これは`dispose`メソッドを持っておらず従って故意に明示的なdiposeの呼び出しはできません。
もし直接クリーンアップする必要があるなら、新しいbagを作成することができます。

```swift
  self.disposeBag = DisposeBag()
```

これは古い参照のクリアしリソースを破棄を引き起こします。

もしまだ明示的な手動の破棄を望んでいるなら、`CompositeDisposable`を使ってください。
**これは望みどおりの動作をしますが、一度`dispose`メソッドが呼ばれるとそれはすぐに新たに追加した任意のdisposableを破棄します。**

### Take until

追加の方法としてsubscriptionを自動的に破棄して解放するのに`takeUntil`オペレーターが使えます。

```swift
sequence
    .takeUntil(self.rx_deallocated)
    .subscribe {
        print($0)
    }
```

## 暗黙的な `Observable` の保証

全てのシーケンス作成者(`Observable`s)がもっとも尊重しなければならない幾つかの追加の保証があります。

それはスレッドで要素を生成するのに重要ではありませんが、
もし一つの要素を作成してオブサーバーに`observer.on(.Next(nextElement))`を送信しても、
`observer.on`メソッドの実行が終わるまで次の要素に送信できません。

作成者はまた、この場合には`.Next`イベントが終了するまで`.Completed` または `.Error`の終了を送信できません。

手短に、この例で考察してください:

```swift
someObservable
  .subscribe { (e: Event<Element>) in
      print("Event processing started")
      // processing
      print("Event processing ended")
  }
```

これが常に表示されます:

```
Event processing started
Event processing ended
Event processing started
Event processing ended
Event processing started
Event processing ended
```

決してこうは表示されません:

```
Event processing started
Event processing started
Event processing ended
Event processing ended
```

## 初めて作る `Observable` (またの名を observable sequence)

observableについて理解するために一つの重要なことがあります。

**observableが作られた時、それは作られただけなのでいかなる簡単な仕事も行いません。**

たしかに`Observable`は多くの方法で作成できます。
そのうち幾つかは副作用を引き起こし、そのうちいくつかはマウスイベントのタッピングなどのように既存の実行中プロセスに入り込みます。

**ただし`Observable`を返すメソッドを呼んだだけでは、一切シーケンスの生成は行われず副作用もありません。  
`Observable`はどのようにシーケンスを生成するのかと要素の生成のために使われるパラメータが何かを定義するだけです。  
シーケンスの生成が開始されるのは｀subscribe`メソッドを呼んだ時です。**

例. 例えば同様のプロトタイプを持つメソッドがあるとします。

```swift
func searchWikipedia(searchTerm: String) -> Observable<Results> {}
```

```swift
let searchForMe = searchWikipedia("me")

// 何の要求も実行されない、何の仕事もしない、何のURLリクエストも発っさない
let cancel = searchForMe
  // シーケンスの生成をすぐ始める、URLリクエストが発行される
  .subscribeNext { results in
      print(results)
  }

```

多くの方法であなた自身の`Observable`シーケンスを作ることができます。
おそらく最も簡単な方法は`create`関数を使うことです。

サブスクリプションに際して一つの要素を返すシーケンスを作る関数を作ってみましょう。
この関数を'just'と呼びます。

*これは実際の実装です*

```swift
func myJust<E>(element: E) -> Observable<E> {
    return Observable.create { observer in
        observer.on(.Next(element))
        observer.on(.Completed)
        return NopDisposable.instance
    }
}

myJust(0)
    .subscribeNext { n in
      print(n)
    }
```

これが表示されます:

```
0
```

悪くないですね。でも`create`関数とは何でしょう？

これはSwiftのクロージャを用いて`subscribe`メソッドを簡単に実装することができるただの便利なメソッドです。  
`subscribe`メソッドと同様に一つの引数と`observer`を取り、disposableを返します。

> Sequence implemented this way is actually synchronous. It will generate elements and terminate before `subscribe` call returns disposable representing subscription. Because of that it doesn't really matter what disposable it returns, process of generating elements can't be interrupted.

この方法で実装したシーケンスは実際に同期します。
それは要素を生成し、`subscribe`を呼び出してsubscriptionを表すdisposableを返した後に終了します。
そのことからこれは本当に問題です、それが返すdisposableは生成した要素の処理を中断できません。

同期的なシーケンスを生成する時、一般的なdisposableは`NopDisposable`のシングルトンインスタンスを返します。


これから配列から要素を返すobservableを作ってみましょう。

*これは実際の実装です*

```swift
func myFrom<E>(sequence: [E]) -> Observable<E> {
    return Observable.create { observer in
        for element in sequence {
            observer.on(.Next(element))
        }

        observer.on(.Completed)
        return NopDisposable.instance
    }
}

let stringCounter = myFrom(["first", "second"])

print("Started ----")

// 初回
stringCounter
    .subscribeNext { n in
        print(n)
    }

print("----")

// もう一度
stringCounter
    .subscribeNext { n in
        print(n)
    }

print("Ended ----")
```

これが表示されます:

```
Started ----
first
second
----
first
second
Ended ----
```

## 仕事をする `Observable` を作る

それではもっと面白くしていきます。前回の例を用いて`interval`オペレーターを作成してみましょう。

*これはディスパッチキュースケジューラと同等の実際の実装です。*

```swift
func myInterval(interval: NSTimeInterval) -> Observable<Int> {
    return Observable.create { observer in
        print("Subscribed")
        let queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)
        let timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue)

        var next = 0

        dispatch_source_set_timer(timer, 0, UInt64(interval * Double(NSEC_PER_SEC)), 0)
        let cancel = AnonymousDisposable {
            print("Disposed")
            dispatch_source_cancel(timer)
        }
        dispatch_source_set_event_handler(timer, {
            if cancel.disposed {
                return
            }
            observer.on(.Next(next))
            next += 1
        })
        dispatch_resume(timer)

        return cancel
    }
}
```

```swift
let counter = myInterval(0.1)

print("Started ----")

let subscription = counter
    .subscribeNext { n in
       print(n)
    }

NSThread.sleepForTimeInterval(0.5)

subscription.dispose()

print("Ended ----")
```

これが表示されます:

```
Started ----
Subscribed
0
1
2
3
4
Disposed
Ended ----
```

こう書くとどうなるでしょう

```swift
let counter = myInterval(0.1)

print("Started ----")

let subscription1 = counter
    .subscribeNext { n in
       print("First \(n)")
    }
let subscription2 = counter
    .subscribeNext { n in
       print("Second \(n)")
    }

NSThread.sleepForTimeInterval(0.5)

subscription1.dispose()

NSThread.sleepForTimeInterval(0.5)

subscription2.dispose()

print("Ended ----")
```

こう表示されるでしょう:

```
Started ----
Subscribed
Subscribed
First 0
Second 0
First 1
Second 1
First 2
Second 2
First 3
Second 3
First 4
Second 4
Disposed
Second 5
Second 6
Second 7
Second 8
Second 9
Disposed
Ended ----
```

**サブスクリプションする際にすべてのサブスクライバーは個別のシーケンスの要素を通常は生成します。  
オペレーターはデフォルトではステートレスです。ステートフルよりも非常に多くのステートレスなオペレーターがあります。**


## 共有サブスクリプションと `shareReplay` オペレーター

ではもし複数のobserverで一つだけのsubscriptionからのイベント(要素)を共有したいならどうしますか？

２つのものを定義する必要があります。

* observeしているそれら(最新のリプレイだけ、すべてのリプレイ、最後のn個のリプレイ)の中で  
  新しいサブスクライバーが興味を持つ過去の要素をハンドルする方法
* 共有したサブスクリプションを発行するタイミングを決定する方法(参照カウント、手動またはいくらかの他のアルゴリズム)

通常の選択は `replay(1).refCount()` 別名`shareReplay()` を組み合わせることです。

```swift
let counter = myInterval(0.1)
    .shareReplay(1)

print("Started ----")

let subscription1 = counter
    .subscribeNext { n in
       print("First \(n)")
    }
let subscription2 = counter
    .subscribeNext { n in
       print("Second \(n)")
    }

NSThread.sleepForTimeInterval(0.5)

subscription1.dispose()

NSThread.sleepForTimeInterval(0.5)

subscription2.dispose()

print("Ended ----")
```

これが表示されます:

```
Started ----
Subscribed
First 0
Second 0
First 1
Second 1
First 2
Second 2
First 3
Second 3
First 4
Second 4
First 5
Second 5
Second 6
Second 7
Second 8
Second 9
Disposed
Ended ----
```

今は`Subscribed`と`Disposed`イベントが一つだけあることに注目してください。

URL observableと同等の振る舞いです。

これはRxでHTTPリクエストをラップする方法です。これは`interval`オペレーターにかなり似ているパターンです。

```swift
extension NSURLSession {
    public func rx_response(request: NSURLRequest) -> Observable<(NSData, NSURLResponse)> {
        return Observable.create { observer in
            let task = self.dataTaskWithRequest(request) { (data, response, error) in
                guard let response = response, data = data else {
                    observer.on(.Error(error ?? RxCocoaURLError.Unknown))
                    return
                }

                guard let httpResponse = response as? NSHTTPURLResponse else {
                    observer.on(.Error(RxCocoaURLError.NonHTTPResponse(response: response)))
                    return
                }

                observer.on(.Next(data, httpResponse))
                observer.on(.Completed)
            }

            task.resume()

            return AnonymousDisposable {
                task.cancel()
            }
        }
    }
}
```

## オペレーター

RxSwiftには多数のオペレーターが実装されています。完全なリストは[ここ](API.md)で見ることができます。

全てのオペレーターのマーブルダイアグラムは[ReactiveX.io](http://reactivex.io/)で見ることができます。

ほとんど全てのオペレーターのデモは[Playgrounds](../Rx.playground)の中にあります。

Playgroundを使うときは`Rx.xcworkspace`を開いて、`RxSwift-OSX`スキーマをビルドしてから`Rx.xcworkspace`ツリービューのPlaygroundを開いてください。

もしオペレーターが必要な場合に、その見つけ方が分からないなら[オペレーターの決定木](http://reactivex.io/documentation/operators.html#tree)があります。

[RxSwiftがサポートしているオペレーター](API.md#rxswift-supported-operators)ではそれらを実行する機能別にグループ分けしているので助けになります。

### カスタムオペレーター

2つの方法でカスタムオペレーターを作成することができます。

#### 簡単な方法

全ての内部コードは非常に最適化されたバージョンのオペレーターを使用します、
そのためそれらはチュートリアルに最適な題材ではありません。
それが標準的なオペレーターを使用することを大いに推奨する理由です。

幸運なことに簡単にオペレーターを作る方法があります。
新しいオペレーターを作ることは実際は全てobservableを作ることで、前章で既にその方法を記述しました。

最適化されていないmapオペレーターを実装できる方法を見てみましょう。

```swift
extension ObservableType {
    func myMap<R>(transform: E -> R) -> Observable<R> {
        return Observable.create { observer in
            let subscription = self.subscribe { e in
                    switch e {
                    case .Next(let value):
                        let result = transform(value)
                        observer.on(.Next(result))
                    case .Error(let error):
                        observer.on(.Error(error))
                    case .Completed:
                        observer.on(.Completed)
                    }
                }

            return subscription
        }
    }
}
```

このとおり、あなたのmapを使うことができます:

```swift
let subscription = myInterval(0.1)
    .myMap { e in
        return "This is simply \(e)"
    }
    .subscribeNext { n in
        print(n)
    }
```

そしてこれが表示されます

```
Subscribed
This is simply 0
This is simply 1
This is simply 2
This is simply 3
This is simply 4
This is simply 5
This is simply 6
This is simply 7
This is simply 8
...
```

### 人生いろいろあるけどなんとかなるさ -Life happens-

カスタムオペレーターで解決するのが難しいいくつかの場合はどうしましょう？  
あなたはRxモナドを終了して、命令的な世界の中でアクションを実行し、それからRxの結果のトンネルはまた`Subject`を使用できます。

ほとんど実践することありませんし、悪いコードの臭いがしますがこれを実行することができます。

```swift
  let magicBeings: Observable<MagicBeing> = summonFromMiddleEarth()

  magicBeings
    .subscribeNext { being in     // Rxモナドを終了  
        self.doSomeStateMagic(being)
    }
    .addDisposableTo(disposeBag)

  //
  // ごちゃごちゃ
  //
  let kitten = globalParty(   // ごちゃごちゃな世界で何か計算する
    being,
    UIApplication.delegate.dataSomething.attendees
  )
  kittens.on(.Next(kitten))   // 結果をRxに送り戻す
  //
  // 別のごちゃごちゃ
  //

  let kittens = Variable(firstKitten) // Rxモナドにまた戻る

  kittens.asObservable()
    .map { kitten in
      return kitten.purr()
    }
    // ....
```

あなたがこれを行うたびに、おそらく誰かがどこかにこのコードを書くでしょう

```swift
  kittens
    .subscribeNext { kitten in
      // so something with kitten
    }
    .addDisposableTo(disposeBag)
```

これは実行しないようにしてください。

## Playgrounds

もし一部のオペレーターの正確な働きに自信がなければ、
[playgrounds](../Rx.playground)にほとんど全てのオペレーターの振る舞いをイラスト化する小さな例が含まれています。

**Playgroundを使うときはRx.xcworkspaceを開いて、RxSwift-OSXスキーマをビルドしてからRx.xcworkspaceツリービューのPlaygroundを開いてください。**

**playgroundの例の結果を見るには、`Assistant Editor`を開いてください。
`Assistant Editor`は`View > Assistant Editor > Show Assistant Editor`をクリックすれば開きます。**

## エラーハンドリング

2つのエラー機構があります。

### observable内の非同期エラーハンドリング機構

エラーハンドリングは非常に簡単です。もし一つのシーケンスがエラーで終了したら、
そのあと全ての依存するシーケンスがエラーで終了します。それは通常短い論理回路です。

catchオペレーターを使うことで失敗したobservableから復帰することができます。
復帰の詳細を指定できるいろいろなオーバーロードがあります。

またエラーシーケンスの場合にはリトライを可能にする`retry`オペレーターもあります。

## コンパイルエラーをデバッグする

エレガントなRxSwift/RxCocoaコードを書く時、  
おそらく`Observable`の型を推定するのにコンパイラに大きく依存するでしょう。  
それはSwiftがすばらしい理由の一つですが、時々イライラさせられます。

```swift
images = word
    .filter { $0.containsString("important") }
    .flatMap { word in
        return self.api.loadFlickrFeed("karate")
            .catchError { error in
                return just(JSON(1))
            }
      }
```

もしコンパイラがこの式のどこかにエラーがあると報告してきたら、  
私は最初の戻り値の型に注釈をつけることをお勧めします。

```swift
images = word
    .filter { s -> Bool in s.containsString("important") }
    .flatMap { word -> Observable<JSON> in
        return self.api.loadFlickrFeed("karate")
            .catchError { error -> Observable<JSON> in
                return just(JSON(1))
            }
      }
```

もしこれが動かなければ、エラーの場所を突き止めるまで更に注釈をつけることを続けてください。

```swift
images = word
    .filter { (s: String) -> Bool in s.containsString("important") }
    .flatMap { (word: String) -> Observable<JSON> in
        return self.api.loadFlickrFeed("karate")
            .catchError { (error: NSError) -> Observable<JSON> in
                return just(JSON(1))
            }
      }
```

**私は最初に戻り値の型とクロージャの引数に注釈をつけることをお勧めします。**

通常エラーが解決した後、再びコードを綺麗にするため型の注釈を削除できます。

## デバッグする

単独でデバッガを使用すると便利です、しかし通常`debug`オペレーターを使用するとより効果的になります。  
`debug`オペレーターは全てのイベントを標準出力に出力します、そしてあなたはそれらのイベントにラベルを付けることができます。

`debug`はプローブのように作用します。これはそれを使った例です:

```swift
let subscription = myInterval(0.1)
    .debug("my probe")
    .map { e in
        return "This is simply \(e)"
    }
    .subscribeNext { n in
        print(n)
    }

NSThread.sleepForTimeInterval(0.5)

subscription.dispose()
```

こう表示されます

```
[my probe] subscribed
Subscribed
[my probe] -> Event Next(Box(0))
This is simply 0
[my probe] -> Event Next(Box(1))
This is simply 1
[my probe] -> Event Next(Box(2))
This is simply 2
[my probe] -> Event Next(Box(3))
This is simply 3
[my probe] -> Event Next(Box(4))
This is simply 4
[my probe] dispose
Disposed
```

また、簡単にあなたのバージョンの`debug`オペレーターを作成できます。

```swift
extension ObservableType {
    public func myDebug(identifier: String) -> Observable<Self.E> {
        return Observable.create { observer in
            print("subscribed \(identifier)")
            let subscription = self.subscribe { e in
                print("event \(identifier)  \(e)")
                switch e {
                case .Next(let value):
                    observer.on(.Next(value))

                case .Error(let error):
                    observer.on(.Error(error))

                case .Completed:
                    observer.on(.Completed)
                }
            }
            return AnonymousDisposable {
                   print("disposing \(identifier)")
                   subscription.dispose()
            }
        }
    }
 }
```

## メモリリークをデバッグする

デバッグモードではRxは全ての確保したリソースをグローバル変数`resourceCount`で追跡しています。

いくつかのリソースリークを検出するロジックが欲しい場合、
最もシンプルな方法は定期的に`RxSwift.resourceCount`を出力することです。

```swift
    /* add somewhere in
    func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject: AnyObject]?) -> Bool
    */
    _ = Observable<Int>.interval(1, scheduler: MainScheduler.instance)
        .subscribeNext { _ in
        print("Resource count \(RxSwift.resourceCount)")
    }
```

メモリリークをテストする最も効果的な方法は:

* 画面に移動してそれを使用します
* navigate back
* 初期リソース数を観察します
* 画面に2度目の移動してそれを使用します
* navigate back
* 最後のリソース数を観察します

最初と最後でリソースカウントに差があった場合、どこかにメモリリークがあるかもしれません。

2つのナビゲーションを提案している理由は最初のナビゲーションでlazyリソースが強制的に読み込まれるからです。

## 変数

`変数`はいくつかのobservableの状態を表しています。  
値を含まない`変数`は存在しません。なぜならイニシャライザーに初期値が必要だからです。

変数は[`Subject`](http://reactivex.io/documentation/subject.html)をラップします。もっと具体的にはそれは`BehaviorSubject`です。  
`BehaviorSubject`と異なり、それは値のインターフェースを晒すだけです、だから変数は決して終了または失敗できません。

それはまたサブスクリプション上の現在値をすぐにブロードキャストします。

変数が解放された後、`.asObservable()`から返されたobservableシーケンスを完了します。

```swift
let variable = Variable(0)

print("Before first subscription ---")

_ = variable.asObservable()
    .subscribe(onNext: { n in
        print("First \(n)")
    }, onCompleted: {
        print("Completed 1")
    })

print("Before send 1")

variable.value = 1

print("Before second subscription ---")

_ = variable.asObservable()
    .subscribe(onNext: { n in
        print("Second \(n)")
    }, onCompleted: {
        print("Completed 2")
    })

variable.value = 2

print("End ---")
```

こう表示されます

```
Before first subscription ---
First 0
Before send 1
First 1
Before second subscription ---
Second 1
First 2
Second 2
End ---
Completed 1
Completed 2
```

## KVO

KVOはObjective-Cの機構です。それは型安全性を念頭に置いて作られていないことを意味します。
このプロジェクトは問題の一部を解決しようとします。

このライブラリでは2つのビルドインの方法でKVOをサポートしています。

```swift
// KVO
extension NSObject {
    public func rx_observe<E>(type: E.Type, _ keyPath: String, options: NSKeyValueObservingOptions, retainSelf: Bool = true) -> Observable<E?> {}
}

#if !DISABLE_SWIZZLING
// KVO
extension NSObject {
    public func rx_observeWeakly<E>(type: E.Type, _ keyPath: String, options: NSKeyValueObservingOptions) -> Observable<E?> {}
}
#endif
```

どのように`UIView`のframeを監視するかの例です。

**WARNING: UIKitはKVOに準拠していませんがこれは動きます。**

```swift
view
  .rx_observe(CGRect.self, "frame")
  .subscribeNext { frame in
    ...
  }
```

または

```swift
view
  .rx_observeWeakly(CGRect.self, "frame")
  .subscribeNext { frame in
    ...
  }
```

### `rx_observe`

`rx_observe`はより効率が良いです、なぜならこれはKVO機構のただのシンプルなラッパーだからです。
しかし使用するシナリオによっては制限があります。

* it can be used to observe paths starting from `self` or from ancestors in ownership graph (`retainSelf = false`)
* it can be used to observe paths starting from descendants in ownership graph (`retainSelf = true`)
* the paths have to consist only of `strong` properties, otherwise you are risking crashing the system by not unregistering KVO observer before dealloc.

例.

```swift
self.rx_observe(CGRect.self, "view.frame", retainSelf: false)
```

### `rx_observeWeakly`

`rx_observeWeakly`は`rx_observe`よりやや遅い、なぜなら弱参照の場合にオブジェクトの解放をハンドルする必要があるからです。

これは`rx_observe`を使用できる全ての場合で使用することができ、さらに

* because it won't retain observed target, it can be used to observe arbitrary object graph whose ownership relation is unknown
* it can be used to observe `weak` properties

例.

```swift
someSuspiciousViewController.rx_observeWeakly(Bool.self, "behavingOk")
```

### 構造体を監視する

KVOはObjective-Cの機構で、それは`NSValue`に大きく依存しています。

**RxCocoaには`CGRect`, `CGSize` それに `CGPoint` 構造体のKVOサポートが組み込まれています。**

いくつかの他の構造体を監視する場合には、手動で`NSValue`から構造体を抽出する必要があります。

[ここ](../RxCocoa/Common/KVORepresentable+CoreGraphics.swift)にどのようにKVO機構を拡張するか、またどのように`KVORepresentable` プロトコルを実装することによって`rx_observe*`メソッドを他の構造体に使うかの例があります。

## UIレイヤーのコツ

UIKitコントロールにバインドする時に、あなたの`Observable`がUIレイヤーで満たす必要がある一定の物事があります。

### スレッディング

`Observable`は`MainScheduler`(UIThread)で値を送信する必要があります。それはまさに通常のUIKit/Cocoaが要求していることです。

あなたのAPIが`MainScheduler`で結果を返すのは通常良いアイデアです。
あなたがバックグラウンドスレッドで何かをUIにバインドしようとした場合、**デバッグ**ビルドのRxCocoaは通常そのことを通知する例外を投げます。

修正するには`observeOn(MainScheduler.instance)`を追加する必要があります。

**デフォルトではNSURLSessionの拡張は結果を`MainScheduler`で返しません。**

### エラー

あなたはUIKitコントロールに失敗をバインドできません、なぜならそれは未定義の動作だからです。

`Observable`が失敗する可能性があるかわからない場合、`catchErrorJustReturn(valueThatIsReturnedWhenErrorHappens)`を使用して失敗しないことを保証することができますが、**配下のシーケンスでエラーが発生した後でも完了(complete)します。**

配下のシーケンスでの望ましい動作が要素の生成を続けることなら、いくつかの`retry`オペレーターのバージョンが必要です。

### サブスクリプションを共有する

通常はUIレイヤーでサブスクリプションを共有したいです。
複数のUI要素に同一のデータをバインドするのに、別のHTTP呼び出しは行いたくありません。

このようにしたとしましょう:

```swift
let searchResults = searchText
    .throttle(0.3, $.mainScheduler)
    .distinctUntilChanged
    .flatMapLatest { query in
        API.getSearchResults(query)
            .retry(3)
            .startWith([]) // clears results on new search term
            .catchErrorJustReturn([])
    }
    .shareReplay(1)              // <- `shareReplay`オペレーターに注意
```

通常一回で計算された検索結果を共有したいでしょう。これが`shareReplay`が意図することです。

**通常UIレイヤーの変換チェーンの最後に`shareReplay`を追加するのは大雑把ですが良いやりかたです、  
  なぜならあなたは本当に計算結果を共有したいからです。  
  `searchResults`を複数のUI要素にバインドするたびに個別のHTTPコネクションを発行したくはないでしょう。**

**また、`Driver` ユニットを見てみましょう。それは`shareReplay`呼び出しを透過的にラップするよう設計されており、  
  要素はメインUIスレッドで監視されエラー無しでUIにバインドできます。**

## HTTPリクエストを作る

HTTPリクエストを作ることは皆が最初に試してみるモノの一つです。

最初に必要なのは必要な作業を行う`NSURLRequest`オブジェクトを構築することです。

決めることを求められるのは、それはGETリクエストまたはPOSTリクエストか、リクエスト本体、クエリパラメーターなど...

これはシンプルなGETリクエストを作る方法です

```swift
let request = NSURLRequest(URL: NSURL(string: "http://en.wikipedia.org/w/api.php?action=parse&page=Pizza&format=json")!)
```

もし別のobservableとの組み合わせの外でリクエストを実行したいなら、これを行う必要があります。

```swift
let responseJSON = NSURLSession.sharedSession().rx_JSON(request)

// no requests will be performed up to this point
// `responseJSON` is just a description how to fetch the response

let cancelRequest = responseJSON
    // this will fire the request
    .subscribeNext { json in
        print(json)
    }

NSThread.sleepForTimeInterval(3)

// if you want to cancel request after 3 seconds have passed just call
cancelRequest.dispose()

```

**デフォルトではNSURLSessionの拡張は結果を`MainScheduler`で返しません。**

この場合、レスポンスへのもっと低レベルなアクセスが欲しいでしょう、使用できます:

```swift
NSURLSession.sharedSession().rx_response(myNSURLRequest)
    .debug("my request") // this will print out information to console
    .flatMap { (data: NSData!, response: NSURLResponse!) -> Observable<String> in
        if let response = response as? NSHTTPURLResponse {
            if 200 ..< 300 ~= response.statusCode {
                return just(transform(data))
            }
            else {
                return failWith(yourNSError)
            }
        }
        else {
            rxFatalError("response = nil")
            return failWith(yourNSError)
        }
    }
    .subscribe { event in
        print(event) // if error happened, this will also print out error to console
    }
```

### HTTPトラフィックをロギングする

デフォルトではデバッグモードのRxCocoaは全てのHTTPリクエストのログをコンソールに出力します。
この場合あなたはその動作を変えたいでしょう、`Logging.URLRequests`フィルターを設定してください。

```swift
// read your own configuration
public struct Logging {
    public typealias LogURLRequest = (NSURLRequest) -> Bool

    public static var URLRequests: LogURLRequest =  { _ in
    #if DEBUG
        return true
    #else
        return false
    #endif
    }
}
```

## RxDataSources

これは`UITableView`と`UICollectionView`に完全に機能するリアクティブデータソースを実装したクラスのセットです。

RxDataSourcesは[ここ](https://github.com/RxSwiftCommunity/RxDataSources)にバンドルされています。

それらをどうやって使うかの完全に機能するデモは[RxExample](../RxExample)プロジェクトに含まれています。

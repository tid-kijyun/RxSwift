スケジューラー -Schedulers-
==========

1. [シリアル vs コンカレント スケジューラー](#シリアル-vs-コンカレント-スケジューラー)
1. [カスタムスケジューラー](#カスタムスケジューラー)
1. [組み込みスケジューラー](#組み込みスケジューラー)

スケジューラーは作業を実行するための抽象化メカニズムです。

作業を実行するための異なるメカニズムは、カレントスレッド、ディスパッチキュー、オペレーションキュー、新しいスレッド、スレッドプール、ループ実行...などを含みます。

スケジューラーと連携する2つの主要なオペレーターがあります。`observeOn` と `subscribeOn` です。

もし別のスケジューラーで作業を実行したければ `observeOn(scheduler)` オペレーターを使用するだけです。

あなたは通常 `subscribeOn` よりも頻繁に `observeOn` を使用するでしょう。

`observeOn` が明示的に指定されていない場合、作業は常にスレッド/スケジューラー要素を生成して実行されます。

`observeOn` オペレーターの使用例です

```
sequence1
  .observeOn(backgroundScheduler)
  .map { n in
      print("これはバックグラウンドスケジューラーで実行されます")
  }
  .observeOn(MainScheduler.instance)
  .map { n in
      print("これはメインスケジューラーで実行されます")
  }
```

もし指定したスケジューラー上でシーケンスの生成を開始(`subscribe` メソッド)してdisposeを呼び出したいなら `subscribeOn(scheduler)` を使います。

`subscribeOn`が明示的に指定されない場合、`subscribeNext` または `subscribe` が呼ばれたら`subscribe`メソッドは同じスレッド/スケジューラー上で呼び出されます。

`subscribeOn`が明示的に指定されない場合、破棄が開始されたら`dispose`メソッドは同じスレッド/スケジューラー上で呼び出されます。

要するに、スケジューラーを明示的に選ばなければ、それらのメソッドはカレントスレッド/スケジューラー上で呼び出されます。

# シリアル vs コンカレント スケジューラー

スケジューラーは何でもできますので、全てのオペレーターのシーケンス変換は、追加の[暗黙的な保証](GettingStarted.md#暗黙的な-observable-の保証)を守る必要があります、それは如何なるスケジューラーを作るのにも重要です。

スケジューラーがコンカレントな場合、Rxの `observeOn` と `subscribeOn` オペレーターは必ず全て完璧に動作します。

もしRxがシリアルだと証明できるいくつかのスケジューラーを使うとしたら、それは追加の最適化を実行できるでしょう。

これまでのところ、ディスパッチキュースケジューラーだけにそれらの最適化を行っています。

シリアルディスパッチキュースケジューラーの`observeOn`の場合、ただ単純に`dispatch_async`を呼ぶだけの最適化が行われています。

# カスタムスケジューラー

現在のスケジューラーの他に、あなた自身のスケジューラーを書くことができます。

すぐに作業を実行する必要があって記述したいなら、あなた自身のスケジューラーを`ImmediateScheduler`プロトコルで実装できます。

```swift
public protocol ImmediateScheduler {
    func schedule<StateType>(state: StateType, action: (/*ImmediateScheduler,*/ StateType) -> RxResult<Disposable>) -> RxResult<Disposable>
}
```

時間ベースの操作をサポートしている新しいスケジューラーを作成したいなら、実装する必要があります。

```swift
public protocol Scheduler: ImmediateScheduler {
    associatedtype TimeInterval
    associatedtype Time

    var now : Time {
        get
    }

    func scheduleRelative<StateType>(state: StateType, dueTime: TimeInterval, action: (StateType) -> RxResult<Disposable>) -> RxResult<Disposable>
}
```

スケジューラーが定期的なスケジューリング能力のみをもつ場合、`PeriodicScheduler`プロトコルで実装することをRxに通知できます。

```swift
public protocol PeriodicScheduler : Scheduler {
    func schedulePeriodic<StateType>(state: StateType, startAfter: TimeInterval, period: TimeInterval, action: (StateType) -> StateType) -> RxResult<Disposable>
}
```

スケジューラーが`PeriodicScheduling`能力をサポートしない場合、  
Rxは透過的に定期的なスケジューリングをエミュレートします。

# 組み込みスケジューラー

Rxは全てのタイプのスケジューラーを使えますが、  
スケジューラーがシリアルだと証明されているならそれはまたいくつかの追加の最適化を実行できます。

これらは現在サポートされているスケジューラーです。

## CurrentThreadScheduler (シリアルスケジューラー)

カレントスレッド上で作業するスケジューラーユニット。  
これは要素を生成するオペレーターのデフォルトのスケジューラーです。

このスケジューラーは時々`trampoline scheduler`(トランポリンスケジューラー)と呼ばれます。

いくつかのスレッド上で最初に`CurrentThreadScheduler.instance.schedule(state) { }`を呼び出したら、  
スケジュールされたアクションはすぐに実行されます。  
そして全ての再帰的にスケジュールされたアクションを一時的にエンキューする隠れたキューが作成されます。

コールスタック上のいくつかの親フレームが既に`CurrentThreadScheduler.instance.schedule(state) { }`を実行していたなら、  
現在実行中のアクションと全ての前もってエンキューされたアクションの実行が完了したらすぐにスケジュールされたアクションはエンキューされて実行されます。

## MainScheduler (シリアルスケジューラー)

`MainThread`上で実行される必要がある作業を抽象化します。  
メインスレッドから`schedule` メソッドが呼ばれた場合、スケジューリング無しですぐにアクションを実行します。

このスケジューラーは通常、UIの作業を実行するために使われます。

## SerialDispatchQueueScheduler (シリアルスケジューラー)

> It will make sure that even if concurrent dispatch queue is passed, it's transformed into a serial one.

特定の`dispatch_queue_t`上で実行される必要がある作業を抽象化します。  
それはたとえコンカレントディスパッチキューが渡されたとしてもシリアルに変換することを確認します。

シリアルスケジューラーは`observeOn`のための特定の最適化を有効にします。

メインスケジューラーは`SerialDispatchQueueScheduler`のインスタンスです。

## ConcurrentDispatchQueueScheduler (コンカレントスケジューラー)

特定の`dispatch_queue_t`上で実行される必要がある作業を抽象化します。  
あなたはまたシリアルディスパッチキューを渡すことができます、それは何の問題も起きません。

いくつかの作業をバックグラウンドで実行する必要がある時はこのスケジューラーが適切です。

## OperationQueueScheduler (コンカレントスケジューラー)

特定の`NSOperationQueue`上で実行される必要がある作業を抽象化します。

このスケジューラーは、バックグラウンドで実行する必要があるいくつかの作業の巨大なチャンクがある時、  
または`maxConcurrentOperationCount`を使用して同時処理を細かくチューニングしたい時に最適です。

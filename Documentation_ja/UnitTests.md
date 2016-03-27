ユニットテスト
==========

## カスタムオペレーターをテストする

ライブラリーは全てのRxSwiftオペレーターのテストで`RxTests`を使います、  
したがって`Rx.xcworkspace`プロジェクト内のAllTests-*ターゲットを見てみることができます。

これは典型的な`RxSwift`オペレータのユニットテストの実例です:

```swift
func testMap_Range() {
        // テストスケジューラを初期化
        // テストスケジューラはローカルマシンのクロックから切り離された仮想時間を実装している
        // それは可能な限り高速にシミュレーションを実行し、
        // 全てのイベントがハンドルされていることを証明できます。
        let scheduler = TestScheduler(initialClock: 0)

        // モックhot observableシーケンスを作る。
        // シーケンスは以下の時間にイベントを発行し
        // いくつかのobserverがsubscribeしていても構いません。
        // (これがhotの意味です)
        // このobservableシーケンスは生存期間中に行われた全てのsubscriptionを記録します。
        // (`subscriptions`プロパティ)
        let xs = scheduler.createHotObservable([
            next(150, 1),  // 1番目の引数は仮想時間、2番目の引数は要素の値
            next(210, 0),
            next(220, 1),
            next(230, 2),
            next(240, 4),
            completed(300) // 送信が完了までの仮想時間
            ])

        // `start`メソッドはデフォルトでは:
        // * シミュレーションを実行し、`res`によって参照したobserverを使用して全てのイベントを記録します。
        // * 仮想時間200でsubscribe
        // * 仮想時間1000でsubscriptionをdispose
        let res = scheduler.start { xs.map { $0 * 2 } }

        let correctMessages = [
            next(210, 0 * 2),
            next(220, 1 * 2),
            next(230, 2 * 2),
            next(240, 4 * 2),
            completed(300)
        ]

        let correctSubscriptions = [
            Subscription(200, 300)
        ]

        XCTAssertEqual(res.events, correctMessages)
        XCTAssertEqual(xs.subscriptions, correctSubscriptions)
    }
```

## オペレーターの組み合わせをテストする(view models, components)

どのようにオペレーターの組み合わせをテストするかの実例は  
`Rx.xcworkspace`内の`RxExample-iOSTests`ターゲットに含まれます。

あなたが読みやすい方法であなたのテストを書くことができますので、簡単に`RxTests`の拡張を定義します。

提供した`RxExample-iOSTests`に含まれている実例は、  
どのようにそれらの拡張を書くことができるかを示す一部にすぎませんが、  
それらのテストを書くための多くの可能性があります。

```swift
    // イベントを予期する、テストデータ
    let (
        usernameEvents,
        passwordEvents,
        repeatedPasswordEvents,
        loginTapEvents,

        expectedValidatedUsernameEvents,
        expectedSignupEnabledEvents
    ) = (
        scheduler.parseEventsAndTimes("e---u1----u2-----u3-----------------", values: stringValues).first!,
        scheduler.parseEventsAndTimes("e----------------------p1-----------", values: stringValues).first!,
        scheduler.parseEventsAndTimes("e---------------------------p2---p1-", values: stringValues).first!,
        scheduler.parseEventsAndTimes("------------------------------------", values: events).first!,

        scheduler.parseEventsAndTimes("e---v--f--v--f---v--o----------------", values: validations).first!,
        scheduler.parseEventsAndTimes("f--------------------------------t---", values: booleans).first!
    )
```

## 統合テスト

`RxBlocking`オペレーターを使用して統合テストを書くことも可能です。

`RxBlocking`ライブラリからオペレーターをインポートすると、  
現在のスレッドのブロックを有効にしてシーケンスの結果を待ちます。

```swift
let result = try fetchResource(location)
        .toBlocking()
        .toArray()

XCTAssertEqual(result, expectedResult)
```

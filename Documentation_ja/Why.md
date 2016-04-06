RxSwiftドキュメント日本語化 (1)Why
## なんのために
**Rxは宣言的な方法によるアプリ作成を可能にします。**

### バインディング


```swift
Observable.combineLatest(firstName.rx_text, lastName.rx_text) { $0 + " " + $1 }
            .map { "Greeting \($0)" }
            .bindTo(greetingLabel.rx_text)
```

これは `UITableView` と `UICollectionView` にも当てはまります。

```swift
viewModel
  .rows
  .bindTo(resultsTableView.rx_itemsWithCellIdentifier("WikipediaSearchCell", cellType: WikipediaSearchCell.self)) { (_, viewModel, cell) in
    cell.title = viewModel.title
    cell.url = viewModel.url
  }
  .addDisposableTo(disposeBag)
```

**常に `.addDisposableTo(disposeBag)` を使用することを提案します。単純なバインディングで必要がなくてもです。**

### リトライ

APIは失敗しないでもらえるとありがたいですが、残念ながら失敗します。
例えばこんなAPIメソッドがあったとします。

```swift
func doSomethingIncredible(forWho: String) throws -> IncredibleThing
```

このまま使おうとすると、失敗した場合のリトライが本当に難しいです。
[exponential backoffs](https://en.wikipedia.org/wiki/Exponential_backoff)をモデル化する複雑さは言うまでもありません。
もちろん可能ですが、おそらくコードはあなたが気にしなくてよいたくさんの一時的な状態を含むでしょう。
そしてそれは再利用できません。

あなたは理想的にリトライの本質を捉えるために、任意の操作を適用できるようにしたいでしょう。
これはRxでシンプルにリトライする方法です。

```swift
doSomethingIncredible("me")
  .retry(3)
```

また、簡単にカスタムリトライ演算子を作成できます。

### デリゲート

退屈で非表現的にやるとこうなります、

```swift
public func scrollViewDidScroll(scrollView: UIScrollView) { // どのScrollViewにバインドされているの？
    self.leftPositionConstraint.constant = scrollView.contentOffset.x
}
```

代わりにこう書けます。

```swift
self.resultsTableView
  .rx_contentOffset
  .map { $0.x }
  .bindTo(self.leftPositionConstraint.rx_constant)
```

### KVO

これと

```
まだキー値オブサーバーに登録しているのに、 `TickTock` が解放されます。
監視情報はリークしたり、誤った他のオブジェクトにアタッチされることがあります。
```

これ

```objc
-(void)observeValueForKeyPath:(NSString *)keyPath
                     ofObject:(id)object
                       change:(NSDictionary *)change
                      context:(void *)context
```

の代わりに[`rx_observe` と `rx_observeWeakly`](GettingStarted.md#kvo)を使います。

このように使用します。

```swift
view.rx_observe(CGRect.self, "frame")
    .subscribeNext { frame in
        print("Got new frame \(frame)")
    }
```

または

```swift
someSuspiciousViewController
    .rx_observeWeakly(Bool.self, "behavingOk")
    .subscribeNext { behavingOk in
        print("Cats can purr? \(behavingOk)")
    }
```

### Notifications

こうする代わりに

```swift
    @available(iOS 4.0, *)
    public func addObserverForName(name: String?, object obj: AnyObject?, queue: NSOperationQueue?, usingBlock block: (NSNotification) -> Void) -> NSObjectProtocol
```

こう書けます。

```swift
    NSNotificationCenter.defaultCenter()
      .rx_notification(UITextViewTextDidBeginEditingNotification, object: myTextView)
      .map { /*do something with data*/ }
      ....
```

### 一時的な状態

非同期なプログラムを書くとき一時的な状態にもたくさんの問題があります。
典型的な例はオートコンプリート検索ボックスです。

Rx無しでオートコンプリートのコードを書くとき、最初の問題は `abc` の `c` が入力された時に解決すべきことで、
`ab` に対する保留中のリクエストがあり、保留中の要求はキャンセルされます。
オーケー、保留中であることを保持する追加の変数だけ作成すれば解決することは難しくありません。

次の問題はリクエストが失敗した時に乱雑なリトライロジックを実行する必要があることです。
しかし大丈夫、複数のフィールドのカップルはリトライ回数をキャッチして、クリーンアップする必要があります。

誰かが高速に長いタイプをした時に我々をスパムにしないために、プログラムがサーバーにリクエストを発行した後にいくらかの時間待機することは素晴らしいことです。
もしかしたら追加のタイマーフィールドがいるかもしれませんね。

また検索実行中にもスクリーンに何かを表示する必要がある問題もありますし、全てのリトライが失敗した場合にもまた何かを表示する必要があります。

これらを全て書くと適切なテストが面倒になります。
これは同じロジックをRxで書いたものです。

```swift
  searchTextField.rx_text
    .throttle(0.3, scheduler: MainScheduler.instance)
    .distinctUntilChanged()
    .flatMapLatest { query in
        API.getSearchResults(query)
            .retry(3)
            .startWith([]) // clears results on new search term
            .catchErrorJustReturn([])
    }
    .subscribeNext { results in
      // bind to ui
    }
```

追加のフラグや必要なフィールドはありません。Rxが全ての乱雑なテンポラリの世話をします。

### 組成処分 -Compositional disposal-

仮にテーブルビューにブラー画像を表示するシナリオがあるとしましょう。

その画像は最初にURLから読み込んでデコードしてブラーをかける必要があります。

帯域幅とブラー処理時間は高価なので、セルが可視テーブルビュー領域から離れた場合に
全てのプロセスをキャンセルできるとよいでしょう。

ユーザーは本当に素早くスワイプしてたくさんのリクエスト要求とキャンセルを行うので、
セルが可視エリア入ってすぐには画像の読み込みを開始しないとよいでしょう。

画像にブラーをかけるのは高価な処理なので、同時に処理する画像の数に制限をかけられるとよいでしょう。

これはそれをRxを使って行う方法です。

```swift
// this is conceptual solution
let imageSubscription = imageURLs
    .throttle(0.2, scheduler: MainScheduler.instance)
    .flatMapLatest { imageURL in
        API.fetchImage(imageURL)
    }
    .observeOn(operationScheduler)
    .map { imageData in
        return decodeAndBlurImage(imageData)
    }
    .observeOn(MainScheduler.instance)
    .subscribeNext { blurredImage in
        imageView.image = blurredImage
    }
    .addDisposableTo(reuseDisposeBag)
```

このコードは全てを行います。`imageSubscription` は処分されると全ての依存する非同期操作をキャンセルし
不正な画像がUIにバインドされていないことを確認します。

### 通信リクエストを集約する

もし2つのリクエストを発行する必要があり、両方の結果を集約する場合はどうするのでしょう？

もちろん `zip` 演算子があります。

```swift
  let userRequest: Observable<User> = API.getUser("me")
  let friendsRequest: Observable<Friends> = API.getFriends("me")

  Observable.zip(userRequest, friendsRequest) { user, friends in
      return (user, friends)
    }
    .subscribeNext { user, friends in
        // bind them to user interface
    }
```

もしこれらのAPIがバックグラウンドスレッドで結果を返す場合、メインUIスレッドでバインドするにはどうしたらいいでしょう？
`observeOn` です。

```swift
  let userRequest: Observable<User> = API.getUser("me")
  let friendsRequest: Observable<[Friend]> = API.getFriends("me")

  Observable.zip(userRequest, friendsRequest) { user, friends in
      return (user, friends)
    }
    .observeOn(MainScheduler.instance)
    .subscribeNext { user, friends in
        // bind them to user interface
    }
```

Rxが本当に輝くより多くの実践的なユースケースがあります。

### 状態

変更可能な言語はグローバルな状態にアクセスし変更することが簡単にできます。
制御されていない共有のグローバルな状態の変更は簡単に[組み合わせの爆発](https://en.wikipedia.org/wiki/Combinatorial_explosion#Computing)を引き起こします。

しかし一方で、スマートな方法を使用した場合、命令型言語はハードウェアに近くさらに効率的なコードを書くことができます。

組み合わせの爆発と戦う通常の方法は、状態をできるだけシンプルに保ち、単方向のデータの流れを派生データのモデル化に使うことです。

これはRxが本当に輝く時です。

Rxは機能と命令の世界の間のスイートスポットです。
それは信頼できる構成可能な方法で、可変状態のスナップショットを処理するために不変の定義と純粋な関数を使用することができます。

いくつかの実用的な例はなんでしょう？

### 簡単な統合

もしあなた自身のobservableを作成する必要がある場合になにをすればよいでしょうか？それは非常に簡単です。
このコードはRxCocoaから取ってきた`NSURLSession`のHTTPリクエストをラップするのに必要な全てです。

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

### 恩恵

要するに、Rxを使うとあなたのコードはこうなります。

* 組み立て可能 <- なぜなら、Rxはコンポジション(組み立て)のニックネームだから。
* 再利用可能 <- なぜなら、組み立て可能だから。
* 宣言的 <- なぜなら、定義は不変でデータのみ変更するから。
* 自明で完結 <- 抽象化のレベルを上げ、一時的な状態を削除するから。
* 安定 <- なぜなら、Rxコードは徹底的にユニットテストされるから。
* レス ステートフル <- なぜなら、あなたはアプリケーションを単方向データフローとしてモデル化するから。
* リーク無し <- なぜならリソース管理が簡単だから。

### これで全てではありません

Rxを使って可能な限りあなたのアプリケーションをモデル化することは通常は良いアイデアです。

しかしあなたはまだ全ての演算子を知りませんし、あなたの特定のケースをモデル化する演算子は存在するのでしょうか？

全てのRx演算子は数学に基づいていて直感的であるべきです。

いい知らせとして10〜15の演算子は最も一般的なユースケースをカバーしています。
そしてその中には `map`, `filter`, `zip`, `observeOn` などのおなじみのものがいくつか含まれています。

こちらに[全てのRx演算子](http://reactivex.io/documentation/operators.html)の巨大なリストと、[現在RxSwiftでサポートされている演算子](API.md)のリストがあります。

各演算子についてそれがどのように動作するのか理解するのに役立つ[マーブルダイアグラム](http://reactivex.io/documentation/operators/retry.html)が示されています。

しかしもしあなたが必要としている演算子がリストに含まれていないとしたらどうしましょう？
その時はあなた自身で演算子を作ることができます。

もし何らかの理由で演算子のようなものを作るのが難しいとしたら、
または何らかのレガシーでステートフルなコード断片を持っていてそれを使って仕事をする必要があるとしたら？
まぁ、あなた自身を混乱させるでしょう。しかしあなたは簡単に[Rxモナドから飛び降りて](GettingStarted.md#life-happens)データの処理に戻ることができます。

Tips
====

* 常にあなたのシステムまたはそれらのパーツを純粋関数としてモデル化する努力をしてください。  
  それらの純粋関数はテストが簡単でオペレーターの振る舞いを変更して使用することができます。
* Rxを使うとき、最初はビルトインのオペレーターで組み立てようとしてください。
* しばしばいくつかのオペレーターを組み合わせて使っている場合、  
  あなたの便利なオペレーター(convenience operators)を作ってください。

例.

```swift
extension ObservableType where E: MaybeCool {

    @warn_unused_result(message="http://git.io/rxs.uo")
    public func coolElements()
        -> Observable<E> {
          return filter { e -> Bool in
              return e.isCool
          }
    }
}
```

  * Rxオペレーターはできるだけ一般的なものです、しかし常にモデル化が難しいエッジケースはあります。これらの場合はおそらくいずれかのビルトインオペレーターを参照してあなたのオペレーターを作ることができます。

  * 常にsubscriptionを構成するオペレーターを使用してください。

  **なんとしてもネストしたsubscribe呼び出しは避けてください。これは悪い臭いがします。**

  ```swift
  textField.rx_text.subscribeNext { text in
      performURLRequest(text).subscribeNext { result in
          ...
      }
      .addDisposableTo(disposeBag)
  }
  .addDisposableTo(disposeBag)
  ```

  **好ましい方法はオペレーターを使用してdisposableをチェインすることです。**

  ```swift
  textField.rx_text
      .flatMapLatest { text in
          // Assuming this doesn't fail and returns result on main scheduler,
          // otherwise `catchError` and `observeOn(MainScheduler.instance)` can be used to
          // correct this.
          return performURLRequest(text)
      }
      ...
      .addDisposableTo(disposeBag) // only one top most disposable
  ```

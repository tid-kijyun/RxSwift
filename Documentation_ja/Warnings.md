Warnings
========

### <a name="unused-disposable"></a>未使用のdisposable (unused-disposable)

`Disposable` を返す `subscribe*`, `bind*` それと `drive*` 関数ファミリーは次のことが有効です。

警告はおそらくこれと同じ文脈で表示されます:

```Swift
let xs: Observable<E> ....

xs
  .filter { ... }
  .map { ... }
  .switchLatest()
  .subscribe(onNext: {
    ...
  }, onError: {
    ...
  })  
```

`subscribe`関数はサブスクリプションした`Disposable`を返し、それは計算のキャンセルとリソース解放に使用できます。

よどみなく呼び出しを終了させる望ましい方法は`DisposeBag`を使用することです。
いずれかのコールチェインを使って`.addDisposableTo(disposeBag)`を呼び出すか、bagに直接disposableを追加します。

```Swift
let xs: Observable<E> ....
let disposeBag = DisposeBag()

xs
  .filter { ... }
  .map { ... }
  .switchLatest()
  .subscribe(onNext: {
    ...
  }, onError: {
    ...
  })
  .addDisposableTo(disposeBag) // <--- `addDisposableTo` に注目
```

`disposeBag`の割り当てが解除されるとき、それに含まれているdisposableが自動的にdisposeされます。

`xs`が`Completed`または`Error`どちらかの予測可能な方法で終了する場合、  
ハンドリングしていないサブスクリプションした`Disposable`はすべてのリソースをリークしません。  
しかしながら、このような場合であっても、  
依然としてdispose bagを使うことはサブスクリプションしたdisposableをハンドルする望ましい方法です。  
要素計算が常に予測可能な時点で終了することを保証し、堅牢で将来性のあるコードを作ります。  
なぜならたとえリソースが適切にdisposeされていても`xs`の実装は変わるかもしれないからです。

サブスクリプションとリソースを確認する他の方法は  
いくつかのオブジェクトの存続期間と結びついている`takeUntil`オペレーターを使用することです。

```Swift
let xs: Observable<E> ....
let someObject: NSObject  ...

_ = xs
  .filter { ... }
  .map { ... }
  .switchLatest()
  .takeUntil(someObject.rx_deallocated) // <-- `takeUntil` オペレーターに注目
  .subscribe(onNext: {
    ...
  }, onError: {
    ...
  })
```

サブスクリプションした`Disposable`を無視しているのは予期した振る舞いです、これはコンパイラの警告を黙らせる方法です。

```Swift
let xs: Observable<E> ....

_ = xs // <-- アンダースコアに注目
  .filter { ... }
  .map { ... }
  .switchLatest()
  .subscribe(onNext: {
    ...
  }, onError: {
    ...
  })
```

### <a name="unused-observable"></a>未使用のobservableシーケンス(unused-observable)

警告はおそらくこれと同じ文脈で表示されます:

```Swift
let xs: Observable<E> ....

xs
  .filter { ... }
  .map { ... }
```

このコードは`xs`シーケンスからフィルターしてマップするobservableシーケンスを定義していますが、その結果は無視します。

このコードはobservableシーケンスを定義してそれを無視しているだけですが、実際には何もしません。

あなたの意図はおそらくobservableシーケンスの定義を保存するか、後でそれを使用することです...

```Swift
let xs: Observable<E> ....

let ys = xs // <--- 名前を `ys` と定義する
  .filter { ... }
  .map { ... }
```

... または定義に基づいて計算を開始する

```Swift
let xs: Observable<E> ....
let disposeBag = DisposeBag()

xs
  .filter { ... }
  .map { ... }
  .subscribeNext { nextElement in       // <-- `subscribe*` メソッドに注目
    // 要素を使う
    print(nextElement)
  }
  .addDisposableTo(disposeBag)
```

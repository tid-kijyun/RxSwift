## ビルド / インストール / 実行

Rxはいかなる外部依存も含んでいません。

現在サポートしているオプションです:

### 手動

Rx.xcworkspace を開いて、`RxExample` を選択して実行します。  
この方法は全てをビルドしてサンプルアプリを実行します。


### [CocoaPods](https://guides.cocoapods.org/using/using-cocoapods.html)

**:warning: 重要! tvOS をサポートするには CocoaPods `0.39` が必要です。 :warning:**


```
# Podfile
use_frameworks!

target 'YOUR_TARGET_NAME' do
    pod 'RxSwift',    '~> 2.0'
    pod 'RxCocoa',    '~> 2.0'
    pod 'RxBlocking', '~> 2.0'
    pod 'RxTests',    '~> 2.0'
end
```

`YOUR_TARGET_NAME` を置き換えてから `Podfile` のディレクトリで下記をタイプしてください。

```
$ pod install
```

### [Carthage](https://github.com/Carthage/Carthage)

**Xcode 7.1 が必要です**

これを `Cartfile` に追加します。

```
github "ReactiveX/RxSwift" ~> 2.0
```

```
$ carthage update
```

### git submodule を使用した手動

* サブモジュールとして RxSwift を追加

```
$ git submodule add git@github.com:ReactiveX/RxSwift.git
```

* プロジェクトナビゲーターに `Rx.xcodeproj` をドラッグ
* `Project > Targets > Build Phases > Link Binary With Libraries` に移動、  
  `+` をクリックして`RxSwift-[Platform]` と `RxCocoa-[Platform]` ターゲットを選択

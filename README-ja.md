<img src="assets/Rx_Logo_M.png" alt="Miss Electric Eel 2016" width="36" height="36"> RxSwift: ReactiveX for Swift
======================================

[![Travis CI](https://travis-ci.org/ReactiveX/RxSwift.svg?branch=master)](https://travis-ci.org/ReactiveX/RxSwift) ![platforms](https://img.shields.io/badge/platforms-iOS%20%7C%20OSX%20%7C%20tvOS%20%7C%20watchOS%20%7C%20Linux%28experimental%29-333333.svg) ![pod](https://img.shields.io/cocoapods/v/RxSwift.svg) [![Carthage compatible](https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat)](https://github.com/Carthage/Carthage)


Xcode 7 Swift 2.1 required

## Rxについて

Rxは[計算の一般的な抽象化](https://youtu.be/looJcaeboBY)を`Observable<Element>`インターフェースで表現します。

これは[Rx](https://github.com/Reactive-Extensions/Rx.NET)のSwiftバージョンです。

可能な限りオリジナルから多くの概念を取り入れようとしますが、いくつかの概念はより快適なパフォーマンスのためにiOS/OSX環境に適合させました。

クロスプラットフォームのドキュメントは[ReactiveX.io](http://reactivex.io/)で見つけることができます。

オリジナルのRxと同様に、非同期操作とイベント/データストリームを簡単に構成することを意図しています。

KVO、非同期操作そしてストリームは抽象化されたシーケンスの下に一元化されます。
これがRxがとてもシンプルでエレガント、そしてパワフルである理由です。

## 私がここにきたのは ...

###### ... 知りたいから

* [なぜRxを使うのか](Documentation_ja/Why.md)
* [RxSwiftの入門と基礎](Documentation_ja/GettingStarted.md)
* [単位](Documentation_ja/Units.md) - `Driver`, `ControlProperty`, `Variable`とは何か、そしてなぜ存在しているのか
* [テスティング](Documentation_ja/UnitTests.md)
* [コツと共通エラー](Documentation_ja/Tips.md)
* [デバッキング](Documentation_ja/GettingStarted.md#debugging)
* [Rxの裏にある数学](Documentation_ja/MathBehindRx.md)
* [hotとcold observable sequenceとは何か](Documentation_ja/HotAndColdObservables.md)
* [どのような公開APIがあるのか](Documentation_ja/API.md)

###### ... インストールしたいから

* RxSwift/RxCocoa をアプリに統合したい. [インストールガイド](Documentation_ja/Installation.md)

###### ... ハックしたいから

* アプリの実例. [実例アプリを実行する](Documentation_ja/ExampleApp.md)
* playgroundで演算子を試す. [Playgrounds](Documentation_ja/Playgrounds.md)

###### ... 交流したいから

* RxSwiftと経験を用いて仲間と交流すると良いでしょう、最高ですよ。<br />[![Slack channel](http://slack.rxswift.org/badge.svg)](http://slack.rxswift.org) [Join Slack Channel](http://slack.rxswift.org/)
* ライブラリを使用して見つけたバグを報告してください。 [バグ報告テンプレートを使ってIssueを作る](Documentation/IssueTemplate.md)
* 新しい機能をリクエストしてください。 [機能リクエストテンプレートを使ってIssueを作る](Documentation/NewFeatureRequestTemplate.md)


###### ... 比較したいから

* [他のライブラリ](Documentation/ComparisonWithOtherLibraries.md)


###### ... 互換性を見つけたいから

* [RxSwiftコミュニティ](https://github.com/RxSwiftCommunity)のライブラリ
* [RxSwiftを使っているPods](https://cocoapods.org/?q=uses%3Arxswift)

###### ... より広範なビジョンを見たいから

* Android向けはありますか? => [RxJava](https://github.com/ReactiveX/RxJava)
* Where is all of this going, what is the future, what about reactive architectures, how do you design entire apps this way? [Cycle.js](https://github.com/cyclejs/cycle-core) - this is javascript, but [RxJS](https://github.com/Reactive-Extensions/RxJS) is javascript version of Rx.
* 全てはどこに向かっていますか？未来は？Reactiveアーキテクチャは？どのようにこの方法でアプリ全体をデザインしますか？ => [Cycle.js](https://github.com/cyclejs/cycle-core) - これはJavascriptですが[RxJS](https://github.com/Reactive-Extensions/RxJS)はRxのJavascriptバージョンです。

##### リファレンス

* [http://reactivex.io/](http://reactivex.io/)
* [Reactive Extensions GitHub (GitHub)](https://github.com/Reactive-Extensions)
* [Erik Meijer (Wikipedia)](http://en.wikipedia.org/wiki/Erik_Meijer_%28computer_scientist%29)
* [Expert to Expert: Brian Beckman and Erik Meijer - Inside the .NET Reactive Framework (Rx) (video)](https://youtu.be/looJcaeboBY)
* [Reactive Programming Overview (Jafar Husain from Netflix)](https://www.youtube.com/watch?v=dwP1TNXE6fc)
* [Subject/Observer is Dual to Iterator (paper)](http://csl.stanford.edu/~christos/pldi2010.fit/meijer.duality.pdf)
* [Rx standard sequence operators visualized (visualization tool)](http://rxmarbles.com/)
* [Haskell](https://www.haskell.org/)

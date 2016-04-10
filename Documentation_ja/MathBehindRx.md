Rxの裏にある数学
==============

## Observer と Iterator / Enumerator / Generator / Sequences の間の双対性

observerとgenerator patternの間には双対性があります。  
それは非同期コールバックの世界からシーケンス変換の同期の世界に移行することを可能にするものです。

要するに、列挙子とオブザーバーパターンの両方がシーケンスを記述します。  
列挙子はシーケンスを定義していますが、observerがもう少し複雑である理由はかなり明白です、

しかしながら数学の多くの知識を必要としない非常に単純な例はあります。  
あなたが与えられた時間周期で画面上のマウスカーソルの位置を監視していると仮定します。  
時間が経つにつれて、これらのマウスの位置がシーケンスを形成します。これは本質的にはobserverシーケンスです。

シーケンスの要素にアクセス可能な2つの基本的な方法があります:

* プッシュインターフェース - Observer (監視された要素が時間をかけてシーケンスを作ります)
* プルインターフェース - Iterator / Enumerator / Generator

このビデオの中でよりフォーマルな説明を見ることができます:

* [Expert to Expert: Brian Beckman and Erik Meijer - Inside the .NET Reactive Framework (Rx) (video)](https://www.youtube.com/watch?v=looJcaeboBY)
* [Reactive Programming Overview (Jafar Husain from Netflix)](https://www.youtube.com/watch?v=dwP1TNXE6fc)

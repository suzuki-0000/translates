## ReactiveCocoa

ReactiveCocoa(以下RAC)は[FRP](http://en.wikipedia.org/wiki/Functional_reactive_programming)に影響をうけたCocoaフレームワークです。RACは時系列上のストリームの値を変換するためのAPIを提供します。

 1. [はじめに](#はじめに)
 1. [オンラインサーチで考えてみる](#オンラインサーチで考えてみる)
 1. [Objective-CとSwift](#objective-cとswift)
 1. [Rxとどういう関係があるの？](#rxとどういう関係があるの)
 1. [さあはじめよう](#さあはじめよう)

もしあなたがすでにFRPに精通しているか、RACがどんなものか知っているならば、より詳細な情報のための[ドキュメント](https://github.com/ReactiveCocoa/ReactiveCocoa/tree/master/Documentation)をチェックしてください。そして、個々のAPIをもっとよく知るために[コード内のコメント](https://github.com/ReactiveCocoa/ReactiveCocoa/tree/master/ReactiveCocoa)を見てください。

なにか質問があったら、まずGithubIssueかStackOverflowにすでに回答がないか探してみてください。
なかったら、ためらわずにissueを開いてください！

### 互換性
このドキュメントはRAC4(現在alpha版）について書かれており、swift2.x用です。
swift1.2 の場合は[RAC3](https://github.com/ReactiveCocoa/ReactiveCocoa/tree/v3.0.0)を参照してください。

ReactiveCocoa 3の開発を支援してくれたRheinfabrikに感謝！

## はじめに
RACは[FRP](http://blog.maybeapps.com/post/42894317939/input-and-output)に影響をうけています。
RACはSignal,SignalProducerといったイベントストリームを提供することで、時系列上の値を提供できます。ミュータブルな変数を使わずとも正しく値を変更できます。

イベントストリームはCocoaの非同期処理パターンを単一化することができます。
 * Delegate methods
 * Callback blocks
 * NSNotifications
 * Control actions and responder chain events
 * [Key-value observing(KVO)](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html)
 * [Futures and promises](https://en.wikipedia.org/wiki/Futures_and_promises)
 
これらの異なる実装パターンをすべて同じインターフェースで表現することができるので、スパゲッティコードになることを防ぎ、宣言的にそれらを組み合わせることが容易になります。

もっとRACについて知りたい人は[こちら](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/Documentation/FrameworkOverview.md).

## オンラインサーチで考えてみる

ではテキストフィールドがタイプされるたびに、通信リクエストを実施したい場合を考えて見ましょう。

#### テキストの変更を監視する

まず最初にRACのUITextFieldのextensionを使います

```swift
let searchStrings = textField.rac_textSignal().toSignalProducer()
    |> map { text in text as! String }
```

これは、Stringを送信するSignalProducerを返します。

#### 通信リクエスト作成

変更のあったそれぞれのStringで、通信リクエストを作成します。
幸運なことに、RACではNSURLSessionのextensionでそれができます。

```swift
let searchResults = searchStrings
    |> flatMap(.Latest) { query in
        let URLRequest = self.searchRequestWithEscapedQuery(query)
        return NSURLSession.sharedSession().rac_dataWithRequest(URLRequest)
    }
    |> map { data, URLResponse in
        let string = String(data: data, encoding: NSUTF8StringEncoding)!
        return parseJSONResultsFromString(string)
    }
    |> observeOn(UIScheduler())
```

これでSignalProducerがもつStringが、検索結果をもったArrayへと変換されます。
検索結果はmain threadで受け取ることができます。（UISchedulerのおかげ）

付け加えると、`flatMap(.Latest)`は最新のStringのみを使用するために使用されます。
もしユーザーが通信リクエスト中に他の文字をタイプした場合、前の文字はキャンセルされ、新しい文字で通信リクエストを実施します。
これをもし自分でやろうと考えてみてくださいよ！

#### 結果を受け取る
これはまだ実際には実行されていません。
なぜなら、Producerは必ず結果を受け取るために`start`せねばなりません。 (そうすることで実際は使われなかった場合も、このコードが実行されるのを防ぎます)。
実装は簡単でこんな感じ:

```swift
searchResults.start(next: { results in
    println("Search results: \(results)")
})
```

ここに`Next`というものがありますね、これが結果を含んでおり、ログへと出力しています。これはたとえばtableViewをupdateしたりlabelをUIに出力する時に便利です。

#### エラーハンドリング

このサンプルではすべての通信エラーは`Error`として生成され、イベントストリームは切断されます。不幸なことに、一度切断されると再トライは実施されません。

このために、エラー時に何をするか宣言する必要があります。
最も簡単な解決方法は、ログを出力して無視するだけです。

```swift
    |> flatMap(.Latest) { query in
        let URLRequest = self.searchRequestWithEscapedQuery(query)

        return NSURLSession.sharedSession().rac_dataWithRequest(URLRequest)
            |> catch { error in
                println("Network error occurred: \(error)")
                return SignalProducer.empty
            }
    }
```

空なイベントストリームを返却することで効率的にエラーを無視できます。
しかし、本来は簡単に諦めるのではなく、何回かリトライするなどの適切な方法があるべきでしょう。
便利なことに、そこには`retry`があります！
ではProducerをパワーアップさせましょう。
こんな感じ：

```swift
let searchResults = searchStrings
    |> flatMap(.Latest) { query in
        let URLRequest = self.searchRequestWithEscapedQuery(query)

        return NSURLSession.sharedSession().rac_dataWithRequest(URLRequest)
            |> retry(2)
            |> catch { error in
                println("Network error occurred: \(error)")
                return SignalProducer.empty
            }
    }
    |> map { data, URLResponse in
        let string = String(data: data, encoding: NSUTF8StringEncoding)!
        return parseJSONResultsFromString(string)
    }
    |> observeOn(UIScheduler())
```

#### Throttlingリクエスト

さて、あなたは今度は最小限の通信トラフィックで、検索結果を得たいと思うでしょう。
RACには`throttle`オペレータがあり、これを検索クエリーに適用することができます。
こんな感じ：

```swift
let searchStrings = textField.rac_textSignal().toSignalProducer()
    |> map { text in text as! String }
    |> throttle(0.5, onScheduler: QueueScheduler.mainQueueScheduler)
```

これで0.5秒間の間、値がsendされるのを防ぐことができます。
これを自分でやらなければいけない時、かなり多くの状態制御コードを書かねばならず、結果、可読性の下がったコードになるでしょう! 
RACではたった一つのオペレータでイベントストリームへ取り込むことができます。

## Objective-CとSwift
RACはObjective-CのFrameworkしてスタートしましたが、バージョン3.0に関しては、すべての主要な機能の開発はSwiftに集中しています。

RACのObjective-CのSwiftのAPIは完全に分離されていますが、両者の間で変換するBridgeがあります。これは主に古いRACプロジェクトのための後方互換としての意味、もしくはまだSwiftで追加されていないココアの拡張機能を使用するためです。

Objective-CのAPIに関して、予測可能な将来に関してはサポートしていきますが、それは多くの改良ではないでしょう。このAPIを使用する方法の詳細については、[レガシードキュメント](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/Documentation/Legacy/README.md)を参照してください。

**我々はとにかく新しいプロジェクトに関しては、Swift版を使うことをおすすめします。**

## Rxとどういう関係があるの？
RACはもともとMicrosoftがつくった[ReactiveExtensions(Rx)](https://msdn.microsoft.com/en-us/data/gg577609.aspx) に強い影響をうけています。なので、Rxと同じふるまいをたくさん持っています（それは[RxSwift](https://github.com/kzaher/RxSwift)のことも含みます）。しかし、RACは意図的に違うふるまいを持っています。

なにがRACはRxと違うのかというと、

* シンプルなAPI
* 同一ソースによる混乱に対処
* Cocoaによせて記述してある

以下、その根拠とともに、具体的な相違点のいくつかを記載します。

### ネーミング
ほとんどのRxは、イベントストリームを`Observable`と称します。これは.NETでは
`Enumerable`に類似します。加えて、ほとんどのRx.NETの操作名は[LINQ](https://msdn.microsoft.com/en-us/library/bb397926.aspx)から来ています。これらはDBのselectやwhereなどを想起させます。
**RACはSwiftのネーミングに何よりも合わせています。**
それは`map`であり、`filter`です。
他のネーミングの違いに関しては[Haskell](https://www.haskell.org)や、[Elm](http://elm-lang.org) などから来ています。(Signalなどの専門用語はここから）

### SignalsとSignalProducersについて(“hot” and “cold” observables)

最もRxとRACを混乱させている一面はRxの[“hot”, “cold”, and “warm” observables](http://www.introtorx.com/content/v1.0.10621.0/14_HotAndColdObservables.html) (event streams)です。

その関数などはC#でこんな感じ:

```csharp
IObservable<string> Search(string query)
```

これは`IObservable`がどんな副作用を起こすか判断することが不可能です。
もし副作用がある場合、それが最初の一つだけあったとしても、それぞれのサブスクリプションで検知することは不可能です。
この例はわざとらしいですが、しかしこのデモンストレーションがいかにRx（そしてpre-3.0のRAC)の理解を困難にしているか示しています。

RAC3.0ではSignalとSignalProducerに分けることでこの問題を解決しました。
これは新しく学ばなければならないことが増えたことを意味しているけれども、コードは明晰を増し、エンジニア同士の意思疎通をより良くするでしょう。

すなわち、**RACはシンプルになりましたが、簡単ではないです。**

### タイプエラー
SignalsとSignalProducersはerrorを可能にしましたが、
いくつかのエラーは必ずシステムエラーを指定しなければいけませんでした。
たとえば`Signal<Int, NSError>`はIntを伝達するSignalですが、エラーは`NSError`を伝達します。

さらに重要なこととして、RACは`NoError`という特殊なタイプを使用でき、これはイベントストリームにエラーを送信することは許されないことを保証します。
**これは予期しないエラーイベントによって引き起こされる多くのバグを排除します。**

Rxではイベントストリームは自分たちが指定したValueのみ指定でき、エラーは指定できないため、RACのできるこれらの事項は不可能です。

### UIプログラミング

Rxは基本的にはどのように使用されるかはかなり隠蔽化されています。
Rxの持つUIプログラミングは非常に一般的ですが、それその特定のケースに合うように調整された機能はほとんど持っていません。

RACは多くのインスピレーションを[ReactiveUI](http://reactiveui.net)より取り入れています。`Actions`などがそうです。

ReactiveUIとは異なり、残念ながらRxはUIプログラミングのためのそれをより使いやすくするために直接変更することはできません。RACは**かなり多くの時間をこのUI操作のためにかけています。**たとえそれがRxとは遠くかけ離れていくとしても。


## さあはじめよう
※割愛

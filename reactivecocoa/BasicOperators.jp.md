# 基本演算子

ここでは、ReactiveCocoaにおいて最も一般的なオペレータのうちのいくつを使い、それらの使用を示している例を説明します。

ここで説明するオペレータはSignalおよびSignalProducerを変換する機能です。Swiftのカスタムオペレータではありません。
すなわち、これらはReactiveCocoaがイベントストリームを提供するために内包されたオペレータです。

ここではSignal,SignalProduerともに、同じ「イベントストリーム」として扱います。
もし区別が必要な場合はタイプは名前によって区別されます。

**[イベントストリームの効用について](#イベントストリームの効用について)**

  1. [Observation](#observation)
  1. [Injecting effects](#injecting-effects)

**[オペレータの構造](#operator-composition)**

  1. [Lifting](#lifting)
  1. [Pipe](#pipe)

**[イベントストリームの変換](#transforming-event-streams)**

  1. [Mapping](#mapping)
  1. [Filtering](#filtering)
  1. [Aggregating](#aggregating)

**[イベントストリームの結合](#combining-event-streams)**

  1. [Combining latest values](#combining-latest-values)
  1. [Zipping](#zipping)

**[Producerの平坦化](#flattening-producers)**

  1. [Concatenating](#concatenating)
  1. [Merging](#merging)
  1. [Switching to the latest](#switching-to-the-latest)

**[Handling errors](#handling-errors)**

  1. [Catching errors](#catch)
  1. [Mapping errors](#mapping-error)
  1. [Retrying](#retrying)

## イベントストリームの効用について

### 監視

`Signal`は`Observe` functionを用いて監視することができます。
Signalはどのようなイベントが送信された場合でも`Observer`を引数として受け取ることができます。

```Swift
signal.observe(Signal.Observer { event in
    switch event {
    case let .Next(next):
        println("Next: \(next)")
    case let .Error(error):
        println("Error: \(error)")
    case .Completed:
        println("Completed")
    case .Interrupted:
        println("Interrupted")
    }
})
```

あるいは、 ` Next`、` ERROR`、Completed`、` Interrupted`イベントが、イベントが応答したときに受け取ることもできます。

```Swift
signal.observe(next: { next in
    println("Next: \(next)")
}, error: { error in
    println("Error: \(error)")
}, completed: {
    println("Completed")
}, interrupted: {
    println("Interrupted")
})
```

すべてはオプショナルであるため、４つのパラメータをすべてを提供する必要はないです。
好きなモノを使えば良いのです。

`observe`はオペレータとしても使用可能で[|>](#pipe)としても表されます。

### Injecting effects

副作用は、`SignalProducer`として注入できます。オペレータは`on`を使います。

```Swift
let producer = signalProducer
    |> on(started: {
        println("Started")
    }, event: { event in
        println("Event: \(event)")
    }, error: { error in
        println("Error: \(error)")
    }, completed: {
        println("Completed")
    }, interrupted: {
        println("Interrupted")
    }, terminated: {
        println("Terminated")
    }, disposed: {
        println("Disposed")
    }, next: { next in
        println("Next: \(next)")
    })
```

`observe`と似ていて、全てはオプショナルです。好きなモノを使ってください。
Producerをスタートさせないと、なにもプリントされないので気をつけてくださいね。

## Operator composition

### Pipe

`|>` オペレータはイベントストリームを適用する際に利用します。複数のオペレータは連鎖できます。
こんな感じ：

```Swift
intSignal
    |> filter { num in num % 2 == 0 }
    |> map(toString)
    |> observe(next: { string in println(string) })
```

### Lifting

`Signal`は`SignalProducer`に作用するように`lift`を用い働きかけます。

これを与えられたそれぞれのSignalの演算子を元にSignalProducerをあらたに作成します。 あたかもそれぞれのSignalに個別に働きかけたかのように。

`|>`演算子は、暗黙のうちにSignal演算子をlistするので、SignalProducerに直接働きかけることができます。

## Transforming event streams

これらのオペレータは、新しいストリームにイベントストリームを変換します。These 

### Mapping

`map`はイベントストリーム内の値の変換に使われ、新しいストリームを変換結果とともに返します。

```Swift
let (signal, sink) = Signal<String, NoError>.pipe()

signal
    |> map { string in string.uppercaseString }
    |> observe(next: println)

sendNext(sink, "a")     // Prints A
sendNext(sink, "b")     // Prints B
sendNext(sink, "c")     // Prints C
```

[mapのよくわかる図式](http://neilpa.me/rac-marbles/#map)

### Filtering

`filter`は宣言されたFunctionを満たしたイベントストリーム内の値のみを変換します。

```Swift
let (signal, sink) = Signal<Int, NoError>.pipe()

signal
    |> filter { number in number % 2 == 0 }
    |> observe(next: println)

sendNext(sink, 1)     // Not printed
sendNext(sink, 2)     // Prints 2
sendNext(sink, 3)     // Not printed
sendNext(sink, 4)     // prints 4
```

[filterのよくわかる図式](http://neilpa.me/rac-marbles/#filter)

### Aggregating（集計）

`reduce`はイベントストリーム内の値を集計し、１つの値とします。
変換した、最後の値のみが送信されることに注意してください

```Swift
let (signal, sink) = Signal<Int, NoError>.pipe()

signal
    |> reduce(1) { $0 * $1 }
    |> observe(next: println)

sendNext(sink, 1)     // nothing printed
sendNext(sink, 2)     // nothing printed
sendNext(sink, 3)     // nothing printed
sendCompleted(sink)   // prints 6
```

`collect`はイベントストリーム内の値を集計し、配列として変換します。
変換した、最後の値のみが送信されることに注意してください


```Swift
let (signal, sink) = Signal<Int, NoError>.pipe()

let collected = signal |> collect

collected.observe(next: println)

sendNext(sink, 1)     // nothing printed
sendNext(sink, 2)     // nothing printed
sendNext(sink, 3)     // nothing printed
sendCompleted(sink)   // prints [1, 2, 3]
```

[reduceのよくわかる図式](http://neilpa.me/rac-marbles/#reduce)

## イベントストリームの結合

以下は、に複数のイベントストリームを結合し、１つのストリームにします。

### 最新の値の結合

`combineLatest`は、2つ（またはそれ以上）のイベントストリームの最新の値を組み合わせます。

各入力で送信された後、得られた最新のストリームのみが値を送信します。
その後、なにか入力があるたびにそれが新しい値がになります。

```Swift
let (numbersSignal, numbersSink) = Signal<Int, NoError>.pipe()
let (lettersSignal, lettersSink) = Signal<String, NoError>.pipe()

combineLatest(numbersSignal, lettersSignal)
    |> observe(next: println, completed: { println("Completed") })

sendNext(numbersSink, 0)    // nothing printed
sendNext(numbersSink, 1)    // nothing printed
sendNext(lettersSink, "A")  // prints (1, A)
sendNext(numbersSink, 2)    // prints (2, A)
sendCompleted(numbersSink)  // nothing printed
sendNext(lettersSink, "B")  // prints (2, B)
sendNext(lettersSink, "C")  // prints (2, C)
sendCompleted(lettersSink)  // prints "Completed"
```

`combineLatestWith`はオペレータであることを除いて同じように動作します。

[`combineLatest`よくわかる図式](http://neilpa.me/rac-marbles/#combineLatest)

### Zipping

`zip`は、2つ（またはそれ以上）の値を結合します。
それぞれN番目のタプルの要素は、入力ストリームのN番目の要素に対応します。

すなわち、出力ストリームのN番目の値がそれぞれ入力されるまで送信できないことを意味します。

```Swift
let (numbersSignal, numbersSink) = Signal<Int, NoError>.pipe()
let (lettersSignal, lettersSink) = Signal<String, NoError>.pipe()

zip(numbersSignal, lettersSignal)
    |> observe(next: println, completed: { println("Completed") })

sendNext(numbersSink, 0)    // nothing printed
sendNext(numbersSink, 1)    // nothing printed
sendNext(lettersSink, "A")  // prints (0, A)
sendNext(numbersSink, 2)    // nothing printed
sendCompleted(numbersSink)  // nothing printed
sendNext(lettersSink, "B")  // prints (1, B)
sendNext(lettersSink, "C")  // prints (2, C) & "Completed"

```

`zipWith`はオペレータであることを除いて同じように動作します。
[`zip`よくわかる図式](http://neilpa.me/rac-marbles/#zip)

## Flattening producers

`flatten`は、SignalProducerとSignalProducerを一つのSignalProducerへと変換します。
その値は'FlattenSterategy'にしたがって提供されます。

なぜそれぞれのsterategyが存在するかは、以下のサンプルを参考にすることで、時間などの列オフセットを想像してみてください。

```Swift
let values = [
// imagine column offset as time
[ 1,    2,      3 ],
   [ 4,      5,     6 ],
         [ 7,     8 ],
]

let merge =
[ 1, 4, 2, 7,5, 3,8,6 ]

let concat = 
[ 1,    2,      3,4,      5,     6,7,     8]

let latest =
[ 1, 4,    7,     8 ]
```

どのように値がインターリーブしているか、また、どのよう値が結果の配列に含まれているか、注意してみてください。

### Merging

`merge`はすぐにすべての値をouter`SignalProducer`へと変換します。
いずれかのproducerで何かしらのエラーが送信された場合、すぐproducerへと変換され切断されます。

strategy immediately forwards every value of the inner `SignalProducer`s to the outer `SignalProducer`. Any error sent on the outer producer or any inner producer is immediately sent on the flattened producer and terminates it.

```Swift
let (producerA, lettersSink) = SignalProducer<String, NoError>.buffer(5)
let (producerB, numbersSink) = SignalProducer<String, NoError>.buffer(5)
let (signal, sink) = SignalProducer<SignalProducer<String, NoError>, NoError>.buffer(5)

signal |> flatten(FlattenStrategy.Merge) |> start(next: println)

sendNext(sink, producerA)
sendNext(sink, producerB)
sendCompleted(sink)

sendNext(lettersSink, "a")    // prints "a"
sendNext(numbersSink, "1")    // prints "1"
sendNext(lettersSink, "b")    // prints "b"
sendNext(numbersSink, "2")    // prints "2"
sendNext(lettersSink, "c")    // prints "c"
sendNext(numbersSink, "3")    // prints "3"
```

[`merge`のよくわかる図式](http://neilpa.me/rac-marbles/#merge)

### Concatenating

`Concat`は内部Producerの直列化をサポートします。外部Producerはすぐにスタートします。
続いておこるproducerはそれぞれのproducerがcompleteするまでスタートしません。
エラーはすぐにproducerへと送られます

The `.Concat` strategy is used to serialize work of the inner `SignalProducer`s. The outer producer is started immediately. Each subsequent producer is not started until the preceeding one has completed. Errors are immediately forwarded to the flattened producer.

```Swift
let (producerA, lettersSink) = SignalProducer<String, NoError>.buffer(5)
let (producerB, numbersSink) = SignalProducer<String, NoError>.buffer(5)
let (signal, sink) = SignalProducer<SignalProducer<String, NoError>, NoError>.buffer(5)

signal |> flatten(FlattenStrategy.Concat) |> start(next: println)

sendNext(sink, producerA)
sendNext(sink, producerB)
sendCompleted(sink)

sendNext(numbersSink, "1")    // nothing printed
sendNext(lettersSink, "a")    // prints "a"
sendNext(lettersSink, "b")    // prints "b"
sendNext(numbersSink, "2")    // nothing printed
sendNext(lettersSink, "c")    // prints "c"
sendCompleted(lettersSink)    // prints "1", "2"
sendNext(numbersSink, "3")    // prints "3"
sendCompleted(numbersSink)
```

[`Concat`のよくわかる図式](http://neilpa.me/rac-marbles/#concat)

### Switching to the latest

`Latest`は最新のinputのみを送信するためのものです。

The `.Latest` strategy forwards only values from the latest input `SignalProducer`.

```Swift
let (producerA, sinkA) = SignalProducer<String, NoError>.buffer(5)
let (producerB, sinkB) = SignalProducer<String, NoError>.buffer(5)
let (producerC, sinkC) = SignalProducer<String, NoError>.buffer(5)
let (signal, sink) = SignalProducer<SignalProducer<String, NoError>, NoError>.buffer(5)

signal |> flatten(FlattenStrategy.Latest) |> start(next: println)

sendNext(sink, producerA)   // nothing printed
sendNext(sinkC, "X")        // nothing printed
sendNext(sinkA, "a")        // prints "a"
sendNext(sinkB, "1")        // nothing printed
sendNext(sink, producerB)   // prints "1"
sendNext(sinkA, "b")        // nothing printed
sendNext(sinkB, "2")        // prints "2"
sendNext(sinkC, "Y")        // nothing printed
sendNext(sinkA, "c")        // nothing printed
sendNext(sink, producerC)   // prints "X", "Y"
sendNext(sinkB, "3")        // nothing printed
sendNext(sinkC, "Z")        // prints "Z"
```

## Handling errors

これから説明する演算子はエラーがイベントストリーム内で発生したさいに使用するものです。

These operators are used to handle errors that might occur on an event stream.

### Catching errors

`catch`はなにかのエラーをSignalProducerで受け取るためのものです。
catchの後に、新しいSignalProducerがスタートします。


The `catch` operator catches any error that may occur on the input `SignalProducer`, then starts a new `SignalProducer` in its place.

```Swift
let (producer, sink) = SignalProducer<String, NSError>.buffer(5)
let error = NSError(domain: "domain", code: 0, userInfo: nil)

producer
    |> catch { error in SignalProducer<String, NSError>(value: "Default") }
    |> start(next: println)


sendNext(sink, "First")     // prints "First"
sendNext(sink, "Second")    // prints "Second"
sendError(sink, error)      // prints "Default"
```

### Retrying

`retry`はエラーが発生した際にオリジナルのSignalProducerを必要な数だけ繰り返します。

The `retry` operator will restart the original `SignalProducer` on error up to `count` times.

```Swift
var tries = 0
let limit = 2
let error = NSError(domain: "domain", code: 0, userInfo: nil)
let producer = SignalProducer<String, NSError> { (sink, _) in
    if tries++ < limit {
        sendError(sink, error)
    } else {
        sendNext(sink, "Success")
        sendCompleted(sink)
    }
}

producer
    |> on(error: {e in println("Error")})             // prints "Error" twice
    |> retry(2)
    |> start(next: println,                           // prints "Success"
            error: { _ in println("Signal Error")})
```

もし`SignalProducer`が必要なretry分で成功しなかった場合、failします。
例えば上記の場合、`"Signal Error"`が`"Success"`の代わりに出力されます。

If the `SignalProducer` does not succeed after `count` tries, the resulting `SignalProducer` will fail. E.g., if  `retry(1)` is used in the example above instead of `retry(2)`, `"Signal Error"` will be printed instead of `"Success"`.

### Mapping errors

`mapError`はストリーム内で発生した何かしらのエラーを新しいエラーへと変換します。

The `mapError` operator transforms any error in an event stream into a new error. 

```Swift
enum CustomError: String, ErrorType {
    case Foo = "Foo"
    case Bar = "Bar"
    case Other = "Other"
    
    var nsError: NSError {
        return NSError(domain: "CustomError.\(rawValue)", code: 0, userInfo: nil)
    }
    
    var description: String {
        return "\(rawValue) Error"
    }
}

let (signal, sink) = Signal<String, NSError>.pipe()

signal
    |> mapError { (error: NSError) -> CustomError in
        switch error.domain {
        case "com.example.foo":
            return .Foo
        case "com.example.bar":
            return .Bar
        default:
            return .Other
        }
    }
    |> observe(error: println)

sendError(sink, NSError(domain: "com.example.foo", code: 42, userInfo: nil))    // prints "Foo Error"
```

### Promote

`promoteErrors`はイベントストリームを複数のエラーを一つのエラーへと昇格させます。

The `promoteErrors` operator promotes an event stream that does not generate errors into one that can. 

```Swift
let (numbersSignal, numbersSink) = Signal<Int, NoError>.pipe()
let (lettersSignal, lettersSink) = Signal<String, NSError>.pipe()

numbersSignal
    |> promoteErrors(NSError)
    |> combineLatestWith(lettersSignal)
```

これらのストリームは実際にはエラーを生成せず、
The given stream will still not _actually_ generate errors, but this is useful
because some operators to [combine streams](#combining-event-streams) require
the inputs to have matching error types.



# 基本演算子

ここでは、ReactiveCocoaにおいて最も一般的なオペレータのうちのいくつを使い、それらの使用を示している例を説明します。

ここで説明するオペレータはSignalおよびSignalProducerを変換する機能です。Swiftのカスタムオペレータではありません。
すなわち、これらはReactiveCocoaがイベントストリームを提供するために内包されたオペレータです。

ここではSignal,SignalProduerともに、同じ「イベントストリーム」として扱います。
もし区別が必要な場合はタイプは名前によって区別されます。

**[イベントストリームの効用について](#イベントストリームの効用について)**

  1. [Observation](#observation)
  1. [Injecting effects](#injecting-effects)

**[オペレータの構造](#オペレータの構造)**

  1. [Lifting](#lifting)

**[イベントストリームの変換](#イベントストリームの変換)**

  1. [Mapping](#mapping)
  1. [Filtering](#filtering)
  1. [Aggregating](#aggregating)

**[イベントストリームの結合](#イベントストリームの結合)**

  1. [Combining latest values](#combining-latest-values)
  1. [Zipping](#zipping)

**[Producerの平坦化](#Producerの平坦化)**

  1. [Concatenating](#concatenating)
  1. [Merging](#merging)
  1. [Switching to the latest](#switching-to-the-latest)

**[Handling errors](#handling-errors)**

  1. [Catching errors](#catch)
  1. [Mapping errors](#mapping-error)
  1. [Retrying](#retrying)

## イベントストリームの効用について

### Observation

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
signal.observeNext { next in 
  print("Next: \(next)") 
}
signal.observeFailed { error in
  print("Failed: \(error)")
}
signal.observeCompleted { 
  print("Completed") 
}
signal.observeInterrupted { 
  print("Interrupted")
}
```

すべてはオプショナルであるため、４つのパラメータをすべてを提供する必要はないです。
好きなモノを使えば良いのです。

### Injecting effects

`SignalProducer`として注入できます。オペレータは`on`を使うことでイベントストリームの生成を行います。

```Swift
let producer = signalProducer
    .on(started: {
        print("Started")
    }, event: { event in
        print("Event: \(event)")
    }, failed: { error in
        print("Failed: \(error)")
    }, completed: {
        print("Completed")
    }, interrupted: {
        print("Interrupted")
    }, terminated: {
        print("Terminated")
    }, disposed: {
        print("Disposed")
    }, next: { value in
        print("Next: \(value)")
    })
```

`observe`と似ていて、全てはオプショナルです。好きなモノを使ってください。
Producerをスタートさせないと、なにもプリントされないので気をつけてくださいね。

## オペレータの構造

### Lifting

`Signal`演算子は`lift`メソッドを使うことで`SignalProducer`上に働きかけることができます。
`lift`を与えられたそれぞれのSignalは新たにSignalProducerを生成します。 


## イベントストリームの変換

これらのオペレータは、新しいストリームにイベントストリームを変換します。 

### Mapping

`map`はイベントストリーム内の値の変換に使われ、新しいストリームを変換結果とともに返します。

```Swift
let (signal, observer) = Signal<String, NoError>.pipe()

signal
    .map { string in string.uppercaseString }
    .observeNext { next in print(next) }

observer.sendNext("a")     // Prints A
observer.sendNext("b")     // Prints B
observer.sendNext("c")     // Prints C
```

[mapのよくわかる図式](http://neilpa.me/rac-marbles/#map)

### Filtering

`filter`は宣言されたFunctionを満たしたイベントストリーム内の値のみを変換します。

```Swift
let (signal, observer) = Signal<Int, NoError>.pipe()

signal
    .filter { number in number % 2 == 0 }
    .observeNext { next in print(next) }

observer.sendNext(1)     // Not printed
observer.sendNext(2)     // Prints 2
observer.sendNext(3)     // Not printed
observer.sendNext(4)     // prints 4
```

[filterのよくわかる図式](http://neilpa.me/rac-marbles/#filter)

### Aggregating

`reduce`はイベントストリーム内の値を集計し、１つの値とします。
変換した、最後の値のみが送信されることに注意してください

```Swift
let (signal, observer) = Signal<Int, NoError>.pipe()

signal
    .reduce(1) { $0 * $1 }
    .observeNext { next in print(next) }

observer.sendNext(1)     // nothing printed
observer.sendNext(2)     // nothing printed
observer.sendNext(3)     // nothing printed
observer.sendCompleted()   // prints 6
```


`collect`はイベントストリーム内の値を集計し、配列として変換します。
変換した、最後の値のみが送信されることに注意してください

```Swift
let (signal, observer) = Signal<Int, NoError>.pipe()

signal
    .collect()
    .observeNext { next in print(next) }

observer.sendNext(1)     // nothing printed
observer.sendNext(2)     // nothing printed
observer.sendNext(3)     // nothing printed
observer.sendCompleted()   // prints [1, 2, 3]
```

[reduceのよくわかる図式](http://neilpa.me/rac-marbles/#reduce)

## イベントストリームの結合

複数のイベントストリームを結合し、１つのストリームにします。

### Combining latest values

`combineLatest`は、2つ（またはそれ以上）のイベントストリームの最新の値を組み合わせます。

各入力で送信された後、得られた最新のストリームのみが値を送信します。
その後、なにか入力があるたびにそれが新しい値がになります。

```Swift
let (numbersSignal, numbersObserver) = Signal<Int, NoError>.pipe()
let (lettersSignal, lettersObserver) = Signal<String, NoError>.pipe()

let signal = combineLatest(numbersSignal, lettersSignal)
signal.observeNext { next in print("Next: \(next)") }
signal.observeCompleted { print("Completed") }

numbersObserver.sendNext(0)      // nothing printed
numbersObserver.sendNext(1)      // nothing printed
lettersObserver.sendNext("A")    // prints (1, A)
numbersObserver.sendNext(2)      // prints (2, A)
numbersObserver.sendCompleted()  // nothing printed
lettersObserver.sendNext("B")    // prints (2, B)
lettersObserver.sendNext("C")    // prints (2, C)
lettersObserver.sendCompleted()  // prints "Completed"
```

`combineLatestWith`はオペレータであることを除いて同じように動作します。

[`combineLatest`よくわかる図式](http://neilpa.me/rac-marbles/#combineLatest)

### Zipping

`zip`は、2つ（またはそれ以上）の値を結合します。
それぞれN番目のタプルの要素は、入力ストリームのN番目の要素に対応します。

すなわち、出力ストリームのN番目の値がそれぞれ入力されるまで送信できないことを意味します。

```Swift
let (numbersSignal, numbersObserver) = Signal<Int, NoError>.pipe()
let (lettersSignal, lettersObserver) = Signal<String, NoError>.pipe()

let signal = zip(numbersSignal, lettersSignal)
signal.observeNext { next in print("Next: \(next)") }
signal.observeCompleted { print("Completed") }

numbersObserver.sendNext(0)      // nothing printed
numbersObserver.sendNext(1)      // nothing printed
lettersObserver.sendNext("A")    // prints (0, A)
numbersObserver.sendNext(2)      // nothing printed
numbersObserver.sendCompleted()  // nothing printed
lettersObserver.sendNext("B")    // prints (1, B)
lettersObserver.sendNext("C")    // prints (2, C) & "Completed"

```

`zipWith`はオペレータであることを除いて同じように動作します。
[`zip`よくわかる図式](http://neilpa.me/rac-marbles/#zip)


## Producerの平坦化

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

どのように値がインターリーブしているか、また、どのような値が結果の配列に含まれているか、注意してみてください。

### Merging

`merge`はinner`SignalProducer`の値をすぐにouter`SignalProducer`へと送信します。
いずれかのproducerで何かしらのエラーが送信された場合、すぐにflattenされたSignalProducerへと変換され、切断されます。

```Swift
let (producerA, lettersObserver) = SignalProducer<String, NoError>.buffer(5)
let (producerB, numbersObserver) = SignalProducer<String, NoError>.buffer(5)
let (signal, observer) = SignalProducer<SignalProducer<String, NoError>, NoError>.buffer(5)

signal.flatten(.Merge).startWithNext { next in print(next) }

observer.sendNext(producerA)
observer.sendNext(producerB)
observer.sendCompleted()

lettersObserver.sendNext("a")    // prints "a"
numbersObserver.sendNext("1")    // prints "1"
lettersObserver.sendNext("b")    // prints "b"
numbersObserver.sendNext("2")    // prints "2"
lettersObserver.sendNext("c")    // prints "c"
numbersObserver.sendNext("3")    // prints "3"
```


[`merge`のよくわかる図式](http://neilpa.me/rac-marbles/#merge)

### Concatenating

`Concat`はinner`SignalProducer`の直列化をサポートします。outer`SignalProducer`はすぐにスタートします。
それぞれのproducerは、現在処理されているproducerがcompleteするまでスタートしません。
エラーはすぐにflattenされたproducerへと送信されます。


```Swift
let (producerA, lettersObserver) = SignalProducer<String, NoError>.buffer(5)
let (producerB, numbersObserver) = SignalProducer<String, NoError>.buffer(5)
let (signal, observer) = SignalProducer<SignalProducer<String, NoError>, NoError>.buffer(5)

signal.flatten(.Concat).startWithNext { next in print(next) }

observer.sendNext(producerA)
observer.sendNext(producerB)
observer.sendCompleted()

numbersObserver.sendNext("1")    // nothing printed
lettersObserver.sendNext("a")    // prints "a"
lettersObserver.sendNext("b")    // prints "b"
numbersObserver.sendNext("2")    // nothing printed
lettersObserver.sendNext("c")    // prints "c"
lettersObserver.sendCompleted()    // prints "1", "2"
numbersObserver.sendNext("3")    // prints "3"
numbersObserver.sendCompleted()
```

[`Concat`のよくわかる図式](http://neilpa.me/rac-marbles/#concat)

### Switching to the latest

`Latest`は最新のinputのみを送信するためのものです。

```Swift
let (producerA, observerA) = SignalProducer<String, NoError>.buffer(5)
let (producerB, observerB) = SignalProducer<String, NoError>.buffer(5)
let (producerC, observerC) = SignalProducer<String, NoError>.buffer(5)
let (signal, observer) = SignalProducer<SignalProducer<String, NoError>, NoError>.buffer(5)

signal.flatten(.Latest).startWithNext { next in print(next) }

observer.sendNext(producerA)   // nothing printed
observerC.sendNext("X")        // nothing printed
observerA.sendNext("a")        // prints "a"
observerB.sendNext("1")        // nothing printed
observer.sendNext(producerB)   // prints "1"
observerA.sendNext("b")        // nothing printed
observerB.sendNext("2")        // prints "2"
observerC.sendNext("Y")        // nothing printed
observerA.sendNext("c")        // nothing printed
observer.sendNext(producerC)   // prints "X", "Y"
observerB.sendNext("3")        // nothing printed
observerC.sendNext("Z")        // prints "Z"
```

## Handling errors

これから説明する演算子はエラーがイベントストリーム内で発生したさいに使用するものです。

### Catching errors

`flatMapError`は発生したエラーを`SignalProducer`で受け取るためのものです。
catchの後に、新しいSignalProducerがスタートします。


```Swift
let (producer, observer) = SignalProducer<String, NSError>.buffer(5)
let error = NSError(domain: "domain", code: 0, userInfo: nil)

producer
    .flatMapError { _ in SignalProducer<String, NoError>(value: "Default") }
    .startWithNext { next in print(next) }


observer.sendNext("First")     // prints "First"
observer.sendNext("Second")    // prints "Second"
observer.sendFailed(error)     // prints "Default"
```

### Retrying

`retry`はエラーが発生した際にオリジナルのSignalProducerを必要な数だけ繰り返します。

```Swift
var tries = 0
let limit = 2
let error = NSError(domain: "domain", code: 0, userInfo: nil)
let producer = SignalProducer<String, NSError> { (observer, _) in
    if tries++ < limit {
        observer.sendFailed(error)
    } else {
        observer.sendNext("Success")
        observer.sendCompleted()
    }
}

producer
    .on(failed: {e in print("Failure")})    // prints "Failure" twice
    .retry(2)
    .start { event in
        switch event {
        case let .Next(next):
            print(next)                     // prints "Success"
        case let .Failed(error):
            print("Failed: \(error)")
        case .Completed:
            print("Completed")
        case .Interrupted:
            print("Interrupted")
        }
    }
```


もし`SignalProducer`が必要なretry分で成功しなかった場合、failします。
例えば上記の場合、`retry(1)`を使った場合、`"Success"`の代わりに`"Signal Failure"`が出力されます。

### Mapping errors

`mapError`はストリーム内で発生した何かしらのエラーを新しいエラーへと変換します。

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

let (signal, observer) = Signal<String, NSError>.pipe()

signal
    .mapError { (error: NSError) -> CustomError in
        switch error.domain {
        case "com.example.foo":
            return .Foo
        case "com.example.bar":
            return .Bar
        default:
            return .Other
        }
    }
    .observeFailed { error in
        print(error)
    }

observer.sendFailed(NSError(domain: "com.example.foo", code: 42, userInfo: nil))    // prints "Foo Error"
```

### Promote

`promoteErrors`はイベントストリームがエラーを生成できないものを一つのエラーへと昇格させます。

```Swift
let (numbersSignal, numbersObserver) = Signal<Int, NoError>.pipe()
let (lettersSignal, lettersObserver) = Signal<String, NSError>.pipe()

numbersSignal
    .promoteErrors(NSError)
    .combineLatestWith(lettersSignal)
```

これらのストリームは実際にはエラーを生成しませんが、これはいくつかのイベントストリームの結合演算子（なにかのエラーを必要とする）において有用です。  


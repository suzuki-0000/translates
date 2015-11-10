# Framework Overview

ここでは、ReactiveCocoaフレームワークの中の構成要素について、さらに詳しく見ていき、それぞれどのように動作し、責任を担っているか説明します。
これは新しいモジュールについて学習し、より具体的な記述を見ていくということです。

RACを使うための理解の手助けとして、そしてサンプルとして、ReadmeかDisignGuideLineを参照すると良いでしょう。

## Events

`Event`は、「なにかが起きた」ということを形式的に表現しています。
RACでは、Eventはコミュニケーションのための中心的役割です。
Eventは、ボタンを押した時、いくつかの情報をAPIから受け取った時、エラーが発生した時、長時間操作の完了を受け取った時、など、さまざまなことを表現します。
いずれのケースにせよ、なにかがEventを生成し、Signalと通していくつかのObserverへと送信します。

`Event`は、値、または3つのいずれかのイベントを表すEnumです。

 * `Next`イベントは、新しい値をソースから提供します。
 * `Error`イベントはシグナルが終わる前にエラーが発生したことを示します。イベントは`ErrorType`とパラメータ化でき、こちらが決定したエラーの形でイベントの中に出現させられます。エラーが許されない場合、`NoError`を使うことで、どのようなエラーが出現することを防ぎます。
 * `Completed`イベントはすべてのシグナルが正しく終了したことを示します。値はソースからこれ以上送られることはありません。
 * `Interrupted`イベントはシグナルがキャンセルされたことで切断された、ということを示します。つまり操作は、成功もしていないし、失敗もしていません。

## Signals

`Signal`は時系列上での監視できる一連のイベントです。

Signalは通常「実行中」の、イベントストリームとして表されます。例えば、notificationや、ユーザーインプットのように実行され、もしくは受け取るものです。
シグナル上でイベントは送信され、それはありとあらゆるオブザーバへに届きます。すべてのオブザーバはイベントを同じタイミングで受け取ります。

ユーザーは必ずイベントにアクセスするために、Signalを監視しなければなりません。
Signalを監視することはなにも引き起こしません。
つまり、Signalは完全にproduer-driven、push駆動型なので、消費者（監視側）はそのライフサイクルの中で影響を持つことができません。Signalを監視している間、ユーザは同じ順序でイベントを評価することができます。そこにランダムなアクセスはありません。

Signalは基本演算子を用いることで、操作することができます。
１つのSignalを操作する典型的なものは`filter`、`map`、`reduce`などです。
同様に複数のSignalを操作するものとして`zip`があります。
これらのSignalは`Next`でのみ適用可能です。
一般的に、`|>`演算子でこれをつなげ、より複雑なものを構成することができます。

Signalの寿命は、イベント（組み合わせではない）を終了する１つの切断イベント、それは例えば何かしらの`Error`であり、`Completed`、`Interrupted`から来ています。
イベントの切断にはシグナルの値は含まれていません。特別に処理する必要があります。

### Pipes

`pipe`は`Signal.pipe()`コマンドから作成され、シグナルをコントロールできます。

pipeは`Signal`と`Observer`を返却します。
シグナルはイベントをオブザーバに送信することでコントロールできます。
これは、ReactiveCocoa的でないコードにSignalの有益性を広める極めて有益な機会です。

例えばblockのcallbackによるアプリの操作のかわりに、ブロックはただシンプルにイベントをオブザーバに送るだけです。その間にシグナルを受け取ることができるので、細かなコールバックの実装を隠蔽することができます。


## Signal Producers

`SignalProducer`は`Signal`を作成し、実際に作用を与えます。

SignalProducerは、通信リクエストのように、操作を表現するために使用することができ、 `start()`により実施の操作を作成し、呼び出し元が結果（複数可）を監視することができます。
`startWithSignal（）`は作成されたSignalへのアクセスを提供します。必要に応じて複数回呼ばれます。

`start()`のおかげで、,同じSignalから作成された個々のProducerは、順序の違いや、イベントのバージョンや、もしくはストリームすら完全に違うことすらあるでしょう！
そのへんの単純なシグナルと異なり、オブザーバーがアタッチされるまではなにも開始しません（イベントも生成されません）。そして操作はそれぞれの追加されたオブザーバーのために再スタートされます。

SignalProducerがスタートすると、`disposable`を返却します。これは、`interrupt`,`cancel`で、SignalProducerが作成したシグナルとその効果を中断、キャンセルするために使用できます。

Signalと同様に、SignalProducerは`map`,`filter`などで値を操作できます。
すべての操作はSignalProducerの`lift`を使うことで、または`|>`演算子を使うことで、 操作できます。さらに、追加の操作演算子もあります。たとえば`times`などです。 

### Buffers

`buffer`は`SignalProducer.buffer()`で作成され、Producerによって新たに作成されたシグナルをリプレイするためのキューです。
pipeに似ていて、Observerを返却します。EventがこのObserverに送信した値はキューされます。
もしbufferがすでに容量一杯であれば、もっとも古いデータから削除されます。

## Observers

`Observer`はシグナルからイベントを待っている、もしくは待っている可能性のあるものです。
RACではObserverは[`SinkType`](http://swiftdoc.org/protocol/SinkType)として表現され、イベントを処理します。

Observerはコールバックベースの`Signal.observe`か`SignalProducer.start`関数によって、暗黙的に作成できます。


## Actions

An **action**, represented by the [`Action`][Action] type, will do some work when
executed with an input. While executing, zero or more output values and/or an
error may be generated.

Actions are useful for performing side-effecting work upon user interaction, like when a button is
clicked. Actions can also be automatically disabled based on a [property](#properties), and this
disabled state can be represented in a UI by disabling any controls associated
with the action.

For interaction with `NSControl` or `UIControl`, RAC provides the
[`CocoaAction`][CocoaAction] type for bridging actions to Objective-C.

## Properties

A **property**, represented by the [`PropertyType`][Property] protocol,
stores a value and notifies observers about future changes to that value.

The current value of a property can be obtained from the `value` getter. The
`producer` getter returns a [signal producer](#signal-producers) that will send
the property’s current value, followed by all changes over time.

The `<~` operator can be used to bind properties in different ways. Note that in
all cases, the target has to be a [`MutablePropertyType`][Property].

* `property <~ signal` binds a [signal](#signals) to the property, updating the
  property’s value to the latest value sent by the signal.
* `property <~ producer` starts the given [signal producer](#signal-producers),
  and binds the property’s value to the latest value sent on the started signal.
* `property <~ otherProperty` binds one property to another, so that the destination
  property’s value is updated whenever the source property is updated.

The [`DynamicProperty`][Property] type can be used to bridge to Objective-C APIs
that require Key-Value Coding (KVC) or Key-Value Observing (KVO), like
`NSOperation`. Note that most AppKit and UIKit properties do _not_ support KVO,
so their changes should be observed through other mechanisms.
[`MutableProperty`][Property] should be preferred over dynamic properties
whenever possible!

## Disposables

A **disposable**, represented by the [`Disposable`][Disposable] protocol, is a mechanism
for memory management and cancellation.

When starting a [signal producer](#signal-producers), a disposable will be returned.
This disposable can be used by the caller to cancel the work that has been started
(e.g. background processing, network requests, etc.), clean up all temporary
resources, then send a final `Interrupted` event upon the particular
[signal](#signals) that was created.

Observing a [signal](#signals) may also return a disposable. Disposing it will
prevent the observer from receiving any future events from that signal, but it
will not have any effect on the signal itself.

For more information about cancellation, see the RAC [Design Guidelines][].

## Schedulers

A **scheduler**, represented by the [`SchedulerType`][Scheduler] protocol, is a
serial execution queue to perform work or deliver results upon.

[Signals](#signals) and [signal producers](#signal-producers) can be ordered to
deliver events on a specific scheduler. [Signal producers](#signal-producers)
can additionally be ordered to start their work on a specific scheduler.

Schedulers are similar to Grand Central Dispatch queues, but schedulers support
cancellation (via [disposables](#disposables)), and always execute serially.
With the exception of the [`ImmediateScheduler`][Scheduler], schedulers do not
offer synchronous execution. This helps avoid deadlocks, and encourages the use
of [signal and signal producer primitives][BasicOperators] instead of blocking work.

Schedulers are also somewhat similar to `NSOperationQueue`, but schedulers
do not allow tasks to be reordered or depend on one another.


[Design Guidelines]: DesignGuidelines.md
[BasicOperators]: BasicOperators.md
[README]: ../README.md
[Signal]: ../ReactiveCocoa/Swift/Signal.swift
[SignalProducer]: ../ReactiveCocoa/Swift/SignalProducer.swift
[Action]: ../ReactiveCocoa/Swift/Action.swift
[CocoaAction]: ../ReactiveCocoa/Swift/Action.swift
[Disposable]: ../ReactiveCocoa/Swift/Disposable.swift
[Scheduler]: ../ReactiveCocoa/Swift/Scheduler.swift
[Property]: ../ReactiveCocoa/Swift/Property.swift
[Event]: ../ReactiveCocoa/Swift/Event.swift
[SinkOf]: http://swiftdoc.org/type/SinkOf/


# Design Guidelines

このドキュメントはReactiveCocoaを使う上でのガイドラインを含みます。
このドキュメントは[Rx Design
Guidelines](http://blogs.msdn.com/b/rxteam/archive/2010/10/28/rx-design-guidelines.aspx)に強く影響を受けています。

このドキュメントでは、RACの基本的な知識があることを前提とします。
`Framework Overview.md`が基本的なRACが提供するコンセプトを理解するのを助けるでしょう。

**[Eventについて](#eventについて)**

 1. Nextは値を提供したり、イベントの発生を示す
 1. Failuresは例外のように動作し、即座に送信する
 1. Completionは成功を示す
 1. Interruptは、処理をキャンセルし、通常はすぐ送信する
 1. EventはSerialである
 1. Eventは再帰的に送信することはできない]
 1. Eventはデフォルトでは同期的に送信される

**[Signalについて](#signalについて)**

 1. Signalは、startされて初めてインスタンスを生成
 1. SignalはObservingすることでの副作用はない
 1. SignalのすべてのObserverは同じイベントを同じ順序でこなす
 1. SignalはObserverがリリースされるまで保持される
 1. Eventを切断することでSignalのリソースを廃棄する

**[SignalProducerについて](#signalProducerについて)**

 1. SignalProducerはSignalの作成によってstartする
 1. produceされたSignalはそれぞれ異なるタイミングで異なるeventを送信しうる
 1. Signal演算子によりSignalProducerに適用する
 1. 破棄されたSignalは中断される

**[ベストプラクティス](#ベストプラクティス)**

 1. [Processは必要な数だけ]
 1. [scheduler上でEventをObserveする]
 1. [schedulerのスイッチは可能な限り少ない箇所で]
 1. [SignalProducer内で副作用を処理する]
 1. [SignalProduerの副作用は、一つのproduceされたSignalで行わせる]
 1. [ライフサイクルの管理は、明示的なdisposal演算子を用いる]

**[新しい演算子を実装する](#新しい演算子を実装する)**

 1. Signal/SignalProducerともに適用できる演算子が好ましい
 1. 可能であれば存在する演算子で構成する
 1. 可能な限り早くFailure/Interruptionを発生させる
 1. Eventの値でスイッチする
 1. 並列性は避ける
 1. 演算子内でブロッキング処理は避ける


## `Event`について

EventはReactiveCocoaにとって根幹をなすものといえます。
Signal/SignalProducerがEventを送信し、それは"イベントストリーム"と称することができます。

イベントストリームは以下の文法でなければいけません。

```
Next* (Interrupted | Failed | Completed)?
```

イベントストリームは以下の要素から構成されます。

 1. いくつかの`Next`イベント
 2. Optionalのいくつかの中断イベント、`Interrupted`、`Failed`、`Completed`

これらいくつかの中断イベントの後は、イベントは送信されません。

#### `Next`は値を提供したり、イベントの発生を示す

`Next`は"Value"と言われる値を持っています。
`Next`イベントだけが、値を持っているという言い方ができます。
ゆえに、イベントストリームはいくつかの`Next`を含み、
これらのValueはすべて同じタイプでなければいけません。
また、その使用方法にはいくつかの制限が存在します。

例えば、"Value"はコレクションからの要素であるかもしれないし、
いくつかの長時間実行操作に関するものかもしれません。

また、 `Next`イベントのValueは何も表現しないこともあるかもしれません。
それは`()`として扱うことができ、具体的になにかが発生した、というよりも、「なにかが起こった」ということを示すためにあります。

ほとんどのイベント上のストリームは`Next`上で動作し、Signal/SignalProduerによって、意味のあるデータとして表現されます。

#### Failuresは例外のように動作し、即座に送信する

`Failed`イベントはなにか悪いことが起こったということを示し、
なにが起こったかというエラーを返します。
これは致命的なので、可能な限り早く使用側に操作できるように伝達されます。

この失敗は例外として振るまいを持ち、"skip"演算子でエラーを送信できます。
逆にいうと、ほとんどの演算子はエラーが発生するとすぐに処理を中断しエラーを送信します。
これは時間操作系の演算子にすら適用され、`delay`などの名前にかかわらず、すぐに伝達されます。

結論で言うと、失敗は「普通でない」中断として扱われるべきです。
もし演算子に重要な処理を終わらせたければ、`Next`で行われるのが適切です。

もしイベントストリームが失敗させない場合、`NoError`としてパラメータ化してください。
これをすることで`Failed`イベントはイベントストリーム上で実行されません。

#### Completionは成功を示す

操作が成功した場合、もしくは正常に中断された場合、イベントストリームは`Completed`を送信します。
多くの演算子が`Completed`を操作し、そのイベントストリーム上の寿命を短縮したり延長したりします。
例えば、`take`はいくつかのvalueを受け取ったあとにcompleteとします。
言い方を変えると、signalかproducerが受け入れるほとんどの演算子は、`Completed`イベントを送信する前のイベントがすべて完了するまで待ちます。結果は通常、すべての入力に依存します。

eived, thereby terminating the stream early. On the other hand, most
operators that accept multiple signals or producers will wait until _all_ of
them have completed before forwarding a `Completed` event, since a successful
outcome will usually depend on all the inputs.

#### Interruption cancels outstanding work and usually propagates immediately
#### Interruptは、処理をキャンセルし、通常はすぐ送信する

`Interrupted`はイベントストリームがキャンセルされた時に送信されます。
中断はsuccessかfailureが成功しなかったが、それは最後まで終わらなかったからで、failではない場合です。
ほとんどの演算子は中断をすぐに伝達しますが、しかし例外もあります。
例えば、`flatten`は内部Producerで起こった何かしらのを、無視するので、内部処理は不必要に大きな単位の処理をキャンセルすべきではありません。

RACは`disposal`時に自動的に`Interrupted`を送信しますが、もし必要であれば手動で送ることができます。
加えて、カスタム演算子は必ず中断イベントを送信する必要があります。

An `Interrupted` event is sent when an event stream should cancel processing.
Interruption is somewhere between [success](#completion-indicates-success)
and [failure](#failures-behave-like-exceptions-and-propagate-immediately)—the
operation was not successful, because it did not get to finish, but it didn’t
necessarily “fail” either.

Most [operators][] will propagate interruption immediately, but there are some
exceptions. For example, the [flattening operators][flatten] will ignore
`Interrupted` events that occur on the _inner_ producers, since the cancellation
of an inner operation should not necessarily cancel the larger unit of work.

RAC will automatically send an `Interrupted` event upon [disposal][Disposables], but it can
also be sent manually if necessary. Additionally, [custom
operators](#implementing-new-operators) must make sure to forward interruption
events to the observer.

#### Events are serial
#### EventはSerialである

RACはイベントストリーム上のすべてのイベントがシリアルであることを保証します。
言い換えると、signalまたはproducerのobserverは複数のイベントを同時に受け取ることは不可能です。
それがもし複数のスレッドによって同時に送信されたとしてもです。


RAC guarantees that all events upon a stream will arrive serially. In other
words, it’s impossible for the observer of a signal or producer to receive
multiple `Event`s concurrently, even if the events are sent on multiple threads
simultaneously.

This simplifies [operator][Operators] implementations and [observers][].

#### Events cannot be sent recursively
#### Eventは再帰的に送信することはできない

RACが複数のイベントを同時に受け取ることを保証しないように、
イベントは再帰的に受け取ることもできません。

もしイベントがすでに処理中の以前のイベントからのイベントである場合、デッドロックが発生します。
再帰的なシグナルは基本的にプログラミングミスとして捉えます。
デットロックの発生は、処理過程の出力結果がイベントなどの順序やタイミングと予期しない（危険な）依存関係にある場合に発生すべきです。

再帰的な信号が望ましい場合、delay演算子などでタイムシフトされるべきで、
それはイベントがすでに実行中でないことを保証する必要があります。

Just like RAC guarantees that [events will not be received
concurrently](#events-are-serial), it also guarantees that they won’t be
received recursively. As a consequence, [operators][] and [observers][] _do not_ need to
be reentrant.

If an event is sent upon a signal from a thread that is _already processing_
a previous event from that signal, deadlock will result. This is because
recursive signals are usually programmer error, and the determinacy of
a deadlock is preferable to nondeterministic race conditions.

When a recursive signal is explicitly desired, the recursive event should be
time-shifted, with an operator like [`delay`][delay], to ensure that it isn’t sent from
an already-running event handler.

#### Events are sent synchronously by default
#### Eventはデフォルトでは同期的に送信される

RACは、暗黙的に同時実行または非同期実行を導入していません。
スケジューラ演算子により、明示的に呼び出さなければいけません。

通常Signal,Producerは、同期的にすべてのイベントを送信します。
デフォルトではObserverは同期的に、それぞれのイベントごとに（イベントが送信されるたびに）呼び出されます。
イベントストリームはイベントが終わるまで再開しません。

これは`UIControl`、` NSNotificationCenter`の処理に似ています。

RAC does not implicitly introduce concurrency or asynchrony. [Operators][] that
accept a [scheduler][Schedulers] may, but they must be explicitly invoked by the consumer of
the framework.

A “vanilla” signal or producer will send all of its events synchronously by
default, meaning that the [observer][Observers] will be synchronously invoked for each event
as it is sent, and that the underlying work will not resume until the event
handler finishes.

This is similar to how `NSNotificationCenter` or `UIControl` events are
distributed.

## The `Signal` contract
## `Signal`について

シグナルは常にイベントに従います。

シグナルは参照型で、それぞれのシグナルはそれぞれ独自性がある、つまり、
シグナル自体がそのライフタイムを有し、そして最終的に終了することができます。
終了したら、Signalは再起動することはできません。


A [signal][Signals] is an “always on” stream that obeys [the `Event`
contract](#the-event-contract).

`Signal` is a reference type, because each signal has identity—in other words, each
signal has its own lifetime, and may eventually terminate. Once terminated,
a signal cannot be restarted.

#### Signals start work when instantiated
#### Signalは、startされて初めてインスタンスを生成

シグナルの初期化により、すぐに生成されたクロージャーを実行します。
これは初期化時点で何からの副作用が起こる可能性があることを意味します。

また、初期化するより前にイベントを送信することもできます。
しかし、Observerがこの時点でアタッチできなくなってからは、送信されるイベントを受信することはできません。


[`Signal.init`][Signal.init] immediately executes the generator closure that is passed to it.
This means that side effects may occur even before the initializer returns.

It is also possible to send [events][] before the initializer returns. However,
since it is impossible for any [observers][] to be attached at this point, any
events sent this way cannot be received.

#### Observing a signal does not have side effects
#### SignalはObservingすることでの副作用はない

処理を紐付けられたシグナルは、Observerが追加されても削除されても、開始、または停止することはありません。
なのでObserveメソッドは副作用を持ちえません。

シグナルの副作用の停止は、シグナルの中断を通してのみ可能です。

The work associated with a `Signal` does not start or stop when [observers][] are
added or removed, so the [`observe`][observe] method (or the cancellation thereof) never
has side effects.

A signal’s side effects can only be stopped through [a terminating
event](#signals-are-retained-until-a-terminating-event-occurs).

#### All observers of a signal see the same events in the same order
#### SignalのすべてのObserverは同じイベントを同じ順序でこなす

Observerは副作用を持ち得ないので、Signalはイベントをカスタマイズすることはできません。
シグナルからイベントが送信されるとき、`NSNotificationCenter`と同じように、同期的に全てのObserverに配信されます。

つまり、Observerごとに異なるイベントストリーム（タイムライン）でないということです。
すべてのObserverは効率的に同じイベントストリームを参照します。

このルールの例外が１つあります。
Observerに、すでに中断されたシグナルを追加する場合、必ず１つの`Interrupted`イベントが
指定のObserverに送られます。

Because [observation does not have side
effects](#observing-a-signal-does-not-have-side-effects), a `Signal` never
customizes events for different [observers][]. When an event is sent upon a signal,
it will be [synchronously](#events-are-sent-synchronously-by-default)
distributed to all observers that are attached at that time, much like
how `NSNotificationCenter` sends notifications.

In other words, there are not different event “timelines” per observer. All
observers effectively see the same stream of events.

There is one exception to this rule: adding an observer to a signal _after_ it
has already terminated will result in exactly one
[`Interrupted`](#interruption-cancels-outstanding-work-and-usually-propagates-immediately)
event sent to that specific observer.

#### A signal is retained until the underlying observer is released
#### SignalはObserverがリリースされるまで保持される

もし呼び出し側がSignalへの参照を保持していない場合でも

 - Signal.initで生成されたシグナルは、生成されたクロージャーがObserverの引数を解放するまで保持し続けます。
 - Signal.pipeで生成されたシグナルは、返却されたObserverがリリースされるまで保持し続けます。

これは長時間作業することを割り当てられたSignalが、途中で解放されないことを保証するためです。

注意してほしいのは、SignalはTerminatingイベントが送信される前に解放可能であるということです。
これは通常メモリーリークを引き起こすため避けられるべきですが、時には中断を無効化する有効な手段にもなります。

Even if the caller does not maintain a reference to the `Signal`:

 - A signal created with [`Signal.init`][Signal.init] is kept alive until the generator closure
   releases the [observer][Observers] argument.
 - A signal created with [`Signal.pipe`][Signal.pipe] is kept alive until the returned observer
   is released.

This ensures that signals associated with long-running work do not deallocate
prematurely.

Note that it is possible to release a signal before a terminating [event][Events] has been
sent upon it. This should usually be avoided, as it can result in resource
leaks, but is sometimes useful to disable termination.

#### Terminating events dispose of signal resources
#### Eventを切断することでSignalのリソースを廃棄する

TerminatingイベントがSignalを通して送られた時、すべてのObserverはリリースされ、
処分されるべきイベント郡を処理するために生成されています。

適切なクリーンアップを確実にする最も簡単な方法は、Terminationが発生した時に処理されるDisposableを生成されたClosureから取得することです。
Disposableはメモリのリリース、I/Oのハンドリング、ネットワーク通信キャンセルのハンドリング、その他、なにかしらの実行されている処理への責任を持っています。

When a terminating [event][Events] is sent along a `Signal`, all [observers][] will be
released, and any resources being used to generate events should be disposed of.

The easiest way to ensure proper resource cleanup is to return a [disposable][Disposables]
from the generator closure, which will be disposed of when termination occurs.
The disposable should be responsible for releasing memory, closing file handles,
canceling network requests, or anything else that may have been associated with
the work being performed.

## The `SignalProducer` contract
## SignalProducerについて

SignalProducerはいわばSignalを生成する「レシピ」のようなものです。
SignalProducerは彼らそのものは何もせず、SignalがProduceされたタイミングでのみ作用します。

なので、SignalProducerはどのようにSignalを生成するかを宣言するだけです。
それはValueTypeであり、メモリー管理の機能を持ちません。

A [signal producer][Signal Producers] is like a “recipe” for creating
[signals][]. Signal producers do not do anything by themselves—[work begins only
when a signal is produced](#signal-producers-start-work-on-demand-by-creating-signals).

Since a signal producer is just a declaration of _how_ to create signals, it is
a value type, and has no memory management to speak of.

#### Signal producers start work on demand by creating signals
#### SignalProducerはSignalの作成によってstartする

`start`, `startWithSignal`メソッドでSignalをProduceします（明示的に、そして暗黙的に、それぞれ）
Signalが生成された後、SignalProducerのinitによって渡されたclosureが実行されます。
それはeventsの流れであり、いずれかのObserverにアタッチされた後です。

Producer自体は、処理のための責任を担いませんが、
"starting"と"canceling"はProducerとの対話で一般的です。
これらの要素はSignalを実際に動作させるために使用し、またSignalを破棄するために使用します。

Producerはゼロを含む任意の数をスタートすることができます
また、それに関連する処理は正確に同じ数だけ実行されます。

The [`start`][start] and [`startWithSignal`][startWithSignal] methods each
produce a `Signal` (implicitly and explicitly, respectively). After
instantiating the signal, the closure that was passed to
[`SignalProducer.init`][SignalProducer.init] will be executed, to start the flow
of [events][] after any observers have been attached.

Although the producer itself is not really responsible for the execution of
work, it’s common to speak of “starting” and “canceling” a producer. These terms
refer to producing a `Signal` that will start work, and disposing of that
signal to stop work.

A producer can be started any number of times (including zero), and the work
associated with it will execute exactly that many times as well.

#### Each produced signal may send different events at different times
#### produceされたSignalはそれぞれ異なるタイミングで異なるeventを送信しうる

SignalProducerは生成されたSignalによってオンデマンドで処理されるため、
紐付けられたObserverによって実行する処理は異なるで可能性があり、
それらのObserverはそれぞれ完全に違ったEventsのタイムラインを見ることになります。

言いかえると、EventsはProducerがスタートするたびに再び生成され、
それは他の時系列からProducerがスタートすることで完全に異なるものでありうるということです

にもかかわらず、それぞれのSignalProducerはEventsの条件に従います。

Because signal producers [start work on
demand](#signal-producers-start-work-on-demand-by-creating-signals), there may
be different [observers][] associated with each execution, and those observers
may see completely different [event][Events] timelines.

In other words, events are generated from scratch for each time the producer is
started, and can be completely different (or in a completely different order)
from other times the producer is started.

Nonetheless, each execution of a signal producer will follow [the `Event`
contract](#the-event-contract).

#### Signal operators can be lifted to apply to signal producers
#### Signal演算子によりSignalProducerを操作しうる

SignalとSignalProducerの関係により、いかなる演算子（１つでも複数でも）も、
liftメソッドを用いることで自動的にSignalからSignalProducerへと変換することができます。

`lift`は適用されます、振るまいを　指定された演算子それぞれのSignalへと　それはSignalProducerがスタートした時に


Due to the relationship between signals and signal producers, it is possible to
automatically promote any [operators][] over one or more `Signal`s to apply to
the same number of `SignalProducer`s instead, using the [`lift`][lift] method.

`lift` will apply the behavior of the specified operator to each `Signal` that
is [created when the signal produced is started](#signal-producers-start-work-on-demand-by-creating-signals).

#### Disposing of a produced signal will interrupt it
#### 破棄されたSignalは中断される

SignalProducerが`start`,`startWithSignal`でスタートした時、`Disposable`が自動的に生成され、
返却されます。

DisposableオブジェクトはSignalを中断するために生成され、
現在実行中の処理を停止し、`Interruputed`イベントをすべてのObserverへ送信します。
また、同時にすべてのCompositeDisposableに含まれる処理を破棄します。

Signalの破棄によって、同じSignalProducerで生成された他のSignalに影響を与えないことに
注意してください。

When a producer is started using the [`start`][start] or
[`startWithSignal`][startWithSignal] methods, a [`Disposable`][Disposables] is
automatically created and passed back.

Disposing of this object will
[interrupt](#interruption-cancels-outstanding-work-and-usually-propagates-immediately)
the produced `Signal`, thereby canceling outstanding work and sending an
`Interrupted` [event][Events] to all [observers][], and will also dispose of
everything added to the [`CompositeDisposable`][CompositeDisposable] in
[SignalProducer.init].

Note that disposing of one produced `Signal` will not affect other signals created
by the same `SignalProducer`.

## Best practices
## ベスト・プラクティス

これから記述するプラクティスは、RACのコードを宣言的に、理解しやすく、同時にパフォーマンスに優れた状態を保つために記載されています。

しかし、これは１つのガイドラインでしかありません。
それぞれのどのような箇所に、どのようなコードを適用するかは、適切な判断をしてください。


The following recommendations are intended to help keep RAC-based code
predictable, understandable, and performant.

They are, however, only guidelines. Use best judgement when determining whether
to apply the recommendations here to a given piece of code.

#### Process only as many values as needed
#### Processは必要な数だけ

使われない処理に対して、
必要以上にイベントストリームを保持することはCPUとメモリーの無駄遣いです。

もしSignal、SignalProducerから取得する数が、正確に決まっている場合は
`take`,`takeUntil`などを使って、自動的に必要な条件を満たした場合にcompleteするようにするべきです。

これを実行することで潜在的にかなりの演算処理を省くことができます。

Keeping an event stream alive longer than necessary can waste CPU and memory, as
unnecessary work is performed for results that will never be used.

If only a certain number of values or certain number of time is required from
a [signal][Signals] or [producer][Signal Producers], operators like
[`take`][take] or [`takeUntil`][takeUntil] can be used to
automatically complete the stream once a certain condition is fulfilled.


The benefit is exponential, too, as this will terminate dependent operators
sooner, potentially saving a significant amount of work.

#### Observe events on a known scheduler
#### scheduler上でEventをObserveする



When receiving a [signal][Signals] or [producer][Signal Producers] from unknown
code, it can be difficult to know which thread [events][] will arrive upon. Although
events are [guaranteed to be serial](#events-are-serial), sometimes stronger
guarantees are needed, like when performing UI updates (which must occur on the
main thread).

Whenever such a guarantee is important, the [`observeOn`][observeOn]
[operator][Operators] should be used to force events to be received upon
a specific [scheduler][Schedulers].

#### Switch schedulers in as few places as possible
 1. [schedulerのスイッチは可能な限り少ない箇所で]


Notwithstanding the [above](#observe-events-on-a-known-scheduler), [events][]
should only be delivered to a specific [scheduler][Schedulers] when absolutely
necessary. Switching schedulers can introduce unnecessary delays and cause an
increase in CPU load.

Generally, [`observeOn`][observeOn] should only be used right before observing
the [signal][Signals], starting the [producer][Signal Producers], or binding to
a [property][Properties]. This ensures that events arrive on the expected
scheduler, without introducing multiple thread hops before their arrival.

#### Capture side effects within signal producers
 1. [SignalProducer内で副作用を処理する]

Because [signal producers start work on
demand](#signal-producers-start-work-on-demand-by-creating-signals), any
functions or methods that return a [signal producer][Signal Producers] should
make sure that side effects are captured _within_ the producer itself, instead
of being part of the function or method call.

For example, a function like this:

```swift
func search(text: String) -> SignalProducer<Result, NetworkError>
```

… should _not_ immediately start a search.

Instead, the returned producer should execute the search once for every time
that it is started. This also means that if the producer is never started,
a search will never have to be performed either.

#### Share the side effects of a signal producer by sharing one produced signal
 1. [SignalProduerの副作用は、一つのproduceされたSignalで行わせる]


If multiple [observers][] are interested in the results of a [signal
producer][Signal Producers], calling [`start`][start] once for each observer
means that the work associated with the producer will [execute that many
times](#signal-producers-start-work-on-demand-by-creating-signals) and [may not
generate the same results](#each-produced-signal-may-send-different-events-at-different-times).

If:

 1. the observers need to receive the exact same results
 1. the observers know about each other, or
 1. the code starting the producer knows about each observer

… it may be more appropriate to start the producer _just once_, and share the
results of that one [signal][Signals] to all observers, by attaching them within
the closure passed to the [`startWithSignal`][startWithSignal] method.

#### Prefer managing lifetime with operators over explicit disposal
 1. [ライフサイクルの管理は、明示的なdisposal演算子を用いる]


Although the [disposable][Disposables] returned from [`start`][start] makes
canceling a [signal producer][Signal Producers] really easy, explicit use of
disposables can quickly lead to a rat's nest of resource management and cleanup
code.

There are almost always higher-level [operators][] that can be used instead of manual
disposal:

 * [`take`][take] can be used to automatically terminate a stream once a certain
   number of values have been received.
 * [`takeUntil`][takeUntil] can be used to automatically terminate
   a [signal][Signals] or producer when an event occurs (for example, when
   a “Cancel” button is pressed in the UI).
 * [Properties][] and the `<~` operator can be used to “bind” the result of
   a signal or producer, until termination or until the property is deallocated.
   This can replace a manual observation that sets a value somewhere.

## Implementing new operators

RAC provides a long list of built-in [operators][] that should cover most use
cases; however, RAC is not a closed system. It's entirely valid to implement
additional operators for specialized uses, or for consideration in ReactiveCocoa
itself.

Implementing a new operator requires a careful attention to detail and a focus
on simplicity, to avoid introducing bugs into the calling code.

These guidelines cover some of the common pitfalls and help preserve the
expected API contracts. It may also help to look at the implementations of
existing `Signal` and `SignalProducer` operators for reference points.

#### Prefer writing operators that apply to both signals and producers

Since any [signal operator can apply to signal
producers](#signal-operators-can-be-lifted-to-apply-to-signal-producers),
writing custom operators in terms of [`Signal`][Signals] means that
[`SignalProducer`][Signal Producers] will get it “for free.”

Even if the caller only needs to apply the new operator to signal producers at
first, this generality can save time and effort in the future.

Of course, some capabilities _require_ producers (for example, any retrying or
repeating), so it may not always be possible to write a signal-based version
instead.

#### Compose existing operators when possible

Considerable thought has been put into the operators provided by RAC, and they
have been validated through automated tests and through their real world use in
other projects. An operator that has been written from scratch may not be as
robust, or might not handle a special case that the built-in operators are aware
of.

To minimize duplication and possible bugs, use the provided operators as much as
possible in a custom operator implementation. Generally, there should be very
little code written from scratch.

#### Forward failure and interruption events as soon as possible

Unless an operator is specifically built to handle
[failures](#failures-behave-like-exceptions-and-propagate-immediately) and
[interruption](#interruption-cancels-outstanding-work-and-usually-propagates-immedaitely)
in a custom way, it should propagate those events to the observer as soon as
possible, to ensure that their semantics are honored.

#### Switch over `Event` values

Instead of using [`start(failed:completed:interrupted:next:)`][start] or
[`observe(failed:completed:interrupted:next:)`][observe], create your own
[observer][Observers] to process raw [`Event`][Events] values, and use
a `switch` statement to determine the event type.

For example:

```swift
producer.start { event in
    switch event {
    case let .Next(value):
        println("Next event: \(value)")

    case let .Failed(error):
        println("Failed event: \(error)")

    case .Completed:
        println("Completed event")

    case .Interrupted:
        println("Interrupted event")
    }
}
```

Since the compiler will generate a warning if the `switch` is missing any case,
this prevents mistakes in a custom operator’s event handling.

#### Avoid introducing concurrency

Concurrency is an extremely common source of bugs in programming. To minimize
the potential for deadlocks and race conditions, operators should not
concurrently perform their work.

Callers always have the ability to [observe events on a specific
scheduler](#observe-events-on-a-known-scheduler), and RAC offers built-in ways
to parallelize work, so custom operators don’t need to be concerned with it.

#### Avoid blocking in operators

Signal or producer operators should return a new signal or producer
(respectively) as quickly as possible. Any work that the operator needs to
perform should be part of the event handling logic, _not_ part of the operator
invocation itself.

This guideline can be safely ignored when the purpose of an operator is to
synchronously retrieve one or more values from a stream, like `single()` or
`wait()`.

[CompositeDisposable]: ../ReactiveCocoa/Swift/Disposable.swift
[Disposables]: FrameworkOverview.md#disposables
[Events]: FrameworkOverview.md#events
[Framework Overview]: FrameworkOverview.md
[NoError]: ../ReactiveCocoa/Swift/Errors.swift
[Observers]: FrameworkOverview.md#observers
[Operators]: BasicOperators.md
[Properties]: FrameworkOverview.md#properties
[Schedulers]: FrameworkOverview.md#schedulers
[Signal Producers]: FrameworkOverview.md#signal-producers
[Signal.init]: ../ReactiveCocoa/Swift/Signal.swift
[Signal.pipe]: ../ReactiveCocoa/Swift/Signal.swift
[SignalProducer.init]: ../ReactiveCocoa/Swift/SignalProducer.swift
[Signals]: FrameworkOverview.md#signals
[delay]: ../ReactiveCocoa/Swift/Signal.swift
[flatten]: BasicOperators.md#flattening-producers
[lift]: ../ReactiveCocoa/Swift/SignalProducer.swift
[observe]: ../ReactiveCocoa/Swift/Signal.swift
[observeOn]: ../ReactiveCocoa/Swift/Signal.swift
[start]: ../ReactiveCocoa/Swift/SignalProducer.swift
[startWithSignal]: ../ReactiveCocoa/Swift/SignalProducer.swift
[take]: ../ReactiveCocoa/Swift/Signal.swift
[takeUntil]: ../ReactiveCocoa/Swift/Signal.swift

# Framework Overview

ここでは、ReactiveCocoaフレームワークの中の構成要素について、さらに詳しく見ていき、それぞれどのように動作し、責任を担っているか説明します。
これは新しいモジュールについて学習し、より具体的な記述を見ていくということです。

RACを使うための理解の手助けとして、そしてサンプルとして、ReadmeかDisignGuideLineを参照すると良いでしょう。

## Events

`Event`は、「なにかが起きた」ということを形式的に表現しています。
RACでは、Eventはコミュニケーションのための中心的役割です。
Eventは、ボタンを押した時、いくつかの情報をAPIから受け取った時、エラーが発生した時、長時間操作の完了を受け取った時など、さまざまなことを表現します。
いずれのケースにせよ、なにかがEventを生成し、Signalと通していくつかのObserverへと送信します。

`Event`は、値、または3つのいずれかのイベントを表すEnumです。

 * `Next`イベントは、新しい値をソースから提供します。
 * `Error`イベントはシグナルが終わる前にエラーが発生したことを示します。イベントは`ErrorType`とパラメータ化でき、こちらが決定したエラーの形でイベントの中に出現させられます。エラーが許されない場合、`NoError`を使うことで、どのようなエラーが出現することを防ぎます。
 * `Completed`イベントはすべてのシグナルが正しく終了したことを示します。値はソースからこれ以上送られることはありません。
 * `Interrupted`イベントはシグナルがキャンセルされたことで切断された、ということを示します。つまり操作は、成功もしていないし、失敗もしていません。

## Signals

`Signal`は時系列上での監視できる一連の`Events`です。

Signalは通常「実行中」の、イベントストリームとして表されます。例えば、notificationや、ユーザーインプットのように実行され、もしくは受け取るものです。
シグナル上でイベントは送信され、それはありとあらゆるオブザーバへに届きます。すべてのオブザーバはイベントを同じタイミングで受け取ります。

ユーザーは必ずEventにアクセスするために、Signalを監視しなければなりません。
Signalを監視することはなにも引き起こしません。
これはどういうことかというと、Signalは完全にproduer駆動／push駆動型なので、消費者（監視側）はそのライフサイクルの中で影響を持つことができません。Signalを監視している間、ユーザは同じ順序でEventを評価することができます。そこにランダムなアクセスはありません。

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

例えばブロックコールバック処理によるアプリの操作のかわりに、ブロックはただシンプルにEventをObserverに送るだけです。その間にSignalを受け取ることができるので、細かなコールバックの実装を隠蔽することができます。


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
RACではObserverはObserverとして表現され、イベントを処理します。

Observerはコールバックベースの`Signal.observe`か`SignalProducer.start`関数によって、暗黙的に作成できます。

## Actions

`Action`は、インプットから、なにかしらの処理を実行します。
実行中は、0以上のアウトプット、必要であればエラーを生成します。

Actionは例えばボタンをクリックした時な、ユーザインタラクションに対して処理を実行するときに有益です。
Actionは`property`によって自動的に無効化されます。
この無効の状態は、関連するアクションを無効にすることによって、 UI上で表現することができます。

`NSControl`、`UIControl`とのインタラクションについては、RACが`CocoaAction`としてObjective-Cへのブリッジを可能にします。


## Properties

`property`は`PropertyType`プロトコルで表現される、値を格納し、変更を感知してオブザーバに通知するための値です。

現在の値は`value`から取得することが可能です。
`producer`からは、SignalProducerを取得でき、それはPropertyの時系列上の変更にしたがって、変更された現在の値を返します。

`<~`演算子で様々なpropertyを紐付けることができます。 
これらのケースでは、propertyは`MutablePropertyType`でなければいけません。

* `property <~ signal`でSignalをひも付け、Signalによって送られた最新の値でpropertyの値を更新します。
* `property <~ producer`はSignalProducerを与え、Signalによって送られた最新の値でpropertyの値を更新します。
* `property <~ otherProperty`でproperty同士をひも付け、propertyが変化したタイミングでどのような状態でも同じように更新します。

`DynamicProperty`はObjective-CのAPIでKVC, KVOの処理が必要な場合の橋渡しするために使われます。例えば`NSOperation`などです。
気をつけてほしいのはほとんどのAppKitとUIKitはKVOをサポートしていないので、これらの変更感知は他のメカニズムで行われるべきなのです。
なにか動的な変更を必要とする場合、可能な限り`MutableProperty`を使うことをおすすめします！


## Disposables

`disposable`は`Disposable`プロトコルで表現される、メモリー管理とキャンセル機能を司るメカニズムです。

SignalProducerをスタートした時、disposableは返却されます。
このdisposableは呼び出し元がすでにスタートしている処理をキャンセルしたり（例えばバックグラウンド作業、通信リクエストなど）、すべての一時ファイルを削除し、Signalに`Interrupted`イベントをシグナルに送信します。

Signalをオブザーブするときもまた、disposableは返却されます。
Signalからの、なにかしらのイベントを受け取ったオブザーバはそれを破棄することができます。しかしそれはSignal自体にはなんの影響も及ぼしません。

さらに詳しく知りたければ、こちらを参照してください。

## Schedulers

`scheduler`は`SchedulerType`プロトコルで表現される、処理を実現するための実行キューです。

Signal、SignalProducerはSchedulerに対し、イベントを送信するように命令することができます。SignalProducerは加えて、特定Schedulerに対して処理を実行するよう命令することができます。
Schedulerはいわば`Central Dispatch queue`のようなものですが、Schedulerはdisposableを用いて、キャンセルすることができ、そしてそれは常に連続的に実行されます。
ImmediateSchedulerを除いて、スケジューラは同期実行を提供していません。これは、デッドロックを避けることができ、Signal、SignalProducerを使う理由になります。

Schedulerはまた` NSOperationQueue`にやや似ています。
しかし、Schedulerは再度並び替えることを許さず、また他に依存することもできません。



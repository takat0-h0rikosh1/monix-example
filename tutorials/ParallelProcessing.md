# Parallel Processing

+ Parallelism with Task
   + The Naive Way
   + Imposing a Parallelism Limit
   + Batched Observables
+ Observable.mapPrallelUnordered
+ Observable.mergeMap
+ Consumer.loadBalancer

+ タスクの並行性
   + ナイーブな方法
   + パラレル化の制限を課す
   + 観測可能なバッチ処理
+ ???
+ ???
+ ???

Monix はユースケースに応じて、並行処理を実現するための複数の方法を提供する。

```scala
// 評価時に必要
import monix.execution.Scheduler.Implicits.global

// タスクを利用するため
import monix.eval._

// Observable を利用するため
import monix.reactive._
```

### Parallelism with Task

Task により順序性を担保した並行処理を実行できる。

##### The Naive Way

Task.gather を利用し、順序性を担保しながら並行処理をおこなう。

しかし、並列処理を実際に行うにはタスクを非同期にする必要があり、<br>
単純な操作ではスレッドを fork する必要がある。

Task.apply を使う時は、Task.fork をそれぞれのタスクに適用する。

```scala
val items = 0 until 1000

// 実行に必要なすべてのタスクリスト
val tasks = items.map(i => Task(1 * 2))

// 並行処理
val aggregate = Task.gather(tasks).map(_.toList)

// 処理結果の評価
aggregate.foreach(println)
```

順序性が重要でない場合、`Task.gatherUnordered` を使う。<br>
ノンブロッキングな実行の場合により良い結果が得られる。

*寄り道*

3.0.0-RC2 の def apply[A](a: => A): Task[A]

タスクコンテキスト内に指定した処理をリフトし、<br>
タスク評価時に同期的に評価する

Monix 2.x シリーズでは、この操作は Task.evalAsync と同等のフォークを許容していた。

以前の動作と同じにしたけれ、Task.evalAsync に切り替えて、<br>
Task.val を Task.executeAsync と組み合わせて使用する。

##### Imposing a Parallelism Limit

Task.gather builder はすべてのタスクを並列して実行する可能性があり、<br>
これは、非効率につながる可能性がある。

例えば、10000件のHTTPリクエストを並列実行することは相手のサーバーを窒息させることに繋がるので、<br>
必ずしも懸命ではない。

これを解決するにはワークロードを並列タスクのバッチで分割し、次に並列化する。

```scala
val items = 0 until 1000
// 実行に必要なすべてのタスク 
val tasks = items.map(i => Task(i * 2))

// 10個のタスクを並列実行するバッチリストを構築 
val batches = tasks.sliding(10,10).map(b => Task.gather(b))

// バッチリストをシーケンスして、処理結果を平坦化
val aggregate = Task.sequence(batches).map(_.flatten.toList)

// 評価
aggregate.foreach(println)
```

Future.sequence ってのがあるけど、Scala の Future では上記の戦略を取ることができない

動作が厳密でシーケンスと並列性を区別できないため

この動作は、遅延または厳密なシーケンスを Future.sequence に渡すことで制御可能だが、<br>
明らかにエラーを引き起こしやすい。

##### Batched Observables

Observable.flatMap を利用することでバッチでリクエストできる。

```scala
// `bufferIntrospective` downstream がパツっているときには特定の `bufferSize` までバッファリングを行う。<br>
// バッファリングされたすべてのイベントのシーケンス全体を一度にストリームする。
val source = Observable.range(0,1000).bufferIntrospective(256)

// `Task` によってい実行されるバッチプロセス
val batched = source.flatMap { items =>
  
  // すべてのタスクリスト
  val tasks = items.map(i => Task(i * 2))
  
  // 並列して実行する10個のタスクのバッチを構築
  val batches = tasks.sliding(10,10).map(b => Task.gather(b))

  // バッチのシーケンスと結果の平坦化
  val aggregate = Task.sequence(batches).map(_.flatten)

  // flatMap に必要な、Observable へ変換
  Observable.fromIterator(aggregate)
}

// 評価
batched.toListL.foreach(println)
```

downstream がビジーの間に着信イベントをバッファリングし、<br>
その後、バッファを単一のバンドルとして発行します。

### Observable.mapParallelUnordered

並列性を達成するもう一つの方法として Observable.mapParallelUnordered がある。

```scala
val source = Observable.range(0,1000)

val processed = source.mapParallelUnordered(parallelism = 10) { i =>
  Task(i * 2)
}

processed.toListL.foreach(println)
```

`Task.gather` に対して、この演算子はソースによって通知された順序付けを維持しない。

少なくともひとつのワーカーがアクティブである限り、<br>
ソースがバックプレッシャーを受けることがないのでより効率的な実行につながる。

ただし、完了に時間がかかる単一の非同期タスクを実行する場合には、<br>
非効率になる可能性がある。

### Observable.mergeMap

もし、 `Observable.mapParallelUnordered` が Task によって動作する場合、<br>
`Observable.mergeMap` は `Observable` インスタンスをマージする。

```scala
val source = Observable.range(0,1000)
// The parallelism factor needs to be specified
val processed = source.mergeMap { i =>
  Observable.fork(Observable.eval(i * 2))
}

// Evaluation:
processed.toListL.foreach(println)
//=> List(0, 4, 6, 2, 8, 10, 12, 14...
```

mergeMap は source によって生成された observable stream が並行してサブスクライブされ、<br>
結果が非決定的であることを除いて concatMap と似ています。

疲れたからこのへんで...

TODO: -----

### Consumer.loadBalancer

# メモ

### 3.0.0-RC で Task.apply の挙動が非同期から同期処理に変わっている
https://twitter.com/OE_uia/status/1060787295022772225?s=19

### 去年のアドカレ → 「非同期処理を便利にする Monix」
https://qiita.com/ichizin/items/bc17e5a127a01189966c

### Monix.Task について

##### 昨年のアドカレより...

Future は未来の結果を保持する値であり、
Future を評価しようとしているスレッドが既に開始済みで完了している可能性もあるし、完了していない可能性もある。

上記は参照透過性がないので扱いづらい。

これに対し、Task は関数であり、こちらが実行を支持するまで何もしない。

### Monix is 何???

+ Monix は非同期プログラムを作成するための Scala / Scala.js ライブラリ。
+ Scala 向け ReactiveX から実装を開始した。
+ cats-effect の親実装の一つ

TypeLevel プロジェクトである Monix はパフォーマンスに関して妥協がなく、<br>
Scala の静的型付けで機能的なプログラミングを誇らしげに表現している。

ほう...

### ハイライト

+ 各データ型(Observable, Iterant, Task, Coeval) の必要な情報を公開しサポートする。
+ 必要なモジュールのみ使用することを可能にしている。
+ 非同期性のために設計されており JVM と Scala.js の両方で動作する。
+ テストカバレッジ、コード品質、APIドキュメントの良さをプロジェクトのポリシーとしている。

### Download and usage

2018/12/19 現在の最新 → 3.0.0-RC2

上は Cats に依存している。

##### Usage

```
libraryDependencies += "io.monix" %% "monix" % "3.0.0-RC2"
```

モジュール毎に分けてサブプロジェクトが定義してあるので、<br>
アラカルトに依存性を注入できる。

### Documentation and tutorial

https://monix.io/docs/3x/

##### Quick-start template

```bash
$ sbt new monix/monix-3x.g8
```

##### Project Structure

```
├── build.sbt
├── project
│   └── build.properties
└── src
    └── main
        └── scala
            └── example
                └── Hello.scala
```

##### build.sbt

```scala
/** [[https://monix.io]] */
val MonixVersion = "3.0.0-RC2"

lazy val root = (project in file(".")).
  settings(
    inThisBuild(List(
      organization := "com.example",
      scalaVersion := "2.12.8",
      version      := "0.1.0-SNAPSHOT"
    )),
    name := "monix-example",
    libraryDependencies ++= Seq(
      "io.monix" %% "monix" % MonixVersion
    )
  )
```

Scala の version を `2.12.8` に上げた。

### Sub Project

+ monix-execution
+ monix-catrap
+ monix-eval
+ monix-reactive
+ monix-tail

##### monix-exwcution

+ 同時実行性を扱うための低レベルの副作用ユーティリティを提供する。
+ JVM のプリミティブを公開し、scala.concurrent パッケージ上に構築する。

##### monix-catrap

+ Cats-Effect に構築された並行性管理のための汎用的な関数型ユーティリティを提供する。

##### monix-eval

+ 基本的な方法で純粋関数型な副作用を処理する Task型、Coeval型を提供する。

##### monix-reactive

+ Scala用 ReactiveX の慣習的な実装。
+ 高性能のストリーミング抽象である Observable型と隣接するユーティリティを公開する。

### Tutorials

+ Parallel Processing
   + 並行処理
+ Converting Scala's Future to Task(Video)
   + Future から Task への変換
+ Task' Bracket Cats-Effect's Resource and Streaming(Video)
   + Task をまとめる、Cat's-Effect's Resource and Streaming

### Parallel Processing

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




### Converting Scala's Future to Task


### Task's Bracket, Cat's-Effect's Resource and Streaming



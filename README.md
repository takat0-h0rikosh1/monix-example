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



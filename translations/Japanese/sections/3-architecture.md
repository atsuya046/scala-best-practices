## 3. アプリケーション・アーキテクチャ

<img src="https://raw.githubusercontent.com/monifu/scala-best-practices/master/assets/scala-logo-256.png"  align="right" width="128" height="128" />

### 3.1. Cakeパターンを使うべきではない

Cakeパターンは[理論的にとても良いアイデアだ](https://www.youtube.com/watch?v=yLbdw06tKPQ) -合成可能なmoduleとしてのtraitを使うことにより、コンパイル時の依存性注入の副作用とともに`import`を上書きする力が得られる。


実際に私が見たすべてのCakeパターンの実装はひどかった、新しいプロジェクトではなるべくCakeパターンを避け、すでにあるプロジェクトではCakeパターンのない状態に移行するべきである。


みんなデザインパターンへの理解が不十分なため、Cakeパターンを正しく実装していない。私は抽象的なmoduleとしてデザインされたtraitを使ったもしくはライフサイクルの問題に適切に注意を払ったCakeパターン実装を見たことがない。このずさんさは結果として大きな毛玉を引き起こす。ScalaがCakeパターンのようなものを許していることは、OOPの真の力を引き立てていて素晴らしい。しかしそれはそうすべきであるという理由ではない。もしも様々なコンポーネントでの依存性注入やデカップリングを行うことが目的である場合は、おそらくひどく失敗し同僚にメンテナンス業務の負担を与えてしまっているだろう。


実際にはすべての依存性注入を行わないもしくは（Playのcontrollersのように）端の部分だけにとどめるよう努めるべきである。コンポーネントは多くのものに依存している（*code smell*として）ので。もしコンポーネントが引数の初期化に強く依存しているならば、それもまた*code smell*である。痛みをラグの下に隠さず、かわりにそれを修正しよう。


### 3.2. PlayのGlobalオブジェクトに押し込んではいけない

私はこれを何度も何度も見てきました。

[PlayのGlobal](https://www.playframework.com/documentation/2.3.x/ScalaGlobal)オブジェクトはよりどころのないコードを押し込んでおくことのできるバケツではない。それはPlayの設定やライフサイクルに接続する以上の何ものでもない。

あなた独自のユーティリティのためのnamespaceを考えてください。


### 3.3. プロファイリングせず最適化をすべきではない

プロファイリングは最適化をするにあたって必要になるものである。決してプロファイリングを通して実際のボトルネックを見つけることなしに最適化にとりかかってはいけない。


これは、しばしば私たちが失敗してしまいシステムがどのように振る舞うかについての洞察と、確かな数値なしに最適化することにより複数の影響が起こるかもしれないことが理由である:

- コードもしくはアーキテクチャを複雑にしてしまうかもしれない、そのため後で全体的に最適化することを困難なものにしてしまう
- あなたの仕事はムダになるかもしれない、もしくは実際にはパフォーマンスの低下を引き起こすかもしれない

複数の取りうる戦略と積極的に実行すべきこととして、

- 良いプロファイラは見つけにくいボトルネックを教えてくれる。私のお気に入りはYourKit Profilerであるが、OracleのVisualVMは無料で十分に良いものである。
- [Dropwizard Metrics](https://dropwizard.github.io/metrics/3.1.0/)のようなライブラリをつかって稼働中のproductionシステムから測定基準を集め、[Graphite](http://graphite.wikidot.com/)などにそれらを追加していくことで、戦略は正しい方向に導かれていく。
- ベンチマークのコードを書いてソリューションを比較したが、ベンチマークは簡単ではないことと少なくとも[Google Caliper](https://code.google.com/p/caliper/)のようなライブラリを利用する必要があることに注意しなければならない。

全体として、測量は推測されない。

### 3.4. SHOULD be mindful of the garbage collector

Don't over allocate resources, unless you need to. We want to avoid
micro optimizations, but always be mindful about the effects
allocations can have on your system.

In the
[words of Martin Thomson](http://www.infoq.com/presentations/top-10-performance-myths),
if you stress the garbage collector, you'll increase the latency on
stop-the-world freezes and the number of such occurrences, with the
garbage collector acting like a GIL and thus limiting performance and
vertical scalability.

Example:

```scala
query.filter(_.someField.inSet(Set(name)))
```

This is a sample that occurred in our project due to a problem with
Slick's API. So instead of a `===` test, the developer chose to do an
`inSet` operation with a sequence of 1 element. This allocation of a
collection of 1 element happens on every method call. Now that's not
good, what can be avoided should be avoided.

Another example:

```scala
someCollection
 .filter(Set(a,b,c).contains)
 .map(_.name)
```

First of all, this creates a Set every single time, on each element of
our collection. Second of all, filter and map can be compressed in one
operation, otherwise we end up with more garbage and more time spent
building the final collection:

```scala
val isIDValid = Set(a,b,c)

someCollection.collect {
  case x if isIDValid(x) => x.name
}
```

A generic example that often pops up, exemplifying useless traversals
and operators that could be compressed:

```scala
collection
  .filter(bySomething)
  .map(toSomethingElse)
  .filter(again)
  .headOption
```

Also, take notice of your requirements and use the data-structure
suitable for your use-case. You want to build a stack? That's a
`List`. You want to index a list? That's a `Vector`. You want to
append to the end of a list? That's again a `Vector`. You want to push
to the front and pull from the back? That's a `Queue`. You have a set
of things and want to check for membership? That's a `Set`. You have a
list of things that you want to keep ordered? That's a
`SortedSet`. This isn't rocket science, just computer science 101.

We are not talking about extreme micro optimizations here, we aren't
even talking about something that's Scala, or FP, or JVM specific
here, but be mindful of what you're doing and try to not do
unnecessary allocations, as it's much harder fixing it later.

BTW, there is an obvious solution for keeping expressiveness while
doing filtering and mapping - lazy collections, which in Scala means
[Stream](http://www.scala-lang.org/api/current/index.html#scala.collection.immutable.Stream)
if you need memoization or
[Iterable](http://docs.oracle.com/javase/7/docs/api/java/lang/Iterable.html)
if you don't need memoization.

Also, make sure to read the
[Rule 3.3](#33-should-not-apply-optimizations-without-profiling) on
profiling.
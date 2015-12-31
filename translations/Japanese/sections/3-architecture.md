## 3. アプリケーション・アーキテクチャ

<img src="https://raw.githubusercontent.com/monifu/scala-best-practices/master/assets/scala-logo-256.png"  align="right" width="128" height="128" />

### 3.1. Cakeパターンを使うべきではない

Cakeパターンは[理論的にとても良いアイデアだ](https://www.youtube.com/watch?v=yLbdw06tKPQ) -合成可能なmoduleとしてのtraitを使うことにより、コンパイル時の依存性注入の副作用とともに`import`を上書きする力が得られる。


実際に私が見たすべてのCakeパターンの実装はひどかった、新しいプロジェクトではなるべくCakeパターンを避け、すでにあるプロジェクトではCakeパターンのない状態に移行するべきである。


みんなデザインパターンへの理解が不十分なため、Cakeパターンを正しく実装していない。私は抽象的なmoduleとしてデザインされたtraitを使ったもしくはライフサイクルの問題に適切に注意を払ったCakeパターン実装を見たことがない。このずさんさは結果として大きな毛玉を引き起こす。ScalaがCakeパターンのようなものを許していることは、OOPの真の力を引き立てていて素晴らしい。しかしそれはそうすべきであるという理由ではない。もしも様々なコンポーネントでの依存性注入やデカップリングを行うことが目的である場合は、おそらくひどく失敗し同僚にメンテナンス業務の負担を与えてしまっているだろう。


実際にはすべての依存性注入を行わないもしくは（Playのcontrollersのように）端の部分だけにとどめるよう努めるべきである。コンポーネントは多くのものに依存している（*code smell*として）ので。もしコンポーネントが引数の初期化に強く依存しているならば、それもまた*code smell*である。痛みをラグの下に隠さず、かわりにそれを修正しよう。


### 3.2. MUST NOT put things in Play's Global

I'm seeing this over and over again.

Folks,
[Play's Global](https://www.playframework.com/documentation/2.3.x/ScalaGlobal)
object is not a bucket in which you can shove your orphaned pieces of
code. Its purpose is to hook into Play's configuration and life-cycle,
nothing more.

Come up with your own freaking namespace for your utilities.

### 3.3. SHOULD NOT apply optimizations without profiling

Profiling is a prerequisite for doing optimizations. Never work on
optimizations, unless through profiling you discover the actual
bottlenecks.

This is because our intuition about how the system behaves often fails
us and multiple effects could happen by applying optimizations without
having hard numbers:

- you could complicate the code or the architecture, thus making it
  harder to apply later optimizations globally
- your work could be in vain or it could actually lead to more
  performance degradation

Multiple strategies available and you should preferably do all of
them:

- a good profiler can tell you about bottlenecks that aren't obvious,
  my favorite being YourKit Profiler, but Oracle's VisualVM is free
  and often good enough
- collect metrics from the running production systems, by means of a
  library such as
  [Dropwizard Metrics](https://dropwizard.github.io/metrics/3.1.0/)
  and push them in something like
  [Graphite](http://graphite.wikidot.com/), a strategy that can lead
  you in the right direction
- compare solutions by writing benchmarking code, but note that
  benchmarking is not easy and you should at least use a library like
  [Google Caliper](https://code.google.com/p/caliper/)

Overall - measure, don't guess.

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
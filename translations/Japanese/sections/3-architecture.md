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

### 3.4. ガーベジコレクタには注意すべき

必要以上にリソースを割り当てすぎてはいけない。我々は小さな最適化を避けたいが、いつもあなたのシステムが所有できるに割り当ての効果について留意しなければならない。

[words of Martin Thomson](http://www.infoq.com/presentations/top-10-performance-myths)の中で,もしガーベジコレクタに頼りきっているならば、stop the worldのフリーズにともなうレイテンシが増加し、それらの発生数を増やし、ガーベジコレクタがGILのように振る舞い、そのためパフォーマンスと垂直のスケーラビリティを制限してしまう。

例えば:

```scala
query.filter(_.someField.inSet(Set(name)))
```

これは我々のプロジェクトがSlickのAPIの問題に起因して発生したサンプルである。sequenceの1要素を`===`でテストする変わりに開発者が`inSet`による操作を選んでしまった。この1要素コレクションの割り当てはすべてのメソッド呼び出しで発生する。これはよろしくない、避けるべきである。

他の例:

```scala
someCollection
 .filter(Set(a,b,c).contains)
 .map(_.name)
```

まずはじめに、これはコレクション中のそれぞれの要素で毎回Setを作成する。次にfilterとmapは一つの操作にまとめることができる。そうでなければ最終的なコレクションを構築するまでに多くのゴミを生成し多くの時間を費やしてしまう。


```scala
val isIDValid = Set(a,b,c)

someCollection.collect {
  case x if isIDValid(x) => x.name
}
```

横断せずに圧縮して操作できる例として、ポップアップをする一般的な例:

```scala
collection
  .filter(bySomething)
  .map(toSomethingElse)
  .filter(again)
  .headOption
```

また、ユースケースに適したデータ構造を使うことに気をつけなければならない。スタックを作りたい？それなら`List`。listにindexを付けたい？それなら`Vector`。listの末尾に追加したい？それならまたも`Vector`。前に入れて後ろから出したい？それなら`Queue`。集合に対して構成員を確認したい？それなら`Set`。listに順序を持たせたい？それなら`SortedSet`。これはロケット科学ではない、コンピュータ科学101だ。

私たちはここでは極端に小さな最適化について話してはいない、ScalaやFPやJVMの仕様についても話していない、しかしあなたのやっていることに留意しなければならない、そして後になって修正が難しくなってしまうような不必要な割り当てをしないようにしなければならない。

ところで、表現力を維持しながらfilteringやmappingをするためのわかりやすいソリューションがある。Scalaで言うところの遅延Collectionだ。もしメモ化する必要があるならば[Stream](http://www.scala-lang.org/api/current/index.html#scala.collection.immutable.Stream)、メモ化する必要がないならば[Iterable](http://docs.oracle.com/javase/7/docs/api/java/lang/Iterable.html)。

また、プロファイリングするにあたって[Rule 3.3](#33-should-not-apply-optimizations-without-profiling)も読んでおいてください。
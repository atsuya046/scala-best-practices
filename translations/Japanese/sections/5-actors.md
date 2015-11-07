## 5. Akka Actors

<img src="https://raw.githubusercontent.com/monifu/scala-best-practices/master/assets/scala-logo-256.png"  align="right" width="128" height="128" />

【訳注】このドキュメントはAkka 2.3.4の時点で書かれたもので、現時点(2015/11/7)でEOLを迎えていますが、多くは2.4.0以降でも有用です。

### 5.1. 外部から受け取ったメッセージにのみ応答してActorの状態を更新"する必要がある"
<!--
### 5.1. SHOULD evolve the state of actors only in response to messages received from the outside
-->

Akka Actorを使用するとき、Actorが内包する可変状態は、外部から受け取ったメッセージにのみ応答して更新する必要があります。
代表的なアンチパターンは次のようなものです:
<!--
When using Akka actors, their mutable state should always evolve in
response to messages received from the outside. An anti-pattern that
comes up a lot is this:
-->

```scala
class SomeActor extends Actor {
  private var counter = 0
  private val scheduler = context.system.scheduler
    .schedule(3.seconds, 3.seconds, self, Tick)

  def receive = {
    case Tick =>
      counter += 1
  }
}
```

例ではActor自身が3秒ごとに状態を更新するようスケジューリングしています。
これは、非常に高くつく代償です。
Actorの動作は非決定論的であり、テストすることも困難なものになります。
<!--
In the example above the actor schedules a Tick every 3 seconds that
evolves its state. This is an extremely costly mistake. The actor's
behavior becomes totally non-deterministic and impossible to test
right.
-->

あなたが本当にActorの内部で定期的になにかをする必要があるのなら、そのスケジューラはActorの内部で初期化してはいけません。
外部に取り出しましょう。
<!--
If you really need to periodically do something inside an actor, then
that scheduler must not be initialized inside the actor. Take it out.
-->

### 5.2. `context.become` のみでActorの状態を変更"する必要がある"
<!--
### 5.2. SHOULD mutate state in actors only with context.become
-->

私たちはActor自身が状態を変更するようにします（多くのActorはそうしています）、
例えば次のようなものでも全く問題ありません:
<!--
Say we've got an actor that mutates its state (most actors do),
doesn't even matter what state that is:
-->

```scala
class MyActor extends Actor {
  val isInSet = mutable.Set.empty[String]

  def receive = {
    case Add(key) =>
      isInSet += key

    case Contains(key) =>
      sender() ! isInSet(key)
  }
}

// Messages
case class Add(key: String)
case class Contains(key: String)
```

私たちはScalaを使っていますので、実際には可能な限り純粋なものにしたいですし、
不変なデータ構造と純粋関数によって対処したくもあり、
[偶有的な複雑性](https://ja.wikipedia.org/wiki/%E9%8A%80%E3%81%AE%E5%BC%BE%E3%81%AA%E3%81%A9%E3%81%AA%E3%81%84)
（【訳注】原文では [accidental complexity](https://en.wikipedia.org/wiki/No_Silver_Bullet)）
の領域を小さくするために関数プログラミング(FP)を目指したいでしょう。
そして、上記のコードは純粋でもなく不変でもなく参照透過もないことをご説明しましょう。 ;-)
<!--
Since we are using Scala, we want to be as pure as practically
possible, we want to deal with immutable data-structures and pure
functions, we want to go FP to reduce the area for
[accidental complexity](https://en.wikipedia.org/wiki/No_Silver_Bullet)
and let me tell you, there's nothing pure, immutable or referentially
transparent about the above ;-)
-->

[context.become](http://doc.akka.io/docs/akka/2.3.4/scala/actors.html#Become_Unbecome)を見てみましょう:
<!--
Meet [context.become](http://doc.akka.io/docs/akka/2.3.4/scala/actors.html#Become_Unbecome):
-->

```scala
import collection.immutable.Set

class MyActor extends Actor {
  def receive = active(Set.empty)

  def active(isInSet: Set[String]): Receive = {
    case Add(key) =>
      context become active(isInSet + key)

    case Contains(key) =>
      sender() ! isInSet(key)
  }
}
```

すぐにピンと来ないとしても、将来、10の状態を扱い、何十もの状態に遷移でき、それに付随する作用のある状態マシンをモデル化するときがくれば、理解できるでしょう。
<!--
If that doesn't instantly ring a bell, just wait until you'll have to
model a state machine with 10 states in it and dozens of possible
transitions and effects to go along with it, then you'll get it.
-->

### 5.3. 非同期クロージャのなかでActorの内部状態を漏ら"してはならない"
<!--
### 5.3. MUST NOT leak the internal state of an actor in asynchronous closures
-->

もう一度、可変状態のもつ問題に焦点をあてましょう:
<!--
Again with the mutable state, spot the problem:
-->

```scala
class MyActor extends Actor {
  val isInSet = mutable.Set.empty[String]

  def receive = {
    case Add(key) =>
      for (shouldAdd <- validate(key)) {
        if (shouldAdd) isInSet += key
      }

    // ...
  }

  def validate(key: String): Future[Boolean] = ???
}
```

カオスは地獄の扉を開き、ありとあらゆる種類の非決定論的なバグがマルチスレッドの問題として当然のように顕在化する事が確実です。
これは、非同期実行され、変数を束縛し、その状況を免れることを意図していない関数における一般的な問題です。
[Spores](http://docs.scala-lang.org/sips/pending/spores.html)は、マクロ有効なクロージャのための提案で、この問題をより安全にするとされていますが、まだ注意してください。
<!--
Chaos ensues, hell's doors open for a whole range of non-deterministic
bugs that could happen due to multi-threading issues. This is a
general problem with functions that execute asynchronously and that
capture variables that aren't meant to escape their
context. [Spores](http://docs.scala-lang.org/sips/pending/spores.html)
is a proposal for macros-enabled closures that are supposed to make
this safer, but until then just be careful.
-->

まず第一に、状態を変更するために `context.become` の使用に関するルールを参照してください。
それは、正しい方向への一歩として先述しています。
次に、Futureが完了したときに、別のメッセージを私たちのActorに送ることによって、この問題に対処する必要があります。
<!--
First of all, see the rule about using `context.become` for mutating
state, which is already a step in the right direction. And then you
need to deal with this by sending another message to our actor when
our future is done:
-->

```scala
import akka.pattern.pipeTo

class MyActor extends Actor {
  val isInSet = mutable.Set.empty[String]

  def receive = {
    case Add(key) =>
      val f = for (isValid <- validate(key))
        yield Validated(key, isValid)

      // 結果をメッセージとして、このActorに送り返します
      f pipeTo self

    case Validated(key, isValid) =>
      if (isValid) isInSet += key

    // ...
  }

  def validate(key: String): Future[Boolean] = ???
}

// Messages
case class Add(key: String)
case class Validated(key: String, isValid: Boolean)
```
<!--
```scala
import akka.pattern.pipeTo

class MyActor extends Actor {
  val isInSet = mutable.Set.empty[String]

  def receive = {
    case Add(key) =>
      val f = for (isValid <- validate(key))
        yield Validated(key, isValid)

      // sending the result as a message back to our actor
      f pipeTo self

    case Validated(key, isValid) =>
      if (isValid) isInSet += key

    // ...
  }

  def validate(key: String): Future[Boolean] = ???
}

// Messages
case class Add(key: String)
case class Validated(key: String, isValid: Boolean)
```
-->

そしてもちろん、最後の一つが完了するまで、それ以上のリクエストを受け入れない状態マシンをモデル化することができます。
私たちは、可変コレクションを取り除き、またバックプレッシャを取り入れることができます。
（つまり、次のアイテムを送信することができたとき、送信者に伝える必要があります。）:
<!--
And of course, we could be modeling a state-machine that doesn't
accept any more requests until the last one is done. Let us also get
rid of that mutable collection and also introduce back-pressure
(i.e. we need to tell the sender when it can send the next item):
-->

```scala
import akka.pattern.pipeTo

class MyActor extends Actor {
  def receive = idle(Set.empty)

  def idle(isInSet: Set[String]): Receive = {
    case Add(key) =>
      // 結果をメッセージとして、このActorに送り返します
      validate(key).map(Validated(key, _)).pipeTo(self)

      // 検証を待ちます
      context.become(waitForValidation(isInSet, sender()))
  }

  def waitForValidation(set: Set[String], source: ActorRef): Receive = {
    case Validated(key, isValid) =>
      val newSet = if (isValid) set + key else set
      // 完了したことを応答します
      source ! Continue
      // 新しいリクエストを受け取れるよう待機状態に移ります
      context.become(idle(newSet))

    case Add(key) =>
      sender() ! Rejected
  }

  def validate(key: String): Future[Boolean] = ???
}

// Messages

case class Add(key: String)
case class Validated(key: String, isValid: Boolean)
case object Continue
case object Rejected
```
<!--
```scala
import akka.pattern.pipeTo

class MyActor extends Actor {
  def receive = idle(Set.empty)

  def idle(isInSet: Set[String]): Receive = {
    case Add(key) =>
      // sending the result as a message back to our actor
      validate(key).map(Validated(key, _)).pipeTo(self)

      // waiting for validation
      context.become(waitForValidation(isInSet, sender()))
  }

  def waitForValidation(set: Set[String], source: ActorRef): Receive = {
    case Validated(key, isValid) =>
      val newSet = if (isValid) set + key else set
      // sending acknowledgement of completion
      source ! Continue
      // go back to idle, accepting new requests
      context.become(idle(newSet))

    case Add(key) =>
      sender() ! Rejected
  }

  def validate(key: String): Future[Boolean] = ???
}

// Messages

case class Add(key: String)
case class Validated(key: String, isValid: Boolean)
case object Continue
case object Rejected
```
-->

えぇ、Actorベースの設計は巧妙にできます。
<!--
Yeap, actor-based designs can get tricky.
-->

### 5.4. バックプレッシャを"する必要がある"
<!--
### 5.4. SHOULD do back-pressure
-->

ところで、あなたが複数の値をプロデュースするActor（プロデューサ）を持っているとして - 例えば、
RabbitMQやMySQLのテーブルを使った低能な独自キューから読みだすアイテムであったり、
Actorが特定のディレクトリなどに現れたことを見つけてすぐに処理しなければいけない監視対象のファイル
のようなものです。
このプロデューサは、いくつかのActor（コンシューマ）に仕事を委譲する必要があります。
<!--
Say you've got an actor that produces values - like reading items from
a RabbitMQ or your own half-assed queue stored in a MySQL table, or
files that have to be observed and processed as soon as the actor sees
them popping up in a certain directory and so on. This producer needs
to push work into a number of variable actors.
-->

問題:
<!--
Problems:
-->

1. もし、メッセージキューに際限がなければ、低速なコンシューマがキューを溢れさせてしまう
2. 配分が非効率だと、あるワーカーが多数のアイテムを処理しているのに、あるワーカーは静止していることになりえる
<!--
1. if the queue of messages is unbounded, with slow consumers that
   queue can blow up
2. distribution can be inefficient, as a worker could end up with
   multiple pending items whereas another worker could be standing
   still
-->

正しく、心配無用な設計はこのようにします:
<!--
A correct, worry-free design does this:
-->

- ワーカーは要求を通知しなければいけません（つまり、アイテムを処理できるようになったときに要求を通知）
- プロデューサはワーカーからの要求があったときのみアイテムを産出しなければいけません
<!--
- workers must signal demand (i.e. when they are ready for processing more items)
- the producer must produce items only when there is demand from workers
-->

コメントを付けたサンプルを用意しました:
<!--
Here's a detailed sample with comments:
-->

```scala
/**
 * 上流が次のアイテムを送ることができることを意味する応答メッセージ
 */
case object Continue

/**
 * ポーリング状態の間、データソースから継続してポーリングするためのプロデューサが使用するメッセージ
 */
case object PollTick

/**
 * 2つの状態をもつ状態マシン:
 *
 *  - Standby, おそらく下流へ送られることを待っているアイテムがキューにありますが、
 *             Actorは要求の通知が送られてくることを待機していることを意味します。
 *
 *  - Polling, 下流から要求がありますが、
 *             Actorは新たなアイテムの発生を待っていることを意味します。
 *
 * 重要: プロトコルの問題として、このActorは複数のContinueイベントを受け取ることができません。
 *       下流のルーターは、次のContinueイベントをこのActorに送る前に配送されるアイテムを待つ必要があります。
 */
class Producer(source: DataSource, router: ActorRef) extends Actor {
  import Producer.PollTick

  override def preStart(): Unit = {
    super.preStart()
    // （Actorは外部のメッセージに応じてのみ変化する必要がある、という）別のルールを無視していますが、
    // これは教育的意図のためにそうしています。
    context.system.scheduler.schedule(1.second, 1.second, self, PollTick)
  }

  // Actorはstandby状態で開始
  def receive = standby

  def standby: Receive = {
    case PollTick =>
      // 無視

    case Continue =>
      // 要求が通知されたため、次のアイテムの送信を試行
      source.next() match {
        case None =>
          // 有効なアイテムがなければ、ポーリングモードに移行
          context.become(polling)

        case Some(item) =>
          // アイテムがあれば、下流にこれを送信し、standby状態にとどまる
          router ! item
      }
  }

  def polling: Receive = {
    case PollTick =>
      source.next() match {
        case None =>
          () // 無視 - ポーリング状態のまま
        case Some(item) =>
          // アイテムがあり、要求がある
          router ! item
          // standbyに移行
          context.become(standby)
      }
  }
}

/**
 * ルーターは上流のプロデューサとワーカーの中間者で、
 * （プロデューサをシンプルに保つために）要求の追跡を行います。
 *
 * NOTE: プロデューサのプロトコルが尊重される必要があります - ですから
 *       ワーカーで処理するためにアイテムが下流へ送信された場合のみ
 *       その後にContinueを上流のプロデューサへ通知します。
 */
class Router(producer: ActorRef) extends Actor {
  var upstreamQueue = Queue.empty[Item]
  var downstreamQueue = Queue.empty[ActorRef]

  override def preStart(): Unit = {
    super.preStart()
    // 初期の要求を上流へ通知
    producer ! Continue
  }

  def receive = {
    case Continue =>
      // 下流から要求が通知され、送信できるアイテムがあるなら送信を行い、
      // そうでなければ下流のコンシューマにエンキューします。
      if (upstreamQueue.isEmpty) {
        downstreamQueue = downstreamQueue.enqueue(sender)
      }
      else {
        // キューにあるアイテムを受け取ってから、上流に通知を送る必要はありません。
        // ただ下流にそれらを送信するだけです。
        val (item, newQueue) = upstreamQueue.dequeue
        upstreamQueue = newQueue
        sender ! item

        // 別のアイテムのために上流に要求を通知
        producer ! Continue
      }

    case item: Item =>
      // 上流からアイテムが通知され、キューにコンシューマがあるなら
      // それを下流へ通知し、そうでなければそれをエンキューします。
      if (downstreamQueue.isEmpty) {
        upstreamQueue = upstreamQueue.enqueue(item)
      }
      else {
        val (consumer, newQueue) = downstreamQueue.dequeue
        downstreamQueue = newQueue
        consumer ! item

        // 別のアイテムのために上流に要求を通知
        producer ! Continue
      }
  }
}

class Worker(router: ActorRef) extends Actor {
  override def preStart(): Unit = {
    super.preStart()
    // 初期の要求を上流へ通知
    router ! Continue
  }

  def receive = {
    case item: Item =>
      process(item)
      router ! Continue
  }
}
```
<!--
```scala
/**
 * Message signifying acknowledgement that upstream can send the next
 * item.
 */
case object Continue

/**
 * Message used by the producer for continuously polling the
 * data-source, while in the polling state.
 */
case object PollTick

/**
 * State machine with 2 states:
 *
 *  - Standby, which means there probably is a pending queue of items waiting to
 *    be sent downstream, but the actor is waiting for demand to be signaled
 *
 *  - Polling, which means that there is demand from downstream, but the
 *    actor is waiting for items to happen
 *
 * IMPORTANT: as a matter of protocol, this actor must not receive multiple
 *            Continue events - downstream Router should wait for an item
 *            to be delivered before sending the next Continue event to this
 *            actor.
 */
class Producer(source: DataSource, router: ActorRef) extends Actor {
  import Producer.PollTick

  override def preStart(): Unit = {
    super.preStart()
    // this is ignoring another rule I care about (actors should evolve
    // only in response to external messages), but we'll let that be
    // for didactical purposes
    context.system.scheduler.schedule(1.second, 1.second, self, PollTick)
  }

  // actor starts in standby state
  def receive = standby

  def standby: Receive = {
    case PollTick =>
      // ignore

    case Continue =>
      // demand signaled, so try to send the next item
      source.next() match {
        case None =>
          // no items available, go in polling mode
          context.become(polling)

        case Some(item) =>
          // item available, send it downstream,
          // and stay in standby state
          router ! item
      }
  }

  def polling: Receive = {
    case PollTick =>
      source.next() match {
        case None =>
          () // ignore - stays in polling
        case Some(item) =>
          // item available, demand available
          router ! item
          // go in standby
          context.become(standby)
      }
  }
}

/**
 * The Router is the middleman between the upstream Producer and
 * the Workers, keeping track of demand (to keep the producer simpler).
 *
 * NOTE: the protocol of Producer needs to be respected - so
 *       we are signaling a Continue to the upstream Producer
 *       after and only after a item has been sent downstream
 *       for processing to a worker.
 */
class Router(producer: ActorRef) extends Actor {
  var upstreamQueue = Queue.empty[Item]
  var downstreamQueue = Queue.empty[ActorRef]

  override def preStart(): Unit = {
    super.preStart()
    // signals initial demand to upstream
    producer ! Continue
  }

  def receive = {
    case Continue =>
      // demand signaled from downstream, if we have items to send
      // then send, otherwise enqueue the downstream consumer
      if (upstreamQueue.isEmpty) {
        downstreamQueue = downstreamQueue.enqueue(sender)
      }
      else {
        // no need to signal demand upstream, since we've got queued
        // items, just send them downstream
        val (item, newQueue) = upstreamQueue.dequeue
        upstreamQueue = newQueue
        sender ! item

        // signal demand upstream for another item
        producer ! Continue
      }

    case item: Item =>
      // item signaled from upstream, if we have queued consumers
      // then signal it downstream, otherwise enqueue it
      if (downstreamQueue.isEmpty) {
        upstreamQueue = upstreamQueue.enqueue(item)
      }
      else {
        val (consumer, newQueue) = downstreamQueue.dequeue
        downstreamQueue = newQueue
        consumer ! item

        // signal demand upstream for another item
        producer ! Continue
      }
  }
}

class Worker(router: ActorRef) extends Actor {
  override def preStart(): Unit = {
    super.preStart()
    // signals initial demand to upstream
    router ! Continue
  }

  def receive = {
    case item: Item =>
      process(item)
      router ! Continue
  }
}
```
-->

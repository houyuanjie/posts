# IO Monad

IO 的意思是 "Input/Output", 这其实就是我们一直所说的 "函数" 或者 "计算" 的一个代名词, 因为函数的一个基本特征就是它是有输入 (参数) 和输出 (返回值) 的

在这篇文章中将要介绍的 IO Monad 是我们上两篇 [表达式和副作用](./04_%E8%A1%A8%E8%BE%BE%E5%BC%8F%E5%92%8C%E5%89%AF%E4%BD%9C%E7%94%A8.md) 以及 [通用的语言](./05_%E9%80%9A%E7%94%A8%E7%9A%84%E8%AF%AD%E8%A8%80.md) 的延申, 我假定你是在了解了相关概念后开始读这一篇的

## 一个用来承载计算的数据类型

在 [表达式和副作用](./04_%E8%A1%A8%E8%BE%BE%E5%BC%8F%E5%92%8C%E5%89%AF%E4%BD%9C%E7%94%A8.md) 一篇中我们介绍了函数式程序员们包装副作用的一个技巧: 将有副作用的代码包装进函数中; 实际上哪怕一段计算没有副作用, 函数式的程序员们也喜欢用函数来包装

他们设计了一个非常简单的数据类型 `IO`, 只为了承载一个函数

```scala
case class IO[A](run: () => A)
```

如果不为它增加任何语法, 那么它的功能也就仅限于承载计算了

```scala
val io1 = IO(() => 1 + 2)

io1.run   // 纯计算

io1.run() // 求值得到 3

val io2 = IO(() => {
  println("Hello")
  2 + 3
})

io2.run   // 含有副作用的计算

io2.run() // 求值得到 5
          // 同时运行了计算中的副作用
          // 控制台打印出了 "Hello"

val io3 = IO(() => println("World"))

io3.run   // 纯副作用

io3.run() // 运行副作用
          // 控制台打印出了 "world"
          // 没有得到值
```

## 可组合的计算

在 [通用的语言](./05_%E9%80%9A%E7%94%A8%E7%9A%84%E8%AF%AD%E8%A8%80.md) 一篇中我们介绍了让一个语言 "通用" 的关键在于 "可组合性", 除此之外当然还有 "可进入" 和 "可求值", 我们希望我们设计的 `IO` 类型也是通用的, 所以我们应当为它实现 Monad 的模式

```scala
trait Monad[F[_]]:
  def pure[A](value: A): F[A]
  def flatMap[A, B](fa: F[A])(f: A => F[B]): F[B]

  def map[A, B](fa: F[A])(f: A => B): F[B] =
    flatMap(fa)(a => pure(f(a)))
```

`pure` 的实现是简单的, 因为我们的数据类型只是一个简单到只有一个成员的积类型, 我们只需要把值包装进函数即可

```scala
def pure[A](value: A): IO[A] =
  IO(() => value)
```

`flatMap` 则要动点心思了, 请你想一想, 既然 IO 中包含的是一个函数, 我们要对值进行转换时是不是应当先求值再转换呢

```scala
// flatMap 参数
val fa: IO[A] = ???
val f: A => F[B] = ???

// 先求值
val value: A =
  fa.run()

// 再转换
val result: F[B] =
  f(value)

// 然后将这段计算包装进函数
val func: () => B =
  () => result.run()

// 最后将函数打包进 IO
val io: IO[B] =
  IO(func)
```

我们将上面的步骤合起来, 就能得到 `flatMap` 的实现啦

```scala
def flatMap[A, B](fa: IO[A])(f: A => IO[B]): IO[B] =
  IO(() => f(fa.run()).run())
```

我们再设计一些语法, `delay`, `println`, `*>` 以及 `as`

完整的代码如下:

```scala
object IO:

  def delay[A](value: => A): IO[A] =
    IO(() => value)

  def println(value: => Any): IO[Unit] =
    IO.delay(println(value))

object IOInstances:
  given ioMonad: Monad[IO] with

    def pure[A](value: A): IO[A] =
      IO(() => value)

    def flatMap[A, B](fa: IO[A])(f: A => IO[B]): IO[B] =
      IO(() => f(fa.run()).run())

object IOSyntax:
  extension [A](io: IO[A])(using ioMonad: Monad[IO])

    def *>[B](other: IO[B]): IO[B] =
      ioMonad.flatMap(io)(_ => other)

    def as[B](value: B): IO[B] =
      ioMonad.map(io)(_ => value)
```

`*>` 只是 `flatMap` 忽略参数的别名, 它代表我们 **先运行一个计算, 忽略它的结果, 然后运行下一个运算**

但请你仔细回想 [表达式和副作用](./04_%E8%A1%A8%E8%BE%BE%E5%BC%8F%E5%92%8C%E5%89%AF%E4%BD%9C%E7%94%A8.md) 中的内容, 当使用 `*>` 组合两个表达式时, 这意味着我们组合起来的程序计算结果将被忽略, 而计算中副作用将被 **顺序** 执行, **我们正在用表达式组合的方式模拟命令式中的 "语句"!**, 所以也有人说 "IO Monad 的 flatMap 不过相当于分号而已"

`as` 只是 `map` 忽略参数的别名, 它代表我们 **不管计算是什么, 结果是...**, 我们同样和命令式的程序相比, 这更类似于 "及早返回" 这样的概念

有了这些语法, 我们写段程序试试吧

```scala
@main
def run(): Unit =
  import IOInstances.given
  import IOSyntax.*

  val effect =
    IO.println("Hello")
      *> IO.delay(Thread.sleep(2000))
      *> IO.println("world")

  effect.as(println("running")).run()
```

上面这段程序将输出

```
running
Hello
world
```

这就是我们组合含副作用计算的方式

## 库

实际上, 我们不需要自己手写这些 `IO`, 因为这些是函数式程序员们常见的东西, 有现成的库可以用, 在 Scala 中主流的目前有两个: "Cats Effect" 和 "ZIO"

"Cats Effect" 属于 "Cats" 生态, 他们提供了一系列经典的函数式开发中常用的抽象和实现, Cats Effect 的 `IO` 类型正是我写本篇文章的参考

"ZIO" 则是一个封装的比较好的框架, 屏蔽了很多函数式中的概念, 给程序员了一套自己的 "语言", 而且官方维护了很大的一批生态, 属于那种类似 "Spring 全家桶" 的开发体验

我的建议是常规的开发中实际上不需要 "IO" 这么函数式的东西, 当你需要处理并发和异步任务时才需要考虑引入这些库, 他们在 IO 的基础上提供了 "Fiber", 这将是你使用 IO Monad 的理由

## 一些我自己的想法

到 IO Monad 为止, 我能想到的函数式中的常见要素已经差不多了, 这个系列的主线也以我预想的顺序顺利讲完了

后面涉及的内容就作为补充, 想到什么说点什么吧, 目前能想到的话题有:

- Tagless Final
- Free Monad
- Effectful Stream
- doobie, tapir, 等等有意思的库

看到这里, 如果你对前面这几篇文章或者后面要聊的话题有什么意见/建议的话, 请毫不吝啬地向我反馈, 因为这将帮助我写出更好的内容

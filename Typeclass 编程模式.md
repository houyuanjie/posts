# Typeclass 编程模式

&emsp;&emsp;简单来说, 使用 typeclass 来编程的基本思想就是 **将数据与其操作分离**, 如果你还不明白我在说什么, 或者感到这样的思想在你的编程实践中也许从来没有听人说过, 而却隐隐约约地有所察觉时, 继续看下去, 你会有所收获

## 面向对象

&emsp;&emsp;在经典的面向对象编程 (OOP) 模式中, 我们所有那些构思精巧的设计模式无外乎在做三件事: 封装, 继承, 多态. 封装指导我们应当将数据和对它的操作封装成类; 继承指导我们将公共的部分作为父类, 用子类来做扩展; 多态指导我们重写父类中某些操作的实现; 这三种手法组合起来, 我们就能写出庞大/复杂/起作用的程序来

&emsp;&emsp;就用这段大家在最初学习面向对象时可能都见到过的代码举例吧:

```scala
abstract class Animal:
  def say(): String

class Cat extends Animal:
  def say(): String = 
    "喵喵~"

class Dog extends Animal:
  def say(): String =
    "汪汪!"
```

&emsp;&emsp;代码是用 Scala 3 写的, 这是一个语法和 Python 非常相似的 JVM 语言, 我相信对于有编程基础的同学, 这样的语法看起来非常亲切

&emsp;&emsp;假设现在我们需要向上面的例子同时添加一种操作 (swim) 和一种数据 (Turtle), 我们需要改动多少代码呢? 让我来写写看:

```scala
abstract class Animal:
  def say(): String
  def swim(): String

class Cat extends Animal:
  def say(): String = 
    "喵喵~"

  def swim(): String =
    "猫猫怕水!"

class Dog extends Animal:
  def say(): String =
    "汪汪!"

  def swim(): String =
    "吼吼吼~"

class Turtle extends Animal:
  def say(): String =
    "???"

  def swim(): String =
    "嗖嗖嗖..."
```

&emsp;&emsp;细心的你发现什么问题了吗?

&emsp;&emsp;没错! 我们修改了几乎所有的代码! 这还只是一种操作加上一个数据, 随着需要添加的操作和数据数增加, 我们需要进行的修改量将急剧地/爆炸式地增加, 其中潜藏有 bug 的可能性也在随之增加, 最终成为无人可以调试, 无人可以理解, 无人可以进行有效维护或重构的 "屎山" (吐槽一下: 不知道是谁发明出的一个如此贴切的词)

&emsp;&emsp;这实际上违背了面向对象编程五原则 ("SOLID") 之一的 "O"

&emsp;&emsp;开闭原则 (Open Closed Principle):

>  "software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification"
>
> "软件实体 (类, 模块, 函数等) 应当对扩展开放, 但对修改关闭"

&emsp;&emsp;我们使用面向对象三种基本手法构建出的软件竟然违反了面向对象重要的原则, 这说明我们的设计出了问题, 哪怕是在猫狗动物这样简单的问题中, 也不能一股脑/无差别地运用封装/继承/多态这些工具, 那我们应当怎么做呢?

## 访问者模式

&emsp;&emsp;"GoF" 曾在他们的巨著 *设计模式: 可复用面向对象软件的基础* 一书中提出了一种复杂而有效的设计模式 - "访问者模式"; 事实上这种模式足够复杂以至于程序员们在一定程度上宁愿放弃开闭原则而拒绝对其进行实践, 我在这里尝试重构上面的代码来简单实现一下

```scala
abstract class Animal:
  def accept[T](visitor: Visitor[T]): T

abstract class Visitor[T]:
  def visit(cat: Cat): T
  def visit(dog: Dog): T
  def visit(turtle: Turtle): T

// 定义数据类

class Cat extends Animal:
  def accept[T](visitor: Visitor[T]): T =
    visitor.visit(this)

class Dog extends Animal:
  def accept[T](visitor: Visitor[T]): T =
    visitor.visit(this)

class Turtle extends Animal:
  def accept[T](visitor: Visitor[T]): T =
    visitor.visit(this)

// 定义访问者

class AnimalSay extends Visitor[String]:
  def visit(cat: Cat) =
    "喵喵~"

  def visit(dog: Dog) =
    "汪汪!"

  def visit(turtle: Turtle) =
    "???"

class AnimalSwim extends Visitor[String]:
  def visit(cat: Cat) =
    "猫猫怕水!"

  def visit(dog: Dog) =
    "吼吼吼~"

  def visit(turtle: Turtle) =
    "嗖嗖嗖..."
```

&emsp;&emsp;上面的代码虽然看起来很长, 但是逻辑是很清楚的, 可能唯一需要我要解释一下的是这里的语法

```
def accept[T](visitor: Visitor[T]): T
```

&emsp;&emsp;有类型参数 `[T]` 表明这是一个泛型函数, 其返回值由函数参数 `visitor` 的类型参数确定, 如果访问者是例子中的 `AnimalSay` 或 `AnimalSwim`, 则 `accept` 返回 `String`, 再比如我新增一个 `AnimalCanFly extends Visitor[Boolean]` 的访问者, 则 `accept` 返回 `Boolean`

&emsp;&emsp;有了访问者模式, 我们把数据的每种操作就分离到了每一个访问者中, 我们每添加一种操作, 就只需要修改抽象访问者并添加一个新的访问者实现即可! 尽管我们添加一种新的数据时仍需修改所有访问者, 但是代码的耦合度已经大大降低, 从直观上来讲, 我们让 **添加一种操作的同时添加一种数据** 的问题简单了一半!

## 表达问题

&emsp;&emsp;借助 "GoF" 的智慧, 我们成功的解决了问题的一半, 那你会自然的问: "那问题的另一半?" 是的, 问题的另一半怎么办呢, 这是一个好问题?

&emsp;&emsp;实际上, 这种 "添加一种操作的同时添加一种数据" 的问题有一个专门的名字 - 表达问题 (Expression Problem), 迄今为止我们仍缺乏一种有效的设计来彻底解决它, 这样的表达问题不仅仅是一直悬在所有面向对象程序员头上的一朵乌云, 它同时也深深地困扰着函数式编程的程序员们

&emsp;&emsp;从上面的例子中我们已经看到面向对象的这些工具功能强大却极易滥用, 不是所有的程序员都能坚守 "SOLID" 的原则, 把 23 种设计模式作为最佳实践; 而与面向对象程序设计不同, 函数式程序员们的工具箱中没有封装/继承/多态这种多面性的工具, 对他们来说, 所有的一切都是函数, 他们怎么看待这样的问题呢

## 从 @FunctionalInterface 开始

&emsp;&emsp;如果你熟悉 Python 中的 lambda 表达式或者 Java 中的 @FunctionalInterface, 你就会发现从上面我们一直所说的 **操作** 实际上就是函数, 我们试着用一种函数式接口的方式表达出来

```java
@FunctionalInterface
interface AnimalSay {
  String say();
}
```

&emsp;&emsp;当然, 我们如果让 `Cat`, `Dog` 这些 `Animal` 直接实现 `AnimalSay`, 那实际上和上面我们最开始写的代码就没有什么区别了, 我们希望把这些接口实例化, 下面继续用 Scala 3 语言作为演示, Scala 中的 `trait` 与 Java 中的 `interface` 等价, `=>` 与 Java 中的 `->` 等价

```scala
abstract class Animal
class Cat extends Animal
class Dog extends Animal
class Turtle extends Animal

trait AnimalSay:
  def say(): String

val catSay: AnimalSay =
  () => "喵喵~"

val dogSay: AnimalSay =
  () => "汪汪!"

val turtleSay: AnimalSay =
  () => "???"

trait AnimalSwim:
  def swim(): String

val catSwim: AnimalSwim =
  () => "猫猫怕水!"

val dogSwim: AnimalSwim =
  () => "吼吼吼~"

val turtleSwim: AnimalSwim =
  () => "嗖嗖嗖..."

```

&emsp;&emsp;不知不觉中, 我们已经把操作和数据完全分离, 但是没有访问者, 我们怎么知道哪个操作对应哪个数据呢?

&emsp;&emsp;仔细想想, 我们的类型是可以接受类型参数的, 我们可以通过为这些函数式接口打上标签的方式来实现操作之间的区分

```scala
trait AnimalSwim[A]:
  def swim(): String

val catSwim: AnimalSwim[Cat] =
  () => "猫猫怕水!"

val dogSwim: AnimalSwim[Dog] =
  () => "吼吼吼~"

val turtleSwim: AnimalSwim[Turtle] =
  () => "嗖嗖嗖..."
```

&emsp;&emsp;这样, 我们又借助函数式的思想, 探索出了一种可以进一步降低操作和数据耦合度的编程模式, 函数式程序员们称这种模式为 **Typeclass**

## 编译器的帮助

&emsp;&emsp;如果你使用过 Spring Framework, 那你一定对它提供的依赖注入功能印象深刻, Spring 是支持  **按类型注入** 的, 我说到这里也许你已经能明白在上面的例子中为类型加上参数来进行标注的目的了

&emsp;&emsp;Scala 编译器提供了一种编译时依赖注入的功能, 在下面的例子中, 我们将通过运用这种 Typeclass + 依赖注入的方式将操作和数据再绑定起来

&emsp;&emsp;要启用这种依赖注入的支持, 我们要将值声明 `val` 改为给予声明 `given`, 再通过 `using` 让编译器注入这个值


```scala
trait AnimalSwim[A]:
  def swim(): String

given catSwim: AnimalSwim[Cat] =
  () => "猫猫怕水!"

given dogSwim: AnimalSwim[Dog] =
  () => "吼吼吼~"

given turtleSwim: AnimalSwim[Turtle] =
  () => "嗖嗖嗖..."

def swim[A](using 
   animalSwim: AnimalSwim[A]
): String =
  animalSwim.swim()
```

&emsp;&emsp;我再来写些使用时的代码, 你也许能理解得更明白

```scala
val str1: String = swim[Cat]
// str1 = "猫猫怕水!"
val str2: String = swim[Dog]
// str2 = "吼吼吼~"
val str3: String = swim[Turtle]
// str3 = "嗖嗖嗖..."
```

## 更简洁而完整的例子

&emsp;&emsp;我们在数据中添加一段信息来构建一个更简洁但是上述功能更完整的例子, 这将是我们最后的代码块

```scala
abstract class Animal(
    val name: String
)

class Cat(name: String)
  extends Animal(name)

class Dog(name: String)
  extends Animal(name)

trait AnimalSwim[A]:
  def swim(animal: A): String

given catSwim: AnimalSwim[Cat] =
  cat => s"${cat.name} 怕水!"

given dogSwim: AnimalSwim[Dog] =
  dog => s"${dog.name} 在水中吼吼吼~"

def swim[A](animal: A)(using
    animalSwim: AnimalSwim[A]
): String =
  animalSwim.swim(animal)

@main
def run(args: String*) =
  val kitty = new Cat("Kitty")
  val papi = new Dog("papi")

  println(swim(kitty))
  println(swim(papi))
```

&emsp;&emsp;你也可以点击 [scasite](https://scastie.scala-lang.org/SJIT4oj8TYGvMok3415Evw) 在浏览器中试试上面这段代码, 做一些自己的修改, 点击 **Run** 来看看结果

&emsp;&emsp;看看这些代码, 想一想, 现在如果你需要增加一种操作和一种数据, 需要做出多少修改?

## 一些我自己的感想

&emsp;&emsp;设计程序实际上也是一种艺术, 没有绝对的好设计, 我们能追求的只是将程序写的 **足够好**, 而随着我们学习和经验越多, "足够好" 也将越来越好

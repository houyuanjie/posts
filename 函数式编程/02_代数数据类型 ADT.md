# 代数数据类型 ADT

在上一篇 [Typeclass 编程模式](./01_Typeclass%20%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%BC%8F.md) 中, 我们认识到有必要将操作与数据分离, 提取到 typeclass 中, 而 ADT (Algebraic Datatype) 则单纯地用于承载数据; 你可能觉得这类似于 POJO 或者结构体, 继续看下去, 你将会得到一个看待数据类型的全新视角

## 常见的数据类型

编程语言们大多都提供了一套原生支持的数据类型

类型    |常见用途       
-------|--------------
string |地址, 名称等
number |个数, 大小等
boolean|状态, 权限等

我们很快发现就这几种类型不足以方便地描述复杂的现实实体, 一方面是这些类型太 "笼统", 不能反映数据表达的信息; 另一方面是这些类型是单一的, 不能反映数据的全貌; 于是我们总是需要把这些类型组合使用

## 常见的组合类型

首先是类型太笼统的问题, 举个例子吧: 交通信号灯只有三种颜色, 我们不需要所有的字符串用来表示交通信号灯的颜色, 于是我们引入了所谓的 "枚举"

```scala
enum LightColor(
  val value: String
):
  case Red
    extends LightColor("0xFF0000")
  case Yellow
    extends LightColor("0xFFFF00")
  case Green
    extends LightColor("0x00FF00")
```

这样, 我们让 String 类型值的个数 (可以想象, 世间存在有极大量的字符串) 大大减少到了固定的 3 个, 组合成一个新的类型, 我们使用这个新的更具体交通信号灯颜色类型也就不再需要关心那个更笼统的 String 类型

其次是类型单一的问题, 同样还是这个例子: 交通信号灯除了颜色还有当前倒数的秒数, 我们假设信号灯最多数 70 下, 只用 `LightColor` 显然不够, 于是我们引入了所谓的 "元组"

```scala
type LightNumber = 0 | 1 | ... | 69

// 元组
type Light = (LightColor, LightNumber)

// Scala 提供了一种模式类来打包数据
// 实际上就是一种起了名字的元组
case class Light(
  color: LightColor,
  number: LightNumber
)
```

显然, 元组或者模式类在一定意义上和 POJO 或者结构体类似, 它们都能同时承载多条数据

## 代数在哪里?

你可能会问: 这枚举和元组我早就认识了, 那数据类型的 "代数" 在哪里?

那么, 请我想请你思考一下这个问题: 你能数出多少个 `LightColor` 类型的值?

显然:

```scala
count[LightColor]
= count[Red] 
  + count[Yellow]
  + count[Green]
= 1 + 1 + 1
= 3
```

好, 再想想有多少个 `Light` 类型的值? 

这显然是个组合问题:

```scala
count[Light]
= count[LightColor]
  * count[LightNumber]
= 3 * 70
= 210
```

于是我们发现了一个惊人的规律: 枚举是 "和类型", 元组是 "积类型", 代数就在这里

## 类型类派生

上面的过程只是帮助进行理解, "ADT" 在学术上有着严格的定义和推导过程, 我们无需关心那些纸片, 写好自己的程序即可

类比想想, 我们就会发现除了枚举, 那些 `sealed trait`, `sealed class`, `sealed interface` 都是和类型, 而那些 `case class`, `record`, `data class` 都是积类型

上一篇 [Typeclass 编程模式](./01_Typeclass%20%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%BC%8F.md) 中的猫狗动物也是代数数据类型

```scala
sealed trait Animal:
  def name: String

case class Cat(
  override
  val name: String,
  val color: String
) extends Animal

case class Dog(
  override
  val name: String,
  val age: Int
) extends Animal
```

现在, 假设我想引入一个新的格式化操作 `stringify`, 它能通用地将数据格式化成能打印到控制台的字符串

```scala
trait Stringify[T]:
  def stringify(value: T): String
```

通常在实现时, 我们会先为基本类型实现

```scala
given Stringify[String] = 
  s => s

given Stringify[Int] =
  i => i.toString()
```

再借助基本类型实现组合类型, 这些组合类型因为没有什么特殊性, 我们希望能用工具自动实现, shapeless 就是一个能帮助我们自动为代数数据类型生成类型类实现的工具

下面注释中的 shapeless 代码仅作为展示, 你无需知道 shapeless 要怎么使用

```scala
import shapeless3.deriving.*

given genProduct[T](using
    inst: K0.ProductInstances[Stringify, T],
    labelling: Labelling[T]
): Stringify[T] with
  def stringify(value: T): String = ???
  // labelling.elemLabels.zipWithIndex
  //   .map { (label, index) =>
  //     inst.project(value)(index) {
  //       [A] =>
  //       (sa: Stringify[A], a: A) =>
  //       s"${label}=${sa.stringify(a)}"
  //     }
  //   }
  //   .mkString(s"${labelling.label}(", ", ", ")")

given genSum[T](using
    inst: K0.CoproductInstances[Stringify, T]
): Stringify[T] with
  def stringify(value: T): String = ???
  // inst.fold[String](value) {
  //   [A] =>
  //   (sa: Stringify[A], a: A) =>
  //   sa.stringify(a)
  // }

inline def autoGen[T](using
    gen: K0.Generic[T]
): Stringify[T] =
  gen.derive(
    genProduct,
    genSum
  )
```

有了自动生成类型类, 我们程序员就能从那种无聊的/重复的 **@Override** 中解脱出来, 真正有时间写一些有价值的 (业务的) 代码


```scala
val animalStringify =
  autoGen[Animal]

@main
def run(): Unit =
  val kitty =
    new Cat("Kitty", "black")
  val papi =
    new Dog("Papi", 5)

  println(
    animalStringify
      .stringify(kitty)
  )
  // Cat(name=Kitty, color=black)
  println(
    animalStringify
      .stringify(papi)
  )
  // Dog(name=Papi, age=5)
```

那现在我来告诉你上面这个例子中, 代数是在什么时候起的作用

shapeless 的原理大致是这样的: 当我们使用 `autoGen[Animal]` 时, shapeless 就能通过一些编译器提供的信息知道 `Animal` 是一个 `sealed trait`, 属于 "和类型", 需要在所有的 `autoGen[Cat]`, `autoGen[Dog]`, `autoGen[Turtle]` 中进行搜索, 这实际上是一个 **树**; 但当具体确定到 `autoGen[Dog]` 后, `Dog` 是一个 `case class`, 属于 "积类型", `Dog` 的类型实际上是一个类似 `(String, (Int, ^))` 的 **列表**, 只要依次取头结点就可以; 尽管现实中的 ADT 可以很复杂, 嵌套定义多层, 但回想到其无外乎"和""积"两种, 我们就能在其上派生任何想要的操作

## 一些我自己的想法

到现在为止, 我们有了强大的 typeclass, 又有了强大的 adt, 还能在 adt 上派生 typeclass, 我们似乎已经无所不能, 但请先别激动, 我们的函数式旅程才刚刚开始, 而这两个只是函数式思想中作为基石的存在

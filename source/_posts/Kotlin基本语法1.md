---
title: Kotlin基本语法1
date: 2016-04-17 21:37:06
tags:
	- kotlin
categories:
	- 编程语言
---
Kotlin是一门与Swift类似的静态类型JVM语言，由JetBrains设计开发并开源，号称“Android世界的Swift”。最近花时间看了下这门语言，整理了一些和Java区别比较大的方面，仅作备忘（这里仅是一部分）。

## 控制流
### if表达式
在Kotlin 中，if是一个带有返回值的表达式（也可以是一般性语句）。因此，在Kotlin中没有三元操作符(condition ? then : else)，因为`if表达式`就能胜任这项工作。
``` kotlin
//定义变量
val a = 1
val b = 2

//if作为表达式，此时必须有else分支
val max = if (a > b) a else b

//if作为表达式，此时必须有else分支（此时分支是代码块，且代码块的最后一个表达式的值是当前代码块的值）
val min = if (a > b) { 
    print("Choose a") 
    b 
  } else { 
    print("Choose b") 
    a 
  } 
```
作为表达式时，一般是将`if表达式`（的值）赋值为变量，或者在函数中return`if表达式`（的值）。在这种情况下，`if表达式`必须包含else分支。

除了作为表达式之外，`if`也可以实现一般性语句，此时则不必包含else分支。
例如：
``` kotlin
fun max(a: Int, b: Int): Int {
    if (a > b) {
        println("a")
        return a
    }
    return b
}
```

<!-- more -->

### When表达式
在Kotlin中，`when`是switch的替代品。`when`既可以作为表达式，也可以是一般性语句。
作为`when`表达式时，else分支是不可缺少的（除非编译器检测到前面的条件分支覆盖了所有的情况）。同时，最终匹配的条件分支的值就是整个`when表达式`的值。（和if表达式一样，when表达式中的分支也可以是代码块，代码块的值则是最后一条表达式的值）。
作为一般性语句时，则忽略每个分支的值。

`when表达式`中比较多变是分支条件（branch condition），可以是普通字面量、常量、表达式、范围判断以及类型判断等。例如:
``` kotlin
val x = 4
when (x) {
    //字面量
    3 -> println("常量")
    //表达式
    if (x > 0) 2 else -1 -> println("表达式")
    //范围匹配
    !in 1..10 -> println("范围匹配")
    //类型判断
    is Int -> println("类型判断")
    else -> println("else")
    }
```
除此之外，`when`能够取代`if-else if`链。例如：
``` kotlin 
when {
  a > b -> print("a > b")
  a < b -> print("a < b")
  else -> print("a == b")
}
```
这种情况下，没有when参数，若分支条件为true，则匹配成功。
### for循环
for操作符可以对任何提供迭代器的对象进行遍历（这点和Java类似）。例如：

``` kotlin
for (item in collection)
  println(item)
```
但是对数组的遍历，会被编译器优化成基于索引值来遍历，而不会创建迭代器对象。
当然，我们也可以显示的通过索引值来遍历一个数组，例如：
``` kotlin
for (i in array.indices)
  println(array[i])
```
另外，针对数组，还有更方便的遍历方式，例如：
``` kotlin
val array = arrayOf("leon", "lt", "zh")
for ((index, value) in array.withIndex()) {
    println("the element at $index is $value")
}
```
### while循环
`while`和 `do..while`的使用方式和Java中相同，例如：
``` kotlin
do {
    var a = 0;
    println("a = $a")
} while (a < 0)//注意，这里变量a可以被访问，这点和java当中不同。
```
## Null安全性
在Java中，出现频率最高的运行时异常应该就是`NullPointerException`。Kotlin致力于消除空指针异常，Kotlin类型系统将引用（reference）分为可空和不可空。

默认情况下，变量是不可空类型，例如：
``` kotlin
var a: String = "abc"
a = null // compilation error
```
但是，我们可以通过在类型后面添加`?`，来明确声明一个可空的变量。
``` kotlin
//此时变量的类型为String?，和String是不同，不可直接赋值
var b: String? = "abc" 
b = null // ok
//因为变量b可能为null，所以下面的语句是编译不过的
val l = b.length //error: variable 'b' can be null
```
针对可空类型的变量，有两种方式来访问他们的属性和方法。
1. 明确地检测变量是否为Null
``` kotlin
val str: String? = "abc"
println(str.length) //编译不通过，str可能为Null
if (str != null) {
    //编译器检测到进行了非空检测，所以允许直接访问字符串的legth属性
    println(str.length)
}
println(str.length)//编译不通过，str可能为Null
```
2. `?.`安全的调用
``` kotlin
val str: String? = "abc"
//如果str不为Null，就会返回字符串的长度，否则返回null。变量len的类型是Int?
val len = str?.length
```
### 针对可空变量的特殊操作符
#### `?:`操作符
``` kotlin
val str: String? = "abc"
//str不为Null，则返回字符串长度，否则返回-1
val len = str?.length ?: -1
```
假如`?:`操作符左边的表达式是非空的，那么`?:`操作符就返回左边表达式的值, 否则就返回右边的内容。并且，仅在左侧值为Null时，右侧表达式才会进行计算。

因为`throw`和`return`也是表达式，所以都可以使用在`?:`操作符的右侧，这在函数头部检查参数的合法性时，很有用。
``` kotlin
fun foo(node: Node): String? {
  //若node.getParent()为Null，那么函数直接返回Null
  val parent = node.getParent() ?: return null
  //若node.getName()为Null，那么函数直接抛出异常
  val name = node.getName() ?: throw IllegalArgumentException("name expected")
  // ...
}
```

#### `!!`操作符 
如果你想要强制访问在一个可空变量的属性，那可以通过`!!`操作符来操作。但是这可能导致空指针异常，请谨慎使用！
```
val str: String? = "abc"
//str不为Null，则返回字符串长度，否则抛出KotlinNullPointerException异常
val len = str!!.length
```
## 字符串
和Java中一样，kotlin中的字符串也是不可变的，但是我们可以直接通过索引运算符`[]`来访问字符串中的字符。

在kotlin中，支持两种类型的字符串字面值：

1. `转义字符串`
`转义字符串`支持转义字符，类似于Java中的字符串
``` kotlin
val str = "Hello, world!\nLeon"
```
2. `原生字符串`
`原生字符串`使用3个双引号`"""`括起来，内部不支持转义字符`\`，但是可以包含多行和任意字符。
``` kotlin
val text = """
  for (c in "foo")
    print(c)
"""
```
### 字符串模板
字符串模板就是包含模板表达式的字符串。kotlin中的两种字符串都可以包含模板表达式，这里会计算出模板表达式的值，并插入在对应的字符串索引处（这对于在Java中拼接字符串来说简直是福音啊）。而所谓模板表达式就是以`$`开头的表达式，包含两种形式：

* `$` + 简单名称
``` kotlin
val i = 10
val s = "i = $i" // s为"i = 10"
```
* `$` + {表达式}
``` kotlin
val str = "abc"
//result为 "abc.length is 3"
val result = "$str.length is ${str.length}" 
```
此外，因为原生字符串不支持转义字符`\`，因此可以使用`${'$'}`在原生字符串中表示`$`。


## 函数
在kotlin中，定义函数使用`fun`关键词。函数的定义可以概括如下：
``` kotlin
fun 函数名（参数名 : 参数类型 = 默认参数 , ...） : 返回类型 { 代码块 }

or

fun 函数名（参数名 : 参数类型 = 默认参数 , ...） : [返回类型] = 单个表达式 
```
函数体可以是代码块，此时必须显示指定函数返回值，除非返回类型为`Unit`；也可以是单个表达式，此时可以省略函数返回值，kotlin可以推断出返回值。
函数参数可以指定默认值，对于这种参数，调用时可以省略该参数值。（很像C++语法）

根据函数的特点，可以分为以下几类：
* 顶级函数
定义在文件顶级作用域，对于顶级函数，不必创建对象就可以直接调用。

* 成员函数
定义在类内部，即成员函数。相当于Java中的类实例函数。

* 扩展函数
很有用的功能，可以对已有的类添加扩展函数。

* 局部函数
定义在函数中的函数，局部函数可以访问外部函数的局部变量，即闭包

* 高阶函数
可以把函数作为参数，或者把函数作为返回值的函数。

* 内联函数
 编译时内联函数会被内联到代码调用处，减少了函数调用的开销。

## 函数字面量 (Function Literal)
未被声明，直接作为表达式被传递的函数称为函数字面量，包括Lambda表达式和匿名函数。函数字面量经常作为参数传递给高阶函数使用。
### Lambda表达式
在Kotlin中，Lambda表达式的格式如下所示：
``` kotlin
{参数 : [参数类型] -> 代码块}
```
若高阶函数的最后一个参数是函数，那么在使用Lambda表达式作为参数值时，可以把Lambda表达式放在参数括号外面（若高阶函数只有一个函数参数，那参数括号就可以完全省略了）。
``` kotlin
//高阶函数1
fun lambda1(a: Int, hanshu: (Int) -> Int) {
    println(hanshu(a))
}
//调用高阶函数1
lambda1(1) {args -> args * 2}

//假如Lambda表达式只有一个参数，则可以使用默认的参数it
lambda1(1) {it * 2}

//高阶函数2
fun lambda2(hanshu: () -> Int) {
    println(hanshu())
}
//调用高阶函数2
lambda2 { 2 }
```
上面的`(Int) -> Int`表示函数类型，即一个接收Int类型参数，返回Int类型值的函数。
### 匿名函数
使用Lambda表达式作为高阶函数参数值存在一个缺点：无法指定函数返回值。虽然多数情况下，编译器可以推断出返回值。但是特殊情况下，若一定需要指定返回值，那么就可以使用匿名函数作为高阶函数参数值。

匿名函数和普通函数只有2点不同
1. 匿名函数没有函数名（废话）
2. 参数类型可以省略（在可以被推断出来的情况下）

当把匿名函数作为高阶函数的参数时，不能写在高阶函数参数括号外面（即：此规则只适用于Lambda表达式）。
因此，针对上面高阶函数的调用，可以写成这样：
``` kotlin
 lambda1(3, fun(a): Int {
        return a * 2
    })
```
除此之外，Lambda表达式和匿名函数在处理`return`语句时，也存在差异。
在Kotlin中，我们能够通过`return`语句，直接从命名函数或者匿名函数退出。因此，如果想从Lambda表达式中退出，必须使用`带标签的return`。例如：
``` kotlin 
fun innerFun(hanshu: () -> Int) {
    println("innerFun")
    println(hanshu())
}

fun foo(): Int {
    innerFun { return@innerFun 3 }
    println("foo fun")
    return 0
}

foo()

输出：
innerFun
3
foo fun
```
那么若我们想从当前函数退出那（即foo函数），理论上应该去掉标签就OK了，但是Kotlin却不允许这么做，这很容易导致误解。
除非调用Lambda表达式的函数是内联函数，即innerFun是内联函数，此时return语句会被内联到foo函数，那么return语句直接从foo函数退出，也就很好理解了。例如：
``` kotlin 
inline fun innerFun(hanshu: () -> Int) {
    println("innerFun")
    hanshu()
}

fun foo(): Int {
    innerFun { return  3 }
    println("foo fun")
    return 0
}

foo()

输出：
innerFun
```
### 闭包
闭包是由函数及其相关的引用环境组合而成的实体(即：闭包=函数+引用环境)。
在kotlin中，函数字面量（Lambda表达式和匿名函数）能够访问它的闭包。并且可以修改闭包中的变量，这点和java不同。例如：
``` kotlin
var count = 0
var ints = listOf(1, 2, 3)
ints.filter { it > 0 }.forEach {
    count += it
}
println("count = $count")

输出：
count = 6
```

### 指定接收者的函数字面量（Function Literals with Receiver）
指定接收者的函数字面量和类的扩展函数很像，在函数字面量函数体内，可以直接访问Receiver的方法。
``` kotlin
//参数sum的类型就是一种Function Literals with Receiver
fun hello(str: String, sum: String.(String) -> Unit) {
    str.sum("World")
}

//以Lambda表达式作为参数，此时Receiver是由编辑器推断出来的
hello("Hello") {
    //this表示接收者字符串，即"Hello",it表示参数，即"World"
		println("${this + it} length = ${this.length + it.length}")
}
    
//以匿名函数作为参数，此时Receiver是由匿名函数明确指定的   
hello("Hello", fun String.(it: String) {
    //this表示接收者字符串，即"Hello",it表示参数，即"World"
		println("${this + it} length = ${this.length + it.length}")
})

输出：
HelloWorld length = 10
```
上述`String.(String) -> Unit`就是指定接收者的函数字面量的类型，表示只有字符串类型对象才可以调用这个以字符串为参数且无返回值的函数。

---
##### 参考文章
1. [kotlin官网](http://kotlinlang.org/)
2. [kotlin官网中文版](https://github.com/cctanfujun/kotlin-web-site-cn)
3. [kotlin-for-android-developers-zh](https://github.com/wangjiegulu/kotlin-for-android-developers-zh/blob/master/SUMMARY.md)

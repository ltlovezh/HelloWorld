---
title: Kotlin基本语法2
date: 2016-04-27 09:47:36
tags:
	- kotlin
categories:
	- 编程语言
---
本文承接上文[Kotlin基本语法1](http://ltlovezh.com/2016/04/17/Kotlin%E5%9F%BA%E6%9C%AC%E8%AF%AD%E6%B3%951/)，继续介绍kotlin基础语法。

## 类
在kotlin中，类的定义包含类名、类头和类体。
``` kotlin
class 类名 修饰符[public,internal,protected,private] constructor（参数：参数类型）{
  init{
    initializer blocks
  }
  //二级构造函数，调用了主构造函数
  constructor(参数：参数类型) : this(参数) {
        parent.children.add(this)
  }
类体
}
其中，类头和类体都是可选的。
```
kotlin中的Class有两种构造函数：主构造函数和二级构造函数。上述`constructor（参数：参数类型）`就指定了主构造函数，它是类头的一部分。

主构造函数不能包含任何代码。任何初始化操作只能在`initializer blocks`中进行（可以直接引用主构造函数中的参数值）。

二级构造函数通过`constructor`关键字进行定义，所有的二级构造函数必须直接或者间接的调用主构造函数（通过this关键字）。

我们在java代码中，经常在类中定义多个属性变量，然后在构造函数中分别初始化它们。对于这种行为，kotlin提供了更加简单的方式：

``` kotlin
class User(val name: String = 默认参数值, var age: Int = 默认参数值) {
  // ...
}
```
上述User类定义了两个成员变量，并在主构造函数中完成了初始化。

<!-- more -->

## 类的组成

在kotlin中，一个Class可以包含以下几类成员：

1. Constructors and initializer blocks
构造函数和初始化代码块，上面已经介绍过了。

2. Properties
成员属性，下面会着重介绍。

3. Functions
成员函数，和类外部的顶级函数没啥区别，只是范围限制在Class内部了。

4. Nested and Inner Classes
在Kotlin中，内部类默认是static的，即不会持有对外部类的引用。但是被`inner`修饰的内部类是非static的，即会持有对外部类的引用。

5. **Object Declarations**
对象表达式和对象声明，下面会着重介绍。


### Properties （属性）
类属性就相当于java中的实例成员属性，它的完整定义如下所示：
``` kotlin
var <propertyName>: <PropertyType> [= <property_initializer>]
  [<getter>]
  [<setter>]
```
上述的`<getter>`和`<setter>`分别在读取和设置属性时被调用（编译器会默认生成）。`val`属性只有`<getter>`函数。
当然，我们可以定制属性的set和get函数，就像定义普通的函数一样。

``` kotlin
class Leontli() {
    var isEmpty: Boolean = false
        get() {
            return field
        }
        set(value) {
            field = true
        }
}
```
其中，filed称为`backing field`，由编译器编译生成，仅仅可在get()和set(value)方法中被访问（若只是重写了get和set方法，而没有主动访问field字段，那么编译器也不会主动生成该字段）。

#### 属性的延迟初始化
在kotlin中，一般情况下，类中的成员属性，必须直接初始化或者在构造函数中初始化。但是有时这是非常不方便的。比如：我们在Activity中经常需要获取各个View的引用，然后进行各种操作。在Java中一般是先定义View变量，然后在onCreate方法中对他们初始化，如下所示：
``` java
public class MainActivity extends Activity{
Bubtton mButton;
public void onCreate(Bundle savedInstanceState){
    super.OnCreate(savedInstanceState);
    setContentView(R.layout.main);
    mButton = (Button) findViewById(R.id.ok);
    aTextView.setText("Click Me");
    // ...
    }
}
```
但是在kotlin中，这样是行不通的，因为非空属性必须被初始化一个非空值。

``` kotlin
class MainActivity : Activity() {
var mButton : Button //error : Property must be initialized or be abstract
}
```
但是如果使用可Null属性，那么后续的操作也会很繁琐，同时也失去了kotlin提供的不可Null属性的便利，可能会导致NPE的频现(又回到了java的老路)。
``` kotlin
class MainActivity : Activity() {
var mButton : Button? = null
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.main)
    mButton = findViewById(R.id.ok) as Button
    //因为mButton可Null，所以只能通过下面的方式来赋值
    mButton!!.text = "Click Me"
    mButton?.text = "Click Me"
    }
}
```
明明知道mButton不会为Null，但是每次调用都要添加`!!` or `?`操作符，岂不是很不爽！

针对这种情况，kotlin也给出了可选的解决方案`Late-Initialized Properties`和`Delegated Properties`。每种方案各有其使用场景，下面一一介绍。
##### Late-Initialized Properties（延迟初始化属性）
通过`lateinit`关键字可以定义延迟初始化属性。例如：
``` kotlin
class MainActivity : Activity() {
lateinit var mButton : Button
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.main)
    mButton = findViewById(R.id.ok) as Button
    //可直接调用
    mButton.text = "Click Me"
    }
}
```
这样就可以满足我们的需求了（和java的写法很类似）。但是使用这种属性存在一些条件：
1. 只适用于可变属性，即`var`修饰的属性
2. 只适用于Class body内的属性，不能是在主构造函数中声明的属性。
3. 延迟初始化属性不能包含自定义的set和get方法
4. 属性不可为Null，且属性类型不能是原始类型。

毕竟存在这些限制，也就意味着在特殊情况下，无法使用延迟初始化属性。（强迫症...）

##### Delegated Properties （委托属性 or 代理属性）
`Delegated Properties`为属性的初始化提供了另外一种方案，即把属性委托给一个表达式。其语法如下所示：
``` kotlin
val/var <property name>: <Type> by <expression>
```
by后面的表达式称为属性代理（delegate）。针对这个delegate，存在一些限制规则：

1.若属性是不可变的（val修饰），那么必须提供如下形式的`getValue`函数:
``` kotlin
operator fun getValue(receiver: Any?, metadata: KProperty<*>): 被代理属性的类型 {
    //返回被代理属性的值
 }
```
其中，关键字`operator`是必不可少的。参数`receiver`表示持有被代理属性的类对象，其参数类型必须是被代理属性持有者的类型或者其父类型。`metadata`表示被代理属性的元数据，其参数类型必须是KProperty<*>或者其父类型。

2.若属性是可变的（var修饰），那么还必须提供`setValue`函数：
``` kotlin
operator fun setValue(receiver: Any?, metadata: KProperty<*>, value: 被代理属性的类型) {
    //设置被代理属性的值
 }
```
其中，receiver和metadata的含义和getValue方法类似，value则表示对被代理属性设置的新值。

这两个函数既可以是类成员函数，也可以是扩展函数。其中，对被代理属性的访问会被委托给delegate的`getValue`函数，对被代理属性的赋值会被委托给delegate的`setValue`函数。下面来看一个例子：
``` kotlin
//代理类
class Delegate<T>(arg: T) {
    var mSave: T = arg //存储被代理属性的值

    operator fun getValue(receiver: Any?, metadata: KProperty<*>): T {
        println("$receiver, thank you for delegating '${metadata.name}' to me!")
        return mSave
    }

    operator fun setValue(receiver: Any?, metadata: KProperty<*>, value: T) {
        println("$value has been assigned to '${metadata.name} in $receiver.'")
        mSave = value
    }
}

//被代理属性的持有者
class Example {
    //把属性p委托给Delegate类对象
    var p: String by Delegate<String>("init")
}

调用:
var example = Example();
println(example.p)
example.p = "Hello"
println(example.p)

输出:
Example@74f2794a, thank you for delegating 'p' to me!
init
Hello has been assigned to 'p in Example@74f2794a.'
Example@74f2794a, thank you for delegating 'p' to me!
Hello
```
从上述代码中可以看出，访问属性p时，调用了getValue方法，接收者就是Example对象，而通过元数据metadata，则可以访问被代理属性的名称等属性。而对属性p赋值时，调用了setValue方法，value则表示将要被赋值的新值。

上面代码中，我们用`mSave`属性存储了被代理对象的值，这是因为我们不能在getValue和setValue函数中访问被代理属性p，道理很简单，因为访问被代理属性又会触发getValue和setValue函数，导致死循环。

当然，上面仅仅是一个简单的例子，通过属性代理（delegate），我们可以实现任何对属性的取值和赋值操作。

针对委托属性，kotlin类库提供了一些现成的属性代理（delegate）。这里简单介绍下其中两种。
###### Lazy
`lazy`其实是一个接收函数字面量，且返回`Lazy`对象的高阶函数。`Lazy`对象是一个delegate，实现了对被代理属性的懒加载（`Lazy`对象的getValue函数就是通过扩展函数形式来提供的）。只有属性在第一次被访问时，才会执行函数字面量对属性进行赋值。且后续属性值不可变更，即懒加载属性必须是不可变的（val修饰）。看下例子：
``` kotlin
val lazyValue: String by lazy {
    println("computed!")
    "Hello"
}

调用:
println(lazyValue)
println(lazyValue)

输出:
computed!
Hello
Hello
```

###### Observable
`Delegates`是一个Standard property delegates。它提供了`observable`和`vetoable`函数，分别用来生成不同的属性代理。它们的函数签名如下所示：

``` kotlin
//接收一个初始值和一个函数（包含属性、旧值和新值共3个参数），返回一个ObservableProperty属性代理对象
public inline fun <T> observable(initialValue: T, crossinline onChange: (property: KProperty<*>, oldValue: T, newValue: T) -> Unit) : ReadWriteProperty<Any?, T> = object : ObservableProperty<T>(initialValue) {
        //重写了afterChange函数，在被代理属性被修改后，才会被调用。
        override fun afterChange(property: KProperty<*>, oldValue: T, newValue: T) = onChange(property, oldValue, newValue)
    }
    
//接收一个初始值和一个函数（包含属性、旧值和新值共3个参数），返回一个ObservableProperty属性代理对象
public inline fun <T> vetoable(initialValue: T, crossinline onChange: (property: KProperty<*>, oldValue: T, newValue: T) -> Boolean) : ReadWriteProperty<Any?, T> = object : ObservableProperty<T>(initialValue) {
        //重写了beforeChange函数，在被代理属性被修改前，会被调用，若onChange返回了false，则赋值会被终止掉。
        override fun beforeChange(property: KProperty<*>, oldValue: T, newValue: T): Boolean = onChange(property, oldValue, newValue)
    }
```
前者`observable`接收的参数函数会在赋值之后被调用，后者`vetoable`接收的参数函数会在赋值之前被调用，并且有权中断赋值。我们可以想一下，这种控制逻辑应该是在ObservableProperty属性代理的setValue方法中实现的。果不其然：
``` kotlin
public override fun setValue(thisRef: Any?, property: KProperty<*>, value: T) {
    val oldValue = this.value
    //真正赋值之前的调用，若beforeChange返回了false，则中断赋值
    if (!beforeChange(property, oldValue, value)) {
        return
    }
    this.value = value //真正赋值
    //赋值之后的调用，若beforeChange返回了false，则不会走到这里
    afterChange(property, oldValue, value)
}
```
但是总感觉有些遗憾，为什么没有函数既可以在赋值前被调用，又可以在赋值后被调用那？其实这也是可以理解的，若是`beforeChange`函数返回了false，那么`afterChange`便不会被调用；若是`beforeChange`函数返回了true，那么`afterChange`的参数和前者完全是一样的，就没啥意义了。

但是，出于练习目的，我们完全可以实现这种函数，基本思路就是为`Delegates`类提供扩展函数，返回同时实现了`beforeChange`和`afterChange`的ObservableProperty属性代理对象。
``` kotlin
//initialValue是初始值，onBefore在赋值前被调用，onAfter在赋值后被调用
inline fun  <T> Delegates.all(initialValue: T,
    crossinline onBefore: (property: KProperty<*>, oldValue: T, newValue: T) -> Boolean,
    crossinline onAfter: (property: KProperty<*>, oldValue: T, newValue: T) -> Unit):
        ReadWriteProperty<Any?, T> = object : ObservableProperty<T>(initialValue) {
    //重写了两个函数
    override fun beforeChange(property: KProperty<*>, oldValue: T, newValue: T): Boolean = onBefore(property, oldValue, newValue)
    override fun afterChange(property: KProperty<*>, oldValue: T, newValue: T) = onAfter(property, oldValue, newValue)
}
```
上面我们通过扩展函数`all`，综合了`observable`和`vetoable`的功能。传入的参数`onBefore`会在赋值前被调用，onAfter会在赋值后被调用（赋值不被中断的前提下）。下面看下具体使用:
``` kotlin
class Example {
    //isPass也是被代理属性，在使用前必须先赋值
    var isPass: Boolean by Delegates.notNull<Boolean>()
    var name: String by Delegates.all("<no name>", { prop, old, new ->
        println("before $old -> $new")
        isPass //闭包哈
    }) {
        prop, old, new ->
        println("after $old -> $new")
    }
}

调用:
var example = Example();
example.isPass = true
println(example.name)
example.name = "leon"
println(example.name)

example.isPass = false
example.name = "android"
println(example.name)

输出:
<no name>
before <no name> -> leon
after <no name> -> leon
leon
before leon -> android 
//这里赋值被阻断后，after没有执行哈
leon
```

### Object Expressions and Declarations (对象表达式和对象声明)
对象表达式和对象声明都是用来创建对象的。其中，前者用来创建匿名类对象，可以直接赋值给一个变量；后者用来创建单例对象，不可直接赋值给一个变量。

除此之外，两者还有一个重要的差异：Object Expressions在定义的地方被立即执行；而Object Declarations则会延迟到第一次被访问时才会执行。

#### Object Expressions
对象表达式的格式如下所示：
``` kotlin
object : 父类 , 接口1 , 接口2 , ...{
    //实现的接口方法
}
```
假如我们要为Button设置Click监听器：
``` kotlin
var mClick = object : View.OnClickListener{
override fun onClick(v: View?) {
    Toast.makeText(this@MainActivity, "Click Me", Toast.LENGTH_LONG).show()
    }
}
mButton.setOnClickListener(mClick)
```
当然还有更直接的方法（直接把Lambda表达式作为参数）：
``` kotlin
mButton.setOnClickListener{
Toast.makeText(this@MainActivity, "Click Me", Toast.LENGTH_LONG).show()
}
```
这种语法只有在参数是一个只有一个方法的类时才可以使用，例如Runnable类。

#### Object Declarations
对象声明的格式如下所示：
``` kotlin
object 单例对象名 ： 父类 , 接口1 , 接口2 , ...{
    //实现自定义方法和接口方法
}
```
后续就可以通过`单例对象名`来访问对象方法了。但是不可以在函数内进行对象声明。

可见对象表达式和对象声明的最明显区别就是：关键词`object`后面是不是有对象名。

#### Companion Objects
在Kotlin中，没有静态方法，而是推荐使用顶级函数来代替。但是我们可以通过`Companion Object`来实现类似静态方法的方法。

具体方式是在类内部的Object Declarations前面加上`Companion`关键字。这样我们就可以像使用类静态方法一样，直接通过类名调用`Companion Objects`内的方法了。看个例子：

``` kotlin
class OurClass {
    var a = 1
    companion object Factory {
        fun create(): OurClass { 
        //error,无法访问外部类的属性。从这里可以看出Factory更像是静态内部类，不过有待确认？
        println("a = $a") 
        return OurClass()
        }
    }
}

//调用
println(OurClass.create())
println(OurClass.Factory.create())
println(OurClass.Factory)
println(OurClass.Factory::class.java)

//输出
OurClass@3e99f610
OurClass@6de9b48b
OurClass$Factory@a4c4a0d
class example.OurClass$Factory
```
从上面的输出可以看出，create方法不是真正的static方法，而是静态内部类Factory的实例方法。因此，Factory也可以像普通的Object Declarations一样，实现接口。

## 类继承
在Kotlin中，所有的Kotlin类都有一个共同的父类`Any`（不是java.lang.Object）。
默认情况下，一个Kotlin类不可以被继承，成员函数也不可以被重写（这点和java不同，倒和C++很像）即默认都是`final`的。除非用关键词`open`明确指明。同时，在一个final Class内部，是不允许有open成员的。看一个官网的例子：
``` kotlin
open class Base {
  open fun v() {}
  fun nv() {}
}
class Derived() : Base() {
  //因为和父类v方法签名相同，所以必须明确指明是重写父类方法，否则编译不过
  override fun v() {}
  
  //因为和父类nv方法签名相同，所以要么修改方法签名，要么把父类nv方法声明为open，且这里明确指明是重写父类方法。
  //fun nv() {}
}
```

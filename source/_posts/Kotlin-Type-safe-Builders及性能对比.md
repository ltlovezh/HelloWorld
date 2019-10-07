---
title: 'Kotlin : Type-safe Builders及性能对比'
date: 2016-04-29 22:39:38
tags:
- kotlin
- Android
- Type-safe Builders
categories:
- 编程语言
---
本文承接上文[Kotlin基本语法2](http://ltlovezh.com/2016/04/27/Kotlin%E5%9F%BA%E6%9C%AC%E8%AF%AD%E6%B3%952/)，主要介绍扩展函数、Type-safe Builders以及使用Kotlin开发Android带来的安装包和方法数增量。
## Extensions
在Kotlin中，我们可以在不继承类的基础上，对某个类添加函数和属性，即扩展函数和扩展属性。
### 扩展函数
扩展函数基本格式：
``` kotlin
fun 被扩展类.扩展函数(函数参数...) : 返回类型{
//代码块内，可以访问被扩展类的成员属性和函数
}
```
我感觉扩展函数最大的特色就是可以访问被扩展类的成员属性和函数，这点太方便了。看个官网的例子：

``` kotlin
fun <T> MutableList<T>.swap(index1: Int, index2: Int) {
    // 'this' corresponds to the list
    val tmp = this[index1] 
    this[index1] = this[index2]
    this[index2] = tmp
}

调用:
var mList = mutableListOf(1,2,3,4)
mList.swap(1,2)
println(mList)

输出：
[1, 3, 2, 4]
```
关于扩展函数有几点需要说明：
1. 扩展函数是静态解析的，不是虚函数，即不能被重写。
2. 当类的成员函数和扩展函数有相同函数签名时，成员函数会优先被调用。

<!-- more -->

### 扩展属性
扩展属性基本格式：
``` kotlin
var 被扩展类.扩展属性 : 扩展属性类型
    set(value)
    get()
```
实际上，扩展属性并没有真正的在类中新增属性，所以扩展属性没有`field`字段，即不能为扩展属性设置初始值。一般情况下，只能重写`set`和`get`方法，为扩展属性提供操作。看个官网的例子：
``` kotlin
val <T> MutableList<T>.lastIndex: Int
  get() = size - 1
  
  
调用：  
var mList = mutableListOf(1, 2, 3, 4)
println(mList.lastIndex)

输出：
3
```
## Type-safe Builders
个人感觉`Type-safe Builders`是kotlin中的一大特色。它让我们以陈述式语言风格来创建对象。非常适合于创建多级嵌套的数据结构，例如：XML文件、布局文件等。其语法类似于gradle，都是基于DSL构建。

`Type-safe Builders`之所以能够成行，很大程度上依赖于扩展函数、扩展属性和`Lambda With Receiver`。关于Lambda With Receiver，可以参考上一篇文章。

下面看一个例子，假如我们要创建任意Android View，若通过java代码来写，那么每个View都是不同的，需要每个View单独来写。但是在Kotlin中，我们可以结合泛型、扩展函数和`Lambda With Receiver`，实现一个通用的函数。

``` kotlin
//对Context的扩展函数，创建任意View，并使用init函数对View进行初始化
inline fun <reified T : View> Context.createView(init: T.() -> Unit): T {
    val construct = T::class.java.getConstructor(Context::class.java)
    val mView = construct.newInstance(this)
    mView.init()
    return mView
}
//需要在Activity中调用,创建一个Button
var mButton = createView<Button> {
            //这里实际调用了Button.setText方法
            text = "Click Me" 
            //这里实际调用了Button.setTextSize方法
            textSize = 20f
        }
//需要在Activity中调用,创建一个TextView
var mTextView = createView<TextView> {
            text = "Hello World"
            textSize = 20f
        }
```
上面针对Context创建了一个扩展函数createView，参数类型是`T.() -> Unit`，即一种Lambda With Receiver，这里Receiver就是泛型T。接着在函数体内根据反射创建了具体的泛型对象（必须是View的子类），然后调用参数函数对泛型对象进行初始化。最后，使用该函数，分别创建了Button和TextView。

理论上，通过createView函数，我们可以创建任意View对象，并且可以针对不同对象指定不同的初始化操作。

实现createView函数有一个很重要的条件，那就是根据泛型类型T，创建对应的对象（在java里，打死也做不到）。这要求函数必须是`inline`的，且T必须是`reified`具体化的。这样，函数在调用时就会内联到函数调用处，此时T的类型实际上是确定的，因而Kotlin通过reified关键字告诉编译器，T可以当实际类型来使用。

OK，上面实现了第一步，可以创建任意View了。但是还无法创建层级关系。下面我们再来一个扩展函数，搞定层级关系：
``` kotlin
//对ViewGroup的扩展函数，创建任意子View，并使用init函数对子View进行初始化，同时把子View添加到Receiver表示的父View当中。
inline fun <reified T : View> ViewGroup.createView(init: T.() -> Unit): TV {
    val construct = T::class.java.getConstructor(Context::class.java)
    val mView = construct.newInstance(context)
    addView(mView)
    mView.init()
    return mView
}
```
上面对ViewGroup定义了扩展函数createView。在createView内部可以访问ViewGroup的成员函数和属性，这里通过addView把创建出来的子View添加到了父View中，实现了层级的关联。

假如我们要用kotlin实现下面的View布局：
``` xml
<LinearLayout
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:orientation="horizontal">
        <Button
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:text="first Button"
            android:textSize="20px" />
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:text="second TextView"
            android:textSize="20px" />
</LinearLayout>
```

利用之前提供的两个createView函数，来实现上述布局，如下所示：
``` kotlin
createView<LinearLayout> {
    orientation = LinearLayout.HORIZONTAL
    //添加第一个子View
    createView<Button> {
        with(layoutParams as LinearLayout.LayoutParams) {
            width = ViewGroup.LayoutParams.WRAP_CONTENT
            height = ViewGroup.LayoutParams.WRAP_CONTENT
            weight = 1f
            }
        textSize = 20f
        text = "first Button"
    }
        
    //添加第二个子View
    createView<TextView> {
        with(layoutParams as LinearLayout.LayoutParams) {
            width = ViewGroup.LayoutParams.WRAP_CONTENT
            height = ViewGroup.LayoutParams.WRAP_CONTENT
            weight = 1f
            }
        textSize = 20f
        text = "second TextView"
    }
}
```
最外层的createView函数是Context的扩展函数，他的函数参数是`LinearLayout.() -> Unit`，所以最外层大括号内的Receiver就是LinearLayout对象。
因此内层的createView函数是ViewGroup的扩展函数，这样内部创建的Button和TextView都添加到了LinearLayout中，实现了层级关联。

上面的代码已经很简洁了，但是还有提升的空间（这里一开始我也没有想到，是[参考的这篇文章](http://mp.weixin.qq.com/s?__biz=MzA3ODg4MDk0Ng==&mid=2651112113&idx=1&sn=e7e3cffe60150e7db20ebd4f648b4c15&scene=23&srcid=04279tiwca4UACLXtISl5ktd#rd)）。

上述构建方式主要缺点是还要指定泛型参数，下面我们就把泛型参数干掉。首先，需要根据View类型为Context和ViewGroup声明扩展函数，如下所示：
``` kotlin
//下面的三个函数调用的都是Context的扩展函数createView，因此创建出来的都是单个View。
fun Context.linearLayout(init: LinearLayout.() -> Unit) = createView(init)
fun Context.textView(init: TextView.() -> Unit) = createView(init)
fun Context.button(init: TextView.() -> Unit) = createView(init)

//下面的三个函数调用的都是ViewGroup的扩展函数createView，因此创建出来的View都会被添加到付ViewGroup中。
fun ViewGroup.linearLayout(init: LinearLayout.() -> Unit) = createView(init)
fun ViewGroup.textView(init: TextView.() -> Unit) = createView(init)
fun ViewGroup.button(init: TextView.() -> Unit) = createView(init)
```
因为上述扩展函数，不再有泛型，而且都是单个函数参数，因此新的构建方式可以简化如下：
``` kotlin
//调用Context.linearLayout函数
linearLayout {
    orientation = LinearLayout.HORIZONTAL
    //调用ViewGroup.button函数
    button {
        with(layoutParams as LinearLayout.LayoutParams) {
            weight = 1f
            }
        textSize = 20f
        text = "first Button"
    }
    //调用ViewGroup.textView函数
    textView {
        with(layoutParams as LinearLayout.LayoutParams) {
            weight = 1f
            }
        textSize = 20f
        text = "second TextView"
    }
}
```
怎样，上述方式是不是很像Gradle配置文件，都是基于DSL的，差不太多。

这种基于`Type-safe Builders`构建View层级的方式相对于Java Code来说确实简洁不少。
但是在Android中，我们一般通过XML文件来进行View布局，这种方式非常直观。但使用XML布局文件也存在着性能问题，因为系统需要先解析XML文件，再去构建View对象。

下面我们简单对比下通过`Type-safe Builders`形式代码和XML文件实现相同布局的耗时。这里的对比方案是使用Github上的项目[kotlin-view-builder](https://github.com/CodingDoug/kotlin-view-builder)，详细过程可以看下代码。

简单来说，一种是通过上面介绍的`Type-safe Builders`方式构建View对象,另一种是通过`LayoutInflater`来加载布局文件。最终要实现的布局如下所示：
``` XML
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="horizontal"
    android:paddingTop="8dp"
    android:paddingBottom="8dp"
    android:paddingLeft="16dp"
    android:paddingRight="16dp">
    
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginRight="16dp"
        android:layout_marginEnd="16dp"
        android:layout_gravity="center_vertical"
        android:textSize="24sp"
        android:text="@string/time"
        />

    <LinearLayout
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:layout_gravity="center_vertical"
        android:orientation="vertical"
        >

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@string/day"
            />

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@string/location"
            />
    </LinearLayout>

</LinearLayout>
```
耗时结果如下表所示（单位：ms，取5次平均值，测试手机为小米4 Android6.0系统）：

| 创建View对象的次数| Type-safe Builders | XML | XML / Builders |
| :--------: | :-----: | :----: | :----: |
| 1 | 2 | 4 | 2.0 |
| 10 | 20 | 35 | 1.75 |
| 100 | 189 | 284 | 1.50 |
| 500 | 968 | 1402 | 1.44 |
| 1000 | 1945 | 2681 | 1.38 |

由上表可知：通过代码构建View对象总体上还是要比XML快一些。但是随着View对象增多，kotlin的优势逐渐变小（这点还没有想明白）。
除了上述性能对比，其实这两种方式各有其使用场景，具体可以参考
[实战kotlin@android（三）：扩展变量与其它技巧](http://mp.weixin.qq.com/s?__biz=MzA3ODg4MDk0Ng==&mid=2651112113&idx=1&sn=e7e3cffe60150e7db20ebd4f648b4c15&scene=23&srcid=04279tiwca4UACLXtISl5ktd#rd)



## 方法数和安装包
在Android开发中，方法数和安装包是永恒的话题。因此我们看下添加kotlin后的安装包和方法数增量。

| 项目 | 安装包 | 方法数 |
| :--------: | :-----: | :----: |
| 新建的Android项目 | 1.2M | 16259 |
| 新增Stdlib和Runtime类库 | 1.5M | 23177 |
| 新增Reflect类库 | 2.1M | 34855 |
| 新增Anko类库 | 2.2M | 36971 |

Stdlib和Runtime是Kotlin必不可少的基础类库，方法数大概是7000个，还可以接受。
相对来说，Reflect类库的安装包和方法数增量都比较大，但是该类库不是必须的，可以选择使用(主要是为了处理kotlin反射和java反射之间的兼容性)。最后一个Anko是为了方便构建动态View层级，可视情况决定是否采用。

---

## 参考文章
1. [实战Kotlin@Android（一）：项目配置和语言转换](http://mp.weixin.qq.com/s?__biz=MzA3ODg4MDk0Ng==&mid=403421478&idx=1&sn=ffaa512eaf23982c2da5e919694fd1ed&scene=21#wechat_redirect)
2. [实战Kotlin@Andorid（二）：界面构建与扩展方法](http://mp.weixin.qq.com/s?__biz=MzA3ODg4MDk0Ng==&mid=403568637&idx=1&sn=e6edfa6ef2029e0f6c1892ed7d2611d2&scene=21#wechat_redirect)
3. [实战kotlin@android（三）：扩展变量与其它技巧](http://mp.weixin.qq.com/s?__biz=MzA3ODg4MDk0Ng==&mid=2651112113&idx=1&sn=e7e3cffe60150e7db20ebd4f648b4c15&scene=23&srcid=04279tiwca4UACLXtISl5ktd#rd)

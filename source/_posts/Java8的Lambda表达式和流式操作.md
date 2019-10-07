---
title: Java8的Lambda表达式和流式操作
date: 2016-05-14 17:16:41
tags:
- Java8
- Lambda
categories:
- 编程语言
---
Java8在2014年就已经发布了，但是直到Android N宣布支持Java8，我才开始关注Java8的一些新特性。本篇主要介绍Java8中的Lambda表达式相关的特性。

## Lambda和函数接口
不像`Groovy`、`Scala`和`Kotlin`等语言，一开始就支持Lambda表达式。在Java8之前，Java一直是使用匿名内部类实现类似功能。所以如果要在Java中实现Lambda，需要借助于函数式接口（Functional Interface)。
所谓函数接口，就是只包含一个方法的接口（排除默认方法和静态方法） 。一般情况下，函数接口使用Java8新提供的注解`@FunctionalInterface`进行修饰，防止开发者往函数接口中添加更多的方法。

<!-- more -->

Lambda表达式让我们能够将函数作为方法参数，或者将代码作为数据对待。能够简化冗余代码，增加可读性。在Java8中，Lambda表达式主要由参数列表、分隔符`->`和方法体三部分构成，其格式如下所示：
``` java
(parameters) -> expression

(parameters) -> { statements; }
```
其中，前者表示方法体为表达式，即该表达式的值就是Lambda的返回值；后者表示方法体是代码块，需要通过return关键字明确指定返回值（返回值为void的除外）。
根据以往经验，`Runnable`是最明显的函数接口。下面我们分别通过匿名内部类和Lambda来创建一个线程，如下所示：
``` java
//匿名内部类
new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("匿名内部类");
            }
        }).start();

//Lambda表达式
new Thread(() -> System.out.println("Lambda")).start();
```
因为`Runnable.run`方法没有参数，所以参数列表为`()`。可见，其实Lambda表达式就是以简化方式实现了函数式接口中的唯一方法。

因为只有当方法参数是函数式接口时，我们才能使用Lambda。所以Java8在java.util.function包中为我们增加了一些通用的函数式接口。这里仅列出常用的几个：

* `Function`函数式接口提供了apply方法，接收T类型参数，返回R类型值,如下所示：
``` java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);

    //默认方法
    ...
}
```
* `Predicate`函数式接口提供了test方法，接收T类型参数，返回布尔值，通常在过滤数据的时候使用，如下所示：
``` java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
    
    //默认方法
    ...
}
```
* `Supplier`函数式接口提供了get方法，无参数，返回T类型值，通常在创建类实例的时候使用，如下所示：
``` java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```
* `Consumer`函数式接口，接收T类型参数，无返回值，如下所示
``` java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
    
    //默认方法
    ...
}
```
## 流式操作（Stream）
Java8在java.util.stream包中，为集合引入了流式操作（Stream）。该操作结合Lambda，可以实现对集合（Collection）的并行处理和函数式操作。

这里要明确Stream和Collection集合的区别：Collection是一种静态数据结构，存储在内存中；而Stream仅仅代表着数据流，没有数据结构，是面向CPU计算的。

### 函数式操作
根据流式操作的返回值，可以将流式操作划分为`中间操作`和`最终操作`。中间操作返回流本身，这样就可以将多个操作依次串联起来。而最终操作则返回结果值。而根据流的并发性，又可以将流分为串行和并行两种。下面分别介绍几种流式操作。
#### 中间操作
中间操作返回流本身，可以促成链式代码结构，几种常见中间操作的定义如下所示：

* 过滤
``` java
//过滤操作，根据函数的返回值，决定是否包含集合元素
Stream<T> filter(Predicate<? super T> predicate);
```

* 排序
``` java
//根据自然顺序对集合元素进行排序
Stream<T> sorted();
//根据Comparator定义的排序规则进行排序
Stream<T> sorted(Comparator<? super T> comparator);
```

* 元素映射
``` java
//把T类型输入值，映射到R类型返回值
<R> Stream<R> map(Function<? super T, ? extends R> mapper);
```
可见，这些中间操作的方法参数都是我们上面介绍的函数式接口，所以我们可以用Lambda表达式作为实参进行调用。最后，我们利用上面的流式操作对集合元素进行处理：
``` java
List<Integer> list = Arrays.asList(6, 5, 4, 3, 2, 1, 4, 4);

list.stream()
    .distinct() //去重
    .sorted() //默认排序
    .filter((it) -> it % 2 == 0) //找出偶数
    .map((it) -> it * 2) //把元素都乘以2
    .forEach((it) -> System.out.println(it)); //遍历最终元素
    
输出：
4
8
12
```
#### 最终操作
最终操作是流式操作的最后一步，基本上就是提取上述操作的结果。

* 对结果遍历
``` java
//通过action接口实例，对元素进行遍历
void forEach(Consumer<? super T> action);
```
* 把结果导出到数组
``` java
Object[] toArray();
```
* 获取元素个数
``` java
long count();
```
最终操作都比较简单，就不再举例了，可参见上面的示例。

### 串行和并行流
串行流是在一个线程上对元素逐个遍历，可以通过stream.sequential()方法获取，而并行流则是在多个线程上同时执行，可以充分发挥多核CPU的优势，可以通过stream.parallel()方法获取。

并行流的工作原理应当是把数据分割为几部分，然后交给多个线程去处理，最后再合并成最终的结果。类似于hadoop的MapReduce思想。

下面通过示例程序，测试下两者的性能：
``` java
List<String> list = new ArrayList<String>();
    for (int i = 0; i < 1000000; i++) {
        double d = Math.random() * 1000;
        list.add(d + "");
    }
long start = System.nanoTime();
//并行流
((Stream) list.stream().parallel()).sorted().count();
//串行流
//((Stream) list.stream().sequential()).sorted().count();
long end = System.nanoTime();
long ms = TimeUnit.NANOSECONDS.toMillis(end - start);
System.out.println(ms + "ms");
```
上述代码分别对1000000个随机数进行排序，其中串行情况下大概需要1000ms，而并行情况下大概需要700ms。


## Java8和Kotlin的Lambda表达式在语法上的区别
Kotlin中的Lambda表达式可参考[这里](http://ltlovezh.com/2016/04/17/Kotlin%E5%9F%BA%E6%9C%AC%E8%AF%AD%E6%B3%951/)。

Java8中的Lambda没有大括号包裹，且不管有没有参数，参数列表都不能省略。
Kotlin中的Lambda外层有大括号包裹，在没有参数的情况下，可以省略参数列表，在只有一个参数的情况下，也可以省略参数列表，此时使用默认参数`it`表示该唯一参数。
在Kotlin中，Lambda表达式作为函数的最后一个参数时，可以写在参数列表外面，这点Java8并不支持。

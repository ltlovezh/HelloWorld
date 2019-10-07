---
title: Activity启动模式一
date: 2016-08-28 21:48:19
tags:
 - Android
 - Activity启动模式
categories:
 - Android
---
众所周知，Activity有4种启动模式，分别是：Standard、SingleTop、SingleTask和SingleInstance，它们控制了被启动Activity的启动行为。本文将通过具体案例，详细分析这几种模式的差异和使用场景，方便日后查阅。

<!-- more -->
在展开具体分析之前，我们首先要了解下两个基础知识：Activity任务栈和`android:taskAffinity`属性。
## 基础知识
### Activity任务栈（Task）
Activity任务栈（Task）是一个标准的栈结构，具有“First In Last Out”的特性，用于在ActivityManagerService侧管理所有的Activity（AMS通过TaskRecord标识一个任务栈，通过ActivityRecord标识一个Activity）。

每当我们打开一个Activity时，就会有一个Activity组件被添加到任务栈，每当我们通过“back”键退出一个Activity时，就会有一个Activity组件从任务栈出栈。任意时刻，只有位于栈顶的Activity才可以跟用户进行交互。

同一时刻，Android系统可以有多个任务栈；每个任务栈可能有一个或多个Activity，这些Activity可能来自于同一个应用程序，也可能来自于多个应用程序。另外，同一个Activity可能只有一个实例，也可能有多个实例，而且这些实例既可能位于同一个任务栈，也可能位于不同的任务栈。**而这些行为都可以通过Activity启动模式进行控制。**

在Android系统的多个任务栈中，只有一个处于前台，即前台任务栈，其它的都位于后台，即后台任务栈。后台任务栈中的Activity处于暂停状态，用户可以通过唤起后台任务栈中的任意Activity，将后台任务栈切换到前台。

### android:taskAffinity属性
`android:taskAffinity`是Activity的一个属性，表示该**Activity期望的任务栈的名称**。默认情况下，一个应用程序中所有Activity的taskAffinity都是相同的，即应用程序的包名。当然，我们可以在配置文件中为每个Activity指定不同的taskAffinity（只有和已有包名不同，才有意义）。一般情况下，该属性主要和SingleTask启动模式或者`android:allowTaskReparenting`属性结合使用（下面会详细介绍），在其他情况下没有意义。

## 四种启动模式
上面简要介绍了Activity任务栈和android:taskAffinity属性。有了这些基础之后，我们就可以详细介绍Activity的四种启动模式了。

一般情况下，我们可以在`AndroidManifest`配置文件中，为Activity指定启动模式和taskAffinity，如下所示：

``` XML
<activity 
android:name="leon.com.activitylaunchmode.FirstActivity" 
android:launchMode="singleTop"
android:taskAffinity="leon.com.activitylaunchmode1"/>
```
除此之外，我们也可以通过设置Intent的某些标志位来达到相同的效果，关于这些和Activity启动模式相关的标志位，我们会在下篇文章进行介绍。

### `Standard`
标准模式，也是系统的默认模式。该模式下，每次启动Activity，都会创建一个新实例，并且将其加入到启动该Activity的那个Activity所在的任务栈中，所以目标Activity的多个实例可以位于不同的任务栈。例如：ActivityA启动了标准模式的ActivityB，那么ActivityB就会在ActivityA所在的任务栈中。

关于标准模式的Activity，有一个很经典的异常：
当我们通过非Activity的Context（例如：Service）启动标准模式的Activity时，就会有以下异常：
``` java
Caused by: Android.util.AndroidRuntimeException: Calling startActivity() from outside of an Activity context requires the FLAG_ACTIVITY_NEW_TASK flag. Is this really what you want?
```
这是因为标准模式的Activity会进入启动它的Activity的任务栈中，但是非Activity的Context又没有任务栈，所以就出错了。解决方案在异常信息中已经给出了：在Intent中添加`FLAG_ACTIVITY_NEW_TASK`标志位，这样就会为目标Activity创建新的任务栈。

OK，下面我们通过示例验证下该启动模式，具体步骤如下所示：

1. 创建MainActivity，并且在MainActivity的onCreate方法中，打印出当前Activity所在的任务栈ID。
2. 设置MainActivity的启动模式为Standard。
3. 然后通过MainActivity不断启动MainActivity本身。

经过上述步骤，最终的任务栈如下所示（adb shell dumpsys activity activities）：
![Standard模式](http://7xs2qy.com1.z0.glb.clouddn.com/Standard%E6%A8%A1%E5%BC%8F.png)

通过MainActivity打印出的日志如下所示：
![Standard模式日志](http://7xs2qy.com1.z0.glb.clouddn.com/Standard%E6%A8%A1%E5%BC%8F%E6%97%A5%E5%BF%97.png)

通过上面的任务栈和日志可知：每次启动MainActivity，都创建了新的实例，且每个实例都在一个任务栈中（任务栈ID都是8191）,最终一个任务栈中累积了多个MainActivity的实例。这样每次回退，都会看到相同的MainActivity。

### `SingleTop`
栈顶复用模式。该模式下，若目标Activity的实例已经存在，但是没有位于栈顶，那么仍然会创建新的实例，并添加到任务栈；若目标Activity的实例已经存在，且位于栈顶，那么就不会创建新的实例，而是复用已有实例，并依次调用目标Activity的`onPause -> onNewIntent -> onResume`方法。

OK，下面通过两个案例详细分析下SingleTop模式：

**案例1**，具体步骤如下所示：

1. 创建MainActivity，并且在onCreate方法中打印出当前Activity所在的任务栈ID，在onNewIntent方法中打印出"onNewIntent"字符串。
2. 设置MainActivity的启动模式为SingleTop。
3. 通过MainActivity不断启动MainActivity本身。

经过上述步骤，最终的任务栈如下所示（adb shell dumpsys activity activities）：
![SingleTop1](http://7xs2qy.com1.z0.glb.clouddn.com/SingleTop1.png)

通过MainActivity打印出的日志如下所示：
![SingleTop1日志](http://7xs2qy.com1.z0.glb.clouddn.com/SingleTop1%E6%97%A5%E5%BF%97.png)

通过上面的任务栈和日志可知：尽管多次启动了MainActivity，但只在第一次的时候创建了实例，后续的多次启动，仅仅调用了已有MainActivity实例的`onNewIntent`方法。最终任务栈（TaskId为2134）内只有一个MainActivity实例。

此外，还有一点需要注意：
多次启动处于栈顶的SingleTop模式的Activity时，其回调函数的顺序是`onPause -> onNewIntent -> onResume`。

**案例2**，具体步骤如下所示：

1. 创建MainActivity，并且在onCreate方法中打印出当前Activity所在的任务栈ID，在onNewIntent方法中打印出"onNewIntent"字符串。
2. 创建FirstActivity，并且在onCreate方法中打印出当前Activity所在的任务栈ID，在onNewIntent方法中打印出"onNewIntent"字符串。
3. 设置MainActivity的启动模式为SingleTop，设置FirstActivity的启动模式为Standard。
4. 首先启动MainActivity，接着通过MainActivity启动FirstActivity，然后再通过FirstActivity启动MainActivity，如此反复。

经过上述步骤，最终的任务栈如下所示（adb shell dumpsys activity activities）：
![SingleTop2](http://7xs2qy.com1.z0.glb.clouddn.com/SingleTop2.png)

通过MainActivity打印出的日志如下所示：
![SingleTop2-MainActivity](http://7xs2qy.com1.z0.glb.clouddn.com/SingleTop2-MainActivity.png)

通过FirstActivity打印出的日志如下所示：
![SingleTop2-FirstActivity](http://7xs2qy.com1.z0.glb.clouddn.com/SingleTop2-FirstActivity.png)

通过上面的任务栈和日志可知：尽管MainActivity的启动模式为`SingleTop`，但是当它不位于栈顶时，仍然会创建新的实例。最终任务栈中会出现多个MainActivity和FirstActivity，且自始至终，都只有一个任务栈（TaskId为2068）。

### `SingleTask`
**栈内复用模式**。该模式是对SingleTop的进一步加强，若Activity实例已经存在，则不管是不是在栈顶，都不会创建新的实例，而是复用已有Activity实例，即清除任务栈中目标Activity之上的所有Activity，使其位于栈顶，同时也会调用其`onNewIntent`方法；若Activity实例不存在，系统首先会确认是否有目标Activity期望的任务栈，如果没有，就首先创建目标Activity期望的任务栈，然后创建目标Activity实例并添加到期望的任务栈中；相反，若存在期望的任务栈，那么就直接创建目标Activity实例并将其添加到期望的任务栈。

而Activity期望的任务栈名称就是通过上面介绍的`android:taskAffinity`属性进行设置的。

OK，针对上述情况，我们来看两个具体案例：

**案例1**，两个Activity的taskAffinity属性相同，具体步骤如下所示：

1. 创建MainActivity，并且在onCreate方法中打印出当前Activity所在的任务栈ID，在onNewIntent方法中打印出"onNewIntent"字符串。
2. 创建FirstActivity，并且在onCreate方法中打印出当前Activity所在的任务栈ID，在onNewIntent方法中打印出"onNewIntent"字符串。
3. 设置MainActivity的启动模式为Standard，设置FirstActivity的启动模式为SingleTask。
4. 首先启动MainActivity，接着通过MainActivity启动FirstActivity，然后再通过FirstActivity启动MainActivity，如此反复。

当第一次通过MainActivity启动FirstActivity时，任务栈如下所示：
![SingleTask案例1-第一次通过MainActivity启动FirstActivity](http://7xs2qy.com1.z0.glb.clouddn.com/SingleTask%E6%A1%88%E4%BE%8B1-%E7%AC%AC%E4%B8%80%E6%AC%A1%E9%80%9A%E8%BF%87MainActivity%E5%90%AF%E5%8A%A8FirstActivity.png)  可见，因为两个Activity期望的任务栈相同，因此MainActivity和FirstActivity处于相同的任务栈中（TaskId为2074）。

然后，通过FirstActivity启动MainActivity时，任务栈如下所示：
![SingleTask案例1-第一次通过FirstActivity启动MainActivity](http://7xs2qy.com1.z0.glb.clouddn.com/SingleTask%E6%A1%88%E4%BE%8B1-%E7%AC%AC%E4%B8%80%E6%AC%A1%E9%80%9A%E8%BF%87FirstActivity%E5%90%AF%E5%8A%A8MainActivity.png)

最后，再次通过MainActivity启动FirstActivity时，任务栈如下所示：
![SingleTask案例1-再次通过MainActivity启动FirstActivity](http://7xs2qy.com1.z0.glb.clouddn.com/SingleTask%E6%A1%88%E4%BE%8B1-%E5%86%8D%E6%AC%A1%E9%80%9A%E8%BF%87MainActivity%E5%90%AF%E5%8A%A8FirstActivity.png)  可见，并没有为FirstActivity创建新的实例，而是把FirstActivity之上的MainActivity出栈，直接复用已有FirstActivity实例。

通过MainActivity打印出的日志如下所示：
![SingleTask案例1-MainActivity](http://7xs2qy.com1.z0.glb.clouddn.com/SingleTask%E6%A1%88%E4%BE%8B1-MainActivity.png)

通过FirstActivity打印出的日志如下所示：
![SingleTask案例1-FirstActivity](http://7xs2qy.com1.z0.glb.clouddn.com/SingleTask%E6%A1%88%E4%BE%8B1-FirstActivity.png)

通过上面的任务栈和日志可知：当两个Activity的taskAffinity相同时，第一次启动FirstActivity时，是直接将其实例入栈到MainActivity所在的任务栈（其实MainActivity所在的任务栈就是FirstActivity期望的任务栈）；当第二次启动FirstActivity时，则是直接复用已有FirstActivity实例，同时调用其`onNewIntent`方法。

**案例2**，两个Activity的taskAffinity属性不同，具体步骤如下所示：

1. 创建MainActivity，并且在onCreate方法中打印出当前Activity所在的任务栈ID，在onNewIntent方法中打印出"onNewIntent"字符串。
2. 创建FirstActivity，并且在onCreate方法中打印出当前Activity所在的任务栈ID，在onNewIntent方法中打印出"onNewIntent"字符串。
3. 设置MainActivity的启动模式为Standard，设置FirstActivity的启动模式为SingleTask。同时设置FirstActivity的`android:taskAffinity`属性为leon.com.activitylaunchmode1，而MainActivity的`android:taskAffinity`属性为默认值，即App包名：leon.com.activitylaunchmode。
4. 首先启动MainActivity，接着通过MainActivity启动FirstActivity，然后再通过FirstActivity启动MainActivity，如此反复。

当第一次通过MainActivity启动FirstActivity时，任务栈如下所示：
![SingleTask案例2-第一次通过MainActivity启动FirstActivity](http://7xs2qy.com1.z0.glb.clouddn.com/SingleTask%E6%A1%88%E4%BE%8B2-%E7%AC%AC%E4%B8%80%E6%AC%A1%E9%80%9A%E8%BF%87MainActivity%E5%90%AF%E5%8A%A8FirstActivity.png)  可见，因为两个Activity期望的任务栈不同，因此系统会为FirstActivity创建新任务栈（TaskId为2078）。

然后，通过FirstActivity启动MainActivity时，任务栈如下所示：
![SingleTask案例2-第一次通过FirstActivity启动MainActivity](http://7xs2qy.com1.z0.glb.clouddn.com/SingleTask%E6%A1%88%E4%BE%8B2-%E7%AC%AC%E4%B8%80%E6%AC%A1%E9%80%9A%E8%BF%87FirstActivity%E5%90%AF%E5%8A%A8MainActivity.png)  因为MainActivity的启动模式为Standard，所以其Activity实例会入栈到启动它的FirstActivity所在的任务栈中。

最后，再次通过MainActivity启动FirstActivity时，任务栈如下所示：
![SingleTask案例2-再次通过MainActivity启动FirstActivity](http://7xs2qy.com1.z0.glb.clouddn.com/SingleTask%E6%A1%88%E4%BE%8B2-%E5%86%8D%E6%AC%A1%E9%80%9A%E8%BF%87MainActivity%E5%90%AF%E5%8A%A8FirstActivity.png)  可见，并没有为FirstActivity创建新的实例，而是把FirstActivity之上的MainActivity出栈，直接复用已有FirstActivity实例。

通过MainActivity打印出的日志如下所示：
![SingleTask案例2-MainActivity](http://7xs2qy.com1.z0.glb.clouddn.com/SingleTask%E6%A1%88%E4%BE%8B2-MainActivity.png)

通过FirstActivity打印出的日志如下所示：
![SingleTask案例2-FirstActivity](http://7xs2qy.com1.z0.glb.clouddn.com/SingleTask%E6%A1%88%E4%BE%8B2-FirstActivity.png)

通过上面的任务栈和日志可知：和案例1的唯一不同就是第一次启动FirstActivity时，系统会为其创建新的任务栈。

---

总的来说：SingleTask模式与android:taskAffinity属性相关。以MainActivity启动FirstActivity为例（MainActivity为Standard模式，FirstActivity为SingleTask模式）：

>1. 当MainActivity和FirstActivity的taskAffinity属性相同时：第一次启动FirstActivity时，并不会启动新的任务栈，而是直接将FirstActivity添加到MainActivity所在的任务栈；否则，将FirstActivity所在任务栈中位于FirstActivity之上的全部Activity都删除，直接跳转到FirstActivity中。

>2. 当MainActivity和FirstActivity的taskAffinity属性不同时：第一次启动FirstActivity时，会创建新的任务栈，然后将FirstActivity添加到新的任务栈中；否则，将FirstActivity所在任务栈中位于FirstActivity之上的全部Activity都删除，直接跳转到FirstActivity中。

另外，当目标Activity处于栈顶时，启动SingleTask模式的目标Activity，其回调函数的顺序是`onPause -> onNewIntent -> onResume`，和SingleTop模式相同。

而当目标Activity存在，但是不位于栈顶时，启动SingleTask模式的目标Activity，其回调函数的顺序是`onNewIntent -> onRestart -> onStart -> onResume`。

### `SingleInstance`
单实例模式。该模式是SingleTask的强化，除了具有SingleTask的所有特性外，还强调任意时刻只允许存在唯一的Activity实例，且该Activity实例独自占有一个任务栈。即该任务栈只能容纳该Activity实例，不能再添加其他Activity实例到该任务栈，如果该Activity实例已经存在于某个任务栈，则直接跳转到该任务栈。

OK，下面通过一个案例详细分析下SingleInstance模式，具体步骤如下所示：

1. 创建MainActivity，并且在onCreate方法中打印出当前Activity所在的任务栈ID，在onNewIntent方法中打印出"onNewIntent"字符串。
2. 创建FirstActivity，并且在onCreate方法中打印出当前Activity所在的任务栈ID，在onNewIntent方法中打印出"onNewIntent"字符串。
3. 设置MainActivity的启动模式为Standard，设置FirstActivity的启动模式为SingleInstance。
4. 首先启动MainActivity，接着通过MainActivity启动FirstActivity，然后再通过FirstActivity启动MainActivity，如此反复。

当第一次通过MainActivity启动FirstActivity时，任务栈如下所示：
![SingleInstance-第一次通过MainActivity启动FirstActivity](http://7xs2qy.com1.z0.glb.clouddn.com/SingleInstance-%E7%AC%AC%E4%B8%80%E6%AC%A1%E9%80%9A%E8%BF%87MainActivity%E5%90%AF%E5%8A%A8FirstActivity.png) 可见，此时FirstActivity处于独立的任务栈中（TaskId为2087）。

然后，通过FirstActivity启动MainActivity时，任务栈如下所示：
![SingleInstance-第一次通过FirstActivity启动MainActivity](http://7xs2qy.com1.z0.glb.clouddn.com/SingleInstance-%E7%AC%AC%E4%B8%80%E6%AC%A1%E9%80%9A%E8%BF%87FirstActivity%E5%90%AF%E5%8A%A8MainActivity.png)  可见，新创建的MainActivity实例不是位于FirstActivity实例所处的任务栈，而是位于之前MainActivity实例所处的任务栈（TaskId为2086）。这也间接证明了SingleInstance模式的Activity是独占一个任务栈的。

最后，再次通过MainActivity启动FirstActivity时，任务栈如下所示：
![SingleInstance-再次通过MainActivity启动FirstActivity](http://7xs2qy.com1.z0.glb.clouddn.com/SingleInstance-%E5%86%8D%E6%AC%A1%E9%80%9A%E8%BF%87MainActivity%E5%90%AF%E5%8A%A8FirstActivity.png)  可见，并没有为FirstActivity创建新的实例，仅仅是把TaskId为2087的任务栈切换到了前台。

通过MainActivity打印出的日志如下所示：
![SingleInstance-MainActivity](http://7xs2qy.com1.z0.glb.clouddn.com/SingleInstance-MainActivity.png)

通过FirstActivity打印出的日志如下所示：
![SingleInstance-FirstActivity](http://7xs2qy.com1.z0.glb.clouddn.com/SingleInstance-FirstActivity.png)

通过上面的任务栈和日志可知：尽管多次启动了SingleInstance模式的FirstActivity，但只在第一次的时候创建了Activity实例，并且为该实例创建了新的任务栈，后续的多次启动，仅仅调用了已有FirstActivity实例的`onNewIntent`方法，并将其任务栈切换到了前台。同时，FirstActivity所处任务栈的TaskId为2087，MainActivity所处任务栈的TaskId为2086，MainActivity的实例永远不可能位于FirstActivity所处的任务栈中，即SingleInstance模式的Activity独占一个任务栈。

此外，还有一点需要注意：
当目标Activity实例已经存在时，启动SingleInstance模式的目标Activity，其回调函数的顺序是`onNewIntent -> onRestart -> onStart -> onResume`。

## 特殊案例
上述通过具体案例详细分析了四种启动模式，但是还有一些特殊使用场景需要详细分析下。
### 针对SingleTask模式的进一步强化
假设现在有两个任务栈：前台任务栈中包含ActivityA和ActivityB，后台任务栈中包含ActivityC和ActivityD，且前台任务栈中Activity的启动模式均为Standard，后台任务栈中Activity的启动模式均为SingleTask，两个任务栈的栈名是不同的。

现在有两种典型的使用场景：
场景1：当通过前台任务栈中的ActivityB启动后台任务栈中ActivityD时，系统会将后台任务栈会切换到前台。此时，当用户通过“back”键退出时，整个退出顺序应该是`ActivityD -> ActivityC -> ActivityB -> ActivityA`。

场景2：当通过前台任务栈中的ActivityB启动后台任务栈中ActivityC时，系统会将后台任务栈会切换到前台，同时把ActivityC之上的ActivityD出栈。此时，当用户通过“back”键退出时，整个退出顺序应该是`ActivityC -> ActivityB -> ActivityA`。

这里我们通过具体案例验证下场景1（场景2比较类似，不再赘述），具体步骤如下所示：

1. 首先创建包名为leon.com.launchmodeab的App1，包含ActivityA和ActivityB，设置其启动模式为Standard，然后创建包名为leon.com.launchmodecd的App2，包含ActivityC和ActivityD，设置其启动模式为SingleTask。
2. 接着启动App2，并依次启动ActivityC和ActivityD，然后通过“home”键切换到后台。
3. 然后启动App1，并依次启动ActivityA和ActivityB。
4. 最后，通过ActivityB启动ActivityD，并通过“back”键依次退出，观察Activity的退出顺序。

当完成步骤1~3时，任务栈如下所示：
![前台任务栈和后台任务栈都已启动](http://7xs2qy.com1.z0.glb.clouddn.com/%E5%89%8D%E5%8F%B0%E4%BB%BB%E5%8A%A1%E6%A0%88%E5%92%8C%E5%90%8E%E5%8F%B0%E4%BB%BB%E5%8A%A1%E6%A0%88%E9%83%BD%E5%B7%B2%E5%90%AF%E5%8A%A8.png)  可知，TaskIdId为2146的任务栈包含ActivityA和ActivityB，TaskId为2145的任务栈包含ActivityA和ActivityB。此时，若是直接通过“back”键退出，那么退出顺序是`ActivityB -> ActivityA`。

当通过ActivityB启动ActivityD后，任务栈如下所示：
![通过ActivityB启动ActivityD后](http://7xs2qy.com1.z0.glb.clouddn.com/%E9%80%9A%E8%BF%87ActivityB%E5%90%AF%E5%8A%A8ActivityD%E5%90%8E.png)  可知，包含ActivityD的任务栈被切换到了前台，虽然ActivityC和ActivityD被隔开了，但是从TaskId和栈名来看，他们还是是同一个任务栈，而ActivityA和ActivityB所在的栈则在后面。此时，若通过“back”键退出，那么退出顺序是`ActivityD -> ActivityC -> ActivityB -> ActivityA`。

### taskAffinity和allowTaskReparenting结合使用场景
`android:allowTaskReparenting`主要用于Activity的迁移。当allowTaskReparenting为true时，表示该Activity能从其他任务栈迁移到期望的任务栈，当allowTaskReparenting的值为“false”，表示该Activity不能从其他任务栈进行迁移，默认为false。这个比较抽象，我们看一个具体的案例，具体步骤如下所示：

1. 首先创建包名为leon.com.launchmodeab的App1，包含ActivityA，设置其启动模式为Standard，然后创建包名为leon.com.launchmodecd的App2，包含ActivityC，设置其启动模式为Standard。
2. 接着启动App1，并通过App1的ActivityA启动App2的ActivityC，，然后通过“home”键切换到后台。
3. 然后启动App2，查看任务栈情况。

OK，首先来看下ActivityC的allowTaskReparenting属性为true的情况。
完成上述1~2步骤后的任务栈如下所示：
![Activity迁移1-2](http://7xs2qy.com1.z0.glb.clouddn.com/Activity%E8%BF%81%E7%A7%BB1-2.png)  可见，此时ActivityC和ActivityA处于同一个任务栈。

完成第3步后的任务栈如下所示：
![Activity迁移3](http://7xs2qy.com1.z0.glb.clouddn.com/Activity%E8%BF%81%E7%A7%BB3.png)  可见，此时ActivityC从TaskId为2166的任务栈迁移到了TaskId为2167的任务栈中，符合我们的预期。

然后看下ActivityC的allowTaskReparenting属性为false的情况。
完成上述1~2步骤后的任务栈如下所示：
![Activity不迁移1-2](http://7xs2qy.com1.z0.glb.clouddn.com/Activity%E4%B8%8D%E8%BF%81%E7%A7%BB1-2.png)  可见，此时ActivityC和ActivityA处于同一个任务栈。

完成第3步后的任务栈如下所示：
![Activity不迁移3](http://7xs2qy.com1.z0.glb.clouddn.com/Activity%E4%B8%8D%E8%BF%81%E7%A7%BB3.png)  可见，原来的ActivityC实例并没有被迁移，而是系统为ActivityC创建了新的实例，也符合我们的预期。

---

先介绍这两个特殊使用场景，后面会陆续补充...


## 参考文章
1. [Activity启动模式与任务栈(Task)全面深入记录（上）](http://blog.csdn.net/javazejian/article/details/52071885)
2. [Activity启动模式与任务栈(Task)全面深入记录（下）](http://blog.csdn.net/javazejian/article/details/52072131)
3. [Android 之Activity启动模式(一)之 lauchMode](http://wangkuiwu.github.io/2014/06/26/LaunchMode/)
4. [Android 之Activity启动模式(二)之 Intent的Flag属性](http://wangkuiwu.github.io/2014/06/26/IntentFlag/)
5. [Android 之Activity启动模式(三)之 启动模式的其它属性](http://wangkuiwu.github.io/2014/06/26/OtherModeAttrs/)

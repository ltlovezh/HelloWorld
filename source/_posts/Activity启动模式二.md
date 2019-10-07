---
title: Activity启动模式二
date: 2016-09-03 17:16:35
tags:
- Android
- Activity启动模式
categories:
- Android
---
上篇文章[Activity启动模式一](http://ltlovezh.com/2016/08/28/Activity%E5%90%AF%E5%8A%A8%E6%A8%A1%E5%BC%8F%E4%B8%80/)主要介绍了Activity的四种启动模式，这些启动模式都是在`AndroidManifest`中进行配置的。除此之外，Android系统还通过Intent类提供了一些标志位，同样可以指定Activity的启动模式。本文将介绍下这些和Activity启动相关的标志位。

<!-- more -->

一般情况下，我们在启动目标Activity的Intent中指定这些标志位，如下所示：

``` java
Intent intent = new Intent();
intent.setClass(MainActivity.this,FirstActivity.class);
intent.addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP);
startActivity(intent);
```
当同时在配置文件中指定launchMode和在Intent中指定上面的标志位时，以标志位为准，即代码的优先级比AndroidManifest的优先级更高！

下面我们通过具体案例看下和Activity启动相关的4种标志位：

## Activity启动相关的四种标志位
### FLAG_ACTIVITY_NEW_TASK
官方文档上介绍该标志位和SingleTask启动模式具有相同效果，其实不然。应该是`FLAG_ACTIVITY_NEW_TASK + FLAG_ACTIVITY_CLEAR_TOP`一起等同于SingleTask启动模式。

这里我们通过4个案例，看下单独使用FLAG_ACTIVITY_NEW_TASK标志位和一起使用FLAG_ACTIVITY_NEW_TASK、FLAG_ACTIVITY_CLEAR_TOP标志位的具体效果。
#### 案例1
两个Activity的taskAffinity属性相同，具体步骤如下所示：

1. 创建MainActivity和FirstActivity，且两者的启动模式都是Standard，TaskAffinity也相同。
2. 首先启动MainActivity，接着通过MainActivity启动FirstActivity，然后再通过FirstActivity启动MainActivity，如此反复。
3. 启动FirstActivity的Intent中指定了FLAG_ACTIVITY_NEW_TASK标志位，启动MainActivity的Intent不包含任何标志位。

经过`MainActivity -> FirstActivity -> MainActivity`后，最终的任务栈如下所示：
![new_task-taskAffinity相同-1](http://7xs2qy.com1.z0.glb.clouddn.com/new_task-taskAffinity%E7%9B%B8%E5%90%8C-1.png)  可见，因为两个Activity的taskAffinity属性相同，所以第一次启动FirstActivity时并没有创建新的任务栈。

最后，MainActivity再次启动FirstActivity后，最终的任务栈如下所示：
![new_task-taskAffinity相同-2](http://7xs2qy.com1.z0.glb.clouddn.com/new_task-taskAffinity%E7%9B%B8%E5%90%8C-2.png)  因为只有FLAG_ACTIVITY_NEW_TASK标志位，没有FLAG_ACTIVITY_CLEAR_TOP，所以无法清除FirstActivity之上的MainActivity，复用已有FirstActivity。而是创建了新的FirstActivity。

通过上面的任务栈可知：在taskAffinity相同的情况下，单独添加FLAG_ACTIVITY_NEW_TASK不起任何作用。
#### 案例2
两个Activity的taskAffinity属性不相同，具体步骤如下所示：

1. 创建MainActivity和FirstActivity，且两者的启动模式都是Standard，但是TaskAffinity不同。
2. 首先启动MainActivity，接着通过MainActivity启动FirstActivity，然后再通过FirstActivity启动MainActivity，如此反复。
3. 启动FirstActivity的Intent中指定了FLAG_ACTIVITY_NEW_TASK标志位，启动MainActivity的Intent不包含任何标志位。

经过`MainActivity -> FirstActivity -> MainActivity`后，最终的任务栈如下所示：
![new_task-taskAffinity不同-1](http://7xs2qy.com1.z0.glb.clouddn.com/new_task-taskAffinity%E4%B8%8D%E5%90%8C-2.png)  可见，因为两个Activity的taskAffinity属性不同，所以第一次启动FirstActivity时，创建了新的任务栈（TaskId为1979）.

最后，MainActivity再次启动FirstActivity后，最终的任务栈如下所示：
![new_task-taskAffinity不同-2](http://7xs2qy.com1.z0.glb.clouddn.com/new_task-taskAffinity%E4%B8%8D%E5%90%8C-2.png)  可以看到，任务栈没有发生任何变化，也没有创建新的FirstActivity实例。这是因为FirstActivity实例已经存在于它所期望的任务栈中，而单独添加FLAG_ACTIVITY_NEW_TASK标志位又不会清除任务栈中位于FirstActivity之上的Activity实例，所以就没有发生跳转。

通过上面的任务栈可知：在taskAffinity不同的情况下，第一次启动FirstActivity时，会新建一个任务栈，并将FirstActivity实例添加到该task中。这与SingleTask启动模式产生的效果是一致的。但是，当企图再次从MainActivity进入到FirstActivity时，却什么也没有发生，原因上面已经说明了。

综上所述，单独使用FLAG_ACTIVITY_NEW_TASK会产生非常奇怪的行为，因此一般和FLAG_ACTIVITY_CLEAR_TOP标志位一起使用，这样可以实现类似SingleTask的效果，但是不完全相同，下面会进行介绍。

下面我们再通过两个案例看下`FLAG_ACTIVITY_NEW_TASK + FLAG_ACTIVITY_CLEAR_TOP`一起使用的效果。
#### 案例3
两个Activity的taskAffinity属性相同，具体步骤如下所示：

1. 创建MainActivity和FirstActivity，且两者的启动模式都是Standard，TaskAffinity也相同。
2. 首先启动MainActivity，接着通过MainActivity启动FirstActivity，然后再通过FirstActivity启动MainActivity，如此反复。
3. 启动FirstActivity的Intent中指定了`FLAG_ACTIVITY_NEW_TASK + FLAG_ACTIVITY_CLEAR_TOP`标志位，启动MainActivity的Intent不包含任何标志位.

经过`MainActivity -> FirstActivity -> MainActivity`后，最终的任务栈如下所示：
![new_task-clear_top-taskAffinity相同-1](http://7xs2qy.com1.z0.glb.clouddn.com/new_task-clear_top-taskAffinity%E7%9B%B8%E5%90%8C-1.png)  可见，因为两个Activity的taskAffinity属性相同，所以第一次启动FirstActivity时并没有创建新的任务栈。

最后，MainActivity再次启动FirstActivity后，最终的任务栈如下所示：
![new_task-clear_top-taskAffinity相同-2](http://7xs2qy.com1.z0.glb.clouddn.com/new_task-clear_top-taskAffinity%E7%9B%B8%E5%90%8C-2.png)  貌似是和SingTask启动模式的效果相同，但是细看一下就会发现区别：**前后两次FirstActivity实例是不同的，也就是没有复用第一次启动的FirstActivity，而是清除了FirstActivity及其之上的所有Activity，然后新建FirstActivity实例添加到了当前任务栈**。

通过上面的任务栈可知：在taskAffinity相同的情况下，`FLAG_ACTIVITY_NEW_TASK + FLAG_ACTIVITY_CLEAR_TOP`标志位组合确实不会创建新的任务栈，但是会清除了FirstActivity及其之上的所有Activity，并新建FirstActivity实例添加到了当前任务栈，这点是和SingleTask模式的差异。（SingleTask仅会清除FirstActivity之上的Activity，然后复用已有的FirstActivity）
#### 案例4
两个Activity的taskAffinity属性不相同，具体步骤如下所示：

1. 创建MainActivity和FirstActivity，且两者的启动模式都是Standard，但是TaskAffinity不同。
2. 首先启动MainActivity，接着通过MainActivity启动FirstActivity，然后再通过FirstActivity启动MainActivity，如此反复。
3. 启动FirstActivity的Intent中指定了`FLAG_ACTIVITY_NEW_TASK + FLAG_ACTIVITY_CLEAR_TOP`标志位，启动MainActivity的Intent不包含任何标志位。

经过`MainActivity -> FirstActivity -> MainActivity`后，最终的任务栈如下所示：
![new_task-clear_top-taskAffinity不同-1](http://7xs2qy.com1.z0.glb.clouddn.com/new_task-clear_top-taskAffinity%E4%B8%8D%E5%90%8C-1.png)  可见，因为两个Activity的taskAffinity属性不同，所以第一次启动FirstActivity时，创建了新的任务栈（TaskId为1980）.

最后，MainActivity再次启动FirstActivity后，最终的任务栈如下所示：
![new_task-clear_top-taskAffinity不同-2](http://7xs2qy.com1.z0.glb.clouddn.com/new_task-clear_top-taskAffinity%E4%B8%8D%E5%90%8C-2.png)  可以看到，和案例3 taskAffinity属性相同的情况类似（除了创建了新的任务栈），直接清除了FirstActivity及其之上的所有Activity，然后创建新的FirstActivity实例添加到新的任务栈（TaskId为1980）。

通过上面的任务栈可知：在taskAffinity不同的情况下，`FLAG_ACTIVITY_NEW_TASK + FLAG_ACTIVITY_CLEAR_TOP`标志位组合会为首次启动的FirstActivity创建新的任务栈，其他的逻辑与案例3基本相同，**都是直接清除了FirstActivity及其之上的所有Activity，然后创建新的FirstActivity实例，这也是和SingleTask模式的区别**。

综上所述，`FLAG_ACTIVITY_NEW_TASK + FLAG_ACTIVITY_CLEAR_TOP`标志位组合产生的效果总体上和SingleTask模式相同，除了不会复用FirstActivity实例之外。
### FLAG_ACTIVITY_CLEAR_TOP
该标志位在上面已经使用过了，作用是清除包含目标Activity的任务栈中位于该Activity实例之上的其他Activity实例。但是是复用已有的目标Activity，还是像上面那样先删除后重建，则有以下规则：

1. 若是单独使用FLAG_ACTIVITY_CLEAR_TOP，那么只有非Standard启动模式的目标Activity才会被复用，其他启动模式的目标Activity都先被删除，然后被重新创建并入栈。
2. 若是使用FLAG_ACTIVITY_SINGLE_TOP和FLAG_ACTIVITY_CLEAR_TOP标志位组合，那么不管目标Activity是什么启动模式，都会被复用。

因为上面的案例3和案例4都不符合上面的两条规则，所以FirstActivity才会先被清除，然后重建。

下面我们通过几个具体案例，验证下上面的两条规则：
#### 案例1
具体步骤如下所示：

1. 创建MainActivity和FirstActivity，且两者的启动模式都是Standard。
2. 首先启动MainActivity，接着通过MainActivity启动FirstActivity，然后再通过FirstActivity启动MainActivity，如此反复。
3. 启动FirstActivity的Intent中指定了`FLAG_ACTIVITY_CLEAR_TOP`标志位，启动MainActivity的Intent不包含任何标志位。

经过`MainActivity -> FirstActivity -> MainActivity`后，最终的任务栈如下所示：
![clear_top-standard-1](http://7xs2qy.com1.z0.glb.clouddn.com/clear_top-standard-1.png)

最后，MainActivity再次启动FirstActivity后，最终的任务栈如下所示：
![clear_top-standard-2](http://7xs2qy.com1.z0.glb.clouddn.com/clear_top-standard-2.png)

通过上面的任务栈可知：第二次`MainActivity -> FirstActivity`时，会先把FirstActivity以及之上的Activity全部销毁，然后创建新的FirstActivity入栈，正符合我们的预期。
#### 案例2 
具体步骤如下所示：

1. 创建MainActivity和FirstActivity，MainActivity的启动模式是Standard，FirstActivity的启动模式是SingleTop。
2. 首先启动MainActivity，接着通过MainActivity启动FirstActivity，然后再通过FirstActivity启动MainActivity，如此反复。
3. 启动FirstActivity的Intent中指定了`FLAG_ACTIVITY_CLEAR_TOP`标志位，启动MainActivity的Intent不包含任何标志位。

经过`MainActivity -> FirstActivity -> MainActivity`后，最终的任务栈如下所示：
![clear_top-SingleTop-1](http://7xs2qy.com1.z0.glb.clouddn.com/clear_top-SingleTop-1.png)

最后，MainActivity再次启动FirstActivity后，最终的任务栈如下所示：
![clear_top-SingleTop-2](http://7xs2qy.com1.z0.glb.clouddn.com/clear_top-SingleTop-2.png)

通过上面的任务栈可知：第二次`MainActivity -> FirstActivity`时，会先把FirstActivity之上的Activity全部销毁，然后复用已有的FirstActivity实例，也符合我们的预期。

---

此外，当FirstActivity是SingleTask和SingleInstance模式时，其效果和SingleTop启动模式一样，也是首先销毁FirstActivity之上的所有Activity，然后复用已有的FirstActivity实例。

这里仅给出我验证的任务栈：
**当FirstActivity是SingleTask模式时（taskAffinity相同）**，经过`MainActivity -> FirstActivity -> MainActivity`后，最终的任务栈如下所示：
![clear_top-SingleTask-1](http://7xs2qy.com1.z0.glb.clouddn.com/clear_top-SingleTask-1.png)

最后，MainActivity再次启动FirstActivity后，最终的任务栈如下所示：
![clear_top-SingleTask-2](http://7xs2qy.com1.z0.glb.clouddn.com/clear_top-SingleTask-2.png)  可见，确实复用了已有的FirstActivity实例。

**当FirstActivity是SingleInstance模式时**，经过`MainActivity -> FirstActivity -> MainActivity`后，最终的任务栈如下所示：
![clear_top-SingleInstance-1](http://7xs2qy.com1.z0.glb.clouddn.com/clear_top-SingleInstance-1.png)

最后，MainActivity再次启动FirstActivity后，最终的任务栈如下所示：
![clear_top-SingleInstance-2](http://7xs2qy.com1.z0.glb.clouddn.com/clear_top-SingleInstance-2.png)
可见，确实复用了已有的FirstActivity实例。

综上所述，通过案例1和案例2，我们验证了规则1是OK的。
下面我们继续验证下规则2：
#### 案例3
具体步骤如下所示：

1. 创建MainActivity和FirstActivity，且两者的启动模式都是Standard。
2. 首先启动MainActivity，接着通过MainActivity启动FirstActivity，然后再通过FirstActivity启动MainActivity，如此反复。
3. 启动FirstActivity的Intent中指定了`FLAG_ACTIVITY_SINGLE_TOP + FLAG_ACTIVITY_CLEAR_TOP`标志位，启动MainActivity的Intent不包含任何标志位。

经过`MainActivity -> FirstActivity -> MainActivity`后，最终的任务栈如下所示：
![single_top-clear_top-1](http://7xs2qy.com1.z0.glb.clouddn.com/single_top-clear_top-1.png)

最后，MainActivity再次启动FirstActivity后，最终的任务栈如下所示：
![single_top-clear_top-2](http://7xs2qy.com1.z0.glb.clouddn.com/single_top-clear_top-2.png)

通过上面的任务栈可知：第二次`MainActivity -> FirstActivity`时，会先把FirstActivity之上的Activity全部销毁，然后复用已有的FirstActivity实例，也符合我们的预期。

综上所述，既然在`FLAG_ACTIVITY_SINGLE_TOP + FLAG_ACTIVITY_CLEAR_TOP`标志位组合的情况下，Standard模式的FirstActivity都已经被复用了，那么其他启动模式的Activity也必然会被复用。（单独使用FLAG_ACTIVITY_CLEAR_TOP都会被复用，何况又添加了FLAG_ACTIVITY_SINGLE_TOP标志位，通过Demo验证也确实如此，就不再给出具体案例了）。

因此，上面的两条规则还要是遵守的^_^。
### FLAG_ACTIVITY_CLEAR_TASK
该标志位一般和FLAG_ACTIVITY_CLEAR_TASK一起使用，目的是启动目标Activity时，首先清空已经存在的目标Activity实例所在的任务栈，这自然也就清除了之前存在的目标Activity实例，然后创建新的目标Activity实例并入栈。

经过Demo测试发现，单独使用FLAG_ACTIVITY_CLEAR_TASK标志位，不管taskAffinity是否相同，都不会产生什么效果。例如下面是单独使用该标志位的最终任务栈（taskAffinity相同 or 不同都是这样）：
![clear_task](http://7xs2qy.com1.z0.glb.clouddn.com/clear_task.png)
感兴趣的可以自行尝试，这里不再赘述。

下面看下`FLAG_ACTIVITY_NEW_TASK + FLAG_ACTIVITY_CLEAR_TASK`标志位组合在taskAffinity相同 and 不同的情况下的效果。
#### 案例1
两个Activity的taskAffinity属性相同，具体步骤如下所示：

1. 创建MainActivity和FirstActivity，且两者的启动模式都是Standard，**taskAffinity也相同**。
2. 首先启动MainActivity，接着通过MainActivity启动FirstActivity，然后再通过FirstActivity启动MainActivity，如此反复。
3. 启动FirstActivity的Intent中指定了`FLAG_ACTIVITY_NEW_TASK + FLAG_ACTIVITY_CLEAR_TASK`标志位，启动MainActivity的Intent不包含任何标志位。

第一次启动MainActivity后，任务栈如下所示：
![new_task-clear_task-1](http://7xs2qy.com1.z0.glb.clouddn.com/new_task-clear_task-1.png)

接着MainActivity启动FirstActivity后，任务栈如下所示：
![new_task-clear_task-2](http://7xs2qy.com1.z0.glb.clouddn.com/new_task-clear_task-2.png)  首先启动的MainActivity居然被清除了，然后才创建了FirstActivity实例。

然后FirstActivity启动MainActivity后，任务栈如下所示：
![new_task-clear_task-3](http://7xs2qy.com1.z0.glb.clouddn.com/new_task-clear_task-3.png)

最后MainActivity再次启动FirstActivity后，任务栈如下所示：
![new_task-clear_task-4](http://7xs2qy.com1.z0.glb.clouddn.com/new_task-clear_task-4.png)  可见，系统再次清除了FirstActivity及其之上的MainActivity，然后重新创建了新的FirstActivity实例并入栈。

通过上面的任务栈可知：当通过`FLAG_ACTIVITY_NEW_TASK + FLAG_ACTIVITY_CLEAR_TASK`标志位组合启动FirstActivity时，**首先会清空FirstActivity所在的任务栈，然后再创建新的FirstActivity实例并入栈**。
#### 案例2
两个Activity的taskAffinity属性不相同，具体步骤如下所示：

1. 创建MainActivity和FirstActivity，且两者的启动模式都是Standard，**taskAffinity不相同**。
2. 首先启动MainActivity，接着通过MainActivity启动FirstActivity，然后再通过FirstActivity启动MainActivity，如此反复。
3. 启动FirstActivity的Intent中指定了`FLAG_ACTIVITY_NEW_TASK + FLAG_ACTIVITY_CLEAR_TASK`标志位，启动MainActivity的Intent不包含任何标志位。

第一次MainActivity启动FirstActivity后，任务栈如下所示：
![new_task-clear_task-5](http://7xs2qy.com1.z0.glb.clouddn.com/new_task-clear_task-5.png)  因为taskAffinity不同，所以为FirstActivity实例创建了新的任务栈。

然后FirstActivity启动MainActivity后，任务栈如下所示：
![new_task-clear_task-6](http://7xs2qy.com1.z0.glb.clouddn.com/new_task-clear_task-6.png)

最后MainActivity再次启动FirstActivity后，任务栈如下所示：
![new_task-clear_task-7](http://7xs2qy.com1.z0.glb.clouddn.com/new_task-clear_task-7.png)  可见，系统首先清除了任务栈（TaskId为2040）中FirstActivity及其之上的MainActivity，然后重新创建了新的FirstActivity实例并入栈（TaskId为2040）。

通过上面的任务栈可知：当通过`FLAG_ACTIVITY_NEW_TASK + FLAG_ACTIVITY_CLEAR_TASK`标志位组合启动FirstActivity时，**首先会清空FirstActivity所在的任务栈，然后再创建新的FirstActivity实例并入栈，这和taskAffinity属性相同是一致的效果，只不过这里第一次为FirstActivity创建了新的任务栈**。

综上所述，当通过`FLAG_ACTIVITY_NEW_TASK + FLAG_ACTIVITY_CLEAR_TASK`标志位组合启动目标Activity时，首先会清空目标Activity所在的任务栈（若目标Activity是第一次启动，那么会清空目标Activity所期望的任务栈），然后创建新的目标Activity实例并入栈。
### FLAG_ACTIVITY_SINGLE_TOP
该标志单独使用就可以达到SingleTop启动模式的效果，该类效果可以参考前篇文章，这里不再赘述。

## Activity启动相关的属性
这里简单介绍一些`AndroidManifest`配置文件中与启动模式相关的属性。其实前一篇文章已经介绍了两个比较常用的属性：`android:taskAffinity`和`android:allowTaskReparenting`，下面再补充一些不常用的属性。
### android:clearTaskOnLaunch
字面意思是：是否在启动时清除任务栈，该属性默认为false，表示是否在APP启动时，清除任务栈中除根Activity之外的其他Activity实例。该属性只对任务栈内的根Activity起作用，任务栈内其他的Activity都会被忽略。

比如：我们有一个App，依次启动了ActivityA、ActivityB和ActivityC，其中ActivityA和ActivityB都设置clearTaskOnLaunch属性为true。那么首先点击“Home”建回到Launcher界面，然后再启动该App，我们看到是ActivityA界面，而ActivityB和ActivityC都被销毁了。
除非栈底的ActivityA已经被销毁，那么上面设置clearTaskOnLaunch属性为true的activity(B)才会生效。

### android:finishOnTaskLaunch
该属性默认值为false，表示当离开当前Activity所在的任务栈时，立即销毁当前Activity，而其他Activity不受影响。
### android:alwaysRetainTaskState
该属性默认值为false，表示App的任务栈是否保持原来的状态。该属性只对task的根Activity起作用，其他的Activity都会被忽略。 默认情况下，若一个App在后台停留的时间太长，那么再启动该App时，系统会清除任务栈中除了根Activity之外的其他Activity，但是如果根Activity设置了该属性，那么再次启动应用时，就不会清除Task中的Activity，而是保持原样，因此仍然可以看到上一次操作的界面。 

---

OK，Activity启动模式相关的内容就介绍这些吧，希望对自己对感兴趣的同学都有所帮助^_^。

---
title: Android View系统那些事一
date: 2016-06-26 21:57:08
tags:
 - Android
 - View
 - View系统
 - OverScroller
 - GestureDetector
categories:
 - Android
---
本篇文章打算介绍下View的坐标、自定义View的手势检测以及实现View内容滚动的几种方式。希望对有需要的同学有所帮助。

## View的坐标
在自定义View中，经常需要处理各种坐标之间的转换，下图展示了View中的各种坐标：
<img src="http://7xs2qy.com1.z0.glb.clouddn.com/View%E5%9D%90%E6%A0%87%E7%B3%BB.png" width="400" alt="View坐标系"/>

<!-- more -->

简单解释下上图的含义：

针对一个普通View：

* `getTop`方法表示view自身的顶边到其父布局顶边的距离
* `getLeft`方法表示view自身的左边到其父布局左边的距离
* `getRight`方法表示view自身的右边到其父布局左边的距离
* `getBottom`方法表示view自身的底边到其父布局顶边的距离。

在处理一个普通View的Touch事件时：

* `getX`方法表示Touch事件在当前View坐标系中X轴上的坐标，即相对于view左边缘的距离
* `getY`方法表示Touch事件在当前View坐标系中Y轴上的坐标，即相对于View上边缘的距离
* `getRawX`方法表示Touch事件在屏幕坐标系中X轴上的坐标，即相对于整个屏幕左边缘的距离
* `getRawY`方法表示Touch事件在屏幕坐标系中Y轴上的坐标，即相对于整个屏幕上边缘的距离。

## `GestureDetector`手势检测
我们在自定义View时，经常需要处理Touch事件，而`GestureDetector`就是一个辅助我们检测用户单击、双击、长按和滚动等行为的工具类。

该工具类的使用也很简单。首先创建一个`GestureDetector`对象，并把实现事件监听接口的类作为参数传进到对象中。其中，可以实现的事件监听接口有两个，分别是`OnGestureListener`接口负责监听单击、长按和滚动等事件，和`OnDoubleTapListener`接口负责监听双击事件。
``` java
GestureDetector mGestureDetector = new GestureDetector(实现上述事件监听接口的类对象);
```
然后，在自定义View的`onTouchEvent`方法中，把事件处理权委托给`GestureDetector`。
``` java
mGestureDetector.onTouchEvent(event);
```

最后，根据具体情况，在不同的事件回调方法中处理业务逻辑就OK了。

下面重点介绍一下两个事件监听接口中的回调方法：

### `onDown`
方法签名：
``` java
boolean onDown(MotionEvent e);
```
说明：**每次`ACTION_DOWN`事件发生时，都会调用该方法，事件e表示该ACTION_DOWN事件**。熟悉Android事件分发机制的同学应该知道`ACTION_DOWN`事件的重要性。若在该方法中返回了false（表示不消费该事件），那么后续的ACTION事件都不再会分发到当前View。因此，一般情况下，该方法要返回true。

### `onShowPress`
方法签名：
``` java
void onShowPress(MotionEvent e);
```
说明：若发生`ACTION_DOWN`事件后，在`ViewConfiguration.TAP_TIMEOUT`时间内，**既没有发生`ACTION_UP`事件，也没有触发`onScroll`滚动方法**，那么就会触发该方法，表示对用户行为的反馈，事件e表示`ACTION_DOWN`事件。

这里要特别注意和`onDown`方法的区别。`onShowPress`方法强调在一段时间内没有松开或者滚动的行为，才会触发。

### `onLongPress`
方法签名：
``` java
void onLongPress(MotionEvent e);
```
说明：若发生`ACTION_DOWN`事件后，在`ViewConfiguration.TAP_TIMEOUT + ViewConfiguration.LONGPRESS_TIMEOUT`时间内，**既没有发生`ACTION_UP`事件，也没有触发`onScroll`滚动方法**，那么就会触发该方法，表示用户的长按行为，事件e表示`ACTION_DOWN`事件。

这里结合前面两个方法看一个例子：下面是一次长按事件的日志：
![GestureDetector长按事件](http://7xs2qy.com1.z0.glb.clouddn.com/OnGestureListener%E9%95%BF%E6%8C%89%E4%BA%8B%E4%BB%B6.png)

从日志可以看出，在我的Nexus 6p上，`ACTION_DOWN`事件后，100ms后触发了onShowPress方法，600ms后触发了onLongPress方法。这个时间对应于`ViewConfiguration`类中相应变量，即取决于Android系统，每个手机都可能不同。

### `onSingleTapUp`
方法签名：
``` java
boolean onSingleTapUp(MotionEvent e);
```
说明：当用户手指离开屏幕，生成UP事件时会触发该方法，事件e表示`ACTION_UP`事件。

关于该方法有几点需要注意：

1. 在`ACTION_DOWN`和`ACTION_UP`事件之间，若触发了`onScroll`滚动方法，则不会触发该方法。
2. 该方法在长按事件发生时（即上面的onLongPress方法），不会被触发。
3. 该方法在双击事件的第一次UP事件时会被触发，但第二次UP事件时不会被触发了。也就是说触发该方法的原因可能并不是一个严格的单击事件，也有可能是双击事件中的第一个UP事件。

### `onDoubleTap`
方法签名：
``` java
boolean onDoubleTap(MotionEvent e);
```
说明：在一定时间内两次单击行为构成一次双击操作。严格来说，如果第二次单击行为的DOWN事件时间和第一次单击行为的UP事件时间之差小于`ViewConfiguration.DOUBLE_TAP_TIMEOUT`，那么则构成一次双击事件，事件e表示第一次单击行为的`ACTION_DOWN`事件。

有一点需要注意：触发该方法的确切时机是第二次单击行为的`ACTION_DOWN`事件，而不是`ACTION_UP`事件。也就是说双击事件是第二次按下屏幕的时候触发，而不是第二次离开屏幕的时候触发。关于双击事件下面还会继续介绍。

### `onSingleTapConfirmed`
方法签名：
``` java
boolean onSingleTapConfirmed(MotionEvent e);
```
说明：严格的单击操作，若该方法被触发，则说明这是一个严格的单击行为，那么后面不可能再紧跟着另一个单击行为，即这只可能是单击，而不可能是双击事件中的一次单击（这点是和onSingleTapUp方法的主要区别）。

这里要说下双击操作和严格的单击操作是怎么实现的，首先看一个示意图：
![双击操作和严格的单击操作示意图](http://7xs2qy.com1.z0.glb.clouddn.com/%E5%8F%8C%E5%87%BB%E4%BA%8B%E4%BB%B6%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

如图所示，A1、A2和A3分别表示三个时间点，时间线之上的两个变量分别表示A2和A3时间点的系统值，时间线之下的数值则是在Nexus 6P上的具体取值。

实际上在`ACTION_DOWN`事件发生后，系统会发出两个延时Message（假设为M1和M2），分别在A2和A3时间点触发。
其中，A2时间点的M1用于实现双击操作：若第二次单击操作发生时，M1还没有执行，那么说明在A1 ~ A2之间发生了两次单击操作，则构成了一次双击事件。
A3时间点的M2用于实现长按事件：若在A1 ~ A3时间内，没有触发`onScroll`滚动方法，也没有发生`ACTION_UP`事件，那么在A3时间点就会触发`onLongPress`方法。

而触发`onSingleTapConfirmed`方法的时机有两个：

1. 若在A1 ~ A2之间只有一次单击行为（即只有一次ACTION_UP事件，且没有触发onScroll滚动方法），那么就会在A2时间点会触发`onSingleTapConfirmed`方法，表示这是一次严格的单击操作。**之所以要等到A2时间点，而不是UP事件发生时就触发该方法，是因为在该次UP事件有可能是双击事件中的第一次单击操作，只有等到A2时间点，才能确定这是一次严格的单击操作，而不是双击操作的一部分**。此时，方法签名中的事件e表示`ACTION_DOWN`事件。

2. 若唯一的一次单击行为发生在A2 ~ A3之间（即只有一次ACTION_UP事件，且没有触发onScroll滚动方法），那么在`ACTION_UP`事件发生时，就会触发`onSingleTapConfirmed`方法，表示这是一次严格的单击操作。此时，方法签名中的事件e表示`ACTION_UP`事件。

总结一下，只有在[A2,A3)时间区间内，才会触发`onSingleTapConfirmed`方法，以表明这是一次严格的单击操作。而且双击操作和严格的单击操作是不可能同时发生的。

### `onDoubleTapEvent`
方法签名：
``` java
boolean onDoubleTapEvent(MotionEvent e);
```
说明：在触发`onDoubleTap`方法之后，就可以在`onDoubleTapEvent`方法中监听到双击事件发生后，从按下到弹起的所有触屏事件。也就是说**双击事件中的第二次单击行为的`ACTION_DOWN`事件之后的所有ACTION事件都会触发该方法**。

### `onScroll`
方法签名：
``` java
/**
该方法在滚动期间，会多次被调用

e1表示初始的ACTION_DOWN事件
e2表示滚动过程中的ACTION_MOVE事件
distanceX为本次在X轴上的滚动距离
distanceY为本次在Y轴上的滚动距离
这里的滚动距离是相对于上一次onScroll事件的距离，而不是e1 down事件和e2 move事件的距离
*/
boolean onScroll(MotionEvent e1, MotionEvent e2, float distanceX, float distanceY);
```
说明：手指按下屏幕并滚动会触发该方法，即由初始的DOWN事件和一些列的MOVE事件驱动该方法。

### `onFling`
方法签名：
``` java
/**
e1表示初始的ACTION_DOWN事件
e2表示手指离开View时的ACTION_UP事件
veloctiyX，velocitY为手指离开当前View时的滚动速度，以这两个速度为初始速度做匀减速运动，就是现在快速拖动列表后的延迟滚动效果。
*/
boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX, float velocityY);
```
说明：在快速滚动屏幕，抬起手指后，滚动效果并不会立即停止。而是会以当前的滚动速度，做匀减速运动，直接速度为0，才会停止滚动。`onFling`方法就是在手指离开屏幕时触发的。

## View的内容滚动
这里讨论的View滚动指的是View内容的滚动，而不是View位置的移动。当View的内容过长时，通过Scroll的方式，可以一点点的查看。View的内容滚动实际是修改View的`mScrollX`和`mScrollY`属性。

要实现对View的滚动，可以通过View的`scrollTo`或者`scrollBy`方法来实现（scrollBy最终也是通过scrollBy来实现的）。同时，View滚动后，会回调onScrollChanged方法。如下所示：
``` java
 //这里是滚动到具体的位置
 public void scrollTo(int x, int y) {
    if (mScrollX != x || mScrollY != y) {
        int oldX = mScrollX;
        int oldY = mScrollY;
        mScrollX = x;
        mScrollY = y;
        invalidateParentCaches();
        //回调方法
        onScrollChanged(mScrollX, mScrollY, oldX, oldY);
        if (!awakenScrollBars()) { //唤醒滚动条
            postInvalidateOnAnimation();
        }
    }
}
```

这里要明确下mScrollX和mScrollY属性的具体含义：
1. `mScrollX`:View左边缘和View内容左边缘在水平方向上的距离，并且当View内容左边缘在View左边缘的左边时，mScrollX为正值，否则为负值。
2. `mScrollY`:View上边缘和View内容上边缘在垂直方向上的距离，并且当View内容上边缘在View上边缘的上边时，mScrollY为正值，否则为负值。

下面来看个几个例子，如下所示：
![ScrollX](http://7xs2qy.com1.z0.glb.clouddn.com/scrollX.png)
上图是分别在X轴上滚动100和-100的示意图。

![ScrollY](http://7xs2qy.com1.z0.glb.clouddn.com/scrollY.png)
上图是分别在Y轴上滚动100和-100的示意图。

![ScrollXY](http://7xs2qy.com1.z0.glb.clouddn.com/ScrollXY.png)
上图是分别在X轴和Y轴上滚动100和-100的示意图。

通过上面几个示意图，应该对`mScrollX`和`mScrollY`属性有一定认识了。

### `OverScroller`类三个滚动相关的方法
下面看下如何通过`OverScroller`类实现View内容的弹性滑动。

上面介绍的scrollTo方法会瞬间把View内容滑动到指定位置，没有任何动画效果，显得比较突兀。但是我们可以通过`OverScroller`（OverScroller是Scroller的超集，在Scroller的基础上添加了对OverScroll的支持）实现弹性滑动。典型的实现如下所示：
``` java
OverScroller mScroller = new OverScroller(mContext);

public void smoothScrollTo(int destX,int destY,int duration){
    //计算出需要滚动的距离
    int deltaX = destX - getScrollX();
    int deltaY = destY - getScrollY();
    //开始滚动（表示在指定时间内，滚动指定的偏移量。）
    mScroller.startScroll(getScrollX(),getScrollY(),deltaX,deltaY,duration);
    invalidate();
}

//需要在自定义View中实现该方法
@Override
public void computeScroll(){
    if(mScroller.computeScrollOffset()){ //判断滚动是否结束
        //真正的滚动操作
        scrollTo(mScroller.getCurrX(),mScroller.getCurrY());
        postInvalidate();
    }
}
```

整个弹性滚动是靠着View绘制来驱动的，当调用`OverScroller.startScroll()`方法后，会触发View重绘，而在`View.draw`方法中就会调用上面的`computeScroll`方法。正如上述代码所示，在computeScroll方法中，获取当前的scrollX和scrollY，然后通过scrollTo来实现滚动，接着又通过`postInvalidate`方法触发下一次重绘，如此反复，直到整个滚动过程结束。

---

仔细看下`OverScroller`的API，就会发现除了可以通过`startScroll`方法实现弹性滚动外，还有两个特殊的API：`fling`和`springBack`，它们可以分别实现Fling效果和回弹效果。

所谓Fling效果，就是当我们快速滑动列表时，手指离开屏幕后，列表会以当前的滚动速度，以一个摩擦系数，做匀减速运动，直到滚动速度为0为止。
看下fling的方法签名：
``` java
/**
startX和startY表示初始的滚动值，一般设置为当前的mScrollX和mScrollY即可
velocityX和velocityY分别表示X轴和Y轴上的速度，单位：像素/秒。
minX和maxX表示在X轴上最小和最大的滚动值，即最终的mScrollX值要在这个区间之内。
minY和maxY表示在Y轴上最小和最大的滚动值，即最终的mScrollY值要在这个区间之内。
overX表示在X轴上可以过度滚动的长度，也就是说，若mScrollX达到了[minX,maxX]的边界值，但是滚动速度还不为0，那么可以在边界值的基础上再滚动overX距离，但是最终还是会回弹到[minX,maxX]区间，即会有一个回弹的效果。
overY表示在Y轴上可以过度滚动的长度，也就是说，若mScrollY达到了[minY,maxY]的边界值，但是滚动速度还不为0，那么可以在边界值的基础上再滚动overY距离，但是最终还是会回弹到[minY,maxY]区间，即会有一个回弹的效果。
*/
public void fling(int startX, int startY, int velocityX, int velocityY,int minX, int maxX, int minY, int maxY, int overX, int overY);
```
`fling`方法的参数含义如上，下面看一个具体的案例：

以5000/S的速度在Y轴上进行fling，最终停止在[0,300]这个mScrollY区间，过度滚动的距离overY为300。
核心代码如下所示：
``` java
mScroller.fling(getScrollX(), getScrollY(), 0, 5000, 0, 0, 0, 300, 0, 300);
myView.invalidate();
```
具体的效果如下所示：
<img src="http://7xs2qy.com1.z0.glb.clouddn.com/fling%E6%95%88%E6%9E%9C.gif" width="300" alt="fling效果图"/>

从上图可以看到，因为速度过大，当mScrollY达到边界300时，又滚动了一段距离（overY），但是最终又回弹到了边界值。

因此，我们可以通过`OverScroller.fling`方法模拟这种带有回弹效果的fling过程。

---

最后一个是用于实现回弹效果的`springBack`方法。
首先看下方法签名:
``` java
/**
startX和startY表示初始的滚动值，一般设置为当前的mScrollX和mScrollY即可
minX和maxX表示在X轴上最小和最大的滚动值，即最终的mScrollX值要在这个区间之内。
minY和maxY表示在Y轴上最小和最大的滚动值，即最终的mScrollY值要在这个区间之内。
minX，maxX，minY，maxY这4个滚动值构成了一个矩形区域。表示回弹后的（mScrollX,mScrollY)要在这个矩形区域内。

返回值若为true，则表示初始的startX和startY还不在最终的矩形区间内，需要进行回弹；若为false，则表示初始的startX和startY已经在最终的矩形区间内，不需要进行回弹了。
*/
public boolean springBack(int startX, int startY, int minX, int maxX, int minY, int maxY);
```
`springBack`方法的参数含义如上，下面看一个具体的案例：
首先把View滚动到(0,-500)位置，然后把mScrollY回弹到[0,100]区间，核心代码如下所示：
``` java
//首先在2S内把mScrollX和mScrollY滚动到（0，-500）
myView.smoothScrollTo(0,-500,2000)

//在当前的位置上进行回弹，目标是mScrollX为0，mScrollY落在[0,100]区间内
if (mScroller.springBack(getScrollX(), getScrollY(), 0, 0, 0, 100)) {
    myView.invalidate();
}
```
具体的效果如下所示：
<img src="http://7xs2qy.com1.z0.glb.clouddn.com/springBack.gif" width="300" alt="springBack效果图"/>

从上图可以看出，mScrollX回弹到0就结束了（只要落在[minY,maxY]区间内就可以了）。

因此，通过`OverScroller.springBack`方法，我们可以轻而易举的把View的内容回弹到我们期望的位置。

OK，至此``OverScroller`类三个滚动相关的方法都介绍完了。它们有一个共同点：这三个函数都仅是初始化函数，都是借助View的不断重绘，通过`View.computeScroll`方法和`OverScroller.computeScrollOffset`方法，把每一帧的scrollX和scrollY值，设置到View上，达到滚动View内容的目的。

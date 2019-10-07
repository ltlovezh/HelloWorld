---
title: Android CoordinatorLayout和Behavior
date: 2016-08-07 16:26:20
tags:
 - CoordinatorLayout
 - Behavior
 - 嵌套滚动机制
categories:
 - Android
---
本篇文章已授权微信公众号 guolin_blog （郭霖）独家发布

上篇文章简要介绍了一些[Android Material Design组件](http://ltlovezh.com/2016/08/05/Android-Material-Design/)，其中最重要的就是CoordinatorLayout了。本文将介绍下CoordinatorLayout是如何协调子View间关系的。

在介绍CoordinatorLayout之前，首先需要了解下嵌套滚动机制（NestedScrolling）。

<!-- more -->

## 嵌套滚动机制
所谓嵌套滚动其实就是界面布局中包含一个可滚动的列表和一个不可滚动的View，这样在滚动列表时，首先将不可滚动View移出屏幕或移进屏幕，待不可滚动View固定时，才会继续滚动滚动列表的内容。具体效果可参见上一篇文章的CoordinatorLayout效果图。

为什么会有嵌套滚动机制那？
之前我们处理Touch事件时，主要通过重写View的`dispatchTouchEvent`、`onInterceptTouchEvent`和`onTouchEvent`等方法处理滚动事件。但这种事件处理方式有一个痛点：

>Touch事件要么被父View处理，要么被子View处理，很难在两者之间协调处理。

也就是说：一旦子View决定处理Touch事件，那么事件就会一直下发到子View，即使子View不想处理中间的某个Touch事件（返回false），那么父View也没办法接着处理这个Touch事件了；除非父View拦截中间的某个Touch事件自己处理，但是一旦拦截了Touch事件，那么后续的Touch事件将永远不会下发到子View了。

针对这种问题，Android提供了NestedScrolling机制，实现嵌套滚动机制主要依赖四个类：

1. `NestedScrollingChild`
2. `NestedScrollingParent`
3. `NestedScrollingChildHelper`
4. `NestedScrollingParentHelper`

一般情况下，滚动列表需要实现NestedScrollingChild接口，以支持将滚动事件分发给父ViewGroup，相应的，父ViewGroup需要实现NestedScrollingParent接口，以支持将滚动事件进一步的分发给各个子View；而下面的两个类则是进行嵌套滚动的辅助类。

一般实现`NestedScrollingChild`接口的滚动列表会把滚动事件委托给NestedScrollingChildHelper辅助类来处理。例如：RecyclerView实现了NestedScrollingChild接口，它内部就会把滚动相关事件委托给NestedScrollingChildHelper对象来处理，如下所示：

``` java
@Override
public boolean startNestedScroll(int axes) {
    return getScrollingChildHelper().startNestedScroll(axes);
}

@Override
public void stopNestedScroll() {
    getScrollingChildHelper().stopNestedScroll();
}
    
@Override
public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed, int dxUnconsumed,int dyUnconsumed, int[] offsetInWindow) {
    return getScrollingChildHelper().dispatchNestedScroll(dxConsumed, dyConsumed,dxUnconsumed, dyUnconsumed, offsetInWindow);
}

@Override
public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow) {
    return getScrollingChildHelper().dispatchNestedPreScroll(dx, dy, consumed, offsetInWindow);
}

//...

```
NestedScrollingChild的方法有很多，更多的可参见源码，上述摘录的是和嵌套滚动机制相关的四个方法：

当我们滚动RecyclerView时，RecyclerView首先会通过startNestedScroll方法通知父ViewGroup（“我马上要滚动了，是否有兄弟节点要一起滚动？”），父ViewGroup会进一步把滚动事件分发给所有子View（实际是分发给和子View绑定的Behavior），感兴趣的子View会特别关注，即Behavior.onStartNestedScroll方法返回true。

针对这个流程，我们看下代码上的实现，RecyclerView会在Down事件时调用`startNestedScroll`方法，我们看下**NestedScrollingChildHelper.startNestedScroll方法的实现**：

``` java
public boolean startNestedScroll(int axes) {
    if (hasNestedScrollingParent()) {
        // Already in progress
        return true;
    }
    if (isNestedScrollingEnabled()) {
        ViewParent p = mView.getParent();
        View child = mView;
        //该循环主要是寻找到能够协调处理滚动事件的父View，即实现NestedScrollingParent接口的父ViewGroup
        while (p != null) {
            if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes)) {
                //记录协调处理滚动事件的父View
                mNestedScrollingParent = p;
                //ViewParentCompat是一个和父ViewGroup交互的兼容类，如果在Android5.0以上，就用View自带的方法，否则若实现了NestedScrollingParent接口，则调用接口方法。
                ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes);
                return true;
            }
            if (p instanceof View) {
                child = (View) p;
            }
            p = p.getParent();
        }
    }
    return false;
}
```

上述方法会找到能够协调处理滚动事件的父ViewGroup，然后调用它的onStartNestedScroll方法，因为CoordinatorLayout实现了NestedScrollingParent接口，所以我们看下**CoordinatorLayout.onStartNestedScroll**方法：

``` java
public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
    boolean handled = false;
    final int childCount = getChildCount();
    //询问每一个子View是否对滚动列表的滚动事件感兴趣？
    for (int i = 0; i < childCount; i++) {
        final View view = getChildAt(i);
        final LayoutParams lp = (LayoutParams) view.getLayoutParams();
        //获取和子View绑定的Behavior
        final Behavior viewBehavior = lp.getBehavior();
        if (viewBehavior != null) {
            final boolean accepted = viewBehavior.onStartNestedScroll(this, view, child, target,nestedScrollAxes);
            handled |= accepted;
            //做一下标注，作为判断后续是否接收滚动事件的标记
            lp.acceptNestedScroll(accepted);
        } else {
            lp.acceptNestedScroll(false);
        }
    }
    return handled;
}
```

上述方法会遍历每一个子View，询问它们是否对滚动列表的滚动事件感兴趣，若`Behavior.onStartNestedScroll`方法返回true，则表示感兴趣，那么滚动列表后续的滚动事件都会分发到该子View的Behavior。而把Behavior绑定到View的方法有两种：

1. 在布局文件中通过`app:layout_behavior`属性指定
2. 在自定义View中通过`@CoordinatorLayout.DefaultBehavior`注解指定，就像AppBarLayout那样。

>因此，我们可以在自定义的`Behavior.onStartNestedScroll`方法中根据实际情况决定是否对滚动事件感兴趣。

OK，假设CoordinatorLayout的某个子View对RecyclerView的滚动事件感兴趣，接下来RecyclerView就会把用户的滚动事件源源不断的分发给之前找到的父ViewGroup，然后父ViewGroup则进一步分发给感兴趣的子View。等到感兴趣的子View处理完滚动事件后，若用户的滚动距离没有被消费完，那么RecyclerView才有机会处理滚动事件，例如：用户一次性滚动了10px，其中某个View消费了8px，那么RecyclerView就只能滚动2px了。

针对这个流程，我们看下代码实现，RecyclerView会在Move事件时，计算出滚动距离，然后通过`dispatchNestedPreScroll`方法进行分发，我们看下NestedScrollingChildHelper.dispatchNestedPreScroll方法的实现：

``` java
public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow) {
    //判断之前是否找到协同处理的父ViewGroup
    if (isNestedScrollingEnabled() && mNestedScrollingParent != null) {
        //dx和dy分别表示X和Y轴上的滚动距离
        if (dx != 0 || dy != 0) { 
            int startX = 0;
            int startY = 0;
            //offsetInWindow用于计算滚动前后，滚动列表本身的偏移量
            if (offsetInWindow != null) {
                mView.getLocationInWindow(offsetInWindow);
                startX = offsetInWindow[0];
                startY = offsetInWindow[1];
            }

            if (consumed == null) {
                if (mTempNestedScrollConsumed == null) {
                    mTempNestedScrollConsumed = new int[2];
                }
                consumed = mTempNestedScrollConsumed;
            }
            consumed[0] = 0;
            consumed[1] = 0;
            //分发给父ViewGroup
            ViewParentCompat.onNestedPreScroll(mNestedScrollingParent, mView, dx, dy, consumed);
            if (offsetInWindow != null) {
                //计算出滚动列表本身的偏移量
                mView.getLocationInWindow(offsetInWindow);
                offsetInWindow[0] -= startX;
                offsetInWindow[1] -= startY;
            }
            return consumed[0] != 0 || consumed[1] != 0;
        } else if (offsetInWindow != null) {
            offsetInWindow[0] = 0;
            offsetInWindow[1] = 0;
        }
    }
    return false;
}
```
该方法的第3个参数是一个长度为2的一维数组，用于记录父ViewGroup（其实是父ViewGroup的子View）消费的滚动长度，若滚动距离没有用完，则滚动列表处理剩下的滚动距离；第4个参数也是一个长度为2的一维数组，用于记录滚动列表本身的偏移量，该参数用于修复用户Touch事件的坐标，以保证下一次滚动距离的正确性。这些处理逻辑可参见RecyclerView对Move事件的代码，此处不再贴代码了。

然后，父ViewGroup就会把滚动事件分发给感兴趣的子View，因为CoordinatorLayout实现了NestedScrollingParent接口，所以我们看下**CoordinatorLayout.onNestedPreScroll**方法：

``` java
public void onNestedPreScroll(View target, int dx, int dy, int[] consumed) {
    int xConsumed = 0;
    int yConsumed = 0;
    boolean accepted = false;
    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        final View view = getChildAt(i);
        final LayoutParams lp = (LayoutParams)view.getLayoutParams();
        //若子View对滚动事件不感兴趣，则直接跳过
        if (!lp.isNestedScrollAccepted()) {
            continue;
        }

        final Behavior viewBehavior = lp.getBehavior();
        if (viewBehavior != null) {
            mTempIntPair[0] = mTempIntPair[1] = 0;
            //分发给每个子View的Behavior处理
            viewBehavior.onNestedPreScroll(this, view,target, dx, dy, mTempIntPair);
            //找出每个子View消费的最大滚动距离就是父ViewGroup消费的滚动距离
            xConsumed = dx > 0 ? Math.max(xConsumed, mTempIntPair[0]): Math.min(xConsumed, mTempIntPair[0]);
            yConsumed = dy > 0 ? Math.max(yConsumed, mTempIntPair[1]): Math.min(yConsumed, mTempIntPair[1]);
            accepted = true;
        }
    }
    //记录父ViewGroup消费的滚动距离
    consumed[0] = xConsumed;
    consumed[1] = yConsumed;
    if (accepted) {
        //处理子View之间的依赖关系
        dispatchOnDependentViewChanged(true);
    }
}
```
CoordinatorLayout的处理很简单，把滚动事件分发给各个子View的`Behavior.onNestedPreScroll`方法处理，并计算出最终消费的滚动距离。

>因此，我们可以在自定义的`Behavior.onNestedPreScroll`方法中处理子View的滚动事件，然后根据实际情况填写消费的滚动距离。

OK，假设RecyclerView的滚动距离没有被CoordinatorLayout消费完，那么接下来RecyclerView应该处理这些滚动事件了。在RecyclerView的onTouchEvent方法中会调用scrollByInternal处理内容滚动，关键代码如下所示：

``` java
//x表示X轴上剩余的滚动距离
if (x != 0) {
    //交给具体的LayoutManager处理滚动事件，并且记录下消费的和剩余的滚动量
    consumedX = mLayout.scrollHorizontallyBy(x, mRecycler,mState);
    unconsumedX = x - consumedX;
}
//y表示Y轴上剩余的滚动距离
if (y != 0) {
    //交给具体的LayoutManager处理滚动事件，并且记录下消费的和剩余的滚动量
    consumedY = mLayout.scrollVerticallyBy(y, mRecycler, mState);
    unconsumedY = y - consumedY;
}

//...
//分发滚动列表本身对剩余滚动量的消费情况
dispatchNestedScroll(consumedX, consumedY, unconsumedX, unconsumedY, mScrollOffset);
```

如上所示，RecyclerView通过LayoutManager处理了剩余的滚动距离，然后计算出对剩余滚动量的消费情况，通过`dispatchNestedScroll`方法继续分发给CoordinatorLayout，而CoordinatorLayout则通过`onNestedScroll`方法分发给感兴趣的子View的Behavior处理。这部分的代码逻辑和onNestedPreScroll类似，就不贴出了，感兴趣的可以直接看源码。

>因此，我们可以在自定义的`Behavior.onNestedScroll`方法中检测到滚动距离的最终消费情况。

OK，现在假设用户结束滚动操作了，即应该结束一系列的滚动事件了，RecyclerView会在UP事件中调用`stopNestedScroll`方法，该方法和上面介绍的三个方法类似，都会先把事件分发给父ViewGroup，然后父ViewGroup再把事件分到各个子View，最终触发子View的`Behavior.onStopNestedScroll`方法，感兴趣可以可接看源码，此处不再贴出。

>因此，我们可以在自定义的`Behavior.onStopNestedScroll`方法中检测到滚动事件的结束。

OK，整个嵌套滚动机制就介绍完了，可见跟我们直接打交道的就是`CoordinatorLayout.Behavior`类了，通过重写该类中的方法，我们不仅可以监听滚动列表的滚动事件，还可以做很多其他的事情，感兴趣的可以详细看下Behavior接口中的方法说明。

## 监听View之间的状态变化

这里我比较感兴趣的是通过Behavior监听View之间的状态变化，例如：位置、大小、背景色等，要实现这种状态监听，需要重写Behavior的两个方法：

1. `layoutDependsOn` ： 决定需要监听的View对象
2. `onDependentViewChanged` ： 当监听对象的状态发生变化时，会回调该方法，我们可以根据监听对象的状态，变换自己View的状态。

首先，我们来看下这两个方法在CoordinatorLayout中是怎么被调用的？

经过代码摸索和测试，发现这两个方法基本都是在`CoordinatorLayout.dispatchOnDependentViewChanged`方法中被调用的，而该方法则会在CoordinatorLayout每次绘制之前被调用，核心代码如下所示：
``` java
//View绘制前的全局监听器
class OnPreDrawListener implements ViewTreeObserver.OnPreDrawListener {
        @Override
        public boolean onPreDraw() {
            //触发关联View的回调
            dispatchOnDependentViewChanged(false);
            return true;
        }
    }

//当CoordinatorLayout绑定到窗口时，就会被注册
public void onAttachedToWindow() {
    super.onAttachedToWindow();
    resetTouchBehaviors();
    if (mNeedsPreDrawListener) {
        if (mOnPreDrawListener == null) {
            mOnPreDrawListener = new OnPreDrawListener();
        }
        final ViewTreeObserver vto = getViewTreeObserver();
        //注册每次绘制前的回调
        vto.addOnPreDrawListener(mOnPreDrawListener);
    }
    //...    
    mIsAttachedToWindow = true;
}
```
从上述代码可知，每次重绘CoordinatorLayout之前，都会调用`dispatchOnDependentViewChanged`方法，好吧，该方法是核心部分，来看下代码：
``` java
void dispatchOnDependentViewChanged(final boolean fromNestedScroll) {
//布局方向    
final int layoutDirection = ViewCompat.getLayoutDirection(this);
//mDependencySortedChildren是根据View之间的依赖关系，重新排序后的子View列表，假如A依赖于B，那么B就在A的前面，可参见prepareChildren方法
final int childCount = mDependencySortedChildren.size();
for (int i = 0; i < childCount; i++) {
    final View child = mDependencySortedChildren.get(i);
    final LayoutParams lp = (LayoutParams)child.getLayoutParams();
    //向前寻找当前View依赖的View，这里的依赖关系是通过app:layout_anchor属性指定的
    for (int j = 0; j < i; j++) {
        final View checkChild = mDependencySortedChildren.get(j);
        //lp.mAnchorDirectChild表示当前View所依赖View的父View，并且为CoordinatorLayout直接子View，因为checkChild是直接子View，只有这样他们才可以在一起比较是否相等
        if (lp.mAnchorDirectChild == checkChild) {
            //找到了当前View依赖的View，offsetChildToAnchor方法会根据当前View的位置和所依赖View的位置，计算出当前View期望的新位置，然后对当前View进行位移，同时调用onDependentViewChanged方法。感兴趣可直接查阅该方法源码，此处不再赘述
            offsetChildToAnchor(child, layoutDirection);
            }
        }

    // 这里oldRect表示当前View上一次的位置，newRect表示当前View现在的位置，若两次位置相同，说明没有发生偏移，则所有依赖于该View的子View都将收不到onDependentViewChanged回调（感觉这里应该是为了效率考虑吧）
    final Rect oldRect = mTempRect1;
    final Rect newRect = mTempRect2;
    getLastChildRect(child, oldRect);
    getChildRect(child, true, newRect);
    if (oldRect.equals(newRect)) {
        continue;
    }
    //重新记录当前View的位置
    recordLastChildRect(child, newRect);
    // Update any behavior-dependent views for the change
    for (int j = i + 1; j < childCount; j++) {
        final View checkChild = mDependencySortedChildren.get(j);
        final LayoutParams checkLp = (LayoutParams) checkChild.getLayoutParams();
        //找出View的Behavior
        final Behavior b = checkLp.getBehavior();
        //通过layoutDependsOn判断checkChild是否依赖于当前View
        if (b != null && b.layoutDependsOn(this, checkChild, child)) {
            //...省略代码
            //若依赖于当前View，那么调用onDependentViewChanged方法
            final boolean handled = b.onDependentViewChanged(this, checkChild, child);
            //...省略代码
        }
    }
}
```

如上所示，核心代码都添加了详细的注释，这里简单总结下：

1. 形成依赖关系的方法有两种
    >1. 通过`app:layout_anchor`属性指定参照的View；
    >2. 通过`layoutDependsOn`方法判断

2. 若是通过第二种方式形成的依赖关系，那么只有当被依赖View的Rect区域发生变化时，所有依赖于该View的其他View才会收到`onDependentViewChanged`回调。
3. 若在`Behavior.onDependentViewChanged`方法中根据所依赖View的状态修改了当前View的位置，那么也应该重写Behavior的`onLayoutChild`，这样才能保持一致。

OK，这两个方法的实现原理已经介绍完了。

## 实际案例

下面我们来看一个同时包含嵌套滚动和View间状态监听的Demo。

首先看一下效果图：
![](http://7xs2qy.com1.z0.glb.clouddn.com/Behavior.gif)

当向上滚动TextView时，首先会把TextView滚动出屏幕，然后才会滚动RecyclerView的内容；当向下滚动时，首先会把TextView滚动到屏幕内，然后才会滚动RecyclerView的内容；同时TextView的位置依赖于Button的位置，RecyclerView的位置依赖于TextView的位置（保证RecyclerView不会被TextView遮盖住）。

实现上述效果的布局文按如下所示：
``` XML
android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/activity_behavior"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:layout_width="match_parent"
        android:layout_height="150dp"
        android:background="#4400ff00"
        android:gravity="center"
        android:text="Hello CoordinatorLayout"
        android:textSize="18dp"
        app:layout_behavior="leon.com.allkindsoflistview.MyBehavior" />

    <android.support.v7.widget.RecyclerView
        android:id="@+id/recycler"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="leon.com.allkindsoflistview.RecyclerBehavior" />

    <Button
        android:id="@+id/button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="bottom|right"
        android:text="Click Me" />

</android.support.design.widget.CoordinatorLayout>
```
CoordinatorLayout包含3个子View，其中RecyclerView依赖于TextView，TextView依赖于Button（通过Behavior.layoutDependsOn方法指定），RecyclerView的滚动会带动TextView的滚动。

下面来看一下RecyclerView的Behavior，该Behavior仅仅实现了layoutDependsOn和onDependentViewChanged方法，目的是根据TextView的位置，计算出RecyclerView的位置，这样才能保证RecyclerView的顶部靠着TextView的底部，而不被TextView盖住。代码如下所示：
``` java
 @Override
public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
    //当前RecyclerView依赖于TextView
    return dependency instanceof TextView;
}

@Override
public boolean onDependentViewChanged(CoordinatorLayout parent, View child, View dependency) {
    //根据TextView的位置，计算出RecyclerView的位置，这样才能保证RecyclerView的顶部靠着TextView的底部，而不被TextView盖住
    int delta = (int) dependency.getTranslationY() + dependency.getBottom();
    delta = delta - child.getTop();
    child.offsetTopAndBottom(delta);
    return true;
}
```

然后来看一下TextView的MyBehavior，该Behavior不仅仅实现了依赖关系，同时还实现了嵌套滚动，代码如下所示：

``` java
 @Override
public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout, View child, View directTargetChild, View target, int nestedScrollAxes) {
    //若是RecyclerView滚动，那么TextView则跟着滚动，即TextView对RecyclerView的滚动感兴趣
    return target instanceof RecyclerView;
}

 @Override
public void onNestedPreScroll(CoordinatorLayout coordinatorLayout, View child, View target, int dx, int dy, int[] consumed) {
    //通过调整TextView的TranslationY值来达到滚动的目的
    if (dy > 0) { //表示向上滚动
        if (child.getTranslationY() > -child.getHeight()) {
            float trY = child.getTranslationY() - dy <= -child.getHeight() ? -child.getHeight() : child.getTranslationY() - dy;
            consumed[1] = (int) (child.getTranslationY() - trY);
            child.setTranslationY(trY);
    }else if (dy < 0){ //向下滚动
        if (child.getTranslationY() < 0) {
            float trY = child.getTranslationY() - dy >= 0 ? 0 : child.getTranslationY() - dy;
            consumed[1] = (int) (child.getTranslationY() - trY);
            child.setTranslationY(trY);
        }
    }
}

@Override
public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
    //表示TextView依赖于Button
    return dependency instanceof Button;
}

@Override
public boolean onDependentViewChanged(CoordinatorLayout parent, View child, View dependency) {
    //根据Button的TranslationY调整TextView的TranslationY
     child.setTranslationY(Math.abs(dependency.getTranslationY()));
     return true;
}
```
TextView的MyBehavior的稍微复杂一些，主要是实现了跟着RecyclerView的滚动而滚动，同时又依赖于Button的位置决定TextView的最终位置。

OK，到此为止，简要介绍了嵌套滚动机制和CoordinatorLayout.Behavior的使用方法，Behavior的方法还有很多，感兴趣的可以多尝试下。

## 参考文章

1. [Android嵌套滑动机制（NestedScrolling）](https://segmentfault.com/a/1190000002873657)
2. [探究Behavior的真实面目](http://www.tuicool.com/articles/uU7vqya)
3. [Android Support Design中CoordinatorLayout与Behaviors 初探](https://segmentfault.com/a/1190000002888109)

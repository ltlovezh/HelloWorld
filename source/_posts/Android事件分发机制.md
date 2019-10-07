---
title: Android事件分发机制
date: 2016-05-09 13:29:51
tags:
- Android
- Android事件
categories:
- Android
---
最近做需求时，对Android的事件分发机制产生了一些困惑，所以周末详细学习了一下，现总结如下：
## 基础知识
1、所有的Touch事件都会被封装成`MotionEvent`对象，该对象代表了Touch事件的坐标、动作等基本信息。
2、一般情况下，每一个Touch事件，总是以`ACTION_DOWN`事件开始，中间穿插着一些`ACTION_MOVE`事件（取决于是否有手势的移动），然后以`ACTION_UP`事件结束，之后才会有`Click`、`LongClick`等事件。
3、事件分发过程中，包括对MotionEvent事件的三种处理操作：
1. 分发操作：`dispatchTouchEvent方法`
2. 拦截操作：`onInterceptTouchEvent方法`
3. 消费操作：`onTouchEvent方法`和`OnTouchListener的onTouch方法`，其中`onTouch`的优先级高于`onTouchEvent`，若onTouch返回true，那么就不会调用onTouchEvent方法。

4、一般情况下，处理View事件时，我们会给View注册OnTouchListener、OnClickListener等事件监听器，我们注意到onTouch回调方法会有布尔返回值，而onClick回调方法则返回void。经测试发现，如果onTouch返回false，那么才会回调onClick方法，若返回true，则不会回调Click事件。

好了，熟悉这些基本用法，应付一般的Coding就没问题了。不过作为有追求的猿们，我们要从源码层面揭开事件分发机制的神秘面纱。

<!-- more -->

## 事件分发层级关系
Touch事件首先会被传递给当前的Activity，接着由Activity向下传递给ViewGroup1，然后是ViewGroup2， ... ，最后传递给目标View。若目标View没有消费该Touch事件，那么事件又会向上回传，直到某个ViewGroup消费了该事件，或者没有ViewGroup消费事件，那么最终Touch事件会回到当前Activity，触发Activity的onTouchEvent方法。

在一系列Touch事件中，`ACTION_DOWN`是比较特殊的一个，若`ACTION_DOWN`事件没有被消费，则说明Touch事件没有找到目标View，那么后续的Touch事件不会继续下发，在Activity层就停止了。如下所示：
`ACTION_DOWN`没有被目标View消费的情况下：
![目标View没有消费ACTION_DOWN事件](http://7xs2qy.com1.z0.glb.clouddn.com/View%E6%B2%A1%E6%9C%89%E6%B6%88%E8%B4%B9Down%E4%BA%8B%E4%BB%B6.png)

`ACTION_DOWN`被目标View消费的情况下：
![目标View消费ACTION_DOWN事件](http://7xs2qy.com1.z0.glb.clouddn.com/View%E6%B6%88%E8%B4%B9Down%E4%BA%8B%E4%BB%B6.png)
### Activity层的事件分发
下面，我们直接从Activity层入手，分析事件传递，至于事件是如何产生的，以及事件是如何传递到Activity层的，可以参考[这里](http://feelyou.info/analyze_android_touch_event/)。
`Activity.dispatchTouchEvent`方法比较简单，如下所示:
``` java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();//系统留的回调接口
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```
由代码可知，这里主要是去调用了`PhoneWindow.superDispatchTouchEvent`方法，而该方法也很简单，直接抛给了`DecorView.superDispatchTouchEvent`处理，而`DecorView`则直接调用了`super.dispatchTouchEvent`去处理，我们知道`DecorView`继承自`FrameLayout`，所以实际最后交给了`ViewGroup.dispatchTouchEvent`处理。至于`ViewGroup.dispatchTouchEvent`，下面会进行详细的分析。
从上面的代码，还可以看出，只有DecorView无法消费Touch事件时，才会调用`Activity.onTouchEvent`。也即我们上面讲的，当子View无法消费Touch事件时，最后会回传给`Activity.onTouchEvent`，而`Activity.onTouchEvent`默认是返回false的。
### ViewGroup层的事件分发
从上面可知，Activity层的事件分发实际是交给了`ViewGroup.dispatchTouchEvent`来处理，而该方法也是Android事件分发的核心，主要代码如下所示，一些关键部分添加了注释：
``` java
public boolean dispatchTouchEvent(MotionEvent ev) {
    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;
        //如果是ACTION_DOWN事件，那么需要清除以前所有的状态，开始一个新的touch事件流程（因为touch事件就是从ACTION_DOWN开始的），最重要的是把mFirstTouchTarget（可以简单理解为消费touch事件的View）赋值null，以开始一个新的周期。
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }

        //检查是否需要拦截事件，disallowIntercept表示是否允许拦截，只有允许拦截，我们重写onInterceptTouchEvent才有意义。然后才是根据onInterceptTouchEvent判断是否需要拦截，true表示拦截touch事件，这样子View就收不到相应事件，而被当前ViewGroup处理掉。默认为false，不拦截事件。
        final boolean intercepted;
        //若是初始的ACTION_DOWN事件或者已经找到能够消费touch事件的View，才进行判断
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // restore action in case it was changed
            } else {
                intercepted = false;
            }
        } else {
        //如果没有消费事件的target View，而且又不是初始的ACTION_DOWN事件，那么就强制拦截事件，也就表示事件不会继续向下传递了。
            intercepted = true;
        }

        // 检查是否是ACTION_CANCEL事件
        final boolean canceled = resetCancelNextUpFlag(this)
                || actionMasked == MotionEvent.ACTION_CANCEL;

        // Update list of touch targets for pointer down, if needed.
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
        if (!canceled && !intercepted) {//非取消事件 && 不需要拦截事件
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {//ACTION_DOWN事件
                final int actionIndex = ev.getActionIndex(); // always 0 for down
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                        : TouchTarget.ALL_POINTER_IDS;

                final int childrenCount = mChildrenCount;
                if (childrenCount != 0) {
                    // Find a child that can receive the event.
                    // Scan children from front to back.
                    final View[] children = mChildren;//子View列表
                    final float x = ev.getX(actionIndex);//事件坐标
                    final float y = ev.getY(actionIndex);
                    //然后从ViewGroup中找出能够消费touch事件的子View
                    final boolean customOrder = isChildrenDrawingOrderEnabled();
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        final int childIndex = customOrder ?
                                getChildDrawingOrder(childrenCount, i) : i;
                        final View child = children[childIndex];
                        //canViewReceivePointerEvents表示是否能够接收事件（条件是VISIBLE的或者存在和该View绑定的动画）
                        //isTransformedTouchPointInView表示MotionEvent的坐标是否在该View的坐标范围内
                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            continue;
                        }

                        newTouchTarget = getTouchTarget(child);//找到能够消费touch事件的子View
                       。。。。。。

                        //把事件交给子View处理，该方法是真正负责向子View分发事件的
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            //若子view消费了该touch事件，则保存相应的值，并且让该View成为touch事件的起点
                            // Child wants to receive touch within its bounds.
                            mLastTouchDownTime = ev.getDownTime();
                            mLastTouchDownIndex = childIndex;
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;//表示Touch事件已经被消费
                            break;
                        }
                    }
                }

                。。。。。。
            }
        }

        // 需要拦截touch事件，或者子View无法消费该touch事件，那么就作为普通的View处理，调用View.dispatchTouchEvent处理（间接触发onTouchEvent方法）
        if (mFirstTouchTarget == null) {
            // No touch targets so treat this as an ordinary view.
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            //否则，开始分发事件
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
                final TouchTarget next = target.next;
                //如果ACTION_DOWN事件已经被消费，那么直接返回true。
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    //判断是否需要拦截事件，这里主要处理没有拦截ACTION_DOWN事件，但是拦截了后续的ACTION_MOVE等事件的情况，这种情况下直接分发ACTION_CANCEL事件给子View
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
                    //交给子View去处理，可能是MOVE、UP事件，也可能是CANCEL事件
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    //若进行了拦截操作(即这里分发给子View的是CANCEL事件)，则下次不再对该子View下发事件。极端情况下，mFirstTouchTarget在这里就直接变成了null，导致后续的事件只能分发到该ViewGroup层。
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        target.recycle();
                        target = next;
                        continue;
                    }
                }
                predecessor = target;
                target = next;
            }
        }

           //如果是ACTION_CANCEL、MotionEvent.ACTION_UP事件，那么清除之前的touch状态，mFirstTouchTarget赋值null，表示后续的事件不会再继续往下传递了。
        if (canceled
                || actionMasked == MotionEvent.ACTION_UP
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            removePointersFromTouchTargets(idBitsToRemove);
        }
    }
    return handled;
}
```
`ViewGroup.dispatchTouchEvent`还是很复杂的，由代码可知，`mFirstTouchTarget`是目标节点列表的首节点，这里为了方便理解，可以把`mFirstTouchTarget`当做消费Touch事件的目标View。若没有找到目标View，那么`ViewGroup`就会被当做普通View来处理，调用`dispatchTransformedTouchEvent`处理。
若是找到了目标View，那么就会把事件分发给子View去处理，同样是调用`dispatchTransformedTouchEvent`。

`dispatchTransformedTouchEvent`的核心代码如下所示：
``` java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,View child, int desiredPointerIdBits) {
    final boolean handled;
    //若是ACTION_CANCEL事件，若是没有找到目标View，那么就当做普通View处理
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {//当做普通View来处理
            handled = super.dispatchTouchEvent(event);
        } else {//分发给子View ACTION_CANCEL事件
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }
    。。。。。。省略和下面较为相似的代码逻辑
    //重新计算事件坐标，交给子View去处理（child.dispatchTouchEvent），因为事件坐标是相对于本View的左上角的，所以要重新计算
    final MotionEvent transformedEvent;
    if (child == null) {
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
        if (! child.hasIdentityMatrix()) {
            transformedEvent.transform(child.getInverseMatrix());
        }
        //分发给子View去处理
        handled = child.dispatchTouchEvent(transformedEvent);
    }
    transformedEvent.recycle();
    return handled;
}
```
根据`ViewGroup.dispatchTouchEvent`和`ViewGroup.dispatchTransformedTouchEvent`方法的注释，我们可以得出：

1. 若是`ACTION_DOWN`事件，并且不会拦截事件，那么会尝试把事件传递给子View（ViewGroup）处理，如果子View消费了`ACTION_DOWN`事件，那么就找到了目标View，即`mFirstTouchTarget`不为`null`,那么接下来的`MOVE`、`UP`事件才会顺利的下发到子View。
2. 若是`ACTION_DOWN`事件，并且会拦截事件，那么就意味着`mFirstTouchTarget`为`null`，就会通过`dispatchTransformedTouchEvent`方法间接的调用`View.dispatchTouchEvent`,表示将该ViewGroup当做普通View处理。***同时，接下来的`MOVE`、`UP`事件，也不会向下传递给子View处理（因为没有找到目标View啊）***，而是直接在Activity层进行处理。`View.dispatchTouchEvent`会在下面进行详细的分析。

***

***下面假设处理`ACTION_DOWN`事件时，找到了目标View，也即`mFirstTouchTarget`不为`null`***。

1. 若是`ACTION_MOVE`、`ACTION_UP`事件，并且不会拦截事件，那么这些Touch事件都会顺利的下发到子View。

2. 若是`ACTION_MOVE`、`ACTION_UP`事件，并且会拦截事件，那么会给子View分发`ACTION_CANVEL`事件，并且会把`mFirstTouchTarget`赋值`null`，表示不会再继续向下分发后续Touch事件了，而是把该ViewGroup当做普通View处理。

所以，如果如果我们拦截了`ACTION_DOWN`事件，导致没有找到目标View，那么后续的move、up事件都无法下发，而是直接调用`Activity.onTouchEvent`（其实是尝试下发了，但是没有子View能够消费，导致DecorView的dispatchTouchEvent方法返回false，所以触发了Activity.onTouchEvent）。
### View（非ViewGroup）层的事件分发
从上面的代码分析，可知，最终，Touch事件要么分发给目标子View处理，要么被当前的ViewGroup消费掉。但不管哪种情况，都是调用`View.dispatchTouchEvent` 处理。但是这两种情况存在一个差异：***默认情况下，ViewGroup是不可点击的，而目标子View（例如：Button、TextView等）是可点击啊的***，这点对`View.dispatchTouchEvent`来说非常重要，下面详细分析该方法。
``` java
public boolean dispatchTouchEvent(MotionEvent event) {
   。。。。。。
    if (onFilterTouchEventForSecurity(event)) {
        ListenerInfo li = mListenerInfo;
        //若注册了OnTouchListener，并且View是enabled的，则回调onTouch方法
        if (li != null && li.mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            return true;
        }

        if (onTouchEvent(event)) {
            return true;
        }
    }
   。。。。。。
    return false;
}
```
相比于`ViewGroup.dispatchTouchEvent`，`View.dispatchTouchEvent`方法很简单，如果为该View注册了`OnTouchListener`监听器，并且该View是enabled的，那么就去回调`onTouch`方法，所以说`onTouch`的优先级比`onTouchEvent`高。
如果onTocuh返回true，那么dispatchTouchEvent直接返回true，onTouchEvent及接下来的onClick方法都不会执行，这和我们在文章开始时讲的是一致的。同时，也说明了onClick方法是在onTouchEvent中被调用的。

好的，接下来，看下`View.onTouchEvent`，该方法虽然很复杂，但是我们只看我们关心的部分，精简后的代码如下所示：
``` java
public boolean onTouchEvent(MotionEvent event) {
//判断是否是CLICKABLE，是否是LONG_CLICKABLE（注意ViewGroup和普通View的区别哈）
    if (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_UP:
                    。。。。。。
                    if (!mHasPerformedLongPress) {
                        // This is a tap, so remove the longpress check
                        removeLongPressCallback();
                        if (!focusTaken) {
                            // Use a Runnable and post this rather than calling
                            // performClick directly. This lets other visual state
                            // of the view update before click actions start.
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                //调用mOnClickListener.onClick(this);
                                performClick();
                            }
                        }
                    }
                    。。。。。。                   
                   }
                break;

            case MotionEvent.ACTION_DOWN:
                。。。。。。
                break;

            case MotionEvent.ACTION_CANCEL:
                。。。。。。
                break;

            case MotionEvent.ACTION_MOVE:
                。。。。。
                break;
        }
        return true;//注意，只要是CLICKABLE或者LONG_CLICKABLE，就返回true哈
    }
    return false;//否则就是返回false。
}
```
从代码中，可知，只要View是CLICKABLE或者LONG_CLICKABLE的，就会进入switch语句，并且不管是什么类型的MotionEvent，onTouchEvent都会返回true。那么，`View.dispatchTouchEvent`也会返回true，表示此事件被消费了。同时，在`ACTION_UP`里，会调用performClick方法，该方法就会回调我们注册的click事件，performClick方法很简单，此处不再详述。

所以如果目标View是不可点击的话（各种ViewGroup、ImageView等），那么就不能消费事件，就会造成后续的click事件无法触发，这点很重要哈。
## 总结

上面详细地分析了`Activity`、`ViewGroup`、`View`层的事件传递策略，其中最复杂也最核心的是`ViewGroup.dispatchTouchEvent`，需要好好理解和运用哈。

此外，`ACTION_DOWN`事件的特殊性也是值得我们关注的。

* 对于拦截函数`onInterceptTouchEvent`来说：
	1. 若是拦截了`ACTION_DOWN`事件，就意味着没有找到目标View，那么接下来的move、up事件都无法正常下发到子View，仅仅止步于Activity层。
	2. 若是没有拦截`ACTION_DOWN`事件，而是拦截了`ACTION_MOVE`事件，那么进行拦截的那层ViewGroup会向下分发`ACTION_CANCEL`事件，并且会将该层的`mFirstTouchTarget`置为null，那么后续的Touch事件只能分发到该ViewGroup层。并且如果不改变ViewGroup的默认可点击性的话，Touch事件永远不会在ViewGroup层被消费，最后Touch事件又会回传给Activity。

* 对于分发函数`dispatchTouchEvent`来说：
	1. 若是在X层对`ACTION_DOWN`事件，直接返回了true，那么该层就是目标View（`默认不可点击啊`），后续的move、up事件也会传递到该X层。只不过默认情况下，该目标View无法消费Touch事件，造成Touch事件又会回传给Activity。
	2. 若是`ACTION_DOWN`事件顺利地找到了能够消费Touch事件的子View，但是在X层对`ACTION_MOVE`事件，直接返回了true（仅仅是丢弃了move事件），那么并不影响后续的Touch事件，它们依然能够传递到目标View，依然能够被消费掉。

## 关于Android事件分发的一些好文章

1. [可能是讲解 Android 事件分发最好的文章](http://mp.weixin.qq.com/s?__biz=MzA3MDMyMjkzNg==&mid=2652261785&idx=1&sn=07372cc47f6bdcad6d945de4544cf455&scene=23&srcid=0709fhigrpOPM7m1fWReZ1mC#rd)
2. [图解 Android 事件分发机制](http://android.jobbole.com/83514/#comment-91758)

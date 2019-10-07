---
title: 你真的了解Android ViewGroup的draw和onDraw的调用时机吗
date: 2017-10-02 18:03:39
tags:
 - ViewGroup.onDraw
 - ViewGroup.draw
categories:
 - Android
---
前几天遇到一个ViewGroup.onDraw不会调用的问题，在网上查了一些资料，发现基本都混淆了`onDraw`和`draw`的区别，趁着十一假期有时间，简单梳理了下这里的逻辑。

<!-- more -->

## `View.draw`和`View.onDraw`的调用关系
首先，`View.draw`和`View.onDraw`是两个不同的方法，只有`View.draw`被调用，`View.onDraw`才有可能被调用。在`View.draw`中有下面一段代码：
``` java
final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);//是否是实心控件

if (!dirtyOpaque) {
    drawBackground(canvas);//绘制背景
}

...

// Step 3, draw the content
if (!dirtyOpaque) onDraw(canvas);//调用onDraw
```
通过上述代码可知：
1. `View.draw`方法中会调用`View.onDraw`
2. 只有`dirtyOpaque`为false（透明，非实心），才会调用`View.onDraw`方法。

因此，如果希望`ViewGroup.onDraw`方法被调用，那么就必须同时满足两个条件：
1. 设法让`ViewGroup.draw`方法被调用
2. 让`draw`方法中的`dirtyOpaque`为false。

---

既然谈到了`View.draw`和`View.onDraw`，这里简单说下两者的区别。查看View源码，可知`View.draw`基本包含6个步骤：
1. Draw the background，通过View.drawBackground方法来实现。
2. If necessary, save the canvas’ layers to prepare for fading，如果需要，保存画布层（Canvas.saveLayer）为淡入或淡出做准备。
3. draw the content，通过`View.onDraw`方法来实现，一般自定义View，就是通过该方法来绘制内容。获得Canvas后，可以draw任何内容，实现个性化的定制。
4. draw the children，通过View.dispatchDraw方法来实现，ViewGroup都会实现该方法，来绘制自己的子View。
5. If necessary, draw the fading edges and restore layers，如果需要，绘制淡入淡出的相关内容并恢复之前保存的画布层（layer）。
6. draw decorations (scrollbars)，通过View.onDrawScrollBars方法来实现，绘制滚动条的操作就是在这里实现的。

简单来说，`View.draw`负责绘制当前View的所有内容以及子View的内容，是一个全集。而`View.onDraw`则只负责绘制本身相关的内容，是一个子集。

## `ViewGroup.draw`的调用时机
其实也是`View.draw`的调用时机，通过查看View源码可知：单参数的`View.draw`方法会在三个参数的`View.draw`方法中被调用，如下所示：
``` java
if (!hasDisplayList) { //软件绘制
    // Fast path for layouts with no backgrounds
    if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {      
        //跳过当前View的绘制,直接绘制子view
        mPrivateFlags &= ~PFLAG_DIRTY_MASK;
        dispatchDraw(canvas);
    } else {                            
        //此时坐标系已经切换到View自身坐标系了,可以纯碎的绘制当前view了,又回到了draw(canvas)
        draw(canvas);
    }
}
```
在软件绘制下，三参数的`View.draw`负责把View坐标系从父View那里切换到当前View，然后再交给当前View去绘制。一般情况下，交给当前View去绘制就是通过调用单参数的`View.draw`方法来实现。
但是，这里有一个优化逻辑：如果当前View不需要绘制（打上了`PFLAG_SKIP_DRAW`标志），那么会通过`dispatchDraw`方法直接绘制当前View的子View。

所以，我们的`ViewGroup.draw`方法会不会被调用，完全取决于mPrivateFlags是不是包含`PFLAG_SKIP_DRAW`标志：
1. 若mPrivateFlags包含`PFLAG_SKIP_DRAW`，那么会跳过当前View的draw方法，直接调用dispatchDraw方法绘制当前View的子View。
2. 若mPrivateFlags不包含`PFLAG_SKIP_DRAW`，那么会调用当前View的draw方法，完成所有内容的绘制。

那么`PFLAG_SKIP_DRAW`取决于哪些因素那？

### setWillNotDraw
View中有一个`setWillNotDraw`方法，从注释上来看，就是控制是否要跳过`View.draw`方法，以进行优化的。我们看一下该方法：
``` java
public void setWillNotDraw(boolean willNotDraw) {
    setFlags(willNotDraw ? WILL_NOT_DRAW : 0, DRAW_MASK);
}
```
该方法很简单，我们继续看下setFlags方法：
``` java
void setFlags(int flags, int mask) {
int old = mViewFlags;
//设置flags
mViewFlags = (mViewFlags & ~mask) | (flags & mask);
int changed = mViewFlags ^ old;
//若mViewFlags前后没有变化，则直接返回
if (changed == 0) {
    return;
}
int privateFlags = mPrivateFlags;

...

if ((changed & DRAW_MASK) != 0) {
    if ((mViewFlags & WILL_NOT_DRAW) != 0) {
        //mViewFlags设置了WILL_NOT_DRAW标志
        if (mBseackground != null) {
            //如果当前View有背景，那么取消mPrivateFlags的PFLAG_SKIP_DRAW标志，但是设置另外一个PFLAG_ONLY_DRAWS_BACKGROUND标志
            mPrivateFlags &= ~PFLAG_SKIP_DRAW;
            mPrivateFlags |= PFLAG_ONLY_DRAWS_BACKGROUND;
        } else {
            //如果当前View没有背景，那么直接设置PrivateFlags的PFLAG_SKIP_DRAW标志
            mPrivateFlags |= PFLAG_SKIP_DRAW;
        }
    } else {
        //因为mViewFlags没有设置WILL_NOT_DRAW标志，所以取消mPrivateFlags的PFLAG_SKIP_DRAW标志
        mPrivateFlags &= ~PFLAG_SKIP_DRAW;
    }
    requestLayout();
    invalidate(true);
    }
}
```
通过上述代码可知，要想对mPrivateFlags设置`PFLAG_SKIP_DRAW`标识，必须满足两个条件：
1. 针对mViewFlags，设置WILL_NOT_DRAW标志
2. 当前View没有背景图

通过`setWillNotDraw(true)`一定会对mViewFlags设置`WILL_NOT_DRAW`标识。如果此时当前View没有背景图，那么就会对mPrivateFlags设置`PFLAG_SKIP_DRAW`标识。
但是若此时当前View有背景图，那么就会取消mPrivateFlags的`PFLAG_SKIP_DRAW`标识，同时设置另外一个`PFLAG_ONLY_DRAWS_BACKGROUND`标识。`setWillNotDraw`方法的相关逻辑如下图所示：
![setWillNotDraw](http://7xs2qy.com1.z0.glb.clouddn.com/setWillNotDraw.jpg)

### 设置背景
那这里就有一个疑问，如果我们在运行过程中，取消了当前View的背景图，那么当前View还会重新为mPrivateFlags设置`PFLAG_SKIP_DRAW`标志吗？
答案：会，这也正是`PFLAG_ONLY_DRAWS_BACKGROUND`标志的作用。

我们看下`View.setBackgroundDrawable`方法的实现：
``` java
public void setBackgroundDrawable(Drawable background) {
if (background == mBackground) {
    return;
}
if (background != null) {
    ...
    mBackground = background;
    if ((mPrivateFlags & PFLAG_SKIP_DRAW) != 0) {
        //若当前View既设置PFLAG_SKIP_DRAW，又添加了背景，那么只能取消mPrivateFlags的PFLAG_SKIP_DRAW标志，同时替换成PFLAG_ONLY_DRAWS_BACKGROUND，这和setFlags方法里面的逻辑一致
        mPrivateFlags &= ~PFLAG_SKIP_DRAW;
        mPrivateFlags |= PFLAG_ONLY_DRAWS_BACKGROUND;
    }
}else{
    //这里取消了背景图
    mBackground = null;
    if ((mPrivateFlags & PFLAG_ONLY_DRAWS_BACKGROUND) != 0){
        /*
        * This view ONLY drew the background before and we're removing
        * the background, so now it won't draw anything
        * (hence we SKIP_DRAW)
        */
        //如果mPrivateFlags包含PFLAG_ONLY_DRAWS_BACKGROUND标志，说明之前mViewFlags设置了WILL_NOT_DRAW标志，但是因为之前当前View有背景图，那么只能先设置PFLAG_ONLY_DRAWS_BACKGROUND标志。现在当前View的背景图取消了，所以可以重新对mPrivateFlags设置PFLAG_SKIP_DRAW了
        mPrivateFlags &= ~PFLAG_ONLY_DRAWS_BACKGROUND;
        mPrivateFlags |= PFLAG_SKIP_DRAW;
    }
}
}
```
上述代码里的注释已经说的很清楚了。如果取消了当前View的背景图，系统会把mPrivateFlags的`PFLAG_ONLY_DRAWS_BACKGROUND`标志重新替换为`PFLAG_SKIP_DRAW`标志。`setBackgroundDrawable`方法的相关逻辑如下图所示：
![setBackgroundDrawable](http://7xs2qy.com1.z0.glb.clouddn.com/%E8%AE%BE%E7%BD%AE%E8%83%8C%E6%99%AF.jpg)

到这里关于`PFLAG_SKIP_DRAW`标志的分析已经结束了。回到我们开头的问题：为什么默认情况下，`ViewGroup.draw`（ViewGroup.onDraw）方法不会被调用。对照上面的分析，可知：肯定是ViewGroup的mPrivateFlags打上了`PFLAG_SKIP_DRAW`标志，那么究竟是在哪里设置的该标志那？

原来默认情况下，ViewGroup在初始化的时候，会通过下面的代码为为mViewFlags设置`WILL_NOT_DRAW`标志。并且默认情况下，ViewGroup也没有背景图，所以就为ViewGroup的mPrivateFlags打上了`PFLAG_SKIP_DRAW`标志。导致`ViewGroup.draw`方法不会被调用，那么`ViewGroup.onDraw`方法就更不会被调用了。
``` java
 private void initViewGroup() {
    // ViewGroup doesn't draw by default
    if (!debugDraw()) {
        setFlags(WILL_NOT_DRAW, DRAW_MASK);
    }
    
    ...
}
```

>总结一下，决定`View.draw`方法是否被调用的直接因素是：**View.mPrivateFlags是否包含PFLAG_SKIP_DRAW标识**；而要包含此标识，需要同时满足两个条件：
1. View.mViewFlags包含WILL_NOT_DRAW标识，可通过View.setWillNotDraw(true)设置该标识。
2. 当前View没有背景图。

>因此，如果我们想让`ViewGroup.draw`被调用，只要破坏上述任何一个条件就可以了。
1. 调用View.setWillNotDraw(false)，取消View.mViewFlags中的**WILL_NOT_DRAW**标识
2. 为ViewGroup设置背景图

## `ViewGroup.onDraw`的调用时机
由上文可知，即使`ViewGroup.draw`被调用了，`ViewGroup.onDraw`也不一定会被调用。必须满足不是实心控件（View.mPrivateFlags没有打上`PFLAG_DIRTY_OPAQUE`标识），`ViewGroup.onDraw`才会被调用。
>实心控件：控件的onDraw方法能够保证此控件的所有区域都会被其所绘制的内容完全覆盖。换句话说，通过此控件所属的区域无法看到此控件之下的内容，也就是既没有半透明也没有空缺的部分。

那么View.mPrivateFlags在什么情况下会被打上`PFLAG_DIRTY_OPAQUE`标识那。通过查看源码，发现相关逻辑在`ViewGroup.invalidateChild`方法中：
``` java
//这里的child表示直接调用invalidate的子View。
public final void invalidateChild(View child, final Rect dirty) {
//计算子View是否是实心的
final boolean isOpaque = child.isOpaque() && !drawAnimation && child.getAnimation() == null && childMatrix.isIdentity();
//PFLAG_DIRTY和PFLAG_DIRTY_OPAQUE是互斥的
int opaqueFlag = isOpaque ? PFLAG_DIRTY_OPAQUE : PFLAG_DIRTY;

do { //循环遍历到ViewRootImpl为止
    View view = null;//父View
    if (parent instanceof View) {
        view = (View) parent;
    }
    if (view != null) { //给当前父View打上相应的flag
        //父View若包含FADING_EDGE_MASK标识，那么只能打上FLAG_DIRTY标识，表示会调用ViewGroup.onDraw方法
        if ((view.mViewFlags & FADING_EDGE_MASK) != 0 &&
                            view.getSolidColor() == 0) {
            opaqueFlag = PFLAG_DIRTY;
        }
        if ((view.mPrivateFlags & PFLAG_DIRTY_MASK) != PFLAG_DIRTY) {
            //PFLAG_DIRTY和PFLAG_DIRTY_OPAQUE是互斥的
            view.mPrivateFlags = (view.mPrivateFlags & ~PFLAG_DIRTY_MASK) | opaqueFlag;
        }
    }
    ...
}
```
通过上述代码可知：View.invalidate方法会向上回溯到ViewRootImpl，在此过程中，若子控件是实心的，则会将**当前父控件**标记为**PFLAG_DIRTY_OPAQUE**，否则为**PFLAG_DIRTY**。
对于包含**PFLAG_DIRTY_OPAQUE**标识的控件，在绘制过程中，会跳过`drawBackground`方法（绘制背景）和`onDraw`方法(绘制自身内容）。 

决定一个View是否实心完全取决于`isOpaque`方法，该方法的默认实现是检查`View.mPrivateFlags`是否包含`PFLAG_OPAQUE_MASK`标识。`PFLAG_OPAQUE_MASK`标识（实心）又由`PFLAG_OPAQUE_BACKGROUND`（背景实心）和`PFLAG_OPAQUE_SCROLLBARS`（滚动条实心）组成。即：只有View同时满足背景实心和滚动条实心，那么它才是opaque的。
真正计算View是否实心的方法是`computeOpaqueFlags`，如下所示：
``` java
 protected void computeOpaqueFlags() {
    // Opaque if:
    //   - Has a background
    //   - Background is opaque
    //   - Doesn't have scrollbars or scrollbars overlay
    //若View包含背景，且背景是不透明的，则打上PFLAG_OPAQUE_BACKGROUND标识
    if (mBackground != null && mBackground.getOpacity() == PixelFormat.OPAQUE) {
        mPrivateFlags |= PFLAG_OPAQUE_BACKGROUND;
    } else {
        mPrivateFlags &= ~PFLAG_OPAQUE_BACKGROUND;
    }

    final int flags = mViewFlags;
    //若没有横竖滚动条，或者滚动条是OVERLAY类型的，则打上PFLAG_OPAQUE_SCROLLBARS标识
    if (((flags & SCROLLBARS_VERTICAL) == 0 && (flags & SCROLLBARS_HORIZONTAL) == 0) ||
                (flags & SCROLLBARS_STYLE_MASK) == SCROLLBARS_INSIDE_OVERLAY ||
                (flags & SCROLLBARS_STYLE_MASK) == SCROLLBARS_OUTSIDE_OVERLAY) {
        mPrivateFlags |= PFLAG_OPAQUE_SCROLLBARS;
    } else {
        mPrivateFlags &= ~PFLAG_OPAQUE_SCROLLBARS;
    }
}
```
只有同时打上了`PFLAG_OPAQUE_BACKGROUND`和`PFLAG_OPAQUE_SCROLLBARS`标识，当前View才是实心的。该方法会在View中的很多地方被调用，以实时确定View是否是实心的。
当然，如果`isOpaque`方法的默认实现不符合我们的需求，我们可以自己实现，这也是官方推荐的做法。
### Demo验证
下面我们通过一个Demo验证上述逻辑：
1. 设定一个自定义父ViewGroupA和子ViewB。
2. 对父ViewGroupA调用setWillNotDraw(false)，保证父ViewGroupA的draw方法会被调用。
3. 对子ViewB设置一个Click事件，具体实现就是调用子ViewB.invalidate方法。
4. 通过点击子ViewB，观察父ViewGroupA和子ViewB的draw和onDraw方法是否会被调用。

>上述Demo必须采用软件绘制才有效。在硬件绘制下，子ViewB调用invalidate方法，只会触发子ViewB自己的draw方法，它的父View是不需要重绘的。

假如我们对子ViewB设置了一个纯色的背景（子ViewB变成实心了），那么可以得到如下结论：
1. 在View树第一次渲染的时候，父ViewGroupA和子ViewB的draw和onDraw方法都会被调用。
2. 在后续点击子ViewB的时候，子ViewB的draw和onDraw方法都会被调用，父ViewGroupA的draw方法也会被调用，**但是父ViewGroupA的onDraw方法不会被调用**。

假如我们没有对子ViewB设置背景（子ViewB变成非实心了），那么可以得到如下结论：
1. 在View树第一次渲染的时候，父ViewGroupA和子ViewB的draw和onDraw方法都会被调用。
2. 在后续点击子ViewB的时候，父ViewGroupA和子ViewB的draw和onDraw方法都会被调用。

当然控制一个View是否实心，我们也可以直接重写`isOpaque`方法，没必要像上面这么麻烦。

>总结一下，**首次渲染View树的时候，只要ViewGroup.draw方法被调用了，那么ViewGroup.onDraw就会被调用**。
**但是后续子View.invalidate的时候，在ViewGroup.draw方法被调用的前提下，还要子View是非实心的，那么ViewGroup.onDraw和ViewGroup.drawBackground才会被调用**。`

## 总结
最后用一张图来总结下ViewGroup的draw和onDraw方法的调用逻辑图。
![ViewGroup的draw和onDraw方法的调用逻辑图](http://7xs2qy.com1.z0.glb.clouddn.com/ViewGroup.jpg)

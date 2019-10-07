---
title: Android SystemBar
date: 2016-05-31 20:42:10
tags:
 - Android
 - SystemBar
categories:
 - Android
---
SystemBar是用来展示通知、表现设备状态和完成设备导航的屏幕区域。主要包括状态栏（1：status bar）和导航栏（2：navigation bar）。借用官方的图，如下所示，我们可以根据需要对SystemBar进行一些操作，满足自己的需求。
![SystemBar](http://7xs2qy.com1.z0.glb.clouddn.com/%E5%AE%98%E6%96%B9.png)
<!-- more -->
### 淡化SystemBar (View.SYSTEM_UI_FLAG_LOW_PROFILE)
从API14，即4.0开始，我们可以借助`View.SYSTEM_UI_FLAG_LOW_PROFILE`来淡化SystemBar，以突出内容区域。

当你使用这个方法的时候，内容区域的大小并不会发生变化，只是系统栏的图标会收起来。一旦用户触摸状态栏或者是导航栏的时候，这两个系统栏就又都会完全显示（无透明度）。
这种方法的优势是SystemBar仍然可见，但是它们的细节被隐藏掉了，因此可以在不牺牲快捷访问系统栏的情况下创建一个沉浸式的体验。
设置代码：
``` java
mView.setSystemUiVisibility(View.SYSTEM_UI_FLAG_LOW_PROFILE);
```
如图所示：
<img src="http://7xs2qy.com1.z0.glb.clouddn.com/%E6%B7%A1%E5%8C%96SystemBar.png" width="300"/>

一旦用户触摸到状态栏或者是系统栏，这个标签就会被清除，使系统栏重新显现（无透明度）。在标签被清除的情况下，如果你想重新淡化系统栏就必须重新设定这个标签。

或者我们也可以通过代码直接清除该标志：
``` java
mView.setSystemUiVisibility(View.SYSTEM_UI_FLAG_VISIBLE);
```
如图所示：
<img src="http://7xs2qy.com1.z0.glb.clouddn.com/%E6%B2%A1%E6%9C%89%E6%B7%A1%E5%8C%96SystemBar.png" width="300"/>
`View.SYSTEM_UI_FLAG_VISIBLE`表示请求系统显示SystemBar.
### 隐藏SystemBar
在Api15及其之下，可以通过设置Window的Flag标志来达到全屏目的,相关的Flag主要有以下三个：

1. WindowManager.LayoutParams.FLAG_FULLSCREEN
    > 设置全屏模式，隐藏窗口装饰。除了动态设置外，还可以在主题中设置windowFullscreen属性来达到全屏目的。
    
2. WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN
    >设置了FLAG_LAYOUT_IN_SCREEN之后，可以拥有与启用FLAG_FULLSCREEN相同的屏幕区域。这个方法防止了状态栏隐藏和显示时，内容区域的大小变化。但是此时，状态栏应该是在压在内容区域之上，所以需要自己处理布局，防止状态栏遮盖重要的内容区域。

3. WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS
    >allow window to extend outside of the screen. 允许窗口超出屏幕


下面来看一个具体的案例：

假如我们一开始想要以全屏模式（布局充满整个屏幕）运行，但是也需要在特定场景下，显示状态栏，并且要求状态栏是漂浮在布局上面，而不是把布局顶下去，该怎么实现那？

首先要在`Activity.onCreate`方法中，添加如下代码：
``` java
//表示布局文件充满整个屏幕，即View布局不受状态栏限制，此时状态栏漂浮在布局之上。
getWindow().addFlags(WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN | WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS)
```
然后根据实际情况，调用以下函数来显示和隐藏状态栏（显示出来的状态栏是漂浮在布局之上的）
``` java
//显示状态栏
public void showSystemBar() {
    getWindow().clearFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN);
}
//隐藏状态栏
public void hideSystemBar() {
    getWindow().addFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN);
}
```
如果不添加`Activity.onCreate`中的代码，仅通过下面的两个方法来显示和隐藏状态栏，那么每次都会导致重新布局（View布局会被状态栏顶下来，充不满全屏）。

当你设置WindowManager标签之后（无论是通过Activity主题还是动态设置），这个标签都会一直生效直到你清除它。这点和View.setSystemUiVisibility设置有本质的区别。

---

在Api16及其之上，可以通过View.setSystemUiVisibility来设置。该方法可以用来控制SystemBar的显示和隐藏，下面简单介绍下其可以设置的Flag：
最重要的两个Flag：
1. View.SYSTEM_UI_FLAG_IMMERSIVE `Api19` (控制SYSTEM_UI_FLAG_HIDE_NAVIGATION flag是否能被用户交互行为所清除) 
    > **If this flag is not set, SYSTEM_UI_FLAG_HIDE_NAVIGATION will be force cleared by the system on any user interaction**。如果没有设置该flag，那么任何用户交互行为都会清除SYSTEM_UI_FLAG_HIDE_NAVIGATION flag，从而导致导航栏重新显示出来。

    > Since this flag is a modifier for SYSTEM_UI_FLAG_HIDE_NAVIGATION, it only has an effect when used in combination with that flag.

2. View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY `Api19`（控制SYSTEM_UI_FLAG_FULLSCREEN和SYSTEM_UI_FLAG_HIDE_NAVIGATION flag是否能被用户交互行为所清除）
    > **If this flag is not set, SYSTEM_UI_FLAG_HIDE_NAVIGATION will be force cleared by the system on any user interaction, and SYSTEM_UI_FLAG_FULLSCREEN will be force-cleared by the system if the user swipes from the top of the screen**.如果没有设置该flag，那么任何用户交互行为都会清除SYSTEM_UI_FLAG_HIDE_NAVIGATION flag，同时从边缘区域向内滑动则会清除SYSTEM_UI_FLAG_FULLSCREEN flag，从而导致状态栏和导航栏重新显示出来。

    > When system bars are hidden in immersive mode, they can be revealed temporarily(临时显示) with system gestures, such as swiping from the top of the screen. These transient system bars will overlay app’s content, may have some degree of transparency, and will automatically hide after a short timeout.这种模式下的SystemBar，是半透明的，在显示出来后，隔一段时间会自动隐藏，不会清除SYSTEM_UI_FLAG_HIDE_NAVIGATION和SYSTEM_UI_FLAG_FULLSCREEN flag。
    
    > Since this flag is a modifier for SYSTEM_UI_FLAG_FULLSCREEN and SYSTEM_UI_FLAG_HIDE_NAVIGATION, it only has an effect when used in combination with one or both of those flags.

* 若仅仅设置了`View.SYSTEM_UI_FLAG_FULLSCREEN | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN | View.SYSTEM_UI_FLAG_LAYOUT_STABLE | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION`，那么用户点击内容区域的任何位置都会唤出SystemBar，清除SYSTEM_UI_FLAG_FULLSCREEN和SYSTEM_UI_FLAG_HIDE_NAVIGATION标志位。这不是真正的沉浸式，因为在这种全屏模式下，用户无法和内容区域进行交互。
    
* 若是设置了`View.SYSTEM_UI_FLAG_IMMERSIVE |  View.SYSTEM_UI_FLAG_FULLSCREEN | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN | View.SYSTEM_UI_FLAG_LAYOUT_STABLE | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION`，那么用户只有在从边缘区域向内滑动时，才能让SystemBar显示。这算是沉浸式了，用户可以和内容区域直接交互了。只有特定的操作(从边缘区域向内滑动)，才会清除标志位，重新唤出SystemBar。
    
* 若是设置了`View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY |  View.SYSTEM_UI_FLAG_FULLSCREEN | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN | View.SYSTEM_UI_FLAG_LAYOUT_STABLE | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION`，那么唤出SystemBar的方式和`SYSTEM_UI_FLAG_IMMERSIVE`相同，只不过唤出的SystemBar是半透明的，淡化的SystemBar，并且不再会清除SYSTEM_UI_FLAG_FULLSCREEN和SYSTEM_UI_FLAG_HIDE_NAVIGATION标志位，在几秒后，SystemBar又会重新隐藏。这个算不算沉浸式那，算吧。

除了`SYSTEM_UI_FLAG_IMMERSIVE_STICKY`之外，其他两种模式在SystemBar的可见性发生改变时，都可以通过`View.OnSystemUiVisibilityChangeListener`监听器来获得通知。
    
请注意，若带有`SYSTEM_UI_FLAG_IMMERSIVE_STICKY`标签，则不会触发任何的监听器，因为在这种模式下展示的SystemBar是处于暂时(transient)的状态。
实际使用中，需要根据具体需求，来选择三种不同的用户交互模式。

其他的Flag如下所示：

| Flag | Comment | API |
| :---: | :------ | :---: |
| SYSTEM_UI_FLAG_VISIBLE | View已经设置SystemBar是可见的 | 14 |
| SYSTEM_UI_FLAG_LOW_PROFILE | 这种模式下，所有的SystemBar都会变淡，但不会隐藏。一旦用户触摸到了状态栏或者导航栏，这个标签就会被清除，使SystemBar重新显示出来。在标签被清除的情况下，如果你想重新淡化SystemBar，就必须重新设定它。 | 14 |
| SYSTEM_UI_FLAG_HIDE_NAVIGATION | 临时性的隐藏导航栏。因为导航栏是如此重要，因此最少的用户交互也会导致该标志和SYSTEM_UI_FLAG_FULLSCREEN标志被清除，相应的，状态栏和导航栏也会重新显示出来。 | 14 |
| SYSTEM_UI_FLAG_FULLSCREEN | 请求隐藏状态栏，若只设置了该标志位，没有设置SYSTEM_UI_FLAG_HIDE_NAVIGATION，那么用户可以和内容区域进行正常的交互。（除了从顶部边缘区域向内滑动时，会清除该标志位，同时重新显示状态栏），也正因为如此，我们需要提供更直观的方式，来让用户更方便地退出全屏模式（例如：点击屏幕区域，主动清除该标志位，以重新显示状态栏）。若同时设置了SYSTEM_UI_FLAG_HIDE_NAVIGATION，那么就以上面介绍的为准。 | 16 |
| SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION | 内容区域占据status bar的位置，但是status bar还是显示，并且覆盖在内容区域上面。 | 16 |
| SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN | 内容区域占据navigation bar的位置，但是navigation bar还是显示，并且覆盖在内容区域上面。 | 16 |
| SYSTEM_UI_FLAG_LAYOUT_STABLE | 一般和SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN、SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION 一起使用，提供稳定的SystemBar隐藏和显示。 | 16 |
| SYSTEM_UI_FLAG_LIGHT_STATUS_BAR | Requests the status bar to draw in a mode that is compatible with light status bar backgrounds。要想该标志起作用，the window must request FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS but not FLAG_TRANSLUCENT_STATUS | 23 |

#### 关于WindowManager.LayoutParams.FLAG_FULLSCREEN和View.SYSTEM_UI_FLAG_FULLSCREEN
两者达到的目标效果是一致的，都可以隐藏status bar，但存在着一些区别：
1. FLAG_FULLSCREEN从Api1开始就有了，不存在版本兼容问题；而SYSTEM_UI_FLAG_FULLSCREEN则是从Api16开始才有的，存在着兼容问题。
2. FLAG_FULLSCREEN是通过Window添加到对应Window的布局参数WindowManager.LayoutParams中；而SYSTEM_UI_FLAG_FULLSCREEN则是作用某个可见的View。
3. **最重要的区别**：FLAG_FULLSCREEN的作用是持久的，一经设置，永久有效（除非主动清除`clearFlags`）;而SYSTEM_UI_FLAG_FULLSCREEN则是临时性的，通过用户交互操作，可以直接清除标志。
4. 若设置FLAG_FULLSCREEN，需要单独处理ActionBar；而若ActionBar设置了Window.FEATURE_ACTION_BAR_OVERLAY，那么再设置SYSTEM_UI_FLAG_FULLSCREEN时，会同时隐藏ActionBar。

### 关于View.setFitsSystemWindows和View.fitSystemWindows
当我们通过设置上述Flag，来让Content View延伸到SystemBar时，SystemBar会覆盖在Content View之上，这会带来一些不好的体验。

Android本身给我们提供了解决方案，即调用`View.setFitsSystemWindows`或者设置属性`android:fitsSystemWindows="true"`。这样，Android系统会调用`View.fitSystemWindows`来修复SystemBar覆盖Content View的情况。简单来说，`fitsSystemWindow="true"` 会使得屏幕上的Content View位于状态栏下方与导航栏上方的区域。

View.fitSystemWindows方法会接收Rect类型的参数，表示Current content insets of the window。fitSystemWindows方法最终会将Rect参数设置到当前View的Padding中，已达到调整Content View的内边距，保证Content View不会被SystemBar覆盖的目的。因此，我感觉Rect.top应该表示Status Bar的高度（不包含ActionBar的情况下），Rect.bottom应该对应Navigation Bar的高度，下面会验证下。
View.fitSystemWindows修改View Padding的核心代码如下所示，
View.fitSystemWindows -> View.internalSetPadding：

``` java
//left、top、right和bottom即为Rect参数对应的值
if (mPaddingLeft != left) {
    mPaddingLeft = left;
    }
if (mPaddingTop != top) {
    mPaddingTop = top;
    }
if (mPaddingRight != right) {
    mPaddingRight = right;
    }
if (mPaddingBottom != bottom) {
    mPaddingBottom = bottom;
    }
```

做了一下实验，主要是验证下fitSystemWindows方法中的Rect参数的取值和SystemBar高度的关系：
设置SystemBar的代码如下所示：
``` java
//在mMyRelativeLayout中重写fitSystemWindows,查看Rect参数
mMyRelativeLayout.setFitsSystemWindows(true);
//设置Flag，控制SystemBar的可见性
mMyRelativeLayout.setSystemUiVisibility(View.SYSTEM_UI_FLAG_FULLSCREEN | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION |  View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION | View.SYSTEM_UI_FLAG_LAYOUT_STABLE)
```
获取状态栏和导航栏的高度：
``` java
//获取状态栏高度
public static int getStatusBarHeight(Context context) {
        int result = 0;
        Resources resources = context.getResources();
        int resourceId = resources.getIdentifier("status_bar_height", "dimen", "android");
        if (resourceId > 0) {
            result = resources.getDimensionPixelSize(resourceId);
        }
        Log.d("leon", "getStatusBarHeight = " + result);
        return result;
    }
 
//获取导航栏的高度   
public static int getNavigationBarHeight(Context context) {
        int result = 0;
        Resources resources = context.getResources();
        int resourceId = resources.getIdentifier("navigation_bar_height", "dimen", "android");
        if (resourceId > 0) {
            result = resources.getDimensionPixelSize(resourceId);
        }
        Log.d("leon", "getNavigationBarHeight = " + result);
        return result;
    }
```
然后，对比Rect的取值和SystemBar的高度：
``` java
//rect参数取值
insets = Rect(0, 50 - 0, 96)
//status bar的高度
getStatusBarHeight = 50
//navigation bar的高度
getNavigationBarHeight = 96
//修改后的mMyRelativeLayout的topPadding
topPadding = 50
//修改后的mMyRelativeLayout的bottomPadding
bottomPadding = 96
```
从上述结果对比来看，验证了我们之前的猜测，参数Rect的取值和状态栏和导航栏的高度相关。

此外，如果fitSystemWindows方法的默认行为无法满足我们的需求，那么我们可以重写该方法，参考Rect的取值，自己决定如何修改mMyRelativeLayout的的padding.

---

从Api20开始，fitSystemWindows方法被废弃了，提供了新的方法dispatchApplyWindowInsets(WindowInsets)、 onApplyWindowInsets(WindowInsets)  和setOnApplyWindowInsetsListener(android.view.View.OnApplyWindowInsetsListener)来实现类似功能，这部分后续再看下吧。


### 参考文章
1. [android-training-course-in-chinese之管理系统UI](http://hukai.me/android-training-course-in-chinese/ui/system-ui/index.html)
2. [官方文档](http://developer.android.com/intl/zh-cn/reference/android/view/View.html#fitSystemWindows(android.graphics.Rect)
3. [透明状态栏和透明导航栏](http://www.cnblogs.com/zhengxt/p/3536905.html)
4. [沉浸模式](http://www.cnblogs.com/zhengxt/p/3508485.html)


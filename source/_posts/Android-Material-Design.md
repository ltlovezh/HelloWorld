---
title: Android Material Design
date: 2016-08-05 20:47:25
tags:
 - Android
 - Material Design
 - Snackbar
 - FloatingActionButton
 - CoordinatorLayout
categories:
 - Android
---

Android Material Design设计规范已经出来很久了，但是国内APP貌似很少使用啊。而且Android Design Support Library提供了良好的兼容性，可以直接向下兼容到Android2.2，完全可以一试哈。本文简单介绍下Design库提供的一些组件，方便日后查询使用。

<!--more -->

## Snackbar
Snackbar是一种介于Toast和AlertDialog之间的轻量级控件，可以很方便的提供消息提示和动作反馈。

Snackbar的使用方式和Toast类似，但与Toast不同的是，Snackbar可以带有按钮，并且只会出现在屏幕底部；Snackbar支持右滑删除，类似通知栏的消息；如果用户没做任何操作，Snackbar在到达设定的时间后会自动消失。

使用Snackbar的代码很简单，如下所示：

``` java
Snackbar.make(myView, 
"This is Snackbar",
Snackbar.LENGTH_LONG)
    .setAction("OK", 
            new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                        //Snackbar上按钮的点击事件
                    }
                }).show();
```

效果图如下所示：

<img src="http://7xs2qy.com1.z0.glb.clouddn.com/Snackbar.png" width="400"/>

## FloatingActionButton
Floating Action Button表示一个悬浮按钮，一般推荐在一个屏幕上展示一个FloatingActionButton，用于重要的全局操作。

FAB按钮默认会使用主题中定义的背景色，但是我们也可以修改背景色，以下是一些可以设置的属性：

* fabSize ： 设置FAB的大小 ('normal'为56dp or 'mini'为40dp)
* backgroundTint ： 设置FAB的背景色
* rippleColor ： 设置按下时的背景色
* src ： 设置FAB中显示的图标
* layout_anchor : 设置FAB相对于哪个锚点View悬浮展示
* layout_anchorGravity : 设置相对于锚点View的对齐方式

使用起来也很简单，只要在布局文件中配置就可以了：

``` XML
<android.support.design.widget.FloatingActionButton
    android:id="@+id/fab"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:layout_margin="20dp"
    app:layout_anchor="@id/pager"
    app:layout_anchorGravity="bottom|right"
    android:clickable="true"
    app:rippleColor="#00ff00"
    android:src="@drawable/qq"
    app:fabSize="normal" />
```

效果图如下所示：

<img src="http://7xs2qy.com1.z0.glb.clouddn.com/FloatingActionButton.png" width="400"/>

## TabLayout
TabLayout用来实现多Tab分组功能，TabLayout既实现了固定的选项卡（fixed）：子Tab View的宽度平均分配，也实现了可滚动的选项卡（scrollable）：子Tab View宽度不固定，当子tab过多时，可以横向滚动。

TabLayout有两个比较重要的属性，分别是：

* tabMode : TabLayout的Tab模式，可以选择fixed or scrollable
* tabGravity : 子Tab的对齐方式, 可以选择 fill 或 center

TabLayout一般和Viewpager一起使用，实现联动的多Tab功能（滑动ViewPager，自动切换上面的子Tab）。

下面我们来看一个例子，布局文件如下所示：
``` XML
    <android.support.design.widget.TabLayout
        android:id="@+id/tabs"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:tabGravity="center"
        app:tabMode="fixed"/>

    <android.support.v4.view.ViewPager
        android:id="@+id/pager"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />
```

相应的实现代码如下所示：

``` java
ViewPager mViewPager = (ViewPager) findViewById(R.id.pager);
TabLayout mTabLayout = (TabLayout) findViewById(R.id.tabs);
//获取ViewPager适配器，views表示ViewPager的选项卡View列表，titleList表示TabLayout中的子Tab标题列表，这里不是重点，代码就不贴了。
MyViewPagerAdapter myViewPagerAdapter = new MyViewPagerAdapter(views, titleList);
mViewPager.setAdapter(myViewPagerAdapter);
//把TabLayout和ViewPager关联起来了
mTabLayout.setupWithViewPager(mViewPager);

//ViewPager适配器
class MyViewPagerAdapter extends PagerAdapter {
    private List<View> mListViews;//
    private List<String> mTitleList;//TabLayout中的子Tab标题

    public MyViewPagerAdapter(List<View> mListViews, List<String> titleList) {
            this.mListViews = mListViews;
            this.mTitleList = titleList;
        }

    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
            container.removeView(mListViews.get(position));
        }


    @Override
    public Object instantiateItem(ViewGroup container, int position) { 
        container.addView(mListViews.get(position));
        return mListViews.get(position);
    }

    @Override
    public int getCount() {
        return mListViews.size();
    }

    @Override
    public boolean isViewFromObject(View arg0, Object arg1){
        return arg0 == arg1;//标准写法...
    }

    @Override
    public CharSequence getPageTitle(int position) {
        //获取TabLayout中的子Tab标题
        return mTitleList.get(position);
    }
}
```

具体效果可以参见CoordinatorLayout中的效果图（都使用了这种关联滚动）。

## CoordinatorLayout
CoordinatorLayout是FrameLayout的增强型，主要用来作为顶层布局，协调子View之间的滑动事件处理，以及实现子View之间的状态变化监听。

CoordinatorLayout的子View可以设置`layout_anchor`和`layout_anchorGravity`属性，用于将某个子View悬浮于指定View指定位置。 

这里我们先看一下CoordinatorLayout结合其他Design组件，一起实现的一些效果，然后用另一篇文章分析下CoordinatorLayout的实现原理。

### 与AppBarLayout结合
AppBarLayout是一个垂直方向的LinearLayout，通常作为CoordinatorLayout的直接子View使用，AppBarLayout的子View需要设置`app:layout_scrollFlags`属性，该属性用来控制子View跟随滚动列表一起滚动，还是固定在最上面，有以下几种取值：

* scroll ： 需要跟着滚动列表一起滚动出屏幕的的子View需要设置这个Flag，若没有该Flag，那么子View将被固定在屏幕顶部。
* enterAlways ： 若设置该Flag，那么当向下滚动时，子View将会随着滚动列表，从屏幕外滚动到屏幕内，即一开始的滚动只是把子View滚动到屏幕内，当子View滚动到原来位置后，滚动列表才会真正的滚动它自己的内容。
* enterAlwaysCollapsed：若子View设置了minHeight属性，又设置该Flag，那么向下滚动时，子View只能以最小高度进入，只有当滚动列表到达顶部时，才继续扩大到子View的完整高度。
* exitUntilCollapsed：向上滚动时收缩View，但最后折叠在屏幕顶端。一般和CollapsingToolbarLayout一起使用。

>Tips：所有设置`scroll`标志的子View必须在没有设置的之前定义，这样可以确保那些设置过的子View都从上面滚动出屏幕,只留下那些固定的子View在下面。

下面我们来看一个例子，布局文件如下所示：
``` XML
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/main_content"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">

    <android.support.design.widget.AppBarLayout
        android:id="@+id/appbar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar">

        <android.support.v7.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            android:background="?attr/colorPrimary"
            app:layout_scrollFlags="scroll|enterAlways"
            app:popupTheme="@style/ThemeOverlay.AppCompat.Light" />

        <android.support.design.widget.TabLayout
            android:id="@+id/tabs"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />

    </android.support.design.widget.AppBarLayout>


    <android.support.v4.view.ViewPager
        android:id="@+id/pager"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior" />


    <android.support.design.widget.FloatingActionButton
        android:id="@+id/fab"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_margin="20dp"
        android:clickable="true"
        android:src="@drawable/qq"
        app:fabSize="normal"
        app:layout_anchor="@id/pager"
        app:layout_anchorGravity="bottom|right"
        app:rippleColor="#00ff00" />

</android.support.design.widget.CoordinatorLayout>
```

这里使用上面介绍的方式把TabLayout和ViewPager关联起来，代码中ViewPager的子View分别是RecyclerView和NestedScrollView，都可以实现滚动操作，所以可以测试ViewPager的上下滚动行为是怎么影响兄弟节点AppBarLayout的。

上面看到，我们对ViewPager设置了`app:layout_behavior="@string/appbar_scrolling_view_behavior"`，该behavior的实现类是`android.support.design.widget.AppBarLayout$ScrollingViewBehavior`。而实际上AppBarLayout也拥有一个behavior（通过注解实现的)，实现类是`android.support.design.widget.AppBarLayout$Behavior`，CoordinatorLayout正是依赖这两个Behavior，协调子View之间的滑动处理的，这块内容后续会进行详细介绍。

所以，要实现上面的ToolBar跟随滚动列表滚动，需要注意以下几个几点：

>1. CoordinatorLayout为顶级父容器，负责协调子View间的滚动事件。
>2. AppBarLayout的子View中需要协同滚动列表滚动的view需要设置`app:layout_scrollFlags`属性
>3. 滚动组件需要设置`app:layout_behavior`属性。

最终的效果图如下所示：

<img src="http://7xs2qy.com1.z0.glb.clouddn.com/AppBarLayout.gif" width="400"/>

如上所示，向上滚动列表时，会优先把ToolBar滚动出屏幕，然后才是滚动列表内容；而向下滚动时，会优先把TooBar滚动入屏幕，然后才是滚动列表内容。可见滚动列表的滚动操作间接影响了AppBarLayout的滚动行为，而这正是CoordinatorLayout的作用：协调子View之间的滚动事件。

### 与CollapsingToolbarLayout结合
CollapsingToolbarLayout一般作为AppBarLayout的直接子View，是对Toolbar的进一步封装，目的是实现可折叠的Toolbar。

CollapsingToolbarLayout提供了一些属性，其中有几个比较重要：

* `app:title`：CollapsingToolbarLayout的标题，当CollapsingToolbarLayout全屏时，字体比较大，随着Toolbar的不断折叠， 字体会不断变小。
* `app:contentScrim`：CollapsingToolbarLayout折叠到最小后，固定在屏幕顶部时的背景。
* `app:statusBarScrim`：状态栏的背景，只在Android5.0以上才有效果。

此外，CollapsingToolbarLayout的子View还需要设置以下属性：

* `app:layout_collapseMode`：表示子View的折叠模式，有两种取值：pin表示固定模式，即CollapsingToolbarLayout完全折叠后，Toolbar仍然被固定在屏幕的顶部。parallax表示视差模式，即CollapsingToolbarLayout折叠过程中，子View可以同时滚动，实现视差滚动效果。
* `app:layout_collapseParallaxMultiplier`,当为视差模式时，表示视差因子，范围为[0.0,1.0]，值越大视差越大。

下面我们来看一个例子，布局文件如下所示：
``` XML
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/main_content"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">

    <android.support.design.widget.AppBarLayout
        android:id="@+id/appbar"
        android:layout_width="match_parent"
        android:layout_height="256dp"
        android:fitsSystemWindows="true"
        android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar">

        <android.support.design.widget.CollapsingToolbarLayout
            android:id="@+id/collapsing_toolbar"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:fitsSystemWindows="true"
            app:collapsedTitleGravity="left"
            app:contentScrim="?attr/colorPrimary"
            app:expandedTitleGravity="center"
            app:expandedTitleMarginStart="30dp"
            app:layout_scrollFlags="scroll|enterAlways|exitUntilCollapsed"
            app:statusBarScrim="@drawable/qq"
            app:title="CoordinatorLayout">

            <ImageView
                android:id="@+id/backdrop"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:fitsSystemWindows="true"
                android:scaleType="centerCrop"
                android:src="@drawable/qzone"
                app:layout_collapseMode="pin"
                app:layout_collapseParallaxMultiplier="0" />

            <android.support.v7.widget.Toolbar
                android:id="@+id/toolbar"
                android:layout_width="match_parent"
                android:layout_height="?attr/actionBarSize"
                app:layout_collapseMode="pin"
                app:popupTheme="@style/ThemeOverlay.AppCompat.Light" />

        </android.support.design.widget.CollapsingToolbarLayout>

    </android.support.design.widget.AppBarLayout>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical"
        app:layout_behavior="@string/appbar_scrolling_view_behavior">
        <android.support.design.widget.TabLayout
            android:id="@+id/tabs"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            app:tabGravity="center"
            app:tabMode="fixed" />

        <android.support.v4.view.ViewPager
            android:id="@+id/pager"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />
    </LinearLayout>

    <android.support.design.widget.FloatingActionButton
        android:id="@+id/fab"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_margin="20dp"
        android:clickable="true"
        android:src="@drawable/qq"
        app:fabSize="normal"
        app:layout_anchor="@id/pager"
        app:layout_anchorGravity="bottom|right"
        app:rippleColor="#00ff00" />

</android.support.design.widget.CoordinatorLayout>
```

与上面一样，TabLayout和ViewPager关联起来，代码中ViewPager的子View分别是RecyclerView和NestedScrollView，都可以实现滚动操作，这样滚动下面的滚动列表，可以观察到上面CollapsingToolbarLayout的折叠情况。

此外，通过CollapsingToolbarLayout实现折叠效果，需要注意以下两点：

>1. AppBarLayout的高度需要固定 
>2. CollapsingToolbarLayout的子View需要设置layout_collapseMode属性 

最终的效果图如下所示：

<img src="http://7xs2qy.com1.z0.glb.clouddn.com/CollapsingToolbarLayout.gif" width="400"/>
如上所示，向上滚动列表时，会折叠图片，然后把ToolBar固定在屏幕顶部，最后才是滚动列表内容；而向下滚动时，会优先把折叠的图片展开，然后才是滚动列表内容。可见滚动列表的滚动操作间接影响了CollapsingToolbarLayout的滚动行为，而这正是CoordinatorLayout的作用：协调子View之间的滚动事件。

## 参考文章

1. [探索新的Android Material Design支持库](https://segmentfault.com/a/1190000002976409)
2. [Android的材料设计兼容库（Design Support Library）](http://www.jcodecraeer.com/a/anzhuokaifa/developer/2015/0531/2958.html)
3. [Android CoordinatorLayout使用](http://blog.csdn.net/xyz_lmn/article/details/48055919)

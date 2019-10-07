---
title: Android踩坑集
date: 2016-09-06 21:41:04
tags:
 - Android 
categories:
 - Android
---
Android开发中，经常会花费很多时间解决一些Bug，但是过了一段时间，遇到相同的问题，还是会一脸懵逼，再次浪费时间去解决一遍。本文将记录开发过程中碰到的一些坑，希望下次碰到类似问题，可以迅速填平。

<!--more-->

### View布局问题
曾经遇到一个布局的bug，后续遇到调用requestLayout后，没有重新布局的问题时，可以参考下：

在父ViewGroup A的onLayout期间，若间接触发了子View B的RequestLayout方法，子view B将不会被重新布局。因为RequestLayout无法传递到父ViewGroup A，因为父VieWGroup A 还没有布局完成，此时还有PFLAG_FORCE_LAYOUT标志位，导致子View B的RequestLayout无法继续向上传递到父ViewGroup A。但是子View B的PFLAG_FORCE_LAYOUT标志位已经打上了。

这会导致严重的问题：若子View B的子View C，调用了RequestLayout，因为子View B拥有PFLAG_FORCE_LAYOUT标志位，导致子View C的RequestLayout方法也无法向上传递到子View B，也就无法正常到达顶层的ViewRootImpl了。最直观的现象就是：子View C调用RequestLayout后，但是没有被重新布局。

### ViewPager Fragment
support-24包中，ViewPager在调用destroyItem时，如果item是Fragment，则会调用Fragment的onDestroy()。但是Fragment并没有被回收，当Tab切回来时还是会被使用。所以需要特别注意，在Fragment的onDestroy()方法中执行的逻辑是否会被这个特性影响。

### 自定义Drawable兼容性问题
在用XML写Drawable的时候，如果是只有边框的按钮类型，不要只写边框色，要把填充色也加上。三星9100机型下面，如果不写填充色，系统会默认把填充色设置成黑色。

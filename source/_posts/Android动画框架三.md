---
title: Android动画框架三
date: 2016-04-27 17:54:58
tags:
- Android
- 动画框架
- Choreographer
- 布局动画
categories:
- Android
---
本篇接Android动画框架二,前两篇文章主要介绍了Frame Animation、Tween Animation和Property Animator的使用方式和实现机制。本篇主要学习下Android中布局动画的使用和实现机制。其中，不管是哪种动画的实现机制，最终都会和Choreographer关联起来。关于Choreographer机制，网上有一篇文章讲解的很详细，本文就不再赘述。
## Choreographer机制

Google在Android4.1上增加了Choreographer机制，用于和Vsync机制配合，实现对用户输入、动画以及布局和绘图的控制。关于Choreographer机制的详细实现，可以参考[Android系统Choreographer机制实现过程](http://blog.csdn.net/yangwen123/article/details/39518923)。

Choreographer机制对外的接口就是它的各种postCallbackXX和postFrameCallbackXX方法，在源码中搜下这几个方法的调用，就可以知道哪里使用了Choreographer机制。其中有我们比较熟悉的：

* View.scheduleTraversals方法，实现View的绘制流程（也实现了Tween Animation）；
* ValueAnimator.AnimationHandler.scheduleAnimation方法，实现Property Animator；
* View.scheduleDrawable方法，实现Frame Animation；
* View.postOnAnimation和View.postOnAnimationDelayed方法，实现自定义的动画。

其他的调用，此处不再赘述。

<!-- more -->

## 布局动画
布局动画是指ViewGroup在布局时产生的动画效果。

### LayoutTransition
LayoutTransition表示对ViewGroup中的子View进行动画显示，一般用在对ViewGroup添加、删除、隐藏、显示子View时添加过渡动画，避免僵硬的显示过程。

#### 使用方式
需要明确一点：LayoutTransition是设置给ViewGroup对象的，而非它的子View。
当一个ViewGroup发生以下情况时，会对子View产生动画效果：
* 在ViewGroup中显示地添加一个子元素
* 在ViewGroup中显示地删除一个子元素
* 通过View.setVisibility()改变View的可见性
* ViewGroup的布局发生改变时

因此，LayoutTransition包含5种类型的过渡动画：
1. APPEARING，当一个子View显示在ViewGroup中时，对该子View设置的动画类型。
2. CHANGE_APPEARING，当一个子View显示在ViewGroup中时，对被该子View影响的其他子View设置的动画类型。
3. DISAPPEARING，当一个子View消息在ViewGroup中时，对该子View设置的动画类型。
4. CHANGE_DISAPPEARING，当一个子View消失在ViewGroup中时，对被该子View影响的其他子View设置的动画类型。
5. CHANGE，除了子View出现或消失造成的对其他子View的布局影响之外，一些其他因素造成的对子View的布局影响，都属于该动画类型。

*** Tips：这里主要关注动画发生在哪些View身上。***


> 默认情况下，DISAPPEARING、CHANGE_APPEARING和CHANGE动画是立即开始的，其他动画都有一个默认的开始延迟（默认是300ms）。这是因为，当一个新的View出现的时候，其他View要立即执行CHANGE_APPEARING动画腾出位置，而新出现的View在一定延迟之后再执行APPEARING出现；相反地，一个View消失的时候，它需要先执行DISAPPEARING动画消失，而其他的View需要先等它消失后再执行CHANGE_DISAPPEARING，占据空出的位置。（<font color="red">这里的逻辑可以参考LayoutTransition的构造函数来理解</font>）

熟悉了LayoutTransition基本概念之后，我们看下如何使用布局动画，最简单的方式就是在布局资源文件中，添加`android:animateLayoutChanges="true"`属性。这样，Android系统会帮我们设置默认的5种动画。
当然，我们也可以定制自己的动画，主要包括3步：
1. 首先，需要为不同过渡类型创建属性动画；
2. 然后，通过*** LayoutTransition.setAnimator(int transitionType, Animator animator) *** 方法设置不同类型的属性动画；
3. 最后，通过*** ViewGroup.setLayoutTransition ***方法，把我们定制的LayoutTransition设置到对应的ViewGroup当中。
这样，当ViewGroup发生布局改变时，就会自动触发对应类型的属性动画。

关于LayoutTransition的具体案例，网上有很多资料，这里不再赘述，可以参考[这里](http://www.cnblogs.com/mengdd/p/3305973.html)
#### 实现机制
下面简单学习下LayoutTransition的实现机制：

既然LayoutTransition最终会设置到ViewGroup对象中，并且是在ViewGroup发生布局变化时触发相应的属性动画，那么我们就从ViewGroup添加子View开始分析。

通过阅读这部分代码，可知，ViewGroup.addView最终会调用到**ViewGroup.addViewInner**方法，而该方法，有如下逻辑：
``` java
if (mTransition != null) {
     mTransition.addChild(this, child);
   }
```
即会调用我们设置的LayoutTransition的addChild方法，接下来会继续流转到**LayoutTransition.addChild(ViewGroup parent, View child, boolean changesLayout)**方法。该方法的关键代码如下所示：
``` java
    //changesLayout表示是否会改变ViewGroup的布局。这里，若ViewGroup的布局发生改变，并且是CHANGE_APPEARING类型的动画，则调用runChangeTransition方法，改变其他子View的layout.也验证了上面添加子View时，会先触发CHANGE_APPEARING动画，为子View腾出位置的逻辑。若ViewGroup的布局不发生改变，则不触发Change动画。
    if (changesLayout && (mTransitionTypes & FLAG_CHANGE_APPEARING) == FLAG_CHANGE_APPEARING) {
        runChangeTransition(parent, child, APPEARING);
    }
    if ((mTransitionTypes & FLAG_APPEARING) == FLAG_APPEARING) {
        //为新的子View添加APPEARING动画
        runAppearingTransition(parent, child);
    }
```
首先，来看下runAppearingTransition（负责*APPEARING*类型的动画）方法，该方法比较简单、直接:

``` java
    //拷贝APPEARING属性动画，设置相关的参数
    Animator anim = mAppearingAnim.clone();
    anim.setTarget(child);
    anim.setStartDelay(mAppearingDelay);
    anim.setDuration(mAppearingDuration);
    //设置插值器
    if (mAppearingInterpolator != sAppearingInterpolator) {
        anim.setInterpolator(mAppearingInterpolator);
    }
    if (anim instanceof ObjectAnimator) {
        ((ObjectAnimator) anim).setCurrentPlayTime(0);
    }
    //这里主要是添加监听器，处理动画结束时的通知
    anim.addListener(new AnimatorListenerAdapter() {
        @Override
        public void onAnimationEnd(Animator anim) {
            currentAppearingAnimations.remove(child);
            if (hasListeners()) {
                ArrayList<TransitionListener> listeners =
                        (ArrayList<TransitionListener>) mListeners.clone();
                for (TransitionListener listener : listeners) {
                    listener.endTransition(LayoutTransition.this, parent, child, APPEARING);
                }
            }
        }
    });
    anim.start();
```
然后，看下runChangeTransition（负责处理3种change事件）方法，该方法会处理**APPEARING、DISAPPEARING和CHANGING**3种类型的动画，基本逻辑是：

1. 根据不同的动画类型，选择相应的动画对象和动画持续时间。
2. 通过setupChangeAnimation方法来处理所有子View的change动画
3. 这一步主要是清理第二步中为不同子View添加的View.OnLayoutChangeListener监听器。（主要是清除那些布局没有发生变化的子View的监听器，布局发生变化的子View的监听器会在对应的Change动画结束后，被主动删除掉）

其中，第二步是处理Change事件的关键，这里的代码比较长，但是并不复杂，

1. 获取对应类型的动画。 
2. <font color="red">设置目标对象，从目标对象获取各个属性的初始值，**作为动画的初始值**。</font>
``` java
  // Set the target object for the animation
  anim.setTarget(child);
	// A ObjectAnimator (or AnimatorSet of them) can extract start values from
  // its target object（从Target对象提取出初始值，这是子View的layout改变前的属性值）
  anim.setupStartValues();
```
3. 为每个子View添加View.OnLayoutChangeListener监听器（该监听器只有在目标View的Layout改变之后，才会触发）。在该监听器里面会设置对应动画的持续时间和延迟时间，<font color="red">最重要的是要获取Layout改变后的各个属性值，**作为动画的结束值**，从这里可以看出，对于Change动画来说，之前设置的各个属性的开始值和结束值是没有意义的，在这里都会被替换成真实的开始值和结束值！！！</font>
4. 把当前LayoutTransition对象添加到ViewRootImpl.mPendingTransitions队列中（ViewGroup.requestTransitionStart -> ViewRootImpl.requestTransitionStart），待下次Vsync信号到来时，才真正地开始动画。可参考ViewRootImpl.performTraversals（执行View重绘的地方）方法，关键代码如下所示：
``` java
if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
    for (int i = 0; i < mPendingTransitions.size(); ++i) {
           //调用LayoutTransition.startChangingAnimations启动Change动画。
           mPendingTransitions.get(i).startChangingAnimations();
        }
    mPendingTransitions.clear();
}
```
对于Change动画的处理逻辑可以简单理解为：
<font color="blue" > ** 当我们得知父ViewGroup的布局发生变更时，我们并不能明确具体哪一个子View的Layout布局会发生改变。因此，我们会为每一个子View添加一个View.OnLayoutChangeListener监听器，并在该监听器内完成对应Change动画的设置(`主要是获取改变后Layout参数值，作为动画结束时的属性值`)。这样对于那些Layout布局发生变化的子View来说，它的Change动画就会得到执行；而那些Layout布局没有发生变化的子View，根本就没有必要启动Change动画。**</font>

至此，添加子View导致的APPEARING动画和CHANGE_APPEARING动画的执行流程都分析完了，删除子View的流程和此类似，此处不再赘述。

下面我们看下通过View.setVisibility()改变View的可见性时的流程是怎样的。通过阅读ViewGroup的代码，可知，ViewGroup.onChildVisibilityChanged方法负责View可见性的变更处理，关键代码如下所示：
``` java
if (newVisibility == VISIBLE) {
     mTransition.showChild(this, child, oldVisibility);
   } else {
     mTransition.hideChild(this, child, newVisibility);
 }
```
根据View变更之后的可见性，分别调用LayoutTransition的showChild和hideChild方法，而showChild方法居然调用到了LayoutTransition.addChild方法，hideChild方法也调用了LayoutTransition.removeChild方法。下面的流程就和上面分析的添加子View的流程相同了。

其实，仔细想想，改变子View的可见性带来的UI上的行为和添加删除子View基本一致，唯一的一点不同就是设置为INVISIBLE属性后，子View还是占据了原来的位置，这样其他子View的Layout也不会发生改变，也就不用生成Change动画了。这在LayoutTransition.addChild方法里，会根据changesLayout参数，进行判断要不要触发Change动画，可参加上面LayoutTransition.addChild方法的介绍。

至此，LayoutTransition的实现机制简单梳理了一遍，可见，LayoutTransition在底层也是通过Property Animator来实现的。
### LayoutAnimationController
LayoutAnimationController用于为一个ViewGroup里的子View设置动画效果，可以在XML文件中设置，也可以通过Java Code来设置。
#### 使用方式
##### XML资源文件
首先，在anim文件夹下定义layoutAnimation资源文件：
``` xml
<layoutAnimation xmlns:android="http://schemas.android.com/apk/res/android"
    android:delay="0.6"
    android:animationOrder="normal"
    android:animation="@anim/anim_in" />

android:delay表示子View动画之间的时间间隔因子，最终的时间间隔为delay*duration.
android:animationOrder表示子View的显示顺序，random表示随机显示，normal表示顺序显示，reverse表示倒序显示。
android:animation则指向具体的Tween Animation资源文件。关于Tween Animation的XML文件如何定义，在之前的Android动画框架(1)已有介绍，此处不再赘述。
```
然后，在需要设置动画效果的ViewGroup标签上设置如下属性即可：
``` xml
android:layoutAnimation="@anim/layoutAnimation_xx"，即第一步中的layoutAnimation资源文件。
```
##### Java Code
``` java
 //具体的Tween Animation
 ScaleAnimation sa = new ScaleAnimation(0, 1, 0, 1);
 sa.setDuration(1000);
 //根据具体Tween Animation和动画时间间隔因子创建LayoutAnimationController
 LayoutAnimationController lac = new LayoutAnimationController(sa, 0.6f);
 //设置控件显示顺序
 lac.setOrder(LayoutAnimationController.ORDER_REVERSE);
 //设置到具体的ViewGroup
 viewGroup.setLayoutAnimation(lac);
```
从具体使用方式，可知，我们仅仅为LayoutAnimationController设置了一种动画，因此，ViewGroup的每一个子View实际使用的是相同的动画，只不过每个子View的动画开始时间不同，LayoutAnimationController会根据每个子View的索引值，计算出不同的延迟时间，以实现多个子View的序列化显示。

关于LayoutAnimationController的使用，网上的资料很多，此处不再赘述，可以参考[这里](http://blog.csdn.net/imdxt1986/article/details/6952943)。
#### 实现机制
从上面的使用方式，可知，LayoutAnimationController是借助Tween Animation来对子View实施动画效果的。因此，LayoutAnimationController的实现应该和Tween Animation类似，仔细阅读了这部分代码，发现在ViewGroup.dispatchDraw方法实现了LayoutAnimationController机制。关键代码如下所示:
``` java
    //若为ViewGroup设置了mLayoutAnimationController,则应用布局动画
    if ((flags & FLAG_RUN_ANIMATION) != 0 && canAnimate()) {
       for (int i = 0; i < childrenCount; i++) {
            final View child = children[i];
            //只有可见的子View才会有动画效果。
            if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE) {
                final LayoutParams params = child.getLayoutParams();
                //这里主要是为子View的布局参数params设置layoutAnimationParameters属性，该属性类是LayoutAnimationController的内部类，封装了子View的索引值和子View的个数，在计算每个子View动画的延迟时间时，会用到。
                attachLayoutAnimationParameters(child, params, i, childrenCount);
                //这里是为每个子View绑定Tween Animation
                bindLayoutAnimation(child);
                if (cache) {
                    child.setDrawingCacheEnabled(true);
                    if (buildCache) {
                        child.buildDrawingCache(true);
                    }
                }
            }
        }

       final LayoutAnimationController controller = mLayoutAnimationController;
       controller.start();
     }
```
上面的代码，bindLayoutAnimation方法是关键，负责为每个子View绑定Tween Animation，这里是不是跟Tween Animation的实现机制很像，也是为View绑定补间动画。然后，补间动画的实现就是在下面的drawChild方法中实现，可以参考Android动画框架一。

可见，实现LayoutAnimationController机制的关键就是为每个子View设置合适的Tween Animation，然后每个Tween Animation的实现就交给View系统了。
下面我们看下bindLayoutAnimation方法（如何为每个子View设置补间动画）：
``` java
private void bindLayoutAnimation(View child) {
  //获取每个子View的动画，然后设置到对应子View。
  Animation a = mLayoutAnimationController.getAnimationForView(child);
  child.setAnimation(a);
}
```
上述代码很简单，主要是获取每个子View的动画，然后设置到对应子View。接着看下LayoutAnimationController.getAnimationForView方法：
``` java
  public final Animation getAnimationForView(View view) {
    //获取每个子View动画的延迟执行时间。
    final long delay = getDelayForView(view) + mAnimation.getStartOffset();
    //保存最大延迟时间，判断整个动画是否结束时有用。
    mMaxDelay = Math.max(mMaxDelay, delay);

    try {
         //copy一个之前设置的动画，可见所有子View拥有相同的动画，只是延迟时间不同
        final Animation animation = mAnimation.clone();
         //设置延迟执行时间。
        animation.setStartOffset(delay);
        return animation;
     } catch (CloneNotSupportedException e) {
        return null;
     }
  }
```
上面的代码思路很清晰，首先获取每个子View动画的延迟执行时间，然后copy一份Tween Animation，为每个子View动画设置不同的延迟时间。其中，getDelayForView方法负责计算每个子View的延迟时间,其关键代码如下所示：
``` java
    //获取配置
    ViewGroup.LayoutParams lp = view.getLayoutParams();
    AnimationParameters params = lp.layoutAnimationParameters;

    if (params == null) {
        return 0;
    }
    //根据时间间隔因子mDelay和子View的索引值，计算延迟时间。
    final float delay = mDelay * mAnimation.getDuration();
    final long viewDelay = (long) (getTransformedIndex(params) * delay);
    final float totalDelay = delay * params.count;

    if (mInterpolator == null) {
        mInterpolator = new LinearInterpolator();
    }

    float normalizedDelay = viewDelay / totalDelay;
    normalizedDelay = mInterpolator.getInterpolation(normalizedDelay);

    return (long) (normalizedDelay * totalDelay);
```
上面的代码，主要是根据时间间隔因子mDelay和子View的索引值，计算出每个子View的延迟时间，还可以根据插值器，来调整延迟时间。getTransformedIndex方法负责计算每个子View的索引值，因为在ORDER_REVERSE、ORDER_RANDOM和ORDER_NORMAL模式下，每个子View的索引值可能是不同的，代码比较简单，这里不再赘述。

这里仅是getDelayForView方法的默认实现，若我们想实现不同的显示顺序，可以重写getDelayForView方法，定制个性化效果。

至此，两种布局动画都已简单介绍完毕，简单总结下：

1. LayoutTransition是在ViewGroup的布局发生变化时，对各个子View执行不同的属性动画，以实现子View的过度效果，不至于特别僵硬的切换。对于同一种类型的LayoutTransition动画，各个子View动画是同时进行的。LayoutTransition在底层是依赖于Property Animator来实现的。
2. LayoutAnimationController一般用在第一次加载ListView或者GridView的时候，希望能有个动画效果，来达到一个很好的过度效果。在后续添加或者删除子View时，都不会再有动画了。各个子View动画拥有不同的延迟时间。底层依赖于Tween Animation来实现。

---

其实，除了这几篇讲的几种动画之外，我们还可以通过View.postOnAnimation和View.postOnAnimationDelayed来实现自定义动画，基本包括以下几个步骤：
1. 通过postOnAnimation方法，发送Runnable，在下一帧到来时，变换属性，实现动画效果。
2. 在Runnable中计算View的各类属性值，更新View的属性。
3. 如果动画还需要继续执行，在Runnable中postOnAnimation到下一个时间帧，以实现循环。否则，直接结束动画。

大致框架如下所示:
``` java
Runnable mAnimationRunnable = new Runnable(){
    @Override  
    public void run()  
    { 
      //计算动画属性值 
      boolean isAnimation = calculateAnimation();  
      //根据最新的动画属性值，修改View的对应属性
      applyForViews();
      if(isAnimation){ //继续执行动画
         mView.postOnAnimation(this);
      }
     };  
mView.postOnAnimation(mAnimationRunnable);//开启动画
```
		
OK，到此为止，我们简单梳理了Frame Animation、Tween Animation、Property Animator和Layout Animation的使用方式和实现机制，其中有一些地方难免有错误，还烦请大神们指点一二哈。

---
title: Android动画框架二
date: 2016-04-27 14:03:56
tags:
- Android
- 动画框架
- Property Animator
categories:
- Android
---
本篇接Android动画框架一,上篇文章主要介绍了Frame Animation和Tween Animation的使用方式和实现机制。本篇主要介绍Property Animator.
## Property Animator
之前我们学习的Frame Animation和Tween Animation是Android3.0之前就存在的动画，但是他们存在着一些不足。因此，从Android3.0开始，Google为Android系统添加了更加强大的动画，即属性动画，它可以改变任何对象的属性，可谓动画界的终极武器哈！Property Animator更改的是对象的实际属性，而Tween Animation改变的则是View的绘制效果，真正的View的属性保持不变。

<!-- more -->

### 使用方式
Property Animator中有两个类可以实现属性的改变，分别是：ObjectAnimator和ValueAnimator，其中ValueAnimator是ObjectAnimator的父类，它是Property Animator的时间引擎，负责计算各个帧的属性值。ValueAnimator定义了属性动画的绝大部分核心功能，包括计算各帧的相关属性值，负责处理属性更新事件，按属性值的类型控制计算规则等。属性动画主要由两部分构成：

1. 计算各帧的相关属性值。
2. 为指定对象设置这些计算后的值。

其中，ValueAnimator只负责第一方面的内容，ObjectAnimator则会完成两部分的内容，因此我们优先选用ObjectAnimator，若由于各种限制无法使用，则选择ValueAnimator。ValueAnimator本身不作用于任何对象，也就是说直接使用它没有任何动画效果。但是，它可以对一个值做动画，然后我们可以监听其动画过程，在动画过程中修改指定对象的属性值，这样也就相当于对我们的对象做了动画，即我们自己来完成第二部分。

下面首先来看通过ObjectAnimator来实现属性动画，也包括XML资源文件和Java Code两种方式：
#### XML资源文件（放在animator目录下）
``` xml 
<objectAnimator xmlns:android="http://schemas.android.com/apk/res/android"
   android:duration="1000" //动画持续时间
   android:propertyName="scaleX" //要改变的属性值
   android:valueFrom="1.0"  //动画开始时的属性值
   android:valueTo="2.0"    //动画结束时的属性值
   android:valueType="floatType" >
</objectAnimator>
```
然后加载资源文件，设置宿主对象，开始动画：
``` java
Animator anim = AnimatorInflater.loadAnimator(Context, R.animator.scalex);  
anim.setTarget(view);  
anim.start()
```
#### Java Code

``` java
ObjectAnimator anim = ObjectAnimator.ofFloat(view,"scaleX",1.0f,2.0f);
anim.setDuration(1000);
anim.start();
```

上面两种方式实现了对View的缩放动画，但是View必须提供SetScaleX方法，才能接受属性的改变。同时，若没有提供动画开始时的属性值，那么还必须提供getScaleX方法，来提供动画起始时的属性值。虽然，两种方式都可以实现属性动画，但是我更喜欢直接Java Code的方式，直观、高效。

要通过`ObjectAnimator`实现属性动画有一些条件必须满足：

1. object必须要提供setXxx方法，如果动画的时候没有传递初始值，那么还要提供getXxx方法，因为系统要去拿xxx属性的初始值（如果这条不满足，程序直接Crash）
2. object的setXxx对属性xxx所做的改变必须能够通过某种方法反映出来，比如会带来ui的改变啥的，一般通过调用invalidate来实现（如果这条不满足，动画无效果但不会Crash）

以上条件缺一不可。

可见，要使用ObjectAnimator，有一些苛刻的条件，但是没有什么事是必然的。针对上述问题，Google告诉我们3种解决方案：

1. 直接的方式：若你有权限，给你的对象加上get和set方法。
2. 方便的方式：用一个类来包装原始对象，间接为其提供get和set方法。
3. 妥协的方式：使用ValueAnimator，自己监听动画过程，实现属性的改变，这也是最灵活的方式。

我们来分析下这三种方案：

1.第一种方式很好理解哈，没有？那就直接Code上去呗，Fuck，我没有权限哈。这种方式看上去很好，但是很多情况下，我们都没有相应的权限。
2.第二种方式很有用，假如我们想动态改变Button的宽度，如果我们使用ObjectAnimator直接操作Button的`width`属性，你会发现根本不起作用。查阅了Button的（其实是继承的TextView的方法）setWidth方法，发现它的作用不是设置View的宽度，而是设置Button的最大宽度和最小宽度。好吧，原来，我们设置View的宽度和高度，是在资源文件里通过`android:layout_width`来完成的，但是这货在View中又没有直接的set和get方法。那就来包装一下吧：
``` java
private static class ButtonWrapper {  
   private View mTarget;  
   public ViewWrapper(View target) {  
      mTarget = target;  
   }  
  
   public int getWidth() {  
      return mTarget.getLayoutParams().width;  
   }  
  
   public void setWidth(int width) {  
      mTarget.getLayoutParams().width = width;  
      mTarget.requestLayout();  
   }  
 }
```
然后通过以下方式，就可以从原来的宽度，在1000ms内，变成宽500px.
``` java
ViewWrapper wrapper = new ViewWrapper(mButton);  
ObjectAnimator.ofInt(wrapper, "width", 500).setDuration(1000).start();
```
3.第三种方式最灵活，我们自己来处理属性的改变，	
``` java
ValueAnimator anim = ValueAnimator.ofInt(0,500);
anim.setDuration(1000);
animator.addUpdateListener(new AnimatorUpdateListener() {  
      @Override  
      public void onAnimationUpdate(ValueAnimator animation) {  
        Float value = (Float) animation.getAnimatedValue();//获取当前属性值  
        mButton.getLayoutParams().width = width;  
       	mButton.requestLayout();   
       }  
    });
anim.start();
```
ObjectAnimator默认只能动态变换一个属性，但是我们也可以通过多种方式来实现同时变换类对象的多个属性，主要包括以下几种：
#### 借助PropertyValuesHolder类来实现
PropertyValuesHolder代表一个属性和具体的属性值集合，通过PropertyValuesHolder就可以构建ObjectAnimator对象了，其实通过ofInt等方法来创建ObjectAnimator对象，最终也会把这些属性值封装成PropertyValuesHolder对象,具体实现方式如下所示：
``` java
PropertyValuesHolder x = PropertyValuesHolder.ofFloat("x",150);
PropertyValuesHolder y = PropertyValuesHolder.ofFloat("y",150);
ObjectAnimator.ofPropertyValuesHolder(myView, x, y).setDuration(1000).start();
```
#### 借助ViewPropertyAnimator类来实现

根据官方文档可知，ViewPropertyAnimator类使用一个单一的Animator对象，对一个View对象的多个属性同时进行并行操作。它的行为非常像ObjectAnimator类，因为它也修改了View对象属性的实际值，但是当多个动画属性同时进行时，它会更加高效。另外，使用ViewPropertyAnimator类的代码更加简洁和易于阅读。
若仅仅动画一个或者两个属性，那么可以直接使用ObjectAnimator。但是若同时动画多个属性，那么ViewPropertyAnimator更加高效和可读，具体代码如下所示，其中myView.animate()获取的就是ViewPropertyAnimator类对象：
``` java
myView.animate().x(150f).y(150f).start();
```
很简单吧，但是该方式必须在api 12及之上的版本才能使用。
#### 借助ValueAnimator类，自己实现多属性变换

我们可以监听ValueAnimator的动画变换进度值，然后把进度值同时作用在多个属性之上，这种方式最灵活。假如：我们要同时改变View的X和Y值，分别从50动画到150，那么可以通过以下方式来实现：
``` java
ValueAnimator valueAnimator = ValueAnimator.ofFloat(0.0f, 1.0f);
valueAnimator.addUpdateListener(new AnimatorUpdateListener() {
   //持有一个FloatEvaluator对象，下面估值的时候使用,关于TypeEvaluator下面会进行介绍  
   FloatEvaluator mEvaluator = new FloatEvaluator();
   @Override  
   public void onAnimationUpdate(ValueAnimator animator) {
      float fraction = (Float)animator.getAnimatedValue();//获得动画进度值
      myView.setX(mEvaluator.evaluate(fraction,50.0f,150.0f));//变换X值
      myView.setY(mEvaluator.evaluate(fraction,50.0f,150.0f));//变换Y值
    }
  }); 
valueAnimator.setDuration(1000).start();  
```
#### 借助AnimatorSet类来实现
AnimatorSet类是一个Animator集合，用于实现多个动画的协同作用，包括多个动画的并行执行、串行执行、延迟执行或者结合多者实现任意的组合，是很灵活的动画操作方案。还是以上面的例子为例：
``` java
ObjectAnimator animX = ObjectAnimator.ofFloat(myView, "x", 50f,150f);
ObjectAnimator animY = ObjectAnimator.ofFloat(myView, "y", 50f,150f);
AnimatorSet animXY = new AnimatorSet();
animXY.setDuration(1000); 
animXY.playTogether(animX,animY);
animSetXY.start();
```
AnimatorSet还可以实现更多的动画组合，例如：通过playSequentially可以实现动画的串行执行；animSet.play(anim1).before(anim2).before(anim3)可以实现先动画anim1，然后同时动画anim2和anim3；animSet.play(anim1).after(anim2)则可以实现先动画anim2，然后动画anim1；关于AnimatorSet的使用可以参考官方文档，此处不再赘述。

关于属性动画，还有一个很重要的类`keyFrame`，它表示一个动画进度值(`经过插值器处理过的进度值`)/属性值，通过它可以定义一个特定时间点的关键帧，而且在两个keyFrame之间还可以定义不同的Interpolator，就好像多个动画的拼接，第一个动画的结束点是第二个动画的开始点,如下所示，为定制myView的Y坐标的动态变换。
``` java
//第一个参数表示动画进度百分比(经过插值器处理过的进度值)，基于0和1之间，第二个参数表示对应的属性值。
Keyframe kf0 = Keyframe.ofFloat(0, 50f);  
Keyframe kf1 = Keyframe.ofFloat(0.25f, 100f);  
Keyframe kf2 = Keyframe.ofFloat(0.5f, 300f);  
Keyframe kf3 = Keyframe.ofFloat(0.75f, 100f);  
Keyframe kf4 = Keyframe.ofFloat(1f, 50f);  
PropertyValuesHolder pvhRotation = PropertyValuesHolder.ofKeyframe("y", kf0, kf1, kf2, kf3, kf4);  
ObjectAnimator anim = ObjectAnimator.ofPropertyValuesHolder(myView, pvhRotation);  
anim.setDuration(1000);
anim.start();
```
上面的动画表示动画开始时X=50，进行到1/4时，X=100，进行到一半时，X=300,进行到3/4时，X=100,最后动画结束时，X又变回了50.

下面来看下估值器Evaluator，它主要负责根据属性的开始值、结束值与动画进度值，计算出当前的属性值。基类为TypeEvaluator，仅仅包括一个方法evaluate(float fraction, T startValue, T endValue)，其中fraction表示动画进度值，基于0和1之间（经过插值器处理过的），startValue表示属性起始值，endValue表示属性结束值。系统提供了4个实现类，分别对应不同的属性类型
* IntEvaluator：属性值类型为int；
* FloatEvaluator：属性值类型为float；
* ArgbEvaluator：属性值类型为十六进制颜色值；
* RectEvaluator：属性值为Rect表示的矩形区域；

若这个4种类型不符合我们的要求，那么我们可以继承TypeEvaluator，实现evaluate方法，来定制自己的估值器，很简单哈。
关于属性动画的使用方式，网上有很多案例，此处不再赘述，下面我们看下属性动画是如何实现的。
### 实现机制
对于属性动画，不管通过哪种方式来实现，总要调用Animator.start()方法开始动画，因此我们从ObjectAnimator.start()方法入手，分析其实现机制。
``` java
    //AnimationHandler负责处理所有的不同状态的Animator，是属性动画的核心。
    AnimationHandler handler = sAnimationHandler.get();
    if (handler != null) {
         //处理当前激活的动画，若他们和当前动画实现相同的效果，则取消这些动画。
        int numAnims = handler.mAnimations.size();
        for (int i = numAnims - 1; i >= 0; i--) {
            if (handler.mAnimations.get(i) instanceof ObjectAnimator) {
                ObjectAnimator anim = (ObjectAnimator) handler.mAnimations.get(i);
                if (anim.mAutoCancel && hasSameTargetAndProperties(anim)) {
                    anim.cancel();
                }
            }
        }
        //处理等待的动画，若他们和当前动画实现相同的效果，则取消这些动画。
        numAnims = handler.mPendingAnimations.size();
        for (int i = numAnims - 1; i >= 0; i--) {
            if (handler.mPendingAnimations.get(i) instanceof ObjectAnimator) {
                ObjectAnimator anim = (ObjectAnimator) handler.mPendingAnimations.get(i);
                if (anim.mAutoCancel && hasSameTargetAndProperties(anim)) {
                    anim.cancel();
                }
            }
        }
        //处理延迟的动画，若他们和当前动画实现相同的效果，则取消这些动画。
        numAnims = handler.mDelayedAnims.size();
        for (int i = numAnims - 1; i >= 0; i--) {
            if (handler.mDelayedAnims.get(i) instanceof ObjectAnimator) {
                ObjectAnimator anim = (ObjectAnimator) handler.mDelayedAnims.get(i);
                if (anim.mAutoCancel && hasSameTargetAndProperties(anim)) {
                    anim.cancel();
                }
            }
        }
    }

    super.start();//调用父类的start方法
```
上面的代码虽然很长，但做的事情很简单，如果当前激活的动画、等待的动画（Pending）和延迟的动画（Delay）中有和当前动画相同的动画，那么就把相同的动画给取消掉。然后调用父类ValueAnimator的start方法，其精简后的代码如下所示：
``` java
    //可见，启动动画的线程必须是Looper线程，
    if (Looper.myLooper() == null) {
        throw new AndroidRuntimeException("Animators may only be run on Looper threads");
    }
    AnimationHandler animationHandler = getOrCreateAnimationHandler();
    animationHandler.mPendingAnimations.add(this);//把当前动画添加到即将执行的动画集合中
    if (mStartDelay == 0) { //若没有延迟，则立即开始动画
        setCurrentPlayTime(0);//这里会计算第一帧动画的属性值，其实这里的逻辑和 animationHandler中的逻辑类似，下面直接分析animationHandler.start方法。
        mPlayingState = STOPPED;
        mRunning = true;
        notifyStartListeners();//通知动画开始的回调
    }
    animationHandler.start();
```
由代码和官方文档可知，被启动的Animator会运行在当前线程中（调用start方法的线程），并且该线程必须是Looper线程。如果该属性动画是对View的属性做动画，那么必须在主线程中调用start方法。（因为要动态修改View的属性，自然要在UI线程），上面的代码最终会调用到AnimationHandler.start方法，这个AnimationHandler并不是Handler，而是一个Runnable。

AnimationHandler.start方法又会调用AnimationHandler.scheduleAnimation方法。而该方法会把AnimationHandler添加到mChoreographer的动画队列中，待Vsync信号到来时，会触发AnimationHandler.run方法的执行。在上一篇文章中我们提到过Choreographer，他负责控制用户的input事件、View的绘制和动画等行为，并且会驱动Frame Animation、Tween Animation的执行，这里又和Property Animator相关来起来，可见这货真是核心啊，别急，我们先把接着把属性动画的流程分析完，最后再来学习下Choreographer，这里只要知道当Vsync信号到来时，会触发AnimationHandler.run方法就可以了。

AnimationHandler.run方法会调用doAnimationFrame方法，我们来看下该方法（精简后）：
``` java
        //上面省略的代码主要是把等待的动画（Pending）、达到执行时间的延迟动画（Delay）添加到激活的动画队列中（mAnimations）,然后在下面统一处理。

        int numAnims = mAnimations.size();
        for (int i = 0; i < numAnims; ++i) {
            //这里又拷贝一份，主要是为了避免并发修改导致的问题
            mTmpAnimations.add(mAnimations.get(i));
        }
        //这里动画的最终执行会交给doAnimationFrame方法来处理，frameTime表示动画执行的时间，他的返回值表示动画是否结束了，true表示动画已经结束。
        for (int i = 0; i < numAnims; ++i) {
            ValueAnimator anim = mTmpAnimations.get(i);
            if (mAnimations.contains(anim) && anim.doAnimationFrame(frameTime))                     //若动画已经结束，直接添加到结束动画列表中
                mEndingAnims.add(anim);
        }
        mTmpAnimations.clear();
        if (mEndingAnims.size() > 0) {
            for (int i = 0; i < mEndingAnims.size(); ++i) {
                mEndingAnims.get(i).endAnimation(this);//触发动画结束的回调
            }
            mEndingAnims.clear();
        }

        // If there are still active or delayed animations, schedule a future call to onAnimate to process the next frame of the animations.
        //若还有激活的或者延迟的动画，那么继续调用scheduleAnimation方法，执行下一帧动画。
        if (!mAnimations.isEmpty() || !mDelayedAnims.isEmpty()) {
            scheduleAnimation();
        }
```
上面的代码有两个关键点：
1. 最终是通过doAnimationFrame方法来完成属性动画；
2. 若还有动画需要执行，会再调用scheduleAnimation方法，直到所有的动画都结束了。

下面我们来继续看下，最终的属性值改变到底发生在哪里？  
doAnimationFrame方法会调用animationFrame方法，该方法的关键代码如下所示：
``` java
    //根据动画的执行时间，计算出当前的动画时间进度值。
    float fraction = mDuration > 0 ? (float)(currentTime - mStartTime) / mDuration : 1f;
    if (fraction >= 1f) { //若动画结束了，则检查下是否达到了repeat次数
          if (mCurrentIteration < mRepeatCount || mRepeatCount == INFINITE) {
              // Time to repeat，若还需要继续repeat，那么先通知onAnimationRepeat回调
              if (mListeners != null) {
                   int numListeners = mListeners.size();
                   for (int i = 0; i < numListeners; ++i) {
                       mListeners.get(i).onAnimationRepeat(this);
                   }
              }
              if (mRepeatMode == REVERSE) { //检查下是正向还是反向重复
                 mPlayingBackwards = !mPlayingBackwards;
              }
              mCurrentIteration += (int)fraction;
              fraction = fraction % 1f;
              mStartTime += mDuration;
            } else {
              done = true;
              fraction = Math.min(fraction, 1.0f);
            }
        }
        if (mPlayingBackwards) { //若是反向重复，则处理下时间进度值
            fraction = 1f - fraction;
        }
        animateValue(fraction);//负责计算动画帧的属性值
```
接下来看下animateValue方法，如下所示：
``` java
    //根据设置的插值器，把时间进度值转换为动画属性进度值，关于插值器可以参考上一篇文章。
    fraction = mInterpolator.getInterpolation(fraction);
    mCurrentFraction = fraction;
    int numValues = mValues.length;
    //计算每个属性的属性值
    for (int i = 0; i < numValues; ++i) {
        mValues[i].calculateValue(fraction);
    }
    //回调通知当前的动画属性值，即是我们我们上面在使用ValueAnimator时，监听的动画帧属性值。
    if (mUpdateListeners != null) {
        int numListeners = mUpdateListeners.size();
        for (int i = 0; i < numListeners; ++i) {
            //其实，到这里ValueAnimator的任务已经完成了，即通知了当前的动画属性值
            mUpdateListeners.get(i).onAnimationUpdate(this);
        }
    }
```
上述代码中calculateValue方法就是计算每帧动画所对应的属性的值,该方法最终会调用到对应PropertyValuesHolder（表示一个具体的属性，前面有介绍）中mKeyframes（表示一个属性的多个属性值集合，其实就是集合了多个Keyframe，关于Keyframe前面已有介绍）属性的getValue方法。Keyframes是一个接口，根据不同的属性值类型，有不同的实现，下面我们看下IntKeyframeSet类的getValue方法实现，如下所示：
``` java
 public int getIntValue(float fraction) {
    if (mNumKeyframes == 2) { //处理只有两个关键帧的动画
        if (firstTime) {
            firstTime = false;
            firstValue = ((IntKeyframe) mKeyframes.get(0)).getIntValue();
            lastValue = ((IntKeyframe) mKeyframes.get(1)).getIntValue();
            deltaValue = lastValue - firstValue;
        }
        if (mInterpolator != null) { //前面已经通过插值器处理过了，这里还可以进一步处理
            fraction = mInterpolator.getInterpolation(fraction);
        }
        //根据估值器来获得最终的属性值
        if (mEvaluator == null) {
            return firstValue + (int)(fraction * deltaValue);
        } else {
            return ((Number)mEvaluator.evaluate(fraction, firstValue, lastValue)).intValue();
        }
    }
    //中间省略的是处理动画进度小于等于0或者大于等于0的情况，这里不是很理解为什么有大于1或者小于0的情况？还需要仔细考虑下

    //处理多个关键帧的情况，这里会根据每个Keyframe的插值器，对动画进度值再次进行调整。
    IntKeyframe prevKeyframe = (IntKeyframe) mKeyframes.get(0);
    for (int i = 1; i < mNumKeyframes; ++i) {
        IntKeyframe nextKeyframe = (IntKeyframe) mKeyframes.get(i);
        if (fraction < nextKeyframe.getFraction()) {
            final TimeInterpolator interpolator = nextKeyframe.getInterpolator();
            if (interpolator != null) {
                fraction = interpolator.getInterpolation(fraction);
            }
            //根据每个Keyframe的插值器，再次处理关键帧之间的动画进度值变换
            float intervalFraction = (fraction - prevKeyframe.getFraction()) /
                (nextKeyframe.getFraction() - prevKeyframe.getFraction());
            int prevValue = prevKeyframe.getIntValue();
            int nextValue = nextKeyframe.getIntValue();
            //根据估值器来获得最终的属性值
            return mEvaluator == null ?
                    prevValue + (int)(intervalFraction * (nextValue - prevValue)) :
                    ((Number)mEvaluator.evaluate(intervalFraction, prevValue, nextValue)).
                            intValue();
        }
        prevKeyframe = nextKeyframe;
    }
    // shouldn't get here
    return ((Number)mKeyframes.get(mNumKeyframes - 1).getValue()).intValue();
}
```
上面的代码虽然很长，但是很简单，基本上就是根据每个Keyframe的插值器，再次处理关键帧之间的动画进度值变换，以获得最终的动画属性值。

到这里为止，属性动画的分析基本结束了，但是还有一个点没有明确：ObjectAnimator在哪里对target设置的属性值。

仔细回顾了ValueAnimator的整个流程，发现ObjectAnimator重写了ValueAnimator的animateValue方法，也就是计算动画属性值的地方。ObjectAnimator.animateValue方法首先会调用父类的方法完成动画属性值的计算，然后会调用每个属性PropertyValuesHolder.setAnimatedValue方法，完成对对象属性的赋值。setAnimatedValue的代码如下所示:
``` java
void setAnimatedValue(Object target) {
    if (mProperty != null) { //通过mProperty来赋值
        mProperty.set(target, getAnimatedValue());
    }
    if (mSetter != null) {
        try {
            //获取当前属性值
            mTmpValueArray[0] = getAnimatedValue();
            //反射set方法对target对象赋值
            mSetter.invoke(target, mTmpValueArray);
        } catch (InvocationTargetException e) {
            Log.e("PropertyValuesHolder", e.toString());
        } catch (IllegalAccessException e) {
            Log.e("PropertyValuesHolder", e.toString());
        }
    }
}
```
---

OK，属性动画的整个流程，我们已经完整的走下来了。
对于Property Animator，我们简单总结下几个关键点：
1. Property Animator更改的是对象的实际属性，而Tween Animation改变的则是View的绘制效果，真正的View的属性保持不变。
2. Property Animator中的插值器（Interpolator）和估值器（TypeEvaluator）很重要，它是实现多种变换速率的关键，我们应该学习定制自己插值器和估值器。
3. Property Animator计算属性值可以简单理解成3步：
		1. 首先，根据动画已进行的时间和动画持续（duration），计算出一个时间因子（0~1）。
		2. 然后，根据TimeInterpolator计算出动画进度因子（0~1）。
		3. 最后，根据TypeEvaluator结合动画进度因子计算出属性值。 
4. 虽然Property Animator在Android3.0之上才可以使用，但是我们可以借助NineOldAndroids开源项目来覆盖各个版本。

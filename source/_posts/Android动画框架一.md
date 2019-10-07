---
title: Android动画框架一
date: 2016-04-17 22:04:49
tags:
- Android
- 动画框架
- Frame Animation
- Tween Animation
categories:
- Android
---
最近，抽空看了下Android中的常用动画使用方式和实现机制，现整理如下：
## Drawable Animation(Frame Animation)
帧动画，就是依次展示一系列Drawable，来模拟动画的效果，类似于GIF图片。
### 使用方式
帧动画的使用方式比较简单，可以通过XML资源文件或者Java Code来实现（对应于AnimationDrawable类）。

#### XML资源文件 (放在anim或者Drawable目录下都可以)

``` xml 
<animation-list xmlns:android="http://schemas.android.com/apk/res/android" android:oneshot="true">
    <item android:drawable="@drawable/one" android:duration="100" />
    <item android:drawable="@drawable/two" android:duration="200" />
    <item android:drawable="@drawable/three" android:duration="300" />
</animation-list>
```

其中android:oneshot表示是否仅播放一次，android:drawable表示每一帧图片，android:duration表示对应Drawable展示的持续时间。

#### Java Code

``` java
	AnimationDrawable anim = new AnimationDrawable();
	anim.addFrame(getDrawable(R.drawable.one),100);
	anim.addFrame(getDrawable(R.drawable.two),200);
	anim.addFrame(getDrawable(R.drawable.three),300);
  //Draw区域，必须指定
	anim.setBounds(Rect);
```

习惯上，我们把AnimationDrawable设置为View的背景，接着我们可以在Java Code中获取AnimationDrawable对象，然后通过start和stop方法来控制动画的播放。

<!-- more -->

### 实现机制
至于AnimationDrawable是如何实现动画的？我首先想到的是：通过Handler根据每帧图片的持续时间，循环发送Message，定期处理Message，取出不同帧的Drawable进行draw。

仔细看了下源码，基本逻辑很类似，方法调用栈可以概括为:
``` java
start -> run -> nextFrame -> setFrame -> scheduleSelf(Drawable)
```
scheduleSelf方法会调用Drawable的Callback.scheduleDrawable方法。然后就是寻找哪里设置了callback属性，我一开始一直在AnimationDrawable的继承体系中寻找，但是只在AnimationDrawable的父类DrawableContainer中找到了Drawable.Callback的实现，但是这里根本就没有消息处理逻辑。后来请教了董大师，原来是在View.setBackgroundDrawable的时候，为Drawable设置了Callback回调,View.scheduleDrawable的代码如下所示：
``` java
public void scheduleDrawable(Drawable who, Runnable what, long when) {
    if (verifyDrawable(who) && what != null) {
        final long delay = when - SystemClock.uptimeMillis();
        if (mAttachInfo != null) { //通过垂直同步信号(Vsync)触发下一帧的绘制
            mAttachInfo.mViewRootImpl.mChoreographer.postCallbackDelayed(Choreographer.CALLBACK_ANIMATION, what, who,Choreographer.subtractFrameDelay(delay));
            } else { //通过Handler消息机制触发下一帧的绘制
            ViewRootImpl.getRunQueue().postDelayed(what, delay);
            }
      }
}
```
代码很简洁，优先选择通过垂直同步信号来处理下一帧的绘制，若mAttachInfo为null，再通过Handler消息机制实现下一帧的绘制。Vsync信号的处理是在`Choreographer`类中，而Choreographer则负责控制用户的input事件、View的绘制和动画等行为，后续我们会详细分析该类。

>Frame Animation的主要缺点是：需要准备每一帧的图片，内存占用较大；并且只能用作View的前景和背景图，局限性比较大。

## Tween Animation（View Animation）

Tween Animation具有3个基本属性：开始帧、结束帧和动画持续时间，系统会根据这3个属性，计算出中间帧，实现渐变的效果。Tween Animation只能作用在View元素上，对于普通的类对象无能为力。

### 使用方式
补间动画包括4种变换，如下所示，分别指出了XML资源文件中的标签和对应的Java类：
<img src="http://7xs2qy.com1.z0.glb.clouddn.com/tween.png" width="620" height="250">
Tween Animation可以通过Xml资源文件来定义，也可以直接创建对应的对象来操作，这里以位移动画为例简单介绍其使用方式。

#### XML资源文件（放在anim目录下）

``` xml
<set xmlns:android="http://schemas.android.com/apk/res/android">
  <translate 
	  android:fromXDelta="20" 
	  android:toXDelta="60" 
	  android:fromYDelta="20" 
	  android:toYDelta="100" 
	  android:fillAfter="true"
	  android:duration="1000" /> 
</set> 
```
其中，fromXDelta,fromYDelta表示动画开始时X,Y座标;toXDelta,toYDelta表示动画结束时X,Y的座标；android:fillAfter="true"，表示这个动画执行完之后保持最后的状态；android:duration表示动画持续的时间。

然后就是加载资源文件，播放动画:
``` java
TranslateAnimation animation = AnimationUtils.loadAnimation(context,R.anim.XXX);
view.startAnimation(animation);
```
#### Java Code

``` java
TranslateAnimation animation = new TranslateAnimation(fromXDelta,toXDelta,fromYDelta,toYDelta);//创建位移动画，指定开始和结束的位移位置。
animation.setFillAfter(true);//表示这个动画执行完之后保持最后的状态
animation.setDuration(1000);//动画持续时间
view.startAnimation(animation);
```
除了使用单一的动画外，我们还可以通过AnimationSet来组合多个动画，并且可以为不同动画设置不同的startOffset时间，以实现不同动画之间的先后顺序。

因为AnimationSet也是继承Animation，所以针对Animation的属性也可以添加到AnimationSet上，但是这些属性对于AnimationSet来说具有不同的含义：

>duration, repeatMode, fillBefore和fillAfter，这四个属性，AnimationSet会把它们Push给他的子元素，也就是具体的补间动画。

>repeatCount, fillEnabled，AnimationSet将忽略这两个属性。

>startOffset, shareInterpolator，AnimationSet可以识别这两个属性，startOffset表示动画开始的延迟时间，shareInterpolator表示是否所有的子元素同享相同的插值器（关于Interpolator，下面会进行详细介绍）
	
**Tips**:*在Android4.0之前只能通过Java Code设置上述属性，XML文件设置会被忽略*。

### 实现机制
关于Tween Animation的使用，网上的例子很多，这里不再赘述，下面来分析下Tween的实现机制。
首先从View.startAnimation(animation)入手，这里的代码很简单，如下所示：
``` java
public void startAnimation(Animation animation) {
    //设定开始时间
    animation.setStartTime(Animation.START_ON_FIRST_FRAME);
    //保存Animation到View的mCurrentAnimation属性中
    setAnimation(animation);
    //通知父View清除相关的缓存
    invalidateParentCaches();
    invalidate(true);//请求重绘
}
```
从这里可以得知，View仅仅保存了Tween Animation，然后请求View重绘来实现动画。
既然Tween Animation动画是通过View重绘来实现的，那么我们简单了解下View的绘制流程，可以参考View.draw方法，基本包括6步：
		
1. Draw the background，通过View.drawBackground方法来实现
2. If necessary, save the canvas' layers to prepare for fading,如果需要，保存画布（canvas）的层为淡入或淡出做准备
3. draw the content，通过View.onDraw方法来实现，一般我们实现自己的View，就是通过该方法来操作，获得Canvas后，可以draw任何view，实现个性化的定制。
4. draw the children，通过View.dispatchDraw方法来实现，ViewGroup都会实现该方法，来绘制自己的孩子，这里也是实现Tween Animation的关键。参看 ViewGroup的代码，可知调用过程为：`dispatchDraw->drawChild->child.draw(canvas,parent,drawingTime)->child.draw(canvas)` ，这样的调用过程可以保证每个子View的draw函数都被调用，通过这种递归，从而让整个View树中的所有View的内容都得到绘制。*** 在调用每个子View的draw函数之前，View的绘制位置是Canvas通过translate函数来进行切换的，坐标原点切换到了每个子View的左上角窗口中的所有View 共用一个Canvas对象 ***。
5. If necessary, draw the fading edges and restore layers,如果需要，绘制淡入淡出相关的内容并恢复保存的画布所在层（layer）
6. draw decorations (scrollbars)，通过View.onDrawScrollBars方法来实现，绘制滚动条的操作就是在这里实现的。

既然View的绘制离不开这几步操作，那么就需要看看具体哪一步操作完成了Tween Animation的绘制，仔细分析了各部分代码后，发现了补间动画的基本绘制流程。

首先，从ViewGroup.dispatchDraw方法开始，该方法会绘制每一个Child，即进入到ViewGroup.drawChild方法中，然后会调用View.draw(Canvas canvas, ViewGroup parent, long drawingTime)方法绘制具体的子View，这里就是实现Tween Animation的地方。极度精简后的关键代码如下所示：
``` java
//获取和每个子View绑定的Animation。
final Animation a = getAnimation();
//这个方法实现具体的动画操作，会调用到每个XXXAnimation子类的applyTransformation方法，applyTransformation方法会把每个动画帧对View的转换保存在Transformation类的Matrix和Alpha属性中，下面会详细分析该方法的实现，这里我们仅需要明确，此方法会把动画的转换保存在了Transformation中就可以了。
more = drawAnimation(parent, drawingTime, a, scalingRequired);
//transformToApply就是保存具体动画帧的转换信息类。
transformToApply = parent.getChildTransformation();
//这里主要是进行坐标系的转换，mLeft和mTop就是该子View在父ViewGroup中的位置，在Onlayout方法中指定，这里会把坐标系从父ViewGroup的左上角移动到子View的左上角，这点非常重要，重绘动画发生在子View自己的坐标系中，即子View的左上角是坐标原点。
if (offsetForScroll) {
    canvas.translate(mLeft - sx, mTop - sy);
    } else {
        if (!usingRenderNodeProperties) {
            canvas.translate(mLeft, mTop);
            }
        }
    // Undo the scroll translation, apply the transformation matrix,then redo the scroll translate to get the correct result.
    canvas.translate(-transX, -transY);//撤销滚动距离
    //把XXXAnimation所做的matrix改变，添加到当前Canvas Matrix上。因为每次重绘时drawingTime都不一样，所以每次的Matrix都不同，所以就实现了动画效果。
    canvas.concat(transformToApply.getMatrix());
    canvas.translate(transX, transY);//重做滚动
    //实现透明度渐变的动画           
    float transformAlpha = transformToApply.getAlpha();
    alpha *= transformAlpha;
```
上面只是实现了对Canvas的转换，下面还会通过draw(Canvas canvas)方法实现子View的具体绘制，此处不再赘述，可以参考View的绘制流程。

上面是Tween Animation的大体绘制流程，但是貌似还没有和我们的XXXAnimation相关联起来，下面我们再来分析下drawAnimation方法，就会涉及到具体Animation了,关键代码如下所示：
``` java
//从父ViewGroup中取得变换（平移、旋转或缩放等）信息类Transformation，它包含了一个矩阵Matrix和alpha值，Matrix就是图形转换矩阵。
final Transformation t = parent.getChildTransformation();
//drawingTime表示当前的绘制时间。more表示动画是否结束，若动画没有结束就返回true，直到动画结束返回false，这里的参数a就是从子View中取出的具体XXXAnimation了。XXXAnimation对View所做的转换，会保存在Transformation中的Matrix矩阵和alpha属性中，供上面的draw方法来实现对Canvas的转换。
boolean more = a.getTransformation(drawingTime, t, 1f);

该方法剩余的代码，会判断动画是否结束，若没有结束，则会调用invalidate来不断的重绘，直到动画结束，此处不再赘述。
```
接下来，我们继续看下具体实现动画变换的getTransformation方法，关键代码如下所示：
``` java
//获取该动画的延迟执行时间
final long startOffset = getStartOffset();
final long duration = mDuration;
float normalizedTime;
if (duration != 0) {
    //根据当前时间、动画持续时间，计算出当前动画的时间进度百分比，介于0和1之间
    normalizedTime = ((float) (currentTime - (mStartTime + startOffset))) / (float) duration;
    } else {
    // time is a step-change with a zero duration，特殊情况，动画直接结束。
    normalizedTime = currentTime < mStartTime ? 0.0f : 1.0f;
    }
    //判断动画是否结束。
    final boolean expired = normalizedTime >= 1.0f;
    mMore = !expired;
    if ((normalizedTime >= 0.0f || mFillBefore) && (normalizedTime <= 1.0f || mFillAfter)) {
        if (!mStarted) {
            //通知回调，动画开始执行
            fireAnimationStart();
            mStarted = true;
            if (USE_CLOSEGUARD) {
                guard.open("cancel or detach or getTransformation");
            }
        }
    //获得插值器，插值器的作用就是根据上面获得的时间进度百分比，计算出动画的当前进度，以此来实现加速、减速等动画效果，可以用函数f(t)=t来表示，其中t表示时间进度，f(t)表示真实的动画进度，关于插值器，下面会进行详细的介绍。
    final float interpolatedTime = mInterpolator.getInterpolation(normalizedTime);
    //applyTransformation是具体的动画实现过程，由每个XXXAnimation子类负责实现。简单来说，就是传入动画进度，然后该函数会根据动画进度，填充具体的转换矩阵。不同时刻对应不同的转换矩阵，通过该转换矩阵，就可以绘制出变换后的子View，从而实现动画效果。
    applyTransformation(interpolatedTime,outTransformation);
    }
    //若动画过期了，即结束了
    if (expired) {
        //若动画已经执行完了所有的重复行为，则通知回调，动画结束了.否则，继续重复动画行为，并通知回调，repetat开始了。
        if (mRepeatCount == mRepeated) {
            if (!mEnded) {
                mEnded = true;
                guard.close();
                fireAnimationEnd();//end回调
            }
        } else { 
            if (mRepeatCount > 0) {
                mRepeated++;
            }

            if (mRepeatMode == REVERSE) { //这里主要处理是反向重复还是正向重复。
                mCycleFlip = !mCycleFlip;
                }

            mStartTime = -1;
            mMore = true;//若重复还没有执行完，则重新对mMore赋值。

            fireAnimationRepeat();//repeat回调
            }
    return mMore;
```
上面分析了getTransformation方法的主要流程，可知具体的动画变换是在applyTransformation方法中执行的，而该方法在Animation类中没有具体实现，需要在子类中实现，也就是说自定义动画需要实现applyTransformation函数。我们可以看下TranslateAnimation的applyTransformation方法实现，如下所示：
``` java
protected void applyTransformation(float interpolatedTime, Transformation t) {
    float dx = mFromXDelta;//起始的X坐标位移
    float dy = mFromYDelta;//起始的Y坐标位移
    if (mFromXDelta != mToXDelta) {
        //根据动画进度，计算出X坐标上具体的位移量
        dx = mFromXDelta + ((mToXDelta - mFromXDelta) * interpolatedTime);
    }
    if (mFromYDelta != mToYDelta) {
        //根据动画进度，计算出Y坐标上具体的位移量
        dy = mFromYDelta + ((mToYDelta - mFromYDelta) * interpolatedTime);
    }
    //通过图形变换矩阵来实现具体的位移。
    t.getMatrix().setTranslate(dx, dy);
    }
```
上面代码分析了位移动画的具体实现，其他XXXAnimation的实现非常类似，此处不再赘述。
### Interpolator
下面看下插值器（Interpolator）,插值器的基类是TimeInterpolator，它只有一个方法getInterpolation(float input);其中参数，input表示动画的时间进度，屏蔽了duration的差异，介于0和1之间。返回值则表示真实的动画进度，可以用函数f(t)=t来表示，其中t表示时间进度，f(t)表示真实的动画进度。系统提供了很多插值器，我们也可以根据自己的需求来实现自己的插值器。下面分别是线性、加速和减速插值器,可以直观感受下：

* 线性插值器，插值函数为f(t) = t;
![线性插值器](http://7xs2qy.com1.z0.glb.clouddn.com/%E7%BA%BF%E6%80%A7%E6%8F%92%E5%80%BC%E5%99%A8.gif)
* 加速插值器,默认情况下，插值函数为f(t)=t*t;
![加速插值器](http://7xs2qy.com1.z0.glb.clouddn.com/%E5%8A%A0%E9%80%9F%E6%8F%92%E5%80%BC%E5%99%A8.gif)
* 减速插值器,默认情况下，插值函数为f(t)=1 - (1 - t)*(1 - t); 
![减速插值器](http://7xs2qy.com1.z0.glb.clouddn.com/%E5%87%8F%E9%80%9F%E6%8F%92%E5%80%BC%E5%99%A8.gif)

关于Tween Animation，到这里基本分析完了，我们简单总结下：
1. Tween Animation不是通过子View来实现的，而是通过ParentView不断调整ChildView的画布坐标系来实现的，即改变的仅仅是子view的绘制位置。而子View的left、top、right和bottom属性都没有改变，即子View的Layout位置并没有发生改变，因此，`子View的Event接收区域也没有发生改变`。假设：子View的left和top属性都为100，然后有一个位移动画使该子View移动（50，50）,那么当动画发生时，父ViewGroup首先会把子 traslate(left,top)，然后随着不同的动画帧到来，再traslate(deltaX,deltaY)，那么动画结束时，该子View的最终位置就是（150,150）了，但它的left和top还是100.
2. 如果要实现自己的Tween Animation，可以重写Animation.applyTransformation()方法来实现，如果要实现不同的动画变换速率，可以重写TimeInterpolator.getInterpolation(float input)方法来实现。

---

其实，这里还有一个疑问，Tween Animation的重绘频率是多少？简单，自己重写Animation.applyTransformation方法打印出时间间隔就OK了，打印出的log如下所示：
<img src="http://7xs2qy.com1.z0.glb.clouddn.com/animation_log.png" width="400">

由log可知，重绘的频率基本在16ms左右，这和Android系统的Vsync信号正好吻合。上面学习Drawable Animation时，我们提到了Choreographer类，该类负责控制用户的input事件、View的绘制和动画等行为，其实我们调用invalidate方法时，View不会立即重绘，而是等到Vsync信号到来时，由Choreographer驱动View绘制事件来实现View的绘制。而Vsync信号一般是60HZ，差不多是16ms一次信号，所以才显示出Tween Animation的绘制时间间隔为16ms左右。关于Choreographer类，我们在学习属性动画时，再详细的分析。

> Tween Animation的主要缺点：只能针对View体系作变换；只支持透明度、位移、缩放和旋转等动画；改变的只是绘制位置，View的真实属性并没有发生改变，处理Event事件的区域也没有发生变化；只支持动画的并行组合，不支持串行动画。

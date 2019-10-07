---
title: Android Bitmap那些事
date: 2016-05-31 17:45:22
tags:
 - Android
 - Bitmap
categories:
 - Android
---
在平时的开发中，Bitmap是我们接触最多的话题之一，因为它时不时地就来个OOM，让我们猝不及防。因此有必要来一次彻底的学习，搞清楚Bitmap的一些本质。
本文主要想讲清楚两点内容：
1. Bitmap到底占多大内存
2. Bitmap复用的限制

OK，开始之前先介绍下解码图片时的控制类`BitmapFactory.Options`。

<!-- more -->

## BitmapFactory.Options类解析
`Options`是`BitmapFactory`从输入源中decode Bitmap的控制参数，其主要属性可以参见源码，此处给出一些常用属性的用法。

### inMutable
若为true，则返回的Bitmap是可变的，可以作为Canvas的底层Bitmap使用。
若为false，则返回的Bitmap是不可变的，只能进行读操作。
如果要修改Bitmap，那就必须返回可变的bitmap，例如:修改某个像素的颜色值(setPixel)
### inJustDecodeBounds、outWidth、outHeight
获取Bitmap的宽度和高度最好的方式：
若inJustDecodeBounds为true，则不会把bitmap加载到内存(**实际是在Native层解码了图片，但是没有生成Java层的Bitmap**)，只是获取该bitmap的**原始**宽（outWidth）和高（outHeight）。例如：
``` java
BitmapFactory.Options options = new BitmapFactory.Options();
options.inJustDecodeBounds = true;
/* 这里返回的bmp是null */
Bitmap bmp = BitmapFactory.decodeFile(path, options);
int width = options.outWidth;
int height = options.outHeight;
```
这里若`inJustDecodeBounds`为true，则`outWidth`表示没有经过Scale的Bitmap的原始宽高（即我们通过图片编辑软件看到的宽高），否则则为加载到内存后，真实的Bitmap宽高（经过Scale之后的宽高）。
### outMimeType
表示被加载图片的格式，例如：若加载png格式的图片，则为
`image/png`;若加载jpg格式的图片，则为`image/jpeg`。
### inPurgeable、inInputShareable
从API21开始，这两个属性被废弃了。
在API19及以下，则表示以下含义：
若inPurgeable为true，则表示BitmapFactory创建的用于存储Bitmap Pixel的内存空间，可以在系统内存不足时被回收。

当APP需要再次访问Bitmap的Pixel时（例如：绘制Bitmap或是调用getPixel)，系统会再次调用BitmapFactory decode方法重新生成Bitmap的Pixel数组。

inInputShareable表示是否进行深拷贝，与inPurgeable结合使用，inPurgeable为false时，该参数无意义。
若为true，share a reference to the input data (inputstream, array, etc.) ，即浅拷贝。
若为false，must make a deep copy.即深拷贝。
### inPreferredConfig
表示一个像素需要多大的存储空间：
默认为ARGB_8888: 每个像素4字节. 共32位。
Alpha_8: 只保存透明度，共8位，1字节。 
ARGB_4444: 共16位，2字节。 
RGB_565:共16位，2字节，只存储RGB值。
### inBitmap
Android在API11添加的属性，用于重用已有的Bitmap，这样可以减少内存的分配与回收，提高性能。但是使用该属性存在很多限制：
在API19及以上，存在两个限制条件：

>1. 被复用的Bitmap必须是Mutable。违反此限制，不会抛出异常，且会返回新申请内存的Bitmap。
>2. 被复用的Bitmap的**内存大小**（通过Bitmap.getAllocationByteCount方法获得，API19及以上才有）必须大于等于被加载的Bitmap的内存大小。违反此限制，将会导致复用失败，抛出异常IllegalArgumentException(Problem decoding into existing bitmap）

在API11 ~ API19之间，还存在额外的限制：
>1. 被复用的Bitmap的**宽高**必须等于被加载的Bitmap的**原始宽高**。（注意这里是指原始宽高，即没进行缩放之前的宽高）
>3. 被加载Bitmap的Options.inSampleSize必须明确指定为1。
>4. 被加载Bitmap的Options.inPreferredConfig字段设置无效，因为会被被复用的Bitmap的inPreferredConfig值所覆盖（不然，所占内存可能就不一样了）

关于`inBitmap`属性，后面会进行详细的介绍，此处不再赘述。

### inScaled、inDensity、inTargetDensity、inScreenDensity
这几个属性值控制了如何对Bitmap进行`缩放`,以及决定了bitmap的密度`density`。
`inScaled`表示是否进行缩放:
若为true，且inDensity和inTargetDensity不为0，那么缩放因子等于(**inTargetDensity/inDensity**)，并且bitmap的density等于inTargetDensity。
若为false，则不会进行缩放，并且bitmap的density等于inDensity(不为0的前提下),否则就是系统默认密度（160）。

`inScaled`属性默认为true，
当从Drawable资源文件夹加载图片资源时（通过BitmapFactory.decodeResource方法加载），`inDensity`默认初始化为图片所在文件夹对应的密度，而`inTargetDensity`则初始化为当前系统密度。

当从SD卡 or 二进制流加载图片资源时，这两个属性都默认为0（即不会对图片资源进行缩放），需要我们根据实际情况进行设置，一般把`inTargetDensity`设置为当前系统密度，`inDensity`则需要根据图片实际尺寸和需求进行设置了。

### inSampleSize
主要用于获取Bitmap的缩略图，例如：inSampleSize=2，那么bitmap的宽度和高度为原来尺寸的1/2。像素总数则为原来的1/4。Any value <= 1 is treated the same as 1.
看了下代码，在Native层解码生成`SKBitmap`的像素数据时，会根据图片原始宽高除以inSampleSize，得到缩略图的宽高。
## Bitmap的内存占用
首先要明确一点，图片在内存和文件系统中的大小是两个不同的概念。这里我们主要考虑Bitmap占用内存的大小。（文件系统中的大小是单独的话题，后续会进行介绍）
决定Bitmap占用内存大小的关键因素有以下几点：
1. 图片的原始宽高（即我们在图片编辑软件中看到的宽高）
2. 解码图片时的Config配置（即每个像素占用几个字节）
3. 解码图片时的缩放因子（即`inTargetDensity/inDensity`）

所以Bitmap的内存计算公式时
``` 
originWidth * originHeight * (inTargetDensity/inDensity) * (inTargetDensity/inDensity) * 每像素占用字节数
```
其实，Bitmap在API12提供了`getByteCount`方法获取占用内存,如下所示：
``` java
public final int getByteCount() {
    //getHeight()表示Bitmap的宽度（单位px）
    return getRowBytes() * getHeight();
}
```
其中getRowBytes()会调用到Native层处理，其实就是表示一行像素所占的内存大小，即`width * 每像素占用字节数`。

在API19提供了`getAllocationByteCount`方法获取实际占用的内存，如下所示：
``` java
public final int getAllocationByteCount() {
    //mBuffer表示存储Bitmap像素数据的字节数组。
    if (mBuffer == null) {
        return getByteCount();
    }
    return mBuffer.length;
    }
```
`mBuffer.length`实际获取的就是用来存储Bitmap像素数据的字节数组的长度。（若是通过复用其他Bitmap来解码图片，那么这个字节数组存储新Bitmap的像素数据时，可能并没有用完）

一般情况下，两者是相同的。但若是通过复用Bitmap来解码图片，那么前者表示新解码图片占用内存的大小（并非实际内存大小），后者表示被复用Bitmap真实占用的内存大小（即mBuffer的长度）。
来看个例子吧：
``` java
//首先以原尺寸加载图片a，这里图片a和图片b的尺寸相同
BitmapFactory.Options options = new BitmapFactory.Options();
options.inMutable = true;
options.inDensity = 160;
options.inTargetDensity = 160;
Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.a, options);
Log.i(TAG, "bitmap = " + bitmap);
Log.i(TAG, "bitmap.size = " + bitmap.getByteCount());
Log.i(TAG, "bitmap.allocSize = " + bitmap.getAllocationByteCount());

//然后复用a图片，解码b图片。
options.inBitmap = bitmap;
//注意这里解码得到的图片宽高为原始尺寸的一半
options.inDensity = 160;
options.inTargetDensity = 80;
options.inMutable = true;
options.inSampleSize = 1;
Bitmap bitmapAIO = BitmapFactory.decodeResource(getResources(), R.drawable.b, options);
Log.i(TAG, "bitmapAIO = " + bitmapAIO);
Log.i(TAG, "bitmap.size = " + bitmap.getByteCount());
Log.i(TAG, "bitmap.allocSize = " + bitmap.getAllocationByteCount());
Log.i(TAG, "bitmapAIO.size = " + bitmapAIO.getByteCount());
Log.i(TAG, "bitmapAIO.allocSize = " + bitmapAIO.getAllocationByteCount());

输出：
bitmap = android.graphics.Bitmap@9fb5d09
bitmap.size = 8294400
bitmap.allocSize = 8294400

bitmapAIO = android.graphics.Bitmap@9fb5d09
bitmap.size = 2073600
bitmap.allocSize = 8294400
bitmapAIO.size = 2073600
bitmapAIO.allocSize = 8294400
```
从上述demo，可以得出：
1. Bitmap复用成功了，因为bitmap和bitmapAIO是相同的对象。
2. 图片a占用内存8294400，图片b（宽和高各缩小一半）占用内存2073600，正好是图片a所占内存的1/4。
3. getByteCount方法返回当前图片应当所占内存大小，getAllocationByteCount返回被复用Bitmap真实占用内存大小。

## 缩放因子和Bitmap复用限制的由来
在上述计算Bitmap占用内存的公式中，有一个缩放因子，决定了对原始图片Scale多少倍。下面我们看看Android API18和API19两个版本是如何处理对原始图片的缩放操作的（其实就是处理inTargetDensity/inDensity）。同时也从源码层面上，验证下上文提及的不同Android版本对Bitmap复用限制的差异。

因为通过`BitmapFactory`解码图片的方法很多，这里我们选择从Drawable文件夹解码的方法`decodeResource`来进行分析。
### Android4.3 API18
``` java
decodeResource(Resources res, int id, Options opts) {
    Bitmap bm = null;
    InputStream is = null; 
    final TypedValue value = new TypedValue();
    is = res.openRawResource(id, value);
    //关键的代码这里
    bm = decodeResourceStream(res, value, is, null, opts);
    //这里很重要，说明若是Bitmap复用失败，会抛出异常，并返回null值，所以在复用Bitmap时要特别留意。
    if (bm == null && opts != null && opts.inBitmap != null) {
        throw new IllegalArgumentException("Problem decoding into existing bitmap");
    }
    
    return bm;
}
```
下面继续看`decodeResourceStream`方法
``` java
decodeResourceStream(Resources res, TypedValue value,
            InputStream is, Rect pad, Options opts) {
    if (opts == null) {
        opts = new Options();
    }
    //正常情况下，opts.inDensity会被赋值为图片所在Drawable文件表示的屏幕密度。
    if (opts.inDensity == 0 && value != null) {
        final int density = value.density;//图片所在Drawable文件夹决定了density
        if (density == TypedValue.DENSITY_DEFAULT) {
            opts.inDensity = DisplayMetrics.DENSITY_DEFAULT;//特殊情况下，inDensity为默认值160
        } else if (density != TypedValue.DENSITY_NONE) {
            opts.inDensity = density;
        }
    }
    //正常情况下，opts.inTargetDensity会被赋值为手机系统密度
    if (opts.inTargetDensity == 0 && res != null) {
        opts.inTargetDensity = res.getDisplayMetrics().densityDpi;
    }
    //这里是关键代码，继续往下走       
    return decodeStream(is, pad, opts);
}
```
下面继续看`decodeStream`方法（极度精简之后的）：
``` java
boolean finish = true; 
//若是没有复用已有Bitmap，则走这个逻辑，此时会在native层处理对图片的缩放
if (opts == null || (opts.inScaled && opts.inBitmap == null)) {
    float scale = 1.0f;
    int targetDensity = 0;
    if (opts != null) {
        final int density = opts.inDensity;
        targetDensity = opts.inTargetDensity;
        if (density != 0 && targetDensity != 0) {
            scale = targetDensity / (float) density;//求出缩放因子
        }
    }
    //通过native方法解码图片，并在native层完成对图片的缩放
    bm = nativeDecodeStream(is, tempStorage, outPadding, opts, true, scale);
    //设置解码出的Bitmap的密度
    if (bm != null && targetDensity != 0) bm.setDensity(targetDensity);

    finish = false;
} else {
    //若复用已有Bitmap来解码图片，那么就不支持在native层对图片进行缩放
    bm = nativeDecodeStream(is, tempStorage, outPadding, opts);
}
//这里很重要，说明若是Bitmap复用失败，会抛出异常，并返回null值，所以在复用Bitmap时要特别留意。
if (bm == null && opts != null && opts.inBitmap != null) {
    throw new IllegalArgumentException("Problem decoding into existing bitmap");
    }
//若没有在native层完成对图片的缩放，则会通过finishDecode方法在java层完成对图片的缩放（显然这种方式会重新分配内存）。
return finish ? finishDecode(bm, outPadding, opts) : bm;
```
`decodeStream`方法很关键，从中可以获取几点关键信息：
1. 若是直接解码图片（不复用已有图片，即opts.inBitmap为null），那么通过native方法解码图片时，支持把缩放因子作为参数传递到native层，并且后续在java层直接返回了native层解码出来的Bitmap。<font color="#ff0000">**说明在native层处理了对图片的缩放**。</font>
2. 若是通过复用已有Bitmap来解码图片，那么通过native方法解码图片时，就不支持传递缩放因子参数了，并且后续在java层，通过`finishDecode`方法完成了对图片的缩放。<font color="#ff0000">**说明若复用已有Bitmap解码图片，则不支持在native层对图片进行缩放处理，需要在java层单独对图片进行缩放处理**。</font>
>简单概括下：Android API18及之前的版本，不支持在native层同时使用Bitmap复用和进行缩放处理。（API18及之后是可以的，下面会介绍）

下面我们先看下，`finishDecode`方法是怎么在java层完成对Bitmap的缩放处理的，再看下native层的解码方法。
首先看下`finishDecode`方法，如下所示(精简之后)：
``` java
finishDecode(Bitmap bm, Rect outPadding, Options opts) {
    final int density = opts.inDensity;
    if (density == 0) { //检查参数是否合法
        return bm;
    }
    
    bm.setDensity(density);
    final int targetDensity = opts.inTargetDensity;
    //检查是否需要进行缩放处理
    if (targetDensity == 0 || density == targetDensity || density == opts.inScreenDensity) {
            return bm;
    }
    if (opts.inScaled || isNinePatch) {
        float scale = targetDensity / (float) density;//计算缩放因子
        if (scale != 1.0f) {
            final Bitmap oldBitmap = bm;
            //这里很关键哈，根据缩放因子，在原图的基础上，直接创建了一个新的Bitmap,并且把原图给回收了。
            bm = Bitmap.createScaledBitmap(oldBitmap,
                        Math.max(1, (int) (bm.getWidth() * scale + 0.5f)),
                        Math.max(1, (int) (bm.getHeight() * scale + 0.5f)), true);
            if (bm != oldBitmap) oldBitmap.recycle();
        }
        //设置缩放后的Bitmap的密度
        bm.setDensity(targetDensity);
    }
    return bm;
}
```
`finishDecode`方法很简单，就是根据缩放因子，在原来Bitmap的基础上，又新建了一个Bitmap，然后把原图回收了。这里新创建的Bitmap就是最终的Bitmap。这种方式会造成在java层重新分配内存，显然不是很好。所以在Android4.4之后，都是在native层完成对Bitmap的缩放处理的。（也就是Android4.4之后，同时支持复用已有Bitmap和在native层对原图进行缩放处理，后面进行介绍）。
下面看下native层的解码方法：
``` C++
static jobject doDecode(JNIEnv* env, SkStream* stream, jobject padding,
        jobject options, bool allowPurgeable, bool forcePurgeable = false,
        bool applyScale = false, float scale = 1.0f) {
int sampleSize = 1;
bool isMutable = false;
bool willScale = applyScale && scale != 1.0f;
//获取java类Options中的配置
prefConfig = GraphicsJNI::getNativeBitmapConfig(env, jconfig);
isMutable = env->GetBooleanField(options, gOptions_mutableFieldID);
//希望被复用的java层Bitmap
javaBitmap = env->GetObjectField(options, gOptions_bitmapFieldID);
//可见，在native层不支持既进行缩放又进行Bitmap复用。
if (willScale && javaBitmap != NULL) {
    return nullObjectReturn("Cannot pre-scale a reused bitmap");
}
//获取图片解码器，png，jpeg等图片格式都有不同的实现类
SkImageDecoder* decoder = SkImageDecoder::Factory(stream);
decoder->setSampleSize(sampleSize);
//java堆内存分配器
JavaPixelAllocator javaAllocator(env);
SkBitmap* bitmap;
bool useExistingBitmap = false;
if (javaBitmap == NULL) {
    bitmap = new SkBitmap;//若没有复用的Bitmap，则创建一个新的SkBitmap
} else {
    if (sampleSize != 1) { //可见sampleSize必须为1
        return nullObjectReturn("SkImageDecoder: Cannot reuse bitmap with sampleSize != 1");
    }
    //获取被复用的native层Bitmap
    bitmap = (SkBitmap*) env->GetIntField(javaBitmap, gBitmap_nativeBitmapFieldID);
    // only reuse the provided bitmap if it is immutable
    if (!bitmap->isImmutable()) { //可见只有可变的bitmap才能被复用
        useExistingBitmap = true;
        // options.config，会被被复用的bitmap的配置所替代
        prefConfig = bitmap->getConfig();
    } else {
        ALOGW("Unable to reuse an immutable bitmap as an image decoder target.");
        bitmap = new SkBitmap;
    }
}

SkBitmap* decoded;
if (willScale) { //有缩放，说明没有bitmap复用，直接创建新的
    decoded = new SkBitmap;
} else {
    decoded = bitmap;
}
//解码得到原始尺寸的Bitmap
if (!decoder->decode(stream, decoded, prefConfig, decodeMode, javaBitmap != NULL)) {
    return nullObjectReturn("decoder->decode returned false");
}

int scaledWidth = decoded->width();
int scaledHeight = decoded->height();
//计算缩放后的Bitmap的宽度和高度
if (willScale && mode != SkImageDecoder::kDecodeBounds_Mode) {
    //这里在计算缩放后的宽高时，有个精度问题
    scaledWidth = int(scaledWidth * scale + 0.5f);
    scaledHeight = int(scaledHeight * scale + 0.5f);
}
// update options (if any)
if (options != NULL) {
    //这里主要是把Bitmap的宽度，高度和图片类型设置到java层的Options里面
    env->SetIntField(options, gOptions_widthFieldID, scaledWidth);
    env->SetIntField(options, gOptions_heightFieldID, scaledHeight);
    env->SetObjectField(options, gOptions_mimeFieldID,getMimeTypeString(env, decoder->getFormat()));
}

// if we're in justBounds mode, return now (skip the java bitmap)
//可见若java层Options.inJustDecodeBounds为true，则就直接返回了，不在生成java层的bitmap
if (mode == SkImageDecoder::kDecodeBounds_Mode) {
    return NULL;
}

if (willScale) {
    //计算出缩放因子
    const float sx = scaledWidth / float(decoded->width());
    const float sy = scaledHeight / float(decoded->height());
    SkBitmap::Config config = decoded->config();
    switch (config) {
        case SkBitmap::kNo_Config:
        case SkBitmap::kIndex8_Config:
        case SkBitmap::kRLE_Index8_Config:
            config = SkBitmap::kARGB_8888_Config;
            break;
        default:
            break;
        }
        //这里操作的时bitmap变量哈，设置缩放后的宽高
        bitmap->setConfig(config, scaledWidth, scaledHeight);
        bitmap->setIsOpaque(decoded->isOpaque());
        //根据新的宽高，重新分配存储像素数据的空间
        if (!bitmap->allocPixels(&javaAllocator, NULL)) {
            return nullObjectReturn("allocation failed for scaled bitmap");
        }
        bitmap->eraseColor(0);

        SkPaint paint;
        paint.setFilterBitmap(true);
        //这里把解码出来的原始尺寸的decoded Bitmap draw到缩放（sx, sy）倍后的bitmap上，最终完成了对原始图片的缩放
        SkCanvas canvas(*bitmap);
        canvas.scale(sx, sy);
        canvas.drawBitmap(*decoded, 0.0f, 0.0f, &paint);
}
//若复用了bitmap，那么返回被复用的java层bitmap（也就是说复用成功后，返回的Bitmap就是被复用的Bitmap，只不过底层的像素数据都发生了改变）
if (useExistingBitmap) {
    // If a java bitmap was passed in for reuse, pass it back
    return javaBitmap;
}
// now create the java bitmap
//否则，根据缩放后的native层bitmap（SkBitmap）生成java层Bitmap并返回（这里是调用到了java层Bitmap的私有构造函数）
return GraphicsJNI::createBitmap(env, bitmap, javaAllocator.getStorageObj(),
            isMutable, ninePatchChunk, layoutBounds, -1);
}
```
OK，通过上述代码，我们了解到以下几点信息：
1. Android 4.3在native层不支持即复用已有Bitmap，又进行缩放。
2. 若缩放因子为1，那么在进行Bitmap复用时（假设满足复用条件），会直接在被复用Bitmap上进行解码操作（主要是修改被复用Bitmap的像素数据信息），同时返回到java层的就是被复用的Bitmap。（即如果被复用的Bitmap == 返回的被加载的Bitmap，那么说明复用成功了）。
3. 若缩放因子大于1，且没有Bitmap复用，那么首先会解码生成一个图片原始宽高的`SkBitmap`,然后在再根据缩放因子，通过绘制的方式，把原始宽高的Bitmap会绘制到经过Scale之后的`SkCanvas`上，以得到一个缩放后的`SkBitmap`，最后调用java层bitmap的构造方法，创建java层的Bitmap，然后返回到放大调用处。

上面我们提出了几点在Android4.4之前复用Bitmap的限制，在`doDecode`方法中基本都得到了验证。唯独对**宽高必须相等**的限制没有见到。其实这个限制是在解码得到原始宽高的Bitmap时进行的，即上面代码中的`decoder->decode`方法中。这里的decoder解码器（SkImageDecoder是父类型）根据解码不同格式的图片，是不同的实现类对象。下面我们看一下`SkImageDecoder_libpng`类的实现（png格式的解码器）
``` C++
bool SkPNGImageDecoder::onDecode(SkStream* sk_stream, SkBitmap* decodedBitmap,
                                 Mode mode) {
const int sampleSize = this->getSampleSize();
//origWidth和origHeight表示被解码图片的原始宽高
SkScaledBitmapSampler sampler(origWidth, origHeight, sampleSize);
//decodedBitmap->getPexels() 尝试获取位图的像素缓冲区首地址，如果取出来为 NULL，说明这个bitmap是新创建的,因为new SkBitmap只创建对象，不分配保存像素的缓冲区，否则就说明这个是可以复用的位图。
decodedBitmap->lockPixels();
void* rowptr = (void*) decodedBitmap->getPixels();
bool reuseBitmap = (rowptr != NULL);
decodedBitmap->unlockPixels();
//若是复用已有的位图，那么这里就会进行宽高必须相等的判断。
if (reuseBitmap && (sampler.scaledWidth() != decodedBitmap->width() ||
            sampler.scaledHeight() != decodedBitmap->height())) {
    // Dimensions must match
    return false;
}
//若不是复用已有Bitmap，那么就需要设置SkBitmap的宽、高、config等配置信息。
if (!reuseBitmap) {
    decodedBitmap->setConfig(config, sampler.scaledWidth(),sampler.scaledHeight(), 0);
}
//若是Options.inJustDecodeBounds为true，那么到此就结束了，因为上面宽和高已经计算出来了。
if (SkImageDecoder::kDecodeBounds_Mode == mode) {
    return true;
}
//若不是复用已有Bitmap，则说明还没有存储像素数据的内存空间，因此这里需要进行内存分配
if (!reuseBitmap) {
    if (!this->allocPixelRef(decodedBitmap,SkBitmap::kIndex8_Config == config ? colorTable : NULL)) {
        return false;
    }
}
 
//后续代码应该就是真正去解码png格式图片，生成像素数据的核心逻辑了，这块暂时不进行介绍
```
上述代码就是解码png格式图片的关键代码，里面印证了复用Bitmap时，宽高必须相等的说法。

通过上述代码的分析，我们可以有以下（反复强调）结论：
>1. 若是直接解码图片（不复用已有图片，即opts.inBitmap为null），则是在naive层处理对原图的缩放操作。
>2. 若是通过复用已有Bitmap来解码图片，则是在java层处理对原图的缩放操作（finishDecode方法）。

### Android4.4 API19
Android从API19开始，放宽了对Bitmap复用的限制。下面我们看下API19对Bitmap复用的两个限制是在哪里实现的，以及该版本是如何对原图进行缩放的。
从`decodeResource -> decodeResourceStream`方法的实现和API18类似，这里不再赘述，差异点出现在`decodeStream`方法。如下所示(精简后)：
``` java
Bitmap decodeStream(InputStream is, Rect outPadding, Options opts) {
//最终调用到native层进行解码
Bitmap bm = decodeStreamInternal(is, outPadding, opts);
}
//这里很重要，说明若是Bitmap复用失败，会抛出异常，并返回null值，所以在复用Bitmap时要特别留意。
if (bm == null && opts != null && opts.inBitmap != null) {
    throw new IllegalArgumentException("Problem decoding into existing bitmap");
}
//该方法做了finishDecode方法除了缩放Bitmap之外所有的事
setDensityFromOptions(bm, opts);
return bm;
```
API19版本的decodeStream和API18相比，最明显的就是删除了Java层缩放Bitmap的逻辑（finishDecode方法），因此可以确定对Bitmap的缩放处理都是在native层处理的。下面我们继续看下native方法的实现（极度精简）：
``` C++
static jobject doDecode(JNIEnv* env, SkStreamRewindable* stream, jobject padding,
        jobject options, bool allowPurgeable, bool forcePurgeable = false) {
SkBitmap::Config prefConfig = SkBitmap::kARGB_8888_Config;//默认conifg配置
bool isMutable = false;
float scale = 1.0f;
jobject javaBitmap = NULL;
sampleSize = env->GetIntField(options, gOptions_sampleSizeFieldID);
if (optionsJustBounds(env, options)) {
    mode = SkImageDecoder::kDecodeBounds_Mode;//是否只获取图片的宽高
}

if (options != NULL) {
    sampleSize = env->GetIntField(options, gOptions_sampleSizeFieldID);
    if (optionsJustBounds(env, options)) {
        mode = SkImageDecoder::kDecodeBounds_Mode;
    }
    // initialize these, in case we fail later on
    env->SetIntField(options, gOptions_widthFieldID, -1);
    env->SetIntField(options, gOptions_heightFieldID, -1);
    env->SetObjectField(options, gOptions_mimeFieldID, 0);
    //获取一些java层Options的属性值
    jobject jconfig = env->GetObjectField(options, gOptions_configFieldID);
    prefConfig = GraphicsJNI::getNativeBitmapConfig(env, jconfig);
    isMutable = env->GetBooleanField(options, gOptions_mutableFieldID);
    //获取被复用的java层bitmap
    javaBitmap = env->GetObjectField(options, gOptions_bitmapFieldID);
    //若java层options.inScaled为true，则进行缩放处理
    if (env->GetBooleanField(options, gOptions_scaledFieldID)) { 
        const int density = env->GetIntField(options, gOptions_densityFieldID);
        const int targetDensity = env->GetIntField(options, gOptions_targetDensityFieldID);
        const int screenDensity = env->GetIntField(options, gOptions_screenDensityFieldID);
        //这里很关键，缩放因子是targetDensity / density
        if (density != 0 && targetDensity != 0 && density != screenDensity) {
                scale = (float) targetDensity / density;
        }
    }
}

const bool willScale = scale != 1.0f;
SkImageDecoder* decoder = SkImageDecoder::Factory(stream);//创建解码器
//为解码器设置一些属性
decoder->setSampleSize(sampleSize);

SkBitmap* outputBitmap = NULL;
unsigned int existingBufferSize = 0;
if (javaBitmap != NULL) {
    //获取被复用的native层Skbitmap
    outputBitmap = (SkBitmap*) env->GetIntField(javaBitmap, gBitmap_nativeBitmapFieldID);
    if (outputBitmap->isImmutable()) { //API19还存在的限制，被复用的bitmap必须是可变的
        ALOGW("Unable to reuse an immutable bitmap as an image decoder target.");
        javaBitmap = NULL;
        outputBitmap = NULL;
    } else { //获取被复用bitmap存储像素数据的缓冲区大小，即有多少内存可以被复用（作为后续判断是否可被复用的依据）。
        existingBufferSize = GraphicsJNI::getBitmapAllocationByteCount 
    }
}
//若没有可被复用的bitmap，那就直接创建一个。
SkAutoTDelete<SkBitmap> adb(outputBitmap == NULL ? new SkBitmap : NULL);
if (outputBitmap == NULL) outputBitmap = adb.get();
//Java堆像素内存分配器
JavaPixelAllocator javaAllocator(env);
//循环复用的像素内存分配器(接收被复用Bitmap的像素缓冲区首地址和大小作为参数)
RecyclingPixelAllocator recyclingAllocator(outputBitmap->pixelRef（）, existingBufferSize);
//带缩放检查的像素内存分配器
ScaleCheckingAllocator scaleCheckingAllocator(scale, existingBufferSize);
//如果设置了Options.inBitmap就使用 RecyclingPixelAllocator，否则使用默认的JavaPixelAllocator
SkBitmap::Allocator* outputAllocator = (javaBitmap != NULL) ? (SkBitmap::Allocator*)&recyclingAllocator : (SkBitmap::Allocator*)&javaAllocator;
if (decodeMode != SkImageDecoder::kDecodeBounds_Mode) {
    if (!willScale) {
        //若不需要缩放处理，就是用上面的outputAllocator
        decoder->setSkipWritingZeroes(outputAllocator == &javaAllocator);
        decoder->setAllocator(outputAllocator);
    } else if (javaBitmap != NULL) { //否则若复用bitmap，则使用scaleCheckingAllocator
        decoder->setAllocator(&scaleCheckingAllocator);
    }
} 

SkBitmap decodingBitmap;
//解码图片decodingBitmap（这里的decodingBitmap只考虑了sampleSize，但是没有考虑缩放因子）
if (!decoder->decode(stream, &decodingBitmap, prefConfig, decodeMode)) {
    return nullObjectReturn("decoder->decode returned false");
}
//此时解码出来的是图片的原始宽高（但是会考虑sampleSize）
int scaledWidth = decodingBitmap.width();
int scaledHeight = decodingBitmap.height();

if (willScale && mode != SkImageDecoder::kDecodeBounds_Mode) {
    //计算出缩放后的宽度和高度
    scaledWidth = int(scaledWidth * scale + 0.5f);
    scaledHeight = int(scaledHeight * scale + 0.5f);
}
//把最终的宽度/高度/图片格式信息更新到java层Options累里面
if (options != NULL) {
    env->SetIntField(options, gOptions_widthFieldID, scaledWidth);
    env->SetIntField(options, gOptions_heightFieldID, scaledHeight);
    env->SetObjectField(options, gOptions_mimeFieldID,getMimeTypeString(env, decoder->getFormat()));
}

 if (willScale) { //开始进行缩放处理了。
    //首先计算出在X和Y轴上的缩放倍数
    const float sx = scaledWidth / float(decodingBitmap.width());
    const float sy = scaledHeight / float(decodingBitmap.height());
    SkBitmap::Config config = configForScaledOutput(decodingBitmap.config());
    //对最终生成的Bitmap设置配置信息（作为下面分配像素数据的依据哈）
    outputBitmap->setConfig(config, scaledWidth, scaledHeight, 0,decodingBitmap.alphaType());
    //这里为最终的Bitmap分配像素数据
    if (!outputBitmap->allocPixels(outputAllocator, NULL)) {
        return nullObjectReturn("allocation failed for scaled bitmap");
    }
    //把之前解码出来的原始尺寸的decodingBitmap draw到放大(sx,sy)倍之后的canvas上，最终实现了对原图的缩放处理。
    SkPaint paint;
    paint.setFilterLevel(SkPaint::kLow_FilterLevel);
    SkCanvas canvas(*outputBitmap);
    canvas.scale(sx, sy);
    canvas.drawBitmap(decodingBitmap, 0.0f, 0.0f, &paint);
} else { //没有缩放处理，则直接把解码出来的decodingBitmap设置到最终的Bitmap（outputBitmap）中。
    outputBitmap->swap(decodingBitmap);
}

//若复用了bitmap，那么返回被复用的java层bitmap（也就是说复用成功后，返回的Bitmap就是被复用的Bitmap，只不过底层的像素数据都发生了改变）
if (javaBitmap != NULL) {
    ...
    // If a java bitmap was passed in for reuse, pass it back
    return javaBitmap;
}

// now create the java bitmap 否则创建java层的bitmap，并返回。
return GraphicsJNI::createBitmap(env, outputBitmap, javaAllocator.getStorageObj(),bitmapCreateFlags, ninePatchChunk, layoutBounds, -1);
}
```

从上述代码，我们可以有以下几点结论
1. API19以上复用Bitmap时依然要求被复用的Bitmap必须可变的。
2. native层是通过把原始bitmap重新draw到一个放大(sx,xy)倍的Bitmap上，来实现缩放处理的。
3. 相比API18，API19引入了两个不同策略的内存分配器：RecyclingPixelAllocator和ScaleCheckingAllocator，并在创建分配器实例时，将被复用位图的像素缓冲区大小传给了分配器实例，而Bitmap复用中的**内存大小限制**就在这两个Allocator里。同时根据上述代码逻辑可知：
    >1. 若是复用Bitmap且缩放因为为1，那么使用RecyclingPixelAllocator进行内存分配。
    >2. 若是复用Bitmap且缩放因子不为1，那么使用ScaleCheckingAllocator进行内存分配。

下面我们看下这两个Allocator怎么进行bitmap复用限制的。

首先看下在`RecyclingPixelAllocator`的实现。
``` c++
class RecyclingPixelAllocator : public SkBitmap::Allocator {
public:
    RecyclingPixelAllocator(SkPixelRef* pixelRef, unsigned int size)
            : mPixelRef(pixelRef), mSize(size) {
        SkSafeRef(mPixelRef);
    }

    ~RecyclingPixelAllocator() {
        SkSafeUnref(mPixelRef);
    }
    //为指定bitmap分配像素缓冲区
    virtual bool allocPixelRef(SkBitmap* bitmap, SkColorTable* ctable) {
        //这里很关键哈，判断bitmap需要的像素缓冲区大小和被复用bitmap的像素缓冲区大小
        if (!bitmap->getSize64().is32() || bitmap->getSize() > mSize) {
            ALOGW("bitmap marked for reuse (%d bytes) can't fit new bitmap (%d bytes)",
                    mSize, bitmap->getSize());
            return false;
        }

        SkImageInfo bitmapInfo;
        if (!bitmap->asImageInfo(&bitmapInfo)) {
            ALOGW("unable to reuse a bitmap as the target has an unknown bitmap configuration");
            return false;
        }

        // Create a new pixelref with the new ctable that wraps the previous pixelref
        //到了这里，说明可以进行bitmap的复用，因此把被复用bitmap的像素缓冲区直接赋值给需要分配像素缓冲区的bitmap，这样就达到了复用bitmap的目的
        SkPixelRef* pr = new AndroidPixelRef(*static_cast<AndroidPixelRef*>(mPixelRef),
                bitmapInfo, bitmap->rowBytes(), ctable);

        bitmap->setPixelRef(pr)->unref();
        // since we're already allocated, we lockPixels right away
        // HeapAllocator/JavaPixelAllocator behaves this way too
        bitmap->lockPixels();
        return true;
    }

private:
    SkPixelRef* const mPixelRef;//被复用Bitmap的像素缓冲区
    const unsigned int mSize;//被复用Bitmap的像素缓冲区大小
};
```
从上述代码可知，API19是在为bitmap分配像素缓冲区时，判断是否可以复用已有bitmap的。若被复用bitmap的像素缓冲区大小大于等于需要为新bitmap分配的像素缓冲区大小，那么就可以复用，否则就不可以复用。

同时还有很关键的一点结论：**RecyclingPixelAllocator.allocPixelRef方法才是真正复用被复用Bitmap像素缓冲区的地方**。

然后，看下`ScaleCheckingAllocator`的实现.

``` C++
class ScaleCheckingAllocator : public SkBitmap::HeapAllocator {
public:
    ScaleCheckingAllocator(float scale, int size)
            : mScale(scale), mSize(size) {
    }
    virtual bool allocPixelRef(SkBitmap* bitmap, SkColorTable* ctable) {
        //这里根据bitmap的config/原始宽高/缩放因子计算出缩放后需要多大的像素缓冲区空间，若大于被复用Bitmap的像素缓冲区大小，则无法复用，否则可以被复用。
        const int bytesPerPixel = SkBitmap::ComputeBytesPerPixel(
                configForScaledOutput(bitmap->config()));
        //计算出缩放后Bitmap需要占用多大内存
        const int requestedSize = bytesPerPixel * (int)(bitmap->width() * mScale + 0.5f) * (int)(bitmap->height() * mScale + 0.5f);
        if (requestedSize > mSize) {
            ALOGW("bitmap for alloc reuse (%d bytes) can't fit scaled bitmap (%d bytes)",
                    mSize, requestedSize);
            return false;
        }
        //真正分配像素缓冲区是在这里（但是这里分配的像素缓冲区没有考虑缩放因子且是申请的新的内存，真正的缩放处理是在BitmapFactory.doDecode方法中，该方法中处理缩放时，又会通过RecyclingPixelAllocator复用被复用bitmap的像素缓冲区）
        return SkBitmap::HeapAllocator::allocPixelRef(bitmap, ctable);
    }
private:
    const float mScale;
    const int mSize;
};
```

`ScaleCheckingAllocator.allocPixelRef`方法根据bitmap的config，原始宽高，缩放因子等因素，计算出缩放后需要多大的像素缓冲区空间，以此来判断是否可以复用。若可以复用，那么这里会解码出一个原始宽高的bitmap（真正分配了像素缓冲区），然后在BitmapFactory.doDecode方法中，处理对原图的缩放，此时RecyclingPixelAllocator类负责为缩放后的bitmap分配像素缓冲区，也就达到了bitmap复用的目的（也就是我们上面说的***RecyclingPixelAllocator.allocPixelRef方法才是真正复用被复用Bitmap像素缓冲区的地方***）。

综合来看，**所谓Bitmap复用，主要就是复用被复用Bitmap的像素缓冲区数据，避免了内存的重复申请和释放**。（同时修改bitmap宽度，高度等信息）

最后我们再看下API19版本，png解码器的工作（精简后）：
``` c++
bool SkPNGImageDecoder::onDecode(SkStream* sk_stream, SkBitmap* decodedBitmap,Mode mode) {
png_uint_32 origWidth, origHeight;
int bitDepth, colorType, interlaceType;
//获取初始信息
png_get_IHDR(png_ptr, info_ptr, &origWidth, &origHeight, &bitDepth,&colorType, &interlaceType, int_p_NULL, int_p_NULL);
//获取bitmap的配置信息
if (!this->getBitmapConfig(png_ptr, info_ptr, &config, &hasAlpha, &theTranspColor)) {
    return false;
}
const int sampleSize = this->getSampleSize();
//根据图片的原始宽高和sampleSize创建sampler对象，里面会生成新的宽高（origWidth/sampleSize）
SkScaledBitmapSampler sampler(origWidth, origHeight, sampleSize);
//这里sampler.scaledWidth()就是除以sampleSize之后的新的原始宽高
//有了位图的高宽及颜色格式，才能计算出该位图需要多大的缓冲区来存储像素数据
decodedBitmap->setConfig(config, sampler.scaledWidth(), sampler.scaledHeight());
}
//若是Options.inJustDecodeBounds为true，那么到此就结束了，因为上面宽和高已经计算出来了
if (SkImageDecoder::kDecotonguodeBounds_Mode == mode) {
    return true;
}
//为bitmap分配像素缓冲区，这里真正分配像素缓冲分区的是我们在BitmapFactory.doDecode方法中设置的XXXAllocator。（见后面的方法解释）
if (!this->allocPixelRef(decodedBitmap,SkBitmap::kIndex8_Config == config ? colorTable : NULL)) {
    return false;
}

//后续代码应该就是真正去解码png格式图片，生成像素数据的核心逻辑了，这块暂时不进行介绍

}

//这里给出上面用到的一些关键方法：
//给解码器设置内存分配器，复用位图时将传入RecyclingPixelAllocator或ScaleCheckingAllocator（在BitmapFactory.doDecode方法中设置）
SkBitmap::Allocator* SkImageDecoder::setAllocator(SkBitmap::Allocator* alloc) {
    if (alloc) alloc->ref();
    if (fAllocator) fAllocator->unref();
    fAllocator = alloc;
    return alloc;
}

//为位图分配像素缓冲（png解码器的父类方法）
bool SkImageDecoder::allocPixelRef(SkBitmap* bitmap,SkColorTable* ctable) const {
    //调用SkBitmap的allocPixels 方法分配像素缓冲                               
    return bitmap->allocPixels(fAllocator, ctable);
}

//最终分配位图像素缓冲区的方法
bool SkBitmap::allocPixels(Allocator* allocator, SkColorTable* ctable) {
    HeapAllocator stdalloc;//默认的内存分配器
    if (NULL == allocator) {
        allocator = &stdalloc;
    }
    //最终调用到解码器的内存分配器去分配内存
    return allocator->allocPixelRef(this, ctable);
}
```
从上述代码可知：相比于API18，API19上`SkImageDecoder_libpng.onDecode`的实现稍微有点不同：主要是不再进行Bitmap复用逻辑的处理，因为在`BitmapFactory.doDecode`方法中每次解码（`SkImageDecoder_libpng.onDecode`）传进来的参数都是新创建的decodingBitmap。现在是在BitmapFactory.doDecode方法中处理Bitmap的复用（通过RecyclingPixelAllocator实现）。

这样整个逻辑就串联起来了，这里简单总结下API19解码图片的几个点：
1. 在java层不再处理图片缩放，而是统一在native层`BitmapFactory.doDecode`方法中处理对Bitmap的缩放。
2. 验证了上文说的API19对Bitmap复用的两个限制。
3. 所谓Bitmap复用，主要就是复用被复用Bitmap的像素缓冲区，避免了内存的重复申请和释放。
4. `RecyclingPixelAllocator.allocPixelRef`方法才是真正复用被复用Bitmap像素缓冲区的地方。

## 使用Bitmap的建议
网上有很多这方面的资料，这里不打算赘述。
推荐仔细阅读下官方教程：[Displaying Bitmaps Efficiently](https://developer.android.com/training/displaying-bitmaps/index.html)

同时如果想在API19以下，突破Bitmap复用的限制（达到和API19以上一样的效果），那么可以参考Github上的一个方案:[BitmapFactoryCompat](https://github.com/badpx/BitmapFactoryCompat)。

OK，任务完成，希望对有需要的人有所帮助。

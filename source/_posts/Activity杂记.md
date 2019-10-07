---
title: Activity杂记
date: 2017-06-08 13:40:45
tags:
 - Android
 - Activity
categories:
 - Android
---
这里仅记录`Activity`相关的知识点和踩坑经历。

<!-- more -->

### FLAG_ACTIVITY_NEW_TASK标志位
通过非Activity的Context调用startActivity时，需要在Intent里面加上`FLAG_ACTIVITY_NEW_TASK`标志位，不然就抛出以下异常：
``` java
Caused by: Android.util.AndroidRuntimeException:
Calling startActivity() from outside of an Activity context requires the FLAG_ACTIVITY_NEW_TASK flag. Is this really what you want?
```
### onActivityResult
ActivityA通过`startActivityForResult `启动LaunchMode为`SingleTask` or `SingleInstance`的ActivityB时。在5.0之前的系统上，`onActivityResult `方法会立即被调用，而不是正常情况下，等到ActivityB关闭时，再回调ActivityA的`onActivityResult `方法，可参考[这里](http://stackoverflow.com/questions/8960072/onactivityresult-with-launchmode-singletask)。 




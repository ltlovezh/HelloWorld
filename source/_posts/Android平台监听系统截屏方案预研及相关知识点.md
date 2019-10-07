---
title: Android平台监听系统截屏方案预研及相关知识点
date: 2016-06-12 13:21:50
tags:
 - Android
 - 系统截屏
 - FileObserver
 - ContentObserver
 - 多媒体数据库
categories:
 - Android
---
最近有个针对系统截屏的需求，所以预研了Android平台上捕获系统截屏的方案。

最直接的方式就是监听手机的系统截屏组合键（电源键+音量下键），但是这种方式实现难度大，且有的机型使用特殊手势进行截屏，兼容性问题难以解决。

所以网上流行的方案是监听系统截屏目录下文件创建事件或者多媒体数据库图片资源变更通知。我对两种方式都做了测试，多多少少都存在一些问题，现整理如下：

<!-- more -->

## 通过FileObserver监听系统截屏目录下的文件创建
`FileObserver`可以对一个文件或者目录进行监听，它是基于linux的inotify实现，可以监听文件创建、访问、修改等操作。

虽然文档上说FileObserver可以实现递归监听，即被监听文件夹下所有文件和级联子目录的改变都会触发监听器。但是，真正实验下来发现，不是这么回事！**被监听目录的子目录的本身改动以及子目录下的文件改动都不会触发监听器。因此，要想实现递归监听，必须自己递归实现对每个子目录的监听**。

FileObserver可以监听多种类型的事件：

| 事件类型 | 说明 |
| :---: | :--- |
| ACCESS | 被监听文件被访问 |
| MODIFY | 被监听文件被修改 |
| ATTRIB | 被监听文件或目录的权限、Owner等属性被改变 |
| CLOSE_WRITE | 被监听的可写文件或者目录（已经被打开）被关闭 |
| CLOSE_NOWRITE | 被监听的只读文件或者目录（已经被打开）被关闭 |
| OPEN | 被监听文件或者目录被打开 |
| MOVED_FROM | 文件或者子目录从当前被监听目录下被移走 |
| MOVED_TO | 文件或者子目录从其他目录被移动到当前被监听目录下 |
| CREATE | 在当前被监听目录下，创建文件或者子目录 |
| DELETE | 在当前被监听目录下删除一个文件 |
| DELETE_SELF | 被监听的文件或者目录本身被删除，此时监听将被停止 |
| MOVE_SELF | 被监听的文件或者目录本身被移动 |
| ALL_EVENTS | 上面多有事件的并集 |

FileObserver是抽象类，我们需要实现`onEvent`方法处理具体业务逻辑。此外，创建FileObserver对象时，需要指定被监听文件或者目录，以及需要监听的事件类型。

经过实际测试，发现使用`FileObserver`进行文件（夹）监控，有几点需要注意：

>1. 不要在`onEvent`方法中进行耗时操作，这样会导致线程被阻塞，无法监听到后续事件，最好在工作线程进行统一处理。
2. 防止出现死循环，比如：若监听`CREATE`事件时，就不能在`onEvent`方法中在被监听目录创建文件，否则又会触发`CREATE`事件，导致死循环。
3. 回调方法`onEvent`中的参数path，仅是文件名，不是完整路径。
4. 在监听到`CREATE`事件时，需要等待几百ms，才能加载到到文件。（这点很坑，不知道有啥解决方案不？！）

OK，FileObserver的基本情况介绍完了，下面我们看下使用FileObserver监听系统截图的方案和可行性：因为我们要监听系统截图，因此理论上只需要监听系统截图目录的`CREATE`事件即可。基本代码如下所示：

``` java
//三星Note3下的系统截图目录
String path = "/storage/emulated/0/Pictures/Screenshots";
//小米4下的系统截图目录
//path = "/storage/emulated/0/DCIM/Screenshots";

//指定监听路径path和事件类型CREATE
FileObserver fileObserver = new FileObserver(path,FileObserver.CREATE) {
    @Override
    public void onEvent(int event, String path) {
        //这里最好启动一个线程去加载系统截屏的图片，否则会导致线程被阻塞，无法监听到后续事件。
        //此外，这里的path仅是图片文件名，不是完整路径
        //收到CREATE事件后，立即去加载图片是获取不到的，需要延迟几百毫秒才可以加载到，估计是图片正在落地。
    }
};
//开始监听
fileObserver.startWatching();
//结束监听
fileObserver.stopWatching();
```
但是实际测试下来发现，在三星Note3上可以准确的监听系统截图，并可以获取到系统截图图片。但是在小米4上，根本监听不到`CREATE`事件（实际上，截屏图片已经在系统截屏目录了）。

在小米4上仅能监听到ACCESS(被触发多次)和OPEN事件。但是OPEN事件在三星Note3上会触发多次，而且Android手机千奇百怪，要想找到一个系统截屏时，所有手机都会触发一次的FileObserver事件，会很难，而且存在很大的兼容性问题。

因此，通过`FileObserver`监听系统截图存在两个比较大的问题：

>1. 每个手机上保存系统截屏图片的目录不完全相同，比如上面三星Note3和小米4就不同。因此，必须先获得每个手机保存截图图片的目录，才能进行监听。
2. 很难找到一个系统截屏时，所有手机仅会触发一次的FileObserver事件。

所以目前来看，通过FileObserver监听系统截图不靠谱。

## 通过ContentObserver监听多媒体数据库（图片）的资源变化

我们知道：通过系统截屏生成一张图片时，这张图片不仅会存储在系统截屏目录中，还会通过`MediaProvider`类在多媒体数据库中插入一条记录，方便系统图库进行查询。而且`MediaProvider`会将唯一标识这张图片的URI通知到感兴趣的`ContentObserver`。（关于多媒体数据库下面会进行详细介绍）

因此，我们的方案就是通过`ContentObserver`监听多媒体数据库图片资源的变化。基本代码如下所示：
``` java
//查询的表字段
static final String[] PROJECTION = new String[]{
    MediaStore.Images.Media.DATA,MediaStore.Images.Media.DATE_ADDED};
//根据时间降序排序
static final String SORT_ORDER = MediaStore.Images.Media.DATE_ADDED + " DESC";
//mHandler表示主线程的Handler，这样回调函数onChange就会在主线程被调用
ContentObserver contentObserver = new ContentObserver(mHandler) {
    @Override
    public void onChange(boolean selfChange) {
        super.onChange(selfChange);
        //从API16开始，才有两个参数的onChange方法，所以这里要主动调用下面的onChange方法。
        onChange(selfChange, null);
    }

    @Override
    public void onChange(boolean selfChange, Uri uri) {
        //若调用父类方法就死循环了
        //super.onChange(selfChange,uri);
        if (uri == null) { //API16以下版本 
            Cursor cursor = contentResolver.query(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, PROJECTION, null, null,SORT_ORDER);
            if (cursor != null && cursor.moveToFirst()) {
                //完整路径
                String path = cursor.getString(cursor.getColumnIndex(MediaStore.Images.Media.DATA));
                //添加图片的时间，单位秒
                long dateAdded = cursor.getLong(cursor.getColumnIndex(MediaStore.Images.Media.DATE_ADDED));
                long currentTime = System.currentTimeMillis() / 1000;
                //加个过滤条件必须是3S内的图片，且路径中包含截图字样“screenshot”
                if (Math.abs(currentTime - dateAdded) <= 3l && path.toLowerCase().contains("screenshot")) {
                    //这就是系统截屏的图片了，这里测试发现需要等待几百MS，才能加载到图片。因此具体实现时，最好在独立线程，每隔100MS尝试加载一次，做好超时处理。
                    Bitmap b1 = BitmapFactory.decodeFile(path);
                }
            }
        } else { //API16及以上版本
            if (uri.toString().matches(EXTERNAL_CONTENT_URI_MATCHER + "/\\d+")) {
                Cursor cursor = contentResolver.query(uri, PROJECTION, null, null, null);
                if (cursor != null && cursor.moveToFirst()){
                    //完整路径
                    String path = cursor.getString(cursor.getColumnIndex(MediaStore.Images.Media.DATA));
                    //添加图片的时间，单位秒
                    long dateAdded = cursor.getLong(cursor.getColumnIndex(MediaStore.Images.Media.DATE_ADDED));
                    long currentTime = System.currentTimeMillis() / 1000;
                    if (Math.abs(currentTime - dateAdded) <= 3l && path.toLowerCase().contains("screenshot"))  {
                        //这就是系统截屏的图片了
                        Bitmap b2 = MediaStore.Images.Media.getBitmap(contentResolver, uri);
                    }
                }
            }
        }
    }
}
//通过ContentResolver注册ContentObserver，监听"content://media/external/video/media"
getContentResolver().registerContentObserver(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, true, contentObserver);

//不需要监听的时候，一定要把原来的ContentObserver注销掉。
getContentResolver().unregisterContentObserver(contentObserver);
```
上述代码中，我们在API16以上和以下采取了两种不同的方案：

1. 方案1：API16以下，因为回调中没有URI，所以只能到多媒体数据库中去查询，然后取出最新的那一条记录，理论上就是系统截屏的图片了。
2. 方案2：API16及以上，因为回调中有唯一标识图片的URI，所以可以通过`MediaStore.Images.Media`和URI，直接获取截屏图片。这种方式既简单，又准确！

上述方案，经过测试，发现存在一些问题：

1. 方案1中，若收到onChange回调，立即去获取图片，是加载不到的，必须等几百毫秒，推测应该是图片还没完全落地。但是这个等待的时间应该跟机器性能有关，因此很难确定一个固定值（和FileObserver存在相同的问题）。
2. 不仅向多媒体数据库中插入一条图片数据会触发onChange回调，更新和删除图片数据，也会触发onChange回调。
3. 若我们主动通过`MediaProvider`向多媒体数据库插入、更新、删除一条图片数据，也会触发onChange回调。

简单来说，就是**没办法完全确定触发onChange回调的事件一定是系统截屏行为**。因此，在onChange回调方法中，判断此次回调是不是系统截屏触发的，是个难点。但是这个问题解决不好，就会造成一定的误差。比如：我通过相机拍摄了一张图片，就会触发上面的onChange回调。所以上面的代码加了两个过滤条件：必须是3S内的图片，且图片路径中包含截图字样“screenshot”。但是这样也不能确保百分之百没有误差。

---

综上所述，不管是通过`FileObserver`还是`ContentObserver`，都不能完全准确地监控系统截屏操作。（相比于IOS直接提供了API级别的支持，Android还是很蛋疼啊...）

## 多媒体数据库
Android中的多媒体数据记录（图片、音频、视频等）是存储在DB中的，即多媒体数据库。这个数据库文件存储在`/data/data/com.android.providers.media/databases`目录中。如下图所示：
<img src="http://7xs2qy.com1.z0.glb.clouddn.com/%E5%A4%9A%E5%AA%92%E4%BD%93%E6%95%B0%E6%8D%AE%E5%BA%93.png" width="500" alt="多媒体数据库"/>

其中`internal.db`是内部存储数据库文件，`external.db`是存储卡数据库文件。多媒体数据操作主要就是围绕这两个数据库来进行的，这两个数据库的结构是完全一样的。如下所示：
<img src="http://7xs2qy.com1.z0.glb.clouddn.com/%E5%A4%9A%E5%AA%92%E4%BD%93%E6%95%B0%E6%8D%AE%E5%BA%93%E8%A1%A8%E6%95%B0%E6%8D%AE.png" width="500" alt="多媒体数据库表数据"/>

上面是存储不同多媒体数据的表，其中video表主要存储视频数据；videothumbnails表主要存储视频缩略图数据；audio_xx表主要存储音频数据，音频数据比较复杂，又需要album相关表存储专辑信息，artist相关表存储歌手信息；`images`表主要存储图片数据。`thumbnails`表主要存储图片缩略图数据。

这里我们主要看下`images`表结构，如下所示：
![image表结构](http://7xs2qy.com1.z0.glb.clouddn.com/image%E8%A1%A8.png)
可见，images表是基于`files`表的视图。其中,`_data`字段表示图片的完整路径，`data_added`字段表示添加图片的时间，`width`和`height`字段分别表示图片的宽度和高度,`_display_name`字段则表示图片名称。

下面看两个具体案例，我们分别通过系统截屏手势和相机获取一张图片，然后看下这两种图片在`images`表中的存储。
首先是截屏获得的图片，其表记录如下所示：
<img src="http://7xs2qy.com1.z0.glb.clouddn.com/%E6%88%AA%E5%9B%BE%E6%95%B0%E6%8D%AE.png" width="500" alt="截图表数据"/>

然后是相机拍摄出的图片，其表记录如下所示：
<img src="http://7xs2qy.com1.z0.glb.clouddn.com/%E6%8B%8D%E7%85%A7%E6%95%B0%E6%8D%AE.png" width="500" alt="拍照数据"/>

从上述两张图片的表数据可知：

* 图片id确实是递增的。
* 系统截图和相机拍摄的图片存储在不同的目录。
* 系统截图图片是png格式，相机拍摄图片是jpeg格式。
* `bucket_display_name`字段指出了图片的来源途径，它是根据`_data`字段生成的。
* 系统截屏图片的宽高就是屏幕的宽高，而相机拍摄图片的宽高则和具体手机有关，但一般都大于屏幕宽高。
* 向其他字段的含义也很明确，此处不再赘述。

上面我们是通过sql语句直接查询图片数据，其实Android系统给我们封装了`MediaStore`类，它提供了多媒体数据存储与获取相关API，其基本结构如下所示（详细结构可参见源码）：
<img src="http://7xs2qy.com1.z0.glb.clouddn.com/MediaStore.png" width="450" alt="MediaStore"/>

其中`Images.ImageColumns`类主要封装了images表的字段。`Images.Media`类主要提供了查询和插入图片数据的API（这类API很简单，都是通过`ContentResolver`和`uri`，呼起对应的`MediaProvider`完成真正的DB操作），以及可以通过`getBitmap`方法获取图片的Bitmap对象，而`Images.Thumbnails`类则提供了操作缩略图的相关API。同样的，其他的内部类（Audio、Video）分别对应音频表和视频表。

`Images.Media.getBitmap`方法很便利，其实现也很简单，首先通过uri获取输入流（详情参见源码），然后通过BitmapFactory类解码获取Bitmap。如下所示：
``` java
public static final Bitmap getBitmap(ContentResolver cr, Uri url)throws FileNotFoundException, IOException {
    InputStream input = cr.openInputStream(url);
    Bitmap bitmap = BitmapFactory.decodeStream(input);
    input.close();
    return bitmap;
}
```

从`MediaStore`类的源码可知，它提供的API都是通过`ContentResolver`和`Uri`呼起对应`MediaProvider`来实现的，`MediaProvider`才是真正实现多媒体数据库操作的场所。关于MediaProvider，又是单独话题了，感兴趣的可以去看源码。

`MediaStore`类为每一种资源分配了单独的Uri地址，例如：视频资源的基础地址是MediaStore.Vedio.MediaEXTERNAL_CONTENT_URI,即`content://media/external/video/media`，图片资源的基础地址是MediaStore.Images.Media.EXTERNAL_CONTENT_URI，即`content://media/external/images/media`。

这些基础地址都是数据集合类型，对应的个体数据类型则是在基础地址后面加上图片ID。例如：上面我们通过系统截屏获得的图片资源ID是**233494**，那么唯一标识这张图片的uri就是`content://media/external/images/media/233494`，通过这个uri，就可以获取这张图片的所有信息了（上面getBitmap方法的第二个参数就是这种个体数据类型uri）。实际操作中，要使用哪种类型的URI，则要根据具体情况而定。

因此，获取系统截屏图片的Bitmap对象有两种方式：

>1. 假如知道了图片的唯一标识URI，那么通过`MediaStore.Images.Media.getBitmap`方法就可以获取了。
>2. 假如不知道URI，而知道图片的本地地址（SD卡地址），那么只能通过`BitmapFactory`类的decodeXXX方法来搞定了。

## ContentProvider的数据更新通知机制
上面介绍的第二种方案，依赖的就是ContentProvider的数据更新通知机制。因为ContentProvider是以URI形式来组织资源的，所以当数据变更时，也是以URI形式通知感兴趣的ContentObserver。

整个数据更新机制的示意图如下所示：
<img src="http://7xs2qy.com1.z0.glb.clouddn.com/ContentProvider%E6%95%B0%E6%8D%AE%E5%8F%98%E6%9B%B4%E9%80%9A%E7%9F%A5%E6%9C%BA%E5%88%B6%E7%A4%BA%E6%84%8F%E5%9B%BE.png" width="600" alt="ContentProvider的数据更新通知机制" />

其中，`ContentService`服务就是管理所有`ContentObserver`监听器的场所，它运行在System进程，以多叉树的形式组织所有监听器。而`MediaProvider`则负责操作多媒体数据库，并以URI的形式发出数据变更通知到`ContentService`服务，`ContentService`负责从树形数据结构中找出对该URI感兴趣的`ContentObserver`，然后跨进程回调`ContentObserver.onChange`方法。

所以这里的关键点就是`ContentService`服务中多叉树数据结构的建立和查询。其中多叉树的节点是`ObserverNode`，如下所示：

``` java
class ObserverNode{
    String mName;//节点名称
    ArrayList<ObserverNode> mChildren = new ArrayList<ObserverNode>();//孩子节点
    ArrayList<ObserverEntry> mObservers = new ArrayList<ObserverEntry>();//该节点上的监听器
}

class ObserverEntry{
    //跨进程回调的接口
    IContentObserver observer;
    //该参数就是注册监听器时的第二个参数，若为false，则表示若变化的URI是正在监听的URI的父节点或者相同节点时，就会触发回调。若为true，则在上述时机之上，若变化的URI是正在监听的URI的子节点时，也会触发回调。
    boolean notifyForDescendants;
}
```

上面我们监听系统截屏事件时，监听的URI是`content://media/external/images/media`，且notifyForDescendents参数为true。因此，注册之后，`ContentService`服务的多叉树数据结构如下所示：
<img src="http://7xs2qy.com1.z0.glb.clouddn.com/%E5%A4%9A%E5%8F%89%E6%A0%91%E7%BB%93%E6%9E%84.png" width="600" alt="ContentService的多叉树结构"/>

而当系统截屏图片插入到多媒体数据库时，`MediaProvider`会发出`content://media/external/images/media/xxx`形式的通知，该通知到达`ContentService`服务后，就会在上面的多叉树数据结构中进行检索，以找到对此URI感兴趣的监听器。

其中当查找到`media`节点时，就会把`media`节点中的notifyForDescendants属性为true（即正在通知的URI是content://media/external/images/media的子节点）的ObserverEntry对象收集起来。最后，通过ObserverEntry对象的observer接口属性回调到应用程序进程的`ContentObserver.onChange`方法，这样整个流程就完整了。

这里在应用程序进程注册URI时，需要特别注意，`ContentService`服务在组织多叉树数据结构时，遇到`/`、`#`、`?`这三个特殊符号，就会停止构造子节点，因此`content://media/external/images/media/#`、`content://media/external/images/media//#`和`content://media/external/images/media/#/?`等URI形成的多叉树结构都是相同的，即上面的树形结构。（一开始我在注册URI时，以为`#`号的作用和ContentProvider中#号一样，代表所有的整型ID，坑了我很久）。

## 参考文档
1. [深入理解MediaScanner](http://wiki.jikexueyuan.com/project/deep-android-v1/mediascanner.html)
2. [Android应用程序组件Content Provider的共享数据更新通知机制分析](http://blog.csdn.net/luoshengyang/article/details/6985171)
3. [Detect only screenshot with FileObserver Android](http://stackoverflow.com/questions/29621903/detect-only-screenshot-with-fileobserver-android/29624090#29624090)

---
title: Android Menu
date: 2016-06-22 21:02:07
tags:
 - Android
 - Menu
 - PopupWindow
 - OptionsMenu
 - ContextMenu
 - SubMenu
categories:
 - Android
---
之前总结的一些知识点，仅当备忘！

## 基础知识篇
Android系统里面有3种类型的菜单：选项菜单（OptionsMenu）、上下文菜单（ContextMenu）和子菜单（SubMenu）。还有一个菜单项MenuItem.

关于Menu相关的XML标签，可以参见官网解释Menu Resource，很详细。

<!-- more -->

### OptionsMenu
默认样式是在屏幕底部弹出一个菜单，这个菜单我们就称为选项菜单。 
选项菜单和当前Activity绑定，支持设置快捷键。

唤起选项菜单有两种方式：

1. 按Menu键。
2. 调用Activity.openOptionsMenu

这两种方式最终都会调用到PhoneWindow.openPanel方法，该方法是真正打开OptionsMenu的地方。 
因为选项菜单是和Activity绑定的，所以我们只需要重写以下Activity相应回调，就可以处理选项菜单了。

1. Activity.onCreateOptionsMenu (Menu menu)
> 创建options menu，这个函数只会在选项菜单第一次显示时调用。通过该方式就可以添加XML描述的菜单：getMenuInflater().inflate(R.menu.menu_main, menu); 选项菜单中的menu为MenuBuilder类对象。

2. Activity.onPrepareOptionsMenu (Menu menu)
> 选项菜单每次显示之前，onPrepareOptionsMenu方法都会被调用，我们可以用此方法来根据当时的情况调整选项菜单。

3. Activity.onOptionsItemSelected (MenuItem item)
> 处理选项菜单中菜单项的点击事件，一般根据菜单项的ID来区分具体的菜单项（tem.getItemId）

4. Activity.onOptionsMenuClosed(Menu menu)
> 当选项菜单被关闭时，不管是用户主动取消（点击back or menu键），还是菜单项被选中，都会被调用。


### ContextMenu
上下文菜单是和指定View相绑定的，不支持设置快捷键。唤起上下文菜单有两种方式，两种方式都有一个前提条件，为View注册上下文菜单Activity.registerForContextMenu(View)

1. 长按某个View，并且没有处理该View上的LongClick事件。
2. 调用Activity.openOptionsMenu方法。

这两种方式最终都会调用到View.showContextMenu方法。 
此外，我们也只需要重写以下Activity相应回调，就可以处理上下文菜单了。

1. Activity.registerForContextMenu(View view)
> 某个View注册ContextMenu，一般在Activity.onCreate里面调用。

2. Activity.onCreateContextMenu(ContextMenu menu, View v, ContextMenu.ContextMenuInfo menuInfo)
> 创建context menu，和options menu不同，context meun每次显示时都会调用这个方法。通过该方式就可以添加XML描述的菜单：getMenuInflater().inflate(R.menu.menu_main, menu); menuInfo是额外信息，可以通过重写View.getContextMenuInfo方法来提供。参数menu为ContextMenuBuilder类对象。

3. Activity.onContextItemSelected(MenuItem item)
> 处理上下文菜单中菜单项的点击事件，一般根据菜单项的ID来区分具体的菜单项（tem.getItemId）

### SubMenu
SubMenu继承自Menu，以上两种Menu都可以加入子菜单，但子菜单不能嵌套子菜单（但是，我试了一下，貌似可以嵌套），这意味着在Android系统，菜单只有两层，设计时需要特别注意！同时子菜单的菜单项不支持icon。

Menu接口是所有菜单的基类，主要提供了add方法来添加菜单项：
``` java
/**
groupId,表示组ID，如果不分组的话就写Menu.NONE。
itemI,菜单项Id，这个很重要，Android根据这个Id来确定不同菜单项的点击事件，不需要的话就写Menu.NONE。
order,菜单项顺序，值越小，越在前面。
title,菜单项标题。
*/
public MenuItem add(int groupId, int itemId, int order, CharSequence title);
```

此外，还提供了addSubMenu来添加子菜单:
``` java
/**
参数的含义和add方法类似，这里的返回值为SubMenuBuilder类对象
*/
SubMenu addSubMenu(final int groupId, final int itemId, int order, final CharSequence title);
```

---

此外，ContextMenu和SubMenu接口，还额外增加了一些setHeaderTitle、setHeaderIcon、setHeaderView等方法，用于设置上下文菜单和子菜单的头部显示区域。

另外，对于SubMenu，它在上一级菜单中的显示也是一个MenuItem，因此可以通过SubMenu.getItem获取子菜单在上一级菜单中的显示项，从而可以对其设置一些属性，以修改子菜单在上级菜单中的展示。

**因为选项菜单默认样式是在屏幕底部弹出一个菜单，所以没有Head区域，而ContextMenu和SubMenu接口都是独立的菜单，所有包含Head区域.**

MenuItem则是菜单项的基类，提供了各种setXX方法来设置各种属性，这些属性和XMl标签中的item标签中的属性是对应的。 
若该菜单项代表了一个子菜单，那么MenuItem.mSubMenu则代表了点击该菜单项后，需要launch的子菜单。

关于Menu的Xml形式，可以参考官网解释[Menu Resource](http://developer.android.com/intl/zh-cn/guide/topics/resources/menu-resource.html)，很详细。

## 原理篇
### OptionsMenu的处理流程
由上文可知，弹出选项菜单的途径有两种，这两种方式最终都会调用到PhoneWindow.openPanel方法，该方法是真正打开OptionsMenu的地方。 
这里来看下点击“menu”触发OptionsMenu的流程。 
用户点击“menu”按键后，会相继触发KeyUp和KeyDown事件。 
触发KeyUp事件的流程：

>PhoneWindow.DecorView.dispatchKeyEvent -> PhoneWindow.onKeyDown -> PhoneWindow.onKeyDownPanel

触发KeyDown事件的流程为：
>PhoneWindow.DecorView.dispatchKeyEvent -> PhoneWindow.onKeyUp -> PhoneWindow.onKeyUpPanel

onKeyDownPanel的关键代码如下所示：
``` java
PanelFeatureState st = getPanelState(featureId, true);
if (!st.isOpen) {
    return preparePanel(st, event);
}
```
onKeyUpPanel的代码较复杂，但是最终会调用到`openPanel`方法，该方法才是真正打开OptionsMenu的地方。此外，**直接调用Activity.openOptionsMenu也会调用到该方法**。

openPanel方法首先会调用preparePanel方法来准备`PhoneWindow.createdPanelView`和`PhoneWindow.menu`，然后构建DecorView，选择合适的子View，最终，通过WindowManager来添加Window。

PanelFeatureState是一个PhoneWindow中一个非常重要的子类，该类包含了PhoneWindow中所有的子窗口，选项菜单窗口只是一种特色（Feature）而已，所有的窗口都保存在PanelFeatureState的mPanels[]中。PanelFeatureState几个重要的属性如下所示:
``` java
/** Dynamic state of the panel. 特色窗口的根视图PhoneWindow.DecorView */
DecorView decorView;

/** The panel that was returned by onCreatePanelView(). 直接创建的特色窗口的内容，可以重写Activity.onCreatePanelView来提供，貌似和menu是互斥的 */
View createdPanelView;

/** The panel that we are actually showing. 最后，实际上展示的特色窗口的内容（从createdPanelView和menu中二选一，createdPanelView优先），会当作decorView的子View*/
View shownPanelView;

/** Use {@link #setMenu} to set this. 选项菜单中的菜单实体，貌似和createdPanelView是互斥的*/
MenuBuilder menu;
```

因此，我们可以通过以下几个途径来提供选项菜单：

1. 重写Activity.onCreatePanelView方法，这里自定义的View是优先级最高的，若这里提供了选项菜单View，那么后续的onCreateOptionsMenu和onPrepareOptionsMenu都不会执行。自定义的View会赋值给PanelFeatureState.createdPanelView。
2. 重写onCreateOptionsMenu和onPrepareOptionsMenu方法，来为Menu添加具体的菜单内容。这两个方法的参数都是PanelFeatureState.menu.

最终在`PhoneWindow.initializePanelContent`方法内，从createdPanelView和menu属性中选择一个赋值给`PhoneWindow.shownPanelView`，然后把shownPanelView添加到`PhoneWindow.decorView`中，该decorView即为Window的根布局。最终调用`WindowManager.addView(st.decorView, lp)`方法来添加该Window，其中该窗口的类型为WindowManager.LayoutParams.TYPE_APPLICATION_ATTACHED_DIALOG。此时，lp.token为null，最终在添加View的过程中，lp.token会被赋值为Activity对应的窗口的ViewRootImpl.W对象。 
可见，添加选项菜单，并没有创建相应的Window对象，仅仅是添加了一个DecorView。

### ContextMenu的处理流程
由上文可知，弹出上下文菜单的途径有两种，这两种方式最终都会调用到View.showContextMenu方法。这里看下长按方式唤起上下文菜单的流程。 
>View.performLongClick -> View.showContextMenu -> PhoneWindow.DecorView.showContextMenuForChild 。

showContextMenuForChild方法是唤起上下文菜单的调度中心，其核心代码如下所示：
``` java
public boolean showContextMenuForChild(View originalView) {
    // Reuse the context menu builder
    if (mContextMenu == null) {
       mContextMenu = new ContextMenuBuilder(getContext());
       mContextMenu.setCallback(mContextMenuCallback);
       } else {
       mContextMenu.clearAll();
     }

    final MenuDialogHelper helper =   mContextMenu.show(originalView,originalView.getWindowToken());
    if (helper != null) {
       helper.setPresenterCallback(mContextMenuCallback);
      }
     ContextMenuHelper = helper;
     return helper != null;
}
```
面的方法中有两个关键的类，其中，ContextMenuBuilder主要是管理菜单中的数据，并提供操作这些数据的方法，该类中包含`ArrayList<MenuItemImpl>`属性，用于保存菜单中的所有菜单项；而`MenuDialogHelper`则负责把菜单数据以窗口形式显示到屏幕上，即完成真正的添加窗口的工作。 

下面来看下ContextMenuBuilder.show方法的实现：
``` java
//originalView表示要显示上下文菜单的View，token表示该View的顶层窗口的ViewRootImpl.W
public MenuDialogHelper show(View originalView, IBinder token) {
     if (originalView != null) {
         //通过一系列回调方法来填充ContextMenu
         originalView.createContextMenu(this);
     }
     if (getVisibleItems().size() > 0) {
            EventLog.writeEvent(50001, 1);
            MenuDialogHelper helper = new MenuDialogHelper(this); 
            helper.show(token);//真正的添加窗口
            return helper;
     }
    return null;  
}
```
originalView.createContextMenu方法会调用一系列回调方法来填充ContextMenuBuilder类，以供下面添加窗口时使用，关键代码如下所示：
``` java
public void createContextMenu(ContextMenu menu) {
    //我们可以重写getContextMenuInfo方法，已提供菜单相关信息
    ContextMenuInfo menuInfo = getContextMenuInfo();
    ((MenuBuilder)menu).setCurrentMenuInfo(menuInfo);
    //第一次机会，可以重写onCreateContextMenu方法来填充menu
    onCreateContextMenu(menu);
    ListenerInfo li = mListenerInfo;
    //第二次机会，设置View的OnCreateContextMenuListener回调方法来填充Menu，我们调用Activity.registerForContextMenu就是为指定View设置该回调。
    if (li != null && li.mOnCreateContextMenuListener != null) {
       li.mOnCreateContextMenuListener.onCreateContextMenu(menu, this, menuInfo);
      }
    ((MenuBuilder)menu).setCurrentMenuInfo(null);
    //第三次机会，调用父视图的同名方法，这就意味着父视图中添加的菜单条目，将会出现在每一个子视图中。
    if (mParent != null) {
        mParent.createContextMenu(menu);
     }
}
```

上面的方法中，首先提供了getContextMenuInfo方法，让我们来提供menuInfo，作为填充Menu的相关数据。然后，提供了3次机会让我们来填充ContextMenu，详细信息可参考上面的注释。

通过`originalView.createContextMenu`方法来填充上下文菜单后，就获取了显示上下文菜单需要的数据，那么下一步就是添加窗口了。

主要代码在`MenuDialogHelper.show`方法中，该方法主要是创建`AlertDialog`，然后把前面获取的Menu的数据填充到Dialog中，然后以Dialog的方式来添加Menu(Shows menu as a dialog)。这里就不详细分析了，基本就是和使用AlertDialog的方式一致。

### OptionsMenu和ContextMenu的对比

1. 上下文菜单实际上是一个AlertDialog，即会创建一个PhoneWindow。而选项菜单仅仅是添加了一个DecorView，没有相应的Window，因此需要自己来处理所有的事件。
2. 一个窗口中多次弹出的选项菜单属于同一个MenuBuilder对象，而同一个View多次弹出的上下文菜单却属于多个ContextMenuBuilder对象。

## 实现在一个View附近展示的Window（Menu、View）的方式
1. 使用上下文菜单（实际是一个AlertDialog）来展示。
2. 使用PopupWindow来展示（该类添加的也是一个单纯的View，没有创建Window类），具体可参考[PopupWindow](http://www.cnblogs.com/mengdd/p/3569127.html)。

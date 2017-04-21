---
layout: post
title: "Android Widget详解（一）"
date: 2015-6-27
category: Android
tag: Widget
---

#### 前言

实习需要最近在研究Android的Widget，看了很多帖子个人觉得比较零散，特地在此总结一下，写的不好大家见谅哈^_^
本博客全套源码稍后会提供下载

#### 什么是Widget

widget是安卓较苹果特有的桌面小控件，Widget可以让我们在主屏幕上面放置一些有趣的面板，比如天气插件，时间插件，Wifi开关等实用的小控件。

#### 创建一个Widget
首先Widget是BroadcastReceiver的实现，由于Widget和App是相分离的，所以Widget是运行再主屏幕进程上的，所以和传统的Activity的一些设置有很大不同，有很多限制。
创建一个Widget需要最基本的四个步骤（实现复杂的布局还需要额外步骤，稍后讲解）

1. 在res/xml目录下新建一个Widget的配置文件（没有自行创建）
2. 写一个Widget的XML布局文件
3. 新建一个MyWidget类继承AppWidgetProvider
4. 在AndroidMainifest文件中定义一个receive（前文说过他是BroadCastReceive的实现）

经过基础的三部就可以好好地创建一个Widget了
接下来对三个步骤进行详细的讲解

#### 创建一个Widget配置文件

```xml

<?xml version="1.0" encoding="utf-8"?>
<appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
    android:initialLayout="@layout/my_widget"
    android:minHeight="146dp"
    android:minWidth="292dp"
    android:previewImage="@drawable/cloud"
    android:resizeMode="horizontal|vertical"
    android:updatePeriodMillis="0"
    android:configure="com.longlong.myblogwidget.ConfigureActivity"
    android:widgetCategory="home_screen|keyguard">
</appwidget-provider>

```
在res/xml目录下写上这么一个Widget的配置文件（一个Widget对应一个配置文件）
这个是一个基本常用的配置文件，其中包含了大部分需要设置的属性，其余属性大家自行文档查询，不太常用就不讲解了。
其中对每样属性分别进行解释

- **initialLayout**:创建Widget时候对应的widget布局文件（稍后讲解怎么创建）
- **minHeight**：控件最小高度，也就是高度占用几个格子（计算公式：结果 = 70dp * 格子数 - 30dp 例如 两个格子的结果为 70 * 2 - 30 = 110）
- **minWidth**：控件最小宽度，也就是宽度占用几个格子（计算公式同上）
- **previewImage**:控件在选择的时候有个预览图，这个预览图的资源id
- **resizeMode**:Widget的大小是可以调整的，所以通过使用horizontal和vertical来设置水平和垂直方向可以调整大小，设置成none表示不能调整大小
- **updatePeriodMillis**：控件的更新频率，一般设置成为24小时，这里是毫秒,当你设置成小于30分钟时，系统会为你自动设置成30分钟，理论上越大越好，我们更新widget并不通过这个更新，如果设备正在休眠，那么更新时候设备会被唤醒
如果你需要比较频繁的更新，或者你不希望在设备休眠的时候执行更新，那么可以将updatePeriodMillis 设为 0，然后在onReeiver方法中实时更新widget。

- **widgetCategory**:设置widget可以出现的位置，在这里我们设置可以在主屏幕和键盘锁的位置出现

#### 写一个XML布局
配置文件创建完成了，我们再给widget创建一个布局文件，这样就可以使用一个Widget了
我们先使用一个简单的布局，里面只放一个TextView，稍后的教程中会写使用复杂布局。


```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="这是一个Widget"
        android:textSize="30sp" />
</LinearLayout>
```

#### 新建一个MyWidget类

```java
public class MyWidget extends AppWidgetProvider {


    /*
     * 到达指定的时间，或者用户第一次创建Appwidget所调用的方法
     */
    @Override
    public void onUpdate(Context context, AppWidgetManager appWidgetManager, int[] appWidgetIds) {
        super.onUpdate(context, appWidgetManager, appWidgetIds);
    }

    /*
     * 删除一个AppWidget所调用的方法
     */
    @Override
    public void onDeleted(Context context, int[] appWidgetIds) {
        super.onDeleted(context, appWidgetIds);
    }

    /*
     * 接收广播事件
     */
    @Override
    public void onReceive(Context context, Intent intent) {
        super.onReceive(context, intent);
    }

    /*
     * 创建第一个AppWiget实例所调用的方法
     */
    @Override
    public void onEnabled(Context context) {
        super.onEnabled(context);
    }

    /*
     * 删除最后一个Appwidget所调用的方法
     */
    @Override
    public void onDisabled(Context context) {
        super.onDisabled(context);
    }

}
```

代码方法讲解：

- onDeleted()：当该类型的AppWidget每次被删除时，调用此方法
- onDisabled()： 当该类型的窗口小部件(AppWidget)全被删除时，调用此方法
- onEnabled()： 当第一次创建该类型的AppWidget时，调用此方法
- onReceive()： 广播接受者方法，用来接受广播消息
- onUpdate()： 每次创建该类型的AppWidget都会调用此方法 ， 通常来说我们需要在该方法里为该AppWidget指定RemoteViews对象。

onUpdate中的参数context为上下文不解释，第二个为当前app的widgetmanager，最后一个参数是用当前provider设置的所有widget的id的数组，也就是说同类的widget可以访问生成的其他的widget（这个widget必须也是用这个provider生成的）

#### Widget中的那些关系

**AppWidgetManager**:管理整个app中的所有widget的工具（每实例化一个widget都算一个widget），每个widget通过自己的widgetId来区别
**AppWidgetProvider**:每一个widget都有一个属于自己的AppWidgetProvider,但是在一个AppWidgetProvider中可以响应其他的相同种类的widget，比如说可以用appWidgetManager.update()第一个参数传字符串就可以同步更新其他的所有的同类Widget,并且再onEnabled中可以响应第一次创建的时候，这就是期间有联系。
**AppWidgetProviderInfo**:所有一类的widget复用一个AppWidgetProviderInfo
**widgetId**:每一个你生成的widget都有自己的id号码，比如说你再屏幕上放两个一样的控件，但是他们的id是不同的

下面上结构图

![图片](/img/2015-6-27/jiegou.png)

#### 在AndroidMainifest文件中注册

```xml
    <receiver android:name=".MyWidget">
            <intent-filter>
                <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
            </intent-filter>
            <meta-data
                android:name="android.appwidget.provider"
                android:resource="@xml/my_widget" />
    </receiver>
```

#### 效果图

当我手指拖动的时候显示在previewImage中设置的图片

![图片](/img/2015-6-27/xiaoguo1.png)

放上之后的效果，期间会有一个配置Activity

![图片](/img/2015-6-27/xiaoguo2.png)

#### 下一章

下一章实现复杂布局和对配置Activity进行详细的讲解

[代码点击下载](https://github.com/smallSohoSolo/smallSohoSolo.github.io/blob/master/code/MyBlogWidget_One.zip?raw=true)


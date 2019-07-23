---
layout: post
title: "Android Widget详解（二）"
date: 2015-6-30
category: Android
tag: Widget
---

#### 前言

上一篇博客我们实现了自定义一个Widget的一些操作,但是没有再Widget中实现任何的点击操作和复杂布局,例如Listview的使用等等.本篇博客就来进行实现一些复杂布局

#### 点击事件的处理

Widget是运行在桌面线程中的,所以我们不能用传统的方式来对Widget进行点击操作,需要用到pendingintent来传递点击事件广播,然后在广播中处理点击事件,对其做出响应.并且我们如果操控Widget中的布局,只能操控Remoteview来达到目的,Remoteview中规定了一些我们可以使用的方法,不提供的方法我们无法实现,只能通过存在的方法变相实现目的.


首先拿到RemoteView

```java
RemoteViews view = new RemoteViews(context.getPackageName(), R.layout.widget_cw);

```


实现点击之后打开系统时钟和日历(简单,发送Intent即可)

```java
//设置打开系统日历
view.setOnClickPendingIntent(R.id.calendarlinear, PendingIntent.getActivity(context, 2, new Intent().setComponent(new ComponentName("com.android.calendar", "com.android.calendar.LaunchActivity")), PendingIntent.FLAG_UPDATE_CURRENT));
//设置打开系统时钟
view.setOnClickPendingIntent(R.id.clock_layout, PendingIntent.getActivity(context, 2, new Intent().setComponent(new ComponentName("com.android.deskclock", "com.android.deskclock.DeskClock")), PendingIntent.FLAG_UPDATE_CURRENT));

```


第一个参数是需要设置的控件ID，第二个参数是一个PendingIntent，在Widget中设置监听无法直接用Intent传递，需要用PendingIntent来进行传递。

其格式大概为 set + (控件名) + 控件属性
格式不绝对，仅供参考，所以view中没有提供的方法，就无法进行修改。

实现上述操作十分的简单,我们只需要给view中的哪个"按钮"(这个按钮可以是任何控件)来绑定pendingintent就可以了.

下面实现一个点击在Widget中做出响应

```java
//给两个按钮设置点击监听
view.setOnClickPendingIntent(R.id.btn_next, getClickIntent(context, appWidgetIds[0], R.id.btn_next, 0, "com.longlong.Widget.Button.Update"));
view.setOnClickPendingIntent(R.id.btn_previous, getClickIntent(context, appWidgetIds[0], R.id.btn_previous, 1, "com.longlong.Widget.Button.Update"));
```

#### PendingIntent

使用这样一个方法来获取我们需要的一个基本的PendingIntent（这个方法我自己实现的，非官方）

```java
//给按钮返回PendingIntent
private PendingIntent getClickIntent(Context context, int widgetId, int viewId, int requestCode, String action) {
        //拿到管理对象
        AppWidgetManager appWidgetManager = AppWidgetManager.getInstance(context);
        //pendingintent中需要的intent，绑定这个类和当前context
        Intent i = new Intent(context, MyWidget.class);
        //设置action
        i.setAction(action); //设置更新动作
        //设置bundle
        Bundle bundle = new Bundle();
        //将widgetId放进bundle
        bundle.putInt(AppWidgetManager.EXTRA_APPWIDGET_ID, widgetId);
        //放进需要设置的viewId
        bundle.putInt("Button", viewId);
        i.putExtras(bundle);
        PendingIntent pendingIntent = PendingIntent.getBroadcast(context, requestCode, i, PendingIntent.FLAG_UPDATE_CURRENT);
        return pendingIntent;
    }

```

使用这样一个方法来获取我们需要的一个基本的PendingIntent（这个方法我自己实现的，非官方）

#### 设置权限

这段代码很好理解,前文提到过,这里的点击是通过广播机制实现的,所以我们仍然需要再我们的Widget中设置权限,修改Mainifests

```xml
        <receiver android:name=".MyWidget">
            <intent-filter>
                <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
                <action android:name="com.longlong.Widget.Button.Update" />
                <action android:name="com.longlong.COLLECTION_VIEW_ACTION" />
            </intent-filter>
            <meta-data
                android:name="android.appwidget.provider"
                android:resource="@xml/my_widget" />
        </receiver>

```

其中包括两个实现自定义按钮的点击权限还有下午你要实现的listview更新的点击权限(原理相同)
然后我们只需要重写我们的上下翻页实现,也就是重写onReceive方法

```java

//这里用一个switch实现了
switch (choice) {
            case "com.longlong.Widget.Button.Update":
				//因为遥操作view所以拿到remotevie用来操作
                RemoteViews view = new RemoteViews(context.getPackageName(), R.layout.widget_cw);
				//bundle中会由系统存好下面两个属性,下面是提取出来
                Bundle bundle = intent.getExtras();
                int widgetId = -1;
                int viewId = -1;
                try {
                    widgetId = bundle.getInt(AppWidgetManager.EXTRA_APPWIDGET_ID);
                    Log.d("widgetId", String.valueOf(widgetId));
                    viewId = bundle.getInt("Button");
                    Log.d("viewId", String.valueOf(viewId));
                } catch (NullPointerException e) {
                    return;
                }
				//根据获得到的viewid进行相应view的操作(这里我是用了一个flipper)
                switch (viewId) {
                    case R.id.btn_next:
                        view.showNext(R.id.flipper);
                        break;
                    case R.id.btn_previous:
                        view.showPrevious(R.id.flipper);
                        break;
                    default:
                        return;
                }
                AppWidgetManager appWidgetManager = AppWidgetManager.getInstance(context);
                appWidgetManager.updateAppWidget(widgetId, view);
                break;

```

注释标注的很详细,我就不再多写了

#### 在Widget中实现复杂布局

Widget这种神奇的东西要想实现ListView还是需要一番周折的.下面说明以下可以在Widget中实现的一些布局

官方有话这样说：

A RemoteViews object (and, consequently, an App Widget) can support the following layout classes:

- FrameLayout
- LinearLayout
- RelativeLayout

And the following widget classes:

- AnalogClock
- Button
- Chronometer
- ImageButton
- ImageView
- ProgressBar
- TextView
- ViewFlipper
- ListView
- GridView
- StackView
- AdapterViewFlipper

Descendants of these classes are not supported.不支持这些类的后代

也就是说，只有这些控件能在布局中使用，其他的控件无法再布局中使用，包括这些类的子类，也就是说Widget中的布局是有局限的，不能像再Activity那么为所欲为0.0 而且由于widget运行在桌面进程中，而不是程序主进程，所以对UI的一切操作需要用到RemoteView类来进行操作。

#### RemoteViewsService

这个是Service的子类，多用于对“GridView，ListView”等控件进行管理。

这么进行设置

- 设置一个pendingintent模板

```java
view.setPendingIntentTemplate(R.id.data_list_view, getClickIntent(context, appWidgetIds[0], R.id.data_list_view, 3, "com.longlong.COLLECTION_VIEW_ACTION"));
```

根据后面这个action来在onReceiver对其事件做出响应

- 设置Adapter

```java
view.setRemoteAdapter(R.id.data_list_view, new Intent(context, MyDataListService.class));
```

在RemoteViewsService中可以实现控制

```java
//首先我们需要继承RemoteviewService
public class MyDataListService extends RemoteViewsService {
    @Override
    public RemoteViewsFactory onGetViewFactory(Intent intent) {
		//返回下文实现的类似Adapter的东东
        return new ViewRemoteService(this, intent);
    }

	//实现一个ViewRemoteService再其中进行adapter的一些操作(官方规定= =)
    private class ViewRemoteService implements RemoteViewsService.RemoteViewsFactory {

        private Context mContext;
        private Intent mIntent;
        private ArrayList<String> data = new ArrayList<String>();

        public ViewRemoteService(Context context, Intent intent) {
            Log.d("构造函数", "执行");
            mContext = context;
            mIntent = intent;
        }

        @Override
        public void onCreate() {
            Log.d("onCreate", "执行");
            for (int i = 0; i < 5; i++) {
                data.add("这是新闻" + i);
            }
        }

        @Override
        public void onDataSetChanged() {

        }

        @Override
        public void onDestroy() {
            data.clear();
        }

        @Override
        public int getCount() {
            return data.size();
        }

        @Override
        public RemoteViews getViewAt(int position) {
		//这里就类似getView函数了,但是实现点击同样是需要remoteview的
            RemoteViews views = new RemoteViews(mContext.getPackageName(), R.layout.item_layout);
            views.setTextViewText(R.id.text, data.get(position));
			//设置点击监听,反应事件在Widget中的onReceived中实现
            views.setOnClickFillInIntent(R.id.item_layout, new Intent().putExtra("POSITION", position));
            return views;
        }

        @Override
        public RemoteViews getLoadingView() {
            return null;
        }

        @Override
        public int getViewTypeCount() {
            return 1;
        }

        @Override
        public long getItemId(int position) {
            return position;
        }

        @Override
        public boolean hasStableIds() {
            return true;
        }
    }
}

```

我们回到Widget类的onReceived函数

```java

case "com.longlong.COLLECTION_VIEW_ACTION":
     Toast.makeText(context, String.valueOf(intent.getIntExtra("POSITION", -1)), Toast.LENGTH_SHORT).show();
```

这里实现的是点击之后弹出Toast,响应点击的是哪个
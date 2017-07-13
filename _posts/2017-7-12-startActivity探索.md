---
layout: post
title: "startActivity探索"
date: 2017-7-12
category: Android
tag: Activity
---

本篇博客代码基于aosp发表日期最新的repo master分支（Android 8.0 API 26）。可能会伴随很多源码，所以略显枯燥，源码已经被我手动精简带上注释。

### 如何启动Activity?

```java
startActivity(new Intent(this,XXX.class));
```

使用startActivity可以很轻易的启动一个Activity，可以在Activity中使用，可以在Service中使用，也就是只要是Context的子类都可以使用。那么从Context说起。首先看下Context的继承关系。Mac快捷键Control+H可以看见，这里用一个图来说明

![class_supe](/img/2017-7-12/class_super.png)

这是Context的继承关系

- Context：抽象类，没有具体实现
- ContextImpl：Context Api的真正实现，其他Context调用方法基本上都依赖于ContextImpl的实现（此处要注意并不是所有Context的子类的方法都是ContextImpl实现的，举个例子，Activity的startActivity方法就是Activity自己实现的）
- ContextWrapper：Context的包装类，包装了其子类的公共属性，传入了一个Context类型的base，调用base的方法进行使用。其实base就是传入了一个ContextImpl
- ContextThemeWrapper：包含了主题的ContextWrapper

因为Service和Application没有主题，所以继承自ContextWrapper，Activity存在主题所以继承自ContextThemeWrapper。

那么回归到最后的问题，从不同的地方startActivity到底执行了什么操作，继续看源码。最后找到的startActivity的不同实现只在两个地方，第一个地方是Activity自己实现的startActivity，第二个地方是ContextImpl实现的startActivity,Service和Application都依赖的ContextImpl的实现。

#### Activity中的实现

```java
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
   if (options != null) {
       startActivityForResult(intent, -1, options);
   } else {
       startActivityForResult(intent, -1);
   }
}
```

其实是调用了startActivityForResult来做，下面看下真实调用

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode, @Nullable Bundle options) {
   if (mParent == null) {
       options = transferSpringboardActivityOptions(options);
       Instrumentation.ActivityResult ar =
           mInstrumentation.execStartActivity(
               this, mMainThread.getApplicationThread(), mToken, this,
               intent, requestCode, options);
       if (ar != null) {
           mMainThread.sendActivityResult(
               mToken, mEmbeddedID, requestCode, ar.getResultCode(),
               ar.getResultData());
       }
       if (requestCode >= 0) {
           mStartedActivity = true;
       }
       cancelInputsAndStartExitTransition(options);
   } else {
       //此处省略调用parent的child方法
   }
}
```

parent是什么？Android3.0之前使用ActivityGroup来解决嵌套的问题，3.0之后推出了Fragment，就将ActivityGroup废弃了，这里是对其的一个兼容。

此处就是一个完整的流程，使用Instrumentation去执行execStartActivity，函数内部做的事情就是如果你的requestCode大于等于0，那么就会回调result，也就是通常请求的时候在onActivityResult中的返回值，否则返回null，如果是null那么就不会进行回调响应。

#### ContextImpl中的实现

```java
//第一步，跳到第二步
@Override
public void startActivity(Intent intent) {
   warnIfCallingFromSystemProcess();
   startActivity(intent, null);
}

//warn函数
private void warnIfCallingFromSystemProcess() {
    if (Process.myUid() == Process.SYSTEM_UID) {
        Slog.w(TAG, "Calling a method in the system process without a qualified user: " + Debug.getCallers(5));
    }
}

//第二步，真正实现
@Override
public void startActivity(Intent intent, Bundle options) {
   warnIfCallingFromSystemProcess();
   if ((intent.getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0
           && options != null 
           && ActivityOptions.fromBundle(options).getLaunchTaskId() == -1) {
       throw new AndroidRuntimeException(
               "Calling startActivity() from outside of an Activity "
               + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
               + " Is this really what you want?");
   }
   mMainThread.getInstrumentation().execStartActivity(
           getOuterContext(), mMainThread.getApplicationThread(), null,
           (Activity) null, intent, -1, options);
}
```

此处加了一个是否是系统进程调用，如果是的话会打印一句log然后继续向下。那么我们可以看到核心代码其实就是那么一句话。

```java
mMainThread.getInstrumentation().execStartActivity(
           getOuterContext(), mMainThread.getApplicationThread(), null,
           (Activity) null, intent, -1, options);
```

调用Instrumentation的execStartActivity方法进行Activity的启动。

### 真正的实现

我们继续进行源码分析。源码过多，所以画了个流程图

![charts](/img/2017-7-12/charts.png)

所以到了performLaunchActivity终于返回了Activity

### handleLaunchActivity之后做了什么

performLaunchActivity是生成了一个Activity，并且执行了onCreate方法
生成之后返回到handlerLaunchActivity，进行继续向下分发。

```java
Activity a = performLaunchActivity(r, customIntent);
```

成功生成了一个Activity，之后开始执行Activity的生命周期

```java
//进入到这个方法
handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
                    
//在handlerResumeActivity方法中开始执行生命周期
r = performResumeActivity(token, clearHide, reason);
```

接着就开始执行生命周期的函数了

```java
r.activity.performResume();     //开始调用

final void performResume() {
    performRestart(); //开始执行
    //...此处省略
    mInstrumentation.callActivityOnResume(this); //使用Instrumentation来执行resume
}

final void performRestart() {
    //restart
    performStart(); 
}
```

```java
public void callActivityOnResume(Activity activity) {
   activity.mResumed = true;
   activity.onResume();
   
   if (mActivityMonitors != null) {
       synchronized (mSync) {
           final int N = mActivityMonitors.size();
           for (int i=0; i<N; i++) {
               final ActivityMonitor am = mActivityMonitors.get(i);
               am.match(activity, activity, activity.getIntent());
           }
       }
   }
}
```

全文结束

### 总结

Instrumentation是一个很重要的类，他负责进行intent分发，调用Activity的onCreate，调用Activity的onResume，在插件化中也有广泛的应用。在查看启动流程中，可以看到在权限检查的步骤进行了在mainfest中的查找，这是一个hook点，可不可以在这里做一些事情来做插件化。其次，使用Instrumentation在跳转的时候动态加载dex，这些点都是可以更改的。查看启动流程不能得到直接的大的收益，更多的是对整个流程的了解，之后再根据需求来做一些事情。



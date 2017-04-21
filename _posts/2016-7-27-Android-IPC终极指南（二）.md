---
layout: post
title: "Android IPC终极指南（二、使用Messenger进行通信）"
date: 2016-7-27
category: Android
tag: IPC
---

> 此篇文章是看完的Android开发艺术的个人总结，摘抄了很多书上的语句，特此说明

IPC学习比较难，我们由浅入深，上一节我们了解了多进程的基础知识，这一节我们尝试进行多进程通信。

#### 武器01：Messenger

Messenger可以翻译为信使，通过它可以在不同的进程中传递Message对象，使用比较辩解，他是一种轻量级的IPC方案，底层是用AIDL实现的，他对AIDL进行了封装，所以不用我们进行繁琐的AIDL编写，同时，**他一次处理一个Message**，因此我们不需要考虑线程同步的问题，因为这根本不存在并发的情况 0.0

#### 再举个栗子

我们来实现一个简单的服务端客户端通信程序，要多简单多简单~~

客户端

```java
public class MainActivity extends AppCompatActivity {

    private static final String TAG = "MainActivity";

    private Messenger mService;

    private ServiceConnection serviceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            mService = new Messenger(iBinder);
            Message msg = Message.obtain(null,1);
            Bundle data = new Bundle();
            data.putString("msg","爷叫你，你敢不应？");
            msg.setData(data);
            try {
                mService.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {

        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Intent intent = new Intent(this,MessengerService.class);
        bindService(intent,serviceConnection,BIND_AUTO_CREATE);
    }

}
```

服务端

```java
public class MessengerService extends Service {

    private static final String TAG = "MessengerService";

    private static class MessengerHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                //来自服务端
                case 1:
                    Log.d(TAG, msg.getData().getString("msg"));
                    break;
                default:
                	super.handleMessage(msg);
                    break;
            }
        }
    }

    private final Messenger messenger = new Messenger(new MessengerHandler());

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return messenger.getBinder();
    }

}
```

别忘了在manifests文件中注册service

```xml
    <service android:name=".MessengerService"  android:process=":remote"/>
```

上面代码实现了客户端绑定的时候给另一个进程的Service使用Messenger发送了一个"爷叫你，你敢不应？"

- 客户端使用服务端拿来的iBinder来初始化Messenger，Messenger就是一个通道，前面说过Messenger内部是用Binder实现的，这里可以用传过来的Binder去获取跟服务端通信的Messenger
- 服务端的Service在onBind方法中，返回服务端生成的Messenger的Binder给客户端。然后消息的处理使用Handler来进行，自定义一个Handler来进行消息的处理，用这个自定义的Handler去生成可用的Messenger
- 两者通信使用Message来进行，使用Handler来当作处理媒介，使用Messenger来当作通道，一次只能处理一条客户端发来的消息
- 所有进行传递的数据都必须是可序列化的，你懂的 0.0，我们会神器的发现Message中有个Obj字段可以赋值，然而2.2之前Obj不支持跨进程传输，2.2之后仅支持系统提供的实现了Parcelable的对象才可以进行传递，所以这字段（传说中的摆设 0.0）。。。，还好，我们可以用Bundle进行各种愉快的传输

#### 栗子加强版（加糖）

上面的栗子仅仅服务端接受客户端发来的程序，而没有进行双向通信，

客户端

```java
public class MainActivity extends AppCompatActivity {

    private static final String TAG = "MainActivity";
    private Messenger mService;
 	//添加个新的Messenger
    private Messenger mGetReplyMessenger = new Messenger(new MessengerHandler());
    private static class MessengerHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case 2:
                    Log.d(TAG,msg.getData().getString("msg"));
                    break;
                default:
                    break;
            }
        }
    }

    private ServiceConnection serviceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            mService = new Messenger(iBinder);
            Message msg = Message.obtain(null,1);
            Bundle data = new Bundle();
            data.putString("msg","爷叫你，你敢不应？");
            msg.setData(data);
          	//此处放进去
            msg.replyTo = mGetReplyMessenger;
            try {
                mService.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {

        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Intent intent = new Intent(this,MessengerService.class);
        bindService(intent,serviceConnection,BIND_AUTO_CREATE);
    }

}
```

服务端

```java
public class MessengerService extends Service {

    private static final String TAG = "MessengerService";

    private static class MessengerHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                //来自服务端
                case 1:
                    Log.d(TAG, msg.getData().getString("msg"));
                	//此处修改
                    Messenger messenger = msg.replyTo;
                    Message message = Message.obtain(null, 2);
                    Bundle data = new Bundle();
                    data.putString("msg", "服务端收到了，不服你来打我啊！");
                    message.setData(data);
                    try {
                        messenger.send(message);
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                    break;
                default:
                    super.handleMessage(msg);
                    break;
            }
        }
    }

    private final Messenger messenger = new Messenger(new MessengerHandler());

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return messenger.getBinder();
    }

}
```

 ![1](/img/2016-7-27/1.png)

Bingo，成功！
我们可以看到，这是一个很好理解的通信过程

- 客户端，我自己弄个Messenger给你，你往这里面发
- 服务端，我的Messenger给你了，客户端你往这里面发
- 一人一个Handler，各自处理来的Message
- 对方的Messenger在replyto中存着，自己拿！！！

咳咳，很好。

#### 小总结

Messenger（信使）的使用还是很方便的，便于理解，这是Android给我们封装好的AIDL，所以使用肯定十分亲民，但是他的弊端就是只能一个一个处理过来的Message，如果在大量消息并发的情况下不太友好 0.0，而且信使的功能比较简单，仅仅是传递消息，有的时候我们需要跨进成调用服务端方法，那么信使的功能就不够强大了，所以我们下一节讲述一下AIDL，强大的通信模式。
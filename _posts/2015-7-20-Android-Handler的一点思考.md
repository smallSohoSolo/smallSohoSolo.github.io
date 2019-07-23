---
layout: post
title: "Android Handler的一点思考"
date: 2015-7-20
category: Android
tag: Handler
---

#### 什么是handler

handler是Android给我们提供的一种消息接受机制，通过handler可以实现在子线程中更新UI，默认生成的handler是运行在ui线程中的。

#### 什么是Looper和MessageQueue

handler中存在一个MessageQueue（消息队列）
Looper是一只“手”，每一个线程只能有一个Looper，这只手无限循环MessageQueue从中拿出来要处理的message，然后交给handler处理，默认生成的handler中的looper是主线程也就是ui线程的looper，所以默认生成的handler是可以处理更新ui的操作的。

```java
handler = new Handler() {
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                switch (msg.what) {
                    case 1:
                        textView.setText("button被点击");
                        Log.d("chenlongbo", "操作1完成");
                        break;
                    case 2:
                        Log.d("chenlongbo", "操作2完成");
                        break;
                    default:
                        break;
                }
            }
        };
findViewById(R.id.button).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                handler.sendEmptyMessage(1);
            }
        });

```

上述代码实现了一个运行在主线程中的handler实现的方法。
Looper是隐含在其中的
这里的handler是直接new出来的，所以其中的handler是ui线程的looper，还有一种方式是利用handlerthread来达到使用子线程的looper处理handler操作

#### 什么是HandlerThread

HandlerThread是Thread的子类，其中自带一个Looper和MessageQueue
HandlerThread由于自带Looper和MessageQueue，所以其他线程同他交互就可以直接用handler给她发送Message，原理和方法同主线程一致。
可以用HandlerThread来做线程间通信


```java
        HandlerThread handlerThread = new HandlerThread("longlongtest");
        handlerThread.start();
		//此处必须start，不然无法生成子线程的looper
        handler = new Handler(handlerThread.getLooper()) {
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                switch (msg.what) {
                    case 1:
                        Log.d("chenlongbo", "操作1完成");
                        break;
                    case 2:
                        Log.d("chenlongbo", "操作2完成");
                        break;
                    default:
                        break;
                }

            }
        };
        findViewById(R.id.button).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                handler.sendEmptyMessage(1);
            }
        });
```


注意，因为此时handler处于子线程中，Android规定子线程无法操作ui。所以此处无法操作ui

#### HandlerThread给我们解决了什么问题

HandlerThread给我们提供了一个线程之间通信的模型，我们可以通过Handler给不同的HandlerThread发送消息。来做到线程间通信的功能。MessageQueue中的消息处理是队列的，所以不会产生并发的问题。

```java

        HandlerThread handlerThread = new HandlerThread("longlongtest");
        handlerThread.start();
        //此处new Handler的时候传入对应Handler的looper
        handler = new Handler(handlerThread.getLooper()) {
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                switch (msg.what) {
                    case 1:
                        try {
                            Thread.sleep(5000);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        Log.d("chenlongbo", "操作1完成");
                        break;
                    case 2:
                        try {
                            Thread.sleep(5000);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        Log.d("chenlongbo", "操作2完成");
                        break;
                    default:
                        break;
                }

            }
        };
        handler.sendEmptyMessage(1);
        handler.sendEmptyMessage(2);
```

#### 总结

Handler给我们提供了一套消息机制，可以让我们顺次执行我们放在队列里面的操作，也可以在子线程中执行耗时操作之后发送消息到主线程的messagequeue中之后更新主线程，但是在handler的handlemenssage方法中是不能执行耗时操作的，那会卡顿ui线程，因为handler message中的执行是在主线程中执行的。

HandlerThread就是帮我们封装了一个有Looper，MessageQueue的线程模型以供我们使用，具体使用方式十分灵活。而且可以在handlermessage方法中执行耗时操作，因为子线程中是可以执行耗时操作的。


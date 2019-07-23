---
layout: post
title: "Android View事件分发解析"
date: 2019-7-23
category: Android
tag: view
typora-root-url: ..
---

### 事件分发的整体的流程图 ###

![event](/img/2019-7-23/event.png)

### 细节原则 ###

1. 事件一定按照图里面的线走。
2. 图里面没标记true 或者 false的线就是一定会走的。
3. 如果事件被消费了，后续的事件都会按照最短路线传给消费的View。
4. onTouchListener > onTouchEvent > onLongClickListener > onClickListener消费顺序
5. 子View可以调用requestDisallowInterceptTouchEvent 方法进行设置，从而阻止父 ViewGroup的onInterceptTouchEvent 拦截事件，强制返回false。
6. dispatch必须调用super方法，否则后面的事件无法连续的被分发，原因是dispatch是分发事件用的，或者你可以手写分发
7. intercept 和 touch 可以不调用super，不会影响事件分发
8. 如果一个ACTION_DOWN被其他View消费，后续事件被父布局（必须是ViewGroup）的InterceptTouchEvent 返回 true，那么会出现一个ACTION_CANCEL 发送给之前消费的View或者ViewGroup

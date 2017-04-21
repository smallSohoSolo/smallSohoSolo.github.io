---
layout: post
title: "Android简易自定义View（一）"
date: 2015-5-26
category: Android
tag: 自定义View
---
#### 基础知识

> 引用官方文档一段话觉得说的挺好

设计良好的类总是相似的。它使用一个好用的接口来封装一个特定的功能，它有效的使用CPU与内存，等等。为了成为一个设计良好的类，自定义的view应该:

- 遵守Android标准规则。
- 提供自定义的风格属性值并能够被Android XML Layout所识别。
- 发出可访问的事件。
- 能够兼容Android的不同平台。

首先Android平台上面的自定义View需要满足以上四种条件，自定义View说白了就是Android系统自带的控件满足不了我们的需求，比如说我们现在来定义一个简单的控件

![图片](/img/2015-5-26/sample2.png)

我们实现这个一个东东，用几个布局组合出来一个控件，外面一个RelativeLayout然后来一个Linearlayout与底部对齐并且更改半透明颜色 **继承LinearLayout实现**

- 实现自定义底部图片
- 实现自定义下部文字
- 实现自定义文字位置（靠左，靠右，居中）

#### View和ViewGroup的区别

1. ViewGroup是View的子类
2. ViewGroup可以拥有子控件，比如常见的LinearLayout,RelativeLAyout等都是ViewGroup。
3. 所有的可视化组件都是View

#### 一个View中都有那些方法


分类     | 方法|   描述   |
-------- | ----|--------|
创建 	   |Constructors		   |View中有两种类型的构造方法，一种是在代码中构建View，另一种是填充布局文件构建View，第二种构造方法要解析并应用布局文件中定义的任何属性。
   		|onFinishInflate()		|在来自于XML的View和它所有的子节点填充之后被调用。
Layout 	| onMeasure				|调用该方法来确定view及它所有子节点需要的尺寸
		|onLayout				|当view需要为它的所有子节点指定大小和布局时，此方法被调用
        |onSizeChanged			|当这个view的大小发生变化时，此方法被调用
Drawing |onDraw					|当view渲染它的内容时被调用
事件处理  |onKeyDown|Called when a new key event occurs.
		|onKeyUp				|Called when a key up event occurs.
    	|onTrackballEvent		|当轨迹球动作事件发生时被调用
    	|onTouchEvent			|Called when a touch screen motion event occurs.
Focus	|onFocusChanged			|Called when the view gains or loses focus.
		|onWindowFocusChanged 	|Called when the window containing the view gains or loses focus.
Attaching|onAttachedToWindow	|Called when the view is attached to a window.
		|onDetachedFromWindow	|Called when the view is detached from its window.
        |onWindowVisibilityChanged|Called when the visibility of the window containing the view has changed.
        

#### 一个自定义View的雏形

Android中所有自定义View都需要继承View或者继承ViewGroup（有时候你会看见一个控件继承LinearLayout，LinearLayout就是一个ViewGroup），比如说：

```java
public class MyView extends LinearLayout //继承ViewGroup一样 {

    public MyView(Context context) {
        super(context);
    }

    public MyView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public MyView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }
//
//    public MyView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
//        super(context, attrs, defStyleAttr, defStyleRes);
//    }
}
```

前面三个constructor是要求最低API 7，最后那个我注释掉最低SDK支持到Android L

Context：传入的上下文

#### 实现自定义属性

AttributeSet：可以通过传入一个attr达到在XML中填写我们自己的属性，例如

```xml
 <com.smallsoho.MyImage
            android:layout_width="fill_parent"
			android:layout_height="fill_parent"
            android:layout_weight="1"
            myapp:text="逗比"
			myapp:drawable="@drawable/ic_luncher"/>
```

稍后我会讲解这个这个详细的使用方法。

#### 其余两个属性

**defStyleAttr**：这个是当前Theme中的一个attribute，是指向style的一个引用，当在layout xml中和style中都没有为View指定属性时，会从Theme中这个attribute指向的Style中查找相应的属性值，这就是defStyle的意思，如果没有指定属性值，就用这个值，所以是默认值，但这个attribute要在Theme中指定，且是指向一个Style的引用，如果这个参数传入0表示不向Theme中搜索默认值

**defStyleRes**：这个也是指向一个Style的资源ID，但是仅在defStyleAttr为0或defStyleAttr不为0但Theme中没有为defStyleAttr属性赋值时起作用

#### 下一节
我们首先讲解简单好用的继承一个LinearLayout实现我们的第二个图片控件


---
layout: post
title: "Android简易自定义View（二）"
date: 2015-5-26
category: Android
tag: 自定义View
---

#### 实现上篇的自定义属性

本章详解讲解一下自定义属性

> 引用官方文档一段话

为了添加一个内置的View到你的UI上，你需要通过XML属性来指定它的样式与行为。良好的自定义views可以通过XML添加和改变样式，为了让你的自定义的view也有如此的行为，你应该:

- 为你的view在资源标签下定义自设的属性
- 在你的XML layout中指定属性值
- 在运行时获取属性值
- 把获取到的属性值应用在你的view上

让我们来实现一下上篇的那个简单的图片控件

为了定义自设的属性，添加 资源到你的项目中。放置于res/values/attrs.xml文件中。下面是一个attrs.xml文件的示例:

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <declare-styleable name="MyView">
        <attr name="imageText" format="string" />
        <attr name="textLocation" format="enum">
            <enum name="left" value="0" />
            <enum name="middle" value="1" />
            <enum name="right" value="2" />
        </attr>
        <attr name="imageRes" format="reference" />
    </declare-styleable>
</resources>
```
其中实现了三个东西，一个是图片下面的白色文字，一个是这个文字显示的位置，一个是图片的图片资源。
name是在xml中定义的时候就要输入这个名字，format输入你这个属性可以用什么填写。

format参考值：

- reference：参考某一资源ID
- color：颜色值
- boolean：布尔值
- dimension：尺寸值
- float：浮点值
- integer：整型值
- string：字符串
- fraction：百分数
- enum：枚举值
- flag：位或运算

#### 在XML中写入我们自己实现的View

```xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >

    <com.longlong.myview.MyView
	    xmlns:myapp="http://schemas.android.com/apk/res/com.longlong.myview"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        myapp:imageRes="@drawable/images"
        myapp:textLocation="middle"
        myapp:imageText="@string/hello_world" />

</RelativeLayout>
```

#### 应用自定义属性

> 引用文档中的精髓

当view从XML layout被创建的时候，在xml标签下的属性值都是从resource下读取出来并传递到view的constructor作为一个AttributeSet参数。尽管可以从AttributeSet中直接读取数值，可是这样做有些弊端：

- 拥有属性的资源并没有经过解析
- Styles并没有运用上

通过 attrs 的方法是可以直接获取到属性值的，但是不能确定值类型

```java
String title = attrs.getAttributeValue(null, "title");
int resId = attrs.getAttributeResourceValue(null, "title", 0);
title = context.getText(resId));
```

都能获取到 "title" 属性，但你不知道值是字符串还是resId，处理起来就容易出问题，下面的方法则能在编译时就发现问题

取而代之的是，通过obtainStyledAttributes()来获取属性值。这个方法会传递一个TypedArray对象，它是间接referenced并且styled的.

我们在代码中获取到我们的属性

```java
    try {
        imageText = a.getString(R.styleable.MyView_imageText);
        textLocation = a.getInt(R.styleable.MyView_textLocation, -1);
        imageRes = a.getDrawable(R.styleable.MyView_imageRes);
    } finally {
        a.recycle();
	}
```

获取到之后我们就应该继续完善我们的代码

```java
        //放进布局
        v = LayoutInflater.from(context).inflate(R.layout.myview, this, true);
        TextView tv = (TextView) v.findViewById(R.id.text);
        ImageView im = (ImageView) v.findViewById(R.id.imageRes);
        tv.setText(imageText);
        tv.setGravity(textLocation);
        im.setImageDrawable(imageRes);
```

这样我们就设置好了各项属性，控件也绑定好了，因为我们是继承自ViewGroup的所以需要重写方法onLayout(),此方法比较高级，在这里我们不做过多讲解，直接继承LinearLayout即可

整个自定义View类就在这里

```java
public class MyView extends LinearLayout {

    private View v;
    private String imageText = null;
    private int textLocation = -1;
    private Drawable imageRes = null;

    public MyView(Context context, AttributeSet attrs) {
        super(context, attrs);
        //获取数据
        TypedArray a = context.getTheme().obtainStyledAttributes(attrs, R.styleable.MyView, 0, 0);

        try {
            imageText = a.getString(R.styleable.MyView_imageText);
            textLocation = a.getInt(R.styleable.MyView_textLocation, -1);
            imageRes = a.getDrawable(R.styleable.MyView_imageRes);
        } finally {
            a.recycle();
        }

        Log.d("ceshi ", imageText + textLocation + imageRes);

        //放进布局
        v = LayoutInflater.from(context).inflate(R.layout.myview, this, true);
        TextView tv = (TextView) v.findViewById(R.id.text);
        ImageView im = (ImageView) v.findViewById(R.id.imageRes);
        tv.setText(imageText);
        tv.setGravity(textLocation);
        im.setImageDrawable(imageRes);
    }
}

```

放进布局即可使用





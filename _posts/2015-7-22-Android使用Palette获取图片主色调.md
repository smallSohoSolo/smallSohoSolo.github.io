---
layout: post
title: "Android使用Palette获取图片主色调"
date: 2015-7-22
category: Android
tag: Palette
---

#### Palette简介

Palette是Android中的调色板，我们可以通过传递一个Bitmap来获取一个颜色列表，可以通过类中封装的分析算法来获取其中的

- Vibrant(充满活力的)
- Vibrant dark(充满活力的黑)
- Vibrant light(充满活力的亮)
- Muted(柔和的)
- Muted dark(柔和的黑)
- Muted lighr(柔和的亮)

也可以获取一个颜色列表，自己写算法挑选你想获取的颜色

可以实现效果

![图片](/img/2015-7-22/image_palette.jpg)

顾名思义，我们可以获取图片中的上面指出的那几种颜色。

#### 代码演示

首先我们需要从Android的v7依赖包中拿出android-support-v7-palette.jar包放进我们的依赖中，然后才能调出Palette类


```java
//首先获取一个bitmap对象，从bitmap中获取到相应的颜色
final Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.mipmap.test4);
setMainColor(mTextView, bitmap);

//然后开始获取颜色
public void setMainColor2(final View view, Bitmap bitmap) {
        Palette.from(bitmap).generate(new Palette.PaletteAsyncListener() {
            @Override
            public void onGenerated(Palette palette) {
                Palette.Swatch swatch = palette.getMutedSwatch();
//                Palette.Swatch swatch = palette.getVibrantSwatch();
//                Palette.Swatch swatch = palette.getDarkMutedSwatch();
//                Palette.Swatch swatch = palette.getDarkVibrantSwatch();
//                Palette.Swatch swatch = palette.getLightMutedSwatch();
//                Palette.Swatch swatch = palette.getLightVibrantSwatch();
                if (swatch != null) {
                    view.setBackgroundColor(swatch.getRgb());
                } else {
                    Log.e("smallsoho", "swatch为空");
                }
            }
        });
    }
```

其中注释掉的代码功能在第一个模块中我们已经注明了，大家根据需求更改就可以了

#### 上面的六种模式无法适配我的图片

我在开发的过程中，想要获取到图片的主色调，由于图片的数量比较大，而且有的图片是这样的

![图片](/img/2015-7-22/test.jpg)

有的图片是这样的

![图片](/img/2015-7-22/test2.png)

用上述六种模式无法准确的识别出我图片的原色

比如第一张用 Vibrant(充满活力的) 获取到的就是哪个橘黄色部分，显然不对，但是第二张图片可以准确识别，但是换种模式，第二张有不能准确识别，那么问题来了，需求不能改，我们只能探究更好的解决方案。查查API

#### 可控的颜色选择

我们可以这么获取我们的主色调

```java

    public void setMainColor(final View view, Bitmap bitmap) {
    	//首先获取一个Palette.Builder（稍后用到）
        Palette.Builder b = new Palette.Builder(bitmap);
        //设置好我们需要获取到多少种颜色
        b.maximumColorCount(1);
        Log.d("chenlongbo", String.valueOf(b.generate().getSwatches().size()));
        //异步的进行颜色分析
        b.generate(new Palette.PaletteAsyncListener() {
            @Override
            public void onGenerated(Palette palette) {
                //获取颜色列表的第一个
                Palette.Swatch swatch = palette.getSwatches().get(0);
                if (swatch != null) {
                    view.setBackgroundColor(swatch.getRgb());
                } else {
                    Log.e("chenlongbo", "swatch为空");
                }
            }
        });
    }

```

代码解析，首先我们需要知道我们可以通过Palette.Builder来设置我们获取到多少中颜色，默认是16
而且他的颜色获取顺序大体上是按照颜色多少和亮暗进行排序的，注意，这里设置的成100也并不一定会获取到100个颜色，前提是bitmap中有那么多种颜色给他获取。

那么我们想当然，整个图片中的最大面积的颜色就是我们要获取到的那个颜色。我们首先b.maximumColorCount(1);设置到只获取一个颜色，那么这个颜色一定是我们要获取的那个最喜欢的颜色

然后我们通过使用Palette.Swatch swatch = palette.getSwatches().get(0);获取到这个颜色然后设置上去就可以了

#### 效果

![图片](/img/2015-7-22/screen.png)




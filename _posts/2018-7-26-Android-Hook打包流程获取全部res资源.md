---
layout: post
title: "Android Hook打包流程获取全部res资源"
date: 2018-7-26
category: Android
tag: gradle
typora-root-url: ../../myblog
---

随着项目的不断迭代，各种自动化工具的需求会涌现出来，这里列举几种场景

- 由于开发人数众多，需求多，导致图片的放置会很混乱，设计给的大图不经过压缩就放进项目，导致包大小无限膨胀
- 目前某音短视频项目存在多个flavor，可能某些字符串需要动态的使用脚本替换解决App名称不一致的问题（当然这个解决方案有很多种，这里列举一种）

对资源的自动化处理的需求是不可避免的，所以如何处理是一个问题：

### 普通处理

普通处理res，这个我们只要一个脚本递归项目目录即可，然后根据相应的结果进行处理。这个原理十分简单，但是有几个弊端：

- 需要再构建之前执行一次脚本，这个时机的处理是比较动态的，手动执行不够智能，每次打包需要人看着，ci执行的话可以，但是依赖ci服务，如果更换ci服务需要配置新的脚本，插入gradle task 比较好，但是不同版本的插入点可能不是很稳定，当然也可以无脑把自己变成第一个task
- 处理的资源不完整，无法处理第三方aar和jar中的资源，这是比较重要的，因为会影响效果

所以弊端比较多，这里不推荐这么处理资源。

### Hook打包流程处理

Hook打包流程处理是本文的主要核心方案，这里开源项目

McIamge：[https://github.com/Mobcase/McImage](https://github.com/Mobcase/McImage)

这个插件，全自动压缩整个项目中的图片，能有效减少包大小。使用的即是这种方案，推荐大家集成。

Hook打包流程会引来几个问题，由于android gradle 2.X 到 3.X的转变，AAPT，即资源处理工具有变化，升级到了AAPT2，所以这里方案不一样。

### Gradle 3.0 构建科普

下面是一张整个构建流程的简单流程图

![Gradle 3.0构建流程图.png](/img/2018-7-26/Gradle 3.0构建流程图.png)

这里我们不需要关注所有的task，只需要关注红框的task即可，这个任务的主要作用就是处理整个app的res

这个AAPT版本的处理方式可以看我之前的一篇文章

[McImage插件解析（0.1.5之前版本）](https://smallsoho.com/android/2017/04/07/McImage%E6%8F%92%E4%BB%B6%E8%A7%A3%E6%9E%90/)

简单说下原理是mergeResource如果使用AAPT1，会在执行之后把res打包到几个指定的文件夹，然后再mergedResource之后插入这个task处理res。

这里重点说一下AAPT2

### AAPT2的变动

这里变动就是AAPT2打包之后，文件变成了flat格式的，无法再用前面叙述的方案来进行解决。

![WX20180726-155241@2x](/img/2018-7-26/WX20180726-155241@2x.png)

所以这里使用新的方案来进行解决

这里分成几个部分，先说一下

#### gradle 3.3版本

之前的版本怎么解决。这里看了下最新版本的gradle的更新

![WX20180726-155612@2x](/img/2018-7-26/WX20180726-155612@2x.png)

很开心，最新版本的更新日志告诉我们，为了让Kotlin适配Android Res资源，3.3之后的gradle会在mergedResources中暴露一个getRawAndroidResources()方法，这样就能一次性拿到所有的资源文件了。皆大欢喜。

#### gradle3.0 - gradle 3.3之间的版本

两种解决方案，一种是关掉AAPT2，3.3版本之前是可以在gradle.properties中填写

```gradle
android.enableAapt2=false
```

关掉AAPT2来强制使用AAPT1，这样我们就能用前面叙述的方案，直接处理mergeResource处理完之后生成的文件夹即可，但是这种方案其实是一种规避，是有缺陷的，3.3版本之后gradle强制使用AAPT2，这个开关会失效，所以这个方法就没有作用了，所以这并不是一个好方法。此处不采用

**好的方案是什么呢？**

这里我们可以看kotlin在3.3之前的版本是如何处理res的呢？

```kotlin
fun MergeResources.computeResourceSetList0(): List<File>? {
    val computeResourceSetListMethod = MergeResources::class.java.declaredMethods
            .firstOrNull { it.name == "computeResourceSetList" && it.parameterCount == 0 }
            ?: return null
    val oldIsAccessible = computeResourceSetListMethod.isAccessible
    try {
        computeResourceSetListMethod.isAccessible = true
        val resourceSets = computeResourceSetListMethod.invoke(this) as? Iterable<*>
        return resourceSets
                ?.mapNotNull { resourceSet ->
                    val getSourceFiles = resourceSet?.javaClass?.methods?.find { it.name == "getSourceFiles" && it.parameterCount == 0 }
                    val files = getSourceFiles?.invoke(resourceSet)
                    @Suppress("UNCHECKED_CAST")
                    files as? Iterable<File>
                }
                ?.flatten()
    } finally {
        computeResourceSetListMethod.isAccessible = oldIsAccessible
    }
}
```

这里可以看到MergedResource里面有一个computeResourceSetList方法，可以反射这个方法获取到res资源文件，很巧妙的获取到了所有的res，所以问题就解决了，我们可以使用这个方法来获取res文件。这个方法可以理解为这个区间的最优解。

### 总结

资源的自动化处理是一个很有意思的课题，可以在里面做很多事情，通过本文的方案可以拿到所有的资源文件夹，来达到对后面逻辑的铺垫任务，希望大家能做出更多好用的插件来^_^
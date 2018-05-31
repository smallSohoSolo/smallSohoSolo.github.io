---
layout: post
title: "Android App Bundle探索"
date: 2018-5-28
category: Android
tag: 组件化
typora-root-url: ../../myblog
---

### 什么是App Bundle

Android App Bundle是Google最新推出的Apk动态打包，动态组件化的技术，通过一个.aab结尾的bundle文件组装一个apk来为你的设备安装。这是一篇科普的文章，后续会对源码进行剖析。
![apk_splits_tree-2x](/img/2018-5-28/apk_splits_tree.png)

通俗理解就是，Appbudle就是把Apk拆分成了多个积木，之前我们是把一个大而全的apk装到你手机里面，但是你其实用不到这么多东西，比如，你xxhdpi的手机屏幕是不需要xhdpi的图片资源的，但是在这之前都是已经打包进去了，会很浪费。

AppBundle将这些特性在多个维度进行拆分，在资源维度，ABI维度和Language维度进行了拆分，你只要按需组装你的Apk然后安装即可，不用安装其他的东西，这堆包大小和方法数还有启动等等有一个十分好的收益。

另外一个重要的创举是支持组件的动态下发，后面会讲到，你可以将一些独立的模块在运行时安装，而不是一次性放到apk里面。这是组件化的一次伟大的创举。

### App Bundle 中的组件

- Base Apk: base为基础模块，包括你业务逻辑中的代码，dex等基础，为主工程的apk
- Configuration Split Apk: 构造apk，区分的维度是[Multiple Apks](https://developer.android.com/studio/build/configure-apk-splits)的划分。用来拼装Dynamic Feature Apks和Base Apk的配置。
- Dynamic Feature Apk: 动态特性Apk，这是组件化的一个好的新方案，通过动态下发模块来做到功能的动态更新

### 兼容性问题

Api < 21的手机无法进行模块化，Google Play会对其进行[Multiple Apks](https://developer.android.com/studio/build/configure-apk-splits)操作来进行下发操作。

当你创建一个Dynamic Module的时候，下图

![动态组件Module创建](/img/2018-5-28/dynamic_module_create.png)

上面有两个选项，一个文本框

- Enable on-demand: 是否启用按需下载，如果不启用，会直接打进Apk
- Fusing：熔断操作，是否安装到不支持按需下载的设备中
- Module title: 模块标题

### 对于动态组件的一些使用场景

##### 语言包的动态下发

当Split 针对语言进行划分时候，用户下载的Apk仅仅只能下载下来一个Base Apk，包含他的当前系统语言，你可以将其他语言包作为Dynamic feature下发给用户，做到语言包的动态下发

##### 功能的动态下发

对于某些独立的feature，这其实跟之前的插件化方案有异曲同工之处，而且天然支持友好，动态下发业务需求能有效的减少包大小，增加启动速度，减少安装时间等等。

##### 热修复场景

通过简单的逻辑判断，可以直接用下发下来的feature来进行对当前feature的替换使用，做到热修复的效果。而且无需考虑后期的版本升级问题。

### 关于动态模块的一些注意事项

1. 当打开on-demand（按需加载）时，必须开启Fusing（熔断操作）才能正常的让Api21以下的手机使用module
2. 一般情况下，动态模块下发之后需要重启App才能加载成功，但是如果你使用SplitCompat library，就可以立即生效，[Access code and resources from downloaded modules](https://developer.android.com/guide/app-bundle/playcore#access_downloaded_modules)
3. 如果下载的模块太大，需要用户确认，GP要求大于10MB需要用户确认
4. module中的AndroidManifest中定义的Activity不能有exported:true因为别的app不知道你何时安装好模块从而会引发问题
5. proguard文件在生效的时候会merge base module和所有的dynamic module中的文件，所以在编写proguard的时候要注意这个问题。
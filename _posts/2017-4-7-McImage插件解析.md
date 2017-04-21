---
layout: post
title: "McImage插件解析"
date: 2017-4-7
category: Android
tag: gradle
---

### 介绍

McImage插件传送门：[https://github.com/Aweme/McImage](https://github.com/Aweme/McImage)
McImage是一个对资源中的png和jpg图片进行压缩和图片大小检查的插件，通过pngquant算法对图片资源进行压缩，并且可以设置图片最大大小来在打包时候对所有图片资源进行检查。如果发现大于设置的大小，默认为1M，那么就会中断打包，并且提示是哪张图

### 与tinypng的比较

McImage插件的优势：

- tinypng是商业api，对图片压缩有限额
- github上的主流tinypng的插件并没有压缩项目中依赖的图，仅仅压缩了你项目中自己的图
- McImage可以对一个Module中所有的依赖进行处理，包括jar中的图和aar中的图
- 采用的算法pngquant仅仅是McImage的一个插件，可以随时替换算法，如果出现更加好用的算法，McImage会及时更换。主流的tinypng插件仅仅适用于tinypng

McImage插件的劣势：

- tinypng采用自研算法，压缩效率高于开源的pngquant算法，大约百分之8左右的压缩度

McImage插件对全量依赖的压缩可以尽可能的减少着百分之8左右的压缩带来的劣势

### 如何在打release时使用

可以在任意module的build.gradle中加入下列代码，即可在打release时才进行检查，使用默认参数。

```groovy
def isRelease() {
    project.gradle.startParameter.taskNames.each { arg ->
        if ((arg == "assembleRelease" || arg == "aR") || arg == "ar" || arg == "resguardRelease") {
            return true
        }
    }
    return false
}

if (isRelease()) {
    apply plugin: 'McImage'
}
```

### 实现原理

首先可以看看我的其他博客 [Android自定义插件](http://smallsoho.com/2017/03/18/Android%E8%87%AA%E5%AE%9A%E4%B9%89Gradle%E6%8F%92%E4%BB%B6.html) 来学习，如何实现一个Gradle插件

#### Gradle的打包流程

![](/img/2017-4-7/14915625760947.jpg)


这是一张Android的整个打包流程图片。我们主要关注红色方框区域，这个区域做的事情是两个步骤

1. 首先把一个Module的所有依赖的Mainfest，asset，resources分类合并
2. 将合并好的放到指定文件夹，然后传给aapt进行打包

#### Hook打包流程

我们知道Android的打包使用的Gradle来进行的，上图演示了Gradle的打包流程，所有的流程都有对应的Task，这里在众多的Task中我们主要关注两个Task

**mergeDebugResources**

- 解压所有的aar包输出到app/build/intermediates/exploded-aar
- 把所有的资源文件合并到app/build/intermediates/res/merged/debug目录里

**processDebugResources**

- 调用aapt生成项目和所有aar依赖的R.java,输出到app/build/generated/source/r/debug目录
- 生成资源索引文件app/build/intermediates/res/resources-debug.ap_
- 把符号表输出到app/build/intermediates/symbols/debug/R.txt 

上述两个流程就是红框圈出来的位置的两个重要的task。

那么我们应该hook哪个？首先我们看到第一个他做的事情总结到一点就是合并，第二个Task做的事情总结到一点就是处理合并。

当mergeDebugResources处理完成之后，相关文件会产生在下列目录中

![](/img/2017-4-7/14915657726242.jpg)

可以看到res中是我们的打包好的res文件，我们处理这个目录即可。
processDebugResources稍后会使用这个目录下的文件生成相关的map，所以我们在processDebugResources执行的前一步插入我们的插件就比较恰当。

### 源码解析

首先整体目录

![](/img/2017-4-7/14915652785690.jpg)

- CompressUtil：使用压缩库进行压缩的工具类
- Config：在Gradle中的配置Model
- ImagePlugin：插件主工程
- SizeUtil：检查图片大小的工具类

#### Config

```groovy
class Config {
    def maxSize = 1 * 1024 * 1024
    def isCheck = true
    def isCompress = true
}
```

首先Model中分别配置了检查文件的大小，是否检查和是否压缩，如果isCheck置为false那么将不进行检查，isCompress置为false将不进行压缩

#### CompressUtil

关键代码

```groovy
def command = "${rootDir.getParentFile().getPath() + '/mctools/'}pngquant --skip-if-larger --speed 11 --force --output ${imgFile.getPath()} -- ${imgFile.getPath()}"
def proc = command.execute()
proc.waitFor()
```

通过groovy调用shell操纵pngquant的二进制库，来进行图片压缩，执行输入输出。

#### SizeUtil

```groovy
if ((imgFile.getName().endsWith('.jpg') || imgFile.getName().endsWith('.png'))
                && !imgFile.getName().contains('.9')) {
    println 'Start Check ' + imgFile.getPath()
    if (imgFile.length() >= maxSize) {
        return true
    }
}
```

进行一步判断然后根据输入的文件大小来检查返回true或者false

#### ImagePlugin

此处是插件的入口，首先进行了一些安全检查，包括读取配置，检查渠道，读取merge目录。然后对查到的目录进行递归检查资源

```groovy
//load img
dir.eachDir() { channelDir ->
    channelDir.eachDir { drawDir->
        def file = new File("${drawDir}")
        if (file.name.contains('drawable') || file.name.contains('mipmap')) {
            file.eachFile { imgFile ->
                if (config.isCheck && SizeUtil.checkImgSize(imgFile, config.maxSize)) {
                    bigImgList.add(file.getPath() + file.getName())
                }
                if (config.isCompress) {
                    CompressUtil.compressImg(imgFile, project.projectDir)
                }
            }
        }
    }
}
```

插入自定义插件

```groovy
project.tasks.findByName(mcPicPlugin).dependsOn processResourceTask.taskDependencies.getDependencies(processResourceTask)
processResourceTask.dependsOn project.tasks.findByName(mcPicPlugin)
```

原理就是更改processResources任务的依赖指针，跟链表插入节点的原理一致。

### 整体梳理

整体流程并不难，找到关键的Gradle Task位置，然后插入我们的task，接着对资源进行压缩和检查，即可完成所有流程。

### 扩展

可以自行修改CompressUtil更换压缩算法，曾经使用Google的无损压缩Guetzli和zopfli进行对jpg和png的处理，此两种算法都是无损压缩。pngquant是有损压缩，不过损失度在可接受范围内。google两个算法的劣势是压缩时间比较长，我用生产环境项目测试大概花费半个小时，而且压缩率只能到百分之30，所以权衡之后换成pngquant。



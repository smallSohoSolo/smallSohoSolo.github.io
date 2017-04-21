---
layout: post
title: "Android自定义Gradle插件"
date: 2017-3-18
category: Android
tag: aop
---

最近各种热修复框架开源，其中使用instant run方式的框架是目前兼容性最好的，典型的有蘑菇街的Aceso和美团的Robust。两者都是使用ASM在编译的时候进行方法级别的修改来达到在运行时热修复。具体原理不详谈，本篇文章主要讲述使用插件在编译时候进行处理。

基础流程：Gradle会将Java代码打包成class，我们可以在打包成class之后插入我们的插件，在插件中使用asm来进行对字节码的操作。

插件的插入方式使用的是Gradle 1.5之后的特性Transform API。

### 编写一个简单可用的Gradle插件

首先创建一个Java Library，取名为libplugin
按照以下方式创建项目

![](/img/2017-3-18/14898466364058.jpg)

此处有坑：注意下面的META-INF和gradle-plugins是父子目录关系，不是一个目录名称，AS可能会给你优化到一个文件夹，导致你插件无法apply，请分开成父子目录写。

##### 首先在src目录下创建groovy和resources目录。

- groovy包用来存储gradle代码，gradle是用groovy来实现的
- resources目录存放的是gradle插件的名称，在外界apply需要根据你定义的这个文件名来进行apply，这里我命名为com.smallsoho.plugin.properties，那么在外界使用就需要apply plugin:'com.smallsoho.plugin'

com.smallsoho.plugin.properties文件内容如下：

```
implementation-class=com.smallsoho.plugin.PluginMain
```

此处存放指定插件的实现是哪个类，这个类在我上面写groovy的com.smallsoho.plugin包下

##### 其次解释groovy包下的代码

com.smallsoho.plugin是包名，具体什么可以自定义，下面是两个类，一个是PluginExt类，是一个Model，里面的代码

```groovy
package com.smallsoho.plugin
class PluginExt {
    String name;
}
```

我们平时在其他的gradle插件中总会定义一些东西，这里我们可以定义一个model来做一会往插件传参数的model

然后是插件的代码

```grooy
package com.smallsoho.plugin

import org.gradle.api.Plugin
import org.gradle.api.Project

class PluginMain implements Plugin<Project> {

    @Override
    void apply(Project project) {
        println 'hello, world!'
    }
}
```

这里我们写的是创建一个Plugin插件的标准方式，我们只要实现apply方法。方法传入的是我们的项目，也就是我们在其他模块下写apply的那个project，project中存储着这个我们传进来的项目的各种参数。此处我们不使用任何东西，仅仅打印一句Hello Plugin。

##### 修改插件的build.gradle

```groovy
apply plugin: 'groovy'
apply plugin: 'maven'

buildscript {
    repositories {
        jcenter()
    }

    dependencies {
        classpath 'com.android.tools.build:gradle:2.3.0'
    }
}

dependencies {
    compile gradleApi()//gradle sdk
    compile localGroovy()//groovy sdk
}

uploadArchives {
    repositories {
        mavenDeployer {
            repository(url: uri("$rootDir/maven"))
            pom.groupId = 'com.smallsoho.plugin'
            pom.artifactId = 'soloplugin'
            pom.version = '1.0.0'
        }
    }
}
```

此处我们在项目的根目录下先手动创建一个目录名称为maven，我们知道gradle的包管理都是存储在maven仓库中的，我们这里定义一个本地的maven仓库，首先我们打包到本地的maven仓库，然后在主项目中依赖我们打包好的包。

代码开始我们apply了一个groovy语言插件，和一个maven插件。然后我们需要依赖两个编译，用于我们写groovy插件，其次我们在下面写了uploadArchives，用于上传我们本次写的插件到maven文件夹下。

##### 把我们的插件发送到本地仓库

点击下列按钮就能将我们的插件打包上传到本地maven，成功之后会在刚才创建的maven目录下面生成对应的jar

![](/img/2017-3-18/14898538240168.jpg)

##### 应用我们的插件

在根目录的build.gradle下配置

```groovy
import com.smallsoho.plugin.PluginMain

buildscript {
    repositories {
        jcenter()
        maven {
            url uri("$rootDir/maven")
        }
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.3.0'
        classpath 'com.smallsoho.plugin:soloplugin:1.0.0'
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

//其他省略

apply plugin: 'com.smallsoho.plugin'
```

此处有可能你会弹出一个error

```
Error:(2, 0) Plugin with id 'com.smallsoho.plugin' not found.
```

检查上面的META-INF和gradle-plugins是不是父子文件夹，as可能会给你优化成一个目录。

##### 验证一下

构建一下项目，你会发现

![](/img/2017-3-18/14898542100728.jpg)

成功打印出来了我们的项目

### 在build.gradle中使用自定义参数

在引入插件的Gradle文件中定义

```groovy
pluginext {
    name 'smallsohosolo'
}
```

修改PluginMain类为

```groovy
public class PluginMain implements Plugin<Project> {

    @Override
    void apply(Project project) {
        //将pluginext和PluginExt连接起来
        project.extensions.create('pluginext', PluginExt)
        //创建一个任务
        project.task('hello') {
            doLast {
                println project.pluginext.name + ' 你好'
            }
        }
    }

}
```

在Gradle中执行

```
./gradlew hello
```

成功输出了我们在Gradle中定义的变量。









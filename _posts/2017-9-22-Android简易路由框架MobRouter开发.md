---
layout: post
title: "Android简易路由框架MobRouter开发"
date: 2017-9-22
category: Android
tag: router
---

### Android Router介绍

Android Router即是Android路由工具，通过使用路由工具，可以做到通过一句话进行跳转，无需知道具体的Activity类。

```java
context.startActivity(context,XXX.class); //常见的跳转
MobRouter.build("image_activity").go(); //使用路由框架跳转
```

通过这种路由跳转的方式，可以做到各个业务模块之间的解耦合。举一个具体的使用场景，我们的模块之间使用了各种AAR的依赖，那么你如果想从一个AAR中跳转到另外一个AAR，那么需要两者有依赖关系才能获取到XXX.class，否则你拿不到类，更不用说跳转了。所以需要一种好的方式来进行解耦合，那么就很需要Router工具。

### Router的使用方式

常见的Router使用方式是在一个Activity的类中使用注解标注

```java
@MobRouterPath({"share_activity"})
public class ShareActivity extends Activity {
	//省略代码
}
```

此处的MobRouterPath标注了这个ShareActivity的名称是什么，之后跳转使用你定义的名称，这里推荐这个名称使用常量来做，能更好的管理名称。

使用MobRouter来进行跳转

```java
MobRouter.build("share_activity")
  .bundle(***) //传入一个bundle
  .with(***) //传入转场动画
  .context(***)  //传入指定的context，默认使用Application
  .XXX //具体的方法可以自行扩展
  .go();
```

此处可以使用各种扩展方法，比如传入转场动画等，就是对Android Api的封装，此处不赘述。

### Router实现细节

Router可以使用两种方式进行跳转

- 使用反射，通过注解拿到对应的class然后进行跳转
- 使用APT，在编译时生成map，从map中查到对应的class进行跳转

使用反射的弊端是，反射是及其消耗性能的，Android官方推荐的是尽可能避免反射，使用APT生成代码的方式可以最大化的减少性能损耗，避免每次跳转都用反射来做。所以我们使用APT来做。

这是一个删掉了无用细节的APT类。

```java
@AutoService(Processor.class)
@SupportedSourceVersion(SourceVersion.RELEASE_7)
public class MainProcess extends AbstractProcessor {

    private Filer mFiler;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        mFiler = processingEnvironment.getFiler();
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnvironment) {
        //...
        Set<? extends Element> elements = roundEnvironment.getElementsAnnotatedWith(MobRouterPath.class);
      
        for (Element element : elements) {
            MobRouterPath routerPath = element.getAnnotation(MobRouterPath.class);
            for (String key : routerPath.value()) {
                //此处进行map输入
            }
        }
        //...
        javaFile.writeTo(mFiler); //把生成好的文件保存
        return true;
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        LinkedHashSet<String> linkedHashSet = new LinkedHashSet<>();
        linkedHashSet.add(MobRouterPath.class.getCanonicalName());
        return linkedHashSet;
    }
}
```

我们可以看到，在编译期间拿到了注解，然后通过注解，使用[Javapoet](https://github.com/square/javapoet)来进行java file 生成，这里产生的对应类为

```java
public class MobRouterPathHolder {
  private static Map sRouterMap = new HashMap<String,Class>();;

  public static Map getRouterMap() {
    sRouterMap.put("image_activity", ImageActivity.class);
    sRouterMap.put("share_activity", ShareActivity.class);
    return sRouterMap;
  }
}
```

我们可以生成一个静态方法，然后拿到Activity和key的对应的map，从map中读取对应的class进行跳转。

那么如何获取到Map呢？

```java
Class tmp = Class.forName("com.bytedance.mobrouter.MobRouterPathHolder");
            sRouterMap = (Map<String, Class>) tmp.getMethod("getRouterMap", new Class[]{}).invoke(tmp, new Object[]{});
```

这里使用反射获取到map，这句最好在Application的onCreate中进行，防止还没有注册就进行跳转。此处使用反射一次拿到map之后就不用再使用任何反射了，不会有性能问题。有了map，就可以根据map进行跳转了。

### 扩展

1. 可以通过对map的修改对跳转进行修改，而且不会有任何兼容性问题
2. 在众多的插件化框架中，动态加载APK并不会走编译流程，那么可以在下发的APK中加入一个路由表，然后加入到全局的路由map中，就可以实现对下发的插件进行路由而没有任何兼容性问题。

MobRouter是一种思路，具体的更多的业务可以自行发挥，进行更多的开发。
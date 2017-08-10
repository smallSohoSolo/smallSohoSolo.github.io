---
layout: post
title: "[译]使用ASM Core API修改类"
date: 2017-8-7
category: Android
tag: aop
typora-root-url: ../../myblog
---

## 2. 章节

这个章节说明了如何使用Core ASM API 构造并且修改编译Java字节码。开始首先介绍字节码的编译，之后指出相应的ASM接口，组件和工具去构造和修改她们，并且有很多实用的例子。方法，注解和泛型的内容在下一章节说明。

### 2.1. 结构

#### 2.1.1. 概览 

字节码的结构是很简单的。并不像一个编译好的Application，字节码保留了结构信息和大多数的源代码符号。事实上，字节码中包括：

- 类的作用域（例如 public 或 private），名字，父类，接口和注解。
- 类中的每一个变量。包括每一个变量的作用域，名称，类型和注解。
- 类中的每一个方法和构造函数。每一个部分包括作用域，名称，返回值，参数和注解。并且也包括方法编译后的字节码。

源码和字节码有很多的不同

- 一个字节码文件仅描述一个类，当一个源码文件包含多个类的时候。举个例子一个源码文件中的一个类中包含一个内部类，那么他会被编译成两个字节码文件。一个是主类的，另外一个是内部类的。主类的文件中包含对他内部类的引用，并且非静态的内部类也持有对外部类的引用。
- 字节码文件并不包含注释，但是包含函数类，变量，方法和能被用于关联结构的额外信息的attribute。源于对Java5注解的介绍，对于相同作用的代码，attributes 大多数是没有用的。
- 字节码并不包含package和import的部分，所以所有的类型名称都要用完整的名称。

另一个非常重要的结构上的不同点是字节码中包含常量池。常量池是一个包括所有这个类中的numeric，string或者其他类型的常量的array。这些常量仅仅被定义一次，并且被类的全局持有引用。有幸的是ASM隐藏了所有跟常量池相关的细节问题，所以你不用为此苦恼。图2.1显示了字节码的结构，更精确的结构在第四节，Java虚拟机规范。

![](/img/2017-8-7/2-1.png)

<center>图2.1：字节码的整体结构</center>

另一个重要的不同是Java的类型在源码和字节码中的表示不同，下一节会谈到这些东西在字节码中的展示。

#### 2.1.2. 内部名称 

在很多情况下类和接口是有类型的。比如一个类的父类，一个类实现的接口，或者一个方法抛出的异常都不能被原始类型，数组类型，必要类和接口表示。这些类型在字节码中用内部名称表示。内部名称就是一个类的标准名称，点会被替换为斜杠。举个例子，String会被替换为java/lang/String。

#### 2.1.3. 类型描述符

内部名称仅仅用明确的类名或接口类型表示。在所有的其他情况，比如变量类型，Java用类型描述符号在字节码中进行表示。（如图2.2）

![](/img/2017-8-7/2-2.png)

<center>图2.2：一些Java类型的类型描述符</center>

基础类型的类型描述符是一个字母：Z 代表 boolean，C代表char，B代表byte，S代表short，I代表int，F代表float，J代表long，D代表double。类的类型描述符就是类的类型名称。L开头，分号结尾。例如String的类型描述符是Ljava/lang/String;。数组类型的描述符就是方括号开头跟上类型描述符，多维数组就是多个方括号。

#### 2.1.4. 方法描述符

一个方法描述符是一组类型描述符的集合，包括变量类型，返回值类型用一整个字符串来表示。一个方法描述符用左括号开头，接着是变量的类型描述符，然后右括号结束。紧跟着返回值的类型描述符。V代表void。

![](/img/2017-8-7/2-3.png)

<center>图2.3：方法描述符例子</center>

一旦你知道类型描述符怎么工作，理解方法描述符更加简单。举个例子，(I)I 描述了一个参数为int，并且返回值是int的方法。如图2.3给了一些方法描述符的例子。

### 2.2. 接口和组件

#### 2.2.1. 介绍

ASM API 构造和处理字节码是基于ClassVisitor抽象类（如图2.4）。ClassVisitor类中的每一个方法对应着一个图2.1中的结构。简单的结构部分使用一个返回值为void的函数进行处理，复杂的部分会返回一个辅助类Visitor。如visitAnnotation，visitField和visitMethod函数，它们的返回值为AnnotationVisitor，FieldVisitor和MethodVisitor。

原则上可以递归地使用这些辅助函数，举个例子，每一个FieldVisitor（如图2.5）

```java
public abstract class ClassVisitor {
    public ClassVisitor(int api);
    public ClassVisitor(int api, ClassVisitor cv);
    public void visit(int version, int access, String name,String signature, String superName, String[] interfaces);
    public void visitSource(String source, String debug);
    public void visitOuterClass(String owner, String name, String desc);
    AnnotationVisitor visitAnnotation(String desc, boolean visible);
    public void visitAttribute(Attribute attr);
    public void visitInnerClass(String name, String outerName,String innerName, int access);
    public FieldVisitor visitField(int access, String name, String desc,String signature, Object value);
    public MethodVisitor visitMethod(int access, String name, String desc,String signature, String[] exceptions);
    void visitEnd();
}
```

<center>图2.4：ClassVisitor类</center>

对应着类文件中的同名方法，类中的visitAnnotation方法也会返回了一个辅助函数AnnotationVisitor。如何使用和创建这些辅助函数会在下一章中进行讲解：可以确定的是，本章节会指引大家使用ClassVisitor解决一些简单的问题。

```java
public abstract class FieldVisitor {
	public FieldVisitor( int api );
	public FieldVisitor( int api, FieldVisitor fv );
	public AnnotationVisitor visitAnnotation( String desc, boolean visible );
	public void visitAttribute( Attribute attr );
	public void visitEnd();
}
```

<center>图2.5：FieldVisitor类</center>

ClassVisitor中方法的顺序必须按照顺序调用，Java文档中规定：

visit visitSource? visitOuterClass? ( visitAnnotation | visitAttribute )*
( visitInnerClass | visitField | visitMethod )*
visitEnd

这意味着visit方法会被最先调用，紧接着多数情况会调用visitSource，接着多数情况会调用visitOuterClass方法，接着会调用任意数量的visitAnnotation和visitAttribute方法，接着会调用任意数量的visitInnerClass，visitField和visitMethod方法。最后会调用一次visitEnd方法。

ASM 基于ClassVisitor API提供了三种核心组件去构造和更改字节码：

- ClassReader会将字节码转化为一个byte数组，接着会调用ClassVisitor中对应的visitXXX函数作为他的接收函数。ClassReader是事件的生产者。
- ClassWriter是ClassVisitor抽象类的子类，用来编译修改好的字节码。他生产了一个包含了编译好的类的二进制的数组，可以用toByteArray方法获取。ClassWriter是事件的消费者。
- ClassVisitor代理了所有来自其他ClassVisitor实例的方法调用，ClassVisitor是事件过滤器

下一节使用具体的例子来展示如何使用这三个组件来构造和修改字节码。

#### 2.2.2. 解析类

解析类仅仅需要ClassReader就够了。让我们用它来举个例子。我们将打印一个使用javap工具类生成的内容。第一步是写一个ClassVisitor的子类去打印要访问的类的信息。这是一个可用的并且简单的实现：

```java
public class ClassPrinter extends ClassVisitor {
    public ClassPrinter() {
        super(ASM4);
    }

    public void visit(int version, int access, String name, String signature,
        String superName, String[] interfaces) {
        System.out.println(name + " extends " + superName + " {");
    }

    public void visitSource(String source, String debug) {
    }

    public void visitOuterClass(String owner, String name, String desc) {
    }

    public AnnotationVisitor visitAnnotation(String desc, boolean visible) {
        return null;
    }

    public void visitAttribute(Attribute attr) {
    }

    public void visitInnerClass(String name, String outerName,
        String innerName, int access) {
    }

    public FieldVisitor visitField(int access, String name, String desc,
        String signature, Object value) {
        System.out.println(" " + desc + " " + name);

        return null;
    }

    public MethodVisitor visitMethod(int access, String name, String desc,
        String signature, String[] exceptions) {
        System.out.println(" " + name + desc);

        return null;
    }

    public void visitEnd() {
        System.out.println("}");
    }
}
```

第二步是组合ClassPrinter和ClassReader，这样ClassReader生产出来的内容就可以被ClassPrinter消费掉。

```java
ClassPrinter cp = new ClassPrinter();
ClassReader cr = new ClassReader("java.lang.Runnable");
cr.accept(cp, 0);
```

第二行创建了一个ClassReader去解析了Runnable函数，accept方法在最后一行被调用用以解析Runnable的字节码并且让ClassPrinter接收。输出的结果是

```java
java/lang/Runnable extends java/lang/Object {
	run() V
}
```

需要注意的是，有很多种方法可以构造一个ClassReader实例。可以使用名字，值，byte数组或者InputStream来构造。一个输入流的内容可以使用ClassLoader的getResourceAsStream方法读取出来。

```java
cl.getResourceAsStream(classname.replace(’.’, ’/’) + ".class");
```

#### 2.2.3. 构造类

构造类仅仅需要ClassWriter组件。让我们使用它来做一个例子。让我们来看看下面这个接口：

```java
package pkg;
public interface Comparable extends Mesurable {
    int LESS = -1;
    int EQUAL = 0;
    int GREATER = 1;
    int compareTo(Object o);
}
```

在ClassVisitor可以使用6个方法来构造它：

```java
ClassWriter cw = new ClassWriter( 0 );
cw.visit( V1_5, ACC_PUBLIC + ACC_ABSTRACT + ACC_INTERFACE,
	 "pkg/Comparable", null, "java/lang/Object",
	 new String[] { "pkg/Mesurable" } );
cw.visitField( ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "LESS", "I",
	 null, new Integer( -1 ) ).visitEnd();
cw.visitField( ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "EQUAL", "I",
	 null, new Integer( 0 ) ).visitEnd();
cw.visitField( ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "GREATER", "I",
	 null, new Integer( 1 ) ).visitEnd();
cw.visitMethod( ACC_PUBLIC + ACC_ABSTRACT, "compareTo",
		"(Ljava/lang/Object;)I", null, null ).visitEnd();
cw.visitEnd();
byte[] b = cw.toByteArray();
```

代码的第一行创建了一个ClassWriter实例来构造类的字节码。（构造函数的参数将会在下一个章节解释）

visit方法调用定义了类的头部。V1_5参数是一个常量，如其他的ASM常量一样，在ASM Opcodes接口中。它表明了类的版本，Java 1.5。ACC_XXX 常量是用来定义Java作用域。这里我们定义了类是一个接口，并且作用域是public和abstract的（因为它并不能被初始化）。下一个参数定义了类的名称。字节码中并不包含包名和import内容，所以所有的类名称都要用完整名称。下一个参数对应的是泛型（详情在章节4.1）。在这个例子中是null，原因是这个接口中并没有应用任何泛型。第五个参数是父类。（接口隐式继承Object）。最后一个参数是继承的接口的数组，用内部名称定义。

下面三个visitField调用效果是相同的，定义了三个接口中的变量。第一个参数是Java作用域的标志集合，我们明确的标明了变量的作用域是public，final和static。第二个参数是源码中定义的变量的名称。第三个参数是变量的类型，使用类型描述符表示。这个变量是int类型的所以变量的描述符是I。第四个参数定义了泛型。在我们的例子中是null的原因是因为我们的类型没有使用任何泛型。最后一个参数是变量的常量。这个参数定义的是一个常量，类型是final static，其他的变量有可能是null。因为这个变量没有注解，所以我们调用visitEnd函数去返回FieldVisitor。而不用调用任何visitAnnotation或者visitAttribute方法。（PS：还记得前面的调用顺序吗？）

visitMethod函数调用用来定义compareTo方法。同样的，第一个参数是方法的作用域，第二个参数是方法的名称，第三个参数是方法的方法描述符，第四个参数指明了泛型。这里是null的原因是我们没有在这里使用泛型。最后一个参数是这个函数能够抛出的异常的数组，使用内部名称定义。这里是null的原因是我们没有定义任何能够抛出的异常。visitMethod函数返回了一个MethodVisitor（如图3.4）继而可以被用于继续定义函数的注解和特性，还有最重要的函数内部代码。这里因为没有任何注解并且这个函数是抽象的，所以我们立即调用了visitEnd函数返回MethodVisitor。

最后调用一次visitEnd方法去告诉cw类的定义已经结束。并且调用了toByteArray方法用以取回一个处理好的byte数组。

**使用构造好的类**

先前构造好的byte数组可以被存储在Comparable.class文件中在未来使用。另外也可以使用ClassLoader动态加载他。定义一个ClassLoader的子类，并且定义一个作用域为public的defineClass方法。

```java
class MyClassLoader extends ClassLoader {
    public Class defineClass(String name, byte[] b) {
        return defineClass(name, b, 0, b.length);
    }
}
```

接着可以用它直接初始化类

```java
Class c = myClassLoader.defineClass("pkg.Comparable", b);
```

另一个更加优雅的初始化类的方法是定义一个ClassLoader的子类然后重写findClass方法去初始化需要的类：

```java
class StubClassLoader extends ClassLoader {
	@Override
	protected Class findClass( String name ) throws ClassNotFoundException {
		if ( name.endsWith( "_Stub" ) ) {
			ClassWriter cw = new ClassWriter( 0 );
			...
			byte[] b = cw.toByteArray();
			return(defineClass( name, b, 0, b.length ) );
		}
		return(super.findClass( name ) );
	}
}
```

事实上，使用你构造的类取决于你当前的环境，这不在ASM API的范围之内。如果你正在写一个编译器，那么类的生成过程将由语法树驱动，生成的类将被存储在磁盘上。如果你正在写一个动态代理类或者你想要使用注入的方式，那么请使用ClassLoader。

#### 2.2.4. 修改类

至今我们一直在独立的使用ClassReader和ClassWriter。事件是由我们“用手”生产之后由一个ClassWriter处理的，对称的，使用ClassReader来产生一个事件，然后我们“用手”来消费它，通过一个ClassVisitor的实现。如果我们一起使用这些组件一定会开始变得很有趣。第一步是将ClassReader生成的事件传递给一个ClassWriter。结果是类被ClassReader解析，然后被ClassWriter重建。

```java
byte[] b1 = ...;
ClassWriter cw = new ClassWriter(0);
ClassReader cr = new ClassReader(b1);
cr.accept(cw, 0);
byte[] b2 = cw.toByteArray(); // b2 represents the same class as b1
```

这并不是它自己真正有趣的地方（复制一个byte数组实在是太简单了！），但是等等。下一步介绍使用ClassVisitor在ClassReader和ClassWriter之间：

```java
byte[] b1 = ...;
ClassWriter cw = new ClassWriter(0);
// cv forwards all events to cw
ClassVisitor cv = new ClassVisitor(ASM4, cw) { };
ClassReader cr = new ClassReader(b1);
cr.accept(cv, 0);
byte[] b2 = cw.toByteArray(); // b2 represents the same class as b1
```

与上述代码相对应的架构如图2.6所示，组件使用正方形来表示，事件的传递使用箭头来表示方向。（纵向的线代表先后时间）。

![](/img/2017-8-7/2-6.png)

<center>图2.6：一个修改链</center>

上述的结果并不会被改变，原因是ClassVisitor事件过滤器并没有过滤任何东西。但是现在已经足够去过滤一些事件，通过重写某些方法，去修改一个类。举个例子，看这个ClassVisitor的子类：

```java
public class ChangeVersionAdapter extends ClassVisitor {
    public ChangeVersionAdapter(ClassVisitor cv) {
        super(ASM4, cv);
    }

    @Override
    public void visit(int version, int access, String name,
    String signature, String superName, String[] interfaces) {
        cv.visit(V1_5, access, name, signature, superName, interfaces);
    }
}
```

这个类仅仅充血了ClassVisitor的一个类。调用ClassVisitor其他的方法都不会有任何的改变，除了调用visit方法，她会修改类的编译版本。相应的时序图如图2.7所示。

![](/img/2017-8-7/2-7.png)

<center>图2.7：ChangeVersionAdapter的时序图</center>

通过修改visit方法的其他函数可以实现其他的修改而不仅仅是只修改一个版本号。举个例子，你可以为类添加一个实现的接口。也可以去修改类的名称。但是这并不是简单的仅仅在visit方法中修改name参数。事实上，类的名称有可能在很多地方被编译进了类中，所有的地方都需要被修改。

**优化**

之前的转换只改变了原始类中的四个字节。然而，在这些代码中，b1被完全解析并且从零开始初始化一个b2并不是十分有效率。复制b1要修改的部分给b2才会更有效率。ASM会为方法自动执行这个优化。

- 如果ClassReader组件检测到MethodVisitor返回的ClassVisitor作为参数传递给它的accept方法来自于一个ClassWriter，这意味着这个方法的内容不会被修改，实际上甚至不会被应用程序所看到。
- 这种情况下ClassReader组件不会解析这个方法的内容，不会构造和响应事件，并且仅仅是复制这个方法的byte数组给ClassWriter。

该优化由ClassReader和ClassWriter组件执行，这里是一个例子：

```java
byte[] b1 = ...
ClassReader cr = new ClassReader(b1);
ClassWriter cw = new ClassWriter(cr, 0);
ChangeVersionAdapter ca = new ChangeVersionAdapter(cw);
cr.accept(ca, 0);
byte[] b2 = cw.toByteArray();
```

感谢于这个优化让上面的代码比之前版本的快了两个数量级，因为ChangeVersionAdapter并没有改变任何方法。对于相同类的转换，转换了一些或者所有方法的情况下，加速幅度会更小，但是仍然是有收益的：大概10%-20%。不幸的是这一步优化需要复制所有在原始类中定义的常量到转换的类中。这对于添加方法，变量或者其他的什么并不是一个阻碍，但是相对于不优化的情况下它会导致class文件变得更大。因此推荐仅仅在有效修改的情况下启动优化。

**使用修改后的类**

如上一节所述，修改后的类b2可以被存储在磁盘上或者用ClassLoader加载。但是ClassLoader只能加载之前加载过的类的修改版，如果你想修改所有的类，那么你不得不使用java.lang.instrument包中的ClassFileTransformer（查看文档了解更多，PS：就是使用javaagent来进行插桩，android可以用transform api代替）：

```java
public static void premain(String agentArgs, Instrumentation inst) {
    inst.addTransformer(new ClassFileTransformer() {
        public byte[] transform(ClassLoader l, String name, Class c,
        ProtectionDomain d, byte[] b) throws IllegalClassFormatException {
            ClassReader cr = new ClassReader(b);
            ClassWriter cw = new ClassWriter(cr, 0);
            ClassVisitor cv = new ChangeVersionAdapter(cw);
            cr.accept(cv, 0);
            return cw.toByteArray();
        }
    });
}
```

#### 2.2.5. 移除类中的成员

用之前修改版本号的方法可以很轻松的应用于使用ClassVisitor的其他方法进行修改。例如，修改visitField和visitMethod中的access或者name参数来修改作用域和名称。在类中你不转发对应的方法，即可将这个部分移除。

例如下面这个类的适配器移除了outer和inner类的信息，以及被编译的类的名称（由此产生的类保持完整的功能，因为这些元素仅用于调试目的），这就是通过不调用相应的方法做到的：

```java
public class RemoveDebugAdapter extends ClassVisitor {
    public RemoveDebugAdapter(ClassVisitor cv) {
        super(ASM4, cv);
    }
    @Override
    public void visitSource(String source, String debug) {}
    @Override
    public void visitOuterClass(String owner, String name, String desc) {}
    @Override
    public void visitInnerClass(String name, String outerName,
    String innerName, int access) {}
}
```

这个方法对变量和方法并不适用，因为visitField和visitMethod方法必须返回一个结果。为了移除一个变量或者方法，你必须不转发这个方法的调用，并且返回null。例如下面这个类适配器移除了一个独立的函数，规定通过她的名称和描述。（名称并不能准确的定位一个函数，因为存在函数重载）：

```java
public class RemoveMethodAdapter extends ClassVisitor {
    private String mName;
    private String mDesc;
    public RemoveMethodAdapter(
    ClassVisitor cv, String mName, String mDesc) {
        super(ASM4, cv);
        this.mName = mName;
        this.mDesc = mDesc;
    }
    @Override
    public MethodVisitor visitMethod(int access, String name,
    String desc, String signature, String[] exceptions) {
        if (name.equals(mName) && desc.equals(mDesc)) {
            // do not delegate to next visitor -> this removes the method
            return null;
        }
        return cv.visitMethod(access, name, desc, signature, exceptions);
    }
}
```

#### 添加类的成员

对于转发更少的你接收到的函数调用，你也可以转发更多的函数调用，这样就可以添加函数的结构。新的调用可以被插入到在原函数调用的任意之间的位置，只要被添加进来的成员符合visitXXX方法的调用顺序即可（详情张杰2.2.1）。

例如，如果你想添加一个变量进一个类，你必须在visitField方法调用之间中插入一个新的调用，然后把这个方法写到你的类适配器中。放置的位置注意调用顺序。

如果是在visitEnd函数中添加变量总会添加成功（除非你有明确的限制），因为这个方法总会被调用。如果你放倒visitField或者visitMethod中，多个变量将会被加入到类中：在原类中的每个方法或变量中。这两个方案都可以实现，取决于你的需求。（PS：在ClassVisitor一次完整的扫描中，visitXXX会被调用多次，但是visitEnd就能被调用一次）。

**Note：**事实上，常用的解决方案是在visitEnd中添加新的成言。一个类一定不能含有相同的成员，唯一可以确保这个成员和现存的成员不冲突的方法就是只在visitEnd方法中调用一次去创建。创建的时候使用的名称最好不要像常见的名称，例如可以起个\_counter$或者\_4B7F这样的名字以避免和现有的成员变量冲突。注意的是，在我们第一章讲述的内容中，使用tree API并没有这个限制，因为它可以在整个转换中随时添加新的成员。

为了说明上面的结论，这里给出一个例子：

```java
public class AddFieldAdapter extends ClassVisitor {
    private int fAcc;
    private String fName;
    private String fDesc;
    private boolean isFieldPresent;
    public AddFieldAdapter(ClassVisitor cv, int fAcc, String fName,
    String fDesc) {
        super(ASM4, cv);
        this.fAcc = fAcc;
        this.fName = fName;
        this.fDesc = fDesc;
    }
    @Override
    public FieldVisitor visitField(int access, String name, String desc,
    String signature, Object value) {
        if (name.equals(fName)) {
            isFieldPresent = true;
        }
        return cv.visitField(access, name, desc, signature, value);
    }
    @Override
    public void visitEnd() {
        if (!isFieldPresent) {
            FieldVisitor fv = cv.visitField(fAcc, fName, fDesc, null, null);
            if (fv != null) {
                fv.visitEnd();
            }
        }
        cv.visitEnd();
    }
}
```

变量在visitEnd方法中被添加了进来。visitField方法并没有被重写去修改已存在的变量活着移除一个变量，这里仅仅决定了是否我们要加入下面的变量。注意一个问题，我们这里有一部fv != null 的判断，是因为在上一节我们提到过，visitField是有可能返回null的。

#### 2.2.7. 转换链 

之前我们看了简单的转换链实现，使用一个ClassReader，一个Adapter，一个ClasWriter。想当然，我们可以使用更加复杂的转换链。使用多了类Adapter一起工作。多个Adapter可以让你编排独立的类转换器去组织复杂的转换链。注意，一个转换链并不一定是线性的。一可以写一个ClassVisitor去转发所有的函数，也可以使用多个ClassVisitor在同一时间接受调用：

```java
public class MultiClassAdapter extends ClassVisitor {
    protected ClassVisitor[] cvs;
    public MultiClassAdapter(ClassVisitor[] cvs) {
        super(ASM4);
        this.cvs = cvs;
    }
    @Override 
    public void visit(int versiaon, int access, String name,
    String signature, String superName, String[] interfaces) {
        for (ClassVisitor cv: cvs) {
            cv.visit(version, access, name, signature, superName, interfaces);
        }
    }...
}
```

对称的多个类适配器可以委托给相同的ClassVisitor（这需要一些措施去确保正确，比如visit和visitEnd函数都只会在这个ClassVisitor中被调用一次）。因此一个很棒的转换链将会在图2.8中展示。

#### 工具类


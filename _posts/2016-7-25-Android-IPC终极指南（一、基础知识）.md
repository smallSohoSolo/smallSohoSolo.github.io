---
layout: post
title: "Android IPC终极指南（一、基础知识）"
date: 2016-7-25
category: Android
tag: IPC
---

> 此篇文章是看完的Android开发艺术的个人总结，摘抄了很多书上的语句，特此说明

Android是基于Linux的操作系统，他有自己的进程间通信方式，Android通过Binder可以轻松的实现进程间通信，当然Android也可以通过Socket进行通信。

### 为什么要进程间通信？

- 可能应用因为某些原因需要进行进程间通信，例如：某些模块需要运行在独立的进程中，例如某些工具App的系统清理
- 可能某些操作对内存的要求比较高，独立app的内存不足以满足，Android对每个app的内存是有限制的多进程可以扩大内存限制
- ContentProvider就是跨进程通信，只是内部封装了我们表层看不到

### 强力开启多进程！

```xml
<application
    android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:theme="@style/AppTheme">
    <activity android:name=".MainActivity">
       <intent-filter>
          <action android:name="android.intent.action.MAIN" />
          <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>
    <activity android:name=".SecondActivity"
        android:process=":remote"/>
    <activity android:name=".ThirdActivity"
        android:process="com.smallSohoSolo.remote"/>
</application>
```

在Activity中写下android:process标志即可为activity开启多进程
大家会发现android:process中SecondActivity和ThirdActivity应用了两种方式去进行创建多进程，这两种方式是有区别的

- ":remote"所创建的多进程是App私有进程，外界无法访问
- "com.smallSohoSolo.remote"这种直接命名的方式创建的是全局进程，可以通过ShareUID的方式可以和他跑在同一个进程中

### ShareUID是什么鬼？

Android会为每个App分配一个唯一的UID，具有相同的UID的app才能共享数据，但是这个UID并不是随便设置的，两个应用通过ShareUID跑在同一个进程中是有要求的，需要两个应用有相同的ShareUID还得签名相同才可以

![](/img/2016-7-25/1.png)

如果他们满足这写条件，那么他们可以互相访问data数据，组件信息等私有数据，当然了两个具有相同ShareUID的应用可以跑在同一个进程中（并不是必须，只是可以），那就什么都可以共享了，包括内存中的数据访问，可以理解为是一个应用中两个部分了。

### 番外篇，聊聊ShareUID

详情咨询：[http://www.cnblogs.com/mythou/p/3258715.html](http://www.cnblogs.com/mythou/p/3258715.html)

![](/img/2016-7-25/2.png)

### 呵呵的多进程~

你以为开了多进程，弄个process就完事大吉了，并没有 0.0
应为开启了多进程，那么他们会被分配给不同的虚拟机副本，举个栗子：
你在A中修改个number = 2（原来等于1），你再B（开启了多进程的另一个Activity）中打印一下，你会发现，我勒个去，还是1！！！

### Why？

记住，多进程是不共享内存的，也就是平常的那些用内存进行共享数据的操作都很快乐的跟你说再见了，666，一般来说，多进程会造成如下几种问题：

- 静态成员和单例模式完全失效
- 线程同步机制完全失效
- SharePreferences可靠性下降（底层操纵XML去进行操作，多进程并发写会造成数据丢失）
- Application会被多次创建

### PM都不怕还怕多进程？

虽然不能直接的共享内存，但是还有很多途径可以共享资源：

- Intent
- 文件
- SharedPreference（低可靠性不代表不能用）
- 基于Binder的Messenger和AIDL
- Socket

### 要想会（fu），先基（xiu）础（lu）

- Serializable
- Parcelable
- Binder

### 来，咱们谈谈序列化

IPC中肯定是要用到序列化的，Android中存在两种序列化，其一是Serializable，这是Java提供的序列化方案，其二是Parcelable，这是Android自带的序列化方案，推荐使用第二种，毕竟亲儿子。233

### Serializable

他的实现很简单

```java
public class User implements Serializable
```

只要轻轻的继承一下就好
然后我们需要给他指定一个serialVersionUID，

```java
private static final long serialVersionUID = XXXXXXXXXXL
```

这个serialVersionUID最好用当前类的hash值，当然自行指定也可以。

### 如何序列化

```java
//序列化
User user = new User();
ObjectOutputStream out = new ObjectOutputStream(
	new FileOutputStream("cache.txt");
);
out.writeObject(user);
out.close();
//反序列化
ObjectInputStream in = new ObjectInputStream(
	new FileInputStream("cache.txt");
);
User newUser = (User)in.readObject();
in.close();
```

这样就能将一个类序列化来序列化去了 0.0 
好厉害有木有！~！~！
**高能预警：虽然两个类表层是一样的，但是这两个对象并不是一个（内存不同）**

### 谈谈serialVersionUID

动手能力强的同学发现，如果我们不添加这个值，序列化仍然成功了有木有！难道博主骗我们= =
其实并不是的，其实这个UID是用来辅助序列化和反序列化的，原则上序列化后的UID只有和当前类的UID相同才能被正常的序列化和反序列化

- 如果不手动填写serialVersionUID，序列化的时候系统会自动算出一个serialVersionUID写到文件中，然后恢复的时候会再次计算，如果两者的serialVersionUID一致，也就是类没有被修改，那么不会影响反序列化，但是，一旦修改了类，那么计算出来的hash就不同了，反序列化的时候就会报错。程序会crash
- 如果手动填写，就算修改了类，只要不是毁灭性的修改（例如修改类名），那么反序列化时会尽最大的限度去进行反序列化，最大限度还原对象，而不是直接crash

### 序列化的注意事项

- 静态成员变量属于类不属于对象，所以不会参与到序列化的过程中
- 用transient关键字标记的成员变量不参与序列化过程

### 控制序列化过程TvT

我们只要在要序列化的类中实现下面两个方法即可重写系统默认的序列化和反序列化过程

```java
private void writeObject(java.io.ObjectOutputStream out) throws IOException {
  //此处写强大的魔法咒语biu biu biu~
}
private void readObject(java.io.ObjectInputStream in) throws IOException,ClassNotFoundException {
  //此处写强大的黑魔法 0.0
}
```

### 主角来了，Parcelable

继承Parcelable接口之后，就可以序列化对象了，Android Studio还能帮我们自动生成下面这一大堆 TvT
下面是个User序列化的栗子：

```java
public class User implements Parcelable {

    private String userName;
    private int userId;
    private boolean isMale;
    private Book book;

    public User(String userName, int userId, boolean isMale, Book book) {
        this.userName = userName;
        this.userId = userId;
        this.isMale = isMale;
        this.book = book;
    }

    protected User(Parcel in) {
        userName = in.readString();
        userId = in.readInt();
        isMale = in.readByte() != 0;
        book = in.readParcelable(Book.class.getClassLoader());
    }

    public static final Creator<User> CREATOR = new Creator<User>() {
        @Override
        public User createFromParcel(Parcel in) {
            return new User(in);
        }

        @Override
        public User[] newArray(int size) {
            return new User[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel parcel, int i) {
        parcel.writeString(userName);
        parcel.writeInt(userId);
        parcel.writeByte((byte) (isMale ? 1 : 0));
        parcel.writeParcelable(book, i);
    }
}
```

其中Book也是一个继承了Parcelable的类，他同样可以被序列化。describeContents在通常情况下返回的是0，仅当当前对象中存在文件描述符中他才会返回1，别的方法大家自行脑补 0.0
我就不告诉你~~

### How to choose?

Serializable是Java中的序列化接口，他在序列化和反序列化中都需要大量的IO操作，虽然使用简单，但是开销很大，而Parcelable是Android推荐的方式，效率很高，Parcelable主要用在内存的序列化上，如果需要序列化到文件的话，Parcelable也是可以的，但是过程会很复杂，所以这时建议大家使用Serializable来进行序列化。


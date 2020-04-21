---
title: Android反射和动态代理浅析
date: 2020-04-21 21:14:45
tags: [Android][java]
---

# 提纲

- java反射基础
- 反射在Android中的应用
- Java动态代理
- 动态代理在Android的应用

## java反射基础

### 相关定义和简单调用

![img](https://upload-images.jianshu.io/upload_images/5714046-a489726f6cae09e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

[java反射官方说明链接](https://docs.oracle.com/javase/9/docs/api/java/lang/reflect/package-summary.html)

Java允许程序在运行时透过Reflection APIs加载一个运行时才得知名称的class，获得其完整结构，包括其modifiers（诸如public, static 等）、superclass（例如Object）、实现之interfaces（例如Cloneable），也包括fields和methods的所有信息，并可于运行时改变fields内容或唤起methods。

在程序运行时，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性。这种 动态的获取信息 以及 动态调用对象的方法 的功能称为 java 的反射机制

![img](https://upload-images.jianshu.io/upload_images/5714046-be2720e3b7e993e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

获取类，类的对象，类具有的成员变量和方法，调用方法，调用构造器创建新类对象，xxx替换为以下方式，可以获得相应目标构造方法，属性

Constructor：代表类的单个构造方法，通过Constructor我们可执行一个类的某个构造方法（有参或者无参）来创建对象时。     

Method：代表类中的单个方法，可以用于执行类的某个普通方法，有参或无参，并可以接收返回值。    

Field：代表类中的单个属性，用于set或get属性 

### Java通过反射访问私有属性

![img](https://upload-images.jianshu.io/upload_images/5714046-4d3d861462ba3716.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](https://upload-images.jianshu.io/upload_images/5714046-91109f82af411f5d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在java中，类的构造方法，方法，属性，都继承于AccessibleObject，AccessibleObject提供了构造方法，普通方法，和属性的访问控制的能力

setAccessible(booleanflag)方法，将此对象的accessible 标志设置为指示的布尔值。值为true 则指示反射的对象在使用时应该取消Java 语言访问检查。值为false 则指示反射的对象应该实施Java 语言访问检查。

setAccessible是启用和禁用访问安全检查的开关,并不是为true就能访问为false就不能访问。通过设置setAccessible（true），java可以调用类的私有属性和方法

### Java通过反射修改final修饰的常量值

![img](https://upload-images.jianshu.io/upload_images/5714046-f502173dcfe4bf3f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

常量是指使用 final 修饰符修饰的成员属性，与变量的区别就在于有无 final 关键字修饰。

而java中会有个Modifier接口对访问修饰符进行解码

Java 虚拟机（JVM）在编译 .java 文件得到 .class 文件时，会优化我们的代码以提升效率。其中一个优化就是：**JVM 在编译阶段会把引用常量的代码替换成具体的常量值**

对于 int 、long 、boolean 以及 String 这些基本类型

JVM 会优化，而对于Integer 、Long 、Boolean 这种包装类型，或者其他诸如Date 、Object 类型则不会被优化。

**总结来说：对于基本类型的静态常量，JVM 在编译阶段会把引用此常量的代码替换成具体的常量值。**

我们在程序运行时刻依然可以使用反射修改常量的值（后面会代码验证），但是JVM 在编译阶段得到的.class 文件已经将常量优化为具体的值，在运行阶段就直接使用具体的值了，所以即使修改了常量的值也已经毫无意义了。

**避免在编译时刻被优化**，这样我们通过反射修改常量之后才有意义，指向一个可变对象，或经过计算（如三目运算符）赋值常量（这些都是运行时进行赋值的，所以有效）

### Java反射效率相比直接调用低 

![img](https://upload-images.jianshu.io/upload_images/5714046-81dd312f525f9f4c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

java反射是要动态的获取实例调用方法，将内存中的对象进行解析，消耗资源去查找和计算，涉及到与底层c语言的交互，反射的过程中有递归和遍历的操作，这些都需要时间

## 反射在Android中的应用 

### 反射在Android源码中的应用 

反射在安卓用的使用相当广泛，源码中也有通过反射创建实例的过程，如Activity的启动过程中Activity的对象的创建

![img](https://upload-images.jianshu.io/upload_images/5714046-4c3ecaf79e87787e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 用于调用SDK中@hide标注的隐藏方法

![img](https://upload-images.jianshu.io/upload_images/5714046-6ae6e48ff6631dbf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Hook源码，添加自己的处理

通过替换源码中的具体实现类为代理类，添加相关操作，达成部分特殊功能

Example:监听activity跳转信息，跳转至未在AndroidManifest注册的activity

### 反射在部分安卓框架中的应用 

**如EventBus 事件总线**

- Register：通过反射的方式获取订阅类中有Subscribe注解的方法，将方法信息，参数类，订阅类及实例通过map保存起来
- Post:根据参数类，获取map保存的订阅实例和方法，一一发到分配器
- Dispatch:根据注解Subscribe中的信息，分配至不同的处理器poster。分配至不同线程
- Run：指定线程，调用invokeSubscriber的方法，调用实例的订阅方法，就是反射调用方法invoke。

也由于EventBus对反射的调用过多，容易导致性能下降问题，EventBus也对此作出了优化

![img](https://upload-images.jianshu.io/upload_images/5714046-044ad214a82bca60.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

编译之后，索引文件在build/generated/source/apt/debug/com/XXX/packge/EventBusIndex.java。

**如Butterknife （反射+annotationProcessor）**

**基于反射和注解解析的IOC框架
**

原理：通过annotationProcessor处理注解，反射得到各个需要绑定的成员和类型，生成Target_ViewBinding类，再通过ButterKnife.bind方法绑定实例。

当然其中还涉及到一个更有趣的技术，编译时注解，后续总结下再分享

大概原理就是声明的注解的生命周期为CLASS,然后继承AbstractProcessor类。继承这个类后，在编译的时候，编译器会扫描所有带有你要处理的注解的类，然后再调用AbstractProcessor的process方法，对注解进行处理，那么我们就可以在处理的时候，动态生成绑定事件或者控件的java代码，然后在运行的时候，直接调用方法完成绑定

参考

ButterKnife解读 https://www.jianshu.com/p/2967ff971177

ButterKnife源码解析https://blog.csdn.net/silenceoo/article/details/77897718

## JDK动态代理

#### 基础定义

- 静态代理：由程序员创建或特定工具自动生成源代码，再对其编译。在程序运行前，代理类的.class文件就已经存在了。
- 动态代理：在程序运行时，运用反射机制动态创建而成。

动态代理类的字节码在程序运行时由Java反射机制动态生成，无需手工编写源代码。

![img](https://upload-images.jianshu.io/upload_images/5714046-d1a5453c5876668d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 相关的类或接口

- java.lang.reflect.Proxy：这是 Java 动态代理机制的主类，它提供了一组静态方法来为一组接口动态地生成代理类及其对象
- [https://](https://docs.oracle.com/javase/9/docs/api/java/lang/reflect/Proxy.html)[docs.oracle.com/javase/9/docs/api/java/lang/reflect/Proxy.html](https://docs.oracle.com/javase/9/docs/api/java/lang/reflect/Proxy.html)
- java.lang.reflect.InvocationHandler：这是调用处理器接口，它自定义了一个invoke方法，用于几种处理在动态代理类对象上的方法调用。通常在该方法中实现对委托类的代理访问。
- java.lang.ClassLoader：Proxy静态方法生成动态代理类同样需要通过类装载器来进行装载才能使用，它与普通类的唯一区别就是其字节码是由JVM 在运行时动态生成的而非预存在于任何一个.class 文件中。

三种动态代理的简单demo

https://github.com/Steven37/three-proxy/blob/48d5d3941f821c90d64a68c00b9a43b6665e5480/src/main/java/jdk/staticproxy/CountProxy.java

#### 动态代理的四个步骤

![img](https://upload-images.jianshu.io/upload_images/5714046-c03516e559b9f4a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](https://upload-images.jianshu.io/upload_images/5714046-4509cc6dd84574ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 通过实现 InvocationHandler 接口创建自己的调用处理器；
- 通过为 Proxy 类指定 ClassLoader 对象和一组 interface 来创建动态代理类
- 通过反射机制获得动态代理类的构造函数，其唯一参数类型是调用处理器接口类型
- 通过构造函数创建动态代理类实例，构造时调用处理器对象作为参数被传入

![img](https://upload-images.jianshu.io/upload_images/5714046-ca0017ff3a044f99.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 动态生成的代理类本身的一些特点 

1.包：如果所代理的接口都是public 的，那么它将被定义在顶层包（即包路径为空），如果所代理的接口中有非public 的接口（因为接口不能被定义为protect或private，所以除 public之外就是默认的package访问级别，那么它将被定义在该接口所在包，这样设计的目的是为了最大程度的保证动态代理类不会因为包管理的问题而无法被成功定义并访问；

2.类修饰符：该代理类具有final 和 public 修饰符，意味着它可以被所有的类访问，但是不能被再度继承；

3.类名：格式是“$ProxyN”，其中 N 是一个逐一递增的阿拉伯数字，代表Proxy 类第 N 次生成的动态代理类，值得注意的一点是，并不是每次调用Proxy 的静态方法创建动态代理类都会使得N 值增加，原因是如果对同一组接口（包括接口排列的顺序相同）试图重复创建动态代理类，它会很聪明地返回先前已经创建好的代理类的类对象，而不会再尝试去创建一个全新的代理类，这样可以节省不必要的代码重复生成，提高了代理类的创建效率。

\4. 类继承关系：Proxy 类是它的父类，这个规则适用于所有由

Proxy 创建的动态代理类。而且该类还实现了其所代理的一组接口;

但Proxy只能对interface进行代理，无法实现对class的动态代理。

## JDK动态代理的应用

### hook源码，动态代理实现 ServiceHook

![img](https://upload-images.jianshu.io/upload_images/5714046-d9613a992132989e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在Android中，Binder是 ServiceManager 连接各种Manager (ActivityManager，WindowManager，等等)和相应 ManagerService 的桥梁；从Android 应用层来说，Binder 是客户端和服务端进行通信的媒介，当

bindService 的时候，服务端会返回一个包含了服务端业务调用的

IBinder 对象

这四者的关系类似于网络访问

- Binder Client 只需要知道自己要使用的 Binder 的名字及其在ServiceManager 中的引用即可获取该Binder 的引用，得到引用后就可以像普通方法调用一样调用Binder 实体的方法;
- Binder Server 在生成一个 IBinder 实体时会为其绑定一个名称并传递给Binder Driver，Binder Driver 会在内核空间中创建相应的Binder 实体节点和节点引用，并将引用传递给ServiceManager。
- ServiceManager 会将该 Binder 的名字和引用插入一张数据表中，这样Binder Client 就能够获取该Binder 实体的引用，并调用上面的方法;ServiceManager 相当于 DNS服务器，负责映射Binder 名称及其引用,其本质同样是一个标准的Binder Server
- Binder Driver 则相当于一个路由器。

先通过 ServiceManager.getService 方法获取一个IBinder 对象，但是这个IBinder 对象不能直接调用，必须要通过asInterface 方法转成对应的IInterface 对象才可以使用，如果在同一个进程中（当然，这是跨进程通信，这种情况很少），就会直接返回通过attachInterface 方法设置的IInterface 对象（上面代码所述），但是如果不是在同一个进程中，就会先通过IBinder 对象创建一个Proxy 对象，然后在Proxy 对象中通过调用IBinder 对象的 transact 方法调用到主进程的Service 中，实现了跨进程的通信。

![img](https://upload-images.jianshu.io/upload_images/5714046-783693a4220fff7e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

常见的，我们可以通过动态代理替换的方式，对android的复制操作的文本，先理解ClipBoardManager的逻辑如上图。

应用在使用剪切板服务，通过getSyetemService方法获取到的对象就是ClipboardManager,但是内容是先通过ServiceManager获取到ClipBoardManager的远程Ibinder对象， 然后通过Iclipboard$Stub方法将binder转化为本地对象，就是iClipboard$proxy对象

接下来开始我们的替换操作，具体流程如下

![img](https://upload-images.jianshu.io/upload_images/5714046-0a1f1f5191895f32.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由于我们动态代理的是本地进程ServiceManager缓存中的Ibinder对象，应用进程的 ServiceManager 其实也就相当于一个 Binder Client，sCache 里面获取不到对应的 Service，它就会自动通过 Binder Driver 和 Binder Server （主进程的 ServiceManager）通信去获取对应的 Service，所以修改 Binder Client，其实是不会影响 Binder Server的操作，所以不会影响别的应用app。

### JDK动态代理在框架中的应用

#### 完成插件化逻辑（360Android 插件项目 DroidPlugin）

https://github.com/Qihoo360/DroidPlugin

360完成的框架DroidPlugin就是基于动态代理hook android系统service的原理

进行了深度的hook，ams，pms，service，brocastReceiver等

例如 Hook掉ActivityManagerNative对于startActivity方法的调用，将真的intent藏起来

Hook掉ActivityThread的handler，修改handleLaunchActivity

DroidPlugin开源插件研究资料整理 https://blog.csdn.net/hmmhhmmhmhhm/article/details/74352139

#### Retrofit网络加载框架  

Retrofit流程如图

![img](https://upload-images.jianshu.io/upload_images/5714046-a8c5f332ba2d79dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Retrofit的实现实际就是动态代理的方式

Retrofit create(finalClass<T> service) 动态代理

Retrofit loadServiceMethod 加载方法统一处理，将retrofit上的注解修改为request的内容

ServiceMethod toCall 完成从ServiceMethod到call的转化转化

createCallAdapter 根据返回类型和注解返回对应adapter

![img](https://upload-images.jianshu.io/upload_images/5714046-cc6725d904f0269a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

简单看看源码，可以很清晰的看到，这是动态代理的实现，当然Retrofit确实是很优秀的框架，不只是动态代理的使用，其中涉及很多良好的设计模式。

![img](https://upload-images.jianshu.io/upload_images/5714046-ea5f064801b7fe76.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 参考链接

[Retrofit2源码分析](https://www.jianshu.com/p/0102af578875)

[Retrofit解析9之流程解析](https://www.jianshu.com/p/855fce462242)

反射详解 https://juejin.im/post/598ea9116fb9a03c335a99a4

EventBus 3.0 最佳实践和原理浅析 https://xiazdong.github.io/2017/08/02/eventbus3/

ProxyGenerator 真正生成字节码的实现类 http://www.docjar.com/html/api/sun/misc/ProxyGenerator.java.html

方法说明及解析 https://www.cnblogs.com/liuyun1995/p/8144706.html
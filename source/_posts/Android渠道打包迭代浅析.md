---
title: "Android渠道打包迭代浅析"
date: 2020-04-16 23:27:34
tags: [Android]
---

<meta name="referrer" content="no-referrer"/>

# 提纲

**渠道打包方案：**v1下的渠道包方案迭代，v2渠道包，常见工具库

**原理深入：**部分源码

## 前言

要进行渠道打包，重点是设置渠道标识，而设置渠道标识，涉及apk打包和签名，可以先阅读[《Android 签名机制及迭代浅析》](https://whytry-gar.github.io/2020/04/14/Android%20%E7%AD%BE%E5%90%8D%E6%9C%BA%E5%88%B6%E5%8F%8A%E8%BF%AD%E4%BB%A3%E6%B5%85%E6%9E%90/#more)，了解相关知识再阅读本文。

本文主要一个个说明渠道包打包的不同做法优劣和迭代，如果想要直接获得比较好的渠道打包方式，可以直接看文章后半部分

## V1签名机制下打渠道包 

由V1签名和校验机制可知，修改APK中的任何文件都会导致安装失败！那怎么添加渠道信息呢？

### 方案1：Android Gradle Plugin 

**原理：**Gradle Plugin本身提供了多渠道的打包策略，Gradle编译生成多渠道包时，会用不同的渠道信息替换AndroidManifest.xml中的占位符。

1.在AndroidManifest.xml中添加渠道信息占位符：

![img](https://upload-images.jianshu.io/upload_images/5714046-d50bab6d22fa2cdd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2.通过Gradle Plugin提供的productFlavors标签，添加渠道信息

![img](https://upload-images.jianshu.io/upload_images/5714046-cdad750eb6eb88ed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**签名包流程：**静态资源整理->编译源码，处理得到dex文件->合成APK->签名，对齐

**缺点：**每生成一个渠道包，都要重新执行一遍构建流程，效率太低，只适用于渠道较少的场景

**渠道包耗时 = 渠道数量 \* （编译源码+添加渠道+签名）**

**Q：能否避免解决多次编译？**

### 方案2：找个文件添加上信息（res/raw。assets/） ，要用再读出来 

**原理：**APK是自签名：没有第三方权威机构认证，用户可以自行生成keystore，Android签名方案无法保证APK不被二次签名。我们可以重新打包app，再设置信息后签名。

了解apk编译的话，可以知道以下目录不会被编译二进制！res/raw 和 assets/

![img](https://upload-images.jianshu.io/upload_images/5714046-d6e611e542ef1b8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以我们可以从这方面下手，设置了信息之后，重签名，app内再读出来。有两种做法（apktool和winrar类压缩工具）

![img](https://upload-images.jianshu.io/upload_images/5714046-2a1f21df9920a625.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**优点：**避免了多次编译，渠道包时长 = 编译源码+ 渠道数量 * （添加渠道 + 签名）

**缺点：**1.项目逐渐膨胀 解压-合成-重签依然耗时

​      2.签名不便于管理，易泄露，大项目签名保密

​      3.不稳定，可能升级了 Gradle Plugin 的版本之后，会导致解包失败

**Q：能否避免解决多次签名？**

**Ps：高效的多渠道打包的几个关键：不能破坏签名，不能重新打包，读取信息必须高效
**

**渠道包时长 = 编译源码+签名+ 渠道数量 \* 添加渠道**

不破坏签名就限制了不能解包以及重新签名，势必对效率有所提高。

### 高效方案一：到/META-INF 目录下，写入一个空文件，以文件名来标识渠道号（来自美团） 

**原理:**了解V1签名，发现V1签名只会校验元文件（/META-INF 目录）以外的数据，所以可以从这个漏洞下手，在/META-INF 目录下写入空文件，文件名为渠道名，就可以高效达到渠道包目的

**步骤：**

1.Java Zip API解析APKZipFile api apk；

2.添加渠道信息META-INF添加空文件，如META-INF/xiaomi.channel

3.获取渠道信息，遍历META-INF/目录，匹配xx.channel获取渠道信息

使用这样的方案，并不会破坏 v1 签名，所以效率会很高。

### 高效方案二：在注释区加渠道信息(腾讯的VasDolly)

**原理:**apk文件本质为zip包，检验只校验数据区，通过修改 Apk 文件的 EOCD 部分，增加渠道的信息

![img](https://upload-images.jianshu.io/upload_images/5714046-4f444f419aaa0afc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## V2签名机制下打渠道包 

Q:上述方式都改了apk文件，对V2不管用，需要寻求快速的V2下渠道包方案

A:在APK签名块中添加一个ID-Value，存储渠道信息

**原理：**V2签名区块存在盲区：未知id-value不校验（当前线上推广的框架都是这个做法，腾讯的VasDolly，美团的Walle都是）

Android系统只会关注ID为0x7109871a的V2签名块，并且忽略其他的ID-Value，同时V2签名只会保护APK本身，不包含签名块

**方案步骤：**

1.找到APK的EOCD块

2.找到APK签名块

3.获取已有的ID-Value Pair

4.添加包含渠道信息的ID-Value

5.基于所有的ID-Value生成新的签名块

6.修改EOCD的中央目录的偏移量（修改EOCD的中央目录偏移量，不会导致数据摘要校验失败）

7.用新的签名块替代旧的签名块，生成带有渠道信息的APK

![img](https://upload-images.jianshu.io/upload_images/5714046-48de6cec91ab85dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

实际上，除了渠道信息，我们可以在APK签名块中添加任何辅助信息。

V3机制下的渠道打包，和V2一样就可以

## 常见的渠道打包工具和对比 



![img](https://upload-images.jianshu.io/upload_images/5714046-61fbdab46c39dab2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ØVasDolly使用简介

   命令行执行jar包（支持多线程）

   java –jar VasDolly.jar

   编译、添加渠道一条龙

   gradlew channelAppDebug

   基于已编译APK添加渠道

   gradlew rebuildChannel

相关框架连接

【美团】Walle：https://github.com/Meituan-Dianping/walle

【腾讯电竞】VasDolly:https://github.com/Tencent/VasDolly

【某大神】packer-ng-plugin：https://github.com/mcxiaoke/packer-ng-plugin

## 原理深入——源码篇

展示关键代码，相关讲解在注释中

### 写入渠道（[V1SchemeUtil](https://www.javatips.net/api/ApkChannelPackage-master/common/src/main/java/com/leon/channel/common/V1SchemeUtil.java)）

![img](https://upload-images.jianshu.io/upload_images/5714046-e236345f0b46462e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 读取渠道（[V1SchemeUtil](https://www.javatips.net/api/ApkChannelPackage-master/common/src/main/java/com/leon/channel/common/V1SchemeUtil.java)）

![img](https://upload-images.jianshu.io/upload_images/5714046-e0adbdfa9f180070.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ØV2获取渠道信息（V2SchemeUtil）

![img](https://upload-images.jianshu.io/upload_images/5714046-a32751ab27d0d179.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### V2添加渠道信息（VasDolly ——[ChannelWriter](https://github.com/Tencent/VasDolly/tree/master/writer)）

![img](https://upload-images.jianshu.io/upload_images/5714046-123acf4f2ff2f917.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### V2添加渠道信息（VasDolly —— IdValueWriter ）

![img](https://upload-images.jianshu.io/upload_images/5714046-92279900e12191bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![img](https://upload-images.jianshu.io/upload_images/5714046-29529271d1d5d01d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### V2获取渠道信息（VasDolly ——IdValueReader）

![img](https://upload-images.jianshu.io/upload_images/5714046-df2338f1d913098c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 参考链接

•Android多渠道打包实践（VasDolly&AndResGuard） https://www.jianshu.com/p/38ad00f73e94
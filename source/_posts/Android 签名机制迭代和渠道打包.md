---
title: Android 签名机制迭代和渠道打包（更新ing）
date: 2020-04-14 22:35:52
tags: [Android]
---

<meta name="referrer" content="no-referrer"/>

# 提纲

**初步了解**：是什么，为什么需要，有什么好处，怎么加密和验证，怎么使用，要点补充

**签名方案：**v1,v2,v3签名及校验流程，版本缺陷

**渠道打包方案：**v1下的渠道包方案迭代，v2渠道包，常见工具库，cc的使用

**原理深入**：部分源码

## 初步了解Android签名机制

###   Android应用程序的签名（是什么）

​     \- 在Android 系统中，所有安装到系统的应用程序都必有一个数字证书，要求每一个安装进系统的应用程序都是经过数字证书签名的，此数字证书用于标识应用程序的作者和在应用程序之间建立信任关系

###    签名的必要性（为什么需要）

   -**证明 APK的所有者。**通过对 Apk 进行签名，开发者可以证明对 Apk 的所有权和控制权，可用于安装和更新其应用。而在 Android 设备上的安装 Apk ，如果是一个没有被签名的 Apk，则会被拒绝安装。

   -**允许 Android市场和设备校验APK的正确性。**在安装 Apk 的时候，软件包管理器也会验证 Apk 是否已经被正确签名，并且通过签名证书和数据摘要验证是否合法没有被篡改。只有确认安全无篡改的情况下，才允许安装在设备上。

###    统一签名的好处（有什么好处）

   \- **有利于程序升级。**当新版程序和旧版程序的数字证书相同时，Android 系统才会认为这两个程序是同一个程序的不同版本。如果新版程序和旧版程序的数字证书不相同，则Android 系统认为他们是不同的程序，并产生冲突，会要求新程序更改包名。

   -**有利于程序的模块化设计和开发。**Android 系统允许拥有同一个数字签名的程序运行在一个进程中，Android程序会将他们视为同一个程序。所以开发者可以将自己的程序分模块开发，而用户只需要在需要的时候下载适当的模块

   -**可以通过权限(permission)的方式在多个程序间共享数据和代码。**Android提供了基于数字证书的权限赋予机制，应用程序可以和其他的程序共享概功能或者数据给那那些与自己拥有相同数字证书的程序。如果某个权限(permission)的protectionLevel是signature，则这个权限就只能授予那些跟该权限所在的包拥有同一个数字证书的程序。

###    签名的验证方式（怎么加密和验证的）

​     发送者把文件用算法生成摘要，再用私钥对摘要进行加密，把证书、文件、加密串、公钥发给接收端。接收端获取到之后，根据证书认真发送者身份，用公钥对加密串进行解密，再对文件也用算法生成摘要，对比解密之后的信息和摘要的信息是否相同。

![img](https://upload-images.jianshu.io/upload_images/5714046-9724f02cb3bc9e25.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

签名的认证方式

签名的动作是执行在打包流程的后期



![img](https://upload-images.jianshu.io/upload_images/5714046-f719a288b8215f5f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

静态资源整理->编译源码，处理得到dex文件->合成APK->签名，对齐

###    对Android应用程序签名（怎么使用的）

使用方面只需要两步

第一步是生成密钥，在as中就有方便的页面生成密钥

第二步则是使用密钥签名，常见的在gradle中设置好signingconfig，不同编译下的使用

![img](https://upload-images.jianshu.io/upload_images/5714046-53f5082beb60948b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

生成密钥



![img](https://upload-images.jianshu.io/upload_images/5714046-d3c0f3f98a45af22.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

使用密钥


这里举例的使用方式是比较常用有效的，也有命令行，gui界面的方式进行设置。在ppt注释中有链接，有兴趣可以看看这些连接

Android APK命令行实现V1、V2签名及验证https://www.jianshu.com/p/e00f9bb12340

Android 开发者->Android Studio->用户指南->为您的应用签名https://developer.android.com/studio/publish/app-signing

###    关于数字证书的要点补充

​    **-所有的应用程序都必须有数字证书（包括未发布的应用）**。Android 系统不会安装一个没有数字证书的应用程序。

​     从IDE 中运行或调试您的项目时，Android Studio 将自动使用通过 Android SDK 工具生成的调试证书为您的应用签名。当您首次在 Android Studio 中运行或调试项目时，IDE 会自动在 `$HOME/.android/debug.keystore` 中创建调试密钥库和证书，并设置密钥库和密钥密码。

​    **-在签名时，需要考虑数字证书的有效期。**数字证书都是有有效期的，Android 只是在应用程序安装的时候才会检查证书的有效期。如果程序已经安装在系统中，即使证书过期也不会影响程序的正常功能。一旦数字证书失效，持有改数字证书的程序将不能正常升级。

## Android中的签名方案

Android 的签名方案，发展到现在，不是一蹴而就的。Android

现在已经有三种应用签名方案，v1升级到v3

v1 到 v2 是颠覆性的，为了解决JAR签名方案的安全性问题，而到了v3方案，其实结构上并没有太大的调整，可以理解为v2签名方案的升级版，有一些资料也把它称之为v2+方案。

签名方案的升级，就是向下兼容的，所以只要使用得当，这个过程对开发者是透明的。

v1 到 v2 方案的升级，对开发者影响最大的，就是渠道打包签署的问题。这个后续会说到。

![img](https://upload-images.jianshu.io/upload_images/5714046-5494251bf97459b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


签名工具有两个jarsigner 和 apksigner。它们的签名算法没什么区别，主要是签名使用的文件不同。

V1(<7.0)基于jar签名，jarsigner

V2（>=7.0）基于apk签名，apksigner

V3(>=9.0)基于apk签名，apksigner，支持密钥轮转

Android提供了两种对Apk的签名方式，区别：

1、jarsigner是为jar签名设计的，apksigner是为apk设计的；

2、jarsigner不能生成v2签名，apksigner可以；

3、jarsigner没有判断4.2（api17）以下，不能用SHA-256摘要算法；

4、支持的算法jarsigner使用keystore文件进行签名；

apksigner除了支持使用keystore文件进行签名外，还支持直接指定pem证书文件和私钥进行签名。

5、与对齐的关系；

### 了解V1签名

![img](https://upload-images.jianshu.io/upload_images/5714046-61377723ea8a2cb2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

反编译文件图


通过解压工具打开apk文件，会发现有一个META-INF目录，该目录中有3个文件，这3个文件是签名以后生成的，显然与签名相关，我们依次看这几个文件中的内容：**MANIFEST.MF、CERT.SF和CERT.RSA**。

**MANIFEST.MF：**APK中所有原始文件的数据摘要的Base64编码,而数据摘要算法就是SHA1或者SHA256

![img](https://upload-images.jianshu.io/upload_images/5714046-c13d182efac36cc5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

MANIFEST.MF

**CERT.SF：**

1》计算这个MANIFEST.MF文件的数据摘要（如SHA1算法），再经过BASE64编码后，记录在CERT.SF主属性块（在文件头上）的“SHA1-Digest-Manifest”属性值值下

2》逐条计算MANIFEST.MF文件中每一个块的数据摘要，并经过BASE64编码后，记录在CERT.SF中的同名块中，属性的名字是“SHA1-Digest

![img](https://upload-images.jianshu.io/upload_images/5714046-f50f4ed1c205226d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

CERT.SF

**CERT.RSA：**把之前生成的 CERT.SF文件， 用私钥计算出签名, 然后将签名以及包含公钥信息的数字证书一同写入生成CERT.RSA文件，用openssl命令才能查看其内容：openssl pkcs7 -inform DER -in CERT.RSA -noout -print_certs –text



![img](https://upload-images.jianshu.io/upload_images/5714046-7e2de469609022d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

CERT.RSA

#### V1签名的详细流程（具体可参考[SignApk.java](https://android.googlesource.com/platform/build/+/7e447ed/tools/signapk/SignApk.java)）    

生成签名过程

1.对APK内部所有文件（条目），提取摘要后BASE64生成指纹清单MANIFEST.MF;

2.对指纹清单提取摘要后BASE64生成CERT.SF;

3.对CERT.SF用私钥加密并且附加公钥生成CERT.RSA.

整个签名机制的最终产物就是MANIFEST.MF、CERT.SF、CERT.RSA三个文件

![img](https://upload-images.jianshu.io/upload_images/5714046-db2cfa81d47178de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

生成签名过程

####  V1 安装验证签名流程

1.校验CERT.SF文件

计算CERT.SF的摘要，与通过签名者公钥解密得到的摘要进行对比，如果一致进行下一步。

2.检验MANIFEST.MF文件

计算MANIFEST.MF文件的摘要，与CERT.SF主属性记录的摘要进行对比。

3.检验apk中每个文件的完整性

逐步计算apk中每个文件的摘要。与MANIFEST.MF中的记录进行对比，如果一致，则校验通过。

如果是升级安装，还需校验证书签名是否与已安装的一致。



![img](https://upload-images.jianshu.io/upload_images/5714046-bab4e5d6e99fb867.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

v1校验流程


在安装APK时，Android系统会校验签名，检查APK是否被篡改。代码流程是：PackageManagerService.java ->

PackageParser.java，PackageParser类负责V1签名的具体校验。

**V1签名是怎么保证APK文件不被篡改的？**

首先，如果破坏者修改了APK中的任何文件，那么被篡改文件的数据摘要的Base64编码就和MANIFEST.MF文件的记录值不一致，导致校验失败。

 其次，如果破坏者同时修改了对应文件在MANIFEST.MF文件中的Base64值，那么MANIFEST.MF中对应数据块的Base64值就和CERT.SF文件中的记录值不一致，导致校验失败。

最后，如果破坏者更进一步，同时修改了对应文件在CERT.SF文件中的Base64值，那么CERT.SF的数字签名就和CERT.RSA记录的签名不一致，也会校验失败。

那有没有可能继续伪造CERT.SF的数字签名那？理论上不可能，因为破坏者没有开发者的私钥。那破坏者是不是可以用自己的私钥和数字证书重新签名那，这倒是完全可以！

综上所述，任何对APK文件的修改，在安装时都会失败，除非对APK重新签名。但是相同包名，不同签名的APK也是不能同时安装的。
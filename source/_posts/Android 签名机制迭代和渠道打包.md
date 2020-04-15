---
title: Android 签名机制迭代（更新ing）
date: 2020-04-14 22:35:52
tags: [Android]
---

<meta name="referrer" content="no-referrer"/>

# 提纲

**初步了解**：是什么，为什么需要，有什么好处，怎么加密和验证，怎么使用，要点补充

**签名方案：**v1,v2,v3签名及校验流程，版本缺陷

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

#### V1签名的缺陷（也是迭代V2的原因）

**Ø不够安全，签名验证范围仅验证 元数据 以外的 文件 条目，这里说的元数据主要指meta-inf目录下的文件**

v1 签名方案，并不会保护

Apk 内的所有内容，有一些例外部分，被修改也并不会导致签名失效

这样，在验证APK 签名的时候，就需要处理大量不可信（尚未经过验证）的数据结构，然后还需要过滤并舍弃掉这部分不受签名保护的数据，再进行签名校验。也就是说你可以在已签名的文件中，增加一些不被签名保护的内容，这将导致受攻击的可能增大。

（图为[V1SchemeVerifier.java](https://www.androidos.net.cn/android/9.0.0_r8/xref/tools/apksig/src/main/java/com/android/apksig/internal/apk/v1/V1SchemeVerifier.java)）

![img](https://upload-images.jianshu.io/upload_images/5714046-8af3409c74537e4c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**Ø不够迅速：验证签名的时候，须解压所有已压缩的文件条目进行数据摘要的校验**

**v1**方案是对 APK 内部的被保护的原始文件（未压缩），单独进行计算数据摘要，所以在验证期间，也需要对每个文件进行解压再进行签名校验，来验证是否被篡改。所以在验证APK 签名的时候，必须解压 APK 的所有已压缩的文件条目进行数据摘要的校验，而这些，都将需要花费更多的时间和内存。

### 了解V2签名（Android 7.0引入） 

#### 引入目的：解决v1的问题

ØMANIFEST.MF中的数据摘要是基于原始未压缩文件计算的。因此**在校验时，需要先解压出原始文件，才能进行校验。而解压操作无疑是耗时的。**

ØV1签名仅仅校验APK第一部分中的文件，**缺少对APK的完整性校验。**因此，在签名后，我们还可以修改APK文件，例如：通过zipalign进行字节对齐后，仍然可以正常安装。

#### 怎么解决的

ØV2签名是**对APK本身进行数据摘要计算**，不存在解压APK的操作，减少了校验时间。

ØV2签名是**针对整个APK进行校验**（不包含签名块本身），因此对APK的任何修改（包括添加注释、zipalign字节对齐）都无法通过V2签名的校验。

某实验室数据（Nexus 6P、Android 7.1.1）可供参考

![img](https://upload-images.jianshu.io/upload_images/5714046-84c0660babb9e0e2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 了解APK文件结构（V1）



![img](https://upload-images.jianshu.io/upload_images/5714046-17602791de2ef5b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

APK文件本质上是一个ZIP压缩包，而ZIP格式是固定的，主要由三部分构成，如下图所示

Ø第一部分是内容块，所有的压缩文件都在这部分。每个压缩文件都有一个local file header，主要记录了文件名、压缩算法、压缩前后的文件大小、修改时间、CRC32值等。

Ø第二部分称为中央目录，包含了多个central directory file header（和第一部分的local file header一一对应），每个中央目录文件头主要记录了压缩算法、注释信息、对应local file header的偏移量等，方便快速定位数据。

Ø最后一部分是EOCD，主要记录了中央目录大小、偏移量和ZIP注释信息等

根据之前的V1签名和校验机制可知，V1签名只会检验第一部分的所有压缩文件，而不理会后两部分内容。

#### 了解APK文件结构（V2区别于V1）

v2 签名会在原先APK块中增加了一个新的块（签名块），新的块存储了签名、摘要、签名算法、证书链和额外属性等信息，这个块有特定的格式。

APK签名块位于中央目录之前，文件数据之后。V2签名同时修改了EOCD中的中央目录的偏移量，使签名后的APK还符合ZIP结构。

v2 签名块负责保护第1、3、4 部分的完整性，以及第2部分包含的APK 签名方案v2分块中的signed data分块的完整性。

![img](https://upload-images.jianshu.io/upload_images/5714046-7e17887fb5c783d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### V2签名原理

![img](https://upload-images.jianshu.io/upload_images/5714046-510338ebb3544c7c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### V2签名块的生成（可参考[ApkSignerV2](https://link.jianshu.com/?t=https://android.googlesource.com/platform/build/+/dd910c5/tools/signapk/src/com/android/signapk/ApkSignerV2.java)）

![img](https://upload-images.jianshu.io/upload_images/5714046-1ef3535758b994c0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

V2签名块的生成可参考[ApkSignerV2](https://link.jianshu.com/?t=https://android.googlesource.com/platform/build/+/dd910c5/tools/signapk/src/com/android/signapk/ApkSignerV2.java)，整体结构和流程如图

首先，根据多个签名算法，计算出整个APK的数据摘要，组成左上角的APK数据摘要集；

接着，把最左侧一列的数据摘要、数字证书和额外属性组装起来，形成类似于V1签名的“MF”文件（第二列第一行）；

其次，再用相同的私钥，不同的签名算法，计算出“MF”文件的数字签名，形成类似于V1签名的“SF”文件（第二列第二行）；

然后，把第二列的类似MF文件、类似SF文件和开发者公钥一起组装成通过单个keystore签名后的v2签名块（第三列第一行）。

最后，把多个keystore签名后的签名块组装起来，就是完整的V2签名块了（Android中允许使用多个keystore对apk进行签名）。

简而言之，单个keystore签名块主要由三部分组成，分别是上图中第二列的三个数据块：类似MF文件、类似SF文件和开发者公钥，其结构如图2所示

#### V2签名的校验流程（PackageManagerService.java 

#### -> PackageParser.java -> ApkSignatureSchemeV2Verifier.java）

![img](https://upload-images.jianshu.io/upload_images/5714046-d0135232252cdfe1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

CERT.SF

Android Gradle Plugin2.2之上默认会同时开启V1和V2签名，同时包含V1和V2签名的**CERT.SF文件会有一个特殊的主属性**，该属性会强制APK走V2校验流程（7.0之上），以充分利用V2签名的优势（速度快和更完善的校验机制）



![img](https://upload-images.jianshu.io/upload_images/5714046-e27e20c0d42cdbfd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同时包含V1和V2签名的APK的校验流程，优先校验V2，没有或者不认识V2，则校验V1。具体校验类似v1，但是不是校验一个个文件，而是apk的分段，如下图所示

![img](https://upload-images.jianshu.io/upload_images/5714046-ae41069523c9d0d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**V2签名是怎么保证APK不被篡改的？**

首先，如果破坏者修改了APK文件的任何部分（签名块本身除外），那么APK的数据摘要就和“MF”数据块中记录的数据摘要不一致，导致校验失败。

其次，如果破坏者同时修改了“MF”数据块中的数据摘要，那么“MF”数据块的数字签名就和“SF”数据块中记录的数字签名不一致，导致校验失败。

然后，如果破坏者使用自己的私钥去加密生成“SF”数据块，那么使用开发者的公钥去解密“SF”数据块中的数字签名就会失败；

最后，更进一步，若破坏者甚至替换了开发者公钥，那么使用数字证书中的公钥校验签名块中的公钥就会失败，这也正是数字证书的作用。

综上所述，任何对APK的修改，在安装时都会失败，除非对APK重新签名。但是相同包名，不同签名的APK也是不能同时安装的。

#### V2签名注意点补充

Q1：V2签名后EOCD的中央目录偏移量更改了，安装时怎么通过校验的？

A1：Android系统在校验APK的数据摘要时，首先会把EOCD的中央目录偏移量替换成签名块的偏移量，然后再计算数据摘要。而签名块的偏移量就是v2签名之前的中央目录偏移量

具体代码逻辑，可参考[ApkSignatureSchemeV2Verifier.java](https://link.jianshu.com/?t=http://link.zhihu.com/?target=https://android.googlesource.com/platform/frameworks/base/%2B/master/core/java/android/util/apk/ApkSignatureSchemeV2Verifier.java)。截图自https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/util/apk/ApkSigningBlockUtils.java



![img](https://upload-images.jianshu.io/upload_images/5714046-a890c2c5122279d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

代码截自ApkSigningBlockUtils.java

Q2：APK先V1签名，还是V2先签名？

A2：先V1，再V2，因为JAR签名需要修改zip数据区和中央目录的内容，先使用V2签名再JAR签名会破坏V2签名的完整性。

Q3：删掉V2签名，能否绕过V2签名？

A3：不行，签名CERT.SF文件中的标志也是防回滚的保护，防止删除高版本签名，仅验证低版本签名



![img](https://upload-images.jianshu.io/upload_images/5714046-d66e2ce0ac116348.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Q4：APK签名时，只有V2签名，没有V1签名行不行？

A4：这种情况是可以编译通过的，并且在Android 7.0之上也可以正确安装和运行。但是7.0之下，因为不认识V2，又没有V1签名，所以会报没有签名的错误。

#### V2签名的缺陷：没支持签名的更新（迭代v3原因） 

Ø签名有默认25年的有效期限

Ø签名泄漏想替换

### 了解V3签名（Android 9.0 引入） 



![img](https://upload-images.jianshu.io/upload_images/5714046-71ef46ed5c53aa37.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

V3签名结构基于V2， 增加支持**密钥轮转**

V3 签名**新增的新块（attr）存储了所有的签名信息，由更小的 Level 块，以链表的形式存储**。

其中每个节点都包含用于为之前版本的应用签名的签名证书，最旧的签名证书对应根节点，系统会让每个节点中的证书为列表中下一个证书签名，从而为每个新密钥提供证据来证明它应该像旧密钥一样可信。



![img](https://upload-images.jianshu.io/upload_images/5714046-c741f6efa5a146e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

V3 方案还没有正式开放，在最新版的Build Tools 版本28.0.3中的Apksigner，尚不支持V3的APK签名方案。想尝鲜可以通过源代码自行编译。

[https://developer.android.google.cn/studio/command-line/apksigner#usage-verify](https://developer.android.google.cn/studio/command-line/apksigner)

到这里，就了解完了签名的机制，下一篇讲下android的打包机制。感谢支持
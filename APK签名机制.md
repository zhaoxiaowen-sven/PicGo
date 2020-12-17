# APK签名机制

# 1、什么是apk签名

android应用在安装过程中会对apk进行签名校验，主要用于验证apk的可靠性、安全性以及唯一性，保证apk是有可信性的发布者发布，防止发布后被篡改；另外在apk升级时除了包名一致，签名也要一致。要了解签名和验签过程需要先了解以下几个基本概念。

## 1.1、基本概念

### 1.1.1、数字摘要 

数字摘要就是采用单向Hash函数将需要加密的明文“摘要”成一串固定长度（128位）的密文这一串密文又称为数字指纹，它有固定的长度，而且不同的明文摘要成密文，其结果总是不同的，而同样的明文其摘要必定一致。常用的数字摘要技术（Digital Digest）也称作为安全HASH编码法（SHA：Secure Hash Algorithm）。对所要传输的数据进行运算生成信息摘要，它并不是一种加密机制，但却能产生信息的数字"指纹"，它的目的是为了确保数据没有被修改或变化，保证信息的完整性不被破坏。

### 1.1.2、数字签名

数字签名的作用就是保证信息传输的完整性、发送者的身份认证、防止交易中的抵赖发生。**数字签名技术是将摘要信息用发送者的私钥加密**，与原文一起传送给接收者。接收者只有**用发送者的公钥才能解密被加密的摘要信息然后用HASH函数对收到的原文产生一个摘要信息，与解密的摘要信息对比**。如果相同，则说明收到的信息是完整的，在传输过程中没有被修改，否则说明信息被修改过，因此数字签名能够验证信息的完整性。

### 1.1.3、数字证书

数字证书是由权威公证的第三方认证机构（即CA，Certificate Authority）负责签发和管理的、个人或企业的网络数字身份证明。A的数字签名可以类比为现实世界中的签名，用来证明一个文件或者消息是A签署的，通常是使用A的私钥对消息摘要加密而得到，其他人可以使用A的公钥对数字签名进行验证。但是怎么才能信任A的公钥呢？让A自己证明自己是一件很难的事情，因此就需要第三方来证明，这就是数字证书的意义所在。

## 1.2、apk签名和验签原理

![image-20201216113655542](pics/image-20201216113655542.png)

### 1.2.1、APK签名过程

1. 计算摘要：使用数字摘要算法计算出apk的摘要；
2. 签名：通过私钥对摘要进行加密，加密后的信息就是签名；
3. 写入签名：将签名信息、证书以及公钥写入到文件中。

### 1.2.2、APK验签过程

1. 解密签名：通过公钥解密签名信息获得摘要；
2. 计算摘要：使用摘要算法从接收的数据中计算摘要；
3. 比较摘要：比较解密出的摘要和通过文件计算的摘要，若一致，则校验通过。

接下来介绍下现有的几种apk签名的方式。

# 2、v1 签名

V1签名又称为JAR签名，是对jar包进行签名的一种机制，由于jar包apk本质上都是zip包，所以可以应用到对apk的签名。解压apk后，META-INF目录中存放的就是签名相关的文件。

## 2.1 META-INF

META-INF 文件夹下有三个文件：MANIFEST.MF、CERT.SF、CERT.RSA。它们就是签名过程中生成的文件，作用如下。

### 2.1.1、MANIFEST.MF

对APK中所有文件计算摘要保存到该文件中。

```
Manifest-Version: 1.0
Created-By: 1.8.0_212 (Oracle Corporation)

Name: AndroidManifest.xml
SHA1-Digest: GpiU1HOPO9rxpTPh43kG1XVG8iw=

Name: META-INF/BdTuringSdk_cnRelease.kotlin_module
SHA1-Digest: PVHPdoZ9+09Zq0PF+eJz0yRVf10=
...
```

### 2.1.2、CERT.SF

- SHA1-Digest-Manifest-Main-Attributes：对 MANIFEST.MF 头部计算摘要。
- SHA1-Digest-Manifest：对 MANIFEST.MF 文件计算摘要。
- SHA1-Digest：对 MANIFEST.MF 的各个条目计算摘要。

```
Signature-Version: 1.0
SHA1-Digest-Manifest-Main-Attributes: TN5zBsqBLAij6alOeMWe+Ejwd4g=
SHA1-Digest-Manifest: PBUX5Kag9TIOJy4jZ57vwuAur1Y=
Created-By: 1.8.0_45-internal (Oracle Corporation)

Name: res/layout/ac.xml
SHA1-Digest: mYQig54fsd3pTRQTmTwMD2oO5CM=
```

### 2.1.3、CERT.RSA

**对CERT.SF 文件的摘要通过私钥加密生成校验串**, 然后和**数字签名、公钥、数字证书**一同写入 CERT.RSA 中保存。很多文章将校验串描述成签名，这样的理解是不准确的。可以比较2个同一个公司出品的apk的RSA文件，你会发现可能除了结尾部分不太一样外，其他部分基本相同，原因其实就是同一个公司出品的apk，它的签名，证书，公钥通常都是相同的，只有通过私钥加密的CERT.SF的摘要不同。 如下图：

![image-20201217160514944](pics/image-20201217160514944.png)

#### 1、查看证书与公钥

1、将.rsa 后缀改为.p7b文件，双击直接打开

2、openssl 命令查看证书信息（公钥在证书信息中）

```
// 查看.RSA文件中证书信息
openssl pkcs7 -inform DER -in XXX.RSA -noout -print_certs -text

// 查看本地证书的公钥和私钥
keytool -list -rfc --keystore test.jks | openssl x509 -inform pem -pubkey
```

通过一个实例去理解一下这两种方式的区别和联系。

1. 使用AS或keytool生成一个.jks签名文件； [Android studio 如何生成jks签名文件](https://www.jianshu.com/p/b28a5be05029)
2. 使用签名文件对apk进行签名；
3. 通过相关命令查看.jks文件以及解压apk中的.rsa 文件。

![image-20201217162850396](pics/image-20201217162850396.png)

#### 2、查看签名

其实能看到也是签名的摘要，不是真正的签名。

1. 将.rsa 后缀改为.p7b文件，双击直接打开
2. 使用keytool的命令查看.RSA文件

```
keytool -printcert -file xxx.RSA
```

**证书信息：**

![image-20201217163225037](pics/image-20201217163225037.png)

## 2.2、v1签名过程

[apksigner源码](https://android.googlesource.com/platform/build/+/7e447ed/tools/signapk/SignApk.java)

## 2.3、v1验签过程

[验签源码](https://android.googlesource.com/platform/frameworks/base/+/android-5.1.1_r38/services/core/java/com/android/server/pm/PackageManagerService.java)

## 2.4、v1签名的劣势

1. 签名校验速度慢

   校验过程中需要对apk中所有文件进行摘要计算，在apk资源很多、性能较差的机器上签名校验会花费较长时间，导致安装速度慢；

2. 完整性保障不够

   META-INF目录用来存放签名，自然此目录本身是不计入签名校验过程的，可以随意在这个目录中添加文件，比如一些快速批量打包方案就选择在这个目录中添加渠道文件。

   为了解决这两个问题，Android 7.0推出了全新的签名方案V2，下面介绍下v2签名。

# 2、v2签名

## 2.1、ZIP文件结构

![image-20201217175101791](pics/image-20201217175101791.png)



zip文件分为3部分：

1. **数据区**

   主要存放压缩的文件数据

2. **中央目录**

   存放数据区压缩文件的索引

3. **中央目录结尾记录**

   存放中央目录的文件索引

查找压缩文件中数据可以先中央目录起始偏移量和size即可定位到中央目录，再遍历中央目录条目，根据本地文件头的起始偏移量即可在数据区中找到相应数据。

## 2.2、v2签名原理

JAR签名是在apk文件中添加META-INF目录，即需要修改数据区、中央目录，此外，添加文件后会导致中央目录大小和偏移量发生变化，还需要修改中央目录结尾记录。

v2方案为加强数据完整性保证，不在数据区和中央目录中插入数据，选择在 数据区和中央目录之间插入一个APK签名分块，从而保证了原始数据的完整性。

![image-20201217174414400](pics/image-20201217174414400.png)

APK 签名方案 v2 负责保护第 1、3、4 部分的完整性，以及第 2 部分包含的“APK 签名方案 v2 分块”中的 `signed data` 分块的完整性。第 1、3 和 4 部分的完整性通过其内容的一个或多个摘要来保护，这些摘要存储在 `signed data` 分块中，而这些分块则通过一个或多个签名来保护。

### 2.2.1、APK摘要计算

![image-20201217202801891](pics/image-20201217202801891.png)

第 1、3 和 4 部分的摘要采用以下计算方式：

1. 将APK拆分成多个大小为 1 MB大小的连续块，最后一个块可能小于1M。之所以分块，是为了可以通过并行计算摘要以加快计算速度；
2. 计算块的摘要，以字节 0xa5 + 块的长度（字节数） + 块的内容 进行计算；
3. 计算整体摘要，字节 0x5a + 块数 + 块的摘要的连接（按块在 APK 中的顺序）进行计算。

注意：由于第 4 部分（ZIP 中央目录结尾）包含“ZIP 中央目录”的偏移量，因此该部分的保护比较复杂。当“APK 签名分块”的大小发生变化（例如，添加了新签名）时，偏移量也会随之改变。因此，在通过“ZIP 中央目录结尾”计算摘要时，必须将包含“ZIP 中央目录”偏移量的字段视为包含“APK 签名分块”的偏移量。

计算完的摘要经过私钥加密写入签名块。

## 2.3、APKSigning Block 

APK签名分块包含了4部分：分块长度、ID-VALUE序列、分块长度、固定magic值。其中APK 签名方案 v2分块存放在ID为**0x7109871a**的键值对中。

![image-20201217212653708](pics/image-20201217212653708.png)

### 2.3.1、v2 Block

### 2.3.2、EOCD结构 

### 2.3.3、v2 Block定位



## 2.4、v2签名过程

在进行签名校验时，

1. 找到zip中央目录结尾记录，从该记录中找到中央目录起始偏移量，
2. 通过magic值（APK Sig Block 42）即可确定前方可能是APK签名分块，再通过前后两个分块长度字段，即可确定APK签名分块的位置
3. 通过ID（0x7109871a）定位APK 签名方案 v2分块位置。

## 2.5、v2验签过程

![image-20201217232339423](pics/image-20201217232339423.png)

# 3、V3签名

# 4、v4签名




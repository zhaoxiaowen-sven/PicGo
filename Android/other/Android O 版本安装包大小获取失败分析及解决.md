##一、背景：
Android O版本上获取安装包大小报错：
![pic0](https://leanote.com/api/file/getImage?fileId=59ca517eab64417152002e2c)

##二、调用过程分析：
(1)8.0 以下读取安装包大小通常会反射调用PackageManage.getPackageSizeInfo
参考http://blog.csdn.net/qinjuning/article/details/6892054 
![pic1](https://leanote.com/api/file/getImage?fileId=59ca4f72ab64417152002df4)
(2)PackageManager 调用到getPackageSizeInfo -> getPackageSizeInfoAsUser

(3)PackageManager是个抽象类，具体的实现是在ApplicationPackageManager.getPackageSizeInfoAsUser中
对应8.0的代码
 ![pic2](https://leanote.com/api/file/getImage?fileId=59ca51a7ab64417152002e2e)
可以看到逻辑上判断如果apk 的targetsdkversion>= 8.0版本才会有这个错误，逻辑上26以下是不会走到这个case里的，这是目前待check的问题。
##三、解决方案：
所以这问题其实是由于Android8.0 新特性导致的问题，目前有2种：
1．	框架修改getPackageSizeInfoAsUser逻辑，不推荐
2．	使用StorageStatsManager，在8.0的PackageManager.getPackageSizeInfo方法中：
![pic3](https://leanote.com/api/file/getImage?fileId=59ca4f72ab64417152002df8)
 Google 推荐我们使用的是这个类StorageStatsManager，这个接口在8.0新加的，8.0以下是不存在的，需要对8.0和8.0 以下做兼容。
frameworks\base\core\java\android\app\usage\StorageStats.java
frameworks\base\core\java\android\content\pm\PackageStats.java
StorageStats应用数据分成了3种：1.apk大小 2.用户数据 3用户缓存数据，用户数据包含缓存数据。3种数据的路径如下：
codeBytes = appBytes data/app/pkgname/ 
dataBytes = data/data/pkgname/  +  sdcard/Android/data/pkgname/
cacheBytes = data/data/pkgname/cache/ + sdcard/Android/data/pkgname/cache/
相比于8.0 之前的PackageStats，将数据做了简化和整合。Android应该是将内部存储和sd分成2个StorageVolume ，根据对不同的uuid获取相关的数据。（个人理解是这样）。
示例代码：
![pic4](https://leanote.com/api/file/getImage?fileId=59ca4f72ab64417152002df7)
注意：
1.	androidstudio编译时，api的编译版本必须是26
![pic5](https://leanote.com/api/file/getImage?fileId=59ca52b9ab64417152002e55)	 
2.	app必须是系统app，且要声明权限
![pic6](https://leanote.com/api/file/getImage?fileId=59ca52f1ab6441737b002f42)

参考：
https://stackoverflow.com/questions/43472398/how-to-use-storagestatsmanager-querystatsforpackage-on-android-o

附：7.0 , 8.0 getPackageSizeInfo逻辑对比：
Android 7.0
frameworks\base\core\java\android\content\pm\PackageManager.java
 ![pic6](https://leanote.com/api/file/getImage?fileId=59ca4f72ab64417152002dfa)
 ![pic7](https://leanote.com/api/file/getImage?fileId=59ca4f72ab64417152002df2)
frameworks/base/core/java/android/app/ApplicationPackageManager.java

Android 8.0 
frameworks\base\core\java\android\content\pm\PackageManager.java
 ![pic8](https://leanote.com/api/file/getImage?fileId=59ca4f72ab64417152002df3)
frameworks/base/core/java/android/app/ApplicationPackageManager.java
 ![pic9](https://leanote.com/api/file/getImage?fileId=59ca4f72ab64417152002df9)
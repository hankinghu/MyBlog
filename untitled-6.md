# 减少apk大小\_有图有真相-CSDN博客

## 前言

apk的大小对于下载apk应用的用户多少有直接的影响，由于手机内存的限制和网络环境的限制，同一个应用随着apk越大下载的和使用的人数就会越少，所以减少apk的大小是非常重要的。本文从apk编译过程，apk的组成，apk大小减少的方法三个方面分析如何减少apk的大小

## 1、apk的编译过程

在构建过程中，Android项目会被编译，打包，生成.apk文件，apk文件包含了运行的全部必要信息。主要包括：  
 **1、.dex文件**

**2、AndroidManifest.xml文件**

**3、编译的资源文件resources.arsc**

**4、未编译的资源文件**

### Android编译打包简单流程图

：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20191221104658393.png)  
 上面的流程图中简单的概括了Android编译和打包过程，Android项目编译和打包生成apk文件，apk文件签名后，使用adb工具安装在Android device或者emulator上就可以运行了。

### Android 具体的build过程

.apk文件要通过打包过程中许多工具生成中间文件最终生成的。

**apk打包工具**

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20191221134559252.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9oYW5raW5nLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70)  
 由上图看出Android 构建过程中包括的构建工具有：

**aapt**（The Android Asset Packaging Tool ）：aapt Android中的资源打包工具，主要是用于处理Android项目中的资源文件比如xml，drawable，等文件。

**Java compiler：** java编译器，编译java文件生成.class文件。

**dex：** dex生成器，dex会把class转化为Dalvik byte code，也会把第三方的源码编译进.dex文件中。

**apkbuilder：** apk生成器，编译过的资源文件和未编译的资源文件以及dex文件还有其他文件通过apk builder生成apk文件

**jarsigner:** 签名器，.apk文件通过签名后生成signed.apk文件，签名后的apk就可以安装和运行了。

**apk的build流程图**

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20191221111458185.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9oYW5raW5nLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70)  
 上面的流程图可以很清晰的看到打包整个打包的过程：

**1、资源打包过程**

首先项目中资源文件通过aapt资源打包工具生成R.java文件，和编译过的资源文件resources.arsc，R.java文件中存放的是各个资源的id，在java代码中通过R文件中的id来找到对应的资源。resources.arsc中放的是编译过的资源文件相关的信息：比如**xml文件的节点**（LinearLayout, RelativeLayout）**布局的属性**（ android:layout\_width）**资源的id**。下图是资源打包更具体的流程：

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20191221113909586.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9oYW5raW5nLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70)  
 **2、class文件生成**

资源打包过程中生成的R.java文件中包含了资源的id，java代码中通过这些id可以寻找到相关的资源。R.java类和项目中其他java源码，以及aidl接口通过java compiler生成.class文件。

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20191221133750970.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9oYW5raW5nLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70)  
 **3、dex文件生成**

上面项目源码中生成的class文件和第三方库通过dex生成.dex文件，Android运行在Dalvik Virtual Machine上，和Java不同，Java运行在jvm上，Dalvik Virtual Machine上运行的不是.class文件，而是.dex文件。  
 可以使用android-sdk中的dexdump 工具来反编译dex文件。dexer工具可以把class文件转换成.dex文件。

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20191221140928922.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9oYW5raW5nLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70)

**dex的优点**  
 dexer打包工具会消除class文件中的冗余信息，所有的class文件会被打包进同一个dex文件中，使用dex会很大的减少apk的大小，下面表格是使用dex工具后文件的大小对比：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20191221141403750.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9oYW5raW5nLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70)  
 由上表可知使用dex文件可以极大的减少应用的大小。

## 2、apk结构

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20191221144337735.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9oYW5raW5nLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70)

由上图可以看出apk主要由META-INF，assets,res,lib,resources.arsc,classes.dex,AndroidManifest.xml,等8个部分组成，下面分别对这个8个组成进行分析。  
 **1、META-INF**  
 apk实际上是一个包含了各种结构信息的jar文件，META-INF文件夹包含了manifest信息和jar的包信息META-INF主要包括下图的三个部分组成：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20191221145228216.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9oYW5raW5nLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70)

**1、MANIFEST.MF**:包含导入jar运行时的信息，包括运行时启动哪个main class。package的版本号，build number。package的创建者信息，安全和允许证书信息等。

**2、CERT.SF**: 包含所有文件列表信息以及文件的SHA-1摘要。

**3、CERT.RSA**: 包含CERT.SF文件的签名内容以及用于对该内容签名的公钥证书。

一个简单的META-INF文件

```text
Manifest-Version: 1.0
Created-By: 1.7.0_60 (Oracle Corporation)

Name: res/drawable-xxhdpi-v4/common_plus_signin_btn_text_dark_pressed.9.png
SHA1-Digest: Db3E0/I85K9Aik2yJ4X1dDP3Wq0=

Name: res/drawable-xhdpi-v4/opt_more_item_close_press.9.png
SHA1-Digest: Xxm9cr4gDbEEnnYvxRWfzcIXBEM=

Name: res/anim/accessibility_guide_translate_out.xml
SHA1-Digest: dp8PyrXMy2IBxgTz19x7DATpqz8=
```

### 2、RES

res会在R.java生成索引ID，在打包的时候判断资源有没有用到，没用到的时候不会被打包进apk中\(res/raw文件夹除外\), res用getResource\(\)访问,res/raw里的文件在打包的时候不会被系统二进制编译，都被原封不动打包进APK，通常用来存放游戏资源、脚本、字体文件等,res/xml会被编译成二进制文件。res/anim存放动画资源。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20191214160946581.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9oYW5raW5nLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70)  
 **注意：res中的资源不会被编译进 resources.arsc中。**

### assets和res的区别

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20191214163502939.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9oYW5raW5nLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70)

### 3、resources.arsc

Android编译过程中Android的资源打包工具（Android Asset Packaging Tool）aapt，会生成一个R.java文件，这个文件保存layout，styles，等资源的唯一id。并且生成一个resources.arsc文件，这个文件中保存了资源的属性信息比如：  
 **1、xml文件的节点**（LinearLayout, RelativeLayout）  
 **2、布局的属性**（ android:layout\_width）  
 **3、资源的id**  
 resources.arsc中的信息会在运行的时候被解析。

## 3、减少apk大小

由上面知道了apk的组成，可知要减少apk的大小就是要减少上面apk组成中的各个模块的大小，减小包大小的方法如下图所示：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20191221151556768.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9oYW5raW5nLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70)

## 减少资源数目移除无用资源

### 1、使用lint工具进行检测

```text
res/layout/preferences.xml: Warning: The resource R.layout.preferences appears
    to be unused [UnusedResources]
```

注意lint不会检测asset文件夹中的文件

### 2、使用shrinkResources

如果如果在 app的build.gradle 文件中配置shrinkResources属性，gradle会自动移除无用资源

```text
android {
    

    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```

进行了上面的配置后在build的过程中，ProGuard会先移除无用的代码但是不会移除无用的资源，Gradle会移除无用的资源。

### 减少动画帧

帧动画由于每一帧都是一幅图片所以非常容易增加apk的大小，减少帧数能够非常有效的减少apk大小。下面是一个帧率为30的动画包含的图片，![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20191208104548300.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9oYW5raW5nLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70)  
 包含了接近30张图片，如果把帧数换成15，图片数目会减少一半。

### 使用xml定义纯色图片

有些图片并不需要静态的图片资源，对于一些纯色的图片尽量使用用shape，color的xml文件来代替图片，使用xml定义的shape能够有效的减少apk大小。

### 复用图片资源

在开发中有些图片只是颜色不一样，有些图片只是大小不一样，有些图片只是角度不一样。Android提供了相应的方法来复用这些图片。

```text
<?xml version="1.0" encoding="utf-8"?>
<rotate xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/ic_arrow_expand"
    android:fromDegrees="180"
    android:pivotX="50%"
    android:pivotY="50%"
    android:toDegrees="180" />

```

### 压缩图片

**Android Asset Packaging Tool**  
 Android Asset Packaging Tool简称aapt，AAPT 能够查看，创建，更新，zip文件，并且能够将资源编译成二进制文件，是Android sdk中自带的打包class和resource工具。

> The Android Asset Packaging Tool \(aapt\) takes your application  
>  resource files, such as the AndroidManifest.xml file and the XML files  
>  for your Activities, and compiles them。

能够在：`..\android-sdk\build-tools\`包中找到。

**aapt命令的使用**

*  创建apks \(Android application bundles\)

```text
aapt package, aapt add 
```

*  aapt dump 查看apk的信息:  1、查看版本号：

```text
aapt dump badging name.apk
```

2、查看权限：

```text
aapt dump permissions name.apk
```

使用aapt工具对图片进行压缩，aapt工具在build过程中能对res/drawable/文件夹下的文件进行无损压缩。

aapt的缺点  
 1、aapt不会压缩 asset/文件夹下的png图片  
 2、  
 3、aapt不会过滤已经压缩过的图片，所以会对已经压缩过的图片造成影响，为了避免这种影响可以在gradle中使用

```text
aaptOptions {
    cruncherEnabled = false
}
```

### 减少使用枚举

一个枚举类会造成class.dex增大1.0到1.4kb大小。  
 If possible, consider using the @IntDef annotation and ProGuard to strip enumerations out and convert them to integers. This type conversion preserves all of the type safety benefits of enums

### 参考文献

1、[https://stackoverflow.com/questions/6517151/how-does-the-mapping-between-android-resources-and-resources-id-work](https://stackoverflow.com/questions/6517151/how-does-the-mapping-between-android-resources-and-resources-id-work)

2、[https://stackoverflow.com/questions/27548810/android-compiled-resources-resources-arsc](https://stackoverflow.com/questions/27548810/android-compiled-resources-resources-arsc)

3、[https://developer.android.com/guide/topics/resources/providing-resources.html\#AlternativeResources](https://developer.android.com/guide/topics/resources/providing-resources.html#AlternativeResources)

4、[https://developer.android.com/reference/android/widget/ImageView\#attr\_android:tint](https://developer.android.com/reference/android/widget/ImageView#attr_android:tint)

5、[https://developer.android.com/guide/topics/graphics/vector-drawable-resources](https://developer.android.com/guide/topics/graphics/vector-drawable-resources)

6、[https://stackoverflow.com/questions/28234671/what-is-aapt-android-asset-packaging-tool-and-how-does-it-work/28234924](https://stackoverflow.com/questions/28234671/what-is-aapt-android-asset-packaging-tool-and-how-does-it-work/28234924)

7、[https://elinux.org/Android\_aapt](https://elinux.org/Android_aapt)

8、[https://developer.android.com/guide/topics/resources/providing-resources\#Accessing](https://developer.android.com/guide/topics/resources/providing-resources#Accessing)  
 9、[https://stackoverflow.com/questions/39305775/what-are-the-purposes-of-files-in-meta-inf-folder-of-an-apk-file/39305776](https://stackoverflow.com/questions/39305775/what-are-the-purposes-of-files-in-meta-inf-folder-of-an-apk-file/39305776)  
 10、[https://stackoverflow.com/questions/5583608/difference-between-res-and-assets-directories](https://stackoverflow.com/questions/5583608/difference-between-res-and-assets-directories)  
 11、[https://stackoverflow.com/questions/18951048/how-does-the-android-build-process-work](https://stackoverflow.com/questions/18951048/how-does-the-android-build-process-work)

12、[https://stuff.mit.edu/afs/sipb/project/android/docs/tools/building/index.html](https://stuff.mit.edu/afs/sipb/project/android/docs/tools/building/index.html)

13、[https://medium.com/@kevalpatel2106/how-you-can-decrease-application-size-by-60-in-only-5-minutes-47eff3e7874e](https://medium.com/@kevalpatel2106/how-you-can-decrease-application-size-by-60-in-only-5-minutes-47eff3e7874e)

14、[https://stackoverflow.com/questions/7750448/what-are-dex-files-in-android](https://stackoverflow.com/questions/7750448/what-are-dex-files-in-android)

15、[https://source.android.com/devices/tech/dalvik/dex-format.html](https://source.android.com/devices/tech/dalvik/dex-format.html)


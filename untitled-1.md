# android中hprof文件分析\_有图有真相-CSDN博客

## Hprof基本概念

hprof最初是由J2SE支持的一种二进制堆转储格式，hprof文件保存了当前java堆上所有的内存使用信息，能够完整的反映虚拟机当前的内存状态。  
 格式  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20210407143616481.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)  
 Hprof文件由FixedHead和一系列的Record组成，Record包含字符串信息、类信息、栈信息、GcRoot信息、对象信息。每个Record都是由1个字节的Tag、4个字节的Time、4个字节的Length和Body组成，Tag表示该Record的类型，Body部分为该Record的内容，长度为Length。

## Android中的hprof

在Android设备上，我们可以通过两种方式生成当前进程的hprof文件

通过调用Debug.dumpHprofData\(String filePath\)方法来生成hprof文件  
 通过执行shell命令adb shell am dumpheap pid /data/local/tmp/x.hprof来生成指定进程的hprof文件到目标目录

然而Android平台上的hprof文件和标准Java的hprof定义有一些区别，主要是版本号不一样，而且增加了一些特殊Tag，在art/runtime/hprof/hprof.cc中有如下定义  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20210407143745254.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)  
 可以看出Android增加了额外的9个HeapTag，其中比较重要的是HPROF\_HEAP\_DUMP\_INFO。Android上将java堆分为Heap-App、Heap-Image、Heap-Zygote三块，这个Tag的作用是切换当前的堆，该Tag后面紧跟着一个4个字节的堆id和一个堆名称id。  
 比如出现一个HPROF\_HEAP\_DUMP\_INFO Record，且该Record表示Heap-Image，那么表示后续所有Record中的类、对象、GCRoot对象均在Heap-Image中存储，直到下一个HPROF\_HEAP\_DUMP\_INFO出现为止。

## 内存泄漏

内存泄漏指的是一个本该被回收的对象因为某些原因导致其不能被回收，通俗来说就是该对象理论上不再使用，但是仍无法被回收。

## Android中的泄漏对象

判断一个对象是否泄漏首先要判断该对象是否不再使用，想要判断这一点则需要对象有明显的生命周期，在Android中有以下对象可以判断是否泄漏:

1、Activity: 通过Activity的mDestroyed属性来判断该Activity是否已经销毁，如果已经销毁且未被回收则认为是泄漏  
 Fragment: 通过Fragment的mFragmentManager是否为空来判断该Fragment是否处于无用状态，如果mFragmentManager为空且未被回收则认为是泄漏。

2、View: 通过unwrapper mContext获得Activity，如果存在Activity，则判断该Activity是否泄漏。

Editor: Editor指的是android.widget包下的Editor，是用于TextView处理editable text的辅助类，通过mTextView是否为空来判断Editor是否处于无用状态，如果mTextView为空且未被回收则认为是泄漏。

3、ContextWrapper: 通过unwrapper ContextWrapper获得Activity，如果存在Activity，则判断该Activity是否泄漏。

4、Dialog: 通过mDecor是否为空判断该Dialog是否处于无用状态，如果mDecor为空且未被回收则认为是泄漏。

5、MessageQueue: 通过mQuitting或者mQuiting\(应该是历史原因，前期拼写错误为mQuiting，后来改正\)来判断MessageQueue是否已经退出，如果已经退出且未被回收则认为是泄漏。

ViewRootImpl: 通过ViewRootImpl的mView是否为空来判断该ViewRootImpl是否处于无用状态，如果mView为空且未被回收则认为是泄漏。

6、Window: 通过mDestroyed来判断该Window是否处于无用状态，如果mDestroyed为true且未被回收则认为是泄漏。  
 Toast: 拿到mTN，通过mTN的mView是否为空来判断当前Toast是否已经hide，如果已经hide且未被回收则认为是泄漏。

## 泄漏的形式

泄漏的本质就是无用对象被持有导致无法回收，具体的形式有如下几种：

1、非静态内部类、匿名内部类持有外部类对象引用: 一般为用于回调的Listener，该Listener被别的地方持有，间接导致外部类对象被泄漏。  
 2、Handler: 在Activity中定义Handler对象的时候，Handler持有Activity同时Message持有Handler，而Message被MessageQueue持有，最终导致Activity泄漏。

3、资源对象未关闭: 数据库连接、Cursor、IO流等使用完后未close。

4、属性动画: 使用ValueAnimator和ObjectAnimator的时候，未及时关闭动画导致泄漏。Animator内部向AnimationHandler注册listener，AnimationHandler是一个单例，如果不及时cancel，会导致Animator泄漏，间接导致Activity/Fragment/View泄漏（比如Animator的updateListener一般都以匿名内部类实现）

5、逻辑问题: 注册监听器之后未及时解注册，比如使用EventBus的时候没有在合适的时候进行解注册

## profiler使用

通过下载hprof文件，解压缩后，直接拖到Android studio中，可以通过Android studio中的profiler工具进行分析，如下图:  
 1、Heap Dump  
 1、筛选  
 左边区域可以通过筛选查看到泄漏的activity或者fragment，在Android中的泄漏主要是activity或者fragment有生命周期的类泄漏。  
 2、Heap区分  
 Heap 中主要分为三种：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20210422113756455.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)  
 **1、app heap**  
 当前APP从堆中分配的内存  
 **2、image heap**  
 系统启动映像，包含启动期间预加载的类。 此处的分配保证绝不会移动或消失  
 **3、zygote heap**  
 zygote是所有APP进程的母进程，linux 的进程使用COW技术，所有的APP共享zygote的内存空间，因此堆的话也继承了，并且zygote的这些空间不允许写入，为了加快java的启动和运行速度，zygote在启动时预加载了许多资源和代码，这样可以提高APP的运行速率.  
 源码

```text
enum HprofHeapId {
 HPROF_HEAP_DEFAULT= 0,
 HPROF_HEAP_ZYGOTE= 'Z',
 HPROF_HEAP_APP= 'A',
 HPROF_HEAP_IMAGE= 'I',
};
 if (space->IsZygoteSpace()) {
     heap_type= HPROF_HEAP_ZYGOTE;
     VisitRoot(obj, RootInfo(kRootVMInternal));
   } else if (space->IsImageSpace() && heap->ObjectIsInBootImageSpace(obj)) {
     // Only countobjects in the boot image as HPROF_HEAP_IMAGE, this leaves app image objects as
     // HPROF_HEAP_APP.b/35762934
     heap_type= HPROF_HEAP_IMAGE;
     VisitRoot(obj, RootInfo(kRootVMInternal));
   }
 } else {
   const auto* los = heap->GetLargeObjectsSpace();
   if (los->Contains(obj) && los->IsZygoteLargeObject(Thread::Current(), obj)) {
     heap_type= HPROF_HEAP_ZYGOTE;
     VisitRoot(obj, RootInfo(kRootVMInternal));
   }

```

## 2、Instance View

点击heap dump中具体的类，右边就会展示instance view  
 Instance view中主要展示对象相关的信息  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/2021042214562094.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)

**1、Depth**  
 depth是从gc roots到选中的当前对象的引用链最短长度。在java垃圾回收机制中，被gc roots引用到的对象不会被回收。如下图：在gc root 引用树之外的对象会被回收。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20210422145735833.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)  
 Android中gc root 主要分为下面几类：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20210422145757845.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)  
 **2、Native Size**  
 native对象的内存大小，这个值只有在Android 7.0以及更高的版本可见。  
 **3、Shallow Size**  
 对象本身占用的内存，不包括它引用的其他实例

```text
Shallow Size = [类定义] + 父类fields所占空间 + 自身fields所占空间 + [alignment]
```

**4、Retained Size**  
 Retained Size是指, 当实例A被回收时, 可以同时被回收的实例的Shallow Size之和，所以进行内存分析时,应该重点关注Retained Size较大的实例; 或者可以通过Retained Size判断出某A实例内部使用的实例是否被其他实例引用.  
 如下图：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/2021042214593829.png)  
 上图中：obj1的retained size=obj1+obj2+obj4的shallow size,注意由于obj3被gc roots直接引用，所以这个值不计算到obj1的retained size中。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20210422145955823.png)  
 如上图：obj1的retained size=obj1+obj2+ obj3+obj4的shallow size

## 3、Reference

reference里面主要是类的引用关系，可以通过引用关系，一层一层的查看类是如何泄漏的。

## 参考

[1、Android中内存泄漏相关](https://1014277960.github.io/2019/10/09/Android%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E7%AE%80%E4%BB%8B/)  
 [2、hprof文件相关](https://1014277960.github.io/2019/09/18/%E5%85%B3%E4%BA%8EHprof%E7%9A%84%E6%96%B9%E6%96%B9%E9%9D%A2%E9%9D%A2/)  
 [3、profile分析内存泄漏](https://developer.android.com/studio/profile/memory-profiler?utm_source=android-studio#profiler-memory-leak-detection)


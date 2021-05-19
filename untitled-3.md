# Android卡顿检测及优化\_有图有真相-CSDN博客

### 前言

之前在项目中做过一些Android卡顿以及性能优化的工作，但是一直没时间总结，趁着这段时间把这部分总结一下。

### 卡顿

在应用开发中如果留意到log的话有时候可能会发下下面的log信息：

```text
I/Choreographer(1200): Skipped 60 frames!  The application may be doing too much work on its main thread.
```

在大部分Android平台的设备上，Android系统是16ms刷新一次，也就是一秒钟60帧。要达到这种刷新速度就要求在ui线程中处理的任务时间必须要小于16ms，如果ui线程中处理时间长，就会导致跳过帧的渲染，也就是导致界面看起来不流畅，卡顿。如果用户点击事件5s中没反应就会导致ANR。

### 帧率

即 Frame Rate，单位 fps，是指 gpu 生成帧的速率，60fps，Android中更帧率相关的类是SurfaceFlinger。

**SurfaceFlinger**  
 surfaceflinger作用是接受多个来源的图形显示数据，将他们合成，然后发送到显示设备。比如打开应用，常见的有三层显示，顶部的statusbar底部或者侧面的导航栏以及应用的界面，每个层是单独更新和渲染，这些界面都是有surfaceflinger合成一个刷新到硬件显示。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200617103126541.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)  
 在显示过程中使用到了bufferqueue，surfaceflinger作为consumer方，比如windowmanager管理的surface作为生产方产生页面，交由surfaceflinger进行合成。

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200617101750496.png)  
 **VSync**

Android系统每隔16ms发出VSYNC信号，触发对UI进行渲染，VSync是Vertical Synchronization\(垂直同步\)的缩写，是一种在PC上很早就广泛使用的技术，可以简单的把它认为是一种定时中断。而在Android 4.1\(JB\)中已经开始引入VSync机制，用来同步渲染，让UI和SurfaceFlinger可以按硬件产生的VSync节奏进行工作。

安卓系统中有 2 种 VSync 信号：  
 1、屏幕产生的硬件 VSync： 硬件 VSync 是一个脉冲信号，起到开关或触发某种操作的作用。  
 2、由 SurfaceFlinger 将其转成的软件 Vsync 信号：经由 Binder 传递给 Choreographer。

除了Vsync的机制，Android还使用了多级缓冲的手段以优化UI流程度，例如双缓冲\(A+B\)，在显示buffer A的数据时，CPU/GPU就开始在buffer B中准备下一帧数据：但是不能保证每一帧CPU、GPU都运行状态良好，可能由于资源抢占等性能问题导致某一帧GPU掉链子，vsync信号到来时buffer B的数据还没准备好，而此时Display又在显示buffer A的数据，导致后面CPU/GPU没有新的buffer着手准备数据，导致卡顿（jank）。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200619101533702.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)

## 卡顿原因

从系统层面上看主要以下几个方面的原因会导致卡顿：  
 **1. SurfaceFlinger 主线程耗时**

SurfaceFlinger 负责 Surface 的合成 , 一旦 SurfaceFlinger 主线程调用超时 , 就会产生掉帧 .  
 SurfaceFlinger 主线程耗时会也会导致 hwc service 和 crtc 不能及时完成, 也会阻塞应用的 binder 调用, 如 dequeueBuffer \ queueBuffer 等.

**2. 后台活动进程太多导致系统繁忙**

后台进程活动太多,会导致系统非常繁忙, cpu \ io \ memory 等资源都会被占用, 这时候很容易出现卡顿问题 , 这也是系统这边经常会碰到的问题。  
 **dumpsys cpuinfo 可以查看一段时间内 cpu 的使用情况：**  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200619101827920.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)  
 **3.主线程调度不到 , 处于 Runnable 状态**

当线程为 Runnable 状态的时候 , 调度器如果迟迟不能对齐进行调度 , 那么就会产生长时间的 Runnable 线程状态 , 导致错过 Vsync 而产生流畅性问题。

**4、System 锁**

system\_server 的 AMS 锁和 WMS 锁 , 在系统异常的情况下 , 会变得非常严重 , 如下图所示 , 许多系统的关键任务都被阻塞 , 等待锁的释放 , 这时候如果有 App 发来的 Binder 请求带锁 , 那么也会进入等待状态 , 这时候 App 就会产生性能问题 ; 如果此时做 Window 动画 , 那么 system\_server 的这些锁也会导致窗口动画卡顿  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200619102031218.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)  
 **5、Layer过多导致 SurfaceFlinger Layer Compute 耗时**

Android P 修改了 Layer 的计算方法 , 把这部分放到了 SurfaceFlinger 主线程去执行, 如果后台 Layer 过多, 就会导致 SurfaceFlinger 在执行 rebuildLayerStacks 的时候耗时 , 导致 SurfaceFlinger 主线程执行时间过长。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200619102324220.png)  
 从应用层来看以下会导致卡顿：

**1、主线程执行时间长**  
 主线程执行 Input \ Animation \ Measure \ Layout \ Draw \ decodeBitmap 等操作超时都会导致卡顿 。

* 1、Measure \ Layout 耗时\超时

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200619103556294.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)

* 2、draw耗时

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200619103632673.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)

* 3、Animation回调耗时

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200619103717321.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)

* 4、View 初始化耗时

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200619103810150.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)

* 5、List Item 初始化耗时

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200619103839118.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)

* 6、主线程操作数据库

**2、主线程 Binder 耗时**

Activity resume 的时候, 与 AMS 通信要持有 AMS 锁, 这时候如果碰到后台比较繁忙的时候, 等锁操作就会比较耗时, 导致部分场景因为这个卡顿, 比如多任务手势操作。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/2020061910395626.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)

**3、WebView 性能不足**

应用里面涉及到 WebView 的时候, 如果页面比较复杂, WebView 的性能就会比较差, 从而造成卡顿  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200619104046115.png)

**4、帧率与刷新率不匹配**

如果屏幕帧率和系统的 fps 不相符 , 那么有可能会导致画面不是那么顺畅. 比如使用 90 Hz 的屏幕搭配 60 fps 的动画。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200619104123839.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)

## 卡顿检测

卡顿检测可以使用以下多种方法同时进行：  
 **1、使用dumpsys gfxinfo**  
 **2、使用Systrace获取相关信息**  
 **3、使用LayoutInspect 检测布局层次**  
 **4、使用BlockCanary**  
 **5、利用Choreographer。**  
 **6、使用严格模式（StrictMode ）。**

### 1、使用dumpsys gfxinfo

在开发过程中发现有卡顿发生时可以使用下面的命令来获取卡顿相关的信息：

```text
adb shell dumpsys gfxinfo [PACKAGE_NAME]
```

输入这个命令后可能会打印下面的信息：

```text
Applications Graphics Acceleration Info:
Uptime: 102809662 Realtime: 196891968
** Graphics info for pid 31148 [com.android.settings] **
Stats since: 524615985046231ns
Total frames rendered: 8325
Janky frames: 729 (8.76%)
90th percentile: 13ms
95th percentile: 20ms
99th percentile: 73ms
Number Missed Vsync: 294
Number High input latency: 47
Number Slow UI thread: 502
Number Slow bitmap uploads: 44
Number Slow issue draw commands: 135
```

上面参数说明：

**Graphics info for pid 31148 \[com.android.settings\]**: 表明当前dump的为设置界面的帧信息，pid为31148  
 **Total frames rendered**: 8325 本次dump搜集了8325帧的信息

**Janky frames** :729 \(8.76%\)出现卡顿的帧数有729帧，占8.76%

**Number Missed Vsync**: 294 垂直同步失败的帧

**Number Slow UI thread**: 502 因UI线程上的工作导致超时的帧数

**Number Slow bitmap uploads**: 44 因bitmap的加载耗时的帧数

**Number Slow issue draw commands**: 135 因绘制导致耗时的帧数

### 2、使用systrace

上面使用的dumpsys是能发现问题或者判断问题的严重性，但无法定位真正的原因。如果要定位原因，应当配合systrace工具使用。

**systrace使用**

Systrace可以帮助分析应用是如何设备上运行起来的，它将系统和应用程序线程集中在一个共同的时间轴上，分析systrace的第一步需要在程序运行的时间段中抓取trace log，在抓取到的trace文件中，包含了这段时间中想要的关键信息，交互情况。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/2020061911445373.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)  
 图1显示的是当一个app在滑动时出现了卡顿的现象，默认的界面下，横轴是时间，纵向为trace event，trace event 先按进程分组，然后再按线程分组.从上到下的信息分别为Kernel，SurfaceFlinger，应用包名。通过配置trace的分类，可以根据配置情况记录每个应用程序的所有线程信息以及trace event的层次结构信息。

**Android studio中使用systrace**

1、在android设备的 设置 – 开发者选项 – 监控 – 开启traces。  
 2、选择要追中的类别，并且点击确定。

完成以上配置后，开始抓trace文件

```text
$ python systrace.py --cpu-freq --cpu-load --time=10 -o mytracefile.html
```

**分析trace文件**  
 抓到trace.html文件后，通过web浏览器打开

检查Frames  
 每个应用程序都有一排代表渲染帧的圆圈，通常为绿色，如果绘制的时间超过16.6毫秒则显示黄色或红色。通过“W”键查看帧。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200619114729357.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)  
 **trace应用程序代码**  
 在framework中的trace marker并没有覆盖到所有代码，因此有些时候需要自己去定义trace marker。在Android4.3之后，可以通过Trace类在代码中添加标记，这样将能够看到在指定时间内应用的线程在做哪些工作，当然，trace 的begin和end操作也会增加一些额外的开销，但都只有几微秒左右。  
 通过下面的例子来说明Trace类的 用法。

```text
public class MyAdapter extends RecyclerView.Adapter<MyViewHolder> {

    ...

    @Override
    public MyViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        Trace.beginSection("MyAdapter.onCreateViewHolder");
        MyViewHolder myViewHolder;
        try {
            myViewHolder = MyViewHolder.newInstance(parent);
        } finally {
            Trace.endSection();
        }
        return myViewHolder;
    }

   @Override
    public void onBindViewHolder(MyViewHolder holder, int position) {
        Trace.beginSection("MyAdapter.onBindViewHolder");
        try {
            try {
                Trace.beginSection("MyAdapter.queryDatabase");
                RowItem rowItem = queryDatabase(position);
                mDataset.add(rowItem);
            } finally {
                Trace.endSection();
            }
            holder.bind(mDataset.get(position));
        } finally {
            Trace.endSection();
        }
    }

…

}
```

### 3 、使用BlockCanary

BlockCanary是国内开发者MarkZhai开发的一套性能监控组件，它对主线程操作进行了完全透明的监控，并能输出有效的信息，帮助开发分析、定位到问题所在，迅速优化应用。  
 其特点有：  
 **1、非侵入式，简单的两行就打开监控，不需要到处打点，破坏代码优雅性。**  
 **2、精准，输出的信息可以帮助定位到问题所在（精确到行），不需要像Logcat一样，慢慢去找。**  
 **3、目前包括了核心监控输出文件，以及UI显示卡顿信息功能**

**BlockCanary基本原理**

android应用程序只有一个主线程ActivityThread，这个主线程会创建一个Looper\(Looper.prepare\)，而Looper又会关联一个MessageQueue，主线程Looper会在应用的生命周期内不断轮询\(Looper.loop\)，从MessageQueue取出Message 更新UI。

```text
public static void loop() {
    ...
    for (;;) {
        ...
        
        Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }
        msg.target.dispatchMessage(msg);
        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
        ...
    }
}
```

BlockCanary主要是检测`msg.target.dispatchMessage(msg);`之前的`>>>>> Dispatching to` 和之后的`<<<<< Finished to`的间隔时间。  
 应用发生卡顿，一定是在dispatchMessage中执行了耗时操作。通过给主线程的Looper设置一个Printer，打点统计dispatchMessage方法执行的时间，如果超出阀值，表示发生卡顿，则dump出各种信息，提供开发者分析性能瓶颈。

### 4、使用Choreographer

Android 主线程运行的本质，其实就是 Message 的处理过程，我们的各种操作，包括每一帧的渲染操作 ，都是通过 Message 的形式发给主线程的 MessageQueue ，MessageQueue 处理完消息继续等下一个消息。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200619115744338.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)  
 Choreographer 的引入，主要是配合 Vsync ，给上层 App 的渲染提供一个稳定的 Message 处理的时机，也就是 Vsync 到来的时候 ，系统通过对 Vsync 信号周期的调整，来控制每一帧绘制操作的时机. 目前大部分手机都是 60Hz 的刷新率，也就是 16.6ms 刷新一次，系统为了配合屏幕的刷新频率，将 Vsync 的周期也设置为 16.6 ms，每个 16.6 ms ， Vsync 信号唤醒 Choreographer 来做 App 的绘制操作 ，这就是引入 Choreographer 的主要作用。

**Choreographer 两个主要作用**

1、承上：负责接收和处理 App 的各种更新消息和回调，等到 Vsync 到来的时候统一处理。比如集中处理 Input\(主要是 Input 事件的处理\) 、Animation\(动画相关\)、Traversal\(包括 measure、layout、draw 等操作\) ，判断卡顿掉帧情况，记录 CallBack 耗时等。

2、启下：负责请求和接收 Vsync 信号。接收 Vsync 事件回调\(通过 FrameDisplayEventReceiver.onVsync \)；请求 Vsync\(FrameDisplayEventReceiver.scheduleVsync\) .

**使用Choreographer 计算帧率**

Choreographer 处理绘制的逻辑核心在 Choreographer.doFrame 函数中，从下图可以看到，FrameDisplayEventReceiver.onVsync post 了自己，其 run 方法直接调用了 doFrame 开始一帧的逻辑处理：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200619120044126.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)  
 Choreographer周期性的在UI重绘时候触发，在代码中记录上一次和下一次绘制的时间间隔，如果超过16ms，就意味着一次UI线程重绘的“丢帧”。丢帧的数量为间隔时间除以16，如果超过3，就开始有卡顿的感知。  
 使用Choreographer检测帧的代码如下：

```text
public class MyFrameCallback implements Choreographer.FrameCallback {
        private String TAG = "性能检测";
        private long lastTime = 0;

        @Override
        public void doFrame(long frameTimeNanos) {
            if (lastTime == 0) {
                
                lastTime = frameTimeNanos;
            } else {
                long times = (frameTimeNanos - lastTime) / 1000000;
                int frames = (int) (times / 16);

                if (times > 16) {
                    Log.w(TAG, "UI线程超时(超过16ms):" + times + "ms" + " , 丢帧:" + frames);
                }

                lastTime = frameTimeNanos;
            }

            Choreographer.getInstance().postFrameCallback(mFrameCallback);
        }
    }
```

### 卡顿优化

由上面的分析可知**对象分配**、**垃圾回收**\(GC\)、**线程调度**以及**Binder调用** 是Android系统中常见的卡顿原因，因此卡顿优化主要以下几种方法，更多的要结合具体的应用来进行：

**1、布局优化**

* 通过减少冗余或者嵌套布局来降低视图层次结构。比如使用约束布局代替线性布局和相对布局。
* 用 ViewStub 替代在启动过程中不需要显示的 UI 控件。
* 使用自定义 View 替代复杂的 View 叠加。
* 
**2、减少主线程耗时操作**

* 主线程中不要直接操作数据库，数据库的操作应该放在数据库线程中完成。
* sharepreference尽量使用apply，少使用commit，可以使用MMKV框架来代替sharepreference。
* 网络请求回来的数据解析尽量放在子线程中，不要在主线程中进行复制的数据解析操作。
* 不要在activity的onResume和onCreate中进行耗时操作，比如大量的计算等。

**3、减少过度绘制**  
 过度绘制是同一个像素点上被多次绘制，减少过度绘制一般减少布局背景叠加等方式，如下图所示右边是过度绘制的图片。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200619153239362.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)  
 **4、列表优化**

* RecyclerView使用优化，使用DiffUtil和notifyItemDataSetChanged进行局部更新等。

**5、对象分配和回收优化**

自从Android引入 ART 并且在Android 5.0上成为默认的运行时之后，对象分配和垃圾回收（GC）造成的卡顿已经显著降低了，但是由于对象分配和GC有额外的开销，它依然又可能使线程负载过重。 在一个调用不频繁的地方（比如按钮点击）分配对象是没有问题的，但如果在在一个被频繁调用的紧密的循环里，就需要避免对象分配来降低GC的压力。

* 减少小对象的频繁分配和回收操作。

### 参考文献

1、[https://source.android.google.cn/devices/graphics/implement-vsync](https://source.android.google.cn/devices/graphics/)  
 2、[https://www.bradcypert.com/what-is-androids-surfaceflinger/](https://www.bradcypert.com/what-is-androids-surfaceflinger/)  
 3、[https://ashishb.net/tech/demystifying-android-rendering/](https://ashishb.net/tech/demystifying-android-rendering/)  
 4、[https://devblogs.microsoft.com/xamarin/tips-for-creating-a-smooth-and-fluid-android-ui/](https://devblogs.microsoft.com/xamarin/tips-for-creating-a-smooth-and-fluid-android-ui/)  
 5、[https://developer.android.com/training/testing/performance](https://developer.android.com/training/testing/performance)  
 6、[https://www.androidperformance.com/2019/09/05/Android-Jank-Due-To-System/](https://www.androidperformance.com/2019/09/05/Android-Jank-Due-To-System/)  
 7、[https://developer.android.com/tools/help/systrace.html](https://developer.android.com/tools/help/systrace.html)  
 8、[https://zhuanlan.zhihu.com/p/87954949](https://zhuanlan.zhihu.com/p/87954949)


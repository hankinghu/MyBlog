# Android中Handler问题汇总\_有图有真相-CSDN博客

## 前言

handler机制几乎是Android面试时必问的问题，虽然看过很多次handler源码，但是有些面试官问的问题却不一定能够回答出来，趁着机会总结一下面试中所覆盖的handler知识点。

## **1、讲讲 Handler 的底层实现原理？**

下面的这幅图很完整的表现了整个handler机制。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200404111049684.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70#pic_center)  
 要理解handler的实现原理，其实最重要的是理解Looper的实现原理，Looper才是实现handler机制的核心。任何一个handler在使用sendMessage或者post时候，都是先构造一个Message，并把自己放到message中，然后把Message放到对应的Looper的MessageQueue，Looper通过控制MessageQueue来获取message执行其中的handler或者runnable。  
 要在当前线程中执行handler指定操作，必须要先看当前线程中有没有looper，如果有looper，handler就会通过sendMessage，或者post先构造一个message，然后把message放到当前线程的looper中，looper会在当前线程中循环取出message执行，如果没有looper，就要通过looper.prepare\(\)方法在当前线程中构建一个looper，然后主动执行looper.loop\(\)来实现循环。

梳理一下其实最简单的就下面四条：

1、每一个线程中最多只有一个Looper，通过ThreadLocal来保存，Looper中有Message队列，保存handler并且执行handler发送的message。

2、在线程中通过Looper.prepare\(\)来创建Looper，并且通过ThreadLocal来保存Looper，每一个线程中只能调用一次Looper.prepare\(\)，也就是说一个线程中最多只有一个Looper，这样可以保证线程中Looper的唯一性。

3、handler中执行sendMessage或者post操作，这些操作执行的线程是handler中Looper所在的线程，和handler在哪里创建没关系，和Handler中的Looper在那创建有关系。

4、一个线程中只能有一个Looper，但是一个Looper可以对应多个handler，在同一个Looper中的消息都在同一条线程中执行。

## **2、Handler机制，sendMessage和post（Runnable）的区别？**

要看sendMessage和post区别，需要从源码来看，下面是几种使用handler的方式，先看下这些方式，然后再从源码分析有什么区别。  
 **例1、 主线程中使用handler**

```text

        Handler mHandler = new Handler(new Handler.Callback() {
            @Override
            public boolean handleMessage(@NonNull Message msg) {
                if (msg.what == 1) {
                    
                }
                return false;
            }
        });
        Message msg = Message.obtain();
        msg.what = 1;
        mHandler.sendMessage(msg);
```

上面是在主线程中使用handler，因为在Android中系统已经在主线程中生成了Looper，所以不需要自己来进行looper的生成。如果上面的代码在子线程中执行，就会报

```text
Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()
```

如果想着子线程中处理handler的操作，就要必须要自己生成Looper了。

**例2 、子线程中使用handler**

```text
        Thread thread=new Thread(new Runnable() {
            @Override
            public void run() {
                Looper.prepare();
                Handler handler=new Handler();
                handler.post(new Runnable() {
                    @Override
                    public void run() {
                        
                    }
                });
                Looper.loop();
            }
        });
```

上面在Thread中使用handler，先执行Looper.prepare方法，来在当前线程中生成一个Looper对象并保存在当前线程的ThreadLocal中。  
 看下Looper.prepare\(\)中的源码：

```text

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }

```

可以看到prepare方法中会先从sThreadLocal中取如果之前已经生成过Looper就会报错，否则就会生成一个新的Looper并且保存在线程的ThreadLocal中，这样可以确保每一个线程中只能有一个唯一的Looper。

另外：由于Looper中拥有当前线程的引用，所以有时候可以用Looper的这种特点来判断当前线程是不是主线程。

```text
    @RequiresApi(api = Build.VERSION_CODES.KITKAT)
    boolean isMainThread() {
        return Objects.requireNonNull(Looper.myLooper()).getThread() == Looper.getMainLooper().getThread();
    }
```

**sendMessage vs post**

**先来看看sendMessage的代码调用链：**  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200404121026656.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70#pic_center)  
 enqueueMessage源码如下：

```text
    private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
            long uptimeMillis) {
        msg.target = this;
        msg.workSourceUid = ThreadLocalWorkSource.getUid();
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

enqueueMessage的代码处理很简单，msg.target = this;就是把当前的handler对象给message.target。然后再讲message进入到队列中。

**post代码调用链：**  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200404121818980.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70#pic_center)  
 调用post时候会先调用getPostMessage生成一个Message，后面和sendMessage的流程一样。下面看下getPostMessage方法的源码：

```text
    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
```

可以看到getPostMessage中会先生成一个Messgae，并且把runnable赋值给message的callback.消息都放到MessageQueue中后，看下Looper是如何处理的。

```text
    for (;;) {
        Message msg = queue.next(); 
        if (msg == null) {
            return;
        }
        msg.target.dispatchMessage(msg);
    }
```

Looper中会遍历message列表，当message不为null时调用msg.target.dispatchMessage\(msg\)方法。看下message结构：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200404132758792.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70#pic_center)  
 也就是说msg.target.dispatchMessage方法其实就是调用的Handler中的dispatchMessage方法，下面看下dispatchMessage方法的源码：

```text
    public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }

 private static void handleCallback(Message message) {
        message.callback.run();
    }
```

因为调用post方法时生成的message.callback=runnable,所以dispatchMessage方法中会直接调用 message.callback.run\(\);也就是说直接执行post中的runnable方法。  
 而sendMessage中如果mCallback不为null就会调用mCallback.handleMessage\(msg\)方法，否则会直接调用handleMessage方法。

**总结**  
 **post方法和handleMessage方法的不同在于，post的runnable会直接在callback中调用run方法执行，而sendMessage方法要用户主动重写mCallback或者handleMessage方法来处理。**

## **3、Looper会一直消耗系统资源吗？**

首先给出结论，Looper不会一直消耗系统资源，当Looper的MessageQueue中没有消息时，或者定时消息没到执行时间时，当前持有Looper的线程就会进入阻塞状态。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200404155135139.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70#pic_center)  
 下面看下looper所在的线程是如何进入阻塞状态的。其实handler机制中并不只有Java层的代码，还有native层的代码，如下图：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20200404194717230.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70#pic_center)  
 looper阻塞肯定跟消息出队有关，因此看下消息出队的代码。  
 **消息出队**

```text
   Message next() {
        
        
        
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            nativePollOnce(ptr, nextPollTimeoutMillis);
            
            
           	if(hasNoMessage)
           	{
           	nextPollTimeoutMillis =-1；
           	}
        }
    }
```

上面的消息出队方法被简写了，主要看下面这段，没有消息的时候nextPollTimeoutMillis=-1；

```text
 	if(hasNoMessage)
           	{
           	nextPollTimeoutMillis =-1；
           	}
```

看for循环里面这个字段所其的作用：

```text
 if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
  nativePollOnce(ptr, nextPollTimeoutMillis);
```

Binder.flushPendingCommands\(\);这个方法的作用可以看源码里面给出的解释：

```text
    
```

也就是说在用户线程要进入阻塞之前跟内核线程发送消息，防止用户线程长时间的持有某个对象。再看看下面这个方法：  
 nativePollOnce\(ptr, nextPollTimeoutMillis\);是阻塞操作，其中nextPollTimeoutMillis代表下一个消息到来前，还需要等待的时长；当nextPollTimeoutMillis = -1时，表示消息队列中无消息，会一直等待下去。  
 当处于空闲时，往往会执行IdleHandler中的方法。当nativePollOnce\(\)返回后，next\(\)从mMessages中提取一个消息。

当消息队列中没有消息的时候looper肯定是被消息入队唤醒的。

**消息入队**

```text
boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                
                
                
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; 
                prev.next = msg;
            }

            
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```

上面可以看到消息入队之后会有一个

```text
  if (needWake) {
                nativeWake(mPtr);
            }
```

方法，调用这个方法就可以唤醒线程了。另外消息入队的时候是根据消息的delay时间来在链表中排序的，delay时间长的排在后面，时间短的排在前面。如果时间相同那么按插入时间先后来排，插入时间早的在前面，插入时间晚的在后面。

## **4、android的Handle机制，Looper关系，主线程的Handler是怎么判断收到的消息是哪个Handler传来的？**

Looper是如何判断Message是从哪个handler传来的呢？其实很简单，在1中分析过，handler在sendMessage的时候会构建一个Message对象，并且把自己放在Message的target里面，这样的话Looper就可以根据Message中的target来判断当前的消息是哪个handler传来的。

## 5、Handler机制流程、Looper中延迟消息谁来唤醒Looper？

从3中知道在消息出队的for循环队列中会调用到下面的方法。

```text
nativePollOnce(ptr, nextPollTimeoutMillis);
```

如果是延时消息，会在被阻塞nextPollTimeoutMillis时间后被叫醒，nextPollTimeoutMillis就是消息要执行的时间和当前的时间差。

## **6、Handler是如何引起内存泄漏的？如何解决？**

在子线程中，如果手动为其创建Looper，那么在所有的事情完成以后应该调用quit方法来终止消息循环，否则这个子线程就会一直处于等待的状态，而如果退出Looper以后，这个线程就会立刻终止，因此建议不需要的时候终止Looper。

```text
Looper.myLooper().quit()
```

那么，如果在Handler的handleMessage方法中（或者是run方法）处理消息，如果这个是一个延时消息，会一直保存在主线程的消息队列里，并且会影响系统对Activity的回收，造成内存泄露。

具体可以参考Handler内存泄漏分析及解决

**总结一下，解决Handler内存泄露主要2点**

1 、有延时消息，要在Activity销毁的时候移除Messages

```text

public final void removeCallbacks(Runnable r) {
    mQueue.removeMessages(this, r, null);
}

public final void removeCallbacks(Runnable r, Object token) {
    mQueue.removeMessages(this, r, token);
}

public final void removeMessages(int what) {
    mQueue.removeMessages(this, what, null);
}

public final void removeMessages(int what, Object object) {
    mQueue.removeMessages(this, what, object);
}

public final void removeCallbacksAndMessages(Object token) {
    mQueue.removeCallbacksAndMessages(this, token);
}
```

2、 匿名内部类导致的泄露改为匿名静态内部类，并且对上下文或者Activity使用弱引用。

## **7、handler机制中如何确保Looper的唯一性？**

Looper是保存在线程的ThreadLocal里面的，使用Handler的时候要调用Looper.prepare\(\)来创建一个Looper并放在当前的线程的ThreadLocal里面。

```text
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```

可以看到，如果多次调用prepare的时候就会报Only one Looper may be created per thread，所以这样就可以保证一个线程中只有唯一的一个Looper。

## **8、Handler 是如何能够线程切换，发送Message的？**

handler的执行跟创建handler的线程无关，跟创建looper的线程相关，加入在子线程中创建一个Handler，但是Handler相关的Looper是主线程的，这样，如果handler执行post一个runnable，或者sendMessage，最终的handle Message都是在主线程中执行的。

```text
        Thread thread=new Thread(new Runnable() {
            @Override
            public void run() {
                Looper.prepare();
                Handler handler=new Handler(getMainLooper());
                handler.post(new Runnable() {
                    @Override
                    public void run() {
                        Toast.makeText(MainActivity.this,"hello,world",Toast.LENGTH_LONG).show();
                    }
                });
                Looper.loop();
            }
        });
        thread.start();
```

## 参考文献

1、[https://www.jianshu.com/p/ea7beaeeee16](https://www.jianshu.com/p/ea7beaeeee16)


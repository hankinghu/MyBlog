# Android 高级进阶之overdraw分析及解决\_有图有真相-CSDN博客

## 前言

最近在看Android中性能优化的，其中提到了LinearLayout会引起overdraw,但是并没有具体的分析原因，我自己查找了一些资料从LinearLayout的绘制等方面来说明为什么使用LinearLayout会引起overdraw和哪些情况下使用LinearLayout会引起overdraw。希望大家看完之后对view的绘制和测量过程更加了解。

## 什么是overdraw

Android中在屏幕上绘制一个像素会花一定的时间，如果在屏幕的同一个位置多次绘制就会花大量的时间，多次在屏幕上同一位置绘制的情况就成为overdraw。overdraw会非常影响应用的性能。一个高效的布局要做到两点：  
 **1、减少overdraw。**

**2、简化布局结构。**

看下一个overdraw的例子  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOS82LzEzLzE2YjRlYTlkNmYwOWMwNzg)  
 上面的六张牌的重合部分被多次绘制引发了overdraw，

## 检测overdraw

可以通过手机的设置来直观的查看应用的overdraw情况。  
 **1、打开设置，打开开发者选项。**  
 **2、选择调试GPU过度渲染。**  
 **3、打开显示过度渲染区域。**  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOS82LzEzLzE2YjRlYTlkNzRiN2E1NmQ)  
 上面左边是正常的未打开，调试GPU过度渲染的情形，右边是打开GPU过度渲染检测之后的。  
 上面右图中不同的颜色对应不同的overdraw的次数。对应关系如下：

![&#x4E0D;&#x540C;&#x989C;&#x8272;&#x5BF9;&#x5E94;&#x91CD;&#x7ED8;&#x6B21;&#x6570;&#x5173;&#x7CFB;](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOS82LzEzLzE2YjRlYTlkNzVjOWM4MjA)  
 如下三幅图，最右边的是基本上没有overdraw或者只有一次overdraw的，而中间的红色区域很多，大部分是有三次及以上的overdraw。overdraw的次数越多越影响性能，所以中间的这种是不提倡的。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOS82LzEzLzE2YjRlYTlkNzcxYzlkYjc)

## 减少overdraw的方法

**1、减少不必要的background**

因为有background的时候会先绘制一遍background然后再在background的基础上面绘制其他元素，所以会增加一次overdraw。  
 所有用户看不到的background都应该删除掉。  
 如下面的这个例子：

**删除background之前**

```text
<ImageView
   android:layout_width="match_parent"
   android:layout_height="match_parent"
   android:src="@drawable/beach"
   android:background="@android:color/white">
</ImageView>
```

**删除background之后**

```text
<ImageView
   android:layout_width="match_parent"
   android:layout_height="match_parent"
   android:src="@drawable/beach" >
</ImageView>
```

因为imageView的background在有src的时候根本就不会被用户观察到，所以应该删除掉。

**2、统一整个app的background颜色**

可以通过设置整个app的统一背景色来防止不同的activity设置不同的背景色而导致多个background。可以在AndroidManifest.xml添加：

```text
android:theme="@android:style/Theme.Light"
```

或者想要的背景色。

## 2、使用clip减少渲染区域

clip具有裁剪功能，在自定义view的时候，渲染的图像可能和用户观察到的图像不一样，自定义view的ondraw方法里面不仅仅只渲染用户可见的图像而且会渲染到被遮挡的图像，使用clip方法裁剪出用户可见的区域，这样可以减少overdraw。

**使用clipRect\(\)方法**

在自定义view中使用 Canvas.clipRect\(\)方法可以有效的减少overdraw。这个方法可以为自定义的view提供一个rectangle 区域，并且只有在这个区域中的内容才会被绘制。上面的扑克牌就可以使用Canvas.clipRect\(\)来减少绘制。如下：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOS82LzEzLzE2YjRlYTlkOTgyZDRlZGY)  
 下面通过一个自定义的view来进行一个对比：

**未使用clipRect\(\)的自定义view**

```text

public class MyView extends View {
    private Paint mPaint;
    private Bitmap mBitmap;
    private int mPadding = 0;

    public MyView(Context context) {
        super(context);
    }

    public MyView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        mPaint = new Paint();
        mBitmap = BitmapFactory.decodeResource(getResources(), R.drawable.over_draw);
    }

    public MyView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        mPaint = new Paint();
        mBitmap = BitmapFactory.decodeResource(getResources(), R.drawable.over_draw);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        for (int i = 0; i < 5; i++) {
            canvas.save();
            canvas.drawBitmap(mBitmap,mPadding,0,mPaint);
            canvas.restore();
            mPadding += 200;
        }
    }
}
```

在开启检测overdraw后的显示效果如下：

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOS82LzEzLzE2YjRlYTlkNzcwMmZiNzU)  
 上图从左到右可以由颜色看出分别进行了一次，两次，三次和多次的overdraw，下面看看使用clipRect\(\)后的显示效果。

**使用clipRect\(\)的自定义view**

```text

public class MyView extends View {
    private Paint mPaint;
    private Bitmap mBitmap;
    private int mPadding = 0;

    public MyView(Context context) {
        super(context);
    }

    public MyView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        mPaint = new Paint();
        mBitmap = BitmapFactory.decodeResource(getResources(), R.drawable.over_draw);
    }

    public MyView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        mPaint = new Paint();
        mBitmap = BitmapFactory.decodeResource(getResources(), R.drawable.over_draw);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        for (int i = 0; i < 5; i++) {
            canvas.save();
            Rect rect = new Rect(mPadding, 0,  200+mPadding, mBitmap.getWidth() + mPadding);
            canvas.clipRect(rect);
            canvas.drawBitmap(mBitmap,mPadding,0,mPaint);
            canvas.restore();
            mPadding += 200;
        }
    }
}
```

运行后的显示效果如下：

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOS82LzEzLzE2YjRlYTlkYjM4NmI3OGE)  
 可以看出从左到右所有的图片都只进行了一次overdraw。大大减少了渲染次数。  
 Canvas.clipRect\(\)该方法用于裁剪画布，也就是设置画布的显示区域 调用clipRect\(\)方法后，只会显示被裁剪的区域，之外的区域将不会显示 .

## 3、尽量少用透明（alpha ）效果

Alpha是图形界面开发中常用的特效，通常我们会使用以下代码来实现Alpha特效：

```text
view.setAlpha(0.5f);
    View.ALPHA.set(view, 0.5f);
    ObjectAnimator.ofFloat(view, "alpha", 0.5f).start();
    view.animate().alpha(0.5f).start();
    view.setAnimation(new AlphaAnimation(1.0f, 0.5f));
```

其效果都等同于：

```text
canvas.saveLayer(l, r, t, b, 127, Canvas.CLIP_TO_LAYER_SAVE_FLAG);
```

渲染带有透明度像素被称为：alpha rendering，alpha rendering会导致overdraw，因为系统会先渲染透明像素，然后渲染透明像素下面的view的像素，最后结合两者，从而产生透明度的效果，因此会导致overdraw。如下图：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOS82LzEzLzE2YjRlYTlkYTYwY2U3NWI)  
 上图左边是没有加透明度的像素，右边是在其上面加上一个透明度的蒙层，下面的图是加上透明度后最终产生的效果。  
 类似于透明动画，淡入，淡出等或者带有阴影的效果都会导致alpha rendering。因此会导致overdraw。下面看一个由于设置alpha引起性能问题的实例。

**Android Performance Case**  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOS82LzEzLzE2YjRlYTlkYWQ4MmE1Y2Y)  
 在开发者模式中打开GPU profiling 工具后，发现上图中右边存在明显的掉帧现象（底下红柱超过蓝线部分的），打开Tracer for OpenGL工具来检测后发现：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOS82LzEzLzE2YjRlYTlkYWRiZTBjNDQ)  
 是由viewpager上面滑动时标志当前位置和其他位置的白色小点引起的，这些白色的小点，设置了透明度，每次都会调用Canvas.saveLayer\(\)生成一个临时图层。正好满足了以下条件：

```text
getAlpha() returns a value < 1
onSetAlpha() returns false
getLayerType() returns LAYER_TYPE_NONE
hasOverlappingRendering() returns true
```

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOS82LzEzLzE2YjRlYTlkYjI4YzVlYjE)  
 上面的用红色区域标记出来的圆点，在每次滑动viewpager的时候都会动态调用setAlpha\(\)方法来改变颜色和透明度，因此引起了overdraw和掉帧。为了解决上面的问题，可以采用下面的任一种方法：

**1、Use a customizable “inactive” color instead of setting an opacity on the View**

**2、 Return false from hasOverlappingRendering\(\) and the framework will set the proper alpha on the Paint for you**

（注意：在android的View里有透明度的属性，当设置透明度setAlpha的时候，android里默认会把当前view绘制到offscreen buffer中，然后再显示出来。 这个offscreen buffer 可以理解为一个临时缓冲区，把当前View放进来并做透明度的转化，然后在显示到屏幕上。这个过程是消耗资源的，所以应该尽量避免这个过程。而当继承了hasOverlappingRendering\(\)方法返回false后，android会自动进行合理的优化，避免使用offscreen buffer。  
 ）

**3、Return true from onSetAlpha\(\) and set an alpha on the Paint used to draw the “gray” circles**

## 如何高效的使用alpha属性

上面已经说到了使用alpha属性的时候会导致overdraw，那么应该如何避免这些情况以减少overdraw呢？  
 下面分别对textview，imageview，和customview中使用到alpha情况进行说明：

**textview**  
 对于TextView我们通常需要文字透明效果，而不是View本身透明，所以，直接设置带有alpha值的TextColor是比较高效的方式。

```text
 
    textView.setAlpha(alpha);
 
 
    
    int newTextColor = (int) (0xFF * alpha) << 24 | baseTextColor & 0xFFFFFF;
    textView.setTextColor(newTextColor);
```

**ImageView**  
 同样的对于只具有src image的ImageView，直接调用setImageAlpha\(\)方法更为合理。

```text

    imageView.setAlpha(0.5f);

 
    
    
    imageView.setImageAlpha((int) alpha * 255);
```

**CustomView**  
 类似的，自定义控件时，应该直接去设置paint的alpha。

```text

    customView.setAlpha(alpha);

    
    paint.setAlpha((int) alpha * 255);
    canvas.draw*(..., paint);
```

## LinearLayout导致的overdraw

未完待续

## 参考文献

1、[https://google-developer-training.github.io/android-developer-advanced-course-concepts/unit-2-make-your-apps-fast-and-small/lesson-4-performance/4-1-c-rendering-and-layout/4-1-c-rendering-and-layout.html](https://google-developer-training.github.io/android-developer-advanced-course-concepts/unit-2-make-your-apps-fast-and-small/lesson-4-performance/4-1-c-rendering-and-layout/4-1-c-rendering-and-layout.html)  
 2、[https://sriramramani.wordpress.com/2015/05/06/custom-viewgroups/](https://sriramramani.wordpress.com/2015/05/06/custom-viewgroups/)  
 3、[https://helw.net/2016/01/27/on-linearlayout-measures/](https://helw.net/2016/01/27/on-linearlayout-measures/)  
 4、[https://www.cnblogs.com/tianzhijiexian/p/4644693.html](https://www.cnblogs.com/tianzhijiexian/p/4644693.html)  
 5、[https://www.androidperformance.com/2015/03/31/android-performance-case-study-follow-up/](https://www.androidperformance.com/2015/03/31/android-performance-case-study-follow-up/)  
 6、[http://www.curious-creature.com/2015/03/25/android-performance-case-study-follow-up/?utm\_source=Android+Weekly&utm\_campaign=0692ef161b-Android\_Weekly\_146&utm\_medium=email&utm\_term=0\_4eb677ad19-0692ef161b-337850757](http://www.curious-creature.com/2015/03/25/android-performance-case-study-follow-up/?utm_source=Android+Weekly&utm_campaign=0692ef161b-Android_Weekly_146&utm_medium=email&utm_term=0_4eb677ad19-0692ef161b-337850757)  
 7、[http://www.curious-creature.com/2012/12/01/android-performance-case-study/](http://www.curious-creature.com/2012/12/01/android-performance-case-study/)  
 8、[http://yangm90.github.io/android-Alpha/](http://yangm90.github.io/android-Alpha/)


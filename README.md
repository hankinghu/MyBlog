# Canvas加动画，实现火柴人跳绳效果\_有图有真相-CSDN博客

## 效果

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20210413104040373.gif#pic_center)

## 涉及到的知识

**1、canvas**  
 **2、path和二阶贝塞尔曲线**  
 **3、bitmap绘制**

## canvas

先引用google官方：

```text
The Canvas class holds the "draw" calls. To draw something, you need 4 basic components: A Bitmap to hold the pixels, a Canvas to host the draw calls (writing into the bitmap), a drawing primitive (e.g. Rect, Path, text, Bitmap), and a paint (to describe the colors and styles for the drawing).
```

canvas画布，Android中绘制时，需要四个基本要素：bitmap用于存放像素点，canvas来进行绘制，基础形状：矩形，path，text，bitmap等，paint用于控制绘制的颜色和样式，canvas相关的很重要的一点是，在Android中坐标的原点是从左上角开始。

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20210413105002224.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)

## Path

**1.moveTo**  
 moveTo表示将绘制点移动到某一个坐标处，该方法并不会进行绘制，主要是用来移动画笔。默认情况下起始坐标位于（0，0）点。

**2.lineTo**  
 lineTo表示绘制一条直线：

```text
 @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        Path path = new Path();
        path.lineTo(50,
                50);
        canvas.drawPath(path, paint);

```

**3.quadTo**  
 用于绘制二阶贝塞尔曲线，从上一个点开始，绘制二阶Bezier曲线\(x1,y1\)为控制点， \(x2,y2\)为终点如果之前没有调用过 moveTo\(\)，则默认从 \(0,0\)作为起点绘制。

```text
 
    public void quadTo(float x1, float y1, float x2, float y2) ；

    
    public void rQuadTo(float dx1, float dy1, float dx2, float dy2)
```

## bitmap绘制

**1、获取图片的bitmap**

```text
private val jumpBitmap1 = BitmapFactory.decodeResource(context.resources, R.drawable.jump_1)
```

**2、将bitmap绘制在画布上**

```text
   
    public void drawBitmap(@NonNull Bitmap bitmap, @Nullable Rect src, @NonNull Rect dst,
            @Nullable Paint paint) {
        super.drawBitmap(bitmap, src, dst, paint);
    }
```

其中 Rect src是图片的子域，通过设置src的大小可以截取图片的部分区域，dst指图片在画布上面的位置，一般情况下不裁剪图片的话，设置src为null就可以了。

## 实现

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/202104131848477.jpg#pic_center)  
 **1、跳绳实现**  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20210413193618801.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)  
 如上图，path的起点和中点在屏幕的中点两边，中点的点是控制贝塞尔曲线的点，这个点上下动就可以控制绘制出一条跳绳。要注意`ValueAnimator.ofInt(0, b.bottom / 2, b.bottom, b.bottom / 2, 0)`的值是从0到.bottom / 2，再到bottom，在到bottom / 2，最后回到0的，如果不是这样直接从（0-》bottom）的话，贝塞尔曲线的运动轨迹就不是连续的。  
 **2、跳动的火柴人**  
 跳动的火柴人由两种状态，由动画的值确定，根据动画的值来改变火柴人的位置，火柴人初始位置是在画布的正中间，根据动画的值，改变上下位置，同时更改bitmap。

## 代码

定义PathDrawable继承Drawable并且实现AnimatorUpdateListener接口

```text
internal class PathDrawable(context: Context) : Drawable(),
    AnimatorUpdateListener
    
```

定义动画，

```text
    mAnimator = ValueAnimator.ofInt(0, b.bottom / 2, b.bottom, b.bottom / 2, 0)
```

更新贝塞尔曲线和火柴人

```text
 override fun onAnimationUpdate(animator: ValueAnimator) {
        Log.d(TAG, "onAnimationUpdate animator${animator.animatedValue}")
        mPath.reset()
        val b: Rect = bounds
        mPath.moveTo(b.left.toFloat(), b.bottom.toFloat() / 2)
        mPath.quadTo(
            ((b.right - b.left) / 2).toFloat(),
            (animator.animatedValue as Int).toFloat(), b.right.toFloat(), b.bottom.toFloat() / 2
        )
        ++count
        if (count % 24 == 0) {

            currentBitmap = if (animator.animatedValue as Int > b.bottom / 2) {
                jumpBitmap1
            } else {
                jumpBitmap3
            }
            count = 0
        }
        destinationRect.set(
            b.left + (halfW(b) - imageW() / 2).toInt(),
            b.top + (halfH(b) - imageH() / 2).toInt() + (animator.animatedValue as Int)/8,
            b.left + (halfW(b) + imageW() / 2).toInt(),
            b.top + (halfH(b) + imageH() / 2).toInt() + (animator.animatedValue as Int)/8
        )
        invalidateSelf()
    }
```

## 参考

[https://stackoverflow.com/questions/18616035/how-to-animate-a-path-on-canvas-android](https://stackoverflow.com/questions/18616035/how-to-animate-a-path-on-canvas-android)  
 [https://developer.android.com/reference/android/graphics/drawable/Drawable](https://developer.android.com/reference/android/graphics/drawable/Drawable)  
 [https://developer.android.com/guide/topics/graphics/drawables](https://developer.android.com/guide/topics/graphics/drawables)  
 [https://developer.android.com/reference/android/animation/ValueAnimator](https://developer.android.com/reference/android/animation/ValueAnimator)  
 [https://stackoverflow.com/questions/13361231/android-draw-bitmap-within-canvas](https://stackoverflow.com/questions/13361231/android-draw-bitmap-within-canvas)


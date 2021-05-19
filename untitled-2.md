# android 算法可视化（1） --冒泡排序可视化实现\_有图有真相-CSDN博客

## 前言

以前写了很多算法相关的博客，每篇博客都会用word或者processing画上很多图，非常浪费时间，那时候就一直有考虑能不能使用程序来实现这种过程，不仅不用自己画那么图，而且编程实现可视化的话，还可以动态更清晰的表现算法的过程。于是查找了相关的资料和自己对算法的理解先实现一个冒泡排序的可视化，代码是Android的。

## 效果

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20210403233226862.gif#pic_center)

## 实现

要实现这个动画效果，实际上需要两个基本的模块组成：一个BubbleView用于绘制，一个control控制器，用于控制BubbleView的绘制。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20210404113121496.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70#pic_center)

**1、BubbleView的实现**  
 要实现上面的动画要定义一个自己的BubbleView继承View然后重写View的onDraw\(\)方法，这个view要包含以下三个部分：

1. 每一个数组元素的绘制
2. 遍历数组中当前元素时的绘制
3. 交换时的绘制

BubbleView中onDraw\(\)方法中具体流程如下：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20210404112619398.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70#pic_center)

1、每一个数组元素的绘制

```text
public class SortingVisualizer extends View {

    Paint paint;
    Paint textPaint;
    int[] array;
    int lineStrokeWidth = getDimensionInPixel(10);

    public SortingVisualizer(Context context) {
        super(context);
        initialise();
    }

    public SortingVisualizer(Context context, AttributeSet atrrs) {
        super(context, atrrs);
        initialise();
    }

    private void initialise() {
        paint = new Paint();
        paint.setColor(Color.DKGRAY);
        paint.setStyle(Paint.Style.FILL);
        paint.setStrokeWidth(lineStrokeWidth);
        textPaint = new TextPaint();
        textPaint.setColor(Color.BLACK);
        textPaint.setTextSize(getDimensionInPixelFromSP(15));
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        if (array != null) {
            int numberOfLines = array.length;
            float margin = (getWidth() - (30 * numberOfLines)) / (numberOfLines + 1);
            float xPos = margin + getDimensionInPixel(10);
            for (int i = 0; i < array.length; i++) {
                    canvas.drawLine(xPos, getHeight() - (float) ((array[i] / 10.0) * getHeight()), xPos, getHeight(), paint);
                canvas.drawText(String.valueOf(array[i]), xPos - lineStrokeWidth / 3, getHeight() - (float) ((array[i] / 10.0) * getHeight()) - 30, textPaint);
                xPos += margin + 30;
        }
    public void setData(int[] integers) {
        this.array = integers;
        invalidate();
    }
```

上面定义了两个画笔Paint，一个用于绘制元素的柱体，

```text
canvas.drawLine(xPos, getHeight() - (float) ((array[i] / 10.0) * getHeight()), xPos, getHeight(), paint);
xPos += margin + 30;
```

一个用于绘制柱体上面的text

```text
canvas.drawText(String.valueOf(array[i]), xPos - lineStrokeWidth / 3, getHeight() - (float) ((array[i] / 10.0) * getHeight()) - 30, textPaint);

```

2、 遍历数组中当前元素时的绘制

```text

    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        if (array != null) {
            int numberOfLines = array.length;

            float margin = (getWidth() - (30 * numberOfLines)) / (numberOfLines + 1);

            float xPos = margin + getDimensionInPixel(10);
            for (int i = 0; i < array.length; i++) {
                if (i == highlightPosition) {
                    canvas.drawLine(xPos, getHeight() - (float) ((array[i] / 10.0) * getHeight()), xPos, getHeight(), highlightPaintTrace);

                } 
                xPos += margin + 30;
            }
        }
    }
```

highlightPosition是外部排序时设置的值

3、交换时的绘制

```text
  protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        if (array != null) {
            int numberOfLines = array.length;

            float margin = (getWidth() - (30 * numberOfLines)) / (numberOfLines + 1);

            float xPos = margin + getDimensionInPixel(10);
            for (int i = 0; i < array.length; i++) {
                if (i == highlightPositionOne) {
                    canvas.drawLine(xPos, getHeight() - (float) ((array[i] / 10.0) * getHeight()), xPos, getHeight(), highlightPaintSwap);
                } 
            highlightPositionOne = -1;
            highlightPositionTwo = -1;
        }


    }
```

## control实现

control里面包括了冒泡排序算法，和控制BubbleView重绘，以及绘制的间隔等功能，

```text
   private void sort() {
        for (int i = 0; i < array.length; i++) {
            boolean swapped = false;
            for (int j = 0; j < array.length - 1 - i; j++) {
                highlightTrace(j);
                sleep();
                if (array[j] > array[j + 1]) {
                    highlightSwap(j, j + 1);
                    addLog("Swapping " + array[j] + " and " + array[j + 1]);
                    int temp = array[j];
                    array[j] = array[j + 1];
                    array[j + 1] = temp;
                    swapped = true;
                    sleep();
                }
            }
            if (!swapped) {
                break;
            }
            sleep();
        }
    }
```

```text
    public void highlightSwap(final int one, final int two) {
        activity.runOnUiThread(new Runnable() {
            @Override
            public void run() {
                bubbleView.highlightSwap(one, two);
            }
        });
    }

    public void highlightTrace(final int position) {
        activity.runOnUiThread(new Runnable() {
            @Override
            public void run() {
                bubbleView.highlightTrace(position);
            }
        });
    }
```

## 冒泡排序算法

```text
        for (int i = 0; i < array.length; i++) {
            boolean swapped = false;
            for (int j = 0; j < array.length - 1 - i; j++) {                
                if (array[j] > array[j + 1]) {
                    int temp = array[j];
                    array[j] = array[j + 1];
                    array[j + 1] = temp;
                    swapped = true;
                }
            }
            if (!swapped) {
                break;
            }
        }
```

## 参考

1、[https://developer.android.com/reference/android/graphics/Canvas](https://developer.android.com/reference/android/graphics/Canvas)


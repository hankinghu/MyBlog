# android实现音乐跳动效果\_有图有真相-CSDN博客

## 效果图

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20210408155627343.gif#pic_center)

## 实现

整体的流程图如下  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20210408172701715.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70#pic_center)  
 上面主要步骤分为3个  
 **1、计算宽度能放下多少列的音频块。**  
 **2、计算每一列中音频块的个数**  
 **3、绘制音频块**

**1、计算宽度能放下多少列的音频块。**  
 设置音频块的宽度为danceWidth，音频块横向之间的间距为danceGap，那么可以算出能放的列数：

```text

        val widthNum = (getAvailableWith() / (danceGap + danceWidth)).toInt()

  
    private fun getAvailableWith() = mCanvasWidth - paddingLeft - paddingRight
```

**2、计算每一列中音频块的个数**  
 在算出横向能放置多少音频块后，遍历横，然后绘制列中的音频块，列中的音频块的个数跟音频的高低相关，这里实现方式是通过Visualizer这个类然后获取到mRawAudioBytes数组，

```text
 mVisualizer.setDataCaptureListener(new Visualizer.OnDataCaptureListener() {
            @Override
            public void onWaveFormDataCapture(Visualizer visualizer, byte[] bytes,
                                              int samplingRate) {
                BaseVisualizer.this.mRawAudioBytes = bytes;
                invalidate();
            }

            @Override
            public void onFftDataCapture(Visualizer visualizer, byte[] bytes,
                                         int samplingRate) {
            }
        }, Visualizer.getMaxCaptureRate() / 2, true, false);
```

这里设置的获取的mRawAudioBytes数组的大小是128，数组的区间范围\[-128,127\],计算列的时候这里做了两个比较重要的操作，第一个是怎么把mRawAudioBytes数组的值与音频的个数做映射，第二个是怎么取mRawAudioBytes数组的值。

```text
 
        val widthNum = (getAvailableWith() / (danceGap + danceWidth)).toInt()
        Log.d(
            TAG,
            "widthNum $widthNum"
        )
        

        
        var lastDanceRight = paddingLeft.toFloat()
        if (widthNum > 0 && mRawAudioBytes != null && mRawAudioBytes.isNotEmpty())
            for (i in 0 until widthNum) {
                
                val num = (getAvailableHeight() / (danceHeight + danceGap)).toInt()
                val index = (mRawAudioBytes.size) * (i.toFloat() / widthNum)
                val b = (mRawAudioBytes[index.toInt()] + 128).toFloat() / 255f
                var heightNum =
                    (b * num).toInt()
                if (heightNum < miniNum) {
                    heightNum = miniNum
                }
                if (heightNum > maxNum) {
                    heightNum = maxNum
                }
                
                var lastHeight = mCanvasHeight - paddingStart.toFloat()
                Log.d(
                    TAG,
                    "heightNum $heightNum lastHeight $lastHeight lastDanceRight $lastDanceRight ${mRawAudioBytes[i]} $num $b $index"
                )
                lastHeight = drawItem(heightNum, lastDanceRight, lastHeight, canvas)
                lastDanceRight += danceWidth + danceGap
            }
```

上面做了两个映射，首先可能有0~n横，但是mRawAudioBytes大小是128，遍历横的时候对下标进行一个映射，保证获得的值是均匀的，

```text

val index = (mRawAudioBytes.size) * (i.toFloat() / widthNum)
```

第二个映射，是得到了代表音频大小的mRawAudioBytes数组，现在要把这里面的值跟列的高度做一个映射，值越大高度越高，音频块就越多。

```text
val num = (getAvailableHeight() / (danceHeight + danceGap)).toInt()
val b = (mRawAudioBytes[index.toInt()] + 128).toFloat() / 255f
var heightNum =(b * num).toInt()
```

上面是先得到列最多能展示多少音频块，再根据mRawAudioBytes的值来算出当前列展示多少个音频块。这一步也叫归一化，区间映射。

**3、绘制每一个音频块**

```text
    private fun drawItem(
        heightNum: Int,
        lastDanceRight: Float,
        lastHeight: Float,
        canvas: Canvas?
    ): Float {
        var lastHeight1 = lastHeight
        for (j in 0 until heightNum) {
            mDanceRect.set(
                lastDanceRight,
                lastHeight1 - danceHeight,
                lastDanceRight + danceWidth,
                lastHeight1
            )
            mPaint.shader = null
            if (j >= heightNum - shaderNum) {
                val backGradient = LinearGradient(
                    lastDanceRight,
                    lastHeight1 - danceHeight,
                    lastDanceRight + danceWidth,
                    lastHeight1,
                    intArrayOf(colorStart, colorCenter, colorEnd),
                    null,
                    Shader.TileMode.CLAMP
                )
                mPaint.shader = backGradient
            }
            canvas?.drawRoundRect(mDanceRect, 8f, 8f, mPaint)
            lastHeight1 -= (danceHeight + danceGap)
        }
        return lastHeight1
    }
```

就是根据高度来绘制rectangle，算出一列能绘制多少个音频块，每一个音频块是一个rectangle，然后绘制rectangle，为了效果更好，判断上面的音频块加上渐变。

## github地址

欢迎点赞收藏，后期会优化  
 [https://github.com/hankinghu/AudioVisulizer](https://github.com/hankinghu/AudioVisulizer)

## 使用方法

```text
<com.masoudss.lib.DanceView
              android:id="@+id/danceView"
              android:layout_width="320dp"
              android:layout_height="300dp"
              android:layout_gravity="center"
              app:color_center="@color/red"
              app:color_end="@color/white"
              app:color_start="@color/yellow"
              app:dance_color="@color/yellow"
              app:dance_corner_radius="2dp"
              app:dance_gap="2dp"
              app:max_dance_num="30"
              app:min_dance_num="2"
              app:shader_num="3" />
```

* shader\_num 顶部加渐变的个数
* color\_end 渐变尾部颜色
* color\_start 渐变开头颜色
* color\_center 渐变中间颜色
* min\_dance\_num 每一列中最少显示的个数
* max\_dance\_num 每一列中最大显示的个数
* dance\_gap 每一个音频格之间的间距


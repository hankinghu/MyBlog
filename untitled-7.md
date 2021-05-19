# android高级进阶之12条代码优化以及性能优化\_有图有真相-CSDN博客

从去年七月份（2018/7/13）入职到现在（2019/8/15）已经一年多了，这一年从一个菜鸟开始慢慢学习到了很多东西，记录一下在开发过程中遇到的代码优化和性能优化经验，方便让其他人少走弯路。

## 性能优化

### **1、装箱带来的内存消耗**

```text
Boolean isShow =new Boolean(true) ;  
```

上面的代码会带来如下问题：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20190815211106457.png)  
 上面的意思总结一下就是，采用装箱在java 5及以上是没必要的，采用装箱的方式构造一个对象会占用更多的内存，而使用比如说Boolean.TRUE的方式只是一个常量所以采用下面的方式更节约内存，正确的方式如下：

```text

Boolean isShow =Boolean.TRUE ;


Integer i= 2;

```

### **2、sharedPreferences使用commit带来的线程阻塞**

```text
                final SharedPreferences.Editor edit = settings.edit();
                edit.putBoolean(TIPS, true);
                edit.commit();
```

上面的代码会带来如下问题：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20190815213028529.png)  
 上面的代码如果在ui线程执行会带来ui线程的阻塞，可能会造成掉帧，原因是commit是在当前线程中执行写内存操作的并且commit执行完后会返回一个bool值来表示是否写成功，而apply会在异步线程里面写操作，执行完后不会返回是否写成功的值，其实大部分情况下并不需要知道sharedPreferences是否写成功所以可以用apply来替代commit。

### **3、selector中item位置错误**

错误

```text

<selector xmlns:android="http://schemas.android.com/apk/res/android">

    <item android:drawable="@drawable/normal" />
    <item android:drawable="@drawable/pressed" android:state_pressed="true"/>

selector>
```

上面的selector会导致如下问题  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20190816095748687.png)  
 selector中press的item属性永远不会被触发，为什么呢？因为在selector中从前往后匹配属性，第一个item和任何属性都会匹配，所以就算是执行了press的也是先匹配到上面的第一个item。  
 正确：

```text

<selector xmlns:android="http://schemas.android.com/apk/res/android">

    <item android:drawable="@drawable/pressed" android:state_pressed="true"/>
    <item android:drawable="@drawable/normal" />

selector>
```

把没有任何条件选择的item放到最后，当前面的item都不匹配时就会选择最后的item。

### **4、context引起的内存泄漏**

```text
public static WifiManager getWIFIManager(Context ctx) {
	WifiManager wm = (WifiManager) ctx.getSystemService(Context.WIFI_SERVICE);
}
```

上面的代码直接使用context来获取系统的WiFi服务，会导致下面问题  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20190816100901617.png)  
 获取WiFi\_SERVICE必需要用application 的context不能使用activity的context，如果上面方法中传入的是activity的context就会造成内存泄漏。  
 正确

```text
public static WifiManager getWIFIManager(Context ctx) {
	WifiManager wm = (WifiManager) ctx.getApplicationContext().getSystemService(Context.WIFI_SERVICE);
	}
```

这里为什么不用activity的context而要用application的context呢？因为activity的context生命周期和activity一致，当activity释放时context也应该被释放，这里由于context被wifiManager持有会导致activity不能被释放，而出现内存泄漏。application的context生命周期和application生命周期一致。所以当获取与当前activity生命周期无关而与整个应用生命周期有关的资源时，要使用application的context。

```text
public Context getApplicationContext ()

```

### **5、使用SparseArray代替HashMap**

在Android中如果要存放的中的key是基本数据类型：int,long,等基本数据类型时可以用SparseArray来代替HashMap,可以避免自动装箱和HashMap中的entry等带来的内存消耗。能被SparseArray代替的类型有如下几种：

```text
SparseArray          
SparseBooleanArray   
SparseIntArray       
SparseLongArray      
LongSparseArray      
LongSparseLongArray     //this is not a public class  
```

对比存放1000个元素的SparseIntArray和HashMap 如下：

**SparseIntArray**

```text
class SparseIntArray {
    int[] keys;
    int[] values;
    int size;
}
```

Class = 12 + 3 \* 4 = 24 bytes  
 Array = 20 + 1000 \* 4 = 4024 bytes  
 Total = 8,072 bytes

**HashMap**

```text
class HashMap<K, V> {
    Entry<K, V>[] table;
    Entry<K, V> forNull;
    int size;
    int modCount;
    int threshold;
    Set<K> keys
    Set<Entry<K, V>> entries;
    Collection<V> values;
}
```

Class = 12 + 8 \* 4 = 48 bytes  
 Entry = 32 + 16 + 16 = 64 bytes  
 Array = 20 + 1000 \* 64 = 64024 bytes  
 Total = 64,136 bytes  
 可以看到存放相同的元素，HashMap占用的内存几乎是SparseIntArray的8倍。

**SparseIntArray的缺点**

SparseIntArray采用的二分查找法来查找个keys，因此查找某个元素的速度没有Hashmap的速度快。存储的元素越多时速度比hashmap的越慢，因此当当数据量不大时可以采用SparseIntArray，但是当数据量特别大时采用HashMap会有更高的查找速度。

### **6、自定义view中在layout、draw、onMeasue中new对象**

**问题代码**

```text
pubic class myview extends View
{

    public myview(Context context) {
        super(context);
    }
    @Override
    protected void onDraw(Canvas canvas)
    {
        super.onDraw(canvas);
        int x=80;
        int y=80;
        int radius=40;
        Paint paint=new Paint();
        
        paint.setColor(Color.parseColor("#CD5C5C"));
        canvas.drawCircle(x,x, radius, paint);
    }

}
```

自定义view的时候经常会重写onLayout ,onDraw ,onMeasue方法，但是要注意的是，如果在这些方法里面new 对象就会有如下问题  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20190816214326747.png)  
 **优化代码**

```text
public class myview extends View
{
		int x;
        int y;
        int radius;
        Paint paint；
    public myview(Context context) {
        super(context);
        init();
    }
    @Override
    protected void onDraw(Canvas canvas)
    {
        super.onDraw(canvas);
        
        paint.setColor(Color.parseColor("#CD5C5C"));
        canvas.drawCircle(x,x, radius, paint);
    }
	private void init（）
	{
		x=80;
        y=80;
        radius=40;
        paint=new Paint();
	}	
}
```

看下Android官网上的解释为什么不能在onLayout ,onDraw ,onMeasue方法里面执行new 对象和执行初始化操作：

> Creating objects ahead of time is an important optimization. Views are redrawn very frequently, and many drawing objects require expensive initialization. Creating drawing objects within your onDraw\(\) method significantly reduces performance and can make your UI appear sluggish.

上面的意思总结一下就是在自定义view的时候最好提前初始化和new 对象，因为onDraw，onMeasure，onLayout的方法调用十分频繁，如果在这里初始化和new 对象会导致频繁的gc操作严重影响性能，也可能会导致掉帧现象。

**ondraw的调用时机**  
 1、view初始化的时候。  
 2、当view的invalidate\(\) 方法被调用。什么时候会调用invalidate\(\)方法呢？当view的状态需要改变的时候，比如botton被点击，editText相应输入等，就会reDraw调用onDraw方法。

### 7、静态变量引起的内存泄漏

问题代码

```text
public class MyDlg extends Dialog {
   
    private View mDialog;
    private static MyDlg sInstance;
    public MyDlg(Context context) {
        super(context);
        sInstance = this;
        init();
    }
    private void init() {
        mDialog = LayoutInflater.from(mCtx).inflate(R.layout.dialog_app_praise, null);
        setContentView(mDialog);
  }       
```

上面的代码会导致如下问题：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20190817171144610.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9oYW5raW5nLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70)  
 上面代码中静态变量sInstance持有来context而这里的context是持有当前dialog的activity，由于静态变量一般只有在App销毁的时候才会进行销毁（此时类经历了，加载、连接、初始化、使用、和卸载）所以当activity执行完时由于被dialog中的静态变量持有无法被gc，所以造成内存泄漏。而这种dialog如果被多个地方调用就会造成严重的内存泄漏。

### **8、overdraw问题**

问题代码

```text

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent"
    android:orientation="vertical"
    android:background="@drawable/layout_content_bkg" >

	<include layout="@layout/title_bar" />
	
	<ScrollView 
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginBottom="12dip"
        android:scrollbars="vertical" >
		
		<LinearLayout
			android:layout_width="fill_parent"
			android:layout_height="wrap_content"
			android:orientation="vertical"
			android:background="@drawable/layout_content_bkg"
			android:layout_marginTop="14dip"
			android:paddingLeft="13dip"
	   	 	android:paddingRight="13dip" >
			.......
		LinearLayout>
	ScrollView>
LinearLayout>
```

上面代码中linearlayout的background和ScrollView 里面的background一样只需要保留父布局LinearLayout里面的background就行了，不然会多进行一次绘制也就是引起overdraw问题。

### 9、inefficent layout weight

**问题代码**

```text
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="48dp"
        android:layout_gravity="bottom"
        android:background="@color/white"
        android:gravity="center_vertical"
        android:orientation="horizontal"
        >

    <TextView
        android:layout_weight="1"
        android:id="@+id/text"
        android:layout_width="165dp"
        android:layout_height="wrap_content"
        android:layout_marginLeft="15dp"
        android:textSize="15sp" />

    <android.support.v7.widget.SwitchCompat
        android:checked="true"
        android:id="@+id/0checkbox"
        android:layout_width="wrap_content"
        android:layout_height="36dp"
        android:layout_marginEnd="15dp" />
    LinearLayout>
```

上面布局中一个linearLayout里面包含一个textview和一个SwitchCompat，textview中的layout\_weight=“1”，此时也明确给出了textview的layout\_width=“165dp”，这个时候会带来下面的问题  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20190817173805566.png)  
 当linearLayout的布局里面只有一个子view使用weight属性时如果LinearLayout是垂直布局这个子view应该设置layout\_height=“0dp”，如果是水平布局这个子view应该layout\_width=“0dp”，这样执行onMeasure的时候会首先不去measure 这个布局，可以提高性能。

### 10、字符串操作优化

```text
String text="bitch"；

StringBuilder.append("  fuck " + text);

mStringBuilder.append("  fuck ").append(text);
```

上面代码中1和2哪个代码性能更好？第二种性能更高，第一种方式会先new一个“fuck”+text的字符串然后在append到StringBuilder中，而第二种方法中不用new一个字符串，所以性能更高。

## 代码优化

### **1、消除redundant “Collection.addAll\(\)”**

原始代码：

```text
        ArrayList<BaseFile> lists = null;
        if (getImages() != null) {
            lists = new ArrayList<>();
            lists .addAll(getImages());
        }
```

优化代码：

```text
        ArrayList<BaseFile> lists = null;
        if (getImages() != null) {
            lists = new ArrayList<>(getImages());
        }
```

### 2、使用array的copy方法

原始代码：

```text
        for(int i=0;i<ITEM_SIZE;i++){
            mContentsArray[i] = contents[i];
        }
```

优化代码：

```text
System.arraycopy(contents, 0, mContentsArray, 0, ITEM_SIZE);
```

### 3、final 修饰static方法

原始代码：

```text
 public static final void goToBrothel() {
    }
```

上面的代码中用final修饰static方法是没有必要的，使用final 是为了防止goToBrothel方法被继承的子类重写，子类有和父类一样的静态方法不叫重写叫hiding：解释：

> Hiding: Parent class methods that are static are not part of a child  
>  class \(although they are accessible\), so there is no question of  
>  overriding it. Even if you add another static method in a subclass,  
>  identical to the one in its parent class, this subclass static method  
>  is unique and distinct from the static method in its parent class.  
>  父类的静态方法并不属于子类的一部分，所以就没有子类重写父类静态方法的问题，如果子类中有一个与父类一样的静态方法，子类的静态方法和父类的静态方法也是不同的。  
>  ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20191218204228448.png)  
>  上面的代码可以修改成：

```text
 public static void goToBrothel() {
    }
```

### 4、final 修饰private方法

原始代码：

```text
 private final void goToBrothel() {
    }
```

private方法不能被子类获取，也不能被子类重写，当子类中有与父类同名的方法，并且父类中方法是private时，使用子类的时候会调用子类中的方法。所以用final修饰private方法是没意义的。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20191218204834923.png)  
 修改后的代码：

```text
 private void goToBrothel() {
    }
```

### 参考文献

1、[https://stackoverflow.com/questions/5960678/whats-the-difference-between-commit-and-apply-in-shared-preference](https://stackoverflow.com/questions/5960678/whats-the-difference-between-commit-and-apply-in-shared-preference)

2、[https://stackoverflow.com/questions/4128589/difference-between-activity-context-and-application-context](https://stackoverflow.com/questions/4128589/difference-between-activity-context-and-application-context)

3[https://developer.android.com/reference/android/content/ContextWrapper.html\#getApplicationContext\(\)](https://developer.android.com/reference/android/content/ContextWrapper.html#getApplicationContext%28%29)

4、[https://stackoverflow.com/questions/25560629/sparsearray-vs-hashmap](https://stackoverflow.com/questions/25560629/sparsearray-vs-hashmap)

5、[https://developer.android.com/training/custom-views/custom-drawing](https://developer.android.com/training/custom-views/custom-drawing)  
 6、[https://stackoverflow.com/questions/11912406/view-ondraw-when-does-it-get-called](https://stackoverflow.com/questions/11912406/view-ondraw-when-does-it-get-called)


# DialogFragment自定义布局和大小踩坑记\_有图有真相-CSDN博客\_dialogfragment自定义布局

## 前言

关于DilogFragment之前写过几篇博客，但是并不是很深入，有些问题也没有解决。  
 **例如：**

1、DialogFragment和Activity的关系。

2、DilogFragment生命周期和和设置布局大小无效问题。

为了解决上面的2个问题，需要从两个方面来入手，1、DialogFragment和Activity的关系。2、DialogFragment和Dialog的关系。理清楚了这些关系后上面的2点问题也就解决了。

## 1、DialogFragment和Activity的关系。

DialogFragment继承Fragment生命周期和所在的Activity生命周期相关，由FragmentManager管理。Activity的生命周期由Framework层中的ActivityManagerService来管理的，而Fragment只对Activity可见，对Framework层并不可见，也就是Framework并不知道Fragment的存在，Fragment的生命周期完全由Activity中的FragmentManager来管理。Activity和Fragment关系如下图所示：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/201901112119447.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)

Activity通过FragmentManager来管理Fragment。FragmentManager可以通过FragmentTransaction把Fragment加入到Back Stack中，FragmentTransaction和数据库操作的方法一样，有add\(\)，remove\(\),commit\(\)等方法来操作Fragment。已经知道了Fragment是依赖于Activity的，下面给出Fragment和Activity生命周期的关系。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20190111212558917.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)  
 Fragment和Activity的生命周期的对应关系后面我会专门写一篇博客。其实原理并不复杂，就是在Activity的各个生命周期中去分别调用其包含的Fragment对应的生命周期的函数。在Activtiy中使用Fragment非常简单如下：

```text
public class DemoActivity extends FragmentActivity {
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_crime);
        FragmentManager fm = getSupportFragmentManager();
        Fragment fragment = fm.findFragmentById(R.id.fragmentContainer);
        if (fragment == null) {
            fragment = new CrimeFragment();
            fm.beginTransaction()
                    .add(R.id.fragmentContainer, fragment)
                    .commit();
        }
    }
}

```

上面的例子是在Activity的onCreate方法中调用FragmentManager 来将Fragment加入到Activity中。

**如果在Activity的其他生命周期中将Fragment加入到Activity中呢？比如说在Activity处于stopped，paused，或者running状态时，加入Fragment的生命周期是怎样的？**  
 这个时候FragmentManager会立即执行Fragment需要的生命状态直到和activity相匹配的状态。比如说在Activity处于running状态时加入Fragment，此时Fragment会执行生命 onAttach\(Activity\), onCreate\(Bundle\),  
 onCreateView\(…\), onActivityCreated\(Bundle\), onStart\(\),最后执行 onResume\(\).  
 如下图：在Activity处于onResume的时候addFragment,此时Fragment执行的生命周期。  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20190112162748391.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)

## 2、DilogFragment生命周期和和设置布局大小无效问题。

下面看看在Activity中使用DialogFragment的例子，然后分析两点：  
 **1、为什么DialogFragment会以弹框的形式显示。**  
 **2、为什么在要在onStart\(\)方法中动态改变DialogFragment才能动态改变布局。**

```text
new MyDialogFragment().show(getFragmentManager(),"id");
```

DialogFragment的使用十分简单，直接在Activity中调用DialogFragment的show\(\)方法同时传入当前Activity中的FragmentManager，传入的FragmentManager肯定是用来管理DialogFragment的。

```text
    public void show(FragmentManager manager, String tag) {
        mDismissed = false;
        mShownByMe = true;
        FragmentTransaction ft = manager.beginTransaction();
        ft.add(this, tag);
        ft.commit();
    }
```

DialogFragment中的show\(\)方法调用FragmentManager把自己加入到back stack中，生命周期由FragmentManager管理。在Activity中调用DialogFragment的show\(\)方法此时并不会立即显示，这个时候关联了DialogFragment和activity的生命周期，那么DialogFragment在什么时候显示的呢？

```text
    @Override
    public void onStart() {
        super.onStart();
        if (mDialog != null) {
            mViewDestroyed = false;
            mDialog.show();
        }
    }
```

只有DialogFragment的生命周期执行到onStart\(\)的时候才会调用mDialog.show\(\);显示弹框，在[DialogFragment源码解析中](https://blog.csdn.net/u013309870/article/details/85210990)已经说过，DialogFragment的布局会被加到mDialog中，所以DialogFragment才会显示成弹框形式。这解决了为什么DialogFragment会以弹框的形式显示的问题。

在onCreateView中定义的布局最后会被加入到Dialog中，所以要改变弹出框的大小其实是要改变Dialog的大小。Dialog的大小要在Dialog.show\(\)方法之后才能动态改变。下面分析一下为什么Dialog的大小在Dialog.show\(\)方法之后才能改变。看看Dialog.show\(\)的源码：

```text
public void show() {
        if (mShowing) {
            return;
        }
        if (!mCreated) {
            dispatchOnCreate(null);
        } else {
        }
    mWindowManager.addView(mDecor, l);    
}
  
```

上面代码是简化之后的，如果Dialog还没有创建就会调用dispatchOnCreate\(null\);方法来创建Dialog的布局，下面看看dispatchOnCreate\(null\)方法，简化后如下：

```text
    void dispatchOnCreate(Bundle savedInstanceState) {
        if (!mCreated) {
            mDialog.setContentView(selectContentView());
            mCreated = true;
        }
    }

    private int selectContentView() {
        if (mButtonPanelSideLayout == 0) {
            return mAlertDialogLayout;
        }
        if (mButtonPanelLayoutHint == AlertDialog.LAYOUT_HINT_SIDE) {
            return mButtonPanelSideLayout;
        }
        return mAlertDialogLayout;
    }
```

由上面可以看出，使用Dialog的子类AlertDialog时，使用的contentView是Android自带样式和大小的Layout，用户自定义的view被加到mDecor上，所以在show\(\)之前设置xml的大小是无效的，最后还是会在show中被覆盖成系统自带的格式，只有在show后面改变布局属性才会生效。

这也就解释了为什么DialogFragment在onCreate\(\)和onCreateView\(\)中设置布局大小无效，因为onCreate\(\)和onCreateView\(\)生命周期在onStart\(\)生命周期之前，此时还未调用Dialog.show\(\)方法，设置大小无效。可以总结出Activity和DialogFragment的关系如下图：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20190112185550598.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)  
 **坑1自定义弹框的大小**  
 在自定义布局中设置的大小是不起作用的，要设置自定义布局的大小只有在代码中动态设置，在`onStart`中重写布局大小，在`onCreat`或者`onCreateView`中无效

```text
    
    @Override
    public void onStart() {
        super.onStart();
        XLLog.d(TAG, "onStart");
        resizeDialogFragment();

    }

    private void resizeDialogFragment() {
        Dialog dialog = getDialog();
        if (null != dialog) {
            Window window = dialog.getWindow();
            WindowManager.LayoutParams lp = getDialog().getWindow().getAttributes();
            lp.height = (25 * ScreenUtil.getScreenHeight(getContext()) / 32);
            lp.width = (8 * ScreenUtil.getScreenWidth(getContext()) / 9);
            if (window != null) {
                window.setLayout(lp.width, lp.height);
            }
        }
    }
```

**坑**：  
 **这里不得不提到的坑是，看起来已经动态设置了自己想要的布局大小。但实际运行出来的大小和定义的尺寸有偏差**。上面代码中设置的宽度是屏幕的8/9，运行代码是得到的 `lp.width=960`，但我用Layout Inspector检测出来自定义的布局宽度仅仅是876，这中间差了84。所以肯定是系统在自定义的布局外面又包了一层其他的东西，导致设置出来的宽度和实际显示的不一样。  
 通过

```text
        Dialog dialog = getDialog();
        if (null != dialog) {
            Window window = dialog.getWindow();
            Log.d(TAG, "padding.................." + window.getDecorView().getPaddingLeft() + "............................." + window.getDecorView().getPaddingTop());

```

由上面结果可知，在自定义布局外面还有一个padding的距离，这个padding 距离四周的距离都是42，876+42\*2=960，正好和设置的宽度相同。  
 **检测布局**  
 用Android studio 中的Layout Inspector检测布局  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20181221182438806.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)  
 由上图可以看到布局结构：  
 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/20181221184109634.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMzMDk4NzA=,size_16,color_FFFFFF,t_70)  
 整个弹出框的根布局是DecorView,DecorView里面包含了一个FragmentLayout，FragmentLayout里面包含两个布局一个是content,就是我们自定义的布局，Action\_mode\_bar\_stub这个是actionBar,我们这里的actionBar布局为null什么都没有。  
 其中DecorView的定义可以看一段英文：

> The DecorView is the view that actually holds the window’s background drawable. Calling getWindow\(\).setBackgroundDrawable\(\) from your Activity changes the background of the window by changing the DecorView‘s background drawable. As mentioned before, this setup is very specific to the current implementation of Android and can change in a future version or even on another device.

其中的padding=42就是DecorView与其他布局的间距，所以获取到DecorView再设置它的padding就好了。  
 **解决方案**  
 **方案1：设置透明背景**  
 要在onCreateView中设置背景为透明，原来dialogFragment在自定义的布局外加了一个背景，设置为透明背景后，尺寸就和设置的尺寸一样了。加上

```text
getDialog().getWindow().setBackgroundDrawable(new ColorDrawable(Color.TRANSPARENT));
```

```text
 @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        getDialog().getWindow().setBackgroundDrawable(new ColorDrawable(Color.TRANSPARENT));
        View view = inflater.inflate(R.layout.message_share_websit_dialog, container);
        return view;

    }
```

搞定

## 总结

本文主要从DialogFragment和Activity的关系以及DialogFragment的生命周期特点来分析为什么DialogFragment会显示弹框形式，以及动态设置布局时为什么要在onStart\(\)方法中生效，在onCreate\(\)和onCreateView\(\)中设置不生效问题。

## 参考文献

1、Android Programming\_ The Big Nerd Ranch Guide  
 2、[https://developer.android.com/courses/fundamentals-training/overview-v2](https://developer.android.com/courses/fundamentals-training/overview-v2)  
 备注：2链接中有google官方提供的初级和高级教程，而且配有ppt讲解十分详细，是很好的学习资源。


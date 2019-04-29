# LayoutInflate的使用

我们在开发中经常会用到**LayoutInflater**，用来加载布局，Activity中的setContentView()方法的内部其实也是使用LayoutInflater来加载布局的，下面就总结一下LayoutInflater的简单使用。

## 1.获取LayoutInflater

* Activity中调用`getLayoutInflater()`方法

```java
LayoutInflater inflater = getLayoutInflater();
```

该方法内部调用了`getWindow().getLayoutInflater()`，这里getWindow()返回的是PhoneWindow对象，因此实际上调用的是PhoneWindow的`getLayoutInflater()`方法。

```java
public PhoneWindow(Context context) {
    super(context);
    mLayoutInflater = LayoutInflater.from(context);
}
```

可以看出最终是调用了`LayoutInflater.from(context)` 。

* 调用`LayoutInflater.from(context)`

```java
public static LayoutInflater from(Context context) {
    LayoutInflater LayoutInflater =
            (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    if (LayoutInflater == null) {
        throw new AssertionError("LayoutInflater not found.");
    }
    return LayoutInflater;
}
```

可以看出该方法最终调用了`context.getSystemService(Context.LAYOUT_INFLATER_SERVICE)`。

* 调用`Context.getSystemService(Context.LAYOUT_INFLATER_SERVICE)`

这三种方式本质上都是通过调用`Context.getSystemService(Context.LAYOUT_INFLATER_SERVICE)`来实现的。

## 2.LayoutInflater的inflate方法

获取了LayoutInflater对象后，需要调用它的`inflate`方法来加载布局。该方法有四种重载方式，分别来看一下。

![](https://ws3.sinaimg.cn/large/005BYqpggy1g2j91j1te6j30kb056q35.jpg)

1）`inflate(int resource,ViewGroup root)`

```java
public View inflate(int resource,ViewGroup root) {
    return inflate(resource, root, root != null);
}
```

内部调用了第三个重载方法。

2）`inflate(XmlPullParser parser,ViewGroup root)`

```java
public View inflate(XmlPullParser parser, ViewGroup root) {
    return inflate(parser, root, root != null);
}
```

内部调用了第四个重载方法。

3）`inflate(int resource,ViewGroup root,boolean attachToRoot)`

```java
public View inflate(int resource, ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    if (DEBUG) {
        Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                + Integer.toHexString(resource) + ")");
    }

    final XmlResourceParser parser = res.getLayout(resource);
    try {
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}
```

内部先将布局资源id转换为XmlResourceParser对象，最后调用了第四个重载方法。

4）`inflate(int resource,ViewGroup root,boolean attachToRoot)`

上面三个方法最终都会调用这个重载方法，因此我们来具体看一下该方法的实现。

```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");
        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        mConstructorArgs[0] = inflaterContext;
        // 默认返回传入的父布局，后面还会有判断
        View result = root;
        try {
            // 获得xml文件根布局节点
            int type;
            while ((type = parser.next()) != XmlPullParser.START_TAG &&
                    type != XmlPullParser.END_DOCUMENT) {
            }
            if (type != XmlPullParser.START_TAG) {
                throw new InflateException(parser.getPositionDescription()
                        + ": No start tag found!");
            }
            final String name = parser.getName();
            if (TAG_MERGE.equals(name)) {
                // <merge />为根节点，必须指定一个父布局，并且设定attachToRoot为true
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }
                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                // 获得根布局View，赋值给temp
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);
                ViewGroup.LayoutParams params = null;
                if (root != null) {
                    params = root.generateLayoutParams(attrs);
                    // 传入的父布局不为空
                    if (!attachToRoot) {
                        // attachToRoot为false，为根布局设置LayoutParams
                        temp.setLayoutParams(params);
                    }
                }
                // 遍历根布局下的子元素
                rInflateChildren(parser, temp, attrs, true);
                // 传入的父布局不为空，并且attachToRoot为true
                if (root != null && attachToRoot) {
                    // 将根布局添加到父布局中
                    root.addView(temp, params);
                }
                // 决定返回的是父布局还是xml文件的根布局
                // 传入的父布局为空或者attachToRoot为false，返回xml文件根布局，否则返回传入的父布局
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }
        } catch (XmlPullParserException e) {
            final InflateException ie = new InflateException(e.getMessage(), e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;
        } catch (Exception e) {
            final InflateException ie = new InflateException(parser.getPositionDescription()
                    + ": " + e.getMessage(), e);
            ie.setStackTrace(EMPTY_STACK_TRACE);
            throw ie;
        } finally {
            // Don't retain static reference on context.
            mConstructorArgs[0] = lastContext;
            mConstructorArgs[1] = null;
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        return result;
    }
}
```

其中关键的代码如下：

```java
...
if (root != null) {
    params = root.generateLayoutParams(attrs);
    // 传入的父布局不为空
    if (!attachToRoot) {
        // attachToRoot为false，为根布局设置LayoutParams
        temp.setLayoutParams(params);
    }
}
...
if (root != null && attachToRoot) {
    root.addView(temp, params);
}
...
if (root == null || !attachToRoot) {
    result = temp;
}
```

可以看出返回的View和传入的参数有关系，具体可分为以下几种情况：

* 如果root为null，那么attachToRoot为true或者false没有区别，都是返回传入的xml文件根布局View，并且不会为该布局设置布局属性。
* root不为空，并且attachToRoot为false，会先为传入的xml文件根布局View设置布局属性，最后返回该布局。
* root不为空，并且attachToRoot为true，会先给xml文件根布局View设置布局属性，然后添加到传入父布局中，最后返回父布局。

对于上面两个参数的方法，attachToRoot是取决于root是否为空的，如果不为空，attachToRoot就为true，否则为false。

总结一下，`inflate()`方法的三个参数，第一个参数**resource**表示要加载的布局资源id；第二个参数**root**表示要将资源文件根布局添加到哪个父布局中；第三个参数**attachToRoot**表示是否要将资源文件根布局添加到父布局中，如果传入了父布局，attachToRoot为true则添加到父布局中，为false则不会添加到父布局。

光说结论可能不是很清楚，下面通过实际情况来验证我们的结论。

首先新建一个布局文件**layout_button.xml**，里面只有一个按钮

```xml
<?xml version="1.0" encoding="utf-8"?>
<Button xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content">

</Button>
```

在Activity的onCreate()方法中加载这个布局

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    LinearLayout llRoot = findViewById(R.id.ll_root);
    LayoutInflater inflater = LayoutInflater.from(this);
    View view = inflater.inflate(R.layout.layout_button, null);
  	Log.e("TAG", "View：" + view.toString());
    Log.e("TAG", "LayoutParams： " + view.getLayoutParams());
    llRoot.addView(view);
}
```

**activity_main.xml**中的布局如下：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/ll_root"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

</LinearLayout>
```

接下来分别看一下`inflate()`方法传不同参数的结果：

1）root传null，attachToRoot不传

![](https://ws3.sinaimg.cn/large/005BYqpggy1g2jd8ueilvj30by013742.jpg)

可以看出，这种情况下`inflate()`方法返回的View就是传入的布局资源文件根布局，也就是Button，并且该View没有设置LayoutParams。

2）root传父布局，attachToRoot不传

![](https://ws3.sinaimg.cn/large/005BYqpggy1g2jfbmbw6vj30l2017dfo.jpg)

虽然attachToRoot没传，通过之前的分析可以得出最后attachToRoot的取值是true，可以看出这种情况下`inflate()`方法返回的View是一个LinearLayout，也就是我们传入的root父布局，因为方法内部会将传入的资源文件根布局添加到父布局中，最后返回父布局。

但是这种情况下应用会抛出以下异常，直接Crash掉。

```
java.lang.IllegalStateException: The specified child already has a parent. You must call removeView() on the child's parent first.
```

可以看出导致异常的原因是调用`addView()`方法时，child已经有了父布局，由于此处的child是**activity_main.xml**的根布局LinearLayout，它已经有了父布局（android.R.id.content，即ContentView）了。

解决方法就是去掉最后的`llRoot.addView(view)`，因为已经将资源文件文件的根布局添加到了父布局中。

3）root传null，attachToRoot传false

这种情况和第一种情况是一样的，因为当attachToRoot不传时，attachToRoot最后的取值取决于root是否为空。

4）root传null，attachToRoot传true

![](https://ws3.sinaimg.cn/large/005BYqpggy1g2jfdgvt7cj30bx01fa9u.jpg)

这种情况也和第一种情况是一样的，因为只要root为null，无论attachToRoot传什么值，`inflate()`方法返回的View都是传入的布局资源文件根布局，并且该View没有设置LayoutParams。

5）root传父布局，attachToRoot传false

![](https://ws3.sinaimg.cn/large/005BYqpggy1g2jfh1yn81j30ft018web.jpg)

这种情况下`inflate()`方法返回的View也是传入的布局资源文件根布局，但是该View是设置了LayoutParams的。

6）root传父布局，attachToRoot传true

这种情况和第二种情况是一样的，这里就不分析了。

搞清楚`inflate()`方法的参数意义后，我们还要注意该方法在实际开发中的使用，有时候参数的取值会影响显示效果或导致应用崩溃。

**1.layout_width和layout_height无效问题**

在上面的例子中，我们加载的布局文件的根布局是一个Button，如果我们设置了它的layout_width和layout_height属性，指定了一个具体的值，对于root传null的情况下是无效的。这是因为这两个属性并不是直接设置了View的宽和高，而是设置了View在父容器中的大小，而root传null时，该View是没有父布局的，因此这两个属性并不会有任何作用。而当root不为空时，会根据root生成布局参数并设置给View。

**2.Fragment的onCreateView中调用inflate**

```java
@Override
public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
  	View view = inflater.inflate(R.layout.fragment_test, container, false);
    return mRootView;
}
```

在Fragment的onCreateView()方法中加载布局时传入的attachToRoot必须为false，因为如果attachToRoot为true，在加载布局时方法内部会将资源文件根布局添加到root父布局中，但是这个操作本来应该是由FragmentManager来控制的，即调用`add()`方法来添加布局，这样会导致The specified child already has a parent异常。

**3.RecyclerView Adapter的onCreateViewHolder中调用inflate**

```java
@Override
public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {  
	LayoutInflater inflater = LayoutInflater.from(mContext);  
    View view = inflater.inflate(R.layout.item, parent, false);  
    return new ViewHolder(view);  
}
```

和Fragment的情况类似，RecyclerView的Adapter中加载item布局文件传入的attachToRoot也必须为false，因为添加View的操作同样是由Adapter控制的，如果在此之前已经添加了View，同样会导致The specified child already has a parent异常。

其实调用`inflater.inflate(R.layout.item, null)`也是可以的，只是这样可能会导致根布局的宽高设置失效，因此一般root还是要传的。
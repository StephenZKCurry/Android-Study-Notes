# View知识体系

[TOC]

**本文的分析基于Android 9.0（API Level 28）的源码**

## 1.View的绘制流程

## 1.1.

**AppCompatActivity的setContentView方法**

```java
@Override
public void setContentView(@LayoutRes int layoutResID) {
    getDelegate().setContentView(layoutResID);
}
```



**AppCompatDelegateImpl的setContentView方法**

```java
@Override
public void setContentView(int resId) {
    ensureSubDecor();
    ViewGroup contentParent = (ViewGroup) mSubDecor.findViewById(android.R.id.content);
    contentParent.removeAllViews();
    LayoutInflater.from(mContext).inflate(resId, contentParent);
    mOriginalWindowCallback.onContentChanged();
}
```



```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    return inflate(resource, root, root != null);
}
```



## 1.2.measure

为了研究View的measure流程，我们首先要介绍两个相关的类：**MeasureSpec**和**LayoutParams**。

### 1.2.1.MeasureSpec

MeasureSpec是由一个32位int值表示的（我想到了MotionEvent），高2位表示，低30位表示。

| 子View\父View  | EXACTLY | AT_MOST | UNSPECIFIED |
| ------------ | ------- | ------- | ----------- |
| 具体数值         | EXACTLY |         |             |
| match_parent | EXACTLY |         |             |
| wrap_content | AT_MOST |         |             |



### 1.2.2.LayoutParams

#### 1.2.2.1.LayoutParams简介

**LayoutParams**这个类在开发中还是很常见的，顾名思义就是布局参数，View中定义了一个LayoutParams类型的成员变量，它的作用就是确定View的宽高，我们平时在xml布局文件中指定的**layout_width**和**layout_height**参数就是用于生成LayoutParams。需要注意的是，这两个属性的前面都带上layout前缀，而不是直接使用**width**和**height**来命名，因此我们要清楚它们的值并不是View的宽高，也可以说它们并不属于View自身的属性。

LayoutParams是ViewGroup中的一个内部类，我们看一下它的定义：

```java
public static class LayoutParams {

    public static final int FILL_PARENT = -1; // 已被MATCH_PARENT取代

    public static final int MATCH_PARENT = -1;

    public static final int WRAP_CONTENT = -2;

    @ViewDebug.ExportedProperty(category = "layout", mapping = {
            @ViewDebug.IntToString(from = MATCH_PARENT, to = "MATCH_PARENT"),
            @ViewDebug.IntToString(from = WRAP_CONTENT, to = "WRAP_CONTENT")
    })
    public int width;

    @ViewDebug.ExportedProperty(category = "layout", mapping = {
            @ViewDebug.IntToString(from = MATCH_PARENT, to = "MATCH_PARENT"),
            @ViewDebug.IntToString(from = WRAP_CONTENT, to = "WRAP_CONTENT")
    })
    public int height;

    public LayoutAnimationController.AnimationParameters layoutAnimationParameters;

    /**
     * xml文件中指定的属性
     */
    public LayoutParams(Context c, AttributeSet attrs) {
        TypedArray a = c.obtainStyledAttributes(attrs, R.styleable.ViewGroup_Layout);
        setBaseAttributes(a,
                R.styleable.ViewGroup_Layout_layout_width,
                R.styleable.ViewGroup_Layout_layout_height);
        a.recycle();
    }

    /**
     * 手动指定宽高属性
     */
    public LayoutParams(int width, int height) {
        this.width = width;
        this.height = height;
    }

    /**
     * 指定LayoutParams
     */
    public LayoutParams(LayoutParams source) {
        this.width = source.width;
        this.height = source.height;
    }

    LayoutParams() {
    }

    // 省略部分方法...
}
```

LayoutParams中定义了几个重载构造函数，分别用于xml文件中指定宽高、手动指定宽高等场景。每个ViewGroup的子类（直接或间接继承）都有对应的LayoutParams类，比如**LinearLayout.LayoutParams**，在各自的LayoutParams中可以定义相应的布局参数属性。因此不止**layout_width**和**layout_height**这两个属性，其他以layout开头的属性（比如**layout_weight**、**layout_margin**等等）也都和布局参数相关。

#### 1.2.2.2.View的LayoutParams属性是何时设置的

了解了LayoutParams的定义后，接下来需要弄清楚View的LayoutParams属性是何时设置的，我们知道在ViewGroup中添加子View的方式有两种：xml文件中添加和代码中添加，我们分别来看一下这两种情况。

* **xml文件中添加View**

在此前的分析中我们知道xml文件中添加的View最终是通过**LayoutInflater**的`inflate()`方法来解析的。

```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        // 省略部分代码...
        View result = root;
        int type;
        while ((type = parser.next()) != XmlPullParser.START_TAG &&
                type != XmlPullParser.END_DOCUMENT) {
        }
        // 省略部分代码...
        final String name = parser.getName();
        // 省略部分代码...
       
        final View temp = createViewFromTag(root, name, inflaterContext, attrs);
        ViewGroup.LayoutParams params = null;
        if (root != null) {
          	// 调用generateLayoutParams()方法生成LayoutParams
            params = root.generateLayoutParams(attrs);
            if (!attachToRoot) {
                temp.setLayoutParams(params);
            }
        }

        rInflateChildren(parser, temp, attrs, true);

        if (root != null && attachToRoot) {
            root.addView(temp, params);
        }
      
        if (root == null || !attachToRoot) {
            result = temp;
        }
    }
    // 省略部分代码...
    return result;
}
```

在`inflate()`方法中有一行代码是`params = root.generateLayoutParams(attrs);`，`generateLayoutParams()`方法的作用就是就是根据xml指定的属性构造出LayoutParams对象。

```java
public LayoutParams generateLayoutParams(AttributeSet attrs) {
    return new LayoutParams(getContext(), attrs);
}
```

构造出LayoutParams对象后根据参数`attachToRoot`的值有两种处理逻辑：

1. 如果`attachToRoot`为true，则会调用`addView()`方法并传入构造好的LayoutParams对象，`addView()`方法内部会将LayoutParams对象设置给View，详细代码后面会展示，这里先这样记住就好。
2. 如果`attachToRoot`为false，则会调用View的`setLayoutParams()`方法直接将构造好的LayoutParams对象设置给View。

```java
public void setLayoutParams(ViewGroup.LayoutParams params) {
    if (params == null) {
        throw new NullPointerException("Layout parameters cannot be null");
    }
  	// 给mLayoutParams变量赋值
    mLayoutParams = params;
    resolveLayoutParams();
    if (mParent instanceof ViewGroup) {
        ((ViewGroup) mParent).onSetLayoutParams(this, params);
    }
  	// 重新执行View的绘制流程，measure->layout->draw
    requestLayout();
}
```

* **代码中添加View**

我们一般会使用ViewGroup的`addView()`方法来添加子View，就像下面这样：

```java
LinearLayout llContainer = findViewById(R.id.ll_container);
View child = new View(this);
llContainer.addView(child);
```

`addView()`方法有很多个重载方法，上面的代码我只展示了最简单的一个参数的情况，下面我们就来具体看一下所有的重载方法。

```java
/**
 * 方法1
 */
public void addView(View child) {
    addView(child, -1);
}

/**
 * 方法2
 */
public void addView(View child, int index) {
    if (child == null) {
        throw new IllegalArgumentException("Cannot add a null child view to a ViewGroup");
    }
    LayoutParams params = child.getLayoutParams();
    if (params == null) {
        params = generateDefaultLayoutParams();
        if (params == null) {
            throw new IllegalArgumentException("generateDefaultLayoutParams() cannot return null");
        }
    }
    addView(child, index, params);
}

/**
 * 方法3
 */
public void addView(View child, int width, int height) {
    final LayoutParams params = generateDefaultLayoutParams();
    params.width = width;
    params.height = height;
    addView(child, -1, params);
}

/**
 * 方法4
 */
@Override
public void addView(View child, LayoutParams params) {
    addView(child, -1, params);
}

/**
 * 方法5
 */
public void addView(View child, int index, LayoutParams params) {
    if (DBG) {
        System.out.println(this + " addView");
    }

    if (child == null) {
        throw new IllegalArgumentException("Cannot add a null child view to a ViewGroup");
    }

    // addViewInner() will call child.requestLayout() when setting the new LayoutParams
    // therefore, we call requestLayout() on ourselves before, so that the child's request
    // will be blocked at our level
    requestLayout();
    invalidate(true);
    addViewInner(child, index, params, false);
}
```

不难看出，方法1内部调用了方法2，而在方法2内部会先判断子View是否设置了LayoutParams属性，如果没有设置，就调用`generateDefaultLayoutParams()`方法创建出一个默认的LayoutParams对象，最后调用方法5。再看方法3和方法4，这两个方法最终同样会调用方法5，因此我们只需要看方法5就好。

在此之前，我们先看一下`generateDefaultLayoutParams()`方法是如何创建出默认的LayoutParams对象的：

```java
protected LayoutParams generateDefaultLayoutParams() {
    return new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
}
```

可以看出，ViewGroup默认创建出LayoutParams对象的宽高属性均为**WRAP_CONTENT**。

回到方法5，可以看到最后调用了`addViewInner()`方法：

```java
private void addViewInner(View child, int index, LayoutParams params,
                          boolean preventRequestLayout) {

    // 省略部分代码...
    if (!checkLayoutParams(params)) {
        params = generateLayoutParams(params);
    }

    if (preventRequestLayout) {
        child.mLayoutParams = params;
    } else {
        child.setLayoutParams(params);
    }

    // 省略部分代码...
}
```

方法内部首先会调用`checkLayoutParams()`方法检查LayoutParams参数是否合法，如果不合法就调用`generateLayoutParams()`方法构造一个新的LayoutParams对象，`generateLayoutParams()`方法我们其实在上面xml文件中添加View的分析中刚见过，不过这里调用的是另一个重载方法，参数为LayoutParams对象。

```java
protected boolean checkLayoutParams(ViewGroup.LayoutParams p) {
    return  p != null;
}

protected LayoutParams generateLayoutParams(ViewGroup.LayoutParams p) {
    return p;
}
```

之后就是为View设置LayoutParams属性了，这里会判断传入的`preventRequestLayout`参数值，如果为true就直接对View的mLayoutParams变量赋值；如果为false则调用`setLayoutParams()`方法来给View设置LayoutParams，这两种情况的区别就是`setLayoutParams()`方法内部会调用`requestLayout()`方法来重新进行View的measure、layout和draw流程。可以看到由于`addView()`方法调用`addViewInner()`时传入的参数为false，因此这里会执行`setLayoutParams()`方法。额外提一下，ViewGroup中有一个`addViewInLayout()`方法，和`addView()`方法类似，内部也调用了`addViewInner()`方法，不过该方法可以显式地指定`preventRequestLayout`参数的值。

#### 1.2.2.3.自定义LayoutParams须知

看到这里关于LayoutParams的作用和使用原理基本上就介绍得差不多了，最后再简单介绍自定义LayoutParams。我们在自定义ViewGroup的同时，根据需求可能需要自定义LayoutParams，这里就以我们最熟悉的**LinearLayout**来看一下有哪些需要注意的地方吧。

* **重写generateDefaultLayoutParams()方法**

```java
@Override
protected LayoutParams generateDefaultLayoutParams() {
    if (mOrientation == HORIZONTAL) {
        return new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
    } else if (mOrientation == VERTICAL) {
        return new LayoutParams(LayoutParams.MATCH_PARENT, LayoutParams.WRAP_CONTENT);
    }
    return null;
}
```

通过前面的分析我们知道`generateDefaultLayoutParams()`方法的作用是在调用`addView()`方法时为没有指定LayoutParams的View创建一个默认的LayoutParams对象。对于LinearLayout来说，如果布局方向为水平则子View的宽和高均为**WRAP_CONTENT**；如果布局方向为竖直则子View的宽为**MATCH_PARENT**，高为**WRAP_CONTENT**。

* **重写checkLayoutParams(ViewGroup.LayoutParams p)方法**

`checkLayoutParams()`方法的作用是检查LayoutParams参数是否合法，我们已经看到了ViewGroup的判断标准为LayoutParams是否非空，那么LinearLayout是如何判断的呢。

```java
protected boolean checkLayoutParams(ViewGroup.LayoutParams p) {
    return p instanceof LinearLayout.LayoutParams;
}
```

可以看到LinearLayout的判断标准为LayoutParams的类型是否为**LinearLayout.LayoutParams**，在自定义LayoutParams时我们可以根据需要来选择重写该方法。

* **重写generateLayoutParams(ViewGroup.LayoutParams p)方法**

在`checkLayoutParams()`方法返回false即LayoutParams不合法的情况下会调用`generateLayoutParams()`方法重新创建一个合法的LayoutParams，LinearLayout中的定义如下：

```java
@Override
protected LayoutParams generateLayoutParams(ViewGroup.LayoutParams lp) {
    if (sPreserveMarginParamsInLayoutParamConversion) {
        if (lp instanceof LayoutParams) {
            return new LayoutParams((LayoutParams) lp);
        } else if (lp instanceof MarginLayoutParams) {
            return new LayoutParams((MarginLayoutParams) lp);
        }
    }
    return new LayoutParams(lp);
}
```

`generateLayoutParams()`方法内部会对传入的LayoutParams进行类型转换，转换为**LinearLayout.LayoutParams**或**ViewGroup.MarginLayoutParams**，这个MarginLayoutParams是什么呢，从名称上也能看出MarginLayoutParams和外边距有关，相比于ViewGroup.LayoutParams，它添加了对外边距的支持，我们平时在xml文件中使用的**layout_margin**和**layout_marginLeft**等属性都是MarginLayoutParams中定义的，而LinearLayout.LayoutParams就是继承自MarginLayoutParams，因此具有设置外边距的几个属性，我们自定义的LayoutParams也可以直接继承自MarginLayoutParams。

需要注意的是，虽然这里的`generateLayoutParams()`方法可以保证LayoutParams类型的正确，但是如果在调用`addView()`方法后再次设置了LayoutParams，就有可能会报错，就像下面这样：

```java
LinearLayout llContainer = findViewById(R.id.ll_container);
View child = new View(this);
llContainer.addView(child);
// 在addView()方法后设置了不正确的LayoutParams
child.setLayoutParams(new ViewGroup.LayoutParams(child.getLayoutParams()));
```

上述代码运行后会抛出异常**java.lang.ClassCastException: android.view.ViewGroup.LayoutParams cannot be cast to android.widget.LinearLayout.LayoutParams**，原因是在LinearLayout的`onMeasure()`方法内部会进行强制类型转换操作：

```java
final LayoutParams lp = (LayoutParams) child.getLayoutParams();
```

因此我们在调用`setLayoutParams()`还是应该保证传入的LayoutParams类型正确。

## 1.3.layout



## 1.4.draw



## 2.自定义View须知

**本文参考自《Android开发艺术探索》**

自定义View的过程中有一些需要注意的地方，处理不好可能会导致设置的属性不生效等问题，影响View的显示和使用。

### 1.支持wrap_content

自定义View需要重写`onMeasure()`方法来支持`wrap_content`属性，否则View的显示效果会和`match_parent`一样，即占满父布局，原因可以查看View的`onMeasure()`方法。

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    // 获取宽-测量规则的模式和大小
    int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
    int widthSpecSize = MeasureSpec.getSize(widthMeasureSpec);
    // 获取高-测量规则的模式和大小
    int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
    int heightSpecSize = MeasureSpec.getSize(heightMeasureSpec);

    // 设置wrap_content的默认宽/高值，可以根据需要设置
    if (widthSpecMode == MeasureSpec.AT_MOST && heightSpecMode == MeasureSpec.AT_MOST) {
        setMeasuredDimension(200, 200);
    } else if (widthSpecMode == MeasureSpec.AT_MOST) {
        setMeasuredDimension(200, heightSpecSize);
    } else if (heightSpecMode == MeasureSpec.AT_MOST) {
        setMeasuredDimension(widthSpecSize, 200);
    }
}
```

### 2.支持padding

如果是直接继承自View需要在`onDraw()`方法中处理`padding`的值，否则不会生效。如果是继承自ViewGroup需要在`onMeasure()`和`onLayout()`方法中考虑`padding`和子View的`margin`属性影响，否则`padding`和子View的`margin`不会生效。

```java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    // 获取设置的padding值
    final int paddingLeft = getPaddingLeft();
    final int paddingRight = getPaddingRight();
    final int paddingTop = getPaddingTop();
    final int paddingBottom = getPaddingBottom();

    // 获取绘制内容实际的高度和宽度，考虑四个方向的padding值
    int width = getWidth() - paddingLeft - paddingRight;
    int height = getHeight() - paddingTop - paddingBottom;

    int radius = Math.min(width / 2, height / 2);
    canvas.drawCircle(paddingLeft + width / 2, paddingTop + height / 2, radius, mPaint);
}
```

### 3.避免使用Handler

View的内部本身提供了post系列的方法，完全可以替代Handler的作用，除非很明确地需要Handler来发送消息。

### 4. 防止内存泄漏

当View退出或不可见时需要及时停止线程和动画，否则可能会导致内存泄漏。可以在`onDetachedFromWindow()`方法中来处理，当包含此View的Activity退出或当前View被remove时会被调用。与此方法对应的是`onAttachedToWindow()` ，该方法会在包含此View的Activity启动时被调用。

### 5.处理好滑动冲突

当自定义View涉及到嵌套滑动时需要处理好滑动冲突。

## 3.invalidate和requestLayout的区别

**a|b: 添加标志位b;**

**(a&b)!=0: 判断是否有标志位b;**

**a&~b:清除标志位b;**

**a^b: 取出a与b的不同部分;**

### 1.invalidate

```java

```



### 2.requestLayout

```java

```



## 4.View获取宽高


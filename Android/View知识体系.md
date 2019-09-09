# View知识体系

[TOC]

> **前言**
>
> View可以说是我们在Android开发中接触得最多的一个类了，虽然不属于四大组件，但是发挥的作用却一点都不亚于四大组件，页面中的各种控件、布局都直接或间接地继承自View，可以说View无处不在。因而了解View的工作原理能让我们更好地处理开发中的诸多问题，尤其是对于老生常谈的自定义View来说，View的工作原理更是必须要掌握的。

在进入正文之前还是要强调一下，**本文的分析基于Android 9.0（API Level 28）的源码**，不同版本的源码可能会有不同，但是基本思路不会变化太多，可以进行参考。

## 1.View的工作原理

### 1.1.几个相关类

在介绍View的工作原理之前首先要介绍几个相关的类，它们在View的工作流程中扮演了重要的角色，了解它们能让我们对于View的工作原理有一个更全面的认识。

#### 1.1.1.Window和WindowManager

**Window**顾名思义是表示一个窗口，虽然我们在开发中可能很少直接去操作Window，我目前接触过的也仅仅是给Window添加一些Flag的操作，但是Window在Android的视图体系中其实是很重要的一环，它可以说是所有视图的**承载器**，我们最熟悉的Activity的视图实际上也是附加在Window上，通过Window来管理的。Window是一个抽象类，它只有一个实现类**PhoneWindow**，因此我们在分析源码时直接看PhoneWindow就可以了。

**WindowManager**可以译为窗口管理者，是外界访问Window的入口，我们可以通过WindowManager来操作Window。WindowManager是一个接口，它的实现类是**WindowManagerImpl**。

#### 1.1.2.DecorView

**DecorView**是最顶层的View，是整个视图的根节点，继承自FrameLayout，因此它也是一个ViewGroup。下面以一张图来展示可能更直观一些。

![](https://github.com/StephenZKCurry/Android-Study-Notes/blob/master/images/Android%E8%A7%86%E5%9B%BE%E5%B1%82%E7%BA%A7.jpg?raw=true)

DecorView下包含一个竖直方向的LinearLayout，它的内部根据页面主题的不同可能会有所不同，但是一定会包含一个子View，它的id为**android.R.id.content**，是一个FrameLayout，我们调用`setContentView()`设置的布局就是添加到了这个contentView中。

#### 1.1.3.ViewRoot

ViewRoot对应于**ViewRootImpl**，是连接WindowManager和DecorView的纽带，View的measure、layout和draw流程都是通过ViewRootImpl完成的。

### 1.2.准备阶段

这里将以下几个流程称作View工作流程的准备阶段，可能不是很确切，主要还是为了和我们熟知的measure、layout、draw三大流程区分开，这一阶段完成的工作是Window和DecorView的创建以及对三大流程的调用。

#### 1.2.1.Window的创建

Window的创建时机是在**ActivityThread**的`performLaunchActivity()`方法中，在之前Activity的启动流程中也分析过该方法，我们再来简单回顾一下：

**ActivityThread的performLaunchActivity方法**

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // ...
    java.lang.ClassLoader cl = appContext.getClassLoader();
    // 1.创建Activity对象
    activity = mInstrumentation.newActivity(
            cl, component.getClassName(), r.intent);
    // ...
    // 2.创建Application对象，如果已经创建则不会重复创建
    Application app = r.packageInfo.makeApplication(false, mInstrumentation);
    // ...
    if (activity != null) {
        // ...
        Window window = null;
        if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
            window = r.mPendingRemoveWindow;
            r.mPendingRemoveWindow = null;
            r.mPendingRemoveWindowManager = null;
        }
        appContext.setOuterContext(activity);
      	// 3.创建PhoneWindow
        activity.attach(appContext, this, getInstrumentation(), r.token,
                r.ident, app, r.intent, r.activityInfo, title, r.parent,
                r.embeddedID, r.lastNonConfigurationInstances, config,
                r.referrer, r.voiceInteractor, window, r.configCallback);

        // ...
      	// 4.调用onCreate()方法
        if (r.isPersistable()) {
            mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
        } else {
            mInstrumentation.callActivityOnCreate(activity, r.state);
        }
        // ...
        r.activity = activity;
    }
    // ...
    return activity;
}
```

该方法内部会依次创建Activity对象和Application对象，最后通过Instrumentation对象调用Activity的`onCreate()`方法，这些都不是我们这里要关注的，我们只需要分析Activity的`attach()`方法。

**Activity的attach方法**

```java
final void attach(Context context, ActivityThread aThread,
                  Instrumentation instr, IBinder token, int ident,
                  Application application, Intent intent, ActivityInfo info,
                  CharSequence title, Activity parent, String id,
                  NonConfigurationInstances lastNonConfigurationInstances,
                  Configuration config, String referrer, IVoiceInteractor voiceInteractor,
                  Window window, ActivityConfigCallback activityConfigCallback) {
    // ...
    // 创建PhoneWindow
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
  	// 设置回调
  	mWindow.setWindowControllerCallback(this);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    // ...
  	// 设置WindowManager
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    mWindowManager = mWindow.getWindowManager();
    // ...
}
```

可以发现PhoneWindow对象就是在`attach()`方法中创建的，之后会为PhoneWindow设置相关回调并创建WindowManager对象（实际上是WindowManagerImpl对象）。

#### 1.2.2.DecorView的创建

在Activity的`onCreate()`方法中我们会调用`setContentView()`来设置页面的布局，DecorView的创建就要从该方法来分析。

**Activity的setContentView方法**

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

`getWindow()`方法获取到的就是上面创建好的PhoneWindow对象，我们接着来看PhoneWindow的`setContentView()`方法：

**PhoneWindow的setContentView方法**

```java
@Override
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```

这里首先会判断mContentParent是否为空，如果为空就调用`installDecor()`方法，否则调用`removeAllViews()`方法移除mContentParent的所有子View。那么这个mContentParent是什么呢，它是一个ViewGroup，我们通过名称可能能够猜到它就是我们页面所展示的内容。全局搜索了一下mContentParent的赋值时机，发现只有在`installDecor()`方法中才会对其赋值，因此这里mContentParent为空，我们来看看`installDecor()`方法：

```java
private void installDecor() {
    mForceDecorInstall = false;
    if (mDecor == null) {
        mDecor = generateDecor(-1);
        // ...
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor);
        // ...
    }
}

protected DecorView generateDecor(int featureId) {
    Context context;
    if (mUseDecorContext) {
        Context applicationContext = getContext().getApplicationContext();
        if (applicationContext == null) {
            context = getContext();
        } else {
            context = new DecorContext(applicationContext, getContext());
            if (mTheme != -1) {
                context.setTheme(mTheme);
            }
        }
    } else {
        context = getContext();
    }
    return new DecorView(context, featureId, this, getAttributes());
}

protected ViewGroup generateLayout(DecorView decor) {
    TypedArray a = getWindowStyle();
    // ...
    int layoutResource;
    // 根据Features（通过requestFeature()方法添加，可以看做是Window的主题样式）设置相应的布局
    // 伪代码
    if () {
        layoutResource =R.layout.xx1;
    } else if () {
        layoutResource =R.layout.xx2;
    } else {
        layoutResource =R.layout.xx;
    }
	// 为DecorView加载布局
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

    ViewGroup contentParent = (ViewGroup) findViewById(ID_ANDROID_CONTENT);
    // ...
    return contentParent;
}
```

可以看到`installDecor()`方法主要做的事有两件：调用`generateDecor()`方法创建DecorView，将返回值赋给mDecor；调用`generateLayout()`方法创建一个ViewGroup，赋值给mContentParent。这里的`generateDecor()`和`generateLayout()`方法都省略了大量代码，只保留了最核心的部分。

创建好了DecorView和mContentParent之后，我们回到PhoneWindow的`setContentView()`方法，可以发现这之后调用了`mLayoutInflater.inflate(layoutResID, mContentParent)`，`inflate()`方法的作用我们都很熟悉了，就是根据我们传入的布局文件构建出View树，这里调用的是两个参数的方法，因此会将创建好的View树添加到mContentParent中。如果不是很清楚`inflate()`方法几个参数的意义可以查阅网上的相关文章，或者参考我之前写过的一篇文章[LayoutInflate的使用](https://github.com/StephenZKCurry/Android-Study-Notes/blob/master/Android/LayoutInflate%E7%9A%84%E4%BD%BF%E7%94%A8.md)，这里我就不具体讲了。现在我们就清楚了mContentParent是什么了吧，它是我们`setContentView()`方法中指定布局的父View，指定的布局会作为一个子View添加到mContentParent中。

我一开始分析时有一个疑问，mContentParent是何时与DecorView产生联系的呢，分析到这里好像并没有看到诸如`mDecor.addView()`之类的代码，我们回到`generateLayout()`方法中，方法内部调用了`findViewById()`方法获取到contentParent，并将其赋给mContentParent，而这个id为content，是DecorView中的一个子View，是一个Framelayout，因此contentParent就是DecorView中的这个子View，mContentParent自然也表示这个子View。`generateLayout()`方法中会根据Window的主题样式为DecorView加载相应的布局，我们就以其中一个**R.layout.screen_simple**为例，看看它的布局层级关系：

**screen_simple.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

可以看到最外层是一个LinearLayout，内部有两个子View，一个是**ViewStub**，它的id为**action_mode_bar_stub**，从名称上大概可以猜出它是页面的标题栏，会根据主题的不同来选择是否加载；另一个子View是id为**content**的**FrameLayout**，也就是上面分析中`findViewById()`方法获取到的contentParent，因此mContentParent就是这个FrameLayout。

看到这里，我们对于整个页面的层级关系就很清楚了，最外层是一个DecorView，它的内部有一个LinearLayout，LinearLayout中有一个FrameLayout，我们在`setContentView()`中指定的布局文件会被添加到这个FrameLayout中，当然实际上根据主题样式的不同可能会更复杂一些，这里只是说明最简单的一种情况。

值得一提的是，**AppCompatActivity**中的`setContentView()`方法和Activity的有所不同，简单分析一下。

**AppCompatActivity的setContentView方法**

```java
@Override
public void setContentView(@LayoutRes int layoutResID) {
    getDelegate().setContentView(layoutResID);
}
```

`getDelegate()`获取到的是一个代理对象，类型为**AppCompatDelegateImpl**（前几个版本的源码中这里会根据SDK版本不同返回不同的对象如**AppCompatDelegateImplN**、**AppCompatDelegateImplV23**等，后面的分析是差不多的），之后调用了AppCompatDelegateImpl的`setContentView()`方法。

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

方法内部首先调用了`ensureSubDecor()`方法，将返回值赋值给mSubDecor：

```java
private void ensureSubDecor() {
    if (!mSubDecorInstalled) {
        mSubDecor = createSubDecor();
        // ...
    }
}

private ViewGroup createSubDecor() {
    TypedArray a = mContext.obtainStyledAttributes(R.styleable.AppCompatTheme);
    // ...
	// 分析1
    mWindow.getDecorView();

    final LayoutInflater inflater = LayoutInflater.from(mContext);
    ViewGroup subDecor = null;

    // 根据主题为subDecor加载布局
    // 伪代码
    if () {
        subDecor = (ViewGroup) LayoutInflater.from(themedContext)
                .inflate(R.layout.xx, null);
    } else {

    }
  
    // ...
  	// 分析2
    final ContentFrameLayout contentView = (ContentFrameLayout) subDecor.findViewById(
            R.id.action_bar_activity_content);

    final ViewGroup windowContentView = (ViewGroup) mWindow.findViewById(android.R.id.content);
    if (windowContentView != null) {
        // 将windowContentView的子View移除并添加到contentView中
        while (windowContentView.getChildCount() > 0) {
            final View child = windowContentView.getChildAt(0);
            windowContentView.removeViewAt(0);
            contentView.addView(child);
        }

        
        windowContentView.setId(View.NO_ID);
        contentView.setId(android.R.id.content);
        // ...
    }
    
  	// 分析3
    mWindow.setContentView(subDecor);
    // ...
    return subDecor;
}
```

`ensureSubDecor()`方法内部调用了`createSubDecor()`方法，我们具体分析一下该方法。

**分析1、mWindow.getDecorView()**

mWindow的类型为PhoneWindow，通过查看PhoneWindow的`getDecorView()`方法我们可以发现，由于此时mDecor并未被赋值过，因此会调用此前分析过的`installDecor()`方法，创建DecorView和mContentParent。

```java
@Override
public final View getDecorView() {
    if (mDecor == null || mForceDecorInstall) {
        installDecor();
    }
    return mDecor;
}
```

之后和`generateLayout()`方法很类似，通过判断主题样式创建出subDecor，加载不同的布局文件。

**分析2、“偷梁换柱”**

这里的逻辑还是很巧妙的，涉及到了两个View，contentView是subDecor中id为**action_bar_activity_content**的子View，类型为ContentFrameLayout；另一个windowContentView的id为**android.R.id.content**，没错，就是此前分析过的那个FrameLayout。之后的操作是将windowContentView的子View移除，添加到contentView中，并将windowContentView的id设置为View.NO_ID，将contentView的id设置为android.R.id.content，看上去很像是两个View之间的互换，因此我把这个操作称为“偷梁换柱”。

**分析3、mWindow.setContentView(subDecor)**

这里调用了PhoneWindow的`setContentView()`方法，将subDecor添加到了mContentParent中，这里的mContentParent其实就是上面的windowContentView，此时它的id已经变成了View.NO_ID。

这时我们再回到AppCompatDelegateImpl的`setContentView()`方法，之后就是根据我们指定的布局文件构建出View树并添加到id为**android.R.id.content**的ViewGroup中，即上面的ContentFrameLayout。

我的表述不是很清楚，大家可能有些懵了，这都是什么乱七八糟的，我最后总结一下，其实大体上的流程和Activity的`setContentView()`方法还是很类似的，不同之处就是在Activity中，我们指定布局文件对应的View树会被添加到FrameLayout中，而对于AppCompatActivity，View树会被添加到一个ContentFrameLayout中，它们的id均为**android.R.id.content**，层级关系如下，ContentFrameLayout的直接父View是mSubDecor所加载布局的根布局，对应不同的主题样式可能不同，这个根布局的父View就是FrameLayout，因此可以看做就是多嵌套了几层View。

其实这里我就不明白了，我们的Activity都是继承自AppCompatActivity，那么相比于继承自Activity，我们的布局层级是要复杂一些的，大家都知道Android开发中是要避免过多的布局层级嵌套的，那么AppCompatActivity这样做的目的是什么呢？鉴于个人能力和认识都还很浅显，想不明白为什么要这样设计，欢迎大佬们提出自己的见解。

#### 1.2.3.三大流程的调用

上面的两个流程中已经完成了PhoneWindow和DecorView的创建，那么大名鼎鼎的View绘制三大流程又是从何时开始的呢，就是ActivityThread的`handleResumeActivity()` 方法，该方法在分析Activity的启动流程时也分析过，`onResume()`回调方法就是经由该方法调用的。

**ActivityThread的handleResumeActivity方法**

```java
@Override
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
                                 String reason) {
    // ...
    // 方法内部调用Activity的onResume()方法
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    // ...
    final Activity a = r.activity;
    // ...
    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
        // 获取DecorView
        View decor = r.window.getDecorView();
        // 将DecorView设置为不可见
        decor.setVisibility(View.INVISIBLE);
        // 获取WindowManager
        ViewManager wm = a.getWindowManager();
        WindowManager.LayoutParams l = r.window.getAttributes();
        // 为Activity的mDecor对象赋值
        a.mDecor = decor;
        // ...
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                // 将DecorView添加到Window中
                wm.addView(decor, l);
            } else {
                a.onWindowAttributesChanged(l);
            }
        }
    }
    // ...
    if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
        // ...
        if (r.activity.mVisibleFromClient) {
            // 将DecorView设置为可见
            r.activity.makeVisible();
        }
    }
}
```

该方法主要做了两件事：调用`performResumeActivity()`方法，进而调用Activity的生命周期回调`onResume()`；获取此前创建好的PhoneWindow、DecorView以及WindowManager对象，调用WindowManager的`addView()`方法将DecorView添加到Window中。前面也说过，WindowManager的实现类是WindowManagerImpl，因此我们来具体看一下WindowManagerImpl的`addView()`方法都做了些什么。

**WindowManagerImpl的addView方法**

```java
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

方法内部调用了mGlobal的`addView()`方法，mGlobal的类型为WindowManagerGlobal，我们接着看：

**WindowManagerGlobal的addView方法**

```java
public void addView(View view, ViewGroup.LayoutParams params,
                    Display display, Window parentWindow) {
    // ...
    ViewRootImpl root;
    // ...
    // 创建ViewRootImpl对象
    root = new ViewRootImpl(view.getContext(), display);
  
    view.setLayoutParams(wparams);

    mViews.add(view);
    mRoots.add(root);
    mParams.add(wparams);

    try {
      	// 核心代码
        root.setView(view, wparams, panelParentView);
    } catch (RuntimeException e) {
        // BadTokenException or InvalidDisplayException, clean up.
        if (index >= 0) {
            removeViewLocked(index, true);
        }
        throw e;
    }
}
```

`addView()`方法内部首先会创建出ViewRootImpl对象，将要添加的View（即DecorView）、ViewRootImpl和布局参数添加到列表中，最后调用ViewRootImpl的`setView()`方法，我们来看一下这个方法。

**ViewRootImpl的setView方法**

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;
            // ...
            requestLayout();
            // ...
        }
    }
}
```

`setView()`方法内部调用了`requestLayout()`方法，这个方法可能大家在自定义View时用到过，用于刷新视图，不过需要注意的是，我们在自定义View中调用的`requestLayout()`方法是在View中定义的，和ViewRootImpl中的逻辑是不一样的，之后会详细分析一下。接下来我们就来看看ViewRootImpl的`requestLayout()`方法：

```java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

方法内部又调用了`scheduleTraversals()`方法：

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
      	// 开启同步屏障机制
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
      	// 利用Handler发送一条异步消息
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        // ...
    }
}
```

`scheduleTraversals()`方法内部首先执行了`mHandler.getLooper().getQueue().postSyncBarrier()`，这行代码的作用是开启Handler的同步屏障机制，关于Handler的同步屏障机制，我这里简单解释一下（因为我也不是很了解o(╥﹏╥)o），我们都知道Handler处理的消息是放到一个消息队列中的，消息默认情况下都是同步的，如果需要发送异步消息需要使用代码来声明，同步屏障机制就使得Looper在从消息队列中获取消息时，只获取异步消息并进行处理。我可能解释得不太好，如果想深入了解一下Handler的同步屏障机制可以自行查找资料，这里推荐一下鸿洋大神WanAndroid上的每日一问[Handler应该是大家再熟悉不过的类了，那么其中有个同步屏障机制，你了解多少呢？](https://www.wanandroid.com/wenda/show/8710)。开启了同步屏障后，调用了mChoreographer的`postCallback()`方法，该方法内部就是利用了Handler，发送了一个Runable对象，如果跟踪源码的话可以发现最后会把Runable封装为一个Message，并将Message设置为异步消息，我就不展示了。清楚了`postCallback()`方法的原理后，我们就知道了需要分析mTraversalRunnable对象，它是一个Runable对象，类型为**TraversalRunnable** ，在`run()`方法中调用了`doTraversal()`方法。

```java
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
```

由于开启了同步屏障，因此当前线程（主线程）的Looper会优先获取异步消息，即直接执行`doTraversal()`方法。

```java
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
      	// 关闭同步屏障
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        // ...
        performTraversals();
        // ...
    }
}
```

`doTraversal()`方法内部首先会关闭同步屏障机制，否则主线程的同步消息就永远无法被处理了，然后调用了`performTraversals()`方法，我们来看一下：

```java
private void performTraversals() {
    // ...
    // 调用performMeasure()方法开始measure流程
    measureHierarchy(host, lp, res,
            desiredWindowWidth, desiredWindowHeight);
    // ...
    // 开始layout流程
    performLayout(lp, mWidth, mHeight);
    // ...
    // 开始draw流程
    performDraw();
    // ...
}
```

这里省略了大量代码，可以看出，在`performTraversals()`方法内会依次调用`measureHierarchy()` 、`performLayout()`、`performDraw()`，进而开始View的三大流程。

分析到这里，View的绘制准备阶段就算完成了，最后再回顾一下，主要分为三个阶段：

* Activity的`onCreate()`方法调用之前，创建Window（PhoneWindow）
* Activity的`onCreate()`方法中调用`setContentView()`方法，创建DecorView和contentView（页面内容根布局），将指定的布局文件加载到contentView中
* Activity的`onResume()`方法调用之后，将DecorView添加到Window中，之后依次开始View的measure、layout和draw流程

从上面几个流程的先后顺序我们就能清楚为什么在`onResume()`方法中或者`onResume()`方法之前获取不到View的宽高，就是因为此时View还未执行measure和layout流程。

### 1.3.measure

到这里算是正式进入到了View的三大流程，首先要分析的是measure流程。在分析View的measure流程之前，我们首先要介绍两个相关的类：**MeasureSpec**和**LayoutParams**。

#### 1.3.1.MeasureSpec

##### 1.3.1.1.MeasureSpec简介

关于**MeasureSpec**大家可能都很熟悉了，它是由一个32位int值表示的，高2位表示**SpecMode**，即测量模式，低30位表示**SpecSize**，即测量尺寸大小。

```java
public static class MeasureSpec {
    private static final int MODE_SHIFT = 30;
    private static final int MODE_MASK = 0x3 << MODE_SHIFT;

    // ...
	
  	/**
     * 三种测量模式
     */
    public static final int UNSPECIFIED = 0 << MODE_SHIFT;

    public static final int EXACTLY = 1 << MODE_SHIFT;

    public static final int AT_MOST = 2 << MODE_SHIFT;

    /**
     * 根据测量尺寸和测量模式生成MeasureSpec
     */
    public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size,
                                      @MeasureSpecMode int mode) {
        if (sUseBrokenMakeMeasureSpec) {
            return size + mode;
        } else {
            return (size & ~MODE_MASK) | (mode & MODE_MASK);
        }
    }

  	/**
     * 获得测量模式
     */
    @MeasureSpecMode
    public static int getMode(int measureSpec) {
        //noinspection ResourceType
        return (measureSpec & MODE_MASK);
    }

  	/**
     * 获得测量尺寸
     */
    public static int getSize(int measureSpec) {
        return (measureSpec & ~MODE_MASK);
    }

    // ...
}
```

可以通过调用`getMode()`和`getSize()`方法获取到测量模式和测量尺寸，方法内部就是通过简单的位运算保留指定位数上的数值。不由得称赞Android系统开发人员的设计巧妙，将两个值封装成了一个变量，可以通过位运算获取相应数值，减少了多余对象的内存分配，其实Android源码中很多地方都有类似设计（比如MotionEvent），这里就不多提啦。

MeasureSpec内部定义了三种测量模式：

* **UNSPECIFIED**：父View不会限制子View的大小，一般用于系统内部，开发中使用很少
* **EXACTLY**：父View能够确定子View的大小，对应两种情况：**精确尺寸（dp或px）**和**match_parent**
* **AT_MOST**：子View的大小不能超过父View尺寸，具体尺寸需要由子View自身来确定，对应**wrap_content**

虽然我们在开发中用到**UNSPECIFIED**模式的情况不多，但是了解一下还是有必要的，我在后面会单独介绍一下这个模式的应用。

##### 1.3.1.2.如何确定MeasureSpec的值

MeasureSpec的值是由**View自身的LayoutParams**和**父View的MeasureSpec**共同确定的。对于特定的View来说，它的MeasureSpec是通过父View（即ViewGroup）的`getChildMeasureSpec()`方法得到的。

**ViewGroup的getChildMeasureSpec方法**

```java
/**
 * 获得子View的MeasureSpec
 *
 * @param spec           父View的MeasureSpec
 * @param padding        父View的padding
 * @param childDimension 子View的LayoutParams指定的宽/高
 * @return
 */
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    // 获得父View的测量模式和测量尺寸
  	int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);
	
  	// 父View实际可用大小
    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
        // 父View的测量模式为EXACTLY，即match_parent或精确尺寸
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
              	// 子View的LayoutParams指定为精确的值
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
              	// 子View的LayoutParams指定为MATCH_PARENT
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
              	// 子View的LayoutParams指定为WRAP_CONTENT
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // 父View的测量模式为AT_MOST，即wrap_content
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // 父View的测量模式为UNSPECIFIED
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
    }
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

`getChildMeasureSpec()`方法也验证了子View的MeasureSpec是由父View的MeasureSpec和子View的LayoutParams共同确定的。上面的判断可能有些复杂，不过别担心。已经有很多大佬总结出了表格，看起来更加直观一些，下图摘自[Carson_Ho大佬的博客](https://www.jianshu.com/p/1dab927b2f36)，表中的childSize表示子View的LayoutParams指定的大小，parentSize表示父View可用空间的大小。

![](https://upload-images.jianshu.io/upload_images/944365-76261325e6576361.png?imageMogr2/auto-orient/)

我们可以先不去看最后一列**UNSPECIFIED**的情况，单看前两列可以找出一定的规律：

* 当子View的LayoutParams指定为精确数值时，不管父View的测量模式是什么，子View的测量模式均为**EXACTLY**，测量尺寸为LayoutParams指定的值
* 当子View的LayoutParams指定为match_parent时，子View的测量模式取决于父View，即如果父View的测量模式为**EXACTLY**，那么子View的测量模式为**EXACTLY**；如果父View的测量模式为**AT_MOST**，那么子View的测量模式为**AT_MOST**，子View的测量尺寸均为父View可用空间大小
* 当子View的LayoutParams指定为wrap_content时，不管父View的测量模式是什么，子View的测量模式均为**AT_MOST**，测量尺寸为父View可用空间大小

普通View的MeasureSpec是如何获取的我们已经清楚了，那么对于DecorView来说，它是没有父View的，它的MeasureSpec是如何得到的呢？我们在上一节分析到ViewRootImpl的`performTraversals()`方法时，介绍到方法内部调用了`measureHierarchy()`方法，进而调用`performMeasure()`方法开始View的measure流程，现在我们就来具体看一下`measureHierarchy()`方法。

**ViewRootImpl的measureHierarchy方法**

```java
private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
                                 final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
    int childWidthMeasureSpec;
    int childHeightMeasureSpec;
  	boolean windowSizeMayChange = false;
    // ...
	
    childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
    childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    // ...
    return windowSizeMayChange;
}

private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    if (mView == null) {
        return;
    }
    // ...
    mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

可以看出这里调用`getRootMeasureSpec()`方法获取到childWidthMeasureSpec和childHeightMeasureSpec，之后调用`performMeasure()`方法，方法内部又调用了mView的`measure()`方法，这个mView是什么呢，我全文检索了一下mView的赋值时机，发现它是在`setView()`方法中被赋值的，还记得`setView()`方法是什么时候调用的吗，就是在`handleResumeActivity()`方法中调用`wm.addView(decor,l)`这行代码之后被调用的，因此这里的mView就是传过来的DecorView，调用`measure()`方法就开始了对DecorView的测量流程。现在就要关注childWidthMeasureSpec和childHeightMeasureSpec了，这两个值就是DecorView的MeasureSpec，我们来看一下获取到这两个值的`getRootMeasureSpec()`方法：

```java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {
        case ViewGroup.LayoutParams.MATCH_PARENT:
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
    }
    return measureSpec;
}
```

逻辑还是比较简单的，参数windowSize传递过来的值是desiredWindowWidth和desiredWindowHeight，通过查看源码可以发现这两个值表示屏幕的宽高尺寸，因此我们可以得出以下结论：

* DecorView的LayoutParams指定为**MATCH_PARENT**时，它的测量模式为**EXACTLY**，测量尺寸为屏幕尺寸
* DecorView的LayoutParams指定为**WRAP_CONTENT**时，它的测量模式为**WRAP_CONTENT**，测量尺寸为屏幕尺寸

可以看出，DecorView作为最顶层的View，它的MeasureSpec只取决于自己的LayoutParams参数。

#### 1.3.2.LayoutParams

##### 1.3.2.1.LayoutParams简介

**LayoutParams**这个类在开发中还是很常见的，顾名思义就是布局参数，View中定义了一个LayoutParams类型的成员变量，它的作用就是确定View的宽高，我们平时在xml布局文件中指定的**layout_width**和**layout_height**属性就是用于生成LayoutParams。需要注意的是，这两个属性的前面都带上layout前缀，而不是直接使用**width**和**height**来命名，因此我们要清楚它们的值并不是View的宽高，也可以说它们并不属于View自身的属性。

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

    // ...
}
```

LayoutParams中定义了几个重载构造函数，分别用于xml文件中指定宽高、手动指定宽高等场景。每个ViewGroup的子类（直接或间接继承）都有对应的LayoutParams类，比如**LinearLayout.LayoutParams**，在各自的LayoutParams中可以定义相应的布局参数属性。因此不止**layout_width**和**layout_height**这两个属性，其他以layout开头的属性（比如**layout_weight**、**layout_margin**等等）也都和LayoutParams相关。

##### 1.3.2.2.View的LayoutParams属性是何时设置的

了解了LayoutParams的定义后，接下来需要弄清楚View的LayoutParams属性是何时设置的，我们知道在ViewGroup中添加子View的方式有两种：xml文件中添加和代码中添加，我们分别来看一下这两种情况。

* **xml文件中添加View**

在`setContentView()`方法的分析中我们知道xml文件中添加的View最终是通过**LayoutInflater**的`inflate()`方法来解析的。

```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        // ...
        View result = root;
        int type;
        while ((type = parser.next()) != XmlPullParser.START_TAG &&
                type != XmlPullParser.END_DOCUMENT) {
        }
        // ...
        final String name = parser.getName();
        // ...
       
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
    // ...
    return result;
}
```

在`inflate()`方法中有一行代码是`params = root.generateLayoutParams(attrs);`，`generateLayoutParams()`方法的作用就是就是根据xml指定的属性构造出LayoutParams对象。

```java
public LayoutParams generateLayoutParams(AttributeSet attrs) {
    return new LayoutParams(getContext(), attrs);
}
```

构造出LayoutParams对象后根据参数**attachToRoot**的值有两种处理逻辑：

* 如果**attachToRoot**为true，则会调用`addView()`方法并传入构造好的LayoutParams对象，`addView()`方法内部会将LayoutParams对象设置给View，详细代码后面会展示，这里先这样记住就好
* 如果**attachToRoot**为false，则会调用View的`setLayoutParams()`方法直接将构造好的LayoutParams对象设置给View

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
    // ...
    if (!checkLayoutParams(params)) {
        params = generateLayoutParams(params);
    }

    if (preventRequestLayout) {
        child.mLayoutParams = params;
    } else {
        child.setLayoutParams(params);
    }
    // ...
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

##### 1.3.2.3.自定义LayoutParams须知

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

#### 1.3.3.measure流程

前面关于两个类的介绍还是比较详细的，现在终于进入到了measure流程的分析，这里会分为两种情况：单一View的measure和ViewGroup的measure，ViewGroup的measure要复杂一些，因为它不仅需要完成对自身的measure，还要完成对所有子View的measure，我们先分析简单的情况——单一View的measure流程。

##### 1.3.3.1.单一View的measure流程

View的measure流程从`measure()`方法开始：

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    // ...
    onMeasure(widthMeasureSpec, heightMeasureSpec);
    // ...
}
```

可以看到它是一个final声明的方法，因此子类无法重写该方法。在方法内部又调用了我们熟悉的`onMeasure`方法，我们自定义View时重写的都是该方法。

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

方法内部调用了`setMeasuredDimension()`方法：

```java
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
    // ...
    setMeasuredDimensionRaw(measuredWidth, measuredHeight);
}

private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
    mMeasuredWidth = measuredWidth;
    mMeasuredHeight = measuredHeight;
	// ...
}
```

可以看到`setMeasuredDimension()`方法完成的工作就是为mMeasuredWidth和mMeasuredHeight这两个变量赋值，这两个变量表示View的测量宽高（**与实际宽高有区别，View的实际宽高还取决于layout过程**），我们可以通过`getMeasuredWidth()`和`getMeasuredHeight()`方法获取到View测量后的宽高尺寸，即这两个变量的低30位。

我们接着来看View的测量宽高是如何得到的，即`getDefaultSize()`方法：

```java
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
    }
    return result;
}
```

可以看出当View的测量模式为**AT_MOST**或**EXACTLY**时，View的测量宽/高等于specSize，即MeasureSpec中的测量尺寸；当View的测量模式为**UNSPECIFIED**时，View的测量宽/高等于该方法的第一个参数的值，即`getSuggestedMinimumWidth()`/`getSuggestedMinimumHeight()`方法的返回值，这里就以`getSuggestedMinimumWidth()`方法为例，`getSuggestedMinimumHeight()`同理。

```java
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
```

这里首先判断了View是否设置了背景，如果没有设置背景，返回值为mMinWidth，它对应于**android:minWidth**属性所指定的值，如果没有指定则为0；如果View设置了背景，返回值为mMinWidth和`mBackground.getMinimumWidth()`两者的最大值，`getMinimumWidth()`方法可以获取到Drawable的原始宽度，但不是所有的Drawable都有原始宽度，如果没有原始宽度，获取到的值就为0（上面这段解释基本上来自《Android开发艺术探索》，目前我对于Drawable的认识还不够，想了解更多的话自行查阅资料吧）。

这里也引出了一个问题，当View的测量模式为**AT_MOST**，即LayoutParams指定为wrap_content时，View的测量宽/高等于specSize，而从`getChildMeasureSpec()`方法的分析中我们也得出此时specSize的值为parentSize，即父View的可用空间大小，这会导致wrap_content产生和match_parent一样的效果，因此我们在自定义View时需要重写`onMeasure()`方法，解决wrap_content的失效问题，具体做法我后面会介绍。

用一张图总结一下单一View的measure流程：

![](https://github.com/StephenZKCurry/Android-Study-Notes/blob/master/images/View%E7%9A%84measure%E6%B5%81%E7%A8%8B.jpg?raw=true)

##### 1.3.3.2.ViewGroup的measure流程

ViewGroup的measure流程同样从`measure()`方法开始，和View是一样，这里就不展示了，之后会调用`onMeasure()`方法，但是我们会发现ViewGroup中并没有重写`onMeasure()`方法，原因其实也不难理解，就是因为每个ViewGroup的布局方式都不一样，无法得出一个统一的实现方式，在自定义ViewGroup时需要根据想要得到的布局效果来重写`onMeasure()`方法。虽然ViewGroup没有提供`onMeasure()`方法的实现方式，但是提供了一个`measureChildren()`方法，从方法名也能猜到是用来测量ViewGroup的所有子View的，我们来看一下这个方法。

```java
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
    final int size = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < size; ++i) {
        final View child = children[i];
        if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
        }
    }
}
```

方法内部遍历所有的子View，依次调用`measureChild()`方法：

```java
protected void measureChild(View child, int parentWidthMeasureSpec,
                            int parentHeightMeasureSpec) {
    final LayoutParams lp = child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

`measureChild()`方法首先调用此前分析过的`getChildMeasureSpec()`方法，根据ViewGroup的MeasureSpec和子View自身的LayoutParams确定出子View的MeasureSpec，然后调用子View的`measure()`方法，对子View进行测量，后面的流程就和单一View的measure流程一样了。我们在自定义ViewGroup时可以在`onMeasure()`方法调用`measureChildren()`方法完成对子View的测量。

下面以ViewGroup的子类LinearLayout为例，分析一下它的measure流程，加深一下对ViewGroup的measure流程的理解。首先来看`onMeasure()`方法：

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    if (mOrientation == VERTICAL) {
        measureVertical(widthMeasureSpec, heightMeasureSpec);
    } else {
        measureHorizontal(widthMeasureSpec, heightMeasureSpec);
    }
}
```

`onMeasure()`方法中会判断LinearLayout的布局方向执行相应的方法，这里就以竖直方向的`measureVertical()`为例进行分析，水平方向是类似的。

```java
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    // 用于记录竖直方向的总高度
    mTotalLength = 0;
    // ...
    float totalWeight = 0;
    final int count = getVirtualChildCount();
    // ...
    for (int i = 0; i < count; ++i) {
        final View child = getVirtualChildAt(i);
        // ...
        // 对子View进行measure
        measureChildBeforeLayout(child, i, widthMeasureSpec, 0,
                heightMeasureSpec, usedHeight);
        // 获取子View测量后的高
        final int childHeight = child.getMeasuredHeight();
        // ...
        final int totalLength = mTotalLength;
        // 每测量一个子View，mTotalLength就会增加，增加的高度包括子View和高度和竖直方向上的margin
        mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
                lp.bottomMargin + getNextLocationOffset(child));
        // ...
    }

    // ...
    // 计算竖直方向的padding
    mTotalLength += mPaddingTop + mPaddingBottom;
    int heightSize = mTotalLength;
    heightSize = Math.max(heightSize, getSuggestedMinimumHeight());
    // 完成自身高度的measure
    int heightSizeAndState = resolveSizeAndState(heightSize, heightMeasureSpec, 0);
    heightSize = heightSizeAndState & MEASURED_SIZE_MASK;
    // ...
    // 宽度的measure
    maxWidth += mPaddingLeft + mPaddingRight;
    maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
            heightSizeAndState);
    // ...
}
```

方法很长，我省略了大量代码，说一下简单的流程吧，高度上，LinearLayout首先会遍历所有子View，调用`measureChildBeforeLayout()`方法对子View进行测量，每测量一个子View，就增加mTotalLength的值，它表示LinearLayout在竖直方向上的总高度，增加的值包括子View的测量高度和子VIew竖直方向上的margin，当所有子View测量完成后，会计算LinearLayout自身的padding值，最后调用`resolveSizeAndState()`方法完成对自身高度的测量。宽度上和单一View的测量类似，不需要考虑子View，调用`resolveSizeAndState()`完成对自身宽度的测量。方法最后依然是调用`setMeasuredDimension()`设置LinearLayout的测量宽高。接下来我们来看一下LinearLayout测量自身的方法`resolveSizeAndState()`，这个方法是在View中定义的。

```java
public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
    final int specMode = MeasureSpec.getMode(measureSpec);
    final int specSize = MeasureSpec.getSize(measureSpec);
    final int result;
    switch (specMode) {
        case MeasureSpec.AT_MOST:
            if (specSize < size) {
              	// 子View的测量总高度超过了LinearLayout可用空间大小
                result = specSize | MEASURED_STATE_TOO_SMALL;
            } else {
                result = size;
            }
            break;
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        case MeasureSpec.UNSPECIFIED:
        default:
            result = size;
    }
    return result | (childMeasuredState & MEASURED_STATE_MASK);
}
```

可以看出，如果LinearLayout的测量模式为**EXACTLY**，那么最终的测量高度为specSize，与子View无关；如果LinearLayout的测量模式为**AT_MOST**，会判断子View的总高度（包括margin、paddding）是否超过了LinearLayout竖直方向上的可用空间，如果没超过则最终测量高度为子View的总高度，如果超过了则最终测量高度为specSize，并设置一个**MEASURED_STATE_TOO_SMALL**标志。

值得一提的是，LinearLayout的测量有一种特殊情况，就是对于自身的测量模式为**EXACTLY**并且子View设置了layout_weight的情况，这种情况会在后面重新进行一次子View的遍历和测量，由于这不是ViewGroup测量的通用流程，这里就不细说了，感兴趣的话可以查看一下这块的源码。

最后用一张图总结一下ViewGroup的measure流程，虽然具体到每个ViewGroup的measure流程可能会有所不同，但是这几个步骤是通用的。

![](https://github.com/StephenZKCurry/Android-Study-Notes/blob/master/images/ViewGroup%E7%9A%84measure%E6%B5%81%E7%A8%8B.jpg?raw=true)

既然ViewGroup和View的measure流程都已经分析完了，我们可以梳理一下一个页面的完整measure流程，首先从ViewRootImpl的`performMeasure()`方法开始对顶层View——DecorView进行测量，调用`measure()`方法，由于DecorView继承自FrameLayout，可以看做一个ViewGroup，因此接着会遍历DecorVIew的所有子View进行测量，如果子View是一个单一View，只需要完成自身的测量，如果子View是一个ViewGroup，就又会重复上面的步骤，遍历该子View下的所有子View进行测量，之后便是一个递归的过程，最后当所有子View的测量都完成后，再进行DecorVIew自身的测量。

#### 1.3.4.补充介绍

##### 1.3.4.1.MeasureSpec.UNSPECIFIED的应用

我们此前介绍**UNSPECIFIED**模式的时候基本上是一笔带过的，只介绍了该模式下是父View不限制子View大小，用于系统内部，开发中一般很少会用到，虽然是这样，我们还是有必要了解一下该模式的常见应用场景，可能我们平时在开发中已经接触过了，只是没有发现而已。

ScrollView相信大家都很熟悉了，在使用时有一个需要注意的地方，就是当ScrollView的子布局没有占满屏幕高度时，它的子View是无法占满全屏的，即使设置了layout_height为**match_parent**也不管用，可能大家都已经知道了这个问题，我这里就简单展示一下。布局文件很简单，ScrollView嵌套一个LinearLayout，LinearLayout中有一个高度为100dp的TextView。

```xml
<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:background="#f00"
        android:orientation="vertical">

        <TextView
            android:layout_width="match_parent"
            android:layout_height="100dp"
            android:text="I can do all things"
            android:textColor="#fff"
            android:textSize="24sp" />
      
    </LinearLayout>
  
</ScrollView>
```

运行效果如下：

![](http://tva1.sinaimg.cn/large/007X8olVly1g6jsh0fgewj305k0c1aa9.jpg)

可以看出LinearLayout的高度为100dp，并没有占满屏幕，但是我们明明设置了layout_height为**match_parent**，其实不止这样，即便是layout_height指定了精确数值（如200dp）也不会生效。解决方案就是为ScrollView添加**android:fillViewport="true"**属性，运行之后发现LinearLayout可以占满全屏了。

![](http://tva1.sinaimg.cn/large/007X8olVly1g6jsimk9nnj305k0c1glr.jpg)

现在我们从源码角度分析一下产生这个问题的原因，看一下ScrollView的`onMeasure()`方法：

**ScrollView的onMeasure方法**

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    if (!mFillViewport) {
        return;
    }

    final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    if (heightMode == MeasureSpec.UNSPECIFIED) {
        return;
    }

    if (getChildCount() > 0) {
        final View child = getChildAt(0);
        final int widthPadding;
        final int heightPadding;
        final int targetSdkVersion = getContext().getApplicationInfo().targetSdkVersion;
        final FrameLayout.LayoutParams lp = (LayoutParams) child.getLayoutParams();
        if (targetSdkVersion >= VERSION_CODES.M) {
            widthPadding = mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin;
            heightPadding = mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin;
        } else {
            widthPadding = mPaddingLeft + mPaddingRight;
            heightPadding = mPaddingTop + mPaddingBottom;
        }

        final int desiredHeight = getMeasuredHeight() - heightPadding;
        if (child.getMeasuredHeight() < desiredHeight) {
            final int childWidthMeasureSpec = getChildMeasureSpec(
                    widthMeasureSpec, widthPadding, lp.width);
            final int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                    desiredHeight, MeasureSpec.EXACTLY);
            child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
    }
}
```

`onMeasure()`方法中首先会判断mFillViewport的值，如果为false则直接return，不执行后面的逻辑。从变量名不难猜到这个mFillViewport就是对应于**android:fillViewport**属性，默认值为false，因此当我们没有设置**android:fillViewport="true"**时，`onMeasure()`方法只会执行父类的`onMeasure()`方法。我们先简单看一下后面的代码，首先计算出ScrollView的可用高度desiredHeight，当`child.getMeasuredHeight() < desiredHeight`，即子View的测量高度小于ScrollView的可用高度时，会将子View高度的测量模式指定为**EXACTLY**，测量尺寸指定为ScrollView的可用高度并进行重新测量，因此子View的最终测量高度就是ScrollView的可用高度，对于上面的例子来说Linearlayout自然就占满了全屏。

清楚了**android:fillViewport="true"**属性为什么可以让子View占满全屏后，我们再来分析一下为什么默认情况下子View不会占满全屏，由于默认情况只会执行父类的`onMeasure()`方法，我们来看一下ScrollView的父类FrameLayout的`onMeasure()`方法。

**FrameLayout的onMeasure方法**

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int count = getChildCount();
    // ...

    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        // ...
        measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
        // ...
    }
    // ...
    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
            resolveSizeAndState(maxHeight, heightMeasureSpec,
                    childState << MEASURED_HEIGHT_STATE_SHIFT));
    // ...
}
```

FrameLayout的`onMeasure()`方法内部调用了`measureChildWithMargins()`方法来对子View进行测量，ScrollView重写了该方法，我们来看一下：

**ScrollView的measureChildWithMargins方法**

```java
@Override
protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed,
                                       int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int usedTotal = mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin +
            heightUsed;
    final int childHeightMeasureSpec = MeasureSpec.makeSafeMeasureSpec(
            Math.max(0, MeasureSpec.getSize(parentHeightMeasureSpec) - usedTotal),
            MeasureSpec.UNSPECIFIED);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

可以看出ScrollView在测量子View时，将子VIew高度的测量模式直接指定为了**UNSPECIFIED**，还记得我们上面分析过的LinearLayout的measure过程吗，在子View测量完成后，会调用`resolveSizeAndState()`方法完成自身的测量，这里再贴一遍代码。

```java
public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
    final int specMode = MeasureSpec.getMode(measureSpec);
    final int specSize = MeasureSpec.getSize(measureSpec);
    final int result;
    switch (specMode) {
        case MeasureSpec.AT_MOST:
            if (specSize < size) {
              	// 子View的测量总高度超过了LinearLayout可用空间大小
                result = specSize | MEASURED_STATE_TOO_SMALL;
            } else {
                result = size;
            }
            break;
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        case MeasureSpec.UNSPECIFIED:
        default:
            result = size;
    }
    return result | (childMeasuredState & MEASURED_STATE_MASK);
}
```

可以看出，当LinearLayout的测量模式为**UNSPECIFIED**时，LinearLayout的测量高度为子View的总高度size，因此当LinearLayout子View的总高度小于LinearLayout指定的高度时，LinearLayout的高度不会生效。

看到这里我们清楚了ScrollView子View无法占满全屏的原因，也见到了**UNSPECIFIED**的应用场景，其实不止ScrollView，**UNSPECIFIED**模式在其他的一些可滚动的ViewGroup中也有应用，比如RecyclerView。和**WRAP_CONTENT**相比，**UNSPECIFIED**模式不会限制View的大小，正是如此，**UNSPECIFIED**模式非常适合应用到可滚动的ViewGroup中，此时ViewGroup不必关心子View的大小是否超出了自身范围，即时超出了也可以通过滚动来查看。

我们在自定义View时该如何处理**UNSPECIFIED**的情况呢，这里引用一下[每日一问 详细的描述下自定义 View 测量时 MesureSpec.UNSPECIFIED](https://www.wanandroid.com/wenda/show/8613)中陈小缘大佬的回答，解释得很好。当遇到**UNSPECIFIED**时就比较自由了，既然尺寸由自己决定，那么可以写死为50，也可以固定为200，但还是建议结合实际需求来定义，比如ImageView，它的做法就是：有设置图片内容(drawable)的话，会直接使用这个drawable的尺寸，但不会超过指定的MaxWidth或MaxHeight， 没有内容的话就是0；而TextView处理**UNSPECIFIED**的方式，和**AT_MOST**是一样的。

### 1.4.layout

measure流程的作用是对View的大小进行测量，而layout的作用就是根据测量大小确定View的最终位置，简单地说就是把View放在哪。和measure流程类似，layout流程同样分为两种情况：单一View的layout和ViewGroup的layout，ViewGroup的layout流程要复杂一些，因为它不仅要进行自身的layout，还要对所有子View进行layout，我们先来看看单一View的layout流程。

#### 1.4.1.单一View的layout流程

View的layout流程从`layout()`方法开始，我们来看一下这个方法：

```java
public void layout(int l, int t, int r, int b) {
    // ...
    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;

    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
        // ...
    }
    // ...
}
```

`layout()`方法内部会根据`isLayoutModeOptical()`的返回值调用`setOpticalFrame()`方法或`setFrame()`方法，`isLayoutModeOptical()`方法会判断**LAYOUT_MODE_OPTICAL_BOUNDS**标志位，它表示一个布局模式，从名称上看应该是和布局边界有关，具体作用我也不是很了解，不过默认情况下都是没有设置该标志位的。这里可以暂且先不去管它的作用，我们会发现`setOpticalFrame()`方法内部最终还是会调用`setFrame()`方法，因此直接来看`setFrame()`方法就可以了。

```java
private boolean setOpticalFrame(int left, int top, int right, int bottom) {
    Insets parentInsets = mParent instanceof View ?
            ((View) mParent).getOpticalInsets() : Insets.NONE;
    Insets childInsets = getOpticalInsets();
    return setFrame(
            left   + parentInsets.left - childInsets.left,
            top    + parentInsets.top  - childInsets.top,
            right  + parentInsets.left + childInsets.right,
            bottom + parentInsets.top  + childInsets.bottom);
}

protected boolean setFrame(int left, int top, int right, int bottom) {
    boolean changed = false
    // ...
    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
        changed = true;
        // ...
        mLeft = left;
        mTop = top;
        mRight = right;
        mBottom = bottom;
        // ...
    }
    return changed;
}
```

`setFrame()`方法首先会判断根据`mLeft != left || mRight != right || mTop != top || mBottom != bottom`，即View的位置是否发生了改变，如果发生了改变，则返回值为true，反之返回值为false。如果View的位置发生了改变，会重新为View的四个顶点位置赋值，对应四个成员变量mLeft、mTop、mRight和mBottom，关于这四个值我们通过一个示意图就可以很清楚了，图片摘自[GcsSloop大佬的博客](https://www.gcssloop.com/#blog)。

![](http://gcsblog.oss-cn-shanghai.aliyuncs.com/blog/2019-04-29-071021.jpg?gcssloop)

首次layout 前这四个变量都没有赋过值，因此这里`setFrame()`方法会返回true，我们回到`layout()`方法，changed的值就为true，接下来会执行`onLayout()`方法，我们接着来看`onLayout()`方法。

```java
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
}
```

View中的`onLayout()`是一个空方法，没有声明任何逻辑，这是因为`layout()`方法已经确定了View的四个顶点的位置，而`onLayout()`方法是用于ViewGroup确定子View的位置，我们会再来分析。

单一View的layout流程就分析完了，是不是很简单，用一张流程图总结一下：

![](https://github.com/StephenZKCurry/Android-Study-Notes/blob/master/images/View%E7%9A%84layout%E6%B5%81%E7%A8%8B.jpg?raw=true)

#### 1.4.2.ViewGroup的layout流程

ViewGroup的layout流程同样从`layout()`方法开始，我们来看一下：

```java
@Override
public final void layout(int l, int t, int r, int b) {
  	// ...
  	super.layout(l, t, r, b);
  	// ...
}
```

ViewGroup的`layout()`方法使用final声明，因此子类无法重写该方法。`layout()`方法中调用了父类即View的`layout()`方法，确定了ViewGroup自身的四个顶点位置，并调用`onLayout()`方法，我们来看一下ViewGroup的`onLayout()`方法：

```java
@Override
protected abstract void onLayout(boolean changed, int l, int t, int r, int b);
```

可以发现ViewGroup的`onLayout()`方法是一个抽象方法，因此当我们自定义ViewGroup时需要重写该方法。ViewGroup没有实现`onLayout()`方法的原因同样是因为不同的ViewGroup具有不同的布局方式，无法得出一个统一实现。在自定义ViewGroup中的`onLayout()`方法中我们需要遍历所有的子View，根据需要的布局方式调用子View的`layout()`方法确定子View的位置。

这样说可能不是很清楚，接下来我们同样以LinearLayout为例，看一下它的layout流程，加深我们对ViewGroup的 layout流程的理解。由于ViewGroup的`layout()`方法无法被子类重写，因此我们直接来看LinearLayout的`onLayout()`方法：

```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    if (mOrientation == VERTICAL) {
        layoutVertical(l, t, r, b);
    } else {
        layoutHorizontal(l, t, r, b);
    }
}
```

和LinearLayout的`onMeasure()`方法类似，`onlayout()`方法中同样会根据LinearLayout的布局方向执行相应的布局方法，我们以竖直方向布局为例，分析一下`layoutVertical()`方法，水平方向同理。

```java
void layoutVertical(int left, int top, int right, int bottom) {
    // ...
  	// childTop和childLeft记录子View的左上位置
    int childTop;
  	int childLeft;
    // ...
    final int count = getVirtualChildCount();
    // ...
    for (int i = 0; i < count; i++) {
        final View child = getVirtualChildAt(i);
        // ...
        final int childWidth = child.getMeasuredWidth();
        final int childHeight = child.getMeasuredHeight();

        final LinearLayout.LayoutParams lp =
                (LinearLayout.LayoutParams) child.getLayoutParams();
        // ...

        childTop += lp.topMargin;
        setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                childWidth, childHeight);
      	// 每完成一个子View的layout，childTop就会增加
        childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

        i += getChildrenSkipCount(child, i);
    }
}
```

`layoutVertical()`方法内部遍历了LinearLayout的所有子View，每次遍历都调用`setChildFrame()`方法，我们来看一下这个方法：

```java
private void setChildFrame(View child, int left, int top, int width, int height) {
    child.layout(left, top, left + width, top + height);
}
```

可以看到`setChildFrame()`方法其实就是调用了子View的`layout()`方法，完成子View的布局。`setChildFrame()`方法调用完成后，会增加childTop的值，它对应子View的mTop，继续下一个子View的layout，还是比较好理解的，竖直方向的LinearLayout的子View是一个接一个往下放置的。

总结一下ViewGroup的layout流程，首先会调用`layout()`方法确定自身的位置，之后调用`onLayout()`方法，遍历所有的子View，根据ViewGroup的布局特性依次确定出每个子View的位置。流程图如下所示：

![](https://github.com/StephenZKCurry/Android-Study-Notes/blob/master/images/ViewGroup%E7%9A%84layout%E6%B5%81%E7%A8%8B.jpg?raw=true)

ViewGroup的layout流程和measure流程还是很相似的，不过在顺序上有一些区别，measure是先遍历子View对子View进行测量，最后根据子View的测量结果对ViewGroup自身进行测量；而layout是先确定ViewGroup的位置，再遍历子View确定子View的位置。

最后我们来梳理一下整个页面的layout过程，前面也提到过，页面的layout流程从ViewRootImpl的`performLayout()`方法开始。

**ViewRootImpl的performLayout方法**

```java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
                           int desiredWindowHeight) {
    // ...
    final View host = mView;
    // ...
    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
    // ...
}
```

方法内部首先将mView赋值给host，这里的mView我此前也提到过，就是在`handleResumeActivity()`方法中调用`wm.addView(decor,l)`时传过来的DecorView，因此后面调用host的`layout()`方法实际上就是调用DecorView的`layout()`方法，从这里就开始最顶层VIew的layout，而我们知道DecorView继承自FrameLayout，因此这里就是执行ViewGroup的`layout()`方法，之后的步骤我们就清楚了，首先确定出DecorView的位置，然后调用`onlayout()`方法遍历DecorView的子View，依次调用子View的`layout()`方法来确定子View的位置，如果子View是一个ViewGroup，还需要接着遍历子View的所有子View进行layout。

#### 1.4.3.getMeasureWidth/getMeasureHeight和getWidth/getHeight的区别

我在View的measure流程中提到过measure完成后可以通过`getMeasuredWidth()`和`getMeasuredHeight()`方法获取View的测量宽高，但是这个测量宽高并不等于View的最终实际宽高，现在就来解释一下这个问题。

我们知道View的宽高可以通过`getWidth()`和`getHeight()`方法来获得，首先来看一下这几个方法的定义，这里就只对比`getMeasureWidth()`和`getWidth()`方法，`getMeasuredHeight()`方法和`getHeight()`方法的区别同理。

```java
public final int getMeasuredWidth() {
    return mMeasuredWidth & MEASURED_SIZE_MASK;
}

public final int getMeasuredHeight() {
    return mMeasuredHeight & MEASURED_SIZE_MASK;
}

public final int getWidth() {
    return mRight - mLeft;
}

public final int getHeight() {
    return mBottom - mTop;
}
```

`getMeasuredWidth()`获取到是mMeasuredWidth的低30位，而mMeasuredWidth是在`onMeasure()`方法中通过`setMeasuredDimension()`赋值的。`getWidth()`获取到的是`mRight - mLeft`，这两个都是在`layout()`方法中通过`setFrame()`赋值的。这样看上去两者获取到的值好像没有什么联系，我们可以再回头看一下`layout()`方法传入的left和right参数的值，就以刚介绍过的DecorView为例吧，left参数传入了0，right参数传入了`host.getMeasuredWidth()`，因此最后计算出的`mRight - mLeft`就是`getMeasureWidth()`方法的返回值。其实不止DecorView，所有的View**在默认情况下`getWidth()`的值和`getMeasureWidth()`的值都是一样**的，需要注意这里强调的是**默认情况下**，那么什么情况下这两个方法的返回值不一样呢？

我们可以重写View的`layout()`方法，就像下面这样：

```java
@Override
public void layout(int l, int t, int r, int b) {
    super.layout(l, t, r + 100, b + 100);
}
```

这样就会导致`getWidth()`/`getHeight()`获取到的值比`getMeasureWidth()`/`getMeasureHeight()`获取到的值大100px，虽然一般情况下不会这样做，只是为了让我们更加清楚`getWidth()`/`getHeight()`和`getMeasureWidth()`/`getMeasureHeight()`的区别。

### 1.5.draw

通过前面的measure和layout两个流程，已经确定出了View的大小和位置，接下里就要把View显示出来了，draw的作用是就将View绘制到屏幕上。相比于前两个流程，View的绘制流程是最简单的，因为源码的逻辑很少，基本上都要靠我们自己去定义如何绘制。同样地，我们分两种情况进行分析，包括单一View的绘制和ViewGroup的绘制。

#### 1.5.1.单一View的draw流程

View的绘制流程从`draw()`方法开始，我们来看一下这个方法：

```java
public void draw(Canvas canvas) {
    // ...
    if (!dirtyOpaque) {
        // 第一步、绘制背景
        drawBackground(canvas);
    }
    // ...
    if (!verticalEdges && !horizontalEdges) {
        // 第二步、绘制自身内容
        if (!dirtyOpaque) onDraw(canvas);
        // 第三步、绘制子View
        dispatchDraw(canvas);
        // ...
        // 第四布、绘制装饰，包括滚动条和前景
        onDrawForeground(canvas);
        // ...
        return;
    }
    // ...
}
```

这里精简了一下源码，可以直观地看出View的`draw()`方法分为四个步骤（源码中提到了6个步骤，另外两个可以跳过的，这里就不列入了）：

* 调用`drawBackground()`方法绘制背景
* 调用`onDraw()`方法绘制自身内容
* 调用`dispatchDraw()`方法绘制子View
* 调用`onDrawForeground()`方法绘制装饰，包括滚动条和前景

下面我们就来分别看一下这四个方法。

**View的drawBackground方法**

```java
private void drawBackground(Canvas canvas) {
    // 获取背景Drawable
    final Drawable background = mBackground;
    if (background == null) {
        return;
    }
    // 根据layout流程确定出的四个顶点位置设置背景的边界
    setBackgroundBounds();

    // ...

    // 获取水平和竖直方向上的滑动距离
    final int scrollX = mScrollX;
    final int scrollY = mScrollY;
    // 绘制背景
    if ((scrollX | scrollY) == 0) {
        background.draw(canvas);
    } else {
        canvas.translate(scrollX, scrollY);
        background.draw(canvas);
        canvas.translate(-scrollX, -scrollY);
    }
}
```

`drawBackground()`方法首先会获取背景Drawable，如果没有设置背景则直接返回；如果设置了背景就调用Drawable的`draw()`方法完成背景的绘制，代码的逻辑还是比较简单的，我就不详细说了。

**View的onDraw方法**

```java
protected void onDraw(Canvas canvas) {
}
```

`onDraw()`方法可以说是我们在自定义View中最熟悉的，View的`onDraw()`是一个空方法，需要子类自己决定如何进行绘制。

**View的dispatchDraw方法**

```java
protected void dispatchDraw(Canvas canvas) {

}
```

View的`dispatchDraw()`方法同样是一个空方法，它的作用是对子View进行绘制，因此单一View自然无需实现该方法，我们稍后会看一下ViewGroup中是如何实现该方法的。

**View的onDrawForeground方法**

```java
public void onDrawForeground(Canvas canvas) {
    // 绘制滚动条
  	onDrawScrollIndicators(canvas);
    onDrawScrollBars(canvas);

  	// 绘制前景
    final Drawable foreground = mForegroundInfo != null ? mForegroundInfo.mDrawable : null;
    if (foreground != null) {
        if (mForegroundInfo.mBoundsChanged) {
            mForegroundInfo.mBoundsChanged = false;
            final Rect selfBounds = mForegroundInfo.mSelfBounds;
            final Rect overlayBounds = mForegroundInfo.mOverlayBounds;

            if (mForegroundInfo.mInsidePadding) {
                selfBounds.set(0, 0, getWidth(), getHeight());
            } else {
                selfBounds.set(getPaddingLeft(), getPaddingTop(),
                        getWidth() - getPaddingRight(), getHeight() - getPaddingBottom());
            }

            final int ld = getLayoutDirection();
            Gravity.apply(mForegroundInfo.mGravity, foreground.getIntrinsicWidth(),
                    foreground.getIntrinsicHeight(), selfBounds, overlayBounds, ld);
            foreground.setBounds(overlayBounds);
        }

        foreground.draw(canvas);
    }
}
```

`onDrawForeground()`方法用于绘制View的一些装饰，包括滚动条和前景，我们一般很少接触到该方法，就不具体分析了。

用一张流程图总结一下单一View的draw流程：

![](https://github.com/StephenZKCurry/Android-Study-Notes/blob/master/images/View%E7%9A%84draw%E6%B5%81%E7%A8%8B.jpg?raw=true)

虽然View的绘制流程可以分为以上四步，但是我们在自定义View中只需要重写`onDraw()`方法，按需要进行绘制就可以了。

#### 1.5.2.ViewGroup的draw流程

ViewGroup的绘制同样从`draw()`方法开始，也可分为和View相同的四个步骤，这里要重点分析一下第三步调用的`dispatchDraw()`方法，ViewGroup重写了该方法。其他三个步骤和View是一样的，这里就不再分析了。

```java
@Override
protected void dispatchDraw(Canvas canvas) {
    // ...
    final int childrenCount = mChildrenCount;
    final View[] children = mChildren;
    // ...
    // 遍历子View
    for (int i = 0; i < childrenCount; i++) {
        // ...
        final View transientChild = mTransientViews.get(transientIndex);
        if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                transientChild.getAnimation() != null) {
            // 绘制子View
            more |= drawChild(canvas, transientChild, drawingTime);
        }
        transientIndex++;
        if (transientIndex >= transientCount) {
            transientIndex = -1;
        }
        // ...
    }
    // ...
}

protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
    return child.draw(canvas, this, drawingTime);
}
```

`dispatchDraw()`方法内部主要做的就是遍历所有的子View，依次调用`drawChild`方法，`drawChild`方法内部又会调用子View的`draw()`方法，注意，这里调用的`draw()`方法并不是此前分析过的那个，它有三个参数。

```java
boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
    // ...
    if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
    	mPrivateFlags &= ~PFLAG_DIRTY_MASK;
    	dispatchDraw(canvas);
	} else {
    	draw(canvas);
	}
    // ...
    return more;
}
```

这里省略了大量代码，可以看出该方法内部会根据条件执行一个参数的`draw()`方法（执行的条件我后面会分析），后面的流程就和单一View的绘制流程相同了。

总结一下ViewGroup的draw流程，整体步骤和单一View的draw流程是一样的，不同的是ViewGroup重写了`dispatchDraw()`方法，在内部遍历子View并完成子View的绘制。

![](https://github.com/StephenZKCurry/Android-Study-Notes/blob/master/images/ViewGroup%E7%9A%84draw%E6%B5%81%E7%A8%8B.jpg?raw=true)

最后还是来梳理一下整个页面的draw流程，从ViewRootImpl的`performDraw()`方法开始：

```java
private void performDraw() {
    // ...
    boolean canUseAsync = draw(fullRedrawNeeded);
    // ...
}

private boolean draw(boolean fullRedrawNeeded) {
    // ...
    if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
        // ...
      	// 开启了硬件加速
        mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this, callback);
    } else {
        // ...
      	// 关闭了硬件加速
        if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                scalingRequired, dirty, surfaceInsets)) {
            return false;
        }
    }
    // ...
    return useAsyncReport;
}
```

`performDraw()`方法内部又会调用`draw()`方法，在`draw()`方法中会根据是否开启了硬件加速执行相应的逻辑，硬件加速就是通过引入GPU来提高绘制和界面刷新的效率，不过也有可能导致自定义View出现问题，在API 13（Android 4.0）及以上版本中，硬件加速是默认开启的，我们可以手动关闭硬件加速。关于硬件加速我也了解得不多，这里就不多提了，感兴趣的话可以查阅一下相关资料。下面我们就分别看一下开启和关闭硬件加速的情况下都是如何完成页面绘制的吧。

* **关闭硬件加速**

关闭硬件加速的情况下会执行`drawSoftware()`方法，我们来看一下这个方法：

```java
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
                             boolean scalingRequired, Rect dirty, Rect surfaceInsets) {
    // ...
    mView.draw(canvas);
    // ...
    return true;
}
```

方法内部会调用mView的`draw()`方法，这里的mView是DecorView，前面已经分析过了，因此现在进入了DecorView的绘制流程，接下来就和ViewGroup的绘制流程一样了，即遍历DecorView的所有子View，完成子View的绘制，如果子View是一个ViewGroup则重复该过程，直到所有的子View都绘制完成。

* **开启硬件加速**

开启硬件加速的情况下会执行`mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this, callback)`，mThreadedRenderer的类型为**ThreadedRenderer**，我们来看一下ThreadedRenderer的`draw()`方法：

```java
void draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks,
          FrameDrawingCallback frameDrawingCallback) {
    // ...
    updateRootDisplayList(view, callbacks);
    // ...
}

private void updateRootDisplayList(View view, DrawCallbacks callbacks) {
    // ...
    updateViewTreeDisplayList(view);
    // ...
}

private void updateViewTreeDisplayList(View view) {
    view.mPrivateFlags |= View.PFLAG_DRAWN;
    view.mRecreateDisplayList = (view.mPrivateFlags & View.PFLAG_INVALIDATED)
            == View.PFLAG_INVALIDATED;
    view.mPrivateFlags &= ~View.PFLAG_INVALIDATED;
    view.updateDisplayListIfDirty();
    view.mRecreateDisplayList = false;
}
```

经过一系列调用最终会调用`updateViewTreeDisplayList()`方法，这里传入的view为DecorView，方法内部会根据view是否设置了**PFLAG_INVALIDATED**标志位来给成员变量mRecreateDisplayList赋值，由于DecorView没有设置该标志位，因此mRecreateDisplayList的值为false。接下来会调用`updateDisplayListIfDirty()`方法，它定义在View中，我们来看一下：

```java
public RenderNode updateDisplayListIfDirty() {
    final RenderNode renderNode = mRenderNode;
    // ...
    if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0
            || !renderNode.isValid()
            || (mRecreateDisplayList)) {
        if (renderNode.isValid()
                && !mRecreateDisplayList) {
            // 不需要重新进行绘制
            mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
            dispatchGetDisplayList();

            return renderNode;
        }

        // 需要重新进行绘制
        mRecreateDisplayList = true;

        try {
            // ...
            if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
              	// 如果设置了PFLAG_SKIP_DRAW标志位，执行dispatchDraw()方法
                dispatchDraw(canvas);
                drawAutofilledHighlight(canvas);
                if (mOverlay != null && !mOverlay.isEmpty()) {
                    mOverlay.getOverlayView().draw(canvas);
                }
                if (debugDraw()) {
                    debugDrawFocus(canvas);
                }
            } else {
             	// 没有设置PFLAG_SKIP_DRAW标志位，执行draw()方法
                draw(canvas);
            }
            // ...
        } finally {
            renderNode.end(canvas);
            setDisplayListProperties(renderNode);
        }
    } else {
        mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
        mPrivateFlags &= ~PFLAG_DIRTY_MASK;
    }
    return renderNode;
}
```

我们先来看一下第一个if判断的几个条件：

* **mPrivateFlags & PFLAG_DRAWING_CACHE_VALID == 0**

由于DecorView没有设置**PFLAG_DRAWING_CACHE_VALID**标志位，因此该条件满足。

* **!renderNode.isValid()**

`isValid()`方法字面意思上是判断renderNode是否有效，那么什么时候是有效的呢？我们会发现`updateDisplayListIfDirty()`方法最后调用了`renderNode.end(canvas)`，点进这个`end()`方法看一下：

```java
/**
 * Ends the recording for this display list. A display list cannot be
 * replayed if recording is not finished. Calling this method marks
 * the display list valid and {@link #isValid()} will return true.
 *
 * @see #start(int, int)
 * @see #isValid()
 */
public void end(DisplayListCanvas canvas) {
    long displayList = canvas.finishRecording();
    nSetDisplayList(mNativeRenderNode, displayList);
    canvas.recycle();
}
```

从注释**Calling this method marks the display list valid and isValid() will return true**中可以看出当调用了`end()`方法后，`isValid()`会返回true，由于此时是页面首次绘制，还没有调用过`end()`方法，因此`isValid()`返回false，`!renderNode.isValid()`为true，该条件满足。

* **mRecreateDisplayList**

上面也说过了，由于设置**PFLAG_INVALIDATED**标志位，此时DecorView的mRecreateDisplayList值为false，该条件不满足。

由于满足了两个条件，因此会进入到第一个if判断中，接下来又是一个if判断，判断条件是`renderNode.isValid() && !mRecreateDisplayList` ，根据上面的分析，该条件不满足，因此不会执行if中的逻辑。接下来会判断是否设置了**PFLAG_SKIP_DRAW**标志位，关于这个标志位的作用我后面会分析，这里先记住ViewGroup默认情况下都会设置这个标志位，由于DecorView就是一个ViewGroup，会设置该标志位，因此会执行`dispatchDraw()`方法，遍历所有子View，完成对子View的绘制，如果子View是一个ViewGroup则接着遍历下面的子View，直到所有子View都完成绘制。

#### 1.5.3.ViewGroup的draw()方法调用问题

首先介绍几个Android中常见的位运算，有助于我们更好地理解源码：

> **a | b**：为a添加标志位b
>
> **(a & b) != 0**：判断a是否有标志位b
>
> **a & ~b**：为a清除标志位b
>
> **a^b**：取出a与b的不同部分

感叹一下，位运算在Android中还是很常见的，尤其是在View的源码中，熟悉上面这个几个位运算操作对我们阅读源码还是有很大帮助的。

下面进入正题，当我们的自定义View继承自ViewGroup时会遇到一个问题，默认情况下`draw()`方法和`onDraw()`方法都不会被调用，只会调用了`dispatchDraw()`方法，可以自己尝试一下，我这里就不展示了。我们下面就来分析一下原因，首先来看上面分析过的三个参数的`draw()`方法：

```java
boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
    // ...
    if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
    	mPrivateFlags &= ~PFLAG_DIRTY_MASK;
    	dispatchDraw(canvas);
	} else {
    	draw(canvas);
	}
    // ...
    return more;
}
```

这个方法是在父View遍历子VIew依次调用`drawChild()`方法后被调用的，可以很明显地看出当满足`(mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW`条件时，执行`dispatchDraw(canvas)`方法，不满足条件就执行一个参数的`draw()`方法，进而执行`onDraw()`方法。**mPrivateFlags**是View中定义的一个全局变量，用于存储各种标志位，上面的条件就是判断mPrivateFlags是否设置了**PFLAG_SKIP_DRAW**标志位。既然ViewGroup默认情况下不会执行`draw()`方法，那么肯定是设置了**PFLAG_SKIP_DRAW**标志位，是在什么时候设置的呢？我们发现在ViewGroup的构造方法中调用了`initViewGroup()`方法：

```java
public ViewGroup(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
    super(context, attrs, defStyleAttr, defStyleRes);

    initViewGroup();
    initFromAttributes(context, attrs, defStyleAttr, defStyleRes);
}
```

接着来看`initViewGroup()`方法：

```java
private void initViewGroup() {
    // ViewGroup doesn't draw by default
    if (!debugDraw()) {
        setFlags(WILL_NOT_DRAW, DRAW_MASK);
    }
    // ...
}
```

从注释**ViewGroup doesn't draw by default**中也能看出ViewGroup默认情况下的确不会调用`draw()`方法，在`initViewGroup()`方法内部执行了`setFlags(WILL_NOT_DRAW, DRAW_MASK)`，从方法名可以看出是设置了一个标志位，我们接下来看一下`setFlags()`方法：

```java
void setFlags(int flags, int mask) {
    // ...
    int old = mViewFlags;
    // 设置标志位
    mViewFlags = (mViewFlags & ~mask) | (flags & mask);
    // 判断标志位是否改变
    int changed = mViewFlags ^ old;
    if (changed == 0) {
        return;
    }
    int privateFlags = mPrivateFlags;
    // ...
    if ((changed & DRAW_MASK) != 0) {
        // 设置了DRAW_MASK模式的标志位
        if ((mViewFlags & WILL_NOT_DRAW) != 0) {
            // 设置了WILL_NOT_DRAW标志位
            if (mBackground != null
                    || mDefaultFocusHighlight != null
                    || (mForegroundInfo != null && mForegroundInfo.mDrawable != null)) {
                // 如果设置了背景、焦点高亮背景或者前景，就移除PFLAG_SKIP_DRAW标志位
                mPrivateFlags &= ~PFLAG_SKIP_DRAW;
            } else {
                // 如果没有设置背景、焦点高亮背景或者前景，就设置PFLAG_SKIP_DRAW标志位
                mPrivateFlags |= PFLAG_SKIP_DRAW;
            }
        } else {
            // 没有设置WILL_NOT_DRAW标志位，移除PFLAG_SKIP_DRAW标志位
            mPrivateFlags &= ~PFLAG_SKIP_DRAW;
        }
        requestLayout();
        invalidate(true);
    }
    // ...
}
```

`setFlags()`方法有两个参数：flags和mask，flags就是要设置的标志位，mask表示标志位的位置，mPrivateFlags和mask进行按位与运算可以得到该mask对应的标志位，举个例子，执行了`setFlags(WILL_NOT_DRAW, DRAW_MASK)`后，通过`mPrivateFlags & DRAW_MASK`就可以得到**WILL_NOT_DRAW**这个标志位。这里省略了大量代码，只保留了和**DRAW_MASK**相关的部分，其实View的可见状态VISIBLE、INVISIBLE和GONE也是通过标志位来实现的，感兴趣的话可以看一看。可以看出，当View设置了**WILL_NOT_DRAW**标志位，并且没有设置背景、焦点高亮背景或者前景（后面统称为背景）的情况下，会设置**PFLAG_SKIP_DRAW**标志位，由于ViewGroup默认情况下是没有设置背景的，因此会设置**PFLAG_SKIP_DRAW**标志位，不会执行`draw()`方法，当然也不会执行`onDraw()`方法。

如果想让ViewGroup的`draw()`方法被执行要怎么做呢？从上面的分析中也能看出，只要ViewGroup移除了**WILL_NOT_DRAW**标志位或者设置了背景，就会移除**PFLAG_SKIP_DRAW**标志位，使得`draw()`方法被调用，下面我们就看一下具体该怎么做。

* 移除**WILL_NOT_DRAW**标志位


View中有一个`setWillNotDraw()`方法，我们来看一下：

```java
public void setWillNotDraw(boolean willNotDraw) {
    setFlags(willNotDraw ? WILL_NOT_DRAW : 0, DRAW_MASK);
}
```

`setWillNotDraw()`方法内部会根据传入的willNotDraw参数调用`setFlags()`方法来设置或移除**WILL_NOT_DRAW**标志位，通过调用`setWillNotDraw(false)` 就可以移除**WILL_NOT_DRAW**标志位，使得ViewGroup的`draw()`方法得到调用。


* 为ViewGroup设置背景（包括背景、焦点高亮背景和前景）

这里就以设置背景的`setBackgroundDrawable()`方法为例分析，设置焦点高亮背景（对应`setDefaultFocusHighlight()`方法）和设置前景（对应`setForeground()`方法）类似。

```java
public void setBackgroundDrawable(Drawable background) {
    // ...
    if (background != null) {
        // ...
      	// 设置背景，移除PFLAG_SKIP_DRAW标志位
        if ((mPrivateFlags & PFLAG_SKIP_DRAW) != 0) {
            mPrivateFlags &= ~PFLAG_SKIP_DRAW;
            requestLayout = true;
        }
    } else {
        mBackground = null;
      	// 取消背景，设置PFLAG_SKIP_DRAW标志位
        if ((mViewFlags & WILL_NOT_DRAW) != 0
                && (mDefaultFocusHighlight == null)
                && (mForegroundInfo == null || mForegroundInfo.mDrawable == null)) {
            mPrivateFlags |= PFLAG_SKIP_DRAW;
        }
        // ...
    }
    // ...
}
```

当设置了背景后，mPrivateFlags会移除**PFLAG_SKIP_DRAW**标志位，因此可以通过设置背景的方式来使得ViewGroup的`draw()`方法得到执行。

通过以上两种方式就可以调用的ViewGroup的`draw()`方法了，从而使得`onDraw()`方法也会被调用。在开发中我们还是要考虑实际需求，因为ViewGroup本身只是一个容器，一般情况下是不需要绘制自身内容的，默认情况设置了**PFLAG_SKIP_DRAW**标志位也是出于系统优化的考虑，如果需要在`onDraw()`中绘制内容时再通过以上两种方式移除**PFLAG_SKIP_DRAW**标志位，或是直接在`dispatchDraw()`方法中进行绘制都可以。

## 2.自定义View须知

**本节内容参考自《Android开发艺术探索》**

自定义View的过程中有一些需要注意的地方，处理不好可能会导致设置的属性不生效等问题，影响View的显示和使用。

### 2.1.支持wrap_content

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

### 2.2.支持padding

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

### 2.3.避免使用Handler

View的内部本身提供了post系列的方法，完全可以替代Handler的作用，除非很明确地需要Handler来发送消息。

### 2.4. 防止内存泄漏

当View退出或不可见时需要及时停止线程和动画，否则可能会导致内存泄漏。可以在`onDetachedFromWindow()`方法中来处理，当包含此View的Activity退出或当前View被remove时会被调用。与此方法对应的是`onAttachedToWindow()` ，该方法会在包含此View的Activity启动时被调用。

### 2.5.处理好滑动冲突

当自定义View涉及到嵌套滑动时需要处理好滑动冲突。

## 3.开发中的常见问题

### 3.1.View获取宽高

根据此前的分析，View的三大流程都是在`onResume()`方法调用之后才开始的，因此在`onCreate()`、`onStart()`和`onResume()`方法中是无法通过`getWidth()`/`getHeight()`获取到View的宽高的。如果在开发中有这样的需求应该怎么办呢，当然还是有办法的，下面就以在`onCreate()`方法中获取View的宽高为例，介绍几种可行的方法。



### 3.2.invalidate和requestLayout的区别

我们在自定义View时可能需要更新View的显示，比如为View添加动画等等，有两个方法是我们经常会用到的`invalidate()`和`requestLayout()`，下面就来具体分析一下这两个方法的区别和使用场景。

#### 3.2.1.invalidate

**View的invalidate方法**

```java
public void invalidate() {
    invalidate(true);
}

public void invalidate(boolean invalidateCache) {
    invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
}
```

可以看出View的`invalidate()`方法最终会调用`invalidateInternal()`方法：

```java
void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
                        boolean fullInvalidate) {
	// ...
  	// 判断是否跳过重绘
    if (skipInvalidate()) {
        return;
    }
    
  	// 判断是否需要重绘
    if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
            || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
            || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
            || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
        if (fullInvalidate) {
            mLastIsOpaque = isOpaque();
            mPrivateFlags &= ~PFLAG_DRAWN;
        }

        mPrivateFlags |= PFLAG_DIRTY;

        if (invalidateCache) {
            // 设置PFLAG_INVALIDATED标志位
            mPrivateFlags |= PFLAG_INVALIDATED;
          	// 移除PFLAG_DRAWING_CACHE_VALID标志位
            mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
        }

        final AttachInfo ai = mAttachInfo;
        final ViewParent p = mParent;
        if (p != null && ai != null && l < r && t < b) {
            // damage表示要重绘的区域
            final Rect damage = ai.mTmpInvalRect;
            damage.set(l, t, r, b);
            // 将要重绘的区域传给父View
            p.invalidateChild(this, damage);
        }
        // ...
    }
}
```

`invalidateInternal()`方法中首先会根据`skipInvalidate()`方法判断是否跳过绘制，如果同时满足以下三个条件就直接return，跳过重绘。

* View是不可见的
* 当前没有设置动画
* 父View的类型不是ViewGroup或者父ViewGoup不处于过渡态

接下来根据一系列条件判断是否需要重绘，如果满足以下任意一条就进行重绘。

* View已经被绘制完成并且具有边界
* invalidateCache为true并且设置了**PFLAG_DRAWING_CACHE_VALID**标志位，即绘制缓存可用
* 没有设置**PFLAG_INVALIDATED**标志位，即没有被重绘过
* fullInvalidate为true并且透明度发生了变化

接下来判断如果invalidateCache为true，就设置**PFLAG_INVALIDATED**标志位，这一步很重要，后面还会提到，通过上面的调用也能看出这里的invalidateCache传入的值为true，因此会设置这个标志位。方法的最后会调用mParent即父View的`invalidateChild()`方法，将要重绘的区域damage传递给父View。接下里我们来看ViewGroup的`invalidateChild()`方法：

**ViewGroup的invalidateChild方法**

```java
@Override
public final void invalidateChild(View child, final Rect dirty) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null && attachInfo.mHardwareAccelerated) {
      	// 开启了硬件加速
        onDescendantInvalidated(child, child);
        return;
    }

    ViewParent parent = this;
    if (attachInfo != null) {
        // ...
        do {
            View view = null;
            if (parent instanceof View) {
                view = (View) parent;
            }
            // ...
            parent = parent.invalidateChildInParent(location, dirty);
            // ...
        } while (parent != null);
    }
}
```

方法内部首先会判断是否开启了硬件加速，接下来我们就分别看一下开启和关闭硬件加速的情况下重绘流程时怎样的。

* 关闭硬件加速

关闭硬件加速的情况下会循环调用`invalidateChildInParent()`方法，将返回值赋给parent，当parent为null时退出循环，我们来看ViewGroup的`invalidateChildInParent()`方法。

```java
@Override
public ViewParent invalidateChildInParent(final int[] location, final Rect dirty) {
    if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID)) != 0) {
        // ...
      	// 移除PFLAG_DRAWING_CACHE_VALID标志位
      	mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
      	// ...
        return mParent;
    }
    return null;
}
```

这里省略了大量代码，主要是对子View传递过来的重绘区域进行运算处理，方法最后会返回mParent。因此在`invalidateChild()`方法中会通过循环逐层调用父View的`invalidateChildInParent()`方法，那么当调用到最顶层ViewGroup——DecorView的`invalidateChild()`方法时，它的mParent是谁呢？我们可以回顾一下ViewRootImpl的`setView()`方法：

**ViewRootImpl的setView方法**

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;
            // ...
            requestLayout();
            // ...
            view.assignParent(this);
            // ...
        }
    }
}
```

可以发现方法执行了`view.assignParent(this)`，由于mView就是DecorView，因此这里的view也是DecorView，我们来看一下`assignParent()`方法，它定义在View中：

**View的assignParent方法**

```java
void assignParent(ViewParent parent) {
    if (mParent == null) {
        mParent = parent;
    } else if (parent == null) {
        mParent = null;
    } else {
        throw new RuntimeException("view " + this + " being added, but"
                + " it already has a parent");
    }
}
```

很明显`assignParent()`方法完成了View的成员变量mParent的赋值，因此DecorView的mParent就是上面传入的this，也就是ViewRootImpl。既然清楚了DecorView的mParent，接下来我们就来看一下ViewRootImpl的`invalidateChildInParent()`方法：

**ViewRootImpl的invalidateChildInParent方法**

```java
@Override
public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
    checkThread();
    // ...
    invalidateRectOnScreen(dirty);
    return null;
}
```

方法内部首先会调用`checkThread()`方法：

```java
void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
                "Only the original thread that created a view hierarchy can touch its views.");
    }
}
```

`checkThread()`方法会判断当前线程是否为主线程，如果不是主线程就直接抛出异常，因此我们需要特别注意，**`invalidate()`方法必须在主线程中调用**。回到ViewRootImpl的`invalidateChildInParent()`方法，最后调用了`invalidateRectOnScreen()`方法，同时由于返回值为null，因此执行完`invalidateChildInParent()`方法后parent被赋值为null，退出do-while循环。接下来我们就来看一下`invalidateRectOnScreen()`方法：

```java
private void invalidateRectOnScreen(Rect dirty) {
    // ...
    if (!mWillDrawSoon && (intersected || mIsAnimating)) {
        scheduleTraversals();
    }
}
```

`invalidateRectOnScreen()`方法内部会调用`scheduleTraversals()`方法，这个方法我们很熟悉了，接下来会调用`performTraversals()`方法，开始View的三大流程，这里再来回顾一下：

```java
private void performTraversals() {
    // ...
    boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
    if (layoutRequested) {
        // ...
        // 调用performMeasure()方法开始measure流程
        measureHierarchy(host, lp, res,
                desiredWindowWidth, desiredWindowHeight);
    }
    // ...
    final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
    if (didLayout) {
        // ...
        // 开始layout流程
        performLayout(lp, mWidth, mHeight);
    }
    // ...
    // 开始draw流程
    performDraw();
    // ...
}
```

由于此时没有给mLayoutRequested赋值，它的默认值为false，因此不会调用`measureHierarchy()`和`performLayout()`方法，只调用`performDraw()`方法，换句话说就是不会执行measure和layout流程，只执行draw流程，接下来就是调用DecorView的`draw()`方法，遍历DecorView的子View，逐层完成子View的绘制。

* 开启硬件加速


开启硬件加速时在ViewGroup的`invalidateChild()`方法中会调用`onDescendantInvalidated()`方法并直接返回，不会执行后面的`invalidateChildInParent()`方法，我们来看一下`onDescendantInvalidated()`方法：

```java
@Override
public void onDescendantInvalidated(@NonNull View child, @NonNull View target) {
    // ...
    if (mParent != null) {
        mParent.onDescendantInvalidated(this, target);
    }
}
```

方法内部会调用mParent的`onDescendantInvalidated()`方法，和`invalidateChildInParent()`类似，接下来会逐级调用父View的`onDescendantInvalidated()`方法，最后来到ViewRootImpl的`onDescendantInvalidated()`方法。

**ViewRootImpl的onDescendantInvalidated方法**

```java
@Override
public void onDescendantInvalidated(@NonNull View child, @NonNull View descendant) {
    if ((descendant.mPrivateFlags & PFLAG_DRAW_ANIMATION) != 0) {
        mIsAnimating = true;
    }
    invalidate();
}
```

接下来会调用ViewRootImpl的`invalidate`方法：

```java
void invalidate() {
    mDirty.set(0, 0, mWidth, mHeight);
    if (!mWillDrawSoon) {
        scheduleTraversals();
    }
}
```

可以看到这里同样调用了`scheduleTraversals()`方法，之后的流程和关闭硬件加速的情况类似，同样是调用`performDraw()`方法，不同的是开启硬件加速的情况下会执行`mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this, callback)`，前面也分析过了，之后会依次调用ThreadedRenderer的`updateRootDisplayList()`、`updateViewTreeDisplayList()`方法。

```java
private void updateViewTreeDisplayList(View view) {
    view.mPrivateFlags |= View.PFLAG_DRAWN;
    view.mRecreateDisplayList = (view.mPrivateFlags & View.PFLAG_INVALIDATED)
            == View.PFLAG_INVALIDATED;
    view.mPrivateFlags &= ~View.PFLAG_INVALIDATED;
    view.updateDisplayListIfDirty();
    view.mRecreateDisplayList = false;
}
```

这里需要注意，根据此前的分析，在调用View的`invalidate()`方法后，会给当前View设置**PFLAG_INVALIDATED**标志位，因此它的mRecreateDisplayList变量值为true，而其他的父级View由于没有设置**PFLAG_INVALIDATED**标志位，mRecreateDisplayList值为false。接下来会调用view的`updateDisplayListIfDirty()`方法，这里的view是DecorView。

```java
public RenderNode updateDisplayListIfDirty() {
    final RenderNode renderNode = mRenderNode;
    // ...
    if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0
            || !renderNode.isValid()
            || (mRecreateDisplayList)) {
        if (renderNode.isValid()
                && !mRecreateDisplayList) {
            // 不需要重新进行绘制
            mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
            dispatchGetDisplayList();

            return renderNode;
        }

        // 需要重新进行绘制
        mRecreateDisplayList = true;

        try {
            // ...
            if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
              	// 如果设置了PFLAG_SKIP_DRAW标志位，执行dispatchDraw()方法
                dispatchDraw(canvas);
                drawAutofilledHighlight(canvas);
                if (mOverlay != null && !mOverlay.isEmpty()) {
                    mOverlay.getOverlayView().draw(canvas);
                }
                if (debugDraw()) {
                    debugDrawFocus(canvas);
                }
            } else {
             	// 没有设置PFLAG_SKIP_DRAW标志位，执行draw()方法
                draw(canvas);
            }
            // ...
        } finally {
            renderNode.end(canvas);
            setDisplayListProperties(renderNode);
        }
    } else {
        mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
        mPrivateFlags &= ~PFLAG_DIRTY_MASK;
    }
    return renderNode;
}
```

这里就和此前的分析不同了，由于页面首次绘制完成后执行了`renderNode.end(canvas)`，因此这里`renderNode.isValid()`返回值为true，而DecorView的mRecreateDisplayList值为false，因不会执行后面的重新绘制逻辑，取而代之的是调用`dispatchGetDisplayList()`方法，我们来看一下这个方法：

```java
@Override
protected void dispatchGetDisplayList() {
    final int count = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < count; i++) {
        final View child = children[i];
        if (((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null)) {
            recreateChildDisplayList(child);
        }
    }
    // ...
}

private void recreateChildDisplayList(View child) {
    child.mRecreateDisplayList = (child.mPrivateFlags & PFLAG_INVALIDATED) != 0;
    child.mPrivateFlags &= ~PFLAG_INVALIDATED;
    child.updateDisplayListIfDirty();
    child.mRecreateDisplayList = false;
}
```

`dispatchGetDisplayList()`方法内部会遍历子View，依次调用`recreateChildDisplayList()`方法，不难看出`recreateChildDisplayList()`方法和`updateViewTreeDisplayList()`方法很像，接下来同样会调用`updateDisplayListIfDirty()`方法，对于没有设置**PFLAG_INVALIDATED**标志位的View，它的mRecreateDisplayList值为false，会重复上面的过程，即调用`dispatchGetDisplayList()`方法；而对于调用了`invalidate()`方法的View，由于设置了**PFLAG_INVALIDATED**标志位，它的mRecreateDisplayList值为true，会执行`updateDisplayListIfDirty()`方法最后的重绘逻辑，即调用`执行dispatchDraw()`方法或者`draw()`方法完成自身及子View的绘制。

最后总结一下`invalidate()`方法，调用View的`invalidate()`方法后会逐级调用父View的方法，最终导致ViewRootImpl的`scheduleTraversals()`方法被调用，进而调用`performTraversals()`方法。由于mLayoutRequested的值为false，因此不会执行measure和layout流程，只执行draw流程。draw流程的执行过程和是否开启硬件加速有关：

* 如果关闭了硬件加速，从DecorView开始的所有View都会重新完成绘制
* 如果开启了硬件加速，只有调用`invalidate()`方法的View（包括它的子View）会完成重新绘制

由此也可以看出，开启硬件加速确实可以提高重绘的效率。

此外，由于`invalidate()`方法必须在主线程中调用，那么如果我们想要在子线程中刷新视图要怎么做呢？不用担心，官方还为我们提供了一个`postInvalidate()`方法，其实从名称上我们也能猜到它的作用了，就是用于在子线程中刷新视图，简单看一下它的定义：

```java
public void postInvalidate() {
    postInvalidateDelayed(0);
}

public void postInvalidateDelayed(long delayMilliseconds) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        attachInfo.mViewRootImpl.dispatchInvalidateDelayed(this, delayMilliseconds);
    }
}

public void dispatchInvalidateDelayed(View view, long delayMilliseconds) {
    Message msg = mHandler.obtainMessage(MSG_INVALIDATE, view);
    mHandler.sendMessageDelayed(msg, delayMilliseconds);
}
```

哈哈，果然还是用到了Handler，mHandler是ViewRootImpl中的一个成员变量，类型为**ViewRootHandler**，我们来看一下ViewRootHandler对**MSG_INVALIDATE**消息的处理：

```java
final class ViewRootHandler extends Handler {
    // ...
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MSG_INVALIDATE:
                ((View) msg.obj).invalidate();
                break;
            // ...
        }
    }
}
```

可以看出最后还是调用了`invalidate()`方法，因此`postInvalidate()`方法其实就是通过Handler完成了线程的切换，使得`invalidate()`方法在主线程中被调用。


#### 3.2.2.requestLayout

**View的requestLayout方法**

```java
public void requestLayout() {
    // ...
  	// 给View添加两个标志位
    mPrivateFlags |= PFLAG_FORCE_LAYOUT;
    mPrivateFlags |= PFLAG_INVALIDATED;

    if (mParent != null && !mParent.isLayoutRequested()) {
      	// 调用父类的requestLayout()方法
        mParent.requestLayout();
    }
    // ...
}
```

`requestLayout()`方法内部会调用mParent即父View的`requestLayout()`方法，最终会来到ViewRootImpl的`requestLayout()`方法：

**ViewRootImpl的requestLayout方法**

```java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

首先还是会进行线程的检查，因此`requestLayout()`方法同样只能在主线程中调用。接着会把mLayoutRequested赋值为true并调用`scheduleTraversals()`方法。后面的流程相信也不用我多说了，调用`performTraversals()`方法，由于将mLayoutRequested赋值为true，因此会依次执行`measureHierarchy()`、`performLayout()`和`performDraw()`方法，开始View的三大流程。

到这里还没完，我们需要探究一下View调用`requrestLayout()`是否会导致View树中的所有View都进行重新测量、布局和绘制。我们注意到调用`requestLayout()`方法后，会为当前View及所有父级View添加**PFLAG_FORCE_LAYOUT**和**PFLAG_INVALIDATED**标志位。首先来回顾一下View的`measure()`方法：

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    // ...
    final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;

    final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec
            || heightMeasureSpec != mOldHeightMeasureSpec;
    final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY
            && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY;
    final boolean matchesSpecSize = getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)
            && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec);
    final boolean needsLayout = specChanged
            && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);

    if (forceLayout || needsLayout) {
        // ...
        onMeasure(widthMeasureSpec, heightMeasureSpec);
        // ...
      	// 设置PFLAG_LAYOUT_REQUIRED标志位
        mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
    }

    mOldWidthMeasureSpec = widthMeasureSpec;
    mOldHeightMeasureSpec = heightMeasureSpec;
    // ...
}
```

可以看出，当View设置了**PFLAG_FORCE_LAYOUT**标志位后，forceLayout的值为true，因此会执行`onMeasure()` 方法，而对于没有设置**PFLAG_FORCE_LAYOUT**标志位的View，需要判断测量尺寸是否发生了改变，如果改变了才会调用`onMeasure()`方法。在调用`onMeasure()`方法后会给View设置**PFLAG_LAYOUT_REQUIRED**标志位，我们再来看View的`layout()`方法：

```java
public void layout(int l, int t, int r, int b) {
    // ...
    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;

    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
        // ...
    }
    // ...
}
```

对于设置了**PFLAG_LAYOUT_REQUIRED**标志位的View，`onLayout()`方法肯定会执行，另一种情况就是View的四个顶点坐标发生改变，也会执行`onLayout()`方法。

结合上面的分析可以得出结论，当View调用了`requestLayout()`方法后，自身及父级View的`onMeasure()`和`onLayout()`方法会被调用，对于它的子View，`onMeasure()`和`onLayout()`方法不一定被调用。

对于draw流程，前面分析过`perform()`方法会调用ViewRootImpl中的`draw()`方法：

**ViewRootImpl的draw方法**

```java
private boolean draw(boolean fullRedrawNeeded) {
    // ...
  	final Rect dirty = mDirty;
  	// ...
    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
        if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
            // ...
            // 开启了硬件加速
            mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this, callback);
        } else {
            // ...
            // 关闭了硬件加速
            if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                    scalingRequired, dirty, surfaceInsets)) {
                return false;
            }
        }
    }
    // ...
    return useAsyncReport;
}
```

dirty指向ViewRootImpl中的一个成员变量mDirty，类型为Rect，在ViewRootImpl的`invalidate()`方法中会调用`set()`方法为其设置四个边界值，由于此时没有调用`invalidate()`方法，因此`mDirty.isEmpty()`返回true，不会执行后面的绘制方法，因此整个View树不会进行重新绘制。不过也有这样一种情况，我们知道在执行VIew的layout流程时会调用`setFrame()`方法，在`setFrame()`方法中有这样的逻辑：

```java
int oldWidth = mRight - mLeft;
int oldHeight = mBottom - mTop;
int newWidth = right - left;
int newHeight = bottom - top;
boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

invalidate(sizeChanged);
```

可以看出当View的宽或高发生改变时会调用`invalidate()`方法，导致View的重新绘制。

最后总结一下`invalidate()`和`requestLayout()`的异同：

**相同点**

* `invalidate()`和`requestLayout()`方法最终都会调用ViewRootImpl的`performTraversals()`方法

**不同点**

* `invalidate()`方法不会执行`measureHierarchy()`和`performLayout()`方法，也就不会执行measure和layout流程，只执行draw流程，如果开启了硬件加速则只进行调用者View的重绘
* `requestLayout()`方法会依次`measureHierarchy()`、`performLayout()`和`performDraw()`方法，调用者View和它的父级View会重新进行measure、layout，一般情况下不会执行draw流程，子View不一定会重新measure和layout。

综上，当只需要进行重新绘制时就调用`invalidate()`，如果需要重新测量和布局就调用`requestLayout()`，但是`requestLayout()`不保证进行重新绘制，如果要进行重新绘制可以再手动调用`invalidate()`。

下面就以一个简单的例子验证一下上面的几个结论，我定义了两个ViewGroup和一个View，代码很简单，如下所示：

**MyViewGroup1.java**

```java
public class MyViewGroup1 extends ViewGroup {

    public MyViewGroup1(Context context) {
        this(context, null);
    }

    public MyViewGroup1(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public MyViewGroup1(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        setWillNotDraw(false);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        Log.e("TAG", "MyViewGroup1执行了onMeasure");
        measureChildren(widthMeasureSpec, heightMeasureSpec);
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        Log.e("TAG", "MyViewGroup1执行了onLayout");
        for (int i = 0; i < getChildCount(); i++) {
            View child = getChildAt(i);

            int width = child.getMeasuredWidth();
            int height = child.getMeasuredHeight();

            child.layout(l, t, l + width, t + height);
        }
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        Log.e("TAG", "MyViewGroup1执行了onDraw");
        canvas.drawColor(Color.YELLOW);
    }
}
```

**MyViewGroup2.java**

```java
public class MyViewGroup2 extends ViewGroup {

    public MyViewGroup2(Context context) {
        this(context, null);
    }

    public MyViewGroup2(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public MyViewGroup2(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        setWillNotDraw(false);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        Log.e("TAG", "MyViewGroup2执行了onMeasure");
        measureChildren(widthMeasureSpec, heightMeasureSpec);
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        Log.e("TAG", "MyViewGroup2执行了onLayout");
        for (int i = 0; i < getChildCount(); i++) {
            View child = getChildAt(i);

            int width = child.getMeasuredWidth();
            int height = child.getMeasuredHeight();

            child.layout(l, t, l + width, t + height);
        }
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        Log.e("TAG", "MyViewGroup2执行了onDraw");
        canvas.drawColor(Color.GREEN);
    }
}
```

**MyView.java**

```java
public class MyView extends View {

    public MyView(Context context) {
        super(context);
    }

    public MyView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
    }

    public MyView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        Log.e("TAG", "MyView执行了onMeasure");
    }

    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        super.onLayout(changed, left, top, right, bottom);
        Log.e("TAG", "MyView执行了onLayout");
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        Log.e("TAG", "MyView执行了onDraw");
        canvas.drawColor(Color.RED);
    }
}
```

为了保证ViewGroup的`onDraw()`方法执行，我在构造方法中调用了`setWillNotDraw(false)`。布局文件也很简单，一个三级的嵌套：

**activity_test.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<com.example.MyViewGroup1 xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/myViewGroup1"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <com.example.MyViewGroup2
        android:id="@+id/myViewGroup2"
        android:layout_width="300dp"
        android:layout_height="300dp">

        <com.example.MyView
            android:id="@+id/myView"
            android:layout_width="200dp"
            android:layout_height="200dp" />

    </com.example.MyViewGroup2>

</com.example.MyViewGroup1>
```

运行后的效果如下：

![](http://tva1.sinaimg.cn/large/007X8olVly1g6td2uu24ij305k0c1glk.jpg)

我给三个View添加了点击事件，点击后分别调用自身的`invalidate()`或`requestLayout()`方法，下面分情况看一下。

* 调用`invalidate()`方法

**开启硬件加速**

调用MyView的`invalidate()`方法：![](http://tva1.sinaimg.cn/large/007X8olVly1g6td3xg644j304s00p0sh.jpg)

调用MyViewGroup2的`invalidate()`方法：![](http://tva1.sinaimg.cn/large/007X8olVly1g6td4ml3v3j306d00r741.jpg)

调用MyViewGroup1的`invalidate()`方法：![](http://tva1.sinaimg.cn/large/007X8olVly1g6td54xcynj306g00v741.jpg)

**关闭硬件加速**

调用MyView的`invalidate()`方法：![](http://tva1.sinaimg.cn/large/007X8olVly1g6td5ichncj306n01rq2q.jpg)



调用MyViewGroup2的`invalidate()`方法：![](http://tva1.sinaimg.cn/large/007X8olVly1g6td5ichncj306n01rq2q.jpg)



调用MyViewGroup1的`invalidate()`方法：![](http://tva1.sinaimg.cn/large/007X8olVly1g6td5ichncj306n01rq2q.jpg)

可以看出，关闭硬件加速时，调用任何一个View的`invalidate()`方法都会导致整个View树的重新绘制；开启硬件加速时，调用哪一个View的`invalidate()`方法就会重绘哪一个View。`invalidate()`方法不会导致`onMeasure()`和`onLayout()`被调用。

* 调用`requestLayout()`方法

**开启硬件加速**

调用MyView的`requestLayout()`方法：![](http://tva1.sinaimg.cn/large/007X8olVly1g6td604gmgj307w03ojr9.jpg)

调用MyViewGroup2的`requestLayout()`方法：![](http://tva1.sinaimg.cn/large/007X8olVly1g6td6ffabcj307o02dweb.jpg)

调用MyViewGroup1的`requestLayout()`方法：![](http://tva1.sinaimg.cn/large/007X8olVly1g6td6ovqtuj307g0183ya.jpg)

**关闭硬件加速**

调用MyView的`requestLayout()`方法：![](http://tva1.sinaimg.cn/large/007X8olVly1g6td604gmgj307w03ojr9.jpg)

调用MyViewGroup2的`requestLayout()`方法：![](http://tva1.sinaimg.cn/large/007X8olVly1g6td6ffabcj307o02dweb.jpg)

调用MyViewGroup1的`requestLayout()`方法：![](http://tva1.sinaimg.cn/large/007X8olVly1g6td6ovqtuj307g0183ya.jpg)

可以看出此时是否开启硬件加速对于`requestLayout()`方法的调用流程没有影响，调用View的`requestLayout()`方法会导致自身及其父View的`onMeasure()`和`onLayout()`方法被调用，并不会调用`onDraw()`方法进行重绘，当然前面也分析过了，`onDraw()`方法不是一定不会被调用，当View重新绘制时硬件加速的作用就会有所体现了。

## 4.个人目前的一些疑问

### 4.1.setContentView()方法只能在onCreate()中调用吗

### 4.2.AppCompatActivity布局层级更深的原因

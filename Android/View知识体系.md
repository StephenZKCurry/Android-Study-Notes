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

![](https://github.com/StephenZKCurry/Android-Study-Notes/blob/master/images/Android%E8%A7%86%E5%9B%BE%E5%B1%82%E7%BA%A7.png?raw=true)

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

从上面几个流程的先后顺序我们就能清楚为什么在`onResume()`方法中或者`onResume()`方法之前获取不到View的宽高，就是因为此时View还未执行measure流程。

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
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

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



| 子View\父View  | EXACTLY | AT_MOST | UNSPECIFIED |
| ------------ | ------- | ------- | ----------- |
| 具体数值         | EXACTLY | EXACTLY | EXACTLY     |
| match_parent | EXACTLY | AT_MOST | UNSPECIFIED |
| wrap_content | AT_MOST | AT_MOST | UNSPECIFIED |

对于DecorView来说，它是没有父View的，那么它的MeasureSpec是如何得到的呢？我们在上一节分析到ViewRootImpl的`performTraversals()`方法时，介绍到方法内部调用了`measureHierarchy()`方法，进而调用`performMeasure()`方法开始View的measure流程，现在我们就来具体看一下`performMeasure()`方法。

**ViewRootImpl的performMeasure方法**

```java

```



#### 1.3.2.LayoutParams

##### 1.3.2.1.LayoutParams简介

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

##### 1.3.2.2.View的LayoutParams属性是何时设置的

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

##### 1.3.3.1.单一View的measure流程



##### 1.3.3.2.ViewGroup的measure流程



#### 1.3.4.补充介绍

##### 1.3.4.1.MeasureSpec.UNSPECIFIED的应用



### 1.4.layout



### 1.5.draw



## 2.自定义View须知

**本文参考自《Android开发艺术探索》**

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



### 3.2.invalidate和requestLayout的区别

**a|b: 添加标志位b;**

**(a&b)!=0: 判断是否有标志位b;**

**a&~b:清除标志位b;**

**a^b: 取出a与b的不同部分;**

#### 3.2.1.invalidate



#### 3.2.2.requestLayout





## 4.个人目前的一些疑问

### 4.1.setContentView()方法只能在onCreate()中调用吗



### 4.2.AppCompatActivity布局层级更深的原因






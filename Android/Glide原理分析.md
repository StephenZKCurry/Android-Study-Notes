# Glide源码分析

[TOC]

**Glide**是Android开发中最常用的图片加载框架之一，使用起来很简单，只需要以下三步即可将图片加载到ImageVIew上：

```java
Glide
        .with(this)
        .load(imageUrl)
        .into(iimageView);
```

虽然使用起来简单，但是Glide的实现原理可并不简单，接下来我们就从源码角度来具体分析Glide的原理。

本文是跟着郭霖大神的博客[Android图片加载框架最全解析（二），从源码的角度理解Glide的执行流程](https://blog.csdn.net/guolin_blog/article/details/53939176)一步一步分析的，对应的Glide版本为3.7.0，对于我来说光是对照着文章看源码有一种很强的催眠效果，因此需要亲自记录一下整个分析的过程，也学习一下郭神的源码分析思路。

##  1.with()方法

**Glide的with()方法**

```java
// 1.参数传入Context
public static RequestManager with(Context context) {
    RequestManagerRetriever retriever = RequestManagerRetriever.get();
    return retriever.get(context);
}

// 2.参数传入FragmentActivity
public static RequestManager with(FragmentActivity activity) {
    RequestManagerRetriever retriever = RequestManagerRetriever.get();
    return retriever.get(activity);
}

// 3.参数传入Activity
public static RequestManager with(Activity activity) {
    RequestManagerRetriever retriever = RequestManagerRetriever.get();
    return retriever.get(activity);
}

// 4.参数传入v4包中的Fragment
public static RequestManager with(Fragment fragment) {
    RequestManagerRetriever retriever = RequestManagerRetriever.get();
    return retriever.get(fragment);
}

// 5.参数传入android.app包中的Fragment
public static RequestManager with(android.app.Fragment fragment) {
    RequestManagerRetriever retriever = RequestManagerRetriever.get();
    return retriever.get(fragment);
}
```

`with()`方法有5个重载方法，可以看出第一步都是调用**RequestManagerRetriever**的`get()`方法，我们点进去看一下：

**RequestManagerRetriever的get()方法**

```java
private static final RequestManagerRetriever INSTANCE = new RequestManagerRetriever();

public static RequestManagerRetriever get() {
    return INSTANCE;
}
```

`get()`方法就是简单地得到一个**RequestManagerRetriever**对象，接下来我们回到Glide的`with()`方法，获取到RequestManagerRetriever对象后根据传入参数类型的不同，调用它的`get()`方法。

**RequestManagerRetriever的get()方法**

```java
// 1.参数传入Context
public RequestManager get(Context context) {
    if (context == null) {
        throw new IllegalArgumentException("You cannot start a load on a null Context");
    } else if (Util.isOnMainThread() && !(context instanceof Application)) {
        if (context instanceof FragmentActivity) {
            return get((FragmentActivity) context);
        } else if (context instanceof Activity) {
            return get((Activity) context);
        } else if (context instanceof ContextWrapper) {
            return get(((ContextWrapper) context).getBaseContext());
        }
    }

    return getApplicationManager(context);
}

// 2.参数传入FragmentActivity
public RequestManager get(FragmentActivity activity) {
    if (Util.isOnBackgroundThread()) {
        return get(activity.getApplicationContext());
    } else {
        assertNotDestroyed(activity);
        FragmentManager fm = activity.getSupportFragmentManager();
        return supportFragmentGet(activity, fm);
    }
}


// 3.参数传入Activity
public RequestManager get(Activity activity) {
    if (Util.isOnBackgroundThread() || Build.VERSION.SDK_INT < Build.VERSION_CODES.HONEYCOMB) {
        return get(activity.getApplicationContext());
    } else {
        assertNotDestroyed(activity);
        android.app.FragmentManager fm = activity.getFragmentManager();
        return fragmentGet(activity, fm);
    }
}

// 4.参数传入v4包中的Fragment
public RequestManager get(Fragment fragment) {
    if (fragment.getActivity() == null) {
        throw new IllegalArgumentException("You cannot start a load on a fragment before it is attached");
    }
    if (Util.isOnBackgroundThread()) {
        return get(fragment.getActivity().getApplicationContext());
    } else {
        FragmentManager fm = fragment.getChildFragmentManager();
        return supportFragmentGet(fragment.getActivity(), fm);
    }
}

// 5.参数传入android.app包中的Fragment
public RequestManager get(android.app.Fragment fragment) {
    if (fragment.getActivity() == null) {
        throw new IllegalArgumentException("You cannot start a load on a fragment before it is attached");
    }
    if (Util.isOnBackgroundThread() || Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN_MR1) {
        return get(fragment.getActivity().getApplicationContext());
    } else {
        android.app.FragmentManager fm = fragment.getChildFragmentManager();
        return fragmentGet(fragment.getActivity(), fm);
    }
}
```

同样是5个重载方法，最终都是返回一个**RequestManager**对象，具体情况可以分为一下几种：

* 参数传入FragmentActivity/Activity/Fragment(v4包)/Fragment(app包)
  * 当前线程为主线程，调用`supportFragmentGet()`/`fragmentGet()`方法返回RequestManager对象
  * 当前线程不为主线程，调用`getApplicationManager()`方法返回RequestManager对象
* 参数传入Context
  * 当前线程为主线程并且Context不是ApplicationContext，调用后面Context类型对应的重载方法
  * 当前线程不为主线程或者Context为ApplicationContext，调用`getApplicationManager()`方法返回RequestManager对象

其实再简单一点来看只有两种情况，当前线程不为主线程或传入了ApplicationContext，调用`getApplicationManager()`方法；当前线程为主线程，调用`supportFragmentGet()`/`fragmentGet()`方法。

```java
private RequestManager getApplicationManager(Context context) {
    if (applicationManager == null) {
        synchronized (this) {
            if (applicationManager == null) {
                applicationManager = new RequestManager(context.getApplicationContext(),
                        new ApplicationLifecycle(), new EmptyRequestManagerTreeNode());
            }
        }
    }

    return applicationManager;
}

RequestManager supportFragmentGet(Context context, FragmentManager fm) {
    SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm);
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
        requestManager = new RequestManager(context, current.getLifecycle(), current.getRequestManagerTreeNode());
        current.setRequestManager(requestManager);
    }
    return requestManager;
}

SupportRequestManagerFragment getSupportRequestManagerFragment(final FragmentManager fm) {
    SupportRequestManagerFragment current = (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
    if (current == null) {
        current = pendingSupportRequestManagerFragments.get(fm);
        if (current == null) {
            current = new SupportRequestManagerFragment();
            pendingSupportRequestManagerFragments.put(fm, current);
          	// 添加一个SupportRequestManagerFragment
            fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
            handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
        }
    }
    return current;
}
```

`getApplicationManager()`方法内部就是简单地返回requestManager，我们重点来看`supportFragmentGet()`方法，`fragmentGet()`方法类似，这里就不分析了。可以看出`supportFragmentGet()`方法调用了`getSupportRequestManagerFragment()`方法，该方法的作用就是添加一个透明Fragment，类型为**SupportRequestManagerFragment**，这种方法在Android开发中还是很常见的，作用主要就是处理生命周期和回调相关事件，对于Glide来说，保证了Activity关闭后图片加载也不再进行。

总结一下Glide的`with()`方法，返回一个RequestManager对象，具体地，如果参数传入了ApplicationContext或者当前线程不为主线程，Glide和整个应用的生命周期同步，应用关闭后Glide停止加载；如果参数传入了Activity或Fragment，Glide与当前所在Activity的生命周期同步，采用添加一个透明Fragment的方式保证了Activity退出后Glide停止加载图片。

## 2.load()方法

通过`with()`方法我们已经得到RequestManager对象，接下来我们就来看一下RequestManager的`load()`方法，由于`load()`方法有多个重载方法，可以加载不同类型的图片资源，这里就不一一分析了，以传入图片路径url为例，我们只看传入参数为String类型的情况。

**RequestManager的load()方法**

```java
public DrawableTypeRequest<String> load(String string) {
    return (DrawableTypeRequest<String>) fromString().load(string);
}
```

可以看到`load()`方法最终返回了一个**DrawableTypeRequest**类型对象，哈哈，其实`load()`方法分析到这就可以了，不过秉着刨根问底的原则我们再来看一下`fromString()`方法。

```java
public DrawableTypeRequest<String> fromString() {
    return loadGeneric(String.class);
}
```

`fromString()`方法又会调用`loadGeneric()` 方法，由于`load()`方法中传入了图片路径字符串，因此这里传入String.class，我们接着来看`loadGeneric()` 方法。

```java
private <T> DrawableTypeRequest<T> loadGeneric(Class<T> modelClass) {
  	// 创建出ModelLoader
    ModelLoader<T, InputStream> streamModelLoader = Glide.buildStreamModelLoader(modelClass, context);
    ModelLoader<T, ParcelFileDescriptor> fileDescriptorModelLoader =
            Glide.buildFileDescriptorModelLoader(modelClass, context);
    if (modelClass != null && streamModelLoader == null && fileDescriptorModelLoader == null) {
        throw new IllegalArgumentException("Unknown type " + modelClass + ". You must provide a Model of a type for"
                + " which there is a registered ModelLoader, if you are using a custom model, you must first call"
                + " Glide#register with a ModelLoaderFactory for your custom model class");
    }

  	// 创建DrawableTypeRequest对象
    return optionsApplier.apply(
            new DrawableTypeRequest<T>(modelClass, streamModelLoader, fileDescriptorModelLoader, context,
                    glide, requestTracker, lifecycle, optionsApplier));
}
```

其实根据方法名我们也能看出这是一个通用方法，瞄了一眼`load()`方法的其他几个重载方法，最终也会调用到`loadGeneric()` 方法，区别只是传入的Class类型不同。`loadGeneric()`方法内部首先会调用`buildStreamModelLoader()`方法创建出ModelLoader对象，最后将这个对象作为参数构造出DrawableTypeRequest对象。

我们点击DrawableTypeRequest方法，可以发现`asBitmap()`、`asGif()`都是在该类中定义的，而`load()`方法定义在它的父类**DrawableRequestBuilder**中，DrawableRequestBuilder类中定义了很多常见的方法，比如`placeholder()`、`error()`等等，我们先来看一下`load()`方法。

**DrawableRequestBuilder的load()方法**

```java
@Override
public DrawableRequestBuilder<ModelType> load(ModelType model) {
    super.load(model);
    return this;
}
```

可以看出DrawableRequestBuilder的`load()`方法又调用了它的父类**GenericRequestBuilder**的`load()`方法。

`load()`方法分析到这里就差不多了，好吧，其实也没多分析什么o(╥﹏╥)o，我们，目前只需要知道`load()`方法返回了DrawableTypeRequest对象就可以了。

## 3.into()方法

上一步`load()`方法获取到了一个DrawableTypeRequest对象，我们接着来看它的`into()`方法。DrawableTypeRequest类中是没有定义`into()`方法的，我们来看它的父类DrawableRequestBuilder：

**DrawableRequestBuilder的into()方法**

```java
@Override
public Target<GlideDrawable> into(ImageView view) {
    return super.into(view);
}
```

可以看出方法内部又调用了父类**GenericRequestBuilder**的`into()`方法：

**GenericRequestBuilder的into()方法**

```java
public Target<TranscodeType> into(ImageView view) {
    // ...
    return into(glide.buildImageViewTarget(view, transcodeClass));
}
```

方法内部最终调用了重载方法，在此之前会调用Glide的`buildImageViewTarget()`方法创建出一个**Target**对象。

```java
// Glide的buildImageViewTarget()方法
<R> Target<R> buildImageViewTarget(ImageView imageView, Class<R> transcodedClass) {
    return imageViewTargetFactory.buildTarget(imageView, transcodedClass);
}

// ImageViewTargetFactory的buildTarget()方法
public <Z> Target<Z> buildTarget(ImageView view, Class<Z> clazz) {
    if (GlideDrawable.class.isAssignableFrom(clazz)) {
        return (Target<Z>) new GlideDrawableImageViewTarget(view);
    } else if (Bitmap.class.equals(clazz)) {
        return (Target<Z>) new BitmapImageViewTarget(view);
    } else if (Drawable.class.isAssignableFrom(clazz)) {
        return (Target<Z>) new DrawableImageViewTarget(view);
    } else {
        throw new IllegalArgumentException("Unhandled class: " + clazz
                + ", try .as*(Class).transcode(ResourceTranscoder)");
    }
}
```

方法内部会调用**ImageViewTargetFactory**的`buildTarget()`方法，根据传入的Class类型创建出相应的Target对象，这个Class类型是从哪里传过来的呢，我简单梳理了一下：

> RequestManager的`load()`方法→`loadGeneric()`方法创建DrawableTypeRequest对象→父类DrawableRequestBuilder构造方法→父类GenericRequestBuilder构造方法，为transcodedClass赋值→`into()`方法→Glide的`buildImageViewTarget()`方法→ImageViewTargetFactory的`buildTarget()`方法

就是这样一步一步传过来的，正常情况下`buildTarget()`方法传入的Class类型为**GlideDrawable.class**，但是如果我们在`into()`方法之前调用了`asBitmap()`方法，那么这里传入的Class类型就为**Bitmap.class**，至于Class为**Drawable.class**的情况一般是用不到的，先不去管它。这里我们只研究最简单的情况，因此`buildTarget()`方法返回的类型为**GlideDrawableImageViewTarget**，我们回到GenericRequestBuilder中的into()方法，接下来会调用`into()`的另一个重载方法，参数传入了GlideDrawableImageViewTarget对象：

**GenericRequestBuilder的into()方法**

```java
public <Y extends Target<TranscodeType>> Y into(Y target) {
    // ...
  	// 创建Request对象
    Request request = buildRequest(target);
    target.setRequest(request);
    lifecycle.addListener(target);
  	// 执行Request
    requestTracker.runRequest(request);

    return target;
}
```

方法内部首先调用了`buildRequest()`方法创建出一个**Request**类型对象：

```java
private Request buildRequest(Target<TranscodeType> target) {
    if (priority == null) {
        priority = Priority.NORMAL;
    }
    return buildRequestRecursive(target, null);
}

private Request buildRequestRecursive(Target<TranscodeType> target, ThumbnailRequestCoordinator parentCoordinator) {
    // ...
    return obtainRequest(target, sizeMultiplier, priority, parentCoordinator);
}

private Request obtainRequest(Target<TranscodeType> target, float sizeMultiplier, Priority priority,
                              RequestCoordinator requestCoordinator) {
    return GenericRequest.obtain(
            loadProvider,
            model,
            signature,
            context,
            priority,
            target,
            sizeMultiplier,
            placeholderDrawable,
            placeholderId,
            errorPlaceholder,
            errorId,
            fallbackDrawable,
            fallbackResource,
            requestListener,
            requestCoordinator,
            glide.getEngine(),
            transformation,
            transcodeClass,
            isCacheable,
            animationFactory,
            overrideWidth,
            overrideHeight,
            diskCacheStrategy);
}
```

`buildRequest()`方法内部调用了`buildRequestRecursive()`方法，`buildRequestRecursive()`方法这里省略了大量缩略图处理的相关代码，最后会调用`obtainRequest()`方法，接下来又会调用**GenericRequest**的`obtain()`方法。

**GenericRequest的obtain()方法**

```java
public static <A, T, Z, R> GenericRequest<A, T, Z, R> obtain(
        LoadProvider<A, T, Z, R> loadProvider,
        A model,
        Key signature,
        Context context,
        Priority priority,
        Target<R> target,
        float sizeMultiplier,
        Drawable placeholderDrawable,
        int placeholderResourceId,
        Drawable errorDrawable,
        int errorResourceId,
        Drawable fallbackDrawable,
        int fallbackResourceId,
        RequestListener<? super A, R> requestListener,
        RequestCoordinator requestCoordinator,
        Engine engine,
        Transformation<Z> transformation,
        Class<R> transcodeClass,
        boolean isMemoryCacheable,
        GlideAnimationFactory<R> animationFactory,
        int overrideWidth,
        int overrideHeight,
        DiskCacheStrategy diskCacheStrategy) {
    @SuppressWarnings("unchecked")
    GenericRequest<A, T, Z, R> request = (GenericRequest<A, T, Z, R>) REQUEST_POOL.poll();
    if (request == null) {
        request = new GenericRequest<A, T, Z, R>();
    }
    request.init(loadProvider,
            model,
            signature,
            context,
            priority,
            target,
            sizeMultiplier,
            placeholderDrawable,
            placeholderResourceId,
            errorDrawable,
            errorResourceId,
            fallbackDrawable,
            fallbackResourceId,
            requestListener,
            requestCoordinator,
            engine,
            transformation,
            transcodeClass,
            isMemoryCacheable,
            animationFactory,
            overrideWidth,
            overrideHeight,
            diskCacheStrategy);
    return request;
}
```

虽然参数很多，但是逻辑比较简单，就是创建出一个**GenericRequest**对象，调用`init()`方法对它的一些属性进行赋值。到这里就创建好了Request对象，我们回到GenericRequestBuilder的`into()`方法，接下来会执行`requestTracker.runRequest(request)`，requestTracker的类型为**RequestTracker**，我们来看一下RequestTracker的`runRequest()`方法：

**RequestTracker的runRequest()方法**

```java
public void runRequest(Request request) {
    requests.add(request);
    if (!isPaused) {
        request.begin();
    } else {
        pendingRequests.add(request);
    }
}
```

方法内部首先将Request对象添加到一个Set集合中，然后会判断Glide当前是否为暂停状态，如果不为暂停状态就直接调用Request对象的`begin()`方法执行这个Request，否则就将Request添加到一个延迟执行的集合中，等待执行。我们这里先不管什么暂停不暂停的，直接来看Request的`begin()`方法，根据上面的分析，这里的Request类型为GenericRequest。

**GenericRequest的begin()方法**

```java
public void begin() {
    // ...
    if (model == null) {
        onException(null);
        return;
    }
    // ...
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        onSizeReady(overrideWidth, overrideHeight);
    } else {
        target.getSize(this);
    }

    if (!isComplete() && !isFailed() && canNotifyStatusChanged()) {
        target.onLoadStarted(getPlaceholderDrawable());
    }
    // ...
}
```

这里的Model对应`load()`方法传入的图片路径，如果model为null，会执行`onException()`方法，我们可以点进去看一下：

```java
@Override
public void onException(Exception e) {
    // ...
    if (requestListener == null || !requestListener.onException(e, model, target, isFirstReadyResource())) {
        setErrorPlaceholder(e);
    }
}

private void setErrorPlaceholder(Exception e) {
    // ...
    Drawable error = model == null ? getFallbackDrawable() : null;
    if (error == null) {
        error = getErrorDrawable();
    }
    if (error == null) {
        error = getPlaceholderDrawable();
    }
    target.onLoadFailed(e, error);
}
```

接下来会调用`setErrorPlaceholder()`方法，在方法内部获取到error占位图Drawable，调用target的`onLoadFailed()`方法，这里的target类型为我们此前分析过的GlideDrawableImageViewTarget。

**GlideDrawableImageViewTarget的onLoadFailed()方法**

```java
@Override
public void onLoadFailed(Exception e, Drawable errorDrawable) {
    view.setImageDrawable(errorDrawable);
}
```

可以看出这里终于看到设置图片的地方，作用就是设置发生错误的占位图。

回到GenericRequest的`begin()`方法，接下有两种情况，如果我们调用了`override()`方法设置加载图片的宽高，就会调用`onSizeReady()`方法，反之则调用GlideDrawableImageViewTarget的`getSize()`方法，通过ImageView指定的尺寸参数确定加载图片的宽高，最终也会调用`onSizeReady()`方法，因此我们直接来看`onSizeReady()`方法：

```java
@Override
public void onSizeReady(int width, int height) {
    // ...
    width = Math.round(sizeMultiplier * width);
    height = Math.round(sizeMultiplier * height);

    ModelLoader<A, T> modelLoader = loadProvider.getModelLoader();
    final DataFetcher<T> dataFetcher = modelLoader.getResourceFetcher(model, width, height);
    // ...
    ResourceTranscoder<Z, R> transcoder = loadProvider.getTranscoder();
    // ...
    loadStatus = engine.load(signature, width, height, dataFetcher, loadProvider, transformation, transcoder,
            priority, isMemoryCacheable, diskCacheStrategy, this);
    // ...
}
```

我们注意到`onSizeReady()`方法中使用到了一个叫loadProvider的变量，关于这个变量的赋值就比较复杂了，我直接梳理一下整个赋值的流程，首先列出我们此前接触过的几个类之间的继承关系，如下所示，箭头方向指向父类：

DrawableTypeRequest→DrawableRequestBuilder→GenericRequestBuilder

在分析`load()`方法时，提到过在RequestManager的`loadGeneric()`方法中创建出了DrawableTypeRequest对象，我们来看一下它的构造方法：

**DrawableTypeRequest的构造方法**

```java
DrawableTypeRequest(Class<ModelType> modelClass, ModelLoader<ModelType, InputStream> streamModelLoader,
                    ModelLoader<ModelType, ParcelFileDescriptor> fileDescriptorModelLoader, Context context, Glide glide,
                    RequestTracker requestTracker, Lifecycle lifecycle, RequestManager.OptionsApplier optionsApplier) {
    super(context, modelClass,
            buildProvider(glide, streamModelLoader, fileDescriptorModelLoader, GifBitmapWrapper.class,
                    GlideDrawable.class, null),
            glide, requestTracker, lifecycle);
    this.streamModelLoader = streamModelLoader;
    this.fileDescriptorModelLoader = fileDescriptorModelLoader;
    this.optionsApplier = optionsApplier;
}

private static <A, Z, R> FixedLoadProvider<A, ImageVideoWrapper, Z, R> buildProvider(Glide glide,
                                                                                     ModelLoader<A, InputStream> streamModelLoader,
                                                                                     ModelLoader<A, ParcelFileDescriptor> fileDescriptorModelLoader, Class<Z> resourceClass,
                                                                                     Class<R> transcodedClass,
                                                                                     ResourceTranscoder<Z, R> transcoder) {
    if (streamModelLoader == null && fileDescriptorModelLoader == null) {
        return null;
    }

    if (transcoder == null) {
        // 创建GifBitmapWrapperDrawableTranscoder对象
        transcoder = glide.buildTranscoder(resourceClass, transcodedClass);
    }
    // 创建ImageVideoGifDrawableLoadProvider对象
    DataLoadProvider<ImageVideoWrapper, Z> dataLoadProvider = glide.buildDataProvider(ImageVideoWrapper.class,
            resourceClass);
  	// 创建ImageVideoModelLoader对象，streamModelLoader(类型为StreamStringLoader)和fileDescriptorModelLoader(类型为FileDescriptoStringLoader)是在RequestManager的loadGeneric()方法中创建的
    ImageVideoModelLoader<A> modelLoader = new ImageVideoModelLoader<A>(streamModelLoader,
            fileDescriptorModelLoader);
  	// 返回一个FixedLoadProvider对象
    return new FixedLoadProvider<A, ImageVideoWrapper, Z, R>(modelLoader, transcoder, dataLoadProvider);
}
```

构造方法中调用了`buildProvider()`方法，该方法返回的参数会传递到GenericRequestBuilder的构造方法中，为成员变量**loadProvider**赋值：

**GenericRequestBuilder的构造方法**

```java
GenericRequestBuilder(Context context, Class<ModelType> modelClass,
                      LoadProvider<ModelType, DataType, ResourceType, TranscodeType> loadProvider,
                      Class<TranscodeType> transcodeClass, Glide glide, RequestTracker requestTracker, Lifecycle lifecycle) {
    this.context = context;
    this.modelClass = modelClass;
    this.transcodeClass = transcodeClass;
    this.glide = glide;
    this.requestTracker = requestTracker;
    this.lifecycle = lifecycle;
    this.loadProvider = loadProvider != null
            ? new ChildLoadProvider<ModelType, DataType, ResourceType, TranscodeType>(loadProvider) : null;

    if (context == null) {
        throw new NullPointerException("Context can't be null");
    }
    if (modelClass != null && loadProvider == null) {
        throw new NullPointerException("LoadProvider must not be null");
    }
}
```

这里会把FixedLoadProvider包装为**ChildLoadProvider**，这之后在调用`obtainRequest()`方法创建GenericRequest时传入loadProvider，GenericRequest中的loadProvider即`onSizeReady()`方法中的loadProvider就是这样一步一步传过来的。接下来我们回到`onSizeReady()`方法：

```java
@Override
public void onSizeReady(int width, int height) {
    // ...
    width = Math.round(sizeMultiplier * width);
    height = Math.round(sizeMultiplier * height);
	// ImageVideoModelLoader
    ModelLoader<A, T> modelLoader = loadProvider.getModelLoader();
  	// ImageVideoFetcher
    final DataFetcher<T> dataFetcher = modelLoader.getResourceFetcher(model, width, height);
    // ...
  	// GifBitmapWrapperDrawableTranscoder
    ResourceTranscoder<Z, R> transcoder = loadProvider.getTranscoder();
    // ...
    loadStatus = engine.load(signature, width, height, dataFetcher, loadProvider, transformation, transcoder,
            priority, isMemoryCacheable, diskCacheStrategy, this);
    // ...
}
```

虽然这里loadProvider的类型为ChildLoadProvider，但其实他就是一个包装类，`getModelLoader()`方法内部依然是调用FixedLoadProvider的`getModelLoader()`方法，获取的是一个**ImageVideoModelLoader**对象，同理`getTranscoder()`方法获取到的是一个**GifBitmapWrapperDrawableTranscoder**对象。中间调用ImageVideoModelLoader的`getResourceFetcher()`方法，我们来看一下这个方法：

**ImageVideoModelLoader的getResourceFetcher()方法**

```java
@Override
public DataFetcher<ImageVideoWrapper> getResourceFetcher(A model, int width, int height) {
    DataFetcher<InputStream> streamFetcher = null;
    if (streamLoader != null) {
      	// streamLoader的类型为StreamStringLoader
        streamFetcher = streamLoader.getResourceFetcher(model, width, height);
    }
    DataFetcher<ParcelFileDescriptor> fileDescriptorFetcher = null;
    if (fileDescriptorLoader != null) {
      	// fileDescriptorLoader的类型为FileDescriptoStringLoader
        fileDescriptorFetcher = fileDescriptorLoader.getResourceFetcher(model, width, height);
    }

    if (streamFetcher != null || fileDescriptorFetcher != null) {
        return new ImageVideoFetcher(streamFetcher, fileDescriptorFetcher);
    } else {
        return null;
    }
}
```

`getResourceFetcher()`方法内部执行了`streamLoader.getResourceFetcher()`，这里的streamLoader就是此前RequestManager的`loadGeneric()`方法创建的StreamStringLoader对象，`getResourceFetcher()`方法获取到的是一个**HttpUrlFetcher**对象。下面的fileDescriptorLoader同样是在`loadGeneric()`方法中创建的，类型为FileDescriptoStringLoader，我在调试时发现调用`getResourceFetcher()`方法获取到fileDescriptorFetcher为null，因此我们这里就先不去管这个变量的作用了。方法最后返回了一个**ImageVideoFetcher**对象，创建时传入了HttpUrlFetcher对象。

回到`onSizeReady()`方法，在创建好上面这些对象后，执行了`engine.load()`方法，参数传入了这些对象。engine的类型为**Engine**，我们来看一下它的`load()`方法：

**Engine的load()方法**

```java
public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
                                 DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
                                 Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {
    // ...
    EngineJob engineJob = engineJobFactory.build(key, isMemoryCacheable);
    DecodeJob<T, Z, R> decodeJob = new DecodeJob<T, Z, R>(key, width, height, fetcher, loadProvider, transformation,
            transcoder, diskCacheProvider, diskCacheStrategy, priority);
    EngineRunnable runnable = new EngineRunnable(engineJob, decodeJob, priority);
    jobs.put(key, engineJob);
    engineJob.addCallback(cb);
    engineJob.start(runnable);
    // ...
}
```

这里省略了一些缓存处理的代码，首先创建了一个**EngineJob**对象，它的作用就是开启子线程，因为图片的加载是耗时操作，不能在主线程执行。第二步是创建一个**DecodeJob**对象，它的作用后面会具体分析。之后创建了一个**EngineRunnable**对象，它继承自Runable，很明显适用于执行任务的，我们也发现方法的最后调用的EngineJob的`start()`方法，方法内部使用线程池执行EngineRunnable任务，我们来看一下EngineRunnable中的`run()`方法：

**EngineRunnable的run()方法**

```java
@Override
public void run() {
    // ...
  	Resource<?> resource = null;
    resource = decode();
    // ...
    if (resource == null) {
        onLoadFailed(exception);
    } else {
        onLoadComplete(resource);
    }
}
```

`run()`方法内部首先调用了`decode()`方法获取到一个Resource对象。

```java
private Resource<?> decode() throws Exception {
    if (isDecodingFromCache()) {
        return decodeFromCache();
    } else {
        return decodeFromSource();
    }
}

private Resource<?> decodeFromSource() throws Exception {
    return decodeJob.decodeFromSource();
}
```

`decode()`方法内部又会调用`decodeFromSource()`方法（这里先不管缓存的问题），接着调用了DecodeJob的`decodeFromSource()`方法。

**DecodeJob的decodeFromSource()方法**

```java
public Resource<Z> decodeFromSource() throws Exception {
    Resource<T> decoded = decodeSource();
    return transformEncodeAndTranscode(decoded);
}
```

方法只有两行，我们先来看第一行调用的`decodeSource()`方法：

```java
private Resource<T> decodeSource() throws Exception {
    Resource<T> decoded = null;
    // ...
    // 关键代码
    // fetcher的类型为ImageVideoModelLoader
    final A data = fetcher.loadData(priority);
    // ...
    decoded = decodeFromSourceData(data);
    return decoded;
}
```

`decodeSource()`方法首先调用了fetcher的`loadData()`方法，这个fetcher就是`onSizeReady()`方法中调用ImageVideoModelLoader的`getResourceFetcher()`方法得到的ImageVideoFetcher对象，我们来看一下它的`loadData()`方法：

**ImageVideoFetcher的loadData()方法**

```java
@Override
public ImageVideoWrapper loadData(Priority priority) throws Exception {
    InputStream is = null;
    if (streamFetcher != null) {
        // ...
        // streamFetcher的类型为HttpUrlFetcher
        is = streamFetcher.loadData(priority);
        // ...
    }
    ParcelFileDescriptor fileDescriptor = null;
    if (fileDescriptorFetcher != null) {
        // ...
        fileDescriptor = fileDescriptorFetcher.loadData(priority);
        // ...
    }
    return new ImageVideoWrapper(is, fileDescriptor);
}
```

方法内部调用streamFetcher的`loadData()`方法获取到了一个InputStream，这个streamFetcher是什么呢，它就是我们上面分析的通过StreamStringLoader的`getResourceFetcher()`方法获取到的HttpUrlFetcher对象，我们来看它的`loadData()`方法：

**HttpUrlFetcher的loadData()方法**

```java
@Override
public InputStream loadData(Priority priority) throws Exception {
    return loadDataWithRedirects(glideUrl.toURL(), 0 /*redirects*/, null /*lastUrl*/, glideUrl.getHeaders());
}

private InputStream loadDataWithRedirects(URL url, int redirects, URL lastUrl, Map<String, String> headers)
        throws IOException {
    if (redirects >= MAXIMUM_REDIRECTS) {
        throw new IOException("Too many (> " + MAXIMUM_REDIRECTS + ") redirects!");
    } else {
        // Comparing the URLs using .equals performs additional network I/O and is generally broken.
        // See http://michaelscharf.blogspot.com/2006/11/javaneturlequals-and-hashcode-make.html.
        try {
            if (lastUrl != null && url.toURI().equals(lastUrl.toURI())) {
                throw new IOException("In re-direct loop");
            }
        } catch (URISyntaxException e) {
            // Do nothing, this is best effort.
        }
    }
    urlConnection = connectionFactory.build(url);
    for (Map.Entry<String, String> headerEntry : headers.entrySet()) {
        urlConnection.addRequestProperty(headerEntry.getKey(), headerEntry.getValue());
    }
    urlConnection.setConnectTimeout(2500);
    urlConnection.setReadTimeout(2500);
    urlConnection.setUseCaches(false);
    urlConnection.setDoInput(true);

    // Connect explicitly to avoid errors in decoders if connection fails.
    urlConnection.connect();
    if (isCancelled) {
        return null;
    }
    final int statusCode = urlConnection.getResponseCode();
    if (statusCode / 100 == 2) {
      	// 请求成功
        return getStreamForSuccessfulRequest(urlConnection);
    } else if (statusCode / 100 == 3) {
        String redirectUrlString = urlConnection.getHeaderField("Location");
        if (TextUtils.isEmpty(redirectUrlString)) {
            throw new IOException("Received empty or null redirect url");
        }
        URL redirectUrl = new URL(url, redirectUrlString);
        return loadDataWithRedirects(redirectUrl, redirects + 1, url, headers);
    } else {
        if (statusCode == -1) {
            throw new IOException("Unable to retrieve response code from HttpUrlConnection.");
        }
        throw new IOException("Request failed " + statusCode + ": " + urlConnection.getResponseMessage());
    }
}

private InputStream getStreamForSuccessfulRequest(HttpURLConnection urlConnection)
        throws IOException {
    if (TextUtils.isEmpty(urlConnection.getContentEncoding())) {
        int contentLength = urlConnection.getContentLength();
        stream = ContentLengthInputStream.obtain(urlConnection.getInputStream(), contentLength);
    } else {
        if (Log.isLoggable(TAG, Log.DEBUG)) {
            Log.d(TAG, "Got non empty content encoding: " + urlConnection.getContentEncoding());
        }
        stream = urlConnection.getInputStream();
    }
    return stream;
}
```

`loadData()`方法内部又调用了`loadDataWithRedirects()`方法，而`loadDataWithRedirects()`方法我们粗略看一下大概也能看出这里就是网络请求的处理，这里使用的是**HttpURLConnection**，根据响应的状态码判断如果请求成功就调用`getStreamForSuccessfulRequest()`方法获取到响应流InputStream。

回到ImageVideoFetcher的`loadData()`方法，获取到InputStream后，将它作为参数创建出了一个**ImageVideoWrapper**对象。再回到DecodeJob的`decodeSource()`方法，将返回的ImageVideoWrapper对象赋值给了data，然后调用了`decodeFromSourceData()`方法，我们来看一下。

**DecodeJob的decodeFromSourceData()方法**

```java
private Resource<T> decodeFromSourceData(A data) throws IOException {
    final Resource<T> decoded;
    if (diskCacheStrategy.cacheSource()) {
        decoded = cacheAndDecodeSourceData(data);
    } else {
        // ...
      	// 获取到的decoded类型为GifBitmapWrapperResource（Resource<GifBitmapWrapper>）
        decoded = loadProvider.getSourceDecoder().decode(data, width, height);
        // ...
    }
    return decoded;
}
```

同样地，我们先不去管if条件中关于缓存的处理，直接来看else分支，这里的loadProvider类型为ChildLoadProvider，包装着FixedLoadProvider对象，`getSourceDecoder()`方法获取到的是一个**GifBitmapWrapperResourceDecoder**对象，接下来调用了GifBitmapWrapperResourceDecoder的`decode()`方法。

```java
public Resource<GifBitmapWrapper> decode(ImageVideoWrapper source, int width, int height) throws IOException {
    ByteArrayPool pool = ByteArrayPool.get();
    byte[] tempBytes = pool.getBytes();

    GifBitmapWrapper wrapper = null;
    try {
      	// 调用decode()重载方法
        wrapper = decode(source, width, height, tempBytes);
    } finally {
        pool.releaseBytes(tempBytes);
    }
  	// 返回GifBitmapWrapperResource（Resource<GifBitmapWrapper>）对象
    return wrapper != null ? new GifBitmapWrapperResource(wrapper) : null;
}

private GifBitmapWrapper decode(ImageVideoWrapper source, int width, int height, byte[] bytes) throws IOException {
    final GifBitmapWrapper result;
    if (source.getStream() != null) {
      	// 获取到的result类型为GifBitmapWrapper
        result = decodeStream(source, width, height, bytes);
    } else {
        result = decodeBitmapWrapper(source, width, height);
    }
    return result;
}

private GifBitmapWrapper decodeStream(ImageVideoWrapper source, int width, int height, byte[] bytes)
        throws IOException {
    InputStream bis = streamFactory.build(source.getStream(), bytes);
    bis.mark(MARK_LIMIT_BYTES);
    // 获取图片格式
    ImageHeaderParser.ImageType type = parser.parse(bis);
    bis.reset();

    GifBitmapWrapper result = null;
    if (type == ImageHeaderParser.ImageType.GIF) {
        // 图片为gif动态图
        result = decodeGifWrapper(bis, width, height);
    }
    if (result == null) {
        // 图片为静态图
        ImageVideoWrapper forBitmapDecoder = new ImageVideoWrapper(bis, source.getFileDescriptor());
      	// 获取到的result类型为GifBitmapWrapper
        result = decodeBitmapWrapper(forBitmapDecoder, width, height);
    }
    return result;
}
```

`decode()`方法会调用另一个重载`decode()`方法，由于`source.getStream()`也就是此前获取到的InputStream不为null，因此会调用`decodeStream()`方法，`decodeStream()`方法首先会判断图片的格式，原理简单来说就是通过读取InputStream的前几个字节，读取前两个字节判断是否是为**JPEG**格式；读取前四个字节判断是否为**PNG**格式（再判断是否包含透明）；读取前三个字节判断是否为**GIF**格式，具体逻辑在**ImageHeaderParser**类的`getType()`方法中感兴趣的话可以看看。接下里会根据图片类型是否为GIF执行相应逻辑，我们这里不研究加载动态图的情况，因此接下来会调用`decodeBitmapWrapper()`方法。

```java
private GifBitmapWrapper decodeBitmapWrapper(ImageVideoWrapper toDecode, int width, int height) throws IOException {
    GifBitmapWrapper result = null;
	// bitmapDecoder的类型为ImageVideoBitmapDecoder
    Resource<Bitmap> bitmapResource = bitmapDecoder.decode(toDecode, width, height);
    if (bitmapResource != null) {
        result = new GifBitmapWrapper(bitmapResource, null);
    }

    return result;
}
```

方法内部又调用了bitmapDecoder的`decode()`方法，这里的bitmapDecoder类型为**ImageVideoBitmapDecoder**，我们来看一下它的`decode()`方法：

**ImageVideoBitmapDecoder的decode()方法**

```java
@Override
public Resource<Bitmap> decode(ImageVideoWrapper source, int width, int height) throws IOException {
    Resource<Bitmap> result = null;
    InputStream is = source.getStream();
    if (is != null) {
        // ...
        result = streamDecoder.decode(is, width, height);
        // ...
    }
    // ...
    return result;
}
```

接下来会调用streamDecoder的`decode()`方法，这里的streamDecoder类型为**StreamBitmapDecoder**，我们接着来看：

**StreamBitmapDecoder的decode()方法**

```java
@Override
public Resource<Bitmap> decode(InputStream source, int width, int height) {
    Bitmap bitmap = downsampler.decode(source, bitmapPool, width, height, decodeFormat);
    return BitmapResource.obtain(bitmap, bitmapPool);
}
```

方法内部首先调用了downsampler的`decode()`方法，这里的downsampler类型为**Downsampler**，我们来看它的`decode()`方法：

**Downsampler的decode()方法**

```java
@Override
public Bitmap decode(InputStream is, BitmapPool pool, int outWidth, int outHeight, DecodeFormat decodeFormat) {
    // ...
    final Bitmap downsampled =
            downsampleWithSize(invalidatingStream, bufferedStream, options, pool, inWidth, inHeight, sampleSize,
                    decodeFormat);
    // ...
}

private Bitmap downsampleWithSize(MarkEnforcingInputStream is, RecyclableBufferedInputStream bufferedStream,
                                  BitmapFactory.Options options, BitmapPool pool, int inWidth, int inHeight, int sampleSize,
                                  DecodeFormat decodeFormat) {
    // ...
    return decodeStream(is, bufferedStream, options);
}

private static Bitmap decodeStream(MarkEnforcingInputStream is, RecyclableBufferedInputStream bufferedStream,
                                   BitmapFactory.Options options) {
    // ...
    final Bitmap result = BitmapFactory.decodeStream(is, null, options);
    // ...
    return result;
}
```

上面的`decode()`方法省略了大量关于图片压缩、旋转等逻辑的处理，只展示了最关键的代码，接下来会调用`downsampleWithSize()`、`decodeStream()`方法，最终调用到了我们熟悉的**BitmapFactory**的`decodeStream()`方法，返回了Bitmap对象。接下来我们回到StreamBitmapDecoder的`decode()`方法，获取到Bitmap对象后，调用了BitmapResource的`obtain()`方法并返回该对象，我们来看一下这个方法：

**BitmapResource的obtain()方法**

```java
public static BitmapResource obtain(Bitmap bitmap, BitmapPool bitmapPool) {
    if (bitmap == null) {
        return null;
    } else {
        return new BitmapResource(bitmap, bitmapPool);
    }
}
```

可以看出该方法就是将Bitmap对象包装为了一个**BitmapResource**对象，BitmapResource实现了**Resource<Bitmap>**。接下来逐级返回，回到GifBitmapWrapperResourceDecoder的`decodeBitmapWrapper()`方法，在调用`bitmapDecoder.decode()`方法后，将返回的BitmapResource（Resource<Bitmap>）对象又封装成了一个**GifBitmapWrapper**对象，我们来简单看一下这个类的定义。

```java
public class GifBitmapWrapper {
    private final Resource<GifDrawable> gifResource;
    private final Resource<Bitmap> bitmapResource;

    public GifBitmapWrapper(Resource<Bitmap> bitmapResource, Resource<GifDrawable> gifResource) {
        // ...
        this.bitmapResource = bitmapResource;
        this.gifResource = gifResource;
    }
 
  	// ...

    public Resource<Bitmap> getBitmapResource() {
        return bitmapResource;
    }

    public Resource<GifDrawable> getGifResource() {
        return gifResource;
    }
}
```

可以看出GifBitmapWrapper就是对Resource<GifDrawable>和Resource<Bitmap>的包装，实现了对动态图和静态图的支持，可以通过调用`getBitmapResource()`和`getGifResource()`方法获取到相应的Resource对象。接下来我们回到GifBitmapWrapperResourceDecoder的`decode()`方法（三个参数那个），在获取到GifBitmapWrapper对象后，最后又将这个对象包装为了一个**GifBitmapWrapperResource**对象并返回，这个类实现了**Resource<GifBitmapWrapper>**。然后接着回溯，最终回到了DecodeJob的`decodeFromSource()`方法，这里再贴一遍代码：

```java
public Resource<Z> decodeFromSource() throws Exception {
  	// 获取到的decoded类型为GifBitmapWrapperResource(Resource<GifBitmapWrapper>)
    Resource<T> decoded = decodeSource();
    return transformEncodeAndTranscode(decoded);
}
```

在获取到Resource<GifBitmapWrapper>对象后，接下来会调用`transformEncodeAndTranscode()`方法：

```java
private Resource<Z> transformEncodeAndTranscode(Resource<T> decoded) {
    // ...
    Resource<T> transformed = transform(decoded);
    // ...
    Resource<Z> result = transcode(transformed);
    // ...
    return result;
}

private Resource<Z> transcode(Resource<T> transformed) {
    if (transformed == null) {
        return null;
    }
    return transcoder.transcode(transformed);
}
```

`transformEncodeAndTranscode()`方法会调用`transcode()`方法，在`transcode()`方法中调用了transcoder的`transcode()`方法，这个transcoder就是此前分析过的在DrawableTypeRequest的`buildProvider()`方法中创建的GifBitmapWrapperDrawableTranscoder对象，之后在创建GenericRequest对象时作为参数传了进来，紧接着在`onSizeReady()`方法中传递到了Engine的`load()`方法，并在创建DecodeJob时传入。我们接着来看GifBitmapWrapperDrawableTranscoder的`transcode()`方法：

**GifBitmapWrapperDrawableTranscoder的transcode()方法**

```java
@Override
public Resource<GlideDrawable> transcode(Resource<GifBitmapWrapper> toTranscode) {
    GifBitmapWrapper gifBitmap = toTranscode.get();
    Resource<Bitmap> bitmapResource = gifBitmap.getBitmapResource();

    final Resource<? extends GlideDrawable> result;
    if (bitmapResource != null) {
        // 加载静态图
        result = bitmapDrawableResourceTranscoder.transcode(bitmapResource);
    } else {
        // 加载动态图
        result = gifBitmap.getGifResource();
    }
    return (Resource<GlideDrawable>) result;
}
```

这里首先取出了GifBitmapWrapper中的Resource<Bitmap>对象，如果Resource<Bitmap>为null，说明加载的是动态图片，直接获取GifBitmapWrapper中的Resource<GifDrawable>对象，反之加载的是静态图片，调用bitmapDrawableResourceTranscoder的`transcode()`方法。这样做的目的就是为了统一静态图和动态图的加载逻辑，由于GifDrawable继承自GlideDrawable，从而是继承自Drawable的，因此这里需要将Bitmap也转换为Drawable类型。这里的bitmapDrawableResourceTranscoder类型为**GlideBitmapDrawableTranscoder**，我们来看一下它的`transcode()`方法：

**GlideBitmapDrawableTranscoder的transcode()方法**

```java
@Override
public Resource<GlideBitmapDrawable> transcode(Resource<Bitmap> toTranscode) {
    GlideBitmapDrawable drawable = new GlideBitmapDrawable(resources, toTranscode.get());
    return new GlideBitmapDrawableResource(drawable, bitmapPool);
}
```

方法的逻辑很简单，首先获取到Resource<Bitmap>中的Bitmap对象，然后将这个Bitmap对象包装为**GlideBitmapDrawable**对象，最后再把这个GlideBitmapDrawable对象包装为一个**GlideBitmapDrawableResource**对象返回，GlideBitmapDrawableResource实现了DrawableResource<GlideBitmapDrawable>，而DrawableResource实现了Resource接口，因此可以看做是一个Resource<GlideBitmapDrawable>对象。

现在我们回到GifBitmapWrapperDrawableTranscoder的`transcode()`方法，现在无论是静态图还是动态图，最终都可以转换为Resource<GlideDrawable>对象（GlideDrawable和GifDrawable都继承自GlideDrawable），回到DecodeJob的`decodeFromSource()`方法，我们现在清楚了`transformEncodeAndTranscode()`方法的作用就是将Resource所指定的泛型类型进行转换，最终返回一个Resource<GlideDrawable>对象。这个对象会返回给EngineRunnable的`decodeFromSource()`方法，最终回到`run()`方法，这里再贴一遍代码：

**EngineRunnable的run()方法**

```java
@Override
public void run() {
    // ...
  	Resource<?> resource = null;
  	// 获取到的resource类型为Resource<GlideDrawable>
    resource = decode();
    // ...
    if (resource == null) {
        onLoadFailed(exception);
    } else {
        onLoadComplete(resource);
    }
}
```

此时已经获取到了Resource<GlideDrawable>对象，接下来来到了最后的环节——图片的展示，这里我们分别来看一下图片加载成功和失败的情况。先来看图片加载失败的情况，对应resource==null，接下来会执行`onLoadFailed()`方法：

```java
private void onLoadFailed(Exception e) {
    if (isDecodingFromCache()) {
        stage = Stage.SOURCE;
        manager.submitForSource(this);
    } else {
        manager.onException(e);
    }
}
```

同样是不管缓存加载的情况，接着会调用manager的`onException()`方法，这里的manager类型为EngineJob，是在创建EngineRunnable时传入的，我们来看EngineJob的`onException()`方法：

**EngineJob的onException()方法**

```java
@Override
public void onException(final Exception e) {
    this.exception = e;
    MAIN_THREAD_HANDLER.obtainMessage(MSG_EXCEPTION, this).sendToTarget();
}
```

**MAIN_THREAD_HANDLER**是一个Handler对象，这里就是使用Handler发送了一条消息，我们找到对消息的处理。

```java
private static final Handler MAIN_THREAD_HANDLER = new Handler(Looper.getMainLooper(), new MainThreadCallback());

private static class MainThreadCallback implements Handler.Callback {

    @Override
    public boolean handleMessage(Message message) {
        if (MSG_COMPLETE == message.what || MSG_EXCEPTION == message.what) {
            EngineJob job = (EngineJob) message.obj;
            if (MSG_COMPLETE == message.what) {
              	// 图片加载成功
                job.handleResultOnMainThread();
            } else {
              	// 图片加载失败
                job.handleExceptionOnMainThread();
            }
            return true;
        }

        return false;
    }
}
```

我们发现其实加载成功的逻辑也在这里，没关系，一会儿再回来分析，这里我们先关注加载失败的情况，接下来会调用`handleExceptionOnMainThread()`方法：

```java
private void handleExceptionOnMainThread() {
    // ...
    for (ResourceCallback cb : cbs) {
        if (!isInIgnoredCallbacks(cb)) {
          	// 调用ResourceCallback的onException()方法
            cb.onException(exception);
        }
    }
}
```

方法内部会遍历所有的ResourceCallback对象，调用它的`onException()`方法，cbs是一个ResourceCallback集合，ResourceCallback是通过调用EngineJob的`addCallback()`方法添加的，这个方法是在Engine的`load()`方法中调用的，这里再贴一遍代码。

**Engine的load()方法**

```java
public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
                                 DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
                                 Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {
    // ...
    EngineJob engineJob = engineJobFactory.build(key, isMemoryCacheable);
    DecodeJob<T, Z, R> decodeJob = new DecodeJob<T, Z, R>(key, width, height, fetcher, loadProvider, transformation,
            transcoder, diskCacheProvider, diskCacheStrategy, priority);
    EngineRunnable runnable = new EngineRunnable(engineJob, decodeJob, priority);
    jobs.put(key, engineJob);
  	// 这里调用了addCallback()方法
    engineJob.addCallback(cb);
    engineJob.start(runnable);
    // ...
}
```

这里的参数传入了cb，这个cb是通过`load()`方法的参数传过来的，还记得是哪里调用的`load()`方法吗，是GenericRequest的`onSizeReady()`方法。

**GenericRequest的onSizeReady()方法**

```java
@Override
public void onSizeReady(int width, int height) {
    // ...
    loadStatus = engine.load(signature, width, height, dataFetcher, loadProvider, transformation, transcoder,
            priority, isMemoryCacheable, diskCacheStrategy, this);
    // ...
}
```

我们会发现调用方法时cb参数传入了this，再来看GenericRequest类的定义，原来它实现了ResourceCallback接口，因此就是一个ResourceCallback对象，我们找到GenericRequest中重写的`onException()`方法，

```java
@Override
public void onException(Exception e) {
    // ...
    if (requestListener == null || !requestListener.onException(e, model, target, isFirstReadyResource())) {
        setErrorPlaceholder(e);
    }
}
```

有没有一点熟悉的感觉，其实我们早在分析GenericRequest的`begin()`方法时就看到这个方法了，当传入的图片路径为null时就会调用`onException()`方法。接下来的过程我就不分析了，就是获取到error占位图，显示到ImageView上。

现在我们回到EngineRunnable的`run()`方法，图片加载成功对应resource不为null，接下来会调用`onLoadComplete()`方法。

```java
private void onLoadComplete(Resource resource) {
    manager.onResourceReady(resource);
}
```

接着调用EngineJob的`onResourceReady()`方法：

```java
@Override
public void onResourceReady(final Resource<?> resource) {
    this.resource = resource;
    MAIN_THREAD_HANDLER.obtainMessage(MSG_COMPLETE, this).sendToTarget();
}

private static class MainThreadCallback implements Handler.Callback {

    @Override
    public boolean handleMessage(Message message) {
        if (MSG_COMPLETE == message.what || MSG_EXCEPTION == message.what) {
            EngineJob job = (EngineJob) message.obj;
            if (MSG_COMPLETE == message.what) {
              	// 图片加载成功
                job.handleResultOnMainThread();
            } else {
              	// 图片加载失败
                job.handleExceptionOnMainThread();
            }
            return true;
        }

        return false;
    }
}

private void handleResultOnMainThread() {
    // ...
    for (ResourceCallback cb : cbs) {
        if (!isInIgnoredCallbacks(cb)) {
            engineResource.acquire();
            cb.onResourceReady(engineResource);
        }
    }
    engineResource.release();
}
```

流程和加载失败的情况类似，使用Handler发送一条消息，最终调用`handleResultOnMainThread()`方法，方法内部会循环调用ResourceCallback的`onResourceReady()`方法，根据上面的分析，这里的ResourceCallback就是GenericRequest，我们找到它的`onResourceReady()`方法：

**GenericRequest的onResourceReady()方法**

```java
@Override
public void onResourceReady(Resource<?> resource) {
    // ...
    // 这里获取到的received类型为GlideDrawable
    Object received = resource.get();
    // ...
    onResourceReady(resource, (R) received);
}

private void onResourceReady(Resource<?> resource, R result) {
    // ...
    target.onResourceReady(result, animation);
    // ...
}
```

`onResourceReady()`方法内部又调用了另一个重载方法，接着会调用target的`onResourceReady()`方法，这里的target类型为GlideDrawableImageViewTarget，我们来看一下它的`onResourceReady()`方法：

**GlideDrawableImageViewTarget的onResourceReady()方法**

```
@Override
public void onResourceReady(GlideDrawable resource, GlideAnimation<? super GlideDrawable> animation) {
    // ...
    super.onResourceReady(resource, animation);
    this.resource = resource;
    resource.setLoopCount(maxLoopCount);
    resource.start();
}
```

方法内部又调用了父类ImageViewTarget的`onResourceReady()`方法，我们来看一下：

**ImageViewTarget的onResourceReady()方法**

```java
@Override
public void onResourceReady(Z resource, GlideAnimation<? super Z> glideAnimation) {
    if (glideAnimation == null || !glideAnimation.animate(resource, this)) {
        setResource(resource);
    }
}
```

这里又调用到了`setResource()`方法，`setResource()`方法是一个抽象方法，实现定义在子类GlideDrawableImageViewTarget中。

**GlideDrawableImageViewTarget的setResource()方法**

```java
@Override
protected void setResource(GlideDrawable resource) {
    view.setImageDrawable(resource);
}
```

这里的view就是我们一开始在`into()`方法传入的ImageView，因此这里就是完成图片的显示。

分析到这里终于算是把整个流程走通了，我是深刻地感受到Glide背后复杂的实现原理，即使我是跟着郭神的博客一步一步分析的，在分析的过程中还是难免会走偏，主要的难点还是在于需要不停地回溯到开始调用的方法。

最后梳理一下Glide加载一张网络静态图片的几个关键步骤：

1.`with()`方法创建**RequestManager**对象，通过传入的参数确定了图片加载的生命周期

2.`load()`方法创建**DrawableTypeRequest**对象

3.将ImageView转换为**GlideDrawableImageViewTarget**

4.创建**GenericRequest**对象，调用`begin()`方法开始图片加载流程

5.创建**EngineRunnable**，负责图片加载任务

6.使用**HttpURLConnection**请求网络图片资源，获取到流InputStream

7.根据InputStream的前几个字节判断出图片格式，如果是静态图最终会通过BitmapFactory的`decodeStream()`方法获取到Bitmap

8.将Bitmap包装为**Resource<GifBitmapWrapper>**，目的是统一静态图和动态图的加载逻辑。包装的流程如下

Bitmap→Resource<Bitmap>→GifBitmapWrapper(包装Resource<Bitmap>和Resource<GifDrawable>)→GifBitmapWrapperResource(Resource<GifBitmapWrapper>)

9.将**Resource<GifBitmapWrapper>**转换为**Resource<GlideDrawable>**

10.获取**GlideDrawable**对象，显示到ImageView上






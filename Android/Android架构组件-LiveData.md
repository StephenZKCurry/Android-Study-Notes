# Android架构组件-LiveData

## 1.简介

[LiveData](https://developer.android.google.cn/topic/libraries/architecture/livedata)是Android官方推出的Android Jetpack中Architecture组件的一员，是可观察的数据持有类， 与常规Observable不同，LiveData是生命周期感知的，这确保了LiveDate仅在LifecycleOwner处于Active（STARTED/RESUMED）状态下才通知更新数据并且在DESTROYED状态下移除Observer，避免了内存泄漏。

## 2.简单使用

LiveData通常是配合ViewModel使用的，这里为了简单，单独使用LiveData。

```java
// 定义LiveData
MutableLiveData<String> liveData = new MutableLiveData<>();
// 注册观察者
liveData.observe(this, new Observer<String>() {
    @Override
    public void onChanged(@Nullable String s) {
        Log.d("TAG", "onChanged:" + s);
    }
});
// 改变数据的值
liveData.setValue("Stephen Curry");
```

首先定义了一个MutableLiveData，它是LiveData的一个子类，使用泛型声明了LiveData持有的数据类型，然后调用`observe()`方法注册观察者，之后调用`setValue()`或`postValue()`方法改变数据的值即会回调`onChanged()`方法。

需要注意的是，LiveDate仅在LifecycleOwner处于Active状态下才会通知更新数据，因此如果我们把`setValue()`或者`postValue()`方法放到`onCreate()`方法中，是无法回调`onChanged()`方法的。

## 3.原理分析

我们知道LiveData可以感知生命周期，那么它是如何实现的呢？我们首先来看LiveData的`observe()`方法。

```java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
  	// 如果当前是DESTROYED状态则不做处理
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
  	// 将observer封装为一个LifecycleBoundObserver对象
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
  	// 将观察者保存到一个Map中
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
  	// 感知生命周期
    owner.getLifecycle().addObserver(wrapper);
}
```

`observe()`方法有两个参数，第一个是LifecycleOwner，第二个是Observer，看到这里我们不难想到LiveData的生命周期感知也是通过Lifecycle来实现的。我们具体看一下方法内部，首先判断了LifecycleOwner当前的生命周期状态是否为DESTROYED，如果是就直接返回，不执行后面的注册观察者等等操作。然后将我们传入的Observer对象封装为了一个**LifecycleBoundObserver**对象，我们来看一下该类。

```java
class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {
    @NonNull final LifecycleOwner mOwner;

    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<T> observer) {
        super(observer);
        mOwner = owner;
    }

    @Override
    boolean shouldBeActive() {
      	// 只有在onStart()之后，onPause()之前才处于Active状态
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }

    @Override
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
      	// 如果当前状态为DESTROYED，就移除所有观察者
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
            removeObserver(mObserver);
            return;
        }
        activeStateChanged(shouldBeActive());
    }

    @Override
    boolean isAttachedTo(LifecycleOwner owner) {
        return mOwner == owner;
    }

    @Override
    void detachObserver() {
        mOwner.getLifecycle().removeObserver(this);
    }
}
```

LifecycleBoundObserver继承了ObserverWrapper，并实现了GenericLifecycleObserver接口，我们接着看。

```java
private abstract class ObserverWrapper {
    final Observer<T> mObserver;
    boolean mActive;
    int mLastVersion = START_VERSION;

    ObserverWrapper(Observer<T> observer) {
        mObserver = observer;
    }

    abstract boolean shouldBeActive();

    boolean isAttachedTo(LifecycleOwner owner) {
        return false;
    }

    void detachObserver() {
    }

    void activeStateChanged(boolean newActive) {
      	// 如果Active状态没有发生改变，则直接返回
        if (newActive == mActive) {
            return;
        }
        // immediately set active state, so we'd never dispatch anything to inactive
        // owner
        mActive = newActive;
        boolean wasInactive = LiveData.this.mActiveCount == 0;
        LiveData.this.mActiveCount += mActive ? 1 : -1;
        if (wasInactive && mActive) {
            onActive();
        }
        if (LiveData.this.mActiveCount == 0 && !mActive) {
            onInactive();
        }
        if (mActive) {
            dispatchingValue(this);
        }
    }
}

public interface GenericLifecycleObserver extends LifecycleObserver {
    /**
     * Called when a state transition event happens.
     *
     * @param source The source of the event
     * @param event The event
     */
    void onStateChanged(LifecycleOwner source, Lifecycle.Event event);
}
```

GenericLifecycleObserver继承自LifecycleObserver，因此它就是一个生命周期观察者，它的内部定义了一个`onStateChanged()`方法，当LifecycleOwner的生命周期发生改变时会回调该方法，我们发现在LifecycleBoundObserver的`onStateChanged()`方法中进行了判断，如果LifecycleOwner当前的生命周期状态是DESTROYED，就移除所有的观察者，避免了内存泄漏问题，紧接着调用了`activeStateChanged()`方法。

LifecycleBoundObserver继承了ObserverWrapper类，实现了`shouldBeActive()`方法，该方法返回当前LifecycleOwner是否处于Active状态，只有当生命周期状态至少为STARTED时才处于Activie状态，即`onStart()` 和`onPause()`之间。接着来看`activeStateChanged()`方法，首先判断了当前状态是否等于上一个状态，即状态是否改变，这里的状态指Active/InActive，如果状态没有发生改变就直接返回，不执行下面的操作；如果状态发生了改变，即Active→InActive或InActive→Active，就执行下面的逻辑，关键代码是最后一个判断：

```java
if (mActive) {
    dispatchingValue(this);
}
```

如果LifecycleOwner当前处于Active状态，执行`dispatchingValue()`方法，综合最开始的判断，也就是状态由InActive变为Active时会执行该方法。我们来看一下这个方法的定义。

```java
private void dispatchingValue(@Nullable ObserverWrapper initiator) {
  	// 省略部分代码
    if (initiator != null) {
        considerNotify(initiator);
        initiator = null;
    } else {
        // 循环获取LiveData添加的所有观察者
        for (Iterator<Map.Entry<Observer<T>, ObserverWrapper>> iterator =
             mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
            considerNotify(iterator.next().getValue());
            if (mDispatchInvalidated) {
                break;
            }
        }
    }
  	// 省略部分代码
}
```

这里省略了部分代码，我们只需要关注核心部分就好，这里判断了传入的initiator是否为空，虽然这里传入的initiator不为空，但无论是否为空最后都会执行`considerNotify()`方法，我们暂且先不研究initiator是否为null的具体意义，后面会具体分析，直接看`considerNotify()`方法。

```java
private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
    //
    // we still first check observer.active to keep it as the entrance for events. So even if
    // the observer moved to an active state, if we've not received that event, we better not
    // notify for a more predictable notification order.
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    //noinspection unchecked
    observer.mObserver.onChanged((T) mData);
}
```

首先判断如果当前LifecycleOwner处于InActive状态则直接返回，不执行后面的逻辑，中间的一些代码可以不去研究，直接看最后一行，看到这里整个流程就通了，观察者的`onChanged()`方法在这里被调用。

重新梳理一下上面分析的流程，LiveData注册的观察者首先会被封装为一个LifecycleBoundObserver对象，该对象是一个LifecycleObserver，当LifecycleOwner的生命周期改变时会回调它的`onStateChanged()`方法，此外LifecycleBoundObserver还定义了何种状态为Active状态，由InActive状态变为Active状态时会执行`dispatchingValue()`方法，进而执行观察者的`onChanged()`方法。

到这里还没完，我们还没有分析`setValue()`和`postValue()`方法，下面分别来看一下这两个方法。

```java
 private final Runnable mPostValueRunnable = new Runnable() {
    @Override
    public void run() {
        Object newValue;
        synchronized (mDataLock) {
            newValue = mPendingData;
            mPendingData = NOT_SET;
        }
        // 调用setValue
        setValue((T) newValue);
    }
};

protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        postTask = mPendingData == NOT_SET;
        mPendingData = value;
    }
    if (!postTask) {
        return;
    }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}

@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}
```

我们可以发现，`setValue()`方法只能在主线程调用，否则会抛出错误；`postValue()`方法内部也是调用了`setValue()`方法，只是将操作post到了主线程执行（通过Handler），因此`postValue()`可以在子线程中调用。既然如此我们只需要研究`setValue()`方法即可，可以看到方法内部调用了之前分析过的`dispatchingValue()`方法，不同的是这里传入的参数为null，记得`dispatchingValue()`方法内部判断了initiator是否为空，看到这里我们大概就清楚了该判断的作用，就是为了区分触发的条件是生命周期改变还是调用了`setValue()`/`postValue()`方法。在之前的分析中` dispatchingValue()`方法传入的参数为this，即ObserverWrapper对象，而这里传入了null，会从保存观察者的Map中循环取出所有LiveData注册的观察者，依次执行`considerNotify()`方法。后面的流程是一样的，这里就不分析了。

通过调用`observeForever()`方法来注册观察者，无论当前处于何种状态都可以接收到数据的改变，但是需要我们手动移除观察者，防止内存泄漏。

最后总结一下

1.LiveData的生命周期感知是通过Lifecycle实现的，将注册的观察者封装为LifecycleObserver，当LifecycleOwner的生命周期状态为DESTROYED时移除观察者，防止内存泄漏。

2.有两种情况会触发观察者的`onChanged()`方法，第一种情况是当LifecycleOwner的生命周期状态由InActive变为Active（STARTED/RESUMED）；第二种情况是调用LIveData的`setValue()`/`postValue()`方法，并且当前LifecycleOwner的生命周期状态为Active。

## 4.应用场景

由于LiveData基于观察者模式，可以用来实现类似EventBus或是RxBus的事件总线，网上有很多关于LiveDataBus 的介绍，可以自行查阅。

## 5.参考文章

[Android官方架构组件LiveData: 观察者模式领域二三事](https://juejin.im/post/5c25753af265da61561f5335)

[【AAC 系列三】深入理解架构组件：LiveData](https://mp.weixin.qq.com/s/glbd3mzU_cUPGAlJMlQ5Hg)

[LiveData官方文档](https://developer.android.google.cn/topic/libraries/architecture/livedata)
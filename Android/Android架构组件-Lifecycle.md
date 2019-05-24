# Android架构组件-Lifecycle

## 1.简介

[Lifecycle](https://developer.android.google.cn/topic/libraries/architecture/lifecycle)是Android官方推出的Android Jetpack中Architecture组件的一员，顾名思义，可以用来管理Activity和Fragment的生命周期，实现对生命周期的监听，为开发者提供了一种更优雅的方式来解决生命周期导致的内存泄漏问题。

## 2.简单使用

Lifecycle的使用还是比较简单的，使用步骤如下：

* 添加依赖

```groovy
implementation "android.arch.lifecycle:extensions:1.1.1"
```

* 定义LifecycleObserver（生命周期观察者）

```java
public class MyLifecycleObserver implements LifecycleObserver {

    private static final String TAG = "MyLifecycleObserver";

    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    public void onCreate() {
        Log.d(TAG, "onCreate");
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    public void onStart() {
        Log.d(TAG, "onStart");
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    public void onResume() {
        Log.d(TAG, "onResume");
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    public void onPause() {
        Log.d(TAG, "onPause");
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    public void onStop() {
        Log.d(TAG, "onStop");
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    public void onDestory() {
        Log.d(TAG, "onDestory");
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_ANY)
    public void onAny() {
        Log.d(TAG, "onAny");
    }
}
```

这里在MyLifecycleObserver中通过注解定义了生命周期的监听方法，从名字上我们大体可以知道对应的生命周期，这里先不关注，后面再详细介绍。

* 在Activity中注册LifecycleObserver

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    // 添加观察者
    getLifecycle().addObserver(new MyLifecycleObserver());
}
```

当Activity执行生命周期方法的同时，会通知LifecycleObserver，并回调对应的生命周期监听方法，这里是打印日志。

![](https://ws3.sinaimg.cn/large/005BYqpggy1g3c7iqz7d3j308q03xwed.jpg)

这样我们就实现了对Activity生命周期的监听。需要注意，只有项目中使用的support库版本在26.1.0及以上才可以直接使用`getLifecycle()`方法。

## 3.原理分析

下面介绍一下Lifecycle实现生命周期监听的原理，首先介绍几个关键类/接口。

* **Lifecycle**

Lifecycle是一个抽象类，是实现原理的核心类，内部定义了三个方法。

`void addObserver(LifecycleObserver observer)`：注册LifecycleObserver 

`void removeObserver(LifecycleObserver observer)`：移除LifecycleObserver

`State getCurrentState()` ：获取当前生命周期状态

此外，Lifecycle中使用枚举定义了一系列生命周期事件和状态。

**Lifecycle.Event**

```java
public enum Event {
    ON_CREATE,
    ON_START,
    ON_RESUME,
    ON_PAUSE,
    ON_STOP,
    ON_DESTROY,
    ON_ANY // 可以匹配上面的所有事件
}
```

**Lifecycle.State**

```java
public enum State {
    DESTROYED,
    INITIALIZED,
    CREATED,
    STARTED,
    RESUMED;
    public boolean isAtLeast(@NonNull State state) {
        return compareTo(state) >= 0;
    }
}
```

Event和State的关系如下图所示：

![](https://developer.android.google.cn/images/topic/libraries/architecture/lifecycle-states.png)

* **LifecycleOwner**

```java
public interface LifecycleOwner {
    /**
     * Returns the Lifecycle of the provider.
     *
     * @return The lifecycle of the provider.
     */
    @NonNull
    Lifecycle getLifecycle();
}
```

LifecycleOwner是一个接口，只定义了一个方法`Lifecycle getLifecycle()`，从名称上可以得知该接口表示生命周期持有者，持有一个Lifecycle对象，**26.1.0版本及以上support库中的AppCompatActivity**和**Fragment**都直接或间接实现了该接口，可以通过`getLifecycle()`方法获得Lifecycle对象。

* **LifecycleObserver**

```java
public interface LifecycleObserver {

}
```

LifecycleObserver也是一个接口，表示生命周期观察者，它的内部没有定义方法，从前面的例子中我们也看到了监听生命周期是通过注解方法来实现的。

了解了几个关键类/接口后我们就可以来具体分析Lifecycle的原理了，这里就以Activity生命周期的监听为例，Fragment也是一样的。AppCompatActivity的继承关系如下：

AppCompatActivity→FragmentActivity→SupportActivity（实现了LifecycleOwner接口）

**SupportActivity**直接实现了**LifecycleOwner**接口，因此它应该就是我们分析的切入点，我们直接来看这个类。该类实现了`getLifecycle()`方法，返回了一个**LifecycleRegistry**类型的对象，那么**LifecycleRegistry**类是什么呢？它继承了**Lifecycle**类，可以说就是它在负责生命周期事件的分发和处理。

```java
private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

public Lifecycle getLifecycle() {
    return this.mLifecycleRegistry;
}
```

我们接着来看SupportActivity，可以发现在`onCreate()`方法中调用了`ReportFragment.injectIfNeededIn(this)`，这行代码的作用是什么呢，我们点进去**ReportFragment**类来看一下。

```java
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    ReportFragment.injectIfNeededIn(this);
}
```

**ReportFragment**

```java
public class ReportFragment extends Fragment {
    private static final String REPORT_FRAGMENT_TAG = "android.arch.lifecycle"
            + ".LifecycleDispatcher.report_fragment_tag";

    public static void injectIfNeededIn(Activity activity) {
        // ProcessLifecycleOwner should always correctly work and some activities may not extend
        // FragmentActivity from support lib, so we use framework fragments for activities
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            // Hopefully, we are the first to make a transaction.
            manager.executePendingTransactions();
        }
    }

    static ReportFragment get(Activity activity) {
        return (ReportFragment) activity.getFragmentManager().findFragmentByTag(
                REPORT_FRAGMENT_TAG);
    }

    private ActivityInitializationListener mProcessListener;

    private void dispatchCreate(ActivityInitializationListener listener) {
        if (listener != null) {
            listener.onCreate();
        }
    }

    private void dispatchStart(ActivityInitializationListener listener) {
        if (listener != null) {
            listener.onStart();
        }
    }

    private void dispatchResume(ActivityInitializationListener listener) {
        if (listener != null) {
            listener.onResume();
        }
    }

    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        dispatchCreate(mProcessListener);
        dispatch(Lifecycle.Event.ON_CREATE);
    }

    @Override
    public void onStart() {
        super.onStart();
        dispatchStart(mProcessListener);
        dispatch(Lifecycle.Event.ON_START);
    }

    @Override
    public void onResume() {
        super.onResume();
        dispatchResume(mProcessListener);
        dispatch(Lifecycle.Event.ON_RESUME);
    }

    @Override
    public void onPause() {
        super.onPause();
        dispatch(Lifecycle.Event.ON_PAUSE);
    }

    @Override
    public void onStop() {
        super.onStop();
        dispatch(Lifecycle.Event.ON_STOP);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        dispatch(Lifecycle.Event.ON_DESTROY);
        // just want to be sure that we won't leak reference to an activity
        mProcessListener = null;
    }

    private void dispatch(Lifecycle.Event event) {
        Activity activity = getActivity();
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }

    void setProcessListener(ActivityInitializationListener processListener) {
        mProcessListener = processListener;
    }

    interface ActivityInitializationListener {
        void onCreate();

        void onStart();

        void onResume();
    }
}
```

可以看到`injectIfNeededIn()`方法的作用就是将ReportFragment添加到SupportActivity中。在ReportFragment的生命周期方法中分别调用了`void dispatch(Lifecycle.Event event)`方法，将相应的生命周期事件分发给LifecycleOwner持有的Lifecycle对象进行处理，这里调用`getLifecycle()`获得的对象就是SupportActivity中的mLifecycleRegistry。

分析到这里我们大概对Lifecycle有了一个初步的认知，即通过**给Activity注入一个Fragment，继而通过该Fragment来进行生命周期事件的分发**。其实通过添加Fragment来简化Activity中的回调是一个很常见的方法，很多优秀的第三方框架都有用到，像RxPermission等，给我们提供了一个很好的思路。

接下来我们继续分析，上线分析到了ReportFragment的`dispatch()`方法，内部通过`getLifecycle().handleLifecycleEvent(event)`获取到SupportActivity中定义的LifecycleRegistry对象，来进行生命周期事件的处理，既然如此，我们就来看LifecycleRegistry的`handleLifecycleEvent(Lifecycle.Event event)`方法。

```java
/**
 * Sets the current state and notifies the observers.
 * <p>
 * Note that if the {@code currentState} is the same state as the last call to this method,
 * calling this method has no effect.
 *
 * @param event The event that was received
 */
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    State next = getStateAfter(event);
    moveToState(next);
}

static State getStateAfter(Event event) {
    switch (event) {
        case ON_CREATE:
        case ON_STOP:
            return CREATED;
        case ON_START:
        case ON_PAUSE:
            return STARTED;
        case ON_RESUME:
            return RESUMED;
        case ON_DESTROY:
            return DESTROYED;
        case ON_ANY:
            break;
    }
    throw new IllegalArgumentException("Unexpected event value " + event);
}

private void moveToState(State next) {
    if (mState == next) {
        return;
    }
    mState = next;
    if (mHandlingEvent || mAddingObserverCounter != 0) {
        mNewEventOccurred = true;
        // we will figure out what to do on upper level.
        return;
    }
    mHandlingEvent = true;
    sync();
    mHandlingEvent = false;
}
```

可以看到，`handleLifecycleEvent()`方法首先调用了`getStateAfter(event)`方法，该方法的作用就是根据当前的Event获得相应的State，可以对照上面的Event和State关系图来看。获取到State之后调动了` moveToState(next)`方法，该方法设置了Lifecycle的当前State，并调用了`sync()`方法，我们接下来看一下该方法。

```java
private void sync() {
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        Log.w(LOG_TAG, "LifecycleOwner is garbage collected, you shouldn't try dispatch "
                + "new events from it.");
        return;
    }
    while (!isSynced()) {
        mNewEventOccurred = false;
        // no need to check eldest for nullability, because isSynced does it for us.
        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
            backwardPass(lifecycleOwner);
        }
        Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
        if (!mNewEventOccurred && newest != null
                && mState.compareTo(newest.getValue().mState) > 0) {
            forwardPass(lifecycleOwner);
        }
    }
    mNewEventOccurred = false;
}
```

`sync()`方法的作用是比较当前State和上一个State，判断出是执行`backwardPass()`还是`forwardPass()` ，即生命周期是“后退”还是“前进”，这两个方法是类似的，我们只看一个就好了，以`backwardPass()`方法为例。

```java
private void backwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
            mObserverMap.descendingIterator();
    while (descendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
        ObserverWithState observer = entry.getValue();
        while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            Event event = downEvent(observer.mState);
            pushParentState(getStateAfter(event));
            observer.dispatchEvent(lifecycleOwner, event);
            popParentState();
        }
    }
}

private static Event downEvent(State state) {
    switch (state) {
        case INITIALIZED:
            throw new IllegalArgumentException();
        case CREATED:
            return ON_DESTROY;
        case STARTED:
            return ON_STOP;
        case RESUMED:
            return ON_PAUSE;
        case DESTROYED:
            throw new IllegalArgumentException();
    }
    throw new IllegalArgumentException("Unexpected state value " + state);
}
```

这里首先调用了`downEvent()`方法获得要分发的事件，然后调用`observer.dispatchEvent(lifecycleOwner, event)`方法完成事件的分发。

```java
static class ObserverWithState {
    State mState;
    GenericLifecycleObserver mLifecycleObserver;

    ObserverWithState(LifecycleObserver observer, State initialState) {
        mLifecycleObserver = Lifecycling.getCallback(observer);
        mState = initialState;
    }

    void dispatchEvent(LifecycleOwner owner, Event event) {
        State newState = getStateAfter(event);
        mState = min(mState, newState);
      	// 具体实现在ReflectiveGenericLifecycleObserver类中
        mLifecycleObserver.onStateChanged(owner, event);
        mState = newState;
    }
}
```

`dispatchEvent()`方法中会执行`mLifecycleObserver.onStateChanged(owner, event)`方法，方法内部最终会根据反射获取到LifecycleObserver中被`@OnLifecycleEvent `注解修饰的方法，再通过反射进行调用，这样就完成了整个生命周期监听回调的流程。

总结一下，Lifecycle的工作流程如下：

* 第一步、Activity（其实是SupportActivity）实现LifecycleOwner接口，添加ReportFragment
* 第二步、ReportFragment中生命周期回调方法中调用`dispatch()`方法将事件分发给LifecycleRegistry，即SupportActivity持有的LifecycleOwner
* 第三步、LifecycleRegistry确定当前的State和Event，最终通过反射调用在LifecycleObserver中定义的注解方法

简单来说就是这样，真实情况要复杂一些，需要的话可以更深入地查看源码。

## 4.应用场景

首先补充一下，由于AppCompatActivity间接地实现了LifecycleOwner接口，因此当我们的Activity继承自AppCompatActivity时自动就会成为一个LifecycleOwner。如果要自己实现一个LifecycleOwner的话需要实现LifecycleOwner接口和`getLifecycle()`方法，方法返回一个Lifecycle对象，官方的建议是直接使用LifecycleRegistry类，因为它本身就是一个成熟的Lifecycle实现类。官方文档给出的实例代码如下：

```
public class MyActivity extends Activity implements LifecycleOwner {
    private LifecycleRegistry lifecycleRegistry;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        lifecycleRegistry = new LifecycleRegistry(this);
        lifecycleRegistry.markState(Lifecycle.State.CREATED);
    }

    @Override
    public void onStart() {
        super.onStart();
        lifecycleRegistry.markState(Lifecycle.State.STARTED);
    }

    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return lifecycleRegistry;
    }
}
```

其次，官方提供了一个**DefaultLifecycleObserver**类，该类是在androidx库中的，方便我们直接实现LifecycleObserver接口，不需要通过注解来定义生命周期回调，官方更推荐我们使用该类。因为一旦Java 8成为Android的主流，注释将被弃用。
# Android架构组件-ViewModel

## 目录

- [1.简介](#1简介)
- [2.简单使用](#2简单使用)
- [3.原理分析](#3原理分析)
- [4.应用场景](#4应用场景)
- [5.参考文章](#5参考文章)

## 1.简介

[VIewModel](https://developer.android.google.cn/topic/libraries/architecture/viewmodel)是Android官方推出的Android Jetpack中Architecture组件的一员，用来管理UI相关数据，能够感知生命周期，并且可以在页面配置改变（如旋转屏幕）的情况下保证数据依然存在。我们知道在Activity的配置发生改变时，例如旋转屏幕，会导致Activity重新创建，这时Activity持有的数据会丢失，而ViewModel则不会被销毁，因此可以保证持有的数据不被销毁，相比于`onSaveInstanceState()`方法只能保存可序列化的少量数据，ViewModel则更加灵活。

## 2.简单使用

首先添加依赖

```groovy
implementation "android.arch.lifecycle:extensions:1.1.1"
```

自定义ViewModel，配合LiveData使用。

```java
public class MyViewModel extends ViewModel {

    private static final String TAG = "MyViewModel";

    private MutableLiveData<String> name;

    public LiveData<String> getName() {
        if (name == null) {
            name = new MutableLiveData<String>();
        }
        return name;
    }

    @Override
    protected void onCleared() {
        super.onCleared();
        Log.d(TAG, "onCleared");
    }
}
```

在ViewModel被销毁时会回调`onCleared()`方法，可以在这里进行资源的回收释放。

使用时通过`ViewModelProviders.of() `方法获得ViewModel实例对象，再通过LiveData监听数据的改变。

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    MyViewModel viewModel = ViewModelProviders.of(this).get(MyViewModel.class);
    viewModel.getName().observe(this, new Observer<String>() {
        @Override
        public void onChanged(@Nullable String s) {
            // 数据改变，可更新UI
        }
    });
}
```

我们可以尝试旋转屏幕，Activity会重新创建，但是重建前后拿到的ViewModel实例对象是同一个。

![](https://ws3.sinaimg.cn/large/005BYqpggy1g43wt4t1l7j30gb041gli.jpg)

## 3.原理分析

我们首先来看一下官网提供的ViewModel生命周期，可以看出旋转屏幕时ViewModel仍然可以存活，只有当Activity调用`finish()`方法时ViewModel才会被销毁并回调`onCleared()`方法。

![ViewModel的生命周期](https://developer.android.google.cn/images/topic/libraries/architecture/viewmodel-lifecycle.png)

下面我们就来具体分析一下ViewModel的实现原理，看看为什么ViewModel可以在Activity重建时依然保持存活。

首先从ViewModel的创建入手，也就是`ViewModelProviders.of(this).get(MyViewModel.class)`这行代码，先来看`ViewModelProviders.of()`方法。

```java
@NonNull
@MainThread
public static ViewModelProvider of(@NonNull FragmentActivity activity) {
    return of(activity, null);
}

@NonNull
@MainThread
public static ViewModelProvider of(@NonNull FragmentActivity activity,
                                   @Nullable Factory factory) {
    Application application = checkApplication(activity);
    if (factory == null) {
        factory = ViewModelProvider.AndroidViewModelFactory.getInstance(application);
    }
    return new ViewModelProvider(ViewModelStores.of(activity), factory);
}
```

`ViewModelProviders.of()`有两个重载方法，区别在于是否传入factory，类型是Factory，从名称上也能看出是用来创建ViewModel的工厂类，Factory是一个接口，内部声明了`create()`方法用来创建ViewModel实例。如果没有指定factory则默认会使用**AndroidViewModelFactory**，下面就来看一下AndroidViewModelFactory：

```java
public static class AndroidViewModelFactory extends ViewModelProvider.NewInstanceFactory {

    private static AndroidViewModelFactory sInstance;

    /**
     * 获取单例对象
     */
    @NonNull
    public static AndroidViewModelFactory getInstance(@NonNull Application application) {
        if (sInstance == null) {
            sInstance = new AndroidViewModelFactory(application);
        }
        return sInstance;
    }

    private Application mApplication;

    /**
     * Creates a {@code AndroidViewModelFactory}
     *
     * @param application an application to pass in {@link AndroidViewModel}
     */
    public AndroidViewModelFactory(@NonNull Application application) {
        mApplication = application;
    }

    /**
     * 通过反射创建ViewModel实例
     */
    @NonNull
    @Override
    public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
        if (AndroidViewModel.class.isAssignableFrom(modelClass)) {
            //noinspection TryWithIdenticalCatches
            try {
                return modelClass.getConstructor(Application.class).newInstance(mApplication);
            } catch (NoSuchMethodException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            } catch (IllegalAccessException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            } catch (InstantiationException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            } catch (InvocationTargetException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            }
        }
        return super.create(modelClass);
    }
}
```

AndroidViewModelFactory是**ViewModelProvider**的内部类，在`create()`方法内部通过反射创建了ViewModel实例。

重新回到`ViewModelProviders.of()`方法，方法内部创建了一个ViewModelProvider对象并返回，可以看到构造ViewModelProvider对象时传入了两个参数，第二个参数就是上面分析的Factory对象，第一个参数是一个**ViewModelStore**对象，这里是通过`ViewModelStores.of(activity)`获取到的，我们先不去管ViewModelStore是如何创建的，后面会具体分析。创建好ViewModelProvider对象后调用了`get()`方法获得ViewModel实例对象，参数传入了要创建的ViewModel的class。

```java
@NonNull
@MainThread
public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
    String canonicalName = modelClass.getCanonicalName();
    if (canonicalName == null) {
        throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
    }
    return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
}

@NonNull
@MainThread
public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    ViewModel viewModel = mViewModelStore.get(key);

    if (modelClass.isInstance(viewModel)) {
        //noinspection unchecked
        return (T) viewModel;
    } else {
        //noinspection StatementWithEmptyBody
        if (viewModel != null) {
            // TODO: log a warning.
        }
    }

    viewModel = mFactory.create(modelClass);
    mViewModelStore.put(key, viewModel);
    //noinspection unchecked
    return (T) viewModel;
}
```

可以看出`get()`方法内部会根据传入的modelClass得到一个key值，再根据这个key值从mViewModelStore中获取对应的ViewModel，这里的mViewModelStore就是在创建ViewModelProvider时通过第一个参数赋值的，如果根据key值获取到的ViewModel为null，则通过Factory来创建ViewModel实例对象，并把创建好的ViewModel对象`put`到mViewModelStore中。看到这里我们就能大概清楚ViewModelStore的作用了，就是用来保存ViewModel实例对象的。ViewModelStore内部维护了一个HashMap，`get()`和`put()`方法实际上就是操作这个HashMap。

通过上文的分析我们基本上了解了ViewModel的创建过程，不过到这里并没有涉及到任何生命周期相关的逻辑，那么ViewModel是如何感知生命周期，保证在Activity重新创建时得以存活的呢。大家可能还记得，上面的分析还遗漏了一点，也就是ViewModelStore的创建，我们索性就从`ViewModelStores.of()`方法入手。

```java
@NonNull
@MainThread
public static ViewModelStore of(@NonNull FragmentActivity activity) {
    if (activity instanceof ViewModelStoreOwner) {
        return ((ViewModelStoreOwner) activity).getViewModelStore();
    }
    return holderFragmentFor(activity).getViewModelStore();
}
```

这里首先判断了Activity是否为**ViewModelStoreOwner**，如果是则直接使用Activity的`getViewModelStore()`方法获取ViewModelStore对象，如果Activity不是ViewModelStoreOwner则调用`holderFragmentFor(activity).getViewModelStore()`获取ViewModelStore对象。ViewModelStoreOwner是一个接口，从名称上我们可以看出这是一个ViewModelStore持有者（类似于LifecycleOwner），接口内部定义了一个`getViewModelStore()`方法用于获取ViewModelStore对象。下面就针对这两种情况分别来看一下。

* **情况一、Activity instanceof ViewModelStoreOwner**

通过查看源码我们发现**FragmentActivity**实现了ViewModelStoreOwner接口，由于当前的Activity继承自AppCompatActivity，而AppCompatActivity又继承自FragmentActivity，因此这里的判断为真，会执行Activity的`getViewModelStore()`方法，也就是FragmentActivity中声明的`getViewModelStore()`方法，我们来看一下。

```java
@NonNull
public ViewModelStore getViewModelStore() {
    if (this.getApplication() == null) {
        throw new IllegalStateException("Your activity is not yet attached to the Application instance. You can't request ViewModel before onCreate call.");
    } else {
        if (this.mViewModelStore == null) {
            FragmentActivity.NonConfigurationInstances nc = (FragmentActivity.NonConfigurationInstances)this.getLastNonConfigurationInstance();
            if (nc != null) {
                this.mViewModelStore = nc.viewModelStore;
            }

            if (this.mViewModelStore == null) {
                this.mViewModelStore = new ViewModelStore();
            }
        }

        return this.mViewModelStore;
    }
}

static final class NonConfigurationInstances {
    Object custom;
    ViewModelStore viewModelStore;
    FragmentManagerNonConfig fragments;

    NonConfigurationInstances() {
    }
}
```

可以看到首先通过`getLastNonConfigurationInstance()`方法获取到了一个NonConfigurationInstances对象nc，NonConfigurationInstances类是FragmentActivity的一个静态内部类，类的内部定义了一个ViewModelStore对象，换句话说就是NonConfigurationInstances持有ViewModelStore对象。如果获取到的nc不为空，就直接获取nc内部声明的这个ViewModelStore对象，如果不为空就赋值给mViewModelStore，它是FragmentActivity的一个成员变量，否则创建一个ViewModelStore对象赋值给mViewModelStore，最后返回mViewModelStore。

那么这个`getLastNonConfigurationInstance()`方法有什么特别之处呢，它与Activity的另一个方法`onRetainNonConfigurationInstance()`是配合使用的，用于在Activity配置发生改变（比如旋转屏幕）时保存数据，在Activity重新创建之后恢复数据，类似于`onSaveInstanceState()`和`onRestoreInstanceState()`，不同之处就是这个方法可以返回一个包含有状态信息的Object，其中甚至可以包含Activity Instance本身。因此这里通过`getLastNonConfigurationInstance()`方法就可以获取到Activity重建时通过`onRetainNonConfigurationInstance()`方法保存的数据，既然已经清楚了原理，我们接下来就看一看`onRetainNonConfigurationInstance()`方法内部是如何保存数据的吧。

```java
public final Object onRetainNonConfigurationInstance() {
    Object custom = this.onRetainCustomNonConfigurationInstance();
    FragmentManagerNonConfig fragments = this.mFragments.retainNestedNonConfig();
    if (fragments == null && this.mViewModelStore == null && custom == null) {
        return null;
    } else {
        FragmentActivity.NonConfigurationInstances nci = new FragmentActivity.NonConfigurationInstances();
        nci.custom = custom;
        nci.viewModelStore = this.mViewModelStore;
        nci.fragments = fragments;
        return nci;
    }
}
```

可以看到如果mViewModelStore不为空，就会创建一个NonConfigurationInstances对象，将mViewModelStore保存到对象中并返回该对象，这样通过`getLastNonConfigurationInstance()`方法就可以获取到保存到mViewModelStore。

看到这里我们应该就清楚了情况一下ViewModel可以在Activity重建时存活的原因，就是在Activity配置发生改变时保存了ViewModelStore对象，进而保证了ViewModelStore内部保存的ViewModel在Activity重建前后得到的是同一个对象。

最后我们来看一下FragmentActivity的`onDestroy()`方法：

```java
protected void onDestroy() {
    super.onDestroy();
    if (this.mViewModelStore != null && !this.isChangingConfigurations()) {
        this.mViewModelStore.clear();
    }

    this.mFragments.dispatchDestroy();
}
```

可以看出，当Activity不是因为配置改变而销毁时，会执行mViewModelStore的`clear()`方法，在方法内部遍历所有的ViewModel，依次执行`onCleared()`方法，完成ViewModel的销毁。

* **情况二、Activity not instanceof ViewModelStoreOwner**

我们已经知道了情况一是通过Activity自身的数据保存机制来实现ViewModel存活的，那么情况二是如何实现的呢？我们需要从`holderFragmentFor(activity)`方法来切入，方法声明在**HoldFragment**类中。

```java
private static final HolderFragmentManager sHolderFragmentManager = new HolderFragmentManager();

public static HolderFragment holderFragmentFor(FragmentActivity activity) {
    return sHolderFragmentManager.holderFragmentFor(activity);
}

// 管理HolderFragment
static class HolderFragmentManager {
    private Map<Activity, HolderFragment> mNotCommittedActivityHolders = new HashMap<>();
    private Map<Fragment, HolderFragment> mNotCommittedFragmentHolders = new HashMap<>();

    private ActivityLifecycleCallbacks mActivityCallbacks =
            new EmptyActivityLifecycleCallbacks() {
                @Override
                public void onActivityDestroyed(Activity activity) {
                    HolderFragment fragment = mNotCommittedActivityHolders.remove(activity);
                    if (fragment != null) {
                        Log.e(LOG_TAG, "Failed to save a ViewModel for " + activity);
                    }
                }
            };

    private boolean mActivityCallbacksIsAdded = false;

    private FragmentLifecycleCallbacks mParentDestroyedCallback =
            new FragmentLifecycleCallbacks() {
                @Override
                public void onFragmentDestroyed(FragmentManager fm, Fragment parentFragment) {
                    super.onFragmentDestroyed(fm, parentFragment);
                    HolderFragment fragment = mNotCommittedFragmentHolders.remove(
                            parentFragment);
                    if (fragment != null) {
                        Log.e(LOG_TAG, "Failed to save a ViewModel for " + parentFragment);
                    }
                }
            };

    void holderFragmentCreated(Fragment holderFragment) {
        Fragment parentFragment = holderFragment.getParentFragment();
        if (parentFragment != null) {
            mNotCommittedFragmentHolders.remove(parentFragment);
            parentFragment.getFragmentManager().unregisterFragmentLifecycleCallbacks(
                    mParentDestroyedCallback);
        } else {
            mNotCommittedActivityHolders.remove(holderFragment.getActivity());
        }
    }

    private static HolderFragment findHolderFragment(FragmentManager manager) {
        if (manager.isDestroyed()) {
            throw new IllegalStateException("Can't access ViewModels from onDestroy");
        }

        Fragment fragmentByTag = manager.findFragmentByTag(HOLDER_TAG);
        if (fragmentByTag != null && !(fragmentByTag instanceof HolderFragment)) {
            throw new IllegalStateException("Unexpected "
                    + "fragment instance was returned by HOLDER_TAG");
        }
        return (HolderFragment) fragmentByTag;
    }

    private static HolderFragment createHolderFragment(FragmentManager fragmentManager) {
        HolderFragment holder = new HolderFragment();
        fragmentManager.beginTransaction().add(holder, HOLDER_TAG).commitAllowingStateLoss();
        return holder;
    }

    HolderFragment holderFragmentFor(FragmentActivity activity) {
      	// 获得FragmentManager
        FragmentManager fm = activity.getSupportFragmentManager();
      	// 根据Tag查找Fragment
        HolderFragment holder = findHolderFragment(fm);
        if (holder != null) {
            return holder;
        }
      	// 从Map中查找Fragment
        holder = mNotCommittedActivityHolders.get(activity);
        if (holder != null) {
            return holder;
        }
		
        if (!mActivityCallbacksIsAdded) {
            mActivityCallbacksIsAdded = true;
            // 注册生命周期监听，在Activity销毁时将Fragment从Map中移除
            activity.getApplication().registerActivityLifecycleCallbacks(mActivityCallbacks);
        }
      	// 如果之前没有添加过就创建一个HolderFragment并添加到Activity中
        holder = createHolderFragment(fm);
        mNotCommittedActivityHolders.put(activity, holder);
        return holder;
    }

    HolderFragment holderFragmentFor(Fragment parentFragment) {
        FragmentManager fm = parentFragment.getChildFragmentManager();
        HolderFragment holder = findHolderFragment(fm);
        if (holder != null) {
            return holder;
        }
        holder = mNotCommittedFragmentHolders.get(parentFragment);
        if (holder != null) {
            return holder;
        }

        parentFragment.getFragmentManager()
                .registerFragmentLifecycleCallbacks(mParentDestroyedCallback, false);
        holder = createHolderFragment(fm);
        mNotCommittedFragmentHolders.put(parentFragment, holder);
        return holder;
    }
}
```

通过跟踪源码我们可以发现，`holderFragmentFor(activity)`方法最终会执行到HolderFragmentManager的`holderFragmentFor(activity)`方法，方法内部会将一个HoldFragment添加到当前的Activity中。我们接下来看看这个HoldFragment有什么特殊之处。

```java
public class HolderFragment extends Fragment implements ViewModelStoreOwner {
    private static final String LOG_TAG = "ViewModelStores";

    private static final HolderFragmentManager sHolderFragmentManager = new HolderFragmentManager();

    private ViewModelStore mViewModelStore = new ViewModelStore();

    public HolderFragment() {
        setRetainInstance(true);
    }

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        sHolderFragmentManager.holderFragmentCreated(this);
    }

    @Override
    public void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        mViewModelStore.clear();
    }

    @NonNull
    @Override
    public ViewModelStore getViewModelStore() {
        return mViewModelStore;
    }
}
```

可以看出HolderFragment实现了ViewModelStoreOwner接口，内部定义了一个ViewModelStore类型的成员变量mViewModelStore，通过`getViewModelStore()`方法可以获取到这个ViewModelStore对象。我们发现在HolderFragment的构造方法中有一行代码`setRetainInstance(true)`，它的作用就是保证Fragment在Activity配置改变时不被销毁，因此就保证了mViewModelStore的存活，进而使得ViewModelStore保存的ViewModel不被销毁（即`onDestory()`方法不被调用）。最后在HoldFragment的`onDestory()`方法中调用ViewModelStore的`clear()`方法，方法内部遍历所有的ViewModel，依次执行`onCleared()`方法。

分析到这里我不由得感叹这种设计的巧妙，又是Fragment，我们可能还记得Lifecycle内部也是利用了Fragment，此外还有很多优秀的框架都是利用添加一个Fragment来实现的，就不一一列举了，这确实为我们今后的开发拓宽了思路。

## 4.应用场景

需要注意，ViewModel不能持有视图、Lifecycle或者Activity的上下文，否则会导致内存泄漏问题。

由于ViewModel是保存在ViewModelStore中的，而每个Activity中只有一个ViewModelStore对象，因此利用ViewModel可以实现数据的共享，比如一个Activity中的多个Fragment可以共享一个ViewModel实例。

## 5.参考文章

[【AAC 系列四】深入理解架构组件：ViewModel](https://mp.weixin.qq.com/s/ZYhLuYNjyXGlYQiMLbQQIA)

[ViewModel官方文档](https://developer.android.google.cn/topic/libraries/architecture/viewmodel)
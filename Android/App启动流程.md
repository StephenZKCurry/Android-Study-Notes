# App启动流程

> 前言
>
> 上一篇文章中我分析了Activity的启动流程，是建立在App进程已经创建完成（即App已经打开）的情况下的，那么App从点击图标到启动Activity这个过程是怎么样的呢，本文就来简单分析一下。

## 源码分析

首先还是先强调一下，**本文的分析基于Android 9.0（API Level 28）的源码**。

Android系统的桌面其实就是一个App，这个特殊的App叫做**Launcher**，很多手机生产厂商都有自己定制的Launcher，因此不同的手机桌面样式会有些许的不同。目前Android原生的Launcher版本是Launcher3，而我们桌面上所展示的Activity的就是Launcher3中的`Launcher.java`，在这个Activity中展示着手机上安装的应用程序快捷方式，它的定义如下：

**Launcher.java**

```java
public class Launcher extends BaseDraggingActivity implements LauncherExterns,
        LauncherModel.Callbacks, LauncherProviderChangeListener, UserEventDelegate {
    // ...
}
```

Launcher中声明了一个`createShortcut()`方法，从方法名也能猜到是用来创建应用快捷方式（图标）的。

```java
public View createShortcut(ViewGroup parent, ShortcutInfo info) {
    BubbleTextView favorite = (BubbleTextView) LayoutInflater.from(parent.getContext())
            .inflate(R.layout.app_icon, parent, false);
    favorite.applyFromShortcutInfo(info);
    favorite.setOnClickListener(ItemClickHandler.INSTANCE);
    favorite.setOnFocusChangeListener(mFocusHandler);
    return favorite;
}
```

由于我们要研究App的启动流程，因此首先要从点击图标的方法来入手，可以看出点击事件定义在了**ItemClickHandler**类中，我们来看一下：

**ItemClickHandler.java**

```java
public class ItemClickHandler {

    /**
     * Instance used for click handling on items
     */
    public static final OnClickListener INSTANCE = ItemClickHandler::onClick;

    private static void onClick(View v) {
        // ...
      	// 应用的启动信息包含在了tag中，包括要启动哪个Activity等等
        Object tag = v.getTag();
        if (tag instanceof ShortcutInfo) {
          	// 核心代码
          	// 点击应用快捷方式调用该方法
            onClickAppShortcut(v, (ShortcutInfo) tag, launcher);
        } else if (tag instanceof FolderInfo) {
            if (v instanceof FolderIcon) {
                onClickFolderIcon(v);
            }
        } else if (tag instanceof AppInfo) {
            startAppShortcutOrInfoActivity(v, (AppInfo) tag, launcher);
        } else if (tag instanceof LauncherAppWidgetInfo) {
            if (v instanceof PendingAppWidgetHostView) {
                onClickPendingWidget((PendingAppWidgetHostView) v, launcher);
            }
        }
    }

    private static void onClickAppShortcut(View v, ShortcutInfo shortcut, Launcher launcher) {
        // ...
        // Start activities
        startAppShortcutOrInfoActivity(v, shortcut, launcher);
    }

    private static void startAppShortcutOrInfoActivity(View v, ItemInfo item, Launcher launcher) {
        Intent intent;
        if (item instanceof PromiseAppInfo) {
            PromiseAppInfo promiseAppInfo = (PromiseAppInfo) item;
            intent = promiseAppInfo.getMarketIntent(launcher);
        } else {
            intent = item.getIntent();
        }
        if (intent == null) {
            throw new IllegalArgumentException("Input must have a valid intent");
        }
        if (item instanceof ShortcutInfo) {
            ShortcutInfo si = (ShortcutInfo) item;
            if (si.hasStatusFlag(ShortcutInfo.FLAG_SUPPORTS_WEB_UI)
                    && intent.getAction() == Intent.ACTION_VIEW) {
                // make a copy of the intent that has the package set to null
                // we do this because the platform sometimes disables instant
                // apps temporarily (triggered by the user) and fallbacks to the
                // web ui. This only works though if the package isn't set
                intent = new Intent(intent);
                intent.setPackage(null);
            }
        }
      	// 核心代码
        launcher.startActivitySafely(v, intent, item);
    }
}
```

ItemClickHandler的`onClick()`方法判断了如果点击的是应用快捷方式就调用`onClickAppShortcut()`方法，接着又调用了`startAppShortcutOrInfoActivity()`方法，最后会调用Launcher的`startActivitySafely()`方法。

```java
public boolean startActivitySafely(View v, Intent intent, ItemInfo item) {
    boolean success = super.startActivitySafely(v, intent, item);
    if (success && v instanceof BubbleTextView) {
        // This is set to the view that launched the activity that navigated the user away
        // from launcher. Since there is no callback for when the activity has finished
        // launching, enable the press state and keep this reference to reset the press
        // state when we return to launcher.
        BubbleTextView btv = (BubbleTextView) v;
        btv.setStayPressed(true);
        setOnResumeCallback(btv);
    }
    return success;
}
```

该方法又调用了父类**BaseDraggingActivity**的`startActivitySafely()`方法：

```java
public boolean startActivitySafely(View v, Intent intent, ItemInfo item) {
    // ...
    // Prepare intent
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    // ...
    // Could be launching some bookkeeping activity
    startActivity(intent, optsBundle);
    // ...
    return false;
}
```

可以看到`startActivitySafely()`方法最终调用了`startActivity()`方法，并设置了**FLAG_ACTIVITY_NEW_TASK**这个Flag，不难理解，启动一个App肯定是要创建一个新的Activity任务栈的。

到这里就又回到了Activity的`startActivity()`方法，也就是Activity的启动流程，这一部分我就不从头开始分析了了，可以查看上一篇文章[Activity启动流程](https://github.com/StephenZKCurry/Android-Study-Notes/blob/master/Android/Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.md)，这里附上Activity启动的流程图。

![Activity启动流程](https://raw.githubusercontent.com/StephenZKCurry/Android-Study-Notes/master/Android/Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.jpg)

我们跳过一部分流程，直接来看**ActivityStackSupervisor**的`startSpecificActivityLocked()`方法：

```java
void startSpecificActivityLocked(ActivityRecord r,
                                 boolean andResume, boolean checkConfig) {
    // 获取应用进程
    ProcessRecord app = mService.getProcessRecordLocked(r.processName,
            r.info.applicationInfo.uid, true);

    getLaunchTimeTracker().setLaunchTime(r);
	
    if (app != null && app.thread != null) {
      	 // 进程已创建
        try {
            // ...
         	// 核心代码
            realStartActivityLocked(r, app, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                    + r.intent.getComponent().flattenToShortString(), e);
        }

        // If a dead object exception was thrown -- fall through to
        // restart the application.
    }
	
  	// 进程不存在则创建应用进程
    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
            "activity", r.intent.getComponent(), false, false, true);
}
```

在Activity的启动流程分析中，我直接默认为App进程已创建，这里就要分析另一种情况了，也就是应用进程未创建，这种情况会执行`mService.startProcessLocked()`方法，mService的类型是**ActivityManagerService（AMS）**，我们来看ActivityManagerService的`startProcessLocked()`方法：

```java
final ProcessRecord startProcessLocked(String processName,
                                       ApplicationInfo info, boolean knownToBeDead, int intentFlags,
                                       String hostingType, ComponentName hostingName, boolean allowWhileBooting,
                                       boolean isolated, boolean keepIfLarge) {
    return startProcessLocked(processName, info, knownToBeDead, intentFlags, hostingType,
            hostingName, allowWhileBooting, isolated, 0 /* isolatedUid */, keepIfLarge,
            null /* ABI override */, null /* entryPoint */, null /* entryPointArgs */,
            null /* crashHandler */);
}

final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
                                       boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName,
                                       boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge,
                                       String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {
    // ...
    final boolean success = startProcessLocked(app, hostingType, hostingNameStr, abiOverride);
    // ...
}

private final boolean startProcessLocked(ProcessRecord app,
                                         String hostingType, String hostingNameStr, String abiOverride) {
    return startProcessLocked(app, hostingType, hostingNameStr,
            false /* disableHiddenApiChecks */, abiOverride);
}

private final boolean startProcessLocked(ProcessRecord app, String hostingType,
                                         String hostingNameStr, boolean disableHiddenApiChecks, String abiOverride) {
    // ...
    return startProcessLocked(hostingType, hostingNameStr, entryPoint, app, uid, gids,
            runtimeFlags, mountExternal, seInfo, requiredAbi, instructionSet, invokeWith,
            startTime);
}

private boolean startProcessLocked(String hostingType, String hostingNameStr, String entryPoint,
                                   ProcessRecord app, int uid, int[] gids, int runtimeFlags, int mountExternal,
                                   String seInfo, String requiredAbi, String instructionSet, String invokeWith,
                                   long startTime) {
    // ...
    final ProcessStartResult startResult = startProcess(app.hostingType, entryPoint,
            app, app.startUid, gids, runtimeFlags, mountExternal, app.seInfo,
            requiredAbi, instructionSet, invokeWith, app.startTime);
    // ...
}

private ProcessStartResult startProcess(String hostingType, String entryPoint,
                                        ProcessRecord app, int uid, int[] gids, int runtimeFlags, int mountExternal,
                                        String seInfo, String requiredAbi, String instructionSet, String invokeWith,
                                        long startTime) {
    // ...
    startResult = Process.start(entryPoint,
            app.processName, uid, uid, gids, runtimeFlags, mountExternal,
            app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
            app.info.dataDir, invokeWith,
            new String[]{PROC_START_SEQ_IDENT + app.startSeq});
    // ...
}
```

经过一系列调用最终执行了`Process.start()`方法：

```java
public static final ProcessStartResult start(final String processClass,
                                             final String niceName,
                                             int uid, int gid, int[] gids,
                                             int runtimeFlags, int mountExternal,
                                             int targetSdkVersion,
                                             String seInfo,
                                             String abi,
                                             String instructionSet,
                                             String appDataDir,
                                             String invokeWith,
                                             String[] zygoteArgs) {
    return zygoteProcess.start(processClass, niceName, uid, gid, gids,
            runtimeFlags, mountExternal, targetSdkVersion, seInfo,
            abi, instructionSet, appDataDir, invokeWith, zygoteArgs);
}
```

接着又调用了**ZygoteProcess**的`start()`方法：

```java
public final Process.ProcessStartResult start(final String processClass,
                                              final String niceName,
                                              int uid, int gid, int[] gids,
                                              int runtimeFlags, int mountExternal,
                                              int targetSdkVersion,
                                              String seInfo,
                                              String abi,
                                              String instructionSet,
                                              String appDataDir,
                                              String invokeWith,
                                              String[] zygoteArgs) {
    try {
        return startViaZygote(processClass, niceName, uid, gid, gids,
                runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                abi, instructionSet, appDataDir, invokeWith, false /* startChildZygote */,
                zygoteArgs);
    } catch (ZygoteStartFailedEx ex) {
        Log.e(LOG_TAG,
                "Starting VM process through Zygote failed");
        throw new RuntimeException(
                "Starting VM process through Zygote failed", ex);
    }
}

private Process.ProcessStartResult startViaZygote(final String processClass,
                                                  final String niceName,
                                                  final int uid, final int gid,
                                                  final int[] gids,
                                                  int runtimeFlags, int mountExternal,
                                                  int targetSdkVersion,
                                                  String seInfo,
                                                  String abi,
                                                  String instructionSet,
                                                  String appDataDir,
                                                  String invokeWith,
                                                  boolean startChildZygote,
                                                  String[] extraArgs)
        throws ZygoteStartFailedEx {
    // ...
    return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
}

private static Process.ProcessStartResult zygoteSendArgsAndGetResult(
        ZygoteState zygoteState, ArrayList<String> args)
        throws ZygoteStartFailedEx {
    // ...
    final BufferedWriter writer = zygoteState.writer;
    final DataInputStream inputStream = zygoteState.inputStream;

    writer.write(Integer.toString(args.size()));
    writer.newLine();

    for (int i = 0; i < sz; i++) {
        String arg = args.get(i);
        writer.write(arg);
        writer.newLine();
    }

    writer.flush();

    // Should there be a timeout on this?
    Process.ProcessStartResult result = new Process.ProcessStartResult();

    // Always read the entire result from the input stream to avoid leaving
    // bytes in the stream for future process starts to accidentally stumble
    // upon.
    result.pid = inputStream.readInt();
    result.usingWrapper = inputStream.readBoolean();

    if (result.pid < 0) {
        throw new ZygoteStartFailedEx("fork() failed");
    }
    return result;
}
```

这里又是经过了一系列的调用，最终执行了`zygoteSendArgsAndGetResult()`方法，该方法的作用就是创建App进程。关于App进程的创建，我粗略地阅读了一下网上的相关文章，简单介绍一下，Android中有一个重要的进程**Zygote**，翻译为受精卵进程，**所有的应用程序进程都是通过Zygote进程fork得来的**。流程简单来说就是通过Binder请求AMS进程，然后AMS再发送Socket消息给Zygote进程，最后统一由Zygote进程fork出应用进程。由于应用程序进程的创建涉及到了Android的一些底层原理，包括Linux的一些知识，目前我也不能说得很清楚，因此就简单介绍一下，感兴趣的话自行查阅资料吧，这里附上一张[Gityuan](http://gityuan.com)大神绘制的的流程图。

![App进程创建图](http://gityuan.com/images/android-process/start_app_process.jpg)

进程创建完成后，就会执行**ActivityThread**的`main()`方法，它是应用程序的入口方法（当然从应用进程创建到执行`main()`方法这中间的过程还要更复杂，本文就不深入探讨了）。

```java
public static void main(String[] args) {
    // ...
    // 创建主线程Looper
    Looper.prepareMainLooper();

    // ...	
  	// 核心代码
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);

    // ...
    // 开启消息循环
    Looper.loop();
    // ...
}
```

`main()`方法内部创建了主线程Looper并开启了消息循环，由于这不是本文的重点，就不多提了。此外，还创建了ActivityThread对象并调用了`attach()`方法，我们接下来看一下这个方法。

```java
private void attach(boolean system, long startSeq) {
    // ...
  	// 核心代码
    final IActivityManager mgr = ActivityManager.getService();
    mgr.attachApplication(mAppThread, startSeq);
    // ...
}
```

这里省略了大量代码，核心代码就是调用`mgr.attachApplication()`方法，其中mgr是通过`ActivityManager.getService()`得到的，因此就是ActivityManagerService对象，我们来看AMS的`attachApplication()`方法：

```java
@Override
public final void attachApplication(IApplicationThread thread, long startSeq) {
    synchronized (this) {
        int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();
        attachApplicationLocked(thread, callingPid, callingUid, startSeq);
        Binder.restoreCallingIdentity(origId);
    }
}
```

`attachApplication()`方法内部又调用了`attachApplicationLocked()`方法：

```java
private final boolean attachApplicationLocked(IApplicationThread thread,
                                              int pid, int callingUid, long startSeq) {
    // ...
    if (app.isolatedEntryPoint != null) {
        // This is an isolated process which should just call an entry point instead of
        // being bound to an application.
        thread.runIsolatedEntryPoint(app.isolatedEntryPoint, app.isolatedEntryPointArgs);
    } else if (app.instr != null) {
        thread.bindApplication(processName, appInfo, providers,
                app.instr.mClass,
                profilerInfo, app.instr.mArguments,
                app.instr.mWatcher,
                app.instr.mUiAutomationConnection, testMode,
                mBinderTransactionTrackingEnabled, enableTrackAllocation,
                isRestrictedBackupMode || !normalMode, app.persistent,
                new Configuration(getGlobalConfiguration()), app.compat,
                getCommonServicesLocked(app.isolated),
                mCoreSettingsObserver.getCoreSettingsLocked(),
                buildSerial, isAutofillCompatEnabled);
    } else {
        thread.bindApplication(processName, appInfo, providers, null, profilerInfo,
                null, null, null, testMode,
                mBinderTransactionTrackingEnabled, enableTrackAllocation,
                isRestrictedBackupMode || !normalMode, app.persistent,
                new Configuration(getGlobalConfiguration()), app.compat,
                getCommonServicesLocked(app.isolated),
                mCoreSettingsObserver.getCoreSettingsLocked(),
                buildSerial, isAutofillCompatEnabled);
    }
    // ...
    if (normalMode) {
        try {
            if (mStackSupervisor.attachApplicationLocked(app)) {
                didSomething = true;
            }
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
            badApp = true;
        }
    }
    // ...
    return true;
}
```

这里省略了大量代码，`attachApplicationLocked()`内部主要调用了两个关键方法：第一个是`thread.bindApplication()`；第二个是`mStackSupervisor.attachApplicationLocked(app)`，下面我们就分别来看一下这两个方法都做了什么。

* `thread.bindApplication()`方法

这里的thread即上面调用ActivityThread的`attach()`方法传过来的mAppThread，它的类型为**ApplicationThread**，我们来看一下ApplicationThread的`bindApplication()`方法：

```java
public final void bindApplication(String processName, ApplicationInfo appInfo,
                                  List<ProviderInfo> providers, ComponentName instrumentationName,
                                  ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                                  IInstrumentationWatcher instrumentationWatcher,
                                  IUiAutomationConnection instrumentationUiConnection, int debugMode,
                                  boolean enableBinderTracking, boolean trackAllocation,
                                  boolean isRestrictedBackupMode, boolean persistent, Configuration config,
                                  CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
                                  String buildSerial, boolean autofillCompatibilityEnabled) {

    // ...
    sendMessage(H.BIND_APPLICATION, data);
}
```

可以看到方法最后调用了我们熟悉的`sendMessage()`方法，接下来找到**H**的`handleMessage()`方法对**BIND_APPLICATION**消息的处理。

```java
public void handleMessage(Message msg) {
    // ...
    switch (msg.what) {
        case BIND_APPLICATION:
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
            AppBindData data = (AppBindData) msg.obj;
            handleBindApplication(data);
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            break;
        // ...
    }
    // ...
}
```

接着调用了`handleBindApplication()`方法：

```java
private void handleBindApplication(AppBindData data) {
    // ...
    if (ii != null) {
        ApplicationInfo instrApp;
        try {
            instrApp = getPackageManager().getApplicationInfo(ii.packageName, 0,
                    UserHandle.myUserId());
        } catch (RemoteException e) {
            instrApp = null;
        }
        if (instrApp == null) {
            instrApp = new ApplicationInfo();
        }
        ii.copyTo(instrApp);
        instrApp.initForUser(UserHandle.myUserId());
        final LoadedApk pi = getPackageInfo(instrApp, data.compatInfo,
                appContext.getClassLoader(), false, true, false);
        final ContextImpl instrContext = ContextImpl.createAppContext(this, pi);

        try {
            final ClassLoader cl = instrContext.getClassLoader();
            // 创建Instrumentation对象
            mInstrumentation = (Instrumentation)
                    cl.loadClass(data.instrumentationName.getClassName()).newInstance();
        } catch (Exception e) {
            throw new RuntimeException(
                    "Unable to instantiate instrumentation "
                            + data.instrumentationName + ": " + e.toString(), e);
        }

        final ComponentName component = new ComponentName(ii.packageName, ii.name);
        mInstrumentation.init(this, instrContext, appContext, component,
                data.instrumentationWatcher, data.instrumentationUiAutomationConnection);

        if (mProfiler.profileFile != null && !ii.handleProfiling
                && mProfiler.profileFd == null) {
            mProfiler.handlingProfiling = true;
            final File file = new File(mProfiler.profileFile);
            file.getParentFile().mkdirs();
            Debug.startMethodTracing(file.toString(), 8 * 1024 * 1024);
        }
    } else {
        mInstrumentation = new Instrumentation();
        mInstrumentation.basicInit(this);
    }

    // ...
    Application app;
    // 创建Application
    app = data.info.makeApplication(data.restrictedBackupMode, null);
    // ...
}
```

方法内部创建了**Instrumentation**实例对象，最后调用了`data.info.makeApplication()`方法，data.info的类型为**LoadedApk**，`makeApplication()`方法在Activity的启动流程分析中也提到过，是通过类加载器创建Application对象，并且可以保证只有一个Application实例，这里就不展示了。

因此`bindApplication()`方法的目的就是创建Application实例对象。

* `mStackSupervisor.attachApplicationLocked(app)`方法

mStackSupervisor的类型为**ActivityStackSupervisor**，我们来看ActivityStackSupervisor的`attachApplicationLocked()`方法：

```java
boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
    // ...    
    if (realStartActivityLocked(activity, app, top == activity, true)) {
        didSomething = true;
    }
    // ...
    return didSomething;
}
```

`attachApplicationLocked()`方法内部调用了`realStartActivityLocked()`方法，这个方法我们可能有点印象，在Activity的启动流程中也有分析到该方法，我们再来简单回顾一下。

```java
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
                                      boolean andResume, boolean checkConfig) throws RemoteException {
    // ...
    // 创建Activity启动事务
    final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
            r.appToken);
    clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
            System.identityHashCode(r), r.info,
            mergedConfiguration.getGlobalConfiguration(),
            mergedConfiguration.getOverrideConfiguration(), r.compat,
            r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
            r.persistentState, results, newIntents, mService.isNextTransitionForward(),
            profilerInfo));

    // Set desired final state.
    final ActivityLifecycleItem lifecycleItem;
    if (andResume) {
        lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());
    } else {
        lifecycleItem = PauseActivityItem.obtain();
    }
    clientTransaction.setLifecycleStateRequest(lifecycleItem);
  
    // 执行事务
    mService.getLifecycleManager().scheduleTransaction(clientTransaction);
    // ...
}
```

接下来的流程就和Activity的启动相同了，最终通过发送消息依次调用ActivityThread的`handleLaunchActivity()`方法和`performLaunchActivity()`方法，完成Activity的创建等工作，回调Activity的`onCreate()`生命周期方法，就不再重复分析了。

分析到这里，App的简单启动流程我们就清楚了，基本上和Activity的启动流程类似，不同之处是需要创建应用进程，一个简单的流程图如下所示：

![App启动流程](https://raw.githubusercontent.com/StephenZKCurry/Android-Study-Notes/master/Android/App%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.jpg)

## 总结

分析过Activity的启动流程后再来看App的启动流程会简单很多，因为有很多流程是相同的，本质上来开还是启动一个新的Activity，区别主要就在于需要创建应用进程，限于自身水平原因，这部分我也没有详细分析，有兴趣的话还是可以了解一下的。

## 参考文章

[死磕Android_App 启动过程（含 Activity 启动过程）](https://blog.csdn.net/xfhy_/article/details/90679525)

[Android启动流程 、app安装和启动原理](https://www.jianshu.com/p/12de32b31836)


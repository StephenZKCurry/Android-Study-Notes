# Activity启动流程

> **前言**
>
> Activity可以算得上是Android开发中接触最早和最多的一个类了，简单的使用相信大家都很熟悉了，本文主要分析一下Activity的启动流程，有助于加深我们对于Android知识体系的理解，拓宽我们的视野。

## 源码分析

首先强调一下，**本文的分析基于Android 9.0（API Level 28）的源码**，不同版本的源码可能有些不同，大体流程还是差不多的。阅读源码没什么可说的，只能一步一步去跟代码，下面我们就直接开始吧。

通常我们在启动一个Activity时只需要调用如下代码即可：

```java
Intent intent = new Intent(this, XXActivity.class);
startActivity(intent);
```

从Activity的`startActivity(intent)`方法入手，方法的调用链如下：

```java
@Override
public void startActivity(Intent intent) {
    this.startActivity(intent, null);
}

@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        // Note we want to go through this call for compatibility with
        // applications that may have overridden the method.
        startActivityForResult(intent, -1);
    }
}

public void startActivityForResult(@RequiresPermission Intent intent, int requestCode) {
    startActivityForResult(intent, requestCode, null);
}
```

可以看出`startActivity(intent)`方法最终会走到`startActivityForResult(Intent intent, int requestCode,Bundle options)`方法，也省去了我们再去分析`startActivityForResult()`方法。

**Activity的startActivityForResult方法**

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
                                   @Nullable Bundle options) {
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
      	// 核心代码
        Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                        this, mMainThread.getApplicationThread(), mToken, this,
                        intent, requestCode, options);
        // ...
    } else {
       	// ...
    }
}
```

这里我们只需要关注`mParent == null`这部分的逻辑，这里的核心代码是调用`mInstrumentation.execStartActivity()`方法。mInstrumentation的类型是**Instrumentation**，它会在应用程序的任何代码运行之前被实例化，能够允许你监视应用程序和系统的所有交互，Application和Activity的创建以及生命周期都会经过这个对象去执行。mMainThread的类型是**ActivityThread**，应用程序的入口就是该类中的`main()`方法，在该方法中会创建主线程Looper，开启消息循环，处理任务。通过`mMainThread.getApplicationThread()`方法获取到的是一个**ApplicationThread**对象，它是ActivityThread的内部类，它的定义如下：

```java
private class ApplicationThread extends IApplicationThread.Stub{
   // ... 
}
```

可以看出它继承自IApplicationThread.Stub，如果了解一点Binder的话从名称就能知道**IApplicationThread**继承自**IInterface**接口，内部定义了一些业务方法，IApplicationThread.Stub则是一个**Binder**对象。接下来我们来看一下Instrumentation的`execStartActivity()`方法。

**Instrumentation的execStartActivity方法**

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
  	// 将contextThread转为IApplicationThread
  	// 这里的contextThread是一个ApplicationThread对象，实现了IApplicationThread接口，因此可以这样转换
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    // ...
    try {
        // ...
      	// 核心代码
        int result = ActivityManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
      	// 检查Activity启动结果
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

这里的核心代码是`ActivityManager.getService().startActivity()`，`ActivityManager.getService()`方法获取到的是一个**ActivityManagerService**对象，简称**AMS**，为了简便，下文也都将其统称为“AMS”。AMS可就厉害了，它是Android中的核心服务，负责调度各应用的进程，管理四大组件。AMS是一个**Binder**，实现了**IActivityManager**接口，因此应用进程能通过Binder机制调用系统服务。

```java
public static IActivityManager getService() {
    return IActivityManagerSingleton.get();
}

private static final Singleton<IActivityManager> IActivityManagerSingleton =
        new Singleton<IActivityManager>() {
            @Override
            protected IActivityManager create() {
              	// 获取服务Binder
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
              	// 获取AMS实例对象
                final IActivityManager am = IActivityManager.Stub.asInterface(b);
                return am;
            }
        };
```

值得一提的是**ServiceManager**管理着所有的系统服务，可以将其类比于DNS域名服务器，通过服务名称可以从ServiceManager中获取到相应的服务Binder对象。`checkStartActivityResult()`方法的作用是检查Activity的启动结果，参数传入了`startActivity()`方法的返回值，根据返回值判断Activity是否启动成功，如果启动失败会抛出异常，常见的**Unable to find explicit activity class; have you declared this activity in your AndroidManifest.xml**异常就是这里抛出的。

接下来我们来看AMS对象的`startActivity()`方法：

**ActivityManagerService的startActivity方法**

```java
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
                               Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
                               int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}

@Override
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
                                     Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
                                     int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
            true /*validateIncomingUser*/);
}

public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
                                     Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
                                     int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
                                     boolean validateIncomingUser) {
    enforceNotIsolatedCaller("startActivity");

    userId = mActivityStartController.checkTargetUser(userId, validateIncomingUser,
            Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

    return mActivityStartController.obtainStarter(intent, "startActivityAsUser")
            .setCaller(caller)
            .setCallingPackage(callingPackage)
            .setResolvedType(resolvedType)
            .setResultTo(resultTo)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setStartFlags(startFlags)
            .setProfilerInfo(profilerInfo)
            .setActivityOptions(bOptions)
            .setMayWait(userId)
            .execute();

}
```

可以看出AMS的`startActivity()`方法最终会走到`startActivityAsUser()`方法中，方法内部是一个挺长的链式调用，第一行的`mActivityStartController.obtainStarter()`返回的是一个**ActivityStarter**对象，之后的一系列调用可以看做是对ActivityStarter的配置，我们直接来看最后调用的`execute()`方法。

**ActivityStarter的execute方法**

```java
int execute() {
    try {
        // for transactional diffs and preprocessing.
        if (mRequest.mayWait) {
            return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                    mRequest.callingPackage, mRequest.intent, mRequest.resolvedType,
                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                    mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                    mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                    mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                    mRequest.inTask, mRequest.reason,
                    mRequest.allowPendingRemoteAnimationRegistryLookup);
        } else {
            return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                    mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                    mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                    mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                    mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                    mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                    mRequest.outActivity, mRequest.inTask, mRequest.reason,
                    mRequest.allowPendingRemoteAnimationRegistryLookup);
        }
    } finally {
        onExecutionComplete();
    }
}
```

这里有一个判断，我们可以不用去了解它的意义，直接来看`startActivityMayWait()`方法：

```java
private int startActivityMayWait(IApplicationThread caller, int callingUid,
                                 String callingPackage, Intent intent, String resolvedType,
                                 IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                                 IBinder resultTo, String resultWho, int requestCode, int startFlags,
                                 ProfilerInfo profilerInfo, WaitResult outResult,
                                 Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
                                 int userId, TaskRecord inTask, String reason,
                                 boolean allowPendingRemoteAnimationRegistryLookup) {
    // ...
  	// 调用startActivity方法
    int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
            voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
            callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
            ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
            allowPendingRemoteAnimationRegistryLookup);
    // ...
    return res;
}
```

这里省略了大量代码，可以看到`startActivityMayWait()`方法最终也是调用了`startActivity()`方法，因此我们不需要搞清楚上面判断的意义，因为最终执行的方法是一样的。接下来来看ActivityStarter的`startActivity()`方法：

```java
// 1
private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
                          String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
                          IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                          IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
                          String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
                          SafeActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
                          ActivityRecord[] outActivity, TaskRecord inTask, String reason,
                          boolean allowPendingRemoteAnimationRegistryLookup) {

    // ...
    mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,
            aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
            callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
            options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
            inTask, allowPendingRemoteAnimationRegistryLookup);
    // ...
    return getExternalResult(mLastStartActivityResult);
}

// 2
private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
                          String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
                          IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                          IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
                          String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
                          SafeActivityOptions options,
                          boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
                          TaskRecord inTask, boolean allowPendingRemoteAnimationRegistryLookup) {
    // ...
    return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
            true /* doResume */, checkedOptions, inTask, outActivity);
}

// 3
private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                          IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                          int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                          ActivityRecord[] outActivity) {

    // ...
    result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, doResume, options, inTask, outActivity);
    // ...
    return result;
}

// 4
private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                          IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                          int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                          ActivityRecord[] outActivity) {
    // ...
    mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack,mStartActivity,
            mOptions);
    // ...
    return START_SUCCESS;
}
```

可以看到经过一系列调用（省略了大量代码）最终走到了`mSupervisor.resumeFocusedStackTopActivityLocked()`方法，mSupervisor的类型是**ActivityStackSupervisor**，我们接着来看ActivityStackSupervisor的`resumeFocusedStackTopActivityLocked()`方法：

**ActivityStackSupervisor的resumeFocusedStackTopActivityLocked方法**

```java
boolean resumeFocusedStackTopActivityLocked(
        ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
    // ...
    return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
    // ...
}
```

方法内部调用了`targetStack.resumeTopActivityUncheckedLocked()`方法，targetStack的类型是**ActivityStack**，我们来看它的`resumeTopActivityUncheckedLocked()`方法：

**ActivityStack的resumeTopActivityUncheckedLocked方法**

```java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    // ...
    result = resumeTopActivityInnerLocked(prev, options);
    // ...
    return result;
}

private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    // ...
    mStackSupervisor.startSpecificActivityLocked(next, true, false);
    // ...
    return true;
}
```

最后又回到了ActivityStackSupervisor的`startSpecificActivityLocked()`方法。

**ActivityStackSupervisor的startSpecificActivityLocked方法**

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

方法内部首先判断了应用程序进程是否存在，如果进程已创建就调用` realStartActivityLocked()` 方法，否则执行` mService.startProcessLocked()`方法常见应用进程，mService就是ActivityManagerService对象。本文只讨论Activity的启动流程，因此默认进程已创建，关于进程未创建的情况，对应于点击应用图标启动App的过程，之后我会再利用一篇笔记进行分析。接下来我们来看一下`realStartActivityLocked()`方法：

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

`mService.getLifecycleManager()`方法获取到的是一个**ClientLifecycleManager**对象，用来管理Activity的生命周期，可以看到`realStartActivityLocked()`方法内创建了一个**ClientTransaction**，表示Activity的启动事务，最后调用ClientLifecycleManager的`scheduleTransaction()`方法执行该事务。接下来我们来看ClientLifecycleManager的`scheduleTransaction()`方法：

**ClientLifecycleManager的scheduleTransaction方法**

```java
void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    final IApplicationThread client = transaction.getClient();
    transaction.schedule();
    if (!(client instanceof Binder)) {
        // If client is not an instance of Binder - it's a remote call and at this point it is
        // safe to recycle the object. All objects used for local calls will be recycled after
        // the transaction is executed on client in ActivityThread.
        transaction.recycle();
    }
}
```

接着调用了ClientTransaction的`schedule()`方法：

**ClientTransaction的schedule方法**

```java
public void schedule() throws RemoteException {
    mClient.scheduleTransaction(this);
}
```

mClient的类型是IApplicationThread，最开始也提到过，ApplicationThread实现了IApplicationThread接口，再回过头看上面的逐级调用，不难发现这里的mClient就是最开始传入的ApplicationThread对象，因此我们直接来看ApplicationThread的`scheduleTransaction()`方法：

**ApplicationThread的scheduleTransaction方法**

```java
@Override
public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    ActivityThread.this.scheduleTransaction(transaction);
}
```

这里又调用了ActivityThread的`scheduleTransaction()`方法，ActivityThread中没有定义这个方法，它定义在ActivityThread的父类**ClientTransactionHandler**中。

**ClientTransactionHandler的scheduleTransaction方法**

```java
void scheduleTransaction(ClientTransaction transaction) {
    transaction.preExecute(this);
    sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
}
```

方法内部调用了`sendMessage()`方法，该方法是一个抽象方法，它的实现在ActivityThread中。

**ActivityThread的sendMessage方法**

```java
void sendMessage(int what, Object obj) {
    sendMessage(what, obj, 0, 0, false);
}

private void sendMessage(int what, Object obj, int arg1) {
    sendMessage(what, obj, arg1, 0, false);
}

private void sendMessage(int what, Object obj, int arg1, int arg2) {
    sendMessage(what, obj, arg1, arg2, false);
}

private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
    if (DEBUG_MESSAGES) Slog.v(
            TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
                    + ": " + arg1 + " / " + obj);
    Message msg = Message.obtain();
    msg.what = what;
    msg.obj = obj;
    msg.arg1 = arg1;
    msg.arg2 = arg2;
    if (async) {
        msg.setAsynchronous(true);
    }
    mH.sendMessage(msg);
}
```

mH的类型是**H**，是ActivityThread内部定义的一个**Handler**类，`sendMessage()`方法所做的就是创建并向Handler发送一个消息，这个消息的what值为**EXECUTE_TRANSACTION**。值得一提的是，在Android 9.0以下版本中，H中定义了**LAUNCH_ACTIVITY**、**PAUSE_ACTIVITY**、**RESUME_ACTIVITY**这些生命周期相关的消息，而在Android 9.0中已经去掉了这些消息，取而代之的就是**EXECUTE_TRANSACTION**这个消息。下面我们就来看H的`handleMessage()`方法，找到对**EXECUTE_TRANSACTION**这个消息的处理。

```java
public void handleMessage(Message msg) {
    // ...
    switch (msg.what) {
        // ...
        case EXECUTE_TRANSACTION:
            final ClientTransaction transaction = (ClientTransaction) msg.obj;
        	// 核心代码
            mTransactionExecutor.execute(transaction);
            if (isSystem()) {
                // Client transactions inside system process are recycled on the client side
                // instead of ClientLifecycleManager to avoid being cleared before this
                // message is handled.
                transaction.recycle();
            }
            // TODO(lifecycler): Recycle locally scheduled transactions.
            break;
        // ...
    }
    // ...
}
```

这里的核心代码是`mTransactionExecutor.execute(transaction)`，mTransactionExecutor的类型是**TransactionExecutor**，我们来看TransactionExecutor的`execute()`方法：

**TransactionExecutor的execute方法**

```java
public void execute(ClientTransaction transaction) {
    final IBinder token = transaction.getActivityToken();
    log("Start resolving transaction for client: " + mTransactionHandler + ", token: " + token);

    executeCallbacks(transaction);

    executeLifecycleState(transaction);
    mPendingActions.clear();
    log("End resolving transaction");
}

public void executeCallbacks(ClientTransaction transaction) {
    // 这里的callbacks是上面调用ActivityStackSupervisor中realStartActivityLocked方法时赋值的
    // ClientTransactionItem的实际类型为LaunchActivityItem
    final List<ClientTransactionItem> callbacks = transaction.getCallbacks();

    // ...
    for (int i = 0; i < size; ++i) {
        final ClientTransactionItem item = callbacks.get(i);
        final int postExecutionState = item.getPostExecutionState();
        final int closestPreExecutionState = mHelper.getClosestPreExecutionState(r,
                item.getPostExecutionState());
        // ...
        // 核心代码
        item.execute(mTransactionHandler, token, mPendingActions);
        item.postExecute(mTransactionHandler, token, mPendingActions);
        // ...
    }
}
```

`execute()`方法中调用了两个方法：`executeCallbacks(transaction)`和`executeLifecycleState(transaction)`。我们先不去管`executeLifecycleState()`方法，直接来看`executeCallbacks()`方法，方法内部首先获取了ClientTransaction中的callbacks，这里的callbacks是上面调用ActivityStackSupervisor中`realStartActivityLocked()`方法时赋值的，类型为**LaunchActivityItem**，之后会调用LaunchActivityItem的`execute()`方法。

**LaunchActivityItem的execute方法**

```java
public void execute(ClientTransactionHandler client, IBinder token,
                    PendingTransactionActions pendingActions) {
    // ...
    // 核心代码
    client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    // ...
}
```

方法内部调用了ClientTransactionHandler的`handleLaunchActivity()`方法，前面也提到过ActivityThread继承自ClientTransactionHandler，因次这里实际上调用的是ActivityThread的`handleLaunchActivity()`方法，终于回到了我们熟悉的类中。

**ActivityThread的handleLaunchActivity方法**

```java
public Activity handleLaunchActivity(ActivityClientRecord r,
                                     PendingTransactionActions pendingActions, Intent customIntent) {
    // ...
    // 核心代码
    final Activity a = performLaunchActivity(r, customIntent);
    // ...
    return a;
}
```

`handleLaunchActivity()`方法中又调用了`performLaunchActivity()`方法：

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

`performLaunchActivity()`方法内部主要做了一下几件事：

* **调用Instrumentation的`newActivity`方法创建Activity对象**

```java
public Activity newActivity(ClassLoader cl, String className,
                            Intent intent)
        throws InstantiationException, IllegalAccessException,
        ClassNotFoundException {
    String pkg = intent != null && intent.getComponent() != null
            ? intent.getComponent().getPackageName() : null;
    return getFactory(pkg).instantiateActivity(cl, className, intent);
}

// AppComponentFactory的instantiateActivity方法
public Activity instantiateActivity(ClassLoader cl, String className,
                                    Intent intent)
        throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    return (Activity) cl.loadClass(className).newInstance();
}
```

可以看出Instrumentation的`newActivity`方法内部就是通过类加载器来创建Activity对象。

* **调用`r.packageInfo.makeApplication()`方法创建Application对象**

```java
public Application makeApplication(boolean forceDefaultAppClass,
                                   Instrumentation instrumentation) {
  // 判断Application对象是否已创建，如果已创建就直接返回
  if (mApplication != null) {
        return mApplication;
    }

    // ...

    Application app = null;

    String appClass = mApplicationInfo.className;
    if (forceDefaultAppClass || (appClass == null)) {
        appClass = "android.app.Application";
    }
    
    java.lang.ClassLoader cl = getClassLoader();
    if (!mPackageName.equals("android")) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                "initializeJavaContextClassLoader");
        initializeJavaContextClassLoader();
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    }
    ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
    // 核心代码
  	// 通过类加载器创建Application对象
    app = mActivityThread.mInstrumentation.newApplication(
            cl, appClass, appContext);
    appContext.setOuterContext(app);

    mActivityThread.mAllApplications.add(app);
    mApplication = app;

    // ...

    return app;
}
```

这里的r.packageInfo类型为**LoadedApk**。`makeApplication()`方法内部首先判断了Application对象是否已创建，保证了只有一个Application对象。Application对象是通过调用Instrumentation的`newApplication()`方法来创建的，方法内部同样使用了类加载器，这里就不展示了。

* **调用Activity的`attach()`方法**

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

    // ...

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

`attach()`方法主要做了两件事：创建Window对象，类型为PhotoWindow；为Window对象关联WindowManager，这属于View绘制的研究范畴，本文中就不具体研究了。

* **调用Instrumentation的`callActivityOnCreate()`方法**

```java
public void callActivityOnCreate(Activity activity, Bundle icicle) {
    prePerformCreate(activity);
    activity.performCreate(icicle);
    postPerformCreate(activity);
}

public void callActivityOnCreate(Activity activity, Bundle icicle,
                                 PersistableBundle persistentState) {
    prePerformCreate(activity);
    activity.performCreate(icicle, persistentState);
    postPerformCreate(activity);
}
```

方法内部调用了Activity的`performCreate()`方法。

**Activity的performCreate方法**

```java
final void performCreate(Bundle icicle) {
    performCreate(icicle, null);
}

final void performCreate(Bundle icicle, PersistableBundle persistentState) {
    // ...
    // 调用Activity的onCreate()方法
    if (persistentState != null) {
        onCreate(icicle, persistentState);
    } else {
        onCreate(icicle);
    }
    // ...
}
```

方法内部调用了我们最熟悉的Activity生命周期`onCreate()`方法，到这里Activity的启动流程基本上就分析得差不多了。不过你可能还会有个疑惑，`onCreate()`之后的`onStart()`和`onResume()`是在哪里调用的呢，我在这里也是绕了好久，我看到网上的一些源码分析中有一些是在`performLaunchActivity()`方法最后调用了Activity的`performStart()`方法，进而回调了`onStart()`方法，可能是由于版本不同，我这里并没有调用该方法。还记得上面的分析中在TransactionExecutor的`execute()`方法中调用的`executeLifecycleState()`吗，是时候来看一下这个方法了。

**TransactionExecutor的executeLifecycleState方法**

```java
private void executeLifecycleState(ClientTransaction transaction) {
    // lifecycleItem的类型为ResumeActivityItem
    final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
    if (lifecycleItem == null) {
        // No lifecycle request, return early.
        return;
    }

    // ...

    // Cycle to the state right before the final requested state.
    cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */);

    // Execute the final transition with proper parameters.
    lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
    lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
}
```

这里`transaction.getLifecycleStateRequest()`获取到的ActivityLifecycleItem同样是在之前调用ActivityStackSupervisor中的`realStartActivityLocked()`方法时赋值的，类型为**ResumeActivityItem**，之后调用了`cycleToPath()`方法。

```java
private void cycleToPath(ActivityClientRecord r, int finish,
                         boolean excludeLastState) {
    final int start = r.getLifecycleState();
    log("Cycle from: " + start + " to: " + finish + " excludeLastState:" + excludeLastState);
    final IntArray path = mHelper.getLifecyclePath(start, finish, excludeLastState);
    performLifecycleSequence(r, path);
}
```

`cycleToPath()`方法内部会调用**TransactionExecutorHelper**的`getLifecyclePath()`方法，该方法的作用是根据传入的起始生命周期状态和最终生命周期状态生成一个数组，比如当前生命周期状态为为**ON_CREATE**，而finish状态为`lifecycleItem.getTargetState()`，也就是`ResumeActivityItem.getTargetState()`即**ON_RESUME**，中间间隔了一个生命周期状态**ON_START**，因此这里生成的数组包含一个元素**ON_START**。Activity生命周期对应的状态值如下：

```java
public static final int ON_CREATE = 1;
public static final int ON_START = 2;
public static final int ON_RESUME = 3;
public static final int ON_PAUSE = 4;
public static final int ON_STOP = 5;
public static final int ON_DESTROY = 6;
public static final int ON_RESTART = 7;
```

接下来会调用`performLifecycleSequence()`方法：

```java
private void performLifecycleSequence(ActivityClientRecord r, IntArray path) {
    final int size = path.size();
    for (int i = 0, state; i < size; i++) {
        state = path.get(i);
        log("Transitioning to state: " + state);
        switch (state) {
            case ON_CREATE:
                mTransactionHandler.handleLaunchActivity(r, mPendingActions,
                        null /* customIntent */);
                break;
            case ON_START:
                mTransactionHandler.handleStartActivity(r, mPendingActions);
                break;
            case ON_RESUME:
                mTransactionHandler.handleResumeActivity(r.token, false /* finalStateRequest */,
                        r.isForward, "LIFECYCLER_RESUME_ACTIVITY");
                break;
            case ON_PAUSE:
                mTransactionHandler.handlePauseActivity(r.token, false /* finished */,
                        false /* userLeaving */, 0 /* configChanges */, mPendingActions,
                        "LIFECYCLER_PAUSE_ACTIVITY");
                break;
            case ON_STOP:
                mTransactionHandler.handleStopActivity(r.token, false /* show */,
                        0 /* configChanges */, mPendingActions, false /* finalStateRequest */,
                        "LIFECYCLER_STOP_ACTIVITY");
                break;
            case ON_DESTROY:
                mTransactionHandler.handleDestroyActivity(r.token, false /* finishing */,
                        0 /* configChanges */, false /* getNonConfigInstance */,
                        "performLifecycleSequence. cycling to:" + path.get(size - 1));
                break;
            case ON_RESTART:
                mTransactionHandler.performRestartActivity(r.token, false /* start */);
                break;
            default:
                throw new IllegalArgumentException("Unexpected lifecycle state: " + state);
        }
    }
}
```

可以看出这里会根据上一步生成的数组依次执行相应的生命周期相关方法，由于这里的数组只包含一个元素**ON_START**，因此会执行`mTransactionHandler.handleStartActivity()`方法，之后的步骤就和`onCreate()`的执行过程很相似了，即调用ActivityThread的`handleStartActivity()`方法，方法内部调用Activity的`performStart()`方法，继而回调生命周期`onStart()`方法，这里就不具体展示源码分析了。

接下来我们回到`executeLifecycleState()`方法中，在执行`cycleToPath()`方法后调用了`lifecycleItem.execute()`方法，由于这里的lifecycleItem是ResumeActivityItem，因此我们来看ResumeActivityItem的`execute()`方法：

**ResumeActivityItem的execute方法**

```java
@Override
public void execute(ClientTransactionHandler client, IBinder token,
                    PendingTransactionActions pendingActions) {
    Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityResume");
  	// 核心代码
    client.handleResumeActivity(token, true /* finalStateRequest */, mIsForward,
            "RESUME_ACTIVITY");
    Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}
```

可以看到和此前的两个生命周期方法调用类似，同样是先调用了ActivityThread的`handleResumeActivity()`方法，之后调用`performResumeActivity()`，最后会调用`onResume()`生命周期方法。

上述的流程只是我的简单分析，实际过程肯定要更复杂，个人水平比较有限，因此只能这样简单地分析一下（PS：加断点调试把我搞得有点蒙，实际执行的流程其实并不是像我分析得这样，有的方法会执行多次），不正确的地方欢迎指出。

到这里整个Activity的启动流程就分析得差不多了，实际的流程要比分析的复杂得多，也会涉及到很多其他的知识点，包括View的绘制和进程的创建等等，由于水平有限很多地方只能浅尝辄止，太过深入源码容易迷失方向。上一张流程图总结一下吧：

![Activity启动流程](https://github.com/StephenZKCurry/Android-Study-Notes/blob/master/Android/Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.jpg?raw=true)

从分析和图中我们也能发现在Activity的启动流程中有几个类起到了至关重要的作用：

- **Instrumentation**

最开始调用AMS方法，最后创建Activity和Application对象都是通过它来实现的。

- **ActivityManagerService**

Android中的核心服务，应用进程通过Binder机制可以调用系统服务。

- **ActivityThread**

应用程序的入口，Activity各个生命周期的调用都是通过ActivityThread内部的Handler接收和处理消息来实现的。

## 总结

本文是针对Android 9.0（API Level 28）的源码进行分析的，与其他版本会有一些不同，但是大体流程不会差太多，主要差别就是在调用`realStartActivityLocked()`方法之后Handler的消息处理，有兴趣的话可以自行查看源码。由于个人水平不足，很多地方可能分析得不准确，也不够全面，希望大佬们可以指出。

最后总结一下我的一些心得，对于目前的我来说分析源码实在是一件枯燥痛苦的事，需要了解的知识点有很多，文章前前后后写了很久，中间中断了好多次，也借鉴了很多大佬的成果，终于算是写完了。但是抛开过程中的痛苦，当真正把一个流程理顺之后还是会觉得心情舒畅，虽然文章中的一些分析可能一段时间后还是会遗忘，但是我收获更多的是源码阅读和分析能力的提高，而且自己真正从头分析了一遍之后再复习起来就会容易的多。希望今后能够更多地阅读一些系统或者优秀第三方库的源码，不断提高自己。

## 参考资料

[死磕Android_App 启动过程（含 Activity 启动过程）](https://blog.csdn.net/xfhy_/article/details/90679525)

[Android9.0 Activity启动流程分析（三）](https://blog.csdn.net/caiyu_09/article/details/84837544)

《Android开发艺术探索》
# Activity启动流程

**本文的分析基于API28的源码**

通常我们在启动一个Activity时只需要调用如下代码即可：

```java
Intent intent = new Intent(this, XXActivity.class);
startActivity(intent);
```

从`startActivity(intent)`方法入手，方法的调用链如下：

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
        // 此处省略部分代码
    } else {
       	// 此处省略部分代码
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
    // 此处省略部分代码
    try {
        // 此处省略部分代码
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

这里的核心代码是`ActivityManager.getService().startActivity()`，`ActivityManager.getService()`方法获取到的是一个**ActivityManagerService**对象，简称**AMS**，为了简便，下文也都将其统称为“AMS”。AMS可就厉害了，它是Android中的核心服务，负责调度各应用的进程，管理四大组件。ActivityManagerService，是一个**Binder**，实现了**IActivityManager**接口，应用进程能通过Binder机制调用系统服务。

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

值得一提的是**ServiceManager**管理着所有的系统服务，可以将其类比于DNS域名服务器，通过服务名称可以从ServiceManager中获取到相应的服务Binder对象。`checkStartActivityResult()`方法的作用是检查Activity的启动结果，参数传入了`startActivity()`方法的返回值，根据返回值判断Activity是否启动成功，如果启动失败会抛出异常，常见的**Unable to find explicit activity class; have you declared this activity in your AndroidManifest.xml?**异常就是这里抛出的。

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

可以看出AMS的`startActivity()`方法最终会走到`startActivityAsUser()`方法中，方法内部是一个挺长的链式调用，第一行的`mActivityStartController.obtainStarter()`返回的是一个**ActivityStarter**对象，之后的一系列调用可以看做是对ActivityStarter的配置，直接来看最后调用的`execute()`方法。

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
    // 此处省略部分代码
  	// 调用startActivity方法
    int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
            voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
            callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
            ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
            allowPendingRemoteAnimationRegistryLookup);
    // 此处省略部分代码
    return res;
}
```

这里省略了大量代码，可以看到`startActivityMayWait()`方法最终也是调用了`startActivity()`方法，因此我们上面不需要搞清楚判断的意义，因为最终执行的方法是一样的。接下来来看ActivityStarter的`startActivity()`方法：

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

    // 此处省略部分代码
    mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,
            aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
            callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
            options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
            inTask, allowPendingRemoteAnimationRegistryLookup);
    // 此处省略部分代码
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
    // 此处省略部分代码
    return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
            true /* doResume */, checkedOptions, inTask, outActivity);
}

// 3
private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                          IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                          int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                          ActivityRecord[] outActivity) {

    // 此处省略部分代码
    result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, doResume, options, inTask, outActivity);
    // 此处省略部分代码
    return result;
}

// 4
private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                          IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                          int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                          ActivityRecord[] outActivity) {
    // 此处省略部分代码
    mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack,mStartActivity,
            mOptions);
    // 此处省略部分代码
    return START_SUCCESS;
}
```

可以看到经过一系列调用（省略了大量代码）最终走到了`mSupervisor.resumeFocusedStackTopActivityLocked()`方法，mSupervisor的类型是**ActivityStackSupervisor**，我们接着来看ActivityStackSupervisor的`resumeFocusedStackTopActivityLocked()`方法：

**ActivityStackSupervisor的resumeFocusedStackTopActivityLocked方法**

```java
boolean resumeFocusedStackTopActivityLocked(
        ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
    // 此处省略部分代码
    return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
    // 此处省略部分代码
}
```

方法内部调用了`targetStack.resumeTopActivityUncheckedLocked()`方法，targetStack的类型是**ActivityStack**，我们来看它的`resumeTopActivityUncheckedLocked()`方法：

**ActivityStack的resumeTopActivityUncheckedLocked方法**

```java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    // 此处省略部分代码
    result = resumeTopActivityInnerLocked(prev, options);
    // 此处省略部分代码
    return result;
}

private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    // 此处省略部分代码
    mStackSupervisor.startSpecificActivityLocked(next, true, false);
    // 此处省略部分代码
    return true;
}
```

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
            // 省略部分代码
         	// 关键代码
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

方法内部首先判断了应用程序进程是否存在，如果进程已创建就调用` realStartActivityLocked()` 方法，否则执行` mService.startProcessLocked()`方法常见应用进程，mService就是ActivityManagerService对象。本文只讨论Activity的启动流程，因此默认进程已创建，关于进程未创建的情况，对应于点击应用图标启动App的过程，之后我会再利用一篇笔记进行分析。接下来我们就看一下`realStartActivityLocked()`方法：

```java
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
                                      boolean andResume, boolean checkConfig) throws RemoteException {

    // 此处省略部分代码
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
    // 此处省略部分代码
}
```

`mService.getLifecycleManager()`方法获取到的是一个**ClientLifecycleManager**对象，用来管理Activity的生命周期，可以看到`realStartActivityLocked()`方法内创建了一个**ClientTransaction**，表示Activity的启动事务，最后调用ClientLifecycleManager的`scheduleTransaction()`方法执行该事务。值得一提的是，在Android 9.0以下版本中。好了，接下来我们来看ClientLifecycleManager的`scheduleTransaction()`方法：

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

接着调用了ClientTransaction的`schedule()`方法。

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

mH的类型是**H**，这个H就是ActivityThread内部定义的一个**Handler**类，`sendMessage()`方法所做的就是创建并向Handler发送一个消息，这个消息的what值为**EXECUTE_TRANSACTION**。下面我们就来看H的`handleMessage()`方法，找到对**EXECUTE_TRANSACTION**这个消息的处理。

```java
public void handleMessage(Message msg) {
    // 此处省略部分代码
    switch (msg.what) {
        // 此处省略部分代码
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
        // 此处省略部分代码
    }
    // 此处省略部分代码
}
```

这里的核心代码是`mTransactionExecutor.execute(transaction)`，mTransactionExecutor的类型是**TransactionExecutor**，我们来看TransactionExecutor的`execute()`方法：

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

    // 此处省略部分代码
    for (int i = 0; i < size; ++i) {
        final ClientTransactionItem item = callbacks.get(i);
        final int postExecutionState = item.getPostExecutionState();
        final int closestPreExecutionState = mHelper.getClosestPreExecutionState(r,
                item.getPostExecutionState());
        // 此处省略部分代码
        // 核心代码
        item.execute(mTransactionHandler, token, mPendingActions);
        item.postExecute(mTransactionHandler, token, mPendingActions);
        // 此处省略部分代码
    }
}
```



**LaunchActivityItem的execute方法**

```java
public void execute(ClientTransactionHandler client, IBinder token,
                    PendingTransactionActions pendingActions) {
    // 此处省略部分代码
    // 核心代码
    client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    // 此处省略部分代码
}
```



**ActivityThread的handleLaunchActivity方法**

```java
public Activity handleLaunchActivity(ActivityClientRecord r,
                                     PendingTransactionActions pendingActions, Intent customIntent) {
    // 省略部分代码
    // 核心代码
    final Activity a = performLaunchActivity(r, customIntent);
    // 省略部分代码
    return a;
}
```


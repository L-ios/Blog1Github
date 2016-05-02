title: startActivity到nativeForkAndSpecialize
date: 2016-03-04 10:11:28
tags: Android, Android framework
---

背景： 因为奇酷手机开发出了双微信，通过反编译奇酷手机代码，发现需要了解手机中的startActivity,
代码背景： AOSP 5.1 for shamu

### Launcher 启动应用分析,OnClick到Acitivity.java中的startActivity
```
// #=> in Launcher.java , in Launcher class
public void onClick(View v) {
    ......
    Object tag = v.getTag();
    if (tag instanceof ShortcutInfo) {
        onClickAppShortcut(v);
    } else if (tag instanceof FolderInfo) {......}
}
```
在第6行执行应用的启动动作。
```
/* Event handler for an app shortcut click.
    这里考虑了特殊的图标的问题，不需要考虑太多，这里可以把函数忽略为如下结构*/
protected void onClickAppShortcut(final View v) {
    ......
    // Start activities
    startAppShortcutOrInfoActivity(v);
    ......
}
```
在第6行启动应用的，这个函数中还判断了点击的图标是不是应用，判断一下点击的是不是一些特殊的类
```
/* 这里主要是在设置应用的启动动画，以及启动的应用的intent*/
private void startAppShortcutOrInfoActivity(View v) {
    Object tag = v.getTag();
    final ShortcutInfo shortcut;
    final Intent intent;
    if (tag instanceof ShortcutInfo) {
        shortcut = (ShortcutInfo) tag;
        intent = shortcut.intent;
        int[] pos = new int[2];
        v.getLocationOnScreen(pos);
        intent.setSourceBounds(new Rect(pos[0], pos[1],
                pos[0] + v.getWidth(), pos[1] + v.getHeight()));

    } else if (tag instanceof AppInfo) {
        intent = ((AppInfo) tag).intent;
    } else {......}
    boolean success = startActivitySafely(v, intent, tag);
}
```
在这个函数中我们继续解封装，这里我们就看到了点击而启动应用的intent，在这个intent中有三个域是我们需要关心的
-   mAction(mAction = android.intent.action.MAIN)
-   mCategories(mCategories = android.intent.category.LAUNCHER)
-   mComponent(其中的mClass是启动应用的主Activity类名，mpackage是包名)

在20行，通过看方法名就可以猜想到这是封装了Activity.java中的startActivity用于防止无法无法启动应用，
而防止桌面挂掉。进行安全的启动活动，并且桌面在通过桌面图标启动应用的时候，只用关系是否启动成功。

```
boolean startActivitySafely(View v, Intent intent, Object tag) {
    boolean success = false;
    ......
    try {
        success = startActivity(v, intent, tag);
    } catch (ActivityNotFoundException e) {......}
    return success;
}
```
这里会判断是否在安全模式，或者是否为系统应用云云之类的。
```
boolean startActivity(View v, Intent intent, Object tag) {
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    try {
        Bundle optsBundle = null;
        .......
        if (user == null || user.equals(UserHandleCompat.myUserHandle())) {
            // Could be launching some bookkeeping activity
            startActivity(intent, optsBundle);
        } else {......}
        return true;
    } catch (SecurityException e) {......}
    return false;
}
```

intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);为将要启动的应用标记为新建任务，也就是表示将要新启动
一个应用,这里传入了optsBundle,optsBundle这里是作为Activity启动的启动动画，或者可以认为是过渡动画。这里
通过捕获的异常还可以发现，启动应用有可能引发权限的问题。在8行调用的startActivity就是Activity中的，也就是
属于系统了。
现在我们看到启动应用有两个参数是需要关心的：
-   intent(Intent)
-   optsBundle(Bundle)
启动optsBundle根据前面的分析，这里是属于启动动画，本次研究中可以不用关心，可以看作为null，然而intent就是
最主要一个参数了，根据前面我们说的intent，此处它有多了一个Flags。

### Activity.java 启动应用分析
```
public void startActivity(Intent intent) {
    this.startActivity(intent, null);
}
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        startActivityForResult(intent, -1);
    }
}
```
这里贴出的两个startActivity中，第一个是比较常用的，也是笔者学习android是调用Activity时使用的，
都可以调用处应用，所以才在前面说optsBundle可以看作为null，通过上面的代码，在调用startActivityForResult，
参数requestCode = -1，表示不要求返回结果，且这里选择options = null进行分析，
```
public void startActivityForResult(Intent intent, int requestCode) {
    startActivityForResult(intent, requestCode, null);
}
public void startActivityForResult(Intent intent, int requestCode /* -1 */, @Nullable Bundle options) {
    if (mParent == null) {
        // 此处mParent为空，因为mParent的赋值存在域setParent和attach中，这两个方法都未进行调用，所以，为空
        // mParent是Activity类
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
        if (ar != null) {......}
    } else {......}
}
```
mInstrumentatin是Activity的一个域，且通过观察Instrumentation中的execStartActivity,发现
这里有一个域是需要我们关心的，也就是mActivityMonitors,其是通过查看与之相关的addMonitor方法发
现，是通过这个方法来进行赋值和元素的添加，但是addMonitor在此之前是不会被调用的，所以
mActivityMonitors = null,返回值也为null。mMainThread.getApplicationThrad()取得应用线
程，mToken则是与ActivityManagerService进行通信的IBinder(IApplicationThread)实例。通过下面的代码我们可以看到在
ActivityManagerProxy中的startActivity进行启动应用的信息发送。
```
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target, // #=> who = target
        Intent intent, int requestCode, Bundle options) {   // #=>intent, -1, null
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    if (mActivityMonitors != null) {
        ......
    }
    try {
        .....
        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
    }
    return null;
}
```
ActivityManagerNative.getDefault()取得IActivityManager实例，这个实例是一个系统全局的实例，
其采用了单例模式，它是一个Binder通信的接口，然后进行启动应用的信息发送。下面的startActivity就
是进行消息的发送。这里相当于就是启动了第二个线程了。
```
// #=> path: frameworks/base/core/java/android/app/ActivityManagerNative.java
// #=> class: ActivityManagerProxy.class
public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
        String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(caller != null ? caller.asBinder() : null);
    data.writeString(callingPackage);
    intent.writeToParcel(data, 0);
    data.writeString(resolvedType);
    data.writeStrongBinder(resultTo);
    data.writeString(resultWho);
    data.writeInt(requestCode);
    data.writeInt(startFlags);
    if (profilerInfo != null) {
        data.writeInt(1);
        profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
    } else {
        data.writeInt(0);
    }
    if (options != null) {
        data.writeInt(1);
        options.writeToParcel(data, 0);
    } else {
        data.writeInt(0);
    }
    mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
    reply.readException();
    int result = reply.readInt();
    reply.recycle();
    data.recycle();
    return result;
}
```
利用mRemote进行消息的发送与服务关联，通过，reply来获取服务的返回值，也就是startActivity的返回
值。接下来将剖析服务层的startActivity————ActivityManagerService中的startActivity。

### ActivityManagerService中的startActivity
因为是采用Binder进行通信的，所以在ActivityManagerNative中的onTransact方法里，根据
START_ACTIVITY_TRANSACTION可以看到上面data的解析以及startActivity的调用。
下面是ActivityManagerService中的startActivity，系统服务中启动应用的起点。
```
public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle options) {
    // #=>(whoThread, who.getBasePackageName(), intent, null, token, target.mEmbeddedID, -1, 0, null, null);
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
        resultWho, requestCode, startFlags, profilerInfo, options,
        UserHandle.getCallingUserId());
}

public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {
    enforceNotIsolatedCaller("startActivity");
    userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
            false, ALLOW_FULL_ONLY, "startActivity", null);
    return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
            resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
            profilerInfo, null, null, options, userId, null, null);
}
```
在startActivity中的UserHandle.getCallingUserId()是通过调用者的userId来取得，但是一般都
是返回（<= 0）的数字，在startActivityAsUser中，通过调用handleIncomingUser会将userId进行
初始化，这里根据应用从launcher启动，分析此方法得知更新后的userId是0，然后ActivityManagerService
将启动Activity的任务交送给Activity栈管理着（ActivityStackSupervisor)。
<!--todo 观察其参数的对应，发现callingUid被传入了-1，没有采用Binder.getCallingUid()（感到奇怪）
-->
在上面resolvedType, resultWho, profilerInfo都是为null的，以及options，我们也可以看作为null。
<!-- 在这里是不是需要改造aInfo中的内容，也就是lib指向的地址 -->
```
final int startActivityMayWait(IApplicationThread caller, int callingUid,
        String callingPackage, Intent intent, String resolvedType,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, WaitResult outResult, Configuration config,
        Bundle options, int userId, IActivityContainer iContainer, TaskRecord inTask) {
        ......
    ActivityContainer container = (ActivityContainer)iContainer; // #=> IContainer is null;
    synchronized (mService) {
        final int realCallingPid = Binder.getCallingPid();
        final int realCallingUid = Binder.getCallingUid();
        int callingPid;
        if (callingUid >= 0) {
            callingPid = -1;
        } else if (caller == null) { // #=> caller is not null
            callingPid = realCallingPid;
            callingUid = realCallingUid;
        } else {
            callingPid = callingUid = -1;
        }
        ......
        int res = startActivityLocked(caller, intent, resolvedType, aInfo,
                voiceSession, voiceInteractor, resultTo, resultWho,
                requestCode, callingPid, callingUid, callingPackage,
                realCallingPid, realCallingUid, startFlags, options,
                componentSpecified, null, container, inTask);
        ......
        return res;
    }
}
```
在startActivityMayWait中发现，启动Activity是一个原子操作，在10和11行中，取得调用者的pid和
uid，这里就是Launcher3的uid和pid，但是在接下的一个if判断中（13-20）中，可以推算出来，这里要求
callingPid和callingUid最后都是为负值（也就是无效值）.接下来解析startActivityLocked
```

final int startActivityLocked(IApplicationThread caller,
        Intent intent, String resolvedType, ActivityInfo aInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode,
        int callingPid, int callingUid, String callingPackage,
        int realCallingPid, int realCallingUid, int startFlags, Bundle options,
        boolean componentSpecified, ActivityRecord[] outActivity, ActivityContainer container,
        TaskRecord inTask) {
    int err = ActivityManager.START_SUCCESS;

    ProcessRecord callerApp = null;
    if (caller != null) {
        callerApp = mService.getRecordForAppLocked(caller);
        if (callerApp != null) {
            callingPid = callerApp.pid;
            callingUid = callerApp.info.uid;
        } else { }
    }

    ActivityRecord sourceRecord = null;
    ActivityRecord resultRecord = null;
    if (resultTo != null) {
        sourceRecord = isInAnyStackLocked(resultTo);
        if (sourceRecord != null) {
            if (requestCode >= 0 && !sourceRecord.finishing) {
                resultRecord = sourceRecord;
            }
        }
    }

    final int launchFlags = intent.getFlags();

    final ActivityStack resultStack = resultRecord == null ? null : resultRecord.task.stack;

    ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,
            intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho,
            requestCode, componentSpecified, this, container, options);
    ......

    err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
            startFlags, true, options, inTask);
    ......
}
```
sourceRecord是根据resultTo得到的，所以sourceRecord是Launcher3的相关信息，然后下面需要构建
启动应用的ActivityRecord,接着分析startActivityUncheckedLocked.
```
final int startActivityUncheckedLocked(ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int startFlags,
        boolean doResume, Bundle options, TaskRecord inTask) {
    // #=> voiceSession = null, voiceInteractor = null, startFlags = 0, doResume = true, options = null, inTask = null
    final Intent intent = r.intent;
    final int callingUid = r.launchedFromUid;

    final boolean launchSingleTop = r.launchMode == ActivityInfo.LAUNCH_SINGLE_TOP;
    final boolean launchSingleInstance = r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE;
    final boolean launchSingleTask = r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK;

    int launchFlags = intent.getFlags();

    final boolean launchTaskBehind = r.mLaunchTaskBehind    // #=> false
            && !launchSingleTask && !launchSingleInstance
            && (launchFlags & Intent.FLAG_ACTIVITY_NEW_DOCUMENT) != 0;

    boolean addingToTask = false;
    TaskRecord reuseTask = null;

    ActivityInfo newTaskInfo = null;
    Intent newTaskIntent = null;
    ActivityStack sourceStack;
    if (sourceRecord != null) {
        if (sourceRecord.finishing) {} else {
            sourceStack = sourceRecord.task.stack;
        }
    } else {}

    ActivityStack targetStack;

    intent.setFlags(launchFlags);

    boolean newTask = false;
    boolean keepCurTransition = false;

    TaskRecord taskToAffiliate = launchTaskBehind && sourceRecord != null ?
            sourceRecord.task : null;   //null

    // Should this be considered a new task?
    if (r.resultTo == null && inTask == null && !addingToTask
            && (launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
        newTask = true;
        targetStack = adjustStackFocus(r, newTask);
        if (!launchTaskBehind) {
            targetStack.moveToFront("startingNewTask");
        }
        if (reuseTask == null) {
            r.setTask(targetStack.createTaskRecord(getNextTaskId(),
                    newTaskInfo != null ? newTaskInfo : r.info,
                    newTaskIntent != null ? newTaskIntent : intent,
                    voiceSession, voiceInteractor, !launchTaskBehind /* toTop */),
                    taskToAffiliate);
        } else {
        }
    }
    mService.grantUriPermissionFromIntentLocked(callingUid, r.packageName,
            intent, r.getUriPermissionsLocked(), r.userId);

    targetStack.mLastPausedActivity = null;
    targetStack.startActivityLocked(r, newTask, doResume, keepCurTransition, options);
    if (!launchTaskBehind) {
        // Don't set focus on an activity that's going to the back.
        mService.setFocusedActivityLocked(r, "startedActivity");
    }
    return ActivityManager.START_SUCCESS;
}
```
从上面跟过来的intent中的启动标志，都没有发生改变，所以都是Launcher3中的代码可知道，存在一个
makeLaunchIntent（），其中构建intent的时候，设置了如下标志位
```
    setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED)
```
所以在此方法中的很多的判断launchFlags的地方可以根据上面的代码来进行忽略、删除，而在当前的startActivityUncheckedLocked
方法最主要的事儿就是将activity设置到前端，获得焦点，检查权限，并将activity的启动交给ActivityStack(targetStack)，
而targetStack是通过adjustStackFocus()方法来获取的，属于一个新建的活动栈，然后通过targetStack
根据ActivityRecord来启动Activity。
```
final void startActivityLocked(ActivityRecord r, boolean newTask,
        boolean doResume, boolean keepCurTransition, Bundle options) {
    // newTask = true; doResume = true; keepCurTransition = false
    TaskRecord rTask = r.task;
    final int taskId = rTask.taskId;
    // mLaunchTaskBehind tasks get placed at the back of the task stack.
    if (!r.mLaunchTaskBehind && (taskForIdLocked(taskId) == null || newTask)) {
        // Last activity in task had been removed or ActivityManagerService is reusing task.
        // Insert or replace.
        // Might not even be in.
        insertTaskAtTop(rTask);
        mWindowManager.moveTaskToTop(taskId);
    }
    TaskRecord task = null;

    // If we are not placing the new activity frontmost, we do not want
    // to deliver the onUserLeaving callback to the actual frontmost
    // activity
    if (task == r.task && mTaskHistory.indexOf(task) != (mTaskHistory.size() - 1)) {
        mStackSupervisor.mUserLeaving = false;
        if (DEBUG_USER_LEAVING) Slog.v(TAG,
                "startActivity() behind front, mUserLeaving=false");
    }

    task = r.task;

    // Slot the activity into the history stack and proceed
    task.addActivityToTop(r);
    task.setFrontOfTask();

    r.putInHistory();
    if (!isHomeStack() || numActivities() > 0) {
        // We want to show the starting preview window if we are
        // switching to a new task, or the next activity's process is
        // not currently running.
        boolean showStartingIcon = newTask;
        ProcessRecord proc = r.app;
        if (proc == null) {
            proc = mService.mProcessNames.get(r.processName, r.info.applicationInfo.uid);
        }
        if (proc == null || proc.thread == null) {
            showStartingIcon = true;
        }
        if (DEBUG_TRANSITION) Slog.v(TAG,
                "Prepare open transition: starting " + r);
        if ((r.intent.getFlags()&Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0) {
            mWindowManager.prepareAppTransition(AppTransition.TRANSIT_NONE, keepCurTransition);
            mNoAnimActivities.add(r);
        } else {
            mWindowManager.prepareAppTransition(newTask
                    ? r.mLaunchTaskBehind
                            ? AppTransition.TRANSIT_TASK_OPEN_BEHIND
                            : AppTransition.TRANSIT_TASK_OPEN
                    : AppTransition.TRANSIT_ACTIVITY_OPEN, keepCurTransition);
            mNoAnimActivities.remove(r);
        }
        mWindowManager.addAppToken(task.mActivities.indexOf(r),
                r.appToken, r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
                (r.info.flags & ActivityInfo.FLAG_SHOW_ON_LOCK_SCREEN) != 0, r.userId,
                r.info.configChanges, task.voiceSession != null, r.mLaunchTaskBehind);
        boolean doShow = true;
        if (newTask) {
            // Even though this activity is starting fresh, we still need
            // to reset it to make sure we apply affinities to move any
            // existing activities from other tasks in to it.
            // If the caller has requested that the target task be
            // reset, then do so.
            if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                resetTaskIfNeededLocked(r, r);
                doShow = topRunningNonDelayedActivityLocked(null) == r;
            }
        } else if (options != null && new ActivityOptions(options).getAnimationType()
                == ActivityOptions.ANIM_SCENE_TRANSITION) {
            doShow = false;
        }
        if (r.mLaunchTaskBehind) {
            // Don't do a starting window for mLaunchTaskBehind. More importantly make sure we
            // tell WindowManager that r is visible even though it is at the back of the stack.
            mWindowManager.setAppVisibility(r.appToken, true);
            ensureActivitiesVisibleLocked(null, 0);
        } else if (SHOW_APP_STARTING_PREVIEW && doShow) {
            // Figure out if we are transitioning from another activity that is
            // "has the same starting icon" as the next one.  This allows the
            // window manager to keep the previous window it had previously
            // created, if it still had one.
            ActivityRecord prev = mResumedActivity;
            if (prev != null) {
                // We don't want to reuse the previous starting preview if:
                // (1) The current activity is in a different task.
                if (prev.task != r.task) {
                    prev = null;
                }
                // (2) The current activity is already displayed.
                else if (prev.nowVisible) {
                    prev = null;
                }
            }
            mWindowManager.setAppStartingWindow(
                    r.appToken, r.packageName, r.theme,
                    mService.compatibilityInfoForPackageLocked(
                            r.info.applicationInfo), r.nonLocalizedLabel,
                    r.labelRes, r.icon, r.logo, r.windowFlags,
                    prev != null ? prev.appToken : null, showStartingIcon);
            r.mStartingWindowShown = true;
        }
    } else {......}

    if (doResume) {
        mStackSupervisor.resumeTopActivitiesLocked(this, r, options);
    }
}
```
<!-- 未剖析 -->




























在startActivityUncheckedLocked中，会通过判断startFlags来判断Activity的启动方式，也就是前面在为intent置下FLAG_ACTIVITY_NEW_TASK
```
final int startActivityUncheckedLocked(ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int startFlags,
        boolean doResume, Bundle options, TaskRecord inTask)
```
这里通过doResume来进行判断是否调用resumeTopActivityLocked()
```
final boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {
    if (!mService.mBooting && !mService.mBooted) {
        // Not ready yet!
        return false;
    }

    ActivityStack lastStack = mStackSupervisor.getLastStack();
    ...
            next.app.thread.scheduleResumeActivity(next.appToken, next.app.repProcState,
                    mService.isNextTransitionForward(), resumeAnimOptions);

}
```
在resumeTopActivityInnerLocked中，我们需要重新启动一个新ApplicationThread来执行启动应用
的线程，而scheduleResumeActivity又是属于ApplicationThread的方法，而ApplicationThread的
父类ApplicationThreadNative有时IApplicationThread的子类，然而这里又通过了发送Handler消
息来进行后续的操作，其代码如下：
```
// #=file=> frameworks/base/core/java/android/app/ActivityThread.java
// #=class=> private class ApplicationThread extends ApplicationThreadNative

public final void scheduleResumeActivity(IBinder token, int processState,
        boolean isForward, Bundle resumeArgs) {
    updateProcessState(processState, false);
    sendMessage(H.RESUME_ACTIVITY, token, isForward ? 1 : 0);
}

// #=file=> frameworks/base/core/java/android/app/ApplicationThreadNative.java
// #=class=> class ApplicationThreadProxy implements IApplicationThread {

public final void scheduleResumeActivity(IBinder token, int procState, boolean isForward,
        Bundle resumeArgs)
        throws RemoteException {
    Parcel data = Parcel.obtain();
    data.writeInterfaceToken(IApplicationThread.descriptor);
    data.writeStrongBinder(token);
    data.writeInt(procState);
    data.writeInt(isForward ? 1 : 0);
    data.writeBundle(resumeArgs);
    mRemote.transact(SCHEDULE_RESUME_ACTIVITY_TRANSACTION, data, null,
            IBinder.FLAG_ONEWAY);
    data.recycle();
}
```
有了sendMessage后，就一定有一个handleMessage(),而在这里面，我们只需要关心一点儿就是switch中
的RESUME_ACTIVITY条件下的东西。
```
case RESUME_ACTIVITY:
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityResume");
    handleResumeActivity((IBinder) msg.obj, true, msg.arg1 != 0, true);
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    break;
```
这里我们就看到了一个Binder对象了，这里就会接收ApplicationThreadProxy中scheduleResumeActivity
发送的消息。

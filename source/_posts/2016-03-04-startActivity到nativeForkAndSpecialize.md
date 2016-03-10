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
    } else if (tag instanceof FolderInfo) {
    ......
    }
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
    } else {
        ......
    }

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
    } catch (ActivityNotFoundException e) {
        ......
    }
    return success;
}
```
这里
```
boolean startActivity(View v, Intent intent, Object tag) {
    Log.e(TAG + "lingyang", "for test");
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    try {
        // Only launch using the new animation if the shortcut has not opted out (this is a
        // private contract between launcher and may be ignored in the future).
        boolean useLaunchAnimation = (v != null) &&
                !intent.hasExtra(INTENT_EXTRA_IGNORE_LAUNCH_ANIMATION);
        LauncherAppsCompat launcherApps = LauncherAppsCompat.getInstance(this);
        UserManagerCompat userManager = UserManagerCompat.getInstance(this);

        UserHandleCompat user = null;
        if (intent.hasExtra(AppInfo.EXTRA_PROFILE)) {
            long serialNumber = intent.getLongExtra(AppInfo.EXTRA_PROFILE, -1);
            user = userManager.getUserForSerialNumber(serialNumber);
        }

        Bundle optsBundle = null;
        if (useLaunchAnimation) {
            ActivityOptions opts = Utilities.isLmpOrAbove() ?
                    ActivityOptions.makeCustomAnimation(this, R.anim.task_open_enter, R.anim.no_anim) :
                    ActivityOptions.makeScaleUpAnimation(v, 0, 0, v.getMeasuredWidth(), v.getMeasuredHeight());
            optsBundle = opts.toBundle();
        }

        if (user == null || user.equals(UserHandleCompat.myUserHandle())) {
            // Could be launching some bookkeeping activity
            startActivity(intent, optsBundle);
        } else {
            // TODO Component can be null when shortcuts are supported for secondary user
            launcherApps.startActivityForProfile(intent.getComponent(), user,
                    intent.getSourceBounds(), optsBundle);
        }
        return true;
    } catch (SecurityException e) {
        Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
        Log.e(TAG, "Launcher does not have the permission to launch " + intent +
                ". Make sure to create a MAIN intent-filter for the corresponding activity " +
                "or use the exported attribute for this activity. "
                + "tag="+ tag + " intent=" + intent, e);
    }
    return false;
}
```











当我们在手机桌面点击了应用图标的时候，这时候就会触发Laucher.java 中的 onClick(View view) 方
法，然后构建intent，利用Launcher.java中封装的startActivitySafely()来启动应用。

Launcher为了我们启动应用的intent添加了标志:FLAG_ACTIVITY_NEW_TASK，表示启动一个应用的时候，重新建立一个任务（Android中任务应该就是我们所说的进程）

Launcher.java对Activity的startActivity的封装中，我们只需要关心两点：
- 有动画的startActivity，这种方式调用的是startActivity(Intent intent, Bundle options);
- 无动画的startActivity，这种方式调用的是startActivity(Intent intent),最后也调用到了上一种startActivity，只是options = null;


### Activity.java 启动应用分析
现在接收到了Launcher的startActivity输入，接着调用startActivityForResult，这里的requestCode = -1，表示不要求返回结果，且这里选择options = null进行分析，

```
public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
    if (mParent == null) {  // #=> 从桌面启动，所以mParent不为null
        ...
    } else {
        if (options != null) {
            mParent.startActivityFromChild(this, intent, requestCode, options);
        } else {
            mParent.startActivityFromChild(this, intent, requestCode);
        }
    }
    if (options != null && !isTopOfTask()) {
        mActivityTransitionState.startExitOutTransition(this, options);
    }
}
```
因为是从Launcher中启动，且options = null，所以我们只看 mParent.startActivityFromChild(this, intent, requestCode)
在startActivityFromChild中，其构造是这样的，
```
public void startActivityFromChild(@NonNull Activity child, Intent intent,
        int requestCode, @Nullable Bundle options) {
    Instrumentation.ActivityResult ar =
        mInstrumentation.execStartActivity(
            this, mMainThread.getApplicationThread(), mToken, child,
            intent, requestCode, options);
    if (ar != null) {
        mMainThread.sendActivityResult(
            mToken, child.mEmbeddedID, requestCode,
            ar.getResultCode(), ar.getResultData());
    }
}
```
这里ar = null, 所以这里的sendActivityResult是不会执行的。通过下面的代码，来看ar = null
```
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target, // #=> who = target
        Intent intent, int requestCode, Bundle options) {   // #=>intent, -1, null
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    if (mActivityMonitors != null) {
        synchronized (mSync) {
            final int N = mActivityMonitors.size();  //#=> 在桌面启动应用中，mActivityMonitors.size = 0;
            for (int i=0; i<N; i++) {
                final ActivityMonitor am = mActivityMonitors.get(i);
                if (am.match(who, null, intent)) {
                    am.mHits++;
                    if (am.isBlocking()) {  // #=> always true;
                        return requestCode >= 0 ? am.getResult() : null;
                    }
                    break;
                }
            }
        }
    }
    try {
        ...
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
从上面的代码中，我么就可以看出ar ！= null的判断是由mActivityMonitors有关联的，然而mActivityMonitors中的元素，
却是在addMonitor()中进行添加的，但是addMonitor却没有实质的调点，所以返回execStartActivity
返回null。

ActivityManagerNative.getDefault()取得全局IActivityManager对象,这个对象是一个ActivityManagerProxy对象，我们需
要这个Binder对象调用ActivityManagerService中的startActivity。
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
    // TODO: Switch to user app stacks here.
    return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
            resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
            profilerInfo, null, null, options, userId, null, null);
}
```
在startActivityAsUser中，可以看出activity的调用管理现在走到了栈（activity栈）的管理处去了

```
final int startActivityMayWait(IApplicationThread caller, int callingUid,
        String callingPackage, Intent intent, String resolvedType,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, WaitResult outResult, Configuration config,
        Bundle options, int userId, IActivityContainer iContainer, TaskRecord inTask) {
    /*
    return mStackSupervisor.startActivityMayWait(caller, -1,
                            callingPackage, intent, resolvedType,
                            null, null,
                            resultTo, resultWho, -1, 0,
            null, null, null, null, userId, null, null);*/

    // Collect information about the target of the Intent.
    ActivityInfo aInfo = resolveActivity(intent, resolvedType, startFlags,
            profilerInfo, userId);

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

        final ActivityStack stack;
        if (container == null || container.mStack.isOnHomeDisplay()) { // #=> container is null
            stack = getFocusedStack();
        } else {
            stack = container.mStack;
        }
        ...
        int res = startActivityLocked(caller, intent, resolvedType, aInfo,
                voiceSession, voiceInteractor, resultTo, resultWho,
                requestCode, callingPid, callingUid, callingPackage,
                realCallingPid, realCallingUid, startFlags, options,
                componentSpecified, null, container, inTask);

        Binder.restoreCallingIdentity(origId);

        if (outResult != null) {    // #=> outResult is null;
            ...
        }
        return res;
    }
}
final int startActivityLocked(IApplicationThread caller,
        Intent intent, String resolvedType, ActivityInfo aInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode,
        int callingPid, int callingUid, String callingPackage,
        int realCallingPid, int realCallingUid, int startFlags, Bundle options,
        boolean componentSpecified, ActivityRecord[] outActivity, ActivityContainer container,
        TaskRecord inTask)  {
    /* #=>
    startActivityLocked(caller, intent, resolvedType, aInfo,
            null, null, resultTo, resultWho,
            -1, -1, -1, callingPackage,
            realCallingPid, realCallingUid, 0, null,
            true, null, null, null);
    *                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         m                                                                                                                                                                                                                                                                                                                                                                                                                                   /
    int err = ActivityManager.START_SUCCESS;

    ...
    err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
            startFlags, true, options, inTask);

    ...
    return err;
}

```
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

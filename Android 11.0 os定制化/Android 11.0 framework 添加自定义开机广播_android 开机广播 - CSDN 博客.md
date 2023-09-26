> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126393713)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0%C2%A0) 

[2.framework 添加自定义开机广播的核心代码](#t1)

[3.framework 添加自定义开机广播的功能分析以及实现  3.1 ActivityThread.java 关于开机的相关代码](#t2)

[3.2 ActivityTaskManagerService.activityIdle() 的分析](#3.2%20ActivityTaskManagerService.activityIdle%28%29%E7%9A%84%E5%88%86%E6%9E%90)

[3.3 ActivityStackSupervisor 开机广播流程处理](#t4)

[3.4 UserController.java 相关开机广播流程](#t5)

1. 概述
-----

在进行系统定制化开发中，在内置一些 app 需要收到开机广播以后然后做一些相关的操作的功能的时候，发现开机广播要好久能收到，要么就收不到开机广播，所以这就需要了解开机广播在哪里发送，然后自定义开机广播来接收自定义开机广播然后开发一些功能

2.framework 添加自定义开机广播的核心代码
--------------------------

```
/frameworks/base/core/java/android/app/ActivityThread.java
   /frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
  /frameworks/base/services/core/java/com/android/server/wm/ActivityStackSupervisor.java
  /frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
 /frameworks/base/services/core/java/com/android/server/am/UserController.java
```

### 3.framework 添加自定义开机广播的功能分析以及实现  
  3.1 ActivityThread.java 关于开机的相关代码

```
public void handleResumeActivity(IBinder token, Boolean finalStateRequest, Boolean isForward,
String reason) {
	// If we are getting ready to gc after going to the background, well
	// we are back active so skip it.
	unscheduleGcIdler();
	mSomeActivitiesChanged = true;
	// TODO Push resumeArgs into the activity for consideration
	final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
	if (r == null) {
		// We didn't actually resume the activity, so skipping any follow-up actions.
		return;
	}
	if (mActivitiesToBeDestroyed.containsKey(token)) {
		// Although the activity is resumed, it is going to be destroyed. So the following
		// UI operations are unnecessary and also prevents exception because its token may
		// be gone that window manager cannot recognize it. All necessary cleanup actions
		// performed below will be done while handling destruction.
		return;
	}
	final Activity a = r.activity;
	if (localLOGV) {
		Slog.v(TAG, "Resume " + r + " started activity: " + a.mStartedActivity
		+ ", hideForNow: " + r.hideForNow + ", finished: " + a.mFinished);
	}
	final int forwardBit = isForward
	? WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;
	// If the window hasn't yet been added to the window manager,
	// and this guy didn't finish itself or start another activity,
	// then go ahead and add the window.
	Boolean willBeVisible = !a.mStartedActivity;
	if (!willBeVisible) {
		try {
			willBeVisible = ActivityTaskManager.getService().willActivityBeVisible(
			a.getActivityToken());
		}
		catch (RemoteException e) {
			throw e.rethrowFromSystemServer();
		}
	}
	if (r.window == null && !a.mFinished && willBeVisible) {
		r.window = r.activity.getWindow();
		View decor = r.window.getDecorView();
		decor.setVisibility(View.INVISIBLE);
		ViewManager wm = a.getWindowManager();
		WindowManager.LayoutParams l = r.window.getAttributes();
		a.mDecor = decor;
		l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
		l.softInputMode |= forwardBit;
		if (r.mPreserveWindow) {
			a.mWindowAdded = true;
			r.mPreserveWindow = false;
			// Normally the ViewRoot sets up callbacks with the Activity
			// in addView->ViewRootImpl#setView. If we are instead reusing
			// the decor view we have to notify the view root that the
			// callbacks may have changed.
			ViewRootImpl impl = decor.getViewRootImpl();
			if (impl != null) {
				impl.notifyChildRebuilt();
			}
		}
		if (a.mVisibleFromClient) {
			if (!a.mWindowAdded) {
				a.mWindowAdded = true;
				wm.addView(decor, l);
			} else {
				// The activity will get a callback for this {@link LayoutParams} change
				// earlier. However, at that time the decor will not be set (this is set
				// in this method), so no action will be taken. This call ensures the
				// callback occurs with the decor set.
				a.onWindowAttributesChanged(l);
			}
		}
		// If the window has already been added, but during resume
		// we started another activity, then don't yet make the
		// window visible.
	} else if (!willBeVisible) {
		if (localLOGV) Slog.v(TAG, "Launch " + r + " mStartedActivity set");
		r.hideForNow = true;
	}
	// Get rid of anything left hanging around.
	cleanUpPendingRemoveWindows(r, false 
	/* force */
	);
	// The window is now visible if it has been added, we are not
	// simply finishing, and we are not starting another activity.
	if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
		if (r.newConfig != null) {
			performConfigurationChangedForActivity(r, r.newConfig);
			if (DEBUG_CONFIGURATION) {
				Slog.v(TAG, "Resuming activity " + r.activityInfo.name + " with newConfig "
				+ r.activity.mCurrentConfig);
			}
			r.newConfig = null;
		}
		if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward=" + isForward);
		ViewRootImpl impl = r.window.getDecorView().getViewRootImpl();
		WindowManager.LayoutParams l = impl != null
		? impl.mWindowAttributes : r.window.getAttributes();
		if ((l.softInputMode
		& WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
		!= forwardBit) {
			l.softInputMode = (l.softInputMode
			& (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
			| forwardBit;
			if (r.activity.mVisibleFromClient) {
				ViewManager wm = a.getWindowManager();
				View decor = r.window.getDecorView();
				wm.updateViewLayout(decor, l);
			}
		}
		r.activity.mVisibleFromServer = true;
		mNumVisibleActivities++;
		if (r.activity.mVisibleFromClient) {
			r.activity.makeVisible();
		}
	}
	r.nextIdle = mNewActivities;
	mNewActivities = r;
	if (localLOGV) Slog.v(TAG, "Scheduling idle handler for " + r);
	Looper.myQueue().addIdleHandler(new Idler());
}
ActivityThread.handleResumeActivity()将launcher界面显示出来，通过Looper.myQueue().addIdleHandler(new Idler())往looper中添加了一条空闲消息，
这个消息主要通知ActivityTaskManagerService当前Activity空闲了，这个时候就可以去做一些其他的事情，比如执行上一个Activity的onStop（）、onDestroy，
还有这里要说到的开机广播，接着就来看看ActivityTaskManagerService.activityIdle()
```

### 3.2 ActivityTaskManagerService.activityIdle() 的分析

```
public final void activityIdle(IBinder token, Configuration config, Boolean stopProfiling) {
	final long origId = Binder.clearCallingIdentity();
	try {
		synchronized (mGlobalLock) {
			Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "activityIdle");
			final ActivityRecord r = ActivityRecord.forTokenLocked(token);
			if (r == null) {
				return;
			}
			mStackSupervisor.activityIdleInternal(r, false 
			/* fromTimeout */
			,
			false 
			/* processPausingActivities */
			, config);
			if (stopProfiling && r.hasProcess()) {
				r.app.clearProfilerIfNeeded();
			}
		}
	}
	finally {
		Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
		Binder.restoreCallingIdentity(origId);
	}
}
 
主要工作将处理转交给ActivityStackSupervisor.activityIdleInternal()来处理了
 
```

### 3.3 ActivityStackSupervisor 开机广播流程处理

```
private Boolean checkFinishBootingLocked() {
	final Boolean booting = mService.isBooting();
	Boolean enableScreen = false;
	mService.setBooting(false);
	if (!mService.isBooted()) {
		mService.setBooted(true);
		enableScreen = true;
	}
	if (booting || enableScreen) {
		mService.postFinishBooting(booting, enableScreen);
	}
	return booting;
}
void activityIdleInternal(ActivityRecord r, Boolean fromTimeout,
Boolean processPausingActivities, Configuration config) {
	if (DEBUG_ALL) Slog.v(TAG, "Activity idle: " + r);
	Boolean booting = false;
	if (r != null) {
		if (DEBUG_IDLE) Slog.d(TAG_IDLE, "activityIdleInternal: Callers="
		+ Debug.getCallers(4));
		mHandler.removeMessages(IDLE_TIMEOUT_MSG, r);
		r.finishLaunchTickingLocked();
		if (fromTimeout) {
			reportActivityLaunchedLocked(fromTimeout, r, INVALID_DELAY,
			-1 
			/* launchState */
			);
		}
		// This is a hack to semi-deal with a race condition
		// in the client where it can be constructed with a
		// newer configuration from when we asked it to launch.
		// We'll update with whatever configuration it now says
		// it used to launch.
		if (config != null) {
			r.setLastReportedGlobalConfiguration(config);
		}
		// We are now idle.  If someone is waiting for a thumbnail from
		// us, we can now deliver.
		r.idle = true;
		//Slog.i(TAG, "IDLE: mBooted=" + mBooted + ", fromTimeout=" + fromTimeout);
		// Check if able to finish booting when device is booting and all resumed activities
		// are idle.
		if ((mService.isBooting() && mRootWindowContainer.allResumedActivitiesIdle())
		|| fromTimeout) {
			booting = checkFinishBootingLocked();
		}
		// When activity is idle, we consider the relaunch must be successful, so let's clear
		// the flag.
		r.mRelaunchReason = RELAUNCH_REASON_NONE;
	}
	if (mRootWindowContainer.allResumedActivitiesIdle()) {
		if (r != null) {
			mService.scheduleAppGcsLocked();
		}
		if (mLaunchingActivityWakeLock.isHeld()) {
			mHandler.removeMessages(LAUNCH_TIMEOUT_MSG);
			if (VALIDATE_WAKE_LOCK_CALLER &&
			Binder.getCallingUid() != Process.myUid()) {
				throw new IllegalStateException("Calling must be system uid");
			}
			mLaunchingActivityWakeLock.release();
		}
		mRootWindowContainer.ensureActivitiesVisible(null, 0, !PRESERVE_WINDOWS);
	}
	// Atomically retrieve all of the other things to do.
	processStoppingAndFinishingActivities(r, processPausingActivities, "idle");
	if (!mStartingUsers.isEmpty()) {
		final ArrayList<UserState> startingUsers = new ArrayList<>(mStartingUsers);
		mStartingUsers.clear();
		if (!booting) {
			// Complete user switch.
			for (int i = 0; i < startingUsers.size(); i++) {
				mService.mAmInternal.finishUserSwitch(startingUsers.get(i));
			}
		}
	}
	mService.mH.post(() -> mService.mAmInternal.trimApplications());
}
最终通过checkFinishBootingLocked()的ActivityTaskManagerService.postFinishBooting()来继续
执行开机广播流程
 
void postFinishBooting(Boolean finishBooting, Boolean enableScreen) {
	mH.post(() -> {
		if (finishBooting) {
                                               //这里会去执行ActivityManagerService的finishBooting(）
			mAmInternal.finishBooting();
		}
		if (enableScreen) {
                                           //这里中间通过转发下，会执行到WindowManagerService的enableScreenAfterBoot()
			mInternal.enableScreenAfterBoot(isBooted());
		}
	}
	);
}
 
这里先来看下WindowManagerService.finishBooting()
public void enableScreenAfterBoot() {
        synchronized (mGlobalLock) {
            ... ...
            if (mSystemBooted) {
                return;
            }
            mSystemBooted = true;
            ... ...
        }
        performEnableScreen();
    }
 
    private void performEnableScreen() {
        ... ...
        try {
            mActivityManager.bootAnimationComplete();
        } catch (RemoteException e) {
        }
        ... ...
    }
enableScreenAfterBoot()只会执行一次，但是performEnableScreen()可能会执行多次，
内部会每隔200ms去查询动画是否完成，如果完成了，就会执行ActivityManagerService.bootAnimationComplete()
最终调用
ActivityManagerService.finishBooting()：
 
    final void finishBooting() {
        ... ...
        mUserController.sendBootCompleted(... ...);
        ... ...
    }
```

### 3.4 UserController.java 相关开机广播流程

```
void sendBootCompleted(IIntentReceiver resultTo) {
	final Boolean systemUserFinishedBooting;
	// Get a copy of mStartedUsers to use outside of lock
	SparseArray<UserState> startedUsers;
	synchronized (mLock) {
		systemUserFinishedBooting = mCurrentUserId != UserHandle.USER_SYSTEM;
		startedUsers = mStartedUsers.clone();
	}
	for (int i = 0; i < startedUsers.size(); i++) {
		UserState uss = startedUsers.valueAt(i);
		if (systemUserFinishedBooting && uss.mHandle.isSystem()) {
			// On Automotive, at this point the system user has already been started and
			// unlocked, and some of the tasks we do here have already been done. So skip those
			// in that case.
			// TODO(b/132262830): this workdound shouldn't be necessary once we move the
			// headless-user start logic to UserManager-land
			Slog.d(TAG, "sendBootCompleted(): skipping on non-current system user");
			continue;
		}
		finishUserBoot(uss, resultTo);
	}
}
private void finishUserUnlockedCompleted(UserState uss) {
	final int userId = uss.mHandle.getIdentifier();
	EventLog.writeEvent(EventLogTags.UC_FINISH_USER_UNLOCKED_COMPLETED, userId);
	synchronized (mLock) {
		// Bail if we ended up with a stale user
		if (mStartedUsers.get(uss.mHandle.getIdentifier()) != uss) return;
	}
	UserInfo userInfo = getUserInfo(userId);
	if (userInfo == null) {
		return;
	}
	// Only keep marching forward if user is actually unlocked
	if (!StorageManager.isUserKeyUnlocked(userId)) return;
	// Remember that we logged in
	mInjector.getUserManager().onUserLoggedIn(userId);
	if (!userInfo.isInitialized()) {
		if (userId != UserHandle.USER_SYSTEM) {
			Slog.d(TAG, "Initializing user #" + userId);
			Intent intent = new Intent(Intent.ACTION_USER_INITIALIZE);
			intent.addFlags(Intent.FLAG_RECEIVER_FOREGROUND
			| Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND);
			mInjector.broadcastIntent(intent, null,
			new IIntentReceiver.Stub() {
				@Override
				public void performReceive(Intent intent, int resultCode,
				String data, Bundle extras, Boolean ordered,
				Boolean sticky, int sendingUser) {
					// Note: performReceive is called with mService lock held
					mInjector.getUserManager().makeInitialized(userInfo.id);
				}
			}
			, 0, null, null, null, AppOpsManager.OP_NONE,
			null, true, false, MY_PID, SYSTEM_UID, Binder.getCallingUid(),
			Binder.getCallingPid(), userId);
		}
	}
	if (userInfo.preCreated) {
		Slog.i(TAG, "Stopping pre-created user " + userInfo.toFullString());
		// Pre-created user was started right after creation so services could properly
		// intialize it; it should be stopped right away as it's not really a "real" user.
		// TODO(b/143092698): in the long-term, it might be better to add a onCreateUser()
		// callback on SystemService instead.
		stopUser(userInfo.id, 
		/* force= */
		true, 
		/* allowDelayedLocking= */
		false,
		/* stopUserCallback= */
		null, 
		/* keyEvictedCallback= */
		null);
		return;
	}
	// Spin up app widgets prior to boot-complete, so they can be ready promptly
	mInjector.startUserWidgets(userId);
	mHandler.obtainMessage(USER_UNLOCKED_MSG, userId, 0).sendToTarget();
	Slog.i(TAG, "Posting BOOT_COMPLETED user #" + userId);
	// Do not report secondary users, runtime restarts or first boot/upgrade
	if (userId == UserHandle.USER_SYSTEM
	&& !mInjector.isRuntimeRestarted() && !mInjector.isFirstBootOrUpgrade()) {
		final long elapsedTimeMs = SystemClock.elapsedRealtime();
		FrameworkStatsLog.write(FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME_REPORTED,
		FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME__EVENT__FRAMEWORK_BOOT_COMPLETED,
		elapsedTimeMs);
	}
	final Intent bootIntent = new Intent(Intent.ACTION_BOOT_COMPLETED, null);
	bootIntent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
	bootIntent.addFlags(Intent.FLAG_RECEIVER_NO_ABORT
	| Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND
	| Intent.FLAG_RECEIVER_OFFLOAD);
	// Widget broadcasts are outbound via FgThread, so to guarantee sequencing
	// we also send the boot_completed broadcast from that thread.
	final int callingUid = Binder.getCallingUid();
	final int callingPid = Binder.getCallingPid();
	FgThread.getHandler().post(() -> {
		mInjector.broadcastIntent(bootIntent, null,
		new IIntentReceiver.Stub() {
			@Override
			public void performReceive(Intent intent, int resultCode, String data,
			Bundle extras, Boolean ordered, Boolean sticky, int sendingUser)
			throws RemoteException {
				Slog.i(UserController.TAG, "Finished processing BOOT_COMPLETED for u"
				+ userId);
				mBootCompleted = true;
			}
		}
		, 0, null, null,
		new String[]{
			android.Manifest.permission.RECEIVE_BOOT_COMPLETED
		}
		,
		AppOpsManager.OP_NONE, null, true, false, MY_PID, SYSTEM_UID,
		callingUid, callingPid, userId);
	}
	);
}
所以具体添加自定义开机广播为：
 
private void finishUserUnlockedCompleted(UserState uss) {
	final int userId = uss.mHandle.getIdentifier();
	EventLog.writeEvent(EventLogTags.UC_FINISH_USER_UNLOCKED_COMPLETED, userId);
	synchronized (mLock) {
		// Bail if we ended up with a stale user
		if (mStartedUsers.get(uss.mHandle.getIdentifier()) != uss) return;
	}
	UserInfo userInfo = getUserInfo(userId);
	if (userInfo == null) {
		return;
	}
	// Only keep marching forward if user is actually unlocked
	if (!StorageManager.isUserKeyUnlocked(userId)) return;
	// Remember that we logged in
	mInjector.getUserManager().onUserLoggedIn(userId);
	if (!userInfo.isInitialized()) {
		if (userId != UserHandle.USER_SYSTEM) {
			Slog.d(TAG, "Initializing user #" + userId);
			Intent intent = new Intent(Intent.ACTION_USER_INITIALIZE);
			intent.addFlags(Intent.FLAG_RECEIVER_FOREGROUND
			| Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND);
			mInjector.broadcastIntent(intent, null,
			new IIntentReceiver.Stub() {
				@Override
				public void performReceive(Intent intent, int resultCode,
				String data, Bundle extras, Boolean ordered,
				Boolean sticky, int sendingUser) {
					// Note: performReceive is called with mService lock held
					mInjector.getUserManager().makeInitialized(userInfo.id);
				}
			}
			, 0, null, null, null, AppOpsManager.OP_NONE,
			null, true, false, MY_PID, SYSTEM_UID, Binder.getCallingUid(),
			Binder.getCallingPid(), userId);
		}
	}
	if (userInfo.preCreated) {
		Slog.i(TAG, "Stopping pre-created user " + userInfo.toFullString());
		// Pre-created user was started right after creation so services could properly
		// intialize it; it should be stopped right away as it's not really a "real" user.
		// TODO(b/143092698): in the long-term, it might be better to add a onCreateUser()
		// callback on SystemService instead.
		stopUser(userInfo.id, 
		/* force= */
		true, 
		/* allowDelayedLocking= */
		false,
		/* stopUserCallback= */
		null, 
		/* keyEvictedCallback= */
		null);
		return;
	}
	// Spin up app widgets prior to boot-complete, so they can be ready promptly
	mInjector.startUserWidgets(userId);
	mHandler.obtainMessage(USER_UNLOCKED_MSG, userId, 0).sendToTarget();
	Slog.i(TAG, "Posting BOOT_COMPLETED user #" + userId);
	// Do not report secondary users, runtime restarts or first boot/upgrade
	if (userId == UserHandle.USER_SYSTEM
	&& !mInjector.isRuntimeRestarted() && !mInjector.isFirstBootOrUpgrade()) {
		final long elapsedTimeMs = SystemClock.elapsedRealtime();
		FrameworkStatsLog.write(FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME_REPORTED,
		FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME__EVENT__FRAMEWORK_BOOT_COMPLETED,
		elapsedTimeMs);
	}
 
 //add cord start 
  final Intent custombootIntent = new Intent("com.custom.intent.action.BOOT_COMPLETED", null);
  custombootIntent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
  custombootIntent.addFlags(Intent.FLAG_RECEIVER_NO_ABORT
              | Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND
              | Intent.FLAG_RECEIVER_OFFLOAD);
    final int callinguid = Binder.getCallingUid();
    final int callingpid = Binder.getCallingPid();
    FgThread.getHandler().post(() -> {
        mInjector.broadcastIntentcustombootIntent,null,null, 0, null, null,
                null,
                AppOpsManager.OP_NONE, null, true, false, MY_PID, SYSTEM_UID,
                callinguid, callingpid, userId);
    }); 
   //add code end
 
	final Intent bootIntent = new Intent(Intent.ACTION_BOOT_COMPLETED, null);
	bootIntent.putExtra(Intent.EXTRA_USER_HANDLE, userId);
	bootIntent.addFlags(Intent.FLAG_RECEIVER_NO_ABORT
	| Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND
	| Intent.FLAG_RECEIVER_OFFLOAD);
	// Widget broadcasts are outbound via FgThread, so to guarantee sequencing
	// we also send the boot_completed broadcast from that thread.
	final int callingUid = Binder.getCallingUid();
	final int callingPid = Binder.getCallingPid();
	FgThread.getHandler().post(() -> {
		mInjector.broadcastIntent(bootIntent, null,
		new IIntentReceiver.Stub() {
			@Override
			public void performReceive(Intent intent, int resultCode, String data,
			Bundle extras, Boolean ordered, Boolean sticky, int sendingUser)
			throws RemoteException {
				Slog.i(UserController.TAG, "Finished processing BOOT_COMPLETED for u"
				+ userId);
				mBootCompleted = true;
			}
		}
		, 0, null, null,
		new String[]{
			android.Manifest.permission.RECEIVE_BOOT_COMPLETED
		}
		,
		AppOpsManager.OP_NONE, null, true, false, MY_PID, SYSTEM_UID,
		callingUid, callingPid, userId);
	}
	);
}
 
2.关于去掉自定义广播的Sending non-protected broadcast的抛出异常和日志打印
虽然可以发送自定义广播，但是不符合系统设计会抛出不安全异常 所以要去掉检查自定义广播
private void checkBroadcastFromSystem(Intent intent, ProcessRecord callerApp,
String callerPackage, int callingUid, Boolean isProtectedBroadcast, List receivers) {
	if ((intent.getFlags() & Intent.FLAG_RECEIVER_FROM_SHELL) != 0) {
		// Don't yell about broadcasts sent via shell
		return;
	}
	final String action = intent.getAction();
	if (isProtectedBroadcast
	|| Intent.ACTION_CLOSE_SYSTEM_DIALOGS.equals(action)
	|| Intent.ACTION_DISMISS_KEYBOARD_SHORTCUTS.equals(action)
	|| Intent.ACTION_MEDIA_BUTTON.equals(action)
	|| Intent.ACTION_MEDIA_SCANNER_SCAN_FILE.equals(action)
	|| Intent.ACTION_SHOW_KEYBOARD_SHORTCUTS.equals(action)
	|| Intent.ACTION_MASTER_CLEAR.equals(action)
	|| Intent.ACTION_FACTORY_RESET.equals(action)
	|| AppWidgetManager.ACTION_APPWIDGET_CONFIGURE.equals(action)
	|| AppWidgetManager.ACTION_APPWIDGET_UPDATE.equals(action)
	|| LocationManager.HIGH_POWER_REQUEST_CHANGE_ACTION.equals(action)
	|| TelephonyManager.ACTION_REQUEST_OMADM_CONFIGURATION_UPDATE.equals(action)
	|| SuggestionSpan.ACTION_SUGGESTION_PICKED.equals(action)
	|| AudioEffect.ACTION_OPEN_AUDIO_EFFECT_CONTROL_SESSION.equals(action)
	|| AudioEffect.ACTION_CLOSE_AUDIO_EFFECT_CONTROL_SESSION.equals(action)) {
		// Broadcast is either protected, or it's a public action that
		// we've relaxed, so it's fine for system internals to send.
		return;
	}
	// This broadcast may be a problem...  but there are often system components that
	// want to send an internal broadcast to themselves, which is annoying to have to
	// explicitly list each action as a protected broadcast, so we will check for that
	// one safe case and allow it: an explicit broadcast, only being received by something
	// that has protected itself.
	if (intent.getPackage() != null || intent.getComponent() != null) {
		if (receivers == null || receivers.size() == 0) {
			// Intent is explicit and there's no receivers.
			// This happens, e.g. , when a system component sends a broadcast to
			// its own runtime receiver, and there's no manifest receivers for it,
			// because this method is called twice for each broadcast,
			// for runtime receivers and manifest receivers and the later check would find
			// no receivers.
			return;
		}
		Boolean allProtected = true;
		for (int i = receivers.size()-1; i >= 0; i--) {
			Object target = receivers.get(i);
			if (target instanceof ResolveInfo) {
				ResolveInfo ri = (ResolveInfo)target;
				if (ri.activityInfo.exported && ri.activityInfo.permission == null) {
					allProtected = false;
					break;
				}
			} else {
				BroadcastFilter bf = (BroadcastFilter)target;
				if (bf.requiredPermission == null) {
					allProtected = false;
					break;
				}
			}
		}
		if (allProtected) {
			// All safe!
			return;
		}
	}
	// The vast majority of broadcasts sent from system internals
	// should be protected to avoid security holes, so yell loudly
	// to ensure we examine these cases.
	if (callerApp != null) {
		Log.wtf(TAG, "Sending non-protected broadcast " + action
		+ " from system " + callerApp.toShortString() + " pkg " + callerPackage,
		new Throwable());
	} else {
		Log.wtf(TAG, "Sending non-protected broadcast " + action
		+ " from system uid " + UserHandle.formatUid(callingUid)
		+ " pkg " + callerPackage,
		new Throwable());
	}
}
具体修改为：
private void checkBroadcastFromSystem(Intent intent, ProcessRecord callerApp,
String callerPackage, int callingUid, Boolean isProtectedBroadcast, List receivers) {
	if ((intent.getFlags() & Intent.FLAG_RECEIVER_FROM_SHELL) != 0) {
		// Don't yell about broadcasts sent via shell
		return;
	}
	final String action = intent.getAction();
	if (isProtectedBroadcast
	|| Intent.ACTION_CLOSE_SYSTEM_DIALOGS.equals(action)
	|| Intent.ACTION_DISMISS_KEYBOARD_SHORTCUTS.equals(action)
	|| Intent.ACTION_MEDIA_BUTTON.equals(action)
	|| Intent.ACTION_MEDIA_SCANNER_SCAN_FILE.equals(action)
	|| Intent.ACTION_SHOW_KEYBOARD_SHORTCUTS.equals(action)
	|| Intent.ACTION_MASTER_CLEAR.equals(action)
	|| Intent.ACTION_FACTORY_RESET.equals(action)
             + || "com.custom.intent.action.BOOT_COMPLETED".equals(action)
	|| AppWidgetManager.ACTION_APPWIDGET_CONFIGURE.equals(action)
	|| AppWidgetManager.ACTION_APPWIDGET_UPDATE.equals(action)
	|| LocationManager.HIGH_POWER_REQUEST_CHANGE_ACTION.equals(action)
	|| TelephonyManager.ACTION_REQUEST_OMADM_CONFIGURATION_UPDATE.equals(action)
	|| SuggestionSpan.ACTION_SUGGESTION_PICKED.equals(action)
	|| AudioEffect.ACTION_OPEN_AUDIO_EFFECT_CONTROL_SESSION.equals(action)
	|| AudioEffect.ACTION_CLOSE_AUDIO_EFFECT_CONTROL_SESSION.equals(action)) {
		// Broadcast is either protected, or it's a public action that
		// we've relaxed, so it's fine for system internals to send.
		return;
	}
	// This broadcast may be a problem...  but there are often system components that
	// want to send an internal broadcast to themselves, which is annoying to have to
	// explicitly list each action as a protected broadcast, so we will check for that
	// one safe case and allow it: an explicit broadcast, only being received by something
	// that has protected itself.
	if (intent.getPackage() != null || intent.getComponent() != null) {
		if (receivers == null || receivers.size() == 0) {
			// Intent is explicit and there's no receivers.
			// This happens, e.g. , when a system component sends a broadcast to
			// its own runtime receiver, and there's no manifest receivers for it,
			// because this method is called twice for each broadcast,
			// for runtime receivers and manifest receivers and the later check would find
			// no receivers.
			return;
		}
		Boolean allProtected = true;
		for (int i = receivers.size()-1; i >= 0; i--) {
			Object target = receivers.get(i);
			if (target instanceof ResolveInfo) {
				ResolveInfo ri = (ResolveInfo)target;
				if (ri.activityInfo.exported && ri.activityInfo.permission == null) {
					allProtected = false;
					break;
				}
			} else {
				BroadcastFilter bf = (BroadcastFilter)target;
				if (bf.requiredPermission == null) {
					allProtected = false;
					break;
				}
			}
		}
		if (allProtected) {
			// All safe!
			return;
		}
	}
	// The vast majority of broadcasts sent from system internals
	// should be protected to avoid security holes, so yell loudly
	// to ensure we examine these cases.
 
    //注释掉检测广播功能
	/*if (callerApp != null) {
		Log.wtf(TAG, "Sending non-protected broadcast " + action
		+ " from system " + callerApp.toShortString() + " pkg " + callerPackage,
		new Throwable());
	} else {
		Log.wtf(TAG, "Sending non-protected broadcast " + action
		+ " from system uid " + UserHandle.formatUid(callingUid)
		+ " pkg " + callerPackage,
		new Throwable());
	}*/
}
```
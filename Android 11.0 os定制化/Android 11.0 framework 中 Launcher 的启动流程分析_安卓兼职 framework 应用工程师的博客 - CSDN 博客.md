> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/130413023)

1. 前言
-----

  
  在 11.0 的系统 rom 定制化开发中，在 rom 定制过程中，在对于开发默认 [Launcher](https://so.csdn.net/so/search?q=Launcher&spm=1001.2101.3001.7020) 功能，解决开机动画后黑屏，了解 fallbackhome 机制等等  
对于 launcher 的启动流程来说很重要，接下来就来分析下 launcher 的启动流程

2.framework 中 Launcher 的启动流程分析的核心类
----------------------------------

```
frameworks/base/services/java/com/android/server/SystemServer.java
frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
```

3.framework 中 Launcher 的启动流程分析的核心功能分析和实现  
3.1 分析下 SystemServer.java 中相关启动 [AMS](https://so.csdn.net/so/search?q=AMS&spm=1001.2101.3001.7020) 的相关方法
---------------------------------------------------------------------------------------------------------------------------------------------------

```
      private void run() {
          TimingsTraceAndSlog t = new TimingsTraceAndSlog();
          try {
              t.traceBegin("InitBeforeStartServices");
  
              // Record the process start information in sys props.
              SystemProperties.set(SYSPROP_START_COUNT, String.valueOf(mStartCount));
              SystemProperties.set(SYSPROP_START_ELAPSED, String.valueOf(mRuntimeStartElapsedTime));
              SystemProperties.set(SYSPROP_START_UPTIME, String.valueOf(mRuntimeStartUptime));
  
...
  
              // Prepare the main looper thread (this thread).
              android.os.Process.setThreadPriority(
                      android.os.Process.THREAD_PRIORITY_FOREGROUND);
              android.os.Process.setCanSelfBackground(false);
              Looper.prepareMainLooper();
              Looper.getMainLooper().setSlowLogThresholdMs(
                      SLOW_DISPATCH_THRESHOLD_MS, SLOW_DELIVERY_THRESHOLD_MS);
  
              SystemServiceRegistry.sEnableServiceNotFoundWtf = true;
  
              // Initialize native services.
              System.loadLibrary("android_servers");
  
              // Allow heap / perf profiling.
              initZygoteChildHeapProfiling();
  
              // Debug builds - spawn a thread to monitor for fd leaks.
              if (Build.IS_DEBUGGABLE) {
                  spawnFdLeakCheckThread();
              }
  
              // Check whether we failed to shut down last time we tried.
              // This call may not return.
              performPendingShutdown();
  
              // Initialize the system context.
              createSystemContext();
			  ...
		}
    private void createSystemContext() {
        ActivityThread activityThread = ActivityThread.systemMain();
        mSystemContext = activityThread.getSystemContext();
        mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);
 
        final Context systemUiContext = activityThread.getSystemUiContext();
        systemUiContext.setTheme(DEFAULT_SYSTEM_THEME);
    }
```

在 SystemServer.java 中的上述方法中，在 run() 方法中，通过调用 createSystemContext(); 来  
创造上下文，来设置系统 theme 样式

```
private void  startBootstrapServices(){
  //创建ATMS和AMS
ActivityTaskManagerService atm = mSystemServiceManager.startService(
        ActivityTaskManagerService.Lifecycle.class).getService();
mActivityManagerService = ActivityManagerService.Lifecycle.startService(
        mSystemServiceManager, atm);
}
 
//ActivityTaskManagerService内部类
public static final class Lifecycle extends SystemService {
   private final ActivityTaskManagerService mService;
 
   public Lifecycle(Context context) {
        super(context);
        mService = new ActivityTaskManagerService(context);
   }
 
   @Override
   public void onStart() {
       publishBinderService(Context.ACTIVITY_TASK_SERVICE, mService);
       mService.start();
   }
}
```

在 SystemServer.java 中的引导服务中启动了 ATMS（ActivityTaskManagerServici）、AMS 等服务，其中 ATMS 是 Android q 以后新增的，本来都是 AMS 来管理  
，由于 google 考虑到 AMS 职责太多功能太多，所以单独拆出来 ATMS 用于管理 Activity。  
所以接下来就来看下 AMS 相关启动 launcher 的代码分析

3.2 ActivityManagerService.java 关于启动 launcher 的相关代码
---------------------------------------------------

```
     */
      public void systemReady(final Runnable goingCallback, @NonNull TimingsTraceAndSlog t) {
          t.traceBegin("PhaseActivityManagerReady");
          mSystemServiceManager.preSystemReady();
 
  
          t.traceBegin("registerActivityLaunchObserver");
          mAtmInternal.getLaunchObserverRegistry().registerLaunchObserver(mActivityLaunchObserver);
          t.traceEnd();
  
          t.traceBegin("watchDeviceProvisioning");
          watchDeviceProvisioning(mContext);
          t.traceEnd();
  
          t.traceBegin("retrieveSettings");
          retrieveSettings();
          t.traceEnd();
  
          t.traceBegin("Ugm.onSystemReady");
          mUgmInternal.onSystemReady();
          t.traceEnd();
  
          t.traceBegin("updateForceBackgroundCheck");
          final PowerManagerInternal pmi = LocalServices.getService(PowerManagerInternal.class);
          if (pmi != null) {
              pmi.registerLowPowerModeObserver(ServiceType.FORCE_BACKGROUND_CHECK,
                      state -> updateForceBackgroundCheck(state.batterySaverEnabled));
              updateForceBackgroundCheck(
                      pmi.getLowPowerState(ServiceType.FORCE_BACKGROUND_CHECK).batterySaverEnabled);
          } else {
              Slog.wtf(TAG, "PowerManagerInternal not found.");
          }
          t.traceEnd();
  
          if (goingCallback != null) goingCallback.run();
  
          t.traceBegin("getCurrentUser"); // should be fast, but these methods acquire locks
          // Check the current user here as a user can be started inside goingCallback.run() from
          // other system services.
          final int currentUserId = mUserController.getCurrentUserId();
          Slog.i(TAG, "Current user:" + currentUserId);
          if (currentUserId != UserHandle.USER_SYSTEM && !mUserController.isSystemUserStarted()) {
              // User other than system user has started. Make sure that system user is already
              // started before switching user.
              throw new RuntimeException("System user not started while current user is:"
                      + currentUserId);
          }
          t.traceEnd();
  
          t.traceBegin("ActivityManagerStartApps");
          mBatteryStatsService.noteEvent(BatteryStats.HistoryItem.EVENT_USER_RUNNING_START,
                  Integer.toString(currentUserId), currentUserId);
          mBatteryStatsService.noteEvent(BatteryStats.HistoryItem.EVENT_USER_FOREGROUND_START,
                  Integer.toString(currentUserId), currentUserId);
  
          // On Automotive, at this point the system user has already been started and unlocked,
          // and some of the tasks we do here have already been done. So skip those in that case.
          // TODO(b/132262830): this workdound shouldn't be necessary once we move the
          // headless-user start logic to UserManager-land
          final boolean bootingSystemUser = currentUserId == UserHandle.USER_SYSTEM;
  
          if (bootingSystemUser) {
              mSystemServiceManager.startUser(t, currentUserId);
          }
  
  
              if (bootingSystemUser) {
                  t.traceBegin("startHomeOnAllDisplays");
                  mAtmInternal.startHomeOnAllDisplays(currentUserId, "systemReady");
                  t.traceEnd();
              }
```

在 ActivityManagerService.java 中的上述方法中, 在 systemReady(final Runnable goingCallback, @NonNull TimingsTraceAndSlog t)  
中在系统 ams 启动完毕后，在启动 atms 的相关功能，在调用 mAtmInternal.startHomeOnAllDisplays(currentUserId, "systemReady");  
来查询系统中符合条件的 Launcher 来启动，接下来分析下 atms 的相关源码

3.3 ActivityTaskManagerService.java 中相关源码分析
-------------------------------------------

  
在 ActivityTaskManagerService.java 中的构造方法中初始化，是由 ATMS 对外提供的一个抽象类，真正的实现获取 launcher 的相关功能是在 ATMS 中的 LocalService，  
所以执行到了 LocalService 的 startHomeOnAllDisplays 方法。

```
    //ActivityTaskManagerService.java # LocalService
    public boolean startHomeOnAllDisplays(int userId, String reason) {
       ...
        synchronized (mGlobalLock) {
            return mRootWindowContainer.startHomeOnAllDisplays(userId, reason);
        }
       ...
    }
```

通过在 RootWindowContainer 中调用 startHomeOnAllDisplays(userId, reason) 方法，  
然后会调用到 startHomeOnTaskDisplayArea 方法，这里 getHomeIntent 方法中创建了一个 category 为  
 CATEGORY_HOME 的 Intent，这个 category 会在 Launcher3 的 AndroidManifest.xml 中配置，表明是 HomeActivity，  
最后再调用 startHomeActivity 方法。

```
       boolean startHomeOnTaskDisplayArea(int userId, String reason, TaskDisplayArea taskDisplayArea,
                boolean allowInstrumenting, boolean fromHomeKey) {
....
            Intent homeIntent = null;
            ActivityInfo aInfo = null;
     
            homeIntent = mService.getHomeIntent();
            aInfo = resolveHomeActivity(userId, homeIntent);
     
            mService.getActivityStartController().startHomeActivity(homeIntent, aInfo, myReason,
                    taskDisplayArea);
....
            return true;
        }
```

在上述的 startHomeOnTaskDisplayArea 方法中，会通过. startHomeActivity(homeIntent, aInfo, myReason,  
                    taskDisplayArea); 来调用到  
ActivityStartController.startHomeActivity 方法  
而 obtainStarter 方法返回的是 ActivityStarter 对象，它负责 Activity 的启动。这里多个 set 方法设置启动需要的参数，  
 最后 execute 方法才是真正的启动逻辑。

```
 
    //ActivityStartController.java
    void startHomeActivity(Intent intent, ActivityInfo aInfo, String reason,
                TaskDisplayArea taskDisplayArea) {
            //obtainStarter 返回一个 ActivityStarter，它负责 Activity 的启动
            mLastHomeActivityStartResult = obtainStarter(intent, "startHomeActivity: " + reason)
                    .setOutActivity(tmpOutRecord)
                    .setCallingUid(0)
                    .setActivityInfo(aInfo)
                    .setActivityOptions(options.toBundle())
                    .execute();
        }
```

在 ActivityStarter 的 extcute 这方法中，在

onExecutionComplete 方法在 ActivityStartController 清除数据放回 startPool 池子中。

```
        int execute() {
            try {
                int res;
                synchronized (mService.mGlobalLock) {
                    ...
                    //执行这个方法
                    res = executeRequest(mRequest);
                    ...
            } finally {
                onExecutionComplete();
            }
        }
 
 ActivityStarter.executeRequest 
 
        private int executeRequest(Request request) {
            //调用 startActivityUnchecked
            mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
                    request.voiceInteractor, startFlags, true /* doResume */, checkedOptions, inTask,
                    restrictedBgActivity, intentGrants);
            return mLastStartActivityResult;
        }
 
 ActivityStarter.startActivityUnchecked
 
        private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
                    IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                    int startFlags, boolean doResume, ActivityOptions options, Task inTask,
                    boolean restrictedBgActivity, NeededUriGrants intentGrants) {
            int result = START_CANCELED;
            try {
                //延时布局
                mService.deferWindowLayout();
                //执行startActivityInner
                result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                        startFlags, doResume, options, inTask, restrictedBgActivity, intentGrants);
            } finally {
                //恢复布局
                mService.continueWindowLayout();
            }
            return result;
        }
 
 ActivityStarter.startActivityInner
 
    int startActivityInner() { 
           if (mDoResume) {
                //ActivityRecord 记录着 Activity 信息
                final ActivityRecord topTaskActivity =
                        mStartActivity.getTask().topRunningActivityLocked();
                if (!mTargetStack.isTopActivityFocusable()
                        || (topTaskActivity != null && topTaskActivity.isTaskOverlay()
                        && mStartActivity != topTaskActivity)) {
                    mTargetStack.getDisplay().mDisplayContent.executeAppTransition();
                } else {
                    //执行resumeFocusedStacksTopActivities
                    mRootWindowContainer.resumeFocusedStacksTopActivities(
                            mTargetStack, mStartActivity, mOptions);
                }
            }
            return START_SUCCESS;
        }
 
RootWindowContainer.resumeFocusedStacksTopActivities 
 
    boolean resumeFocusedStacksTopActivities(
                ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
            //如果是栈顶Activity，启动resumeTopActivityUncheckedLocked
            if (targetStack != null && (targetStack.isTopStackInDisplayArea()
                    || getTopDisplayFocusedStack() == targetStack)) {
                result = targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
            }
            if (!resumedOnDisplay) {
                //获取栈顶的 ActivityRecord
                final ActivityStack focusedStack = display.getFocusedStack();
                if (focusedStack != null) {
                    //执行resumeTopActivityUncheckedLocked
                    result |= focusedStack.resumeTopActivityUncheckedLocked(target, targetOptions);
                }
            }
          return result;
    }
 
ActivityStack.resumeTopActivityUncheckedLocked
 
        boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
            boolean result = false;
            try {
                mInResumeTopActivity = true;
                //执行resumeTopActivityInnerLocked,
                //最终调用到ActivityStackSupervisor.startSpecificActivity
                result = resumeTopActivityInnerLocked(prev, options);
            } finally {
                mInResumeTopActivity = false;
            }
            return result;
        }
```

在调用到  
ActivityStackSupervisor.startSpecificActivity

这个方法是关键方法，如果进程已经存在直接执行 realStartActivityLocked 去启动 Activity，进程不存在则通过 AMS 去创建 Socket，  
然后通知 Zygote 去 fork 进程。由于这里第一次创建，所以走到 startProcessAsync

```
       void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) {
            // Is this activity's application already running?
            final WindowProcessController wpc =
                    mService.getProcessController(r.processName, r.info.applicationInfo.uid);
     
            boolean knownToBeDead = false;
            //进程存在
            if (wpc != null && wpc.hasThread()) {
                try {
                    //真正启动Activity方法
                    realStartActivityLocked(r, wpc, andResume, checkConfig);
                    return;
                } catch (RemoteException e) {
                    Slog.w(TAG, "Exception when starting activity "
                            + r.intent.getComponent().flattenToShortString(), e);
                }
                // If a dead object exception was thrown -- fall through to
                // restart the application.
                knownToBeDead = true;
            }
     
            r.notifyUnknownVisibilityLaunchedForKeyguardTransition();
     
            final boolean isTop = andResume && r.isTopRunningActivity();
            //进程不存在 mService =ATMS
            mService.startProcessAsync(r, knownToBeDead, isTop, isTop ? "top-activity" : "activity");
        }
```
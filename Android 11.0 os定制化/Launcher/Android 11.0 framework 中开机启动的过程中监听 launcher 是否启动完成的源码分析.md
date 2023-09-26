> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/131504680)

1. 前言
-----

在 11.0 的系统开发中，在系统开机启动阶段，对于首次开机动画播放完毕后，有些产品会出现黑屏的情况，为了解决黑屏问题，这时候就需要判断当前 [Launcher](https://so.csdn.net/so/search?q=Launcher&spm=1001.2101.3001.7020) 是否启动完毕，然后  
在做相关的处理，接下来就来分析下关于判断 launcher 是否启动完毕的源码分析

2.framework 中开机启动的过程中监听 launcher 是否启动完成的源码分析的核心类
------------------------------------------------

```
   frameworks/base/core/java/android/app/ActivityThread.java
    frameworks/base/services/core/java/com/android/server/am/ActivityTaskManagerService.java
    frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java
    frameworks/base/core/java/android/os/MessageQueue.java
```

3.framework 中开机启动的过程中监听 launcher 是否启动完成的源码分析的功能分析和实现
----------------------------------------------------

  
在 android 系统中，系统的启动流程中是在 [AMS](https://so.csdn.net/so/search?q=AMS&spm=1001.2101.3001.7020) 即 ActivityManagerService.java 中主要负责启动 Launcher，在 Launcher 启动完成后即 Activity onResume 之后，就会发送开机完成广播，表示当前  
系统启动完毕。而在 Launcher 的 Activity 在 onResume 的方法中，  
执行了 handleResumeActivity 方法，在 handleResumeActivity 中加载完 window 之后将自己实现的 IdleHandler 添加到自己的[消息队列](https://so.csdn.net/so/search?q=%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97&spm=1001.2101.3001.7020)中。

3.1 ActivityThread.java 的启动 activity 的方法如下
------------------------------------------

```
       @Override
        public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
                String reason) {
            // If we are getting ready to gc after going to the background, well
            // we are back active so skip it.
            unscheduleGcIdler();
            mSomeActivitiesChanged = true;
    //执行onResume方法
            // TODO Push resumeArgs into the activity for consideration
            final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
            if (r == null) {
                // We didn't actually resume the activity, so skipping any follow-up actions.
                return;
            }
    .......
            r.nextIdle = mNewActivities;
            mNewActivities = r;
            if (localLOGV) Slog.v(TAG, "Scheduling idle handler for " + r);
    //将IdleHandler加入到消息队列
            Looper.myQueue().addIdleHandler(new Idler());
            // UNISOC: USIM notifier feature
            if (UsimNotifier.isNotifyEnabled(getApplication())) {
                UsimNotifier.getInstance().startNotify(r.activity);
            }
        }
```

在上述的 ActivityThread.java 的相关源码中，可以看出在 handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,  
            String reason) 中，主要是执行 Activity 的 onResume 相关流程，  
另外一个重要功能就是将 IdleHandler 加入到消息队列中，当消息队列空闲的时候执行 idler.queueIdle() 的回调

从上面可以看到当执行 handleResumeActivity 方法后，通过 View decor=r.window.getDecorView();  
获取到 decor，并设置为隐藏。然后获取 WindowManager, 然后把 decor 添加到 WindowManager 中，之后我们在看 WindowManager 中，发现它是一个接口

```
       private class Idler implements MessageQueue.IdleHandler {
            @Override
            public final boolean queueIdle() {
                ActivityClientRecord a = mNewActivities;
                boolean stopProfiling = false;
                if (mBoundApplication != null && mProfiler.profileFd != null
                        && mProfiler.autoStopProfiler) {
                    stopProfiling = true;
                }
                if (a != null) {
                    mNewActivities = null;
                    IActivityTaskManager am = ActivityTaskManager.getService();
                    ActivityClientRecord prev;
                    do {
                        if (localLOGV) Slog.v(
                            TAG, "Reporting idle of " + a +
                            " finished=" +
                            (a.activity != null && a.activity.mFinished));
                        if (a.activity != null && !a.activity.mFinished) {
                            try {
                                am.activityIdle(a.token, a.createdConfig, stopProfiling);
                                a.createdConfig = null;
                            } catch (RemoteException ex) {
                                throw ex.rethrowFromSystemServer();
                            }
                        }
                        prev = a;
                        a = a.nextIdle;
                        prev.nextIdle = null;
                    } while (a != null);
                }
                if (stopProfiling) {
                    mProfiler.stopProfiling();
                }
                applyPendingProcessState();
                return false;
            }
        }
```

在 ActivityThread.java 的 Idler 中，在消息队列空闲的时候，会调用 queueIdle() 进行消息的处理  
而在 queueIdle() 中，首选是获取到 IActivityTaskManager am = ActivityTaskManager.getService();  
这个 atms 的对象，然后调用 am.activityIdle(a.token, a.createdConfig, stopProfiling);  
来进行具体的 activity 组件空闲的时候 进行的相关流程处理

```
       @Override
        public final void activityIdle(IBinder token, Configuration config, boolean stopProfiling) {
            final long origId = Binder.clearCallingIdentity();
            try {
                WindowProcessController proc = null;
                synchronized (mGlobalLock) {
                    ActivityStack stack = ActivityRecord.getStackLocked(token);
                    if (stack == null) {
                        return;
                    }
                    final ActivityRecord r = mStackSupervisor.activityIdleInternalLocked(token,
                            false /* fromTimeout */, false /* processPausingActivities */, config);
                    if (r != null) {
                        proc = r.app;
                    }
                    if (stopProfiling && proc != null) {
                        proc.clearProfilerIfNeeded();
                    }
                }
            } finally {
                Binder.restoreCallingIdentity(origId);
            }
        }
```

系统源码中 ActivityThread 的 Idler，在 handleResumeActivity() 方法内会注册 Idler()，等待  
handleResumeActivity 后视图绘制完成，消息队列暂时空闲时再调用 AMS 的 activityIdle 方法，  
检查页面的生命周期状态，触发 activity 的 stop 生命周期等。

在 ActivityTaskManagerService.java 的上述源码中，可以看出在 activityIdle(IBinder token, Configuration config, boolean stopProfiling)  
中处理 activity 组件空闲的时候，主要是调用 mStackSupervisor.activityIdleInternalLocked(token,  
                        false /* fromTimeout */, false /* processPausingActivities */, config)  
来处理具体的流程

```
     @GuardedBy("mService")
        final ActivityRecord activityIdleInternalLocked(final IBinder token, boolean fromTimeout,
                boolean processPausingActivities, Configuration config) {
            if (DEBUG_ALL) Slog.v(TAG, "Activity idle: " + token);
     
            ArrayList<ActivityRecord> finishes = null;
            ArrayList<UserState> startingUsers = null;
            int NS = 0;
            int NF = 0;
            boolean booting = false;
            boolean activityRemoved = false;
     
            ActivityRecord r = ActivityRecord.forTokenLocked(token);
            if (r != null) {
                if (DEBUG_IDLE) Slog.d(TAG_IDLE, "activityIdleInternalLocked: Callers="
                        + Debug.getCallers(4));
                mHandler.removeMessages(IDLE_TIMEOUT_MSG, r);
    //add core start
    			String boot_completed = android.os.SystemProperties.get("sys.boot_completed");
    			android.util.Log.e("ASS","activityIdleInternalLocked--package+boot_completed);
     //add core end          
                r.finishLaunchTickingLocked();
                if (fromTimeout) {
                    reportActivityLaunchedLocked(fromTimeout, r, INVALID_DELAY,
                            -1 /* launchState */);
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
                if ((mService.isBooting() && mRootActivityContainer.allResumedActivitiesIdle())
                        || fromTimeout) {
                    booting = checkFinishBootingLocked();
                }
                // When activity is idle, we consider the relaunch must be successful, so let's clear
                // the flag.
                r.mRelaunchReason = RELAUNCH_REASON_NONE;
            }
    ...
     
    }
```

activity 退出内存回收流程:  
1. 启动 Activity 和按下 back 键都涉及 Activity 的销毁。

2. 启动 Activity 时会先去暂停当前显示的 Activity，当待启动 Activity 启动完成时回调 AMS 的 activityIdleInternalLocked() 函数。在该函数中会执行内存回收操作。

3.activityIdleInternalLocked() 函数内部依次处理 stop、finish 状态的 Activity，最后会执行到 AMS 的 trimApplications()，在该函数中执行内存回收。

4.activityIdleInternalLocked() 是执行内存回收的关键函数。

在 ActivityStackSupervisor.java 中的 activityIdleInternalLocked(）中，在  
  r.finishLaunchTickingLocked(); 中就是判断当前 Launcher 是否启动完毕的，  
然后可以在这里判断包名，调用 r.packageName 来获取当前页面的包名
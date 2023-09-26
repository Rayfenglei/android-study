> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127309069)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. app 进程保活白名单功能实现的核心类](#t1)

[3. app 进程保活白名单功能实现的核心功能实现和分析](#t2)

 [3.1 在 IActivityManager.aidl 中增加进程白名单的接口](#t3)

 [3.2 在 ActivityManager.java 中增加白名单接口给 app 调用](#t4)

[3.3 ActivityManagerService.java 中实现 IActivityManager.aidl 的白名单接口](#t5)

[3.4 ActivityStackSupervisor.java 中关于任务栈中杀进程的相关功能分析](#t6)

1. 概述
-----

  在 11.0 的系统产品开发中，在某些重要的 app 即使进入后台，产品需求要求也不想被系统杀掉进程，需要 app 长时间保活，就是 app 进程保活白名单功能的实现，所以需要在系统[杀进程](https://so.csdn.net/so/search?q=%E6%9D%80%E8%BF%9B%E7%A8%8B&spm=1001.2101.3001.7020)的时候不杀掉白名单的进程

2. app 进程保活白名单功能实现的核心类
----------------------

```
  frameworks/base/core/java/android/app/IActivityManager.aidl
  frameworks/base/core/java/android/app/ActivityManager.java
  frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
  frameworks/base/services/core/java/com/android/server/wm/ActivityStackSupervisor.java
```

3. app 进程保活白名单功能实现的核心功能实现和分析
----------------------------

###   3.1 在 IActivityManager.aidl 中增加进程白名单的接口

```
    void killUidForPermissionChange(int appId, int userId, String reason);
    boolean enableAppFreezer(in boolean enable);
 
+    List<String> getWhiteAppProcessList();
+    void setWhiteAppProcessList(in List<String> whiteAppProcessList);
```

###   3.2 在 ActivityManager.java 中增加白名单接口给 app 调用

```
   public List<String> getWhiteAppProcessList() {
        try {
            return getService().getWhiteAppProcessList();
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
    public void setWhiteAppProcessList(List<String> whiteAppProcessList) {
        try {
            getService().setWhiteAppList(whiteAppProcessList);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```

增加这两个关于 app 进程白名单的接口，通过调用 ActivityManagerService.java 的白名单接口来设置 app 进程白名单  
来保活 app 进程

### 3.3 ActivityManagerService.java 中实现 IActivityManager.aidl 的白名单接口

   1. 实现保活白名单接口

```
    public class ActivityManagerService extends IActivityManager.Stub
            implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
     private List<String> mWhiteAppProcessList = new ArrayList<String>();
    public void setWindowManager(WindowManagerService wm) {
          synchronized (this) {
              mWindowManager = wm;
              mWmInternal = LocalServices.getService(WindowManagerInternal.class);
              mActivityTaskManager.setWindowManager(wm);
          }
      }
  
      public void setUsageStatsManager(UsageStatsManagerInternal usageStatsManager) {
          mUsageStatsService = usageStatsManager;
          mActivityTaskManager.setUsageStatsManager(usageStatsManager);
      }
//add core start
private List<String> mWhiteAppProcessList;
    @Override
    public List<String> getWhiteAppProcessList() {
        return mWhiteAppProcessList;
    }
    @Override
    public void setWhiteAppProcessList(List<String> whiteAppProcessList) {
        this.mWhiteAppProcessList = whiteAppProcessList;
    }
//add core end
```

从 ActivityManagerService 中可以看出继承了 IActivityManager.Stub 所以说就是 IActivityManager 的子类所以需要实现 IActivityManager 定义的保活白名单接口  
接下来看下关于杀进程的相关方法

```
      /**
       * Main function for removing an existing process from the activity manager
       * as a result of that process going away.  Clears out all connections
       * to the process.
       */
      @GuardedBy("this")
      final void handleAppDiedLocked(ProcessRecord app,
              boolean restarting, boolean allowRestart) {
          int pid = app.pid;
          boolean kept = cleanUpApplicationRecordLocked(app, restarting, allowRestart, -1,
                  false /*replacingPid*/);
          if (!kept && !restarting) {
              removeLruProcessLocked(app);
              if (pid > 0) {
                  ProcessList.remove(pid);
              }
          }
  
          if (mProfileData.getProfileProc() == app) {
              clearProfilerLocked();
          }
  
          mAtmInternal.handleAppDied(app.getWindowProcessController(), restarting, () -> {
              Slog.w(TAG, "Crash of app " + app.processName
                      + " running instrumentation " + app.getActiveInstrumentation().mClass);
              Bundle info = new Bundle();
              info.putString("shortMsg", "Process crashed.");
              finishInstrumentationLocked(app, Activity.RESULT_CANCELED, info);
          });
      }
```

从 handleAppDiedLocked 的注释中可以看出在 ActivityManagerService.java 中主要是调用 handleAppDiedLocked 处理杀掉进程的  
功能的，而在 cleanUpApplicationRecordLocked 中主要处理杀进程的工作  
接下来看 cleanUpApplicationRecordLocked 相关进程处理的方法

```
     @GuardedBy("this")
      final boolean cleanUpApplicationRecordLocked(ProcessRecord app,
              boolean restarting, boolean allowRestart, int index, boolean replacingPid) {
          if (index >= 0) {
              removeLruProcessLocked(app);
              ProcessList.remove(app.pid);
          }
  
          mProcessesToGc.remove(app);
          mPendingPssProcesses.remove(app);
          ProcessList.abortNextPssTime(app.procStateMemTracker);
  
          // Dismiss any open dialogs.
          app.getDialogController().clearAllErrorDialogs();
  
          app.setCrashing(false);
          app.setNotResponding(false);
  
          app.resetPackageList(mProcessStats);
          app.unlinkDeathRecipient();
          app.makeInactive(mProcessStats);
          app.waitingToKill = null;
          app.forcingToImportant = null;
          updateProcessForegroundLocked(app, false, 0, false);
          app.setHasForegroundActivities(false);
          app.hasShownUi = false;
          app.treatLikeActivity = false;
          app.hasAboveClient = false;
          app.setHasClientActivities(false);
  
          mServices.killServicesLocked(app, allowRestart);
  
          boolean restart = false;
  
          // Remove published content providers.
          for (int i = app.pubProviders.size() - 1; i >= 0; i--) {
              ContentProviderRecord cpr = app.pubProviders.valueAt(i);
              if (cpr.proc != app) {
                  // If the hosting process record isn't really us, bail out
                  continue;
              }
              final boolean alwaysRemove = app.bad || !allowRestart;
              final boolean inLaunching = removeDyingProviderLocked(app, cpr, alwaysRemove);
              if (!alwaysRemove && inLaunching && cpr.hasConnectionOrHandle()) {
                  // We left the provider in the launching list, need to
                  // restart it.
                  restart = true;
              }
  
              cpr.provider = null;
              cpr.setProcess(null);
          }
          app.pubProviders.clear();
  
          // Take care of any launching providers waiting for this process.
          if (cleanupAppInLaunchingProvidersLocked(app, false)) {
              mProcessList.noteProcessDiedLocked(app);
              restart = true;
          }
  
          // Unregister from connected content providers.
          if (!app.conProviders.isEmpty()) {
              for (int i = app.conProviders.size() - 1; i >= 0; i--) {
                  ContentProviderConnection conn = app.conProviders.get(i);
                  conn.provider.connections.remove(conn);
                  stopAssociationLocked(app.uid, app.processName, conn.provider.uid,
                          conn.provider.appInfo.longVersionCode, conn.provider.name,
                          conn.provider.info.processName);
              }
              app.conProviders.clear();
          }
  
          // At this point there may be remaining entries in mLaunchingProviders
          // where we were the only one waiting, so they are no longer of use.
          // Look for these and clean up if found.
          // XXX Commented out for now.  Trying to figure out a way to reproduce
          // the actual situation to identify what is actually going on.
          if (false) {
              for (int i = mLaunchingProviders.size() - 1; i >= 0; i--) {
                  ContentProviderRecord cpr = mLaunchingProviders.get(i);
                  if (cpr.connections.size() <= 0 && !cpr.hasExternalProcessHandles()) {
                      synchronized (cpr) {
                          cpr.launchingApp = null;
                          cpr.notifyAll();
                      }
                  }
              }
          }
  
          skipCurrentReceiverLocked(app);
  
          // Unregister any receivers.
          for (int i = app.receivers.size() - 1; i >= 0; i--) {
              removeReceiverLocked(app.receivers.valueAt(i));
          }
          app.receivers.clear();
  
          // If the app is undergoing backup, tell the backup manager about it
          final BackupRecord backupTarget = mBackupTargets.get(app.userId);
          if (backupTarget != null && app.pid == backupTarget.app.pid) {
              if (DEBUG_BACKUP || DEBUG_CLEANUP) Slog.d(TAG_CLEANUP, "App "
                      + backupTarget.appInfo + " died during backup");
              mHandler.post(new Runnable() {
                  @Override
                  public void run(){
                      try {
                          IBackupManager bm = IBackupManager.Stub.asInterface(
                                  ServiceManager.getService(Context.BACKUP_SERVICE));
                          bm.agentDisconnectedForUser(app.userId, app.info.packageName);
                      } catch (RemoteException e) {
                          // can't happen; backup manager is local
                      }
                  }
              });
          }
  
          for (int i = mPendingProcessChanges.size() - 1; i >= 0; i--) {
              ProcessChangeItem item = mPendingProcessChanges.get(i);
              if (app.pid > 0 && item.pid == app.pid) {
                  mPendingProcessChanges.remove(i);
                  mAvailProcessChanges.add(item);
              }
          }
          mUiHandler.obtainMessage(DISPATCH_PROCESS_DIED_UI_MSG, app.pid, app.info.uid,
                  null).sendToTarget();
  
          // If this is a precede instance of another process instance
          allowRestart = true;
          synchronized (app) {
              if (app.mSuccessor != null) {
                  // We don't allow restart with this ProcessRecord now,
                  // because we have created a new one already.
                  allowRestart = false;
                  // If it's persistent, add the successor to mPersistentStartingProcesses
                  if (app.isPersistent() && !app.removed) {
                      if (mPersistentStartingProcesses.indexOf(app.mSuccessor) < 0) {
                          mPersistentStartingProcesses.add(app.mSuccessor);
                      }
                  }
                  // clean up the field so the successor's proc starter could proceed.
                  app.mSuccessor.mPrecedence = null;
                  app.mSuccessor = null;
                  // Notify if anyone is waiting for it.
                  app.notifyAll();
              }
          }
  
          // If the caller is restarting this app, then leave it in its
          // current lists and let the caller take care of it.
          if (restarting) {
              return false;
          }
  
          if (!app.isPersistent() || app.isolated) {
              if (DEBUG_PROCESSES || DEBUG_CLEANUP) Slog.v(TAG_CLEANUP,
                      "Removing non-persistent process during cleanup: " + app);
              if (!replacingPid) {
                  mProcessList.removeProcessNameLocked(app.processName, app.uid, app);
              }
              mAtmInternal.clearHeavyWeightProcessIfEquals(app.getWindowProcessController());
          } else if (!app.removed) {
              // This app is persistent, so we need to keep its record around.
              // If it is not already on the pending app list, add it there
              // and start a new process for it.
              if (mPersistentStartingProcesses.indexOf(app) < 0) {
                  mPersistentStartingProcesses.add(app);
                  restart = true;
              }
          }
         .....
          return false;
      }
```

在 cleanUpApplicationRecordLocked 中的进行进程列表遍历的时候，判断 app 是否清除的时候根据  
 if (!app.isPersistent() || app.isolated)  
调用 mAtmInternal.clearHeavyWeightProcessIfEquals(app.getWindowProcessController()); 来杀掉进程所以可以在这里增加判断添加，看是否杀掉进程  
具体修改如下:

```
--- a/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
+++ b/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
@@ -14079,8 +14079,13 @@ public class ActivityManagerService extends IActivityManager.Stub
         if (restarting) {
             return false;
         }
-
-        if (!app.isPersistent() || app.isolated) {
+                List<String> lists=    this.mWhiteAppProcessList;
+                boolean iskeepAlive=false;
+                if(lists!=null && lists.contains(app.processName)){
+                        iskeepAlive=true;
+                }
+        if ((!app.isPersistent() || app.isolated) && !iskeepAlive) {
             if (DEBUG_PROCESSES || DEBUG_CLEANUP) Slog.v(TAG_CLEANUP,
                     "Removing non-persistent process during cleanup: " + app);
             if (!replacingPid) {
@@ -14097,6 +14102,7 @@ public class ActivityManagerService extends IActivityManager.Stub
             // This app is persistent, so we need to keep its record around.
             // If it is not already on the pending app list, add it there
             // and start a new process for it.
             if (mPersistentStartingProcesses.indexOf(app) < 0) {
                 mPersistentStartingProcesses.add(app);
                 restart = true;
```

### 3.4 ActivityStackSupervisor.java 中关于任务栈中杀进程的相关功能分析

```
         @Override
      public void onRecentTaskRemoved(Task task, boolean wasTrimmed, boolean killProcess) {
          if (wasTrimmed) {
              // Task was trimmed from the recent tasks list -- remove the active task record as well
              // since the user won't really be able to go back to it
              removeTaskById(task.mTaskId, killProcess, false /* removeFromRecents */,
                      "recent-task-trimmed");
          }
          task.removedFromRecents();
      }
 
       boolean removeTaskById(int taskId, boolean killProcess, boolean removeFromRecents,
              String reason) {
          final Task task =
                  mRootWindowContainer.anyTaskForId(taskId, MATCH_TASK_IN_STACKS_OR_RECENT_TASKS);
          if (task != null) {
              removeTask(task, killProcess, removeFromRecents, reason);
              return true;
          }
          Slog.w(TAG, "Request to remove task ignored for non-existent task " + taskId);
          return false;
      }
  
      void removeTask(Task task, boolean killProcess, boolean removeFromRecents, String reason) {
          if (task.mInRemoveTask) {
              // Prevent recursion.
              return;
          }
          task.mInRemoveTask = true;
          try {
              task.performClearTask(reason);
              cleanUpRemovedTaskLocked(task, killProcess, removeFromRecents);
              mService.getLockTaskController().clearLockedTask(task);
              mService.getTaskChangeNotificationController().notifyTaskStackChanged();
              if (task.isPersistable) {
                  mService.notifyTaskPersisterLocked(null, true);
              }
          } finally {
              task.mInRemoveTask = false;
          }
      }
```

  在 ActivityStackSupervisor.java 中，关于移除栈内的 app 进程，主要是在 onRecentTaskRemoved 中通过调用 removeTaskById 实现的，而 removeTaskById 又是调用 removeTask 实现的，具体是在 removeTask 中处理的杀进程  
所以具体实现如下:

```
      boolean removeTaskById(int taskId, boolean killProcess, boolean removeFromRecents,
              String reason) {
          final Task task =
                  mRootWindowContainer.anyTaskForId(taskId, MATCH_TASK_IN_STACKS_OR_RECENT_TASKS);
          if (task != null) {
            //add code start
			ComponentName component = tr.getBaseIntent().getComponent();
			if(component!=null){
				String pkg=component.getPackageName();
				        ActivityManager am = (ActivityManager) mService.mContext.getSystemService(
                Context.ACTIVITY_SERVICE);
				List<String> list=am.getWhiteAppProcessList();
				if(list!=null && list.contains(pkg)){
					return false;
				}
			}
           //add code end
              removeTask(task, killProcess, removeFromRecents, reason);
              return true;
          }
          Slog.w(TAG, "Request to remove task ignored for non-existent task " + taskId);
          return false;
      }
```

3.5 OomAdjuster.java 中关于对保活白名单的分析
---------------------------------

```
+    private boolean isInWhitelist(ProcessRecord pr) {
+        String pkgName = pr.info.packageName;
+               List<String> mLmKillerBypassPackages = mService.getKeepAliveList();
+               if(mLmKillerBypassPackages!=null && mLmKillerBypassPackages.size()>0){
+                       for (String token : mLmKillerBypassPackages) {
+                               if (pkgName.startsWith(token)) {
+                                       return true;
+                               }
+                       }
+               }
+        return false;
+    }
 
      /** Applies the computed oomadj, procstate and sched group values and freezes them in set* */
    @GuardedBy("mService")
    private final boolean applyOomAdjLocked(ProcessRecord app, boolean doingAll, long now,
            long nowElapsed) {
        boolean success = true;
 
        if (app.getCurRawAdj() != app.setRawAdj) {
            app.setRawAdj = app.getCurRawAdj();
        }
 
        int changes = 0;
 
        // don't compact during bootup
        if (mCachedAppOptimizer.useCompaction() && mService.mBooted) {
            // Cached and prev/home compaction
            if (app.curAdj != app.setAdj) {
                // Perform a minor compaction when a perceptible app becomes the prev/home app
                // Perform a major compaction when any app enters cached
                // reminder: here, setAdj is previous state, curAdj is upcoming state
                if (app.setAdj <= ProcessList.PERCEPTIBLE_APP_ADJ &&
                        (app.curAdj == ProcessList.PREVIOUS_APP_ADJ ||
                                app.curAdj == ProcessList.HOME_APP_ADJ)) {
                    mCachedAppOptimizer.compactAppSome(app);
                } else if ((app.setAdj < ProcessList.CACHED_APP_MIN_ADJ
                                || app.setAdj > ProcessList.CACHED_APP_MAX_ADJ)
                        && app.curAdj >= ProcessList.CACHED_APP_MIN_ADJ
                        && app.curAdj <= ProcessList.CACHED_APP_MAX_ADJ) {
                    mCachedAppOptimizer.compactAppFull(app);
                }
            } else if (mService.mWakefulness != PowerManagerInternal.WAKEFULNESS_AWAKE
                    && app.setAdj < ProcessList.FOREGROUND_APP_ADJ
                    // Because these can fire independent of oom_adj/procstate changes, we need
                    // to throttle the actual dispatch of these requests in addition to the
                    // processing of the requests. As a result, there is throttling both here
                    // and in CachedAppOptimizer.
                    && mCachedAppOptimizer.shouldCompactPersistent(app, now)) {
                mCachedAppOptimizer.compactAppPersistent(app);
            } else if (mService.mWakefulness != PowerManagerInternal.WAKEFULNESS_AWAKE
                    && app.getCurProcState()
                        == ActivityManager.PROCESS_STATE_BOUND_FOREGROUND_SERVICE
                    && mCachedAppOptimizer.shouldCompactBFGS(app, now)) {
                mCachedAppOptimizer.compactAppBfgs(app);
            }
        }
 
        if (app.curAdj != app.setAdj) {
 
 
// 主要修改部分开始
             // 注释代码开始
            /*ProcessList.setOomAdj(app.pid, app.uid, app.curAdj);
            if (DEBUG_SWITCH || DEBUG_OOM_ADJ || mService.mCurOomAdjUid == app.info.uid) {
                String msg = "Set " + app.pid + " " + app.processName + " adj "
                        + app.curAdj + ": " + app.adjType;
                reportOomAdjMessageLocked(TAG_OOM_ADJ, msg);
            }
            app.setAdj = app.curAdj;
            app.verifiedAdj = ProcessList.INVALID_ADJ;*/
         // 注释代码结束
  
+            boolean isAppWhiteProcess = false;
+            if(isInWhitelist(app) && (app.curAdj > ProcessList.PERSISTENT_SERVICE_ADJ))isAppWhiteProcess = true;
+            if(isAppWhiteProcess){
+                Slog.d(TAG,"isAppWhiteProcess");
+                ProcessList.setOomAdj(app.pid, app.uid, ProcessList.PERSISTENT_SERVICE_ADJ);
+                if (DEBUG_SWITCH || DEBUG_OOM_ADJ || mService.mCurOomAdjUid == app.info.uid) {
+                    String msg = "Set " + app.pid + " " + app.processName + " adj "
+                            + app.curAdj + ": " + app.adjType;
+                    reportOomAdjMessageLocked(TAG_OOM_ADJ, msg);
+                }
+                app.setAdj = ProcessList.PERSISTENT_SERVICE_ADJ;
+                app.verifiedAdj = ProcessList.INVALID_ADJ;
+            }else{
+                ProcessList.setOomAdj(app.pid, app.uid, app.curAdj);
+                if (DEBUG_SWITCH || DEBUG_OOM_ADJ || mService.mCurOomAdjUid == app.info.uid) {
+                    String msg = "Set " + app.pid + " " + app.processName + " adj "
+                            + app.curAdj + ": " + app.adjType;
+                    reportOomAdjMessageLocked(TAG_OOM_ADJ, msg);
+                }
+                app.setAdj = app.curAdj;
+                app.verifiedAdj = ProcessList.INVALID_ADJ;
+            }
 
        }
 
        .....
        return success;
    }
```

在上述的 OomAdjuster.java 的代码中，首选通过 isInWhitelist(ProcessRecord pr)  
判断当前进程是否在白名单之内，然后通过设置 app.setAdj 和 app.verifiedAdj 两个值  
来确保 app 不会被杀掉
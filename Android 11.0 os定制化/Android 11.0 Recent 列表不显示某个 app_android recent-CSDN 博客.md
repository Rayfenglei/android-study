> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124774712)

在 11.0 的产品在点击导航栏最近任务列表时，如果做到不显示某个 app 呢 一种做法是在 app 中直接处理 一种做法是在 framework 中处理  
接下来看这两种处理方法

1, app 中处理  
为该应用 AndroidManifest xml 文件中主 MainActivity 设置属性

android:excludeFromRecents=“true”

例如：

```
 <activity android:
		android:excludeFromRecents="true" 
		android:label="@string/app_name"> 
		
		<intent-filter> 
			<action android: /> 
			<category android: /> 
		</intent-filter>	
 </activity>		

```

应用是否具有 android.intent.category.[LAUNCHER](https://so.csdn.net/so/search?q=LAUNCHER&spm=1001.2101.3001.7020) 属性有关，在主 Activity 有 LAUNCHER 的前提下，android:excludeFromRecents=“true”, 才能达到在最近任务列表中隐藏该应用的目的。

2. 在 framework 中处理的  
源码地址：frameworks/base/services/core/java/com/android/server/am/ActivityTaskManagerService.java

```
@Override
public ParceledListSlice<ActivityManager.RecentTaskInfo> getRecentTasks(int maxNum, int flags,
        int userId) {
    final int callingUid = Binder.getCallingUid();
    userId = handleIncomingUser(Binder.getCallingPid(), callingUid, userId, "getRecentTasks");
    final boolean allowed = isGetTasksAllowed("getRecentTasks", Binder.getCallingPid(),
            callingUid);
    final boolean detailed = checkGetTasksPermission(
            android.Manifest.permission.GET_DETAILED_TASKS, Binder.getCallingPid(),
            UserHandle.getAppId(callingUid))
            == PackageManager.PERMISSION_GRANTED;

    synchronized (mGlobalLock) {
        return mRecentTasks.getRecentTasks(maxNum, flags, allowed, detailed, userId,
                callingUid);
    }
}

```

最终调用 mRecentTasks.getRecentTasks（）；

源码地址：frameworks/base/services/core/java/com/android/server/am/RecentTasks.java

```
所以修改如下：

 private ArrayList<ActivityManager.RecentTaskInfo> getRecentTasksImpl(int maxNum, int flags,
            boolean getTasksAllowed, boolean getDetailedTasks, int userId, int callingUid) {
        final boolean withExcluded = (flags & RECENT_WITH_EXCLUDED) != 0;

        if (!isUserRunning(userId, FLAG_AND_UNLOCKED)) {
            Slog.i(TAG, "user " + userId + " is still locked. Cannot load recents");
            return new ArrayList<>();
        }
        loadUserRecentsLocked(userId);

        final Set<Integer> includedUsers = getProfileIds(userId);
        includedUsers.add(Integer.valueOf(userId));

        final ArrayList<ActivityManager.RecentTaskInfo> res = new ArrayList<>();
        final int size = mTasks.size();
        int numVisibleTasks = 0;
        for (int i = 0; i < size; i++) {
            final TaskRecord tr = mTasks.get(i);

            if (isVisibleRecentTask(tr)) {
                numVisibleTasks++;
                if (isInVisibleRange(tr, i, numVisibleTasks, withExcluded)) {
                    // Fall through
                } else {
                    // Not in visible range
                    continue;
                }
            } else {
                // Not visible
                continue;
            }

            // Skip remaining tasks once we reach the requested size
            if (res.size() >= maxNum) {
                continue;
            }

            // Only add calling user or related users recent tasks
            if (!includedUsers.contains(Integer.valueOf(tr.userId))) {
                if (DEBUG_RECENTS) Slog.d(TAG_RECENTS, "Skipping, not user: " + tr);
                continue;
            }

            if (tr.realActivitySuspended) {
                if (DEBUG_RECENTS) Slog.d(TAG_RECENTS, "Skipping, activity suspended: " + tr);
                continue;
            }

            if (!getTasksAllowed) {
                // If the caller doesn't have the GET_TASKS permission, then only
                // allow them to see a small subset of tasks -- their own and home.
                if (!tr.isActivityTypeHome() && tr.effectiveUid != callingUid) {
                    if (DEBUG_RECENTS) Slog.d(TAG_RECENTS, "Skipping, not allowed: " + tr);
                    continue;
                }
            }

            if (tr.autoRemoveRecents && tr.getTopActivity() == null) {
                // Don't include auto remove tasks that are finished or finishing.
                if (DEBUG_RECENTS) {
                    Slog.d(TAG_RECENTS, "Skipping, auto-remove without activity: " + tr);
                }
                continue;
            }
            if ((flags & RECENT_IGNORE_UNAVAILABLE) != 0 && !tr.isAvailable) {
                if (DEBUG_RECENTS) {
                    Slog.d(TAG_RECENTS, "Skipping, unavail real act: " + tr);
                }
                continue;
            }

            if (!tr.mUserSetupComplete) {
                // Don't include task launched while user is not done setting-up.
                if (DEBUG_RECENTS) {
                    Slog.d(TAG_RECENTS, "Skipping, user setup not complete: " + tr);
                }
                continue;
            }

            final ActivityManager.RecentTaskInfo rti = createRecentTaskInfo(tr);
            if (!getDetailedTasks) {
                rti.baseIntent.replaceExtras((Bundle) null);
            }

        -    res.add(rti);
       +                // add code start
    +                ComponentName cn = tr.intent.getComponent();
    +                if (cn != null && cn.getPackageName().equals("com.android.jni")) { //移除app的baom
    +                    Log.d(TAG," name "+cn.getPackageName());
    +                }else {
    +                   res.add(rti);
    +                }
    +                // add code end
        }
        return res;
    }

```
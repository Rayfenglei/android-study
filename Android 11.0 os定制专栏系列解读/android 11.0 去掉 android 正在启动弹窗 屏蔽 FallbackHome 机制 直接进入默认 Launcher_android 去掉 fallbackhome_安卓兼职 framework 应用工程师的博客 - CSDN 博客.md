> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124710885)

1. 概述
-----

在 11.0 的产品定制化开发中，由于产品要求不需要锁屏功能，所以在每次开启启动的过程中，都会弹窗显示  
android 正在启动，过 2 秒钟进入系统 Launcher3, 产品要求去掉这个弹窗直接进入系统 Launcher3 桌面

2. 去掉 android 正在启动弹窗 屏蔽 FallbackHome 机制 直接进入默认 Launcher 核心代码
------------------------------------------------------------

```
frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
frameworks/base/services/core/java/com/android/server/wm/ActivityRecord.java

```

3. 去掉 android 正在启动弹窗 屏蔽 FallbackHome 机制 直接进入默认 Launcher 功能分析和实现
---------------------------------------------------------------

核心功能分析如下：

FallbackHome 机制是为了在系统还没有解锁前 先进入 Setting 中的 android 正在启动弹窗 的页面  
等系统完全解锁后 然后进入默认 Launcher 但是弹窗也会影响产品效果 所以最后去掉这个弹窗 不显示[桌面壁纸](https://so.csdn.net/so/search?q=%E6%A1%8C%E9%9D%A2%E5%A3%81%E7%BA%B8&spm=1001.2101.3001.7020)  
直接进入 Launcher  
具体功能实现:

3.1 延长开机动画 在解锁后直接进去 Launcher
----------------------------

在 WindowManagerService.java 中，延时开机动画 注释掉结束开机动画相关代码  
路径: frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java

```
private void performEnableScreen() {
synchronized (mGlobalLock) {
if (DEBUG_BOOT) Slog.i(TAG_WM, "performEnableScreen: mDisplayEnabled=" + mDisplayEnabled
" mForceDisplayEnabled=" + mForceDisplayEnabled
" mShowingBootMessages=" + mShowingBootMessages
" mSystemBooted=" + mSystemBooted
" mOnlyCore=" + mOnlyCore,
new RuntimeException("here").fillInStackTrace());
if (mDisplayEnabled) {
return;
}
if (!mSystemBooted && !mShowingBootMessages) {
return;
}
if (!mShowingBootMessages && !mPolicy.canDismissBootAnimation()) {
return;
}
// Don't enable the screen until all existing windows have been drawn.
if (!mForceDisplayEnabled
// TODO(multidisplay): Expand to all displays?
&& getDefaultDisplayContentLocked().checkWaitingForWindows()) {
return;
}


// add core start 判断是否有设置锁屏密码来看是否执行结束开机动画的操作
LockPatternUtils lockPatternUtils = new LockPatternUtils(mContext);
if(lockPatternUtils.isSecure(mCurrentUserId)){
if (!mBootAnimationStopped) {
Trace.asyncTraceBegin(TRACE_TAG_WINDOW_MANAGER, "Stop bootanim", 0);
// stop boot animation
// formerly we would just kill the process, but we now ask it to exit so it
// can choose where to stop the animation.
SystemProperties.set("service.bootanim.exit", "1");
mBootAnimationStopped = true;
}
if (!mForceDisplayEnabled && !checkBootAnimationCompleteLocked()) {
if (DEBUG_BOOT) Slog.i(TAG_WM, "performEnableScreen: Waiting for anim complete");
return;
}
try {
IBinder surfaceFlinger = ServiceManager.getService("SurfaceFlinger");
if (surfaceFlinger != null) {
Slog.i(TAG_WM, "******* TELLING SURFACE FLINGER WE ARE BOOTED!");
Parcel data = Parcel.obtain();
data.writeInterfaceToken("android.ui.ISurfaceComposer");
surfaceFlinger.transact(IBinder.FIRST_CALL_TRANSACTION, // BOOT_FINISHED
data, null, 0);
data.recycle();
}
} catch (RemoteException ex) {
Slog.e(TAG_WM, "Boot completed: SurfaceFlinger is dead!");
}
}
// add core start


EventLog.writeEvent(EventLogTags.WM_BOOT_ANIMATION_DONE, SystemClock.uptimeMillis());
Trace.asyncTraceEnd(TRACE_TAG_WINDOW_MANAGER, "Stop bootanim", 0);
mDisplayEnabled = true;
if (DEBUG_SCREEN_ON || DEBUG_BOOT) Slog.i(TAG_WM, "******************** ENABLING SCREEN!");
// Enable input dispatch.
mInputManagerCallback.setEventDispatchingLw(mEventDispatchingEnabled);
}
try {
mActivityManager.bootAnimationComplete();
} catch (RemoteException e) {
}
mPolicy.enableScreenAfterBoot();
// Make sure the last requested orientation has been applied.
updateRotationUnchecked(false, false);
}
private boolean checkBootAnimationCompleteLocked() {
if (SystemService.isRunning(BOOT_ANIMATION_SERVICE)) {
mH.removeMessages(H.CHECK_IF_BOOT_ANIMATION_FINISHED);
mH.sendEmptyMessageDelayed(H.CHECK_IF_BOOT_ANIMATION_FINISHED,
BOOT_ANIMATION_POLL_INTERVAL);
if (DEBUG_BOOT) Slog.i(TAG_WM, "checkBootAnimationComplete: Waiting for anim complete");
return false;
}
if (DEBUG_BOOT) Slog.i(TAG_WM, "checkBootAnimationComplete: Animation complete!");
return true;
}
在这里处理播放完开机动画后是否退出开机动画
判断是否有设置锁屏密码来看是否执行结束开机动画的操作
if (!mBootAnimationStopped) {
Trace.asyncTraceBegin(TRACE_TAG_WINDOW_MANAGER, "Stop bootanim", 0);
// stop boot animation
// formerly we would just kill the process, but we now ask it to exit so it
// can choose where to stop the animation.
SystemProperties.set("service.bootanim.exit", "1");
mBootAnimationStopped = true;
}
if (!mForceDisplayEnabled && !checkBootAnimationCompleteLocked()) {
if (DEBUG_BOOT) Slog.i(TAG_WM, "performEnableScreen: Waiting for anim complete");
return;
}
try {
IBinder surfaceFlinger = ServiceManager.getService("SurfaceFlinger");
if (surfaceFlinger != null) {
Slog.i(TAG_WM, "******* TELLING SURFACE FLINGER WE ARE BOOTED!");
Parcel data = Parcel.obtain();
data.writeInterfaceToken("android.ui.ISurfaceComposer");
surfaceFlinger.transact(IBinder.FIRST_CALL_TRANSACTION, // BOOT_FINISHED
data, null, 0);
data.recycle();
}
} catch (RemoteException ex) {
Slog.e(TAG_WM, "Boot completed: SurfaceFlinger is dead!");
}

```

在 [AMS](https://so.csdn.net/so/search?q=AMS&spm=1001.2101.3001.7020) 启动完毕后会调用 WMS 的 performEnableScreen() 来通知开机动画结束播放，判断是否有设置锁屏密码来看是否执行结束开机动画的操作，有锁屏密码就执行上述的操作，无锁屏密码就不需要执行上述的操作  
解锁结束后退出开机动画

3.2 在 ActivityRecord 中 onWindowsDrawn( 退出动画 开启系统手势触摸功能
------------------------------------------------------

在 ActivityRecord 的源码中 , 一个 ActivityRecord 对应一个 Activity, 保存了一个 Activity 的所有信息, 所以在  
进入默认 Launcher 后依然会调用 onWindowsDrawn(boolean drawn, long timestamp) 来绘制相关页面  
所以可以在这里执行开机动画结束的动作，结束开机动画  
路径: frameworks/base/services/core/java/com/android/server/wm/ActivityRecord.java

```
public void onWindowsDrawn(boolean drawn, long timestamp) {
mDrawn = drawn;
if (!drawn) {
return;
}
final TransitionInfoSnapshot info = mStackSupervisor
.getActivityMetricsLogger().notifyWindowsDrawn(this, timestampNs);
final boolean validInfo = info != null;
final int windowsDrawnDelayMs = validInfo ? info.windowsDrawnDelayMs : INVALID_DELAY;
final @LaunchState int launchState = validInfo ? info.getLaunchState() : -1;
// The activity may have been requested to be invisible (another activity has been launched)
// so there is no valid info. But if it is the current top activity (e.g. sleeping), the
// invalid state is still reported to make sure the waiting result is notified.
if (validInfo || this == getDisplayArea().topRunningActivity()) {
mStackSupervisor.reportActivityLaunchedLocked(false /* timeout */, this,
windowsDrawnDelayMs, launchState);
mStackSupervisor.stopWaitingForActivityVisible(this, windowsDrawnDelayMs);
}
finishLaunchTickingLocked();
if (task != null) {
task.setHasBeenVisible(true);
}
 // add core start
 if (isHomeIntent(intent) && shortComponentName != null && !shortComponentName.contains("FallbackHome")) {
     SystemProperties.set("service.bootanim.exit", "1");
     android.util.Log.e("ActivityRecord","real home....." + shortComponentName);
	   LockPatternUtils mLockPatternUtils = new LockPatternUtils(mAtmService.mContext);
      if(!mLockPatternUtils.isSecure(mUserId)){
try {
IBinder surfaceFlinger = ServiceManager.getService("SurfaceFlinger");
if (surfaceFlinger != null) {
Slog.i(TAG_WM, "******* TELLING SURFACE FLINGER WE ARE BOOTED!");
Parcel data = Parcel.obtain();
data.writeInterfaceToken("android.ui.ISurfaceComposer");
surfaceFlinger.transact(IBinder.FIRST_CALL_TRANSACTION, // BOOT_FINISHED
data, null, 0);
data.recycle();
}
} catch (RemoteException ex) {
Slog.e(TAG, "Boot completed: SurfaceFlinger is dead!");
}
}
}
// add code end
}

```

在 onWindowsDrawn(boolean drawn, long timestamp) 中判断是否有设置锁屏密码来看是否执行结束开机动画的操作，有锁屏密码就执行上述的操作，无锁屏密码就不需要执行上述的操作

3.3 去掉 Settings 的 Android 正在启动… 弹窗
----------------------------------

在 Setting 中注释掉关于 Android 正在启动… 的相关页面功能

```
android:text=“@android:string/android_start_title” 就是 android is starting 的提示语
private final Runnable mProgressTimeoutRunnable = () -> {
        //View v = getLayoutInflater().inflate(
        //        R.layout.fallback_home_finishing_boot, null /* root */);
        /*setContentView(v);
        v.setAlpha(0f);
        v.animate()
                .alpha(1f)
                .setDuration(500)
                .setInterpolator(AnimationUtils.loadInterpolator(
                        this, android.R.interpolator.fast_out_slow_in))
                .start();
        getWindow().addFlags(LayoutParams.FLAG_KEEP_SCREEN_ON);*/
    };

```
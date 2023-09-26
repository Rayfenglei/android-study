> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124893936)

### 1. 概述

在 android 11.0 12.0 系统中开机时通过按音量减就会进入[安全模式](https://so.csdn.net/so/search?q=%E5%AE%89%E5%85%A8%E6%A8%A1%E5%BC%8F&spm=1001.2101.3001.7020) ，而有些客户不需要安全模式，所以就要去掉安全模式，通过查询资料发现，安全模式在 WMS 中有设置，这样就需要在进入 WMS 的时候去掉安全模式就可以了

### 2. 去掉系统安全模式（SAFE MODE）的相关代码

```
frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
frameworks/base/services/java/com/android/server/SystemServer.java

```

### 3. 去掉系统安全模式（SAFE MODE）的核心代码功能分析

### 3.1 SystemServer.java 关于是否启动安全模式的代码分析

```
   private void startOtherServices() {
          final Context context = mSystemContext;
          VibratorService vibrator = null;
          DynamicSystemService dynamicSystem = null;
          IStorageManager storageManager = null;
          NetworkManagementService networkManagement = null;  
 IpSecService ipSecService = null;
          NetworkStatsService networkStats = null;
          NetworkPolicyManagerService networkPolicy = null;
          ConnectivityService connectivity = null;
          NsdService serviceDiscovery = null;
          WindowManagerService wm = null;
          SerialService serial = null;
          NetworkTimeUpdateService networkTimeUpdater = null;
          InputManagerService inputManager = null;
          TelephonyRegistry telephonyRegistry = null;
          ConsumerIrService consumerIr = null;
          MmsServiceBroker mmsService = null;
          HardwarePropertiesManagerService hardwarePropertiesService = null;
  
          boolean disableSystemTextClassifier = SystemProperties.getBoolean(
                  "config.disable_systemtextclassifier", false);
  
          boolean disableNetworkTime = SystemProperties.getBoolean("config.disable_networktime",
                  false);
          boolean disableCameraService = SystemProperties.getBoolean("config.disable_cameraservice",
                  false);
          boolean disableSlices = SystemProperties.getBoolean("config.disable_slices", false);
          boolean enableLeftyService = SystemProperties.getBoolean("config.enable_lefty", false);
  
          boolean isEmulator = SystemProperties.get("ro.kernel.qemu").equals("1");
  
          boolean isWatch = context.getPackageManager().hasSystemFeature(
                  PackageManager.FEATURE_WATCH);
  
          boolean isArc = context.getPackageManager().hasSystemFeature(
                  "org.chromium.arc");
  
          boolean enableVrService = context.getPackageManager().hasSystemFeature(
                  PackageManager.FEATURE_VR_MODE_HIGH_PERFORMANCE);
  
          // For debugging RescueParty
          if (Build.IS_DEBUGGABLE && SystemProperties.getBoolean("debug.crash_system", false)) {
              throw new RuntimeException();
          }
  
          try {
              final String SECONDARY_ZYGOTE_PRELOAD = "SecondaryZygotePreload";
              // We start the preload ~1s before the webview factory preparation, to
              // ensure that it completes before the 32 bit relro process is forked
              // from the zygote. In the event that it takes too long, the webview
              // RELRO process will block, but it will do so without holding any locks.
              mZygotePreload = SystemServerInitThreadPool.get().submit(() -> {
                  try {
                      Slog.i(TAG, SECONDARY_ZYGOTE_PRELOAD);
                      TimingsTraceLog traceLog = new TimingsTraceLog(
                              SYSTEM_SERVER_TIMING_ASYNC_TAG, Trace.TRACE_TAG_SYSTEM_SERVER);
                      traceLog.traceBegin(SECONDARY_ZYGOTE_PRELOAD);
                      if (!Process.ZYGOTE_PROCESS.preloadDefault(Build.SUPPORTED_32_BIT_ABIS[0])) {
                          Slog.e(TAG, "Unable to preload default resources");
                      }
                      traceLog.traceEnd();
                  } catch (Exception ex) {
                      Slog.e(TAG, "Exception preloading default resources", ex);
                  }
              }, SECONDARY_ZYGOTE_PRELOAD);
  
              traceBeginAndSlog("StartKeyAttestationApplicationIdProviderService");
              ServiceManager.addService("sec_key_att_app_id_provider",
                      new KeyAttestationApplicationIdProviderService(context));
              traceEnd();
  
              traceBeginAndSlog("StartKeyChainSystemService");
              mSystemServiceManager.startService(KeyChainSystemService.class);
              traceEnd();
  
              traceBeginAndSlog("StartSchedulingPolicyService");
              ServiceManager.addService("scheduling_policy", new SchedulingPolicyService());
              traceEnd();
  		  
		final boolean safeMode = wm.detectSafeMode();
          if (safeMode) {
              // If yes, immediately turn on the global setting for airplane mode.
              // Note that this does not send broadcasts at this stage because
              // subsystems are not yet up. We will send broadcasts later to ensure
              // all listeners have the chance to react with special handling.
              Settings.Global.putInt(context.getContentResolver(),
                      Settings.Global.AIRPLANE_MODE_ON, 1);
          }

```

在 SystemServer 中的 startOtherServices(）中通过调用 wm.detectSafeMode() 判断是否进入安全模式  
接下来看下 [WMS](https://so.csdn.net/so/search?q=WMS&spm=1001.2101.3001.7020) 源码看是怎么判断是否进入安全模式的，然后从这里屏蔽掉进入安全模式的入口  
就可以了

### 3.2 WindowManagerService.java 关于是否进入安全模式的代码分析

frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java

```
 

public boolean detectSafeMode() {
        if (!mInputManagerCallback.waitForInputDevicesReady(
                INPUT_DEVICES_READY_FOR_SAFE_MODE_DETECTION_TIMEOUT_MILLIS)) {
            Slog.w(TAG_WM, "Devices still not ready after waiting "
                   + INPUT_DEVICES_READY_FOR_SAFE_MODE_DETECTION_TIMEOUT_MILLIS
                   + " milliseconds before attempting to detect safe mode.");
        }
 
        if (Settings.Global.getInt(
                mContext.getContentResolver(), Settings.Global.SAFE_BOOT_DISALLOWED, 0) != 0) {
            return false;
        }
 
        int menuState = mInputManager.getKeyCodeState(-1, InputDevice.SOURCE_ANY,
                KeyEvent.KEYCODE_MENU);
        int sState = mInputManager.getKeyCodeState(-1, InputDevice.SOURCE_ANY, KeyEvent.KEYCODE_S);
        int dpadState = mInputManager.getKeyCodeState(-1, InputDevice.SOURCE_DPAD,
                KeyEvent.KEYCODE_DPAD_CENTER);
        int trackballState = mInputManager.getScanCodeState(-1, InputDevice.SOURCE_TRACKBALL,
                InputManagerService.BTN_MOUSE);
        int volumeDownState = mInputManager.getKeyCodeState(-1, InputDevice.SOURCE_ANY,
                KeyEvent.KEYCODE_VOLUME_DOWN);
        mSafeMode = menuState > 0 || sState > 0 || dpadState > 0 || trackballState > 0
                || volumeDownState > 0;
        try {
            if (SystemProperties.getInt(ShutdownThread.REBOOT_SAFEMODE_PROPERTY, 0) != 0
                    || SystemProperties.getInt(ShutdownThread.RO_SAFEMODE_PROPERTY, 0) != 0) {
                mSafeMode = true;
                SystemProperties.set(ShutdownThread.REBOOT_SAFEMODE_PROPERTY, "");
            }
        } catch (IllegalArgumentException e) {
        }
        if (mSafeMode) {
            Log.i(TAG_WM, "SAFE MODE ENABLED (menu=" + menuState + " s=" + sState
                    + " dpad=" + dpadState + " trackball=" + trackballState + ")");
            // May already be set if (for instance) this process has crashed
            if (SystemProperties.getInt(ShutdownThread.RO_SAFEMODE_PROPERTY, 0) == 0) {
                SystemProperties.set(ShutdownThread.RO_SAFEMODE_PROPERTY, "1");
            }
        } else {
            Log.i(TAG_WM, "SAFE MODE not enabled");
        }
        mPolicy.setSafeMode(mSafeMode);
        return mSafeMode;
    }

```

从上面 detectSafeMode() 源码看出 return mSafeMode; 然后进入安全模式 不进入  
安全模式返回 false 就进不到安全模式

具体修改如下:

```
--- a/frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
+++ b/frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
@@ -4502,7 +4502,7 @@ public class WindowManagerService extends AbsWindowManagerService
             Log.i(TAG_WM, "SAFE MODE not enabled");
         }
         mPolicy.setSafeMode(mSafeMode);
-        return mSafeMode;
+        return false;
     }
 
     public void displayReady() {

```

### 3.3 长按电源键 弹出 关机 重新启动 屏幕截屏 对话框

关于在 SystemUI 长按关机和重新启动 进入安全模式的分析处理  
在 GlobalActionsDialog 中去掉长按事件改成短按事件

改成和短按事件就可以了  
具体实现如下:

```
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/globalactions/GlobalActionsDialog.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/globalactions/GlobalActionsDialog.java
@@ -503,11 +503,12 @@ public class GlobalActionsDialog implements DialogInterface.OnDismissListener,
 
         @Override
         public boolean onLongPress() {
-            UserManager um = (UserManager) mContext.getSystemService(Context.USER_SERVICE);
+            /*UserManager um = (UserManager) mContext.getSystemService(Context.USER_SERVICE);
             if (!um.hasUserRestriction(UserManager.DISALLOW_SAFE_BOOT)) {
                 mWindowManagerFuncs.reboot(true);
                 return true;
-            }
+            }*/
+                       mWindowManagerFuncs.shutdown();
             return false;
         }
 
@@ -608,11 +609,12 @@ public class GlobalActionsDialog implements DialogInterface.OnDismissListener,
 
         @Override
         public boolean onLongPress() {
-            UserManager um = (UserManager) mContext.getSystemService(Context.USER_SERVICE);
+            /*UserManager um = (UserManager) mContext.getSystemService(Context.USER_SERVICE);
             if (!um.hasUserRestriction(UserManager.DISALLOW_SAFE_BOOT)) {
                 mWindowManagerFuncs.reboot(true);
                 return true;
-            }
+            }*/
+                       mWindowManagerFuncs.reboot(false);
             return false;
         }

```
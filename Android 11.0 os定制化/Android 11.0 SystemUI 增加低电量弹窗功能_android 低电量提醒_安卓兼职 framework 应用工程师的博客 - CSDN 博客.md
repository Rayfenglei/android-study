> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125690117)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.SystemUI 中实现低电量弹窗功能分析](#t1)

[3.SystemUI 中实现低电量弹窗功能代码分析](#t2)

[3.1 SystemUI 启动 PowerUI 的相关主要代码分析](#t3)

[3.2 PowerUI 相关电量的分析](#t4)

[3.3    PowerNotificationWarnings.java 的 showLowBatteryWarning（）增加低电量弹窗](#t5)

[3.4 自定义低电量提醒资源](#t6)

[3.5. 在 symbols 文件中添加对应 java-symbol 方便 Framework 代码引用 code](#t7)

1. 概述
-----

在开发 11.0 的产品时，对于低电量提醒也是个很好的体验，由于产品要求在低电量的时候增加个弹窗提醒用户电量低及时充电，所以就开发了这个功能

2.[SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 中实现低电量弹窗功能分析
-------------------------------------------------------------------------------------------

在 SystemUI 中，PowerUI 是 SystemUI 控制电量提醒的模块，包括低电量提醒、危急电量关机提醒、高温关机提醒、省电模式等功能，实现低电量弹窗功能就是在这里实现

  
3.SystemUI 中实现低电量弹窗功能代码分析
----------------------------

#### 3.1 SystemUI 启动 PowerUI 的相关主要代码分析

```
public class SystemUIService extends Service {
 
    @Override
    public void onCreate() {
        super.onCreate();
        ((SystemUIApplication) getApplication()).startServicesIfNeeded();
 
        // For debugging RescueParty
        if (Build.IS_DEBUGGABLE && SystemProperties.getBoolean("debug.crash_sysui", false)) {
            throw new RuntimeException();
        }
 
        if (Build.IS_DEBUGGABLE) {
            // b/71353150 - looking for leaked binder proxies
            BinderInternal.nSetBinderProxyCountEnabled(true);
            BinderInternal.nSetBinderProxyCountWatermarks(1000,900);
            BinderInternal.setBinderProxyCountCallback(
                    new BinderInternal.BinderProxyLimitListener() {
                        @Override
                        public void onLimitReached(int uid) {
                            Slog.w(SystemUIApplication.TAG,
                                    "uid " + uid + " sent too many Binder proxies to uid "
                                    + Process.myUid());
                        }
                    }, Dependency.get(Dependency.MAIN_HANDLER));
        }
    }
  在SystemUIService 启动时，启动SystemUIApplication的startServicesIfNeeded() 来启动SystemUI的各种服务
  在SystemUIApplication中 启动PowerUI
  config.xml 中config_systemUIServiceComponents
  <string-array >
        <item>com.android.systemui.Dependency$DependencyCreator</item>
        <item>com.android.systemui.util.NotificationChannels</item>
        <item>com.android.systemui.statusbar.CommandQueue$CommandQueueStart</item>
        <item>com.android.systemui.keyguard.KeyguardViewMediator</item>
        <item>com.android.systemui.recents.Recents</item>
        <item>com.android.systemui.volume.VolumeUI</item>
        <item>com.android.systemui.stackdivider.Divider</item>
        <item>com.android.systemui.SystemBars</item>
        <item>com.android.systemui.usb.StorageNotification</item>
        <item>com.android.systemui.power.PowerUI</item>
        <item>com.android.systemui.media.RingtonePlayer</item>
        <item>com.android.systemui.keyboard.KeyboardUI</item>
        <item>com.android.systemui.pip.PipUI</item>
        <item>com.android.systemui.shortcut.ShortcutKeyDispatcher</item>
        <item>@string/config_systemUIVendorServiceComponent</item>
        <item>com.android.systemui.util.leak.GarbageMonitor$Service</item>
        <item>com.android.systemui.LatencyTester</item>
        <item>com.android.systemui.globalactions.GlobalActionsComponent</item>
        <item>com.android.systemui.ScreenDecorations</item>
        <item>com.android.systemui.biometrics.BiometricDialogImpl</item>
        <item>com.android.systemui.SliceBroadcastRelayHandler</item>
        <item>com.android.systemui.SizeCompatModeActivityController</item>
        <item>com.android.systemui.statusbar.notification.InstantAppNotifier</item>
        <item>com.android.systemui.theme.ThemeOverlayController</item>
    </string-array>
      /**
     * Makes sure that all the SystemUI services are running. If they are already running, this is a
     * no-op. This is needed to conditinally start all the services, as we only need to have it in
     * the main process.
     * <p>This method must only be called from the main thread.</p>
     */
    public void startServicesIfNeeded() {
        String[] names = SystemUIFactory.getInstance().getSystemUIServiceComponents(getResources());
        startServicesIfNeeded(/* metricsPrefix= */ "StartServices", names);
    }
   SystemUIFactory的getSystemUIServiceComponents(Resources resources)
   public String[] getSystemUIServiceComponents(Resources resources) {
          return resources.getStringArray(R.array.config_systemUIServiceComponents);
     }
   private void startServicesIfNeeded(String[] services) {
        if (mServicesStarted) {
            return;
        }
        mServices = new SystemUI[services.length];
 
        if (!mBootCompleted) {
            // check to see if maybe it was already completed long before we began
            // see ActivityManagerService.finishBooting()
            if ("1".equals(SystemProperties.get("sys.boot_completed"))) {
                mBootCompleted = true;
                if (DEBUG) Log.v(TAG, "BOOT_COMPLETED was already sent");
            }
        }
 
        Log.v(TAG, "Starting SystemUI services for user " +
                Process.myUserHandle().getIdentifier() + ".");
        TimingsTraceLog log = new TimingsTraceLog("SystemUIBootTiming",
                Trace.TRACE_TAG_APP);
        log.traceBegin("StartServices");
        final int N = services.length;
        for (int i = 0; i < N; i++) {
            String clsName = services[i];
            if (DEBUG) Log.d(TAG, "loading: " + clsName);
            log.traceBegin("StartServices" + clsName);
            long ti = System.currentTimeMillis();
            Class cls;
            try {
                cls = Class.forName(clsName);
                Object o = cls.newInstance();
                if (o instanceof SystemUI.Injector) {
                    o = ((SystemUI.Injector) o).apply(this);
                }
                mServices[i] = (SystemUI) o;
            } catch(ClassNotFoundException ex){
                throw new RuntimeException(ex);
            } catch (IllegalAccessException ex) {
                throw new RuntimeException(ex);
            } catch (InstantiationException ex) {
                throw new RuntimeException(ex);
            }
 
            mServices[i].mContext = this;
            mServices[i].mComponents = mComponents;
            if (DEBUG) Log.d(TAG, "running: " + mServices[i]);
            mServices[i].start();
            log.traceEnd();
 
            // Warn if initialization of component takes too long
            ti = System.currentTimeMillis() - ti;
            if (ti > 1000) {
                Log.w(TAG, "Initialization of " + cls.getName() + " took " + ti + " ms");
            }
            if (mBootCompleted) {
                mServices[i].onBootCompleted();
            }
        }
        Dependency.get(InitController.class).executePostInitTasks();
        log.traceEnd();
        final Handler mainHandler = new Handler(Looper.getMainLooper());
        Dependency.get(PluginManager.class).addPluginListener(
                new PluginListener<OverlayPlugin>() {
                    private ArraySet<OverlayPlugin> mOverlays = new ArraySet<>();
 
                    @Override
                    public void onPluginConnected(OverlayPlugin plugin, Context pluginContext) {
                        mainHandler.post(new Runnable() {
                            @Override
                            public void run() {
                                StatusBar statusBar = getComponent(StatusBar.class);
                                if (statusBar != null) {
                                    plugin.setup(statusBar.getStatusBarWindow(),
                                            statusBar.getNavigationBarView(), new Callback(plugin));
                                }
                            }
                        });
                    }
 
                    @Override
                    public void onPluginDisconnected(OverlayPlugin plugin) {
                        mainHandler.post(new Runnable() {
                            @Override
                            public void run() {
                                mOverlays.remove(plugin);
                                Dependency.get(StatusBarWindowController.class).setForcePluginOpen(
                                        mOverlays.size() != 0);
                            }
                        });
                    }
 
                    class Callback implements OverlayPlugin.Callback {
                        private final OverlayPlugin mPlugin;
 
                        Callback(OverlayPlugin plugin) {
                            mPlugin = plugin;
                        }
 
                        @Override
                        public void onHoldStatusBarOpenChange() {
                            if (mPlugin.holdStatusBarOpen()) {
                                mOverlays.add(mPlugin);
                            } else {
                                mOverlays.remove(mPlugin);
                            }
                            mainHandler.post(new Runnable() {
                                @Override
                                public void run() {
                                    Dependency.get(StatusBarWindowController.class)
                                            .setStateListener(b -> mOverlays.forEach(
                                                    o -> o.setCollapseDesired(b)));
                                    Dependency.get(StatusBarWindowController.class)
                                            .setForcePluginOpen(mOverlays.size() != 0);
                                }
                            });
                        }
                    }
                }, OverlayPlugin.class, true /* Allow multiple plugins */);
 
        mServicesStarted = true;
    }
```

#### 3.2 PowerUI 相关电量的分析

```
public void start() {
mPowerManager = (PowerManager) mContext.getSystemService(Context.POWER_SERVICE);
mScreenOffTime = mPowerManager.isScreenOn() ? -1 : SystemClock.elapsedRealtime();
mWarnings = Dependency.get(WarningsUI.class);
mEnhancedEstimates = Dependency.get(EnhancedEstimates.class);
mLastConfiguration.setTo(mContext.getResources().getConfiguration());
 
ContentObserver obs = new ContentObserver(mHandler) {
@Override
public void onChange(boolean selfChange) {
updateBatteryWarningLevels();
}
};
final ContentResolver resolver = mContext.getContentResolver();
resolver.registerContentObserver(Settings.Global.getUriFor(
Settings.Global.LOW_POWER_MODE_TRIGGER_LEVEL),
false, obs, UserHandle.USER_ALL);
updateBatteryWarningLevels();
mReceiver.init();
 
// Check to see if we need to let the user know that the phone previously shut down due
// to the temperature being too high.
showWarnOnThermalShutdown();
 
// Register an observer to configure mEnableSkinTemperatureWarning and perform the
// registration of skin thermal event listener upon Settings change.
resolver.registerContentObserver(
Settings.Global.getUriFor(Settings.Global.SHOW_TEMPERATURE_WARNING),
false /*notifyForDescendants*/,
new ContentObserver(mHandler) {
@Override
public void onChange(boolean selfChange) {
doSkinThermalEventListenerRegistration();
}
});
// Register an observer to configure mEnableUsbTemperatureAlarm and perform the
// registration of usb thermal event listener upon Settings change.
resolver.registerContentObserver(
Settings.Global.getUriFor(Settings.Global.SHOW_USB_TEMPERATURE_ALARM),
false /*notifyForDescendants*/,
new ContentObserver(mHandler) {
@Override
public void onChange(boolean selfChange) {
doUsbThermalEventListenerRegistration();
}
});
initThermalEventListeners();
mCommandQueue.addCallback(this);
}
 
@VisibleForTesting
final class Receiver extends BroadcastReceiver {
 
private boolean mHasReceivedBattery = false;
 
public void init() {
// Register for Intent broadcasts for...
IntentFilter filter = new IntentFilter();
filter.addAction(PowerManager.ACTION_POWER_SAVE_MODE_CHANGED);
filter.addAction(Intent.ACTION_BATTERY_CHANGED);
filter.addAction(Intent.ACTION_SCREEN_OFF);
filter.addAction(Intent.ACTION_SCREEN_ON);
filter.addAction(Intent.ACTION_USER_SWITCHED);
mBroadcastDispatcher.registerReceiverWithHandler(this, filter, mHandler);
// Force get initial values. Relying on Sticky behavior until API for getting info.
if (!mHasReceivedBattery) {
// Get initial state
Intent intent = mContext.registerReceiver(
null,
new IntentFilter(Intent.ACTION_BATTERY_CHANGED)
);
if (intent != null) {
onReceive(mContext, intent);
}
}
}
 
@Override
public void onReceive(Context context, Intent intent) {
String action = intent.getAction();
if (PowerManager.ACTION_POWER_SAVE_MODE_CHANGED.equals(action)) {
ThreadUtils.postOnBackgroundThread(() -> {
if (mPowerManager.isPowerSaveMode()) {
mWarnings.dismissLowBatteryWarning();
}
});
} else if (Intent.ACTION_BATTERY_CHANGED.equals(action)) {
mHasReceivedBattery = true;
final int oldBatteryLevel = mBatteryLevel;
mBatteryLevel = intent.getIntExtra(BatteryManager.EXTRA_LEVEL, 100);
final int oldBatteryStatus = mBatteryStatus;
mBatteryStatus = intent.getIntExtra(BatteryManager.EXTRA_STATUS,
BatteryManager.BATTERY_STATUS_UNKNOWN);
final int oldPlugType = mPlugType;
mPlugType = intent.getIntExtra(BatteryManager.EXTRA_PLUGGED, 1);
final int oldInvalidCharger = mInvalidCharger;
mInvalidCharger = intent.getIntExtra(BatteryManager.EXTRA_INVALID_CHARGER, 0);
mLastBatteryStateSnapshot = mCurrentBatteryStateSnapshot;
 
final boolean plugged = mPlugType != 0;
final boolean oldPlugged = oldPlugType != 0;
 
int oldBucket = findBatteryLevelBucket(oldBatteryLevel);
int bucket = findBatteryLevelBucket(mBatteryLevel);
 
if (DEBUG) {
Slog.d(TAG, "buckets   ....." + mLowBatteryAlertCloseLevel
+ " .. " + mLowBatteryReminderLevels[0]
+ " .. " + mLowBatteryReminderLevels[1]);
Slog.d(TAG, "level          " + oldBatteryLevel + " --> " + mBatteryLevel);
Slog.d(TAG, "status         " + oldBatteryStatus + " --> " + mBatteryStatus);
Slog.d(TAG, "plugType       " + oldPlugType + " --> " + mPlugType);
Slog.d(TAG, "invalidCharger " + oldInvalidCharger + " --> " + mInvalidCharger);
Slog.d(TAG, "bucket         " + oldBucket + " --> " + bucket);
Slog.d(TAG, "plugged        " + oldPlugged + " --> " + plugged);
}
 
mWarnings.update(mBatteryLevel, bucket, mScreenOffTime);
if (oldInvalidCharger == 0 && mInvalidCharger != 0) {
Slog.d(TAG, "showing invalid charger warning");
mWarnings.showInvalidChargerWarning();
return;
} else if (oldInvalidCharger != 0 && mInvalidCharger == 0) {
mWarnings.dismissInvalidChargerWarning();
} else if (mWarnings.isInvalidChargerWarningShowing()) {
// if invalid charger is showing, don't show low battery
if (DEBUG) {
Slog.d(TAG, "Bad Charger");
}
return;
}
// Show the correct version of low battery warning if needed
if (mLastShowWarningTask != null) {
mLastShowWarningTask.cancel(true);
if (DEBUG) {
Slog.d(TAG, "cancelled task");
}
}
//
mLastShowWarningTask = ThreadUtils.postOnBackgroundThread(() -> {
maybeShowBatteryWarningV2(
plugged, bucket);
});
} else if (Intent.ACTION_SCREEN_OFF.equals(action)) {
mScreenOffTime = SystemClock.elapsedRealtime();
} else if (Intent.ACTION_SCREEN_ON.equals(action)) {
mScreenOffTime = -1;
} else if (Intent.ACTION_USER_SWITCHED.equals(action)) {
mWarnings.userSwitched();
} else {
Slog.w(TAG, "unknown intent: " + intent);
}
}
}
在PowerUI的start()方法中启动广播监听电量变化
mLastShowWarningTask = ThreadUtils.postOnBackgroundThread(() -> {
maybeShowBatteryWarningV2(
plugged, bucket);
});
来判断是否开启低电量警告
protected void maybeShowBatteryWarningV2(boolean plugged, int bucket) {
final boolean hybridEnabled = mEnhancedEstimates.isHybridNotificationEnabled();
final boolean isPowerSaverMode = mPowerManager.isPowerSaveMode();
// Stick current battery state into an immutable container to determine if we should show
// a warning.
if (DEBUG) {
Slog.d(TAG, "evaluating which notification to show");
}
if (hybridEnabled) {
if (DEBUG) {
Slog.d(TAG, "using hybrid");
}
Estimate estimate = refreshEstimateIfNeeded();
mCurrentBatteryStateSnapshot = new BatteryStateSnapshot(mBatteryLevel, isPowerSaverMode,
plugged, bucket, mBatteryStatus, mLowBatteryReminderLevels[1],
mLowBatteryReminderLevels[0], estimate.getEstimateMillis(),
estimate.getAverageDischargeTime(),
mEnhancedEstimates.getSevereWarningThreshold(),
mEnhancedEstimates.getLowWarningThreshold(), estimate.isBasedOnUsage(),
mEnhancedEstimates.getLowWarningEnabled());
} else {
if (DEBUG) {
Slog.d(TAG, "using standard");
}
mCurrentBatteryStateSnapshot = new BatteryStateSnapshot(mBatteryLevel, isPowerSaverMode,
plugged, bucket, mBatteryStatus, mLowBatteryReminderLevels[1],
mLowBatteryReminderLevels[0]);
}
mWarnings.updateSnapshot(mCurrentBatteryStateSnapshot);
if (mCurrentBatteryStateSnapshot.isHybrid()) {
maybeShowHybridWarning(mCurrentBatteryStateSnapshot, mLastBatteryStateSnapshot);
} else {
//低电量警告
maybeShowBatteryWarning(mCurrentBatteryStateSnapshot, mLastBatteryStateSnapshot);
}
protected void maybeShowBatteryWarning(
              BatteryStateSnapshot currentSnapshot,
             BatteryStateSnapshot lastSnapshot) {
         final boolean playSound = currentSnapshot.getBucket() != lastSnapshot.getBucket()
                  || lastSnapshot.getPlugged();
  
         if (shouldShowLowBatteryWarning(currentSnapshot, lastSnapshot)) {
              mWarnings.showLowBatteryWarning(playSound);//低电量警告
          } else if (shouldDismissLowBatteryWarning(currentSnapshot, lastSnapshot)) {
              mWarnings.dismissLowBatteryWarning();//去掉低电量警告
          } else {
              mWarnings.updateLowBatteryWarning();
          }
      }
  PowerNotificationWarnings.java的低电量提醒方法
       @Override
      public void showLowBatteryWarning(boolean playSound) {
          Slog.i(TAG,
                  "show low battery warning: level=" + mBatteryLevel
                         + " [" + mBucket + "] playSound=" + playSound);
          mPlaySound = playSound;
          mWarning = true;
          updateNotification();
      }
}
```

#### 3.3    PowerNotificationWarnings.java 的 showLowBatteryWarning（）增加低电量弹窗

```
import android.app.AlertDialog;
    import android.view.WindowManager;
    import android.content.DialogInterface;
    private AlertDialog mbatteryLowDialog = null;
      // 自定义电池温度Dialog弹窗
    private void batterylowDialog(String lowbattery) {
        mbatteryLowDialog = new AlertDialog(mContext);
        AlertDialog.Builder builder = new AlertDialog.Builder(mContext);
        builder.setTitle(mContext.getResources().getString(
                com.android.internal.R.string.lowbatteryWarning));
        builder.setCancelable(false);
        builder.setMessage(lowbattery);
        builder.setIconAttribute(android.R.attr.alertDialogIcon);
        builder.setPositiveButton(com.android.internal.R.string.batteryLowOk,
                new DialogInterface.OnClickListener() {
                    public void onClick(DialogInterface dialog, int id) {
                        dialog.cancel();
                        mbatteryLowDialog = null;
                    }
                });
        mbatteryLowDialog = builder.create();
        mbatteryLowDialog.getWindow().setType(
                WindowManager.LayoutParams.TYPE_SYSTEM_ALERT);
        if (mbatteryLowDialog != null&& !mbatteryLowDialog.isShowing()) {
            mbatteryLowDialog.show();
        }
 
    }
 
       @Override
      public void showLowBatteryWarning(boolean playSound) {
          Slog.i(TAG,
                  "show low battery warning: level=" + mBatteryLevel
                         + " [" + mBucket + "] playSound=" + playSound);
          mPlaySound = playSound;
          mWarning = true;
          updateNotification();
        + batterylowDialog("低电量")
      }
```

####   
3.4 自定义低电量提醒资源

```
主要修改
frameworks/base/core/res/res/values/string.xml
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:xliff="urn:oasis:names:tc:xliff:document:1.2">
    ... ...
    <!-- Shutdown if the battery temperature exceeds (this value * 0.1) Celsius. -->
    <string >低电量提醒</integer>
    <!-- add code start-->
    <string >ok</integer>
    <!-- add code end-->
    ... ...
</resources>
```

#### 3.5. 在 symbols 文件中添加对应 java-symbol 方便 Framework 代码引用 code

```
在symbols文件中为全部string，int值注册，方便Framework层代码的引用。
frameworks/base/core/res/res/values/symbols.xml
 
<?xml version="1.0" encoding="utf-8"?>
<resources>
  <!-- Private symbols that we need to reference from framework code.  See
       frameworks/base/core/res/MakeJavaSymbols.sed for how to easily generate
       this.
 
       Can be referenced in java code as: com.android.internal.R.<type>.<name>
       and in layout xml as: "@*android:<type>/<name>"
  -->
 
  <!-- add code start-->
  <java-symbol type="string"  />
  <java-symbol type="string"  />
```
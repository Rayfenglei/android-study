> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125305786)

**目录**

[1. 简述](#1.%E7%AE%80%E8%BF%B0)

[2. 电池电量为 0 延迟关机的核心代码](#t1)

[3. 电池电量为 0 延迟关机的功能分析和实现](#t2)

[3.1 分析问题](#t3)

[3.2 问题分析](#%E9%97%AE%E9%A2%98%E5%88%86%E6%9E%90)

                  [3.3 解决问题方法](#t4)

1. 简述
-----

在 11.0 定制化开发中，遇到了在电池电量为 0 时，延时关机的问题，下面就来分析问题所在，然后解决这个问题

2. 电池电量为 0 延迟关机的核心代码
--------------------

```
 /frameworks/base/services/core/java/com/android/server/BatteryService.java

```

3. 电池电量为 0 延迟关机的功能分析和实现
-----------------------

3.1 分析问题
--------

在 11.0 中 电池电量管理都是由 BatteryService.java 来管理的，电池电量显示更新电量的变化都是在这里处理的, 当底层驱动检测到电池电量变化时，会上报事件然后更新电池状态

```
public final class BatteryService extends SystemService {
	private static final Boolean DEBUG = false;
	private static final int BATTERY_SCALE = 100;
	// battery capacity is a percentage
	private static final long HEALTH_HAL_WAIT_MS = 1000;
	private static final long BATTERY_LEVEL_CHANGE_THROTTLE_MS = 60_000;
	private static final int MAX_BATTERY_LEVELS_QUEUE_SIZE = 100;
	// Used locally for determining when to make a last ditch effort to log
	// discharge stats before the device dies.
	private int mCriticalBatteryLevel;
	// TODO: Current args don't work since "--unplugged" flag was purposefully removed.
	private static final String[] DUMPSYS_ARGS = new String[] { "--checkin", "--unplugged" };
	private static final String DUMPSYS_DATA_PATH = "/data/system/";
	// This should probably be exposed in the API, though it's not critical
	private static final int BATTERY_PLUGGED_NONE = OsProtoEnums.BATTERY_PLUGGED_NONE;
	// = 0
	private final Context mContext;
	private final IBatteryStats mBatteryStats;
	BinderService mBinderService;
	private final Handler mHandler;
	private final Object mLock = new Object();
	private HealthInfo mHealthInfo;
	private final HealthInfo mLastHealthInfo = new HealthInfo();
	private android.hardware.health.V2_1.HealthInfo mHealthInfo2p1;
	private Boolean mBatteryLevelCritical;
	private int mLastBatteryStatus;
	private int mLastBatteryHealth;
	private Boolean mLastBatteryPresent;
	private int mLastBatteryLevel;
	private int mLastBatteryVoltage;
	private int mLastBatteryTemperature;
	private Boolean mLastBatteryLevelCritical;
	private int mLastMaxChargingCurrent;
	private int mLastMaxChargingVoltage;
	private int mLastChargeCounter;
	private int mSequence = 1;
	private int mInvalidCharger;
	private int mLastInvalidCharger;
	private int mLowBatteryWarningLevel;
	private int mLowBatteryCloseWarningLevel;
	private int mShutdownBatteryTemperature;
	private int mPlugType;
	private int mLastPlugType = -1;
	// Extra state so we can detect first run
	private Boolean mBatteryLevelLow;
	private long mDischargeStartTime;
	private int mDischargeStartLevel;
	private long mChargeStartTime;
	private int mChargeStartLevel;
	private Boolean mUpdatesStopped;
	private Led mLed;
	private Boolean mSentLowBatteryBroadcast = false;
	private ActivityManagerInternal mActivityManagerInternal;
	private HealthServiceWrapper mHealthServiceWrapper;
	private HealthHalCallback mHealthHalCallback;
	private BatteryPropertiesRegistrar mBatteryPropertiesRegistrar;
	private ArrayDeque<Bundle> mBatteryLevelsEventQueue;
	private long mLastBatteryLevelChangedSentMs;
	private MetricsLogger mMetricsLogger;
	public BatteryService(Context context) {
		super(context);
		mContext = context;
		mHandler = new Handler(true 
		/*async*/
		);
		mLed = new Led(context, getLocalService(LightsManager.class));
		mBatteryStats = BatteryStatsService.getService();
		mActivityManagerInternal = LocalServices.getService(ActivityManagerInternal.class);
		mCriticalBatteryLevel = mContext.getResources().getInteger(
		com.android.internal.R.integer.config_criticalBatteryWarningLevel);
		mLowBatteryWarningLevel = mContext.getResources().getInteger(
		com.android.internal.R.integer.config_lowBatteryWarningLevel);
		mLowBatteryCloseWarningLevel = mLowBatteryWarningLevel + mContext.getResources().getInteger(
		com.android.internal.R.integer.config_lowBatteryCloseWarningBump);
		mShutdownBatteryTemperature = mContext.getResources().getInteger(
		com.android.internal.R.integer.config_shutdownBatteryTemperature);
		mBatteryLevelsEventQueue = new ArrayDeque<>();
		mMetricsLogger = new MetricsLogger();
		// watch for invalid charger messages if the invalid_charger switch exists
		if (new File("/sys/devices/virtual/switch/invalid_charger/state").exists()) {
			UEventObserver invalidChargerObserver = new UEventObserver() {
				@Override
				public void onUEvent(UEvent event) {
					final int invalidCharger = "1".equals(event.get("SWITCH_STATE")) ? 1 : 0;
					synchronized (mLock) {
						if (mInvalidCharger != invalidCharger) {
							mInvalidCharger = invalidCharger;
						}
					}
				}
			}
			;
			invalidChargerObserver.startObserving(
			"DEVPATH=/devices/virtual/switch/invalid_charger");
		}
	}
 
	@Override
	public void onStart() {
		registerHealthCallback();
		mBinderService = new BinderService();
		publishBinderService("battery", mBinderService);
		mBatteryPropertiesRegistrar = new BatteryPropertiesRegistrar();
		publishBinderService("batteryproperties", mBatteryPropertiesRegistrar);
		publishLocalService(BatteryManagerInternal.class, new LocalService());
	}
 
	@Override
	public void onBootPhase(int phase) {
		if (phase == PHASE_ACTIVITY_MANAGER_READY) {
			// check our power situation now that it is safe to display the shutdown dialog.
			synchronized (mLock) {
				ContentObserver obs = new ContentObserver(mHandler) {
					@Override
					public void onChange(Boolean selfChange) {
						synchronized (mLock) {
							updateBatteryWarningLevelLocked();
						}
					}
				}
				;
				final ContentResolver resolver = mContext.getContentResolver();
				resolver.registerContentObserver(Settings.Global.getUriFor(
				Settings.Global.LOW_POWER_MODE_TRIGGER_LEVEL),
				false, obs, UserHandle.USER_ALL);
				updateBatteryWarningLevelLocked();
			}
		}
	}
 
```

#### updateBatteryWarningLevelLocked() 根据电量变化更新电量

```
	private void updateBatteryWarningLevelLocked() {
		final ContentResolver resolver = mContext.getContentResolver();
		int defWarnLevel = mContext.getResources().getInteger(
		com.android.internal.R.integer.config_lowBatteryWarningLevel);
		mLowBatteryWarningLevel = Settings.Global.getint(resolver,
		Settings.Global.LOW_POWER_MODE_TRIGGER_LEVEL, defWarnLevel);
		if (mLowBatteryWarningLevel == 0) {
			mLowBatteryWarningLevel = defWarnLevel;
		}
		if (mLowBatteryWarningLevel < mCriticalBatteryLevel) {
			mLowBatteryWarningLevel = mCriticalBatteryLevel;
		}
		mLowBatteryCloseWarningLevel = mLowBatteryWarningLevel + mContext.getResources().getInteger(
		com.android.internal.R.integer.config_lowBatteryCloseWarningBump);
		processValuesLocked(true);
	}
	private void shutdownIfNoPowerLocked() {
		// shut down gracefully if our battery is critically low and we are not powered.
		// wait until the system has booted before attempting to display the shutdown dialog.
		if (shouldShutdownLocked()) {
			mHandler.post(new Runnable() {
				@Override
				public void run() {
					if (mActivityManagerInternal.isSystemReady()) {
						Intent intent = new Intent(Intent.ACTION_REQUEST_SHUTDOWN);
						intent.putExtra(Intent.EXTRA_KEY_CONFIRM, false);
						intent.putExtra(Intent.EXTRA_REASON,
						PowerManager.SHUTDOWN_LOW_BATTERY);
						intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
						mContext.startActivityAsUser(intent, UserHandle.CURRENT);
					}
				}
			}
			);
		}
	}
```

#### 3.2 问题分析

BatteryService.[java 构造方法](https://so.csdn.net/so/search?q=java%E6%9E%84%E9%80%A0%E6%96%B9%E6%B3%95&spm=1001.2101.3001.7020)中，获取电池的状态，监听电池电量 温度异常等相关信息  
通过 ContentResolver 来监听电量变化情况，然后更新电池电量和异常的相关警告弹窗等等信息

从上述代码分析，就是由于 shouldShutdownLocked() 而判断关机的相关状态是否立即关机还是延迟关机，所以就是在这里导致的延迟关机, 在 shouldShutdownLocked() 判断是否需要关机是否延迟

经过分析发现这个方法导致延迟

```
     private boolean shouldShutdownLocked() {
          if (mHealthInfo2p1.batteryCapacityLevel != BatteryCapacityLevel.UNSUPPORTED) {
             return (mHealthInfo2p1.batteryCapacityLevel == BatteryCapacityLevel.CRITICAL);
          }
          if (mHealthInfo.batteryLevel > 0) {
              return false;
         }
  
          // Battery-less devices should not shutdown.
          if (!mHealthInfo.batteryPresent) {
              return false;
          }
 
          // If battery state is not CHARGING, shutdown.
          // - If battery present and state == unknown, this is an unexpected error state.
          // - If level <= 0 and state == full, this is also an unexpected state
          // - All other states (NOT_CHARGING, DISCHARGING) means it is not charging.
          return mHealthInfo.batteryStatus != BatteryManager.BATTERY_STATUS_CHARGING;
     }
```

根据 shouldShutdownLocked() 返回 true 导致延迟，所以需要去掉这个判断条件 所以需要修改返回条件即可实现功能
--------------------------------------------------------------------

3.3 解决问题方法
----------

```
private void shutdownIfNoPowerLocked() {
         // shut down gracefully if our battery is critically low and we are not powered.
         // wait until the system has booted before attempting to display the shutdown dialog.
-        if (shouldShutdownLocked()) {
+        //if (shouldShutdownLocked()) {
+        if (mHealthInfo.batteryLevel == 0 && !isPoweredLocked(BatteryManager.BATTERY_PLUGGED_ANY)) {
             mHandler.post(new Runnable() {
                 @Override
                 public void run() {
if (mActivityManagerInternal.isSystemReady()) {
Intent intent = new Intent(Intent.ACTION_REQUEST_SHUTDOWN);
intent.putExtra(Intent.EXTRA_KEY_CONFIRM, false);
intent.putExtra(Intent.EXTRA_REASON,
PowerManager.SHUTDOWN_LOW_BATTERY);
intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
mContext.startActivityAsUser(intent, UserHandle.CURRENT);
}
}
});
}
}
```
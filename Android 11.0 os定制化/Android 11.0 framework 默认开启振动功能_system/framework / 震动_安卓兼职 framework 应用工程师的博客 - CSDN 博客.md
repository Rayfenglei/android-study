> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124754609)

### 1. 概述

在 11.0 系统定制化中，在系统 settings 的声音菜单下 有一个开启震动的 功能开关，默认是关闭的，由于项目的需要要求  
开启震动功能，所以就要在 framework [源码](https://so.csdn.net/so/search?q=%E6%BA%90%E7%A0%81&spm=1001.2101.3001.7020)震动开关是怎么打开的，然后开启 完成功能开发需求

### 2.framework 默认开启振动功能的核心类

```
framework/base/services/core/java/com/android/server/VibratorService.java
device/sprd/sharkl5/ums512_2h10/system.prop

```

### 3.framework 默认开启振动功能核心功能实现和分析

功能实现分析：  
Android 开启振动主要运用了 Vibrator（振动器），系统中有一个 Vibrator [抽象类](https://so.csdn.net/so/search?q=%E6%8A%BD%E8%B1%A1%E7%B1%BB&spm=1001.2101.3001.7020)，我们可以通过获取 Vibrator 实例调用里面的方法来完成振动功能。  
app 实现方法如下

```
Vibrator vibrator = (Vibrator) getSystemServic(Service.VIBRATOR_SERVICE);
vibrator.vibrate(1000);  //设置手机振动
vibrator.hasVibrator();  //判断手机硬件是否有振动器
vibrator.cancel();//关闭振动

```

通过上面的例子发现 设置震动功能 首选要判断硬件是否支持震动功能 就是 VibratorServices.java 中的 hasVibrator() 是否为 true  
接下来看下 VibratorServices.java 的源码  
路径为：framework/base/services/core/java/com/android/server/VibratorService.java

```
public class VibratorService extends IVibratorService.Stub
          implements InputManager.InputDeviceListener {
   VibratorService(Context context) {
          vibratorInit();
          // Reset the hardware to a default state, in case this is a runtime
          // restart instead of a fresh boot.
          vibratorOff();
  
          mSupportsAmplitudeControl = vibratorSupportsAmplitudeControl();
          mSupportsExternalControl = vibratorSupportsExternalControl();
          mSupportedEffects = asList(vibratorGetSupportedEffects());
          mCapabilities = vibratorGetCapabilities();
  
          mContext = context;
          PowerManager pm = context.getSystemService(PowerManager.class);
          mWakeLock = pm.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "*vibrator*");
          mWakeLock.setReferenceCounted(true);
  
          mAppOps = mContext.getSystemService(AppOpsManager.class);
          mBatteryStatsService = IBatteryStats.Stub.asInterface(ServiceManager.getService(
                  BatteryStats.SERVICE_NAME));
          mSystemUiPackage = LocalServices.getService(PackageManagerInternal.class)
                  .getSystemUiServiceComponent().getPackageName();
  
          mPreviousVibrationsLimit = mContext.getResources().getInteger(
                  com.android.internal.R.integer.config_previousVibrationsDumpLimit);
  
          mDefaultVibrationAmplitude = mContext.getResources().getInteger(
                  com.android.internal.R.integer.config_defaultVibrationAmplitude);
  
          mAllowPriorityVibrationsInLowPowerMode = mContext.getResources().getBoolean(
                  com.android.internal.R.bool.config_allowPriorityVibrationsInLowPowerMode);
  
          mPreviousRingVibrations = new LinkedList<>();
          mPreviousNotificationVibrations = new LinkedList<>();
          mPreviousAlarmVibrations = new LinkedList<>();
          mPreviousVibrations = new LinkedList<>();
          mPreviousExternalVibrations = new LinkedList<>();
  
          IntentFilter filter = new IntentFilter();
          filter.addAction(Intent.ACTION_SCREEN_OFF);
          context.registerReceiver(mIntentReceiver, filter);
  
          VibrationEffect clickEffect = createEffectFromResource(
                  com.android.internal.R.array.config_virtualKeyVibePattern);
          VibrationEffect doubleClickEffect = VibrationEffect.createWaveform(
                  DOUBLE_CLICK_EFFECT_FALLBACK_TIMINGS, -1 /*repeatIndex*/);
          VibrationEffect heavyClickEffect = createEffectFromResource(
                  com.android.internal.R.array.config_longPressVibePattern);
          VibrationEffect tickEffect = createEffectFromResource(
                  com.android.internal.R.array.config_clockTickVibePattern);
  
          mFallbackEffects = new SparseArray<>();
          mFallbackEffects.put(VibrationEffect.EFFECT_CLICK, clickEffect);
          mFallbackEffects.put(VibrationEffect.EFFECT_DOUBLE_CLICK, doubleClickEffect);
          mFallbackEffects.put(VibrationEffect.EFFECT_TICK, tickEffect);
          mFallbackEffects.put(VibrationEffect.EFFECT_HEAVY_CLICK, heavyClickEffect);
  
          mFallbackEffects.put(VibrationEffect.EFFECT_TEXTURE_TICK,
                  VibrationEffect.get(VibrationEffect.EFFECT_TICK, false));
  
          mScaleLevels = new SparseArray<>();
          mScaleLevels.put(SCALE_VERY_LOW,
                  new ScaleLevel(SCALE_VERY_LOW_GAMMA, SCALE_VERY_LOW_MAX_AMPLITUDE));
          mScaleLevels.put(SCALE_LOW, new ScaleLevel(SCALE_LOW_GAMMA, SCALE_LOW_MAX_AMPLITUDE));
          mScaleLevels.put(SCALE_NONE, new ScaleLevel(SCALE_NONE_GAMMA));
          mScaleLevels.put(SCALE_HIGH, new ScaleLevel(SCALE_HIGH_GAMMA));
          mScaleLevels.put(SCALE_VERY_HIGH, new ScaleLevel(SCALE_VERY_HIGH_GAMMA));
  
          ServiceManager.addService(EXTERNAL_VIBRATOR_SERVICE, new ExternalVibratorService());
      }
@Override // Binder call
public boolean hasVibrator() {
    return doVibratorExists();
}
private boolean doVibratorExists() {
   - //return vibratorExists();
   + return vibratorExists() && SystemProperties.get("persist.sys.support.vibration", "false").equals("true");
}

```

在 updateInputDeviceVibratorsLocked() 更新设备振动器的时候 会调用 vibrator.hasVibrator() 判断是不是  
有振动功能，如果有振动则开启振动功能 而在 doVibratorExists() 中判断是否开启  
在 system.prop 中增加设置属性 persist.sys.support.vibration 的值，根据他的值来判断是否开启振动模式  
设置属性如下:  
在 system.prop 中设置属性值

```
diff --git a/device/sprd/sharkl5/ums512_2h10/system.prop b/device/sprd/sharkl5/ums512_2h10/system.prop

old mode 100644 (file)

new mode 100755 (executable)

index aa6fba5..0d8c10f

--- a/device/sprd/sharkl5Pro/ums512_1h10/system.prop

+++ b/device/sprd/sharkl5Pro/ums512_1h10/system.prop

@@ -69,3 +69,4 @@ ro.audio.offload_wakelock=0
# Default density config
#ro.sf.lcd_density=320
#ro.sf.lcd_width=54
#ro.sf.lcd_height=96
ro.product.hardware=ums512_2h10
ro.vendor.audio.product.hardware=ums512_2h10
# Set Opengl ES Version
#ro.opengles.version=196609

# ro.fm.chip.port.UART.androidm=true

# FRP property for pst device
ro.frp.pst=/dev/block/platform/soc/soc:ap-apb/71400000.sdio/by-name/persist

#enable audio nr tuning
ro.vendor.audio_tunning.nr=1

#config dreamcamera begin
persist.sys.cam.battery.flash = 10
persist.sys.cam.3dnr=true
persist.sys.cam.normalhdr=false
persist.sys.cam.beauty.fullfuc=true
persist.sys.cam.sfv.alter=true
persist.sys.cam.filter.version=2
persist.sys.cam.manual.shutter=true
persist.sys.cam.manual.focus=true
persist.sys.cam.wide.8M=true
persist.sys.cam.wide.power=true
#config dreamcamera end

#Enable sdcardfs feature
ro.sys.sdcardfs=true
persist.sys.sdcardfs=force_on

persist.sys.cam.ba.blur.version=6
persist.sys.cam.api.version=0
persist.sys.blending.enable=true
persist.sys.sprd.refocus.bokeh=true

persist.sys.cam.fr.blur.version=1
#eis function
persist.sys.cam.eois.dc.back=false
persist.sys.cam.eois.dc.front=false
persist.sys.cam.eois.dv.back=true
persist.sys.cam.eois.dv.front=false

#enable 3d calibration
persist.sys.3d.calibraion=1
persist.sys.cam3.type=back_bokeh
persist.sys.cam3.multi.cam.id=2
 

 #add for lowpower vowifi

 persist.dbg.lowpower_vowifi=0

+persist.sys.support.vibration=true

```
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/128243991)

1. 概述
-----

  在 11.0 的系统产品开发中，对于设置默认屏幕亮度和屏幕亮度百分比的功能，在开发中也是常见的功能，在 [rk](https://so.csdn.net/so/search?q=rk&spm=1001.2101.3001.7020) mtk 展讯的平台设置可能有一些不同，一般的都是在 SettingProvider 中设置就可以了  
但 [mtk](https://so.csdn.net/so/search?q=mtk&spm=1001.2101.3001.7020) 的略有不同

2. 设置默认屏幕亮度百分比的核心类
------------------

```
frameworks/base/packages/apps/SettingsProvider/res/values/defaults.xml
vendor\mediatek\proprietary\packages\apps\MtkSettings\src\com\android\settings\display\BrightnessLevelPreferenceController.java
frameworks\base\services\core\java\com\android\server\power\PowerManagerService.java
```

3. 设置默认屏幕亮度百分比的核心功能分析和实现
------------------------

  3.1 mtk 以外的平台修改默认亮度的方法
------------------------

     在 rk 展讯等平台设置默认屏幕亮度，就是需要在 SettingsProvider 中的 defaults.xml 中的 def_screen_brightness 就是默认亮度，修改这个 SettingsProvider 中的 def_screen_brightness 的值即可 默认是 102 就是 40% 的亮度因为亮度最大 255 最小 0, 所以亮度默认 40% 就是 102

  3.2 mtk 修改默认亮度百分比的方法
----------------------

     系统 Settings 的显示菜单中，二级菜单中亮度下就显示屏幕亮度的默认百分比值，所以可以从这里看下屏幕亮度的默认百分比值该如何设置的，接下来看下 BrightnessLevelPreferenceController 中的相关源码看 BrightnessLevelPreferenceController 是如何设置屏幕亮度百分比的

```
   public class BrightnessLevelPreferenceController extends AbstractPreferenceController implements
        PreferenceControllerMixin, LifecycleObserver, OnStart, OnStop {
 
    private static final String TAG = "BrightnessPrefCtrl";
    private static final String KEY_BRIGHTNESS = "brightness";
    private static final Uri BRIGHTNESS_URI;
    private static final Uri BRIGHTNESS_FOR_VR_URI;
    private static final Uri BRIGHTNESS_ADJ_URI;
 
    private final float mMinBrightness;
    private final float mMaxBrightness;
    private final float mMinVrBrightness;
    private final float mMaxVrBrightness;
    private final float mDefaultBacklight;
    private final ContentResolver mContentResolver;
 
    private Preference mPreference;
 
    static {
        BRIGHTNESS_URI = System.getUriFor(System.SCREEN_BRIGHTNESS_FLOAT);
        BRIGHTNESS_FOR_VR_URI = System.getUriFor(System.SCREEN_BRIGHTNESS_FOR_VR);
        BRIGHTNESS_ADJ_URI = System.getUriFor(System.SCREEN_AUTO_BRIGHTNESS_ADJ);
    }
 
    private ContentObserver mBrightnessObserver =
            new ContentObserver(new Handler(Looper.getMainLooper())) {
                @Override
                public void onChange(boolean selfChange) {
                    updatedSummary(mPreference);
                }
            };
 
    public BrightnessLevelPreferenceController(Context context, Lifecycle lifecycle) {
        super(context);
        if (lifecycle != null) {
            lifecycle.addObserver(this);
        }
        PowerManager powerManager = (PowerManager) context.getSystemService(Context.POWER_SERVICE);
        mMinBrightness = powerManager.getBrightnessConstraint(
                PowerManager.BRIGHTNESS_CONSTRAINT_TYPE_MINIMUM);
        mMaxBrightness = powerManager.getBrightnessConstraint(
                PowerManager.BRIGHTNESS_CONSTRAINT_TYPE_MAXIMUM);
        mMinVrBrightness = powerManager.getBrightnessConstraint(
                PowerManager.BRIGHTNESS_CONSTRAINT_TYPE_MINIMUM_VR);
        mMaxVrBrightness = powerManager.getBrightnessConstraint(
                PowerManager.BRIGHTNESS_CONSTRAINT_TYPE_MAXIMUM_VR);
        mContentResolver = mContext.getContentResolver();
        mDefaultBacklight = powerManager.getBrightnessConstraint(
                PowerManager.BRIGHTNESS_CONSTRAINT_TYPE_DEFAULT);
    }
```

在 BrightnessLevelPreferenceController 的上述源码可以发现在 BrightnessLevelPreferenceController 的构造方法中  
可以看到 mDefaultBacklight 是默认屏幕亮度的值，而 mMinBrightness 为最小屏幕亮度值  
mMaxBrightness 就是最大默认屏幕亮度值，而他们的值都是通过 powerManager.getBrightnessConstraint  
来获取的，接下来看下 PowerManager 的相关方法来获取对应的值

```
private void updatedSummary(Preference preference) {
        if (preference != null) {
            preference.setSummary(NumberFormat.getPercentInstance().format(getCurrentBrightness()));
        }
    }
 
    private double getCurrentBrightness() {
        final int value;
        if (isInVrMode()) {
            value = convertLinearToGammaFloat(System.getFloat(mContentResolver,
                    System.SCREEN_BRIGHTNESS_FOR_VR_FLOAT, mMaxBrightness),
                    mMinVrBrightness, mMaxVrBrightness);
        } else {
            value = convertLinearToGammaFloat(Settings.System.getFloat(mContentResolver,
                    System.SCREEN_BRIGHTNESS_FLOAT, mDefaultBacklight),
                    mMinBrightness, mMaxBrightness);
        }
        return getPercentage(value, GAMMA_SPACE_MIN, GAMMA_SPACE_MAX);
    }
 
    private double getPercentage(double value, int min, int max) {
        if (value > max) {
            return 1.0;
        }
        if (value < min) {
            return 0.0;
        }
        return (value - min) / (max - min);
    }
```

而在 BrightnessLevelPreferenceController 中的 getCurrentBrightness() 是获取当前默认屏幕的百分比的值的，是根据 convertLinearToGammaFloat(方法来根据亮度的最大值 最小值 默认值来计算默认屏幕亮度值的，然后通过 updatedSummary(设置默认亮度值来显示

3.3 PowerManagerService.java 中相关源码分析
------------------------------------

```
    public final class PowerManagerService extends SystemService
        implements Watchdog.Monitor {
        // Screen brightness setting limits.
    private float mScreenBrightnessSettingMinimum;
    private float mScreenBrightnessSettingMaximum;
    private float mScreenBrightnessSettingDefault;
    public final float mScreenBrightnessMinimum;
    public final float mScreenBrightnessMaximum;
    public final float mScreenBrightnessDefault;
    public final float mScreenBrightnessDoze;
    public final float mScreenBrightnessDim;
    public final float mScreenBrightnessMinimumVr;
    public final float mScreenBrightnessMaximumVr;
    public final float mScreenBrightnessDefaultVr;
 
@VisibleForTesting
    PowerManagerService(Context context, Injector injector) {
        super(context);
 
        mContext = context;
......
        // Save brightness values:
        // Get float values from config.
        // Store float if valid
        // Otherwise, get int values and convert to float and then store.
        final float min = mContext.getResources().getFloat(com.android.internal.R.dimen
                .config_screenBrightnessSettingMinimumFloat);
        final float max = mContext.getResources().getFloat(com.android.internal.R.dimen
                .config_screenBrightnessSettingMaximumFloat);
        final float def = mContext.getResources().getFloat(com.android.internal.R.dimen
                .config_screenBrightnessSettingDefaultFloat);
        final float doze = mContext.getResources().getFloat(com.android.internal.R.dimen
                .config_screenBrightnessDozeFloat);
        final float dim = mContext.getResources().getFloat(com.android.internal.R.dimen
                .config_screenBrightnessDimFloat);
 
        if (min == INVALID_BRIGHTNESS_IN_CONFIG || max == INVALID_BRIGHTNESS_IN_CONFIG
                || def == INVALID_BRIGHTNESS_IN_CONFIG) {
            mScreenBrightnessMinimum = BrightnessSynchronizer.brightnessIntToFloat(
                    mContext.getResources().getInteger(com.android.internal.R.integer
                            .config_screenBrightnessSettingMinimum),
                    PowerManager.BRIGHTNESS_OFF + 1, PowerManager.BRIGHTNESS_ON,
                    PowerManager.BRIGHTNESS_MIN, PowerManager.BRIGHTNESS_MAX);
            mScreenBrightnessMaximum = BrightnessSynchronizer.brightnessIntToFloat(
                    mContext.getResources().getInteger(com.android.internal.R.integer
                            .config_screenBrightnessSettingMaximum),
                    PowerManager.BRIGHTNESS_OFF + 1, PowerManager.BRIGHTNESS_ON,
                    PowerManager.BRIGHTNESS_MIN, PowerManager.BRIGHTNESS_MAX);
            mScreenBrightnessDefault = BrightnessSynchronizer.brightnessIntToFloat(
                    mContext.getResources().getInteger(com.android.internal.R.integer
                            .config_screenBrightnessSettingDefault),
                    PowerManager.BRIGHTNESS_OFF + 1, PowerManager.BRIGHTNESS_ON,
                    PowerManager.BRIGHTNESS_MIN, PowerManager.BRIGHTNESS_MAX);
        } else {
            mScreenBrightnessMinimum = min;
            mScreenBrightnessMaximum = max;
            mScreenBrightnessDefault = def;
        }
..
}
```

在 PowerManagerService 的构造方法中，获取 mScreenBrightnessDefault 的值就是从 res 的 config 中读取的 config_screenBrightnessSettingDefaultFloat  
, 同样的 config_screenBrightnessSettingMaximumFloat 和 config_screenBrightnessSettingMinimumFloat 同样也是在 config 中定义的

```
       public float getBrightnessConstraint(int constraint) {
            switch (constraint) {
                case PowerManager.BRIGHTNESS_CONSTRAINT_TYPE_MINIMUM:
                    return mScreenBrightnessMinimum;
                case PowerManager.BRIGHTNESS_CONSTRAINT_TYPE_MAXIMUM:
                    return mScreenBrightnessMaximum;
                case PowerManager.BRIGHTNESS_CONSTRAINT_TYPE_DEFAULT:
                    return mScreenBrightnessDefault;
                case PowerManager.BRIGHTNESS_CONSTRAINT_TYPE_DIM:
                    return mScreenBrightnessDim;
                case PowerManager.BRIGHTNESS_CONSTRAINT_TYPE_DOZE:
                    return mScreenBrightnessDoze;
                case PowerManager.BRIGHTNESS_CONSTRAINT_TYPE_MINIMUM_VR:
                    return mScreenBrightnessMinimumVr;
                case PowerManager.BRIGHTNESS_CONSTRAINT_TYPE_MAXIMUM_VR:
                    return mScreenBrightnessMaximumVr;
                case PowerManager.BRIGHTNESS_CONSTRAINT_TYPE_DEFAULT_VR:
                    return mScreenBrightnessDefaultVr;
                default:
                    return PowerManager.BRIGHTNESS_INVALID_FLOAT;
            }
        }
```

在 PowerManagerService 中的 getBrightnessConstraint(int constraint) 中获取根据类型获取  
这些值

3.4 具体设置默认屏幕亮度值修改为：
-------------------

> ```
> --- a/vendor/mediatek/proprietary/packages/overlay/vendor/FrameworkResOverlay/res/values/config.xml
> +++ b/vendor/mediatek/proprietary/packages/overlay/vendor/FrameworkResOverlay/res/values/config.xml
> @@ -28,5 +28,5 @@
>      <item >1.0</item>
>      <!-- Default screen brightness setting set.
>      Set this to 0.4 for Default brightness Float.-->
> -    <item >0.4</item>
> +    <item >0.35</item>
>  </resources>
> ```
> 
> 根据适当的 config_screenBrightnessSettingDefaultFloat 值获取默认屏幕亮度值，每个平台每款产品亮度值都是有差异的  
> 但是最终都是根据 config_screenBrightnessSettingDefaultFloat 的值来设置的
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125091581)

### 1. 概述

在进行 11.0 定制化开发中，由于产品的分辨率大所以显得系统默认字体有点小，所以需要针对系统默认字体大小的进行修改，由于对于系统默认字体的更改. 所以设置默认系统字体大小

### 2. 系统默认字体修改的相关代码

```
frameworks/base/core/java/android/content/res/Configuration.java
frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/SettingsHelper.java

```

### 3. 系统默认字体修改的相关功能分析和实现

### 3.1 关于 Configuration.java 的相关系统默认字体的分析

这个类描述了设备的所有配置信息，这些配置信息会影响到应用程序检索的资源。包括了用户指定的选项（locale 和 scaling）也包括设备本身配置 (例如 input modes，screen size and screen orientation)  
系统字体 系统字体大小等  
先分析源码

```
public final class Configuration implements Parcelable, Comparable<Configuration> {
    /** @hide */
    public static final Configuration EMPTY = new Configuration();

    private static final String TAG = "Configuration";
     public Configuration() {
        unset();
    }

    /**
     * Makes a deep copy suitable for modification.
     */
    public Configuration(Configuration o) {
        setTo(o);
    }

    /* This brings mLocaleList in sync with locale in case a user of the older API who doesn't know
     * about setLocales() has changed locale directly. */
    private void fixUpLocaleList() {
        if ((locale == null && !mLocaleList.isEmpty()) ||
                (locale != null && !locale.equals(mLocaleList.get(0)))) {
            mLocaleList = locale == null ? LocaleList.getEmptyLocaleList() : new LocaleList(locale);
        }
    }

    /**
     * Sets the fields in this object to those in the given Configuration.
     *
     * @param o The Configuration object used to set the values of this Configuration's fields.
     */
    public void setTo(Configuration o) {
        fontScale = o.fontScale;
        mcc = o.mcc;
        mnc = o.mnc;
        if (o.locale == null) {
            locale = null;
        } else if (!o.locale.equals(locale)) {
            // Only clone a new Locale instance if we need to:  the clone() is
            // both CPU and GC intensive.
            locale = (Locale) o.locale.clone();
        }
        o.fixUpLocaleList();
        mLocaleList = o.mLocaleList;
        userSetLocale = o.userSetLocale;
        touchscreen = o.touchscreen;
        keyboard = o.keyboard;
        keyboardHidden = o.keyboardHidden;
        hardKeyboardHidden = o.hardKeyboardHidden;
        navigation = o.navigation;
        navigationHidden = o.navigationHidden;
        orientation = o.orientation;
        screenLayout = o.screenLayout;
        colorMode = o.colorMode;
        uiMode = o.uiMode;
        screenWidthDp = o.screenWidthDp;
        screenHeightDp = o.screenHeightDp;
        smallestScreenWidthDp = o.smallestScreenWidthDp;
        densityDpi = o.densityDpi;
        compatScreenWidthDp = o.compatScreenWidthDp;
        compatScreenHeightDp = o.compatScreenHeightDp;
        compatSmallestScreenWidthDp = o.compatSmallestScreenWidthDp;
        assetsSeq = o.assetsSeq;
        seq = o.seq;
        windowConfiguration.setTo(o.windowConfiguration);
    }
  /**
      * Set this object to the system defaults.
      */
     public void setToDefaults() {
         fontScale = 1;
         mcc = mnc = 0;
         mLocaleList = LocaleList.getEmptyLocaleList();
         locale = null;
         userSetLocale = false;
         touchscreen = TOUCHSCREEN_UNDEFINED;
         keyboard = KEYBOARD_UNDEFINED;
         keyboardHidden = KEYBOARDHIDDEN_UNDEFINED;
         hardKeyboardHidden = HARDKEYBOARDHIDDEN_UNDEFINED;
         navigation = NAVIGATION_UNDEFINED;
         navigationHidden = NAVIGATIONHIDDEN_UNDEFINED;
         orientation = ORIENTATION_UNDEFINED;
         screenLayout = SCREENLAYOUT_UNDEFINED;
         colorMode = COLOR_MODE_UNDEFINED;
         uiMode = UI_MODE_TYPE_UNDEFINED;
         screenWidthDp = compatScreenWidthDp = SCREEN_WIDTH_DP_UNDEFINED;
         screenHeightDp = compatScreenHeightDp = SCREEN_HEIGHT_DP_UNDEFINED;
         smallestScreenWidthDp = compatSmallestScreenWidthDp = SMALLEST_SCREEN_WIDTH_DP_UNDEFINED;
         densityDpi = DENSITY_DPI_UNDEFINED;
         assetsSeq = ASSETS_SEQ_UNDEFINED;
         seq = 0;
         windowConfiguration.setToDefaults();
     }
 

```

setTo（）负责拷贝 config 参数而 setToDefaults() 负责设置默认 config 的参数  
具体修改方法如下:

系统字体的大小修改主要是修改数据库的默认字体大小如下:

```
--- a/core/java/android/content/res/Configuration.java
+++ b/core/java/android/content/res/Configuration.java
@@ -1356,7 +1356,7 @@ public final class Configuration implements Parcelable, Comparable<Configuration
      * Set this object to the system defaults.
      */
     public void setToDefaults() {
-        fontScale = 1.0f; //默认为1.of
+        fontScale = 1.3f;
         mcc = mnc = 0;
         mLocaleList = LocaleList.getEmptyLocaleList();
         locale = null;
                userSetLocale = false;
         touchscreen = TOUCHSCREEN_UNDEFINED;
         keyboard = KEYBOARD_UNDEFINED;
         keyboardHidden = KEYBOARDHIDDEN_UNDEFINED;
         hardKeyboardHidden = HARDKEYBOARDHIDDEN_UNDEFINED;
         navigation = NAVIGATION_UNDEFINED;
         navigationHidden = NAVIGATIONHIDDEN_UNDEFINED;
         orientation = ORIENTATION_UNDEFINED;
         screenLayout = SCREENLAYOUT_UNDEFINED;
         colorMode = COLOR_MODE_UNDEFINED;
         uiMode = UI_MODE_TYPE_UNDEFINED;
         screenWidthDp = compatScreenWidthDp = SCREEN_WIDTH_DP_UNDEFINED;
         screenHeightDp = compatScreenHeightDp = SCREEN_HEIGHT_DP_UNDEFINED;
         smallestScreenWidthDp = compatSmallestScreenWidthDp = SMALLEST_SCREEN_WIDTH_DP_UNDEFINED;
         densityDpi = DENSITY_DPI_UNDEFINED;
         assetsSeq = ASSETS_SEQ_UNDEFINED;
         seq = 0;
         windowConfiguration.setToDefaults();
     }
 

```

通过修改 fontScale 比例来调整字体大小

### 3.2 DatabaseHelper.java 数据库关于系统默认大小修改

在数据库初始化的过程中，设置默认的字体大小，可以通过修改这个默认值来实现修改系统默认字体的修改

```
diff --git a/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java b/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
index 4743137..26ee0b8 100755
--- a/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
+++ b/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
@@ -2275,7 +2275,7 @@ class DatabaseHelper extends SQLiteOpenHelper {
private void loadSystemSettings(SQLiteDatabase db) {
          SQLiteStatement stmt = null;
          try {
              stmt = db.compileStatement("INSERT OR IGNORE INTO system(name,value)"
                      + " VALUES(?,?);");
  
              loadBooleanSetting(stmt, Settings.System.DIM_SCREEN,
                      R.bool.def_dim_screen);
              loadIntegerSetting(stmt, Settings.System.SCREEN_OFF_TIMEOUT,
                      R.integer.def_screen_off_timeout);
  
              // Set default cdma DTMF type
              loadSetting(stmt, Settings.System.DTMF_TONE_TYPE_WHEN_DIALING, 0);
  
              // Set default hearing aid
              loadSetting(stmt, Settings.System.HEARING_AID, 0);
      loadIntegerSetting(stmt, Settings.System.SCREEN_BRIGHTNESS_FOR_VR,
                      com.android.internal.R.integer.config_screenBrightnessForVrSettingDefault);
  
              loadBooleanSetting(stmt, Settings.System.SCREEN_BRIGHTNESS_MODE,
                      R.bool.def_screen_brightness_automatic_mode);
  
              loadBooleanSetting(stmt, Settings.System.ACCELEROMETER_ROTATION,
                      R.bool.def_accelerometer_rotation);
  
              loadDefaultHapticSettings(stmt);
  
              loadBooleanSetting(stmt, Settings.System.NOTIFICATION_LIGHT_PULSE,
                      R.bool.def_notification_pulse);
  
              loadUISoundEffectsSettings(stmt);
  
              loadIntegerSetting(stmt, Settings.System.POINTER_SPEED,
                      R.integer.def_pointer_speed);
-
+            loadSetting(stmt, Settings.System.FONT_SCALE, 1.3f);
             loadIntegerSetting(stmt, Settings.System.SCREEN_BRIGHTNESS,
                     R.integer.def_screen_brightness);

```

通过添加系统默认字体的大小比例 来放大系统默认字体大小

### 3.3 SettingsHelper.java 加载默认字体大小

在 SettingsHelper 中关于获取系统默认字体大小的修改

```
private boolean isAlreadyConfiguredCriticalAccessibilitySetting(String name) {
          // These are the critical accessibility settings that are required for users with
          // accessibility needs to be able to interact with the device. If these settings are
          // already configured, we will not overwrite them. If they are already set,
          // it means that the user has performed a global gesture to enable accessibility or set
          // these settings in the Accessibility portion of the Setup Wizard, and definitely needs
          // these features working after the restore.
          switch (name) {
              case Settings.Secure.ACCESSIBILITY_ENABLED:
              case Settings.Secure.TOUCH_EXPLORATION_ENABLED:
              case Settings.Secure.ACCESSIBILITY_DISPLAY_DALTONIZER_ENABLED:
              case Settings.Secure.ACCESSIBILITY_DISPLAY_MAGNIFICATION_ENABLED:
              case Settings.Secure.ACCESSIBILITY_DISPLAY_MAGNIFICATION_NAVBAR_ENABLED:
                  return Settings.Secure.getInt(mContext.getContentResolver(), name, 0) != 0;
              case Settings.Secure.TOUCH_EXPLORATION_GRANTED_ACCESSIBILITY_SERVICES:
              case Settings.Secure.ENABLED_ACCESSIBILITY_SERVICES:
              case Settings.Secure.ACCESSIBILITY_DISPLAY_DALTONIZER:
                  return !TextUtils.isEmpty(Settings.Secure.getString(
                          mContext.getContentResolver(), name));
              case Settings.Secure.ACCESSIBILITY_DISPLAY_MAGNIFICATION_SCALE:
                  float defaultScale = mContext.getResources().getFraction(
                          R.fraction.def_accessibility_display_magnification_scale, 1, 1);
                  float currentScale = Settings.Secure.getFloat(
                          mContext.getContentResolver(), name, defaultScale);
                  return Math.abs(currentScale - defaultScale) >= FLOAT_TOLERANCE;
              case Settings.System.FONT_SCALE:
                  return Settings.System.getFloat(mContext.getContentResolver(), name, 1.0f) != 1.0f;
              default:
                  return false;
          }
      }

```

isAlreadyConfiguredCriticalAccessibilitySetting 对是否是系统默认字体的判断 然后返回相应的字体  
case Settings.System.FONT_SCALE: 在关于系统默认字体大小时修改成 1.3 就可以了  
修改如下：

```
diff --git a/packages/SettingsProvider/src/com/android/providers/settings/SettingsHelper.java b/packages/SettingsProvider/src/com/android/providers/settings/SettingsHelper.java
old mode 100644
new mode 100755
index 36bb8ef..9f4d138
--- a/packages/SettingsProvider/src/com/android/providers/settings/SettingsHelper.java
+++ b/packages/SettingsProvider/src/com/android/providers/settings/SettingsHelper.java
@@ -274,7 +274,7 @@ public class SettingsHelper {
                         mContext.getContentResolver(), name, defaultScale);
                 return Math.abs(currentScale - defaultScale) >= FLOAT_TOLERANCE;
             case Settings.System.FONT_SCALE:
-                return Settings.System.getFloat(mContext.getContentResolver(), name, 1.0f) != 1.0f;
+                return Settings.System.getFloat(mContext.getContentResolver(), name, 1.3f) != 1.3f;
             default:
                 return false;
         }

```
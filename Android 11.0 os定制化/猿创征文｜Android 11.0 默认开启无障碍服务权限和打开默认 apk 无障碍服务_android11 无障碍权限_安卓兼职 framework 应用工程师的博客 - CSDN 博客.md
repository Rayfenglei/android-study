> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126691385)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 默认开启无障碍服务权限和打开默认 apk 无障碍服务核心代码](#t1)

[3. 默认开启无障碍服务权限和打开默认 apk 无障碍服务的功能分析和实现](#t2)

 [3.1 Settings.java 关于无障碍服务的定义](#t3)

[3.2 DatabaseHelper.java 的相关功能设置](#t4)

1. 概述
-----

在第三方 [app 开发](https://so.csdn.net/so/search?q=app%E5%BC%80%E5%8F%91&spm=1001.2101.3001.7020)中，需要开启无障碍服务功能，就不需要在代码中开启无障碍服务了，为了简便就需要在系统中开启无障碍服务

2. 默认开启无障碍服务权限和打开默认 apk 无障碍服务核心代码
---------------------------------

```
    frameworks/base/core/java/android/provider/Settings.java
    frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
```

3. 默认开启无障碍服务权限和打开默认 apk 无障碍服务的功能分析和实现
-------------------------------------

###    3.1 Settings.java 关于无障碍服务的定义

```
      public final class Settings {     
			 /**
           * Indicates which clock face to show on lock screen and AOD while docked.
           * @hide
           */
          public static final String DOCKED_CLOCK_FACE = "docked_clock_face";
  
          /**
           * Set by the system to track if the user needs to see the call to action for
           * the lockscreen notification policy.
           * @hide
           */
          public static final String SHOW_NOTE_ABOUT_NOTIFICATION_HIDING =
                  "show_note_about_notification_hiding";
  
          /**
           * Set to 1 by the system after trust agents have been initialized.
           * @hide
           */
          public static final String TRUST_AGENTS_INITIALIZED =
                  "trust_agents_initialized";
  
          /**
           * The Logging ID (a unique 64-bit value) as a hex string.
           * Used as a pseudonymous identifier for logging.
           * @deprecated This identifier is poorly initialized and has
           * many collisions.  It should not be used.
           */
          @Deprecated
          public static final String LOGGING_ID = "logging_id";
  
          /**
           * @deprecated Use {@link android.provider.Settings.Global#NETWORK_PREFERENCE} instead
           */
          @Deprecated
          public static final String NETWORK_PREFERENCE = Global.NETWORK_PREFERENCE;
  
          /**
           * No longer supported.
           */
          public static final String PARENTAL_CONTROL_ENABLED = "parental_control_enabled";
  
          /**
           * No longer supported.
           */
          public static final String PARENTAL_CONTROL_LAST_UPDATE = "parental_control_last_update";
  
          /**
           * No longer supported.
           */
          public static final String PARENTAL_CONTROL_REDIRECT_URL = "parental_control_redirect_url";
  
          /**
           * Settings classname to launch when Settings is clicked from All
           * Applications.  Needed because of user testing between the old
           * and new Settings apps.
           */
          // TODO: 881807
          public static final String SETTINGS_CLASSNAME = "settings_classname";
  
          /**
           * @deprecated Use {@link android.provider.Settings.Global#USB_MASS_STORAGE_ENABLED} instead
           */
          @Deprecated
          public static final String USB_MASS_STORAGE_ENABLED = Global.USB_MASS_STORAGE_ENABLED;
  
          /**
           * @deprecated Use {@link android.provider.Settings.Global#USE_GOOGLE_MAIL} instead
           */
          @Deprecated
          public static final String USE_GOOGLE_MAIL = Global.USE_GOOGLE_MAIL;
  
          /**
           * If accessibility is enabled.
           */
          public static final String ACCESSIBILITY_ENABLED = "accessibility_enabled";
  
          /**
           * Setting specifying if the accessibility shortcut is enabled.
           * @hide
           */
          public static final String ACCESSIBILITY_SHORTCUT_ON_LOCK_SCREEN =
                  "accessibility_shortcut_on_lock_screen";
  
          /**
           * Setting specifying if the accessibility shortcut dialog has been shown to this user.
           * @hide
           */
          public static final String ACCESSIBILITY_SHORTCUT_DIALOG_SHOWN =
                  "accessibility_shortcut_dialog_shown";
  
          /**
           * Setting specifying the accessibility services, accessibility shortcut targets,
           * or features to be toggled via the accessibility shortcut.
           *
           * <p> This is a colon-separated string list which contains the flattened
           * {@link ComponentName} and the class name of a system class implementing a supported
           * accessibility feature.
           * @hide
           */
          @UnsupportedAppUsage
          @TestApi
          public static final String ACCESSIBILITY_SHORTCUT_TARGET_SERVICE =
                  "accessibility_shortcut_target_service";
....
}
 
 
```

首先从 Settings.java 类中找到相关定义无障碍服务的常量
----------------------------------

ACCESSIBILITY_ENABLED 就是是否开启无障碍服务  
ENABLED_ACCESSIBILITY_SERVICES 就是需要开启无障碍服务，需要包名和类名  
接下来就是在 DatabaseHelper.java 中设置相关开启无障碍服务，然后在系统数据库保存

无障碍服务的默认值开启无障碍服务

3.2 DatabaseHelper.java 的相关功能设置
-------------------------------

```
class DatabaseHelper extends SQLiteOpenHelper {   
   private void loadSettings(SQLiteDatabase db) {
          loadSystemSettings(db);
          loadSecureSettings(db);
          // The global table only exists for the 'owner/system' user
          if (mUserHandle == UserHandle.USER_SYSTEM) {
              loadGlobalSettings(db);
          }
      }
  
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
  
              // Set default tty mode
              loadSetting(stmt, Settings.System.TTY_MODE, 0);
  
              loadIntegerSetting(stmt, Settings.System.SCREEN_BRIGHTNESS,
                      R.integer.def_screen_brightness);
  
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
  
              /*
               * IMPORTANT: Do not add any more upgrade steps here as the global,
               * secure, and system settings are no longer stored in a database
               * but are kept in memory and persisted to XML.
               *
               * See: SettingsProvider.UpgradeController#onUpgradeLocked
               */
          } finally {
              if (stmt != null) stmt.close();
          }
      }
 private void loadSecureSettings(SQLiteDatabase db) {
          SQLiteStatement stmt = null;
          try {
              stmt = db.compileStatement("INSERT OR IGNORE INTO secure(name,value)"
                      + " VALUES(?,?);");
  
              // Don't do this.  The SystemServer will initialize ADB_ENABLED from a
              // persistent system property instead.
              //loadSetting(stmt, Settings.Secure.ADB_ENABLED, 0);
  
              // Allow mock locations default, based on build
              loadSetting(stmt, Settings.Secure.ALLOW_MOCK_LOCATION,
                      "1".equals(SystemProperties.get("ro.allow.mock.location")) ? 1 : 0);
  
              loadSecure35Settings(stmt);
  
              loadBooleanSetting(stmt, Settings.Secure.MOUNT_PLAY_NOTIFICATION_SND,
                      R.bool.def_mount_play_notification_snd);
  
              loadBooleanSetting(stmt, Settings.Secure.MOUNT_UMS_AUTOSTART,
                      R.bool.def_mount_ums_autostart);
  
              loadBooleanSetting(stmt, Settings.Secure.MOUNT_UMS_PROMPT,
                      R.bool.def_mount_ums_prompt);
  
              loadBooleanSetting(stmt, Settings.Secure.MOUNT_UMS_NOTIFY_ENABLED,
                      R.bool.def_mount_ums_notify_enabled);
  
              loadIntegerSetting(stmt, Settings.Secure.LONG_PRESS_TIMEOUT,
                      R.integer.def_long_press_timeout_millis);
  
              loadBooleanSetting(stmt, Settings.Secure.TOUCH_EXPLORATION_ENABLED,
                      R.bool.def_touch_exploration_enabled);
  
              loadBooleanSetting(stmt, Settings.Secure.ACCESSIBILITY_SPEAK_PASSWORD,
                      R.bool.def_accessibility_speak_password);
  
              if (SystemProperties.getBoolean("ro.lockscreen.disable.default", false) == true) {
                  loadSetting(stmt, Settings.System.LOCKSCREEN_DISABLED, "1");
              } else {
                  loadBooleanSetting(stmt, Settings.System.LOCKSCREEN_DISABLED,
                          R.bool.def_lockscreen_disabled);
              }
  
              loadBooleanSetting(stmt, Settings.Secure.SCREENSAVER_ENABLED,
                      com.android.internal.R.bool.config_dreamsEnabledByDefault);
              loadBooleanSetting(stmt, Settings.Secure.SCREENSAVER_ACTIVATE_ON_DOCK,
                      com.android.internal.R.bool.config_dreamsActivatedOnDockByDefault);
              loadBooleanSetting(stmt, Settings.Secure.SCREENSAVER_ACTIVATE_ON_SLEEP,
                      com.android.internal.R.bool.config_dreamsActivatedOnSleepByDefault);
              loadStringSetting(stmt, Settings.Secure.SCREENSAVER_COMPONENTS,
                      com.android.internal.R.string.config_dreamsDefaultComponent);
              loadStringSetting(stmt, Settings.Secure.SCREENSAVER_DEFAULT_COMPONENT,
                      com.android.internal.R.string.config_dreamsDefaultComponent);
  
              loadBooleanSetting(stmt, Settings.Secure.ACCESSIBILITY_DISPLAY_MAGNIFICATION_ENABLED,
                      R.bool.def_accessibility_display_magnification_enabled);
  
              loadFractionSetting(stmt, Settings.Secure.ACCESSIBILITY_DISPLAY_MAGNIFICATION_SCALE,
                      R.fraction.def_accessibility_display_magnification_scale, 1);
  
              loadBooleanSetting(stmt, Settings.Secure.USER_SETUP_COMPLETE,
                      R.bool.def_user_setup_complete);
  
              loadStringSetting(stmt, Settings.Secure.IMMERSIVE_MODE_CONFIRMATIONS,
                          R.string.def_immersive_mode_confirmations);
  
              loadBooleanSetting(stmt, Settings.Secure.INSTALL_NON_MARKET_APPS,
                      R.bool.def_install_non_market_apps);
  
              loadBooleanSetting(stmt, Settings.Secure.WAKE_GESTURE_ENABLED,
                      R.bool.def_wake_gesture_enabled);
  
              loadIntegerSetting(stmt, Secure.LOCK_SCREEN_SHOW_NOTIFICATIONS,
                      R.integer.def_lock_screen_show_notifications);
  
              loadBooleanSetting(stmt, Secure.LOCK_SCREEN_ALLOW_PRIVATE_NOTIFICATIONS,
                      R.bool.def_lock_screen_allow_private_notifications);
  
              loadIntegerSetting(stmt, Settings.Secure.SLEEP_TIMEOUT,
                      R.integer.def_sleep_timeout);
  
              /*
               * IMPORTANT: Do not add any more upgrade steps here as the global,
               * secure, and system settings are no longer stored in a database
               * but are kept in memory and persisted to XML.
               *
               * See: SettingsProvider.UpgradeController#onUpgradeLocked
               */
          } finally {
              if (stmt != null) stmt.close();
          }
      }
  
....
}
 
```

loadsettings() 来负责设置系统数据库相关系统数据的默认值，          loadSystemSettings(db);  
          loadSecureSettings(db);  
           loadGlobalSettings(db);

根据不同类型保存到不同的数据库表中，而无障碍服务是在 loadSystemSettings(db); 中添加的相关服务默认值，所以具体开启无障碍服务修改如下:

```
在loadSystemSettings(SQLiteDatabase db)中开启相应的无障碍服务功能
修改如下:
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
              
// add core start
  loadSetting(stmt,Settings.Secure.ACCESSIBILITY_ENABLED,1);
  loadSetting(stmt,Settings.Secure.ENABLED_ACCESSIBILITY_SERVICES,"com.spr.service/com.spr.service.MyAccessibilityService"); //指定apk的服务名
 //add core end  
    
              // Set default tty mode
              loadSetting(stmt, Settings.System.TTY_MODE, 0);
  
              loadIntegerSetting(stmt, Settings.System.SCREEN_BRIGHTNESS,
                      R.integer.def_screen_brightness);
  
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
  
              /*
               * IMPORTANT: Do not add any more upgrade steps here as the global,
               * secure, and system settings are no longer stored in a database
               * but are kept in memory and persisted to XML.
               *
               * See: SettingsProvider.UpgradeController#onUpgradeLocked
               */
          } finally {
              if (stmt != null) stmt.close();
          }
      }
```
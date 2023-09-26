> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124828598)

### 1. 概述

在 11.0 产品定制化开发中，有产品需求要求系统去掉锁屏功能，默认永不锁屏，需要对去掉系统默认的锁屏功能和 息屏功能 让屏幕永远不要熄灭，

### 2. 去掉锁屏功能和息屏功能 (永不息屏) 的核心代码

```
frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
frameworks/base/packages/SettingsProvider/res/values/defaults.xml
frameworks/base/packages/SystemUI/res/values/config.xml

```

### 3. 去掉锁屏功能和息屏功能 (永不息屏) 的核心功能分析

### 3.1 关于 SettingsProvider 中息屏时间的分析

而系统默认的息屏时间的配置 是由 config.[xml](https://so.csdn.net/so/search?q=xml&spm=1001.2101.3001.7020) 设置数据由 DatabaseHelper.java 在系统开启时默认设置系统息屏时间和锁屏功能是否开启  
接下来先看 config.xml 的相关配置

```
 <resources>
     <bool >true</bool>
     <integer >60000</integer>
     <integer >-1</integer>
     <bool >false</bool>
     <bool >false</bool>
     <!-- Comma-separated list of bluetooth, wifi, and cell. -->
     <string >cell,bluetooth,wifi,nfc,wimax</string>
     <string >bluetooth,wifi,nfc</string>
     <string >0</string>
     <integer >1000</integer>
     <integer >15000</integer>
 

    <bool >true</bool>
     <bool >false</bool>
     <integer >1</integer>

```

在 config.xml 中默认的息屏时间就是 def_screen_off_timeout 为 60 秒即为 1 分钟，而 def_lockscreen_disabled 即为是否默认开启锁屏默认为 true 所以修改改成 false 默认不开启锁屏

在 DatabaseHelper.java 中设置默认息屏时间的相关代码为

```
      private void upgradeScreenTimeout(SQLiteDatabase db) {
          // Change screen timeout to current default
          db.beginTransaction();
          SQLiteStatement stmt = null;
          try {
              stmt = db.compileStatement("INSERT OR REPLACE INTO system(name,value)"
                      + " VALUES(?,?);");
              loadIntegerSetting(stmt, Settings.System.SCREEN_OFF_TIMEOUT,
                      R.integer.def_screen_off_timeout);
              db.setTransactionSuccessful();
          } finally {
              db.endTransaction();
              if (stmt != null)
                  stmt.close();
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

```

会默认设置最长息屏时间为 30 分钟  
所以修改息屏时间就要从这两个地方入手  
修改如下:

```
diff --git a/packages/SettingsProvider/res/values/defaults.xml b/packages/SettingsProvider/res/values/defaults.xml
old mode 100644
new mode 100755
index 7b81d9a..1b735f6
--- a/packages/SettingsProvider/res/values/defaults.xml
+++ b/packages/SettingsProvider/res/values/defaults.xml
@@ -18,7 +18,7 @@
-->
<resources>
<bool >true</bool>
-<integer >60000</integer>
+<integer >-1</integer>
<bool >false</bool>
<bool >false</bool>
-    <bool >false</bool>
+    <bool >true</bool>

```

### 3.2 DatabaseHelper.java 的修改

upgradeScreenTimeout(SQLiteDatabase db) 的设置息屏时间的代码

```
diff --git a/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java b/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
old mode 100644
new mode 100755
index b3ff9d0..4743137
--- a/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
+++ b/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
@@ -2020,7 +2020,7 @@ class DatabaseHelper extends SQLiteOpenHelper {
             // Set the timeout to 30 minutes in milliseconds
             loadSetting(stmt, Settings.System.SCREEN_OFF_TIMEOUT,

                   Integer.toString(30 * 60 * 1000));

                   Integer.toString(2147483647));

       } finally {
           if (stmt != null) stmt.close();
       }

```

### 3.3 SystemUI 的 config 中不启用锁屏功能

通过查看源码可知 config_enableKeyguardService 即为默认的锁屏功能是否开启默认为 true 修改为 false  
true 为默认开启锁屏功能  
修改如下：

```
diff --git a/packages/SystemUI/res/values/config.xml b/packages/SystemUI/res/values/config.xml
old mode 100644
new mode 100755
index be1e1ca..3ca3255
--- a/packages/SystemUI/res/values/config.xml
+++ b/packages/SystemUI/res/values/config.xml
@@ -165,7 +165,7 @@
<integer >250</integer>
 <!-- Whether to enable KeyguardService or not -->

- <bool >true</bool>
+ <bool >false</bool>
<integer >3</integer>
 
     <!-- Defines the implementation of the velocity tracker to be used for the panel expansion. Can
          be 'platform' or 'noisy' (i.e. for noisy touch screens). -->
     <string >platform</string>
 
     <!-- Doze: does this device support STATE_DOZE?  -->
     <bool >false</bool>
 
     <!-- Doze: does this device support STATE_DOZE_SUSPEND?  -->
     <bool >false</bool>
 
     <!-- Doze: should the significant motion sensor be used as a pulse signal? -->
     <bool >false</bool>
 
     <!-- Doze: check proximity sensor before pulsing? -->
     <bool >true</bool>
 
     <!-- Doze: should notifications be used as a pulse signal? -->
     <bool >true</bool>

```
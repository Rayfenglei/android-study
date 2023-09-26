> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124893973)

### 1. 概述

在 11.0 的产品开发中，有功能需要要求设置默认[输入法](https://so.csdn.net/so/search?q=%E8%BE%93%E5%85%A5%E6%B3%95&spm=1001.2101.3001.7020)，替换掉系统的输入法，所以这就需要了解设置  
输入法的相关功能需求，然后根据输入法包名来设置默认输入法

### 2. 第三方输入法 app 设置系统默认输入法的核心代码

```
frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java 
frameworks/base/packages/SettingsProvider/res/values/defaults.xml

```

3. 第三方输入法 app 设置系统默认输入法的核心功能分析
------------------------------

在系统输入法中 每个系统输入法的 id 不同 根据 id 设置输入法  
#Android 键盘 ([AOSP](https://so.csdn.net/so/search?q=AOSP&spm=1001.2101.3001.7020)) ~ 系统默认  
com.android.inputmethod.latin/.LatinIME

#谷歌拼音输入法  
com.google.android.inputmethod.pinyin/.PinyinIME

#谷歌 Gboard 输入法  
com.google.android.inputmethod.latin/com.android.inputmethod.latin.LatinIME

#触宝输入法国际版  
com.cootek.smartinputv5/com.cootek.smartinput5.TouchPalIME

#Go 输入法  
com.jb.emoji.gokeyboard/com.jb.gokeyboard.GoKeyboard

#SwiftKey [Keyboard](https://so.csdn.net/so/search?q=Keyboard&spm=1001.2101.3001.7020) 输入法  
com.touchtype.swiftkey/com.touchtype.KeyboardService

#搜狗输入法：  
com.sohu.inputmethod.sogou/.SogouIME

#微软必应输入法  
com.bingime.ime/.BingIme

### 3.1 SettingsProvider 默认输入法添加到系统数据库

系统属性中 Settings.Secure.ENABLED_INPUT_METHODS 即为系统默认输入法的属性  
设置这个属性即可

```
diff --git a/frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java b/frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
index 62996cbdcc..60eacba782 100755
--- a/frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
+++ b/frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
@@ -2398,7 +2398,8 @@ class DatabaseHelper extends SQLiteOpenHelper {
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
                     com.android.internal.R.string.config_dreamsDefaultComponent);
             loadStringSetting(stmt, Settings.Secure.SCREENSAVER_DEFAULT_COMPONENT,
                     com.android.internal.R.string.config_dreamsDefaultComponent);
-
+            loadStringSetting(stmt, Settings.Secure.ENABLED_INPUT_METHODS,
+                   R.string.def_enabled_input_methods);
             loadBooleanSetting(stmt, Settings.Secure.ACCESSIBILITY_DISPLAY_MAGNIFICATION_ENABLED,
                     R.bool.def_accessibility_display_magnification_enabled);
                     }

```

```
 在loadSystemSettings(SQLiteDatabase db)中添加默认系统输入法的id到数据库即可
​

```

### 3. 2 添加默认输入法 id

需要在 defaults.xml 中修改默认输入法 id 的值，修改为默认输入法

```
diff --git a/frameworks/base/packages/SettingsProvider/res/values/defaults.xml b/frameworks/base/packages/SettingsProvider/res/values/defaults.xml
index 98c2e66159..4d7f9bde83 100755
--- a/frameworks/base/packages/SettingsProvider/res/values/defaults.xml
+++ b/frameworks/base/packages/SettingsProvider/res/values/defaults.xml
@@ -79,7 +79,7 @@
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
     <bool >true</bool>
     <bool >true</bool>
     <bool >false</bool>
     <!-- Default screen brightness, from 0 to 255.  102 is 40%. -->
     <integer >102</integer>
     <bool >false</bool>
     <fraction >100%</fraction>
     <fraction >100%</fraction>
     <bool >true</bool>
 
     <bool >true</bool>
     <bool >false</bool>
     <bool >false</bool>
     <!-- 0 == off, 3 == on -->
     <integer >3</integer>
     <bool >true</bool>
     <bool >true</bool>
     <bool >true</bool>
     <bool >false</bool>
     <!-- 0 == never, 1 == only when plugged in, 2 == always -->
     <integer >2</integer>
     <bool >true</bool>
     <bool >true</bool>
 
     <bool >false</bool>
     <string >com.android.localtransport/.LocalTransport</string>
 
     <!-- Default value for whether or not to pulse the notification LED when there is a
          pending notification -->
     <bool >true</bool>
 
     <bool >true</bool>
     <bool >false</bool>
     <bool >true</bool>
     <bool >true</bool>
     <string >/product/media/audio/ui/Trusted.ogg</string>
     <string >/product/media/audio/ui/WirelessChargingStarted.ogg</string>
     <string >/product/media/audio/ui/ChargingStarted.ogg</string>
-
+    <string >com.xinshuru.inputmethod/.FTInputService</string>
     <!-- sound trigger detection service default values -->
     <integer >1000</integer>

```

在 def_enabled_input_methods 修改输入法 id 为需要修改输入法的 id，然后编译验证即可实现功能
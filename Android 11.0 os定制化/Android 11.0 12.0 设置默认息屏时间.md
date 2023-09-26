> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124755013)

### 1. 概述

在 11.0 12.0 定制化开发中，在系统设置中，息屏时间默认为 1 分钟，对于这个息屏时间感觉太短了，所以系统默认息屏时间修改也是常见的修改功能，在系统 Settings 中屏幕超时会根据默认息屏时间来显示屏幕超时的选项，来设置系统息屏时间

### 2. 设置默认息屏时间的核心类

```
/packages/apps/Settings/res/xml/display_settings.xml
framework/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java

```

### 3. 设置默认息屏时间的核心功能分析和实现

由于系统设置息屏时间是在显示菜单下的所以先看  
在系统设置中的 display_settings.[xml](https://so.csdn.net/so/search?q=xml&spm=1001.2101.3001.7020) 中的相关源码分析

### 3.1display_settings.xml 关于息屏时间分析

```
<?xml version="1.0" encoding="utf-8"?>

<PreferenceScreen
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:settings="http://schemas.android.com/apk/res-auto"
    android:key="display_settings_screen"
    android:title="@string/display_settings"
    settings:keywords="@string/keywords_display"
    settings:initialExpandedChildrenCount="5">

    <com.android.settingslib.RestrictedPreference
        android:key="brightness"
        android:title="@string/brightness"
        settings:keywords="@string/keywords_display_brightness_level"
        settings:useAdminDisabledSummary="true"
        settings:userRestriction="no_config_brightness">
        <intent android:action="com.android.intent.action.SHOW_BRIGHTNESS_DIALOG" />
    </com.android.settingslib.RestrictedPreference>

    <com.android.settings.display.NightDisplayPreference
        android:key="night_display"
        android:title="@string/night_display_title"
        android:fragment="com.android.settings.display.NightDisplaySettings"
        android:widgetLayout="@null"
        settings:widgetLayout="@null"
        settings:searchable="false" />

    <!-- UNISOC: Add for Color Temperature Adjusting -->
    <Preference
        android:key="colors_contrast"
        android:title="@string/colors_contrast_title">
    </Preference>

    <Preference
        android:key="auto_brightness_entry"
        android:title="@string/auto_brightness_title"
        android:summary="@string/summary_placeholder"
        android:fragment="com.android.settings.display.AutoBrightnessSettings"
        settings:controller="com.android.settings.display.AutoBrightnessPreferenceController"/>

    <com.android.settingslib.RestrictedPreference
        android:key="wallpaper"
        android:title="@string/wallpaper_settings_title"
        settings:keywords="@string/keywords_display_wallpaper"
        settings:useAdminDisabledSummary="true"
        settings:controller="com.android.settings.display.WallpaperPreferenceController">
    </com.android.settingslib.RestrictedPreference>


    <SwitchPreference
        android:key="dark_ui_mode"
        android:title="@string/dark_ui_mode"
        settings:keywords="@string/keywords_dark_ui_mode"
        settings:controller="com.android.settings.display.DarkUIPreferenceController"/>

    <!-- Cross-listed item, if you change this, also change it in power_usage_summary.xml -->
    <com.android.settings.display.TimeoutListPreference
        android:key="screen_timeout"
        android:title="@string/screen_timeout"
        android:summary="@string/summary_placeholder"
        android:entries="@array/screen_timeout_entries"
        android:entryValues="@array/screen_timeout_values"
        settings:keywords="@string/keywords_screen_timeout" />

    <!-- UNISOC:1185786 Support "One-handed mode" start -->
    <Preference
        android:key="one_handed_mode"
        android:title="@string/one_handed_mode"
        android:summary="@string/summary_placeholder"
        android:fragment="com.sprd.settings.onehandedmode.OneHandedModeSettings"
        settings:controller="com.sprd.settings.onehandedmode.OneHandedModeSettingsController" />
    <!-- UNISOC:1185786 Support "One-handed mode" end -->

    <Preference
        android:key="adaptive_sleep_entry"
        android:title="@string/adaptive_sleep_title"
        android:summary="@string/summary_placeholder"
        android:fragment="com.android.settings.display.AdaptiveSleepSettings"
        settings:controller="com.android.settings.display.AdaptiveSleepPreferenceController" />

    <SwitchPreference
        android:key="auto_rotate"
        android:title="@string/accelerometer_title"
        settings:keywords="@string/keywords_auto_rotate"
        settings:controller="com.android.settings.display.AutoRotatePreferenceController" />

    <Preference
        android:key="color_mode"
        android:title="@string/color_mode_title"
        android:fragment="com.android.settings.display.ColorModePreferenceFragment"
        settings:controller="com.android.settings.display.ColorModePreferenceController"
        settings:keywords="@string/keywords_color_mode" />

    <SwitchPreference
        android:key="display_white_balance"
        android:title="@string/display_white_balance_title"
        settings:controller="com.android.settings.display.DisplayWhiteBalancePreferenceController" />

    <Preference
        android:key="font_size"
        android:title="@string/title_font_size"
        android:fragment="com.android.settings.display.ToggleFontSizePreferenceFragment"
        settings:controller="com.android.settings.display.FontSizePreferenceController" />

    <com.android.settings.display.ScreenZoomPreference
        android:key="display_settings_screen_zoom"
        android:title="@string/screen_zoom_title"
        android:fragment="com.android.settings.display.ScreenZoomSettings"
        settings:searchable="false"/>

    <SwitchPreference
        android:key="show_operator_name"
        android:title="@string/show_operator_name_title"
        android:summary="@string/show_operator_name_summary" />

    <Preference
        android:key="screensaver"
        android:title="@string/screensaver_settings_title"
        android:fragment="com.android.settings.dream.DreamSettings"
        settings:searchable="false" />

    <com.android.settingslib.RestrictedPreference
        android:key="lockscreen_from_display_settings"
        android:title="@string/lockscreen_settings_title"
        android:fragment="com.android.settings.security.LockscreenDashboardFragment"
        settings:controller="com.android.settings.security.screenlock.LockScreenPreferenceController"
        settings:userRestriction="no_ambient_display" />

    <SwitchPreference
        android:key="camera_gesture"
        android:title="@string/camera_gesture_title"
        android:summary="@string/camera_gesture_desc" />

</PreferenceScreen>

```

而 从 display_settings.xml 的相关源码发现

```
<com.android.settings.display.TimeoutListPreference
    android:key="screen_timeout"
    android:title="@string/screen_timeout"
    android:summary="@string/summary_placeholder"
    android:entries="@array/screen_timeout_entries"
    android:entryValues="@array/screen_timeout_values"
    settings:keywords="@string/keywords_screen_timeout" />

```

就是屏幕超时项 ，就是通过这里设置息屏时间的，可以根据息屏时间项选择息屏时间  
而修改完息屏时间的处理是在

```
TimeoutPreferenceController.java 中处理的
    @Override
    public boolean onPreferenceChange(Preference preference, Object newValue) {
        try {
            //int value = Integer.parseInt((String) newValue);
            //Settings.System.putInt(mContext.getContentResolver(), SCREEN_OFF_TIMEOUT, value);
            long value = Integer.parseInt((String) newValue);
            Settings.System.putLong(mContext.getContentResolver(), SCREEN_OFF_TIMEOUT, value);
            updateTimeoutPreferenceDescription((TimeoutListPreference) preference, value);
        } catch (NumberFormatException e) {
            Log.e(TAG, "could not persist screen timeout setting", e);
        }
        return true;
    }

```

在 onPreferenceChange 代码中可以看出修改息屏时间会把值保存在 Settings.System.putLong(mContext.getContentResolver(), SCREEN_OFF_TIMEOUT, value); 就是说在  
数据库的值就是 screen_off_timeout 就是用来保存息屏时间的

所以只要在 framework/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java  
中修改 SCREEN_OFF_TIMEOUT 这个值就可以了

### 3.2DatabaseHelper.java 关于息屏时间的设置

```
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

```

在 loadSystemSettings 方法中数据库加载的时候 从 def_screen_off_timeout 来获取默认值

所以修改如下 在

```
--- a/frameworks/base/packages/SettingsProvider/res/values/defaults.xml
+++ b/frameworks/base/packages/SettingsProvider/res/values/defaults.xml
@@ -19,7 +19,7 @@
 <resources>
     <string >24</string>
     <bool >true</bool>
-    <integer >60000</integer>
+    <integer >1800000</integer>
     <integer >-1</integer>
     <bool >false</bool>
     <bool >false</bool>

```
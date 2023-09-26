> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124754793)

### 1. 概述

在 11.0 12.0 的系统中产品开发中，系统的 wifi 功能是默认关闭的，由于项目需要要求默认打开 wifi, 开机后直接连 wifi 就可以了  
所以需要找到系统默认的关闭 wifi 的地方 打开 wifi 就可以了

### 2. 系统默认开启 wifi 核心类分析

```
frameworks\base\packages\SettingsProvider\src\com\android\providers\settings\DatebaseHelper.java
frameworks/base/packages/SettingsProvider/res/values/defaults.xml

```

### 3. 系统默认开启 wifi 的核心功能分析和实现

在 11.0 的系统中 wifi 的默认开关 会保持在系统数据库中，通常是在 SettingProvider 中开系统首次开机的时候设置相关  
属性，于是就需要来找相应的是否保存 wifi 开启状态的相关源码

首选查看 DatebaseHelper.java 看有没相关的数据  
路径:  
frameworks\base\packages\SettingsProvider\src\com\android\providers\settings\DatebaseHelper.java  
代码如下:

```
private void loadGlobalSettings(SQLiteDatabase db) {
        SQLiteStatement stmt = null;
        final Resources res = mContext.getResources();
        try {
            stmt = db.compileStatement("INSERT OR IGNORE INTO global(name,value)"
                    + " VALUES(?,?);");

            // --- Previously in 'system'
            ....

            loadIntegerSetting(stmt, Settings.Global.WIFI_SLEEP_POLICY,
                    R.integer.def_wifi_sleep_policy);

            loadSetting(stmt, Settings.Global.MODE_RINGER,
                    AudioManager.RINGER_MODE_NORMAL);

            loadDefaultAnimationSettings(stmt);

            // --- Previously in 'secure'
            loadBooleanSetting(stmt, Settings.Global.PACKAGE_VERIFIER_ENABLE,
                    R.bool.def_package_verifier_enable);

            loadBooleanSetting(stmt, Settings.Global.WIFI_ON,
                    R.bool.def_wifi_on);

            loadBooleanSetting(stmt, Settings.Global.WIFI_NETWORKS_AVAILABLE_NOTIFICATION_ON,
                    R.bool.def_networks_available_notification_on);

            loadBooleanSetting(stmt, Settings.Global.BLUETOOTH_ON,
                    R.bool.def_bluetooth_on);

            // Enable or disable Cell Broadcast SMS
            loadSetting(stmt, Settings.Global.CDMA_CELL_BROADCAST_SMS,
                    RILConstants.CDMA_CELL_BROADCAST_SMS_DISABLED);

            // Data roaming default, based on build
            loadSetting(stmt, Settings.Global.DATA_ROAMING,
                    "true".equalsIgnoreCase(
                            SystemProperties.get("ro.com.android.dataroaming",
                                    "false")) ? 1 : 0);

            loadBooleanSetting(stmt, Settings.Global.DEVICE_PROVISIONED,
                    R.bool.def_device_provisioned);

            final int maxBytes = res.getInteger(
                    R.integer.def_download_manager_max_bytes_over_mobile);
            if (maxBytes > 0) {
                loadSetting(stmt, Settings.Global.DOWNLOAD_MAX_BYTES_OVER_MOBILE,
                        Integer.toString(maxBytes));
            }

            final int recommendedMaxBytes = res.getInteger(
                    R.integer.def_download_manager_recommended_max_bytes_over_mobile);
            if (recommendedMaxBytes > 0) {
                loadSetting(stmt, Settings.Global.DOWNLOAD_RECOMMENDED_MAX_BYTES_OVER_MOBILE,
                        Integer.toString(recommendedMaxBytes));
            }

            // Mobile Data default, based on build
            loadSetting(stmt, Settings.Global.MOBILE_DATA,
                    "true".equalsIgnoreCase(
                            SystemProperties.get("ro.com.android.mobiledata",
                                    "true")) ? 1 : 0);

            loadBooleanSetting(stmt, Settings.Global.NETSTATS_ENABLED,
                    R.bool.def_netstats_enabled);

            loadBooleanSetting(stmt, Settings.Global.USB_MASS_STORAGE_ENABLED,
                    R.bool.def_usb_mass_storage_enabled);

            loadIntegerSetting(stmt, Settings.Global.WIFI_MAX_DHCP_RETRY_COUNT,
                    R.integer.def_max_dhcp_retries);

            loadBooleanSetting(stmt, Settings.Global.WIFI_DISPLAY_ON,
                    R.bool.def_wifi_display_on);

            loadStringSetting(stmt, Settings.Global.LOCK_SOUND,
                    R.string.def_lock_sound);
            loadStringSetting(stmt, Settings.Global.UNLOCK_SOUND,
                    R.string.def_unlock_sound);
            loadStringSetting(stmt, Settings.Global.TRUSTED_SOUND,
                    R.string.def_trusted_sound);
            loadIntegerSetting(stmt, Settings.Global.POWER_SOUNDS_ENABLED,
                    R.integer.def_power_sounds_enabled);
            loadStringSetting(stmt, Settings.Global.LOW_BATTERY_SOUND,
                    R.string.def_low_battery_sound);
            loadIntegerSetting(stmt, Settings.Global.DOCK_SOUNDS_ENABLED,
                    R.integer.def_dock_sounds_enabled);
            loadIntegerSetting(stmt, Settings.Global.DOCK_SOUNDS_ENABLED_WHEN_ACCESSIBILITY,
                    R.integer.def_dock_sounds_enabled_when_accessibility);
            loadStringSetting(stmt, Settings.Global.DESK_DOCK_SOUND,
                    R.string.def_desk_dock_sound);
            loadStringSetting(stmt, Settings.Global.DESK_UNDOCK_SOUND,
                    R.string.def_desk_undock_sound);
            loadStringSetting(stmt, Settings.Global.CAR_DOCK_SOUND,
                    R.string.def_car_dock_sound);
            loadStringSetting(stmt, Settings.Global.CAR_UNDOCK_SOUND,
                    R.string.def_car_undock_sound);
            loadStringSetting(stmt, Settings.Global.CHARGING_STARTED_SOUND,
                    R.string.def_wireless_charging_started_sound);

            loadIntegerSetting(stmt, Settings.Global.DOCK_AUDIO_MEDIA_ENABLED,
                    R.integer.def_dock_audio_media_enabled);

            loadSetting(stmt, Settings.Global.SET_INSTALL_LOCATION, 0);
            loadSetting(stmt, Settings.Global.DEFAULT_INSTALL_LOCATION,
                    PackageHelper.APP_INSTALL_AUTO);

            // Set default cdma emergency tone
            loadSetting(stmt, Settings.Global.EMERGENCY_TONE, 0);

            // Set default cdma call auto retry
            loadSetting(stmt, Settings.Global.CALL_AUTO_RETRY, 0);

            // Set the preferred network mode to target desired value or Default
            // value defined in system property
            String val = "";
            String mode;
            for (int phoneId = 0;
                    phoneId < TelephonyManager.getDefault().getPhoneCount(); phoneId++) {
                mode = TelephonyManager.getTelephonyProperty(phoneId,
                        "ro.telephony.default_network",
                        Integer.toString(RILConstants.PREFERRED_NETWORK_MODE));
                if (phoneId == 0) {
                    val = mode;
                } else {
                    val = val + "," + mode;
                }
            }
            loadSetting(stmt, Settings.Global.PREFERRED_NETWORK_MODE, val);

            // Set the preferred cdma subscription source to target desired value or default
            // value defined in Phone
            int type = SystemProperties.getInt("ro.telephony.default_cdma_sub",
                    Phone.PREFERRED_CDMA_SUBSCRIPTION);
            loadSetting(stmt, Settings.Global.CDMA_SUBSCRIPTION_MODE, type);

            loadIntegerSetting(stmt, Settings.Global.LOW_BATTERY_SOUND_TIMEOUT,
                    R.integer.def_low_battery_sound_timeout);

            loadIntegerSetting(stmt, Settings.Global.WIFI_SCAN_ALWAYS_AVAILABLE,
                    R.integer.def_wifi_scan_always_available);

            loadIntegerSetting(stmt, Global.HEADS_UP_NOTIFICATIONS_ENABLED,
                    R.integer.def_heads_up_enabled);

            loadSetting(stmt, Settings.Global.DEVICE_NAME, getDefaultDeviceName());

            // Set default lid/cover behaviour according to legacy device config
            final int defaultLidBehavior;
            if (res.getBoolean(com.android.internal.R.bool.config_lidControlsSleep)) {
                // WindowManagerFuncs.LID_BEHAVIOR_SLEEP
                defaultLidBehavior = 1;
            } else if (res.getBoolean(com.android.internal.R.bool.config_lidControlsScreenLock)) {
                // WindowManagerFuncs.LID_BEHAVIOR_LOCK
                defaultLidBehavior = 2;
            } else {
                // WindowManagerFuncs.LID_BEHAVIOR_NONE
                defaultLidBehavior = 0;
            }
            loadSetting(stmt, Settings.Global.LID_BEHAVIOR, defaultLidBehavior);

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

在 loadGlobalSettings(SQLiteDatabase db) 中的设置系统默认值的  
代码中发现  
loadBooleanSetting(stmt, Settings.Global.WIFI_ON,  
R.bool.def_wifi_on);

Settings.Global.WIFI_ON 就是系统 wifi 是否启动的开关而 R.bool.def_wifi_on  
就是 wifi 的默认开关

所以具体修改如下:

```
--- a/frameworks/base/packages/SettingsProvider/res/values/defaults.xml

+++ b/frameworks/base/packages/SettingsProvider/res/values/defaults.xml

@@ -46,7 +46,7 @@

     <bool >true</bool>

     <bool >true</bool>

     <bool >true</bool>

-    <bool >false</bool>

+    <bool >true</bool><!-- motify-->

     <!-- 0 == never, 1 == only when plugged in, 2 == always -->

     <integer >2</integer>

     <bool >true</bool>

```

编译验证后发现 wifi 已经默认打开了
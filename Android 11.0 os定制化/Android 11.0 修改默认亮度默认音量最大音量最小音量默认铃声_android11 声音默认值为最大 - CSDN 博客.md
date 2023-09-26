> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124828085)

### 1. 概述

11.0 的产品定制化开发中，要求对系统亮度 最大音量 最小音量设置默认值 默认铃声等系统默认值做修改  
这些默认值找到相应的值修改就好了

### 2. 修改默认亮度默认音量最大音量最小音量默认铃声核心类

```
frameworks\base\packages\SettingsProvider\res\values\default.xml
frameworks\base\packages\SettingsProvider\src\com\android\providers\settings\DatabaseHelper.java
frameworks/base/media/java/android/media/AudioSystem.java
frameworks/base/media/java/android/media/AudioSystem.java

```

### 3. 修改默认亮度默认音量最大音量最小音量默认铃声的核心功能分析

### 3.1 修改系统默认亮度

系统默认亮度值是在 SettingProvider 中定义的，现在看下相应的配置文件

```
frameworks\base\packages\SettingsProvider\res\values\default.xml
<!-- Default screen brightness, from 0 to 255.  102 is 40%. -->
<integer >114</integer>
<bool >false</bool>

```

def_screen_brightness 就是系统默认亮度值修改为新的默认值就行

下面看 DatabaseHelper.java 中的应用加载系统默认值

```
frameworks\base\packages\SettingsProvider\src\com\android\providers\settings\DatabaseHelper.java
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

}
loadIntegerSetting(stmt, Settings.System.SCREEN_BRIGHTNESS,
R.integer.def_screen_brightness);

```

通过下面代码设置屏幕亮度到数据库

```
loadBooleanSetting(stmt, Settings.System.SCREEN_BRIGHTNESS_MODE,
R.bool.def_screen_brightness_automatic_mode);

```

### 3.2 在 AudioSystem.java 源码中设置默认音量

```
frameworks/base/media/java/android/media/AudioSystem.java
public static int getDefaultStreamVolume(int streamType) {
return DEFAULT_STREAM_VOLUME[streamType];
}
public static int[] DEFAULT_STREAM_VOLUME = new int[] {
4,  // STREAM_VOICE_CALL
7,  // STREAM_SYSTEM
5,  // STREAM_RING
12, // STREAM_MUSIC
6,  // STREAM_ALARM
5,  // STREAM_NOTIFICATION
7,  // STREAM_BLUETOOTH_SCO
7,  // STREAM_SYSTEM_ENFORCED
5, // STREAM_DTMF
5, // STREAM_TTS
5, // STREAM_ACCESSIBILITY
};

```

DEFAULT_STREAM_VOLUME 就是系统各种参数的默认值  
根据下标类型来获取默认音量值

### 3.3 修改最大音量和最小音量 AudioService.java

frameworks/base/services/core/java/com/android/server/audio/AudioService.java  
MAX_STREAM_VOLUME 中设置各种音量的最大值

```
/** Maximum volume index values for audio streams */
protected static int[] MAX_STREAM_VOLUME = new int[] {
5,  // STREAM_VOICE_CALL
7,  // STREAM_SYSTEM
7,  // STREAM_RING
15, // STREAM_MUSIC
7,  // STREAM_ALARM
7,  // STREAM_NOTIFICATION
15, // STREAM_BLUETOOTH_SCO
7,  // STREAM_SYSTEM_ENFORCED
15, // STREAM_DTMF
15, // STREAM_TTS
15  // STREAM_ACCESSIBILITY
};

```

MIN_STREAM_VOLUME 设置各种音量的最小值

```
/** Minimum volume index values for audio streams */
protected static int[] MIN_STREAM_VOLUME = new int[] {
1,  // STREAM_VOICE_CALL
0,  // STREAM_SYSTEM
0,  // STREAM_RING
0,  // STREAM_MUSIC
1,  // STREAM_ALARM
0,  // STREAM_NOTIFICATION
0,  // STREAM_BLUETOOTH_SCO
0,  // STREAM_SYSTEM_ENFORCED
0,  // STREAM_DTMF
0,  // STREAM_TTS
1   // STREAM_ACCESSIBILITY
};

```

获取 Music 的最小最大值

```
private int getSafeUsbMediaVolumeIndex() {
    // determine UI volume index corresponding to the wanted safe gain in dBFS
    int min = MIN_STREAM_VOLUME[AudioSystem.STREAM_MUSIC];
    int max = MAX_STREAM_VOLUME[AudioSystem.STREAM_MUSIC];

    mSafeUsbMediaVolumeDbfs = mContext.getResources().getInteger(
            com.android.internal.R.integer.config_safe_media_volume_usb_mB) / 100.0f;

    while (Math.abs(max - min) > 1) {
        int index = (max + min) / 2;
        float gainDB = AudioSystem.getStreamVolumeDB(
                AudioSystem.STREAM_MUSIC, index, AudioSystem.DEVICE_OUT_USB_HEADSET);
        if (Float.isNaN(gainDB)) {
            //keep last min in case of read error
            break;
        } else if (gainDB == mSafeUsbMediaVolumeDbfs) {
            min = index;
            break;
        } else if (gainDB < mSafeUsbMediaVolumeDbfs) {
            min = index;
        } else {
            max = index;
        }
    }
    return min * 10;
}

private VolumeStreamState(String settingName, int streamType) {
        mVolumeIndexSettingName = settingName;

        mStreamType = streamType;
        mIndexMin = MIN_STREAM_VOLUME[streamType] * 10;
        mIndexMax = MAX_STREAM_VOLUME[streamType] * 10;
        AudioSystem.initStreamVolume(streamType, mIndexMin / 10, mIndexMax / 10);

        readSettings();
        mVolumeChanged = new Intent(AudioManager.VOLUME_CHANGED_ACTION);
        mVolumeChanged.putExtra(AudioManager.EXTRA_VOLUME_STREAM_TYPE, mStreamType);
        //UNISOC:bug1031064 Duplicate key in ArrayMap, multi thread add key twice.
        mVolumeChanged.putExtra(AudioManager.EXTRA_VOLUME_STREAM_VALUE, 0);
        mVolumeChanged.putExtra(AudioManager.EXTRA_PREV_VOLUME_STREAM_VALUE, 0);
        mVolumeChanged.putExtra(AudioManager.EXTRA_VOLUME_STREAM_TYPE_ALIAS, 0);
        Log.d(TAG, "init VOLUME_CHANGED_ACTION intent for "+mStreamType);
        mStreamDevicesChanged = new Intent(AudioManager.STREAM_DEVICES_CHANGED_ACTION);
        mStreamDevicesChanged.putExtra(AudioManager.EXTRA_VOLUME_STREAM_TYPE, mStreamType);
    }

```

在 VolumeStreamState 中通过 MIN_STREAM_VOLUME 获取最小值，通过 MAX_STREAM_VOLUME 获取最大值  
默认铃声

```
设置闹钟铃声
build/make/target/product/handheld_system.mk:90:
ro.config.alarm_alert=Alarm_Classic.ogg
设置提示音和通知
build/make/target/product/gsi_common.mk:29:    ro.config.notification_sound=pixiedust.ogg 
build/make/target/product/handheld_system.mk:89:    ro.config.notification_sound=OnTheHunt.ogg 
build/make/target/product/full_base.mk:46:    ro.config.notification_sound=pixiedust.ogg
设置手机铃声
build/make/target/product/gsi_common.mk:28:    ro.config.ringtone=Ring_Synth_04.ogg 
build/make/target/product/mainline.mk:30:    ro.config.ringtone=Ring_Synth_04.ogg 
build/make/target/product/full_base.mk:45:    ro.config.ringtone=Ring_Synth_04.ogg \

```
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124739360)

### 1. 概述

在 11.0 的定制化产品的需求的需要 要求增加屏保功能，设置屏保前提必须是是需要设置休眠时间大于屏保时间，当屏保时间大于休眠时间时，休眠以后 屏保功能就无效，所以就必须设置好屏保时间  
最终效果图:  
![](https://img-blog.csdnimg.cn/e025667c4ea94f2293d76e52f86b9ccd.png#pic_center)

不然休眠时间小于屏保时间 就会先进入休眠 就没有屏保效果了

### 2.Settings 增加屏保功能的核心类

```
packages/apps/Settings/res/xml/dream_fragment_overview.xml
/packages/apps/Settings/res/values/arrays.xml
packages/apps/Settings/res/values/strings.xml
packages\apps\Settings\src\com\android\settings\dream\ScreenSaverTimerPreferenceController.java
packages/apps/Settings/src/com/android/settings/dream/StartNowPreferenceController.java

```

### 3.Settings 增加屏保功能的核心功能分析和实现

### 3.1 在系统 Settings 增加屏保相关资源

设置–> 显示–> 屏保中添加 ListPreference

```
packages/apps/Settings/res/xml/dream_fragment_overview.xml
<ListPreference
   android:key="screensaver_timer"

   android:title="@string/screensaver_timer"

   android:entries="@array/screensaver_timer_entries"

   android:entryValues="@array/screensaver_timer_values"

   settings:controller="com.android.settings.dream.ScreenSaverTimerPreferenceController" />

/packages/apps/Settings/res/values/arrays.xml
<!--  Titles for screensaver preference. -->
<string-array  >
   <item>@string/screensaver_timer_1min</item>

   <item>@string/screensaver_timer_5mins</item>

   <item>@string/screensaver_timer_10mins</item>

   <item>@string/screensaver_timer_15mins</item>

   <item>@string/screensaver_timer_30mins</item>

</string-array>
<!--  Values for screensaver preference. -->
<string-array  >
   <item>60000</item>

   <item>300000</item>

   <item>600000</item>

   <item>900000</item>

   <item>1800000</item>

</string-array>

packages/apps/Settings/res/values/strings.xml
<string >Screen Saver</string>
<string >1 min</string>
<string >5 min</string>
<string >10 min</string>
<string >15 min</string>
<string >30 min</string>
<string </string>

packages/apps/Settings/res/values-zh-rCN/strings.xml
<string >屏保</string>
<string >1 分钟</string>
<string >5 分钟</string>
<string >10 分钟</string>
<string >15 分钟</string>
<string >30 分钟</string>
<string </string>

```

### 3.2 添加设置屏保时间 ScreenSaverTimerPreferenceController.java

通过增加设置屏保时间来控制屏保的时间操作  
路径:  
packages\apps\Settings\src\com\android\settings\dream\ScreenSaverTimerPreferenceController.java

```
/*
Copyright (C) 2018 The Android Open Source Project
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at
 http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License
*/
package com.android.settings.dream;
import android.content.Context;
import android.provider.Settings;
import androidx.preference.ListPreference;
import androidx.preference.Preference;
import androidx.preference.PreferenceScreen;
import android.util.FeatureFlagUtils;
import com.android.settings.R;
import com.android.settings.core.BasePreferenceController;
import com.android.settings.core.PreferenceControllerMixin;
import com.android.settingslib.core.AbstractPreferenceController;
/**
Setting where user can pick if SystemUI will be light, dark or try to match
the wallpaper colors.
*/
public class ScreenSaverTimerPreferenceController extends BasePreferenceController
implements Preference.OnPreferenceChangeListener {private ListPreference mScreenSaverTimeoutPref;
public static final String SCREENSAVER_TIMEOUT= "screensavers.timeout";
public ScreenSaverTimerPreferenceController(Context context, String preferenceKey) {
super(context, preferenceKey);
}@Override
public int getAvailabilityStatus() {
return AVAILABLE;
}@Override
public void displayPreference(PreferenceScreen screen) {
super.displayPreference(screen);
mScreenSaverTimeoutPref = (ListPreference) screen.findPreference(getPreferenceKey());
int value = Settings.Secure.getInt(mContext.getContentResolver(), SCREENSAVER_TIMEOUT, 60000);
mScreenSaverTimeoutPref.setValue(Integer.toString(value));
}@Override
public boolean onPreferenceChange(Preference preference, Object newValue) {
int value = Integer.parseInt((String) newValue);
Settings.Secure.putInt(mContext.getContentResolver(), SCREENSAVER_TIMEOUT, value);
refreshSummary(preference);
return true;
}@Override
public CharSequence getSummary() {
int value = Settings.Secure.getInt(mContext.getContentResolver(), SCREENSAVER_TIMEOUT, 60000);
int index = mScreenSaverTimeoutPref.findIndexOfValue(Integer.toString(value)); return mContext.getString(R.string.screensaver_timeout_summary, mScreenSaverTimeoutPref.getEntries()[index]);
}
}

```

在 ScreenSaverTimerPreferenceController 中，通过设置 onPreferenceChange(Preference preference, Object newValue) 监听来判断屏保时间的改变，然后设置屏保时间

### 3.3 去掉屏保的启动时间

```
packages/apps/Settings/src/com/android/settings/dream/StartNowPreferenceController.java
@@ -40,7 +40,7 @@ public class StartNowPreferenceController extends AbstractPreferenceController i
 @Override
 public boolean isAvailable() {

   return true;

   return false;
}

```

在 StartNowPreferenceController.java 的 isAvailable() 表示是否启动屏保时间，设置为 false 就代表不启动屏保时间

去掉启动时间一律不

```
--- a/packages/apps/Settings/src/com/android/settings/dream/WhenToDreamPreferenceController.java
+++ b/packages/apps/Settings/src/com/android/settings/dream/WhenToDreamPreferenceController.java
@@ -46,7 +46,7 @@ public class WhenToDreamPreferenceController extends AbstractPreferenceControlle
 @Override
 public boolean isAvailable() {

   return true;

   return false;
}

```

在 WhenToDreamPreferenceController.java 的 isAvailable() 表示是否启动时间一律不，设置为 false 就代表不启动启动时间一律不这个选项

framework 层的修改  
修改 "config_dreamsEnabledOnBattery" 修改为 true 即实现任何时候都能进入屏保  
修改默认屏保的值 config_dreamsDefaultComponent 用时钟做屏保

```
--- a/frameworks/base/core/res/res/values/config.xml
+++ b/frameworks/base/core/res/res/values/config.xml
@@ -2274,10 +2274,10 @@
<!-- If supported and enabled, are dreams activated when asleep and charging? (by default) -->
<bool >false</bool>
<!-- ComponentName of the default dream (Settings.Secure.DEFAULT_SCREENSAVER_COMPONENT) -->
<string >com.google.android.deskclock/com.android.deskclock.Screensaver</string>
<string >com.android.deskclock/com.android.deskclock.Screensaver</string><!-- Are we allowed to dream while not plugged in? -->
<bool >false</bool>
<bool >true</bool>

```

接下来在 PowerManagerService.java 中修改进入屏保的条件

```
frameworks/base/services/core/java/com/android/server/power/PowerManagerService.java
private boolean updateWakefulnessLocked(int dirty) {
boolean changed = false;
if ((dirty & (DIRTY_WAKE_LOCKS | DIRTY_USER_ACTIVITY | DIRTY_BOOT_COMPLETED
| DIRTY_WAKEFULNESS | DIRTY_STAY_ON | DIRTY_PROXIMITY_POSITIVE
| DIRTY_DOCK_STATE)) != 0) {
if (mWakefulness == WAKEFULNESS_AWAKE && isItBedTimeYetLocked()) {
if (DEBUG_SPEW) {
Slog.d(TAG, "updateWakefulnessLocked: Bed time...");
}
final long time = SystemClock.uptimeMillis();
if (shouldNapAtBedTimeLocked()) {
changed = napNoUpdateLocked(time, Process.SYSTEM_UID);
} else {
changed = goToSleepNoUpdateLocked(time,
PowerManager.GO_TO_SLEEP_REASON_TIMEOUT, 0, Process.SYSTEM_UID);
}
}
}
return changed;
}

/**
* Returns true if the device should automatically nap and start dreaming when the user
* activity timeout has expired and it's bedtime.
*/
private boolean shouldNapAtBedTimeLocked() {
return true/mDreamsActivateOnSleepSetting|| (mDreamsActivateOnDockSetting&& mDockState != Intent.EXTRA_DOCK_STATE_UNDOCKED)/;
}
private void updateUserActivitySummaryLocked(long now, int dirty) {
    // Update the status of the user activity timeout timer.
    if ((dirty & (DIRTY_WAKE_LOCKS | DIRTY_USER_ACTIVITY
            | DIRTY_WAKEFULNESS | DIRTY_SETTINGS)) != 0) {
        mHandler.removeMessages(MSG_USER_ACTIVITY_TIMEOUT);

        long nextTimeout = 0;
        if (mWakefulness == WAKEFULNESS_AWAKE
                || mWakefulness == WAKEFULNESS_DREAMING
                || mWakefulness == WAKEFULNESS_DOZING) {
            final long sleepTimeout = getSleepTimeoutLocked();
            final long screenOffTimeout = getScreenOffTimeoutLocked(sleepTimeout);
            final long screenDimDuration = getScreenDimDurationLocked(screenOffTimeout);
            final boolean userInactiveOverride = mUserInactiveOverrideFromWindowManager;
            final long nextProfileTimeout = getNextProfileTimeoutLocked(now);

            mUserActivitySummary = 0;
            if (mLastUserActivityTime >= mLastWakeTime) {
                nextTimeout = mLastUserActivityTime
                        + screenOffTimeout - screenDimDuration;
                if (now < nextTimeout) {
                    mUserActivitySummary = USER_ACTIVITY_SCREEN_BRIGHT;
                } else {
                    nextTimeout = mLastUserActivityTime + screenOffTimeout;
                    if (now < nextTimeout) {
                        mUserActivitySummary = USER_ACTIVITY_SCREEN_DIM;
                    }
                }
            }
            if (mUserActivitySummary == 0
                    && mLastUserActivityTimeNoChangeLights >= mLastWakeTime) {
                nextTimeout = mLastUserActivityTimeNoChangeLights + screenOffTimeout;
                if (now < nextTimeout) {
                    if (mDisplayPowerRequest.policy == DisplayPowerRequest.POLICY_BRIGHT
                            || mDisplayPowerRequest.policy == DisplayPowerRequest.POLICY_VR) {
                        mUserActivitySummary = USER_ACTIVITY_SCREEN_BRIGHT;
                    } else if (mDisplayPowerRequest.policy == DisplayPowerRequest.POLICY_DIM) {
                        mUserActivitySummary = USER_ACTIVITY_SCREEN_DIM;
                    }
                }
            }

            if (mUserActivitySummary == 0) {
                if (sleepTimeout >= 0) {
                    final long anyUserActivity = Math.max(mLastUserActivityTime,
                            mLastUserActivityTimeNoChangeLights);
                    if (anyUserActivity >= mLastWakeTime) {
                        nextTimeout = anyUserActivity + sleepTimeout;
                        if (now < nextTimeout) {
                            mUserActivitySummary = USER_ACTIVITY_SCREEN_DREAM;
                        }
                    }
                } else {
                    mUserActivitySummary = USER_ACTIVITY_SCREEN_DREAM;
                    nextTimeout = -1;
                }
            }

            if (mUserActivitySummary != USER_ACTIVITY_SCREEN_DREAM && userInactiveOverride) {
                if ((mUserActivitySummary &
                        (USER_ACTIVITY_SCREEN_BRIGHT | USER_ACTIVITY_SCREEN_DIM)) != 0) {
                    // Device is being kept awake by recent user activity
                    if (nextTimeout >= now && mOverriddenTimeout == -1) {
                        // Save when the next timeout would have occurred
                        mOverriddenTimeout = nextTimeout;
                    }
                }
                mUserActivitySummary = USER_ACTIVITY_SCREEN_DREAM;
                nextTimeout = -1;
            }

            if ((mUserActivitySummary & USER_ACTIVITY_SCREEN_BRIGHT) != 0
                    && (mWakeLockSummary & WAKE_LOCK_STAY_AWAKE) == 0) {
                nextTimeout = mAttentionDetector.updateUserActivity(nextTimeout);
            }

            if (nextProfileTimeout > 0) {
                nextTimeout = Math.min(nextTimeout, nextProfileTimeout);
            }

            if (mUserActivitySummary != 0 && nextTimeout >= 0) {
                scheduleUserInactivityTimeout(nextTimeout);
            }
        } else {
            mUserActivitySummary = 0;
        }

        if (DEBUG_SPEW) {
            Slog.d(TAG, "updateUserActivitySummaryLocked: mWakefulness="
                    + PowerManagerInternal.wakefulnessToString(mWakefulness)
                    + ", mUserActivitySummary=0x" + Integer.toHexString(mUserActivitySummary)
                    + ", nextTimeout=" + TimeUtils.formatUptime(nextTimeout));
        }
    }
}

```

在 PowerManagerService.java 中的 shouldNapAtBedTimeLocked(）判断进入屏保的条件所以直接返回 true 就表示进不到屏保

具体补丁如下:

```
--- a/frameworks/base/services/core/java/com/android/server/power/PowerManagerService.java
+++ b/frameworks/base/services/core/java/com/android/server/power/PowerManagerService.java
         mHandler.removeMessages(MSG_USER_ACTIVITY_TIMEOUT);

         long nextTimeout = 0;

                  long screenSaverNextTimeout = 0;

       if (mWakefulness == WAKEFULNESS_AWAKE
               || mWakefulness == WAKEFULNESS_DREAMING
               || mWakefulness == WAKEFULNESS_DOZING) {

@@ -2178,13 +2179,18 @@ public final class PowerManagerService extends SystemService
final long screenDimDuration = getScreenDimDurationLocked(screenOffTimeout);
final boolean userInactiveOverride = mUserInactiveOverrideFromWindowManager;
final long nextProfileTimeout = getNextProfileTimeoutLocked(now);
           final long screenSaverTimeout = Settings.Secure.getInt(mContext.getContentResolver(),"screensavers.timeout",60000);

           mUserActivitySummary = 0;
           if (mLastUserActivityTime >= mLastWakeTime) {
               nextTimeout = mLastUserActivityTime
                       + screenOffTimeout - screenDimDuration;

                                  screenSaverNextTimeout = mLastUserActivityTime + screenSaverTimeout;

               if (now < nextTimeout) {

                   mUserActivitySummary = USER_ACTIVITY_SCREEN_BRIGHT;

                                          if( screenOffTimeout > screenSaverTimeout && now > screenSaverNextTimeout ){

                       mUserActivitySummary = USER_ACTIVITY_SCREEN_DREAM;

                   }else{//否则保持屏保亮屏

                       mUserActivitySummary = USER_ACTIVITY_SCREEN_BRIGHT;

                   }

               } else {
                   nextTimeout = mLastUserActivityTime + screenOffTimeout;
                   if (now < nextTimeout) {

@@ -2216,7 +2222,7 @@ public final class PowerManagerService extends SystemService
}
}
} else {
                   mUserActivitySummary = USER_ACTIVITY_SCREEN_DREAM;

                   //mUserActivitySummary = USER_ACTIVITY_SCREEN_DREAM;

                   nextTimeout = -1;
               }
           }

@@ -2244,7 +2250,11 @@ public final class PowerManagerService extends SystemService
}
             if (mUserActivitySummary != 0 && nextTimeout >= 0) {

               scheduleUserInactivityTimeout(nextTimeout);

               if(screenOffTimeout > screenSaverTimeout && now <= screenSaverNextTimeout ){

                   scheduleUserInactivityTimeout(screenSaverNextTimeout);

               }else{//否则发送进入休眠的信息

                   scheduleUserInactivityTimeout(nextTimeout);

               }

           }
       } else {
           mUserActivitySummary = 0;

@@ -2362,9 +2372,9 @@ public final class PowerManagerService extends SystemService
* activity timeout has expired and it's bedtime.
*/
private boolean shouldNapAtBedTimeLocked() {
   return mDreamsActivateOnSleepSetting

   return true/*mDreamsActivateOnSleepSetting

           || (mDreamsActivateOnDockSetting

                   && mDockState != Intent.EXTRA_DOCK_STATE_UNDOCKED);

                   && mDockState != Intent.EXTRA_DOCK_STATE_UNDOCKED)*/;
}

```
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124955226)

### 1. 概述

在 11.0 的产品开发中，对于 wifi 的功能定制需求，有要求需要通过系统属性来控制 wifi 开关是否可以打开  
来控制是否可以连接 wifi, 打开控制 wifi 的功能

### 2.wifi 开关控制的核心代码

```
frameworks/base/wifi/java/android/net/wifi/WifiManager.java
packages/apps/Settings/src/com/android/settings/wifi/WifiEnabler.java
packages/apps/Settings/src/com/android/settings/wifi/WifiSettings.java

```

### 3.wifi 开关控制功能分析和实现

关于 wifi 的管理是在 wifiManager 中负责管理的，而在系统 Setting 中的网络菜单中，开关打开 wifi  
然后连接 wifi 实现联网功能

### 3.1WifiManager 打开关闭 wifi

首选看下 WifiManger 关于管理 wifi 的功能

```
@Deprecated
      public boolean setWifiEnabled(boolean enabled) {
          try {
              return mService.setWifiEnabled(mContext.getOpPackageName(), enabled);
          } catch (RemoteException e) {
              throw e.rethrowFromSystemServer();
          }
      }
  
      /**
       * Gets the Wi-Fi enabled state.
       * @return One of {@link #WIFI_STATE_DISABLED},
       *         {@link #WIFI_STATE_DISABLING}, {@link #WIFI_STATE_ENABLED},
       *         {@link #WIFI_STATE_ENABLING}, {@link #WIFI_STATE_UNKNOWN}
       * @see #isWifiEnabled()
       */
      public int getWifiState() {
          try {
              return mService.getWifiEnabledState();
          } catch (RemoteException e) {
              throw e.rethrowFromSystemServer();
          }
      }
      具体修改为:
--- a/frameworks/base/wifi/java/android/net/wifi/WifiManager.java
+++ b/frameworks/base/wifi/java/android/net/wifi/WifiManager.java
@@ -19,7 +19,7 @@ package android.net.wifi;
import static android.Manifest.permission.ACCESS_FINE_LOCATION;
import static android.Manifest.permission.ACCESS_WIFI_STATE;
import static android.Manifest.permission.READ_WIFI_CREDENTIAL;
-
+import android.os.SystemProperties;
import android.annotation.CallbackExecutor;
import android.annotation.IntDef;
import android.annotation.NonNull;
@@ -2455,6 +2455,11 @@ public class WifiManager {
*/

@Deprecated
public boolean setWifiEnabled(boolean enabled) {
+               String flag = SystemProperties.get("persist.sys.enableWIFI", "true");
+               Log.d("dong","setWifiEnabled enableWIFI:"+flag);
+               if(!"true".equals(flag)){
+                       return false;
+               }
try {
return mService.setWifiEnabled(mContext.getOpPackageName(), enabled);
} catch (RemoteException e) {

```

通过 setWifiEnabled(boolean enabled) 来控制 wifi 是否开启和关闭，所以  
可以在这里添加系统属性来控制是否打开 wifi

### 3.2WifiEnabler 控制 wifi 开关修改

在系统 Settings 中通过 WifiEnabler.java 来开启 wifi 开关实现联网的

```
   private final BroadcastReceiver mReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            String action = intent.getAction();
            if (WifiManager.WIFI_STATE_CHANGED_ACTION.equals(action)) {
                handleWifiStateChanged(mWifiManager.getWifiState());
            } else if (WifiManager.SUPPLICANT_STATE_CHANGED_ACTION.equals(action)) {
                if (!mConnected.get()) {
                    handleStateChanged(WifiInfo.getDetailedStateOf((SupplicantState)
                            intent.getParcelableExtra(WifiManager.EXTRA_NEW_STATE)));
                }
            } else if (WifiManager.NETWORK_STATE_CHANGED_ACTION.equals(action)) {
                NetworkInfo info = (NetworkInfo) intent.getParcelableExtra(
                        WifiManager.EXTRA_NETWORK_INFO);
                mConnected.set(info.isConnected());
                handleStateChanged(info.getDetailedState());
            }
        }
    };
    
--- a/packages/apps/Settings/src/com/android/settings/wifi/WifiEnabler.java
+++ b/packages/apps/Settings/src/com/android/settings/wifi/WifiEnabler.java
@@ -30,9 +30,9 @@ import android.os.UserHandle;
import android.os.UserManager;
import android.provider.Settings;
import android.widget.Toast;
-
+import android.os.SystemProperties;
import androidx.annotation.VisibleForTesting;
-
+import android.util.Log;
import com.android.settings.R;
import com.android.settings.widget.SwitchWidgetController;
import com.android.settingslib.RestrictedLockUtils.EnforcedAdmin;
@@ -141,6 +141,11 @@ public class WifiEnabler implements SwitchWidgetController.OnSwitchChangeListene
}

private void handleWifiStateChanged(int state) {
+               String flag = SystemProperties.get("persist.sys.enableWIFI", "false");
+               Log.d("dong","setSwitchBarChecked enableWIFI:"+flag);
+               if(!"true".equals(flag)){
+                       return ;
+               }
// Clear any previous state
mSwitchWidget.setDisabledByAdmin(null);

@@ -196,6 +201,11 @@ public class WifiEnabler implements SwitchWidgetController.OnSwitchChangeListene

@Override
public boolean onSwitchToggled(boolean isChecked) {
+               String flag = SystemProperties.get("persist.sys.enableWIFI", "true");
+               Log.d("dong","onSwitchToggledenableWIFI:"+flag);
+               if(!"true".equals(flag)){
+                       return false;
+               }
//Do nothing if called as a result of a state machine event
if (mStateMachineEvent) {
return true;

```

mReceiver 监听广播接收 wifi 状态改变调用 handleStateChanged 设置改变后的 wifi 状态在 handleWifiStateChanged(int state)  
和 onSwitchToggled(boolean isChecked) 根据系统属性设置是否打开 wifi

### 3.3wifi 开关按钮

```
      @Override
      public void onStart() {
          super.onStart();
  
          // On/off switch is hidden for Setup Wizard (returns null)
          mWifiEnabler = createWifiEnabler();
  
          if (mIsRestricted) {
              restrictUi();
              return;
          }
  
          onWifiStateChanged(mWifiManager.getWifiState());
      }
	   @Override
     public void onViewCreated(View view, Bundle savedInstanceState) {
         super.onViewCreated(view, savedInstanceState);
         final Activity activity = getActivity();
         if (activity != null) {
             mProgressHeader = setPinnedHeaderView(R.layout.progress_header)
                     .findViewById(R.id.progress_bar_animation);
             setProgressBarVisible(false);
         }
         ((SettingsActivity) activity).getSwitchBar().setSwitchBarText(
                 R.string.wifi_settings_master_switch_title,
                 R.string.wifi_settings_master_switch_title);
     }
 
     @Override
     public void onCreate(Bundle icicle) {
         super.onCreate(icicle);
 
         if (FeatureFlagUtils.isEnabled(getContext(), FeatureFlagUtils.SETTINGS_WIFITRACKER2)) {
             final Intent intent = new Intent("android.settings.WIFI_SETTINGS2");
             final Bundle extras = getActivity().getIntent().getExtras();
             if (extras != null) {
                 intent.putExtras(extras);
             }
             getContext().startActivity(intent);
             finish();
             return;
         }
 
         // TODO(b/37429702): Add animations and preference comparator back after initial screen is
         // loaded (ODR).
         setAnimationAllowed(false);
 
         addPreferences();
 
         mIsRestricted = isUiRestricted();
     }
 
--- a/packages/apps/Settings/src/com/android/settings/wifi/WifiSettings.java
+++ b/packages/apps/Settings/src/com/android/settings/wifi/WifiSettings.java
@@ -50,7 +50,7 @@ import android.view.Menu;
import android.view.MenuItem;
import android.view.View;
import android.widget.Toast;
-
+import android.os.SystemProperties;
import androidx.annotation.VisibleForTesting;
import androidx.preference.Preference;
import androidx.preference.PreferenceCategory;
@@ -221,6 +221,12 @@ public class WifiSettings extends RestrictedSettingsFragment
((SettingsActivity) activity).getSwitchBar().setSwitchBarText(
R.string.wifi_settings_master_switch_title,
R.string.wifi_settings_master_switch_title);
+               String flag = SystemProperties.get("persist.sys.enableWIFI", "true");
+               Log.d("dong","isWIFISupported() onViewCreated:"+flag);
+               if(!"true".equals(flag)){
+                       Log.d("dong","isWIFISupported() onViewCreated:"+flag);
+                        ((SettingsActivity) activity).getSwitchBar().setEnabled(false);
+               }
}

```

在 onViewCreated(View view, Bundle savedInstanceState) 添加设置是否默认打开 wifi 功能 通过系统属性来控制这个功能
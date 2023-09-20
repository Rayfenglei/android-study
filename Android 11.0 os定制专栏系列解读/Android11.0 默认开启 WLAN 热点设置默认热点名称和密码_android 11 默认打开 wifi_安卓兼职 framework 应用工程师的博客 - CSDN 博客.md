> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124828391)

### 1. 概述

11.0 定制产品需要在开机后默认开启热点的产品，这就需要在开机后默认打开热点而开机后第一个弹出来的就是锁屏界面 所以就想在锁屏界面收到开机广播后添加开启热点，实现开启热点的功能

### 2. 默认开启 WLAN 热点设置默认热点名称和密码的核心类

```
/framework/base/packages/SystemUI/src/com/android/keyguard/KeyguardUpdateMonitor.java
frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiApConfigStore.java

```

### 3. 默认开启 WLAN 热点设置默认热点名称和密码的核心功能分析和实现

### 3.1 在 KeyguardUpdateMonitor.java 中添加开启默认热点功能

首先找到收到开机广播的功能，然后在收到开机广播后开启热点

```
/framework/base/packages/SystemUI/src/com/android/keyguard/KeyguardUpdateMonitor.java
@VisibleForTesting
protected final BroadcastReceiver mBroadcastReceiver = new BroadcastReceiver() {
@Override
public void onReceive(Context context, Intent intent) {
final String action = intent.getAction();
if (DEBUG) Log.d(TAG, "received broadcast " + action);
    if (Intent.ACTION_TIME_TICK.equals(action)
            || Intent.ACTION_TIME_CHANGED.equals(action)) {
        mHandler.sendEmptyMessage(MSG_TIME_UPDATE);
    } else if (Intent.ACTION_TIMEZONE_CHANGED.equals(action)) {
        final Message msg = mHandler.obtainMessage(
                MSG_TIMEZONE_UPDATE, intent.getStringExtra("time-zone"));
        mHandler.sendMessage(msg);
    } else if (Intent.ACTION_BATTERY_CHANGED.equals(action)) {
        final int status = intent.getIntExtra(EXTRA_STATUS, BATTERY_STATUS_UNKNOWN);
        final int plugged = intent.getIntExtra(EXTRA_PLUGGED, 0);
        final int level = intent.getIntExtra(EXTRA_LEVEL, 0);
        final int health = intent.getIntExtra(EXTRA_HEALTH, BATTERY_HEALTH_UNKNOWN);

        final int maxChargingMicroAmp = intent.getIntExtra(EXTRA_MAX_CHARGING_CURRENT, -1);
        int maxChargingMicroVolt = intent.getIntExtra(EXTRA_MAX_CHARGING_VOLTAGE, -1);
        final int maxChargingMicroWatt;

        if (maxChargingMicroVolt <= 0) {
            maxChargingMicroVolt = DEFAULT_CHARGING_VOLTAGE_MICRO_VOLT;
        }
        if (maxChargingMicroAmp > 0) {
            // Calculating muW = muA * muV / (10^6 mu^2 / mu); splitting up the divisor
            // to maintain precision equally on both factors.
            maxChargingMicroWatt = (maxChargingMicroAmp / 1000)
                    * (maxChargingMicroVolt / 1000);
        } else {
            maxChargingMicroWatt = -1;
        }
        final Message msg = mHandler.obtainMessage(
                MSG_BATTERY_UPDATE, new BatteryStatus(status, level, plugged, health,
                        maxChargingMicroWatt));
        mHandler.sendMessage(msg);
    } else if (TelephonyIntents.ACTION_SIM_STATE_CHANGED.equals(action)) {
        SimData args = SimData.fromIntent(intent);
        // ACTION_SIM_STATE_CHANGED is rebroadcast after unlocking the device to
        // keep compatibility with apps that aren't direct boot aware.
        // SysUI should just ignore this broadcast because it was already received
        // and processed previously.
        String stateExtra = intent.getStringExtra(IccCardConstants.INTENT_KEY_ICC_STATE);
        if (intent.getBooleanExtra(TelephonyIntents.EXTRA_REBROADCAST_ON_UNLOCK, false)
                    || IccCardConstants.INTENT_VALUE_ICCID_LOADED.equals(stateExtra)) {
            // Guarantee mTelephonyCapable state after SysUI crash and restart
            if (args.simState == State.ABSENT) {
                mHandler.obtainMessage(MSG_TELEPHONY_CAPABLE, true).sendToTarget();
            }
            return;
        }
        if (DEBUG_SIM_STATES) {
            Log.v(TAG, "action " + action + " state: " + stateExtra
                + " slotId: " + args.slotId + " subid: " + args.subId);
        }
        mHandler.obtainMessage(MSG_SIM_STATE_CHANGE, args.subId, args.slotId, args.simState)
                .sendToTarget();
        /* UNISOC: Bug 926152 Refresh carrier info when sim is under simlock status.@{*/
        if (SimLockUtil.getInstance(mContext).isSimlockStatusChange(intent)) {
            mHandler.sendEmptyMessage(MSG_SIMLOCK_UPDATE);
        }
        /* @} */
    } else if (AudioManager.RINGER_MODE_CHANGED_ACTION.equals(action)) {
        mHandler.sendMessage(mHandler.obtainMessage(MSG_RINGER_MODE_CHANGED,
                intent.getIntExtra(AudioManager.EXTRA_RINGER_MODE, -1), 0));
    } else if (TelephonyManager.ACTION_PHONE_STATE_CHANGED.equals(action)) {
        String state = intent.getStringExtra(TelephonyManager.EXTRA_STATE);
        mHandler.sendMessage(mHandler.obtainMessage(MSG_PHONE_STATE_CHANGED, state));
    } else if (Intent.ACTION_AIRPLANE_MODE_CHANGED.equals(action)) {
        mHandler.sendEmptyMessage(MSG_AIRPLANE_MODE_CHANGED);
       //开机广播
    } else if (Intent.ACTION_BOOT_COMPLETED.equals(action)) {
        dispatchBootCompleted();
//添加开启热点的功能		
         wifiApControl();
    } else if (TelephonyIntents.ACTION_SERVICE_STATE_CHANGED.equals(action)) {
        ServiceState serviceState = ServiceState.newFromBundle(intent.getExtras());
        int subId = intent.getIntExtra(PhoneConstants.SUBSCRIPTION_KEY,
                SubscriptionManager.INVALID_SUBSCRIPTION_ID);
        if (DEBUG) {
            Log.v(TAG, "action " + action + " serviceState=" + serviceState + " subId="
                    + subId);
        }
        mHandler.sendMessage(
                mHandler.obtainMessage(MSG_SERVICE_STATE_CHANGE, subId, 0, serviceState));
    } else if (DevicePolicyManager.ACTION_DEVICE_POLICY_MANAGER_STATE_CHANGED.equals(
            action)) {
        mHandler.sendEmptyMessage(MSG_DEVICE_POLICY_MANAGER_STATE_CHANGED);
    /* UNISOC: modify for bug693456 @{ */
    } else if (ACTION_MODEM_CHANGE.equals(intent.getAction())) {
        String state = intent.getStringExtra(MODEM_STAT);
        Log.d(TAG, "modem start state : " + state + "  modemreset:"
                + SystemProperties.get(MODEM_RESET_KEY));
        if (MODEM_ASSERT.equals(state)) {
            if (MODEM_RESET_ENABLE.equals(SystemProperties.get(MODEM_RESET_KEY))) {
                Log.d(TAG, "modem reset open ");
                Message msg = new Message();
                msg.what = MODEM_RESET_MSG;
                for (int i = 0; i < mCallbacks.size(); i++) {
                    KeyguardUpdateMonitorCallback cb = mCallbacks.get(i).get();
                    if (cb != null) {
                        cb.onModemAssert(true);
                    }
                }
                mHandler.removeMessages(MODEM_RESET_MSG);
                mHandler.sendMessageDelayed(msg, MODEM_RESET_TIME_OUT);
            }
        }
    } else if (LocationManager.PROVIDERS_CHANGED_ACTION.equals(action)) {
        mHandler.sendEmptyMessage(MSG_CARRIERTEXT_REFRESH);
    }
    /* @} */
}

};

```

首先在 BroadcastReceiver 的接收开机广播中的 Intent.ACTION_AIRPLANE_MODE_CHANGED 中去开启热点  
然后增加系统开启热点的功能，如下：

```
private void wifiApControl() {
Settings.Global.putInt(mContext.getContentResolver(),
Settings.Global.SOFT_AP_TIMEOUT_ENABLED, 0);
ConnectivityManager connectivityManager = (ConnectivityManager) mContext.getSystemService(Context.CONNECTIVITY_SERVICE);
connectivityManager.startTethering(ConnectivityManager.TETHERING_WIFI,
true, new ConnectivityManager.OnStartTetheringCallback() {
@Override
public void onTetheringFailed() {
super.onTetheringFailed();
Log.d("menghua", "onTetheringFailed");
}
@Override
public void onTetheringStarted() {
super.onTetheringStarted();
Log.d("menghua", "onTetheringStarted");
}
});

```

通过 wifiApControl() 开启热点 等待收到开机广播后连接热点

### 3.2 WifiApConfigStore.java 设置默认热点密码和名称

WLAN 热点默认名称和密码  
可通过 prop 值灵活配置默认名称和密码

```
frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiApConfigStore.java
 private WifiConfiguration getDefaultApConfiguration() {
     WifiConfiguration config = new WifiConfiguration();

   config.SSID = mContext.getResources().getString(

           R.string.wifi_tether_configure_ssid_default) + "_" + getRandomIntForDefaultSsid();

   //config.SSID = mContext.getResources().getString(

   //        R.string.wifi_tether_configure_ssid_default) + "_" + getRandomIntForDefaultSsid();

   config.SSID = "Pnr_AP";

   config.allowedKeyManagement.set(KeyMgmt.WPA2_PSK);
   String randomUUID = UUID.randomUUID().toString();
   //first 12 chars from xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx

   config.preSharedKey = randomUUID.substring(0, 8) + randomUUID.substring(9, 13);

   //config.preSharedKey = randomUUID.substring(0, 8) + randomUUID.substring(9, 13);

   config.preSharedKey ="12345678";

   return config;

}

```
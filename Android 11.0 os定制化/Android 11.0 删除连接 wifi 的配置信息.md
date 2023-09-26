> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124890555)

### 1. 概述

在 11.0 的产品开发中，对于 wifi 的功能定制需求功能也是挺多的，目前对于 wifi 模块有这么个需求，要求在  
提供接口实现删除已连接 wifi 的需求，所以需要了解 wifi 相关的配置情况，然后移除 wifi 即可

### 2. 删除连接 wifi 的配置信息的相关代码

```
frameworks/base/wifi/java/android/net/wifi/IWifiManager.aidl
frameworks/base/wifi/java/android/net/wifi/WifiManager.java
frameworks/opt/net/wifi/service/java/com/android/server/wifi/BaseWifiService.java
frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiServiceImpl.java


```

### 3. 删除连接 wifi 的配置信息的相关功能分析

删除 wifi 配置的实现方法  
在 wifi 的相关 [aidl](https://so.csdn.net/so/search?q=aidl&spm=1001.2101.3001.7020) wifiservice 的管理类等相关 wifi 模块提供需要删除 wifi 信息的接口，然后在  
wifiManager 这个 wifiManager 核心类里面根据 wifi 的 [ssid](https://so.csdn.net/so/search?q=ssid&spm=1001.2101.3001.7020) 来删除 wifi 断开连接

### 3.1IWifiManager.aidl 增加 wifi 接口

IWifiManager.aidl 增加移除 wifi 配置接口

```
--- a/frameworks/base/wifi/java/android/net/wifi/IWifiManager.aidl
+++ b/frameworks/base/wifi/java/android/net/wifi/IWifiManager.aidl
@@ -262,4 +262,7 @@ interface IWifiManager
     boolean registerWifiRssiLinkSpeedAndFrequencyObserver(IWifiRssiLinkSpeedAndFrequencyObserver observer);
     boolean unregisterWifiRssiLinkSpeedAndFrequencyObserver(IWifiRssiLinkSpeedAndFrequencyObserver observer);
     //<-- Add for Wifi Rssi LinkSpeed And Frequency Observer
+       void removeWiFiConfiguration(String ssid);
 }

```

### 3.2 WifiManager.java 里增加移除 wifi 配置相关信息

WifiManager 是 Android 暴露给开发者使用的一个系统服务管理类, 会调用 service 简介地和 framework 层, 驱动层进行函数调用,  
然后驱动层会回调至上层, 以广播的形式实现通知

```
public class WifiManager {
  public WifiManager(@NonNull Context context, @NonNull IWifiManager service,
          @NonNull Looper looper) {
          mContext = context;
          mService = service;
          mLooper = looper;
          mTargetSdkVersion = context.getApplicationInfo().targetSdkVersion;
          updateVerboseLoggingEnabledFromService();
      }
      public List<WifiConfiguration> getConfiguredNetworks() {
          try {
              ParceledListSlice<WifiConfiguration> parceledList =
                      mService.getConfiguredNetworks(mContext.getOpPackageName(),
                              mContext.getAttributionTag());
              if (parceledList == null) {
                  return Collections.emptyList();
              }
              return parceledList.getList();
          } catch (RemoteException e) {
              throw e.rethrowFromSystemServer();
          }
      }
  
      /** @hide */
      @SystemApi
      @RequiresPermission(allOf = {ACCESS_FINE_LOCATION, ACCESS_WIFI_STATE, READ_WIFI_CREDENTIAL})
      public List<WifiConfiguration> getPrivilegedConfiguredNetworks() {
          try {
              ParceledListSlice<WifiConfiguration> parceledList =
                      mService.getPrivilegedConfiguredNetworks(mContext.getOpPackageName(),
                              mContext.getAttributionTag());
              if (parceledList == null) {
                  return Collections.emptyList();
              }
              return parceledList.getList();
          } catch (RemoteException e) {
              throw e.rethrowFromSystemServer();
          }
      }

```

在 forgetNetwork(String ssid) 中根据 ssid 的类型调用 wifimanager 的. removePasspointConfiguration 和 forget 的 api 来删除 wifi 配置

```
 
diff --git a/frameworks/base/wifi/java/android/net/wifi/WifiManager.java b/frameworks/base/wifi/java/android/net/wifi/WifiManager.java
index a55dbde9c0..4bdc586e9b 100755
--- a/frameworks/base/wifi/java/android/net/wifi/WifiManager.java
+++ b/frameworks/base/wifi/java/android/net/wifi/WifiManager.java
@@ -1764,7 +1764,28 @@ public class WifiManager {
             throw e.rethrowFromSystemServer();
         }
     }
+       public void removeWiFiConfiguration(String ssid){
+               try {
+            forgetNetwork(ssid)；
+        } catch (RemoteException e) {
+            throw e.rethrowFromSystemServer();
+        }
+    }

    public void forgetNetwork(String ssid) {
        if (TextUtils.isEmpty(ssid)) {
            return;
        }
        WifiManager mWifiManager = (WifiManager) mContext.getSystemService(Context.WIFI_SERVICE);
         WifiInfo mWifiInfo = mWifiManager.getConnectionInfo();
         WifiConfiguration mWifiConfig = IsExsits(ssid, mWifiManager);
         if (mWifiInfo != null && mWifiInfo.isEphemeral() && ssid.equals(mWifiInfo.getSSID())) {
            mWifiManager.disableEphemeralNetwork(mWifiInfo.getSSID());
         } else if (mWifiConfig != null) {
			if (mWifiConfig.isPasspoint()) {
			   mWifiManager.removePasspointConfiguration(mWifiConfig.FQDN);
			} else {
			   mWifiManager.forget(mWifiConfig.networkId, null);
			}
         } else {
            Log.e(TAG,"failed to forget ssid:" + ssid);
         }
    }
// 判断ssid是否存在
    private WifiConfiguration IsExsits(String SSID, WifiManager mWifiManager) {
        List<WifiConfiguration> existingConfigs = mWifiManager.getConfiguredNetworks();
        for (WifiConfiguration existingConfig : existingConfigs) {
            if (existingConfig.SSID.equals("\"" + SSID + "\"")) {
                return existingConfig;
            }
        }
        return null;
    }

```

### 3.3 BaseWifiService.java 添加 Iwifimanager.aidl 的接口实现

BaseWifiService 的核心类主要承担 wifi 服务端 [binder](https://so.csdn.net/so/search?q=binder&spm=1001.2101.3001.7020) 通讯的角色，所以需要实现 wifi 的接口

```
diff --git a/frameworks/opt/net/wifi/service/java/com/android/server/wifi/BaseWifiService.java b/frameworks/opt/net/wifi/service/java/com/android/server/wifi/BaseWifiService.java
old mode 100644
new mode 100755
index 8ca3753b7b..d826f33a25
--- a/frameworks/base/wifi/java/com/android/server/wifi/BaseWifiService.java
+++ b/frameworks/base/wifi/java/com/android/server/wifi/BaseWifiService.java
@@ -565,4 +565,19 @@ public class BaseWifiService extends IWifiManager.Stub {
         throw new UnsupportedOperationException();
 
     }
     @Override
      public void stopLocalOnlyHotspot() {
          throw new UnsupportedOperationException();
      }
  
      @Override
      public void startWatchLocalOnlyHotspot(ILocalOnlyHotspotCallback callback) {
          throw new UnsupportedOperationException();
      }
  
      @Override
      public void stopWatchLocalOnlyHotspot() {
          throw new UnsupportedOperationException();
      }
  
      @Override
      public int getWifiApEnabledState() {
          throw new UnsupportedOperationException();
      }
  
      @Override
      public WifiConfiguration getWifiApConfiguration() {
          throw new UnsupportedOperationException();
      }
  
      @Override
      public SoftApConfiguration getSoftApConfiguration() {
          throw new UnsupportedOperationException();
      }
  
      @Override
      public boolean setWifiApConfiguration(WifiConfiguration wifiConfig, String packageName) {
          throw new UnsupportedOperationException();
      }
  
      @Override
      public boolean setSoftApConfiguration(SoftApConfiguration softApConfig, String packageName) {
          throw new UnsupportedOperationException();
+       @Override
+       public void removeWiFiConfiguration(String ssid){
+                throw new UnsupportedOperationException();
+               
+       }
 }

```
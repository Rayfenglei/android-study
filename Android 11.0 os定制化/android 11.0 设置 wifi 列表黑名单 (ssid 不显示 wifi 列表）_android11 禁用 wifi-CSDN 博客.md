> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124893880)

### 1. 概述

在 11.0 的定制化开发中，可以要求设置某些 wifi 不出现在 wifi 列表中，然后实现不让连接此 wifi 的功能，就是设置 wifi 黑名单的功能，屏蔽这个 wifi 的连接功能，实现这个功能就需要了解 wifi 管理机制，  
然后在 wifi 列表也去掉这个特定的 wifi，就可以实现这个功能了  
功能分析  
wifiManager 管理所有 wifi 操作，所有的连接搜索显示功能都是在 wifiManager 中通过调用接口实现的，所有接口就加在 wifiManager 就可以了

### 2. 设置 wifi 列表黑名单 (ssid 不显示 wifi 列表）核心代码

```
frameworks/base/wifi/java/android/net/wifi/IWifiManager.aidl
frameworks/base/wifi/java/android/net/wifi/WifiManager.java
frameworks/base/wifi/java/com/android/server/wifi/BaseWifiService.java
frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiServiceImpl.java

```

### 3. 设置 wifi 列表黑名单 (ssid 不显示 wifi 列表）的功能分析和实现

实现步骤：  
1. 在 IWifiManager.[aidl](https://so.csdn.net/so/search?q=aidl&spm=1001.2101.3001.7020) 中增加黑名单列表的相关接口  
2. 在 BaseWifiService.java 实现这个接口  
3. 在 WifiManager.java 中调用 service 设置这些 wifi 黑名单接口  
4. 在 WifiServiceImpl.java 搜索 wifi 列表中去掉这些 wifi 黑名单列表

### 3.1IWifiManager.aidl 中增加黑名单列表的相关接口代码如下:

具体实现如下

```
  --- a/frameworks/base/wifi/java/android/net/wifi/IWifiManager.aidl
+++ b/frameworks/base/wifi/java/android/net/wifi/IWifiManager.aidl
@@ -262,4 +262,7 @@ interface IWifiManager
     boolean registerWifiRssiLinkSpeedAndFrequencyObserver(IWifiRssiLinkSpeedAndFrequencyObserver observer);
     boolean unregisterWifiRssiLinkSpeedAndFrequencyObserver(IWifiRssiLinkSpeedAndFrequencyObserver observer);
     //<-- Add for Wifi Rssi LinkSpeed And Frequency Observer
+       void setWiFiBlackList(in List<String> blackList);
+       List<String> getWiFiBlackList();
 }

```

通过在 IWifiManager.aidl 增加两个接口来实现设置获取黑名单接口

### 3.2 在 WifiManager.java 中调用 service 设置这些 wifi 黑名单接口

通过在 WifiManager.java 提供对外设置和获取 wifi 黑名单的接口，方便管理 wifi 黑名单

```
diff --git a/frameworks/base/wifi/java/android/net/wifi/WifiManager.java b/frameworks/base/wifi/java/android/net/wifi/WifiManager.java
index a55dbde9c0..4bdc586e9b 100755
--- a/frameworks/base/wifi/java/android/net/wifi/WifiManager.java
+++ b/frameworks/base/wifi/java/android/net/wifi/WifiManager.java
@@ -1764,7 +1764,28 @@ public class WifiManager {
            throw e.rethrowFromSystemServer();
        }
    }
+       public void setWiFiBlackList(List<String> blackList){
+               try {
+            mService.setWiFiBlackList(blackList); 
+        } catch (RemoteException e) {
+            throw e.rethrowFromSystemServer();
+        }
+    }

+    public List<String> getWiFiBlackList(){
+               try {
+            return mService.getWiFiBlackList();
+        } catch (RemoteException e) {
+            throw e.rethrowFromSystemServer();
+        }
+    }

```

### 3.3 在 BaseWifiService.java 实现这个接口

BaseWifiService 是 IWifiManager.aidl 的实现类作为 [binder](https://so.csdn.net/so/search?q=binder&spm=1001.2101.3001.7020) 通讯的服务端，对增加的黑名单接口需要在这里实现对应的方法。

```
diff --git a/frameworks/base/wifi/java/com/android/server/wifi/BaseWifiService.java b/frameworks/base/wifi/java/com/android/server/wifi/BaseWifiService.j
ava
old mode 100644
new mode 100755
index 8ca3753b7b..d826f33a25
--- a/frameworks/base/wifi/java/com/android/server/wifi/BaseWifiService.java
+++ b/frameworks/base/wifi/java/com/android/server/wifi/BaseWifiService.java
@@ -565,4 +565,19 @@ public class BaseWifiService extends IWifiManager.Stub {
        throw new UnsupportedOperationException();

    }
+       @Override
+       public void setWiFiBlackList( List<String> blackList){
+                throw new UnsupportedOperationException();
+               
+       };
+       @Override
+       public List<String> getWiFiBlackList(){
+                throw new UnsupportedOperationException();
+               
+       };
}


```

3.4 在 WifiServiceImpl.java 搜索 wifi 列表中去掉这些 wifi 黑名单列表  
最关键的实现步骤，其实在 WifiService 服务中，最关键的核心类就是在 WifiServiceImpl 中，这里面大部分工作都是在这里进行的，所以在搜索 wifi 列表的时候，可以去掉黑名单的列表  
接下来看下

```
```bash
public class WifiServiceImpl extends BaseWifiService {
   public WifiServiceImpl(Context context, WifiInjector wifiInjector, AsyncChannel asyncChannel) {
         mContext = context;
         mWifiInjector = wifiInjector;
         mClock = wifiInjector.getClock();
 
         mFacade = mWifiInjector.getFrameworkFacade();
         mWifiMetrics = mWifiInjector.getWifiMetrics();
         mWifiTrafficPoller = mWifiInjector.getWifiTrafficPoller();
         mUserManager = mWifiInjector.getUserManager();
         mCountryCode = mWifiInjector.getWifiCountryCode();
         mClientModeImpl = mWifiInjector.getClientModeImpl();
         mActiveModeWarden = mWifiInjector.getActiveModeWarden();
         mScanRequestProxy = mWifiInjector.getScanRequestProxy();
         mSettingsStore = mWifiInjector.getWifiSettingsStore();
         mPowerManager = mContext.getSystemService(PowerManager.class);
         mAppOps = (AppOpsManager) mContext.getSystemService(Context.APP_OPS_SERVICE);
         mWifiLockManager = mWifiInjector.getWifiLockManager();
         mWifiMulticastLockManager = mWifiInjector.getWifiMulticastLockManager();
         mClientModeImplHandler = new ClientModeImplHandler(TAG,
                 mWifiInjector.getAsyncChannelHandlerThread().getLooper(), asyncChannel);
         mWifiBackupRestore = mWifiInjector.getWifiBackupRestore();
         mSoftApBackupRestore = mWifiInjector.getSoftApBackupRestore();
         mWifiApConfigStore = mWifiInjector.getWifiApConfigStore();
         mWifiPermissionsUtil = mWifiInjector.getWifiPermissionsUtil();
         mLog = mWifiInjector.makeLog(TAG);
         mFrameworkFacade = wifiInjector.getFrameworkFacade();
         mTetheredSoftApTracker = new TetheredSoftApTracker();
         mActiveModeWarden.registerSoftApCallback(mTetheredSoftApTracker);
         mLohsSoftApTracker = new LohsSoftApTracker();
         mActiveModeWarden.registerLohsCallback(mLohsSoftApTracker);
         mWifiNetworkSuggestionsManager = mWifiInjector.getWifiNetworkSuggestionsManager();
         mDppManager = mWifiInjector.getDppManager();
         mWifiThreadRunner = mWifiInjector.getWifiThreadRunner();
         mWifiConfigManager = mWifiInjector.getWifiConfigManager();
         mPasspointManager = mWifiInjector.getPasspointManager();
         mWifiScoreCard = mWifiInjector.getWifiScoreCard();
         mMemoryStoreImpl = new MemoryStoreImpl(mContext, mWifiInjector,
                 mWifiScoreCard,  mWifiInjector.getWifiHealthMonitor());
     }
     /**
      * Return the results of the most recent access point scan, in the form of
      * a list of {@link ScanResult} objects.
      * @return the list of results
      */
     @Override
     public List<ScanResult> getScanResults(String callingPackage, String callingFeatureId) {
         enforceAccessPermission();
         int uid = Binder.getCallingUid();
         long ident = Binder.clearCallingIdentity();
         if (mVerboseLoggingEnabled) {
             mLog.info("getScanResults uid=%").c(uid).flush();
         }
         try {
             mWifiPermissionsUtil.enforceCanAccessScanResults(callingPackage, callingFeatureId,
                     uid, null);
             List<ScanResult> scanResults = mWifiThreadRunner.call(
                     mScanRequestProxy::getScanResults, Collections.emptyList());
             return scanResults;
         } catch (SecurityException e) {
             Log.e(TAG, "Permission violation - getScanResults not allowed for uid="
                     + uid + ", package + e);
             return new ArrayList<>();
         } finally {
             Binder.restoreCallingIdentity(ident);
         }
     }

```

在代码中可以看出 WifiServiceImpl 在构造方法中，初始化了关于 wifi 的相关功能，而在搜索列表中，在 List getScanResults 中，实现搜索 wifi 列表功能，所以具体实现就在这里

```
diff --git a/frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiServiceImpl.java b/frameworks/opt/net/wifi/service/java/com/android/server/
wifi/WifiServiceImpl.java
old mode 100644
new mode 100755
index c429630db2..561e03a49b
--- a/frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiServiceImpl.java
+++ b/frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiServiceImpl.java
@@ -2423,6 +2423,16 @@ public class WifiServiceImpl extends BaseWifiService {
List<ScanResult> scanResults = mWifiThreadRunner.call(
                      mScanRequestProxy::getScanResults, Collections.emptyList());
+                       if(this.ssid_blackList!=null && this.ssid_blackList.size()!=0){
+                                    List<ScanResult> result1 =new ArrayList<>();
+                for (ScanResult resultbean : scanResults){
+                    if(this.ssid_blackList.contains(resultbean.SSID)){
+                        continue;
+                    }
+                    result1.add(resultbean);
+                           }
+                               return  result1;
+                       }
             return scanResults;
         } catch (SecurityException e) {
             Slog.e(TAG, "Permission violation - getScanResults not allowed for uid="
@@ -2432,7 +2442,27 @@ public class WifiServiceImpl extends BaseWifiService {
             Binder.restoreCallingIdentity(ident);
         }
     }
-
+       private List<String> ssid_blackList;
+       @Override
+       public void setWiFiBlackList(List<String> blackList){
+               this.ssid_blackList=blackList;
+               
+       }
+       @Override
+       public List<String> getWiFiBlackList(){
+               return this.ssid_blackList;
+               
+       }

```
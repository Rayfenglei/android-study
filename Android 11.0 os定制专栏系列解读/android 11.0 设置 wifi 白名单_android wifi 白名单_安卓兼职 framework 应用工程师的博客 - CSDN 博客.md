> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124676852)

### 1. 概述

在 11.0 产品定制化开发中，产品需求要求对 wifi 的管理比较严格，所以设置 wifi 白名单和 wifi 黑名单这样的需求也是常见的，  
需求要求设置 wifi 白名单功能，就是在这个白名单的 wifi ssid 可以显示出来，可以连接 wifi 其他的就不可以连接  
那么就要在搜索列表中过滤只显示白名单即可

### 2. 设置 wifi 白名单的核心类

```
frameworks/base/wifi/java/com/android/server/wifi/BaseWifiService.java
frameworks/base/wifi/java/android/net/wifi/IWifiManager.aidl
frameworks/base/wifi/java/android/net/wifi/WifiManager.java
frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiServiceImpl.java

```

### 3. 设置 wifi 白名单的核心功能实现和分析

功能分析  
WifiService 在[构造方法](https://so.csdn.net/so/search?q=%E6%9E%84%E9%80%A0%E6%96%B9%E6%B3%95&spm=1001.2101.3001.7020)中新建了一个 WifiServiceImpl 实例，它是 Wifi 管理服务真正的实现者，在前面的 WifiService  
启动过程中调用了 WifiService 的 onStart 方法；在 onStart 方法中发布了 Wifi 服务, WifiServiceImpl 才是真正的 WifiService  
实现了 WifiService 的很多具体功能  
所以解决方案就是在 WifiServiceImpl 中的 getScanResults（）中返回白名单里的 ssid

```
public class WifiServiceImpl extends BaseWifiService {
      private static final String TAG = "WifiService";
      private static final int APP_INFO_FLAGS_SYSTEM_APP =
              ApplicationInfo.FLAG_SYSTEM | ApplicationInfo.FLAG_UPDATED_SYSTEM_APP;
      private static final boolean VDBG = false;
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

在构造方法中进行了大量的实例化对象来管理 WIFI 的各个功能：  
WifiStateMachineHandler 用于发送和处理 wifi 状态机相关的消息  
mTrafficPoller 用来查询流量统计信息比通知主要是给客户端调用  
mWifiStateMachine 它是一个 Wifi 状态机，它定义了 wifi 的很多状态，通过消息驱动状态的转变。  
mPowerManager 它主要用于 wifi 的电源管理。

具体实现步骤如下:

### 1.IWifiManager.aidi 增加白名单接口

```
--- a/frameworks/base/wifi/java/android/net/wifi/IWifiManager.aidl
+++ b/frameworks/base/wifi/java/android/net/wifi/IWifiManager.aidl
@@ -262,4 +262,7 @@ interface IWifiManager
     boolean registerWifiRssiLinkSpeedAndFrequencyObserver(IWifiRssiLinkSpeedAndFrequencyObserver observer);
     boolean unregisterWifiRssiLinkSpeedAndFrequencyObserver(IWifiRssiLinkSpeedAndFrequencyObserver observer);
     //<-- Add for Wifi Rssi LinkSpeed And Frequency Observer
+       void setWiFiWhiteList(in List<String> blackList);
+       List<String> getWiFiWhiteList();
 }

```

### 2.WifiManager.java 增加白名单接口

```
diff --git a/frameworks/base/wifi/java/android/net/wifi/WifiManager.java b/frameworks/base/wifi/java/android/net/wifi/WifiManager.java
index a55dbde9c0..4bdc586e9b 100755
--- a/frameworks/base/wifi/java/android/net/wifi/WifiManager.java
+++ b/frameworks/base/wifi/java/android/net/wifi/WifiManager.java
@@ -1764,7 +1764,28 @@ public class WifiManager {
throw e.rethrowFromSystemServer();
}
}
  public void setWiFiWhiteList(List<String> blackList){

          try {

       mService.setWiFiWhiteList(blackList); 

   } catch (RemoteException e) {

       throw e.rethrowFromSystemServer();

   }

}
public List<String> getWiFiWhiteList(){
          try {

       return mService.getWiFiWhiteList();

   } catch (RemoteException e) {

       throw e.rethrowFromSystemServer();

   }

}

```

### 3.BaseWifiService.java 增加接口

```
diff --git a/frameworks/base/wifi/java/com/android/server/wifi/BaseWifiService.java b/frameworks/base/wifi/java/com/android/server/wifi/BaseWifiService.java
old mode 100644
new mode 100755
index 8ca3753b7b..d826f33a25
--- a/frameworks/base/wifi/java/com/android/server/wifi/BaseWifiService.java
+++ b/frameworks/base/wifi/java/com/android/server/wifi/BaseWifiService.java
@@ -565,4 +565,19 @@ public class BaseWifiService extends IWifiManager.Stub {
         throw new UnsupportedOperationException();
 
     }
+       @Override
+       public void setWiFiWhiteList( List<String> blackList){
+                throw new UnsupportedOperationException();
+               
+       };
+       @Override
+       public List<String> getWiFiWhiteList(){
+                throw new UnsupportedOperationException();
+               
+       };
 }

```

### 4.WifiServiceImpl.java 适配白名单

```
diff --git a/frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiServiceImpl.java b/frameworks/opt/net/wifi/service/java/com/android/server/
wifi/WifiServiceImpl.java
old mode 100644
new mode 100755
index c429630db2..561e03a49b
--- a/frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiServiceImpl.java
+++ b/frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiServiceImpl.java
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
              +                       if(this.ssid_whiteList!=null && this.ssid_whiteList.size()!=0){
+                                    List<ScanResult> result1 =new ArrayList<>();
+                for (ScanResult resultbean : scanResults){
+                    if(!this.ssid_whiteList.contains(resultbean.SSID)){
+                        continue;
+                    }
+                    result1.add(resultbean);
+                           }
+                               return  result1;
+                       }     
             return scanResults;
         } catch (SecurityException e) {
              Log.e(TAG, "Permission violation - getScanResults not allowed for uid="
                      + uid + ", package + e);
              return new ArrayList<>();
          } finally {
              Binder.restoreCallingIdentity(ident);
          }
     }


+       private List<String> ssid_whiteList;
+       @Override
+       public void setWiFiWhiteList(List<String> whiteList){
+               this.ssid_whiteList=whiteList;
+               
+       }
+       @Override
+       public List<String> getWiFiwhiteList(){
+               return this.ssid_whiteList;
+               
+       }

```

最主要的功能是在 WifiServiceImpl.java 中的 getScanResults(String callingPackage, String callingFeatureId)  
在获取 wifi 列表时，只有是在 wifi 白名单的 ssid 才显示出来，这样就可以做到只能添加白名单的 ssid 到 wifi  
列表了
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126842374)

**目录**

 [1. 概述](#%C2%A01.%E6%A6%82%E8%BF%B0)

[2. 修改 wifi 信号强度的核心代码](#t1)

[3. 修改 wifi 信号强度的功能分析以及实现功能](#t2)

 [3.1 app 中 wifi 信号强度的计算](#t3)

 [3.2 WifiManager.java 中关于信号强度代码](#t4)

[3.2 AccessPoint.java 关于 wifi 信号强度的分析](#t5)

 1. 概述
------

  在进行产品开发中，各种各样的需求，对纯 wifi 产品而言，对于 wifi 要求也是越来越高，因此有客户要求对 wifi 信号强度做定制，修改信号强度来增强显示 wifi 信号，所以要对 wifi 显示信号强度的相关代码做修改

2. 修改 wifi 信号强度的核心代码
--------------------

```
  frameworks/base/wifi/java/android/net/wifi/WifiManager.java
  frameworks/base/packages/SettingsLib/src/com/android/settingslib/wifi/AccessPoint.java
```

3. 修改 wifi 信号强度的功能分析以及实现功能
--------------------------

###   3.1 app 中 wifi 信号强度的计算

```
      WifiManager mwifiManager=(WifiManager) getSystemService(WIFI_SERVICE);// 获取wifiManager wifi管理类
      WifiInfo mwifiInfo=mwifiManager.getConnectionInfo();//当前wifi连接信息
     List mscanResults=mwifiManager.getScanResults();//搜索到的设备列表
    
for (ScanResult scanResult : mscanResults) {//遍历搜索到的wifi
 
tv.append(“\n设备名：”+scanResult.SSID
 
+” 信号强度：”+scanResult.level+”/n :”+wifiManager.calculateSignalLevel(scanResult.level,4));
 
}
```

###  3.2 WifiManager.java 中关于信号强度代码

```
  public class WifiManager {
     @Deprecated
      public static final int WIFI_MODE_FULL = 1;
  
      /**
       * In this Wi-Fi lock mode, Wi-Fi will be kept active,
       * but the only operation that will be supported is initiation of
       * scans, and the subsequent reporting of scan results. No attempts
       * will be made to automatically connect to remembered access points,
       * nor will periodic scans be automatically performed looking for
       * remembered access points. Scans must be explicitly requested by
       * an application in this mode.
       *
       * @deprecated This API is non-functional and will have no impact.
       */
      @Deprecated
      public static final int WIFI_MODE_SCAN_ONLY = 2;
  
      /**
       * In this Wi-Fi lock mode, Wi-Fi will not go to power save.
       * This results in operating with low packet latency.
       * The lock is only active when the device is connected to an access point.
       * The lock is active even when the device screen is off or the acquiring application is
       * running in the background.
       * This mode will consume more power and hence should be used only
       * when there is a need for this tradeoff.
       * <p>
       * An example use case is when a voice connection needs to be
       * kept active even after the device screen goes off.
       * Holding a {@link #WIFI_MODE_FULL_HIGH_PERF} lock for the
       * duration of the voice call may improve the call quality.
       * <p>
       * When there is no support from the hardware, the {@link #WIFI_MODE_FULL_HIGH_PERF}
       * lock will have no impact.
       */
      public static final int WIFI_MODE_FULL_HIGH_PERF = 3;
  
      /**
       * In this Wi-Fi lock mode, Wi-Fi will operate with a priority to achieve low latency.
       * {@link #WIFI_MODE_FULL_LOW_LATENCY} lock has the following limitations:
       * <ol>
       * <li>The lock is only active when the device is connected to an access point.</li>
       * <li>The lock is only active when the screen is on.</li>
       * <li>The lock is only active when the acquiring app is running in the foreground.</li>
       * </ol>
       * Low latency mode optimizes for reduced packet latency,
       * and as a result other performance measures may suffer when there are trade-offs to make:
       * <ol>
       * <li>Battery life may be reduced.</li>
       * <li>Throughput may be reduced.</li>
       * <li>Frequency of Wi-Fi scanning may be reduced. This may result in: </li>
       * <ul>
       * <li>The device may not roam or switch to the AP with highest signal quality.</li>
       * <li>Location accuracy may be reduced.</li>
       * </ul>
       * </ol>
       * <p>
       * Example use cases are real time gaming or virtual reality applications where
       * low latency is a key factor for user experience.
       * <p>
       * Note: For an app which acquires both {@link #WIFI_MODE_FULL_LOW_LATENCY} and
       * {@link #WIFI_MODE_FULL_HIGH_PERF} locks, {@link #WIFI_MODE_FULL_LOW_LATENCY}
       * lock will be effective when app is running in foreground and screen is on,
       * while the {@link #WIFI_MODE_FULL_HIGH_PERF} lock will take effect otherwise.
       */
      public static final int WIFI_MODE_FULL_LOW_LATENCY = 4;
  
  
      /** Anything worse than or equal to this will show 0 bars. */
      @UnsupportedAppUsage
      private static final int MIN_RSSI = -100; //最小信号强度
  
      /** Anything better than or equal to this will show the max bars. */
      @UnsupportedAppUsage
      private static final int MAX_RSSI = -55;  //最大信号强度
  
      /**
       * Number of RSSI levels used in the framework to initiate {@link #RSSI_CHANGED_ACTION}
       * broadcast, where each level corresponds to a range of RSSI values.
       * The {@link #RSSI_CHANGED_ACTION} broadcast will only fire if the RSSI
       * change is significant enough to change the RSSI signal level.
       * @hide
       */
      @UnsupportedAppUsage
      public static final int RSSI_LEVELS = 5;  //wifi信号格数
  
      //TODO (b/146346676): This needs to be removed, not used in the code.
      /**
       * Auto settings in the driver. The driver could choose to operate on both
       * 2.4 GHz and 5 GHz or make a dynamic decision on selecting the band.
       * @hide
       */
      @UnsupportedAppUsage
      public static final int WIFI_FREQUENCY_BAND_AUTO = 0;
  
      /**
       * Operation on 5 GHz alone
       * @hide
       */
      @UnsupportedAppUsage
      public static final int WIFI_FREQUENCY_BAND_5GHZ = 1;
  
      /**
       * Operation on 2.4 GHz alone
       * @hide
       */
      @UnsupportedAppUsage
      public static final int WIFI_FREQUENCY_BAND_2GHZ = 2;
 
      // 根据ssid的当前信号 计算信号强度  
     // rssi当前ssid的信号 numLevels表示wifi信号分为几个格数
      public static int calculateSignalLevel(int rssi, int numLevels) {
          if (rssi <= MIN_RSSI) {
              return 0; // 最小信号强度
          } else if (rssi >= MAX_RSSI) {
              return numLevels - 1;// 最大信号强度
          } else {
              // 计算当前信号强度
              float inputRange = (MAX_RSSI - MIN_RSSI);
              float outputRange = (numLevels - 1);
              return (int)((float)(rssi - MIN_RSSI) * outputRange / inputRange);
          }
      }
  
      /**
       * Given a raw RSSI, return the RSSI signal quality rating using the system default RSSI
       * quality rating thresholds.
       * @param rssi a raw RSSI value, in dBm, usually between -55 and -90
       * @return the RSSI signal quality rating, in the range
       * [0, {@link #getMaxSignalLevel()}], where 0 is the lowest (worst signal) RSSI
       * rating and {@link #getMaxSignalLevel()} is the highest (best signal) RSSI rating.
       */
      @IntRange(from = 0)
//根据rssi获取wifi信号强度
      public int calculateSignalLevel(int rssi) {
          try {
              return mService.calculateSignalLevel(rssi);
          } catch (RemoteException e) {
              throw e.rethrowFromSystemServer();
          }
      }
// 根据以上所述是根据MAX_RSSI 和 MIN_RSSI 以及numLevels这三个值来计算信号强度的 
所以想修改信号强度 可以修改MAX_RSSI 的值 也可以增加rssi 的值
```

### 3.2 AccessPoint.java 关于 wifi 信号强度的分析

```
   //获取wificonfiguration
    public WifiConfiguration getConfig() {
          return mConfig;
      }
  
      public String getPasspointFqdn() {
          return mFqdn;
      }
  //清除config
      public void clearConfig() {
          mConfig = null;
          networkId = WifiConfiguration.INVALID_NETWORK_ID;
      }
  //获取wifiINfo
      public WifiInfo getInfo() {
          return mInfo;
      }
  
      /**
       * Returns the number of levels to show for a Wifi icon, from 0 to
       * {@link WifiManager#getMaxSignalLevel()}.
       *
       * <p>Use {@link #isReachable()} to determine if an AccessPoint is in range, as this method will
       * always return at least 0.
       */
      public int getLevel() {
      //获取wifi信号强度                    
          return getWifiManager().calculateSignalLevel(mRssi);
      }
修改方法如下
     // rssi当前ssid的信号 numLevels表示wifi信号分为几个格数
      public static int calculateSignalLevel(int rssi) {
      try {
      //add core start
          + rssi+=10; // 增加rssi的值 也同样增强wifi信号强度
          if (rssi <= MIN_RSSI) {
              return 0; // 最小信号强度
          } else if (rssi >= MAX_RSSI) {
              return RSSL_LEVELS - 1;// 最大信号强度
          } else {
              // 计算当前信号强度
              float inputRange = (MAX_RSSI - MIN_RSSI);
              float outputRange = (RSSL_LEVELS - 1);
              return (int)((float)(rssi - MIN_RSSI) * outputRange / inputRange);
          }
          //add core end
              return mService.calculateSignalLevel(rssi);
          } catch (RemoteException e) {
              throw e.rethrowFromSystemServer();
          }
      }
```
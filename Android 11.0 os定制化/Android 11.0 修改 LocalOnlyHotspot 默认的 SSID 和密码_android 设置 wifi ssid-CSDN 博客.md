> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126590206)

今年的中秋又要到啦，作为程序员发表优秀博文来作为送给需要的人作为中秋礼物

1. 概述
-----

在设备产品开发中，需要开启本地热点 LocalOnlyHotspot，由于默认 ssid 和密码都是随机生成的，所以根据需求要求修改为默认的 ssid 和密码，所以这就要查询  
相关的设置流程，然后修改为默认的 ssid 和密码.  
  app 中获取 LocalOnlyHotspot 的默认 ssid 和密码的代码为:

```
mWifiManager = (WifiManager)getSystemService(Context.WIFI_SERVICE);
        mWifiManager.startLocalOnlyHotspot(new WifiManager.LocalOnlyHotspotCallback() {
                                               @Override
                                               public void onStarted(WifiManager.LocalOnlyHotspotReservation reservation) {
                                                   super.onStarted(reservation);
                                                   String ssid = reservation.getWifiConfiguration().SSID;
                                                   String pwd = reservation.getWifiConfiguration().preSharedKey;
                                                   Log.e("SetLocalOnlyHotSpotController", "ssid and pwd is" + ssid + "and:" + pwd);
                                               }
 
                                               @Override
                                               public void onStopped() {
                                                   super.onStopped();
                                               }
 
                                               @Override
                                               public void onFailed(int reason) {
                                                   super.onFailed(reason);
                                               }
                                           }, new Handler());
```

通过上面的案例是通过 mWifiManager.startLocalOnlyHotspot
---------------------------------------------

调用 wifimanager 的方法来实现获取当前 Hotspot 的 ssid 和密码
--------------------------------------------

然后分析解决怎么修改默认的 ssid 和密码的值
------------------------

  
2. 修改 LocalOnlyHotspot 默认的 SSID 和密码的核心代码
-------------------------------------------

```
  frameworks/base/wifi/java/android/net/wifi/WifiManager.java
  frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiServiceImpl.java
  frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiApConfigStore.java
```

3. 修改 LocalOnlyHotspot 默认的 SSID 和密码的代码分析和功能实现
---------------------------------------------

###   3.1 WifiManager 关于本地热点的相关方法

```
public class WifiManager {
  
      private static final String TAG = "WifiManager";
	  @UnsupportedAppUsage
      IWifiManager mService;
    @SystemApi
      @RequiresPermission(anyOf = {
              android.Manifest.permission.NETWORK_SETTINGS,
              android.Manifest.permission.NETWORK_SETUP_WIZARD})
      public void startLocalOnlyHotspot(@NonNull SoftApConfiguration config,
              @Nullable Executor executor,
              @Nullable LocalOnlyHotspotCallback callback) {
          Objects.requireNonNull(config);
          startLocalOnlyHotspotInternal(config, executor, callback);
      }
  
      /**
       * Common implementation of both configurable and non-configurable LOHS.
       *
       * @param config App-specified configuration, or null. When present, additional privileges are
       *               required, and the hotspot cannot be shared with other clients.
       * @param executor Executor to run callback methods on, or null to use the main thread.
       * @param callback Callback object for updates about hotspot status, or null for no updates.
       */
      private void startLocalOnlyHotspotInternal(
              @Nullable SoftApConfiguration config,
              @Nullable Executor executor,
              @Nullable LocalOnlyHotspotCallback callback) {
          if (executor == null) {
              executor = mContext.getMainExecutor();
          }
          synchronized (mLock) {
              LocalOnlyHotspotCallbackProxy proxy =
                      new LocalOnlyHotspotCallbackProxy(this, executor, callback);
              try {
                  String packageName = mContext.getOpPackageName();
                  String featureId = mContext.getAttributionTag();
                  int returnCode = mService.startLocalOnlyHotspot(proxy, packageName, featureId,
                          config);
                  if (returnCode != LocalOnlyHotspotCallback.REQUEST_REGISTERED) {
                      // Send message to the proxy to make sure we call back on the correct thread
                      proxy.onHotspotFailed(returnCode);
                      return;
                  }
                  mLOHSCallbackProxy = proxy;
              } catch (RemoteException e) {
                  throw e.rethrowFromSystemServer();
              }
          }
      }
....
}
```

### 通过查看 wifimanage 的源码发现最终是通过 wifiserviceimpl 的 startLocalOnlyHotspot 的方法来具体开启本地热点

接下来看下 WifiServiceImpl 的相关方法

### 3.2 WifiServiceImpl.java 关于本地热点的相关调用方法

```
    public class WifiServiceImpl extends BaseWifiService {
     private static final String TAG = "WifiService";
     private static final int APP_INFO_FLAGS_SYSTEM_APP =
             ApplicationInfo.FLAG_SYSTEM | ApplicationInfo.FLAG_UPDATED_SYSTEM_APP;
     private static final boolean VDBG = false;
 
     /** Max wait time for posting blocking runnables */
     private static final int RUN_WITH_SCISSORS_TIMEOUT_MILLIS = 4000;
 
     private final ClientModeImpl mClientModeImpl;
     private final ActiveModeWarden mActiveModeWarden;
     private final ScanRequestProxy mScanRequestProxy;
 
     private final Context mContext;
     private final FrameworkFacade mFacade;
     private final Clock mClock;
 
     private final PowerManager mPowerManager;
     private final AppOpsManager mAppOps;
     private final UserManager mUserManager;
     private final WifiCountryCode mCountryCode;
 
     /** Polls traffic stats and notifies clients */
     private final WifiTrafficPoller mWifiTrafficPoller;
     /** Tracks the persisted states for wi-fi & airplane mode */
     private final WifiSettingsStore mSettingsStore;
     /** Logs connection events and some general router and scan stats */
     private final WifiMetrics mWifiMetrics;
 
     private final WifiInjector mWifiInjector;
     /** Backup/Restore Module */
     private final WifiBackupRestore mWifiBackupRestore;
     private final SoftApBackupRestore mSoftApBackupRestore;
     private final WifiNetworkSuggestionsManager mWifiNetworkSuggestionsManager;
     private final WifiConfigManager mWifiConfigManager;
     private final PasspointManager mPasspointManager;
     private final WifiLog mLog;
 /**
       * Method to start LocalOnlyHotspot.  In this method, permissions, settings and modes are
       * checked to verify that we can enter softapmode.  This method returns
       * {@link LocalOnlyHotspotCallback#REQUEST_REGISTERED} if we will attempt to start, otherwise,
       * possible startup erros may include tethering being disallowed failure reason {@link
       * LocalOnlyHotspotCallback#ERROR_TETHERING_DISALLOWED} or an incompatible mode failure reason
       * {@link LocalOnlyHotspotCallback#ERROR_INCOMPATIBLE_MODE}.
       *
       * see {@link WifiManager#startLocalOnlyHotspot(LocalOnlyHotspotCallback)}
       *
       * @param callback Callback to communicate with WifiManager and allow cleanup if the app dies.
       * @param packageName String name of the calling package.
       * @param featureId The feature in the package
       * @param customConfig Custom configuration to be applied to the hotspot, or null for a shared
       *                     hotspot with framework-generated config.
       *
       * @return int return code for attempt to start LocalOnlyHotspot.
       *
       * @throws SecurityException if the caller does not have permission to start a Local Only
       * Hotspot.
       * @throws IllegalStateException if the caller attempts to start the LocalOnlyHotspot while they
       * have an outstanding request.
       */
      @Override
      public int startLocalOnlyHotspot(ILocalOnlyHotspotCallback callback, String packageName,
              String featureId, SoftApConfiguration customConfig) {
          // first check if the caller has permission to start a local only hotspot
          // need to check for WIFI_STATE_CHANGE and location permission
          final int uid = Binder.getCallingUid();
          final int pid = Binder.getCallingPid();
  
          mLog.info("start uid=% pid=%").c(uid).c(pid).flush();
  
          // Permission requirements are different with/without custom config.
          if (customConfig == null) {
              if (enforceChangePermission(packageName) != MODE_ALLOWED) {
                  return LocalOnlyHotspotCallback.ERROR_GENERIC;
              }
              enforceLocationPermission(packageName, featureId, uid);
              long ident = Binder.clearCallingIdentity();
              try {
                  // also need to verify that Locations services are enabled.
                  if (!mWifiPermissionsUtil.isLocationModeEnabled()) {
                      throw new SecurityException("Location mode is not enabled.");
                  }
              } finally {
                  Binder.restoreCallingIdentity(ident);
              }
          } else {
              if (!isSettingsOrSuw(Binder.getCallingPid(), Binder.getCallingUid())) {
                  throw new SecurityException(TAG + ": Permission denied");
              }
          }
  
          // verify that tethering is not disabled
          if (mUserManager.hasUserRestrictionForUser(
                  UserManager.DISALLOW_CONFIG_TETHERING, UserHandle.getUserHandleForUid(uid))) {
              return LocalOnlyHotspotCallback.ERROR_TETHERING_DISALLOWED;
          }
  
          // the app should be in the foreground
          long ident = Binder.clearCallingIdentity();
          try {
              // also need to verify that Locations services are enabled.
              if (!mFrameworkFacade.isAppForeground(mContext, uid)) {
                  return LocalOnlyHotspotCallback.ERROR_INCOMPATIBLE_MODE;
              }
          } finally {
              Binder.restoreCallingIdentity(ident);
          }
  
          // check if we are currently tethering
          if (!mActiveModeWarden.canRequestMoreSoftApManagers()
                  && mTetheredSoftApTracker.getState() == WIFI_AP_STATE_ENABLED) {
              // Tethering is enabled, cannot start LocalOnlyHotspot
              mLog.info("Cannot start localOnlyHotspot when WiFi Tethering is active.")
                      .flush();
              return LocalOnlyHotspotCallback.ERROR_INCOMPATIBLE_MODE;
          }
  
          // now create the new LOHS request info object
          LocalOnlyHotspotRequestInfo request = new LocalOnlyHotspotRequestInfo(callback,
                  new LocalOnlyRequestorCallback(), customConfig);
  
          return mLohsSoftApTracker.start(pid, request);
      }
....
}
 
 
 /**
       * Implements LOHS behavior on top of the existing SoftAp API.
       */
      private final class LohsSoftApTracker implements WifiManager.SoftApCallback {
          @GuardedBy("mLocalOnlyHotspotRequests")
          private final HashMap<Integer, LocalOnlyHotspotRequestInfo>
                  mLocalOnlyHotspotRequests = new HashMap<>();
  
          /** Currently-active config, to be sent to shared clients registering later. */
          @GuardedBy("mLocalOnlyHotspotRequests")
          private SoftApModeConfiguration mActiveConfig = null;
  
          /**
           * Whether we are currently operating in exclusive mode (i.e. whether a custom config is
           * active).
           */
          @GuardedBy("mLocalOnlyHotspotRequests")
          private boolean mIsExclusive = false;
  
          @GuardedBy("mLocalOnlyHotspotRequests")
          private String mLohsInterfaceName;
  
          /**
           * State of local-only hotspot
           * One of:  {@link WifiManager#WIFI_AP_STATE_DISABLED},
           *          {@link WifiManager#WIFI_AP_STATE_DISABLING},
           *          {@link WifiManager#WIFI_AP_STATE_ENABLED},
           *          {@link WifiManager#WIFI_AP_STATE_ENABLING},
           *          {@link WifiManager#WIFI_AP_STATE_FAILED}
           */
          @GuardedBy("mLocalOnlyHotspotRequests")
          private int mLohsState = WIFI_AP_STATE_DISABLED;
  
          @GuardedBy("mLocalOnlyHotspotRequests")
          private int mLohsInterfaceMode = WifiManager.IFACE_IP_MODE_UNSPECIFIED;
  
          private SoftApCapability mLohsSoftApCapability = null;
  
          public SoftApCapability getSoftApCapability() {
              if (mLohsSoftApCapability == null) {
                  mLohsSoftApCapability =  ApConfigUtil.updateCapabilityFromResource(mContext);
              }
              return mLohsSoftApCapability;
          }
  
          public void updateInterfaceIpState(String ifaceName, int mode) {
              // update interface IP state related to local-only hotspot
              synchronized (mLocalOnlyHotspotRequests) {
                  Log.d(TAG, "updateInterfaceIpState: iface + mode
                          + " previous LOHS mode= " + mLohsInterfaceMode);
  
                  switch (mode) {
                      case WifiManager.IFACE_IP_MODE_LOCAL_ONLY:
                          // first make sure we have registered requests.
                          if (mLocalOnlyHotspotRequests.isEmpty()) {
                              // we don't have requests...  stop the hotspot
                              Log.wtf(TAG, "Starting LOHS without any requests?");
                              stopSoftApInternal(WifiManager.IFACE_IP_MODE_LOCAL_ONLY);
                              return;
                          }
                          // LOHS is ready to go!  Call our registered requestors!
                          mLohsInterfaceName = ifaceName;
                          mLohsInterfaceMode = mode;
                          sendHotspotStartedMessageToAllLOHSRequestInfoEntriesLocked();
                          break;
                      case WifiManager.IFACE_IP_MODE_TETHERED:
                          if (mLohsInterfaceName != null
                                  && mLohsInterfaceName.equals(ifaceName)) {
                              /* This shouldn't happen except in a race, but if it does, tear down
                               * the LOHS and let tethering win.
                               *
                               * If concurrent SAPs are allowed, the interface names will differ,
                               * so we don't have to check the config here.
                               */
                              Log.e(TAG, "Unexpected IP mode change on " + ifaceName);
                              mLohsInterfaceName = null;
                              mLohsInterfaceMode = WifiManager.IFACE_IP_MODE_UNSPECIFIED;
                              sendHotspotFailedMessageToAllLOHSRequestInfoEntriesLocked(
                                      LocalOnlyHotspotCallback.ERROR_INCOMPATIBLE_MODE);
                          }
                          break;
                      case WifiManager.IFACE_IP_MODE_CONFIGURATION_ERROR:
                          if (ifaceName == null) {
                              // All softAps
                              mLohsInterfaceName = null;
                              mLohsInterfaceMode = mode;
                              sendHotspotFailedMessageToAllLOHSRequestInfoEntriesLocked(
                                      LocalOnlyHotspotCallback.ERROR_GENERIC);
                              stopSoftApInternal(WifiManager.IFACE_IP_MODE_UNSPECIFIED);
                          } else if (ifaceName.equals(mLohsInterfaceName)) {
                              mLohsInterfaceName = null;
                              mLohsInterfaceMode = mode;
                              sendHotspotFailedMessageToAllLOHSRequestInfoEntriesLocked(
                                      LocalOnlyHotspotCallback.ERROR_GENERIC);
                              stopSoftApInternal(WifiManager.IFACE_IP_MODE_LOCAL_ONLY);
                          } else {
                              // Not for LOHS. This is the wrong place to do this, but...
                              stopSoftApInternal(WifiManager.IFACE_IP_MODE_TETHERED);
                          }
                          break;
                      case WifiManager.IFACE_IP_MODE_UNSPECIFIED:
                          if (ifaceName == null || ifaceName.equals(mLohsInterfaceName)) {
                              mLohsInterfaceName = null;
                              mLohsInterfaceMode = mode;
                          }
                          break;
                      default:
                          mLog.warn("updateInterfaceIpState: unknown mode %").c(mode).flush();
                  }
              }
          }
  
          /**
           * Helper method to send a HOTSPOT_FAILED message to all registered LocalOnlyHotspotRequest
           * callers and clear the registrations.
           *
           * Callers should already hold the mLocalOnlyHotspotRequests lock.
           */
          @GuardedBy("mLocalOnlyHotspotRequests")
          private void sendHotspotFailedMessageToAllLOHSRequestInfoEntriesLocked(int reason) {
              for (LocalOnlyHotspotRequestInfo requestor : mLocalOnlyHotspotRequests.values()) {
                  try {
                      requestor.sendHotspotFailedMessage(reason);
                      requestor.unlinkDeathRecipient();
                  } catch (RemoteException e) {
                      // This will be cleaned up by binder death handling
                  }
              }
  
              // Since all callers were notified, now clear the registrations.
              mLocalOnlyHotspotRequests.clear();
          }
  
          /**
           * Helper method to send a HOTSPOT_STOPPED message to all registered LocalOnlyHotspotRequest
           * callers and clear the registrations.
           *
           * Callers should already hold the mLocalOnlyHotspotRequests lock.
           */
          @GuardedBy("mLocalOnlyHotspotRequests")
          private void sendHotspotStoppedMessageToAllLOHSRequestInfoEntriesLocked() {
              for (LocalOnlyHotspotRequestInfo requestor : mLocalOnlyHotspotRequests.values()) {
                  try {
                      requestor.sendHotspotStoppedMessage();
                      requestor.unlinkDeathRecipient();
                  } catch (RemoteException e) {
                      // This will be cleaned up by binder death handling
                  }
              }
  
              // Since all callers were notified, now clear the registrations.
              mLocalOnlyHotspotRequests.clear();
          }
  
          /**
           * Add a new LOHS client
           */
          private int start(int pid, LocalOnlyHotspotRequestInfo request) {
              synchronized (mLocalOnlyHotspotRequests) {
                  // does this caller already have a request?
                  if (mLocalOnlyHotspotRequests.get(pid) != null) {
                      mLog.trace("caller already has an active request").flush();
                      throw new IllegalStateException(
                              "Caller already has an active LocalOnlyHotspot request");
                  }
  
                  // Never accept exclusive requests (with custom configuration) at the same time as
                  // shared requests.
                  if (!mLocalOnlyHotspotRequests.isEmpty()) {
                      boolean requestIsExclusive = request.getCustomConfig() != null;
                      if (mIsExclusive || requestIsExclusive) {
                          mLog.trace("Cannot share with existing LOHS request due to custom config")
                                  .flush();
                          return LocalOnlyHotspotCallback.ERROR_GENERIC;
                      }
                  }
  
                  // At this point, the request is accepted.
                  if (mLocalOnlyHotspotRequests.isEmpty()) {
                      startForFirstRequestLocked(request);
                  } else if (mLohsInterfaceMode == WifiManager.IFACE_IP_MODE_LOCAL_ONLY) {
                      // LOHS has already started up for an earlier request, so we can send the
                      // current config to the incoming request right away.
                      try {
                          mLog.trace("LOHS already up, trigger onStarted callback").flush();
                          request.sendHotspotStartedMessage(mActiveConfig.getSoftApConfiguration());
                      } catch (RemoteException e) {
                          return LocalOnlyHotspotCallback.ERROR_GENERIC;
                      }
                  }
  
                  mLocalOnlyHotspotRequests.put(pid, request);
                  return LocalOnlyHotspotCallback.REQUEST_REGISTERED;
              }
          }
  
          @GuardedBy("mLocalOnlyHotspotRequests")
          private void startForFirstRequestLocked(LocalOnlyHotspotRequestInfo request) {
              int band = SoftApConfiguration.BAND_2GHZ;
  
              // For auto only
              if (hasAutomotiveFeature(mContext)) {
                  if (mContext.getResources().getBoolean(R.bool.config_wifiLocalOnlyHotspot6ghz)
                          && mContext.getResources().getBoolean(R.bool.config_wifiSoftap6ghzSupported)
                          && is6GhzBandSupportedInternal()) {
                      band = SoftApConfiguration.BAND_6GHZ;
                  } else if (mContext.getResources().getBoolean(
                          R.bool.config_wifi_local_only_hotspot_5ghz)
                          && is5GhzBandSupportedInternal()) {
                      band = SoftApConfiguration.BAND_5GHZ;
                  }
              }
  
              SoftApConfiguration softApConfig = WifiApConfigStore.generateLocalOnlyHotspotConfig(
                      mContext, band, request.getCustomConfig());
  
              mActiveConfig = new SoftApModeConfiguration(
                      WifiManager.IFACE_IP_MODE_LOCAL_ONLY,
                      softApConfig, mLohsSoftApTracker.getSoftApCapability());
              mIsExclusive = (request.getCustomConfig() != null);
  
              startSoftApInternal(mActiveConfig);
          }
  
          /**
           * Requests that any local-only hotspot be stopped.
           */
          public void stopAll() {
              synchronized (mLocalOnlyHotspotRequests) {
                  if (!mLocalOnlyHotspotRequests.isEmpty()) {
                      // This is used to take down LOHS when tethering starts, and in that
                      // case we send failed instead of stopped.
                      // TODO check if that is right. Calling onFailed instead of onStopped when the
                      // hotspot is already started does not seem to match the documentation
                      sendHotspotFailedMessageToAllLOHSRequestInfoEntriesLocked(
                              LocalOnlyHotspotCallback.ERROR_INCOMPATIBLE_MODE);
                      stopIfEmptyLocked();
                  }
              }
          }
  
          /**
           * Unregisters the LOHS request from the given process and stops LOHS if no other clients.
           */
          public void stopByPid(int pid) {
              synchronized (mLocalOnlyHotspotRequests) {
                  LocalOnlyHotspotRequestInfo requestInfo = mLocalOnlyHotspotRequests.remove(pid);
                  if (requestInfo == null) return;
                  requestInfo.unlinkDeathRecipient();
                  stopIfEmptyLocked();
              }
          }
  
          /**
           * Unregisters LocalOnlyHotspot request and stops the hotspot if needed.
           */
          public void stopByRequest(LocalOnlyHotspotRequestInfo request) {
              synchronized (mLocalOnlyHotspotRequests) {
                  if (mLocalOnlyHotspotRequests.remove(request.getPid()) == null) {
                      mLog.trace("LocalOnlyHotspotRequestInfo not found to remove").flush();
                      return;
                  }
                  stopIfEmptyLocked();
              }
          }
  
          @GuardedBy("mLocalOnlyHotspotRequests")
          private void stopIfEmptyLocked() {
              if (mLocalOnlyHotspotRequests.isEmpty()) {
                  mActiveConfig = null;
                  mIsExclusive = false;
                  mLohsInterfaceName = null;
                  mLohsInterfaceMode = WifiManager.IFACE_IP_MODE_UNSPECIFIED;
                  stopSoftApInternal(WifiManager.IFACE_IP_MODE_LOCAL_ONLY);
              }
          }
  
          /**
           * Helper method to send a HOTSPOT_STARTED message to all registered LocalOnlyHotspotRequest
           * callers.
           *
           * Callers should already hold the mLocalOnlyHotspotRequests lock.
           */
          @GuardedBy("mLocalOnlyHotspotRequests")
          private void sendHotspotStartedMessageToAllLOHSRequestInfoEntriesLocked() {
              for (LocalOnlyHotspotRequestInfo requestor : mLocalOnlyHotspotRequests.values()) {
                  try {
                      requestor.sendHotspotStartedMessage(mActiveConfig.getSoftApConfiguration());
                  } catch (RemoteException e) {
                      // This will be cleaned up by binder death handling
                  }
              }
          }
  
          @Override
          public void onStateChanged(int state, int failureReason) {
              // The AP state update from ClientModeImpl for softap
              synchronized (mLocalOnlyHotspotRequests) {
                  Log.d(TAG, "lohs.onStateChanged: currentState=" + state
                          + " previousState=" + mLohsState + " errorCode= " + failureReason
                          + " ifaceName=" + mLohsInterfaceName);
  
                  // check if we have a failure - since it is possible (worst case scenario where
                  // WifiController and ClientModeImpl are out of sync wrt modes) to get two FAILED
                  // notifications in a row, we need to handle this first.
                  if (state == WIFI_AP_STATE_FAILED) {
                      // update registered LOHS callbacks if we see a failure
                      int errorToReport = ERROR_GENERIC;
                      if (failureReason == SAP_START_FAILURE_NO_CHANNEL) {
                          errorToReport = ERROR_NO_CHANNEL;
                      }
                      // holding the required lock: send message to requestors and clear the list
                      sendHotspotFailedMessageToAllLOHSRequestInfoEntriesLocked(errorToReport);
                      // also need to clear interface ip state
                      updateInterfaceIpState(mLohsInterfaceName,
                              WifiManager.IFACE_IP_MODE_UNSPECIFIED);
                  } else if (state == WIFI_AP_STATE_DISABLING || state == WIFI_AP_STATE_DISABLED) {
                      // softap is shutting down or is down...  let requestors know via the
                      // onStopped call
                      // if we are currently in hotspot mode, then trigger onStopped for registered
                      // requestors, otherwise something odd happened and we should clear state
                      if (mLohsInterfaceName != null
                              && mLohsInterfaceMode == WifiManager.IFACE_IP_MODE_LOCAL_ONLY) {
                          // holding the required lock: send message to requestors and clear the list
                          sendHotspotStoppedMessageToAllLOHSRequestInfoEntriesLocked();
                      } else {
                          // LOHS not active: report an error (still holding the required lock)
                          sendHotspotFailedMessageToAllLOHSRequestInfoEntriesLocked(ERROR_GENERIC);
                      }
                      // also clear interface ip state
                      updateInterfaceIpState(mLohsInterfaceName,
                              WifiManager.IFACE_IP_MODE_UNSPECIFIED);
                  }
                  // For enabling and enabled, just record the new state
                  mLohsState = state;
              }
          }
在start()中通过startForFirstRequestLocked(request);来启动热点
@GuardedBy("mLocalOnlyHotspotRequests")
          private void startForFirstRequestLocked(LocalOnlyHotspotRequestInfo request) {
              int band = SoftApConfiguration.BAND_2GHZ;
  
              // For auto only
              if (hasAutomotiveFeature(mContext)) {
                  if (mContext.getResources().getBoolean(R.bool.config_wifiLocalOnlyHotspot6ghz)
                          && mContext.getResources().getBoolean(R.bool.config_wifiSoftap6ghzSupported)
                          && is6GhzBandSupportedInternal()) {
                      band = SoftApConfiguration.BAND_6GHZ;
                  } else if (mContext.getResources().getBoolean(
                          R.bool.config_wifi_local_only_hotspot_5ghz)
                          && is5GhzBandSupportedInternal()) {
                      band = SoftApConfiguration.BAND_5GHZ;
                  }
              }
             // 启动本地热点
              SoftApConfiguration softApConfig = WifiApConfigStore.generateLocalOnlyHotspotConfig(
                      mContext, band, request.getCustomConfig());
  
              mActiveConfig = new SoftApModeConfiguration(
                      WifiManager.IFACE_IP_MODE_LOCAL_ONLY,
                      softApConfig, mLohsSoftApTracker.getSoftApCapability());
              mIsExclusive = (request.getCustomConfig() != null);
  
              startSoftApInternal(mActiveConfig);
          }
 
```

### 在 startLocalOnlyHotspot 的方法中发现最终通过 mLohsSoftApTracker.start(pid, request); 来启动本地热点

在内部类 LohsSoftApTracker 中的 start 方法中通过 startForFirstRequestLocked(request); 来启动热点

最终发现 WifiApConfigStore.generateLocalOnlyHotspotConfig() 来启动本地热点

### 3.3 WifiApConfigStore 相关代码分析

```
public static SoftApConfiguration generateLocalOnlyHotspotConfig(Context context, int apBand,
              @Nullable SoftApConfiguration customConfig) {
          SoftApConfiguration.Builder configBuilder;
          if (customConfig != null) {
              configBuilder = new SoftApConfiguration.Builder(customConfig);
          } else {
              configBuilder = new SoftApConfiguration.Builder();
              // Default to disable the auto shutdown
              configBuilder.setAutoShutdownEnabled(false);
          }
  
          configBuilder.setBand(apBand);
  
          if (customConfig == null || customConfig.getSsid() == null) {
              configBuilder.setSsid(generateLohsSsid(context));
          }
          if (customConfig == null) {
              if (ApConfigUtil.isWpa3SaeSupported(context)) {
                  configBuilder.setPassphrase(generatePassword(),
                          SoftApConfiguration.SECURITY_TYPE_WPA3_SAE_TRANSITION);
              } else {
                  configBuilder.setPassphrase(generatePassword(),
                          SoftApConfiguration.SECURITY_TYPE_WPA2_PSK);
              }
          }
  
          return configBuilder.build();
      }
    // 默认ssid
     private static String generateLohsSsid(Context context) {
          return context.getResources().getString(
                  R.string.wifi_localhotspot_configure_ssid_default) + "_"
                  + getRandomIntForDefaultSsid();
      }
   // 默认密码
      private static String generatePassword() {
          // Characters that will be used for password generation. Some characters commonly known to
          // be confusing like 0 and O excluded from this list.
          final String allowed = "23456789abcdefghijkmnpqrstuvwxyz";
          final int passLength = 15;
  
          StringBuilder sb = new StringBuilder(passLength);
          SecureRandom random = new SecureRandom();
          for (int i = 0; i < passLength; i++) {
              sb.append(allowed.charAt(random.nextInt(allowed.length())));
          }
          return sb.toString();
      }
 
```

通过代码发现 generatePassword() 设置默认密码 generateLohsSsid(Context context) 设置默认 ssid

所以具体修改为:

```
所以具体为:
 // 默认ssid
     private static String generateLohsSsid(Context context) {
         - return context.getResources().getString(
         -         R.string.wifi_localhotspot_configure_ssid_default) + "_"
         -         + getRandomIntForDefaultSsid();
         + return context.getResources().getString(
         +         R.string.wifi_localhotspot_configure_ssid_default) + "_123";
      }
   // 默认密码
      private static String generatePassword() {
          // Characters that will be used for password generation. Some characters commonly known to
          // be confusing like 0 and O excluded from this list.
          /*final String allowed = "23456789abcdefghijkmnpqrstuvwxyz";
          final int passLength = 15;
  
          StringBuilder sb = new StringBuilder(passLength);
          SecureRandom random = new SecureRandom();
          for (int i = 0; i < passLength; i++) {
              sb.append(allowed.charAt(random.nextInt(allowed.length())));
          }
          return sb.toString();*/
          return "12345678";
      }
```
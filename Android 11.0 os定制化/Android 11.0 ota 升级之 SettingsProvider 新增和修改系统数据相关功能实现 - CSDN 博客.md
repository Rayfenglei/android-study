> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/132655930)

1. 前言
-----

  在 11.0 的系统 rom 定制化开发中，在解决一些已经上线的 bug 后，进行 [ota 升级](https://so.csdn.net/so/search?q=ota%E5%8D%87%E7%BA%A7&spm=1001.2101.3001.7020)的过程中，由于在 SettingsProvider 中新增了系统属性和修改某项系统属性值，但是在 ota 升级以后发现没有  
更新，需要[恢复出厂设置](https://so.csdn.net/so/search?q=%E6%81%A2%E5%A4%8D%E5%87%BA%E5%8E%82%E8%AE%BE%E7%BD%AE&spm=1001.2101.3001.7020)以后才会更改，但是恢复出厂设置 会丢掉一些数据，这是应为系统数据库没更新，所以需要在 ota 的时候同样升级系统数据库

2.ota 升级之 SettingsProvider 新增和修改系统数据相关功能实现的核心类
----------------------------------------------

```
    \frameworks\base\services\java\com\android\server\SystemServer.java
    \frameworks\base\services\core\java\com\android\server\am\ActivityManagerService.java
    \frameworks\base\packages\SettingsProvider\src\com\android\providers\settings\SettingsProvider.java
    \frameworks\base\packages\SettingsProvider\res\values\defaults.xml
```

3.ota 升级之 SettingsProvider 新增和修改系统数据相关功能实现的核心功能分析和实现
----------------------------------------------------

ota 升级之 SettingsProvider 新增和修改系统数据相关功能实现中，

SettingsProvider 顾名思义是一个提供设置数据共享的 Provider，SettingsProvider 和 Android 系统其它 Provider 有很多不一样的地方，如：  
SettingsProvider 只接受 int、float、string 等基本类型的数据；  
SettingsProvider 由 Android 系统 framework 进行了封装，使用更加快捷方便  
SettingsProvider 的数据由键值对组成  
SettingsProvider 有点类似 Android 的 properties 系统（Android 属性系统）：SystemProperties。  
在系统中，SettingsProvider 是系统设置的内容提供者。它将设置类型分为三种

    Global，全局，对系统中所有用户公开，第三方 App 没有写权限  
    Secure，安全相关的用户偏好设置，第三方 App 没有写权限  
    System，用户偏好系统设置

在最新 android 9.0 系统中，数据存储由原来的数据库 settings.db，转移保存到 xml 中：

    data/system/users/0/settings_global.xml  
    data/system/users/userid/settings_system.xml  
    data/system/users/userid/settings_secure.xml

3.1 SystemServer.java 中启动 SettingsProvider 的相关代码分析
--------------------------------------------------

```
    private void startOtherServices() {
    	//省略一部分代码
    	//...
    	
    	traceBeginAndSlog("InstallSystemProviders");
    	mActivityManagerService.installSystemProviders();
    	// Now that SettingsProvider is ready, reactivate SQLiteCompatibilityWalFlags
    	SQLiteCompatibilityWalFlags.reset();
    	traceEnd();
    	
    	//省略一部分代码
    	//...
    }
```

ota 升级之 SettingsProvider 新增和修改系统数据相关功能实现中，

在系统 11.0 的 SystemServer.java 中的上述相关代码中，可以发现在 SystemServer 启动的过程中，在启动 startOtherServices() 中，调用  
mActivityManagerService.installSystemProviders(); 来进行 SettingsProvider 的安装，然后接下来  
初始化 SettingProvider 的相关方法，经过分析发现最终来加载系统中的 Global Secure System 等相关系统属性，接下来  
看下 AMS 中相关 SettingsProvider 的安装流程，来看下具体的安装流程

```
    public final void installSystemProviders() {
              List<ProviderInfo> providers;
              synchronized (this) {
                  ProcessRecord app = mProcessList.mProcessNames.get("system", SYSTEM_UID);
                  providers = generateApplicationProvidersLocked(app);
                  if (providers != null) {
                      for (int i=providers.size()-1; i>=0; i--) {
                          ProviderInfo pi = (ProviderInfo)providers.get(i);
                          if ((pi.applicationInfo.flags&ApplicationInfo.FLAG_SYSTEM) == 0) {
                              Slog.w(TAG, "Not installing system proc provider " + pi.name
                                      + ": not system .apk");
                              providers.remove(i);
                          }
                      }
                  }
              }
              if (providers != null) {
                  mSystemThread.installSystemProviders(providers);
              }
      
              synchronized (this) {
                  mSystemProvidersInstalled = true;
              }
              mConstants.start(mContext.getContentResolver());
              mCoreSettingsObserver = new CoreSettingsObserver(this);
              mActivityTaskManager.installSystemProviders();
              mDevelopmentSettingsObserver = new DevelopmentSettingsObserver();
              SettingsToPropertiesMapper.start(mContext.getContentResolver());
              mOomAdjuster.initSettings();
      
              // Now that the settings provider is published we can consider sending
              // in a rescue party.
              RescueParty.onSettingsProviderPublished(mContext);
      
              //mUsageStatsService.monitorPackages();
          }
```

ota 升级之 SettingsProvider 新增和修改系统数据相关功能实现中，

在 ActivityManagerService.java 中的上述代码中可以看出，在 installSystemProviders()  
中是调用 mSystemThread.installSystemProviders(providers); 来继续[安装数据库](https://so.csdn.net/so/search?q=%E5%AE%89%E8%A3%85%E6%95%B0%E6%8D%AE%E5%BA%93&spm=1001.2101.3001.7020)  
和保存相关系统数据的，最终会通过调用 installSystemProviders() 会调用到 SettingsProvider 中的 onCreate() 方法  
接下来看下 SettingsProvider.java 的相关方法

3.2 SettingsProvider.java 的相关数据方法分析
-----------------------------------

```
     @Override
         public boolean onCreate() {
             Settings.setInSystemServer();
     
             // fail to boot if there're any backed up settings that don't have a non-null validator
             ensureAllBackedUpSystemSettingsHaveValidators();
             ensureAllBackedUpGlobalSettingsHaveValidators();
             ensureAllBackedUpSecureSettingsHaveValidators();
     
             synchronized (mLock) {
                 mUserManager = UserManager.get(getContext());
                 mUserManagerInternal = LocalServices.getService(UserManagerInternal.class);
                 mPackageManager = AppGlobals.getPackageManager();
                 mHandlerThread = new HandlerThread(LOG_TAG,
                         Process.THREAD_PRIORITY_BACKGROUND);
                 mHandlerThread.start();
                 mHandler = new Handler(mHandlerThread.getLooper());
                 mSettingsRegistry = new SettingsRegistry();
             }
             mHandler.post(() -> {
                 registerBroadcastReceivers();
                 startWatchingUserRestrictionChanges();
             });
             ServiceManager.addService("settings", new SettingsService(this));
             ServiceManager.addService("device_config", new DeviceConfigService(this));
             return true;
         }
       final class SettingsRegistry {
      
              public SettingsRegistry() {
                  mHandler = new MyHandler(getContext().getMainLooper());
                  mGenerationRegistry = new GenerationRegistry(mLock);
                  mBackupManager = new BackupManager(getContext());
                  migrateAllLegacySettingsIfNeeded();
                  syncSsaidTableOnStart();
              }
```

ota 升级之 SettingsProvider 新增和修改系统数据相关功能实现中，

在 SettingsProvider.java 的上述方法中，在 onCreate() 构建系统数据库的时候，最关键的就是  
mSettingsRegistry = new SettingsRegistry(); 在 SettingsRegistry(); 中的构造方法中，调用  
migrateAllLegacySettingsIfNeeded(); 来往数据库添加值，而在 migrateAllLegacySettingsIfNeeded(); 中  
主要的就是调用 UpgradeController upgrader = new UpgradeController(userId);  
 upgrader.upgradeIfNeededLocked(); 来更新数据库了  
所以说 ota 的过程中，需要更新数据库 就需要在 upgrader.upgradeIfNeededLocked(); 中来更新

接下来来实现功能

```
           private final class UpgradeController {
              -    private static final int SETTINGS_VERSION = 182;
              +    private static final int SETTINGS_VERSION = 183; 
                  private final int mUserId;
      
                  public UpgradeController(int userId) {
                      mUserId = userId;
                  }
      
                  public void upgradeIfNeededLocked() {
                      // The version of all settings for a user is the same (all users have secure).
                      SettingsState secureSettings = getSettingsLocked(
                              SETTINGS_TYPE_SECURE, mUserId);
      
                      // Try an update from the current state.
                      final int oldVersion = secureSettings.getVersionLocked();
                      final int newVersion = SETTINGS_VERSION;
      
                      // If up do date - done.
                      if (oldVersion == newVersion) {
                          return;
                      }
      
                      // Try to upgrade.
                      final int curVersion = onUpgradeLocked(mUserId, oldVersion, newVersion);
    ....
                 private int onUpgradeLocked(int userId, int oldVersion, int newVersion) {
                     if (DEBUG) {
                          Slog.w(LOG_TAG, "Upgrading settings for user: " + userId + " from version: "
                                 + oldVersion + " to version: " + newVersion);
                      }
      
                      int currentVersion = oldVersion;
    ....
                        currentVersion = 182;
                    }
     
    //add core start
                    if (currentVersion == 182) {
                        // Version cd : by default, add STREAM_BLUETOOTH_SCO to list of streams that can
                        // be muted.
                        final SettingsState systemSettings = getSecureSettingsLocked(userId);
                        // vXXX: Add new settings above this point.
                        //if (systemSettings.getSettingLocked(Settings.Secure.ENABLED_INPUT_METHODS).isNull()) {
                            String def_input_methods = getContext()
                                    .getResources().getString(R.string.def_enabled_input_methods);
                            systemSettings.insertSettingLocked(Settings.Secure.ENABLED_INPUT_METHODS,
                                    def_input_methods, null, true,
                                    SettingsState.SYSTEM_PACKAGE_NAME);
                        //}
                        currentVersion = 183;
    	}
    //add core end
     
                    if (currentVersion != newVersion) {
                        Slog.wtf("SettingsProvider", "warning: upgrading settings database to version "
                                + newVersion + " left it at "
                                + currentVersion +
                                " instead; this is probably a bug. Did you update SETTINGS_VERSION?",
                                new Throwable());
                        if (DEBUG) {
                            throw new RuntimeException("db upgrade error");
                        }
                    }
```

ota 升级之 SettingsProvider 新增和修改系统数据相关功能实现中，

在 SettingsProvider.java 的上述方法中，通过在内部类 UpgradeController 中实现数据库的升级，首选需要更改 SETTINGS_VERSION 的值 要和 currentVersion = 183; 保持一致，然后在原来的  
onUpgradeLocked(int userId, int oldVersion, int newVersion) 中，添加  
判断，当 if (currentVersion == 182) 的时候，添加系统数据或者修改系统数据都需要在这里从新  
添加到数据库中，通过上述的案例中，就实现了在 ota 的过程中，修改了系统默认输入法，最终  
通过 ota 升级后发现已经实现了功能
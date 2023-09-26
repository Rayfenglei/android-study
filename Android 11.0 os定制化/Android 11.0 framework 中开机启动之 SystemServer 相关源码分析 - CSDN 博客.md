> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/131564432)

1. 前言
-----

  在 11.0 的系统开发中，systemserver 进程也是非常重要的, system_server 进程承载着整个 framework 的核心服务，  
例如创建 ActivityManagerService、PowerManagerService、DisplayManagerService、PackageManagerService、WindowManagerService、  
LauncherAppsService 等 80 多个核心系统服务。这些服务以不同的线程方式存在于 system_server 这个进程中, 接下来简单分析下启动的相关的  
服务的源码

2. framework 中开机启动之 SystemServer 相关源码分析的核心类
-------------------------------------------

```
 /frameworks/base/services/java/com/android/server/SystemServer.java

```

3. framework 中开机启动之 SystemServer 相关源码分析
---------------------------------------

Zygote 是所有应用的鼻祖。SystemServer 和其他所有 Dalivik 虚拟机进程都是由 Zygote fork 而来。Zygote fork 的第一个进程就是 SystemServer，其在手机中的进程名为 system_server。  
 system_server 进程承载着整个 framework 的核心服务，例如用来创建 ActivityManagerService、PowerManagerService、DisplayManagerService、PackageManagerService、  
WindowManagerService、LauncherAppsService 等核心系统服务

```
   private void run() {
            try {
                traceBeginAndSlog("InitBeforeStartServices");
    .....
     
     
    //准备主线程lopper
                // Prepare the main looper thread (this thread).
                android.os.Process.setThreadPriority(
                        android.os.Process.THREAD_PRIORITY_FOREGROUND);
                android.os.Process.setCanSelfBackground(false);
                Looper.prepareMainLooper();
                Looper.getMainLooper().setSlowLogThresholdMs(
                        SLOW_DISPATCH_THRESHOLD_MS, SLOW_DELIVERY_THRESHOLD_MS);
    // 加载libandroid_servers.so库
                // Initialize native services.
                System.loadLibrary("android_servers");
     
                // Debug builds - allow heap profiling.
                if (Build.IS_DEBUGGABLE) {
                    initZygoteChildHeapProfiling();
                }
    //检测上次关机过程是否失败
                // Check whether we failed to shut down last time we tried.
                // This call may not return.
                performPendingShutdown();
    //初始化系统上下文
                // Initialize the system context.
                createSystemContext();
    //创建系统服务管理者--SystemServiceManager
                // Create the system service manager.
                mSystemServiceManager = new SystemServiceManager(mSystemContext);
                mSystemServiceManager.setStartInfo(mRuntimeRestart,
                        mRuntimeStartElapsedTime, mRuntimeStartUptime);
                LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
                // Prepare the thread pool for init tasks that can be parallelized
                //为可以并行化的init任务准备线程池
    SystemServerInitThreadPool.get();
            } finally {
                traceEnd();  // InitBeforeStartServices
            }
     
            // Start services.
            try {
                traceBeginAndSlog("StartServices");
    // 启动引导服务
                startBootstrapServices();
    // 启动核心服务
                startCoreServices();
    // 启动其他服务
                startOtherServices();
    //停止init线程池
                SystemServerInitThreadPool.shutdown();
            } catch (Throwable ex) {
                Slog.e("System", "******************************************");
                Slog.e("System", "************ Failure starting system services", ex);
                throw ex;
            } finally {
                traceEnd();
            }
     
            StrictMode.initVmDefaults(null);
     
            if (!mRuntimeRestart && !isFirstBootOrUpgrade()) {
                int uptimeMillis = (int) SystemClock.elapsedRealtime();
                MetricsLogger.histogram(null, "boot_system_server_ready", uptimeMillis);
                final int MAX_UPTIME_MILLIS = 60 * 1000;
                if (uptimeMillis > MAX_UPTIME_MILLIS) {
                    Slog.wtf(SYSTEM_SERVER_TIMING_TAG,
                            "SystemServer init took too long. uptimeMillis=" + uptimeMillis);
                }
            }
     
            // Diagnostic to ensure that the system is in a base healthy state. Done here as a common
            // non-zygote process.
            if (!VMRuntime.hasBootImageSpaces()) {
                Slog.wtf(TAG, "Runtime is not running with a boot image!");
            }
     
            // Loop forever.
            Looper.loop();
            throw new RuntimeException("Main thread loop unexpectedly exited");
        }
```

在上述的 SystemServer.java 的 run（）方法中，在系统启动 systemserver 进程的时候，就会在这里启动  
引导服务，核心服务 其他服务等等重要的服务，这也是整个系统非常关键的相关服务，在这里  
Zygote fork 系统的引导服务，核心服务 其他服务等等重要的服务

```
   private void startBootstrapServices() {
            // 最先启动Watchdog看门狗，这样可以防止在早期启动陷入死锁时就可以使system server崩溃重启。
            final Watchdog watchdog = Watchdog.getInstance();
            watchdog.start();
    ...
            //启动Installer Service，这个Service 通过binder与installd进程通讯，负责apk安装相关的工作
            Installer installer = mSystemServiceManager.startService(Installer.class);
            //设备标识符策略服务
            mSystemServiceManager.startService(DeviceIdentifiersPolicyService.class);
            // 管理uri授权
            mSystemServiceManager.startService(UriGrantsManagerService.Lifecycle.class);
            //启动ActivityTaskManagerService和ActivityManagerService
            ActivityTaskManagerService atm = mSystemServiceManager.startService(
                    ActivityTaskManagerService.Lifecycle.class).getService();
            mActivityManagerService = ActivityManagerService.Lifecycle.startService(
                    mSystemServiceManager, atm);
            mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
            mActivityManagerService.setInstaller(installer);
            mWindowManagerGlobalLock = atm.getGlobalLock();
            //电源管理器需要尽早启动，因为其他服务需要它。
            mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
            //启动热缓解服务，目的是在手机开始过热时进行有效的热缓解
            mSystemServiceManager.startService(ThermalManagerService.class);
            // Now that the power manager has been started, let the activity manager
            // initialize power management features.
            mActivityManagerService.initPowerManagement();
            //启动系统恢复服务，负责协调设备上与恢复有关的功能。
            mSystemServiceManager.startService(RecoverySystemService.class);
            //到这里为止，系统启动的必须服务已经加载完毕
            RescueParty.noteBoot(mSystemContext);
            //管理LED和屏幕背光，我们需要它来显示
            mSystemServiceManager.startService(LightsService.class);
            //管理显示设备
            //在package manager 启动之前，需要启动display manager 提供display metrics
            mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);
         //只有DisplayManagerService会对PHASE_WAIT_FOR_DEFAULT_DISPLAY做处理
         //目的是在初始化包管理器之前，首先需要获取一个默认的显示设备
         mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);
            // 启动 package manager.
            if (!mRuntimeRestart) {
                MetricsLogger.histogram(null, "boot_package_manager_init_start",
                        (int) SystemClock.elapsedRealtime());
            }
            try {
                Watchdog.getInstance().pauseWatchingCurrentThread("packagemanagermain");
                mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                        mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
            } finally {
                Watchdog.getInstance().resumeWatchingCurrentThread("packagemanagermain");
            }
            mFirstBoot = mPackageManagerService.isFirstBoot();
            mPackageManager = mSystemContext.getPackageManager();
            //启动UserManager Service
            mSystemServiceManager.startService(UserManagerService.LifeCycle.class);
           //为系统进程设置应用程序实例并启动
            mActivityManagerService.setSystemProcess();
            //使用ActivityManager实例完成看门狗设置并监听是否重启
            watchdog.init(mSystemContext, mActivityManagerService);
            // DisplayManagerService needs to setup android.display scheduling related policies
            // since setSystemProcess() would have overridden policies due to setProcessGroup
            mDisplayManagerService.setupSchedulerPolicies();
            //负责动态资源overlay
            mSystemServiceManager.startService(new OverlayManagerService(mSystemContext, installer));
            mSystemServiceManager.startService(new SensorPrivacyService(mSystemContext));
            if (SystemProperties.getInt("persist.sys.displayinset.top", 0) > 0) {
                // DisplayManager needs the overlay immediately.
                mActivityManagerService.updateSystemUiContext();
                LocalServices.getService(DisplayManagerInternal.class).onOverlayChanged();
            }
              //传感器服务需要访问包管理器服务、app ops服务和权限服务，
             //因此我们在它们之后启动它。
             //在单独的线程中启动传感器服务。在使用它之前应该检查完成情况。
            mSensorServiceStart = SystemServerInitThreadPool.get().submit(() -> {
                TimingsTraceLog traceLog = new TimingsTraceLog(
                        SYSTEM_SERVER_TIMING_ASYNC_TAG, Trace.TRACE_TAG_SYSTEM_SERVER);
                traceLog.traceBegin(START_SENSOR_SERVICE);
                startSensorService();
                traceLog.traceEnd();
            }, START_SENSOR_SERVICE);
        }
```

在启动引导服务中，大概启动了 15 个比较重要的服务，也就是  
Installer                                    负责 apk 安装相关的工作  
DeviceIdentifiersPolicyService    设备标识符策略服务  
UriGrantsManagerService    Uri 授权管理  
ActivityTaskManagerService    用于管理 Activity 及其容器 (task, stacks, displays,...) 的系统服务  
ActivityManagerService    管理 Activity 的启动，调度等工作  
PowerManagerService    负责协调设备上的电源管理功能  
ThermalManagerService    热缓解服务  
RecoverySystemService    负责协调设备上与恢复有关的功能  
LightsService                    管理 LED 和屏幕背光  
DisplayManagerService    管理显示设备  
PackageManagerService    主要负责 APK、jar 包的管理  
UserManagerService                    管理用户的系统服务  
OverlayManagerService    负责动态资源 [overlay](https://so.csdn.net/so/search?q=overlay&spm=1001.2101.3001.7020) 工作，具体请搜索 android RRO 技术  
SensorPrivacyService                    和传感器有关，具体作用不明  
SensorPrivacySere                    传感器服务

```
      private void startCoreServices() {
    // 追踪电池充电状态和电量。需要LightService
            traceBeginAndSlog("StartBatteryService");
            // Tracks the battery level.  Requires LightService.
            mSystemServiceManager.startService(BatteryService.class);
            traceEnd();
    //跟踪应用程序使用状态
            // Tracks application usage stats.
            traceBeginAndSlog("StartUsageService");
            mSystemServiceManager.startService(UsageStatsService.class);
            mActivityManagerService.setUsageStatsManager(
                    LocalServices.getService(UsageStatsManagerInternal.class));
            traceEnd();
    // 跟踪可更新的WebView是否处于就绪状态，并监视更新安装。
            // Tracks whether the updatable WebView is in a ready state and watches for update installs.
            if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_WEBVIEW)) {
                traceBeginAndSlog("StartWebViewUpdateService");
                mWebViewUpdateService = mSystemServiceManager.startService(WebViewUpdateService.class);
                traceEnd();
            }
    //跟踪并缓存设备状态。
            // Tracks and caches the device state.
            traceBeginAndSlog("StartCachedDeviceStateService");
            mSystemServiceManager.startService(CachedDeviceStateService.class);
            traceEnd();
    // 跟踪在Binder调用中花费的cpu时间
            // Tracks cpu time spent in binder calls
            traceBeginAndSlog("StartBinderCallsStatsService");
            mSystemServiceManager.startService(BinderCallsStatsService.LifeCycle.class);
            traceEnd();
    // 跟踪handlers中处理messages所花费的时间。
            // Tracks time spent in handling messages in handlers.
            traceBeginAndSlog("StartLooperStatsService");
            mSystemServiceManager.startService(LooperStatsService.Lifecycle.class);
            traceEnd();
     //管理apk回滚
            // Manages apk rollbacks.
            String tinyFlag = SystemProperties.get("ro.tiny", "");
            if (tinyFlag == "") {
                traceBeginAndSlog("StartRollbackManagerService");
                mSystemServiceManager.startService(RollbackManagerService.class);
                traceEnd();
            }
    // 用于捕获bugreport,adb bugreport 命令调用的就是这个服务
            // Service to capture bugreports.
            traceBeginAndSlog("StartBugreportManagerService");
            mSystemServiceManager.startService(BugreportManagerService.class);
            traceEnd();
     // 管理Gpu和Gpu驱动的服务
            // Serivce for GPU and GPU driver.
            traceBeginAndSlog("GpuService");
            mSystemServiceManager.startService(GpuService.class);
            traceEnd();
        }
```

在 systemserver 进程中，启动的核心服务中有 9 个比较重要的服务如下:  
服务名称                                    描述  
BatteryService                    追踪电池充电状态和电量  
UsageStatsManagerInternal    跟踪应用程序使用状态  
WebViewUpdateService    跟踪可更新的 [WebView](https://so.csdn.net/so/search?q=WebView&spm=1001.2101.3001.7020) 是否处于就绪状态，并监视更新安装。  
CachedDeviceStateService    跟踪并缓存设备状态  
BinderCallsStatsService    跟踪在 [Binder](https://so.csdn.net/so/search?q=Binder&spm=1001.2101.3001.7020) 调用中花费的 cpu 时间  
LooperStatsService                    跟踪 handlers 中处理 messages 所花费的时间。  
RollbackManagerService    管理 apk 回滚  
BugreportManagerService    用于捕获 bugreport  
GpuService                    管理 Gpu 和 Gpu 驱动

关于其他服务的简单说明  
ClipboardService 粘贴板服务  
Installer 安装程序服务

DeviceIdentifiersPolicyService 注册设备标识符服务

ActivityManagerService 活动管理服务 本身并非是继承至 SystemService 而是内部的 LifeCycle

PowerManagerService 电源管理服务  
RecoverySystemService 系统恢复

DisplayManagerService 显示管理服务

PackageManagerService 包管理服务  
UriGrantsManagerService Uri 权限管理服务  
ThermalManagerService 硬件温度监测服务  
OverlayManagerService 动态替换 res 图片等服务  
UsageStatsService 应用使用统计服务  
BinderCallsStatsService，跟踪 Binder 调用的 CPU 时间消耗  
RollbackManagerService RollBack 机制服务  
BugreportManagerService 错误报告服务  
KeyChainSystemService 处理应用删除时的清理动作服务  
SchedulingPolicyService – 调度策略管理服务  
TelecomLoaderService – 用于启动通讯服务  
ConsumerIrService – 红外遥控管理服务  
DropBoxManagerService(简称 DBMS) 记录着系统关键 log 信息，主要功能用于 Debug 调试  
ContextHubSystemService – 主要是为了启动 ContextHubService  
DiskStatsService – 磁盘统计服务，只是在使用 adb shell dumpsys diskstats 时被调用  
CountryDetectorService – 国家检测服务，类 ComprehensiveCountryDetector.java，检测顺序：移动网络、位置、sim 卡的国家、手机的位置  
ContextHubSystemService – 主要是为了启动 ContextHubService  
DiskStatsService – 磁盘统计服务，只是在使用 adb shell dumpsys diskstats 时被调用  
RulesManagerService  
NetworkTimeUpdateService  
CommonTimeManagementService  
EmergencyAffordanceService – 紧急呼叫服务，从 2017 年 1 月 1 日开始，在印度地区销售的所有移动设备都必须应印度电信部门 (DoT) 的要求提供紧急呼叫按钮。为响应这些监管要求，Android 包含了 “提供紧急呼叫” 功能的参考实现，以启用 Android 设备上的紧急呼叫按钮。此功能在 Android 8.0 和更高版本中默认启用，但较早版本中必须安装相应的补丁程序。目前，该功能专门针对在印度市场销售的设备；不过，鉴于该功能在印度境外无效，因此也可以在全球范围销售的所有设备上提供。  
DreamManagerService – 屏保服务  
GraphicsStatsService – 用来汇总屏幕卡顿数据的，通过 adb shell dumpsys graphicsstats 调用查看汇总结果  
CoverageService – JaCoCo 相关服务，和 coverage 命令相对应，用于导出 JaCoCo 的输出文件  
GraphicsStatsService 渲染剖面数据服务  
PrintManagerService – 打印管理服务  
CompanionDeviceManagerService – 伙伴设备管理服务  
RestrictionsManagerService – 限制管理服务，好像是 app 可以通过 manifest 定义一些限制规则  
MediaSessionService – 多媒体会话服务，在多个进程中共享多媒体播放状态  
HdmiControlService – hdmi 服务，控制 hdmi 设备的显示  
TvInputManagerService – 电视输入服务  
MediaResourceMonitorService – 不清楚有啥用  
TvRemoteService – 电视摇控服务  
MediaRouterService – 媒体路由服务，用于给实现了 Google Cast 的设备提供远程播放、远程控制音乐和视频服务  
FingerprintService – 指纹服务  
BackgroundDexOptService – dex 后台优化服务，在 packmanager 中有调用  
PruneInstantAppsJobService  
ShortcutService – 快捷方式服务  
LauncherAppsService – Launcher 应用相关的服务  
MediaProjectionManagerService – 媒体投影服务，可用于录屏或者屏幕投射  
WearConnectivityService – 穿戴设备连接服务  
WearDisplayService – 穿戴设备显示服务  
WearLeftyService – 穿戴设备左撇子服务  
WearTimeService – 穿戴设备时间服务  
CameraServiceProxy  
MmsServiceBroker – 彩信服务  
AutofillManagerService – 自动填充服务，比如自动填充密码  
CarServiceHelperService  
SystemUIService – SystemUi 状态栏等  
StatsCompanionService 后台运行并收集指标的本地服务  
IncidentCompanionService 事件和转储的辅助服务，为要获取的错误和事件报告提供用户反馈和授权和权限管理相关  
DynamicSystemService 动态分区服务  
AccountManagerService 账号管理服务
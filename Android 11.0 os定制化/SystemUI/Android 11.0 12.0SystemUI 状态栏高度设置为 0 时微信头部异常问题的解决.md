> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124828449)

### 1. 概述

在 11.0 12.0 定制化开发中，由于产品需要全屏功能, 所以不需要 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 的状态栏所以就把状态栏高度设置为 0，这样虽然 SystemUI 的状态栏是隐藏不见了, 但是又会有新的问题出现，比如安装微信，qq 等 首页的头像好像被挡住了一部分，经过思考影响头部的只可能是 Status.java 了，所以需要分析状态栏的问题

### 2.SystemUI 状态栏高度设置为 0 时微信头部异常问题的解决的核心类

```
/frameworks/base/core/res/res/values/dimens.xml
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java

```

### 3.SystemUI 状态栏高度设置为 0 时微信头部异常问题的解决的核心功能分析

### 3.1 关于状态栏高度的调整

首先把导航栏高度修改为 1dp

```
diff --git a/frameworks/base/core/res/res/values/dimens.xml b/frameworks/base/core/res/res/values/dimens.xml
index 9886a4f..af7038e 100755 (executable)
--- a/frameworks/base/core/res/res/values/dimens.xml
+++ b/frameworks/base/core/res/res/values/dimens.xml
 <!-- Height of the status bar in portrait -->

<dimen >0dp</dimen>
<dimen >1dp</dimen><!-- Height of the status bar in landscape -->
<dimen >@dimen/status_bar_height_portrait</dimen>
<!-- Height of area above QQS where battery/time go -->

```

在 status_bar_height_portrait 中设置状态栏高度

### 3.2StatusBar.java 源码关于状态栏相关代码分析

在 makeStatusBarView 负责对状态栏布局加载

```
protected void makeStatusBarView(@Nullable RegisterStatusBarResult result) {
final Context context = mContext;
updateDisplaySize(); // populates mDisplayMetrics
updateResources();
updateTheme();
    inflateStatusBarWindow(context);
    mStatusBarWindow.setService(this);
    mStatusBarWindow.setOnTouchListener(getStatusBarWindowTouchListener());

    // TODO: Deal with the ugliness that comes from having some of the statusbar broken out
    // into fragments, but the rest here, it leaves some awkward lifecycle and whatnot.
    mNotificationPanel = mStatusBarWindow.findViewById(R.id.notification_panel);
    mStackScroller = mStatusBarWindow.findViewById(R.id.notification_stack_scroller);
    mZenController.addCallback(this);
    NotificationListContainer notifListContainer = (NotificationListContainer) mStackScroller;
    mNotificationLogger.setUpWithContainer(notifListContainer);

    mNotificationIconAreaController = SystemUIFactory.getInstance()
            .createNotificationIconAreaController(context, this, mStatusBarStateController);
    inflateShelf();
    mNotificationIconAreaController.setupShelf(mNotificationShelf);

    Dependency.get(DarkIconDispatcher.class).addDarkReceiver(mNotificationIconAreaController);
    // Allow plugins to reference DarkIconDispatcher and StatusBarStateController
    Dependency.get(PluginDependencyProvider.class)
            .allowPluginDependency(DarkIconDispatcher.class);
    Dependency.get(PluginDependencyProvider.class)
            .allowPluginDependency(StatusBarStateController.class);
    FragmentHostManager.get(mStatusBarWindow)
            .addTagListener(CollapsedStatusBarFragment.TAG, (tag, fragment) -> {
                CollapsedStatusBarFragment statusBarFragment =
                        (CollapsedStatusBarFragment) fragment;
                statusBarFragment.initNotificationIconArea(mNotificationIconAreaController);
                PhoneStatusBarView oldStatusBarView = mStatusBarView;
                mStatusBarView = (PhoneStatusBarView) fragment.getView();
                mStatusBarView.setBar(this);
                mStatusBarView.setPanel(mNotificationPanel);
                mStatusBarView.setScrimController(mScrimController);

                // CollapsedStatusBarFragment re-inflated PhoneStatusBarView and both of
                // mStatusBarView.mExpanded and mStatusBarView.mBouncerShowing are false.
                // PhoneStatusBarView's new instance will set to be gone in
                // PanelBar.updateVisibility after calling mStatusBarView.setBouncerShowing
                // that will trigger PanelBar.updateVisibility. If there is a heads up showing,
                // it needs to notify PhoneStatusBarView's new instance to update the correct
                // status by calling mNotificationPanel.notifyBarPanelExpansionChanged().
                if (mHeadsUpManager.hasPinnedHeadsUp()) {
                    mNotificationPanel.notifyBarPanelExpansionChanged();
                }
                mStatusBarView.setBouncerShowing(mBouncerShowing);
                if (oldStatusBarView != null) {
                    float fraction = oldStatusBarView.getExpansionFraction();
                    boolean expanded = oldStatusBarView.isExpanded();
                    mStatusBarView.panelExpansionChanged(fraction, expanded);
                }

                HeadsUpAppearanceController oldController = mHeadsUpAppearanceController;
                if (mHeadsUpAppearanceController != null) {
                    // This view is being recreated, let's destroy the old one
                    mHeadsUpAppearanceController.destroy();
                }
                mHeadsUpAppearanceController = new HeadsUpAppearanceController(
                        mNotificationIconAreaController, mHeadsUpManager, mStatusBarWindow);
                mHeadsUpAppearanceController.readFrom(oldController);
                mStatusBarWindow.setStatusBarView(mStatusBarView);
                updateAreThereNotifications();
                checkBarModes();
            }).getFragmentManager()
            .beginTransaction()
            .replace(R.id.status_bar_container, new CollapsedStatusBarFragment(),
                    CollapsedStatusBarFragment.TAG)
            .commit();
    mIconController = Dependency.get(StatusBarIconController.class);

    mHeadsUpManager = new HeadsUpManagerPhone(context, mStatusBarWindow, mGroupManager, this,
            mVisualStabilityManager);
    Dependency.get(ConfigurationController.class).addCallback(mHeadsUpManager);
    mHeadsUpManager.addListener(this);
    mHeadsUpManager.addListener(mNotificationPanel);
    mHeadsUpManager.addListener(mGroupManager);
    mHeadsUpManager.addListener(mGroupAlertTransferHelper);
    mHeadsUpManager.addListener(mVisualStabilityManager);
    mAmbientPulseManager.addListener(this);
    mAmbientPulseManager.addListener(mGroupManager);
    mAmbientPulseManager.addListener(mGroupAlertTransferHelper);
    mNotificationPanel.setHeadsUpManager(mHeadsUpManager);
    mGroupManager.setHeadsUpManager(mHeadsUpManager);
    mGroupAlertTransferHelper.setHeadsUpManager(mHeadsUpManager);
    mNotificationLogger.setHeadsUpManager(mHeadsUpManager);
    putComponent(HeadsUpManager.class, mHeadsUpManager);

    createNavigationBar(result);

    if (ENABLE_LOCKSCREEN_WALLPAPER) {
        mLockscreenWallpaper = new LockscreenWallpaper(mContext, this, mHandler);
    }

    mKeyguardIndicationController =
            SystemUIFactory.getInstance().createKeyguardIndicationController(mContext,
                    mStatusBarWindow.findViewById(R.id.keyguard_indication_area),
                    mStatusBarWindow.findViewById(R.id.lock_icon));
    mNotificationPanel.setKeyguardIndicationController(mKeyguardIndicationController);

    mAmbientIndicationContainer = mStatusBarWindow.findViewById(
            R.id.ambient_indication_container);

    // TODO: Find better place for this callback.
    mBatteryController.addCallback(new BatteryStateChangeCallback() {
        @Override
        public void onPowerSaveChanged(boolean isPowerSave) {
            mHandler.post(mCheckBarModes);
            if (mDozeServiceHost != null) {
                mDozeServiceHost.firePowerSaveChanged(isPowerSave);
            }
        }

        @Override
        public void onBatteryLevelChanged(int level, boolean pluggedIn, boolean charging) {
            // noop
        }
    });

    mAutoHideController = Dependency.get(AutoHideController.class);
    mAutoHideController.setStatusBar(this);

    mLightBarController = Dependency.get(LightBarController.class);

    ScrimView scrimBehind = mStatusBarWindow.findViewById(R.id.scrim_behind);
    ScrimView scrimInFront = mStatusBarWindow.findViewById(R.id.scrim_in_front);
    mScrimController = SystemUIFactory.getInstance().createScrimController(
            scrimBehind, scrimInFront, mLockscreenWallpaper,
            (state, alpha, color) -> mLightBarController.setScrimState(state, alpha, color),
            scrimsVisible -> {
                if (mStatusBarWindowController != null) {
                    mStatusBarWindowController.setScrimsVisibility(scrimsVisible);
                }
                if (mStatusBarWindow != null) {
                    mStatusBarWindow.onScrimVisibilityChanged(scrimsVisible);
                }
            }, DozeParameters.getInstance(mContext),
            mContext.getSystemService(AlarmManager.class));
    mNotificationPanel.initDependencies(this, mGroupManager, mNotificationShelf,
            mHeadsUpManager, mNotificationIconAreaController, mScrimController);
    mDozeScrimController = new DozeScrimController(DozeParameters.getInstance(context));

    BackDropView backdrop = mStatusBarWindow.findViewById(R.id.backdrop);
    mMediaManager.setup(backdrop, backdrop.findViewById(R.id.backdrop_front),
            backdrop.findViewById(R.id.backdrop_back), mScrimController, mLockscreenWallpaper);

    // Other icons
    mVolumeComponent = getComponent(VolumeComponent.class);

    mNotificationPanel.setUserSetupComplete(mUserSetup);
    if (UserManager.get(mContext).isUserSwitcherEnabled()) {
        createUserSwitcher();
    }

    mNotificationPanel.setLaunchAffordanceListener(
            mStatusBarWindow::onShowingLaunchAffordanceChanged);

    // Set up the quick settings tile panel
    View container = mStatusBarWindow.findViewById(R.id.qs_frame);
    if (container != null) {
        FragmentHostManager fragmentHostManager = FragmentHostManager.get(container);
        ExtensionFragmentListener.attachExtensonToFragment(container, QS.TAG, R.id.qs_frame,
                Dependency.get(ExtensionController.class)
                        .newExtension(QS.class)
                        .withPlugin(QS.class)
                        .withDefault(this::createDefaultQSFragment)
                        .build());
        mBrightnessMirrorController = new BrightnessMirrorController(mStatusBarWindow,
                (visible) -> {
                    mBrightnessMirrorVisible = visible;
                    updateScrimController();
                });
        fragmentHostManager.addTagListener(QS.TAG, (tag, f) -> {
            QS qs = (QS) f;
            if (qs instanceof QSFragment) {
                mQSPanel = ((QSFragment) qs).getQsPanel();
                mQSPanel.setBrightnessMirror(mBrightnessMirrorController);
                mFooter = ((QSFragment) qs).getFooter();
            }
        });
    }

    mReportRejectedTouch = mStatusBarWindow.findViewById(R.id.report_rejected_touch);
    if (mReportRejectedTouch != null) {
        updateReportRejectedTouchVisibility();
        mReportRejectedTouch.setOnClickListener(v -> {
            Uri session = mFalsingManager.reportRejectedTouch();
            if (session == null) { return; }

            StringWriter message = new StringWriter();
            message.write("Build info: ");
            message.write(SystemProperties.get("ro.build.description"));
            message.write("\nSerial number: ");
            message.write(SystemProperties.get("ro.serialno"));
            message.write("\n");

            PrintWriter falsingPw = new PrintWriter(message);
            FalsingLog.dump(falsingPw);
            falsingPw.flush();

            startActivityDismissingKeyguard(Intent.createChooser(new Intent(Intent.ACTION_SEND)
                            .setType("*/*")
                            .putExtra(Intent.EXTRA_SUBJECT, "Rejected touch report")
                            .putExtra(Intent.EXTRA_STREAM, session)
                            .putExtra(Intent.EXTRA_TEXT, message.toString()),
                    "Share rejected touch report")
                            .addFlags(Intent.FLAG_ACTIVITY_NEW_TASK),
                    true /* onlyProvisioned */, true /* dismissShade */);
        });
    }

    PowerManager pm = (PowerManager) mContext.getSystemService(Context.POWER_SERVICE);
    if (!pm.isScreenOn()) {
        mBroadcastReceiver.onReceive(mContext, new Intent(Intent.ACTION_SCREEN_OFF));
    }
    mGestureWakeLock = pm.newWakeLock(PowerManager.SCREEN_BRIGHT_WAKE_LOCK,
            "GestureWakeLock");
    mVibrator = mContext.getSystemService(Vibrator.class);
    int[] pattern = mContext.getResources().getIntArray(
            R.array.config_cameraLaunchGestureVibePattern);
    mCameraLaunchGestureVibePattern = new long[pattern.length];
    for (int i = 0; i < pattern.length; i++) {
        mCameraLaunchGestureVibePattern[i] = pattern[i];
    }

    // receive broadcasts
    IntentFilter filter = new IntentFilter();
    filter.addAction(Intent.ACTION_CLOSE_SYSTEM_DIALOGS);
    filter.addAction(Intent.ACTION_SCREEN_OFF);
    filter.addAction(DevicePolicyManager.ACTION_SHOW_DEVICE_MONITORING_DIALOG);
    /* UNISOC: Bug 1074234, 885650, Super power feature @{ */
    if(SprdPowerManagerUtil.SUPPORT_SUPER_POWER_SAVE){
        filter.addAction(SprdPowerManagerUtil.ACTION_POWEREX_SAVE_MODE_CHANGED);
    }
    /* @} */
    context.registerReceiverAsUser(mBroadcastReceiver, UserHandle.ALL, filter, null, null);

    IntentFilter demoFilter = new IntentFilter();
    if (DEBUG_MEDIA_FAKE_ARTWORK) {
        demoFilter.addAction(ACTION_FAKE_ARTWORK);
    }
    demoFilter.addAction(ACTION_DEMO);
    context.registerReceiverAsUser(mDemoReceiver, UserHandle.ALL, demoFilter,
            android.Manifest.permission.DUMP, null);

    // listen for USER_SETUP_COMPLETE setting (per-user)
    mDeviceProvisionedController.addCallback(mUserSetupObserver);
    mUserSetupObserver.onUserSetupChanged();

    // disable profiling bars, since they overlap and clutter the output on app windows
    ThreadedRenderer.overrideProperty("disableProfileBars", "true");

    // Private API call to make the shadows look better for Recents
    ThreadedRenderer.overrideProperty("ambientRatio", String.valueOf(1.5f));

    /* UNISOC: Bug 1074234, 885650, Super power feature @{ */
    if(SprdPowerManagerUtil.SUPPORT_SUPER_POWER_SAVE && mNotificationPanel != null){
        int mode = SprdPowerManagerUtil.getPowerSaveModeInternal();
        if(mFooter != null){
            mFooter.setPowerSaveMode(mode);
        }
        if(mQSPanel != null){
            mQSPanel.setPowerSaveMode(mode);
        }
        if (mNotificationPanel != null){
            mNotificationPanel.setPowerSaveMode(mode);
        }
        /* UNISOC: Modify for bug974161 {@ */
        mNavigationBarController.setPowerSaveMode(mode);
        /* @} */
    }
    /* @} */
    /* UNISOC: Bug 1104465 add for screen pinning @{ */
    mScreenPinningNotify = new ScreenPinningNotify(mContext);
    /* @} */
}

```

在 makeStatusBarView 来初始化一切状态栏导航栏相关的任务而在  
FragmentHostManager.get(mStatusBarWindow).addTagListener  
当重新加载状态栏和状态栏有变化都会回调这里，在导航栏设置为隐藏来实现功能  
所以修改如下:

```
diff --git a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java b/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java
index 776f872…28bc07b 100755 (executable)
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java
@@ -907,6 +907,14 @@ public class StatusBar extends SystemUI implements DemoMode,
statusBarFragment.initNotificationIconArea(mNotificationIconAreaController);
PhoneStatusBarView oldStatusBarView = mStatusBarView;
mStatusBarView = (PhoneStatusBarView) fragment.getView();
               //add code start

                   if(mStatusBarView!=null){

                      if(mStatusBarView.getVisibility()==View.VISIBLE)

                      mStatusBarView.setVisibility(View.GONE);

                   }

               }

               //add code end

               mStatusBarView.setBar(this);
               mStatusBarView.setPanel(mNotificationPanel);
               mStatusBarView.setScrimController(mScrimController);

```
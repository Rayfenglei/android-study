> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/128069297)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 系统 Settings 去掉开发者模式功能的核心类](#t1)

[3. 系统 Settings 去掉开发者模式功能的核心功能实现和分析](#t2)

 [3.1 SettingsActivity.java 关于去掉开发者模式选项的二级菜单显示](#t3)

[3.2 BuildNumberPreferenceController.java 关于去掉点击打开开发者模式的功能实现](#t4)

1. 概述
-----

  在系统产品开发中，原生系统中，对于可以点击版本号 5 次就可以打开开发者模式功能，在产品开发中由于产品开发需要不让用户打开开发者模式功能，所以要求去掉开发者模式的功能，这部分就需要在系统 Settings 中找到相关的功能点去掉就可以了

2. 系统 Settings 去掉开发者模式功能的核心类
----------------------------

```
packages\apps\Settings\src\com\android\settings\SettingsActivity.java
packages\apps\Settings\src\com\android\settings\deviceinfo\BuildNumberPreferenceController.java
```

3. 系统 Settings 去掉开发者模式功能的核心功能实现和分析
----------------------------------

  3.1 SettingsActivity.java 关于去掉开发者模式选项的二级菜单显示
----------------------------------------------

```
    @Override
    protected void onResume() {
        super.onResume();
 
        mDevelopmentSettingsListener = new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                updateTilesList();
            }
        };
        LocalBroadcastManager.getInstance(this).registerReceiver(mDevelopmentSettingsListener,
                new IntentFilter(DevelopmentSettingsEnabler.DEVELOPMENT_SETTINGS_CHANGED_ACTION));
 
        registerReceiver(mBatteryInfoReceiver, new IntentFilter(Intent.ACTION_BATTERY_CHANGED));
 
        updateTilesList();
    }
```

通过在 SettingsActivity.java 中的相关源码发现，在这里注册了监听开发者模式的广播，在收到打开开发者模式的  
广播时，会调用 updateTilesList(); 来更新菜单栏关于开发者模式的菜单项  
接下来分析下 updateTilesList(); 中有关开发者选项的相关代码

```
private void updateTilesList() {
        // Generally the items that are will be changing from these updates will
        // not be in the top list of tiles, so run it in the background and the
        // SettingsBaseActivity will pick up on the updates automatically.
        AsyncTask.execute(() -> doUpdateTilesList());
    }
 
    private void doUpdateTilesList() {
        PackageManager pm = getPackageManager();
        final UserManager um = UserManager.get(this);
        final boolean isAdmin = um.isAdminUser();
        boolean somethingChanged = false;
        final String packageName = getPackageName();
        final StringBuilder changedList = new StringBuilder();
        somethingChanged = setTileEnabled(changedList,
                new ComponentName(packageName, WifiSettingsActivity.class.getName()),
                pm.hasSystemFeature(PackageManager.FEATURE_WIFI), isAdmin) || somethingChanged;
 
        somethingChanged = setTileEnabled(changedList, new ComponentName(packageName,
                        Settings.BluetoothSettingsActivity.class.getName()),
                pm.hasSystemFeature(PackageManager.FEATURE_BLUETOOTH), isAdmin)
                || somethingChanged;
 
        // Enable DataUsageSummaryActivity if the data plan feature flag is turned on otherwise
        // enable DataPlanUsageSummaryActivity.
        somethingChanged = setTileEnabled(changedList,
                new ComponentName(packageName, Settings.DataUsageSummaryActivity.class.getName()),
                Utils.isBandwidthControlEnabled() /* enabled */,
                isAdmin) || somethingChanged;
 
        somethingChanged = setTileEnabled(changedList,
                new ComponentName(packageName,
                        Settings.ConnectedDeviceDashboardActivity.class.getName()),
                !UserManager.isDeviceInDemoMode(this) /* enabled */,
                isAdmin) || somethingChanged;
 
        somethingChanged = setTileEnabled(changedList, new ComponentName(packageName,
                        Settings.PowerUsageSummaryActivity.class.getName()),
                mBatteryPresent, isAdmin) || somethingChanged;
 
        somethingChanged = setTileEnabled(changedList, new ComponentName(packageName,
                        Settings.DataUsageSummaryActivity.class.getName()),
                Utils.isBandwidthControlEnabled(), isAdmin)
                || somethingChanged;
 
        somethingChanged = setTileEnabled(changedList, new ComponentName(packageName,
                        Settings.UserSettingsActivity.class.getName()),
                UserHandle.MU_ENABLED && UserManager.supportsMultipleUsers()
                        && !Utils.isMonkeyRunning(), isAdmin)
                || somethingChanged;
 
        final boolean showDev = DevelopmentSettingsEnabler.isDevelopmentSettingsEnabled(this)
                && !Utils.isMonkeyRunning();
        somethingChanged = setTileEnabled(changedList, new ComponentName(packageName,
                        Settings.DevelopmentSettingsDashboardActivity.class.getName()),
                showDev, isAdmin)
                || somethingChanged;
 
        somethingChanged = setTileEnabled(changedList, new ComponentName(packageName,
                        Settings.WifiDisplaySettingsActivity.class.getName()),
                WifiDisplaySettings.isAvailable(this), isAdmin)
                || somethingChanged;
 
        if (UserHandle.MU_ENABLED && !isAdmin) {
            // When on restricted users, disable all extra categories (but only the settings ones).
            final List<DashboardCategory> categories = mDashboardFeatureProvider.getAllCategories();
            synchronized (categories) {
                for (DashboardCategory category : categories) {
                    final int tileCount = category.getTilesCount();
                    for (int i = 0; i < tileCount; i++) {
                        final ComponentName component = category.getTile(i)
                                .getIntent().getComponent();
                        final String name = component.getClassName();
                        final boolean isEnabledForRestricted = ArrayUtils.contains(
                                SettingsGateway.SETTINGS_FOR_RESTRICTED, name);
                        if (packageName.equals(component.getPackageName())
                                && !isEnabledForRestricted) {
                            somethingChanged =
                                    setTileEnabled(changedList, component, false, isAdmin)
                                            || somethingChanged;
                        }
                    }
                }
            }
        }
 
        // Final step, refresh categories.
        if (somethingChanged) {
            Log.d(LOG_TAG, "Enabled state changed for some tiles, reloading all categories "
                    + changedList.toString());
            updateCategories();
        } else {
            Log.d(LOG_TAG, "No enabled state changed, skipping updateCategory call");
        }
    }
```

在 SettingsActivity.java 中的 doUpdateTilesList() 的代码中，都是关于蓝牙 wifi 开发者模式改变后  
对于系统 Settings 菜单项更新的相关配置，而在代码中  
        final boolean showDev = DevelopmentSettingsEnabler.isDevelopmentSettingsEnabled(this)  
                && !Utils.isMonkeyRunning();  
        somethingChanged = setTileEnabled(changedList, new ComponentName(packageName,  
                        Settings.DevelopmentSettingsDashboardActivity.class.getName()),  
                showDev, isAdmin)  
                || somethingChanged;  
就是关于开发者选项判断是否打开和关闭开发者模式的相关更新代码，而 showDev 变量关系这  
是否显示和隐藏开发者选项的 想要去掉开发者选项就把这个值改为 false 就好了 具体修改如下:  
        final boolean showDev = false/*DevelopmentSettingsEnabler.isDevelopmentSettingsEnabled(this)  
                && !Utils.isMonkeyRunning()*/;

3.2 BuildNumberPreferenceController.java 关于去掉点击打开开发者模式的功能实现
-----------------------------------------------------------

  在关于设备的 my_device_info.xml 的布局文件中，通过代码发现在版本号的连接 Controller 中  
是 BuildNumberPreferenceController.java，就是说开发者模式是由 BuildNumberPreferenceController.java 来负责  
管理的 ，接下来分析 BuildNumberPreferenceController.java 就可以了

```
   @Override
    public boolean handlePreferenceTreeClick(Preference preference) {
        /*if (!TextUtils.equals(preference.getKey(), getPreferenceKey())) {
            return false;
        }
        if (Utils.isMonkeyRunning()) {
            return false;
        }
        // Don't enable developer options for secondary non-demo users.
        if (!(mUm.isAdminUser() || mUm.isDemoUser())) {
            mMetricsFeatureProvider.action(
                    mContext, SettingsEnums.ACTION_SETTINGS_BUILD_NUMBER_PREF);
            return false;
        }
        // Don't enable developer options until device has been provisioned
        if (!WizardManagerHelper.isDeviceProvisioned(mContext)) {
            mMetricsFeatureProvider.action(
                    mContext, SettingsEnums.ACTION_SETTINGS_BUILD_NUMBER_PREF);
            return false;
        }
 
        if (mUm.hasUserRestriction(UserManager.DISALLOW_DEBUGGING_FEATURES)) {
            if (mUm.isDemoUser()) {
                // Route to demo device owner to lift the debugging restriction.
                final ComponentName componentName = Utils.getDeviceOwnerComponent(mContext);
                if (componentName != null) {
                    final Intent requestDebugFeatures = new Intent()
                            .setPackage(componentName.getPackageName())
                            .setAction("com.android.settings.action.REQUEST_DEBUG_FEATURES");
                    final ResolveInfo resolveInfo = mContext.getPackageManager().resolveActivity(
                            requestDebugFeatures, 0);
                    if (resolveInfo != null) {
                        mContext.startActivity(requestDebugFeatures);
                        return false;
                    }
                }
            }
            if (mDebuggingFeaturesDisallowedAdmin != null &&
                    !mDebuggingFeaturesDisallowedBySystem) {
                RestrictedLockUtils.sendShowAdminSupportDetailsIntent(mContext,
                        mDebuggingFeaturesDisallowedAdmin);
            }
            mMetricsFeatureProvider.action(
                    mContext, SettingsEnums.ACTION_SETTINGS_BUILD_NUMBER_PREF);
            return false;
        }
 
        if (mDevHitCountdown > 0) {
            mDevHitCountdown--;
            if (mDevHitCountdown == 0 && !mProcessingLastDevHit) {
                // Add 1 count back, then start password confirmation flow.
                mDevHitCountdown++;
                final ChooseLockSettingsHelper helper =
                        new ChooseLockSettingsHelper(mActivity, mFragment);
                mProcessingLastDevHit = helper.launchConfirmationActivity(
                        REQUEST_CONFIRM_PASSWORD_FOR_DEV_PREF,
                        mContext.getString(R.string.unlock_set_unlock_launch_picker_title));
                if (!mProcessingLastDevHit) {
                    enableDevelopmentSettings();
                }
                mMetricsFeatureProvider.action(
                        mMetricsFeatureProvider.getAttribution(mActivity),
                        MetricsEvent.FIELD_SETTINGS_BUILD_NUMBER_DEVELOPER_MODE_ENABLED,
                        mFragment.getMetricsCategory(),
                        null,
                        mProcessingLastDevHit ? 0 : 1);
            } else if (mDevHitCountdown > 0
                    && mDevHitCountdown < (TAPS_TO_BE_A_DEVELOPER - 2)) {
                if (mDevHitToast != null) {
                    mDevHitToast.cancel();
                }
                mDevHitToast = Toast.makeText(mContext,
                        mContext.getResources().getQuantityString(
                                R.plurals.show_dev_countdown, mDevHitCountdown,
                                mDevHitCountdown),
                        Toast.LENGTH_SHORT);
                mDevHitToast.show();
            }
 
            mMetricsFeatureProvider.action(
                    mMetricsFeatureProvider.getAttribution(mActivity),
                    MetricsEvent.FIELD_SETTINGS_BUILD_NUMBER_DEVELOPER_MODE_ENABLED,
                    mFragment.getMetricsCategory(),
                    null,
                    0);
        } else if (mDevHitCountdown < 0) {
            if (mDevHitToast != null) {
                mDevHitToast.cancel();
            }
            mDevHitToast = Toast.makeText(mContext, R.string.show_dev_already,
                    Toast.LENGTH_LONG);
            mDevHitToast.show();
            mMetricsFeatureProvider.action(
                    mMetricsFeatureProvider.getAttribution(mActivity),
                    MetricsEvent.FIELD_SETTINGS_BUILD_NUMBER_DEVELOPER_MODE_ENABLED,
                    mFragment.getMetricsCategory(),
                    null,
                    1);
        }*/
        return true;
    }
```

在 BuildNumberPreferenceController.java 的 handlePreferenceTreeClick(Preference preference) 中会判断点击次数 当点击次数超过  
5 次时，就会打开开发者模式，所以就需要注释掉这里的所有代码就可以了
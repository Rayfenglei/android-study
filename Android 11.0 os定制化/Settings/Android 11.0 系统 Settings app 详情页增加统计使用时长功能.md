> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127737851)

1. 概述
-----

 在系统产品开发中，在 app 详情页展示着权限，使用缓存数据等数据，由于产品需求需要在 app  
详情页增加 app 使用时长功能的需求来查看 app 使用情况的功能，所以就需要统计 app 使用的时间了  
来实现这个功能了

2. 系统 Settings app 详情页增加统计使用时长功能的核心类
------------------------------------

```
packages\apps\Settings\src\com\android\settings\applications\appinfo\AppInfoDashboardFragment.java
packages\apps\Settings\res\xml\app_info_settings.xml
```

3. 系统 Settings app 详情页增加统计使用时长功能的核心功能分析和实现
------------------------------------------

  3.1 AppInfoDashboardFragment 关于 app 详情页的分析
--------------------------------------------

```
  public class AppInfoDashboardFragment extends DashboardFragment
        implements ApplicationsState.Callbacks,
        ButtonActionDialogFragment.AppButtonsDialogListener {
@Override
    public void onAttach(Context context) {
        super.onAttach(context);
        final String packageName = getPackageName();
        final TimeSpentInAppPreferenceController timeSpentInAppPreferenceController = use(
                TimeSpentInAppPreferenceController.class);
        timeSpentInAppPreferenceController.setPackageName(packageName);
        timeSpentInAppPreferenceController.initLifeCycleOwner(this);
 
        use(AppDataUsagePreferenceController.class).setParentFragment(this);
        final AppInstallerInfoPreferenceController installer =
                use(AppInstallerInfoPreferenceController.class);
        installer.setPackageName(packageName);
        installer.setParentFragment(this);
        use(AppInstallerPreferenceCategoryController.class).setChildren(Arrays.asList(installer));
        use(AppNotificationPreferenceController.class).setParentFragment(this);
 
        use(AppOpenByDefaultPreferenceController.class)
                .setPackageName(packageName)
                .setParentFragment(this);
 
        use(AppPermissionPreferenceController.class).setParentFragment(this);
        use(AppPermissionPreferenceController.class).setPackageName(packageName);
        use(AppSettingPreferenceController.class)
                .setPackageName(packageName)
                .setParentFragment(this);
        use(AppStoragePreferenceController.class).setParentFragment(this);
        use(AppVersionPreferenceController.class).setParentFragment(this);
        use(InstantAppDomainsPreferenceController.class).setParentFragment(this);
 
        final WriteSystemSettingsPreferenceController writeSystemSettings =
                use(WriteSystemSettingsPreferenceController.class);
        writeSystemSettings.setParentFragment(this);
 
        final DrawOverlayDetailPreferenceController drawOverlay =
                use(DrawOverlayDetailPreferenceController.class);
        drawOverlay.setParentFragment(this);
 
        final PictureInPictureDetailPreferenceController pip =
                use(PictureInPictureDetailPreferenceController.class);
        pip.setPackageName(packageName);
        pip.setParentFragment(this);
 
        final ExternalSourceDetailPreferenceController externalSource =
                use(ExternalSourceDetailPreferenceController.class);
        externalSource.setPackageName(packageName);
        externalSource.setParentFragment(this);
 
        final InteractAcrossProfilesDetailsPreferenceController acrossProfiles =
                use(InteractAcrossProfilesDetailsPreferenceController.class);
        acrossProfiles.setPackageName(packageName);
        acrossProfiles.setParentFragment(this);
 
        use(AdvancedAppInfoPreferenceCategoryController.class).setChildren(Arrays.asList(
                writeSystemSettings, drawOverlay, pip, externalSource, acrossProfiles));
    }
```

在 AppInfoDashboardFragment 的 onAttach 中主要是绑定页面的布局，各种 Controller 类的  
初始化 父类的 api 的初始化等功能实现

```
  @Override
    public void onCreate(Bundle icicle) {
        super.onCreate(icicle);
        mFinishing = false;
        final Activity activity = getActivity();
        mDpm = (DevicePolicyManager) activity.getSystemService(Context.DEVICE_POLICY_SERVICE);
        mUserManager = (UserManager) activity.getSystemService(Context.USER_SERVICE);
        mPm = activity.getPackageManager();
        if (!ensurePackageInfoAvailable(activity)) {
            return;
        }
        if (!ensureDisplayableModule(activity)) {
            return;
        }
        startListeningToPackageRemove();
 
        setHasOptionsMenu(true);
    }
```

在 AppInfoDashboardFragment 的 onCreate(Bundle icicle) 中实现对 mDpm mUserManager mPm 的初始化

```
@Override
    public void onCreatePreferences(Bundle savedInstanceState, String rootKey) {
        if (!ensurePackageInfoAvailable(getActivity())) {
            return;
        }
        super.onCreatePreferences(savedInstanceState, rootKey);
    // add core start 
        PreferenceScreen preferenceScreen = getPreferenceScreen();
        Preference uerstimers_pre = preferenceScreen.findPreference("app_usedtimes");
        uerstimers_pre.setSummary(getappusedtimes());
   // add core end
    }
 
    @Override
    public void onDestroy() {
        stopListeningToPackageRemove();
        super.onDestroy();
    }
 
    @Override
    public int getMetricsCategory() {
        return SettingsEnums.APPLICATIONS_INSTALLED_APP_DETAILS;
    }
 
    @Override
    public void onResume() {
        super.onResume();
        final Activity activity = getActivity();
        mAppsControlDisallowedAdmin = RestrictedLockUtilsInternal.checkIfRestrictionEnforced(
                activity, UserManager.DISALLOW_APPS_CONTROL, mUserId);
        mAppsControlDisallowedBySystem = RestrictedLockUtilsInternal.hasBaseUserRestriction(
                activity, UserManager.DISALLOW_APPS_CONTROL, mUserId);
 
        if (!refreshUi()) {
            setIntentAndFinish(true, true);
        }
    }
 
    @Override
    protected int getPreferenceScreenResId() {
        return R.xml.app_info_settings;
    }
 
    @Override
    protected String getLogTag() {
        return TAG;
    }
 
    @Override
    protected List<AbstractPreferenceController> createPreferenceControllers(Context context) {
        retrieveAppEntry();
        if (mPackageInfo == null) {
            return null;
        }
        final String packageName = getPackageName();
        final List<AbstractPreferenceController> controllers = new ArrayList<>();
        final Lifecycle lifecycle = getSettingsLifecycle();
 
        // The following are controllers for preferences that needs to refresh the preference state
        // when app state changes.
        controllers.add(
                new AppHeaderViewPreferenceController(context, this, packageName, lifecycle));
 
        for (AbstractPreferenceController controller : controllers) {
            mCallbacks.add((Callback) controller);
        }
 
        // The following are controllers for preferences that don't need to refresh the preference
        // state when app state changes.
        mInstantAppButtonPreferenceController =
                new InstantAppButtonsPreferenceController(context, this, packageName, lifecycle);
        controllers.add(mInstantAppButtonPreferenceController);
        mAppButtonsPreferenceController = new AppButtonsPreferenceController(
                (SettingsActivity) getActivity(), this, lifecycle, packageName, mState,
                REQUEST_UNINSTALL, REQUEST_REMOVE_DEVICE_ADMIN);
        controllers.add(mAppButtonsPreferenceController);
        controllers.add(new AppBatteryPreferenceController(context, this, packageName, lifecycle));
        controllers.add(new AppMemoryPreferenceController(context, this, lifecycle));
        controllers.add(new DefaultHomeShortcutPreferenceController(context, packageName));
        controllers.add(new DefaultBrowserShortcutPreferenceController(context, packageName));
        controllers.add(new DefaultPhoneShortcutPreferenceController(context, packageName));
        controllers.add(new DefaultEmergencyShortcutPreferenceController(context, packageName));
        controllers.add(new DefaultSmsShortcutPreferenceController(context, packageName));
 
        return controllers;
    }
```

在上述的方法中可以看出是在 onCreatePreferences(Bundle savedInstanceState, String rootKey) 中初始化 xml 布局中的 key 键值，在 createPreferenceControllers 中添加每个 key 键值的相对应的 Controllers 的类，所以在 onCreatePreferences(）中增加 app 使用时长的 key 布局  
增加 app 使用时长的统计

```
private String getappusedtimes(){
		String usedtimes = "";
        UsageStatsManager usageStatsManager = (UsageStatsManager) getSystemService(Context.USAGE_STATS_SERVICE);
		try {
            Calendar calendar = Calendar.getInstance();
            calendar.set(Calendar.HOUR_OF_DAY, 0);  
            calendar.set(Calendar.MINUTE, 0);
            calendar.set(Calendar.SECOND, 0);
            List<UsageStats> stats =usageStatsManager.queryUsageStats(UsageStatsManager.INTERVAL_DAILY,
                    calendar.getTimeInMillis(), System.currentTimeMillis());
            for(int i=0;i<stats.size();i++){
                final android.app.usage.UsageStats pkgStats = stats.get(i);
                String package_name=pkgStats.getPackageName();
                long time_begin= pkgStats.getFirstTimeStamp();
                long time_end=pkgStats.getLastTimeStamp();
                long time_used=pkgStats.getLastTimeUsed();
                long time_totals=pkgStats.getTotalTimeInForeground();
                if(time_total>0&&package_name.equals(getPackageName())) {
		Log.e("AppInfoDashboardFragment","package_name:"+package_name+"--time_begin:"+time_begin+"--time_used:"+time_used+"--time_total:"+time_total);
		return time_totals/1000+"s";
	}
            }
        } catch (Exception e) {
            e.printStackTrace();
        } 
       return usedtimes;
}
```

3.2 在 app_info_settings.xml 中增加 app 使用时间统计的 key 值
-------------------------------------------------

```
 <PreferenceScreen
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:settings="http://schemas.android.com/apk/res-auto"
    android:key="installed_app_detail_settings_screen"
    settings:initialExpandedChildrenCount="6">
 
    <com.android.settingslib.widget.LayoutPreference
        android:key="header_view"
        android:layout="@layout/settings_entity_header"
        android:selectable="false"
        android:order="-10000"
        settings:allowDividerBelow="true"/>
 
    <com.android.settingslib.widget.LayoutPreference
        android:key="instant_app_buttons"
        android:layout="@layout/instant_app_buttons"
        android:selectable="false"
        android:order="-9999"
        settings:allowDividerAbove="true"
        settings:allowDividerBelow="true"/>
 
    <com.android.settingslib.widget.ActionButtonsPreference
        android:key="action_buttons"
        android:order="-9998" />
 
    <Preference
        android:key="notification_settings"
        android:title="@string/notifications_label"
        settings:controller="com.android.settings.applications.appinfo.AppNotificationPreferenceController"
        settings:allowDividerAbove="true"/>
 
    <com.android.settings.widget.FixedLineSummaryPreference
        android:key="permission_settings"
        android:title="@string/permissions_label"
        android:summary="@string/summary_placeholder"
        settings:summaryLineCount="1"
        settings:controller="com.android.settings.applications.appinfo.AppPermissionPreferenceController" />
 
    <Preference
        android:key="storage_settings"
        android:title="@string/storage_settings_for_app"
        android:summary="@string/summary_placeholder"
        settings:controller="com.android.settings.applications.appinfo.AppStoragePreferenceController" />
 
    <Preference
        android:key="app_usedtimes"
        android:title="@string/app_used_time"
        android:summary="@string/summary_placeholder" />
```

在 app_info_settings.xml 中增加 Preference 控件 key 的名称就是 app_usedtimes title 名称就是  
app_used_time, 接下来增加 app_used_time 的资源

```
--- a/packages/apps/Settings/res/values-zh-rCN/strings.xml
+++ b/packages/apps/Settings/res/values-zh-rCN/strings.xml
@@ -21,6 +21,7 @@
 
     <string </string>
+    <string >最近使用时间</string>
     <string </string>
     <plurals >
```
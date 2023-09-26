> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124889365)

在 11.0 12.0 定制化开发 Settings 时, 客户对主页设置项需要动态控制显示和隐藏，这就需要用两个页面来区分加载不同 settings 页面  
实现思路:  
1. 用系统变量控制显示和隐藏某些项  
2. 增加一个自定义页面来适配不同页面

原理分析

从启动开始说起

进入 setting 的 AndroidManifest.xml 里看一看，找启动 Activity

```
 <!-- Alias for launcher activity only, as this belongs to each profile. -->
        <activity-alias android:
                android:label="@string/settings_label_launcher"
                android:launchMode="singleTask"
                android:targetActivity=".homepage.SettingsHomepageActivity">
            <intent-filter>
                <action android: />
                <category android: />
                <category android: />
            </intent-filter>
            <meta-data android:@xml/shortcuts"/>
        </activity-alias>

```

发现启动 Activity 是 Settings, 但是前面的标签是 activity-alias，所以这是另一个 Activity 的别名，然后它真实的启动 Activity 应该是 targetActivity 所标注的 SettingsHomepageActivity。

走进 SettingsHomepageActivity.java

```
packages\apps\Settings\src\com\android\settings\homepage\SettingsHomepageActivity.java

public class SettingsHomepageActivity extends FragmentActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        setContentView(R.layout.settings_homepage_container);
        final View root = findViewById(R.id.settings_homepage_container);
        root.setSystemUiVisibility(
                View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION | View.SYSTEM_UI_FLAG_LAYOUT_STABLE);

        
        setHomepageContainerPaddingTop();

        final Toolbar toolbar = findViewById(R.id.search_action_bar);
        FeatureFactory.getFactory(this).getSearchFeatureProvider()
                .initSearchToolbar(this /* activity */, toolbar, SettingsEnums.SETTINGS_HOMEPAGE);

        final ImageView avatarView = findViewById(R.id.account_avatar);
        final AvatarViewMixin avatarViewMixin = new AvatarViewMixin(this, avatarView);
        getLifecycle().addObserver(avatarViewMixin);

        if (!getSystemService(ActivityManager.class).isLowRamDevice()) {
            // Only allow contextual feature on high ram devices.
            showFragment(new ContextualCardsFragment(), R.id.contextual_cards_content);
        }
        showFragment(new TopLevelSettings(), R.id.main_content);
        ((FrameLayout) findViewById(R.id.main_content))
                .getLayoutTransition().enableTransitionType(LayoutTransition.CHANGING);
    }

    private void showFragment(Fragment fragment, int id) {
        final FragmentManager fragmentManager = getSupportFragmentManager();
        final FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
        final Fragment showFragment = fragmentManager.findFragmentById(id);

        if (showFragment == null) {
            fragmentTransaction.add(id, fragment);
        } else {
            fragmentTransaction.show(showFragment);
        }
        fragmentTransaction.commit();
    }

    @VisibleForTesting
    void setHomepageContainerPaddingTop() {
        final View view = this.findViewById(R.id.homepage_container);

        final int searchBarHeight = getResources().getDimensionPixelSize(R.dimen.search_bar_height);
        final int searchBarMargin = getResources().getDimensionPixelSize(R.dimen.search_bar_margin);

        // The top padding is the height of action bar(48dp) + top/bottom margins(16dp)
        final int paddingTop = searchBarHeight + searchBarMargin * 2;
        view.setPadding(0 /* left */, paddingTop, 0 /* right */, 0 /* bottom */);
    }
}

```

代码不多，布局文件对应 settings_homepage_container.xml, 布局加载完成后增加顶部 padding 为了给 SearchActionBar 预留空间，

如果不需要 SeacherActionBar 直接将这部分代码注释即可。接下来看到新创建 TopLevelSettings 填充 main_content，主角登场啦。

TopLevelSettings 就是我们看到的 Settings 主界面。

进入 TopLevelSettings

```
packages\apps\Settings\src\com\android\settings\homepage\TopLevelSettings.java

public class TopLevelSettings extends DashboardFragment implements
        PreferenceFragmentCompat.OnPreferenceStartFragmentCallback {

    private static final String TAG = "TopLevelSettings";

    public TopLevelSettings() {
        final Bundle args = new Bundle();
        // Disable the search icon because this page uses a full search view in actionbar.
        args.putBoolean(NEED_SEARCH_ICON_IN_ACTION_BAR, false);
        setArguments(args);
    }

    @Override
    protected int getPreferenceScreenResId() {
        return R.xml.top_level_settings;
    }

```

top_level_settings.xml

packages\apps\Settings\res\xml\top_level_settings.xml

```
<PreferenceScreen
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:settings="http://schemas.android.com/apk/res-auto"
    android:key="top_level_settings">

    <Preference
        android:key="top_level_network"
        android:title="@string/network_dashboard_title"
        android:summary="@string/summary_placeholder"
        android:icon="@drawable/ic_homepage_network"
        android:order="-120"
        android:fragment="com.android.settings.network.NetworkDashboardFragment"
        settings:controller="com.android.settings.network.TopLevelNetworkEntryPreferenceController"/>

    <Preference
        android:key="top_level_connected_devices"
        android:title="@string/connected_devices_dashboard_title"
        android:summary="@string/summary_placeholder"
        android:icon="@drawable/ic_homepage_connected_device"
        android:order="-110"
        android:fragment="com.android.settings.connecteddevice.ConnectedDeviceDashboardFragment"
        settings:controller="com.android.settings.connecteddevice.TopLevelConnectedDevicesPreferenceController"/>

    <Preference
        android:key="top_level_apps_and_notifs"
        android:title="@string/app_and_notification_dashboard_title"
        android:summary="@string/app_and_notification_dashboard_summary"
        android:icon="@drawable/ic_homepage_apps"
        android:order="-100"
        android:fragment="com.android.settings.applications.AppAndNotificationDashboardFragment"/>

```

可以看到主界面对应布局 top_level_settings.xml 中都是一个个 [Preference](https://so.csdn.net/so/search?q=Preference&spm=1001.2101.3001.7020)，也就对应了主页面每一个条目，可以看到

xml 中 Preference 数目和主界面显示数目是不对等了，为啥呢？因为存在动态添加的，查阅 TopLevelSettings 代码发现没啥

特殊而且代码量很少，看到 TopLevelSettings 继承 DashboardFragment，点进去看看

DashboardFragment.java

packages\apps\Settings\src\com\android\settings\dashboard\DashboardFragment.java

```
@Override
    public void onAttach(Context context) {
        super.onAttach(context);
        mSuppressInjectedTileKeys = Arrays.asList(context.getResources().getStringArray(
                R.array.config_suppress_injected_tile_keys));
        mDashboardFeatureProvider = FeatureFactory.getFactory(context).
                getDashboardFeatureProvider(context);
        final List<AbstractPreferenceController> controllers = new ArrayList<>();
        // Load preference controllers from code
        final List<AbstractPreferenceController> controllersFromCode =
                createPreferenceControllers(context);
        // Load preference controllers from xml definition
        final List<BasePreferenceController> controllersFromXml = PreferenceControllerListHelper
                .getPreferenceControllersFromXml(context, getPreferenceScreenResId());
        // Filter xml-based controllers in case a similar controller is created from code already.
        final List<BasePreferenceController> uniqueControllerFromXml =
                PreferenceControllerListHelper.filterControllers(
                        controllersFromXml, controllersFromCode);

        // Add unique controllers to list.
        if (controllersFromCode != null) {
            controllers.addAll(controllersFromCode);
        }
        controllers.addAll(uniqueControllerFromXml);
      
    }

```

注释已经写得很清楚了，分别从 java 代码和 xml 中加载 PreferenceController，然后过滤去重最终加载页面显示。

java 代码加载

createPreferenceControllers() return null, 而且子类 TopLevelSettings 并未覆盖实现，所以 controllersFromCode 为 null

xml 加载

getPreferenceControllersFromXml(context, getPreferenceScreenResId()), getPreferenceScreenResId 对应刚刚的 top_level_settings

具体的遍历解析 xml 文件, 然后显示出来主页面的设置项, 所以我们可以根据系统变量的值 来显示不同的页面

自定义 top_level_settings_custom.xml 注释掉不需要显示的设置项即可 如下:

```
<?xml version="1.0" encoding="utf-8"?>


<PreferenceScreen
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:settings="http://schemas.android.com/apk/res-auto"
    android:key="top_level_settings">

    <Preference
        android:key="top_level_network"
        android:title="@string/network_dashboard_title"
        android:summary="@string/summary_placeholder"
        android:icon="@drawable/ic_homepage_network"
        android:order="-120"
        android:fragment="com.android.settings.network.NetworkDashboardFragment"
        settings:controller="com.android.settings.network.TopLevelNetworkEntryPreferenceController"/>

    <!--Preference
        android:key="top_level_connected_devices"
        android:title="@string/connected_devices_dashboard_title"
        android:summary="@string/summary_placeholder"
        android:icon="@drawable/ic_homepage_connected_device"
        android:order="-110"
        android:fragment="com.android.settings.connecteddevice.ConnectedDeviceDashboardFragment"
        settings:controller="com.android.settings.connecteddevice.TopLevelConnectedDevicesPreferenceController"/-->

    <!--Preference
        android:key="top_level_apps_and_notifs"
        android:title="@string/app_and_notification_dashboard_title"
        android:summary="@string/app_and_notification_dashboard_summary"
        android:icon="@drawable/ic_homepage_apps"
        android:order="-100"
        android:fragment="com.android.settings.applications.AppAndNotificationDashboardFragment"/-->

    <!--Preference
        android:key="top_level_battery"
        android:title="@string/power_usage_summary_title"
        android:summary="@string/summary_placeholder"
        android:icon="@drawable/ic_homepage_battery"
        android:fragment="com.android.settings.fuelgauge.PowerUsageSummary"
        android:order="-90"
        settings:controller="com.android.settings.fuelgauge.TopLevelBatteryPreferenceController"/-->

    <Preference
        android:key="top_level_display"
        android:title="@string/display_settings"
        android:summary="@string/summary_placeholder"
        android:icon="@drawable/ic_homepage_display"
        android:order="-80"
        android:fragment="com.android.settings.DisplaySettings"
        settings:controller="com.android.settings.display.TopLevelDisplayPreferenceController"/>

     <!--Preference
        android:key="top_level_timerpower"
        android:title="@string/swtichmachine"
        android:summary="@string/summary_placeholder"
        android:icon="@drawable/ic_homepage_timerpower"
        android:order="-75"
        android:fragment="com.sprd.settings.timerpower.TimerPower"
        settings:controller="com.sprd.settings.timerpower.TopLevelTimerPowerPreferenceController"/-->

    <Preference
        android:key="top_level_sound"
        android:title="@string/sound_settings"
        android:summary="@string/sound_dashboard_summary"
        android:icon="@drawable/ic_homepage_sound"
        android:order="-70"
        android:fragment="com.android.settings.notification.SoundSettings"/>

    <!--Preference
        android:key="top_level_storage"
        android:title="@string/storage_settings"
        android:summary="@string/summary_placeholder"
        android:icon="@drawable/ic_homepage_storage"
        android:order="-60"
        android:fragment="com.android.settings.deviceinfo.StorageSettings"
        settings:controller="com.android.settings.deviceinfo.TopLevelStoragePreferenceController"/-->

    <!--Preference
        android:key="top_level_privacy"
        android:title="@string/privacy_dashboard_title"
        android:summary="@string/privacy_dashboard_summary"
        android:icon="@drawable/ic_homepage_privacy"
        android:order="-55"
        android:fragment="com.android.settings.privacy.PrivacyDashboardFragment"/-->

    <!--Preference
        android:key="top_level_location"
        android:title="@string/location_settings_title"
        android:summary="@string/location_settings_loading_app_permission_stats"
        android:icon="@drawable/ic_homepage_location"
        android:order="-50"
        android:fragment="com.android.settings.location.LocationSettings"
        settings:controller="com.android.settings.location.TopLevelLocationPreferenceController"/-->

    <!--Preference
        android:key="top_level_security"
        android:title="@string/security_settings_title"
        android:summary="@string/summary_placeholder"
        android:icon="@drawable/ic_homepage_security"
        android:order="-40"
        android:fragment="com.android.settings.security.SecuritySettings"
        settings:controller="com.android.settings.security.TopLevelSecurityEntryPreferenceController"/-->

    <!--Preference
        android:key="top_level_accounts"
        android:title="@string/account_dashboard_title"
        android:summary="@string/summary_placeholder"
        android:icon="@drawable/ic_homepage_accounts"
        android:order="-30"
        android:fragment="com.android.settings.accounts.AccountDashboardFragment"
        settings:controller="com.android.settings.accounts.TopLevelAccountEntryPreferenceController"/-->

    <!--Preference
        android:key="top_level_accessibility"
        android:title="@string/accessibility_settings"
        android:summary="@string/accessibility_settings_summary"
        android:icon="@drawable/ic_homepage_accessibility"
        android:order="-20"
        android:fragment="com.android.settings.accessibility.AccessibilitySettings"
        settings:controller="com.android.settings.accessibility.TopLevelAccessibilityPreferenceController"/-->

    <!-- Add for Smart Controls -->
    <!--Preference
        android:key="top_level_smartcontrols"
        android:title="@string/smart_controls"
        android:icon="@drawable/ic_settings_smart_controls"
        android:order="-10"
        android:fragment="com.sprd.settings.smartcontrols.SmartControlsSettings"
        settings:controller="com.sprd.settings.smartcontrols.TopLevelSmartControlsPreferenceController"/-->

    <Preference
        android:key="top_level_system"
        android:title="@string/header_category_system"
        android:summary="@string/summary_placeholder"
        android:icon="@drawable/ic_homepage_system_dashboard"
        android:order="10"
        android:fragment="com.android.settings.system.SystemDashboardFragment"
        settings:controller="com.android.settings.system.TopLevelSystemPreferenceController"/>

    <Preference
        android:key="top_level_about_device"
        android:title="@string/about_settings"
        android:summary="@string/summary_placeholder"
        android:icon="@drawable/ic_homepage_about"
        android:order="20"
        android:fragment="com.android.settings.deviceinfo.aboutphone.MyDeviceInfoFragment"
        settings:controller="com.android.settings.deviceinfo.aboutphone.TopLevelAboutDevicePreferenceController"/>

    <Preference
        android:key="top_level_support"
        android:summary="@string/support_summary"
        android:title="@string/page_tab_title_support"
        android:icon="@drawable/ic_homepage_support"
        android:order="100"
        settings:controller="com.android.settings.support.SupportPreferenceController"/>

</PreferenceScreen>

```

TopLevelSettings.java 修改部分如下:

```
--- a/packages/apps/Settings/src/com/android/settings/homepage/TopLevelSettings.java
+++ b/packages/apps/Settings/src/com/android/settings/homepage/TopLevelSettings.java
@@ -39,7 +39,7 @@ import com.android.settingslib.search.SearchIndexable;
 import android.provider.Settings;
 import java.util.Arrays;
 import java.util.List;
-
+import android.util.Log;
 @SearchIndexable(forTarget = MOBILE)
 public class TopLevelSettings extends DashboardFragment implements
         PreferenceFragmentCompat.OnPreferenceStartFragmentCallback {
@@ -79,7 +79,7 @@ public class TopLevelSettings extends DashboardFragment implements

 
     //重点部分就是这里，根据显示和隐藏 加载不同的页面即可
@Override
     protected int getPreferenceScreenResId() {
-        return R.xml.top_level_settings;
+               int settings_custom = Settings.Global.getInt(getContext().getContentResolver(),"settings_custom",1);
+        Log.e("Settings","getPreferenceScreenResId---settings_custom:"+settings_custom);
+        if(settings_custom==1){
+           return R.xml.top_level_settings;
+               }else{
+           return R.xml.top_level_settings_custom;
+               }
     }

```
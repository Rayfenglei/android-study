> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127351630)

1. 概述
-----

 在 11.0 的系统产品开发中，对于系统原生是有多用户功能的，但是产品开发需求要求去掉多用户功能，[systemui](https://so.csdn.net/so/search?q=systemui&spm=1001.2101.3001.7020) 和 Settings 中的多用户功能都要求去掉，所以就需要找到系统关于多用户的地方去掉多用户功能

2. 系统去掉多用户功能的核心类
----------------

```
frameworks/base/core/java/android/os/UserManager.java
framework/base/core/res/res/values/config.xml
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/KeyguardStatusBarView.java
frameworks/base/packages/SystemUI/src/com/android/systemui/qs/QSFooterImpl.java
packages/apps/Settings/AndroidManifest.xml
```

3. 系统去掉多用户功能的核心功能分析和实现
----------------------

###      3.1 UserManager.java 中关于多用户的相关方法分析

```
      public static boolean supportsMultipleUsers() {
          return getMaxSupportedUsers() > 1
                  && SystemProperties.getBoolean("fw.show_multiuserui",
                  Resources.getSystem().getBoolean(R.bool.config_enableMultiUserUI));
      }
	  @UnsupportedAppUsage
      public static int getMaxSupportedUsers() {
          // Don't allow multiple users on certain builds
          if (android.os.Build.ID.startsWith("JVP")) return 1;
          return SystemProperties.getInt("fw.max_users",
                  Resources.getSystem().getInteger(R.integer.config_multiuserMaximumUsers));
      }
```

在 UserManager.java 中 supportsMultipleUsers() 中判断是否支持多用户功能，取值主要是 config_enableMultiUserUI 的值  
决定的，而 getMaxSupportedUsers() 是获取最大的多用户数，而它的定义在 config_multiuserMaximumUsers 中  
所以接下来看这两个值的定义  
路径: framework/base/core/res/res/values/config.xml

```
<!--  Maximum number of supported users -->
<integer >1</integer>
<!--  Whether Multiuser UI should be shown -->
<bool >false</bool>
```

从 config.xml 中发现系统默认不支持多用户的

### 3.2 SystemUI 中去掉关于多用户功能的相关代码

###    3.2.1 KeyguardStatusBarView.java 锁屏页面去掉多用户图标的功能

```
   public class KeyguardStatusBarView extends RelativeLayout
         implements BatteryStateChangeCallback, OnUserInfoChangedListener, ConfigurationListener {
 
     private TextView mCarrierLabel;
     private MultiUserSwitch mMultiUserSwitch;
     public KeyguardStatusBarView(Context context, AttributeSet attrs) {
          super(context, attrs);
      }
  
      @Override
      protected void onFinishInflate() {
          super.onFinishInflate();
          mSystemIconsContainer = findViewById(R.id.system_icons_container);
          mMultiUserSwitch = findViewById(R.id.multi_user_switch);
          mMultiUserAvatar = findViewById(R.id.multi_user_avatar);
          mCarrierLabel = findViewById(R.id.keyguard_carrier_text);
          mBatteryView = mSystemIconsContainer.findViewById(R.id.battery);
          mCutoutSpace = findViewById(R.id.cutout_space_view);
          mStatusIconArea = findViewById(R.id.status_icon_area);
          mStatusIconContainer = findViewById(R.id.statusIcons);
  
          loadDimens();
          updateUserSwitcher();
          mBatteryController = Dependency.get(BatteryController.class);
      }
      private void updateVisibilities() {
          if (mMultiUserSwitch.getParent() != mStatusIconArea && !mKeyguardUserSwitcherShowing) {
              if (mMultiUserSwitch.getParent() != null) {
                  getOverlay().remove(mMultiUserSwitch);
              }
              mStatusIconArea.addView(mMultiUserSwitch, 0);
          } else if (mMultiUserSwitch.getParent() == mStatusIconArea && mKeyguardUserSwitcherShowing) {
              mStatusIconArea.removeView(mMultiUserSwitch);
          }
          if (mKeyguardUserSwitcher == null) {
              // If we have no keyguard switcher, the screen width is under 600dp. In this case,
              // we only show the multi-user switch if it's enabled through UserManager as well as
              // by the user.
              if (mMultiUserSwitch.isMultiUserEnabled()) {
                  mMultiUserSwitch.setVisibility(View.VISIBLE);
              } else {
                  mMultiUserSwitch.setVisibility(View.GONE);
              }
          }
      // add core start 
          mMultiUserSwitch.setVisibility(View.GONE);
     //add core end
          mBatteryView.setForceShowPercent(mBatteryCharging && mShowPercentAvailable);
      }
```

在 KeyguardStatusBarView 的源码中可以看出 mMultiUserSwitch 就是在锁屏界面状态栏显示的多用户图标在显示锁屏页面的时候，都会调用 updateVisibilities() 来设置状态栏相关图标的显示  
所以可以在这里隐藏掉多用户图标就可以了

### 3.2.2 QSFooterImpl.java 中关于下拉状态栏中的去掉多用户图标的功能

```
            @VisibleForTesting
      public QSFooterImpl(Context context, AttributeSet attrs) {
          this(context, attrs,
                  Dependency.get(ActivityStarter.class),
                  Dependency.get(UserInfoController.class),
                  Dependency.get(DeviceProvisionedController.class));
      }
  
      @Override
      protected void onFinishInflate() {
          super.onFinishInflate();
          mEdit = findViewById(android.R.id.edit);
          mEdit.setOnClickListener(view ->
                  mActivityStarter.postQSRunnableDismissingKeyguard(() ->
                          mQsPanel.showEdit(view)));
  
          mPageIndicator = findViewById(R.id.footer_page_indicator);
  
          mSettingsButton = findViewById(R.id.settings_button);
          mSettingsContainer = findViewById(R.id.settings_button_container);
          mSettingsButton.setOnClickListener(this);
  
          mMultiUserSwitch = findViewById(R.id.multi_user_switch);
          mMultiUserAvatar = mMultiUserSwitch.findViewById(R.id.multi_user_avatar);
  
          mActionsContainer = findViewById(R.id.qs_footer_actions_container);
          mEditContainer = findViewById(R.id.qs_footer_actions_edit_container);
          mBuildText = findViewById(R.id.build);
  
          // RenderThread is doing more harm than good when touching the header (to expand quick
          // settings), so disable it for this view
          ((RippleDrawable) mSettingsButton.getBackground()).setForceSoftware(true);
  
          updateResources();
  
          addOnLayoutChangeListener((v, left, top, right, bottom, oldLeft, oldTop, oldRight,
                  oldBottom) -> updateAnimator(right - left));
          setImportantForAccessibility(IMPORTANT_FOR_ACCESSIBILITY_YES);
          updateEverything();
          setBuildText();
      }
```

在 QSFooterImpl.java 的 onFinishInflate() 中，通过查看 qs_footer_impl.xml 的布局文件发现 mMultiUserSwitch 就是下拉状态栏的多用户图标的功能，所以在这些隐藏掉这个图标就可以了

```
   private void updateVisibilities() {
          mSettingsContainer.setVisibility(mQsDisabled ? View.GONE : View.VISIBLE);
          mSettingsContainer.findViewById(R.id.tuner_icon).setVisibility(
                  TunerService.isTunerEnabled(mContext) ? View.VISIBLE : View.INVISIBLE);
          final boolean isDemo = UserManager.isDeviceInDemoMode(mContext);
        -  mMultiUserSwitch.setVisibility(showUserSwitcher() ? View.VISIBLE : View.INVISIBLE);
        +  mMultiUserSwitch.setVisibility(View.INVISIBLE);
          mEditContainer.setVisibility(isDemo || !mExpanded ? View.INVISIBLE : View.VISIBLE);
          mSettingsButton.setVisibility(isDemo || !mExpanded ? View.INVISIBLE : View.VISIBLE);
  
          mBuildText.setVisibility(mExpanded && mShouldShowBuildText ? View.VISIBLE : View.GONE);
      }
```

在 updateVisibilities() 中就是设置 mMultiUserSwitch 的 根据 showUserSwitcher() 显示隐藏的，所以可以在这里默认设置 mMultiUserSwitch 隐藏就可以了

### 3.3 Settings 中关于去掉系统菜单下多用户功能

  在系统设置的主菜单中的系统这一项中的二级菜单有个多用户的菜单，但是在系统 SystemDashboardFragment 这个菜单的布局文件 system_dashboard_fragment.xml 中并没有发现  
关于多用户的相关 xml 定义  
  下面看下 system_dashboard_fragment.xml 的相关源码

```
<PreferenceScreen
     xmlns:android="http://schemas.android.com/apk/res/android"
     xmlns:settings="http://schemas.android.com/apk/res-auto"
     android:key="system_dashboard_screen"
     android:title="@string/header_category_system"
     settings:initialExpandedChildrenCount="4">
 
     <Preference
         android:key="language_input_settings"
         android:title="@string/language_settings"
         android:icon="@drawable/ic_settings_language"
         android:order="-260"
         android:fragment="com.android.settings.language.LanguageAndInputSettings"
         settings:controller="com.android.settings.language.LanguageAndInputPreferenceController"/>
 
     <Preference
         android:key="gesture_settings"
         android:title="@string/gesture_preference_title"
         android:icon="@drawable/ic_settings_gestures"
         android:order="-250"
         android:fragment="com.android.settings.gestures.GestureSettings"
         settings:controller="com.android.settings.gestures.GesturesSettingPreferenceController"/>
 
     <Preference
         android:key="date_time_settings"
         android:title="@string/date_and_time"
         android:icon="@drawable/ic_settings_date_time"
         android:order="-240"
         android:fragment="com.android.settings.datetime.DateTimeSettings"
         settings:controller="com.android.settings.datetime.DateTimePreferenceController"/>
 
     <Preference
         android:key="reset_dashboard"
         android:title="@string/reset_dashboard_title"
         android:summary="@string/reset_dashboard_summary"
         android:icon="@drawable/ic_restore"
         android:order="-50"
         android:fragment="com.android.settings.system.ResetDashboardFragment"
         settings:controller="com.android.settings.system.ResetPreferenceController"/>
 
     <!-- System updates -->
     <Preference
         android:key="system_update_settings"
         android:title="@string/system_update_settings_list_item_title"
         android:summary="@string/summary_placeholder"
         android:icon="@drawable/ic_system_update"
         android:order="-30"
         settings:keywords="@string/keywords_system_update_settings"
         settings:controller="com.android.settings.system.SystemUpdatePreferenceController">
         <intent android:action="android.settings.SYSTEM_UPDATE_SETTINGS"/>
     </Preference>
 
     <Preference
         android:key="additional_system_update_settings"
         android:title="@string/additional_system_update_settings_list_item_title"
         android:order="-31"
         settings:controller="com.android.settings.system.AdditionalSystemUpdatePreferenceController">
         <intent android:action="android.intent.action.MAIN"
                 android:targetPackage="@string/additional_system_update"
                 android:targetClass="@string/additional_system_update_menu"/>
     </Preference>
 
 </PreferenceScreen>
```

 从 system_dashboard_fragment.xml 的源码中发现找不到多用户的定义，而在全局搜索多用户发现  
只有在 AndroidManifest.xml 中有关于多用户的定义

```
<activity
             android:
             android:label="@string/user_settings_title"
             android:icon="@drawable/ic_settings_multiuser">
             <intent-filter android:priority="1">
                 <action android: />
                 <category android: />
             </intent-filter>
             <intent-filter>
                 <action android: />
             </intent-filter>
             <meta-data android:/>
             <meta-data android:
                        android:value="com.android.settings.category.ia.system" />
             <meta-data android:
                        android:value="content://com.android.settings.dashboard.SummaryProvider/user" />
             <meta-data android:
                        android:value="com.android.settings.users.UserSettings" />
             <meta-data android:
                        android:value="true" />
         </activity>
```

在这里发现 android:label="@string/user_settings_title" 就是多用户的定义  
所以去掉 Settings 中的 SystemDashboardFragment 这个多用户菜单就需要在这里改  
注释掉相关 meta-data 属性  
具体修改如下：

```
<activity
             android:
             android:label="@string/user_settings_title"
             android:icon="@drawable/ic_settings_multiuser">
             <intent-filter android:priority="1">
                 <action android: />
                 <category android: />
             </intent-filter>
             <intent-filter>
                 <action android: />
             </intent-filter>
             <!--meta-data android:/>
             <meta-data android:
                        android:value="com.android.settings.category.ia.system" />
             <meta-data android:
                        android:value="content://com.android.settings.dashboard.SummaryProvider/user" />
             <meta-data android:
                        android:value="com.android.settings.users.UserSettings" />
             <meta-data android:
                        android:value="true" /-->
         </activity>
```

注释掉 activity 下的所有 meta-data 属性 编译 Settings 发现 SystemDashboardFragment 这个多用户菜单不见了 就完成了功能
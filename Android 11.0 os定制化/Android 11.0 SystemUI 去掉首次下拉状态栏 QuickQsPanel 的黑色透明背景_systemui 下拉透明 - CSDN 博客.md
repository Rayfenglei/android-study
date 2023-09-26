> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124740050)

### 1. 概述

在 11.0 的原生系统产品定制化开发中，在 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 的首次下拉状态栏的 QuickQsPanel 默认会有黑色透明背景，当设置默认壁纸比较透明的时候 能够看到很清楚，显得特别不美观，所以要去掉这些黑色透明背景，以免在带透明度的背景下显示一层黑色透明

### 2.SystemUI 去掉首次下拉状态栏 QuickQsPanel 的黑色透明背景的核心类

```
frameworks/base/packages/SystemUI/res/layout/qs_panel.xml
/frameworks/base/packages/SystemUI/src/com/android/systemui/qs/QuickStatusBarHeader.java

```

### 3.SystemUI 去掉首次下拉状态栏 QuickQsPanel 的黑色透明背景核心功能分析和实现

而下拉状态栏布局文件为 qs_panel.xml

### 3.1qs_panel.xml 关于布局的相关功能分析

```
<com.android.systemui.qs.QSContainerImpl
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/quick_settings_container"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:clipToPadding="false"
    android:clipChildren="false" >

    <!-- Main QS background -->
    <View
        android:id="@+id/quick_settings_background"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:elevation="4dp"
        android:background="@drawable/qs_background_primary" />

    <!-- Black part behind the status bar -->
    <View
        android:id="@+id/quick_settings_status_bar_background"
        android:layout_width="match_parent"
        android:layout_height="@*android:dimen/quick_qs_offset_height"
        android:clipToPadding="false"
        android:clipChildren="false"
        android:background="#ff000000" />

    <!-- Gradient view behind QS -->
    <View
        android:id="@+id/quick_settings_gradient_view"
        android:layout_width="match_parent"
        android:layout_height="126dp"
        android:layout_marginTop="@*android:dimen/quick_qs_offset_height"
        android:clipToPadding="false"
        android:clipChildren="false"
        android:background="@drawable/qs_bg_gradient" />


    <com.android.systemui.qs.QSPanel
        android:id="@+id/quick_settings_panel"
        android:layout_marginTop="@*android:dimen/quick_qs_offset_height"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginBottom="@dimen/qs_footer_height"
        android:elevation="4dp"
        android:background="@android:color/transparent"
        android:focusable="true"
        android:accessibilityTraversalBefore="@android:id/edit"
    />

    <include layout="@layout/quick_status_bar_expanded_header" />

    <include layout="@layout/qs_footer_impl" />

    <include android:id="@+id/qs_detail" layout="@layout/qs_detail" />

    <include android:id="@+id/qs_customize" layout="@layout/qs_customize_panel"
        android:visibility="gone" />

</com.android.systemui.qs.QSContainerImpl>

   

quick_settings_status_bar_background 就是最上面status的背景
quick_settings_gradient_view 就是功能开关键的背景
所以修改为

  <View
        android:id="@+id/quick_settings_status_bar_background"
        android:layout_width="match_parent"
        android:layout_height="@*android:dimen/quick_qs_offset_height"
        android:clipToPadding="false"
        android:clipChildren="false"
        android:background="#00000000" />

    <!-- Gradient view behind QS -->
    <View
        android:id="@+id/quick_settings_gradient_view"
        android:layout_width="match_parent"
        android:layout_height="126dp"
        android:layout_marginTop="@*android:dimen/quick_qs_offset_height"
        android:clipToPadding="false"
        android:clipChildren="false"/>


```

所以把 quick_settings_status_bar_background 和 quick_settings_gradient_view 的关于下拉状态栏的背景 android:background=“#00000000” 设置为透明就可以了

### 3.2 关于 QuickQsPanel 设置功能开关键的透明度的修改

在 QuickStatusBarHeader.java 中相关源码分析

```
      @Override
      protected void onFinishInflate() {
          super.onFinishInflate();
  
          mHeaderQsPanel = findViewById(R.id.quick_qs_panel);
          mSystemIconsView = findViewById(R.id.quick_status_bar_system_icons);
          mQuickQsStatusIcons = findViewById(R.id.quick_qs_status_icons);
          StatusIconContainer iconContainer = findViewById(R.id.statusIcons);
          // Ignore privacy icons because they show in the space above QQS
          iconContainer.addIgnoredSlots(getIgnoredIconSlots());
          iconContainer.setShouldRestrictIcons(false);
          mIconManager = new TintedIconManager(iconContainer, mCommandQueue);
  
          // Views corresponding to the header info section (e.g. ringer and next alarm).
          mHeaderTextContainerView = findViewById(R.id.header_text_container);
          mStatusSeparator = findViewById(R.id.status_separator);
          mNextAlarmIcon = findViewById(R.id.next_alarm_icon);
          mNextAlarmTextView = findViewById(R.id.next_alarm_text);
          mNextAlarmContainer = findViewById(R.id.alarm_container);
          mNextAlarmContainer.setOnClickListener(this::onClick);
          mRingerModeIcon = findViewById(R.id.ringer_mode_icon);
          mRingerModeTextView = findViewById(R.id.ringer_mode_text);
          mRingerContainer = findViewById(R.id.ringer_container);
          mRingerContainer.setOnClickListener(this::onClick);
          mPrivacyChip = findViewById(R.id.privacy_chip);
          mPrivacyChip.setOnClickListener(this::onClick);
          mCarrierGroup = findViewById(R.id.carrier_group);
  
  
          updateResources();
  
          Rect tintArea = new Rect(0, 0, 0, 0);
          int colorForeground = Utils.getColorAttrDefaultColor(getContext(),
                  android.R.attr.colorForeground);
          float intensity = getColorIntensity(colorForeground);
          int fillColor = mDualToneHandler.getSingleColor(intensity);
  
          // Set light text on the header icons because they will always be on a black background
          applyDarkness(R.id.clock, tintArea, 0, DarkIconDispatcher.DEFAULT_ICON_TINT);
  
          // Set the correct tint for the status icons so they contrast
          mIconManager.setTint(fillColor);
          mNextAlarmIcon.setImageTintList(ColorStateList.valueOf(fillColor));
          mRingerModeIcon.setImageTintList(ColorStateList.valueOf(fillColor));
  
          mClockView = findViewById(R.id.clock);
          mClockView.setOnClickListener(this);
          mDateView = findViewById(R.id.date);
          mSpace = findViewById(R.id.space);
  
          // Tint for the battery icons are handled in setupHost()
          mBatteryRemainingIcon = findViewById(R.id.batteryRemainingIcon);
          // Don't need to worry about tuner settings for this icon
          mBatteryRemainingIcon.setIgnoreTunerUpdates(true);
          // QS will always show the estimate, and BatteryMeterView handles the case where
          // it's unavailable or charging
          mBatteryRemainingIcon.setPercentShowMode(BatteryMeterView.MODE_ESTIMATE);
          mRingerModeTextView.setSelected(true);
          mNextAlarmTextView.setSelected(true);
  
          mAllIndicatorsEnabled = mPrivacyItemController.getAllIndicatorsAvailable();
          mMicCameraIndicatorsEnabled = mPrivacyItemController.getMicCameraAvailable();
      }
      @Override
      protected void onConfigurationChanged(Configuration newConfig) {
          super.onConfigurationChanged(newConfig);
          updateResources();
  
          // Update color schemes in landscape to use wallpaperTextColor
          boolean shouldUseWallpaperTextColor =
                  newConfig.orientation == Configuration.ORIENTATION_LANDSCAPE;
          mClockView.useWallpaperTextColor(shouldUseWallpaperTextColor);
      }
  
      @Override
      public void onRtlPropertiesChanged(int layoutDirection) {
          super.onRtlPropertiesChanged(layoutDirection);
          updateResources();
      }
      private void updateResources() {
        Resources resources = mContext.getResources();
        updateMinimumHeight();

        // Update height for a few views, especially due to landscape mode restricting space.
        mHeaderTextContainerView.getLayoutParams().height =
                resources.getDimensionPixelSize(R.dimen.qs_header_tooltip_height);
        mHeaderTextContainerView.setLayoutParams(mHeaderTextContainerView.getLayoutParams());

        mSystemIconsView.getLayoutParams().height = resources.getDimensionPixelSize(
                com.android.internal.R.dimen.quick_qs_offset_height);
        mSystemIconsView.setLayoutParams(mSystemIconsView.getLayoutParams());

        FrameLayout.LayoutParams lp = (FrameLayout.LayoutParams) getLayoutParams();
        if (mQsDisabled) {
            lp.height = resources.getDimensionPixelSize(
                    com.android.internal.R.dimen.quick_qs_offset_height);
        } else {
            lp.height = Math.max(getMinimumHeight(),
                    resources.getDimensionPixelSize(
                            com.android.internal.R.dimen.quick_qs_total_height));
        }

        setLayoutParams(lp);

        updateStatusIconAlphaAnimator();
        updateHeaderTextContainerAlphaAnimator();
    }
    private void updateStatusIconAlphaAnimator() {
        mStatusIconsAlphaAnimator = new TouchAnimator.Builder()
                .addFloat(mQuickQsStatusIcons, "alpha", 1, 0, 0)
                .build();
    }

```

在布局完成的 onFinishInflate() 和 onConfigurationChanged(Configuration newConfig) 和 onRtlPropertiesChanged(int layoutDirection) 等和布局相关的方法中，都调用 updateResources() 可以看出 通过 updateResources() 更新背景  
而在 updateResources() 中的  
updateStatusIconAlphaAnimator(); 就是 mQuickQsStatusIcons 动画 设置它的透明度  
会有所影响 所以应该把它去掉  
修改为：`/*updateStatusIconAlphaAnimator();*/`
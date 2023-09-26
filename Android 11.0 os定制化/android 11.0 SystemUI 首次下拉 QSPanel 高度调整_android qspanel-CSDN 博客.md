> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124739168)

### 1. 概述

在 11.0 系统的产品开发中，对 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 首次下拉显示的是 时间 各种 SystemUI 相关的系统图标 和 QuickQSPanel 等 而关于 QSPanel 的高度调整就是需求功能

### 2.SystemUI 首次下拉 QSPanel 高度调整的核心类

```
frameworks/base/packages/SystemUI/res/layout/quick_status_bar_expanded_header.xml
frameworks/base/packages/SystemUI/src/com/android/systemui/qs/QuickStatusBarHeader.java

```

### 3 SystemUI 首次下拉 QSPanel 高度调整的核心功能分析和实现

从下拉状态栏的 QSPanel 的源码中发现布局就是 quick_status_bar_expanded_header.xml  
先看看布局源码

### 3.1quick_status_bar_expanded_header.xml 布局源码

```
<com.android.systemui.qs.QuickStatusBarHeader
xmlns:android="http://schemas.android.com/apk/res/android"
android:id="@+id/header"
android:layout_width="match_parent"
android:layout_height="@*android:dimen/quick_qs_total_height"
android:layout_gravity="@integer/notification_panel_layout_gravity"
android:background="@android:color/transparent"
android:baselineAligned="false"
android:clickable="false"
android:clipChildren="false"
android:clipToPadding="false"
android:paddingTop="0dp"
android:paddingEnd="0dp"
android:paddingStart="0dp"
android:elevation="4dp" >
<include layout="@layout/quick_status_bar_header_system_icons" />

<!-- Status icons within the panel itself (and not in the top-most status bar) -->
<include layout="@layout/quick_qs_status_icons" />

<!-- Layout containing tooltips, alarm text, etc. -->
<include layout="@layout/quick_settings_header_info" />

<com.android.systemui.qs.QuickQSPanel
    android:id="@+id/quick_qs_panel"
    android:layout_width="match_parent"
    android:layout_height="48dp"
    android:layout_below="@id/quick_qs_status_icons"
    android:layout_marginStart="@dimen/qs_header_tile_margin_horizontal"
    android:layout_marginEnd="@dimen/qs_header_tile_margin_horizontal"
    android:accessibilityTraversalAfter="@+id/date_time_group"
    android:accessibilityTraversalBefore="@id/expand_indicator"
    android:clipChildren="false"
    android:clipToPadding="false"
    android:focusable="true"
    android:importantForAccessibility="yes" />

<com.android.systemui.statusbar.AlphaOptimizedImageView
    android:id="@+id/qs_detail_header_progress"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_alignParentBottom="true"
    android:alpha="0"
    android:background="@color/qs_detail_progress_track"
    android:src="@drawable/indeterminate_anim"/>

<TextView
    android:id="@+id/header_debug_info"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:layout_gravity="center_vertical"
    android:fontFamily="sans-serif-condensed"
    android:padding="2dp"
    android:textColor="#00A040"
    android:textSize="11dp"
    android:textStyle="bold"
    android:visibility="invisible"/>

```

从 quick_status_bar_expanded_header.xml 中源码中可以看出  
</com.android.systemui.qs.QuickStatusBarHeader>  
从布局中可以看到 QuickStatusBarHeader.java 是根布局

### 3.2QuickStatusBarHeader.java 的相关布局分析

路径为: frameworks/base/packages/SystemUI/src/com/android/systemui/qs/QuickStatusBarHeader.java  
接下来看下 QuickStatusBarHeader.java 的源码

```
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

、、、省略部分代码
}

```

在 onFinishInflate() 中的布局完成后调用 updateResources(); 进行布局  
而 updateResources(); 就是设置相关布局参数

```
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

}

```

在 FrameLayout.LayoutParams lp = (FrameLayout.LayoutParams) getLayoutParams(); 中可以看到  
lp.height 就是高度值  
所以具体修改参数为:

```
        FrameLayout.LayoutParams lp = (FrameLayout.LayoutParams) getLayoutParams();
         if (mQsDisabled) {
-            lp.height = resources.getDimensionPixelSize(
-                    com.android.internal.R.dimen.quick_qs_offset_height);
+            lp.height = resources.getDimensionPixelSize(R.dimen.quick_qs_height);
         } else {
-            lp.height = Math.max(getMinimumHeight(),
-                    resources.getDimensionPixelSize(
-                            com.android.internal.R.dimen.quick_qs_total_height));
+            lp.height = Math.max(getMinimumHeight(),resources.getDimensionPixelSize(R.dimen.quick_qs_height));
         }

```

quick_qs_height 就是 dimens 的高度值, 修改合适的高度，赋值给 lp.height 就达到了设置下拉  
QSPanel 高度的目的
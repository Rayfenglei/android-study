> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/128448706)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.SystemUI 音量条 UI 定制的功能的核心类 (二）](#2.SystemUI%20%E9%9F%B3%E9%87%8F%E6%9D%A1UI%E5%AE%9A%E5%88%B6%E7%9A%84%E5%8A%9F%E8%83%BD%E7%9A%84%E6%A0%B8%E5%BF%83%E7%B1%BB%20%28%E4%BA%8C%EF%BC%89)

[3.SystemUI 音量条 UI 定制的功能的核心功能分析和实现 (二）](#3.SystemUI%20%E9%9F%B3%E9%87%8F%E6%9D%A1UI%E5%AE%9A%E5%88%B6%E7%9A%84%E5%8A%9F%E8%83%BD%E7%9A%84%E6%A0%B8%E5%BF%83%E5%8A%9F%E8%83%BD%E5%88%86%E6%9E%90%E5%92%8C%E5%AE%9E%E7%8E%B0%20%28%E4%BA%8C%EF%BC%89)

[3.1 关于 volume_dialog_row.xml 的修改](#t3)

[3.2 VolumeDialogImpl.java 关于去掉 ringer 和 音量条菜单的修改](#t4)

1. 概述
-----

在 11.0 的系统软件开发中，对于 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 音量条的定制功能（二），项目需求要求在音量条 UI 做定制 根据 ui 设计图来重新定制音量条 UI，这就需要在 SystemUI 中找到对应的音量条布局来重新设置音量条布局，来完成 音量条 UI 的布局设置，接下来完成第二部分的功能

效果图如图:

![](https://img-blog.csdnimg.cn/a94eea883b7d4ae1a918de16d4e56a86.png)

2.SystemUI 音量条 UI 定制的功能的核心类 (二）
-------------------------------

```
frameworks/base/packages/SystemUI/src/com/android/systemui/volume/VolumeDialogImpl.java
frameworks/base/packages/apps/SystemUI/res/layout/volume_dialog_row.xml
frameworks/base/packages/apps/SystemUI/res/drawable/volume_progress_drawable.xml
```

3.SystemUI 音量条 UI 定制的功能的核心功能分析和实现 (二）
-------------------------------------

3.1 关于 volume_dialog_row.xml 的修改
--------------------------------

具体修改的 patch:

```
  --- a/vendor/mediatek/proprietary/packages/apps/SystemUI/res/layout/volume_dialog_row.xml
+++ b/vendor/mediatek/proprietary/packages/apps/SystemUI/res/layout/volume_dialog_row.xml
@@ -23,7 +23,7 @@
     android:theme="@style/qs_theme">
 
     <LinearLayout
-        android:layout_height="wrap_content"
+        android:layout_height="match_parent"
         android:layout_width="match_parent"
         android:gravity="center"
         android:layout_gravity="center"
@@ -31,7 +31,7 @@
         <TextView
             android:id="@+id/volume_row_header"
             android:layout_width="wrap_content"
-            android:layout_height="wrap_content"
+            android:layout_height="0dp"
             android:ellipsize="end"
             android:maxLength="10"
             android:maxLines="1"
@@ -41,15 +41,19 @@
         <FrameLayout
             android:id="@+id/volume_row_slider_frame"
             android:layout_width="match_parent"
-            android:layout_marginTop="@dimen/volume_dialog_slider_margin_top"
-            android:layout_marginBottom="@dimen/volume_dialog_slider_margin_bottom"
             android:layoutDirection="rtl"
-            android:layout_height="@dimen/volume_dialog_slider_height">
+            android:layout_height="350px">
             <SeekBar
                 android:id="@+id/volume_row_slider"
                 android:clickable="true"
-                android:layout_width="@dimen/volume_dialog_slider_height"
-                android:layout_height="match_parent"
+                android:layout_width="350px"
+                android:layout_height="50px"
+                android:thumb="@drawable/icon_jindu"
+                               android:splitTrack="false"
+                android:thumbOffset="6px"
+                android:paddingLeft="0px"
+                       android:paddingRight="0px"
+                android:progressDrawable="@drawable/volume_progress_drawable"
                 android:layoutDirection="rtl"
                 android:layout_gravity="center"
                 android:rotation="90" />
@@ -60,9 +64,6 @@
             style="@style/VolumeButtons"
             android:layout_width="@dimen/volume_dialog_tap_target_size"
             android:layout_height="@dimen/volume_dialog_tap_target_size"
-            android:background="@drawable/ripple_drawable_20dp"
-            android:layout_marginBottom="@dimen/volume_dialog_row_margin_bottom"
-            android:tint="@color/accent_tint_color_selector"
             android:soundEffectsEnabled="false" />
     </LinearLayout>
```

在 volume_dialog_row.xml 上述代码中可以看出 volume_row_slider 就是音量条的布局，给他新增了  
volume_progress_drawable.xml 布局 android:thumbOffset="6px" 主要是 thumb 偏移的控制  
android:splitTrack="false" 就是 thumb 拖动过程中的阴影问题  
接下来看下新增的 volume_progress_drawable.xml 布局

```
<?xml version="1.0" encoding="utf-8"?>
<layer-list xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:id="@android:id/background"
          android:gravity="center_vertical|fill_horizontal">
        <shape android:shape="rectangle">
            <size android:height="15px" />
            <solid android:color="#6495ED" />
            <corners android:radius="10px" />
        </shape>
    </item>
    <item android:id="@android:id/progress"
          android:gravity="center_vertical|fill_horizontal">
        <scale android:scaleWidth="100%">
            <shape android:shape="rectangle">
                <size android:height="15px" />
                <solid android:color="@android:color/white" />
                <corners android:radius="10px" />
            </shape>
        </scale>
    </item>
</layer-list>
```

3.2 VolumeDialogImpl.java 关于去掉 ringer 和 音量条菜单的修改
------------------------------------------------

```
     --- a/vendor/mediatek/proprietary/packages/apps/SystemUI/src/com/android/systemui/volume/VolumeDialogImpl.java
+++ b/vendor/mediatek/proprietary/packages/apps/SystemUI/src/com/android/systemui/volume/VolumeDialogImpl.java
@@ -259,9 +259,11 @@ public class VolumeDialogImpl implements VolumeDialog,
 
         mDialogRowsView = mDialog.findViewById(R.id.volume_dialog_rows);
         mRinger = mDialog.findViewById(R.id.ringer);
         if (mRinger != null) {
             mRingerIcon = mRinger.findViewById(R.id.ringer_icon);
             mZenIcon = mRinger.findViewById(R.id.dnd_icon);
+                       mRinger.setVisibility(View.GONE);
         }
 
         mODICaptionsView = mDialog.findViewById(R.id.odi_captions);
@@ -276,7 +278,7 @@ public class VolumeDialogImpl implements VolumeDialog,
 
         mSettingsView = mDialog.findViewById(R.id.settings_container);
         mSettingsIcon = mDialog.findViewById(R.id.settings);
 
         if (mRows.isEmpty()) {
             if (!AudioSystem.isSingleVolume(mContext)) {
                 addRow(STREAM_ACCESSIBILITY, R.drawable.ic_volume_accessibility,
@@ -461,9 +463,9 @@ public class VolumeDialogImpl implements VolumeDialog,
     public void initSettingsH() {
         if (mSettingsView != null) {
             mSettingsView.setVisibility(
-                    mDeviceProvisionedController.isCurrentUserSetup() &&
+                    /*mDeviceProvisionedController.isCurrentUserSetup() &&
                             mActivityManager.getLockTaskModeState() == LOCK_TASK_MODE_NONE ?
-                            VISIBLE : GONE);
+                            VISIBLE : */GONE);
         }
         if (mSettingsIcon != null) {
             mSettingsIcon.setOnClickListener(v -> {
@@ -1108,11 +1110,11 @@ public class VolumeDialogImpl implements VolumeDialog,
                 ? Color.alpha(tint.getDefaultColor())
                 : getAlphaAttr(android.R.attr.secondaryContentAlpha);
         if (tint == row.cachedTint) return;
-        row.slider.setProgressTintList(tint);
-        row.slider.setThumbTintList(tint);
-        row.slider.setProgressBackgroundTintList(tint);
+        //row.slider.setProgressTintList(tint);
+        //row.slider.setThumbTintList(tint);
+        //row.slider.setProgressBackgroundTintList(tint);
         row.slider.setAlpha(((float) alpha) / 255);
-        row.icon.setImageTintList(tint);
+        //row.icon.setImageTintList(tint);
         row.icon.setImageAlpha(alpha);
         row.cachedTint = tint;
     }
```

在上述 VolumeDialogImpl.java 的代码中 mRinger.setVisibility(View.GONE); 就是隐藏最顶部  
ringer 布局图标，而 mSettingsView 部分的相关代码就是隐藏 音量条菜单设置的图标，  
关于 row.[slider](https://so.csdn.net/so/search?q=slider&spm=1001.2101.3001.7020) 的相关属性就是 在按音量 + 和音量 - 键后 对音量条 seekbar 的重新布局上色的  
相关代码
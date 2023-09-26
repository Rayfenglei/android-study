> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124760377)

### 1. 概述

在 11.0 的产品开发中，[SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 的下拉状态栏中，由需求要求对通知栏通知做圆角背景就是对下拉状态栏中的通知栏每条 Notification 做 ui 的圆角背景，首选看 NotificationRow 长按时，弹出的背景布局就是 NotificationGuts

### 2. 修改下拉通知栏的 NotificationGuts 背景为圆角背景的核心类

```
frameworks/base/packages/SystemUI/res/layout/notification_info.xml
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/notification/row/NotificationGuts.java

```

### 3. 修改下拉通知栏的 NotificationGuts 背景为圆角背景的核心功能分析和实现

接下来就需要对 NotificationGuts 做背景圆角布局做分析，然后修改 NotificationGuts 的圆角背景  
通过阅读源码发现布局为  
notification_info.xml

### 3.1notification_info.xml 相关源码功能分析

```
<?xml version="1.0" encoding="utf-8"?>

<com.android.systemui.statusbar.notification.row.NotificationInfo
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/notification_guts"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:clickable="true"
    android:clipChildren="false"
    android:clipToPadding="true"
    android:orientation="vertical"
    android:paddingStart="@*android:dimen/notification_content_margin_start"
    android:background="@color/notification_guts_bg_color">

    <!-- Package Info -->
    <RelativeLayout
        android:id="@+id/header"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:clipChildren="false"
        android:clipToPadding="false">
        <ImageView
            android:id="@+id/pkgicon"
            android:layout_width="@dimen/notification_guts_header_height"
            android:layout_height="@dimen/notification_guts_header_height"
            android:layout_centerVertical="true"
            android:layout_alignParentStart="true"
            android:layout_marginEnd="3dp" />
        <TextView
            android:id="@+id/pkgname"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_centerVertical="true"
            style="@style/TextAppearance.NotificationImportanceHeader"
            android:layout_marginStart="3dp"
            android:layout_marginEnd="2dp"
            android:layout_toEndOf="@id/pkgicon"
            android:singleLine="true" />
        <TextView
            android:id="@+id/pkg_divider"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_centerVertical="true"
            style="@style/TextAppearance.NotificationImportanceHeader"
            android:layout_marginStart="2dp"
            android:layout_marginEnd="2dp"
            android:layout_toEndOf="@id/pkgname"
            android:text="@*android:string/notification_header_divider_symbol" />
        <TextView
            android:id="@+id/delegate_name"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_centerVertical="true"
            style="@style/TextAppearance.NotificationImportanceHeader"
            android:layout_marginStart="2dp"
            android:layout_marginEnd="2dp"
            android:ellipsize="end"
            android:text="@string/notification_delegate_header"
            android:layout_toEndOf="@id/pkg_divider"
            android:maxLines="1" />
        <!-- Optional link to app. Only appears if the channel is not disabled and the app
asked for it -->
        <ImageButton
            android:id="@+id/app_settings"
            android:layout_width="@dimen/notification_importance_toggle_size"
            android:layout_height="@dimen/notification_importance_toggle_size"
            android:layout_centerVertical="true"
            android:visibility="gone"
            android:background="@drawable/ripple_drawable"
            android:contentDescription="@string/notification_app_settings"
            android:src="@drawable/ic_info"
            android:layout_toStartOf="@id/info"
            android:tint="@color/notification_guts_link_icon_tint"/>
        <ImageButton
            android:id="@+id/info"
            android:layout_width="@dimen/notification_importance_toggle_size"
            android:layout_height="@dimen/notification_importance_toggle_size"
            android:layout_centerVertical="true"
            android:background="@drawable/ripple_drawable"
            android:contentDescription="@string/notification_more_settings"
            android:src="@drawable/ic_settings"
            android:layout_alignParentEnd="true"
            android:tint="@color/notification_guts_link_icon_tint"/>
    </RelativeLayout>
 
     <!-- Channel Info Block -->
     <LinearLayout
         android:id="@+id/channel_info"
         android:layout_width="match_parent"
         android:layout_height="wrap_content"
         android:paddingEnd="@*android:dimen/notification_content_margin_end"
         android:gravity="center"
         android:orientation="vertical">
         <!-- Channel Name -->
         <TextView
             android:id="@+id/channel_name"
             android:layout_width="wrap_content"
             android:layout_height="wrap_content"
             android:layout_weight="1"
             style="@style/TextAppearance.NotificationImportanceChannel"/>
         <TextView
             android:id="@+id/group_name"
             android:layout_width="wrap_content"
             android:layout_height="wrap_content"
             style="@style/TextAppearance.NotificationImportanceChannelGroup"
             android:ellipsize="end"
             android:maxLines="1"/>
     </LinearLayout>
 
     <LinearLayout
         android:id="@+id/blocking_helper"
         android:layout_width="match_parent"
         android:layout_height="wrap_content"
         android:layout_marginTop="@dimen/notification_guts_button_spacing"
         android:layout_marginBottom="@dimen/notification_guts_button_spacing"
         android:paddingEnd="@*android:dimen/notification_content_margin_end"
         android:clipChildren="false"
         android:clipToPadding="false"
         android:orientation="vertical">
         <!-- blocking helper text. no need for non-configurable check b/c controls won't be
         activated in that case -->
         <TextView
             android:id="@+id/blocking_helper_text"
             android:layout_width="wrap_content"
             android:layout_height="wrap_content"
             android:layout_marginTop="2dp"
             android:text="@string/inline_blocking_helper"
             style="@*android:style/TextAppearance.DeviceDefault.Notification" />
         <RelativeLayout
             android:id="@+id/block_buttons"
             android:layout_width="match_parent"
             android:layout_height="wrap_content"
             android:layout_marginTop="@dimen/notification_guts_button_spacing">
             <TextView
                 android:id="@+id/blocking_helper_turn_off_notifications"
                 android:text="@string/inline_turn_off_notifications"
                 android:layout_width="wrap_content"
                 android:layout_height="wrap_content"
                 android:layout_centerVertical="true"
                 android:layout_alignParentStart="true"
                 android:width="110dp"
                 android:paddingEnd="15dp"
                 android:breakStrategy="simple"
                 style="@style/TextAppearance.NotificationInfo.Button"/>
             <TextView
                 android:id="@+id/deliver_silently"
                 android:text="@string/inline_deliver_silently_button"
                 android:layout_width="wrap_content"
                 android:layout_height="wrap_content"
                 android:layout_centerVertical="true"
                 android:layout_marginStart="@dimen/notification_guts_button_horizontal_spacing"
                 android:paddingEnd="15dp"
                 android:width="110dp"
                 android:breakStrategy="simple"
                 android:layout_toStartOf="@+id/keep_showing"
                 style="@style/TextAppearance.NotificationInfo.Button"/>
             <TextView
                 android:id="@+id/keep_showing"
                 android:text="@string/inline_keep_button"
                 android:layout_width="wrap_content"
                 android:layout_height="wrap_content"
                 android:layout_centerVertical="true"
                 android:layout_marginStart="@dimen/notification_guts_button_horizontal_spacing"
                 android:width="110dp"
                 android:breakStrategy="simple"
                 android:layout_alignParentEnd="true"
                 style="@style/TextAppearance.NotificationInfo.Button"/>
         </RelativeLayout>
 
     </LinearLayout>
 
     <LinearLayout
         android:id="@+id/inline_controls"
         android:layout_width="match_parent"
         android:layout_height="wrap_content"
         android:paddingEnd="@*android:dimen/notification_content_margin_end"
         android:layout_marginTop="@dimen/notification_guts_option_vertical_padding"
         android:clipChildren="false"
         android:clipToPadding="false"
         android:orientation="vertical">
 
         <!-- Non configurable app/channel text. appears instead of @+id/interruptiveness_settings-->
         <TextView
             android:id="@+id/non_configurable_text"
             android:text="@string/notification_unblockable_desc"
             android:visibility="gone"
             android:layout_width="match_parent"
             android:layout_height="wrap_content"
             style="@*android:style/TextAppearance.DeviceDefault.Notification" />
 
         <!-- Non configurable multichannel text. appears instead of @+id/interruptiveness_settings-->
         <TextView
             android:id="@+id/non_configurable_multichannel_text"
             android:text="@string/notification_multichannel_desc"
             android:visibility="gone"
             android:layout_width="match_parent"
             android:layout_height="wrap_content"
             style="@*android:style/TextAppearance.DeviceDefault.Notification" />
 
         <LinearLayout
             android:id="@+id/interruptiveness_settings"
             android:layout_width="match_parent"
             android:layout_height="wrap_content"
             android:gravity="center"
             android:orientation="vertical">
 
             <com.android.systemui.statusbar.notification.row.ButtonLinearLayout
                 android:id="@+id/alert"
                 android:layout_width="match_parent"
                 android:layout_height="wrap_content"
                 android:padding="@dimen/notification_importance_button_padding"
                 android:clickable="true"
                 android:focusable="true"
                 android:background="@drawable/notification_guts_priority_button_bg"
                 android:orientation="vertical">
                 <LinearLayout
                     android:layout_width="match_parent"
                     android:layout_height="wrap_content"
                     android:orientation="horizontal"
                     android:gravity="center"
                     >
                     <ImageView
                         android:id="@+id/alert_icon"
                         android:layout_width="wrap_content"
                         android:layout_height="wrap_content"
                         android:src="@drawable/ic_notifications_alert"
                         android:background="@android:color/transparent"
                         android:tint="@color/notification_guts_priority_contents"
                         android:clickable="false"
                         android:focusable="false"/>
                     <TextView
                         android:id="@+id/alert_label"
                         android:layout_width="0dp"
                         android:layout_height="wrap_content"
                         android:layout_marginStart="@dimen/notification_importance_drawable_padding"
                         android:layout_weight="1"
                         android:ellipsize="end"
                         android:maxLines="1"
                         android:clickable="false"
                         android:focusable="false"
                         android:textAppearance="@style/TextAppearance.NotificationImportanceButton"
                         android:text="@string/notification_alert_title"/>
                 </LinearLayout>
                 <TextView
                     android:id="@+id/alert_summary"
                     android:layout_width="match_parent"
                     android:layout_height="wrap_content"
                     android:layout_marginTop="@dimen/notification_importance_button_description_top_margin"
                     android:visibility="gone"
                     android:text="@string/notification_channel_summary_default"
                     android:clickable="false"
                     android:focusable="false"
                     android:ellipsize="end"
                     android:maxLines="2"
                     android:textAppearance="@style/TextAppearance.NotificationImportanceDetail"/>
             </com.android.systemui.statusbar.notification.row.ButtonLinearLayout>
 
             <com.android.systemui.statusbar.notification.row.ButtonLinearLayout
                 android:id="@+id/silence"
                 android:layout_width="match_parent"
                 android:layout_height="wrap_content"
                 android:layout_marginTop="@dimen/notification_importance_button_separation"
                 android:padding="@dimen/notification_importance_button_padding"
                 android:clickable="true"
                 android:focusable="true"
                 android:background="@drawable/notification_guts_priority_button_bg"
                 android:orientation="vertical">
                 <LinearLayout
                     android:layout_width="match_parent"
                     android:layout_height="wrap_content"
                     android:orientation="horizontal"
                     android:gravity="center"
                     >
                     <ImageView
                         android:id="@+id/silence_icon"
                         android:src="@drawable/ic_notifications_silence"
                         android:background="@android:color/transparent"
                         android:tint="@color/notification_guts_priority_contents"
                         android:layout_gravity="center"
                         android:layout_width="wrap_content"
                         android:layout_height="wrap_content"
                         android:clickable="false"
                         android:focusable="false"/>
                     <TextView
                         android:id="@+id/silence_label"
                         android:layout_width="match_parent"
                         android:layout_height="wrap_content"
                         android:ellipsize="end"
                         android:maxLines="1"
                         android:clickable="false"
                         android:focusable="false"
                         android:layout_toEndOf="@id/silence_icon"
                         android:layout_marginStart="@dimen/notification_importance_drawable_padding"
                         android:textAppearance="@style/TextAppearance.NotificationImportanceButton"
                         android:text="@string/notification_silence_title"/>
                 </LinearLayout>
                 <TextView
                     android:id="@+id/silence_summary"
                     android:layout_width="match_parent"
                     android:layout_height="wrap_content"
                     android:layout_marginTop="@dimen/notification_importance_button_description_top_margin"
                     android:visibility="gone"
                     android:text="@string/notification_channel_summary_low"
                     android:clickable="false"
                     android:focusable="false"
                     android:ellipsize="end"
                     android:maxLines="2"
                     android:textAppearance="@style/TextAppearance.NotificationImportanceDetail"/>
             </com.android.systemui.statusbar.notification.row.ButtonLinearLayout>
 
         </LinearLayout>
 
         <RelativeLayout
             android:id="@+id/bottom_buttons"
             android:layout_width="match_parent"
             android:layout_height="60dp"
             android:gravity="center_vertical"
             android:paddingStart="4dp"
             android:paddingEnd="4dp"
             >
             <TextView
                 android:id="@+id/turn_off_notifications"
                 android:text="@string/inline_turn_off_notifications"
                 android:layout_width="wrap_content"
                 android:layout_height="wrap_content"
                 android:layout_alignParentStart="true"
                 android:gravity="start|center_vertical"
                 android:minWidth="@dimen/notification_importance_toggle_size"
                 android:minHeight="@dimen/notification_importance_toggle_size"
                 android:maxWidth="200dp"
                 style="@style/TextAppearance.NotificationInfo.Button"/>
             <TextView
                 android:id="@+id/done"
                 android:text="@string/inline_ok_button"
                 android:layout_width="wrap_content"
                 android:layout_height="wrap_content"
                 android:layout_alignParentEnd="true"
                 android:gravity="end|center_vertical"
                 android:minWidth="@dimen/notification_importance_toggle_size"
                 android:minHeight="@dimen/notification_importance_toggle_size"
                 android:maxWidth="125dp"
                 style="@style/TextAppearance.NotificationInfo.Button"/>
         </RelativeLayout>
 
     </LinearLayout>
 
     <com.android.systemui.statusbar.notification.row.NotificationUndoLayout
         android:id="@+id/confirmation"
         android:layout_width="match_parent"
         android:layout_height="wrap_content"
         android:visibility="gone"
         android:orientation="horizontal" >
         <TextView
             android:id="@+id/confirmation_text"
             android:layout_width="wrap_content"
             android:layout_height="wrap_content"
             android:layout_gravity="start|center_vertical"
             android:layout_marginStart="@*android:dimen/notification_content_margin_start"
             android:layout_marginEnd="@*android:dimen/notification_content_margin_start"
             android:text="@string/notification_channel_disabled"
             style="@style/TextAppearance.NotificationInfo.Confirmation"/>
         <TextView
             android:id="@+id/undo"
             android:layout_width="wrap_content"
             android:layout_height="wrap_content"
             android:minWidth="@dimen/notification_importance_toggle_size"
             android:minHeight="@dimen/notification_importance_toggle_size"
             android:layout_marginTop="@dimen/notification_guts_button_spacing"
             android:layout_marginBottom="@dimen/notification_guts_button_spacing"
             android:layout_marginStart="@dimen/notification_guts_button_side_margin"
             android:layout_marginEnd="@dimen/notification_guts_button_side_margin"
             android:layout_gravity="end|center_vertical"
             android:text="@string/inline_undo"
             style="@style/TextAppearance.NotificationInfo.Button"/>
     </com.android.systemui.statusbar.notification.row.NotificationUndoLayout>
 </com.android.systemui.statusbar.notification.row.NotificationInfo>

```

从 notification_info.xml 源码中发现 com.android.systemui.statusbar.[notification](https://so.csdn.net/so/search?q=notification&spm=1001.2101.3001.7020).row.NotificationInfo 为  
根布局 所以设置圆角如下

```
--- a/frameworks/base/packages/SystemUI/res/layout/notification_info.xml
+++ b/frameworks/base/packages/SystemUI/res/layout/notification_info.xml
@@ -25,7 +25,7 @@
     android:clipToPadding="true"
     android:orientation="vertical"
     android:paddingStart="@*android:dimen/notification_content_margin_start"
-    android:background="@color/notification_guts_bg_color">
+    android:background="@drawable/qs_background_primary">
 
     <!-- Package Info -->
     <RelativeLayout

```

添加圆角背景如下  
qs_background_primary.xml

```
<inset xmlns:android="http://schemas.android.com/apk/res/android">
    <shape>
        <solid android:color="#ffffff"/>
        <corners android:radius="20dp" />
    </shape>
</inset>

```

设置上述的圆角后接下来设置 NotificationGuts.java 的圆角布局  
接下来对 NotificationGuts.java 中的背景也改成圆角背景

### 3.2NotificationGuts.java 的相关源码分析

```
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/notification/row/NotificationGuts.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/notification/row/NotificationGuts.java
@@ -173,7 +173,7 @@ public class NotificationGuts extends FrameLayout {
     @Override
     protected void onFinishInflate() {
         super.onFinishInflate();
-        mBackground = mContext.getDrawable(R.drawable.notification_guts_bg);
+        mBackground = mContext.getDrawable(R.drawable.qs_background_primary);
         if (mBackground != null) {
             mBackground.setCallback(this);
         }

```

在 onFinishInflate() 中设置 mBackground 的背景为圆角布局就行

这样简单的做修改就把 NotificationGuts 改为圆角背景
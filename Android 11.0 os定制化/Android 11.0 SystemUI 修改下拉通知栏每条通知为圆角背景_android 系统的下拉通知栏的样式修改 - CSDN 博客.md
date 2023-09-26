> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124760344)

### 1. 概述

在 11.0 定制化开发 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020), 对下拉通知栏的通知布局背景要求是圆角的, 下拉通知栏每条通知的背景修改为圆角背景, 而下拉通知的布局文件为 status_bar_notification_row.xml

### 2.SystemUI 修改下拉通知栏每条通知为圆角背景的核心类

```
frameworks/base/packages/SystemUI/res/layout/status_bar_notification_row.xml
frameworks/base/packages/SystemUI/res/layout/status_bar_notification_section_header.xml
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/notification/row/ActivatableNotificationView.java

```

### 3.SystemUI 修改下拉通知栏每条通知为圆角背景核心功能分析和实现

### 3.1 对布局文件添加圆角背景为:

qs_background_primary.xml

```
<inset xmlns:android="http://schemas.android.com/apk/res/android">
    <shape>
        <solid android:color="#ffffff"/>
        <corners android:radius="20dp" />
    </shape>
</inset>

```

通过 shape 来增加一个半径为 20dp 的圆角背景

### 3.2SystemUI 下拉通知的每条通知的布局分析

```
--- a/frameworks/base/packages/SystemUI/res/layout/status_bar_notification_row.xml
+++ b/frameworks/base/packages/SystemUI/res/layout/status_bar_notification_row.xml
@@ -39,12 +39,13 @@
     <com.android.systemui.statusbar.notification.row.NotificationContentView
         android:id="@+id/expanded"
        android:layout_width="match_parent"
-       android:layout_height="wrap_content" />
+       android:layout_height="wrap_content"
+          android:background="@drawable/qs_background_primary"/>
 
     <com.android.systemui.statusbar.notification.row.NotificationContentView
         android:id="@+id/expandedPublic"
         android:layout_width="match_parent"
-        android:layout_height="wrap_content" />
+        android:layout_height="wrap_content"           />

 
     <ViewStub

```

在 NotificationContentView 中增加圆角背景作为通知的背景

### 3.3 status_bar_notification_section_header.xml 圆角背景的修改

```
--- a/frameworks/base/packages/SystemUI/res/layout/status_bar_notification_section_header.xml
+++ b/frameworks/base/packages/SystemUI/res/layout/status_bar_notification_section_header.xml
@@ -21,6 +21,7 @@
     android:layout_height="@dimen/notification_section_header_height"
     android:focusable="true"
     android:clickable="true"
+       android:background="@drawable/qs_background_primary"
     >
 
     <com.android.systemui.statusbar.notification.row.NotificationBackgroundView
@@ -38,6 +39,7 @@
         android:layout_height="match_parent"
         android:gravity="center_vertical"
         android:orientation="horizontal"
+               android:background="@drawable/qs_background_primary"
         >
         <include layout="@layout/status_bar_notification_section_header_contents"/>
     </LinearLayout>

```

增加 status_bar_notification_section_header 的布局圆角背景 设置为 qs_background_primary

### 3.4ActivatableNotificationView.java 中圆角背景的修改

```
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/notification/row/ActivatableNotificationView.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/notification/row/ActivatableNotificationView.java
@@ -233,8 +233,8 @@ public abstract class ActivatableNotificationView extends ExpandableOutlineView
      * be useful in a configuration change.
      */
     protected void initBackground() {
-        mBackgroundNormal.setCustomBackground(R.drawable.notification_material_bg);
-        mBackgroundDimmed.setCustomBackground(R.drawable.notification_material_bg_dim);
+        mBackgroundNormal.setCustomBackground(R.drawable.qs_background_primary);
+        mBackgroundDimmed.setCustomBackground(R.drawable.qs_background_primary);
     }
 
     private final Runnable mTapTimeoutRunnable = new Runnable() {

```

在 ActivatableNotificationView 的 initBackground() 中为 mBackgroundDimmed 和 mBackgroundNormal 设置圆角背景

### 3.5NotificationStackScrollLayout.java 圆角布局的修改

```
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/notification/stack/NotificationStackScrollLayout.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/notification/stack/NotificationStackScrollLayout.java
@@ -100,6 +100,7 @@ import com.android.systemui.statusbar.AmbientPulseManager;
 import com.android.systemui.statusbar.CommandQueue;
 import com.android.systemui.statusbar.DragDownHelper.DragDownCallback;
 import com.android.systemui.statusbar.EmptyShadeView;

 import com.android.systemui.statusbar.NotificationLockscreenUserManager;
 import com.android.systemui.statusbar.NotificationRemoteInputManager;
 import com.android.systemui.statusbar.NotificationShelf;
@@ -153,7 +154,7 @@ import java.util.Comparator;
 import java.util.HashSet;
 import java.util.List;
 import java.util.function.BiConsumer;

 import javax.inject.Inject;
 import javax.inject.Named;
 
@@ -283,6 +284,7 @@ public class NotificationStackScrollLayout extends ViewGroup implements ScrollAd
     protected boolean mScrollingEnabled;
     protected FooterView mFooterView;
     protected EmptyShadeView mEmptyShadeView;
     private boolean mDismissAllInProgress;
     private boolean mFadeNotificationsOnDismiss;
 
@@ -608,6 +610,9 @@ public class NotificationStackScrollLayout extends ViewGroup implements ScrollAd
             }
         });
         dynamicPrivacyController.addListener(this);

     }
 
     private void updateDismissRtlSetting(boolean dismissRtl) {
@@ -624,7 +629,7 @@ public class NotificationStackScrollLayout extends ViewGroup implements ScrollAd
     @ShadeViewRefactor(RefactorComponent.SHADE_VIEW)
     protected void onFinishInflate() {
         super.onFinishInflate();

         inflateEmptyShadeView();
         inflateFooterView();
         mVisualStabilityManager.setVisibilityLocationProvider(this::isInVisibleLocation);
@@ -651,6 +656,7 @@ public class NotificationStackScrollLayout extends ViewGroup implements ScrollAd
     }
 
     private void reinflateViews() {

         inflateFooterView();
         inflateEmptyShadeView();
         updateFooter();
@@ -834,11 +840,11 @@ public class NotificationStackScrollLayout extends ViewGroup implements ScrollAd
         int right = (int) MathUtils.lerp(darkLeft, lockScreenRight, xProgress);
         int top = (int) MathUtils.lerp(darkTop, lockScreenTop, yProgress);
         int bottom = (int) MathUtils.lerp(darkTop, lockScreenBottom, yProgress);
-        mBackgroundAnimationRect.set(
+        /*mBackgroundAnimationRect.set(
                 left,
                 top,
                 right,
-                bottom);
+                bottom);*/
 
         int backgroundTopAnimationOffset = top - lockScreenTop;
         // TODO(kprevas): this may not be necessary any more since we don't display the shelf in AOD
@@ -850,7 +856,7 @@ public class NotificationStackScrollLayout extends ViewGroup implements ScrollAd
             }
         }
         if (!mAmbientState.isDark() || anySectionHasVisibleChild) {
-            drawBackgroundRects(canvas, left, right, top, backgroundTopAnimationOffset);
+            //drawBackgroundRects(canvas, left, right, top, backgroundTopAnimationOffset);
         }

```

在 reinflateViews() 去掉关于对背景的布局绘制 达到绘制圆角背景的目的

通过上面的修改 通知的背景修改为圆角布局
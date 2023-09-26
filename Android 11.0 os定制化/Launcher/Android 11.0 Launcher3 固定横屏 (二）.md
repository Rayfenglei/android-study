> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124890504)

11.0 定制化 launcher3 中，平板由于竖屏显示不太美观，所以需要固定横屏, 所以就需要在 Launcher.java 中来实现固定横屏功能  
1. 设置 Launche3 固定横屏

```
--- a/packages/apps/Launcher3/src/com/android/launcher3/Launcher.java
+++ b/packages/apps/Launcher3/src/com/android/launcher3/Launcher.java
@@ -19,7 +19,7 @@ package com.android.launcher3;
 import static android.content.pm.ActivityInfo.CONFIG_LOCALE;
 import static android.content.pm.ActivityInfo.CONFIG_ORIENTATION;
 import static android.content.pm.ActivityInfo.CONFIG_SCREEN_SIZE;
-
+import android.content.pm.ActivityInfo;
 import static com.android.launcher3.AbstractFloatingView.TYPE_DISCOVERY_BOUNCE;
 import static com.android.launcher3.AbstractFloatingView.TYPE_SNACKBAR;
 import static com.android.launcher3.LauncherAnimUtils.SPRING_LOADED_EXIT_DELAY;
@@ -31,7 +31,7 @@ import static com.android.launcher3.dragndrop.DragLayer.ALPHA_INDEX_LAUNCHER_LOA
 import static com.android.launcher3.dragndrop.DragLayer.ALPHA_INDEX_OVERLAY;
 import static com.android.launcher3.logging.LoggerUtils.newContainerTarget;
 import static com.android.launcher3.logging.LoggerUtils.newTarget;
-import static com.android.launcher3.states.RotationHelper.REQUEST_NONE;
+import static com.android.launcher3.states.RotationHelper.REQUEST_LOCK;
 import static com.android.launcher3.util.RaceConditionTracker.ENTER;
 import static com.android.launcher3.util.RaceConditionTracker.EXIT;
 import static com.sprd.ext.FeatureOption.SPRD_APP_REMOTE_ANIM_SUPPORT;
@@ -428,7 +428,7 @@ public class Launcher extends BaseDraggingActivity implements LauncherExterns,
    public void onEnterAnimationComplete() {
         super.onEnterAnimationComplete();
         UiFactory.onEnterAnimationComplete(this);
         mAllAppsController.highlightWorkTabIfNecessary();
         //设置锁定屏幕 不会随着屏幕的旋转而改变方向
-        mRotationHelper.setCurrentTransitionRequest(REQUEST_NONE);
+        mRotationHelper.setCurrentTransitionRequest(REQUEST_LOCK);
     }
 
     @Override
@@ -1025,6 +1025,7 @@ public class Launcher extends BaseDraggingActivity implements LauncherExterns,
    protected void onResume() {
         mAppMonitor.onLauncherResumed();
         TraceHelper.endSection("ON_RESUME");
         RaceConditionTracker.onEvent(ON_RESUME_EVT, EXIT);
         //屏幕方向固定横屏
+        setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE);
     }

```

```
默认屏幕不旋转

```

```
packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java

loadBooleanSetting(stmt, Settings.System.ACCELEROMETER_ROTATION,
        false);

```
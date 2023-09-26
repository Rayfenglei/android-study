> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127894063)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.Launcher3 去掉抽屉模式 双层改成单层 (二) 的核心类](#2.Launcher3%E5%8E%BB%E6%8E%89%E6%8A%BD%E5%B1%89%E6%A8%A1%E5%BC%8F%20%E5%8F%8C%E5%B1%82%E6%94%B9%E6%88%90%E5%8D%95%E5%B1%82%28%E4%BA%8C%29%E7%9A%84%E6%A0%B8%E5%BF%83%E7%B1%BB)

[3.Launcher3 去掉抽屉模式 双层改成单层 (二) 的核心功能分析和实现](#3.Launcher3%E5%8E%BB%E6%8E%89%E6%8A%BD%E5%B1%89%E6%A8%A1%E5%BC%8F%20%E5%8F%8C%E5%B1%82%E6%94%B9%E6%88%90%E5%8D%95%E5%B1%82%28%E4%BA%8C%29%E7%9A%84%E6%A0%B8%E5%BF%83%E5%8A%9F%E8%83%BD%E5%88%86%E6%9E%90%E5%92%8C%E5%AE%9E%E7%8E%B0)

 [3.1 AllAppsTransitionController.java 关于单层的相关修改](#t3)

[3.2 在 DragController.java 中关于拖拽的时候 判断是系统 app 去掉卸载功能](#t4)

[3.3 DeleteDropTarget.java 中删除改成取消](#t5)

[3.4 AllAppsList.java 关于显示安装 app 的相关属性](#t6)

[3.5 BaseModelUpdateTask.java 中添加关于更新 app 的相关方法](#t7)

[3.6 PackageUpdatedTask.java 中安装 app 更新 apps 列表的相关代码](#t8)

[3.7 PortraitStatesTouchController.java 关于单层取消上滑功能的修改](#t9)

1. 概述
-----

  在系统产品开发中，对于 Launcher3 系统原生是默认双层就是带抽屉的，产品开发需要需求要求  
改成单层的，所以就需要了解 Launcher3 关于 apps 绑定流程就可以了修改成单层，上一篇已经讲解了第一部分，接下来继续第二部分的修改

2.Launcher3 去掉抽屉模式 双层改成单层 (二) 的核心类
----------------------------------

```
packages/apps/Launcher3/src/com/android/launcher3/allapps/AllAppsTransitionController.java
packages/apps/Launcher3/src/com/android/launcher3/DeleteDropTarget.java
packages/apps/Launcher3/src/com/android/launcher3/dragndrop/DragController.java
packages/apps/Launcher3/src/com/android/launcher3/model/AllAppsList.java
packages/apps/Launcher3/src/com/android/launcher3/model/BaseModelUpdateTask.java
packages/apps/Launcher3/src/com/android/launcher3/model/PackageUpdatedTask.java
packages/apps/Launcher3/quickstep/src/com/android/launcher3/uioverrides/touchcontrollers/PortraitStatesTouchController.java
```

  
3.Launcher3 去掉抽屉模式 双层改成单层 (二) 的核心功能分析和实现
-------------------------------------------

  3.1 AllAppsTransitionController.java 关于单层的相关修改
------------------------------------------------

```
 --- a/packages/apps/Launcher3/src/com/android/launcher3/allapps/AllAppsTransitionController.java
+++ b/packages/apps/Launcher3/src/com/android/launcher3/allapps/AllAppsTransitionController.java
@@ -39,7 +39,7 @@ import com.android.launcher3.uioverrides.plugins.PluginManagerWrapper;
 import com.android.launcher3.views.ScrimView;
 import com.android.systemui.plugins.AllAppsSearchPlugin;
 import com.android.systemui.plugins.PluginListener;
-
+import com.android.launcher3.config.FeatureFlags;
 /**
  * Handles AllApps view transition.
  * 1) Slides all apps view using direct manipulation
@@ -157,6 +157,9 @@ public class AllAppsTransitionController implements StateHandler<LauncherState>,
     @Override
     public void setStateWithAnimation(LauncherState toState,
             StateAnimationConfig config, PendingAnimation builder) {
+        if(FeatureFlags.REMOVE_DRAWER){
+           return;
+        }
         float targetProgress = toState.getVerticalProgress(mLauncher);
         if (Float.compare(mProgress, targetProgress) == 0) {
             if (!config.onlyPlayAtomicComponent()) {
@@ -211,7 +214,7 @@ public class AllAppsTransitionController implements StateHandler<LauncherState>,
             setter.setViewAlpha(mAppsView.getContentView(), 0, allAppsFade);
             setter.setViewAlpha(mAppsView.getScrollBar(), 0, allAppsFade);
         }
-        mAppsView.getSearchUiManager().setContentVisibility(visibleElements, setter, allAppsFade);
+        //mAppsView.getSearchUiManager().setContentVisibility(visibleElements, setter, allAppsFade);
 
         setter.setInt(mScrimView, ScrimView.DRAG_HANDLE_ALPHA,
                 (visibleElements & VERTICAL_SWIPE_INDICATOR) != 0 ? 255 : 0, allAppsFade);
```

在关于 setStateWithAnimation(）的设置上滑动画的方法中，去掉上滑抽屉的动画

3.2 在 DragController.java 中关于拖拽的时候 判断是系统 app 去掉卸载功能
---------------------------------------------------

```
--- a/packages/apps/Launcher3/src/com/android/launcher3/dragndrop/DragController.java
+++ b/packages/apps/Launcher3/src/com/android/launcher3/dragndrop/DragController.java
@@ -44,7 +44,9 @@ import com.android.launcher3.model.data.ItemInfo;
 import com.android.launcher3.model.data.WorkspaceItemInfo;
 import com.android.launcher3.util.ItemInfoMatcher;
 import com.android.launcher3.util.TouchController;
-
+import com.android.launcher3.config.FeatureFlags;
+import com.android.launcher3.DeleteDropTarget;
+import com.android.launcher3.LauncherSettings;
 import java.util.ArrayList;
 
 /**
@@ -541,13 +543,20 @@ public class DragController implements DragDriver.EventListener, TouchController
                     dropTarget.onDrop(mDragObject, mOptions);
                 }
                 accepted = true;
+                if (FeatureFlags.REMOVE_DRAWER && dropTarget instanceof DeleteDropTarget &&
+                        isNeedCancelDrag(mDragObject.dragInfo)) {
+                    cancelDrag();
+                }
             }
         }
         final View dropTargetAsView = dropTarget instanceof View ? (View) dropTarget : null;
         mLauncher.getUserEventDispatcher().logDragNDrop(mDragObject, dropTargetAsView);
         dispatchDropComplete(dropTargetAsView, accepted);
     }
-
+    private boolean isNeedCancelDrag(ItemInfo item){
+        return (item.itemType == LauncherSettings.Favorites.ITEM_TYPE_APPLICATION ||
+            item.itemType == LauncherSettings.Favorites.ITEM_TYPE_FOLDER);
+    }
```

在 DragController.java 中的 drop(DropTarget dropTarget, Runnable flingAnimation) 执行  
拖拽动作时，通过 isNeedCancelDrag 进行判断是否可以卸载

3.3 DeleteDropTarget.java 中删除改成取消
---------------------------------

```
--- a/packages/apps/Launcher3/src/com/android/launcher3/DeleteDropTarget.java
+++ b/packages/apps/Launcher3/src/com/android/launcher3/DeleteDropTarget.java
@@ -38,6 +38,7 @@ import com.android.launcher3.model.data.WorkspaceItemInfo;
 import com.android.launcher3.userevent.nano.LauncherLogProto.ControlType;
 import com.android.launcher3.userevent.nano.LauncherLogProto.Target;
 import com.android.launcher3.views.Snackbar;
+import com.android.launcher3.config.FeatureFlags;
 
 public class DeleteDropTarget extends ButtonDropTarget {
 
@@ -102,6 +103,11 @@ public class DeleteDropTarget extends ButtonDropTarget {
             mText = getResources().getString(canRemove(item)
                     ? R.string.remove_drop_target_label
                     : android.R.string.cancel);
+            if(FeatureFlags.REMOVE_DRAWER){
+                mText = getResources().getString(isCanDrop(item)
+                        ? R.string.remove_drop_target_label
+                        : android.R.string.cancel);
+            }
             setContentDescription(mText);
             requestLayout();
         }
@@ -117,8 +123,15 @@ public class DeleteDropTarget extends ButtonDropTarget {
     private void setControlTypeBasedOnDragSource(ItemInfo item) {
         mControlType = item.id != ItemInfo.NO_ID ? ControlType.REMOVE_TARGET
                 : ControlType.CANCEL_TARGET;
+        if(FeatureFlags.REMOVE_DRAWER) {
+            mControlType = isCanDrop(item) ? ControlType.REMOVE_TARGET
+                    : ControlType.CANCEL_TARGET;
+        }
+    }
+    private boolean isCanDrop(ItemInfo item){
+        return !(item.itemType == LauncherSettings.Favorites.ITEM_TYPE_APPLICATION ||
+                item.itemType == LauncherSettings.Favorites.ITEM_TYPE_FOLDER);
     }
```

在 DeleteDropTarget 的 setTextBasedOnDragSource 和 setControlTypeBasedOnDragSource 中  
判断是否需要显示删除还是显示取消的字样

3.4 AllAppsList.java 关于显示安装 app 的相关属性
-------------------------------------

```
   --- a/packages/apps/Launcher3/src/com/android/launcher3/model/AllAppsList.java
+++ b/packages/apps/Launcher3/src/com/android/launcher3/model/AllAppsList.java
@@ -64,7 +64,7 @@ public class AllAppsList {
 
     /** The list off all apps. */
     public final ArrayList<AppInfo> data = new ArrayList<>(DEFAULT_APPLICATIONS_NUMBER);
-
+    public ArrayList<AppInfo> added = new ArrayList<>(DEFAULT_APPLICATIONS_NUMBER);
     private IconCache mIconCache;
     private AppFilter mAppFilter;
 
@@ -142,6 +142,7 @@ public class AllAppsList {
         info.sectionName = mIndex.computeSectionName(info.title);
 
         data.add(info);
+        added.add(info);
         mDataChanged = true;
     }
 
@@ -155,6 +156,7 @@ public class AllAppsList {
             info.sectionName = mIndex.computeSectionName(info.title);
 
             data.add(info);
+            added.add(info);
             mDataChanged = true;
         }
     }
@@ -189,6 +191,7 @@ public class AllAppsList {
 
     public void clear() {
         data.clear();
+        added.clear();
         mDataChanged = false;
         // Reset the index as locales might have changed
         mIndex = new AlphabeticIndexCompat(LocaleList.getDefault());
```

通过在 AllAppsList.java 中添加全局属性 added 来保存安装的 app 的信息 这样可以在安装  
app 以后绑定到 workspaces 中

3.5 BaseModelUpdateTask.java 中添加关于更新 app 的相关方法
----------------------------------------------

```
  --- a/packages/apps/Launcher3/src/com/android/launcher3/model/BaseModelUpdateTask.java
+++ b/packages/apps/Launcher3/src/com/android/launcher3/model/BaseModelUpdateTask.java
@@ -27,7 +27,7 @@ import com.android.launcher3.model.data.WorkspaceItemInfo;
 import com.android.launcher3.util.ComponentKey;
 import com.android.launcher3.util.ItemInfoMatcher;
 import com.android.launcher3.widget.WidgetListRowEntry;
-
+import com.android.launcher3.config.FeatureFlags;
 import java.util.ArrayList;
 import java.util.HashMap;
 import java.util.concurrent.Executor;
@@ -62,7 +62,9 @@ public abstract class BaseModelUpdateTask implements ModelUpdateTask {
                 Log.d(TAG, "Ignoring model task since loader is pending=" + this);
             }
             // Loader has not yet run.
-            return;
+            if(!FeatureFlags.REMOVE_DRAWER){
+                               return;
+                       }
         }
         execute(mApp, mDataModel, mAllAppsList);
     }
@@ -115,10 +117,10 @@ public abstract class BaseModelUpdateTask implements ModelUpdateTask {
     }
 
     public void bindApplicationsIfNeeded() {
-        if (mAllAppsList.getAndResetChangeFlag()) {
+        /*if (mAllAppsList.getAndResetChangeFlag()) {*/
             AppInfo[] apps = mAllAppsList.copyData();
             int flags = mAllAppsList.getFlags();
             scheduleCallbackTask(c -> c.bindAllApplications(apps, flags));
-        }
+        //}
     }
 }
```

通过在 BaseModelUpdateTask.java 中的 run() 在单层的时候注释掉 return , 可以继续加载 app  
在 bindApplicationsIfNeeded() 中通过注释掉判断条件，所有的 app 都绑定到 workspace 中

3.6 PackageUpdatedTask.java 中安装 app 更新 apps 列表的相关代码
---------------------------------------------------

```
 --- a/packages/apps/Launcher3/src/com/android/launcher3/model/PackageUpdatedTask.java
+++ b/packages/apps/Launcher3/src/com/android/launcher3/model/PackageUpdatedTask.java
@@ -50,7 +50,10 @@ import com.android.launcher3.util.ItemInfoMatcher;
 import com.android.launcher3.util.PackageManagerHelper;
 import com.android.launcher3.util.PackageUserKey;
 import com.android.launcher3.util.SafeCloseable;
-
+import android.util.Pair;
+import android.content.pm.LauncherActivityInfo;
+import java.util.List;
+import com.android.launcher3.model.data.AppInfo;
 import java.util.ArrayList;
 import java.util.Arrays;
 import java.util.Collections;
@@ -169,6 +172,11 @@ public class PackageUpdatedTask extends BaseModelUpdateTask {
         }
 
         bindApplicationsIfNeeded();
+        if(FeatureFlags.REMOVE_DRAWER){
+            final ArrayList<AppInfo> addedOrModified = new ArrayList<>();
+            addedOrModified.addAll(appsList.added);
+            updateToWorkSpace(context, app, appsList);
+        }
 
         final IntSparseArrayMap<Boolean> removedShortcuts = new IntSparseArrayMap<>();
 
@@ -335,7 +343,30 @@ public class PackageUpdatedTask extends BaseModelUpdateTask {
             bindUpdatedWidgets(dataModel);
         }
     }
-
+    public void updateToWorkSpace( Context context, LauncherAppState app , AllAppsList appsList){
+        if(FeatureFlags.REMOVE_DRAWER){
+            ArrayList<Pair<ItemInfo, Object>> install_queue = new ArrayList<>();
+            final List<UserHandle> profiles = UserCache.INSTANCE.get(context).getUserProfiles();
+            ArrayList<InstallShortcutReceiver.PendingInstallShortcutInfo> added = new ArrayList<InstallShortcutReceiver.PendingInstallShortcutInfo>();
+            for (UserHandle user : profiles) {
+                final List<LauncherActivityInfo> applications = context.getSystemService(LauncherApps.class).getActivityList(null, user);
+                synchronized (this) {
+                    for (LauncherActivityInfo info : applications) {
+                        for (AppInfo appInfo : appsList.added) {
+                            if(info.getComponentName().equals(appInfo.componentName)){
+                                InstallShortcutReceiver.PendingInstallShortcutInfo mPendingInstallShortcutInfo =  new InstallShortcutReceiver.PendingInstallShortcutInfo(info,context);
+                                added.add(mPendingInstallShortcutInfo);
+                                install_queue .add(mPendingInstallShortcutInfo.getItemInfo());
+                            }
+                        }
+                    }
+                }
+            }
+            if (!added.isEmpty()) {
+                LauncherAppState.getInstance(context).getModel().addAndBindAddedWorkspaceItems(install_queue );
+            }
+        }
+    }
```

在 PackageUpdatedTask.java 中关于安装更新 app 时在 execute(LauncherAppState app, BgDataModel dataModel, AllAppsList appsList) 方法的  
bindApplicationsIfNeeded(); 方法之后添加 app 安装以后更新 apps 列表时 绑定新安装的 app 显示在 workspace 的 app 列表页中

3.7 PortraitStatesTouchController.java 关于单层取消上滑功能的修改
----------------------------------------------------

```
  --- a/packages/apps/Launcher3/quickstep/src/com/android/launcher3/uioverrides/touchcontrollers/PortraitStatesTouchController.java
+++ b/packages/apps/Launcher3/quickstep/src/com/android/launcher3/uioverrides/touchcontrollers/PortraitStatesTouchController.java
@@ -37,7 +37,7 @@ import android.animation.ValueAnimator;
 import android.util.Log;
 import android.view.MotionEvent;
 import android.view.animation.Interpolator;
-
+import com.android.launcher3.config.FeatureFlags;
 import com.android.launcher3.DeviceProfile;
 import com.android.launcher3.Launcher;
 import com.android.launcher3.LauncherState;
@@ -161,9 +161,15 @@ public class PortraitStatesTouchController extends AbstractStateChangeTouchContr
                         "PortraitStatesTouchController.getTargetState 3");
             }
             int stateFlags = SystemUiProxy.INSTANCE.get(mLauncher).getLastSystemUiStateFlags();
-            return mAllowDragToOverview && TouchInteractionService.isConnected()
+            if(FeatureFlags.REMOVE_DRAWER){
+                           return mAllowDragToOverview && TouchInteractionService.isConnected()
+                    && (stateFlags & SYSUI_STATE_OVERVIEW_DISABLED) == 0
+                    ? OVERVIEW : NORMAL;
+                       }else{
+                               return mAllowDragToOverview && TouchInteractionService.isConnected()
                     && (stateFlags & SYSUI_STATE_OVERVIEW_DISABLED) == 0
                     ? OVERVIEW : ALL_APPS;
+                       }
         }
         return fromState;
     }
```

在 PortraitStatesTouchController.java 中的 getTargetState(LauncherState fromState, boolean isDragTowardPositive) 中  
判断返回的是 NORMAL 就相当于取消上滑事件无返回 app 值列表，返回 ALL_APPS 就是返回 apps 列表相应上滑事件
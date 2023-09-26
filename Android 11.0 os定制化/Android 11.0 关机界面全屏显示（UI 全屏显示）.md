> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124760517)

### 1. 概述

在 11.0 的系统定制化开发中，原生系统关机界面 UI 是靠右边显示的，但是客户需求要求全屏显示 重启和关机功能键居中显示，所以就涉及到调整 UI 然后全屏显示，需要实现窗口的全局布局实现全屏功能

### 2. 关机界面全屏显示（UI 全屏显示）的核心类

```
frameworks / base / packages / SystemUI / src / com / android / systemui / globalactions / GlobalActionsDialog.java

```

### 3. 关机界面全屏显示（UI 全屏显示）的核心功能分析

经过 [adb](https://so.csdn.net/so/search?q=adb&spm=1001.2101.3001.7020) shell 命令查看系统关机界面的布局 UI 就是 GlobalActionsDialog.java 就是长按 power 弹出的关机界面

### 3.1 GlobalActionsDialog.java 的核心功能分析

```
private static final class ActionsDialog extends Dialog implements DialogInterface,
              ColorExtractor.OnColorsChangedListener {
  
          private final Context mContext;
          private final MyAdapter mAdapter;
          private final IStatusBarService mStatusBarService;
          private final IBinder mToken = new Binder();
          private MultiListLayout mGlobalActionsLayout;
          private Drawable mBackgroundDrawable;
          private final SysuiColorExtractor mColorExtractor;
          private final GlobalActionsPanelPlugin.PanelViewController mPanelController;
          private boolean mKeyguardShowing;
          private boolean mShowing;
          private float mScrimAlpha;
          private ResetOrientationData mResetOrientationData;
  
          ActionsDialog(Context context, MyAdapter adapter,
                  GlobalActionsPanelPlugin.PanelViewController plugin) {
              super(context, com.android.systemui.R.style.Theme_SystemUI_Dialog_GlobalActions);
              mContext = context;
              mAdapter = adapter;
              mColorExtractor = Dependency.get(SysuiColorExtractor.class);
              mStatusBarService = Dependency.get(IStatusBarService.class);
  
              // Window initialization
              Window window = getWindow();
              window.requestFeature(Window.FEATURE_NO_TITLE);
              // Inflate the decor view, so the attributes below are not overwritten by the theme.
              window.getDecorView();
              window.getAttributes().systemUiVisibility |= View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
                      | View.SYSTEM_UI_FLAG_LAYOUT_STABLE
                      | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION;
              window.setLayout(MATCH_PARENT, MATCH_PARENT);
              window.clearFlags(WindowManager.LayoutParams.FLAG_DIM_BEHIND);
              window.addFlags(
                      WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN
                      | WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL
                      | WindowManager.LayoutParams.FLAG_LAYOUT_INSET_DECOR
                      | WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED
                      | WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH
                      | WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED);
              window.setType(WindowManager.LayoutParams.TYPE_VOLUME_OVERLAY);
              setTitle(R.string.global_actions);
  
              mPanelController = plugin;
              initializeLayout();
          }
  
          private boolean shouldUsePanel() {
              return mPanelController != null && mPanelController.getPanelContent() != null;
          }
  
          private void initializePanel() {
              int rotation = RotationUtils.getRotation(mContext);
              boolean rotationLocked = RotationPolicy.isRotationLocked(mContext);
              if (rotation != RotationUtils.ROTATION_NONE) {
                  if (rotationLocked) {
                      if (mResetOrientationData == null) {
                          mResetOrientationData = new ResetOrientationData();
                          mResetOrientationData.locked = true;
                          mResetOrientationData.rotation = rotation;
                      }
  
                      // Unlock rotation, so user can choose to rotate to portrait to see the panel.
                      // This call is posted so that the rotation does not change until post-layout,
                      // otherwise onConfigurationChanged() may not get invoked.
                      mGlobalActionsLayout.post(() ->
                              RotationPolicy.setRotationLockAtAngle(
                                      mContext, false, RotationUtils.ROTATION_NONE));
                  }
              } else {
                  if (!rotationLocked) {
                      if (mResetOrientationData == null) {
                          mResetOrientationData = new ResetOrientationData();
                          mResetOrientationData.locked = false;
                      }
  
                      // Lock to portrait, so the user doesn't accidentally hide the panel.
                      // This call is posted so that the rotation does not change until post-layout,
                      // otherwise onConfigurationChanged() may not get invoked.
                      mGlobalActionsLayout.post(() ->
                              RotationPolicy.setRotationLockAtAngle(
                                      mContext, true, RotationUtils.ROTATION_NONE));
                  }
  
                  // Disable rotation suggestions, if enabled
                  setRotationSuggestionsEnabled(false);
  
                  FrameLayout panelContainer =
                          findViewById(com.android.systemui.R.id.global_actions_panel_container);
                  FrameLayout.LayoutParams panelParams =
                          new FrameLayout.LayoutParams(
                                  FrameLayout.LayoutParams.MATCH_PARENT,
                                  FrameLayout.LayoutParams.MATCH_PARENT);
                  panelContainer.addView(mPanelController.getPanelContent(), panelParams);
                  mBackgroundDrawable = mPanelController.getBackgroundDrawable();
                  mScrimAlpha = 1f;
              }
          }
  
          private void initializeLayout() {
              setContentView(getGlobalActionsLayoutId(mContext));
              fixNavBarClipping();
              mGlobalActionsLayout = findViewById(com.android.systemui.R.id.global_actions_view);
              mGlobalActionsLayout.setOutsideTouchListener(view -> dismiss());
              ((View) mGlobalActionsLayout.getParent()).setOnClickListener(view -> dismiss());
              mGlobalActionsLayout.setListViewAccessibilityDelegate(new View.AccessibilityDelegate() {
                  @Override
                  public boolean dispatchPopulateAccessibilityEvent(
                          View host, AccessibilityEvent event) {
                      // Populate the title here, just as Activity does
                      event.getText().add(mContext.getString(R.string.global_actions));
                      return true;
                  }
              });
              mGlobalActionsLayout.setRotationListener(this::onRotate);
              mGlobalActionsLayout.setAdapter(mAdapter);
  
              if (shouldUsePanel()) {
                  initializePanel();
              }
              if (mBackgroundDrawable == null) {
                  mBackgroundDrawable = new ScrimDrawable();
                  mScrimAlpha = ScrimController.GRADIENT_SCRIM_ALPHA;
              }
              getWindow().setBackgroundDrawable(mBackgroundDrawable);
          }

```

而在 ActionsDialog(）中的构造方法中，通过 Window 窗口的相关属性来设置关机窗口弹出的  
布局从源码中可以看出 window.getAttributes().systemUiVisibility 就是设置 window 窗口的属性  
所以增加全屏属性 修改为：

```
window.getAttributes().systemUiVisibility |= View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN

                     | View.SYSTEM_UI_FLAG_LAYOUT_STABLE

+                                       | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION

+                                       | View.SYSTEM_UI_FLAG_FULLSCREEN

+                                       | View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY

                     | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION;

```

### 3.2 关机弹窗中关机 重启功能键居中

global_actions_column.xml  
就是布局界面 增加居中属性修改为：

```
--- a/frameworks/base/packages/SystemUI/res/layout-land/global_actions_column.xml

+++ b/frameworks/base/packages/SystemUI/res/layout-land/global_actions_column.xml

@@ -19,18 +19,18 @@

     xmlns:android="http://schemas.android.com/apk/res/android"

     android:id="@id/global_actions_view"

     android:layout_width="match_parent"

-    android:layout_height="match_parent"

+    android:layout_height="1200px"

     android:orientation="horizontal"

     android:clipToPadding="false"

     android:theme="@style/qs_theme"

-    android:gravity="center_horizontal | top"

+    android:gravity="center"

     android:clipChildren="false"

+       android:background="@drawable/bg_action"

 >

-    <LinearLayout

-        android:layout_height="wrap_content"

-        android:layout_width="wrap_content"

+    <RelativeLayout

+        android:layout_height="match_parent"

+        android:layout_width="match_parent"

         android:padding="0dp"

-        android:orientation="horizontal"

         android:clipChildren="false"

         android:clipToPadding="false"

     >

@@ -40,13 +40,8 @@

             android:layout_width="wrap_content"

             android:layout_height="wrap_content"

             android:orientation="horizontal"

-            android:layout_marginTop="@dimen/global_actions_grid_side_margin"

+                       android:layout_centerInParent="true"

             android:translationZ="@dimen/global_actions_translate"

-            android:paddingLeft="@dimen/global_actions_grid_vertical_padding"

-            android:paddingRight="@dimen/global_actions_grid_vertical_padding"

-            android:paddingTop="@dimen/global_actions_grid_horizontal_padding"

-            android:paddingBottom="@dimen/global_actions_grid_horizontal_padding"

-            android:background="?android:attr/colorBackgroundFloating"

         />

         <!-- For separated items-->

         <LinearLayout

@@ -60,9 +55,10 @@

             android:paddingTop="@dimen/global_actions_grid_horizontal_padding"

             android:paddingBottom="@dimen/global_actions_grid_horizontal_padding"

             android:orientation="horizontal"

+                       android:visibility="gone"

             android:background="?android:attr/colorBackgroundFloating"

             android:translationZ="@dimen/global_actions_translate"

         />

-    </LinearLayout>

+    </RelativeLayout>

  

</com.android.systemui.globalactions.GlobalActionsColumnLayout>

```

### 3.3 去掉关机弹窗中的关机和重启白色的背景色

白色背景布局就是 GlobalActionsLayout.java 接下来看 GlobalActionsLayout.java 的源码分析去掉白色背景

```
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/globalactions/GlobalActionsLayout.java

+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/globalactions/GlobalActionsLayout.java

@@ -50,7 +50,7 @@ public abstract class GlobalActionsLayout extends MultiListLayout {

         HardwareBgDrawable separatedBackground = new HardwareBgDrawable(true, true, getContext());

         listBackground.setTint(gridBackgroundColor);

         separatedBackground.setTint(separatedBackgroundColor);

-        getListView().setBackground(listBackground);

+        //getListView().setBackground(listBackground);

         getSeparatedView().setBackground(separatedBackground);

     }

```
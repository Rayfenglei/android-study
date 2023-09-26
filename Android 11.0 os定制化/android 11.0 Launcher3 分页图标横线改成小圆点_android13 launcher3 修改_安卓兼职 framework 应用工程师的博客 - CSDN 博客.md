> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124870345)

### 1. 概述

在 11.0 的 Launcher3 中，双层页面中滑动 workspace 时，分页线是一条横线 看起来不太美观，而单层页面分页线却是个小圆点  
所以改成小圆点还是比较好看点

### 2.Launcher3 分页图标横线改成小圆点的核心类

```
packages/apps/Launcher3/res/layout/launcher.xml
packages/apps/Launcher3/src/com/android/launcher3/Launcher.java
packages/apps/Launcher3/src/com/android/launcher3/Workspace.java
packages/apps/Launcher3/src/com/android/launcher3/pageindicators/PageIndicatorDots.java
packages/apps/Launcher3/quickstep/src/com/android/launcher3/QuickstepAppTransitionManagerImpl.java

```

### 3.Launcher3 分页图标横线改成小圆点的核心功能分析和实现

### 1. 先看 Launcher3 布局的 xml

res/layout/[launcher](https://so.csdn.net/so/search?q=launcher&spm=1001.2101.3001.7020).xml

…

```
        <!-- The workspace contains 5 screens of cells -->
        <!-- DO NOT CHANGE THE ID -->
        <com.android.launcher3.Workspace
            android:id="@+id/workspace"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:layout_gravity="center"
            android:theme="@style/HomeScreenElementTheme"
            launcher:pageIndicator="@+id/page_indicator" />

        <include
            android:id="@+id/overview_panel_container"
            layout="@layout/overview_panel"
            android:visibility="gone" />

        <!-- Keep these behind the workspace so that they are not visible when
         we go into AllApps -->
        <com.android.launcher3.pageindicators.WorkspacePageIndicator
            android:id="@+id/page_indicator"
            android:layout_width="match_parent"
            android:layout_height="4dp"
            android:layout_gravity="bottom|center_horizontal"
            android:theme="@style/HomeScreenElementTheme" />

   ...
</com.android.launcher3.LauncherRootView>

```

从 xml 可以看出这个 WorkspacePageIndicator 分页线  
而在 com.android.launcher3.pageindicators 下发现还有个 PageIndicatorDots 这个就是单层时的小圆点 分页线

### 2. 替换圆点分页线

所以把 WorkspacePageIndicator 给替换成 PageIndicatorDots

```
<!-- Keep these behind the workspace so that they are not visible when
 we go into AllApps -->
<com.android.launcher3.pageindicators.PageIndicatorDots
    android:id="@+id/page_indicator"
    android:layout_width="match_parent"
    android:layout_height="4dp"
    android:layout_gravity="bottom|center_horizontal"
    android:theme="@style/HomeScreenElementTheme" />

```

### 3. 修改 workspace 中 PagedView 的相关代码

PagedView

```
/**
 * The workspace is a wide area with a wallpaper and a finite number of pages.
 * Each page contains a number of icons, folders or widgets the user can
 * interact with. A workspace is meant to be used with a fixed width only.
 */
public class Workspace extends PagedView<PageIndicatorDots>
        implements DropTarget, DragSource, View.OnTouchListener,
        DragController.DragListener, Insettable, LauncherStateManager.StateHandler {
    private static final String TAG = "Launcher.Workspace";
    修改PagedView<WorkspacePageIndicator>为PagedView<PageIndicatorDots>

```

注释报错的地方 就是圆点类 没有的方法注释掉  
在 Launcher.java 中注释掉 WorkspacePageIndicator 的 setShouldAutoHide 的调用地方

```
@@ -1012,7 +1012,7 @@ public class Launcher extends StatefulActivity<LauncherState> implements Launche
    @Override
    public void onStateSetStart(LauncherState state) {
        super.onStateSetStart(state);
        if (mDeferOverlayCallbacks) {
            scheduleDeferredCheck();
        }
        addActivityFlags(ACTIVITY_STATE_TRANSITION_ACTIVE);

        if (state.hasFlag(FLAG_CLOSE_POPUPS)) {
            AbstractFloatingView.closeAllOpenViews(this, !state.hasFlag(FLAG_NON_INTERACTIVE));
        }

        if (state == SPRING_LOADED) {
            // Prevent any Un/InstallShortcutReceivers from updating the db while we are
            // not on homescreen
            InstallShortcutReceiver.enableInstallQueue(FLAG_DRAG_AND_DROP);
            getRotationHelper().setCurrentStateRequest(REQUEST_LOCK);

            mWorkspace.showPageIndicatorAtCurrentScroll();
            mWorkspace.setClipChildren(false);
        }
         // When multiple pages are visible, show persistent page indicator
-        mWorkspace.getPageIndicator().setShouldAutoHide(!state.hasFlag(FLAG_MULTI_PAGE));
+        //mWorkspace.getPageIndicator().setShouldAutoHide(!state.hasFlag(FLAG_MULTI_PAGE));
     }

```

### 4. 修改 PageIndicatorDots 的 setInset 方法

public class PageIndicatorDots extends View implements Insettable, PageIndicator {

然后直接把 WorkspacePageIndicator 的 setInsets copy 过来

```
@Override
public void setInsets(Rect insets) {
DeviceProfile grid = mLauncher.getDeviceProfile();
FrameLayout.LayoutParams lp = (FrameLayout.LayoutParams) getLayoutParams();

if (grid.isVerticalBarLayout()) {
    Rect padding = grid.workspacePadding;
    lp.leftMargin = padding.left + grid.workspaceCellPaddingXPx;
    lp.rightMargin = padding.right + grid.workspaceCellPaddingXPx;
    lp.bottomMargin = padding.bottom;
} else {
    lp.leftMargin = lp.rightMargin = 0;
    lp.gravity = Gravity.CENTER_HORIZONTAL | Gravity.BOTTOM;
    lp.bottomMargin = grid.hotseatBarSizePx + insets.bottom;
}
setLayoutParams(lp);

}

```

最后在构造方法初始化 mLauncher

```
public PageIndicatorDots(Context context, AttributeSet attrs, int defStyleAttr) {
super(context, attrs, defStyleAttr);
mLauncher = Launcher.getLauncher(context);
mCirclePaint = new Paint(Paint.ANTI_ALIAS_FLAG);

```

在设置里将 Launcher3 的缓存清掉后, 有时会发生奔溃问题  
检查发现 PageIndicatorDots setScroll 的 totalScroll 有时会为 0 导致 scrollPerPage 会有分母为 0 的情况

```
  @Override
    public void setScroll(int currentScroll, int totalScroll) {
        if (mNumPages > 1) {
            if (mIsRtl) {
                currentScroll = totalScroll - currentScroll;
            }
            int scrollPerPage = totalScroll / (mNumPages - 1);
            //add core
            if(scrollPerPage == 0)
                return;
            //end core
            int pageToLeft = currentScroll / scrollPerPage;
            int pageToLeftScroll = pageToLeft * scrollPerPage;
            int pageToRightScroll = pageToLeftScroll + scrollPerPage;

            float scrollThreshold = SHIFT_THRESHOLD * scrollPerPage;
            if (currentScroll < pageToLeftScroll + scrollThreshold) {
                // scroll is within the left page's threshold
                animateToPosition(pageToLeft);
            } else if (currentScroll > pageToRightScroll - scrollThreshold) {

```

### 5. 在 packages/apps/Launcher3/quickstep/src/com/android/launcher3/QuickstepAppTransitionManagerImpl.java 中 注释掉关于 WorkspacePageIndicator 的关于动画的调用方法

```
 /**
      * Content is everything on screen except the background and the floating view (if any).
      *
      * @param isAppOpening True when this is called when an app is opening.
      *                     False when this is called when an app is closing.
      * @param trans Array that contains the start and end translation values for the content.
      */
     private Pair<AnimatorSet, Runnable> getLauncherContentAnimator(boolean isAppOpening,
             float[] trans) {
         AnimatorSet launcherAnimator = new AnimatorSet();
         Runnable endListener;
 
         float[] alphas = isAppOpening
                 ? new float[] {1, 0}
                 : new float[] {0, 1};
 
         if (mLauncher.isInState(ALL_APPS)) {
             // All Apps in portrait mode is full screen, so we only animate AllAppsContainerView.
             final View appsView = mLauncher.getAppsView();
             final float startAlpha = appsView.getAlpha();
             final float startY = appsView.getTranslationY();
             appsView.setAlpha(alphas[0]);
             appsView.setTranslationY(trans[0]);
 
             ObjectAnimator alpha = ObjectAnimator.ofFloat(appsView, View.ALPHA, alphas);
             alpha.setDuration(CONTENT_ALPHA_DURATION);
             alpha.setInterpolator(LINEAR);
             appsView.setLayerType(View.LAYER_TYPE_HARDWARE, null);
             alpha.addListener(new AnimatorListenerAdapter() {
                 @Override
                 public void onAnimationEnd(Animator animation) {
                     appsView.setLayerType(View.LAYER_TYPE_NONE, null);
                 }
             });
             ObjectAnimator transY = ObjectAnimator.ofFloat(appsView, View.TRANSLATION_Y, trans);
             transY.setInterpolator(AGGRESSIVE_EASE);
             transY.setDuration(CONTENT_TRANSLATION_DURATION);
 
             launcherAnimator.play(alpha);
             launcherAnimator.play(transY);
 
             endListener = () -> {
                 appsView.setAlpha(startAlpha);
                 appsView.setTranslationY(startY);
                 appsView.setLayerType(View.LAYER_TYPE_NONE, null);
             };
         } else if (mLauncher.isInState(OVERVIEW)) {
             AllAppsTransitionController allAppsController = mLauncher.getAllAppsController();
             launcherAnimator.play(ObjectAnimator.ofFloat(allAppsController, ALL_APPS_PROGRESS,
                     allAppsController.getProgress(), ALL_APPS_PROGRESS_OFF_SCREEN));
             endListener = composeViewContentAnimator(launcherAnimator, alphas, trans);
         } else {
             mDragLayerAlpha.setValue(alphas[0]);
             ObjectAnimator alpha =
                     ObjectAnimator.ofFloat(mDragLayerAlpha, MultiValueAlpha.VALUE, alphas);
             alpha.setDuration(CONTENT_ALPHA_DURATION);
             alpha.setInterpolator(LINEAR);
             launcherAnimator.play(alpha);
 
             Workspace workspace = mLauncher.getWorkspace();
             View currentPage = ((CellLayout) workspace.getChildAt(workspace.getCurrentPage()))
                     .getShortcutsAndWidgets();
             View hotseat = mLauncher.getHotseat();
             View qsb = mLauncher.findViewById(R.id.search_container_all_apps);
 
             currentPage.setLayerType(View.LAYER_TYPE_HARDWARE, null);
             hotseat.setLayerType(View.LAYER_TYPE_HARDWARE, null);
             qsb.setLayerType(View.LAYER_TYPE_HARDWARE, null);
 
             launcherAnimator.play(ObjectAnimator.ofFloat(currentPage, View.TRANSLATION_Y, trans));
             launcherAnimator.play(ObjectAnimator.ofFloat(hotseat, View.TRANSLATION_Y, trans));
             launcherAnimator.play(ObjectAnimator.ofFloat(qsb, View.TRANSLATION_Y, trans));
 
             // Pause page indicator animations as they lead to layer trashing.
             mLauncher.getWorkspace().getPageIndicator().pauseAnimations();
 
             endListener = () -> {
                 currentPage.setTranslationY(0);
                 hotseat.setTranslationY(0);
                 qsb.setTranslationY(0);
 
                 currentPage.setLayerType(View.LAYER_TYPE_NONE, null);
                 hotseat.setLayerType(View.LAYER_TYPE_NONE, null);
                 qsb.setLayerType(View.LAYER_TYPE_NONE, null);
 
                 mDragLayerAlpha.setValue(1f);
                 mLauncher.getWorkspace().getPageIndicator().skipAnimationsToEnd();
             };
         }
         return new Pair<>(launcherAnimator, endListener);
     }

```

在 getLauncherContentAnimator 中关于换页的动画调用  
所以需要注释掉相关方法修改如下:

```
--- a/packages/apps/Launcher3/quickstep/src/com/android/launcher3/QuickstepAppTransitionManagerImpl.java
+++ b/packages/apps/Launcher3/quickstep/src/com/android/launcher3/QuickstepAppTransitionManagerImpl.java
@@ -411,7 +411,7 @@ public abstract class QuickstepAppTransitionManagerImpl extends LauncherAppTrans
             launcherAnimator.play(ObjectAnimator.ofFloat(qsb, View.TRANSLATION_Y, trans));
 
             // Pause page indicator animations as they lead to layer trashing.
-            mLauncher.getWorkspace().getPageIndicator().pauseAnimations();
+            //mLauncher.getWorkspace().getPageIndicator().pauseAnimations();
 
             endListener = () -> {
                 currentPage.setTranslationY(0);
@@ -423,7 +423,7 @@ public abstract class QuickstepAppTransitionManagerImpl extends LauncherAppTrans
                 qsb.setLayerType(View.LAYER_TYPE_NONE, null);
 
                 mDragLayerAlpha.setValue(1f);
-                mLauncher.getWorkspace().getPageIndicator().skipAnimationsToEnd();
+                //mLauncher.getWorkspace().getPageIndicator().skipAnimationsToEnd();
             };
         }
         return new Pair<>(launcherAnimator, endListener);

```
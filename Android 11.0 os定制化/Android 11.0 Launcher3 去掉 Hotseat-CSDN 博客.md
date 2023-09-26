> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124695669)

### 1. 概述

在 11.0 产品定制化开发中 由产品需求 Launcher3 页面布局的原因，要求 Launcher3 需要去掉 Hotseat 不显示 Hotseat 下面几个图标，而做满屏 app 的显示，从而达到美观的效果，下面就来分析去掉 Hotseat 从而实现这个功能

### 2.Launcher3 去掉 Hotseat 的核心类

```
packages/apps/Launcher3/res/layout/launcher.xml
packages/apps/Launcher3/src/com/android/launcher3/DeviceProfile.java

```

### 3.Launcher3 去掉 Hotseat 的核心功能分析和实现

在 Launcher3 中主页面就是 launcher.[xml](https://so.csdn.net/so/search?q=xml&spm=1001.2101.3001.7020) 布局，hotseat 布局也在这里面，所以隐藏 hotseat 可以从这里  
先看 launcher.xml 的布局  
首先看下 launcher.xml 的布局

### 3.1 launcher.xml 中 hotseat 布局

```
<com.android.launcher3.LauncherRootView
xmlns:android="http://schemas.android.com/apk/res/android"
xmlns:launcher="http://schemas.android.com/apk/res-auto"
android:id="@+id/launcher"
android:layout_width="match_parent"
android:layout_height="match_parent"
android:fitsSystemWindows="true">
<com.android.launcher3.dragndrop.DragLayer
    android:id="@+id/drag_layer"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:clipChildren="false"
    android:clipToPadding="false"
    android:importantForAccessibility="no">

    <!-- The workspace contains 5 screens of cells -->
    <!-- DO NOT CHANGE THE ID -->
    <com.android.launcher3.Workspace
        android:id="@+id/workspace"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_gravity="center"
        android:theme="@style/HomeScreenElementTheme"
        launcher:pageIndicator="@+id/page_indicator" />

    <include layout="@layout/memoryinfo_ext" />

    <!-- DO NOT CHANGE THE ID -->
    <include
        android:id="@+id/hotseat"
        layout="@layout/hotseat" />

    <include
        android:id="@+id/overview_panel"
        layout="@layout/overview_panel"
        android:visibility="gone" />

    <!-- Keep these behind the workspace so that they are not visible when
     we go into AllApps -->
    <com.sprd.ext.pageindicators.WorkspacePageIndicatorLine
        android:id="@+id/page_indicator"
        android:layout_width="match_parent"
        android:layout_height="@dimen/vertical_drag_handle_size"
        android:layout_gravity="bottom"
        android:theme="@style/HomeScreenElementTheme" />

    <include
        android:id="@+id/page_indicator_customize"
        layout="@layout/page_indicator_customize" />

    <include
        android:id="@+id/drop_target_bar"
        layout="@layout/drop_target_bar" />

    <include
        android:id="@+id/scrim_view"
        layout="@layout/scrim_view" />

    <include
        android:id="@+id/apps_view"
        layout="@layout/all_apps"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</com.android.launcher3.dragndrop.DragLayer>

</com.android.launcher3.LauncherRootView>

```

从布局中可以看到 android:id="@+id/hotseat" 就是 hotseat 布局，所以隐藏 hotseat  
就是需要设置属性为 gone

```
<include
        android:id="@+id/hotseat"
        layout="@layout/hotseat"
        android:visibility="gone" />

```

### 3.2DeviceProfile.java 关于 hotseat 高度的修改

```
public DeviceProfile(Context context, InvariantDeviceProfile inv,
Point minSize, Point maxSize,
int width, int height, boolean isLandscape, boolean isMultiWindowMode) {
    this.inv = inv;
    this.isLandscape = isLandscape;
    this.isMultiWindowMode = isMultiWindowMode;

    // Determine sizes.
    widthPx = width;
    heightPx = height;
    if (isLandscape) {
        availableWidthPx = maxSize.x;
        availableHeightPx = minSize.y;
    } else {
        availableWidthPx = minSize.x;
        availableHeightPx = maxSize.y;
    }

    Resources res = context.getResources();
    DisplayMetrics dm = res.getDisplayMetrics();

    // Constants from resources
    isTablet = res.getBoolean(R.bool.is_tablet);
    isLargeTablet = res.getBoolean(R.bool.is_large_tablet);
    isPhone = !isTablet && !isLargeTablet;
    aspectRatio = ((float) Math.max(widthPx, heightPx)) / Math.min(widthPx, heightPx);
    boolean isTallDevice = Float.compare(aspectRatio, TALL_DEVICE_ASPECT_RATIO_THRESHOLD) >= 0;

    // Some more constants
    transposeLayoutWithOrientation =
            res.getBoolean(R.bool.hotseat_transpose_layout_with_orientation);

    context = getContext(context, isVerticalBarLayout()
            ? Configuration.ORIENTATION_LANDSCAPE
            : Configuration.ORIENTATION_PORTRAIT);
    res = context.getResources();

    edgeMarginPx = res.getDimensionPixelSize(R.dimen.dynamic_grid_edge_margin);
    desiredWorkspaceLeftRightMarginPx = isVerticalBarLayout() ? 0 : edgeMarginPx;

    int cellLayoutPaddingLeftRightMultiplier = !isVerticalBarLayout() && isTablet
            ? PORTRAIT_TABLET_LEFT_RIGHT_PADDING_MULTIPLIER : 1;
    int cellLayoutPadding = res.getDimensionPixelSize(R.dimen.dynamic_grid_cell_layout_padding);
    if (isLandscape) {
        cellLayoutPaddingLeftRightPx = 0;
        cellLayoutBottomPaddingPx = cellLayoutPadding;
    } else {
        cellLayoutPaddingLeftRightPx = cellLayoutPaddingLeftRightMultiplier * cellLayoutPadding;
        cellLayoutBottomPaddingPx = 0;
    }

    verticalDragHandleSizePx = res.getDimensionPixelSize(
            R.dimen.vertical_drag_handle_size);
    verticalDragHandleOverlapWorkspace =
            res.getDimensionPixelSize(R.dimen.vertical_drag_handle_overlap_workspace);

    IconLabelController ilc = LauncherAppMonitor.getInstance(context).getIconLabelController();
    maxIconLabelLines = ilc != null ?
            ilc.getIconLabelLine() : IconLabelController.MIN_ICON_LABEL_LINE;

    iconDrawablePaddingOriginalPx =
            res.getDimensionPixelSize(R.dimen.dynamic_grid_icon_drawable_padding);
    dropTargetBarSizePx = res.getDimensionPixelSize(R.dimen.dynamic_grid_drop_target_size);
    workspaceSpringLoadedBottomSpace =
            res.getDimensionPixelSize(R.dimen.dynamic_grid_min_spring_loaded_space);

    workspaceCellPaddingXPx = res.getDimensionPixelSize(R.dimen.dynamic_grid_cell_padding_x);

    hotseatBarTopPaddingPx =
            res.getDimensionPixelSize(R.dimen.dynamic_grid_hotseat_top_padding);
    hotseatBarBottomPaddingPx = (isTallDevice ? 0
            : res.getDimensionPixelSize(R.dimen.dynamic_grid_hotseat_bottom_non_tall_padding))
            + res.getDimensionPixelSize(R.dimen.dynamic_grid_hotseat_bottom_padding);
    hotseatBarSidePaddingEndPx =
            res.getDimensionPixelSize(R.dimen.dynamic_grid_hotseat_side_padding);
    // Add a bit of space between nav bar and hotseat in vertical bar layout.
    hotseatBarSidePaddingStartPx = isVerticalBarLayout() ? verticalDragHandleSizePx : 0;
    hotseatBarSizePx = ResourceUtils.pxFromDp(inv.iconSize, dm) + (isVerticalBarLayout()
            ? (hotseatBarSidePaddingStartPx + hotseatBarSidePaddingEndPx)
            : (res.getDimensionPixelSize(R.dimen.dynamic_grid_hotseat_extra_vertical_size)
                    + hotseatBarTopPaddingPx + hotseatBarBottomPaddingPx));
  ....

}

```

在 DeviceProfile 构造函数中的 hotseatBarSizePx 就是设置的导航栏高度，在这里构建 hotseat 布局的时候，可以通过设置这个高度了布局  
hotseatBarSizePx 就是 hotseat 的高度  
直接设为 0 即可  
修改如下:

```
hotseatBarSizePx = 0/*ResourceUtils.pxFromDp(inv.iconSize, dm) + (isVerticalBarLayout()? (hotseatBarSidePaddingStartPx + hotseatBarSidePaddingEndPx): (res.getDimensionPixelSize(R.dimen.dynamic_grid_hotseat_extra_vertical_size)+ hotseatBarTopPaddingPx + hotseatBarBottomPaddingPx))*/;

```
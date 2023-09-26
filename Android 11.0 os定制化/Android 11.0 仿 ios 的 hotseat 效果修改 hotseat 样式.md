> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124806786)

### 1. 概述

最近在 11.0 产品项目需求的需要，系统原生 [Launcher](https://so.csdn.net/so/search?q=Launcher&spm=1001.2101.3001.7020) 的布局样式很一般，所以需要重新设计 ui 对布局样式做调整，产品在看到  
ios 的 hotseat 效果觉得特别美观，所以要仿 ios 一样不需要横屏铺满的效果 居中显示就行了，所以就要看 hotseat 的具体布局显示了  
效果图如下:  
![](https://img-blog.csdnimg.cn/29390d8fc9974b2caa37e3d03cd9963a.png#pic_center)

### 2. 仿 ios 的 hotseat 效果修改 hotseat 样式的核心类

```
packages/apps/Launcher3/res/layout/launcher.xml
packages/apps/Launcher3/src/com/android/launcher3/Hotseat.java

```

### 3. 仿 ios 的 hotseat 效果修改 hotseat 样式的核心功能实现和分析

### 3.1 首选看下 Launcher 布局的 laucher.xml 的相关源码分析

接下来先看 hotseat 布局 在 laucher.xml 中

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

在上述的 launcher.xml 中

```
  <include
            android:id="@+id/hotseat"
            layout="@layout/hotseat" />

```

hotseat 布局 就是 hotseat.xml

```
<com.android.launcher3.Hotseat
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:launcher="http://schemas.android.com/apk/res-auto"
    android:id="@+id/hotseat"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:theme="@style/HomeScreenElementTheme"
    android:importantForAccessibility="no"
    launcher:containerType="hotseat" />

```

所以 com.android.launcher3.Hotseat 就是 hotseat 布局 也就是 Hotseat.java

### 3.2 接下来看 Hotseat.java 源码

```
public class Hotseat extends CellLayout implements LogContainerProvider, Insettable, Transposable {

    @ViewDebug.ExportedProperty(category = "launcher")
    public boolean mHasVerticalHotseat;
    private final HotseatController mController;

    public Hotseat(Context context) {
        this(context, null);
    }

    public Hotseat(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public Hotseat(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        mController = LauncherAppMonitor.getInstance(context).getHotseatController();
		
    }

    public HotseatController getController() {
        return mController;
    }

    /* Get the orientation specific coordinates given an invariant order in the hotseat. */
    public int getCellXFromOrder(int rank) {
        return mHasVerticalHotseat ? 0 : rank;
    }

    public int getCellYFromOrder(int rank) {
        return mHasVerticalHotseat ? (getCountY() - (rank + 1)) : 0;
    }

    public void resetLayout(boolean hasVerticalHotseat) {
        removeAllViewsInLayout();
        mHasVerticalHotseat = hasVerticalHotseat;
        InvariantDeviceProfile idp = mActivity.getDeviceProfile().inv;
        if (hasVerticalHotseat) {
            setGridSize(1, idp.numHotseatIcons);
        } else {
            setGridSize(idp.numHotseatIcons, 1);
        }
       // 添加背景
	setBackgroundResource(R.drawable.shape_corner);
    }

    @Override
    public void fillInLogContainerData(View v, ItemInfo info, Target target, Target targetParent) {
        target.gridX = info.cellX;
        target.gridY = info.cellY;
        targetParent.containerType = LauncherLogProto.ContainerType.HOTSEAT;
    }

    @Override
    public void setInsets(Rect insets) {
        FrameLayout.LayoutParams lp = (FrameLayout.LayoutParams) getLayoutParams();
        DeviceProfile grid = mActivity.getWallpaperDeviceProfile();
        insets = grid.getInsets();

        if (grid.isVerticalBarLayout()) {
            lp.height = ViewGroup.LayoutParams.WRAP_CONTENT;
            if (grid.isSeascape()) {
                lp.gravity = Gravity.LEFT;
                lp.width = grid.hotseatBarSizePx + insets.left;
            } else {
                lp.gravity = Gravity.RIGHT;
                lp.width = grid.hotseatBarSizePx + insets.right;
            }
        } else {
            lp.gravity = Gravity.BOTTOM;
            lp.width = 960;
			lp.leftMargin=480;
			lp.bottomMargin = 100;
            lp.height = grid.hotseatBarSizePx;
        }
        Rect padding = grid.getHotseatLayoutPadding();
        setPadding(0, padding.top, 0,0);
        
        setLayoutParams(lp);
        InsettableFrameLayout.dispatchInsets(this, insets);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        // Don't let if follow through to workspace
        return true;
    }

    @Override
    public RotationMode getRotationMode() {
        return Launcher.getLauncher(getContext()).getRotationMode();
    }
}

```

在 Hotseat 中可以发现由  
resetLayout 就是负责布局的 当 hotseat 增加减少时都会重新布局  
所以在 setBackgroundResource(R.drawable.shape_corner); 添加背景就可以了

```
  public void resetLayout(boolean hasVerticalHotseat) {
        removeAllViewsInLayout();
        mHasVerticalHotseat = hasVerticalHotseat;
        InvariantDeviceProfile idp = mActivity.getDeviceProfile().inv;
        if (hasVerticalHotseat) {
            setGridSize(1, idp.numHotseatIcons);
        } else {
            setGridSize(idp.numHotseatIcons, 1);
        }
       // 添加背景
	setBackgroundResource(R.drawable.shape_corner);
    }

shape_corner.xml

<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android">
    <!--背景颜色-->
    <solid android:color="#FFFAFA" />
    <!--角的半径-->
    <corners android:radius="10dp"/>
    <!--边框颜色-->
    <stroke android:width="1dp" android:color="#00000000" />
</shape>





 public void setInsets(Rect insets) {
        FrameLayout.LayoutParams lp = (FrameLayout.LayoutParams) getLayoutParams();
        DeviceProfile grid = mActivity.getWallpaperDeviceProfile();
        insets = grid.getInsets();
        //竖屏布局
        if (grid.isVerticalBarLayout()) {
            lp.height = ViewGroup.LayoutParams.WRAP_CONTENT;
            if (grid.isSeascape()) {
                lp.gravity = Gravity.LEFT;
                lp.width = grid.hotseatBarSizePx + insets.left;
            } else {
                lp.gravity = Gravity.RIGHT;
                lp.width = grid.hotseatBarSizePx + insets.right;
            }
        } else {
           // 横屏布局
           // 平板开发项目 固定横屏，所以要在这里设置参数
            lp.gravity = Gravity.BOTTOM;
           // 设置宽高  左边底部的间距
            lp.width = 960;
			lp.leftMargin=480;
			lp.bottomMargin = 100;
            lp.height = grid.hotseatBarSizePx;
        }
        Rect padding = grid.getHotseatLayoutPadding();
        // 设置padding 布局
           setPadding(0, padding.top, 0,0);
        
        setLayoutParams(lp);
        InsettableFrameLayout.dispatchInsets(this, insets);
    }

```

而 setInset() 负责设置绘制布局 的参数 这里设置 hotseat 的宽高等参数布局  
其实只需要修改 lp 的参数就行了 然后 hotseat 会根据长宽等参数 来具体布局每一个 hotseat 的具体坐标
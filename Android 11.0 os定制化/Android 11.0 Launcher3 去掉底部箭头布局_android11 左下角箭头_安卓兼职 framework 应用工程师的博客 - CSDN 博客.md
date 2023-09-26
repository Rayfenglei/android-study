> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125961781)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.Launcher3 去掉底部箭头布局的核心代码](#t1)

[3.Launcher3 去掉底部箭头布局的核心代码功能分析](#t2)

[3.1luncher.xml 相关代码分析](#t3)

[3.2 scrim_view.xml 相关代码分析](#t4)

[3.3 ScrimView 相关箭头布局分析](#t5)

[3.4 Launcher3 去掉底部箭头布局功能实现](#t6)

1. 概述
-----

在 Launcher3 的原生布局中，在首页的左下角有个向下箭头，点击后进入 app 列表，在修改单层布局的时候，这个箭头就显得很不合适了所以在进行 ui 重新布局的时候，要求去掉这个箭头布局

2.Launcher3 去掉底部箭头布局的核心代码
-------------------------

```
主要核心代码:
  packages/apps/Launcher3/res/layout/luncher.xml
  packages/apps/Launcher3/res/layout/scrim_view.xml
  packages/apps/Launcher3/src/com/android/launcher3/views/ScrimView.java
```

3.Launcher3 去掉底部箭头布局的核心代码功能分析
-----------------------------

#### 3.1luncher.xml 相关代码分析

```
<?xml version="1.0" encoding="utf-8"?>
<!-- Copyright (C) 2007 The Android Open Source Project
 
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at
 
http://www.apache.org/licenses/LICENSE-2.0
 
Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->
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
 
<!-- DO NOT CHANGE THE ID -->
<include
android:id="@+id/hotseat"
layout="@layout/hotseat" />
 
<include
android:id="@+id/overview_panel"
layout="@layout/overview_panel" />
 
 
<!-- Keep these behind the workspace so that they are not visible when
we go into AllApps -->
<com.android.launcher3.pageindicators.WorkspacePageIndicator
android:id="@+id/page_indicator"
android:layout_width="match_parent"
android:layout_height="@dimen/workspace_page_indicator_height"
android:layout_gravity="bottom|center_horizontal"
android:theme="@style/HomeScreenElementTheme" />
 
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
 
其中<include
android:id="@+id/scrim_view"
layout="@layout/scrim_view" />就是底部箭头布局
```

#### 3.2 scrim_view.xml 相关代码分析

```
<?xml version="1.0" encoding="utf-8"?>
<!-- Copyright (C) 2018 The Android Open Source Project
 
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at
 
http://www.apache.org/licenses/LICENSE-2.0
 
Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->
<com.android.launcher3.views.ScrimView
xmlns:android="http://schemas.android.com/apk/res/android"
android:layout_width="match_parent"
android:layout_height="match_parent"
android:id="@+id/scrim_view" />
```

#### 3.3 ScrimView 相关箭头布局分析

```
public class ScrimView<T extends Launcher> extends View implements Insettable, OnChangeListener,
AccessibilityStateChangeListener {
 
public static final IntProperty<ScrimView> DRAG_HANDLE_ALPHA =
new IntProperty<ScrimView>("dragHandleAlpha") {
 
@Override
public Integer get(ScrimView scrimView) {
return scrimView.mDragHandleAlpha;
}
 
@Override
public void setValue(ScrimView scrimView, int value) {
scrimView.setDragHandleAlpha(value);
}
};
private static final int WALLPAPERS = R.string.wallpaper_button_text;
private static final int WIDGETS = R.string.widget_button_text;
private static final int SETTINGS = R.string.settings_button_text;
private static final int ALPHA_CHANNEL_COUNT = 1;
 
private static final long DRAG_HANDLE_BOUNCE_DURATION_MS = 300;
// How much to delay before repeating the bounce.
private static final long DRAG_HANDLE_BOUNCE_DELAY_MS = 200;
// Repeat this many times (i.e. total number of bounces is 1 + this).
private static final int DRAG_HANDLE_BOUNCE_REPEAT_COUNT = 2;
 
private final Rect mTempRect = new Rect();
private final int[] mTempPos = new int[2];
 
protected final T mLauncher;
private final WallpaperColorInfo mWallpaperColorInfo;
private final AccessibilityManager mAM;
protected final int mEndScrim;
protected final boolean mIsScrimDark;
 
private final StateListener<LauncherState> mAccessibilityLauncherStateListener =
new StateListener<LauncherState>() {
@Override
public void onStateTransitionComplete(LauncherState finalState) {
setImportantForAccessibility(finalState == ALL_APPS
? IMPORTANT_FOR_ACCESSIBILITY_NO_HIDE_DESCENDANTS
: IMPORTANT_FOR_ACCESSIBILITY_AUTO);
}
};
 
protected float mMaxScrimAlpha;
 
protected float mProgress = 1;
protected int mScrimColor;
 
protected int mCurrentFlatColor;
protected int mEndFlatColor;
protected int mEndFlatColorAlpha;
 
protected final Point mDragHandleSize;
private final int mDragHandleTouchSize;
private final int mDragHandlePaddingInVerticalBarLayout;
protected float mDragHandleOffset;
private final Rect mDragHandleBounds;
private final RectF mHitRect = new RectF();
private ObjectAnimator mDragHandleAnim;
 
private final MultiValueAlpha mMultiValueAlpha;
 
private final AccessibilityHelper mAccessibilityHelper;
@Nullable
protected Drawable mDragHandle;
 
private int mDragHandleAlpha = 255;
 
public ScrimView(Context context, AttributeSet attrs) {
super(context, attrs);
mLauncher = Launcher.cast(Launcher.getLauncher(context));
mWallpaperColorInfo = WallpaperColorInfo.INSTANCE.get(context);
mEndScrim = Themes.getAttrColor(context, R.attr.allAppsScrimColor);
mIsScrimDark = ColorUtils.calculateLuminance(mEndScrim) < 0.5f;
 
mMaxScrimAlpha = 0.7f;
 
Resources res = context.getResources();
mDragHandleSize = new Point(res.getDimensionPixelSize(R.dimen.vertical_drag_handle_width),
res.getDimensionPixelSize(R.dimen.vertical_drag_handle_height));
mDragHandleBounds = new Rect(0, 0, mDragHandleSize.x, mDragHandleSize.y);
mDragHandleTouchSize = res.getDimensionPixelSize(R.dimen.vertical_drag_handle_touch_size);
mDragHandlePaddingInVerticalBarLayout = context.getResources()
getDimensionPixelSize(R.dimen.vertical_drag_handle_padding_in_vertical_bar_layout);
 
mAccessibilityHelper = createAccessibilityHelper();
ViewCompat.setAccessibilityDelegate(this, mAccessibilityHelper);
 
mAM = (AccessibilityManager) context.getSystemService(ACCESSIBILITY_SERVICE);
setFocusable(false);
mMultiValueAlpha = new MultiValueAlpha(this, ALPHA_CHANNEL_COUNT);
}
 
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
super.onMeasure(widthMeasureSpec, heightMeasureSpec);
updateDragHandleBounds();
}
 
@Override
protected void onAttachedToWindow() {
super.onAttachedToWindow();
mWallpaperColorInfo.addOnChangeListener(this);
onExtractedColorsChanged(mWallpaperColorInfo);
 
mAM.addAccessibilityStateChangeListener(this);
onAccessibilityStateChanged(mAM.isEnabled());
}
 
protected void updateDragHandleBounds() {
DeviceProfile grid = mLauncher.getDeviceProfile();
final int left;
final int width = getMeasuredWidth();
final int top = getMeasuredHeight() - mDragHandleSize.y - grid.getInsets().bottom;
final int topMargin;
 
if (grid.isVerticalBarLayout()) {
topMargin = grid.workspacePadding.bottom + mDragHandlePaddingInVerticalBarLayout;
if (grid.isSeascape()) {
left = width - grid.getInsets().right - mDragHandleSize.x
- mDragHandlePaddingInVerticalBarLayout;
} else {
left = grid.getInsets().left + mDragHandlePaddingInVerticalBarLayout;
}
} else {
left = Math.round((width - mDragHandleSize.x) / 2f);
topMargin = grid.hotseatBarSizePx;
}
mDragHandleBounds.offsetTo(left, top - topMargin);
mHitRect.set(mDragHandleBounds);
// Inset outwards to increase touch size.
mHitRect.inset((mDragHandleSize.x - mDragHandleTouchSize) / 2f,
(mDragHandleSize.y - mDragHandleTouchSize) / 2f);
 
if (mDragHandle != null) {
mDragHandle.setBounds(mDragHandleBounds);
}
}
 
@Override
public void onAccessibilityStateChanged(boolean enabled) {
StateManager<LauncherState> stateManager = mLauncher.getStateManager();
stateManager.removeStateListener(mAccessibilityLauncherStateListener);
 
if (enabled) {
stateManager.addStateListener(mAccessibilityLauncherStateListener);
mAccessibilityLauncherStateListener.onStateTransitionComplete(stateManager.getState());
} else {
setImportantForAccessibility(IMPORTANT_FOR_ACCESSIBILITY_NO_HIDE_DESCENDANTS);
}
updateDragHandleVisibility();
}
 
public void updateDragHandleVisibility() {
updateDragHandleVisibility(null);
}
 
private void updateDragHandleVisibility(@Nullable Drawable recycle) {
boolean visible = shouldDragHandleBeVisible();
boolean wasVisible = mDragHandle != null;
if (visible != wasVisible) {
if (visible) {
mDragHandle = recycle != null ? recycle :
mLauncher.getDrawable(R.drawable.drag_handle_indicator_shadow);
mDragHandle.setBounds(mDragHandleBounds);
 
updateDragHandleAlpha();
} else {
mDragHandle = null;
}
invalidate();
}
}
 
在onMeasure(int widthMeasureSpec, int heightMeasureSpec)中绘制布局时，调用updateDragHandleBounds();
最终调用updateDragHandleVisibility(@Nullable Drawable recycle) 来负责绘制向下箭头
而R.drawable.drag_handle_indicator_shadow就是向下箭头的图标
```

#### 3.4 Launcher3 去掉底部箭头布局功能实现

```
private void updateDragHandleVisibility(@Nullable Drawable recycle) {
- boolean visible = shouldDragHandleBeVisible();
+ boolean visible = false;
boolean wasVisible = mDragHandle != null;
if (visible != wasVisible) {
if (visible) {
mDragHandle = recycle != null ? recycle :
mLauncher.getDrawable(R.drawable.drag_handle_indicator_shadow);
mDragHandle.setBounds(mDragHandleBounds);
 
updateDragHandleAlpha();
} else {
mDragHandle = null;
}
invalidate();
}
}
 
```
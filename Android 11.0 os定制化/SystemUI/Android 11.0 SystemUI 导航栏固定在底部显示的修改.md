> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126940977)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.SystemUI 导航栏固定在底部显示的修改的相关代码](#t1)

[3.SystemUI 导航栏固定在底部显示的修改的代码分析和功能实现](#t2)

 [3.1 分析 DisplayContent.java 关于导航栏布局的相关代码块](#t3)

[3.2 对 DisplayPolicy.java 相关导航栏布局的分析](#t4)

1. 概述
-----

 在产品开发中，对于定制化 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 的功能有好多，在导航栏布局方面，根据屏幕分辨率  
有定位在左边底部和右边的，但对于横屏的产品来说，定位在底部是比较好的布局，所以要导航栏固定在底部要根据显示流程然后决定定位方向

  
2.SystemUI 导航栏固定在底部显示的修改的相关代码
--------------------------------

```
  frameworks/base/services/core/java/com/android/server/wm/DisplayContent.java
  frameworks/base/services/core/java/com/android/server/wm/DisplayPolicy.java
```

3.SystemUI 导航栏固定在底部显示的修改的代码分析和功能实现
----------------------------------

###   3.1 分析 DisplayContent.java 关于导航栏布局的相关代码块

```
private void performLayoutNoTrace(boolean initial, boolean updateInputWindows) {
          if (!isLayoutNeeded()) {
              return;
          }
          clearLayoutNeeded();
  
          final int dw = mDisplayInfo.logicalWidth;
          final int dh = mDisplayInfo.logicalHeight;
          if (DEBUG_LAYOUT) {
              Slog.v(TAG, "-------------------------------------");
              Slog.v(TAG, "performLayout: needed=" + isLayoutNeeded() + " dw=" + dw
                      + " dh=" + dh);
          }
  
          mDisplayFrames.onDisplayInfoUpdated(mDisplayInfo,
                  calculateDisplayCutoutForRotation(mDisplayInfo.rotation));
          // TODO: Not sure if we really need to set the rotation here since we are updating from
          // the display info above...
          mDisplayFrames.mRotation = getRotation();
          mDisplayPolicy.beginLayoutLw(mDisplayFrames, getConfiguration().uiMode);
  
          int seq = mLayoutSeq + 1;
          if (seq < 0) seq = 0;
          mLayoutSeq = seq;
  
          // Used to indicate that we have processed the dream window and all additional windows are
          // behind it.
          mTmpWindow = null;
          mTmpInitial = initial;
  
          // Used to indicate that we have processed the IME window.
          mTmpWindowsBehindIme = false;
  
          // First perform layout of any root windows (not attached to another window).
          forAllWindows(mPerformLayout, true /* traverseTopToBottom */);
  
          // Used to indicate that we have processed the dream window and all additional attached
          // windows are behind it.
          mTmpWindow2 = mTmpWindow;
          mTmpWindow = null;
  
          // Now perform layout of attached windows, which usually depend on the position of the
          // window they are attached to. XXX does not deal with windows that are attached to windows
          // that are themselves attached.
          forAllWindows(mPerformLayoutAttached, true /* traverseTopToBottom */);
  
          // Window frames may have changed. Tell the input dispatcher about it.
          mInputMonitor.layoutInputConsumers(dw, dh);
          mInputMonitor.setUpdateInputWindowsNeededLw();
          if (updateInputWindows) {
              mInputMonitor.updateInputWindowsLw(false /*force*/);
          }
  
          mWmService.mH.sendEmptyMessage(UPDATE_MULTI_WINDOW_STACKS);
      }
```

代码分析如下:  
          if (!isLayoutNeeded()) {  
              return;  
          }// 如果 mLayoutNeeded 为 false, 直接返回  
clearLayoutNeeded();// 清理原来布局  
mDisplayPolicy.beginLayoutLw(mDisplayFrames, getConfiguration().uiMode);// 将进行布局的前期工作，将重置一些状态数据  
forAllWindows(mPerformLayout, true /* traverseTopToBottom */);// 进行非子窗口的 layout  
forAllWindows(mPerformLayoutAttached, true /* traverseTopToBottom */);// 进行子窗口的 layout

所以相关布局流程

    1. 执行 DisplayPolicy 的 beginLayoutLw() 方法，这个方法中会对状态栏和导航栏进行布局，并根据状态栏和导航栏的状态，更新 DisplayFrame 中的各个 Rect 属性，完成对 DisplayFrame 的确定；  
    2. 对所有 WindowState 进行 layout 操作，mPerformLayout 是一个函数接口 Consumer 的实例，通过 forAllWindows() 完成所有 WindowState 的遍历，完成对 WindowFrame 的确定；  
    3. 对非 TYPE_APPLICATION_ATTACHED_DIALOG 类型的依附窗口进行额外的处理，WindowState#mLayoutAttached 属性表示依附窗口，即属于 SubWindow 类型的窗口。

### 3.2 对 DisplayPolicy.java 相关导航栏布局的分析

```
	  public void beginLayoutLw(DisplayFrames displayFrames, int uiMode) {
          displayFrames.onBeginLayout();
          updateInsetsStateForDisplayCutout(displayFrames,
                  mDisplayContent.getInsetsStateController().getRawInsetsState());
          mSystemGestures.screenWidth = displayFrames.mUnrestricted.width();
          mSystemGestures.screenHeight = displayFrames.mUnrestricted.height();
  
          // For purposes of putting out fake window up to steal focus, we will
          // drive nav being hidden only by whether it is requested.
          final int sysui = mLastSystemUiFlags;
          final int behavior = mLastBehavior;
          final InsetsSourceProvider provider =
                  mDisplayContent.getInsetsStateController().peekSourceProvider(ITYPE_NAVIGATION_BAR);
          boolean navVisible = ViewRootImpl.sNewInsetsMode != ViewRootImpl.NEW_INSETS_MODE_FULL
                  ? (sysui & View.SYSTEM_UI_FLAG_HIDE_NAVIGATION) == 0
                  : provider != null
                          ? provider.isClientVisible()
                          : InsetsState.getDefaultVisibility(ITYPE_NAVIGATION_BAR);
          boolean navTranslucent = (sysui
                  & (View.NAVIGATION_BAR_TRANSLUCENT | View.NAVIGATION_BAR_TRANSPARENT)) != 0;
          boolean immersive = (sysui & View.SYSTEM_UI_FLAG_IMMERSIVE) != 0
                  || (behavior & BEHAVIOR_SHOW_BARS_BY_SWIPE) != 0;
          boolean immersiveSticky = (sysui & View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY) != 0
                  || (behavior & BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE) != 0;
          boolean navAllowedHidden = immersive || immersiveSticky;
          navTranslucent &= !immersiveSticky;  // transient trumps translucent
          boolean isKeyguardShowing = isKeyguardShowing() && !isKeyguardOccluded();
          boolean notificationShadeForcesShowingNavigation =
                  !isKeyguardShowing && mNotificationShade != null
                  && (mNotificationShade.getAttrs().privateFlags
                  & PRIVATE_FLAG_STATUS_FORCE_SHOW_NAVIGATION) != 0;
  
          updateHideNavInputEventReceiver();
  
          // For purposes of positioning and showing the nav bar, if we have decided that it can't
          // be hidden (because of the screen aspect ratio), then take that into account.
          navVisible |= !canHideNavigationBar();
  
          boolean updateSysUiVisibility = layoutNavigationBar(displayFrames, uiMode, navVisible,
                  navTranslucent, navAllowedHidden, notificationShadeForcesShowingNavigation,
                  null /* simulatedContentFrame */);
          if (DEBUG_LAYOUT) Slog.i(TAG, "mDock rect:" + displayFrames.mDock);
          updateSysUiVisibility |= layoutStatusBar(displayFrames, sysui,
                  null /* simulatedContentFrame */);
          if (updateSysUiVisibility) {
              updateSystemUiVisibilityLw();
          }
          layoutScreenDecorWindows(displayFrames, null /* simulatedFrames */);
          postAdjustDisplayFrames(displayFrames);
          mLastNavVisible = navVisible;
          mLastNavTranslucent = navTranslucent;
          mLastNavAllowedHidden = navAllowedHidden;
          mLastNotificationShadeForcesShowingNavigation = notificationShadeForcesShowingNavigation;
      }
```

代码分析:  
          mSystemGestures.screenWidth = displayFrames.mUnrestricted.width();  
          mSystemGestures.screenHeight = displayFrames.mUnrestricted.height();  
// 获取屏幕宽和高  
displayFrames.onBeginLayout();// 开始绘制功能  
boolean isKeyguardShowing = isKeyguardShowing() && !isKeyguardOccluded();// 判断当前是否是锁屏状态  
          boolean updateSysUiVisibility = layoutNavigationBar(displayFrames, uiMode, navVisible,  
                  navTranslucent, navAllowedHidden, notificationShadeForcesShowingNavigation,  
                  null /* simulatedContentFrame */);// 绘制导航栏布局 就是需要修改的导航栏布局部分  
接下来看下具体的导航栏布局相关代码

```
 private boolean layoutNavigationBar(DisplayFrames displayFrames, int uiMode, boolean navVisible,
              boolean navTranslucent, boolean navAllowedHidden,
              boolean statusBarForcesShowingNavigation, Rect simulatedContentFrame) {
          if (mNavigationBar == null) {
              return false;
          }
  
          final Rect navigationFrame = sTmpNavFrame;
          boolean transientNavBarShowing = mNavigationBarController.isTransientShowing();
          // Force the navigation bar to its appropriate place and size. We need to do this directly,
          // instead of relying on it to bubble up from the nav bar, because this needs to change
          // atomically with screen rotations.
          final int rotation = displayFrames.mRotation;
          final int displayHeight = displayFrames.mDisplayHeight;
          final int displayWidth = displayFrames.mDisplayWidth;
          final Rect dockFrame = displayFrames.mDock;
          final int navBarPosition = navigationBarPosition(displayWidth, displayHeight, rotation);
  
          final Rect cutoutSafeUnrestricted = sTmpRect;
          cutoutSafeUnrestricted.set(displayFrames.mUnrestricted);
          cutoutSafeUnrestricted.intersectUnchecked(displayFrames.mDisplayCutoutSafe);
  
          if (navBarPosition == NAV_BAR_BOTTOM) {
              // It's a system nav bar or a portrait screen; nav bar goes on bottom.
              final int topNavBar = cutoutSafeUnrestricted.bottom
                      - getNavigationBarFrameHeight(rotation, uiMode);
              final int top = mNavButtonForcedVisible
                      ? topNavBar
                      : cutoutSafeUnrestricted.bottom - getNavigationBarHeight(rotation, uiMode);
              navigationFrame.set(0, topNavBar, displayWidth, displayFrames.mUnrestricted.bottom);
              displayFrames.mStable.bottom = displayFrames.mStableFullscreen.bottom = top;
              if (transientNavBarShowing) {
                  mNavigationBarController.setBarShowingLw(true);
              } else if (navVisible) {
                  mNavigationBarController.setBarShowingLw(true);
                  dockFrame.bottom = displayFrames.mRestricted.bottom = top;
              } else {
                  // We currently want to hide the navigation UI - unless we expanded the status bar.
                  mNavigationBarController.setBarShowingLw(statusBarForcesShowingNavigation);
              }
              if (navVisible && !navTranslucent && !navAllowedHidden
                      && !mNavigationBar.isAnimatingLw()
                      && !mNavigationBarController.wasRecentlyTranslucent()) {
                  // If the opaque nav bar is currently requested to be visible and not in the process
                  // of animating on or off, then we can tell the app that it is covered by it.
                  displayFrames.mSystem.bottom = top;
              }
          } else if (navBarPosition == NAV_BAR_RIGHT) {
              // Landscape screen; nav bar goes to the right.
              final int left = cutoutSafeUnrestricted.right
                      - getNavigationBarWidth(rotation, uiMode);
              navigationFrame.set(left, 0, displayFrames.mUnrestricted.right, displayHeight);
              displayFrames.mStable.right = displayFrames.mStableFullscreen.right = left;
              if (transientNavBarShowing) {
                  mNavigationBarController.setBarShowingLw(true);
              } else if (navVisible) {
                  mNavigationBarController.setBarShowingLw(true);
                  dockFrame.right = displayFrames.mRestricted.right = left;
              } else {
                  // We currently want to hide the navigation UI - unless we expanded the status bar.
                  mNavigationBarController.setBarShowingLw(statusBarForcesShowingNavigation);
              }
              if (navVisible && !navTranslucent && !navAllowedHidden
                      && !mNavigationBar.isAnimatingLw()
                      && !mNavigationBarController.wasRecentlyTranslucent()) {
                  // If the nav bar is currently requested to be visible, and not in the process of
                  // animating on or off, then we can tell the app that it is covered by it.
                  displayFrames.mSystem.right = left;
              }
          } else if (navBarPosition == NAV_BAR_LEFT) {
              // Seascape screen; nav bar goes to the left.
              final int right = cutoutSafeUnrestricted.left
                      + getNavigationBarWidth(rotation, uiMode);
              navigationFrame.set(displayFrames.mUnrestricted.left, 0, right, displayHeight);
              displayFrames.mStable.left = displayFrames.mStableFullscreen.left = right;
              if (transientNavBarShowing) {
                  mNavigationBarController.setBarShowingLw(true);
              } else if (navVisible) {
                  mNavigationBarController.setBarShowingLw(true);
                  dockFrame.left = displayFrames.mRestricted.left = right;
              } else {
                  // We currently want to hide the navigation UI - unless we expanded the status bar.
                  mNavigationBarController.setBarShowingLw(statusBarForcesShowingNavigation);
              }
              if (navVisible && !navTranslucent && !navAllowedHidden
                      && !mNavigationBar.isAnimatingLw()
                      && !mNavigationBarController.wasRecentlyTranslucent()) {
                  // If the nav bar is currently requested to be visible, and not in the process of
                  // animating on or off, then we can tell the app that it is covered by it.
                  displayFrames.mSystem.left = right;
              }
          }
  
          // Make sure the content and current rectangles are updated to account for the restrictions
          // from the navigation bar.
          displayFrames.mCurrent.set(dockFrame);
          displayFrames.mVoiceContent.set(dockFrame);
          displayFrames.mContent.set(dockFrame);
          // And compute the final frame.
          sTmpRect.setEmpty();
          final WindowFrames windowFrames = mNavigationBar.getLayoutingWindowFrames();
          windowFrames.setFrames(navigationFrame /* parentFrame */,
                  navigationFrame /* displayFrame */,
                  displayFrames.mDisplayCutoutSafe /* contentFrame */,
                  navigationFrame /* visibleFrame */, sTmpRect /* decorFrame */,
                  navigationFrame /* stableFrame */);
          mNavigationBar.computeFrame(displayFrames);
          if (simulatedContentFrame != null) {
              simulatedContentFrame.set(windowFrames.mContentFrame);
          } else {
              mNavigationBarPosition = navBarPosition;
              mNavigationBarController.setContentFrame(windowFrames.mContentFrame);
          }
  
          if (DEBUG_LAYOUT) Slog.i(TAG, "mNavigationBar frame: " + navigationFrame);
          return mNavigationBarController.checkHiddenLw();
      }
```

分析代码:  
          boolean transientNavBarShowing = mNavigationBarController.isTransientShowing();// 导航栏是否显示  
          final int rotation = displayFrames.mRotation;  
          final int displayHeight = displayFrames.mDisplayHeight;  
          final int displayWidth = displayFrames.mDisplayWidth;  
          final Rect dockFrame = displayFrames.mDock;  
这部分获取导航栏显示宽高方向  
navigationBarPosition(displayWidth, displayHeight, rotation);// 根据宽高方向判断导航栏显示位置  
接下来看具体定位代码

```
     @NavigationBarPosition
      int navigationBarPosition(int displayWidth, int displayHeight, int displayRotation) {
          if (navigationBarCanMove() && displayWidth > displayHeight) {
              if (displayRotation == Surface.ROTATION_270) {
                  return NAV_BAR_LEFT;
              } else if (displayRotation == Surface.ROTATION_90) {
                  return NAV_BAR_RIGHT;
              }
          }
          return NAV_BAR_BOTTOM;
      }
```

 // 当宽大于高就是横屏并且 navigationBarCanMove() 为 true 是根据旋转方向来判断导航栏  
是显示在左边或者右边  
所以要固定底部 就需要 navigationBarCanMove() 为 fasle 就可以了

```
  /**
       * @return The side of the screen where navigation bar is positioned.
       * @see WindowManagerPolicyConstants#NAV_BAR_LEFT
       * @see WindowManagerPolicyConstants#NAV_BAR_RIGHT
       * @see WindowManagerPolicyConstants#NAV_BAR_BOTTOM
       */
      @NavigationBarPosition
      public int getNavBarPosition() {
          return mNavigationBarPosition;
      }
   public boolean navigationBarCanMove() {
          return mNavigationBarCanMove;
      }
     void updateConfigurationAndScreenSizeDependentBehaviors() {
          final Resources res = getCurrentUserResources();
          mNavigationBarCanMove =
                 mDisplayContent.mBaseDisplayWidth != mDisplayContent.mBaseDisplayHeight
                          && res.getBoolean(R.bool.config_navBarCanMove);
          mDisplayContent.getDisplayRotation().updateUserDependentConfiguration(res);
      }
```

mNavigationBarCanMove 的值最终可以由 R.bool.config_navBarCanMove 决定  
所以修改为 false 就可以了

具体修改为:

```
路径：framework/base/core/res/res/values/config.xml
-<bool >true</bool>
+<bool >false</bool>
```
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125091493)

### 1. 概述

在 11.0 定制化开发中，要求导航栏左边显示的定制化，这时需要了解导航栏的显示控制方向，然后修改显示方向  
在 10.0 以后关于导航栏显示位置都是在 DisplayPolicy.java 中处理的所以查询相关的设置方法，然后修改导航栏显示方向

### 2.NavigationBarView 导航栏 左边显示的修改的核心代码

```
 /frameworks/base/services/core/java/com/android/server/wm/DisplayPolicy.java
 /framework/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarView.java

```

### 3.NavigationBarView 导航栏 左边显示的修改核心代码分析和功能实现

### 3.1DisplayPolicy.java 关于导航栏显示方向的相关代码分析

路径:  
/frameworks/base/services/core/java/com/android/server/wm/DisplayPolicy.java  
分两部分实现:

```
   /**
       * Called when a window is being added to the system.  Must not throw an exception.
       *
       * @param win The window being added.
       * @param attrs Information about the window to be added.
       */
      void addWindowLw(WindowState win, WindowManager.LayoutParams attrs) {
          if ((attrs.privateFlags & PRIVATE_FLAG_IS_SCREEN_DECOR) != 0) {
              mScreenDecorWindows.add(win);
          }
  
          switch (attrs.type) {
              case TYPE_NOTIFICATION_SHADE:
                  mNotificationShade = win;
                  if (mDisplayContent.isDefaultDisplay) {
                      mService.mPolicy.setKeyguardCandidateLw(win);
                  }
                  break;
              case TYPE_STATUS_BAR:
                  mStatusBar = win;
                  mStatusBarController.setWindow(win);
                  final TriConsumer<DisplayFrames, WindowState, Rect> frameProvider =
                          (displayFrames, windowState, rect) -> {
                              rect.top = 0;
                              rect.bottom = getStatusBarHeight(displayFrames);
                          };
                  mDisplayContent.setInsetProvider(ITYPE_STATUS_BAR, win, frameProvider);
                  mDisplayContent.setInsetProvider(ITYPE_TOP_GESTURES, win, frameProvider);
                  mDisplayContent.setInsetProvider(ITYPE_TOP_TAPPABLE_ELEMENT, win, frameProvider);
                  break;
              case TYPE_NAVIGATION_BAR:
                  mNavigationBar = win;
                  mNavigationBarController.setWindow(win);
                  mNavigationBarController.setOnBarVisibilityChangedListener(
                          mNavBarVisibilityListener, true);
                  mDisplayContent.setInsetProvider(ITYPE_NAVIGATION_BAR, win,
                          (displayFrames, windowState, inOutFrame) -> {
  
                              // In Gesture Nav, navigation bar frame is larger than frame to
                              // calculate inset.
                              if (navigationBarPosition(displayFrames.mDisplayWidth,
                                      displayFrames.mDisplayHeight,
                                      displayFrames.mRotation) == NAV_BAR_BOTTOM
                                      && !mNavButtonForcedVisible) {
  
                                  sTmpRect.set(displayFrames.mUnrestricted);
                                  sTmpRect.intersectUnchecked(displayFrames.mDisplayCutoutSafe);
                                  inOutFrame.top = sTmpRect.bottom
                                          - getNavigationBarHeight(displayFrames.mRotation,
                                          mDisplayContent.getConfiguration().uiMode);
                              }
                          },
  
                          // For IME we use regular frame.
                          (displayFrames, windowState, inOutFrame) ->
                                  inOutFrame.set(windowState.getFrameLw()));
  
                  mDisplayContent.setInsetProvider(ITYPE_BOTTOM_GESTURES, win,
                          (displayFrames, windowState, inOutFrame) -> {
                              inOutFrame.top -= mBottomGestureAdditionalInset;
                          });
                  mDisplayContent.setInsetProvider(ITYPE_LEFT_GESTURES, win,
                          (displayFrames, windowState, inOutFrame) -> {
                              inOutFrame.left = 0;
                              inOutFrame.top = 0;
                              inOutFrame.bottom = displayFrames.mDisplayHeight;
                              inOutFrame.right = displayFrames.mUnrestricted.left + mLeftGestureInset;
                          });
                  mDisplayContent.setInsetProvider(ITYPE_RIGHT_GESTURES, win,
                          (displayFrames, windowState, inOutFrame) -> {
                              inOutFrame.left = displayFrames.mUnrestricted.right
                                      - mRightGestureInset;
                              inOutFrame.top = 0;
                              inOutFrame.bottom = displayFrames.mDisplayHeight;
                              inOutFrame.right = displayFrames.mDisplayWidth;
                          });
                  mDisplayContent.setInsetProvider(ITYPE_BOTTOM_TAPPABLE_ELEMENT, win,
                          (displayFrames, windowState, inOutFrame) -> {
                              if ((windowState.getAttrs().flags & FLAG_NOT_TOUCHABLE) != 0
                                      || mNavigationBarLetsThroughTaps) {
                                  inOutFrame.setEmpty();
                              }
                          });
                  if (DEBUG_LAYOUT) Slog.i(TAG, "NAVIGATION BAR: " + mNavigationBar);
                  break;
              default:
                  if (attrs.providesInsetsTypes != null) {
                      for (@InternalInsetsType int insetsType : attrs.providesInsetsTypes) {
                          switch (insetsType) {
                              case ITYPE_STATUS_BAR:
                                  mStatusBarAlt = win;
                                  mStatusBarController.setWindow(mStatusBarAlt);
                                  mStatusBarAltPosition = getAltBarPosition(attrs);
                                  break;
                              case ITYPE_NAVIGATION_BAR:
                                  mNavigationBarAlt = win;
                                  mNavigationBarController.setWindow(mNavigationBarAlt);
                                  mNavigationBarAltPosition = getAltBarPosition(attrs);
                                  break;
                              case ITYPE_CLIMATE_BAR:
                                  mClimateBarAlt = win;
                                  mClimateBarAltPosition = getAltBarPosition(attrs);
                                  break;
                              case ITYPE_EXTRA_NAVIGATION_BAR:
                                  mExtraNavBarAlt = win;
                                  mExtraNavBarAltPosition = getAltBarPosition(attrs);
                                  break;
                          }
                          mDisplayContent.setInsetProvider(insetsType, win, null);
                      }
                  }
                  break;
          }
      }
      
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

在 addWindowLw(）屏幕添加布局时在 TYPE_NAVIGATION_BAR 类型时调用 navigationBarPosition(int displayWidth, int displayHeight, int displayRotation) 来判断显示屏幕的方向，所以可以在这里通过增加旋转方向来设置  
DisplayPolicy 显示方向的控制

```
--- a/services/core/java/com/android/server/wm/DisplayPolicy.java
+++ b/services/core/java/com/android/server/wm/DisplayPolicy.java
@@ -3015,7 +3015,7 @@ public class DisplayPolicy {
     @NavigationBarPosition
     int navigationBarPosition(int displayWidth, int displayHeight, int displayRotation) {
         if (navigationBarCanMove() && displayWidth > displayHeight) {
         // 根据屏幕旋转方向判断导航栏显示的位置
-            if (displayRotation == Surface.ROTATION_270) {
+            if (displayRotation == Surface.ROTATION_270 || displayRotation == Surface.ROTATION_0) {
                 return NAV_BAR_LEFT;
             } else if (displayRotation == Surface.ROTATION_90) {
                 return NAV_BAR_RIGHT;

```

### 3.2NavigationBarView 使用竖屏布局的 UI

在屏幕左边实现导航栏时，也必须默认导航栏布局竖屏显示，默认是横屏显示，所以需要修改默认竖屏，然后调用竖屏的 UI  
布局  
路径:/framework/base/packages/[SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020)/src/com/android/systemui/statusbar/phone/NavigationBarView.java  
diff --git a/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarView.java b/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarView.java  
old mode 100644  
new mode 100755  
index 421a58f…16e9c9f

```
--- a/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarView.java
+++ b/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarView.java
@@ -266,7 +266,7 @@ public class NavigationBarView extends FrameLayout implements
     public NavigationBarView(Context context, AttributeSet attrs) {
         super(context, attrs);
 
-        mIsVertical = false;
+        mIsVertical = true;
         mLongClickableAccessibilityButton = false;
         mNavBarMode = Dependency.get(NavigationModeController.class).addListener(this);

```

mIsVertical 就代表是否是需要竖屏显示 默认为 fasle 改为 true 就好了
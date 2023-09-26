> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124893953)

### 1. 概述

在 11.0 的产品开发中，对于 SystemUI 导航栏状态栏的功能定制需求，有要求需要通过上滑来控制导航栏显示  
然后 3 秒钟后自动消失导航栏，实现一个通过手势上滑显示导航栏的需求

### 2.SystemUI 手势上滑显示导航栏和隐藏导航栏相关核心代码

```
frameworks/base/services/core/java/com/android/server/wm/DisplayPolicy.java
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/NavigationBarController.java

```

### 3.SystemUI 手势上滑显示导航栏和隐藏导航栏功能分析实现

### 3.1 全局手势事件[监听](https://so.csdn.net/so/search?q=%E7%9B%91%E5%90%AC&spm=1001.2101.3001.7020)

在系统中首选要在全局系统手势事件中增加自定义上滑手势广播, 然后在 SystemUI 中监听广播，收到广播  
后显示导航栏，而系统全局手势就是  
SystemGesturesPointerEventListener  
接下来看下手势事件

```
mSystemGestures = new SystemGesturesPointerEventListener(mContext, mHandler,
                  new SystemGesturesPointerEventListener.Callbacks() {
                      @Override
                      public void onSwipeFromTop() {
                          synchronized (mLock) {
                              if (mStatusBar != null) {
                                  requestTransientBars(mStatusBar);
                              }
                              checkAltBarSwipeForTransientBars(ALT_BAR_TOP);
                          }
                      }
  
                      @Override
                      public void onSwipeFromBottom() {
                          synchronized (mLock) {
                              if (mNavigationBar != null
                                      && mNavigationBarPosition == NAV_BAR_BOTTOM) {
                                  requestTransientBars(mNavigationBar);
                              }
                              checkAltBarSwipeForTransientBars(ALT_BAR_BOTTOM);
                          }
                      }
  
                      @Override
                      public void onSwipeFromRight() {
                          final Region excludedRegion = Region.obtain();
                          synchronized (mLock) {
                              mDisplayContent.calculateSystemGestureExclusion(
                                      excludedRegion, null /* outUnrestricted */);
                              final boolean sideAllowed = mNavigationBarAlwaysShowOnSideGesture
                                      || mNavigationBarPosition == NAV_BAR_RIGHT;
                              if (mNavigationBar != null && sideAllowed
                                      && !mSystemGestures.currentGestureStartedInRegion(
                                              excludedRegion)) {
                                  requestTransientBars(mNavigationBar);
                              }
                              checkAltBarSwipeForTransientBars(ALT_BAR_RIGHT);
                          }
                          excludedRegion.recycle();
                      }
  
                      @Override
                      public void onSwipeFromLeft() {
                          final Region excludedRegion = Region.obtain();
                          synchronized (mLock) {
                              mDisplayContent.calculateSystemGestureExclusion(
                                      excludedRegion, null /* outUnrestricted */);
                              final boolean sideAllowed = mNavigationBarAlwaysShowOnSideGesture
                                      || mNavigationBarPosition == NAV_BAR_LEFT;
                              if (mNavigationBar != null && sideAllowed
                                      && !mSystemGestures.currentGestureStartedInRegion(
                                              excludedRegion)) {
                                  requestTransientBars(mNavigationBar);
                              }
                              checkAltBarSwipeForTransientBars(ALT_BAR_LEFT);
                          }
                          excludedRegion.recycle();
                      }
  
                      @Override
                      public void onFling(int duration) {
                          if (mService.mPowerManagerInternal != null) {
                              mService.mPowerManagerInternal.powerHint(
                                      PowerHint.INTERACTION, duration);
                          }
                      }
  
                      @Override
                      public void onDebug() {
                          // no-op
                      }
  
                      private WindowOrientationListener getOrientationListener() {
                          final DisplayRotation rotation = mDisplayContent.getDisplayRotation();
                          return rotation != null ? rotation.getOrientationListener() : null;
                      }
  
                      @Override
                      public void onDown() {
                          final WindowOrientationListener listener = getOrientationListener();
                          if (listener != null) {
                              listener.onTouchStart();
                          }
                      }
  
                      @Override
                      public void onUpOrCancel() {
                          final WindowOrientationListener listener = getOrientationListener();
                          if (listener != null) {
                              listener.onTouchEnd();
                          }
                      }
  
                      @Override
                      public void onMouseHoverAtTop() {
                          mHandler.removeMessages(MSG_REQUEST_TRANSIENT_BARS);
                          Message msg = mHandler.obtainMessage(MSG_REQUEST_TRANSIENT_BARS);
                          msg.arg1 = MSG_REQUEST_TRANSIENT_BARS_ARG_STATUS;
                          mHandler.sendMessageDelayed(msg, 500 /* delayMillis */);
                      }
  
                      @Override
                      public void onMouseHoverAtBottom() {
                          mHandler.removeMessages(MSG_REQUEST_TRANSIENT_BARS);
                          Message msg = mHandler.obtainMessage(MSG_REQUEST_TRANSIENT_BARS);
                          msg.arg1 = MSG_REQUEST_TRANSIENT_BARS_ARG_NAVIGATION;
                          mHandler.sendMessageDelayed(msg, 500 /* delayMillis */);
                      }
  
                      @Override
                      public void onMouseLeaveFromEdge() {
                          mHandler.removeMessages(MSG_REQUEST_TRANSIENT_BARS);
                      }
                  });
                
                  接下来修改为：
--- a/frameworks/base/services/core/java/com/android/server/wm/DisplayPolicy.java
+++ b/frameworks/base/services/core/java/com/android/server/wm/DisplayPolicy.java
@@ -178,7 +178,7 @@ import com.android.server.policy.WindowOrientationListener;
 import com.android.server.statusbar.StatusBarManagerInternal;
 import com.android.server.wallpaper.WallpaperManagerInternal;
 import com.android.server.wm.utils.InsetUtils;
-
+import android.util.Log;
 import java.io.PrintWriter;
 //unisoc: For Power Hint
 import android.os.PowerHintVendorSprd;
@@ -472,6 +472,7 @@ public class DisplayPolicy extends AbsDisplayPolicy {
                         if (mStatusBar != null) {
                             requestTransientBars(mStatusBar);
                         }
+                                               //mContext.sendBroadcast(new Intent("com.android.action.swipefromtop"));
                     }
 
                     @Override
@@ -488,6 +489,8 @@ public class DisplayPolicy extends AbsDisplayPolicy {
                                 updateShowHideNavSettings(true);
                             }
                         }
     //自定义上滑广播
+                        mContext.sendBroadcast(new Intent("com.android.action.swipefrombottom"));
+                                               Log.e("SuperPower","onSwipeFromBottom");
                     }

```

全局手势的处理然后然后在上滑事件 onSwipeFromBottom() 中增加自定义广播

### 3.2 StatusBar 监听自定义广播

StatusBar 中负责对导航栏布局的添加，所以可以在开机的时候默认不添加导航栏布局，然后通过注册监听自定义  
上滑的广播添加导航栏

```
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java
@@ -673,7 +673,7 @@ public class StatusBar extends SystemUI implements DemoMode,
             };
     private ActivityIntentHelper mActivityIntentHelper;
     private ShadeController mShadeController;
-
+    private RegisterStatusBarResult result = null;
     @Override
     public void onActiveStateChanged(int code, int uid, String packageName, boolean active) {
         Dependency.get(MAIN_HANDLER).post(() -> {
@@ -779,7 +779,6 @@ public class StatusBar extends SystemUI implements DemoMode,
         mCommandQueue = getComponent(CommandQueue.class);
         mCommandQueue.addCallback(this);
 
-        RegisterStatusBarResult result = null;
         try {
             result = mBarService.registerStatusBar(mCommandQueue);
         } catch (RemoteException ex) {
@@ -824,6 +823,8 @@ public class StatusBar extends SystemUI implements DemoMode,
         IntentFilter internalFilter = new IntentFilter();
         internalFilter.addAction(BANNER_ACTION_CANCEL);
         internalFilter.addAction(BANNER_ACTION_SETUP);
+        internalFilter.addAction("com.android.action.swipefrombottom");
         //internalFilter.addAction("com.android.action.swipefromtop");
         mContext.registerReceiver(mBannerActionBroadcastReceiver, internalFilter, PERMISSION_SELF,
                 null);
 
@@ -963,7 +964,7 @@ public class StatusBar extends SystemUI implements DemoMode,
         mNotificationLogger.setHeadsUpManager(mHeadsUpManager);
         putComponent(HeadsUpManager.class, mHeadsUpManager);
 
-        createNavigationBar(result);
+        //createNavigationBar(result);
 
         if (ENABLE_LOCKSCREEN_WALLPAPER) {
             mLockscreenWallpaper = new LockscreenWallpaper(mContext, this, mHandler);
@@ -4515,7 +4516,19 @@ public class StatusBar extends SystemUI implements DemoMode,
 
                     );
                 }
-            }
+            }else if("com.android.action.swipefrombottom".equals(action)){
+                 //上滑事件               android.util.Log.d("SuperPower","swipefrombottom-----");
                   //加载导航栏
+                 createNavigationBar(result);
+                                mHandler.postDelayed(new Runnable() {
+                    @Override
+                    public void run() {
// 移除导航栏
+                       mNavigationBarController.removeNavigationBar(mDisplayId);
+                    }
+                 },3000);
+                       }else if("com.android.action.swipefromtop".equals(action)){
+                                android.util.Log.d("SuperPower","swipefromtop---");
+                 下滑事件
//mNavigationBarController.removeNavigationBar(mDisplayId);
+                       }
         }
     };

```

导航栏布局就是 createNavigationBar(result); 在 stausbar 中不加载导航栏布局接收到广播显示导航栏，然后 3 秒钟移除导航栏  
移除导航栏通过 NavigationBarController.java 的 removeNavigationBar(int displayId) 方法  
然而系统默认是 private 需要改成 public 才能调用

### 3.3 修改隐藏导航栏方法

修改 removeNavigationBar 的修饰符

```
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/NavigationBarController.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/NavigationBarController.java
@@ -164,7 +164,7 @@ public class NavigationBarController implements Callbacks {
         });
     }
     //隐藏导航栏方法变为public 方便调用
-    private void removeNavigationBar(int displayId) {
+    public void removeNavigationBar(int displayId) {
         NavigationBarFragment navBar = mNavigationBars.get(displayId);
         if (navBar != null) {
             View navigationWindow = navBar.getView().getRootView();

```
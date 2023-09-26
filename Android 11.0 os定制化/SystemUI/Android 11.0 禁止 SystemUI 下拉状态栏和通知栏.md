> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125130188)

### 1. 概述

11.0 产品开发中，产品开发中对 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 状态栏开发有需求要求禁止下拉状态栏，防止通过下拉状态栏的设置点击进入系统原生设置页面，屏蔽系统原生设置

### 2. 禁止 SystemUI 下拉状态栏和通知栏的核心代码部分

```
framework/base/packages/apps/SystemUI/src/com/android/systemui/keyguard/KeyguardViewMediator.java
framework/base/packages/apps/SystemUI/src/com/android/systemui/statusbar/phone/CollapsedStatusBarFragment.java
framework/base/packages/apps/SystemUI/src/com/android/systemui/statusbar/phone/NotificationPanelView.java
/framework/base/packages/apps/SystemUI/src/com/android/systemui/statusbar/notification/NotificationStackScrollLayout.java


```

### 3. 禁止 SystemUI 下拉状态栏和通知栏的核心代码部分功能分析和实现

在 SystemUI 中可以在锁屏界面下拉状态栏也可以在进入 Launcher 桌面系统后通过下拉下拉出状态栏  
所以 SystemUI 禁止下拉状态栏 四部分修改:

### 3.1 未锁屏时禁止状态栏和通知栏下拉

未锁屏页面主要是通过 KeyguardViewMediator.java 来在 adjustStatusBarLocked(）中通过设置 StatusBarManager 的 flag  
属性来设置禁用下拉状态栏，然后在开机以后就禁用下拉状态栏达到禁用的目的  
framework/base/packages/apps/SystemUI/src/com/android/systemui/keyguard/KeyguardViewMediator.java  
代码分析如下：

```
private void handleShow(Bundle options) {
          Trace.beginSection("KeyguardViewMediator#handleShow");
          final int currentUser = KeyguardUpdateMonitor.getCurrentUser();
          if (mLockPatternUtils.isSecure(currentUser)) {
              mLockPatternUtils.getDevicePolicyManager().reportKeyguardSecured(currentUser);
          }
          synchronized (KeyguardViewMediator.this) {
              if (!mSystemReady) {
                  if (DEBUG) Log.d(TAG, "ignoring handleShow because system is not ready.");
                  return;
              } else {
                  if (DEBUG) Log.d(TAG, "handleShow");
              }
  
              mHiding = false;
              mWakeAndUnlocking = false;
              setShowingLocked(true);
              mKeyguardViewControllerLazy.get().show(options);
              resetKeyguardDonePendingLocked();
              mHideAnimationRun = false;
              adjustStatusBarLocked();
              userActivity();
              mUpdateMonitor.setKeyguardGoingAway(false);
              mKeyguardViewControllerLazy.get().setKeyguardGoingAwayState(false);
              mShowKeyguardWakeLock.release();
          }
          
private void adjustStatusBarLocked(boolean forceHideHomeRecentsButtons,
boolean forceClearFlags) {
if (mStatusBarManager == null) {
mStatusBarManager = (StatusBarManager)
mContext.getSystemService(Context.STATUS_BAR_SERVICE);
}

if (mStatusBarManager == null) {
Log.w(TAG, "Could not get status bar manager");
} else {
// Disable aspects of the system/status/navigation bars that must not be re-enabled by
// windows that appear on top, ever
int flags = StatusBarManager.DISABLE_NONE;

// TODO (b/155663717) After restart, status bar will not properly hide home button
//  unless disable is called to show un-hide it once first
if (forceClearFlags) {
mStatusBarManager.disable(flags);
}

if (forceHideHomeRecentsButtons || isShowingAndNotOccluded()) {
if (!mShowHomeOverLockscreen || !mInGestureNavigationMode) {
flags |= StatusBarManager.DISABLE_HOME;
}
flags |= StatusBarManager.DISABLE_RECENT;
}

if (DEBUG) {
Log.d(TAG, "adjustStatusBarLocked: mShowing=" + mShowing + " mOccluded=" + mOccluded
+ " isSecure=" + isSecure() + " force=" + forceHideHomeRecentsButtons
+  " --> flags=0x" + Integer.toHexString(flags));
}

mStatusBarManager.disable(flags);
}
}

```

在调用 show 显示状态栏的时候，在 adjustStatusBarLocked(）可以设置 mStatusBarManager 的 flag 为 StatusBarManager.DISABLE_EXPAND 表示禁用下拉状态栏  
具体修改如下:

```
@@ -2179,7 +2179,7 @@ public class KeyguardViewMediator extends SystemUI {
                         + " isSecure=" + isSecure() + " force=" + forceHideHomeRecentsButtons
                         +  " --> flags=0x" + Integer.toHexString(flags));
             }
-
+            flags = StatusBarManager.DISABLE_EXPAND;
             mStatusBarManager.disable(flags);
         }
     }

```

### 3.2 StatusBar 中不显示通知信息的 icon

在锁屏通知栏中去掉显示通知的部分，达到禁用下拉状态栏的功能  
主要实现如下:  
framework/base/packages/apps/SystemUI/src/com/android/systemui/statusbar/phone/CollapsedStatusBarFragment.java

```
@@ -162,7 +162,8 @@ public class CollapsedStatusBarFragment extends Fragment implements CommandQueue
             if ((state1 & DISABLE_NOTIFICATION_ICONS) != 0) {
                 hideNotificationIconArea(animate);
             } else {
-                showNotificationIconArea(animate);
+                //showNotificationIconArea(animate);
+                hideNotificationIconArea(animate);
             }
         }

```

### 3.3 锁屏时禁止状态栏下拉

在锁屏状态下禁用下拉状态栏 通知界面 NotificationPanelView.java 去掉下拉开展状态栏部分的功能

framework/base/packages/apps/SystemUI/src/com/android/systemui/statusbar/phone/NotificationPanelView.java

```
@@ -908,7 +908,7 @@ public class NotificationPanelView extends PanelView implements
         if (!isFullyCollapsed()) {
             handleQsDown(event);
         }
-        if (!mQsExpandImmediate && mQsTracking) {
+        if (!mKeyguardShowing && !mQsExpandImmediate && mQsTracking) {
             onQsTouch(event);
             if (!mConflictingQsExpansionGesture) {
                 return true;
@@ -1114,6 +1114,9 @@ public class NotificationPanelView extends PanelView implements
     }
 
     private void setQsExpanded(boolean expanded) {
+        if (mKeyguardShowing) {
+            return;
+        }
         boolean changed = mQsExpanded != expanded;
         if (changed) {
             mQsExpanded = expanded;
@@ -1508,7 +1511,7 @@ public class NotificationPanelView extends PanelView implements
         if (!mQsExpansionEnabled || mCollapsedOnDown) {
             return false;
         }
-        View header = mKeyguardShowing ? mKeyguardStatusBar : mQs.getHeader();
+        View header = /*mKeyguardShowing ? mKeyguardStatusBar :*/ mQs.getHeader();
         final boolean onHeader = x >= mQsFrame.getX()
                 && x <= mQsFrame.getX() + mQsFrame.getWidth()
                 && y >= header.getTop() && y <= header.getBottom();

```

### 3.4 锁屏时隐藏通知栏的显示

锁屏情况下 禁止显示通知栏 在收到通知的时候，不添加通知 布局不改变  
/framework/base/packages/apps/SystemUI/src/com/android/systemui/statusbar/notification/NotificationStackScrollLayout.java

```
@@ -717,7 +717,8 @@ public class NotificationStackScrollLayout extends ViewGroup
     }
 
     private void setMaxLayoutHeight(int maxLayoutHeight) {
-        mMaxLayoutHeight = maxLayoutHeight;
+        //mMaxLayoutHeight = maxLayoutHeight;
+        mMaxLayoutHeight = 0;
         mShelf.setMaxLayoutHeight(maxLayoutHeight);
         updateAlgorithmHeightAndPadding();
     }
@@ -2590,9 +2591,10 @@ public class NotificationStackScrollLayout extends ViewGroup
         } else {
             mTopPaddingOverflow = 0;
         }
-        setTopPadding(ignoreIntrinsicPadding ? topPadding : clampPadding(topPadding),
-                animate);
-        setExpandedHeight(mExpandedHeight);
+        //setTopPadding(ignoreIntrinsicPadding ? topPadding : clampPadding(topPadding),
+        //        animate);
+        //setExpandedHeight(mExpandedHeight);
+        setTopPadding(-500,animate);
     }

```
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124754837)

1. 概述
-----

在 11.0 12.0 的定制化开发系统手势功能的时候，客户需求要求在上滑手势的时候，在底部上滑时候进入系统桌面，也就是增加  
home 键功能 就可以实现这个功能了

2. 系统上滑手势增加 home 的功能的核心类
------------------------

```
frameworks/base/services/core/java/com/android/server/wm/DisplayPolicy.java

```

3. 系统上滑手势增加 home 的功能的核心功能分析和实现
------------------------------

首选在找到 11.0 的系统手势具体处理类 DisplayPolicy.java 就是系统处理手势的  
而主要是靠 mSystemGestures 来负责监听手势事件，然后处理上滑下滑左右滑事件  
路径为: frameworks/base/services/core/java/com/android/server/wm/DisplayPolicy.java

3.1DisplayPolicy.java 关于手势事件分析
------------------------------

```
mSystemGestures = new SystemGesturesPointerEventListener(mContext, mHandler,
                new SystemGesturesPointerEventListener.Callbacks() {
                    @Override
                    public void onSwipeFromTop() {
                        if (mStatusBar != null) {
                            requestTransientBars(mStatusBar);
                        }
                    }

                    @Override
                    public void onSwipeFromBottom() {
                        if (mNavigationBar != null && mNavigationBarPosition == NAV_BAR_BOTTOM) {
                            requestTransientBars(mNavigationBar);
                        }
                    }

                    @Override
                    public void onSwipeFromRight() {
                        final Region excludedRegion = Region.obtain();
                        synchronized (mLock) {
                            mDisplayContent.calculateSystemGestureExclusion(
                                    excludedRegion, null /* outUnrestricted */);
                        }
                        final boolean sideAllowed = mNavigationBarAlwaysShowOnSideGesture
                                || mNavigationBarPosition == NAV_BAR_RIGHT;
                        if (mNavigationBar != null && sideAllowed
                                && !mSystemGestures.currentGestureStartedInRegion(excludedRegion)) {
                            requestTransientBars(mNavigationBar);
                        }
                        excludedRegion.recycle();
                    }

                    @Override
                    public void onSwipeFromLeft() {
                        final Region excludedRegion = Region.obtain();
                        synchronized (mLock) {
                            mDisplayContent.calculateSystemGestureExclusion(
                                    excludedRegion, null /* outUnrestricted */);
                        }
                        final boolean sideAllowed = mNavigationBarAlwaysShowOnSideGesture
                                || mNavigationBarPosition == NAV_BAR_LEFT;
                        if (mNavigationBar != null && sideAllowed
                                && !mSystemGestures.currentGestureStartedInRegion(excludedRegion)) {
                            requestTransientBars(mNavigationBar);
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
        displayContent.registerPointerEventListener(mSystemGestures);
        displayContent.mAppTransition.registerListenerLocked(
                mStatusBarController.getAppTransitionListener());
        mImmersiveModeConfirmation = new ImmersiveModeConfirmation(mContext, looper,
                mService.mVrModeEnabled);
        mAcquireSleepTokenRunnable = () -> {
            if (mWindowSleepToken != null) {
                return;
            }
            mWindowSleepToken = service.mAtmInternal.acquireSleepToken(
                    "WindowSleepTokenOnDisplay" + displayId, displayId);
        };
        mReleaseSleepTokenRunnable = () -> {
            if (mWindowSleepToken == null) {
                return;
            }
            mWindowSleepToken.release();
            mWindowSleepToken = null;
        };

        // TODO: Make it can take screenshot on external display
        mScreenshotHelper = displayContent.isDefaultDisplay
                ? new ScreenshotHelper(mContext) : null;

        if (mDisplayContent.isDefaultDisplay) {
            mHasStatusBar = true;
            mHasNavigationBar = mContext.getResources().getBoolean(R.bool.config_showNavigationBar);

            // Allow a system property to override this. Used by the emulator.
            // See also hasNavigationBar().
            String navBarOverride = SystemProperties.get("qemu.hw.mainkeys");
            if ("1".equals(navBarOverride)) {
                mHasNavigationBar = false;
            } else if ("0".equals(navBarOverride)) {
                mHasNavigationBar = true;
            }
        } else {
            mHasStatusBar = false;
            mHasNavigationBar = mDisplayContent.supportsSystemDecorations();
        }

        mRefreshRatePolicy = new RefreshRatePolicy(mService,
                mDisplayContent.getDisplayInfo(),
                mService.mHighRefreshRateBlacklist);
    }

```

在上面代码发现 DisplayPolicy 中的  
SystemGesturesPointerEventListener 就是具体处理系统手势的 onSwipeFromBottom()  
就是上滑事件的处理 所以就在这里增加 Home 事件就可以了，就可以模拟 home 功能

具体修改如下:

```
--- a/frameworks/base/services/core/java/com/android/server/wm/DisplayPolicy.java

+++ b/frameworks/base/services/core/java/com/android/server/wm/DisplayPolicy.java

@@ -178,11 +178,11 @@ import com.android.server.policy.WindowOrientationListener;

 import com.android.server.statusbar.StatusBarManagerInternal;

 import com.android.server.wallpaper.WallpaperManagerInternal;

 import com.android.server.wm.utils.InsetUtils;

-

+import android.view.KeyEvent;

 import java.io.PrintWriter;

 //unisoc: For Power Hint

 import android.os.PowerHintVendorSprd;

-

+import android.provider.Settings;

 /**

  * The policy that provides the basic behaviors and states of a display to show UI.

  */

@@ -472,6 +472,7 @@ public class DisplayPolicy extends AbsDisplayPolicy {

                         if (mStatusBar != null) {

                             requestTransientBars(mStatusBar);

                         }

+

                     }

 

                     @Override

@@ -488,6 +489,17 @@ public class DisplayPolicy extends AbsDisplayPolicy {

                                 updateShowHideNavSettings(true);

                             }

                         }

+                                          //add code start


+                                                 long now = SystemClock.uptimeMillis();

+                          KeyEvent down =  new KeyEvent(now, now,KeyEvent.ACTION_DOWN, 3, 0);

+                          KeyEvent up = new KeyEvent(now, now,KeyEvent.ACTION_UP, 3, 0);

+                          InputManager.getInstance().injectInputEvent(down, InputManager.INJECT_INPUT_EVENT_MODE_ASYNC);

+                                                 InputManager.getInstance().injectInputEvent(up, InputManager.INJECT_INPUT_EVENT_MODE_ASYNC);

+                                           //add code end

                     }

```

然后编译 services.jar 替换就可以了
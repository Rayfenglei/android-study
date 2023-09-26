> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124889258)

### 1. 概述

在 11.0 的 framework 系统全局手势事件也是系统非常重要的功能，但是当隐藏状态栏，  
当把状态栏高度设置为 0 时，这时全局手势事件失效，这就要从系统手势滑动流程来分析 看怎么样实现  
系统手势功能的，然后根据功能做修改

### 2. 状态栏高度为 0 时，系统全局手势失效的解决方案的核心代码

```
frameworks/base/services/core/java/com/android/server/wm/DisplayPolicy.java
frameworks/base/services/core/java/com/android/server/wm/SystemGesturesPointerEventListener.java

```

### 3. 状态栏高度为 0 时，系统全局手势失效的解决方案的功能分析和实现

### 3.1DisplayPolicy.java 中全局手势事件的响应流程分析

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
                            /**
                             * unisoc bug 1074912: add for dynamic navigationbar
                             * dont need to layout when is keyguard(who is hide navidationbar) & navigationbar showing.
                             * showNavigationbar can layout&change provider.then mShowHideNavObserver also can layout.so just modify provider here.
                             */
                            if (mDynamicNavigationBar&&!mService.isKeyguardShowingAndNotOccluded()&&!isNavigationBarShowing()) {
                                Slog.d(TAG," onSwipeFromBottom show = true");
                                updateShowHideNavSettings(true);
                            }
                        }
                    }

                    @Override
                    public void onSwipeFromRight() {
                        final Region excludedRegion;
                        synchronized (mLock) {
                            excludedRegion = mDisplayContent.calculateSystemGestureExclusion();
                        }
                        final boolean sideAllowed = mNavigationBarAlwaysShowOnSideGesture
                                || mNavigationBarPosition == NAV_BAR_RIGHT;
                        if (mNavigationBar != null && sideAllowed
                                && !mSystemGestures.currentGestureStartedInRegion(excludedRegion)) {
                            requestTransientBars(mNavigationBar);
                        }
                        /**
                         * unisoc bug 1074912/1145137: add for dynamic navigationbar
                         * dont need to layout when is keyguard(who is hide navidationbar) & navigationbar showing.
                         * showNavigationbar can layout&change provider.then mShowHideNavObserver also can layout.so just modify provider here.
                         */
                        if ((mNavigationBarPosition == NAV_BAR_RIGHT)&&mDynamicNavigationBar
                                &&!mService.isKeyguardShowingAndNotOccluded()
                                &&!isNavigationBarShowing()) {
                            Slog.d(TAG," onSwipeFromRight show = true");
                            updateShowHideNavSettings(true);
                         }
                    }

                    @Override
                    public void onSwipeFromLeft() {
                        final Region excludedRegion;
                        synchronized (mLock) {
                            excludedRegion = mDisplayContent.calculateSystemGestureExclusion();
                        }
                        final boolean sideAllowed = mNavigationBarAlwaysShowOnSideGesture
                                || mNavigationBarPosition == NAV_BAR_LEFT;
                        if (mNavigationBar != null && sideAllowed
                                && !mSystemGestures.currentGestureStartedInRegion(excludedRegion)) {
                            requestTransientBars(mNavigationBar);
                        }
                        /**
                         * unisoc bug 1074912/1145137: add for dynamic navigationbar
                         * dont need to layout when is keyguard(who is hide navidationbar) & navigationbar showing.
                         * showNavigationbar can layout&change provider.then mShowHideNavObserver also can layout.so just modify provider here.
                         */
                        if ((mNavigationBarPosition == NAV_BAR_LEFT)&&mDynamicNavigationBar
                                &&!mService.isKeyguardShowingAndNotOccluded()
                                &&!isNavigationBarShowing()) {
                            Slog.d(TAG," onSwipeFromLeft show = true");
                            updateShowHideNavSettings(true);
                        }
                    }

                    @Override
                    public void onFling(int duration) {
                        if (mPerformance != null) mPerformance.scheduleBoostWhenTouch();
                        if (mService.mPowerManagerInternal != null) {
                            //unisoc: modify scene ID
                            mService.mPowerManagerInternal.powerHint(
                                    PowerHintVendorSprd.POWER_HINT_VENDOR_INTERACTION_FLING, duration);
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
                        if (mPerformance != null) mPerformance.scheduleBoostWhenTouch();
                        final WindowOrientationListener listener = getOrientationListener();
                        if (listener != null) {
                            listener.onTouchStart();
                        }

                    }

                    @Override
                    public void onUpOrCancel() {
                        if (mPerformance != null) mPerformance.scheduleBoostWhenTouch();
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

```

看代码可知 SystemGesturesPointerEventListener 的构造函数是通过 SystemGesturesPointerEventListener.[Callbacks](https://so.csdn.net/so/search?q=Callbacks&spm=1001.2101.3001.7020)() 来回调事件继续跟踪 SystemGesturesPointerEventListener.java 代码:

### 3.2 SystemGesturesPointerEventListener.java 的相关代码分析

```
 private final Callbacks mCallbacks;
    SystemGesturesPointerEventListener(Context context, Handler handler, Callbacks callbacks) {
        mContext = checkNull("context", context);
        mHandler = handler;
        mCallbacks = checkNull("callbacks", callbacks);

        onConfigurationChanged();
    }


在初始化时callbacks的值赋值给mCallbacks，用这个对象来跟踪手势事件

@Override
    public void onPointerEvent(MotionEvent event) {
        if (mGestureDetector != null && event.isTouchEvent()) {
            mGestureDetector.onTouchEvent(event);
        }
        switch (event.getActionMasked()) {
            case MotionEvent.ACTION_DOWN:
                mSwipeFireable = true;
                mDebugFireable = true;
                mDownPointers = 0;
                captureDown(event, 0);
                if (mMouseHoveringAtEdge) {
                    mMouseHoveringAtEdge = false;
                    mCallbacks.onMouseLeaveFromEdge();
                }
                mCallbacks.onDown();
                break;
            case MotionEvent.ACTION_POINTER_DOWN:
                captureDown(event, event.getActionIndex());
                if (mDebugFireable) {
                    mDebugFireable = event.getPointerCount() < 5;
                    if (!mDebugFireable) {
                        if (DEBUG) Slog.d(TAG, "Firing debug");
                        mCallbacks.onDebug();
                    }
                }
                break;
            case MotionEvent.ACTION_MOVE:
                if (mSwipeFireable) {
                    final int swipe = detectSwipe(event);
                    mSwipeFireable = swipe == SWIPE_NONE;
                    if (swipe == SWIPE_FROM_TOP) {
                        if (DEBUG) Slog.d(TAG, "Firing onSwipeFromTop");
                        mCallbacks.onSwipeFromTop();
                    } else if (swipe == SWIPE_FROM_BOTTOM) {
                        if (DEBUG) Slog.d(TAG, "Firing onSwipeFromBottom");
                        mCallbacks.onSwipeFromBottom();
                    } else if (swipe == SWIPE_FROM_RIGHT) {
                        if (DEBUG) Slog.d(TAG, "Firing onSwipeFromRight");
                        mCallbacks.onSwipeFromRight();
                    } else if (swipe == SWIPE_FROM_LEFT) {
                        if (DEBUG) Slog.d(TAG, "Firing onSwipeFromLeft");
                        mCallbacks.onSwipeFromLeft();
                    }
                }
                break;
            case MotionEvent.ACTION_HOVER_MOVE:
                if (event.isFromSource(InputDevice.SOURCE_MOUSE)) {
                    if (!mMouseHoveringAtEdge && event.getY() == 0) {
                        mCallbacks.onMouseHoverAtTop();
                        mMouseHoveringAtEdge = true;
                    } else if (!mMouseHoveringAtEdge && event.getY() >= screenHeight - 1) {
                        mCallbacks.onMouseHoverAtBottom();
                        mMouseHoveringAtEdge = true;
                    } else if (mMouseHoveringAtEdge
                            && (event.getY() > 0 && event.getY() < screenHeight - 1)) {
                        mCallbacks.onMouseLeaveFromEdge();
                        mMouseHoveringAtEdge = false;
                    }
                }
                break;
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL:
                mSwipeFireable = false;
                mDebugFireable = false;
                mCallbacks.onUpOrCancel();
                break;
            default:
                if (DEBUG) Slog.d(TAG, "Ignoring " + event);
        }
    }

```

在 手势移动 MotionEvent.ACTION_MOVE 中 根据 detectSwipe(event) 返回的值来  
返回手势事件

```
private int detectSwipe(MotionEvent move) {
        final int historySize = move.getHistorySize();
        final int pointerCount = move.getPointerCount();
        for (int p = 0; p < pointerCount; p++) {
            final int pointerId = move.getPointerId(p);
            final int i = findIndex(pointerId);
            if (i != UNTRACKED_POINTER) {
                for (int h = 0; h < historySize; h++) {
                    final long time = move.getHistoricalEventTime(h);
                    final float x = move.getHistoricalX(p, h);
                    final float y = move.getHistoricalY(p,  h);
                    final int swipe = detectSwipe(i, time, x, y);
                    if (swipe != SWIPE_NONE) {
                        return swipe;
                    }
                }
                final int swipe = detectSwipe(i, move.getEventTime(), move.getX(p), move.getY(p));
                if (swipe != SWIPE_NONE) {
                    return swipe;
                }
            }
        }
        return SWIPE_NONE;
    }

    private int detectSwipe(int i, long time, float x, float y) {
        final float fromX = mDownX[i];
        final float fromY = mDownY[i];
        final long elapsed = time - mDownTime[i];
        if (DEBUG) Slog.d(TAG, "pointer " + mDownPointerId[i]
                + " moved (" + fromX + "->" + x + "," + fromY + "->" + y + ") in " + elapsed);
        if (fromY <= mSwipeStartThreshold
                && y > fromY + mSwipeDistanceThreshold
                && elapsed < SWIPE_TIMEOUT_MS) {
            return SWIPE_FROM_TOP;
        }
        if (fromY >= screenHeight - mSwipeStartThreshold
                && y < fromY - mSwipeDistanceThreshold
                && elapsed < SWIPE_TIMEOUT_MS) {
            return SWIPE_FROM_BOTTOM;
        }
        if (fromX >= screenWidth - mSwipeStartThreshold
                && x < fromX - mSwipeDistanceThreshold
                && elapsed < SWIPE_TIMEOUT_MS) {
            return SWIPE_FROM_RIGHT;
        }
        if (fromX <= mSwipeStartThreshold
                && x > fromX + mSwipeDistanceThreshold
                && elapsed < SWIPE_TIMEOUT_MS) {
            return SWIPE_FROM_LEFT;
        }
        return SWIPE_NONE;
    }

```

而 detectSwipe 又会根据 mSwipeDistanceThreshold 的值来判断当前是什么事件

跟踪 mSwipeDistanceThreshold 的值 在 onConfigurationChanged() 中 又会加上状态栏的原始高度

```
void onConfigurationChanged() {
        mSwipeStartThreshold = mContext.getResources()
                .getDimensionPixelSize(com.android.internal.R.dimen.status_bar_height);

        final Display display = DisplayManagerGlobal.getInstance()
                .getRealDisplay(Display.DEFAULT_DISPLAY);
        final DisplayCutout displayCutout = display.getCutout();
        if (displayCutout != null) {
            final Rect bounds = displayCutout.getBoundingRectTop();
            if (!bounds.isEmpty()) {
                // Expand swipe start threshold such that we can catch touches that just start below
                // the notch area
                mDisplayCutoutTouchableRegionSize = mContext.getResources().getDimensionPixelSize(
                        com.android.internal.R.dimen.display_cutout_touchable_region_size);
                mSwipeStartThreshold += mDisplayCutoutTouchableRegionSize;
            }
        }
        mSwipeDistanceThreshold = mSwipeStartThreshold;
        if (DEBUG) Slog.d(TAG,  "mSwipeStartThreshold=" + mSwipeStartThreshold
            + " mSwipeDistanceThreshold=" + mSwipeDistanceThreshold);
    }

```

所有如果 status_bar_height 设置为 0，全局手势事件就会失效，  
解决方案如下:

```
diff --git a/frameworks/base/core/res/res/values/dimens.xml b/frameworks/base/core/res/res/values/dimens.xml
index 7b54bfb87e..946b26b9d5 100755
--- a/frameworks/base/core/res/res/values/dimens.xml
+++ b/frameworks/base/core/res/res/values/dimens.xml
@@ -31,6 +31,7 @@
     <dimen >48dip</dimen>
 
     <dimen >24dp</dimen>
+        <dimen >24dp</dimen><!--add code -->
     <!-- Height of the status bar -->
     <dimen >@dimen/status_bar_height_portrait</dimen>
     <!-- Height of the status bar in portrait -->

diff --git a/frameworks/base/core/res/res/values/symbols.xml b/frameworks/base/core/res/res/values/symbols.xml
old mode 100644
new mode 100755
index f18cb5a26f..273c87b7e0
--- a/frameworks/base/core/res/res/values/symbols.xml
+++ b/frameworks/base/core/res/res/values/symbols.xml
@@ -148,6 +148,7 @@
   <java-symbol type="dimen"  />
   <java-symbol type="dimen"  />
   <java-symbol type="dimen"  />
+  <java-symbol type="dimen"  />
   <java-symbol type="bool"  />
   <java-symbol type="integer"  />
   <java-symbol type="integer"  />

diff --git a/frameworks/base/services/core/java/com/android/server/wm/SystemGesturesPointerEventListener.java b/frameworks/base/services/core/java/com/android/server/wm/SystemGesturesPointerEventListener.java
old mode 100644
new mode 100755
index fb781b06f0..0708753b81
--- a/frameworks/base/services/core/java/com/android/server/wm/SystemGesturesPointerEventListener.java
+++ b/frameworks/base/services/core/java/com/android/server/wm/SystemGesturesPointerEventListener.java
@@ -80,7 +80,7 @@ class SystemGesturesPointerEventListener implements PointerEventListener {
   重点部分，重新添加一个新的状态栏高度就行了
     void onConfigurationChanged() {
         mSwipeStartThreshold = mContext.getResources()
-                .getDimensionPixelSize(com.android.internal.R.dimen.status_bar_height);
+                .getDimensionPixelSize(com.android.internal.R.dimen.status_bar_height_custom);// status_bar_height
 
         final Display display = DisplayManagerGlobal.getInstance()
                 .getRealDisplay(Display.DEFAULT_DISPLAY);

```
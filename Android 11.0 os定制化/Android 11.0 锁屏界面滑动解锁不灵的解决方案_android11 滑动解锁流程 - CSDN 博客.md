> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/128028004)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 锁屏界面滑动解锁不灵的解决方案的核心类](#t1)

[3. 锁屏界面滑动解锁不灵的解决方案的核心功能分析和实现](#t2)

1. 概述
-----

  在 11.0 的系统产品开发中，锁屏界面默认是上滑解锁进入 [Launcher](https://so.csdn.net/so/search?q=Launcher&spm=1001.2101.3001.7020) 页面的，原生的上滑解锁不太好用解锁有点困难，所以产品需求要求查找源码解决这个问题，所以这就需要从滑动解锁流程分析来解决问题

2. 锁屏界面滑动解锁不灵的解决方案的核心类
----------------------

```
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/PanelViewController.java

```

3. 锁屏界面滑动解锁不灵的解决方案的核心功能分析和实现
----------------------------

  在 SystemUI 中关于滑动解锁上滑事件的处理都是在 PanelViewController.java 中处理的，首选看下  
PanelViewController.java 中的 onTounch 事件的处理，来分析相关源码如下

```
         @Override
          public boolean onTouch(View v, MotionEvent event) {
              if (mInstantExpanding || (mTouchDisabled
                      && event.getActionMasked() != MotionEvent.ACTION_CANCEL) || (mMotionAborted
                      && event.getActionMasked() != MotionEvent.ACTION_DOWN)) {
                  return false;
              }
  
              // If dragging should not expand the notifications shade, then return false.
              if (!mNotificationsDragEnabled) {
                  if (mTracking) {
                      // Turn off tracking if it's on or the shade can get stuck in the down position.
                      onTrackingStopped(true /* expand */);
                  }
                  return false;
              }
  
              // On expanding, single mouse click expands the panel instead of dragging.
              if (isFullyCollapsed() && event.isFromSource(InputDevice.SOURCE_MOUSE)) {
                  if (event.getAction() == MotionEvent.ACTION_UP) {
                      expand(true);
                  }
                  return true;
              }
  
              /*
               * We capture touch events here and update the expand height here in case according to
               * the users fingers. This also handles multi-touch.
               *
               * Flinging is also enabled in order to open or close the shade.
               */
  
              int pointerIndex = event.findPointerIndex(mTrackingPointer);
              if (pointerIndex < 0) {
                  pointerIndex = 0;
                  mTrackingPointer = event.getPointerId(pointerIndex);
              }
              final float x = event.getX(pointerIndex);
              final float y = event.getY(pointerIndex);
  
              if (event.getActionMasked() == MotionEvent.ACTION_DOWN) {
                  mGestureWaitForTouchSlop = shouldGestureWaitForTouchSlop();
                  mIgnoreXTouchSlop = isFullyCollapsed() || shouldGestureIgnoreXTouchSlop(x, y);
              }
  
              switch (event.getActionMasked()) {
                  case MotionEvent.ACTION_DOWN:
                      startExpandMotion(x, y, false /* startTracking */, mExpandedHeight);
                      mMinExpandHeight = 0.0f;
                      mPanelClosedOnDown = isFullyCollapsed();
                      mHasLayoutedSinceDown = false;
                      mUpdateFlingOnLayout = false;
                      mMotionAborted = false;
                      mDownTime = SystemClock.uptimeMillis();
                      mTouchAboveFalsingThreshold = false;
                      mCollapsedAndHeadsUpOnDown =
                              isFullyCollapsed() && mHeadsUpManager.hasPinnedHeadsUp();
                      addMovement(event);
                      boolean regularHeightAnimationRunning = mHeightAnimator != null
                              && !mHintAnimationRunning && !mIsSpringBackAnimation;
                      if (!mGestureWaitForTouchSlop || regularHeightAnimationRunning) {
                          mTouchSlopExceeded = regularHeightAnimationRunning
                                          || mTouchSlopExceededBeforeDown;
                          cancelHeightAnimator();
                          onTrackingStarted();
                      }
                      if (isFullyCollapsed() && !mHeadsUpManager.hasPinnedHeadsUp()
                              && !mStatusBar.isBouncerShowing()) {
                          startOpening(event);
                      }
                      break;
  
                  case MotionEvent.ACTION_POINTER_UP:
                      final int upPointer = event.getPointerId(event.getActionIndex());
                      if (mTrackingPointer == upPointer) {
                          // gesture is ongoing, find a new pointer to track
                          final int newIndex = event.getPointerId(0) != upPointer ? 0 : 1;
                          final float newY = event.getY(newIndex);
                          final float newX = event.getX(newIndex);
                          mTrackingPointer = event.getPointerId(newIndex);
                          mHandlingPointerUp = true;
                          startExpandMotion(newX, newY, true /* startTracking */, mExpandedHeight);
                          mHandlingPointerUp = false;
                      }
                      break;
                  case MotionEvent.ACTION_POINTER_DOWN:
                      if (mStatusBarStateController.getState() == StatusBarState.KEYGUARD) {
                          mMotionAborted = true;
                          endMotionEvent(event, x, y, true /* forceCancel */);
                          return false;
                      }
                      break;
                  case MotionEvent.ACTION_MOVE:
                      addMovement(event);
                      float h = y - mInitialTouchY;
  
                      // If the panel was collapsed when touching, we only need to check for the
                      // y-component of the gesture, as we have no conflicting horizontal gesture.
                      if (Math.abs(h) > getTouchSlop(event)
                              && (Math.abs(h) > Math.abs(x - mInitialTouchX)
                              || mIgnoreXTouchSlop)) {
                          mTouchSlopExceeded = true;
                          if (mGestureWaitForTouchSlop && !mTracking && !mCollapsedAndHeadsUpOnDown) {
                              if (mInitialOffsetOnTouch != 0f) {
                                  startExpandMotion(x, y, false /* startTracking */, mExpandedHeight);
                                  h = 0;
                              }
                              cancelHeightAnimator();
                              onTrackingStarted();
                          }
                      }
                      float newHeight = Math.max(0, h + mInitialOffsetOnTouch);
                      newHeight = Math.max(newHeight, mMinExpandHeight);
                      if (-h >= getFalsingThreshold()) {
                          mTouchAboveFalsingThreshold = true;
                          mUpwardsWhenThresholdReached = isDirectionUpwards(x, y);
                      }
                      if ((!mGestureWaitForTouchSlop || mTracking) && !isTrackingBlocked()) {
                          setExpandedHeightInternal(newHeight);
                      }
                      break;
  
                  case MotionEvent.ACTION_UP:
                  case MotionEvent.ACTION_CANCEL:
                      addMovement(event);
                      endMotionEvent(event, x, y, false /* forceCancel */);
                      // mHeightAnimator is null, there is no remaining frame, ends instrumenting.
                      if (mHeightAnimator == null) {
                          if (event.getActionMasked() == MotionEvent.ACTION_UP) {
                              endJankMonitoring(CUJ_NOTIFICATION_SHADE_EXPAND_COLLAPSE);
                          } else {
                              cancelJankMonitoring(CUJ_NOTIFICATION_SHADE_EXPAND_COLLAPSE);
                          }
                      }
                      break;
              }
              return !mGestureWaitForTouchSlop || mTracking;
          }
```

在 PanelViewController.java 的 onTouch 的上滑手势中，在 endMotionEvent(event, x, y, false /* forceCancel */); 处理上滑解锁功能，根据 x y 的值判断滑动解锁是否在一定值的范围内，然后解锁  
接下来看 endMotionEvent 的相关方法分析代码

```
private void endMotionEvent(MotionEvent event, float x, float y, boolean forceCancel) {
          mTrackingPointer = -1;
          if ((mTracking && mTouchSlopExceeded) || Math.abs(x - mInitialTouchX) > mTouchSlop
                  || Math.abs(y - mInitialTouchY) > mTouchSlop
                  || event.getActionMasked() == MotionEvent.ACTION_CANCEL || forceCancel) {
              mVelocityTracker.computeCurrentVelocity(1000);
              float vel = mVelocityTracker.getYVelocity();
              float vectorVel = (float) Math.hypot(
                      mVelocityTracker.getXVelocity(), mVelocityTracker.getYVelocity());
  
              final boolean onKeyguard =
                      mStatusBarStateController.getState() == StatusBarState.KEYGUARD;
  
              final boolean expand;
              if (event.getActionMasked() == MotionEvent.ACTION_CANCEL || forceCancel) {
                  // If we get a cancel, put the shade back to the state it was in when the gesture
                  // started
                  if (onKeyguard) {
                      expand = true;
                  } else {
                      expand = !mPanelClosedOnDown;
                  }
              } else {
                  expand = flingExpands(vel, vectorVel, x, y);
              }
  
              mDozeLog.traceFling(expand, mTouchAboveFalsingThreshold,
                      mStatusBar.isFalsingThresholdNeeded(), mStatusBar.isWakeUpComingFromTouch());
              // Log collapse gesture if on lock screen.
              if (!expand && onKeyguard) {
                  float displayDensity = mStatusBar.getDisplayDensity();
                  int heightDp = (int) Math.abs((y - mInitialTouchY) / displayDensity);
                  int velocityDp = (int) Math.abs(vel / displayDensity);
                  mLockscreenGestureLogger.write(MetricsEvent.ACTION_LS_UNLOCK, heightDp, velocityDp);
                  mLockscreenGestureLogger.log(LockscreenUiEvent.LOCKSCREEN_UNLOCK);
              }
              @Classifier.InteractionType int interactionType = vel == 0 ? GENERIC
                      : vel > 0 ? QUICK_SETTINGS
                              : (mKeyguardStateController.canDismissLockScreen()
                                      ? UNLOCK : BOUNCER_UNLOCK);
  
              fling(vel, expand, isFalseTouch(x, y, interactionType));
              onTrackingStopped(expand);
              mUpdateFlingOnLayout = expand && mPanelClosedOnDown && !mHasLayoutedSinceDown;
              if (mUpdateFlingOnLayout) {
                  mUpdateFlingVelocity = vel;
              }
          } else if (!mStatusBar.isBouncerShowing()
                  && !mStatusBarKeyguardViewManager.isShowingAlternateAuthOrAnimating()) {
              boolean expands = onEmptySpaceClick(mInitialTouchX);
              onTrackingStopped(expands);
          }
  
          mVelocityTracker.clear();
      }
```

在 PanelViewController.java 中的 endMotionEvent 中，根据上滑滑动手势的速度和手势的 x y 的值，来计算是否上滑的速度和距离达到解锁的值来实现上滑解锁功能，而在 fling(vel, expand, isFalseTouch(x, y, interactionType)); 来根据坐标值计算是否可以解锁，接下来看下 fling(vel, expand, isFalseTouch(x, y, interactionType)); 的相关源码分析问题  
主要取决于 isFalseTouch(）的值

```
private boolean isFalseTouch(float x, float y,
              @Classifier.InteractionType int interactionType) {
          if (!mStatusBar.isFalsingThresholdNeeded()) {
              return false;
          }
          if (mFalsingManager.isClassifierEnabled()) {
              return mFalsingManager.isFalseTouch(interactionType);
          }
          if (!mTouchAboveFalsingThreshold) {
              return true;
          }
          if (mUpwardsWhenThresholdReached) {
              return false;
          }
          return !isDirectionUpwards(x, y);
      }
```

在 isFalseTouch(）中 根据相关的日志发现主要是由于 mFalsingManager.isFalseTouch(interactionType);  
返回 true 导致解锁困难，所以可以注释掉这段代码，其他条件满足就可以了  
具体修改为:

```
private boolean isFalseTouch(float x, float y,
              @Classifier.InteractionType int interactionType) {
          if (!mStatusBar.isFalsingThresholdNeeded()) {
              return false;
          }
         /*if (mFalsingManager.isClassifierEnabled()) {
              return mFalsingManager.isFalseTouch(interactionType);
          }*/
          if (!mTouchAboveFalsingThreshold) {
              return true;
          }
          if (mUpwardsWhenThresholdReached) {
              return false;
          }
          return !isDirectionUpwards(x, y);
      }
```
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125417572)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 核心代码](#t1)

[3. 核心代码功能实现分析](#t2)

 [3.1NotificationPanelViewController.java 相关代码分析](#t3)

[3.2 在 onTouch 事件的处理](#t4)

[3.3 关于 StatusBar.java 中关于点击返回键 收起下拉框的处理](#t5)

1. 概述
=====

在 11.0 定制 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 下拉状态栏的时候 ，需要默认展开下拉框 显示出所以的下拉快捷图标 这就要从 NotificationPanelView.java 中 下拉事件处理 而在 11.0 中下拉事件全都有 NotificationPanelViewController.java 来处理了

2. 核心代码
=======

```
主要代码为:
/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NotificationPanelViewController.java
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java
```

3. 核心代码功能实现分析
=============

###  3.1NotificationPanelViewController.java 相关代码分析

```
@Override
protected TouchHandler createTouchHandler() {
return new TouchHandler() {
@Override
public boolean onInterceptTouchEvent(MotionEvent event) {
if (mBlockTouches || mQsFullyExpanded && mQs.disallowPanelTouches()) {
return false;
}
initDownStates(event);
// Do not let touches go to shade or QS if the bouncer is visible,
// but still let user swipe down to expand the panel, dismissing the bouncer.
if (mStatusBar.isBouncerShowing()) {
return true;
}
if (mBar.panelEnabled() && mHeadsUpTouchHelper.onInterceptTouchEvent(event)) {
mMetricsLogger.count(COUNTER_PANEL_OPEN, 1);
mMetricsLogger.count(COUNTER_PANEL_OPEN_PEEK, 1);
return true;
}
if (!shouldQuickSettingsIntercept(mDownX, mDownY, 0)
&& mPulseExpansionHandler.onInterceptTouchEvent(event)) {
return true;
}
 
if (!isFullyCollapsed() && onQsIntercept(event)) {
return true;
}
return super.onInterceptTouchEvent(event);
}
 
@Override
public boolean onTouch(View v, MotionEvent event) {
if (mBlockTouches || (mQsFullyExpanded && mQs != null
&& mQs.disallowPanelTouches())) {
return false;
}
 
// Do not allow panel expansion if bouncer is scrimmed, otherwise user would be able
// to pull down QS or expand the shade.
if (mStatusBar.isBouncerShowingScrimmed()) {
return false;
}
 
// Make sure the next touch won't the blocked after the current ends.
if (event.getAction() == MotionEvent.ACTION_UP
|| event.getAction() == MotionEvent.ACTION_CANCEL) {
mBlockingExpansionForCurrentTouch = false;
}
// When touch focus transfer happens, ACTION_DOWN->ACTION_UP may happen immediately
// without any ACTION_MOVE event.
// In such case, simply expand the panel instead of being stuck at the bottom bar.
if (mLastEventSynthesizedDown && event.getAction() == MotionEvent.ACTION_UP) {
expand(true /* animate */);
}
initDownStates(event);
if (!mIsExpanding && !shouldQuickSettingsIntercept(mDownX, mDownY, 0)
&& mPulseExpansionHandler.onTouchEvent(event)) {
// We're expanding all the other ones shouldn't get this anymore
return true;
}
if (mListenForHeadsUp && !mHeadsUpTouchHelper.isTrackingHeadsUp()
&& mHeadsUpTouchHelper.onInterceptTouchEvent(event)) {
mMetricsLogger.count(COUNTER_PANEL_OPEN_PEEK, 1);
}
boolean handled = false;
if ((!mIsExpanding || mHintAnimationRunning) && !mQsExpanded
&& mBarState != StatusBarState.SHADE && !mDozing) {
handled |= mAffordanceHelper.onTouchEvent(event);
}
if (mOnlyAffordanceInThisMotion) {
return true;
}
handled |= mHeadsUpTouchHelper.onTouchEvent(event);
if (!mHeadsUpTouchHelper.isTrackingHeadsUp() && handleQsTouch(event)) {
return true;
}
if (event.getActionMasked() == MotionEvent.ACTION_DOWN && isFullyCollapsed()) {
mMetricsLogger.count(COUNTER_PANEL_OPEN, 1);
updateVerticalPanelPosition(event.getX());
handled = true;
}
handled |= super.onTouch(v, event);
return !mDozing || mPulsing || handled;
}
};
}
private boolean onQsIntercept(MotionEvent event) {
        int pointerIndex = event.findPointerIndex(mTrackingPointer);
        if (pointerIndex < 0) {
            pointerIndex = 0;
            mTrackingPointer = event.getPointerId(pointerIndex);
        }
        final float x = event.getX(pointerIndex);
        final float y = event.getY(pointerIndex);
        switch (event.getActionMasked()) {
            case MotionEvent.ACTION_DOWN:
                mIntercepting = true;
                mInitialTouchY = y;
                mInitialTouchX = x;
                initVelocityTracker();
                trackMovement(event);
                if (shouldQuickSettingsIntercept(mInitialTouchX, mInitialTouchY, 0)) {
                    getParent().requestDisallowInterceptTouchEvent(true);
                }
                if (mQsExpansionAnimator != null) {
                    onQsExpansionStarted();
                    mInitialHeightOnTouch = mQsExpansionHeight;
                    mQsTracking = true;
                    mIntercepting = false;
                    mNotificationStackScroller.cancelLongPress();
                }
                break;
            case MotionEvent.ACTION_POINTER_UP:
                final int upPointer = event.getPointerId(event.getActionIndex());
                if (mTrackingPointer == upPointer) {
                    // gesture is ongoing, find a new pointer to track
                    final int newIndex = event.getPointerId(0) != upPointer ? 0 : 1;
                    mTrackingPointer = event.getPointerId(newIndex);
                    mInitialTouchX = event.getX(newIndex);
                    mInitialTouchY = event.getY(newIndex);
                }
                break;
            case MotionEvent.ACTION_MOVE:
                final float h = y - mInitialTouchY;
                trackMovement(event);
                if (mQsTracking) {
                    // Already tracking because onOverscrolled was called. We need to update here
                    // so we don't stop for a frame until the next touch event gets handled in
                    // onTouchEvent.
                    setQsExpansion(h + mInitialHeightOnTouch);
                    trackMovement(event);
                    mIntercepting = false;
                    return true;
                }
                if (Math.abs(h) > mTouchSlop && Math.abs(h) > Math.abs(x - mInitialTouchX)
                        && shouldQuickSettingsIntercept(mInitialTouchX, mInitialTouchY, h)) {
                    mQsTracking = true;
                    onQsExpansionStarted();
                    notifyExpandingFinished();
                    mInitialHeightOnTouch = mQsExpansionHeight;
                    mInitialTouchY = y;
                    mInitialTouchX = x;
                    mIntercepting = false;
                    mNotificationStackScroller.cancelLongPress();
                    return true;
                }
                break;
 
            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP:
                trackMovement(event);
                if (mQsTracking) {
                    flingQsWithCurrentVelocity(y,
                            event.getActionMasked() == MotionEvent.ACTION_CANCEL);
                    mQsTracking = false;
                }
                mIntercepting = false;
                break;
        }
        return false;
    }
 
/**
     * @return Whether we should intercept a gesture to open Quick Settings.
     */
    private boolean shouldQuickSettingsIntercept(float x, float y, float yDiff) {
        if (!mQsExpansionEnabled || mCollapsedOnDown) {
            return false;
        }
        View header = mKeyguardShowing || mQs == null ? mKeyguardStatusBar : mQs.getHeader();
        final boolean onHeader = x >= mQsFrame.getX()
                && x <= mQsFrame.getX() + mQsFrame.getWidth()
                && y >= header.getTop() && y <= header.getBottom();
        if (mQsExpanded) {
            return onHeader || (yDiff < 0 && isInQsArea(x, y));
        } else {
            return onHeader;
        }
    }
 
 
 
 
 
```

从上述滑动事件可以看出 在下拉的时候 MotionEvent.ACTION_MOVE 的事件 会在  
if (Math.abs(h) > mTouchSlop && Math.abs(h) > Math.abs(x - mInitialTouchX)  
&& shouldQuickSettingsIntercept(mInitialTouchX, mInitialTouchY, h))  
// 在此处拦截 QsDetail 界面上进行上下滚动 touch 事件

所以要去掉拦截就需要将这部分代码设置为 false

修改如下：

```
 
 case MotionEvent.ACTION_MOVE:
                if (Math.abs(h) > mTouchSlop && Math.abs(h) > Math.abs(x - mInitialTouchX)
               -         && shouldQuickSettingsIntercept(mInitialTouchX, mInitialTouchY, h)) {
               +        && false/*shouldQuickSettingsIntercept(mInitialTouchX, mInitialTouchY, h)*/) {
                    mQsTracking = true;
                    onQsExpansionStarted();
                    notifyExpandingFinished();
                    mInitialHeightOnTouch = mQsExpansionHeight;
                    mInitialTouchY = y;
                    mInitialTouchX = x;
                    mIntercepting = false;
                    mNotificationStackScroller.cancelLongPress();
                    return true;
                }
                break;
```

### 3.2 在 onTouch 事件的处理

```
private boolean handleQsTouch(MotionEvent event) {
    final int action = event.getActionMasked();
    if (action == MotionEvent.ACTION_DOWN && getExpandedFraction() == 1f
            && mBarState != StatusBarState.KEYGUARD && !mQsExpanded
            && mQsExpansionEnabled) {
 
        // Down in the empty area while fully expanded - go to QS.
        mQsTracking = true;
        mConflictingQsExpansionGesture = true;
        onQsExpansionStarted();
        mInitialHeightOnTouch = mQsExpansionHeight;
        mInitialTouchY = event.getX();
        mInitialTouchX = event.getY();
    }
    if (!isFullyCollapsed()) {
        handleQsDown(event);
    }
    if (!mQsExpandImmediate && mQsTracking) {
        onQsTouch(event);
        if (!mConflictingQsExpansionGesture) {
            return true;
        }
    }
    if (action == MotionEvent.ACTION_CANCEL || action == MotionEvent.ACTION_UP) {
        mConflictingQsExpansionGesture = false;
    }
    if (action == MotionEvent.ACTION_DOWN && isFullyCollapsed()
            && mQsExpansionEnabled) {
        mTwoFingerQsExpandPossible = true;
    }
    if (mTwoFingerQsExpandPossible && isOpenQsEvent(event)
            && event.getY(event.getActionIndex()) < mStatusBarMinHeight) {
        MetricsLogger.count(mContext, COUNTER_PANEL_OPEN_QS, 1);
        mQsExpandImmediate = true;
        mNotificationStackScroller.setShouldShowShelfOnly(true);
        requestPanelHeightUpdate();
 
        // Normally, we start listening when the panel is expanded, but here we need to start
        // earlier so the state is already up to date when dragging down.
        setListening(true);
    }
    return false;
}
 
private boolean isOpenQsEvent(MotionEvent event) {
    final int pointerCount = event.getPointerCount();
    final int action = event.getActionMasked();
 
    final boolean twoFingerDrag = action == MotionEvent.ACTION_POINTER_DOWN
            && pointerCount == 2;
 
    final boolean stylusButtonClickDrag = action == MotionEvent.ACTION_DOWN
            && (event.isButtonPressed(MotionEvent.BUTTON_STYLUS_PRIMARY)
            || event.isButtonPressed(MotionEvent.BUTTON_STYLUS_SECONDARY));
 
    final boolean mouseButtonClickDrag = action == MotionEvent.ACTION_DOWN
            && (event.isButtonPressed(MotionEvent.BUTTON_SECONDARY)
            || event.isButtonPressed(MotionEvent.BUTTON_TERTIARY));
  return twoFingerDrag || stylusButtonClickDrag || mouseButtonClickDrag;
}
 
 
 
 
  
```

在 OnTouchEvent(); 中要调用 handleQsTouch(MotionEvent event) 事件 而 handleQsTouch(MotionEvent event) 要调用  
isOpenQsEvent(MotionEvent event) 而 isOpenQsEvent(MotionEvent event) 来返回是否展开下拉快捷列表

因为这里要默认展开 所以修改为 true

修改如下:

```
  private boolean isOpenQsEvent(MotionEvent event) {
        final int pointerCount = event.getPointerCount();
        final int action = event.getActionMasked();
 
        final boolean twoFingerDrag = action == MotionEvent.ACTION_POINTER_DOWN
                && pointerCount == 2;
 
        final boolean stylusButtonClickDrag = action == MotionEvent.ACTION_DOWN
                && (event.isButtonPressed(MotionEvent.BUTTON_STYLUS_PRIMARY)
                || event.isButtonPressed(MotionEvent.BUTTON_STYLUS_SECONDARY));
 
        final boolean mouseButtonClickDrag = action == MotionEvent.ACTION_DOWN
                && (event.isButtonPressed(MotionEvent.BUTTON_SECONDARY)
                || event.isButtonPressed(MotionEvent.BUTTON_TERTIARY));
 
 -       return twoFingerDrag || stylusButtonClickDrag || mouseButtonClickDrag;
 +       return true;
    }
```

### 3.3 关于 StatusBar.java 中关于点击返回键 收起下拉框的处理

```
public boolean onBackPressed() {
        boolean isScrimmedBouncer = mScrimController.getState() == ScrimState.BOUNCER_SCRIMMED;
        if (mStatusBarKeyguardViewManager.onBackPressed(isScrimmedBouncer /* hideImmediately */)) {
            if (!isScrimmedBouncer) {
                mNotificationPanel.expandWithoutQs();
            }
            return true;
        }
        if (mNotificationPanel.isQsExpanded()) {
            if (mNotificationPanel.isQsDetailShowing()) {
                mNotificationPanel.closeQsDetail();
            } else {
			mNotificationPanel.animateCloseQs(false /* animateAway */);
            }
            return true;
        }
        if (mState != StatusBarState.KEYGUARD && mState != StatusBarState.SHADE_LOCKED) {
            if (mNotificationPanel.canPanelBeCollapsed()) {
                animateCollapsePanels();
            } else {
                mBubbleController.performBackPressIfNeeded();
            }
            return true;
        }
        if (mKeyguardUserSwitcher != null && mKeyguardUserSwitcher.hideIfNotSimple(true)) {
            return true;
        }
        return false;
    }
 
在返回事件处理 在onBackPressed() 中
 
 if (mNotificationPanel.isQsExpanded()) {
        if (mNotificationPanel.isQsDetailShowing()) {
            mNotificationPanel.closeQsDetail();
        } else {
            mNotificationPanel.animateCloseQs(false /* animateAway */);
        }
        return true;
    }
 
是关于下拉框的通知栏的处理
当下拉展开时 通知框默认是关闭的
所以要把mNotificationPanel.animateCloseQs(false /* animateAway */); 修改成animateCollapsePanels();
把通知框收起来
 
修改如下:
public boolean onBackPressed() {
        boolean isScrimmedBouncer = mScrimController.getState() == ScrimState.BOUNCER_SCRIMMED;
        if (mStatusBarKeyguardViewManager.onBackPressed(isScrimmedBouncer /* hideImmediately */)) {
            if (!isScrimmedBouncer) {
                mNotificationPanel.expandWithoutQs();
            }
            return true;
        }
        if (mNotificationPanel.isQsExpanded()) {
            if (mNotificationPanel.isQsDetailShowing()) {
                mNotificationPanel.closeQsDetail();
            } else {
                    -  mNotificationPanel.animateCloseQs(false /* animateAway */);
		    +	animateCollapsePanels();
            }
            return true;
        }
        if (mState != StatusBarState.KEYGUARD && mState != StatusBarState.SHADE_LOCKED) {
            if (mNotificationPanel.canPanelBeCollapsed()) {
                animateCollapsePanels();
            } else {
                mBubbleController.performBackPressIfNeeded();
            }
            return true;
        }
        if (mKeyguardUserSwitcher != null && mKeyguardUserSwitcher.hideIfNotSimple(true)) {
            return true;
        }
        return false;
    }
```
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124754530)

在定制 11.0 12.0 的 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 的下拉通知 UI 后，对整个下拉状态栏做了调整和修改，这就涉及到要全部展开  
QSPanel 画面，但是展开以后就相当于第二次下拉状态栏的界面效果，这时系统会默认收缩  
通知栏，不会全部展示通知栏，这个时候 就要实现一个功能 就是改收缩通知栏为默认展开通知栏  
关于这部分功能都是在 NotificationPanelView.java 中实现的，接下来看怎么做到收缩通知栏的

路径：frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NotificationPanelViewController.java

在 NotificationPanelViewController.java 中，会对下拉 QsPanel 判断是否展开状态的判断的  
首次下拉默认是不展开的 第二次下拉状态栏的时候 就会确定是展开的  
private boolean mQsFullyExpanded; 就是用来判断 QsPanel 是否是展开的  
当为 true 时就代表现在已经是展开的状态了

```
private void setQsExpansion(float height) {
    height = Math.min(Math.max(height, mQsMinExpansionHeight), mQsMaxExpansionHeight);
    mQsFullyExpanded = height == mQsMaxExpansionHeight && mQsMaxExpansionHeight != 0;
    if (height > mQsMinExpansionHeight && !mQsExpanded && !mStackScrollerOverscrolling
            && !mDozing) {
        setQsExpanded(true);
    } else if (height <= mQsMinExpansionHeight && mQsExpanded) {
        setQsExpanded(false);
    }
    mQsExpansionHeight = height;
    updateQsExpansion();
    requestScrollerTopPaddingUpdate(false /* animate */);
    updateHeaderKeyguardAlpha();
    if (mBarState == StatusBarState.SHADE_LOCKED
            || mBarState == StatusBarState.KEYGUARD) {
        updateKeyguardBottomAreaAlpha();
        updateBigClockAlpha();
    }
    if (mBarState == StatusBarState.SHADE && mQsExpanded
            && !mStackScrollerOverscrolling && mQsScrimEnabled) {
        mQsNavbarScrim.setAlpha(getQsExpansionFraction());
    }

    if (mAccessibilityManager.isEnabled()) {
        setAccessibilityPaneTitle(determineAccessibilityPaneTitle());
    }

    if (!mFalsingManager.isUnlockingDisabled() && mQsFullyExpanded
            && mFalsingManager.shouldEnforceBouncer()) {
        mStatusBar.executeRunnableDismissingKeyguard(null, null /* cancelAction */,
                false /* dismissShade */, true /* afterKeyguardGone */, false /* deferred */);
    }
    for (int i = 0; i < mExpansionListeners.size(); i++) {
        mExpansionListeners.get(i).onQsExpansionChanged(mQsMaxExpansionHeight != 0
                ? mQsExpansionHeight / mQsMaxExpansionHeight : 0);
    }
    if (DEBUG) {
        invalidate();
    }
}

```

在手势滑动的时候 会判断 mQsFullyExpanded = height == mQsMaxExpansionHeight && mQsMaxExpansionHeight != 0; QSPanel 的高度是最大高度后 就认为是展开的状态了

而在

```
@Override
protected void onTrackingStarted() {
    mFalsingManager.onTrackingStarted(mStatusBar.isKeyguardCurrentlySecure());
    super.onTrackingStarted();
    if (mQsFullyExpanded) {
        mQsExpandImmediate = true;
        mNotificationStackScroller.setShouldShowShelfOnly(true);
    }
    if (mBarState == StatusBarState.KEYGUARD
            || mBarState == StatusBarState.SHADE_LOCKED) {
        mAffordanceHelper.animateHideLeftRightIcon();
    }
    mNotificationStackScroller.onPanelTrackingStarted();
}

```

onTrackingStarted() 中会根据这个参数设置 mNotificationStackScroller.setShouldShowShelfOnly(true);  
就是通知栏是否收缩

setShouldShowShelfOnly 就是设置通知栏是否收缩的  
所以具体修改如下：

```
@Override
    protected void onTrackingStarted() {
        mFalsingManager.onTrackingStarted(mStatusBar.isKeyguardCurrentlySecure());
        super.onTrackingStarted();
        if (mQsFullyExpanded) {
            mQsExpandImmediate = true;
           -  mNotificationStackScroller.setShouldShowShelfOnly(true);
           + mNotificationStackScroller.setShouldShowShelfOnly(false);
        }
        if (mBarState == StatusBarState.KEYGUARD
                || mBarState == StatusBarState.SHADE_LOCKED) {
            mAffordanceHelper.animateHideLeftRightIcon();
        }
        mNotificationStackScroller.onPanelTrackingStarted();
    }

```

mNotificationStackScroller.setShouldShowShelfOnly(false) 参数改为 fasle 这样就不会收缩了 默认就展开了就这样就可以了
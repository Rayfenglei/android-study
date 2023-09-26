> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124792278)

### 1. 概述

在 11.0 的产品定制化开发中，在系统原生的 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 状态栏下拉和通知栏，默认是根据手势的 x 坐标的位置居中显示，但是如果太靠两边感觉不太好，下拉太靠边不太好看所以产品提出不管手势在哪里下滑 都要去下拉和通知栏居中显示 会比较好看些 下面就来实现这个需求

### 2.SystemUI 状态栏下拉和通知栏始终居中的核心类

```
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NotificationPanelView.java
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NotificationPanelViewController.java


```

### 3.SystemUI 状态栏下拉和通知栏始终居中的核心功能实现和分析

而在 SystemUI 中处理下拉展示 UI 的就是 NotificationPanelView.java  
接下来就来看 NotificationPanelViewController.java 的关于 onTouchEvent 源码 分析原因  
下拉触摸是在 onTouchEvent(MotionEvent event) 中处理的

### 3.1NotificationPanelView.java 相关功能分析

```
      @Inject
      public NotificationPanelViewController(NotificationPanelView view,
              InjectionInflationController injectionInflationController,
              NotificationWakeUpCoordinator coordinator, PulseExpansionHandler pulseExpansionHandler,
              DynamicPrivacyController dynamicPrivacyController,
              KeyguardBypassController bypassController, FalsingManager falsingManager,
              ShadeController shadeController,
              NotificationLockscreenUserManager notificationLockscreenUserManager,
              NotificationEntryManager notificationEntryManager,
              KeyguardStateController keyguardStateController,
              StatusBarStateController statusBarStateController, DozeLog dozeLog,
              DozeParameters dozeParameters, CommandQueue commandQueue, VibratorHelper vibratorHelper,
              LatencyTracker latencyTracker, PowerManager powerManager,
              AccessibilityManager accessibilityManager, @DisplayId int displayId,
              KeyguardUpdateMonitor keyguardUpdateMonitor, MetricsLogger metricsLogger,
              ActivityManager activityManager, ZenModeController zenModeController,
              ConfigurationController configurationController,
              FlingAnimationUtils.Builder flingAnimationUtilsBuilder,
              StatusBarTouchableRegionManager statusBarTouchableRegionManager,
              ConversationNotificationManager conversationNotificationManager,
              MediaHierarchyManager mediaHierarchyManager,
              BiometricUnlockController biometricUnlockController,
              StatusBarKeyguardViewManager statusBarKeyguardViewManager) {
          super(view, falsingManager, dozeLog, keyguardStateController,
                  (SysuiStatusBarStateController) statusBarStateController, vibratorHelper,
                  latencyTracker, flingAnimationUtilsBuilder, statusBarTouchableRegionManager);
          mView = view;
          mMetricsLogger = metricsLogger;
          mActivityManager = activityManager;
          mZenModeController = zenModeController;
          mConfigurationController = configurationController;
          mFlingAnimationUtilsBuilder = flingAnimationUtilsBuilder;
          mMediaHierarchyManager = mediaHierarchyManager;
          mStatusBarKeyguardViewManager = statusBarKeyguardViewManager;
          mView.setWillNotDraw(!DEBUG);
          mInjectionInflationController = injectionInflationController;
          mFalsingManager = falsingManager;
          mPowerManager = powerManager;
          mWakeUpCoordinator = coordinator;
          mAccessibilityManager = accessibilityManager;
          mView.setAccessibilityPaneTitle(determineAccessibilityPaneTitle());
          setPanelAlpha(255, false /* animate */);
          mCommandQueue = commandQueue;
          mDisplayId = displayId;
          mPulseExpansionHandler = pulseExpansionHandler;
          mDozeParameters = dozeParameters;
          mBiometricUnlockController = biometricUnlockController;
          pulseExpansionHandler.setPulseExpandAbortListener(() -> {
              if (mQs != null) {
                  mQs.animateHeaderSlidingOut();
              }
          });
          mThemeResId = mView.getContext().getThemeResId();
          mKeyguardBypassController = bypassController;
          mUpdateMonitor = keyguardUpdateMonitor;
          mFirstBypassAttempt = mKeyguardBypassController.getBypassEnabled();
          KeyguardStateController.Callback
                  keyguardMonitorCallback =
                  new KeyguardStateController.Callback() {
                      @Override
                      public void onKeyguardFadingAwayChanged() {
                          if (!mKeyguardStateController.isKeyguardFadingAway()) {
                              mFirstBypassAttempt = false;
                              mDelayShowingKeyguardStatusBar = false;
                          }
                      }
                  };
          mKeyguardStateController.addCallback(keyguardMonitorCallback);
          DynamicPrivacyControlListener
                  dynamicPrivacyControlListener =
                  new DynamicPrivacyControlListener();
          dynamicPrivacyController.addListener(dynamicPrivacyControlListener);
  
          mBottomAreaShadeAlphaAnimator = ValueAnimator.ofFloat(1f, 0);
          mBottomAreaShadeAlphaAnimator.addUpdateListener(animation -> {
              mBottomAreaShadeAlpha = (float) animation.getAnimatedValue();
              updateKeyguardBottomAreaAlpha();
          });
          mBottomAreaShadeAlphaAnimator.setDuration(160);
          mBottomAreaShadeAlphaAnimator.setInterpolator(Interpolators.ALPHA_OUT);
          mShadeController = shadeController;
          mLockscreenUserManager = notificationLockscreenUserManager;
          mEntryManager = notificationEntryManager;
          mConversationNotificationManager = conversationNotificationManager;
  
          mView.setBackgroundColor(Color.TRANSPARENT);
          OnAttachStateChangeListener onAttachStateChangeListener = new OnAttachStateChangeListener();
          mView.addOnAttachStateChangeListener(onAttachStateChangeListener);
          if (mView.isAttachedToWindow()) {
              onAttachStateChangeListener.onViewAttachedToWindow(mView);
          }
  
          mView.setOnApplyWindowInsetsListener(new OnApplyWindowInsetsListener());
  
          if (DEBUG) {
              mView.getOverlay().add(new DebugDrawable());
          }
  
          onFinishInflate();
      }
      
 @Override
    public boolean onTouchEvent(MotionEvent event) {
        /* UNISOC: Modify for bug1173815,1163448,1191373 {@ */
        if (DEBUG) {
            Log.d("NotificationPanelView","onTouchEvent mBlockTouches=" + mBlockTouches);
        }

        if (mBlockTouches || (mQs != null && mQs.isCustomizing())) {
            if (DEBUG && mQs != null && mQs.isCustomizing()) {
                Log.d("NotificationPanelView","onTouchEvent false");
            }
            return false;
        }

        // Do not allow panel expansion if bouncer is scrimmed, otherwise user would be able to
        // pull down QS or expand the shade.
        if (mStatusBar.isBouncerShowingScrimmed()) {
            if (DEBUG) {
                Log.d("NotificationPanelView","onTouchEvent bouncer false ");
            }
            return false;
        }

        initDownStates(event);
        // Make sure the next touch won't the blocked after the current ends.
        if (event.getAction() == MotionEvent.ACTION_UP
                || event.getAction() == MotionEvent.ACTION_CANCEL) {
            mBlockingExpansionForCurrentTouch = false;
        }
        if (!mIsExpanding && mPulseExpansionHandler.onTouchEvent(event)) {
            if (DEBUG) {
                Log.d("NotificationPanelView","onTouchEvent handler true");
            }
            // We're expanding all the other ones shouldn't get this anymore
            return true;
        }
        if (mListenForHeadsUp && !mHeadsUpTouchHelper.isTrackingHeadsUp()
                && mHeadsUpTouchHelper.onInterceptTouchEvent(event)) {
            mIsExpansionFromHeadsUp = true;
            MetricsLogger.count(mContext, COUNTER_PANEL_OPEN_PEEK, 1);
        }
        boolean handled = false;
        if ((!mIsExpanding || mHintAnimationRunning)
                && !mQsExpanded
                && mBarState != StatusBarState.SHADE
                && !mDozing) {
            handled |= mAffordanceHelper.onTouchEvent(event);
        }
        if (mOnlyAffordanceInThisMotion) {
            if (DEBUG) {
                Log.d("NotificationPanelView","onTouchEvent motion true");
            }
            return true;
        }
        handled |= mHeadsUpTouchHelper.onTouchEvent(event);

        if (!mHeadsUpTouchHelper.isTrackingHeadsUp() && handleQsTouch(event)) {
            if (DEBUG) {
                Log.d("NotificationPanelView","onTouchEvent qstouch true");
            }
            return true;
        }
        if (event.getActionMasked() == MotionEvent.ACTION_DOWN && isFullyCollapsed()) {
            MetricsLogger.count(mContext, COUNTER_PANEL_OPEN, 1);
            updateVerticalPanelPosition(event.getX());
            handled = true;
        }
        handled |= super.onTouchEvent(event);
        if (DEBUG) {
            Log.d("NotificationPanelView","end mDozing:" + mDozing + " mPulsing:" + mPulsing + " handled:" +handled);
        }
        /* @} */
        return !mDozing || mPulsing || handled;
    }

```

从构造方法中可以看出 NotificationPanelViewController 负责对 NotificationPanelView 下拉通知栏的布局管理  
而在下拉时在 updateVerticalPanelPosition(event.getX()); 来处理偏移量从而显示状态栏布局的位置

```
protected void updateVerticalPanelPosition(float x) {
    if (mNotificationStackScroller.getWidth() * 1.75f > getWidth()) {
        resetHorizontalPanelPosition();
        return;
    }
    float leftMost = mPositionMinSideMargin + mNotificationStackScroller.getWidth() / 2;
    float rightMost = getWidth() - mPositionMinSideMargin
            - mNotificationStackScroller.getWidth() / 2;
    if (Math.abs(x - getWidth() / 2) < mNotificationStackScroller.getWidth() / 4) {
        x = getWidth() / 2;
    }
    x = Math.min(rightMost, Math.max(leftMost, x));
    float center =
            mNotificationStackScroller.getLeft() + mNotificationStackScroller.getWidth() / 2;
    setHorizontalPanelTranslation(x - center);
}

  

setHorizontalPanelTranslation(x - center); 这里设置下拉和通知栏偏移多少

protected void setHorizontalPanelTranslation(float translation) {
    mNotificationStackScroller.setHorizontalPanelTranslation(translation);
    mQsFrame.setTranslationX(translation);
    int size = mVerticalTranslationListener.size();
    for (int i = 0; i < size; i++) {
        mVerticalTranslationListener.get(i).run();
    }
}

   

而

  mNotificationStackScroller.setHorizontalPanelTranslation(translation);
        mQsFrame.setTranslationX(translation);

```

通过上述方法  
来具体计算通知栏和下拉框具体偏移多少  
所以如果不想偏移 就注释掉这两行代码修改如下:

```
protected void setHorizontalPanelTranslation(float translation) {
        //mNotificationStackScroller.setHorizontalPanelTranslation(translation);
        //mQsFrame.setTranslationX(translation);
        int size = mVerticalTranslationListener.size();
        for (int i = 0; i < size; i++) {
            mVerticalTranslationListener.get(i).run();
        }
    }

```
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/130628505)

1. 概述
-----

  在 11.0 系统 rom 定制化开发中，在系统中默认手势中有三键导航和系统手势导航，在设置默认系统手势导航以后，左右滑动手势返回功能  
是在 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 中具体实现的，现在有需要要求控制左右滑动手势返回功能的启用和禁用，所以要分析手势返回功能的具体实现流程

2.SystemUI 控制系统手势左右滑返回功能核心代码
----------------------------

```
      frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarView.java
      frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/EdgeBackGestureHandler.java
```

3.SystemUI 控制系统手势左右滑返回功能分析  
  3.1 NavigationBarView.java 关于系统手势的功能分析
---------------------------------------------------------------------

    NavigationBarView 在构造的时候通过创建 EdgeBackGestureHandler 实例，其是整个返回手势的核心管理类。

```
          public NavigationBarView(Context context, AttributeSet attrs) {
            super(context, attrs);
            mIsVertical = false;
            mLongClickableAccessibilityButton = false;
            mNavBarMode = Dependency.get(NavigationModeController.class).addListener(this);
            /* UNISCO: Bug 1072090,1116092 new feature of dynamic navigationbar @{*/
            mSupportDynamicBar = isSupportDynamicNavBar(mContext, mNavBarMode);
            /* }@ */
            boolean isGesturalMode = isGesturalMode(mNavBarMode);
....
     
            mButtonDispatchers.put(R.id.back, backButton);
            mButtonDispatchers.put(R.id.home, new ButtonDispatcher(R.id.home));
            mButtonDispatchers.put(R.id.home_handle, new ButtonDispatcher(R.id.home_handle));
            mButtonDispatchers.put(R.id.recent_apps, new ButtonDispatcher(R.id.recent_apps));
            mButtonDispatchers.put(R.id.ime_switcher, imeSwitcherButton);
            mButtonDispatchers.put(R.id.accessibility_button, accessibilityButton);
            mButtonDispatchers.put(R.id.rotate_suggestion, rotateSuggestionButton);
            mButtonDispatchers.put(R.id.menu_container, mContextualButtonGroup);
....
            mEdgeBackGestureHandler = new EdgeBackGestureHandler(context, mOverviewProxyService);
 
        }
```

从上述的代码中可以看出，在 NavigationBarView.java 中的构造方法 NavigationBarView(Context context, AttributeSet [attrs](https://so.csdn.net/so/search?q=attrs&spm=1001.2101.3001.7020))  
 在初始化 NavigationBarView.java 的相关方法中，在加载相关页面时，初次添加到 Window 上的时候会调用 EdgeBackGestureHandler 开始工作  
来实现手势返回的相关功能初始化

```
       @Override
        protected void onAttachedToWindow() {
            super.onAttachedToWindow();
            requestApplyInsets();
            reorient();
            onNavigationModeChanged(mNavBarMode);
            setUpSwipeUpOnboarding(isQuickStepSwipeUpEnabled());
            if (mRotationButtonController != null) {
                mRotationButtonController.registerListeners();
            }
     
            mEdgeBackGestureHandler.onNavBarAttached();
            getViewTreeObserver().addOnComputeInternalInsetsListener(mOnComputeInternalInsetsListener);
        }
```

3.2 EdgeBackGestureHandler 关于手势的相关功能
------------------------------------

```
      public EdgeBackGestureHandler(Context context, OverviewProxyService overviewProxyService) {
            final Resources res = context.getResources();
            mContext = context;
            mDisplayId = context.getDisplayId();
            mMainExecutor = context.getMainExecutor();
            mWm = context.getSystemService(WindowManager.class);
            mOverviewProxyService = overviewProxyService;
     
            // Reduce the default touch slop to ensure that we can intercept the gesture
            // before the app starts to react to it.
            // TODO(b/130352502) Tune this value and extract into a constant
            mTouchSlop = ViewConfiguration.get(context).getScaledTouchSlop() * 0.75f;
            mLongPressTimeout = Math.min(MAX_LONG_PRESS_TIMEOUT,
                    ViewConfiguration.getLongPressTimeout());
     
            mNavBarHeight = res.getDimensionPixelSize(R.dimen.navigation_bar_frame_height);
            mMinArrowPosition = res.getDimensionPixelSize(R.dimen.navigation_edge_arrow_min_y);
            mFingerOffset = res.getDimensionPixelSize(R.dimen.navigation_edge_finger_offset);
            updateCurrentUserResources(res);
        }
```

在上述的方法中，可以看出在 EdgeBackGestureHandler.java 中的相关方法中，在 EdgeBackGestureHandler 类在  
EdgeBackGestureHandler(Context context, OverviewProxyService overviewProxyService),  
构造方法的时候初始化一些手势判断需要的参数和变量。

    // 判断手势是否可用

```
    private void updateIsEnabled() {
            boolean isEnabled = mIsAttached && mIsGesturalModeEnabled;
            if (isEnabled == mIsEnabled) {
                return;
            }
            mIsEnabled = isEnabled;
    // 如果无效的话结束监听 Input
            disposeInputChannel();
     
            if (mEdgePanel != null) {
                mWm.removeView(mEdgePanel);
                mEdgePanel = null;
                mRegionSamplingHelper.stop();
                mRegionSamplingHelper = null;
            }
     
            if (!mIsEnabled) {
               // 注销监听返回手势参数的设置变化
                WindowManagerWrapper.getInstance().removePinnedStackListener(mImeChangedListener);
                mContext.getSystemService(DisplayManager.class).unregisterDisplayListener(this);
     
                try {
                   // 注销 WMS 里保存的除外区域监听
                    WindowManagerGlobal.getWindowManagerService()
                            .unregisterSystemGestureExclusionListener(
                                    mGestureExclusionListener, mDisplayId);
                } catch (RemoteException e) {
                    Log.e(TAG, "Failed to unregister window manager callbacks", e);
                }
     
            } else {
                updateDisplaySize();
      // 监听返回手势参数的设置变化
                mContext.getSystemService(DisplayManager.class).registerDisplayListener(this,
                        mContext.getMainThreadHandler());
     
                try {
    // 监听 WMS 里保存的除外区域
                    WindowManagerWrapper.getInstance().addPinnedStackListener(mImeChangedListener);
                    WindowManagerGlobal.getWindowManagerService()
                            .registerSystemGestureExclusionListener(
                                    mGestureExclusionListener, mDisplayId);
                } catch (RemoteException e) {
                    Log.e(TAG, "Failed to register window manager callbacks", e);
                }
     
                // Register input event receiver
                mInputMonitor = InputManager.getInstance().monitorGestureInput(
                        "edge-swipe", mDisplayId);
                mInputEventReceiver = new SysUiInputEventReceiver(
                        mInputMonitor.getInputChannel(), Looper.getMainLooper());
     
                // Add a nav bar panel window
                 // 添加 NavigationBarEdgePanel 为 Edge Back 事件的处理实现
                mEdgePanel = new NavigationBarEdgePanel(mContext);
                mEdgePanelLp = new WindowManager.LayoutParams(
                        mContext.getResources()
                                .getDimensionPixelSize(R.dimen.navigation_edge_panel_width),
                        mContext.getResources()
                                .getDimensionPixelSize(R.dimen.navigation_edge_panel_height),
                        WindowManager.LayoutParams.TYPE_NAVIGATION_BAR_PANEL,
                        WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                                | WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL
                                | WindowManager.LayoutParams.FLAG_SPLIT_TOUCH
                                | WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN,
                        PixelFormat.TRANSLUCENT);
                mEdgePanelLp.privateFlags |=
                        WindowManager.LayoutParams.PRIVATE_FLAG_SHOW_FOR_ALL_USERS;
                mEdgePanelLp.setTitle(TAG + mDisplayId);
                mEdgePanelLp.accessibilityTitle = mContext.getString(R.string.nav_bar_edge_panel);
                mEdgePanelLp.windowAnimations = 0;
                mEdgePanel.setLayoutParams(mEdgePanelLp);
                mWm.addView(mEdgePanel, mEdgePanelLp);
                mRegionSamplingHelper = new RegionSamplingHelper(mEdgePanel,
                        new RegionSamplingHelper.SamplingCallback() {
                            @Override
                            public void onRegionDarknessChanged(boolean isRegionDark) {
                                mEdgePanel.setIsDark(!isRegionDark, true /* animate */);
                            }
     
                            @Override
                            public Rect getSampledRegion(View sampledView) {
                                return mSamplingRect;
                            }
                        });
            }
        }
     
       // 监听左右手势滑动事件 然后构造返回UI 调用返回事件 通知app响应返回事件
        private void onMotionEvent(MotionEvent ev) {
              // add core start  当is_enable为0 表示禁用左右滑动返回功能 默认为1是启用手势返回功能
            int enable = Settings.Global.getInt(mContext.getContentResolver(),"is_enable",1);
            if(enable ==0)return;
             // add core end
     
     
            int action = ev.getActionMasked();
            if (action == MotionEvent.ACTION_DOWN) {
                // Verify if this is in within the touch region and we aren't in immersive mode, and
                // either the bouncer is showing or the notification panel is hidden
                int stateFlags = mOverviewProxyService.getSystemUiStateFlags();
                mIsOnLeftEdge = ev.getX() <= mEdgeWidth + mLeftInset;
                mAllowGesture = !QuickStepContract.isBackGestureDisabled(stateFlags)
                        && isWithinTouchRegion((int) ev.getX(), (int) ev.getY());
                if (mAllowGesture) {
                    mEdgePanelLp.gravity = mIsOnLeftEdge
                            ? (Gravity.LEFT | Gravity.TOP)
                            : (Gravity.RIGHT | Gravity.TOP);
                    mEdgePanel.setIsLeftPanel(mIsOnLeftEdge);
                    mEdgePanel.handleTouch(ev);
                    updateEdgePanelPosition(ev.getY());
                    mWm.updateViewLayout(mEdgePanel, mEdgePanelLp);
                    mRegionSamplingHelper.start(mSamplingRect);
                    mDownPoint.set(ev.getX(), ev.getY());
                    mThresholdCrossed = false;
                }
            } else if (mAllowGesture) {
                if (!mThresholdCrossed) {
                    if (action == MotionEvent.ACTION_POINTER_DOWN) {
                        // We do not support multi touch for back gesture
                        cancelGesture(ev);
                        return;
                    } else if (action == MotionEvent.ACTION_MOVE) {
                        if ((ev.getEventTime() - ev.getDownTime()) > mLongPressTimeout) {
                            cancelGesture(ev);
                            return;
                        }
                        float dx = Math.abs(ev.getX() - mDownPoint.x);
                        float dy = Math.abs(ev.getY() - mDownPoint.y);
                        if (dy > dx && dy > mTouchSlop) {
                            cancelGesture(ev);
                            return;
                        } else if (dx > dy && dx > mTouchSlop) {
                            mThresholdCrossed = true;
                            // Capture inputs
                            mInputMonitor.pilferPointers();
                        }
                    }
                }
                // forward touch
                mEdgePanel.handleTouch(ev);
                boolean isUp = action == MotionEvent.ACTION_UP;
                if (isUp) {
                    boolean performAction = mEdgePanel.shouldTriggerBack();
                    if (performAction) {
                        // Perform back
                        sendEvent(KeyEvent.ACTION_DOWN, KeyEvent.KEYCODE_BACK);
                        sendEvent(KeyEvent.ACTION_UP, KeyEvent.KEYCODE_BACK);
                    }
                    mOverviewProxyService.notifyBackAction(performAction, (int) mDownPoint.x,
                            (int) mDownPoint.y, false /* isButton */, !mIsOnLeftEdge);
                }
                if (isUp || action == MotionEvent.ACTION_CANCEL) {
                    mRegionSamplingHelper.stop();
                } else {
                    updateSamplingRect();
                    mRegionSamplingHelper.updateSamplingRect();
                }
            }
        }
```

在上述的 EdgeBackGestureHandler.java 的相关方法中，可以看出，在 updateIsEnabled() 的  
主要功能，就是判断当前的手势是否可以使用，而在 onMotionEvent(MotionEvent ev) 中，  
主要是处理当前手势滑动的相关功能，所有的手势返回都是在这里处理的，所有可以在这里  
添加 Settings.Global.getInt(mContext.getContentResolver(),"is_enable",1); 用系统属性来  
判断当前是否需要启用手势功能，来实现对手势的开启和禁用
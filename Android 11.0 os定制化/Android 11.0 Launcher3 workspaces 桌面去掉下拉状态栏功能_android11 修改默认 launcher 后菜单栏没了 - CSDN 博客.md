> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127954352)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.Launcher3 workspaces 桌面去掉下拉状态栏功能的核心类](#t1)

[3.Launcher3 workspaces 桌面去掉下拉状态栏功能的核心功能实现和分析](#t2)

 [3.1 首选分析下 StatusBarTouchController.java 关于下拉状态栏的相关 api](#t3)

[3.2 关于 QuickstepLauncher.java 下拉状态栏事件的相关 api 分析](#t4)

1. 概述
-----

  在 11.0 的系统产品开发中，在 Launcher3 的 workspace 每一屏通过下拉都可以下滑出状态栏，而  
在禁用下拉状态栏的产品中，这样的功能是不允许存在的，所以产品需求要求去掉这个功能  
这就需要分析下拉状态栏流程，然后去掉这个下拉状态栏流程就可以了

2.Launcher3 workspaces 桌面去掉下拉状态栏功能的核心类
--------------------------------------

```
packages/apps/Launcher3/quickstep/recents_ui_overrides/src/com/android/launcher3/uioverrides/QuickstepLauncher.java
packages/apps/Launcher3/quickstep/src/com/android/launcher3/uioverrides/touchcontrollers/StatusBarTouchController.java
```

3.Launcher3 workspaces 桌面去掉下拉状态栏功能的核心功能实现和分析
--------------------------------------------

   3.1 首选分析下 StatusBarTouchController.java 关于下拉状态栏的相关 api
---------------------------------------------------------

```
    private void dispatchTouchEvent(MotionEvent ev) {
        if (mSystemUiProxy.isActive()) {
            mLastAction = ev.getActionMasked();
            mSystemUiProxy.onStatusBarMotionEvent(ev);
        }
    }
 
    @Override
    public final boolean onControllerInterceptTouchEvent(MotionEvent ev) {
        int action = ev.getActionMasked();
        int idx = ev.getActionIndex();
        int pid = ev.getPointerId(idx);
        if (action == ACTION_DOWN) {
            mCanIntercept = canInterceptTouch(ev);
            if (!mCanIntercept) {
                return false;
            }
            mDownEvents.put(pid, new PointF(ev.getX(), ev.getY()));
        } else if (ev.getActionMasked() == MotionEvent.ACTION_POINTER_DOWN) {
           // Check!! should only set it only when threshold is not entered.
           mDownEvents.put(pid, new PointF(ev.getX(idx), ev.getY(idx)));
        }
        if (!mCanIntercept) {
            return false;
        }
        if (action == ACTION_MOVE) {
            float dy = ev.getY(idx) - mDownEvents.get(pid).y;
            float dx = ev.getX(idx) - mDownEvents.get(pid).x;
            // Currently input dispatcher will not do touch transfer if there are more than
            // one touch pointer. Hence, even if slope passed, only set the slippery flag
            // when there is single touch event. (context: InputDispatcher.cpp line 1445)
            if (dy > mTouchSlop && dy > Math.abs(dx) && ev.getPointerCount() == 1) {
                ev.setAction(ACTION_DOWN);
                dispatchTouchEvent(ev);
                setWindowSlippery(true);
                return true;
            }
            if (Math.abs(dx) > mTouchSlop) {
                mCanIntercept = false;
            }
        }
        return false;
    }
 
    @Override
    public final boolean onControllerTouchEvent(MotionEvent ev) {
        int action = ev.getAction();
        if (action == ACTION_UP || action == ACTION_CANCEL) {
            dispatchTouchEvent(ev);
            mLauncher.getUserEventDispatcher().logActionOnContainer(action == ACTION_UP ?
                    Touch.FLING : Touch.SWIPE, Direction.DOWN, ContainerType.WORKSPACE,
                    mLauncher.getWorkspace().getCurrentPage());
            setWindowSlippery(false);
            return true;
        }
        return true;
    }
```

通过 StatusBarTouchController.java 的 dispatchTouchEvent(MotionEvent ev) 中的方法可以看出  
在通过 mSystemUiProxy.onStatusBarMotionEvent(ev); 来响应状态栏下拉的事件，而 dispatchTouchEvent(MotionEvent ev)  
是在 onControllerTouchEvent(MotionEvent ev) 和 onControllerInterceptTouchEvent(MotionEvent ev) 中根据手势下拉来判断下滑距离然后做出响应事件的

3.2 关于 QuickstepLauncher.java 下拉状态栏事件的相关 api 分析
-----------------------------------------------

```
 public class QuickstepLauncher extends BaseQuickstepLauncher {
 
    public static final boolean GO_LOW_RAM_RECENTS_ENABLED = false;
    /**
     * Reusable command for applying the shelf height on the background thread.
     */
    public static final AsyncCommand SET_SHELF_HEIGHT = (context, arg1, arg2) ->
            SystemUiProxy.INSTANCE.get(context).setShelfHeight(arg1 != 0, arg2);
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        if (mHotseatPredictionController != null) {
            mHotseatPredictionController.createPredictor();
        }
    }
 
    @Override
    protected void setupViews() {
        super.setupViews();
        if (FeatureFlags.ENABLE_HYBRID_HOTSEAT.get()) {
            mHotseatPredictionController = new HotseatPredictionController(this);
        }
    }
 
    @Override
    protected void logAppLaunch(ItemInfo info, InstanceId instanceId) {
        StatsLogger logger = getStatsLogManager()
                .logger().withItemInfo(info).withInstanceId(instanceId);
        OptionalInt allAppsRank = PredictionUiStateManager.INSTANCE.get(this).getAllAppsRank(info);
        allAppsRank.ifPresent(logger::withRank);
        logger.log(LAUNCHER_APP_LAUNCH_TAP);
 
        if (mHotseatPredictionController != null) {
            mHotseatPredictionController.logLaunchedAppRankingInfo(info, instanceId);
        }
    }
 
    @Override
    public void onConfigurationChanged(Configuration newConfig) {
        super.onConfigurationChanged(newConfig);
        onStateOrResumeChanging(false /* inTransition */);
    }
 
    @Override
    public boolean startActivitySafely(View v, Intent intent, ItemInfo item,
            @Nullable String sourceContainer) {
        if (mHotseatPredictionController != null) {
            mHotseatPredictionController.setPauseUIUpdate(true);
        }
        return super.startActivitySafely(v, intent, item, sourceContainer);
    }
```

在 QuickstepLauncher.java 中的相关 api 中，onCreate(Bundle savedInstanceState) 中加载  
mHotseatPredictionController 关于 hotseat 的控制类，在 startActivitySafely(）是启动某个 activity 的  
方法，logAppLaunch(ItemInfo info, InstanceId instanceId) 是启动 hotseat 中点击图标的 app 的启动入口

```
@Override
    public TouchController[] createTouchControllers() {
        if (TestProtocol.sDebugTracing) {
            Log.d(TestProtocol.PAUSE_NOT_DETECTED, "createTouchControllers.1");
        }
        Mode mode = SysUINavigationMode.getMode(this);
 
        ArrayList<TouchController> list = new ArrayList<>();
        list.add(getDragController());
        if (mode == NO_BUTTON) {
            list.add(new NoButtonQuickSwitchTouchController(this));
            list.add(new NavBarToHomeTouchController(this));
            if (TestProtocol.sDebugTracing) {
                Log.d(TestProtocol.PAUSE_NOT_DETECTED, "createTouchControllers.2");
            }
            if (FeatureFlags.ENABLE_OVERVIEW_ACTIONS.get()) {
                if (TestProtocol.sDebugTracing) {
                    Log.d(TestProtocol.PAUSE_NOT_DETECTED, "createTouchControllers.3");
                }
                list.add(new NoButtonNavbarToOverviewTouchController(this));
            } else {
                list.add(new FlingAndHoldTouchController(this));
            }
        } else {
            if (getDeviceProfile().isVerticalBarLayout()) {
                list.add(new OverviewToAllAppsTouchController(this));
                list.add(new LandscapeEdgeSwipeController(this));
                if (mode.hasGestures) {
                    list.add(new TransposedQuickSwitchTouchController(this));
                }
            } else {
                list.add(new PortraitStatesTouchController(this,
                        mode.hasGestures /* allowDragToOverview */));
                if (mode.hasGestures) {
                    list.add(new QuickSwitchTouchController(this));
                }
            }
        }
 
        if (!getDeviceProfile().isMultiWindowMode) {
            //list.add(new StatusBarTouchController(this));
        }
 
        list.add(new LauncherTaskViewController(this));
        return list.toArray(new TouchController[list.size()]);
    }
```

在 QuickstepLauncher.java 中的 createTouchControllers() 是添加各种 Touch 事件的集合  
在这里通过判断这种情景下的 TouchController 类，然后添加到集合中，而 StatusBarTouchController  
就是关于状态栏的手势监听控制类，所以需要去掉这个类 就可以实现在  
workspace 各个屏中下滑就不在弹出下拉状态栏的功能了
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125922013)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 自定义仿小米全面屏手势导航返回 ui 布局的核心代码](#t1) 

[3. 自定义左右手势返回 UI 样式的核心代码功能分析](#t2)

[3.1 NavigationBarView 手势导航布局左右手势返回的相关代码](#t3)

[3.2 EdgeBackGestureHandler 手势导航布局左右手势返回的相关代码](#t4)

[3.3 NavigationBarEdgePanel 手势导航布局左右手势返回的相关代码](#t5)

[4. 自定义仿小米全面屏手势导航返回 ui 布局功能实现](#t6)

1. 概述
-----

 在定制化开发中，对 SystemUI 的 UI 效果修改定制也是常有的功能，由于发现小米全面屏手势左右滑动返回 UI 效果比较美观  
所以有需求要求仿小米全面屏手势来做左右滑动返回 UI 的效果，来自定义 UI 做手势返回的布局样式, 这样的弧线就涉及到用贝塞尔曲线来实现功能

效果图：

![](https://img-blog.csdnimg.cn/e0d213442cbf4a62a7faa77e1f77c94c.png)

2. 自定义仿小米全面屏手势导航返回 ui 布局的核心代码 
------------------------------

```
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarView.java
  frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/EdgeBackGestureHandler.java
  frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarEdgePanel.java
```

3. 自定义左右手势返回 UI 样式的核心代码功能分析
---------------------------

#### 3.1 NavigationBarView 手势导航布局左右手势返回的相关代码

```
public class NavigationBarView extends FrameLayout implements
NavigationModeController.ModeChangedListener {
final static boolean DEBUG = false;
final static String TAG = "StatusBar/NavBarView";
 
// slippery nav bar when everything is disabled, e.g. during setup
final static boolean SLIPPERY_WHEN_DISABLED = true;
 
final static boolean ALTERNATE_CAR_MODE_UI = false;
private final RegionSamplingHelper mRegionSamplingHelper;
private final int mNavColorSampleMargin;
private final SysUiState mSysUiFlagContainer;
private final PluginManager mPluginManager;
 
View mCurrentView = null;
private View mVertical;
private View mHorizontal;
 
/** Indicates that navigation bar is vertical. */
private boolean mIsVertical;
private int mCurrentRotation = -1;
 
boolean mLongClickableAccessibilityButton;
int mDisabledFlags = 0;
int mNavigationIconHints = 0;
private int mNavBarMode;
 
private final Region mTmpRegion = new Region();
private final int[] mTmpPosition = new int[2];
private Rect mTmpBounds = new Rect();
 
private KeyButtonDrawable mBackIcon;
private KeyButtonDrawable mHomeDefaultIcon;
private KeyButtonDrawable mRecentIcon;
private KeyButtonDrawable mDockedIcon;
 
private EdgeBackGestureHandler mEdgeBackGestureHandler;
private final DeadZone mDeadZone;
private boolean mDeadZoneConsuming = false;
private final NavigationBarTransitions mBarTransitions;
private final OverviewProxyService mOverviewProxyService;
private AutoHideController mAutoHideController;
 
// performs manual animation in sync with layout transitions
private final NavTransitionListener mTransitionListener = new NavTransitionListener();
 
private OnVerticalChangedListener mOnVerticalChangedListener;
private boolean mLayoutTransitionsEnabled = true;
private boolean mWakeAndUnlocking;
private boolean mUseCarModeUi = false;
private boolean mInCarMode = false;
private boolean mDockedStackExists;
private boolean mImeVisible;
private boolean mScreenOn = true;
 
private final SparseArray<ButtonDispatcher> mButtonDispatchers = new SparseArray<>();
private final ContextualButtonGroup mContextualButtonGroup;
private Configuration mConfiguration;
private Configuration mTmpLastConfiguration;
 
private NavigationBarInflaterView mNavigationInflaterView;
private RecentsOnboarding mRecentsOnboarding;
private NotificationPanelViewController mPanelView;
private FloatingRotationButton mFloatingRotationButton;
private RotationButtonController mRotationButtonController;
 
/**
* Helper that is responsible for showing the right toast when a disallowed activity operation
* occurred. In pinned mode, we show instructions on how to break out of this mode, whilst in
* fully locked mode we only show that unlocking is blocked.
*/
private ScreenPinningNotify mScreenPinningNotify;
private Rect mSamplingBounds = new Rect();
 
public NavigationBarView(Context context, AttributeSet attrs) {
super(context, attrs);
 
mIsVertical = false;
mLongClickableAccessibilityButton = false;
mNavBarMode = Dependency.get(NavigationModeController.class).addListener(this);
boolean isGesturalMode = isGesturalMode(mNavBarMode);
 
mSysUiFlagContainer = Dependency.get(SysUiState.class);
mPluginManager = Dependency.get(PluginManager.class);
// Set up the context group of buttons
mContextualButtonGroup = new ContextualButtonGroup(R.id.menu_container);
final ContextualButton imeSwitcherButton = new ContextualButton(R.id.ime_switcher,
R.drawable.ic_ime_switcher_default);
final RotationContextButton rotateSuggestionButton = new RotationContextButton(
R.id.rotate_suggestion, R.drawable.ic_sysbar_rotate_button);
final ContextualButton accessibilityButton =
new ContextualButton(R.id.accessibility_button,
R.drawable.ic_sysbar_accessibility_button);
mContextualButtonGroup.addButton(imeSwitcherButton);
if (!isGesturalMode) {
mContextualButtonGroup.addButton(rotateSuggestionButton);
}
mContextualButtonGroup.addButton(accessibilityButton);
 
mOverviewProxyService = Dependency.get(OverviewProxyService.class);
mRecentsOnboarding = new RecentsOnboarding(context, mOverviewProxyService);
mFloatingRotationButton = new FloatingRotationButton(context);
mRotationButtonController = new RotationButtonController(context,
R.style.RotateButtonCCWStart90,
isGesturalMode ? mFloatingRotationButton : rotateSuggestionButton,
mRotationButtonListener);
 
mConfiguration = new Configuration();
mTmpLastConfiguration = new Configuration();
mConfiguration.updateFrom(context.getResources().getConfiguration());
 
mScreenPinningNotify = new ScreenPinningNotify(mContext);
mBarTransitions = new NavigationBarTransitions(this, Dependency.get(CommandQueue.class));
 
mButtonDispatchers.put(R.id.back, new ButtonDispatcher(R.id.back));
mButtonDispatchers.put(R.id.home, new ButtonDispatcher(R.id.home));
mButtonDispatchers.put(R.id.home_handle, new ButtonDispatcher(R.id.home_handle));
mButtonDispatchers.put(R.id.recent_apps, new ButtonDispatcher(R.id.recent_apps));
mButtonDispatchers.put(R.id.ime_switcher, imeSwitcherButton);
mButtonDispatchers.put(R.id.accessibility_button, accessibilityButton);
mButtonDispatchers.put(R.id.rotate_suggestion, rotateSuggestionButton);
mButtonDispatchers.put(R.id.menu_container, mContextualButtonGroup);
mDeadZone = new DeadZone(this);
 
mNavColorSampleMargin = getResources()
getDimensionPixelSize(R.dimen.navigation_handle_sample_horizontal_margin);
mEdgeBackGestureHandler = new EdgeBackGestureHandler(context, mOverviewProxyService,
mSysUiFlagContainer, mPluginManager, this::updateStates);
mRegionSamplingHelper = new RegionSamplingHelper(this,
new RegionSamplingHelper.SamplingCallback() {
@Override
public void onRegionDarknessChanged(boolean isRegionDark) {
getLightTransitionsController().setIconsDark(!isRegionDark ,
true /* animate */);
}
 
@Override
public Rect getSampledRegion(View sampledView) {
if (mOrientedHandleSamplingRegion != null) {
return mOrientedHandleSamplingRegion;
}
 
updateSamplingRect();
return mSamplingBounds;
}
 
@Override
public boolean isSamplingEnabled() {
return isGesturalModeOnDefaultDisplay(getContext(), mNavBarMode);
}
});
}
在mEdgeBackGestureHandler = new EdgeBackGestureHandler(context, mOverviewProxyService,
mSysUiFlagContainer, mPluginManager, this::updateStates);中可以看出手势返回的UI布局在
EdgeBackGestureHandler中加载 接下来看下EdgeBackGestureHandler的相关布局
```

#### 3.2 EdgeBackGestureHandler 手势导航布局左右手势返回的相关代码

```
public class EdgeBackGestureHandler extends CurrentUserTracker implements DisplayListener,
PluginListener<NavigationEdgeBackPlugin>, ProtoTraceable<SystemUiTraceProto> {
 
private static final String TAG = "EdgeBackGestureHandler";
private static final int MAX_LONG_PRESS_TIMEOUT = SystemProperties.getInt(
"gestures.back_timeout", 250);
 
private ISystemGestureExclusionListener mGestureExclusionListener =
new ISystemGestureExclusionListener.Stub() {
@Override
public void onSystemGestureExclusionChanged(int displayId,
Region systemGestureExclusion, Region unrestrictedOrNull) {
if (displayId == mDisplayId) {
mMainExecutor.execute(() -> {
mExcludeRegion.set(systemGestureExclusion);
mUnrestrictedExcludeRegion.set(unrestrictedOrNull != null
? unrestrictedOrNull : systemGestureExclusion);
});
}
}
};
 
private OverviewProxyService.OverviewProxyListener mQuickSwitchListener =
new OverviewProxyService.OverviewProxyListener() {
@Override
public void onQuickSwitchToNewTask(@Surface.Rotation int rotation) {
mStartingQuickstepRotation = rotation;
updateDisabledForQuickstep();
}
};
 
private TaskStackChangeListener mTaskStackListener = new TaskStackChangeListener() {
@Override
public void onTaskStackChanged() {
mGestureBlockingActivityRunning = isGestureBlockingActivityRunning();
}
};
 
private final Context mContext;
private final OverviewProxyService mOverviewProxyService;
private final Runnable mStateChangeCallback;
 
private final PluginManager mPluginManager;
// Activities which should not trigger Back gesture.
private final List<ComponentName> mGestureBlockingActivities = new ArrayList<>();
 
private final Point mDisplaySize = new Point();
private final int mDisplayId;
 
private final Executor mMainExecutor;
 
private final Region mExcludeRegion = new Region();
private final Region mUnrestrictedExcludeRegion = new Region();
 
// The left side edge width where touch down is allowed
private int mEdgeWidthLeft;
// The right side edge width where touch down is allowed
private int mEdgeWidthRight;
// The bottom gesture area height
private float mBottomGestureHeight;
// The slop to distinguish between horizontal and vertical motion
private float mTouchSlop;
// Duration after which we consider the event as longpress.
private final int mLongPressTimeout;
private int mStartingQuickstepRotation = -1;
// We temporarily disable back gesture when user is quickswitching
// between apps of different orientations
private boolean mDisabledForQuickstep;
 
private final PointF mDownPoint = new PointF();
private final PointF mEndPoint = new PointF();
private boolean mThresholdCrossed = false;
private boolean mAllowGesture = false;
private boolean mLogGesture = false;
private boolean mInRejectedExclusion = false;
private boolean mIsOnLeftEdge;
 
private boolean mIsAttached;
private boolean mIsGesturalModeEnabled;
private boolean mIsEnabled;
private boolean mIsNavBarShownTransiently;
private boolean mIsBackGestureAllowed;
private boolean mGestureBlockingActivityRunning;
 
private InputMonitor mInputMonitor;
private InputEventReceiver mInputEventReceiver;
 
private NavigationEdgeBackPlugin mEdgeBackPlugin;
private int mLeftInset;
private int mRightInset;
private int mSysUiFlags;
 
public EdgeBackGestureHandler(Context context, OverviewProxyService overviewProxyService,
SysUiState sysUiFlagContainer, PluginManager pluginManager,
Runnable stateChangeCallback) {
super(Dependency.get(BroadcastDispatcher.class));
mContext = context;
mDisplayId = context.getDisplayId();
mMainExecutor = context.getMainExecutor();
mOverviewProxyService = overviewProxyService;
mPluginManager = pluginManager;
mStateChangeCallback = stateChangeCallback;
ComponentName recentsComponentName = ComponentName.unflattenFromString(
context.getString(com.android.internal.R.string.config_recentsComponentName));
if (recentsComponentName != null) {
String recentsPackageName = recentsComponentName.getPackageName();
PackageManager manager = context.getPackageManager();
try {
Resources resources = manager.getResourcesForApplication(recentsPackageName);
int resId = resources.getIdentifier(
"gesture_blocking_activities", "array", recentsPackageName);
 
if (resId == 0) {
Log.e(TAG, "No resource found for gesture-blocking activities");
} else {
String[] gestureBlockingActivities = resources.getStringArray(resId);
for (String gestureBlockingActivity : gestureBlockingActivities) {
mGestureBlockingActivities.add(
ComponentName.unflattenFromString(gestureBlockingActivity));
}
}
} catch (NameNotFoundException e) {
Log.e(TAG, "Failed to add gesture blocking activities", e);
}
}
 
mLongPressTimeout = Math.min(MAX_LONG_PRESS_TIMEOUT,
ViewConfiguration.getLongPressTimeout());
 
mGestureNavigationSettingsObserver = new GestureNavigationSettingsObserver(
mContext.getMainThreadHandler(), mContext, this::onNavigationSettingsChanged);
 
updateCurrentUserResources();
sysUiFlagContainer.addCallback(sysUiFlags -> mSysUiFlags = sysUiFlags);
}
 
public void updateCurrentUserResources() {
Resources res = Dependency.get(NavigationModeController.class).getCurrentUserContext()
getResources();
mEdgeWidthLeft = mGestureNavigationSettingsObserver.getLeftSensitivity(res);
mEdgeWidthRight = mGestureNavigationSettingsObserver.getRightSensitivity(res);
mIsBackGestureAllowed =
!mGestureNavigationSettingsObserver.areNavigationButtonForcedVisible();
 
final DisplayMetrics dm = res.getDisplayMetrics();
final float defaultGestureHeight = res.getDimension(
com.android.internal.R.dimen.navigation_bar_gesture_height) / dm.density;
final float gestureHeight = DeviceConfig.getFloat(DeviceConfig.NAMESPACE_SYSTEMUI,
SystemUiDeviceConfigFlags.BACK_GESTURE_BOTTOM_HEIGHT,
defaultGestureHeight);
mBottomGestureHeight = TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, gestureHeight,
dm);
 
// Reduce the default touch slop to ensure that we can intercept the gesture
// before the app starts to react to it.
// TODO(b/130352502) Tune this value and extract into a constant
final float backGestureSlop = DeviceConfig.getFloat(DeviceConfig.NAMESPACE_SYSTEMUI,
SystemUiDeviceConfigFlags.BACK_GESTURE_SLOP_MULTIPLIER, 0.75f);
mTouchSlop = ViewConfiguration.get(mContext).getScaledTouchSlop() * backGestureSlop;
}
 
private void onNavigationSettingsChanged() {
boolean wasBackAllowed = isHandlingGestures();
updateCurrentUserResources();
if (wasBackAllowed != isHandlingGestures()) {
mStateChangeCallback.run();
}
}
 
@Override
public void onUserSwitched(int newUserId) {
updateIsEnabled();
updateCurrentUserResources();
}
 
/**
* @see NavigationBarView#onAttachedToWindow()
*/
public void onNavBarAttached() {
mIsAttached = true;
Dependency.get(ProtoTracer.class).add(this);
mOverviewProxyService.addCallback(mQuickSwitchListener);
updateIsEnabled();
startTracking();
}
 
/**
* @see NavigationBarView#onDetachedFromWindow()
*/
public void onNavBarDetached() {
mIsAttached = false;
Dependency.get(ProtoTracer.class).remove(this);
mOverviewProxyService.removeCallback(mQuickSwitchListener);
updateIsEnabled();
stopTracking();
}
 
/**
* @see NavigationModeController.ModeChangedListener#onNavigationModeChanged
*/
public void onNavigationModeChanged(int mode) {
mIsGesturalModeEnabled = QuickStepContract.isGesturalMode(mode);
updateIsEnabled();
updateCurrentUserResources();
}
 
public void onNavBarTransientStateChanged(boolean isTransient) {
mIsNavBarShownTransiently = isTransient;
}
 
private void disposeInputChannel() {
if (mInputEventReceiver != null) {
mInputEventReceiver.dispose();
mInputEventReceiver = null;
}
if (mInputMonitor != null) {
mInputMonitor.dispose();
mInputMonitor = null;
}
}
 
private void updateIsEnabled() {
boolean isEnabled = mIsAttached && mIsGesturalModeEnabled;
if (isEnabled == mIsEnabled) {
return;
}
mIsEnabled = isEnabled;
disposeInputChannel();
 
if (mEdgeBackPlugin != null) {
mEdgeBackPlugin.onDestroy();
mEdgeBackPlugin = null;
}
 
if (!mIsEnabled) {
mGestureNavigationSettingsObserver.unregister();
mContext.getSystemService(DisplayManager.class).unregisterDisplayListener(this);
mPluginManager.removePluginListener(this);
ActivityManagerWrapper.getInstance().unregisterTaskStackListener(mTaskStackListener);
 
try {
WindowManagerGlobal.getWindowManagerService()
unregisterSystemGestureExclusionListener(
mGestureExclusionListener, mDisplayId);
} catch (RemoteException | IllegalArgumentException e) {
Log.e(TAG, "Failed to unregister window manager callbacks", e);
}
 
} else {
mGestureNavigationSettingsObserver.register();
updateDisplaySize();
mContext.getSystemService(DisplayManager.class).registerDisplayListener(this,
mContext.getMainThreadHandler());
ActivityManagerWrapper.getInstance().registerTaskStackListener(mTaskStackListener);
 
try {
WindowManagerGlobal.getWindowManagerService()
registerSystemGestureExclusionListener(
mGestureExclusionListener, mDisplayId);
} catch (RemoteException | IllegalArgumentException e) {
Log.e(TAG, "Failed to register window manager callbacks", e);
}
 
// Register input event receiver
mInputMonitor = InputManager.getInstance().monitorGestureInput(
"edge-swipe", mDisplayId);
mInputEventReceiver = new SysUiInputEventReceiver(
mInputMonitor.getInputChannel(), Looper.getMainLooper());
 
// Add a nav bar panel window
setEdgeBackPlugin(new NavigationBarEdgePanel(mContext));
mPluginManager.addPluginListener(
this, NavigationEdgeBackPlugin.class, /*allowMultiple=*/ false);
}
}
 
setEdgeBackPlugin(new NavigationBarEdgePanel(mContext));这里加载真正的手势导航返回布局
private void setEdgeBackPlugin(NavigationEdgeBackPlugin edgeBackPlugin) {
if (mEdgeBackPlugin != null) {
mEdgeBackPlugin.onDestroy();
}
mEdgeBackPlugin = edgeBackPlugin;
mEdgeBackPlugin.setBackCallback(mBackCallback);
mEdgeBackPlugin.setLayoutParams(createLayoutParams());
updateDisplaySize();
}
 
public boolean isHandlingGestures() {
return mIsEnabled && mIsBackGestureAllowed;
}
 
private WindowManager.LayoutParams createLayoutParams() {
Resources resources = mContext.getResources();
WindowManager.LayoutParams layoutParams = new WindowManager.LayoutParams(
resources.getDimensionPixelSize(R.dimen.navigation_edge_panel_width),
resources.getDimensionPixelSize(R.dimen.navigation_edge_panel_height),
WindowManager.LayoutParams.TYPE_NAVIGATION_BAR_PANEL,
WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
| WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL
| WindowManager.LayoutParams.FLAG_SPLIT_TOUCH
| WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN,
PixelFormat.TRANSLUCENT);
layoutParams.privateFlags |=
WindowManager.LayoutParams.SYSTEM_FLAG_SHOW_FOR_ALL_USERS;
layoutParams.setTitle(TAG + mContext.getDisplayId());
layoutParams.accessibilityTitle = mContext.getString(R.string.nav_bar_edge_panel);
layoutParams.windowAnimations = 0;
layoutParams.setFitInsetsTypes(0 /* types */);
return layoutParams;
}
 
resources.getDimensionPixelSize(R.dimen.navigation_edge_panel_width),
resources.getDimensionPixelSize(R.dimen.navigation_edge_panel_height)
设置手势返回UI样式的宽高
```

#### 3.3 NavigationBarEdgePanel 手势导航布局左右手势返回的相关代码

```
public class NavigationBarEdgePanel extends View implements NavigationEdgeBackPlugin {
 
private static final String TAG = "NavigationBarEdgePanel";
 
private static final long COLOR_ANIMATION_DURATION_MS = 120;
private static final long DISAPPEAR_FADE_ANIMATION_DURATION_MS = 80;
private static final long DISAPPEAR_ARROW_ANIMATION_DURATION_MS = 100;
 
/**
* The time required since the first vibration effect to automatically trigger a click
*/
private static final int GESTURE_DURATION_FOR_CLICK_MS = 400;
 
/**
* The size of the protection of the arrow in px. Only used if this is not background protected
*/
private static final int PROTECTION_WIDTH_PX = 2;
 
/**
* The basic translation in dp where the arrow resides
*/
private static final int BASE_TRANSLATION_DP = 32;
 
/**
* The length of the arrow leg measured from the center to the end
*/
private static final int ARROW_LENGTH_DP = 18;
 
/**
* The angle measured from the xAxis, where the leg is when the arrow rests
*/
private static final int ARROW_ANGLE_WHEN_EXTENDED_DEGREES = 56;
 
/**
* The angle that is added per 1000 px speed to the angle of the leg
*/
private static final int ARROW_ANGLE_ADDED_PER_1000_SPEED = 4;
 
/**
* The maximum angle offset allowed due to speed
*/
private static final int ARROW_MAX_ANGLE_SPEED_OFFSET_DEGREES = 4;
 
/**
* The thickness of the arrow. Adjusted to match the home handle (approximately)
*/
private static final float ARROW_THICKNESS_DP = 2.5f;
 
/**
* The amount of rubber banding we do for the vertical translation
*/
private static final int RUBBER_BAND_AMOUNT = 15;
 
/**
* The interpolator used to rubberband
*/
private static final Interpolator RUBBER_BAND_INTERPOLATOR
= new PathInterpolator(1.0f / 5.0f, 1.0f, 1.0f, 1.0f);
 
/**
* The amount of rubber banding we do for the translation before base translation
*/
private static final int RUBBER_BAND_AMOUNT_APPEAR = 4;
 
/**
* The interpolator used to rubberband the appearing of the arrow.
*/
private static final Interpolator RUBBER_BAND_INTERPOLATOR_APPEAR
= new PathInterpolator(1.0f / RUBBER_BAND_AMOUNT_APPEAR, 1.0f, 1.0f, 1.0f);
 
private final WindowManager mWindowManager;
private final VibratorHelper mVibratorHelper;
 
/**
* The paint the arrow is drawn with
*/
private final Paint mPaint = new Paint();
/**
* The paint the arrow protection is drawn with
*/
private final Paint mProtectionPaint;
 
private final float mDensity;
private final float mBaseTranslation;
private final float mArrowLength;
private final float mArrowThickness;
 
/**
* The minimum delta needed in movement for the arrow to change direction / stop triggering back
*/
private final float mMinDeltaForSwitch;
// The closest to y = 0 that the arrow will be displayed.
private int mMinArrowPosition;
// The amount the arrow is shifted to avoid the finger.
private int mFingerOffset;
 
private final float mSwipeThreshold;
private final Path mArrowPath = new Path();
private final Point mDisplaySize = new Point();
 
private final SpringAnimation mAngleAnimation;
private final SpringAnimation mTranslationAnimation;
private final SpringAnimation mVerticalTranslationAnimation;
private final SpringForce mAngleAppearForce;
private final SpringForce mAngleDisappearForce;
private final ValueAnimator mArrowColorAnimator;
private final ValueAnimator mArrowDisappearAnimation;
private final SpringForce mRegularTranslationSpring;
private final SpringForce mTriggerBackSpring;
 
private VelocityTracker mVelocityTracker;
private boolean mIsDark = false;
private boolean mShowProtection = false;
private int mProtectionColorLight;
private int mArrowPaddingEnd;
private int mArrowColorLight;
private int mProtectionColorDark;
private int mArrowColorDark;
private int mProtectionColor;
private int mArrowColor;
private RegionSamplingHelper mRegionSamplingHelper;
private final Rect mSamplingRect = new Rect();
private WindowManager.LayoutParams mLayoutParams;
private int mLeftInset;
private int mRightInset;
 
/**
* True if the panel is currently on the left of the screen
*/
private boolean mIsLeftPanel;
 
private float mStartX;
private float mStartY;
private float mCurrentAngle;
/**
* The current translation of the arrow
*/
private float mCurrentTranslation;
/**
* Where the arrow will be in the resting position.
*/
private float mDesiredTranslation;
 
private boolean mDragSlopPassed;
private boolean mArrowsPointLeft;
private float mMaxTranslation;
private boolean mTriggerBack;
private float mPreviousTouchTranslation;
private float mTotalTouchDelta;
private float mVerticalTranslation;
private float mDesiredVerticalTranslation;
private float mDesiredAngle;
private float mAngleOffset;
private int mArrowStartColor;
private int mCurrentArrowColor;
private float mDisappearAmount;
private long mVibrationTime;
private int mScreenSize;
 
public NavigationBarEdgePanel(Context context) {
super(context);
 
mWindowManager = context.getSystemService(WindowManager.class);
mVibratorHelper = Dependency.get(VibratorHelper.class);
 
mDensity = context.getResources().getDisplayMetrics().density;
 
mBaseTranslation = dp(BASE_TRANSLATION_DP);
mArrowLength = dp(ARROW_LENGTH_DP);
mArrowThickness = dp(ARROW_THICKNESS_DP);
mMinDeltaForSwitch = dp(32);
 
mPaint.setStrokeWidth(mArrowThickness);
mPaint.setStrokeCap(Paint.Cap.ROUND);
mPaint.setAntiAlias(true);
mPaint.setStyle(Paint.Style.STROKE);
mPaint.setStrokeJoin(Paint.Join.ROUND);
 
mArrowColorAnimator = ValueAnimator.ofFloat(0.0f, 1.0f);
mArrowColorAnimator.setDuration(COLOR_ANIMATION_DURATION_MS);
mArrowColorAnimator.addUpdateListener(animation -> {
int newColor = ColorUtils.blendARGB(
mArrowStartColor, mArrowColor, animation.getAnimatedFraction());
setCurrentArrowColor(newColor);
});
 
mArrowDisappearAnimation = ValueAnimator.ofFloat(0.0f, 1.0f);
mArrowDisappearAnimation.setDuration(DISAPPEAR_ARROW_ANIMATION_DURATION_MS);
mArrowDisappearAnimation.setInterpolator(Interpolators.FAST_OUT_SLOW_IN);
mArrowDisappearAnimation.addUpdateListener(animation -> {
mDisappearAmount = (float) animation.getAnimatedValue();
invalidate();
});
 
mAngleAnimation =
new SpringAnimation(this, CURRENT_ANGLE);
mAngleAppearForce = new SpringForce()
setStiffness(500)
setDampingRatio(0.5f);
mAngleDisappearForce = new SpringForce()
setStiffness(SpringForce.STIFFNESS_MEDIUM)
setDampingRatio(SpringForce.DAMPING_RATIO_MEDIUM_BOUNCY)
setFinalPosition(90);
mAngleAnimation.setSpring(mAngleAppearForce).setMaxValue(90);
 
mTranslationAnimation =
new SpringAnimation(this, CURRENT_TRANSLATION);
mRegularTranslationSpring = new SpringForce()
setStiffness(SpringForce.STIFFNESS_MEDIUM)
setDampingRatio(SpringForce.DAMPING_RATIO_LOW_BOUNCY);
mTriggerBackSpring = new SpringForce()
setStiffness(450)
setDampingRatio(SpringForce.DAMPING_RATIO_LOW_BOUNCY);
mTranslationAnimation.setSpring(mRegularTranslationSpring);
mVerticalTranslationAnimation =
new SpringAnimation(this, CURRENT_VERTICAL_TRANSLATION);
mVerticalTranslationAnimation.setSpring(
new SpringForce()
setStiffness(SpringForce.STIFFNESS_MEDIUM)
setDampingRatio(SpringForce.DAMPING_RATIO_LOW_BOUNCY));
 
mProtectionPaint = new Paint(mPaint);
mProtectionPaint.setStrokeWidth(mArrowThickness + PROTECTION_WIDTH_PX);
loadDimens();
 
loadColors(context);
updateArrowDirection();
 
mSwipeThreshold = context.getResources()
getDimension(R.dimen.navigation_edge_action_drag_threshold);
setVisibility(GONE);
 
mRegionSamplingHelper = new RegionSamplingHelper(this,
new RegionSamplingHelper.SamplingCallback() {
@Override
public void onRegionDarknessChanged(boolean isRegionDark) {
setIsDark(!isRegionDark, true /* animate */);
}
 
@Override
public Rect getSampledRegion(View sampledView) {
return mSamplingRect;
}
});
mRegionSamplingHelper.setWindowVisible(true);
}
 
@Override
protected void onDraw(Canvas canvas) {
float pointerPosition = mCurrentTranslation - mArrowThickness / 2.0f;
canvas.save();
canvas.translate(
mIsLeftPanel ? pointerPosition : getWidth() - pointerPosition,
(getHeight() * 0.5f) + mVerticalTranslation);
 
// Let's calculate the position of the end based on the angle
float x = (polarToCartX(mCurrentAngle) * mArrowLength);
float y = (polarToCartY(mCurrentAngle) * mArrowLength);
Path arrowPath = calculatePath(x,y);
if (mShowProtection) {
canvas.drawPath(arrowPath, mProtectionPaint);
}
canvas.drawPath(arrowPath, mPaint);
canvas.restore();
}
在 onDraw(Canvas canvas)中绘制手势返回UI布局 所以修改就在这里即可
```

  
4. 自定义仿小米全面屏手势导航返回 ui 布局功能实现
-------------------------------

```
NavigationBarEdgePanel中定义相关参数
    private Path bgPath;
    private Paint bgPaint,arrowPaint;
    private Path arrowPath;
    private float viewHeight=310.5f;
    private float currentSlideLength = 100.5f;
    private float viewArrowSize = 5f; // 箭头图标大小
    private float viewMaxLength = 75f; // 最大拉动距离
 
在NavigationBarEdgePanel(Context context)构造方法中实现
        arrowPath = new Path();
        bgPath = new Path();
        bgPaint = new Paint();
        bgPaint.setColor(Color.BLACK);
        bgPaint.setStrokeWidth(1f);
        bgPaint.setStyle(Paint.Style.FILL_AND_STROKE);
        bgPaint.setAlpha(180);
        bgPaint.setAntiAlias(true);
        arrowPaint = new Paint();
        arrowPaint.setColor(Color.WHITE);
        arrowPaint.setStrokeWidth(4f);
        arrowPaint.setStyle(Paint.Style.STROKE);
        arrowPaint.setStrokeJoin(Paint.Join.ROUND);
 
在onDraw(Canvas canvas)中的修改
@Override
protected void onDraw(Canvas canvas) {
/*float pointerPosition = mCurrentTranslation - mArrowThickness / 2.0f;
canvas.save();
canvas.translate(
mIsLeftPanel ? pointerPosition : getWidth() - pointerPosition,
(getHeight() * 0.5f) + mVerticalTranslation);
// Let's calculate the position of the end based on the angle
float x = (polarToCartX(mCurrentAngle) * mArrowLength);
float y = (polarToCartY(mCurrentAngle) * mArrowLength);
Path arrowPath = calculatePath(x,y);
if (mShowProtection) {
canvas.drawPath(arrowPath, mProtectionPaint);
}
canvas.drawPath(arrowPath, mPaint);
canvas.restore();*/
 
//add core start
float arrowZoom = currentSlideLength / viewMaxLength ;// 箭头大小变化率
		float arrowAngle = 0f;
		if (arrowZoom < 0.75f) {
			arrowAngle = 0f;
		}else {
			arrowAngle = (arrowZoom - 0.75f) * 2;// 箭头角度变化率
		}
		if(mIsLeftPanel){//手势右滑
			bgPath.reset();
			bgPath.moveTo(0f, 0f);
			bgPath.cubicTo(0f, viewHeight*2/9 , currentSlideLength, viewHeight / 3, currentSlideLength, viewHeight / 2);
			bgPath.cubicTo(currentSlideLength, viewHeight * 2 / 3, 0f, viewHeight * 7 / 9, 0f, viewHeight);
			canvas.drawPath(bgPath, bgPaint);
			arrowPath.reset();
			arrowPath.moveTo(currentSlideLength / 2 + viewArrowSize * arrowAngle, viewHeight / 2 - arrowZoom * viewArrowSize-5);
			arrowPath.lineTo(currentSlideLength / 2 - viewArrowSize * arrowAngle, viewHeight / 2);
			arrowPath.lineTo(currentSlideLength / 2 + viewArrowSize * arrowAngle, viewHeight / 2 + arrowZoom * viewArrowSize+5);
			canvas.drawPath(arrowPath, arrowPaint);
		}else{//手势左滑
			bgPath.reset();
            bgPath.moveTo(getWidth(), 0f);
            bgPath.cubicTo(getWidth(), viewHeight*2/9 , getWidth()-currentSlideLength, viewHeight / 3, getWidth()-currentSlideLength, viewHeight / 2);
            bgPath.cubicTo(getWidth()-currentSlideLength, viewHeight * 2 / 3, getWidth(), viewHeight * 7 / 9, getWidth(), viewHeight);
			canvas.drawPath(bgPath, bgPaint);
			arrowPath.reset();
			arrowPath.moveTo(currentSlideLength / 2 - viewArrowSize * arrowAngle+getWidth()-currentSlideLength, viewHeight / 2 - arrowZoom * viewArrowSize-5);
			arrowPath.lineTo(currentSlideLength / 2 + viewArrowSize * arrowAngle+getWidth()-currentSlideLength, viewHeight / 2);
			arrowPath.lineTo(currentSlideLength / 2 - viewArrowSize * arrowAngle+getWidth()-currentSlideLength, viewHeight / 2 + arrowZoom * viewArrowSize+5);
			canvas.drawPath(arrowPath, arrowPaint);
		}
// add core end
}
 
关于手势返回UI高度的修改
--- a/frameworks/base/packages/SystemUI/res/values/dimens.xml
+++ b/frameworks/base/packages/SystemUI/res/values/dimens.xml
@@ -43,7 +43,7 @@
-    <dimen >96dp</dimen>
+    <dimen >150dp</dimen>
```
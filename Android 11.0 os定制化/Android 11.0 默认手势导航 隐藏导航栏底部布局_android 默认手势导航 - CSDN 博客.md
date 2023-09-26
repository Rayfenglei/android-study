> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125879912)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 手势导航隐藏导航栏底部布局的核心代码](#t1)

[3. 手势导航隐藏导航栏底部布局的核心代码以及功能分析](#t2)

 [3.1NavigationBarView.java 关于手势导航的布局代码分析](#t3)

[3.2 查看 home_handle.xml 的相关代码](#t4)

[3.3 NavigationHandle 中关于 Home 布局的相关代码](#t5)

[4. 手势导航隐藏导航栏底部布局具体功能实现](#t6)

1. 概述
-----

原生系统默认是三键导航，也可以切换到手势导航，当默认设置为手势导航时，底部导航栏 home 键布局就是一条黑线，感觉不太美观  
所以需要默认隐藏这一条黑线的布局，这就需要找到黑线布局相关的代码

2. 手势导航隐藏导航栏底部布局的核心代码
---------------------

```
主要代码:
frameworks\base\packages\SystemUI\res\layout\home_handle.xml
  frameworks\base\packages\SystemUI\src\com\android\systemui\statusbar\phone\NavigationHandle.java
  frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarView.java
```

3. 手势导航隐藏导航栏底部布局的核心代码以及功能分析
---------------------------

####   3.1NavigationBarView.java 关于手势导航的布局代码分析

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
/**
* When quickswitching between apps of different orientations, we draw a secondary home handle
* in the position of the first app's orientation. This rect represents the region of that
* home handle so we can apply the correct light/dark luma on that.
* @see {@link NavigationBarFragment#mOrientationHandle}
*/
@Nullable
private Rect mOrientedHandleSamplingRegion;
 
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
 
```

#### 通过查看代码发现 mButtonDispatchers.put(R.id.home_handle, new ButtonDispatcher(R.id.home_handle));  
就是手势导航时的底部 home 布局

#### 3.2 查看 home_handle.xml 的相关代码

```
<com.android.systemui.statusbar.phone.NavigationHandle
xmlns:android="http://schemas.android.com/apk/res/android"
android:id="@+id/home_handle"
android:layout_width="@dimen/navigation_home_handle_width"
android:layout_height="match_parent"
android:layout_weight="0"
android:contentDescription="@string/accessibility_home"
android:paddingStart="@dimen/navigation_key_padding"
android:paddingEnd="@dimen/navigation_key_padding"
/>
从布局中看到正在的绘制在NavigationHandle中
```

#### 3.3 NavigationHandle 中关于 Home 布局的相关代码

```
public class NavigationHandle extends View implements ButtonInterface {
 
protected final Paint mPaint = new Paint();
private @ColorInt final int mLightColor;
private @ColorInt final int mDarkColor;
protected final int mRadius;
protected final int mBottom;
private boolean mRequiresInvalidate;
 
public NavigationHandle(Context context) {
this(context, null);
}
 
public NavigationHandle(Context context, AttributeSet attr) {
super(context, attr);
final Resources res = context.getResources();
mRadius = res.getDimensionPixelSize(R.dimen.navigation_handle_radius);
mBottom = res.getDimensionPixelSize(R.dimen.navigation_handle_bottom);
 
final int dualToneDarkTheme = Utils.getThemeAttr(context, R.attr.darkIconTheme);
final int dualToneLightTheme = Utils.getThemeAttr(context, R.attr.lightIconTheme);
Context lightContext = new ContextThemeWrapper(context, dualToneLightTheme);
Context darkContext = new ContextThemeWrapper(context, dualToneDarkTheme);
mLightColor = Utils.getColorAttrDefaultColor(lightContext, R.attr.homeHandleColor);
mDarkColor = Utils.getColorAttrDefaultColor(darkContext, R.attr.homeHandleColor);
mPaint.setAntiAlias(true);
setFocusable(false);
}
 
@Override
public void setAlpha(float alpha) {
super.setAlpha(alpha);
if (alpha > 0f && mRequiresInvalidate) {
mRequiresInvalidate = false;
invalidate();
}
}
 
@Override
protected void onDraw(Canvas canvas) {
super.onDraw(canvas);
 
// Draw that bar
int navHeight = getHeight();
int height = mRadius * 2;
int width = getWidth();
int y = (navHeight - mBottom - height);
canvas.drawRoundRect(0, y, width, y + height, mRadius, mRadius, mPaint);
}
 
@Override
public void setImageDrawable(Drawable drawable) {
}
 
@Override
public void abortCurrentGesture() {
}
 
@Override
public void setVertical(boolean vertical) {
}
 
@Override
public void setDarkIntensity(float intensity) {
int color = (int) ArgbEvaluator.getInstance().evaluate(intensity, mLightColor, mDarkColor);
if (mPaint.getColor() != color) {
mPaint.setColor(color);
if (getVisibility() == VISIBLE && getAlpha() > 0) {
invalidate();
} else {
// If we are currently invisible, then invalidate when we are next made visible
mRequiresInvalidate = true;
}
}
}
 
@Override
public void setDelayTouchFeedback(boolean shouldDelay) {
}
}
```

  
代码中发现 protected void onDraw(Canvas canvas) 负责绘制手势导航 Home 布局  
所以具体修改就是在这里
----------------------------------------------------------------------------

4. 手势导航隐藏导航栏底部布局具体功能实现
----------------------

```
@Override
protected void onDraw(Canvas canvas) {
super.onDraw(canvas);
 
// Draw that bar
int navHeight = getHeight();
int height = mRadius * 2;
int width = getWidth();
int y = (navHeight - mBottom - height);
// canvas.drawRoundRect(0, y, width, y + height, mRadius, mRadius, mPaint);
}
注释掉canvas.drawRoundRect(0, y, width, y + height, mRadius, mRadius, mPaint);即可
```
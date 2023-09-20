> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125228773)

### 1. 概述

在 11.0 产品定制化中，[SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 的相关功能需求需要为导航栏添加虚拟按键来实现某些功能，比如添加 wifi，可以通过点击 wifi 跳转到 wifi 页面，日期可以弹出当前万年历功能，所以需要对导航栏进行定制来实现导航栏的功能

### 2.SystemUI 导航栏 添加虚拟按键 (一) 相关代码

```
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarFragment.java
/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarInflaterView.java

```

### 3.SystemUI 导航栏 添加虚拟按键 (一) 功能分析和实现功能

### 3.1 SystemUI 导航栏布局的分析

NavigationBarFragment 的创建是从 StatusBar.makeStatusBarView  
创建导航栏功能 而具体实现是在 NavigationBarFragment

```
protected void makeStatusBarView() {
       ...
       try {
            boolean showNav = mWindowManagerService.hasNavigationBar();
            if (DEBUG) Log.v(TAG, "hasNavigationBar=" + showNav);
            if (showNav) {
                //创建导航栏
                createNavigationBar();
            }
        } catch (Exception ex) {
            // no window manager? good luck with that
        }
  }
  ...

  protected void createNavigationBar() {
        mNavigationBarView = NavigationBarFragment.create(mContext, (tag, fragment) -> {
            mNavigationBar = (NavigationBarFragment) fragment;
            if (mLightBarController != null) {
                mNavigationBar.setLightBarController(mLightBarController);
            }
            mNavigationBar.setCurrentSysuiVisibility(mSystemUiVisibility);
        });
  }

```

下面接着看下 NavigationBarFragment 的代码

```
@Override
      public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container,
              Bundle savedInstanceState) {
          return inflater.inflate(R.layout.navigation_bar, container, false);
      }
  
      @Override
      public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
          super.onViewCreated(view, savedInstanceState);
          mNavigationBarView = (NavigationBarView) view;
          final Display display = view.getDisplay();
          // It may not have display when running unit test.
          if (display != null) {
              mDisplayId = display.getDisplayId();
              mIsOnDefaultDisplay = mDisplayId == Display.DEFAULT_DISPLAY;
          }
  
          mNavigationBarView.setComponents(mStatusBarLazy.get().getPanelController());
          mNavigationBarView.setDisabledFlags(mDisabledFlags1);
          mNavigationBarView.setOnVerticalChangedListener(this::onVerticalChanged);
          mNavigationBarView.setOnTouchListener(this::onNavigationTouch);
          if (savedInstanceState != null) {
              mNavigationBarView.getLightTransitionsController().restoreState(savedInstanceState);
          }
          mNavigationBarView.setNavigationIconHints(mNavigationIconHints);
          mNavigationBarView.setWindowVisible(isNavBarWindowVisible());
  
          prepareNavigationBarView();
          checkNavBarModes();
  
          IntentFilter filter = new IntentFilter(Intent.ACTION_SCREEN_OFF);
          filter.addAction(Intent.ACTION_SCREEN_ON);
          filter.addAction(Intent.ACTION_USER_SWITCHED);
          mBroadcastDispatcher.registerReceiverWithHandler(mBroadcastReceiver, filter,
                  Handler.getMain(), UserHandle.ALL);
          notifyNavigationBarScreenOn();
  
          mOverviewProxyService.addCallback(mOverviewProxyListener);
          updateSystemUiStateFlags(-1);
  
          // Currently there is no accelerometer sensor on non-default display.
          if (mIsOnDefaultDisplay) {
              final RotationButtonController rotationButtonController =
                      mNavigationBarView.getRotationButtonController();
              rotationButtonController.addRotationCallback(mRotationWatcher);
  
              // Reset user rotation pref to match that of the WindowManager if starting in locked
              // mode. This will automatically happen when switching from auto-rotate to locked mode.
              if (display != null && rotationButtonController.isRotationLocked()) {
                  rotationButtonController.setRotationLockedAtAngle(display.getRotation());
              }
          } else {
              mDisabledFlags2 |= StatusBarManager.DISABLE2_ROTATE_SUGGESTIONS;
          }
          setDisabled2Flags(mDisabledFlags2);
          if (mIsOnDefaultDisplay) {
              mAssistHandlerViewController =
                  new AssistHandleViewController(mHandler, mNavigationBarView);
              getBarTransitions().addDarkIntensityListener(mAssistHandlerViewController);
          }
  
          initSecondaryHomeHandleForRotation();
      }

```

通过 onCreateView 发现具体的布局文件是 navigation_bar.xml 文件  
而具体的布局就是 com.android.systemui.statusbar.phone.NavigationBarInflaterView  
所以说 SystemUI 导航栏布局是由 NavigationBarInflaterView.java 负责绘制的  
接下来看 NavigationBarInflaterView 相关代码

### 3.2NavigationBarInflaterView 相关布局代码分析

路径: /frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarInflaterView.java  
主要分析关于构建按键的相关布局和位置的代码，然后新增布局来实现功能  
查看源码如下:

```
public class NavigationBarInflaterView extends FrameLayout
implements NavigationModeController.ModeChangedListener {

private static final String TAG = "NavBarInflater";

public static final String NAV_BAR_VIEWS = "sysui_nav_bar";
public static final String NAV_BAR_LEFT = "sysui_nav_bar_left";
public static final String NAV_BAR_RIGHT = "sysui_nav_bar_right";

public static final String MENU_IME_ROTATE = "menu_ime";
public static final String BACK = "back";
public static final String HOME = "home";
public static final String RECENT = "recent";
public static final String NAVSPACE = "space";
public static final String CLIPBOARD = "clipboard";
public static final String HOME_HANDLE = "home_handle";
public static final String KEY = "key";
public static final String LEFT = "left";
public static final String RIGHT = "right";
public static final String CONTEXTUAL = "contextual";
public static final String IME_SWITCHER = "ime_switcher";

public static final String GRAVITY_SEPARATOR = ";";
public static final String BUTTON_SEPARATOR = ",";

public static final String SIZE_MOD_START = "[";
public static final String SIZE_MOD_END = "]";

public static final String KEY_CODE_START = "(";
public static final String KEY_IMAGE_DELIM = ":";
public static final String KEY_CODE_END = ")";
private static final String WEIGHT_SUFFIX = "W";
private static final String WEIGHT_CENTERED_SUFFIX = "WC";
private static final String ABSOLUTE_SUFFIX = "A";
private static final String ABSOLUTE_VERTICAL_CENTERED_SUFFIX = "C";

protected LayoutInflater mLayoutInflater;
protected LayoutInflater mLandscapeInflater;

protected FrameLayout mHorizontal;
protected FrameLayout mVertical;

@VisibleForTesting
SparseArray<ButtonDispatcher> mButtonDispatchers;
private String mCurrentLayout;

private View mLastPortrait;
private View mLastLandscape;

private boolean mIsVertical;
private boolean mAlternativeOrder;

private OverviewProxyService mOverviewProxyService;
private int mNavBarMode = NAV_BAR_MODE_3BUTTON;

public NavigationBarInflaterView(Context context, AttributeSet attrs) {
super(context, attrs);
createInflaters();
mOverviewProxyService = Dependency.get(OverviewProxyService.class);
mNavBarMode = Dependency.get(NavigationModeController.class).addListener(this);
}

@VisibleForTesting
void createInflaters() {
mLayoutInflater = LayoutInflater.from(mContext);
Configuration landscape = new Configuration();
landscape.setTo(mContext.getResources().getConfiguration());
landscape.orientation = Configuration.ORIENTATION_LANDSCAPE;
mLandscapeInflater = LayoutInflater.from(mContext.createConfigurationContext(landscape));
}

@Override
protected void onFinishInflate() {
super.onFinishInflate();
inflateChildren();
clearViews();
inflateLayout(getDefaultLayout());
}

private void inflateChildren() {
removeAllViews();
mHorizontal = (FrameLayout) mLayoutInflater.inflate(R.layout.navigation_layout,
this /* root */, false /* attachToRoot */);
addView(mHorizontal);
mVertical = (FrameLayout) mLayoutInflater.inflate(R.layout.navigation_layout_vertical,
this /* root */, false /* attachToRoot */);
addView(mVertical);
updateAlternativeOrder();
}

protected String getDefaultLayout() {
final int defaultResource = QuickStepContract.isGesturalMode(mNavBarMode)
? R.string.config_navBarLayoutHandle
: mOverviewProxyService.shouldShowSwipeUpUI()
? R.string.config_navBarLayoutQuickstep
: R.string.config_navBarLayout;
return getContext().getString(defaultResource);
}

```

在通过增加日志打印  
发现 默认布局 getDefaultLayout() 获取的是 R.string.config_navBarLayout 的值  
接下来就看这个字符串关于布局的相关代码

```
<string >left[.5W],back[1WC];home;recent[1WC],right[.5W]</string>

```

通过 left right 和; 来区分布局然后解析字符串加载相对于布局  
状态栏布局加载

```
protected void inflateLayout(String newLayout) {
mCurrentLayout = newLayout;
if (newLayout == null) {
newLayout = getDefaultLayout();
}
String[] sets = newLayout.split(GRAVITY_SEPARATOR, 3);
if (sets.length != 3) {
Log.d(TAG, "Invalid layout.");
newLayout = getDefaultLayout();
sets = newLayout.split(GRAVITY_SEPARATOR, 3);
}
String[] start = sets[0].split(BUTTON_SEPARATOR);
String[] center = sets[1].split(BUTTON_SEPARATOR);
String[] end = sets[2].split(BUTTON_SEPARATOR);
// Inflate these in start to end order or accessibility traversal will be messed up.
inflateButtons(start, mHorizontal.findViewById(R.id.ends_group),
false /* landscape */, true /* start */);
inflateButtons(start, mVertical.findViewById(R.id.ends_group),
true /* landscape */, true /* start */);

inflateButtons(center, mHorizontal.findViewById(R.id.center_group),
false /* landscape */, false /* start */);
inflateButtons(center, mVertical.findViewById(R.id.center_group),
true /* landscape */, false /* start */);

addGravitySpacer(mHorizontal.findViewById(R.id.ends_group));
addGravitySpacer(mVertical.findViewById(R.id.ends_group));

inflateButtons(end, mHorizontal.findViewById(R.id.ends_group),
false /* landscape */, false /* start */);
inflateButtons(end, mVertical.findViewById(R.id.ends_group),
true /* landscape */, false /* start */);

updateButtonDispatchersCurrentView();
}

```

这里会根据 navLayoutSetting 来加载 导航栏图标 所以增加虚拟按键也是从这里开始，用字符串对应一个布局文件，在 inflateButton 里添加布局

```
@Nullable
protected View inflateButton(String buttonSpec, ViewGroup parent, boolean landscape,
boolean start) {
LayoutInflater inflater = landscape ? mLandscapeInflater : mLayoutInflater;
View v = createView(buttonSpec, parent, inflater);
if (v == null) return null;
v = applySize(v, buttonSpec, landscape, start);
parent.addView(v);
addToDispatchers(v);
View lastView = landscape ? mLastLandscape : mLastPortrait;
View accessibilityView = v;
if (v instanceof ReverseRelativeLayout) {
accessibilityView = ((ReverseRelativeLayout) v).getChildAt(0);
}
if (lastView != null) {
accessibilityView.setAccessibilityTraversalAfter(lastView.getId());
}
if (landscape) {
mLastLandscape = accessibilityView;
} else {
mLastPortrait = accessibilityView;
}
return v;
}
// 计算导航栏图标的大小
private View applySize(View v, String buttonSpec, boolean landscape, boolean start) {
String sizeStr = extractSize(buttonSpec);
if (sizeStr == null) return v;

if (sizeStr.contains(WEIGHT_SUFFIX) || sizeStr.contains(ABSOLUTE_SUFFIX)) {
// To support gravity, wrap in RelativeLayout and apply gravity to it.
// Children wanting to use gravity must be smaller then the frame.
ReverseRelativeLayout frame = new ReverseRelativeLayout(mContext);
LayoutParams childParams = new LayoutParams(v.getLayoutParams());

// Compute gravity to apply
int gravity = (landscape) ? (start ? Gravity.TOP : Gravity.BOTTOM)
: (start ? Gravity.START : Gravity.END);
if (sizeStr.endsWith(WEIGHT_CENTERED_SUFFIX)) {
gravity = Gravity.CENTER;
} else if (sizeStr.endsWith(ABSOLUTE_VERTICAL_CENTERED_SUFFIX)) {
gravity = Gravity.CENTER_VERTICAL;
}

// Set default gravity, flipped if needed in reversed layouts (270 RTL and 90 LTR)
frame.setDefaultGravity(gravity);
frame.setGravity(gravity); // Apply gravity to root

frame.addView(v, childParams);

```

// 根据 config_navBarLayout_right 的值 来计算每个图标的宽度 如果为 W 或 WC 则表示为 weight 如果为 AC 则表示是像素值

```
if (sizeStr.contains(WEIGHT_SUFFIX)) {
// Use weighting to set the width of the frame
float weight = Float.parseFloat(
sizeStr.substring(0, sizeStr.indexOf(WEIGHT_SUFFIX)));
frame.setLayoutParams(new LinearLayout.LayoutParams(0, MATCH_PARENT, weight));
} else {
int width = (int) convertDpToPx(mContext,
Float.parseFloat(sizeStr.substring(0, sizeStr.indexOf(ABSOLUTE_SUFFIX))));
frame.setLayoutParams(new LinearLayout.LayoutParams(width, MATCH_PARENT));
}

// Ensure ripples can be drawn outside bounds
frame.setClipChildren(false);
frame.setClipToPadding(false);

return frame;
}

float size = Float.parseFloat(sizeStr);
ViewGroup.LayoutParams params = v.getLayoutParams();
params.width = (int) (params.width * size);
return v;
}

```

如果需要添加自定义图标  
可以在 config_navBarLayout_right 添加  
如:

```
<string >back[70AC],home[70AC],recent[70AC];hide;bluetooth[50AC],volume[50AC],brightness[50AC],keyboard[50AC],wifi[50AC],clock[50AC],battery[15AC]</string>

```
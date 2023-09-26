> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124774920)

### 1. 概述

在 11.0 12.0 定制化产品开发中国的 [app 开发](https://so.csdn.net/so/search?q=app%E5%BC%80%E5%8F%91&spm=1001.2101.3001.7020)中，沉浸式状态栏也是常有的样式，但是设置沉浸式状态栏后，会导致状态栏和导航栏会有灰色蒙层的问题存在

### 2. 沉浸式状态栏导致导航栏状态栏灰色蒙层的解决方案的核心类

```
framework/base/core/java/com/android/internal/policy/DecorView.java

```

### 3. 沉浸式状态栏导致导航栏状态栏灰色蒙层的解决方案的核心功能分析和实现

解决方案:  
DecorView 是整个 Window 界面的最顶层 View，它只有一个子元素为 LinearLayout。代表整个 Window 界面，包含通知栏，标题栏，内容显示栏三块区域。

Android7.0 以下，DecorView 是 PhoneWindow 的内部类，而在 7.0 以上，是一个单独的类，并且有新的属性和方法。新的属性如：mSemiTransparentBarColor，看字面意思应该就是我们要找的，我们对它进行跟踪

接下来查看源码 DecorView.java  
路径：framework/base/core/java/com/android/internal/policy/DecorView.java

```
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {
      private static final String TAG = "DecorView";
  
      private static final boolean DEBUG_MEASURE = false;
  
      private static final boolean SWEEP_OPEN_MENU = false;
  
      // The height of a window which has focus in DIP.
      public static final int DECOR_SHADOW_FOCUSED_HEIGHT_IN_DIP = 20;
      // The height of a window which has not in DIP.
      public static final int DECOR_SHADOW_UNFOCUSED_HEIGHT_IN_DIP = 5;
  
      private static final int SCRIM_LIGHT = 0xe6ffffff; // 90% white
  
      public static final ColorViewAttributes STATUS_BAR_COLOR_VIEW_ATTRIBUTES =
              new ColorViewAttributes(SYSTEM_UI_FLAG_FULLSCREEN, FLAG_TRANSLUCENT_STATUS,
                      Gravity.TOP, Gravity.LEFT, Gravity.RIGHT,
                      Window.STATUS_BAR_BACKGROUND_TRANSITION_NAME,
                      com.android.internal.R.id.statusBarBackground,
                      FLAG_FULLSCREEN, ITYPE_STATUS_BAR);
  
      public static final ColorViewAttributes NAVIGATION_BAR_COLOR_VIEW_ATTRIBUTES =
              new ColorViewAttributes(
                      SYSTEM_UI_FLAG_HIDE_NAVIGATION, FLAG_TRANSLUCENT_NAVIGATION,
                      Gravity.BOTTOM, Gravity.RIGHT, Gravity.LEFT,
                      Window.NAVIGATION_BAR_BACKGROUND_TRANSITION_NAME,
                      com.android.internal.R.id.navigationBarBackground,
                      0 /* hideWindowFlag */, ITYPE_NAVIGATION_BAR);
 DecorView(Context context, int featureId, PhoneWindow window,
            WindowManager.LayoutParams params) {
        super(context);
        mFeatureId = featureId;

        mShowInterpolator = AnimationUtils.loadInterpolator(context,
                android.R.interpolator.linear_out_slow_in);
        mHideInterpolator = AnimationUtils.loadInterpolator(context,
                android.R.interpolator.fast_out_linear_in);

        mBarEnterExitDuration = context.getResources().getInteger(
                R.integer.dock_enter_exit_duration);
        mForceWindowDrawsBarBackgrounds = context.getResources().getBoolean(
                R.bool.config_forceWindowDrawsStatusBarBackground)
                && context.getApplicationInfo().targetSdkVersion >= N;
        mSemiTransparentBarColor = context.getResources().getColor(
                R.color.system_bar_background_semi_transparent, null /* theme */);

        updateAvailableWidth();

        setWindow(window);

        updateLogTag(params);

        mResizeShadowSize = context.getResources().getDimensionPixelSize(
                R.dimen.resize_shadow_size);
        initResizingPaints();

        mLegacyNavigationBarBackgroundPaint.setColor(Color.BLACK);
    }
    @Override
      public void onDraw(Canvas c) {
          super.onDraw(c);
  
          mBackgroundFallback.draw(this, mContentRoot, c, mWindow.mContentParent,
                  mStatusColorViewState.view, mNavigationColorViewState.view);
      }
  
      @Override
      public boolean dispatchKeyEvent(KeyEvent event) {
          final int keyCode = event.getKeyCode();
          final int action = event.getAction();
          final boolean isDown = action == KeyEvent.ACTION_DOWN;
  
          if (isDown && (event.getRepeatCount() == 0)) {
              // First handle chording of panel key: if a panel key is held
              // but not released, try to execute a shortcut in it.
              if ((mWindow.mPanelChordingKey > 0) && (mWindow.mPanelChordingKey != keyCode)) {
                  boolean handled = dispatchKeyShortcutEvent(event);
                  if (handled) {
                      return true;
                  }
              }
  
              // If a panel is open, perform a shortcut on it without the
              // chorded panel key
              if ((mWindow.mPreparedPanel != null) && mWindow.mPreparedPanel.isOpen) {
                  if (mWindow.performPanelShortcut(mWindow.mPreparedPanel, keyCode, event, 0)) {
                      return true;
                  }
              }
          }
  
          if (!mWindow.isDestroyed()) {
              final Window.Callback cb = mWindow.getCallback();
              final boolean handled = cb != null && mFeatureId < 0 ? cb.dispatchKeyEvent(event)
                      : super.dispatchKeyEvent(event);
              if (handled) {
                  return true;
              }
          }
  
          return isDown ? mWindow.onKeyDown(mFeatureId, event.getKeyCode(), event)
                  : mWindow.onKeyUp(mFeatureId, event.getKeyCode(), event);
      }

```

在 DecorView 的构造方法中，初始化各种布局的参数，而  
mSemiTransparentBarColor 又是沉浸式状态栏设置的背景颜色，而它的初始值为：res 下的 system_bar_background_semi_transparent

而这就是灰色

```
private int calculateStatusBarColor() {
    return calculateBarColor(mWindow.getAttributes().flags, FLAG_TRANSLUCENT_STATUS,
            mSemiTransparentBarColor, mWindow.mStatusBarColor,
            getWindowSystemUiVisibility(), SYSTEM_UI_FLAG_LIGHT_STATUS_BAR,
            mWindow.mEnsureStatusBarContrastWhenTransparent);
}

private int calculateNavigationBarColor() {
    return calculateBarColor(mWindow.getAttributes().flags, FLAG_TRANSLUCENT_NAVIGATION,
            mSemiTransparentBarColor, mWindow.mNavigationBarColor,
            getWindowSystemUiVisibility(), SYSTEM_UI_FLAG_LIGHT_NAVIGATION_BAR,
            mWindow.mEnsureNavigationBarContrastWhenTransparent
                    && getContext().getResources().getBoolean(R.bool.config_navBarNeedsScrim));
}

```

而通过上述方法 calculateStatusBarColor() 和 calculateNavigationBarColor() 发现  
而在设置沉浸式时，会对状态栏导航栏的背景颜色做成改变 也就是设置 mSemiTransparentBarColor 为背景

所以修改就在这里

```
 DecorView(Context context, int featureId, PhoneWindow window,
            WindowManager.LayoutParams params) {
        super(context);
        mFeatureId = featureId;

        mShowInterpolator = AnimationUtils.loadInterpolator(context,
                android.R.interpolator.linear_out_slow_in);
        mHideInterpolator = AnimationUtils.loadInterpolator(context,
                android.R.interpolator.fast_out_linear_in);

        mBarEnterExitDuration = context.getResources().getInteger(
                R.integer.dock_enter_exit_duration);
        mForceWindowDrawsBarBackgrounds = context.getResources().getBoolean(
                R.bool.config_forceWindowDrawsStatusBarBackground)
                && context.getApplicationInfo().targetSdkVersion >= N;

        -  mSemiTransparentBarColor = context.getResources().getColor(
                R.color.system_bar_background_semi_transparent, null /* theme */);
        + mSemiTransparentBarColor = context.getResources().getColor(
                R.color.resize_shadow_end_color, null /* theme */);

        updateAvailableWidth();

        setWindow(window);

        updateLogTag(params);

        mResizeShadowSize = context.getResources().getDimensionPixelSize(
                R.dimen.resize_shadow_size);
        initResizingPaints();

        mLegacyNavigationBarBackgroundPaint.setColor(Color.BLACK);
    }

```

mSemiTransparentBarColor 修改为透明色就可以了
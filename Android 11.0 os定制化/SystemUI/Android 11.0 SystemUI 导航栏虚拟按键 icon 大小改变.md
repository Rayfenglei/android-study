> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127167343)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.SystemUI 导航栏虚拟按键 icon 大小改变的核心类](#t1)

[3.SystemUI 导航栏虚拟按键 icon 大小改变的核心功能分析和实现](#t2)

 [3.1 关于加载导航栏的相关代码分析](#t3)

[3.2NavigationBarView 的相关源码分析](#t4)

1. 概述
-----

  在 11.0 的系统产品开发中，对应 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 的 NavigationBar 导航栏的定制化开发也是比较常见的功能，由于产品的小屏幕的，所以对于屏幕导航栏的三个虚拟按键需要做定制，产品需求要求对导航栏虚拟按键的宽高做调整，放大虚拟按键宽高，所以需要查看关键源码然后修改就好了

2.SystemUI 导航栏虚拟按键 icon 大小改变的核心类
--------------------------------

```
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarFragment.java
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarView.java
```

3.SystemUI 导航栏虚拟按键 icon 大小改变的核心功能分析和实现
--------------------------------------

###      3.1 关于加载导航栏的相关代码分析

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

  在上述代码中 onCreateView(LayoutInflater inflater, @Nullable ViewGroup container,  
              Bundle savedInstanceState) 中加载代码布局代码就是 navigation_bar.xml 布局文件  
而在 onViewCreated(View view, @Nullable Bundle savedInstanceState) 中，也是设置 mNavigationBarView  
的相关参数，所以需要具体看 NavigationBarView 的相关源码

```
    private void prepareNavigationBarView() {
         mNavigationBarView.reorient();
 
         ButtonDispatcher recentsButton = mNavigationBarView.getRecentsButton();
         recentsButton.setOnClickListener(this::onRecentsClick);
         recentsButton.setOnTouchListener(this::onRecentsTouch);
         recentsButton.setLongClickable(true);
         recentsButton.setOnLongClickListener(this::onLongPressBackRecents);
 
         ButtonDispatcher backButton = mNavigationBarView.getBackButton();
         backButton.setLongClickable(true);
 
         ButtonDispatcher homeButton = mNavigationBarView.getHomeButton();
         homeButton.setOnTouchListener(this::onHomeTouch);
          homeButton.setOnLongClickListener(this::onHomeLongClick);
  
          ButtonDispatcher accessibilityButton = mNavigationBarView.getAccessibilityButton();
          accessibilityButton.setOnClickListener(this::onAccessibilityClick);
          accessibilityButton.setOnLongClickListener(this::onAccessibilityLongClick);
          updateAccessibilityServicesState(mAccessibilityManager);
  
          updateScreenPinningGestures();
      }
```

在 NavigationBarFragment.java 中的 prepareNavigationBarView() 源码中可以看到，backButton recentsButton homeButton 等到导航栏三大键都是调用  
NavigationBarView 的对应方法的，所以具体的 icon 设置都是在 NavigationBarView 中定义的，所以需要看 NavigationBarView 的相关源码

### 3.2NavigationBarView 的相关源码分析

```
       public ButtonDispatcher getRecentsButton() {
          return mButtonDispatchers.get(R.id.recent_apps);
      }
  
      public ButtonDispatcher getBackButton() {
          return mButtonDispatchers.get(R.id.back);
      }
  
      public ButtonDispatcher getHomeButton() {
          return mButtonDispatchers.get(R.id.home);
      }
```

在 NavigationBarFragment.java 中通过上面三个方法调用导航栏的虚拟按键，然后操作这三个键，来实现对他们的点击事件的处理

```
      @Override
      public void setLayoutDirection(int layoutDirection) {
          reloadNavIcons();
          super.setLayoutDirection(layoutDirection);
      }
      private void reloadNavIcons() {
          updateIcons(Configuration.EMPTY);
      }
 
      private void updateIcons(Configuration oldConfig) {
          final boolean orientationChange = oldConfig.orientation != mConfiguration.orientation;
          final boolean densityChange = oldConfig.densityDpi != mConfiguration.densityDpi;
          final boolean dirChange = oldConfig.getLayoutDirection() != mConfiguration.getLayoutDirection();
  
          if (orientationChange || densityChange) {
              mDockedIcon = getDrawable(R.drawable.ic_sysbar_docked);
              mHomeDefaultIcon = getHomeDrawable();
          }
          if (densityChange || dirChange) {
              mRecentIcon = getDrawable(R.drawable.ic_sysbar_recent);
              mContextualButtonGroup.updateIcons();
          }
          if (orientationChange || densityChange || dirChange) {
              mBackIcon = getBackDrawable();
          }
      }
```

 在 NavigationBarView 的布局是在 setLayoutDirection(int layoutDirection) 调用 reloadNavIcons()，最终调用 updateIcons(Configuration oldConfig) 中来加载导航栏的  
从上述代码可以发现，home 键的图标是 getHomeDrawable() 而 recents 键图标就是 getDrawable(R.drawable.ic_sysbar_recent)，而 back 键就是 getBackDrawable()

```
     public KeyButtonDrawable getBackDrawable() {
          KeyButtonDrawable drawable = getDrawable(getBackDrawableRes());
          orientBackButton(drawable);
          return drawable;
      }
  
      public @DrawableRes int getBackDrawableRes() {
          return chooseNavigationIconDrawableRes(R.drawable.ic_sysbar_back,
                  R.drawable.ic_sysbar_back_quick_step);
      }
  
      public KeyButtonDrawable getHomeDrawable() {
          final boolean quickStepEnabled = mOverviewProxyService.shouldShowSwipeUpUI();
          KeyButtonDrawable drawable = quickStepEnabled
                  ? getDrawable(R.drawable.ic_sysbar_home_quick_step)
                  : getDrawable(R.drawable.ic_sysbar_home);
          orientHomeButton(drawable);
          return drawable;
      }
```

通过上述分析，所以 back 键就是 ic_sysbar_back.xml 布局，而 home 键就是 ic_sysbar_home 布局 recent 键就是 ic_sysbar_recent 布局  
接下来看下 res/drawable 下的 icon 的 xml 布局  
例如 ic_sysbar_back.xml 布局如下：

```
<vector xmlns:android="http://schemas.android.com/apk/res/android"
     android:width="28dp"
     android:height="28dp"
     android:autoMirrored="true"
     android:viewportWidth="28"
     android:viewportHeight="28">
 
     <path
         android:fillColor="?attr/singleToneColor"
         android:pathData="M6.49,14.86c-0.66-0.39-0.66-1.34,0-1.73l6.02-3.53l5.89-3.46C19.11,5.73,20,6.26,20,7.1V14v6.9 c0,0.84-0.89,1.37-1.6,0.95l-5.89-3.46L6.49,14.86z" />
 </vector>
```

只需要修改 width 和 height 属性值就可以了  
修改为:

```
<vector xmlns:android="http://schemas.android.com/apk/res/android"
     android:width="42dp"
     android:height="42dp"
     android:autoMirrored="true"
     android:viewportWidth="42"
     android:viewportHeight="42">
 
     <path
         android:fillColor="?attr/singleToneColor"
         android:pathData="M6.49,14.86c-0.66-0.39-0.66-1.34,0-1.73l6.02-3.53l5.89-3.46C19.11,5.73,20,6.26,20,7.1V14v6.9 c0,0.84-0.89,1.37-1.6,0.95l-5.89-3.46L6.49,14.86z" />
 </vector>
```

其他的 home 键和 recent 键按照这种方式修改就好了 就可以完成功能了
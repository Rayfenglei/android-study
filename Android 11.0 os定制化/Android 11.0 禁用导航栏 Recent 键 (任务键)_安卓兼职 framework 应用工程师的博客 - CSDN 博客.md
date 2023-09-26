> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127434644)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 禁用导航栏 Recent 键 (任务键) 的核心类](#2.%E7%A6%81%E7%94%A8%E5%AF%BC%E8%88%AA%E6%A0%8FRecent%E9%94%AE%28%E4%BB%BB%E5%8A%A1%E9%94%AE%29%E7%9A%84%E6%A0%B8%E5%BF%83%E7%B1%BB)

[3. 禁用导航栏 Recent 键 (任务键) 的核心功能实现和分析](#3.%E7%A6%81%E7%94%A8%E5%AF%BC%E8%88%AA%E6%A0%8FRecent%E9%94%AE%28%E4%BB%BB%E5%8A%A1%E9%94%AE%29%E7%9A%84%E6%A0%B8%E5%BF%83%E5%8A%9F%E8%83%BD%E5%AE%9E%E7%8E%B0%E5%92%8C%E5%88%86%E6%9E%90)

 [3.1 NavigationBarFragment.java 中 recent 键的相关功能分析](#t3)

[3.2 Recents.java 中关于最近任务的相关方法](#t4)

[3.3 RecentsImplementation 的相关最近任务的方法分析](#t5)

[3.4 OverviewProxyRecentsImpl.java 中关于最近任务方法分析](#t6)

1. 概述
-----

  在 11.0 的系统产品开发中，对于 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 的导航栏定制功能中也是常有的功能，最近产品需求需要禁用 recent 最近任务键的功能，所以需要在 recent 的流程中禁用 recent 的相关功能

2. 禁用导航栏 Recent 键 (任务键) 的核心类
----------------------------

```
核心代码如下:
frameworks\base\packages\SystemUI\src\com\android\systemui\statusbar\phone\NavigationBarFragment.java
frameworks\base\packages\SystemUI\src\com\android\systemui\recents\Recents.java
frameworks/base/packages/SystemUI/src/com/android/systemui/recents/RecentsImplementation.java
frameworks/base/packages/SystemUI/src/com/android/systemui/recents/OverviewProxyRecentsImpl.java
```

3. 禁用导航栏 Recent 键 (任务键) 的核心功能实现和分析
----------------------------------

###   3.1 NavigationBarFragment.java 中 recent 键的相关功能分析

```
   public class NavigationBarFragment extends LifecycleFragment implements Callbacks,
          NavigationModeController.ModeChangedListener, DisplayManager.DisplayListener {
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
 ....
         
     }
```

 在 NavigationBarFragment 的 onViewCreated（）方法中，开始构造导航栏的相关初始化参数  
而在 prepareNavigationBarView(); 中来构建导航栏虚拟按键的点击事件的处理，接下来看下  
prepareNavigationBarView(); 的点击事件相关源码

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

在 NavigationBarFragment 的 prepareNavigationBarView() 中的虚拟按键 recentsButton 就是最近任务 recent 键  
所以 recent 的点击事件就是 onRecentsClick 接下来看下 onRecentsClick 的相关 recent 的相关  
调用分析功能

```
 private void onRecentsClick(View v) {
          if (LatencyTracker.isEnabled(getContext())) {
              LatencyTracker.getInstance(getContext()).onActionStart(
                      LatencyTracker.ACTION_TOGGLE_RECENTS);
          }
          mStatusBarLazy.get().awakenDreams();
          mCommandQueue.toggleRecentApps();
  }
```

在 NavigationBarFragment 的 onRecentsClick 的方法中可以看到 mCommandQueue.toggleRecentApps(); 就是  
获取最近 app 列表的相关方法，而 mCommandQueue 就是 CommandQueue 的相关方法，  
而在 Recents.java 中实现 CommandQueue 的接口，所以接下来看下而在 Recents.java 中实现 CommandQueue 的相关方法

### 3.2 Recents.java 中关于最近任务的相关方法

```
   public class Recents extends SystemUI implements CommandQueue.Callbacks {
 
     private final RecentsImplementation mImpl;
     private final CommandQueue mCommandQueue;
 
     public Recents(Context context, RecentsImplementation impl, CommandQueue commandQueue) {
         super(context);
         mImpl = impl;
         mCommandQueue = commandQueue;
     }
 
     @Override
     public void start() {
         mCommandQueue.addCallback(this);
         mImpl.onStart(mContext);
     }
 
     @Override
     public void onBootCompleted() {
         mImpl.onBootCompleted();
     }
     @Override
     public void toggleRecentApps() {
         // Ensure the device has been provisioned before allowing the user to interact with
         // recents
         if (!isUserSetup()) {
             return;
          }
  
          mImpl.toggleRecentApps();
      }
```

通过上述 Recents.java 的相关方法分析可以看出 toggleRecentApps() 中其实是调用的是 RecentsImplementation  
的 toggleRecentApps(); 来实现最近任务列表查询显示的，接下来看下 RecentsImplementation 的 toggleRecentApps();  
的相关方法

### 3.3 RecentsImplementation 的相关最近任务的方法分析

```
public interface RecentsImplementation {
      default void onStart(Context context) {}
      default void onBootCompleted() {}
      default void onAppTransitionFinished() {}
      default void onConfigurationChanged(Configuration newConfig) {}
  
      default void preloadRecentApps() {}
      default void cancelPreloadRecentApps() {}
      default void showRecentApps(boolean triggeredFromAltTab) {}
      default void hideRecentApps(boolean triggeredFromAltTab, boolean triggeredFromHomeKey) {}
      default void toggleRecentApps() {}
      default void growRecents() {}
      default boolean splitPrimaryTask(int stackCreateMode, Rect initialBounds,
              int metricsDockAction) {
          return false;
      }
  
      default void dump(PrintWriter pw) {}
  }
```

### 3.4 OverviewProxyRecentsImpl.java 中关于最近任务方法分析

 在 systemUI 中，OverviewProxyRecentsImpl.java 就是对 RecentsImplementation 的具体实现类的  
所以需要具体看 toggleRecentApps() 的相关方法

```
public class OverviewProxyRecentsImpl implements RecentsImplementation {
 
     private final static String TAG = "OverviewProxyRecentsImpl";
     @Nullable
     private final Lazy<StatusBar> mStatusBarLazy;
     private final Optional<Divider> mDividerOptional;
 
     private Context mContext;
     private Handler mHandler;
     private TrustManager mTrustManager;
     private OverviewProxyService mOverviewProxyService;
 
     @SuppressWarnings("OptionalUsedAsFieldOrParameterType")
     @Inject
     public OverviewProxyRecentsImpl(Optional<Lazy<StatusBar>> statusBarLazy,
             Optional<Divider> dividerOptional) {
         mStatusBarLazy = statusBarLazy.orElse(null);
         mDividerOptional = dividerOptional;
     }
 
     @Override
     public void onStart(Context context) {
         mContext = context;
         mHandler = new Handler();
         mTrustManager = (TrustManager) context.getSystemService(Context.TRUST_SERVICE);
         mOverviewProxyService = Dependency.get(OverviewProxyService.class);
     }
     @Override
      public void toggleRecentApps() {
          // If connected to launcher service, let it handle the toggle logic
          IOverviewProxy overviewProxy = mOverviewProxyService.getProxy();
          if (overviewProxy != null) {
              final Runnable toggleRecents = () -> {
                  try {
                      if (mOverviewProxyService.getProxy() != null) {
                          mOverviewProxyService.getProxy().onOverviewToggle();
                          mOverviewProxyService.notifyToggleRecentApps();
                      }
                  } catch (RemoteException e) {
                      Log.e(TAG, "Cannot send toggle recents through proxy service.", e);
                  }
              };
              // Preload only if device for current user is unlocked
              if (mStatusBarLazy != null && mStatusBarLazy.get().isKeyguardShowing()) {
                  mStatusBarLazy.get().executeRunnableDismissingKeyguard(() -> {
                          // Flush trustmanager before checking device locked per user
                          mTrustManager.reportKeyguardShowingChanged();
                          mHandler.post(toggleRecents);
                      }, null,  true /* dismissShade */, false /* afterKeyguardGone */,
                      true /* deferred */);
              } else {
                  toggleRecents.run();
              }
              return;
          } else {
              // Do nothing
          }
      }
```

从 toggleRecentApps() 就可以看出这里是具体实现 recent 任务列表的功能，所以禁用 recent  
功能，就需要注释掉这里的功能即可  
具体修改为:

```
     @Override
      public void toggleRecentApps() {
          // If connected to launcher service, let it handle the toggle logic
         /* IOverviewProxy overviewProxy = mOverviewProxyService.getProxy();
          if (overviewProxy != null) {
              final Runnable toggleRecents = () -> {
                  try {
                      if (mOverviewProxyService.getProxy() != null) {
                          mOverviewProxyService.getProxy().onOverviewToggle();
                          mOverviewProxyService.notifyToggleRecentApps();
                      }
                  } catch (RemoteException e) {
                      Log.e(TAG, "Cannot send toggle recents through proxy service.", e);
                  }
              };
              // Preload only if device for current user is unlocked
              if (mStatusBarLazy != null && mStatusBarLazy.get().isKeyguardShowing()) {
                  mStatusBarLazy.get().executeRunnableDismissingKeyguard(() -> {
                          // Flush trustmanager before checking device locked per user
                          mTrustManager.reportKeyguardShowingChanged();
                          mHandler.post(toggleRecents);
                      }, null,  true /* dismissShade */, false /* afterKeyguardGone */,
                      true /* deferred */);
              } else {
                  toggleRecents.run();
              }
              return;
          } else {
              // Do nothing
          }*/
      }
```
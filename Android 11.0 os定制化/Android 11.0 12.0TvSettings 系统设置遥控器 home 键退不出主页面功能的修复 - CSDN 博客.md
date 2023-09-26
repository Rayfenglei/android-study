> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126474310)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.TvSettings 系统设置遥控器 home 键退不出主页面功能的修复的核心代码](#t1)

[3.TvSettings 系统设置遥控器 home 键退不出主页面功能的修复的功能分析以及修复](#t2)

[3.1 MainSettings.java 代码分析](#t3)

[3.2 TvSettingsActivity.java 相关代码分析](#t4)

1. 概述
-----

在进行系统开发中，有些产品是 TvSettings 作为系统设置 而有些产品是以 Settings 作为系统设置，在定制 一款产品中，TvSettings 在退回到主页面时，点击 Home 键不能回到桌面，而 back 键可以退出主页面 所以这就要添加日志看是不是没收到 home 事件还是不响应 Home 按键事件了

2.TvSettings 系统设置遥控器 home 键退不出主页面功能的修复的核心代码
-------------------------------------------

```
   /packages/apps/TvSettings/Settings/src/com/android/tv/settings/TvSettingsActivity.java
  /packages/apps/TvSettings/Settings/src/com/android/tv/settings/MainSettings.java
```

3.TvSettings 系统设置遥控器 home 键退不出主页面功能的修复的功能分析以及修复
-----------------------------------------------

### 3.1 MainSettings.java 代码分析

```
    public class MainSettings extends TvSettingsActivity {
  
      @Override
      protected Fragment createSettingsFragment() {
          return FeatureFactory.getFactory(this).getSettingsFragmentProvider()
              .newSettingsFragment(MainFragment.class.getName(), null);
      }
  }
```

### 从代码可以看出 MainSettings 是实现了 TvSettingsActivity 的子类，主要是构建 MainFragment 作为  
系统设置的主页面加载一级菜单项，所以说在 TvSettignsActivity 添加监听事件就可以了

### 3.2 TvSettingsActivity.java 相关代码分析

```
public abstract class TvSettingsActivity extends Activity {
     private static final String TAG = "TvSettingsActivity";
 
     private static final String SETTINGS_FRAGMENT_TAG =
             "com.android.tv.settings.MainSettings.SETTINGS_FRAGMENT";
 
     @Override
     protected void onCreate(@Nullable Bundle savedInstanceState) {
         super.onCreate(savedInstanceState);
         if (savedInstanceState == null) {
 
             final Fragment fragment = createSettingsFragment();
             if (fragment == null) {
                 return;
             }
             if (FeatureFactory.getFactory(this).isTwoPanelLayout()) {
                 getFragmentManager().beginTransaction()
                         .setCustomAnimations(android.R.animator.fade_in,
                                 android.R.animator.fade_out)
                         .add(android.R.id.content, fragment, SETTINGS_FRAGMENT_TAG)
                         .commitNow();
                 return;
             }
 
             final ViewGroup root = findViewById(android.R.id.content);
             root.getViewTreeObserver().addOnPreDrawListener(
                     new ViewTreeObserver.OnPreDrawListener() {
                         @Override
                         public boolean onPreDraw() {
                             root.getViewTreeObserver().removeOnPreDrawListener(this);
                             final Scene scene = new Scene(root);
                             scene.setEnterAction(() -> {
                                 if (getFragmentManager().isStateSaved()
                                         || getFragmentManager().isDestroyed()) {
                                     Log.d(TAG, "Got torn down before adding fragment");
                                     return;
                                 }
                                 getFragmentManager().beginTransaction()
                                         .add(android.R.id.content, fragment,
                                                 SETTINGS_FRAGMENT_TAG)
                                         .commitNow();
                             });
 
                             final Slide slide = new Slide(Gravity.END);
                             slide.setSlideFraction(
                                     getResources().getDimension(R.dimen.lb_settings_pane_width)
                                             / root.getWidth());
                             TransitionManager.go(scene, slide);
 
                             // Skip the current draw, there's nothing in it
                             return false;
                         }
                     });
         }
     }
 
     @Override
     public void finish() {
         final Fragment fragment = getFragmentManager().findFragmentByTag(SETTINGS_FRAGMENT_TAG);
         if (FeatureFactory.getFactory(this).isTwoPanelLayout()) {
             super.finish();
             return;
          }
  
          if (isResumed() && fragment != null) {
              final ViewGroup root = findViewById(android.R.id.content);
              final Scene scene = new Scene(root);
              scene.setEnterAction(() -> getFragmentManager().beginTransaction()
                      .remove(fragment)
                      .commitNow());
              final Slide slide = new Slide(Gravity.END);
              slide.setSlideFraction(
                      getResources().getDimension(R.dimen.lb_settings_pane_width) / root.getWidth());
              slide.addListener(new Transition.TransitionListener() {
                  @Override
                  public void onTransitionStart(Transition transition) {
                      getWindow().setDimAmount(0);
                  }
  
                  @Override
                  public void onTransitionEnd(Transition transition) {
                      transition.removeListener(this);
                      TvSettingsActivity.super.finish();
                  }
  
                  @Override
                  public void onTransitionCancel(Transition transition) {
                  }
  
                  @Override
                  public void onTransitionPause(Transition transition) {
                  }
  
                  @Override
                  public void onTransitionResume(Transition transition) {
                  }
              });
              TransitionManager.go(scene, slide);
          } else {
              super.finish();
          }
      }
  
      protected abstract Fragment createSettingsFragment();
  
      private String getMetricsTag() {
          String tag = getClass().getName();
          if (tag.startsWith("com.android.tv.settings.")) {
              tag = tag.replace("com.android.tv.settings.", "");
          }
          return tag;
      }
  
      @Override
      public SharedPreferences getSharedPreferences(String name, int mode) {
          if (name.equals(getPackageName() + "_preferences")) {
              return new SharedPreferencesLogger(this, getMetricsTag(),
                      new MetricsFeatureProvider());
          }
          return super.getSharedPreferences(name, mode);
      }
  }
 
 
 
 
```

所以说在增加 back 和 home 键监听的广播 然后收到 home 键广播后 finish 掉主页面就可以了

具体修改如下:  
// 增加 Home 键监听广播  
 

```
 public class HomeKeyReceiver extends BroadcastReceiver {
    private static final String TAG = "HomeKeyReceiver";
    private static final String SYSTEM_DIALOG_REASON_KEY = "reason";
    private static final String SYSTEM_DIALOG_REASON_HOME_KEY = "homekey";
 
    @Override
    public void onReceive(Context context, Intent intent) {
        String action = intent.getAction();
        Log.e(TAG, "onReceive: action: " + action);
        if (action.equals(Intent.ACTION_CLOSE_SYSTEM_DIALOGS)) {
            // android.intent.action.CLOSE_SYSTEM_DIALOGS
            String reason = intent.getStringExtra(SYSTEM_DIALOG_REASON_KEY);
            Log.i(TAG, "reason: " + reason);
 
            if (SYSTEM_DIALOG_REASON_HOME_KEY.equals(reason)) {
                // 短按Home键 当home键时，finish掉TvSettingsActivity就可以了 这样就退出了TvSettings了
                Log.e(TAG, "homekey");
                finish();
            }else if (SYSTEM_DIALOG_REASON__KEY.equals(reason)) {
                Log.e(TAG, "reason");
            }
        }
    }
}
```

修改如下：

```
public abstract class TvSettingsActivity extends Activity {
     private static final String TAG = "TvSettingsActivity";
 
     private static final String SETTINGS_FRAGMENT_TAG =
             "com.android.tv.settings.MainSettings.SETTINGS_FRAGMENT";
 + private HomeKeyReceiver mHomeKeyReceiver = null;
     @Override
     protected void onCreate(@Nullable Bundle savedInstanceState) {
         super.onCreate(savedInstanceState);
         if (savedInstanceState == null) {
 
             final Fragment fragment = createSettingsFragment();
             if (fragment == null) {
                 return;
             }
             if (FeatureFactory.getFactory(this).isTwoPanelLayout()) {
                 getFragmentManager().beginTransaction()
                         .setCustomAnimations(android.R.animator.fade_in,
                                 android.R.animator.fade_out)
                         .add(android.R.id.content, fragment, SETTINGS_FRAGMENT_TAG)
                         .commitNow();
                 return;
             }
 
             final ViewGroup root = findViewById(android.R.id.content);
             root.getViewTreeObserver().addOnPreDrawListener(
                     new ViewTreeObserver.OnPreDrawListener() {
                         @Override
                         public boolean onPreDraw() {
                             root.getViewTreeObserver().removeOnPreDrawListener(this);
                             final Scene scene = new Scene(root);
                             scene.setEnterAction(() -> {
                                 if (getFragmentManager().isStateSaved()
                                         || getFragmentManager().isDestroyed()) {
                                     Log.d(TAG, "Got torn down before adding fragment");
                                     return;
                                 }
                                 getFragmentManager().beginTransaction()
                                         .add(android.R.id.content, fragment,
                                                 SETTINGS_FRAGMENT_TAG)
                                         .commitNow();
                             });
 
                             final Slide slide = new Slide(Gravity.END);
                             slide.setSlideFraction(
                                     getResources().getDimension(R.dimen.lb_settings_pane_width)
                                             / root.getWidth());
                             TransitionManager.go(scene, slide);
 
                             // Skip the current draw, there's nothing in it
                             return false;
                         }
                     });
         }
 
   // add code start
    mHomeKeyReceiver = new HomeKeyReceiver();
    IntentFilter homekeyFilter = new IntentFilter(Intent.ACTION_CLOSE_SYSTEM_DIALOGS);
    registerReceiver(mHomeKeyReceiver,homekeyFilter);
    // add code end
     }
 
 
 
     @Override
     public void finish() {
+ unregisterReceiver(mHomeKeyReceiver);
         final Fragment fragment = getFragmentManager().findFragmentByTag(SETTINGS_FRAGMENT_TAG);
         if (FeatureFactory.getFactory(this).isTwoPanelLayout()) {
             super.finish();
             return;
          }
  
          if (isResumed() && fragment != null) {
              final ViewGroup root = findViewById(android.R.id.content);
              final Scene scene = new Scene(root);
              scene.setEnterAction(() -> getFragmentManager().beginTransaction()
                      .remove(fragment)
                      .commitNow());
              final Slide slide = new Slide(Gravity.END);
              slide.setSlideFraction(
                      getResources().getDimension(R.dimen.lb_settings_pane_width) / root.getWidth());
              slide.addListener(new Transition.TransitionListener() {
                  @Override
                  public void onTransitionStart(Transition transition) {
                      getWindow().setDimAmount(0);
                  }
  
                  @Override
                  public void onTransitionEnd(Transition transition) {
                      transition.removeListener(this);
                      TvSettingsActivity.super.finish();
                  }
  
                  @Override
                  public void onTransitionCancel(Transition transition) {
                  }
  
                  @Override
                  public void onTransitionPause(Transition transition) {
                  }
  
                  @Override
                  public void onTransitionResume(Transition transition) {
                  }
              });
              TransitionManager.go(scene, slide);
          } else {
              super.finish();
          }
      }
  
      protected abstract Fragment createSettingsFragment();
  
      private String getMetricsTag() {
          String tag = getClass().getName();
          if (tag.startsWith("com.android.tv.settings.")) {
              tag = tag.replace("com.android.tv.settings.", "");
          }
          return tag;
      }
  
      @Override
      public SharedPreferences getSharedPreferences(String name, int mode) {
          if (name.equals(getPackageName() + "_preferences")) {
              return new SharedPreferencesLogger(this, getMetricsTag(),
                      new MetricsFeatureProvider());
          }
          return super.getSharedPreferences(name, mode);
      }
  }
 
 
```
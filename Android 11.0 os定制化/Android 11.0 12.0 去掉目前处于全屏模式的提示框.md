> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124774680)

### 1. 概述

在 11.0 12.0 系统开发中，在 app 设置为全屏状态时，第一次进入 app 时会有个 目前处于全屏的 对话框提醒用户现在处于全屏状态，提示已经进入全屏状态，但是对于这个功能，产品不一定会喜欢这个提示框，所以考虑掉要把这个提示框给去掉

### 2. 去掉目前处于全屏模式的提示框的核心类

```
frameworks\base\services\core\java\com\android\server\wm\ImmersiveModeConfirmation.java

```

### 3. 去掉目前处于全屏模式的提示框的核心功能分析和实现

功能分析 对于这种功能，其实主要是通过搜索看是在哪里弹窗的  
所以根据提示语开始在 framework/base 下搜索  
“目前处于全屏模式”  
接着继续搜索 immersive_cling_title 发现在  
R.layout.immersive_mode_cling 中引用了这个资源 接下来就来查找这个是哪里用到了

结果在 ImmersiveModeConfirmation.java 中找到了应用的地方  
路径: frameworks\base\services\core\java\com\android\server\wm\ImmersiveModeConfirmation.java

```
public class ImmersiveModeConfirmation {
     private static final String TAG = "ImmersiveModeConfirmation";
     private static final boolean DEBUG = false;
     private static final boolean DEBUG_SHOW_EVERY_TIME = false; // super annoying, use with caution
     private static final String CONFIRMED = "confirmed";
 
     private static boolean sConfirmed;
 
     private final Context mContext;
     private final H mHandler;
     private final long mShowDelayMs;
     private final long mPanicThresholdMs;
     private final IBinder mWindowToken = new Binder();
 
     private ClingWindowView mClingWindow;
     private long mPanicTime;
     private WindowManager mWindowManager;
     // Local copy of vr mode enabled state, to avoid calling into VrManager with
     // the lock held.
     private boolean mVrModeEnabled;
     private int mLockTaskState = LOCK_TASK_MODE_NONE;
 
     ImmersiveModeConfirmation(Context context, Looper looper, boolean vrModeEnabled) {
         final Display display = context.getDisplay();
         final Context uiContext = ActivityThread.currentActivityThread().getSystemUiContext();
         mContext = display.getDisplayId() == DEFAULT_DISPLAY
                 ? uiContext : uiContext.createDisplayContext(display);
         mHandler = new H(looper);
         mShowDelayMs = getNavBarExitDuration() * 3;
         mPanicThresholdMs = context.getResources()
                 .getInteger(R.integer.config_immersive_mode_confirmation_panic);
         mVrModeEnabled = vrModeEnabled;
     }
 
     private long getNavBarExitDuration() {
         Animation exit = AnimationUtils.loadAnimation(mContext, R.anim.dock_bottom_exit);
         return exit != null ? exit.getDuration() : 0;
      }
  
      static boolean loadSetting(int currentUserId, Context context) {
          final boolean wasConfirmed = sConfirmed;
          sConfirmed = false;
          if (DEBUG) Slog.d(TAG, String.format("loadSetting() currentUserId=%d", currentUserId));
          String value = null;
          try {
              value = Settings.Secure.getStringForUser(context.getContentResolver(),
                      Settings.Secure.IMMERSIVE_MODE_CONFIRMATIONS,
                      UserHandle.USER_CURRENT);
              sConfirmed = CONFIRMED.equals(value);
              if (DEBUG) Slog.d(TAG, "Loaded sConfirmed=" + sConfirmed);
          } catch (Throwable t) {
              Slog.w(TAG, "Error loading confirmations, value=" + value, t);
          }
          return sConfirmed != wasConfirmed;
      }
  
      private static void saveSetting(Context context) {
          if (DEBUG) Slog.d(TAG, "saveSetting()");
          try {
              final String value = sConfirmed ? CONFIRMED : null;
              Settings.Secure.putStringForUser(context.getContentResolver(),
                      Settings.Secure.IMMERSIVE_MODE_CONFIRMATIONS,
                      value,
                      UserHandle.USER_CURRENT);
              if (DEBUG) Slog.d(TAG, "Saved value=" + value);
          } catch (Throwable t) {
              Slog.w(TAG, "Error saving confirmations, sConfirmed=" + sConfirmed, t);
          }
      }
  
      void immersiveModeChangedLw(String pkg, boolean isImmersiveMode,
              boolean userSetupComplete, boolean navBarEmpty) {
          mHandler.removeMessages(H.SHOW);
          if (isImmersiveMode) {
              final boolean disabled = PolicyControl.disableImmersiveConfirmation(pkg);
              if (DEBUG) Slog.d(TAG, String.format("immersiveModeChanged() disabled=%s sConfirmed=%s",
                      disabled, sConfirmed));
              if (!disabled
                      && (DEBUG_SHOW_EVERY_TIME || !sConfirmed)
                      && userSetupComplete
                      && !mVrModeEnabled
                      && !navBarEmpty
                      && !UserManager.isDeviceInDemoMode(mContext)
                      && (mLockTaskState != LOCK_TASK_MODE_LOCKED)) {
                  mHandler.sendEmptyMessageDelayed(H.SHOW, mShowDelayMs);
              }
          } else {
              mHandler.sendEmptyMessage(H.HIDE);
          }
      }
  
      boolean onPowerKeyDown(boolean isScreenOn, long time, boolean inImmersiveMode,
              boolean navBarEmpty) {
          if (!isScreenOn && (time - mPanicTime < mPanicThresholdMs)) {
              // turning the screen back on within the panic threshold
              return mClingWindow == null;
          }
          if (isScreenOn && inImmersiveMode && !navBarEmpty) {
              // turning the screen off, remember if we were in immersive mode
              mPanicTime = time;
          } else {
              mPanicTime = 0;
          }
          return false;
      }
  
      void confirmCurrentPrompt() {
          if (mClingWindow != null) {
              if (DEBUG) Slog.d(TAG, "confirmCurrentPrompt()");
              mHandler.post(mConfirm);
          }
      }
  
      private void handleHide() {
          if (mClingWindow != null) {
              if (DEBUG) Slog.d(TAG, "Hiding immersive mode confirmation");
              getWindowManager().removeView(mClingWindow);
              mClingWindow = null;
          }
      }
  
      private WindowManager.LayoutParams getClingWindowLayoutParams() {
          final WindowManager.LayoutParams lp = new WindowManager.LayoutParams(
                  ViewGroup.LayoutParams.MATCH_PARENT,
                  ViewGroup.LayoutParams.MATCH_PARENT,
                  WindowManager.LayoutParams.TYPE_STATUS_BAR_SUB_PANEL,
                  WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN
                          | WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED
                          | WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL,
                  PixelFormat.TRANSLUCENT);
          lp.setFitInsetsTypes(lp.getFitInsetsTypes() & ~Type.statusBars());
          lp.privateFlags |= WindowManager.LayoutParams.SYSTEM_FLAG_SHOW_FOR_ALL_USERS;
          lp.setTitle("ImmersiveModeConfirmation");
          lp.windowAnimations = com.android.internal.R.style.Animation_ImmersiveModeConfirmation;
          lp.token = getWindowToken();
          return lp;
      }
  
      private FrameLayout.LayoutParams getBubbleLayoutParams() {
          return new FrameLayout.LayoutParams(
                  mContext.getResources().getDimensionPixelSize(
                          R.dimen.immersive_mode_cling_width),
                  ViewGroup.LayoutParams.WRAP_CONTENT,
                  Gravity.CENTER_HORIZONTAL | Gravity.TOP);
      }
  
    private void handleShow() {
          if (DEBUG) Slog.d(TAG, "Showing immersive mode confirmation");
  
          mClingWindow = new ClingWindowView(mContext, mConfirm);
  
          // we will be hiding the nav bar, so layout as if it's already hidden
          mClingWindow.setSystemUiVisibility(
                  View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION | View.SYSTEM_UI_FLAG_LAYOUT_STABLE);
  
          // show the confirmation
          WindowManager.LayoutParams lp = getClingWindowLayoutParams();
          getWindowManager().addView(mClingWindow, lp);
      }

```

通过代码发现  
而 handleShow() 负责启动 ClingWindowView 这个窗口从而显示提醒弹窗的功能

```
private void handleShow() {
    if (DEBUG) Slog.d(TAG, "Showing immersive mode confirmation");

    mClingWindow = new ClingWindowView(mContext, mConfirm);

    // we will be hiding the nav bar, so layout as if it's already hidden
    mClingWindow.setSystemUiVisibility(
            View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION | View.SYSTEM_UI_FLAG_LAYOUT_STABLE);

    // show the confirmation
    WindowManager.LayoutParams lp = getClingWindowLayoutParams();
    getWindowManager().addView(mClingWindow, lp);
}

private final class H extends Handler {
    private static final int SHOW = 1;
    private static final int HIDE = 2;

    H(Looper looper) {
        super(looper);
    }

    @Override
    public void handleMessage(Message msg) {
        switch(msg.what) {
            case SHOW:
                //handleShow();
                break;
            case HIDE:
                handleHide();
                break;
        }
    }
}

```

具体修改应该是在收到弹窗消息后不弹出窗口  
这里收到 Handler 消息时 弹出提示窗口  
所以 注释掉 handleShow（）编译后发现不会在有提示窗口了
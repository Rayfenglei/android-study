> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124830875)

### 1. 概述

在 11.0 12.0 的 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 状态栏中，时间控件默认显示是不带秒的只有时分，有产品需求要求状态栏时间要求带秒显示，用于一些功能开发，所以就需要修改时间控件的时间显示格式满足功能要求

### 2.SystemUI 状态栏[时间显示](https://so.csdn.net/so/search?q=%E6%97%B6%E9%97%B4%E6%98%BE%E7%A4%BA&spm=1001.2101.3001.7020)秒的核心类

```
frameworks/base/packages/SystemUI/res/layout/status_bar.xml
frameworks/base/packages/SystemUI/

```

### 3.SystemUI 状态栏时间显示秒的核心类功能分析和实现

在 SystemUI 的状态栏布局中发现状态栏布局  
就是 status_bar.xml 中的 clock 接下来看下 status_bar.xml 的布局文件

### 3.1status_bar.xml 中源码分析

```
 <com.android.systemui.statusbar.policy.Clock
                    android:id="@+id/clock"
                    android:layout_width="wrap_content"
                    android:layout_height="match_parent"
                    android:textAppearance="@style/TextAppearance.StatusBar.Clock"
                    android:singleLine="true"
                    android:paddingStart="@dimen/status_bar_left_clock_starting_padding"
                    android:paddingEnd="@dimen/status_bar_left_clock_end_padding"
                    android:gravity="center_vertical|start"
                />

```

通过源码分析得知 com.android.systemui.statusbar.policy.Clock 就是状态栏时间控件类  
也就是 Clock .java  
接下来看 Clock.java 源码

### 3.2 SystemUI 状态栏时间显示秒的功能分析

```
public class Clock extends TextView implements DemoMode, Tunable, CommandQueue.Callbacks,
          DarkReceiver, ConfigurationListener {
 public Clock(Context context, AttributeSet attrs) {
          this(context, attrs, 0);
      }
  
      public Clock(Context context, AttributeSet attrs, int defStyle) {
          super(context, attrs, defStyle);
          mCommandQueue = Dependency.get(CommandQueue.class);
          TypedArray a = context.getTheme().obtainStyledAttributes(
                  attrs,
                  R.styleable.Clock,
                  0, 0);
          try {
              mAmPmStyle = a.getInt(R.styleable.Clock_amPmStyle, AM_PM_STYLE_GONE);
              mShowDark = a.getBoolean(R.styleable.Clock_showDark, true);
              mNonAdaptedColor = getCurrentTextColor();
          } finally {
              a.recycle();
          }
          mBroadcastDispatcher = Dependency.get(BroadcastDispatcher.class);
          mCurrentUserTracker = new CurrentUserTracker(mBroadcastDispatcher) {
              @Override
              public void onUserSwitched(int newUserId) {
                  mCurrentUserId = newUserId;
              }
          };
      }
         @Override
      protected void onAttachedToWindow() {
          super.onAttachedToWindow();
  
          if (!mAttached) {
              mAttached = true;
              IntentFilter filter = new IntentFilter();
  
              filter.addAction(Intent.ACTION_TIME_TICK);
              filter.addAction(Intent.ACTION_TIME_CHANGED);
              filter.addAction(Intent.ACTION_TIMEZONE_CHANGED);
              filter.addAction(Intent.ACTION_CONFIGURATION_CHANGED);
              filter.addAction(Intent.ACTION_USER_SWITCHED);
  
              // NOTE: This receiver could run before this method returns, as it's not dispatching
              // on the main thread and BroadcastDispatcher may not need to register with Context.
              // The receiver will return immediately if the view does not have a Handler yet.
              mBroadcastDispatcher.registerReceiverWithHandler(mIntentReceiver, filter,
                      Dependency.get(Dependency.TIME_TICK_HANDLER), UserHandle.ALL);
              Dependency.get(TunerService.class).addTunable(this, CLOCK_SECONDS,
                      StatusBarIconController.ICON_BLACKLIST);
              mCommandQueue.addCallback(this);
              if (mShowDark) {
                  Dependency.get(DarkIconDispatcher.class).addDarkReceiver(this);
              }
              mCurrentUserTracker.startTracking();
              mCurrentUserId = mCurrentUserTracker.getCurrentUserId();
          }
  
          // The time zone may have changed while the receiver wasn't registered, so update the Time
          mCalendar = Calendar.getInstance(TimeZone.getDefault());
          mClockFormatString = "";
  
          // Make sure we update to the current time
          updateClock();
          updateClockVisibility();
          updateShowSeconds();
      }
      
private void updateShowSeconds() {
    if (mShowSeconds) {
        // Wait until we have a display to start trying to show seconds.
        if (mSecondsHandler == null && getDisplay() != null) {
            mSecondsHandler = new Handler();
            if (getDisplay().getState() == Display.STATE_ON) {
                mSecondsHandler.postAtTime(mSecondTick,
                        SystemClock.uptimeMillis() / 1000 * 1000 + 1000);
            }
            IntentFilter filter = new IntentFilter(Intent.ACTION_SCREEN_OFF);
            filter.addAction(Intent.ACTION_SCREEN_ON);
            mContext.registerReceiver(mScreenReceiver, filter);
        }
    } else {
        if (mSecondsHandler != null) {
            mContext.unregisterReceiver(mScreenReceiver);
            mSecondsHandler.removeCallbacks(mSecondTick);
            mSecondsHandler = null;
            updateClock();
        }
    }
}
private final BroadcastReceiver mIntentReceiver = new BroadcastReceiver() {
         @Override
         public void onReceive(Context context, Intent intent) {
             // If the handler is null, it means we received a broadcast while the view has not
             // finished being attached or in the process of being detached.
             // In that case, do not post anything.
             Handler handler = getHandler();
             if (handler == null) return;
 
             String action = intent.getAction();
             if (action.equals(Intent.ACTION_TIMEZONE_CHANGED)) {
                 String tz = intent.getStringExtra(Intent.EXTRA_TIMEZONE);
                 handler.post(() -> {
                     mCalendar = Calendar.getInstance(TimeZone.getTimeZone(tz));
                     if (mClockFormat != null) {
                         mClockFormat.setTimeZone(mCalendar.getTimeZone());
                     }
                 });
             } else if (action.equals(Intent.ACTION_CONFIGURATION_CHANGED)) {
                 final Locale newLocale = getResources().getConfiguration().locale;
                 handler.post(() -> {
                     if (!newLocale.equals(mLocale)) {
                         mLocale = newLocale;
                         mClockFormatString = ""; // force refresh
                     }
                 });
             }
             handler.post(() -> updateClock());
         }
     };
    private void updateClockVisibility() {
          boolean visible = shouldBeVisible();
          int visibility = visible ? View.VISIBLE : View.GONE;
          super.setVisibility(visibility);
      }
  
      final void updateClock() {
          if (mDemoMode) return;
          mCalendar.setTimeInMillis(System.currentTimeMillis());
          setText(getSmallTime());
          setContentDescription(mContentDescriptionFormat.format(mCalendar.getTime()));
      }

```

在 onAttachedToWindow() 中调用 updateShowSeconds() 来负责更新状态栏的时间格式，这时会对时间进行刷新显示调用 updateClock() 更新时间，在收到时间变化广播同样也是在这里调用时间更新而在 updateShowSeconds() 用  
private boolean mShowSeconds;  
是否显示秒

在这里根据是否显示秒来显示时间  
所以要查看哪里赋值了

```
@Override
    public void onTuningChanged(String key, String newValue) {
        if (CLOCK_SECONDS.equals(key)) {
            mShowSeconds = TunerService.parseIntegerSwitch(newValue, false);
            updateShowSeconds();
        } else {
            setClockVisibleByUser(!StatusBarIconController.getIconBlacklist(newValue)
                    .contains("clock"));
            updateClockVisibility();
        }
    }

```

从上述代码发现在 onTuningChanged（）

中有针对 mShowSeconds 赋值

在这里 吧 mShowSeconds=true 就行了

修改如下:

```
public void onTuningChanged(String key, String newValue) {
        if (CLOCK_SECONDS.equals(key)) {
            mShowSeconds = TunerService.parseIntegerSwitch(newValue, false);
			mShowSeconds = true;
            updateShowSeconds();
        } else {
            setClockVisibleByUser(!StatusBarIconController.getIconBlacklist(newValue)
                    .contains("clock"));
            updateClockVisibility();
        }
    }

```

编译修改完成发现实现了功能需求
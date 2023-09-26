> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126511128)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 下拉状态栏录屏去掉弹窗直接录屏的核心代码](#t1)

[3. 下拉状态栏录屏去掉弹窗直接录屏的功能分析和具体实现](#t2)

[3.1 ScreenRecordTile.java 相关录屏代码](#t3)

[3.2 RecordingController 的相关方法分析](#t4)

[3.3 ScreenRecordDialog 相关代码分析](#t5)

1. 概述
-----

在下拉状态栏中有个录屏的快捷按钮，可以通过点击录屏实现录屏功能，但是在录屏的时候发现需要先弹出 [dialog](https://so.csdn.net/so/search?q=dialog&spm=1001.2101.3001.7020)，然后点击开始实现录屏，这有的麻烦，所以需要去掉弹窗直接开始录屏，就需要弹窗的相关代码来实现功能

2. 下拉状态栏录屏去掉弹窗直接录屏的核心代码
-----------------------

```
frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tiles/ScreenRecordTile.java
frameworks/base/packages/SystemUI/src/com/android/systemui/screenrecord/RecordingController.java
frameworks/base/packages/SystemUI/src/com/android/systemui/screenrecord/ScreenRecordDialog.java
```

3. 下拉状态栏录屏去掉弹窗直接录屏的功能分析和具体实现
----------------------------

### 3.1 ScreenRecordTile.java 相关录屏代码

```
public class ScreenRecordTile extends QSTileImpl<QSTile.BooleanState>
         implements RecordingController.RecordingStateChangeCallback {
     private static final String TAG = "ScreenRecordTile";
     private RecordingController mController;
     private KeyguardDismissUtil mKeyguardDismissUtil;
     private long mMillisUntilFinished = 0;
     private Callback mCallback = new Callback();
 
     @Inject
     public ScreenRecordTile(QSHost host, RecordingController controller,
             KeyguardDismissUtil keyguardDismissUtil) {
         super(host);
         mController = controller;
         mController.observe(this, mCallback);
         mKeyguardDismissUtil = keyguardDismissUtil;
     }
 
     @Override
     public BooleanState newTileState() {
         BooleanState state = new BooleanState();
         state.label = mContext.getString(R.string.quick_settings_screen_record_label);
         state.handlesLongClick = false;
         return state;
     }
 
     @Override
     protected void handleClick() {
         if (mController.isStarting()) {
             cancelCountdown();
         } else if (mController.isRecording()) {
             stopRecording();
         } else {
             mUiHandler.post(() -> showPrompt());
         }
         refreshState();
     }
      @Override
      public int getMetricsCategory() {
          return 0;
      }
  
      @Override
      public Intent getLongClickIntent() {
          return null;
      }
  
      @Override
      public CharSequence getTileLabel() {
          return mContext.getString(R.string.quick_settings_screen_record_label);
      }
  
      private void showPrompt() {
          // Close QS, otherwise the dialog appears beneath it
          getHost().collapsePanels();
          Intent intent = mController.getPromptIntent();
          ActivityStarter.OnDismissAction dismissAction = () -> {
              mContext.startActivity(intent);
              return false;
          };
          mKeyguardDismissUtil.executeWhenUnlocked(dismissAction, false);
      }
  
      private void cancelCountdown() {
          Log.d(TAG, "Cancelling countdown");
          mController.cancelCountdown();
      }
  
      private void stopRecording() {
          mController.stopRecording();
      }
}
从showPrompt()，cancelCountdown()，stopRecording()可以看出是通过RecordingController的
相关方法实现录屏
```

### 3.2 RecordingController 的相关方法分析

```
public class RecordingController
         implements CallbackController<RecordingController.RecordingStateChangeCallback> {
     private static final String TAG = "RecordingController";
     private static final String SYSUI_PACKAGE = "com.android.systemui";
     private static final String SYSUI_SCREENRECORD_LAUNCHER =
             "com.android.systemui.screenrecord.ScreenRecordDialog";
 
     private boolean mIsStarting;
     private boolean mIsRecording;
     private PendingIntent mStopIntent;
     private CountDownTimer mCountDownTimer = null;
     private BroadcastDispatcher mBroadcastDispatcher;
 
     private ArrayList<RecordingStateChangeCallback> mListeners = new ArrayList<>();
 
     @VisibleForTesting
     protected final BroadcastReceiver mUserChangeReceiver = new BroadcastReceiver() {
         @Override
         public void onReceive(Context context, Intent intent) {
             if (mStopIntent != null) {
                 stopRecording();
             }
         }
     };
 
     /**
      * Create a new RecordingController
      */
     @Inject
     public RecordingController(BroadcastDispatcher broadcastDispatcher) {
         mBroadcastDispatcher = broadcastDispatcher;
     }
 
     /**
      * Get an intent to show screen recording options to the user.
      */
     public Intent getPromptIntent() {
         final ComponentName launcherComponent = new ComponentName(SYSUI_PACKAGE,
                 SYSUI_SCREENRECORD_LAUNCHER);
         final Intent intent = new Intent();
         intent.setComponent(launcherComponent);
         intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
         return intent;
     }
 
     /**
      * Start counting down in preparation to start a recording
      * @param ms Total time in ms to wait before starting
      * @param interval Time in ms per countdown step
      * @param startIntent Intent to start a recording
      * @param stopIntent Intent to stop a recording
      */
     public void startCountdown(long ms, long interval, PendingIntent startIntent,
             PendingIntent stopIntent) {
         mIsStarting = true;
         mStopIntent = stopIntent;
 
         mCountDownTimer = new CountDownTimer(ms, interval) {
              @Override
              public void onTick(long millisUntilFinished) {
                  for (RecordingStateChangeCallback cb : mListeners) {
                      cb.onCountdown(millisUntilFinished);
                  }
              }
  
              @Override
              public void onFinish() {
                  mIsStarting = false;
                  mIsRecording = true;
                  for (RecordingStateChangeCallback cb : mListeners) {
                      cb.onCountdownEnd();
                  }
                  try {
                      startIntent.send();
                      IntentFilter userFilter = new IntentFilter(Intent.ACTION_USER_SWITCHED);
                      mBroadcastDispatcher.registerReceiver(mUserChangeReceiver, userFilter, null,
                              UserHandle.ALL);
                      Log.d(TAG, "sent start intent");
                  } catch (PendingIntent.CanceledException e) {
                      Log.e(TAG, "Pending intent was cancelled: " + e.getMessage());
                  }
              }
          };
  
          mCountDownTimer.start();
      }
  
      /**
       * Cancel a countdown in progress. This will not stop the recording if it already started.
       */
      public void cancelCountdown() {
          if (mCountDownTimer != null) {
              mCountDownTimer.cancel();
          } else {
              Log.e(TAG, "Timer was null");
          }
          mIsStarting = false;
  
          for (RecordingStateChangeCallback cb : mListeners) {
              cb.onCountdownEnd();
          }
      }
  
      /**
       * Check if the recording is currently counting down to begin
       * @return
       */
      public boolean isStarting() {
          return mIsStarting;
      }
  
      /**
       * Check if the recording is ongoing
       * @return
       */
      public synchronized boolean isRecording() {
          return mIsRecording;
      }
  
      /**
       * Stop the recording
       */
      public void stopRecording() {
          try {
              if (mStopIntent != null) {
                  mStopIntent.send();
              } else {
                  Log.e(TAG, "Stop intent was null");
              }
              updateState(false);
          } catch (PendingIntent.CanceledException e) {
              Log.e(TAG, "Error stopping: " + e.getMessage());
          }
          mBroadcastDispatcher.unregisterReceiver(mUserChangeReceiver);
      }
  }
```

### 3.3 ScreenRecordDialog 相关代码分析

```
public class ScreenRecordDialog extends Activity {
     private static final long DELAY_MS = 3000;
     private static final long INTERVAL_MS = 1000;
     private static final String TAG = "ScreenRecordDialog";
 
     private final RecordingController mController;
     private final CurrentUserContextTracker mCurrentUserContextTracker;
     private Switch mTapsSwitch;
     private Switch mAudioSwitch;
     private Spinner mOptions;
     private List<ScreenRecordingAudioSource> mModes;
 
     @Inject
     public ScreenRecordDialog(RecordingController controller,
             CurrentUserContextTracker currentUserContextTracker) {
         mController = controller;
         mCurrentUserContextTracker = currentUserContextTracker;
     }
 
     @Override
     public void onCreate(Bundle savedInstanceState) {
         super.onCreate(savedInstanceState);
 
         Window window = getWindow();
         // Inflate the decor view, so the attributes below are not overwritten by the theme.
         window.getDecorView();
         window.setLayout(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT);
         window.addPrivateFlags(WindowManager.LayoutParams.SYSTEM_FLAG_SHOW_FOR_ALL_USERS);
         window.setGravity(Gravity.TOP);
         setTitle(R.string.screenrecord_name);
 
         setContentView(R.layout.screen_record_dialog);
 
         Button cancelBtn = findViewById(R.id.button_cancel);
         cancelBtn.setOnClickListener(v -> {
             finish();
         });
 
         Button startBtn = findViewById(R.id.button_start);
         startBtn.setOnClickListener(v -> {
             requestScreenCapture();
             finish();
         });
 
         mModes = new ArrayList<>();
         mModes.add(MIC);
         mModes.add(INTERNAL);
         mModes.add(MIC_AND_INTERNAL);
 
         mAudioSwitch = findViewById(R.id.screenrecord_audio_switch);
         mTapsSwitch = findViewById(R.id.screenrecord_taps_switch);
         mOptions = findViewById(R.id.screen_recording_options);
          ArrayAdapter a = new ScreenRecordingAdapter(getApplicationContext(),
                  android.R.layout.simple_spinner_dropdown_item,
                  mModes);
          a.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item);
          mOptions.setAdapter(a);
          mOptions.setOnItemClickListenerInt((parent, view, position, id) -> {
              mAudioSwitch.setChecked(true);
          });
      }
  
      private void requestScreenCapture() {
          Context userContext = mCurrentUserContextTracker.getCurrentUserContext();
          boolean showTaps = mTapsSwitch.isChecked();
          ScreenRecordingAudioSource audioMode = mAudioSwitch.isChecked()
                  ? (ScreenRecordingAudioSource) mOptions.getSelectedItem()
                  : NONE;
          PendingIntent startIntent = PendingIntent.getForegroundService(userContext,
                  RecordingService.REQUEST_CODE,
                  RecordingService.getStartIntent(
                          userContext, RESULT_OK,
                          audioMode.ordinal(), showTaps),
                  PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_IMMUTABLE);
          PendingIntent stopIntent = PendingIntent.getService(userContext,
                  RecordingService.REQUEST_CODE,
                  RecordingService.getStopIntent(userContext),
                  PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_IMMUTABLE);
          mController.startCountdown(DELAY_MS, INTERVAL_MS, startIntent, stopIntent);
      }
  }
 
具体实现如下:
public class ScreenRecordDialog extends Activity {
     private static final long DELAY_MS = 3000;
     private static final long INTERVAL_MS = 1000;
     private static final String TAG = "ScreenRecordDialog";
 
     private final RecordingController mController;
     private final CurrentUserContextTracker mCurrentUserContextTracker;
     private Switch mTapsSwitch;
     private Switch mAudioSwitch;
     private Spinner mOptions;
     private List<ScreenRecordingAudioSource> mModes;
 
     @Inject
     public ScreenRecordDialog(RecordingController controller,
             CurrentUserContextTracker currentUserContextTracker) {
         mController = controller;
         mCurrentUserContextTracker = currentUserContextTracker;
     }
 
     @Override
     public void onCreate(Bundle savedInstanceState) {
         
//add core start 
+        requestScreenCapture();
         super.onCreate(savedInstanceState);
+        finish();
    // 注释掉下面代码
        /* super.onCreate(savedInstanceState);
 
         Window window = getWindow();
         // Inflate the decor view, so the attributes below are not overwritten by the theme.
         window.getDecorView();
         window.setLayout(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT);
         window.addPrivateFlags(WindowManager.LayoutParams.SYSTEM_FLAG_SHOW_FOR_ALL_USERS);
         window.setGravity(Gravity.TOP);
         setTitle(R.string.screenrecord_name);
 
         setContentView(R.layout.screen_record_dialog);
 
         Button cancelBtn = findViewById(R.id.button_cancel);
         cancelBtn.setOnClickListener(v -> {
             finish();
         });
 
         Button startBtn = findViewById(R.id.button_start);
         startBtn.setOnClickListener(v -> {
             requestScreenCapture();
             finish();
         });
 
         mModes = new ArrayList<>();
         mModes.add(MIC);
         mModes.add(INTERNAL);
         mModes.add(MIC_AND_INTERNAL);
 
         mAudioSwitch = findViewById(R.id.screenrecord_audio_switch);
         mTapsSwitch = findViewById(R.id.screenrecord_taps_switch);
         mOptions = findViewById(R.id.screen_recording_options);
          ArrayAdapter a = new ScreenRecordingAdapter(getApplicationContext(),
                  android.R.layout.simple_spinner_dropdown_item,
                  mModes);
          a.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item);
          mOptions.setAdapter(a);
          mOptions.setOnItemClickListenerInt((parent, view, position, id) -> {
              mAudioSwitch.setChecked(true);
          });*/
      }
  
      private void requestScreenCapture() {
          Context userContext = mCurrentUserContextTracker.getCurrentUserContext();
        -  boolean showTaps = mTapsSwitch.isChecked();
        + boolean showTaps = true;
        -  ScreenRecordingAudioSource audioMode = mAudioSwitch.isChecked()
        -          ? (ScreenRecordingAudioSource) mOptions.getSelectedItem()
        -          : NONE;
       + ScreenRecordingAudioSource audioMode = MIC_AND_INTERNAL;
          PendingIntent startIntent = PendingIntent.getForegroundService(userContext,
                  RecordingService.REQUEST_CODE,
                  RecordingService.getStartIntent(
                          userContext, RESULT_OK,
                          audioMode.ordinal(), showTaps),
                  PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_IMMUTABLE);
          PendingIntent stopIntent = PendingIntent.getService(userContext,
                  RecordingService.REQUEST_CODE,
                  RecordingService.getStopIntent(userContext),
                  PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_IMMUTABLE);
          mController.startCountdown(DELAY_MS, INTERVAL_MS, startIntent, stopIntent);
      }
  }
```
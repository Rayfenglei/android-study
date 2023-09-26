> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124893996)

### 1. 概述

在 11.0 定制化开发中, 有需求是用开关按钮控制电源键是否可操作，这样的需求要通过系统属性来判断当收到事件  
后判断是否往下传递事件达到控制电源键是否可以用的功能

### 2. 禁用电源键 (屏蔽关机短按长按事件) 的核心代码

```
frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java

```

### 3. 禁用电源键 (屏蔽关机短按长按事件) 的功能分析

关于按键的事件 电源键音量键 等等事件都是在 PhoneWindowManager 中处理的

电源按键事件是通过驱动事件上报到 PhoneWindowManager 中来处理的  
接下来看下 PhoneWindowManager 的相关代码  
短按事件  
有 powerPress(long eventTime, [boolean](https://so.csdn.net/so/search?q=boolean&spm=1001.2101.3001.7020) interactive, int count) 来处理

```
private class PolicyHandler extends Handler {
          @Override
          public void handleMessage(Message msg) {
              switch (msg.what) {
                  case MSG_DISPATCH_MEDIA_KEY_WITH_WAKE_LOCK:
                      dispatchMediaKeyWithWakeLock((KeyEvent)msg.obj);
                      break;
                  case MSG_DISPATCH_MEDIA_KEY_REPEAT_WITH_WAKE_LOCK:
                      dispatchMediaKeyRepeatWithWakeLock((KeyEvent)msg.obj);
                      break;
                  case MSG_DISPATCH_SHOW_RECENTS:
                      showRecentApps(false);
                      break;
                  case MSG_DISPATCH_SHOW_GLOBAL_ACTIONS:
                      showGlobalActionsInternal();
                      break;
                  case MSG_KEYGUARD_DRAWN_COMPLETE:
                      if (DEBUG_WAKEUP) Slog.w(TAG, "Setting mKeyguardDrawComplete");
                      finishKeyguardDrawn();
                      break;
                  case MSG_KEYGUARD_DRAWN_TIMEOUT:
                      Slog.w(TAG, "Keyguard drawn timeout. Setting mKeyguardDrawComplete");
                      finishKeyguardDrawn();
                      break;
                  case MSG_WINDOW_MANAGER_DRAWN_COMPLETE:
                      if (DEBUG_WAKEUP) Slog.w(TAG, "Setting mWindowManagerDrawComplete");
                      finishWindowsDrawn();
                      break;
                  case MSG_HIDE_BOOT_MESSAGE:
                      handleHideBootMessage();
                      break;
                  case MSG_LAUNCH_ASSIST:
                      final int deviceId = msg.arg1;
                      final String hint = (String) msg.obj;
                      launchAssistAction(hint, deviceId);
                      break;
                  case MSG_LAUNCH_ASSIST_LONG_PRESS:
                      launchAssistLongPressAction();
                      break;
                  case MSG_LAUNCH_VOICE_ASSIST_WITH_WAKE_LOCK:
                      launchVoiceAssistWithWakeLock();
                      break;
                  case MSG_POWER_DELAYED_PRESS:
                      powerPress((Long) msg.obj, msg.arg1 != 0, msg.arg2);
                      finishPowerKeyPress();
                      break;
                  case MSG_POWER_LONG_PRESS:
                      powerLongPress();
                      break;
                  case MSG_POWER_VERY_LONG_PRESS:
                      powerVeryLongPress();
                      break;
                  case MSG_SHOW_PICTURE_IN_PICTURE_MENU:
                      showPictureInPictureMenuInternal();
                      break;
                  case MSG_BACK_LONG_PRESS:
                      backLongPress();
                      break;
                  case MSG_ACCESSIBILITY_SHORTCUT:
                      accessibilityShortcutActivated();
                      break;
                  case MSG_BUGREPORT_TV:
                      requestBugreportForTv();
                      break;
                  case MSG_ACCESSIBILITY_TV:
                      if (mAccessibilityShortcutController.isAccessibilityShortcutAvailable(false)) {
                          accessibilityShortcutActivated();
                      }
                      break;
                  case MSG_DISPATCH_BACK_KEY_TO_AUTOFILL:
                      mAutofillManagerInternal.onBackKeyPressed();
                      break;
                  case MSG_SYSTEM_KEY_PRESS:
                      sendSystemKeyToStatusBar(msg.arg1);
                      break;
                  case MSG_HANDLE_ALL_APPS:
                      launchAllAppsAction();
                      break;
                  case MSG_RINGER_TOGGLE_CHORD:
                      handleRingerChordGesture();
                      break;
              }
          }
      }
      
private void powerPress(long eventTime, boolean interactive, int count) {
if (mDefaultDisplayPolicy.isScreenOnEarly() && !mDefaultDisplayPolicy.isScreenOnFully()) {
Slog.i(TAG, "Suppressed redundant power key press while "
+ "already in the process of turning the screen on.");
return;
}
Slog.d(TAG, "powerPress: eventTime=" + eventTime + " interactive=" + interactive
+ " count=" + count + " beganFromNonInteractive=" + mBeganFromNonInteractive +
" mShortPressOnPowerBehavior=" + mShortPressOnPowerBehavior);

//添加标志位 当为false时返回
String aTrue = SystemProperties.get("persist.sys.roco.power.enable", "true");
Log.d("dong", "powershortPress true:" +aTrue);
if(!"true".equals(aTrue)){
return;
}

if (count == 2) {
powerMultiPressAction(eventTime, interactive, mDoublePressOnPowerBehavior);
} else if (count == 3) {
powerMultiPressAction(eventTime, interactive, mTriplePressOnPowerBehavior);
} else if (interactive && !mBeganFromNonInteractive) {
switch (mShortPressOnPowerBehavior) {
case SHORT_PRESS_POWER_NOTHING:
break;
case SHORT_PRESS_POWER_GO_TO_SLEEP:
goToSleepFromPowerButton(eventTime, 0);
break;
case SHORT_PRESS_POWER_REALLY_GO_TO_SLEEP:
goToSleepFromPowerButton(eventTime, PowerManager.GO_TO_SLEEP_FLAG_NO_DOZE);
break;
case SHORT_PRESS_POWER_REALLY_GO_TO_SLEEP_AND_GO_HOME:
if (goToSleepFromPowerButton(eventTime,
PowerManager.GO_TO_SLEEP_FLAG_NO_DOZE)) {
launchHomeFromHotKey(DEFAULT_DISPLAY);
}
break;
case SHORT_PRESS_POWER_GO_HOME:
shortPressPowerGoHome();
break;
case SHORT_PRESS_POWER_CLOSE_IME_OR_GO_HOME: {
if (mDismissImeOnBackKeyPressed) {
if (mInputMethodManagerInternal == null) {
mInputMethodManagerInternal =
LocalServices.getService(InputMethodManagerInternal.class);
}
if (mInputMethodManagerInternal != null) {
mInputMethodManagerInternal.hideCurrentInputMethod();
}
} else {
shortPressPowerGoHome();
}
break;
}
}
}
}

```

从上述代码可以看出  
case MSG_POWER_DELAYED_PRESS:  
powerPress((Long) msg.obj, msg.arg1 != 0, msg.arg2);  
finishPowerKeyPress();  
break;  
case MSG_POWER_LONG_PRESS:  
powerLongPress();  
break;  
powerPress((Long) msg.obj, msg.arg1 != 0, msg.arg2); 就是短按事件  
而 powerLongPress(); 就是长按事件  
长按事件处理

```
private void powerLongPress() {
//添加标志位 当为false时返回
String aTrue = SystemProperties.get("persist.sys.roco.power.enable", "true");
Log.d("dong", "powerLongPress true:" +aTrue);
if(!"true".equals(aTrue)){
return;
}

final int behavior = getResolvedLongPressOnPowerBehavior();
switch (behavior) {
case LONG_PRESS_POWER_NOTHING:
break;
case LONG_PRESS_POWER_GLOBAL_ACTIONS:
mPowerKeyHandled = true;
performHapticFeedback(HapticFeedbackConstants.LONG_PRESS, false,
"Power - Long Press - Global Actions");
showGlobalActionsInternal();
break;
case LONG_PRESS_POWER_SHUT_OFF:
case LONG_PRESS_POWER_SHUT_OFF_NO_CONFIRM:
mPowerKeyHandled = true;
performHapticFeedback(HapticFeedbackConstants.LONG_PRESS, false,
"Power - Long Press - Shut Off");
sendCloseSystemWindows(SYSTEM_DIALOG_REASON_GLOBAL_ACTIONS);
mWindowManagerFuncs.shutdown(behavior == LONG_PRESS_POWER_SHUT_OFF);
break;
case LONG_PRESS_POWER_GO_TO_VOICE_ASSIST:
mPowerKeyHandled = true;
performHapticFeedback(HapticFeedbackConstants.LONG_PRESS, false,
"Power - Long Press - Go To Voice Assist");
// Some devices allow the voice assistant intent during setup (and use that intent
// to launch something else, like Settings). So we explicitly allow that via the
// config_allowStartActivityForLongPressOnPowerInSetup resource in config.xml.
launchVoiceAssist(mAllowStartActivityForLongPressOnPowerDuringSetup);
break;
case LONG_PRESS_POWER_ASSISTANT:
mPowerKeyHandled = true;
performHapticFeedback(HapticFeedbackConstants.LONG_PRESS, false,
"Power - Long Press - Go To Assistant");
final int powerKeyDeviceId = Integer.MIN_VALUE;
launchAssistAction(null, powerKeyDeviceId);
break;
}
}

```

通过在长按和短按事件中添加系统属性控制来禁用电源键
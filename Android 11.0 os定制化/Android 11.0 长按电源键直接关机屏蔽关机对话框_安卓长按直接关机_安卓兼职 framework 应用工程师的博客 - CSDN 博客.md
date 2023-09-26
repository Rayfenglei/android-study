> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125026987)

### 1. 概述

11.0 定制化开发中，有客户需求要求长按电源键直接关机，不需要弹出关机对话框来确认关机，所以这样从电源键  
长按弹出代码流程分析, 所以就要分析 PhoneWindowManager.java 电源键的事件都是这里处理的，

### 2. 长按电源键直接关机屏蔽关机对话框核心代码

```
frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java

```

### 3. 长按电源键直接关机屏蔽关机对话框核心代码功能分析和实现功能

### 3.1 PhoneWindowManager.java 关于长按电源键的分析

在系统中 PhoneWindowManager 主要负责对按键的处理 电源键 音量键 导航栏的按键处理，所以长按电源键需要从流程分析代码然后做相应的修改  
源码路径：  
frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java

首选看下 KeyEvent.KEYCODE_POWER 处理电源键

```
case KeyEvent.KEYCODE_POWER: {
                EventLogTags.writeInterceptPower(
                        KeyEvent.actionToString(event.getAction()),
                        mPowerKeyHandled ? 1 : 0, mPowerKeyPressCounter);
                // Any activity on the power button stops the accessibility shortcut
                cancelPendingAccessibilityShortcutAction();
                result &= ~ACTION_PASS_TO_USER;
                isWakeKey = false; // wake-up will be handled separately
                if (down) {
                    /*SPRD : add power debug log start*/
                    Slog.d(TAG, "Receive Input KeyEvent of Powerkey down");
                    /*SPRD : add power debug log end*/
                    interceptPowerKeyDown(event, interactive);
                } else {
                    /*SPRD : add power debug log start*/
                    Slog.d(TAG, "Receive Input KeyEvent of Powerkey up");
                    /*SPRD : add power debug log end*/
                    interceptPowerKeyUp(event, interactive, canceled);
                }
                break;
            }

```

长按电源键按下的时候具体是靠 interceptPowerKeyDown(event, interactive); 进行处理的  
接下来看下 interceptPowerKeyDown(KeyEvent event, boolean interactive)

```
private void interceptPowerKeyDown(KeyEvent event, boolean interactive) {
// Hold a wake lock until the power key is released.
if (!mPowerKeyWakeLock.isHeld()) {
mPowerKeyWakeLock.acquire();
}

// Cancel multi-press detection timeout.
if (mPowerKeyPressCounter != 0) {
mHandler.removeMessages(MSG_POWER_DELAYED_PRESS);
}

mWindowManagerFuncs.onPowerKeyDown(interactive);

// Latch power key state to detect screenshot chord.
if (interactive && !mScreenshotChordPowerKeyTriggered
&& (event.getFlags() & KeyEvent.FLAG_FALLBACK) == 0) {
mScreenshotChordPowerKeyTriggered = true;
mScreenshotChordPowerKeyTime = event.getDownTime();
interceptScreenshotChord();
interceptRingerToggleChord();
}

// Stop ringing or end call if configured to do so when power is pressed.
TelecomManager telecomManager = getTelecommService();
boolean hungUp = false;
if (telecomManager != null) {
if (telecomManager.isRinging()) {
// Pressing Power while there's a ringing incoming
// call should silence the ringer.
telecomManager.silenceRinger();
} else if ((mIncallPowerBehavior
& Settings.Secure.INCALL_POWER_BUTTON_BEHAVIOR_HANGUP) != 0
&& telecomManager.isInCall() && interactive) {
// Otherwise, if "Power button ends call" is enabled,
// the Power button will hang up any current active call.
hungUp = telecomManager.endCall();
}
}

final boolean handledByPowerManager = mPowerManagerInternal.interceptPowerKeyDown(event);

GestureLauncherService gestureService = LocalServices.getService(
GestureLauncherService.class);
boolean gesturedServiceIntercepted = false;
if (gestureService != null) {
gesturedServiceIntercepted = gestureService.interceptPowerKeyDown(event, interactive,
mTmpBoolean);
if (mTmpBoolean.value && mRequestedOrGoingToSleep) {
mCameraGestureTriggeredDuringGoingToSleep = true;
}
}

// Inform the StatusBar; but do not allow it to consume the event.
sendSystemKeyToStatusBarAsync(event.getKeyCode());

schedulePossibleVeryLongPressReboot();

// If the power key has still not yet been handled, then detect short
// press, long press, or multi press and decide what to do.
mPowerKeyHandled = hungUp || mScreenshotChordVolumeDownKeyTriggered
|| mA11yShortcutChordVolumeUpKeyTriggered || gesturedServiceIntercepted
|| handledByPowerManager;
if (!mPowerKeyHandled) {
if (interactive) {
// When interactive, we're already awake.
// Wait for a long press or for the button to be released to decide what to do.
if (hasLongPressOnPowerBehavior()) {
if ((event.getFlags() & KeyEvent.FLAG_LONG_PRESS) != 0) {
powerLongPress();
} else {
Message msg = mHandler.obtainMessage(MSG_POWER_LONG_PRESS);
msg.setAsynchronous(true);
mHandler.sendMessageDelayed(msg,
ViewConfiguration.get(mContext).getDeviceGlobalActionKeyTimeout());

if (hasVeryLongPressOnPowerBehavior()) {
Message longMsg = mHandler.obtainMessage(MSG_POWER_VERY_LONG_PRESS);
longMsg.setAsynchronous(true);
mHandler.sendMessageDelayed(longMsg, mVeryLongPressTimeout);
}
}
}
} else {
wakeUpFromPowerKey(event.getDownTime());

if (mSupportLongPressPowerWhenNonInteractive && hasLongPressOnPowerBehavior()) {
if ((event.getFlags() & KeyEvent.FLAG_LONG_PRESS) != 0) {
powerLongPress();
} else {
Message msg = mHandler.obtainMessage(MSG_POWER_LONG_PRESS);
msg.setAsynchronous(true);
mHandler.sendMessageDelayed(msg,
ViewConfiguration.get(mContext).getDeviceGlobalActionKeyTimeout());

if (hasVeryLongPressOnPowerBehavior()) {
Message longMsg = mHandler.obtainMessage(MSG_POWER_VERY_LONG_PRESS);
longMsg.setAsynchronous(true);
mHandler.sendMessageDelayed(longMsg, mVeryLongPressTimeout);
}
}

mBeganFromNonInteractive = true;
} else {
final int maxCount = getMaxMultiPressPowerCount();

if (maxCount <= 1) {
mPowerKeyHandled = true;
} else {
mBeganFromNonInteractive = true;
}
}
}
}
}

```

从上述代码可以看出最终会发送 MSG_POWER_LONG_PRESS 消息的操作。因此我们搜索：MSG_POWER_LONG_PRESS，  
在之前定义好的 Handler 中找到了相关代码，最后通过 Handler 调用了 “powerLongPress()” 方法。

```
private void powerLongPress() {
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

LONG_PRESS_POWER_GLOBAL_ACTIONS 就是长按电源处理事件而 showGlobalActionsInternal(); 就是弹出对话框  
所以屏蔽对话框然后增加关机功能 在这里修改就行了  
而关机功能在系统中可以调用 pm.shutdown 来实现关机功能，  
修改如下:

```
--- a/frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
+++ b/frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
@@ -1335,7 +1335,10 @@ public class PhoneWindowManager extends AbsPhoneWindowManager implements WindowM
                 mPowerKeyHandled = true;
                 performHapticFeedback(HapticFeedbackConstants.LONG_PRESS, false,
                         "Power - Long Press - Global Actions");
-                showGlobalActionsInternal();
+                //showGlobalActionsInternal();
+                PowerManager pm = (PowerManager) mContext.getSystemService(Context.POWER_SERVICE);
+                pm.shutdown(false, null, false);
                 break;
             case LONG_PRESS_POWER_SHUT_OFF:
             case LONG_PRESS_POWER_SHUT_OFF_NO_CONFIRM:
@@ -1343,7 +1346,7 @@ public class PhoneWindowManager extends AbsPhoneWindowManager implements WindowM
                 performHapticFeedback(HapticFeedbackConstants.LONG_PRESS, false,
                         "Power - Long Press - Shut Off");
                 sendCloseSystemWindows(SYSTEM_DIALOG_REASON_GLOBAL_ACTIONS);
                mWindowManagerFuncs.shutdown(behavior == LONG_PRESS_POWER_SHUT_OFF);
                 break;
             case LONG_PRESS_POWER_GO_TO_VOICE_ASSIST:
                 mPowerKeyHandled = true;

```
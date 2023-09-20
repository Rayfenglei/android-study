> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124872690)

### 1. 概述

11.0 的产品开发中，系统默认可以通过音量键和电源键来截图的，但是产品不需要截图功能，所以要求去掉音量和电源键的截图功能，所以要分析组合键截图功能屏蔽掉就好了

### 2. 去掉音量键电源键组合键的屏幕截图功能的核心代码

```
frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java

```

### 3. 去掉音量键电源键组合键的屏幕截图功能分析

功能分析:  
关于按键的处理都是在 PhoneWindowManager 中有两个方法 interceptKeyBeforeDispatching 和 interceptKeyBeforeQueueing，其中包括了几乎所有按键的处理，interceptKeyBeforeDispatching 主要处理 Home 键、音量键、back 键等，

interceptKeyBeforeQueueing 主要处理音量键、电源键、耳机键等。接下来分析下源码

首先分析 interceptKeyBeforeQueueing（）如下:

```
// TODO(b/117479243): handle it in InputPolicy
    /** {@inheritDoc} */
    @Override
    public int interceptKeyBeforeQueueing(KeyEvent event, int policyFlags) {
        if (!mSystemBooted) {
            // If we have not yet booted, don't let key events do anything.
            return 0;
        }
        // add for control disable all the key operation while receive the  emergency cb message
        if (mDisableAllKeyAction){
            return 0;
        }
        final boolean interactive = (policyFlags & FLAG_INTERACTIVE) != 0;
        final boolean down = event.getAction() == KeyEvent.ACTION_DOWN;
        final boolean canceled = event.isCanceled();
        final int keyCode = event.getKeyCode();
        final int displayId = event.getDisplayId();

        final boolean isInjected = (policyFlags & WindowManagerPolicy.FLAG_INJECTED) != 0;

        // If screen is off then we treat the case where the keyguard is open but hidden
        // the same as if it were open and in front.
        // This will prevent any keys other than the power button from waking the screen
        // when the keyguard is hidden by another activity.
        final boolean keyguardActive = (mKeyguardDelegate == null ? false :
                                            (interactive ?
                                                isKeyguardShowingAndNotOccluded() :
                                                mKeyguardDelegate.isShowing()));

        if (DEBUG_INPUT) {
            Log.d(TAG, "interceptKeyTq keycode=" + keyCode
                    + " interactive=" + interactive + " keyguardActive=" + keyguardActive
                    + " policyFlags=" + Integer.toHexString(policyFlags));
        }

        // Basic policy based on interactive state.
        int result;
        boolean isWakeKey = (policyFlags & WindowManagerPolicy.FLAG_WAKE) != 0
                || event.isWakeKey();
        if (interactive || (isInjected && !isWakeKey)) {
            // When the device is interactive or the key is injected pass the
            // key to the application.
            result = ACTION_PASS_TO_USER;
            isWakeKey = false;

            if (interactive) {
                // If the screen is awake, but the button pressed was the one that woke the device
                // then don't pass it to the application
                if (keyCode == mPendingWakeKey && !down) {
                    result = 0;
                }
                // Reset the pending key
                mPendingWakeKey = PENDING_KEY_NULL;
            }
        } else if (!interactive && shouldDispatchInputWhenNonInteractive(displayId, keyCode)) {
            // If we're currently dozing with the screen on and the keyguard showing, pass the key
            // to the application but preserve its wake key status to make sure we still move
            // from dozing to fully interactive if we would normally go from off to fully
            // interactive.
            result = ACTION_PASS_TO_USER;
            // Since we're dispatching the input, reset the pending key
            mPendingWakeKey = PENDING_KEY_NULL;
        } else {
            // When the screen is off and the key is not injected, determine whether
            // to wake the device but don't pass the key to the application.
            result = 0;
            if (isWakeKey && (!down || !isWakeKeyWhenScreenOff(keyCode))) {
                isWakeKey = false;
            }
            // Cache the wake key on down event so we can also avoid sending the up event to the app
            if (isWakeKey && down) {
                mPendingWakeKey = keyCode;
            }
        }

        // If the key would be handled globally, just return the result, don't worry about special
        // key processing.
        if (isValidGlobalKey(keyCode)
                && mGlobalKeyManager.shouldHandleGlobalKey(keyCode, event)) {
            if (isWakeKey) {
                wakeUp(event.getEventTime(), mAllowTheaterModeWakeFromKey,
                        PowerManager.WAKE_REASON_WAKE_KEY, "android.policy:KEY");
            }
            return result;
        }

        // Enable haptics if down and virtual key without multiple repetitions. If this is a hard
        // virtual key such as a navigation bar button, only vibrate if flag is enabled.
        final boolean isNavBarVirtKey = ((event.getFlags() & KeyEvent.FLAG_VIRTUAL_HARD_KEY) != 0);
        boolean useHapticFeedback = down
                && (policyFlags & WindowManagerPolicy.FLAG_VIRTUAL) != 0
                && (!isNavBarVirtKey || mNavBarVirtualKeyHapticFeedbackEnabled)
                && event.getRepeatCount() == 0;

        // Handle special keys.
        switch (keyCode) {
            case KeyEvent.KEYCODE_BACK: {
				/*String enable=SystemProperties.get("persist.sys.nv.back", "false");
				android.util.Log.d("dong","onback enable:"+enable);
				if("true".equals(enable) ){
					break;
				}*/
                if (down) {
                    interceptBackKeyDown();
                } else {
                    boolean handled = interceptBackKeyUp(event);

                    // Don't pass back press to app if we've already handled it via long press
                    if (handled) {
                        result &= ~ACTION_PASS_TO_USER;
                    }
                }
                break;
            }
......
        return result;
    }

```

而由 handleVolumeKey(KeyEvent event, int policyFlags) 处理音量键，接下来分析下核心代码

```
public void handleVolumeKey(KeyEvent event, int policyFlags){
    final boolean down = event.getAction() == KeyEvent.ACTION_DOWN;
    final int keyCode = event.getKeyCode();
    final boolean interactive = (policyFlags & FLAG_INTERACTIVE) != 0;
    if (keyCode == KeyEvent.KEYCODE_VOLUME_DOWN) {
        if (down) {
            // Any activity on the vol down button stops the ringer toggle shortcut
            cancelPendingRingerToggleChordAction();

            if (interactive && !mScreenshotChordVolumeDownKeyTriggered
                    && (event.getFlags() & KeyEvent.FLAG_FALLBACK) == 0) {
                mScreenshotChordVolumeDownKeyTriggered = true;
                mScreenshotChordVolumeDownKeyTime = event.getDownTime();
                mScreenshotChordVolumeDownKeyConsumed = false;
                cancelPendingPowerKeyAction();
                interceptScreenshotChord();
                interceptAccessibilityShortcutChord();
            }
        } else {
            mScreenshotChordVolumeDownKeyTriggered = false;
            cancelPendingScreenshotChordAction();
            cancelPendingAccessibilityShortcutAction();
        }
    } else if (keyCode == KeyEvent.KEYCODE_VOLUME_UP) {
        if (down) {
            if (interactive && !mA11yShortcutChordVolumeUpKeyTriggered
                    && (event.getFlags() & KeyEvent.FLAG_FALLBACK) == 0) {
                mA11yShortcutChordVolumeUpKeyTriggered = true;
                mA11yShortcutChordVolumeUpKeyTime = event.getDownTime();
                mA11yShortcutChordVolumeUpKeyConsumed = false;
                cancelPendingPowerKeyAction();
                cancelPendingScreenshotChordAction();
                cancelPendingRingerToggleChordAction();

                interceptAccessibilityShortcutChord();
                interceptRingerToggleChord();
            }
        } else {
            mA11yShortcutChordVolumeUpKeyTriggered = false;
            cancelPendingScreenshotChordAction();
            cancelPendingAccessibilityShortcutAction();
            cancelPendingRingerToggleChordAction();
        }
    }
    return;
}

```

由 interceptPowerKeyDown(KeyEvent event, boolean interactive) 处理 power 键，接下来分析下核心方法

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

        //SprdGestureLauncherService gestureService = LocalServices.getService(
        //        SprdGestureLauncherService.class);
        boolean gesturedServiceIntercepted = false;
        if (getGestureLauncherService()) {
            gesturedServiceIntercepted = mGestureService.interceptPowerKeyDown(event, interactive,
                    mTmpBoolean, isScreenOn(), hasInPowerUtrlSavingMode());
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
                || mA11yShortcutChordVolumeUpKeyTriggered || gesturedServiceIntercepted;
      ....
    }

```

最终的电源音量键组合的截图功能是在 interceptScreenshotChord() 处理的，接下来看下相关代码

```
  private void interceptScreenshotChord() {
        if (mScreenshotChordEnabled
                && mScreenshotChordVolumeDownKeyTriggered && mScreenshotChordPowerKeyTriggered
                && !mA11yShortcutChordVolumeUpKeyTriggered) {
            final long now = SystemClock.uptimeMillis();
            if (now <= mScreenshotChordVolumeDownKeyTime + SCREENSHOT_CHORD_DEBOUNCE_DELAY_MILLIS
                    && now <= mScreenshotChordPowerKeyTime
                            + SCREENSHOT_CHORD_DEBOUNCE_DELAY_MILLIS) {
                mScreenshotChordVolumeDownKeyConsumed = true;
                if(hasInPowerUtrlSavingMode()){
                    Slog.d(TAG,"In power ULTRA saving mode: Do not allow to take screen shot");
                    return;
                }
                cancelPendingPowerKeyAction();
                mScreenshotRunnable.setScreenshotType(TAKE_SCREENSHOT_FULLSCREEN);
                mHandler.postDelayed(mScreenshotRunnable, getScreenshotChordLongPressDelay());
            }
        }
    }

```

所以解决方案如下:  
在 power 键和 音量键处理地方 去掉截图功能即可:  
如下：

```
--- a/frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
+++ b/frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
@@ -1015,7 +1015,7 @@ public class PhoneWindowManager extends AbsPhoneWindowManager implements WindowM
                 && (event.getFlags() & KeyEvent.FLAG_FALLBACK) == 0) {
             mScreenshotChordPowerKeyTriggered = true;
             mScreenshotChordPowerKeyTime = event.getDownTime();
-            interceptScreenshotChord();
+            //interceptScreenshotChord();
             interceptRingerToggleChord();
         }
 
@@ -4328,7 +4328,7 @@ public class PhoneWindowManager extends AbsPhoneWindowManager implements WindowM
                     mScreenshotChordVolumeDownKeyTime = event.getDownTime();
                     mScreenshotChordVolumeDownKeyConsumed = false;
                     cancelPendingPowerKeyAction();
-                    interceptScreenshotChord();
+                    //interceptScreenshotChord();
                     interceptAccessibilityShortcutChord();
                 }
             } else {

```
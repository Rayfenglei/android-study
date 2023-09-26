> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125813294)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.framework 增加音量 + 音量 - 键唤醒屏幕的功能的核心代码](#t1)

[3.framework 增加音量 + 音量 - 键唤醒屏幕的功能的核心代码功能分析](#t2)

[3.1 KeyEvent.java 按键功能分析](#t3)

[3.2 PhoneWindowManager.java 关于对按键拦截的处理相关代码分析](#t4)

1. 概述
-----

在 Android 系统中，系统 Power 按键是默认唤醒屏幕的按键，在屏幕息屏后按 power 按键 系统就唤醒屏幕  
而在 11.0 的产品开发中，需求要求音量键在息屏的时候也可以唤醒屏幕，这就需要参考 Power 键来实现功能

  
2.framework 增加音量 + 音量 - 键唤醒屏幕的功能的核心代码
----------------------------------------

```
主要核心代码：
 frameworks/base/core/java/android/view/KeyEvent.java
  frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
```

3.framework 增加音量 + 音量 - 键唤醒屏幕的功能的核心代码功能分析
-----------------------------------------

#### 3.1 KeyEvent.java 按键功能分析

```
public class KeyEvent extends InputEvent implements Parcelable {
  private KeyEvent() {}
 
 
public KeyEvent(int action, int code) {
mId = nativeNextId();
mAction = action;
mKeyCode = code;
mRepeatCount = 0;
mDeviceId = KeyCharacterMap.VIRTUAL_KEYBOARD;
}
 
 
public KeyEvent(long downTime, long eventTime, int action,
int code, int repeat) {
mId = nativeNextId();
mDownTime = downTime;
mEventTime = eventTime;
mAction = action;
mKeyCode = code;
mRepeatCount = repeat;
mDeviceId = KeyCharacterMap.VIRTUAL_KEYBOARD;
}
 
 
public KeyEvent(long downTime, long eventTime, int action,
int code, int repeat, int metaState) {
mId = nativeNextId();
mDownTime = downTime;
mEventTime = eventTime;
mAction = action;
mKeyCode = code;
mRepeatCount = repeat;
mMetaState = metaState;
mDeviceId = KeyCharacterMap.VIRTUAL_KEYBOARD;
}
 
 
public KeyEvent(long downTime, long eventTime, int action,
int code, int repeat, int metaState,
int deviceId, int scancode) {
mId = nativeNextId();
mDownTime = downTime;
mEventTime = eventTime;
mAction = action;
mKeyCode = code;
mRepeatCount = repeat;
mMetaState = metaState;
mDeviceId = deviceId;
mScanCode = scancode;
}
 
/**
 
public KeyEvent(long downTime, long eventTime, int action,
int code, int repeat, int metaState,
int deviceId, int scancode, int flags) {
mId = nativeNextId();
mDownTime = downTime;
mEventTime = eventTime;
mAction = action;
mKeyCode = code;
mRepeatCount = repeat;
mMetaState = metaState;
mDeviceId = deviceId;
mScanCode = scancode;
mFlags = flags;
}
 
/**
* Create a new key event.
*
* @param downTime The time (in {@link android.os.SystemClock#uptimeMillis})
* at which this key code originally went down.
* @param eventTime The time (in {@link android.os.SystemClock#uptimeMillis})
* at which this event happened.
* @param action Action code: either {@link #ACTION_DOWN},
* {@link #ACTION_UP}, or {@link #ACTION_MULTIPLE}.
* @param code The key code.
* @param repeat A repeat count for down events (> 0 if this is after the
* initial down) or event count for multiple events.
* @param metaState Flags indicating which meta keys are currently pressed.
* @param deviceId The device ID that generated the key event.
* @param scancode Raw device scan code of the event.
* @param flags The flags for this key event
* @param source The input source such as {@link InputDevice#SOURCE_KEYBOARD}.
*/
public KeyEvent(long downTime, long eventTime, int action,
int code, int repeat, int metaState,
int deviceId, int scancode, int flags, int source) {
mId = nativeNextId();
mDownTime = downTime;
mEventTime = eventTime;
mAction = action;
mKeyCode = code;
mRepeatCount = repeat;
mMetaState = metaState;
mDeviceId = deviceId;
mScanCode = scancode;
mFlags = flags;
mSource = source;
mDisplayId = INVALID_DISPLAY;
}
 
/**
* Create a new key event for a string of characters.  The key code,
* action, repeat count and source will automatically be set to
* {@link #KEYCODE_UNKNOWN}, {@link #ACTION_MULTIPLE}, 0, and
* {@link InputDevice#SOURCE_KEYBOARD} for you.
*
* @param time The time (in {@link android.os.SystemClock#uptimeMillis})
* at which this event occured.
* @param characters The string of characters.
* @param deviceId The device ID that generated the key event.
* @param flags The flags for this key event
*/
public KeyEvent(long time, String characters, int deviceId, int flags) {
mId = nativeNextId();
mDownTime = time;
mEventTime = time;
mCharacters = characters;
mAction = ACTION_MULTIPLE;
mKeyCode = KEYCODE_UNKNOWN;
mRepeatCount = 0;
mDeviceId = deviceId;
mFlags = flags;
mSource = InputDevice.SOURCE_KEYBOARD;
mDisplayId = INVALID_DISPLAY;
}
 
/**
* Make an exact copy of an existing key event.
*/
public KeyEvent(KeyEvent origEvent) {
mId = origEvent.mId;
mDownTime = origEvent.mDownTime;
mEventTime = origEvent.mEventTime;
mAction = origEvent.mAction;
mKeyCode = origEvent.mKeyCode;
mRepeatCount = origEvent.mRepeatCount;
mMetaState = origEvent.mMetaState;
mDeviceId = origEvent.mDeviceId;
mSource = origEvent.mSource;
mDisplayId = origEvent.mDisplayId;
mHmac = origEvent.mHmac == null ? null : origEvent.mHmac.clone();
mScanCode = origEvent.mScanCode;
mFlags = origEvent.mFlags;
mCharacters = origEvent.mCharacters;
}
 
/**
* Copy an existing key event, modifying its time and repeat count.
*
* @deprecated Use {@link #changeTimeRepeat(KeyEvent, long, int)}
* instead.
*
* @param origEvent The existing event to be copied.
* @param eventTime The new event time
* (in {@link android.os.SystemClock#uptimeMillis}) of the event.
* @param newRepeat The new repeat count of the event.
*/
@Deprecated
public KeyEvent(KeyEvent origEvent, long eventTime, int newRepeat) {
mId = nativeNextId();  // Not an exact copy so assign a new ID.
mDownTime = origEvent.mDownTime;
mEventTime = eventTime;
mAction = origEvent.mAction;
mKeyCode = origEvent.mKeyCode;
mRepeatCount = newRepeat;
mMetaState = origEvent.mMetaState;
mDeviceId = origEvent.mDeviceId;
mSource = origEvent.mSource;
mDisplayId = origEvent.mDisplayId;
mHmac = null; // Don't copy HMAC, it will be invalid because eventTime is changing
mScanCode = origEvent.mScanCode;
mFlags = origEvent.mFlags;
mCharacters = origEvent.mCharacters;
}
private static KeyEvent obtain() {
final KeyEvent ev;
synchronized (gRecyclerLock) {
ev = gRecyclerTop;
if (ev == null) {
return new KeyEvent();
}
gRecyclerTop = ev.mNext;
gRecyclerUsed -= 1;
}
ev.mNext = null;
ev.prepareForReuse();
return ev;
}
/**
* Obtains a (potentially recycled) key event. Used by native code to create a Java object.
*
* @hide
*/
public static KeyEvent obtain(int id, long downTime, long eventTime, int action,
int code, int repeat, int metaState,
int deviceId, int scancode, int flags, int source, int displayId, @Nullable byte[] hmac,
String characters) {
KeyEvent ev = obtain();
ev.mId = id;
ev.mDownTime = downTime;
ev.mEventTime = eventTime;
ev.mAction = action;
ev.mKeyCode = code;
ev.mRepeatCount = repeat;
ev.mMetaState = metaState;
ev.mDeviceId = deviceId;
ev.mScanCode = scancode;
ev.mFlags = flags;
ev.mSource = source;
ev.mDisplayId = displayId;
ev.mHmac = hmac;
ev.mCharacters = characters;
return ev;
}
/**
* Obtains a (potentially recycled) key event.
*
* @hide
*/
public static KeyEvent obtain(long downTime, long eventTime, int action,
int code, int repeat, int metaState,
int deviceId, int scanCode, int flags, int source, int displayId, String characters) {
return obtain(nativeNextId(), downTime, eventTime, action, code, repeat, metaState,
deviceId, scanCode, flags, source, displayId, null /* hmac */, characters);
}
/**
* Obtains a (potentially recycled) key event.
*
* @hide
*/
@UnsupportedAppUsage
public static KeyEvent obtain(long downTime, long eventTime, int action,
int code, int repeat, int metaState,
int deviceId, int scancode, int flags, int source, String characters) {
return obtain(downTime, eventTime, action, code, repeat, metaState, deviceId, scancode,
flags, source, INVALID_DISPLAY, characters);
}
/**
/**
* Obtains a (potentially recycled) copy of another key event.
*
* @hide
*/
public static KeyEvent obtain(KeyEvent other) {
KeyEvent ev = obtain();
ev.mId = other.mId;
ev.mDownTime = other.mDownTime;
ev.mEventTime = other.mEventTime;
ev.mAction = other.mAction;
ev.mKeyCode = other.mKeyCode;
ev.mRepeatCount = other.mRepeatCount;
ev.mMetaState = other.mMetaState;
ev.mDeviceId = other.mDeviceId;
ev.mScanCode = other.mScanCode;
ev.mFlags = other.mFlags;
ev.mSource = other.mSource;
ev.mDisplayId = other.mDisplayId;
ev.mHmac = other.mHmac == null ? null : other.mHmac.clone();
ev.mCharacters = other.mCharacters;
return ev;
}
public final boolean isWakeKey() {
          return isWakeKey(mKeyCode);
      }
public static final boolean isWakeKey(int keyCode) {
switch (keyCode) {
case KeyEvent.KEYCODE_CAMERA:
case KeyEvent.KEYCODE_MENU:
case KeyEvent.KEYCODE_PAIRING:
case KeyEvent.KEYCODE_STEM_1:
case KeyEvent.KEYCODE_STEM_2:
case KeyEvent.KEYCODE_STEM_3:
case KeyEvent.KEYCODE_WAKEUP:
return true;
}
return false;
}
```

#### 在代码中可以看出 isWakeKey(int keyCode) 就会来判断按键事件是否是唤醒键  
所以修改为:  
public static final boolean isWakeKey(int keyCode) {  
switch (keyCode) {  
case KeyEvent.KEYCODE_BACK:  
case KeyEvent.KEYCODE_MENU:  
case KeyEvent.KEYCODE_WAKEUP:  
case KeyEvent.KEYCODE_PAIRING:  
case KeyEvent.KEYCODE_STEM_1:  
case KeyEvent.KEYCODE_STEM_2:  
case KeyEvent.KEYCODE_STEM_3:  
 +     case KeyEvent.KEYCODE_VOLUME_DOWN:  
 +     case KeyEvent.KEYCODE_VOLUME_UP:  
          return true;  
  }  
  return false;  
}

#### 3.2 PhoneWindowManager.java 关于对按键拦截的处理相关代码分析

```
@Override
public int interceptKeyBeforeQueueing(KeyEvent event, int policyFlags) {
if (!mSystemBooted) {
// If we have not yet booted, don't let key events do anything.
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
 
case KeyEvent.KEYCODE_VOLUME_DOWN:
case KeyEvent.KEYCODE_VOLUME_UP:
case KeyEvent.KEYCODE_VOLUME_MUTE: {
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
if (down) {
sendSystemKeyToStatusBarAsync(event.getKeyCode());
 
NotificationManager nm = getNotificationService();
if (nm != null && !mHandleVolumeKeysInWM) {
nm.silenceNotificationSound();
}
 
TelecomManager telecomManager = getTelecommService();
if (telecomManager != null && !mHandleVolumeKeysInWM) {
// When {@link #mHandleVolumeKeysInWM} is set, volume key events
// should be dispatched to WM.
if (telecomManager.isRinging()) {
// If an incoming call is ringing, either VOLUME key means
// "silence ringer".  We handle these keys here, rather than
// in the InCallScreen, to make sure we'll respond to them
// even if the InCallScreen hasn't come to the foreground yet.
// Look for the DOWN event here, to agree with the "fallback"
// behavior in the InCallScreen.
Log.i(TAG, "interceptKeyBeforeQueueing:"
+ " VOLUME key-down while ringing: Silence ringer!");
 
// Silence the ringer.  (It's safe to call this
// even if the ringer has already been silenced.)
telecomManager.silenceRinger();
 
// And *don't* pass this key thru to the current activity
// (which is probably the InCallScreen.)
result &= ~ACTION_PASS_TO_USER;
break;
}
}
int audioMode = AudioManager.MODE_NORMAL;
try {
audioMode = getAudioService().getMode();
} catch (Exception e) {
Log.e(TAG, "Error getting AudioService in interceptKeyBeforeQueueing.", e);
}
boolean isInCall = (telecomManager != null && telecomManager.isInCall()) ||
audioMode == AudioManager.MODE_IN_COMMUNICATION;
if (isInCall && (result & ACTION_PASS_TO_USER) == 0) {
// If we are in call but we decided not to pass the key to
// the application, just pass it to the session service.
MediaSessionLegacyHelper.getHelper(mContext).sendVolumeKeyEvent(
event, AudioManager.USE_DEFAULT_STREAM_TYPE, false);
break;
}
}
if (mUseTvRouting || mHandleVolumeKeysInWM) {
// Defer special key handlings to
// {@link interceptKeyBeforeDispatching()}.
result |= ACTION_PASS_TO_USER;
} else if ((result & ACTION_PASS_TO_USER) == 0) {
// If we aren't passing to the user and no one else
// handled it send it to the session manager to
// figure out.
MediaSessionLegacyHelper.getHelper(mContext).sendVolumeKeyEvent(
event, AudioManager.USE_DEFAULT_STREAM_TYPE, true);
}
break;
}
 
case KeyEvent.KEYCODE_ENDCALL: {
result &= ~ACTION_PASS_TO_USER;
if (down) {
TelecomManager telecomManager = getTelecommService();
boolean hungUp = false;
if (telecomManager != null) {
hungUp = telecomManager.endCall();
}
if (interactive && !hungUp) {
mEndCallKeyHandled = false;
mHandler.postDelayed(mEndCallLongPress,
ViewConfiguration.get(mContext).getDeviceGlobalActionKeyTimeout());
} else {
mEndCallKeyHandled = true;
}
} else {
if (!mEndCallKeyHandled) {
mHandler.removeCallbacks(mEndCallLongPress);
if (!canceled) {
if ((mEndcallBehavior
& Settings.System.END_BUTTON_BEHAVIOR_HOME) != 0) {
if (goHome()) {
break;
}
}
if ((mEndcallBehavior
& Settings.System.END_BUTTON_BEHAVIOR_SLEEP) != 0) {
goToSleep(event.getEventTime(),
PowerManager.GO_TO_SLEEP_REASON_POWER_BUTTON, 0);
isWakeKey = false;
}
}
}
}
break;
}
 
case KeyEvent.KEYCODE_POWER: {
EventLogTags.writeInterceptPower(
KeyEvent.actionToString(event.getAction()),
mPowerKeyHandled ? 1 : 0, mPowerKeyPressCounter);
// Any activity on the power button stops the accessibility shortcut
cancelPendingAccessibilityShortcutAction();
result &= ~ACTION_PASS_TO_USER;
isWakeKey = false; // wake-up will be handled separately
if (down) {
interceptPowerKeyDown(event, interactive);
} else {
interceptPowerKeyUp(event, interactive, canceled);
}
break;
}
 
case KeyEvent.KEYCODE_SYSTEM_NAVIGATION_DOWN:
// fall through
case KeyEvent.KEYCODE_SYSTEM_NAVIGATION_UP:
// fall through
case KeyEvent.KEYCODE_SYSTEM_NAVIGATION_LEFT:
// fall through
case KeyEvent.KEYCODE_SYSTEM_NAVIGATION_RIGHT: {
result &= ~ACTION_PASS_TO_USER;
interceptSystemNavigationKey(event);
break;
}
 
case KeyEvent.KEYCODE_SLEEP: {
result &= ~ACTION_PASS_TO_USER;
isWakeKey = false;
if (!mPowerManager.isInteractive()) {
useHapticFeedback = false; // suppress feedback if already non-interactive
}
if (down) {
sleepPress();
} else {
sleepRelease(event.getEventTime());
}
break;
}
 
case KeyEvent.KEYCODE_SOFT_SLEEP: {
result &= ~ACTION_PASS_TO_USER;
isWakeKey = false;
if (!down) {
mPowerManagerInternal.setUserInactiveOverrideFromWindowManager();
}
break;
}
 
case KeyEvent.KEYCODE_WAKEUP: {
result &= ~ACTION_PASS_TO_USER;
isWakeKey = true;
break;
}
 
case KeyEvent.KEYCODE_MEDIA_PLAY:
case KeyEvent.KEYCODE_MEDIA_PAUSE:
case KeyEvent.KEYCODE_MEDIA_PLAY_PAUSE:
case KeyEvent.KEYCODE_HEADSETHOOK:
case KeyEvent.KEYCODE_MUTE:
case KeyEvent.KEYCODE_MEDIA_STOP:
case KeyEvent.KEYCODE_MEDIA_NEXT:
case KeyEvent.KEYCODE_MEDIA_PREVIOUS:
case KeyEvent.KEYCODE_MEDIA_REWIND:
case KeyEvent.KEYCODE_MEDIA_RECORD:
case KeyEvent.KEYCODE_MEDIA_FAST_FORWARD:
case KeyEvent.KEYCODE_MEDIA_AUDIO_TRACK: {
if (MediaSessionLegacyHelper.getHelper(mContext).isGlobalPriorityActive()) {
// If the global session is active pass all media keys to it
// instead of the active window.
result &= ~ACTION_PASS_TO_USER;
}
if ((result & ACTION_PASS_TO_USER) == 0) {
// Only do this if we would otherwise not pass it to the user. In that
// case, the PhoneWindow class will do the same thing, except it will
// only do it if the showing app doesn't process the key on its own.
// Note that we need to make a copy of the key event here because the
// original key event will be recycled when we return.
mBroadcastWakeLock.acquire();
Message msg = mHandler.obtainMessage(MSG_DISPATCH_MEDIA_KEY_WITH_WAKE_LOCK,
new KeyEvent(event));
msg.setAsynchronous(true);
msg.sendToTarget();
}
break;
}
 
case KeyEvent.KEYCODE_CALL: {
if (down) {
TelecomManager telecomManager = getTelecommService();
if (telecomManager != null) {
if (telecomManager.isRinging()) {
Log.i(TAG, "interceptKeyBeforeQueueing:"
+ " CALL key-down while ringing: Answer the call!");
telecomManager.acceptRingingCall();
 
// And *don't* pass this key thru to the current activity
// (which is presumably the InCallScreen.)
result &= ~ACTION_PASS_TO_USER;
}
}
}
break;
}
case KeyEvent.KEYCODE_ASSIST: {
final boolean longPressed = event.getRepeatCount() > 0;
if (down && longPressed) {
Message msg = mHandler.obtainMessage(MSG_LAUNCH_ASSIST_LONG_PRESS);
msg.setAsynchronous(true);
msg.sendToTarget();
}
if (!down && !longPressed) {
Message msg = mHandler.obtainMessage(MSG_LAUNCH_ASSIST, event.getDeviceId(),
/* unused */, null /* hint */);
msg.setAsynchronous(true);
msg.sendToTarget();
}
result &= ~ACTION_PASS_TO_USER;
break;
}
case KeyEvent.KEYCODE_VOICE_ASSIST: {
if (!down) {
mBroadcastWakeLock.acquire();
Message msg = mHandler.obtainMessage(MSG_LAUNCH_VOICE_ASSIST_WITH_WAKE_LOCK);
msg.setAsynchronous(true);
msg.sendToTarget();
}
result &= ~ACTION_PASS_TO_USER;
break;
}
case KeyEvent.KEYCODE_WINDOW: {
if (mShortPressOnWindowBehavior == SHORT_PRESS_WINDOW_PICTURE_IN_PICTURE) {
if (mPictureInPictureVisible) {
// Consumes the key only if picture-in-picture is visible to show
// picture-in-picture control menu. This gives a chance to the foreground
// activity to customize PIP key behavior.
if (!down) {
showPictureInPictureMenu(event);
}
result &= ~ACTION_PASS_TO_USER;
}
}
break;
}
}
 
// Intercept the Accessibility keychord for TV (DPAD_DOWN + Back) before the keyevent is
// processed through interceptKeyEventBeforeDispatch since Talkback may consume this event
// before it has a chance to reach that method.
if (mHasFeatureLeanback) {
switch (keyCode) {
case KeyEvent.KEYCODE_DPAD_DOWN:
case KeyEvent.KEYCODE_BACK: {
boolean handled = interceptAccessibilityGestureTv(keyCode, down);
if (handled) {
result &= ~ACTION_PASS_TO_USER;
}
break;
}
}
}
 
// Intercept the Accessibility keychord (CTRL + ALT + Z) for keyboard users.
if (mAccessibilityShortcutController.isAccessibilityShortcutAvailable(isKeyguardLocked())) {
switch (keyCode) {
case KeyEvent.KEYCODE_Z: {
if (down && event.isCtrlPressed() && event.isAltPressed()) {
mHandler.sendMessage(mHandler.obtainMessage(MSG_ACCESSIBILITY_SHORTCUT));
result &= ~ACTION_PASS_TO_USER;
}
break;
}
}
}
 
if (useHapticFeedback) {
performHapticFeedback(HapticFeedbackConstants.VIRTUAL_KEY, false,
"Virtual Key - Press");
}
 
if (isWakeKey) {
wakeUp(event.getEventTime(), mAllowTheaterModeWakeFromKey,
PowerManager.WAKE_REASON_WAKE_KEY, "android.policy:KEY");
}
 
if ((result & ACTION_PASS_TO_USER) != 0) {
// If the key event is targeted to a specific display, then the user is interacting with
// that display. Therefore, give focus to the display that the user is interacting with.
if (!mPerDisplayFocusEnabled
&& displayId != INVALID_DISPLAY && displayId != mTopFocusedDisplayId) {
// An event is targeting a non-focused display. Move the display to top so that
// it can become the focused display to interact with the user.
// This should be done asynchronously, once the focus logic is fully moved to input
// from windowmanager. Currently, we need to ensure the setInputWindows completes,
// which would force the focus event to be queued before the current key event.
// TODO(b/70668286): post call to 'moveDisplayToTop' to mHandler instead
Log.i(TAG, "Moving non-focused display " + displayId + " to top "
+ "because a key is targeting it");
mWindowManagerFuncs.moveDisplayToTop(displayId);
}
}
 
return result;
}
这里负责拦截Power和音量加减键
boolean isWakeKey = (policyFlags & WindowManagerPolicy.FLAG_WAKE) != 0 || event.isWakeKey();
判断是唤醒键 wakeUp(event.getEventTime(), mAllowTheaterModeWakeFromKey,
                    PowerManager.WAKE_REASON_WAKE_KEY, "android.policy:KEY");
唤醒屏幕
 
if (isWakeKey && (!down || !isWakeKeyWhenScreenOff(keyCode))) {
                isWakeKey = false;
   }
 
private boolean isWakeKeyWhenScreenOff(int keyCode) {
switch (keyCode) {
case KeyEvent.KEYCODE_VOLUME_UP:
case KeyEvent.KEYCODE_VOLUME_DOWN:
case KeyEvent.KEYCODE_VOLUME_MUTE:
return mDefaultDisplayPolicy.getDockMode() != Intent.EXTRA_DOCK_STATE_UNDOCKED;
 
case KeyEvent.KEYCODE_MUTE:
case KeyEvent.KEYCODE_HEADSETHOOK:
case KeyEvent.KEYCODE_MEDIA_PLAY:
case KeyEvent.KEYCODE_MEDIA_PAUSE:
case KeyEvent.KEYCODE_MEDIA_PLAY_PAUSE:
case KeyEvent.KEYCODE_MEDIA_STOP:
case KeyEvent.KEYCODE_MEDIA_NEXT:
case KeyEvent.KEYCODE_MEDIA_PREVIOUS:
case KeyEvent.KEYCODE_MEDIA_REWIND:
case KeyEvent.KEYCODE_MEDIA_RECORD:
case KeyEvent.KEYCODE_MEDIA_FAST_FORWARD:
case KeyEvent.KEYCODE_MEDIA_AUDIO_TRACK:
return false;
 
case KeyEvent.KEYCODE_DPAD_UP:
case KeyEvent.KEYCODE_DPAD_DOWN:
case KeyEvent.KEYCODE_DPAD_LEFT:
case KeyEvent.KEYCODE_DPAD_RIGHT:
case KeyEvent.KEYCODE_DPAD_CENTER:
return mWakeOnDpadKeyPress;
 
case KeyEvent.KEYCODE_ASSIST:
return mWakeOnAssistKeyPress;
 
case KeyEvent.KEYCODE_BACK:
return mWakeOnBackKeyPress;
}
 
return true;
}
 
这里判断音量键是否是当返回false时 isWakeKey为false
所以需要修改为:
private boolean isWakeKeyWhenScreenOff(int keyCode) {
switch (keyCode) {
// ignore volume keys unless docked
     -  case KeyEvent.KEYCODE_VOLUME_UP:
     -  case KeyEvent.KEYCODE_VOLUME_DOWN:
     +  /*case KeyEvent.KEYCODE_VOLUME_UP:
     +   case KeyEvent.KEYCODE_VOLUME_DOWN:*/
      case KeyEvent.KEYCODE_VOLUME_MUTE:
          return mDefaultDisplayPolicy.getDockMode() != Intent.EXTRA_DOCK_STATE_UNDOCKED;
      // ignore media and camera keys
      case KeyEvent.KEYCODE_MUTE:
      case KeyEvent.KEYCODE_HEADSETHOOK:
      case KeyEvent.KEYCODE_MEDIA_PLAY:
      case KeyEvent.KEYCODE_MEDIA_PAUSE:
      case KeyEvent.KEYCODE_MEDIA_PLAY_PAUSE:
      case KeyEvent.KEYCODE_MEDIA_STOP:
      case KeyEvent.KEYCODE_MEDIA_NEXT:
      case KeyEvent.KEYCODE_MEDIA_PREVIOUS:
      case KeyEvent.KEYCODE_MEDIA_REWIND:
      case KeyEvent.KEYCODE_MEDIA_RECORD:
      case KeyEvent.KEYCODE_MEDIA_FAST_FORWARD:
      case KeyEvent.KEYCODE_MEDIA_AUDIO_TRACK:
      case KeyEvent.KEYCODE_CAMERA:
          return false;
  }
  return true;
}
```
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124893900)

### 1. 概述

在 11.0 的系统定制化开发中，要求屏蔽掉音量 + 音量 - 的功能，根据系统属性来判断是否响应音量加减的功能，在系统上层中是由 PhoneWindowManage 来管理音量键的功能，  
所以就要看是  
PhoneWindowManage.java 中怎么处理的音量键的功能  
首选看的[源码](https://so.csdn.net/so/search?q=%E6%BA%90%E7%A0%81&spm=1001.2101.3001.7020)关于音量键的处理

### 2. 音量键功能开关 (屏蔽音量加减功能) 的核心代码

```
/frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java

```

### 3. 音量键功能开关 (屏蔽音量加减功能) 的功能分析

### 3.1PhoneWindowManager.java 音量键的处理分析

路径:/frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java  
看下 PhoneWindowManager.java 的关于拦截音量键的功能

```
@Override
     public long interceptKeyBeforeDispatching(IBinder focusedToken, KeyEvent event,
int policyFlags) {
final boolean keyguardOn = keyguardOn();
final int repeatCount = event.getRepeatCount();
final int metaState = event.getMetaState();
final int flags = event.getFlags();
final boolean down = event.getAction() == KeyEvent.ACTION_DOWN;
final boolean canceled = event.isCanceled();
final int displayId = event.getDisplayId();


if (mScreenshotChordEnabled && (flags & KeyEvent.FLAG_FALLBACK) == 0) {
if (mScreenshotChordVolumeDownKeyTriggered && !mScreenshotChordPowerKeyTriggered) {
final long now = SystemClock.uptimeMillis();
final long timeoutTime = mScreenshotChordVolumeDownKeyTime
+ SCREENSHOT_CHORD_DEBOUNCE_DELAY_MILLIS;
if (now < timeoutTime) {
return timeoutTime - now;
}
}
if (keyCode == KeyEvent.KEYCODE_VOLUME_DOWN
&& mScreenshotChordVolumeDownKeyConsumed) {
if (!down) {
mScreenshotChordVolumeDownKeyConsumed = false;
}
return -1;
}
}

// If an accessibility shortcut might be partially complete, hold off dispatching until we
// know if it is complete or not
if (mAccessibilityShortcutController.isAccessibilityShortcutAvailable(false)
&& (flags & KeyEvent.FLAG_FALLBACK) == 0) {
if (mScreenshotChordVolumeDownKeyTriggered ^ mA11yShortcutChordVolumeUpKeyTriggered) {
final long now = SystemClock.uptimeMillis();
final long timeoutTime = (mScreenshotChordVolumeDownKeyTriggered
? mScreenshotChordVolumeDownKeyTime : mA11yShortcutChordVolumeUpKeyTime)
+ SCREENSHOT_CHORD_DEBOUNCE_DELAY_MILLIS;
if (now < timeoutTime) {
return timeoutTime - now;
}
}
if (keyCode == KeyEvent.KEYCODE_VOLUME_DOWN && mScreenshotChordVolumeDownKeyConsumed) {
if (!down) {
mScreenshotChordVolumeDownKeyConsumed = false;
}
return -1;
}
if (keyCode == KeyEvent.KEYCODE_VOLUME_UP && mA11yShortcutChordVolumeUpKeyConsumed) {
if (!down) {
mA11yShortcutChordVolumeUpKeyConsumed = false;
}
return -1;
}
}

// If a ringer toggle chord could be on the way but we're not sure, then tell the dispatcher
// to wait a little while and try again later before dispatching.
if (mRingerToggleChord != VOLUME_HUSH_OFF && (flags & KeyEvent.FLAG_FALLBACK) == 0) {
if (mA11yShortcutChordVolumeUpKeyTriggered && !mScreenshotChordPowerKeyTriggered) {
final long now = SystemClock.uptimeMillis();
final long timeoutTime = mA11yShortcutChordVolumeUpKeyTime
+ SCREENSHOT_CHORD_DEBOUNCE_DELAY_MILLIS;
if (now < timeoutTime) {
return timeoutTime - now;
}
}
if (keyCode == KeyEvent.KEYCODE_VOLUME_UP && mA11yShortcutChordVolumeUpKeyConsumed) {
if (!down) {
mA11yShortcutChordVolumeUpKeyConsumed = false;
}
return -1;
}
}

// Cancel any pending meta actions if we see any other keys being pressed between the down
// of the meta key and its corresponding up.
if (mPendingMetaAction && !KeyEvent.isMetaKey(keyCode)) {
mPendingMetaAction = false;
}
// Any key that is not Alt or Meta cancels Caps Lock combo tracking.
if (mPendingCapsLockToggle && !KeyEvent.isMetaKey(keyCode) && !KeyEvent.isAltKey(keyCode)) {
mPendingCapsLockToggle = false;
}

// First we always handle the home key here, so applications
// can never break it, although if keyguard is on, we do let
// it handle it, because that gives us the correct 5 second
// timeout.
if (keyCode == KeyEvent.KEYCODE_HOME) {
DisplayHomeButtonHandler handler = mDisplayHomeButtonHandlers.get(displayId);
if (handler == null) {
handler = new DisplayHomeButtonHandler(displayId);
mDisplayHomeButtonHandlers.put(displayId, handler);
}
return handler.handleHomeButton(focusedToken, event);
} else if (keyCode == KeyEvent.KEYCODE_MENU) {
// Hijack modified menu keys for debugging features
final int chordBug = KeyEvent.META_SHIFT_ON;

if (down && repeatCount == 0) {
if (mEnableShiftMenuBugReports && (metaState & chordBug) == chordBug) {
Intent intent = new Intent(Intent.ACTION_BUG_REPORT);
mContext.sendOrderedBroadcastAsUser(intent, UserHandle.CURRENT,
null, null, null, 0, null, null);
return -1;
}
}
} else if (keyCode == KeyEvent.KEYCODE_SEARCH) {
if (down) {
if (repeatCount == 0) {
mSearchKeyShortcutPending = true;
mConsumeSearchKeyUp = false;
}
} else {
mSearchKeyShortcutPending = false;
if (mConsumeSearchKeyUp) {
mConsumeSearchKeyUp = false;
return -1;
}
}
return 0;
} else if (keyCode == KeyEvent.KEYCODE_APP_SWITCH) {
if (!keyguardOn) {
if (down && repeatCount == 0) {
preloadRecentApps();
} else if (!down) {
toggleRecentApps();
}
}
return -1;
} else if (keyCode == KeyEvent.KEYCODE_N && event.isMetaPressed()) {
if (down) {
IStatusBarService service = getStatusBarService();
if (service != null) {
try {
service.expandNotificationsPanel();
} catch (RemoteException e) {
// do nothing.
}
}
}
} else if (keyCode == KeyEvent.KEYCODE_S && event.isMetaPressed()
&& event.isCtrlPressed()) {
if (down && repeatCount == 0) {
int type = event.isShiftPressed() ? TAKE_SCREENSHOT_SELECTED_REGION
: TAKE_SCREENSHOT_FULLSCREEN;
mScreenshotRunnable.setScreenshotType(type);
mScreenshotRunnable.setScreenshotSource(SCREENSHOT_KEY_OTHER);
mHandler.post(mScreenshotRunnable);
return -1;
}
} else if (keyCode == KeyEvent.KEYCODE_SLASH && event.isMetaPressed()) {
if (down && repeatCount == 0 && !isKeyguardLocked()) {
toggleKeyboardShortcutsMenu(event.getDeviceId());
}
} else if (keyCode == KeyEvent.KEYCODE_ASSIST) {
Slog.wtf(TAG, "KEYCODE_ASSIST should be handled in interceptKeyBeforeQueueing");
return -1;
} else if (keyCode == KeyEvent.KEYCODE_VOICE_ASSIST) {
Slog.wtf(TAG, "KEYCODE_VOICE_ASSIST should be handled in interceptKeyBeforeQueueing");
return -1;
} else if (keyCode == KeyEvent.KEYCODE_SYSRQ) {
if (down && repeatCount == 0) {
mScreenshotRunnable.setScreenshotType(TAKE_SCREENSHOT_FULLSCREEN);
mScreenshotRunnable.setScreenshotSource(SCREENSHOT_KEY_OTHER);
mHandler.post(mScreenshotRunnable);
}
return -1;
} else if (keyCode == KeyEvent.KEYCODE_BRIGHTNESS_UP
|| keyCode == KeyEvent.KEYCODE_BRIGHTNESS_DOWN) {
if (down) {
int direction = keyCode == KeyEvent.KEYCODE_BRIGHTNESS_UP ? 1 : -1;

// Disable autobrightness if it's on
int auto = Settings.System.getIntForUser(
mContext.getContentResolver(),
Settings.System.SCREEN_BRIGHTNESS_MODE,
Settings.System.SCREEN_BRIGHTNESS_MODE_MANUAL,
UserHandle.USER_CURRENT_OR_SELF);
if (auto != 0) {
Settings.System.putIntForUser(mContext.getContentResolver(),
Settings.System.SCREEN_BRIGHTNESS_MODE,
Settings.System.SCREEN_BRIGHTNESS_MODE_MANUAL,
UserHandle.USER_CURRENT_OR_SELF);
}
float minFloat = mPowerManager.getBrightnessConstraint(
PowerManager.BRIGHTNESS_CONSTRAINT_TYPE_MINIMUM);
float maxFloat = mPowerManager.getBrightnessConstraint(
PowerManager.BRIGHTNESS_CONSTRAINT_TYPE_MAXIMUM);
float stepFloat = (maxFloat - minFloat) / BRIGHTNESS_STEPS * direction;
float brightnessFloat = Settings.System.getFloatForUser(
mContext.getContentResolver(), Settings.System.SCREEN_BRIGHTNESS_FLOAT,
mPowerManager.getBrightnessConstraint(
PowerManager.BRIGHTNESS_CONSTRAINT_TYPE_DEFAULT),
UserHandle.USER_CURRENT_OR_SELF);
brightnessFloat += stepFloat;
// Make sure we don't go beyond the limits.
brightnessFloat = Math.min(maxFloat, brightnessFloat);
brightnessFloat = Math.max(minFloat, brightnessFloat);

Settings.System.putFloatForUser(mContext.getContentResolver(),
Settings.System.SCREEN_BRIGHTNESS_FLOAT, brightnessFloat,
UserHandle.USER_CURRENT_OR_SELF);
startActivityAsUser(new Intent(Intent.ACTION_SHOW_BRIGHTNESS_DIALOG),
UserHandle.CURRENT_OR_SELF);
}
return -1;
} else if (keyCode == KeyEvent.KEYCODE_VOLUME_UP
|| keyCode == KeyEvent.KEYCODE_VOLUME_DOWN
|| keyCode == KeyEvent.KEYCODE_VOLUME_MUTE) {
if (mUseTvRouting || mHandleVolumeKeysInWM) {
// On TVs or when the configuration is enabled, volume keys never
// go to the foreground app.
dispatchDirectAudioEvent(event);
return -1;
}

// If the device is in VR mode and keys are "internal" (e.g. on the side of the
// device), then drop the volume keys and don't forward it to the application/dispatch
// the audio event.
if (mDefaultDisplayPolicy.isPersistentVrModeEnabled()) {
final InputDevice d = event.getDevice();
if (d != null && !d.isExternal()) {
return -1;
}
}
} else if (keyCode == KeyEvent.KEYCODE_TAB && event.isMetaPressed()) {
// Pass through keyboard navigation keys.
return 0;
}
.....代码省略

```

从 PhoneWindowManager 代码可以看出  
interceptKeyBeforeDispatching(）处理音量按键事件根据音量 + 和音量 - 来处理增加音量 + 和音量减  
所以修改为:

```
else if (keyCode == KeyEvent.KEYCODE_VOLUME_UP
|| keyCode == KeyEvent.KEYCODE_VOLUME_DOWN
|| keyCode == KeyEvent.KEYCODE_VOLUME_MUTE) {

     //这里就是处理音量键的地方
+                       String aTrue = SystemProperties.get("persist.sys.roco.volumekey.enable", "true");
+                       Log.e("dong", "volume:"+aTrue);
+                               if(!"true".equals(aTrue)){
+                               return -1;
+                       }

if (mUseTvRouting || mHandleVolumeKeysInWM) {
// On TVs or when the configuration is enabled, volume keys never
// go to the foreground app.
dispatchDirectAudioEvent(event);
return -1;
}

```

2. 在 devices 下的 system.prop 中定义 persist.sys.roco.volumekey.enable 属性值
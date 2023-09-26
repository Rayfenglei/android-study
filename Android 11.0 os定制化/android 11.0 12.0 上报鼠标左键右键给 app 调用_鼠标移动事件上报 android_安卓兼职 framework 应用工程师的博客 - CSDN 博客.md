> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124828530)

### 1. 概述

在 11.0 12.0 定制化开发时，有些产品是有多个 usb 口的，可以通过 usb 口连接鼠标键盘等设备，这样可以  
通过外设来操作产品，在一款设备中，有功能需求需要系统上报鼠标左键右键给 app 做处理，所以需要  
增加 KeyEvent 事件上报给上层

### 2. 上报鼠标左键右键给 app 调用的核心类

```
frameworks\base\core\java\android\view\KeyEvent.java
frameworks/base/core/java/android/app/Activity.java
frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java

```

### 3. 上报鼠标左键右键给 app 调用的核心功能实现和分析

在系统中鼠标左键和右键事件  
也就是 KEYCODE_META_LEFT 和 KEYCODE_META_RIGHT 这两个事件  
下面看这两个事件是怎么分发的

### 3.1 在 KeyEvent.java 里面鼠标左右键事件处理分析

```
frameworks\base\core\java\android\view\KeyEvent.java
      /** @hide */
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
  
      /** @hide */
      public static final boolean isMetaKey(int keyCode) {
          return keyCode == KeyEvent.KEYCODE_META_LEFT || keyCode == KeyEvent.KEYCODE_META_RIGHT;
      }
  
      /** @hide */
      public static final boolean isAltKey(int keyCode) {
          return keyCode == KeyEvent.KEYCODE_ALT_LEFT || keyCode == KeyEvent.KEYCODE_ALT_RIGHT;
      }

```

在上述代码中在 boolean isMetaKey(int keyCode) 根据 keycode 来判断是否是鼠标事件  
通过查询代码发现在 PhoneWindowManager.java 对这个事件进行处理

### 3.2PhoneWindowManager.java 中相关代码调用

```
public long interceptKeyBeforeDispatching(IBinder focusedToken, KeyEvent event,
              int policyFlags) {
          final boolean keyguardOn = keyguardOn();
          final int keyCode = event.getKeyCode();
          final int repeatCount = event.getRepeatCount();
          final int metaState = event.getMetaState();
          final int flags = event.getFlags();
          final boolean down = event.getAction() == KeyEvent.ACTION_DOWN;
          final boolean canceled = event.isCanceled();
          final int displayId = event.getDisplayId();
  ....
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
          }
         if (KeyEvent.isMetaKey(keyCode)) {
              if (down) {
                  mPendingMetaAction = true;
              } else if (mPendingMetaAction) {
                  launchAssistAction(Intent.EXTRA_ASSIST_INPUT_HINT_KEYBOARD, event.getDeviceId());
              }
              return -1;
          }
  

```

在事件中的  
if (KeyEvent.isMetaKey(keyCode)) {  
if (down) {  
mPendingMetaAction = true;  
} else if (mPendingMetaAction) {  
launchAssistAction(Intent.EXTRA_ASSIST_INPUT_HINT_KEYBOARD, event.getDeviceId());  
}  
return -1;  
}  
对于如果是 KeyEvent.isMetaKey(keyCode)  
返回 - 1 表示不处理事件 所以就不会上报了，上层事件就收不到事件了  
所以具体修改为:  
/** @hide */

```
public static final boolean isMetaKey(int keyCode) {
- return keyCode == KeyEvent.KEYCODE_META_LEFT || keyCode == KeyEvent.KEYCODE_META_RIGHT;
+ false;
}
在 isModifierKey中 也处理这两个事件
public static boolean isModifierKey(int keyCode) {
switch (keyCode) {
case KEYCODE_SHIFT_LEFT:
case KEYCODE_SHIFT_RIGHT:
case KEYCODE_ALT_LEFT:
case KEYCODE_ALT_RIGHT:
case KEYCODE_CTRL_LEFT:
case KEYCODE_CTRL_RIGHT:
- case KEYCODE_META_LEFT:
- case KEYCODE_META_RIGHT:
case KEYCODE_SYM:
case KEYCODE_NUM:
case KEYCODE_FUNCTION:
return true;
default:
return false;
}
}

```

所以需要注释掉  
同样的在 PhoneWindowManager.java 中 处理 KeyEvent.isModifierKey 事件  
所以只能把这两个事件作为系统事件 才会被 Activity 处理

```
public static final boolean isSystemKey(int keyCode) {
switch (keyCode) {
case KeyEvent.KEYCODE_MENU:
case KeyEvent.KEYCODE_SOFT_RIGHT:
case KeyEvent.KEYCODE_HOME:
case KeyEvent.KEYCODE_BACK:
case KeyEvent.KEYCODE_CALL:
case KeyEvent.KEYCODE_ENDCALL:
case KeyEvent.KEYCODE_VOLUME_UP:
case KeyEvent.KEYCODE_VOLUME_DOWN:
case KeyEvent.KEYCODE_VOLUME_MUTE:
case KeyEvent.KEYCODE_MUTE:
case KeyEvent.KEYCODE_POWER:
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
case KeyEvent.KEYCODE_CAMERA:
case KeyEvent.KEYCODE_FOCUS:
case KeyEvent.KEYCODE_SEARCH:
case KeyEvent.KEYCODE_BRIGHTNESS_DOWN:
case KeyEvent.KEYCODE_BRIGHTNESS_UP:
case KeyEvent.KEYCODE_MEDIA_AUDIO_TRACK:
case KeyEvent.KEYCODE_SYSTEM_NAVIGATION_UP:
case KeyEvent.KEYCODE_SYSTEM_NAVIGATION_DOWN:
case KeyEvent.KEYCODE_SYSTEM_NAVIGATION_LEFT:
case KeyEvent.KEYCODE_SYSTEM_NAVIGATION_RIGHT:
+ case KEYCODE_META_LEFT:
+ case KEYCODE_META_RIGHT:
return true;
}
    return false;
}


/** Is this a system key?  System keys can not be used for menu shortcuts.
 */
public final boolean isSystem() {
    return isSystemKey(mKeyCode);
}


在 Activity.java
public boolean onKeyDown(int keyCode, KeyEvent event)  {
if (keyCode == KeyEvent.KEYCODE_BACK) {
if (getApplicationInfo().targetSdkVersion
>= Build.VERSION_CODES.ECLAIR) {
event.startTracking();
} else {
onBackPressed();
}
return true;
}
    if (mDefaultKeyMode == DEFAULT_KEYS_DISABLE) {
        return false;
    } else if (mDefaultKeyMode == DEFAULT_KEYS_SHORTCUT) {
        Window w = getWindow();
        if (w.hasFeature(Window.FEATURE_OPTIONS_PANEL) &&
                w.performPanelShortcut(Window.FEATURE_OPTIONS_PANEL, keyCode, event,
                        Menu.FLAG_ALWAYS_PERFORM_CLOSE)) {
            return true;
        }
        return false;
    } else if (keyCode == KeyEvent.KEYCODE_TAB) {
        // Don't consume TAB here since it's used for navigation. Arrow keys
        // aren't considered "typing keys" so they already won't get consumed.
        return false;
    } else {
        // Common code for DEFAULT_KEYS_DIALER & DEFAULT_KEYS_SEARCH_*
        boolean clearSpannable = false;
        boolean handled;
        if ((event.getRepeatCount() != 0) || event.isSystem()) {
            clearSpannable = true;
            handled = false;
        } else {
            handled = TextKeyListener.getInstance().onKeyDown(
                    null, mDefaultKeySsb, keyCode, event);
            if (handled && mDefaultKeySsb.length() > 0) {
                // something useable has been typed - dispatch it now.

                final String str = mDefaultKeySsb.toString();
                clearSpannable = true;

                switch (mDefaultKeyMode) {
                case DEFAULT_KEYS_DIALER:
                    Intent intent = new Intent(Intent.ACTION_DIAL,  Uri.parse("tel:" + str));
                    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                    startActivity(intent);
                    break;
                case DEFAULT_KEYS_SEARCH_LOCAL:
                    startSearch(str, false, null, false);
                    break;
                case DEFAULT_KEYS_SEARCH_GLOBAL:
                    startSearch(str, false, null, true);
                    break;
                }
            }
        }
        if (clearSpannable) {
            mDefaultKeySsb.clear();
            mDefaultKeySsb.clearSpans();
            Selection.setSelection(mDefaultKeySsb,0);
        }
        return handled;
    }
}

```

可以看到 event.isSystem() 时 handled=false；  
表示事件继续传下去处理  
所以只有 SystemKey 事件 Activity 中重写的 OnKeyDown 事件会继续处理
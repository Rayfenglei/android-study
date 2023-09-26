> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125250959)

### 1. 概述

在 11.0 的产品开发中，对于 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 导航栏功能开发也是由相当多的需求开发，目前根据产品需求要求在导航栏增加 wifi 键盘亮度等功能，这就需要从导航栏增加 back 和 home 键分析入手然后开始实现这篇博客主要是实现这些功能

### 2.SystemUI 导航栏添加虚拟按键二关键代码

```
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarInflaterView.java
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarView.java
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarFragment.java
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/ButtonDispatcher.java

```

### 3.SystemUI 导航栏添加虚拟按键二相关功能分析和实现

### 3.1NavigationBarInflaterView.java 增加按键布局的修改

NavigationBarInflaterView.java 主要是根据按键的名称来加载相对于的布局，比如在 inflateButtons(）  
中根据对应名称加载对应 xml 布局，然后在导航栏就显示出对应的 ui，然后添加点击事件就实现其功能  
具体实现如下:

```
-- a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarInflaterView.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarInflaterView.java
@@ -60,8 +60,17 @@ public class NavigationBarInflaterView extends FrameLayout
public static final String NAV_BAR_RIGHT = "sysui_nav_bar_right";

public static final String MENU_IME_ROTATE = "menu_ime";
+       public static final String ARROW ="arrow";
+       public static final String ARROWEXPAND ="arrowexpand";
+       public static final String BRIGHTNESS ="brightness";
+       public static final String KEYBOARD ="keyboard";
+       public static final String WIFI ="wifi";
public static final String BACK = "back";
+    public static final String BLUETOOTH = "bluetooth";
public static final String HOME = "home";
+    public static final String CLOCK = "clock";
+    public static final String VOLUME = "volume";
+    public static final String BATTERY = "battery";
public static final String RECENT = "recent";
public static final String NAVSPACE = "space";
public static final String CLIPBOARD = "clipboard";
@@ -202,7 +211,7 @@ public class NavigationBarInflaterView extends FrameLayout
getString(R.string.config_navBarLayout_default);
break;
}
-        android.util.Log.d(TAG, "navLayoutSetting = " + navLayoutSetting);
+        Log.d(TAG, "navLayoutSetting = " + navLayoutSetting);
return navLayoutSetting;
}

@@ -368,7 +377,7 @@ public class NavigationBarInflaterView extends FrameLayout
}

private void addGravitySpacer(LinearLayout layout) {
-        layout.addView(new Space(mContext), new LinearLayout.LayoutParams(0, 0, 1));
+        layout.addView(new Space(mContext), new LinearLayout.LayoutParams(0, 0, 0.2f));
}

private void inflateButtons(String[] buttons, ViewGroup parent, boolean landscape,
@@ -392,7 +401,6 @@ public class NavigationBarInflaterView extends FrameLayout
LayoutInflater inflater = landscape ? mLandscapeInflater : mLayoutInflater;
View v = createView(buttonSpec, parent, inflater);
if (v == null) return null;
-
v = applySize(v, buttonSpec, landscape, start);
parent.addView(v);
addToDispatchers(v);
@@ -473,6 +481,24 @@ public class NavigationBarInflaterView extends FrameLayout
v = inflater.inflate(R.layout.home, parent, false);
} else if (BACK.equals(button)) {
v = inflater.inflate(R.layout.back, parent, false);
+        } else if (BLUETOOTH.equals(button)) {
+            v = inflater.inflate(R.layout.bluetooth, parent, false);
+        } else if (CLOCK.equals(button)) {
+            v = inflater.inflate(R.layout.clock, parent, false);
+        } else if (BATTERY.equals(button)) {
+            v = inflater.inflate(R.layout.battery, parent, false);
+        } else if (ARROW.equals(button)) {
+            v = inflater.inflate(R.layout.arrow, parent, false);
+        } else if (ARROWEXPAND.equals(button)) {
+            v = inflater.inflate(R.layout.arrowexpand, parent, false);
+        } else if (BRIGHTNESS.equals(button)) {
+            v = inflater.inflate(R.layout.brightness, parent, false);
+        } else if (KEYBOARD.equals(button)) {
+            v = inflater.inflate(R.layout.keyboard, parent, false);
+        } else if (WIFI.equals(button)) {
+            v = inflater.inflate(R.layout.wifi, parent, false);
+        } else if (VOLUME.equals(button)) {
+            v = inflater.inflate(R.layout.volume, parent, false);
} else if (RECENT.equals(button)) {
v = inflater.inflate(R.layout.recent_apps, parent, false);
} else if (MENU_IME_ROTATE.equals(button)) {
@@ -555,6 +581,7 @@ public class NavigationBarInflaterView extends FrameLayout
private void addToDispatchers(View v) {
if (mButtonDispatchers != null) {
final int indexOfKey = mButtonDispatchers.indexOfKey(v.getId());
+                       Log.e(TAG,"addToDispatchers---indexOfKey:"+indexOfKey);
if (indexOfKey >= 0) {
mButtonDispatchers.valueAt(indexOfKey).addView(v);
}

```

### 3.2 关于 NavigationBarView.java 的实现按键事件调用的增加

在 NavigationBarView 中主要是定义一些按键的点击触摸事件，然后可以通过这些事件在 ButtonDispatchers 对象在 NavigationBarFragment.java 中调用这些事件，做相关的操作  
所有的导航栏事件都是在这里进行添加的，所以定义对应的事件就好

```
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarView.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarView.java
@@ -122,6 +122,12 @@ public class NavigationBarView extends FrameLayout implements
private KeyButtonDrawable mHomeDefaultIcon;
private KeyButtonDrawable mRecentIcon;
private KeyButtonDrawable mDockedIcon;
+    private KeyButtonDrawable mVolumeIcon;
+    private KeyButtonDrawable mBluetoothIcon;
+    private KeyButtonDrawable mArrowIcon;
+       private KeyButtonDrawable mArrowExpandIcon;
+    private KeyButtonDrawable mBrightnessIcon;
+    private KeyButtonDrawable mWifiIcon;

/* UNISOC: Bug 1072090 & 1106464 new feature of dynamic navigationbar @{ */
private KeyButtonDrawable mHideIcon;
@@ -315,10 +321,19 @@ public class NavigationBarView extends FrameLayout implements
mScreenPinningNotify = new ScreenPinningNotify(mContext);
mBarTransitions = new NavigationBarTransitions(this);

-        mButtonDispatchers.put(R.id.back, backButton);
+               mButtonDispatchers.put(R.id.back, backButton);
mButtonDispatchers.put(R.id.home, new ButtonDispatcher(R.id.home));
mButtonDispatchers.put(R.id.home_handle, new ButtonDispatcher(R.id.home_handle));
mButtonDispatchers.put(R.id.recent_apps, new ButtonDispatcher(R.id.recent_apps));
+        mButtonDispatchers.put(R.id.navi_clock, new ButtonDispatcher(R.id.navi_clock));
+        mButtonDispatchers.put(R.id.navi_battery, new ButtonDispatcher(R.id.navi_battery));
+        mButtonDispatchers.put(R.id.navi_volume, new ButtonDispatcher(R.id.navi_volume));
+        mButtonDispatchers.put(R.id.navi_bluetooth, new ButtonDispatcher(R.id.navi_bluetooth));
+        mButtonDispatchers.put(R.id.navi_wifi, new ButtonDispatcher(R.id.navi_wifi));
+        mButtonDispatchers.put(R.id.navi_arrow, new ButtonDispatcher(R.id.navi_arrow));
+        mButtonDispatchers.put(R.id.navi_brightness, new ButtonDispatcher(R.id.navi_brightness));
+        mButtonDispatchers.put(R.id.navi_keyboard, new ButtonDispatcher(R.id.navi_keyboard));
+        mButtonDispatchers.put(R.id.navi_expand, new ButtonDispatcher(R.id.navi_expand));
mButtonDispatchers.put(R.id.ime_switcher, imeSwitcherButton);
mButtonDispatchers.put(R.id.accessibility_button, accessibilityButton);
mButtonDispatchers.put(R.id.rotate_suggestion, rotateSuggestionButton);
@@ -437,10 +452,57 @@ public class NavigationBarView extends FrameLayout implements
return mButtonDispatchers.get(R.id.home);
}

+    public ButtonDispatcher getVolumeButton() {
+        return mButtonDispatchers.get(R.id.navi_volume);
+    }
+
+    public ButtonDispatcher getBluetoothButton() {
+        return mButtonDispatchers.get(R.id.navi_bluetooth);
+    }
+
+    public ButtonDispatcher getWifiButton() {
+        return mButtonDispatchers.get(R.id.navi_wifi);
+    }
+
+    public ButtonDispatcher getArrowButton() {
+        return mButtonDispatchers.get(R.id.navi_arrow);
+    }
+
+    public ButtonDispatcher getArrowExpandButton() {
+        return mButtonDispatchers.get(R.id.navi_expand);
+    }
+
+    public ButtonDispatcher getKeyboardButton() {
+        return mButtonDispatchers.get(R.id.navi_keyboard);
+    }
+
+    public ButtonDispatcher getBrightnessButton() {
+        return mButtonDispatchers.get(R.id.navi_brightness);
+    }
+
public ButtonDispatcher getImeSwitchButton() {
return mButtonDispatchers.get(R.id.ime_switcher);
}

+    public void setArrow(boolean folded) {
+               Log.e("NavBarInflater","folded:"+folded);
+               KeyButtonDrawable bgDrawable = null;
+               if(folded){
+                  //getArrowExpandButton().setVisibility(View.GONE);
+                  getBluetoothButton().setVisibility(View.GONE);
+                  getBrightnessButton().setVisibility(View.GONE);
+                  getVolumeButton().setVisibility(View.GONE);
+                  bgDrawable = getDrawable(R.drawable.arrow_folded);
+               }else{
+                  //getArrowExpandButton().setVisibility(View.VISIBLE);
+                  getBluetoothButton().setVisibility(View.VISIBLE);
+                  getBrightnessButton().setVisibility(View.VISIBLE);
+                  getVolumeButton().setVisibility(View.VISIBLE);
+                  bgDrawable = getDrawable(R.drawable.arrow_expanded);
+               }
+               getArrowButton().setImageDrawable(bgDrawable);
+    }
+
public ButtonDispatcher getAccessibilityButton() {
return mButtonDispatchers.get(R.id.accessibility_button);
}
@@ -487,7 +549,13 @@ public class NavigationBarView extends FrameLayout implements
final boolean orientationChange = oldConfig.orientation != mConfiguration.orientation;
final boolean densityChange = oldConfig.densityDpi != mConfiguration.densityDpi;
final boolean dirChange = oldConfig.getLayoutDirection() != mConfiguration.getLayoutDirection();
-
+               mVolumeIcon = getDrawable(R.drawable.ic_lock_silent_mode_off);
+               mBluetoothIcon = getDrawable(R.drawable.bluetooth_tray_icon_on);
+               mArrowIcon = getDrawable(R.drawable.arrow_folded);
+               mBrightnessIcon = getDrawable(R.drawable.brightness);
+               mWifiIcon = getDrawable(R.drawable.wifi_normal_level_0);
+               //mKeyboardIcon = getDrawable(R.drawable.input_method_keyboard_icon);
+               mArrowExpandIcon = getDrawable(R.drawable.arrow_expanded);
if (orientationChange || densityChange) {
mDockedIcon = getDrawable(R.drawable.ic_sysbar_docked);
mHomeDefaultIcon = getHomeDrawable();
@@ -660,6 +728,13 @@ public class NavigationBarView extends FrameLayout implements
}
getHomeButton().setImageDrawable(homeIcon);
getBackButton().setImageDrawable(backIcon);
+               getVolumeButton().setImageDrawable(mVolumeIcon);
+               getBluetoothButton().setImageDrawable(mBluetoothIcon);
+               getWifiButton().setImageDrawable(mWifiIcon);
+               getBrightnessButton().setImageDrawable(mBrightnessIcon);
+               getArrowButton().setImageDrawable(mArrowIcon);
+               //getKeyboardButton().setImageDrawable(mKeyboardIcon);
+               getArrowExpandButton().setImageDrawable(mArrowExpandIcon);

updateRecentsIcon();

@@ -708,6 +783,10 @@ public class NavigationBarView extends FrameLayout implements
getBackButton().setVisibility(disableBack      ? View.INVISIBLE : View.VISIBLE);
getHomeButton().setVisibility(disableHome      ? View.INVISIBLE : View.VISIBLE);
getRecentsButton().setVisibility(disableRecent ? View.INVISIBLE : View.VISIBLE);
+               //getBluetoothButton().setVisibility(View.GONE);
+               //getBrightnessButton().setVisibility(View.GONE);
+               //getVolumeButton().setVisibility(View.GONE);
+               getArrowExpandButton().setVisibility(View.GONE);

/* UNISOC: Bug 1072090,1134144,1134237 new feature of dynamic navigationbar @{ */
if(mSupportDynamicBar){

```

### 3.3NavigationBarFragment.java 的关于增加虚拟按键的事件

NavigationBarFragment.java 主要负责对导航栏布局的添加，在这里也是需要添加按键的点击触摸事件的，通过事件来做相关的操作 home 键实现 home 功能 back 键实现返回功能，等所以在这里增加的虚拟按键就需要增加对应的功能

```
diff --git a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarFragment.java b/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarFragment.java
index 80a9bedf79..37d1d53474 100755
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarFragment.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarFragment.java
@@ -80,7 +80,6 @@ import android.view.accessibility.AccessibilityManager;
import android.view.accessibility.AccessibilityManager.AccessibilityServicesStateChangeListener;

import androidx.annotation.VisibleForTesting;
-
import com.android.internal.logging.MetricsLogger;
import com.android.internal.logging.nano.MetricsProto.MetricsEvent;
import com.android.internal.util.LatencyTracker;
@@ -106,16 +105,41 @@ import com.android.systemui.statusbar.policy.AccessibilityManagerWrapper;
import com.android.systemui.statusbar.policy.DeviceProvisionedController;
import com.android.systemui.statusbar.policy.KeyButtonView;
import com.android.systemui.util.LifecycleFragment;
-
import com.sprd.systemui.util.SprdPowerManagerUtil;
-
import java.io.FileDescriptor;
import java.io.PrintWriter;
import java.util.List;
import java.util.Locale;
import java.util.function.Consumer;
-
import javax.inject.Inject;
+import android.view.LayoutInflater;
+
+import android.content.pm.ApplicationInfo;
+import android.content.pm.PackageManager;
+import android.graphics.drawable.ColorDrawable;
+import android.provider.Settings;
+import android.text.TextUtils;
+import android.util.Log;
+import android.view.Gravity;
+import android.view.LayoutInflater;
+import android.view.View;
+import android.view.ViewGroup;
+import android.view.inputmethod.InputMethodInfo;
+import android.view.inputmethod.InputMethodManager;
+import android.widget.Button;
+import android.widget.ImageView;
+import android.widget.PopupWindow;
+import android.widget.RadioButton;
+import android.widget.RadioGroup;
+import android.widget.RelativeLayout;
+import java.util.HashMap;
+import java.util.List;
+import java.util.Map;
+import android.view.Window;
+import android.app.Dialog;
+
+
+

/**
* Fragment containing the NavigationBarFragment. Contains logic for what happens
@@ -150,7 +174,7 @@ public class NavigationBarFragment extends LifecycleFragment implements Callback
private MagnificationContentObserver mMagnificationObserver;
private ContentResolver mContentResolver;
private boolean mAssistantAvailable;
-
+       private boolean mIsFolded = true;
private int mDisabledFlags1;
private int mDisabledFlags2;
private StatusBar mStatusBar;
@@ -675,6 +699,24 @@ public class NavigationBarFragment extends LifecycleFragment implements Callback
homeButton.setOnTouchListener(this::onHomeTouch);
homeButton.setOnLongClickListener(this::onHomeLongClick);

+        ButtonDispatcher arrowButton = mNavigationBarView.getArrowButton();
+        arrowButton.setOnTouchListener(this::onArrowTouch);
+
+               ButtonDispatcher keyboardButton = mNavigationBarView.getKeyboardButton();
+               keyboardButton.setOnTouchListener(this::onKeyboardTouch);
+
+               ButtonDispatcher wifiButton = mNavigationBarView.getWifiButton();
+               wifiButton.setOnTouchListener(this::onWifiTouch);
+
+               ButtonDispatcher brightnessButton = mNavigationBarView.getBrightnessButton();
+               brightnessButton.setOnTouchListener(this::onBrightnessTouch);
+
+               ButtonDispatcher volumeButton = mNavigationBarView.getVolumeButton();
+               volumeButton.setOnTouchListener(this::onVolumeTouch);
+
+               ButtonDispatcher bluetoothButton = mNavigationBarView.getBluetoothButton();
+               bluetoothButton.setOnTouchListener(this::onBluetoothTouch);
+
ButtonDispatcher accessibilityButton = mNavigationBarView.getAccessibilityButton();
accessibilityButton.setOnClickListener(this::onAccessibilityClick);
accessibilityButton.setOnLongClickListener(this::onAccessibilityLongClick);
@@ -783,6 +825,73 @@ public class NavigationBarFragment extends LifecycleFragment implements Callback
mCommandQueue.toggleRecentApps();
}

+       private boolean onArrowTouch(View v, MotionEvent event) {
+                if (event.getAction() == MotionEvent.ACTION_UP) {
+                        mIsFolded = !mIsFolded;
+                        mNavigationBarView.setArrow(mIsFolded);
+                }
+                return false;
+       }
+
+       private boolean onBrightnessTouch(View v, MotionEvent event) {
+                if (event.getAction() == MotionEvent.ACTION_UP) {
+                Intent intent = new Intent("com.android.intent.action.SHOW_BRIGHTNESS_DIALOG");
+                intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK );
+                getContext().startActivity(intent);
+                }
+                return false;
+       }
+
+       private boolean onVolumeTouch(View v, MotionEvent event) {
+                if (event.getAction() == MotionEvent.ACTION_UP) {
+                Intent intent = new Intent(Settings.ACTION_SOUND_SETTINGS);
+                intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK );
+                getContext().startActivity(intent);
+                }
+                return false;
+       }
+
+       private boolean onBluetoothTouch(View v, MotionEvent event) {
+                if (event.getAction() == MotionEvent.ACTION_UP) {
+                Intent intent = new Intent("com.android.intent.action.SHOW_BLUETOOTH_DETAIL");
+                intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK );
+                getContext().startActivity(intent);
+                }
+                return false;
+       }
+
+       private boolean onKeyboardTouch(View v, MotionEvent event) {
+                if (event.getAction() == MotionEvent.ACTION_UP) {
+                        int width = mNavigationBarView.getKeyboardButton().getWidth();
+                        int[] location = new int[2];
+                        mNavigationBarView.getKeyboardButton().getLocationOnScreen(location);
+                        int location_x = location[0];
+                        int location_y = location[1];
+                        Intent keyboard_intent = new Intent("com.android.systemui.keyboard");
+                        keyboard_intent.putExtra("width",width);
+             keyboard_intent.putExtra("location_x",location_x);
+                        keyboard_intent.putExtra("location_y",location_y);
+                        getContext().sendBroadcast(keyboard_intent);
+                }
+                return false;
+       }
+
+       private boolean onWifiTouch(View v, MotionEvent event) {
+                if (event.getAction() == MotionEvent.ACTION_UP) {
+                        int width = mNavigationBarView.getWifiButton().getWidth();
+                        int[] location = new int[2];
+                        mNavigationBarView.getWifiButton().getLocationOnScreen(location);
+                        int location_x = location[0];
+                        int location_y = location[1];
+                        Intent keyboard_intent = new Intent("com.android.systemui.wifi");
+                        keyboard_intent.putExtra("width",width);
+             keyboard_intent.putExtra("location_x",location_x);
+                        keyboard_intent.putExtra("location_y",location_y);
+                        getContext().sendBroadcast(keyboard_intent);
+                }
+                return false;
+       }
+
private boolean onLongPressBackHome(View v) {
return onLongPressNavigationButtons(v, R.id.back, R.id.home);
}

```

### 3.4ButtonDispatcher.java 的添加虚拟按键的到集合

在 ButtonDispatcher 中每个按键都是添加到集合中, 根据对应的 id 来获取该对象，所以对于增加的一些属性也可以添加到这里面来满足功能需要，比如获取按键的宽高等  
具体实现如下：

```
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/ButtonDispatcher.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/ButtonDispatcher.java
@@ -122,6 +122,17 @@ public class ButtonDispatcher {
return mAlpha != null ? mAlpha : 1;
}

+       public void getLocationOnScreen(int[] location){
+               mCurrentView.getLocationOnScreen(location);
+       }
+
+       public int getWidth(){
+               return mCurrentView.getWidth();
+       }
+
+       public View getView(){
+               return mCurrentView;
+       }
public KeyButtonDrawable getImageDrawable() {
return mImageDrawable;
}

```

新增布局文件  
volume.xml

```
<com.android.systemui.statusbar.policy.KeyButtonView
xmlns:android="http://schemas.android.com/apk/res/android"
xmlns:systemui="http://schemas.android.com/apk/res-auto"
android:id="@+id/navi_volume"
android:layout_width="@dimen/navigation_key_width"
android:layout_height="match_parent"
android:layout_weight="0"
android:scaleType="center"
android:paddingStart="@dimen/navigation_key_padding"
android:paddingEnd="@dimen/navigation_key_padding"
/>

```
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124889230)

### 1. 概述

在 11.0 产品开发中，原生系统在锁屏界面默认是显示导航栏的，但是由于产品定制是没有导航栏的所以要求在锁屏解锁界面不要导航栏，这就需要看锁屏界面导航栏的加载流程，分析完成功能

### 2.KeyguardHostView 隐藏导航栏的核心代码部分

```
frameworks/base/packages/SystemUI/src/com/android/keyguard/KeyguardHostView.java 
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarFragment.java

```

### 3.KeyguardHostView 隐藏导航栏的功能分析和功能实现

### 3.1 先看锁屏解锁界面 KeyguardHostView.java

思路:  
1. 在进入解锁界面时 OnResume() 标志位只为 0 隐藏状态栏  
2. 在离开解锁界面时，标志位只为 1 显示状态栏

```
diff --git a/frameworks/base/packages/SystemUI/src/com/android/keyguard/KeyguardHostView.java b/frameworks/base/packages/SystemUI/src/com/android/keyguard/KeyguardHostView.java

old mode 100644

new mode 100755

index b5ff148fe7..22c5d4d8bf

--- a/frameworks/base/packages/SystemUI/src/com/android/keyguard/KeyguardHostView.java

+++ b/frameworks/base/packages/SystemUI/src/com/android/keyguard/KeyguardHostView.java

@@ -36,7 +36,7 @@ import com.android.keyguard.KeyguardSecurityContainer.SecurityCallback;

 import com.android.keyguard.KeyguardSecurityModel.SecurityMode;

 import com.android.settingslib.Utils;

 import com.android.systemui.plugins.ActivityStarter.OnDismissAction;

-

+import android.provider.Settings;

 import java.io.File;

 

 /**

@@ -103,7 +103,7 @@ public class KeyguardHostView extends FrameLayout implements SecurityCallback {

     private static final boolean KEYGUARD_MANAGES_VOLUME = false;

     public static final boolean DEBUG = KeyguardConstants.DEBUG;

     private static final String TAG = "KeyguardViewBase";

-

+    private Context mContext;//add code

     private KeyguardSecurityContainer mSecurityContainer;

 

     public KeyguardHostView(Context context) {

@@ -113,6 +113,7 @@ public class KeyguardHostView extends FrameLayout implements SecurityCallback {

     public KeyguardHostView(Context context, AttributeSet attrs) {

         super(context, attrs);

         KeyguardUpdateMonitor.getInstance(context).registerCallback(mUpdateCallback);

+               mContext = context;

     }

```

在 KeyguardHostView 的 dismiss() 当解锁界面消失时调用系统属性值设置为 0 表示要求显示导航栏

```
     @Override

@@ -211,6 +212,12 @@ public class KeyguardHostView extends FrameLayout implements SecurityCallback {

 

     @Override

     public boolean dismiss(boolean authenticated, int targetUserId) {

+               //add code start

+               int showNavButton = Settings.System.getInt(mContext.getContentResolver(),"show_nav_button",1);

+               if(showNavButton == 0){

+                       Settings.System.putInt(mContext.getContentResolver(),"show_nav_button",1);

+               }

+               //add code end

         if (DEBUG) Log.d(TAG , "dismiss and showNextSecurityScreenOrFinish authenticated =" + authenticated  + " caller:"+android.os.Debug.getCallers(12

));

         return mSecurityContainer.showNextSecurityScreenOrFinish(authenticated, targetUserId);

     }





@@ -291,6 +298,17 @@ public class KeyguardHostView extends FrameLayout implements SecurityCallback {

 /**
     * Called when the Keyguard is actively shown on the screen.
     */
    public void onResume() {
        if (DEBUG) Log.d(TAG, "screen on, instance " + Integer.toHexString(hashCode()));
        mSecurityContainer.onResume(KeyguardSecurityView.SCREEN_ON);
        requestFocus();

+               //add code start

+               boolean lock  = mLockPatternUtils.isSecure(KeyguardUpdateMonitor.getCurrentUser());

+               if(lock){

+                       Settings.System.putInt(mContext.getContentResolver(),"show_nav_button",0);

+               }else {

+                       int showNavButton = Settings.System.getInt(mContext.getContentResolver(),"show_nav_button",1);

+                       if(showNavButton == 0){

+                               Settings.System.putInt(mContext.getContentResolver(),"show_nav_button",1);

+                       }

+               }

+               //add code end

     }

```

在 onResume（）当进入解锁界面时调用设置系统属性值为 1 表示需要影藏导航栏  
通过在 onResume（）和 dismiss() 设置不同的系统属性值

### 3.2NavigationBarFragment 导航栏界面控制显示和隐藏导航栏

当[监听](https://so.csdn.net/so/search?q=%E7%9B%91%E5%90%AC&spm=1001.2101.3001.7020)到 show_nav_button 系统属性值变化时，做显示和隐藏导航栏的操作

```
    /**

diff --git a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarFragment.java b/frameworks/base/packages/SystemUI/sr

c/com/android/systemui/statusbar/phone/NavigationBarFragment.java

index 57766f595d..5ef9682f4e 100755

--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarFragment.java

+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarFragment.java

@@ -447,8 +447,45 @@ public class NavigationBarFragment extends LifecycleFragment implements Callback

         if (mScreenDecorations != null) {

             getBarTransitions().addDarkIntensityListener(mScreenDecorations);

         }

+               //add code start

+                  getContext().getContentResolver().registerContentObserver(

+                Settings.System.getUriFor("show_nav_button"), true,

+                mNavigationRightBarShowObserver);

+               //add code end

     }

 

+       //add code start

+    private final ContentObserver mNavigationRightBarShowObserver = new ContentObserver(new Handler()) {

+        @Override

+        public void onChange(boolean selfChange) {

+            boolean lock = Settings.System.getInt(getContext().getContentResolver(), "show_nav_button", 0) == 0;


+            if (lock) {

+                //hide

+                               mNavigationBarView.getBluetoothButton().setVisibility(View.GONE);

+                       mNavigationBarView.getBrightnessButton().setVisibility(View.GONE);

+                       mNavigationBarView.getVolumeButton().setVisibility(View.GONE);

+                               mNavigationBarView.getWifiButton().setVisibility(View.GONE);

+                               mNavigationBarView.getKeyboardButton().setVisibility(View.GONE);

+                               mNavigationBarView.getClock().setVisibility(View.GONE);

+                               mNavigationBarView.getmBattery().setVisibility(View.GONE);

+                               mNavigationBarView.getPanelButton().setVisibility(View.GONE);

+            } else {

+                //show

+                               mNavigationBarView.getBluetoothButton().setVisibility(View.VISIBLE);

+                       mNavigationBarView.getBrightnessButton().setVisibility(View.VISIBLE);

+                       mNavigationBarView.getVolumeButton().setVisibility(View.VISIBLE);

+                               mNavigationBarView.getWifiButton().setVisibility(View.VISIBLE);

+                               mNavigationBarView.getKeyboardButton().setVisibility(View.VISIBLE);

+                               mNavigationBarView.getClock().setVisibility(View.VISIBLE);

+                               mNavigationBarView.getmBattery().setVisibility(View.VISIBLE);

+                               mNavigationBarView.getPanelButton().setVisibility(View.VISIBLE);

+            }

+

+        }

+    };

+   //add code end

+


diff --git a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarView.java b/frameworks/base/packages/SystemUI/src/co

m/android/systemui/statusbar/phone/NavigationBarView.java

index 2be14f33f3..a7f719533d 100755

--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarView.java

+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarView.java

@@ -496,8 +496,12 @@ public class NavigationBarView extends FrameLayout implements

     public ButtonDispatcher getClock() {

         return mButtonDispatchers.get(R.id.navi_clock);

     }


+       public void setArrow(boolean folded) {


                KeyButtonDrawable bgDrawable = null;

                if(folded){

```
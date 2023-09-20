> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124739736)

1. 概述
-----

在 11.0 定制化产品开发中，在系统中默认 use 版本 [adb](https://so.csdn.net/so/search?q=adb&spm=1001.2101.3001.7020) 是关闭的 ，但是产品需求要求 user 版本的软件也要默认打开 adb 连接的，方便抓取 log，等等需求. 这就需要打开 user 模式的调试接口

2.framework user 模式默认打开 adb 功能的核心类
----------------------------------

```
build/make/core/main.mk b/build/make/core/main.mk
frameworks/base/packages/SystemUI/src/com/android/systemui/usb/UsbDebuggingActivity.java
frameworks/base/packages/SystemUI/src/com/android/systemui/usb/UsbPermissionActivity.java
/system/core/adb/Android.mk

```

3.framework user 模式默认打开 adb 功能的核心功能分析和实现
----------------------------------------

解决思路  
1. 打开 main.[mk](https://so.csdn.net/so/search?q=mk&spm=1001.2101.3001.7020) 的 user 调试模式 将 ro.debuggable=1 就可以了  
2. 默认授予 usb 权限 去掉 usb 授权弹窗  
授予 user 模式 调试权限

3.1main.mk 的核心功能分析
------------------

修改如下:

```
diff --git a/build/make/core/main.mk b/build/make/core/main.mk

index 9c04d96f6c..79a0a0c302 100755

--- a/build/make/core/main.mk

+++ b/build/make/core/main.mk

 ifeq ($(user_variant),user)
-    ADDITIONAL_DEFAULT_PROPERTIES += ro.adb.secure=1
+    ADDITIONAL_DEFAULT_PROPERTIES += ro.adb.secure=0
 endif

 else # !enable_target_debugging 
    # Target is less debuggable and adbd is off by default 
-    ADDITIONAL_DEFAULT_PROPERTIES += ro.debuggable=0
+    ADDITIONAL_DEFAULT_PROPERTIES += ro.debuggable=1
 endif # !enable_target_debugging

```

接下来在 adb 连接电脑时 授予 usb 权限 在 SystemUI 中的 UsbDebuggingActivity.java 默认授予 usb 权限  
2. UsbDebuggingActivity.java 授权 usb 权限  
修改如下：

```
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/usb/UsbDebuggingActivity.java

+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/usb/UsbDebuggingActivity.java

@@ -126,9 +126,9 @@ public class UsbDebuggingActivity extends AlertActivity

     @Override
     public void onCreate(Bundle icicle) {
         Window window = getWindow();
         window.addSystemFlags(
                 WindowManager.LayoutParams.SYSTEM_FLAG_HIDE_NON_SYSTEM_OVERLAY_WINDOWS);
         window.setType(WindowManager.LayoutParams.TYPE_SYSTEM_DIALOG);
 
         super.onCreate(icicle);
 
         if (SystemProperties.getInt("service.adb.tcp.port", 0) == 0) {
             mDisconnectedReceiver = new UsbDisconnectedReceiver(this);
             IntentFilter filter = new IntentFilter(UsbManager.ACTION_USB_STATE);
             mBroadcastDispatcher.registerReceiver(mDisconnectedReceiver, filter);
         }
 
         Intent intent = getIntent();
         String fingerprints = intent.getStringExtra("fingerprints");
         mKey = intent.getStringExtra("key");
 
         if (fingerprints == null || mKey == null) {
             finish();
             return;
         }
 
         final AlertController.AlertParams ap = mAlertParams;
         ap.mTitle = getString(R.string.usb_debugging_title);
         ap.mMessage = getString(R.string.usb_debugging_message, fingerprints);
         ap.mPositiveButtonText = getString(R.string.usb_debugging_allow);
         ap.mNegativeButtonText = getString(android.R.string.cancel);
         ap.mPositiveButtonListener = this;
         ap.mNegativeButtonListener = this;
 
         // add "always allow" checkbox
         LayoutInflater inflater = LayoutInflater.from(ap.mContext);
         View checkbox = inflater.inflate(com.android.internal.R.layout.always_use_checkbox, null);
         mAlwaysAllow = (CheckBox)checkbox.findViewById(com.android.internal.R.id.alwaysUse);
         mAlwaysAllow.setText(getString(R.string.usb_debugging_always));
         ap.mView = checkbox;
          window.setCloseOnTouchOutside(false);
  
          setupAlert();
      }
	  private class UsbDisconnectedReceiver extends BroadcastReceiver {
          private final Activity mActivity;
          UsbDisconnectedReceiver(Activity activity) {
              mActivity = activity;
          }
  
          @Override
          public void onReceive(Context content, Intent intent) {
              String action = intent.getAction();
              if (!UsbManager.ACTION_USB_STATE.equals(action)) {
                  return;
              }
              boolean connected = intent.getBooleanExtra(UsbManager.USB_CONNECTED, false);
              if (!connected) {
                  Log.d(TAG, "USB disconnected, notifying service");
                  notifyService(false);
                  mActivity.finish();
              }
          }
      }
   private void notifyService(boolean allow, boolean alwaysAllow) {
          try {
              IBinder b = ServiceManager.getService(ADB_SERVICE);
              IAdbManager service = IAdbManager.Stub.asInterface(b);
              if (allow) {
                  service.allowDebugging(alwaysAllow, mKey);
              } else {
                  service.denyDebugging();
              }
              mServiceNotified = true;
          } catch (Exception e) {
              Log.e(TAG, "Unable to notify Usb service", e);
          }
      }
      

             if (!UsbManager.ACTION_USB_STATE.equals(action)) {

                 return;

             }

-            boolean connected = intent.getBooleanExtra(UsbManager.USB_CONNECTED, false);

+            //boolean connected = intent.getBooleanExtra(UsbManager.USB_CONNECTED, false);

             //modified by elink_qyj start<<<

-            try {

+            /*try {

                 if(android.provider.Settings.Global.getInt(content.getContentResolver(), "elink.admin.adb", 0) == 1){

                     connected = false;

                     IBinder b = ServiceManager.getService(ADB_SERVICE);

@@ -137,11 +137,25 @@ public class UsbDebuggingActivity extends AlertActivity

                 }

             } catch (Exception e) {

                 Log.e(TAG, "Unable to notify Usb service", e);

-            }

+            }*/

             //modified end>>>

-            if (!connected) {

+            boolean connected = false;

+                       if (!connected) {

+                 mActivity.finish();

+             }

+             //usb授权

+

+                       try {

+                                IBinder b = ServiceManager.getService(ADB_SERVICE); 

+                                IAdbManager service = IAdbManager.Stub.asInterface(b);

+                                service.allowDebugging(true, mKey);

+                       } catch (Exception e) {

+                                       Log.e(TAG, "Unable to notify Usb service", e); 

+                       }

+

+            /*if (!connected) {

                 mActivity.finish();

-            }

+            }*/

         }

     }

```

在 onCreate(Bundle icicle) 中注册监听 usb 连接的广播，然后在广播中通过对  
boolean connected = intent.getBooleanExtra(UsbManager.USB_CONNECTED, false) 的判断  
如果为 fasle, 重新通过 Ibinder 通讯挂载 usb, 所以在这里直接赋值为 false，然后直接挂载 usb

3.3UsbPermissionActivity.java 去掉授权弹窗
------------------------------------

修改如下:

```
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/usb/UsbPermissionActivity.java

+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/usb/UsbPermissionActivity.java

@@ -117,7 +117,10 @@ public class UsbPermissionActivity extends AlertActivity

             mClearDefaultHint.setVisibility(View.GONE);

         }

 

-        setupAlert();

+        //setupAlert();

+               Log.d(TAG, "grant usb permission automatically");

+               mPermissionGranted = true;

+       finish();

     }

```

在 onCreate(Bundle icicle) 注释掉关于弹窗 setupAlert(); 的相关代码，然后 finish 掉窗口，  
然后编译发现 adb 已经默认可以连接电脑了

3.4adb 相关系统签名的属性
----------------

/system/core/adb/Android.mk

```
- LOCAL_CFLAGS += -DALLOW_ADBD_NO_AUTH=$(if $(filter userdebug eng,$(TARGET_BUILD_VARIANT)),1,0)
+ LOCAL_CFLAGS += -DALLOW_ADBD_NO_AUTH=$(if $(filter user userdebug eng,$(TARGET_BUILD_VARIANT)),1,0)

```
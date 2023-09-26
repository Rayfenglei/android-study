> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124955141)

### 1. 概述

在 11.0 的产品开发中，在进行 [iot](https://so.csdn.net/so/search?q=iot&spm=1001.2101.3001.7020) 开发过程中, 在插入 usb 设备时会弹出 usb 授权提示框，也带来一些不便，这个需要默认授予 USB 权限，插拔 usb 都不弹出 usb 弹窗所以这要从 usb 授权相关管理页默认给与 usb 权限

### 2. 去掉 usb 授权提示框 默认给予权限的相关代码

```
frameworks/base/packages/SystemUI/src/com/android/systemui/usb/UsbPermissionActivity.java
frameworks/base/packages/SystemUI/src/com/android/systemui/usb/UsbDebuggingActivity.java

```

### 3. 去掉 usb 授权提示框 默认给予权限的相关代码功能分析

### 3.1UsbPermissionActivity.java 关于 usb 授权弹窗的相关代码

在插入 usb 后，通过 [adb](https://so.csdn.net/so/search?q=adb&spm=1001.2101.3001.7020) shell 命令 adb shell dumpsys window w |findstr / |findstr name = 发现 usb 授权窗就是 UsbPermissionActivity 接下来看下相关代码，是怎么弹窗的  
UsbPermissionActivity.java 去掉 usb 授权提示框

```
public class UsbPermissionActivity extends AlertActivity
          implements DialogInterface.OnClickListener, CheckBox.OnCheckedChangeListener {
@Override
     public void onCreate(Bundle icicle) {
         super.onCreate(icicle);
 
         Intent intent = getIntent();
         mDevice = (UsbDevice)intent.getParcelableExtra(UsbManager.EXTRA_DEVICE);
         mAccessory = (UsbAccessory)intent.getParcelableExtra(UsbManager.EXTRA_ACCESSORY);
         mPendingIntent = (PendingIntent)intent.getParcelableExtra(Intent.EXTRA_INTENT);
         mUid = intent.getIntExtra(Intent.EXTRA_UID, -1);
         mPackageName = intent.getStringExtra(UsbManager.EXTRA_PACKAGE);
         boolean canBeDefault = intent.getBooleanExtra(UsbManager.EXTRA_CAN_BE_DEFAULT, false);
 
         PackageManager packageManager = getPackageManager();
         ApplicationInfo aInfo;
         try {
             aInfo = packageManager.getApplicationInfo(mPackageName, 0);
         } catch (PackageManager.NameNotFoundException e) {
             Log.e(TAG, "unable to look up package name", e);
             finish();
             return;
         }
         String appName = aInfo.loadLabel(packageManager).toString();
 
         final AlertController.AlertParams ap = mAlertParams;
         ap.mTitle = appName;
         boolean useRecordWarning = false;
         if (mDevice == null) {
             // Accessory Case
 
             ap.mMessage = getString(R.string.usb_accessory_permission_prompt, appName,
                     mAccessory.getDescription());
             mDisconnectedReceiver = new UsbDisconnectedReceiver(this, mAccessory);
         } else {
             boolean hasRecordPermission =
                     PermissionChecker.checkPermissionForPreflight(
                             this, android.Manifest.permission.RECORD_AUDIO, -1, aInfo.uid,
                             mPackageName)
                             == android.content.pm.PackageManager.PERMISSION_GRANTED;
              boolean isAudioCaptureDevice = mDevice.getHasAudioCapture();
              useRecordWarning = isAudioCaptureDevice && !hasRecordPermission;
  
              int strID = useRecordWarning
                      ? R.string.usb_device_permission_prompt_warn
                      : R.string.usb_device_permission_prompt;
              ap.mMessage = getString(strID, appName, mDevice.getProductName());
              mDisconnectedReceiver = new UsbDisconnectedReceiver(this, mDevice);
  
          }
  
          ap.mPositiveButtonText = getString(android.R.string.ok);
          ap.mNegativeButtonText = getString(android.R.string.cancel);
          ap.mPositiveButtonListener = this;
          ap.mNegativeButtonListener = this;
  
          // Don't show the "always use" checkbox if the USB/Record warning is in effect
          if (!useRecordWarning && canBeDefault && (mDevice != null || mAccessory != null)) {
              // add "open when" checkbox
              LayoutInflater inflater = (LayoutInflater) getSystemService(
                      Context.LAYOUT_INFLATER_SERVICE);
              ap.mView = inflater.inflate(com.android.internal.R.layout.always_use_checkbox, null);
              mAlwaysUse = (CheckBox) ap.mView.findViewById(com.android.internal.R.id.alwaysUse);
              if (mDevice == null) {
                  mAlwaysUse.setText(getString(R.string.always_use_accessory, appName,
                          mAccessory.getDescription()));
              } else {
                  mAlwaysUse.setText(getString(R.string.always_use_device, appName,
                          mDevice.getProductName()));
              }
              mAlwaysUse.setOnCheckedChangeListener(this);
  
              mClearDefaultHint = (TextView)ap.mView.findViewById(
                      com.android.internal.R.id.clearDefaultHint);
              mClearDefaultHint.setVisibility(View.GONE);
          }
  
          setupAlert();
      }

```

在进入 UsbPermissionActivity 的时候在 onCreate 中的 setupAlert() 就是弹出授权框的相关代码 所以去掉授权框  
就需要注释掉相关代码就好 具体修改为:

```
diff --git a/frameworks/base/packages/SystemUI/src/com/android/systemui/usb/UsbPermissionActivity.java b/frameworks/base/packages/SystemUI/src/com/android/systemui/usb/UsbPermissionActivity.java
index 238407a..afeddd4 100644
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/usb/UsbPermissionActivity.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/usb/UsbPermissionActivity.java
@@ -134,6 +134,9 @@ public class UsbPermissionActivity extends AlertActivity
}

-         setupAlert();
+   //setupAlert();
+		Log.d(TAG, "grant usb permission automatically");
+		mPermissionGranted = true;
+    	finish();

}

```

### 3.2 UsbDebuggingActivity.java 中给 usb 授权

分析相关代码

```
public class UsbDebuggingActivity extends AlertActivity
                                    implements DialogInterface.OnClickListener {
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
          public UsbDisconnectedReceiver(Activity activity) {
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
                  mActivity.finish();
              }
          }
      }

```

在 oncreate 中通过注册 UsbDisconnectedReceiver 的监听广播。当 usb 状态变化时，连接 usb 当未连接时关闭当前窗口  
所以可以在此修改默认为 false，然后用 usb 的 binder 通讯默认授权连接 usb

```
diff --git a/frameworks/base/packages/SystemUI/src/com/android/systemui/usb/UsbDebuggingActivity.java b/frameworks/base/packages/SystemUI/src/com/android/systemui/usb/UsbDebuggingActivity.java
index 66d5ee1..05cc9a2 100644
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/usb/UsbDebuggingActivity.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/usb/UsbDebuggingActivity.java
@@ -126,10 +126,20 @@ public class UsbDebuggingActivity extends AlertActivity
if (!UsbManager.ACTION_USB_STATE.equals(action)) {
return;
}
-            boolean connected = intent.getBooleanExtra(UsbManager.USB_CONNECTED, false);
-            if (!connected) {
+            //boolean connected = intent.getBooleanExtra(UsbManager.USB_CONNECTED, false);
+            boolean connected = false;
+			if (!connected) {
mActivity.finish();
}
//增加usb授权的相关代码
+
+			try {
+				 IBinder b = ServiceManager.getService(ADB_SERVICE);
+				 IAdbManager service = IAdbManager.Stub.asInterface(b);
+				 service.allowDebugging(true, mKey);
+				 } catch (Exception e) {
+				 	Log.e(TAG, "Unable to notify Usb service", e);
+				 }
+
}
}

```
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/128176562)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 默认授予 app 的使用 wifi 和流量的权限的核心类](#t1)

[3. 默认授予 app 的使用 wifi 和流量的权限的核心功能分析和实现](#t2)

 [3.1 NetworkDataAlertDialog 关于授权弹窗的分析](#t3)

[3.2 NetworkDataControllerPreferenceController.java 中关于是否弹窗的判断](#t4)

[3.3 AndroidManifest.xml 关于弹窗页面的源码](#t5)

[3.4 NetworkDataControllerActivity 的相关源码分析](#t6)

[3.5 关于默认 wifi 授权的修改](#t7)

1. 概述
-----

  在 11.0 的系统产品开发中，对于 [mtk](https://so.csdn.net/so/search?q=mtk&spm=1001.2101.3001.7020) 平台的系统来说，会在 app 使用 wifi 和移动网络时，会弹出  
权限授权 Dialog 来让默认授权允许使用 wifi 和移动网络，授权以后才能使用网络，项目需求要求  
默认授予 app 使用 wifi 和移动网络，去掉弹窗

效果图如下:

![](https://img-blog.csdnimg.cn/b29bc9ce32ae432f9a09d22fb6047080.png)

2. 默认授予 app 的使用 wifi 和流量的权限的核心类
-------------------------------

```
vendor\mediatek\proprietary\packages\apps\NetworkDataController\NetworkDataControllerService\src\com\mediatek\security\service\NetworkDataAlertDialog.java
vendor\mediatek\proprietary\packages\apps\NetworkDataController\NetworkDataControllerActivity\AndroidManifest.xml
vendor\mediatek\proprietary\packages\apps\NetworkDataController\NetworkDataControllerActivity\src\com\mediatek\security\activity\NetworkDataControllerActivity.java
vendor\mediatek\proprietary\packages\apps\MtkSettings\src\com\mediatek\settings\network\NetworkDataControllerPreferenceController.java
```

3. 默认授予 app 的使用 wifi 和流量的权限的核心功能分析和实现
-------------------------------------

  3.1 NetworkDataAlertDialog 关于授权弹窗的分析
--------------------------------------

```
public class NetworkDataAlertDialog implements OnClickListener,
        OnDismissListener, OnShowListener {
    /**
     * Show a system confirm dialog from service
     *
     * @param record
     *            the PermissionRecord data type
     * @param flag
     *            the flag of the PermissionRecord
     */
    public void ShowDialog() {
        Log.d(TAG, "ShowDialog");
        ContextThemeWrapper contxtThemeWrapper = new ContextThemeWrapper(
                mContext, R.style.PermissionDialog);
 
        final LayoutInflater inflater = LayoutInflater.from(contxtThemeWrapper);
        final View dialogView = inflater.inflate(
                R.layout.notify_dialog_customview, null);
 
        TextView messageText = (TextView) dialogView.findViewById(R.id.message);
 
        String label = NetworkControllerUtils.getApplicationName(mContext,
                mNetworkDataRecord.getPackageName());
 
        String msg = mContext.getString(R.string.notify_dialog_msg_body, label);
        messageText.setText(msg);
        TextView promptText = (TextView) dialogView.findViewById(R.id.prompt);
        promptText.setText(R.string.prompt);
 
        AlertDialog dialog = createDialog();
        dialog.setButton(DialogInterface.BUTTON_POSITIVE,
                mContext.getText(R.string.accept_perm), this);
        dialog.setButton(DialogInterface.BUTTON_NEUTRAL,
                mContext.getText(R.string.deny_perm), this);
        if(!(mIsData ||
             PermControlUtils.isPackageInNoWifiOnlyList(mNetworkDataRecord.getPackageName()))) {
            dialog.setButton(DialogInterface.BUTTON_NEGATIVE,
                mContext.getText(R.string.wifi_only), this);
        }
        dialog.setCancelable(false);
        dialog.setView(dialogView);
        dialog.setOnDismissListener(this);
        dialog.setOnShowListener(this);
        dialog.getWindow().setType(
                WindowManager.LayoutParams.TYPE_SYSTEM_ERROR);
        dialog.show();
    }
 
    private AlertDialog createDialog() {
        Log.d(TAG, "createDialog");
        AlertDialog dialog = new AlertDialog(mContext) {
            @Override
            public void onWindowFocusChanged(boolean hasFocus) {
                if (hasFocus) {
                    setStatusBarEnableStatus(false);
                } else {
                    setStatusBarEnableStatus(true);
                }
            }
        };
 
        return dialog;
    }
```

在 NetworkDataAlertDialog.java 的上述代码中就可以看出，ShowDialog() 就可以在 app 首次启动通过弹窗来弹出是否允许 wifi 访问的 Dialog, 点击允许后，可以访问 wifi, 接下来看下是从哪里弹出 dialog, 怎么判断是否授权的

3.2 NetworkDataControllerPreferenceController.java 中关于是否弹窗的判断
-------------------------------------------------------------

```
public class NetworkDataControllerPreferenceController extends AbstractPreferenceController{
 
    private static final String KEY_NETWORKDATA_PREF = "network_data_controller";
    private static final String NETWORK_DATA_CONTROLLER =
            "com.mediatek.security.NETWORK_DATA_CONTROLLER";
 
    public NetworkDataControllerPreferenceController(Context context) {
        super(context);
 
    }
 
    @Override
    public boolean isAvailable() {
        boolean isCtaSupport = true;
        isCtaSupport = (SystemProperties.getInt("persist.vendor.sys.disable.moms", 0) != 1) &&
            (SystemProperties.getInt("ro.vendor.mtk_mobile_management", 0) == 1);
        Intent intent = new Intent(NETWORK_DATA_CONTROLLER);
        return isCtaSupport && (mContext.getPackageManager().resolveActivity(intent, 0) != null);
    }
 
    @Override
    public String getPreferenceKey() {
        return KEY_NETWORKDATA_PREF;
    }
 
}
```

在 MtkSettings 中的 NetworkDataControllerPreferenceController.java 中可以看到是否有 wifi 授权的判断的 isAvailable() 方法中，其中 ro.vendor.mtk_mobile_management 的值很关键，当默认为 0 的时候，默认授权 wifi, 不需要弹窗授权，否则需要弹窗授权

3.3 AndroidManifest.xml 关于弹窗页面的源码
---------------------------------

```
          <activity
            android:
            android:label="@string/network_data_control_title"
            android:launchMode="singleTask"
            android:permission="com.mediatek.security.NETWORK_DATA_CONTROLLER_ACTIVITY" >
            <intent-filter>
                <action android: />
                <category android: />
            </intent-filter>
        </activity>
```

通过在 NetworkDataController 的 AndroidManifest.xml 中发现，授权弹窗是 NetworkDataControllerActivity.java  
来处理的，接下来看下 NetworkDataControllerActivity 的相关源码

3.4 NetworkDataControllerActivity 的相关源码分析
-----------------------------------------

```
    private void bindService(){
        Intent intent = new Intent("com.mediatek.security.START_SERVICE");
        intent.setClassName("com.mediatek.security.service",
                "com.mediatek.security.service.NetworkDataControllerService");
        mShouldUnbind = bindService(intent, mServiceConnection, Context.BIND_AUTO_CREATE);
    }
 
    private void unbindService(){
        if (mShouldUnbind) {
            unbindService(mServiceConnection);
            mShouldUnbind = false;
        }
    }
    @Override
    protected void onResume() {
        Log.d(TAG, "oncreate() start");
        super.onResume();
        bindService();
        addUI();
        Log.d(TAG, "oncreate() end");
 
    }
```

在 NetworkDataControllerActivity 中的 onResume() 通过启动 NetworkDataControllerService 这个服务  
这个服务里面会根据 app 的是否授权来弹出 wifi 授权框

3.5 关于默认 wifi 授权的修改
-------------------

```
diff --git a/device/mediatek/common/device.mk b/device/mediatek/common/device.mk
old mode 100644 (file)
new mode 100755 (executable)
index b4509d7..62390ac
--- a/device/mediatek/common/device.mk
+++ b/device/mediatek/common/device.mk
@@ -1821,7 +1821,7 @@ endif
 ifeq ($(strip $(MTK_MOBILE_MANAGEMENT)), yes)
   ifneq ($(strip $(BUILD_GMS)), yes)
     ifneq ($(strip $(MTK_GMO_ROM_OPTIMIZE)), yes)
-      PRODUCT_PROPERTY_OVERRIDES += ro.vendor.mtk_mobile_management=1
+      PRODUCT_PROPERTY_OVERRIDES += ro.vendor.mtk_mobile_management=0
     endif
   endif
 endif
diff --git a/device/mediatek/vendor/common/device.mk b/device/mediatek/vendor/common/device.mk
index 034bb7d..15fc7f0 100755 (executable)
--- a/device/mediatek/vendor/common/device.mk
+++ b/device/mediatek/vendor/common/device.mk
@@ -1572,7 +1572,7 @@ endif
 ifeq ($(strip $(MTK_MOBILE_MANAGEMENT)), yes)
   ifneq ($(strip $(BUILD_GMS)), yes)
     ifneq ($(strip $(MTK_GMO_ROM_OPTIMIZE)), yes)
-      PRODUCT_PROPERTY_OVERRIDES += ro.vendor.mtk_mobile_management=1
+      PRODUCT_PROPERTY_OVERRIDES += ro.vendor.mtk_mobile_management=0
     endif
   endif
 endif
```
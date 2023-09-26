> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124706059)

在 11.0 原生系统中对于安装第三方 app 会弹出未知来源弹窗确认以后才允许安装  
这样显得有些麻烦，所以默认是去掉安装未来来源的 要授予未知来源权限的  
1. 在 AppOpsManager.java 中授予未知来源的权限  
路径: frameworks/base/core/java/android/app/AppOpsManager.java

```
/**
This specifies the default mode for each operation.
*/
private static int[] sOpDefaultMode = new int[] {
......
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_DEFAULT, // OP_GET_USAGE_STATS
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_IGNORED, // OP_PROJECT_MEDIA
AppOpsManager.MODE_IGNORED, // OP_ACTIVATE_VPN
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ERRORED,  // OP_MOCK_LOCATION
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,  // OP_TURN_ON_SCREEN
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_ALLOWED,  // OP_RUN_IN_BACKGROUND
AppOpsManager.MODE_ALLOWED,  // OP_AUDIO_ACCESSIBILITY_VOLUME
AppOpsManager.MODE_ALLOWED,
AppOpsManager.MODE_DEFAULT,  // OP_REQUEST_INSTALL_PACKAGES
AppOpsManager.MODE_ALLOWED,  // OP_PICTURE_IN_PICTURE
AppOpsManager.MODE_DEFAULT,  // OP_INSTANT_APP_START_FOREGROUND
AppOpsManager.MODE_ALLOWED,  // ANSWER_PHONE_CALLS
AppOpsManager.MODE_ALLOWED,  // OP_RUN_ANY_IN_BACKGROUND
AppOpsManager.MODE_ALLOWED,  // OP_CHANGE_WIFI_STATE
AppOpsManager.MODE_ALLOWED,  // REQUEST_DELETE_PACKAGES
......
};

```

从 sOpDefaultMode 默认权限中可以看出  
AppOpsManager.MODE_DEFAULT, // OP_REQUEST_INSTALL_PACKAGES  
这个未知来源权限默认是关闭的 所以默认要打开，修改为:  
AppOpsManager.MODE_ALLOWED, // OP_REQUEST_INSTALL_PACKAGES  
2. 在 framework 中 PackageInstaller 的 app 中修改部分  
这里处理整个安装 app 的过程，安装之前会判断安装权限什么的 主要由 PackageInstallerActivity.java 来处理  
路径: frameworks/base/packages/PackageInstaller/src/com/android/packageinstaller

```
/PackageInstallerActivity.java
分析PackageInstallerActivity.java 源码
@Override
protected void onCreate(Bundle icicle) {
getWindow().addSystemFlags(SYSTEM_FLAG_HIDE_NON_SYSTEM_OVERLAY_WINDOWS);
  ......
  checkIfAllowedAndInitiateInstall();
 // 安装前检查是否有权限

}
private void checkIfAllowedAndInitiateInstall() {
.....
if (mAllowUnknownSources || !isInstallRequestFromUnknownSource(getIntent())) {
initiateInstall();
} else {
.....
} else {
// 检查如果未知来源进入
handleUnknownSources();
}
}
}
private void handleUnknownSources() {
if (mOriginatingPackage == null) {
Log.i(TAG, "No source found for package " + mPkgInfo.packageName);
showDialogInner(DLG_ANONYMOUS_SOURCE);
return;
}
....代码省略
}
/**
Create a new dialog.
@param id The id of the dialog (determines dialog type)
@return The dialog
*/
private DialogFragment createDialog(int id) {
switch (id) {
case DLG_PACKAGE_ERROR:
return SimpleErrorDialog.newInstance(R.string.Parse_error_dlg_text);
case DLG_OUT_OF_SPACE:
return OutOfSpaceDialog.newInstance(
mPm.getApplicationLabel(mPkgInfo.applicationInfo));
case DLG_INSTALL_ERROR:
return InstallErrorDialog.newInstance(
mPm.getApplicationLabel(mPkgInfo.applicationInfo));
case DLG_NOT_SUPPORTED_ON_WEAR:
return NotSupportedOnWearDialog.newInstance();
case DLG_INSTALL_APPS_RESTRICTED_FOR_USER:
return SimpleErrorDialog.newInstance(
R.string.install_apps_user_restriction_dlg_text);
case DLG_UNKNOWN_SOURCES_RESTRICTED_FOR_USER:
return SimpleErrorDialog.newInstance(
R.string.unknown_apps_user_restriction_dlg_text);
case DLG_EXTERNAL_SOURCE_BLOCKED:
return ExternalSourcesBlockedDialog.newInstance(mOriginatingPackage);
case DLG_ANONYMOUS_SOURCE:
return AnonymousSourceDialog.newInstance();
break;
}
return null;
}

```

case DLG_ANONYMOUS_SOURCE 这里就会弹出未知来源弹窗  
所以默认给与权限就这样修改：  
修改 patch 如下：

```
--- a/frameworks/base/packages/PackageInstaller/src/com/android/packageinstaller/PackageInstallerActivity.java
+++ b/frameworks/base/packages/PackageInstaller/src/com/android/packageinstaller/PackageInstallerActivity.java
@@ -183,7 +183,10 @@ public class PackageInstallerActivity extends AlertActivity {
case DLG_EXTERNAL_SOURCE_BLOCKED:
return ExternalSourcesBlockedDialog.newInstance(mOriginatingPackage);
case DLG_ANONYMOUS_SOURCE:
//去掉弹出AnonymousSourceDialog对话框 默认安装
-           return AnonymousSourceDialog.newInstance();

+            mAllowUnknownSources = true;

+             initiateInstall();

+                       break;

+          //return AnonymousSourceDialog.newInstance();

   }
   return null;

}

```
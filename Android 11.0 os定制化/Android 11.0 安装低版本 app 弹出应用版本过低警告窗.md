> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124889279)

在 11.0 中当安装低版本 (TARGET_SDK 小于 23 以下的 app）时，会弹出应用版本过低提示框，其实也是有些麻烦的 如下图:  
在这里插入图片描述

跟踪代码 原来是 AMS 在启动 app 时会对 app 版本做检测，如果过低就会弹出版本过低提示  
具体流程如下:

```
--- a/frameworks/base/services/core/java/com/android/server/wm/AppWarnings.java
+++ b/frameworks/base/services/core/java/com/android/server/wm/AppWarnings.java
@@ -287,7 +287,7 @@ class AppWarnings {
    
     /**
     * Called when an activity is being started.
     *
     * @param r record for the activity being started
     */

   启动Activity时进行检测
    public void onStartActivity(ActivityRecord r) {
        showUnsupportedCompileSdkDialogIfNeeded(r);
        showUnsupportedDisplaySizeDialogIfNeeded(r);
       showDeprecatedTargetDialogIfNeeded(r)；


    }

检测app版本
 /** 
     * Shows the "deprecated target sdk" warning, if necessary.
     *
     * @param r activity record for which the warning may be displayed
     */
    public void showDeprecatedTargetDialogIfNeeded(ActivityRecord r) {
        //MIN_SUPPORTED_TARGET_SDK_INT =23；
        if (r.appInfo.targetSdkVersion < Build.VERSION.MIN_SUPPORTED_TARGET_SDK_INT) {
            mUiHandler.showDeprecatedTargetDialog(r);
        }
    }

/**
     * Handles messages on the system process UI thread.
     */
    private final class UiHandler extends Handler {
        private static final int MSG_SHOW_UNSUPPORTED_DISPLAY_SIZE_DIALOG = 1;
        private static final int MSG_HIDE_UNSUPPORTED_DISPLAY_SIZE_DIALOG = 2;
        private static final int MSG_SHOW_UNSUPPORTED_COMPILE_SDK_DIALOG = 3;
        private static final int MSG_HIDE_DIALOGS_FOR_PACKAGE = 4;
        private static final int MSG_SHOW_DEPRECATED_TARGET_SDK_DIALOG = 5;

        public UiHandler(Looper looper) {
            super(looper, null, true);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_SHOW_UNSUPPORTED_DISPLAY_SIZE_DIALOG: {
                    final ActivityRecord ar = (ActivityRecord) msg.obj;
                    showUnsupportedDisplaySizeDialogUiThread(ar);
                } break;
                case MSG_HIDE_UNSUPPORTED_DISPLAY_SIZE_DIALOG: {
                    hideUnsupportedDisplaySizeDialogUiThread();
                } break;
                case MSG_SHOW_UNSUPPORTED_COMPILE_SDK_DIALOG: {
                    final ActivityRecord ar = (ActivityRecord) msg.obj;
                    showUnsupportedCompileSdkDialogUiThread(ar);
                } break;
                case MSG_HIDE_DIALOGS_FOR_PACKAGE: {
                    final String name = (String) msg.obj;
                    hideDialogsForPackageUiThread(name);
                } break;
                case MSG_SHOW_DEPRECATED_TARGET_SDK_DIALOG: {
                    final ActivityRecord ar = (ActivityRecord) msg.obj;
                    showDeprecatedTargetSdkDialogUiThread(ar);
                } break;
            }
        }

        public void showUnsupportedDisplaySizeDialog(ActivityRecord r) {
            removeMessages(MSG_SHOW_UNSUPPORTED_DISPLAY_SIZE_DIALOG);
            obtainMessage(MSG_SHOW_UNSUPPORTED_DISPLAY_SIZE_DIALOG, r).sendToTarget();
        }

        public void hideUnsupportedDisplaySizeDialog() {
            removeMessages(MSG_HIDE_UNSUPPORTED_DISPLAY_SIZE_DIALOG);
            sendEmptyMessage(MSG_HIDE_UNSUPPORTED_DISPLAY_SIZE_DIALOG);
        }

        public void showUnsupportedCompileSdkDialog(ActivityRecord r) {
            removeMessages(MSG_SHOW_UNSUPPORTED_COMPILE_SDK_DIALOG);
            obtainMessage(MSG_SHOW_UNSUPPORTED_COMPILE_SDK_DIALOG, r).sendToTarget();
        }

        public void showDeprecatedTargetDialog(ActivityRecord r) {
            removeMessages(MSG_SHOW_DEPRECATED_TARGET_SDK_DIALOG);
            obtainMessage(MSG_SHOW_DEPRECATED_TARGET_SDK_DIALOG, r).sendToTarget();
        }

        public void hideDialogsForPackage(String name) {
            obtainMessage(MSG_HIDE_DIALOGS_FOR_PACKAGE, name).sendToTarget();
        }
    }

  版本过低显示提示框
    /**  
     * Shows the "deprecated target sdk version" warning for the given application.
     * <p>
     * <strong>Note:</strong> Must be called on the UI thread.
     *
     * @param ar record for the activity that triggered the warning
     */
    @UiThread
    private void showDeprecatedTargetSdkDialogUiThread(ActivityRecord ar) {
        if (mDeprecatedTargetSdkVersionDialog != null) {
            mDeprecatedTargetSdkVersionDialog.dismiss();
            mDeprecatedTargetSdkVersionDialog = null;
        }
        if (ar != null && !hasPackageFlag(
                ar.packageName, FLAG_HIDE_DEPRECATED_SDK)) {
            mDeprecatedTargetSdkVersionDialog = new DeprecatedTargetSdkVersionDialog(
                    AppWarnings.this, mUiContext, ar.info.applicationInfo);
            mDeprecatedTargetSdkVersionDialog.show();
        }
    }

```

解决方案：

```
--- a/frameworks/base/services/core/java/com/android/server/wm/AppWarnings.java
+++ b/frameworks/base/services/core/java/com/android/server/wm/AppWarnings.java
@@ -287,7 +287,7 @@ class AppWarnings {


     /**
     * Called when an activity is being started.
     *
     * @param r record for the activity being started
     */
    public void onStartActivity(ActivityRecord r) {
        showUnsupportedCompileSdkDialogIfNeeded(r);
        showUnsupportedDisplaySizeDialogIfNeeded(r);

      
- showDeprecatedTargetDialogIfNeeded(r);
 +        
 //showDeprecatedTargetDialogIfNeeded(r);

    }

```
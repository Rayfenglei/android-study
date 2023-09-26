> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124830703)

在 11.0 平板开发中，客户不需要锁屏, 每次开机锁屏显得麻烦，所以需要禁用锁屏功能  
而锁屏功能都是在 SystemUI 中处理的  
解决方案 就是在 SystemUI 里面禁止开启锁屏功能

步骤 1：

```
KeyguardManager.KeyguardLock 关闭锁屏服务

如下：
SystemUI的SystemUIApplication里面添加关闭锁屏服务；

  @Override

    public void onCreate() {

        super.onCreate();

		



        new android.os.Handler().postDelayed(new Runnable() {

            @Override

              public void run() {



                    android.app.KeyguardManager km = (android.app.KeyguardManager) SystemUIApplication.this.getSystemService(Context.KEYGUARD_SERVICE); /* 获取KeyguardLock对象 */

                    android.app.KeyguardManager.KeyguardLock keyguardLock = km.newKeyguardLock(TAG);

                    // 关闭系统锁屏服务

                    keyguardLock.disableKeyguard();

                }

        },6000);

    }

```

步骤 2：  
在 KeyguardViewMediator.java 锁屏界面禁止锁屏启动

```
frameworks/base/packages/SystemUI/src/com/android/systemui/keyguard/KeyguardViewMediator.java
/**
     * Let us know that the system is ready after startup.
     */
    public void onSystemReady() {
        mHandler.obtainMessage(SYSTEM_READY).sendToTarget();
    }

    private void handleSystemReady() {
        synchronized (this) {
            if (DEBUG) Log.d(TAG, "onSystemReady");
            mSystemReady = true;
            doKeyguardLocked(null);
            mUpdateMonitor.registerCallback(mUpdateCallback);
        }
        /* UNISOC: carrier datausage dialog warning feature for 644278 @{ */
        boolean isShowDialog = false;
        if (mContext.getResources().getBoolean(R.bool.config_show_datausage_dialog)) {
            isShowDialog = Settings.Global.getInt(
                    mContext.getContentResolver(), SHOW_TRAFFIC_TIPS, 1) == 1;
        } else {
            isShowDialog = false;
        }
        Log.d(TAG,
                "StartupReceiver value = "
                        + Settings.Global.getInt(mContext.getContentResolver(),
                        SHOW_TRAFFIC_TIPS, 1) + ", isShowDialog = "
                        + isShowDialog);
        if (isShowDialog) {
            Handler handler = new Handler(Looper.getMainLooper());
            handler.post(new Runnable() {
                @Override
                public void run() {
                    showPackageDialogIfNeeded(mContext);
                }
            });
        }
        /* @} */
        // Most services aren't available until the system reaches the ready state, so we
        // send it here when the device first boots.
        maybeSendUserPresentBroadcast();
    }

```

从源码可以看出 onSystemReady 开机完成后，调用 doKeyguardLocked(null); 启动锁屏功能 所以 在开机完成后去掉 启动锁屏  
注释掉 doKeyguardLocked(null); 即可

修改如下:

```
private void handleSystemReady() {
        synchronized (this) {
            if (DEBUG) Log.d(TAG, "onSystemReady");
            mSystemReady = true;
            - doKeyguardLocked(null);
            mUpdateMonitor.registerCallback(mUpdateCallback);
        }
        /* UNISOC: carrier datausage dialog warning feature for 644278 @{ */
        boolean isShowDialog = false;
        if (mContext.getResources().getBoolean(R.bool.config_show_datausage_dialog)) {
            isShowDialog = Settings.Global.getInt(
                    mContext.getContentResolver(), SHOW_TRAFFIC_TIPS, 1) == 1;
        } else {
            isShowDialog = false;
        }
        Log.d(TAG,
                "StartupReceiver value = "
                        + Settings.Global.getInt(mContext.getContentResolver(),
                        SHOW_TRAFFIC_TIPS, 1) + ", isShowDialog = "
                        + isShowDialog);
        if (isShowDialog) {
            Handler handler = new Handler(Looper.getMainLooper());
            handler.post(new Runnable() {
                @Override
                public void run() {
                    showPackageDialogIfNeeded(mContext);
                }
            });
        }
        /* @} */
        // Most services aren't available until the system reaches the ready state, so we
        // send it here when the device first boots.
        maybeSendUserPresentBroadcast();
    }

```
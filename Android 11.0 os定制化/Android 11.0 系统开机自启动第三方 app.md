> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/131054154)

1. 前言
-----

  在 11.0 的系统 rom 定制化开发中，由于有些第三方 app, 需要在接收到开机广播后，启动 app, 但是在 10.0 以后第三方 app 就接收不到开机广播了  
只有系统 app 才可以接收到开机广播了，所以在 app 内通过接收开机广播自启动就没法实现了 这就需要在系统中添加[监听](https://so.csdn.net/so/search?q=%E7%9B%91%E5%90%AC&spm=1001.2101.3001.7020)开机完成广播的功能，  
然后在接收到开机广播后启动第三方 app 就可以了

2. 系统开机自启动第三方 app 的核心类
----------------------

```
 frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java

```

3. 系统开机自启动第三方 app 的核心功能分析和实现  
3.1 app 中常用的接收开机广播的方法
----------------------------------------------------

```
             <receiver
                android:
                android:enabled="true"
                android:exported="true"
                android:permission="android.permission.RECEIVE_BOOT_COMPLETED">
                <intent-filter android:priority="1000">
                    <action android:/>
                    <category android:/>
                </intent-filter>
            </receiver>
    在app中使用的范例
      import android.content.BroadcastReceiver;
    import android.content.Context;
    import android.content.Intent;
    import android.util.Log;
     
    import com.pne.kotlin.ReplaceIconActivity;
     
    public class StartSelfReceiver extends BroadcastReceiver {
        @Override
        public void onReceive(Context context, Intent intent) {
            String action = intent.getAction();
            //Android设备开机时会发送一条开机广播："android.intent.action.BOOT_COMPLETED"
            if (Intent.ACTION_BOOT_COMPLETED.equals(action)) {
                Intent selfIntent = new Intent(context, ReplaceIconActivity.class);
                selfIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                context.startActivity(selfIntent);
            }
        }
    }
```

在 11.0 的系统中，这种方式会收不到开机广播的，所以对于启动 app 也是做不到的，接下来就只能在 framework 中某个功能模块里面  
接收到开机广播后，启动某个 app 了

3.2 PhoneWindowManager.java 中启动第三方 app 的功能实现
--------------------------------------------

```
       /** {@inheritDoc} */
          @Override
          public void systemReady() {
        // In normal flow, systemReady is called before other system services are ready.
          // So it is better not to bind keyguard here.
          mKeyguardDelegate.onSystemReady();
  
          mVrManagerInternal = LocalServices.getService(VrManagerInternal.class);
          if (mVrManagerInternal != null) {
              mVrManagerInternal.addPersistentVrModeStateListener(mPersistentVrModeListener);
          }
  
          readCameraLensCoverState();
          updateUiMode();
          mDefaultDisplayRotation.updateOrientationListener();
          synchronized (mLock) {
              mSystemReady = true;
              mHandler.post(new Runnable() {
                  @Override
                  public void run() {
                      updateSettings();
                  }
              });
              // If this happens, for whatever reason, systemReady came later than systemBooted.
              // And keyguard should be already bound from systemBooted
              if (mSystemBooted) {
                  mKeyguardDelegate.onBootCompleted();
              }
          }
                //add code start
            IntentFilter intentFilter = new IntentFilter("android.intent.action.BOOT_COMPLETED");
            mContext.registerReceiver(mWallpaperChangedReceiver, intentFilter);
            //add code end
          mAutofillManagerInternal = LocalServices.getService(AutofillManagerInternal.class);
 
     
          }
      
          /** {@inheritDoc} */
          @Override
          public void systemBooted() {
           bindKeyguard();
          synchronized (mLock) {
              mSystemBooted = true;
              if (mSystemReady) {
                  mKeyguardDelegate.onBootCompleted();
              }
          }
          startedWakingUp(ON_BECAUSE_OF_UNKNOWN);
          finishedWakingUp(ON_BECAUSE_OF_UNKNOWN);
          screenTurningOn(null);
          screenTurnedOn();
          }
     
    // add core start
        BroadcastReceiver mWallpaperChangedReceiver  = new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                                  String action = intent.getAction();
            if (action.equals("android.intent.action.BOOT_COMPLETED")) {
                  Slog.d("WindowManager", "android.intent.action.BOOT_COMPLETED");
               Intent selfIntent = new Intent(context, ReplaceIconActivity.class);
                selfIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                mContext.startActivity(selfIntent);
                }
            }
        };
    //add core end
```

在上述的 PhoneWindowManager.java 中相关源码可以看出，在系统相关服务启动完毕后，在调用 systemReady()  
执行 PhoneWindowManager.java 的相关功能中，可以添加相关的开机启动完成广播的监听，然后在  
广播中，收到开机完成广播以后，启动第三方 app 就实现了功能的需求
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126558809)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 首次开机进入 Launcher3 前黑屏几秒的几种情况问题的总结](#t1)

[2.1 开机向导引起的进入 Launcher 桌面短暂黑屏](#t2)

 [2.2 开机动画引起的进入 launcher 前的黑屏情况](#t3)

[2.3 FallbackHome 导致的黑屏问题](#t4)

1. 概述
-----

在工作中，对于系统开发确实有些难度，特别是在开机阶段遇到的问题，比如开机动画播放完毕进入锁屏界面黑屏几秒然后进入 锁屏界面，这就需要根据开机日志来分析问题所在，在工作中遇到的几种黑屏情况做下记录

2. 首次开机进入 Launcher3 前黑屏几秒的几种情况问题的总结
-----------------------------------

### 2.1 开机向导引起的进入 Launcher 桌面短暂黑屏

       在系统中默认是有开机向导的，首次开机会首选进入开机向导，然后进入锁屏桌面，如果某些原因  
引起开机向导卡顿，会造成短暂黑屏然后进入 Launcher 界面  
     去掉开机向导的相关修改为:  
    frameworks/base/packages/SettingsProvider/res/values/defaults.xml  
    <!-- 本文的关键属性 ====== 默认是否开启跳过开机向导 -->  
        <bool >false</bool>  
        <!-- 设备是否已经提供，开机首次是否进入锁屏界面 -->  
        <bool >false</bool>

```
 
      可以修改如下：
frameworks/base/packages/SettingsProvider/res/values/defaults.xml
<bool >true</bool>
<bool >true</bool>
再在产品mk中去掉这两个app:
packages/apps/OneTimeInitializer
packages/apps/Provision
在build的handheld_product.mk中参与编译这两个apk
$(call inherit-product, $(SRC_TARGET_DIR)/product/media_product.mk)
 
# /product packages
PRODUCT_PACKAGES += \
    Camera2 \
    DeskClock \
    LatinIME \
    Launcher3QuickStep \
    OneTimeInitializer \
    Provision \
    Music \
    Settings \
    SettingsIntelligence \
    StorageManager \
    SystemUI \
    WallpaperCropper \
    frameworks-base-overlays
 
PRODUCT_PACKAGES_DEBUG += \
    frameworks-base-overlays-debug
 
修改如下：
build\make\target\product\handheld_product.mk
$(call inherit-product, $(SRC_TARGET_DIR)/product/media_product.mk)
 
# /product packages
PRODUCT_PACKAGES += \
    Camera2 \
    DeskClock \
    LatinIME \
    Launcher3QuickStep \
-    OneTimeInitializer \
-    Provision \
    Music \
    Settings \
    SettingsIntelligence \
    StorageManager \
    SystemUI \
    WallpaperCropper \
    frameworks-base-overlays
 
PRODUCT_PACKAGES_DEBUG += \
    frameworks-base-overlays-debug
 
 
```

###  让系统直接启动桌面，不用启动 Provision。Provision 干的事情和 SetupWizard、  
OneTimeInitializer 类似。都是设置 DEVICE_PROVISIONED 和 USER_SETUP_COMPLETE。

### 2.2 开机动画引起的进入 launcher 前的黑屏情况

  在系统进入首次开机的时候，由于需要加载相当多的系统数据和服务，首次开机耗时会长一点，如果  
开机动画过少，在播放完开机动画以后，还没等到 AMS 执行完停止播放动画的相关通知，就会陷入  
黑屏状态等得到 AMS 停止播放动画以后就会进入桌面

```
 
frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
    final void finishBooting() {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "FinishBooting");
 
        synchronized (this) {
            if (!mBootAnimationComplete) {
                mCallFinishBooting = true;
                return;
            }
            mCallFinishBooting = false;
        }
 
        ArraySet<String> completedIsas = new ArraySet<String>();
        for (String abi : Build.SUPPORTED_ABIS) {
            ZYGOTE_PROCESS.establishZygoteConnectionForAbi(abi);
            final String instructionSet = VMRuntime.getInstructionSet(abi);
            if (!completedIsas.contains(instructionSet)) {
                try {
                    mInstaller.markBootComplete(VMRuntime.getInstructionSet(abi));
                } catch (InstallerException e) {
                    if (!VMRuntime.didPruneDalvikCache()) {
                        // This is technically not the right filter, as different zygotes may
                        // have made different pruning decisions. But the log is best effort,
                        // anyways.
                        Slog.w(TAG, "Unable to mark boot complete for abi: " + abi + " (" +
                                e.getMessage() +")");
                    }
                }
                completedIsas.add(instructionSet);
            }
        }
 
        IntentFilter pkgFilter = new IntentFilter();
        pkgFilter.addAction(Intent.ACTION_QUERY_PACKAGE_RESTART);
        pkgFilter.addDataScheme("package");
        mContext.registerReceiver(new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                String[] pkgs = intent.getStringArrayExtra(Intent.EXTRA_PACKAGES);
                if (pkgs != null) {
                    for (String pkg : pkgs) {
                        synchronized (ActivityManagerService.this) {
                            if (forceStopPackageLocked(pkg, -1, false, false, false, false, false,
                                    0, "query restart")) {
                                setResultCode(Activity.RESULT_OK);
                                return;
                            }
                        }
                    }
                }
            }
        }, pkgFilter);
 
        IntentFilter dumpheapFilter = new IntentFilter();
        dumpheapFilter.addAction(DumpHeapActivity.ACTION_DELETE_DUMPHEAP);
        mContext.registerReceiver(new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                final long delay = intent.getBooleanExtra(
                        DumpHeapActivity.EXTRA_DELAY_DELETE, false) ? 5 * 60 * 1000 : 0;
                mHandler.sendEmptyMessageDelayed(DELETE_DUMPHEAP_MSG, delay);
            }
        }, dumpheapFilter);
 
        // Inform checkpointing systems of success
        try {
            // This line is needed to CTS test for the correct exception handling
            // See b/138952436#comment36 for context
            Slog.i(TAG, "About to commit checkpoint");
            IStorageManager storageManager = PackageHelper.getStorageManager();
            storageManager.commitChanges();
        } catch (Exception e) {
            PowerManager pm = (PowerManager)
                     mContext.getSystemService(Context.POWER_SERVICE);
            pm.reboot("Checkpoint commit failed");
        }
 
        // Let system services know.
        mSystemServiceManager.startBootPhase(SystemService.PHASE_BOOT_COMPLETED);
 
        synchronized (this) {
            // Ensure that any processes we had put on hold are now started
            // up.
            final int NP = mProcessesOnHold.size();
            if (NP > 0) {
                ArrayList<ProcessRecord> procs =
                    new ArrayList<ProcessRecord>(mProcessesOnHold);
                for (int ip=0; ip<NP; ip++) {
                    if (DEBUG_PROCESSES) Slog.v(TAG_PROCESSES, "Starting process on hold: "
                            + procs.get(ip));
                    mProcessList.startProcessLocked(procs.get(ip), new HostingRecord("on-hold"));
                }
            }
            if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL) {
                return;
            }
            // Start looking for apps that are abusing wake locks.
            Message nmsg = mHandler.obtainMessage(CHECK_EXCESSIVE_POWER_USE_MSG);
            mHandler.sendMessageDelayed(nmsg, mConstants.POWER_CHECK_INTERVAL);
            // Tell anyone interested that we are done booting!
            SystemProperties.set("sys.boot_completed", "1");
 
            // And trigger dev.bootcomplete if we are not showing encryption progress
            if (!"trigger_restart_min_framework".equals(VoldProperties.decrypt().orElse(""))
                    || "".equals(VoldProperties.encrypt_progress().orElse(""))) {
                SystemProperties.set("dev.bootcomplete", "1");
            }
       //发送有序的开机广播ACTION_LOCKED_BOOT_COMPLETED
            mUserController.sendBootCompleted(
                    new IIntentReceiver.Stub() {
                        @Override
                        public void performReceive(Intent intent, int resultCode,
                                String data, Bundle extras, boolean ordered,
                                boolean sticky, int sendingUser) {
                            synchronized (ActivityManagerService.this) {
                                mOomAdjuster.mAppCompact.compactAllSystem();
                                requestPssAllProcsLocked(SystemClock.uptimeMillis(), true, false);
                            }
                        }
                    });
            mUserController.scheduleStartProfiles();
        }
 
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    }
 
在SurfaceFlinger.bootFinished中停止动画相关方法
frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::bootFinished()
  {
      if (mBootFinished == true) {
          ALOGE("Extra call to bootFinished");
          return;
      }
      mBootFinished = true;
      if (mStartPropertySetThread->join() != NO_ERROR) {
          ALOGE("Join StartPropertySetThread failed!");
      }
      const nsecs_t now = systemTime();
      const nsecs_t duration = now - mBootTime;
      ALOGI("Boot is finished (%ld ms)", long(ns2ms(duration)) );
  
      mFrameTracer->initialize();
      mTimeStats->onBootFinished();
  
      // wait patiently for the window manager death
      const String16 name("window");
      mWindowManager = defaultServiceManager()->getService(name);
      if (mWindowManager != 0) {
          mWindowManager->linkToDeath(static_cast<IBinder::DeathRecipient*>(this));
      }
      sp<IBinder> input(defaultServiceManager()->getService(
              String16("inputflinger")));
      if (input == nullptr) {
          ALOGE("Failed to link to input service");
      } else {
          mInputFlinger = interface_cast<IInputFlinger>(input);
      }
  
      if (mVrFlinger) {
        mVrFlinger->OnBootFinished();
      }
  
      // stop boot animation
      // formerly we would just kill the process, but we now ask it to exit so it
      // can choose where to stop the animation.
      property_set("service.bootanim.exit", "1");
  
      const int LOGTAG_SF_STOP_BOOTANIM = 60110;
      LOG_EVENT_LONG(LOGTAG_SF_STOP_BOOTANIM,
                     ns2ms(systemTime(SYSTEM_TIME_MONOTONIC)));
  
      static_cast<void>(schedule([this] {
          readPersistentProperties();
          mPowerAdvisor.onBootFinished();
          mBootStage = BootStage::FINISHED;
  
          if (property_get_bool("sf.debug.show_refresh_rate_overlay", false)) {
              enableRefreshRateOverlay(true);
          }
      }));
  }
在BootAnimation.cpp退出动画的相关方法
frameworks/base/cmds/bootanimation/BootAnimation.cpp
  void BootAnimation::checkExit() {
      // Allow surface flinger to gracefully request shutdown
      char value[PROPERTY_VALUE_MAX];
      property_get(EXIT_PROP_NAME, value, "0");
      int exitnow = atoi(value);
      if (exitnow) {
          requestExit();
          mCallbacks->shutdown();
      }
  }
 
 
```

### 在项目中遇到开机动画只有几张的项目 在首次开机的时候会出现黑屏  
把开机动画增加到 30 张左右的就解决了这个问题

### 2.3 FallbackHome 导致的黑屏问题

    在无锁屏的情况下 会在开机动画播放完毕后进入系统设置的 FallbackHome，等收到解锁通知后  
然后进入默认桌面，这时可以在 onCreate 设置开机动画最后一张作为背景

```
 
  public class FallbackHome extends Activity {
    private static final String TAG = "FallbackHome";
    private static final int PROGRESS_TIMEOUT = 2000;
 
    private boolean mProvisioned;
    private WallpaperManager mWallManager;
 
    private final Runnable mProgressTimeoutRunnable = () -> {
        View v = getLayoutInflater().inflate(
                R.layout.fallback_home_finishing_boot, null /* root */);
        setContentView(v);
        v.setAlpha(0f);
        v.animate()
                .alpha(1f)
                .setDuration(500)
                .setInterpolator(AnimationUtils.loadInterpolator(
                        this, android.R.interpolator.fast_out_slow_in))
                .start();
        getWindow().addFlags(LayoutParams.FLAG_KEEP_SCREEN_ON);
    };
 
    private final OnColorsChangedListener mColorsChangedListener = new OnColorsChangedListener() {
        @Override
        public void onColorsChanged(WallpaperColors colors, int which) {
            if (colors != null) {
                final View decorView = getWindow().getDecorView();
                decorView.setSystemUiVisibility(
                        updateVisibilityFlagsFromColors(colors, decorView.getSystemUiVisibility()));
                mWallManager.removeOnColorsChangedListener(this);
            }
        }
    };
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
 
        // Set ourselves totally black before the device is provisioned so that
        // we don't flash the wallpaper before SUW
        mProvisioned = Settings.Global.getInt(getContentResolver(),
                Settings.Global.DEVICE_PROVISIONED, 0) != 0;
        final int flags;
        if (!mProvisioned) {
            setTheme(R.style.FallbackHome_SetupWizard);
            flags = View.SYSTEM_UI_FLAG_FULLSCREEN | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
                    | View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY;
        } else {
            flags = View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
                    | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION;
        }
 
        mWallManager = getSystemService(WallpaperManager.class);
        if (mWallManager == null) {
            Log.w(TAG, "Wallpaper manager isn't ready, can't listen to color changes!");
        } else {
            loadWallpaperColors(flags);
        }
        getWindow().getDecorView().setSystemUiVisibility(flags);
       // 增加背景
+     getWindow().setBackgroundDrawableResource(R.drawable.background);
        registerReceiver(mReceiver, new IntentFilter(Intent.ACTION_USER_UNLOCKED));
        maybeFinish();
    }
 
    @Override
    protected void onResume() {
        super.onResume();
        if (mProvisioned) {
            mHandler.postDelayed(mProgressTimeoutRunnable, PROGRESS_TIMEOUT);
        }
    }
}
 
第二种改法
<FrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
 -   android:background="#80000000"
 +  android:background="@drawable/bg"
    android:forceHasOverlappingRendering="false">
 
    <LinearLayout
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:layout_gravity="center"
        android:layout_marginStart="16dp"
        android:layout_marginEnd="16dp">
 
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textSize="20sp"
            android:textColor="?android:attr/textColorPrimary"
            android:text="@*android:string/android_start_title"/>
 
        <ProgressBar
            style="@android:style/Widget.Material.ProgressBar.Horizontal"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="12.75dp"
            android:colorControlActivated="?android:attr/textColorPrimary"
            android:indeterminate="true"/>
 
    </LinearLayout>
</FrameLayout>
 
```

总结：这三种情况目前是开发中，遇到常见的情况，具体情况可以根据开机[日志分析](https://so.csdn.net/so/search?q=%E6%97%A5%E5%BF%97%E5%88%86%E6%9E%90&spm=1001.2101.3001.7020)来解决相关的问题
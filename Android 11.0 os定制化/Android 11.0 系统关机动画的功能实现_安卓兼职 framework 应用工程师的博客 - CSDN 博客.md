> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/130066853)

1. 前言
-----

  
  在系统 11.0 的系统 rom 定制化开发中，在原生系统中默认有开机动画功能，但是在系统关机动画方面功能不是很完善，如果只内置关机动画，会发现关机动画还没播放完  
就关机了，所以需要在系统关机要等到关机动画播完才执行关机的动作，接下来就来分析关机流程，然后来实现这个功能

2. 系统关机动画的功能实现的核心类
------------------

```
frameworks/base/cmds/bootanimation/BootAnimation.cpp
frameworks/base/services/core/java/com/android/server/power/ShutdownThread.java
```

3. 系统关机动画的功能实现的核心功能实现和分析  
3.1 增加关机动画类 ShutdownAnimation.java
-------------------------------------------------------------

```
package com.android.server.power;
 
import android.os.RemoteException;
import android.os.ServiceManager;
import android.os.SystemClock;
import android.os.SystemProperties;
import android.view.IWindowManager;
import android.util.Slog;
import java.io.File;
 
/** {@hide} */
public class ShutdownAnimation {
    private static final String TAG = "ShutdownAnimation";
    private static final int BOOT_ANIMATION_CHECK_SPAN = 200;
    private static final int MAX_BOOTANIMATION_WAIT_TIME = 15*1000;
    private final Object mAnimationDoneSync = new Object();
    private boolean mPlayAnim = false;
    private static String SHUT_DOWN_ANIMATION_RES_PRODUCT_PATH = "/product/media/shutdownanimation.zip";
    private static String SHUT_DOWN_ANIMATION_RES_SYSTEM_PATH = "/system/media/shutdownanimation.zip";
    private static String SHUT_DOWN_ANIMATION_RES_OEM_PATH = "/oem/media/shutdownanimation.zip";
 
    boolean hasShutdownAnimation() {
        File fileProduct = new File(SHUT_DOWN_ANIMATION_RES_PRODUCT_PATH);
        File fileDefault = new File(SHUT_DOWN_ANIMATION_RES_SYSTEM_PATH);
        File fileOem = new File(SHUT_DOWN_ANIMATION_RES_OEM_PATH);
        return fileProduct.exists() || fileDefault.exists() || fileOem.exists();
    }
 
    boolean playShutdownAnimation() {
        IWindowManager wm = IWindowManager.Stub.asInterface(ServiceManager.getService("window"));
        try {
            wm.updateRotation(false, false);
        } catch (RemoteException e) {
            Slog.e(TAG, "stop orientation failed!", e);
        }
 
        //String[] bootcmd = {"bootanimation", "shutdown"} ;
        try {
            mPlayAnim = true;
            Slog.i(TAG, "exec the bootanimation ");
       +     SystemProperties.set("service.wait_for_bootanim", "1");
            SystemProperties.set("service.bootanim.exit", "0");
            SystemProperties.set("service.bootanim.end", "0");
            SystemProperties.set("service.bootanim.shutdown", "1");
            SystemProperties.set("ctl.start", "bootanim");
            //Runtime.getRuntime().exec(bootcmd);
        } catch (Exception e) {
            mPlayAnim = false;
            Slog.e(TAG,"bootanimation command exe err!");
        }
        return true;
    }
 
    void waitForBootanim() {
        // SPRD: SPCSS00331921 Add for extended shutdown time off BEG-->
        if (mPlayAnim && SystemProperties.get("service.wait_for_bootanim").equals("1")) {
            mPlayAnim = false;
            synchronized (mAnimationDoneSync) {
                final long endBootAnimationTime = SystemClock.elapsedRealtime() + MAX_BOOTANIMATION_WAIT_TIME;
                while (SystemProperties.get("service.bootanim.end").equals("0")) {
                    long delay = endBootAnimationTime - SystemClock.elapsedRealtime();
                    if (delay <= 0) {
                        Slog.w(TAG, "BootAnimation wait timed out");
                        break;
                    }
                    try {
                        Slog.d(TAG,"-----waiting boot animation completed-----");
                        mAnimationDoneSync.wait(BOOT_ANIMATION_CHECK_SPAN);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
}
```

从上述代码可以看出，在 frameworks/base/services/core/java/com/android/server/power 的目录下增加关机动画工具类,  
在 playShutdownAnimation() 增加 SystemProperties.set(“service.wait_for_bootanim”, “1”); 来判断是否开始执行播放关机动画的功能  
在 waitForBootanim() 中一直等待播放关机动画执行完毕后，退出 while, 然后执行下一步动作

3.2 在 ShutdownThread.java 中增加等待关机动画的播放的相关方法
-------------------------------------------

```
+     private static ShutdownAnimation shutdownAnim = new ShutdownAnimation();
  /**
       * Makes sure we handle the shutdown gracefully.
       * Shuts off power regardless of radio state if the allotted time has passed.
       */
      public void run() {
          TimingsTraceLog shutdownTimingLog = newTimingsLog();
          shutdownTimingLog.traceBegin("SystemServerShutdown");
          metricShutdownStart();
          metricStarted(METRIC_SYSTEM_SERVER);
  
.....  
          /*
           * If we are rebooting into safe mode, write a system property
           * indicating so.
           */
          if (mRebootSafeMode) {
              SystemProperties.set(REBOOT_SAFEMODE_PROPERTY, "1");
          }
  
          metricStarted(METRIC_SEND_BROADCAST);
          shutdownTimingLog.traceBegin("SendShutdownBroadcast");
          Log.i(TAG, "Sending shutdown broadcast...");
........
  
          if (mRebootHasProgressBar) {
              sInstance.setRebootProgress(MOUNT_SERVICE_STOP_PERCENT, null);
  
              // If it's to reboot to install an update and uncrypt hasn't been
              // done yet, trigger it now.
              uncrypt();
          }
  
          shutdownTimingLog.traceEnd(); // SystemServerShutdown
          metricEnded(METRIC_SYSTEM_SERVER);
          saveMetrics(mReboot, mReason);
// 添加等待关机动画播放完毕 然后执行关机动作
+         shutdownAnim.waitForBootanim();
          // Remaining work will be done by init, including vold shutdown
          rebootOrShutdown(mContext, mReboot, mReason);
      }
 
    private static void beginShutdownSequence(Context context) {
        synchronized (sIsStartedGuard) {
            if (sIsStarted) {
                Log.d(TAG, "Shutdown sequence already running, returning.");
                return;
            }
            sIsStarted = true;
        }
 
 
//modify core start 
        // SPRD:add for shutdownanim
        if (shutdownAnim.hasShutdownAnimation() &&
                !(mReason != null && mReason.startsWith(PowerManager.REBOOT_RECOVERY_UPDATE))) {
            shutdownAnim.playShutdownAnimation();
        } else {
            sInstance.mProgressDialog = showShutdownDialog(context);
        }
//modify core end
 
 
        sInstance.mContext = context;
 
        // make sure we never fall asleep again
        sInstance.mCpuWakeLock = null;
        try {
            sInstance.mCpuWakeLock = sInstance.mPowerManager.newWakeLock(
                    PowerManager.PARTIAL_WAKE_LOCK, TAG + "-cpu");
            sInstance.mCpuWakeLock.setReferenceCounted(false);
            sInstance.mCpuWakeLock.acquire();
        } catch (SecurityException e) {
            Log.w(TAG, "No permission to acquire wake lock", e);
            sInstance.mCpuWakeLock = null;
        }
...
}
```

在 ShutdownThread.java 中的上述方法中，可以看出，在主要执行关机的 ShutdownThread.java 中的 run() 方法中，在  
执行关机动作前的方法添加 shutdownAnim.waitForBootanim(); 来等待关机动画播放完毕后，来执行关机动作  
rebootOrShutdown(mContext, mReboot, mReason); 然后在开始执行关机动作的时候，增加  
shutdownAnim.playShutdownAnimation(); 来执行播放关机动画的准备工作，

3.3 BootAnimation.cpp 中相关关机动画的相关功能分析
------------------------------------

```
  BootAnimation::BootAnimation(sp<Callbacks> callbacks)
          : Thread(false), mClockEnabled(true), mTimeIsAccurate(false),
          mTimeFormat12Hour(false), mTimeCheckThread(NULL), mCallbacks(callbacks) {
      mSession = new SurfaceComposerClient();
  
      std::string powerCtl = android::base::GetProperty("sys.powerctl", "");
      if (powerCtl.empty()) {
          mShuttingDown = false;
      } else {
          mShuttingDown = true;
      }
	  
	  //add core start
    if (!mShuttingDown) {
        mShuttingDown = android::base::GetProperty("service.bootanim.shutdown", "").compare("1") == 0;
    }
    if (mShuttingDown) {
        mWaitForComplete = android::base::GetBoolProperty("service.wait_for_bootanim", false);
    }
	  //add core end
	  
  }
```

在 BootAnimation.cpp 中的上述相关方法中，在 BootAnimation(）的构造方法中，增加 mWaitForComplete 变量用来初始化 mWaitForComplete 来判断  
当前是否已经播放完关机动画，用来在 playAnimation(const [Animation](https://so.csdn.net/so/search?q=Animation&spm=1001.2101.3001.7020)& animation) 判断是否退出关机动画，进入到关机状态

```
bool BootAnimation::playAnimation(const Animation& animation)
{
    const size_t pcount = animation.parts.size();
    nsecs_t frameDuration = s2ns(1) / animation.fps;
    const int animationX = (mWidth - animation.width) / 2;
    const int animationY = (mHeight - animation.height) / 2;
 
    ALOGD("%sAnimationShownTiming start time: %" PRId64 "ms,folder count is %zu",
            mShuttingDown ? "Shutdown" : "Boot", elapsedRealtime(), pcount);
#ifdef BOOTANIMATION_EXT
    scaleSurfaceIfNeeded();
#endif
    for (size_t i=0 ; i<pcount ; i++) {
        const Animation::Part& part(animation.parts[i]);
        const size_t fcount = part.frames.size();
        ALOGD("folder%zu pictures count is %zu",i,fcount);
        glBindTexture(GL_TEXTURE_2D, 0);
 
        // Handle animation package
        if (part.animation != nullptr) {
            playAnimation(*part.animation);
            if (exitPending())
                break;
            continue; //to next part
        }
//add core start
      +   if (mShuttingDown && mfd == -1 && mWaitForComplete && (i==(pcount-1))) {
      +      ALOGD("shutdown animation finished, quit");
      +       property_set("service.bootanim.end", "1");
      +   }
//add core end
        for (int r=0 ; !part.count || r<part.count ; r++) {
            // Exit any non playuntil complete parts immediately
            if(exitPending() && !part.playUntilComplete)
                break;
 
            mCallbacks->playPart(i, part, r);
 
            glClearColor(
                    part.backgroundColor[0],
                    part.backgroundColor[1],
                    part.backgroundColor[2],
                    1.0f);
 
            for (size_t j=0 ; j<fcount && (!exitPending() || part.playUntilComplete) ; j++) {
                const Animation::Frame& frame(part.frames[j]);
                nsecs_t lastFrame = systemTime();
 
                if (r > 0) {
                    glBindTexture(GL_TEXTURE_2D, frame.tid);
                } else {
                    if (part.count != 1) {
                        glGenTextures(1, &frame.tid);
                        glBindTexture(GL_TEXTURE_2D, frame.tid);
                        glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
                        glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
                    }
                    int w, h;
                    initTexture(frame.map, &w, &h);
                }
 
                const int xc = animationX + frame.trimX;
                const int yc = animationY + frame.trimY;
                Region clearReg(Rect(mWidth, mHeight));
                ALOGD_ANIMATION("subtractSelf Surface xc is %d, yc is %d,xc+frame.trimWidth is %d,yc+frame.trimHeight is %d",
                                xc, yc, (xc+frame.trimWidth),(yc+frame.trimHeight));
                clearReg.subtractSelf(Rect(xc, yc, xc+frame.trimWidth, yc+frame.trimHeight));
                if (!clearReg.isEmpty()) {
                    Region::const_iterator head(clearReg.begin());
                    Region::const_iterator tail(clearReg.end());
                    glEnable(GL_SCISSOR_TEST);
                    while (head != tail) {
                        const Rect& r2(*head++);
                        glScissor(r2.left, mHeight - r2.bottom, r2.width(), r2.height());
                        glClear(GL_COLOR_BUFFER_BIT);
                    }
                    glDisable(GL_SCISSOR_TEST);
                }
                // specify the y center as ceiling((mHeight - frame.trimHeight) / 2)
                // which is equivalent to mHeight - (yc + frame.trimHeight)
                glDrawTexiOES(xc, mHeight - (yc + frame.trimHeight),
                              0, frame.trimWidth, frame.trimHeight);
                if (mClockEnabled && mTimeIsAccurate && validClock(part)) {
                    drawClock(animation.clockFont, part.clockPosX, part.clockPosY);
                }
                handleViewport(frameDuration);
 
                eglSwapBuffers(mDisplay, mSurface);
 
                nsecs_t now = systemTime();
                nsecs_t delay = frameDuration - (now - lastFrame);
                ALOGD_ANIMATION("frameDuration=%" PRId64 "ms,this frame costs time=%" PRId64 "ms, delay=%" PRId64 "ms", ns2ms(frameDuration), ns2ms(now - lastFrame), ns2ms(delay));
                lastFrame = now;
 
                if (delay > 0) {
                    struct timespec spec;
                    spec.tv_sec  = (now + delay) / 1000000000;
                    spec.tv_nsec = (now + delay) % 1000000000;
                    int err;
                    do {
                        err = clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, &spec, nullptr);
                    } while (err<0 && errno == EINTR);
                }
 
                if (!mShuttingDown) {
                    checkExit();
                }
            }
 
            usleep(part.pause * ns2us(frameDuration));
 
            // For infinite parts, we've now played them at least once, so perhaps exit
            if(exitPending() && !part.count && mCurrentInset >= mTargetInset)
                break;
        }
    }
    // Free textures created for looping parts now that the animation is done.
    for (const Animation::Part& part : animation.parts) {
        if (part.count != 1) {
            const size_t fcount = part.frames.size();
            for (size_t j = 0; j < fcount; j++) {
                const Animation::Frame& frame(part.frames[j]);
                glDeleteTextures(1, &frame.tid);
            }
        }
    }
    return true;
}
```

在 BootAnimation.cpp 中的上述相关方法中，在 BootAnimation(）的构造方法中，用来初始化 mWaitForComplete 来判断  
当前是否已经播放完关机动画，然后在 playAnimation(const Animation& animation) 中播放关机动画中，判断  
如果关机动画播放完毕，就设置系统属性 service.bootanim.end 为 1, 然后就在 ShutdownThread.java 中  
开始执行关机动作来实现关机功能，这样就实现了完整播放关机动画，等关机动画播放完毕后执行关机动作
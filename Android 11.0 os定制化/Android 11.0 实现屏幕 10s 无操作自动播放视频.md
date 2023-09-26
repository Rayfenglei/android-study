> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124760569)

### 1. 概述

在 11.0 的产品定制化中，由于定制化的是广告机项目，需要不间断的播放视频，当对屏幕没有操作时继续播放视频，对于这个功能，在 app 中，可以通过 onTouch 事件来判断手势抬起后倒计时 10s 无操作时然后播放视频，但是对于系统而言就难判断屏幕是否操作了，需要从  
无操作息屏的流程分析问题了

### 2. 实现屏幕 10s 无操作自动播放视频的核心类

```
frameworks/base/services/core/java/com/android/server/power/PowerManagerService.java

```

### 3. 实现屏幕 10s 无操作自动播放视频的核心功能实现和分析

在对于系统息屏的管理就是在 PowerManagerService.java 做处理，管理上层的电源相关的操作  
在 PowerManagerService.java 中有对操作屏幕做了记录时间

### 3.1 PowerManagerService.java 息屏相关的分析

```
private void updatePowerStateLocked() {
          if (!mSystemReady || mDirty == 0) {
              return;
          }
          if (!Thread.holdsLock(mLock)) {
              Slog.wtf(TAG, "Power manager lock was not held when calling updatePowerStateLocked");
          }
  
          Trace.traceBegin(Trace.TRACE_TAG_POWER, "updatePowerState");
          try {
              // Phase 0: Basic state updates.
              updateIsPoweredLocked(mDirty);
              updateStayOnLocked(mDirty);
              updateScreenBrightnessBoostLocked(mDirty);
  
              // Phase 1: Update wakefulness.
              // Loop because the wake lock and user activity computations are influenced
              // by changes in wakefulness.
              final long now = mClock.uptimeMillis();
              int dirtyPhase2 = 0;
              for (;;) {
                  int dirtyPhase1 = mDirty;
                  dirtyPhase2 |= dirtyPhase1;
                  mDirty = 0;
  
                  updateWakeLockSummaryLocked(dirtyPhase1);
                  updateUserActivitySummaryLocked(now, dirtyPhase1);
                  updateAttentiveStateLocked(now, dirtyPhase1);
                  if (!updateWakefulnessLocked(dirtyPhase1)) {
                      break;
                  }
              }
  
              // Phase 2: Lock profiles that became inactive/not kept awake.
              updateProfilesLocked(now);
  
              // Phase 3: Update display power state.
              final boolean displayBecameReady = updateDisplayPowerStateLocked(dirtyPhase2);
  
              // Phase 4: Update dream state (depends on display ready signal).
              updateDreamLocked(dirtyPhase2, displayBecameReady);
  
              // Phase 5: Send notifications, if needed.
              finishWakefulnessChangeIfNeededLocked();
  
              // Phase 6: Update suspend blocker.
              // Because we might release the last suspend blocker here, we need to make sure
              // we finished everything else first!
              updateSuspendBlockerLocked();
          } finally {
              Trace.traceEnd(Trace.TRACE_TAG_POWER);
          }
      }

/**
 * Updates the value of mUserActivitySummary to summarize the user requested
 * state of the system such as whether the screen should be bright or dim.
 * Note that user activity is ignored when the system is asleep.
 *
 * This function must have no other side-effects.
 */
private void updateUserActivitySummaryLocked(long now, int dirty) {
    // Update the status of the user activity timeout timer.
    if ((dirty & (DIRTY_WAKE_LOCKS | DIRTY_USER_ACTIVITY
            | DIRTY_WAKEFULNESS | DIRTY_SETTINGS)) != 0) {
        mHandler.removeMessages(MSG_USER_ACTIVITY_TIMEOUT);

        long nextTimeout = 0;
        if (mWakefulness == WAKEFULNESS_AWAKE
                || mWakefulness == WAKEFULNESS_DREAMING
                || mWakefulness == WAKEFULNESS_DOZING) {
            final long sleepTimeout = getSleepTimeoutLocked();
            final long screenOffTimeout = getScreenOffTimeoutLocked(sleepTimeout);
            final long screenDimDuration = getScreenDimDurationLocked(screenOffTimeout);
            final boolean userInactiveOverride = mUserInactiveOverrideFromWindowManager;
            final long nextProfileTimeout = getNextProfileTimeoutLocked(now);

            mUserActivitySummary = 0;
            if (mLastUserActivityTime >= mLastWakeTime) {
                nextTimeout = mLastUserActivityTime
                        + screenOffTimeout - screenDimDuration;
                if (now < nextTimeout) {
                    mUserActivitySummary = USER_ACTIVITY_SCREEN_BRIGHT;
                } else {
                    nextTimeout = mLastUserActivityTime + screenOffTimeout;
                    if (now < nextTimeout) {
                        mUserActivitySummary = USER_ACTIVITY_SCREEN_DIM;
                    }
                }
            }
            if (mUserActivitySummary == 0
                    && mLastUserActivityTimeNoChangeLights >= mLastWakeTime) {
                nextTimeout = mLastUserActivityTimeNoChangeLights + screenOffTimeout;
                if (now < nextTimeout) {
                    if (mDisplayPowerRequest.policy == DisplayPowerRequest.POLICY_BRIGHT
                            || mDisplayPowerRequest.policy == DisplayPowerRequest.POLICY_VR) {
                        mUserActivitySummary = USER_ACTIVITY_SCREEN_BRIGHT;
                    } else if (mDisplayPowerRequest.policy == DisplayPowerRequest.POLICY_DIM) {
                        mUserActivitySummary = USER_ACTIVITY_SCREEN_DIM;
                    }
                }
            }

            if (mUserActivitySummary == 0) {
                if (sleepTimeout >= 0) {
                    final long anyUserActivity = Math.max(mLastUserActivityTime,
                            mLastUserActivityTimeNoChangeLights);
                    if (anyUserActivity >= mLastWakeTime) {
                        nextTimeout = anyUserActivity + sleepTimeout;
                        if (now < nextTimeout) {
                            mUserActivitySummary = USER_ACTIVITY_SCREEN_DREAM;
                        }
                    }
                } else {
                    mUserActivitySummary = USER_ACTIVITY_SCREEN_DREAM;
                    nextTimeout = -1;
                }
            }

            if (mUserActivitySummary != USER_ACTIVITY_SCREEN_DREAM && userInactiveOverride) {
                if ((mUserActivitySummary &
                        (USER_ACTIVITY_SCREEN_BRIGHT | USER_ACTIVITY_SCREEN_DIM)) != 0) {
                    // Device is being kept awake by recent user activity
                    if (nextTimeout >= now && mOverriddenTimeout == -1) {
                        // Save when the next timeout would have occurred
                        mOverriddenTimeout = nextTimeout;
                    }
                }
                mUserActivitySummary = USER_ACTIVITY_SCREEN_DREAM;
                nextTimeout = -1;
            }

            if ((mUserActivitySummary & USER_ACTIVITY_SCREEN_BRIGHT) != 0
                    && (mWakeLockSummary & WAKE_LOCK_STAY_AWAKE) == 0) {
                nextTimeout = mAttentionDetector.updateUserActivity(nextTimeout);
            }

            if (nextProfileTimeout > 0) {
                nextTimeout = Math.min(nextTimeout, nextProfileTimeout);
            }

            if (mUserActivitySummary != 0 && nextTimeout >= 0) {
                scheduleUserInactivityTimeout(nextTimeout);
            }
        } else {
            mUserActivitySummary = 0;
        }

        if (DEBUG_SPEW) {
            Slog.d(TAG, "updateUserActivitySummaryLocked: mWakefulness="
                    + PowerManagerInternal.wakefulnessToString(mWakefulness)
                    + ", mUserActivitySummary=0x" + Integer.toHexString(mUserActivitySummary)
                    + ", nextTimeout=" + TimeUtils.formatUptime(nextTimeout));
        }
    }
}

```

在系统关于对息屏的操作是首先调用 updatePowerStateLocked() 来更新电源的状态，而在 updatePowerStateLocked() 中  
通过源码发现 updateUserActivitySummaryLocked(）会对每次比较 mLastUserActivityTime 的值  
来做相应的操作  
核心功能分析就是在  
所以就在这里添加判断无操作时间是否操作 10 秒如下

```
if(now-mLastUserActivityTime>=10000 && mBootCompleted && mSystemReady){

 Message msg = mHandler.obtainMessage(MSG_USER_NOT_ACTION);

 msg.setAsynchronous(true);

 mHandler.sendMessage(msg);

}

```

在开机完成开始判断，以免会有其他 bug 出现  
now 为开机启动到现在的时间，只需要计算开机启动后到现在的时间与最后一次用户操作的时间之差即可满足一段时间内用户无操作的需求。而用户每一次操作系统，mLastUserActivityTime 都会更新，因此只需这个条件即可，发送 msg 消息来播放视频
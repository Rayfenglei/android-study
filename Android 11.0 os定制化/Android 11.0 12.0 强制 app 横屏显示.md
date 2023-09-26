> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124705443)

### 1. 概述

在 11.012.0 产品定制化开发中, 对于处理屏幕旋转方向，首先有 kernel 底层处理，从底层驱动 gsensor 中获取数据，从而判断屏幕方向的，然后事件上报后 最后由 [WMS](https://so.csdn.net/so/search?q=WMS&spm=1001.2101.3001.7020) 就是 WindowManagerService 来处理旋转的相关事件

### 2. 强制 app 横屏显示的核心类

```
/framework/base/services/java/com/android/server/wm/DisplayRotation.java

```

### 3. 强制 app 横屏显示核心功能分析和处理

关于处理屏幕方法的 api 在 11.0 的系统中也是 DisplayRotation.java 里负责处理的，  
具体需要看源码然后分析具体的旋转功能  
路径为:/framework/base/services/java/com/android/server/wm/DisplayRotation.java

```
int rotationForOrientation(int orientation, int lastRotation) {
if (DEBUG_ORIENTATION) {
Slog.v(TAG, "rotationForOrientation(orient="
+ orientation + ", last=" + lastRotation
+ "); user=" + mUserRotation + " "
+ (mUserRotationMode == WindowManagerPolicy.USER_ROTATION_LOCKED
? "USER_ROTATION_LOCKED" : "")
);
}
if (isFixedToUserRotation()) {
return mUserRotation;
}
  int sensorRotation = mOrientationListener != null
          ? mOrientationListener.getProposedRotation() // may be -1
          : -1;
  if (sensorRotation < 0) {
      sensorRotation = lastRotation;
  }

  final int lidState = mDisplayPolicy.getLidState();
  final int dockMode = mDisplayPolicy.getDockMode();
  final boolean hdmiPlugged = mDisplayPolicy.isHdmiPlugged();
  final boolean carDockEnablesAccelerometer =
          mDisplayPolicy.isCarDockEnablesAccelerometer();
  final boolean deskDockEnablesAccelerometer =
          mDisplayPolicy.isDeskDockEnablesAccelerometer();

  final int preferredRotation;
  if (!isDefaultDisplay) {
      // For secondary displays we ignore things like displays sensors, docking mode and
      // rotation lock, and always prefer user rotation.
      preferredRotation = mUserRotation;
  } else if (lidState == LID_OPEN && mLidOpenRotation >= 0) {
      // Ignore sensor when lid switch is open and rotation is forced.
     preferredRotation = mLidOpenRotation;
  } else if (dockMode == Intent.EXTRA_DOCK_STATE_CAR
          && (carDockEnablesAccelerometer || mCarDockRotation >= 0)) {
      // Ignore sensor when in car dock unless explicitly enabled.
      // This case can override the behavior of NOSENSOR, and can also
      // enable 180 degree rotation while docked.
      preferredRotation = carDockEnablesAccelerometer ? sensorRotation : mCarDockRotation;
  } else if ((dockMode == Intent.EXTRA_DOCK_STATE_DESK
          || dockMode == Intent.EXTRA_DOCK_STATE_LE_DESK
          || dockMode == Intent.EXTRA_DOCK_STATE_HE_DESK)
          && (deskDockEnablesAccelerometer || mDeskDockRotation >= 0)) {
      // Ignore sensor when in desk dock unless explicitly enabled.
      // This case can override the behavior of NOSENSOR, and can also
      // enable 180 degree rotation while docked.
      preferredRotation = deskDockEnablesAccelerometer ? sensorRotation : mDeskDockRotation;
  } else if (hdmiPlugged && mDemoHdmiRotationLock) {
      // Ignore sensor when plugged into HDMI when demo HDMI rotation lock enabled.
      // Note that the dock orientation overrides the HDMI orientation.
      preferredRotation = mDemoHdmiRotation;
  } else if (hdmiPlugged && dockMode == Intent.EXTRA_DOCK_STATE_UNDOCKED
          && mUndockedHdmiRotation >= 0) {
      // Ignore sensor when plugged into HDMI and an undocked orientation has
      // been specified in the configuration (only for legacy devices without
      // full multi-display support).
      // Note that the dock orientation overrides the HDMI orientation.
      preferredRotation = mUndockedHdmiRotation;
  } else if (mDemoRotationLock) {
      // Ignore sensor when demo rotation lock is enabled.
      // Note that the dock orientation and HDMI rotation lock override this.
      preferredRotation = mDemoRotation;
  } else if (mDisplayPolicy.isPersistentVrModeEnabled()) {
      // While in VR, apps always prefer a portrait rotation. This does not change
      // any apps that explicitly set landscape, but does cause sensors be ignored,
      // and ignored any orientation lock that the user has set (this conditional
      // should remain above the ORIENTATION_LOCKED conditional below).
      preferredRotation = mPortraitRotation;
  } else if (orientation == ActivityInfo.SCREEN_ORIENTATION_LOCKED) {
      // Application just wants to remain locked in the last rotation.
      preferredRotation = lastRotation;
  } else if (!mSupportAutoRotation) {
      // If we don't support auto-rotation then bail out here and ignore
      // the sensor and any rotation lock settings.
      preferredRotation = -1;
  } else if ((mUserRotationMode == WindowManagerPolicy.USER_ROTATION_FREE
                  && (orientation == ActivityInfo.SCREEN_ORIENTATION_USER
                          || orientation == ActivityInfo.SCREEN_ORIENTATION_UNSPECIFIED
                          || orientation == ActivityInfo.SCREEN_ORIENTATION_USER_LANDSCAPE
                          || orientation == ActivityInfo.SCREEN_ORIENTATION_USER_PORTRAIT
                          || orientation == ActivityInfo.SCREEN_ORIENTATION_FULL_USER))
          || orientation == ActivityInfo.SCREEN_ORIENTATION_SENSOR
          || orientation == ActivityInfo.SCREEN_ORIENTATION_FULL_SENSOR
          || orientation == ActivityInfo.SCREEN_ORIENTATION_SENSOR_LANDSCAPE
          || orientation == ActivityInfo.SCREEN_ORIENTATION_SENSOR_PORTRAIT) {
      // Otherwise, use sensor only if requested by the application or enabled
      // by default for USER or UNSPECIFIED modes.  Does not apply to NOSENSOR.
      if (mAllowAllRotations < 0) {
          // Can't read this during init() because the context doesn't
          // have display metrics at that time so we cannot determine
          // tablet vs. phone then.
          mAllowAllRotations = mContext.getResources().getBoolean(
                  com.android.internal.R.bool.config_allowAllRotations) ? 1 : 0;
      }
      if (sensorRotation != Surface.ROTATION_180
              || mAllowAllRotations == 1
              || orientation == ActivityInfo.SCREEN_ORIENTATION_FULL_SENSOR
              || orientation == ActivityInfo.SCREEN_ORIENTATION_FULL_USER) {
          preferredRotation = sensorRotation;
      } else {
          preferredRotation = lastRotation;
      }
  } else if (mUserRotationMode == WindowManagerPolicy.USER_ROTATION_LOCKED
          && orientation != ActivityInfo.SCREEN_ORIENTATION_NOSENSOR) {
      // Apply rotation lock.  Does not apply to NOSENSOR.
      // The idea is that the user rotation expresses a weak preference for the direction
      // of gravity and as NOSENSOR is never affected by gravity, then neither should
      // NOSENSOR be affected by rotation lock (although it will be affected by docks).
      preferredRotation = mUserRotation;
  } else {
      // No overriding preference.
      // We will do exactly what the application asked us to do.
      preferredRotation = -1;
  }

  switch (orientation) {
      case ActivityInfo.SCREEN_ORIENTATION_PORTRAIT:
          // Return portrait unless overridden.
          if (isAnyPortrait(preferredRotation)) {
              return preferredRotation;
          }
          return mPortraitRotation;

      case ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE:
          // Return landscape unless overridden.
          if (isLandscapeOrSeascape(preferredRotation)) {
              return preferredRotation;
          }
          return mLandscapeRotation;

      case ActivityInfo.SCREEN_ORIENTATION_REVERSE_PORTRAIT:
          // Return reverse portrait unless overridden.
          if (isAnyPortrait(preferredRotation)) {
              return preferredRotation;
          }
          return mUpsideDownRotation;

      case ActivityInfo.SCREEN_ORIENTATION_REVERSE_LANDSCAPE:
          // Return seascape unless overridden.
          if (isLandscapeOrSeascape(preferredRotation)) {
              return preferredRotation;
          }
          return mSeascapeRotation;

      case ActivityInfo.SCREEN_ORIENTATION_SENSOR_LANDSCAPE:
      case ActivityInfo.SCREEN_ORIENTATION_USER_LANDSCAPE:
          // Return either landscape rotation.
          if (isLandscapeOrSeascape(preferredRotation)) {
              return preferredRotation;
          }
          if (isLandscapeOrSeascape(lastRotation)) {
              return lastRotation;
          }
          return mLandscapeRotation;

      case ActivityInfo.SCREEN_ORIENTATION_SENSOR_PORTRAIT:
      case ActivityInfo.SCREEN_ORIENTATION_USER_PORTRAIT:
          // Return either portrait rotation.
          if (isAnyPortrait(preferredRotation)) {
              return preferredRotation;
          }
          if (isAnyPortrait(lastRotation)) {
              return lastRotation;
          }
          return mPortraitRotation;

      default:
          // For USER, UNSPECIFIED, NOSENSOR, SENSOR and FULL_SENSOR,
          // just return the preferred orientation we already calculated.
          if (preferredRotation >= 0) {
              return preferredRotation;
          }
          return Surface.ROTATION_0;
  }

}

```

rotationForOrientation(int orientation, int lastRotation) 来负责处理 app 具体的方向旋转当 app 进入到 activity 后由 rotationForOrientation() 来判断屏幕需要显示的方向，最后处理屏幕的具体旋转方向  
具体是在

```
 switch (orientation) {
      case ActivityInfo.SCREEN_ORIENTATION_PORTRAIT:
          // Return portrait unless overridden.
          if (isAnyPortrait(preferredRotation)) {
              return preferredRotation;
          }
          return mPortraitRotation;

```

根据解析 app 的 orientation 的方向值，返回具体的旋转方向  
所以强制 app 横屏做如下修改:

```
case ActivityInfo.SCREEN_ORIENTATION_PORTRAIT:
// Return portrait unless overridden.
if (isAnyPortrait(preferredRotation)) {
return preferredRotation;
}
- return mPortraitRotation;
+ return Surface.ROTATION_90;
case ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE:
// Return landscape unless overridden.
if (isLandscapeOrSeascape(preferredRotation)) {
return preferredRotation;
}
 return mLandscapeRotation;
}

```

当处理屏幕旋转方向时，需要处理当是  
case ActivityInfo.SCREEN_ORIENTATION_PORTRAIT 的时候，需要旋转到什么方向，进入 app 的时候当设置屏幕方向的时候，设置方向旋转 90 度，在这里处理旋转 90 的屏幕方向就可以了从而实现功能
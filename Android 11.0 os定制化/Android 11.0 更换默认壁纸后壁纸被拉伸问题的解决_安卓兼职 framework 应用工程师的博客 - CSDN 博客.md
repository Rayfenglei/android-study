> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/128377935)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 更换默认壁纸后壁纸被拉伸问题的解决的核心类](#t1)

[3. 更换默认壁纸后壁纸被拉伸问题的解决的核心功能分析和实现](#t2)

 [3.1WallpaperWindowToken.java 关于 Wallpaper 的管理](#t3)

[3.2 WallpaperController 关于设置壁纸的相关源码分析](#t4)

[3.3 config.xml 中关于 mMaxWallpaperScale 的相关参数设置](#t5)

1. 概述
-----

  在 11.0 的系统原生开发中，更换默认[壁纸](https://so.csdn.net/so/search?q=%E5%A3%81%E7%BA%B8&spm=1001.2101.3001.7020)也是常有的工作，但是在更换默认壁纸后，发现 壁纸被默认拉伸了，显然不符合美观要求，所以需要找到默认拉伸的原因，还原被默认拉伸的壁纸就好了，

2. 更换默认壁纸后壁纸被拉伸问题的解决的核心类
------------------------

```
frameworks/base/services/core/java/com/android/server/wm/WallpaperController.java
frameworks/base/services/core/java/com/android/server/wm/WallpaperWindowToken.java
frameworks/base/core/res/res/values/config.xml
```

3. 更换默认壁纸后壁纸被拉伸问题的解决的核心功能分析和实现
------------------------------

  3.1WallpaperWindowToken.java 关于 Wallpaper 的管理
-----------------------------------------------

```
    class WallpaperWindowToken extends WindowToken {
  
      private static final String TAG = TAG_WITH_CLASS_NAME ? "WallpaperWindowToken" : TAG_WM;
  
      WallpaperWindowToken(WindowManagerService service, IBinder token, boolean explicit,
              DisplayContent dc, boolean ownerCanManageAppTokens) {
          super(service, token, TYPE_WALLPAPER, explicit, dc, ownerCanManageAppTokens);
          dc.mWallpaperController.addWallpaperToken(this);
          setWindowingMode(WINDOWING_MODE_FULLSCREEN);
      }
 
     void sendWindowWallpaperCommand(
             String action, int x, int y, int z, Bundle extras, boolean sync) {
         for (int wallpaperNdx = mChildren.size() - 1; wallpaperNdx >= 0; wallpaperNdx--) {
             final WindowState wallpaper = mChildren.get(wallpaperNdx);
             try {
                 wallpaper.mClient.dispatchWallpaperCommand(action, x, y, z, extras, sync);
                 // We only want to be synchronous with one wallpaper.
                 sync = false;
             } catch (RemoteException e) {
             }
         }
     }
 
     void updateWallpaperOffset(boolean sync) {
         final WallpaperController wallpaperController = mDisplayContent.mWallpaperController;
         for (int wallpaperNdx = mChildren.size() - 1; wallpaperNdx >= 0; wallpaperNdx--) {
             final WindowState wallpaper = mChildren.get(wallpaperNdx);
             if (wallpaperController.updateWallpaperOffset(wallpaper, sync)) {
                 // We only want to be synchronous with one wallpaper.
                 sync = false;
             }
         }
     }
 
     void updateWallpaperVisibility(boolean visible) {
         if (isVisible() != visible) {
             // Need to do a layout to ensure the wallpaper now has the correct size.
             mDisplayContent.setLayoutNeeded();
         }
 
         final WallpaperController wallpaperController = mDisplayContent.mWallpaperController;
         for (int wallpaperNdx = mChildren.size() - 1; wallpaperNdx >= 0; wallpaperNdx--) {
             final WindowState wallpaper = mChildren.get(wallpaperNdx);
             if (visible) {
                 wallpaperController.updateWallpaperOffset(wallpaper, false /* sync */);
              }
  
              wallpaper.dispatchWallpaperVisibility(visible);
          }
      }
```

在 WallpaperWindowToken 的源码中，可以看出主要是通过 WallpaperController 来设置当前壁纸的宽高等等，在 updateWallpaperVisibility(boolean visible)  
中来设置壁纸是否显示，如果显示壁纸则调用 updateWallpaperOffset 来设置壁纸的相关尺寸，updateWallpaperOffset(boolean sync)  
则是主要来更新壁纸的，sendWindowWallpaperCommand(  
             String action, int x, int y, int z, Bundle extras, boolean sync) 主要是设置壁纸的相关指令  
接下来具体分析下 WallpaperController 的相关源码

3.2 WallpaperController 关于设置壁纸的相关源码分析
-------------------------------------

```
  /**
   * Controls wallpaper windows visibility, ordering, and so on.
   * NOTE: All methods in this class must be called with the window manager service lock held.
   */
  class WallpaperController {
      private static final String TAG = TAG_WITH_CLASS_NAME ? "WallpaperController" : TAG_WM;
      private WindowManagerService mService;
      private final DisplayContent mDisplayContent;
  
      private final ArrayList<WallpaperWindowToken> mWallpaperTokens = new ArrayList<>();
  
      // If non-null, this is the currently visible window that is associated
      // with the wallpaper.
      private WindowState mWallpaperTarget = null;
      // If non-null, we are in the middle of animating from one wallpaper target
      // to another, and this is the previous wallpaper target.
      private WindowState mPrevWallpaperTarget = null;
  
      private float mLastWallpaperX = -1;
      private float mLastWallpaperY = -1;
      private float mLastWallpaperXStep = -1;
      private float mLastWallpaperYStep = -1;
      private float mLastWallpaperZoomOut = 0;
      private int mLastWallpaperDisplayOffsetX = Integer.MIN_VALUE;
      private int mLastWallpaperDisplayOffsetY = Integer.MIN_VALUE;
      private final float mMaxWallpaperScale;
....
  WallpaperController(WindowManagerService service, DisplayContent displayContent) {
          mService = service;
          mDisplayContent = displayContent;
          mMaxWallpaperScale = service.mContext.getResources()
                  .getFloat(com.android.internal.R.dimen.config_wallpaperMaxScale);
      }
```

在 WallpaperController 的上述参数和构造方法中，发现 mMaxWallpaperScale 就是系统默认壁纸的缩放比例，根据 mMaxWallpaperScale  
这个壁纸缩放比例在 updateWallpaperOffset(WindowState wallpaperWin, boolean sync) 中设置壁纸，接下来看下  
updateWallpaperOffset(WindowState wallpaperWin, boolean sync) 的相关壁纸设置

```
   boolean updateWallpaperOffset(WindowState wallpaperWin, boolean sync) {
         final DisplayInfo displayInfo = wallpaperWin.getDisplayInfo();
         final int dw = displayInfo.logicalWidth;
         final int dh = displayInfo.logicalHeight;
 
         int xOffset = 0;
         int yOffset = 0;
         boolean rawChanged = false;
         // Set the default wallpaper x-offset to either edge of the screen (depending on RTL), to
         // match the behavior of most Launchers
         float defaultWallpaperX = wallpaperWin.isRtl() ? 1f : 0f;
         float wpx = mLastWallpaperX >= 0 ? mLastWallpaperX : defaultWallpaperX;
         float wpxs = mLastWallpaperXStep >= 0 ? mLastWallpaperXStep : -1.0f;
         int availw = wallpaperWin.getFrameLw().right - wallpaperWin.getFrameLw().left - dw;
         int offset = availw > 0 ? -(int)(availw * wpx + .5f) : 0;
         if (mLastWallpaperDisplayOffsetX != Integer.MIN_VALUE) {
             offset += mLastWallpaperDisplayOffsetX;
         }
         xOffset = offset;
 
         if (wallpaperWin.mWallpaperX != wpx || wallpaperWin.mWallpaperXStep != wpxs) {
             wallpaperWin.mWallpaperX = wpx;
             wallpaperWin.mWallpaperXStep = wpxs;
             rawChanged = true;
         }
 
         float wpy = mLastWallpaperY >= 0 ? mLastWallpaperY : 0.5f;
         float wpys = mLastWallpaperYStep >= 0 ? mLastWallpaperYStep : -1.0f;
         int availh = wallpaperWin.getFrameLw().bottom - wallpaperWin.getFrameLw().top - dh;
         offset = availh > 0 ? -(int)(availh * wpy + .5f) : 0;
         if (mLastWallpaperDisplayOffsetY != Integer.MIN_VALUE) {
             offset += mLastWallpaperDisplayOffsetY;
         }
         yOffset = offset;
 
         if (wallpaperWin.mWallpaperY != wpy || wallpaperWin.mWallpaperYStep != wpys) {
             wallpaperWin.mWallpaperY = wpy;
             wallpaperWin.mWallpaperYStep = wpys;
             rawChanged = true;
         }
 
         if (Float.compare(wallpaperWin.mWallpaperZoomOut, mLastWallpaperZoomOut) != 0) {
             wallpaperWin.mWallpaperZoomOut = mLastWallpaperZoomOut;
             rawChanged = true;
         }
 
         boolean changed = wallpaperWin.mWinAnimator.setWallpaperOffset(xOffset, yOffset,
                 wallpaperWin.mShouldScaleWallpaper
                         ? zoomOutToScale(wallpaperWin.mWallpaperZoomOut) : 1);
 
         if (rawChanged && (wallpaperWin.mAttrs.privateFlags &
                 WindowManager.LayoutParams.PRIVATE_FLAG_WANTS_OFFSET_NOTIFICATIONS) != 0) {
             try {
                 if (DEBUG_WALLPAPER) Slog.v(TAG, "Report new wp offset "
                         + wallpaperWin + " x=" + wallpaperWin.mWallpaperX
                         + " y=" + wallpaperWin.mWallpaperY
                         + " zoom=" + wallpaperWin.mWallpaperZoomOut);
                 if (sync) {
                     mWaitingOnWallpaper = wallpaperWin;
                 }
                 wallpaperWin.mClient.dispatchWallpaperOffsets(
                         wallpaperWin.mWallpaperX, wallpaperWin.mWallpaperY,
                         wallpaperWin.mWallpaperXStep, wallpaperWin.mWallpaperYStep,
                         wallpaperWin.mWallpaperZoomOut, sync);
 
                 if (sync) {
                     if (mWaitingOnWallpaper != null) {
                         long start = SystemClock.uptimeMillis();
                         if ((mLastWallpaperTimeoutTime + WALLPAPER_TIMEOUT_RECOVERY)
                                 < start) {
                             try {
                                 if (DEBUG_WALLPAPER) Slog.v(TAG,
                                         "Waiting for offset complete...");
                                 mService.mGlobalLock.wait(WALLPAPER_TIMEOUT);
                             } catch (InterruptedException e) {
                             }
                             if (DEBUG_WALLPAPER) Slog.v(TAG, "Offset complete!");
                             if ((start + WALLPAPER_TIMEOUT) < SystemClock.uptimeMillis()) {
                                 Slog.i(TAG, "Timeout waiting for wallpaper to offset: "
                                         + wallpaperWin);
                                 mLastWallpaperTimeoutTime = start;
                             }
                         }
                         mWaitingOnWallpaper = null;
                     }
                 }
             } catch (RemoteException e) {
             }
         }
 
         return changed;
}
 
private float zoomOutToScale(float zoom) {
          return MathUtils.lerp(1, mMaxWallpaperScale, 1 - zoom);
      }
```

在 WallpaperController 的上述方法中，可以看到在 updateWallpaperOffset(WindowState wallpaperWin, boolean sync) 中调用  
zoomOutToScale(float zoom) 来具体调用缩放比例来设置系统默认壁纸的宽和高，所以就会拉伸壁纸。

3.3 config.xml 中关于 mMaxWallpaperScale 的相关参数设置
---------------------------------------------

```
   <!-- These resources are around just to allow their values to be customized
      for different hardware and product builds.  Do not translate.
 
      NOTE: The naming convention is "config_camelCaseValue". Some legacy
      entries do not follow the convention, but all new entries should. -->
 
 <resources xmlns:xliff="urn:oasis:names:tc:xliff:document:1.2">
     <!-- Do not translate. Defines the slots for the right-hand side icons.  That is to say, the
          icons in the status bar that are not notifications. -->
  <string ></string>
 
     <!-- The max scale for the wallpaper when it's zoomed in -->
 -    <item >1.10</item>
 +   <item >1.00</item>
     <!-- Package name that will receive an explicit manifest broadcast for
          android.os.action.POWER_SAVE_MODE_CHANGED. -->
     <string ></string>
 
     <!-- Set to true to enable the user switcher on the keyguard. -->
     <bool >false</bool>
```

从上述 config_wallpaperMaxScale 的值可以看出 原来的值为 1.1 会存在拉伸的情况，修改为 1.0 就可以了
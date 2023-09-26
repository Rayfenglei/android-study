> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124828493)

### 1. 概述

在 11.0 产品定制化开发中，由于摄像头方向默认是竖屏的，但是平板电脑一般都是要横屏拍摄的  
所以就需要旋转摄像头方向, 来适应拍摄的需要，这就需要在 Camera 中打开摄像头的时候，设置参数  
旋转摄像头方向

### 2.Camera 旋转摄像头方向的核心类

```
frameworks/base/core/java/android/hardware/camera2/CameraManager.java
frameworks/base/core/java/android/hardware/Camera.java

```

### 3.Camera 旋转摄像头方向的功能实现和分析

### 3.1 CameraManager.java 关于 Camera 的相关 api 分析

```
public final class CameraManager {
  
      private static final String TAG = "CameraManager";
      private final boolean DEBUG = false;
  
      private static final int USE_CALLING_UID = -1;
  
      @SuppressWarnings("unused")
      private static final int API_VERSION_1 = 1;
      private static final int API_VERSION_2 = 2;
  
      private static final int CAMERA_TYPE_BACKWARD_COMPATIBLE = 0;
      private static final int CAMERA_TYPE_ALL = 1;
  
      private ArrayList<String> mDeviceIdList;
  
      private final Context mContext;
      private final Object mLock = new Object();
  
      /**
       * @hide
       */
      public CameraManager(Context context) {
          synchronized(mLock) {
              mContext = context;
          }
      }
   @NonNull
      public String[] getCameraIdList() throws CameraAccessException {
          return CameraManagerGlobal.get().getCameraIdList();
      }
   @TestApi
      public String[] getCameraIdListNoLazy() throws CameraAccessException {
          return CameraManagerGlobal.get().getCameraIdListNoLazy();
      }
      ...
      /**
       * Return the set of combinations of currently connected camera device identifiers, which
       * support configuring camera device sessions concurrently.
       *
       * <p>The devices in these combinations can be concurrently configured by the same
       * client camera application. Using these camera devices concurrently by two different
       * applications is not guaranteed to be supported, however.</p>
       *
       * <p>For concurrent operation, in chronological order :
       * - Applications must first close any open cameras that have sessions configured, using
       *   {@link CameraDevice#close}.
       * - All camera devices intended to be operated concurrently, must be opened using
       *   {@link #openCamera}, before configuring sessions on any of the camera devices.</p>
       *
       * <p>Each device in a combination, is guaranteed to support stream combinations which may be
       * obtained by querying {@link #getCameraCharacteristics} for the key
       * {@link android.hardware.camera2.CameraCharacteristics#SCALER_MANDATORY_CONCURRENT_STREAM_COMBINATIONS}.</p>
       *
       * <p>For concurrent operation, if a camera device has a non null zoom ratio range as specified
       * by
       * {@link android.hardware.camera2.CameraCharacteristics#CONTROL_ZOOM_RATIO_RANGE},
       * its complete zoom ratio range may not apply. Applications can use
       * {@link android.hardware.camera2.CaptureRequest#CONTROL_ZOOM_RATIO} >=1 and  <=
       * {@link android.hardware.camera2.CameraCharacteristics#SCALER_AVAILABLE_MAX_DIGITAL_ZOOM}
       * during concurrent operation.
       * <p>
       *
       * <p>The set of combinations may include camera devices that may be in use by other camera API
       * clients.</p>
       *
       * <p>The set of combinations doesn't contain physical cameras that can only be used as
       * part of a logical multi-camera device.</p>
       *
       * <p> If a new camera id becomes available through
       * {@link AvailabilityCallback#onCameraUnavailable(String)}, clients can call
       * this method to check if new combinations of camera ids which can stream concurrently are
       * available.
       *
       * @return The set of combinations of currently connected camera devices, that may have
       *         sessions configured concurrently. The set of combinations will be empty if no such
       *         combinations are supported by the camera subsystem.
       *
       * @throws CameraAccessException if the camera device has been disconnected.
       */
      @NonNull
      public Set<Set<String>> getConcurrentCameraIds() throws CameraAccessException {
          return CameraManagerGlobal.get().getConcurrentCameraIds();
      }

```

在 CameraManager 的 camera 管理类中，getCameraIdList() 和 getCameraIdListNoLazy() 为不同形式的 camera 摄像头列表，在进行打开摄像头的工作中，首先获取摄像头列表，然后判断使用哪个摄像头来进行拍照

### 3.2 Camera.java 关于摄像头功能分析

Camera 是相机的关键类，主要是通过调用 open 方法来打开摄像头工作  
接下来看 Camera.java 的源码

```
public class Camera {
      private static final String TAG = "Camera";
  
      // These match the enums in frameworks/base/include/camera/Camera.h
      private static final int CAMERA_MSG_ERROR            = 0x001;
      private static final int CAMERA_MSG_SHUTTER          = 0x002;
      private static final int CAMERA_MSG_FOCUS            = 0x004;
      private static final int CAMERA_MSG_ZOOM             = 0x008;
      private static final int CAMERA_MSG_PREVIEW_FRAME    = 0x010;
      private static final int CAMERA_MSG_VIDEO_FRAME      = 0x020;
      private static final int CAMERA_MSG_POSTVIEW_FRAME   = 0x040;
      private static final int CAMERA_MSG_RAW_IMAGE        = 0x080;
      private static final int CAMERA_MSG_COMPRESSED_IMAGE = 0x100;
      private static final int CAMERA_MSG_RAW_IMAGE_NOTIFY = 0x200;
      private static final int CAMERA_MSG_PREVIEW_METADATA = 0x400;
      private static final int CAMERA_MSG_FOCUS_MOVE       = 0x800;
	  ....
	   public native static int getNumberOfCameras();
  
      /**
       * Returns the information about a particular camera.
       * If {@link #getNumberOfCameras()} returns N, the valid id is 0 to N-1.
       *
       * @throws RuntimeException if an invalid ID is provided, or if there is an
       *    error retrieving the information (generally due to a hardware or other
       *    low-level failure).
       */
      public static void getCameraInfo(int cameraId, CameraInfo cameraInfo) {
          _getCameraInfo(cameraId, cameraInfo);
          IBinder b = ServiceManager.getService(Context.AUDIO_SERVICE);
          IAudioService audioService = IAudioService.Stub.asInterface(b);
          try {
              if (audioService.isCameraSoundForced()) {
                  // Only set this when sound is forced; otherwise let native code
                  // decide.
                  cameraInfo.canDisableShutterSound = false;
              }
          } catch (RemoteException e) {
              Log.e(TAG, "Audio service is unavailable for queries");
          }
      }
public static class CameraInfo {
          /**
           * The facing of the camera is opposite to that of the screen.
           */
          public static final int CAMERA_FACING_BACK = 0;
  
          /**
           * The facing of the camera is the same as that of the screen.
           */
          public static final int CAMERA_FACING_FRONT = 1;
  
          /**
           * The direction that the camera faces. It should be
           * CAMERA_FACING_BACK or CAMERA_FACING_FRONT.
           */
          public int facing;
             public int orientation;
			 }
public static Camera open(int cameraId) {
return new Camera(cameraId);
}

public static Camera open() {
    int numberOfCameras = getNumberOfCameras();
    CameraInfo cameraInfo = new CameraInfo();
    for (int i = 0; i < numberOfCameras; i++) {
        getCameraInfo(i, cameraInfo);
        if (cameraInfo.facing == CameraInfo.CAMERA_FACING_BACK) {
            return new Camera(i);
        }
    }
    return null;
}

```

在 CameraInfo 中的参数 前后摄像头 摄像头方向等 api

每次在调用 open 的时候 打开摄像头  
通过查看 Camera 的 Parameters 的 setRotation() 可以旋转摄像头方向  
所以可以在构造 Camera 时，设置好旋转方向  
添加旋转方向如下:

```
private static Camera rotateCameraDirection(int cameraId) {
Camera rationcamera = new Camera(cameraId);
Parameters parameters = rationcamera.getParameters();
CameraInfo camerainfo = new CameraInfo();
getCameraInfo(cameraId, camerainfo);
if (camerainfo.facing == CameraInfo.CAMERA_FACING_BACK) {
rationcamera.setDisplayOrientation(270);
parameters.setRotation(270);
} else if (cameraInfo.facing == CameraInfo.CAMERA_FACING_FRONT) {
rationcamera.setDisplayOrientation(90);
parameters.setRotation(90);
}
rationcamera.setParameters(parameters);
return camera;
}

```

旋转方向修改如下:

```
public static Camera open(int cameraId) {
- return new Camera(cameraId);
+ return rotateCameraDirection(cameraId);
}
public static Camera open() {
int numberOfCameras = getNumberOfCameras();
CameraInfo cameraInfo = new CameraInfo();
for (int i = 0; i < numberOfCameras; i++) {
getCameraInfo(i, cameraInfo);
if (cameraInfo.facing == CameraInfo.CAMERA_FACING_BACK) {
- return new Camera(i);
+ return rotateCameraDirection(i);
}
}
return null;
}

```
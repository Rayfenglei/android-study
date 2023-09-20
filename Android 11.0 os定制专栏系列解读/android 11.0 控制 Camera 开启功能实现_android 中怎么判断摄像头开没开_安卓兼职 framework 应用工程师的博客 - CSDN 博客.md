> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124955263)

1. 概述
-----

在 11.0 的产品开发中，需要增加系统属性， 通过系统属性值来控制 camera 开关来实现是否可用 camera 的目的, 这就需要通过相关管理类来控制相机是否可用打开来实现

2. 控制 Camera 开启功能实现的核心代码
------------------------

```
frameworks/base/core/java/android/hardware/camera2/CameraManager.java
frameworks/base/core/java/android/hardware/Camera.java
frameworks/base/services/devicepolicy/java/com/android/server/devicepolicy/DevicePolicyManagerService.java

```

3. 控制 Camera 开启功能实现的核心代码
------------------------

3.1 控制系统打开 camera，通过 CameraManager 来实现控制打开 camera
-------------------------------------------------

在 CameraManager 中打开 camera 的时候根据属性来判断是否打开摄像头

```
 private CameraDevice openCameraDeviceUserAsync(String cameraId,
              CameraDevice.StateCallback callback, Executor executor, final int uid)
              throws CameraAccessException {
          CameraCharacteristics characteristics = getCameraCharacteristics(cameraId);
          CameraDevice device = null;
  
          synchronized (mLock) {
  
              ICameraDeviceUser cameraUser = null;
  
              android.hardware.camera2.impl.CameraDeviceImpl deviceImpl =
                      new android.hardware.camera2.impl.CameraDeviceImpl(
                          cameraId,
                          callback,
                          executor,
                          characteristics,
                          mContext.getApplicationInfo().targetSdkVersion);
  
              ICameraDeviceCallbacks callbacks = deviceImpl.getCallbacks();
  
              try {
                  if (supportsCamera2ApiLocked(cameraId)) {
                      // Use cameraservice's cameradeviceclient implementation for HAL3.2+ devices
                      ICameraService cameraService = CameraManagerGlobal.get().getCameraService();
                      if (cameraService == null) {
                          throw new ServiceSpecificException(
                              ICameraService.ERROR_DISCONNECTED,
                              "Camera service is currently unavailable");
                      }
                      cameraUser = cameraService.connectDevice(callbacks, cameraId,
                              mContext.getOpPackageName(), mContext.getAttributionTag(), uid);
                  } else {
                      // Use legacy camera implementation for HAL1 devices
                      int id;
                      try {
                          id = Integer.parseInt(cameraId);
                      } catch (NumberFormatException e) {
                          throw new IllegalArgumentException("Expected cameraId to be numeric, but it was: "
                                  + cameraId);
                      }
  
                      Log.i(TAG, "Using legacy camera HAL.");
                      cameraUser = CameraDeviceUserShim.connectBinderShim(callbacks, id,
                              getDisplaySize());
                  }
              } catch (ServiceSpecificException e) {
                  if (e.errorCode == ICameraService.ERROR_DEPRECATED_HAL) {
                      throw new AssertionError("Should've gone down the shim path");
                  } else if (e.errorCode == ICameraService.ERROR_CAMERA_IN_USE ||
                          e.errorCode == ICameraService.ERROR_MAX_CAMERAS_IN_USE ||
                          e.errorCode == ICameraService.ERROR_DISABLED ||
                          e.errorCode == ICameraService.ERROR_DISCONNECTED ||
                          e.errorCode == ICameraService.ERROR_INVALID_OPERATION) {
                      // Received one of the known connection errors
                      // The remote camera device cannot be connected to, so
                      // set the local camera to the startup error state
                      deviceImpl.setRemoteFailure(e);
  
                      if (e.errorCode == ICameraService.ERROR_DISABLED ||
                              e.errorCode == ICameraService.ERROR_DISCONNECTED ||
                              e.errorCode == ICameraService.ERROR_CAMERA_IN_USE) {
                          // Per API docs, these failures call onError and throw
                          throwAsPublicException(e);
                      }
                  } else {
                      // Unexpected failure - rethrow
                      throwAsPublicException(e);
                  }
              } catch (RemoteException e) {
                  // Camera service died - act as if it's a CAMERA_DISCONNECTED case
                  ServiceSpecificException sse = new ServiceSpecificException(
                      ICameraService.ERROR_DISCONNECTED,
                      "Camera service is currently unavailable");
                  deviceImpl.setRemoteFailure(sse);
                  throwAsPublicException(sse);
              }
  
              // TODO: factor out callback to be non-nested, then move setter to constructor
              // For now, calling setRemoteDevice will fire initial
              // onOpened/onUnconfigured callbacks.
              // This function call may post onDisconnected and throw CAMERA_DISCONNECTED if
              // cameraUser dies during setup.
              deviceImpl.setRemoteDevice(cameraUser);
              device = deviceImpl;
          }
  
          return device;
      }
      
      具体相关功能修改为：
      
--- a/frameworks/base/core/java/android/hardware/camera2/CameraManager.java
+++ b/frameworks/base/core/java/android/hardware/camera2/CameraManager.java
@@ -52,7 +52,7 @@ import java.util.concurrent.Executors;
 import java.util.concurrent.RejectedExecutionException;
 import java.util.concurrent.ScheduledExecutorService;
 import java.util.concurrent.TimeUnit;
-
+import android.os.SystemProperties;
 //SPRD import
 import android.app.ActivityThread;
 import android.hardware.Camera;
@@ -378,7 +378,11 @@ public final class CameraManager {
             throws CameraAccessException {
         CameraCharacteristics characteristics = getCameraCharacteristics(cameraId);
         CameraDevice device = null;
-
+               String flag = SystemProperties.get("persist.sys.disableCamera", "false");
+               Log.d("dong","setBluetoothEnabled:"+flag);
+               if("true".equals(flag)){
+                       return null;
+               }
         synchronized (mLock) {
 
             ICameraDeviceUser cameraUser = null;

```

通过 openCameraDeviceUserAsync 来调用摄像头来打开 camera，这时可以通过添加系统属性来限制是否可以摄像头

3.2 控制第三方 app 打开 camera
-----------------------

第三方 app 是通过调用 camera 的 open 方法 api 来打开 camera  
所以在 camera 中相关 open 的地方添加限制

```
--- a/frameworks/base/core/java/android/hardware/Camera.java
+++ b/frameworks/base/core/java/android/hardware/Camera.java
@@ -52,13 +52,12 @@ import android.view.SurfaceHolder;
 import com.android.internal.annotations.GuardedBy;
 import com.android.internal.app.IAppOpsCallback;
 import com.android.internal.app.IAppOpsService;
-
+import android.os.SystemProperties;
 import java.io.IOException;
 import java.lang.ref.WeakReference;
 import java.util.ArrayList;
 import java.util.LinkedHashMap;
 import java.util.List;
-
 /**
  * The Camera class is used to set image capture settings, start/stop preview,
  * snap pictures, and retrieve frames for encoding for video.  This class is a
@@ -411,6 +410,11 @@ public class Camera {
      * @see android.app.admin.DevicePolicyManager#getCameraDisabled(android.content.ComponentName)
      */
     public static Camera open(int cameraId) {
+               String flag = SystemProperties.get("persist.sys.disableCamera", "false");
+               Log.d("dong","setBluetoothEnabled:"+flag);
+               if("true".equals(flag)){
+                       return null;
+               }
         return new Camera(cameraId);
     }
 
@@ -424,6 +428,11 @@ public class Camera {
      * @see #open(int)
      */
     public static Camera open() {
+               String flag = SystemProperties.get("persist.sys.disableCamera", "false");
+               Log.d("dong","setBluetoothEnabled:"+flag);
+               if("true".equals(flag)){
+                       return null;
+               }
         int numberOfCameras = getNumberOfCameras();
         CameraInfo cameraInfo = new CameraInfo();
         for (int i = 0; i < numberOfCameras; i++) {
@@ -478,7 +487,11 @@ public class Camera {
         if (halVersion < CAMERA_HAL_API_VERSION_1_0) {
             throw new IllegalArgumentException("Invalid HAL version " + halVersion);
         }
-
+               String flag = SystemProperties.get("persist.sys.disableCamera", "false");
+               Log.d("dong","setBluetoothEnabled:"+flag);
+               if("true".equals(flag)){
+                       return null;
+               }
         return new Camera(cameraId, halVersion);
     }
 
@@ -600,6 +613,11 @@ public class Camera {
      * @hide
      */
     public static Camera openUninitialized() {
+               String flag = SystemProperties.get("persist.sys.disableCamera", "false");
+               Log.d("dong","setBluetoothEnabled:"+flag);
+               if("true".equals(flag)){
+                       return null;
+               }
         return new Camera();
     }

```

3.3 DevicePolicyManagerService 关于对摄像头的控制
----------------------------------------

在对硬件设备的管理类中添加系统属性的控制 hasGrantedPolicy 判断是否有[权限控制](https://so.csdn.net/so/search?q=%E6%9D%83%E9%99%90%E6%8E%A7%E5%88%B6&spm=1001.2101.3001.7020)

```
@Override
    public boolean hasGrantedPolicy(ComponentName adminReceiver, int policyId, int userHandle) {
        if (!mHasFeature) {
            return false;
        }
        enforceFullCrossUsersPermission(userHandle);
        synchronized (getLockObject()) {
            ActiveAdmin administrator = getActiveAdminUncheckedLocked(adminReceiver, userHandle);
            if (administrator == null) {
                throw new SecurityException("No active admin " + adminReceiver);
            }
            return administrator.info.usesPolicy(policyId);
        }
    }
    具体修改为：
--- a/frameworks/base/services/devicepolicy/java/com/android/server/devicepolicy/DevicePolicyManagerService.java
+++ b/frameworks/base/services/devicepolicy/java/com/android/server/devicepolicy/DevicePolicyManagerService.java
@@ -393,7 +393,16 @@ public class DevicePolicyManagerService extends BaseIDevicePolicyManager {
             DELEGATION_NETWORK_LOGGING,
             DELEGATION_CERT_SELECTION,
     });

     /**
      *  System property whose value is either "true" or "false", indicating whether
      *  device owner is present.
@@ -7632,6 +7641,13 @@ public class DevicePolicyManagerService extends BaseIDevicePolicyManager {
         if (!mHasFeature) {
             return false;
         }
+               if(SystemProperties.get("persist.sys.disableCamera").equals("true")){
+            return true;
+         }
+                
+               if(SystemProperties.get("persist.sys.disableCamera").equals("false")){
+            return false;
+         }
         synchronized (getLockObject()) {
             if (who != null) {
                 ActiveAdmin admin = getActiveAdminUncheckedLocked(who, userHandle);

```
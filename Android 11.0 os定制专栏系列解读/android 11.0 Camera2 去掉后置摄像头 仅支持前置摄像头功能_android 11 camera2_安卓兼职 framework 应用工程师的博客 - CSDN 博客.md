> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124717126)

1. 概述
-----

在定制化 11.0 的产品时，只有一个前置摄像头单摄像头，这时调用相机时就需要默认打开前置摄像头  
就需要来看调用摄像头这块的代码，屏蔽掉后置摄像头的调用 api 就可以了

2.[Camera2](https://so.csdn.net/so/search?q=Camera2&spm=1001.2101.3001.7020) 去掉后置摄像头 仅支持前置摄像头功能核心类
--------------------------------------------------------------------------------------------------

```
/packages/apps/Camera2/src/com/android/camera/app/CameraController.java

```

3.Camera2 去掉后置摄像头 仅支持前置摄像头功能的核心功能实现和分析
--------------------------------------

在 11.0 系统中，关于摄像头的管理就是在 CameraController.java 中负责管理和实现，所以  
接下来看 CameraController.java 的相关调用代码  
路径:/packages/apps/Camera2/src/com/android/camera/app/CameraController.java

```
public class CameraController implements CameraAgent.CameraOpenCallback, CameraProvider {
     private static final Log.Tag TAG = new Log.Tag("CameraController");
     private static final int EMPTY_REQUEST = -1;
     private final Context mContext;
     private final Handler mCallbackHandler;
     private final CameraAgent mCameraAgent;
     private final CameraAgent mCameraAgentNg;
     private final ActiveCameraDeviceTracker mActiveCameraDeviceTracker;
 
     private CameraAgent.CameraOpenCallback mCallbackReceiver;
       public CameraController(@Nonnull Context context,
           @Nullable CameraAgent.CameraOpenCallback callbackReceiver,
           @Nonnull Handler handler,
           @Nonnull CameraAgent cameraManager,
           @Nonnull CameraAgent cameraManagerNg,
           @Nonnull ActiveCameraDeviceTracker activeCameraDeviceTracker) {
         mContext = context;
         mCallbackReceiver = callbackReceiver;
         mCallbackHandler = handler;
         mCameraAgent = cameraManager;
         // If the new implementation is the same as the old, the
         // CameraAgentFactory decided this device doesn't support the new API.
         mCameraAgentNg = cameraManagerNg != cameraManager ? cameraManagerNg : null;
         mActiveCameraDeviceTracker = activeCameraDeviceTracker;
         mInfo = mCameraAgent.getCameraDeviceInfo();
         if (mInfo == null && mCallbackReceiver != null) {
             mCallbackReceiver.onDeviceOpenFailure(-1, "GETTING_CAMERA_INFO");
         }
     }
 
     @Override
      public void setCameraExceptionHandler(CameraExceptionHandler exceptionHandler) {
          mCameraAgent.setCameraExceptionHandler(exceptionHandler);
          if (mCameraAgentNg != null) {
              mCameraAgentNg.setCameraExceptionHandler(exceptionHandler);
          }
      }
  
      @Override
      public CameraDeviceInfo.Characteristics getCharacteristics(int cameraId) {
          if (mInfo == null) {
              return null;
          }
          return mInfo.getCharacteristics(cameraId);
      }
  
      @Override
      @Deprecated
      public CameraId getCurrentCameraId() {
          return mActiveCameraDeviceTracker.getActiveOrPreviousCamera();
      }
  
      @Override
      public int getNumberOfCameras() {
          if (mInfo == null) {
              return 0;
          }
          return mInfo.getNumberOfCameras();
      }
  
      @Override
      public int getFirstBackCameraId() {
          if (mInfo == null) {
              return -1;
          }
          return mInfo.getFirstBackCameraId();
      }
@Override
public void requestCamera(int id) {
requestCamera(id, false);
}
  @Override
  public void requestCamera(int id, boolean useNewApi) {
      Log.v(TAG, "requestCamera");
      // Based on
      // (mRequestingCameraId == id, mRequestingCameraId == EMPTY_REQUEST),
      // we have (T, T), (T, F), (F, T), (F, F).
      // (T, T): implies id == EMPTY_REQUEST. We don't allow this to happen
      //         here. Return.
      // (F, F): A previous request hasn't been fulfilled yet. Return.
     // (T, F): Already requested the same camera. No-op. Return.
      // (F, T): Nothing is going on. Continue.
      if (mRequestingCameraId != EMPTY_REQUEST || mRequestingCameraId == id) {
          return;
      }
      if (mInfo == null) {
          return;
      }
      mRequestingCameraId = id;
      mActiveCameraDeviceTracker.onCameraOpening(CameraId.fromLegacyId(id));

      // Only actually use the new API if it's supported on this device.
      useNewApi = mCameraAgentNg != null && useNewApi;
     CameraAgent cameraManager = useNewApi ? mCameraAgentNg : mCameraAgent;

      if (mCameraProxy == null) {
          // No camera yet.
          checkAndOpenCamera(cameraManager, id, mCallbackHandler, this);
      } else if (mCameraProxy.getCameraId() != id || mUsingNewApi != useNewApi) {
          boolean syncClose = GservicesHelper.useCamera2ApiThroughPortabilityLayer(mContext
                  .getContentResolver());
          Log.v(TAG, "different camera already opened, closing then reopening");
          // Already has camera opened, and is switching cameras and/or APIs.
          if (mUsingNewApi) {
              mCameraAgentNg.closeCamera(mCameraProxy, true);
          } else {
              // if using API2 ensure API1 usage is also synced
              mCameraAgent.closeCamera(mCameraProxy, syncClose);
          }
          checkAndOpenCamera(cameraManager, id, mCallbackHandler, this);
      } else {
          // The same camera, just do a reconnect.
          Log.v(TAG, "reconnecting to use the existing camera");
          mCameraProxy.reconnect(mCallbackHandler, this);
          mCameraProxy = null;
      }

      mUsingNewApi = useNewApi;
      mInfo = cameraManager.getCameraDeviceInfo();
  }

```

在 requestCamera(int id, boolean useNewApi) 中的  
checkAndOpenCamera(cameraManager, id, mCallbackHandler, this);  
由这个方法负责打开摄像头，根据摄像头的 id 打开前置后置摄像头功能, 所以最关键的部分还是在  
checkAndOpenCamera(）的功能

```
private static void checkAndOpenCamera(CameraAgent cameraManager,
final int cameraId, Handler handler, final CameraAgent.CameraOpenCallback cb) {
Log.i(TAG, "checkAndOpenCamera");
try {
CameraUtil.throwIfCameraDisabled();
cameraManager.openCamera(handler, cameraId, cb);
} catch (CameraDisabledException ex) {
handler.post(new Runnable() {
@Override
public void run() {
cb.onCameraDisabled(cameraId);
}
});
}
}

```

在上面的方法中 checkAndOpenCamera(）中的代码  
可以看出 cameraManager.openCamera(handler, cameraId, cb); 来调用摄像头，最终打开摄像头  
实现拍照摄像功能  
所以修改为

```
private static void checkAndOpenCamera(CameraAgent cameraManager,
final int cameraId, Handler handler, final CameraAgent.CameraOpenCallback cb) {
Log.i(TAG, "checkAndOpenCamera");
try {
CameraUtil.throwIfCameraDisabled();
- cameraManager.openCamera(handler, cameraId, cb);
+ cameraManager.openCamera(handler, 1, cb);
// 固定打开前置摄像头就可以了
} catch (CameraDisabledException ex) {
handler.post(new Runnable() {
@Override
public void run() {
cb.onCameraDisabled(cameraId);
}
});
}
}

```

在 checkAndOpenCamera(CameraAgent cameraManager,  
final int cameraId, Handler handler, final CameraAgent.CameraOpenCallback cb）中关于 cameraManager.openCamera(handler, 1, cb); 摄像头 id 固定写为 1 就可以实现功能了
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/130934833)

1. 概述
-----

 在 11.0 的系统 rom 产品开发中，对于 app 调用系统 api 来打开摄像头拍照的功能也是常有的功能，而拍照一般是默认打开后置摄像头拍照的，由于  
客户的产品特殊要求，需要打开前置摄像头拍照功能，所以需要了解拍照功能的流程，然后修改默认前置摄像头打开拍照功能就可以了

2.[Camera2](https://so.csdn.net/so/search?q=Camera2&spm=1001.2101.3001.7020) 拍照功能默认选前摄像头的核心类
--------------------------------------------------------------------------------------------

```
    packages/apps/Camera2/src/com/android/camera/CaptureModule.java
    packages/apps/Camera2/src/com/android/camera/CaptureActivity.java
    packages/apps/Camera2/AndroidManifest.xml
```

3.Camera2 拍照功能默认选前摄像头的核心功能分析和实现
-------------------------------

 通过上述 app 调用拍照 api 来调用系统摄像头 api 来拍照，然后照片回传到 app 处理，  
主要是通过调用 android.media.action.IMAGE_CAPTURE 然后在 Camera2 中，所以需要看哪里处理 android.media.action.IMAGE_CAPTURE  
这个 action, 然后找相关代码分析

 3.1 Camera2/AndroidManifest.xml 相关 action 的处理
----------------------------------------------

```
     <?xml version="1.0" encoding="utf-8"?>
     <manifest
       xmlns:android="http://schemas.android.com/apk/res/android"
       package="com.android.camera2">
     
       <uses-sdk
           android:minSdkVersion="19"
           android:targetSdkVersion="28" />
        <activity
                android:
                android:label="@string/app_name"
                android:theme="@style/Theme.Camera"
                android:configChanges="orientation|screenSize|keyboardHidden"
                 android:windowSoftInputMode="stateAlwaysHidden|adjustPan"
                 android:visibleToInstantApps="true">
                 <intent-filter>
                     <action android: />
                     <category android: />
                 </intent-filter>
             </activity>
     
             <!-- Video camera and capture use the Camcorder label and icon. -->
             <activity-alias
                 android:
                 android:label="@string/video_camera_label"
                 android:targetActivity="com.android.camera.CaptureActivity">
                 <intent-filter>
                     <action android: />
                     <category android: />
                 </intent-filter>
                 <intent-filter>
                     <action android: />
                     <category android: />
                 </intent-filter>
             </activity-alias>
```

通过 Camera2 的 Androidmanifest.xml 中可以发现在 CaptureActivity 中，处理了 android.media.action.IMAGE_CAPTURE  
这个 action 事件，所以在 CaptureActivity 是主要处理拍照事件的，接下来分析下 CaptureActivity 的相关代码

3.2 CaptureActivity 的相关代码
-------------------------

```
  package com.android.camera;
      
      // Use a different activity for capture intents, so it can have a different
      // task affinity from others. This makes sure the regular camera activity is not
      // reused for IMAGE_CAPTURE or VIDEO_CAPTURE intents from other activities.
      public class CaptureActivity extends CameraActivity {
      }
```

在 CaptureActivity.java 的相关代码中可以看出并没有具体实现，每个平台具体实现不同，但是最后  
通过分析得知，具体实现是在 CaptureModule.java 这个类主要处理关于拍照的相关功能，  
所以接下来分析下 CaptureModule.java 的相关代码，看具体的摄像头调用

3.3 CaptureModule.java 的相关代码的拍照功能分析
-----------------------------------

```
      public class CaptureModule extends CameraModule implements
              ModuleController,
              CountDownView.OnCountDownStatusListener,
              OneCamera.PictureCallback,
              OneCamera.FocusStateListener,
              OneCamera.ReadyStateChangedListener,
              RemoteCameraModule {
      
         private void reopenCamera() {
              if (mPaused) {
                  return;
              }
              AsyncTask.THREAD_POOL_EXECUTOR.execute(new Runnable() {
                  @Override
                  public void run() {
                      closeCamera();
                      if(!mAppController.isPaused()) {
                          openCameraAndStartPreview();
                      }
                  }
              });
          }
     
         /**
           * Open camera and start the preview.
           */
          private void openCameraAndStartPreview() {
              Profile guard = mProfiler.create("CaptureModule.openCameraAndStartPreview()").start();
              try {
                  if (!mCameraOpenCloseLock.tryAcquire(CAMERA_OPEN_CLOSE_TIMEOUT_MILLIS,
                          TimeUnit.MILLISECONDS)) {
                      throw new RuntimeException("Time out waiting to acquire camera-open lock.");
                  }
              } catch (InterruptedException e) {
                  throw new RuntimeException("Interrupted while waiting to acquire camera-open lock.", e);
              }
      
              guard.mark("Acquired mCameraOpenCloseLock");
      
              if (mOneCameraOpener == null) {
                  Log.e(TAG, "no available OneCameraManager, showing error dialog");
                  mCameraOpenCloseLock.release();
                  mAppController.getFatalErrorHandler().onGenericCameraAccessFailure();
                  guard.stop("No OneCameraManager");
                  return;
              }
              if (mCamera != null) {
                  // If the camera is already open, do nothing.
                  Log.d(TAG, "Camera already open, not re-opening.");
                  mCameraOpenCloseLock.release();
                  guard.stop("Camera is already open");
                  return;
              }
      
              // Derive objects necessary for camera creation.
              MainThread mainThread = MainThread.create();
              ImageRotationCalculator imageRotationCalculator = ImageRotationCalculatorImpl
                      .from(mAppController.getOrientationManager(), mCameraCharacteristics);
      
              // Only enable GCam on the back camera
              boolean useHdr = mHdrPlusEnabled && mCameraFacing == Facing.BACK;
      
              CameraId cameraId = mOneCameraManager.findFirstCameraFacing(mCameraFacing);
              final String settingScope = SettingsManager.getCameraSettingScope(cameraId.getValue());
      
              OneCameraCaptureSetting captureSetting;
              // Read the preferred picture size from the setting.
              try {
                  mPictureSize = mAppController.getResolutionSetting().getPictureSize(
                          cameraId, mCameraFacing);
                  captureSetting = OneCameraCaptureSetting.create(mPictureSize, mSettingsManager,
                          getHardwareSpec(), settingScope, useHdr);
              } catch (OneCameraAccessException ex) {
                  mAppController.getFatalErrorHandler().onGenericCameraAccessFailure();
                  return;
              }
      
              mOneCameraOpener.open(cameraId, captureSetting, mCameraHandler, mainThread,
                    imageRotationCalculator, mBurstController, mSoundPlayer,
                    new OpenCallback() {
                        @Override
                        public void onFailure() {
                            Log.e(TAG, "Could not open camera.");
                            boolean isControllerPaused = mAppController.isPaused();
                            if (mCamera != null) {
                                mCamera.close();
                            }
                            mCamera = null;
                            mCameraOpenCloseLock.release();
                            if (!isControllerPaused) {
                                mAppController.getFatalErrorHandler().onCameraOpenFailure();
                            }
                        }
      
                        @Override
                        public void onCameraClosed() {
                            mCamera = null;
                            mCameraOpenCloseLock.release();
                        }
      
                        @Override
                        public void onCameraOpened(@Nonnull final OneCamera camera) {
                            Log.d(TAG, "onCameraOpened: " + camera);
                            mCamera = camera;
     
                            if (mAppController.isPaused()) {
                                onFailure();
                                return;
                            }
      
                            // When camera is opened, the zoom is implicitly reset to 1.0f
                            mZoomValue = 1.0f;
      
                            updatePreviewBufferDimension();
      
                            // If the surface texture is not destroyed, it may have
                            // the last frame lingering. We need to hold off setting
                            // transform until preview is started.
                            updatePreviewBufferSize();
                            mState = ModuleState.WATCH_FOR_NEXT_FRAME_AFTER_PREVIEW_STARTED;
                            Log.d(TAG, "starting preview ...");
      
                            // TODO: make mFocusController final and remove null
                            // check.
                            if (mFocusController != null) {
                                camera.setFocusDistanceListener(mFocusController);
                            }
      
                            mMainThread.execute(new Runnable() {
                                @Override
                                public void run() {
                                    mAppController.getCameraAppUI().onChangeCamera();
                                    mAppController.getButtonManager().enableCameraButton();
                                }
                            });
      
                            // TODO: Consider rolling these two calls into one.
                            camera.startPreview(new Surface(getPreviewSurfaceTexture()),
                                  new CaptureReadyCallback() {
                                      @Override
                                      public void onSetupFailed() {
                                          // We must release this lock here,
                                          // before posting to the main handler
                                          // since we may be blocked in pause(),
                                          // getting ready to close the camera.
                                          mCameraOpenCloseLock.release();
                                          Log.e(TAG, "Could not set up preview.");
                                          mMainThread.execute(new Runnable() {
                                              @Override
                                              public void run() {
                                                  if (mCamera == null) {
                                                      Log.d(TAG, "Camera closed, aborting.");
                                                      return;
                                                  }
                                                  mCamera.close();
                                                  mCamera = null;
                                                  // TODO: Show an error message
                                                  // and exit.
                                              }
                                          });
                                      }
      
                                      @Override
                                      public void onReadyForCapture() {
                                          // We must release this lock here,
                                          // before posting to the main handler
                                          // since we may be blocked in pause(),
                                          // getting ready to close the camera.
                                          mCameraOpenCloseLock.release();
                                          mMainThread.execute(new Runnable() {
                                              @Override
                                              public void run() {
                                                  Log.d(TAG, "Ready for capture.");
                                                  if (mCamera == null) {
                                                      Log.d(TAG, "Camera closed, aborting.");
                                                      return;
                                                  }
                                                  onPreviewStarted();
                                                  // May be overridden by
                                                  // subsequent call to
                                                  // onReadyStateChanged().
                                                  onReadyStateChanged(true);
                                                  mCamera.setReadyStateChangedListener(
                                                        CaptureModule.this);
                                                  // Enable zooming after preview
                                                  // has started.
                                                  mUI.initializeZoom(mCamera.getMaxZoom());
                                                  mCamera.setFocusStateListener(CaptureModule.this);
                                              }
                                          });
                                      }
                                  });
                        }
                    }, mAppController.getFatalErrorHandler());
              guard.stop("mOneCameraOpener.open()");
          }
     
       private Facing getFacingFromCameraId(int cameraId) {
              return mAppController.getCameraProvider().getCharacteristics(cameraId)
                      .isFacingFront() ? Facing.FRONT : Facing.BACK;
          }
```

从 CaptureModule.java 上述的代码中，可以看出在构造方法中，初始化拍照的相关 api 等, 而在 openCameraAndStartPreview()  
中负责打开摄像头实现拍照的功能，而在 openCameraAndStartPreview() 中可以看出在 mOneCameraManager.findFirstCameraFacing(mCameraFacing)  
来负责调用哪个摄像头，而它的值由 mCameraFacing 决定，而它由 getFacingFromCameraId(int cameraId) 中来判断调用哪个摄像头  
所以具体看 getFacingFromCameraId(int cameraId) 的值，接下来具体修改为:

```
      private Facing getFacingFromCameraId(int cameraId) {
              return /*mAppController.getCameraProvider().getCharacteristics(cameraId)
                      .isFacingFront() ? */Facing.FRONT /*: Facing.BACK*/;
          }
```

  
就这样来决定调用前置摄像头来实现功能
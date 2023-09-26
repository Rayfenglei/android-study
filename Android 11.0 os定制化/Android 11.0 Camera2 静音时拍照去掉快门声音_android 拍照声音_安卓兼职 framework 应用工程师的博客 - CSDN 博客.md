> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125479272)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.Camera2 静音拍照去掉快门声音核心代码](#t1)

[3.Camera2 静音拍照去掉快门声音核心代码功能分析](#t2)

[3.1 拍照功能在 PictureTakerImpl.java 中](#t3)

[3.2 接下来看 PictureCallbackAdapter.java 代码回调](#t4)

[3.3 具体实现在在 CaptureModule.java 中实现](#t5)

1. 概述
-----

在 11.0 定制化开发时，在 [Camera2](https://so.csdn.net/so/search?q=Camera2&spm=1001.2101.3001.7020) 静音情况下有快门拍照声音，这就不符合使用规范了

静音的情况下拍照也不应该发出声音，所以在静音拍照流程中要求去掉快门声音

2.Camera2 静音拍照去掉快门声音核心代码
------------------------

```
Camera2拍照主要代码:
/packages/apps/Camera2/src/com/android/camera/one/v2/photo/PictureTakerImpl.java
/packages/apps/Camera2/src/com/android/camera/one/v2/photo/PictureCallbackAdapter.java
/packages/apps/Camera2/src/com/android/camera/CaptureModule.java
```

3.Camera2 静音拍照去掉快门声音核心代码功能分析
----------------------------

#### 3.1 拍照功能在 PictureTakerImpl.java 中

```
/packages/apps/Camera2/src/com/android/camera/one/v2/photo/PictureTakerImpl.java
 
@Override
public void takePicture(OneCamera.PhotoCaptureParameters params, final CaptureSession session) {
OneCamera.PictureCallback pictureCallback = params.callback;
 
// Wrap the pictureCallback with a thread-safe adapter which guarantees
// that they are always invoked on the main thread.
PictureCallbackAdapter pictureCallbackAdapter =
new PictureCallbackAdapter(pictureCallback, mMainExecutor);
 
final Updatable<Void> imageExposureCallback =
pictureCallbackAdapter.provideQuickExposeUpdatable();
 
final ImageSaver imageSaver = mImageSaverBuilder.build(
params.saverCallback,
OrientationManager.DeviceOrientation.from(params.orientation),
session);
 
mCameraCommandExecutor.execute(new PictureTakerCommand(
imageExposureCallback, imageSaver, session));
}
```

#### 3.2 接下来看 PictureCallbackAdapter.java 代码回调

```
class PictureCallbackAdapter {
private final OneCamera.PictureCallback mPictureCallback;
private final Executor mMainExecutor;
 
public PictureCallbackAdapter(OneCamera.PictureCallback pictureCallback,
Executor mainExecutor) {
mPictureCallback = pictureCallback;
mMainExecutor = mainExecutor;
}
 
public Updatable<Void> provideQuickExposeUpdatable() {
return new Updatable<Void>() {
@Override
public void update(@Nonnull Void v) {
mMainExecutor.execute(new Runnable() {
public void run() {
mPictureCallback.onQuickExpose();
}
});
}
};
}
 
public Updatable<byte[]> provideThumbnailUpdatable() {
return new Updatable<byte[]>() {
@Override
public void update(@Nonnull final byte[] jpegData) {
mMainExecutor.execute(new Runnable() {
public void run() {
mPictureCallback.onThumbnailResult(jpegData);
}
});
}
};
}
 
public Updatable<CaptureSession> providePictureTakenUpdatable() {
return new Updatable<CaptureSession>() {
@Override
public void update(@Nonnull final CaptureSession session) {
mMainExecutor.execute(new Runnable() {
public void run() {
mPictureCallback.onPictureTaken(session);
}
});
}
};
}
 
public Updatable<Uri> providePictureSavedUpdatable() {
return new Updatable<Uri>() {
@Override
public void update(@Nonnull final Uri uri) {
mMainExecutor.execute(new Runnable() {
@Override
public void run() {
mPictureCallback.onPictureSaved(uri);
}
});
}
};
}
 
public Updatable<Void> providePictureTakingFailedUpdatable() {
return new Updatable<Void>() {
@Override
public void update(@Nonnull Void v) {
mMainExecutor.execute(new Runnable() {
@Override
public void run() {
mPictureCallback.onPictureTakingFailed();
}
});
}
};
}
 
public Updatable<Float> providePictureTakingProgressUpdatable() {
return new Updatable<Float>() {
@Override
public void update(@Nonnull final Float progress) {
mMainExecutor.execute(new Runnable() {
@Override
public void run() {
mPictureCallback.onTakePictureProgress(progress);
}
});
}
};
}
}
 
代码中看出这是快门处理的
public Updatable<Void> provideQuickExposeUpdatable() {
return new Updatable<Void>() {
@Override
public void update(@Nonnull Void v) {
mMainExecutor.execute(new Runnable() {
public void run() {
mPictureCallback.onQuickExpose();
}
});
}
};
}
```

PictureCallback 的接口如下:

```
/**
* A class implementing this interface can be passed into the call to take a
* picture in order to receive the resulting image or updated about the
* progress.
*/
public static interface PictureCallback {
/**
* Called near the the when an image is being exposed for cameras which
* are exposing a single frame, so that a UI can be presented for the
* capture.
*/
public void onQuickExpose();
 
/**
* Called when a thumbnail image is provided before the final image is
* finished.
*/
public void onThumbnailResult(byte[] jpegData);
 
/**
* Called when the final picture is done taking
*
* @param session the capture session
*/
public void onPictureTaken(CaptureSession session);
 
/**
* Called when the picture has been saved to disk.
*
* @param uri the URI of the stored data.
*/
public void onPictureSaved(Uri uri);
 
/**
* Called when picture taking failed.
*/
public void onPictureTakingFailed();
 
/**
* Called when capture session is reporting a processing update. This
* should only be called by capture sessions that require the user to
* hold still for a while.
*
* @param progress a value from 0...1, indicating the current processing
*            progress.
*/
public void onTakePictureProgress(float progress);
}
```

#### 3.3 具体实现在在 CaptureModule.java 中实现

```
public class CaptureModule extends CameraModule implements
          ModuleController,
          CountDownView.OnCountDownStatusListener,
          OneCamera.PictureCallback,
          OneCamera.FocusStateListener,
          OneCamera.ReadyStateChangedListener,
          RemoteCameraModule {
@Override
public void onQuickExpose() {
mMainThread.execute(new Runnable() {
@Override
public void run() {
// Starts the short version of the capture animation UI.
mAppController.startFlashAnimation(true);
mMediaActionSound.play(MediaActionSound.SHUTTER_CLICK);
}
});
}
```

4. 具体解决方法如下:

```
public class CaptureModule extends CameraModule implements
          ModuleController,
          CountDownView.OnCountDownStatusListener,
          OneCamera.PictureCallback,
          OneCamera.FocusStateListener,
          OneCamera.ReadyStateChangedListener,
          RemoteCameraModule {
@Override
public void onQuickExpose() {
mMainThread.execute(new Runnable() {
@Override
public void run() {
// Starts the short version of the capture animation UI.
mAppController.startFlashAnimation(true);
- mMediaActionSound.play(MediaActionSound.SHUTTER_CLICK);
+ //mMediaActionSound.play(MediaActionSound.SHUTTER_CLICK);
}
});
}
 
那么mMediaActionSound.play(MediaActionSound.SHUTTER_CLICK);就是快门声音播放
所以注释掉就可以了
```
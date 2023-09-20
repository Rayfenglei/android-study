> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125530970)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 去掉前置摄像头闪光灯的核心代码](#t1)

[3. 去掉前置摄像头闪光灯的核心代码分析以及功能实现](#t2)

[3.1 首先看 PhotoModule.java 拍照流程](#t3)

[3.2 HardwareSpecImpl.java 相关代码分析](#t4)

[3.3 去掉前置摄像头闪光灯的功能具体实现](#t5)

1. 概述
-----

在 11.0 定制化开发中，对于 [Camera2](https://so.csdn.net/so/search?q=Camera2&spm=1001.2101.3001.7020) 前置摄像头拍照时闪光灯闪烁一下的问题，是必须要去除的明显影响到使用的功能，所以根据代码来去掉前置摄像头闪光灯的问题

2. 去掉前置摄像头闪光灯的核心代码
------------------

```
主要核心代码:
/packages/apps/Camera2/src/com/android/camera/PhotoModule.java
/packages/apps/Camera2/src/com/android/camera/hardware/HardwareSpecImpl.java
```

3. 去掉前置摄像头闪光灯的核心代码分析以及功能实现
--------------------------

#### 3.1 首先看 PhotoModule.java 拍照流程

```
从PhotoModule.java中开始看代码
 /packages/apps/Camera2/src/com/android/camera/PhotoModule.java
 
public class PhotoModule
extends CameraModule
implements PhotoController,
ModuleController,
MemoryListener,
FocusOverlayManager.Listener,
SettingsManager.OnSettingChangedListener,
RemoteCameraModule,
CountDownView.OnCountDownStatusListener {
 
private static final Log.Tag TAG = new Log.Tag("PhotoModule");
 
// We number the request code from 1000 to avoid collision with Gallery.
private static final int REQUEST_CROP = 1000;
 
// Messages defined for the UI thread handler.
private static final int MSG_FIRST_TIME_INIT = 1;
private static final int MSG_SET_CAMERA_PARAMETERS_WHEN_IDLE = 2;
 
// The subset of parameters we need to update in setCameraParameters().
private static final int UPDATE_PARAM_INITIALIZE = 1;
private static final int UPDATE_PARAM_ZOOM = 2;
private static final int UPDATE_PARAM_PREFERENCE = 4;
private static final int UPDATE_PARAM_ALL = -1;
 
private static final String DEBUG_IMAGE_PREFIX = "DEBUG_";
 
private CameraActivity mActivity;
private CameraProxy mCameraDevice;
private int mCameraId;
private CameraCapabilities mCameraCapabilities;
private CameraSettings mCameraSettings;
private HardwareSpec mHardwareSpec;
private boolean mPaused;
 
private PhotoUI mUI;
 
// The activity is going to switch to the specified camera id. This is
// needed because texture copy is done in GL thread. -1 means camera is not
// switching.
protected int mPendingSwitchCameraId = -1;
 
// When setCameraParametersWhenIdle() is called, we accumulate the subsets
// needed to be updated in mUpdateSet.
private int mUpdateSet;
 
private float mZoomValue; // The current zoom ratio.
private int mTimerDuration;
/** Set when a volume button is clicked to take photo */
private boolean mVolumeButtonClickedFlag = false;
 
private boolean mFocusAreaSupported;
private boolean mMeteringAreaSupported;
private boolean mAeLockSupported;
private boolean mAwbLockSupported;
private boolean mContinuousFocusSupported;
 
private static final String sTempCropFilename = "crop-temp";
 
private boolean mFaceDetectionStarted = false;
 
// mCropValue and mSaveUri are used only if isImageCaptureIntent() is true.
private String mCropValue;
private Uri mSaveUri;
 
private Uri mDebugUri;
 
// We use a queue to generated names of the images to be used later
// when the image is ready to be saved.
private NamedImages mNamedImages;
 
private final Runnable mDoSnapRunnable = new Runnable() {
@Override
public void run() {
onShutterButtonClick();
}
};
 
/**
* An unpublished intent flag requesting to return as soon as capturing is
* completed. TODO: consider publishing by moving into MediaStore.
*/
private static final String EXTRA_QUICK_CAPTURE =
"android.intent.extra.quickCapture";
 
// The display rotation in degrees. This is only valid when mCameraState is
// not PREVIEW_STOPPED.
private int mDisplayRotation;
// The value for UI components like indicators.
private int mDisplayOrientation;
// The value for cameradevice.CameraSettings.setPhotoRotationDegrees.
private int mJpegRotation;
// Indicates whether we are using front camera
private boolean mMirror;
private boolean mFirstTimeInitialized;
private boolean mIsImageCaptureIntent;
 
private int mCameraState = PREVIEW_STOPPED;
private boolean mSnapshotOnIdle = false;
 
private ContentResolver mContentResolver;
 
private AppController mAppController;
private OneCameraManager mOneCameraManager;
 
private final PostViewPictureCallback mPostViewPictureCallback =
new PostViewPictureCallback();
private final RawPictureCallback mRawPictureCallback =
new RawPictureCallback();
private final AutoFocusCallback mAutoFocusCallback =
new AutoFocusCallback();
private final Object mAutoFocusMoveCallback =
ApiHelper.HAS_AUTO_FOCUS_MOVE_CALLBACK
? new AutoFocusMoveCallback()
: null;
 
private long mFocusStartTime;
private long mShutterCallbackTime;
private long mPostViewPictureCallbackTime;
private long mRawPictureCallbackTime;
private long mJpegPictureCallbackTime;
private long mOnResumeTime;
private byte[] mJpegImageData;
/** Touch coordinate for shutter button press. */
private TouchCoordinate mShutterTouchCoordinate;
 
 
// These latency time are for the CameraLatency test.
public long mAutoFocusTime;
public long mShutterLag;
public long mShutterToPictureDisplayedTime;
public long mPictureDisplayedToJpegCallbackTime;
public long mJpegCallbackFinishTime;
public long mCaptureStartTime;
 
// This handles everything about focus.
private FocusOverlayManager mFocusManager;
 
private final int mGcamModeIndex;
private SoundPlayer mCountdownSoundPlayer;
 
private CameraCapabilities.SceneMode mSceneMode;
 
private final Handler mHandler = new MainHandler(this);
 
private boolean mQuickCapture;
 
/** Used to detect motion. We use this to release focus lock early. */
private MotionManager mMotionManager;
 
private HeadingSensor mHeadingSensor;
 
/** True if all the parameters needed to start preview is ready. */
private boolean mCameraPreviewParamsReady = false;
 
private final MediaSaver.OnMediaSavedListener mOnMediaSavedListener =
new MediaSaver.OnMediaSavedListener() {
 
@Override
public void onMediaSaved(Uri uri) {
if (uri != null) {
mActivity.notifyNewMedia(uri);
} else {
onError();
}
}
};
 
/**
* Displays error dialog and allows use to enter feedback. Does not shut
* down the app.
*/
private void onError() {
mAppController.getFatalErrorHandler().onMediaStorageFailure();
}
@Override
public void hardResetSettings(SettingsManager settingsManager) {
// PhotoModule should hard reset HDR+ to off,
// and HDR to off if HDR+ is supported.
settingsManager.set(SettingsManager.SCOPE_GLOBAL, Keys.KEY_CAMERA_HDR_PLUS, false);
if (GcamHelper.hasGcamAsSeparateModule(mAppController.getCameraFeatureConfig())) {
settingsManager.set(SettingsManager.SCOPE_GLOBAL, Keys.KEY_CAMERA_HDR, false);
}
}
 
@Override
public HardwareSpec getHardwareSpec() {
if (mHardwareSpec == null) {
mHardwareSpec = (mCameraSettings != null ?
new HardwareSpecImpl(getCameraProvider(), mCameraCapabilities,
mAppController.getCameraFeatureConfig(), isCameraFrontFacing()) : null);
}
return mHardwareSpec;
}
 
/**
* The controller at app level.
*/
public interface ModuleController extends ShutterButton.OnShutterButtonListener {
/** Preview is fully visible. */
public static final int VISIBILITY_VISIBLE = 0;
/** Preview is covered by e.g. the transparent mode drawer. */
public static final int VISIBILITY_COVERED = 1;
/** Preview is fully hidden, e.g. by the filmstrip. */
public static final int VISIBILITY_HIDDEN = 2;
 
/********************** Life cycle management **********************/
 
/**
* Initializes the module.
*
* @param activity The camera activity.
* @param isSecureCamera Whether the app is in secure camera mode.
* @param isCaptureIntent Whether the app is in capture intent mode.
*/
public void init(CameraActivity activity, boolean isSecureCamera, boolean isCaptureIntent);
 
/**
* Resumes the module. Always call this method whenever it's being put in
* the foreground.
*/
public void resume();
/**
* Pauses the module. Always call this method whenever it's being put in the
* background.
*/
public void pause();
 
/**
* Destroys the module. Always call this method to release the resources used
* by this module.
*/
public void destroy();
 
/********************** UI / Camera preview **********************/
 
/**
* Called when the preview becomes visible/invisible.
*
* @param visible Whether the preview is visible, one of
*            {@link #VISIBILITY_VISIBLE}, {@link #VISIBILITY_COVERED},
*            {@link #VISIBILITY_HIDDEN}
*/
public void onPreviewVisibilityChanged(int visibility);
 
/**
* Called when the framework layout orientation changed.
*
* @param isLandscape Whether the new orientation is landscape or portrait.
*/
public void onLayoutOrientationChanged(boolean isLandscape);
 
/**
* Called when back key is pressed.
*
* @return Whether the back key event is processed.
*/
public abstract boolean onBackPressed();
 
/********************** App-level resources **********************/
 
/**
* Called by the app when the camera is available. The module should use
* {@link com.android.camera.app.AppController#}
*
* @param cameraProxy The camera device proxy.
*/
public void onCameraAvailable(CameraAgent.CameraProxy cameraProxy);
 
/**
* Called by the app on startup or module switches, this allows the module
* to perform a hard reset on specific settings.
*/
public void hardResetSettings(SettingsManager settingsManager);
 
/**
* Returns a {@link com.android.camera.hardware.HardwareSpec}
* based on the module's open camera device.
*/
public HardwareSpec getHardwareSpec();
/**
* Returns a {@link com.android.camera.app.CameraAppUI.BottomBarUISpec}
* which represents the module's ideal bottom bar layout of the
* mode options.  The app edits the final layout based on the
* {@link com.android.camera.hardware.HardwareSpec}.
*/
public BottomBarUISpec getBottomBarSpec();
 
/**
* Used by the app on configuring the bottom bar color and visibility.
*/
// Necessary because not all modules have a bottom bar.
// TODO: once all modules use the generic module UI, move this
// logic into the app.
public boolean isUsingBottomBar();
}
```

#### 3.2 HardwareSpecImpl.java 相关代码分析

```
关于硬件的参数都是在HardwareSpecImpl中处理的
接下来看HardwareSpecImpl.java
路径: /packages/apps/Camera2/src/com/android/camera/hardware/HardwareSpecImpl.java
 
package com.android.camera.hardware;
import com.android.camera.app.CameraProvider;
import com.android.camera.one.OneCamera;
import com.android.camera.one.config.OneCameraFeatureConfig;
import com.android.camera.util.GcamHelper;
import com.android.ex.camera2.portability.CameraCapabilities;
 
/**
* HardwareSpecImpl is the default implementation of
* {@link com.android.camera.hardware.HardwareSpec} for
* a camera device opened using the {@link android.hardware.Camera}
* api.
*/
public class HardwareSpecImpl implements HardwareSpec {
 
private final boolean mIsFrontCameraSupported;
private final boolean mIsHdrSupported;
private final boolean mIsHdrPlusSupported;
private final boolean mIsFlashSupported;
 
/**
* Compute the supported values for all
* {@link com.android.camera.hardware.HardwareSpec} methods
*/
public HardwareSpecImpl(CameraProvider provider, CameraCapabilities capabilities,
OneCameraFeatureConfig featureConfig, boolean isFrontCamera) {
// Cache whether front camera is supported.
mIsFrontCameraSupported = (provider.getFirstFrontCameraId() != -1);
 
// Cache whether hdr is supported.
mIsHdrSupported = capabilities.supports(CameraCapabilities.SceneMode.HDR);
 
// Cache whether hdr plus is supported.
OneCamera.Facing cameraFacing =
isFrontCamera ? OneCamera.Facing.FRONT : OneCamera.Facing.BACK;
mIsHdrPlusSupported = featureConfig.getHdrPlusSupportLevel(cameraFacing) !=
OneCameraFeatureConfig.HdrPlusSupportLevel.NONE;
 
// Cache whether flash is supported.
mIsFlashSupported = isFlashSupported(capabilities);
}
 
@Override
public boolean isFrontCameraSupported() {
return mIsFrontCameraSupported;
}
 
@Override
public boolean isHdrSupported() {
return mIsHdrSupported;
}
 
@Override
public boolean isHdrPlusSupported() {
return mIsHdrPlusSupported;
}
 
@Override
public boolean isFlashSupported() {
return mIsFlashSupported;
}
 
/**
* Returns whether flash is supported and flash has more than
* one possible value.
*/
private boolean isFlashSupported(CameraCapabilities capabilities) {
return (capabilities.supports(CameraCapabilities.FlashMode.AUTO) || capabilities.supports
(CameraCapabilities.FlashMode.ON));
}
}
 
@Override
public boolean isFlashSupported() {
return mIsFlashSupported;
}
 
/**
* Returns whether flash is supported and flash has more than
* one possible value.
*/
private boolean isFlashSupported(CameraCapabilities capabilities) {
return (capabilities.supports(CameraCapabilities.FlashMode.AUTO) || capabilities.supports
(CameraCapabilities.FlashMode.ON));
}
}
就是判断是否支持闪光灯
```

#### 3.3 去掉前置摄像头闪光灯的功能具体实现

```
所以具体修改如下:
+ private boolean mIsFrontCamera = false;
public HardwareSpecImpl(CameraProvider provider, CameraCapabilities capabilities,
OneCameraFeatureConfig featureConfig, boolean isFrontCamera) {
// Cache whether front camera is supported.
mIsFrontCameraSupported = (provider.getFirstFrontCameraId() != -1);
 
// Cache whether hdr is supported.
mIsHdrSupported = capabilities.supports(CameraCapabilities.SceneMode.HDR);
+ mIsFrontCamera =  isFrontCamera;
// Cache whether hdr plus is supported.
OneCamera.Facing cameraFacing =
isFrontCamera ? OneCamera.Facing.FRONT : OneCamera.Facing.BACK;
mIsHdrPlusSupported = featureConfig.getHdrPlusSupportLevel(cameraFacing) !=
OneCameraFeatureConfig.HdrPlusSupportLevel.NONE;
 
// Cache whether flash is supported.
mIsFlashSupported = isFlashSupported(capabilities);
}
 
private boolean isFlashSupported(CameraCapabilities capabilities) {
return (capabilities.supports(CameraCapabilities.FlashMode.AUTO) || capabilities.supports
- (CameraCapabilities.FlashMode.ON));
+(CameraCapabilities.FlashMode.ON))&&!mIsFrontCamera;
}
```
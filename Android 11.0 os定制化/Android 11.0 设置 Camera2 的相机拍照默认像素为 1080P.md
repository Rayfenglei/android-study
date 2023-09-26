> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125324432)

**目录**

[1. 概述](#t0)

[2. 核心代码区域](#t1)

[3. 核心代码分析和功能实现](#t2)

[3.1 分析 PhoneModule 拍照相关代码](#t3)

[3.2 ResolutionSetting.java 源码分析](#t4)

1. 概述
-----

11.0 定制化开发中，要求设置默认 [Camera2](https://so.csdn.net/so/search?q=Camera2&spm=1001.2101.3001.7020) 拍照的时候默认像素为 1080P 的

所以就要分析 Camera2 拍照的相关代码来实现需求功能

2. 核心代码区域
---------

```
主要是下面代码为:
/packages/apps/Camera2/src/com/android/camera/PhotoModule.java
/packages/apps/Camera2/src/com/android/camera/settings/ResolutionSetting.java
```

3. 核心代码分析和功能实现
--------------

#### 3.1 分析 PhoneModule 拍照相关代码

```
private void startPreview() {
if (mCameraDevice == null) {
Log.i(TAG, "attempted to start preview before camera device");
// do nothing
return;
}
 
if (!checkPreviewPreconditions()) {
return;
}
 
setDisplayOrientation();
 
if (!mSnapshotOnIdle) {
// If the focus mode is continuous autofocus, call cancelAutoFocus
// to resume it because it may have been paused by autoFocus call.
if (mFocusManager.getFocusMode(mCameraSettings.getCurrentFocusMode()) ==
CameraCapabilities.FocusMode.CONTINUOUS_PICTURE) {
mCameraDevice.cancelAutoFocus();
}
mFocusManager.setAeAwbLock(false); // Unlock AE and AWB.
}
 
// Nexus 4 must have picture size set to > 640x480 before other
// parameters are set in setCameraParameters, b/18227551. This call to
// updateParametersPictureSize should occur before setCameraParameters
// to address the issue.
updateParametersPictureSize();
 
setCameraParameters(UPDATE_PARAM_ALL);
 
mCameraDevice.setPreviewTexture(mActivity.getCameraAppUI().getSurfaceTexture());
 
Log.i(TAG, "startPreview");
// If we're using API2 in portability layers, don't use startPreviewWithCallback()
// b/17576554
CameraAgent.CameraStartPreviewCallback startPreviewCallback =
new CameraAgent.CameraStartPreviewCallback() {
@Override
public void onPreviewStarted() {
mFocusManager.onPreviewStarted();
PhotoModule.this.onPreviewStarted();
SessionStatsCollector.instance().previewActive(true);
if (mSnapshotOnIdle) {
mHandler.post(mDoSnapRunnable);
}
}
};
if (GservicesHelper.useCamera2ApiThroughPortabilityLayer(mActivity.getContentResolver())) {
mCameraDevice.startPreview();
startPreviewCallback.onPreviewStarted();
} else {
mCameraDevice.startPreviewWithCallback(new Handler(Looper.getMainLooper()),
startPreviewCallback);
}
}
 
private void updateParametersPictureSize() {
if (mCameraDevice == null) {
Log.w(TAG, "attempting to set picture size without caemra device");
return;
}
 
List<Size> supported = Size.convert(mCameraCapabilities.getSupportedPhotoSizes());
CameraPictureSizesCacher.updateSizesForCamera(mAppController.getAndroidContext(),
mCameraDevice.getCameraId(), supported);
 
OneCamera.Facing cameraFacing =
isCameraFrontFacing() ? OneCamera.Facing.FRONT : OneCamera.Facing.BACK;
Size pictureSize;
try {
pictureSize = mAppController.getResolutionSetting().getPictureSize(
mAppController.getCameraProvider().getCurrentCameraId(),
cameraFacing);
} catch (OneCameraAccessException ex) {
mAppController.getFatalErrorHandler().onGenericCameraAccessFailure();
return;
}
 
mCameraSettings.setPhotoSize(pictureSize.toPortabilitySize());
 
if (ApiHelper.IS_NEXUS_5) {
if (ResolutionUtil.NEXUS_5_LARGE_16_BY_9.equals(pictureSize)) {
mShouldResizeTo16x9 = true;
} else {
mShouldResizeTo16x9 = false;
}
}
 
// Set a preview size that is closest to the viewfinder height and has
// the right aspect ratio.
List<Size> sizes = Size.convert(mCameraCapabilities.getSupportedPreviewSizes());
Size optimalSize = CameraUtil.getOptimalPreviewSize(sizes,
(double) pictureSize.width() / pictureSize.height());
Size original = new Size(mCameraSettings.getCurrentPreviewSize());
if (!optimalSize.equals(original)) {
Log.v(TAG, "setting preview size. optimal: " + optimalSize + "original: " + original);
mCameraSettings.setPreviewSize(optimalSize.toPortabilitySize());
 
mCameraDevice.applySettings(mCameraSettings);
mCameraSettings = mCameraDevice.getSettings();
}
 
if (optimalSize.width() != 0 && optimalSize.height() != 0) {
Log.v(TAG, "updating aspect ratio");
mUI.updatePreviewAspectRatio((float) optimalSize.width()
/ (float) optimalSize.height());
}
Log.d(TAG, "Preview size is " + optimalSize);
}
 
 
 
```

####   
这里会调用 pictureSize = mAppController.getResolutionSetting().getPictureSize(  
mAppController.getCameraProvider().getCurrentCameraId(),  
cameraFacing); 获取预览大小

#### 3.2 ResolutionSetting.[java 源码](https://so.csdn.net/so/search?q=java%E6%BA%90%E7%A0%81&spm=1001.2101.3001.7020)分析

根据上面代码分析得出就是调用 ResolutionSetting.java 的 getPictureSize(）来获取预览相机拍照的像素大小

```
public Size getPictureSize(CameraId cameraId, Facing cameraFacing)
throws OneCameraAccessException {
final String pictureSizeSettingKey = cameraFacing == OneCamera.Facing.FRONT ?
Keys.KEY_PICTURE_SIZE_FRONT : Keys.KEY_PICTURE_SIZE_BACK;
 
Size pictureSize = null;
 
String blacklist = "";
if (cameraFacing == OneCamera.Facing.BACK) {
blacklist = mResolutionBlackListBack;
} else if (cameraFacing == OneCamera.Facing.FRONT) {
blacklist = mResolutionBlackListFront;
}
 
// If there is no saved picture size preference or the saved on is
// blacklisted., pick a largest size with 4:3 aspect
boolean isPictureSizeSettingSet =
mSettingsManager.isSet(SettingsManager.SCOPE_GLOBAL, pictureSizeSettingKey);
boolean isPictureSizeBlacklisted = false;
 
// If a picture size is set, check whether it's blacklisted.
if (isPictureSizeSettingSet) {
pictureSize = SettingsUtil.sizeFromSettingString(
mSettingsManager.getString(SettingsManager.SCOPE_GLOBAL,
pictureSizeSettingKey));
isPictureSizeBlacklisted = pictureSize == null ||
ResolutionUtil.isBlackListed(pictureSize, blacklist);
}
 
// Due to b/21758681, it is possible that an invalid picture size has
// been saved to the settings. Therefore, picture size is set AND is not
// blacklisted, but completely invalid. In these cases, need to take the
// fallback, instead of the saved value. This logic should now save a
// valid picture size to the settings and self-correct the state of the
// settings.
final boolean isPictureSizeFromSettingsValid = pictureSize != null &&
pictureSize.width() > 0 && pictureSize.height() > 0;
 
if (!isPictureSizeSettingSet || isPictureSizeBlacklisted || !isPictureSizeFromSettingsValid) {
final Rational aspectRatio = ResolutionUtil.ASPECT_RATIO_4x3;
 
OneCameraCharacteristics cameraCharacteristics =
mOneCameraManager.getOneCameraCharacteristics(cameraId);
 
final List<Size> supportedPictureSizes =
ResolutionUtil.filterBlackListedSizes(
cameraCharacteristics.getSupportedPictureSizes(ImageFormat.JPEG),
blacklist);
final Size fallbackPictureSize =
ResolutionUtil.getLargestPictureSize(aspectRatio, supportedPictureSizes);
mSettingsManager.set(
SettingsManager.SCOPE_GLOBAL,
pictureSizeSettingKey,
SettingsUtil.sizeToSettingString(fallbackPictureSize));
pictureSize = fallbackPictureSize;
Log.e(TAG, "Picture size setting is not set. Choose " + fallbackPictureSize);
// Crash here if invariants are violated
Preconditions.checkNotNull(fallbackPictureSize);
Preconditions.checkState(fallbackPictureSize.width() > 0
&& fallbackPictureSize.height() > 0);
}
return pictureSize;
}
 
 
 
```

#### 代码中 if (!isPictureSizeSettingSet || isPictureSizeBlacklisted || !isPictureSizeFromSettingsValid) {  
final Rational aspectRatio = ResolutionUtil.ASPECT_RATIO_4x3;  
}  
就是默认 1280*960 的像素  
 

#### 3.3 解决方法

```
所以修改为:
 if (!isPictureSizeSettingSet || isPictureSizeBlacklisted || !isPictureSizeFromSettingsValid) {
    - final Rational aspectRatio = ResolutionUtil.ASPECT_RATIO_4x3;
    + final Rational aspectRatio = ResolutionUtil.ASPECT_RATIO_16x9;
}
```
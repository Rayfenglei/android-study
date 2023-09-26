> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125731075)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 微信视频通话上下颠倒问题分析](#t1)

[3. 微信视频通话上下颠倒问题相关代码分析](#t2)

[3.1Camera.java 相关代码分析](#t3)

[3.2 微信视频通话上下颠倒问题的解决](#t4)

1. 概述
-----

在 11.0 12.0 的产品定制化中，在安装微信 app 后，进行视频通话的过程中，发现视频画面上下颠倒，这样会给客户体验带来很不好的影响  
所以要求修复这个视频通话上下颠倒的问题

2. 微信视频通话上下颠倒问题分析
-----------------

在视频通话中，调用的是系统 Camera 来进行视频通话的，这时候需要通过 Camera 的相关参数来了解下视频通话后设置的相关方向来  
进一步排查修复问题

```
 
  CameraInfo 相关参数
  CameraInfo类用来描述相机信息，通过Camera类中getCameraInfo(int cameraId, CameraInfo cameraInfo)方法获得，主要包括以下两个成员变量：
    facing
    facing 代表相机的方向，它的值只能是CAMERA_FACING_BACK（后置摄像头） 或者CAMERA_FACING_FRONT（前置摄像头）。
    CAMERA_FACING_BACK 和 CAMERA_FACING_FRONT 是CameraInfo类中的静态变量
 
    orientation
    orientation是相机采集图片的角度。这个值是相机所采集的图片需要顺时针旋转至自然方向的角度值。它必须是0，90,180或270中的一个。
    举个栗子：
    假如你自然地竖着拿着手机（就是自拍时候的样子...），后置摄像头的传感器在手机里是水平方向的，你现在看着手机，如果传感器的顶部在自然方向上手机屏幕的右边（此时，手机是竖屏，传感器是横屏），那么这个orientation的值就是90。 如果前置摄像头的传感器顶部在手机屏幕右边，那么这个值就是270.
 
    setDisplayOrientation
设置预览画面顺时针旋转的角度。这个方法会影响预览图像和拍照后显示的照片。这个方法对竖屏应用非常有用。
注意，前置摄像头在进行角度旋转之前，图像会进行一个水平的镜像翻转。
所以用户在看预览图像的时候就像照镜子，看到的是现实的水平方向的镜像。
 
Size
 
图片大小，里面包含两个变量：width和height（图片的宽和高）
Parameters
 
Parameters是相机服务设置，不同的相机可能是不相同的。比如相机所支持的图片大小，对焦模式等等。下面介绍一下这个类中常用的方法
 
    getSupportedPreviewSizes()
    获得相机支持的预览图片大小，返回值是一个List<Size>数组
 
    setPreviewSize(int width, int height)
    设置相机预览图片的大小
 
    getSupportedPreviewFormats()
    获得相机支持的图片预览格式，所有的相机都支持ImageFormat.NV21
    更多的图片格式可以自行百度或是查看ImageFormat类
 
    setPreviewFormat(int pixel_format)
    设置预览图片的格式
 
    getSupportedPictureSizes()
    获得相机支持的采集的图片大小(即拍照后保存的图片的大小)
 
    setPictureSize(int width, int height)
    设置保存的图片的大小
 
    getSupportedPictureFormats()
    获得相机支持的图片格式
 
    setPictureFormat(int pixel_format)
    设置保存的图片的格式
 
    getSupportedFocusModes()
    获得相机支持的对焦模式
 
    setFocusMode(String value)
    设置相机的对焦模式
 
    getMaxNumDetectedFaces()
    返回当前相机所支持的最大的人脸检测个数
    比如，我的手机是vivo x9，后置摄像头所支持最大的人脸检测个数是10，也就是说当画面中人脸数超过10个的时候，只能检测到10张人脸
 
PreviewCallback
 
PreviewCallback是一个抽象接口
 
    void onPreviewFrame(byte[] data, Camera camera)
    通过onPreviewFrame方法来获取到相机预览的数据，第一个参数data，就是相机预览到的原始数据。
 
这些预览到的原始数据是非常有用的，比如我们可以保存下来当做一张照片，还有很多第三方的人脸检测及静默活体检测的sdk，都需要我们把相机预览的数据实时地传递过去。
 
Face
Face类用来描述通过Camera的人脸检测功能检测到的人脸信息
 
    rect
    rect 是一个Rect对象，它所表示的就是检测到的人脸的区域。
 
    注意：这个Rect对象中的坐标系并不是安卓屏幕的坐标系，需要进行转换后才能使用，具体会在后面实现人脸检测功能的时候详细介绍
 
    score
    检测到的人脸的可信度，范围是1 到100
 
    leftEye
    leftEye 是一个Point对象，表示检测到的左眼的位置坐标
 
    rightEye
    rightEye是一个Point对象，表示检测到的右眼的位置坐标
 
    mouth
    mouth是一个Point对象，表示检测到的嘴的位置坐标
    leftEye ，rightEye和mouth这3个人脸中关键点，并不是所有相机都支持的，如果相机不支持的话，这3个的值为null
```

3. 微信视频通话上下颠倒问题相关代码分析
---------------------

#### 3.1Camera.java 相关代码分析

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
 
@UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P, trackingBug = 115609023)
private long mNativeContext; // accessed by native methods
private EventHandler mEventHandler;
private ShutterCallback mShutterCallback;
private PictureCallback mRawImageCallback;
private PictureCallback mJpegCallback;
private PreviewCallback mPreviewCallback;
private boolean mUsingPreviewAllocation;
private PictureCallback mPostviewCallback;
private AutoFocusCallback mAutoFocusCallback;
private AutoFocusMoveCallback mAutoFocusMoveCallback;
private OnZoomChangeListener mZoomListener;
private FaceDetectionListener mFaceListener;
private ErrorCallback mErrorCallback;
private ErrorCallback mDetailedErrorCallback;
private boolean mOneShot;
private boolean mWithBuffer;
private boolean mFaceDetectionRunning = false;
private final Object mAutoFocusCallbackLock = new Object();
 
private final Object mShutterSoundLock = new Object();
// for AppOps
private @Nullable IAppOpsService mAppOps;
private IAppOpsCallback mAppOpsCallback;
@GuardedBy("mShutterSoundLock")
private boolean mHasAppOpsPlayAudio = true;
@GuardedBy("mShutterSoundLock")
private boolean mShutterSoundEnabledFromApp = true;
 
private static final int NO_ERROR = 0;
 
/**
* Broadcast Action:  A new picture is taken by the camera, and the entry of
* the picture has been added to the media store.
* {@link android.content.Intent#getData} is URI of the picture.
*
* <p>In {@link android.os.Build.VERSION_CODES#N Android N} this broadcast was removed, and
* applications are recommended to use
* {@link android.app.job.JobInfo.Builder JobInfo.Builder}.{@link android.app.job.JobInfo.Builder#addTriggerContentUri}
* instead.</p>
*
* <p>In {@link android.os.Build.VERSION_CODES#O Android O} this broadcast has been brought
* back, but only for <em>registered</em> receivers.  Apps that are actively running can
* again listen to the broadcast if they want an immediate clear signal about a picture
* being taken, however anything doing heavy work (or needing to be launched) as a result of
* this should still use JobScheduler.</p>
*/
@SdkConstant(SdkConstantType.BROADCAST_INTENT_ACTION)
public static final String ACTION_NEW_PICTURE = "android.hardware.action.NEW_PICTURE";
 
/**
* Broadcast Action:  A new video is recorded by the camera, and the entry
* of the video has been added to the media store.
* {@link android.content.Intent#getData} is URI of the video.
*
* <p>In {@link android.os.Build.VERSION_CODES#N Android N} this broadcast was removed, and
* applications are recommended to use
* {@link android.app.job.JobInfo.Builder JobInfo.Builder}.{@link android.app.job.JobInfo.Builder#addTriggerContentUri}
* instead.</p>
*
* <p>In {@link android.os.Build.VERSION_CODES#O Android O} this broadcast has been brought
* back, but only for <em>registered</em> receivers.  Apps that are actively running can
* again listen to the broadcast if they want an immediate clear signal about a video
* being taken, however anything doing heavy work (or needing to be launched) as a result of
* this should still use JobScheduler.</p>
*/
@SdkConstant(SdkConstantType.BROADCAST_INTENT_ACTION)
public static final String ACTION_NEW_VIDEO = "android.hardware.action.NEW_VIDEO";
 
/**
* Camera HAL device API version 1.0
* @hide
*/
@UnsupportedAppUsage
public static final int CAMERA_HAL_API_VERSION_1_0 = 0x100;
 
/**
* A constant meaning the normal camera connect/open will be used.
*/
private static final int CAMERA_HAL_API_VERSION_NORMAL_CONNECT = -2;
 
/**
* Used to indicate HAL version un-specified.
*/
private static final int CAMERA_HAL_API_VERSION_UNSPECIFIED = -1;
 
/**
* Hardware face detection. It does not use much CPU.
*/
private static final int CAMERA_FACE_DETECTION_HW = 0;
 
/**
* Software face detection. It uses some CPU.
*/
private static final int CAMERA_FACE_DETECTION_SW = 1;
 
/**
* Returns the number of physical cameras available on this device.
* The return value of this method might change dynamically if the device
* supports external cameras and an external camera is connected or
* disconnected.
*
* If there is a
* {@link android.hardware.camera2.CameraCharacteristics#REQUEST_AVAILABLE_CAPABILITIES_LOGICAL_MULTI_CAMERA
* logical multi-camera} in the system, to maintain app backward compatibility, this method will
* only expose one camera per facing for all logical camera and physical camera groups.
* Use camera2 API to see all cameras.
*
* @return total number of accessible camera devices, or 0 if there are no
*   cameras or an error was encountered enumerating them.
*/
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
private native static void _getCameraInfo(int cameraId, CameraInfo cameraInfo);
 
/**
* Information about a camera
*
* @deprecated We recommend using the new {@link android.hardware.camera2} API for new
*             applications.
*/
@Deprecated
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
 
/**
* <p>The orientation of the camera image. The value is the angle that the
* camera image needs to be rotated clockwise so it shows correctly on
* the display in its natural orientation. It should be 0, 90, 180, or 270.</p>
*
* <p>For example, suppose a device has a naturally tall screen. The
* back-facing camera sensor is mounted in landscape. You are looking at
* the screen. If the top side of the camera sensor is aligned with the
* right edge of the screen in natural orientation, the value should be
* 90. If the top side of a front-facing camera sensor is aligned with
* the right of the screen, the value should be 270.</p>
*
* @see #setDisplayOrientation(int)
* @see Parameters#setRotation(int)
* @see Parameters#setPreviewSize(int, int)
* @see Parameters#setPictureSize(int, int)
* @see Parameters#setJpegThumbnailSize(int, int)
*/
public int orientation;
 
/**
* <p>Whether the shutter sound can be disabled.</p>
*
* <p>On some devices, the camera shutter sound cannot be turned off
* through {@link #enableShutterSound enableShutterSound}. This field
* can be used to determine whether a call to disable the shutter sound
* will succeed.</p>
*
* <p>If this field is set to true, then a call of
* {@code enableShutterSound(false)} will be successful. If set to
* false, then that call will fail, and the shutter sound will be played
* when {@link Camera#takePicture takePicture} is called.</p>
*/
public boolean canDisableShutterSound;
};
 
/**
* Creates a new Camera object to access a particular hardware camera. If
* the same camera is opened by other applications, this will throw a
* RuntimeException.
*
* <p>You must call {@link #release()} when you are done using the camera,
* otherwise it will remain locked and be unavailable to other applications.
*
* <p>Your application should only have one Camera object active at a time
* for a particular hardware camera.
*
* <p>Callbacks from other methods are delivered to the event loop of the
* thread which called open().  If this thread has no event loop, then
* callbacks are delivered to the main application event loop.  If there
* is no main application event loop, callbacks are not delivered.
*
* <p class="caution"><b>Caution:</b> On some devices, this method may
* take a long time to complete.  It is best to call this method from a
* worker thread (possibly using {@link android.os.AsyncTask}) to avoid
* blocking the main application UI thread.
*
* @param cameraId the hardware camera to access, between 0 and
*     {@link #getNumberOfCameras()}-1.
* @return a new Camera object, connected, locked and ready for use.
* @throws RuntimeException if opening the camera fails (for example, if the
*     camera is in use by another process or device policy manager has
*     disabled the camera).
* @see android.app.admin.DevicePolicyManager#getCameraDisabled(android.content.ComponentName)
*/
public static Camera open(int cameraId) {
return new Camera(cameraId);
}
 
/**
* Creates a new Camera object to access the first back-facing camera on the
* device. If the device does not have a back-facing camera, this returns
* null. Otherwise acts like the {@link #open(int)} call.
*
* @return a new Camera object for the first back-facing camera, or null if there is no
*  backfacing camera
* @see #open(int)
*/
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
 
/**
* Creates a new Camera object to access a particular hardware camera with
* given hal API version. If the same camera is opened by other applications
* or the hal API version is not supported by this device, this will throw a
* RuntimeException.
* <p>
* You must call {@link #release()} when you are done using the camera,
* otherwise it will remain locked and be unavailable to other applications.
* <p>
* Your application should only have one Camera object active at a time for
* a particular hardware camera.
* <p>
* Callbacks from other methods are delivered to the event loop of the
* thread which called open(). If this thread has no event loop, then
* callbacks are delivered to the main application event loop. If there is
* no main application event loop, callbacks are not delivered.
* <p class="caution">
* <b>Caution:</b> On some devices, this method may take a long time to
* complete. It is best to call this method from a worker thread (possibly
* using {@link android.os.AsyncTask}) to avoid blocking the main
* application UI thread.
*
* @param cameraId The hardware camera to access, between 0 and
* {@link #getNumberOfCameras()}-1.
* @param halVersion The HAL API version this camera device to be opened as.
* @return a new Camera object, connected, locked and ready for use.
*
* @throws IllegalArgumentException if the {@code halVersion} is invalid
*
* @throws RuntimeException if opening the camera fails (for example, if the
* camera is in use by another process or device policy manager has disabled
* the camera).
*
* @see android.app.admin.DevicePolicyManager#getCameraDisabled(android.content.ComponentName)
* @see #CAMERA_HAL_API_VERSION_1_0
*
* @hide
*/
@UnsupportedAppUsage
public static Camera openLegacy(int cameraId, int halVersion) {
if (halVersion < CAMERA_HAL_API_VERSION_1_0) {
throw new IllegalArgumentException("Invalid HAL version " + halVersion);
}
 
return new Camera(cameraId, halVersion);
}
 
/**
* Create a legacy camera object.
*
* @param cameraId The hardware camera to access, between 0 and
* {@link #getNumberOfCameras()}-1.
* @param halVersion The HAL API version this camera device to be opened as.
*/
private Camera(int cameraId, int halVersion) {
int err = cameraInitVersion(cameraId, halVersion);
if (checkInitErrors(err)) {
if (err == -EACCES) {
throw new RuntimeException("Fail to connect to camera service");
} else if (err == -ENODEV) {
throw new RuntimeException("Camera initialization failed");
} else if (err == -ENOSYS) {
throw new RuntimeException("Camera initialization failed because some methods"
+ " are not implemented");
} else if (err == -EOPNOTSUPP) {
throw new RuntimeException("Camera initialization failed because the hal"
+ " version is not supported by this device");
} else if (err == -EINVAL) {
throw new RuntimeException("Camera initialization failed because the input"
+ " arugments are invalid");
} else if (err == -EBUSY) {
throw new RuntimeException("Camera initialization failed because the camera"
+ " device was already opened");
} else if (err == -EUSERS) {
throw new RuntimeException("Camera initialization failed because the max"
+ " number of camera devices were already opened");
}
// Should never hit this.
throw new RuntimeException("Unknown camera error");
}
}
...
}
 
 
}
```

#### 在 Camera open() 打开摄像头的时候，调用 getCameraInfo(i, cameraInfo); 设置相关的参数  
所以就可以在 getCameraInfo(i, cameraInfo); 来旋转摄像头方向

#### 3.2 微信视频通话上下颠倒问题的解决

```
public static void getCameraInfo(int cameraId, CameraInfo cameraInfo) {
_getCameraInfo(cameraId, cameraInfo);
 
// 获取当前app是微信app时旋转摄像头方向
+if(ActivityThread.currentOpPackageName().contains("com.tencent.mm")){
+cameraInfo.orientation = (cameraInfo.orientation+180)%360;
+}
 
 
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
```
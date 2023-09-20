> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124855256)

1. 概述
-----

在 11.0 的产品开发中，对于在系统 api 中添加自定义的服务 接口 [jni](https://so.csdn.net/so/search?q=jni&spm=1001.2101.3001.7020) 也是常见的功能，最近有需求要求在系统添加对数据的读取，所以用 jni 更方便读取，所以需要增加自定义 jni 方法

2.framework 系统 api 添加 jni 方法的相关类
--------------------------------

```
frameworks/base/core/jni/android_mdm_SystemUtils.cpp
frameworks/base/core/android/mdm/SystemUtils.java
frameworks/base/core/jni/Android.bp
frameworks/base/core/jni/AndroidRuntime.cpp


```

3.framework 系统 api 添加 jni 方法的核心功能分析实现
-------------------------------------

3.1 添加自定义类 SystemUtils.java 类供上层调用
----------------------------------

android 下新添加 mdm 文件夹  
建立 SystemUtils.java 类  
首先定义自定义类来提供给上层调用，在自定义类中调用 jni 方法

```
package android.mdm;
import android.util.Log;

public class SystemUtils {
    private native void native_setValue(int vlaue);
    private native int native_getValue();
    public int getValue() {
        Log.e("SystemUtils","read Begin");
        int value = native_getValue();
        Log.e("SystemUtils","value:"+value);
        return value;
    }
	
	public void setValue(int value){
		native_setValue(value);
	}
}

```

3.2 在 frameworks/base/core/jni 下添加对应的 cpp 文件
--------------------------------------------

新建 android_mdm_SystemUtils.cpp 文件 在 android_mdm_SystemUtils.cpp 中实现 jni 的接口  
在 jni 方法中，实现该实现的功能

```
#define LOG_TAG "SystemUtilsJNI"
 
 #include "jni.h"
 #include "android_runtime/AndroidRuntime.h"
 
 #include <utils/misc.h>
 #include <utils/Log.h>
 #include <stdio.h>
 
namespace android
{
	static int jniData = 1;
	
	static void systemUtils_setVal(JNIEnv* env, jobject clazz, jint val)
	{
		jniData = val;
		ALOGI("mdmManager_setVal......jniData=%d\n",jniData);
	}
	
	static jint systemUtils_getVal(JNIEnv* env, jobject clazz)
	{
		ALOGI("mdmManager_getVal......jniData=%d\n",jniData);
		return jniData+50;
	}
	static const char* const kClassPathName = "android/mdm/SystemUtils";
	/*Java本地接口方法表*/
     static const JNINativeMethod method_table[] = {
		{"native_setValue", "(I)V",(void*)systemUtils_setVal},
		{"native_getValue", "()I",(void*)systemUtils_getVal},
	};
	 //注册java navive方法到AndroidRuntime中
	int register_android_mdm_SystemUtils(JNIEnv* env)
	{
		// Get the VendorUtilsClass class
		jclass vendorUtilsClass = env->FindClass(kClassPathName);
		if (vendorUtilsClass == NULL) {
			ALOGE("Can't find %s", kClassPathName);
			return -1;
		}
		int status = AndroidRuntime::registerNativeMethods(env,
					   kClassPathName, method_table, NELEM(method_table));
		return status;
	}
}

```

3. 3 在 AndroidRuntime.cpp 中注册 JNI 参与到系统编译中去
-------------------------------------------

```
static const RegJNIRec gRegJNI[] = {
+    REG_JNI(register_android_mdm_SystemUtils),
}
namespace android {
+extern int register_android_mdm_SystemUtils(JNIEnv *env);
}

```

3.4 frameworks/base/core/jni 目录下 Android.bp 文件中添加需要编译的本地文件
----------------------------------------------------------

```
 srcs: [
        "AndroidRuntime.cpp",
        "com_android_internal_content_NativeLibraryHelper.cpp",
        "com_google_android_gles_jni_EGLImpl.cpp",
        "com_google_android_gles_jni_GLImpl.cpp", // TODO: .arm
        "android_app_Activity.cpp",
        "android_app_ActivityThread.cpp",
        "android_app_NativeActivity.cpp",
        "android_app_admin_SecurityLog.cpp",
        "android_opengl_EGL14.cpp",
        "android_opengl_EGL15.cpp",
        "android_opengl_EGLExt.cpp",
        "android_opengl_GLES10.cpp",
        "android_opengl_GLES10Ext.cpp",
        "android_opengl_GLES11.cpp",
        "android_opengl_GLES11Ext.cpp",
        "android_opengl_GLES20.cpp",
        "android_opengl_GLES30.cpp",
        "android_opengl_GLES31.cpp",
        "android_opengl_GLES31Ext.cpp",
        "android_opengl_GLES32.cpp",
        "android_database_CursorWindow.cpp",
        "android_database_SQLiteCommon.cpp",
        "android_database_SQLiteConnection.cpp",
        "android_database_SQLiteGlobal.cpp",
        "android_database_SQLiteDebug.cpp",
        "android_graphics_Canvas.cpp",
        "android_graphics_ColorSpace.cpp",
        "android_graphics_drawable_AnimatedVectorDrawable.cpp",
        "android_graphics_drawable_VectorDrawable.cpp",
        "android_graphics_Picture.cpp",
        "android_view_CompositionSamplingListener.cpp",
        "android_view_DisplayEventReceiver.cpp",
        "android_view_DisplayListCanvas.cpp",
        "android_view_TextureLayer.cpp",
        "android_view_InputChannel.cpp",
        "android_view_InputDevice.cpp",
        "android_view_InputEventReceiver.cpp",
        "android_view_InputEventSender.cpp",
        "android_view_InputQueue.cpp",
        "android_view_KeyCharacterMap.cpp",
        "android_view_KeyEvent.cpp",
        "android_view_MotionEvent.cpp",
        "android_view_PointerIcon.cpp",
        "android_view_RenderNode.cpp",
        "android_view_RenderNodeAnimator.cpp",
        "android_view_Surface.cpp",
        "android_view_SurfaceControl.cpp",
        "android_view_SurfaceSession.cpp",
        "android_view_TextureView.cpp",
        "android_view_ThreadedRenderer.cpp",
        "android_view_VelocityTracker.cpp",
        "android_text_AndroidCharacter.cpp",
        "android_text_Hyphenator.cpp",
        "android_os_Debug.cpp",
        "android_os_GraphicsEnvironment.cpp",
        "android_os_HidlSupport.cpp",
        "android_os_HwBinder.cpp",
        "android_os_HwBlob.cpp",
        "android_os_HwParcel.cpp",
        "android_os_HwRemoteBinder.cpp",
        "android_os_NativeHandle.cpp",
        "android_os_MemoryFile.cpp",
        "android_os_MessageQueue.cpp",
        "android_os_Parcel.cpp",
        "android_os_SELinux.cpp",
        "android_os_SharedMemory.cpp",
        "android_os_SystemClock.cpp",
        "android_os_SystemProperties.cpp",
        "android_os_Trace.cpp",
        "android_os_UEventObserver.cpp",
        "android_os_VintfObject.cpp",
        "android_os_VintfRuntimeInfo.cpp",
        "android_net_LocalSocketImpl.cpp",
        "android_net_NetUtils.cpp",
        "android_nio_utils.cpp",
        "android_util_AssetManager.cpp",
        "android_util_Binder.cpp",
        "android_util_EventLog.cpp",
        "android_util_Log.cpp",
        "android_util_StatsLog.cpp",
        "android_util_MemoryIntArray.cpp",
        "android_util_PathParser.cpp",
        "android_util_Process.cpp",
        "android_util_StringBlock.cpp",
        "android_util_XmlBlock.cpp",
        "android_util_jar_StrictJarFile.cpp",
        "android/graphics/AnimatedImageDrawable.cpp",
        "android/graphics/Bitmap.cpp",
        "android/graphics/BitmapFactory.cpp",
        "android/graphics/BitmapFactoryEx.cpp",
        "android/graphics/ByteBufferStreamAdaptor.cpp",
        "android/graphics/Camera.cpp",
        "android/graphics/CanvasProperty.cpp",
        "android/graphics/ColorFilter.cpp",
        "android/graphics/FontFamily.cpp",
        "android/graphics/FontUtils.cpp",
        "android/graphics/CreateJavaOutputStreamAdaptor.cpp",
        "android/graphics/GIFMovie.cpp",
        "android/graphics/GraphicBuffer.cpp",
        "android/graphics/Graphics.cpp",
        "android/graphics/ImageDecoder.cpp",
        "android/graphics/Interpolator.cpp",
        "android/graphics/MaskFilter.cpp",
        "android/graphics/Matrix.cpp",
        "android/graphics/Movie.cpp",
        "android/graphics/MovieImpl.cpp",
        "android/graphics/NinePatch.cpp",
        "android/graphics/NinePatchPeeker.cpp",
        "android/graphics/Paint.cpp",
        "android/graphics/PaintFilter.cpp",
        "android/graphics/Path.cpp",
        "android/graphics/PathMeasure.cpp",
        "android/graphics/PathEffect.cpp",
        "android/graphics/Picture.cpp",
        "android/graphics/BitmapRegionDecoder.cpp",
        "android/graphics/Region.cpp",
        "android/graphics/Shader.cpp",
        "android/graphics/SkDrmStream.cpp",
        "android/graphics/SurfaceTexture.cpp",
        "android/graphics/Typeface.cpp",
        "android/graphics/Utils.cpp",
        "android/graphics/YuvToJpegEncoder.cpp",
        "android/graphics/fonts/Font.cpp",
        "android/graphics/fonts/FontFamily.cpp",
        "android/graphics/pdf/PdfDocument.cpp",
        "android/graphics/pdf/PdfEditor.cpp",
        "android/graphics/pdf/PdfRenderer.cpp",
        "android/graphics/pdf/PdfUtils.cpp",
        "android/graphics/text/LineBreaker.cpp",
        "android/graphics/text/MeasuredText.cpp",
        "android_media_AudioEffectDescriptor.cpp",
        "android_media_AudioRecord.cpp",
        "android_media_AudioSystem.cpp",
        "android_media_AudioTrack.cpp",
        "android_media_AudioAttributes.cpp",
        "android_media_AudioProductStrategies.cpp",
        "android_media_AudioVolumeGroups.cpp",
        "android_media_AudioVolumeGroupCallback.cpp",
        "android_media_DeviceCallback.cpp",
        "android_media_JetPlayer.cpp",
        "android_media_MediaMetricsJNI.cpp",
        "android_media_MicrophoneInfo.cpp",
        "android_media_midi.cpp",
        "android_media_RemoteDisplay.cpp",
        "android_media_ToneGenerator.cpp",
        "android_hardware_Camera.cpp",
        "android_hardware_camera2_CameraMetadata.cpp",
        "android_hardware_camera2_legacy_LegacyCameraDevice.cpp",
        "android_hardware_camera2_legacy_PerfMeasurement.cpp",
        "android_hardware_camera2_DngCreator.cpp",
        "android_hardware_display_DisplayViewport.cpp",
        "android_hardware_HardwareBuffer.cpp",
        "android_hardware_SensorManager.cpp",
        "android_hardware_SerialPort.cpp",
        "android_hardware_SoundTrigger.cpp",
        "android_hardware_UsbDevice.cpp",
        "android_hardware_UsbDeviceConnection.cpp",
        "android_hardware_UsbRequest.cpp",
        "android_hardware_location_ActivityRecognitionHardware.cpp",
        "android_util_FileObserver.cpp",
        "android/opengl/poly_clip.cpp", // TODO: .arm
        "android/opengl/util.cpp",
        "android_server_NetworkManagementSocketTagger.cpp",
        "android_ddm_DdmHandleNativeHeap.cpp",
        "android_backup_BackupDataInput.cpp",
        "android_backup_BackupDataOutput.cpp",
        "android_backup_FileBackupHelperBase.cpp",
        "android_backup_BackupHelperDispatcher.cpp",
        "android_app_backup_FullBackup.cpp",
        "android_content_res_ApkAssets.cpp",
        "android_content_res_ObbScanner.cpp",
        "android_content_res_Configuration.cpp",
        "android_animation_PropertyValuesHolder.cpp",
        "android_security_Scrypt.cpp",
        "com_android_internal_os_AtomicDirectory.cpp",
        "com_android_internal_os_ClassLoaderFactory.cpp",
        "com_android_internal_os_FuseAppLoop.cpp",
        "com_android_internal_os_Zygote.cpp",
        "com_android_internal_os_ZygoteInit.cpp",
        "com_android_internal_util_VirtualRefBasePtr.cpp",
        "com_android_internal_view_animation_NativeInterpolatorFactoryHelper.cpp",
        "hwbinder/EphemeralStorage.cpp",
        "fd_utils.cpp",
        "android_hardware_input_InputWindowHandle.cpp",
        "android_hardware_input_InputApplicationHandle.cpp",
    +    "android_mdm_SystemUtils.cpp",
    ],

```

```
我们修改的代码位于libandroid_runtime.so和framework.jar，接下来到android 源码根目录下，执行 make
libandroid_runtime 和 make framework 进行重新编译。将得到的libandroid_runtime.so

```
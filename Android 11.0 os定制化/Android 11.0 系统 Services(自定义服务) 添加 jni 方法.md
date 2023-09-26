> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124855246)

在 11.0 系统开发中由于与底层进行通讯，涉及到 [jni](https://so.csdn.net/so/search?q=jni&spm=1001.2101.3001.7020) 方法添加，本篇讲解正确在系统服务中添加 jni 的方法详解

1. 在 [aidl](https://so.csdn.net/so/search?q=aidl&spm=1001.2101.3001.7020) 文件中 增加调用 native 方法接口

```
--- a/frameworks/base/core/java/android/os/IMdmManager.aidl
+++ b/frameworks/base/core/java/android/os/IMdmManager.aidl
@@ -80,4 +80,6 @@ interface IMdmManager {
     void allowAccessUsageDetails(String pkg);
     void turnOnEyeComfort(boolean on);
     boolean isEyeComfortTurnedOn();
+       void setVal(int val);
+    int getVal();
 }

```

2. 在 MdmManager.java 中实现接口

```
+    public void setVal_native(int val){
+        try {
+            mMdmManagerSerivce.setVal(val);
+        } catch (RemoteException e) {
+            e.printStackTrace();
+        }
+    }
+
+    public int getValue(){
+        try {
+            return mMdmManagerSerivce.getVal();
+        } catch (RemoteException e) {
+            e.printStackTrace();
+        }
+        return -1;
+    }

```

3. 在 MdmManagerServices.java 系统服务中 添加 [native 方法](https://so.csdn.net/so/search?q=native%E6%96%B9%E6%B3%95&spm=1001.2101.3001.7020)的调用

```
    @Override
    public void setVal(int val) throws RemoteException {
        setVal_native(val);
    }

    @Override
    public int getVal() throws RemoteException {
        return getVal_native();
    }

    public native void setVal_native(int val);
    public native int getVal_native();

4. 在添加提供app调用的自定义SuperPower.java 添加调用接口
    /**
     * @see
     * @param value
     */
    public void setValue(int value){
        mdmApiManager.setVal_native(value);
    }
    /**
     * @see
     * @return
     */
    public int getValue(){
        return mdmApiManager.getValue();
    }	

```

5. 接下来 绑定 MdmManagerServices.java 和 com_android_server_MdmManagerService.cpp  
在 frameworks\base\services\core\jni 目录下  
新建 com_android_server_MdmManagerService.cpp 文件  
新建 cpp 名称要和 MdmManagerService 的包名对应

```
 
 #define LOG_TAG "MdmManagerServiceJNI"
 
 #include "jni.h"
 #include "android_runtime/AndroidRuntime.h"
 
 #include <utils/misc.h>
 #include <utils/Log.h>
 #include <stdio.h>
 
 namespace android
 {
	static int jniData = 1;
	
	static void mdmManager_setVal(JNIEnv* env, jobject clazz, jint val)
	{
		jniData = val;
		ALOGI("mdmManager_setVal......jniData=%d\n",jniData);
	}
	
	static jint mdmManager_getVal(JNIEnv* env, jobject clazz)
	{
		ALOGI("mdmManager_getVal......jniData=%d\n",jniData);
		return jniData+3;
	}
	
	/*Java本地接口方法表*/
	static const JNINativeMethod method_table[] = {
		{"setVal_native", "(I)V",(void*)mdmManager_setVal},
		{"getVal_native", "()I",(void*)mdmManager_getVal},
	};
	
	/*注册Java本地接口方法*/
	int register_android_server_MdmManagerService(JNIEnv *env)
	{
		return AndroidRuntime::registerNativeMethods(env, "com/android/server/MdmManagerService", method_table, NELEM(method_table));
	}
 };

```

6. 添加 cpp 文件到 frameworks\base\services\core\jni\Android.bp 文件中

```
srcs: [
        "BroadcastRadio/JavaRef.cpp",
        "BroadcastRadio/NativeCallbackThread.cpp",
        "BroadcastRadio/BroadcastRadioService.cpp",
        "BroadcastRadio/Tuner.cpp",
        "BroadcastRadio/TunerCallback.cpp",
        "BroadcastRadio/convert.cpp",
        "BroadcastRadio/regions.cpp",
        "com_android_server_AlarmManagerService.cpp",
        "com_android_server_am_BatteryStatsService.cpp",
        "com_android_server_connectivity_Vpn.cpp",
        "com_android_server_connectivity_tethering_OffloadHardwareInterface.cpp",
        "com_android_server_ConsumerIrService.cpp",
        "com_android_server_devicepolicy_CryptoTestHelper.cpp",
        "com_android_server_HardwarePropertiesManagerService.cpp",
        "com_android_server_hdmi_HdmiCecController.cpp",
        "com_android_server_input_InputManagerService.cpp",
        "com_android_server_lights_LightsService.cpp",
        "com_android_server_location_GnssLocationProvider.cpp",
        "com_android_server_locksettings_SyntheticPasswordManager.cpp",
        "com_android_server_net_NetworkStatsService.cpp",
        "com_android_server_power_PowerManagerService.cpp",
        "com_android_server_security_VerityUtils.cpp",
        "com_android_server_SerialService.cpp",
        "com_android_server_storage_AppFuseBridge.cpp",
        "com_android_server_SystemServer.cpp",
        "com_android_server_TestNetworkService.cpp",
        "com_android_server_tv_TvUinputBridge.cpp",
        "com_android_server_tv_TvInputHal.cpp",
        "com_android_server_vr_VrManagerService.cpp",
        "com_android_server_UsbAlsaJackDetector.cpp",
        "com_android_server_UsbDeviceManager.cpp",
        "com_android_server_UsbDescriptorParser.cpp",
        "com_android_server_UsbMidiDevice.cpp",
        "com_android_server_UsbHostManager.cpp",
        "com_android_server_VibratorService.cpp",
        "com_android_server_PersistentDataBlockService.cpp",
        "com_android_server_GraphicsStatsService.cpp",
        "com_android_server_am_AppCompactor.cpp",
        "com_android_server_am_LowMemDetector.cpp",
	+ "com_android_server_MdmManagerService.cpp",
        "onload.cpp",
        ":lib_networkStatsFactory_native",
    ],

```

7. 到 frameworks\base\services\core\jni\onload.cpp 中注册 com_android_server_MdmManagerService.cpp 中的 navite 方法  
这个 onload.cpp 文件实现在 libandroid_servers 模块中，当系统加载 libandroid_servers 模块时，就会调用实现在 onload.cpp 文件中的 JNI_OnLoad 函数，这样就把前面定义的三个 JNI 方法注册到 Java 虚拟机中去了。  
添加如下：

```
namespace android {
........
int register_android_server_MdmManagerService(JNIEnv *env);
};
 
..............
extern "C" jint JNI_OnLoad(JavaVM* vm, void* reserved)
{
    .....................
    register_android_server_MdmManagerService(env);
    return JNI_VERSION_1_4;
}

```
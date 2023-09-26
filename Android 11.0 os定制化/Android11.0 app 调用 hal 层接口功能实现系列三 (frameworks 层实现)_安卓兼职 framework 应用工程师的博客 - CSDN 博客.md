> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/132253607)

1. 前言
-----

 在 11.0 的系统 rom 定制化开发中，对于一些需要在 app 中调用 [hal](https://so.csdn.net/so/search?q=hal&spm=1001.2101.3001.7020) 层的一些接口来实现某些功能而言，就需要  
打通 app 到 hal 的接口，实现功能需求，这一节首先讲在 hal 层中提供接口然后在 jni 层实现 hal 层接口调用，在  
framework 层实现添加服务调用 jni 接口

2.app 调用 hal 层接口功能实现系列三 (frameworks 层实现) 的核心类
---------------------------------------------

```
    frameworks\base\core\java\android\app\ITestService.aidl
    frameworks\base\services\core\java\com\android\server\TestService.java
    frameworks/base/services/java/com/android/server/SystemServer.java
```

3.app 调用 hal 层接口功能实现系列三 (frameworks 层实现) 的核心功能分析和实现
---------------------------------------------------

Android 系统的硬件抽象层（Hardware Abstract Layer, HAL）运行在用户空间中，它向下屏蔽硬件驱动模块的实现细节，  
向上提供硬件访问服务。通过硬件抽象层，Android 系统分为两层来支持硬件设备，其中一层  
实现在用户空间（User Space），另外一层是现在内核空间（Kernel Space）。  
传统的 Linux 系统把对硬件的支持完全是现在在内核空间，  
即把对硬件的支持完全实现在硬件驱动模块中。

HAL 是硬件抽象层，它向下屏蔽了硬件的实现细节，向上提供了抽象接口，  
HAL 是底层硬件和上层框架直接的接口，框架层通过 HAL 可以操作硬件设备  
HAL 和内核驱动，HAL 实现在用户空间，驱动在内核空间，所以为了维护各大厂商  
的利益，核心算法之类的就需要在 hal 层实现了

HIDL 的全称是 HAL interface definition language（硬件抽象层接口定义语言），在此之前 Android 有 AIDL，  
架构在 Android binder 之上，用来定义 Android 基于 Binder 通信的 Client 与 Service 之间的接口。  
HIDL 也是类似的作用，只不过定义的是 Android Framework 与 Android HAL 实现之间的接口。

在自定义 hal 模块中，在前面两节中进行了简单的接收，接下来这一节就需要在 framework 层中  
实现通过通过 aidl 来调用 hal 模块的相关接口的实现，  
首先在 frameworks\base\core\java\android\app 下  
添加 ITestService.aidl 的 aidl 来供 app 层调用

```
    package android.app;
     
    interface ITestService {
        int add(int a,int b);
    }
     
    package com.android.server;
    import android.app.ITestService;
    public class TestService extends ITestService.Stub {
        public TestService(){
          android.util.Log.d("test","Start TestService...");
          nativeInit();
        }
        @Override 
        public int add(int a, int b){
          android.util.Log.d("test","TestService add()...a = :"+a+",b = :"+b);
          return nativeAdd(a,b);
        }
        private static native void nativeInit();
        private static native int nativeAdd(int a,int b);
    }
     
    --- a/frameworks/base/services/java/com/android/server/SystemServer.java
    +++ b/frameworks/base/services/java/com/android/server/SystemServer.java
    @@ -180,7 +180,7 @@ import java.util.Locale;
     import java.util.Timer;
     import java.util.concurrent.CountDownLatch;
     import java.util.concurrent.Future;
    -
    +import com.android.server.TestService;
     public final class SystemServer {
     
         private static final String TAG = "SystemServer";
    @@ -1052,6 +1052,14 @@ public final class SystemServer {
                 ServiceManager.addService("api", mMdmManagerService);
                 traceEnd();
     
    +            try{
    +               traceBeginAndSlog("TestService");
    +                          ServiceManager.addService("test", new TestService());
    +               traceEnd();
    +                       }catch(Exception e){
    +                               e.printStackTrace();
    +                       }
    +
                 traceBeginAndSlog("StartDynamicSystemService");
                 dynamicSystem = new DynamicSystemService(context);
                 ServiceManager.addService("dynamic_system", dynamicSystem);
    @@ -1932,7 +1940,6 @@ public final class SystemServer {
                 mSystemServiceManager.startService(CrossProfileAppsService.class);
                 traceEnd();
             }
```

在上述的自定义 hal 层模块中，在  
为 Android 硬件抽象层编写 [JNI 方法](https://so.csdn.net/so/search?q=JNI%E6%96%B9%E6%B3%95&spm=1001.2101.3001.7020)供硬件服务程序调用

    4.1 JNI 实现  
    4.2 声明 JNI 注册方法  
    4.3 添加 JNI 方法代码

通过上述的 TestService.java 和 ITestService.aidl 来增加调用 hal 层的接口，在关于通过提供自定义服务来提供给  
app 调用接口，最终执行调用 hal 模块接口，而在 SystemServer.java 中在 fork 进程的过程中，启动系统其服务的时候，调用的 startOtherServices()  
中来启动 TestService(）这个自定义服务通过 new TestService() 来在 TestService（）这个自定义系统服务中的构造方法中，来打开 hal 层的模块，通过这样在开机启动服务的时候，就可以直接初始化 android_server_TestService_nativeInit 来打开 hal 自定义模块，这样就可以

方便调用 hal 的相关接口，实现对 hal 层的操作  
接下来就来解决关于服务调用的相关 selinux 的权限问题

```
   diff --git a/system/sepolicy/prebuilts/api/26.0/public/service.te b/system/sepolicy/prebuilts/api/26.0/public/service.te
    index 16a42216b2..d77121e831 100755
    --- a/system/sepolicy/prebuilts/api/26.0/public/service.te
    +++ b/system/sepolicy/prebuilts/api/26.0/public/service.te
    @@ -146,3 +146,4 @@ type wificond_service, service_manager_type;
     type wifiaware_service, app_api_service, system_server_service, service_manager_type;
     type window_service, system_api_service, system_server_service, service_manager_type;
     type api_service, system_api_service, system_server_service, service_manager_type;
    +type test_service, system_api_service, system_server_service, service_manager_type;
    diff --git a/system/sepolicy/prebuilts/api/27.0/public/service.te b/system/sepolicy/prebuilts/api/27.0/public/service.te
    index 32ca6a0e05..44b4d660c4 100755
    --- a/system/sepolicy/prebuilts/api/27.0/public/service.te
    +++ b/system/sepolicy/prebuilts/api/27.0/public/service.te
    @@ -149,3 +149,4 @@ type wificond_service, service_manager_type;
     type wifiaware_service, app_api_service, system_server_service, service_manager_type;
     type window_service, system_api_service, system_server_service, service_manager_type;
     type api_service, system_api_service, system_server_service, service_manager_type;
    +type test_service, system_api_service, system_server_service, service_manager_type;
    diff --git a/system/sepolicy/prebuilts/api/28.0/public/service.te b/system/sepolicy/prebuilts/api/28.0/public/service.te
    index 25054562d6..efe4655c77 100755
    --- a/system/sepolicy/prebuilts/api/28.0/public/service.te
    +++ b/system/sepolicy/prebuilts/api/28.0/public/service.te
    @@ -160,3 +160,4 @@ type wifiaware_service, app_api_service, system_server_service, service_manager_
     type window_service, system_api_service, system_server_service, service_manager_type;
     type wpantund_service, system_api_service, service_manager_type;
     type api_service, system_api_service, system_server_service, service_manager_type;
    +type test_service, system_api_service, system_server_service, service_manager_type;
    diff --git a/system/sepolicy/prebuilts/api/29.0/private/service_contexts b/system/sepolicy/prebuilts/api/29.0/private/service_contexts
    index cf5af11f3a..0d0660d81c 100755
    --- a/system/sepolicy/prebuilts/api/29.0/private/service_contexts
    +++ b/system/sepolicy/prebuilts/api/29.0/private/service_contexts
    @@ -220,4 +220,5 @@ wifiaware                                 u:object_r:wifiaware_service:s0
     wifirtt                                   u:object_r:rttmanager_service:s0
     window                                    u:object_r:window_service:s0
     api                                       u:object_r:api_service:s0
    +test                                     u:object_r:test_service:s0
     *                                         u:object_r:default_android_service:s0
    diff --git a/system/sepolicy/prebuilts/api/29.0/public/service.te b/system/sepolicy/prebuilts/api/29.0/public/service.te
    index 8604c191da..c12ed94183 100755
    --- a/system/sepolicy/prebuilts/api/29.0/public/service.te
    +++ b/system/sepolicy/prebuilts/api/29.0/public/service.te
    @@ -188,6 +188,7 @@ type window_service, system_api_service, system_server_service, service_manager_
     type inputflinger_service, system_api_service, system_server_service, service_manager_type;
     type wpantund_service, system_api_service, service_manager_type;
     type api_service, app_api_service, ephemeral_app_api_service, system_server_service, service_manager_type;
    +type test_service, app_api_service, ephemeral_app_api_service, system_server_service, service_manager_type;
     ###
     ### Neverallow rules
     ###
    diff --git a/system/sepolicy/private/service_contexts b/system/sepolicy/private/service_contexts
    index cf5af11f3a..0d0660d81c 100755
    --- a/system/sepolicy/private/service_contexts
    +++ b/system/sepolicy/private/service_contexts
    @@ -220,4 +220,5 @@ wifiaware                                 u:object_r:wifiaware_service:s0
     wifirtt                                   u:object_r:rttmanager_service:s0
     window                                    u:object_r:window_service:s0
     api                                       u:object_r:api_service:s0
    +test                                     u:object_r:test_service:s0
     *                                         u:object_r:default_android_service:s0
    diff --git a/system/sepolicy/public/service.te b/system/sepolicy/public/service.te
    index 8604c191da..c12ed94183 100755
    --- a/system/sepolicy/public/service.te
    +++ b/system/sepolicy/public/service.te
    @@ -188,6 +188,7 @@ type window_service, system_api_service, system_server_service, service_manager_
     type inputflinger_service, system_api_service, system_server_service, service_manager_type;
     type wpantund_service, system_api_service, service_manager_type;
     type api_service, app_api_service, ephemeral_app_api_service, system_server_service, service_manager_type;
    +type test_service, app_api_service, ephemeral_app_api_service, system_server_service, service_manager_type;
     ###
     ### Neverallow rules
     ###
```

通过上述的 system/sepolicy 下的相关文件的修改，就可以授权关于添加自定义服务的相关  
selinux 的权限，在上述的开机过程中，可以利用抓取 selinux 权限的方法，来查找调用 hal 模块接口，所需要的 selinux 权限，然后就添加相关的 selinux 需要的权限，最终就可以调用 hal 层模块的接口就可以在 SystemServer 中添加自定义服务，还可以实现在 app 中调用  
系统自定义服务接口  
通过上述在 framework 层中添加自定义服务，来调用上篇实现 native 层 jni 的 hal 层的接口，就可以在  
开机启动的时候，打开 hal 层模块接口，就实现在 app 层调用 hal 层模块的功能，所以接下来下篇  
就可以在 app 层实现部分功能
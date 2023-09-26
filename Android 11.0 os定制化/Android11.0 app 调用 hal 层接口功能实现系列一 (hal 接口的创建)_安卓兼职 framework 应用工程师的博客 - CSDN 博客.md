> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/132127412)

1. 前言
-----

在 11.0 的系统 rom 定制化开发中，对于一些需要在 app 中调用 [hal](https://so.csdn.net/so/search?q=hal&spm=1001.2101.3001.7020) 层的一些接口来实现某些功能而言，就需要  
打通 app 到 hal 的接口，实现功能需求，这一节首先讲在 hal 层中提供接口然后通过 [jni](https://so.csdn.net/so/search?q=jni&spm=1001.2101.3001.7020) 来调用，首先来建立  
hal 层的相关接口和 c++ 文件，提供 hal 层供上层调用的接口

2.app 调用 hal 层接口功能实现系列一 (hal 接口的创建) 的核心类
----------------------------------------

```
    hardware/libhardware/include/hardware/test.h
    hardware/libhardware/modules/test/test.c
    hardware/libhardware/modules/test/Android.bp
```

3.app 调用 hal 层接口功能实现系列一 (hal 接口的创建) 的核心功能分析和实现
----------------------------------------------

[HIDL](https://so.csdn.net/so/search?q=HIDL&spm=1001.2101.3001.7020) 的全称是 HAL interface definition language（硬件抽象层接口定义语言），在此之前 Android 有 AIDL，  
架构在 Android binder 之上，用来定义 Android 基于 Binder 通信的 Client 与 Service 之间的接口。  
HIDL 也是类似的作用，只不过定义的是 Android Framework 与 Android HAL 实现之间的接口。

HAL 是硬件抽象层，它向下屏蔽了硬件的实现细节，向上提供了抽象接口，  
HAL 是底层硬件和上层框架直接的接口，框架层通过 HAL 可以操作硬件设备  
HAL 和内核驱动，HAL 实现在用户空间，驱动在内核空间，所以为了维护各大厂商  
的利益，核心算法之类的就需要在 hal 层实现了

对于在 11.0 中新增一个自定义的 hal 层模块，然后让上层 app 调用，首先就得参考其他 hal 层模块  
来创建 hal 层模块的相关类，来实现 hal 层 的接口，接下来就来  
手动实现一个 HAL 模块，这个 HAL 提供一个加法函数  
首先在 hardware/libhardware/include/hardware 目录下创建 test.h 文件

```
    #include <sys/cdefs.h>
    #include <sys/types.h>
    #include <hardware/hardware.h>
    //HAL模块名
    #define TEST_HARDWARE_MODULE_ID "test"
    //HAL版本号
    #define TEST_MODULE_API_VERSION_1_0 HARDWARE_MODULE_API_VERSION(0, 1)
    //设备名
    #define HARDWARE_TEST "test"
    #define DEVICE_NAME "/dev/test"
     
    //自定义HAL模块结构体
    typedef struct test_module {
        struct hw_module_t common;
    } test_module_t;
     
    //自定义HAL设备结构体
    typedef struct test_device {
        struct hw_device_t common;
        //加法函数
        int (*additionTest)(const struct test_device *dev,int a,int b);
    } test_device_t;
     
    //给外部调用提供打开设备的函数
    static inline int _test_open(const struct hw_module_t *module,
            test_device_t **device) {
        return module->methods->open(module, HARDWARE_TEST,
                (struct hw_device_t **) device);
    }
```

在自定义 hal 层模块中上述的 test.h 中，我们可以看到在 test.h 中自定义了两个结构体，分别继承 hw_module_t 和 hw_device_t 且在第一个变量，满足前面说的规则，另外定义在 device 结构体中的函数的第一个参数也必须是 device 结构体  
接着在 hardware/libhardware/modules / 目录下创建一个 test 的文件夹，里面包含一个 test.c 和 Android.bp，这里就是主要实现  
hal 层的核心代码，接下来我们来看 test.c 文件

```
    #define LOG_TAG "TestHal"
     
    #include <malloc.h>
    #include <stdint.h>
    #include <string.h>
     
    #include <log/log.h>
     
    #include <hardware/test.h>
    #include <hardware/hardware.h>
     
    //加法函数实现
    static int additionTest(const struct test_device *dev,int a,int b)
    {
    	if(!dev){
    		return -1;
    	}
        return a + b;
    }
     
    //关闭设备函数
    static int test_close(hw_device_t *dev)
    {
        if (dev) {
            free(dev);
            return 0;
        } else {
            return -1;
        }
    }
    //打开设备函数
    static int test_open(const hw_module_t* module,const char __unused *id,
                                hw_device_t** device)
    {
        if (device == NULL) {
            ALOGE("NULL device on open");
            return -1;
        }
     
        test_device_t *dev = malloc(sizeof(test_device_t));
        memset(dev, 0, sizeof(test_device_t));
     
        dev->common.tag = HARDWARE_DEVICE_TAG;
        dev->common.version = TEST_MODULE_API_VERSION_1_0;
        dev->common.module = (struct hw_module_t*) module;
        dev->common.close = test_close;
        dev->additionTest = additionTest;
     
        *device = &(dev->common);
        return 0;
    }
     
    static struct hw_module_methods_t test_module_methods = {
        .open = test_open,
    };
    //导出符号HAL_MODULE_INFO_SYM，指向自定义模块
    test_module_t HAL_MODULE_INFO_SYM = {
        .common = {
            .tag                = HARDWARE_MODULE_TAG,
            .module_api_version = TEST_MODULE_API_VERSION_1_0,
            .hal_api_version    = HARDWARE_HAL_API_VERSION,
            .id                 = TEST_HARDWARE_MODULE_ID,
            .name               = "Demo Test HAL",
            .author             = "9496044@qq.com",
            .methods            = &test_module_methods,
        },
    };
```

在自定义 hal 层模块中上述的 test.c 中的上述源码中，在自定义的 HAL 的模块中，在  
自定义的 HAL 模块被外部使用需要导出符号 HAL_MODULE_INFO_SYM，导出的这个模块  
结构体最重要的其实就是 tag，id 和 methods，tag 必须定义为 HARDWARE_MODULE_TAG，  
id 就是在 test.h 中定义的模块名，methods 指向 test_module_methods 地址，test_module_methods  
指向结构体 hw_module_methods_t，它里面的 open 函数指针作用是打开模块下的设备，  
它指向 test_open 函数, 上述的代码中，就定义好了 hal 模块，接下来定义下 Android.dp 来  
添加到系统编译中

```
    hardware\libhardware\modules\test\Android.bp
     
    cc_library_shared {
        name: "hello.default",
        relative_install_path: "hw",
        proprietary: true,
        srcs: ["hello.c"],
        header_libs: ["libhardware_headers"],
        shared_libs: ["liblog",
            "libcutils",
            "libutils",],
    }
```

通过上述 Android.bp 的上述源码 编译成功后生成了一个 hello.default.so 共享 lib 库 接下来把他添加到  
系统中 full_base.mk 进行编译，接下来就可以在 framework 层中调用 hal 层接口的相关方法，

```
   diff --git a/build/make/target/product/full_base.mk b/build/make/target/product/full_base.mk
    old mode 100644
    new mode 100755
    index ae1258ead6..e6d210191d
    --- a/build/make/target/product/full_base.mk
    +++ b/build/make/target/product/full_base.mk
     
    #
    # Copyright (C) 2009 The Android Open Source Project
    #
    # Licensed under the Apache License, Version 2.0 (the "License");
    # you may not use this file except in compliance with the License.
    # You may obtain a copy of the License at
    #
    #      http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS,
    # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    # See the License for the specific language governing permissions and
    # limitations under the License.
    #
     
    # This is a build configuration for a full-featured build of the
    # Open-Source part of the tree. It's geared toward a US-centric
    # build of the emulator, but all those aspects can be overridden
    # in inherited configurations.
     
    PRODUCT_PACKAGES := \
        WAPPushManager
     
    PRODUCT_PACKAGES += \
        LiveWallpapersPicker \
        PhotoTable
     
    @@ -30,7 +30,11 @@ PRODUCT_PACKAGES += \
     #   audio.a2dp.default is a system module. Generic system image includes
     #   audio.a2dp.default to support A2DP if board has the capability.
     PRODUCT_PACKAGES += \
    -    audio.a2dp.default
    +    audio.a2dp.default \
    +    hello.default
     
     
    # Additional settings used in all AOSP builds
    PRODUCT_PROPERTY_OVERRIDES := \
        ro.config.ringtone=Ring_Synth_04.ogg \
        ro.config.notification_sound=pixiedust.ogg
     
    # Put en_US first in the list, so make it default.
    PRODUCT_LOCALES := en_US
     
    # Get some sounds
    $(call inherit-product-if-exists, frameworks/base/data/sounds/AllAudio.mk)
     
    # Get a list of languages.
    $(call inherit-product, $(SRC_TARGET_DIR)/product/languages_full.mk)
     
    # Get everything else from the parent package
    $(call inherit-product, $(SRC_TARGET_DIR)/product/generic_no_telephony.mk)
     
    # Add adb keys to debuggable AOSP builds (if they exist)
    $(call inherit-product-if-exists, vendor/google/security/adb/vendor_key.mk)
```

在自定义 hal 层模块中, 到这里我们最简单的 HAL 就已经实现了，这个 HAL 并不设计底层驱动，实际开发中自己可以在设备结构体中定义函数访问底层硬件  
接下来会实现从 native 层通过 HIDL 访问 HAL 的相关功能
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126880227)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 修改系统默认产品设备类型相关代码](#t1)

[3. 修改系统默认产品设备类型相关代码的分析](#t2)

 [3.1 string.xml 中关于不同设备类型显示不同字符的相关代码](#t3)

[3.2 在 devices 下查询设置产品类型的属性](#t4)

[3.3 所以具体修改为:](#t5)

1. 概述
-----

  在产品开发中，对于产品设备类型都默认为 tablet 即平板电脑类型，即 product="tablet" 在一些不是平板的项目中，可能需要修改这个类型为 device 类型 即 product="device", 这就需要找到相关设置系统属性的代码，修改系统属性就可以了

2. 修改系统默认产品设备类型相关代码
-------------------

```
device\rockchip\rk356x\rk3568_x\rk3568_s.mk
packages\apps\Settings\res\values\strings.xml
```

3. 修改系统默认产品设备类型相关代码的分析
----------------------

###   3.1 string.xml 中关于不同设备类型显示不同字符的相关代码

```
    <!-- Preference summary for gesture settings (phone) [CHAR LIMIT=NONE]-->类型为手机
    <string >Quick gestures to control your phone</string>
    <!-- Preference summary for gesture settings (tablet) [CHAR LIMIT=NONE]-->类型为平板电脑
    <string >Quick gestures to control your tablet</string>
    <!-- Preference summary for gesture settings (device) [CHAR LIMIT=NONE]-->类型为设备
    <string >Quick gestures to control your device</string>
    
<!-- Preference and settings suggestion title text for ambient display double tap (phone) [CHAR LIMIT=60]-->
    <string >Double-tap to check phone</string>类型为手机
    <!-- Preference and settings suggestion title text for ambient display double tap (tablet) [CHAR LIMIT=60]-->
    <string >Double-tap to check tablet</string>类型为平板电脑
    <!-- Preference and settings suggestion title text for ambient display double tap (device) [CHAR LIMIT=60]-->
    <string >Double-tap to check device</string>类型为设备
 
    <!-- Preference and settings suggestion title text for ambient display pick up (phone) [CHAR LIMIT=60]-->
    <string >Lift to check phone</string>类型为手机
    <!-- Preference and settings suggestion title text for ambient display pick up (tablet) [CHAR LIMIT=60]-->
    <string >Lift to check tablet</string>类型为平板电脑
    <!-- Preference and settings suggestion title text for ambient display pick up (device) [CHAR LIMIT=60]-->
    <string >Lift to check device</string>类型为设备
 
    <!-- Preference and settings suggestion title text for display wake-up gesture [CHAR LIMIT=60]-->
    <string >Wake up display</string>
 
    <!-- Summary text for ambient display (phone) [CHAR LIMIT=NONE]-->类型为手机
    <string >To check time, notifications, and other info, pick up your phone.</string>
    <!-- Summary text for ambient display (tablet) [CHAR LIMIT=NONE]-->类型为平板电脑
    <string >To check time, notifications, and other info, pick up your tablet.</string>
    <!-- Summary text for ambient display (device) [CHAR LIMIT=NONE]-->类型为设备
    <string >To check time, notifications, and other info, pick up your device.</string>
```

像这些资源中，根据设备类型显示不同的字符串资源，但是需要找到这些产品类型在哪里设置修改为需要的类型就可以了  
从上面的代码可以看出 product 的类型分别为 default 代码是手机类型 tablet 代码是平板电脑类型 device 代表为设备类型  
从查阅相关的资料发现这些类型会在 devices 目录下设置

### 3.2 在 devices 下查询设置产品类型的属性

    每一个平台关于产品属性的设置路径都不同，基本上都是以芯片类型来命名的 如 [rk3568](https://so.csdn.net/so/search?q=rk3568&spm=1001.2101.3001.7020) 就找关于 3568 的产品路径，然后设置 3568 的 mk 文件找到相关的属性来修改就可以了  
比如：rk3568 的一款产品的属性修改

```
#
# Copyright 2014 The Android Open-Source Project
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
 
# First lunching is S, api_level is 31
PRODUCT_SHIPPING_API_LEVEL := 31 --sdk 版本
PRODUCT_DTBO_TEMPLATE := $(LOCAL_PATH)/dt-overlay.in
PRODUCT_SDMMC_DEVICE := fe2b0000.dwmmc
 
include device/rockchip/common/build/rockchip/DynamicPartitions.mk
include device/rockchip/rk356x/rk3568_s/BoardConfig.mk
include device/rockchip/common/BoardConfig.mk
$(call inherit-product, device/rockchip/rk356x/device.mk)
$(call inherit-product, device/rockchip/common/device.mk)
$(call inherit-product, frameworks/native/build/tablet-10in-xhdpi-2048-dalvik-heap.mk)
 
DEVICE_PACKAGE_OVERLAYS += $(LOCAL_PATH)/../overlay
 
PRODUCT_CHARACTERISTICS := tablet --产品类型 就是今天需要修改的地方
 
PRODUCT_NAME := rk3568_s --产品名称
PRODUCT_DEVICE := rk3568_s 
PRODUCT_BRAND := rockchip --产品平台哪家的产品
PRODUCT_MODEL := rk3568_s --产品model
PRODUCT_MANUFACTURER := rockchip --芯片厂家
PRODUCT_AAPT_PREF_CONFIG := mdpi 
#
## add Rockchip properties
#
PRODUCT_PROPERTY_OVERRIDES += ro.sf.lcd_density=320 --设置density 产品密度
PRODUCT_PROPERTY_OVERRIDES += ro.wifi.sleep.power.down=true 
PRODUCT_PROPERTY_OVERRIDES += persist.wifi.sleep.delay.ms=0
PRODUCT_PROPERTY_OVERRIDES += persist.bt.power.down=true
```

PRODUCT_CHARACTERISTICS := tablet -- 产品类型 就是今天需要修改的地方

PRODUCT_NAME := rk3568_s -- 产品名称  
PRODUCT_DEVICE := rk3568_s  
PRODUCT_BRAND := rockchip -- 产品平台哪家的产品  
PRODUCT_MODEL := rk3568_s -- 产品 model  
PRODUCT_MANUFACTURER := rockchip -- 芯片厂家  
PRODUCT_AAPT_PREF_CONFIG := mdpi

设备相关的主要信息

### 3.3 所以具体修改为:

```
#
# Copyright 2014 The Android Open-Source Project
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
 
# First lunching is S, api_level is 31
PRODUCT_SHIPPING_API_LEVEL := 31 --sdk 版本
PRODUCT_DTBO_TEMPLATE := $(LOCAL_PATH)/dt-overlay.in
PRODUCT_SDMMC_DEVICE := fe2b0000.dwmmc
 
include device/rockchip/common/build/rockchip/DynamicPartitions.mk
include device/rockchip/rk356x/rk3568_s/BoardConfig.mk
include device/rockchip/common/BoardConfig.mk
$(call inherit-product, device/rockchip/rk356x/device.mk)
$(call inherit-product, device/rockchip/common/device.mk)
$(call inherit-product, frameworks/native/build/tablet-10in-xhdpi-2048-dalvik-heap.mk)
 
DEVICE_PACKAGE_OVERLAYS += $(LOCAL_PATH)/../overlay
 
- PRODUCT_CHARACTERISTICS := tablet --产品类型 就是今天需要修改的地方
+ PRODUCT_CHARACTERISTICS := device --产品类型 就是今天需要修改的地方修改为device类型
PRODUCT_NAME := rk3568_s --产品名称
PRODUCT_DEVICE := rk3568_s 
PRODUCT_BRAND := rockchip --产品平台哪家的产品
PRODUCT_MODEL := rk3568_s --产品model
PRODUCT_MANUFACTURER := rockchip --芯片厂家
PRODUCT_AAPT_PREF_CONFIG := mdpi 
#
## add Rockchip properties
#
PRODUCT_PROPERTY_OVERRIDES += ro.sf.lcd_density=320 --设置density 产品密度
PRODUCT_PROPERTY_OVERRIDES += ro.wifi.sleep.power.down=true 
PRODUCT_PROPERTY_OVERRIDES += persist.wifi.sleep.delay.ms=0
PRODUCT_PROPERTY_OVERRIDES += persist.bt.power.down=true
```
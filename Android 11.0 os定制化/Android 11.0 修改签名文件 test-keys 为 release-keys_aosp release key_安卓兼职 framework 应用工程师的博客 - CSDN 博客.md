> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124774887)

### 1. 概述

在 android11.0 进行定制化开发设备的时候，在设备上一般会安装安兔兔 来测试机器的性能，所以就难免会被检测出来被 root 过，但是由于一些设备特殊的权限要求，有些机器开发的需求会被 root 掉，  
设备开启 root 权限，但是又不想让用户看到机器被 root 过，所以要从检测的方式看怎么被检测出来的，修改掉就好

### 2. 修改签名文件 test-keys 为 release-keys 的核心类

```
/build/core/config.mk
/build/make/core/Makefile

```

### 3. 修改签名文件 test-keys 为 release-keys 的核心功能实现和分析

关于系统签名的相关问题分析  
在 app 中 系统 app 的签名  
AndroidManifest.xml 中的 android:sharedUserId=“android.uid.system”，代表的意思是和系统相同的 uid，可以拥有修改系统时间，文件操作等权限。  
​ ​android​​的标准签名 key 有：  
testkey  
media  
platform  
shared  
以上的四种，可以在源码的 / build/target/product/security 里面看到对应的密钥，其中 shared.pk8 代表私钥  
系统中的 apk 的 android.mk 中没有设置 LOCAL_CERTIFICATE 的值，就默认使用 testkey。  
而如果设置成：  
LOCAL_CERTIFICATE := platform

而关于系统默认签名 key  
android 配置在 / build/core/config.mk 中定义变量  
接下来看下 config 关于系统签名变量的定义

### 3.1config.mk 关于签名文件的定义

```
ifndef BOARD_SYSTEMSDK_VERSIONS
   ifdef PRODUCT_SHIPPING_API_LEVEL
   ifneq ($(call math_gt_or_eq,$(PRODUCT_SHIPPING_API_LEVEL),28),)
     ifeq (REL,$(PLATFORM_VERSION_CODENAME))
       BOARD_SYSTEMSDK_VERSIONS := $(PLATFORM_SDK_VERSION)
     else
       BOARD_SYSTEMSDK_VERSIONS := $(PLATFORM_VERSION_CODENAME)
     endif
   endif
   endif
 endif
 
 
 ifdef PRODUCT_SHIPPING_API_LEVEL
   ifneq ($(call numbers_less_than,$(PRODUCT_SHIPPING_API_LEVEL),$(BOARD_SYSTEMSDK_VERSIONS)),)
     $(error BOARD_SYSTEMSDK_VERSIONS ($(BOARD_SYSTEMSDK_VERSIONS)) must all be greater than or equal to PRODUCT_SHIPPING_API_LEVEL ($(PRODUCT_SHIPPING_API_LEVEL)))
   endif
   ifneq ($(call math_gt_or_eq,$(PRODUCT_SHIPPING_API_LEVEL),28),)
     ifneq ($(TARGET_IS_64_BIT), true)
       ifneq ($(TARGET_USES_64_BIT_BINDER), true)
         $(error When PRODUCT_SHIPPING_API_LEVEL >= 28, TARGET_USES_64_BIT_BINDER must be true)
       endif
     endif
   endif
   ifneq ($(call math_gt_or_eq,$(PRODUCT_SHIPPING_API_LEVEL),29),)
     ifneq ($(BOARD_OTA_FRAMEWORK_VBMETA_VERSION_OVERRIDE),)
       $(error When PRODUCT_SHIPPING_API_LEVEL >= 29, BOARD_OTA_FRAMEWORK_VBMETA_VERSION_OVERRIDE cannot be set)
     endif
   endif
 endif
 
 # The default key if not set as LOCAL_CERTIFICATE
 ifdef PRODUCT_DEFAULT_DEV_CERTIFICATE
   DEFAULT_SYSTEM_DEV_CERTIFICATE := $(PRODUCT_DEFAULT_DEV_CERTIFICATE)
 else
   DEFAULT_SYSTEM_DEV_CERTIFICATE := build/make/target/product/security/testkey
 endif
 .KATI_READONLY := DEFAULT_SYSTEM_DEV_CERTIFICATE
 
 # Certificate for the NetworkStack sepolicy context
 ifdef PRODUCT_MAINLINE_SEPOLICY_DEV_CERTIFICATES
   MAINLINE_SEPOLICY_DEV_CERTIFICATES := $(PRODUCT_MAINLINE_SEPOLICY_DEV_CERTIFICATES)
 else
   MAINLINE_SEPOLICY_DEV_CERTIFICATES := $(dir $(DEFAULT_SYSTEM_DEV_CERTIFICATE))
 endif
 
 BUILD_NUMBER_FROM_FILE := $$(cat $(SOONG_OUT_DIR)/build_number.txt)
 BUILD_DATETIME_FROM_FILE := $$(cat $(BUILD_DATETIME_FILE))

```

从上述代码可以看出

```
 # The default key if not set as LOCAL_CERTIFICATE
 ifdef PRODUCT_DEFAULT_DEV_CERTIFICATE
   DEFAULT_SYSTEM_DEV_CERTIFICATE := $(PRODUCT_DEFAULT_DEV_CERTIFICATE)
 else
   DEFAULT_SYSTEM_DEV_CERTIFICATE := build/make/target/product/security/testkey
 endif
 .KATI_READONLY := DEFAULT_SYSTEM_DEV_CERTIFICATE

```

PRODUCT_DEFAULT_DEV_CERTIFICATE 就是默认的系统签名变量  
PRODUCT_DEFAULT_DEV_CERTIFICATE 变量在系统编译 [Makefile](https://so.csdn.net/so/search?q=Makefile&spm=1001.2101.3001.7020) 根据编译类型定义系统签名  
接下来在 Makefile 中看关于系统签名的相关代码分析

### 3.2 Makefile 的系统签名分析

```
$(INSTALLED_DEFAULT_PROP_TARGET): $(BUILDINFO_COMMON_SH) $(POST_PROCESS_PROPS) $(intermediate_system_build_prop)
 	@echo Target buildinfo: $@
 	@mkdir -p $(dir $@)
 	@rm -f $@
 	$(hide) echo "#" > $@; \
 	        echo "# ADDITIONAL_DEFAULT_PROPERTIES" >> $@; \
 	        echo "#" >> $@;
 	$(hide) $(foreach line,$(FINAL_DEFAULT_PROPERTIES), \
 	    echo "$(line)" >> $@;)
 	$(hide) $(POST_PROCESS_PROPS) $@
 ifdef property_overrides_split_enabled
 	$(hide) mkdir -p $(TARGET_ROOT_OUT)
 	$(hide) ln -sf system/etc/prop.default $(INSTALLED_DEFAULT_PROP_OLD_TARGET)
 endif
 
 # -----------------------------------------------------------------
 # vendor default.prop
 INSTALLED_VENDOR_DEFAULT_PROP_TARGET :=
 ifdef property_overrides_split_enabled
 INSTALLED_VENDOR_DEFAULT_PROP_TARGET := $(TARGET_OUT_VENDOR)/default.prop
 ALL_DEFAULT_INSTALLED_MODULES += $(INSTALLED_VENDOR_DEFAULT_PROP_TARGET)
 
 $(INSTALLED_VENDOR_DEFAULT_PROP_TARGET): $(INSTALLED_DEFAULT_PROP_TARGET) $(POST_PROCESS_PROPS)
 	@echo Target buildinfo: $@
 	@mkdir -p $(dir $@)
 	$(hide) echo "#" > $@; \
 	        echo "# ADDITIONAL VENDOR DEFAULT PROPERTIES" >> $@; \
 	        echo "#" >> $@;
 	$(hide) $(foreach line,$(FINAL_VENDOR_DEFAULT_PROPERTIES), \
 	    echo "$(line)" >> $@;)
 	$(hide) $(POST_PROCESS_PROPS) $@
 
 endif  # property_overrides_split_enabled
 
 # -----------------------------------------------------------------
 # build.prop
 INSTALLED_BUILD_PROP_TARGET := $(TARGET_OUT)/build.prop
 ALL_DEFAULT_INSTALLED_MODULES += $(INSTALLED_BUILD_PROP_TARGET)
 FINAL_BUILD_PROPERTIES := \
     $(call collapse-pairs, $(ADDITIONAL_BUILD_PROPERTIES))
 FINAL_BUILD_PROPERTIES := $(call uniq-pairs-by-first-component, \
     $(FINAL_BUILD_PROPERTIES),=)
 
 # A list of arbitrary tags describing the build configuration.
 # Force ":=" so we can use +=
 BUILD_VERSION_TAGS := $(BUILD_VERSION_TAGS)
 ifeq ($(TARGET_BUILD_TYPE),debug)
   BUILD_VERSION_TAGS += debug
 endif
 # The "test-keys" tag marks builds signed with the old test keys,
 # which are available in the SDK.  "dev-keys" marks builds signed with
 # non-default dev keys (usually private keys from a vendor directory).
 # Both of these tags will be removed and replaced with "release-keys"
 # when the target-files is signed in a post-build step.
 ifeq ($(DEFAULT_SYSTEM_DEV_CERTIFICATE),build/make/target/product/security/testkey)
 BUILD_KEYS := test-keys
 else
 BUILD_KEYS := dev-keys
 endif
 BUILD_VERSION_TAGS += $(BUILD_KEYS)
 BUILD_VERSION_TAGS := $(subst $(space),$(comma),$(sort $(BUILD_VERSION_TAGS)))
 
 # A human-readable string that descibes this build in detail.
 build_desc := $(TARGET_PRODUCT)-$(TARGET_BUILD_VARIANT) $(PLATFORM_VERSION) $(BUILD_ID) $(BUILD_NUMBER_FROM_FILE) $(BUILD_VERSION_TAGS)
 $(intermediate_system_build_prop): PRIVATE_BUILD_DESC := $(build_desc)
 
 # The string used to uniquely identify the combined build and product; used by the OTA server.
 ifeq (,$(strip $(BUILD_FINGERPRINT)))
   ifeq ($(strip $(HAS_BUILD_NUMBER)),false)
     BF_BUILD_NUMBER := $(BUILD_USERNAME)$$($(DATE_FROM_FILE) +%m%d%H%M)
   else
     BF_BUILD_NUMBER := $(file <$(BUILD_NUMBER_FILE))
   endif
   BUILD_FINGERPRINT := $(PRODUCT_BRAND)/$(TARGET_PRODUCT)/$(TARGET_DEVICE):$(PLATFORM_VERSION)/$(BUILD_ID)/$(BF_BUILD_NUMBER):$(TARGET_BUILD_VARIANT)/$(BUILD_VERSION_TAGS)
 endif
 # unset it for safety.
 BF_BUILD_NUMBER :=
 
 BUILD_FINGERPRINT_FILE := $(PRODUCT_OUT)/build_fingerprint.txt
 ifneq (,$(shell mkdir -p $(PRODUCT_OUT) && echo $(BUILD_FINGERPRINT) >$(BUILD_FINGERPRINT_FILE) && grep " " $(BUILD_FINGERPRINT_FILE)))
   $(error BUILD_FINGERPRINT cannot contain spaces: "$(file <$(BUILD_FINGERPRINT_FILE))")
 endif
 BUILD_FINGERPRINT_FROM_FILE := $$(cat $(BUILD_FINGERPRINT_FILE))
 # unset it for safety.
 BUILD_FINGERPRINT :=
 
 # The string used to uniquely identify the system build; used by the OTA server.
 # This purposefully excludes any product-specific variables.
 ifeq (,$(strip $(BUILD_THUMBPRINT)))
   BUILD_THUMBPRINT := $(PLATFORM_VERSION)/$(BUILD_ID)/$(BUILD_NUMBER_FROM_FILE):$(TARGET_BUILD_VARIANT)/$(BUILD_VERSION_TAGS)
 endif

```

在 Makefile 中的 DEFAULT_SYSTEM_DEV_CERTIFICATE 编译系统签名的相关代码发现

```
/build/make/core/Makefile

 ifeq ($(DEFAULT_SYSTEM_DEV_CERTIFICATE),build/make/target/product/security/testkey)
 BUILD_KEYS := test-keys
 else
 BUILD_KEYS := dev-keys
 endif

BUILD_VERSION_TAGS += $(BUILD_KEYS)
BUILD_VERSION_TAGS := $(subst $(space),$(comma),$(sort $(BUILD_VERSION_TAGS)))

```

确保系统签名始终为 release-keys 这样安兔兔在检测系统时，就不会出现 root 的字样  
具体修改如下：

```
 ifeq ($(DEFAULT_SYSTEM_DEV_CERTIFICATE),build/make/target/product/security/testkey)
 BUILD_KEYS := release-keys
 else
 BUILD_KEYS := release-keys
 endif

```
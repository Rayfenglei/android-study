> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124806838)

### 1. 概述

在 11.0 12.0 定制化开发中，由于需要新增加自定义的功能，所以要增加自定义服务，上层通过调用服务，来调用相应的功能，所以系统需要先生成 jar, 然后生成 jar 给上层 app 调用  
从而来实现所需要的功能

### 第一步：

添加自定义服务

1. 创建 aidl  
2. 在 frameworks\base\Android.bp 中添加我们的 AIDL，让其编译进系统  
3. 在 frameworks\base\services\core\java\com\android\server \ 下创建自己的文件夹 lgy，并创建自己的 service  
4. 在 frameworks\base\services\java\com\android\server\SystemServer.java 中启动我们的服务  
5. 添加给应用层调用的接口  
6.frameworks\base\core\java\android\content\Context.java 添加  
7.frameworks\base\core\java\android\app\SystemServiceRegistry.java 注册服务  
8. 新增自定义类 调用服务，然后提供给上层调用该类的接口 (这一步也可以省略)  
9. 新增的 service 配置 selinux 策略

这几步就完成了自定义服务  
具体实现 请看我的另一篇博客 ：https://blog.csdn.net/baidu_41666295/article/details/117959553?spm=1001.2014.3001.5502

### 第二步就是生成 jar 提供给 app 调用

MAKEFILE 的生成的顺序来说明下吧。  
首先在 / frameworks/base/Android.[mk](https://so.csdn.net/so/search?q=mk&spm=1001.2101.3001.7020) 中定义了进行 sdk building 的基本目标对象。  
包括对哪些. java 文件需要生成 API 文档，以及这些文档的路径。  
然后在 / build/core/droiddoc.mk 中定义了最终进行 build 的规则和语句。

Android 使用 [javadoc](https://so.csdn.net/so/search?q=javadoc&spm=1001.2101.3001.7020) 这个工具来生成所有 API 文档。  
Javadoc 这个工具可以带一个参数指定一个文件，该文件包含了所有要生成文档的源文件的名字（全路径）。  
该文件的内容就是通过在 / framework/base/android.mk 里的变量生成的。当然在 droiddoc.mk 中还添加了 build 过程中生成的 intermediates 目录下的文件。

另外 javadoc 还可以指定定制的 doclet（doclet 是基于 javadoc 特定的 API 开发的小程序，该程序负责实际的文档输出).android 的编译系统就包含了这样一个 doclet 叫 DroidDoc。可以在 / build/tools/DroidDoc 目录下找到该工具的全部源代码。

正是该工具在生成 HTML 的同时在 / out/target/common/obj/JAVA_LIBRARIES/android_stubs_current_intermediates 下面 copy（或者说重新生成了）所有将生成到 android.jar 中的所有源代码（.java 文件）.  
该工具把所有生成 document 的源文件重新按 Package 组织生成在以上目录下。  
然后进行编译和打包成 android.jar。  
根据以上分析，其实 android.jar 文件是各个公布出来的 API 的源文件经过 javadoc 重新组织以后再次编译产生的。 故，android.jar 的内容实际上受到 javadoc 的 notation 控制和 makefile 的控制。 对于 android 中已存在的代码比如 wifi native，可以通过修改源代码中 javadoc 的 notation 的方法重新 build 得到新的包含 wifi native 接口的 android.jar（将源文件中的 @hide 这个 notation 换成别的，然后 make update-api;make  
sdk)。而对于新加入的代码，则需要如上方法来修改 makefile 了。  
下面总结一下：

1.  javadoc 和 doclet，简单的看了一下工具的使用和参数，另外看了一下 DriodDoc 这个 doclet 的源代码，找出哪里生成的. java 源文件。  
    2.makefile 分析，android 的 make showcommands 命令可以和任何其他目标一起使用来察看 make 过程中实际做了一些什么事情。（这点还需要调查这个 showcommands 如何实现的，因为 make -d 这个命令给出的信息对于找到问题帮助不大）  
    3. 在跟踪 makefile build 过程时，使用 ( w a r n i n g x x x x x ) 和 (warning xxxxx) 和 (warningxxxxx) 和 (error xxxx) 可以在除规则以外的地方打印出变量的值通过这个方法找出了实际建立要编译的文件列表的地方

当中 android 根目录下敲击 make 时候，根目录下的 Makefile 就一句话 include build/core/main.mk，即调用 main.mk，以下为 main.mk 的依赖规则

```
ifndef KATI
$(warning Calling make directly is no longer supported.)
$(warning Either use 'envsetup.sh; m' or 'build/soong/soong_ui.bash --make-mode')
$(error done)
endif

$(info [1/1] initializing build system ...)

# Absolute path of the present working direcotry.
 # This overrides the shell variable $PWD, which does not necessarily points to
 # the top of the source tree, for example when "make -C" is used in m/mm/mmm.
 PWD := $(shell pwd)
 
 # This is the default target.  It must be the first declared target.
 .PHONY: droid
 DEFAULT_GOAL := droid
 $(DEFAULT_GOAL): droid_targets
 
 .PHONY: droid_targets
 droid_targets:
 
 # Set up various standard variables based on configuration
 # and host information.
 include build/make/core/config.mk
 
 ifneq ($(filter $(dont_bother_goals), $(MAKECMDGOALS)),)
 dont_bother := true
 endif
 
 .KATI_READONLY := SOONG_CONFIG_NAMESPACES
 .KATI_READONLY := $(foreach n,$(SOONG_CONFIG_NAMESPACES),SOONG_CONFIG_$(n))
 .KATI_READONLY := $(foreach n,$(SOONG_CONFIG_NAMESPACES),$(foreach k,$(SOONG_CONFIG_$(n)),SOONG_CONFIG_$(n)_$(k)))
 
 include $(SOONG_MAKEVARS_MK)
 
 YACC :=$= $(BISON) -d
 
 include $(BUILD_SYSTEM)/clang/config.mk
 
 # Write the build number to a file so it can be read back in
 # without changing the command line every time.  Avoids rebuilds
 # when using ninja.
 $(shell mkdir -p $(SOONG_OUT_DIR) && \
     echo -n $(BUILD_NUMBER) > $(SOONG_OUT_DIR)/build_number.tmp; \
     if ! cmp -s $(SOONG_OUT_DIR)/build_number.tmp $(SOONG_OUT_DIR)/build_number.txt; then \
         mv $(SOONG_OUT_DIR)/build_number.tmp $(SOONG_OUT_DIR)/build_number.txt; \
     else \
         rm $(SOONG_OUT_DIR)/build_number.tmp; \
     fi)
 BUILD_NUMBER_FILE := $(SOONG_OUT_DIR)/build_number.txt
 .KATI_READONLY := BUILD_NUMBER_FILE
 $(KATI_obsolete_var BUILD_NUMBER,See https://android.googlesource.com/platform/build/+/master/Changes.md#BUILD_NUMBER)
 $(BUILD_NUMBER_FILE):
 	touch $@
 
 DATE_FROM_FILE := date -d @$(BUILD_DATETIME_FROM_FILE)
 .KATI_READONLY := DATE_FROM_FILE
 
 # Pick a reasonable string to use to identify files.
 ifeq ($(strip $(HAS_BUILD_NUMBER)),false)
   # BUILD_NUMBER has a timestamp in it, which means that
   # it will change every time.  Pick a stable value.
   FILE_NAME_TAG := eng.$(BUILD_USERNAME)
 else
   FILE_NAME_TAG := $(file <$(BUILD_NUMBER_FILE))
 endif
 .KATI_READONLY := FILE_NAME_TAG
 
 # Make an empty directory, which can be used to make empty jars
 EMPTY_DIRECTORY := $(OUT_DIR)/empty
 $(shell mkdir -p $(EMPTY_DIRECTORY) && rm -rf $(EMPTY_DIRECTORY)/*)
 
 # CTS-specific config.
 -include cts/build/config.mk
 # VTS-specific config.
 -include test/vts/tools/vts-tradefed/build/config.mk
 # device-tests-specific-config.
 -include tools/tradefederation/build/suites/device-tests/config.mk
 # general-tests-specific-config.
 -include tools/tradefederation/build/suites/general-tests/config.mk
 # STS-specific config.
 -include test/sts/tools/sts-tradefed/build/config.mk
 # CTS-Instant-specific config
 -include test/suite_harness/tools/cts-instant-tradefed/build/config.mk
 # MTS-specific config.
 -include test/mts/tools/build/config.mk
 # VTS-Core-specific config.
 -include test/vts/tools/vts-core-tradefed/build/config.mk
 # CSUITE-specific config.
 -include test/app_compat/csuite/tools/build/config.mk
 # CATBox-specific config.
 -include test/catbox/tools/build/config.mk
 # CTS-Root-specific config.
 -include test/cts-root/tools/build/config.mk

```

这就是 Makefile 编译流程  
在工作中  
添加完自定义 service 后 编译 编译完成后来生成 jar 包

导出我们需要的 jar 包

在 out/target/common/obj/JAVA_LIBRARIES 目录下找到 framework-minus-apex_intermediates 目录, 11 之前的好像是 framework_intermediates 目录 现在改成了这个 framework-minus-apex_intermediates  
使用下面的命令生成 jar

```
/work/sprdroid10_trunk_19c/out/target/common/obj/JAVA_LIBRARIES/framework_intermediates$  jar -xvf classes.jar android/mdm/MyPower.class
 inflated: android/mdm/MyPower.class
/work/sprdroid10_trunk_19c/out/target/common/obj/JAVA_LIBRARIES/framework_intermediates$ jar -cvf mypower.jar android           
added manifest
adding: android/(in = 0) (out= 0)(stored 0%)
adding: android/mdm/(in = 0) (out= 0)(stored 0%)
adding: android/mdm/MyPower.class(in = 9093) (out= 2774)(deflated 69%)
/work/sprdroid10_trunk_19c/out/target/common/obj/JAVA_LIBRARIES/framework_intermediates$

```

首选 cd 到 out/target/common/obj/JAVA_LIBRARIES/framework_intermediates 下  
执行命令 jar -xvf classes.jar android/mdm/MyPower.class MyPower.class 就是要提供给上层调用的类的 classes 文件  
执行命令 jar -cvf mypower.jar android 把 MyPower.class 文件生成 mypower.jar 的 jar 包给上层调用

生成完 jar 后 在 out/target/common/obj/JAVA_LIBRARIES/framework_intermediates 目录下会看到生成的 jar 文件  
和 android/mdm/ 下的 class 文件  
然后 将 jar 导入到 app 里面 调用相应的功能 发现已经能正常调用自定义的 api 接口
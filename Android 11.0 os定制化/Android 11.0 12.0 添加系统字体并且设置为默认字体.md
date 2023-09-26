> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124830802)

### 1. 概述

在 11.0 12.0 系统定制化开发中，在产品定制中，有产品需求对于系统字体风格不太满意，所以想要更换系统的默认字体，对于系统字体的修改也是常有的功能，而系统默认也支持增加字体，所以就来添加楷体字体为系统字体，并替换为系统默认字体

### 2. 添加系统字体并且设置为默认字体的核心类

```
frameworks/base/data/fonts/
frameworks/base/data/fonts/fonts.mk
frameworks/base/data/fonts/Android.mk
frameworks/base/data/fonts/fonts.xml 

```

### 3. 添加系统字体并且设置为默认字体核心功能实现和分析

对于系统添加新字体功能，是默认支持的但是有些字体会导致系统的支持性不是太好，所以  
要选择好系统字体也是比较关键的  
具体步骤如下：

### 3.1fonts 下增加新字体

在目录 frameworks/base/data/fonts/ 添加 KTFont.[ttf](https://so.csdn.net/so/search?q=ttf&spm=1001.2101.3001.7020)

### 3.2 在 frameworks/base/data/fonts/fonts.[mk](https://so.csdn.net/so/search?q=mk&spm=1001.2101.3001.7020) 中添加新的字体

具体先看 fonts.mk 文件

```
# Copyright (C) 2008 The Android Open Source Project
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
 
 # Warning: this is actually a product definition, to be inherited from
 
 PRODUCT_PACKAGES := \
     DroidSansMono.ttf \
     AndroidClock.ttf \
     fonts.xml

```

增加新字体如下

```
PRODUCT_PACKAGES := \
    DroidSansMono.ttf \
    AndroidClock.ttf \
+	KTFont.ttf \
    fonts.xml

```

### 3.3 在 frameworks/base/data/fonts/Android.mk 中添加新的字体

在 font_src_files 添加新增的字体格式，如下

```
 ################################
 # Build the rest of font files as prebuilt.
 
 # $(1): The source file name in LOCAL_PATH.
 #       It also serves as the module name and the dest file name.
 define build-one-font-module
 $(eval include $(CLEAR_VARS))\
 $(eval LOCAL_MODULE := $(1))\
 $(eval LOCAL_SRC_FILES := $(1))\
 $(eval LOCAL_MODULE_CLASS := ETC)\
 $(eval LOCAL_MODULE_TAGS := optional)\
 $(eval LOCAL_MODULE_PATH := $(TARGET_OUT)/fonts)\
 $(eval include $(BUILD_PREBUILT))
 endef
font_src_files := \
    AndroidClock.ttf \
	KTFont.ttf

```

### 3.4fonts.xml 配置默认字体作为默认系统字体

frameworks/base/data/fonts/fonts.xml 中替换默认字体  
先来看下 fonts.xml

```
<?xml version="1.0" encoding="utf-8"?>
<!--
    WARNING: Parsing of this file by third-party apps is not supported. The
    file, and the font files it refers to, will be renamed and/or moved out
    from their respective location in the next Android release, and/or the
    format or syntax of the file may change significantly. If you parse this
    file for information about system fonts, do it at your own risk. Your
    application will almost certainly break with the next major Android
    release.

    In this file, all fonts without names are added to the default list.
    Fonts are chosen based on a match: full BCP-47 language tag including
    script, then just language, and finally order (the first font containing
    the glyph).

    Order of appearance is also the tiebreaker for weight matching. This is
    the reason why the 900 weights of Roboto precede the 700 weights - we
    prefer the former when an 800 weight is requested. Since bold spans
    effectively add 300 to the weight, this ensures that 900 is the bold
    paired with the 500 weight, ensuring adequate contrast.
-->
<familyset version="23">
    <!-- first font is default -->
    <family >
        <font weight="100" style="normal">Roboto-Thin.ttf</font>
        <font weight="100" style="italic">Roboto-ThinItalic.ttf</font>
        <font weight="300" style="normal">Roboto-Light.ttf</font>
        <font weight="300" style="italic">Roboto-LightItalic.ttf</font>
        <font weight="400" style="normal">Roboto-Regular.ttf</font>
        <font weight="400" style="italic">Roboto-Italic.ttf</font>
        <font weight="500" style="normal">Roboto-Medium.ttf</font>
        <font weight="500" style="italic">Roboto-MediumItalic.ttf</font>
        <font weight="900" style="normal">Roboto-Black.ttf</font>
        <font weight="900" style="italic">Roboto-BlackItalic.ttf</font>
        <font weight="700" style="normal">Roboto-Bold.ttf</font>
        <font weight="700" style="italic">Roboto-BoldItalic.ttf</font>
    </family>

    <!-- Note that aliases must come after the fonts they reference. -->
    <alias  />
    <alias  />
    <alias  />
    <alias  />
    <alias  />
    <alias  />
    <alias  />
    <alias  />

    <family >
        <font weight="300" style="normal">RobotoCondensed-Light.ttf</font>
        <font weight="300" style="italic">RobotoCondensed-LightItalic.ttf</font>
        <font weight="400" style="normal">RobotoCondensed-Regular.ttf</font>
        <font weight="400" style="italic">RobotoCondensed-Italic.ttf</font>
        <font weight="500" style="normal">RobotoCondensed-Medium.ttf</font>
        <font weight="500" style="italic">RobotoCondensed-MediumItalic.ttf</font>
        <font weight="700" style="normal">RobotoCondensed-Bold.ttf</font>
        <font weight="700" style="italic">RobotoCondensed-BoldItalic.ttf</font>
    </family>
 。。。。。

```

通过阅读代码发现第一个 family 就是默认的字体 KTFont.ttf 所以要把添加在一条就行了  
所以修改如下:

```
 <familyset version="23">
     <!-- first font is default -->
     <family >
+           <font weight="400" style="normal">KTFont.ttf</font>
         <font weight="100" style="normal">Roboto-Thin.ttf</font>
         <font weight="100" style="italic">Roboto-ThinItalic.ttf</font>
         <font weight="300" style="normal">Roboto-Light.ttf</font>
         <font weight="300" style="italic">Roboto-LightItalic.ttf</font>
-        <font weight="400" style="normal">Roboto-Regular.ttf</font>
+        <!--font weight="400" style="normal">Roboto-Regular.ttf</font-->
         <font weight="400" style="italic">Roboto-Italic.ttf</font>
         <font weight="500" style="normal">Roboto-Medium.ttf</font>
         <font weight="500" style="italic">Roboto-MediumItalic.ttf</font>
@@ -545,10 +546,7 @@
     <family>

```

同时注意要注释掉相同属性的 Roboto-Regular.ttf 的字体 不然系统会抛异常开不了机  
在加载系统默认字体的时候 weight 400 的字体已经存在了 所以要注释掉一个即可
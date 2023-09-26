> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124807165)

### 1. 概述

在 11.0 12.0 系统的产品开发中，产品觉得系统原生自带的 Launcher3 中的 workspace app 列表页的白色字体不太好看，  
想更换好点的字体样式，所以要求更换字体颜色，这就需要知道相关的字体样式，修改样式就可以了

### 2.Launcher3 修改 workspace 字体颜色的核心类

```
packages\apps\Launcher3\AndroidManifest.xml
packages\apps\Launcher3\res\values\style.xml

```

### 3.Launcher3 修改 workspace 字体颜色的功能分析和实现

### 3.1 首选看 Launcher3 的 AndroidManifest.xml 了

看下 Launcher3 的 application 的设置布局样式  
路径: packages\apps\Launcher3\AndroidManifest.xml

```
** Copyright 2008, The Android Open Source Project
**
** Licensed under the Apache License, Version 2.0 (the "License");
** you may not use this file except in compliance with the License.
** You may obtain a copy of the License at
 **
 **     http://www.apache.org/licenses/LICENSE-2.0
 **
 ** Unless required by applicable law or agreed to in writing, software
 ** distributed under the License is distributed on an "AS IS" BASIS,
 ** WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 ** See the License for the specific language governing permissions and
 ** limitations under the License.
 */
 -->
 <manifest
     xmlns:android="http://schemas.android.com/apk/res/android"
     package="com.android.launcher3">
     <uses-sdk android:targetSdkVersion="29" android:minSdkVersion="25"/>
     <!--
     Manifest entries specific to Launcher3. This is merged with AndroidManifest-common.xml.
     Refer comments around specific entries on how to extend individual components.
     -->
 
     <application
         android:backupAgent="com.android.launcher3.LauncherBackupAgent"
         android:fullBackupOnly="true"
         android:fullBackupContent="@xml/backupscheme"
         android:hardwareAccelerated="true"
         android:icon="@drawable/ic_launcher_home"
         android:label="@string/derived_app_name"
         android:theme="@style/AppTheme"
         android:largeHeap="@bool/config_largeHeap"
         android:restoreAnyVersion="true"
         android:supportsRtl="true" >
 
         <!--
         Main launcher activity. When extending only change the name, and keep all the
         attributes and intent filters the same
         -->
         <activity
             android:
             android:launchMode="singleTask"
             android:clearTaskOnLaunch="true"
             android:stateNotNeeded="true"
             android:windowSoftInputMode="adjustPan"
             android:screenOrientation="unspecified"
             android:configChanges="keyboard|keyboardHidden|mcc|mnc|navigation|orientation|screenSize|screenLayout|smallestScreenSize|uiMode"
             android:resizeableActivity="true"
             android:resumeWhilePausing="true"
             android:taskAffinity=""
             android:enabled="true">
             <intent-filter>
                 <action android: />
                 <category android: />
                 <category android: />
                 <category android:/>
                 <category android: />
             </intent-filter>
             <meta-data
                 android:
                 android:value="${packageName}.grid_control" />
         </activity>
 
     </application>
 </manifest>

```

在 AndroidManifest.xml 中 application 中的属性可以看到用的样式是 android:theme=“@style/AppTheme”  
接下来看下 style.xml 中的 AppTheme 样式了来分析当前字体的样式中的颜色，然后适配合适的字体颜色

### 3.2style.xml 中关于字体的颜色分析

下面来查看 style.xml 的相关字体样式  
路径为: packages\apps\Launcher3\res\values\style.xml

```
    <!-- Launcher theme -->
    <style @android:style/Theme.DeviceDefault.Light">
        <item >@null</item>
        <item >#FF757575</item>
        <item >false</item>
        <item >@android:color/transparent</item>
        <item >true</item>
        <item >true</item>
        <item >?attr/workspaceTextColor</item>
    </style>

    <style @style/BaseLauncherTheme">
        <item >#DE000000</item>
        <item >#FFFFFFFF</item>
        <item >46</item>
        <item >#66FFFFFF</item>
        <item >@style/AllAppsTheme</item>
        <item >#FFF</item>
        <item >#F1F3F4</item>
        <item >#E0E0E0</item> <!-- Gray 300 -->
        <item >false</item>
        <item >false</item>
        <item >@android:color/white</item>
        <item >#B0000000</item>
        <item >#33000000</item>
        <item >#44000000</item>
        <item >@drawable/workspace_bg</item>
        <item >@style/WidgetContainerTheme</item>
        <item >?android:attr/colorPrimary</item>
        <item >#CDFFFFFF</item>
        <item >?android:attr/colorPrimary</item>
        <item >#FF212121</item>
        <item >#89616161</item>
        <item >#CCFFFFFF</item>
        <item >?android:attr/textColorSecondary</item>
        <item >#FF212121</item>
        <item >?android:attr/colorAccent</item>
        <item >.54</item>

        <item >false</item>
        <item >false</item>
        <item >true</item>
        <item >#00000000</item>
        <item >#00000000</item>


    </style>

    <style @style/LauncherTheme">
        <item >#FF3C4043</item> <!-- 100% GM2 800 -->
        <item >?attr/workspaceTextColor</item>
        <item >.254</item>

    </style>

    <style @style/LauncherTheme">
        <item >#FF212121</item>
        <item >128</item>
        <item >@android:color/transparent</item>
        <item >@android:color/transparent</item>
        <item >@android:color/transparent</item>
        <item >true</item>
        <item >@null</item>
        <item >#FF464646</item>
        <item >#CDFFFFFF</item>
        <item >#FF80868B</item>
        <item >?attr/workspaceTextColor</item>
    </style>

    <style @style/LauncherTheme">
        <item >#FFFFFFFF</item>
        <item >#FFFFFFFF</item>
        <item >#CCFFFFFF</item>
        <item >#A0FFFFFF</item>
        <item >#A0FFFFFF</item>
        <item >#FF212121</item>
        <item >#FF000000</item>
        <item >102</item>
        <item >#80000000</item>
         <item >@style/AllAppsTheme.Dark</item>
         <item >#3C4043</item> <!-- Gray 800 -->
         <item >#202124</item>
         <item >#757575</item> <!-- Gray 600 -->
         <item >@style/WidgetContainerTheme.Dark</item>
         <item >#FF464646</item>
         <item >#DD3C4043</item> <!-- 87% GM2 800 -->
         <item >#FF80868B</item>
         <item >@android:color/white</item>
         <item >#89CCCCCC</item>
         <item >true</item>
         <item >#99FFFFFF</item>
         <item >#B3FFFFFF</item>
         <item >@android:color/white</item>
         <item >#DD000000</item>
     </style>
 
     <!-- A derivative project can extend these themes to customize the application theme without
          affecting the base theme -->
     <style @style/LauncherTheme" />

```

从样式中可以看出 AppTheme 的字体样式是继承自 LauncherTheme 的，所以在 LauncherTheme 中看到

```
 <item >#FFA07A</item>

```

即为字体样式颜色  
所以修改颜色值即可修改 workspace app 的字体颜色  
修改完后编译 Launcher3QuickStep 替换 apk 就可以了
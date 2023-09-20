> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124704944)

### 1. 概述

在 11.0 12.0 的系统产品开发中，需要自定义开机向导 app 页面，而系统[源码](https://so.csdn.net/so/search?q=%E6%BA%90%E7%A0%81&spm=1001.2101.3001.7020)中只提供了 Provision 作为开机向导 app 有些平台没有把它编译到源码中 作为开机向导，所以自定义开机向导，可以在这里定制自己的 UI 就可以了

### 2. 自定义开机向导 app 的核心类

```
 /packages/apps/Provision/AndroidManifest.xml
 packages/apps/Provision/src/com/android/provision/DefaultActivity.java

```

### 3. 自定义开机向导 app 的核心功能分析和实现

关于开机向导 app 的核心标志，都是在 AndroidManifest.[xml](https://so.csdn.net/so/search?q=xml&spm=1001.2101.3001.7020) 中体现的，可以先看相关源码然后分析  
首选看下 AndroidManifest.xml

### 3.1AndroidManifest.xml 关于开机向导的相关分析

```
<!--
 Copyright (C) 2008 The Android Open Source Project

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0
 
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
  -->
 <manifest xmlns:android="http://schemas.android.com/apk/res/android"
         package="com.android.provision">
 
     <original-package android: />
 
     <!-- For miscellaneous settings -->
     <uses-permission android: />
     <uses-permission android: />
 
     <application>
         <activity android:
                 android:excludeFromRecents="true">
             <intent-filter android:priority="1">
                 <action android: />
                 <category android: />
                 <category android: />
                 <category android: />
             </intent-filter>
         </activity>
     </application>
 </manifest>
 

```

在 application 的 intent-filter 的标签中其中  
作为  
开机向导的标志 添加了这个标志的 app 页面首次开机会进入开机向导页面，就可以作为开机向导页面, 而具体实现开机向导的一些功能是在 DefaultActivity.java 类中  
接下来看下 DefaultActivity.java 代码

### 3.2DefaultActivity.java 开机向导的功能分析和实现

```
package com.android.provision;
  
  import android.app.Activity;
  import android.content.ComponentName;
  import android.content.pm.PackageManager;
  import android.os.Bundle;
  import android.provider.Settings;
  
  /**
   * Application that sets the provisioned bit, like SetupWizard does.
   */
  public class DefaultActivity extends Activity {
  
      @Override
      protected void onCreate(Bundle icicle) {
          super.onCreate(icicle);
 
          // Add a persistent setting to allow other apps to know the device has been provisioned.
          Settings.Global.putInt(getContentResolver(), Settings.Global.DEVICE_PROVISIONED, 1);
          Settings.Secure.putInt(getContentResolver(), Settings.Secure.USER_SETUP_COMPLETE, 1);
  
          // remove this activity from the package manager.
          PackageManager pm = getPackageManager();
          ComponentName name = new ComponentName(this, DefaultActivity.class);
          pm.setComponentEnabledSetting(name, PackageManager.COMPONENT_ENABLED_STATE_DISABLED,
                  PackageManager.DONT_KILL_APP);
  
          // terminate the activity.
          finish();
      }
  }

```

通过上述代码可以看出，在 onCreate(Bundle icicle) 后，设置了相关的属性后  
可以看出 走到这里后 finish 掉当前页面直接跳过了 进入开机 Launcher 界面

如果需要定制 可以把上述代码 抽调成一个方法 setProvision()

```
setProvision(){
     Settings.Global.putInt(getContentResolver(), Settings.Global.DEVICE_PROVISIONED, 1);
     Settings.Secure.putInt(getContentResolver(), Settings.Secure.USER_SETUP_COMPLETE, 1);
  
          // remove this activity from the package manager.
          PackageManager pm = getPackageManager();
          ComponentName name = new ComponentName(this, DefaultActivity.class);
          pm.setComponentEnabledSetting(name, PackageManager.COMPONENT_ENABLED_STATE_DISABLED,
                  PackageManager.DONT_KILL_APP);
  
          // terminate the activity.
          finish();
}

```

初次进入 onCreate() 展示自定义的页面 等所以需要展示的内容展示完以后 最后一步时调用 setProvision()  
就可以了  
具体修改如下

```
  import android.app.Activity;
  import android.content.ComponentName;
  import android.content.pm.PackageManager;
  import android.os.Bundle;
  import android.provider.Settings;
  
  /**
   * Application that sets the provisioned bit, like SetupWizard does.
   */
  public class DefaultActivity extends Activity {
  
      @Override
      protected void onCreate(Bundle icicle) {
          super.onCreate(icicle);
 
          // Add a persistent setting to allow other apps to know the device has been provisioned.
          Settings.Global.putInt(getContentResolver(), Settings.Global.DEVICE_PROVISIONED, 1);
          Settings.Secure.putInt(getContentResolver(), Settings.Secure.USER_SETUP_COMPLETE, 1);
  
          // remove this activity from the package manager.
          PackageManager pm = getPackageManager();
          ComponentName name = new ComponentName(this, DefaultActivity.class);
          pm.setComponentEnabledSetting(name, PackageManager.COMPONENT_ENABLED_STATE_DISABLED,
                  PackageManager.DONT_KILL_APP);
+  setProvision()
          // terminate the activity.
          //finish();
      }
setProvision(){
     Settings.Global.putInt(getContentResolver(), Settings.Global.DEVICE_PROVISIONED, 1);
     Settings.Secure.putInt(getContentResolver(), Settings.Secure.USER_SETUP_COMPLETE, 1);
  
          // remove this activity from the package manager.
          PackageManager pm = getPackageManager();
          ComponentName name = new ComponentName(this, DefaultActivity.class);
          pm.setComponentEnabledSetting(name, PackageManager.COMPONENT_ENABLED_STATE_DISABLED,
                  PackageManager.DONT_KILL_APP);
  
          // terminate the activity.
          finish();
}
  }

```

可以增加一些 xml 页面实现具体的功能

通过上述的操作后就可以实现开机向导的定制化了
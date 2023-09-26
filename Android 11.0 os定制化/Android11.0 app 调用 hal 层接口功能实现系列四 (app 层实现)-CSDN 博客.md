> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/132327966)

1. 前言
-----

 在 11.0 的系统 rom 定制化开发中，对于一些需要在 app 中调用 [hal](https://so.csdn.net/so/search?q=hal&spm=1001.2101.3001.7020) 层的一些接口来实现某些功能而言，就需要  
打通 app 到 hal 的接口，实现功能需求，这一节首先讲在 hal 层中提供接口然后在 [jni](https://so.csdn.net/so/search?q=jni&spm=1001.2101.3001.7020) 层实现 hal 层接口调用，在  
framework 层实现添加服务调用 jni 接口，最后在 app 中实现对 hal 层的调用

  
2.app 调用 hal 层接口功能实现系列四 (app 层实现) 的核心类
-----------------------------------------

```
package/apps/TestService

```

3.app 调用 hal 层接口功能实现系列四 (app 层实现) 的核心功能分析和实现
--------------------------------------------

HAL 是硬件抽象层，它向下屏蔽了硬件的实现细节，向上提供了抽象接口，  
HAL 是底层硬件和上层框架直接的接口，框架层通过 HAL 可以操作硬件设备  
HAL 和内核驱动，HAL 实现在用户空间，驱动在内核空间，所以为了维护各大厂商  
的利益，核心算法之类的就需要在 hal 层实现了

  
HIDL 的全称是 HAL interface definition language（硬件抽象层接口定义语言），在此之前 Android 有 AIDL，  
架构在 Android binder 之上，用来定义 Android 基于 Binder 通信的 Client 与 Service 之间的接口。  
HIDL 也是类似的作用，只不过定义的是 Android Framework 与 Android HAL 实现之间的接口。  
Android 系统的硬件抽象层（Hardware Abstract Layer, HAL）运行在用户空间中，它向下屏蔽硬件驱动模块的实现细节，向上提供硬件访问服务。通过硬件抽象层，Android 系统分为两层来支持硬件设备，其中一层实现在用户空间（User Space），另外一层是现在内核空间（Kernel Space）。传统的 Linux 系统把对硬件的支持完全是现在在内核空间，即把对硬件的支持完全实现在硬件驱动模块中。

   
在实现自定义 hal 模块的开发中，从上述几篇博文中实现了 HAL，HIDL，JNI，AIDL 服务，现在只差最后一步，应用层的实现我们就可以打通应用层到 HAL 的整个调用流程了，话不多说，上代码

接下来实现 app 实现对 hal 层的调用，在 package/apps 下添加 TestService 的 app  
来实现功能

```
    package com.android.hello;
     
    import android.app.Activity;
    import android.os.Bundle;
    import android.util.Log;
    import android.view.View;
    import android.widget.Button;
    import android.widget.EditText;
    import android.os.IBinder;
    import android.app.IHelloService;
    import android.os.ServiceManager;
    import android.os.RemoteException;
    import android.view.Window;
    public class MainActivity extends Activity implements View.OnClickListener {
        private Button button;
        private EditText edt1;
        private EditText edt2;
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            requestWindowFeature(Window.FEATURE_NO_TITLE);
            setContentView(R.layout.activity_main);
            button = findViewById(R.id.button);
            edt1 = findViewById(R.id.edt1);
            edt2 = findViewById(R.id.edt2);
            button.setOnClickListener(this);
        }
     
        @Override
        public void onClick(View v) {
            ITestService testService = getTestService();
            if(testService == null){
               Log.d("test","onClick...faile to get testService...");
               return;
            }
            Log.d("test","onClick...success to get testService...");
            int a = Integer.parseInt(edt1.getText().toString());
            int b = Integer.parseInt(edt2.getText().toString());
            try{
               int total = testService.add(a,b);
               Log.e(TAG,total+"");
            }catch(Exception e){
                 Log.d("test","RemoteException = :"+e);
            }
        }
        private ITestService getTestService(){
             IBinder b = ServiceManager.getService("test");
             ITestService testService = ITestService.Stub.asInterface(b);
             return testService;
       }
    }
```

activity_main.xml

```
   <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        xmlns:tools="http://schemas.android.com/tools"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        tools:context=".MainActivity">
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal">
            <EditText
                android:id="@+id/edt1"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:hint="请输入"
                android:inputType="number"
                />
            <EditText
                android:id="@+id/edt2"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:hint="请输入"
                android:inputType="number"
                />
        </LinearLayout>
     
        <Button
            android:id="@+id/button"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="click"
            />
        <!--LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal">
            <EditText
                android:id="@+id/edt1"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:hint="请输入"
                android:inputType="number"
                />
            <EditText
                android:id="@+id/edt2"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:hint="请输入"
                android:inputType="number"
                />
        </LinearLayout-->
     
        <!--Button
            android:id="@+id/button"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="click2"
            --/>
    </LinearLayout>
```

AndroidManifest.xml

```
    <?xml version="1.0" encoding="utf-8"?>
    <manifest xmlns:android="http://schemas.android.com/apk/res/android"
        package="com.android.test">
     
        <application
            android:allowBackup="true"
            android:icon="@drawable/ic_launcher"
            android:label="HelloDemo"
            android:roundIcon="@drawable/ic_launcher"
            android:supportsRtl="true"
            android:theme="@android:style/Theme.Material.Light">
            <activity android:>
                <intent-filter>
                    <action android: />
     
                    <category android: />
                </intent-filter>
            </activity>
        </application>
     
    </manifest>
     
```

Android.bp

```
    android_app {
        name: "HelloDemo",
        srcs: ["src/**/*.java"],
        platform_apis: true,
    }
```

在通过系统编译成功后将这个 APK push 到手机中  
adb push out/target/product/RK_TF_arm64/system/app/TestDemo/TestDemo.apk /system/app/TestDemo  
重启手机，这个 APK 界面就这样：  
在这里插入图片描述  
上述在 app 中实现的功能相对简单，这样就是通过 aidl 调用 hal 模块的接口，在上述的三个 EditText 和一个 Button，前两个 EditText 用来接收两个加数，后一个 EditText 接收它们的和，通过 Button 调用 TestService 的 add 方法，TestService 的 add 方法通过 JNI 到 native 层再到 HIDL 调用 addition_hidl 函数，最终到 HAL 中调用 additionTest 函数实现两个数相加，然后返回给 APP，  
我们最终目的就是如此  
   
通过上述 app 的相关代码的调用，就实现了从 APP 的 button 的 click 事件开始，到 AIDL，JNI，HIDL，HAL  
调用，就会通过 SystemServer 获取 test 服务，然后调用其两个接口，并打印 log。以此来检查前面添加的 TestService 是否正常  
到这里我们这个系列的文章已经结束，从应用层到 HAL 的整体流程已经了解，  
其实 Android 系统的整体流程就是这样的，只不过它每一层包含复杂的逻辑，但是架构都类似，上层到底层，层层调用，  
一个 APP 就可以简单操纵硬件，一层一层提供接口，最终实现了从 framework 层通过 jni 调用  
hal 层模块的接口功能的实现

在这个自定义 hal 模块的开发中，功能虽简单，麻雀虽小，五脏俱全，到这里我们这个系列的  
文章已经结束，从应用层到 HAL 的整体流程已经了解，其实 Android 系统的整体流程  
就是这样的，只不过它每一层包含复杂的逻辑，但是架构都类似，上层到底层，层层调用，一个 APP 就可以简单操纵硬件，一层一层提供接口
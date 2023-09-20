> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124908366)

在做 11.0 的定制化开发过程中，需要提供自定义接口供 app 调用 自定义步骤如下  
自定义包名为 android.mom 在 frameworks/base/core/java/android 下创建  
mom 文件夹 然后创建自定义 java 类

```
package android.mom;

import android.content.Context;
import android.graphics.Bitmap;
import android.os.MdmManager;
import java.util.List;
import android.annotation.NonNull;
public class MyPower {
private Context mContext;
private MyManager mdmApiManager;
public SuperPower(){

}
/**
*
* @param context
* @see
*/
public MyPower(@NonNull Context context){
this.mContext = context;
mdmApiManager = (MyManager) mContext.getSystemService("api");
}

/**
*
* @param packageName
* @param className
* @see
*/
public void setDefaultLauncher(@NonNull String packageName,@NonNull String className){
mdmApiManager.setDefaultLauncher(packageName,className);
}

/**
* @see
*/
public void clearLauncher(){
mdmApiManager.clearLauncher();
}

/**
*
* @return
* @see
*/
public Bitmap captureScreen(){
return mdmApiManager.captureScreen();
}
}

```

给每一个方法添加注释 @see 表示 app 可以调用 @hide 表示隐藏 app 不能调用  
编译步骤  
source build/envsetup.sh  
lunch  
make update-api  
app 调用  
import android.mom.MyPower;  
如果没问题就表示成功

2.make-api 出现异常情况的解决  
如图:  
![](https://img-blog.csdnimg.cn/00a0104a2cb74375a40ff561fc4cbe0a.png#pic_center)  
解决方法：

这个问题是因为 Android 11 开启了 [lint](https://so.csdn.net/so/search?q=lint&spm=1001.2101.3001.7020) 代码检查，所以我们需要在 framework/base 下的 Android.bp 忽略掉代码检查

// TODO(b/145644363): move this to under StubLibraries.[bp](https://so.csdn.net/so/search?q=bp&spm=1001.2101.3001.7020) or ApiDocs.bp  
metalava_framework_docs_args = "–manifest $(location core/res/AndroidManifest.xml)" +  
"–ignore-classes-on-classpath" +  
"–hide-package com.android.server" +  
"–error UnhiddenSystemApi" +  
"–hide RequiresPermission" +  
"–hide CallbackInterface" +  
"–hide MissingPermission --hide BroadcastBehavior" +  
"–hide HiddenSuperclass --hide DeprecationMismatch --hide UnavailableSymbol" +  
"–hide SdkConstant --hide HiddenTypeParameter --hide Todo --hide Typo" +  
"–force-convert-to-warning-nullability-annotations +_:-android._:+android.icu._:-dalvik._ " +  
"–api-lint-ignore-prefix android.icu." +  
"–api-lint-ignore-prefix java." +  
"–api-lint-ignore-prefix junit." +  
"–api-lint-ignore-prefix org."

*   “–api-lint-ignore-prefix android.mom.”

build = [  
“StubLibraries.bp”,  
“ApiDocs.bp”,  
]

添加  
metalava_framework_docs_args = "  
"–api-lint-ignore-prefix android.mom."

其中 android.mom 是包名的前缀。
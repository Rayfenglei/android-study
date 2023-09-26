> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124955049)

11.0 定制化添加自定义系统服务给 app 调用  
首先要自定义服务 然后给 app 调用就好  
添加 [AIDL](https://so.csdn.net/so/search?q=AIDL&spm=1001.2101.3001.7020)

```
frameworks\base\core\java\android\os\ILgyManager.aidl

package android.os;
/** @hide */

interface ILgyManager
{
String getVal();

}

```

，在 frameworks\base\Android.[bp](https://so.csdn.net/so/search?q=bp&spm=1001.2101.3001.7020) 中添加我们的 AIDL，让其编译进系统

```
"core/java/android/os/ILgyManager.aidl",

```

，添加 service，

在 frameworks\base\services\core\java\com\android\server \ 下创建自己的文件夹 lgy，并创建自己的 service

```
lgy\LgyManagerService.java

package com.android.server.lgy;

import com.android.server.SystemService;
import android.content.Context;
import android.util.Log;
import java.util.HashMap;
import android.os.ILgyManager;


public final class LgyManagerService extends ILgyManager.Stub{

private static final String TAG = "LgyManagerService";
final Context mContext;
public LgyManagerService(Context context) {
mContext = context;

}
@Override
public  String getVal(){

try{
Log.d("lgy_bubug", "GetFromJni  ");
return "GetFromJni  ";
}catch(Exception e){
Log.d("lgy_bubug", "nativeReadPwd Exception msg = " + e.getMessage());
return " read nothings!!!";
}
}

}

```

在 frameworks\base\services\java\com\android\server\SystemServer.java 中启动我们的服务

```
private void startOtherServices() {
LgyManagerService lgyService = null;//lgy

traceBeginAndSlog("StartVibratorService");
vibrator = new VibratorService(context);
ServiceManager.addService("vibrator", vibrator);
traceEnd();

//lgy
traceBeginAndSlog("StartLgyManService");
try {
if(lgyService==null){
lgyService = new LgyManagerService(context);
}
ServiceManager.addService("lgy", lgyService);
} catch (Throwable e) {
Slog.e(TAG, "Failure starting LgyManagerService ", e);
}
traceEnd();
//lgy

}

```

, 添加给运用层调用的接口

```
frameworks\base\core\java\android\os\LgyManager.java
package android.os;

import android.annotation.SystemService;
import android.content.Context;
import android.util.Log;
import android.os.Handler;
import android.os.SystemProperties;
import java.io.IOException;
import java.io.DataInputStream;
//@SystemService(Context.LGY_SERVICE)
public final class LgyManager {
private static final String TAG = "LgyManager";
final Context mContext;
final ILgyManager mService;
/**
* {@hide}
*/
public LgyManager(Context context, ILgyManager service, Handler handler) {
mContext = context;
mService = service;
mHandler = handler;
}

public String getVal() {
Log.d(TAG,"lgy_debug   getVal ");
try {
return mService.getVal();
} catch (RemoteException e) {
throw e.rethrowFromSystemServer();
}
}
}


frameworks\base\core\java\android\content\Context.java 添加

+++  public static final String LGY_SERVICE = "lgy";


/** @hide */
@StringDef(suffix = { "_SERVICE" }, value = {
POWER_SERVICE,
WINDOW_SERVICE,
LAYOUT_INFLATER_SERVICE,
ACCOUNT_SERVICE,
ACTIVITY_SERVICE,
ALARM_SERVICE,
NOTIFICATION_SERVICE,
ACCESSIBILITY_SERVICE,
CAPTIONING_SERVICE,
KEYGUARD_SERVICE,
LOCATION_SERVICE,
//@hide: COUNTRY_DETECTOR,
SEARCH_SERVICE,
SENSOR_SERVICE,
STORAGE_SERVICE,
STORAGE_STATS_SERVICE,
WALLPAPER_SERVICE,
TIME_ZONE_RULES_MANAGER_SERVICE,
VIBRATOR_SERVICE,
+++   LGY_SERVICE,

frameworks\base\core\java\android\app\SystemServiceRegistry.java

registerService(Context.LGY_SERVICE, LgyManager.class,
new CachedServiceFetcher<LgyManager>() {
@Override
public LgyManager createService(ContextImpl ctx) throws ServiceNotFoundException {
IBinder b = ServiceManager.getServiceOrThrow(Context.LGY_SERVICE);
ILgyManager service = ILgyManager.Stub.asInterface(b);
return new LgyManager(ctx.getOuterContext(),
service, ctx.mMainThread.getHandler());
}});

```

app 中调用 api

```
package android.mdm;

import android.content.Context;
import android.graphics.Bitmap;
import android.os.ILgyManager;
import java.util.List;

public class SuperPower {
private Context mContext;
private ILgyManager mdmApiManager;
public SuperPower(){

}
/**
*
* @param context
* @see
*/
public SuperPower(Context context){
this.mContext = context;
mdmApiManager = (ILgyManager) mContext.getSystemService(Context.LGY_SERVICE);
}

public void getval(){
mdmApiManager.getVal();
}
}

```

新增的 service 配置 selinux 策略

默认新增的 GetWifiMacService 服务的 seandroid 类型为 default_android_service。如果不配置第三方 App 访问不了。具体配置如下:

在文件 system\sepolicy\private\service_contexts 中添加如下内容

```
# ///ADD START
lgy                                  u:object_r:lgy_service:s0
# ///ADD END

```

在文件 system\sepolicy\prebuilts\api\29.0\private\service_contexts 中添加如下内容

```
# ///ADD START
lgy                                  u:object_r:lgy_service:s0
# ///ADD END

```

​​​​​​​

在文件 system\sepolicy\public\service.te 中添加如下内容

```
# ///ADD START
type lgy_service, app_api_service, ephemeral_app_api_service, system_server_service, service_manager_type;
# ///ADD END

```

​​​​​​​

在文件 system\sepolicy\prebuilts\api\29.0\public\service.te 中添加如下内容

```
# ///ADD START
type lgy_service, app_api_service, ephemeral_app_api_service, system_server_service, service_manager_type;
# ///ADD END

```
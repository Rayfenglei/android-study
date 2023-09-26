> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127232887)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 根据包名授予读取 IMEI 权限的核心类](#t1)

[3. 根据包名授予读取 IMEI 权限的核心功能实现和分析](#t2)

[3.1 TelephonyManager 的关于 imei 的相关源码](#t3)

[3.2 PhoneSubInfoController.java 关于获取 imei 的相关源码](#t4)

[3.3 TelephonyPermissions 的相关获取 imei 的代码](#t5)

1. 概述
-----

 在 11.0 的产品开发中，对于读取设备的 [imei](https://so.csdn.net/so/search?q=imei&spm=1001.2101.3001.7020) sn 号功能也是常有的，而在 10.0 以后对于读取 imei 也是受权限要求越来越多了一般的 app 是读取不到这个权限了，根据产品需求需要读取这个权限，所以需要在系统中对这个 app 授权让它读取包名，从而实现功能

2. 根据包名授予读取 IMEI 权限的核心类
-----------------------

```
frameworks/base/telephony/java/android/telephony/TelephonyManager.java
frameworks/opt/telephony/src/java/com/android/internal/telephony/PhoneSubInfoController.java
frameworks/base/telephony/common/com/android/internal/telephony/TelephonyPermissions.java
```

3. 根据包名授予读取 IMEI 权限的核心功能实现和分析
-----------------------------

 首选在 app 中看如何获取 imei 的  
 String imei =((TelephonyManager) context.getSystemService(TELEPHONY_SERVICE)).getDeviceId();  
 从上面代码可以看出是 TelephonyManager 的 getDeviceId(); 来获取 imei 号的  
 接下来看下 TelephonyManager 的 getDeviceId() 的相关源码

### 3.1 TelephonyManager 的关于 imei 的相关源码

```
       @Deprecated
      @SuppressAutoDoc // No support for device / profile owner or carrier privileges (b/72967236).
      @RequiresPermission(android.Manifest.permission.READ_PRIVILEGED_PHONE_STATE)
      public String getDeviceId(int slotIndex) {
          // FIXME this assumes phoneId == slotIndex
          try {
              IPhoneSubInfo info = getSubscriberInfoService();
              if (info == null)
                  return null;
              return info.getDeviceIdForPhone(slotIndex, mContext.getOpPackageName(),
                      mContext.getAttributionTag());
          } catch (RemoteException ex) {
              return null;
          } catch (NullPointerException ex) {
              return null;
          }
      }
```

从 TelephonyManager 的源码可以看出需要 android.Manifest.permission.READ_PRIVILEGED_PHONE_STATE  
这个权限，而这个权限只有系统 app 可以申请这个权限，普通 app 没有这个权限，代码中主要是通过 IPhoneSubInfo 的 getDeviceIdForPhone 获取的  
接下来看下 IPhoneSubInfo 的相关源码

### 3.2 PhoneSubInfoController.java 关于获取 imei 的相关源码

```
  public class PhoneSubInfoController extends IPhoneSubInfo.Stub {
     public PhoneSubInfoController(Context context) {
         ServiceRegisterer phoneSubServiceRegisterer = TelephonyFrameworkInitializer
                 .getTelephonyServiceManager()
                 .getPhoneSubServiceRegisterer();
         if (phoneSubServiceRegisterer.get() == null) {
             phoneSubServiceRegisterer.register(this);
         }
         mContext = context;
         mAppOps = (AppOpsManager) mContext.getSystemService(Context.APP_OPS_SERVICE);
     }
```

PhoneSubInfoController 就是 IPhoneSubInfo 的实现类，而从构造方法中可以看出主要是初始化相关的参数，

```
public String getDeviceIdForPhone(int phoneId, String callingPackage,
             String callingFeatureId) {
         return callPhoneMethodForPhoneIdWithReadDeviceIdentifiersCheck(phoneId, callingPackage,
                 callingFeatureId, "getDeviceId", (phone) -> phone.getDeviceId());
     }
```

这里主要是在 TelephonyManager 的 getDeviceId(int slotIndex) 获取 imei 的主要方法，从这里看出主要是调用  
callPhoneMethodForPhoneIdWithReadDeviceIdentifiersCheck 来实现查询 imei 的核心方法

```
private <T> T callPhoneMethodForPhoneIdWithReadDeviceIdentifiersCheck(int phoneId,
              String callingPackage, @Nullable String callingFeatureId, String message,
              CallPhoneMethodHelper<T> callMethodHelper) {
          // Getting subId before doing permission check.
          if (!SubscriptionManager.isValidPhoneId(phoneId)) {
              phoneId = 0;
          }
          final Phone phone = PhoneFactory.getPhone(phoneId);
          if (phone == null) {
              return null;
          }
          if (!TelephonyPermissions.checkCallingOrSelfReadDeviceIdentifiers(mContext,
                  phone.getSubId(), callingPackage, callingFeatureId, message)) {
              return null;
          }
  
          final long identity = Binder.clearCallingIdentity();
          try {
              return callMethodHelper.callMethod(phone);
          } finally {
              Binder.restoreCallingIdentity(identity);
          }
      }
```

从 callPhoneMethodForPhoneIdWithReadDeviceIdentifiersCheck 的上述代码可以看出在 TelephonyPermissions.checkCallingOrSelfReadDeviceIdentifiers  
这个方法中判断是否有权限获取 imei 号，所以继续看出 TelephonyPermissions 的相关代码

### 3.3 TelephonyPermissions 的相关获取 imei 的代码

```
      public static boolean checkCallingOrSelfReadSubscriberIdentifiers(Context context, int subId,
              String callingPackage, @Nullable String callingFeatureId, String message) {
          return checkPrivilegedReadPermissionOrCarrierPrivilegePermission(
                  context, subId, callingPackage, callingFeatureId, message, false);
      }
  
      private static boolean checkPrivilegedReadPermissionOrCarrierPrivilegePermission(
              Context context, int subId, String callingPackage, @Nullable String callingFeatureId,
              String message, boolean allowCarrierPrivilegeOnAnySub) {
          int uid = Binder.getCallingUid();
          int pid = Binder.getCallingPid();
  
          // If the calling package has carrier privileges for specified sub, then allow access.
          if (checkCarrierPrivilegeForSubId(context, subId)) return true;
  
          // If the calling package has carrier privileges for any subscription
          // and allowCarrierPrivilegeOnAnySub is set true, then allow access.
          if (allowCarrierPrivilegeOnAnySub && checkCarrierPrivilegeForAnySubId(context, uid)) {
              return true;
          }
  
          PermissionManager permissionManager = (PermissionManager) context.getSystemService(
                  Context.PERMISSION_SERVICE);
          if (permissionManager.checkDeviceIdentifierAccess(callingPackage, message, callingFeatureId,
                  pid, uid) == PackageManager.PERMISSION_GRANTED) {
              return true;
          }
  
          return reportAccessDeniedToReadIdentifiers(context, subId, pid, uid, callingPackage,
                  message);
      }
```

从 checkCallingOrSelfReadSubscriberIdentifiers 中可以看出主要是在 checkPrivilegedReadPermissionOrCarrierPrivilegePermission 中  
处理是否有权限的功能处理，而在 checkPrivilegedReadPermissionOrCarrierPrivilegePermission 中  
permissionManager.checkDeviceIdentifierAccess 是判断是否有权限的地方所以可以在这里加上包名的判断  
具体修改如下:

```
 private static boolean checkPrivilegedReadPermissionOrCarrierPrivilegePermission(
              Context context, int subId, String callingPackage, @Nullable String callingFeatureId,
              String message, boolean allowCarrierPrivilegeOnAnySub) {
          int uid = Binder.getCallingUid();
          int pid = Binder.getCallingPid();
  
          // If the calling package has carrier privileges for specified sub, then allow access.
          if (checkCarrierPrivilegeForSubId(context, subId)) return true;
  
          // If the calling package has carrier privileges for any subscription
          // and allowCarrierPrivilegeOnAnySub is set true, then allow access.
          if (allowCarrierPrivilegeOnAnySub && checkCarrierPrivilegeForAnySubId(context, uid)) {
              return true;
          }
        
          PermissionManager permissionManager = (PermissionManager) context.getSystemService(
                  Context.PERMISSION_SERVICE);
           // 主要修改的地方，在这里添加callingPackage.equals("包名")
        -  if (permissionManager.checkDeviceIdentifierAccess(callingPackage, message, callingFeatureId,
        -          pid, uid) == PackageManager.PERMISSION_GRANTED) {
        +if (permissionManager.checkDeviceIdentifierAccess(callingPackage, message, callingFeatureId,
        +          pid, uid) == PackageManager.PERMISSION_GRANTED || callingPackage.equals("包名")) {
 
              return true;
          }
  
          return reportAccessDeniedToReadIdentifiers(context, subId, pid, uid, callingPackage,
                  message);
      }
```

主要是在判断权限的地方加上一个判断包名的代码，根据包名判断是否返回 true，就这样实现功能要求
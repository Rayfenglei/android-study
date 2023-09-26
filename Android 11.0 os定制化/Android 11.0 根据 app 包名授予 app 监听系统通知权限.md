> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/132590556)

1. 概述
-----

在 11.0 的系统 rom 产品定制化开发中，在一些[产品开发](https://so.csdn.net/so/search?q=%E4%BA%A7%E5%93%81%E5%BC%80%E5%8F%91&spm=1001.2101.3001.7020)中，第三方 app 需要开启系统通知权限，然后可以在 app 中，监听系统所有通知，来做个通知中心的功能，所以需要授权  
获取系统通知的权限，然后来顺利的监听系统通知。来做系统通知的功能，首选分析下相关授权通知的功能，看如何实现根据 app 包名授予 app 监听系统通知权限的功能  
接下来看下 app 中是如何申请通知权限的

```
private boolean notificationListenerEnable() {
 
boolean enable = false;
 
String packageName = getPackageName();
 
String flat = Settings.Secure.getString(getContentResolver(), "enabled_notification_listeners");
 
if (flat != null) {
 
enable = flat.contains(packageName);
 
}
 
return enable;
 
}
 
/**
 
* 是否启用通知监听服务
 
*
 
* @return
 
*/
 
public boolean isNLServiceEnabled() {
 
Set<String> packageNames = NotificationManagerCompat.getEnabledListenerPackages(this);
 
if (packageNames.contains(getPackageName())) {
 
return true;
 
}
 
return false;
 
}
 
boolean isEnable = notificationListenerEnable();
 
boolean isSe = isNLServiceEnabled();
 
if(!isEnable)startActivity(new Intent("android.settings.ACTION_NOTIFICATION_LISTENER_SETTINGS"));
```

在上述的 app 关于监听通知的相关代码中，关于实现根据 app 包名授予 app 监听系统通知权限的功能, 可以通过上述的 notificationListenerEnable(); 判断通知权限是否开启的方法来确定是否需要调用 startActivity(new Intent("android.settings.ACTION_NOTIFICATION_LISTENER_SETTINGS"));  
来手动开启通知权限，如果没有这个 app 的通知权限就需要手动开启权限，接下来分析下相关代码看下如何默认授予 app 的通知权限的功能

2. 根据 app 包名授予 app 监听系统通知权限的核心类
-------------------------------

```
   packages/apps/Settings/src/com/android/settings/notification/NotificationAccessSettings.java
    frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
```

3. 根据 app 包名授予 app 监听系统通知权限的核心功能分析和实现
-------------------------------------

在实现根据 app 包名授予 app 监听系统通知权限的功能, 通过系统的源码中分析，NotificationAccessSettings.java 就是管理所有 app 的通知使用权的工具类，通过列表来开启关闭每个 app 的相关通知权限，  
在通过上述的 app 的代码分析中，可以发现，最终是需要在 NotificationAccessSettings.java 中，来开启这个通知的权限的，接下来分析下看在  
NotificationAccessSettings.java 这个里面来怎么样授予通知的权限的，通过这个分析，看如何

实现根据 app 包名授予 app 监听系统通知权限的功能

```
  /**
       * Settings screen for managing notification listener permissions
       */
      public class NotificationAccessSettings extends ManagedServiceSettings {
          private static final String TAG = NotificationAccessSettings.class.getSimpleName();
          private static final Config CONFIG =  new Config.Builder()
                  .setTag(TAG)
                  .setSetting(Settings.Secure.ENABLED_NOTIFICATION_LISTENERS)
                  .setIntentAction(NotificationListenerService.SERVICE_INTERFACE)
                  .setPermission(android.Manifest.permission.BIND_NOTIFICATION_LISTENER_SERVICE)
                  .setNoun("notification listener")
                  .setWarningDialogTitle(R.string.notification_listener_security_warning_title)
                  .setWarningDialogSummary(R.string.notification_listener_security_warning_summary)
                  .setEmptyText(R.string.no_notification_listeners)
                  .build();
      
          private NotificationManager mNm;
      
          @Override
          public int getMetricsCategory() {
              return MetricsEvent.NOTIFICATION_ACCESS;
          }
      
          @Override
          public void onAttach(Context context) {
              super.onAttach(context);
              mNm = context.getSystemService(NotificationManager.class);
          }
```

在实现根据 app 包名授予 app 监听系统通知权限的功能, 在 NotificationAccessSettings.java 中的上述代码中，可以看出，在 onAttach(Context context) 主要是[实例化](https://so.csdn.net/so/search?q=%E5%AE%9E%E4%BE%8B%E5%8C%96&spm=1001.2101.3001.7020)了 NotificationManager 这个通知的管理类  
然后通过这个 NotificationManager 类来负责授权通知授权和拒绝通知权限的相关功能的实现，[主要的](https://so.csdn.net/so/search?q=%E4%B8%BB%E8%A6%81%E7%9A%84&spm=1001.2101.3001.7020)通知管理就是在 NotificationManager.java 来负责处理的

```
     @Override
         protected boolean setEnabled(ComponentName service, String title, boolean enable) {
             logSpecialPermissionChange(enable, service.getPackageName());
             if (!enable) {
                 if (!isServiceEnabled(service)) {
                     return true; // already disabled
                 }
                 // show a friendly dialog
                 new FriendlyWarningDialogFragment()
                         .setServiceInfo(service, title, this)
                         .show(getFragmentManager(), "friendlydialog");
                 return false;
             } else {
                 if (isServiceEnabled(service)) {
                     return true; // already enabled
                 }
                 // show a scary dialog
                 new ScaryWarningDialogFragment()
                         .setServiceInfo(service, title, this)
                         .show(getFragmentManager(), "dialog");
                 return false;
             }
         }
     
         @Override
         protected boolean isServiceEnabled(ComponentName cn) {
             return mNm.isNotificationListenerAccessGranted(cn);
         }
     
          @Override
          protected void enable(ComponentName service) {
              mNm.setNotificationListenerAccessGranted(service, true);
          }
      
          @Override
          protected int getPreferenceScreenResId() {
              return R.xml.notification_access_settings;
          }
      
          @VisibleForTesting
          void logSpecialPermissionChange(boolean enable, String packageName) {
              int logCategory = enable ? MetricsEvent.APP_SPECIAL_PERMISSION_NOTIVIEW_ALLOW
                      : MetricsEvent.APP_SPECIAL_PERMISSION_NOTIVIEW_DENY;
              FeatureFactory.getFactory(getContext()).getMetricsFeatureProvider().action(getContext(),
                      logCategory, packageName);
          }
      
          private static void disable(final NotificationAccessSettings parent, final ComponentName cn) {
              parent.mNm.setNotificationListenerAccessGranted(cn, false);
              AsyncTask.execute(() -> {
                  if (!parent.mNm.isNotificationPolicyAccessGrantedForPackage(
                          cn.getPackageName())) {
                      parent.mNm.removeAutomaticZenRules(cn.getPackageName());
                  }
              });
          }
```

在实现根据 app 包名授予 app 监听系统通知权限的功能, 从 NotificationAccessSettings.java 中的上述的代码中可以看出，主要是在 isServiceEnabled(ComponentName cn) 中进行判断当前的 app，是否有系统通知的权限，  
而在 enable(ComponentName service) 中是具体根据包名来授予 app 的系统通知权限的方法，调用 mNm.setNotificationListenerAccessGranted(service, true);  
来授予权限，而在 disable(final NotificationAccessSettings parent, final ComponentName cn) 就是拒绝某个 app 的系统通知权限，具体也是调用  
mNm.setNotificationListenerAccessGranted(cn, false); 来拒绝这个 app 的权限的实现功能

3.2 PhoneWindowManager.java 在系统开机完成的时候授予 app 的系统通知的权限
-----------------------------------------------------

```
        /** {@inheritDoc} */
          @Override
          public void systemReady() {
              // In normal flow, systemReady is called before other system services are ready.
              // So it is better not to bind keyguard here.
              mKeyguardDelegate.onSystemReady();
      
              mVrManagerInternal = LocalServices.getService(VrManagerInternal.class);
              if (mVrManagerInternal != null) {
                  mVrManagerInternal.addPersistentVrModeStateListener(mPersistentVrModeListener);
              }
      
              readCameraLensCoverState();
              updateUiMode();
              synchronized (mLock) {
                  updateOrientationListenerLp();
                  mSystemReady = true;
                  mHandler.post(new Runnable() {
                      @Override
                      public void run() {
                          updateSettings();
                      }
                  });
                  // If this happens, for whatever reason, systemReady came later than systemBooted.
                  // And keyguard should be already bound from systemBooted
                  if (mSystemBooted) {
                      mKeyguardDelegate.onBootCompleted();
                  }
              }
      
              mSystemGestures.systemReady();
              mImmersiveModeConfirmation.systemReady();
      
              mAutofillManagerInternal = LocalServices.getService(AutofillManagerInternal.class);
          }
```

在实现根据 app 包名授予 app 监听系统通知权限的功能, 在 PhoneWindowManager.java 的上述方法中，在 systemReady() 这个方法中，系统会在主要服务启动完毕后，调用 systemReady() 来完成其他功能的启动，所以也同样可以在  
这里通过调用 NotificationManager 这个通知管理类的 setNotificationListenerAccessGranted 授予通知权限的类，  
来根据包名授予某个 app 的通知权限，接下来实现这个核心功能

```
        /** {@inheritDoc} */
          @Override
          public void systemReady() {
              // In normal flow, systemReady is called before other system services are ready.
              // So it is better not to bind keyguard here.
              mKeyguardDelegate.onSystemReady();
      
              mVrManagerInternal = LocalServices.getService(VrManagerInternal.class);
              if (mVrManagerInternal != null) {
                  mVrManagerInternal.addPersistentVrModeStateListener(mPersistentVrModeListener);
              }
      
              readCameraLensCoverState();
              updateUiMode();
              synchronized (mLock) {
                  updateOrientationListenerLp();
                  mSystemReady = true;
                  mHandler.post(new Runnable() {
                      @Override
                      public void run() {
                          updateSettings();
                      }
                  });
                  // If this happens, for whatever reason, systemReady came later than systemBooted.
                  // And keyguard should be already bound from systemBooted
                  if (mSystemBooted) {
                      mKeyguardDelegate.onBootCompleted();
                  }
              }
      
              mSystemGestures.systemReady();
              mImmersiveModeConfirmation.systemReady();
      
    // add core start
            NotificationManager mNm = mContext.getSystemService(NotificationManager.class);
            ComponentName componentName = new ComponentName("com.pne.jnitest","com.pne.jnitest.EditActivity");
            mNm.setNotificationListenerAccessGranted(componentName, true);
    // add core end
     
              mAutofillManagerInternal = LocalServices.getService(AutofillManagerInternal.class);
          }
```

在实现根据 app 包名授予 app 监听系统通知权限的功能, 在 PhoneWindowManager.java 的上述方法中，在 systemReady() 这个方法中，通过添加 NotificationManager 这个通知管理类的  
setNotificationListenerAccessGranted 来授予 app 通知的权限，最终调用 mNm.setNotificationListenerAccessGranted(componentName, true);  
来实现对 app 的通知权限的默认授权，这样就实现了功能
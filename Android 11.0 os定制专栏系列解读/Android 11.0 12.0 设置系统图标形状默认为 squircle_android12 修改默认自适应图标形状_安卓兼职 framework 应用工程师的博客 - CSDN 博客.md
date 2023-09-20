> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124774794)

### 1. 概述

在 android11.0 12.0 系统中，图标默认形状分为 teardrop,squircle,square,roundedrect 和默认图标，而对于系统图标，可以选择就这几种，产品需求要求设置默认图标形状为 squircle

### 2. 设置系统图标形状默认为 squircle 的核心类

```
frameworks\base\services\core\java\com\android\server\policy\PhoneWindowManager.java
packages/apps/Settings/src/com/android/settings/development/OverlayCategoryPreferenceController.java

```

### 3. 设置系统图标形状默认为 squircle 的功能分析和实现

### 3.1OverlayCategoryPreferenceController.java 源码分析

在系统设置修改系统图标里面就是 OverlayCategoryPreferenceController.java 通过调用 api 实现的  
于是就来查看 OverlayCategoryPreferenceController.java 的源码  
路径为:

接下来看具体源码

```
 @VisibleForTesting
     OverlayCategoryPreferenceController(Context context, PackageManager packageManager,
             IOverlayManager overlayManager, String category) {
         super(context);
         mOverlayManager = overlayManager;
         mPackageManager = packageManager;
         mCategory = category;
         mAvailable = overlayManager != null && !getOverlayInfos().isEmpty();
     }
 
     public OverlayCategoryPreferenceController(Context context, String category) {
         this(context, context.getPackageManager(), IOverlayManager.Stub
                 .asInterface(ServiceManager.getService(Context.OVERLAY_SERVICE)), category);
     }
 
     @Override
     public boolean isAvailable() {
         return mAvailable;
     }
 
     @Override
     public String getPreferenceKey() {
         return mCategory;
     }
 
     @Override
     public void displayPreference(PreferenceScreen screen) {
         super.displayPreference(screen);
         setPreference(screen.findPreference(getPreferenceKey()));
     }
 
     @VisibleForTesting
     void setPreference(ListPreference preference) {
         mPreference = preference;
     }
  
      @Override
      public boolean onPreferenceChange(Preference preference, Object newValue) {
          return setOverlay((String) newValue);
      }
  
      private boolean setOverlay(String packageName) {
          final String currentPackageName = getOverlayInfos().stream()
                  .filter(info -> info.isEnabled())
                  .map(info -> info.packageName)
                  .findFirst()
                  .orElse(null);
  
          if (PACKAGE_DEVICE_DEFAULT.equals(packageName) && TextUtils.isEmpty(currentPackageName)
                  || TextUtils.equals(packageName, currentPackageName)) {
              // Already set.
              return true;
          }
  
          new AsyncTask<Void, Void, Boolean>() {
              @Override
              protected Boolean doInBackground(Void... params) {
                  try {
                      if (PACKAGE_DEVICE_DEFAULT.equals(packageName)) {
                          return mOverlayManager.setEnabled(currentPackageName, false, USER_SYSTEM);
                      } else {
                          return mOverlayManager.setEnabledExclusiveInCategory(packageName,
                                  USER_SYSTEM);
                      }
                  } catch (SecurityException | IllegalStateException | RemoteException e) {
                      Log.w(TAG, "Error enabling overlay.", e);
                      return false;
                  }
              }
  
              @Override
              protected void onPostExecute(Boolean success) {
                  updateState(mPreference);
                  if (!success) {
                      Toast.makeText(
                              mContext, R.string.overlay_toast_failed_to_apply, Toast.LENGTH_LONG)
                              .show();
                  }
              }
          }.execute();
  
          return true; // Assume success; toast on failure.
      }

```

通过上面的 setOverlay(String packageName) 源码发现

```
mOverlayManager.setEnabledExclusiveInCategory(packageName,
USER_SYSTEM);

```

就是这里通过设置图标形状改变系统默认图标的

而: teardrop 对应的 packageName 为：com.android.theme.icon.teardrop,  
squircle 对应的 packageName 为：com.android.theme.icon.squircle,  
square 对应的 packageName 为：com.android.theme.icon.square,  
roundedrect 对应的 packageName 为：com.android.theme.icon.roundedrect,

所以要设置默认图标为 squircle 就需要这样设置

```
overlayManager.setEnabledExclusiveInCategory(
“com.android.theme.icon.squircle”, UserHandle.USER_CURRENT);

```

即可  
具体修改如下：  
在 PhoneWindowManager.java 中在系统开机完成后修改如下:  
路径: frameworks\base\services\core\java\com\android\server\policy\PhoneWindowManager.java

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
          mDefaultDisplayRotation.updateOrientationListener();
          synchronized (mLock) {
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
  
          mAutofillManagerInternal = LocalServices.getService(AutofillManagerInternal.class);
      }
  
      /** {@inheritDoc} */
      @Override
      public void systemBooted() {
          bindKeyguard();
          synchronized (mLock) {
              mSystemBooted = true;
              if (mSystemReady) {
                  mKeyguardDelegate.onBootCompleted();
              }
          }
          startedWakingUp(ON_BECAUSE_OF_UNKNOWN);
          finishedWakingUp(ON_BECAUSE_OF_UNKNOWN);
          screenTurningOn(null);
          screenTurnedOn();
        +  setOverlay()
      }
  
      @Override
      public boolean canDismissBootAnimation() {
          return mDefaultDisplayPolicy.isKeyguardDrawComplete();
      }

```

在 PhoneWindowManager.java  
的 systemBooted() 此方法中添加 setOverlay() 即可

```
private void setOverlay() {
	int overlay = Settings.Global.getInt(mContext.getContentResolver(),"setoverlay",0);
    android.util.Log.e("Overlay","overlay:"+overlay);
	if(overlay==0){
        final IOverlayManager overlayManager = IOverlayManager.Stub.asInterface(
                            ServiceManager.getService(Context.OVERLAY_SERVICE));
                    try {
                        overlayManager.setEnabledExclusiveInCategory(
                                "com.android.theme.icon.squircle", UserHandle.USER_CURRENT);
                    } catch (RemoteException e) {
                        throw new IllegalStateException(
                                "Failed to set nav bar interaction mode overlay");
                    }
        Settings.Global.putInt(mContext.getContentResolver(),"setoverlay",1);						
	}			
}

```

然后编译发现实现了功能
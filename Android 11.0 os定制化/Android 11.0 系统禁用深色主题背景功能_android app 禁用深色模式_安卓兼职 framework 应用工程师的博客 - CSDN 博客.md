> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/129997762)

1. 前言
-----

  
 在 11.0 的系统 rom 定制化开发中，在 11.0 的原生系统中，默认有正常背景和深色主题背景，当系统设置深色主题背景或者进入省电模式情况下会进入  
深色主题背景模式这样就会导致系统页面都是黑色的显得很不美观，进入了深色主题模式，产品要求禁用深色主题模式，  
所以功能开发需要要求禁用深色主题功能，这就要从系统设置深色主题功能流程入手来分析，看怎样设置深色主题背景

如图:

![](https://img-blog.csdnimg.cn/79ac01a9be3246adaf671845f0e8d11f.png)

2. 系统禁用深色主题背景功能的核心类
-------------------

```
  package/apps/Settings/src/com/android/settings/display/DarkUIPreferenceController.java
  frameworks/base/services/core/java/com/android/server/UiModeManagerService.java
  frameworks/base/core/java/android/app/UiModeManager.java
```

3. 系统禁用深色主题背景功能的核心功能分析和实现  
3.1 DarkUIPreferenceController.java 关于深色主题背景相关代码分析
------------------------------------------------------------------------------

```
     public DarkUIPreferenceController(Context context, String key) {
          super(context, key);
          mContext = context;
          mUiModeManager = context.getSystemService(UiModeManager.class);
          mPowerManager = context.getSystemService(PowerManager.class);
      }
  
      @Override
      public boolean isChecked() {
           return (mContext.getResources().getConfiguration().uiMode
                   & Configuration.UI_MODE_NIGHT_YES) != 0;
      }
  
      @Override
      public void displayPreference(PreferenceScreen screen) {
          super.displayPreference(screen);
          mPreference = screen.findPreference(getPreferenceKey());
      }
   @Override
      public boolean setChecked(boolean isChecked) {
          final boolean dialogSeen =
                  Settings.Secure.getInt(mContext.getContentResolver(),
                          Settings.Secure.DARK_MODE_DIALOG_SEEN, 0) == DIALOG_SEEN;
          if (!dialogSeen && isChecked) {
              showDarkModeDialog();
          }
          return mUiModeManager.setNightModeActivated(isChecked);
      }
```

在 DarkUIPreferenceController.java 的上述代码中可以看出，重点是在 setChecked(boolean isChecked) 的相关代码中发现最终是通过  
UiModeManager.setNightMode(）来控制是否打开深色主题背景的, 所以说核心控制深色主题背景模式的就是 mUiModeManager 中的  
相关方法，所以接下来看下 UiModeManager 的相关设置深色主题的相关代码

3.2 UiModeManager.java 关于深色主题背景的相关代码
------------------------------------

```
        public void setNightMode(@NightMode int mode) {
            if (mService != null) {
                try {
                    mService.setNightMode(mode);
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
        }
     
        /**
         * Returns the currently configured night mode.
         * <p>
         * May be one of:
         * <ul>
         *   <li>{@link #MODE_NIGHT_NO}</li>
         *   <li>{@link #MODE_NIGHT_YES}</li>
         *   <li>{@link #MODE_NIGHT_AUTO}</li>
         *   <li>{@code -1} on error</li>
         * </ul>
         *
         * @return the current night mode, or {@code -1} on error
         * @see #setNightMode(int)
         */
        public @NightMode int getNightMode() {
            if (mService != null) {
                try {
                    return mService.getNightMode();
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            return -1;
        }
```

通过 UiModeManager.java 中的上述代码发现，在 setNightMode(@NightMode int mode)  
要获取和设置深色主题背景而 UiModeManager.java 中的 setNightMode(@NightMode int mode)  
最终是通过 UiModeManagerService.java 的 setNightMode(mode); 来设置深色主题背景，  
，并且是在通过 UiModeManager.java 中的 getNightMode() 判断是否是黑白模式的判断，  
接下来看下 UiModeManagerService.java 中关于设置深色主题的相关源码

3.3 UiModeManagerService.java 设置深色主题背景的相关代码分析
---------------------------------------------

```
             @Override
              public void setNightMode(int mode) {
     
                //add core start
                if(true)return;
               //add core end
     
                  if (isNightModeLocked() && (getContext().checkCallingOrSelfPermission(
                          android.Manifest.permission.MODIFY_DAY_NIGHT_MODE)
                          != PackageManager.PERMISSION_GRANTED)) {
                      Slog.e(TAG, "Night mode locked, requires MODIFY_DAY_NIGHT_MODE permission");
                      return;
                  }
                  switch (mode) {
                      case UiModeManager.MODE_NIGHT_NO:
                      case UiModeManager.MODE_NIGHT_YES:
                      case MODE_NIGHT_AUTO:
                      case MODE_NIGHT_CUSTOM:
                          break;
                      default:
                          throw new IllegalArgumentException("Unknown mode: " + mode);
                  }
      
                  final int user = UserHandle.getCallingUserId();
                  final long ident = Binder.clearCallingIdentity();
                  try {
                      synchronized (mLock) {
                          if (mNightMode != mode) {
                              if (mNightMode == MODE_NIGHT_AUTO || mNightMode == MODE_NIGHT_CUSTOM) {
                                  unregisterScreenOffEventLocked();
                                  cancelCustomAlarm();
                              }
      
                              mNightMode = mode;
                              resetNightModeOverrideLocked();
                              persistNightMode(user);
                              // on screen off will update configuration instead
                              if ((mNightMode != MODE_NIGHT_AUTO && mNightMode != MODE_NIGHT_CUSTOM)
                                      || shouldApplyAutomaticChangesImmediately()) {
                                  unregisterScreenOffEventLocked();
                                  updateLocked(0, 0);
                              } else {
                                  registerScreenOffEventLocked();
                              }
                          }
                      }
                  } finally {
                      Binder.restoreCallingIdentity(ident);
                  }
              }
```

通过 UiModeManagerService.java 中的上述源码中，可以发现在 setNightMode(int mode) 就是最终设置

深色主题的代码，接下来就需要在设置这个深色主题的时候直接返回就可以了，所以在上面增加了

//add core start

if(true)return;

//add core end

这样就可以保证调用这个方法的时候，直接会返回，就不会执行深色主题模式了
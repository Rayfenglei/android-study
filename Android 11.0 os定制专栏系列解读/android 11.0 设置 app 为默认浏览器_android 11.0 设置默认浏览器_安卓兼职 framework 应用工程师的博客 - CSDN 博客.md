> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124908266)

1. 概述
-----

在 11.0 的产品定制化中，如果[系统安装](https://so.csdn.net/so/search?q=%E7%B3%BB%E7%BB%9F%E5%AE%89%E8%A3%85&spm=1001.2101.3001.7020)多个浏览器时，需要设置默认浏览器来完成需求，这就需要看系统设置中的相关源码  
当出现多个浏览器时，该如何设置默认浏览器呢,  
其实在 Settings 默认应用 -> 浏览器应用 当点击选择浏览器时会调用 /package/app/PermissionController 的代码

2. 设置 app 为默认浏览器的相关代码
---------------------

```
packages\apps\PermissionController\src\com\android\packageinstaller\role\ui\ManageRoleHolderStateLiveData.java
frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java

```

3. 设置 app 为默认浏览器的相关代码功能分析
-------------------------

3.1ManageRoleHolderStateLiveData 关于切换默认浏览器相关代码分析
------------------------------------------------

点击 [Preference](https://so.csdn.net/so/search?q=Preference&spm=1001.2101.3001.7020) 切换是最终调用的代码走到 setRoleHolderAsUser()

```
packages\apps\PermissionController\src\com\android\packageinstaller\role\ui\ManageRoleHolderStateLiveData.java
public void setRoleHolderAsUser(@NonNull String roleName, @NonNull String packageName,
boolean add, int flags, @NonNull UserHandle user, @NonNull Context context) {
if (getValue() != STATE_IDLE) {
Log.e(LOG_TAG, "Already (tried) managing role holders, requested role: " + roleName
+ ", requested package: " + packageName);
return;
}
if (DEBUG) {
Log.i(LOG_TAG, (add ? "Adding" : "Removing") + " package as role holder, role: "
+ roleName + ", package: " + packageName);
}
mLastPackageName = packageName;
mLastAdd = add;
mLastFlags = flags;
mLastUser = user;
setValue(STATE_WORKING);

RoleManager roleManager = context.getSystemService(RoleManager.class);
Executor executor = context.getMainExecutor();
Consumer<Boolean> callback = successful -> {
if (successful) {
if (DEBUG) {
Log.i(LOG_TAG, "Package " + (add ? "added" : "removed")
+ " as role holder, role: " + roleName + ", package: " + packageName);
}
setValue(STATE_SUCCESS);
} else {
if (DEBUG) {
Log.i(LOG_TAG, "Failed to " + (add ? "add" : "remove")
+ " package as role holder, role: " + roleName + ", package: "
+ packageName);
}
setValue(STATE_FAILURE);
}
};
if (add) {
roleManager.addRoleHolderAsUser(roleName, packageName, flags, user, executor, callback);
} else {
roleManager.removeRoleHolderAsUser(roleName, packageName, flags, user, executor,
callback);
}
}

```

各默认应用对应的 roleName 如下，改造一下 setRoleHolderAsUser(), 传递 roleName 和 packageName  
电话  
role: android.app.role.DIALER, package: com.android.dialer  
短信  
role: android.app.role.SMS, package: com.android.mms  
主屏幕  
role: android.app.role.HOME, package: com.android.launcher3  
浏览器  
role: android.app.role.BROWSER, package: com.android.browser  
所以 RoleManager 的 addRoleHolderAsUser 来实现默认浏览器的切换

3.2 PhoneWindowManager.java 中设置默认浏览器的功能实现
-----------------------------------------

```
public class PhoneWindowManager implements WindowManagerPolicy {
      static final String TAG = "WindowManager";
      static final boolean localLOGV = false;
      static final boolean DEBUG_INPUT = false;
      static final boolean DEBUG_KEYGUARD = false;
      static final boolean DEBUG_SPLASH_SCREEN = false;
      static final boolean DEBUG_WAKEUP = false;
      static final boolean SHOW_SPLASH_SCREENS = true;
  
      // Whether to allow dock apps with METADATA_DOCK_HOME to temporarily take over the Home key.
      // No longer recommended for desk docks;
      static final boolean ENABLE_DESK_DOCK_HOME_CAPTURE = false;
  
      // Whether to allow devices placed in vr headset viewers to have an alternative Home intent.
      static final boolean ENABLE_VR_HEADSET_HOME_CAPTURE = true;
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
      }
  }

```

在系统启动完毕后会调用 systemReady() 执行相关的方法  
所以需要在 systemReady() 中设置默认的浏览器

```
import android.app.role.RoleManager;
import android.os.UserHandle;
import android.os.Process;
import java.util.concurrent.Executor;
import java.util.function.Consumer;
// 添加默认浏览器方法
public void setDefaultBrowser(Context context, String packageName) {
String roleName = "android.app.role.BROWSER";
boolean add = true;
int flags = 0;
UserHandle user = Process.myUserHandle();
Log.i("permission", (add ? "Adding" : "Removing") + " package as role holder, role: "
+ roleName + ", package: " + packageName);
RoleManager roleManager = context.getSystemService(RoleManager.class);
Executor executor = context.getMainExecutor();
Consumer<Boolean> callback = successful -> {
if (successful) {
Log.d("permission", "Package " + (add ? "added" : "removed")
+ " as role holder, role: " + roleName + ", package: " + packageName);
} else {
Log.d("permission", "Failed to " + (add ? "add" : "remove")
+ " package as role holder, role: " + roleName + ", package: "
+ packageName);
}
};
roleManager.addRoleHolderAsUser(roleName, packageName, flags, user, executor, callback);
}

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
        + setDefaultBrowser(mContext, "浏览器包名");
    }

```

这样我们就可以在 PhoneWindowManager 的 systemBooted() 方法调用 setDefaultBrowser 就可以了
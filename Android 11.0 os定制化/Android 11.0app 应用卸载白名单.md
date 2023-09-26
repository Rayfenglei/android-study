> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124889405)

### 1. 概述

在 11.0 定制化开发中, 客户需求要实现应用卸载白名单功能，用来管理第三方 app 卸载功能，需要在白名单之中的应用可以卸载，其他的 app 不准卸载，实现一个管理第三方 app 卸载的功能，这需要从 app 卸载流程入手就可以实现功能，而 [PMS](https://so.csdn.net/so/search?q=PMS&spm=1001.2101.3001.7020) 负责对 app 的安装和卸载功能管理所以从这里入手

### 2.app 应用卸载白名单的核心代码

```
frameworks/base/core/java/android/content/pm/IPackageManager.aidl
frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

```

### 3.app 应用卸载白名单的核心代码功能分析

实现 app 卸载白名单实现思路  
1 . IPackageManager.aidl 增加卸载白名单接口  
2. 找到系统安卸载 apk 核心代码，查询 app 包名列表，实施拦截  
安装卸载的核心代码都在 PackageManagerService.java 中

### 3.1IPackageManager.aidl 增加卸载 app 白名单接口

```
diff --git a/frameworks/base/core/java/android/content/pm/IPackageManager.aidl b/frameworks/base/core/java/android/content/pm/IPackageManager.aidl

--- a/frameworks/base/core/java/android/content/pm/IPackageManager.aidl

+++ b/frameworks/base/core/java/android/content/pm/IPackageManager.aidl

@@ -798,4 +798,7 @@ interface IPackageManager {

     */

     int restoreAppData(String sourceDir, String pkgName);

    /* @} */

+   

+       void setUnInstallPackageWhiteList(in List<String> packageNames);

+       List<String> getUnInstallPackageWhiteList();

 }

```

在 PackageManagerService.java 中添加卸载白名单的接口

```
diff --git a/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java b/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

index 45289f2e39..6727b10e35 100755

--- a/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

+++ b/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

@@ -111,7 +111,13 @@ import static com.android.server.pm.PackageManagerServiceUtils.getCompressedFile

 import static com.android.server.pm.PackageManagerServiceUtils.getLastModifiedTime;

 import static com.android.server.pm.PackageManagerServiceUtils.logCriticalInfo;

 import static com.android.server.pm.PackageManagerServiceUtils.verifySignatures;

 import android.Manifest;

 import android.annotation.IntDef;

 import android.annotation.NonNull;

@@ -2141,7 +2147,16 @@ public class PackageManagerService extends PackageManagerServiceExAbs

             }

         }

     }

-

+       private List<String> unInstallwhitepackageNames;

+           @Override

+    public void setInstallPackageWhiteList( List<String> packageNames) {

+               this.unInstallwhitepackageNames=packageNames;

+    }

+       

+       @Override

+    public List<String> getInstallPackageWhiteList(){

+               return this.unInstallwhitepackageNames;

+    }

```

### 3.2 在 PackageManagerService 中根据白名单拦截卸载功能

经过查询相关资料发现 PMS 的 deletePackageX() 负责对 app 的卸载功能做相关的管理

```
int deletePackageX(String packageName, long versionCode, int userId, int deleteFlags) {
          final PackageRemovedInfo info = new PackageRemovedInfo(this);
          final boolean res;
  
          final int removeUser = (deleteFlags & PackageManager.DELETE_ALL_USERS) != 0
                  ? UserHandle.USER_ALL : userId;
  
          if (isPackageDeviceAdmin(packageName, removeUser)) {
              Slog.w(TAG, "Not removing package " + packageName + ": has active device admin");
              return PackageManager.DELETE_FAILED_DEVICE_POLICY_MANAGER;
          }
  
          final PackageSetting uninstalledPs;
          final PackageSetting disabledSystemPs;
          final AndroidPackage pkg;
  
          // for the uninstall-updates case and restricted profiles, remember the per-
          // user handle installed state
          int[] allUsers;
          /** enabled state of the uninstalled application */
          final int origEnabledState;
          synchronized (mLock) {
              uninstalledPs = mSettings.mPackages.get(packageName);
              if (uninstalledPs == null) {
                  Slog.w(TAG, "Not removing non-existent package " + packageName);
                  return PackageManager.DELETE_FAILED_INTERNAL_ERROR;
              }
  
              if (versionCode != PackageManager.VERSION_CODE_HIGHEST
                      && uninstalledPs.versionCode != versionCode) {
                  Slog.w(TAG, "Not removing package " + packageName + " with versionCode "
                          + uninstalledPs.versionCode + " != " + versionCode);
                  return PackageManager.DELETE_FAILED_INTERNAL_ERROR;
              }
  
              disabledSystemPs = mSettings.getDisabledSystemPkgLPr(packageName);
              // Save the enabled state before we delete the package. When deleting a stub
              // application we always set the enabled state to 'disabled'.
              origEnabledState = uninstalledPs == null
                      ? COMPONENT_ENABLED_STATE_DEFAULT : uninstalledPs.getEnabled(userId);
              // Static shared libs can be declared by any package, so let us not
              // allow removing a package if it provides a lib others depend on.
              pkg = mPackages.get(packageName);
  
              allUsers = mUserManager.getUserIds();
  
              if (pkg != null && pkg.getStaticSharedLibName() != null) {
                  SharedLibraryInfo libraryInfo = getSharedLibraryInfoLPr(
                          pkg.getStaticSharedLibName(), pkg.getStaticSharedLibVersion());
                  if (libraryInfo != null) {
                      for (int currUserId : allUsers) {
                          if (removeUser != UserHandle.USER_ALL && removeUser != currUserId) {
                              continue;
                          }
                          List<VersionedPackage> libClientPackages = getPackagesUsingSharedLibraryLPr(
                                  libraryInfo, MATCH_KNOWN_PACKAGES, currUserId);
                          if (!ArrayUtils.isEmpty(libClientPackages)) {
                              Slog.w(TAG, "Not removing package " + pkg.getManifestPackageName()
                                      + " hosting lib " + libraryInfo.getName() + " version "
                                      + libraryInfo.getLongVersion() + " used by " + libClientPackages
                                      + " for user " + currUserId);
                              return PackageManager.DELETE_FAILED_USED_SHARED_LIBRARY;
                          }
                      }
                  }
              }
  
              info.origUsers = uninstalledPs.queryInstalledUsers(allUsers, true);
          }
  
          final int freezeUser;
          if (isUpdatedSystemApp(uninstalledPs)
                  && ((deleteFlags & PackageManager.DELETE_SYSTEM_APP) == 0)) {
              // We're downgrading a system app, which will apply to all users, so
              // freeze them all during the downgrade
              freezeUser = UserHandle.USER_ALL;
          } else {
              freezeUser = removeUser;
          }
  
          synchronized (mInstallLock) {
              if (DEBUG_REMOVE) Slog.d(TAG, "deletePackageX: pkg=" + packageName + " user=" + userId);
              try (PackageFreezer freezer = freezePackageForDelete(packageName, freezeUser,
                      deleteFlags, "deletePackageX")) {
                  res = deletePackageLIF(packageName, UserHandle.of(removeUser), true, allUsers,
                          deleteFlags | PackageManager.DELETE_CHATTY, info, true, null);
              }
              synchronized (mLock) {
                  if (res) {
                      if (pkg != null) {
                          mInstantAppRegistry.onPackageUninstalledLPw(pkg, uninstalledPs,
                                  info.removedUsers);
                      }
                      updateSequenceNumberLP(uninstalledPs, info.removedUsers);
                      updateInstantAppInstallerLocked(packageName);
                  }
              }
          }
  
          if (res) {
              final boolean killApp = (deleteFlags & PackageManager.DELETE_DONT_KILL_APP) == 0;
              info.sendPackageRemovedBroadcasts(killApp);
              info.sendSystemPackageUpdatedBroadcasts();
          }
          // Force a gc here.
          Runtime.getRuntime().gc();
          // Delete the resources here after sending the broadcast to let
          // other processes clean up before deleting resources.
          synchronized (mInstallLock) {
              if (info.args != null) {
                  info.args.doPostDeleteLI(true);
              }
              final AndroidPackage stubPkg =
                      (disabledSystemPs == null) ? null : disabledSystemPs.pkg;
              if (stubPkg != null && stubPkg.isStub()) {
                  final PackageSetting stubPs;
                  synchronized (mLock) {
                      // restore the enabled state of the stub; the state is overwritten when
                      // the stub is uninstalled
                      stubPs = mSettings.getPackageLPr(stubPkg.getPackageName());
                      if (stubPs != null) {
                          stubPs.setEnabled(origEnabledState, userId, "android");
                      }
                  }
                  if (origEnabledState == COMPONENT_ENABLED_STATE_DEFAULT
                          || origEnabledState == COMPONENT_ENABLED_STATE_ENABLED) {
                      if (DEBUG_COMPRESSION) {
                          Slog.i(TAG, "Enabling system stub after removal; pkg: "
                                  + stubPkg.getPackageName());
                      }
                      enableCompressedPackage(stubPkg, stubPs);
                  }
              }
          }
  
          return res ? PackageManager.DELETE_SUCCEEDED : PackageManager.DELETE_FAILED_INTERNAL_ERROR;
      }
  

```

不论是 adb 卸载还是通过系统设置拖拽卸载最终都会走 deletePackageX() 所以可以在这里判断是否是白名单的 app  
可以控制卸载功能修改如下：

```
    public int deletePackageX(String packageName, long versionCode, int userId, int deleteFlags) {
        final PackageRemovedInfo info = new PackageRemovedInfo(this);
        final boolean res;

        final int removeUser = (deleteFlags & PackageManager.DELETE_ALL_USERS) != 0
                ? UserHandle.USER_ALL : userId;

        if (isPackageDeviceAdmin(packageName, removeUser)) {
            Slog.w(TAG, "Not removing package " + packageName + ": has active device admin");
            return PackageManager.DELETE_FAILED_DEVICE_POLICY_MANAGER;
        }

        //  add for uninstaller black list start
       +  if(!isUninstallWhiteListApp(packageName)){
        +     return PackageManager.DELETE_FAILED_INTERNAL_ERROR;
        + }
        //  add for uninstaller black list end

        final PackageSetting uninstalledPs;
        final PackageSetting disabledSystemPs;
        final PackageParser.Package pkg;

        // for the uninstall-updates case and restricted profiles, remember the per-
        // user handle installed state
        int[] allUsers;


		......

}

+    private boolean isUninstallWhiteListApp(String packagename){

 

+               if(this.unInstallwhitepackageNames ==null || this.unInstallwhitepackageNames.size()==0){

+                       return true;

+               }

+               

+        Iterator<String> it = this.unInstallwhitepackageNames.iterator();

+        while (it.hasNext()) {

+            String whitelistItem = it.next();

+            if (whitelistItem.equals(packagename)) {

+                return true;

+            }

+        }

+        return false;

+    }

```
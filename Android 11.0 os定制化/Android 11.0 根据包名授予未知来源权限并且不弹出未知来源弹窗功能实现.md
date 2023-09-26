> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126980773)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 根据包名设置不弹出未知来源弹窗功能实现的相关代码](#t1)

[3. 根据包名设置不弹出未知来源弹窗功能分析和实现](#t2)

 [3.1 DefaultPermissionGrantPolicy.java 根据包名设置默认权限](#t3)

[3.2 PackageInstallerActivity.java 根据包名去掉未知来源弹窗部分功能实现](#t4)

**1. 概述**
---------

  在进行新产品开发中，系统默认对于安装第三方应用时，会弹出未知来源弹窗，点击确定后可以安装第三方 app 对于未知来源权限默认会根据包名来授予权限，在安装过程中不弹出未知来源弹窗

2. 根据包名设置不弹出未知来源弹窗功能实现的相关代码
---------------------------

```
   frameworks/base/services/core/java/com/android/server/pm/permission/DefaultPermissionGrantPolicy.java
   frameworks/base/packages/PackageInstaller/src/com/android/packageinstaller/PackageInstallerActivity.java
```

3. 根据包名设置不弹出未知来源弹窗功能分析和实现
-------------------------

###   3.1 DefaultPermissionGrantPolicy.java 根据包名设置默认权限

```
   public final class DefaultPermissionGrantPolicy {
      private static final String TAG = "DefaultPermGrantPolicy"; // must be <= 23 chars
      private static final boolean DEBUG = false;
  
      @PackageManager.ResolveInfoFlags
      private static final int DEFAULT_INTENT_QUERY_FLAGS =
              PackageManager.MATCH_DIRECT_BOOT_AWARE | PackageManager.MATCH_DIRECT_BOOT_UNAWARE
                      | PackageManager.MATCH_UNINSTALLED_PACKAGES;
  
      @PackageManager.PackageInfoFlags
      private static final int DEFAULT_PACKAGE_INFO_QUERY_FLAGS =
              PackageManager.MATCH_UNINSTALLED_PACKAGES
                      | PackageManager.MATCH_DISABLED_UNTIL_USED_COMPONENTS
                      | PackageManager.MATCH_HIDDEN_UNTIL_INSTALLED_COMPONENTS
                      | PackageManager.GET_PERMISSIONS;
  
      private static final String AUDIO_MIME_TYPE = "audio/mpeg";
  
      private static final String TAG_EXCEPTIONS = "exceptions";
      private static final String TAG_EXCEPTION = "exception";
      private static final String TAG_PERMISSION = "permission";
      private static final String ATTR_PACKAGE = "package";
      private static final String ATTR_NAME = "name";
      private static final String ATTR_FIXED = "fixed";
      private static final String ATTR_WHITELISTED = "whitelisted";
  
      private static final Set<String> PHONE_PERMISSIONS = new ArraySet<>();
  
  
      static {
          PHONE_PERMISSIONS.add(Manifest.permission.READ_PHONE_STATE);
          PHONE_PERMISSIONS.add(Manifest.permission.CALL_PHONE);
          PHONE_PERMISSIONS.add(Manifest.permission.READ_CALL_LOG);
          PHONE_PERMISSIONS.add(Manifest.permission.WRITE_CALL_LOG);
          PHONE_PERMISSIONS.add(Manifest.permission.ADD_VOICEMAIL);
          PHONE_PERMISSIONS.add(Manifest.permission.USE_SIP);
          PHONE_PERMISSIONS.add(Manifest.permission.PROCESS_OUTGOING_CALLS);
      }
}
```

static 部分是添加了 PHONE_PERMISSIONS 的默认权限  
所以要添加未知来源权限 需要定义为

```
private static final Set<String> UNKOWN_SOURCE_PERMISSIONS = new ArraySet<>();
      static {
          UNKOWN_SOURCE_PERMISSIONS.add(Manifest.permission.REQUEST_INSTALL_PACKAGES);
      }
 
private void grantRuntimePermissionsForSystemPackage(PackageManagerWrapper pm,
              int userId, PackageInfo pkg) {
          Set<String> permissions = new ArraySet<>();
          for (String permission : pkg.requestedPermissions) {
              final PermissionInfo perm = pm.getPermissionInfo(permission);
              if (perm == null) {
                  continue;
              }
              if (perm.isRuntime()) {
                  permissions.add(permission);
              }
          }
          if (!permissions.isEmpty()) {
              grantRuntimePermissions(pm, pkg, permissions, true /*systemFixed*/, userId);
          }
      }
  
      public void scheduleReadDefaultPermissionExceptions() {
          mHandler.sendEmptyMessage(MSG_READ_DEFAULT_PERMISSION_EXCEPTIONS);
      }
  
      private void grantPermissionsToSysComponentsAndPrivApps(DelayingPackageManagerCache pm,
              int userId) {
          Log.i(TAG, "Granting permissions to platform components for user " + userId);
          List<PackageInfo> packages = mContext.getPackageManager().getInstalledPackagesAsUser(
                  DEFAULT_PACKAGE_INFO_QUERY_FLAGS, UserHandle.USER_SYSTEM);
          for (PackageInfo pkg : packages) {
              if (pkg == null) {
                  continue;
              }
  
              // Package info is already loaded, cache it
              pm.addPackageInfo(pkg.packageName, pkg);
  
              if (!pm.isSysComponentOrPersistentPlatformSignedPrivApp(pkg)
                      || !doesPackageSupportRuntimePermissions(pkg)
                      || ArrayUtils.isEmpty(pkg.requestedPermissions)) {
                  continue;
              }
              grantRuntimePermissionsForSystemPackage(pm, userId, pkg);
          }
  
          // Grant READ_PHONE_STATE to all system apps that have READ_PRIVILEGED_PHONE_STATE
          for (PackageInfo pkg : packages) {
              if (pkg == null
                      || !doesPackageSupportRuntimePermissions(pkg)
                      || ArrayUtils.isEmpty(pkg.requestedPermissions)
                      || !pkg.applicationInfo.isPrivilegedApp()) {
                  continue;
              }
              for (String permission : pkg.requestedPermissions) {
                  if (Manifest.permission.READ_PRIVILEGED_PHONE_STATE.equals(permission)) {
                      grantRuntimePermissions(pm, pkg,
                              Collections.singleton(Manifest.permission.READ_PHONE_STATE),
                              true, // systemFixed
                              userId);
                  }
              }
          }
  
      }
  
 public void grantDefaultPermissions(int userId) {
          DelayingPackageManagerCache pm = new DelayingPackageManagerCache();
  
          grantPermissionsToSysComponentsAndPrivApps(pm, userId);
          grantDefaultSystemHandlerPermissions(pm, userId);
          grantDefaultPermissionExceptions(pm, userId);
  +     grantPermissionsToCusApp(userId);
          // Apply delayed state
          pm.apply();
      }
```

添加默认权限 所以添加自定义权限 就在这里添加根据包名的权限如下:  
定义包名和方法名实现

```
private static final String PG_NAME_CAMERA = "com.android.camera2";
private void grantPermissionsToCusApp(int userId){
    try{
        PackageInfo cameraPck = getPackageInfo(PG_NAME_CAMERA);
        if (cameraPck != null) {
            grantRuntimePermissions(cameraPck, UNKOWN_SOURCE_PERMISSIONS,false, userId);
            grantRuntimePermissions(cameraPck, STORAGE_PERMISSIONS,false, userId);
            grantRuntimePermissions(cameraPck, CONTACTS_PERMISSIONS,false, userId);
        }
    }catch(Exception e) {
        e.printStackTrace();
    }
}
```

通过上述代码实现给与 app 添加默认权限 根据包名添加未知来源权限

### 3.2 PackageInstallerActivity.java 根据包名去掉未知来源弹窗部分功能实现

```
public class PackageInstallerActivity extends AlertActivity {
      private static final String TAG = "PackageInstaller";
  
      private static final int REQUEST_TRUST_EXTERNAL_SOURCE = 1;
  
      static final String SCHEME_PACKAGE = "package";
  
      static final String EXTRA_CALLING_PACKAGE = "EXTRA_CALLING_PACKAGE";
      static final String EXTRA_ORIGINAL_SOURCE_INFO = "EXTRA_ORIGINAL_SOURCE_INFO";
      private static final String ALLOW_UNKNOWN_SOURCES_KEY =
              PackageInstallerActivity.class.getName() + "ALLOW_UNKNOWN_SOURCES_KEY";
private void showDialogInner(int id) {
          DialogFragment currentDialog =
                  (DialogFragment) getFragmentManager().findFragmentByTag("dialog");
          if (currentDialog != null) {
              currentDialog.dismissAllowingStateLoss();
          }
  
          DialogFragment newDialog = createDialog(id);
          if (newDialog != null) {
              newDialog.showAllowingStateLoss(getFragmentManager(), "dialog");
          }
      }
  
      /**
       * Create a new dialog.
       *
       * @param id The id of the dialog (determines dialog type)
       *
       * @return The dialog
       */
      private DialogFragment createDialog(int id) {
          switch (id) {
              case DLG_PACKAGE_ERROR:
                  return SimpleErrorDialog.newInstance(R.string.Parse_error_dlg_text);
              case DLG_OUT_OF_SPACE:
                  return OutOfSpaceDialog.newInstance(
                          mPm.getApplicationLabel(mPkgInfo.applicationInfo));
              case DLG_INSTALL_ERROR:
                  return InstallErrorDialog.newInstance(
                          mPm.getApplicationLabel(mPkgInfo.applicationInfo));
              case DLG_NOT_SUPPORTED_ON_WEAR:
                  return NotSupportedOnWearDialog.newInstance();
              case DLG_INSTALL_APPS_RESTRICTED_FOR_USER:
                  return SimpleErrorDialog.newInstance(
                          R.string.install_apps_user_restriction_dlg_text);
              case DLG_UNKNOWN_SOURCES_RESTRICTED_FOR_USER:
                  return SimpleErrorDialog.newInstance(
                          R.string.unknown_apps_user_restriction_dlg_text);
              case DLG_EXTERNAL_SOURCE_BLOCKED:
                  return ExternalSourcesBlockedDialog.newInstance(mOriginatingPackage);
              case DLG_ANONYMOUS_SOURCE:
                  return AnonymousSourceDialog.newInstance();
          }
          return null;
      }
  private void handleUnknownSources() {
          if (mOriginatingPackage == null) {
              Log.i(TAG, "No source found for package " + mPkgInfo.packageName);
              showDialogInner(DLG_ANONYMOUS_SOURCE);
              return;
          }
          // Shouldn't use static constant directly, see b/65534401.
          final int appOpCode =
                  AppOpsManager.permissionToOpCode(Manifest.permission.REQUEST_INSTALL_PACKAGES);
          final int appOpMode = mAppOpsManager.noteOpNoThrow(appOpCode,
                  mOriginatingUid, mOriginatingPackage);
          switch (appOpMode) {
              case AppOpsManager.MODE_DEFAULT:
                  mAppOpsManager.setMode(appOpCode, mOriginatingUid,
                          mOriginatingPackage, AppOpsManager.MODE_ERRORED);
                  // fall through
              case AppOpsManager.MODE_ERRORED:
                  showDialogInner(DLG_EXTERNAL_SOURCE_BLOCKED);
                  break;
              case AppOpsManager.MODE_ALLOWED:
                  initiateInstall();
                  break;
              default:
                  Log.e(TAG, "Invalid app op mode " + appOpMode
                          + " for OP_REQUEST_INSTALL_PACKAGES found for uid " + mOriginatingUid);
                  finish();
                  break;
          }
      }
    private void checkIfAllowedAndInitiateInstall() {
          // Check for install apps user restriction first.
          final int installAppsRestrictionSource = mUserManager.getUserRestrictionSource(
                  UserManager.DISALLOW_INSTALL_APPS, Process.myUserHandle());
          if ((installAppsRestrictionSource & UserManager.RESTRICTION_SOURCE_SYSTEM) != 0) {
              showDialogInner(DLG_INSTALL_APPS_RESTRICTED_FOR_USER);
              return;
          } else if (installAppsRestrictionSource != UserManager.RESTRICTION_NOT_SET) {
              startActivity(new Intent(Settings.ACTION_SHOW_ADMIN_SUPPORT_DETAILS));
              finish();
              return;
          }
  
          if (mAllowUnknownSources || !isInstallRequestFromUnknownSource(getIntent())) {
              initiateInstall();
          } else {
              // Check for unknown sources restrictions.
              final int unknownSourcesRestrictionSource = mUserManager.getUserRestrictionSource(
                      UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES, Process.myUserHandle());
              final int unknownSourcesGlobalRestrictionSource = mUserManager.getUserRestrictionSource(
                      UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES_GLOBALLY, Process.myUserHandle());
              final int systemRestriction = UserManager.RESTRICTION_SOURCE_SYSTEM
                      & (unknownSourcesRestrictionSource | unknownSourcesGlobalRestrictionSource);
              if (systemRestriction != 0) {
                  showDialogInner(DLG_UNKNOWN_SOURCES_RESTRICTED_FOR_USER);
              } else if (unknownSourcesRestrictionSource != UserManager.RESTRICTION_NOT_SET) {
                  startAdminSupportDetailsActivity(UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES);
              } else if (unknownSourcesGlobalRestrictionSource != UserManager.RESTRICTION_NOT_SET) {
                  startAdminSupportDetailsActivity(
                          UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES_GLOBALLY);
              } else {
                  handleUnknownSources();
              }
          }
      }
```

在安装过程中 会在进入 PackageInstallerActivity.java 中 调用 checkIfAllowedAndInitiateInstall() 检验是否  
是未知来源的第三方 app 然后调用 handleUnknownSources() 查询是否有权限  
而在 handleUnknownSources() 中的 mOriginatingPackage == null 会在为空的情况下 进入 showDialogInner(）  
弹出未知来源弹窗，而 mOriginatingPackage 在 Oncreate 中会判断是未知来源的话 为空  
在未知来源的时候会弹出               
调用 case DLG_ANONYMOUS_SOURCE:  
                  return AnonymousSourceDialog.newInstance();  
弹出未知来源弹窗  
所以需要在 createDialog(int id) 根据包名去掉弹窗直接进行安装  
修改如下:

```
     private DialogFragment createDialog(int id) {
          switch (id) {
              case DLG_PACKAGE_ERROR:
                  return SimpleErrorDialog.newInstance(R.string.Parse_error_dlg_text);
              case DLG_OUT_OF_SPACE:
                  return OutOfSpaceDialog.newInstance(
                          mPm.getApplicationLabel(mPkgInfo.applicationInfo));
              case DLG_INSTALL_ERROR:
                  return InstallErrorDialog.newInstance(
                          mPm.getApplicationLabel(mPkgInfo.applicationInfo));
              case DLG_NOT_SUPPORTED_ON_WEAR:
                  return NotSupportedOnWearDialog.newInstance();
              case DLG_INSTALL_APPS_RESTRICTED_FOR_USER:
                  return SimpleErrorDialog.newInstance(
                          R.string.install_apps_user_restriction_dlg_text);
              case DLG_UNKNOWN_SOURCES_RESTRICTED_FOR_USER:
                  return SimpleErrorDialog.newInstance(
                          R.string.unknown_apps_user_restriction_dlg_text);
              case DLG_EXTERNAL_SOURCE_BLOCKED:
                  return ExternalSourcesBlockedDialog.newInstance(mOriginatingPackage);
              case DLG_ANONYMOUS_SOURCE:
                 -  return AnonymousSourceDialog.newInstance();
                 + if(mPkgInfo.packageName.equals("包名")){
                 +            mAllowUnknownSources = true;
                 +             initiateInstall();
                 +}else{
                   + return AnonymousSourceDialog.newInstance();
                  }
          }
          return null;
      }
```
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/128524178)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 设置默认 Launcher 安装一款 Launcher 默认 Launcher 无效的解决方案的核心类](#t1)

[3. 设置默认 Launcher 安装一款 Launcher 默认 Launcher 无效的解决方案的核心功能分析和实现](#t2)

1. 概述
-----

 在 11.0 的系统产品开发中，对于设置默认 Launcher 的功能，也是常见的，但是在设置默认 Launcher 以后，工作开发测试中，发现安装另外安装一款  
Launcher 以后，发现设置的默认 Launcher 无效，在系统设置的默认 Launcher 应用中为未选择的状态，所以需要从安装 Launcher 的相关方法  
中分析，看做了哪些操作，然后修复这个功能

2. 设置默认 Launche[r 安装](https://so.csdn.net/so/search?q=r%E5%AE%89%E8%A3%85&spm=1001.2101.3001.7020)一款 Launcher 默认 Launcher 无效的解决方案的核心类
-------------------------------------------------------------------------------------------------------------------------------------

```
frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

```

3. 设置默认 Launcher 安装一款 Launcher 默认 Launcher 无效的解决方案的核心功能分析和实现
------------------------------------------------------------

```
void doHandleMessage(Message msg) {
              switch (msg.what) {
                  ....
                  case POST_INSTALL: {
......
  
                      if (data != null && data.mPostInstallRunnable != null) {
                          data.mPostInstallRunnable.run();
                      } else if (data != null && data.args != null) {
                          InstallArgs args = data.args;
                          PackageInstalledInfo parentRes = data.res;
  
                          final boolean grantPermissions = (args.installFlags
                                  & PackageManager.INSTALL_GRANT_RUNTIME_PERMISSIONS) != 0;
                          final boolean killApp = (args.installFlags
                                  & PackageManager.INSTALL_DONT_KILL_APP) == 0;
                          final boolean virtualPreload = ((args.installFlags
                                  & PackageManager.INSTALL_VIRTUAL_PRELOAD) != 0);
                          final String[] grantedPermissions = args.installGrantPermissions;
                          final List<String> whitelistedRestrictedPermissions = ((args.installFlags
                                  & PackageManager.INSTALL_ALL_WHITELIST_RESTRICTED_PERMISSIONS) != 0
                                      && parentRes.pkg != null)
                                  ? parentRes.pkg.getRequestedPermissions()
                                  : args.whitelistedRestrictedPermissions;
                          int autoRevokePermissionsMode = args.autoRevokePermissionsMode;
  
                          handlePackagePostInstall(parentRes, grantPermissions,
                                  killApp, virtualPreload, grantedPermissions,
                                  whitelistedRestrictedPermissions, autoRevokePermissionsMode,
                                  didRestore, args.installSource.installerPackageName, args.observer,
                                  args.mDataLoaderType);
  
                          // Log tracing if needed
                          if (args.traceMethod != null) {
                              Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, args.traceMethod,
                                      args.traceCookie);
                          }
                      } else if (DEBUG_INSTALL) {
                          // No post-install when we run restore from installExistingPackageForUser
                          Slog.i(TAG, "Nothing to do for post-install token " + msg.arg1);
                      }
  
                      Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "postInstall", msg.arg1);
                  } break;
```

在 PackageManagerService.java 的上述方法中可以看出，在 doHandleMessage(Message msg) 的方法中的 POST_INSTALL 安装默认 Launcherapp 的过程中  
会调用 handlePackagePostInstall(parentRes, grantPermissions,  
                                  killApp, virtualPreload, grantedPermissions,  
                                  whitelistedRestrictedPermissions, autoRevokePermissionsMode,  
                                  didRestore, args.installSource.installerPackageName, args.[observer](https://so.csdn.net/so/search?q=observer&spm=1001.2101.3001.7020),  
                                  args.mDataLoaderType);  
进行相关的安装 Launcher 的 app 的处理工作，接下来看下 handlePackagePostInstall(parentRes, grantPermissions,  
                                  killApp, virtualPreload, grantedPermissions,  
                                  whitelistedRestrictedPermissions, autoRevokePermissionsMode,  
                                  didRestore, args.installSource.installerPackageName, args.observer,  
                                  args.mDataLoaderType); 的相关方法

```
private void handlePackagePostInstall(PackageInstalledInfo res, boolean grantPermissions,
              boolean killApp, boolean virtualPreload,
              String[] grantedPermissions, List<String> whitelistedRestrictedPermissions,
              int autoRevokePermissionsMode,
              boolean launchedForRestore, String installerPackage,
              IPackageInstallObserver2 installObserver, int dataLoaderType) {
....
  
              // Send installed broadcasts if the package is not a static shared lib.
              if (res.pkg.getStaticSharedLibName() == null) {
                  mProcessLoggingHandler.invalidateProcessLoggingBaseApkHash(
                          res.pkg.getBaseCodePath());
  
                  // Send added for users that see the package for the first time
                  // sendPackageAddedForNewUsers also deals with system apps
                  int appId = UserHandle.getAppId(res.uid);
                  boolean isSystem = res.pkg.isSystem();
                  sendPackageAddedForNewUsers(packageName, isSystem || virtualPreload,
                          virtualPreload /*startReceiver*/, appId, firstUserIds, firstInstantUserIds,
                          dataLoaderType);
  
                  // Send added for users that don't see the package for the first time
                  Bundle extras = new Bundle(1);
                  extras.putInt(Intent.EXTRA_UID, res.uid);
                  if (update) {
                      extras.putBoolean(Intent.EXTRA_REPLACING, true);
                  }
                  extras.putInt(PackageInstaller.EXTRA_DATA_LOADER_TYPE, dataLoaderType);
                  // Send to all running apps.
                  final SparseArray<int[]> newBroadcastWhitelist;
  
                  synchronized (mLock) {
                      newBroadcastWhitelist = mAppsFilter.getVisibilityWhitelist(
                              getPackageSettingInternal(res.name, Process.SYSTEM_UID),
                              updateUserIds, mSettings.mPackages);
                  }
                  sendPackageBroadcast(Intent.ACTION_PACKAGE_ADDED, packageName,
                          extras, 0 /*flags*/,
                          null /*targetPackage*/, null /*finishedReceiver*/,
                          updateUserIds, instantUserIds, newBroadcastWhitelist);
                  if (installerPackageName != null) {
                      // Send to the installer, even if it's not running.
                      sendPackageBroadcast(Intent.ACTION_PACKAGE_ADDED, packageName,
                              extras, 0 /*flags*/,
                              installerPackageName, null /*finishedReceiver*/,
                              updateUserIds, instantUserIds, null /* broadcastWhitelist */);
                  }
                  // if the required verifier is defined, but, is not the installer of record
                  // for the package, it gets notified
                  final boolean notifyVerifier = mRequiredVerifierPackage != null
                          && !mRequiredVerifierPackage.equals(installerPackageName);
                  if (notifyVerifier) {
                      sendPackageBroadcast(Intent.ACTION_PACKAGE_ADDED, packageName,
                              extras, 0 /*flags*/,
                              mRequiredVerifierPackage, null /*finishedReceiver*/,
                              updateUserIds, instantUserIds, null /* broadcastWhitelist */);
                  }
                  // If package installer is defined, notify package installer about new
                  // app installed
                  if (mRequiredInstallerPackage != null) {
                      sendPackageBroadcast(Intent.ACTION_PACKAGE_ADDED, packageName,
                              extras, Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND /*flags*/,
                              mRequiredInstallerPackage, null /*finishedReceiver*/,
                              firstUserIds, instantUserIds, null /* broadcastWhitelist */);
                  }
  
                  // Send replaced for users that don't see the package for the first time
                  if (update) {
                      sendPackageBroadcast(Intent.ACTION_PACKAGE_REPLACED,
                              packageName, extras, 0 /*flags*/,
                              null /*targetPackage*/, null /*finishedReceiver*/,
                              updateUserIds, instantUserIds, res.removedInfo.broadcastWhitelist);
                      if (installerPackageName != null) {
                          sendPackageBroadcast(Intent.ACTION_PACKAGE_REPLACED, packageName,
                                  extras, 0 /*flags*/,
                                  installerPackageName, null /*finishedReceiver*/,
                                  updateUserIds, instantUserIds, null /*broadcastWhitelist*/);
                      }
                      if (notifyVerifier) {
                          sendPackageBroadcast(Intent.ACTION_PACKAGE_REPLACED, packageName,
                                  extras, 0 /*flags*/,
                                  mRequiredVerifierPackage, null /*finishedReceiver*/,
                                  updateUserIds, instantUserIds, null /*broadcastWhitelist*/);
                      }
                      sendPackageBroadcast(Intent.ACTION_MY_PACKAGE_REPLACED,
                              null /*package*/, null /*extras*/, 0 /*flags*/,
                              packageName /*targetPackage*/,
                              null /*finishedReceiver*/, updateUserIds, instantUserIds,
                              null /*broadcastWhitelist*/);
                  } else if (launchedForRestore && !res.pkg.isSystem()) {
                      // First-install and we did a restore, so we're responsible for the
                      // first-launch broadcast.
                      if (DEBUG_BACKUP) {
                          Slog.i(TAG, "Post-restore of " + packageName
                                  + " sending FIRST_LAUNCH in " + Arrays.toString(firstUserIds));
                      }
                      sendFirstLaunchBroadcast(packageName, installerPackage,
                              firstUserIds, firstInstantUserIds);
                  }
  
                  // Send broadcast package appeared if external for all users
                  if (res.pkg.isExternalStorage()) {
                      if (!update) {
                          final StorageManager storage = mInjector.getStorageManager();
                          VolumeInfo volume =
                                  storage.findVolumeByUuid(
                                          res.pkg.getStorageUuid().toString());
                          int packageExternalStorageType =
                                  getPackageExternalStorageType(volume, res.pkg.isExternalStorage());
                          // If the package was installed externally, log it.
                          if (packageExternalStorageType != StorageEnums.UNKNOWN) {
                              FrameworkStatsLog.write(
                                      FrameworkStatsLog.APP_INSTALL_ON_EXTERNAL_STORAGE_REPORTED,
                                      packageExternalStorageType, packageName);
                          }
                      }
                      if (DEBUG_INSTALL) {
                          Slog.i(TAG, "upgrading pkg " + res.pkg + " is external");
                      }
                      final int[] uidArray = new int[]{res.pkg.getUid()};
                      ArrayList<String> pkgList = new ArrayList<>(1);
                      pkgList.add(packageName);
                      sendResourcesChangedBroadcast(true, true, pkgList, uidArray, null);
                  }
              } else if (!ArrayUtils.isEmpty(res.libraryConsumers)) { // if static shared lib
                  int[] allUsers = mInjector.getUserManagerService().getUserIds();
                  for (int i = 0; i < res.libraryConsumers.size(); i++) {
                      AndroidPackage pkg = res.libraryConsumers.get(i);
                      // send broadcast that all consumers of the static shared library have changed
                      sendPackageChangedBroadcast(pkg.getPackageName(), false /* dontKillApp */,
                              new ArrayList<>(Collections.singletonList(pkg.getPackageName())),
                              pkg.getUid(), null);
                  }
              }
  
              // Work that needs to happen on first install within each user
              if (firstUserIds != null && firstUserIds.length > 0) {
                  for (int userId : firstUserIds) {
                      clearRolesAndRestorePermissionsForNewUserInstall(packageName,
                              pkgSetting.getInstallReason(userId), userId);
                  }
              }
 
.....
      }
```

在 PackageManagerService.java 的上述方法中可以看出 handlePackagePostInstall 中，在安装成功后，会调用  
             if (firstUserIds != null && firstUserIds.length> 0) {  
                  for (int userId : firstUserIds) {  
                      clearRolesAndRestorePermissionsForNewUserInstall(packageName,  
                              pkgSetting.getInstallReason(userId), userId);  
                  }  
              }  
来清理掉一些默认浏览器 [sms](https://so.csdn.net/so/search?q=sms&spm=1001.2101.3001.7020) launcher 等相关的操作，所以需要看 clearRolesAndRestorePermissionsForNewUserInstall(  
来分析具体的功能看下是如何来清理默认 Launcher sms 浏览器等等功能

```
  private void clearRolesAndRestorePermissionsForNewUserInstall(String packageName,
              int installReason, @UserIdInt int userId) {
          // If this app is a browser and it's newly-installed for some
          // users, clear any default-browser state in those users. The
          // app's nature doesn't depend on the user, so we can just check
          // its browser nature in any user and generalize.
          if (packageIsBrowser(packageName, userId)) {
              // If this browser is restored from user's backup, do not clear
              // default-browser state for this user
             if (installReason != PackageManager.INSTALL_REASON_DEVICE_RESTORE) {
                  mPermissionManager.setDefaultBrowser(null, true, true, userId);
              }
          }
  
          // We may also need to apply pending (restored) runtime permission grants
          // within these users.
          mPermissionManager.restoreDelayedRuntimePermissions(packageName,
                  UserHandle.of(userId));
  
          // Persistent preferred activity might have came into effect due to this
          // install.
          updateDefaultHomeNotLocked(userId);
      }
```

在 PackageManagerService.java 的上述方法中可以看出在 clearRolesAndRestorePermissionsForNewUserInstall 的方法中，会  
调用 updateDefaultHomeNotLocked(userId); 来设置默认 Launcher 等功能，所以需要看是如何处理默认 Launcher 的功能，接下来  
看下 updateDefaultHomeNotLocked(userId); 的相关方法分析

```
private boolean updateDefaultHomeNotLocked(int userId) {
          if (Thread.holdsLock(mLock)) {
              Slog.wtf(TAG, "Calling thread " + Thread.currentThread().getName()
                      + " is holding mLock", new Throwable());
          }
          if (!mSystemReady) {
              // We might get called before system is ready because of package changes etc, but
              // finding preferred activity depends on settings provider, so we ignore the update
              // before that.
              return false;
          }
          final Intent intent = getHomeIntent();
          final List<ResolveInfo> resolveInfos = queryIntentActivitiesInternal(intent, null,
                  PackageManager.GET_META_DATA, userId);
          final ResolveInfo preferredResolveInfo = findPreferredActivityNotLocked(
                  intent, null, 0, resolveInfos, 0, true, false, false, userId);
          final String packageName = preferredResolveInfo != null
                  && preferredResolveInfo.activityInfo != null
                  ? preferredResolveInfo.activityInfo.packageName : null;
          final String currentPackageName = mPermissionManager.getDefaultHome(userId);
          if (TextUtils.equals(currentPackageName, packageName)) {
              return false;
          }
          final String[] callingPackages = getPackagesForUid(Binder.getCallingUid());
          if (callingPackages != null && ArrayUtils.contains(callingPackages,
                  mRequiredPermissionControllerPackage)) {
              // PermissionController manages default home directly.
              return false;
          }
        +  if(packageName!=null)
          mPermissionManager.setDefaultHome(packageName, userId, (successful) -> {
              if (successful) {
                  postPreferredActivityChangedBroadcast(userId);
              }
          });
          return true;
      }
```

在 PackageManagerService.java 的上述代码中可以看出，在 updateDefaultHomeNotLocked(int userId) 设置默认 Launcher 的方法中  
调用 mPermissionManager.setDefaultHome 来设置默认 Launcher. 但是在打印日志发现当 packageName 为空的时候，也会设置默认  
Launcher, 所以会在安装 Launcher 的时候导致默认 Launcher 值为空的情况，所以增加判断 if(packageName!=null) 来设置默认 Launcher  
发现实现了功能，解决了问题所在
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124828029)

在 11.0 定制开发中第三方 app 第一次进入的时候 会弹出授予权限的权限框 有时候觉得挺麻烦的，所以根据客户要求  
默认授予权限，这时我们就需要根据包名 PackageName 来给与所需要的权限

首选来看下 framework/base/services/core/java/com/android/server/pm/PackageManagerService.java

PackageManagerService.java  
1 管理系统的 jar 包和 apk，负责系统权限  
2 负责程序的安装，卸载，更新，解析  
3 对于其他应用和服务提供安装卸载服务

从 PackageMangerService.java 的作用看 负责 app 的安装和权限等工作

```
 @Override
    public void systemReady() {
        enforceSystemOrRoot("Only the system can claim the system is ready");

        mSystemReady = true;
        final ContentResolver resolver = mContext.getContentResolver();
        ContentObserver co = new ContentObserver(mHandler) {
            @Override
            public void onChange(boolean selfChange) {
                final boolean ephemeralFeatureDisabled =
                        Global.getInt(resolver, Global.ENABLE_EPHEMERAL_FEATURE, 1) == 0;
                for (int userId : UserManagerService.getInstance().getUserIds()) {
                    final boolean instantAppsDisabledForUser =
                            ephemeralFeatureDisabled || Secure.getIntForUser(resolver,
                                    Secure.INSTANT_APPS_ENABLED, 1, userId) == 0;
                    mWebInstantAppsDisabled.put(userId, instantAppsDisabledForUser);
                }
            }
        };
        mContext.getContentResolver().registerContentObserver(android.provider.Settings.Global
                        .getUriFor(Global.ENABLE_EPHEMERAL_FEATURE),
                false, co, UserHandle.USER_ALL);
        mContext.getContentResolver().registerContentObserver(android.provider.Settings.Secure
                        .getUriFor(Secure.INSTANT_APPS_ENABLED), false, co, UserHandle.USER_ALL);
        co.onChange(true);

        // Disable any carrier apps. We do this very early in boot to prevent the apps from being
        // disabled after already being started.
        CarrierAppUtils.disableCarrierAppsUntilPrivileged(mContext.getOpPackageName(), this,
                mContext.getContentResolver(), UserHandle.USER_SYSTEM);

        disableSkuSpecificApps();

        // Read the compatibilty setting when the system is ready.
        boolean compatibilityModeEnabled = android.provider.Settings.Global.getInt(
                mContext.getContentResolver(),
                android.provider.Settings.Global.COMPATIBILITY_MODE, 1) == 1;
        PackageParser.setCompatibilityModeEnabled(compatibilityModeEnabled);

        if (DEBUG_SETTINGS) {
            Log.d(TAG, "compatibility mode:" + compatibilityModeEnabled);
        }

        int[] grantPermissionsUserIds = EMPTY_INT_ARRAY;

        synchronized (mPackages) {
            // Verify that all of the preferred activity components actually
            // exist.  It is possible for applications to be updated and at
            // that point remove a previously declared activity component that
            // had been set as a preferred activity.  We try to clean this up
            // the next time we encounter that preferred activity, but it is
            // possible for the user flow to never be able to return to that
            // situation so here we do a sanity check to make sure we haven't
            // left any junk around.
            ArrayList<PreferredActivity> removed = new ArrayList<>();
            for (int i=0; i<mSettings.mPreferredActivities.size(); i++) {
                PreferredIntentResolver pir = mSettings.mPreferredActivities.valueAt(i);
                removed.clear();
                for (PreferredActivity pa : pir.filterSet()) {
                    if (!mComponentResolver.isActivityDefined(pa.mPref.mComponent)) {
                        removed.add(pa);
                    }
                }
                if (removed.size() > 0) {
                    for (int r=0; r<removed.size(); r++) {
                        PreferredActivity pa = removed.get(r);
                        Slog.w(TAG, "Removing dangling preferred activity: "
                                + pa.mPref.mComponent);
                        pir.removeFilter(pa);
                    }
                    mSettings.writePackageRestrictionsLPr(
                            mSettings.mPreferredActivities.keyAt(i));
                }
            }

            for (int userId : UserManagerService.getInstance().getUserIds()) {
                if (!mSettings.areDefaultRuntimePermissionsGrantedLPr(userId)) {
                    grantPermissionsUserIds = ArrayUtils.appendInt(
                            grantPermissionsUserIds, userId);
                }
            }
        }

        sUserManager.systemReady();

        getStorageManagerInternal().addExternalStoragePolicy(
                new StorageManagerInternal.ExternalStorageMountPolicy() {
                    @Override
                    public int getMountMode(int uid, String packageName) {
                        if (Process.isIsolated(uid)) {
                            return Zygote.MOUNT_EXTERNAL_NONE;
                        }
                        if (checkUidPermission(READ_EXTERNAL_STORAGE, uid) == PERMISSION_DENIED) {
                            return Zygote.MOUNT_EXTERNAL_DEFAULT;
                        }
                        if (checkUidPermission(WRITE_EXTERNAL_STORAGE, uid) == PERMISSION_DENIED) {
                            return Zygote.MOUNT_EXTERNAL_READ;
                        }
                        return Zygote.MOUNT_EXTERNAL_WRITE;
                    }

                    @Override
                    public boolean hasExternalStorage(int uid, String packageName) {
                        return true;
                    }
                });

        // If we upgraded grant all default permissions before kicking off.
        for (int userId : grantPermissionsUserIds) {
            mDefaultPermissionPolicy.grantDefaultPermissions(userId);
        }

        if (grantPermissionsUserIds == EMPTY_INT_ARRAY) {
            // If we did not grant default permissions, we preload from this the
            // default permission exceptions lazily to ensure we don't hit the
            // disk on a new user creation.
            mDefaultPermissionPolicy.scheduleReadDefaultPermissionExceptions();
        }

        // Now that we've scanned all packages, and granted any default
        // permissions, ensure permissions are updated. Beware of dragons if you
        // try optimizing this.
        synchronized (mPackages) {
            mPermissionManager.updateAllPermissions(
                    StorageManager.UUID_PRIVATE_INTERNAL, false, mPackages.values(),
                    mPermissionCallback);

            final PermissionPolicyInternal permissionPolicyInternal =
                    LocalServices.getService(PermissionPolicyInternal.class);
            permissionPolicyInternal.setOnInitializedCallback(userId -> {
                // The SDK updated case is already handled when we run during the ctor.
                synchronized (mPackages) {
                    mPermissionManager.updateAllPermissions(
                            StorageManager.UUID_PRIVATE_INTERNAL, false /*sdkUpdated*/,
                            mPackages.values(), mPermissionCallback);
                }
            });
        }

        // Watch for external volumes that come and go over time
        final StorageManager storage = mContext.getSystemService(StorageManager.class);
        storage.registerListener(mStorageListener);

        mInstallerService.systemReady();
        mApexManager.systemReady();
        mPackageDexOptimizer.systemReady();

        // Now that we're mostly running, clean up stale users and apps
        sUserManager.reconcileUsers(StorageManager.UUID_PRIVATE_INTERNAL);
        reconcileApps(StorageManager.UUID_PRIVATE_INTERNAL);

        mPermissionManager.systemReady();

        if (mInstantAppResolverConnection != null) {
            mContext.registerReceiver(new BroadcastReceiver() {
                @Override
                public void onReceive(Context context, Intent intent) {
                    mInstantAppResolverConnection.optimisticBind();
                    mContext.unregisterReceiver(this);
                }
            }, new IntentFilter(Intent.ACTION_BOOT_COMPLETED));
        }

        mModuleInfoProvider.systemReady();

        // Installer service might attempt to install some packages that have been staged for
        // installation on reboot. Make sure this is the last component to be call since the
        // installation might require other components to be ready.
        mInstallerService.restoreAndApplyStagedSessionIfNeeded();
    }

```

在系统启动成功后 来通过

```
int[] grantPermissionsUserIds = EMPTY_INT_ARRAY;
    // If we upgraded grant all default permissions before kicking off.
    for (int userId : grantPermissionsUserIds) {
        mDefaultPermissionPolicy.grantDefaultPermissions(userId);
    }

```

来负责给 app 授予权限  
final DefaultPermissionGrantPolicy mDefaultPermissionPolicy;  
实际上调用的是 DefaultPermissionGrantPolicy 的 grantDefaultPermissions(userId) 方法

接下来看下 DefaultPermissionGrantPolicy 的 grantDefaultPermissions(userId) 方法

```
public void grantDefaultPermissions(int userId) {
    grantPermissionsToSysComponentsAndPrivApps(userId);
    grantDefaultSystemHandlerPermissions(userId);
    grantDefaultPermissionExceptions(userId);
    synchronized (mLock) {
        mDefaultPermissionsGrantedUsers.put(userId, userId);
    }
}

```

来通过 app 的 userId 来授予 app 所需要的权限

所以要想给第三方 app 授予权限 就在这里添加自定义的授权方法即可

添加如下:

// 声明要赋予 apk 包名

```
private static final String PCK_NAME_MICROSOFT = "com.android.contacts";
private static final String PCK_NAME_TTS = "com.example.edcationcloud";
private static final String PCK_NAME_SEARCH = "com.android.dialer";
private static final String PCK_NAME_CAMERA = "com.android.camera2";
private static final String PCK_NAME_GALLERY = "com.android.gallery3d"; 
private void grantPermissionsToCustomApp(int userId){
    try{
        PackageInfo contactPck = getPackageInfo(PCK_NAME_MICROSOFT);
        if (contactPck != null) {
            grantRuntimePermissions(contactPck, ALWAYS_LOCATION_PERMISSIONS,false, userId);
            grantRuntimePermissions(contactPck, STORAGE_PERMISSIONS,false, userId);
            grantRuntimePermissions(contactPck, CONTACTS_PERMISSIONS,false, userId);
        }

        PackageInfo clundPck = getPackageInfo(PCK_NAME_TTS);
        if (clundPck != null) {
            grantRuntimePermissions(clundPck, ALWAYS_LOCATION_PERMISSIONS,false, userId);
            grantRuntimePermissions(clundPck, STORAGE_PERMISSIONS,false, userId);
            grantRuntimePermissions(clundPck, MICROPHONE_PERMISSIONS,false, userId);
        }

       PackageInfo dialerPck = getPackageInfo(PCK_NAME_SEARCH);
        if (dialerPck != null) {
            grantRuntimePermissions(dialerPck, ALWAYS_LOCATION_PERMISSIONS,false, userId);
            grantRuntimePermissions(dialerPck, STORAGE_PERMISSIONS,false, userId);
            grantRuntimePermissions(dialerPck, MICROPHONE_PERMISSIONS,false, userId);
        }
       PackageInfo cameraPck = getPackageInfo(PCK_NAME_CAMERA);
        if (cameraPck != null) {
            grantRuntimePermissions(cameraPck, ALWAYS_LOCATION_PERMISSIONS,false, userId);
            grantRuntimePermissions(cameraPck, STORAGE_PERMISSIONS,false, userId);
            grantRuntimePermissions(cameraPck, MICROPHONE_PERMISSIONS,false, userId);
            grantRuntimePermissions(cameraPck, CAMERA_PERMISSIONS,false, userId);
        }
       PackageInfo galleryPck = getPackageInfo(PCK_NAME_GALLERY);
        if (galleryPck != null) {
            grantRuntimePermissions(galleryPck, ALWAYS_LOCATION_PERMISSIONS,false, userId);
            grantRuntimePermissions(galleryPck, STORAGE_PERMISSIONS,false, userId);
            grantRuntimePermissions(galleryPck, MICROPHONE_PERMISSIONS,false, userId);
            grantRuntimePermissions(galleryPck, CAMERA_PERMISSIONS,false, userId);
            grantRuntimePermissions(galleryPck, SMS_PERMISSIONS,false, userId);
			}
    }catch(Exception e) {
        e.printStackTrace();
    }
}

```

在 grantDefaultPermissions(int userId) 中，添加 grantPermissionsToCustomApp(int userId) 方法

修改如下：

```
  public void grantDefaultPermissions(int userId) {
        grantPermissionsToSysComponentsAndPrivApps(userId);
        grantDefaultSystemHandlerPermissions(userId);
        + grantPermissionsToCustomApp(userId);
        grantDefaultPermissionExceptions(userId);
        synchronized (mLock) {
            mDefaultPermissionsGrantedUsers.put(userId, userId);
        }
    }

```

然后编译验证 发现 app 已经授权了
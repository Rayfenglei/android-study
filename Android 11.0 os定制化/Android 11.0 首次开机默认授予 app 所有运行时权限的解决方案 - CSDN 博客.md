> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/128569933)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 首次开机默认授予 app 所有运行时权限的解决方案的核心类](#t1)

[3. 首次开机默认授予 app 所有运行时权限的解决方案的核心功能分析和实现](#t2)

 [3.1 PackageManagerService.java 中关于增加 DefaultPermissionGrantPolicy.java 的接口来授予 app 的所有权限](#t3)

[3.2 DefaultPermissionGrantPolicy.java 中关于对所有 app 默认所有运行时权限的功能实现](#t4)

1. 概述
-----

  在 11.0 的系统产品开发中，对于系统首次启动后，app 在首次运行时，会弹出授权窗口，  
会让手动授予 app 运行时权限，在由于系统产品开发需要要求默认授予 app 运行时权限  
所以需要在首次开机默认授予所有 app 运行时权限

2. 首次开机默认授予 app 所有运行时权限的解决方案的核心类
--------------------------------

```
frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java
frameworks/base/services/core/java/com/android/server/pm/permission/DefaultPermissionGrantPolicy.java
```

3. 首次开机默认授予 app 所有运行时权限的解决方案的核心功能分析和实现
--------------------------------------

  在系统首次启动的过程中，会在系统 [PMS](https://so.csdn.net/so/search?q=PMS&spm=1001.2101.3001.7020) 来负责 app 的安装 卸载 授权等等，关于授予默认的  
权限会在 DefaultPermissionGrantPolicy.java 中根据要求授予一些权限，那么所以首次开机  
授予 app 的默认权限，也可以在 DefaultPermissionGrantPolicy.java 中添加一些接口来实现  
对 app 的默认授予权限

  3.1 PackageManagerService.java 中关于增加 DefaultPermissionGrantPolicy.java 的  
接口来授予 app 的所有权限
--------------------------------------------------------------------------------------------

   首选增加 DefaultPermissionGrantPolicy.java 的实例如下:

```
   public class PackageManagerService extends IPackageManager.Stub
        implements PackageSender {
.....
       final boolean mIsPreQUpgrade;
 
    @GuardedBy("mLock")
    private boolean mDexOptDialogShown;
    // TODO remove this and go through mPermissonManager directly
+    final DefaultPermissionGrantPolicy mDefaultPermissionPolicy;
....
```

  然后在 PackageManagerService 的构造方法中，实例化 mDefaultPermissionPolicy 参数  
具体实现如下:

```
    public PackageManagerService(Injector injector, boolean onlyCore, boolean factoryTest) {
        PackageManager.disableApplicationInfoCache();
        PackageManager.disablePackageInfoCache();
 
        // Avoid invalidation-thrashing by preventing cache invalidations from causing property
        // writes if the cache isn't enabled yet.  We re-enable writes later when we're
        // done initializing.
        PackageManager.corkPackageInfoCache();
 
        final TimingsTraceAndSlog t = new TimingsTraceAndSlog(TAG + "Timing",
                Trace.TRACE_TAG_PACKAGE_MANAGER);
        mPendingBroadcasts = new PendingPackageBroadcasts();
....
       +         mDefaultPermissionPolicy = mPermissionManager.getDefaultPermissionGrantPolicy();
....
}
 
    /**
     * A extremely minimal constructor designed to start up a PackageManagerService instance for
     * testing.
     *
     * It is assumed that all methods under test will mock the internal fields and thus
     * none of the initialization is needed.
     */
    @VisibleForTesting(visibility = VisibleForTesting.Visibility.PRIVATE)
    public PackageManagerService(@NonNull Injector injector, @NonNull TestParams testParams) {
        mInjector = injector;
        mInjector.bootstrap(this);
        mAppsFilter = injector.getAppsFilter();
        mComponentResolver = injector.getComponentResolver();
        mContext = injector.getContext();
        mInstaller = injector.getInstaller();
        mInstallLock = injector.getInstallLock();
        mLock = injector.getLock();
+        mPermissionManager = injector.getPermissionManagerServiceInternal();
        mSettings = injector.getSettings();
        mUserManager = injector.getUserManagerService();
        mDefaultPermissionPolicy = mPermissionManager.getDefaultPermissionGrantPolicy();
        mApexManager = testParams.apexManager;
        mArtManagerService = testParams.artManagerService;
....
}
```

通过在 PackageManagerService 的上述方法中的增加 mDefaultPermissionPolicy 的实例，然后在  
构造方法中实现了 mDefaultPermissionPolicy 的实例，接下来需要在 systemReady() 中  
在 app 安装完成后给 app 授予运行时权限 具体实现如下：

```
@Override
    public void systemReady() {
        enforceSystemOrRoot("Only the system can claim the system is ready");
....
 
        mInjector.getStorageManagerInternal().addExternalStoragePolicy(
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
 
// add core start 
        mHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
               mDefaultPermissionPolicy.grantAllRuntimePermissions();
            }
        },5000);
// add core end
 
 
        // Now that we're mostly running, clean up stale users and apps
        mUserManager.reconcileUsers(StorageManager.UUID_PRIVATE_INTERNAL);
        reconcileApps(StorageManager.UUID_PRIVATE_INTERNAL);
        mPermissionManager.systemReady();
....
}
```

通过 PackageManagerService 的上述的 systemReady() 的方法中，增加 mDefaultPermissionPolicy  
的授权 app 运行时权限，来实现在开机启动完成后授予所有 app 运行时权限，这样就不会弹出  
首次启动授权运行时权限的弹窗了，接下来就需要在 mDefaultPermissionPolicy 中增加  
grantAllRuntimePermissions() 来实现对所有 app 的授权

3.2 DefaultPermissionGrantPolicy.java 中关于对所有 app 默认所有运行时权限的功能实现
---------------------------------------------------------------

```
public final class DefaultPermissionGrantPolicy {
    private static final String TAG = "DefaultPermGrantPolicy"; // must be <= 23 chars
    private static final boolean DEBUG = false;
    public void grantDefaultPermissions(int userId) {
        DelayingPackageManagerCache pm = new DelayingPackageManagerCache();
 
        grantPermissionsToSysComponentsAndPrivApps(pm, userId);
        grantDefaultSystemHandlerPermissions(pm, userId);
        grantDefaultPermissionExceptions(pm, userId);
 
        // Apply delayed state
        pm.apply();
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
```

通过 DefaultPermissionGrantPolicy.java 的 grantDefaultPermissions(int userId) 和 grantRuntimePermissionsForSystemPackage(PackageManagerWrapper pm,  
            int userId, PackageInfo [pkg](https://so.csdn.net/so/search?q=pkg&spm=1001.2101.3001.7020)) 是授予一些 app 的运行时权限，所有可以根据这两个方法来增加授予所有 app 的权限

```
  // add core start
 private void grantRuntimePermissionsForPackage(int userId, PackageInfo pkg) {
         Set<String> permissions = new ArraySet<>();
         DelayingPackageManagerCache pm = new DelayingPackageManagerCache();
		 String [] pkgPermissions = pkg.requestedPermissions;
		 if(pkgPermissions== null || pkgPermissions.length==0)return;
         for (String permission :  pkgPermissions) {
             final PermissionInfo perm = pm.getPermissionInfo(permission);
             if (perm == null) {
                 continue;
             }
             if (perm.isRuntime()) {
			     Log.i(TAG, "Granting all:permission="+permission);
                 permissions.add(permission);
             }
         }
         if (!permissions.isEmpty()) {
            grantRuntimePermissions(pm,pkg, permissions, true, userId);
         }
         pm.apply();
    }
    public void grantAllRuntimePermissions() {
         Log.i(TAG, "Granting all runtime permissions for user ");
        final long ident = Binder.clearCallingIdentity();
        try {
			Intent intent = new Intent(Intent.ACTION_MAIN);
			intent.addCategory(Intent.CATEGORY_LAUNCHER);
			List<ResolveInfo> resolveInfos = mContext.getPackageManager().queryIntentActivities(intent, PackageManager.MATCH_ALL);
			Log.i(TAG, "resolveInfos.size=" + resolveInfos.size());
			for (ResolveInfo resoveInfo : resolveInfos) {
				String packageName = resoveInfo.activityInfo.packageName;
				final PackageInfo pkg = NO_PM_CACHE.getPackageInfo(packageName);
				Log.i(TAG, "pkg=" + pkg +"--packageName:"+packageName);
				if (pkg == null) {
					continue;
				}
				grantRuntimePermissionsForPackage(UserHandle.USER_SYSTEM, pkg);
			}
		
                final PackageInfo pkg1 = NO_PM_CACHE.getPackageInfo("com.android.launcher3");
				Log.i(TAG, "pkg1=" + pkg1);
				if (pkg1 != null) {
				    grantRuntimePermissionsForPackage(UserHandle.USER_SYSTEM, pkg1);
				}
		} catch (Exception e) {
            e.printStackTrace();
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
 // add core end
```

通过在 DefaultPermissionGrantPolicy.java 的方法中增加 grantRuntimePermissionsForPackage(int userId, PackageInfo pkg)  
来实现根据包名来授予运行时权限，而在 grantAllRuntimePermissions() 中根据 getPackageManager().queryIntentActivities(intent, PackageManager.MATCH_ALL)  
来查询系统中所有 app, 然后调用 grantRuntimePermissionsForPackage(UserHandle.USER_SYSTEM, pkg); 来实现对 app 授予运行时权限  
而对于其他一些特殊 app 可以根据包名授予 app 运行时权限，编译运行后可以在系统设置中  
的权限详情页看到 app 权限都是默认授予了
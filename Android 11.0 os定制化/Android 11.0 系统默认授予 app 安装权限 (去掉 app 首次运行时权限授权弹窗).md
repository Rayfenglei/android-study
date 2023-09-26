> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125648314)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 系统默认授予 app 安装权限去掉 app 首次运行时权限授权弹窗功能分析](#t1)

[3. 系统默认授予 app 安装权限去掉 app 首次运行时权限授权弹窗功能代码分析](#t2)

[4. 系统默认授予 app 安装权限去掉 app 首次运行时权限授权弹窗功能具体实现](#t3)

1. 概述
-----

在 11.0 的产品开发中，由于 6.0 以后 app 需要动态申请权限, 会在 app 首次启动时，弹窗授权，开发需求要求第三方 app 默认授予运行时权限，不让弹授权框，所以要求从 frameworks 层来要求来默认授予 app 权限

2. 系统默认授予 app 安装权限去掉 app 首次运行时权限授权弹窗功能分析
----------------------------------------

android6.0 以后, 由于对系统运行安全的考虑，既然在 AndroidManifest.[xml](https://so.csdn.net/so/search?q=xml&spm=1001.2101.3001.7020) 中，申请了权限为了安全考虑， 对于一些特殊权限和危险权限的申请权限都要 app 申请运行时权限, 而 PermissionManagerService.java 就是负责对系统权限管理的服务 所以要在 PermissionManagerService.java 默认授予安装时权限

3. 系统默认授予 app 安装权限去掉 app 首次运行时权限授权弹窗功能代码分析
------------------------------------------

源码路径：  
/frameworks/base/services/core/java/com/android/server/pm/permission/PermissionManagerService.java

首选看 updatePermissions 更新权限的相关方法

```
 
         @Override
          public void updatePermissions(@NonNull String packageName, @Nullable AndroidPackage pkg) {
              PermissionManagerService.this
                      .updatePermissions(packageName, pkg, mDefaultPermissionCallback);
          }
 
      private void updatePermissions(@NonNull String packageName, @Nullable AndroidPackage pkg,
              @NonNull PermissionCallback callback) {
          // If the package is being deleted, update the permissions of all the apps
          final int flags =
                  (pkg == null ? UPDATE_PERMISSIONS_ALL | UPDATE_PERMISSIONS_REPLACE_PKG
                          : UPDATE_PERMISSIONS_REPLACE_PKG);
          updatePermissions(
                  packageName, pkg, getVolumeUuidForPackage(pkg), flags, callback);
      }
 
private void updatePermissions(final @Nullable String changingPkgName,
final @Nullable AndroidPackage changingPkg,
final @Nullable String replaceVolumeUuid,
@UpdatePermissionFlags int flags,
final @Nullable PermissionCallback callback) {
// TODO: Most of the methods exposing BasePermission internals [source package name,
// etc..] shouldn't be needed. Instead, when we've parsed a permission that doesn't
// have package settings, we should make note of it elsewhere [map between
// source package name and BasePermission] and cycle through that here. Then we
// define a single method on BasePermission that takes a PackageSetting, changing
// package name and a package.
// NOTE: With this approach, we also don't need to tree trees differently than
// normal permissions. Today, we need two separate loops because these BasePermission
// objects are stored separately.
// Make sure there are no dangling permission trees.
boolean permissionTreesSourcePackageChanged = updatePermissionTreeSourcePackage(
changingPkgName, changingPkg);
// Make sure all dynamic permissions have been assigned to a package,
// and make sure there are no dangling permissions.
boolean permissionSourcePackageChanged = updatePermissionSourcePackage(changingPkgName,
changingPkg, callback);
 
if (permissionTreesSourcePackageChanged | permissionSourcePackageChanged) {
// Permission ownership has changed. This e.g. changes which packages can get signature
// permissions
Slog.i(TAG, "Permission ownership changed. Updating all permissions.");
flags |= UPDATE_PERMISSIONS_ALL;
}
 
cacheBackgroundToForegoundPermissionMapping();
 
Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "restorePermissionState");
// Now update the permissions for all packages.
if ((flags & UPDATE_PERMISSIONS_ALL) != 0) {
final boolean replaceAll = ((flags & UPDATE_PERMISSIONS_REPLACE_ALL) != 0);
mPackageManagerInt.forEachPackage((AndroidPackage pkg) -> {
if (pkg == changingPkg) {
return;
}
// Only replace for packages on requested volume
final String volumeUuid = getVolumeUuidForPackage(pkg);
final boolean replace = replaceAll && Objects.equals(replaceVolumeUuid, volumeUuid);
restorePermissionState(pkg, replace, changingPkgName, callback);
});
}
 
if (changingPkg != null) {
// Only replace for packages on requested volume
final String volumeUuid = getVolumeUuidForPackage(changingPkg);
final boolean replace = ((flags & UPDATE_PERMISSIONS_REPLACE_PKG) != 0)
&& Objects.equals(replaceVolumeUuid, volumeUuid);
restorePermissionState(changingPkg, replace, changingPkgName, callback);
}
Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
}
 
从上述代码 可以看到最终调用了可以看到 最终会调用 restorePermissionState（）
我们再来看看restorePermissionState（）
private void restorePermissionState(@NonNull AndroidPackage pkg, boolean replace,
@Nullable String packageOfInterest, @Nullable PermissionCallback callback) {
// IMPORTANT: There are two types of permissions: install and runtime.
// Install time permissions are granted when the app is installed to
// all device users and users added in the future. Runtime permissions
// are granted at runtime explicitly to specific users. Normal and signature
// protected permissions are install time permissions. Dangerous permissions
// are install permissions if the app's target SDK is Lollipop MR1 or older,
// otherwise they are runtime permissions. This function does not manage
// runtime permissions except for the case an app targeting Lollipop MR1
// being upgraded to target a newer SDK, in which case dangerous permissions
// are transformed from install time to runtime ones.
 
final PackageSetting ps = (PackageSetting) mPackageManagerInt.getPackageSetting(
pkg.getPackageName());
if (ps == null) {
return;
}
 
final PermissionsState permissionsState = ps.getPermissionsState();
 
final int[] userIds = getAllUserIds();
 
boolean runtimePermissionsRevoked = false;
int[] updatedUserIds = EMPTY_INT_ARRAY;
 
for (int userId : userIds) {
if (permissionsState.isMissing(userId)) {
Collection<String> requestedPermissions;
int targetSdkVersion;
if (!ps.isSharedUser()) {
requestedPermissions = pkg.getRequestedPermissions();
targetSdkVersion = pkg.getTargetSdkVersion();
} else {
requestedPermissions = new ArraySet<>();
targetSdkVersion = Build.VERSION_CODES.CUR_DEVELOPMENT;
List<AndroidPackage> packages = ps.getSharedUser().getPackages();
int packagesSize = packages.size();
for (int i = 0; i < packagesSize; i++) {
AndroidPackage sharedUserPackage = packages.get(i);
requestedPermissions.addAll(sharedUserPackage.getRequestedPermissions());
targetSdkVersion = Math.min(targetSdkVersion,
sharedUserPackage.getTargetSdkVersion());
}
}
 
for (String permissionName : requestedPermissions) {
BasePermission permission = mSettings.getPermission(permissionName);
if (permission == null) {
continue;
}
if (Objects.equals(permission.getSourcePackageName(), PLATFORM_PACKAGE_NAME)
&& permission.isRuntime() && !permission.isRemoved()) {
if (permission.isHardOrSoftRestricted()
|| permission.isImmutablyRestricted()) {
permissionsState.updatePermissionFlags(permission, userId,
FLAG_PERMISSION_RESTRICTION_UPGRADE_EXEMPT,
FLAG_PERMISSION_RESTRICTION_UPGRADE_EXEMPT);
}
if (targetSdkVersion < Build.VERSION_CODES.M) {
permissionsState.updatePermissionFlags(permission, userId,
PackageManager.FLAG_PERMISSION_REVIEW_REQUIRED
| PackageManager.FLAG_PERMISSION_REVOKED_COMPAT,
PackageManager.FLAG_PERMISSION_REVIEW_REQUIRED
| PackageManager.FLAG_PERMISSION_REVOKED_COMPAT);
permissionsState.grantRuntimePermission(permission, userId);
}
}
}
 
permissionsState.setMissing(false, userId);
updatedUserIds = ArrayUtils.appendInt(updatedUserIds, userId);
}
}
 
PermissionsState origPermissions = permissionsState;
 
boolean changedInstallPermission = false;
 
if (replace) {
ps.setInstallPermissionsFixed(false);
if (!ps.isSharedUser()) {
origPermissions = new PermissionsState(permissionsState);
permissionsState.reset();
} else {
// We need to know only about runtime permission changes since the
// calling code always writes the install permissions state but
// the runtime ones are written only if changed. The only cases of
// changed runtime permissions here are promotion of an install to
// runtime and revocation of a runtime from a shared user.
synchronized (mLock) {
updatedUserIds = revokeUnusedSharedUserPermissionsLocked(
ps.getSharedUser(), userIds);
if (!ArrayUtils.isEmpty(updatedUserIds)) {
runtimePermissionsRevoked = true;
}
}
}
}
 
permissionsState.setGlobalGids(mGlobalGids);
 
synchronized (mLock) {
ArraySet<String> newImplicitPermissions = new ArraySet<>();
 
final int N = pkg.getRequestedPermissions().size();
for (int i = 0; i < N; i++) {
final String permName = pkg.getRequestedPermissions().get(i);
final String friendlyName = pkg.getPackageName() + "(" + pkg.getUid() + ")";
final BasePermission bp = mSettings.getPermissionLocked(permName);
final boolean appSupportsRuntimePermissions =
pkg.getTargetSdkVersion() >= Build.VERSION_CODES.M;
String upgradedActivityRecognitionPermission = null;
if (DEBUG_INSTALL && bp != null) {
Log.i(TAG, "Package " + friendlyName
+ " checking " + permName + ": " + bp);
}
 
if (bp == null || getSourcePackageSetting(bp) == null) {
if (packageOfInterest == null || packageOfInterest.equals(
pkg.getPackageName())) {
if (DEBUG_PERMISSIONS) {
Slog.i(TAG, "Unknown permission " + permName
+ " in package " + friendlyName);
}
}
continue;
}
 
// Cache newImplicitPermissions before modifing permissionsState as for the shared
// uids the original and new state are the same object
if (!origPermissions.hasRequestedPermission(permName)
&& (pkg.getImplicitPermissions().contains(permName)
|| (permName.equals(Manifest.permission.ACTIVITY_RECOGNITION)))) {
if (pkg.getImplicitPermissions().contains(permName)) {
// If permName is an implicit permission, try to auto-grant
newImplicitPermissions.add(permName);
 
if (DEBUG_PERMISSIONS) {
Slog.i(TAG, permName + " is newly added for " + friendlyName);
}
} else {
// Special case for Activity Recognition permission. Even if AR permission
// is not an implicit permission we want to add it to the list (try to
// auto-grant it) if the app was installed on a device before AR permission
// was split, regardless of if the app now requests the new AR permission
// or has updated its target SDK and AR is no longer implicit to it.
// This is a compatibility workaround for apps when AR permission was
// split in Q.
final List<SplitPermissionInfoParcelable> permissionList =
getSplitPermissions();
int numSplitPerms = permissionList.size();
for (int splitPermNum = 0; splitPermNum < numSplitPerms; splitPermNum++) {
SplitPermissionInfoParcelable sp = permissionList.get(splitPermNum);
String splitPermName = sp.getSplitPermission();
if (sp.getNewPermissions().contains(permName)
&& origPermissions.hasInstallPermission(splitPermName)) {
upgradedActivityRecognitionPermission = splitPermName;
newImplicitPermissions.add(permName);
 
if (DEBUG_PERMISSIONS) {
Slog.i(TAG, permName + " is newly added for " + friendlyName);
}
break;
}
}
}
}
 
// TODO(b/140256621): The package instant app method has been removed
//  as part of work in b/135203078, so this has been commented out in the meantime
// Limit ephemeral apps to ephemeral allowed permissions.
//                if (/*pkg.isInstantApp()*/ false && !bp.isInstant()) {
//                    if (DEBUG_PERMISSIONS) {
//                        Log.i(TAG, "Denying non-ephemeral permission " + bp.getName()
//                                + " for package " + friendlyName);
//                    }
//                    continue;
//                }
 
if (bp.isRuntimeOnly() && !appSupportsRuntimePermissions) {
if (DEBUG_PERMISSIONS) {
Log.i(TAG, "Denying runtime-only permission " + bp.getName()
+ " for package " + friendlyName);
}
continue;
}
 
final String perm = bp.getName();
boolean allowedSig = false;
int grant = GRANT_DENIED;
 
// Keep track of app op permissions.
if (bp.isAppOp()) {
mSettings.addAppOpPackage(perm, pkg.getPackageName());
}
 
if (bp.isNormal()) {
// For all apps normal permissions are install time ones.
grant = GRANT_INSTALL;
} else if (bp.isRuntime()) {
if (origPermissions.hasInstallPermission(bp.getName())
|| upgradedActivityRecognitionPermission != null) {
// Before Q we represented some runtime permissions as install permissions,
// in Q we cannot do this anymore. Hence upgrade them all.
grant = GRANT_UPGRADE;
} else {
// For modern apps keep runtime permissions unchanged.
grant = GRANT_RUNTIME;
}
} else if (bp.isSignature()) {
// For all apps signature permissions are install time ones.
allowedSig = grantSignaturePermission(perm, pkg, ps, bp, origPermissions);
if (allowedSig) {
grant = GRANT_INSTALL;
}
}
 
if (DEBUG_PERMISSIONS) {
Slog.i(TAG, "Considering granting permission " + perm + " to package "
+ friendlyName);
}
 
if (grant != GRANT_DENIED) {
if (!ps.isSystem() && ps.areInstallPermissionsFixed() && !bp.isRuntime()) {
// If this is an existing, non-system package, then
// we can't add any new permissions to it. Runtime
// permissions can be added any time - they ad dynamic.
if (!allowedSig && !origPermissions.hasInstallPermission(perm)) {
// Except...  if this is a permission that was added
// to the platform (note: need to only do this when
// updating the platform).
if (!isNewPlatformPermissionForPackage(perm, pkg)) {
grant = GRANT_DENIED;
}
}
}
 
switch (grant) {
case GRANT_INSTALL: {
// Revoke this as runtime permission to handle the case of
// a runtime permission being downgraded to an install one.
// Also in permission review mode we keep dangerous permissions
// for legacy apps
for (int userId : userIds) {
if (origPermissions.getRuntimePermissionState(
perm, userId) != null) {
// Revoke the runtime permission and clear the flags.
origPermissions.revokeRuntimePermission(bp, userId);
origPermissions.updatePermissionFlags(bp, userId,
PackageManager.MASK_PERMISSION_FLAGS_ALL, 0);
// If we revoked a permission permission, we have to write.
updatedUserIds = ArrayUtils.appendInt(
updatedUserIds, userId);
}
}
// Grant an install permission.
if (permissionsState.grantInstallPermission(bp) !=
PERMISSION_OPERATION_FAILURE) {
changedInstallPermission = true;
}
} break;
 
case GRANT_RUNTIME: {
boolean hardRestricted = bp.isHardRestricted();
boolean softRestricted = bp.isSoftRestricted();
 
for (int userId : userIds) {
// If permission policy is not ready we don't deal with restricted
// permissions as the policy may whitelist some permissions. Once
// the policy is initialized we would re-evaluate permissions.
final boolean permissionPolicyInitialized =
mPermissionPolicyInternal != null
&& mPermissionPolicyInternal.isInitialized(userId);
 
PermissionState permState = origPermissions
getRuntimePermissionState(perm, userId);
int flags = permState != null ? permState.getFlags() : 0;
 
boolean wasChanged = false;
 
boolean restrictionExempt =
(origPermissions.getPermissionFlags(bp.name, userId)
& FLAGS_PERMISSION_RESTRICTION_ANY_EXEMPT) != 0;
boolean restrictionApplied = (origPermissions.getPermissionFlags(
bp.name, userId) & FLAG_PERMISSION_APPLY_RESTRICTION) != 0;
 
if (appSupportsRuntimePermissions) {
// If hard restricted we don't allow holding it
if (permissionPolicyInitialized && hardRestricted) {
if (!restrictionExempt) {
if (permState != null && permState.isGranted()
&& permissionsState.revokeRuntimePermission(
bp, userId) != PERMISSION_OPERATION_FAILURE) {
wasChanged = true;
}
if (!restrictionApplied) {
flags |= FLAG_PERMISSION_APPLY_RESTRICTION;
wasChanged = true;
}
}
// If soft restricted we allow holding in a restricted form
} else if (permissionPolicyInitialized && softRestricted) {
// Regardless if granted set the restriction flag as it
// may affect app treatment based on this permission.
if (!restrictionExempt && !restrictionApplied) {
flags |= FLAG_PERMISSION_APPLY_RESTRICTION;
wasChanged = true;
}
}
 
// Remove review flag as it is not necessary anymore
if ((flags & FLAG_PERMISSION_REVIEW_REQUIRED) != 0) {
flags &= ~FLAG_PERMISSION_REVIEW_REQUIRED;
wasChanged = true;
}
 
if ((flags & FLAG_PERMISSION_REVOKED_COMPAT) != 0) {
flags &= ~FLAG_PERMISSION_REVOKED_COMPAT;
wasChanged = true;
// Hard restricted permissions cannot be held.
} else if (!permissionPolicyInitialized
|| (!hardRestricted || restrictionExempt)) {
if (permState != null && permState.isGranted()) {
if (permissionsState.grantRuntimePermission(bp, userId)
== PERMISSION_OPERATION_FAILURE) {
wasChanged = true;
}
}
}
} else {
if (permState == null) {
// New permission
if (PLATFORM_PACKAGE_NAME.equals(
bp.getSourcePackageName())) {
if (!bp.isRemoved()) {
flags |= FLAG_PERMISSION_REVIEW_REQUIRED
| FLAG_PERMISSION_REVOKED_COMPAT;
wasChanged = true;
}
}
}
 
if (!permissionsState.hasRuntimePermission(bp.name, userId)
&& permissionsState.grantRuntimePermission(bp, userId)
!= PERMISSION_OPERATION_FAILURE) {
wasChanged = true;
}
 
// If legacy app always grant the permission but if restricted
// and not exempt take a note a restriction should be applied.
if (permissionPolicyInitialized
&& (hardRestricted || softRestricted)
&& !restrictionExempt && !restrictionApplied) {
flags |= FLAG_PERMISSION_APPLY_RESTRICTION;
wasChanged = true;
}
}
 
// If unrestricted or restriction exempt, don't apply restriction.
if (permissionPolicyInitialized) {
if (!(hardRestricted || softRestricted) || restrictionExempt) {
if (restrictionApplied) {
flags &= ~FLAG_PERMISSION_APPLY_RESTRICTION;
// Dropping restriction on a legacy app implies a review
if (!appSupportsRuntimePermissions) {
flags |= FLAG_PERMISSION_REVIEW_REQUIRED;
}
wasChanged = true;
}
}
}
 
if (wasChanged) {
updatedUserIds = ArrayUtils.appendInt(updatedUserIds, userId);
}
 
permissionsState.updatePermissionFlags(bp, userId,
MASK_PERMISSION_FLAGS_ALL, flags);
}
} break;
 
case GRANT_UPGRADE: {
// Upgrade from Pre-Q to Q permission model. Make all permissions
// runtime
PermissionState permState = origPermissions
getInstallPermissionState(perm);
int flags = (permState != null) ? permState.getFlags() : 0;
 
BasePermission bpToRevoke =
upgradedActivityRecognitionPermission == null
? bp : mSettings.getPermissionLocked(
upgradedActivityRecognitionPermission);
// Remove install permission
if (origPermissions.revokeInstallPermission(bpToRevoke)
!= PERMISSION_OPERATION_FAILURE) {
origPermissions.updatePermissionFlags(bpToRevoke,
UserHandle.USER_ALL,
(MASK_PERMISSION_FLAGS_ALL
& ~FLAG_PERMISSION_APPLY_RESTRICTION), 0);
changedInstallPermission = true;
}
 
boolean hardRestricted = bp.isHardRestricted();
boolean softRestricted = bp.isSoftRestricted();
 
for (int userId : userIds) {
// If permission policy is not ready we don't deal with restricted
// permissions as the policy may whitelist some permissions. Once
// the policy is initialized we would re-evaluate permissions.
final boolean permissionPolicyInitialized =
mPermissionPolicyInternal != null
&& mPermissionPolicyInternal.isInitialized(userId);
 
boolean wasChanged = false;
 
boolean restrictionExempt =
(origPermissions.getPermissionFlags(bp.name, userId)
& FLAGS_PERMISSION_RESTRICTION_ANY_EXEMPT) != 0;
boolean restrictionApplied = (origPermissions.getPermissionFlags(
bp.name, userId) & FLAG_PERMISSION_APPLY_RESTRICTION) != 0;
 
if (appSupportsRuntimePermissions) {
// If hard restricted we don't allow holding it
if (permissionPolicyInitialized && hardRestricted) {
if (!restrictionExempt) {
if (permState != null && permState.isGranted()
&& permissionsState.revokeRuntimePermission(
bp, userId) != PERMISSION_OPERATION_FAILURE) {
wasChanged = true;
}
if (!restrictionApplied) {
flags |= FLAG_PERMISSION_APPLY_RESTRICTION;
wasChanged = true;
}
}
// If soft restricted we allow holding in a restricted form
} else if (permissionPolicyInitialized && softRestricted) {
// Regardless if granted set the  restriction flag as it
// may affect app treatment based on this permission.
if (!restrictionExempt && !restrictionApplied) {
flags |= FLAG_PERMISSION_APPLY_RESTRICTION;
wasChanged = true;
}
}
 
// Remove review flag as it is not necessary anymore
if ((flags & FLAG_PERMISSION_REVIEW_REQUIRED) != 0) {
flags &= ~FLAG_PERMISSION_REVIEW_REQUIRED;
wasChanged = true;
}
 
if ((flags & FLAG_PERMISSION_REVOKED_COMPAT) != 0) {
flags &= ~FLAG_PERMISSION_REVOKED_COMPAT;
wasChanged = true;
// Hard restricted permissions cannot be held.
} else if (!permissionPolicyInitialized ||
(!hardRestricted || restrictionExempt)) {
if (permissionsState.grantRuntimePermission(bp, userId) !=
PERMISSION_OPERATION_FAILURE) {
wasChanged = true;
}
}
} else {
if (!permissionsState.hasRuntimePermission(bp.name, userId)
&& permissionsState.grantRuntimePermission(bp,
userId) != PERMISSION_OPERATION_FAILURE) {
flags |= FLAG_PERMISSION_REVIEW_REQUIRED;
wasChanged = true;
}
 
// If legacy app always grant the permission but if restricted
// and not exempt take a note a restriction should be applied.
if (permissionPolicyInitialized
&& (hardRestricted || softRestricted)
&& !restrictionExempt && !restrictionApplied) {
flags |= FLAG_PERMISSION_APPLY_RESTRICTION;
wasChanged = true;
}
}
 
// If unrestricted or restriction exempt, don't apply restriction.
if (permissionPolicyInitialized) {
if (!(hardRestricted || softRestricted) || restrictionExempt) {
if (restrictionApplied) {
flags &= ~FLAG_PERMISSION_APPLY_RESTRICTION;
// Dropping restriction on a legacy app implies a review
if (!appSupportsRuntimePermissions) {
flags |= FLAG_PERMISSION_REVIEW_REQUIRED;
}
wasChanged = true;
}
}
}
 
if (wasChanged) {
updatedUserIds = ArrayUtils.appendInt(updatedUserIds, userId);
}
 
permissionsState.updatePermissionFlags(bp, userId,
MASK_PERMISSION_FLAGS_ALL, flags);
}
} break;
 
default: {
if (packageOfInterest == null
|| packageOfInterest.equals(pkg.getPackageName())) {
if (DEBUG_PERMISSIONS) {
Slog.i(TAG, "Not granting permission " + perm
+ " to package " + friendlyName
+ " because it was previously installed without");
}
}
} break;
}
} else {
if (permissionsState.revokeInstallPermission(bp) !=
PERMISSION_OPERATION_FAILURE) {
// Also drop the permission flags.
permissionsState.updatePermissionFlags(bp, UserHandle.USER_ALL,
MASK_PERMISSION_FLAGS_ALL, 0);
changedInstallPermission = true;
if (DEBUG_PERMISSIONS) {
Slog.i(TAG, "Un-granting permission " + perm
+ " from package " + friendlyName
+ " (protectionLevel=" + bp.getProtectionLevel()
+ " flags=0x"
+ Integer.toHexString(PackageInfoUtils.appInfoFlags(pkg, ps))
+ ")");
}
} else if (bp.isAppOp()) {
// Don't print warning for app op permissions, since it is fine for them
// not to be granted, there is a UI for the user to decide.
if (DEBUG_PERMISSIONS
&& (packageOfInterest == null
|| packageOfInterest.equals(pkg.getPackageName()))) {
Slog.i(TAG, "Not granting permission " + perm
+ " to package " + friendlyName
+ " (protectionLevel=" + bp.getProtectionLevel()
+ " flags=0x"
+ Integer.toHexString(PackageInfoUtils.appInfoFlags(pkg, ps))
+ ")");
}
}
}
}
 
if ((changedInstallPermission || replace) && !ps.areInstallPermissionsFixed() &&
!ps.isSystem() || ps.getPkgState().isUpdatedSystemApp()) {
// This is the first that we have heard about this package, so the
// permissions we have now selected are fixed until explicitly
// changed.
ps.setInstallPermissionsFixed(true);
}
 
updatedUserIds = revokePermissionsNoLongerImplicitLocked(permissionsState, pkg,
updatedUserIds);
updatedUserIds = setInitialGrantForNewImplicitPermissionsLocked(origPermissions,
permissionsState, pkg, newImplicitPermissions, updatedUserIds);
updatedUserIds = checkIfLegacyStorageOpsNeedToBeUpdated(pkg, replace, updatedUserIds);
}
 
// Persist the runtime permissions state for users with changes. If permissions
// were revoked because no app in the shared user declares them we have to
// write synchronously to avoid losing runtime permissions state.
if (callback != null) {
callback.onPermissionUpdated(updatedUserIds, runtimePermissionsRevoked);
}
 
for (int userId : updatedUserIds) {
notifyRuntimePermissionStateChanged(pkg.getPackageName(), userId);
}
}
 
从下面代码中可以看到
                final String perm = bp.getName();
                boolean allowedSig = false;
                int grant = GRANT_DENIED;
 
                // Keep track of app op permissions.
                if (bp.isAppOp()) {
                    mSettings.addAppOpPackage(perm, pkg.packageName);
                }
 
                if (bp.isNormal()) {
                    // For all apps normal permissions are install time ones.
                    grant = GRANT_INSTALL;
                } else if (bp.isRuntime()) {
                    if (origPermissions.hasInstallPermission(bp.getName())
                            || upgradedActivityRecognitionPermission != null) {
                        // Before Q we represented some runtime permissions as install permissions,
                        // in Q we cannot do this anymore. Hence upgrade them all.
                        grant = GRANT_UPGRADE;
                    } else {
                         // For modern apps keep runtime permissions unchanged.
                         grant = GRANT_RUNTIME;
                     }
                 }
 
```

根据 bp 判断 app 是否是运行时状态  
bp.isRuntime() 为 true 就是运行时授权 bp.isNormal() 为 true 系统启动安装时授予权限  
所以在 bp.isRuntime() 和 bp.isNormal() 一样 给与安装权限就行了
----------------------------------------------------------------------------------------------------------------------------------------

4. 系统默认授予 app 安装权限去掉 app 首次运行时权限授权弹窗功能具体实现
------------------------------------------

```
diff --git a/frameworks/base/services/core/java/com/android/server/pm/permission/PermissionManagerService.java b/frameworks/base/services/core/java/com/android/server/pm/permission/PermissionManagerService.java
old mode 100644
new mode 100755
index bb66c2de00..f7a40171af
--- a/frameworks/base/services/core/java/com/android/server/pm/permission/PermissionManagerService.java
+++ b/frameworks/base/services/core/java/com/android/server/pm/permission/PermissionManagerService.java
@@ -1027,6 +1027,8 @@ public class PermissionManagerService {
  private void restorePermissionState(@NonNull PackageParser.Package pkg, boolean replace,
            @Nullable String packageOfInterest, @Nullable PermissionCallback callback) {
        // IMPORTANT: There are two types of permissions: install and runtime.
        // Install time permissions are granted when the app is installed to
        // all device users and users added in the future. Runtime permissions
        // are granted at runtime explicitly to specific users. Normal and signature
        // protected permissions are install time permissions. Dangerous permissions
        // are install permissions if the app's target SDK is Lollipop MR1 or older,
        // otherwise they are runtime permissions. This function does not manage
        // runtime permissions except for the case an app targeting Lollipop MR1
        // being upgraded to target a newer SDK, in which case dangerous permissions
        // are transformed from install time to runtime ones.
 
        final PackageSetting ps = (PackageSetting) pkg.mExtras;
        if (ps == null) {
            return;
        }
 
        final PermissionsState permissionsState = ps.getPermissionsState();
        PermissionsState origPermissions = permissionsState;
 
        final int[] currentUserIds = UserManagerService.getInstance().getUserIds();
 
        boolean runtimePermissionsRevoked = false;
        int[] updatedUserIds = EMPTY_INT_ARRAY;
 
        boolean changedInstallPermission = false;
 
        if (replace) {
            ps.setInstallPermissionsFixed(false);
            if (!ps.isSharedUser()) {
                origPermissions = new PermissionsState(permissionsState);
                permissionsState.reset();
            } else {
                // We need to know only about runtime permission changes since the
                // calling code always writes the install permissions state but
                // the runtime ones are written only if changed. The only cases of
                // changed runtime permissions here are promotion of an install to
                // runtime and revocation of a runtime from a shared user.
                synchronized (mLock) {
                    updatedUserIds = revokeUnusedSharedUserPermissionsLocked(
                            ps.getSharedUser(), UserManagerService.getInstance().getUserIds());
                    if (!ArrayUtils.isEmpty(updatedUserIds)) {
                        runtimePermissionsRevoked = true;
                    }
                }
            }
        }
 
        permissionsState.setGlobalGids(mGlobalGids);
 
        synchronized (mLock) {
            ArraySet<String> newImplicitPermissions = new ArraySet<>();
 
            final int N = pkg.requestedPermissions.size();
            for (int i = 0; i < N; i++) {
                final String permName = pkg.requestedPermissions.get(i);
                final BasePermission bp = mSettings.getPermissionLocked(permName);
                final boolean appSupportsRuntimePermissions =
                        pkg.applicationInfo.targetSdkVersion >= Build.VERSION_CODES.M;
                String upgradedActivityRecognitionPermission = null;
 
                if (DEBUG_INSTALL) {
                    Log.i(TAG, "Package " + pkg.packageName + " checking " + permName + ": " + bp);
                }
 
                if (bp == null || bp.getSourcePackageSetting() == null) {
                    if (packageOfInterest == null || packageOfInterest.equals(pkg.packageName)) {
                        if (DEBUG_PERMISSIONS) {
                            Slog.i(TAG, "Unknown permission " + permName
                                    + " in package " + pkg.packageName);
                        }
                    }
                    continue;
                }
 
                // Cache newImplicitPermissions before modifing permissionsState as for the shared
                // uids the original and new state are the same object
                if (!origPermissions.hasRequestedPermission(permName)
                        && (pkg.implicitPermissions.contains(permName)
                                || (permName.equals(Manifest.permission.ACTIVITY_RECOGNITION)))) {
                    if (pkg.implicitPermissions.contains(permName)) {
                        // If permName is an implicit permission, try to auto-grant
                        newImplicitPermissions.add(permName);
 
                        if (DEBUG_PERMISSIONS) {
                            Slog.i(TAG, permName + " is newly added for " + pkg.packageName);
                        }
                    } else {
                        // Special case for Activity Recognition permission. Even if AR permission
                        // is not an implicit permission we want to add it to the list (try to
                        // auto-grant it) if the app was installed on a device before AR permission
                        // was split, regardless of if the app now requests the new AR permission
                        // or has updated its target SDK and AR is no longer implicit to it.
                        // This is a compatibility workaround for apps when AR permission was
                        // split in Q.
                        final List<PermissionManager.SplitPermissionInfo> permissionList =
                                getSplitPermissions();
                        int numSplitPerms = permissionList.size();
                        for (int splitPermNum = 0; splitPermNum < numSplitPerms; splitPermNum++) {
                            PermissionManager.SplitPermissionInfo sp =
                                    permissionList.get(splitPermNum);
                            String splitPermName = sp.getSplitPermission();
                            if (sp.getNewPermissions().contains(permName)
                                    && origPermissions.hasInstallPermission(splitPermName)) {
                                upgradedActivityRecognitionPermission = splitPermName;
                                newImplicitPermissions.add(permName);
 
                                if (DEBUG_PERMISSIONS) {
                                    Slog.i(TAG, permName + " is newly added for "
                                            + pkg.packageName);
                                }
                                break;
                            }
                        }
                    }
                }
 
                // Limit ephemeral apps to ephemeral allowed permissions.
                if (pkg.applicationInfo.isInstantApp() && !bp.isInstant()) {
                    if (DEBUG_PERMISSIONS) {
                        Log.i(TAG, "Denying non-ephemeral permission " + bp.getName()
                                + " for package " + pkg.packageName);
                    }
                    continue;
                }
 
                if (bp.isRuntimeOnly() && !appSupportsRuntimePermissions) {
                    if (DEBUG_PERMISSIONS) {
                        Log.i(TAG, "Denying runtime-only permission " + bp.getName()
                                + " for package " + pkg.packageName);
                    }
                    continue;
                }
 
                final String perm = bp.getName();
                boolean allowedSig = false;
                int grant = GRANT_DENIED;
 
                // Keep track of app op permissions.
                if (bp.isAppOp()) {
                    mSettings.addAppOpPackage(perm, pkg.packageName);
                }
 
                if (bp.isNormal()) {
                    // For all apps normal permissions are install time ones.
                    grant = GRANT_INSTALL;
                } else if (bp.isRuntime()) {
                    if (origPermissions.hasInstallPermission(bp.getName())
                            || upgradedActivityRecognitionPermission != null) {
                        // Before Q we represented some runtime permissions as install permissions,
                        // in Q we cannot do this anymore. Hence upgrade them all.
                        grant = GRANT_UPGRADE;
                    } else {
                         // For modern apps keep runtime permissions unchanged.
                         grant = GRANT_RUNTIME;
                     }
如果是运行时 给予安装授予权限就行
+                    // open all permissions
+                    grant = GRANT_INSTALL;
                 } else if (bp.isSignature()) {
                     // For all apps signature permissions are install time ones.
                     allowedSig = grantSignaturePermission(perm, pkg, bp, origPermissions
```
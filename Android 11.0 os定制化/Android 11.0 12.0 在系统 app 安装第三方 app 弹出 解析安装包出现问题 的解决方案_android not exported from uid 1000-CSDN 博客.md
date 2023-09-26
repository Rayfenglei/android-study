> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124643333)

### 1. 概述

在 11.0 12.0 的系统定制化开发中，对于系统内置 app 中用代码调用系统安装接口安装 app 时抛出 Permission Denial: that is not exported from UID 1000 的异常，查询资料这个异常发现通常是由于 Uri 权限导致的问题，这就需要看 PMS 在安装的时候，需要什么权限，避免掉就可以了  
例如:

```
File apk = new File(...);
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
Uri uri = FileProvider.getUriForFile(this, "com.example.demo.fileprovider", apk);
intent.setDataAndType(uri, "application/vnd.android.package-archive");
startActivity(intent);

```

在上面的代码 通过在参数中增加 URI 在安装的过程中需要权限  
在执行这样的代码会抛出异常是由于在安装时要申请 uri 权限

### 2. 在系统 app 安装第三方 app 弹出 解析安装包出现问题 的解决方案核心类

```
frameworks\base\services\core\java\com\android\server\uri\UriGrantsManagerService.java

```

### 3. 在系统 app 安装第三方 app 弹出 解析安装包出现问题 的解决方案核心功能实现

在 11.0 中的 FiveProvider 权限，负责在进行 Uri 调用的时候，申请权限，而在 11.0 的系统中  
在 UriGrantsManagerService.java 里面负责管理 FileProvider 权限的处理，所以需要  
分析 UriGrantsManagerService.java 的相关安装权限的功能  
路径为: frameworks\base\services\core\java\com\android\server\uri\UriGrantsManagerService.java

```
 @Override
  public int checkGrantUriPermission(int callingUid, String targetPkg, Uri uri, int modeFlags,
          int userId) {
      enforceNotIsolatedCaller("checkGrantUriPermission");
      return UriGrantsManagerService.this.checkGrantUriPermissionUnlocked(
              callingUid, targetPkg, uri, modeFlags, userId);
  }
  
  int checkGrantUriPermissionUnlocked(int callingUid, String targetPkg, GrantUri grantUri,
        final int modeFlags, int lastTargetUid) {
    if (!Intent.isAccessUriMode(modeFlags)) {
        return -1;
    }

    if (targetPkg != null) {
        if (DEBUG) Slog.v(TAG, "Checking grant " + targetPkg + " permission to " + grantUri);
    }

    final IPackageManager pm = AppGlobals.getPackageManager();

    // If this is not a content: uri, we can't do anything with it.
    if (!ContentResolver.SCHEME_CONTENT.equals(grantUri.uri.getScheme())) {
        if (DEBUG) Slog.v(TAG, "Can't grant URI permission for non-content URI: " + grantUri);
        return -1;
    }

    // Bail early if system is trying to hand out permissions directly; it
    // must always grant permissions on behalf of someone explicit.
    final int callingAppId = UserHandle.getAppId(callingUid);
    if ((callingAppId == SYSTEM_UID) || (callingAppId == ROOT_UID)) {
        if ("com.android.settings.files".equals(grantUri.uri.getAuthority())
                || "com.android.settings.module_licenses".equals(grantUri.uri.getAuthority())) {
            // Exempted authority for
            // 1. cropping user photos and sharing a generated license html
            //    file in Settings app
            // 2. sharing a generated license html file in TvSettings app
            // 3. Sharing module license files from Settings app
        } else {
            Slog.w(TAG, "For security reasons, the system cannot issue a Uri permission"
                    + " grant to " + grantUri + "; use startActivityAsCaller() instead");
            return -1;
        }
    }

    final String authority = grantUri.uri.getAuthority();
    final ProviderInfo pi = getProviderInfo(authority, grantUri.sourceUserId,
            MATCH_DEBUG_TRIAGED_MISSING);
    if (pi == null) {
        Slog.w(TAG, "No content provider found for permission check: " +
                grantUri.uri.toSafeString());
        return -1;
    }

    int targetUid = lastTargetUid;
    if (targetUid < 0 && targetPkg != null) {
        try {
            targetUid = pm.getPackageUid(targetPkg, MATCH_DEBUG_TRIAGED_MISSING,
                    UserHandle.getUserId(callingUid));
            if (targetUid < 0) {
                if (DEBUG) Slog.v(TAG, "Can't grant URI permission no uid for: " + targetPkg);
                return -1;
            }
        } catch (RemoteException ex) {
            return -1;
        }
    }

    // Figure out the value returned when access is allowed
    final int allowedResult;
    if ((modeFlags & Intent.FLAG_GRANT_PERSISTABLE_URI_PERMISSION) != 0
            || pi.forceUriPermissions) {
        // If we're extending a persistable grant or need to force, then we need to return
        // "targetUid" so that we always create a grant data structure to
        // support take/release APIs
        allowedResult = targetUid;
    } else {
        // Otherwise, we can return "-1" to indicate that no grant data
        // structures need to be created
        allowedResult = -1;
    }

    if (targetUid >= 0) {
        // First...  does the target actually need this permission?
        if (checkHoldingPermissions(pm, pi, grantUri, targetUid, modeFlags)) {
            // No need to grant the target this permission.
            if (DEBUG) Slog.v(TAG,
                    "Target " + targetPkg + " already has full permission to " + grantUri);
            return allowedResult;
        }
    } else {
        // First...  there is no target package, so can anyone access it?
        boolean allowed = pi.exported;
        if ((modeFlags&Intent.FLAG_GRANT_READ_URI_PERMISSION) != 0) {
            if (pi.readPermission != null) {
                allowed = false;
            }
        }
        if ((modeFlags&Intent.FLAG_GRANT_WRITE_URI_PERMISSION) != 0) {
            if (pi.writePermission != null) {
                allowed = false;
            }
        }
        if (pi.pathPermissions != null) {
            final int N = pi.pathPermissions.length;
            for (int i=0; i<N; i++) {
                if (pi.pathPermissions[i] != null
                        && pi.pathPermissions[i].match(grantUri.uri.getPath())) {
                    if ((modeFlags&Intent.FLAG_GRANT_READ_URI_PERMISSION) != 0) {
                        if (pi.pathPermissions[i].getReadPermission() != null) {
                            allowed = false;
                        }
                    }
                    if ((modeFlags&Intent.FLAG_GRANT_WRITE_URI_PERMISSION) != 0) {
                        if (pi.pathPermissions[i].getWritePermission() != null) {
                            allowed = false;
                        }
                    }
                    break;
                }
            }
        }
        if (allowed) {
            return allowedResult;
        }
    }

    /* There is a special cross user grant if:
     * - The target is on another user.
     * - Apps on the current user can access the uri without any uid permissions.
     * In this case, we grant a uri permission, even if the ContentProvider does not normally
     * grant uri permissions.
     */
    boolean specialCrossUserGrant = targetUid >= 0
            && UserHandle.getUserId(targetUid) != grantUri.sourceUserId
            && checkHoldingPermissionsInternal(pm, pi, grantUri, callingUid,
            modeFlags, false /*without considering the uid permissions*/);

    // Second...  is the provider allowing granting of URI permissions?
    if (!specialCrossUserGrant) {
        if (!pi.grantUriPermissions) {
            throw new SecurityException("Provider " + pi.packageName
                    + "/" + pi.name
                    + " does not allow granting of Uri permissions (uri "
                    + grantUri + ")");
        }
        if (pi.uriPermissionPatterns != null) {
            final int N = pi.uriPermissionPatterns.length;
            boolean allowed = false;
            for (int i=0; i<N; i++) {
                if (pi.uriPermissionPatterns[i] != null
                        && pi.uriPermissionPatterns[i].match(grantUri.uri.getPath())) {
                    allowed = true;
                    break;
                }
            }
            if (!allowed) {
                throw new SecurityException("Provider " + pi.packageName
                        + "/" + pi.name
                        + " does not allow granting of permission to path of Uri "
                        + grantUri);
            }
        }
    }

    // Third...  does the caller itself have permission to access this uri?
    if (!checkHoldingPermissions(pm, pi, grantUri, callingUid, modeFlags)) {
        // Require they hold a strong enough Uri permission
        if (!checkUriPermission(grantUri, callingUid, modeFlags)) {
            if (android.Manifest.permission.MANAGE_DOCUMENTS.equals(pi.readPermission)) {
                throw new SecurityException(
                        "UID " + callingUid + " does not have permission to " + grantUri
                                + "; you could obtain access using ACTION_OPEN_DOCUMENT "
                                + "or related APIs");
            } else {
                throw new SecurityException(
                        "UID " + callingUid + " does not have permission to " + grantUri);
            }
        }
    }
    return targetUid;
}

```

在上述的方法中可以看到 checkHoldingPermissions 就是判断这个 app 是否有 FiveProvider 安装权限  
需要在这里添加包名，然后授予安装权限

修改如下：com.example.demo.fileprovider 替换成 自己 app 的 fiveprovider 就可以了

```
final int callingAppId = UserHandle.getAppId(callingUid);
if ((callingAppId == SYSTEM_UID) || (callingAppId == ROOT_UID)) {
if ("com.android.settings.files".equals(grantUri.uri.getAuthority())
|| "com.example.demo.fileprovider".equals(grantUri.uri.getAuthority())
|| "com.android.settings.module_licenses".equals(grantUri.uri.getAuthority())) {
// Exempted authority for
// 1. cropping user photos and sharing a generated license html
//    file in Settings app
// 2. sharing a generated license html file in TvSettings app
// 3. Sharing module license files from Settings app
} else {
Slog.w(TAG, "For security reasons, the system cannot issue a Uri permission"
+ " grant to " + grantUri + "; use startActivityAsCaller() instead");
return -1;
}
}

```
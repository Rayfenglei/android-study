> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124870502)

### 1. 概述

在 11.0 定制化开发中, 最近由项目需求要实现对某些 app 应用安装限制也就是 app 安装黑名单功能，在黑名单之中的应用会被限制安装，不能安装到系统中  
功能分析  
在系统中 [PMS](https://so.csdn.net/so/search?q=PMS&spm=1001.2101.3001.7020) 就是负责管理 app 安装和卸载的，在安装的时候判断是不是在安装黑名单中，然后决定是否安装这个 app

### 2.app 安装黑名单（限制 app 安装）核心代码

```
frameworks/base/core/java/android/content/pm/IPackageManager.aidl
frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

```

### 3.app 安装黑名单（限制 app 安装）功能分析和实现

### 3.1PackManagerService.java 安装 app 相关的源码分析

所以接下来看下 PackManagerService.java 的源码

```
@GuardedBy("mInstallLock")
    private PrepareResult preparePackageLI(InstallArgs args, PackageInstalledInfo res)
            throws PrepareFailure {
        final int installFlags = args.installFlags;
        final String installerPackageName = args.installerPackageName;
        final String volumeUuid = args.volumeUuid;
        final File tmpPackageFile = new File(args.getCodePath());
        final boolean onExternal = args.volumeUuid != null;
        final boolean instantApp = ((installFlags & PackageManager.INSTALL_INSTANT_APP) != 0);
        final boolean fullApp = ((installFlags & PackageManager.INSTALL_FULL_APP) != 0);
        final boolean virtualPreload =
                ((installFlags & PackageManager.INSTALL_VIRTUAL_PRELOAD) != 0);
        @ScanFlags int scanFlags = SCAN_NEW_INSTALL | SCAN_UPDATE_SIGNATURE;
        if (args.move != null) {
            // moving a complete application; perform an initial scan on the new install location
            scanFlags |= SCAN_INITIAL;
        }
        if ((installFlags & PackageManager.INSTALL_DONT_KILL_APP) != 0) {
            scanFlags |= SCAN_DONT_KILL_APP;
        }
        if (instantApp) {
            scanFlags |= SCAN_AS_INSTANT_APP;
        }
        if (fullApp) {
            scanFlags |= SCAN_AS_FULL_APP;
        }
        if (virtualPreload) {
            scanFlags |= SCAN_AS_VIRTUAL_PRELOAD;
        }

        if (DEBUG_INSTALL) Slog.d(TAG, "installPackageLI: path=" + tmpPackageFile);

        // Sanity check
        if (instantApp && onExternal) {
            Slog.i(TAG, "Incompatible ephemeral install; external=" + onExternal);
            throw new PrepareFailure(PackageManager.INSTALL_FAILED_INSTANT_APP_INVALID);
        }

        // Retrieve PackageSettings and parse package
        @ParseFlags final int parseFlags = mDefParseFlags | PackageParser.PARSE_CHATTY
                | PackageParser.PARSE_ENFORCE_CODE
                | (onExternal ? PackageParser.PARSE_EXTERNAL_STORAGE : 0);

        PackageParser pp = new PackageParser();
        pp.setSeparateProcesses(mSeparateProcesses);
        pp.setDisplayMetrics(mMetrics);
        pp.setCallback(mPackageParserCallback);

        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "parsePackage");
        final PackageParser.Package pkg;
        try {
            pkg = pp.parsePackage(tmpPackageFile, parseFlags);
            DexMetadataHelper.validatePackageDexMetadata(pkg);
        } catch (PackageParserException e) {
            throw new PrepareFailure("Failed parse during installPackageLI", e);
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }

        // Instant apps have several additional install-time checks.
        if (instantApp) {
            if (pkg.applicationInfo.targetSdkVersion < Build.VERSION_CODES.O) {
                Slog.w(TAG,
                        "Instant app package " + pkg.packageName + " does not target at least O");
                throw new PrepareFailure(INSTALL_FAILED_INSTANT_APP_INVALID,
                        "Instant app package must target at least O");
            }
            if (pkg.mSharedUserId != null) {
                Slog.w(TAG, "Instant app package " + pkg.packageName
                        + " may not declare sharedUserId.");
                throw new PrepareFailure(INSTALL_FAILED_INSTANT_APP_INVALID,
                        "Instant app package may not declare a sharedUserId");
            }
        }
        ...代码省略

```

在 pms 中 preparePackageLI 负责对 app 的安装工作, 无论是 pm 安装或者是 代码安装 都会走 preparePackageLI 所以在这里添加判断包名是否在黑名单就能控制是否安装 app, 具体实现功能如下

### 3.2 在 IPackageManager.aidl 中 添加安装黑名单接口供 app 调用

```
diff --git a/frameworks/base/core/java/android/content/pm/IPackageManager.aidl b/frameworks/base/core/java/android/content/pm/IPackageManager.aidl

old mode 100644

new mode 100755

index a369cc89a3..90fafe5a8f

--- a/frameworks/base/core/java/android/content/pm/IPackageManager.aidl

+++ b/frameworks/base/core/java/android/content/pm/IPackageManager.aidl

@@ -798,4 +798,7 @@ interface IPackageManager {

     */

     int restoreAppData(String sourceDir, String pkgName);

    /* @} */

+   

+       void setInstallPackageBlackList(in List<String> packageNames);

+       List<String> getInstallPackageBlackList();

 }

```

通过添加 setInstallPackageBlackList 和 getInstallPackageBlackList() 作为安装黑名单的接口

### 3.3 在 PMS 中实现接口并实现安装黑名单功能

pms 中实现 setInstallPackageBlackList 和 getInstallPackageBlackList() 这两个安装黑名单的接口

```
diff --git a/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java b/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

index 45289f2e39..6727b10e35 100755

--- a/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

+++ b/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

@@ -111,7 +111,13 @@ import static com.android.server.pm.PackageManagerServiceUtils.getCompressedFile

 import static com.android.server.pm.PackageManagerServiceUtils.getLastModifiedTime;

 import static com.android.server.pm.PackageManagerServiceUtils.logCriticalInfo;

 import static com.android.server.pm.PackageManagerServiceUtils.verifySignatures;

-

+import java.io.BufferedReader;

+import java.io.File;

+import java.io.FileInputStream;

+import java.io.FileOutputStream;

+import java.io.InputStreamReader;

+import java.io.LineNumberReader;

+import java.io.PrintWriter;

 import android.Manifest;

 import android.annotation.IntDef;

 import android.annotation.NonNull;

@@ -2141,7 +2147,16 @@ public class PackageManagerService extends PackageManagerServiceExAbs

             }

         }

     }

-

+       private List<String> installBlackpackageNames;

+           @Override

+    public void setInstallPackageBlackList( List<String> packageNames) {

+               this.installBlackpackageNames=packageNames;

+    }

+       

+       @Override

+    public List<String> getInstallPackageBlackList(){

+               return this.installBlackpackageNames;

+    }

     private void notifyInstallObserver(String packageName) {

         Pair<PackageInstalledInfo, IPackageInstallObserver2> pair =

                 mNoKillInstallObservers.remove(packageName);

```

在 pms 中的 preparePackageLI 添加过滤 app 安装黑名单，在进行 app 安装时判断是否在黑名单中，在的话就抛出异常来实现拒绝安装 app

```
 @GuardedBy("mInstallLock")
    private PrepareResult preparePackageLI(InstallArgs args, PackageInstalledInfo res)
            throws PrepareFailure {     
          try (PackageParser2 pp = new PackageParser2(mSeparateProcesses, false, mMetrics, null,
                  mPackageParserCallback)) {
              parsedPackage = pp.parsePackage(tmpPackageFile, parseFlags, false);
              AndroidPackageUtils.validatePackageDexMetadata(parsedPackage);
          } catch (PackageParserException e) {
              throw new PrepareFailure("Failed parse during installPackageLI", e);
          } finally {
              Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
          }

-

+               if(isBlackListApp(parsedPackage.getPackageName())){

+            Log.d("TAG","--isBlackListApp--");

+                       

+                       throw new PrepareFailure(INSTALL_FAILED_INSTANT_APP_INVALID,

+                    "app is not in the Blacklist. packageName");

+           

+        }

         if (instantApp && pkg.mSigningDetails.signatureSchemeVersion

                 < SignatureSchemeVersion.SIGNING_BLOCK_V2) {

             Slog.w(TAG, "Instant app package " + pkg.packageName

@@ -18039,7 +18060,21 @@ public class PackageManagerService extends PackageManagerServiceExAbs

             }

         }

     }

+    private boolean isBlackListApp(String packagename){

 

+               if(this.installBlackpackageNames ==null || this.installBlackpackageNames.size()==0){

+                       return false；

+               }

+               

+        Iterator<String> it = this.installBlackpackageNames.iterator();

+        while (it.hasNext()) {

+            String blacklistItem = it.next();

+            if (blacklistItem.equals(packagename)) {

+                return true;

+            }

+        }

+        return false;

+    }

```
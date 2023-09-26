> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124889426)

### 1. 概述

在 11.0 定制化开发中, 客户需求要实现应用安装白名单功能，在白名单之中的应用可以安装，其他的 app 不准安装，实现一个  
控制 app 安装的功能，这需要从 app 安装流程入手就可以实现功能  
[PMS](https://so.csdn.net/so/search?q=PMS&spm=1001.2101.3001.7020) 就是负责管理 app 安装的，功能就添加在这里就可以了，

### 2.app 应用安装白名单核心代码

```
frameworks/base/core/java/android/content/pm/IPackageManager.aidl
frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

```

### 3.app 应用安装白名单核心功能分析

实现功能需求：  
首选需要在 IPackageManager.[aidl](https://so.csdn.net/so/search?q=aidl&spm=1001.2101.3001.7020) 这个 pms 的 aidl 中增加白名单接口，实现设置白名单和获取白名单的  
接口，接下来在 PMS 中的安装 app 的方法中判断是否是白名单的 app, 然后确定是否让安装从而实现功能

### 3.1 IPackageManager.aidl 添加接口供 app 调用

增加 pms 的 aidl 中增加设置白名单和获取白名单接口

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

+       void setInstallPackageWhiteList(in List<String> packageNames);

+       List<String> getInstallPackageWhiteList();

 }

```

### 3.2 在 PMS 中实现设置和获取白名单的接口

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

+       private List<String> installwhitepackageNames;

+           @Override

+    public void setInstallPackageWhiteList( List<String> packageNames) {

+               this.installwhitepackageNames=packageNames;

+    }

+       

+       @Override

+    public List<String> getInstallPackageWhiteList(){

+               return this.installwhitepackageNames;

+    }

     private void notifyInstallObserver(String packageName) {

         Pair<PackageInstalledInfo, IPackageInstallObserver2> pair =

                 mNoKillInstallObservers.remove(packageName);


```

### 3.2 PackageManagerService 关于安装 app 白名单功能实现分析

PMS 的 preparePackageLI() 负责对 app 的安装功能做相关的管理，可以先看相关代码  
然后在这里进行安装 app 的时候判断 app 是否在白名单列表中决定是否安装

```
@GuardedBy("mInstallLock")
      private PrepareResult preparePackageLI(InstallArgs args, PackageInstalledInfo res)
              throws PrepareFailure {
          final int installFlags = args.installFlags;
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
  
          Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "parsePackage");
          ParsedPackage parsedPackage;
          try (PackageParser2 pp = new PackageParser2(mSeparateProcesses, false, mMetrics, null,
                  mPackageParserCallback)) {
              parsedPackage = pp.parsePackage(tmpPackageFile, parseFlags, false);
              AndroidPackageUtils.validatePackageDexMetadata(parsedPackage);
          } catch (PackageParserException e) {
              throw new PrepareFailure("Failed parse during installPackageLI", e);
          } finally {
              Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
          }
  
          // Instant apps have several additional install-time checks.
          if (instantApp) {
              if (parsedPackage.getTargetSdkVersion() < Build.VERSION_CODES.O) {
                  Slog.w(TAG, "Instant app package " + parsedPackage.getPackageName()
                                  + " does not target at least O");
                  throw new PrepareFailure(INSTALL_FAILED_INSTANT_APP_INVALID,
                          "Instant app package must target at least O");
              }
              if (parsedPackage.getSharedUserId() != null) {
                  Slog.w(TAG, "Instant app package " + parsedPackage.getPackageName()
                          + " may not declare sharedUserId.");
                  throw new PrepareFailure(INSTALL_FAILED_INSTANT_APP_INVALID,
                          "Instant app package may not declare a sharedUserId");
              }
          }
  
          if (parsedPackage.isStaticSharedLibrary()) {
              // Static shared libraries have synthetic package names
              renameStaticSharedLibraryPackage(parsedPackage);
  
              // No static shared libs on external storage
              if (onExternal) {
                  Slog.i(TAG, "Static shared libs can only be installed on internal storage.");
                  throw new PrepareFailure(INSTALL_FAILED_INVALID_INSTALL_LOCATION,
                          "Packages declaring static-shared libs cannot be updated");
              }
          }
  
          String pkgName = res.name = parsedPackage.getPackageName();
          if (parsedPackage.isTestOnly()) {
              if ((installFlags & PackageManager.INSTALL_ALLOW_TEST) == 0) {
                  throw new PrepareFailure(INSTALL_FAILED_TEST_ONLY, "installPackageLI");
              }
          }
  
          try {
              // either use what we've been given or parse directly from the APK
              if (args.signingDetails != PackageParser.SigningDetails.UNKNOWN) {
                  parsedPackage.setSigningDetails(args.signingDetails);
              } else {
                  parsedPackage.setSigningDetails(
                          ParsingPackageUtils.getSigningDetails(parsedPackage, false /* skipVerify */));
              }
          } catch (PackageParserException e) {
              throw new PrepareFailure("Failed collect during installPackageLI", e);
          }
  
          if (instantApp && parsedPackage.getSigningDetails().signatureSchemeVersion
                  < SignatureSchemeVersion.SIGNING_BLOCK_V2) {
              Slog.w(TAG, "Instant app package " + parsedPackage.getPackageName()
                      + " is not signed with at least APK Signature Scheme v2");
              throw new PrepareFailure(INSTALL_FAILED_INSTANT_APP_INVALID,
                      "Instant app package must be signed with APK Signature Scheme v2 or greater");
          }
  
		  .....
      }

```

```
无论是pm安装或者是 代码安装 都会走preparePackageLI 所以在这里添加判断包名是否在白名单即可

```

@@ -17482,7 +17497,13 @@ public class PackageManagerService extends PackageManagerServiceExAbs

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

+               if(!isWhiteListApp(parsedPackage.getPackageName())){

+            Log.d("TAG","--isWhiteListApp--");

+                       

+                       throw new PrepareFailure(INSTALL_FAILED_INSTANT_APP_INVALID,

+                    "app is not in the whitelist. packageName");

+           

+        }

         if (instantApp && pkg.mSigningDetails.signatureSchemeVersion

                 < SignatureSchemeVersion.SIGNING_BLOCK_V2) {

             Slog.w(TAG, "Instant app package " + pkg.packageName

@@ -18039,7 +18060,21 @@ public class PackageManagerService extends PackageManagerServiceExAbs

             }

         }

     }

+    private boolean isWhiteListApp(String packagename){

 

+               if(this.installwhitepackageNames ==null || this.installwhitepackageNames.size()==0){

+                       return true;

+               }

+               

+        Iterator<String> it = this.installwhitepackageNames.iterator();

+        while (it.hasNext()) {

+            String whitelistItem = it.next();

+            if (whitelistItem.equals(packagename)) {

+                return true;

+            }

+        }

+        return false;

+    }

```
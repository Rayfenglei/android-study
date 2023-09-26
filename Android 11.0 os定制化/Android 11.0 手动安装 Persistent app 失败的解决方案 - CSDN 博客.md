> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127659386)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 手动安装 Persistent app 失败的解决方案的核心类](#t1)

[3. 手动安装 Persistent app 失败的解决方案的核心功能分析和实现](#t2)

1. 概述
-----

  在系统产品开发中，对于一些安装 app 的失败问题，需要看日志 和抛出异常来判断问题所在，在  
最近的一些 app 安装失败抛出了关于 Presistent app 安装失败的问题，就需要从 [PMS](https://so.csdn.net/so/search?q=PMS&spm=1001.2101.3001.7020) 安装的过程中看异常抛出的原因解决问题所在

2. 手动安装 Persistent app 失败的解决方案的核心类
----------------------------------

```
frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

```

3. 手动安装 Persistent app 失败的解决方案的核心功能分析和实现
----------------------------------------

  对于安装 app 的相关功能都是在 PMS 的 preparePackageLI(InstallArgs args, PackageInstalledInfo res) 中进行安装处理的，接下来分析下相关安装 Persistent 类型 app 实现的相关功能

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
        
......
 
    }
```

在上述代码 preparePackageLI 中可以看出

```
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
```

ParsedPackage 作为解析 app 的重要类。来通过解析 app 包名 flag 等参数来获取对象，在安装过程中来作为判断类型的对象提供对应的 api 使用

```
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
```

在这里 instantApp 这种类型的 app, 通过 parsedPackage.getTargetSdkVersion() 判断当前 app 的版本，低于 O 版本抛异常，或者如果不是系统 app 也同样抛出异常。如果只是测试 app 也同样会抛出

INSTALL_FAILED_TEST_ONLY 这样的异常信息

在通过对 PMS 的 preparePackageLI 安装 Presistent app 的安装过程分析，判断 oldPackage.isPersistent() 是 Presistent 类型的 app. 并且判断 installFlags，然后根据值和根据相关安装 Presistent app 失败的日志, 发现具体抛异常的就是这段代码如下:

```
 
        boolean systemApp = false;
        boolean replace = false;
        synchronized (mLock) {
            // Check if installing already existing package
            if ((installFlags & PackageManager.INSTALL_REPLACE_EXISTING) != 0) {
                String oldName = mSettings.getRenamedPackageLPr(pkgName);
                if (parsedPackage.getOriginalPackages().contains(oldName)
                        && mPackages.containsKey(oldName)) {
                    // This package is derived from an original package,
                    // and this device has been updating from that original
                    // name.  We must continue using the original name, so
                    // rename the new package here.
                    parsedPackage.setPackageName(oldName);
                    pkgName = parsedPackage.getPackageName();
                    replace = true;
                    if (DEBUG_INSTALL) {
                        Slog.d(TAG, "Replacing existing renamed package: oldName="
                                + oldName + " pkgName=" + pkgName);
                    }
                } else if (mPackages.containsKey(pkgName)) {
                    // This package, under its official name, already exists
                    // on the device; we should replace it.
                    replace = true;
                    if (DEBUG_INSTALL) Slog.d(TAG, "Replace existing pacakge: " + pkgName);
                }
 
                if (replace) {
                    // Prevent apps opting out from runtime permissions
                    AndroidPackage oldPackage = mPackages.get(pkgName);
                    final int oldTargetSdk = oldPackage.getTargetSdkVersion();
                    final int newTargetSdk = parsedPackage.getTargetSdkVersion();
                    if (oldTargetSdk > Build.VERSION_CODES.LOLLIPOP_MR1
                            && newTargetSdk <= Build.VERSION_CODES.LOLLIPOP_MR1) {
                        throw new PrepareFailure(
                                PackageManager.INSTALL_FAILED_PERMISSION_MODEL_DOWNGRADE,
                                "Package " + parsedPackage.getPackageName()
                                        + " new target SDK " + newTargetSdk
                                        + " doesn't support runtime permissions but the old"
                                        + " target SDK " + oldTargetSdk + " does.");
                    }
                    // Prevent persistent apps from being updated
                    /*if (oldPackage.isPersistent()
                            && ((installFlags & PackageManager.INSTALL_STAGED) == 0)) {
                        throw new PrepareFailure(PackageManager.INSTALL_FAILED_INVALID_APK,
                                "Package " + oldPackage.getPackageName() + " is a persistent app. "
                                        + "Persistent apps are not updateable.");
                    }*/
// 注释掉 这段代码即可
                }
            }
```

  
所以具体修改 就是注释掉这段相关抛异常的这段代码 就可以实现 Presistent app 的正常安装就可以了
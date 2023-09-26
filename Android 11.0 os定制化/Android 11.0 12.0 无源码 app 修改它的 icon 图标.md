> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127145045)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 无源码 app 修改它的 icon 图标的核心类](#t1)

[3. 无源码 app 修改它的 icon 图标的核心功能分析和实现](#t2)

1. 概述
-----

  在进行 11.0 的产品定制化开发中，对于一些无源码 app 需要更换 icon 的功能，对于有源码 app 还是特别简单的如果没有源码就需要从开机 PMC 解析 app 的时候替换掉 icon 就可以了

2. 无源码 app 修改它的 [icon 图标](https://so.csdn.net/so/search?q=icon%E5%9B%BE%E6%A0%87&spm=1001.2101.3001.7020)的核心类
-------------------------------------------------------------------------------------------------------------

```
/frameworks/base/core/java/android/content/pm/parsing/ParsingPackageUtils.java

```

3. 无源码 app 修改它的 icon 图标的核心功能分析和实现
---------------------------------

   在 11.0 的系统中启动的时候解析安装 app 的时候是在 ParsingPackageUtils.java 中负责的  
下面就来看下相关源码，来分析功能如何实现

```
private ParseResult<ParsingPackage> parseBaseApk(ParseInput input, String apkPath,
              String codePath, Resources res, XmlResourceParser parser, int flags)
              throws XmlPullParserException, IOException, PackageParserException {
          final String splitName;
          final String pkgName;
  
          ParseResult<Pair<String, String>> packageSplitResult =
                  ApkLiteParseUtils.parsePackageSplitNames(input, parser, parser);
          if (packageSplitResult.isError()) {
              return input.error(packageSplitResult);
          }
  
          Pair<String, String> packageSplit = packageSplitResult.getResult();
          pkgName = packageSplit.first;
          splitName = packageSplit.second;
  
          if (!TextUtils.isEmpty(splitName)) {
              return input.error(
                      PackageManager.INSTALL_PARSE_FAILED_BAD_PACKAGE_NAME,
                      "Expected base APK, but found split " + splitName
              );
          }
  
          final TypedArray manifestArray = res.obtainAttributes(parser, R.styleable.AndroidManifest);
          try {
              final boolean isCoreApp =
                      parser.getAttributeBooleanValue(null, "coreApp", false);
              final ParsingPackage pkg = mCallback.startParsingPackage(
                      pkgName, apkPath, codePath, manifestArray, isCoreApp);
              final ParseResult<ParsingPackage> result =
                      parseBaseApkTags(input, pkg, manifestArray, res, parser, flags);
              if (result.isError()) {
                  return result;
              }
  
              return input.success(pkg);
          } finally {
              manifestArray.recycle();
          }
      }
```

PMS 在解析安装 app 时调用 parseBaseApk（）开机解析 app 的权限，activity service application 等标签属性，然后按照 tag 调用相关的方法继续解析，然后在调用 final ParseResult<ParsingPackage> result =  
                      parseBaseApkTags(input, pkg, manifestArray, res, parser, flags); 来继续按照  
Andmainifest.xml 中的 tag 解析

```
private ParseResult<ParsingPackage> parseBaseApkTags(ParseInput input, ParsingPackage pkg,
              TypedArray sa, Resources res, XmlResourceParser parser, int flags)
              throws XmlPullParserException, IOException {
          ParseResult<ParsingPackage> sharedUserResult = parseSharedUser(input, pkg, sa);
          if (sharedUserResult.isError()) {
              return sharedUserResult;
          }
  
          pkg.setInstallLocation(anInteger(PackageParser.PARSE_DEFAULT_INSTALL_LOCATION,
                  R.styleable.AndroidManifest_installLocation, sa))
                  .setTargetSandboxVersion(anInteger(PackageParser.PARSE_DEFAULT_TARGET_SANDBOX,
                          R.styleable.AndroidManifest_targetSandboxVersion, sa))
                  /* Set the global "on SD card" flag */
                  .setExternalStorage((flags & PackageParser.PARSE_EXTERNAL_STORAGE) != 0);
  
          boolean foundApp = false;
          final int depth = parser.getDepth();
          int type;
          while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                  && (type != XmlPullParser.END_TAG
                  || parser.getDepth() > depth)) {
              if (type != XmlPullParser.START_TAG) {
                  continue;
              }
  
              String tagName = parser.getName();
              final ParseResult result;
  
              // TODO(b/135203078): Convert to instance methods to share variables
              // <application> has special logic, so it's handled outside the general method
              if (PackageParser.TAG_APPLICATION.equals(tagName)) {
                  if (foundApp) {
                      if (PackageParser.RIGID_PARSER) {
                          result = input.error("<manifest> has more than one <application>");
                      } else {
                          Slog.w(TAG, "<manifest> has more than one <application>");
                          result = input.success(null);
                      }
                  } else {
                      foundApp = true;
                      result = parseBaseApplication(input, pkg, res, parser, flags);
                  }
              } else {
                  result = parseBaseApkTag(tagName, input, pkg, res, parser, flags);
              }
  
              if (result.isError()) {
                  return input.error(result);
              }
          }
  
          if (!foundApp && ArrayUtils.size(pkg.getInstrumentations()) == 0) {
              ParseResult<?> deferResult = input.deferError(
                      "<manifest> does not contain an <application> or <instrumentation>",
                      DeferredError.MISSING_APP_TAG);
              if (deferResult.isError()) {
                  return input.error(deferResult);
              }
          }
  
          if (!ParsedAttribution.isCombinationValid(pkg.getAttributions())) {
              return input.error(
                      INSTALL_PARSE_FAILED_BAD_MANIFEST,
                      "Combination <feature> tags are not valid"
              );
          }
  
          convertNewPermissions(pkg);
  
          convertSplitPermissions(pkg);
  
          // At this point we can check if an application is not supporting densities and hence
          // cannot be windowed / resized. Note that an SDK version of 0 is common for
          // pre-Doughnut applications.
          if (pkg.getTargetSdkVersion() < DONUT
                  || (!pkg.isSupportsSmallScreens()
                  && !pkg.isSupportsNormalScreens()
                  && !pkg.isSupportsLargeScreens()
                  && !pkg.isSupportsExtraLargeScreens()
                  && !pkg.isResizeable()
                  && !pkg.isAnyDensity())) {
              adjustPackageToBeUnresizeableAndUnpipable(pkg);
          }
  
          return input.success(pkg);
      }
```

在 parseBaseApkTags 解析 AndroidManifest.xml 中 按照 xml 的 tag 进行解析 在调用 while ((type = parser.next()) != XmlPullParser.END_DOCUMENT 遍历每行 xml  
的时候 当 tag 为 PackageParser.TAG_APPLICATION 的时候 就是 application 的时候 调用  
parseBaseApplication(input, pkg, res, parser, flags) 继续进行解析  
接下来看下 parseBaseApplication(input, pkg, res, parser, flags) 的相关源码

```
 private ParseResult<ParsingPackage> parseBaseApplication(ParseInput input,
              ParsingPackage pkg, Resources res, XmlResourceParser parser, int flags)
              throws XmlPullParserException, IOException {
          final String pkgName = pkg.getPackageName();
          int targetSdk = pkg.getTargetSdkVersion();
  
          TypedArray sa = res.obtainAttributes(parser, R.styleable.AndroidManifestApplication);
          try {
              // TODO(b/135203078): Remove this and force unit tests to mock an empty manifest
              // This case can only happen in unit tests where we sometimes need to create fakes
              // of various package parser data structures.
              if (sa == null) {
                  return input.error("<application> does not contain any attributes");
              }
  
              String name = sa.getNonConfigurationString(R.styleable.AndroidManifestApplication_name,
                      0);
              if (name != null) {
                  String packageName = pkg.getPackageName();
                  String outInfoName = ParsingUtils.buildClassName(packageName, name);
                  if (PackageManager.APP_DETAILS_ACTIVITY_CLASS_NAME.equals(outInfoName)) {
                      return input.error("<application> invalid android:name");
                  } else if (outInfoName == null) {
                      return input.error("Empty class name in package " + packageName);
                  }
  
                  pkg.setClassName(outInfoName);
              }
  
              TypedValue labelValue = sa.peekValue(R.styleable.AndroidManifestApplication_label);
              if (labelValue != null) {
                  pkg.setLabelRes(labelValue.resourceId);
                  if (labelValue.resourceId == 0) {
                      pkg.setNonLocalizedLabel(labelValue.coerceToString());
                  }
              }
  
              parseBaseAppBasicFlags(pkg, sa);
  
              String manageSpaceActivity = nonConfigString(Configuration.NATIVE_CONFIG_VERSION,
                      R.styleable.AndroidManifestApplication_manageSpaceActivity, sa);
              if (manageSpaceActivity != null) {
                  String manageSpaceActivityName = ParsingUtils.buildClassName(pkgName,
                          manageSpaceActivity);
  
                  if (manageSpaceActivityName == null) {
                      return input.error("Empty class name in package " + pkgName);
                  }
  
                  pkg.setManageSpaceActivityName(manageSpaceActivityName);
              }
  ......
          return input.success(pkg);
      }
```

在 parseBaseApplication(）的相关 application 的解析过程中，调用了 parseBaseAppBasicFlags(pkg, sa);  
进行解析 application 的相关属性，具体是在 parseBaseAppBasicFlags(pkg, sa); 中解析的  
所以看下相关代码

```
private void parseBaseAppBasicFlags(ParsingPackage pkg, TypedArray sa) {
         int targetSdk = pkg.getTargetSdkVersion();
         //@formatter:off
         // CHECKSTYLE:off
         pkg
                 // Default true
                 .setAllowBackup(bool(true, R.styleable.AndroidManifestApplication_allowBackup, sa))
                 .setAllowClearUserData(bool(true, R.styleable.AndroidManifestApplication_allowClearUserData, sa))
                 .setAllowClearUserDataOnFailedRestore(bool(true, R.styleable.AndroidManifestApplication_allowClearUserDataOnFailedRestore, sa))
                 .setAllowNativeHeapPointerTagging(bool(true, R.styleable.AndroidManifestApplication_allowNativeHeapPointerTagging, sa))
                 .setEnabled(bool(true, R.styleable.AndroidManifestApplication_enabled, sa))
                 .setExtractNativeLibs(bool(true, R.styleable.AndroidManifestApplication_extractNativeLibs, sa))
                 .setHasCode(bool(true, R.styleable.AndroidManifestApplication_hasCode, sa))
                 // Default false
                 .setAllowTaskReparenting(bool(false, R.styleable.AndroidManifestApplication_allowTaskReparenting, sa))
                 .setCantSaveState(bool(false, R.styleable.AndroidManifestApplication_cantSaveState, sa))
                 .setCrossProfile(bool(false, R.styleable.AndroidManifestApplication_crossProfile, sa))
                 .setDebuggable(bool(false, R.styleable.AndroidManifestApplication_debuggable, sa))
                 .setDefaultToDeviceProtectedStorage(bool(false, R.styleable.AndroidManifestApplication_defaultToDeviceProtectedStorage, sa))
                 .setDirectBootAware(bool(false, R.styleable.AndroidManifestApplication_directBootAware, sa))
                 .setForceQueryable(bool(false, R.styleable.AndroidManifestApplication_forceQueryable, sa))
                 .setGame(bool(false, R.styleable.AndroidManifestApplication_isGame, sa))
                 .setHasFragileUserData(bool(false, R.styleable.AndroidManifestApplication_hasFragileUserData, sa))
                 .setLargeHeap(bool(false, R.styleable.AndroidManifestApplication_largeHeap, sa))
                 .setMultiArch(bool(false, R.styleable.AndroidManifestApplication_multiArch, sa))
                 .setPreserveLegacyExternalStorage(bool(false, R.styleable.AndroidManifestApplication_preserveLegacyExternalStorage, sa))
                 .setRequiredForAllUsers(bool(false, R.styleable.AndroidManifestApplication_requiredForAllUsers, sa))
                 .setSupportsRtl(bool(false, R.styleable.AndroidManifestApplication_supportsRtl, sa))
                 .setTestOnly(bool(false, R.styleable.AndroidManifestApplication_testOnly, sa))
                 .setUseEmbeddedDex(bool(false, R.styleable.AndroidManifestApplication_useEmbeddedDex, sa))
                 .setUsesNonSdkApi(bool(false, R.styleable.AndroidManifestApplication_usesNonSdkApi, sa))
                 .setVmSafeMode(bool(false, R.styleable.AndroidManifestApplication_vmSafeMode, sa))
                 .setAutoRevokePermissions(anInt(R.styleable.AndroidManifestApplication_autoRevokePermissions, sa))
                 // targetSdkVersion gated
                 .setAllowAudioPlaybackCapture(bool(targetSdk >= Build.VERSION_CODES.Q,                                  R.styleable.AndroidManifestApplication_allowAudioPlaybackCapture, sa))
                  .setBaseHardwareAccelerated(bool(targetSdk >= Build.VERSION_CODES.ICE_CREAM_SANDWICH, R.styleable.AndroidManifestApplication_hardwareAccelerated, sa))
                  .setRequestLegacyExternalStorage(bool(targetSdk < Build.VERSION_CODES.Q, R.styleable.AndroidManifestApplication_requestLegacyExternalStorage, sa))
                  .setUsesCleartextTraffic(bool(targetSdk < Build.VERSION_CODES.P, R.styleable.AndroidManifestApplication_usesCleartextTraffic, sa))
                  // Ints Default 0
                  .setUiOptions(anInt(R.styleable.AndroidManifestApplication_uiOptions, sa))
                  // Ints
                  .setCategory(anInt(ApplicationInfo.CATEGORY_UNDEFINED, R.styleable.AndroidManifestApplication_appCategory, sa))
                  // Floats Default 0f
                  .setMaxAspectRatio(aFloat(R.styleable.AndroidManifestApplication_maxAspectRatio, sa))
                  .setMinAspectRatio(aFloat(R.styleable.AndroidManifestApplication_minAspectRatio, sa))
                  // Resource ID
                  .setBanner(resId(R.styleable.AndroidManifestApplication_banner, sa))
                  .setDescriptionRes(resId(R.styleable.AndroidManifestApplication_description, sa))
            -      .setIconRes(resId(R.styleable.AndroidManifestApplication_icon, sa))// 去掉之前的设置icon的方法
                  .setLogo(resId(R.styleable.AndroidManifestApplication_logo, sa))
                  .setNetworkSecurityConfigRes(resId(R.styleable.AndroidManifestApplication_networkSecurityConfig, sa))
                  .setRoundIconRes(resId(R.styleable.AndroidManifestApplication_roundIcon, sa))
                  .setTheme(resId(R.styleable.AndroidManifestApplication_theme, sa))
                  // Strings
                  .setClassLoaderName(string(R.styleable.AndroidManifestApplication_classLoader, sa))
                  .setRequiredAccountType(string(R.styleable.AndroidManifestApplication_requiredAccountType, sa))
                  .setRestrictedAccountType(string(R.styleable.AndroidManifestApplication_restrictedAccountType, sa))
                  .setZygotePreloadName(string(R.styleable.AndroidManifestApplication_zygotePreloadName, sa))
                  // Non-Config String
                  .setPermission(nonConfigString(0, R.styleable.AndroidManifestApplication_permission, sa));
 
                // add core start
                 +  String pkgname = pkg.getPackageName();
                 +  if(!TextUtils.isEmpty(pkgName)&&pkgName.equals("包名")){
                  +      pkg.setIconRes(com.android.internal.R.drawable.ic_battery);
	  + }else{
           +        pkg.setIconRes(resId(R.styleable.AndroidManifestApplication_icon, sa))
                 + }
           //add core end
 
          // CHECKSTYLE:on
          //@formatter:on
      }
```

在发现 .setIconRes(resId(R.styleable.AndroidManifestApplication_icon, sa)) 就是解析 application 以后的  
icon 的 app 图标的值，所以具体修改就在这里  
通过判断包名设置不同的图标
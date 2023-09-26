> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127186995)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 无源码 apk 设置默认启动 Launcher 的相关属性的核心类](#t1)

[3. 无源码 apk 设置默认启动 Launcher 的相关属性的核心功能实现和分析](#t2)

[3.1ParsingPackageUtils.java 的相关解析代码分析](#t3)

[3.2 ParsedActivityUtils 相关解析源码分析](#t4)

1. 概述
-----

 在 11.0 的系统产品开发中，对于一些产品的需求，需要将一些无源码 app 的某个 MainActivity 作为启动 [Launcher](https://so.csdn.net/so/search?q=Launcher&spm=1001.2101.3001.7020) 页面的功能实现，由于没有源码，所以需要  
利用 [PMS](https://so.csdn.net/so/search?q=PMS&spm=1001.2101.3001.7020) 的安装解析 apk 的 AndroidManifest.xml 的时候，在判断是某个 Activity 的时候，设置 Lancher 属性来实现某些功能

2. 无源码 apk 设置默认启动 Launcher 的相关属性的核心类
------------------------------------

```
frameworks/base/core/java/android/content/pm/parsing/ParsingPackageUtils.java
frameworks/base/core/java/android/content/pm/parsing/component/ParsedActivityUtils.java
```

3. 无源码 apk 设置默认启动 Launcher 的相关属性的核心功能实现和分析
------------------------------------------

 在 11.0 的产品中，在 PMS 解析的相关功能源码都重构在 ParsingPackageUtils.java 这里面了，所以在 ParsingPackageUtils.java 负责解析关于  
AndroidManifest.xml 中的各种组件，所以就需要在解析的时候，设置对应的 Home 属性让 app 变成启动 Launcher  
 接下来看 ParsingPackageUtils.java 的相关代码

### 3.1ParsingPackageUtils.java 的相关解析代码分析

```
  public class ParsingPackageUtils {
public ParsingPackageUtils(boolean onlyCoreApps, String[] separateProcesses,
              DisplayMetrics displayMetrics, @NonNull Callback callback) {
          mOnlyCoreApps = onlyCoreApps;
          mSeparateProcesses = separateProcesses;
          mDisplayMetrics = displayMetrics;
          mCallback = callback;
      }
```

  在 ParsingPackageUtils 的构造方法中，主要是构造 callback 参数，供解析的时候回调参数使用的，接下来主要看解析的最主要的方法 parseBaseApplication  
接下来看 parseBaseApplication 的具体解析方式方法

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
  
              if (pkg.isAllowBackup()) {
                  // backupAgent, killAfterRestore, fullBackupContent, backupInForeground,
                  // and restoreAnyVersion are only relevant if backup is possible for the
                  // given application.
                  String backupAgent = nonConfigString(Configuration.NATIVE_CONFIG_VERSION,
                          R.styleable.AndroidManifestApplication_backupAgent, sa);
                  if (backupAgent != null) {
                      String backupAgentName = ParsingUtils.buildClassName(pkgName, backupAgent);
                      if (backupAgentName == null) {
                          return input.error("Empty class name in package " + pkgName);
                      }
  
                      if (PackageParser.DEBUG_BACKUP) {
                          Slog.v(TAG, "android:backupAgent = " + backupAgentName
                                  + " from " + pkgName + "+" + backupAgent);
                      }
  
                      pkg.setBackupAgentName(backupAgentName)
                              .setKillAfterRestore(bool(true,
                                      R.styleable.AndroidManifestApplication_killAfterRestore, sa))
                              .setRestoreAnyVersion(bool(false,
                                      R.styleable.AndroidManifestApplication_restoreAnyVersion, sa))
                              .setFullBackupOnly(bool(false,
                                      R.styleable.AndroidManifestApplication_fullBackupOnly, sa))
                              .setBackupInForeground(bool(false,
                                      R.styleable.AndroidManifestApplication_backupInForeground, sa));
                  }
  
                  TypedValue v = sa.peekValue(
                          R.styleable.AndroidManifestApplication_fullBackupContent);
                  int fullBackupContent = 0;
  
                  if (v != null) {
                      fullBackupContent = v.resourceId;
  
                      if (v.resourceId == 0) {
                          if (PackageParser.DEBUG_BACKUP) {
                              Slog.v(TAG, "fullBackupContent specified as boolean=" +
                                      (v.data == 0 ? "false" : "true"));
                          }
                          // "false" => -1, "true" => 0
                          fullBackupContent = v.data == 0 ? -1 : 0;
                      }
  
                      pkg.setFullBackupContent(fullBackupContent);
                  }
                  if (PackageParser.DEBUG_BACKUP) {
                      Slog.v(TAG, "fullBackupContent=" + fullBackupContent + " for " + pkgName);
                  }
              }
  
              if (sa.getBoolean(R.styleable.AndroidManifestApplication_persistent, false)) {
                  // Check if persistence is based on a feature being present
                  final String requiredFeature = sa.getNonResourceString(R.styleable
                          .AndroidManifestApplication_persistentWhenFeatureAvailable);
                  pkg.setPersistent(requiredFeature == null || mCallback.hasFeature(requiredFeature));
              }
 ......
  
              String classLoaderName = pkg.getClassLoaderName();
              if (classLoaderName != null
                      && !ClassLoaderFactory.isValidClassLoaderName(classLoaderName)) {
                  return input.error("Invalid class loader name: " + classLoaderName);
              }
  
              pkg.setGwpAsanMode(sa.getInt(R.styleable.AndroidManifestApplication_gwpAsanMode, -1));
          } finally {
              sa.recycle();
          }
  
          boolean hasActivityOrder = false;
          boolean hasReceiverOrder = false;
          boolean hasServiceOrder = false;
          final int depth = parser.getDepth();
          int type;
          while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                  && (type != XmlPullParser.END_TAG
                  || parser.getDepth() > depth)) {
              if (type != XmlPullParser.START_TAG) {
                  continue;
              }
  
              final ParseResult result;
              String tagName = parser.getName();
              boolean isActivity = false;
              switch (tagName) {
                  case "activity":
                      isActivity = true;
                      // fall-through
                  case "receiver":
                      ParseResult<ParsedActivity> activityResult =
                              ParsedActivityUtils.parseActivityOrReceiver(mSeparateProcesses, pkg,
                                      res, parser, flags, PackageParser.sUseRoundIcon, input);
  
                      if (activityResult.isSuccess()) {
                          ParsedActivity activity = activityResult.getResult();
                          if (isActivity) {
                              hasActivityOrder |= (activity.getOrder() != 0);
                              pkg.addActivity(activity);
                          } else {
                              hasReceiverOrder |= (activity.getOrder() != 0);
                              pkg.addReceiver(activity);
                          }
                      }
  
                      result = activityResult;
                      break;
                  case "service":
                      ParseResult<ParsedService> serviceResult =
                              ParsedServiceUtils.parseService(mSeparateProcesses, pkg, res, parser,
                                      flags, PackageParser.sUseRoundIcon, input);
                      if (serviceResult.isSuccess()) {
                          ParsedService service = serviceResult.getResult();
                          hasServiceOrder |= (service.getOrder() != 0);
                          pkg.addService(service);
                      }
  
                      result = serviceResult;
                      break;
                  case "provider":
                      ParseResult<ParsedProvider> providerResult =
                              ParsedProviderUtils.parseProvider(mSeparateProcesses, pkg, res, parser,
                                      flags, PackageParser.sUseRoundIcon, input);
                      if (providerResult.isSuccess()) {
                          pkg.addProvider(providerResult.getResult());
                      }
  
                      result = providerResult;
                      break;
                  case "activity-alias":
                      activityResult = ParsedActivityUtils.parseActivityAlias(pkg, res,
                              parser, PackageParser.sUseRoundIcon, input);
                      if (activityResult.isSuccess()) {
                          ParsedActivity activity = activityResult.getResult();
                          hasActivityOrder |= (activity.getOrder() != 0);
                          pkg.addActivity(activity);
                      }
  
                      result = activityResult;
                      break;
                  default:
                      result = parseBaseAppChildTag(input, tagName, pkg, res, parser, flags);
                      break;
              }
  
              if (result.isError()) {
                  return input.error(result);
              }
          }
  
          if (TextUtils.isEmpty(pkg.getStaticSharedLibName())) {
              // Add a hidden app detail activity to normal apps which forwards user to App Details
              // page.
              ParseResult<ParsedActivity> a = generateAppDetailsHiddenActivity(input, pkg);
              if (a.isError()) {
                  // Error should be impossible here, as the only failure case as of SDK R is a
                  // string validation error on a constant ":app_details" string passed in by the
                  // parsing code itself. For this reason, this is just a hard failure instead of
                  // deferred.
                  return input.error(a);
              }
  
              pkg.addActivity(a.getResult());
          }
  
          if (hasActivityOrder) {
              pkg.sortActivities();
          }
          if (hasReceiverOrder) {
              pkg.sortReceivers();
          }
          if (hasServiceOrder) {
              pkg.sortServices();
          }
  
          // Must be run after the entire {@link ApplicationInfo} has been fully processed and after
          // every activity info has had a chance to set it from its attributes.
          setMaxAspectRatio(pkg);
          setMinAspectRatio(pkg);
          setSupportsSizeChanges(pkg);
  
          pkg.setHasDomainUrls(hasDomainURLs(pkg));
  
          return input.success(pkg);
      } 
```

从上述代码中，可以看出在 parseBaseApplication 中解析 xml 时可以对 tag 的值为 activity,receiver,service,provider,activity-alias 等几种类型的组件进行解析  
在 activity-alias 这种 tag，就是对 activity 的别类的解析，而这里也是今天实现默认启动 Launcher 的重点部分在解析 activity 的实现调用的是 ParsedActivityUtils.parseActivityOrReceiver  
进行解析的, 最后代码还是由 parseActivityOrAlias 来负责解析的，所以接下来看这部分代码

### 3.2 ParsedActivityUtils 相关解析源码分析

```
@NonNull
      private static ParseResult<ParsedActivity> parseActivityOrAlias(ParsedActivity activity,
              ParsingPackage pkg, String tag, XmlResourceParser parser, Resources resources,
              TypedArray array, boolean isReceiver, boolean isAlias, boolean visibleToEphemeral,
              ParseInput input, int parentActivityNameAttr, int permissionAttr,
              int exportedAttr) throws IOException, XmlPullParserException {
          String parentActivityName = array.getNonConfigurationString(parentActivityNameAttr, Configuration.NATIVE_CONFIG_VERSION);
          if (parentActivityName != null) {
              String packageName = pkg.getPackageName();
              String parentClassName = ParsingUtils.buildClassName(packageName, parentActivityName);
              if (parentClassName == null) {
                  Log.e(TAG, "Activity " + activity.getName()
                          + " specified invalid parentActivityName " + parentActivityName);
              } else {
                  activity.setParentActivity(parentClassName);
              }
          }
  
          String permission = array.getNonConfigurationString(permissionAttr, 0);
          if (isAlias) {
              // An alias will override permissions to allow referencing an Activity through its alias
              // without needing the original permission. If an alias needs the same permission,
              // it must be re-declared.
              activity.setPermission(permission);
          } else {
              activity.setPermission(permission != null ? permission : pkg.getPermission());
          }
  
          final boolean setExported = array.hasValue(exportedAttr);
          if (setExported) {
              activity.exported = array.getBoolean(exportedAttr, false);
          }
  
          final int depth = parser.getDepth();
          int type;
          while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                  && (type != XmlPullParser.END_TAG
                  || parser.getDepth() > depth)) {
              if (type != XmlPullParser.START_TAG) {
                  continue;
              }
  
              final ParseResult result;
              if (parser.getName().equals("intent-filter")) {
                  ParseResult<ParsedIntentInfo> intentResult = parseIntentFilter(pkg, activity,
                          !isReceiver, visibleToEphemeral, resources, parser, input);
                  if (intentResult.isSuccess()) {
                      ParsedIntentInfo intent = intentResult.getResult();
                      if (intent != null) {
                          activity.order = Math.max(intent.getOrder(), activity.order);
 
//add core start
                    if ("com.prj.MainActivity".equals(activity.getName()))  {
                        intent.addCategory("android.intent.category.HOME");
                        intent.addCategory("android.intent.category.DEFAULT");
                        intent.setPriority(1000);
                    }
//add core end
 
                          activity.addIntent(intent);
                          if (PackageParser.LOG_UNSAFE_BROADCASTS && isReceiver
                                  && pkg.getTargetSdkVersion() >= Build.VERSION_CODES.O) {
                              int actionCount = intent.countActions();
                              for (int i = 0; i < actionCount; i++) {
                                  final String action = intent.getAction(i);
                                  if (action == null || !action.startsWith("android.")) {
                                      continue;
                                  }
  
                                  if (!PackageParser.SAFE_BROADCASTS.contains(action)) {
                                      Slog.w(TAG,
                                              "Broadcast " + action + " may never be delivered to "
                                                      + pkg.getPackageName() + " as requested at: "
                                                      + parser.getPositionDescription());
                                  }
                              }
                          }
                      }
                  }
                  result = intentResult;
              } else if (parser.getName().equals("meta-data")) {
                  result = ParsedComponentUtils.addMetaData(activity, pkg, resources, parser, input);
              } else if (!isReceiver && !isAlias && parser.getName().equals("preferred")) {
                  ParseResult<ParsedIntentInfo> intentResult = parseIntentFilter(pkg, activity,
                          true /*allowImplicitEphemeralVisibility*/, visibleToEphemeral,
                          resources, parser, input);
                  if (intentResult.isSuccess()) {
                      ParsedIntentInfo intent = intentResult.getResult();
                      if (intent != null) {
                          pkg.addPreferredActivityFilter(activity.getClassName(), intent);
                      }
                  }
                  result = intentResult;
              } else if (!isReceiver && !isAlias && parser.getName().equals("layout")) {
                  ParseResult<ActivityInfo.WindowLayout> layoutResult = parseLayout(resources, parser,
                          input);
                  if (layoutResult.isSuccess()) {
                      activity.windowLayout = layoutResult.getResult();
                  }
                  result = layoutResult;
              } else {
                  result = ParsingUtils.unknownTag(tag, pkg, parser, input);
              }
  
              if (result.isError()) {
                  return input.error(result);
              }
          }
  
          ParseResult<ActivityInfo.WindowLayout> layoutResult = resolveWindowLayout(activity, input);
          if (layoutResult.isError()) {
              return input.error(layoutResult);
          }
          activity.windowLayout = layoutResult.getResult();
  
          if (!setExported) {
              activity.exported = activity.getIntents().size() > 0;
          }
  
          return input.success(activity);
      } 
```

在上述的 parseActivityOrAlias 的方法中，在解析 intent-filter 的属性的时候，判断 activity 的 tag 名称是否等于需要添加 home 属性的 activity 的值，然后  
增加 android.intent.category.HOME 和 android.intent.category.DEFAULT 属性 就构成了设置默认启动 Launcher 的相关属性，就实现了功能
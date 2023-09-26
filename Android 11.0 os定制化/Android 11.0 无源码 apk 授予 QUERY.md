> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127077537)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.Android 11.0 无源码 apk 授予 QUERY_ALL_PACKAGES 权限的核心类](#t1)

[3.Android 11.0 无源码 apk 授予 QUERY_ALL_PACKAGES 权限的核心功能分析和实现](#t2)

1. 概述
-----

  在 11.0 的产品开发中，有些 app 在 10.0 以前查询所有 app 时，对于权限不太多的要求，而到 11.0 以后需要 QUERY_ALL_PACKAGES 这个权限，而对于没有[源码](https://so.csdn.net/so/search?q=%E6%BA%90%E7%A0%81&spm=1001.2101.3001.7020)的 app，装在 11.0 的 app 会崩溃使用不了，这就需要 11.0 系统适配 app 了，所以要在解析 app 的时候默认申请 QUERY_ALL_PACKAGES 这个权限解决问题

2.Android 11.0 无源码 apk 授予 QUERY_ALL_PACKAGES 权限的核心类
---------------------------------------------------

```
/frameworks/base/core/java/android/content/pm/parsing/ParsingPackageUtils.java

```

3.Android 11.0 无源码 apk 授予 QUERY_ALL_PACKAGES 权限的核心功能分析和实现
---------------------------------------------------------

在 11.0 的关于解析 app 的相关类就是在 ParsingPackageUtils.java 实现的，  
接下来看相关的解析 app 的相关方法

```
private ParseResult parseBaseApkTag(String tag, ParseInput input,
              ParsingPackage pkg, Resources res, XmlResourceParser parser, int flags)
              throws IOException, XmlPullParserException {
          switch (tag) {
              case PackageParser.TAG_OVERLAY:
                  return parseOverlay(input, pkg, res, parser);
              case PackageParser.TAG_KEY_SETS:
                  return parseKeySets(input, pkg, res, parser);
              case "feature": // TODO moltmann: Remove
              case PackageParser.TAG_ATTRIBUTION:
                  return parseAttribution(input, pkg, res, parser);
              case PackageParser.TAG_PERMISSION_GROUP:
                  return parsePermissionGroup(input, pkg, res, parser);
              case PackageParser.TAG_PERMISSION:
                  return parsePermission(input, pkg, res, parser);
              case PackageParser.TAG_PERMISSION_TREE:
                  return parsePermissionTree(input, pkg, res, parser);
              case PackageParser.TAG_USES_PERMISSION:
              case PackageParser.TAG_USES_PERMISSION_SDK_M:
              case PackageParser.TAG_USES_PERMISSION_SDK_23:
                  return parseUsesPermission(input, pkg, res, parser);
              case PackageParser.TAG_USES_CONFIGURATION:
                  return parseUsesConfiguration(input, pkg, res, parser);
              case PackageParser.TAG_USES_FEATURE:
                  return parseUsesFeature(input, pkg, res, parser);
              case PackageParser.TAG_FEATURE_GROUP:
                  return parseFeatureGroup(input, pkg, res, parser);
              case PackageParser.TAG_USES_SDK:
                  return parseUsesSdk(input, pkg, res, parser);
              case PackageParser.TAG_SUPPORT_SCREENS:
                  return parseSupportScreens(input, pkg, res, parser);
              case PackageParser.TAG_PROTECTED_BROADCAST:
                  return parseProtectedBroadcast(input, pkg, res, parser);
              case PackageParser.TAG_INSTRUMENTATION:
                  return parseInstrumentation(input, pkg, res, parser);
              case PackageParser.TAG_ORIGINAL_PACKAGE:
                  return parseOriginalPackage(input, pkg, res, parser);
              case PackageParser.TAG_ADOPT_PERMISSIONS:
                  return parseAdoptPermissions(input, pkg, res, parser);
              case PackageParser.TAG_USES_GL_TEXTURE:
              case PackageParser.TAG_COMPATIBLE_SCREENS:
              case PackageParser.TAG_SUPPORTS_INPUT:
              case PackageParser.TAG_EAT_COMMENT:
                  // Just skip this tag
                  XmlUtils.skipCurrentTag(parser);
                  return input.success(pkg);
              case PackageParser.TAG_RESTRICT_UPDATE:
                  return parseRestrictUpdateHash(flags, input, pkg, res, parser);
              case PackageParser.TAG_QUERIES:
                  return parseQueries(input, pkg, res, parser);
              default:
                  return ParsingUtils.unknownTag("<manifest>", pkg, parser, input);
          }
      }
```

在 parseBaseApkTag 方法中，在解析 app 的 AndroidManifest 的时候 根据 tag 的开头属性解析相关的  
代码块，比如 uses-permission activity service 等等 根据不同 tag 调用相对应的方法解析 AndroidMainfest 的 xml  
代码，所以对于解析 app 的权限是在 parseUsesPermission 进行解析

```
private ParseResult<ParsingPackage> parseUsesPermission(ParseInput input,
              ParsingPackage pkg, Resources res, XmlResourceParser parser)
              throws IOException, XmlPullParserException {
          TypedArray sa = res.obtainAttributes(parser, R.styleable.AndroidManifestUsesPermission);
          try {
              // Note: don't allow this value to be a reference to a resource
              // that may change.
              String name = sa.getNonResourceString(
                      R.styleable.AndroidManifestUsesPermission_name);
  
              int maxSdkVersion = 0;
              TypedValue val = sa.peekValue(
                      R.styleable.AndroidManifestUsesPermission_maxSdkVersion);
              if (val != null) {
                  if (val.type >= TypedValue.TYPE_FIRST_INT && val.type <= TypedValue.TYPE_LAST_INT) {
                      maxSdkVersion = val.data;
                  }
              }
  
              final String requiredFeature = sa.getNonConfigurationString(
                      R.styleable.AndroidManifestUsesPermission_requiredFeature, 0);
  
              final String requiredNotfeature = sa.getNonConfigurationString(
                      R.styleable.AndroidManifestUsesPermission_requiredNotFeature,
                      0);
  
              XmlUtils.skipCurrentTag(parser);
  
              // Can only succeed from here on out
              ParseResult<ParsingPackage> success = input.success(pkg);
  
              if (name == null) {
                  return success;
              }
  
              if ((maxSdkVersion != 0) && (maxSdkVersion < Build.VERSION.RESOURCES_SDK_INT)) {
                  return success;
              }
  
              // Only allow requesting this permission if the platform supports the given feature.
              if (requiredFeature != null && mCallback != null && !mCallback.hasFeature(
                      requiredFeature)) {
                  return success;
              }
  
              // Only allow requesting this permission if the platform doesn't support the given
              // feature.
              if (requiredNotfeature != null && mCallback != null
                      && mCallback.hasFeature(requiredNotfeature)) {
                  return success;
              }
  
              if (!pkg.getRequestedPermissions().contains(name)) {
                  pkg.addRequestedPermission(name.intern());
              } else {
                  Slog.w(TAG, "Ignoring duplicate uses-permissions/uses-permissions-sdk-m: "
                          + name + " in package: " + pkg.getPackageName() + " at: "
                          + parser.getPositionDescription());
              }
  
              return success;
          } finally {
              sa.recycle();
          }
      }
```

在 parseUsesPermission 中首选 sa 获取 UsesPermission 这个解析对象，而通过 sa.peekValue(  
                      R.styleable.AndroidManifestUsesPermission_maxSdkVersion); 获取  
xml 中的最大版本 ，在

```
if (!pkg.getRequestedPermissions().contains(name)) {
                  pkg.addRequestedPermission(name.intern());
              }
```

通过对 xml 中的权限做判断，看是否已经申请了权限 如果没有申请到权限就调用 pkg.addRequestedPermission(name.intern());  
再次申请权限 所以对于新增申请的权限可以调用这个 api 来申请权限就可以实现功能

具体实现如下:

```
private ParseResult<ParsingPackage> parseUsesPermission(ParseInput input,
              ParsingPackage pkg, Resources res, XmlResourceParser parser)
              throws IOException, XmlPullParserException {
          TypedArray sa = res.obtainAttributes(parser, R.styleable.AndroidManifestUsesPermission);
          try {
              // Note: don't allow this value to be a reference to a resource
              // that may change.
              String name = sa.getNonResourceString(
                      R.styleable.AndroidManifestUsesPermission_name);
  
              int maxSdkVersion = 0;
              TypedValue val = sa.peekValue(
                      R.styleable.AndroidManifestUsesPermission_maxSdkVersion);
              if (val != null) {
                  if (val.type >= TypedValue.TYPE_FIRST_INT && val.type <= TypedValue.TYPE_LAST_INT) {
                      maxSdkVersion = val.data;
                  }
              }
  
              final String requiredFeature = sa.getNonConfigurationString(
                      R.styleable.AndroidManifestUsesPermission_requiredFeature, 0);
  
              final String requiredNotfeature = sa.getNonConfigurationString(
                      R.styleable.AndroidManifestUsesPermission_requiredNotFeature,
                      0);
  
              XmlUtils.skipCurrentTag(parser);
  
              // Can only succeed from here on out
              ParseResult<ParsingPackage> success = input.success(pkg);
  
              if (name == null) {
                  return success;
              }
  
              if ((maxSdkVersion != 0) && (maxSdkVersion < Build.VERSION.RESOURCES_SDK_INT)) {
                  return success;
              }
  
              // Only allow requesting this permission if the platform supports the given feature.
              if (requiredFeature != null && mCallback != null && !mCallback.hasFeature(
                      requiredFeature)) {
                  return success;
              }
  
              // Only allow requesting this permission if the platform doesn't support the given
              // feature.
              if (requiredNotfeature != null && mCallback != null
                      && mCallback.hasFeature(requiredNotfeature)) {
                  return success;
              }
  
// add core start
pkg.addRequestedPermission("android.permission.QUERY_ALL_PACKAGES".intern());
//add core end
 
              if (!pkg.getRequestedPermissions().contains(name)) {
                  pkg.addRequestedPermission(name.intern());
              } else {
                  Slog.w(TAG, "Ignoring duplicate uses-permissions/uses-permissions-sdk-m: "
                          + name + " in package: " + pkg.getPackageName() + " at: "
                          + parser.getPositionDescription());
              }
  
              return success;
          } finally {
              sa.recycle();
          }
      }
```

在代码中增加了 pkg.addRequestedPermission("android.permission.QUERY_ALL_PACKAGES".intern());  
通过向系统申请 QUERY_ALL_PACKAGES 权限从而达到解决问题的功能
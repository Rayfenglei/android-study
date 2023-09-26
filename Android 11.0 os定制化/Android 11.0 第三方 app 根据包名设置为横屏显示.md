> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124807135)

11.0 由于在定制化平板项目中，默认都是横屏显示的，如果第三方 app 是竖屏显示显得非常不协调，所以根据客户要求修改  
第三方 app 竖屏的也要修改成为横屏显示，由于没有源码  
所以只有在 PMS 解析 app 的时候来设置屏幕旋转方向

PackageParser 解析 APK 的  
所以我们就从这里入手  
路径：frameworks\base\core\java\android\content\pm\packageparser.java  
1、PackageParse#parsePackage(File, int) 方法解析  
如果是目录则调用 parseMonolithicPackage(File,int) 方法，如果不是目录，是文件则调用 parseClusterPackage(File,int)

2.PackageParse#parseMonolithicPackage(File, int) 方法解析

首先 判断是不是 mOnlyCoreApps，mOnlyCoreApps 该标示表明解析只考虑应用清单属性有效的应用，主要为了创建一个最小的启动环境，如果该标示为 true 则表示为轻量级解析，调用 parseMonolithicPackageLite 来进行解析

其次 如果 mOnlyCoreApps 不为空，则 new 了一个 AssetManager 对象

再次 调用 parseBaseApk() 方法解析一个 apk 并生成一个 Package 对象

最后 给 [pkg](https://so.csdn.net/so/search?q=pkg&spm=1001.2101.3001.7020) 的 codePath 赋值

这里面涉及了两个方法分别是 parseMonolithicPackageLite(apkFile, flags); 和 parseBaseApk()，

3.PackageParse#parseMonolithicPackageLite(File, int) 方法解析  
主要解决三个问题:  
parseApkLite(File,int) 函数内部的实现  
2、ApkLite 类  
3、PackageLite 类

parseMonolithicPackageLite 内部又调用了 parseApkLite 函数并且返回一个 ApkLite 对象，根据返回的 ApkLite 对象和包的绝对路径构造了一个 PackegeLite 对象作为返回值。

4.PackageParse#parseApkLite(File,int) 方法解析  
1、创建 AssetManager 对象 assets

2、assets 调用 addAssetPath(apkPath)

3、用 assets 作为入参来创建 Resources 对象 res

4、调用 assets 的 openXmlResourceParser() 来获取 XmlResourceParser 对象 parser 对象

5、设置签名

6、调用 parseApkLite(String,Resource,XmlPullParser,AttributeSet,int,Signature[]) 方法来返回一个 ApkLite 对象

```
PackageParse#parseApkLite(File,int)方法解析
这个方法又调用了parsePackageSplitNames()方法来获取Pair<String, String> packageSplit

```

遍历属性，并获取相应的值，这里面主要是循环解析出 installLocation，versionCode，revisionCode，coreApp

继续解析 package-verifier 节点，该节点主要解析两个属性

6.parseActivity(Package owner, Resources res,  
XmlResourceParser parser, int flags, String[] outError, CachedComponentArgs cachedArgs,  
boolean receiver, boolean hardwareAccelerated)  
负责解析每个启动的 Activity

所以添加 apk 是屏幕方向就在这个方法里面进行

源码如下:

```
private Activity parseActivity(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError, CachedComponentArgs cachedArgs,
            boolean receiver, boolean hardwareAccelerated)
            throws XmlPullParserException, IOException {
        TypedArray sa = res.obtainAttributes(parser, R.styleable.AndroidManifestActivity);

        if (cachedArgs.mActivityArgs == null) {
            cachedArgs.mActivityArgs = new ParseComponentArgs(owner, outError,
                    R.styleable.AndroidManifestActivity_name,
                    R.styleable.AndroidManifestActivity_label,
                    R.styleable.AndroidManifestActivity_icon,
                    R.styleable.AndroidManifestActivity_roundIcon,
                    R.styleable.AndroidManifestActivity_logo,
                    R.styleable.AndroidManifestActivity_banner,
                    mSeparateProcesses,
                    R.styleable.AndroidManifestActivity_process,
                    R.styleable.AndroidManifestActivity_description,
                    R.styleable.AndroidManifestActivity_enabled);
        }

        cachedArgs.mActivityArgs.tag = receiver ? "<receiver>" : "<activity>";
        cachedArgs.mActivityArgs.sa = sa;
        cachedArgs.mActivityArgs.flags = flags;

        Activity a = new Activity(cachedArgs.mActivityArgs, new ActivityInfo());
        if (outError[0] != null) {
            sa.recycle();
            return null;
        }
        Log.e(TAG, "Activity----" + a.info.name + " specified invalid parentActivityName");
        boolean setExported = sa.hasValue(R.styleable.AndroidManifestActivity_exported);
        if (setExported) {
            a.info.exported = sa.getBoolean(R.styleable.AndroidManifestActivity_exported, false);
        }

        a.info.theme = sa.getResourceId(R.styleable.AndroidManifestActivity_theme, 0);

        a.info.uiOptions = sa.getInt(R.styleable.AndroidManifestActivity_uiOptions,
                a.info.applicationInfo.uiOptions);

        String parentName = sa.getNonConfigurationString(
                R.styleable.AndroidManifestActivity_parentActivityName,
                Configuration.NATIVE_CONFIG_VERSION);
        if (parentName != null) {
            String parentClassName = buildClassName(a.info.packageName, parentName, outError);
            if (outError[0] == null) {
                a.info.parentActivityName = parentClassName;
            } else {
                Log.e(TAG, "Activity " + a.info.name + " specified invalid parentActivityName " +
                        parentName);
                outError[0] = null;
            }
        }

        String str;
        str = sa.getNonConfigurationString(R.styleable.AndroidManifestActivity_permission, 0);
        if (str == null) {
            a.info.permission = owner.applicationInfo.permission;
        } else {
            a.info.permission = str.length() > 0 ? str.toString().intern() : null;
        }

        str = sa.getNonConfigurationString(
                R.styleable.AndroidManifestActivity_taskAffinity,
                Configuration.NATIVE_CONFIG_VERSION);
        a.info.taskAffinity = buildTaskAffinityName(owner.applicationInfo.packageName,
                owner.applicationInfo.taskAffinity, str, outError);

        a.info.splitName =
                sa.getNonConfigurationString(R.styleable.AndroidManifestActivity_splitName, 0);

        a.info.flags = 0;
        if (sa.getBoolean(
                R.styleable.AndroidManifestActivity_multiprocess, false)) {
            a.info.flags |= ActivityInfo.FLAG_MULTIPROCESS;
        }

        if (sa.getBoolean(R.styleable.AndroidManifestActivity_finishOnTaskLaunch, false)) {
            a.info.flags |= ActivityInfo.FLAG_FINISH_ON_TASK_LAUNCH;
        }

        if (sa.getBoolean(R.styleable.AndroidManifestActivity_clearTaskOnLaunch, false)) {
            a.info.flags |= ActivityInfo.FLAG_CLEAR_TASK_ON_LAUNCH;
        }

        if (sa.getBoolean(R.styleable.AndroidManifestActivity_noHistory, false)) {
            a.info.flags |= ActivityInfo.FLAG_NO_HISTORY;
        }

        if (sa.getBoolean(R.styleable.AndroidManifestActivity_alwaysRetainTaskState, false)) {
            a.info.flags |= ActivityInfo.FLAG_ALWAYS_RETAIN_TASK_STATE;
        }

        if (sa.getBoolean(R.styleable.AndroidManifestActivity_stateNotNeeded, false)) {
            a.info.flags |= ActivityInfo.FLAG_STATE_NOT_NEEDED;
        }

        if (sa.getBoolean(R.styleable.AndroidManifestActivity_excludeFromRecents, false)) {
            a.info.flags |= ActivityInfo.FLAG_EXCLUDE_FROM_RECENTS;
        }

        if (sa.getBoolean(R.styleable.AndroidManifestActivity_allowTaskReparenting,
                (owner.applicationInfo.flags&ApplicationInfo.FLAG_ALLOW_TASK_REPARENTING) != 0)) {
            a.info.flags |= ActivityInfo.FLAG_ALLOW_TASK_REPARENTING;
        }

        if (sa.getBoolean(R.styleable.AndroidManifestActivity_finishOnCloseSystemDialogs, false)) {
            a.info.flags |= ActivityInfo.FLAG_FINISH_ON_CLOSE_SYSTEM_DIALOGS;
        }

        if (sa.getBoolean(R.styleable.AndroidManifestActivity_showOnLockScreen, false)
                || sa.getBoolean(R.styleable.AndroidManifestActivity_showForAllUsers, false)) {
            a.info.flags |= ActivityInfo.FLAG_SHOW_FOR_ALL_USERS;
        }

        if (sa.getBoolean(R.styleable.AndroidManifestActivity_immersive, false)) {
            a.info.flags |= ActivityInfo.FLAG_IMMERSIVE;
        }

        if (sa.getBoolean(R.styleable.AndroidManifestActivity_systemUserOnly, false)) {
            a.info.flags |= ActivityInfo.FLAG_SYSTEM_USER_ONLY;
        }

        if (!receiver) {
            if (sa.getBoolean(R.styleable.AndroidManifestActivity_hardwareAccelerated,
                    hardwareAccelerated)) {
                a.info.flags |= ActivityInfo.FLAG_HARDWARE_ACCELERATED;
            }

            a.info.launchMode = sa.getInt(
                    R.styleable.AndroidManifestActivity_launchMode, ActivityInfo.LAUNCH_MULTIPLE);
            a.info.documentLaunchMode = sa.getInt(
                    R.styleable.AndroidManifestActivity_documentLaunchMode,
                    ActivityInfo.DOCUMENT_LAUNCH_NONE);
            a.info.maxRecents = sa.getInt(
                    R.styleable.AndroidManifestActivity_maxRecents,
                    ActivityTaskManager.getDefaultAppRecentsLimitStatic());
            a.info.configChanges = getActivityConfigChanges(
                    sa.getInt(R.styleable.AndroidManifestActivity_configChanges, 0),
                    sa.getInt(R.styleable.AndroidManifestActivity_recreateOnConfigChanges, 0));
            a.info.softInputMode = sa.getInt(
                    R.styleable.AndroidManifestActivity_windowSoftInputMode, 0);

            a.info.persistableMode = sa.getInteger(
                    R.styleable.AndroidManifestActivity_persistableMode,
                    ActivityInfo.PERSIST_ROOT_ONLY);

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_allowEmbedded, false)) {
                a.info.flags |= ActivityInfo.FLAG_ALLOW_EMBEDDED;
            }

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_autoRemoveFromRecents, false)) {
                a.info.flags |= ActivityInfo.FLAG_AUTO_REMOVE_FROM_RECENTS;
            }

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_relinquishTaskIdentity, false)) {
                a.info.flags |= ActivityInfo.FLAG_RELINQUISH_TASK_IDENTITY;
            }

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_resumeWhilePausing, false)) {
                a.info.flags |= ActivityInfo.FLAG_RESUME_WHILE_PAUSING;
            }

        // 修改代码部分 start 
           if("com.ss.android.ugc.aweme".equals(a.info.packageName)||"com.tencent.mm".equals(a.info.packageName)||"com.sprd.sprdnote".equals(a.info.packageName)) {
               a.info.screenOrientation = ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE;
           }else{
               a.info.screenOrientation = sa.getInt(
                    R.styleable.AndroidManifestActivity_screenOrientation,
                    SCREEN_ORIENTATION_UNSPECIFIED);
		   }	
        // 修改代码部分结束		   
            setActivityResizeMode(a.info, sa, owner);

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_supportsPictureInPicture,
                    false)) {
                a.info.flags |= FLAG_SUPPORTS_PICTURE_IN_PICTURE;
            }

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_alwaysFocusable, false)) {
                a.info.flags |= FLAG_ALWAYS_FOCUSABLE;
            }

            if (sa.hasValue(R.styleable.AndroidManifestActivity_maxAspectRatio)
                    && sa.getType(R.styleable.AndroidManifestActivity_maxAspectRatio)
                    == TypedValue.TYPE_FLOAT) {
                a.setMaxAspectRatio(sa.getFloat(R.styleable.AndroidManifestActivity_maxAspectRatio,
                        0 /*default*/));
            }

            if (sa.hasValue(R.styleable.AndroidManifestActivity_minAspectRatio)
                    && sa.getType(R.styleable.AndroidManifestActivity_minAspectRatio)
                    == TypedValue.TYPE_FLOAT) {
                a.setMinAspectRatio(sa.getFloat(R.styleable.AndroidManifestActivity_minAspectRatio,
                        0 /*default*/));
            }

            a.info.lockTaskLaunchMode =
                    sa.getInt(R.styleable.AndroidManifestActivity_lockTaskMode, 0);

            a.info.encryptionAware = a.info.directBootAware = sa.getBoolean(
                    R.styleable.AndroidManifestActivity_directBootAware,
                    false);

            a.info.requestedVrComponent =
                sa.getString(R.styleable.AndroidManifestActivity_enableVrMode);

            a.info.rotationAnimation =
                sa.getInt(R.styleable.AndroidManifestActivity_rotationAnimation, ROTATION_ANIMATION_UNSPECIFIED);

            a.info.colorMode = sa.getInt(R.styleable.AndroidManifestActivity_colorMode,
                    ActivityInfo.COLOR_MODE_DEFAULT);

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_showWhenLocked, false)) {
                a.info.flags |= ActivityInfo.FLAG_SHOW_WHEN_LOCKED;
            }

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_turnScreenOn, false)) {
                a.info.flags |= ActivityInfo.FLAG_TURN_SCREEN_ON;
            }

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_inheritShowWhenLocked, false)) {
                a.info.privateFlags |= ActivityInfo.FLAG_INHERIT_SHOW_WHEN_LOCKED;
            }
        } else {
            a.info.launchMode = ActivityInfo.LAUNCH_MULTIPLE;
            a.info.configChanges = 0;

            if (sa.getBoolean(R.styleable.AndroidManifestActivity_singleUser, false)) {
                a.info.flags |= ActivityInfo.FLAG_SINGLE_USER;
            }

            a.info.encryptionAware = a.info.directBootAware = sa.getBoolean(
                    R.styleable.AndroidManifestActivity_directBootAware,
                    false);
        }

        if (a.info.directBootAware) {
            owner.applicationInfo.privateFlags |=
                    ApplicationInfo.PRIVATE_FLAG_PARTIALLY_DIRECT_BOOT_AWARE;
        }

        // can't make this final; we may set it later via meta-data
        boolean visibleToEphemeral =
                sa.getBoolean(R.styleable.AndroidManifestActivity_visibleToInstantApps, false);
        if (visibleToEphemeral) {
            a.info.flags |= ActivityInfo.FLAG_VISIBLE_TO_INSTANT_APP;
            owner.visibleToInstantApps = true;
        }

        sa.recycle();

        if (receiver && (owner.applicationInfo.privateFlags
                &ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0) {
            // A heavy-weight application can not have receives in its main process
            // We can do direct compare because we intern all strings.
            if (a.info.processName == owner.packageName) {
                outError[0] = "Heavy-weight applications can not have receivers in main process";
            }
        }

        if (outError[0] != null) {
            return null;
        }

        int outerDepth = parser.getDepth();
        int type;
        while ((type=parser.next()) != XmlPullParser.END_DOCUMENT
               && (type != XmlPullParser.END_TAG
                       || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            if (parser.getName().equals("intent-filter")) {
                ActivityIntentInfo intent = new ActivityIntentInfo(a);
                if (!parseIntent(res, parser, true /*allowGlobs*/, true /*allowAutoVerify*/,
                        intent, outError)) {
                    return null;
                }
                if (intent.countActions() == 0) {
                    Slog.w(TAG, "No actions in intent filter at "
                            + mArchiveSourcePath + " "
                            + parser.getPositionDescription());
                } else {
                    a.order = Math.max(intent.getOrder(), a.order);
                    a.intents.add(intent);
                }
                // adjust activity flags when we implicitly expose it via a browsable filter
                final int visibility = visibleToEphemeral
                        ? IntentFilter.VISIBILITY_EXPLICIT
                        : !receiver && isImplicitlyExposedIntent(intent)
                                ? IntentFilter.VISIBILITY_IMPLICIT
                                : IntentFilter.VISIBILITY_NONE;
                intent.setVisibilityToInstantApp(visibility);
                if (intent.isVisibleToInstantApp()) {
                    a.info.flags |= ActivityInfo.FLAG_VISIBLE_TO_INSTANT_APP;
                }
                if (intent.isImplicitlyVisibleToInstantApp()) {
                    a.info.flags |= ActivityInfo.FLAG_IMPLICITLY_VISIBLE_TO_INSTANT_APP;
                }
                if (LOG_UNSAFE_BROADCASTS && receiver
                        && (owner.applicationInfo.targetSdkVersion >= Build.VERSION_CODES.O)) {
                    for (int i = 0; i < intent.countActions(); i++) {
                        final String action = intent.getAction(i);
                        if (action == null || !action.startsWith("android.")) continue;
                        if (!SAFE_BROADCASTS.contains(action)) {
                            Slog.w(TAG, "Broadcast " + action + " may never be delivered to "
                                    + owner.packageName + " as requested at: "
                                    + parser.getPositionDescription());
                        }
                    }
                }
            } else if (!receiver && parser.getName().equals("preferred")) {
                ActivityIntentInfo intent = new ActivityIntentInfo(a);
                if (!parseIntent(res, parser, false /*allowGlobs*/, false /*allowAutoVerify*/,
                        intent, outError)) {
                    return null;
                }
                if (intent.countActions() == 0) {
                    Slog.w(TAG, "No actions in preferred at "
                            + mArchiveSourcePath + " "
                            + parser.getPositionDescription());
                } else {
                    if (owner.preferredActivityFilters == null) {
                        owner.preferredActivityFilters = new ArrayList<ActivityIntentInfo>();
                    }
                    owner.preferredActivityFilters.add(intent);
                }
                // adjust activity flags when we implicitly expose it via a browsable filter
                final int visibility = visibleToEphemeral
                        ? IntentFilter.VISIBILITY_EXPLICIT
                        : !receiver && isImplicitlyExposedIntent(intent)
                                ? IntentFilter.VISIBILITY_IMPLICIT
                                : IntentFilter.VISIBILITY_NONE;
                intent.setVisibilityToInstantApp(visibility);
                if (intent.isVisibleToInstantApp()) {
                    a.info.flags |= ActivityInfo.FLAG_VISIBLE_TO_INSTANT_APP;
                }
                if (intent.isImplicitlyVisibleToInstantApp()) {
                    a.info.flags |= ActivityInfo.FLAG_IMPLICITLY_VISIBLE_TO_INSTANT_APP;
                }
            } else if (parser.getName().equals("meta-data")) {
                if ((a.metaData = parseMetaData(res, parser, a.metaData,
                        outError)) == null) {
                    return null;
                }
            } else if (!receiver && parser.getName().equals("layout")) {
                parseLayout(res, parser, a);
            } else {
                if (!RIGID_PARSER) {
                    Slog.w(TAG, "Problem in package " + mArchiveSourcePath + ":");
                    if (receiver) {
                        Slog.w(TAG, "Unknown element under <receiver>: " + parser.getName()
                                + " at " + mArchiveSourcePath + " "
                                + parser.getPositionDescription());
                    } else {
                        Slog.w(TAG, "Unknown element under <activity>: " + parser.getName()
                                + " at " + mArchiveSourcePath + " "
                                + parser.getPositionDescription());
                    }
                    XmlUtils.skipCurrentTag(parser);
                    continue;
                } else {
                    if (receiver) {
                        outError[0] = "Bad element under <receiver>: " + parser.getName();
                    } else {
                        outError[0] = "Bad element under <activity>: " + parser.getName();
                    }
                    return null;
                }
            }
        }

        if (!setExported) {
            a.info.exported = a.intents.size() > 0;
        }

        return a;
    }

```

在代码中可以看出 setActivityResizeMode 根据屏幕显示方向来设置屏幕分辨率  
所以就在这段代码添加设置 app 的显示方向  
修改部分如下:

```
 a.info.screenOrientation = sa.getInt(
                    R.styleable.AndroidManifestActivity_screenOrientation,
                    SCREEN_ORIENTATION_UNSPECIFIED);

```

修改为:

```
if("com.ss.android.ugc.aweme".equals(a.info.packageName)||"com.tencent.mm".equals(a.info.packageName)||"com.sprd.sprdnote".equals(a.info.packageName)) {
               a.info.screenOrientation = ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE;
           }else{
               a.info.screenOrientation = sa.getInt(
                    R.styleable.AndroidManifestActivity_screenOrientation,
                    SCREEN_ORIENTATION_UNSPECIFIED);
		   }

```
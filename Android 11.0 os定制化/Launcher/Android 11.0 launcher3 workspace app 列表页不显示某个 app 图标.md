> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/125417426)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2. 核心代码](#t1)

[3. 核心代码功能分析](#t2)

 [3.1 LoadTask.java 中代码分析](#t3)

[3.2 安装卸载 app 更新 app 列表时 PackageUpdatedTask.java 代码分析](#t4)

1. 概述
-----

在 11.0 的开发中，Launcher3 workspace 的 app 列表页 会负责加载系统中 app 的所有图标 但针对某个不需要显示在桌面的 app 图标需要过滤掉 所以需要在加载和更新的时候过滤 需要更改两处地方，一处是 加在列表时 一处是安装卸载 app 更新 app 列表时

2. 核心代码
-------

```
主要代码:
packages\apps\Launcher3\src\com\android\launcher3\model\LoadTask.java
packages\apps\Launcher3\src\com\android\launcher3\model\PackageUpdatedTask.java
```

3. 核心代码功能分析
-----------

####   3.1 LoadTask.java 中代码分析

通过 run() 加载 Launcher3 的相关数据

```
public void run() {
          synchronized (this) {
              // Skip fast if we are already stopped.
              if (mStopped) {
                  return;
              }
          }
  
          Object traceToken = TraceHelper.INSTANCE.beginSection(TAG);
          TimingLogger logger = new TimingLogger(TAG, "run");
          try (LauncherModel.LoaderTransaction transaction = mApp.getModel().beginLoader(this)) {
              List<ShortcutInfo> allShortcuts = new ArrayList<>();
              loadWorkspace(allShortcuts);
              logASplit(logger, "loadWorkspace");
  
              // Sanitize data re-syncs widgets/shortcuts based on the workspace loaded from db.
              // sanitizeData should not be invoked if the workspace is loaded from a db different
              // from the main db as defined in the invariant device profile.
              // (e.g. both grid preview and minimal device mode uses a different db)
              if (mApp.getInvariantDeviceProfile().dbFile.equals(mDbName)) {
                  verifyNotStopped();
                  sanitizeData();
                  logASplit(logger, "sanitizeData");
              }
  
              verifyNotStopped();
              mResults.bindWorkspace();
              logASplit(logger, "bindWorkspace");
  
              mModelDelegate.workspaceLoadComplete();
              // Notify the installer packages of packages with active installs on the first screen.
              sendFirstScreenActiveInstallsBroadcast();
              logASplit(logger, "sendFirstScreenActiveInstallsBroadcast");
  
              // Take a break
              waitForIdle();
              logASplit(logger, "step 1 complete");
              verifyNotStopped();
  
              // second step
              List<LauncherActivityInfo> allActivityList = loadAllApps();
              logASplit(logger, "loadAllApps");
  
              verifyNotStopped();
              mResults.bindAllApps();
              logASplit(logger, "bindAllApps");
  
              verifyNotStopped();
              IconCacheUpdateHandler updateHandler = mIconCache.getUpdateHandler();
              setIgnorePackages(updateHandler);
              updateHandler.updateIcons(allActivityList,
                      LauncherActivityCachingLogic.newInstance(mApp.getContext()),
                      mApp.getModel()::onPackageIconsUpdated);
              logASplit(logger, "update icon cache");
  
              if (FeatureFlags.ENABLE_DEEP_SHORTCUT_ICON_CACHE.get()) {
                  verifyNotStopped();
                  logASplit(logger, "save shortcuts in icon cache");
                  updateHandler.updateIcons(allShortcuts, new ShortcutCachingLogic(),
                          mApp.getModel()::onPackageIconsUpdated);
              }
  
              // Take a break
              waitForIdle();
              logASplit(logger, "step 2 complete");
              verifyNotStopped();
  
              // third step
              List<ShortcutInfo> allDeepShortcuts = loadDeepShortcuts();
              logASplit(logger, "loadDeepShortcuts");
  
              verifyNotStopped();
              mResults.bindDeepShortcuts();
              logASplit(logger, "bindDeepShortcuts");
  
              if (FeatureFlags.ENABLE_DEEP_SHORTCUT_ICON_CACHE.get()) {
                  verifyNotStopped();
                  logASplit(logger, "save deep shortcuts in icon cache");
                  updateHandler.updateIcons(allDeepShortcuts,
                          new ShortcutCachingLogic(), (pkgs, user) -> { });
              }
  
              // Take a break
              waitForIdle();
              logASplit(logger, "step 3 complete");
              verifyNotStopped();
  
              // fourth step
              List<ComponentWithLabelAndIcon> allWidgetsList =
                      mBgDataModel.widgetsModel.update(mApp, null);
              logASplit(logger, "load widgets");
  
              verifyNotStopped();
              mResults.bindWidgets();
              logASplit(logger, "bindWidgets");
              verifyNotStopped();
  
              updateHandler.updateIcons(allWidgetsList,
                      new ComponentWithIconCachingLogic(mApp.getContext(), true),
                      mApp.getModel()::onWidgetLabelsUpdated);
              logASplit(logger, "save widgets in icon cache");
  
              // fifth step
              if (FeatureFlags.FOLDER_NAME_SUGGEST.get()) {
                  loadFolderNames();
              }
  
              verifyNotStopped();
              updateHandler.finish();
              logASplit(logger, "finish icon update");
  
              mModelDelegate.modelLoadComplete();
              transaction.commit();
          } catch (CancellationException e) {
              // Loader stopped, ignore
              logASplit(logger, "Cancelled");
          } finally {
              logger.dumpToLog();
          }
          TraceHelper.INSTANCE.endSection(traceToken);
      }
```

在 loadAllApps() 中加载所有 app 显示到桌面，所以需要去掉不需要显示的 app

```
 
private List<LauncherActivityInfo> loadAllApps() {
        final List<UserHandle> profiles = mUserManager.getUserProfiles();
        List<LauncherActivityInfo> allActivityList = new ArrayList<>();
        // Clear the list of apps
        mBgAllAppsList.clear();
        for (UserHandle user : profiles) {
            // Query for the set of apps
            final List<LauncherActivityInfo> apps = mLauncherApps.getActivityList(null, user);
            // Fail if we don't have any apps
            // TODO: Fix this. Only fail for the current user.
            if (apps == null || apps.isEmpty()) {
                return allActivityList;
            }
            boolean quietMode = mUserManager.isQuietModeEnabled(user);
            // Create the ApplicationInfos
            for (int i = 0; i < apps.size(); i++) {
                LauncherActivityInfo app = apps.get(i);
				String apppackagename = app.getComponentName().getPackageName();
                // This builds the icon bitmaps.
                mBgAllAppsList.add(new AppInfo(app, user, quietMode), app);
				
            }
            allActivityList.addAll(apps);
        }
 
        if (FeatureFlags.LAUNCHER3_PROMISE_APPS_IN_ALL_APPS) {
            // get all active sessions and add them to the all apps list
            for (PackageInstaller.SessionInfo info :
                    mPackageInstaller.getAllVerifiedSessions()) {
                mBgAllAppsList.addPromiseApp(mApp.getContext(),
                        PackageInstallerCompat.PackageInstallInfo.fromInstallingState(info));
            }
        }
 
        mBgAllAppsList.getAndResetChangeFlag();
        return allActivityList;
    }
 
 
 
 
 
```

在这段代码可以看出 loadAllApps() 中加载所有 apps 然后 mBgAllAppsList.add(new AppInfo(app, user, quietMode), app); 添加到 apps 列表中

所以具体修改为:

```
private List<LauncherActivityInfo> loadAllApps() {
        final List<UserHandle> profiles = mUserManager.getUserProfiles();
        List<LauncherActivityInfo> allActivityList = new ArrayList<>();
        // Clear the list of apps
        mBgAllAppsList.clear();
        for (UserHandle user : profiles) {
            // Query for the set of apps
            final List<LauncherActivityInfo> apps = mLauncherApps.getActivityList(null, user);
            // Fail if we don't have any apps
            // TODO: Fix this. Only fail for the current user.
            if (apps == null || apps.isEmpty()) {
                return allActivityList;
            }
            boolean quietMode = mUserManager.isQuietModeEnabled(user);
            // Create the ApplicationInfos
            for (int i = 0; i < apps.size(); i++) {
                LauncherActivityInfo app = apps.get(i);
				String apppackagename = app.getComponentName().getPackageName();
                // This builds the icon bitmaps.
               
                //在此处添加需要去掉的app 根据包名去掉不需要显示的app
				if(!apppackagename.equals("com.example.zxwmcuupdata")){
                   mBgAllAppsList.add(new AppInfo(app, user, quietMode), app);
				}
 
            }
            allActivityList.addAll(apps);
        }
 
        if (FeatureFlags.LAUNCHER3_PROMISE_APPS_IN_ALL_APPS) {
            // get all active sessions and add them to the all apps list
            for (PackageInstaller.SessionInfo info :
                    mPackageInstaller.getAllVerifiedSessions()) {
                mBgAllAppsList.addPromiseApp(mApp.getContext(),
                        PackageInstallerCompat.PackageInstallInfo.fromInstallingState(info));
            }
        }
 
        mBgAllAppsList.getAndResetChangeFlag();
        return allActivityList;
    }
 
 
```

### 3.2 安装卸载 app 更新 app 列表时 PackageUpdatedTask.java 代码分析

路径: packages\apps\Launcher3\src\com\android\launcher3\model\PackageUpdatedTask.java

```
 
 
@Override
    public void execute(LauncherAppState app, BgDataModel dataModel, AllAppsList appsList) {
        final Context context = app.getContext();
        final IconCache iconCache = app.getIconCache();
 
        final String[] packages = mPackages;
        final int N = packages.length;
        FlagOp flagOp = FlagOp.NO_OP;
        final HashSet<String> packageSet = new HashSet<>(Arrays.asList(packages));
        ItemInfoMatcher matcher = ItemInfoMatcher.ofPackages(packageSet, mUser);
        final HashSet<ComponentName> removedComponents = new HashSet<>();
 
        switch (mOp) {
            case OP_ADD: {
                for (int i = 0; i < N; i++) {
                    if (DEBUG) Log.d(TAG, "mAllAppsList.addPackage " + packages[i]);
                    iconCache.updateIconsForPkg(packages[i], mUser);
                    if (FeatureFlags.LAUNCHER3_PROMISE_APPS_IN_ALL_APPS) {
                        appsList.removePackage(packages[i], mUser);
                    }
                    appsList.addPackage(context, packages[i], mUser);
 
                    // Automatically add homescreen icon for work profile apps for below O device.
                    if (!Utilities.ATLEAST_OREO && !Process.myUserHandle().equals(mUser)) {
                        SessionCommitReceiver.queueAppIconAddition(context, packages[i], mUser);
                    }
                }
                flagOp = FlagOp.removeFlag(WorkspaceItemInfo.FLAG_DISABLED_NOT_AVAILABLE);
                break;
            }
            case OP_UPDATE:
                try (SafeCloseable t =
                             appsList.trackRemoves(a -> removedComponents.add(a.componentName))) {
                    for (int i = 0; i < N; i++) {
                        if (DEBUG) Log.d(TAG, "mAllAppsList.updatePackage " + packages[i]);
                 // 此处去掉需要隐藏的app 此处是安装卸载app 更新app列表
						if(!packages[i].equals("com.example.zxwmcuupdata")){
							iconCache.updateIconsForPkg(packages[i], mUser);
							appsList.updatePackage(context, packages[i], mUser);
							app.getWidgetCache().removePackage(packages[i], mUser);
						}
                    }
                }
                // Since package was just updated, the target must be available now.
                flagOp = FlagOp.removeFlag(WorkspaceItemInfo.FLAG_DISABLED_NOT_AVAILABLE);
                break;
            case OP_REMOVE: {
                for (int i = 0; i < N; i++) {
                    FileLog.d(TAG, "Removing app icon" + packages[i]);
                    iconCache.removeIconsForPkg(packages[i], mUser);
                }
                // Fall through
            }
            case OP_UNAVAILABLE:
                for (int i = 0; i < N; i++) {
                    if (DEBUG) Log.d(TAG, "mAllAppsList.removePackage " + packages[i]);
                    appsList.removePackage(packages[i], mUser);
                    app.getWidgetCache().removePackage(packages[i], mUser);
                }
                flagOp = FlagOp.addFlag(WorkspaceItemInfo.FLAG_DISABLED_NOT_AVAILABLE);
                break;
            case OP_SUSPEND:
            case OP_UNSUSPEND:
                flagOp = mOp == OP_SUSPEND ?
                        FlagOp.addFlag(WorkspaceItemInfo.FLAG_DISABLED_SUSPENDED) :
                        FlagOp.removeFlag(WorkspaceItemInfo.FLAG_DISABLED_SUSPENDED);
                if (DEBUG) Log.d(TAG, "mAllAppsList.(un)suspend " + N);
                appsList.updateDisabledFlags(matcher, flagOp);
                break;
            case OP_USER_AVAILABILITY_CHANGE:
                flagOp = UserManagerCompat.getInstance(context).isQuietModeEnabled(mUser)
                        ? FlagOp.addFlag(WorkspaceItemInfo.FLAG_DISABLED_QUIET_USER)
                        : FlagOp.removeFlag(WorkspaceItemInfo.FLAG_DISABLED_QUIET_USER);
                // We want to update all packages for this user.
                matcher = ItemInfoMatcher.ofUser(mUser);
                appsList.updateDisabledFlags(matcher, flagOp);
                break;
        }
 
        bindApplicationsIfNeeded();
 
        final IntSparseArrayMap<Boolean> removedShortcuts = new IntSparseArrayMap<>();
 
        // Update shortcut infos
        if (mOp == OP_ADD || flagOp != FlagOp.NO_OP) {
            final ArrayList<WorkspaceItemInfo> updatedWorkspaceItems = new ArrayList<>();
            final ArrayList<LauncherAppWidgetInfo> widgets = new ArrayList<>();
 
            // For system apps, package manager send OP_UPDATE when an app is enabled.
            final boolean isNewApkAvailable = mOp == OP_ADD || mOp == OP_UPDATE;
            synchronized (dataModel) {
                for (ItemInfo info : dataModel.itemsIdMap) {
                    if (info instanceof WorkspaceItemInfo && mUser.equals(info.user)) {
                        WorkspaceItemInfo si = (WorkspaceItemInfo) info;
                        boolean infoUpdated = false;
                        boolean shortcutUpdated = false;
 
                        // Update shortcuts which use iconResource.
                        if ((si.iconResource != null)
                                && packageSet.contains(si.iconResource.packageName)) {
                            LauncherIcons li = LauncherIcons.obtain(context);
                            BitmapInfo iconInfo = li.createIconBitmap(si.iconResource);
                            li.recycle();
                            if (iconInfo != null) {
                                si.applyFrom(iconInfo);
                                infoUpdated = true;
                            }
                        }
 
                        ComponentName cn = si.getTargetComponent();
                        if (cn != null && matcher.matches(si, cn)) {
                            String packageName = cn.getPackageName();
 
                            if (si.hasStatusFlag(WorkspaceItemInfo.FLAG_SUPPORTS_WEB_UI)) {
                                removedShortcuts.put(si.id, false);
                                if (mOp == OP_REMOVE) {
                                    continue;
                                }
                            }
 
                            if (si.isPromise() && isNewApkAvailable) {
                                boolean isTargetValid = true;
                                if (si.itemType == Favorites.ITEM_TYPE_DEEP_SHORTCUT) {
                                    List<ShortcutInfo> shortcut = DeepShortcutManager
                                            .getInstance(context).queryForPinnedShortcuts(
                                                    cn.getPackageName(),
                                                    Arrays.asList(si.getDeepShortcutId()), mUser);
                                    if (shortcut.isEmpty()) {
                                        isTargetValid = false;
                                    } else {
                                        si.updateFromDeepShortcutInfo(shortcut.get(0), context);
                                        infoUpdated = true;
                                    }
                                } else if (!cn.getClassName().equals(IconCache.EMPTY_CLASS_NAME)) {
                                    isTargetValid = LauncherAppsCompat.getInstance(context)
                                            .isActivityEnabledForProfile(cn, mUser);
                                }
                                if (si.hasStatusFlag(FLAG_RESTORED_ICON | FLAG_AUTOINSTALL_ICON)) {
                                    if (updateWorkspaceItemIntent(context, si, packageName)) {
                                        infoUpdated = true;
                                    } else if (si.hasPromiseIconUi()) {
                                        removedShortcuts.put(si.id, true);
                                        continue;
                                    }
                                } else if (!isTargetValid) {
                                    removedShortcuts.put(si.id, true);
                                    FileLog.e(TAG, "Restored shortcut no longer valid "
                                            + si.intent);
                                    continue;
                                } else {
                                    si.status = WorkspaceItemInfo.DEFAULT;
                                    infoUpdated = true;
                                }
                            } else if (isNewApkAvailable && removedComponents.contains(cn)) {
                                if (updateWorkspaceItemIntent(context, si, packageName)) {
                                    infoUpdated = true;
                                }
                            }
 
                            if (isNewApkAvailable &&
                                    si.itemType == Favorites.ITEM_TYPE_APPLICATION) {
                                iconCache.getTitleAndIcon(si, si.usingLowResIcon());
                                infoUpdated = true;
                            }
 
                            int oldRuntimeFlags = si.runtimeStatusFlags;
                            si.runtimeStatusFlags = flagOp.apply(si.runtimeStatusFlags);
                            if (si.runtimeStatusFlags != oldRuntimeFlags) {
                                shortcutUpdated = true;
                            }
                        }
 
                        if (infoUpdated || shortcutUpdated) {
                            updatedWorkspaceItems.add(si);
                        }
                        if (infoUpdated) {
                            getModelWriter().updateItemInDatabase(si);
                        }
                    } else if (info instanceof LauncherAppWidgetInfo && isNewApkAvailable) {
                        LauncherAppWidgetInfo widgetInfo = (LauncherAppWidgetInfo) info;
                        if (mUser.equals(widgetInfo.user)
                                && widgetInfo.hasRestoreFlag(LauncherAppWidgetInfo.FLAG_PROVIDER_NOT_READY)
                                && packageSet.contains(widgetInfo.providerName.getPackageName())) {
                            widgetInfo.restoreStatus &=
                                    ~LauncherAppWidgetInfo.FLAG_PROVIDER_NOT_READY &
                                            ~LauncherAppWidgetInfo.FLAG_RESTORE_STARTED;
 
                            // adding this flag ensures that launcher shows 'click to setup'
                            // if the widget has a config activity. In case there is no config
                            // activity, it will be marked as 'restored' during bind.
                            widgetInfo.restoreStatus |= LauncherAppWidgetInfo.FLAG_UI_NOT_READY;
 
                            widgets.add(widgetInfo);
                            getModelWriter().updateItemInDatabase(widgetInfo);
                        }
                    }
                }
            }
 
            bindUpdatedWorkspaceItems(updatedWorkspaceItems);
            if (!removedShortcuts.isEmpty()) {
                deleteAndBindComponentsRemoved(ItemInfoMatcher.ofItemIds(removedShortcuts, false));
            }
 
            if (!widgets.isEmpty()) {
                scheduleCallbackTask(c -> c.bindWidgetsRestored(widgets));
            }
        }
 
。。。。
    }
 
```

case OP_UPDATE 的实现检测 app 是否更新这时候需要

负责更新的时候 去掉不显示 app 的更新就可以了
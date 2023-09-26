> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127855867)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.Launcher3 去掉抽屉模式 双层改成单层 (一) 的核心类](#2.Launcher3%E5%8E%BB%E6%8E%89%E6%8A%BD%E5%B1%89%E6%A8%A1%E5%BC%8F%20%E5%8F%8C%E5%B1%82%E6%94%B9%E6%88%90%E5%8D%95%E5%B1%82%28%E4%B8%80%29%E7%9A%84%E6%A0%B8%E5%BF%83%E7%B1%BB)

[3.Launcher3 去掉抽屉模式 双层改成单层 (一) 的核心功能分析和实现](#3.Launcher3%E5%8E%BB%E6%8E%89%E6%8A%BD%E5%B1%89%E6%A8%A1%E5%BC%8F%20%E5%8F%8C%E5%B1%82%E6%94%B9%E6%88%90%E5%8D%95%E5%B1%82%28%E4%B8%80%29%E7%9A%84%E6%A0%B8%E5%BF%83%E5%8A%9F%E8%83%BD%E5%88%86%E6%9E%90%E5%92%8C%E5%AE%9E%E7%8E%B0)

 [3.1 关于在 FeatureFlags.java 中设置常量来负责管理单双层](#t3)

[3.2 在 LoaderTask.java 中设置单层加载所有 app 到桌面](#t4)

[3.3 InstallShortcutReceiver.java 中的修改](#t5)

[3.4 AddWorkspaceItemsTask.java 关于绑定 apps 到 workspace 的更改](#t6)

1. 概述
-----

  在系统产品开发中，对于 Launcher3 系统原生是默认双层就是带抽屉的，产品开发需要需求要求  
改成单层的，所以就需要了解 Launcher3 关于 apps 绑定流程就可以了修改成单层即可

2.Launcher3 去掉抽屉模式 双层改成单层 (一) 的核心类
----------------------------------

```
packages/apps/Launcher3/src/com/android/launcher3/config/FeatureFlags.java
packages/apps/Launcher3/src/com/android/launcher3/model/LoaderTask.java
packages/apps/Launcher3/src/com/android/launcher3/model/AddWorkspaceItemsTask.java
packages/apps/Launcher3/src/com/android/launcher3/InstallShortcutReceiver.java
```

3.Launcher3 去掉抽屉模式 双层改成单层 (一) 的核心功能分析和实现
----------------------------------------

 3.1 关于在 FeatureFlags.java 中设置常量来负责管理单双层
----------------------------------------

```
 public final class FeatureFlags {
 
    private static final List<DebugFlag> sDebugFlags = new ArrayList<>();
 
    public static final String FLAGS_PREF_NAME = "featureFlags";
    public static final boolean REMOVE_DRAWER = true;
    private FeatureFlags() { }
 
    public static boolean showFlagTogglerUi(Context context) {
        return Utilities.IS_DEBUG_DEVICE && Utilities.isDevelopersOptionsEnabled(context);
    }
 
    /**
     * True when the build has come from Android Studio and is being used for local debugging.
     */
    public static final boolean IS_STUDIO_BUILD = BuildConfig.DEBUG;
 
    /**
     * Enable moving the QSB on the 0th screen of the workspace. This is not a configuration feature
     * and should be modified at a project level.
     */
    public static final boolean QSB_ON_FIRST_SCREEN = true;
```

在 FeatureFlags 中通过添加 REMOVE_DRAWER 来设置单双层切换 为 true 表示为单层，为 false  
表示为双层

3.2 在 LoaderTask.java 中设置单层加载所有 app 到桌面
---------------------------------------

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
            loadCachedPredictions();
            logger.addSplit("loadWorkspace");
 
            verifyNotStopped();
            mResults.bindWorkspace();
            logger.addSplit("bindWorkspace");
 
            // Notify the installer packages of packages with active installs on the first screen.
            sendFirstScreenActiveInstallsBroadcast();
            logger.addSplit("sendFirstScreenActiveInstallsBroadcast");
 
            // Take a break
            waitForIdle();
            logger.addSplit("step 1 complete");
            verifyNotStopped();
 
            // second step
            List<LauncherActivityInfo> allActivityList = loadAllApps();
            logger.addSplit("loadAllApps");
 
            //add core start
            if (FeatureFlags.REMOVE_DRAWER) {
                binderApplications() ;
            }
            //add core end
 
            verifyNotStopped();
            mResults.bindAllApps();
            logger.addSplit("bindAllApps");
 
            verifyNotStopped();
            IconCacheUpdateHandler updateHandler = mIconCache.getUpdateHandler();
            setIgnorePackages(updateHandler);
            updateHandler.updateIcons(allActivityList,
                    LauncherActivityCachingLogic.newInstance(mApp.getContext()),
                    mApp.getModel()::onPackageIconsUpdated);
            logger.addSplit("update icon cache");
 
            if (FeatureFlags.ENABLE_DEEP_SHORTCUT_ICON_CACHE.get()) {
                verifyNotStopped();
                logger.addSplit("save shortcuts in icon cache");
                updateHandler.updateIcons(allShortcuts, new ShortcutCachingLogic(),
                        mApp.getModel()::onPackageIconsUpdated);
            }
 
            // Take a break
            waitForIdle();
            logger.addSplit("step 2 complete");
            verifyNotStopped();
 
            // third step
            List<ShortcutInfo> allDeepShortcuts = loadDeepShortcuts();
            logger.addSplit("loadDeepShortcuts");
 
            verifyNotStopped();
            mResults.bindDeepShortcuts();
            logger.addSplit("bindDeepShortcuts");
 
            if (FeatureFlags.ENABLE_DEEP_SHORTCUT_ICON_CACHE.get()) {
                verifyNotStopped();
                logger.addSplit("save deep shortcuts in icon cache");
                updateHandler.updateIcons(allDeepShortcuts,
                        new ShortcutCachingLogic(), (pkgs, user) -> { });
            }
 
            // Take a break
            waitForIdle();
            logger.addSplit("step 3 complete");
            verifyNotStopped();
 
            // fourth step
            List<ComponentWithLabelAndIcon> allWidgetsList =
                    mBgDataModel.widgetsModel.update(mApp, null);
            logger.addSplit("load widgets");
 
            verifyNotStopped();
            mResults.bindWidgets();
            logger.addSplit("bindWidgets");
            verifyNotStopped();
 
            updateHandler.updateIcons(allWidgetsList,
                    new ComponentWithIconCachingLogic(mApp.getContext(), true),
                    mApp.getModel()::onWidgetLabelsUpdated);
            logger.addSplit("save widgets in icon cache");
 
            // fifth step
            if (FeatureFlags.FOLDER_NAME_SUGGEST.get()) {
                loadFolderNames();
            }
 
            verifyNotStopped();
            updateHandler.finish();
            logger.addSplit("finish icon update");
 
            transaction.commit();
        } catch (CancellationException e) {
            // Loader stopped, ignore
            logger.addSplit("Cancelled");
        } finally {
            logger.dumpToLog();
        }
        TraceHelper.INSTANCE.endSection(traceToken);
    }
	private void binderApplications() {
		final Context context = mApp.getContext();
		ArrayList<Pair<ItemInfo, Object>> installqueue_list = new ArrayList<>();
		final List<UserHandle> profiles = mUserManager.getUserProfiles();
		for (UserHandle user : profiles) {
			final List<LauncherActivityInfo> apps = mLauncherApps.getActivityList(null, user);
			ArrayList<InstallShortcutReceiver.PendingInstallShortcutInfo> added = new ArrayList<InstallShortcutReceiver.PendingInstallShortcutInfo>();
			synchronized (this) {
				for (LauncherActivityInfo appinfo : apps) {
					InstallShortcutReceiver.PendingInstallShortcutInfo pendingInstallShortcutInfo = new InstallShortcutReceiver.PendingInstallShortcutInfo(appinfo, context);
					added.add(pendingInstallShortcutInfo);
					installqueue_list .add(pendingInstallShortcutInfo.getItemInfo());
				}
			}
 
			if (!added.isEmpty()) {
				LauncherAppState.getInstance(context).getModel().addAndBindAddedWorkspaceItems(installqueue_list );
			}
		}
	}
```

在 LoaderTask.java 中的 run（）方法中，在 loadAllApps(); 后添加 binderApplications() 来添加  
所有 apps 绑定到 workspaces 中的 app 列表页，查询完 apps 后通过调用 LauncherAppState.getInstance(context).getModel().addAndBindAddedWorkspaceItems(installqueue_list ); 完成 apps 和 workspace 列表页的绑定

3.3 InstallShortcutReceiver.java 中的修改
-------------------------------------

  在 binderApplications() 中调用的 InstallShortcutReceiver.PendingInstallShortcutInfo  
是私有的成员类，所以需要改成共有的

```
--- a/packages/apps/Launcher3/src/com/android/launcher3/InstallShortcutReceiver.java
+++ b/packages/apps/Launcher3/src/com/android/launcher3/InstallShortcutReceiver.java
@@ -315,7 +315,7 @@ public class InstallShortcutReceiver extends BroadcastReceiver {
         return name;
     }
 
-    private static class PendingInstallShortcutInfo {
+    public static class PendingInstallShortcutInfo {
 
         final boolean isActivity;
         @Nullable final ShortcutInfo shortcutInfo;
```

3.4 AddWorkspaceItemsTask.java 关于绑定 apps 到 workspace 的更改
--------------------------------------------------------

   在调用 LauncherAppState.getInstance(context).getModel().addAndBindAddedWorkspaceItems(installqueue_list );  
这个绑定 item 时是通过调用 LauncherModel 的 enqueueModelUpdateTask(new AddWorkspaceItemsTask(itemList));  
最终是在 AddWorkspaceItemsTask.java 中实现绑定 item

```
@Override
    public void execute(LauncherAppState app, BgDataModel dataModel, AllAppsList apps) {
        if (mItemList.isEmpty()) {
            return;
        }
 
        final ArrayList<ItemInfo> addedItemsFinal = new ArrayList<>();
        final IntArray addedWorkspaceScreensFinal = new IntArray();
 
        synchronized(dataModel) {
            IntArray workspaceScreens = dataModel.collectWorkspaceScreens();
 
            List<ItemInfo> filteredItems = new ArrayList<>();
            for (Pair<ItemInfo, Object> entry : mItemList) {
                ItemInfo item = entry.first;
                if (item.itemType == LauncherSettings.Favorites.ITEM_TYPE_APPLICATION ||
                        item.itemType == LauncherSettings.Favorites.ITEM_TYPE_SHORTCUT) {
                    // Short-circuit this logic if the icon exists somewhere on the workspace
                    if (shortcutExists(dataModel, item.getIntent(), item.user)) {
                        continue;
                    }
 
                    if(!FeatureFlags.REMOVE_DRAWER){
                        // b/139663018 Short-circuit this logic if the icon is a system app
                        if (PackageManagerHelper.isSystemApp(app.getContext(), item.getIntent())) {
                           continue;
                        }
					}
                }
 
                if (item.itemType == LauncherSettings.Favorites.ITEM_TYPE_APPLICATION) {
                    if (item instanceof AppInfo) {
                        item = ((AppInfo) item).makeWorkspaceItem();
                    }
                }
                if (item != null) {
                    filteredItems.add(item);
                }
            }
 
            InstallSessionHelper packageInstaller =
                    InstallSessionHelper.INSTANCE.get(app.getContext());
            LauncherApps launcherApps = app.getContext().getSystemService(LauncherApps.class);
 
            for (ItemInfo item : filteredItems) {
                // Find appropriate space for the item.
                int[] coords = findSpaceForItem(app, dataModel, workspaceScreens,
                        addedWorkspaceScreensFinal, item.spanX, item.spanY);
                int screenId = coords[0];
 
                ItemInfo itemInfo;
                if (item instanceof WorkspaceItemInfo || item instanceof FolderInfo ||
                        item instanceof LauncherAppWidgetInfo) {
                    itemInfo = item;
                } else if (item instanceof AppInfo) {
                    itemInfo = ((AppInfo) item).makeWorkspaceItem();
                } else {
                    throw new RuntimeException("Unexpected info type");
                }
....
}
```

在 AddWorkspaceItemsTask.java 的 execute 中在绑定 item 时，会通过 PackageManagerHelper.isSystemApp  
来判断系统 app 时会忽略掉所以需要修改为:

```
                   if(!FeatureFlags.REMOVE_DRAWER){
                        // b/139663018 Short-circuit this logic if the icon is a system app
                        if (PackageManagerHelper.isSystemApp(app.getContext(), item.getIntent())) {
                           continue;
                        }
	   }
```
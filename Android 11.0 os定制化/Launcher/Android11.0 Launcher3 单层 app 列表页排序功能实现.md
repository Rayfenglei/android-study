> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/132460892)

1. 概述
-----

在 11.0 的定制化开发中，对于 Launcher3 的功能定制也是好多的，而对于单层 app 列表页来说排序功能的开发，也是常有的功能这就需要了解加载 app 数据的流程，然后根据需要进行排序就可以了，

如图:

![](https://img-blog.csdnimg.cn/a2646ca96ecc4b94b3e9fdfb707ca93e.png)

 2. Launcher3 单层 app 列表页排序功能实现
------------------------------

```
          packages\apps\Launcher3\src\com\android\launcher3\Launcher.java
           packages\apps\Launcher3\src\com\android\launcher3\LauncherModel.java
           packages\apps\Launcher3\src\com\android\launcher3\LoaderTask.java
           packages\apps\Launcher3\src\com\android\launcher3\LauncherProvider.java
```

3. Launcher3 单层 app 列表页排序功能实现
-----------------------------

在系统原生的 Launcher3 中，launcher3 为默认 home 程序，作为系统第一个 app(由 ActivityManagerService 的 systemReady 函数通过 Intent（intent.addCategory(Intent.CATEGORY_HOME);  
这里注册为 Intent.CATEGORY_HOME 的 Activity）方式打开 home 程序的，所以说 Launcher.java 就是第一个启动的页面，从这里来加载桌面显示数据，  
LauncherModel 是 Launcher3 处理数据的核心类，LauncherModel 本身继承自 BroadcastReceiver，实现了 OnAppsChangedCallbackCompat 接口，该接口在 LauncherAppsCompat.java 中定义，  
由 LauncherModel 中实现，并在 LauncherAppsCompat 子类的[内部类](https://so.csdn.net/so/search?q=%E5%86%85%E9%83%A8%E7%B1%BB&spm=1001.2101.3001.7020) PackageMonitor（API16 以上）（继承 BroadcastReceiver 类型）或 WrapperCallback（API15 以下）调用的  
而 LauncherProvider.java 主要就是就是桌面[启动器](https://so.csdn.net/so/search?q=%E5%90%AF%E5%8A%A8%E5%99%A8&spm=1001.2101.3001.7020) Launcher3 的核心 ContentProvider, 主要作用就是保存桌面应用程序的详细信息的，接下来首先分析下 Launcher.java 的  
相关源码

  3.1 Launcher.java 关于 app 加载的相关代码
----------------------------------

```
         public class Launcher extends BaseDraggingActivity implements LauncherExterns,
                LauncherModel.Callbacks, LauncherProviderChangeListener, UserEventDelegate,
                InvariantDeviceProfile.OnIDPChangeListener {
        @Override
            protected void onCreate(Bundle savedInstanceState) {
         
                super.onCreate(savedInstanceState);
                TraceHelper.partitionSection("Launcher-onCreate", "super call");
         ...
         
                mDragController = new DragController(this);
                mAllAppsController = new AllAppsTransitionController(this);
                mStateManager = new LauncherStateManager(this);
                UiFactory.onCreate(this);
         
                mAppWidgetManager = AppWidgetManagerCompat.getInstance(this);
         
                mAppWidgetHost = new LauncherAppWidgetHost(this);
                mAppWidgetHost.startListening();
         
                mLauncherView = LayoutInflater.from(this).inflate(R.layout.launcher, null);
         
                setupViews();
         .....
                if (!mModel.startLoader(currentScreen)) {
                    if (!internalStateHandled) {
                        // If we are not binding synchronously, show a fade in animation when
                        // the first page bind completes.
                        mDragLayer.getAlphaProperty(ALPHA_INDEX_LAUNCHER_LOAD).setValue(0);
                    }
                } else {
                    // Pages bound synchronously.
                    mWorkspace.setCurrentPage(currentScreen);
         
                    setWorkspaceLoading(true);
                }
         
                // For handling default keys
                setDefaultKeyMode(DEFAULT_KEYS_SEARCH_LOCAL);
         
                setContentView(mLauncherView);
                getRootView().dispatchInsets();
         
                // Listen for broadcasts
                registerReceiver(mScreenOffReceiver, new IntentFilter(Intent.ACTION_SCREEN_OFF));
         
                ....
            }
```

在上述的 Launcher.java 中的相关源码分析得知，在 onCreate(Bundle savedInstanceState) 中构建 Launcher3 桌面的核心数据中，在这里  
代码中的  
 if (!mModel.startLoader(currentScreen)) {  
            if (!internalStateHandled) {  
                // If we are not binding synchronously, show a fade in animation when  
                // the first page bind completes.  
                mDragLayer.getAlphaProperty(ALPHA_INDEX_LAUNCHER_LOAD).setValue(0);  
            }  
        } else {  
            // Pages bound synchronously.  
            mWorkspace.setCurrentPage(currentScreen);

            setWorkspaceLoading(true);  
        }  
负责开始加载 workspace 相关数据，核心功能还是通过 LauncherModel.java 中的 startLoader(currentScreen) 这个方法  
来具体加载 workspace 的数据，然后显示桌面数据信息，接下来分析下 LauncherModel.java 中的核心源码来看具体数据加载

  
3.2 LauncherModel 中关于加载数据的方法
-------------------------------

```
 
            public boolean startLoader(int synchronousBindPage) {
                // Enable queue before starting loader. It will get disabled in Launcher#finishBindingItems
                InstallShortcutReceiver.enableInstallQueue(InstallShortcutReceiver.FLAG_LOADER_RUNNING);
                synchronized (mLock) {
                    // Don't bother to start the thread if we know it's not going to do anything
                    if (mCallbacks != null && mCallbacks.get() != null) {
                        final Callbacks oldCallbacks = mCallbacks.get();
                        // Clear any pending bind-runnables from the synchronized load process.
                        mUiExecutor.execute(oldCallbacks::clearPendingBinds);
         
                        // If there is already one running, tell it to stop.
                        stopLoader();
                        LoaderResults loaderResults = new LoaderResults(mApp, sBgDataModel,
                                mBgAllAppsList, synchronousBindPage, mCallbacks);
                        if (mModelLoaded && !mIsLoaderTaskRunning) {
                            // Divide the set of loaded items into those that we are binding synchronously,
                            // and everything else that is to be bound normally (asynchronously).
                            loaderResults.bindWorkspace();
                            // For now, continue posting the binding of AllApps as there are other
                            // issues that arise from that.
                            loaderResults.bindAllApps();
                            loaderResults.bindDeepShortcuts();
                            loaderResults.bindWidgets();
                            return true;
                        } else {
                            startLoaderForResults(loaderResults);
                        }
                    }
                }
                return false;
            }
```

在上述的 LauncherModel 中的核心源码分析得知，  
在 LauncherModel 的相关方法中发现最后在 run（）中来具体负责加载和绑定 workspace 的数据的方法，  
所以可以看出这里是 LauncherModel 的数据处理的核心方法，接下来看下 loadWorkspace() 的数据处理的  
核心方法如下

```
  private void loadWorkspace() {
           ....
         
                                case LauncherSettings.Favorites.ITEM_TYPE_FOLDER:
                                    FolderInfo folderInfo = mBgDataModel.findOrMakeFolder(c.id);
                                    c.applyCommonProperties(folderInfo);
         
                                    // Do not trim the folder label, as is was set by the user.
                                    folderInfo.title = c.getString(c.titleIndex);
                                    folderInfo.spanX = 1;
                                    folderInfo.spanY = 1;
                                    folderInfo.options = c.getInt(optionsIndex);
         
                                    // no special handling required for restored folders
                                    c.markRestored();
         
                                    c.checkAndAddItem(folderInfo, mBgDataModel);
                                    break;
         
                                case LauncherSettings.Favorites.ITEM_TYPE_APPWIDGET:
                                    if (FeatureFlags.GO_DISABLE_WIDGETS) {
                                        c.markDeleted("Only legacy shortcuts can have null package");
                                        continue;
                                    }
                                    // Follow through
                                .....
                }
            }
         
                LogUtils.d(TAG, "loadWorkspace: loading default favorites");
                LauncherSettings.Settings.call(contentResolver,
                        LauncherSettings.Settings.METHOD_LOAD_DEFAULT_FAVORITES);
```

从上述的在上述的 LauncherModel 中的核心源码分析得知，在加载数据的核心方法中 loadWorkspace() 的核心源码中分析得知，这里主要是  
调用 LauncherProvider 的 loadDefaultFavoritesIfNecessary() 方法, 此方法比较简单, 有很详细的注解, 主要功能就是读取客制化主界面的配置文件, 保存到数据库  
所以说接下来分析下 LauncherProvider 的核心方法

  
3.3 LauncherProvider 关于加载默认数据方法
----------------------------------

```
           synchronized private void loadDefaultFavoritesIfNecessary() {
         
                if (getFlagEmptyDbCreated(getContext(), mOpenHelper.getDatabaseName())) {
                    Log.d(TAG, "loading default workspace");
         
                    AppWidgetHost widgetHost = mOpenHelper.newLauncherWidgetHost();
                    AutoInstallsLayout loader = createWorkspaceLoaderFromAppRestriction(widgetHost);
                    if (loader == null) {
                        loader = AutoInstallsLayout.get(getContext(), widgetHost, mOpenHelper);
                    }
                    if (loader == null) {
                        final Partner partner = Partner.get(getContext().getPackageManager());
                        if (partner != null && partner.hasDefaultLayout()) {
                            final Resources partnerRes = partner.getResources();
                            int workspaceResId = partnerRes.getIdentifier(Partner.RES_DEFAULT_LAYOUT,
                                    "xml", partner.getPackageName());
                            if (workspaceResId != 0) {
                                loader = new DefaultLayoutParser(getContext(), widgetHost,
                                        mOpenHelper, partnerRes, workspaceResId);
                            }
                        }
                    }
         
                    final boolean usingExternallyProvidedLayout = loader != null;
                    if (loader == null) {
                        loader = getDefaultLayoutParser(widgetHost);
                    }
         
                    if ((mOpenHelper.loadFavorites(mOpenHelper.getWritableDatabase(), loader) <= 0)
                            && usingExternallyProvidedLayout) {
                        // Unable to load external layout. Cleanup and load the internal layout.
                        mOpenHelper.createEmptyDB(mOpenHelper.getWritableDatabase());
                        mOpenHelper.loadFavorites(mOpenHelper.getWritableDatabase(),
                                getDefaultLayoutParser(widgetHost));
                    }
                    clearFlagEmptyDbCreated(getContext(), mOpenHelper.getDatabaseName());
                }
            }
```

在上述的 LauncherProvider.java 中的核心方法中，通过源码分析得知，在 loadDefaultFavoritesIfNecessary() 这个方法中，来加载数据的相关  
源码分析得知，核心代码端就是这里，在  
这段代码  
final Resources partnerRes = partner.getResources();  
                    int workspaceResId = partnerRes.getIdentifier(Partner.RES_DEFAULT_LAYOUT,  
                            "xml", partner.getPackageName());  
                    if (workspaceResId != 0) {  
                        loader = new DefaultLayoutParser(getContext(), widgetHost,  
                                mOpenHelper, partnerRes, workspaceResId);  
                    }  
就是解析默认 xml 的数据来加载到数据库  
Partner.RES_DEFAULT_LAYOUT 就是 res/xml/partner_default_layout.xml 文件，在某些平台像 mtk，展现，  
高通，rk 等等，可能不会存在 partner_default_layout.xml 这个文件，所以就需要找到  
default_workspace_5x5.xml default_workspace_4x4.xml 等默认的布局文件，来增加自己  
所需要的排序作为单层桌面的默认排序功能的实现

  
3.4 Launcher3 单层 app 列表页排序功能具体实现
-----------------------------------

        修改 partner_default_layout.xml 的布局排序

```
      <favorites xmlns:launcher="http://schemas.android.com/apk/res-auto/com.android.launcher3">
 
            <appwidget
            launcher:package
            launcher:class
            launcher:screen="0"
            launcher:x="1"
            launcher:y="1"
            launcher:spanX="4"
            launcher:spanY="2" 
          />
         
            <!-- Hotseat (We use the screen as the position of the item in the hotseat) -->
            <!-- Dialer, Messaging, Browser, Camera -->
         
            <resolve
                launcher:container="-101"
                launcher:screen="0"
                launcher:x="0"
                launcher:y="0" >
                <favorite launcher:uri="#Intent;action=android.intent.action.MAIN;category=android.intent.category.APP_MAPS;end" />
                <favorite launcher:uri="#Intent;action=android.intent.action.MAIN;category=android.intent.category.APP_MUSIC;end" />
            </resolve>
         
            <resolve
                launcher:container="-101"
                launcher:screen="1"
                launcher:x="1"
                launcher:y="0" >
                <favorite launcher:uri="#Intent;action=android.intent.action.MAIN;category=android.intent.category.APP_GALLERY;end" />
                <favorite launcher:uri="#Intent;type=images/*;end" />
            </resolve>
         
            <favorite
                launcher:container="-101"
                launcher:screen="2"
                launcher:x="2"
                launcher:y="0" 
                launcher:package
                launcher:class/>
         
         ....
```

根据自己的需要排序  
    favorite: 应用程序快捷方式。  
    shortcut: 链接，如网址，本地磁盘路径等。  
    search: 搜索框。  
    clock: 桌面上的钟表 Widget  
       
    // 支持的属性有：  
    launcher:title: 图标下面的文字，目前只支持引用，不能直接书写字符串;  
    launcher:icon: 图标引用;  
    launcher:uri: 链接地址, 链接网址用的，使用 shortcut 标签就可以定义一个超链接，打开某个网址。  
    launcher:packageName: 应用程序的包名;  
    launcher:className: 应用程序的启动类名;  
    launcher:screen: 图标所在的屏幕编号，从 0 开始表示在第几屏;  
    launcher:x: 图标在横向排列上的序号;  
    launcher:y: 图标在纵向排列上的序号;  
       
    // 例如：  
    <appwidget  
        launcher:package // 应用包名  
        launcher:class         // 该应用的类名  
        launcher:screen="1"             // 第 1 屏, 0-4 屏共 5 屏  
        launcher:x="2"                      // 图标 X 位置, 左上角第一个为 0, 向左递增, 0-4 共 5 个  
        launcher:y="1"                                                 // 图标 Y 位置, 左上角第一个为 0, 向下递增, 0-2 共 3 个  
        launcher:spanX="3"                                             // 在 x 方向上所占格数  
        launcher:spanY="2" />                                          // 在 y 方向上所占格数
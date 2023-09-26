> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124870465)

### 1. 概述

在 11.0 定制化开发中，Launcher3 去掉搜索栏也是个常见的功能开发，搜索栏就是 workspace 第一页和第二页，所以去掉这两页就可以了

### 2.Launcher3 去掉默认的 google 搜索栏的核心类

```
packages\apps\Launcher3\src\com\android\launcher3\Workspace.java
packages/apps/Launcher3/src/com/android/launcher3/allapps/AllAppsContainerView.java
packages\apps\Launcher3\src\com\android\launcher3\allapps\AllAppsTransitionController.java

```

### 3.Launcher3 去掉默认的 google 搜索栏的核心功能分析和实现

一：去掉首页绑定的搜索栏  
修改位置  
packages\apps\Launcher3\src\com\android\launcher3\Workspace.java

```
 bindAndInitFirstWorkspaceScreen（）
    public void bindAndInitFirstWorkspaceScreen(View qsb) {
        if (!FeatureFlags.QSB_ON_FIRST_SCREEN) {
            return;
        }
        // Add the first page
        CellLayout firstPage = insertNewWorkspaceScreen(Workspace.FIRST_SCREEN_ID, 0);
        // Always add a QSB on the first screen.
        if (qsb == null) {
            // In transposed layout, we add the QSB in the Grid. As workspace does not touch the
            // edges, we do not need a full width QSB.
            qsb = LayoutInflater.from(getContext())
                    .inflate(R.layout.search_container_workspace,firstPage, false);
        }

        CellLayout.LayoutParams lp = new CellLayout.LayoutParams(0, 0, firstPage.getCountX(), 1);
        lp.canReorder = false;
        if (!firstPage.addViewToCellLayout(qsb, 0, R.id.search_container_workspace, lp, true)) {
            Log.e(TAG, "Failed to add to item at (0, 0) to CellLayout");
        }
    }

```

从源码中看到 qsb 就是搜索栏 注释掉 qsb 就可以了

```
public void bindAndInitFirstWorkspaceScreen(View qsb) {
        if (!FeatureFlags.QSB_ON_FIRST_SCREEN) {
            return;
        }
        // Add the first page
        CellLayout firstPage = insertNewWorkspaceScreen(Workspace.FIRST_SCREEN_ID, 0);
        if (FeatureFlags.PULLDOWN_SEARCH) {
            .....
        }
        //add don't show google quick search box[qsb]
        // Always add a QSB on the first screen.
       
       -if (qsb == null) {
       -     // In transposed layout, we add the QSB in the Grid. As workspace does not touch the
       -    // edges, we do not need a full width QSB.
       -     qsb = LayoutInflater.from(getContext())
       -             .inflate(R.layout.search_container_workspace,firstPage, false);
       - }

        
        -CellLayout.LayoutParams lp = new CellLayout.LayoutParams(0, 0, firstPage.getCountX(), 1);
        -lp.canReorder = false;
        -if (!firstPage.addViewToCellLayout(qsb, 0, R.id.search_container_workspace, lp, true)) {
        -    Log.e(TAG, "Failed to add to item at (0, 0) to CellLayout");
        -}
        
    }

```

二、去掉所有 app 界面搜索应用栏  
AllAppsContainerView.java 中

```
@Override
protected void onFinishInflate() {
    super.onFinishInflate();

    // This is a focus listener that proxies focus from a view into the list view.  This is to
    // work around the search box from getting first focus and showing the cursor.
    setOnFocusChangeListener((v, hasFocus) -> {
        if (hasFocus && getActiveRecyclerView() != null) {
            getActiveRecyclerView().requestFocus();
        }
    });

    mHeader = findViewById(R.id.all_apps_header);
    rebindAdapters(mUsingTabs, true /* force */);

    mSearchContainer = findViewById(R.id.search_container_all_apps);
    mSearchUiManager = (SearchUiManager) mSearchContainer;
    mSearchUiManager.initialize(this);
}

```

从源码可以看出 search_container_all_apps 就是搜索栏

修改位置  
packages\apps\Launcher3\src\com\android\launcher3\allapps\AllAppsContainerView.java  
在 onFinishInflate() 中添加一行 mSearchContainer.setVisibility(View.GONE);

```
@Override
protected void onFinishInflate() {
    super.onFinishInflate();
    mSearchContainer = findViewById(R.id.search_container_all_apps);
    mSearchUiManager = (SearchUiManager) mSearchContainer;
    mSearchUiManager.initialize(mApps, mAppsRecyclerView);
    /// add this code don't show all app quick search box
    + mSearchContainer.setVisibility(View.GONE);
}

```

三 AllAppsTransitionController.java 的修改  
packages\apps\Launcher3\src\com\android\launcher3\allapps\AllAppsTransitionController.java  
注释 setAlphas() 中的 mAppsView.getSearchUiManager()

```
public void setAlphas(int visibleElements, AnimationConfig config, AnimatorSetBuilder builder) {
        PropertySetter setter = config == null ? NO_ANIM_PROPERTY_SETTER
                : config.getPropertySetter(builder);
        boolean hasHeaderExtra = (visibleElements & ALL_APPS_HEADER_EXTRA) != 0;
        boolean hasContent = (visibleElements & ALL_APPS_CONTENT) != 0;

        Interpolator allAppsFade = builder.getInterpolator(ANIM_ALL_APPS_FADE, LINEAR);
        setter.setViewAlpha(mAppsView.getContentView(), hasContent ? 1 : 0, allAppsFade);
        setter.setViewAlpha(mAppsView.getScrollBar(), hasContent ? 1 : 0, allAppsFade);
        mAppsView.getFloatingHeaderView().setContentVisibility(hasHeaderExtra, hasContent, setter,
                allAppsFade);
        ///annotaion this code don't show all app quick search box
        //mAppsView.getSearchUiManager().setContentVisibility(visibleElements, setter, allAppsFade);

        setter.setInt(mScrimView, ScrimView.DRAG_HANDLE_ALPHA,
                (visibleElements & VERTICAL_SWIPE_INDICATOR) != 0 ? 255 : 0, allAppsFade);
    }

```
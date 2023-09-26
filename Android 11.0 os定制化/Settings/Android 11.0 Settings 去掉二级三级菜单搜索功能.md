> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124890459)

### 1. 概述

在 11.0 由于客户定制开发需求，需要去掉 Settings 里面的[搜索功能](https://so.csdn.net/so/search?q=%E6%90%9C%E7%B4%A2%E5%8A%9F%E8%83%BD&spm=1001.2101.3001.7020)，主页面的搜索功能，在前面的章节已经讲了  
这里需要去掉二级三级菜单的搜索功能，需要从搜索功能流程分析去掉搜索功能  
博客地址：10.0 Settings 去掉搜索框

### 2.Settings 去掉二级三级菜单搜索功能核心代码

```
packages/apps/Settings/src/com/android/settings/search/actionbar/SearchMenuController.java
 /packages/apps/Settings/src/com/android/settings/SettingsPreferenceFragment.java

```

### 3.Settings 去掉二级三级菜单搜索功能核心功能分析

### 3.1SettingsPreferenceFragment 关于菜单管理类的相关初始化操作

二级三级菜单就需要一步步跟源码来根据原理实现  
每一个 [Fragment](https://so.csdn.net/so/search?q=Fragment&spm=1001.2101.3001.7020) 都要继承 DashboardFragment 而 DashboardFragment 又继承 SettingsPreferenceFragment 进入 SettingsPreferenceFragment 后发现

```
public abstract class SettingsPreferenceFragment extends InstrumentedPreferenceFragment
         implements DialogCreatable, HelpResourceProvider, Indexable {
  @Override
    public void onCreate(Bundle icicle) {
        super.onCreate(icicle);
        SearchMenuController.init(this /* host */);
        HelpMenuController.init(this /* host */);

        if (icicle != null) {
            mPreferenceHighlighted = icicle.getBoolean(SAVE_HIGHLIGHTED_KEY);
        }
        HighlightablePreferenceGroupAdapter.adjustInitialExpandedChildCount(this /* host */);
    }
    @Override
     public View onCreateView(LayoutInflater inflater, ViewGroup container,
             Bundle savedInstanceState) {
         final View root = super.onCreateView(inflater, container, savedInstanceState);
         mPinnedHeaderFrameLayout = root.findViewById(R.id.pinned_header);
         return root;
     }
 
     @Override
     public void addPreferencesFromResource(@XmlRes int preferencesResId) {
         super.addPreferencesFromResource(preferencesResId);
         checkAvailablePrefs(getPreferenceScreen());
     }
 
     @VisibleForTesting
     void checkAvailablePrefs(PreferenceGroup preferenceGroup) {
         if (preferenceGroup == null) return;
         for (int i = 0; i < preferenceGroup.getPreferenceCount(); i++) {
             Preference pref = preferenceGroup.getPreference(i);
             if (pref instanceof SelfAvailablePreference
                     && !((SelfAvailablePreference) pref).isAvailable(getContext())) {
                 pref.setVisible(false);
             } else if (pref instanceof PreferenceGroup) {
                 checkAvailablePrefs((PreferenceGroup) pref);
             }
         }
     }
 
     public View setPinnedHeaderView(int layoutResId) {
         final LayoutInflater inflater = getActivity().getLayoutInflater();
         final View pinnedHeader =
                 inflater.inflate(layoutResId, mPinnedHeaderFrameLayout, false);
         setPinnedHeaderView(pinnedHeader);
         return pinnedHeader;
     }
 
     public void setPinnedHeaderView(View pinnedHeader) {
         mPinnedHeaderFrameLayout.addView(pinnedHeader);
         mPinnedHeaderFrameLayout.setVisibility(View.VISIBLE);
     }
 
     public void showPinnedHeader(boolean show) {
         mPinnedHeaderFrameLayout.setVisibility(show ? View.VISIBLE : View.INVISIBLE);
     }
 

```

SearchMenuController.init(this /* host _/);  
HelpMenuController.init(this /_ host */);  
就是关于菜单搜索管理类，在这里实现对搜索功能的管理  
都会初始化 SearchMenuController 来实现搜索功能的管理类

### 3.2 SearchMenuController 关于搜索相关内容显示列表相关功能实现

```
package com.android.settings.search.actionbar;

import android.annotation.NonNull;
import android.app.settings.SettingsEnums;
import android.content.Context;
import android.content.Intent;
import android.content.pm.PackageManager;
import android.os.Bundle;
import android.view.Menu;
import android.view.MenuInflater;
import android.view.MenuItem;
import androidx.fragment.app.Fragment;

import com.android.settings.R;
import com.android.settings.Utils;
import com.android.settings.core.InstrumentedFragment;
import com.android.settings.core.InstrumentedPreferenceFragment;
import com.android.settings.overlay.FeatureFactory;
import com.android.settings.search.SearchFeatureProvider;
import com.android.settingslib.core.lifecycle.LifecycleObserver;
import com.android.settingslib.core.lifecycle.events.OnCreateOptionsMenu;

public class SearchMenuController implements LifecycleObserver, OnCreateOptionsMenu {

    public static final String NEED_SEARCH_ICON_IN_ACTION_BAR = "need_search_icon_in_action_bar";

    private final Fragment mHost;
    private final int mPageId;

    public static void init(@NonNull InstrumentedPreferenceFragment host) {
        host.getSettingsLifecycle().addObserver(
                new SearchMenuController(host, host.getMetricsCategory()));
    }

    public static void init(@NonNull InstrumentedFragment host) {
        host.getSettingsLifecycle().addObserver(
                new SearchMenuController(host, host.getMetricsCategory()));
    }

    private SearchMenuController(@NonNull Fragment host, int pageId) {
        mHost = host;
        mPageId = pageId;
    }

    @Override
    public void onCreateOptionsMenu(Menu menu, MenuInflater inflater) {
        final Context context = mHost.getContext();
        final String SettingsIntelligencePkgName = context.getString(
                R.string.config_settingsintelligence_package_name);
        if (!Utils.isDeviceProvisioned(mHost.getContext())) {
            return;
        }
        if (!Utils.isPackageEnabled(mHost.getContext(), SettingsIntelligencePkgName)) {
            return;
        }
        if (menu == null) {
            return;
        }
        /* bug 1104944 :on Ultra power saving mode, need to hide SearchMenu and Multiuser settings. @{*/
        if (Utils.inUtraPowerSavingMode()) {
            return;
        }
        /*@}*/
        final Bundle arguments = mHost.getArguments();
        if (arguments != null && !arguments.getBoolean(NEED_SEARCH_ICON_IN_ACTION_BAR, true)) {
            return;
        }
        final MenuItem searchItem = menu.add(Menu.NONE, Menu.NONE, 0 /* order */,
                R.string.search_menu);
        searchItem.setIcon(R.drawable.ic_search_24dp);
        searchItem.setShowAsAction(MenuItem.SHOW_AS_ACTION_NEVER);

        searchItem.setOnMenuItemClickListener(target -> {
            final Intent intent = FeatureFactory.getFactory(context)
                    .getSearchFeatureProvider()
                    .buildSearchIntent(context, mPageId);

            if (context.getPackageManager().queryIntentActivities(intent,
                    PackageManager.MATCH_DEFAULT_ONLY).isEmpty()) {
                return true;
            }

            FeatureFactory.getFactory(context).getMetricsFeatureProvider()
                    .action(context, SettingsEnums.ACTION_SEARCH_RESULTS);
            //Add for bug 1169791, avoid ActivityNotFoundException
            if (Utils.isIntentCanBeResolved(context, intent)) {
                mHost.startActivityForResult(intent, SearchFeatureProvider.REQUEST_CODE);
            }
            return true;
        });
    }
}

```

而 SearchMenuController 通过动态添加 search 图标  
final [MenuItem](https://so.csdn.net/so/search?q=MenuItem&spm=1001.2101.3001.7020) searchItem = menu.add(Menu.NONE, Menu.NONE, 0 /* order */,  
R.string.search_menu); 就是搜索项  
而 searchItem.setShowAsAction(MenuItem.SHOW_AS_ACTION_NEVER);  
就是设置不显示搜索项  
修改如下:

```
--- a/packages/apps/Settings/src/com/android/settings/search/actionbar/SearchMenuController.java
+++ b/packages/apps/Settings/src/com/android/settings/search/actionbar/SearchMenuController.java
@@ -84,7 +84,7 @@ public class SearchMenuController implements LifecycleObserver, OnCreateOptionsM
         final MenuItem searchItem = menu.add(Menu.NONE, Menu.NONE, 0 /* order */,
                 R.string.search_menu);
         searchItem.setIcon(R.drawable.ic_search_24dp);
-        searchItem.setShowAsAction(MenuItem.SHOW_AS_ACTION_ALWAYS);
+        searchItem.setShowAsAction(MenuItem.SHOW_AS_ACTION_NEVER);
+        menu.setGroupVisible(0,false); //隐藏Toolbar 按钮 去掉搜索设置显示
         searchItem.setOnMenuItemClickListener(target -> {
             final Intent intent = FeatureFactory.getFactory(context)

```
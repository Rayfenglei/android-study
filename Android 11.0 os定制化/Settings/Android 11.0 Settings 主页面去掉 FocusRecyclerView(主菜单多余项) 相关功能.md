> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/129936890)

1. 前言
-----

  
 在 11.0 的系统 rom 产品定制化开发中，在原生系统 Settings 主页面的主菜单中，在系统 Settings 测试某些功能的时候，比如开启护眼模式和改变系统密度  
会在主菜单第一项的网络菜单头部增加 自定义您的设备和设置护眼模式时间安排 等等相关的设置模块这对于原生系统设置菜单布局来说显示相当不美观，所以根据系统定制要求需要去掉这部分功能，这就需要根据系统 settings 显示流程来分析这部分功能  
然后来实现功能

2.Settings 主页面去掉 FocusRecyclerView(主菜单多余项) 相关功能的核心类
---------------------------------------------------

```
   packages/apps/Settings/src/com/android/settings/homepage/contextualcards/ContextualCardsFragment.java
   packages/apps/Settings/res/layout/settings_homepage.xml
   packages/apps/Settings/src/com/android/settings/homepage/SettingsHomepageActivity.java
```

3.Settings 主页面去掉 FocusRecyclerView(主菜单多余项) 相关功能的核心功能分析和实现
---------------------------------------------------------

在原生系统 Settings 主页面的主菜单中，在系统 settings 中的主菜单中开启护眼模式和改变系统 density 等操作，会在主菜单第一项的网络菜单头部增加 自定义您的设备和设置护眼模式时间安排 等等相关的设置模块经过 android studio 布局工具等分析得知主要是 ContextualCardsFragment.java 中的相关布局来负责加载这些菜单项 接下来分析  
ContextualCardsFragment.java 的相关源码如下:

```
 public class ContextualCardsFragment extends InstrumentedFragment implements
        FocusRecyclerView.FocusListener {
  @Override
      public void onCreate(Bundle savedInstanceState) {
          super.onCreate(savedInstanceState);
          final Context context = getContext();
          if (savedInstanceState == null) {
              FeatureFactory.getFactory(context).getSlicesFeatureProvider().newUiSession();
              BluetoothUpdateWorker.initLocalBtManager(getContext());
          }
          mContextualCardManager = new ContextualCardManager(context, getSettingsLifecycle(),
                  savedInstanceState);
          mKeyEventReceiver = new KeyEventReceiver();
      }
  
      @Override
      public void onStart() {
          super.onStart();
          registerScreenOffReceiver();
          registerKeyEventReceiver();
          ContextualWifiScanWorker.newVisibleUiSession();
          mContextualCardManager.loadContextualCards(LoaderManager.getInstance(this),
                  sRestartLoaderNeeded);
          sRestartLoaderNeeded = false;
      }
      
 @Override
      public View onCreateView(LayoutInflater inflater, ViewGroup container,
              Bundle savedInstanceState) {
          final Context context = getContext();
          final View rootView = inflater.inflate(R.layout.settings_homepage, container, false);
          mCardsContainer = rootView.findViewById(R.id.card_container);
          mLayoutManager = new GridLayoutManager(getActivity(), SPAN_COUNT,
                  GridLayoutManager.VERTICAL, false /* reverseLayout */) {
              @Override
              public void onLayoutCompleted(RecyclerView.State state) {
                  super.onLayoutCompleted(state);
                  // Once cards finish laying out, make the RV back to wrap content for flexibility.
                  final ViewGroup.LayoutParams params = mCardsContainer.getLayoutParams();
                  if (params.height != WRAP_CONTENT) {
                      params.height = WRAP_CONTENT;
                      mCardsContainer.setLayoutParams(params);
                  }
              }
          };
          mCardsContainer.setLayoutManager(mLayoutManager);
          preAllocateHeight(context);
          mContextualCardsAdapter = new ContextualCardsAdapter(context, this /* lifecycleOwner */,
                  mContextualCardManager);
          mCardsContainer.setItemAnimator(null);
          mCardsContainer.setAdapter(mContextualCardsAdapter);
          mContextualCardManager.setListener(mContextualCardsAdapter);
          mCardsContainer.setListener(this);
          mItemTouchHelper = new ItemTouchHelper(new SwipeDismissalDelegate(mContextualCardsAdapter));
          mItemTouchHelper.attachToRecyclerView(mCardsContainer);
  
          return rootView;
      }
```

在上述的 ContextualCardsFragment 的相关源码中分析得知，在 onCreate(Bundle savedInstanceState) 中主要就是  
构建 mContextualCardManager 管理类，onStart() 中的 mContextualCardManager.loadContextualCards 加载相关的自定义设置布局  
在 onCreateView([LayoutInflater](https://so.csdn.net/so/search?q=LayoutInflater&spm=1001.2101.3001.7020) inflater, ViewGroup container,  
Bundle savedInstanceState) 这个构造自定义菜单的方法中，ContextualCardsAdapter 这个适配器中，在通过适配器类中的  
mCardsContainer，这个从 settings_homepage 中的控件来加载上述图中的自定义菜单项 ，接下来分析下 settings_homepage.xml 中的相关源码

3.2 settings_homepage.xml 的相关代码分析
---------------------------------

```
<?xml version="1.0" encoding="utf-8"?>
<!--
     Copyright (C) 2018 The Android Open Source Project
 
     Licensed under the Apache License, Version 2.0 (the "License");
     you may not use this file except in compliance with the License.
     You may obtain a copy of the License at
 
          http://www.apache.org/licenses/LICENSE-2.0
 
      Unless required by applicable law or agreed to in writing, software
      distributed under the License is distributed on an "AS IS" BASIS,
      WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
      See the License for the specific language governing permissions and
      limitations under the License.
 -->
 
 <LinearLayout
     xmlns:android="http://schemas.android.com/apk/res/android"
     android:layout_width="match_parent"
     android:layout_height="wrap_content"
     android:orientation="vertical">
 
     <com.android.settings.homepage.contextualcards.FocusRecyclerView
         android:id="@+id/card_container"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
+ android:visibility="gone"
         android:layoutAnimation="@anim/layout_animation_fade_in"
         android:importantForAccessibility="no"/>
 
 </LinearLayout>
```

从 settings_homepage.xml 的布局的上述代码中, 可以发现在 com.android.settings.homepage.contextualcards.FocusRecyclerView  
就是上图中的显示多出来的提醒条目，主要是在 ContextualCardsFragment 来加载多余出来的主菜单选项，所以需要在这里  
进行相关的处理，所以就需要在布局中隐藏这个控件显示就可以了
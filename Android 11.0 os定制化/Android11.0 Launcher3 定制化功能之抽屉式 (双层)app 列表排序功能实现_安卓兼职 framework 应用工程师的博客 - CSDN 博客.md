> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/132515924)

1. 概述
-----

  在 11.0 的系统开发中，在定制 Launcher3 的开发中，对于抽屉式即双层桌面的 workspace 的 app [列表排序](https://so.csdn.net/so/search?q=%E5%88%97%E8%A1%A8%E6%8E%92%E5%BA%8F&spm=1001.2101.3001.7020)的功能，也是常有的需求，把常用的 app 图标放在前面，其他的可以放在列表后面做个整体的排序，这就需要了解 app 列表排序的流程，然后根据需求来实现功能

如图:

![](https://img-blog.csdnimg.cn/710e2620a5c34882833784baf6a00c22.png)

2.Launcher3 抽屉式 (双层)app 列表排序的相关代码
---------------------------------

```
      packages\apps\Launcher3\src\com\android\launcher3\allapps\AllAppsStore.java
      packages\apps\Launcher3\src\com\android\launcher3\allapps\AlphabeticalAppsList.java
      packages\apps\Launcher3\src\com\android\launcher3\model\BaseModelUpdateTask.java
      packages\apps\Launcher3\res\values\config.xml
```

3.Launcher3 抽屉式 (双层)app 列表排序的相关代码和功能实现  
 
------------------------------------------

在 11.0 中， [Launcher](https://so.csdn.net/so/search?q=Launcher&spm=1001.2101.3001.7020) 顾名思义, 就是桌面的意思, 也是 android 系统启动后第一个启动的应用程序, 这里以 android11 为例  
在 Launcher3 就是系统原生的 Launcher, 同样也是一个 app, 所以 Launcher 就是一个 Activity，Launcher 的源码中也是继承的 Activity，Launcher3 里面有好多个复杂的 activitity,

核心类:  
AppWidgetManagerCompat：兼容抽象基类，负责处理不通版本下应用和 Widget 管理  
LauncherAppWidgetHost：继承子 AppWidgetHost，顾名思义，AppWidgetHost 是桌面 app、widget 等的宿主，之所以继承是为了 LauncherAppWidgetHostView 能更好的处理长按事件；  
FocusIndicatorView：一个实现了 View.OnFocusChangeListener 的 View（具体作用上不清楚）  
DragLayer：一个用来协调子 View 拖拽事件的 ViewGroup，实际上事件的分发拦截等是在 DragController，因为 DragLayer 持有 DragController 的实例，并调用了 setup 方法初始化了它；  
AllAppsStore.java 主要就是负责更新 app 列表，在安装 app 和卸载 app 的时候，收到相关的广播后执行相关的刷新 app 的动作

AlphabeticalAppsList.java 这个类主要就是在加载完 app 列表数据以后，在根据首字母的排序来

显示 app 列表，所以可以说关于 app 显示列表排序可以从这里出发分析相关代码

3.1 AllAppsStore.java 相关代码分析
----------------------------

```
       public class AllAppsStore {
     
        public Collection<AppInfo> getApps() {
            return mComponentToAppMap.values();
        }
     
        public void addOrUpdateApps(List<AppInfo> apps) {
            for (AppInfo app : apps) {
                mComponentToAppMap.put(app.toComponentKey(), app);
            }
            notifyUpdate();
        }
     
        /**
         * Removes some apps from the list.
         */
        public void removeApps(List<AppInfo> apps) {
            for (AppInfo app : apps) {
                mComponentToAppMap.remove(app.toComponentKey());
            }
            notifyUpdate();
        }
     
     
        private void notifyUpdate() {
            if (mDeferUpdatesFlags != 0) {
                mUpdatePending = true;
                if (LogUtils.DEBUG_ALL) {
                    LogUtils.d("AllAppsStore", "notifyUpdate mDeferUpdatesFlags:"
                            + Integer.toBinaryString(mDeferUpdatesFlags));
                }
                return;
            }
            int count = mUpdateListeners.size();
            for (int i = 0; i < count; i++) {
                mUpdateListeners.get(i).onAppsUpdated();
            }
        }
 
        public void unregisterIconContainer(ViewGroup container) {
            mIconContainers.remove(container);
        }
     
        public void updateNotificationDots(Predicate<PackageUserKey> updatedDots) {
            updateAllIcons((child) -> {
                if (child.getTag() instanceof ItemInfo) {
                    ItemInfo info = (ItemInfo) child.getTag();
                    if (mTempKey.updateFromItemInfo(info) && updatedDots.test(mTempKey)) {
                        child.applyDotState(info, true /* animate */);
                    }
                }
            });
        }
     
     
     
        private void updateAllIcons(Consumer<BubbleTextView> action) {
            for (int i = mIconContainers.size() - 1; i >= 0; i--) {
                ViewGroup parent = mIconContainers.get(i);
     
                if (parent instanceof AllAppsRecyclerView) {
                    AlphabeticalAppsList apps = ((AllAppsRecyclerView) parent).getApps();
                    if (null != apps) {
                        List<AlphabeticalAppsList.AdapterItem> items = apps.getAdapterItems();
                        for (int j = 0; j < items.size(); j++) {
                            BubbleTextView child = items.get(j).iconView;
                            if (null != child) {
                                action.accept(child);
                            }
                        }
                    }
                } else {
                    int childCount = parent.getChildCount();
     
                    for (int j = 0; j < childCount; j++) {
                        View child = parent.getChildAt(j);
                        if (child instanceof BubbleTextView) {
                            action.accept((BubbleTextView) child);
                        }
                    }
                }
            }
        }
     
    }
```

在 AllAppsStore.java 中的相关源码中，可以看出 addOrUpdateApps(List<AppInfo> apps) 负责更新 app 列表  
当安装和卸载的 app 的时候，所以 mUpdateListeners.get(i).onAppsUpdated(); 来更新 app 的列表然后更新桌面的 app 的列表  
具体是由 OnUpdateListener 的 onAppsUpdated(); 实现的，接下来看下 AlphabeticalAppsList.java 关于 app 排序的相关功能

3.2 AlphabeticalAppsList.java 关于 app 排序的相关功能分析
----------------------------------------------

```
  public class AlphabeticalAppsList implements AllAppsStore.OnUpdateListener {
 
        @Override
        public void onAppsUpdated() {
            // Sort the list of apps
            mApps.clear();
     
            for (AppInfo app : mAllAppsStore.getApps()) {
                if (mItemFilter == null || mItemFilter.matches(app, null) || hasFilter()) {
                    mApps.add(app);
                }
            }
     
            Collections.sort(mApps, mAppNameComparator);
     
            // As a special case for some languages (currently only Simplified Chinese), we may need to
            // coalesce sections
            Locale curLocale = mLauncher.getResources().getConfiguration().locale;
            boolean localeRequiresSectionSorting = curLocale.equals(Locale.SIMPLIFIED_CHINESE);
            if (localeRequiresSectionSorting) {
                // Compute the section headers. We use a TreeMap with the section name comparator to
                // ensure that the sections are ordered when we iterate over it later
                TreeMap<String, ArrayList<AppInfo>> sectionMap = new TreeMap<>(new LabelComparator());
                for (AppInfo info : mApps) {
                    // Add the section to the cache
                    String sectionName = getAndUpdateCachedSectionName(info.title);
     
                    // Add it to the mapping
                    ArrayList<AppInfo> sectionApps = sectionMap.get(sectionName);
                    if (sectionApps == null) {
                        sectionApps = new ArrayList<>();
                        sectionMap.put(sectionName, sectionApps);
                    }
                    sectionApps.add(info);
                }
     
                // Add each of the section apps to the list in order
                mApps.clear();
                for (Map.Entry<String, ArrayList<AppInfo>> entry : sectionMap.entrySet()) {
                    mApps.addAll(entry.getValue());
                }
            } else {
                // Just compute the section headers for use below
                for (AppInfo info : mApps) {
                    // Add the section to the cache
                    getAndUpdateCachedSectionName(info.title);
                }
            }
     
            LauncherAppMonitor.getInstance(mLauncher).onAllAppsListUpdated(mApps);
            updateAdapterItems();
        }
```

在 AlphabeticalAppsList.java 中的上述的相关源码分析，得知在负责更新 app 列表的 onAppsUpdated() 中的  
核心方法中，在这里主要是就是由  
Collections.sort(mApps, mAppNameComparator); 所以可以在 onAppsUpdated() 中的相关源码中，  
负责对 app 列表按照名称排序增加排序的相关方法，来具体实现桌面的相关排序

```
 private List<AppInfo> getSortArrayApps(){
    		 String[] appPkgAndClsName = new String[]{
    				"com.android.music/com.android.music.MusicBrowserActivity",
    				"com.android.deskclock/com.android.deskclock.DeskClock",
    				"com.android.settings/com.android.settings.Settings",
    				"com.android.calendar/com.android.calendar.AllInOneActivity",
    				"com.sprd.sprdnote/com.sprd.sprdnote.NoteActivity"
    				};
    		 List<AppInfo> sortAppsArray = new ArrayList<>();
    		    for(int i=0; i<appPkgAndClsName.length; i++){		
    				String pkgName = appPkgAndClsName[i].split("/")[0];
    				String clsName = appPkgAndClsName[i].split("/")[1];
    				for(int j=0; j<mApps.size(); j++){
    					AppInfo appinfo = mApps.get(j);
    					String packagename = appinfo.componentName.getPackageName();
    					String classname = appinfo.componentName.getClassName();
    					if(packagename.equals(pkgName)&&classname.equals(clsName)){	
    						sortAppsArray.add(appinfo);
    						break;
    					}
    				}
    			}
    			return sortAppsArray;
    	}
        private List<AppInfo> getOutArrayApps(){
    		 List<AppInfo> outAppsArray = new ArrayList<>(mApps);
    		 outAppsArray.removeAll(getSortArrayApps());
    		 return outAppsArray;
    	}
    	/**
    	*	将需要排序的和不需排序的列表传入，当排序列表放在前面
    	*/
        private void sortNewAllApps(){
    	 List<AppInfo> sortAppsArray = new ArrayList<>();
    	 List<AppInfo> arrayApps = getSortArrayApps();
    	 List<AppInfo> notSortArrayApps = getOutArrayApps();
    		
    	 if(arrayApps.size()>0 && notSortArrayApps.size()>0){
    			//对集合2进行排序
    			Collections.sort(notSortArrayApps, mAppNameComparator);
    			boolean appIsBefore = mContext.getResources().getBoolean(R.bool.sort_is_before);
    			if(!appIsBefore){
    				//合并集合1和2,并将集合1放在后面
    				notSortArrayApps.addAll(arrayApps);
    				sortAppsArray = notSortArrayApps;
    			}else{
    				//合并集合1和2,并将集合1放在前面
    				arrayApps.addAll(notSortArrayApps);
    				sortAppsArray = arrayApps;
    			}
    			//将集合mApps清空				
    			mApps.clear();
    			//将集合1和集合2合并后放入mApps集合
    			mApps.addAll(sortAppsArray);
    		}
    	}
```

在上述的 AlphabeticalAppsList.java 的相关源码中分析得知，在 onAppsUpdated() 中的源码，主要负责管理 app 更新的时候，进行桌面 app 列表的更新，所以说在这里通过添加 app 列表排序的相关方法，然后从小对 apps 集合进行排序，就实现了 app 排序的功能

3.2.2 在 onAppsUpdated() 增加排序方法
------------------------------

```
    public void onAppsUpdated() {
            // Sort the list of apps
            mApps.clear();
     
            for (AppInfo app : mAllAppsStore.getApps()) {
                if (mItemFilter == null || mItemFilter.matches(app, null) || hasFilter()) {
                    mApps.add(app);
                }
            }
     
            // 注释掉按名称排序
            //Collections.sort(mApps, mAppNameComparator);
     
            // As a special case for some languages (currently only Simplified Chinese), we may need to
            // coalesce sections
            Locale curLocale = mLauncher.getResources().getConfiguration().locale;
            boolean localeRequiresSectionSorting = curLocale.equals(Locale.SIMPLIFIED_CHINESE);
            if (localeRequiresSectionSorting) {
                // Compute the section headers. We use a TreeMap with the section name comparator to
                // ensure that the sections are ordered when we iterate over it later
                TreeMap<String, ArrayList<AppInfo>> sectionMap = new TreeMap<>(new LabelComparator());
                for (AppInfo info : mApps) {
                    // Add the section to the cache
                    String sectionName = getAndUpdateCachedSectionName(info.title);
     
                    // Add it to the mapping
                    ArrayList<AppInfo> sectionApps = sectionMap.get(sectionName);
                    if (sectionApps == null) {
                        sectionApps = new ArrayList<>();
                        sectionMap.put(sectionName, sectionApps);
                    }
                    sectionApps.add(info);
                }
     
                // Add each of the section apps to the list in order
                mApps.clear();
                for (Map.Entry<String, ArrayList<AppInfo>> entry : sectionMap.entrySet()) {
                    mApps.addAll(entry.getValue());
                }
            } else {
                // Just compute the section headers for use below
                for (AppInfo info : mApps) {
                    // Add the section to the cache
                    getAndUpdateCachedSectionName(info.title);
                }
            }
     
            LauncherAppMonitor.getInstance(mLauncher).onAllAppsListUpdated(mApps);
     
            //add core start 增加排序的代码
    		boolean appIsSort = mContext.getResources().getBoolean(R.bool.apps_is_sort);
    		if(appIsSort){
    			sortNewAllApps();
    		}else {
    			Collections.sort(mApps, mAppNameComparator);
    		}
    		// Recompose the set of adapter items from the current set of apps
     
         // add core end
     
            updateAdapterItems();
        }
```

在 AlphabeticalAppsList.java 的上述核心方法中，分析得知，在上述的 onAppsUpdated() 中是具体  
对双层抽屉模式排序的相关核心代码块，所以可以在这里添加上述的增加的排序的核心方法，然后  
绑定 workspace 的桌面 app 排序列表，就实现了 app 列表双层排序的功能，  
      packages\apps\Launcher3\res\values\config.xml 中增加的属性  
        <bool >true</bool>  
        <bool >true</bool>

3.3 BaseModelUpdateTask.java 关于排序的相关修改
--------------------------------------

BaseModelUpdateTask，实际也是 Runnable,PMS 安装应用后更新 Launcher 图标及逻辑的实现类  
处理了应用安装、更新和卸载等过程，我们这里只对新应用安装做下分析，在这里的 run 方法中  
来更新绑定 app 的列表，具体功能如下

```
     @Override
        public final void run() {
            if (!mModel.isModelLoaded() && !mIgnoreLoaded) {
                if (DEBUG_TASKS) {
                    Log.d(TAG, "Ignoring model task since loader is pending=" + this);
                }
                // 注释掉这里
                // Loader has not yet run.
                //return;
            }
            execute(mApp, mDataModel, mAllAppsList);
        }
```

在上述的 BaseModelUpdateTask.java 中的上述相关源码分析得知，在  
通过上述的几个类来的相关代码的修改，可以发现最终在开机进入 Launcher3 的时候修改了排序方式，  
同时在安装或卸载 app 的时候，app 列表更新的时候，同样也是按顺序排序显示 app 列表，最终实现了这个功能排序
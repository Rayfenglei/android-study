> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/127043940)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.Launcher3 app 列表页桌面图标按安装时间排序的相关代码](#t1)

[3.Launcher3 app 列表页桌面图标按安装时间排序相关功能分析](#t2)

 [3.1 AllAppsRecyclerView.java 关于获取 app 列表的功能分析](#t3)

[3.2 AlphabeticalAppsList 关于排序的方法功能分析](#t4)

[3.3 AppNameComparator 相关排序方法分析](#t5)

1. 概述
-----

  在对 Launcher3 进行功能开发时，系统默认的 app 列表页排序是安装 app 名称进行排序的，由于功能的需要要求按照 app 安装时间进行排序，这就需要找到相关的排序地方，进行排序方式的修改就能完成这个功能

2.Launcher3 app 列表页桌面图标按安装时间排序的相关代码
-----------------------------------

```
  packages/apps/Launcher3/src/com/android/launcher3/allapps/AllAppsRecyclerView.java
  packages/apps/Launcher3/src/com/android/launcher3/allapps/AlphabeticalAppsList.java
  packages/apps/Launcher3/src/com/android/launcher3/allapps/AppInfoComparator.java
```

3.Launcher3 app 列表页桌面图标按安装时间排序相关功能分析
------------------------------------

###   3.1 AllAppsRecyclerView.java 关于获取 app 列表的功能分析

```
   public class AllAppsRecyclerView extends BaseRecyclerView implements LogContainerProvider {
  
      private AlphabeticalAppsList mApps;
      private final int mNumAppsPerRow;
  
      // The specific view heights that we use to calculate scroll
      private final SparseIntArray mViewHeights = new SparseIntArray();
      private final SparseIntArray mCachedScrollPositions = new SparseIntArray();
      private final AllAppsFastScrollHelper mFastScrollHelper;
  
      // The empty-search result background
      private AllAppsBackgroundDrawable mEmptySearchBackground;
      private int mEmptySearchBackgroundTopOffset;
  
      private ArrayList<View> mAutoSizedOverlays = new ArrayList<>();
  
      public AllAppsRecyclerView(Context context) {
          this(context, null);
      }
  
      public AllAppsRecyclerView(Context context, AttributeSet attrs) {
          this(context, attrs, 0);
      }
  
      public AllAppsRecyclerView(Context context, AttributeSet attrs, int defStyleAttr) {
          this(context, attrs, defStyleAttr, 0);
      }
	  public void setApps(AlphabeticalAppsList apps) {
         mApps = apps;
     }
 
     public AlphabeticalAppsList getApps() {
         return mApps;
      }
```

从上面的 setApps 和 getApps 发现 app 列表页的排序是通过 AlphabeticalApps[List 排序](https://so.csdn.net/so/search?q=List%E6%8E%92%E5%BA%8F&spm=1001.2101.3001.7020)以后的集合来作为  
app 列表页从新参与显示的 所以要从 AlphabeticalAppsList 来找到相关排序的依据在进行修改排序方法

### 3.2 AlphabeticalAppsList 关于排序的方法功能分析

```
  public class AlphabeticalAppsList implements AllAppsStore.OnUpdateListener {
 
     public static final String TAG = "AlphabeticalAppsList";
 
     private static final int FAST_SCROLL_FRACTION_DISTRIBUTE_BY_ROWS_FRACTION = 0;
     private static final int FAST_SCROLL_FRACTION_DISTRIBUTE_BY_NUM_SECTIONS = 1;
 
     private final int mFastScrollDistributionMode = FAST_SCROLL_FRACTION_DISTRIBUTE_BY_NUM_SECTIONS;
 // The set of apps from the system
      private final List<AppInfo> mApps = new ArrayList<>();
      private final AllAppsStore mAllAppsStore;
  
      // The set of filtered apps with the current filter
      private final List<AppInfo> mFilteredApps = new ArrayList<>();
      // The current set of adapter items
      private final ArrayList<AdapterItem> mAdapterItems = new ArrayList<>();
      // The set of sections that we allow fast-scrolling to (includes non-merged sections)
      private final List<FastScrollSectionInfo> mFastScrollerSections = new ArrayList<>();
      // Is it the work profile app list.
      private final boolean mIsWork;
  
      // The of ordered component names as a result of a search query
      private ArrayList<ComponentKey> mSearchResults;
      private AllAppsGridAdapter mAdapter;
      private AppInfoComparator mAppNameComparator;
      private final int mNumAppsPerRow;
      private int mNumAppRowsInAdapter;
      private ItemInfoMatcher mItemFilter;
  
      public AlphabeticalAppsList(Context context, AllAppsStore appsStore, boolean isWork) {
          mAllAppsStore = appsStore;
          mLauncher = BaseDraggingActivity.fromContext(context);
          mAppNameComparator = new AppInfoComparator(context);
          mIsWork = isWork;
          mNumAppsPerRow = mLauncher.getDeviceProfile().inv.numColumns;
          mAllAppsStore.addUpdateListener(this);
      }
 
 /**
       * Updates internals when the set of apps are updated.
       */
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
                  String sectionName = info.sectionName;
  
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
          }
  
          // Recompose the set of adapter items from the current set of apps
          updateAdapterItems();
      }
```

mApps 用于保存查询到的所以 app 集合 然后在每次更新 app 列表页时 都会通过 Collections.sort(mApps, mAppNameComparator);  
进行排序后，然后在绑定到 app 适配器显示最终由 updateAdapterItems() 更新列表页  
所以具体的排序是在 mAppNameComparator 进行的所以要分析这里面的排序方法

### 3.3 AppNameComparator 相关排序方法分析

```
   public class AppInfoComparator implements Comparator<AppInfo> {
  
      private final UserCache mUserManager;
      private final UserHandle mMyUser;
      private final LabelComparator mLabelComparator;
  
      public AppInfoComparator(Context context) {
          mUserManager = UserCache.INSTANCE.get(context);
          mMyUser = Process.myUserHandle();
          mLabelComparator = new LabelComparator();
      }
  
      @Override
      public int compare(AppInfo a, AppInfo b) {
          // Order by the title in the current locale
          int result = mLabelComparator.compare(a.title.toString(), b.title.toString());
          if (result != 0) {
              return result;
          }
  
          // If labels are same, compare component names
          result = a.componentName.compareTo(b.componentName);
          if (result != 0) {
              return result;
          }
  
          if (mMyUser.equals(a.user)) {
              return -1;
          } else {
              Long aUserSerial = mUserManager.getSerialNumberForUser(a.user);
              Long bUserSerial = mUserManager.getSerialNumberForUser(b.user);
              return aUserSerial.compareTo(bUserSerial);
          }
      }
  }
```

从代码中可以看到 compare(AppInfo a, AppInfo b) 就是对两个 app 名称进行比较排序  
比较名称开头字母是否相同想让然后比较 user  
所以具体修改排序方式就在这里  
具体修改为:

```
在AppInfoComparator的主要代码如下:
+import android.content.pm.PackageInfo;
+import android.content.pm.PackageManager;
@Override
public int compare(AppInfo a, AppInfo b) {
// Order by the title in the current locale
// 注释掉原来的排序方式 添加新的排序方式如下
/*int result = mLabelComparator.compare(a.title.toString(), b.title.toString());if (result != 0) {return result;}*/
    
//add code start
    String a_pkgname = a.componentName.getPackageName();
	String b_pkgname = b.componentName.getPackageName();
	int result = getInstallTime(a_pkgname).compareTo(getInstallTime(b_pkgname));
    if (result != 0) {
        return result;
    }
   //add code end
 
    // If labels are same, compare component names
    result = a.componentName.compareTo(b.componentName);
    if (result != 0) {
        return result;
    }
 
    if (mMyUser.equals(a.user)) {
        return -1;
    } else {
        Long aUserSerial = mUserManager.getSerialNumberForUser(a.user);
        Long bUserSerial = mUserManager.getSerialNumberForUser(b.user);
        return aUserSerial.compareTo(bUserSerial);
    }
}
//根据包名获取安装时间
public String getInstallTime(String packageName){
	String installtime ="";
	try {
        PackageManager mPackageManager = mContext.getPackageManager();
        PackageInfo packageInfo = mPackageManager.getPackageInfo(packageName,0);
        installtime = packageInfo.firstInstallTime+"";
        android.util.Log.e("MainActivity","packageName:"+packageName+"--installtime:"+installtime);
    } catch (Exception e) {
        e.printStackTrace();
    }
	return installtime;
}
```
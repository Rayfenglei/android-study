> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126734709)

1. 概述
-----

  在 Launcher3 桌面显示列表中，由于在 app 列表页中，由于有些 app 名称长度有些长，而系统默认显示一行，显示不下就[省略号](https://so.csdn.net/so/search?q=%E7%9C%81%E7%95%A5%E5%8F%B7&spm=1001.2101.3001.7020)显示，由于页面高度有多余的，所以要求显示全 app 名称，这就需要看哪里配置 app 显示行数了

2.Launcher3 中 app 列表页的 app 名称分两行显示的核心代码
---------------------------------------

```
  packages/apps/Launcher3/src/com/android/launcher3/allapps/AllAppsGridAdapter.java
  packages/apps/Launcher3/res/layout/all_apps_icon.xml
```

3.Launcher3 中 app 列表页的 app 名称分两行显示的功能分析以及实现
-------------------------------------------

###   3.1AllAppsGridAdapter.java 相关绑定 app 列表的相关代码分析

```
public class AllAppsGridAdapter extends RecyclerView.Adapter<AllAppsGridAdapter.ViewHolder> {
 
     public static final String TAG = "AppsGridAdapter";
 
     // A normal icon
     public static final int VIEW_TYPE_ICON = 1 << 1;
     // The message shown when there are no filtered results
     public static final int VIEW_TYPE_EMPTY_SEARCH = 1 << 2;
     // The message to continue to a market search when there are no filtered results
     public static final int VIEW_TYPE_SEARCH_MARKET = 1 << 3;
 
     // We use various dividers for various purposes.  They share enough attributes to reuse layouts,
     // but differ in enough attributes to require different view types
 
     // A divider that separates the apps list and the search market button
     public static final int VIEW_TYPE_ALL_APPS_DIVIDER = 1 << 4;
 
     // Common view type masks
     public static final int VIEW_TYPE_MASK_DIVIDER = VIEW_TYPE_ALL_APPS_DIVIDER;
     public static final int VIEW_TYPE_MASK_ICON = VIEW_TYPE_ICON;
 
     /**
      * ViewHolder for each icon.
      */
     public static class ViewHolder extends RecyclerView.ViewHolder {
 
         public ViewHolder(View v) {
             super(v);
         }
     }
 
     /**
      * A subclass of GridLayoutManager that overrides accessibility values during app search.
      */
     public class AppsGridLayoutManager extends GridLayoutManager {
 
         public AppsGridLayoutManager(Context context) {
             super(context, 1, GridLayoutManager.VERTICAL, false);
         }
 
         @Override
         public void onInitializeAccessibilityEvent(AccessibilityEvent event) {
             super.onInitializeAccessibilityEvent(event);
 
             // Ensure that we only report the number apps for accessibility not including other
             // adapter views
             final AccessibilityRecordCompat record = AccessibilityEventCompat
                      .asRecord(event);
              record.setItemCount(mApps.getNumFilteredApps());
              record.setFromIndex(Math.max(0,
                      record.getFromIndex() - getRowsNotForAccessibility(record.getFromIndex())));
              record.setToIndex(Math.max(0,
                      record.getToIndex() - getRowsNotForAccessibility(record.getToIndex())));
          }
  
          @Override
          public int getRowCountForAccessibility(RecyclerView.Recycler recycler,
                  RecyclerView.State state) {
              return super.getRowCountForAccessibility(recycler, state) -
                      getRowsNotForAccessibility(mApps.getAdapterItems().size() - 1);
          }
  
          @Override
          public void onInitializeAccessibilityNodeInfoForItem(RecyclerView.Recycler recycler,
                  RecyclerView.State state, View host, AccessibilityNodeInfoCompat info) {
              super.onInitializeAccessibilityNodeInfoForItem(recycler, state, host, info);
  
              ViewGroup.LayoutParams lp = host.getLayoutParams();
              AccessibilityNodeInfoCompat.CollectionItemInfoCompat cic = info.getCollectionItemInfo();
              if (!(lp instanceof LayoutParams) || (cic == null)) {
                  return;
              }
              LayoutParams glp = (LayoutParams) lp;
              info.setCollectionItemInfo(AccessibilityNodeInfoCompat.CollectionItemInfoCompat.obtain(
                      cic.getRowIndex() - getRowsNotForAccessibility(glp.getViewAdapterPosition()),
                      cic.getRowSpan(),
                      cic.getColumnIndex(),
                      cic.getColumnSpan(),
                      cic.isHeading(),
                      cic.isSelected()));
          }
  
          /**
           * Returns the number of rows before {@param adapterPosition}, including this position
           * which should not be counted towards the collection info.
           */
          private int getRowsNotForAccessibility(int adapterPosition) {
              List<AdapterItem> items = mApps.getAdapterItems();
              adapterPosition = Math.max(adapterPosition, mApps.getAdapterItems().size() - 1);
              int extraRows = 0;
              for (int i = 0; i <= adapterPosition; i++) {
                  if (!isViewType(items.get(i).viewType, VIEW_TYPE_MASK_ICON)) {
                      extraRows++;
                  }
              }
              return extraRows;
          }
      }
  
      /**
       * Helper class to size the grid items.
       */
      public class GridSpanSizer extends GridLayoutManager.SpanSizeLookup {
  
          public GridSpanSizer() {
              super();
              setSpanIndexCacheEnabled(true);
          }
  
          @Override
          public int getSpanSize(int position) {
              if (isIconViewType(mApps.getAdapterItems().get(position).viewType)) {
                  return 1;
              } else {
                  // Section breaks span the full width
                  return mAppsPerRow;
              }
          }
      }
  
      private final BaseDraggingActivity mLauncher;
      private final LayoutInflater mLayoutInflater;
      private final AlphabeticalAppsList mApps;
      private final GridLayoutManager mGridLayoutMgr;
      private final GridSpanSizer mGridSizer;
  
      private final OnClickListener mOnIconClickListener;
      private OnLongClickListener mOnIconLongClickListener = INSTANCE_ALL_APPS;
  
      private int mAppsPerRow;
  
      private OnFocusChangeListener mIconFocusListener;
  
      // The text to show when there are no search results and no market search handler.
      protected String mEmptySearchMessage;
      // The intent to send off to the market app, updated each time the search query changes.
      private Intent mMarketSearchIntent;
  
      public AllAppsGridAdapter(BaseDraggingActivity launcher, LayoutInflater inflater,
              AlphabeticalAppsList apps) {
          Resources res = launcher.getResources();
          mLauncher = launcher;
          mApps = apps;
          mEmptySearchMessage = res.getString(R.string.all_apps_loading_message);
          mGridSizer = new GridSpanSizer();
          mGridLayoutMgr = new AppsGridLayoutManager(launcher);
          mGridLayoutMgr.setSpanSizeLookup(mGridSizer);
          mLayoutInflater = inflater;
  
          mOnIconClickListener = launcher.getItemOnClickListener();
  
          setAppsPerRow(mLauncher.getDeviceProfile().inv.numAllAppsColumns);
      }
  
      public void setAppsPerRow(int appsPerRow) {
          mAppsPerRow = appsPerRow;
          mGridLayoutMgr.setSpanCount(mAppsPerRow);
      }
  
      /**
       * Sets the long click listener for icons
       */
      public void setOnIconLongClickListener(@Nullable OnLongClickListener listener) {
          mOnIconLongClickListener = listener;
      }
  
      public static boolean isDividerViewType(int viewType) {
          return isViewType(viewType, VIEW_TYPE_MASK_DIVIDER);
      }
  
      public static boolean isIconViewType(int viewType) {
          return isViewType(viewType, VIEW_TYPE_MASK_ICON);
      }
  
      public static boolean isViewType(int viewType, int viewTypeMask) {
          return (viewType & viewTypeMask) != 0;
      }
  
      public void setIconFocusListener(OnFocusChangeListener focusListener) {
          mIconFocusListener = focusListener;
      }
  
      /**
       * Sets the last search query that was made, used to show when there are no results and to also
       * seed the intent for searching the market.
       */
      public void setLastSearchQuery(String query) {
          Resources res = mLauncher.getResources();
          mEmptySearchMessage = res.getString(R.string.all_apps_no_search_results, query);
          mMarketSearchIntent = PackageManagerHelper.getMarketSearchIntent(mLauncher, query);
      }
  
      /**
       * Returns the grid layout manager.
       */
      public GridLayoutManager getLayoutManager() {
          return mGridLayoutMgr;
      }
  
      @Override
      public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
          switch (viewType) {
              case VIEW_TYPE_ICON:
                  // 构造BubbleTextView的布局是all_apps_icon
                  BubbleTextView icon = (BubbleTextView) mLayoutInflater.inflate(
                          R.layout.all_apps_icon, parent, false);
                  icon.setOnClickListener(mOnIconClickListener);
                  icon.setOnLongClickListener(mOnIconLongClickListener);
                  icon.setLongPressTimeoutFactor(1f);
                  icon.setOnFocusChangeListener(mIconFocusListener);
  
                  // Ensure the all apps icon height matches the workspace icons in portrait mode.
                  icon.getLayoutParams().height = mLauncher.getDeviceProfile().allAppsCellHeightPx;
                  return new ViewHolder(icon);
              case VIEW_TYPE_EMPTY_SEARCH:
                  return new ViewHolder(mLayoutInflater.inflate(R.layout.all_apps_empty_search,
                          parent, false));
              case VIEW_TYPE_SEARCH_MARKET:
                  View searchMarketView = mLayoutInflater.inflate(R.layout.all_apps_search_market,
                          parent, false);
                  searchMarketView.setOnClickListener(v -> mLauncher.startActivitySafely(
                          v, mMarketSearchIntent, null, AppLaunchTracker.CONTAINER_SEARCH));
                  return new ViewHolder(searchMarketView);
              case VIEW_TYPE_ALL_APPS_DIVIDER:
                  return new ViewHolder(mLayoutInflater.inflate(
                          R.layout.all_apps_divider, parent, false));
              default:
                  throw new RuntimeException("Unexpected view type");
          }
      }
  
      @Override
      public void onBindViewHolder(ViewHolder holder, int position) {
          switch (holder.getItemViewType()) {
              case VIEW_TYPE_ICON:
                  AppInfo info = mApps.getAdapterItems().get(position).appInfo;
                  BubbleTextView icon = (BubbleTextView) holder.itemView;
                  icon.reset();
                  icon.applyFromApplicationInfo(info);
                  break;
              case VIEW_TYPE_EMPTY_SEARCH:
                  TextView emptyViewText = (TextView) holder.itemView;
                  emptyViewText.setText(mEmptySearchMessage);
                  emptyViewText.setGravity(mApps.hasNoFilteredResults() ? Gravity.CENTER :
                          Gravity.START | Gravity.CENTER_VERTICAL);
                  break;
              case VIEW_TYPE_SEARCH_MARKET:
                  TextView searchView = (TextView) holder.itemView;
                  if (mMarketSearchIntent != null) {
                      searchView.setVisibility(View.VISIBLE);
                  } else {
                      searchView.setVisibility(View.GONE);
                  }
                  break;
              case VIEW_TYPE_ALL_APPS_DIVIDER:
                  // nothing to do
                  break;
          }
      }
  
      @Override
      public boolean onFailedToRecycleView(ViewHolder holder) {
          // Always recycle and we will reset the view when it is bound
          return true;
      }
  
      @Override
      public int getItemCount() {
          return mApps.getAdapterItems().size();
      }
  
      @Override
      public int getItemViewType(int position) {
          AlphabeticalAppsList.AdapterItem item = mApps.getAdapterItems().get(position);
          return item.viewType;
      }
  }
```

### AllAppsGridAdapter 负责构造 app 列表每个 app 的 icon 和名称的显示

onCreateViewHolder 构造每个 app 的布局，而 onBindViewHolder(ViewHolder holder, int position) 负责根据不同类型显示不同布局，在在 onBindViewHolder(ViewHolder holder, int position) 中可以看到 BubbleTextView 就是构造 app 列表的图标和名称的类 同样在 all_apps_icon.xml 就是显示 app 的布局文件 所以接下来分析 all_apps_icon.xml 文件

### 3.2 all_apps_icon.xml 的相关布局分析

```
   <com.android.launcher3.BubbleTextView
     xmlns:android="http://schemas.android.com/apk/res/android"
     xmlns:launcher="http://schemas.android.com/apk/res-auto"
     style="@style/BaseIcon"
     android:id="@+id/icon"
     android:layout_width="match_parent"
     android:layout_height="wrap_content"
     android:stateListAnimator="@animator/all_apps_fastscroll_icon_anim"
     launcher:iconDisplay="all_apps"
     launcher:centerVertically="true"
     android:paddingLeft="@dimen/dynamic_grid_cell_padding_x"
     android:paddingRight="@dimen/dynamic_grid_cell_padding_x" />
 
```

而样式就是在 style="@style/BaseIcon" 中接下来看下行数等相关样式  
    <style >  
         <item >1</item>  
     </style>  
从样式中可以看出设置行数为 1  
所以可以修改为：  
    <style >  
       -  <item >1</item>  
       +  <item >2</item>  
     </style>
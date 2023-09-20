> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124739892)

1. 概述
-----

在 11.0 12.0 的产品定制化开发中，系统默认的 Launcher3 在 workspace 第二屏通常都会显示 app 列表 点击进入 app 列表页，长按 app 的 icon 图标会弹出 应用信息 弹窗  
等信息，而由于不需要弹出这些信息，所以要求去掉 app 的 icon 图标的长按功能

2. 屏蔽 Launcher3 桌面 app 图标的长按功能的核心类
----------------------------------

```
packages\apps\Launcher3\src\com\android\launcher3\allapps\AllAppsGridAdapter.java
packages/apps/Launcher3/src/com/android/launcher3/WorkspaceLayoutManager.java

```

3. 屏蔽 Launcher3 桌面 app 图标的长按功能的核心功能分析和实现
----------------------------------------

3.1AllAppsGridAdapter.java 部分屏蔽长按事件
-----------------------------------

路径: packages\apps\Launcher3\src\com\android\launcher3\allapps\AllAppsGridAdapter.java  
主要是负责布局 app 列表  
在 workspace 的 AllAppsGridAdapter 就是绑定 app 的适配器，所以在 AllAppsGridAdapter 中的  
onCreateViewHolder(ViewGroup parent, int viewType) 根据不同类型调用不同布局文件  
VIEW_TYPE_ICON 就是 app 的 icon 图标布局

```
public class AllAppsGridAdapter extends RecyclerView.Adapter<AllAppsGridAdapter.ViewHolder> {
  
      public static final String TAG = "AppsGridAdapter";
public static class ViewHolder extends RecyclerView.ViewHolder {
 
         public ViewHolder(View v) {
             super(v);
         }
     }
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
  @Override
      public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
          switch (viewType) {
              case VIEW_TYPE_ICON:
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

```

其中在 onCreateViewHolder(ViewGroup parent, int viewType) 的  
case VIEW_TYPE_ICON 就是构建 app 图标布局

```
              case VIEW_TYPE_ICON:
                  BubbleTextView icon = (BubbleTextView) mLayoutInflater.inflate(
                          R.layout.all_apps_icon, parent, false);
                  icon.setOnClickListener(mOnIconClickListener);
                  icon.setOnLongClickListener(mOnIconLongClickListener);
                  icon.setLongPressTimeoutFactor(1f);
                  icon.setOnFocusChangeListener(mIconFocusListener);
  
                  // Ensure the all apps icon height matches the workspace icons in portrait mode.
                  icon.getLayoutParams().height = mLauncher.getDeviceProfile().allAppsCellHeightPx;
                  return new ViewHolder(icon);

```

通过上述代码发现  
icon.setOnLongClickListener(mOnIconLongClickListener); 就是 app 的 icon 的长按事件  
就是 app 图标的长按事件 注释掉这部分代码

3.2WorkspaceLayoutManager.java 中 Item 不响应长按事件的修改
------------------------------------------------

```
    default void addInScreenFromBind(View child, ItemInfo info) {
         int x = info.cellX;
         int y = info.cellY;
         if (info.container == LauncherSettings.Favorites.CONTAINER_HOTSEAT
                 || info.container == LauncherSettings.Favorites.CONTAINER_HOTSEAT_PREDICTION) {
             int screenId = info.screenId;
             x = getHotseat().getCellXFromOrder(screenId);
             y = getHotseat().getCellYFromOrder(screenId);
         }
         addInScreen(child, info.container, info.screenId, x, y, info.spanX, info.spanY);
     }
 
     default void addInScreen(View child, ItemInfo info) {
         addInScreen(child, info.container, info.screenId, info.cellX, info.cellY,
                 info.spanX, info.spanY);
     }
 
     default void addInScreen(View child, int container, int screenId, int x, int y,
             int spanX, int spanY) {
         if (container == LauncherSettings.Favorites.CONTAINER_DESKTOP) {
             if (getScreenWithId(screenId) == null) {
                 Log.e(TAG, "Skipping child, screenId " + screenId + " not found");
                 // DEBUGGING - Print out the stack trace to see where we are adding from
                 new Throwable().printStackTrace();
                 return;
             }
         }
         if (screenId == EXTRA_EMPTY_SCREEN_ID) {
             // This should never happen
             throw new RuntimeException("Screen id should not be EXTRA_EMPTY_SCREEN_ID");
         }
 
         final CellLayout layout;
         if (container == LauncherSettings.Favorites.CONTAINER_HOTSEAT
                 || container == LauncherSettings.Favorites.CONTAINER_HOTSEAT_PREDICTION) {
             layout = getHotseat();
 
             // Hide folder title in the hotseat
             if (child instanceof FolderIcon) {
                 ((FolderIcon) child).setTextVisible(false);
             }
         } else {
             // Show folder title if not in the hotseat
             if (child instanceof FolderIcon) {
                 ((FolderIcon) child).setTextVisible(true);
              }
              layout = getScreenWithId(screenId);
          }
  
          ViewGroup.LayoutParams genericLp = child.getLayoutParams();
          CellLayout.LayoutParams lp;
          if (genericLp == null || !(genericLp instanceof CellLayout.LayoutParams)) {
              lp = new CellLayout.LayoutParams(x, y, spanX, spanY);
          } else {
              lp = (CellLayout.LayoutParams) genericLp;
              lp.cellX = x;
              lp.cellY = y;
              lp.cellHSpan = spanX;
              lp.cellVSpan = spanY;
          }
  
          if (spanX < 0 && spanY < 0) {
              lp.isLockedToGrid = false;
          }
  
          // Get the canonical child id to uniquely represent this view in this screen
          ItemInfo info = (ItemInfo) child.getTag();
          int childId = info.getViewId();
  
          boolean markCellsAsOccupied = !(child instanceof Folder);
          if (!layout.addViewToCellLayout(child, -1, childId, lp, markCellsAsOccupied)) {
              // TODO: This branch occurs when the workspace is adding views
              // outside of the defined grid
              // maybe we should be deleting these items from the LauncherModel?
              Log.e(TAG, "Failed to add to item at (" + lp.cellX + "," + lp.cellY + ") to CellLayout");
          }
  
          child.setHapticFeedbackEnabled(false);
          child.setOnLongClickListener(ItemLongClickListener.INSTANCE_WORKSPACE);
          if (child instanceof DropTarget) {
              onAddDropTarget((DropTarget) child);
          }
      }

```

在 WorkspaceLayoutManager.java 中通过 addInScreen(View child, int container, int screenId, int x, int y,  
int spanX, int spanY) 来绑定每一屏的 app 列表，而在这个方法中  
通过 child.setOnLongClickListener(ItemLongClickListener.INSTANCE_WORKSPACE);  
来设置每一个 app 的 icon 长按事件

```
--- a/packages/apps/Launcher3/src/com/android/launcher3/WorkspaceLayoutManager.java
+++ b/packages/apps/Launcher3/src/com/android/launcher3/WorkspaceLayoutManager.java
@@ -127,7 +127,9 @@ public interface WorkspaceLayoutManager {
         }
 
         child.setHapticFeedbackEnabled(false);
-        child.setOnLongClickListener(ItemLongClickListener.INSTANCE_WORKSPACE);
+        //child.setOnLongClickListener(ItemLongClickListener.INSTANCE_WORKSPACE);
         if (child instanceof DropTarget) {
             onAddDropTarget((DropTarget) child);
         }

```
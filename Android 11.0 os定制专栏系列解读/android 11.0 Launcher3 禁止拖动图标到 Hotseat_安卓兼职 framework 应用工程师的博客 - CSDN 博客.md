> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124872379)

1. 概述
-----

在 11.0 系统 Launcher3 进行定制化开发中，对于 hotseat 的开发中，由功能需求要求禁止拖动图标到 Hotseat 的功能，而拖拽也是在 workspace.java 中处理的  
接下来就从 workspace.java 开始找解决的办法

2.Launcher3 禁止拖动图标到 Hotseat 相关代码分析
----------------------------------

```
packages/apps/Launcher3/src/com/android/launcher3/DropTarget.java
/packages/apps/Launcher3/src/com/android/launcher3/Workspace.java

```

3.Launcher3 禁止拖动图标到 Hotseat 功能分析和实现
-----------------------------------

3.1DropTarget.java 相关拖拽的接口
--------------------------

```
public interface DropTarget {
 
     class DragObject {        
	  void onDrop(DragObject dragObject, DragOptions options);
  
      void onDragEnter(DragObject dragObject);
  
      void onDragOver(DragObject dragObject);
  
      void onDragExit(DragObject dragObject);
  
      /**
       * Check if a drop action can occur at, or near, the requested location.
       * This will be called just before onDrop.
       * @return True if the drop will be accepted, false otherwise.
       */
      boolean acceptDrop(DragObject dragObject);
  
      void prepareAccessibilityDrop();
  
      // These methods are implemented in Views
      void getHitRectRelativeToDragLayer(Rect outRect);
  }
  

```

onDrop onDragEnter onDragOver onDragExit 都是相关拖拽的方法  
接下来看 WorkSpace.java 的相关方法

3.2 WorkSpace.java 的相关拖拽方法分析
----------------------------

首选来看 onDrop 拖拽事件都是从这里开始的

```
public void onDrop(final DragObject d, DragOptions options) {
        mDragViewVisualCenter = d.getVisualCenter(mDragViewVisualCenter);
        CellLayout dropTargetLayout = mDropToLayout;

        // We want the point to be mapped to the dragTarget.
        if (dropTargetLayout != null) {
            mapPointFromDropLayout(dropTargetLayout, mDragViewVisualCenter);
        }

        boolean droppedOnOriginalCell = false;

        int snapScreen = -1;
        boolean resizeOnDrop = false;
        if (d.dragSource != this || mDragInfo == null) {
            final int[] touchXY = new int[] { (int) mDragViewVisualCenter[0],
                    (int) mDragViewVisualCenter[1] };
            onDropExternal(touchXY, dropTargetLayout, d);
        } else {
            AbstractFloatingView.closeOpenViews(mLauncher, false, AbstractFloatingView.TYPE_WIDGET_RESIZE_FRAME);
            final View cell = mDragInfo.cell;
            boolean droppedOnOriginalCellDuringTransition = false;
            Runnable onCompleteRunnable = null;
           // 判断拖拽的目标区域


           if (dropTargetLayout != null && !d.cancelled) {
                 // Move internally
                 boolean hasMovedLayouts = (getParentCellLayoutForView(cell) != dropTargetLayout);
                 boolean hasMovedIntoHotseat = mLauncher.isHotseatLayout(dropTargetLayout);
                 int container = hasMovedIntoHotseat ?
                        LauncherSettings.Favorites.CONTAINER_HOTSEAT :
                        LauncherSettings.Favorites.CONTAINER_DESKTOP;
                int screenId = (mTargetCell[0] < 0) ?
                        mDragInfo.screenId : getIdForScreen(dropTargetLayout);
                int spanX = mDragInfo != null ? mDragInfo.spanX : 1;
                int spanY = mDragInfo != null ? mDragInfo.spanY : 1;
                // First we find the cell nearest to point at which the item is
                // dropped, without any consideration to whether there is an item there.

                mTargetCell = findNearestArea((int) mDragViewVisualCenter[0], (int)
                        mDragViewVisualCenter[1], spanX, spanY, dropTargetLayout, mTargetCell);
                float distance = dropTargetLayout.getDistanceFromCell(mDragViewVisualCenter[0],
                        mDragViewVisualCenter[1], mTargetCell);

                // If the item being dropped is a shortcut and the nearest drop
                // cell also contains a shortcut, then create a folder with the two shortcuts.
                if (createUserFolderIfNecessary(cell, container,
                        dropTargetLayout, mTargetCell, distance, false, d.dragView) ||
                        addToExistingFolderIfNecessary(cell, dropTargetLayout, mTargetCell,
                                distance, d, false)) {
                    mLauncher.getStateManager().goToState(NORMAL, SPRING_LOADED_EXIT_DELAY);
                    return;
                }

                // Aside from the special case where we're dropping a shortcut onto a shortcut,
                // we need to find the nearest cell location that is vacant
                ItemInfo item = d.dragInfo;
                int minSpanX = item.spanX;
                int minSpanY = item.spanY;
                if (item.minSpanX > 0 && item.minSpanY > 0) {
                    minSpanX = item.minSpanX;
                    minSpanY = item.minSpanY;
                }

                droppedOnOriginalCell = item.screenId == screenId && item.container == container
                        && item.cellX == mTargetCell[0] && item.cellY == mTargetCell[1];
                droppedOnOriginalCellDuringTransition = droppedOnOriginalCell && mIsSwitchingState;

                // When quickly moving an item, a user may accidentally rearrange their
                // workspace. So instead we move the icon back safely to its original position.
                boolean returnToOriginalCellToPreventShuffling = !isFinishedSwitchingState()
                        && !droppedOnOriginalCellDuringTransition && !dropTargetLayout
                        .isRegionVacant(mTargetCell[0], mTargetCell[1], spanX, spanY);
                int[] resultSpan = new int[2];
                if (returnToOriginalCellToPreventShuffling) {
                    mTargetCell[0] = mTargetCell[1] = -1;
                } else {
                    mTargetCell = dropTargetLayout.performReorder((int) mDragViewVisualCenter[0],
                            (int) mDragViewVisualCenter[1], minSpanX, minSpanY, spanX, spanY, cell,
                            mTargetCell, resultSpan, CellLayout.MODE_ON_DROP);
                }

                boolean foundCell = mTargetCell[0] >= 0 && mTargetCell[1] >= 0;

                // if the widget resizes on drop
                if (foundCell && (cell instanceof AppWidgetHostView) &&
                        (resultSpan[0] != item.spanX || resultSpan[1] != item.spanY)) {
                    resizeOnDrop = true;
                    item.spanX = resultSpan[0];
                    item.spanY = resultSpan[1];
                    AppWidgetHostView awhv = (AppWidgetHostView) cell;
                    AppWidgetResizeFrame.updateWidgetSizeRanges(awhv, mLauncher, resultSpan[0],
                            resultSpan[1]);
                }

                if (foundCell) {
                    if (getScreenIdForPageIndex(mCurrentPage) != screenId && !hasMovedIntoHotseat) {
                        snapScreen = getPageIndexForScreenId(screenId);
                        snapToPage(snapScreen);
                    }

                    final ItemInfo info = (ItemInfo) cell.getTag();
                    if (hasMovedLayouts) {
                        // Reparent the view
                        CellLayout parentCell = getParentCellLayoutForView(cell);
                        if (parentCell != null) {
                            parentCell.removeView(cell);
                        } else if (FeatureFlags.IS_DOGFOOD_BUILD) {
                            throw new NullPointerException("mDragInfo.cell has null parent");
                        }
                        addInScreen(cell, container, screenId, mTargetCell[0], mTargetCell[1],
                                info.spanX, info.spanY);
                    }

                    // update the item's position after drop
                    CellLayout.LayoutParams lp = (CellLayout.LayoutParams) cell.getLayoutParams();
                    lp.cellX = lp.tmpCellX = mTargetCell[0];
                    lp.cellY = lp.tmpCellY = mTargetCell[1];
                    lp.cellHSpan = item.spanX;
                    lp.cellVSpan = item.spanY;
                    lp.isLockedToGrid = true;

                    if (container != LauncherSettings.Favorites.CONTAINER_HOTSEAT &&
                            cell instanceof LauncherAppWidgetHostView) {
                        final CellLayout cellLayout = dropTargetLayout;
                        // We post this call so that the widget has a chance to be placed
                        // in its final location

                        final LauncherAppWidgetHostView hostView = (LauncherAppWidgetHostView) cell;
                        AppWidgetProviderInfo pInfo = hostView.getAppWidgetInfo();
                        if (pInfo != null && pInfo.resizeMode != AppWidgetProviderInfo.RESIZE_NONE
                                && !d.accessibleDrag) {
                            onCompleteRunnable = new Runnable() {
                                public void run() {
                                    if (!isPageInTransition()) {
                                        AppWidgetResizeFrame.showForWidget(hostView, cellLayout);
                                    }
                                }
                            };
                        }
                    }

                    mLauncher.getModelWriter().modifyItemInDatabase(info, container, screenId,
                            lp.cellX, lp.cellY, item.spanX, item.spanY);
                } else {
                    if (!returnToOriginalCellToPreventShuffling) {
                        onNoCellFound(dropTargetLayout);
                    }

                    // If we can't find a drop location, we return the item to its original position
                    CellLayout.LayoutParams lp = (CellLayout.LayoutParams) cell.getLayoutParams();
                    mTargetCell[0] = lp.cellX;
                    mTargetCell[1] = lp.cellY;
                    if (cell.getParent() != null) {
                        CellLayout layout = (CellLayout) cell.getParent().getParent();
                        layout.markCellsAsOccupiedForView(cell);
                    }
                }
            }

```

从 阅读 onDrop 的源码可以发现  
在 对 dropTargetLayout 进行判断拖拽到哪个区域 如果加个判断 不拖拽到 Hotseat 区域 就可以了

具体方法如下：

```
diff --git a/packages/apps/Launcher3/src/com/android/launcher3/Workspace.java b/packages/apps/Launcher3/src/com/android/launcher3/Workspace.java
index 924b821e00..c803d29328 100755
--- a/packages/apps/Launcher3/src/com/android/launcher3/Workspace.java
+++ b/packages/apps/Launcher3/src/com/android/launcher3/Workspace.java
@@ -1876,8 +1876,8 @@ public class Workspace extends CyclePagedView<WorkspaceBasePageIndicator>
    public void onDrop(final DragObject d, DragOptions options) {
        mDragViewVisualCenter = d.getVisualCenter(mDragViewVisualCenter);
        CellLayout dropTargetLayout = mDropToLayout;

        // We want the point to be mapped to the dragTarget.
        if (dropTargetLayout != null) {
            mapPointFromDropLayout(dropTargetLayout, mDragViewVisualCenter);
        }

        boolean droppedOnOriginalCell = false;

        int snapScreen = -1;
        boolean resizeOnDrop = false;
        if (d.dragSource != this || mDragInfo == null) {
            final int[] touchXY = new int[] { (int) mDragViewVisualCenter[0],
                    (int) mDragViewVisualCenter[1] };
            onDropExternal(touchXY, dropTargetLayout, d);
        } else {
            AbstractFloatingView.closeOpenViews(mLauncher, false, AbstractFloatingView.TYPE_WIDGET_RESIZE_FRAME);
            final View cell = mDragInfo.cell;
            boolean droppedOnOriginalCellDuringTransition = false;
            Runnable onCompleteRunnable = null;
-
-            if (dropTargetLayout != null && !d.cancelled) {
+            //motify 2021.7.5
+            if (dropTargetLayout != null && !d.cancelled && !mLauncher.isHotseatLayout(dropTargetLayout)) {
                 // Move internally
                 boolean hasMovedLayouts = (getParentCellLayoutForView(cell) != dropTargetLayout);
                 boolean hasMovedIntoHotseat = mLauncher.isHotseatLayout(dropTargetLayout);
                 int container = hasMovedIntoHotseat ?
                        LauncherSettings.Favorites.CONTAINER_HOTSEAT :
                        LauncherSettings.Favorites.CONTAINER_DESKTOP;
                int screenId = (mTargetCell[0] < 0) ?
                        mDragInfo.screenId : getIdForScreen(dropTargetLayout);
                int spanX = mDragInfo != null ? mDragInfo.spanX : 1;
                int spanY = mDragInfo != null ? mDragInfo.spanY : 1;
                // First we find the cell nearest to point at which the item is
                // dropped, without any consideration to whether there is an item there.

                mTargetCell = findNearestArea((int) mDragViewVisualCenter[0], (int)
                        mDragViewVisualCenter[1], spanX, spanY, dropTargetLayout, mTargetCell);
                float distance = dropTargetLayout.getDistanceFromCell(mDragViewVisualCenter[0],
                        mDragViewVisualCenter[1], mTargetCell);

                // If the item being dropped is a shortcut and the nearest drop
                // cell also contains a shortcut, then create a folder with the two shortcuts.
                if (createUserFolderIfNecessary(cell, container,
                        dropTargetLayout, mTargetCell, distance, false, d.dragView) ||
                        addToExistingFolderIfNecessary(cell, dropTargetLayout, mTargetCell,
                                distance, d, false)) {
                    mLauncher.getStateManager().goToState(NORMAL, SPRING_LOADED_EXIT_DELAY);
                    return;
                }

                // Aside from the special case where we're dropping a shortcut onto a shortcut,
                // we need to find the nearest cell location that is vacant
                ItemInfo item = d.dragInfo;
                int minSpanX = item.spanX;
                int minSpanY = item.spanY;
                if (item.minSpanX > 0 && item.minSpanY > 0) {
                    minSpanX = item.minSpanX;
                    minSpanY = item.minSpanY;
                }

                droppedOnOriginalCell = item.screenId == screenId && item.container == container
                        && item.cellX == mTargetCell[0] && item.cellY == mTargetCell[1];
                droppedOnOriginalCellDuringTransition = droppedOnOriginalCell && mIsSwitchingState;

                // When quickly moving an item, a user may accidentally rearrange their
                // workspace. So instead we move the icon back safely to its original position.
                boolean returnToOriginalCellToPreventShuffling = !isFinishedSwitchingState()
                        && !droppedOnOriginalCellDuringTransition && !dropTargetLayout
                        .isRegionVacant(mTargetCell[0], mTargetCell[1], spanX, spanY);
                int[] resultSpan = new int[2];
                if (returnToOriginalCellToPreventShuffling) {
                    mTargetCell[0] = mTargetCell[1] = -1;
                } else {
                    mTargetCell = dropTargetLayout.performReorder((int) mDragViewVisualCenter[0],
                            (int) mDragViewVisualCenter[1], minSpanX, minSpanY, spanX, spanY, cell,
                            mTargetCell, resultSpan, CellLayout.MODE_ON_DROP);
                }

                boolean foundCell = mTargetCell[0] >= 0 && mTargetCell[1] >= 0;

                // if the widget resizes on drop
                if (foundCell && (cell instanceof AppWidgetHostView) &&
                        (resultSpan[0] != item.spanX || resultSpan[1] != item.spanY)) {
                    resizeOnDrop = true;
                    item.spanX = resultSpan[0];
                    item.spanY = resultSpan[1];
                    AppWidgetHostView awhv = (AppWidgetHostView) cell;
                    AppWidgetResizeFrame.updateWidgetSizeRanges(awhv, mLauncher, resultSpan[0],
                            resultSpan[1]);
                }

                if (foundCell) {
                    if (getScreenIdForPageIndex(mCurrentPage) != screenId && !hasMovedIntoHotseat) {
                        snapScreen = getPageIndexForScreenId(screenId);
                        snapToPage(snapScreen);
                    }

                    final ItemInfo info = (ItemInfo) cell.getTag();
                    if (hasMovedLayouts) {
                        // Reparent the view
                        CellLayout parentCell = getParentCellLayoutForView(cell);
                        if (parentCell != null) {
                            parentCell.removeView(cell);
                        } else if (FeatureFlags.IS_DOGFOOD_BUILD) {
                            throw new NullPointerException("mDragInfo.cell has null parent");
                        }
                        addInScreen(cell, container, screenId, mTargetCell[0], mTargetCell[1],
                                info.spanX, info.spanY);
                    }

                    // update the item's position after drop
                    CellLayout.LayoutParams lp = (CellLayout.LayoutParams) cell.getLayoutParams();
                    lp.cellX = lp.tmpCellX = mTargetCell[0];
                    lp.cellY = lp.tmpCellY = mTargetCell[1];
                    lp.cellHSpan = item.spanX;
                    lp.cellVSpan = item.spanY;
                    lp.isLockedToGrid = true;

                    if (container != LauncherSettings.Favorites.CONTAINER_HOTSEAT &&
                            cell instanceof LauncherAppWidgetHostView) {
                        final CellLayout cellLayout = dropTargetLayout;
                        // We post this call so that the widget has a chance to be placed
                        // in its final location

                        final LauncherAppWidgetHostView hostView = (LauncherAppWidgetHostView) cell;
                        AppWidgetProviderInfo pInfo = hostView.getAppWidgetInfo();
                        if (pInfo != null && pInfo.resizeMode != AppWidgetProviderInfo.RESIZE_NONE
                                && !d.accessibleDrag) {
                            onCompleteRunnable = new Runnable() {
                                public void run() {
                                    if (!isPageInTransition()) {
                                        AppWidgetResizeFrame.showForWidget(hostView, cellLayout);
                                    }
                                }
                            };
                        }
                    }

                    mLauncher.getModelWriter().modifyItemInDatabase(info, container, screenId,
                            lp.cellX, lp.cellY, item.spanX, item.spanY);
                } else {
                    if (!returnToOriginalCellToPreventShuffling) {
                        onNoCellFound(dropTargetLayout);
                    }

                    // If we can't find a drop location, we return the item to its original position
                    CellLayout.LayoutParams lp = (CellLayout.LayoutParams) cell.getLayoutParams();
                    mTargetCell[0] = lp.cellX;
                    mTargetCell[1] = lp.cellY;
                    if (cell.getParent() != null) {
                        CellLayout layout = (CellLayout) cell.getParent().getParent();
                        layout.markCellsAsOccupiedForView(cell);
                    }
                }
            }

```

```
diff --git a/packages/apps/Launcher3/src/com/android/launcher3/Launcher.java b/packages/apps/Launcher3/src/com/android/launcher3/Launcher.java
index 78a002a792…b5d0ae2338 100755
— a/packages/apps/Launcher3/src/com/android/launcher3/Launcher.java
+++ b/packages/apps/Launcher3/src/com/android/launcher3/Launcher.java
@@ -2045,7 +2045,7 @@ public class Launcher extends BaseDraggingActivity implements LauncherExterns,
// 判断是否是hotseat区域

     boolean isHotseatLayout(View layout) {
         // TODO: Remove this method
-        return false;
+        return mHotseat != null && (layout == mHotseat);
     }

```
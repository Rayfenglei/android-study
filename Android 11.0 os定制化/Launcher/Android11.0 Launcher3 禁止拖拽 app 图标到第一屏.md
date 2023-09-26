> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124870932)

### 1. 概述

在 11.0 进行定制化开发 Launcher3 中，会对 Launcher3 做些要求，比如现在的需求就是 Launcher3 第一屏的图标固定，不让其他屏的图标拖动到  
第一屏所以说这个需求和 禁止拖拽图标到 Hotseat 类似，也是从 WorkSpace.java 里面寻找解决方案

### 2.Launcher3 禁止拖拽 app 图标到第一屏相关代码

```
packages/apps/Launcher3/src/com/android/launcher3/DropTarget.java
/packages/apps/Launcher3/src/com/android/launcher3/Workspace.java

```

### 3.Launcher3 禁止拖拽 app 图标到第一屏相关功能分析

### 3.1DropTarget.java 相关拖拽的接口分析

```
public interface DropTarget {
  
      class DragObject {
          public int x = -1;
          public int y = -1;
  
          /** X offset from the upper-left corner of the cell to where we touched.  */
          public int xOffset = -1;
  
          /** Y offset from the upper-left corner of the cell to where we touched.  */
          public int yOffset = -1;
  
          /** This indicates whether a drag is in final stages, either drop or cancel. It
           * differentiates onDragExit, since this is called when the drag is ending, above
           * the current drag target, or when the drag moves off the current drag object.
           */
          public boolean dragComplete = false;       
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

在 DropTarget 中调用内部类 DragObject 中的 onDragEnter(DragObject dragObject)  
onDragOver(DragObject dragObject)  
onDragExit(DragObject dragObject) 等接口实现对拖拽过程的监听

### 3.2 WorkSpace.java 关于拖拽过程的相关代码分析

从上述拖拽接口的拖拽方法分析得知  
先来看下 WorkSpace.java 的 onDrop 方法

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



        
            if (dropTargetLayout != null && !d.cancelled ) {
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

```

同样的也是 dropTargetLayout 来进行判断 拖动到哪个区域 而通过 Workspace.java 源码我们知道，getIdForScreen(dropTargetLayout)!=0 判断拖拽到哪一屏  
的拖拽图标  
mTargetCell[1] 判断拖拽到哪一行 所以也是在这里修改，来解决问题

修改如下:

```
--- a/packages/apps/Launcher3/src/com/android/launcher3/Workspace.java
+++ b/packages/apps/Launcher3/src/com/android/launcher3/Workspace.java
@@ -1876,8 +1876,9 @@ public class Workspace extends CyclePagedView<WorkspaceBasePageIndicator>
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



             //motify 2021.7.5
-            if (dropTargetLayout != null && !d.cancelled && !mLauncher.isHotseatLayout(dropTargetLayout)) {
+            if (dropTargetLayout != null && !d.cancelled && !mLauncher.isHotseatLayout(dropTargetLayout) && getIdForScreen(dropTargetLayout)!=0 && mTargetCell[1]!=0) {
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

```
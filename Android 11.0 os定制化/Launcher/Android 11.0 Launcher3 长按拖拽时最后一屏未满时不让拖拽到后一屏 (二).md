> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124828322)

### 1. 概述

在 11.0 定制化开发中, 如果专门适配老年机的时候，这时客户提出要求，如果最后一屏未满时，不让拖拽到后面一屏的空屏中，等当前屏填满了以后，才能拖到下一屏的功能，所以要从 workspace 的拖拽类开始着手分析

### 2. 长按拖拽时最后一屏未满时不让拖拽到后一屏 (二) 核心类

```
packages/apps/Launcher3/src/com/android/launcher3/DropTarget.java
/packages/apps/Launcher3/src/com/android/launcher3/Workspace.java

```

### 3. 长按拖拽时最后一屏未满时不让拖拽到后一屏 (二) 核心功能分析

分析：  
想了解当前是哪一屏和当前屏 item 个数  
请进  
接下来当知道了当前是哪一屏和当前屏 Item 个数后

### 3.1DropTarget.java 相关拖拽的定义接口

```
public interface DropTarget {
 
     class DragObject {
         public int x = -1;
         public int y = -1;
 
         /** X offset from the upper-left corner of the cell to where we touched.  */
         public int xOffset = -1;
 
         /** Y offset from the upper-left corner of the cell to where we touched.  */
         public int yOffset = -1;

          boolean isDropEnabled();

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

上述的 DropTarget 中定义了拖拽的相关接口在 workspace 中实现这些接口，然后拖拽相关的工作都是在这里面做的

### 3.2 WorkSpace.java 相关拖拽的功能分析

在 WorkSpace.java 中的 onDrop 中就可以来做判断了

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

        // 当判断dropTargetLayout不为null并且d.cancelled为false时才执行拖拽到哪一屏动作 否则就会返回到原来的位置 所以就要在这里增加个条件
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

        final CellLayout parent =
                cell.getParent() == null ? null : (CellLayout) cell.getParent().getParent();
        if (d.dragView.hasDrawn()) {
            if (droppedOnOriginalCellDuringTransition) {
                // Animate the item to its original position, while simultaneously exiting
                // spring-loaded mode so the page meets the icon where it was picked up.
                if (cell.getParent() != null) {
                    mLauncher.getDragController().animateDragViewToOriginalPosition(
                            onCompleteRunnable, cell, SPRING_LOADED_TRANSITION_MS);
                } else {
                    resetDragCellIfNeed(mDragInfo);
                }
                mLauncher.getStateManager().goToState(NORMAL);
                mLauncher.getDropTargetBar().onDragEnd();
                if (parent != null) {
                    parent.onDropChild(cell);
                }
                return;
            }
            final ItemInfo info = (ItemInfo) cell.getTag();
            boolean isWidget = info.itemType == LauncherSettings.Favorites.ITEM_TYPE_APPWIDGET
                    || info.itemType == LauncherSettings.Favorites.ITEM_TYPE_CUSTOM_APPWIDGET;
            if (isWidget) {
                int animationType = resizeOnDrop ? ANIMATE_INTO_POSITION_AND_RESIZE :
                        ANIMATE_INTO_POSITION_AND_DISAPPEAR;
                if (parent != null) {
                    animateWidgetDrop(info, parent, d.dragView, null, animationType, cell, false);
                }
            } else {
                int duration = snapScreen < 0 ? -1 : ADJACENT_SCREEN_DROP_DURATION;
                if (cell.getParent() != null) {
                    mLauncher.getDragLayer().animateViewIntoPosition(d.dragView, cell, duration,
                            this);
                } else {
                    d.deferDragViewCleanupPostAnimation = false;
                    resetDragCellIfNeed(mDragInfo);
                }
            }
        } else {
            d.deferDragViewCleanupPostAnimation = false;
            if (cell.getParent() != null) {
                cell.setVisibility(VISIBLE);
            } else {
                resetDragCellIfNeed(mDragInfo);
            }
        }
        if (parent != null) {
            parent.onDropChild(cell);
        }

        mLauncher.getStateManager().goToState(
                NORMAL, SPRING_LOADED_EXIT_DELAY, onCompleteRunnable);
    }

    if (d.stateAnnouncer != null && !droppedOnOriginalCell) {
        d.stateAnnouncer.completeAction(R.string.item_moved);
    }
}

```

通过上文已经获取到当前拖拽所占屏 mCurScrrenId 的值 和 Item 的个数 mCurChildCount  
而 mWorkspaceScreens.size() 的个数包含了一个空屏 当在最后一屏拖拽时会显示个空屏  
mCurScrrenId+1==mWorkspaceScreens.size()-1 则表示是最后一屏在拖拽  
getIdForScreen(dropTargetLayout)+1>=mWorkspaceScreens.size() 则表示要拖拽到空屏  
具体修改如下:

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
        Log.e("Launcher3","countx:"+dropTargetLayout.getCountX()+"--county:"+dropTargetLayout.getCountY()+"---childcount:"+dropTargetLayout.childCount()+"--mTargetCell[0]:"+mTargetCell[0]+"--mTargetCell[1]:"+mTargetCell[1]
		             +"--screenId:"+getIdForScreen(dropTargetLayout)+"--mWorkspaceScreens.size():"+mWorkspaceScreens.size());
		int count = dropTargetLayout.getCountX()*dropTargetLayout.getCountY();
		boolean isAllowDrag = true;
		if(mCurScrrenId+1==mWorkspaceScreens.size()-1&&mCurChildCount<count-6&&getIdForScreen(dropTargetLayout)+1>=mWorkspaceScreens.size())isAllowDrag=false;
        if (dropTargetLayout != null && !d.cancelled&&isAllowDrag) {
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

        final CellLayout parent =
                cell.getParent() == null ? null : (CellLayout) cell.getParent().getParent();
        if (d.dragView.hasDrawn()) {
            if (droppedOnOriginalCellDuringTransition) {
                // Animate the item to its original position, while simultaneously exiting
                // spring-loaded mode so the page meets the icon where it was picked up.
                if (cell.getParent() != null) {
                    mLauncher.getDragController().animateDragViewToOriginalPosition(
                            onCompleteRunnable, cell, SPRING_LOADED_TRANSITION_MS);
                } else {
                    resetDragCellIfNeed(mDragInfo);
                }
                mLauncher.getStateManager().goToState(NORMAL);
                mLauncher.getDropTargetBar().onDragEnd();
                if (parent != null) {
                    parent.onDropChild(cell);
                }
                return;
            }
            final ItemInfo info = (ItemInfo) cell.getTag();
            boolean isWidget = info.itemType == LauncherSettings.Favorites.ITEM_TYPE_APPWIDGET
                    || info.itemType == LauncherSettings.Favorites.ITEM_TYPE_CUSTOM_APPWIDGET;
            if (isWidget) {
                int animationType = resizeOnDrop ? ANIMATE_INTO_POSITION_AND_RESIZE :
                        ANIMATE_INTO_POSITION_AND_DISAPPEAR;
                if (parent != null) {
                    animateWidgetDrop(info, parent, d.dragView, null, animationType, cell, false);
                }
            } else {
                int duration = snapScreen < 0 ? -1 : ADJACENT_SCREEN_DROP_DURATION;
                if (cell.getParent() != null) {
                    mLauncher.getDragLayer().animateViewIntoPosition(d.dragView, cell, duration,
                            this);
                } else {
                    d.deferDragViewCleanupPostAnimation = false;
                    resetDragCellIfNeed(mDragInfo);
                }
            }
        } else {
            d.deferDragViewCleanupPostAnimation = false;
            if (cell.getParent() != null) {
                cell.setVisibility(VISIBLE);
            } else {
                resetDragCellIfNeed(mDragInfo);
            }
        }
        if (parent != null) {
            parent.onDropChild(cell);
        }

        mLauncher.getStateManager().goToState(
                NORMAL, SPRING_LOADED_EXIT_DELAY, onCompleteRunnable);
    }

    if (d.stateAnnouncer != null && !droppedOnOriginalCell) {
        d.stateAnnouncer.completeAction(R.string.item_moved);
    }
}

```
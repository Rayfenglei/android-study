> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126432502)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.Launcher3 禁止首屏时钟部件拖动到其他屏的核心代码](#t1)

[3.Launcher3 禁止首屏时钟部件拖动到其他屏的核心功能分析](#t2) 

 [3.1DragController.java 关于拖拽的接口](#3.Launcher3%20%E7%A6%81%E6%AD%A2%E9%A6%96%E5%B1%8F%E6%97%B6%E9%92%9F%E9%83%A8%E4%BB%B6%E6%8B%96%E5%8A%A8%E5%88%B0%E5%85%B6%E4%BB%96%E5%B1%8F%E7%9A%84%E6%A0%B8%E5%BF%83%E5%8A%9F%E8%83%BD%E5%88%86%E6%9E%90%C2%A0%C2%A0%203.1DragController.java%20%E5%85%B3%E4%BA%8E%E6%8B%96%E6%8B%BD%E7%9A%84%E6%8E%A5%E5%8F%A3)

 [3.2 Workspace.java 中拖拽的相关方法](#t3)

1. 概述
-----

   在系统 Launcher3 中，首页中间默认有个时钟部件来显示时间，并且可以任意拖拽到其他地方，如果拖动到其他屏显的很不美观，所以根据需要要求时钟部件  
不能拖拽到其他屏，所以就要从拖拽开始处理，判断如果是时钟部件，就不让拖拽到其他屏，先从拖拽流程分析

2.Launcher3 禁止首屏时钟部件拖动到其他屏的核心代码
-------------------------------

```
   packages/apps/Launcher3/src/com/android/launcher3/Workspace.java
   packages/apps/Launcher3/src/com/android/launcher3/dragndrop/DragController.java
```

### 3.Launcher3 禁止首屏时钟部件拖动到其他屏的核心功能分析  
   3.1DragController.java 关于拖拽的接口

```
public class DragController implements DragDriver.EventListener, TouchController {
	public interface DragListener {
		/**
* A drag has begun
*
* @param dragObject The object being dragged
* @param options Options used to start the drag
*/
		void onDragStart(DropTarget.DragObject dragObject, DragOptions options);
		/**
* The drag has ended
*/
		void onDragEnd();
	}
```

###   DragController 拖拽接口，开始拖拽 拖拽结束接口调用 然后实现对 app 拖拽的位置调整

### 3.2 Workspace.java 中拖拽的相关方法

```
    public class Workspace extends PagedView<WorkspacePageIndicator>
implements DropTarget, DragSource, View.OnTouchListener,
DragController.DragListener, Insettable, StateHandler<LauncherState>,
WorkspaceLayoutManager {
	@Override
	public void onDragStart(DragObject dragObject, DragOptions options) {
		if (ENFORCE_DRAG_EVENT_ORDER) {
			enforceDragParity("onDragStart", 0, 0);
		}
		if (mDragInfo != null && mDragInfo.cell != null) {
			CellLayout layout = (CellLayout) mDragInfo.cell.getParent().getParent();
			layout.markCellsAsUnoccupiedForView(mDragInfo.cell);
		}
		if (mOutlineProvider != null) {
			if (dragObject.dragView != null) {
				Bitmap preview = dragObject.dragView.getPreviewBitmap();
				// The outline is used to visualize where the item will land if dropped
				mOutlineProvider.generateDragOutline(preview);
			}
		}
		updateChildrenLayersEnabled();
		// Do not add a new page if it is a accessible drag which was not started by the workspace.
		// We do not support accessibility drag from other sources and instead provide a direct
		// action for move/add to homescreen.
		// When a accessible drag is started by the folder, we only allow rearranging withing the
		// folder.
		Boolean addNewPage = !(options.isAccessibleDrag && dragObject.dragSource != this);
		if (addNewPage) {
			mDeferRemoveExtraEmptyScreen = false;
			addExtraEmptyScreenOnDrag();
			if (dragObject.dragInfo.itemType == LauncherSettings.Favorites.ITEM_TYPE_APPWIDGET
			&& dragObject.dragSource != this) {
				// When dragging a widget from different source, move to a page which has
				// enough space to place this widget (after rearranging/resizing). We special case
				// widgets as they cannot be placed inside a folder.
				// Start at the current page and search right (on LTR) until finding a page with
				// enough space. Since an empty screen is the furthest right, a page must be found.
				int currentPage = getPageNearestToCenterOfScreen();
				for (int pageIndex = currentPage; pageIndex < getPageCount(); pageIndex++) {
					CellLayout page = (CellLayout) getPageAt(pageIndex);
					if (page.hasReorderSolution(dragObject.dragInfo)) {
						setCurrentPage(pageIndex);
						break;
					}
				}
			}
		}
		// Always enter the spring loaded mode
		mLauncher.getStateManager().goToState(SPRING_LOADED);
		mStatsLogManager.logger().withItemInfo(dragObject.dragInfo)
		withInstanceId(dragObject.logInstanceId)
		log(LauncherEvent.LAUNCHER_ITEM_DRAG_STARTED);
	}
	@Override
	public void onDragEnd() {
		if (ENFORCE_DRAG_EVENT_ORDER) {
			enforceDragParity("onDragEnd", 0, 0);
		}
		if (!mDeferRemoveExtraEmptyScreen) {
			removeExtraEmptyScreen(mDragSourceInternal != null);
		}
		updateChildrenLayersEnabled();
		mDragInfo = null;
		mOutlineProvider = null;
		mDragSourceInternal = null;
	}
	@Override
	public void onDrop(final DragObject d, DragOptions options) {
		mDragViewVisualCenter = d.getVisualCenter(mDragViewVisualCenter);
		CellLayout dropTargetLayout = mDropToLayout;
		// We want the point to be mapped to the dragTarget.
		if (dropTargetLayout != null) {
			mapPointFromDropLayout(dropTargetLayout, mDragViewVisualCenter);
		}
		Boolean droppedOnOriginalCell = false;
		int snapScreen = -1;
		Boolean resizeOnDrop = false;
		if (d.dragSource != this || mDragInfo == null) {
			final int[] touchXY = new int[] { (int) mDragViewVisualCenter[0],
			(int) mDragViewVisualCenter[1]
		}
		;
		onDropExternal(touchXY, dropTargetLayout, d);
	} else {
		final View cell = mDragInfo.cell;
		Boolean droppedOnOriginalCellDuringTransition = false;
		Runnable onCompleteRunnable = null;
		if (dropTargetLayout != null && !d.cancelled) {
			// Move internally
			Boolean hasMovedLayouts = (getParentCellLayoutForView(cell) != dropTargetLayout);
			Boolean hasMovedIntoHotseat = mLauncher.isHotseatLayout(dropTargetLayout);
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
			dropTargetLayout, mTargetCell, distance, false, d)
			|| addToExistingFolderIfNecessary(cell, dropTargetLayout, mTargetCell,
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
			Boolean returnToOriginalCellToPreventShuffling = !isFinishedSwitchingState()
			&& !droppedOnOriginalCellDuringTransition && !dropTargetLayout
			isRegionVacant(mTargetCell[0], mTargetCell[1], spanX, spanY);
			int[] resultSpan = new int[2];
			if (returnToOriginalCellToPreventShuffling) {
				mTargetCell[0] = mTargetCell[1] = -1;
			} else {
				mTargetCell = dropTargetLayout.performReorder((int) mDragViewVisualCenter[0],
				(int) mDragViewVisualCenter[1], minSpanX, minSpanY, spanX, spanY, cell,
				mTargetCell, resultSpan, CellLayout.MODE_ON_DROP);
			}
			Boolean foundCell = mTargetCell[0] >= 0 && mTargetCell[1] >= 0;
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
					} else if (FeatureFlags.IS_STUDIO_BUILD) {
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
					&& !options.isAccessibleDrag) {
						onCompleteRunnable = () -> {
							if (!isPageInTransition()) {
								AppWidgetResizeFrame.showForWidget(hostView, cellLayout);
							}
						}
						;
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
				CellLayout layout = (CellLayout) cell.getParent().getParent();
				layout.markCellsAsOccupiedForView(cell);
			}
		}
		final CellLayout parent = (CellLayout) cell.getParent().getParent();
		if (d.dragView.hasDrawn()) {
			if (droppedOnOriginalCellDuringTransition) {
				// Animate the item to its original position, while simultaneously exiting
				// spring-loaded mode so the page meets the icon where it was picked up.
				mLauncher.getDragController().animateDragViewToOriginalPosition(
				onCompleteRunnable, cell,
				SPRING_LOADED.getTransitionDuration(mLauncher));
				mLauncher.getStateManager().goToState(NORMAL);
				mLauncher.getDropTargetBar().onDragEnd();
				parent.onDropChild(cell);
				return;
			}
			final ItemInfo info = (ItemInfo) cell.getTag();
			Boolean isWidget = info.itemType == LauncherSettings.Favorites.ITEM_TYPE_APPWIDGET
			|| info.itemType == LauncherSettings.Favorites.ITEM_TYPE_CUSTOM_APPWIDGET;
			if (isWidget) {
				int animationType = resizeOnDrop ? ANIMATE_INTO_POSITION_AND_RESIZE :
				ANIMATE_INTO_POSITION_AND_DISAPPEAR;
				animateWidgetDrop(info, parent, d.dragView, null, animationType, cell, false);
			} else {
				int duration = snapScreen < 0 ? -1 : ADJACENT_SCREEN_DROP_DURATION;
				mLauncher.getDragLayer().animateViewIntoPosition(d.dragView, cell, duration,
				this);
			}
		} else {
			d.deferDragViewCleanupPostAnimation = false;
			cell.setVisibility(VISIBLE);
		}
		parent.onDropChild(cell);
		mLauncher.getStateManager().goToState(
		NORMAL, SPRING_LOADED_EXIT_DELAY, onCompleteRunnable);
		mStatsLogManager.logger().withItemInfo(d.dragInfo).withInstanceId(d.logInstanceId)
		log(LauncherEvent.LAUNCHER_ITEM_DROP_COMPLETED);
	}
	if (d.stateAnnouncer != null && !droppedOnOriginalCell) {
		d.stateAnnouncer.completeAction(R.string.item_moved);
	}
}
}
```

在 Workspace 中 addInScreen 方法最后，给图标设置的[监听器](https://so.csdn.net/so/search?q=%E7%9B%91%E5%90%AC%E5%99%A8&spm=1001.2101.3001.7020)是 Launcher 对象，他实现了 onLongClick 方法。  
拖拽流程:

在 workspaces 中在检测到拖拽的过程中，onDragStart() 处理的是开始拖拽的事件  
onDrop() 处理的是松手时新布局相关的事件处理, onDragEnd() 处理拖拽结束后更新布局

所以重点在 onDrop() 中处理事件

具体修改为:

```
具体修改为:
	@Override
	public void onDrop(final DragObject d, DragOptions options) {
		mDragViewVisualCenter = d.getVisualCenter(mDragViewVisualCenter);
		CellLayout dropTargetLayout = mDropToLayout;
		// We want the point to be mapped to the dragTarget.
		if (dropTargetLayout != null) {
			mapPointFromDropLayout(dropTargetLayout, mDragViewVisualCenter);
		}
		Boolean droppedOnOriginalCell = false;
		int snapScreen = -1;
		Boolean resizeOnDrop = false;
		if (d.dragSource != this || mDragInfo == null) {
			final int[] touchXY = new int[] { (int) mDragViewVisualCenter[0],
			(int) mDragViewVisualCenter[1]
		}
		;
		onDropExternal(touchXY, dropTargetLayout, d);
	} else {
		final View cell = mDragInfo.cell;
		Boolean droppedOnOriginalCellDuringTransition = false;
		Runnable onCompleteRunnable = null;
  // add code start
        ItemInfo iteminfo = d.dragInfo;
	String packagename = "";
	if(iteminfo!=null && iteminfo instanceof LauncherAppWidgetInfo){
		ComponentName componentName = ((LauncherAppWidgetInfo)iteminfo).providerName;
		if(componentName!=null){
			packagename = componentName.getPackageName();
		}
	}
        // add code end
      - if (dropTargetLayout != null && !d.cancelled ) { // 判断包名是否是时钟
      +  if (dropTargetLayout != null && !d.cancelled && !packagename.equals("com.android.deskclock")) {
			// Move internally
			Boolean hasMovedLayouts = (getParentCellLayoutForView(cell) != dropTargetLayout);
			Boolean hasMovedIntoHotseat = mLauncher.isHotseatLayout(dropTargetLayout);
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
			dropTargetLayout, mTargetCell, distance, false, d)
			|| addToExistingFolderIfNecessary(cell, dropTargetLayout, mTargetCell,
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
			Boolean returnToOriginalCellToPreventShuffling = !isFinishedSwitchingState()
			&& !droppedOnOriginalCellDuringTransition && !dropTargetLayout
			isRegionVacant(mTargetCell[0], mTargetCell[1], spanX, spanY);
			int[] resultSpan = new int[2];
			if (returnToOriginalCellToPreventShuffling) {
				mTargetCell[0] = mTargetCell[1] = -1;
			} else {
				mTargetCell = dropTargetLayout.performReorder((int) mDragViewVisualCenter[0],
				(int) mDragViewVisualCenter[1], minSpanX, minSpanY, spanX, spanY, cell,
				mTargetCell, resultSpan, CellLayout.MODE_ON_DROP);
			}
			Boolean foundCell = mTargetCell[0] >= 0 && mTargetCell[1] >= 0;
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
					} else if (FeatureFlags.IS_STUDIO_BUILD) {
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
					&& !options.isAccessibleDrag) {
						onCompleteRunnable = () -> {
							if (!isPageInTransition()) {
								AppWidgetResizeFrame.showForWidget(hostView, cellLayout);
							}
						}
						;
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
				CellLayout layout = (CellLayout) cell.getParent().getParent();
				layout.markCellsAsOccupiedForView(cell);
			}
		}
		final CellLayout parent = (CellLayout) cell.getParent().getParent();
		if (d.dragView.hasDrawn()) {
			if (droppedOnOriginalCellDuringTransition) {
				// Animate the item to its original position, while simultaneously exiting
				// spring-loaded mode so the page meets the icon where it was picked up.
				mLauncher.getDragController().animateDragViewToOriginalPosition(
				onCompleteRunnable, cell,
				SPRING_LOADED.getTransitionDuration(mLauncher));
				mLauncher.getStateManager().goToState(NORMAL);
				mLauncher.getDropTargetBar().onDragEnd();
				parent.onDropChild(cell);
				return;
			}
			final ItemInfo info = (ItemInfo) cell.getTag();
			Boolean isWidget = info.itemType == LauncherSettings.Favorites.ITEM_TYPE_APPWIDGET
			|| info.itemType == LauncherSettings.Favorites.ITEM_TYPE_CUSTOM_APPWIDGET;
			if (isWidget) {
				int animationType = resizeOnDrop ? ANIMATE_INTO_POSITION_AND_RESIZE :
				ANIMATE_INTO_POSITION_AND_DISAPPEAR;
				animateWidgetDrop(info, parent, d.dragView, null, animationType, cell, false);
			} else {
				int duration = snapScreen < 0 ? -1 : ADJACENT_SCREEN_DROP_DURATION;
				mLauncher.getDragLayer().animateViewIntoPosition(d.dragView, cell, duration,
				this);
			}
		} else {
			d.deferDragViewCleanupPostAnimation = false;
			cell.setVisibility(VISIBLE);
		}
		parent.onDropChild(cell);
		mLauncher.getStateManager().goToState(
		NORMAL, SPRING_LOADED_EXIT_DELAY, onCompleteRunnable);
		mStatsLogManager.logger().withItemInfo(d.dragInfo).withInstanceId(d.logInstanceId)
		log(LauncherEvent.LAUNCHER_ITEM_DROP_COMPLETED);
	}
	if (d.stateAnnouncer != null && !droppedOnOriginalCell) {
		d.stateAnnouncer.completeAction(R.string.item_moved);
	}
}
```
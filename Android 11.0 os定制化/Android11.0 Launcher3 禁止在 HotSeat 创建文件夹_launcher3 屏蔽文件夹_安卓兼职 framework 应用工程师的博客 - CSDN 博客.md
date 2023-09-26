> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124872458)

### 1. 概述

在 11.0laucher3 中拖拽 item 时 靠近某个图标时会形成文件夹（folder）, 而根据客户需求不想再 hotseat 形成文件夹， 这就要从 workspace.java 从来寻找解决方案了分析 hotseat 是怎么变成 folder 的

### 2.Launcher3 禁止在 HotSeat 创建文件夹核心代码

```
/packages/apps/Launcher3/src/com/android/launcher3/Workspace.java
/packages/apps/Launcher3/src/com/android/launcher3/CellLayout.java

```

3.Launcher3 禁止在 HotSeat 创建文件夹功能分析和实现
------------------------------------

功能分析：  
Launcher3 中形成文件夹，是在 Workspace.java 中的 onDrop() 方法里面实现的，拖动图标落点处可以合成一个 Folder，如果不满足文件夹的条件，则调用 CellLayout.java 的 performReorder 方法  
createUserFolderIfNecessary() 方法

### 3.1 CellLayout.java 相关方法分析

```
boolean createUserFolderIfNecessary(View newView, long container, CellLayout target,
            int[] targetCell, float distance, boolean external, DragView dragView,
            Runnable postAnimationRunnable) {
if (distance > mMaxDistanceForFolderCreation) return false;
        View v = target.getChildAt(targetCell[0], targetCell[1]);

        boolean hasntMoved = false;
        if (mDragInfo != null && mDragInfo.cell.getParent() != null) {
            CellLayout cellParent = getParentCellLayoutForView(mDragInfo.cell);
            hasntMoved = (mDragInfo.cellX == targetCell[0] &&
                    mDragInfo.cellY == targetCell[1]) && (cellParent == target);
        }

        if (v == null || hasntMoved || !mCreateUserFolderOnDrop) return false;
        mCreateUserFolderOnDrop = false;
        final int screenId = getIdForScreen(target);

        boolean aboveShortcut = (v.getTag() instanceof WorkspaceItemInfo);
        boolean willBecomeShortcut = (newView.getTag() instanceof WorkspaceItemInfo);

        if (aboveShortcut && willBecomeShortcut) {
            WorkspaceItemInfo sourceInfo = (WorkspaceItemInfo) newView.getTag();
            WorkspaceItemInfo destInfo = (WorkspaceItemInfo) v.getTag();
            // if the drag started here, we need to remove it from the workspace
            if (!external && mDragInfo.cell.getParent() != null) {
                getParentCellLayoutForView(mDragInfo.cell).removeView(mDragInfo.cell);
            }

            Rect folderLocation = new Rect();
            float scale = mLauncher.getDragLayer().getDescendantRectRelativeToSelf(v, folderLocation);
            target.removeView(v);

            FolderIcon fi =
                mLauncher.addFolder(target, container, screenId, targetCell[0], targetCell[1]);
            destInfo.cellX = -1;
            destInfo.cellY = -1;
            sourceInfo.cellX = -1;
            sourceInfo.cellY = -1;

            // If the dragView is null, we can't animate
            boolean animate = dragView != null;
            if (animate) {
                // In order to keep everything continuous, we hand off the currently rendered
                // folder background to the newly created icon. This preserves animation state.
                fi.setFolderBackground(mFolderCreateBg);
                mFolderCreateBg = new PreviewBackground();
                fi.performCreateAnimation(destInfo, v, sourceInfo, dragView, folderLocation, scale
                );
            } else {
                fi.prepareCreateAnimation(v);
                fi.addItem(destInfo);
                fi.addItem(sourceInfo);
            }
            return true;
        }
        return false;
    }

```

所以不想再 hotseat 形成文件夹  
就做如下修改:

```
boolean createUserFolderIfNecessary(View newView, long container, CellLayout target,
            int[] targetCell, float distance, boolean external, DragView dragView,
            Runnable postAnimationRunnable) {
        //core begin
        if (container == LauncherSettings.Favorites.CONTAINER_HOTSEAT) {
            return false;
        }
        //core end
if (distance > mMaxDistanceForFolderCreation) return false;
        View v = target.getChildAt(targetCell[0], targetCell[1]);

```

运行一下，发现确实有效果 但是在 HotSeat 中拖动一个 APP 放到目标 APP 上方仍然会有一个文件夹的虚影  
继续在 workspace.java 中看到了 manageFolderFeedback()，管理文件夹 进入方法看下：

### 3.2workspace.java 相关文件夹代码分析

```
private void manageFolderFeedback(CellLayout targetLayout,
            int[] targetCell, float distance, DragObject dragObject) {
        if (distance > mMaxDistanceForFolderCreation) {
            if (mDragMode != DRAG_MODE_NONE) {
                setDragMode(DRAG_MODE_NONE);
            }
            return;
        }

        final View dragOverView = mDragTargetLayout.getChildAt(mTargetCell[0], mTargetCell[1]);
        ItemInfo info = dragObject.dragInfo;
        boolean userFolderPending = willCreateUserFolder(info, dragOverView, false);
        if (mDragMode == DRAG_MODE_NONE && userFolderPending &&
                !mFolderCreationAlarm.alarmPending()) {

            FolderCreationAlarmListener listener = new
                    FolderCreationAlarmListener(targetLayout, targetCell[0], targetCell[1]);

            if (!dragObject.accessibleDrag) {
                mFolderCreationAlarm.setOnAlarmListener(listener);
                mFolderCreationAlarm.setAlarm(FOLDER_CREATION_TIMEOUT);
            } else {
                listener.onAlarm(mFolderCreationAlarm);
            }

            if (dragObject.stateAnnouncer != null) {
                dragObject.stateAnnouncer.announce(WorkspaceAccessibilityHelper
                        .getDescriptionForDropOver(dragOverView, getContext()));
            }
            return;
        }

        boolean willAddToFolder = willAddToExistingUserFolder(info, dragOverView) && !mLauncher.isHotseatLayout(targetLayout);
        if (willAddToFolder && mDragMode == DRAG_MODE_NONE) {
            mDragOverFolderIcon = ((FolderIcon) dragOverView);
            mDragOverFolderIcon.onDragEnter(info);
            if (targetLayout != null) {
                targetLayout.clearDragOutlines();
            }
            setDragMode(DRAG_MODE_ADD_TO_FOLDER);

            if (dragObject.stateAnnouncer != null) {
                dragObject.stateAnnouncer.announce(WorkspaceAccessibilityHelper
                        .getDescriptionForDropOver(dragOverView, getContext()));
            }
            return;
        }

        if (mDragMode == DRAG_MODE_ADD_TO_FOLDER && !willAddToFolder) {
            setDragMode(DRAG_MODE_NONE);
        }
        if (mDragMode == DRAG_MODE_CREATE_FOLDER && !userFolderPending) {
            setDragMode(DRAG_MODE_NONE);
        }
    }

```

改完后 测试发现找到解决方法:

修改如下:

```
private void manageFolderFeedback(CellLayout targetLayout,
            int[] targetCell, float distance, DragObject dragObject) {
        if (distance > mMaxDistanceForFolderCreation) return;

        final View dragOverView = mDragTargetLayout.getChildAt(mTargetCell[0], mTargetCell[1]);
        ItemInfo info = dragObject.dragInfo;

      -  boolean userFolderPending = willCreateUserFolder(info, dragOverView, false);





      + boolean userFolderPending = willCreateUserFolder(info, dragOverView, false)
                && !mLauncher.isHotseatLayout(targetLayout);

```
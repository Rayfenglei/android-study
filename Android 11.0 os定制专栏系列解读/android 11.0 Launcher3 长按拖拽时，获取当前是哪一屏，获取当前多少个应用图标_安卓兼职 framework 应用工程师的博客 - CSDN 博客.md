> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124828360)

### 1. 概述

在 11.0 定制化开发手机项目中, 如果专门适配老年机的时候，这时客户提出要求，如果最后一屏未满时，不让拖拽到后面一屏的空屏中这样就需要获取当前是哪一屏，并且要知道当前有多少个 Item，总共一屏最多多少个 item  
所以就需要从 Workspace.java 入手

### 2.Launcher3 长按拖拽时，获取当前是哪一屏，获取当前多少个应用图标的核心类

```
packages/apps/Launcher3/src/com/android/launcher3/Workspace.java
packages/apps/Launcher3/src/com/android/launcher3/CellLayout.java

```

### 3.Launcher3 长按拖拽时，获取当前是哪一屏，获取当前多少个应用图标的核心功能实现和分析

首选来看源码 开始拖拽时会调用 startDrag()

```
public void startDrag(CellLayout.CellInfo cellInfo, DragOptions options) {
View child = cellInfo.cell;
mDragInfo = cellInfo;
child.setVisibility(INVISIBLE);
if (options.isAccessibleDrag) {
mDragController.addDragListener(new AccessibleDragListenerAdapter(
this, CellLayout.WORKSPACE_ACCESSIBILITY_DRAG) {
@Override
protected void enableAccessibleDrag(boolean enable) {
super.enableAccessibleDrag(enable);
setEnableForLayout(mLauncher.getHotseat(),enable);
}
});
}
beginDragShared(child, this, options);

}
    public void beginDragShared(View child, DragSource source, DragOptions options) {
          Object dragObject = child.getTag();
          if (!(dragObject instanceof ItemInfo)) {
              String msg = "Drag started with a view that has no tag set. This "
                      + "will cause a crash (issue 11627249) down the line. "
                      + "View: " + child + "  tag: " + child.getTag();
              throw new IllegalStateException(msg);
          }
          beginDragShared(child, null, source, (ItemInfo) dragObject,
                  new DragPreviewProvider(child), options);
      }

       * Core functionality for beginning a drag operation for an item that will be dropped within
       * the workspace
       */
      public DragView beginDragShared(View child, DraggableView draggableView, DragSource source,
              ItemInfo dragObject, DragPreviewProvider previewProvider, DragOptions dragOptions) {
  
          float iconScale = 1f;
          if (child instanceof BubbleTextView) {
              Drawable icon = ((BubbleTextView) child).getIcon();
              if (icon instanceof FastBitmapDrawable) {
                  iconScale = ((FastBitmapDrawable) icon).getAnimatedScale();
              }
          }
  
          // Clear the pressed state if necessary
          child.clearFocus();
          child.setPressed(false);
          if (child instanceof BubbleTextView) {
              BubbleTextView icon = (BubbleTextView) child;
              icon.clearPressedBackground();
          }
  
          mOutlineProvider = previewProvider;
  
          if (draggableView == null && child instanceof DraggableView) {
              draggableView = (DraggableView) child;
          }
  
          final View contentView = previewProvider.getContentView();
          final float scale;
          // The draggable drawable follows the touch point around on the screen
          final Drawable drawable;
          if (contentView == null) {
              drawable = previewProvider.createDrawable();
              scale = previewProvider.getScaleAndPosition(drawable, mTempXY);
          } else {
              drawable = null;
              scale = previewProvider.getScaleAndPosition(contentView, mTempXY);
          }
  
          int halfPadding = previewProvider.previewPadding / 2;
          int dragLayerX = mTempXY[0];
          int dragLayerY = mTempXY[1];
  
          Point dragVisualizeOffset = null;
          Rect dragRect = new Rect();
  
          if (draggableView != null) {
              draggableView.getSourceVisualDragBounds(dragRect);
              dragLayerY += dragRect.top;
              dragVisualizeOffset = new Point(- halfPadding, halfPadding);
          }
  
  
          if (child.getParent() instanceof ShortcutAndWidgetContainer) {
              mDragSourceInternal = (ShortcutAndWidgetContainer) child.getParent();
          }
  
          if (child instanceof BubbleTextView && !dragOptions.isAccessibleDrag) {
              PopupContainerWithArrow popupContainer = PopupContainerWithArrow
                      .showForIcon((BubbleTextView) child);
              if (popupContainer != null) {
                  dragOptions.preDragCondition = popupContainer.createPreDragCondition();
              }
          }
  
          final DragView dv;
          if (contentView instanceof View) {
              if (contentView instanceof LauncherAppWidgetHostView) {
                  mDragController.addDragListener(new AppWidgetHostViewDragListener(mLauncher));
              }
              dv = mDragController.startDrag(
                      contentView,
                      draggableView,
                      dragLayerX,
                      dragLayerY,
                      source,
                      dragObject,
                      dragVisualizeOffset,
                      dragRect,
                      scale * iconScale,
                      scale,
                      dragOptions);
          } else {
              dv = mDragController.startDrag(
                      drawable,
                      draggableView,
                      dragLayerX,
                      dragLayerY,
                      source,
                      dragObject,
                      dragVisualizeOffset,
                      dragRect,
                      scale * iconScale,
                      scale,
                      dragOptions);
          }
          return dv;
      }
  

```

在 WorkSpace.java 中，对手势拖拽 icon 的相关方法，从 startDrag（）beginDragShared() onDrap() 等相关  
方法来处理拖拽的相关动作，  
所以要首选看 CellInfo 这个内部类：

```
public static final class CellInfo extends CellAndSpan {
public final View cell;
final int screenId;
public final int container;
public CellInfo(View v, ItemInfo info) {
    cellX = info.cellX;
    cellY = info.cellY;
    spanX = info.spanX;
    spanY = info.spanY;
    cell = v;
    screenId = info.screenId;
    container = info.container;
}

@Override
public String toString() {
    return "Cell[view=" + (cell == null ? "null" : cell.getClass())
            + ", x=" + cellX + ", y=" + cellY + "]";
}

}

```

CellInfo 这个内部类主要是记录每个 Item 的相关信息 属于哪一屏 x y 坐标等  
从源码中可以看出 刚好有 screenId 这个变量 就代表当前是哪一屏  
继续看 CellLayout.[java 源码](https://so.csdn.net/so/search?q=java%E6%BA%90%E7%A0%81&spm=1001.2101.3001.7020)

```
public int getCountX() {
return mCountX;
}
public int getCountY() {
return mCountY;
}

```

发现 getCountX 代表有多少行 getCountY() 有多少列  
所以最多每一屏就是 getCountX()*getCountY();  
而当前屏有多少个 Item  
就是 mShortcutsAndWidgets.getChildCount();

```
所以CellLayout.java增加个方法
public int childCount(){
if(mShortcutsAndWidgets==null)return 0;
return mShortcutsAndWidgets.getChildCount();
}

```

所以最终修改为：

```
private int mCurScrrenId=-1,mCurChildCount=-1;
public void startDrag(CellLayout.CellInfo cellInfo, DragOptions options) {
View child = cellInfo.cell;
mDragInfo = cellInfo;
child.setVisibility(INVISIBLE);// add code start
mCurScrrenId = cellInfo.screenId;
CellLayout mSourceCellLayout = mWorkspaceScreens.valueAt(mCurScrrenId);
mCurChildCount = mSourceCellLayout.childCount();
if(mSourceCellLayout!=null)Log.e("Launcher3","source--countx:"+mSourceCellLayout.getCountX()+"--county:"+mSourceCellLayout.getCountY()+"---childcount:"+mCurChildCount+"mCurScrrenId:"+mCurScrrenId);
// add code endif (options.isAccessibleDrag) {
mDragController.addDragListener(new AccessibleDragListenerAdapter(
this, CellLayout.WORKSPACE_ACCESSIBILITY_DRAG) {
@Override
protected void enableAccessibleDrag(boolean enable) {
super.enableAccessibleDrag(enable);
setEnableForLayout(mLauncher.getHotseat(),enable);
}
});
}beginDragShared(child, this, options);
}

```

经过编译验证拖拽时当前屏和 Item 个数都是一样的
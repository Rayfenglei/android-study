> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124830745)

1. 概述
-----

在 11.0 Launcher3 开发中，在长按屏幕的时候，会弹出窗口，修改主屏幕配置，壁纸，等信息，由于要默认设置一些配置  
不想让用户修改相关配置，这时候就需要去掉长按弹窗功能了，禁止修改相关配置

2.Launcher3 去掉 workspace 长按弹出壁纸弹窗的核心类
-------------------------------------

```
 /packages/apps/Launcher3/src/com/android/launcher3/Workspace.java
 /packages/apps/Launcher3/src/com/android/launcher3/touch/WorkspaceTouchListener.java

```

3.Launcher3 去掉 workspace 长按弹出壁纸弹窗的相关核心功能实现
------------------------------------------

3.1 workspace.java 相关 app 控件布局的分析
---------------------------------

下面来分析下 workspace 相关长按事件的功能实现  
先看 workspace.java 源码

```
         public class Workspace extends PagedView<WorkspacePageIndicator>
          implements DropTarget, DragSource, View.OnTouchListener,
          DragController.DragListener, Insettable, StateHandler<LauncherState>,
          WorkspaceLayoutManager {
         public Workspace(Context context, AttributeSet attrs) {
          this(context, attrs, 0);
      }
      public Workspace(Context context, AttributeSet attrs, int defStyle) {
          super(context, attrs, defStyle);
  
          mLauncher = Launcher.getLauncher(context);
          mStateTransitionAnimation = new WorkspaceStateTransitionAnimation(mLauncher, this);
          mWallpaperManager = WallpaperManager.getInstance(context);
  
          mWallpaperOffset = new WallpaperOffsetInterpolator(this);
  
          setHapticFeedbackEnabled(false);
          initWorkspace();
  
          // Disable multitouch across the workspace/all apps/customize tray
          setMotionEventSplittingEnabled(true);
          setOnTouchListener(new WorkspaceTouchListener(mLauncher, this));
          mStatsLogManager = StatsLogManager.newInstance(context);
      }
  
      @Override
      public void setInsets(Rect insets) {
          DeviceProfile grid = mLauncher.getDeviceProfile();
  
          mMaxDistanceForFolderCreation = grid.isTablet
                  ? 0.75f * grid.iconSizePx : 0.55f * grid.iconSizePx;
          mWorkspaceFadeInAdjacentScreens = grid.shouldFadeAdjacentWorkspaceScreens();
  
          Rect padding = grid.workspacePadding;
          setPadding(padding.left, padding.top, padding.right, padding.bottom);
          mInsets.set(insets);
  
          if (mWorkspaceFadeInAdjacentScreens) {
              // In landscape mode the page spacing is set to the default.
              setPageSpacing(grid.edgeMarginPx);
          } else {
              // In portrait, we want the pages spaced such that there is no
              // overhang of the previous / next page into the current page viewport.
              // We assume symmetrical padding in portrait mode.
              int maxInsets = Math.max(insets.left, insets.right);
              int maxPadding = Math.max(grid.edgeMarginPx, padding.left + 1);
              setPageSpacing(Math.max(maxInsets, maxPadding));
          }
  
  
          int paddingLeftRight = grid.cellLayoutPaddingLeftRightPx;
          int paddingBottom = grid.cellLayoutBottomPaddingPx;
          for (int i = mWorkspaceScreens.size() - 1; i >= 0; i--) {
              mWorkspaceScreens.valueAt(i)
                      .setPadding(paddingLeftRight, 0, paddingLeftRight, paddingBottom);
          }
      }



```

通过在 Workspace 的构造相关代码发现，在 setOnTouchListener(new WorkspaceTouchListener(mLauncher, this));  
注册监听 OnTouch 事件，而具体处理是在 WorkspaceTouchListener 中处理长按事件，接下来  
分析下 WorkspaceTouchListener 的相关代码

3.2 WorkspaceTouchListener.java 相关长按事件分析
----------------------------------------

```
@Override
    public boolean onTouch(View view, MotionEvent ev) {
        mGestureDetector.onTouchEvent(ev);

        int action = ev.getActionMasked();
        if (action == ACTION_DOWN) {
            // Check if we can handle long press.
            boolean handleLongPress = canHandleLongPress();

            if (handleLongPress) {
                // Check if the event is not near the edges
                DeviceProfile dp = mLauncher.getDeviceProfile();
                DragLayer dl = mLauncher.getDragLayer();
                Rect insets = dp.getInsets();

                mTempRect.set(insets.left, insets.top, dl.getWidth() - insets.right,
                        dl.getHeight() - insets.bottom);
                mTempRect.inset(dp.edgeMarginPx, dp.edgeMarginPx);
                handleLongPress = mTempRect.contains((int) ev.getX(), (int) ev.getY());
            }

            if (handleLongPress) {
                mLongPressState = STATE_REQUESTED;
                mTouchDownPoint.set(ev.getX(), ev.getY());
            }

            mWorkspace.onTouchEvent(ev);
            // Return true to keep receiving touch events
            return true;
        }

        if (mLongPressState == STATE_PENDING_PARENT_INFORM) {
            // Inform the workspace to cancel touch handling
            ev.setAction(ACTION_CANCEL);
            mWorkspace.onTouchEvent(ev);

            ev.setAction(action);
            mLongPressState = STATE_COMPLETED;
        }

        final boolean result;
        if (mLongPressState == STATE_COMPLETED) {
            // We have handled the touch, so workspace does not need to know anything anymore.
            result = true;
        } else if (mLongPressState == STATE_REQUESTED) {
            mWorkspace.onTouchEvent(ev);
            if (mWorkspace.isHandlingTouch()) {
                cancelLongPress();
            } else if (action == ACTION_MOVE && PointF.length(
                    mTouchDownPoint.x - ev.getX(), mTouchDownPoint.y - ev.getY()) > mTouchSlop) {
                cancelLongPress();
            }

            result = true;
        } else {
            // We don't want to handle touch, let workspace handle it as usual.
            result = false;
        }

        if (action == ACTION_UP || action == ACTION_POINTER_UP) {
            if (!mWorkspace.isHandlingTouch()) {
                final CellLayout currentPage =
                        (CellLayout) mWorkspace.getChildAt(mWorkspace.getCurrentPage());
                if (currentPage != null) {
                    mWorkspace.onWallpaperTap(ev);
                }
            }
        }

        if (action == ACTION_UP || action == ACTION_CANCEL) {
            cancelLongPress();
        }

        return result;
    }

```

在触摸事件 onTouch(View view, MotionEvent ev) 中监听长按短按事件，然后判断是长按事件调用  
onLongPress 处理长按  
从 touch 事件中可以看到 canHandleLongPress 就是判断是否是长按事件的

```
@Override
public void onLongPress(MotionEvent event) {
    if (mLongPressState == STATE_REQUESTED) {
        if (canHandleLongPress()) {
            mLongPressState = STATE_PENDING_PARENT_INFORM;
            mWorkspace.getParent().requestDisallowInterceptTouchEvent(true);

            mWorkspace.performHapticFeedback(HapticFeedbackConstants.LONG_PRESS,
                    HapticFeedbackConstants.FLAG_IGNORE_VIEW_SETTING);
            mLauncher.getUserEventDispatcher().logActionOnContainer(Action.Touch.LONGPRESS,
                    Action.Direction.NONE, ContainerType.WORKSPACE,
                    mWorkspace.getCurrentPage());
            OptionsPopupView.showDefaultOptions(mLauncher, mTouchDownPoint.x, mTouchDownPoint.y);
        } else {
            cancelLongPress();
        }
    }
}

```

onLongPress 就是处理长按事件的  
从代码中看出

```
OptionsPopupView.showDefaultOptions(mLauncher, mTouchDownPoint.x, mTouchDownPoint.y);

```

就是负责处理长按弹窗

所以去掉弹窗 注释掉这段代码就行修改如下

```
@Override
public void onLongPress(MotionEvent event) {
    if (mLongPressState == STATE_REQUESTED) {
        if (canHandleLongPress()) {
            mLongPressState = STATE_PENDING_PARENT_INFORM;
            mWorkspace.getParent().requestDisallowInterceptTouchEvent(true);

            mWorkspace.performHapticFeedback(HapticFeedbackConstants.LONG_PRESS,
                    HapticFeedbackConstants.FLAG_IGNORE_VIEW_SETTING);
            mLauncher.getUserEventDispatcher().logActionOnContainer(Action.Touch.LONGPRESS,
                    Action.Direction.NONE, ContainerType.WORKSPACE,
                    mWorkspace.getCurrentPage());
          -  OptionsPopupView.showDefaultOptions(mLauncher, mTouchDownPoint.x, mTouchDownPoint.y);
        } else {
            cancelLongPress();
        }
    }
}

```
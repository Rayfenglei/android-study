> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/126810142)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.Launcher3 时钟动态图标的定制化的核心代码](#t1)

[3.Launcher3 时钟动态图标的定制化的代码功能分析](#t2)

 [3.1 BubbleTextView.java 关于图标的相关代码分析](#t3)

[3.2 定制绘制时间图标类](#t4)

[3.3 计算当前时间工具类](#t5)

[3.4 时钟工具类](#t6)

[3.5 时钟动态化图标定制主要修改为：](#t7)

1. 概述
-----

在 Launcher3 桌面的 app 列表页中，时钟图标默认是静态图标，但是由于时钟是时时刻刻在变化的，所以动态 图标更符合实际要求，所以根据需要定制动态时钟图标

2.Launcher3 时钟动态图标的定制化的核心代码
---------------------------

```
   packages/apps/Launcher3/src/com/android/launcher3/BubbleTextView.java
   packages/apps/Launcher3/src/com/android/launcher3/Launcher.java
```

3.Launcher3 时钟动态图标的定制化的代码功能分析
-----------------------------

###   3.1 BubbleTextView.java 关于图标的相关代码分析

```
    由于app列表图标都是由BubbleTextView来负责显示的 所以需要从这里来替换默认的时钟图标
     public class BubbleTextView extends TextView implements ItemInfoUpdateReceiver, OnResumeCallback,
         IconLabelDotView, DraggableView, Reorderable {
 
     private static final int DISPLAY_WORKSPACE = 0;
     private static final int DISPLAY_ALL_APPS = 1;
     private static final int DISPLAY_FOLDER = 2;
 
     private static final int[] STATE_PRESSED = new int[] {android.R.attr.state_pressed};
 
     private final PointF mTranslationForReorderBounce = new PointF(0, 0);
     private final PointF mTranslationForReorderPreview = new PointF(0, 0);
 
     private float mScaleForReorderBounce = 1f;
 
     private static final Property<BubbleTextView, Float> DOT_SCALE_PROPERTY
             = new Property<BubbleTextView, Float>(Float.TYPE, "dotScale") {
         @Override
         public Float get(BubbleTextView bubbleTextView) {
             return bubbleTextView.mDotParams.scale;
         }
 
         @Override
         public void set(BubbleTextView bubbleTextView, Float value) {
             bubbleTextView.mDotParams.scale = value;
             bubbleTextView.invalidate();
         }
      };
  
      public static final Property<BubbleTextView, Float> TEXT_ALPHA_PROPERTY
              = new Property<BubbleTextView, Float>(Float.class, "textAlpha") {
          @Override
          public Float get(BubbleTextView bubbleTextView) {
              return bubbleTextView.mTextAlpha;
          }
  
          @Override
          public void set(BubbleTextView bubbleTextView, Float alpha) {
              bubbleTextView.setTextAlpha(alpha);
          }
      };
  
      private final ActivityContext mActivity;
      private Drawable mIcon;
      private boolean mCenterVertically;
  
      private final int mDisplay;
  
      private final CheckLongPressHelper mLongPressHelper;
  
      private final boolean mLayoutHorizontal;
      private final int mIconSize;
  
      @ViewDebug.ExportedProperty(category = "launcher")
      private boolean mIsIconVisible = true;
      @ViewDebug.ExportedProperty(category = "launcher")
      private int mTextColor;
      @ViewDebug.ExportedProperty(category = "launcher")
      private float mTextAlpha = 1;
  
      @ViewDebug.ExportedProperty(category = "launcher")
      private DotInfo mDotInfo;
      private DotRenderer mDotRenderer;
      @ViewDebug.ExportedProperty(category = "launcher", deepExport = true)
      private DotRenderer.DrawParams mDotParams;
      private Animator mDotScaleAnim;
      private boolean mForceHideDot;
  
      @ViewDebug.ExportedProperty(category = "launcher")
      private boolean mStayPressed;
      @ViewDebug.ExportedProperty(category = "launcher")
      private boolean mIgnorePressedStateChange;
      @ViewDebug.ExportedProperty(category = "launcher")
      private boolean mDisableRelayout = false;
  
      private IconLoadRequest mIconLoadRequest;
  
      public BubbleTextView(Context context) {
          this(context, null, 0);
      }
  
      public BubbleTextView(Context context, AttributeSet attrs) {
          this(context, attrs, 0);
      }
  
      public BubbleTextView(Context context, AttributeSet attrs, int defStyle) {
          super(context, attrs, defStyle);
          mActivity = ActivityContext.lookupContext(context);
  
          TypedArray a = context.obtainStyledAttributes(attrs,
                  R.styleable.BubbleTextView, defStyle, 0);
          mLayoutHorizontal = a.getBoolean(R.styleable.BubbleTextView_layoutHorizontal, false);
          DeviceProfile grid = mActivity.getDeviceProfile();
  
          mDisplay = a.getInteger(R.styleable.BubbleTextView_iconDisplay, DISPLAY_WORKSPACE);
          final int defaultIconSize;
          if (mDisplay == DISPLAY_WORKSPACE) {
              setTextSize(TypedValue.COMPLEX_UNIT_PX, grid.iconTextSizePx);
              setCompoundDrawablePadding(grid.iconDrawablePaddingPx);
              defaultIconSize = grid.iconSizePx;
          } else if (mDisplay == DISPLAY_ALL_APPS) {
              setTextSize(TypedValue.COMPLEX_UNIT_PX, grid.allAppsIconTextSizePx);
              setCompoundDrawablePadding(grid.allAppsIconDrawablePaddingPx);
              defaultIconSize = grid.allAppsIconSizePx;
          } else if (mDisplay == DISPLAY_FOLDER) {
              setTextSize(TypedValue.COMPLEX_UNIT_PX, grid.folderChildTextSizePx);
              setCompoundDrawablePadding(grid.folderChildDrawablePaddingPx);
              defaultIconSize = grid.folderChildIconSizePx;
          } else {
              // widget_selection or shortcut_popup
              defaultIconSize = grid.iconSizePx;
          }
  
          mCenterVertically = a.getBoolean(R.styleable.BubbleTextView_centerVertically, false);
  
          mIconSize = a.getDimensionPixelSize(R.styleable.BubbleTextView_iconSizeOverride,
                  defaultIconSize);
          a.recycle();
  
          mLongPressHelper = new CheckLongPressHelper(this);
  
          mDotParams = new DotRenderer.DrawParams();
  
          setEllipsize(TruncateAt.END);
          setAccessibilityDelegate(mActivity.getAccessibilityDelegate());
          setTextAlpha(1f);
      }
  
      @Override
      protected void onFocusChanged(boolean focused, int direction, Rect previouslyFocusedRect) {
          // Disable marques when not focused to that, so that updating text does not cause relayout.
          setEllipsize(focused ? TruncateAt.MARQUEE : TruncateAt.END);
          super.onFocusChanged(focused, direction, previouslyFocusedRect);
      }
  
      /**
       * Resets the view so it can be recycled.
       */
      public void reset() {
          mDotInfo = null;
          mDotParams.color = Color.TRANSPARENT;
          cancelDotScaleAnim();
          mDotParams.scale = 0f;
          mForceHideDot = false;
          setBackground(null);
      }
  
      private void cancelDotScaleAnim() {
          if (mDotScaleAnim != null) {
              mDotScaleAnim.cancel();
          }
      }
  
      private void animateDotScale(float... dotScales) {
          cancelDotScaleAnim();
          mDotScaleAnim = ObjectAnimator.ofFloat(this, DOT_SCALE_PROPERTY, dotScales);
          mDotScaleAnim.addListener(new AnimatorListenerAdapter() {
              @Override
              public void onAnimationEnd(Animator animation) {
                  mDotScaleAnim = null;
              }
          });
          mDotScaleAnim.start();
      }
  
      public void applyFromWorkspaceItem(WorkspaceItemInfo info) {
          applyFromWorkspaceItem(info, false);
      }
  
      @Override
      public void setAccessibilityDelegate(AccessibilityDelegate delegate) {
          if (delegate instanceof LauncherAccessibilityDelegate) {
              super.setAccessibilityDelegate(delegate);
          } else {
              // NO-OP
              // Workaround for b/129745295 where RecyclerView is setting our Accessibility
              // delegate incorrectly. There are no cases when we shouldn't be using the
              // LauncherAccessibilityDelegate for BubbleTextView.
          }
      }
  
      public void applyFromWorkspaceItem(WorkspaceItemInfo info, boolean promiseStateChanged) {
          applyIconAndLabel(info);
          setTag(info);
          if (promiseStateChanged || (info.hasPromiseIconUi())) {
              applyPromiseState(promiseStateChanged);
          }
  
          applyDotState(info, false /* animate */);
      }
  
      public void applyFromApplicationInfo(AppInfo info) {
          //根据info信息 构建app图标
          applyIconAndLabel(info);
  
          // We don't need to check the info since it's not a WorkspaceItemInfo
          super.setTag(info);
  
          // Verify high res immediately
          verifyHighRes();
  
          if (info instanceof PromiseAppInfo) {
              PromiseAppInfo promiseAppInfo = (PromiseAppInfo) info;
              applyProgressLevel(promiseAppInfo.level);
          }
          applyDotState(info, false /* animate */);
      }
  
      public void applyFromPackageItemInfo(PackageItemInfo info) {
          applyIconAndLabel(info);
          // We don't need to check the info since it's not a WorkspaceItemInfo
          super.setTag(info);
  
          // Verify high res immediately
          verifyHighRes();
      }
  
      private void applyIconAndLabel(ItemInfoWithIcon info) {
          FastBitmapDrawable iconDrawable = newIcon(getContext(), info);
          mDotParams.color = IconPalette.getMutedColor(info.bitmap.color, 0.54f);
  
          setIcon(iconDrawable);
          setText(info.title);
          if (info.contentDescription != null) {
              setContentDescription(info.isDisabled()
                      ? getContext().getString(R.string.disabled_app_label, info.contentDescription)
                      : info.contentDescription);
          }
      }
  
      /**
       * Overrides the default long press timeout.
       */
      public void setLongPressTimeoutFactor(float longPressTimeoutFactor) {
          mLongPressHelper.setLongPressTimeoutFactor(longPressTimeoutFactor);
      }
  
      @Override
      public void refreshDrawableState() {
          if (!mIgnorePressedStateChange) {
              super.refreshDrawableState();
          }
      }
  
      @Override
      protected int[] onCreateDrawableState(int extraSpace) {
          final int[] drawableState = super.onCreateDrawableState(extraSpace + 1);
          if (mStayPressed) {
              mergeDrawableStates(drawableState, STATE_PRESSED);
          }
          return drawableState;
      }
  
      /** Returns the icon for this view. */
      public Drawable getIcon() {
          return mIcon;
      }
  
      @Override
      public boolean onTouchEvent(MotionEvent event) {
          // ignore events if they happen in padding area
          if (event.getAction() == MotionEvent.ACTION_DOWN
                  && shouldIgnoreTouchDown(event.getX(), event.getY())) {
              return false;
          }
          if (isLongClickable()) {
              super.onTouchEvent(event);
              mLongPressHelper.onTouchEvent(event);
              // Keep receiving the rest of the events
              return true;
          } else {
              return super.onTouchEvent(event);
          }
      }
  
      /**
       * Returns true if the touch down at the provided position be ignored
       */
      protected boolean shouldIgnoreTouchDown(float x, float y) {
          return y < getPaddingTop()
                  || x < getPaddingLeft()
                  || y > getHeight() - getPaddingBottom()
                  || x > getWidth() - getPaddingRight();
      }
  
      void setStayPressed(boolean stayPressed) {
          mStayPressed = stayPressed;
          refreshDrawableState();
      }
  
      @Override
      public void onVisibilityAggregated(boolean isVisible) {
          super.onVisibilityAggregated(isVisible);
          if (mIcon != null) {
              mIcon.setVisible(isVisible, false);
          }
      }
  
      @Override
      public void onLauncherResume() {
          // Reset the pressed state of icon that was locked in the press state while activity
          // was launching
          setStayPressed(false);
      }
  
      void clearPressedBackground() {
          setPressed(false);
          setStayPressed(false);
      }
  
      @Override
      public boolean onKeyUp(int keyCode, KeyEvent event) {
          // Unlike touch events, keypress event propagate pressed state change immediately,
          // without waiting for onClickHandler to execute. Disable pressed state changes here
          // to avoid flickering.
          mIgnorePressedStateChange = true;
          boolean result = super.onKeyUp(keyCode, event);
          mIgnorePressedStateChange = false;
          refreshDrawableState();
          return result;
      }
  
      @SuppressWarnings("wrongcall")
      protected void drawWithoutDot(Canvas canvas) {
          super.onDraw(canvas);
      }
  
      @Override
      public void onDraw(Canvas canvas) {
          super.onDraw(canvas);
          drawDotIfNecessary(canvas);
      }
  
      /**
       * Draws the notification dot in the top right corner of the icon bounds.
       * @param canvas The canvas to draw to.
       */
      protected void drawDotIfNecessary(Canvas canvas) {
          if (!mForceHideDot && (hasDot() || mDotParams.scale > 0)) {
              getIconBounds(mDotParams.iconBounds);
              Utilities.scaleRectAboutCenter(mDotParams.iconBounds, IconShape.getNormalizationScale());
              final int scrollX = getScrollX();
              final int scrollY = getScrollY();
              canvas.translate(scrollX, scrollY);
              mDotRenderer.draw(canvas, mDotParams);
              canvas.translate(-scrollX, -scrollY);
          }
      }
  
      @Override
      public void setForceHideDot(boolean forceHideDot) {
          if (mForceHideDot == forceHideDot) {
              return;
          }
          mForceHideDot = forceHideDot;
  
          if (forceHideDot) {
              invalidate();
          } else if (hasDot()) {
              animateDotScale(0, 1);
          }
      }
  
      private boolean hasDot() {
          return mDotInfo != null;
      }
  
      public void getIconBounds(Rect outBounds) {
          getIconBounds(this, outBounds, mIconSize);
      }
  
      public static void getIconBounds(View iconView, Rect outBounds, int iconSize) {
          int top = iconView.getPaddingTop();
          int left = (iconView.getWidth() - iconSize) / 2;
          int right = left + iconSize;
          int bottom = top + iconSize;
          outBounds.set(left, top, right, bottom);
      }
  
      @Override
      protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
          if (mCenterVertically) {
              Paint.FontMetrics fm = getPaint().getFontMetrics();
              int cellHeightPx = mIconSize + getCompoundDrawablePadding() +
                      (int) Math.ceil(fm.bottom - fm.top);
              int height = MeasureSpec.getSize(heightMeasureSpec);
              setPadding(getPaddingLeft(), (height - cellHeightPx) / 2, getPaddingRight(),
                      getPaddingBottom());
          }
          super.onMeasure(widthMeasureSpec, heightMeasureSpec);
      }
  
      @Override
      public void setTextColor(int color) {
          mTextColor = color;
          super.setTextColor(getModifiedColor());
      }
  
      @Override
      public void setTextColor(ColorStateList colors) {
          mTextColor = colors.getDefaultColor();
          if (Float.compare(mTextAlpha, 1) == 0) {
              super.setTextColor(colors);
          } else {
              super.setTextColor(getModifiedColor());
          }
      }
  
      public boolean shouldTextBeVisible() {
          // Text should be visible everywhere but the hotseat.
          Object tag = getParent() instanceof FolderIcon ? ((View) getParent()).getTag() : getTag();
          ItemInfo info = tag instanceof ItemInfo ? (ItemInfo) tag : null;
          return info == null || (info.container != LauncherSettings.Favorites.CONTAINER_HOTSEAT
                  && info.container != LauncherSettings.Favorites.CONTAINER_HOTSEAT_PREDICTION);
      }
  
      public void setTextVisibility(boolean visible) {
          setTextAlpha(visible ? 1 : 0);
      }
  
      private void setTextAlpha(float alpha) {
          mTextAlpha = alpha;
          super.setTextColor(getModifiedColor());
      }
  
      private int getModifiedColor() {
          if (mTextAlpha == 0) {
              // Special case to prevent text shadows in high contrast mode
              return Color.TRANSPARENT;
          }
          return setColorAlphaBound(mTextColor, Math.round(Color.alpha(mTextColor) * mTextAlpha));
      }
  
      /**
       * Creates an animator to fade the text in or out.
       * @param fadeIn Whether the text should fade in or fade out.
       */
      public ObjectAnimator createTextAlphaAnimator(boolean fadeIn) {
          float toAlpha = shouldTextBeVisible() && fadeIn ? 1 : 0;
          return ObjectAnimator.ofFloat(this, TEXT_ALPHA_PROPERTY, toAlpha);
      }
  
      @Override
      public void cancelLongPress() {
          super.cancelLongPress();
          mLongPressHelper.cancelLongPress();
      }
  
      public void applyPromiseState(boolean promiseStateChanged) {
          if (getTag() instanceof WorkspaceItemInfo) {
              WorkspaceItemInfo info = (WorkspaceItemInfo) getTag();
              final boolean isPromise = info.hasPromiseIconUi();
              final int progressLevel = isPromise ?
                      ((info.hasStatusFlag(WorkspaceItemInfo.FLAG_INSTALL_SESSION_ACTIVE) ?
                              info.getInstallProgress() : 0)) : 100;
  
              PreloadIconDrawable preloadDrawable = applyProgressLevel(progressLevel);
              if (preloadDrawable != null && promiseStateChanged) {
                  preloadDrawable.maybePerformFinishedAnimation();
              }
          }
      }
  
      public PreloadIconDrawable applyProgressLevel(int progressLevel) {
          if (getTag() instanceof ItemInfoWithIcon) {
              ItemInfoWithIcon info = (ItemInfoWithIcon) getTag();
              if (progressLevel >= 100) {
                  setContentDescription(info.contentDescription != null
                          ? info.contentDescription : "");
              } else if (progressLevel > 0) {
                  setContentDescription(getContext()
                          .getString(R.string.app_downloading_title, info.title,
                                  NumberFormat.getPercentInstance().format(progressLevel * 0.01)));
              } else {
                  setContentDescription(getContext()
                          .getString(R.string.app_waiting_download_title, info.title));
              }
              if (mIcon != null) {
                  final PreloadIconDrawable preloadDrawable;
                  if (mIcon instanceof PreloadIconDrawable) {
                      preloadDrawable = (PreloadIconDrawable) mIcon;
                      preloadDrawable.setLevel(progressLevel);
                  } else {
                      preloadDrawable = newPendingIcon(getContext(), info);
                      preloadDrawable.setLevel(progressLevel);
                      setIcon(preloadDrawable);
                  }
                  return preloadDrawable;
              }
          }
          return null;
      }
  
      public void applyDotState(ItemInfo itemInfo, boolean animate) {
          if (mIcon instanceof FastBitmapDrawable) {
              boolean wasDotted = mDotInfo != null;
              mDotInfo = mActivity.getDotInfoForItem(itemInfo);
              boolean isDotted = mDotInfo != null;
              float newDotScale = isDotted ? 1f : 0;
              if (mDisplay == DISPLAY_ALL_APPS) {
                  mDotRenderer = mActivity.getDeviceProfile().mDotRendererAllApps;
              } else {
                  mDotRenderer = mActivity.getDeviceProfile().mDotRendererWorkSpace;
              }
              if (wasDotted || isDotted) {
                  // Animate when a dot is first added or when it is removed.
                  if (animate && (wasDotted ^ isDotted) && isShown()) {
                      animateDotScale(newDotScale);
                  } else {
                      cancelDotScaleAnim();
                      mDotParams.scale = newDotScale;
                      invalidate();
                  }
              }
              if (itemInfo.contentDescription != null) {
                  if (itemInfo.isDisabled()) {
                      setContentDescription(getContext().getString(R.string.disabled_app_label,
                              itemInfo.contentDescription));
                  } else if (hasDot()) {
                      int count = mDotInfo.getNotificationCount();
                      setContentDescription(getContext().getResources().getQuantityString(
                              R.plurals.dotted_app_label, count, itemInfo.contentDescription, count));
                  } else {
                      setContentDescription(itemInfo.contentDescription);
                  }
              }
          }
      }
  
      /**
       * Sets the icon for this view based on the layout direction.
       */
      private void setIcon(Drawable icon) {
          if (mIsIconVisible) {
              applyCompoundDrawables(icon);
          }
          mIcon = icon;
          if (mIcon != null) {
              mIcon.setVisible(getWindowVisibility() == VISIBLE && isShown(), false);
          }
      }
  
      @Override
      public void setIconVisible(boolean visible) {
          mIsIconVisible = visible;
          if (!mIsIconVisible) {
              resetIconScale();
          }
          Drawable icon = visible ? mIcon : new ColorDrawable(Color.TRANSPARENT);
          applyCompoundDrawables(icon);
      }
  
      protected void applyCompoundDrawables(Drawable icon) {
          // If we had already set an icon before, disable relayout as the icon size is the
          // same as before.
          mDisableRelayout = mIcon != null;
  
          icon.setBounds(0, 0, mIconSize, mIconSize);
          if (mLayoutHorizontal) {
              setCompoundDrawablesRelative(icon, null, null, null);
          } else {
              setCompoundDrawables(null, icon, null, null);
          }
          mDisableRelayout = false;
      }
  
      @Override
      public void requestLayout() {
          if (!mDisableRelayout) {
              super.requestLayout();
          }
      }
  
      /**
       * Applies the item info if it is same as what the view is pointing to currently.
       */
      @Override
      public void reapplyItemInfo(ItemInfoWithIcon info) {
          if (getTag() == info) {
              mIconLoadRequest = null;
              mDisableRelayout = true;
  
              // Optimization: Starting in N, pre-uploads the bitmap to RenderThread.
              info.bitmap.icon.prepareToDraw();
  
              if (info instanceof AppInfo) {
                 // 负责加载构建app图标
                  applyFromApplicationInfo((AppInfo) info);
              } else if (info instanceof WorkspaceItemInfo) {
                  applyFromWorkspaceItem((WorkspaceItemInfo) info);
                  mActivity.invalidateParent(info);
              } else if (info instanceof PackageItemInfo) {
                  applyFromPackageItemInfo((PackageItemInfo) info);
              }
  
              mDisableRelayout = false;
          }
      }
  
      /**
       * Verifies that the current icon is high-res otherwise posts a request to load the icon.
       */
      public void verifyHighRes() {
          if (mIconLoadRequest != null) {
              mIconLoadRequest.cancel();
              mIconLoadRequest = null;
          }
          if (getTag() instanceof ItemInfoWithIcon) {
              ItemInfoWithIcon info = (ItemInfoWithIcon) getTag();
              if (info.usingLowResIcon()) {
                  mIconLoadRequest = LauncherAppState.getInstance(getContext()).getIconCache()
                          .updateIconInBackground(BubbleTextView.this, info);
              }
          }
      }
  
      public int getIconSize() {
          return mIconSize;
      }
  
      private void updateTranslation() {
          super.setTranslationX(mTranslationForReorderBounce.x + mTranslationForReorderPreview.x);
          super.setTranslationY(mTranslationForReorderBounce.y + mTranslationForReorderPreview.y);
      }
  
      public void setReorderBounceOffset(float x, float y) {
          mTranslationForReorderBounce.set(x, y);
          updateTranslation();
      }
  
      public void getReorderBounceOffset(PointF offset) {
          offset.set(mTranslationForReorderBounce);
      }
  
      @Override
      public void setReorderPreviewOffset(float x, float y) {
          mTranslationForReorderPreview.set(x, y);
          updateTranslation();
      }
  
      @Override
      public void getReorderPreviewOffset(PointF offset) {
          offset.set(mTranslationForReorderPreview);
      }
  
      public void setReorderBounceScale(float scale) {
          mScaleForReorderBounce = scale;
          super.setScaleX(scale);
          super.setScaleY(scale);
      }
  
      public float getReorderBounceScale() {
          return mScaleForReorderBounce;
      }
  
      public View getView() {
          return this;
      }
  
      @Override
      public int getViewType() {
          return DRAGGABLE_ICON;
      }
  
      @Override
      public void getWorkspaceVisualDragBounds(Rect bounds) {
          DeviceProfile grid = mActivity.getDeviceProfile();
          BubbleTextView.getIconBounds(this, bounds, grid.iconSizePx);
      }
  
      private int getIconSizeForDisplay(int display) {
          DeviceProfile grid = mActivity.getDeviceProfile();
          switch (display) {
              case DISPLAY_ALL_APPS:
                  return grid.allAppsIconSizePx;
              case DISPLAY_WORKSPACE:
              case DISPLAY_FOLDER:
              default:
                  return grid.iconSizePx;
          }
      }
  
      public void getSourceVisualDragBounds(Rect bounds) {
          BubbleTextView.getIconBounds(this, bounds, getIconSizeForDisplay(mDisplay));
      }
  
      @Override
      public SafeCloseable prepareDrawDragView() {
          resetIconScale();
          setForceHideDot(true);
          return () -> { };
      }
  
      private void resetIconScale() {
          if (mIcon instanceof FastBitmapDrawable) {
              ((FastBitmapDrawable) mIcon).setScale(1f);
          }
      }
  }
reapplyItemInfo(）负责加载appInfo workspaceitem PackageItemInfo的相关信息
```

### 3.2 定制绘制时间图标类

```
    package com.android.launcher3;
import android.content.Context;
import android.graphics.*;
import com.android.launcher3.R;
public class IconUtil {
private static final String TAG = "IconUtil";
private static Bitmap getBitmap(Context context, int res) {
BitmapFactory.Options options = new BitmapFactory.Options();
options.inPreferredConfig = Bitmap.Config.ARGB_4444;
return BitmapFactory.decodeResource(context.getResources(), res, options);
}
public static Bitmap getDeskClockIcon(Context context) {
Bitmap empty = getBitmap(context, R.drawable.icon_time);
int x = empty.getWidth();
int y = empty.getHeight();
Bitmap deskClock = Bitmap.createBitmap(x, y, Bitmap.Config.ARGB_4444);
Canvas canvas = new Canvas(deskClock);
Paint paint = new Paint();
paint.setAntiAlias(true);
canvas.drawBitmap(empty, 0, 0, paint);
paint.setStrokeCap(Paint.Cap.ROUND);
paint.setStrokeWidth(30);
paint.setColor(Color.parseColor("#FFA500"));
int radius = 200;
int cx = x / 2;
int cy = y / 2;
int hour = Integer.parseInt(DateUtils.getCurrentHour());
int min = Integer.parseInt(DateUtils.getCurrentMin());
//时针的角度，这里是整点的角度。因为0°是从3点开始，所以这里减90°，从9点开始计算角度
int drgeeHour = hour * 30 - 90;
if (drgeeHour < 0) {
drgeeHour += 360;
}
// 加上时针在两个整点之间的角度，一分钟在分针上是6°，在时针上是min * 6 / 12
drgeeHour += min * 6 / 12;
//时针 针尖的x y坐标，相当于已知圆心坐标和半径，求圆上任意一点的坐标
int xHour = (int) (cx + radius * Math.cos(drgeeHour * 3.14 / 180));
int yHour = (int) (cy + radius * Math.sin(drgeeHour * 3.14 / 180));
canvas.drawLine(xHour, yHour, cx, cy, paint);
//分针的长度
radius = 260;
paint.setStrokeWidth(20);
paint.setColor(Color.RED);
//分针的角度
int drgeeMin = min * 6 - 90;
if (drgeeMin < 0) {
drgeeMin += 360;
}
//分针 针尖的x y坐标
int x1 = (int) (cx + radius * Math.cos(drgeeMin * Math.PI / 180));
int y1 = (int) (cy + radius * Math.sin(drgeeMin * Math.PI / 180));
canvas.drawLine(x1, y1, cx, cy, paint);
return deskClock;
}
}
```

### 3.3 计算当前时间工具类

```
  package com.android.launcher3;
import java.text.SimpleDateFormat;
public class DateUtils {
public static String getCurrentDay() {
SimpleDateFormat format = new SimpleDateFormat("dd");
Long t = new Long(System.currentTimeMillis());
String d = format.format(t);
return d;
}
public static String getCurrentSecond() {
SimpleDateFormat format = new SimpleDateFormat("ss");
Long t = new Long(System.currentTimeMillis());
String d = format.format(t);
return d;
}
public static String getCurrentMin() {
SimpleDateFormat format = new SimpleDateFormat("mm");
Long t = new Long(System.currentTimeMillis());
String d = format.format(t);
return d;
}
public static String getCurrentHour() {
SimpleDateFormat format = new SimpleDateFormat("HH");
Long t = new Long(System.currentTimeMillis());
String d = format.format(t);
return d;
}
}
```

###   
3.4 时钟工具类

```
  package com.android.launcher3;
import android.content.Context;
import android.graphics.Bitmap;
import android.os.Handler;
import android.os.Message;
import com.android.launcher3.LauncherSettings;
public class DeskClockUtil {
private OnDeskClockIconChangeListener mListener;
private boolean mIsResume;
private Handler mHandler = new Handler() {
@Override
public void handleMessage(Message msg) {
super.handleMessage(msg);
if (msg.what == 100) {
Message msg1 = new Message();
msg1.what = 100;
msg1.obj = msg.obj;
mHandler.sendMessageDelayed(msg1, 60000);
if (mListener != null) {
mListener.onChange(IconUtil.getDeskClockIcon((Context) msg.obj));
}
}
}
};
private static DeskClockUtil sInstance;
private DeskClockUtil() {
}
public static DeskClockUtil getInstance() {
if (sInstance == null) {
sInstance = new DeskClockUtil();
}
return sInstance;
}
private void refresh(Context context) {
if (mListener != null) {
mListener.onChange(IconUtil.getDeskClockIcon(context));
}
if (mHandler.hasMessages(100)) {
mHandler.removeMessages(100);
}
Message msg = new Message();
msg.what = 100;
msg.obj = context;
mHandler.sendMessageDelayed(msg,
1000 * (60 - Integer.parseInt(DateUtils.getCurrentSecond())));
}
public void onResume(Context context) {
mIsResume = true;
refresh(context);
}
public void onPause() {
mIsResume = false;
mHandler.removeMessages(100);
}
public void setListener(OnDeskClockIconChangeListener listener,Context context) {
mListener = listener;
if (mIsResume) {
refresh(context);
}
}
public interface OnDeskClockIconChangeListener {
void onChange(Bitmap icon);
}
}
```

### 3.5 时钟动态化图标定制主要修改为：

```
private void applyIconAndLabel(ItemInfoWithIcon info) {
String pkgname = "";
if (info.getIntent() != null && info.getIntent().getComponent() != null) {
pkgname = info.getIntent().getComponent().getPackageName();
}
android.util.Log.e("Launcher3","pkgname:"+pkgname);
FastBitmapDrawable iconDrawable = null;
if(pkgname.equals("com.android.deskclock")){
DeskClockUtil.getInstance().setListener(new DeskClockUtil.OnDeskClockIconChangeListener() {
@Override
public void onChange(Bitmap icon) {
FastBitmapDrawable deskiconDrawable = new FastBitmapDrawable(icon);
android.util.Log.e("Launcher3","deskiconDrawable:"+deskiconDrawable+"--icon:"+icon);
if(deskiconDrawable!=null)setIcon(deskiconDrawable);
}
}, getContext());
}else{
iconDrawable = DrawableFactory.INSTANCE.get(getContext())
.newIcon(getContext(), info);
}
//FastBitmapDrawable iconDrawable = DrawableFactory.INSTANCE.get(getContext())
//.newIcon(getContext(), info);
mDotParams.color = IconPalette.getMutedColor(info.iconColor, 0.54f);
    if(iconDrawable!=null)setIcon(iconDrawable);
    setText(info.title);
    if (info.contentDescription != null) {
        setContentDescription(info.isDisabled()
                ? getContext().getString(R.string.disabled_app_label, info.contentDescription)
                : info.contentDescription);
    }
}
 
Launcher.java 启动和暂停动态时钟
 
protected void onResume() {
......
//add core start
DeskClockUtil.getInstance().onResume(this);
//add core end
 
if (mLauncherCallbacks != null) {
mLauncherCallbacks.onResume();
}
}
@Override
protected void onPause() {
//add core start
DeskClockUtil.getInstance().onPause();
//add core end
 
....
}
```
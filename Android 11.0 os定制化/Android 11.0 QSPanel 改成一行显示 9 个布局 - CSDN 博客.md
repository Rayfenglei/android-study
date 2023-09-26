> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124760435)

### 1. 概述

在 11.0 的产品开发中，对 SystemUI 下拉状态栏做定制需求，有功能要求在下拉状态栏第二次展开时，显示 QSPanel 快捷功能键默认是 3 行 3 列显示的 现在由于功能的需要修改为 1 行 9 列显示  
这样就需要看详细布局的代码

### 2.QSPanel 改成一行显示 9 个布局的核心类

```
frameworks/base/packages/SystemUI/src/com/android/systemui/qs/QSPanel.java
frameworks/base/packages/SystemUI/src/com/android/systemui/qs/TileLayout.java

```

### 3.QSPanel 改成一行显示 9 个布局的核心功能分析和实现

### 3.1QSPanel.java 的相关布局的核心功能分析

```
public class QSPanel extends LinearLayout implements Tunable, Callback, BrightnessMirrorListener,
         Dumpable {
 
     public static final String QS_SHOW_BRIGHTNESS = "qs_show_brightness";
     public static final String QS_SHOW_HEADER = "qs_show_header";
 
     private static final String TAG = "QSPanel";
 
     protected final Context mContext;
     protected final ArrayList<TileRecord> mRecords = new ArrayList<>();
     protected final View mBrightnessView;
     private final H mHandler = new H();
     private final MetricsLogger mMetricsLogger = Dependency.get(MetricsLogger.class);
     private final QSTileRevealController mQsTileRevealController;
 
     protected boolean mExpanded;
     protected boolean mListening;
 
     private QSDetail.Callback mCallback;
     private BrightnessController mBrightnessController;
     private DumpController mDumpController;
     protected QSTileHost mHost;
 
     protected QSSecurityFooter mFooter;
     private PageIndicator mFooterPageIndicator;
     private boolean mGridContentVisible = true;
 
     protected QSTileLayout mTileLayout;
 
     private QSCustomizer mCustomizePanel;
     private Record mDetailRecord;
 
     private BrightnessMirrorController mBrightnessMirrorController;
     private View mDivider;
  
      public QSPanel(Context context) {
          this(context, null);
      }
  
      public QSPanel(Context context, AttributeSet attrs) {
          this(context, attrs, null);
      }
  
      @Inject
      public QSPanel(@Named(VIEW_CONTEXT) Context context, AttributeSet attrs,
              DumpController dumpController) {
          super(context, attrs);
          mContext = context;
  
          setOrientation(VERTICAL);
  
          mBrightnessView = LayoutInflater.from(mContext).inflate(
              R.layout.quick_settings_brightness_dialog, this, false);
          addView(mBrightnessView);
  
          mTileLayout = (QSTileLayout) LayoutInflater.from(mContext).inflate(
                  R.layout.qs_paged_tile_layout, this, false);
          mTileLayout.setListening(mListening);
          addView((View) mTileLayout);
  
          mQsTileRevealController = new QSTileRevealController(mContext, this,
                  (PagedTileLayout) mTileLayout);
  
          addDivider();
  
          mFooter = new QSSecurityFooter(this, context);
          addView(mFooter.getView());
  
          updateResources();
  
          mBrightnessController = new BrightnessController(getContext(),
                  findViewById(R.id.brightness_slider));
          mDumpController = dumpController;
      }
  
      protected void addDivider() {
          mDivider = LayoutInflater.from(mContext).inflate(R.layout.qs_divider, this, false);
          mDivider.setBackgroundColor(Utils.applyAlpha(mDivider.getAlpha(),
                  getColorForState(mContext, Tile.STATE_ACTIVE)));
          addView(mDivider);
      }
  
      @Override
      protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
          // We want all the logic of LinearLayout#onMeasure, and for it to assign the excess space
          // not used by the other children to PagedTileLayout. However, in this case, LinearLayout
          // assumes that PagedTileLayout would use all the excess space. This is not the case as
          // PagedTileLayout height is quantized (because it shows a certain number of rows).
          // Therefore, after everything is measured, we need to make sure that we add up the correct
          // total height
          super.onMeasure(widthMeasureSpec, heightMeasureSpec);
          int height = getPaddingBottom() + getPaddingTop();
          int numChildren = getChildCount();
          for (int i = 0; i < numChildren; i++) {
              View child = getChildAt(i);
              if (child.getVisibility() != View.GONE) height += child.getMeasuredHeight();
          }
          setMeasuredDimension(getMeasuredWidth(), height);
      }
  
      public View getDivider() {
          return mDivider;
      }
  
      public QSTileRevealController getQsTileRevealController() {
          return mQsTileRevealController;
      }
  
      public boolean isShowingCustomize() {
          return mCustomizePanel != null && mCustomizePanel.isCustomizing();
      }
  
      @Override
      protected void onAttachedToWindow() {
          super.onAttachedToWindow();
          final TunerService tunerService = Dependency.get(TunerService.class);
          tunerService.addTunable(this, QS_SHOW_BRIGHTNESS);
  
          if (mHost != null) {
              setTiles(mHost.getTiles());
          }
          if (mBrightnessMirrorController != null) {
              mBrightnessMirrorController.addCallback(this);
          }
          if (mDumpController != null) mDumpController.addListener(this);
      }
  
      @Override
      protected void onDetachedFromWindow() {
          Dependency.get(TunerService.class).removeTunable(this);
          if (mHost != null) {
              mHost.removeCallback(this);
          }
          for (TileRecord record : mRecords) {
              record.tile.removeCallbacks();
          }
          if (mBrightnessMirrorController != null) {
              mBrightnessMirrorController.removeCallback(this);
          }
          if (mDumpController != null) mDumpController.removeListener(this);
          super.onDetachedFromWindow();
      }

```

从上述的源码中可以看出在 QSPanel(）的构造方法中可以得出 QSPanel 的布局属性 mTileLayout 就是加载的 QSTileLayout 的  
相关布局

```
          mTileLayout = (QSTileLayout) LayoutInflater.from(mContext).inflate(
                  R.layout.qs_paged_tile_layout, this, false);

```

看出 mTileLayout 为每页的布局  
mTileLayout 又是实现 QSTileLayout 的一些接口  
经过追踪代码发现 布局其实有 frameworks/base/packages/SystemUI/src/com/android/systemui/qs/TileLayout.java  
来实现的

### 3.2 TileLayout.java 的相关布局分析

在 TileLayout.java 的相关代码中可以发现在 updateResources() 中负责更新相关的布局参数  
而 mMaxAllowedRows 就是代表 TileLayout 布局的行数  
而 mResourceColumns 就是代表 TileLayout 布局的列数  
所以修改 1 行 9 列布局就需要修改这两个值就可以了，具体修改为:

```
public boolean updateResources() {
         final Resources res = mContext.getResources();
          mResourceColumns = 9/*Math.max(1, res.getInteger(R.integer.quick_settings_num_columns))*/;
          mCellHeight = mContext.getResources().getDimensionPixelSize(R.dimen.qs_tile_height);
          mCellMarginHorizontal = res.getDimensionPixelSize(R.dimen.qs_tile_margin_horizontal);
          mCellMarginVertical= res.getDimensionPixelSize(R.dimen.qs_tile_margin_vertical);
          mCellMarginTop = res.getDimensionPixelSize(R.dimen.qs_tile_margin_top);
          mMaxAllowedRows = 1/*Math.max(1, getResources().getInteger(R.integer.quick_settings_max_rows))*/;
          if (mLessRows) mMaxAllowedRows = Math.max(mMinRows, mMaxAllowedRows - 1);
          if (updateColumns()) {
              requestLayout();
              return true;
          }
          return false;
      }

```

修改 mMaxAllowedRows 和 mResourceColumns 的值 经过编译发现这个值就改变了，
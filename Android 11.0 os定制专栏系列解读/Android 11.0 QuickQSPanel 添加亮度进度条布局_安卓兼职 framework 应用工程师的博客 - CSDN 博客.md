> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124739108)

### 1. 概述

在 11.0 产品进行定制化开发时，在定制化 framework 上层的 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 虽然和 10.0 有些差别 但是思路是一样的，差别不大而在 QuickQSPanel 中添加系统亮度条布局思路也是一样的，就是需要在 QuickQSPanel 中增加系统的亮度条布局就可以了

### 2.QuickQSPanel 添加亮度进度条布局的相关核心类

```
frameworks / base / packages / SystemUI / src / com / android / 
systemui / qs / QuickQSPanel.java

```

### 3.QuickQSPanel 添加亮度进度条布局的核心功能分析和实现

在下拉状态栏中，首次下拉的类就是 QuickQSPanel，接下来看 QuickQSPanel 的相关方法看如何添加亮度条布局  
具体实现如下:  
首选看 QuickQSPanel.java 的布局文件  
路径为: frameworks / base / packages / SystemUI / src / com / android / systemui / qs / QuickQSPanel.java

```
public class QuickQSPanel extends QSPanel {
 
     public static final String NUM_QUICK_TILES = "sysui_qqs_count";
     private static final String TAG = "QuickQSPanel";
     // Start it at 6 so a non-zero value can be obtained statically.
     private static int sDefaultMaxTiles = 6;
 
     private boolean mDisabledByPolicy;
     private int mMaxTiles;
     protected QSPanel mFullPanel;
 
 
     @Inject
     public QuickQSPanel(
             @Named(VIEW_CONTEXT) Context context,
             AttributeSet attrs,
             DumpManager dumpManager,
             BroadcastDispatcher broadcastDispatcher,
             QSLogger qsLogger,
             MediaHost mediaHost,
             UiEventLogger uiEventLogger
     ) {
         super(context, attrs, dumpManager, broadcastDispatcher, qsLogger, mediaHost, uiEventLogger);
         sDefaultMaxTiles = getResources().getInteger(R.integer.quick_qs_panel_max_columns);
         applyBottomMargin((View) mRegularTileLayout);
     }
 
     private void applyBottomMargin(View view) {
         int margin = getResources().getDimensionPixelSize(R.dimen.qs_header_tile_margin_bottom);
         MarginLayoutParams layoutParams = (MarginLayoutParams) view.getLayoutParams();
         layoutParams.bottomMargin = margin;
         view.setLayoutParams(layoutParams);
     }
 
     @Override
     protected void addSecurityFooter() {
         // No footer needed
     }
 
     @Override
     protected void addViewsAboveTiles() {
         // Nothing to add above the tiles
     }
 
     @Override
     protected TileLayout createRegularTileLayout() {
         return new QuickQSPanel.HeaderTileLayout(mContext, mUiEventLogger);
      }
  
      @Override
      protected QSTileLayout createHorizontalTileLayout() {
          return new DoubleLineTileLayout(mContext, mUiEventLogger);
      }
  
      @Override
      protected void initMediaHostState() {
          mMediaHost.setExpansion(0.0f);
          mMediaHost.setShowsOnlyActiveMedia(true);
          mMediaHost.init(MediaHierarchyManager.LOCATION_QQS);
      }
  
      @Override
      protected boolean needsDynamicRowsAndColumns() {
          return false; // QQS always have the same layout
      }
  
      @Override
      protected boolean displayMediaMarginsOnMedia() {
          // Margins should be on the container to visually center the view
          return false;
      }
  
      @Override
      protected void updatePadding() {
          // QS Panel is setting a top padding by default, which we don't need.
      }
  
      @Override
      protected void onAttachedToWindow() {
          super.onAttachedToWindow();
          Dependency.get(TunerService.class).addTunable(mNumTiles, NUM_QUICK_TILES);
      }
  
      @Override
      protected void onDetachedFromWindow() {
          super.onDetachedFromWindow();
          Dependency.get(TunerService.class).removeTunable(mNumTiles);
      }
  
      @Override
      protected String getDumpableTag() {
          return TAG;
      }
  
      public void setQSPanelAndHeader(QSPanel fullPanel, View header) {
          mFullPanel = fullPanel;
      }
  
      @Override
      protected boolean shouldShowDetail() {
          return !mExpanded;
      }
  
      @Override
      protected void drawTile(TileRecord r, State state) {
          if (state instanceof SignalState) {
              SignalState copy = new SignalState();
              state.copyTo(copy);
              // No activity shown in the quick panel.
              copy.activityIn = false;
              copy.activityOut = false;
              state = copy;
          }
          super.drawTile(r, state);
      }
  
      @Override
      public void setHost(QSTileHost host, QSCustomizer customizer) {
          super.setHost(host, customizer);
          setTiles(mHost.getTiles());
      }
  
      public void setMaxTiles(int maxTiles) {
          mMaxTiles = maxTiles;
          if (mHost != null) {
              setTiles(mHost.getTiles());
          }
      }
  
      @Override
      public void onTuningChanged(String key, String newValue) {
          if (QS_SHOW_BRIGHTNESS.equals(key)) {
              // No Brightness or Tooltip for you!
              super.onTuningChanged(key, "0");
          }
      }
  
      @Override
      public void setTiles(Collection<QSTile> tiles) {
          ArrayList<QSTile> quickTiles = new ArrayList<>();
          for (QSTile tile : tiles) {
              quickTiles.add(tile);
              if (quickTiles.size() == mMaxTiles) {
                  break;
              }
          }
          super.setTiles(quickTiles, true);
      }
  
      private final Tunable mNumTiles = new Tunable() {
          @Override
          public void onTuningChanged(String key, String newValue) {
              setMaxTiles(parseNumTiles(newValue));
          }
      };
  
      public int getNumQuickTiles() {
          return mMaxTiles;
      }
  
      /**
       * Parses the String setting into the number of tiles. Defaults to {@code mDefaultMaxTiles}
       *
       * @param numTilesValue value of the setting to parse
       * @return parsed value of numTilesValue OR {@code mDefaultMaxTiles} on error
       */
      public static int parseNumTiles(String numTilesValue) {
          try {
              return Integer.parseInt(numTilesValue);
          } catch (NumberFormatException e) {
              // Couldn't read an int from the new setting value. Use default.
              return sDefaultMaxTiles;
          }
      }
  
      public static int getDefaultMaxTiles() {
          return sDefaultMaxTiles;
      }
  
      void setDisabledByPolicy(boolean disabled) {
          if (disabled != mDisabledByPolicy) {
              mDisabledByPolicy = disabled;
              setVisibility(disabled ? View.GONE : View.VISIBLE);
          }
      }
  
      /**
       * Sets the visibility of this {@link QuickQSPanel}. This method has no effect when this panel
       * is disabled by policy through {@link #setDisabledByPolicy(boolean)}, and in this case the
       * visibility will always be {@link View#GONE}. This method is called externally by
       * {@link QSAnimator} only.
       */
      @Override
      public void setVisibility(int visibility) {
          if (mDisabledByPolicy) {
              if (getVisibility() == View.GONE) {
                  return;
              }
              visibility = View.GONE;
          }
          super.setVisibility(visibility);
      }
  
      @Override
      protected QSEvent openPanelEvent() {
          return QSEvent.QQS_PANEL_EXPANDED;
      }
  
      @Override
      protected QSEvent closePanelEvent() {
          return QSEvent.QQS_PANEL_COLLAPSED;
      }
  

```

在 QuickQSPanel( ) 的构造方法中初始化 QuickQSPanel 的各种参数，  
从代码中发现 view.setLayoutParams(layoutParams); 就是添加快捷布局  
所以添加亮度条 ui 为：

```
+    protected View mBrightnessView = null;
+    private BrightnessController mBrightnessController=null;

 private void applyBottomMargin(View view) {
      int margin = getResources().getDimensionPixelSize(R.dimen.qs_header_tile_margin_bottom);
      MarginLayoutParams layoutParams = (MarginLayoutParams) view.getLayoutParams();
     layoutParams.bottomMargin = margin;
      view.setLayoutParams(layoutParams);
      +        mBrightnessView = LayoutInflater.from(mContext).inflate(
+            R.layout.quick_settings_brightness_dialog, this, false);
+        LinearLayout.LayoutParams layoutParams = new LinearLayout.LayoutParams(LinearLayout.LayoutParams.WRAP_CONTENT,LinearLayout.LayoutParams.WRAP_CONTENT);
+        layoutParams.topMargin = context.getResources().getDimensionPixelSize(R.dimen.status_bar_padding_start);
+        mBrightnessView.setLayoutParams(layoutParams);
+               addView(mBrightnessView,1);
+        mBrightnessController = new BrightnessController(getContext(),
+                findViewById(R.id.brightness_slider));
+               setBrightnessListening(true);
  }

```

在 applyBottomMargin(View view) 中增加亮度条布局和亮度条控制类负责管理亮度条拖拽的相关功能实现  
亮度条相关回调方法

```
+    public void setBrightnessListening(boolean listening) {
+        if (listening) {
+            mBrightnessController.registerCallbacks();
+        } else {
+            mBrightnessController.unregisterCallbacks();
+        }
+    }

```

通过增加 setBrightnessListening(boolean listening) 来监听亮度条拖拽的回调事件，从而做相关操作
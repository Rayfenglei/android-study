> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124717154)

### 1. 概述

在 11.0 的系统产品开发中，在 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 中定制化开发也是常见，最近产品项目要求对于下拉状态栏和通知栏也是需要做定制化开发的，修改 UI 的常见功能，产品需要在下滑展开状态栏的时候在 QSPanel 部分添加时间显示功能，可以在下拉状态栏的实现显示日期

### 2.SystemUI 的 QSPanel 添加日期布局的核心类

```
frameworks/base/packages/SystemUI/src/com/android/systemui/qs/QSPanel.java

```

### 3.SystemUI 的 QSPanel 添加日期布局核心功能分析和实现

在下拉状态栏中 QSPanel.java 就是下拉展开负责绘制页面的，对于增加日期布局，其实就是增加 SystemUI 的 DateView 类就可以了实现功能  
接下来看下  
在 QSPanel.java 中如何添加日期布局  
路径为：  
frameworks/base/packages/SystemUI/src/com/android/systemui/qs/QSPanel.java

```
public class QSPanel extends LinearLayout implements Tunable, Callback, BrightnessMirrorListener,
         Dumpable {
      @Inject
      public QSPanel(
              @Named(VIEW_CONTEXT) Context context,
              AttributeSet attrs,
              DumpManager dumpManager,
              BroadcastDispatcher broadcastDispatcher,
              QSLogger qsLogger,
              MediaHost mediaHost,
              UiEventLogger uiEventLogger
      ) {
          super(context, attrs);
          mUsingMediaPlayer = useQsMediaPlayer(context);
          mMediaTotalBottomMargin = getResources().getDimensionPixelSize(
                  R.dimen.quick_settings_bottom_margin_media);
          mMediaHost = mediaHost;
          mMediaHost.addVisibilityChangeListener((visible) -> {
              onMediaVisibilityChanged(visible);
              return null;
          });
          mContext = context;
          mQSLogger = qsLogger;
          mDumpManager = dumpManager;
          mBroadcastDispatcher = broadcastDispatcher;
          mUiEventLogger = uiEventLogger;
  
          setOrientation(VERTICAL);
  
          addViewsAboveTiles();
          mMovableContentStartIndex = getChildCount();
          mRegularTileLayout = createRegularTileLayout();
  
          if (mUsingMediaPlayer) {
              mHorizontalLinearLayout = new RemeasuringLinearLayout(mContext);
              mHorizontalLinearLayout.setOrientation(LinearLayout.HORIZONTAL);
              mHorizontalLinearLayout.setClipChildren(false);
              mHorizontalLinearLayout.setClipToPadding(false);
  
              mHorizontalContentContainer = new RemeasuringLinearLayout(mContext);
              mHorizontalContentContainer.setOrientation(LinearLayout.VERTICAL);
              mHorizontalContentContainer.setClipChildren(false);
              mHorizontalContentContainer.setClipToPadding(false);
  
              mHorizontalTileLayout = createHorizontalTileLayout();
              LayoutParams lp = new LayoutParams(0, LayoutParams.WRAP_CONTENT, 1);
              int marginSize = (int) mContext.getResources().getDimension(R.dimen.qqs_media_spacing);
              lp.setMarginStart(0);
              lp.setMarginEnd(marginSize);
              lp.gravity = Gravity.CENTER_VERTICAL;
              mHorizontalLinearLayout.addView(mHorizontalContentContainer, lp);
  
              lp = new LayoutParams(LayoutParams.MATCH_PARENT, 0, 1);
              addView(mHorizontalLinearLayout, lp);
  
              initMediaHostState();
          }
          addSecurityFooter();
          if (mRegularTileLayout instanceof PagedTileLayout) {
              mQsTileRevealController = new QSTileRevealController(mContext, this,
                      (PagedTileLayout) mRegularTileLayout);
          }
          mQSLogger.logAllTilesChangeListening(mListening, getDumpableTag(), mCachedSpecs);
          updateResources();
      }
  
      @Override
      protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
          if (mTileLayout instanceof PagedTileLayout) {
              // Since PageIndicator gets measured before PagedTileLayout, we preemptively set the
              // # of pages before the measurement pass so PageIndicator is measured appropriately
              if (mFooterPageIndicator != null) {
                  mFooterPageIndicator.setNumPages(((PagedTileLayout) mTileLayout).getNumPages());
              }
  
              // Allow the UI to be as big as it want's to, we're in a scroll view
              int newHeight = 10000;
              int availableHeight = MeasureSpec.getSize(heightMeasureSpec);
              int excessHeight = newHeight - availableHeight;
              // Measure with EXACTLY. That way, The content will only use excess height and will
              // be measured last, after other views and padding is accounted for. This only
              // works because our Layouts in here remeasure themselves with the exact content
              // height.
              heightMeasureSpec = MeasureSpec.makeMeasureSpec(newHeight, MeasureSpec.EXACTLY);
              ((PagedTileLayout) mTileLayout).setExcessHeight(excessHeight);
          }
          super.onMeasure(widthMeasureSpec, heightMeasureSpec);
  
          // We want all the logic of LinearLayout#onMeasure, and for it to assign the excess space
          // not used by the other children to PagedTileLayout. However, in this case, LinearLayout
          // assumes that PagedTileLayout would use all the excess space. This is not the case as
          // PagedTileLayout height is quantized (because it shows a certain number of rows).
          // Therefore, after everything is measured, we need to make sure that we add up the correct
          // total height
          int height = getPaddingBottom() + getPaddingTop();
          int numChildren = getChildCount();
          for (int i = 0; i < numChildren; i++) {
              View child = getChildAt(i);
              if (child.getVisibility() != View.GONE) {
                  height += child.getMeasuredHeight();
                  MarginLayoutParams layoutParams = (MarginLayoutParams) child.getLayoutParams();
                  height += layoutParams.topMargin + layoutParams.bottomMargin;
              }
          }
          setMeasuredDimension(getMeasuredWidth(), height);
      }
  .....

```

在上述 QSPanel 的相关源码中发现，它是 LinearLayout 的子类，在它的构造方法中，初始化相关参数用于定义 QSPanel 的相关功能的实现，对于自定义 View 的布局中，实现三部曲中，发现 onMeasure(int widthMeasureSpec, int heightMeasureSpec) 负责绘制 QSPanel 的布局页面  
在 QSPanel.java 的方法中在 addViewsAboveTiles(); 即为亮度条代码，是显示在 QSPanel.java 布局中的，所以也可以在这里添加日期的布局，

```
protected void addViewsAboveTiles() {
mBrightnessView = LayoutInflater.from(mContext).inflate(
R.layout.quick_settings_brightness_dialog, this, false);
addView(mBrightnessView);
mBrightnessController = new BrightnessController(getContext(),
findViewById(R.id.brightness_slider), mBroadcastDispatcher);
}

```

在 addViewsAboveTiles() 中添加日期布局的相关方法  
mQsDateview 就是日期的布局，  
添加代码如下：

```
private View mQsDateview;
protected void addViewsAboveTiles() {
   mQsDateview = LayoutInflater.from(mContext).inflate(

           R.layout.quick_qs_date, this, false);

          addView(mQsDateview);

    mBrightnessView = LayoutInflater.from(mContext).inflate(
        R.layout.quick_settings_brightness_dialog, this, false);
    addView(mBrightnessView);
    mBrightnessController = new BrightnessController(getContext(),
            findViewById(R.id.brightness_slider), mBroadcastDispatcher);
}

```

在 addViewsAboveTiles() 中通过添加 Dateview 的布局，然后调用 addView 往 QSPanel.java 这个容器中添加布局，就实现了添加日期的功能。
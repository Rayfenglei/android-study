> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124754748)

### 1. 概述

在 11.0 的系统定制化开发中，在 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 的下拉状态栏中的定制化开发中，对于下拉状态栏二次展开时候, QuickQsPanel 中功能开关键的字体，字体会随着系统字体的改变而改变，首选要找到布局字体的相关类，然后设置 configuration 属性固定不变就可以了

### 2.QSTileView 的字体大小不随系统字体改变而改变时字体大小的核心类

```
frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tileimpl/QSTileView.java

```

### 3.QSTileView 的字体大小不随系统字体改变而改变时字体大小的核心功能实现

通过 QuickQsPanel 的相关源代码发现，功能开关的的相关布局发现  
功能开关的布局就是 QSTileView.java  
路径是 frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tileimpl/QSTileView.java  
于是就来看相关代码

### 3.1QSTileView.java 相关源码分析

```
public class QSTileView extends QSTileBaseView {
     private static final int MAX_LABEL_LINES = 2;
     private static final boolean DUAL_TARGET_ALLOWED = false;
     private View mDivider;
     protected TextView mLabel;
     protected TextView mSecondLine;
     private ImageView mPadLock;
     private int mState;
     private ViewGroup mLabelContainer;
     private View mExpandIndicator;
     private View mExpandSpace;
     private ColorStateList mColorLabelDefault;
     private ColorStateList mColorLabelUnavailable;
 
     public QSTileView(Context context, QSIconView icon) {
         this(context, icon, false);
     }
 
     public QSTileView(Context context, QSIconView icon, boolean collapsedView) {
         super(context, icon, collapsedView);
 
         setClipChildren(false);
         setClipToPadding(false);
 
         setClickable(true);
         setId(View.generateViewId());
         createLabel();
         setOrientation(VERTICAL);
         setGravity(Gravity.CENTER_HORIZONTAL | Gravity.TOP);
         mColorLabelDefault = Utils.getColorAttr(getContext(), android.R.attr.textColorPrimary);
         // The text color for unavailable tiles is textColorSecondary, same as secondaryLabel for
         // contrast purposes
         mColorLabelUnavailable = Utils.getColorAttr(getContext(),
                 android.R.attr.textColorSecondary);
     }
 
     TextView getLabel() {
         return mLabel;
     }
 
     @Override
     protected void onConfigurationChanged(Configuration newConfig) {
         super.onConfigurationChanged(newConfig);
         FontSizeUtils.updateFontSize(mLabel, R.dimen.qs_tile_text_size);
         FontSizeUtils.updateFontSize(mSecondLine, R.dimen.qs_tile_text_size);
     }
 
     @Override
     public int getDetailY() {
         return getTop() + mLabelContainer.getTop() + mLabelContainer.getHeight() / 2;
     }
 
     protected void createLabel() {
         mLabelContainer = (ViewGroup) LayoutInflater.from(getContext())
                 .inflate(R.layout.qs_tile_label, this, false);
         mLabelContainer.setClipChildren(false);
         mLabelContainer.setClipToPadding(false);
         mLabel = mLabelContainer.findViewById(R.id.tile_label);
         mPadLock = mLabelContainer.findViewById(R.id.restricted_padlock);
         mDivider = mLabelContainer.findViewById(R.id.underline);
         mExpandIndicator = mLabelContainer.findViewById(R.id.expand_indicator);
         mExpandSpace = mLabelContainer.findViewById(R.id.expand_space);
          mSecondLine = mLabelContainer.findViewById(R.id.app_label);
          addView(mLabelContainer);
      }
  
      @Override
      protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
          super.onMeasure(widthMeasureSpec, heightMeasureSpec);
  
          // Remeasure view if the primary label requires more then 2 lines or the secondary label
          // text will be cut off.
          if (mLabel.getLineCount() > MAX_LABEL_LINES || !TextUtils.isEmpty(mSecondLine.getText())
                          && mSecondLine.getLineHeight() > mSecondLine.getHeight()) {
              mLabel.setSingleLine();
              super.onMeasure(widthMeasureSpec, heightMeasureSpec);
          }
      }
  
      @Override
      protected void handleStateChanged(QSTile.State state) {
          super.handleStateChanged(state);
          if (!Objects.equals(mLabel.getText(), state.label) || mState != state.state) {
              mLabel.setTextColor(state.state == Tile.STATE_UNAVAILABLE ? mColorLabelUnavailable
                      : mColorLabelDefault);
              mState = state.state;
              mLabel.setText(state.label);
          }
          if (!Objects.equals(mSecondLine.getText(), state.secondaryLabel)) {
              mSecondLine.setText(state.secondaryLabel);
              mSecondLine.setVisibility(TextUtils.isEmpty(state.secondaryLabel) ? View.GONE
                      : View.VISIBLE);
          }
          boolean dualTarget = DUAL_TARGET_ALLOWED && state.dualTarget;
          mExpandIndicator.setVisibility(dualTarget ? View.VISIBLE : View.GONE);
          mExpandSpace.setVisibility(dualTarget ? View.VISIBLE : View.GONE);
          mLabelContainer.setContentDescription(dualTarget ? state.dualLabelContentDescription
                  : null);
          if (dualTarget != mLabelContainer.isClickable()) {
              mLabelContainer.setClickable(dualTarget);
              mLabelContainer.setLongClickable(dualTarget);
              mLabelContainer.setBackground(dualTarget ? newTileBackground() : null);
          }
          mLabel.setEnabled(!state.disabledByPolicy);
          mPadLock.setVisibility(state.disabledByPolicy ? View.VISIBLE : View.GONE);
      }
  
      @Override
      public void init(OnClickListener click, OnClickListener secondaryClick,
              OnLongClickListener longClick) {
          super.init(click, secondaryClick, longClick);
          mLabelContainer.setOnClickListener(secondaryClick);
          mLabelContainer.setOnLongClickListener(longClick);
          mLabelContainer.setClickable(false);
          mLabelContainer.setLongClickable(false);
      }
  }

```

在 onConfigurationChanged(Configuration newConfig) 中就是设置 Configuration 属性的，当系统  
Configuration 改变时 QSTileView.java 的属性也会跟着改变  
发现 onConfigurationChanged 就是系统字体改变时就是要改对 QSTileView.java 的字体改变的方法  
所以具体修改如下:

```
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tileimpl/QSTileView.java

+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tileimpl/QSTileView.java

@@ -78,8 +78,8 @@ public class QSTileView extends QSTileBaseView {

     @Override

     protected void onConfigurationChanged(Configuration newConfig) {

         super.onConfigurationChanged(newConfig);

-        FontSizeUtils.updateFontSize(mLabel, R.dimen.qs_tile_text_size);

-        FontSizeUtils.updateFontSize(mSecondLine, R.dimen.qs_tile_text_size);

+        //FontSizeUtils.updateFontSize(mLabel, R.dimen.qs_tile_text_size);

+        //FontSizeUtils.updateFontSize(mSecondLine, R.dimen.qs_tile_text_size);

     }

```

然后编译 SystemUI 验证后发现问题已经解决
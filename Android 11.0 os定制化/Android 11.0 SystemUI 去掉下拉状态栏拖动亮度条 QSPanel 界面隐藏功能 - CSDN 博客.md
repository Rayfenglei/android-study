> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124760278)

### 1. 概述

在 11.0 的产品开发中，对于 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 的定制化开发中系列定制功能也挺多的，在 SystemUI 的系统原生下拉状态栏中，发现亮度条 SeekBar 当拖动的时候，亮度会改变但是同时整个 QSPanel 下拉状态栏也隐藏掉了产品要求去掉这个拖动亮度条时隐藏下拉状态栏的功能

### 2.SystemUI 去掉下拉状态栏拖动亮度条 QSPanel 界面隐藏功能类

```
frameworks/base/packages/SystemUI/res/layout/quick_settings_brightness_dialog.xml
frameworks/base/packages/SystemUI/src/com/android/systemui/settings/ToggleSliderView.java

```

### 3.SystemUI 去掉下拉状态栏拖动亮度条 QSPanel 界面隐藏功能分析和实现

首先看 QSPanel 的亮度条的布局  
通过查看相关布局文件  
quick_settings_brightness_dialog.xml

### 3.1quick_settings_brightness_dialog.xml 的分析

```
<?xml version="1.0" encoding="utf-8"?>
<!-- Copyright (C) 2012 The Android Open Source Project

     Licensed under the Apache License, Version 2.0 (the "License");
     you may not use this file except in compliance with the License.
     You may obtain a copy of the License at

          http://www.apache.org/licenses/LICENSE-2.0

      Unless required by applicable law or agreed to in writing, software
      distributed under the License is distributed on an "AS IS" BASIS,
      WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
      See the License for the specific language governing permissions and
      limitations under the License.
 -->
 <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
     xmlns:systemui="http://schemas.android.com/apk/res-auto"
     android:layout_height="wrap_content"
     android:layout_width="match_parent"
     android:layout_gravity="center_vertical"
     android:paddingLeft="16dp"
     android:paddingRight="16dp"
     style="@style/BrightnessDialogContainer">
 
     <com.android.systemui.settings.ToggleSliderView
         android:id="@+id/brightness_slider"
         android:layout_width="0dp"
         android:layout_height="48dp"
         android:layout_gravity="center_vertical"
         android:layout_weight="1"
         android:contentDescription="@string/accessibility_brightness"
         android:importantForAccessibility="no"
         systemui:text="@string/status_bar_settings_auto_brightness_label" />
 
 </LinearLayout>

```

从 quick_settings_brightness_dialog.xml 布局文件看 com.android.systemui.settings.ToggleSliderView 来处理亮度条的功能  
接了下来看 ToggleSliderView.java

### 3.2 ToggleSliderView 的关于进度条的功能分析

```
public class ToggleSliderView extends RelativeLayout implements ToggleSlider {
     private Listener mListener;
     private boolean mTracking;
 
     private CompoundButton mToggle;
     private ToggleSeekBar mSlider;
     private TextView mLabel;
 
     private ToggleSliderView mMirror;
     private BrightnessMirrorController mMirrorController;
 
     public ToggleSliderView(Context context) {
         this(context, null);
     }
 
     public ToggleSliderView(Context context, AttributeSet attrs) {
         this(context, attrs, 0);
     }
 
     public ToggleSliderView(Context context, AttributeSet attrs, int defStyle) {
         super(context, attrs, defStyle);
 
         View.inflate(context, R.layout.status_bar_toggle_slider, this);
 
         final Resources res = context.getResources();
         final TypedArray a = context.obtainStyledAttributes(
                 attrs, R.styleable.ToggleSliderView, defStyle, 0);
 
         mToggle = findViewById(R.id.toggle);
         mToggle.setOnCheckedChangeListener(mCheckListener);
 
         mSlider = findViewById(R.id.slider);
         mSlider.setOnSeekBarChangeListener(mSeekListener);
 
         mLabel = findViewById(R.id.label);
         mLabel.setText(a.getString(R.styleable.ToggleSliderView_text));
 
         mSlider.setAccessibilityLabel(getContentDescription().toString());
 
         a.recycle();
     }
 
     public void setMirror(ToggleSliderView toggleSlider) {
         mMirror = toggleSlider;
         if (mMirror != null) {
             mMirror.setChecked(mToggle.isChecked());
             mMirror.setMax(mSlider.getMax());
             mMirror.setValue(mSlider.getProgress());
         }
     }
 
     public void setMirrorController(BrightnessMirrorController c) {
         mMirrorController = c;
     }
 
     @Override
     protected void onAttachedToWindow() {
         super.onAttachedToWindow();
         if (mListener != null) {
             mListener.onInit(this);
         }
     }
 
     public void setEnforcedAdmin(RestrictedLockUtils.EnforcedAdmin admin) {
          mToggle.setEnabled(admin == null);
          mSlider.setEnabled(admin == null);
          mSlider.setEnforcedAdmin(admin);
      }
  
      public void setOnChangedListener(Listener l) {
          mListener = l;
      }
  
      @Override
      public void setChecked(boolean checked) {
          mToggle.setChecked(checked);
      }
  
      @Override
      public boolean isChecked() {
          return mToggle.isChecked();
      }
  
      @Override
      public void setMax(int max) {
          mSlider.setMax(max);
          if (mMirror != null) {
              mMirror.setMax(max);
          }
      }
  
      @Override
      public void setValue(int value) {
          mSlider.setProgress(value);
          if (mMirror != null) {
              mMirror.setValue(value);
          }
      }
  
      @Override
      public int getValue() {
          return mSlider.getProgress();
      }
  
      @Override
      public boolean dispatchTouchEvent(MotionEvent ev) {
          if (mMirror != null) {
              MotionEvent copy = ev.copy();
              mMirror.dispatchTouchEvent(copy);
              copy.recycle();
          }
          return super.dispatchTouchEvent(ev);
      }
  
      private final OnCheckedChangeListener mCheckListener = new OnCheckedChangeListener() {
          @Override
          public void onCheckedChanged(CompoundButton toggle, boolean checked) {
              mSlider.setEnabled(!checked);
  
              if (mListener != null) {
                  mListener.onChanged(
                          ToggleSliderView.this, mTracking, checked, mSlider.getProgress(), false);
              }
  
              if (mMirror != null) {
                  mMirror.mToggle.setChecked(checked);
              }
          }
      };
  
      private final OnSeekBarChangeListener mSeekListener = new OnSeekBarChangeListener() {
          @Override
          public void onProgressChanged(SeekBar seekBar, int progress, boolean fromUser) {
              if (mListener != null) {
                  mListener.onChanged(
                          ToggleSliderView.this, mTracking, mToggle.isChecked(), progress, false);
              }
          }
  
          @Override
          public void onStartTrackingTouch(SeekBar seekBar) {
              mTracking = true;
  
              if (mListener != null) {
                  mListener.onChanged(ToggleSliderView.this, mTracking, mToggle.isChecked(),
                          mSlider.getProgress(), false);
              }
  
              mToggle.setChecked(false);
  
              if (mMirrorController != null) {
                  mMirrorController.showMirror();
                  mMirrorController.setLocation((View) getParent());
              }
          }
  
          @Override
          public void onStopTrackingTouch(SeekBar seekBar) {
              mTracking = false;
  
              if (mListener != null) {
                  mListener.onChanged(ToggleSliderView.this, mTracking, mToggle.isChecked(),
                          mSlider.getProgress(), true);
              }
  
              if (mMirrorController != null) {
                  mMirrorController.hideMirror();
              }
          }
      };
  }

```

在对 [SeekBar](https://so.csdn.net/so/search?q=SeekBar&spm=1001.2101.3001.7020) 的拖动事件中 OnSeekBarChangeListener 是监听 seekbar 拖动的，所以相关拖动  
进度条的工作都是在这里进行的  
OnSeekBarChangeListener 处理拖动进度条的一些功能

```
 /*if (mMirrorController != null) {
                mMirrorController.showMirror();
                mMirrorController.setLocation((View) getParent());
            }*/

  /*if (mMirrorController != null) {
                mMirrorController.hideMirror();
            }*/

```

处理显示和隐藏下拉状态栏的功能
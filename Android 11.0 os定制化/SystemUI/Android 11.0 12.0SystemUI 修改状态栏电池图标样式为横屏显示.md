> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124806904)

### 1. 概述

在 11.0 12.0 的产品定制化开发中，对于原生系统中 SystemUId 状态栏的电池图标是竖着显示的，一般手机的电池图标都是横屏显示的  
可以觉得样式挺不错的，所以客户要求横着显示和手机的样式一样，所以就得重新更换 SystemUI 状态栏的电池样式了  
如图:  
![](https://img-blog.csdnimg.cn/649a239b80394b8bb7d2633e18837c63.png#pic_center)

### 2.SystemUI 修改状态栏电池图标样式为横屏显示的核心类

```
frameworks\base\packages\SystemUI\src\com\android\systemui\BatteryMeterView.java
frameworks/base/packages/SystemUI/res/layout/status_bar.xml
frameworks/base/packages/SystemUI/res/system_icons.xml

```

3.SystemUI 修改状态栏电池图标样式为横屏显示的核心功能分析
----------------------------------

在 SystemUI 中状态栏的布局就是 status_bar.[xml](https://so.csdn.net/so/search?q=xml&spm=1001.2101.3001.7020)，  
接下来看 SystemUI 中的电池布局 status_bar.xml 中

### 3.1status_bar.xml 相关布局分析

```
       <com.android.keyguard.AlphaOptimizedLinearLayout android:id="@+id/system_icon_area"
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="1"
            android:orientation="horizontal"
            android:gravity="center_vertical|end"
            >

            <include layout="@layout/system_icons" />
        </com.android.keyguard.AlphaOptimizedLinearLayout>
继续看下 system_icons.xml 中的 
 <com.android.systemui.BatteryMeterView android:id="@+id/battery"
        android:layout_height="match_parent"
        android:layout_width="wrap_content"
        android:clipToPadding="false"
        android:clipChildren="false"
        systemui:textAppearance="@style/TextAppearance.StatusBar.Clock" />

```

在 system_icons.xml 中可以看出 com.android.systemui.BatteryMeterView 即为电池图标  
接下来看下 BatteryMeterView 的相关源码分析

### 3.2 BatteryMeterView 的相关源码分析

具体路径为: frameworks\base\packages\SystemUI\src\com\android\systemui\BatteryMeterView.java  
具体 BatteryMeterView 的相关源码分析如下:

```
import com.android.systemui.BatteryView;
import android.view.View;

private final ImageView mBatteryIconView;
    private BatteryView mNewStyleBatteryView;
    private ImageView mChangeIconView;
public BatteryMeterView(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);

        setOrientation(LinearLayout.HORIZONTAL);
        setGravity(Gravity.CENTER_VERTICAL | Gravity.START);

        TypedArray atts = context.obtainStyledAttributes(attrs, R.styleable.BatteryMeterView,
                defStyle, 0);
        final int frameColor = atts.getColor(R.styleable.BatteryMeterView_frameColor,
                context.getColor(R.color.meter_background_color));
        mPercentageStyleId = atts.getResourceId(R.styleable.BatteryMeterView_textAppearance, 0);
        /*Bug 1072082 add charge animation of batteryView*/
        mBatteryAnimation = mContext.getResources().getBoolean(
                R.bool.config_battery_animation);
        if(mBatteryAnimation){
            mSprdDrawable = new BatteryMeterDrawable(context, new Handler(), frameColor, false);
        }else{
            mDrawable = new ThemedBatteryDrawable(context, frameColor);
        }
        /*@}*/

        atts.recycle();

        mSettingObserver = new SettingObserver(new Handler(context.getMainLooper()));
        mShowPercentAvailable = context.getResources().getBoolean(
                com.android.internal.R.bool.config_battery_percentage_setting_available);


        addOnAttachStateChangeListener(
                new DisableStateTracker(DISABLE_NONE, DISABLE2_SYSTEM_ICONS));

        setupLayoutTransition();

        mSlotBattery = context.getString(
                com.android.internal.R.string.status_bar_battery);
        mBatteryIconView = new ImageView(context);
        /*Bug 1072082 add charge animation of batteryView*/
        if(mBatteryAnimation){
            mBatteryIconView.setImageDrawable(mSprdDrawable);
        }else{
            mBatteryIconView.setImageDrawable(mDrawable);
        }

        final MarginLayoutParams mlp = new MarginLayoutParams(
                getResources().getDimensionPixelSize(R.dimen.status_bar_battery_icon_width),
                getResources().getDimensionPixelSize(R.dimen.status_bar_battery_icon_height));
        mlp.setMargins(0, 0, 0,
                getResources().getDimensionPixelOffset(R.dimen.battery_margin_bottom));
        addView(mBatteryIconView, mlp);

           // add code start
           //添加电池图标
           mBatteryIconView.setVisibility(View.GONE);
            mNewStyleBatteryView = (BatteryView) LayoutInflater.from(getContext())
                 .inflate(R.layout.newstyle_battery_view, null);
            addView(mNewStyleBatteryView,new ViewGroup.LayoutParams(
                  44,
                  22));
            //添加电池充电图标
           mChangeIconView = (ImageView) LayoutInflater.from(getContext())
                 .inflate(R.layout.newstyle_battery_electricity_view, null);
            addView(mChangeIconView,new ViewGroup.LayoutParams(
                   20,
                   22));

            // add code end
        updateShowPercent();
        mDualToneHandler = new DualToneHandler(context);
        // Init to not dark at all.
        onDarkChanged(new Rect(), 0, DarkIconDispatcher.DEFAULT_ICON_TINT);

        mUserTracker = new CurrentUserTracker(mContext) {
            @Override
            public void onUserSwitched(int newUserId) {
                mUser = newUserId;
                getContext().getContentResolver().unregisterContentObserver(mSettingObserver);
                getContext().getContentResolver().registerContentObserver(
                        Settings.System.getUriFor(SHOW_BATTERY_PERCENT), false, mSettingObserver,
                        newUserId);
                updateShowPercent();
            }
        };
           //屏蔽原有电池图标
           mBatteryIconView.setVisibility(View.GONE);
        setClipChildren(false);
        setClipToPadding(false);
        Dependency.get(ConfigurationController.class).observe(viewAttachLifecycle(this), this);
    }

 
@Override
public void onBatteryLevelChanged(int level, boolean pluggedIn, boolean charging) {
    /*Bug 1072082 add charge animation of batteryView*/
    if(mDrawable != null){
        mDrawable.setCharging(pluggedIn);
        mDrawable.setBatteryLevel(level);
    }
    mCharging = pluggedIn;
    mLevel = level;
    updatePercentText();

      // add code start
       mNewStyleBatteryView.setProgress(mLevel);//设置电池图标内电量值
        //设置充电图标
       if (charging && (level != 100)) {
           mChangeIconView.setVisibility(View.VISIBLE);
       } else {
           mChangeIconView.setVisibility(View.GONE);
       }
      // add code end
}

```

长上述代码可以看出 mBatteryIconView 即为原来的电池样式 onBatteryLevelChanged 中电量改变后电池图标重新绘制  
// 电池电量改变时，充电时，显示充电图标

而在 updateColors(中改变充电颜色源码如下：

```
private void updateColors(int foregroundColor, int backgroundColor, int singleToneColor) {
        /*Bug 1072082 add charge animation of batteryView*/
        if(mDrawable != null){
          mDrawable.setColors(foregroundColor, backgroundColor, singleToneColor);
        }
        if(mSprdDrawable != null){
          mSprdDrawable.setColors(foregroundColor, backgroundColor);
        }
        mTextColor = singleToneColor;
        if (mBatteryPercentView != null) {
            mBatteryPercentView.setTextColor(singleToneColor);
        }

       // add code start
       //设置电池图标颜色
       if (mNewStyleBatteryView != null) {
           mNewStyleBatteryView.setColor(singleToneColor);
       }
        //设置电池充电图标颜色
        if (mChangeIconView != null) {
            mChangeIconView.setColorFilter(singleToneColor);
        }
       //add code end

    }

```

去掉百分比原来的样式

```
 private void updateShowPercent() {
        final boolean showing = mBatteryPercentView != null;
        final boolean systemSetting = 0 != Settings.System
                .getIntForUser(getContext().getContentResolver(),
                SHOW_BATTERY_PERCENT, 0, mUser);

        if ((mShowPercentAvailable && systemSetting && mShowPercentMode != MODE_OFF)
                || mShowPercentMode == MODE_ON || mShowPercentMode == MODE_ESTIMATE) {
            if (!showing) {
                mBatteryPercentView = loadPercentView();

                 // add code start
		//屏蔽原有电池图标显示
                   mBatteryPercentView.setVisibility(View.GONE);
               //add code end

                if (mPercentageStyleId != 0) { // Only set if specified as attribute
                    mBatteryPercentView.setTextAppearance(mPercentageStyleId);
                }
                if (mTextColor != 0) mBatteryPercentView.setTextColor(mTextColor);
                updatePercentText();
                addView(mBatteryPercentView,
                        new ViewGroup.LayoutParams(
                                LayoutParams.WRAP_CONTENT,
                                LayoutParams.MATCH_PARENT));
            }
        } else {
            if (showing) {
                removeView(mBatteryPercentView);
                mBatteryPercentView = null;
            }
        }
    }

```

### 3.3 添加自定义电池 View

```
package com.android.systemui;


import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.graphics.RectF;
import android.util.AttributeSet;
import android.util.DisplayMetrics;
import android.view.View;

import android.os.BatteryManager;
import android.content.Intent;
import android.content.IntentFilter;
import android.content.BroadcastReceiver;
public class BatteryView extends View {
    private float percent = 0f;
    Paint paint = new Paint();
    Paint paint1 = new Paint();
    Paint paint2 = new Paint();

    public BatteryView(Context context, AttributeSet set) {
        super(context, set);
        paint.setAntiAlias(true);
        paint.setStyle(Paint.Style.FILL);
	paint.setColor(Color.GRAY);
        paint1.setAntiAlias(true);
        paint1.setStyle(Paint.Style.STROKE);
        paint1.setStrokeWidth(dip2px(1.5f));
        paint1.setColor(-1728053248);
        paint2.setAntiAlias(true);
        paint2.setStyle(Paint.Style.FILL);
        paint2.setColor(-1728053248);

        DisplayMetrics dm = getResources().getDisplayMetrics();
        int mScreenWidth = dm.widthPixels;
        int mScreenHeight = dm.heightPixels;

        float ratioWidth = (float) mScreenWidth / 720;
        float ratioHeight = (float) mScreenHeight / 1080;
        float ratioMetrics = Math.min(ratioWidth, ratioHeight);
        int textSize = Math.round(20 * ratioMetrics);
        paint2.setTextSize(textSize);
    }

    private int dip2px(float dpValue) {
        final float scale = getResources().getDisplayMetrics().density;
        return (int) (dpValue * scale + 0.5f);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        int a = getWidth() - dip2px(2f);
        int b = getHeight() - dip2px(1.5f);
        float d = a * percent;
        float left = dip2px(0.5f);
        float top = dip2px(0.5f);
        float right = dip2px(2.5f);
        float bottom = dip2px(1.5f);

        RectF re1 = new RectF(left, top, d - right, b + bottom); 
        RectF re2 = new RectF(0, 0, a - right, b + bottom); 
        RectF re3 = new RectF(a - right + 2, b / 5, a, b + bottom - b / 5);  

        canvas.drawRect(re1, paint);
        canvas.drawRect(re2, paint1);
        canvas.drawRect(re3, paint1);
        canvas.drawText(String.valueOf((int) (percent * 100)), getWidth() / 4 - dip2px(3), getHeight() - getHeight() / 5, paint2);
    }

    public synchronized void setProgress(int percent) {
        this.percent = (float) (percent / 100.0);
        postInvalidate();
    }

    public void setColor(int color) {
	paint1.setColor(color);
	paint2.setColor(color);
	postInvalidate();
    }
}

```

资源文件:  
newstyle_battery_view.xml

```
<?xml version="1.0" encoding="utf-8"?>
<!-- Loaded into BatteryMeterView as necessary -->
<com.android.systemui.BatteryView
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:id="@+id/newstyle_battery_view"
        android:layout_width="30dp"
        android:layout_height="10dp"
	android:singleLine="true"
	android:gravity="center_vertical|start"
        />

```

newstyle_battery_electricity_view.xml

```
<?xml version="1.0" encoding="utf-8"?>
<ImageView
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:id="@+id/newstyle_battery_electricity_view"
        android:layout_width="20dp"
        android:layout_height="20dp"
	android:src="@drawable/icon_electricity"
        />

```

icon_electricity.xml

```
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="16dp"
    android:height="16dp"
    android:viewportWidth="1024"
    android:viewportHeight="1024">
  <path
      android:pathData="M788.48,410.11h-112.13c-24.06,0 -92.16,11.78 -115.2,-2.56 -4.61,-16.38 10.24,-46.08 14.34,-60.42 11.78,-41.47 23.04,-82.43 34.82,-124.42L660.48,41.98c6.66,-23.55 -24.06,-33.79 -36.86,-15.87L218.11,582.66c-9.22,12.8 3.58,30.72 17.92,30.72h164.35c11.26,0 54.78,-7.17 61.95,3.58 6.66,8.7 -6.14,35.84 -8.7,45.57 -30.72,106.5 -60.42,212.99 -90.11,319.49 -6.66,23.55 24.06,33.79 36.86,15.87l404.99,-556.54c9.73,-13.31 -2.56,-31.23 -16.9,-31.23z"
      android:fillColor="#BEBEBE"/>
</vector>

```
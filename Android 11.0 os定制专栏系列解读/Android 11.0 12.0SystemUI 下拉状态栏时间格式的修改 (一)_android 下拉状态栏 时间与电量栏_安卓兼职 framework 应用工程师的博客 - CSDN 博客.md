> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124774576)

### 1. 概述

在 11.0 12.0 产品定制化开发中，原生的下拉状态栏[时间格式](https://so.csdn.net/so/search?q=%E6%97%B6%E9%97%B4%E6%A0%BC%E5%BC%8F&spm=1001.2101.3001.7020)为 某月某日周几 这样的格式 根据产品需要修改为年月日周几 某时某分这种格式这就需要修改 显示时间的格式 在更新时间时 按照这个格式更新就可以了

### 2.[SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 下拉状态栏时间格式的修改 (一) 的核心类

```
frameworks\base\packages\SystemUI\res\layout\quick_qs_status_icons.xml
frameworks\base\packages\SystemUI\res-keyguard\values\donottranslate.xml
frameworks\base\packages\SystemUI\src\com\android\systemui\statusbar\policy\DateView.java

```

### 3.SystemUI 下拉状态栏时间格式的修改 (一) 的核心功能分析和实现

首选来看 时间控件的布局文件 quick_qs_status_icons.xml  
路径: frameworks\base\packages\SystemUI\res\layout\quick_qs_status_icons.xml

### 3.1quick_qs_status_icons.xml 的代码分析

```
<?xml version="1.0" encoding="utf-8"?>
<!-- Copyright (C) 2017 The Android Open Source Project

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
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:systemui="http://schemas.android.com/apk/res-auto"
    android:id="@+id/quick_qs_status_icons"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_marginTop="@*android:dimen/quick_qs_offset_height"
    android:paddingBottom="@dimen/qs_header_bottom_padding"
    android:paddingStart="@dimen/status_bar_padding_start"
    android:paddingEnd="@dimen/status_bar_padding_end"
    android:layout_below="@id/quick_status_bar_system_icons"
    android:clipChildren="false"
    android:clipToPadding="false"
    android:minHeight="20dp"
    android:clickable="false"
    android:focusable="true"
    android:theme="@style/QSHeaderTheme">

    <com.android.systemui.statusbar.policy.DateView
        android:id="@+id/date"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="start|center_vertical"
        android:gravity="center_vertical"
        android:singleLine="true"
        android:textAppearance="@style/TextAppearance.QS.Status"
        systemui:datePattern="@string/abbrev_wday_month_day_no_year_alarm" />



    <com.android.systemui.BatteryMeterView
        android:id="@+id/batteryRemainingIcon"
        android:layout_height="match_parent"
        android:layout_width="wrap_content"
        systemui:textAppearance="@style/TextAppearance.QS.Status"
        android:paddingEnd="2dp" />

</LinearLayout>

```

从上述的 quick_qs_status_icons.xml 的  
xml 文件中 com.android.systemui.statusbar.policy.DateView 就是显示时间的控件  
而 `systemui:datePattern=“@string/abbrev_wday_month_day_no_year_alarm” 就是显示时间的格式`  
而他定义在 donottranslate.xml

### 3.2donottranslate.xml 的相关代码分析

路径: frameworks\base\packages\SystemUI\res-keyguard\values\donottranslate.xml

```
<resources xmlns:xliff="urn:oasis:names:tc:xliff:document:1.2">
    <!-- Skeleton string format for displaying the date. -->
    <string >EEEEMMMMd</string>

    <!-- Skeleton string format for displaying the date when an alarm is set. -->
    <string >eeeMMMMd</string>

    <!-- Skeleton string format for displaying the time in 12-hour format. -->
    <string >hm</string>

    <!-- Skeleton string format for displaying the time in 24-hour format. -->
    <string >Hm</string>
</resources>

```

通过查询时间格式 YYYYeeeMMMMd HH:mm 就是年月日周几 某时某分  
修改时间格式为:

```
 <string >YYYYeeeMMMMd HH:mm</string>

```

接下来看更新时间 DateView.java 的相关源码

### 3.3DateView.java 的相关源码分析

```
frameworks\base\packages\SystemUI\src\com\android\systemui\statusbar\policy\DateView.java
private BroadcastReceiver mIntentReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            final String action = intent.getAction();
            if (Intent.ACTION_TIME_TICK.equals(action)
                    || Intent.ACTION_TIME_CHANGED.equals(action)
                    || Intent.ACTION_TIMEZONE_CHANGED.equals(action)
                    || Intent.ACTION_LOCALE_CHANGED.equals(action)) {
                if (Intent.ACTION_LOCALE_CHANGED.equals(action)
                        || Intent.ACTION_TIMEZONE_CHANGED.equals(action)) {
                    // need to get a fresh date format
                    getHandler().post(() -> mDateFormat = null);
                }
                getHandler().post(() -> updateClock());
            }
        }
    };

    public DateView(Context context, AttributeSet attrs) {
        super(context, attrs);
        TypedArray a = context.getTheme().obtainStyledAttributes(
                attrs,
                R.styleable.DateView,
                0, 0);

        try {
            mDatePattern = a.getString(R.styleable.DateView_datePattern);
        } finally {
            a.recycle();
        }
        if (mDatePattern == null) {
            mDatePattern = getContext().getString(R.string.abbrev_wday_month_day_no_year_alarm);
        }
    }



    @Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();

        mDateFormat = null; // reload the locale next time
        getContext().unregisterReceiver(mIntentReceiver);
    }

    protected void updateClock() {
        if (mDateFormat == null) {
            final Locale l = Locale.getDefault();
            DateFormat format = DateFormat.getInstanceForSkeleton(mDatePattern, l);
            format.setContext(DisplayContext.CAPITALIZATION_FOR_STANDALONE);
            mDateFormat = format;
        }

        mCurrentTime.setTime(System.currentTimeMillis());

        final String text = mDateFormat.format(mCurrentTime);
        if (!text.equals(mLastText)) {
            setText(text);
            mLastText = text;
        }
    }

```

通过时间监听广播发现时间变了更新时间，所以调用 updateClock() 更新时间  
在 updateClock() 的时间格式是由 mDatePattern 来决定的 mDatePattern 是在构造方法中定义的  
所以修改 mDatePattern 为:

```
if (mDatePattern == null) {
 -   mDatePattern = getContext().getString(R.string.system_ui_date_pattern);
 +   mDatePattern = getContext().getString(R.string.abbrev_wday_month_day_no_year_alarm);
}

```
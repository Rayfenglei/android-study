> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124774541)

### 1. 概述

通过上一篇博客已经实现修改了时间显示格式，但是客户修改下拉状态栏时间显示格式为分行显示，即第一行显示时间用大字体显示，  
第二行用小字体显示当前日期和周几这样的显示格式 于是继续进行修改

### 2.SystemUI 下拉状态栏时间格式的修改 (二) 的核心类

```
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/policy/DateView.java

```

### 3.SystemUI 下拉状态栏时间格式的修改 (二) 的核心功能分析和实现

通过上篇代码分析发现时间显示控件就是 DateView.java 来负责显示时间  
具体路径为：  
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/policy/DateView.java  
SpannableString 的相关用法分析，它的 api 就是用显示不同颜色不同字体的字符串功能  
所以同一个字符串 用不同的颜色和样式显示可以用 SpannableString 的相关 api 来实现

```
    SpannableString ss = new SpannableString(text);
    //设置显示字体大小

    AbsoluteSizeSpan ass = new AbsoluteSizeSpan(20,true);
   //从哪个文字开始设置字体大小

    ss.setSpan(ass, 0, 5, Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);

               AbsoluteSizeSpan ass2 = new AbsoluteSizeSpan(14,true);
    ss.setSpan(ass2, 5, ss.length(), Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
    setText(ss, TextView.BufferType.SPANNABLE);

```

根据设置 SpannableString 的下标来实现不同颜色不同字体的功能实现

而在 DateView 的源码中接收到时间变化的通知后在通过代码中 updateClock() 来负责更新时间

```
public class DateView extends TextView {
     private static final String TAG = "DateView";
 
     private final Date mCurrentTime = new Date();
 
     private DateFormat mDateFormat;
     private String mLastText;
     private String mDatePattern;
     private final BroadcastDispatcher mBroadcastDispatcher;
 
     private BroadcastReceiver mIntentReceiver = new BroadcastReceiver() {
         @Override
         public void onReceive(Context context, Intent intent) {
             // If the handler is null, it means we received a broadcast while the view has not
             // finished being attached or in the process of being detached.
             // In that case, do not post anything.
             Handler handler = getHandler();
             if (handler == null) return;
 
             final String action = intent.getAction();
             if (Intent.ACTION_TIME_TICK.equals(action)
                     || Intent.ACTION_TIME_CHANGED.equals(action)
                     || Intent.ACTION_TIMEZONE_CHANGED.equals(action)
                     || Intent.ACTION_LOCALE_CHANGED.equals(action)) {
                 if (Intent.ACTION_LOCALE_CHANGED.equals(action)
                         || Intent.ACTION_TIMEZONE_CHANGED.equals(action)) {
                     // need to get a fresh date format
                     handler.post(() -> mDateFormat = null);
                 }
                 handler.post(() -> updateClock());
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
             mDatePattern = getContext().getString(R.string.system_ui_date_pattern);
         }
         mBroadcastDispatcher = Dependency.get(BroadcastDispatcher.class);
     }
 
     @Override
     protected void onAttachedToWindow() {
         super.onAttachedToWindow();
 
         IntentFilter filter = new IntentFilter();
         filter.addAction(Intent.ACTION_TIME_TICK);
         filter.addAction(Intent.ACTION_TIME_CHANGED);
         filter.addAction(Intent.ACTION_TIMEZONE_CHANGED);
         filter.addAction(Intent.ACTION_LOCALE_CHANGED);
         mBroadcastDispatcher.registerReceiverWithHandler(mIntentReceiver, filter,
                  Dependency.get(Dependency.TIME_TICK_HANDLER));
  
          updateClock();
      }
  
      @Override
      protected void onDetachedFromWindow() {
          super.onDetachedFromWindow();
  
          mDateFormat = null; // reload the locale next time
          mBroadcastDispatcher.unregisterReceiver(mIntentReceiver);
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
  
      public void setDatePattern(String pattern) {
          if (TextUtils.equals(pattern, mDatePattern)) {
              return;
          }
          mDatePattern = pattern;
          mDateFormat = null;
          if (isAttachedToWindow()) {
              updateClock();
          }
      }
  }

```

在 DateView.java 的源码中通过注册监听时间的变化，然后调用 updateClock() 来更新时间

所以具体修改显示时间的样式 就是在这里修改即可

```
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/policy/DateView.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/policy/DateView.java
@@ -26,13 +26,16 @@ import android.icu.text.DisplayContext;
 import android.text.TextUtils;
 import android.util.AttributeSet;
 import android.widget.TextView;
-
+import android.content.res.Configuration;
 import com.android.systemui.Dependency;
 import com.android.systemui.R;
-
+import android.widget.RelativeLayout;
 import java.util.Date;
 import java.util.Locale;
-
+import android.text.SpannableString;
+import android.text.Spanned;
+import android.text.SpannedString;
+import android.text.style.AbsoluteSizeSpan;
 public class DateView extends TextView {
     private static final String TAG = "DateView";
 
@@ -73,8 +76,9 @@ public class DateView extends TextView {
             a.recycle();
         }
         if (mDatePattern == null) {
-            mDatePattern = getContext().getString(R.string.system_ui_date_pattern);
+            mDatePattern = getContext().getString(R.string.abbrev_wday_month_day_no_year_alarm);
         }
+
     }
 
     @Override
@@ -110,9 +114,18 @@ public class DateView extends TextView {
 
         mCurrentTime.setTime(System.currentTimeMillis());
 
-        final String text = mDateFormat.format(mCurrentTime);
+        String text = mDateFormat.format(mCurrentTime);
+        String [] date_txt = text.split(" ");
+        if(date_txt!=null&&date_txt.length>=2){
+                   text = date_txt[1]+"\n"+date_txt[0].replace("周"," 星期");
+        }
         if (!text.equals(mLastText)) {
-            setText(text);
+                   SpannableString ss = new SpannableString(text);
+            AbsoluteSizeSpan ass = new AbsoluteSizeSpan(20,true);
+            StyleSpan styleSpan_B  = new StyleSpan(Typeface.BOLD);
+            ss.setSpan(styleSpan_B, 0, 5, Spanned.SPAN_INCLUSIVE_EXCLUSIVE);

+            ss.setSpan(ass, 0, 5, Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
+                       AbsoluteSizeSpan ass2 = new AbsoluteSizeSpan(14,true);
+            ss.setSpan(ass2, 5, ss.length(), Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
+            setText(ss, TextView.BufferType.SPANNABLE);
             mLastText = text;
         }
     }

```

通过设置时间字符串不同格式来修改功能
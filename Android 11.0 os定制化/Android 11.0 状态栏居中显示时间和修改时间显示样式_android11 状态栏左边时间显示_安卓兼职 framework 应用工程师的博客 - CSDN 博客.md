> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124807054)

11.0 状态栏系统时间默认显示在左边和通知显示在一起，但是客户想修改显示位置，想显示在中间，所以就要修改 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020)  
的 Clock.java 文件这个就是管理显示时间的, 居中显示的话就得修改布局文件了  
效果图如下:  
在这里插入图片描述  
![](https://img-blog.csdnimg.cn/e5a3df87d4e84f2791482e61cab8ee77.png#pic_center)

1. 布局文件的修改:  
/frameworks/base/packages/SystemUI/res/layout/status_bar.xml

```
<com.android.systemui.statusbar.phone.PhoneStatusBarView
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:systemui="http://schemas.android.com/apk/res/com.android.systemui"
    android:layout_width="match_parent"
    android:layout_height="@dimen/status_bar_height"
    android:id="@+id/status_bar"
    android:background="@drawable/system_bar_background"
    android:orientation="vertical"
    android:focusable="false"
    android:descendantFocusability="afterDescendants"
    android:accessibilityPaneTitle="@string/status_bar"
    >

    <ImageView
        android:id="@+id/notification_lights_out"
        android:layout_width="@dimen/status_bar_icon_size"
        android:layout_height="match_parent"
        android:paddingStart="@dimen/status_bar_padding_start"
        android:paddingBottom="2dip"
        android:src="@drawable/ic_sysbar_lights_out_dot_small"
        android:scaleType="center"
        android:visibility="gone"
        />

    <LinearLayout android:id="@+id/status_bar_contents"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:paddingStart="@dimen/status_bar_padding_start"
        android:paddingEnd="@dimen/status_bar_padding_end"
        android:paddingTop="@dimen/status_bar_padding_top"
        android:orientation="horizontal"
        >
        // 放大比重 原来的由1修改为2
        <FrameLayout
            android:layout_height="match_parent"
            android:layout_width="0dp"
            android:layout_weight="2">

            <include layout="@layout/heads_up_status_bar_layout" />

            <!-- The alpha of the left side is controlled by PhoneStatusBarTransitions, and the
             individual views are controlled by StatusBarManager disable flags DISABLE_CLOCK and
             DISABLE_NOTIFICATION_ICONS, respectively -->
            <LinearLayout
                android:id="@+id/status_bar_left_side"
                android:layout_height="match_parent"
                android:layout_width="match_parent"
                android:clipChildren="false"
            >
                <ViewStub
                    android:id="@+id/operator_name"
                    android:layout_width="wrap_content"
                    android:layout_height="match_parent"
                    android:layout="@layout/operator_name" />
                //调整原来的布局位置 所以要注释掉
                <!--com.android.systemui.statusbar.policy.Clock
                    android:id="@+id/clock"
                    android:layout_width="wrap_content"
                    android:layout_height="match_parent"
                    android:textAppearance="@style/TextAppearance.StatusBar.Clock"
                    android:singleLine="true"
                    android:paddingStart="@dimen/status_bar_left_clock_starting_padding"
                    android:paddingEnd="@dimen/status_bar_left_clock_end_padding"
                    android:gravity="center_vertical|start"
                /-->

                <com.android.systemui.statusbar.AlphaOptimizedFrameLayout
                    android:id="@+id/notification_icon_area"
                    android:layout_width="0dp"
                    android:layout_height="match_parent"
                    android:layout_weight="1"
                    android:orientation="horizontal"
                    android:clipChildren="false"/>

            </LinearLayout>
            
            // Clock位置的调整 放在通知栏的后面，然后属性居中显示
            // add code start 
		<com.android.systemui.statusbar.policy.Clock
			android:id="@+id/clock"
			android:layout_width="wrap_content"
			android:layout_height="match_parent"
			android:textAppearance="@style/TextAppearance.StatusBar.Clock"
			android:singleLine="true"
			android:maxEms="50"
			android:gravity="center_horizontal"/>
        // add code end 
            
        </FrameLayout>

        <!-- Space should cover the notch (if it exists) and let other views lay out around it -->
        <android.widget.Space
            android:id="@+id/cutout_space_view"
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:gravity="center_horizontal|center_vertical"
        />

        <com.android.systemui.statusbar.AlphaOptimizedFrameLayout
            android:id="@+id/centered_icon_area"
            android:layout_width="wrap_content"
            android:layout_height="match_parent"
            android:orientation="horizontal"
            android:clipChildren="false"
            android:gravity="center_horizontal|center_vertical"/>

        <com.android.keyguard.AlphaOptimizedLinearLayout android:id="@+id/system_icon_area"
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="1"
            android:orientation="horizontal"
            android:gravity="center_vertical|end"
            >

            <include layout="@layout/system_icons" />
        </com.android.keyguard.AlphaOptimizedLinearLayout>
    </LinearLayout>

    <ViewStub
        android:id="@+id/emergency_cryptkeeper_text"
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:layout="@layout/emergency_cryptkeeper_text"
    />

</com.android.systemui.statusbar.phone.PhoneStatusBarView>

    
    mContext.getResources().getDimensionPixelSize(
                    R.dimen.status_bar_clock_starting_padding),
            0,
            mContext.getResources().getDimensionPixelSize(
                    R.dimen.status_bar_clock_end_padding),
            0);
}

```

// 布局显示

setPaddingRelative(  
mContext.getResources().getDimensionPixelSize(  
R.dimen.status_bar_clock_starting_padding),  
0,  
mContext.getResources().getDimensionPixelSize(  
R.dimen.status_bar_clock_end_padding),  
0);

时钟布局显示不居中，修改为:

```
 @Override
    public void onDensityOrFontScaleChanged() {
        FontSizeUtils.updateFontSize(this, R.dimen.status_bar_clock_size);
        /*setPaddingRelative(
                mContext.getResources().getDimensionPixelSize(
                        R.dimen.status_bar_clock_starting_padding),
                0,
                mContext.getResources().getDimensionPixelSize(
                        R.dimen.status_bar_clock_end_padding),
                0);*/
		setPaddingRelative(750,0,0,0);
    }

    @Override
    public void dispatchDemoCommand(String command, Bundle args) {
        if (!mDemoMode && command.equals(COMMAND_ENTER)) {
            mDemoMode = true;
        } else if (mDemoMode && command.equals(COMMAND_EXIT)) {
            mDemoMode = false;
            updateClock();
        } else if (mDemoMode && command.equals(COMMAND_CLOCK)) {
            String millis = args.getString("millis");
            String hhmm = args.getString("hhmm");
            if (millis != null) {
                mCalendar.setTimeInMillis(Long.parseLong(millis));
            } else if (hhmm != null && hhmm.length() == 4) {
                int hh = Integer.parseInt(hhmm.substring(0, 2));
                int mm = Integer.parseInt(hhmm.substring(2));
                boolean is24 = DateFormat.is24HourFormat(getContext(), mCurrentUserId);
                if (is24) {
                    mCalendar.set(Calendar.HOUR_OF_DAY, hh);
                } else {
                    mCalendar.set(Calendar.HOUR, hh);
                }
                mCalendar.set(Calendar.MINUTE, mm);
            }
            setText(getSmallTime());
            setContentDescription(mContentDescriptionFormat.format(mCalendar.getTime()));
        }
    }

```

代码中 setText(); 来显示时间 而时间又是由 getSmallTime() 来计算的  
接下来 看 getSmallTime();

```
private final CharSequence getSmallTime() {
        Context context = getContext();
        boolean is24 = DateFormat.is24HourFormat(context, mCurrentUserId);
        LocaleData d = LocaleData.get(context.getResources().getConfiguration().locale);

        final char MAGIC1 = '\uEF00';
        final char MAGIC2 = '\uEF01';

        SimpleDateFormat sdf;
        String format = mShowSeconds
                ? is24 ? d.timeFormat_Hms : d.timeFormat_hms
                : is24 ? d.timeFormat_Hm : d.timeFormat_hm;
        if (!format.equals(mClockFormatString)) {
            mContentDescriptionFormat = new SimpleDateFormat(format);
            /*
             * Search for an unquoted "a" in the format string, so we can
             * add dummy characters around it to let us find it again after
             * formatting and change its size.
             */
            if (mAmPmStyle != AM_PM_STYLE_NORMAL) {
                int a = -1;
                boolean quoted = false;
                for (int i = 0; i < format.length(); i++) {
                    char c = format.charAt(i);

                    if (c == '\'') {
                        quoted = !quoted;
                    }
                    if (!quoted && c == 'a') {
                        a = i;
                        break;
                    }
                }

                if (a >= 0) {
                    // Move a back so any whitespace before AM/PM is also in the alternate size.
                    final int b = a;
                    while (a > 0 && Character.isWhitespace(format.charAt(a-1))) {
                        a--;
                    }
                    format = format.substring(0, a) + MAGIC1 + format.substring(a, b)
                        + "a" + MAGIC2 + format.substring(b + 1);
                }
            }
            mClockFormat = sdf = new SimpleDateFormat(format);
            mClockFormatString = format;
        } else {
            sdf = mClockFormat;
        }
        String result = sdf.format(mCalendar.getTime());

        /* UNISOC: 1072085 clock add am/pm @{ */
        if(KeyguardSupportAmPm.getInstance(mContext).isEnabled()) {
            mAmPmStyle = AM_PM_STYLE_SMALL;
        }
        /* @} */

        if (mAmPmStyle != AM_PM_STYLE_NORMAL) {
            int magic1 = result.indexOf(MAGIC1);
            int magic2 = result.indexOf(MAGIC2);
            if (magic1 >= 0 && magic2 > magic1) {
                SpannableStringBuilder formatted = new SpannableStringBuilder(result);
                if (mAmPmStyle == AM_PM_STYLE_GONE) {
                    formatted.delete(magic1, magic2+1);
                } else {
                    if (mAmPmStyle == AM_PM_STYLE_SMALL) {
                        CharacterStyle style = new RelativeSizeSpan(0.7f);
                        formatted.setSpan(style, magic1, magic2,
                                          Spannable.SPAN_EXCLUSIVE_INCLUSIVE);
                    }
                    formatted.delete(magic2, magic2 + 1);
                    formatted.delete(magic1, magic1 + 1);
                }
                return formatted;
            }
        }

        return result;

    }

```

时间样式修改为:

```
  private final CharSequence getSmallTime() {
        Context context = getContext();
        boolean is24 = DateFormat.is24HourFormat(context, mCurrentUserId);
        LocaleData d = LocaleData.get(context.getResources().getConfiguration().locale);
    +    onDensityOrFontScaleChanged();
        final char MAGIC1 = '\uEF00';
        final char MAGIC2 = '\uEF01';

        SimpleDateFormat sdf;
        String format = mShowSeconds
                ? is24 ? d.timeFormat_Hms : d.timeFormat_hms
                : is24 ? d.timeFormat_Hm : d.timeFormat_hm;
        if (!format.equals(mClockFormatString)) {
            mContentDescriptionFormat = new SimpleDateFormat(format);
            /*
             * Search for an unquoted "a" in the format string, so we can
             * add dummy characters around it to let us find it again after
             * formatting and change its size.
             */
            if (mAmPmStyle != AM_PM_STYLE_NORMAL) {
                int a = -1;
                boolean quoted = false;
                for (int i = 0; i < format.length(); i++) {
                    char c = format.charAt(i);

                    if (c == '\'') {
                        quoted = !quoted;
                    }
                    if (!quoted && c == 'a') {
                        a = i;
                        break;
                    }
                }

                if (a >= 0) {
                    // Move a back so any whitespace before AM/PM is also in the alternate size.
                    final int b = a;
                    while (a > 0 && Character.isWhitespace(format.charAt(a-1))) {
                        a--;
                    }
-                    format = format.substring(0, a) + MAGIC1 + format.substring(a, b)
-                        + "a" + MAGIC2 + format.substring(b + 1);
+                    format = format.substring(0, a) + format.substring(a, b)
+                        + "a" + format.substring(b + 1);
                }
            }
            mClockFormat = sdf = new SimpleDateFormat(format);
            mClockFormatString = format;
        } else {
            sdf = mClockFormat;
        }
        String result = sdf.format(mCalendar.getTime());
   +     String str=null;
        /* UNISOC: 1072085 clock add am/pm @{ */
        if(KeyguardSupportAmPm.getInstance(mContext).isEnabled()) {
            -            mAmPmStyle = AM_PM_STYLE_SMALL;
            +            mAmPmStyle = AM_PM_STYLE_NORMAL;
        }
        /* @} */
        +     long time=System.currentTimeMillis();
        +    Date date=new Date(time);
        +    SimpleDateFormat format2=new SimpleDateFormat("yyyy年MM月dd日 EEEE");

            int magic1 = result.indexOf(MAGIC1);
            int magic2 = result.indexOf(MAGIC2);
            if (magic1 >= 0 && magic2 > magic1) {
                SpannableStringBuilder formatted = new SpannableStringBuilder(result);
                if (mAmPmStyle == AM_PM_STYLE_GONE) {
                    formatted.delete(magic1, magic2+1);
                } else {
                    if (mAmPmStyle == AM_PM_STYLE_SMALL) {
                        CharacterStyle style = new RelativeSizeSpan(0.7f);
                        formatted.setSpan(style, magic1, magic2,
                                          Spannable.SPAN_EXCLUSIVE_INCLUSIVE);
                    }
                    formatted.delete(magic2, magic2 + 1);
                    formatted.delete(magic1, magic1 + 1);
                }
                //return formatted;
              // add code start
		str = formatted+"";
                if(str.contains("上午")){
                    str=str.substring(2)+" AM";
                }else if(str.contains("下午")){
                    str=str.substring(2)+" PM";
                }              
                return format2.format(date)+" "+str;
               // add code end
            }
        // add code start
     +       str = result;
     +       if(str.contains("上午")){
     +           str=str.substring(2)+" AM";
     +       }else if(str.contains("下午")){
    +            str=str.substring(2)+" PM";
      +      }
      +      return format2.format(date)+" "+str;
 // add code end
//        return result;

    }

```

修改 git 记录:

```
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/policy/Clock.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/policy/Clock.java
@@ -37,7 +37,7 @@ import android.util.AttributeSet;
 import android.view.Display;
 import android.view.View;
 import android.widget.TextView;
-
+import java.util.Date;
 import com.android.settingslib.Utils;
 import com.android.systemui.DemoMode;
 import com.android.systemui.Dependency;
@@ -330,13 +330,14 @@ public class Clock extends TextView implements DemoMode, Tunable, CommandQueue.C
     @Override
     public void onDensityOrFontScaleChanged() {
         FontSizeUtils.updateFontSize(this, R.dimen.status_bar_clock_size);
-        setPaddingRelative(
+        /*setPaddingRelative(
                 mContext.getResources().getDimensionPixelSize(
                         R.dimen.status_bar_clock_starting_padding),
                 0,
                 mContext.getResources().getDimensionPixelSize(
                         R.dimen.status_bar_clock_end_padding),
-                0);
+                0);*/
+               setPaddingRelative(750,0,0,0);
     }
 
     /**
@@ -385,7 +386,7 @@ public class Clock extends TextView implements DemoMode, Tunable, CommandQueue.C
         Context context = getContext();
         boolean is24 = DateFormat.is24HourFormat(context, mCurrentUserId);
         LocaleData d = LocaleData.get(context.getResources().getConfiguration().locale);
-
+        onDensityOrFontScaleChanged();
         final char MAGIC1 = '\uEF00';
         final char MAGIC2 = '\uEF01';
 
@@ -421,8 +422,8 @@ public class Clock extends TextView implements DemoMode, Tunable, CommandQueue.C
                     while (a > 0 && Character.isWhitespace(format.charAt(a-1))) {
                         a--;
                     }
-                    format = format.substring(0, a) + MAGIC1 + format.substring(a, b)
-                        + "a" + MAGIC2 + format.substring(b + 1);
+                    format = format.substring(0, a) + format.substring(a, b)
+                        + "a" + format.substring(b + 1);
                 }
             }
             mClockFormat = sdf = new SimpleDateFormat(format);
@@ -431,14 +432,16 @@ public class Clock extends TextView implements DemoMode, Tunable, CommandQueue.C
             sdf = mClockFormat;
         }
         String result = sdf.format(mCalendar.getTime());
-
+        String str=null;
         /* UNISOC: 1072085 clock add am/pm @{ */
         if(KeyguardSupportAmPm.getInstance(mContext).isEnabled()) {
-            mAmPmStyle = AM_PM_STYLE_SMALL;
+            mAmPmStyle = AM_PM_STYLE_NORMAL;
         }
         /* @} */
+            long time=System.currentTimeMillis();
+            Date date=new Date(time);
+            SimpleDateFormat format2=new SimpleDateFormat("yyyy年MM月dd日 EEEE");
 
-        if (mAmPmStyle != AM_PM_STYLE_NORMAL) {
             int magic1 = result.indexOf(MAGIC1);
             int magic2 = result.indexOf(MAGIC2);
             if (magic1 >= 0 && magic2 > magic1) {
@@ -454,11 +457,25 @@ public class Clock extends TextView implements DemoMode, Tunable, CommandQueue.C
                     formatted.delete(magic2, magic2 + 1);
                     formatted.delete(magic1, magic1 + 1);
                 }
-                return formatted;
+                //return formatted;
+                               str = formatted+"";
+                if(str.contains("上午")){
+                    str=str.substring(2)+" AM";
+                }else if(str.contains("下午")){
+                    str=str.substring(2)+" PM";
+                }              
+                return format2.format(date)+" "+str;
             }
-        }
+        
+            str = result;
+            if(str.contains("上午")){
+                str=str.substring(2)+" AM";
+            }else if(str.contains("下午")){
+                str=str.substring(2)+" PM";
+            }
+            return format2.format(date)+" "+str;
 
-        return result;
+//        return result;
 
     }

```
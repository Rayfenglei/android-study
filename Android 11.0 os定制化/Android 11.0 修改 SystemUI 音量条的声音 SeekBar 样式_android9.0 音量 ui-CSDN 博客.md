> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124754925)

在 11.0 系统中针对 SystemUI 音量条样式的定制化，就是要对 VolumeDialogImpl.java 进行布局的修改就可以了  
接下来先看下 VolumeDialogImpl.java 找下相关的布局

```
private void initDialog() {
        mDialog = new CustomDialog(mContext);

        mConfigurableTexts = new ConfigurableTexts(mContext);
        mHovering = false;
        mShowing = false;
        mWindow = mDialog.getWindow();
        mWindow.requestFeature(Window.FEATURE_NO_TITLE);
        mWindow.setBackgroundDrawable(new ColorDrawable(Color.TRANSPARENT));
        mWindow.clearFlags(WindowManager.LayoutParams.FLAG_DIM_BEHIND
                | WindowManager.LayoutParams.FLAG_LAYOUT_INSET_DECOR);
        mWindow.addFlags(WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                | WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN
                | WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL
                | WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED
                | WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH
                | WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED);
        mWindow.setType(WindowManager.LayoutParams.TYPE_VOLUME_OVERLAY);
        mWindow.setWindowAnimations(com.android.internal.R.style.Animation_Toast);
        WindowManager.LayoutParams lp = mWindow.getAttributes();
        lp.format = PixelFormat.TRANSLUCENT;
        lp.setTitle(VolumeDialogImpl.class.getSimpleName());
        lp.windowAnimations = -1;
        lp.gravity = Gravity.RIGHT | Gravity.CENTER_VERTICAL;
        mWindow.setAttributes(lp);
        mWindow.setLayout(WRAP_CONTENT, WRAP_CONTENT);

        mDialog.setContentView(R.layout.volume_dialog);
        mDialogView = mDialog.findViewById(R.id.volume_dialog);
        //mDialogView.setAlpha(0);
        mDialog.setCanceledOnTouchOutside(true);
        mDialog.setOnShowListener(dialog -> {
            if (!isLandscape()) mDialogView.setTranslationX(mDialogView.getWidth() / 2.0f);
            //mDialogView.setAlpha(0);
            mDialogView.animate()
                    .alpha(1)
                    .translationX(0)
                    .setDuration(DIALOG_SHOW_ANIMATION_DURATION)
                    .setInterpolator(new SystemUIInterpolators.LogDecelerateInterpolator())
                    .withEndAction(() -> {
                        if (!Prefs.getBoolean(mContext, Prefs.Key.TOUCHED_RINGER_TOGGLE, false)) {
                            if (mRingerIcon != null) {
                                mRingerIcon.postOnAnimationDelayed(
                                        getSinglePressFor(mRingerIcon), 1500);
                            }
                        }
                    })
                    .start();
        });

        mDialogView.setOnHoverListener((v, event) -> {
            int action = event.getActionMasked();
            mHovering = (action == MotionEvent.ACTION_HOVER_ENTER)
                    || (action == MotionEvent.ACTION_HOVER_MOVE);
            rescheduleTimeoutH();
            return true;
        });

        mDialogRowsView = mDialog.findViewById(R.id.volume_dialog_rows);
        mRinger = mDialog.findViewById(R.id.ringer);
        if (mRinger != null) {
            mRingerIcon = mRinger.findViewById(R.id.ringer_icon);
            mZenIcon = mRinger.findViewById(R.id.dnd_icon);
            mRinger.setVisibility(View.GONE);//add code
        }

        mODICaptionsView = mDialog.findViewById(R.id.odi_captions);
        if (mODICaptionsView != null) {
            mODICaptionsIcon = mODICaptionsView.findViewById(R.id.odi_captions_icon);
        }
        mODICaptionsTooltipViewStub = mDialog.findViewById(R.id.odi_captions_tooltip_stub);
        if (mHasSeenODICaptionsTooltip && mODICaptionsTooltipViewStub != null) {
            mDialogView.removeView(mODICaptionsTooltipViewStub);
            mODICaptionsTooltipViewStub = null;
        }

        mSettingsView = mDialog.findViewById(R.id.settings_container);
        mSettingsIcon = mDialog.findViewById(R.id.settings);

        if (mRows.isEmpty()) {
            if (!AudioSystem.isSingleVolume(mContext)) {
                addRow(STREAM_ACCESSIBILITY, R.drawable.ic_volume_accessibility,
                        R.drawable.ic_volume_accessibility, true, false);
            }
            addRow(AudioManager.STREAM_MUSIC,
                    R.drawable.ic_volume_media, R.drawable.ic_volume_media_mute, true, true);
            if (!AudioSystem.isSingleVolume(mContext)) {
                addRow(AudioManager.STREAM_RING,
                        R.drawable.ic_volume_ringer, R.drawable.ic_volume_ringer_mute, true, false);
                addRow(STREAM_ALARM,
                        R.drawable.ic_volume_alarm, R.drawable.ic_volume_alarm_mute, true, false);
                addRow(AudioManager.STREAM_VOICE_CALL,
                        com.android.internal.R.drawable.ic_phone,
                        com.android.internal.R.drawable.ic_phone, false, false);
                addRow(AudioManager.STREAM_BLUETOOTH_SCO,
                        R.drawable.ic_volume_bt_sco, R.drawable.ic_volume_bt_sco, false, false);
                addRow(AudioManager.STREAM_SYSTEM, R.drawable.ic_volume_system,
                        R.drawable.ic_volume_system_mute, false, false);
            }
        } else {
            addExistingRows();
        }

        updateRowsH(getActiveRow());
        initRingerH();
        initSettingsH();
        initODICaptionsH();
    }

```

在 initDialog() 中可以看到 volume_dialog.xml 就是音量条布局文件

```
@SuppressLint("InflateParams")
    private void initRow(final VolumeRow row, final int stream, int iconRes, int iconMuteRes,
            boolean important, boolean defaultStream) {
        row.stream = stream;
        row.iconRes = iconRes;
        row.iconMuteRes = iconMuteRes;
        row.important = important;
        row.defaultStream = defaultStream;
        row.view = mDialog.getLayoutInflater().inflate(R.layout.volume_dialog_row, null);
        row.view.setId(row.stream);
        row.view.setTag(row);
        row.header = row.view.findViewById(R.id.volume_row_header);
        row.header.setId(20 * row.stream);
        if (stream == STREAM_ACCESSIBILITY) {
            row.header.setFilters(new InputFilter[] {new InputFilter.LengthFilter(13)});
        }
        row.dndIcon = row.view.findViewById(R.id.dnd_icon);
        row.slider = row.view.findViewById(R.id.volume_row_slider);
        row.slider.setOnSeekBarChangeListener(new VolumeSeekBarChangeListener(row));

        row.anim = null;

        row.icon = row.view.findViewById(R.id.volume_row_icon);
        row.icon.setImageResource(iconRes);
        if (row.stream != AudioSystem.STREAM_ACCESSIBILITY) {
            row.icon.setOnClickListener(v -> {
                Events.writeEvent(mContext, Events.EVENT_ICON_CLICK, row.stream, row.iconState);
                mController.setActiveStream(row.stream);
                if (row.stream == AudioManager.STREAM_RING) {
                    final boolean hasVibrator = mController.hasVibrator();
                    if (mState.ringerModeInternal == AudioManager.RINGER_MODE_NORMAL) {
                        if (hasVibrator) {
                            mController.setRingerMode(AudioManager.RINGER_MODE_VIBRATE, false);
                        } else {
                            final boolean wasZero = row.ss.level == 0;
                            mController.setStreamVolume(stream,
                                    wasZero ? row.lastAudibleLevel : 0);
                        }
                    } else {
                        mController.setRingerMode(AudioManager.RINGER_MODE_NORMAL, false);
                        if (row.ss.level == 0) {
                            mController.setStreamVolume(stream, 1);
                        }
                    }
                } else {
                    final boolean vmute = row.ss.level == row.ss.levelMin;
                    mController.setStreamVolume(stream,
                            vmute ? row.lastAudibleLevel : row.ss.levelMin);
                }
                row.userAttempt = 0;  // reset the grace period, slider updates immediately
            });
        } else {
            row.icon.setImportantForAccessibility(View.IMPORTANT_FOR_ACCESSIBILITY_NO);
        }
    }

```

而每条音量条布局就是在 volume_dialog_row.xml 中

```
<FrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:tag="row"
    android:layout_height="wrap_content"
    android:layout_width="@dimen/volume_dialog_panel_width"
    android:clipChildren="false"
    android:clipToPadding="false"
    android:theme="@style/qs_theme">

    <LinearLayout
        android:layout_height="wrap_content"
        android:layout_width="match_parent"
        android:gravity="center"
        android:layout_gravity="center"
        android:orientation="vertical" >
        <TextView
            android:id="@+id/volume_row_header"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:ellipsize="end"
            android:maxLength="10"
            android:maxLines="1"
            android:visibility="gone"
            android:textColor="?android:attr/colorControlNormal"
            android:textAppearance="@style/TextAppearance.Volume.Header" />
        <FrameLayout
            android:id="@+id/volume_row_slider_frame"
            android:layout_width="match_parent"
            android:layout_marginTop="@dimen/volume_dialog_slider_margin_top"
            android:layout_marginBottom="@dimen/volume_dialog_slider_margin_bottom"
            android:layoutDirection="rtl"
            android:layout_height="@dimen/volume_dialog_slider_height">
            <SeekBar
                android:id="@+id/volume_row_slider"
                android:clickable="true"
                android:layout_width="@dimen/volume_dialog_slider_height"
                android:layout_height="match_parent"
                android:thumb="@drawable/icon_progress"
                android:progressDrawable="@drawable/brightness_progress_drawable"
                android:layoutDirection="rtl"
                android:layout_gravity="center"
                android:rotation="90" />
        </FrameLayout>
        <!--  android:tint="@color/accent_tint_color_selector"-->
        <com.android.keyguard.AlphaOptimizedImageButton
            android:id="@+id/volume_row_icon"
            style="@style/VolumeButtons"
            android:layout_width="@dimen/volume_dialog_tap_target_size"
            android:layout_height="@dimen/volume_dialog_tap_target_size"
            android:background="@drawable/ripple_drawable_20dp"
            android:layout_marginBottom="@dimen/volume_dialog_row_margin_bottom"
            android:soundEffectsEnabled="false" />
    </LinearLayout>

    <include layout="@layout/volume_dnd_icon"/>

</FrameLayout>

```

[SeekBar](https://so.csdn.net/so/search?q=SeekBar&spm=1001.2101.3001.7020) 就是具体音量条  
所以增加样式 android:thumb=“@drawable/icon_progress”  
android:progressDrawable=“@drawable/brightness_progress_drawable”  
icon_progress 就是一张圆形图片  
而 brightness_progress_drawable.xml

```
<layer-list xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:id="@android:id/background"
          android:gravity="center_vertical|fill_horizontal">
        <shape android:shape="rectangle">
            <size android:height="@dimen/seek_bar_height" />
            <solid android:color="#eeeeee" />
            <corners android:radius="@dimen/seek_bar_corner_radius" />
        </shape>
    </item>
    <item android:id="@android:id/progress"
          android:gravity="center_vertical|fill_horizontal">
        <scale android:scaleWidth="100%">
            <shape android:shape="rectangle">
                <size android:height="@dimen/seek_bar_height" />
                <solid android:color="#0091ff" />
                <corners android:radius="@dimen/seek_bar_corner_radius" />
            </shape>
        </scale>
    </item>
</layer-list>

```

然后修改完发现 在音量条消失后 音量条会变灰色 然后第二次 音量条弹出的时候 又变回原来的颜色了  
跟代码发现是对音量条消失做了样式修改 要去掉这部分功能  
具体修改如下:  
1. 隐藏上半部分

```
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/volume/VolumeDialogImpl.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/volume/VolumeDialogImpl.java
@@ -264,6 +264,7 @@ public class VolumeDialogImpl implements VolumeDialog,
         if (mRinger != null) {
             mRingerIcon = mRinger.findViewById(R.id.ringer_icon);
             mZenIcon = mRinger.findViewById(R.id.dnd_icon);
+                       mRinger.setVisibility(View.GONE);//add code
         }
 
         mODICaptionsView = mDialog.findViewById(R.id.odi_captions);   

```

下半部分的处理

```
--- a/frameworks/base/packages/SystemUI/src/com/android/systemui/volume/VolumeDialogImpl.java
+++ b/frameworks/base/packages/SystemUI/src/com/android/systemui/volume/VolumeDialogImpl.java
@@ -230,11 +230,11 @@ public class VolumeDialogImpl implements VolumeDialog,
 
         mDialog.setContentView(R.layout.volume_dialog);
         mDialogView = mDialog.findViewById(R.id.volume_dialog);
-        mDialogView.setAlpha(0);
+        //mDialogView.setAlpha(0);
         mDialog.setCanceledOnTouchOutside(true);
         mDialog.setOnShowListener(dialog -> {
             if (!isLandscape()) mDialogView.setTranslationX(mDialogView.getWidth() / 2.0f);
-            mDialogView.setAlpha(0);
+            //mDialogView.setAlpha(0);
             mDialogView.animate()
                     .alpha(1)
                     .translationX(0)
@@ -550,7 +550,7 @@ public class VolumeDialogImpl implements VolumeDialog,
         }
 
         if (mODICaptionsTooltipView != null) {
-            mODICaptionsTooltipView.setAlpha(0.f);
+            //mODICaptionsTooltipView.setAlpha(0.f);
             mODICaptionsTooltipView.animate()
                 .alpha(1.f)
                 .setStartDelay(DIALOG_SHOW_ANIMATION_DURATION)
@@ -571,7 +571,7 @@ public class VolumeDialogImpl implements VolumeDialog,
     private void hideCaptionsTooltip() {
         if (mODICaptionsTooltipView != null && mODICaptionsTooltipView.getVisibility() == VISIBLE) {
             mODICaptionsTooltipView.animate().cancel();
-            mODICaptionsTooltipView.setAlpha(1.f);
+            //mODICaptionsTooltipView.setAlpha(1.f);
             mODICaptionsTooltipView.animate()
                     .alpha(0.f)
                     .setStartDelay(0)
@@ -752,7 +752,7 @@ public class VolumeDialogImpl implements VolumeDialog,
             Events.writeEvent(mContext, Events.EVENT_DISMISS_DIALOG, reason);
         }
         mDialogView.setTranslationX(0);
-        mDialogView.setAlpha(1);
+        //mDialogView.setAlpha(1);
         ViewPropertyAnimator animator = mDialogView.animate()
                 .alpha(0)
                 .setDuration(DIALOG_HIDE_ANIMATION_DURATION)
@@ -764,7 +764,7 @@ public class VolumeDialogImpl implements VolumeDialog,
         if (!isLandscape()) animator.translationX(mDialogView.getWidth() / 2.0f);
         animator.start();
         checkODICaptionsTooltip(true);
-        mController.notifyVisible(false);
+        //mController.notifyVisible(false);
         synchronized (mSafetyWarningLock) {
             if (mSafetyWarning != null) {
                 if (D.BUG) Log.d(TAG, "SafetyWarning dismissed");

```
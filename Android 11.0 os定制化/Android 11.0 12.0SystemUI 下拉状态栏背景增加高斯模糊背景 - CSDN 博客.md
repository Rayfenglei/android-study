> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124596601)

### 1. 概述

在 11.0 12.0 的产品开发中，发现现在很多产品都是[高斯模糊](https://so.csdn.net/so/search?q=%E9%AB%98%E6%96%AF%E6%A8%A1%E7%B3%8A&spm=1001.2101.3001.7020)背景的，这种高斯模糊背景看起来效果很不错，  
产品开发需要 SystemUI 下拉状态栏背景也是高斯模糊背景，所以就要来实现下拉状态栏高斯模糊背景

### 2.SystemUI 下拉状态栏背景增加高斯模糊背景核心类

```
frameworks/base/packages/SystemUI/res/layout/status_bar_expanded.xml
frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NotificationPanelViewController.java

```

### 3.SystemUI 下拉状态栏背景增加高斯模糊背景核心功能分析和实现

需要给 SystemUI 下拉状态栏布局增加高斯模糊背景，首选要从它的布局入手，然后根据布局分析，添加高斯模糊背景  
它的布局就是 status_bar_expanded.xml，而从布局中发现处理下拉的事件就是在 NotificationPanelView.java 中处理

### 1. 在 status_bar_expanded.xml 关于布局的相关源码

```
--- a/frameworks/base/packages/SystemUI/res/layout/status_bar_expanded.xml
+++ b/frameworks/base/packages/SystemUI/res/layout/status_bar_expanded.xml
@@ -24,6 +24,19 @@
android:layout_width="match_parent"
android:layout_height="match_parent"
android:background="@android:color/transparent" >
//增加部分
<ImageView
   android:id="@+id/blur_view"

   android:layout_width="match_parent"

   android:scaleType="fitXY"

   android:layout_height="match_parent"/>

<View
   android:id="@+id/alpha_view"

   android:layout_width="match_parent"

   android:layout_height="match_parent"/>

// 增加结束
<FrameLayout
android:id="@+id/big_clock_container"
android:layout_width="match_parent"
android:layout_height="match_parent"
android:visibility="gone" />
 <include
     layout="@layout/keyguard_status_view"
    android:visibility="gone" />

```

在下拉状态栏的布局 status_bar_expanded.xml 布局中，增加全屏 view 来作为显示高斯模糊的背景

### 3.2 添加自定义高斯模糊实现类

```
package com.android.systemui.util;
import android.content.Context;
import android.graphics.Bitmap;
import android.graphics.Matrix;
import android.graphics.drawable.BitmapDrawable;
import android.renderscript.Allocation;
import android.renderscript.Element;
import android.renderscript.RenderScript;
import android.renderscript.ScriptIntrinsicBlur;
public class BlurUtil {
private static final float BITMAP_SCALE = 0.4f;
private static final int BLUR_RADIUS = 7;
public static final int BLUR_RADIUS_MAX = 25;
public static Bitmap blur(Context context, Bitmap bitmap) {
return blur(context, bitmap, BITMAP_SCALE, BLUR_RADIUS);
}
public static Bitmap blur(Context context, Bitmap bitmap, float bitmap_scale) {
return blur(context, bitmap, bitmap_scale, BLUR_RADIUS);
}
public static Bitmap blur(Context context, Bitmap bitmap, int blur_radius) {
//return blur(context, bitmap, BITMAP_SCALE, blur_radius);
return blurbitmap(context,bitmap);
}
public static Bitmap blur(Context context, Bitmap bitmap, float bitmap_scale, int blur_radius) {
Bitmap inputBitmap = Bitmap.createScaledBitmap(bitmap, Math.round(bitmap.getWidth() * bitmap_scale),
Math.round(bitmap.getHeight() * bitmap_scale), false);
Bitmap outputBitmap = Bitmap.createBitmap(inputBitmap);
RenderScript rs = RenderScript.create(context);
ScriptIntrinsicBlur theIntrinsic = ScriptIntrinsicBlur.create(rs, Element.U8_4(rs));
Allocation tmpIn = Allocation.createFromBitmap(rs, inputBitmap);
Allocation tmpOut = Allocation.createFromBitmap(rs, outputBitmap);
theIntrinsic.setRadius(blur_radius);
theIntrinsic.setInput(tmpIn);
theIntrinsic.forEach(tmpOut);
tmpOut.copyTo(outputBitmap);
rs.destroy();
bitmap.recycle();
return outputBitmap;
}
public static Bitmap blurbitmap(Context context, Bitmap bitmap) {
//用需要创建高斯模糊bitmap创建一个空的bitmap
Bitmap outBitmap = Bitmap.createBitmap(bitmap.getWidth(), bitmap.getHeight(), Bitmap.Config.ARGB_8888);
// 初始化Renderscript，该类提供了RenderScript context，创建其他RS类之前必须先创建这个类，其控制RenderScript的初始化，资源管理及释放
RenderScript rs = RenderScript.create(context);
// 创建高斯模糊对象
ScriptIntrinsicBlur blurScript = ScriptIntrinsicBlur.create(rs, Element.U8_4(rs));
// 创建Allocations，此类是将数据传递给RenderScript内核的主要方 法，并制定一个后备类型存储给定类型
Allocation allIn = Allocation.createFromBitmap(rs, bitmap);
Allocation allOut = Allocation.createFromBitmap(rs, outBitmap);
//设定模糊度(注：Radius最大只能设置25.f)
blurScript.setRadius(25.0f);
// Perform the Renderscript
blurScript.setInput(allIn);
blurScript.forEach(allOut);
// Copy the final bitmap created by the out Allocation to the outBitmap
allOut.copyTo(outBitmap);
// recycle the original bitmap
// bitmap.recycle();
// After finishing everything, we destroy the Renderscript.
rs.destroy();
return outBitmap;
}
private static Bitmap blurBitmap(Bitmap bkg,Context context) {
//设定模糊度(注：Radius最大只能设置25.f)
float radius = 25.0f;
//背景图片缩放处理
bkg = smallBitmap(bkg);
Bitmap bitmap = bkg.copy(bkg.getConfig(), false);
final RenderScript rs = RenderScript.create(context);
final Allocation input = Allocation.createFromBitmap(rs, bkg, Allocation.MipmapControl.MIPMAP_NONE,
Allocation.USAGE_SCRIPT);
final Allocation output = Allocation.createTyped(rs, input.getType());
final ScriptIntrinsicBlur script = ScriptIntrinsicBlur.create(rs, Element.U8_4(rs));
script.setRadius(radius);
script.setInput(input);
script.forEach(output);
output.copyTo(bitmap);
//背景图片放大处理
bitmap = bigBitmap(bitmap);
rs.destroy();
return bitmap;
}
private static Bitmap bigBitmap(Bitmap bitmap) {
Matrix matrix = new Matrix();
matrix.postScale(4f,4f); //长和宽放大缩小的比例
Bitmap resizeBmp = Bitmap.createBitmap(bitmap,0,0,bitmap.getWidth(),bitmap.getHeight(),matrix,true);
return resizeBmp;
}
private static Bitmap smallBitmap(Bitmap bitmap) {
Matrix matrix = new Matrix();
matrix.postScale(0.25f,0.25f); //长和宽放大缩小的比例
Bitmap resizeBmp = Bitmap.createBitmap(bitmap,0,0,bitmap.getWidth(),bitmap.getHeight(),matrix,true);
return resizeBmp;
}
}

```

通过 RenderScript 关于高斯模糊的 api 来实现高斯模糊相关参数的设置从而达到设置高斯模糊背景的方法

### 3.3 增加 BitmapUtils.java 的工具类

```
package com.android.systemui.util;
import android.graphics.Bitmap;
import android.graphics.drawable.BitmapDrawable;
import android.graphics.drawable.Drawable;
import android.view.View;
import android.widget.ImageView;
public class BitmapUtils {
public static void recycleImageView(View view) {
if (view == null) return;
if (view instanceof ImageView) {
Drawable drawable = ((ImageView) view).getDrawable();
if (drawable instanceof BitmapDrawable) {
Bitmap bmp = ((BitmapDrawable) drawable).getBitmap();
if (bmp != null && !bmp.isRecycled()) {
((ImageView) view).setImageBitmap(null);
bmp.recycle();
bmp = null;
}
}
}
}
}

```

3.4 在 NotificationPanelViewController.java 下拉监听事件中，设置高斯模糊背景颜色  
由于在 11.0 由 NotificationPanelViewController.java 来处理 触摸等事件 所以添加高斯模糊背景就在这里添加  
路径:/frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NotificationPanelViewController.java

```
private static final int BLUR_START = 200;
private static final int BLUR_END = 700;
private View mAlphaView;
private ImageView mBlurView;
private void onFinishInflate() {
mAlphaView = findViewById(R.id.alpha_view);
mBlurView = findViewById(R.id.blur_view);
}
@Override
protected TouchHandler createTouchHandler() {
return new TouchHandler() {
@Override
public boolean onInterceptTouchEvent(MotionEvent event) {
if (mBlockTouches || mQsFullyExpanded && mQs.disallowPanelTouches()) {
return false;
}
initDownStates(event);
// Do not let touches go to shade or QS if the bouncer is visible,
// but still let user swipe down to expand the panel, dismissing the bouncer.
if (mStatusBar.isBouncerShowing()) {
return true;
}
if (mBar.panelEnabled() && mHeadsUpTouchHelper.onInterceptTouchEvent(event)) {
mMetricsLogger.count(COUNTER_PANEL_OPEN, 1);
mMetricsLogger.count(COUNTER_PANEL_OPEN_PEEK, 1);
return true;
}
if (!shouldQuickSettingsIntercept(mDownX, mDownY, 0)
&& mPulseExpansionHandler.onInterceptTouchEvent(event)) {
return true;
}         if (!isFullyCollapsed() && onQsIntercept(event)) {
            return true;
         }
        return super.onInterceptTouchEvent(event);
     }

     @Override
     public boolean onTouch(View v, MotionEvent event) {
         if (mBlockTouches || (mQsFullyExpanded && mQs != null
                 && mQs.disallowPanelTouches())) {
             return false;
         }

         // Do not allow panel expansion if bouncer is scrimmed, otherwise user would be able
         // to pull down QS or expand the shade.
        if (mStatusBar.isBouncerShowingScrimmed()) {
             return false;
         }

         // Make sure the next touch won't the blocked after the current ends.
         if (event.getAction() == MotionEvent.ACTION_UP
                 || event.getAction() == MotionEvent.ACTION_CANCEL) {
             mBlockingExpansionForCurrentTouch = false;
         }
         // When touch focus transfer happens, ACTION_DOWN->ACTION_UP may happen immediately
         // without any ACTION_MOVE event.
         // In such case, simply expand the panel instead of being stuck at the bottom bar.
         if (mLastEventSynthesizedDown && event.getAction() == MotionEvent.ACTION_UP) {
             expand(true /* animate */);
         }
         initDownStates(event);
         if (!mIsExpanding && !shouldQuickSettingsIntercept(mDownX, mDownY, 0)
                && mPulseExpansionHandler.onTouchEvent(event)) {
             // We're expanding all the other ones shouldn't get this anymore
            return true;
         }
         if (mListenForHeadsUp && !mHeadsUpTouchHelper.isTrackingHeadsUp()
                 && mHeadsUpTouchHelper.onInterceptTouchEvent(event)) {
            mMetricsLogger.count(COUNTER_PANEL_OPEN_PEEK, 1);
         }
         boolean handled = false;
         if ((!mIsExpanding || mHintAnimationRunning) && !mQsExpanded
                && mBarState != StatusBarState.SHADE && !mDozing) {
             handled |= mAffordanceHelper.onTouchEvent(event);
         }
         if (mOnlyAffordanceInThisMotion) {
             return true;
         }
         handled |= mHeadsUpTouchHelper.onTouchEvent(event);

         if (!mHeadsUpTouchHelper.isTrackingHeadsUp() && handleQsTouch(event)) {
             return true;
         }
         if (event.getActionMasked() == MotionEvent.ACTION_DOWN && isFullyCollapsed()) {
             mMetricsLogger.count(COUNTER_PANEL_OPEN, 1);
             updateVerticalPanelPosition(event.getX());
            handled = true;
        }

//添加 begin
float y = event.getY();
switch (event.getAction()) {
   case MotionEvent.ACTION_DOWN:

       if (!mPanelExpanded) {

           setBlurBackground();

       }

       break;

   case MotionEvent.ACTION_MOVE:

       if (y <= BLUR_END) {

           float alpha = (y - BLUR_START) / (BLUR_END - BLUR_START);

           if (alpha < 0) {

               alpha = 0;

           }

           mAlphaView.setAlpha(alpha);

           mBlurView.setAlpha(alpha);

       }

       break;

   case MotionEvent.ACTION_UP:

       float a = mBlurView.getAlpha();

       startAlphaAnimation(a, 1.0f);

       break;

}
// 添加 end
             handled |= super.onTouch(v, event);
              return !mDozing || mPulsing || handled;
          }
      };
  }

```

在 NotificationPanelViewController 的 onTouch(View v, MotionEvent event) 添加设置高斯模糊

```
private void startAlphaAnimation(float start, float end) {
ValueAnimator va = ValueAnimator.ofFloat(start, end);
va.setDuration((long) (Math.abs(end - start) * 500));
va.start();
va.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
   @Override
   public void onAnimationUpdate(ValueAnimator animation) {
       float alpha = (float) animation.getAnimatedValue();
       mAlphaView.setAlpha(alpha/2);
       mBlurView.setAlpha(alpha);
   }
});
}
private void setBlurBackground() {
Bitmap bitmap = BitmapFactory.decodeResource(mContext.getResources(), R.drawable.notification_bg);/ScreenShotUtil.takeScreenShot(mContext)/;
if (bitmap == null) {
   Log.d("NotificationPanelView", "setBlurBackground bitmap == null");
   return;
}
//bitmap.setConfig(Bitmap.Config.ARGB_8888);
Bitmap blurBitmap = BlurUtil.blur(mContext, bitmap, BlurUtil.BLUR_RADIUS_MAX);
BitmapUtils.recycleImageView(mBlurView);
mBlurView.setImageBitmap(blurBitmap);
}

```
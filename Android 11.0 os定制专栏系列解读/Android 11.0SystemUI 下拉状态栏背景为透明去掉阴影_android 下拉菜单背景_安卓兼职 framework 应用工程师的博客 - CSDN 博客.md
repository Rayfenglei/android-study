> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124790012)

11.0 对于 SystemUI 状态栏的定制也是非常多的，最近有需求要求在下拉状态栏时，背景去掉原来的灰色，改为透明色  
所以就要从状态栏下拉中，找到灰色背景是怎么生成的

接下来先看下  
StatusBar  
从相关的布局文件 [xml](https://so.csdn.net/so/search?q=xml&spm=1001.2101.3001.7020) 中可以找到状态栏主要的 Layout：  
SystemUI 下拉通知栏的布局为 super_status_bar.xml

```
<!-- This is the combined status bar / notification panel window. -->
<com.android.systemui.statusbar.phone.StatusBarWindowView
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true">

 ...

    <com.android.systemui.statusbar.ScrimView android:id="@+id/scrim_behind"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:importantForAccessibility="no" />

    <include layout="@layout/status_bar"
        android:layout_width="match_parent"
        android:layout_height="@dimen/status_bar_height" />

    <FrameLayout android:id="@+id/brightness_mirror"
                 android:layout_width="@dimen/notification_panel_width"
                 android:layout_height="wrap_content"
                 android:layout_gravity="@integer/notification_panel_layout_gravity"
                 android:paddingLeft="@dimen/notification_side_padding"
                 android:paddingRight="@dimen/notification_side_padding"
                 android:visibility="gone">
        <FrameLayout
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:elevation="2dp"
                android:background="@drawable/brightness_mirror_background">
            <include layout="@layout/quick_settings_brightness_dialog"
                     android:layout_width="match_parent"
                     android:layout_height="wrap_content" />
        </FrameLayout>
    </FrameLayout>

    <com.android.systemui.statusbar.phone.PanelHolder
        android:id="@+id/panel_holder"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:background="@color/transparent" >
        <include layout="@layout/status_bar_expanded"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:visibility="gone" />
    </com.android.systemui.statusbar.phone.PanelHolder>

    <com.android.systemui.statusbar.ScrimView android:id="@+id/scrim_in_front"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:importantForAccessibility="no" />

</com.android.systemui.statusbar.phone.StatusBarWindowView>

```

分析如下:  
1 StatusBarWindowView 是状态栏根布局  
2 BackDropView  
3 ScrimView 是状态栏下拉后，背景，半透明灰色  
4 status_bar 状态栏的布局  
5 PanelHolder，下拉通知栏布局

所以说具体来看 ScrimView.java  
路径为: /framework/base/packages/SystemUI/src/com/android/systemui/statusbar/ScrimView.java

```
    @Override
    protected void onDraw(Canvas canvas) {
        if (mDrawable.getAlpha() > 0) {
            mDrawable.draw(canvas);
        }
    }
 绘制背景

   
    public void setViewAlpha(float alpha) {
        if (alpha != mViewAlpha) {
            mViewAlpha = alpha;

            mDrawable.setAlpha((int) (255 * alpha));
            if (mChangeRunnable != null) {
                mChangeRunnable.run();
            }
        }
    }

```

public void setViewAlpha(float alpha) 即为设置透明度

所以要设置无透明就在这里修改就可以了 修改为:

```
     public void setViewAlpha(float alpha) {
         if (alpha != mViewAlpha) {
-            mViewAlpha = alpha;
+            alpha = 0;
+            mViewAlpha = 0;
+           //mViewAlpha = alpha;
             if (mAlphaAnimator != null) {
                 mAlphaAnimator.cancel();
             } 

```
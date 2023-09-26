> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/128412431)

**目录**

[1. 概述](#1.%E6%A6%82%E8%BF%B0)

[2.SystemUI 音量条 UI 定制的功能 (一) 的核心类](#2.SystemUI%20%E9%9F%B3%E9%87%8F%E6%9D%A1UI%E5%AE%9A%E5%88%B6%E7%9A%84%E5%8A%9F%E8%83%BD%28%E4%B8%80%29%E7%9A%84%E6%A0%B8%E5%BF%83%E7%B1%BB)

[3.SystemUI 音量条 UI 定制的功能的核心功能 (一) 分析和实现](#3.SystemUI%20%E9%9F%B3%E9%87%8F%E6%9D%A1UI%E5%AE%9A%E5%88%B6%E7%9A%84%E5%8A%9F%E8%83%BD%E7%9A%84%E6%A0%B8%E5%BF%83%E5%8A%9F%E8%83%BD%28%E4%B8%80%29%E5%88%86%E6%9E%90%E5%92%8C%E5%AE%9E%E7%8E%B0)

[3.1 volume_dialog.xml 中相关布局的分析](#t3)

[3.2 ic_volume_media.xml 和 ic_volume_media_mute.xml 关于音量条下静音的图标布局的修改](#t4)

1. 概述
-----

在 11.0 的系统软件开发中，对于 [SystemUI](https://so.csdn.net/so/search?q=SystemUI&spm=1001.2101.3001.7020) 音量条的定制也是常有的功能，项目需求要求在音量条 UI 做定制 根据 ui 设计图来重新定制音量条 UI，这就需要在 SystemUI 中找到对应的音量条布局来重新设置音量条布局，来完成 音量条 UI 的布局设置，下面来实现第一部分

效果图如图:

![](https://img-blog.csdnimg.cn/25105ceeeea841cf91d9d20c1f606cc7.png)

2.SystemUI 音量条 UI 定制的功能 (一) 的核心类
--------------------------------

```
frameworks/base/packages/SystemUI/src/com/android/systemui/volume/VolumeDialogImpl.java
frameworks/base/packages/apps/SystemUI/res/drawable/ic_volume_media.xml
frameworks/base/packages/apps/SystemUI/res/drawable/ic_volume_media_mute.xml
```

3.SystemUI 音量条 UI 定制的功能的核心功能 (一) 分析和实现
--------------------------------------

在 SystemUI 中的音量条其实就是 VolumeDialogImpl.java 这里主要处理音量条的相关逻辑，而在 它的音量条布局就是 volume_dialog.xml，接下来分析下 volume_dialog.xml 关于音量条布局

3.1 volume_dialog.xml 中相关布局的分析
------------------------------

```
<FrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:sysui="http://schemas.android.com/apk/res-auto"
    android:id="@+id/volume_dialog_container"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:gravity="right"
    android:layout_gravity="right"
    android:background="@android:color/transparent"
    android:theme="@style/qs_theme">
 
    <!-- right-aligned to be physically near volume button -->
    <LinearLayout
        android:id="@+id/volume_dialog"
        android:minWidth="@dimen/volume_dialog_panel_width"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:gravity="right"
        android:layout_gravity="right"
        android:background="@android:color/transparent"
        android:paddingRight="@dimen/volume_dialog_panel_transparent_padding_right"
        android:paddingLeft="@dimen/volume_dialog_panel_transparent_padding"
        android:orientation="vertical"
        android:clipToPadding="false">
 
        <FrameLayout
            android:id="@+id/ringer"
            android:layout_width="@dimen/volume_dialog_ringer_size"
            android:layout_height="0dp"
            android:gravity="right"
            android:layout_gravity="right"
            android:translationZ="@dimen/volume_dialog_elevation"
            android:clipToPadding="false">
            <com.android.keyguard.AlphaOptimizedImageButton
                android:id="@+id/ringer_icon"
                style="@style/VolumeButtons"
                android:background="@drawable/rounded_ripple"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:scaleType="fitCenter"
                android:padding="@dimen/volume_dialog_ringer_icon_padding"
                android:tint="@color/accent_tint_color_selector"
                android:layout_gravity="center"
                android:soundEffectsEnabled="false" />
 
            <include layout="@layout/volume_dnd_icon"
                     android:layout_width="match_parent"
                     android:layout_height="wrap_content"
                     android:layout_marginRight="@dimen/volume_dialog_stream_padding"
                     android:layout_marginTop="6dp"/>
        </FrameLayout>
 
        <LinearLayout
            android:id="@+id/main"
            android:minWidth="@dimen/volume_dialog_panel_width"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:gravity="right|center_vertical"
            android:layout_gravity="right|center_vertical"
            android:orientation="vertical"
            android:translationZ="@dimen/volume_dialog_elevation"
            android:clipChildren="false"
            android:clipToPadding="false"
       +     android:background="@drawable/volume_rounded_bg" >
            <LinearLayout
                android:id="@+id/volume_dialog_rows"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:minWidth="@dimen/volume_dialog_panel_width"
                android:gravity="center"
                android:orientation="horizontal"
                android:paddingRight="@dimen/volume_dialog_stream_padding"
                android:paddingLeft="@dimen/volume_dialog_stream_padding">
                    <!-- volume rows added and removed here! :-) -->
            </LinearLayout>
            <FrameLayout
                android:id="@+id/settings_container"
                android:layout_width="match_parent"
                android:layout_height="0px">
                <com.android.keyguard.AlphaOptimizedImageButton
                    android:id="@+id/settings"
                    android:src="@drawable/ic_tune_black_16dp"
                    android:layout_width="@dimen/volume_dialog_tap_target_size"
                    android:layout_height="@dimen/volume_dialog_tap_target_size"
                    android:layout_gravity="center"
                    android:contentDescription="@string/accessibility_volume_settings"
                    android:background="@drawable/ripple_drawable_20dp"
                    android:tint="?android:attr/textColorSecondary"
                    android:soundEffectsEnabled="false" />
            </FrameLayout>
        </LinearLayout>
 
       ..
 
    </LinearLayout>
 
    <ViewStub
        android:id="@+id/odi_captions_tooltip_stub"
        android:inflatedId="@+id/odi_captions_tooltip_view"
        android:layout="@layout/volume_tool_tip_view"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="bottom | right"
        android:layout_marginRight="@dimen/volume_tool_tip_right_margin"
        android:layout_marginBottom="@dimen/volume_tool_tip_bottom_margin"/>
 
</FrameLayout>
```

从上述的代码可以看出，在 volume_dialog.xml 的布局中，关于在 android:id="@+id/main" 的 [LinearLayout](https://so.csdn.net/so/search?q=LinearLayout&spm=1001.2101.3001.7020) 的布局就是音量条的  
主要布局，而上面的 android:id="@+id/ringer" 就是铃声布局， android:id="@+id/settings_container"  
就是音量条下面的设置布局，所以只保留音量条就需要隐藏另外两个部分，具体实现如下:

```
--- a/vendor/mediatek/proprietary/packages/apps/SystemUI/res/layout/volume_dialog.xml
+++ b/vendor/mediatek/proprietary/packages/apps/SystemUI/res/layout/volume_dialog.xml
@@ -34,8 +34,6 @@
         android:layout_gravity="right"
         android:background="@android:color/transparent"
         android:paddingRight="@dimen/volume_dialog_panel_transparent_padding_right"
-        android:paddingTop="@dimen/volume_dialog_panel_transparent_padding"
-        android:paddingBottom="@dimen/volume_dialog_panel_transparent_padding"
         android:paddingLeft="@dimen/volume_dialog_panel_transparent_padding"
         android:orientation="vertical"
         android:clipToPadding="false">
@@ -43,13 +41,11 @@
         <FrameLayout
             android:id="@+id/ringer"
             android:layout_width="@dimen/volume_dialog_ringer_size"
-            android:layout_height="@dimen/volume_dialog_ringer_size"
-            android:layout_marginBottom="@dimen/volume_dialog_spacer"
+            android:layout_height="0dp"
             android:gravity="right"
             android:layout_gravity="right"
             android:translationZ="@dimen/volume_dialog_elevation"
-            android:clipToPadding="false"
-            android:background="@drawable/rounded_bg_full">
+            android:clipToPadding="false">
             <com.android.keyguard.AlphaOptimizedImageButton
                 android:id="@+id/ringer_icon"
                 style="@style/VolumeButtons"
@@ -74,13 +70,13 @@
             android:minWidth="@dimen/volume_dialog_panel_width"
             android:layout_width="wrap_content"
             android:layout_height="wrap_content"
-            android:gravity="right"
-            android:layout_gravity="right"
+            android:gravity="right|center_vertical"
+            android:layout_gravity="right|center_vertical"
             android:orientation="vertical"
             android:translationZ="@dimen/volume_dialog_elevation"
             android:clipChildren="false"
             android:clipToPadding="false"
-            android:background="@drawable/rounded_bg_full" >
+            android:background="@drawable/volume_rounded_bg" >
             <LinearLayout
                 android:id="@+id/volume_dialog_rows"
                 android:layout_width="wrap_content"
@@ -95,8 +91,7 @@
             <FrameLayout
                 android:id="@+id/settings_container"
                 android:layout_width="match_parent"
-                android:layout_height="wrap_content"
-                android:background="@drawable/rounded_bg_bottom_background">
+                android:layout_height="0px">
                 <com.android.keyguard.AlphaOptimizedImageButton
                     android:id="@+id/settings"
                     android:src="@drawable/ic_tune_black_16dp"
@@ -113,13 +108,12 @@
         <FrameLayout
             android:id="@+id/odi_captions"
             android:layout_width="@dimen/volume_dialog_caption_size"
-            android:layout_height="@dimen/volume_dialog_caption_size"
+            android:layout_height="0px"
             android:layout_marginTop="@dimen/volume_dialog_spacer"
             android:gravity="right"
             android:layout_gravity="right"
             android:clipToPadding="false"
-            android:translationZ="@dimen/volume_dialog_elevation"
-            android:background="@drawable/rounded_bg_full">
+            android:translationZ="@dimen/volume_dialog_elevation">
             <com.android.systemui.volume.CaptionsToggleImageButton
                 android:id="@+id/odi_captions_icon"
                 android:src="@drawable/ic_volume_odi_captions_disabled"
```

3.2 ic_volume_media.xml 和 ic_volume_media_mute.xml 关于音量条下静音的图标布局的修改
-------------------------------------------------------------------

```
ic_volume_media.xml的具体修改patch
   --- a/vendor/mediatek/proprietary/packages/apps/SystemUI/res/drawable/ic_volume_media.xml
+++ b/vendor/mediatek/proprietary/packages/apps/SystemUI/res/drawable/ic_volume_media.xml
@@ -14,14 +14,30 @@
      limitations under the License.
 -->
 <vector xmlns:android="http://schemas.android.com/apk/res/android"
-    android:height="24.0dp"
-    android:viewportHeight="24.0"
-    android:viewportWidth="24.0"
-    android:width="24.0dp"
-    android:tint="?android:attr/colorControlNormal" >
-
+    android:width="24dp"
+    android:height="24dp"
+    android:viewportWidth="36"
+    android:viewportHeight="36">
     <path
-        android:fillColor="#FFFFFFFF"
-        android:pathData="M12 3l0.01 10.55c-0.59-0.34-1.27-0.55-2-0.55C7.79 13 6 14.79 6 17s1.79 4 4.01 4S14 19.21 14 17V7h4V3h-6z" />
-
+        android:fillAlpha="0"
+        android:fillColor="#000000"
+        android:fillType="nonZero"
+        android:pathData="M3.6,4.065h28.8v28.8h-28.8z"
+        android:strokeWidth="1"
+        android:strokeAlpha="0"
+        android:strokeColor="#00000000" />
+    <path
+        android:fillAlpha="0"
+        android:fillColor="#000000"
+        android:fillType="nonZero"
+        android:pathData="M0,0h36v36h-36z"
+        android:strokeWidth="1"
+        android:strokeAlpha="0"
+        android:strokeColor="#00000000" />
+    <path
+        android:fillColor="#FFFFFF"
+        android:fillType="nonZero"
+        android:pathData="M19.194,7.314C18.4503,6.9054 17.5438,6.9307 16.824,7.38L8.16,12.828L5.28,12.828C3.996,12.828 2.946,13.872 2.946,15.162L2.946,23.592C2.946,24.876 3.99,25.926 5.28,25.926L8.16,25.926L16.83,31.374C17.208,31.614 17.64,31.728 18.072,31.728C18.462,31.728 18.852,31.632 19.2,31.434C19.9436,31.0244 20.4056,30.2429 20.406,29.394L20.406,9.354C20.4006,8.5049 19.9372,7.7248 19.194,7.314L19.194,7.314ZM18,29.274L9.144,23.706C8.952,23.586 8.73,23.52 8.508,23.52L5.346,23.52L5.346,15.228L8.508,15.228C8.736,15.228 8.958,15.162 9.144,15.042L18,9.474L18,29.274ZM28.362,19.374C28.362,16.68 26.916,14.178 24.582,12.84C24.006,12.51 23.274,12.708 22.944,13.284C22.614,13.86 22.812,14.592 23.388,14.922C24.9805,15.8417 25.9615,17.541 25.9615,19.38C25.9615,21.219 24.9805,22.9183 23.388,23.838C22.812,24.168 22.614,24.9 22.944,25.476C23.1035,25.7526 23.3665,25.9543 23.675,26.0365C23.9835,26.1187 24.3121,26.0746 24.588,25.914C26.916,24.57 28.362,22.068 28.362,19.374L28.362,19.374ZM27.594,8.436C27.018,8.106 26.286,8.304 25.956,8.874C25.626,9.444 25.824,10.182 26.394,10.512C29.55,12.336 31.512,15.732 31.512,19.374C31.512,23.016 29.55,26.412 26.394,28.236C25.818,28.566 25.626,29.304 25.956,29.874C26.178,30.258 26.58,30.474 26.994,30.474C27.198,30.474 27.402,30.42 27.594,30.312C31.4978,28.0515 33.9044,23.885 33.912,19.374C33.912,14.874 31.494,10.686 27.594,8.436Z"
+        android:strokeWidth="0.5"
+        android:strokeColor="#00000000" />
 </vector>
```

在上述代码中可以看到，通过增加自定义的 ic_volume_media.xml 作为喇叭的矢量布局的 xml 文件

来增加到布局中，方便对自定义音量条的调整

ic_volume_media_mute.xml 的具体修改 patch

```
--- a/vendor/mediatek/proprietary/packages/apps/SystemUI/res/drawable/ic_volume_media_mute.xml
+++ b/vendor/mediatek/proprietary/packages/apps/SystemUI/res/drawable/ic_volume_media_mute.xml
@@ -14,14 +14,43 @@
      limitations under the License.
 -->
 <vector xmlns:android="http://schemas.android.com/apk/res/android"
-    android:height="24.0dp"
-    android:viewportHeight="24.0"
-    android:viewportWidth="24.0"
-    android:width="24.0dp"
-    android:tint="?android:attr/colorControlNormal" >
-
+    android:width="24dp"
+    android:height="24dp"
+    android:viewportWidth="36"
+    android:viewportHeight="36">
     <path
-        android:fillColor="#FFFFFFFF"
-        android:pathData="M21.19 21.19L14 14l-2-2-9.2-9.2-1.41 1.42 8.79 8.79c-0.06 0-0.12-0.01-0.18-0.01-2.21 0-4 1.79-4 4s1.79 4 4.01 4S14 19.21 14 17v-0.17l5.78 5.78 1.41-1.42zM14 11.17V7h4V3h-6v6.17z" />
-
+        android:fillAlpha="0"
+        android:fillColor="#000000"
+        android:fillType="nonZero"
+        android:pathData="M3.6,4.065h28.8v28.8h-28.8z"
+        android:strokeWidth="1"
+        android:strokeAlpha="0"
+        android:strokeColor="#00000000" />
+    <path
+        android:fillAlpha="0"
+        android:fillColor="#000000"
+        android:fillType="nonZero"
+        android:pathData="M0,0h36v36h-36z"
+        android:strokeWidth="1"
+        android:strokeAlpha="0"
+        android:strokeColor="#00000000" />
+    <path
+        android:fillColor="#FFFFFF"
+        android:fillType="nonZero"
+        android:pathData="M19.3149,7.0952C20.0824,7.5194 20.5803,8.2996 20.6481,9.1669L20.656,9.354L20.656,29.3941C20.6555,30.3341 20.1439,31.1994 19.3236,31.6513C18.9441,31.8672 18.5129,31.978 18.072,31.978C17.6514,31.978 17.2416,31.8802 16.876,31.689L16.697,31.5857L8.088,26.176L5.28,26.176C3.9659,26.176 2.8795,25.1911 2.717,23.9222L2.7011,23.7551L2.696,23.592L2.696,15.162C2.696,13.7907 3.7684,12.6675 5.1169,12.5831L5.28,12.578L8.088,12.577L16.6916,7.1679C17.4882,6.6707 18.4913,6.6427 19.3149,7.0952ZM27.7189,8.2195C31.6957,10.5138 34.162,14.784 34.162,19.3744C34.1542,23.9745 31.7001,28.2232 27.7166,30.5299C27.489,30.6579 27.243,30.724 26.994,30.724C26.4811,30.724 26.0015,30.4523 25.7396,29.9993C25.3392,29.3076 25.5753,28.4169 26.2689,28.0196C29.348,26.24 31.262,22.9262 31.262,19.374C31.262,15.8218 29.348,12.508 26.2687,10.7284C25.6196,10.3526 25.3702,9.5412 25.6758,8.8723L25.7396,8.7487C26.1392,8.0586 27.0239,7.8212 27.7189,8.2195ZM17.75,9.926L9.2795,15.2521C9.0975,15.3695 8.8887,15.4437 8.6719,15.4686L8.508,15.478L5.596,15.477L5.596,23.269L8.508,23.27C8.7224,23.27 8.9368,23.3204 9.1333,23.4151L9.2771,23.4944L17.75,28.821L17.75,9.926ZM24.7063,12.6231C27.1173,14.0053 28.612,16.5896 28.612,19.374C28.612,22.1559 27.1195,24.7412 24.7137,26.1301C24.3804,26.3241 23.9834,26.3773 23.6107,26.278C23.2379,26.1787 22.9201,25.935 22.7271,25.6003C22.3284,24.9044 22.568,24.0196 23.263,23.6215C24.7781,22.7465 25.7115,21.1297 25.7115,19.38C25.7115,17.7064 24.8575,16.1544 23.4579,15.2572L23.2637,15.1389C22.609,14.7638 22.3581,13.958 22.6633,13.2844L22.7271,13.1597C23.1257,12.464 24.0104,12.2244 24.7063,12.6231Z"
+        android:strokeWidth="1"
+        android:strokeColor="#00000000" />
+    <path
+        android:fillColor="#FFFFFF"
+        android:fillType="nonZero"
+        android:pathData="M31.2995,11.1729L2.8768,25.5623L2.7491,25.635C2.0444,26.0924 1.8021,26.9819 2.172,27.7125C2.5711,28.5009 3.5338,28.8164 4.3222,28.4173L32.7449,14.0278L32.8726,13.9551C33.5774,13.4977 33.8196,12.6082 33.4497,11.8777C33.0506,11.0893 32.0879,10.7737 31.2995,11.1729ZM33.0036,12.1035C33.2597,12.6094 33.0878,13.2196 32.6231,13.5218L32.5191,13.5817L4.0963,27.9712C3.5543,28.2456 2.8925,28.0287 2.6181,27.4866C2.362,26.9808 2.5339,26.3705 2.9986,26.0683L3.1026,26.0084L31.5254,11.6189C32.0674,11.3445 32.7292,11.5615 33.0036,12.1035Z"
+        android:strokeWidth="1"
+        android:strokeColor="#00000000" />
+    <path
+        android:fillColor="#FFFFFF"
+        android:fillType="nonZero"
+        android:pathData="M31.5254,11.6189C32.0674,11.3445 32.7292,11.5615 33.0036,12.1035C33.2597,12.6094 33.0878,13.2196 32.6231,13.5218L32.5191,13.5817L4.0963,27.9712C3.5543,28.2456 2.8925,28.0287 2.6181,27.4866C2.362,26.9808 2.5339,26.3705 2.9986,26.0683L3.1026,26.0084L31.5254,11.6189Z"
+        android:strokeWidth="1"
+        android:strokeColor="#00000000" />
 </vector>
```

通过上述的 ic_volume_media_mute.xml 和 ic_volume_media.xml 关于音量条下方的喇叭的  
矢量图的 patch 可以替换原来的布局图标

上面这些 对布局的修改，就是实现自定义系统音量条的第一部分，接下来会实现第二部分
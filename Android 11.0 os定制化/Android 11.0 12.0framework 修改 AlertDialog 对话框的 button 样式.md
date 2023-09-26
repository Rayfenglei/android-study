> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/124739942)

### 1. 概述

在 11.0 12.0 系统产品开发中 [AlertDialog](https://so.csdn.net/so/search?q=AlertDialog&spm=1001.2101.3001.7020) 系统对话框原生的确定和取消 两个 button 字体默认颜色的不太好看, 由于产品的需求修改 button 字体的颜色，所以需要找到 AlertDialog 的字体样式然后修改就可以了

### 2.framework 修改 AlertDialog 对话框的 button 样式的核心类

```
frameworks\base\core\res\res\layout\alert_dialog.xml
frameworks/base/core/res/res/values/styles_device_defaults.xml

```

### 3.framework 修改 AlertDialog 对话框的 button 样式的核心功能实现和分析

### 3.1alert_dialog.xml 相关源码分析

首选先看布局文件 alert_dialog.xml 中采用的哪个布局样式  
布局为: frameworks\base\core\res\res\layout\alert_dialog.xml

```
<?xml version="1.0" encoding="utf-8"?>

<LinearLayout
xmlns:android="http://schemas.android.com/apk/res/android"
android:id="@+id/parentPanel"
android:layout_width="match_parent"
android:layout_height="wrap_content"
android:orientation="vertical"
android:paddingTop="9dip"
android:paddingBottom="3dip"
android:paddingStart="3dip"
android:paddingEnd="1dip">
<LinearLayout android:id="@+id/topPanel"
android:layout_width="match_parent"
android:layout_height="wrap_content"
android:minHeight="54dip"
android:orientation="vertical">
<LinearLayout android:id="@+id/title_template"
android:layout_width="match_parent"
android:layout_height="wrap_content"
android:orientation="horizontal"
android:gravity="center_vertical"
android:layout_marginTop="6dip"
android:layout_marginBottom="9dip"
android:layout_marginStart="10dip"
android:layout_marginEnd="10dip">
<ImageView android:id="@+id/icon"
android:layout_width="wrap_content"
android:layout_height="wrap_content"
android:layout_gravity="top"
android:paddingTop="6dip"
android:paddingEnd="10dip"
android:src="@drawable/ic_dialog_info" />
<com.android.internal.widget.DialogTitle android:id="@+id/alertTitle"
style="?android:attr/textAppearanceLarge"
android:singleLine="true"
android:ellipsize="end"
android:layout_width="match_parent"
android:layout_height="wrap_content"
android:textAlignment="viewStart" />
</LinearLayout>
<ImageView android:id="@+id/titleDivider"
android:layout_width="match_parent"
android:layout_height="1dip"
android:visibility="gone"
android:scaleType="fitXY"
android:gravity="fill_horizontal"
android:src="@android:drawable/divider_horizontal_dark" />
<!-- If the client uses a customTitle, it will be added here. -->
</LinearLayout>
<LinearLayout android:id="@+id/contentPanel"
android:layout_width="match_parent"
android:layout_height="wrap_content"
android:layout_weight="1"
android:orientation="vertical">
<ScrollView android:id="@+id/scrollView"
android:layout_width="match_parent"
android:layout_height="wrap_content"
android:paddingTop="2dip"
android:paddingBottom="12dip"
android:paddingStart="14dip"
android:paddingEnd="10dip"
android:overScrollMode="ifContentScrolls">
<TextView android:id="@+id/message"
style="?android:attr/textAppearanceMedium"
android:layout_width="match_parent"
android:layout_height="wrap_content"
android:padding="5dip" />
</ScrollView>
</LinearLayout>
<FrameLayout android:id="@+id/customPanel"
android:layout_width="match_parent"
android:layout_height="wrap_content"
android:layout_weight="1">
<FrameLayout android:id="@+android:id/custom"
android:layout_width="match_parent"
android:layout_height="wrap_content"
android:paddingTop="5dip"
android:paddingBottom="5dip" />
</FrameLayout>
<LinearLayout android:id="@+id/buttonPanel"
android:layout_width="match_parent"
android:layout_height="wrap_content"
android:minHeight="54dip"
android:orientation="vertical" >
<LinearLayout
style="?android:attr/buttonBarStyle"
android:layout_width="match_parent"
android:layout_height="wrap_content"
android:orientation="horizontal"
android:paddingTop="4dip"
android:paddingStart="2dip"
android:paddingEnd="2dip"
android:measureWithLargestChild="true">
<LinearLayout android:id="@+id/leftSpacer"
android:layout_weight="0.25"
android:layout_width="0dip"
android:layout_height="wrap_content"
android:orientation="horizontal"
android:visibility="gone" />
<Button android:id="@+id/button1"
android:layout_width="0dip"
android:layout_gravity="start"
android:layout_weight="1"
style="?android:attr/buttonBarButtonStyle"
android:maxLines="2"
android:layout_height="wrap_content" />
<Button android:id="@+id/button3"
android:layout_width="0dip"
android:layout_gravity="center_horizontal"
android:layout_weight="1"
style="?android:attr/buttonBarButtonStyle"
android:maxLines="2"
android:layout_height="wrap_content" />
<Button android:id="@+id/button2"
android:layout_width="0dip"
android:layout_gravity="end"
android:layout_weight="1"
style="?android:attr/buttonBarButtonStyle"
android:maxLines="2"
android:layout_height="wrap_content" />
<LinearLayout android:id="@+id/rightSpacer"
android:layout_width="0dip"
android:layout_weight="0.25"
android:layout_height="wrap_content"
android:orientation="horizontal"
android:visibility="gone" />
</LinearLayout>
</LinearLayout>
</LinearLayout>

```

从上述的代码可以看出 button1 和 button2 的 Button 的样式为  
样式为 style=“?android:attr/buttonBarButtonStyle”  
而在 themes_device_defaults.xml 中

### 3.2 themes_device_defaults.xml 核心代码分析

路径：framework/base/core/res/res/values/themes_device_defaults.xml:1663:

```
 <item >@style/Widget.DeviceDefault.Button.ButtonBar.AlertDialog</item>

```

而 Widget.DeviceDefault.Button.ButtonBar.AlertDialog 样式在 styles_device_defaults.xml  
路径 frameworks/base/core/res/res/values/styles_device_defaults.xml

### 3.3 styles_device_defaults.xml 的相关样式分析

所以具体修改如下

```
--- a/frameworks/base/core/res/res/values/styles_device_defaults.xml
+++ b/frameworks/base/core/res/res/values/styles_device_defaults.xml
@@ -103,6 +103,7 @@ easier.
<style >
<item >@dimen/alert_dialog_button_bar_width</item>
<item >@dimen/alert_dialog_button_bar_height</item>
+ <item >#E8E8E8</item> 
</style>
<style />
<style />

```

在 Widget.DeviceDefault.Button.ButtonBar.AlertDialog 的系统 AlertDialog 增加字体默认颜色  
增加 textColor 字段

```
<item >#E8E8E8</item>

```

这个字段就行了
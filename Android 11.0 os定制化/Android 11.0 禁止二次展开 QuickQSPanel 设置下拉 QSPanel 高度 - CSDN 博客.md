> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/133101081)

1. 前言
-----

  在 11.0 的系统定制化需求中，在进行 systemui 的 ui 定制开发中，有些产品中有需求对原生 systemui 下拉状态栏中的二次展开 QSPanel 修改成  
一次展开禁止二次展开，所以就需要修改 QuickQSpanel 的高度，然后在 QuickQsPanel 做定制，然后禁止二次展开就可以了  
如图:

 ![](https://img-blog.csdnimg.cn/eead5bbce39c4f2b855444e520cb7891.png)  
2. 禁止二次展开 QuickQSPanel 设置下拉 QSPanel 高度的核心类
-------------------------------------------------------------------------------------------------------------------

```
    framework\base\packages\SystemUI\res\layout\quick_status_bar_expanded_header.xml
    framework\base\packages\SystemUI\res\values\dimens.xml
    framework\base\packages\SystemUI\src\com\android\systemui\qs\QuickStatusBarHeader.java
    framework\base\packages\SystemUI\src\com\android\systemui\statusbar\phone\NotificationPanelView.java
```

3. 禁止二次展开 QuickQSPanel 设置下拉 QSPanel 高度的核心功能分析和实现
------------------------------------------------

在系统 SystemUI 中，它主要负责反馈系统及应用状态并与用户保持大量的交互, systemui 中主要核心布局控件为以下部分  
在 SystemUI 中，QSPanel 创建是从 StatusBar#makeStatusBarView 开始的  
接下来分析的核心功能和布局如下:  
StatusBar：通知消息提示和状态展现  
NavigationBar：返回，HOME，Recent  
KeyGuard：锁屏模块可以看做单独的应用，提供基本的手机个人隐私保护  
Recents：近期应用管理，以堆叠栈的形式展现。  
Notification Panel：展示系统或应用通知内容。提供快速系统[设置开关](https://so.csdn.net/so/search?q=%E8%AE%BE%E7%BD%AE%E5%BC%80%E5%85%B3&spm=1001.2101.3001.7020)。  
VolumeUI：来用展示或控制音量的变化：媒体音量、铃声音量与闹钟音量  
截屏界面：长按电源键 + 音量下键后截屏，用以展示截取的屏幕照片 / 内容  
PowerUI：主要处理和 [Power](https://so.csdn.net/so/search?q=Power&spm=1001.2101.3001.7020) 相关的事件，比如省电模式切换、电池电量变化和开关屏事件等。  
RingtonePlayer：铃声播放  
StackDivider：控制管理分屏  
PipUI：提供对于画中画模式的管理

3.1 quick_status_bar_expanded_header.xml 关于下拉状态栏头部布局分析
------------------------------------------------------

  
禁止二次展开 QuickQSPanel 设置下拉 QSPanel 高度的核心功能实现中，在系统 SystemUI 中的下拉状态栏中关于下拉状态栏头部的布局  
就是 quick_status_bar_expanded_header.xml，接下来具体分析下 quick_status_bar_expanded_header.xml 的核心源码

```
    <com.android.systemui.qs.QuickStatusBarHeader
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:id="@+id/header"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_gravity="@integer/notification_panel_layout_gravity"
        android:background="@android:color/transparent"
        android:baselineAligned="false"
        android:clickable="false"
        android:clipChildren="false"
        android:clipToPadding="false"
        android:paddingTop="0dp"
        android:paddingEnd="0dp"
        android:paddingStart="0dp"
        android:elevation="4dp" >
     
        <!-- The clock -->
        <include layout="@layout/quick_status_bar_header_system_icons" />
     
        <!-- Status icons within the panel itself (and not in the top-most status bar) -->
        <include layout="@layout/quick_qs_status_icons" />
     
        <!-- Layout containing tooltips, alarm text, etc. -->
        <include layout="@layout/quick_settings_header_info" />
     
        <com.android.systemui.qs.QuickQSPanel
            android:id="@+id/quick_qs_panel"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_below="@id/quick_qs_status_icons"
            android:accessibilityTraversalAfter="@+id/date_time_group"
            android:accessibilityTraversalBefore="@id/expand_indicator"
            android:clipChildren="false"
            android:clipToPadding="false"
            android:focusable="true"
            android:paddingBottom="10dp"
            android:importantForAccessibility="yes" />
    ....
```

在上述的 quick_status_bar_expanded_header.xml 的相关源码中，从中分析得知，在 QuickStatusBarHeader  
这个 systemui 下拉状态栏头部布局中，这里负责构建管理 QuickQSPanel 状态栏图标等等相关布局，所以  
可以从 QuickStatusBarHeader 中来设置下拉状态栏的头部布局，接下来看下  
QuickStatusBarHeader 的相关布局分析

3.2 QuickStatusBarHeader 下拉状态栏头部布局分析
------------------------------------

  
禁止二次展开 QuickQSPanel 设置下拉 QSPanel 高度的核心功能实现中，在系统 SystemUI 中的下拉状态栏中，从上述的  
quick_status_bar_expanded_header.xml 的相关源码中分析得知，在 QuickStatusBarHeader 就是关于状态栏的头部文件  
类，接下来分析相关的源码

```
     public class QuickStatusBarHeader extends RelativeLayout implements
             View.OnClickListener, NextAlarmController.NextAlarmChangeCallback,
             ZenModeController.Callback {
         private static final String TAG = "QuickStatusBarHeader";
         private static final boolean DEBUG = false;
     
    @Override
          protected void onFinishInflate() {
              super.onFinishInflate();
      
              mHeaderQsPanel = findViewById(R.id.quick_qs_panel);
              mSystemIconsView = findViewById(R.id.quick_status_bar_system_icons);
              mQuickQsStatusIcons = findViewById(R.id.quick_qs_status_icons);
              StatusIconContainer iconContainer = findViewById(R.id.statusIcons);
              iconContainer.setShouldRestrictIcons(false);
              mIconManager = new TintedIconManager(iconContainer);
      
              // Views corresponding to the header info section (e.g. ringer and next alarm).
              mHeaderTextContainerView = findViewById(R.id.header_text_container);
              mStatusSeparator = findViewById(R.id.status_separator);
              mNextAlarmIcon = findViewById(R.id.next_alarm_icon);
              mNextAlarmTextView = findViewById(R.id.next_alarm_text);
              mNextAlarmContainer = findViewById(R.id.alarm_container);
              mNextAlarmContainer.setOnClickListener(this::onClick);
              mRingerModeIcon = findViewById(R.id.ringer_mode_icon);
              mRingerModeTextView = findViewById(R.id.ringer_mode_text);
              mRingerContainer = findViewById(R.id.ringer_container);
              mCarrierGroup = findViewById(R.id.carrier_group);
      
      
              updateResources();
             ....
    		}
     
     
          private void updateResources() {
              Resources resources = mContext.getResources();
              updateMinimumHeight();
      
              // Update height for a few views, especially due to landscape mode restricting space.
              mHeaderTextContainerView.getLayoutParams().height =
                      resources.getDimensionPixelSize(R.dimen.qs_header_tooltip_height);
              mHeaderTextContainerView.setLayoutParams(mHeaderTextContainerView.getLayoutParams());
      
              mSystemIconsView.getLayoutParams().height = resources.getDimensionPixelSize(
                      com.android.internal.R.dimen.quick_qs_offset_height);
              mSystemIconsView.setLayoutParams(mSystemIconsView.getLayoutParams());
      
              FrameLayout.LayoutParams lp = (FrameLayout.LayoutParams) getLayoutParams();
              if (mQsDisabled) {
                  lp.height = resources.getDimensionPixelSize(
           -               com.android.internal.R.dimen.quick_qs_offset_height);
         +              com.android.internal.R.dimen.quick_qs_height);
              } else {
                  lp.height = Math.max(getMinimumHeight(),
                          resources.getDimensionPixelSize(
            -               com.android.internal.R.dimen.quick_qs_offset_height);
         +              com.android.internal.R.dimen.quick_qs_height);
              }
      
              setLayoutParams(lp);
      
              updateStatusIconAlphaAnimator();
              updateHeaderTextContainerAlphaAnimator();
          }
```

禁止二次展开 QuickQSPanel 设置下拉 QSPanel 高度的核心功能实现中，  
从上述的 QuickStatusBarHeader.java 中的源码中分析得知，在 onFinishInflate() 中负责构建  
下拉状态栏头部布局的相关源码，而在 updateResources(); 中负责更新相关的布局高度等参数，  
在 updateResources(); 中 FrameLayout.LayoutParams lp 的 height 就是关于布局的高度，  
所以可以在这里重新定义下拉状态栏头部的布局高度，这样就可以加高 QuickQSPanel 的高度  
在自定义 QuickQSPanel 的时候 可以增加其他布局，  
具体在 [framework](https://so.csdn.net/so/search?q=framework&spm=1001.2101.3001.7020)\base\packages\SystemUI\res\values\dimens.xml 中增加 quick_qs_height 的参数为:  
 

```
    <resources>
        <!-- Recommended minimum clickable element dimension -->
        <dimen >48dp</dimen>
     +   <dimen >256dp</dimen>
        <!-- Amount to offset bottom of notification peek window from top of status bar. -->
        <dimen >-12dp</dimen>
     
        <!-- thickness (height) of the navigation bar on phones that require it -->
        <dimen >@*android:dimen/navigation_bar_height</dimen>
        <!-- Minimum swipe distance to catch the swipe gestures to invoke assist or switch tasks. -->
        <dimen >48dp</dimen>
        <dimen >500dp</dimen>
```

3.3 NotificationPanelView.java 中禁止展开下拉状态栏 QuickQSPanel 布局的相关分析
--------------------------------------------------------------

禁止二次展开 QuickQSPanel 设置下拉 QSPanel 高度的核心功能实现中，  
在状态栏从头下拉时，直接走 NotificationPanelView 的 onTouchEvent 方法，不走拦截方法，  
down 事件经过 PanelView 中 onTouchEvent 时，mExpandedHeight = 0.0，  
调用 onTrackingStarted();  
再调用 setExpandedHeightInternal(newHeight) 设置高度，走一遍 onLayout 方法；  
此后 move 事件继续传入，走 setExpandedHeightInternal(newHeight) 方法。  
一旦 newHeight 到达一个值后，调用 setExpandedHeightInternal() 方法中的 setOverExpansion 方法，  
继而调用 mNotificationStackScroller.setOverScrolledPixels(overExpansion, true /* onTop */, false /* animate */) 方法，  
从而调用 setQsExpansion(mQsMinExpansionHeight + rounded) 方法；

```
    private void setQsExpansion(float height) {
     
    +     // add core start
    +        if (mQs != null) return;
    +    //add core end
     
              height = Math.min(Math.max(height, mQsMinExpansionHeight), mQsMaxExpansionHeight);
              mQsFullyExpanded = height == mQsMaxExpansionHeight && mQsMaxExpansionHeight != 0;
              if (height > mQsMinExpansionHeight && !mQsExpanded && !mStackScrollerOverscrolling) {
                  setQsExpanded(true);
              } else if (height <= mQsMinExpansionHeight && mQsExpanded) {
                  setQsExpanded(false);
              }
              mQsExpansionHeight = height;
              updateQsExpansion();
              requestScrollerTopPaddingUpdate(false /* animate */);
              updateHeaderKeyguardAlpha();
    ...
              if (DEBUG) {
                  invalidate();
              }
          }
```

禁止二次展开 QuickQSPanel 设置下拉 QSPanel 高度的核心功能实现中，  
通过上述的 NotificationPanelView.java 中源码中，分析得知在从状态栏头部下拉状态栏的时候，最终是由  
setQsExpansion(float height) 来负责展开 QSPanel 布局的，所以可以在这里禁止展开 QSPanel 布局，就需要  
添加条件 mQs != null 这样就可以在状态栏界面禁止下拉状态栏了